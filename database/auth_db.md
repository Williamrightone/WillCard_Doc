# auth_db — DDL Reference

> **Database:** `uaa_db`
> **Service:** auth-service
> **Charset:** utf8mb4 / utf8mb4_unicode_ci
> **Migration tool:** Flyway
> **Sync policy:** Update this file whenever a Flyway migration adds/alters/drops a table or column. DML-only migrations (data fixes, seed inserts) do NOT require updates here.

---

## Tables

### `member`

> **Source migration:** `V0.0.1__create_member_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `member` (
  `member_id`  BIGINT                        NOT NULL COMMENT 'PK, Snowflake ID (application-generated, no auto-increment)',
  `account`    VARCHAR(100)                  NOT NULL COMMENT 'Login account',
  `password_hash`     VARCHAR(255)                  NOT NULL COMMENT 'Bcrypt password hash',
  `name`       VARCHAR(100)                  NOT NULL COMMENT 'Display name',
  `role`       ENUM('MEMBER', 'ADMIN')       NOT NULL DEFAULT 'MEMBER' COMMENT 'User role',
  `status`     ENUM('ACTIVE', 'DISABLED')    NOT NULL DEFAULT 'ACTIVE' COMMENT 'Account status',
  `created_at` DATETIME                      NOT NULL COMMENT 'Record creation timestamp',
  `updated_at` DATETIME                      NOT NULL COMMENT 'Last update timestamp',
  PRIMARY KEY (`member_id`),
  UNIQUE KEY `uk_member_account` (`account`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

---

### `member_otp_channel`

> **Source migration:** `V0.0.2__create_member_otp_channel_table.sql`
> **Last updated:** 2026-03

Stores each member's registered OTP delivery channels. Supports multiple channel types per member (one row per channel type). Used by notify-service to determine how to dispatch OTP messages during card-payment authorization.

```sql
CREATE TABLE `member_otp_channel` (
  `channel_id`    BIGINT                         NOT NULL  COMMENT 'PK, Snowflake ID (application-generated)',
  `member_id`     BIGINT                         NOT NULL  COMMENT 'Owner member ID; referential integrity at application layer',
  `channel_type`  ENUM('TELEGRAM', 'SMS')        NOT NULL  COMMENT 'OTP delivery channel type',
  `contact_value` VARCHAR(255)                   NOT NULL  COMMENT 'Telegram chat_id (numeric string) or E.164 phone number; never logged in plaintext',
  `masked_value`  VARCHAR(100)                   NOT NULL  COMMENT 'Log-safe masked value, e.g. ****5678 or tg:****4321',
  `is_primary`    TINYINT(1)                     NOT NULL  DEFAULT 1  COMMENT '1 = preferred channel for this type; at most one primary per member enforced at application layer',
  `status`        ENUM('ACTIVE', 'DISABLED')     NOT NULL  DEFAULT 'ACTIVE',
  `created_at`    DATETIME(3)                    NOT NULL,
  `updated_at`    DATETIME(3)                    NOT NULL,
  PRIMARY KEY (`channel_id`),
  UNIQUE KEY `uk_member_channel_type` (`member_id`, `channel_type`),
  KEY `idx_member_otp_channel_member_id` (`member_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> **Channel type strategy:**
> - `TELEGRAM` — implemented; notify-service sends via Telegram Bot API using `contact_value` as `chat_id`.
> - `SMS` — interface only (stub); `SmsOtpAdapter` logs a warning and does not send.
>
> **contact_value security:** Never include in application logs. Use `masked_value` for all log output.
> **is_primary:** Application layer enforces at most one `ACTIVE + is_primary = 1` row per `(member_id, channel_type)`. No DB-level partial unique index.
