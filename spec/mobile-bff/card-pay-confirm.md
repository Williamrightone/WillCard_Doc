# Card Pay Confirm — BFF Spec

## Overview

Receives the member's OTP submission and forwards it to Mock NCCC Service for final
authorization verification. Returns `APPROVED` with the authorization code, transaction
ID, and settlement detail on success, or `DECLINED` with a reason code on failure.
This is the second and final step of the two-phase card payment Saga.

---

## Operation Log Level

**Level: L1**
**Trigger:** Every call, regardless of success or failure.
**Kafka Topic:** `operation-log.card.pay-confirm`

| OperationLogEvent Field | Value Source |
|------------------------|--------------|
| service | `bff` |
| action | `card.pay-confirm` |
| userId | Extracted from JWT (`X-User-Id` header) |
| userAccount | — |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | Error description on failure; null on success |
| errorCode | Error code on failure; null on success |
| beforeSnapshot | — |
| afterSnapshot | — |
| requestId | HTTP correlation ID |
| txnId | `CardPayConfirmRs.txnId` on `APPROVED`; null otherwise |

**Publish Triggers:**

| Scenario | result | failReason / errorCode |
|----------|--------|------------------------|
| JWT validation failed (Step 1) | FAIL | `UNAUTHORIZED` / — |
| Request validation failed (Step 2) | FAIL | `INVALID_REQUEST` / — |
| Session expired (Step 3) | FAIL | `SESSION_EXPIRED` / — |
| OTP failed — attempts remain (Step 3) | FAIL | `OTP_FAILED` / — |
| Card locked due to OTP threshold (Step 3) | FAIL | `CARD_LOCKED` / — |
| Payment approved (Step 3) | SUCCESS | — |

---

## API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | `/api/v1/card/pay/confirm` |
| BFF Received Path | `/card/pay/confirm` |
| Auth Required | Yes (JWT Bearer Token) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | `Bearer {accessToken}` |

### Request Body

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | OTP session reference returned from `/card/pay` |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 6-digit OTP received via SMS |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| status | String | `APPROVED` / `DECLINED` |
| authCode | String | Authorization code; present on `APPROVED` |
| txnId | String | Snowflake transaction ID; present on `APPROVED` |
| pointsUsed | Integer | Points actually deducted; present on `APPROVED` |
| cardAmount | BigDecimal | Final amount charged to card; present on `APPROVED` |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED`; present on `DECLINED` |
| attemptsRemaining | Integer | Remaining OTP attempts for this session; present when `declineReason = OTP_FAILED` |

---

## BFF UseCase Flow

**UseCase:** `CardPayConfirmUseCase.confirm(CardPayConfirmRq)`

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[VALIDATE]` | Validate JWT; extract `X-User-Id` from header | Invalid / missing token → 401 UNAUTHORIZED; publish L1 log |
| 2 | `[VALIDATE]` | Validate request body (`challengeRef`, `otp` format) | Validation failure → INVALID_REQUEST; publish L1 log |
| 3 | `[FEIGN]` | Call Mock NCCC Service → [spec/mock-nccc/pay-confirm.md](../../mock-nccc/pay-confirm.md) | Map downstream DECLINED responses to BFF error codes; publish L1 log |
| 4 | `[KAFKA]` | Publish `operation-log.card.pay-confirm` event (async) | — |
| 5 | `[RETURN]` | Return `{ status, authCode?, txnId?, pointsUsed?, cardAmount?, declineReason?, attemptsRemaining? }` | — |

---

## Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed or missing fields | `challengeRef` blank or `otp` not 6 digits |
| 401 | UNAUTHORIZED | Authentication failed | Invalid or missing JWT |
| 422 | SESSION_EXPIRED | OTP session has expired | Redis key `otp:{challengeRef}` not found (TTL 180s) |
| 422 | OTP_FAILED | OTP verification failed | Incorrect OTP; attemptsRemaining > 0 |
| 422 | CARD_LOCKED | Card temporarily locked due to repeated failures | Per-card OTP failure threshold (5 / 15 min) exceeded |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
