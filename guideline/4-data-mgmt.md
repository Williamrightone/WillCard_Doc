# Data Management

## 0. Database Environment

### 0.1 MySQL Version

The system runs on **MySQL 8.0** for the following reasons:

- Native support for Window Functions and CTEs (`WITH` syntax), simplifying complex queries
- Improved InnoDB performance and concurrency handling
- Descending index support, beneficial for monotonically increasing Snowflake IDs
- More complete JSON column support

### 0.2 Character Set and Collation Configuration

All databases, tables, and columns use the following unified configuration:

| Item | Value |
|---|---|
| Character set | `utf8mb4` |
| Default collation | `utf8mb4_unicode_ci` |
| Exception field collation | `utf8mb4_bin` |

`utf8mb4` provides full Unicode support (including emoji), avoiding the 3-byte limitation of the legacy `utf8` charset.

`utf8mb4_unicode_ci` conforms to the full Unicode comparison standard and is case-insensitive, making it suitable for general business fields such as usernames and email addresses.

Fields that require strict case sensitivity (e.g. tokens, password hashes, fixed-format codes) should override the collation at the column level using `utf8mb4_bin`.

MySQL server-level configuration:

```ini
[mysqld]
character-set-server  = utf8mb4
collation-server      = utf8mb4_unicode_ci
```

Flyway migration DDL should declare charset and collation explicitly:

```sql
CREATE TABLE user_account (
    id            BIGINT       NOT NULL,
    email         VARCHAR(255) NOT NULL,
    token         VARCHAR(512) NOT NULL COLLATE utf8mb4_bin,
    created_at    DATETIME(3)  NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

The Spring Datasource connection URL must also specify the charset explicitly:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/auth_db?useUnicode=true&characterEncoding=utf8mb4&connectionCollation=utf8mb4_unicode_ci
```

---

## 1. Primary Key Strategy — Snowflake ID

### 1.1 Motivation

The system uses Snowflake ID as the global primary key for the following reasons:

- Globally unique, avoiding ID collisions across sharded databases
- Monotonically increasing, B-Tree index friendly, reducing page splits
- Generated entirely in the application layer, eliminating DB auto-increment bottlenecks across services
- Embeds timestamp information, allowing the creation time to be derived from the ID itself

Tables with low data volume that do not require sharding (e.g. RBAC-related tables) are not required to use Snowflake ID.

---

### 1.2 ID Structure

A Snowflake ID is a 64-bit signed long integer with the following layout:

| 1 bit | 41 bits | 5 bits | 5 bits | 12 bits |
|---|---|---|---|---|
| Sign bit | Timestamp (ms) | DC ID | Pod ID | Sequence |

| Field | Bits | Description |
|---|---|---|
| Sign bit | 1 bit | Always 0, ensures the ID is a positive number |
| Timestamp | 41 bits | Milliseconds since the custom epoch; supports ~69 years |
| DC ID | 5 bits | Data center identifier, range 1–31, mapped to physical DCs (e.g. Taipei / Kaohsiung) |
| Pod ID | 5 bits | Pod identifier, range 1–31, dynamically leased via Redis |
| Sequence | 12 bits | Per-millisecond sequence number, range 0–4095; waits for next millisecond on overflow |

**Custom Epoch:** `2025-07-22 00:00:00 UTC` (timestamp = `1753056000000`)

---

### 1.3 ID Generation Flow

```
nextId() is called
  │
  ├─ Get current timestamp (ms)
  │
  ├─ timestamp < lastTimestamp?
  │     └─ YES → throw exception (clock moved backwards, reject generation)
  │
  ├─ timestamp == lastTimestamp?
  │     ├─ YES → sequence + 1
  │     │         └─ sequence overflow (> 4095)?
  │     │               └─ YES → wait for next millisecond
  │     └─ NO  → reset sequence to 0
  │
  └─ Assemble ID:
       (timestamp - EPOCH) << 22
       | dataCenterId      << 17
       | podId             << 12
       | sequence
```

---

### 1.4 Pod ID Dynamic Leasing (Kubernetes)

In a Kubernetes environment, Pods are created and destroyed dynamically, making static Pod ID assignment impractical. A Redis-based dynamic leasing mechanism is used instead:

```
Pod startup (@PostConstruct)
  → Scan Redis, attempt to claim the first available snowflake:pod:{dcId}:{id}
  → SETNX key "locked" EX 30
  → Success → store podId in instance variable

Pod running (@Scheduled, every 10 seconds)
  → EXPIRE the Redis key, renew TTL to 30 seconds
  → Ensures the key does not expire while the Pod is alive

Pod graceful shutdown (@PreDestroy)
  → DEL the corresponding Redis key, immediately return the Pod ID to the pool

Pod crash (abnormal termination)
  → TTL expires (within 30 seconds), key is automatically removed, Pod ID returned to pool
```

Redis key format:

```
snowflake:pod:{dataCenterId}:{podId}
snowflake:pod:1:3   → DC 1, Pod ID 3
snowflake:pod:2:7   → DC 2, Pod ID 7
```

**Capacity limits:**

| Dimension | Limit |
|---|---|
| Maximum concurrent Pods per DC | 31 |
| Maximum concurrent Pods system-wide (2 DCs) | 62 |
| Maximum IDs generated per millisecond (single Pod) | 4,096 |

When the pool is exhausted, the Pod will fail to start and throw an exception. No ID will be generated, preventing duplicates.

---

### 1.5 DC ID Configuration

DC ID is statically configured and injected via Kubernetes environment variables. It does not participate in dynamic leasing:

```yaml
# K8s Deployment — Taipei DC
env:
  - name: SNOWFLAKE_DATACENTER_ID
    value: "1"

# K8s Deployment — Kaohsiung DC
env:
  - name: SNOWFLAKE_DATACENTER_ID
    value: "2"
```

Corresponding Spring configuration:

```yaml
snowflake:
  datacenter-id: ${SNOWFLAKE_DATACENTER_ID:1}
```

---

### 1.6 Module Location

```
common-biz
└── id
      ├── SnowflakePodRegistry        # Redis Pod ID lease / renewal / release
      ├── SnowflakeIdGenerator        # ID generation logic
      └── SnowflakeAutoConfiguration  # Auto-configuration, activated on common-biz import
```

Services that import `common-biz` require no additional configuration. The Pod ID leasing process runs automatically on startup.

---

## 2. Foreign Key Strategy

Database-level FK constraints are not enforced in MySQL. Referential integrity is validated at the application layer for the following reasons:

- In a microservice architecture, related data may reside in different databases; DB-level FKs cannot span database boundaries
- FKs introduce additional locking and performance overhead in high-write scenarios
- FKs prevent horizontal data partitioning in sharded environments

Cross-service data consistency is managed by the Saga pattern.

---

## 3. JPA and Native Query

| Scenario | Approach |
|---|---|
| Simple CRUD | JPA Repository |
| Heavy aggregation, batch computation, complex multi-table queries | Native SQL |

---

## 4. Flyway Database Version Management

### 4.1 Naming Conventions

- Database names: uppercase with underscores, e.g. `AUTH_DB`
- Table names: lowercase with underscores, e.g. `table_info`

### 4.2 Version Number Rules

Version format is `{major}.{minor}.{patch}`, starting from `1.0.0`:

| Case | Rule |
|---|---|
| General change | patch + 1, e.g. `1.0.0` → `1.0.1` |
| Cross-module or release boundary | minor + 1, e.g. `1.0.x` → `1.1.0` |

### 4.3 File Naming Rules

- DDL changes (create table, alter column): filename must begin with `create_` or `alter_`
- DML changes (seed data): filename must begin with `insert_`

```
V1.0.0__create_table_casha_user.sql
V1.0.1__create_table_role.sql
V1.0.6__alter_table_function_menu_add_column_permission_code.sql
V1.1.0__create_table_wallet.sql
```