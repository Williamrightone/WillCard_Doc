# Txn Authorized Consumer — ledger-service Spec

> **Service:** ledger-service
> **Component Type:** Kafka Consumer
> **Topic:** `txn.card.authorized`
> **Last Updated:** 2026-03

---

## 1. Overview

Consumes `txn.card.authorized` events and creates double-entry journal records in `ledger_db`. Each authorized transaction generates one or more debit/credit line pairs depending on the transaction type (domestic / overseas / points redemption).

Idempotent via `uk_je_idempotency_key` (`{txnId}_{entrySeq}`) — duplicate Kafka events are silently ignored.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Event source (ORCH) | [txn-orch/card-pay-confirm.md](../txn-orch/card-pay-confirm.md) |
| Database | [ledger_db.md](../../database/ledger_db.md) |

---

## 3. Message Consumer Definition

### Kafka Binding

| Field | Value |
|-------|-------|
| Topic | `txn.card.authorized` |
| Consumer group | `ledger-service` |
| Consumer class | `ledger.service.consumer.TxnAuthorizedConsumer` |

### Event Fields Consumed

| Field | Type | Usage |
|-------|------|-------|
| txnId | String | Idempotency key prefix (`{txnId}_1`, `{txnId}_2` ...) |
| cardType | String | Lookup key for `fx_rate_snapshot` |
| txnTwdBaseAmount | Long | Base amount for all journal entries (TWD cents) |
| isOverseas | Boolean | Determines whether FX fee entry is created |
| pointsToUse | Long | Determines whether points redemption offset entry is created |

---

## 4. Account Codes

| Code | Account Name | Normal Balance | Description |
|------|-------------|---------------|-------------|
| 1001 | Accounts Receivable | Debit | Amount owed by cardholder to WillCard |
| 2001 | Accounts Payable | Credit | Amount owed by WillCard to merchant |
| 3001 | FX Fee Income | Credit | Cross-border transaction fee (overseas only) |
| 4001 | Points Liability | Credit | Points redeemed by cardholder |

---

## 5. Journal Entry Scenarios

### Scenario A — Domestic, No Points Deduction

**Condition:** `isOverseas = false` AND `pointsToUse = 0`

| seq | Account | Debit | Credit |
|-----|---------|-------|--------|
| 1 | 1001 Receivable | `txnTwdBaseAmount` | — |
| 2 | 2001 Payable | — | `txnTwdBaseAmount` |

---

### Scenario B — Domestic, With Points Deduction

**Condition:** `isOverseas = false` AND `pointsToUse > 0`

| seq | Account | Debit | Credit | Amount |
|-----|---------|-------|--------|--------|
| 1 | 1001 Receivable | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 Payable | — | ✓ | `txnTwdBaseAmount` |
| 3 | 4001 Points Liability | — | ✓ | `pointsToUse` |
| 4 | 1001 Receivable | — | ✓ | `pointsToUse` (offset: user pays less cash) |

> Entry 3–4: The points redemption reduces the cardholder's net cash obligation while creating a points liability that WillCard owes on behalf of the cardholder.

---

### Scenario C — Overseas, No Points Deduction

**Condition:** `isOverseas = true` AND `pointsToUse = 0`

Let `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)` where `fx_fee_rate` from `fx_rate_snapshot`.

| seq | Account | Debit | Credit | Amount |
|-----|---------|-------|--------|--------|
| 1 | 1001 Receivable | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 Payable | — | ✓ | `txnTwdBaseAmount - fxFee` |
| 3 | 3001 FX Fee Income | — | ✓ | `fxFee` |

---

### Scenario D — Overseas, With Points Deduction

**Condition:** `isOverseas = true` AND `pointsToUse > 0`

Let `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)`.

| seq | Account | Debit | Credit | Amount |
|-----|---------|-------|--------|--------|
| 1 | 1001 Receivable | ✓ | — | `txnTwdBaseAmount` |
| 2 | 2001 Payable | — | ✓ | `txnTwdBaseAmount - fxFee` |
| 3 | 3001 FX Fee Income | — | ✓ | `fxFee` |
| 4 | 4001 Points Liability | — | ✓ | `pointsToUse` |
| 5 | 1001 Receivable | — | ✓ | `pointsToUse` |

---

## 6. Service UseCase Flow

**UseCase:** `TxnAuthorizedConsumer.consume(TxnCardAuthorizedEvent)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `journal_entry` WHERE `txn_id = event.txnId` LIMIT 1; exists → **ACK** (already processed) |
| 2 | `[DOMAIN]` | Determine scenario (A / B / C / D) based on `isOverseas` and `pointsToUse` |
| 3 | `[DB READ]` | If `isOverseas = true`: SELECT `fx_rate_snapshot` WHERE `card_type = event.cardType`; compute `fxFee = floor(txnTwdBaseAmount × fx_fee_rate)` |
| 4 | `[DOMAIN]` | Build entry list: generate Snowflake `entry_id` and assign `entry_seq` for each line; set `idempotency_key = "{txnId}_{entrySeq}"` |
| 5 | `[DB WRITE]` | Batch INSERT `journal_entry` rows; on duplicate `idempotency_key` → `ON DUPLICATE KEY IGNORE` (safe re-entry) |
| 6 | `[RETURN]` | **ACK** |

> **Atomicity:** All entries for a single `txnId` are inserted in one batch within a single database transaction. Either all succeed or all roll back.

---

## 7. Database

**Tables read:**

| Table | Condition |
|-------|-----------|
| `journal_entry` | Idempotency check: `txn_id = ?` LIMIT 1 |
| `fx_rate_snapshot` | Rate lookup: `card_type = event.cardType` (overseas only) |

**Tables written:**

| Table | Operation | Description |
|-------|-----------|-------------|
| `journal_entry` | Batch INSERT | All scenario entries in single transaction |

---

## 8. Error Handling

| Condition | Behavior |
|-----------|----------|
| `txnId` already has journal entries | ACK immediately (idempotent skip) |
| `fx_rate_snapshot` not found for `cardType` (overseas) | Log ERROR, NACK with DLQ |
| DB insert failure | NACK with DLQ (not a business failure; requires investigation) |

---

## 9. Changelog

### v1.0 — 2026-03 — Initial spec
