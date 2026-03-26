# 交易流程規範

本章定義 WillCard 信用卡交易的業務規則、流程設計與撰寫慣例。所有涉及刷卡交易的 spec 文件（`spec/mobile-bff/card-pay*`、`spec/card-service/`、`spec/wallet-service/`、`spec/points-service/`、`spec/ledger-service/`）均須遵循本章規定。

---

## 1. WillCard 作為發卡機構（Issuer）

WillCard 發行以會員電子錢包餘額為基礎的虛擬 Visa/Mastercard 信用卡。在台灣支付生態系中，**NCCC（聯合信用卡處理中心 / 財金公司）** 扮演卡組織與清算中心的角色。

**WillCard 是 Issuer。授權請求從外部流入——NCCC 呼叫 WillCard，不是 WillCard 主動打 NCCC。**

```
[正式環境 — 外部流入]
持卡人在特店使用虛擬卡
  → 特店收銀機 / 結帳頁
  → 收單銀行
  → NCCC
  → WillCard Card Service（Issuer Authorization API）← 在此被呼叫
       ↓ 授權成功（非同步）
  內部 Saga：帳本 · 點數 · 通知

[開發環境 — Mock NCCC]
App → BFF → Mock NCCC Service → WillCard Card Service（走相同授權路徑）
```

**關鍵識別符：**

| 欄位 | 產生方 | 生命週期 | 用途 |
|------|--------|---------|------|
| `challengeRef` | Card Service（Phase 1）| 授權 session | OTP 綁定；串接 Phase 1 與 Phase 2 |
| `txnId` | Card Service（Phase 2，授權成功時）| 交易記錄 | 帳本、Kafka event、對帳 |

---

## 2. Issuer Authorization API（Card Service）

Card Service 對外暴露兩個端點，供 **NCCC 直接呼叫**，不透過 BFF 路由。

| 端點 | 呼叫方 | 用途 |
|------|--------|------|
| `POST /card/auth/authorize` | NCCC | 初始授權請求——驗卡、檢查餘額、觸發 3DS OTP |
| `POST /card/auth/verify-challenge` | NCCC | OTP 驗證——最終核准 / 拒絕 |

安全性：正式環境使用 NCCC 用戶端憑證（mutual TLS）；開發環境 Mock NCCC 走內網。

### 2.1 Phase 1 — authorize

```
NCCC → POST /card/auth/authorize
  Request:
    encryptedCard     — NCCC 以 combineKey 加密的持卡人資料
    amount            — 交易金額
    currency          — TWD / USD / ...
    merchantId
    merchantMCC       — 用於回饋率查詢

  Card Service 執行步驟：
    1. 以記憶體中的 combineKey 解密 encryptedCard
    2. 由 PAN 識別會員
    3. 驗證卡片：ACTIVE、未過期
    4. 檢查交易限額（單筆 + 當日累計）
    5. 讀取會員點數偏好 → 計算 pointsToUse（見 §4）
    6. 驗證可用資金：wallet_balance - pointsToUse >= cardAmount
    7. 預留點數：Wallet Service.reserve(userId, pointsToUse)
    8. 若外幣：FX Service.lockRate() → 取得 fxRateId
    9. 生成 6 位數 OTP → SMS 發送至會員註冊手機
   10. 儲存 pending auth 至 Redis：otp:{challengeRef}（TTL 180s）
       Payload: { txnDetails, pointsToUse, reservationId, fxRateId? }

  Response:
    CHALLENGE_REQUIRED  + challengeRef + pointsPreview
    DECLINED            + declineReason
```

### 2.2 Phase 2 — verify-challenge

```
NCCC → POST /card/auth/verify-challenge
  Request:
    challengeRef
    otp               — 持卡人輸入的驗證碼

  Card Service 執行步驟：
    1. 從 Redis 讀取 otp:{challengeRef} 的 pending auth 狀態
       → key 不存在（session 過期 / 已作廢）：直接回傳 DECLINED + SESSION_EXPIRED
    2. 驗證 OTP 值
    3a. OTP 正確：
        → Wallet Service.confirmDeduct(reservationId)
        → 分配 txnId（Snowflake）
        → Publish txn.card.authorized Kafka event（非同步 Saga 啟動）
        → 清除 Redis key
    3b. OTP 錯誤：
        → 累加 otp:attempt:{challengeRef}（per-session）
        → 累加 otp:card:fail:{cardId}（per-card，TTL 900s）
        → 檢查閾值（見 §2.3）

  Response（OTP 正確）：
    APPROVED  + authCode + txnId + pointsUsed + cardAmount
  Response（OTP 錯誤）：
    DECLINED  + OTP_FAILED + attemptsRemaining
  Response（卡片已鎖定）：
    DECLINED  + CARD_LOCKED
  Response（session 過期）：
    DECLINED  + SESSION_EXPIRED
```

### 2.3 OTP 失敗閾值

兩層獨立機制並存，分別應對「誤輸入重試」與「盜卡試探」。

**第一層 — Per-challengeRef（session 層級）**

| 項目 | 值 |
|------|---|
| Redis Key | `otp:attempt:{challengeRef}`（TTL 180s）|
| 最大重試次數 | 3 次 |
| 達到上限 | 本次授權作廢；卡片狀態不受影響 |

**第二層 — Per-card 滑動視窗（詐欺偵測）**

| 項目 | 值 |
|------|---|
| Redis Key | `otp:card:fail:{cardId}`（TTL 900s / 15 分鐘）|
| 觸發閾值 | **15 分鐘內跨 challengeRef 累計失敗 5 次** |
| 達到閾值 | 卡片暫時鎖定 + Publish `card.risk.otp-threshold-exceeded` → 風控系統 |
| 鎖定時長 | 由風控團隊定義 |

每次失敗**同時**累加兩層計數器。

### 2.4 卡片資料規則

詳見 `guideline/6-pcidss-zh-tw.md` §2.2。

- 解密後的明文卡片資料**僅存 JVM Heap**，不序列化、不落地
- CVV 於授權完成後立即丟棄
- PAN 僅以遮罩格式儲存：`****-****-****-1234`

---

## 3. Mock NCCC（開發與測試）

Mock NCCC 是**僅用於開發環境**的服務，模擬 NCCC 對 Issuer Authorization API 的呼叫，讓整個交易流程可不依賴真實 NCCC 進行端對端測試。

### 3.1 架構

```
App
  │  POST /api/v1/card/pay（透過 BFF）
  ↓
Mock NCCC Service
  ├─ 呼叫 Card Service: POST /card/auth/authorize
  │     ← CHALLENGE_REQUIRED + challengeRef + pointsPreview
  └─ 回傳 challengeRef + pointsPreview 給 App

App 顯示點數折抵預覽畫面（如 LinePay 樣式）與 OTP 輸入框
  │  POST /api/v1/card/pay/confirm（透過 BFF）
  ↓
Mock NCCC Service
  └─ 呼叫 Card Service: POST /card/auth/verify-challenge
        ← APPROVED / DECLINED

Mock NCCC 回傳最終結果給 App
```

### 3.2 API 定義

**POST /mock-nccc/pay — 發起付款**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `cardId` | String | ✅ | 會員卡片識別碼 |
| `amount` | BigDecimal | ✅ | 交易金額 |
| `currency` | String | ✅ | `TWD` / `USD` / … |
| `merchantId` | String | ✅ | 特店識別碼 |

Response：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `challengeRef` | String | 需要 3DS 驗證時回傳 |
| `status` | String | `CHALLENGE_REQUIRED` / `APPROVED` / `DECLINED` |
| `pointsPreview.pointsToUse` | Integer | 由伺服器端偏好計算 |
| `pointsPreview.cardAmount` | BigDecimal | 刷卡扣款金額 |

**POST /mock-nccc/pay/confirm — 提交 OTP**

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `challengeRef` | String | ✅ | 來自 pay 回應 |
| `otp` | String | ✅ | 6 位數 OTP |

Response：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `authCode` | String | 授權通過時回傳 |
| `txnId` | String | 交易 Snowflake ID |
| `status` | String | `APPROVED` / `DECLINED` |
| `declineReason` | String | 拒絕時回傳 |

### 3.3 Points Preview 顯示邏輯

Card Service 在 Phase 1 依據會員偏好計算 `pointsToUse`，附加在回應中。Mock NCCC 將此資訊作為 `pointsPreview` 回傳給 App，App 在使用者輸入 OTP 前顯示折抵摘要（如 LinePay 樣式）。

> `pointsToUse` 由伺服器端決定，App 顯示僅供參考——實際點數在 Phase 1 已完成預留，Phase 2 不得覆蓋。

---

## 4. 點數折抵規則（偏好驅動）

### 4.1 偏好設定

點數優先折抵為**每位會員的個人設定**，由 Wallet Service 管理。

**Table: `member_points_preference`（Wallet Service DB）**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `user_id` | BIGINT PK | |
| `points_first_enabled` | BOOLEAN | 交易時自動套用點數折抵 |
| `max_points_per_txn` | INT | 單筆折抵上限（NULL = 無上限）|
| `updated_at` | DATETIME | |

### 4.2 授權時的自動計算

Card Service 在 Phase 1 讀取偏好：

```
if points_first_enabled:
  pointsToUse = min(
    available_points_balance,
    amount_in_twd,
    max_points_per_txn ?? ∞
  )
else:
  pointsToUse = 0

cardAmount = amount_in_twd - pointsToUse
```

### 4.3 換算規則

| 規則 | 值 |
|------|---|
| 換算比例 | **1 點 = NT$1**（1:1）|
| 最小折抵 | 1 點 |
| 最大折抵 | 全額（100% offset）|
| 交易記錄 | `points_used` 與 `card_amount` 均須記錄 |

### 4.4 Reserve-Confirm-Release

```
Phase 1（authorize）：
  Wallet Service.reserve(userId, pointsToUse)
  → available_balance -= pointsToUse
  → reserved_balance  += pointsToUse
  Reserve TTL = 180s（與 OTP TTL 對齊，惰性清理）

Phase 2 — APPROVED：
  Wallet Service.confirmDeduct(reservationId)
  → reserved_balance -= pointsToUse

補償（OTP 失敗 / DECLINED / TTL 到期）：
  Wallet Service.release(reservationId)
  → reserved_balance -= pointsToUse
  → available_balance += pointsToUse
```

---

## 5. Saga 補償規則

| Saga 步驟 | 補償動作 | 觸發條件 |
|-----------|---------|---------|
| 點數 reserve | `release(reservationId)` | Phase 1 資金不足 / 驗卡失敗、OTP 失敗、Session 過期、TTL 到期（惰性）|
| 匯率鎖定 | FX Rate TTL 自動到期 | 無需主動補償 |
| 帳本分錄 | 寫入 REVERSAL 反向分錄（不得刪除）| 退款流程（另立 spec）|

帳本分錄為**不可變**。帳本層的補償一律以新的反向分錄處理，禁止 UPDATE 或 DELETE。

---

## 5. 點數回饋計算規則

### 5.1 回饋基數與計算方式

- 回饋僅基於**實際刷卡金額**（points_used 部分不計算）
- 計算結果**無條件捨去（floor）**，不四捨五入
- 外幣交易以**換算後台幣金額**為計算基礎

```
回饋點數 = floor(card_amount_twd × reward_rate)

範例：card_amount = 60, reward_rate = 2%
  60 × 0.02 = 1.2 → floor → 1 點
```

### 5.2 回饋方案（MCC 對應）

回饋率儲存於 Points Service DB 的 `reward_plan` 表，後台可動態設定。管理 UI 為後續功能，但資料表需從第一天開始建立。

**Cache 載入策略：** Points Service 在啟動時（`ApplicationReadyEvent`）將所有有效回饋方案載入記憶體 `Map<String, BigDecimal>`（以 `mcc_code` 為 key）。Map 僅在回饋方案新增或更新時刷新（寫入時 cache invalidation），讀取路徑不觸及 DB。

**Table: `reward_plan`**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `plan_id` | BIGINT PK | Snowflake |
| `mcc_code` | VARCHAR(10) | MCC 代碼；`DEFAULT` 為全類別預設 |
| `reward_rate` | DECIMAL(5,4) | 例：`0.0200` 表示 2% |
| `description` | VARCHAR(100) | 說明標籤 |
| `effective_from` | DATE | 生效日期 |
| `effective_to` | DATE | 到期日期（null = 永久有效）|
| `created_at` | DATETIME | |

初始 MCC 回饋率透過 Flyway DML 寫入。若找不到 MCC 對應設定，回退至 `mcc_code = 'DEFAULT'`。

### 5.3 回饋點數 Batch 設計

每次回饋發放產生一筆 `point_reward_batch` 記錄，每個 Batch 獨立追蹤剩餘餘額與到期時間，支援 FIFO 扣抵與精準到期管理。

**Table: `point_reward_batch`**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `batch_id` | BIGINT PK | Snowflake |
| `user_id` | BIGINT | |
| `source_txn_id` | BIGINT | 來源交易 |
| `issued_amount` | INT | 發放點數 |
| `remaining_balance` | INT | 剩餘可用（兌換時遞減）|
| `status` | VARCHAR | `PENDING` / `CONFIRMED` / `EXPIRED` / `CANCELLED` |
| `expires_at` | DATETIME | `issued_at + 1 年` |
| `created_at` | DATETIME | |

**狀態流轉：**

```
PENDING   →  CONFIRMED   （T+1 清算成功）
PENDING   →  CANCELLED   （清算前退款）
CONFIRMED →  EXPIRED     （到期日到達）
CONFIRMED →  CANCELLED   （清算後退款）
```

---

## 6. 外幣交易規則

| 規則 | 值 |
|------|---|
| 台幣交易 | **不收手續費** |
| 外幣手續費 | `twd_base` 的 **1.5%**，加收於換算金額之上 |
| 匯率鎖定 | Step 1 呼叫 FX Service `lockRate()`，回傳 `fxRateId`（有效期 10 分鐘）|
| 帳本記帳幣別 | 一律以台幣記帳，同時記錄原幣金額與使用匯率 |
| 回饋計算基數 | **`twd_base`**（純匯率換算），不含 1.5% 手續費 |
| 特店手續費計算基礎 | **`twd_base`**——`fx_fee` 是 WillCard 向持卡人額外收取的費用，不屬於特店服務範疇；特店結算金額與手續費均以 `twd_base` 計算 |

**外幣金額計算：**

```
twd_base      = foreign_amount × fx_rate
fx_fee        = floor(twd_base × 0.015)
total_twd     = twd_base + fx_fee        ← 會員實際被扣款金額
reward_points = floor(twd_base × reward_rate)  ← 回饋基數為 twd_base，不含手續費
```

範例：USD$100，匯率 31.0，回饋率 1%
```
twd_base      = 100 × 31.0 = 3,100
fx_fee        = floor(3,100 × 0.015) = 46
total_twd     = 3,100 + 46 = 3,146   ← 會員扣 TWD$3,146
reward_points = floor(3,100 × 0.01) = 31 點
```

---

## 7. 交易限額規則

限額依卡片類型設定，儲存於 Card Service DB 的 `card_type_limit` 表。

| 預設值 | |
|-------|---|
| 單筆上限 | NT$100,000 |
| 單日累計上限 | NT$200,000 |

**Table: `card_type_limit`**

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | BIGINT PK | |
| `card_type` | VARCHAR | `STANDARD` / `GOLD` / `PLATINUM` / ... |
| `single_txn_limit` | DECIMAL(15,2) | 單筆上限 |
| `daily_limit` | DECIMAL(15,2) | 單日累計上限 |
| `currency` | VARCHAR(3) | `TWD` |
| `updated_at` | DATETIME | |

Card Service 在 Step 1 生成 OTP 之前，須同時驗證單筆限額與當日累計限額。

---

## 8. 複式帳本規則

### 8.1 設計原則

- 帳本分錄**不可變**——僅允許 INSERT，禁止 UPDATE / DELETE
- 每筆交易產生一組借貸平衡的分錄
- 補償一律寫入新的 REVERSAL 反向分錄

### 8.2 帳戶結構

**資產（Assets）**

| 帳戶代碼 | 帳戶名稱 | 說明 |
|----------|---------|------|
| `1001` | 應收會員款（Member Receivable）| 授權時對會員虛擬卡收取的金額 |
| `1002` | 清算備付金（Settlement Reserve）| 銀行清算帳戶現金；T+1 付款特店時借出 |

**負債（Liabilities）**

| 帳戶代碼 | 帳戶名稱 | 說明 |
|----------|---------|------|
| `2001` | 應付特店款（Merchant Payable）| 應結算給特店的淨額（含點數補貼）；手續費已扣除 |
| `2002` | 點數負債（Points Liability）| 已發放但未使用的回饋點數（1 點 = NT$1）；兌換時沖銷 |

**收入（Revenue）**

| 帳戶代碼 | 帳戶名稱 | 說明 |
|----------|---------|------|
| `3001` | 特店手續費收入（Merchant Fee Income）| 每筆台幣交易向特店收取的服務費 |
| `3002` | 外幣手續費收入（FX Fee Income）| 外幣交易收取的 1.5% 手續費 |

**費用（Expense）**

| 帳戶代碼 | 帳戶名稱 | 說明 |
|----------|---------|------|
| `4001` | 點數回饋費用（Reward Points Expense）| 發放回饋點數給會員的成本 |

**其他（Other）**

| 帳戶代碼 | 帳戶名稱 | 說明 |
|----------|---------|------|
| `5001` | 匯差損益（FX Gain/Loss）| WillCard 持有外幣部位時的匯差損益 |

### 8.3 分錄類型

| `journal_type` | 觸發時機 |
|---------------|---------|
| `AUTHORIZATION` | NCCC 授權成功 |
| `SETTLEMENT` | T+1 批次清算 — 現金移動：DR 2001 / CR 1002 |
| `REVERSAL` | 退款（另立 spec）|
| `REWARD` | 回饋點數發放（非同步，由 Kafka event 觸發）|

### 8.4 分錄範例

**情境一：台幣 NT$100，無折抵，回饋率 1%，特店手續費 3%**

| 借方 | 金額 | 貸方 | 金額 | 說明 |
|------|------|------|------|------|
| 應收會員款(1001) | 100.00 | 應付特店款(2001) | 97.00 | 授權分錄 |
| | | 特店手續費收入(3001) | 3.00 | |
| 回饋費用(4001) | 1.00 | 點數負債(2002) | 1.00 | 發放 1 點 — floor(100×1%) |

**情境二：台幣 NT$100，折抵 34 點 + 刷卡 NT$66，回饋率 1%，手續費 3%**

| 借方 | 金額 | 貸方 | 金額 | 說明 |
|------|------|------|------|------|
| 應收會員款(1001) | 66.00 | 應付特店款(2001) | 64.02 | 刷卡部分 |
| | | 特店手續費收入(3001) | 1.98 | 刷卡部分手續費 |
| 點數負債(2002) | 34.00 | 應付特店款(2001) | 34.00 | WillCard 補貼特店差額 |
| 回饋費用(4001) | 0.00 | 點數負債(2002) | 0.00 | floor(66×1%) = 0 點 |

> 特店共收到 NT$100（64.02 + 1.98 手續費保留 + 34.00 補貼 = NT$100 毛額）。

**情境三：外幣 USD$100，匯率 31.0，回饋率 1%，手續費 3%**

```
twd_base = 3,100 | fx_fee = 46 | total_twd = 3,146 | 回饋 = floor(3,100×1%) = 31 點
```

| 借方 | 金額 | 貸方 | 金額 | 說明 |
|------|------|------|------|------|
| 應收會員款(1001) | 3,146.00 | 應付特店款(2001) | 3,007.00 | twd_base × 0.97 |
| | | 特店手續費收入(3001) | 93.00 | twd_base × 3% |
| | | 外幣手續費收入(3002) | 46.00 | twd_base × 1.5% |
| 回饋費用(4001) | 31.00 | 點數負債(2002) | 31.00 | floor(twd_base × 1%) |

### 8.5 `journal_entry` 欄位定義

| 欄位 | 型別 | 說明 |
|------|------|------|
| `entry_id` | BIGINT PK | Snowflake |
| `idempotency_key` | VARCHAR(100) UNIQUE | `{kafka_message_key}_{entry_seq}`——防止 Kafka Consumer 重試時重複寫入 |
| `txn_id` | BIGINT | 來源交易 ID |
| `journal_type` | VARCHAR(20) | 見 §8.3 |
| `entry_type` | VARCHAR(10) | `DEBIT` / `CREDIT` |
| `account_code` | VARCHAR(10) | 見 §8.2 |
| `amount` | DECIMAL(15,4) | 金額 |
| `currency` | VARCHAR(3) | `TWD` / `USD` / ... |
| `fx_rate` | DECIMAL(10,6) | 台幣交易填 `1.0` |
| `amount_twd` | DECIMAL(15,4) | 台幣換算金額 |
| `memo` | VARCHAR(255) | 說明 |
| `created_at` | DATETIME(3) | 不可變，無 `updated_at` |

**冪等設計說明：**
- `txn.card.authorized` Kafka event payload 必須攜帶穩定的 `messageKey`（於 publish 時由 Snowflake 生成）。
- Ledger Service 對同一 event 內的每筆分錄依序指定 `entry_seq`（0, 1, 2 …），組合成 `idempotency_key = {messageKey}_{entry_seq}`。
- Consumer 重試時，`INSERT` 觸碰唯一約束即丟棄，不拋出例外（以 upsert-ignore 或捕捉 duplicate key 處理），確保分錄不重複、不漏記。

---

## 9. Operation Log Level

兩支 API 均為 **L1 等級**（金流操作）。`OperationLogEvent` 欄位格式詳見 `guideline/7-spec-guideline-zh-tw.md` §4.2。

| API | Kafka Topic |
|-----|------------|
| `card-pay`（Step 1）| `operation-log.card.pay` |
| `card-pay/confirm`（Step 2）| `operation-log.card.pay-confirm` |

每次呼叫無論成功或失敗均須發布事件。

---

## 10. 本章不涵蓋的範圍

以下功能明確**不在本章範圍**，將於獨立 spec 中定義：

- **退款流程** — 使用 REVERSAL 反向分錄；定義於 `11-refund-flow`
- **回饋方案後台管理** — 後續功能
- **卡片類型管理** — 於 Card Service onboarding spec 中定義

下一章：[退款流程](/guideline/11-refund-flow-zh-tw.md)（待規劃）
