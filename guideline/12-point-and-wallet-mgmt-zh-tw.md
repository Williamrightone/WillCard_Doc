# 點數與錢包管理

本文件說明 WillCard 如何管理用戶的現金回饋點數與錢包帳戶。
涵蓋點數發放、每批到期追蹤、FIFO 扣除順序、Reserve-Confirm-Release 機制，以及錢包層級的管理操作。
實作細節（API 合約、DB Schema、UseCase 流程）詳見 `spec/wallet-service/` 與 `spec/mobile-bff/` 下的對應 spec。

---

## 1. 錢包帳戶（Wallet Account）

每位用戶在 **wallet-service** 中有一個錢包帳戶，儲存兩個餘額欄位，單位皆為 **TWD 分（cents）**。

| 欄位 | 型別 | 角色 | 說明 |
|------|------|------|------|
| `available_balance` | BIGINT | **反正規化快取** | 所有未到期 `point_reward_batch.remaining_balance` 的加總；每次點數異動事件時原子更新 |
| `reserved_balance` | BIGINT | **顯示用** | 所有 PENDING reservation 的加總；讓用戶知道「進行中交易已鎖定多少點數」 |

> **餘額的唯一事實來源是 `point_reward_batch.remaining_balance`**，而非 `wallet_account.available_balance`。
> `available_balance` 的存在是為了 Phase 7 的快速驗證，須透過第 7 節列出的事件保持同步。

**單位：** 1 單位 = NT$0.01（1 TWD 分）。NT$100 = 10,000 單位。

---

## 2. 點數發放（如何獲得點數）

點數以**現金回饋（cashback）**形式在卡片交易授權成功並結算後發放。每次發放建立一筆獨立的 `point_reward_batch` 記錄，各有其到期日。

### 發放時機

| 階段 | 事件 | 行為 |
|------|------|------|
| 授權成功 | 發布 Kafka 事件 `txn.card.authorized` | Points Service 建立 `point_reward_batch`，初始狀態為 `PENDING` |
| T+1 清算 | 批次清算作業執行 | `point_reward_batch` 轉為 `CONFIRMED`；`wallet_account.available_balance` 累加 |

### 到期政策

每批點數的到期日為**發放月份後一年的同月最後一天 23:59:59**。

| 發放月份 | `expires_at` |
|---------|-------------|
| 2026-01 | 2027-01-31 23:59:59 |
| 2026-02 | 2027-02-28 23:59:59 |
| 2026-03 | 2027-03-31 23:59:59 |

> **Phase 3 範疇：** `point_reward_batch` 與回饋計算邏輯於 Phase 3 定義。Phase 2 定義接收入帳的錢包資料模型。

### 發放公式

```
reward_points = floor(txn_twd_base_amount × base_rate)
```

| 變數 | 來源 |
|------|------|
| `txn_twd_base_amount` | 交易金額（TWD 分），不含匯率手續費 |
| `base_rate` | `card_type_limit.domestic_reward_rate` 或 `overseas_reward_rate`（依交易幣別決定使用國內或海外費率） |

> 點數一律以**不含跨國手續費的 TWD 等值基礎金額**計算。

> **未來擴充：** 待 MCC 獎勵計畫與商家加成功能引入後，公式將擴充為 `floor(txn_twd_base_amount × (base_rate + mcc_bonus + merchant_bonus))`。請參閱 [guideline/13-reward-plan-zh-tw.md](/guideline/13-reward-plan-zh-tw.md) 第 8 節。

---

## 3. 點數折抵（如何使用點數）

前端每次刷卡請求時，帶入 `usePoints: Boolean` 旗標決定本次是否使用點數。**系統不儲存任何用戶偏好設定。**

### 折抵規則

```
若 usePoints == true：
    pointsToUse = min(available_balance, cardAmount)
否則：
    pointsToUse = 0
```

| 限制 | 說明 |
|------|------|
| `available_balance` | 不可折抵超過用戶實際持有的總點數 |
| `cardAmount` | 不可折抵超過本次交易金額 |

`usePoints = true` 時，點數一律**盡量多折抵**。無用戶自訂上限。若餘額不足以抵消全部交易金額，點數折抵能折的部分，剩餘金額透過帳本結算。

### 淨結算金額

```
net_settlement_amount = cardAmount - pointsToUse
```

---

## 4. 折抵比例

點數以 **1:1 比例**兌換 TWD 分：

```
1 點 = NT$0.01（1 TWD 分）
```

**快速對照：**

| 可用點數 | usePoints | 交易金額 | pointsToUse | 淨結算 |
|---------|-----------|---------|-------------|--------|
| 10,000 | true | NT$200（20,000 分） | 10,000 | NT$100 |
| 30,000 | true | NT$200（20,000 分） | 20,000 | NT$0 |
| 50,000 | false | NT$200（20,000 分） | 0 | NT$200 |
| 3,000 | true | NT$200（20,000 分） | 3,000 | NT$170 |

---

## 5. 每批到期與 FIFO 扣除順序

由於每筆 `point_reward_batch` 各有其 `expires_at`，點數必須依**最早到期優先（FIFO）**的順序消耗，確保較早發放的點數在到期前先被使用。

### FIFO 扣除演算法

```
Input:  userId, totalNeeded
Output: list of (batch_id, amount) pairs

query:
  SELECT batch_id, remaining_balance, expires_at
  FROM point_reward_batch
  WHERE user_id = userId
    AND status = 'CONFIRMED'
    AND remaining_balance > 0
    AND expires_at > NOW()
  ORDER BY expires_at ASC

逐批消耗直到 totalNeeded 集齊：
  take = min(batch.remaining_balance, remaining_needed)
  append (batch.batch_id, take) to result
  remaining_needed -= take

若 remaining_needed > 0：餘額不足 → 中止
```

每個 `(batch_id, take)` 對應一筆 `wallet_reservation_item`。

### 批次狀態生命週期

```
PENDING ──► CONFIRMED ──► DEPLETED   （remaining_balance = 0）
                      └──► EXPIRED   （expires_at < NOW()，排程作業處理）
```

| 狀態 | 條件 |
|------|------|
| `PENDING` | 已發放但 T+1 清算尚未完成 |
| `CONFIRMED` | 可供折抵 |
| `DEPLETED` | 已完全消耗（`remaining_balance = 0`） |
| `EXPIRED` | 到期前未完全使用；殘餘 `remaining_balance` 作廢 |

---

## 6. Reserve-Confirm-Release 機制

OTP 視窗在授權與確認之間存在時間差，點數在 Pending 狀態期間必須**鎖定**，以防重複消費。Reserve-Confirm-Release 在批次層級實作此鎖定。

### 資料模型

**`wallet_reservation`** — 每次授權嘗試一筆記錄：

| 欄位 | 說明 |
|------|------|
| `reservation_id` | Snowflake PK |
| `user_id` | 持有者 |
| `total_amount` | 跨所有批次的總鎖定點數 |
| `status` | `PENDING` → `CONFIRMED` 或 `RELEASED` |
| `created_at` / `updated_at` | 稽核欄位 |

**`wallet_reservation_item`** — 每批次消耗各一筆記錄：

| 欄位 | 說明 |
|------|------|
| `item_id` | Snowflake PK |
| `reservation_id` | FK to `wallet_reservation` |
| `batch_id` | FK to `point_reward_batch`（從哪一批扣） |
| `amount` | 從該批次扣除的點數 |

### 三個方法

> **Phase 2 範疇：** 以下三個方法在 Phase 2 僅定義為 wallet-service 的 service 層內部方法。HTTP endpoint spec 延至 **Phase 7**，屆時 card-service 在發卡機構授權流程中才需要呼叫這些端點。

| 方法 | 簽名 | 執行步驟 |
|------|------|---------|
| `reserve` | `reserve(userId, amount) → reservationId` | 1. 執行 FIFO 演算法 → (batch_id, amount) 清單<br>2. 各批次 `remaining_balance` 減少<br>3. 建立 `wallet_reservation`（PENDING）及 `wallet_reservation_item`<br>4. `available_balance -= amount`，`reserved_balance += amount` |
| `confirmDeduct` | `confirmDeduct(reservationId)` | 1. `wallet_reservation` → CONFIRMED<br>2. `reserved_balance -= amount`（點數永久消耗；批次已在 reserve 時扣除） |
| `release` | `release(reservationId)` | 1. 查詢所有 `wallet_reservation_item`<br>2. 各 item 的 `amount` 還回對應批次<br>3. `wallet_reservation` → RELEASED<br>4. `available_balance += total_amount`，`reserved_balance -= total_amount` |

> **為何在 reserve 時就扣除 `remaining_balance`？**
> 確保並發請求無法看到相同點數為可用，防止雙重消費。

### 各步驟後的餘額狀態

```
初始狀態：batch A: remaining=8,000   batch B: remaining=5,000
          available_balance = 13,000  reserved_balance = 0

reserve(userId, 10,000)：
  FIFO：從 A 取 8,000，從 B 取 2,000
  batch A: remaining = 0（→ DEPLETED）
  batch B: remaining = 3,000
  reservation_item: [(A, 8000), (B, 2000)]
  available_balance = 3,000   reserved_balance = 10,000

confirmDeduct(reservationId)：
  reservation → CONFIRMED；reserved_balance = 0

release(reservationId)：
  batch A: remaining = 8,000（還原）
  batch B: remaining = 5,000（還原）
  available_balance = 13,000  reserved_balance = 0
```

---

## 7. `available_balance` 快取維護

`wallet_account.available_balance` 必須與每個改變真實可用點數的事件**原子同步**。

| 事件 | `available_balance` | `reserved_balance` |
|------|--------------------|--------------------|
| 批次 PENDING → CONFIRMED（T+1 入帳） | `+= issued_amount` | — |
| `reserve()` 呼叫 | `-= amount` | `+= amount` |
| `confirmDeduct()` 呼叫 | — | `-= amount` |
| `release()` 呼叫 | `+= amount` | `-= amount` |
| 到期作業：批次 CONFIRMED → EXPIRED | `-= remaining_balance`（歸零前） | — |

所有更新皆在與對應 `point_reward_batch` 或 `wallet_reservation` 寫入**同一個 DB transaction** 中完成。

---

## 8. 交易時點數的生命週期

```
Client 刷卡請求，帶入 usePoints: Boolean
        │
        ├──── usePoints = false ──────────────────────────────────►
        │                  pointsToUse = 0；跳過 reserve
        │                  交易全額透過帳本結算
        │
        └──── usePoints = true ───────────────────────────────────►
                           pointsToUse = min(available_balance, cardAmount)
                           │
                           ▼
        [Phase 7 — authorize（第一段授權）]
          Wallet.reserve(userId, pointsToUse) → reservationId
          FIFO：從最早到期批次開始扣除 remaining_balance
          建立 wallet_reservation + wallet_reservation_item
          available_balance -= pointsToUse
          產生 OTP，Redis session 儲存（TTL 180s）
          Payload：{ pointsToUse, reservationId, ... }
                           │
                           ├──── OTP 正確 ──────────────────────►
                           │     [Phase 7 — verify-challenge]
                           │     Wallet.confirmDeduct(reservationId)
                           │     reservation → CONFIRMED
                           │     reserved_balance -= pointsToUse
                           │     發布 Kafka：txn.card.authorized
                           │           │
                           │           ▼
                           │     [Phase 9 — Saga]
                           │     Points Service 建立新批次（PENDING）
                           │     T+1：→ CONFIRMED；available_balance += reward_points
                           │
                           ├──── OTP 錯誤（< 3 次）─────────────►
                           │     Session 繼續有效；reservation 不變
                           │
                           ├──── OTP 錯誤（第 3 次 / 卡片鎖定）►
                           │     Wallet.release(reservationId)
                           │     各批次還原；available_balance += pointsToUse
                           │
                           └──── Session 過期（TTL）─────────────►
                                 [Phase 9 — 惰性清理]
                                 Wallet.release(reservationId)
```

---

## 9. 錢包管理

### API 一覽

| 操作 | Method | 路徑 | 說明 |
|------|--------|------|------|
| 餘額查詢 | GET | `/api/v1/wallet/balance` | 回傳 `availableBalance` 與 `reservedBalance` |
| 點數批次清單 | GET | `/api/v1/wallet/points-batches` | 列出 `point_reward_batch` 記錄（Phase 3 加入） |

> 系統不儲存任何用戶偏好。`usePoints` 旗標由前端每筆交易帶入，不需設定頁面。

### 到期清理作業

排程作業每日執行，處理逾期批次：

```sql
SELECT * FROM point_reward_batch
  WHERE status = 'CONFIRMED'
    AND remaining_balance > 0
    AND expires_at < NOW()

每筆過期批次：
  BEGIN TRANSACTION
    UPDATE point_reward_batch SET status = 'EXPIRED', remaining_balance = 0
    UPDATE wallet_account SET available_balance -= 原 remaining_balance
      WHERE user_id = batch.user_id
  COMMIT
```

> **冪等性：** WHERE 條件（`status = 'CONFIRMED'`）確保已到期批次不重複處理。

### 職責邊界

| 關注點 | 所屬 |
|--------|------|
| 聚合餘額快取（`available_balance`、`reserved_balance`） | `wallet_account` |
| 每批餘額與到期（`remaining_balance`、`expires_at`） | `point_reward_batch` |
| 進行中鎖定（哪些批次、各鎖多少） | `wallet_reservation` + `wallet_reservation_item` |
| 回饋率計算 | Points Service / Phase 3 |

---

## 10. 範例

### 範例 1 — 完全折抵（點數覆蓋全部交易金額）

`available_balance = 80,000`（NT$800），`usePoints = true`
交易：NT$300（30,000 分），CLASSIC 卡（1% 回饋）

```
pointsToUse = min(80,000, 30,000) = 30,000
淨結算 = 0
reward_points = floor(30,000 × 0.0100) = 300
```

`available_balance`：80,000 → 50,000（confirm 後）→ 50,300（T+1 後）

---

### 範例 2 — 部分折抵（點數不足）

`available_balance = 5,000`（NT$50），`usePoints = true`
交易：NT$200（20,000 分），OVERSEAS 卡（5% 回饋）

```
pointsToUse = min(5,000, 20,000) = 5,000
淨結算 = 15,000 分（NT$150 透過帳本結算）
reward_points = floor(20,000 × 0.0500) = 1,000
```

`available_balance`：5,000 → 0（confirm 後）→ 1,000（T+1 後）

---

### 範例 3 — 不使用點數

`available_balance = 50,000`，`usePoints = false`
交易：NT$500（50,000 分），PREMIUM 卡（2% 回饋）

```
pointsToUse = 0
淨結算 = NT$500 透過帳本結算
reward_points = floor(50,000 × 0.0200) = 1,000
```

`available_balance`：50,000 → 51,000（T+1 後）

---

### 範例 4 — 跨兩批次 FIFO 扣除

`usePoints = true`，交易：NT$80（8,000 分），OVERSEAS 卡（5%）

| batch_id | remaining_balance | expires_at |
|----------|------------------|------------|
| B1 | 3,000 | 2026-12-31（較早） |
| B2 | 10,000 | 2027-03-31 |

```
pointsToUse = min(13,000, 8,000) = 8,000
FIFO：從 B1 取 3,000（→ DEPLETED），從 B2 取 5,000
淨結算 = 0；reward_points = 400
```

---

### 範例 5 — OTP 失敗，點數精準還原

`available_balance = 13,000`，`usePoints = true`，`pointsToUse = 8,000`
Reserve：B1 全部（3,000），B2 部分（5,000）

OTP 連錯 3 次 → `release()`：
- B1、B2 各自還原
- `available_balance` 還原為 13,000，無淨變動

---

### 範例 6 — 點數到期

- B1：`remaining_balance = 1,500`，今晚到期
- B2：`remaining_balance = 8,000`，2027-03-31

到期作業執行：B1 → EXPIRED，`available_balance`：9,500 → 8,000

---

## 11. 已知缺口（Phase 2 範疇）

| 缺口 | 說明 |
|------|------|
| 到期寬限期 | `expires_at` 到期後無寬限期，點數立即作廢 |
| 到期通知 | 點數即將到期時無 Kafka 事件或推播通知 |
| 點數轉讓 | 無用戶之間轉移點數的 API |
| 點數批次清單 | 延至 Phase 3（需 `point_reward_batch` 有資料後才有意義） |
| 手動調整 | Phase 2 無後台 API 手動增減點數 |
| 批次並發競爭 | FIFO 查詢與 UPDATE 之間若批次被並發耗盡，reserve 需重試；Phase 2 不詳述重試邏輯 |

---

下一章：[點數發放（Reward Plan）](/guideline/13-reward-plan-zh-tw.md)
