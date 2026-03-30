# ledger_db — DDL Reference

> **Database:** `ledger_db`
> **Service:** ledger-service
> **Charset:** utf8mb4 / utf8mb4_unicode_ci
> **Migration tool:** Flyway
> **Sync policy:** Update this file whenever a Flyway migration adds/alters/drops a table or column.

---

## Migration Overview

| File | Type | Description |
|------|------|-------------|
| `V6.0.0__create_journal_entry_table.sql` | DDL | Double-entry journal records for authorized card transactions |
| `V6.0.1__create_fx_rate_snapshot_table.sql` | DDL | FX fee rate snapshot per card type (local copy from card-service) |
| `V6.0.2__seed_fx_rate_snapshot.sql` | DML | Initial rate seed data |

---

## Tables

### `journal_entry`

> **Source migration:** `V6.0.0__create_journal_entry_table.sql`
> **Last updated:** 2026-03

Insert-only double-entry accounting ledger. Each authorized card transaction generates one or more debit/credit line pairs. No `updated_at` — rows are immutable.

```sql
CREATE TABLE `journal_entry` (
  `entry_id`         BIGINT        NOT NULL  COMMENT 'PK, Snowflake ID (application-generated)',
  `txn_id`           VARCHAR(64)   NOT NULL  COMMENT 'Source transaction ID from txn.card.authorized event',
  `entry_seq`        INT           NOT NULL  COMMENT 'Sequential line number within the same txn_id (1, 2, 3 ...)',
  `account_code`     VARCHAR(10)   NOT NULL  COMMENT 'Account code: 1001 Receivable / 1002 Settlement / 2001 Payable / 3001 FX Fee Income / 4001 Points Liability',
  `debit_amount`     BIGINT        NOT NULL  DEFAULT 0  COMMENT 'Debit amount in TWD cents; 0 when this is a credit line',
  `credit_amount`    BIGINT        NOT NULL  DEFAULT 0  COMMENT 'Credit amount in TWD cents; 0 when this is a debit line',
  `currency`         VARCHAR(3)    NOT NULL  DEFAULT 'TWD'  COMMENT 'ISO 4217; always TWD (all amounts pre-converted by ORCH)',
  `idempotency_key`  VARCHAR(128)  NOT NULL  COMMENT 'Unique key: {txnId}_{entrySeq}; prevents duplicate journal entries on Kafka replay',
  `created_at`       DATETIME(3)   NOT NULL,
  PRIMARY KEY (`entry_id`),
  UNIQUE KEY `uk_je_idempotency_key` (`idempotency_key`),
  KEY `idx_je_txn_id` (`txn_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Immutability:** Rows are insert-only. No `updated_at` column. Corrections are made via reversal entries, not updates.
>
> **Idempotency:** `uk_je_idempotency_key` (`{txnId}_{entrySeq}`) prevents duplicate insertions on Kafka event replay. Duplicate INSERT → `ON DUPLICATE KEY IGNORE` or application-level skip.
>
> **Account codes:**
>
> | Code | Name | Description |
> |------|------|-------------|
> | 1001 | Accounts Receivable | Amount owed by cardholder to WillCard |
> | 1002 | Settlement Receivable | Amount awaiting clearing (T+1 settlement entry) |
> | 2001 | Accounts Payable | Amount owed by WillCard to merchant |
> | 3001 | FX Fee Income | Cross-border transaction fee earned by WillCard |
> | 4001 | Points Liability | Points redeemed by cardholder (offset against receivable) |

---

### `fx_rate_snapshot`

> **Source migration:** `V6.0.1__create_fx_rate_snapshot_table.sql`
> **Seed migration:** `V6.0.2__seed_fx_rate_snapshot.sql`
> **Last updated:** 2026-03
> **Mutability:** Seed data — read-only at runtime. Updated via migration when rates change.

Local copy of FX fee rates per card type. Decoupled from `card_type_limit` in `card_db` to avoid cross-service DB access.

```sql
CREATE TABLE `fx_rate_snapshot` (
  `card_type`    VARCHAR(20)   NOT NULL  COMMENT 'PK: CLASSIC / OVERSEAS / PREMIUM / INFINITE',
  `fx_fee_rate`  DECIMAL(5,4)  NOT NULL  COMMENT 'Cross-border transaction fee rate, e.g. 0.0150 = 1.5%',
  `created_at`   DATETIME(3)   NOT NULL,
  `updated_at`   DATETIME(3)   NOT NULL,
  PRIMARY KEY (`card_type`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

**Seed data (`V6.0.2__seed_fx_rate_snapshot.sql`):**

```sql
INSERT INTO `fx_rate_snapshot` (`card_type`, `fx_fee_rate`, `created_at`, `updated_at`)
VALUES
  ('CLASSIC',  0.0150, NOW(3), NOW(3)),
  ('OVERSEAS', 0.0100, NOW(3), NOW(3)),
  ('PREMIUM',  0.0100, NOW(3), NOW(3)),
  ('INFINITE', 0.0100, NOW(3), NOW(3));
```

> **Usage:** Ledger Service reads `fx_fee_rate` by `cardType` when `isOverseas = true` to compute the FX Fee Income journal entry.
