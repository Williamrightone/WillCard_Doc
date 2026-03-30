# 查詢點數批次列表 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [wallet-service/points-batch-list-zh-tw.md](../wallet-service/points-batch-list-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 查詢使用者點數批次列表的請求。
轉發至 wallet-service，回傳依最早到期優先排序的 `point_reward_batch` 分頁列表。

> **L3 操作：** 唯讀查詢，不發布 Kafka 事件，僅 EFK 記錄。

---

## 2. 操作日誌等級

**等級：L3**
**行為：** 僅 EFK 記錄 — 不發布 Kafka 事件，不寫入 Audit DB。

---

## 3. API Definition — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| Client URL | `/api/v1/wallet/points-batches` |
| BFF 接收路徑 | `/wallet/points-batches` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Query Parameters

| 參數 | 型別 | 必填 | 預設值 | 說明 |
|------|------|------|--------|------|
| page | Integer | ❌ | 0 | 頁碼（0 起始） |
| size | Integer | ❌ | 20 | 每頁筆數；最大 50 |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| items | Array | 當頁的批次記錄 |
| items[].batchId | Long | Snowflake 批次 ID |
| items[].issuedAmount | Long | 批次原始發放點數（TWD 分） |
| items[].remainingBalance | Long | 批次目前可用點數（TWD 分） |
| items[].status | String | `PENDING / CONFIRMED / DEPLETED / EXPIRED` |
| items[].expiresAt | String | ISO-8601 到期日期時間 |
| totalCount | Long | 批次總筆數 |
| page | Integer | 目前頁碼（0 起始） |
| size | Integer | 使用的每頁筆數 |

---

## 4. BFF UseCase Flow

**UseCase：** `PointsBatchListUseCase.list()`
**UseCase Impl：** `PointsBatchListUseCaseImpl`
**Adapter：** `WalletFeignAdapter` implements `WalletClientPort`
**Feign：** `WalletFeignClient`（使用 `wallet-contract` 的 `PointsBatchListRs`）

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[FEIGN]` | `WalletFeignAdapter` 呼叫 wallet-service → [wallet-service/points-batch-list-zh-tw.md](../wallet-service/points-batch-list-zh-tw.md) | 系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 2 | `[RETURN]` | 將 `PointsBatchListRs` 中繼回傳給 Client | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
