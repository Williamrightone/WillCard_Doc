# 查詢卡片資訊 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [card-service/card-info-zh-tw.md](../card-service/card-info-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 以 `cardId` 查詢卡片詳細資訊的請求。
轉發至 card-service，由其解密有效期限與持卡人姓名後組裝回傳。
BFF 不執行任何解密，僅負責流程轉發與回應中繼。
`pan_encrypted` 絕不回傳給 Client。

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
| Client URL | `/api/v1/card/info` |
| BFF 接收路徑 | `/card/info` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Request 參數

| 參數 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 目標卡片的 Snowflake ID |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardId | String | Snowflake ID（String） |
| maskedPan | String | `****-****-****-1234` |
| cardType | String | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| cardNetwork | String | VISA / MASTERCARD / JCB |
| expiryDate | String | 有效期限，`MM/YY` 格式（解密後） |
| cardholderNameZh | String | 中文持卡人姓名（解密後） |
| cardholderNameEn | String | 英文持卡人姓名（解密後） |
| status | String | ACTIVE / FROZEN / CANCELLED |
| activatedAt | String | ISO-8601 時間戳記；未啟用時為 null |

---

## 4. BFF UseCase Flow

**UseCase：** `CardInfoUseCase.getInfo(CardInfoRq)`
**UseCase Impl：** `CardInfoUseCaseImpl`
**Adapter：** `CardFeignAdapter` implements `CardClientPort`
**Feign：** `CardFeignClient`（使用 `card-contract` 的 `CardInfoRq` / `CardInfoRs`）

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[VALIDATE]` | 驗證 `cardId` 不為空 | 驗證失敗 → 拋出 `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` 呼叫 card-service → [card-service/card-info-zh-tw.md](../card-service/card-info-zh-tw.md) | `CARD_NOT_FOUND` → 拋出 `CARD_NOT_FOUND`；系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 3 | `[RETURN]` | 將 card-contract 的 `CardInfoRs` 中繼回傳給 Client | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | `cardId` 為空 |
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 404 | CARD_NOT_FOUND | 卡片不存在或不屬於此用戶 | card-service 回傳 CA00001 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
