# wallet_db — DDL / DML Reference

> **Database:** `wallet_db`
> **Service:** wallet-service
> **Charset:** utf8mb4 / utf8mb4_unicode_ci
> **Migration tool:** Flyway
> **Sync policy:** Update this file whenever a Flyway migration adds/alters/drops a table or column. DML-only migrations (data fixes, seed inserts) do NOT require updates here unless the seed schema changes.

---

## Migration Overview

| File | Type | Description |
|------|------|-------------|
| `V2.0.0__create_wallet_account_table.sql` | DDL | Wallet account per user |
| `V2.0.1__create_wallet_reservation_table.sql` | DDL | Reservation header for in-flight point locks |
| `V2.0.2__create_wallet_reservation_item_table.sql` | DDL | Per-batch detail of each reservation |
| `V2.0.3__create_point_reward_batch_table.sql` | DDL | Per-transaction reward batch with expiry (populated by Phase 3 — Points Service) |

---

## Tables

### `wallet_account`

> **Source migration:** `V2.0.0__create_wallet_account_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `wallet_account` (
  `user_id`           BIGINT        NOT NULL COMMENT 'PK; maps to member.member_id; referential integrity enforced at application layer',
  `available_balance` BIGINT        NOT NULL DEFAULT 0 COMMENT 'Denormalized cache: sum of non-expired point_reward_batch.remaining_balance; updated atomically with each credit/debit event. NOT the authoritative source — point_reward_batch.remaining_balance is.',
  `reserved_balance`  BIGINT        NOT NULL DEFAULT 0 COMMENT 'Display only: sum of PENDING wallet_reservation.total_amount; reflects points locked for in-flight authorizations',
  `created_at`        DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  `updated_at`        DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  PRIMARY KEY (`user_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Creation timing:** One record is inserted per user at registration time (member onboarding flow). `available_balance` and `reserved_balance` start at 0.
>
> **Cache maintenance rules:**
>
> | Event | `available_balance` | `reserved_balance` |
> |-------|--------------------|--------------------|
> | Batch PENDING → CONFIRMED (T+1 credit) | `+= issued_amount` | — |
> | `reserve()` | `-= amount` | `+= amount` |
> | `confirmDeduct()` | — | `-= amount` |
> | `release()` | `+= amount` | `-= amount` |
> | Expiry job: batch → EXPIRED | `-= remaining_balance` (before zeroing) | — |

---

### `wallet_reservation`

> **Source migration:** `V2.0.1__create_wallet_reservation_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `wallet_reservation` (
  `reservation_id`  BIGINT        NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `user_id`         BIGINT        NOT NULL COMMENT 'Owner; referential integrity enforced at application layer',
  `total_amount`    BIGINT        NOT NULL COMMENT 'Total points reserved across all batches in TWD cents',
  `status`          VARCHAR(20)   NOT NULL COMMENT 'PENDING / CONFIRMED / RELEASED',
  `idempotency_key` VARCHAR(64)   NULL     COMMENT 'Caller-supplied deduplication key (ORCH passes challengeRef); NULL allowed for legacy or non-ORCH callers',
  `created_at`      DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  `updated_at`      DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  PRIMARY KEY (`reservation_id`),
  UNIQUE KEY `uk_wr_idempotency_key` (`idempotency_key`),
  KEY `idx_wallet_reservation_user_id` (`user_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Status lifecycle:** `PENDING → CONFIRMED` (on `confirmDeduct`) or `PENDING → RELEASED` (on `release`). Once finalized, the status cannot be modified again.
> **Idempotency:** `idempotency_key` UNIQUE constraint prevents duplicate reservations on ORCH retry. The scheduled cleanup job uses `created_at` to identify orphaned PENDING rows.

---

### `wallet_reservation_item`

> **Source migration:** `V2.0.2__create_wallet_reservation_item_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `wallet_reservation_item` (
  `item_id`        BIGINT        NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `reservation_id` BIGINT        NOT NULL COMMENT 'FK to wallet_reservation (no DB-level FK; enforced at application layer)',
  `batch_id`       BIGINT        NOT NULL COMMENT 'Logical reference to point_reward_batch in same DB; no DB-level FK to allow Phase 3 migration decoupling',
  `amount`         BIGINT        NOT NULL COMMENT 'Points drawn from this batch for this reservation in TWD cents',
  `created_at`     DATETIME(3)   NOT NULL,
  PRIMARY KEY (`item_id`),
  KEY `idx_wri_reservation_id` (`reservation_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Immutability:** `wallet_reservation_item` rows are never updated after insert. No `updated_at` column.
> When `release()` is called, the items are read to restore batch balances, but the item rows themselves remain as an audit record.

---

### `point_reward_batch`

> **Source migration:** `V2.0.3__create_point_reward_batch_table.sql`
> **Last updated:** 2026-03
> **Populated by:** Phase 3 (Points Service) — this table lives in wallet-service's DB but is credited by the Points Service via wallet-service's internal credit API.

```sql
CREATE TABLE `point_reward_batch` (
  `batch_id`          BIGINT        NOT NULL COMMENT 'PK, Snowflake ID (application-generated)',
  `user_id`           BIGINT        NOT NULL COMMENT 'Owner; referential integrity enforced at application layer',
  `source_txn_id`     VARCHAR(64)   NOT NULL COMMENT 'Source transaction ID that generated this reward batch; used for idempotency',
  `issued_amount`     BIGINT        NOT NULL COMMENT 'Total points issued in this batch in TWD cents; immutable after insert',
  `remaining_balance` BIGINT        NOT NULL COMMENT 'Authoritative remaining points available for deduction; decremented by reserve(), restored by release()',
  `status`            VARCHAR(20)   NOT NULL COMMENT 'PENDING / CONFIRMED / DEPLETED / EXPIRED',
  `expires_at`        DATETIME(3)   NOT NULL COMMENT 'Expiry timestamp: end of the same calendar month one year after issuance (e.g., issued 2026-03 → 2027-03-31 23:59:59)',
  `created_at`        DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  `updated_at`        DATETIME(3)   NOT NULL COMMENT 'Inherited from BaseTimeEntity',
  PRIMARY KEY (`batch_id`),
  UNIQUE KEY `uk_prb_source_txn_id` (`source_txn_id`),
  KEY `idx_prb_user_status_expires` (`user_id`, `status`, `expires_at`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Source of truth:** `point_reward_batch.remaining_balance` is the authoritative per-batch available balance. `wallet_account.available_balance` is a derived cache.
>
> **FIFO deduction query:**
> ```sql
> SELECT batch_id, remaining_balance, expires_at
>   FROM point_reward_batch
>  WHERE user_id = ?
>    AND status = 'CONFIRMED'
>    AND remaining_balance > 0
>    AND expires_at > NOW()
>  ORDER BY expires_at ASC;
> ```
>
> **Idempotency:** `uk_prb_source_txn_id` prevents duplicate batch creation for the same transaction.
>
> **Status lifecycle:** `PENDING → CONFIRMED` (T+1 settlement) → `DEPLETED` (remaining_balance = 0) or `EXPIRED` (expiry job).
>
> **Expiry policy:** `expires_at` = last moment of the same calendar month one year after `created_at`.
