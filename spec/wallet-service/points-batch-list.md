# Get Points Batch List — Wallet Service Spec

> **Service:** wallet-service
> **Last updated:** 2026-03

---

## 1. Overview

Returns a paginated list of the caller's `point_reward_batch` records, sorted by `expires_at ASC` (earliest-expiring first).
Used by the BFF to display the user's point balance breakdown per batch, including remaining balance, expiry date, and status.

> This API was deferred from Phase 2 (wallet data model only) and is implemented in Phase 3 once `point_reward_batch` is populated by Points Service.

---

## 2. API Definition — wallet-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| Path | `/wallet-service/points-batches` |
| Context-path | `server.servlet.context-path: /wallet-service` |
| Caller | BFF via `WalletFeignClient` |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| X-Member-Id | ✅ | Authenticated member ID forwarded by BFF |

### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | Integer | ❌ | 0 | 0-indexed page number |
| size | Integer | ❌ | 20 | Page size; maximum 50 |

### Response Body

```json
{
  "items": [
    {
      "batchId": 1234567890,
      "issuedAmount": 1750,
      "remainingBalance": 1200,
      "status": "CONFIRMED",
      "expiresAt": "2027-03-31T23:59:59"
    }
  ],
  "totalCount": 5,
  "page": 0,
  "size": 20
}
```

| Field | Type | Description |
|-------|------|-------------|
| items | Array | Batch records for the current page |
| items[].batchId | Long | Snowflake batch ID |
| items[].issuedAmount | Long | Total points originally issued in this batch (TWD cents) |
| items[].remainingBalance | Long | Current available points in this batch (TWD cents) |
| items[].status | String | `PENDING / CONFIRMED / DEPLETED / EXPIRED` |
| items[].expiresAt | String | ISO-8601 expiry datetime |
| totalCount | Long | Total number of batches matching the query |
| page | Integer | Current page (0-indexed) |
| size | Integer | Page size used |

> **Display note:** Batches with `status = DEPLETED` or `EXPIRED` are included in the list for historical reference. Clients should highlight batches approaching expiry (e.g., within 30 days).

---

## 3. Service UseCase Flow

**UseCase:** `PointsBatchListUseCase.list()`
**UseCase Impl:** `PointsBatchListUseCaseImpl`

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[DB READ]` | Query `point_reward_batch` WHERE `user_id = :userId` ORDER BY `expires_at ASC`, with pagination (`LIMIT :size OFFSET :page*size`) | System error → throw `INTERNAL_ERROR` |
| 2 | `[DB READ]` | Query `COUNT(*)` from `point_reward_batch` WHERE `user_id = :userId` for `totalCount` | System error → throw `INTERNAL_ERROR` |
| 3 | `[RETURN]` | Map to `PointsBatchListRs` and return | — |

---

## 4. Database

### Tables Read

| Table | Operation | Condition |
|-------|-----------|-----------|
| `point_reward_batch` | SELECT | `user_id = :userId` ORDER BY `expires_at ASC` with pagination |
| `point_reward_batch` | COUNT | `user_id = :userId` |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
