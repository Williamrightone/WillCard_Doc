# 卡片明文資料 — card-service Spec（內部 / 僅限開發）

> **服務：** card-service
> **環境：** 僅限開發環境
> **Contract Module：** card-service-contract（internal）
> **呼叫方：** [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

內部專用端點，提供給 Mock NCCC 在開發環境交易模擬中組裝 `encryptedCard` 所需的解密卡片資料。

從 `card` 表中解密 `pan_encrypted` 與 `expiry_date`，並回傳 `cvv_itoken`（CVV 的 HMAC-SHA256 Token，符合 PCI DSS 規定，不儲存原始 CVV），供 card-service `auth/authorize` 進行 CVV 驗證。

> **僅限 dev profile。** 此端點不得在正式環境中暴露。必須以 Spring Profile 條件（`@Profile("dev")` 或等效方式）保護，使該路由在非 dev 部署中無法存取。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（Mock NCCC） | [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md) |
| 回傳資料的使用方式 | [card-service/card-auth-authorize-zh-tw.md](card-auth-authorize-zh-tw.md) |

---

## 3. API 定義 — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| context-path | /card-service |
| Controller mapping | /internal/cards/{cardId}/plain |
| 實際接收路徑 | /card-service/internal/cards/{cardId}/plain |
| 呼叫方 | Mock NCCC（`CardServiceFeignClient`） |
| Profile 限制 | 僅 `dev` — 其他 profile 路由不存在 |

### Path Variable

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| cardId | String | ✅ | 卡片 Snowflake ID（String 格式） |

### Response（`card.service.internal.dto.CardPlainRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| pan | String | 完整明文 PAN（16 位數）；**不得記錄於 log** |
| expiryMMYY | String | MMYY 格式的有效期限，例如 `0329`；**不得記錄於 log** |
| cvvItoken | String | CVV 的 HMAC-SHA256 Token（儲存值）；不回傳原始 CVV |

---

## 4. Service UseCase Flow

**UseCase：** `CardPlainInternalUseCase.getPlainCard(String cardId)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | SELECT `card` WHERE `card_id = cardId`；不存在 → 404 `CARD_NOT_FOUND` |
| 2 | `[DOMAIN]` | 解密 `card.pan_encrypted`（AES-256-GCM，combineKey）→ `pan`；解密 `card.expiry_date`（AES-256-GCM）→ `expiryMMYY` |
| 3 | `[RETURN]` | 回傳 `CardPlainRs { pan, expiryMMYY, cvvItoken: card.cvv_itoken }` |

> `pan` 與 `expiryMMYY` 僅存在於本次呼叫的 JVM heap 中。`CardPlainRs` DTO **絕不可**序列化至任何 log、指標或持久化儲存（PCI DSS）。
> `cvvItoken` 為儲存於 `card.cvv_itoken` 的 HMAC-SHA256 Token，原樣回傳。

---

## 5. 資料庫

**讀取表：** `card`

| 欄位 | 用途 |
|------|------|
| `card_id` | 查詢鍵 |
| `pan_encrypted` | AES-256-GCM 解密 → `pan` |
| `expiry_date` | AES-256-GCM 解密 → `expiryMMYY` |
| `cvv_itoken` | 原樣回傳於 Response |

---

## 6. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | CA00001 | CARD_NOT_FOUND | 找不到對應 `cardId` 的卡片 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 7. Changelog

### v1.0 — 2026-03 — 初始版本
