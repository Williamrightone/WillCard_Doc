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
