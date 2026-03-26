# 卡片授權 Phase 1 — Card Service Biz Spec

## 概述

發卡機構授權 API — Phase 1。由 NCCC（正式環境）或 Mock NCCC（開發環境）直接呼叫。
在記憶體中解密加密卡片資料、識別持卡人、驗證卡片狀態與交易限額、計算點數折抵金額、
透過 Wallet Service 暫時凍結點數準備金、鎖定外幣匯率（非台幣交易）、產生 6 位數 OTP
並寫入 Redis pending auth 工作階段。
成功時回傳 `CHALLENGE_REQUIRED`、`challengeRef` 與 `pointsPreview`；
失敗時回傳 `DECLINED` 及拒絕原因代碼。

---

## API 定義 — card-service

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/auth/authorize` |
| 實際接收路徑 | `/card/auth/authorize` |
| 呼叫方 | NCCC（正式環境 mTLS）/ Mock NCCC Service（開發環境 Feign） |

### Request（CardAuthAuthorizeRq — card-contract）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| encryptedCard | String | ✅ | @NotBlank | NCCC 使用 combineKey 加密的卡片資料（PAN + 有效期限 + CVV；AES-256 GCM） |
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | 原始幣別交易金額 |
| currency | String | ✅ | @NotBlank, @Size(min=3,max=3) | ISO 4217 幣別代碼，如 `TWD`、`USD` |
| merchantId | String | ✅ | @NotBlank | 商家識別碼 |
| merchantMCC | String | ✅ | @NotBlank | 商家類別代碼（MCC），用於回饋率查詢 |

### Response（CardAuthAuthorizeRs — card-contract）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| status | String | `CHALLENGE_REQUIRED` / `DECLINED` | |
| challengeRef | String | OTP 工作階段識別碼（UUID） | `CHALLENGE_REQUIRED` 時存在 |
| pointsPreview.pointsToUse | Integer | 伺服端計算之點數折抵數量 | `CHALLENGE_REQUIRED` 時存在 |
| pointsPreview.cardAmount | BigDecimal | 折抵後刷卡金額（TWD） | `CHALLENGE_REQUIRED` 時存在 |
| declineReason | String | 拒絕原因代碼 | `DECLINED` 時存在 |

---

## Service UseCase Flow

**UseCase：** `CardAuthAuthorizeUseCase.authorize(CardAuthAuthorizeRq)`

| Step | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DOMAIN]` | 使用記憶體 combineKey（AES-256 GCM）解密 `encryptedCard` → 明文卡片 `{ pan, expiry, cvv }`，僅存 JVM Heap | `CardEncryptionService.decrypt()` |
| 2 | `[DB READ]` | 從解密後 PAN 計算 `panHash`（HMAC-SHA256）；以 `pan_hash` 查詢 `card` 表 | `CardQueryService.findByPanHash()` |
| 3 | `[DOMAIN]` | 驗證卡片狀態必須為 `ACTIVE`；解密 `expiry_date` 並確認未過期 | `CardValidationService.validateActive()` |
| 4 | `[DB READ]` | 讀取對應 `card_type` 的 `card_type_limit` 記錄 | `CardTypeLimitQueryService.findByCardType()` |
| 5 | `[REDIS READ]` | 讀取 `card:daily:{cardId}:{yyyyMMdd}` — 今日累計金額（key 不存在視為 0）；`INFINITE` 卡種跳過 | `DailyLimitService.getDailyAccumulated()` |
| 6 | `[DOMAIN]` | 驗證單筆上限：`amount_twd <= single_txn_limit`（null = 無上限，跳過） | `CardValidationService.checkSingleTxnLimit()` |
| 7 | `[DOMAIN]` | 驗證每日累計：`accumulated + amount_twd <= daily_limit`（null = 無上限，跳過） | `CardValidationService.checkDailyLimit()` |
| 8 | `[FEIGN]` | 呼叫 Wallet Service 取得 `{ availableBalance, pointsFirstEnabled, maxPointsPerTxn }` → [spec/wallet-service/get-member-balance-and-preference.md](../wallet-service/get-member-balance-and-preference.md) | — |
| 9 | `[DOMAIN]` | 計算 `pointsToUse`：若 `pointsFirstEnabled` → `min(availableBalance, amountTwd, maxPointsPerTxn ?? ∞)`；否則 `pointsToUse = 0`；`cardAmount = amountTwd - pointsToUse` | `PointsOffsetService.calculate()` |
| 10 | `[FEIGN]` | （非台幣才執行）呼叫 FX Service `lockRate(currency)` → `{ fxRateId, fxRate }`（TTL 10 分鐘）；計算 `amountTwd = floor(amount × fxRate)`、`fxFee = floor(amountTwd × fx_fee_rate)`、`totalTwd = amountTwd + fxFee` → [spec/fx-service/get-rate.md](../fx-service/get-rate.md) | — |
| 11 | `[DOMAIN]` | 驗證資金充足：`availableBalance - pointsToUse >= cardAmount`；不足 → `DECLINED + INSUFFICIENT_FUNDS` | `WalletValidationService.checkFunds()` |
| 12 | `[FEIGN]` | 呼叫 Wallet Service `reserve(userId, pointsToUse)` → `reservationId` → [spec/wallet-service/reserve.md](../wallet-service/reserve.md) | — |
| 13 | `[DOMAIN]` | 產生 `challengeRef`（UUID v4）；產生 6 位數密碼學安全亂數 OTP | `ChallengeService.generate()` |
| 14 | `[KAFKA]` | 發布 OTP 簡訊事件至 RabbitMQ（Notification Service 消費；payload：遮罩手機號碼 `09xx****xx`、OTP、challengeRef） | — |
| 15 | `[REDIS WRITE]` | 寫入 `otp:{challengeRef}` TTL 180s — payload：`{ cardId, userId, amountTwd, fxFee?, currency, merchantId, merchantMCC, pointsToUse, cardAmount, reservationId, fxRateId? }` | `PendingAuthStore.save()` |
| 16 | `[DOMAIN]` | 立即清除 JVM Heap 中的 `cvv` 明文 | `CardEncryptionService.clearSensitiveData()` |
| 17 | `[RETURN]` | 回傳 `CHALLENGE_REQUIRED` + `challengeRef` + `pointsPreview { pointsToUse, cardAmount }` | — |

> **PCI DSS：** 解密後的 PAN 與有效期限僅存 JVM Heap，生命週期僅限本 UseCase 執行期間。CVV 於 Step 16 立即清除。PAN 與 CVV 不得出現於任何 log、Redis key 或 Kafka event。

---

## 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card` | READ | 以 `pan_hash` 識別持卡人；讀取 `card_type`、`status`、`expiry_date`（加密）、`user_id` |
| `card_type_limit` | READ | 讀取卡種的 `single_txn_limit`、`daily_limit`、`fx_fee_rate` |

### Redis

| Key 模式 | 操作 | TTL | 說明 |
|----------|------|-----|------|
| `card:daily:{cardId}:{yyyyMMdd}` | READ | 當日 23:59:59 到期 | 今日累計交易金額（分，TWD） |
| `otp:{challengeRef}` | WRITE | 180s | Pending auth 工作階段 payload |

### 金鑰管理

`pan_hash` 以 **HMAC-SHA256** 計算，所用的 HMAC 密鑰與 AES-256 卡片加密金鑰採相同的注入模式：

```yaml
# application.yml（card-service）
card:
  security:
    pan-hmac-key: ${CARD_PAN_HMAC_KEY}   # 由 K8s Secret 於 runtime 注入
    aes-key:      ${CARD_AES_KEY}
```

```yaml
# k8s/card-service-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: card-service-secrets
type: Opaque
stringData:
  CARD_PAN_HMAC_KEY: "<base64-encoded-256-bit-key>"
  CARD_AES_KEY:      "<base64-encoded-256-bit-key>"
```

- 金鑰透過 `@ConfigurationProperties` 於啟動時載入，不得硬編碼。
- `pan_hash` 為**寫入一次**欄位（卡片申請時產生）；若需輪換 HMAC 金鑰，必須執行全量重新計算 hash 的 migration。
- 金鑰不得出現於 log、DB 欄位或任何 Kafka event。

### 資料表 Schema

#### card（相關欄位）

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| card_id | BIGINT | NO | PK，Snowflake |
| user_id | BIGINT | NO | 持卡會員 |
| card_type | VARCHAR(20) | NO | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` |
| card_network | VARCHAR(10) | NO | `VISA` / `MASTERCARD` / `JCB` |
| status | VARCHAR(20) | NO | `PENDING` / `ACTIVE` / `FROZEN` / `CANCELLED` |
| pan_encrypted | VARCHAR(512) | NO | AES-256 加密 PAN |
| pan_masked | VARCHAR(20) | NO | `****-****-****-1234` |
| pan_hash | VARCHAR(64) | NO | PAN 的 HMAC-SHA256（查詢用途） |
| expiry_date | VARCHAR(512) | NO | AES-256 加密，格式 `MMYY` |
| created_at | DATETIME(3) | NO | |
| updated_at | DATETIME(3) | NO | |

#### card_type_limit

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| card_type | VARCHAR(20) | NO | PK |
| single_txn_limit | BIGINT | YES | 單筆上限（分）；NULL = 無上限 |
| daily_limit | BIGINT | YES | 單日累計上限（分）；NULL = 無上限 |
| domestic_reward_rate | DECIMAL(5,4) | NO | 如 `0.0100` |
| overseas_reward_rate | DECIMAL(5,4) | NO | 如 `0.0500` |
| fx_fee_rate | DECIMAL(5,4) | NO | 如 `0.0100` |

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 422 | CA00001 `CARD_NOT_FOUND` | 找不到卡片 | 無 `pan_hash` 對應的卡片記錄 |
| 422 | CA00002 `CARD_NOT_ACTIVE` | 卡片非 ACTIVE 狀態 | `status ≠ ACTIVE` |
| 422 | CA00003 `CARD_EXPIRED` | 卡片已過期 | `expiry_date` 早於今日 |
| 422 | CA00004 `CARD_FROZEN` | 卡片已凍結 | `status = FROZEN` |
| 422 | CA00005 `CARD_CANCELLED` | 卡片已註銷 | `status = CANCELLED` |
| 422 | CA00006 `SINGLE_TXN_LIMIT_EXCEEDED` | 超過單筆交易上限 | `amount_twd > single_txn_limit` |
| 422 | CA00007 `DAILY_LIMIT_EXCEEDED` | 超過單日累計上限 | `accumulated + amount_twd > daily_limit` |
| 422 | `INSUFFICIENT_FUNDS` | 錢包餘額不足 | `availableBalance - pointsToUse < cardAmount` |
| 500 | `INTERNAL_ERROR` | 未預期的系統錯誤 | 未處理例外 |
