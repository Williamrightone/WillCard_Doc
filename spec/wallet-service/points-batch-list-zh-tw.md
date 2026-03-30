# 查詢點數批次列表 — Wallet Service Spec

> **服務：** wallet-service
> **最後更新：** 2026-03

---

## 1. 概述

以分頁方式回傳呼叫方的 `point_reward_batch` 記錄，依 `expires_at ASC`（最早到期優先）排序。
供 BFF 向使用者展示每個批次的點數明細，包含剩餘餘額、到期日及狀態。

> 此 API 在 Phase 2（僅定義 wallet 資料模型）時遞延，待 Phase 3 Points Service 開始填充 `point_reward_batch` 後實作。

---

## 2. API Definition — wallet-service

### Endpoint

| 欄位 | 值 |
|-------|-------|
| Method | GET |
| Path | `/wallet-service/points-batches` |
| Context-path | `server.servlet.context-path: /wallet-service` |
| 呼叫方 | BFF（透過 `WalletFeignClient`） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| X-Member-Id | ✅ | BFF 轉發的已認證會員 ID |

### Query Parameters

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| page | Integer | ❌ | 0 | 頁碼（0 起始） |
| size | Integer | ❌ | 20 | 每頁筆數；最大 50 |

### Response Body

```json
{
  "items": [
    {
      "batchId": 1234567890,
      "issuedAmount": 1750,
      "remainingBalance": 1200,
      "status": "CONFIRMED",
      "expiresAt": "2027-03-31T23:59:59"
    }
  ],
  "totalCount": 5,
  "page": 0,
  "size": 20
}
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| items | Array | 當頁的批次記錄 |
| items[].batchId | Long | Snowflake 批次 ID |
| items[].issuedAmount | Long | 批次原始發放點數（TWD 分） |
| items[].remainingBalance | Long | 批次目前可用點數（TWD 分） |
| items[].status | String | `PENDING / CONFIRMED / DEPLETED / EXPIRED` |
| items[].expiresAt | String | ISO-8601 到期日期時間 |
| totalCount | Long | 符合查詢條件的批次總筆數 |
| page | Integer | 目前頁碼（0 起始） |
| size | Integer | 使用的每頁筆數 |

> **前端顯示建議：** `status = DEPLETED` 或 `EXPIRED` 的批次仍包含在列表中供歷史查閱。客戶端應對即將到期（例如 30 天內）的批次加以提示。

---

## 3. Service UseCase Flow

**UseCase：** `PointsBatchListUseCase.list()`
**UseCase Impl：** `PointsBatchListUseCaseImpl`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[DB READ]` | 查詢 `point_reward_batch` WHERE `user_id = :userId` ORDER BY `expires_at ASC`，套用分頁（`LIMIT :size OFFSET :page*size`） | 系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 2 | `[DB READ]` | 查詢 `COUNT(*)` from `point_reward_batch` WHERE `user_id = :userId` 以取得 `totalCount` | 系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 3 | `[RETURN]` | 映射為 `PointsBatchListRs` 並回傳 | — |

---

## 4. Database

### 讀取資料表

| 資料表 | 操作 | 條件 |
|-------|------|------|
| `point_reward_batch` | SELECT | `user_id = :userId` ORDER BY `expires_at ASC` 含分頁 |
| `point_reward_batch` | COUNT | `user_id = :userId` |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
