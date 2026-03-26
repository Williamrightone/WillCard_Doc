# 查詢特約優惠 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [card-service/card-benefits-zh-tw.md](../card-service/card-benefits-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 查詢指定卡種目前有效特約優惠的請求。
轉發至 card-service，由其查詢 `card_type_merchant_benefit` 中符合條件的記錄並回傳清單。

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
| Client URL | `/api/v1/card/benefits` |
| BFF 接收路徑 | `/card/benefits` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Request 參數

| 參數 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardType | String | ✅ | @NotBlank；enum：CLASSIC / OVERSEAS / PREMIUM / INFINITE | 查詢優惠的卡種 |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardType | String | 查詢的卡種 |
| benefits | List\<BenefitItem\> | 有效優惠清單（可為空） |

**BenefitItem：**

| 欄位 | 型別 | 說明 |
|------|------|------|
| benefitId | String | Snowflake ID（String） |
| merchantId | String | 特定商家 ID；null = 適用整個 MCC |
| mccCode | String | MCC 類別碼；null = 適用所有商家 |
| bonusRate | String | 額外回饋率（例：`0.0300` = 3%） |
| descriptionZh | String | 優惠說明（中文） |
| descriptionEn | String | 優惠說明（英文） |
| effectiveFrom | String | 生效日（`YYYY-MM-DD`） |
| effectiveTo | String | 到期日（`YYYY-MM-DD`）；null = 長期有效 |

---

## 4. BFF UseCase Flow

**UseCase：** `CardBenefitsUseCase.getBenefits(CardBenefitsRq)`
**UseCase Impl：** `CardBenefitsUseCaseImpl`
**Adapter：** `CardFeignAdapter` implements `CardClientPort`
**Feign：** `CardFeignClient`（使用 `card-contract` 的 `CardBenefitsRq` / `CardBenefitsRs`）

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[VALIDATE]` | 驗證 `cardType` 為合法 enum 值 | 驗證失敗 → 拋出 `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` 呼叫 card-service → [card-service/card-benefits-zh-tw.md](../card-service/card-benefits-zh-tw.md) | 系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 3 | `[RETURN]` | 將 `CardBenefitsRs` 中繼回傳給 Client | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | `cardType` 為空或無效 |
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
