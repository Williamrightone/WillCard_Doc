# 查詢卡片資訊 — card-service Spec

> **服務：** card-service
> **Contract 模組：** card-contract
> **呼叫方：** bff → [mobile-bff/card-info-zh-tw.md](../mobile-bff/card-info-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 BFF 透過 Feign 轉發的卡片查詢請求。
驗證卡片存在且歸屬於請求用戶。
在 JVM Heap 解密 `expiry_date`、`cardholder_name_zh`、`cardholder_name_en`，組裝後回傳。
`pan_encrypted` 不包含於回應 — 僅回傳 `pan_masked`。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| context-path | `/card` |
| Controller mapping | `/info` |
| 實際接收路徑 | `/card/info` |
| 呼叫方 | BFF（`CardFeignClient`） |

### Request（`card.contract.dto.CardInfoRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 目標卡片的 Snowflake ID |

> **注意：** GET 請求，欄位以 Query Parameter 傳遞（`?cardId=...`）。

### Response（`card.contract.dto.CardInfoRs`）

| 欄位 | 型別 | 說明 | 備注 |
|------|------|------|------|
| cardId | String | Snowflake ID（String） | |
| maskedPan | String | `****-****-****-1234` | 直接由 DB 讀取，不需解密 |
| cardType | String | 卡種 | |
| cardNetwork | String | 卡片網路 | |
| expiryDate | String | `MM/YY` 格式 | 解密自 `expiry_date`（DB 儲存為加密 MMYY） |
| cardholderNameZh | String | 中文姓名 | 解密自 `cardholder_name_zh` |
| cardholderNameEn | String | 英文姓名 | 解密自 `cardholder_name_en` |
| status | String | ACTIVE / FROZEN / CANCELLED | |
| activatedAt | String | ISO-8601 時間戳記 | 未啟用時為 null |

---

## 3. Service UseCase Flow

**UseCase：** `CardInfoUseCase.getInfo(CardInfoRq)`（card-service）
**UseCase Impl：** `CardInfoUseCaseImpl`
**Repository：** `CardJpaRepository`（Spring Data JPA）

| 步驟 | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 以 `card_id` 查詢 `card`；不存在 → 拋出 `CARD_NOT_FOUND`（CA00001） | `CardQueryService.findById()` |
| 2 | `[DOMAIN]` | 驗證 `card.user_id == X-Member-Id`；不符 → 拋出 `CARD_NOT_FOUND`（CA00001，不暴露歸屬資訊） | `CardAuthorizationService.assertOwnership()` |
| 3 | `[DOMAIN]` | 解密 `expiry_date`（MMYY）→ 格式化為 `MM/YY`；解密 `cardholder_name_zh`；解密 `cardholder_name_en` | `CardEncryptionService.decrypt()` |
| 4 | `[RETURN]` | 組裝並回傳 `CardInfoRs`（maskedPan 直接取 DB 值；其餘欄位取 Step 3 解密結果） | — |

---

## 4. 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card` | READ | 以 `card_id` 查詢卡片記錄 |

### Redis

> 本端點不進行任何 Redis 操作。

### 資料表 Schema

#### card（本操作相關欄位）

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| card_id | BIGINT | NO | PK |
| user_id | BIGINT | NO | 歸屬驗證：須等於 X-Member-Id |
| card_type | VARCHAR(20) | NO | — |
| card_network | VARCHAR(10) | NO | — |
| status | VARCHAR(20) | NO | — |
| pan_masked | VARCHAR(20) | NO | 直接回傳 |
| cardholder_name_zh | VARCHAR(512) | NO | AES-256-GCM 加密；在 JVM Heap 解密 |
| cardholder_name_en | VARCHAR(512) | NO | AES-256-GCM 加密；在 JVM Heap 解密 |
| expiry_date | VARCHAR(512) | NO | AES-256-GCM 加密 MMYY；解密後格式化為 MM/YY |
| activated_at | DATETIME(3) | YES | — |

> 完整 card 資料表 Schema：請參閱 [card-apply-zh-tw.md](./card-apply-zh-tw.md#資料表-schema)。

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | CA00001 | CARD_NOT_FOUND | 卡片不存在，或 `user_id` 不符合 `X-Member-Id` |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
