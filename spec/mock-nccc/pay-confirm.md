# Pay Confirm — Mock NCCC Spec

> **Service:** mock-nccc
> **Environment:** Development only
> **Contract Module:** mock-nccc-contract
> **Caller:** [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Simulates NCCC card network behavior for the OTP confirmation leg.

Receives `challengeRef` and `otp` from ORCH, forwards to card-service auth/verify-challenge, and passes the full result (including `cardType` and `maskedPan` for ORCH's Kafka payload) back to ORCH.

This service acts as a pure pass-through for the confirm leg — no encryption or card data lookup is needed.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (ORCH) | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| Mock NCCC → card-service (OTP verification) | [card-service/card-auth-verify-challenge.md](../card-service/card-auth-verify-challenge.md) |
| Leg 1 | [mock-nccc/pay.md](pay.md) |

---

## 3. API Definition — mock-nccc

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /mock-nccc |
| Controller mapping | /pay/confirm |
| Effective received path | /mock-nccc/pay/confirm |
| Caller | ORCH (MockNcccFeignClient) |

### Request (`mock-nccc.contract.dto.MockNcccPayConfirmRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | UUID from Leg 1; identifies the OTP session in card-service |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 6-digit OTP entered by user |

### Response (`mock-nccc.contract.dto.MockNcccPayConfirmRs`)

| Field | Type | Description |
|-------|------|-------------|
| result | String | `APPROVED` or `DECLINED` |
| txnId | String | Snowflake txnId assigned by card-service (APPROVED only) |
| cardType | String | Card type: `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` (APPROVED only; for ORCH Kafka payload) |
| maskedPan | String | Masked PAN, e.g. `****1234` (APPROVED only; for ORCH Kafka payload) |
| reason | String | Decline reason code (DECLINED only); see table below |
| remainingAttempts | Integer | OTP attempts remaining (present only when reason = `OTP_FAILED`) |

**Decline reason values (defined in card-service verify-challenge):**

| reason | Cause |
|--------|-------|
| `SESSION_EXPIRED` | Redis OTP session not found (TTL 180s expired) |
| `OTP_FAILED` | Wrong OTP; session still valid |
| `SESSION_VOIDED` | ≥ 3 wrong OTPs per session; session voided |
| `CARD_LOCKED` | ≥ 5 wrong OTPs per card within 15 min; card FROZEN |

---

## 4. Service UseCase Flow

**UseCase:** `MockNcccPayConfirmUseCase.confirm(MockNcccPayConfirmRq)` (mock-nccc)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[FEIGN]` | `CardServiceFeignClient.verifyChallenge(CardAuthVerifyChallengeRq)` → `POST /card-service/auth/verify-challenge`; request: `{ challengeRef, otp }`; response: `{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }`; see [card-service/card-auth-verify-challenge.md](../card-service/card-auth-verify-challenge.md) |
| 2 | `[RETURN]` | Return `MockNcccPayConfirmRs { result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }` |

> All `APPROVED` / `DECLINED` outcomes are returned as **HTTP 200** — they are business results passed through from card-service.

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

> OTP-related results (`SESSION_EXPIRED`, `OTP_FAILED`, `SESSION_VOIDED`, `CARD_LOCKED`) are returned as **HTTP 200** with `result = DECLINED`.

---

## 6. Changelog

### v1.0 — 2026-03 — Initial spec
