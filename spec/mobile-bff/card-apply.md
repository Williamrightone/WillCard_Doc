# Card Apply — BFF Spec

> **Service:** bff
> **Related biz spec:** [card-service/card-apply.md](../card-service/card-apply.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the client's virtual card application request.
Validates the request and forwards it to card-service for card issuance.
On success, returns the new card details including the **one-time CVV**.
The BFF does not generate card numbers or handle encryption — it is responsible only for flow coordination, L1 operation log, and response assembly.

> ⚠️ **CVV Notice:** The CVV is returned **once only** in this response. It is never stored anywhere and cannot be retrieved again. The client must display it immediately to the user and instruct them to record it securely.

> **No Idempotency-Key:** This endpoint does not use the standard `Idempotency-Key` mechanism, because caching a response containing CVV in Redis would violate PCI DSS (Requirement 3.2.1). Duplicate applications are prevented by server-side uniqueness validation. Clients must treat `CARD_ALREADY_EXISTS` as a terminal failure.

---

## 2. Operation Log Level

**Level: L1**
**Trigger:** Every call, regardless of success or failure.
**Kafka Topic:** `operation-log.card.apply`

| OperationLogEvent Field | Value Source |
|------------------------|--------------|
| service | `bff` |
| action | `card.apply` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | Error description on failure; null on success |
| errorCode | Error code on failure; null on success |
| beforeSnapshot | — |
| afterSnapshot | `{ "cardId": "...", "cardType": "CLASSIC", "maskedPan": "****-****-****-1234", "status": "ACTIVE" }` (SUCCESS only; CVV excluded per PCI DSS) |
| requestId | HTTP correlation ID |
| txnId | — |

**Publish Triggers:**

| Scenario | result | failReason / errorCode |
|----------|--------|------------------------|
| Validation failure (Step 1) | FAIL | `INVALID_REQUEST` / — |
| card-service returns CARD_ALREADY_EXISTS (Step 2) | FAIL | `CARD_ALREADY_EXISTS` / CA00009 |
| card-service returns other error (Step 2) | FAIL | From card-service errorCode |
| Apply success (Step 3) | SUCCESS | — |

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | `/api/v1/card/apply` |
| BFF Received Path | `/card/apply` |
| Auth Required | Yes (Bearer JWT — Spring Security authenticated) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardType | String | ✅ | @NotBlank; enum: CLASSIC / OVERSEAS / PREMIUM / INFINITE | Card type to apply for |
| cardNetwork | String | ✅ | @NotBlank; enum: VISA / MASTERCARD / JCB | Card network |
| cardholderNameEn | String | ✅ | @NotBlank; @Size(max=26) | Cardholder English name (printed on card face); pre-fill from member profile recommended |
| cardholderNameZh | String | ✅ | @NotBlank; @Size(max=50) | Cardholder Chinese name; pre-fill from member profile recommended |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| cardId | String | Snowflake ID (serialized as String to prevent JavaScript precision loss) |
| maskedPan | String | Masked card number (`****-****-****-1234`) |
| cardType | String | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| cardNetwork | String | VISA / MASTERCARD / JCB |
| expiryDate | String | Expiry date in `MM/YY` format |
| cardholderNameEn | String | Cardholder English name |
| cvv | String | 3-digit CVV — **one-time only; cannot be retrieved after this response** |
| status | String | `ACTIVE` |
| activatedAt | String | Activation timestamp (ISO-8601) |

---

## 4. BFF UseCase Flow

**UseCase:** `CardApplyUseCase.apply(CardApplyRq)`
**UseCase Impl:** `CardApplyUseCaseImpl`
**Adapter:** `CardFeignAdapter` implements `CardClientPort`
**Feign:** `CardFeignClient` (uses `CardApplyRq` / `CardApplyRs` from `card-contract`)

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[VALIDATE]` | Validate cardType and cardNetwork are valid enum values; validate cardholder name fields are not blank | Validation failure → publish FAIL event (async) → throw `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` calls card-service → [card-service/card-apply.md](../card-service/card-apply.md) | Business error → proceed to Step 3; system error → publish FAIL event (async) → throw `INTERNAL_ERROR` |
| 3 | `[KAFKA]` | **[On failure]** Publish FAIL event (errorCode from card-service, async) → rethrow mapped BFF error code | Log error; do not suppress original exception |
| 4 | `[KAFKA]` | **[On success]** Publish SUCCESS event with `afterSnapshot` (CVV excluded per PCI DSS, async) | Log error; do not throw exception |
| 5 | `[RETURN]` | Assemble BFF `CardApplyRs` from card-contract `CardApplyRs` and return | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | cardType / cardNetwork invalid enum; cardholder name blank |
| 401 | UNAUTHORIZED | Not authenticated | Missing or invalid JWT |
| 409 | CARD_ALREADY_EXISTS | Active card already exists | card-service returned CA00009 |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
