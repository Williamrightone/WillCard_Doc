# Points Issuance (Phase 3)

This document describes how WillCard calculates and issues cashback points after a card transaction.
It covers the reward calculation formula, the Points Service Kafka consumer flow, points batch credit, T+1 settlement, and points batch query.
Implementation details (API contracts, DB schema, UseCase flows) are in the corresponding specs under `spec/points-service/`, `spec/wallet-service/`, and `spec/mobile-bff/`.

> **Phase 3 scope:** Points issuance based on `base_rate` only. MCC-based bonus (`mcc_bonus`) and merchant bonus (`merchant_bonus`) are deferred to a future phase. See Section 8 for the planned upgrade direction.

---

## 1. Overview

Phase 3 introduces **Points Service**, which is responsible for:

- Consuming `txn.card.authorized` Kafka events
- Calculating `reward_points` using the card type's `base_rate`
- Calling wallet-service's internal credit API to create `point_reward_batch` records

Wallet-service's T+1 settlement job (defined in Phase 2) confirms PENDING batches to CONFIRMED, incrementing `wallet_account.available_balance`.

---

## 2. Reward Rate

Points are calculated on the **pre-FX-fee TWD base amount** of each transaction:

```
reward_points = floor(txn_twd_base_amount × base_rate)
```

| Variable | Source | Description |
|----------|--------|-------------|
| `txn_twd_base_amount` | `txn.card.authorized` Kafka event | Transaction amount in TWD cents, before FX fees |
| `base_rate` | `card_type_limit.domestic_reward_rate` or `overseas_reward_rate` | Fixed rate per card type; domestic or overseas depending on transaction currency |

**Card type base rates (from Phase 1 seed data):**

| Card Type | Domestic | Overseas |
|-----------|----------|----------|
| CLASSIC | 1% | 0% |
| OVERSEAS | 0% | 5% |
| PREMIUM | 2% | 5% |
| INFINITE | 2% | 5% |

> **FX Note:** Points are calculated on the **pre-FX-fee TWD base amount** only. FX fees do not earn rewards.

> **Floor Note:** `reward_points` is always rounded **down** (`floor`). Fractional points are forfeited.

> **Zero reward:** If `base_rate = 0` (e.g., CLASSIC card on an overseas transaction), `reward_points = 0`. No `point_reward_batch` is created; Points Service records a SKIP in `point_issuance_log`.

---

## 3. Points Issuance Flow

Points Service subscribes to the `txn.card.authorized` Kafka topic and issues points as follows:

```
[Kafka] txn.card.authorized received
    │
    ├── Step 1: Check point_issuance_log for source_txn_id
    │          If already processed → skip (idempotent)
    │
    ├── Step 2: Extract event payload fields:
    │          userId, cardType, isOverseas, txnTwdBaseAmount, txnTimestamp, sourceTxnId
    │
    ├── Step 3: Lookup base_rate from card_type_limit
    │          isOverseas = true  → overseas_reward_rate
    │          isOverseas = false → domestic_reward_rate
    │
    ├── Step 4: Calculate:
    │          reward_points = floor(txnTwdBaseAmount × base_rate)
    │
    ├── Step 5: If reward_points == 0
    │          → insert point_issuance_log (status = SKIP); stop
    │
    └── Step 6: Call wallet-service POST /wallet-service/internal/points/credit
               Request: { userId, sourceTxnId, issuedAmount: reward_points, expiresAt }
               Response: { batchId }
               Success → insert point_issuance_log (status = SUCCESS, batchId)
               Failure → insert point_issuance_log (status = FAILED); publish to DLQ
```

### Expiry Calculation

```
expiresAt = last moment of the same calendar month one year after txnTimestamp
```

| `txnTimestamp` | `expiresAt` |
|---------------|-------------|
| 2026-03-15 | 2027-03-31 23:59:59.999 |
| 2026-12-01 | 2027-12-31 23:59:59.999 |
| 2027-02-10 | 2028-02-29 23:59:59.999 (leap year) |

> Points Service computes `expiresAt` and passes it to the credit API. Wallet-service stores it without modification.

---

## 4. T+1 Settlement

Defined in Phase 2 (wallet-service). Documented here for completeness.

A scheduled job in wallet-service runs nightly to confirm pending point batches:

```sql
SELECT * FROM point_reward_batch
 WHERE status = 'PENDING'
   AND DATE(created_at) < CURDATE()

For each batch:
  BEGIN TRANSACTION
    UPDATE point_reward_batch SET status = 'CONFIRMED', updated_at = NOW()
    UPDATE wallet_account
       SET available_balance = available_balance + issued_amount
     WHERE user_id = batch.user_id
  COMMIT
```

> **Idempotency:** The `status = 'PENDING'` filter ensures already-confirmed batches are never reprocessed.

After confirmation, points become available for deduction in the FIFO algorithm (Phase 2, Section 5).

---

## 5. APIs

### 5.1 Points Batch List (User-Facing)

| Field | Value |
|-------|-------|
| Operation | GET `/api/v1/wallet/points-batches` |
| BFF → Service | wallet-service |
| Purpose | Lists the user's `point_reward_batch` records sorted by `expiresAt ASC` |
| Auth | Required (Bearer JWT) |
| Log Level | L3 |

Query parameters: `page` (0-indexed, default 0), `size` (default 20, max 50).

Spec: [wallet-service/points-batch-list.md](../spec/wallet-service/points-batch-list.md)

### 5.2 Points Credit (Internal)

| Field | Value |
|-------|-------|
| Operation | POST `/wallet-service/internal/points/credit` |
| Caller | points-service only |
| Purpose | Creates a `point_reward_batch` in `PENDING` state |
| Auth | None (internal; Nginx blocks `/internal/` from external traffic) |

Spec: [wallet-service/points-credit.md](../spec/wallet-service/points-credit.md)

---

## 6. Examples

### Example 1 — PREMIUM Card, Domestic Transaction

- Card type: PREMIUM (`domestic_reward_rate = 2%`)
- Transaction: NT$500 (50,000 ¢), issued 2026-03-15

```
reward_points = floor(50,000 × 0.0200) = 1,000
expiresAt     = 2027-03-31 23:59:59
```

`point_reward_batch` created: `issued_amount = 1,000`, `status = PENDING`
After T+1 settlement: `status = CONFIRMED`, `available_balance += 1,000`

---

### Example 2 — OVERSEAS Card, Overseas Transaction

- Card type: OVERSEAS (`overseas_reward_rate = 5%`)
- Transaction: NT$1,000 overseas (100,000 ¢)

```
reward_points = floor(100,000 × 0.0500) = 5,000
```

---

### Example 3 — Zero Reward, No Batch Created

- Card type: CLASSIC (`overseas_reward_rate = 0%`)
- Transaction: NT$300 overseas (30,000 ¢)

```
reward_points = floor(30,000 × 0) = 0
```

Points Service inserts `point_issuance_log (status = SKIP)`. No `point_reward_batch` is created.

---

## 7. Known Gaps (Phase 3 Scope)

| Gap | Notes |
|-----|-------|
| Retry for failed issuance | `point_issuance_log (status = FAILED)` is recorded; automated DLQ retry mechanism not specified |
| Rate caching | `base_rate` lookup via card-service Feign; local cache strategy deferred |
| Points expiry notification | No push or email reminder before `point_reward_batch` expires |

---

## 8. Future Enhancement Direction

The current Phase 3 formula uses `base_rate` only. A future phase will expand the reward calculation to support:

```
reward_points = floor(txn_twd_base_amount × (base_rate + mcc_bonus + merchant_bonus))
```

| Addition | Description |
|----------|-------------|
| `mcc_bonus` | MCC-based bonus rates defined in `reward_plan` and `reward_plan_mcc` tables (points-service). Multiple active plans stack additively. |
| `merchant_bonus` | Merchant-specific bonuses from `card_type_merchant_benefit` (card-service, Phase 1 seed data). |
| Reward plan query API | `GET /api/v1/reward/plans` — lets users see active MCC bonus campaigns before transacting. |
| Reward plan management | Admin API for creating, editing, and deactivating reward plans with validity windows and card-type scopes. |

---
