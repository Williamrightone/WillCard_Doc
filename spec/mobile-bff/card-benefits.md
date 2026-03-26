# Card Benefits — BFF Spec

> **Service:** bff
> **Related biz spec:** [card-service/card-benefits.md](../card-service/card-benefits.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the client's request to query currently active merchant benefits for a given card type.
Forwards the request to card-service, which queries `card_type_merchant_benefit` for all active entries matching the card type.
Returns a list of benefit records.

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
| Client URL | `/api/v1/card/benefits` |
| BFF Received Path | `/card/benefits` |
| Auth Required | Yes (Bearer JWT — Spring Security authenticated) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Parameters

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| cardType | String | ✅ | @NotBlank; enum: CLASSIC / OVERSEAS / PREMIUM / INFINITE | Card type to query benefits for |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| cardType | String | Card type queried |
| benefits | List\<BenefitItem\> | List of active benefits (may be empty) |

**BenefitItem:**

| Field | Type | Description |
|-------|------|-------------|
| benefitId | String | Snowflake ID (String) |
| merchantId | String | Specific merchant ID; null if applies to entire MCC |
| mccCode | String | MCC category code; null if applies to all MCCs |
| bonusRate | String | Additional reward rate (e.g., `0.0300` = 3%) |
| descriptionZh | String | Benefit description (Chinese) |
| descriptionEn | String | Benefit description (English) |
| effectiveFrom | String | Effective start date (`YYYY-MM-DD`) |
| effectiveTo | String | Effective end date (`YYYY-MM-DD`); null if indefinite |

---

## 4. BFF UseCase Flow

**UseCase:** `CardBenefitsUseCase.getBenefits(CardBenefitsRq)`
**UseCase Impl:** `CardBenefitsUseCaseImpl`
**Adapter:** `CardFeignAdapter` implements `CardClientPort`
**Feign:** `CardFeignClient` (uses `CardBenefitsRq` / `CardBenefitsRs` from `card-contract`)

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[VALIDATE]` | Validate `cardType` is a valid enum value | Validation failure → throw `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` calls card-service → [card-service/card-benefits.md](../card-service/card-benefits.md) | System error → throw `INTERNAL_ERROR` |
| 3 | `[RETURN]` | Relay `CardBenefitsRs` to client | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | `cardType` is blank or invalid enum |
| 401 | UNAUTHORIZED | Not authenticated | Missing or invalid JWT |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
