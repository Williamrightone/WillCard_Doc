# Card Pay Confirm — Transaction Orchestrator Spec

> **Service:** txn-orch
> **Contract Module:** txn-orch-contract
> **Caller:** [mobile-bff/card-pay-confirm.md](../mobile-bff/card-pay-confirm.md)
> **Last Updated:** 2026-03

---

## 1. Overview

**Leg 2** of the card payment Saga. Reads the Saga State stored by Leg 1, forwards the OTP to Mock NCCC (which calls card-service auth/verify-challenge), and completes the Saga:

- **APPROVED**: confirms wallet reservation, publishes `txn.card.authorized` to Kafka, deletes Saga State.
- **DECLINED + SESSION_VOIDED / CARD_LOCKED**: releases wallet reservation, deletes Saga State.
- **DECLINED + OTP_FAILED**: no wallet action, Saga State retained (session still valid).

Returns the final transaction result to BFF.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (BFF) | [mobile-bff/card-pay-confirm.md](../mobile-bff/card-pay-confirm.md) |
| ORCH → Mock NCCC (OTP verification) | [mock-nccc/pay-confirm.md](../mock-nccc/pay-confirm.md) |
| ORCH → Wallet Service (confirmDeduct / release) | [wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) |
| Leg 1 (establishes Saga State + challengeRef) | [txn-orch/card-pay.md](card-pay.md) |

---

## 3. API Definition — txn-orch

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /txn-orch |
| Controller mapping | /card/pay/confirm |
| Effective received path | /txn-orch/card/pay/confirm |
| Caller | BFF (TxnOrchFeignClient) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| X-Member-Id | ✅ | Authenticated user Snowflake ID (forwarded by BFF) |

### Request (`txn-orch.contract.dto.CardPayConfirmOrchRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | UUID from Leg 1; identifies the OTP session |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 6-digit OTP entered by user |

### Response (`txn-orch.contract.dto.CardPayConfirmOrchRs`)

| Field | Type | Description |
|-------|------|-------------|
| result | String | `APPROVED` or `DECLINED` |
| txnId | String | Snowflake txnId assigned by card-service (APPROVED only) |
| reason | String | Decline reason code (DECLINED only); see table below |
| remainingAttempts | Integer | OTP attempts remaining (present only when reason = `OTP_FAILED`) |

**Decline reason values (defined in card-service verify-challenge):**

| reason | Cause |
|--------|-------|
| `SESSION_EXPIRED` | Saga State not found in Redis (TTL expired before confirm call) |
| `OTP_FAILED` | Wrong OTP; card-service session still valid; `remainingAttempts` included |
| `SESSION_VOIDED` | ≥ 3 wrong OTPs per session; card-service session voided; reservation released |
| `CARD_LOCKED` | ≥ 5 wrong OTPs per card within 15 min; card FROZEN; reservation released |

---

## 4. Service UseCase Flow

**UseCase:** `CardPayConfirmOrchUseCase.confirm(CardPayConfirmOrchRq)` (txn-orch)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[REDIS READ]` | GET `saga:{challengeRef}`; not found → immediately return `{ result: DECLINED, reason: SESSION_EXPIRED }` |
| 2 | `[FEIGN]` | `MockNcccFeignClient.confirm(MockNcccPayConfirmRq)` → `POST /mock-nccc/pay/confirm`; request: `{ challengeRef, otp }`; response: `{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }`; see [mock-nccc/pay-confirm.md](../mock-nccc/pay-confirm.md) |

**If `result = APPROVED`:**

| Step | Type | Description |
|------|------|-------------|
| 3 | `[FEIGN]` | If `sagaState.reservationId != null`: `WalletFeignClient.confirmDeduct(reservationId)` — see [wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) |
| 4 | `[KAFKA]` | Publish `txn.card.authorized` (see Kafka Event section below) |
| 5 | `[REDIS WRITE]` | DEL `saga:{challengeRef}` |
| 6 | `[RETURN]` | Return `{ result: APPROVED, txnId }` |

**If `result = DECLINED` + reason = `SESSION_VOIDED` or `CARD_LOCKED`:**

| Step | Type | Description |
|------|------|-------------|
| 7 | `[FEIGN]` | If `sagaState.reservationId != null`: `WalletFeignClient.release(reservationId)` — see [wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) |
| 8 | `[REDIS WRITE]` | DEL `saga:{challengeRef}` |
| 9 | `[RETURN]` | Return `{ result: DECLINED, reason }` |

**If `result = DECLINED` + reason = `OTP_FAILED`:**

| Step | Type | Description |
|------|------|-------------|
| 10 | `[RETURN]` | Return `{ result: DECLINED, reason: OTP_FAILED, remainingAttempts }` — do NOT release reservation; Saga State retained; session still valid |

> All `DECLINED` outcomes are returned as **HTTP 200** — they are business results, not errors.

---

## 5. Kafka Event — `txn.card.authorized`

**Topic:** `txn.card.authorized`
**Published by:** ORCH (Step 4, APPROVED path only)
**Event Class:** `txn-orch.event.TxnCardAuthorizedEvent`

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| txnId | String | verify-challenge response | Snowflake txnId assigned by card-service |
| userId | Long | Saga State | Authenticated user Snowflake ID |
| cardId | Long | Saga State | Card Snowflake ID |
| cardType | String | verify-challenge response | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` |
| maskedPan | String | verify-challenge response | Masked PAN (e.g. `****1234`) for display/audit |
| merchantId | String | Saga State | Merchant identifier |
| txnAmount | Long | Saga State | Original transaction amount in cents |
| txnCurrency | String | Saga State | ISO 4217 currency code |
| txnTwdBaseAmount | Long | Saga State | FX-converted TWD base amount |
| isOverseas | Boolean | Saga State | Whether this is an overseas transaction |
| pointsToUse | Long | Saga State | Points deducted; `0` if none |
| txnTimestamp | String | ORCH publish time | ISO-8601 timestamp (e.g. `2026-03-31T10:00:00.000+08:00`) |

---

## 6. Saga State (Redis)

**Key:** `saga:{challengeRef}` — written by [txn-orch/card-pay.md](card-pay.md)
**Lifecycle:**

| Event | Action |
|-------|--------|
| Leg 2 — APPROVED | DEL after wallet confirmDeduct + Kafka publish (Step 5) |
| Leg 2 — SESSION_VOIDED / CARD_LOCKED | DEL after wallet release (Step 8) |
| Leg 2 — OTP_FAILED | Retained (session and reservation still active) |
| TTL expires (180s) | Auto-deleted by Redis |

> **Orphaned reservation risk:** When Saga State expires via TTL, any `wallet_reservation` created in Leg 1 remains `PENDING` indefinitely — the `reservationId` is lost along with the Saga State and cannot be recovered from ORCH. A scheduled cleanup job in wallet-service must periodically release PENDING reservations older than 10 minutes (see `reserve.md` § Scheduled Cleanup).

---

## 7. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

> `DECLINED` outcomes (`SESSION_EXPIRED`, `OTP_FAILED`, `SESSION_VOIDED`, `CARD_LOCKED`) are returned as **HTTP 200** — they are business results, not errors.

---

## 8. Changelog

### v1.1 — 2026-03 — Add orphaned reservation note
- Added design note: TTL-expired Saga State leaves wallet reservation in PENDING; requires wallet-service scheduled cleanup.

### v1.0 — 2026-03 — Initial spec
