# Transaction Flow

This chapter defines the business rules, flow design, and conventions for credit card transactions in WillCard. All specs under `spec/mobile-bff/card-pay*`, `spec/mock-nccc/`, `spec/card-service/`, `spec/wallet-service/`, `spec/points-service/`, and `spec/ledger-service/` must conform to these rules.

---

## 1. WillCard as Issuer

WillCard issues virtual Visa/Mastercard cards backed by the member's e-wallet balance. In Taiwan's card payment ecosystem, **NCCC (聯合信用卡處理中心 / 財金公司)** acts as the card network and clearing house.

**WillCard is the Issuer. The cardholder initiates from the App; WillCard BFF actively calls Mock NCCC, forwarding transaction details and the CVV iToken; Mock NCCC then calls card-service to perform issuer authorization.**

> In a real production network, the card network (NCCC) would route inbound authorization requests to WillCard as the issuer. In this project's mock setup the direction is reversed: the App triggers the flow through BFF, and WillCard is the party that actively calls out to Mock NCCC.

```
[Production reference — inbound from card network]
Cardholder uses virtual card at merchant
  → Merchant terminal / checkout
  → Acquiring bank
  → NCCC
  → WillCard Card Service (Issuer Authorization API)   ← called here
       ↓ on approval (async, Kafka fan-out)
  Points Service · Ledger Service · Notification Service

[Development — WillCard calls Mock NCCC outbound]
App → BFF → Mock NCCC Service
              ↓ (forward txn details + cvvItoken)
              → WillCard Card Service (auth/authorize)
                   ↓ on approval (async, Kafka fan-out)
             Points Service · Ledger Service · Notification Service
```

**Key identifiers:**

| Field | Produced by | Lifecycle | Purpose |
|-------|------------|-----------|---------|
| `challengeRef` | Card Service (auth/authorize) | OTP session (TTL 180s) | OTP binding; ties authorize and verify-challenge together |
| `txnId` | Card Service (verify-challenge, on approval) | Transaction record | Ledger, Kafka events, reconciliation |
| `reservationId` | Wallet Service (reserve) | Points hold (TTL 180s) | Links points reservation to transaction; null if no points used |

---

## 2. Complete End-to-End Transaction Flow

### 2.1 Sequence Diagram

```mermaid
sequenceDiagram
    participant App
    participant BFF as mobile-bff
    participant MNCCC as mock-nccc
    participant CS as card-service
    participant NS as notify-service
    participant WS as wallet-service
    participant FX as fx-service
    participant Redis
    participant KFK as Kafka
    participant PS as points-service
    participant LS as ledger-service

    Note over App,KFK: ── Leg 1: Initiate Payment ──

    App->>BFF: POST /api/v1/card/pay<br/>{cardId, amount, currency, merchantId, usePoints}
    BFF->>BFF: Validate JWT → inject X-Member-Id
    BFF->>Redis: SET idempotency key (TTL 60s)
    BFF->>KFK: Publish operation-log.card.pay (L1)
    BFF->>MNCCC: POST /mock-nccc/pay (Feign)

    Note over MNCCC,CS: ── Mock NCCC: Assemble encryptedCard ──
    MNCCC->>CS: GET /card-service/internal/cards/{cardId}/plain
    CS-->>MNCCC: {pan, expiryMMYY, cvvItoken}
    MNCCC->>MNCCC: AES-256-GCM encrypt<br/>{pan,expiryMMYY,cvvItoken} → encryptedCard
    MNCCC->>CS: POST /card-service/auth/authorize<br/>{encryptedCard, amount, currency, merchantId, usePoints}

    Note over CS: ── card-service: auth/authorize (Steps 1–15) ──
    CS->>CS: 1. CombineKeyHolder.get() — null→CA00013
    CS->>CS: 2. AES-256-GCM decrypt encryptedCard<br/>→ {pan, expiryMMYY, cvvItoken}
    CS->>CS: 3. HMAC-SHA256(pan)→pan_hash<br/>DB lookup → userId/cardId
    CS->>CS: 4. Compare cvvItoken vs card.cvv_itoken<br/>mismatch → CA00014
    CS->>CS: 5. Validate status=ACTIVE, expiry valid<br/>fail→CA00002/CA00003/CA00004

    alt Foreign currency
        CS->>FX: POST /fx-service/convert {amount, currency}
        FX-->>CS: txnTwdBaseAmount
    end

    CS->>CS: 6. Single txn limit check → CA00006
    CS->>Redis: 7. GET card:daily:{cardId}:{yyyyMMdd}
    CS->>CS: Daily cumulative check → CA00007

    alt usePoints = true
        CS->>WS: POST /wallet-service/reserve {userId, pointsToUse}
        WS-->>CS: reservationId
        Note right of WS: available_balance -= pointsToUse<br/>reserved_balance += pointsToUse
    end

    CS->>CS: 11. Generate 6-digit OTP (SecureRandom)
    CS->>NS: RabbitMQ: notification.otp.sms<br/>{userId, maskedPhone, otp}
    NS-->>App: SMS: "WillCard OTP: xxxxxx (3 min)"
    CS->>Redis: 13. SET otp:{challengeRef} (TTL 180s)<br/>{userId,cardId,txnAmount,txnTwdBaseAmount,<br/>isOverseas,pointsToUse,reservationId,otpValue}
    CS->>CS: 14. Zero-fill PAN byte array
    CS-->>MNCCC: {challengeRef, pointsToUse, estimatedTwdAmount}
    MNCCC-->>BFF: {challengeRef, pointsToUse, estimatedTwdAmount}
    BFF-->>App: {challengeRef, pointsToUse, estimatedTwdAmount}

    Note over App: Display OTP input + points offset preview

    Note over App,KFK: ── Leg 2: Submit OTP ──

    App->>BFF: POST /api/v1/card/pay/confirm<br/>{challengeRef, otp}
    BFF->>KFK: Publish operation-log.card.pay-confirm (L1)
    BFF->>MNCCC: POST /mock-nccc/pay/confirm (Feign)
    MNCCC->>CS: POST /card-service/auth/verify-challenge<br/>{challengeRef, otp}

    Note over CS: ── card-service: verify-challenge ──
    CS->>Redis: GET otp:{challengeRef}
    Note right of Redis: not found → SESSION_EXPIRED

    alt OTP correct
        CS->>WS: POST /wallet-service/confirm-deduct {reservationId}
        Note right of WS: reserved_balance -= pointsToUse
        CS->>Redis: INCR card:daily:{cardId}:{yyyyMMdd} (add txnTwdBaseAmount)
        CS->>CS: Assign txnId (Snowflake)
        CS->>KFK: Publish txn.card.authorized
        CS->>Redis: DEL otp:{challengeRef}
        CS-->>MNCCC: {result: APPROVED, txnId}
        MNCCC-->>BFF: {result: APPROVED, txnId}
        BFF-->>App: {result: APPROVED, txnId}
    else OTP wrong (≥ 3 per-session → void; ≥ 5 per-card → FROZEN)
        CS->>Redis: INCR otp:attempt:{challengeRef}
        CS->>Redis: INCR otp:card:fail:{cardId}
        CS-->>MNCCC: {result: DECLINED, reason}
        MNCCC-->>BFF: DECLINED
        BFF-->>App: DECLINED + reason
    end

    Note over KFK,LS: ── Post-auth Async (Kafka fan-out) ──
    KFK->>PS: txn.card.authorized → calc reward points
    PS->>WS: credit(userId, rewardPoints)
    KFK->>LS: txn.card.authorized → journal_entry INSERT
    KFK->>NS: txn.card.authorized → push receipt + email
    NS-->>App: Push notification: transaction receipt
```

### 2.2 Step-by-Step Service Breakdown

| # | Service | Action | Redis / DB / Event State |
|---|---------|--------|--------------------------|
| **Leg 1 — Initiate Payment** | | | |
| 1 | App | `POST /api/v1/card/pay` — `{cardId, amount, currency, merchantId, usePoints}` | — |
| 2 | BFF | Validate JWT; extract `userId`; inject `X-Member-Id` header | — |
| 3 | BFF | Check Redis idempotency key; duplicate → return cached response | Redis `SET idempotency:{hash}` TTL 60s |
| 4 | BFF | Publish `operation-log.card.pay` (L1) | Kafka: `operation-log.card.pay` |
| 5 | BFF→Mock NCCC | Feign `POST /mock-nccc/pay` | — |
| 6 | Mock NCCC | Feign `GET /card-service/internal/cards/{cardId}/plain` | — |
| 7 | Card Service | Return `{pan, expiryMMYY, cvvItoken}` from DB (dev/mock profile only) | DB READ `card` |
| 8 | Mock NCCC | AES-256-GCM encrypt `{pan, expiryMMYY, cvvItoken}` → `encryptedCard` using `MOCK_COMBINE_KEY` | — |
| 9 | Mock NCCC→Card Service | Feign `POST /card-service/auth/authorize` | — |
| **auth/authorize — Decrypt & Validate** | | | |
| 10 | Card Service | `CombineKeyHolder.get()` combineKey; null → throw `CA00013` HTTP 503 | — |
| 11 | Card Service | AES-256-GCM decrypt `encryptedCard` → `{pan, expiryMMYY, cvvItoken}` in JVM heap | — |
| 12 | Card Service | `HMAC-SHA256(pan, PAN_HMAC_KEY)` → `pan_hash`; DB lookup by `pan_hash` → `cardId`, `userId` | DB READ `card` |
| 13 | Card Service | Compare received `cvvItoken` vs `card.cvv_itoken`; mismatch → throw `CA00014` | DB READ |
| 14 | Card Service | Validate `status = ACTIVE`, expiry not past; fail → `CA00002` / `CA00003` / `CA00004` | DB READ |
| **auth/authorize — Limits & Points** | | | |
| 15 | Card Service | Determine `isOverseas` (currency ≠ TWD); if foreign → Feign `POST /fx-service/convert` → `txnTwdBaseAmount` | Feign: fx-service |
| 16 | Card Service | Single txn limit: `txnTwdBaseAmount ≤ single_txn_limit` (INFINITE skips) → `CA00006` | DB READ `card_type_limit` |
| 17 | Card Service | Daily limit: Redis `GET card:daily:{cardId}:{yyyyMMdd}` + `txnTwdBaseAmount ≤ daily_limit` → `CA00007` | Redis READ |
| 18 | Card Service | Points: `usePoints ? min(available_balance, txnTwdBaseAmount) : 0` → `pointsToUse` | Feign: wallet-service |
| 19 | Card Service | If `pointsToUse > 0`: Feign `wallet-service/reserve(userId, pointsToUse)` → `reservationId` | `available_balance -= pts`, `reserved_balance += pts` |
| **auth/authorize — OTP & Response** | | | |
| 20 | Card Service | Generate 6-digit OTP (`SecureRandom`) | JVM heap only |
| 21 | Card Service | Publish RabbitMQ `notification.otp.sms` `{userId, maskedPhone, otp}` | RabbitMQ queue |
| 22 | Notify Service | Consume `notification.otp.sms`; send SMS to member's registered phone | SMS sent |
| 23 | Card Service | `SET` Redis OTP session `otp:{challengeRef}` (TTL 180s) with full txn context + `otpValue` | Redis WRITE |
| 24 | Card Service | Zero-fill PAN plaintext byte array (try-finally) | — |
| 25 | Card Service | Return `{challengeRef, pointsToUse, estimatedTwdAmount}` | — |
| 26 | Mock NCCC | Forward response to BFF | — |
| 27 | BFF | Return `{challengeRef, pointsToUse, estimatedTwdAmount}` to App | — |
| 28 | App | Display OTP input screen + points offset preview | — |
| **Leg 2 — Submit OTP** | | | |
| 29 | App | `POST /api/v1/card/pay/confirm` — `{challengeRef, otp}` | — |
| 30 | BFF | Validate JWT; publish `operation-log.card.pay-confirm` (L1) | Kafka: `operation-log.card.pay-confirm` |
| 31 | BFF→Mock NCCC | Feign `POST /mock-nccc/pay/confirm` | — |
| 32 | Mock NCCC→Card Service | Feign `POST /card-service/auth/verify-challenge` | — |
| **verify-challenge — OTP Correct Path** | | | |
| 33 | Card Service | Redis `GET otp:{challengeRef}`; not found → `DECLINED + SESSION_EXPIRED` | Redis READ |
| 34 | Card Service | Compare `session.otpValue == submitted otp`; match → continue | — |
| 35 | Card Service | If `reservationId` not null: Feign `wallet-service/confirm-deduct(reservationId)` | `reserved_balance -= pts` |
| 36 | Card Service | Redis `INCR card:daily:{cardId}:{yyyyMMdd}` by `txnTwdBaseAmount`; set TTL to 23:59:59 | Redis WRITE |
| 37 | Card Service | Assign `txnId` (Snowflake) | — |
| 38 | Card Service | Kafka Publish `txn.card.authorized` (full payload) | Kafka: `txn.card.authorized` |
| 39 | Card Service | Redis `DEL otp:{challengeRef}` | Redis DELETE |
| 40 | Card Service | Return `{result: APPROVED, txnId}` | — |
| 41 | Mock NCCC / BFF | Forward `APPROVED + txnId` to App | — |
| **verify-challenge — OTP Wrong Path** | | | |
| 42 | Card Service | `INCR otp:attempt:{challengeRef}` (TTL 180s); ≥ 3 → void session + release if `reservationId` → `DECLINED + SESSION_VOIDED` | Redis WRITE |
| 43 | Card Service | `INCR otp:card:fail:{cardId}` (TTL 900s); ≥ 5 → freeze card + Kafka `card.risk.otp-threshold-exceeded` + release → `DECLINED + CARD_LOCKED` | Redis WRITE; Kafka |
| 44 | Card Service | Return `DECLINED + OTP_FAILED + attemptsRemaining` | — |
| **Post-Auth Async (Kafka fan-out)** | | | |
| 45 | Points Service | Consume `txn.card.authorized`; calculate `reward_points`; Feign `wallet-service/credit`; insert `point_reward_batch` (PENDING) | DB WRITE `point_reward_batch` |
| 46 | Ledger Service | Consume `txn.card.authorized`; insert `journal_entry` rows (idempotent: `ON DUPLICATE IGNORE`) | DB WRITE `journal_entry` |
| 47 | Notify Service | Consume `txn.card.authorized`; push notification + email receipt | Push + email sent |

---

## 3. Issuer Authorization API (Card Service)

Card Service exposes two endpoints that **NCCC calls directly** (production: mutual TLS; development: Mock NCCC on internal network).

| Endpoint | Caller | Purpose |
|----------|--------|---------|
| `POST /card-service/auth/authorize` | NCCC / Mock NCCC | Decrypt card data, validate, generate OTP |
| `POST /card-service/auth/verify-challenge` | NCCC / Mock NCCC | Verify OTP; finalize approval or decline |

### 3.1 auth/authorize — UseCase Flow

```
Request: { encryptedCard, amount, currency, merchantId, usePoints, userId }

Step  1  [DOMAIN]   CombineKeyHolder.get() — null → CA00013 HTTP 503
Step  2  [DOMAIN]   AES-256-GCM decrypt encryptedCard → { pan, expiryMMYY, cvvItoken }
Step  3  [DB READ]  HMAC-SHA256(pan, PAN_HMAC_KEY) → pan_hash; SELECT card WHERE pan_hash = ?
Step  4  [DOMAIN]   Compare cvvItoken: encryptedCard.cvvItoken == card.cvv_itoken; mismatch → CA00014
Step  5  [DOMAIN]   Validate card: status = ACTIVE, expiry ≥ today; fail → CA00002 / CA00003 / CA00004
Step  6  [DOMAIN]   Currency: TWD → txnTwdBaseAmount = amount; foreign → Feign POST /fx-service/convert
Step  7  [DOMAIN]   Single txn limit: txnTwdBaseAmount ≤ single_txn_limit (INFINITE skips) → CA00006
Step  8  [REDIS]    Daily limit: GET card:daily:{cardId}:{yyyyMMdd} + txnTwdBaseAmount ≤ daily_limit → CA00007
Step  9  [FEIGN]    Read wallet available_balance; pointsToUse = usePoints ? min(balance, txnTwdBaseAmount) : 0
Step 10  [FEIGN]    If pointsToUse > 0: wallet-service.reserve(userId, pointsToUse) → reservationId
Step 11  [DOMAIN]   Generate 6-digit OTP (SecureRandom)
Step 12  [RABBIT]   Publish notification.otp.sms { userId, maskedPhone, otp }
Step 13  [REDIS]    SET otp:{challengeRef} TTL 180s (see §3.3 for payload)
Step 14  [DOMAIN]   Zero-fill PAN plaintext byte array (try-finally guarantee)
Step 15  [RETURN]   { challengeRef, pointsToUse, estimatedTwdAmount }

Compensation: any failure at Step 6+ after reserve → release(reservationId)
```

### 3.2 auth/verify-challenge — UseCase Flow

```
Request: { challengeRef, otp }

Step  1  [REDIS]    GET otp:{challengeRef}; not found → DECLINED + SESSION_EXPIRED

── OTP Correct ──
Step  2  [FEIGN]    If reservationId != null: wallet-service.confirmDeduct(reservationId)
Step  3  [REDIS]    INCR card:daily:{cardId}:{yyyyMMdd} by txnTwdBaseAmount; EXPIREAT 23:59:59
Step  4  [DOMAIN]   Assign txnId (Snowflake)
Step  5  [KAFKA]    Publish txn.card.authorized (see §6.1 for payload)
Step  6  [REDIS]    DEL otp:{challengeRef}
Step  7  [RETURN]   { result: APPROVED, txnId }

── OTP Wrong ──
Step  8  [REDIS]    INCR otp:attempt:{challengeRef} (TTL 180s)
         ≥ 3 → void session; if reservationId → release; RETURN DECLINED + SESSION_VOIDED
Step  9  [REDIS]    INCR otp:card:fail:{cardId} (TTL 900s)
         ≥ 5 → UPDATE card SET status=FROZEN; Kafka card.risk.otp-threshold-exceeded; release; RETURN DECLINED + CARD_LOCKED
Step 10  [RETURN]   DECLINED + OTP_FAILED + attemptsRemaining
```

### 3.3 Redis OTP Session Payload

```json
{
  "userId": 123,
  "cardId": 456,
  "txnAmount": 10000,
  "txnCurrency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "pointsToUse": 5000,
  "reservationId": 789,
  "merchantId": "MERCHANT_001",
  "otpValue": "123456"
}
```

> `txnTwdBaseAmount`: TWD if domestic; FX converted amount if foreign.
> `reservationId`: null if `pointsToUse = 0`.

---

## 4. OTP Failure Thresholds

Two independent layers protect against accidental retry and card theft probing.

**Layer 1 — Per-challengeRef (session-level)**

| Item | Value |
|------|-------|
| Redis key | `otp:attempt:{challengeRef}` (TTL 180s) |
| Max retries | 3 |
| On limit reached | This session cancelled; card unaffected; release reservationId if present |

**Layer 2 — Per-card sliding window (fraud detection)**

| Item | Value |
|------|-------|
| Redis key | `otp:card:fail:{cardId}` (TTL 900s / 15 min) |
| Threshold | **5 failures within 15 minutes** (across any challengeRef) |
| On threshold reached | Card frozen (`status = FROZEN`) + Kafka `card.risk.otp-threshold-exceeded` + release reservationId |

Each failure increments **both** counters simultaneously.

---

## 5. Mock NCCC (Development & Testing)

Mock NCCC is a **development-only** service that bridges BFF to card-service by simulating NCCC behavior.

**card-plain internal endpoint** (`GET /card-service/internal/cards/{cardId}/plain`):
- Dev/mock profile only; Nginx blocks `/internal/` from external traffic
- Returns `{ pan, expiryMMYY, cvvItoken }` in plaintext from DB
- Spec: [spec/card-service/card-plain.md](../spec/card-service/card-plain.md)

**encryptedCard composition:**
```
encryptedCard = AES-256-GCM(
  plaintext: JSON{ pan, expiryMMYY, cvvItoken },
  key: MOCK_COMBINE_KEY,
  iv: randomly generated, prepended to ciphertext
)
```

**MOCK_COMBINE_KEY** is a fixed 256-bit key in `application-dev.yml`. In production, the real combineKey is assembled from 3 encrypted source keys in `card_key_parts` DB table. See [guideline/14-3ds-combine-key.md](14-3ds-combine-key.md).

---

## 6. Post-Auth Async Processing

After `txn.card.authorized` is published to Kafka, three independent consumers process it in parallel (fan-out pattern).

### 6.1 Kafka Payload — `txn.card.authorized`

```json
{
  "txnId": "Snowflake ID",
  "userId": 123,
  "cardId": 456,
  "cardType": "OVERSEAS",
  "maskedPan": "****-****-****-1234",
  "merchantId": "MERCHANT_001",
  "txnAmount": 10000,
  "txnCurrency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "pointsToUse": 5000,
  "txnTimestamp": "2026-03-30T10:00:00.000Z"
}
```

### 6.2 Points Service

- Kafka Consumer: `txn.card.authorized` (idempotent: `uk_pil_source_txn_id`)
- Calculate `reward_points = floor(txnTwdBaseAmount × reward_rate)` (see §9)
- Feign `wallet-service/credit(userId, rewardPoints)`
- Insert `point_reward_batch` (status = PENDING; expires 1 year from txnTimestamp month-end)

### 6.3 Ledger Service

- Kafka Consumer: `txn.card.authorized` (idempotent: `ON DUPLICATE KEY IGNORE` on `idempotency_key`)
- Insert `journal_entry` rows based on scenario (see §12.4)

### 6.4 Notification Service — Transaction Receipt

- Kafka Consumer: `txn.card.authorized`
- Internal forward to RabbitMQ queue `notification.txn.receipt`
- Push notification + email with transaction details

---

## 7. Points-First Preference

### 7.1 Preference Storage

**Table: `member_points_preference` (Wallet Service DB)**

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | BIGINT PK | |
| `points_first_enabled` | BOOLEAN | Auto-apply points offset on every transaction |
| `max_points_per_txn` | INT | Per-transaction cap (NULL = no cap) |
| `updated_at` | DATETIME | |

### 7.2 Auto-Calculation at Authorization

```
if points_first_enabled:
  pointsToUse = min(
    available_points_balance,
    txnTwdBaseAmount,
    max_points_per_txn ?? ∞
  )
else:
  pointsToUse = 0

estimatedTwdAmount = txnTwdBaseAmount - pointsToUse
```

### 7.3 Points Exchange Rules

| Rule | Value |
|------|-------|
| Exchange rate | **1 point = NT$1** (1:1) |
| Minimum | 1 point |
| Maximum | Full amount (100% offset) |
| Both `pointsToUse` and `estimatedTwdAmount` | Must be recorded on the transaction |

### 7.4 Reserve-Confirm-Release

```
auth/authorize:
  wallet-service.reserve(userId, pointsToUse)
  → available_balance -= pointsToUse
  → reserved_balance  += pointsToUse
  Reserve TTL = 180s (aligned with OTP TTL; lazy cleanup on expiry)

verify-challenge — APPROVED:
  wallet-service.confirmDeduct(reservationId)
  → reserved_balance -= pointsToUse

Compensation (OTP failure / DECLINED / TTL expiry):
  wallet-service.release(reservationId)
  → reserved_balance -= pointsToUse
  → available_balance += pointsToUse
```

---

## 8. Saga Compensation Rules

| Saga Step | Compensation Action | Trigger |
|-----------|--------------------|---------|
| Points reserve | `release(reservationId)` | Card validation fail, limit exceeded, OTP failure (≥3), card locked, session expiry (lazy) |
| FX rate lock | Auto-expires via TTL | No active compensation needed |
| Ledger entries | Write REVERSAL entries (never delete) | Refund flow (separate spec) |

Ledger entries are **immutable**. Compensation at the ledger level is always a new reversal entry, never an update or delete.

---

## 9. Reward Calculation Rules

### 9.1 Reward Basis

- Reward is calculated on `txnTwdBaseAmount` (points-offset portion excluded)
- Result is **floor (truncate)** — no rounding
- For foreign currency: use the TWD-converted base amount (`twd_base`, excluding FX fee)

```
reward_points = floor(txnTwdBaseAmount × reward_rate)

Example: txnTwdBaseAmount = 60, reward_rate = 2%
  60 × 0.02 = 1.2 → floor → 1 point
```

### 9.2 Reward Plan (MCC-based)

Reward rates are stored in `reward_plan` (Points Service DB). Loaded into in-memory `Map<String, BigDecimal>` at startup; invalidated on write. Read path never hits DB. If no MCC match, fall back to `mcc_code = 'DEFAULT'`.

**Table: `reward_plan`**

| Column | Type | Description |
|--------|------|-------------|
| `plan_id` | BIGINT PK | Snowflake |
| `mcc_code` | VARCHAR(10) | MCC code; `DEFAULT` for catch-all |
| `reward_rate` | DECIMAL(5,4) | e.g., `0.0200` = 2% |
| `effective_from` | DATE | Plan start date |
| `effective_to` | DATE | Plan end date (null = no expiry) |
| `created_at` | DATETIME | |

### 9.3 Reward Point Batch

Each reward issuance creates one `point_reward_batch` record with its own balance and expiry, enabling FIFO deduction and precise expiry management.

**Table: `point_reward_batch`**

| Column | Type | Description |
|--------|------|-------------|
| `batch_id` | BIGINT PK | Snowflake |
| `user_id` | BIGINT | |
| `source_txn_id` | BIGINT | Source transaction |
| `issued_amount` | INT | Points issued |
| `remaining_balance` | INT | Decremented on redemption |
| `status` | VARCHAR | `PENDING` / `CONFIRMED` / `EXPIRED` / `CANCELLED` |
| `expires_at` | DATETIME | `issued_at + 1 year` |
| `created_at` | DATETIME | |

**Status transitions:**
```
PENDING   →  CONFIRMED   (T+1 settlement)
PENDING   →  CANCELLED   (refund before settlement)
CONFIRMED →  EXPIRED     (expires_at reached)
CONFIRMED →  CANCELLED   (refund after settlement)
```

---

## 10. Foreign Currency Rules

| Rule | Value |
|------|-------|
| TWD transactions | No handling fee |
| FX handling fee | **1.5%** of `twd_base`, added on top |
| Ledger currency | Always record in TWD; store original currency + rate for reference |
| Reward basis | `twd_base` only — FX fee excluded from reward |
| Merchant fee basis | `twd_base` only — FX fee is charged to cardholder only |

**FX amount calculation:**

```
twd_base      = foreign_amount × fx_rate
fx_fee        = floor(twd_base × 0.015)
total_twd     = twd_base + fx_fee        ← charged to member (estimatedTwdAmount)
reward_points = floor(twd_base × reward_rate)
```

Example: USD$100, rate 31.0, reward rate 1%
```
twd_base      = 100 × 31.0  = 3,100
fx_fee        = floor(3,100 × 0.015) = 46
total_twd     = 3,100 + 46 = 3,146
reward_points = floor(3,100 × 0.01) = 31 points
```

---

## 11. Transaction Limits

Limits are enforced per card type, stored in `card_type_limit` (Card Service DB).

| card_type | single_txn_limit | daily_limit |
|-----------|-----------------|-------------|
| CLASSIC | NT$100,000 | NT$200,000 |
| OVERSEAS | NT$100,000 | NT$200,000 |
| PREMIUM | NT$200,000 | NT$500,000 |
| INFINITE | No limit | No limit |

Limits are stored in TWD cents (`BIGINT`). `NULL` = no limit (INFINITE). Both limits are checked before OTP generation; if either fails, the request is declined before any points reserve.

---

## 12. Double-Entry Ledger Rules

### 12.1 Principles

- Ledger entries are **immutable** — INSERT only, no UPDATE or DELETE
- Every transaction produces a balanced DEBIT / CREDIT entry set
- Compensation is always a new REVERSAL entry set

### 12.2 Account Structure

| Code | Account | Description |
|------|---------|-------------|
| `1001` | Member Receivable | Amount charged to the member's virtual card at authorization |
| `1002` | Settlement Reserve | Cash pool; debited at T+1 settlement to merchant |
| `2001` | Merchant Payable | Net amount owed to merchant (after fee); includes points-offset subsidy |
| `2002` | Points Liability | Issued but unredeemed reward points (1 pt = NT$1) |
| `3001` | Merchant Fee Income | Service fee on TWD transactions |
| `3002` | FX Fee Income | 1.5% handling fee on foreign currency transactions |
| `4001` | Reward Points Expense | Cost of issuing reward points |
| `5001` | FX Gain/Loss | Exchange rate difference |

### 12.3 Journal Entry Types

| `journal_type` | Trigger |
|---------------|---------|
| `AUTHORIZATION` | Kafka `txn.card.authorized` |
| `SETTLEMENT` | T+1 batch — DR 2001 / CR 1002 |
| `REVERSAL` | Refund (separate spec) |
| `REWARD` | Reward points issuance (async, via Kafka) |

### 12.4 Entry Examples

**Case 1: TWD NT$100, no offset, reward 1%, merchant fee 3%**

| Debit | Amt | Credit | Amt |
|-------|-----|--------|-----|
| Member Receivable (1001) | 100 | Merchant Payable (2001) | 97 |
| | | Merchant Fee Income (3001) | 3 |
| Reward Expense (4001) | 1 | Points Liability (2002) | 1 |

**Case 2: TWD NT$100, 34 pts offset + NT$66 card, reward 1%, fee 3%**

| Debit | Amt | Credit | Amt |
|-------|-----|--------|-----|
| Member Receivable (1001) | 66 | Merchant Payable (2001) | 64.02 |
| | | Merchant Fee Income (3001) | 1.98 |
| Points Liability (2002) | 34 | Merchant Payable (2001) | 34 |
| Reward Expense (4001) | 0 | Points Liability (2002) | 0 |

> Merchant receives NT$100 total (64.02 + 1.98 retained + 34.00 subsidy).

**Case 3: Foreign USD$100, rate 31.0, reward 1%, fee 3%**
```
twd_base = 3,100 | fx_fee = 46 | total_twd = 3,146 | reward = 31 pts
```

| Debit | Amt | Credit | Amt |
|-------|-----|--------|-----|
| Member Receivable (1001) | 3,146 | Merchant Payable (2001) | 3,007 |
| | | Merchant Fee Income (3001) | 93 |
| | | FX Fee Income (3002) | 46 |
| Reward Expense (4001) | 31 | Points Liability (2002) | 31 |

### 12.5 `journal_entry` Schema

| Column | Type | Note |
|--------|------|------|
| `entry_id` | BIGINT PK | Snowflake |
| `idempotency_key` | VARCHAR(100) UNIQUE | `{messageKey}_{entry_seq}` — prevents duplicate on Kafka retry |
| `txn_id` | BIGINT | Source transaction |
| `journal_type` | VARCHAR(20) | See §12.3 |
| `entry_type` | VARCHAR(10) | `DEBIT` / `CREDIT` |
| `account_code` | VARCHAR(10) | See §12.2 |
| `amount` | DECIMAL(15,4) | |
| `currency` | VARCHAR(3) | `TWD`, `USD`, etc. |
| `fx_rate` | DECIMAL(10,6) | `1.0` for TWD |
| `amount_twd` | DECIMAL(15,4) | TWD equivalent |
| `memo` | VARCHAR(255) | |
| `created_at` | DATETIME(3) | Immutable — no `updated_at` |

---

## 13. Operation Log Level

Both BFF APIs are **L1** (financial operations). See `guideline/7-spec-guideline.md` §4.2 for the `OperationLogEvent` specification.

| API | Kafka Topic |
|-----|------------|
| `POST /api/v1/card/pay` | `operation-log.card.pay` |
| `POST /api/v1/card/pay/confirm` | `operation-log.card.pay-confirm` |

Every call — success or failure — must publish an event.

---

## 14. Card Data Security Rules

- Decrypted card data (`pan`, `expiryMMYY`, `cvvItoken`) exists **in JVM Heap only** for the lifetime of a single request
- Must not appear in any log, serialized object, or persisted record
- PAN byte array is zero-filled in a `finally` block after use (Step 24 in authorize)
- `cvvItoken` is never regenerated at auth time — it is fetched from DB via `encryptedCard` and compared directly
- See [guideline/6-pcidss.md](6-pcidss.md) and [guideline/14-3ds-combine-key.md](14-3ds-combine-key.md)

---

## 15. Out of Scope

- **Refund flow** — uses REVERSAL journal entries; defined in `11-refund-flow`
- **Reward plan back-office management** — future feature
- **Production NCCC integration** — mutual TLS, real card data path; defined in production runbook

Next Chapter: [Card Management](/guideline/11-card-management.md)
