# Transaction Flow

This chapter defines the business rules, flow design, and conventions for credit card transactions in WillCard. All specs under `spec/mobile-bff/card-pay*`, `spec/card-service/`, `spec/wallet-service/`, `spec/points-service/`, and `spec/ledger-service/` must conform to these rules.

---

## 1. WillCard as Card Issuer

WillCard issues virtual Visa/Mastercard cards backed by the member's e-wallet balance. In Taiwan's card payment ecosystem, **NCCC (聯合信用卡處理中心 / 財金公司)** acts as the card network and clearing house.

**WillCard is the Issuer. Authorization requests flow inward — NCCC calls WillCard, not the other way around.**

```
[Production — inbound]
Cardholder uses virtual card at merchant
  → Merchant terminal / checkout
  → Acquiring bank
  → NCCC
  → WillCard Card Service (Issuer Authorization API)   ← called here
       ↓ on approval (async)
  Internal Saga: Ledger · Points · Notification

[Development — Mock NCCC]
App → BFF → Mock NCCC Service → WillCard Card Service (same auth path)
```

**Key identifiers:**

| Field | Produced by | Scope | Purpose |
|-------|------------|-------|---------|
| `challengeRef` | Card Service (Phase 1) | Auth session | OTP binding; ties Phase 1 and Phase 2 together |
| `txnId` | Card Service (Phase 2, on approval) | Transaction record | Ledger, Kafka events, reconciliation |

---

## 2. Issuer Authorization API (Card Service)

Card Service exposes two endpoints that **NCCC calls directly** — not routed through BFF.

| Endpoint | Caller | Purpose |
|----------|--------|---------|
| `POST /card/auth/authorize` | NCCC | Initial auth request — validate card, check funds, trigger 3DS OTP |
| `POST /card/auth/verify-challenge` | NCCC | OTP verification — final approval / decline |

Security: Production uses mutual TLS with NCCC client certificate. Development uses Mock NCCC on internal network.

### 2.1 Phase 1 — authorize

```
NCCC → POST /card/auth/authorize
  Request:
    encryptedCard     — card data encrypted by NCCC using combineKey
    amount            — transaction amount
    currency          — TWD / USD / ...
    merchantId
    merchantMCC       — used for reward rate lookup

  Card Service steps:
    1. Decrypt encryptedCard (in-memory combineKey)
    2. Identify member from PAN
    3. Validate card: ACTIVE, not expired
    4. Enforce transaction limits (single txn + daily cumulative)
    5. Read member's points preference → calculate pointsToUse (see §4)
    6. Check funds: (wallet_balance - pointsToUse) >= cardAmount
    7. Reserve points tentatively: Wallet Service.reserve(userId, pointsToUse)
    8. Lock FX rate if currency ≠ TWD: FX Service.lockRate() → fxRateId
    9. Generate 6-digit OTP → SMS to member's registered phone
   10. Store pending auth in Redis: otp:{challengeRef} TTL 180s
       Payload: { txnDetails, pointsToUse, reservationId, fxRateId? }

  Response:
    CHALLENGE_REQUIRED  + challengeRef + pointsPreview
    DECLINED            + declineReason
```

### 2.2 Phase 2 — verify-challenge

```
NCCC → POST /card/auth/verify-challenge
  Request:
    challengeRef
    otp               — submitted by cardholder

  Card Service steps:
    1. Load pending auth from Redis otp:{challengeRef}
    2. Verify OTP value
    3a. OTP valid:
        → Wallet Service.confirmDeduct(reservationId)
        → Assign txnId (Snowflake)
        → Publish txn.card.authorized Kafka event (async Saga)
        → Clear Redis key
    3b. OTP invalid:
        → Increment otp:attempt:{challengeRef} (per-session)
        → Increment otp:card:fail:{cardId}     (per-card, TTL 900s)
        → Check thresholds (see §2.3)

  Response (OTP valid):
    APPROVED  + authCode + txnId + pointsUsed + cardAmount
  Response (OTP invalid):
    DECLINED  + OTP_FAILED + attemptsRemaining
  Response (card locked):
    DECLINED  + CARD_LOCKED
```

### 2.3 OTP Failure Thresholds

Two independent layers protect against accidental retry and card theft probing.

**Layer 1 — Per-challengeRef (session-level)**

| Item | Value |
|------|-------|
| Redis key | `otp:attempt:{challengeRef}` (TTL 180s) |
| Max retries | 3 |
| On limit reached | This authorization cancelled; card unaffected |

**Layer 2 — Per-card sliding window (fraud detection)**

| Item | Value |
|------|-------|
| Redis key | `otp:card:fail:{cardId}` (TTL 900s / 15 min) |
| Threshold | **5 failures within 15 minutes** (across any challengeRef) |
| On threshold reached | Temporary card lock + publish `card.risk.otp-threshold-exceeded` → Risk Control |
| Lock duration | Defined by Risk Control team |

Each failure increments **both** counters simultaneously.

### 2.4 Card Data Rules

See `guideline/6-pcidss.md` §2.2 for the full combineKey / encryptedCard specification.

- Decrypted card data lives **in JVM Heap only** — never serialized or persisted
- CVV discarded immediately after authorization
- PAN stored as masked format only: `****-****-****-1234`

---

## 3. Mock NCCC (Development & Testing)

Mock NCCC is a **development-only** service that simulates NCCC's inbound authorization calls, enabling end-to-end testing without a real NCCC connection.

### 3.1 Architecture

```
App
  │  POST /api/v1/card/pay  (via BFF)
  ↓
Mock NCCC Service
  ├─ Calls Card Service: POST /card/auth/authorize
  │     ← CHALLENGE_REQUIRED + challengeRef + pointsPreview
  └─ Returns challengeRef + pointsPreview to App

App displays LinePay-style points offset preview + OTP input screen
  │  POST /api/v1/card/pay/confirm  (via BFF)
  ↓
Mock NCCC Service
  └─ Calls Card Service: POST /card/auth/verify-challenge
        ← APPROVED / DECLINED

Mock NCCC returns final result to App
```

### 3.2 API Contract

**POST /mock-nccc/pay** — Initiate payment

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cardId` | String | ✅ | Member's card identifier |
| `amount` | BigDecimal | ✅ | Transaction amount |
| `currency` | String | ✅ | `TWD` / `USD` / … |
| `merchantId` | String | ✅ | Merchant identifier |

Response:

| Field | Type | Description |
|-------|------|-------------|
| `challengeRef` | String | Present when 3DS required |
| `status` | String | `CHALLENGE_REQUIRED` / `APPROVED` / `DECLINED` |
| `pointsPreview.pointsToUse` | Integer | Server-computed from member preference |
| `pointsPreview.cardAmount` | BigDecimal | Amount charged to card |

**POST /mock-nccc/pay/confirm** — Submit OTP

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `challengeRef` | String | ✅ | From pay response |
| `otp` | String | ✅ | 6-digit OTP from SMS |

Response:

| Field | Type | Description |
|-------|------|-------------|
| `authCode` | String | Present on approval |
| `txnId` | String | Snowflake transaction ID |
| `status` | String | `APPROVED` / `DECLINED` |
| `declineReason` | String | Present on decline |

### 3.3 Points Preview

`pointsToUse` is calculated server-side in Phase 1 (from member preference) and surfaced as `pointsPreview` in the Mock NCCC response. The App uses this to display the offset summary screen — the value is **informational only**; the actual reserve is already set in Phase 1.

---

## 4. Points-First Preference

### 4.1 Preference Storage

Points-first is a **per-member setting** managed by Wallet Service.

**Table: `member_points_preference` (Wallet Service DB)**

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | BIGINT PK | |
| `points_first_enabled` | BOOLEAN | Auto-apply points offset on every transaction |
| `max_points_per_txn` | INT | Per-transaction cap (NULL = no cap) |
| `updated_at` | DATETIME | |

### 4.2 Auto-Calculation at Authorization

Card Service reads the preference during Phase 1:

```
if points_first_enabled:
  pointsToUse = min(
    available_points_balance,
    amount_in_twd,
    max_points_per_txn ?? ∞
  )
else:
  pointsToUse = 0

cardAmount = amount_in_twd - pointsToUse
```

### 4.3 Points Exchange Rules

| Rule | Value |
|------|-------|
| Exchange rate | **1 point = NT$1** (1:1) |
| Minimum | 1 point |
| Maximum | Full amount (100% offset) |
| Both `points_used` and `card_amount` | Must be recorded on the transaction |

### 4.4 Reserve-Confirm-Release

```
Phase 1 (authorize):
  Wallet Service.reserve(userId, pointsToUse)
  → available_balance -= pointsToUse
  → reserved_balance  += pointsToUse
  Reserve TTL = 180s (aligns with OTP TTL, lazy cleanup on expiry)

Phase 2 — APPROVED:
  Wallet Service.confirmDeduct(reservationId)
  → reserved_balance -= pointsToUse

Compensation (OTP failure / DECLINED / TTL expiry):
  Wallet Service.release(reservationId)
  → reserved_balance -= pointsToUse
  → available_balance += pointsToUse
```

---

## 5. Saga Compensation Rules

| Saga Step | Compensation Action | Trigger |
|-----------|--------------------|---------|
| Points reserve | `release(reservationId)` | OTP failure, auth failure, TTL expiry (lazy) |
| FX rate lock | Auto-expires via TTL | No active compensation needed |
| Ledger entries | Write REVERSAL entries (never delete) | Refund flow (separate spec) |

Ledger entries are **immutable**. Compensation at the ledger level is always a new reversal entry, never an update or delete.

---

## 5. Reward Calculation Rules

### 5.1 Reward Basis

- Reward is calculated on **card_amount only** (points-offset portion is excluded)
- Result is **floor (truncate)** — no rounding up
- For foreign currency transactions, use the **TWD-converted card amount** as the basis

```
reward_points = floor(card_amount_twd × reward_rate)

Example: card_amount = 60, reward_rate = 2%
  60 × 0.02 = 1.2 → floor → 1 point
```

### 5.2 Reward Plan (MCC-based)

Reward rates are stored in the `reward_plan` table (Points Service DB) and configurable via back-office. The management UI is a future feature; the table must exist from day one.

**Cache loading strategy:** Points Service loads all active reward plan rows into an in-memory `Map<String, BigDecimal>` (keyed by `mcc_code`) at startup (`ApplicationReadyEvent`). The map is refreshed only when a plan record is created or updated (cache invalidation on write). Read path never hits the DB.

**Table: `reward_plan`**

| Column | Type | Description |
|--------|------|-------------|
| `plan_id` | BIGINT PK | Snowflake |
| `mcc_code` | VARCHAR(10) | MCC code; `DEFAULT` for catch-all |
| `reward_rate` | DECIMAL(5,4) | e.g., `0.0200` for 2% |
| `description` | VARCHAR(100) | Human-readable label |
| `effective_from` | DATE | Plan effective date |
| `effective_to` | DATE | Plan expiry date (null = no expiry) |
| `created_at` | DATETIME | |

Initial MCC rates are seeded via Flyway DML. If no MCC match is found, fall back to `mcc_code = 'DEFAULT'`.

### 5.3 Reward Point Batch

Every reward issuance creates one `point_reward_batch` record. Each batch tracks its own remaining balance and expiry independently, enabling accurate FIFO deduction and expiry management.

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

## 6. Foreign Currency Rules

| Rule | Value |
|------|-------|
| TWD transactions | **No handling fee** |
| FX handling fee | **1.5%** of `twd_base`, added on top |
| Rate lock | FX Service `lockRate()` called in Step 1; returns `fxRateId` (TTL: 10 min) |
| Ledger currency | Always record in TWD; store original currency + rate for reference |
| Reward basis | **`twd_base` only** — FX fee is excluded from reward calculation |
| Merchant fee basis | **`twd_base` only** — FX fee is WillCard's own charge to the cardholder and does not form part of the merchant service; merchant settlement is calculated solely on `twd_base` |

**FX amount calculation:**

```
twd_base      = foreign_amount × fx_rate
fx_fee        = floor(twd_base × 0.015)
total_twd     = twd_base + fx_fee        ← amount charged to member's card
reward_points = floor(twd_base × reward_rate)  ← reward based on twd_base, not total_twd
```

Example: USD$100, rate 31.0, reward rate 1%
```
twd_base      = 100 × 31.0  = 3,100
fx_fee        = floor(3,100 × 0.015) = 46
total_twd     = 3,100 + 46 = 3,146     ← member is charged TWD$3,146
reward_points = floor(3,100 × 0.01)  = 31 points
```

---

## 7. Transaction Limits

Limits are enforced per card type and stored in `card_type_limit` (Card Service DB).

| Default | Value |
|---------|-------|
| Single transaction | NT$100,000 |
| Daily cumulative | NT$200,000 |

**Table: `card_type_limit`**

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT PK | |
| `card_type` | VARCHAR | e.g., `STANDARD`, `GOLD`, `PLATINUM` |
| `single_txn_limit` | DECIMAL(15,2) | |
| `daily_limit` | DECIMAL(15,2) | |
| `currency` | VARCHAR(3) | `TWD` |
| `updated_at` | DATETIME | |

Card Service validates both limits before generating the OTP in Step 1.

---

## 8. Double-Entry Ledger Rules

### 8.1 Principles

- Ledger entries are **immutable** — INSERT only, no UPDATE or DELETE
- Every transaction produces a balanced set of DEBIT / CREDIT entries
- Compensation is always a new REVERSAL entry set

### 8.2 Account Structure

**Assets (資產)**

| Code | Account | Description |
|------|---------|-------------|
| `1001` | Member Receivable（應收會員款）| Amount charged to the member's virtual card at authorization |
| `1002` | Settlement Reserve（清算備付金）| Cash pool used for merchant settlement; debited at T+1 settlement |

**Liabilities (負債)**

| Code | Account | Description |
|------|---------|-------------|
| `2001` | Merchant Payable（應付特店款）| Net amount owed to merchant after deducting service fee; also receives points-offset subsidy |
| `2002` | Points Liability（點數負債）| Issued but unredeemed reward points (1 pt = NT$1); drawn down on redemption |

**Revenue (收入)**

| Code | Account | Description |
|------|---------|-------------|
| `3001` | Merchant Fee Income（特店手續費收入）| Service fee charged to merchant on every TWD transaction |
| `3002` | FX Fee Income（外幣手續費收入）| 1.5% handling fee on foreign currency transactions |

**Expense (費用)**

| Code | Account | Description |
|------|---------|-------------|
| `4001` | Reward Points Expense（點數回饋費用）| Cost of issuing reward points to members |

**Other (其他)**

| Code | Account | Description |
|------|---------|-------------|
| `5001` | FX Gain/Loss（匯差損益）| Exchange rate difference when WillCard holds FX position |

### 8.3 Journal Entry Types

| `journal_type` | Trigger |
|---------------|---------|
| `AUTHORIZATION` | NCCC authorization success |
| `SETTLEMENT` | T+1 batch settlement — cash moves: DR 2001 / CR 1002 |
| `REVERSAL` | Refund (separate spec) |
| `REWARD` | Reward points issuance (async, via Kafka event) |

### 8.4 Entry Examples

**Case 1: TWD NT$100, no offset, reward 1%, merchant fee 3%**

| Debit | Amt | Credit | Amt | Note |
|-------|-----|--------|-----|------|
| Member Receivable (1001) | 100.00 | Merchant Payable (2001) | 97.00 | Authorization |
| | | Merchant Fee Income (3001) | 3.00 | |
| Reward Expense (4001) | 1.00 | Points Liability (2002) | 1.00 | 1 point — floor(100×1%) |

**Case 2: TWD NT$100, 34 pts offset + NT$66 card, reward 1%, merchant fee 3%**

| Debit | Amt | Credit | Amt | Note |
|-------|-----|--------|-----|------|
| Member Receivable (1001) | 66.00 | Merchant Payable (2001) | 64.02 | Card portion |
| | | Merchant Fee Income (3001) | 1.98 | fee on card portion |
| Points Liability (2002) | 34.00 | Merchant Payable (2001) | 34.00 | Points subsidy to merchant |
| Reward Expense (4001) | 0.00 | Points Liability (2002) | 0.00 | floor(66×1%) = 0 pts |

> Merchant still receives NT$100 in total (64.02 + 1.98 fee retained + 34.00 subsidy = NT$100 gross).

**Case 3: Foreign USD$100, rate 31.0, reward 1%, merchant fee 3%**

```
twd_base = 3,100 | fx_fee = 46 | total_twd = 3,146 | reward = floor(3,100×1%) = 31 pts
```

| Debit | Amt | Credit | Amt | Note |
|-------|-----|--------|-----|------|
| Member Receivable (1001) | 3,146.00 | Merchant Payable (2001) | 3,007.00 | twd_base × 0.97 |
| | | Merchant Fee Income (3001) | 93.00 | twd_base × 3% |
| | | FX Fee Income (3002) | 46.00 | twd_base × 1.5% |
| Reward Expense (4001) | 31.00 | Points Liability (2002) | 31.00 | floor(twd_base × 1%) |

### 8.5 `journal_entry` Schema

| Column | Type | Note |
|--------|------|------|
| `entry_id` | BIGINT PK | Snowflake |
| `idempotency_key` | VARCHAR(100) UNIQUE | `{kafka_message_key}_{entry_seq}` — prevents duplicate writes on Kafka consumer retry |
| `txn_id` | BIGINT | Source transaction |
| `journal_type` | VARCHAR(20) | See §8.3 |
| `entry_type` | VARCHAR(10) | `DEBIT` / `CREDIT` |
| `account_code` | VARCHAR(10) | See §8.2 |
| `amount` | DECIMAL(15,4) | |
| `currency` | VARCHAR(3) | `TWD`, `USD`, etc. |
| `fx_rate` | DECIMAL(10,6) | `1.0` for TWD transactions |
| `amount_twd` | DECIMAL(15,4) | TWD equivalent |
| `memo` | VARCHAR(255) | |
| `created_at` | DATETIME(3) | Immutable — no `updated_at` |

**Idempotency design:**
- The Kafka event payload for `txn.card.authorized` must include a stable `messageKey` (e.g., Snowflake-generated at event publish time).
- Ledger Service assigns `entry_seq` (0, 1, 2 …) to each entry within one event, forming `idempotency_key = {messageKey}_{entry_seq}`.
- On retry, the `INSERT` hits the unique constraint and is discarded — no duplicate entries, no exception propagation needed (handle as upsert-ignore or catch duplicate key).

---

## 9. Operation Log Level

Both APIs are **L1** (financial operation). See `guideline/7-spec-guideline.md` §4.2 for the full `OperationLogEvent` specification.

| API | Kafka Topic |
|-----|------------|
| `card-pay` (Step 1) | `operation-log.card.pay` |
| `card-pay/confirm` (Step 2) | `operation-log.card.pay-confirm` |

Every call — success or failure — must publish an event.

---

## 10. Out of Scope

The following are explicitly **not** covered by this chapter and will be defined in separate specs:

- **Refund flow** — uses REVERSAL journal entries; defined in `11-refund-flow`
- **Reward plan back-office management** — future feature
- **Card type management** — defined with Card Service onboarding spec

Next chapter: [Refund Flow](/guideline/11-refund-flow.md) *(pending)*
