# Reserve Points — wallet-service Spec

> **Service:** wallet-service
> **Contract Module:** wallet-contract
> **Caller:** [txn-orch/card-pay.md](../txn-orch/card-pay.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Locks a specified amount of points for an in-flight transaction. Called by the Transaction Orchestrator during the payment Leg 1 flow, after OTP session is established.

Performs FIFO deduction from `CONFIRMED` batches ordered by earliest expiry, creates a `wallet_reservation` header and `wallet_reservation_item` detail rows, and atomically updates the balance cache.

Returns a `reservationId` for the Orchestrator to reference in Leg 2 (`confirmDeduct` or `release`).

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (ORCH Leg 1) | [txn-orch/card-pay.md](../txn-orch/card-pay.md) |
| Finalize reservation | [wallet-service/confirm-deduct.md](confirm-deduct.md) |

---

## 3. API Definition — wallet-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations |
| Effective received path | /wallet-service/reservations |
| Caller | ORCH (`WalletFeignClient`) |

### Request (`wallet.contract.dto.WalletReserveRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| userId | Long | ✅ | @NotNull | Member Snowflake ID |
| amount | Long | ✅ | @NotNull; @Min(1) | Points to reserve (TWD cents) |
| idempotencyKey | String | ❌ | @Size(max=64) | Caller-supplied deduplication key (ORCH passes `challengeRef`). If a PENDING reservation with the same key exists, the existing `reservationId` is returned immediately without creating a new reservation. |

### Response (`wallet.contract.dto.WalletReserveRs`)

| Field | Type | Description |
|-------|------|-------------|
| reservationId | Long | Snowflake ID of the created `wallet_reservation`; passed back to ORCH for Leg 2 |

---

## 4. Service UseCase Flow

**UseCase:** `WalletReserveUseCase.reserve(WalletReserveRq)`

| Step | Type | Description |
|------|------|-------------|
| 0 | `[DB READ]` | **Idempotency check:** If `rq.idempotencyKey` is present → SELECT `wallet_reservation` WHERE `idempotency_key = rq.idempotencyKey AND status = 'PENDING'`; found → return existing `{ reservationId }` immediately (idempotent success) |
| 1 | `[DB READ]` | SELECT `wallet_account` WHERE `user_id = rq.userId`; not found → `WA00001` |
| 2 | `[DOMAIN]` | Validate `available_balance >= rq.amount`; insufficient → `WA00002` |
| 3 | `[DB READ]` | SELECT `point_reward_batch` WHERE `user_id = rq.userId AND status = 'CONFIRMED' AND remaining_balance > 0 AND expires_at > NOW()` ORDER BY `expires_at ASC` (FIFO) |
| 4 | `[DOMAIN]` | Traverse batches in FIFO order, accumulating until `rq.amount` is satisfied; if total available < `rq.amount` → `WA00002`; build `allocationList` of `(batchId, drawAmount)` pairs |
| 5 | `[DOMAIN]` | Generate Snowflake `reservationId` |
| 6 | `[DB WRITE]` | INSERT `wallet_reservation { reservation_id, user_id, total_amount: rq.amount, status: 'PENDING', idempotency_key: rq.idempotencyKey, created_at, updated_at }` |
| 7 | `[DB WRITE]` | INSERT `wallet_reservation_item` rows — one row per `(batchId, drawAmount)` in `allocationList`; generate Snowflake `item_id` per row |
| 8 | `[DB WRITE]` | For each item: UPDATE `point_reward_batch SET remaining_balance -= drawAmount` WHERE `batch_id = item.batchId` |
| 9 | `[DB WRITE]` | UPDATE `wallet_account SET available_balance -= rq.amount, reserved_balance += rq.amount, updated_at = NOW(3)` WHERE `user_id = rq.userId` |
| 10 | `[RETURN]` | Return `WalletReserveRs { reservationId }` |

> **Atomicity:** Steps 6–9 must execute within a single database transaction. Any failure triggers a full rollback; no partial state is persisted.
>
> **FIFO policy:** Batches with the earliest `expires_at` are drawn first to prevent older points from expiring unused.
>
> **available_balance vs remaining_balance:** `point_reward_batch.remaining_balance` is the authoritative per-batch source of truth. `wallet_account.available_balance` is a derived cache updated atomically in Step 9.

---

## 5. Database

**Tables read:**

| Table | Operation | Condition |
|-------|-----------|-----------|
| `wallet_account` | SELECT | `user_id = rq.userId` |
| `point_reward_batch` | SELECT (FIFO) | `user_id, status='CONFIRMED', remaining_balance>0, expires_at>NOW() ORDER BY expires_at ASC` |

**Tables written (single transaction):**

| Table | Operation | Description |
|-------|-----------|-------------|
| `wallet_reservation` | INSERT | New reservation header (`status = PENDING`, `idempotency_key` stored) |
| `wallet_reservation_item` | INSERT | One row per batch drawn in FIFO allocation |
| `point_reward_batch` | UPDATE | `remaining_balance -= drawAmount` per allocated batch |
| `wallet_account` | UPDATE | `available_balance -= amount`, `reserved_balance += amount` |

---

## 6. Scheduled Cleanup — Orphaned PENDING Reservations

When the ORCH Saga State expires (TTL 180s), any associated `wallet_reservation` created in Leg 1 becomes unreachable — the `reservationId` is lost and can never be confirmed or released via normal flow.

A scheduled job must run periodically to release these orphaned reservations:

| Field | Value |
|-------|-------|
| Trigger | Every 5 minutes (configurable) |
| Target | `wallet_reservation` WHERE `status = 'PENDING' AND created_at < NOW() - INTERVAL 10 MINUTE` |
| Action | For each orphaned reservation: execute the same logic as `release()` — restore `point_reward_batch.remaining_balance` per item, UPDATE `wallet_account.available_balance += total_amount, reserved_balance -= total_amount`, UPDATE `status = 'RELEASED'` |
| Atomicity | Each reservation released in its own DB transaction |

> **Why 10 minutes?** Saga TTL is 180s (~3 min). A 10-minute threshold ensures no in-flight reservation is accidentally released due to clock skew or processing delay.

---

## 7. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | WA00001 | WALLET_NOT_FOUND | No `wallet_account` for given `userId` |
| 422 | WA00002 | INSUFFICIENT_BALANCE | `available_balance < amount` or total CONFIRMED batches < amount |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 8. Changelog

### v1.1 — 2026-03 — Add idempotency + scheduled cleanup
- Added `idempotencyKey` (Optional) to request; Step 0 idempotency check before reserve execution.
- Added § Scheduled Cleanup for orphaned PENDING reservations (Issue 2 + Issue 3).
- `wallet_reservation` INSERT now stores `idempotency_key`.

### v1.0 — 2026-03 — Initial spec
