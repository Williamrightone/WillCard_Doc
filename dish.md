# 信用卡交易功能開發順序

## Phase 1 — 卡片管理（Card Management）

**目標：** 建立虛擬卡的完整生命週期管理，為授權流程打底

- [ ] Card Service DB schema（Flyway migration）
  - `card`：cardId（Snowflake）、userId、pan_masked、card_type、status、expiry_date
  - `card_type_limit`：card_type、single_txn_limit、daily_limit
- [ ] 卡片 CRUD API（BFF + Card Service）
  - 申請虛擬卡（PENDING → ACTIVE）
  - 查詢卡片資訊
  - 凍結 / 解凍
- [ ] 卡片狀態機：`PENDING → ACTIVE → FROZEN → CANCELLED`
  - JVM Heap-only，禁止序列化與落地

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

**目標：** 建立 MCC → 回饋率映射，為授權流程中的點數回饋計算做準備

- [ ] Points Service DB schema（Flyway migration）
  - `reward_plan`：plan_id、mcc_code、reward_rate、effective_from、effective_to
  - `point_reward_batch`：batch_id、userId、source_txn_id、issued_amount、remaining_balance、status、expires_at
- [ ] Flyway DML：初始 MCC 回饋率種子資料（含 `DEFAULT` fallback）
- [ ] `ApplicationReadyEvent` 預載 `Map<mcc_code, rate>` 至記憶體；寫入時 cache invalidation
- [ ] 回饋率查詢 Feign API（供 Card Service 呼叫）

---

## Phase 4 — 匯率服務（FX Service）

**目標：** 提供外幣交易所需的無狀態匯率查詢與鎖定

- [ ] 匯率來源（開發環境 mock 固定值；正式接外部 API）
- [ ] `lockRate(currency)` → fxRateId（Redis TTL 10 分鐘）
- [ ] `getRate(fxRateId)` → 取回鎖定匯率
- [ ] 外幣計算邏輯驗證（twd_base、fx_fee 1.5%、回饋基數為 twd_base）

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
  - 從 KMS / 環境變數載入
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
- [ ] 由 PAN 識別會員
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

## NCCC 3DS 流程參考（機密，不得外流）

1. 持卡人確認購物，使用信用卡付款
2. 驗證封包(AReq)傳送國際組織 DS; DS 回覆 ACS Server 驗證的結果(ARes)
3. 國際組織將 AReq 封包加上 DS 欄位後轉給 ACS Server; 驗證結果(ARes)回覆 DS
4. 若驗證結果需要進行卡人身份確認，導轉到 ACS Server 進行 OTP 的驗證
5. ACS Server 產生取得 OTP 的身份驗證畫面
6. 持卡人由身份驗證畫面按下"取得動態密碼"按鈕
7. 通知銀行傳送 OTP 給持卡人
8. 銀行由 SMS Server 發送 OTP 密碼至持卡人手機
9. 持卡人輸入由簡訊收到的 OTP 密碼
10. ACS Server 將持卡人輸入的 OTP 密碼傳送給銀行驗證是否正確
11. 銀行回覆 OTP 密碼驗證結果
12. 若 OTP 驗證結果正確，ACS Server 產生 CAVV/AAV 後組成身份驗證的 RReq 封包，
RReq 轉送給 DS
13. 國際組織將 RReq 封包加上 DS 欄位後轉給 3DS Server
14. ACS Server 經由轉址通知商店身份認證結果完成
