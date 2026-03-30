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

> **Future migrations (deferred):** `reward_plan` and `reward_plan_mcc` tables will be added in a future phase when MCC-based bonus calculation is implemented.

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
