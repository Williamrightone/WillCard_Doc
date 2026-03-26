# Mock NCCC 刷卡 — Biz Spec

## 概述

**Mock NCCC 為獨立的 Spring Boot 微服務**（Maven module：`mock-nccc`，
僅部署於開發環境的 Kubernetes cluster）。用於模擬 NCCC 的入站授權呼叫，
讓 App 可在沒有真實 NCCC 連線的情況下完整端對端測試。

接收 App（透過 BFF Feign）的付款請求，向 Card Service 取得會員的加密卡片資料，
於本地解密後以 mock combineKey 重新加密（模擬 NCCC 的加密角色），
再將 `encryptedCard` payload 轉送至 Card Service 發卡機構授權 API。
成功時回傳 `CHALLENGE_REQUIRED`、`challengeRef` 與 `pointsPreview`。

**部署說明：**
- Maven module：`mock-nccc` — 獨立可部署 jar，與 `card-service` 完全分離
- `server.servlet.context-path: /mock-nccc`
- K8s：僅在 `dev` namespace 部署單一 Replica Deployment；無 HPA，無正式環境 manifest
- mock combineKey 透過 K8s Secret 注入：`${MOCK_NCCC_COMBINE_KEY}`

> **適用範圍：** 僅限開發與整合測試環境，不得部署至正式環境。

---

## API 定義 — mock-nccc

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| context-path | `/mock-nccc` |
| Controller mapping | `/pay` |
| 實際接收路徑 | `/mock-nccc/pay` |
| 呼叫方 | BFF（FeignClient） |

### Request（MockNcccPayRq — mock-nccc-contract）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| cardId | String | ✅ | @NotBlank | 會員虛擬卡識別碼（Snowflake ID 以字串傳遞） |
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | 原始幣別交易金額 |
| currency | String | ✅ | @NotBlank, @Size(min=3,max=3) | ISO 4217 幣別代碼，如 `TWD`、`USD` |
| merchantId | String | ✅ | @NotBlank | 商家識別碼 |

### Response（MockNcccPayRs — mock-nccc-contract）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| status | String | `CHALLENGE_REQUIRED` / `DECLINED` | |
| challengeRef | String | OTP 工作階段參考碼 | `CHALLENGE_REQUIRED` 時存在 |
| pointsPreview.pointsToUse | Integer | 伺服端計算之點數折抵數量 | `CHALLENGE_REQUIRED` 時存在 |
| pointsPreview.cardAmount | BigDecimal | 折抵後刷卡金額（TWD） | `CHALLENGE_REQUIRED` 時存在 |
| declineReason | String | 拒絕原因代碼 | `DECLINED` 時存在 |

---

## Service UseCase Flow

**UseCase：** `MockNcccPayUseCase.pay(MockNcccPayRq)`

| Step | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[FEIGN]` | 呼叫 Card Service 內部開發端點 `GET /card/internal/raw-data/{cardId}` 取得 `{ panEncrypted, expiryDateEncrypted }`（僅限開發環境，正式環境不暴露）→ [spec/card-service/card-internal-raw-data.md](../card-service/card-internal-raw-data.md) | — |
| 2 | `[DOMAIN]` | 使用 AES-256 解密 `panEncrypted` 及 `expiryDateEncrypted`（開發環境共用金鑰，透過環境變數注入） | `MockCardDecryptionService.decrypt()` |
| 3 | `[DOMAIN]` | 組裝 mock 卡片 payload `{ pan, expiry, cvv="000" }`（CVV 為開發固定值）；使用 mock combineKey（AES-256 GCM）加密 → `encryptedCard` | `MockCombineKeyService.encrypt()` |
| 4 | `[DOMAIN]` | 依 `merchantId` 從記憶體 mock MCC 映射表解析 `merchantMCC`（啟動時以 seed data 載入） | `MockMerchantMccResolver.resolve()` |
| 5 | `[FEIGN]` | 呼叫 Card Service `POST /card/auth/authorize`，帶入 `{ encryptedCard, amount, currency, merchantId, merchantMCC }` → [spec/card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md) | — |
| 6 | `[RETURN]` | 將 `CardAuthAuthorizeRs` 直接作為 `MockNcccPayRs` 回傳 | — |

---

## 資料庫

### MySQL

_（無 — Mock NCCC 為無狀態服務；所有狀態由 Card Service 管理）_

### Redis

_（無）_

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 422 | `CARD_NOT_FOUND` | 找不到卡片 | 從 Card Service CA00001 透傳 |
| 422 | `CARD_NOT_ACTIVE` | 卡片非 ACTIVE 狀態 | 從 Card Service CA00002 透傳 |
| 422 | `CARD_EXPIRED` | 卡片已過期 | 從 Card Service CA00003 透傳 |
| 422 | `CARD_FROZEN` | 卡片已凍結 | 從 Card Service CA00004 透傳 |
| 422 | `CARD_CANCELLED` | 卡片已註銷 | 從 Card Service CA00005 透傳 |
| 422 | `SINGLE_TXN_LIMIT_EXCEEDED` | 超過單筆交易上限 | 從 Card Service CA00006 透傳 |
| 422 | `DAILY_LIMIT_EXCEEDED` | 超過單日累計上限 | 從 Card Service CA00007 透傳 |
| 422 | `INSUFFICIENT_FUNDS` | 錢包餘額不足 | 從 Card Service 透傳 |
| 500 | `INTERNAL_ERROR` | 未預期的系統錯誤 | 未處理例外 |
