# 點數發放（Phase 3）

本文件描述 WillCard 如何在卡片交易後計算並發放回饋點數。
涵蓋回饋計算公式、Points Service Kafka 消費流程、點數批次入帳、T+1 結算，以及點數批次查詢。
實作細節（API 合約、DB 結構、UseCase 流程）請參閱 `spec/points-service/`、`spec/wallet-service/` 與 `spec/mobile-bff/` 下的對應 spec 文件。

> **Phase 3 範疇：** 僅基於 `base_rate` 計算點數回饋。MCC 加成（`mcc_bonus`）與商家加成（`merchant_bonus`）遞延至後續 Phase。請參閱第 8 節的未來升級方向。

---

## 1. 概述

Phase 3 引入 **Points Service**，負責：

- 消費 `txn.card.authorized` Kafka 事件
- 使用卡別的 `base_rate` 計算 `reward_points`
- 呼叫 wallet-service 的內部入帳 API 建立 `point_reward_batch` 記錄

wallet-service 的 T+1 結算排程工作（Phase 2 定義）負責將 PENDING 批次確認為 CONFIRMED，並累加 `wallet_account.available_balance`。

---

## 2. 回饋費率

點數以交易的 **匯率費前 TWD 基準金額** 計算：

```
reward_points = floor(txn_twd_base_amount × base_rate)
```

| 變數 | 來源 | 說明 |
|------|------|------|
| `txn_twd_base_amount` | `txn.card.authorized` Kafka 事件 | 交易金額（TWD 分），不含匯率手續費 |
| `base_rate` | `card_type_limit.domestic_reward_rate` 或 `overseas_reward_rate` | 各卡別固定費率；依交易幣別決定使用國內或海外費率 |

**各卡別基本回饋費率（Phase 1 seed data）：**

| 卡別 | 國內 | 海外 |
|------|------|------|
| CLASSIC | 1% | 0% |
| OVERSEAS | 0% | 5% |
| PREMIUM | 2% | 5% |
| INFINITE | 2% | 5% |

> **匯率說明：** 點數一律以**匯率費前的 TWD 基準金額**計算。匯率手續費不計入獎勵。

> **Floor 說明：** `reward_points` 一律**無條件捨去**（`floor`）。小數點以下的點數直接捨棄。

> **零回饋：** 若 `base_rate = 0`（例如 CLASSIC 卡海外交易），`reward_points = 0`。不建立 `point_reward_batch`；Points Service 在 `point_issuance_log` 記錄 SKIP。

---

## 3. 點數發放流程

Points Service 訂閱 `txn.card.authorized` Kafka Topic，依以下流程發放點數：

```
[Kafka] 收到 txn.card.authorized 事件
    │
    ├── Step 1：查詢 point_issuance_log，確認 source_txn_id 是否已處理
    │          若已處理 → 跳過（冪等性保護）
    │
    ├── Step 2：解析事件 payload：
    │          userId、cardType、isOverseas、txnTwdBaseAmount、txnTimestamp、sourceTxnId
    │
    ├── Step 3：查詢 base_rate（來自 card_type_limit）
    │          isOverseas = true  → overseas_reward_rate
    │          isOverseas = false → domestic_reward_rate
    │
    ├── Step 4：計算：
    │          reward_points = floor(txnTwdBaseAmount × base_rate)
    │
    ├── Step 5：若 reward_points == 0
    │          → 插入 point_issuance_log（status = SKIP）；結束
    │
    └── Step 6：呼叫 wallet-service POST /wallet-service/internal/points/credit
               Request: { userId, sourceTxnId, issuedAmount: reward_points, expiresAt }
               Response: { batchId }
               成功 → 插入 point_issuance_log（status = SUCCESS, batchId）
               失敗 → 插入 point_issuance_log（status = FAILED）；發布至 DLQ
```

### 到期日計算

```
expiresAt = 交易時間戳所在月份，一年後同月的最後一刻
```

| `txnTimestamp` | `expiresAt` |
|---------------|-------------|
| 2026-03-15 | 2027-03-31 23:59:59.999 |
| 2026-12-01 | 2027-12-31 23:59:59.999 |
| 2027-02-10 | 2028-02-29 23:59:59.999（閏年） |

> Points Service 負責計算 `expiresAt` 並傳入入帳 API。wallet-service 原封不動地儲存。

---

## 4. T+1 結算

Phase 2（wallet-service）已定義，此處補充說明。

wallet-service 的排程工作每晚執行，確認待處理的點數批次：

```sql
SELECT * FROM point_reward_batch
 WHERE status = 'PENDING'
   AND DATE(created_at) < CURDATE()

對每個批次：
  BEGIN TRANSACTION
    UPDATE point_reward_batch SET status = 'CONFIRMED', updated_at = NOW()
    UPDATE wallet_account
       SET available_balance = available_balance + issued_amount
     WHERE user_id = batch.user_id
  COMMIT
```

> **冪等性保護：** `status = 'PENDING'` 的條件確保已確認的批次不會被重複處理。

確認後，點數即可在下次授權時被 FIFO 演算法（Phase 2 第 5 節）扣抵。

---

## 5. API 清單

### 5.1 點數批次列表（使用者端）

| 欄位 | 值 |
|-------|-------|
| 操作 | GET `/api/v1/wallet/points-batches` |
| BFF → Service | wallet-service |
| 用途 | 列出使用者的 `point_reward_batch` 記錄，依 `expiresAt ASC` 排序 |
| 需要認證 | 是（Bearer JWT） |
| 操作日誌等級 | L3 |

Query 參數：`page`（0 起始，預設 0）、`size`（預設 20，最大 50）。

Spec：[wallet-service/points-batch-list-zh-tw.md](../spec/wallet-service/points-batch-list-zh-tw.md)

### 5.2 點數入帳（內部）

| 欄位 | 值 |
|-------|-------|
| 操作 | POST `/wallet-service/internal/points/credit` |
| 呼叫方 | 僅限 points-service |
| 用途 | 在 wallet_db 中以 `PENDING` 狀態建立 `point_reward_batch` |
| 需要認證 | 否（內部服務；Nginx 封鎖 `/internal/` 對外流量） |

Spec：[wallet-service/points-credit-zh-tw.md](../spec/wallet-service/points-credit-zh-tw.md)

---

## 6. 範例

### 範例 1 — PREMIUM 卡、國內交易

- 卡別：PREMIUM（`domestic_reward_rate = 2%`）
- 交易：NT$500（50,000 分），交易時間 2026-03-15

```
reward_points = floor(50,000 × 0.0200) = 1,000
expiresAt     = 2027-03-31 23:59:59
```

建立 `point_reward_batch`：`issued_amount = 1,000`，`status = PENDING`
T+1 結算後：`status = CONFIRMED`，`available_balance += 1,000`

---

### 範例 2 — OVERSEAS 卡、海外交易

- 卡別：OVERSEAS（`overseas_reward_rate = 5%`）
- 海外交易：NT$1,000（100,000 分）

```
reward_points = floor(100,000 × 0.0500) = 5,000
```

---

### 範例 3 — 零回饋，不建立批次

- 卡別：CLASSIC（`overseas_reward_rate = 0%`）
- 海外交易：NT$300（30,000 分）

```
reward_points = floor(30,000 × 0) = 0
```

Points Service 插入 `point_issuance_log（status = SKIP）`，不建立 `point_reward_batch`。

---

## 7. 已知待辦（Phase 3 範疇）

| 待辦項目 | 說明 |
|-----|-------|
| 失敗重試機制 | `point_issuance_log（status = FAILED）` 僅記錄；DLQ 自動重試機制未在本階段定義 |
| 費率快取 | `base_rate` 透過 card-service Feign 查詢；本地快取策略遞延 |
| 點數到期通知 | 點數到期前無推播或 Email 提醒 |

---

## 8. 未來升級方向

目前 Phase 3 的公式僅使用 `base_rate`。後續 Phase 將擴充回饋計算，支援：

```
reward_points = floor(txn_twd_base_amount × (base_rate + mcc_bonus + merchant_bonus))
```

| 新增項目 | 說明 |
|----------|------|
| `mcc_bonus` | 以 MCC 為基礎的加成費率，定義於 `reward_plan` 與 `reward_plan_mcc` 資料表（points-service）。多個有效計畫的費率疊加累加。 |
| `merchant_bonus` | 商家專屬加成，來自 `card_type_merchant_benefit`（card-service，Phase 1 seed data）。 |
| 獎勵計畫查詢 API | `GET /api/v1/reward/plans` — 讓使用者在交易前查看目前有效的 MCC 加碼活動。 |
| 獎勵計畫管理 | 管理員 API，用於建立、編輯及停用含有效期間與卡別範圍的獎勵計畫。 |

---
