# 信用卡交易功能開發順序

## ✅ Phase 1–4 已完成

| Phase | 已完成內容 |
|-------|-----------|
| Phase 1 | 卡片 CRUD（apply / info / freeze / unfreeze / benefits），card_db schema，card_key_parts DDL |
| Phase 2 | Wallet Service：wallet_account / reservation / point_reward_batch，Reserve / Confirm / Release 核心機制 |
| Phase 3 | Points Service：Kafka Consumer `txn.card.authorized` → 計算回饋（base_rate）→ wallet credit API |
| Phase 4 | FX Service：GET /fx/rate、POST /fx/convert（Cache-Aside Redis，台灣銀行 Spot Rate） |

### 卡片種類

| card_type | 國內回饋 | 國外回饋 | 跨國手續費 | 單筆上限 | 單日上限 |
|-----------|---------|---------|-----------|---------|---------|
| CLASSIC | 1% | 0% | 1.5% | NT$100,000 | NT$200,000 |
| OVERSEAS | 0% | 5% | 1% | NT$100,000 | NT$200,000 |
| PREMIUM | 2% | 5% | 1% | NT$200,000 | NT$500,000 |
| INFINITE | 2% | 5% | 1% | 無上限 | 無上限 |

### 卡片狀態機

`PENDING → ACTIVE → FROZEN → CANCELLED`
（合法轉移：PENDING→ACTIVE、ACTIVE→FROZEN、FROZEN→ACTIVE、ACTIVE→CANCELLED、FROZEN→CANCELLED）

### 卡片相關 Redis Key

| Key | 說明 | TTL |
|-----|------|-----|
| `card:daily:{cardId}:{yyyyMMdd}` | 當日累計交易金額（分）；INFINITE 跳過 | 當日 23:59:59 |

### 點數計算

```
reward_points = floor(txnTwdBaseAmount × base_rate)
isOverseas = true  → overseas_reward_rate
isOverseas = false → domestic_reward_rate
```

### 錯誤碼（CardServiceException，ModelType: `CA`）

| Code | Name | 說明 |
|------|------|------|
| CA00001 | CARD_NOT_FOUND | 找不到卡片 |
| CA00002 | CARD_NOT_ACTIVE | 卡片非 ACTIVE 狀態 |
| CA00003 | CARD_EXPIRED | 卡片已過期 |
| CA00004 | CARD_FROZEN | 卡片已凍結 |
| CA00005 | CARD_CANCELLED | 卡片已取消 |
| CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | 超過單筆交易限額 |
| CA00007 | DAILY_LIMIT_EXCEEDED | 超過當日累計限額 |
| CA00008 | CARD_TYPE_NOT_FOUND | 卡種設定不存在 |
| CA00009 | CARD_ALREADY_EXISTS | 同卡種已有有效卡 |
| CA00010 | KEY_PART_NOT_FOUND | card_key_parts 無指定 partSeq 資料（Phase 5） |
| CA00011 | KCV_MISMATCH | Source Key KCV 驗證不符（Phase 5） |
| CA00012 | INVALID_KEY_FORMAT | partSeq 不合法或 sourceKeyHex 非有效 64 位十六進位（Phase 5） |
| CA00013 | COMBINE_KEY_UNAVAILABLE | combineKey 載入失敗，auth/authorize 無法執行（Phase 5） |
| CA00014 | CVV_ITOKEN_MISMATCH | 收到的 cvvItoken 與 DB 儲存值不符（Phase 8） |

---

## Phase 5 — AES-256 / combineKey 加解密（3DS Encryption）Spec Ready

**目標：** 建立授權流程中的卡片資料加解密機制，為 Phase 8 auth/authorize 提供 combineKey

> 完整架構設計請參閱 [guideline/14-3ds-combine-key-zh-tw.md](guideline/14-3ds-combine-key-zh-tw.md)

- [ ] **combineKey 基礎設施**（card-service）
  - `card_key_parts` 表（card_db，Flyway migration `V5.0.0`）：
    - `part_seq` INT PK（1/2/3）、`encrypted_part` VARCHAR(512)、`updated_at` DATETIME(3)
    - 三把 Source Key 各自以 Master Key（AES-256-GCM）加密儲存
    - Master Key 由 KMS / 環境變數注入，不寫入 DB
  - 初始播種（`V5.0.1__seed_card_key_parts.sql`）
  - `ApplicationReadyEvent`：讀 `card_key_parts` → 解密三 part → XOR 合成 combineKey，常駐 JVM Heap（`CombineKeyHolder`）
  - `@PreDestroy`：JVM 關閉時清零 Heap 中的 combineKey 位元組陣列
  - 降級啟動：Part 不足 3 筆或解密失敗 → combineKey = null → auth/authorize 回傳 HTTP 503（CA00013）
  - spec: [spec/card-service/combine-key-startup.md](spec/card-service/combine-key-startup.md)

- [ ] **Source Key 管理 API**（card-service，內部）
  - 更新 Source Key（含 KCV 驗證）— spec: [spec/card-service/source-key-update.md](spec/card-service/source-key-update.md)
    - `PUT /card-service/internal/keys/source-key/{partSeq}`
    - Request：`{ sourceKeyHex: 64 hex chars, kcv: 6 hex chars }`
    - KCV 驗證通過 → 加密存入 DB → 熱更新 combineKey
  - 驗證 Source Key KCV — spec: [spec/card-service/source-key-kcv.md](spec/card-service/source-key-kcv.md)
    - `POST /card-service/internal/keys/source-key/{partSeq}/kcv/verify`
    - Request：`{ kcv: 6 hex chars }`；Response：`{ matched: Boolean, updatedAt }`

- [ ] **Mock 環境**：combineKey 固定值設定（`application-dev.yml willcard.security.mock-combine-key`），供 Mock NCCC 使用相同金鑰加密

---

## Phase 6 — BFF Card Pay API（App 入口）

**目標：** 對 App 暴露刷卡與 OTP 確認兩支入口，呼叫 Mock NCCC

- [ ] **`POST /api/v1/card/pay`**（Nginx strip → BFF `/card/pay`）
  - Request：`{ cardId, amount, currency, merchantId, usePoints: Boolean }`
  - JWT 驗證 + X-User-Id header 注入
  - 冪等控制（BFF Redis idempotency key，TTL 60s）
  - Feign → `POST /mock-nccc/pay`
  - Response：`{ challengeRef, pointsToUse, estimatedTwdAmount }`
  - Operation Log L1：Kafka `operation-log.card.pay`
  - spec: `spec/mobile-bff/card-pay.md`（待補）

- [ ] **`POST /api/v1/card/pay/confirm`**（Nginx strip → BFF `/card/pay/confirm`）
  - Request：`{ challengeRef, otp }`
  - Feign → `POST /mock-nccc/pay/confirm`
  - Response：`{ result: APPROVED | DECLINED, reason? }`
  - Operation Log L1：Kafka `operation-log.card.pay-confirm`
  - spec: `spec/mobile-bff/card-pay-confirm.md`（待補）

- [ ] Error mapping：Card Service 錯誤碼（CA000xx）→ BFF 對外錯誤碼

---

## Phase 7 — Mock NCCC Service（開發環境）

**目標：** 模擬 NCCC 行為，橋接 BFF ↔ card-service，讓整體流程可端對端測試

> **設計說明：** 真實環境中 App 直接與 NCCC iframe 互動，NCCC 收到使用者輸入的卡片資料後以 combineKey 加密再呼叫我方 card-service。
> Mock NCCC 代為模擬此行為：從 card-service 取得卡片明文資料 → 以固定 mock combineKey 加密 → 呼叫 auth/authorize。

- [ ] **`POST /mock-nccc/pay`**
  - Request（來自 BFF）：`{ cardId, amount, currency, merchantId, usePoints }`
  - 呼叫 card-service 內部 API 取得 `{ pan, expiryMMYY, cvvItoken }`
  - 以 `MOCK_COMBINE_KEY` 組裝 `encryptedCard`（AES-256-GCM）：`{ pan, expiryMMYY, cvvItoken }`
  - Feign → `POST /card-service/auth/authorize`（encryptedCard、amount、currency、merchantId、usePoints）
  - 回傳 `{ challengeRef, pointsToUse, estimatedTwdAmount }` 給 BFF
  - spec: `spec/mock-nccc/pay.md`（待補）

- [ ] **`POST /mock-nccc/pay/confirm`**
  - Request（來自 BFF）：`{ challengeRef, otp }`
  - Feign → `POST /card-service/auth/verify-challenge`
  - 回傳最終結果給 BFF：`{ result: APPROVED | DECLINED, reason? }`
  - spec: `spec/mock-nccc/pay-confirm.md`（待補）

- [ ] **card-service 內部卡片查詢端點**（供 Mock NCCC 取得明文卡片資料）
  - `GET /card-service/internal/cards/{cardId}/plain`（內部路由，Nginx 封鎖外部流量）
  - Response：`{ pan, expiryMMYY, cvvItoken }`（明文，僅 dev/mock profile 啟用；cvvItoken 從 DB 讀出，供 Mock NCCC 組裝 encryptedCard 使用）
  - spec: `spec/card-service/card-plain.md`（待補）

---

## Phase 8 — card-service auth/authorize（OTP 生成 + 點數預授權 + SMS）

**目標：** 實作 NCCC 呼叫的主要授權端點，完成解密驗卡、限額核查、點數預留、OTP 發送

> spec: `spec/card-service/card-auth-authorize.md`（待補）

### Redis OTP Session

- Key：`otp:{challengeRef}`
- TTL：180 秒
- Payload：

  ```json
  {
    "userId": 123,
    "cardId": 456,
    "txnAmount": 10000,
    "txnCurrency": "USD",
    "txnTwdBaseAmount": 316000,
    "isOverseas": true,
    "pointsToUse": 5000,
    "reservationId": 789,
    "merchantId": "MERCHANT_001",
    "otpValue": "123456"
  }
  ```

  > `txnTwdBaseAmount`：台幣交易等於 txnAmount；外幣呼叫 `POST /fx/convert` 換算（分）。
  > `reservationId`：若 pointsToUse = 0 則為 null。

### UseCase Flow（`POST /card-service/auth/authorize`）

| Step | 說明 |
|------|------|
| 1 | 以 `CombineKeyHolder` 取得 combineKey；null → 拋 CA00013 HTTP 503 |
| 2 | AES-256-GCM 解密 `encryptedCard` → `{ pan, expiryMMYY, cvvItoken }` |
| 3 | HMAC-SHA256(pan) → pan_hash，查詢 DB 識別 userId / cardId |
| 4 | 比對 cvvItoken：received cvvItoken == card.cvv_itoken；不符 → CA00014 CVV_ITOKEN_MISMATCH |
| 5 | 驗卡：status = ACTIVE、expiry 未過期（CA00002 / CA00003 / CA00004） |
| 6 | 幣別判斷：TWD → txnTwdBaseAmount = amount；外幣 → Feign `POST /fx/convert` |
| 7 | 單筆限額：txnTwdBaseAmount ≤ single_txn_limit（INFINITE 跳過）→ CA00006 |
| 8 | 當日累計：Redis `card:daily:{cardId}:{yyyyMMdd}` + txnTwdBaseAmount ≤ daily_limit → CA00007 |
| 9 | 點數折抵：`pointsToUse = usePoints ? min(available_balance, txnTwdBaseAmount) : 0` |
| 10 | 若 pointsToUse > 0：Feign `wallet-service reserve(userId, pointsToUse)` → reservationId |
| 11 | 產生 6 位 OTP（SecureRandom） |
| 12 | 發布 RabbitMQ `notification.otp.sms`（userId、手機號遮罩、otp）→ notify-service 送 SMS |
| 13 | 儲存 Redis OTP Session（TTL 180s） |
| 14 | 清零 PAN 明文位元組（try-finally 保證執行） |
| 15 | Response：`{ challengeRef, pointsToUse, estimatedTwdAmount }` |

> 驗卡失敗 / 限額超過 → 若已 reserve 則呼叫 `wallet-service release(reservationId)`。

### Notification Service — OTP SMS

- RabbitMQ Consumer：Queue `notification.otp.sms`
- 手機號碼遮罩 log（`09xx****xx`）
- SMS 範本：`您的 WillCard 交易驗證碼為 {otp}，有效期限 3 分鐘，請勿告知他人。`

---

## Phase 9 — card-service auth/verify-challenge（OTP 驗證 + 授權完成）

**目標：** 驗證使用者輸入的 OTP，觸發點數確認與 Kafka 事件發布

> spec: `spec/card-service/card-auth-verify-challenge.md`（待補）

### UseCase Flow（`POST /card-service/auth/verify-challenge`）

| Step | 說明 |
|------|------|
| 1 | Redis `otp:{challengeRef}` 不存在 → `DECLINED + SESSION_EXPIRED` |
| 2 | 比對 OTP 值（session.otpValue == 輸入值） |
| **OTP 正確** | |
| 3 | 若有 reservationId → Feign `wallet-service confirmDeduct(reservationId)` |
| 4 | Redis INCR `card:daily:{cardId}:{yyyyMMdd}`（加 txnTwdBaseAmount，TTL 至當日 23:59:59） |
| 5 | 分配 txnId（Snowflake） |
| 6 | Kafka Publish `txn.card.authorized`（見 Kafka Payload） |
| 7 | 刪除 Redis OTP Session key |
| 8 | Response：`{ result: APPROVED, txnId }` |
| **OTP 錯誤** | |
| 9 | INCR `otp:attempt:{challengeRef}`（TTL 180s）；≥ 3 → session 作廢，若有 reservationId → release；Response：`DECLINED + SESSION_VOIDED` |
| 10 | INCR `otp:card:fail:{cardId}`（TTL 900s）；≥ 5 → 卡片 FROZEN + Kafka `card.risk.otp-threshold-exceeded` + release；Response：`DECLINED + CARD_LOCKED` |
| 11 | Response：`DECLINED + OTP_FAILED`（含剩餘次數） |

### Kafka Payload — `txn.card.authorized`

```json
{
  "txnId": "Snowflake ID",
  "userId": 123,
  "cardId": 456,
  "cardType": "OVERSEAS",
  "maskedPan": "****-****-****-1234",
  "merchantId": "MERCHANT_001",
  "txnAmount": 10000,
  "txnCurrency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "pointsToUse": 5000,
  "txnTimestamp": "2026-03-30T10:00:00.000Z"
}
```

---

## Phase 10 — 交易後非同步處理 + 帳本（Post-Auth Async + Ledger）

**目標：** 處理 `txn.card.authorized` 事件的所有下游消費

> **架構說明：** Points Service、Ledger Service、Notification Service 各自獨立訂閱同一 Kafka Topic（fan-out），互不依賴。

### Points Service（Phase 3 設計，Phase 10 實作）

- Kafka Consumer：`txn.card.authorized`（冪等：`uk_pil_source_txn_id`）
- 計算 reward_points → 呼叫 wallet-service credit API → 建立 `point_reward_batch`（PENDING）
- 到期日：txnTimestamp 所在月份一年後同月最後一刻

### Ledger Service

- [ ] `journal_entry` 表（ledger_db，insert-only，無 updated_at）
  - `entry_id` BIGINT Snowflake PK
  - `txn_id` VARCHAR(64)
  - `entry_seq` INT（同一交易內序號）
  - `account_code` VARCHAR(10)（1001 應收款、2001 代付款、3001 台幣收入）
  - `debit_amount` / `credit_amount` BIGINT（分）
  - `currency` VARCHAR(3)
  - `created_at` DATETIME(3)
  - `idempotency_key` VARCHAR(128) UNIQUE — `{txnId}_{entrySeq}`

- [ ] Kafka Consumer：`txn.card.authorized`（冪等：ON DUPLICATE KEY IGNORE）

- [ ] 三種情境分錄：
  - **台幣、無折抵**：DR 應收款(1001) / CR 代付款(2001) + CR 台幣收入(3001)
  - **台幣、點數折抵**（pointsToUse > 0）：額外加點數折抵科目 CR/DR 對沖
  - **外幣**（isOverseas = true）：額外加 DR/CR 外幣手續費（txnTwdBaseAmount × fx_fee_rate）

- [ ] SETTLEMENT 分錄（T+1 排程）：DR 代付款(2001) / CR 清算款(1002)

### Notification Service — 交易收據

- Kafka Consumer：`txn.card.authorized` → 推播 + Email 消費明細
- Queue：`notification.txn.receipt`（內部轉發）

---

## Phase 11 — 整合測試（End-to-End）

**目標：** 驗證完整交易流程的所有路徑

- [ ] 正常台幣刷卡（usePoints = false）：App → BFF → Mock NCCC → card-service → Kafka → Ledger / Notification
- [ ] 台幣刷卡 + 點數全額折抵（available_balance >= txnAmount）
- [ ] 台幣刷卡 + 點數部分折抵（available_balance < txnAmount）
- [ ] 外幣交易（USD）：驗證 fx-service convert、txnTwdBaseAmount、fx_fee 分錄
- [ ] OTP 失敗 → 重試 → 第 3 次授權作廢（per-session）→ wallet release
- [ ] 卡片鎖定（per-card ≥ 5 次 / 15 分鐘）→ Kafka `card.risk.otp-threshold-exceeded` + wallet release
- [ ] Session 過期（Redis TTL 180s 到期 → SESSION_EXPIRED）
- [ ] 單筆限額超過 → DECLINED（CA00006），無 reserve
- [ ] 當日累計超限 → DECLINED（CA00007），無 reserve
- [ ] Ledger 冪等驗證（重複 `txn.card.authorized` → duplicate idempotency_key → 靜默忽略）
- [ ] Points Service 冪等驗證（重複事件 → uk_pil_source_txn_id 已存在 → SKIP）
- [ ] T+1 結算後 available_balance 正確更新（PENDING → CONFIRMED）
- [ ] 點數批次 FIFO 順序驗證（最早到期先扣）

---
