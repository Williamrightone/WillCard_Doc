# 信用卡交易功能開發順序

## Phase 1 — 卡片管理（Card Management）

**目標：** 建立虛擬卡的完整生命週期管理，為授權流程打底

### 卡片種類（4 種）

| card_type | 中文名稱 | 國內回饋 | 國外回饋 | 跨國手續費 | 單筆上限 | 單日累計上限 |
|-----------|---------|---------|---------|-----------|---------|------------|
| CLASSIC | WillCard 經典卡 | 1% | 0% | 1.5% | NT$100,000 | NT$200,000 |
| OVERSEAS | WillCard 海外卡 | 0% | 5% | 1% | NT$100,000 | NT$200,000 |
| PREMIUM | WillCard 貴賓卡 | 2% | 5% | 1% | NT$200,000 | NT$500,000 |
| INFINITE | WillCard 無限卡 | 2% | 5% | 1% | 無上限 | 無上限 |

> **回饋率三層疊加邏輯（Phase 7 計算時使用）：**
> 1. `card_type_limit.domestic/overseas_reward_rate`：卡種基礎回饋（本 Phase 定義）
> 2. `reward_plan`（Phase 3）：MCC 類別加成，可綁定特定卡種（`card_type` nullable）或全卡種
> 3. `card_type_merchant_benefit`（本 Phase）：特約商家額外加成，真正疊加於前兩層之上
>
> `final_rate = base_rate + mcc_bonus + merchant_bonus`

---

### DB Schema（Flyway migration）

- [ ] **`card` 表**
  - `card_id` BIGINT — Snowflake ID，PK
  - `user_id` BIGINT — 持卡會員
  - `card_type` VARCHAR(20) — `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE`
  - `card_network` VARCHAR(10) — `VISA` / `MASTERCARD` / `JCB`
  - `status` VARCHAR(20) — 卡片狀態（見狀態機）
  - `pan_encrypted` VARCHAR(512) — **AES-256 加密**儲存完整卡號（PCI DSS）
  - `pan_masked` VARCHAR(20) — 遮罩格式 `****-****-****-1234`，用於顯示與 Kafka payload
  - `cardholder_name_zh` VARCHAR(512) — 持卡人中文姓名，**AES-256 加密**儲存
  - `cardholder_name_en` VARCHAR(512) — 持卡人英文姓名（卡面印刷用），**AES-256 加密**儲存
  - `expiry_date` VARCHAR(512) — **AES-256 加密**儲存，格式 `MMYY`；申請日 + 5 年（PCI DSS）
  - `activated_at` DATETIME(3) nullable — 卡片首次 ACTIVE 時間（稽核用）
  - `cancelled_at` DATETIME(3) nullable — 卡片 CANCELLED 時間（稽核用）
  - `created_at` / `updated_at` — 繼承 BaseTimeEntity
  - **Unique constraint**：`(user_id, card_type)` WHERE status != 'CANCELLED'（應用層驗證，每人每卡種限一張有效卡）

- [ ] **`card_type_limit` 表**（Flyway DML 種子資料）
  - `card_type` VARCHAR(20) — PK
  - `single_txn_limit` BIGINT — 單筆上限（分；NULL = 無上限）
  - `daily_limit` BIGINT — 單日累計上限（分；NULL = 無上限）
  - `domestic_reward_rate` DECIMAL(5,4) — 國內基礎回饋率（e.g., 0.0100）
  - `overseas_reward_rate` DECIMAL(5,4) — 國外基礎回饋率（e.g., 0.0500）
  - `fx_fee_rate` DECIMAL(5,4) — 跨國手續費率（e.g., 0.0100）
  - `created_at` / `updated_at`

  種子資料（Flyway DML）：

  | card_type | single_txn_limit | daily_limit | domestic | overseas | fx_fee |
  |-----------|-----------------|-------------|----------|----------|--------|
  | CLASSIC | 10,000,000 | 20,000,000 | 0.0100 | 0.0000 | 0.0150 |
  | OVERSEAS | 10,000,000 | 20,000,000 | 0.0000 | 0.0500 | 0.0100 |
  | PREMIUM | 20,000,000 | 50,000,000 | 0.0200 | 0.0500 | 0.0100 |
  | INFINITE | NULL | NULL | 0.0200 | 0.0500 | 0.0100 |

- [ ] **`card_type_merchant_benefit` 表**（特約優惠，Flyway DML 種子資料）
  - `benefit_id` BIGINT — Snowflake PK
  - `card_type` VARCHAR(20) — 適用卡種
  - `merchant_id` VARCHAR(50) nullable — 特定商家（null = 套用整個 MCC）
  - `mcc_code` VARCHAR(10) nullable — 指定消費類別
  - `bonus_rate` DECIMAL(5,4) — 加成回饋率（疊加於 base + mcc_bonus 之上）
  - `description_zh` VARCHAR(255) — 優惠說明（中文）
  - `description_en` VARCHAR(255) — 優惠說明（英文）
  - `effective_from` DATE — 生效日
  - `effective_to` DATE nullable — 到期日（null = 長期有效）
  - `created_at` / `updated_at`

- [ ] **每日交易累計追蹤**（供 Phase 7 daily_limit 查核）
  - Redis key：`card:daily:{cardId}:{yyyyMMdd}`，值為當日累計金額（分）
  - TTL：當日 23:59:59 到期（或固定 86400s）
  - 無限卡（INFINITE）跳過此檢查

---

### 卡片 CRUD API（BFF + Card Service）

- [ ] **申請虛擬卡**（`POST /api/v1/card/apply`）
  - 驗證：同一 user 該 card_type 不得已有 PENDING / ACTIVE / FROZEN 的卡
  - Mock 實作：依 `card_network` 使用對應 BIN prefix 產生合法 PAN（Luhn check）
  - PAN 以 AES-256 加密落地（`pan_encrypted`）；同步產生並儲存 `pan_masked`
  - **CVV**：申請時即時產生，僅在本次 Response 回傳一次，**絕不儲存**（PCI DSS）；之後無法再查詢
  - `expiry_date` = 申請日 + 5 年，AES-256 加密後落地
  - 初始狀態 `PENDING → ACTIVE`（本 MVP 直接核卡，略過審核流程）；寫入 `activated_at`
  - BFF spec：`spec/mobile-bff/card-apply.md`
  - biz spec：`spec/card-service/card-apply.md`

- [ ] **查詢卡片資訊**（`GET /api/v1/card/info`）
  - `pan_masked` 直接讀取回傳（已落地，不需解密）
  - 解密 `expiry_date` / `cardholder_name_zh` / `cardholder_name_en` 後組裝回傳
  - `pan_encrypted` 不回傳給 App（僅授權流程使用）
  - BFF spec：`spec/mobile-bff/card-info.md`
  - biz spec：`spec/card-service/card-info.md`

- [ ] **凍結卡片**（`POST /api/v1/card/freeze`）
  - 狀態轉移：`ACTIVE → FROZEN`
  - BFF spec：`spec/mobile-bff/card-freeze.md`
  - biz spec：`spec/card-service/card-freeze.md`

- [ ] **解凍卡片**（`POST /api/v1/card/unfreeze`）
  - 狀態轉移：`FROZEN → ACTIVE`
  - BFF spec：`spec/mobile-bff/card-unfreeze.md`
  - biz spec：`spec/card-service/card-unfreeze.md`

- [ ] **查詢特約優惠**（`GET /api/v1/card/benefits`）
  - 依 card_type 查詢目前有效的 `card_type_merchant_benefit`
  - BFF spec：`spec/mobile-bff/card-benefits.md`
  - biz spec：`spec/card-service/card-benefits.md`

---

### 卡片狀態機：`PENDING → ACTIVE → FROZEN → CANCELLED`

- 狀態落地到 DB `card.status` 欄位（允許 UPDATE）
- 合法轉移：PENDING→ACTIVE、ACTIVE→FROZEN、FROZEN→ACTIVE、ACTIVE→CANCELLED、FROZEN→CANCELLED
- ⚠️ **「JVM Heap-only，禁止序列化與落地」是針對解密後的明文卡片資料，不是狀態機本身**

---

### PCI DSS 合規實作

- [ ] **AES-256 加解密工具**：統一的加解密 Service，供 Card Service 使用；金鑰由環境變數 / KMS 注入，不寫入程式碼或 DB
- [ ] **PAN 遮罩顯示**：解密後以程式碼產生遮罩格式 `****-****-****-1234`；可評估使用 Jackson 自訂 Serializer（`@JsonSerialize`）或 Logback `MaskingPatternLayout` 統一處理
- [ ] **Log sanitization**：Logback filter，攔截 log 中的完整 PAN / CVV，強制替換為遮罩（`pan_encrypted` 在 DB 層，log 只可能印出解密後的值，需在此防守）
- [ ] **CVV 不儲存**：申請卡時即時產生、一次性回傳，之後任何場景均不可查詢；授權流程中 CVV 僅存在 JVM，完成後立即清除（PCI DSS）
- [ ] **Kafka event payload**：傳遞遮罩格式 `maskedPan`（runtime 從解密 PAN 產生），禁止 CVV 出現在任何 event

---

### ModelType 與錯誤碼（CardServiceException）

- [ ] ModelType：`CA`，錯誤碼格式 `CA00001`
  - `CARD_NOT_FOUND` CA00001
  - `CARD_NOT_ACTIVE` CA00002
  - `CARD_EXPIRED` CA00003
  - `CARD_FROZEN` CA00004
  - `CARD_CANCELLED` CA00005
  - `SINGLE_TXN_LIMIT_EXCEEDED` CA00006
  - `DAILY_LIMIT_EXCEEDED` CA00007
  - `CARD_TYPE_NOT_FOUND` CA00008
  - `CARD_ALREADY_EXISTS` CA00009 — 同卡種已有有效卡

---

## Phase 2 — 點數折抵偏好設定（Points Preference）

**目標：** 建立點數折抵偏好設定，以及 Reserve-Confirm-Release 核心方法

- [ ] Wallet Service DB schema（Flyway migration）
  - `wallet_account`：userId、available_balance、reserved_balance
  - `member_points_preference`：userId、points_first_enabled、max_points_per_txn
- [ ] 偏好設定 API（BFF + Wallet Service）
  - 查詢 / 更新偏好（points_first_enabled、max_points_per_txn）
- [ ] Reserve / Confirm / Release 核心方法
  - `reserve(userId, amount)` → reservationId
  - `confirmDeduct(reservationId)` → reserved_balance 扣除
  - `release(reservationId)` → 退回 available_balance

---

## Phase 3 — 回饋方案設定（Reward Plan）

**目標：** 建立 MCC → 回饋率映射，可綁定卡種，為授權流程中的點數回饋計算做準備

- [ ] Points Service DB schema（Flyway migration）
  - `reward_plan`：plan_id、`card_type`（nullable，null = 全卡種適用）、mcc_code、reward_rate、effective_from、effective_to
    - 查詢優先順序：`card_type + mcc` > `null + mcc` > `DEFAULT`
  - `point_reward_batch`：batch_id、userId、source_txn_id、issued_amount、remaining_balance、status、expires_at
- [ ] Flyway DML：初始 MCC 回饋率種子資料（含 `DEFAULT` fallback）
- [ ] `ApplicationReadyEvent` 預載 `Map<card_type+mcc_code, rate>` 至記憶體；寫入時 cache invalidation
- [ ] 回饋率查詢 Feign API（供 Card Service 呼叫，傳入 card_type + mcc_code）

---

## Phase 4 — 匯率服務（FX Service）

**目標：** 提供外幣交易所需的無狀態匯率查詢與鎖定

- [ ] 匯率來源（開發環境 mock 固定值；正式接外部 API）
- [ ] `lockRate(currency)` → fxRateId（Redis TTL 10 分鐘）
- [ ] `getRate(fxRateId)` → 取回鎖定匯率
- [ ] 外幣計算邏輯驗證（twd_base、fx_fee 依卡種 card_type_limit.fx_fee_rate、回饋基數為 twd_base）

---

## Phase 5 — AES-256 / combineKey 加解密（3DS Encryption）

**目標：** 完成卡片資料加解密機制，確保 PCI DSS 合規

- [ ] AES-256 GCM 加解密實作
  - `encryptedCard` 解密流程（以記憶體中的 combineKey 解密）
  - 明文卡片資料僅存 JVM Heap，不序列化、不落地
  - CVV 授權完成後立即丟棄
- [ ] PAN 遮罩格式驗證（`****-****-****-1234`）
- [ ] Mock 環境的 combineKey 固定值設定（供 Mock NCCC 加密使用）
- [ ] Log sanitization：確認 PAN / CVV 不出現於任何 log
- [ ] combineKey 基礎設施
  - `card_key_parts` 表：三個 sourceKey 各自 AES-256 加密儲存；master key 從 KMS / 環境變數載入
  - `ApplicationReadyEvent`：讀 `card_key_parts` → 解密三 part → 組合 combineKey，常駐 JVM Heap
  - `@PreDestroy`：JVM 正常關閉時清除 Heap 中的 combineKey

---

## Phase 6 — OTP 產生與 SMS 通知（Notification）

**目標：** 建立 OTP 生命週期，完成 SMS 下行通路

- [ ] Notification Service — OTP SMS 範本 + RabbitMQ Consumer
- [ ] Card Service — 6 位數 OTP 安全亂數產生
- [ ] Redis OTP session：`otp:{challengeRef}`（TTL 180s）
  - Payload：`{ txnDetails, pointsToUse, reservationId, fxRateId? }`
- [ ] 手機號碼遮罩記錄（log 僅顯示 `09xx****xx`）

---

## Phase 7 — Issuer Authorization API（Card Service 核心）

**目標：** 實作 NCCC 直接呼叫的兩支授權端點，這是整個交易流程的核心

### Phase 1：`POST /card/auth/authorize`

- [ ] 以記憶體 combineKey 解密 encryptedCard
- [ ] 由 PAN 識別會員（pan_hash 查詢）
- [ ] 驗證卡片：ACTIVE、未過期
- [ ] 檢查交易限額（single_txn_limit + 當日累計 daily_limit）
- [ ] 讀取點數偏好 → 計算 pointsToUse（min 三值規則）
- [ ] 驗證可用資金：wallet_balance - pointsToUse >= cardAmount
- [ ] `Wallet Service.reserve(userId, pointsToUse)`
- [ ] 外幣：`FX Service.lockRate()` → fxRateId
- [ ] 產生 OTP → 發送至 RabbitMQ（Notification Service 消費）
- [ ] 儲存 Redis pending auth（TTL 180s）
- [ ] Response：`CHALLENGE_REQUIRED + challengeRef + pointsPreview` / `DECLINED`

### Phase 2：`POST /card/auth/verify-challenge`

- [ ] Redis key 不存在 → `DECLINED + SESSION_EXPIRED`
- [ ] 驗證 OTP 值
- [ ] OTP 正確 → `Wallet Service.confirmDeduct(reservationId)` → 分配 txnId → Publish Kafka `txn.card.authorized` → 清除 Redis key
- [ ] OTP 錯誤 → 雙層計數器：
  - `otp:attempt:{challengeRef}`（per-session，TTL 180s，上限 3 次 → 授權作廢）
  - `otp:card:fail:{cardId}`（per-card，TTL 900s，上限 5 次 → 卡片鎖定 + Kafka `card.risk.otp-threshold-exceeded`）
- [ ] Response：`APPROVED` / `DECLINED + OTP_FAILED` / `DECLINED + CARD_LOCKED` / `DECLINED + SESSION_EXPIRED`

---

## Phase 8 — Mock NCCC Service（開發環境）

**目標：** 模擬 NCCC 對 Issuer API 的呼叫，讓 App 可以完整端對端測試

- [ ] `POST /mock-nccc/pay`
  - 接收 App 刷卡請求（cardId、amount、currency、merchantId）
  - 用 mock combineKey 加密卡片資料 → 組成 encryptedCard
  - Feign 呼叫 `POST /card/auth/authorize`
  - 回傳 `challengeRef + pointsPreview` 給 App
- [ ] `POST /mock-nccc/pay/confirm`
  - 接收 App 的 OTP（challengeRef + otp）
  - Feign 呼叫 `POST /card/auth/verify-challenge`
  - 回傳最終結果（`APPROVED` / `DECLINED`）

---

## Phase 9 — 交易 Saga（Transaction Orchestrator）

**目標：** 處理授權成功後的下游非同步流程，以及補償邏輯

- [ ] Kafka Consumer：`txn.card.authorized`
- [ ] 下游觸發（非同步，依序或並行）：
  - Ledger Service：寫入 AUTHORIZATION 複式分錄
  - Points Service：建立 `point_reward_batch`（PENDING，T+1 清算後 CONFIRMED）
  - Reconciliation：登記交易記錄
  - Notification Service：推播 + Email receipt
- [ ] 補償觸發：
  - Phase 1 資金不足 / 驗卡失敗 → `Wallet Service.release(reservationId)`
  - OTP 失敗 / Session 過期 → `Wallet Service.release(reservationId)`
  - TTL 到期惰性清理 → `Wallet Service.release(reservationId)`

---

## Phase 10 — 複式帳本（Ledger Service）

**目標：** 不可變帳本分錄，支援稽核與 T+1 清算

- [ ] `journal_entry` 表（Flyway migration；無 updated_at）
- [ ] Kafka Consumer：`txn.card.authorized`
- [ ] 三種情境分錄實作：
  - 台幣、無折抵（1001/2001/3001/4001/2002）
  - 台幣、點數折抵（多加 2002 → 2001 補貼特店）
  - 外幣（額外加 3002 外幣手續費）
- [ ] 冪等設計：`idempotency_key = {messageKey}_{entry_seq}`；duplicate key → ignore（不拋例外）
- [ ] SETTLEMENT 分錄（T+1 批次：DR 2001 / CR 1002）

---

## Phase 11 — BFF Card Pay API

**目標：** 對 App 暴露完整刷卡入口，串接 Mock NCCC

- [ ] `POST /api/v1/card/pay`（Nginx strip prefix → BFF `/card/pay`）
  - JWT 驗證 + X-User-Id header
  - 冪等控制（idempotency key）
  - Feign 呼叫 `POST /mock-nccc/pay`
  - Operation Log L1：Kafka topic `operation-log.card.pay`
- [ ] `POST /api/v1/card/pay/confirm`（Nginx strip → BFF `/card/pay/confirm`）
  - Feign 呼叫 `POST /mock-nccc/pay/confirm`
  - Operation Log L1：Kafka topic `operation-log.card.pay-confirm`
- [ ] Error mapping：Card Service 錯誤碼 → BFF 對外錯誤碼

---

## Phase 12 — 整合測試（End-to-End）

**目標：** 驗證完整交易流程的所有路徑

- [ ] 正常台幣刷卡（無折抵）：App → BFF → Mock NCCC → Card Service → Saga → Ledger
- [ ] 點數折抵（points_first_enabled = true）
- [ ] 外幣交易（USD，驗證 fx_fee 與回饋基數）
- [ ] OTP 失敗 → 重試 → 第 3 次授權作廢（per-session）
- [ ] 卡片鎖定（per-card 5 次 / 15 分鐘，Kafka risk event）
- [ ] Session 過期（Redis TTL 180s）
- [ ] 資金不足 DECLINED → Wallet reserve release 補償
- [ ] Kafka 冪等驗證（Ledger duplicate key 處理）
- [ ] Phase 2 收到已過期 challengeRef → SESSION_EXPIRED

---

---

