# Point & Wallet Management

This document describes how WillCard manages the user's cashback points balance and wallet account.
It covers point issuance, per-batch expiry tracking, FIFO deduction order, the Reserve-Confirm-Release mechanism, and wallet-level management operations.
Implementation details (API contracts, DB schema, UseCase flows) are in the corresponding specs under `spec/wallet-service/` and `spec/mobile-bff/`.

---

## 1. Wallet Account

Each user has a single wallet account managed by **wallet-service**. It holds two balance fields, both in **TWD cents (分)**.

| Field | Type | Role | Description |
|-------|------|------|-------------|
| `available_balance` | BIGINT | **Denormalized cache** | Sum of all non-expired `point_reward_batch.remaining_balance`; updated atomically on every credit/debit event |
| `reserved_balance` | BIGINT | **Display only** | Sum of all PENDING reservation amounts; shown to users as "points locked for in-flight transactions" |

> **Source of truth for balances is `point_reward_batch.remaining_balance`**, not `wallet_account.available_balance`.
> `available_balance` exists for fast validation in Phase 7 (avoids a SUM query on every authorization). It must be kept in sync via the events listed in Section 7.

**Unit:** 1 unit = NT$0.01 (1 TWD cent). NT$100 = 10,000 units.

---

## 2. Points Issuance (How Points Are Awarded)

Points are issued as **cashback** after a card transaction is successfully authorized and settled. Each issuance creates a distinct `point_reward_batch` record with its own expiry date.

### Issuance Timing

| Stage | Event | Action |
|-------|-------|--------|
| Authorization | `txn.card.authorized` Kafka event | Points Service creates a `point_reward_batch` in `PENDING` state |
| T+1 Settlement | Batch settlement job | `point_reward_batch` transitions to `CONFIRMED`; `wallet_account.available_balance` is incremented |

### Expiry Policy

Each batch expires at the **end of the same calendar month one year after issuance**.

| Issued in | `expires_at` |
|-----------|-------------|
| 2026-01 | 2027-01-31 23:59:59 |
| 2026-02 | 2027-02-28 23:59:59 |
| 2026-03 | 2027-03-31 23:59:59 |

> **Phase 3 scope:** `point_reward_batch` and reward calculation logic are defined in Phase 3. Phase 2 defines the wallet data model that receives the credited amount.

### Issuance Formula

```
reward_points = floor(txn_twd_base_amount × base_rate)
```

| Variable | Source |
|----------|--------|
| `txn_twd_base_amount` | Transaction amount in TWD cents, before FX fees |
| `base_rate` | `card_type_limit.domestic_reward_rate` or `overseas_reward_rate` (domestic vs. overseas determined by transaction currency) |

> Points are always calculated on the **pre-FX-fee TWD base amount**. FX fees do not earn rewards.

> **Future enhancement:** The formula will be expanded to `floor(txn_twd_base_amount × (base_rate + mcc_bonus + merchant_bonus))` when MCC-based reward plans and merchant bonuses are introduced. See [guideline/13-reward-plan.md](/guideline/13-reward-plan.md) Section 8.

---

## 3. Points Deduction (How Points Are Used)

The client sends a `usePoints: Boolean` flag with every card pay request. There are no stored per-user preferences.

### Deduction Rule

```
if usePoints == true:
    pointsToUse = min(available_balance, cardAmount)
else:
    pointsToUse = 0
```

| Constraint | Description |
|------------|-------------|
| `available_balance` | Cannot deduct more than the user's total available points |
| `cardAmount` | Cannot offset more than the full transaction amount |

Points are always applied to the **maximum possible amount** when `usePoints = true`. There is no user-configurable per-transaction cap. If the user's points balance is less than the transaction amount, points cover what they can and the remainder is settled via ledger.

### Net Settlement Amount

```
net_settlement_amount = cardAmount - pointsToUse
```

---

## 4. Deduction Ratio

Points are redeemed at a **1:1 ratio** with TWD cents:

```
1 point = NT$0.01 (1 TWD cent)
```

**Quick reference:**

| Available Points | usePoints | Transaction | pointsToUse | Net Settlement |
|-----------------|-----------|-------------|-------------|----------------|
| 10,000 | true | NT$200 (20,000 ¢) | 10,000 | NT$100 |
| 30,000 | true | NT$200 (20,000 ¢) | 20,000 | NT$0 |
| 50,000 | false | NT$200 (20,000 ¢) | 0 | NT$200 |
| 3,000 | true | NT$200 (20,000 ¢) | 3,000 | NT$170 |

---

## 5. Per-Batch Expiry & FIFO Deduction Order

Because each `point_reward_batch` has its own `expires_at`, points must be consumed in **earliest-expiry-first (FIFO) order** to ensure older points are used before they lapse.

### FIFO Deduction Algorithm

```
Input:  userId, totalNeeded
Output: list of (batch_id, amount) pairs

query:
  SELECT batch_id, remaining_balance, expires_at
  FROM point_reward_batch
  WHERE user_id = userId
    AND status = 'CONFIRMED'
    AND remaining_balance > 0
    AND expires_at > NOW()
  ORDER BY expires_at ASC

iterate batches until totalNeeded is satisfied:
  take = min(batch.remaining_balance, remaining_needed)
  append (batch.batch_id, take) to result
  remaining_needed -= take

if remaining_needed > 0: insufficient balance → abort
```

Each `(batch_id, take)` pair becomes one `wallet_reservation_item` row.

### Batch Status Lifecycle

```
PENDING ──► CONFIRMED ──► DEPLETED   (remaining_balance = 0)
                      └──► EXPIRED   (expires_at < NOW(), scheduled job)
```

| Status | Condition |
|--------|-----------|
| `PENDING` | Issued but T+1 settlement not yet run |
| `CONFIRMED` | Available for deduction |
| `DEPLETED` | Fully consumed (`remaining_balance = 0`) |
| `EXPIRED` | Expired before full consumption; residual `remaining_balance` is forfeited |

---

## 6. Reserve-Confirm-Release Pattern

The OTP window creates a time gap between authorization and confirmation. Points must be **locked** during this window to prevent double-spending. The Reserve-Confirm-Release pattern implements this lock at the batch level.

### Data Model

**`wallet_reservation`** — one record per authorization attempt:

| Field | Description |
|-------|-------------|
| `reservation_id` | Snowflake PK |
| `user_id` | Owner |
| `total_amount` | Total points reserved across all batches |
| `status` | `PENDING` → `CONFIRMED` or `RELEASED` |
| `created_at` / `updated_at` | Audit fields |

**`wallet_reservation_item`** — one record per batch consumed by a reservation:

| Field | Description |
|-------|-------------|
| `item_id` | Snowflake PK |
| `reservation_id` | FK to `wallet_reservation` |
| `batch_id` | FK to `point_reward_batch` (which batch was drawn from) |
| `amount` | How many points were drawn from this batch |

### Methods

> **Phase 2 scope:** The three methods below are defined as wallet-service internal service-layer methods. Their HTTP endpoint specs are deferred to **Phase 7**, when card-service requires them during the issuer authorization flow.

| Method | Signature | Steps |
|--------|-----------|-------|
| `reserve` | `reserve(userId, amount) → reservationId` | 1. Run FIFO algorithm → list of (batch_id, amount) pairs<br>2. Decrement each batch's `remaining_balance`<br>3. Create `wallet_reservation` (PENDING) + `wallet_reservation_item` rows<br>4. Decrement `wallet_account.available_balance`, increment `reserved_balance` |
| `confirmDeduct` | `confirmDeduct(reservationId)` | 1. Mark `wallet_reservation` → CONFIRMED<br>2. `reserved_balance` decremented (points permanently consumed — batch `remaining_balance` already decremented at reserve time) |
| `release` | `release(reservationId)` | 1. Query all `wallet_reservation_item` for this reservation<br>2. Restore each item's `amount` back to its `batch.remaining_balance`<br>3. Mark `wallet_reservation` → RELEASED<br>4. `available_balance += total_amount`, `reserved_balance -= total_amount` |

> **Why decrement `remaining_balance` at reserve time?**
> Prevents a concurrent authorization from seeing the same points as available. The FIFO query reads `remaining_balance > 0`, so a reserved batch appears depleted to concurrent requests immediately.

### Balance State After Each Step

```
Initial:  batch A: remaining=8,000   batch B: remaining=5,000
          available_balance = 13,000  reserved_balance = 0

reserve(userId, 10,000):
  FIFO: take 8,000 from A, 2,000 from B
  batch A: remaining = 0 (→ DEPLETED)
  batch B: remaining = 3,000
  reservation_item: [(A, 8000), (B, 2000)]
  available_balance = 3,000   reserved_balance = 10,000

confirmDeduct(reservationId):
  reservation → CONFIRMED
  reserved_balance = 0   (points permanently consumed)

release(reservationId):
  batch A: remaining = 8,000 (restored)
  batch B: remaining = 5,000 (restored)
  available_balance = 13,000  reserved_balance = 0
```

---

## 7. `available_balance` Cache Maintenance

`wallet_account.available_balance` must be updated atomically with every event that changes the true available points.

| Event | `available_balance` | `reserved_balance` |
|-------|--------------------|--------------------|
| Batch PENDING → CONFIRMED (T+1 credit) | `+= issued_amount` | — |
| `reserve()` called | `-= amount` | `+= amount` |
| `confirmDeduct()` called | — | `-= amount` |
| `release()` called | `+= amount` | `-= amount` |
| Expiry job: batch CONFIRMED → EXPIRED | `-= remaining_balance` (before zeroing) | — |

All updates are performed within the same DB transaction as the corresponding `point_reward_batch` or `wallet_reservation` write.

---

## 8. Points Lifecycle During a Transaction

```
Client sends card pay request with usePoints: Boolean
        │
        ├──── usePoints = false ────────────────────────────────────►
        │                  pointsToUse = 0; skip reserve
        │                  card transaction proceeds in full via ledger
        │
        └──── usePoints = true ─────────────────────────────────────►
                           pointsToUse = min(available_balance, cardAmount)
                           │
                           ▼
        [Phase 7 — authorize]
          Wallet.reserve(userId, pointsToUse) → reservationId
          FIFO: decrement earliest-expiring batches' remaining_balance
          Create wallet_reservation + wallet_reservation_item
          available_balance -= pointsToUse
          OTP generated, Redis session saved (TTL 180s)
          Payload: { pointsToUse, reservationId, ... }
                           │
                           ├──── OTP correct ──────────────────────►
                           │     [Phase 7 — verify-challenge]
                           │     Wallet.confirmDeduct(reservationId)
                           │     reservation → CONFIRMED
                           │     reserved_balance -= pointsToUse
                           │     Publish Kafka: txn.card.authorized
                           │           │
                           │           ▼
                           │     [Phase 9 — Saga]
                           │     Points Service creates new point_reward_batch (PENDING)
                           │     T+1: → CONFIRMED; available_balance += reward_points
                           │
                           ├──── OTP wrong (< 3 attempts) ─────────►
                           │     Session alive; reservation unchanged
                           │
                           ├──── OTP wrong (3rd / CARD_LOCKED) ────►
                           │     Wallet.release(reservationId)
                           │     Restore each batch via reservation_item
                           │     available_balance += pointsToUse
                           │
                           └──── Session expired (Redis TTL) ───────►
                                 [Phase 9 — TTL lazy cleanup]
                                 Wallet.release(reservationId)
```

---

## 9. Wallet Management

### APIs

| Operation | Method | Path | Description |
|-----------|--------|------|-------------|
| Balance inquiry | GET | `/api/v1/wallet/balance` | Returns `availableBalance` and `reservedBalance` |
| Points batch list | GET | `/api/v1/wallet/points-batches` | Lists `point_reward_batch` records with `remainingBalance`, `issuedAmount`, `expiresAt`, `status`; sorted by `expiresAt ASC` (Phase 3 addition) |

> There are no stored user preference settings. The `usePoints` flag is supplied per transaction at the BFF/client layer.

### Expiry Cleanup Job

A scheduled job runs daily to expire overdue batches:

```sql
SELECT * FROM point_reward_batch
  WHERE status = 'CONFIRMED'
    AND remaining_balance > 0
    AND expires_at < NOW()

For each expired batch:
  BEGIN TRANSACTION
    UPDATE point_reward_batch SET status = 'EXPIRED', remaining_balance = 0
    UPDATE wallet_account SET available_balance -= old_remaining_balance
      WHERE user_id = batch.user_id
  COMMIT
```

> **Idempotency:** `status = 'CONFIRMED'` in the WHERE clause ensures already-expired batches are not processed twice.

### Scope Boundary

| Concern | Owner |
|---------|-------|
| Aggregate balance cache (`available_balance`, `reserved_balance`) | `wallet_account` |
| Per-batch balance and expiry (`remaining_balance`, `expires_at`) | `point_reward_batch` |
| In-flight locks (which batches, how much) | `wallet_reservation` + `wallet_reservation_item` |
| Reward rate calculation | Points Service / Phase 3 |

All managed within **wallet-service**. The wallet IS the points container in WillCard.

---

## 10. Examples

### Example 1 — Full Deduction (Points Cover Full Transaction)

**Setup:** `available_balance = 80,000` (NT$800), `usePoints = true`
Transaction: NT$300 (30,000 ¢), CLASSIC card (1% cashback)

```
pointsToUse = min(80,000, 30,000) = 30,000
net_settlement = 0
reward_points = floor(30,000 × 0.0100) = 300
```

Points fully cover the transaction. Net charge = NT$0.
`available_balance`: 80,000 → 50,000 (after confirm) → 50,300 (after T+1 reward)

---

### Example 2 — Partial Deduction (Insufficient Points)

**Setup:** `available_balance = 5,000` (NT$50), `usePoints = true`
Transaction: NT$200 (20,000 ¢), OVERSEAS card (5% cashback)

```
pointsToUse = min(5,000, 20,000) = 5,000
net_settlement = 15,000 ¢ (NT$150 via ledger)
reward_points = floor(20,000 × 0.0500) = 1,000
```

Points cover NT$50; NT$150 is settled via ledger.
`available_balance`: 5,000 → 0 (after confirm) → 1,000 (after T+1 reward)

---

### Example 3 — Points Not Used

**Setup:** `available_balance = 50,000`, `usePoints = false`
Transaction: NT$500 (50,000 ¢), PREMIUM card (2% cashback)

```
pointsToUse = 0
net_settlement = 50,000 ¢ (NT$500 via ledger)
reward_points = floor(50,000 × 0.0200) = 1,000
```

`available_balance`: 50,000 → 51,000 (after T+1 reward)

---

### Example 4 — FIFO Across Two Batches

**Setup:** `usePoints = true`, Transaction: NT$80 (8,000 ¢), OVERSEAS card (5%)

| batch_id | remaining_balance | expires_at |
|----------|------------------|------------|
| B1 | 3,000 | 2026-12-31 (earlier) |
| B2 | 10,000 | 2027-03-31 |

```
pointsToUse = min(13,000, 8,000) = 8,000
FIFO: take 3,000 from B1 (→ DEPLETED), 5,000 from B2
reservation_item: [(B1, 3000), (B2, 5000)]
net_settlement = 0
reward_points = floor(8,000 × 0.0500) = 400
```

---

### Example 5 — OTP Failed, Points Released

**Setup:** `available_balance = 13,000`, `usePoints = true`
Transaction: NT$80 (8,000 ¢) → `pointsToUse = 8,000`
Reserve draws: B1 fully (3,000), B2 partially (5,000)

OTP fails 3 times → `release()`:
- B1.remaining_balance restored to 3,000
- B2.remaining_balance restored to 10,000
- `available_balance` restored to 13,000

No net change. FIFO order preserved for next attempt.

---

### Example 6 — Points Expiry

**User has:**
- B1: `remaining_balance = 1,500`, expires tonight
- B2: `remaining_balance = 8,000`, expires 2027-03-31

**Expiry job runs:**
- B1 → `EXPIRED`, `remaining_balance = 0`
- `available_balance`: 9,500 → 8,000

1,500 points in B1 are forfeited. B2 unaffected.

---

## 11. Known Gaps (Phase 2 Scope)

| Gap | Notes |
|-----|-------|
| Expiry grace period | No grace period after `expires_at`; points lapse immediately |
| Expiry notification | No Kafka event or push notification when points are about to expire |
| Points transfer | No API to transfer points between users |
| Points batch list API | Deferred to Phase 3 (requires `point_reward_batch` to be populated) |
| Manual adjustment | No admin API to manually credit or debit points |
| Concurrent batch contention | If a batch is concurrently depleted between FIFO query and UPDATE, reserve must retry; retry logic not specified in Phase 2 |

---

Next Chapter: [Reward Plan](/guideline/13-reward-plan.md)
