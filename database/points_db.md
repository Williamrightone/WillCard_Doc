# points_db — DDL / DML Reference

> **Database:** `points_db`
> **Service:** points-service
> **Charset:** utf8mb4 / utf8mb4_unicode_ci
> **Migration tool:** Flyway
> **Sync policy:** Update this file whenever a Flyway migration adds/alters/drops a table or column. DML-only migrations (data fixes, seed inserts) do NOT require updates here unless the seed schema changes.

---

## Migration Overview

| File | Type | Description |
|------|------|-------------|
| `V3.0.0__create_point_issuance_log_table.sql` | DDL | Idempotency and audit log for Kafka-driven point issuance events |
| `V3.0.1__create_reward_plan_table.sql` | DDL | Base reward rates per card type |
| `V3.0.2__seed_reward_plan.sql` | DML | Initial rate seed data (mirrors `card_type_limit` base rates) |

> **Deferred:** `reward_plan_mcc` (MCC-based bonus rates) will be added in a future phase.

---

## Tables

### `point_issuance_log`

> **Source migration:** `V3.0.0__create_point_issuance_log_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `point_issuance_log` (
  `log_id`         BIGINT       NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `source_txn_id`  VARCHAR(64)  NOT NULL COMMENT 'Source transaction ID from Kafka event; idempotency key — prevents duplicate processing',
  `user_id`        BIGINT       NOT NULL COMMENT 'Owner; referential integrity enforced at application layer',
  `batch_id`       BIGINT       NULL COMMENT 'References point_reward_batch in wallet_db; NULL when status = SKIP or FAILED (no batch was created)',
  `issued_amount`  BIGINT       NOT NULL DEFAULT 0 COMMENT 'Points issued in TWD cents; 0 when status = SKIP or FAILED',
  `status`         VARCHAR(20)  NOT NULL COMMENT 'SUCCESS: batch created in wallet-service; SKIP: reward_points = 0; FAILED: credit API error',
  `created_at`     DATETIME(3)  NOT NULL,
  PRIMARY KEY (`log_id`),
  UNIQUE KEY `uk_pil_source_txn_id` (`source_txn_id`),
  KEY `idx_pil_user_id` (`user_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Idempotency:** Before processing any `txn.card.authorized` event, Points Service checks `uk_pil_source_txn_id`. If a row already exists, the event is skipped regardless of its `status`.
>
> **Status meanings:**
>
> | Status | Condition |
> |--------|-----------|
> | `SUCCESS` | `reward_points > 0` and wallet-service credit API returned HTTP 200 |
> | `SKIP` | `reward_points = 0` (zero base_rate for this card type / transaction type); no `point_reward_batch` created |
> | `FAILED` | Credit API returned an error; requires manual review or DLQ reprocessing |
>
> **Immutability:** Rows are insert-only. No `updated_at` column.
>
> **Cross-service reference:** `batch_id` references `point_reward_batch.batch_id` in `wallet_db`. No DB-level FK across databases.

---

### `reward_plan`

> **Source migration:** `V3.0.1__create_reward_plan_table.sql`
> **Seed migration:** `V3.0.2__seed_reward_plan.sql`
> **Last updated:** 2026-03
> **Mutability:** Seed data — read-only at runtime. One row per card type.

Points Service's local copy of base reward rates. Decoupled from `card_type_limit` in `card_db` to avoid cross-service DB access. Must be kept in sync with `card_type_limit` via manual migration when rates change.

```sql
CREATE TABLE `reward_plan` (
  `card_type`             VARCHAR(20)   NOT NULL  COMMENT 'PK: CLASSIC / OVERSEAS / PREMIUM / INFINITE',
  `domestic_reward_rate`  DECIMAL(5,4)  NOT NULL  COMMENT 'Base domestic cashback rate, e.g. 0.0100 = 1%',
  `overseas_reward_rate`  DECIMAL(5,4)  NOT NULL  COMMENT 'Base overseas cashback rate, e.g. 0.0500 = 5%',
  `created_at`            DATETIME(3)   NOT NULL,
  `updated_at`            DATETIME(3)   NOT NULL,
  PRIMARY KEY (`card_type`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

**Seed data (`V3.0.2__seed_reward_plan.sql`):**

```sql
INSERT INTO `reward_plan`
  (`card_type`, `domestic_reward_rate`, `overseas_reward_rate`, `created_at`, `updated_at`)
VALUES
  ('CLASSIC',  0.0100, 0.0000, NOW(3), NOW(3)),
  ('OVERSEAS', 0.0000, 0.0500, NOW(3), NOW(3)),
  ('PREMIUM',  0.0200, 0.0500, NOW(3), NOW(3)),
  ('INFINITE', 0.0200, 0.0500, NOW(3), NOW(3));
```

> **Rate selection:** `isOverseas = true` → use `overseas_reward_rate`; `isOverseas = false` → use `domestic_reward_rate`.
> **OVERSEAS domestic rate is 0:** OVERSEAS card type earns no domestic cashback (only overseas). If `reward_points` rounds to 0, Points Service logs `SKIP` and issues no batch.
