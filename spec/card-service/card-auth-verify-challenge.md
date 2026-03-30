# Card Auth Verify Challenge — card-service Spec

> **Service:** card-service
> **Contract Module:** card-service-contract
> **Caller:** [mock-nccc/pay-confirm.md](../mock-nccc/pay-confirm.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives `challengeRef` and `otp` from Mock NCCC, verifies the OTP against the Redis session, and returns the final authorization result.

On approval: accumulates the daily spending total, allocates a Snowflake `txnId`, and returns `{ result: APPROVED, txnId, cardType, maskedPan }` for ORCH to assemble the Kafka payload.

On failure: increments both the per-session attempt counter and the per-card cross-session fail counter. Card lock (≥ 5 failures in 15 min) takes priority over session void (≥ 3 failures in session).

> **No wallet calls. No `txn.card.authorized` Kafka.** Both actions are owned by the Transaction Orchestrator (Phase 7b).
> The only Kafka event published here is `card.risk.otp-threshold-exceeded` — a risk/security event triggered when the card is frozen due to excessive OTP failures.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (Mock NCCC) | [mock-nccc/pay-confirm.md](../mock-nccc/pay-confirm.md) |
| OTP Session written by | [card-service/card-auth-authorize.md](card-auth-authorize.md) |
| ORCH Leg 2 (consumes this response) | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |

---

## 3. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /card-service |
| Controller mapping | /auth/verify-challenge |
| Effective received path | /card-service/auth/verify-challenge |
| Caller | Mock NCCC (`CardServiceFeignClient`) |

### Request (`card.service.contract.dto.CardAuthVerifyChallengeRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | UUID from Leg 1; identifies the OTP session in Redis |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 6-digit OTP entered by user |

### Response (`card.service.contract.dto.CardAuthVerifyChallengeRs`)

| Field | Type | Description |
|-------|------|-------------|
| result | String | `APPROVED` or `DECLINED` |
| txnId | String | Snowflake txnId allocated by card-service (APPROVED only) |
| cardType | String | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` (APPROVED only; for ORCH's Kafka payload) |
| maskedPan | String | Masked PAN, e.g. `****1234` (APPROVED only; for ORCH's Kafka payload) |
| reason | String | Decline reason code (DECLINED only); see table below |
| remainingAttempts | Integer | OTP attempts remaining in this session (present only when reason = `OTP_FAILED`) |

**Decline reason values:**

| reason | Cause |
|--------|-------|
| `SESSION_EXPIRED` | OTP session not found in Redis (TTL 180 s expired, or was never created) |
| `OTP_FAILED` | Wrong OTP; session still active |
| `SESSION_VOIDED` | ≥ 3 wrong OTPs in this session; session voided |
| `CARD_LOCKED` | ≥ 5 wrong OTPs for this card within 15 min; card frozen |

---

## 4. Service UseCase Flow

**UseCase:** `CardAuthVerifyChallengeUseCase.verify(CardAuthVerifyChallengeRq)`

### Step 1 — Session Lookup

| Step | Type | Description |
|------|------|-------------|
| 1 | `[REDIS READ]` | GET `otp:{challengeRef}`; not found → **RETURN** `{ result: DECLINED, reason: SESSION_EXPIRED }` |
| 2 | `[DOMAIN]` | Compare `session.otpValue == rq.otp`; if match → **OTP Correct path** (Step 3); if mismatch → **OTP Wrong path** (Step 8) |

---

### OTP Correct Path

| Step | Type | Description |
|------|------|-------------|
| 3 | `[REDIS WRITE]` | `INCRBY card:daily:{session.cardId}:{yyyyMMdd} session.txnTwdBaseAmount`; if key is new (result = txnTwdBaseAmount) → set TTL to seconds remaining until 23:59:59 Asia/Taipei |
| 4 | `[DOMAIN]` | Allocate `txnId` (Snowflake) |
| 5 | `[DB READ]` | `SELECT card_type, pan_masked FROM card WHERE card_id = session.cardId` |
| 6 | `[REDIS WRITE]` | DEL `otp:{challengeRef}` |
| 7 | `[RETURN]` | `{ result: APPROVED, txnId, cardType: card.card_type, maskedPan: card.pan_masked }` |

---

### OTP Wrong Path

| Step | Type | Description |
|------|------|-------------|
| 8 | `[REDIS WRITE]` | `INCR otp:card:fail:{session.cardId}`; if result = 1 (new key) → set TTL 900 s; store as `cardFailCount` |
| 9 | `[REDIS WRITE]` | `INCR otp:attempt:{challengeRef}`; if result = 1 (new key) → set TTL 180 s; store as `attemptCount` |
| 10 | `[DOMAIN]` | If `cardFailCount ≥ 5` → **Card Lock sub-path** (Steps 11–13) |
| 11 | `[DB WRITE]` | `UPDATE card SET status = 'FROZEN', updated_at = NOW(3) WHERE card_id = session.cardId` |
| 12 | `[REDIS WRITE]` | DEL `otp:{challengeRef}` |
| 13 | `[KAFKA]` | Publish `card.risk.otp-threshold-exceeded` (see Kafka section); **RETURN** `{ result: DECLINED, reason: CARD_LOCKED }` |
| 14 | `[DOMAIN]` | If `attemptCount ≥ 3` → **Session Void sub-path**: DEL `otp:{challengeRef}`; **RETURN** `{ result: DECLINED, reason: SESSION_VOIDED }` |
| 15 | `[RETURN]` | `{ result: DECLINED, reason: OTP_FAILED, remainingAttempts: 3 - attemptCount }` |

> **Counter ordering (Steps 8–9):** Both counters are incremented before any condition check.
> This ensures `otp:card:fail` is always updated regardless of session state — preventing a voided session from hiding card-level failure accumulation.
>
> **Card lock takes priority over session void (Step 10 before Step 14):**
> On the 3rd wrong OTP that also triggers the 5th card-level failure, `CARD_LOCKED` is returned (more severe).
>
> **TTL behaviour:** `INCR` on an existing key preserves its TTL. Only set TTL when the key is newly created (result = 1).

---

## 5. Kafka Event — `card.risk.otp-threshold-exceeded`

**Topic:** `card.risk.otp-threshold-exceeded`
**Published by:** card-service (Step 13, Card Lock sub-path only)
**Event Class:** `card.service.event.CardRiskOtpThresholdExceededEvent`

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| cardId | Long | `session.cardId` | Frozen card ID |
| userId | Long | `session.userId` | Card owner member ID |
| lockedAt | String | System clock | ISO-8601 timestamp when card was frozen |

> This is a **risk/security event** — distinct from `txn.card.authorized` which is published by ORCH.
> Downstream consumers (e.g. notification, audit) may subscribe to this topic independently.

---

## 6. Redis Keys

### OTP Session (read & deleted here)

**Key:** `otp:{challengeRef}` — written by `card-auth-authorize`; see [card-auth-authorize.md](card-auth-authorize.md)

### Daily Spending Accumulator

| Field | Value |
|-------|-------|
| Key | `card:daily:{cardId}:{yyyyMMdd}` |
| Operation | `INCRBY txnTwdBaseAmount` (Step 3; APPROVED path only) |
| TTL | Seconds remaining until 23:59:59 Asia/Taipei on the current day |
| Read by | `card-auth-authorize` Step 8 (limit pre-check) |

### Per-Session OTP Attempt Counter

| Field | Value |
|-------|-------|
| Key | `otp:attempt:{challengeRef}` |
| Operation | `INCR` (Step 9; OTP Wrong path) |
| TTL | 900 s (same lifetime as OTP session) |
| Threshold | ≥ 3 → SESSION_VOIDED |

### Per-Card Cross-Session Fail Counter

| Field | Value |
|-------|-------|
| Key | `otp:card:fail:{cardId}` |
| Operation | `INCR` (Step 8; OTP Wrong path) |
| TTL | 900 s (15 min rolling window) |
| Threshold | ≥ 5 → CARD_LOCKED |

---

## 7. Database

| Table | Operation | Columns | Condition |
|-------|-----------|---------|-----------|
| `card` | READ (Step 5) | `card_type`, `pan_masked` | `card_id = session.cardId`; APPROVED path only |
| `card` | WRITE (Step 11) | `status = 'FROZEN'`, `updated_at` | `card_id = session.cardId`; Card Lock sub-path only |

---

## 8. Error Codes

All `DECLINED` results (including `SESSION_EXPIRED`) are returned as **HTTP 200** — they are business outcomes, not errors.

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 9. Changelog

### v1.0 — 2026-03 — Initial spec
