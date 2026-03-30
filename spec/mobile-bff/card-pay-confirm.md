# Card Pay Confirm — BFF Spec

> **Service:** bff
> **Related biz spec:** [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the OTP confirmation from the App (**Leg 2** of the transaction flow).
Publishes an L1 operation log, then forwards the OTP to Mock NCCC via Feign.
Returns the final transaction result (`APPROVED` or `DECLINED`) to the App.

> `challengeRef` is established in [Leg 1 — card-pay](card-pay.md). The OTP session expires after **180 seconds**; a submission after expiry returns `DECLINED + SESSION_EXPIRED`.

---

## 2. Connected Specs — Cross-Spec Data Contract

This endpoint is **Leg 2** of the transaction flow. The following field contract must be consistent with Leg 1 and downstream specs:

| Interface | Spec |
|-----------|------|
| BFF → Transaction Orchestrator request / response | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| ORCH → Mock NCCC request / response | [mock-nccc/pay-confirm.md](../mock-nccc/pay-confirm.md) |
| ORCH → Wallet Service (confirmDeduct / release) | [wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) |
| Mock NCCC → card-service auth/verify-challenge | [card-service/card-auth-verify-challenge.md](../card-service/card-auth-verify-challenge.md) |
| Leg 1 (establishes challengeRef) | [mobile-bff/card-pay.md](card-pay.md) |

**Shared field contract (pinned across all above specs):**

| Field | Type | Direction | Notes |
|-------|------|-----------|-------|
| `challengeRef` | String | App → BFF → ORCH → MockNCCC → CS | UUID produced at Leg 1; identifies OTP session |
| `otp` | String | App → BFF → ORCH → MockNCCC → CS | 6-digit OTP entered by user |
| `result` | String | CS → MockNCCC → ORCH → BFF → App | `APPROVED` or `DECLINED` |
| `txnId` | String | CS → MockNCCC → ORCH → BFF → App | Snowflake txnId; assigned by card-service on APPROVED |
| `reason` | String | CS → MockNCCC → ORCH → BFF → App | Decline reason; present only on `DECLINED` |

**Decline reason values (defined in card-service verify-challenge):**

| reason | Cause |
|--------|-------|
| `SESSION_EXPIRED` | Redis OTP session not found (TTL expired) |
| `OTP_FAILED` | Wrong OTP; session still valid; `remainingAttempts` field included |
| `SESSION_VOIDED` | ≥ 3 wrong OTPs per session; reservation released |
| `CARD_LOCKED` | ≥ 5 wrong OTPs per card within 15 min; card FROZEN; reservation released |

---

## 3. Operation Log Level

**Level: L1**
**Trigger:** Every call, regardless of success or failure.
**Kafka Topic:** `operation-log.card.pay-confirm`

| OperationLogEvent Field | Value Source |
|------------------------|--------------|
| service | `bff` |
| action | `card.pay-confirm` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | Client IP |
| result | `SUCCESS` (APPROVED) / `FAIL` (DECLINED or error) |
| failReason | Decline reason or error description; null on APPROVED |
| errorCode | — |
| beforeSnapshot | — |
| afterSnapshot | `{ "txnId": "...", "challengeRef": "..." }` (APPROVED only) |
| requestId | HTTP correlation ID |
| txnId | Snowflake txnId from verify-challenge (APPROVED only) |

**Publish Triggers:**

| Scenario | result | failReason |
|----------|--------|------------|
| Validation failure (Step 1) | FAIL | `INVALID_REQUEST` |
| DECLINED + any reason (Step 4) | FAIL | `SESSION_EXPIRED` / `OTP_FAILED` / `SESSION_VOIDED` / `CARD_LOCKED` |
| APPROVED (Step 4) | SUCCESS | — |

---

## 4. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | `/api/v1/card/pay/confirm` |
| BFF Received Path | `/card/pay/confirm` |
| Auth Required | Yes (Bearer JWT) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body (`bff.contract.dto.CardPayConfirmBffRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | UUID from Leg 1 response |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 6-digit OTP |

### Response Body (`bff.contract.dto.CardPayConfirmBffRs`)

| Field | Type | Description |
|-------|------|-------------|
| result | String | `APPROVED` or `DECLINED` |
| txnId | String | Snowflake txnId (present only when result = `APPROVED`) |
| reason | String | Decline reason code (present only when result = `DECLINED`); see reason table in Section 2 |
| remainingAttempts | Integer | OTP attempts remaining (present only when reason = `OTP_FAILED`) |

---

## 5. BFF UseCase Flow

**UseCase:** `CardPayConfirmUseCase.confirm(CardPayConfirmBffRq)` (bff)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[VALIDATE]` | Validate request (@Valid); on failure → throw 400 INVALID_REQUEST |
| 2 | `[HEADER]` | Extract `X-Member-Id` from Security Context |
| 3 | `[KAFKA]` | Publish `operation-log.card.pay-confirm` (L1) — result determined after Step 4 |
| 4 | `[FEIGN]` | `TxnOrchFeignClient.confirm(CardPayConfirmOrchRq)` → `POST /txn-orch/card/pay/confirm`; pass `X-Member-Id` header; see [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| 5 | `[RETURN]` | Return `CardPayConfirmBffRs { result, txnId?, reason?, remainingAttempts? }` |

> `DECLINED` responses from Feign are treated as successful BFF calls (HTTP 200). The decline reason is surfaced in the response body, not as an HTTP error.
> If Step 4 throws a system error (non-business failure), re-publish operation-log with `FAIL` before propagating.

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | — | INVALID_REQUEST | Request validation failure |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

> OTP-related declines (`SESSION_EXPIRED`, `OTP_FAILED`, `SESSION_VOIDED`, `CARD_LOCKED`) are returned as **HTTP 200** with `result = DECLINED` — they are business outcomes, not errors.

---

## 7. Changelog

### v1.0 — 2026-03 — Initial spec
