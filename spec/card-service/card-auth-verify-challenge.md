# Card Auth Verify Challenge — Card Service Biz Spec

## Overview

Issuer Authorization API — Phase 2. Called directly by NCCC (production) or Mock NCCC
(development) after the member submits the OTP. Loads the pending authorization session
from Redis, verifies the OTP, and either confirms the points deduction via Wallet Service
and publishes the `txn.card.authorized` Kafka event on approval, or handles OTP failure
with dual-layer counter logic. A failed session triggers a points reservation release
(Saga compensation).

---

## API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/auth/verify-challenge` |
| Effective received path | `/card/auth/verify-challenge` |
| Caller | NCCC (production mTLS) / Mock NCCC Service (development Feign) |

### Request (CardAuthVerifyChallengeRq — card-contract)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | OTP session reference from Phase 1 |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 6-digit OTP submitted by cardholder |

### Response (CardAuthVerifyChallengeRs — card-contract)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| status | String | `APPROVED` / `DECLINED` | |
| authCode | String | Authorization code (Snowflake-based) | Present on `APPROVED` |
| txnId | String | Snowflake transaction ID | Present on `APPROVED` |
| pointsUsed | Integer | Points actually deducted | Present on `APPROVED` |
| cardAmount | BigDecimal | Final amount charged to card (TWD) | Present on `APPROVED` |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED` | Present on `DECLINED` |
| attemptsRemaining | Integer | Remaining OTP attempts for this session | Present when `declineReason = OTP_FAILED` and session still active |

---

## Service UseCase Flow

**UseCase:** `CardAuthVerifyChallengeUseCase.verify(CardAuthVerifyChallengeRq)`

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[REDIS READ]` | Load `otp:{challengeRef}` → pending auth payload; key absent → `DECLINED + SESSION_EXPIRED` | `PendingAuthStore.load()` |
| 2 | `[DOMAIN]` | Compare submitted `otp` against stored OTP value | `OtpService.verify()` |
| **— OTP VALID path —** | | | |
| 3a | `[FEIGN]` | Call Wallet Service `confirmDeduct(reservationId)` → deduct points → [spec/wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) | — |
| 4a | `[DOMAIN]` | Assign `txnId` (Snowflake); generate `authCode` | `SnowflakeIdGenerator.next()` |
| 5a | `[REDIS WRITE]` | Increment `card:daily:{cardId}:{yyyyMMdd}` by `cardAmount` (in cents); set TTL to end of calendar day if key is new | `DailyLimitService.increment()` |
| 6a | `[KAFKA]` | Publish `txn.card.authorized` to Kafka (payload includes `txnId`, `messageKey`, `cardId`, `userId`, `amountTwd`, `fxFee?`, `fxRate?`, `currency`, `merchantId`, `merchantMCC`, `pointsUsed`, `cardAmount`) | — |
| 7a | `[REDIS WRITE]` | Delete `otp:{challengeRef}` | `PendingAuthStore.clear()` |
| 8a | `[RETURN]` | Return `APPROVED` + `authCode`, `txnId`, `pointsUsed`, `cardAmount` | — |
| **— OTP INVALID path —** | | | |
| 3b | `[REDIS WRITE]` | Increment `otp:attempt:{challengeRef}` (TTL inherits session; max 3) | `OtpAttemptTracker.increment()` |
| 4b | `[REDIS WRITE]` | Increment `otp:card:fail:{cardId}` (TTL 900s; max 5) | `OtpAttemptTracker.incrementCardFail()` |
| 5b | `[DOMAIN]` | Check per-session threshold: `otp:attempt:{challengeRef} >= 3` → session cancelled | `OtpAttemptTracker.isSessionExhausted()` |
| 6b (session exhausted) | `[FEIGN]` | Compensation: call Wallet Service `release(reservationId)` → [spec/wallet-service/release.md](../wallet-service/release.md) | — |
| 7b (session exhausted) | `[REDIS WRITE]` | Delete `otp:{challengeRef}`; delete `otp:attempt:{challengeRef}` | — |
| 8b (session exhausted) | `[RETURN]` | Return `DECLINED + OTP_FAILED` (attemptsRemaining = 0) | — |
| 5c | `[DOMAIN]` | Check per-card threshold: `otp:card:fail:{cardId} >= 5` → card temporarily locked | `OtpAttemptTracker.isCardLocked()` |
| 6c (card locked) | `[DB WRITE]` | Update `card.status = LOCKED` (temporary risk lock) | `CardStatusService.lock()` |
| 7c (card locked) | `[FEIGN]` | Compensation: call Wallet Service `release(reservationId)` | — |
| 8c (card locked) | `[REDIS WRITE]` | Delete `otp:{challengeRef}` | — |
| 9c (card locked) | `[KAFKA]` | Publish `card.risk.otp-threshold-exceeded` (payload: `cardId`, `userId`, `failCount`, `windowSeconds=900`) | — |
| 10c (card locked) | `[RETURN]` | Return `DECLINED + CARD_LOCKED` | — |
| 5d (normal failure) | `[RETURN]` | Return `DECLINED + OTP_FAILED` + `attemptsRemaining = 3 - attempt_count` | — |

> **Lazy compensation on TTL expiry:** When `otp:{challengeRef}` expires naturally (TTL 180s) before Phase 2 is called, the Wallet Service reservation is released by a scheduled cleanup job or the next authorization attempt. The release TTL aligns with the OTP TTL (180s).

---

## Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card` | READ / WRITE | Read `user_id`, `card_type`; write `status = LOCKED` on card lock |

### Redis

| Key Pattern | Operation | TTL | Description |
|-------------|-----------|-----|-------------|
| `otp:{challengeRef}` | READ / DELETE | 180s (written by authorize) | Pending authorization session |
| `otp:attempt:{challengeRef}` | READ / WRITE | 180s | Per-session OTP failure counter (max 3) |
| `otp:card:fail:{cardId}` | READ / WRITE | 900s | Per-card OTP failure counter within 15 min window (max 5) |
| `card:daily:{cardId}:{yyyyMMdd}` | WRITE | Until end of day | Increment today's cumulative amount on APPROVED |

### Table Schema

#### card (relevant columns)

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| card_id | BIGINT | NO | PK, Snowflake |
| user_id | BIGINT | NO | FK to member |
| card_type | VARCHAR(20) | NO | Card type |
| status | VARCHAR(20) | NO | Updated to `LOCKED` on card lock |
| updated_at | DATETIME(3) | NO | Updated on status change |

---

## Kafka Events Published

| Topic | Trigger | Key Payload Fields |
|-------|---------|-------------------|
| `txn.card.authorized` | OTP approved | `txnId`, `messageKey` (Snowflake for Ledger idempotency), `cardId`, `userId`, `amountTwd`, `fxFee`, `fxRate`, `currency`, `merchantId`, `merchantMCC`, `pointsUsed`, `cardAmount`, `maskedPan` |
| `card.risk.otp-threshold-exceeded` | Per-card threshold hit | `cardId`, `userId`, `failCount`, `windowSeconds` |

> **PCI DSS:** `maskedPan` in Kafka payloads must use the masked format `****-****-****-1234`. CVV must never appear in any event payload.

---

## Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 422 | `SESSION_EXPIRED` | OTP session not found or expired | `otp:{challengeRef}` key absent |
| 422 | `OTP_FAILED` | Incorrect OTP submitted | OTP mismatch; session still active |
| 422 | `CARD_LOCKED` | Card temporarily locked | Per-card threshold (5 failures / 15 min) exceeded |
| 500 | `INTERNAL_ERROR` | Unexpected system error | Unhandled exception |
