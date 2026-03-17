# 資料管理

## 0. 資料庫環境

### 0.1 MySQL 版本

本系統使用 **MySQL 8.0**，主要原因如下：

- 原生支援 Window Function、CTE（`WITH` 語法），簡化複雜查詢
- 改善 InnoDB 效能與並發處理能力
- 支援降序索引，對趨勢遞增的 Snowflake ID 更友好
- JSON 欄位支援更完整

### 0.2 字符集與 Collation 配置

所有資料庫、資料表、欄位統一使用以下配置：

| 項目 | 設定值 |
|---|---|
| 字符集 | `utf8mb4` |
| 預設 Collation | `utf8mb4_unicode_ci` |
| 例外欄位 Collation | `utf8mb4_bin` |

`utf8mb4` 完整支援 Unicode（含 Emoji），避免 `utf8` 的 3-byte 限制問題。

`utf8mb4_unicode_ci` 符合完整 Unicode 比較標準，大小寫不敏感，適合一般業務欄位（帳號、Email 等）。

對需要嚴格區分大小寫的欄位（如 token、password hash、固定格式 code），在欄位層級單獨覆寫為 `utf8mb4_bin`。

MySQL Server 層級配置：

```ini
[mysqld]
character-set-server  = utf8mb4
collation-server      = utf8mb4_unicode_ci
```

Flyway migration 建表時明確宣告：

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

Spring Datasource 連線字串也須明確帶上：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/auth_db?useUnicode=true&characterEncoding=utf8mb4&connectionCollation=utf8mb4_unicode_ci
```

---

## 1. 主鍵策略 — Snowflake ID

### 1.1 設計動機

本系統採用 Snowflake ID 作為全域主鍵，原因如下：

- 全域唯一，避免分庫分表下的 ID 衝突
- 趨勢遞增，對 B-Tree 索引友好，避免頁分裂
- 純應用層生成，不依賴資料庫自增，消除跨服務的 DB 瓶頸
- 攜帶時間資訊，可從 ID 反推生成時間

部分資料量較少且不需要分片的表（如 RBAC 相關）不強制使用 Snowflake ID。

---

### 1.2 ID 結構

一個 Snowflake ID 是一個 64 bit 的長整數，結構如下：

| 1 bit | 41 bits | 5 bits | 5 bits | 12 bits |
|---|---|---|---|---|
| 符號位 | 時間戳(ms) | DC ID | Pod ID | 序列號 |

| 欄位 | 長度 | 說明 |
|---|---|---|
| 符號位 | 1 bit | 固定為 0，確保 ID 為正數 |
| 時間戳 | 41 bits | 距自定義 Epoch 的毫秒數，可使用約 69 年 |
| DC ID | 5 bits | 資料中心編號，範圍 1～31，對應台北 / 高雄等實體 DC |
| Pod ID | 5 bits | Pod 編號，範圍 1～31，由 Redis 動態租用 |
| 序列號 | 12 bits | 同一毫秒內的序號，範圍 0～4095，超過則等待下一毫秒 |

**自定義 Epoch：** `2025-07-22 00:00:00 UTC`（timestamp = `1753056000000`）

---

### 1.3 ID 生成流程

```
nextId() 被呼叫
  │
  ├─ 取得當前時間戳 (ms)
  │
  ├─ 時間戳 < lastTimestamp？
  │     └─ YES → 拋出例外（時鐘回撥，拒絕生成）
  │
  ├─ 時間戳 == lastTimestamp？
  │     ├─ YES → sequence + 1
  │     │         └─ sequence 溢出（> 4095）？
  │     │               └─ YES → 等待下一毫秒
  │     └─ NO  → sequence 歸零
  │
  └─ 組裝 ID：
       (timestamp - EPOCH) << 22
       | dataCenterId      << 17
       | podId             << 12
       | sequence
```

---

### 1.4 Pod ID 動態租用（K8s 環境）

在 K8s 環境下，Pod 是動態擴縮的，無法靜態指定 Pod ID。因此透過 Redis 實作動態租用機制：

```
Pod 啟動（@PostConstruct）
  → 掃描 Redis，搶占第一個可用的 snowflake:pod:{dcId}:{id}
  → SETNX key "locked" EX 30
  → 成功 → 寫入 podId 實例變數

Pod 運行中（@Scheduled，每 10 秒）
  → 對 Redis key 執行 EXPIRE，續期 30 秒
  → 確保 Pod 存活期間 key 不過期

Pod 正常關閉（@PreDestroy）
  → DEL 對應的 Redis key，立即歸還 Pod ID

Pod 異常崩潰
  → TTL 到期（最長 30 秒）後 key 自動消失，Pod ID 自動歸還
```

Redis key 格式：

```
snowflake:pod:{dataCenterId}:{podId}
snowflake:pod:1:3   → DC 1，Pod ID 3
snowflake:pod:2:7   → DC 2，Pod ID 7
```

**容量上限：**

| 維度 | 上限 |
|---|---|
| 單一 DC 最大同時 Pod 數 | 31 |
| 全系統（2 DC）最大同時 Pod 數 | 62 |
| 單一毫秒內最大 ID 生成量（單 Pod） | 4,096 |

Pool 耗盡時 Pod 將啟動失敗並拋出例外，不產生 ID，以防止重複。

---

### 1.5 DC ID 配置

DC ID 為靜態設定，透過 K8s 環境變數注入，不參與動態租用：

```yaml
# K8s Deployment — 台北 DC
env:
  - name: SNOWFLAKE_DATACENTER_ID
    value: "1"

# K8s Deployment — 高雄 DC
env:
  - name: SNOWFLAKE_DATACENTER_ID
    value: "2"
```

對應的 Spring 配置：

```yaml
snowflake:
  datacenter-id: ${SNOWFLAKE_DATACENTER_ID:1}
```

---

### 1.6 模組位置

```
common-biz
└── id
      ├── SnowflakePodRegistry        # Redis Pod ID 租用 / 續期 / 歸還
      ├── SnowflakeIdGenerator        # ID 生成邏輯
      └── SnowflakeAutoConfiguration  # Auto-configuration，引入 common-biz 自動生效
```

引入 `common-biz` 的服務無需額外配置，啟動時自動執行 Pod ID 租用流程。

---

## 2. Foreign Key 策略

在 MySQL 中不強制設定 FK，改由應用層驗證關聯完整性。原因如下：

- 微服務架構下，相關資料可能分散於不同資料庫，DB 層 FK 無法跨庫
- FK 在高寫入場景下會造成額外的鎖定與效能瓶頸
- 分庫分表時，FK 會阻礙資料的水平切分

跨服務的資料一致性由 Saga 模式處理。

---

## 3. JPA 與 Native Query

| 場景 | 做法 |
|---|---|
| 簡單 CRUD | JPA Repository |
| 大量聚合、批量計算、複雜多表查詢 | Native SQL |

---

## 4. Flyway 資料庫版本管理

### 4.1 命名規範

- 資料庫名稱：大寫 + 底線，如 `AUTH_DB`
- 資料表名稱：小寫 + 底線，如 `table_info`

### 4.2 版本號規則

版本號格式為 `{major}.{minor}.{patch}`，從 `1.0.0` 開始：

| 情況 | 規則 |
|---|---|
| 一般變更 | patch + 1，如 `1.0.0` → `1.0.1` |
| 跨功能模組或 release 版本 | minor + 1，如 `1.0.x` → `1.1.0` |

### 4.3 檔案命名規則

- DDL 變更（建表、修改欄位）：檔名以 `create_` 或 `alter_` 開頭
- DML 變更（初始資料）：檔名以 `insert_` 開頭

```
V1.0.0__create_table_casha_user.sql
V1.0.1__create_table_role.sql
V1.0.6__alter_table_function_menu_add_column_permission_code.sql
V1.1.0__create_table_wallet.sql
```

## 5. BaseTimeEntity
 
所有 JPA Entity 必須繼承 `BaseTimeEntity`，統一管理建立時間與更新時間。
 
```java
@MappedSuperclass
@Getter
@Setter
public abstract class BaseTimeEntity {
 
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
 
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
 
    @PrePersist
    protected void onCreate() {
        LocalDateTime now = LocalDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }
 
    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```
 
| 欄位 | 說明 |
|---|---|
| `created_at` | 資料建立時間，寫入後不可更新（`updatable = false`） |
| `updated_at` | 資料最後更新時間，每次 `@PreUpdate` 自動刷新 |
 
**規則**
 
- `BaseTimeEntity` 定義於 `common-biz`，所有 `*-service` 的 Entity 皆須繼承
- Entity 本身不得手動設定 `createdAt` 或 `updatedAt`，由 JPA lifecycle callback 統一管理
- `@MappedSuperclass` 確保欄位對應到子類別的資料表，不產生獨立的表
