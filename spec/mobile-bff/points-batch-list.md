# Get Points Batch List — BFF Spec

> **Service:** bff
> **Biz spec:** [wallet-service/points-batch-list.md](../wallet-service/points-batch-list.md)
> **Last updated:** 2026-03

---

## 1. Overview

Receives the client's request to query the user's point reward batch list.
Forwards to wallet-service, which returns a paginated list of `point_reward_batch` records sorted by earliest expiry first.

> **Operation level:** L3 — read-only query; EFK log only, no Kafka event.

---

## 2. Operation Log Level

**Level: L3**
**Behavior:** EFK log only — no Kafka event, no Audit DB write.

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| Client URL | `/api/v1/wallet/points-batches` |
| BFF receives | `/wallet/points-batches` |
| Auth required | Yes (Bearer JWT — Spring Security) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | Integer | ❌ | 0 | 0-indexed page number |
| size | Integer | ❌ | 20 | Page size; maximum 50 |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| items | Array | Batch records for the current page |
| items[].batchId | Long | Snowflake batch ID |
| items[].issuedAmount | Long | Total points originally issued in this batch (TWD cents) |
| items[].remainingBalance | Long | Current available points in this batch (TWD cents) |
| items[].status | String | `PENDING / CONFIRMED / DEPLETED / EXPIRED` |
| items[].expiresAt | String | ISO-8601 expiry datetime |
| totalCount | Long | Total number of batches |
| page | Integer | Current page (0-indexed) |
| size | Integer | Page size used |

---

## 4. BFF UseCase Flow

**UseCase:** `PointsBatchListUseCase.list()`
**UseCase Impl:** `PointsBatchListUseCaseImpl`
**Adapter:** `WalletFeignAdapter` implements `WalletClientPort`
**Feign:** `WalletFeignClient` (uses `wallet-contract`'s `PointsBatchListRs`)

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[FEIGN]` | `WalletFeignAdapter` calls wallet-service → [wallet-service/points-batch-list.md](../wallet-service/points-batch-list.md) | System error → throw `INTERNAL_ERROR` |
| 2 | `[RETURN]` | Relay `PointsBatchListRs` to client | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 401 | UNAUTHORIZED | Not authenticated | JWT missing or invalid |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
