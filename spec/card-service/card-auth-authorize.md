# Card Auth Authorize — Card Service Biz Spec

## Overview

Issuer Authorization API — Phase 1. Called directly by NCCC (production) or Mock NCCC
(development). Decrypts the incoming encrypted card data in memory, identifies the
cardholder, validates the card status and transaction limits, calculates the
points-offset amount, reserves points tentatively via Wallet Service, locks the FX rate
for non-TWD transactions, generates a 6-digit OTP, and stores a pending authorization
session in Redis. Returns `CHALLENGE_REQUIRED` with a `challengeRef` and `pointsPreview`
on success, or `DECLINED` with a reason code on failure.

---

## API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/auth/authorize` |
| Effective received path | `/card/auth/authorize` |
| Caller | NCCC (production mTLS) / Mock NCCC Service (development Feign) |

### Request (CardAuthAuthorizeRq — card-contract)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| encryptedCard | String | ✅ | @NotBlank | Card data (PAN + expiry + CVV) encrypted by NCCC using combineKey (AES-256 GCM) |
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | Transaction amount in original currency |
| currency | String | ✅ | @NotBlank, @Size(min=3,max=3) | ISO 4217 currency code, e.g. `TWD`, `USD` |
| merchantId | String | ✅ | @NotBlank | Merchant identifier |
| merchantMCC | String | ✅ | @NotBlank | Merchant Category Code — used for reward rate lookup |

### Response (CardAuthAuthorizeRs — card-contract)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| status | String | `CHALLENGE_REQUIRED` / `DECLINED` | |
| challengeRef | String | OTP session identifier (UUID) | Present when `CHALLENGE_REQUIRED` |
| pointsPreview.pointsToUse | Integer | Server-computed points to deduct | Present when `CHALLENGE_REQUIRED` |
| pointsPreview.cardAmount | BigDecimal | Amount charged to card (TWD) after offset | Present when `CHALLENGE_REQUIRED` |
| declineReason | String | Decline reason code | Present when `DECLINED` |

---

## Service UseCase Flow

**UseCase:** `CardAuthAuthorizeUseCase.authorize(CardAuthAuthorizeRq)`

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DOMAIN]` | Decrypt `encryptedCard` using in-memory combineKey (AES-256 GCM) → plainCard `{ pan, expiry, cvv }` in JVM Heap only | `CardEncryptionService.decrypt()` |
| 2 | `[DB READ]` | Derive `panHash` from decrypted PAN (HMAC-SHA256); query `card` by `pan_hash` | `CardQueryService.findByPanHash()` |
| 3 | `[DOMAIN]` | Validate card status: must be `ACTIVE`; check expiry (`expiry_date` decrypted and compared to today) | `CardValidationService.validateActive()` |
| 4 | `[DB READ]` | Load `card_type_limit` row for the card's `card_type` | `CardTypeLimitQueryService.findByCardType()` |
| 5 | `[REDIS READ]` | Read `card:daily:{cardId}:{yyyyMMdd}` — today's cumulative amount (0 if absent); skip for `INFINITE` card type | `DailyLimitService.getDailyAccumulated()` |
| 6 | `[DOMAIN]` | Validate `single_txn_limit`: `amount_twd <= single_txn_limit` (skip if null) | `CardValidationService.checkSingleTxnLimit()` |
| 7 | `[DOMAIN]` | Validate `daily_limit`: `accumulated + amount_twd <= daily_limit` (skip if null) | `CardValidationService.checkDailyLimit()` |
| 8 | `[FEIGN]` | Call Wallet Service → retrieve `{ availableBalance, pointsFirstEnabled, maxPointsPerTxn }` → [spec/wallet-service/get-member-balance-and-preference.md](../wallet-service/get-member-balance-and-preference.md) | — |
| 9 | `[DOMAIN]` | Calculate `amountTwd` (for non-TWD: call FX Service in Step 10 first); calculate `pointsToUse = min(availableBalance, amountTwd, maxPointsPerTxn ?? ∞)` if `pointsFirstEnabled`; else `pointsToUse = 0`; `cardAmount = amountTwd - pointsToUse` | `PointsOffsetService.calculate()` |
| 10 | `[FEIGN]` | (Non-TWD only) Call FX Service `lockRate(currency)` → `{ fxRateId, fxRate }` (TTL 10 min); compute `amountTwd = floor(amount × fxRate)`, `fxFee = floor(amountTwd × fx_fee_rate)`, `totalTwd = amountTwd + fxFee` → [spec/fx-service/get-rate.md](../fx-service/get-rate.md) | — |
| 11 | `[DOMAIN]` | Verify sufficient funds: `availableBalance - pointsToUse >= cardAmount`; if not → `DECLINED + INSUFFICIENT_FUNDS` | `WalletValidationService.checkFunds()` |
| 12 | `[FEIGN]` | Call Wallet Service `reserve(userId, pointsToUse)` → `reservationId` → [spec/wallet-service/reserve.md](../wallet-service/reserve.md) | — |
| 13 | `[DOMAIN]` | Generate `challengeRef` (UUID v4); generate 6-digit cryptographically random OTP | `ChallengeService.generate()` |
| 14 | `[KAFKA]` | Publish OTP SMS event to RabbitMQ topic consumed by Notification Service (payload: masked phone `09xx****xx`, OTP, challengeRef) | — |
| 15 | `[REDIS WRITE]` | Write `otp:{challengeRef}` TTL 180s — payload: `{ cardId, userId, amountTwd, fxFee?, currency, merchantId, merchantMCC, pointsToUse, cardAmount, reservationId, fxRateId? }` | `PendingAuthStore.save()` |
| 16 | `[DOMAIN]` | Clear `cvv` from JVM Heap immediately after OTP is generated | `CardEncryptionService.clearSensitiveData()` |
| 17 | `[RETURN]` | Return `CHALLENGE_REQUIRED` + `challengeRef` + `pointsPreview { pointsToUse, cardAmount }` | — |

> **PCI DSS:** Decrypted PAN and expiry remain in JVM Heap for the duration of this UseCase only. CVV is discarded at Step 16. Neither PAN nor CVV appears in any log, Redis key, or Kafka event.

---

## Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card` | READ | Identify cardholder by `pan_hash`; read `card_type`, `status`, `expiry_date` (encrypted), `user_id` |
| `card_type_limit` | READ | Load `single_txn_limit`, `daily_limit`, `fx_fee_rate` for the card's `card_type` |

### Redis

| Key Pattern | Operation | TTL | Description |
|-------------|-----------|-----|-------------|
| `card:daily:{cardId}:{yyyyMMdd}` | READ | Until end of day (23:59:59 local) | Today's cumulative transaction amount (in cents TWD) |
| `otp:{challengeRef}` | WRITE | 180s | Pending authorization session payload |

### Key Management

`pan_hash` is computed with **HMAC-SHA256** using a dedicated secret key that follows
the same injection pattern as the AES-256 card encryption keys:

```yaml
# application.yml (card-service)
card:
  security:
    pan-hmac-key: ${CARD_PAN_HMAC_KEY}   # injected from K8s Secret at runtime
    aes-key:      ${CARD_AES_KEY}
```

```yaml
# k8s/card-service-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: card-service-secrets
type: Opaque
stringData:
  CARD_PAN_HMAC_KEY: "<base64-encoded-256-bit-key>"
  CARD_AES_KEY:      "<base64-encoded-256-bit-key>"
```

- The key is loaded at startup via `@ConfigurationProperties` — never hard-coded.
- `pan_hash` is **write-once** on card creation; the HMAC key must not be rotated
  without a full re-hash migration.
- The key must not be logged, stored in DB, or appear in any Kafka event.

### Table Schema

#### card (relevant columns)

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| card_id | BIGINT | NO | PK, Snowflake |
| user_id | BIGINT | NO | FK to member |
| card_type | VARCHAR(20) | NO | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` |
| card_network | VARCHAR(10) | NO | `VISA` / `MASTERCARD` / `JCB` |
| status | VARCHAR(20) | NO | `PENDING` / `ACTIVE` / `FROZEN` / `CANCELLED` |
| pan_encrypted | VARCHAR(512) | NO | AES-256 encrypted PAN |
| pan_masked | VARCHAR(20) | NO | `****-****-****-1234` |
| pan_hash | VARCHAR(64) | NO | HMAC-SHA256 of plain PAN (lookup key) |
| expiry_date | VARCHAR(512) | NO | AES-256 encrypted `MMYY` |
| created_at | DATETIME(3) | NO | |
| updated_at | DATETIME(3) | NO | |

#### card_type_limit

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| card_type | VARCHAR(20) | NO | PK |
| single_txn_limit | BIGINT | YES | Single transaction limit (cents); NULL = unlimited |
| daily_limit | BIGINT | YES | Daily cumulative limit (cents); NULL = unlimited |
| domestic_reward_rate | DECIMAL(5,4) | NO | e.g. `0.0100` |
| overseas_reward_rate | DECIMAL(5,4) | NO | e.g. `0.0500` |
| fx_fee_rate | DECIMAL(5,4) | NO | e.g. `0.0100` |

---

## Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 422 | CA00001 `CARD_NOT_FOUND` | Card not found | No card record matches `pan_hash` |
| 422 | CA00002 `CARD_NOT_ACTIVE` | Card is not active | Card `status ≠ ACTIVE` |
| 422 | CA00003 `CARD_EXPIRED` | Card has expired | `expiry_date` is in the past |
| 422 | CA00004 `CARD_FROZEN` | Card is frozen | Card `status = FROZEN` |
| 422 | CA00005 `CARD_CANCELLED` | Card has been cancelled | Card `status = CANCELLED` |
| 422 | CA00006 `SINGLE_TXN_LIMIT_EXCEEDED` | Single transaction limit exceeded | `amount_twd > single_txn_limit` |
| 422 | CA00007 `DAILY_LIMIT_EXCEEDED` | Daily cumulative limit exceeded | `accumulated + amount_twd > daily_limit` |
| 422 | `INSUFFICIENT_FUNDS` | Wallet balance insufficient | `availableBalance - pointsToUse < cardAmount` |
| 500 | `INTERNAL_ERROR` | Unexpected system error | Unhandled exception |
