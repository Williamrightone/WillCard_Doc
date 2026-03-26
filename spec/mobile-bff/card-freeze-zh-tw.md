# 凍結卡片 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [card-service/card-freeze-zh-tw.md](../card-service/card-freeze-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 的凍結卡片請求。
轉發至 card-service，由其驗證卡片狀態後將 `ACTIVE` 轉換為 `FROZEN`。
凍結後卡片不允許任何新的授權交易。

---

## 2. 操作日誌等級

**等級：L2**
**觸發時機：** 每次呼叫，無論成功或失敗。
**Kafka Topic：** `operation-log.card.freeze`

| OperationLogEvent 欄位 | 值來源 |
|----------------------|-------|
| service | `bff` |
| action | `card.freeze` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗時的錯誤說明；成功時為 null |
| errorCode | 失敗時的錯誤碼；成功時為 null |
| beforeSnapshot | `{ "cardId": "...", "status": "ACTIVE" }`（僅 SUCCESS） |
| afterSnapshot | `{ "cardId": "...", "status": "FROZEN" }`（僅 SUCCESS） |
| requestId | HTTP 關聯 ID |
| txnId | — |

**發布時機：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| 驗證失敗（Step 1） | FAIL | `INVALID_REQUEST` / — |
| card-service 錯誤（Step 2） | FAIL | 來自 card-service 的錯誤碼 |
| 凍結成功（Step 3） | SUCCESS | — |

---

## 3. API Definition — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| Client URL | `/api/v1/card/freeze` |
| BFF 接收路徑 | `/card/freeze` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 目標卡片的 Snowflake ID |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardId | String | 卡片 Snowflake ID（String） |
| status | String | `FROZEN` |

---

## 4. BFF UseCase Flow

**UseCase：** `CardFreezeUseCase.freeze(CardFreezeRq)`
**UseCase Impl：** `CardFreezeUseCaseImpl`
**Adapter：** `CardFeignAdapter` implements `CardClientPort`
**Feign：** `CardFeignClient`（使用 `card-contract` 的 `CardFreezeRq` / `CardStatusRs`）

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[VALIDATE]` | 驗證 `cardId` 不為空 | 驗證失敗 → 發布 FAIL 事件（async）→ 拋出 `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` 呼叫 card-service → [card-service/card-freeze-zh-tw.md](../card-service/card-freeze-zh-tw.md) | 業務錯誤 → 進入 Step 3；系統錯誤 → 發布 FAIL 事件（async）→ 拋出 `INTERNAL_ERROR` |
| 3 | `[KAFKA]` | **【失敗時】** 發布 FAIL 事件（來自 card-service 的錯誤碼，async）→ 重拋映射後的 BFF 錯誤碼 | 記錄錯誤；不壓制原始例外 |
| 4 | `[KAFKA]` | **【成功時】** 發布 SUCCESS 事件，含 `beforeSnapshot` 與 `afterSnapshot`（async） | 記錄錯誤；不拋出例外 |
| 5 | `[RETURN]` | 回傳 `CardStatusRs`（cardId、status = `FROZEN`） | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | `cardId` 為空 |
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 404 | CARD_NOT_FOUND | 卡片不存在或不屬於此用戶 | card-service 回傳 CA00001 |
| 409 | CARD_NOT_ACTIVE | 卡片非 ACTIVE 狀態 | card-service 回傳 CA00002 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
