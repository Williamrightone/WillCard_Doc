# Card Freeze — BFF Spec

> **Service:** bff
> **Related biz spec:** [card-service/card-freeze.md](../card-service/card-freeze.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the client's request to freeze a card.
Forwards the request to card-service, which validates the card status and transitions it from `ACTIVE` to `FROZEN`.
Once frozen, the card cannot be used for new authorizations.

---

## 2. Operation Log Level

**Level: L2**
**Trigger:** Every call, regardless of success or failure.
**Kafka Topic:** `operation-log.card.freeze`

| OperationLogEvent Field | Value Source |
|------------------------|--------------|
| service | `bff` |
| action | `card.freeze` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | Error description on failure; null on success |
| errorCode | Error code on failure; null on success |
| beforeSnapshot | `{ "cardId": "...", "status": "ACTIVE" }` (SUCCESS only) |
| afterSnapshot | `{ "cardId": "...", "status": "FROZEN" }` (SUCCESS only) |
| requestId | HTTP correlation ID |
| txnId | — |

**Publish Triggers:**

| Scenario | result | failReason / errorCode |
|----------|--------|------------------------|
| Validation failure (Step 1) | FAIL | `INVALID_REQUEST` / — |
| card-service error (Step 2) | FAIL | From card-service errorCode |
| Freeze success (Step 3) | SUCCESS | — |

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | `/api/v1/card/freeze` |
| BFF Received Path | `/card/freeze` |
| Auth Required | Yes (Bearer JWT — Spring Security authenticated) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Target card Snowflake ID |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| cardId | String | Card Snowflake ID (String) |
| status | String | `FROZEN` |

---

## 4. BFF UseCase Flow

**UseCase:** `CardFreezeUseCase.freeze(CardFreezeRq)`
**UseCase Impl:** `CardFreezeUseCaseImpl`
**Adapter:** `CardFeignAdapter` implements `CardClientPort`
**Feign:** `CardFeignClient` (uses `CardFreezeRq` / `CardStatusRs` from `card-contract`)

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[VALIDATE]` | Validate `cardId` is not blank | Validation failure → publish FAIL event (async) → throw `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` calls card-service → [card-service/card-freeze.md](../card-service/card-freeze.md) | Business error → proceed to Step 3; system error → publish FAIL event (async) → throw `INTERNAL_ERROR` |
| 3 | `[KAFKA]` | **[On failure]** Publish FAIL event (errorCode from card-service, async) → rethrow mapped BFF error code | Log error; do not suppress original exception |
| 4 | `[KAFKA]` | **[On success]** Publish SUCCESS event with `beforeSnapshot` and `afterSnapshot` (async) | Log error; do not throw exception |
| 5 | `[RETURN]` | Return `CardStatusRs` (cardId, status = `FROZEN`) | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | `cardId` is blank |
| 401 | UNAUTHORIZED | Not authenticated | Missing or invalid JWT |
| 404 | CARD_NOT_FOUND | Card does not exist or does not belong to user | card-service returned CA00001 |
| 409 | CARD_NOT_ACTIVE | Card is not in ACTIVE status | card-service returned CA00002 |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
