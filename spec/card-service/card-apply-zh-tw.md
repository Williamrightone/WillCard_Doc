# 申請虛擬卡 — card-service Spec

> **服務：** card-service
> **Contract 模組：** card-contract
> **呼叫方：** bff → [mobile-bff/card-apply-zh-tw.md](../mobile-bff/card-apply-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 BFF 透過 Feign 轉發的申請請求。
驗證用戶是否已持有相同卡種的有效卡。
產生模擬 PAN（BIN 前綴 + `SecureRandom` + Luhn 驗證），衍生 `pan_masked` 和 `pan_hash`，產生一次性 CVV，計算有效期（申請日 + 5 年），以 AES-256-GCM 加密所有敏感欄位，並以 `status = ACTIVE` 寫入卡片記錄。
將卡片資訊（含一次性 CVV）回傳給 BFF。

> **CVV：** 以 `SecureRandom` 產生。絕不儲存於 DB、Redis 或 Log。僅存在於本次請求的 JVM Heap，並包含於 Response 中一次性回傳。

> **pan_hash：** 完整 PAN 的 HMAC-SHA256。建立唯一索引。Phase 7（Issuer 授權）需以 PAN 查詢持卡人，此欄位為查詢依據。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/apply` |
| 實際接收路徑 | `/card/apply` |
| 呼叫方 | BFF（`CardFeignClient`） |

### Request（`card.contract.dto.CardApplyRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardType | String | ✅ | @NotBlank | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| cardNetwork | String | ✅ | @NotBlank | VISA / MASTERCARD / JCB |
| cardholderNameEn | String | ✅ | @NotBlank | 英文持卡人姓名 |
| cardholderNameZh | String | ✅ | @NotBlank | 中文持卡人姓名 |

### Response（`card.contract.dto.CardApplyRs`）

| 欄位 | 型別 | 說明 | 備注 |
|------|------|------|------|
| cardId | String | Snowflake ID（序列化為 String） | |
| maskedPan | String | `****-****-****-1234` | |
| cardType | String | 卡種 | |
| cardNetwork | String | 卡片網路 | |
| expiryDate | String | `MM/YY` 格式（解密後回傳） | DB 儲存為加密 MMYY |
| cardholderNameEn | String | 英文持卡人姓名 | 解密後回傳；儲存時已加密 |
| cvv | String | 3 位 CVV | 一次性回傳；不儲存 |
| status | String | `ACTIVE` | |
| activatedAt | String | ISO-8601 時間戳記 | |

---

## 3. Service UseCase Flow

**UseCase：** `CardApplyUseCase.apply(CardApplyRq)`（card-service）
**UseCase Impl：** `CardApplyUseCaseImpl`
**Repository：** `CardJpaRepository`（Spring Data JPA）

| 步驟 | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 查詢 `card` WHERE `user_id = X-Member-Id` AND `card_type = cardType` AND `status != 'CANCELLED'`；查到資料 → 拋出 `CARD_ALREADY_EXISTS`（CA00009） | `CardQueryService.findActiveByUserAndType()` |
| 2 | `[DOMAIN]` | 產生 16 位 PAN：依 cardNetwork 選擇 BIN 前綴（VISA：`4`，MC：`51`，JCB：`35`），以 `SecureRandom` 填充剩餘位數，Luhn 演算法驗證 | `CardNumberService.generate(cardNetwork)` |
| 3 | `[DOMAIN]` | 產生 `pan_masked`：前 12 位替換為 `****-****-****`，保留末 4 位 | `CardNumberService.mask(pan)` |
| 4 | `[DOMAIN]` | 產生 `pan_hash`：`HMAC-SHA256(pan, hmacKey)` | `CardEncryptionService.hash(pan)` |
| 5 | `[DOMAIN]` | 產生 CVV：`SecureRandom` 3 位數字字串 | `CardNumberService.generateCvv()` |
| 6 | `[DOMAIN]` | 計算有效期：`申請日 + 5 年`，儲存格式 `MMYY`，回傳格式 `MM/YY` | `CardExpiryService.calculate()` |
| 7 | `[DOMAIN]` | AES-256-GCM 加密：`pan` → `pan_encrypted`；`cardholderNameEn` → `cardholder_name_en`；`cardholderNameZh` → `cardholder_name_zh`；`MMYY` → `expiry_date` | `CardEncryptionService.encrypt()` |
| 8 | `[DB WRITE]` | 新增 `card` 記錄：`card_id` = Snowflake，`user_id` = X-Member-Id，`card_type`，`card_network`，`status = ACTIVE`，`pan_encrypted`，`pan_masked`，`pan_hash`，`cardholder_name_zh`，`cardholder_name_en`，`expiry_date`，`activated_at = now()` | `CardJpaRepository.save()` |
| 9 | `[RETURN]` | 回傳 `CardApplyRs`（cardId、maskedPan、cardType、cardNetwork、expiryDate `MM/YY`、cardholderNameEn 解密值、cvv、status = `ACTIVE`、activatedAt）— **CVV 僅在 JVM Heap 中傳遞，不持久化** | — |

---

## 4. 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card` | READ | 檢查是否已有相同 `(user_id, card_type)` 的有效卡 |
| `card` | WRITE | 新增卡片記錄 |

### Redis

> 本端點不進行任何 Redis 操作。

### 資料表 Schema

#### card

| 欄位 | 型別 | Nullable | 預設值 | 說明 |
|------|------|----------|--------|------|
| card_id | BIGINT | NO | — | PK，Snowflake ID（應用層產生） |
| user_id | BIGINT | NO | — | 會員 ID；參照完整性由應用層確保 |
| card_type | VARCHAR(20) | NO | — | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| card_network | VARCHAR(10) | NO | — | VISA / MASTERCARD / JCB |
| status | VARCHAR(20) | NO | — | PENDING / ACTIVE / FROZEN / CANCELLED |
| pan_encrypted | VARCHAR(512) | NO | — | AES-256-GCM 加密完整卡號 |
| pan_masked | VARCHAR(20) | NO | — | `****-****-****-1234`；明文儲存供顯示用 |
| pan_hash | VARCHAR(64) | NO | — | PAN 的 HMAC-SHA256；供 Phase 7 PAN 查詢使用；**Unique Index** |
| cardholder_name_zh | VARCHAR(512) | NO | — | AES-256-GCM 加密中文姓名 |
| cardholder_name_en | VARCHAR(512) | NO | — | AES-256-GCM 加密英文姓名 |
| expiry_date | VARCHAR(512) | NO | — | AES-256-GCM 加密 MMYY |
| activated_at | DATETIME(3) | YES | NULL | 首次 ACTIVE 時間戳記；寫入後不覆寫 |
| cancelled_at | DATETIME(3) | YES | NULL | CANCELLED 時間戳記；寫入後不覆寫 |
| created_at | DATETIME(3) | NO | — | 繼承 BaseTimeEntity |
| updated_at | DATETIME(3) | NO | — | 繼承 BaseTimeEntity |

> **應用層限制：** `(user_id, card_type)` WHERE `status != 'CANCELLED'` — 由 Step 1 重複檢查執行，非 DB 層面條件唯一約束。
> **Unique Index：** `pan_hash` — 確保所有用戶的 PAN 不重複。

#### card_type_limit（種子資料，唯讀）

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| card_type | VARCHAR(20) | NO | PK：CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| single_txn_limit | BIGINT | YES | 單筆上限（分）；NULL = 無上限 |
| daily_limit | BIGINT | YES | 單日累計上限（分）；NULL = 無上限 |
| domestic_reward_rate | DECIMAL(5,4) | NO | 國內基礎回饋率（例：0.0100 = 1%） |
| overseas_reward_rate | DECIMAL(5,4) | NO | 國外基礎回饋率 |
| fx_fee_rate | DECIMAL(5,4) | NO | 跨國手續費率 |
| created_at | DATETIME(3) | NO | — |
| updated_at | DATETIME(3) | NO | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 409 | CA00009 | CARD_ALREADY_EXISTS | 用戶已有相同卡種的有效卡（PENDING / ACTIVE / FROZEN） |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
