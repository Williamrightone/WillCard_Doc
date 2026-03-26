# 卡片授權 Phase 2（OTP 驗證）— Card Service Biz Spec

## 概述

發卡機構授權 API — Phase 2。由 NCCC（正式環境）或 Mock NCCC（開發環境）於持卡人送出
OTP 後直接呼叫。從 Redis 載入 pending auth 工作階段、驗證 OTP，核准時透過 Wallet Service
確認點數扣除並發布 `txn.card.authorized` Kafka 事件，觸發下游 Saga；
失敗時執行雙層計數器邏輯，超過門檻則釋放點數準備金（補償）並視需要鎖定卡片。

---

## API 定義 — card-service

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/auth/verify-challenge` |
| 實際接收路徑 | `/card/auth/verify-challenge` |
| 呼叫方 | NCCC（正式環境 mTLS）/ Mock NCCC Service（開發環境 Feign） |

### Request（CardAuthVerifyChallengeRq — card-contract）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| challengeRef | String | ✅ | @NotBlank | Phase 1 回傳的 OTP 工作階段參考碼 |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 持卡人提交的 6 位數 OTP |

### Response（CardAuthVerifyChallengeRs — card-contract）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| status | String | `APPROVED` / `DECLINED` | |
| authCode | String | 授權碼（以 Snowflake 為基礎產生） | `APPROVED` 時存在 |
| txnId | String | Snowflake 交易 ID | `APPROVED` 時存在 |
| pointsUsed | Integer | 實際扣除的點數 | `APPROVED` 時存在 |
| cardAmount | BigDecimal | 最終刷卡金額（TWD） | `APPROVED` 時存在 |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED` | `DECLINED` 時存在 |
| attemptsRemaining | Integer | 本次工作階段剩餘嘗試次數 | `declineReason = OTP_FAILED` 且工作階段仍有效時存在 |

---

## Service UseCase Flow

**UseCase：** `CardAuthVerifyChallengeUseCase.verify(CardAuthVerifyChallengeRq)`

| Step | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[REDIS READ]` | 載入 `otp:{challengeRef}` → pending auth payload；key 不存在 → `DECLINED + SESSION_EXPIRED` | `PendingAuthStore.load()` |
| 2 | `[DOMAIN]` | 比對提交的 `otp` 與儲存的 OTP 值 | `OtpService.verify()` |
| **— OTP 正確路徑 —** | | | |
| 3a | `[FEIGN]` | 呼叫 Wallet Service `confirmDeduct(reservationId)` → 確認扣除點數 → [spec/wallet-service/confirm-deduct.md](../wallet-service/confirm-deduct.md) | — |
| 4a | `[DOMAIN]` | 分配 `txnId`（Snowflake）；產生 `authCode` | `SnowflakeIdGenerator.next()` |
| 5a | `[REDIS WRITE]` | 以 `cardAmount`（分）累加 `card:daily:{cardId}:{yyyyMMdd}`；若 key 為新建則設 TTL 至當日結束 | `DailyLimitService.increment()` |
| 6a | `[KAFKA]` | 發布 `txn.card.authorized`（payload：`txnId`、`messageKey`、`cardId`、`userId`、`amountTwd`、`fxFee?`、`fxRate?`、`currency`、`merchantId`、`merchantMCC`、`pointsUsed`、`cardAmount`、`maskedPan`） | — |
| 7a | `[REDIS WRITE]` | 刪除 `otp:{challengeRef}` | `PendingAuthStore.clear()` |
| 8a | `[RETURN]` | 回傳 `APPROVED` + `authCode`、`txnId`、`pointsUsed`、`cardAmount` | — |
| **— OTP 錯誤路徑 —** | | | |
| 3b | `[REDIS WRITE]` | 遞增 `otp:attempt:{challengeRef}`（TTL 繼承工作階段；上限 3 次） | `OtpAttemptTracker.increment()` |
| 4b | `[REDIS WRITE]` | 遞增 `otp:card:fail:{cardId}`（TTL 900s；上限 5 次） | `OtpAttemptTracker.incrementCardFail()` |
| 5b | `[DOMAIN]` | 檢查每工作階段門檻：`otp:attempt:{challengeRef} >= 3` → 本次授權作廢 | `OtpAttemptTracker.isSessionExhausted()` |
| 6b（工作階段耗盡） | `[FEIGN]` | 補償：呼叫 Wallet Service `release(reservationId)` → [spec/wallet-service/release.md](../wallet-service/release.md) | — |
| 7b（工作階段耗盡） | `[REDIS WRITE]` | 刪除 `otp:{challengeRef}`；刪除 `otp:attempt:{challengeRef}` | — |
| 8b（工作階段耗盡） | `[RETURN]` | 回傳 `DECLINED + OTP_FAILED`（attemptsRemaining = 0） | — |
| 5c | `[DOMAIN]` | 檢查每張卡門檻：`otp:card:fail:{cardId} >= 5` → 暫時鎖卡 | `OtpAttemptTracker.isCardLocked()` |
| 6c（卡片鎖定） | `[DB WRITE]` | 更新 `card.status = LOCKED`（風控暫時鎖定） | `CardStatusService.lock()` |
| 7c（卡片鎖定） | `[FEIGN]` | 補償：呼叫 Wallet Service `release(reservationId)` | — |
| 8c（卡片鎖定） | `[REDIS WRITE]` | 刪除 `otp:{challengeRef}` | — |
| 9c（卡片鎖定） | `[KAFKA]` | 發布 `card.risk.otp-threshold-exceeded`（payload：`cardId`、`userId`、`failCount`、`windowSeconds=900`） | — |
| 10c（卡片鎖定） | `[RETURN]` | 回傳 `DECLINED + CARD_LOCKED` | — |
| 5d（一般失敗） | `[RETURN]` | 回傳 `DECLINED + OTP_FAILED` + `attemptsRemaining = 3 - attempt_count` | — |

> **TTL 惰性補償：** 若 `otp:{challengeRef}` 在 Phase 2 呼叫前自然過期（TTL 180s），Wallet Service 準備金由排程清理任務或下次授權嘗試時觸發 release。準備金 TTL 與 OTP TTL 一致（均為 180s）。

---

## 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `card` | READ / WRITE | 讀取 `user_id`、`card_type`；卡片鎖定時更新 `status = LOCKED` |

### Redis

| Key 模式 | 操作 | TTL | 說明 |
|----------|------|-----|------|
| `otp:{challengeRef}` | READ / DELETE | 180s（authorize 寫入） | Pending auth 工作階段 |
| `otp:attempt:{challengeRef}` | READ / WRITE | 180s | 每工作階段 OTP 失敗計數器（上限 3 次） |
| `otp:card:fail:{cardId}` | READ / WRITE | 900s | 每張卡 15 分鐘內 OTP 失敗計數器（上限 5 次） |
| `card:daily:{cardId}:{yyyyMMdd}` | WRITE | 當日結束 | APPROVED 時累加今日交易金額 |

### 資料表 Schema

#### card（相關欄位）

| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| card_id | BIGINT | NO | PK，Snowflake |
| user_id | BIGINT | NO | 持卡會員 |
| card_type | VARCHAR(20) | NO | 卡種 |
| status | VARCHAR(20) | NO | 卡片鎖定時更新為 `LOCKED` |
| updated_at | DATETIME(3) | NO | 狀態變更時更新 |

---

## 發布的 Kafka 事件

| Topic | 觸發時機 | 關鍵 Payload 欄位 |
|-------|---------|-----------------|
| `txn.card.authorized` | OTP 核准 | `txnId`、`messageKey`（Ledger 冪等鍵）、`cardId`、`userId`、`amountTwd`、`fxFee`、`fxRate`、`currency`、`merchantId`、`merchantMCC`、`pointsUsed`、`cardAmount`、`maskedPan` |
| `card.risk.otp-threshold-exceeded` | 每張卡門檻達到 | `cardId`、`userId`、`failCount`、`windowSeconds` |

> **PCI DSS：** Kafka event payload 中的 `maskedPan` 必須使用遮罩格式 `****-****-****-1234`。CVV 不得出現於任何 event payload。

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 422 | `SESSION_EXPIRED` | OTP 工作階段不存在或已過期 | `otp:{challengeRef}` key 不存在 |
| 422 | `OTP_FAILED` | OTP 驗證失敗 | OTP 不符；工作階段仍有效 |
| 422 | `CARD_LOCKED` | 卡片因連續失敗暫時鎖定 | 每張卡門檻（15 分鐘內 5 次）超過 |
| 500 | `INTERNAL_ERROR` | 未預期的系統錯誤 | 未處理例外 |
