# Card Auth Authorize — card-service Spec

> **Service:** card-service
> **Contract Module:** card-service-contract
> **Caller:** [mock-nccc/pay.md](../mock-nccc/pay.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the encrypted card data (`encryptedCard`) assembled by Mock NCCC, decrypts it using the in-memory `combineKey`, performs card validation and transaction limit checks, generates a one-time OTP, and publishes a notification event for the notify-service to deliver via the member's registered OTP channel.

Returns only `{ challengeRef }` to Mock NCCC. All Saga coordination (wallet reserve/confirm/release, Kafka publishing) is owned by the Transaction Orchestrator.

> **No cross-service calls.** card-service does not call FX Service or Wallet Service. `txnTwdBaseAmount` is pre-computed by ORCH and forwarded through Mock NCCC.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (Mock NCCC) | [mock-nccc/pay.md](../mock-nccc/pay.md) |
| Plain card data source | [card-service/card-plain.md](card-plain.md) |
| OTP delivery consumer | [notify-service/otp-delivery.md](../notify-service/otp-delivery.md) |
| OTP verification (Leg 2) | [card-service/card-auth-verify-challenge.md](card-auth-verify-challenge.md) |
| CombineKey startup | [card-service/combine-key-startup.md](combine-key-startup.md) |

---

## 3. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /card-service |
| Controller mapping | /auth/authorize |
| Effective received path | /card-service/auth/authorize |
| Caller | Mock NCCC (`CardServiceFeignClient`) |

### Request (`card.service.contract.dto.CardAuthAuthorizeRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| encryptedCard | String | ✅ | @NotBlank | AES-256-GCM encrypted `{ pan, expiryMMYY, cvvItoken }` assembled by Mock NCCC using `MOCK_COMBINE_KEY` |
| amount | Long | ✅ | @NotNull; @Min(1) | Original transaction amount (cents) |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 currency code |
| txnTwdBaseAmount | Long | ✅ | @NotNull; @Min(1) | TWD-equivalent amount pre-computed by ORCH; used for limit checks |
| merchantId | String | ✅ | @NotBlank | Merchant identifier |

### Response (`card.service.contract.dto.CardAuthAuthorizeRs`)

| Field | Type | Description |
|-------|------|-------------|
| challengeRef | String | UUID identifying the OTP session in Redis; valid for 180 s |

---

## 4. Service UseCase Flow

**UseCase:** `CardAuthAuthorizeUseCase.authorize(CardAuthAuthorizeRq)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DOMAIN]` | Get `combineKey` from `CombineKeyHolder`; null → throw 503 `COMBINE_KEY_UNAVAILABLE` |
| 2 | `[DOMAIN]` | AES-256-GCM decrypt `encryptedCard` using `combineKey` → `{ pan, expiryMMYY, cvvItoken }`; wrap in try-finally to zero-fill `pan` byte array at end of method |
| 3 | `[DB READ]` | HMAC-SHA256(`pan`) → `panHash`; `SELECT card WHERE pan_hash = panHash`; not found → `CA00001` |
| 4 | `[DOMAIN]` | Compare `cvvItoken`: received value == `card.cvv_itoken`; mismatch → `CA00014` |
| 5 | `[DOMAIN]` | Validate card `status`: `ACTIVE` only; `DISABLED/PENDING` → `CA00002`; `FROZEN` → `CA00004` |
| 6 | `[DOMAIN]` | Validate expiry: decrypt `card.expiry_date` → `expiryMMYY`; parse → expiry date; past current month → `CA00003` |
| 7 | `[DOMAIN]` | Single-txn limit: if `card_type != INFINITE` and `txnTwdBaseAmount > card_type_limit.single_txn_limit` → `CA00006` |
| 8 | `[REDIS READ]` | GET `card:daily:{cardId}:{yyyyMMdd}`; if `(currentTotal + txnTwdBaseAmount) > card_type_limit.daily_limit` and `daily_limit != null` → `CA00007` |
| 9 | `[DOMAIN]` | Generate `challengeRef` (UUID v4) + `otp` (6-digit, `SecureRandom`) |
| 10 | `[MQ PUBLISH]` | Publish to RabbitMQ exchange `notification`, routing key `otp.send`; payload: `{ userId: card.user_id, otp }`; see [notify-service/otp-delivery.md](../notify-service/otp-delivery.md) |
| 11 | `[REDIS WRITE]` | SET `otp:{challengeRef}` = OTP Session (see below); TTL 180 s |
| 12 | `[DOMAIN]` | Zero-fill `pan` byte array (ensured by try-finally from Step 2) |
| 13 | `[RETURN]` | Return `CardAuthAuthorizeRs { challengeRef }` |

> **Step 4 (cvvItoken):** Lookup is by `card_id` after PAN-hash resolution — not by iToken directly.
> Raw CVV is never stored; `cvv_itoken` is a one-way HMAC-SHA256 token (see `card_db.md`).
>
> **Step 12:** The try-finally from Step 2 guarantees PAN bytes are zeroed even if an exception is thrown in Steps 3–11.

---

## 5. Redis — OTP Session

**Key:** `otp:{challengeRef}`
**TTL:** 180 seconds
**Written by:** Step 11 (this UseCase)
**Read/deleted by:** `card-service/card-auth-verify-challenge.md`

```json
{
  "userId": 123456789,
  "cardId": 987654321,
  "txnAmount": 10000,
  "txnCurrency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "otpValue": "123456"
}
```

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| userId | Long | `card.user_id` | Card owner member ID |
| cardId | Long | `card.card_id` | Card ID |
| txnAmount | Long | Request `amount` | Original transaction amount (cents) |
| txnCurrency | String | Request `currency` | ISO 4217 currency code |
| txnTwdBaseAmount | Long | Request `txnTwdBaseAmount` | TWD-equivalent amount (pre-computed by ORCH) |
| isOverseas | Boolean | `currency != "TWD"` | Whether transaction is overseas |
| otpValue | String | Step 9 | 6-digit OTP; read by verify-challenge |

> **Note:** `merchantId`, `pointsToUse`, and `reservationId` are owned by the ORCH Saga State — not stored here.

---

## 6. Redis — Daily Accumulator (read-only in this UseCase)

**Key:** `card:daily:{cardId}:{yyyyMMdd}`
**Read by:** Step 8 (check only; does not update)
**Updated by:** `card-auth-verify-challenge` Step 3 (on OTP approved; INCR by `txnTwdBaseAmount`)

---

## 7. Database

**Tables read:**

| Table | Columns | Purpose |
|-------|---------|---------|
| `card` | `pan_hash`, `cvv_itoken`, `status`, `expiry_date`, `user_id`, `card_id`, `card_type`, `pan_masked` | Card lookup and validation |
| `card_type_limit` | `single_txn_limit`, `daily_limit` | Transaction limit checks (Step 7–8) |

---

## 8. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | `CombineKeyHolder.combineKey` is null at startup |
| 404 | CA00001 | CARD_NOT_FOUND | PAN hash not found in `card` table |
| 422 | CA00014 | CVV_ITOKEN_MISMATCH | Received `cvvItoken` does not match `card.cvv_itoken` |
| 409 | CA00002 | CARD_NOT_ACTIVE | Card `status` is not `ACTIVE` |
| 409 | CA00003 | CARD_EXPIRED | Card expiry date is in the past |
| 409 | CA00004 | CARD_FROZEN | Card `status` is `FROZEN` |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | `txnTwdBaseAmount` exceeds `single_txn_limit` |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | Accumulated daily total would exceed `daily_limit` |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 9. Changelog

### v1.0 — 2026-03 — Initial spec
