# Txn Authorized Consumer — points-service Spec

> **Service:** points-service
> **Component Type:** Kafka Consumer
> **Topic:** `txn.card.authorized`
> **Last Updated:** 2026-03

---

## 1. Overview

Consumes `txn.card.authorized` events published by the Transaction Orchestrator. Calculates reward points based on the card type's base rate, then calls wallet-service's internal credit API to create a `PENDING` point batch.

All processing is idempotent via `uk_pil_source_txn_id` in `point_issuance_log`.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Event source (ORCH) | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| Downstream credit call | [wallet-service/points-credit.md](../wallet-service/points-credit.md) |

---

## 3. Message Consumer Definition

### Kafka Binding

| Field | Value |
|-------|-------|
| Topic | `txn.card.authorized` |
| Consumer group | `points-service` |
| Consumer class | `points.service.consumer.TxnAuthorizedConsumer` |

### Event Fields Consumed

| Field | Type | Usage |
|-------|------|-------|
| txnId | String | Idempotency key (`source_txn_id`); also used as `sourceTxnId` for wallet credit |
| userId | Long | Wallet account owner for credit call |
| cardType | String | Lookup key for `reward_plan` rate |
| txnTwdBaseAmount | Long | Base for reward calculation (TWD cents) |
| isOverseas | Boolean | Determines which rate to apply (`overseas_reward_rate` vs `domestic_reward_rate`) |
| txnTimestamp | String | ISO-8601; determines reward batch expiry month |
| pointsToUse | Long | Points redeemed in this transaction — **excluded from reward calculation base** (see note) |

---

## 4. Service UseCase Flow

**UseCase:** `TxnAuthorizedConsumer.consume(TxnCardAuthorizedEvent)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `point_issuance_log` WHERE `source_txn_id = event.txnId`; exists → **ACK** (already processed) |
| 2 | `[DB READ]` | SELECT `reward_plan` WHERE `card_type = event.cardType` |
| 3 | `[DOMAIN]` | Select rate: `rate = isOverseas ? overseas_reward_rate : domestic_reward_rate` |
| 4 | `[DOMAIN]` | Calculate: `reward_points = floor(event.txnTwdBaseAmount × rate)`; if `reward_points == 0` → Step 5a |
| 5a | `[DB WRITE]` | (reward = 0 path) INSERT `point_issuance_log { source_txn_id, user_id, issued_amount: 0, status: 'SKIP' }`; **ACK** |
| 5b | `[DOMAIN]` | (reward > 0 path) Calculate `expiresAt`: last moment of same calendar month one year after `txnTimestamp` (e.g. `2026-03-15T...` → `2027-03-31T23:59:59.999`) |
| 6 | `[FEIGN]` | `WalletFeignClient.credit(PointsCreditRq { userId, sourceTxnId: txnId, issuedAmount: reward_points, expiresAt })`→ `POST /wallet-service/internal/points/credit`; see [wallet-service/points-credit.md](../wallet-service/points-credit.md) |
| 7 | `[DB WRITE]` | INSERT `point_issuance_log { source_txn_id, user_id, batch_id: response.batchId, issued_amount: reward_points, status: 'SUCCESS' }` |
| 8 | `[RETURN]` | **ACK** |

**On credit API error (Step 6 throws):**

| Step | Type | Description |
|------|------|-------------|
| E1 | `[DB WRITE]` | INSERT `point_issuance_log { source_txn_id, user_id, issued_amount: 0, status: 'FAILED' }` |
| E2 | `[RETURN]` | **ACK** (log failure; do not NACK — DLQ not used for business failures) |

> **Reward base excludes points redeemed (`pointsToUse`):** Reward is calculated on the full `txnTwdBaseAmount` — points redemption does not reduce the reward base. This is an intentional business rule (WillCard rewards based on gross transaction amount).
>
> **Idempotency guard (Step 1):** Any re-delivery of the same `txnId` event is safely skipped regardless of prior `status`. The `FAILED` case is also idempotent — a re-delivery after `FAILED` is silently skipped (requires manual DLQ reprocess for retry).

---

## 5. Database

**Tables read:**

| Table | Purpose |
|-------|---------|
| `point_issuance_log` | Idempotency check (Step 1) |
| `reward_plan` | Rate lookup by `cardType` (Step 2) |

**Tables written:**

| Table | Operation | Condition |
|-------|-----------|-----------|
| `point_issuance_log` | INSERT | Always (SKIP, SUCCESS, or FAILED) |

---

## 6. Reward Calculation Reference

| Card Type | Domestic Rate | Overseas Rate |
|-----------|--------------|--------------|
| CLASSIC | 1% | 0% |
| OVERSEAS | 0% | 5% |
| PREMIUM | 2% | 5% |
| INFINITE | 2% | 5% |

> Rates sourced from `reward_plan` table. `floor()` rounding applied — fractional cents are discarded.

---

## 7. Error Handling

| Condition | Behavior |
|-----------|----------|
| `txnId` already in `point_issuance_log` | ACK immediately (idempotent skip) |
| `reward_plan` not found for `cardType` | Log ERROR, INSERT FAILED, ACK |
| `reward_points == 0` | INSERT SKIP, ACK |
| wallet credit API error | INSERT FAILED, ACK (no DLQ; requires manual review) |

---

## 8. Changelog

### v1.0 — 2026-03 — Initial spec
