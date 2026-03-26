# Card Info — BFF Spec

> **Service:** bff
> **Related biz spec:** [card-service/card-info.md](../card-service/card-info.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the client's request to query card details by `cardId`.
Forwards the request to card-service, which decrypts expiry date and cardholder names, then assembles and returns the card record.
The BFF performs no decryption — it is responsible only for flow coordination and response relay.
`pan_encrypted` is never returned to the client.

> **L3 Operation:** This is a read-only query. No Kafka event is published. EFK logging only.

---

## 2. Operation Log Level

**Level: L3**
**Behavior:** EFK logging only — no Kafka event, no Audit DB write.

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| Client URL | `/api/v1/card/info` |
| BFF Received Path | `/card/info` |
| Auth Required | Yes (Bearer JWT — Spring Security authenticated) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Parameters

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Target card Snowflake ID |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| cardId | String | Snowflake ID (String) |
| maskedPan | String | `****-****-****-1234` |
| cardType | String | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| cardNetwork | String | VISA / MASTERCARD / JCB |
| expiryDate | String | `MM/YY` format (decrypted) |
| cardholderNameZh | String | Cardholder Chinese name (decrypted) |
| cardholderNameEn | String | Cardholder English name (decrypted) |
| status | String | ACTIVE / FROZEN / CANCELLED |
| activatedAt | String | ISO-8601 timestamp; null if not yet activated |

---

## 4. BFF UseCase Flow

**UseCase:** `CardInfoUseCase.getInfo(CardInfoRq)`
**UseCase Impl:** `CardInfoUseCaseImpl`
**Adapter:** `CardFeignAdapter` implements `CardClientPort`
**Feign:** `CardFeignClient` (uses `CardInfoRq` / `CardInfoRs` from `card-contract`)

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[VALIDATE]` | Validate `cardId` is not blank | Validation failure → throw `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` calls card-service → [card-service/card-info.md](../card-service/card-info.md) | `CARD_NOT_FOUND` → throw `CARD_NOT_FOUND`; system error → throw `INTERNAL_ERROR` |
| 3 | `[RETURN]` | Relay `CardInfoRs` from card-contract to client | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | `cardId` is blank |
| 401 | UNAUTHORIZED | Not authenticated | Missing or invalid JWT |
| 404 | CARD_NOT_FOUND | Card does not exist or does not belong to user | card-service returned CA00001 |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
