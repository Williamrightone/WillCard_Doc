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

**目標：** 對 App 暴露刷卡與 OTP 確認兩支入口，呼叫 Transaction Orchestrator

> **架構說明：** BFF 不直接呼叫 Mock NCCC，而是呼叫 Transaction Orchestrator，由 ORCH 統一協調後續的 FX 換算、Wallet Saga 與 Mock NCCC 呼叫。

- [ ] **`POST /api/v1/card/pay`**（Nginx strip → BFF `/card/pay`）
  - Request：`{ cardId, amount, currency, merchantId, usePoints: Boolean }`
  - JWT 驗證 + X-Member-Id header 注入
  - 冪等控制（BFF Redis `bff:pay:idem:{userId}:{idempotencyKey}`，TTL 60s）
  - Feign → `POST /txn-orch/card/pay`（Transaction Orchestrator）
  - Response：`{ challengeRef, pointsToUse, estimatedTwdAmount }`
  - Operation Log L1：Kafka `operation-log.card.pay`
  - spec: `spec/mobile-bff/card-pay.md` ✅

- [ ] **`POST /api/v1/card/pay/confirm`**（Nginx strip → BFF `/card/pay/confirm`）
  - Request：`{ challengeRef, otp }`
  - Feign → `POST /txn-orch/card/pay/confirm`（Transaction Orchestrator）
  - Response：`{ result: APPROVED | DECLINED, txnId?, reason?, remainingAttempts? }`
  - Operation Log L1：Kafka `operation-log.card.pay-confirm`
  - spec: `spec/mobile-bff/card-pay-confirm.md` ✅

---

## Phase 7 — Transaction Orchestrator + Mock NCCC（開發環境）

**目標：** ORCH 統一協調完整交易 Saga；Mock NCCC 模擬卡組織行為

> **ORCH 職責：** FX 換算、Wallet reserve/confirm/release、Saga 狀態管理、Kafka 發布。
> **Mock NCCC 職責：** 組裝 encryptedCard、呼叫 card-service auth/authorize 與 verify-challenge。
> **card-service 職責：** 驗卡、限額核查、OTP 生成與驗證——**不再呼叫 FX 或 Wallet Service**。

---

### Phase 7a — Transaction Orchestrator

#### ORCH Saga State（Redis，key: `saga:{challengeRef}`，TTL 180s）

```json
{
  "userId": 123,
  "cardId": 456,
  "amount": 10000,
  "currency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "merchantId": "MERCHANT_001",
  "pointsToUse": 5000,
  "reservationId": 789
}
```

> `reservationId`：若 pointsToUse = 0 則為 null。

#### UseCase Flow — `POST /txn-orch/card/pay`（Leg 1）

| Step | 說明 |
|------|------|
| 1 | 幣別判斷：`TWD` → `txnTwdBaseAmount = amount`；外幣 → Feign `POST /fx-service/convert` → `txnTwdBaseAmount` |
| 2 | `isOverseas = (currency != TWD)` |
| 3 | 若 `usePoints`：Feign `wallet-service GET /balance/{userId}` → `availableBalance`；`pointsToUse = min(availableBalance, txnTwdBaseAmount)`；否則 `pointsToUse = 0` |
| 4 | Feign → `POST /mock-nccc/pay`（`{ cardId, amount, currency, txnTwdBaseAmount, merchantId }`）→ 得到 `challengeRef` |
| 5 | 若 Step 4 成功且 `pointsToUse > 0`：Feign `wallet-service reserve(userId, pointsToUse)` → `reservationId` |
| 6 | 儲存 Saga State（key: `saga:{challengeRef}`，TTL 180s） |
| 7 | Response：`{ challengeRef, pointsToUse, estimatedTwdAmount: txnTwdBaseAmount }` |

> Step 4 失敗（驗卡失敗、限額超過）→ 直接回傳錯誤，無需 release（Step 5 尚未執行）。
> Step 5 失敗（wallet reserve 失敗）→ 回傳錯誤（OTP Session 存在但無 reservation；TTL 自然過期）。

#### UseCase Flow — `POST /txn-orch/card/pay/confirm`（Leg 2）

| Step | 說明 |
|------|------|
| 1 | 讀取 Saga State `saga:{challengeRef}`；不存在 → 回傳 `DECLINED + SESSION_EXPIRED` |
| 2 | Feign → `POST /mock-nccc/pay/confirm`（`{ challengeRef, otp }`）→ 得到 `{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }` |
| **result = APPROVED** | |
| 3 | 若 `reservationId != null`：Feign `wallet-service confirmDeduct(reservationId)` |
| 4 | Kafka Publish `txn.card.authorized`（從 Saga State + verify-challenge response 組裝，見 Kafka Payload） |
| 5 | DEL `saga:{challengeRef}` |
| 6 | Response：`{ result: APPROVED, txnId }` |
| **result = DECLINED + SESSION_VOIDED 或 CARD_LOCKED** | |
| 7 | 若 `reservationId != null`：Feign `wallet-service release(reservationId)` |
| 8 | DEL `saga:{challengeRef}` |
| 9 | Response：`{ result: DECLINED, reason }` |
| **result = DECLINED + OTP_FAILED** | |
| 10 | 不執行 release（Session 仍有效，Reservation 繼續持有） |
| 11 | Response：`{ result: DECLINED, reason: OTP_FAILED, remainingAttempts }` |

- spec: `spec/txn-orch/card-pay.md`（待補）
- spec: `spec/txn-orch/card-pay-confirm.md`（待補）

---

### Phase 7b — Mock NCCC Service（開發環境）

> **設計說明：** ORCH 主動呼叫 Mock NCCC。Mock NCCC 從 card-service 取得明文卡片資料、以固定 MOCK_COMBINE_KEY 組裝 encryptedCard，再呼叫 card-service auth/authorize。

#### `POST /mock-nccc/pay`

- Request（來自 ORCH）：`{ cardId, amount, currency, txnTwdBaseAmount, merchantId }`
- Feign → `GET /card-service/internal/cards/{cardId}/plain` → `{ pan, expiryMMYY, cvvItoken }`
- 以 `MOCK_COMBINE_KEY` AES-256-GCM 加密 `{ pan, expiryMMYY, cvvItoken }` → `encryptedCard`
- Feign → `POST /card-service/auth/authorize`（`{ encryptedCard, amount, currency, txnTwdBaseAmount, merchantId }`）
- 回傳 `{ challengeRef }` 給 ORCH
- spec: `spec/mock-nccc/pay.md`（待補）

#### `POST /mock-nccc/pay/confirm`

- Request（來自 ORCH）：`{ challengeRef, otp }`
- Feign → `POST /card-service/auth/verify-challenge`（`{ challengeRef, otp }`）
- 回傳 `{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }` 給 ORCH
- spec: `spec/mock-nccc/pay-confirm.md`（待補）

#### card-service 內部卡片查詢端點

- `GET /card-service/internal/cards/{cardId}/plain`（Nginx 封鎖外部流量）
- Response：`{ pan, expiryMMYY, cvvItoken }`（僅 dev/mock profile 啟用）
- spec: `spec/card-service/card-plain.md`（待補）

---

## Phase 8 — card-service auth/authorize（OTP 生成 + 限額核查）

**目標：** 驗卡、限額核查、生成 OTP，透過 OTP 發送策略（Telegram 實作，SMS stub）通知使用者

> **重要：** card-service 在此設計下**不呼叫 FX Service 也不呼叫 Wallet Service**。`txnTwdBaseAmount` 由 ORCH 計算後透過 MockNCCC 傳入請求。Wallet reserve/confirm/release 由 ORCH 統一處理。

> spec: `spec/card-service/card-auth-authorize.md` ✅
> spec: `spec/card-service/card-plain.md` ✅（Mock NCCC 用內部明文端點）
> spec: `spec/notify-service/otp-delivery.md` ✅（OTP 策略消費者）

### DB — member_otp_channel（uaa_db）

會員 OTP 發送頻道設定表，位於 auth-service 的 `uaa_db`。

| 欄位 | 型別 | 說明 |
|------|------|------|
| channel_id | BIGINT | PK，Snowflake ID |
| member_id | BIGINT | 所屬會員 |
| channel_type | ENUM('TELEGRAM','SMS') | 頻道類型 |
| contact_value | VARCHAR(255) | Telegram chat_id 或 E.164 手機號碼；不記錄 log |
| masked_value | VARCHAR(100) | Log 安全遮罩值 |
| is_primary | TINYINT(1) | 是否為主要頻道（每個 channel_type 至多一筆） |
| status | ENUM('ACTIVE','DISABLED') | 頻道狀態 |

> 詳見 `database/auth_db.md` — `member_otp_channel` 表。

### OTP 通知策略（notify-service）

| Adapter | 狀態 | 行為 |
|---------|------|------|
| `TelegramOtpAdapter` | **已實作** | 呼叫 Telegram Bot API，以 `chat_id` 發送 OTP 訊息 |
| `SmsOtpAdapter` | **僅 Stub** | 記錄 WARN log，不實際發送 |

notify-service 消費 RabbitMQ `notification.otp.send` 訊息（`{ userId, otp }`），查詢 auth-service internal 取得會員頻道，依策略發送。

### Redis OTP Session（card-service 所有）

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
    "otpValue": "123456"
  }
  ```

  > `txnTwdBaseAmount`：由 ORCH 計算並傳入 authorize 請求，card-service 儲存至 OTP Session 供 verify-challenge 使用（daily limit 累計）。
  > `pointsToUse` / `reservationId` 已移至 ORCH Saga State，不存於 OTP Session。

### UseCase Flow（`POST /card-service/auth/authorize`）

| Step | 說明 |
|------|------|
| 1 | 以 `CombineKeyHolder` 取得 combineKey；null → 拋 CA00013 HTTP 503 |
| 2 | AES-256-GCM 解密 `encryptedCard` → `{ pan, expiryMMYY, cvvItoken }`；try-finally 確保清零 pan |
| 3 | HMAC-SHA256(pan) → pan_hash，查詢 DB；找不到 → CA00001 |
| 4 | 比對 cvvItoken：received == card.cvv_itoken；不符 → CA00014 |
| 5 | 驗卡：status = ACTIVE（CA00002 / CA00004）、expiry 未過期（CA00003） |
| 6 | 單筆限額：txnTwdBaseAmount ≤ single_txn_limit（INFINITE 跳過）→ CA00006 |
| 7 | 當日累計：Redis GET `card:daily:{cardId}:{yyyyMMdd}` + txnTwdBaseAmount ≤ daily_limit → CA00007 |
| 8 | 產生 challengeRef（UUID）+ 6 位 OTP（SecureRandom） |
| 9 | 發布 RabbitMQ exchange `notification` routing key `otp.send`：`{ userId, otp }` → notify-service 依策略發送 |
| 10 | 儲存 Redis OTP Session（TTL 180s） |
| 11 | 清零 PAN 明文位元組（try-finally 保證執行） |
| 12 | Response：`{ challengeRef }` |

---

## Phase 9 — card-service auth/verify-challenge（OTP 驗證）

**目標：** 驗證 OTP、更新每日累計限額、分配 txnId；Wallet confirm 與 `txn.card.authorized` Kafka 由 ORCH 執行

> **重要：** verify-challenge **不呼叫** wallet-service，也**不發布** `txn.card.authorized`。這兩個動作已移至 ORCH Leg 2 flow（Phase 7a Step 3–4）。
> 唯一的 Kafka 事件為 `card.risk.otp-threshold-exceeded`（風控事件，僅卡片鎖定時觸發）。

> spec: `spec/card-service/card-auth-verify-challenge.md` ✅

### 關鍵設計說明

- **cardType / maskedPan** 不在 OTP Session，APPROVED 路徑需額外 DB READ（Step 5）
- **兩個計數器先遞增，再判斷條件**：`otp:card:fail:{cardId}` 與 `otp:attempt:{challengeRef}` 在任何條件判斷前先 INCR
- **嚴重度排序**：卡片鎖定（≥ 5 次 / 15 分鐘）優先於 Session 作廢（≥ 3 次 / Session）

### UseCase Flow（`POST /card-service/auth/verify-challenge`）

| Step | 說明 |
|------|------|
| 1 | Redis GET `otp:{challengeRef}`；不存在 → Response `{ result: DECLINED, reason: SESSION_EXPIRED }` |
| 2 | 比對 OTP 值（session.otpValue == 輸入值） |
| **OTP 正確** | |
| 3 | INCRBY `card:daily:{cardId}:{yyyyMMdd}` txnTwdBaseAmount；新 key → TTL 至當日 23:59:59 Asia/Taipei |
| 4 | 分配 txnId（Snowflake） |
| 5 | DB READ：SELECT card_type, pan_masked FROM card WHERE card_id = session.cardId |
| 6 | DEL `otp:{challengeRef}` |
| 7 | Response：`{ result: APPROVED, txnId, cardType, maskedPan }` |
| **OTP 錯誤** | |
| 8 | INCR `otp:card:fail:{cardId}`（TTL 900s，新 key 時設定）→ cardFailCount |
| 9 | INCR `otp:attempt:{challengeRef}`（TTL 180s，新 key 時設定）→ attemptCount |
| 10 | 若 cardFailCount ≥ 5 → DB WRITE FROZEN + DEL session + Kafka `card.risk.otp-threshold-exceeded` + Response CARD_LOCKED |
| 11 | 若 attemptCount ≥ 3 → DEL session + Response SESSION_VOIDED |
| 12 | Response：`{ result: DECLINED, reason: OTP_FAILED, remainingAttempts: 3 - attemptCount }` |

> **ORCH 負責** wallet release（SESSION_VOIDED / CARD_LOCKED 時）與 wallet confirmDeduct / Kafka txn.card.authorized publish（APPROVED 時）。

### Kafka Payload — `txn.card.authorized`（由 ORCH 發布）

```json
{
  "txnId": "Snowflake ID（來自 verify-challenge response）",
  "userId": "來自 Saga State",
  "cardId": "來自 Saga State",
  "cardType": "來自 verify-challenge response",
  "maskedPan": "來自 verify-challenge response",
  "merchantId": "來自 Saga State",
  "txnAmount": "來自 Saga State",
  "txnCurrency": "來自 Saga State",
  "txnTwdBaseAmount": "來自 Saga State",
  "isOverseas": "來自 Saga State",
  "pointsToUse": "來自 Saga State",
  "txnTimestamp": "ORCH 發布時間（ISO-8601）"
}
```

---

## Phase 10 — 交易後非同步處理 + 帳本（Post-Auth Async + Ledger）

**目標：** 處理 `txn.card.authorized` 事件的所有下游消費

> **架構說明：** Points Service、Ledger Service、Notification Service 各自獨立訂閱同一 Kafka Topic（fan-out），互不依賴。

### Wallet Service 補齊（Phase 2 缺口，Phase 10 補寫）

> spec: `spec/wallet-service/reserve.md` ✅
> spec: `spec/wallet-service/confirm-deduct.md` ✅（含 release 端點）

### Points Service（Phase 3 設計，Phase 10 實作）

> spec: `spec/points-service/txn-authorized-consumer.md` ✅

- Kafka Consumer：`txn.card.authorized`（冪等：`uk_pil_source_txn_id`）
- `reward_plan` 表（points_db V3.0.1 / V3.0.2）提供基礎費率（與 card_type_limit 同步，不跨服務查詢）
- 計算 `floor(txnTwdBaseAmount × rate)` → 呼叫 wallet-service credit API → `point_reward_batch`（PENDING）
- 回饋以完整 txnTwdBaseAmount 計算（不扣除 pointsToUse）
- 到期日：txnTimestamp 所在月份一年後同月最後一刻

### Ledger Service

> spec: `spec/ledger-service/txn-authorized-consumer.md` ✅
> database: `database/ledger_db.md` ✅（journal_entry + fx_rate_snapshot）

- Kafka Consumer：`txn.card.authorized`（冪等：`{txnId}_{entrySeq}` unique key）
- 四種情境分錄（台幣/外幣 × 有無點數折抵）：
  - **情境 A** 台幣、無折抵：DR 1001 / CR 2001
  - **情境 B** 台幣、有折抵：DR 1001 / CR 2001 + CR 4001 / CR 1001（對沖）
  - **情境 C** 外幣、無折抵：DR 1001 / CR 2001(net) + CR 3001(fxFee)
  - **情境 D** 外幣、有折抵：情境 C + CR 4001 / CR 1001（對沖）
- `fx_rate_snapshot` 表（ledger_db V6.0.1）提供外幣手續費率（與 card_type_limit 同步）
- [ ] SETTLEMENT 分錄（T+1 排程）：DR 2001 / CR 1002（待後續 Phase）

### Notification Service — 交易收據

- [ ] Kafka Consumer：`txn.card.authorized` → 推播 + Email 消費明細（規格待補）
- Queue：`notification.txn.receipt`（內部轉發）

---

## Phase 11 — 整合測試（End-to-End）

**目標：** 驗證完整交易流程的所有路徑

- [ ] 正常台幣刷卡（usePoints = false）：App → BFF → ORCH → MockNCCC → card-service → ORCH Kafka → Ledger / Notification
- [ ] 台幣刷卡 + 點數全額折抵（available_balance >= txnAmount）
- [ ] 台幣刷卡 + 點數部分折抵（available_balance < txnAmount）
- [ ] 外幣交易（USD）：驗證 ORCH FX convert、txnTwdBaseAmount、fx_fee 分錄
- [ ] OTP 失敗 → 重試 → 第 3 次授權作廢（per-session）→ ORCH wallet release
- [ ] 卡片鎖定（per-card ≥ 5 次 / 15 分鐘）→ Kafka `card.risk.otp-threshold-exceeded` + ORCH wallet release
- [ ] Session 過期（Redis TTL 180s 到期 → SESSION_EXPIRED）
- [ ] 單筆限額超過 → DECLINED（CA00006），無 reserve
- [ ] 當日累計超限 → DECLINED（CA00007），無 reserve
- [ ] Ledger 冪等驗證（重複 `txn.card.authorized` → duplicate idempotency_key → 靜默忽略）
- [ ] Points Service 冪等驗證（重複事件 → uk_pil_source_txn_id 已存在 → SKIP）
- [ ] T+1 結算後 available_balance 正確更新（PENDING → CONFIRMED）
- [ ] 點數批次 FIFO 順序驗證（最早到期先扣）

---
