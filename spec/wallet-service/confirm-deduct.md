# Confirm Deduct / Release — wallet-service Spec

> **Service:** wallet-service
> **Contract Module:** wallet-contract
> **Caller:** [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Finalizes a `PENDING` wallet reservation created by the `reserve` endpoint. Two outcomes:

- **Confirm Deduct** — Transaction approved; permanently deducts the reserved points. Updates reservation status to `CONFIRMED` and decrements `reserved_balance`.
- **Release** — Transaction declined or cancelled; restores the reserved points to their source batches. Updates reservation status to `RELEASED`, increments `available_balance`, and restores per-batch `remaining_balance`.

Both operations are idempotent-safe via reservation status guard: calling either endpoint on an already-finalized reservation (`CONFIRMED` or `RELEASED`) returns `WA00004`.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller — ORCH Leg 2 APPROVED path | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| Reservation created by | [wallet-service/reserve.md](reserve.md) |

---

## 3. API Definition — wallet-service

### Endpoint A — Confirm Deduct

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations/{reservationId}/confirm |
| Effective received path | /wallet-service/reservations/{reservationId}/confirm |
| Caller | ORCH (`WalletFeignClient.confirmDeduct(reservationId)`) |

### Endpoint B — Release

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations/{reservationId}/release |
| Effective received path | /wallet-service/reservations/{reservationId}/release |
| Caller | ORCH (`WalletFeignClient.release(reservationId)`) |

### Path Variable (both endpoints)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reservationId | Long | ✅ | Snowflake ID of the `wallet_reservation` to finalize |

### Response (both endpoints)

HTTP 200 with empty body on success.

---

## 4. Service UseCase Flow — Confirm Deduct

**UseCase:** `WalletConfirmDeductUseCase.confirm(Long reservationId)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `wallet_reservation` WHERE `reservation_id = reservationId`; not found → `WA00003` |
| 2 | `[DOMAIN]` | Validate `status == 'PENDING'`; otherwise → `WA00004` |
| 3 | `[DB WRITE]` | UPDATE `wallet_reservation SET status = 'CONFIRMED', updated_at = NOW(3)` |
| 4 | `[DB WRITE]` | UPDATE `wallet_account SET reserved_balance -= reservation.total_amount, updated_at = NOW(3)` WHERE `user_id = reservation.user_id` |
| 5 | `[RETURN]` | HTTP 200 |

> **Points are permanently deducted** — `available_balance` was already decremented at `reserve()` time. Confirm only clears `reserved_balance`.
>
> **No batch updates:** `point_reward_batch.remaining_balance` was already reduced at `reserve()` and is not restored here.

---

## 5. Service UseCase Flow — Release

**UseCase:** `WalletReleaseUseCase.release(Long reservationId)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `wallet_reservation` WHERE `reservation_id = reservationId`; not found → `WA00003` |
| 2 | `[DOMAIN]` | Validate `status == 'PENDING'`; otherwise → `WA00004` |
| 3 | `[DB READ]` | SELECT `wallet_reservation_item` WHERE `reservation_id = reservationId` |
| 4 | `[DB WRITE]` | For each item: UPDATE `point_reward_batch SET remaining_balance += item.amount, updated_at = NOW(3)` WHERE `batch_id = item.batch_id` |
| 5 | `[DB WRITE]` | UPDATE `wallet_reservation SET status = 'RELEASED', updated_at = NOW(3)` |
| 6 | `[DB WRITE]` | UPDATE `wallet_account SET available_balance += reservation.total_amount, reserved_balance -= reservation.total_amount, updated_at = NOW(3)` WHERE `user_id = reservation.user_id` |
| 7 | `[RETURN]` | HTTP 200 |

> **Atomicity:** Steps 4–6 execute within a single database transaction.
>
> **Audit trail:** `wallet_reservation_item` rows are never deleted; they remain as an immutable record of which batches were drawn from, even after release.
>
> **Batch status:** Restored batches remain `CONFIRMED`; `remaining_balance` is simply incremented back. No status change on the batch.

---

## 6. Database

### Confirm Deduct

| Table | Operation | Condition |
|-------|-----------|-----------|
| `wallet_reservation` | SELECT | `reservation_id = ?` |
| `wallet_reservation` | UPDATE | `status = 'CONFIRMED'` |
| `wallet_account` | UPDATE | `reserved_balance -= total_amount` |

### Release

| Table | Operation | Condition |
|-------|-----------|-----------|
| `wallet_reservation` | SELECT | `reservation_id = ?` |
| `wallet_reservation_item` | SELECT | `reservation_id = ?` |
| `point_reward_batch` | UPDATE (per item) | `remaining_balance += item.amount` |
| `wallet_reservation` | UPDATE | `status = 'RELEASED'` |
| `wallet_account` | UPDATE | `available_balance += total_amount`, `reserved_balance -= total_amount` |

---

## 7. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | WA00003 | RESERVATION_NOT_FOUND | No `wallet_reservation` for given `reservationId` |
| 409 | WA00004 | INVALID_RESERVATION_STATUS | Reservation is not in `PENDING` status |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 8. Changelog

### v1.0 — 2026-03 — Initial spec
