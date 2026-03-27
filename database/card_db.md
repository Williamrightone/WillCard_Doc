# card_db — DDL / DML Reference

> **Database:** `card_db`
> **Service:** card-service
> **Charset:** utf8mb4 / utf8mb4_unicode_ci
> **Migration tool:** Flyway
> **Sync policy:** Update this file whenever a Flyway migration adds/alters/drops a table or column. DML-only migrations (data fixes, seed inserts) do NOT require updates here unless the seed schema changes.

---

## Tables

### `card`

> **Source migration:** `V1.0.0__create_card_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `card` (
  `card_id`            BIGINT        NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `user_id`            BIGINT        NOT NULL COMMENT 'Owner member ID; referential integrity enforced at application layer',
  `card_type`          VARCHAR(20)   NOT NULL COMMENT 'CLASSIC / OVERSEAS / PREMIUM / INFINITE',
  `card_network`       VARCHAR(10)   NOT NULL COMMENT 'VISA / MASTERCARD / JCB',
  `status`             VARCHAR(20)   NOT NULL COMMENT 'PENDING / ACTIVE / FROZEN / CANCELLED',
  `pan_encrypted`      VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM encrypted full PAN (PCI DSS)',
  `pan_masked`         VARCHAR(20)   NOT NULL COMMENT 'Display-safe masked PAN, e.g. ****-****-****-1234',
  `pan_hash`           VARCHAR(64)   NOT NULL COMMENT 'HMAC-SHA256 of full PAN; used by Phase 7 issuer auth for PAN-to-member lookup',
  `cardholder_name_zh` VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM encrypted Chinese cardholder name',
  `cardholder_name_en` VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM encrypted English cardholder name (printed on card)',
  `expiry_date`        VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM encrypted expiry in MMYY format; apply date + 5 years',
  `activated_at`       DATETIME(3)   NULL     DEFAULT NULL COMMENT 'Timestamp when card first became ACTIVE; written once, never overwritten',
  `cancelled_at`       DATETIME(3)   NULL     DEFAULT NULL COMMENT 'Timestamp when card was CANCELLED; written once, never overwritten',
  `created_at`         DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  `updated_at`         DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  PRIMARY KEY (`card_id`),
  UNIQUE KEY `uk_card_pan_hash` (`pan_hash`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Application-layer uniqueness constraint:** Only one non-cancelled card per `(user_id, card_type)` is allowed. Enforced by Step 1 of `CardApplyUseCaseImpl` (query WHERE `status != 'CANCELLED'`). No DB-level partial unique index.

> **Encryption note:** `pan_encrypted`, `cardholder_name_zh`, `cardholder_name_en`, `expiry_date` are all AES-256-GCM ciphertext. Decrypted values exist only in JVM heap during request processing and must never be serialized, logged, or persisted.

---

### `card_type_limit`

> **Source migration:** `V1.0.1__create_card_type_limit_table.sql`
> **Seed migration:** `V1.0.2__seed_card_type_limit.sql`
> **Last updated:** 2026-03
> **Mutability:** Seed data — read-only at runtime.

```sql
CREATE TABLE `card_type_limit` (
  `card_type`             VARCHAR(20)   NOT NULL COMMENT 'PK: CLASSIC / OVERSEAS / PREMIUM / INFINITE',
  `single_txn_limit`      BIGINT        NULL     DEFAULT NULL COMMENT 'Per-transaction limit in TWD cents; NULL = no limit',
  `daily_limit`           BIGINT        NULL     DEFAULT NULL COMMENT 'Daily cumulative limit in TWD cents; NULL = no limit',
  `domestic_reward_rate`  DECIMAL(5,4)  NOT NULL COMMENT 'Base domestic cashback rate, e.g. 0.0100 = 1%',
  `overseas_reward_rate`  DECIMAL(5,4)  NOT NULL COMMENT 'Base overseas cashback rate, e.g. 0.0500 = 5%',
  `fx_fee_rate`           DECIMAL(5,4)  NOT NULL COMMENT 'Cross-border transaction fee rate, e.g. 0.0150 = 1.5%',
  `created_at`            DATETIME(3)   NOT NULL,
  `updated_at`            DATETIME(3)   NOT NULL,
  PRIMARY KEY (`card_type`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

**Seed data (`V1.0.2__seed_card_type_limit.sql`):**

```sql
INSERT INTO `card_type_limit`
  (`card_type`, `single_txn_limit`, `daily_limit`, `domestic_reward_rate`, `overseas_reward_rate`, `fx_fee_rate`, `created_at`, `updated_at`)
VALUES
  ('CLASSIC',  10000000, 20000000, 0.0100, 0.0000, 0.0150, NOW(3), NOW(3)),
  ('OVERSEAS', 10000000, 20000000, 0.0000, 0.0500, 0.0100, NOW(3), NOW(3)),
  ('PREMIUM',  20000000, 50000000, 0.0200, 0.0500, 0.0100, NOW(3), NOW(3)),
  ('INFINITE', NULL,     NULL,     0.0200, 0.0500, 0.0100, NOW(3), NOW(3));
```

> **Limit unit:** TWD cents (分). `10000000` = NT$100,000. `NULL` = no limit (INFINITE card type).
>
> **Reward rate layer:** `card_type_limit` provides layer 1 (base rate). Layer 2 (MCC bonus) comes from `reward_plan` (Phase 3). Layer 3 (merchant bonus) comes from `card_type_merchant_benefit`. Final rate = `base + mcc_bonus + merchant_bonus`.

---

### `card_type_merchant_benefit`

> **Source migration:** `V1.0.3__create_card_type_merchant_benefit_table.sql`
> **Seed migration:** `V1.0.4__seed_card_type_merchant_benefit.sql`
> **Last updated:** 2026-03
> **Mutability:** Seed data — read-only at runtime.

```sql
CREATE TABLE `card_type_merchant_benefit` (
  `benefit_id`     BIGINT        NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `card_type`      VARCHAR(20)   NOT NULL COMMENT 'CLASSIC / OVERSEAS / PREMIUM / INFINITE',
  `merchant_id`    VARCHAR(50)   NULL     DEFAULT NULL COMMENT 'Specific merchant ID; NULL = applies to entire MCC category',
  `mcc_code`       VARCHAR(10)   NULL     DEFAULT NULL COMMENT 'MCC category code; NULL = applies to all merchants',
  `bonus_rate`     DECIMAL(5,4)  NOT NULL COMMENT 'Additional bonus rate stacked on top of base + mcc_bonus, e.g. 0.0300 = 3%',
  `description_zh` VARCHAR(255)  NOT NULL COMMENT 'Benefit description in Traditional Chinese',
  `description_en` VARCHAR(255)  NOT NULL COMMENT 'Benefit description in English',
  `effective_from` DATE          NOT NULL COMMENT 'Date from which benefit is active (inclusive)',
  `effective_to`   DATE          NULL     DEFAULT NULL COMMENT 'Date until which benefit is active (inclusive); NULL = indefinite',
  `created_at`     DATETIME(3)   NOT NULL,
  `updated_at`     DATETIME(3)   NOT NULL,
  PRIMARY KEY (`benefit_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

**Seed data (`V1.0.4__seed_card_type_merchant_benefit.sql`):**

```sql
-- benefit_id values are pre-generated Snowflake IDs (application-layer tool or migration helper)
-- Replace placeholder IDs with actual Snowflake IDs before running migration.
-- Rows below are representative examples; populate with actual contracted merchant benefits.

INSERT INTO `card_type_merchant_benefit`
  (`benefit_id`, `card_type`, `merchant_id`, `mcc_code`, `bonus_rate`, `description_zh`, `description_en`, `effective_from`, `effective_to`, `created_at`, `updated_at`)
VALUES
  -- PREMIUM / INFINITE: 3% bonus on all dining (MCC 5812) — indefinite
  (1000000000000000001, 'PREMIUM',  NULL, '5812', 0.0300, '餐飲消費額外 3% 回饋', 'Extra 3% cashback on dining',          '2026-01-01', NULL,         NOW(3), NOW(3)),
  (1000000000000000002, 'INFINITE', NULL, '5812', 0.0300, '餐飲消費額外 3% 回饋', 'Extra 3% cashback on dining',          '2026-01-01', NULL,         NOW(3), NOW(3)),
  -- OVERSEAS: 2% bonus on online shopping (MCC 5945) — indefinite
  (1000000000000000003, 'OVERSEAS', NULL, '5945', 0.0200, '網購消費額外 2% 回饋', 'Extra 2% cashback on online shopping', '2026-01-01', NULL,         NOW(3), NOW(3));
```

> **Query pattern used by `card-benefits` API:**
> ```sql
> SELECT * FROM card_type_merchant_benefit
>  WHERE card_type = ?
>    AND effective_from <= CURDATE()
>    AND (effective_to IS NULL OR effective_to >= CURDATE())
>  ORDER BY effective_from ASC;
> ```
