# 點數入帳（內部）— Wallet Service Spec

> **服務：** wallet-service
> **最後更新：** 2026-03

---

## 1. 概述

僅供 Points Service 呼叫的內部 API，用於以 `PENDING` 狀態建立 `point_reward_batch` 記錄。
在 Points Service 從 `txn.card.authorized` Kafka 事件計算出 `reward_points` 後觸發。
批次將由 wallet-service 的 T+1 結算排程工作（每晚）轉換為 `CONFIRMED`。

> **不對 BFF 或外部呼叫方暴露。** `/internal/` 路徑前綴由 Nginx 層攔截。

---

## 2. API Definition — wallet-service

### Endpoint

| 欄位 | 值 |
|-------|-------|
| Method | POST |
| Path | `/wallet-service/internal/points/credit` |
| Context-path | `server.servlet.context-path: /wallet-service` |
| 呼叫方 | 僅限 points-service（內部服務網路） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Content-Type | ✅ | `application/json` |

### Request Body

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| userId | Long | ✅ | 擁有者會員 ID |
| sourceTxnId | String | ✅ | 來源交易 ID；冪等性鍵。重複請求（相同 `sourceTxnId`）將被拒絕並回傳 `DUPLICATE_SOURCE_TXN` |
| issuedAmount | Long | ✅ | 發放點數（TWD 分）；必須 > 0 |
| expiresAt | String | ✅ | ISO-8601 日期時間；由 Points Service 計算為交易時間 `txnTimestamp` 一年後同月最後一刻 |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| batchId | Long | 新建立的 `point_reward_batch` 的 Snowflake ID |

---

## 3. Service UseCase Flow

**UseCase：** `PointsCreditUseCase.credit()`
**UseCase Impl：** `PointsCreditUseCaseImpl`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[DB READ]` | 查詢 `point_reward_batch` WHERE `source_txn_id = :sourceTxnId`；若已存在 → 拒絕 | 重複 → 拋出 `DUPLICATE_SOURCE_TXN`（409） |
| 2 | `[DB READ]` | 查詢 `wallet_account` WHERE `user_id = :userId`；確認帳戶存在 | 不存在 → 拋出 `WALLET_NOT_FOUND`（404） |
| 3 | `[DOMAIN]` | 產生 Snowflake `batch_id`；設定 `remaining_balance = issuedAmount`；`status = PENDING` | — |
| 4 | `[DB WRITE]` | INSERT `point_reward_batch`（`batch_id`、`user_id`、`source_txn_id`、`issued_amount`、`remaining_balance`、`status = PENDING`、`expires_at`、timestamps） | DB 錯誤 → 拋出 `INTERNAL_ERROR` |
| 5 | `[RETURN]` | 回傳 `PointsCreditRs { batchId }` | — |

> **注意：** 此步驟不累加 `wallet_account.available_balance`。available_balance 僅在 T+1 結算排程將批次轉換為 `CONFIRMED` 時才增加。

---

## 4. Database

### 寫入資料表

| 資料表 | 操作 | 說明 |
|-------|------|------|
| `point_reward_batch` | INSERT | 以 `PENDING` 狀態新增批次 |

### 讀取資料表

| 資料表 | 操作 | 條件 |
|-------|------|------|
| `point_reward_batch` | SELECT | `source_txn_id = :sourceTxnId`（冪等性檢查） |
| `wallet_account` | SELECT | `user_id = :userId`（帳戶存在性確認） |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | WALLET_NOT_FOUND | 錢包帳戶不存在 | `userId` 無對應的 `wallet_account` 記錄 |
| 409 | DUPLICATE_SOURCE_TXN | 來源交易重複 | `sourceTxnId` 已存在於 `point_reward_batch` |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
