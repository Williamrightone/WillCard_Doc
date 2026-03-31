# Points Credit (Internal) — Wallet Service Spec

> **Service:** wallet-service
> **Last updated:** 2026-03

---

## 1. Overview

Internal API called exclusively by Points Service to create a `point_reward_batch` record in `PENDING` state.
Invoked after Points Service calculates `reward_points` from a `txn.card.authorized` Kafka event.
The batch transitions to `CONFIRMED` via the wallet-service T+1 settlement job (nightly).

> **Not exposed to BFF or external callers.** Path prefix `/internal/` is blocked at the Nginx layer.

---

## 2. API Definition — wallet-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Path | `/wallet-service/internal/points/credit` |
| Context-path | `server.servlet.context-path: /wallet-service` |
| Caller | points-service only (internal service network) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Content-Type | ✅ | `application/json` |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| userId | Long | ✅ | Owner member ID |
| sourceTxnId | String | ✅ | Source transaction ID; idempotency key. Duplicate requests (same `sourceTxnId`) are rejected with `DUPLICATE_SOURCE_TXN` |
| issuedAmount | Long | ✅ | Points to credit in TWD cents; must be > 0 |
| expiresAt | String | ✅ | ISO-8601 datetime; computed by Points Service as the last moment of the same calendar month one year after `txnTimestamp` |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| batchId | Long | Snowflake ID of the newly created `point_reward_batch` |

---

## 3. Service UseCase Flow

**UseCase:** `PointsCreditUseCase.credit()`
**UseCase Impl:** `PointsCreditUseCaseImpl`

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[DB READ]` | Check `point_reward_batch` WHERE `source_txn_id = :sourceTxnId`; if exists → **return existing `{ batchId }` immediately (HTTP 200, idempotent success)** | — |
| 2 | `[DB READ]` | Check `wallet_account` WHERE `user_id = :userId`; verify account exists | Not found → throw `WALLET_NOT_FOUND` (404) |
| 3 | `[DOMAIN]` | Generate Snowflake `batch_id`; compute `remaining_balance = issuedAmount`; set `status = PENDING` | — |
| 4 | `[DB WRITE]` | INSERT `point_reward_batch` (`batch_id`, `user_id`, `source_txn_id`, `issued_amount`, `remaining_balance`, `status = PENDING`, `expires_at`, timestamps) | DB error → throw `INTERNAL_ERROR` |
| 5 | `[RETURN]` | Return `PointsCreditRs { batchId }` | — |

> **Note:** `wallet_account.available_balance` is NOT incremented here. It is incremented only when the batch transitions to `CONFIRMED` during T+1 settlement.

---

## 4. Database

### Tables Written

| Table | Operation | Description |
|-------|-----------|-------------|
| `point_reward_batch` | INSERT | New batch in `PENDING` state |

### Tables Read

| Table | Operation | Condition |
|-------|-----------|-----------|
| `point_reward_batch` | SELECT | `source_txn_id = :sourceTxnId` (idempotency check) |
| `wallet_account` | SELECT | `user_id = :userId` (existence check) |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 404 | WALLET_NOT_FOUND | Wallet account not found | `userId` has no `wallet_account` row |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |

> **Idempotent by design:** Duplicate `sourceTxnId` requests return HTTP 200 with the existing `batchId` — no error is thrown. This prevents Points Service from incorrectly recording `FAILED` status when the wallet batch was already created but the Points Service log INSERT had not yet committed (crash-recovery scenario).

---

## 6. Changelog

### v1.1 — 2026-03 — Make duplicate sourceTxnId idempotent
- Step 1: Changed duplicate handling from 409 DUPLICATE_SOURCE_TXN to idempotent HTTP 200 (return existing batchId). Prevents crash-recovery inconsistency in Points Service.

### v1.0 — 2026-03 — Initial spec
