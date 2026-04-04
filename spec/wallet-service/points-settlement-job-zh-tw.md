# 積分結算 Job — Wallet Service Spec

> **Service:** wallet-service
> **元件類型：** 排程批次 Job
> **最後更新：** 2026-04

---

## 1. Overview

每日排程執行的批次 Job，將前一個工作日（T 日）建立的所有 `PENDING` 積分批次結算為 `CONFIRMED`，並同步累加 `wallet_account.available_balance`。

交易當日（T 日）積分維持 `PENDING` 狀態，待 T+1 日本 Job 執行完畢後才可使用。每筆 batch 獨立開啟 DB transaction，確保單筆失敗不影響其他批次。

---

## 2. Job 定義

| 欄位 | 值 |
|------|-----|
| 類別 | `wallet.service.job.PointsSettlementJob` |
| 觸發方式 | `@Scheduled(cron = "0 0 2 * * *", zone = "Asia/Taipei")` |
| 執行時間 | 每日 02:00 CST |
| 處理範圍 | 所有 `point_reward_batch` 中 `status = 'PENDING'` 且 `DATE(created_at) < CURRENT_DATE`（Asia/Taipei）的資料列 |
| 並發防護 | Redis 分散式鎖 `job:lock:points-settlement` |

> **範圍說明：** 查詢條件使用 `created_at < CURRENT_DATE`（而非嚴格限定昨日），以便自動補跑先前失敗或跳過的 run 所遺留的 PENDING 批次。

---

## 3. Job UseCase Flow

**Job：** `PointsSettlementJob.run()`

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[REDIS WRITE]` | SETNX `job:lock:points-settlement`，TTL 10 分鐘 | 鎖已被持有 → log WARN，中止本次執行 |
| 2 | `[DB READ]` | SELECT 所有 `point_reward_batch` WHERE `status = 'PENDING'` AND `DATE(created_at, 'Asia/Taipei') < CURRENT_DATE`；取得待結算清單 | DB 錯誤 → 釋放鎖，log ERROR，中止 |
| 3 | `[DOMAIN]` | 對每筆 batch 開啟獨立 DB transaction（隔離等級：READ COMMITTED） | — |
| 4 | `[DB WRITE]` | UPDATE `point_reward_batch` SET `status = 'CONFIRMED'`, `updated_at = NOW()` WHERE `batch_id = :batchId` AND `status = 'PENDING'` | 更新 0 筆 → 已被其他程序處理；計為 SKIPPED，跳過 |
| 5 | `[DB WRITE]` | UPDATE `wallet_account` SET `available_balance = available_balance + :issuedAmount`, `updated_at = NOW()` WHERE `user_id = :userId` | DB 錯誤 → rollback 本批次 transaction，log ERROR 附 `batch_id`，計為 FAILED；繼續處理下一筆 |
| 6 | `[DOMAIN]` | Commit transaction；計為 SETTLED | — |
| 7 | `[REDIS WRITE]` | 釋放 `job:lock:points-settlement` | — |
| 8 | `[RETURN]` | Log 結算摘要：`{ totalFound, settled, skipped, failed }` | 若 `failed > 0` → 輸出 ERROR 等級告警，待人工處理 |

> **Step 4–6 屬同一筆 DB transaction。** Step 5 失敗時會 rollback Step 4 的狀態更新，該 batch 維持 `PENDING`，可在下次執行時重試。
>
> **冪等性：** Step 4 的 `WHERE status = 'PENDING'` 防止手動補跑時重複結算。更新 0 筆 = 已處理過；Job 繼續往下執行，不視為錯誤。
>
> **`wallet_account.available_balance` 語義：** 此欄位為 denormalized 快取（詳見 `wallet_db.md`）。權威餘額為所有 `CONFIRMED` 且未過期批次的 `remaining_balance` 加總。Step 5 的 UPDATE 與 Step 4 在同一 transaction 內原子更新，確保快取一致。

---

## 4. Database

### 讀取資料表

| 資料表 | 操作 | 條件 |
|--------|------|------|
| `point_reward_batch` | SELECT | `status = 'PENDING'` AND `DATE(created_at) < CURRENT_DATE` |

### 寫入資料表

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `point_reward_batch` | UPDATE | `status: PENDING → CONFIRMED` |
| `wallet_account` | UPDATE | `available_balance += issued_amount` |

### Redis Key

| Key | 操作 | TTL | 說明 |
|-----|------|-----|------|
| `job:lock:points-settlement` | SETNX / DEL | 10 分鐘 | 分散式防重複執行鎖 |

---

## 5. Error Handling

| 情境 | 行為 |
|------|------|
| 啟動時鎖已被持有 | 跳過本次執行；log WARN；待下次排程時重試 |
| Batch 已為 `CONFIRMED`（Step 4 更新 0 筆） | 計為 SKIPPED；繼續處理下一筆 |
| UPDATE DB 錯誤（Step 5） | Rollback 本批次；計為 FAILED；log ERROR 附 `batch_id`；繼續 |
| `wallet_account` 查無 `user_id` 對應記錄 | Rollback 本批次；計為 FAILED；log ERROR — 資料一致性異常，需人工介入 |
| 全部處理後 `failed > 0` | 輸出 ERROR 等級摘要；透過 EFK 觸發告警；不自動重試 |

---

## 6. Changelog

### v1.0 — 2026-04 — 初版
