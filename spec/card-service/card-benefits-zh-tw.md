# 查詢特約優惠 — card-service Spec

> **服務：** card-service
> **Contract 模組：** card-contract
> **呼叫方：** bff → [mobile-bff/card-benefits-zh-tw.md](../mobile-bff/card-benefits-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 BFF 透過 Feign 轉發的特約優惠查詢請求。
查詢 `card_type_merchant_benefit` 中符合指定 `card_type` 且目前有效（今日介於 `[effective_from, effective_to]` 之間，或 `effective_to IS NULL`）的所有記錄。
回傳符合條件的優惠清單。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| context-path | `/card` |
| Controller mapping | `/benefits` |
| 實際接收路徑 | `/card/benefits` |
| 呼叫方 | BFF（`CardFeignClient`） |

### Request（`card.contract.dto.CardBenefitsRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardType | String | ✅ | @NotBlank | CLASSIC / OVERSEAS / PREMIUM / INFINITE |

> **注意：** GET 請求，欄位以 Query Parameter 傳遞（`?cardType=CLASSIC`）。

### Response（`card.contract.dto.CardBenefitsRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardType | String | 查詢的卡種 |
| benefits | List\<CardBenefitItem\> | 有效優惠清單；無資料時為空清單 |

**CardBenefitItem：**

| 欄位 | 型別 | 說明 |
|------|------|------|
| benefitId | String | Snowflake ID（String） |
| merchantId | String | 商家 ID；null = 適用整個 MCC |
| mccCode | String | MCC 碼；null = 適用所有商家 |
| bonusRate | BigDecimal | 額外回饋率（例：0.0300） |
| descriptionZh | String | 中文說明 |
| descriptionEn | String | 英文說明 |
| effectiveFrom | String | 生效日（`YYYY-MM-DD`） |
| effectiveTo | String | 到期日（`YYYY-MM-DD`）；null = 長期有效 |

---

## 3. Service UseCase Flow

**UseCase：** `CardBenefitsUseCase.getBenefits(CardBenefitsRq)`（card-service）
**UseCase Impl：** `CardBenefitsUseCaseImpl`
**Repository：** `CardBenefitJpaRepository`（Spring Data JPA）

| 步驟 | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 查詢 `card_type_merchant_benefit` WHERE `card_type = ?` AND `effective_from <= today` AND（`effective_to IS NULL` OR `effective_to >= today`）；依 `effective_from ASC` 排序 | `CardBenefitQueryService.findActiveByType()` |
| 2 | `[RETURN]` | 組裝並回傳 `CardBenefitsRs` | — |

---

## 4. 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card_type_merchant_benefit` | READ | 依 `card_type` 及日期範圍查詢有效優惠 |

### Redis

> 本端點不進行任何 Redis 操作。

### 資料表 Schema

#### card_type_merchant_benefit（種子資料，唯讀）

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| benefit_id | BIGINT | NO | PK，Snowflake ID |
| card_type | VARCHAR(20) | NO | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| merchant_id | VARCHAR(50) | YES | 特定商家；null = 適用整個 MCC |
| mcc_code | VARCHAR(10) | YES | MCC 類別；null = 適用所有商家 |
| bonus_rate | DECIMAL(5,4) | NO | 疊加於 base + mcc_bonus 之上的額外回饋率 |
| description_zh | VARCHAR(255) | NO | 優惠說明（中文） |
| description_en | VARCHAR(255) | NO | 優惠說明（英文） |
| effective_from | DATE | NO | 優惠生效日 |
| effective_to | DATE | YES | 優惠到期日；null = 長期有效 |
| created_at | DATETIME(3) | NO | — |
| updated_at | DATETIME(3) | NO | — |

> **查詢條件：** `effective_from <= today AND (effective_to IS NULL OR effective_to >= today)`

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | CA00008 | CARD_TYPE_NOT_FOUND | 無法識別的 `cardType` 值 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
