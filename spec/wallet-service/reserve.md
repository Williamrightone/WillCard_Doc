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

### Response (`wallet.contract.dto.WalletReserveRs`)

| Field | Type | Description |
|-------|------|-------------|
| reservationId | Long | Snowflake ID of the created `wallet_reservation`; passed back to ORCH for Leg 2 |

---

## 4. Service UseCase Flow

**UseCase:** `WalletReserveUseCase.reserve(WalletReserveRq)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `wallet_account` WHERE `user_id = rq.userId`; not found → `WA00001` |
| 2 | `[DOMAIN]` | Validate `available_balance >= rq.amount`; insufficient → `WA00002` |
| 3 | `[DB READ]` | SELECT `point_reward_batch` WHERE `user_id = rq.userId AND status = 'CONFIRMED' AND remaining_balance > 0 AND expires_at > NOW()` ORDER BY `expires_at ASC` (FIFO) |
| 4 | `[DOMAIN]` | Traverse batches in FIFO order, accumulating until `rq.amount` is satisfied; if total available < `rq.amount` → `WA00002`; build `allocationList` of `(batchId, drawAmount)` pairs |
| 5 | `[DOMAIN]` | Generate Snowflake `reservationId` |
| 6 | `[DB WRITE]` | INSERT `wallet_reservation { reservation_id, user_id, total_amount: rq.amount, status: 'PENDING', created_at, updated_at }` |
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
| `wallet_reservation` | INSERT | New reservation header (`status = PENDING`) |
| `wallet_reservation_item` | INSERT | One row per batch drawn in FIFO allocation |
| `point_reward_batch` | UPDATE | `remaining_balance -= drawAmount` per allocated batch |
| `wallet_account` | UPDATE | `available_balance -= amount`, `reserved_balance += amount` |

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | WA00001 | WALLET_NOT_FOUND | No `wallet_account` for given `userId` |
| 422 | WA00002 | INSUFFICIENT_BALANCE | `available_balance < amount` or total CONFIRMED batches < amount |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 7. Changelog

### v1.0 — 2026-03 — Initial spec
