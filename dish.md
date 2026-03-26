# 信用卡交易流程討論稿 (dish.md)

> **用途：** 業務邏輯討論草稿，已確認事項會同步更新至 `guideline/10-txn-flow.md`。
> **對應功能：** `10-credit-card-txn`
> **狀態：** QA 第一輪已確認完畢，guideline 已產出。退款流程另立獨立 spec。

---

## 一、整體交易架構概念

信用卡交易在 WillCard 中是一個**兩步驟 Saga 流程**：

```
Step 1: card-pay（發起交易）
  ├─ 點數優先扣抵 reserve
  ├─ 外幣匯率鎖定（若為外幣交易）
  ├─ 3DS OTP 請求
  └─ 回傳 txnRef（Saga session identifier）

Step 2: card-pay/confirm（OTP 驗證 + 正式授權）
  ├─ OTP 驗證
  ├─ Orchestrator Saga 執行
  │     ├─ 卡片狀態驗證
  │     ├─ encryptedCard 解密 → NCCC 授權
  │     ├─ 點數確認扣除
  │     └─ 帳本分錄寫入
  └─ 非同步：回饋點數計算、對帳、通知
```

---

## 二、參與角色與外部系統

| 角色 | 說明 |
|------|------|
| **Client（App）** | 使用者手機 App，發起交易、輸入 OTP |
| **BFF** | 唯一對外入口，JWT 驗證、冪等控制、Saga 發起 |
| **NCCC（聯合信用卡處理中心）** | 台灣信用卡授權機構，負責 3DS 加密、轉發至 Visa/Mastercard |
| **Card Service** | 虛擬卡管理、combineKey 解密、OTP 生成與驗證、卡片額度表 |
| **Wallet Service** | 台幣點數餘額管理（Reserve / Confirm / Release）|
| **Points Service** | 點數回饋計算、點數 Batch 帳戶、到期管理 |
| **FX Service** | 外幣交易匯率鎖定（無狀態工具）|
| **Transaction Orchestrator** | Saga 狀態機協調、補償交易 |
| **Ledger Service** | 不可變複式帳本 |
| **Notification Service** | OTP SMS、交易成功推播、Email receipt |
| **Reconciliation Service** | T+0 即時對帳、T+1 批次清算、稽核日誌消費者 |

---

## 三、前置條件

1. 使用者已登入，BFF 持有有效 JWT
2. 使用者已綁定信用卡（Card Service，狀態 `ACTIVE`，未過期）
3. 未超過卡片類型對應的單筆 / 單日限額
4. 商家已完成 onboarding（有 merchantId 與 MCC Code）

---

## 四、點數優先扣抵邏輯（Points-First）

### 4.1 換算規則（已確認）

| 項目 | 規則 |
|------|------|
| 換算比例 | **1 點 = NT$1**（1:1） |
| 最小折抵單位 | **NT$1**（即 1 點） |
| 最大折抵比例 | **全額折抵**（可 100% 用點數付款） |

### 4.2 折抵流程

使用者在 App 輸入折抵點數（`pointsToUse`），系統計算：

```
刷卡金額 = 原始金額 - pointsToUse
（pointsToUse ≤ 可用點數餘額，且 pointsToUse ≤ 原始金額）
```

範例：
```
原始金額：NT$100
使用點數：34 點（= NT$34）
實際刷卡：NT$66

交易紀錄需同時記錄：
  - points_used:    34
  - card_amount:    66
  - total_amount:   100
```

### 4.3 Reserve-Confirm-Release 機制

```
Step 1 card-pay：
  Wallet Service.reserve(userId, 34點)
  → 點數帳戶 available_balance -= 34，reserved_balance += 34
  → Reserve 有效期與 OTP TTL 對齊（180 秒），TTL 到期視為自動 Release

Step 2 card-pay/confirm（授權成功）：
  Wallet Service.confirmDeduct(reservationId)
  → reserved_balance -= 34（實際扣除）

補償（OTP 失敗 / 授權失敗 / TTL 到期）：
  Wallet Service.release(reservationId)
  → reserved_balance -= 34，available_balance += 34
```

> **設計備注（Q4 可靠性）：** Redis TTL 到期後 OTP 失效，Reserve 的 DB 狀態若未即時清理，需考慮補償：
> 建議 Reserve 記錄本身帶有 `expire_at`，Wallet Service 在查詢可用餘額時過濾已過期的 reserve（惰性清理），避免引入 scheduler 依賴。

---

## 五、3DS 認證流程

### 5.1 OTP 生命週期

| 項目 | 規則 |
|------|------|
| OTP 長度 | 6 位數 |
| 有效期 | 180 秒 |
| Redis Key | `otp:{txnRef}`（TTL 180s） |
| 重試上限 | 3 次（計數存於 Redis `otp:attempt:{txnRef}`）|
| 失敗達上限 | **此次 txnRef 作廢**，卡片狀態不受影響 |

### 5.2 加解密規則

詳見 `guideline/6-pcidss-zh-tw.md` §2.2。關鍵規則：

- `encryptedCard` 由 NCCC 使用 combineKey 加密
- 解密後明文**僅存 JVM Heap**，授權完成後立即清除
- CVV **絕不**寫入任何持久化儲存
- PAN 僅以遮罩格式 `****-****-****-1234` 記錄

---

## 六、完整交易 Saga 流程

### 6.1 Saga 發起（Step 1: card-pay）

```
Client → BFF  POST /api/v1/card/pay
  ├─ [VALIDATE] JWT、請求參數（amount > 0, cardId, merchantId, pointsToUse）
  ├─ [REDIS WRITE] 冪等 SETNX idempotency:{Idempotency-Key} PROCESSING
  ├─ [FEIGN → Wallet Service] reserve(userId, pointsToUse)（若 pointsToUse > 0）
  │     └─ 失敗（點數不足）→ 回傳錯誤，前端需重新輸入折抵金額
  ├─ [FEIGN → FX Service] lockRate(currency, amount)（若為外幣）→ 回傳 fxRateId
  ├─ [FEIGN → Card Service] generateOtp(cardId) → 回傳 txnRef
  ├─ [KAFKA L1] Publish operation-log.card.pay（INITIATED）
  └─ [RETURN] { txnRef, cardAmount, pointsUsed, fxRateId? }
```

### 6.2 Saga 確認（Step 2: card-pay/confirm）

```
Client → BFF  POST /api/v1/card/pay/confirm
  ├─ [VALIDATE] JWT、txnRef、otp
  ├─ [REDIS WRITE] 冪等 SETNX idempotency:{Idempotency-Key} PROCESSING
  ├─ [FEIGN → Card Service] verifyOtp(txnRef, otp)
  │     └─ 失敗 → 補償 release(reservationId)，回傳 OTP_FAILED
  │
  ├─ [FEIGN → Transaction Orchestrator] authorize(txnRef, fxRateId?)
  │     │
  │     │  Orchestrator Saga 步驟：
  │     ├─ S1. Card Service: validateCard(cardId) → 狀態 / 過期 / 限額
  │     ├─ S2. Card Service: decryptAndAuthorize(encryptedCard)
  │     ├─ S3. NCCC External: sendAuthRequest → 取得 authCode
  │     │     └─ 失敗 → 補償 release(reservationId)，Saga = AUTH_FAILED
  │     ├─ S4. Wallet Service: confirmDeduct(reservationId)（若有折抵）
  │     ├─ S5. Ledger Service: recordJournalEntries(txnId, ...)
  │     ├─ S6. Kafka: Publish txn.card.authorized
  │     │     ├─ Points Service → 計算回饋點數（PENDING batch）
  │     │     ├─ Reconciliation Service → T+0 對帳
  │     │     └─ Notification Service → Push + Email
  │     └─ Saga state = COMPLETED
  │
  ├─ [KAFKA L1] Publish operation-log.card.pay-confirm（SUCCESS / FAIL）
  ├─ [REDIS WRITE] 更新 idempotency key = COMPLETED
  └─ [RETURN] { txnId, authCode, totalAmount, cardAmount, pointsUsed, estimatedRewardPoints }
```

### 6.3 Saga 補償表

| 步驟 | 補償動作 | 觸發條件 |
|------|---------|---------|
| Reserve 點數 | `release(reservationId)` | OTP 失敗、授權失敗、OTP TTL 到期（惰性）|
| 鎖定匯率 | FX Rate TTL 自動過期 | 無需主動補償 |
| Ledger 分錄 | 不刪除；寫入 REVERSAL 反向分錄 | 退款流程（另立 spec）|

---

## 七、點數回饋計算（已確認）

### 7.1 計算規則

```
回饋點數 = floor(card_amount × reward_rate)

範例：
  card_amount = 60, reward_rate = 2%
  60 × 0.02 = 1.2 → floor → 1 點
```

**規則：**
- 回饋基數為**實際刷卡金額**（不含點數折抵部分）
- 無條件捨去（floor），不四捨五入
- 外幣交易以**換算後台幣金額**計算回饋

### 7.2 Reward Plan（MCC 對應回饋率）

回饋率**存於 DB**，後台可動態設定。先建立 `reward_plan` 表，管理 UI 為後續功能。

| 商家類別（MCC 範例）| 回饋率 | 說明 |
|--------------------|--------|------|
| 餐廳（5812, 5814）| 3% | 待業務確認初始值 |
| 超商（5411）| 2% | |
| 網購（5965, 5999）| 2% | |
| 其他 | 1% | 預設回饋 |

> **備注：** 初始 MCC 回饋率數值需業務確認後填入 DB 初始化腳本（Flyway DML）。

### 7.3 點數 Batch 設計

每次回饋產生一筆獨立 Batch，記錄可用餘額與到期時間：

```
point_reward_batch（Points Service DB）
  batch_id         BIGINT PK (Snowflake)
  user_id          BIGINT
  source_txn_id    BIGINT    關聯交易
  issued_amount    INT       發放點數
  remaining_balance INT      可用餘額（折抵時遞減）
  status           VARCHAR   PENDING / CONFIRMED / EXPIRED / CANCELLED
  expires_at       DATETIME  發放日 + 1 年
  created_at       DATETIME
```

---

## 八、複式記帳（Double-Entry Bookkeeping）

### 8.1 帳戶結構（初稿，需會計師確認）

| 帳戶代碼 | 帳戶名稱 | 類型 | 說明 |
|----------|---------|------|------|
| `1001` | 應收帳款 | Asset | 持卡人消費授權金額 |
| `2001` | 應付商家款 | Liability | 應付給商家的結算金額（含點數補貼）|
| `2002` | 點數負債 | Liability | 已發放但未使用的回饋點數 |
| `3001` | 手續費收入 | Revenue | Interchange fee / 商家手續費 |
| `4001` | 回饋點數費用 | Expense | 發放回饋點數的成本 |
| `5001` | 匯差損益 | Revenue/Expense | 外幣交易匯差 |

### 8.2 分錄範例

**情境一：純刷卡 NT$100（無折抵，回饋率 1%，手續費假設 3%）**

| 借方 | 金額 | 貸方 | 金額 | 說明 |
|------|------|------|------|------|
| 應收帳款(1001) | 100 | 應付商家款(2001) | 97 | 刷卡授權 |
|  | | 手續費收入(3001) | 3 | |
| 回饋費用(4001) | 1 | 點數負債(2002) | 1 | 回饋 1 點 = NT$1 |

**情境二：折抵 NT$34 + 刷卡 NT$66（原價 NT$100，回饋率 1%，手續費 3%）**

| 借方 | 金額 | 貸方 | 金額 | 說明 |
|------|------|------|------|------|
| 應收帳款(1001) | 66 | 應付商家款(2001) | 64.02 | 刷卡部分 |
|  | | 手續費收入(3001) | 1.98 | |
| 點數負債(2002) | 34 | 應付商家款(2001) | 34 | WillCard 補貼商家差額 |
| 回饋費用(4001) | 0 | 點數負債(2002) | 0 | floor(66 × 1%) = 0 點 |

> **商家補貼確認：** 商家仍收到 NT$100，WillCard（與合作銀行）共同承擔 NT$34 的點數補貼成本。

### 8.3 Journal Entry Schema（初稿）

```
journal_entry（Ledger Service DB）
  entry_id        BIGINT PK (Snowflake)
  txn_id          BIGINT
  journal_type    VARCHAR(20)    AUTHORIZATION / SETTLEMENT / REVERSAL / REWARD
  entry_type      VARCHAR(10)    DEBIT / CREDIT
  account_code    VARCHAR(10)
  amount          DECIMAL(15,4)
  currency        VARCHAR(3)     TWD / USD / ...
  fx_rate         DECIMAL(10,6)  台幣交易為 1.0
  amount_twd      DECIMAL(15,4)  台幣換算金額
  memo            VARCHAR(255)
  created_at      DATETIME(3)    不可變，無 updated_at
```

---

## 九、外幣交易處理（已確認）

| 項目 | 規則 |
|------|------|
| 外幣手續費 | **固定 1.5%**，加收於刷卡金額之上 |
| 匯率來源 | 待確認（台銀牌告 / 第三方 API）→ 目前由 FX Service 管理 |
| 匯率鎖定 | Step 1 呼叫 FX Service 鎖定，回傳 `fxRateId`（有效期 10 分鐘）|
| 帳本記錄 | 以台幣記帳，附記原幣金額 + 匯率 |
| 回饋計算 | 以換算後**台幣金額**計算（含手續費？待確認）|

外幣刷卡計算範例：
```
原幣：USD$100
匯率：31.0
台幣等值：TWD$3,100
外幣手續費（1.5%）：TWD$46.5
實際扣款：TWD$3,146.5
```

---

## 十、交易限額設計（已確認）

| 限制類型 | 預設值 | 說明 |
|---------|--------|------|
| 單筆上限 | NT$100,000 | 依卡片類型可覆蓋 |
| 單日累計上限 | NT$200,000 | 依卡片類型可覆蓋 |

限額**依卡片類型設定**，需一張 `card_type_limit` 表：

```
card_type_limit（Card Service DB）
  id                BIGINT PK
  card_type         VARCHAR     STANDARD / GOLD / PLATINUM / ...
  single_txn_limit  DECIMAL     單筆上限
  daily_limit       DECIMAL     單日累計上限
  currency          VARCHAR     TWD
  updated_at        DATETIME
```

---

## 十一、通知設計

| 時機 | 通知方式 | 內容摘要 |
|------|---------|---------|
| OTP 生成 | SMS | 「WillCard 驗證碼：{OTP}，3 分鐘內有效」 |
| 授權成功 | App Push + Email | 商家、總金額、點數折抵、預計回饋點數 |
| 授權失敗 | App Push | 失敗原因 |

---

## 十二、Operation Log Level

| API | Log Level | Kafka Topic |
|-----|-----------|-------------|
| card-pay（Step 1）| L1 | `operation-log.card.pay` |
| card-pay/confirm（Step 2）| L1 | `operation-log.card.pay-confirm` |

---

## 十三、已確認 QA 紀錄

| # | 問題 | 答案 |
|---|------|------|
| Q1 | 點數換算比例 | 1:1（1 點 = NT$1），最小折抵 1 點，可全額折抵 |
| Q2 | 點數 reserve 失敗行為 | 點數不足時回傳錯誤，使用者重新輸入折抵金額 |
| Q3 | OTP 失敗鎖定範圍 | 鎖定此次 txnRef，卡片不受影響 |
| Q4 | Saga 逾時清理 | TTL 到期即失效，Reserve 用惰性清理（expire_at 欄位）|
| Q5 | MCC 回饋率設定 | 存 DB，後台可動態設定，管理 UI 另行開發 |
| Q6 | 點數折抵部分計算回饋 | 不計算，回饋僅基於實際刷卡金額，floor 無條件捨去 |
| Q7 | 回饋點數到期日 | 發放後 1 年，每批次獨立追蹤餘額與到期時間 |
| Q8 | 商家結算 | 商家仍收全額，WillCard + 合作銀行承擔折抵差額 |
| Q9 | 外幣交易 | 包含，固定 1.5% 手續費 |
| Q10 | 退款流程 | 另立獨立 spec，此次不包含 |
| Q11 | 交易限額 | 單筆 NT$100,000，單日 NT$200,000，依卡片類型可覆蓋 |

---

## 十四、後續待確認項目

| # | 問題 | 狀態 | 說明 |
|---|------|------|------|
| P1 | 初始 MCC 回饋率數值 | ✅ 確認 | 存 DB，啟動時載入 Map，更新時刷新 cache；初始值由業務填入 Flyway DML |
| P2 | 外幣回饋計算基數 | ✅ 確認 | 使用 `twd_base`（純匯率換算），不含 1.5% 手續費 |
| P3 | 匯率來源 | ✅ 確認 | 由 FX Service 統一管理（台銀牌告 / 第三方 API 由 FX Service 決定）|
| P4 | 帳本帳戶設計 | ✅ 確認 | 已設計，見 guideline §8.2；台幣不收手續費，外幣收 1.5% 手續費 |
| P5 | 卡片類型清單 | ⏳ 延後 | Card Type 定義於 Card Service onboarding spec，後面再定義 |

---

## 十五、架構修正紀錄（重要）

### 修正一：WillCard 是 Issuer，NCCC 呼叫 WillCard

原始設計錯誤地將 WillCard 設計為主動呼叫 NCCC。
實際上：NCCC 收到 Acquirer 轉來的交易後，主動呼叫 WillCard 的 Issuer Authorization API 詢問是否授權。

**Card Service 對外暴露兩個端點（不透過 BFF）：**
- `POST /card/auth/authorize` — NCCC 呼叫，驗卡 + 觸發 OTP
- `POST /card/auth/verify-challenge` — NCCC 呼叫，驗 OTP + 最終授權決策

### 修正二：OTP 觸發時機

原始設計：App 主動發起 → 生成 OTP → 用戶確認 → 才去 NCCC 授權。
正確設計：NCCC 先將交易送到 Issuer（Card Service）→ Card Service 判斷需要 3DS → 觸發 OTP → 用戶輸入 → Card Service 回傳授權結果給 NCCC。

### 修正三：Mock NCCC 設計

為了在 App 中測試完整流程，設計 Mock NCCC Service：
- App → BFF → Mock NCCC → Card Service（走相同授權路徑）
- Mock NCCC 扮演 NCCC 角色，不繞過授權邏輯

### 修正四：Points-First 改為 Preference 驅動

原始設計：App 每次輸入 `pointsToUse`。
正確設計：會員在 App 設定「點數優先」偏好（`member_points_preference` table）。
Card Service 收到授權請求時自動讀取偏好並計算 `pointsToUse`，回傳 `pointsPreview` 供 App 顯示（如 LinePay 樣式）。

---

## 十六、禁止事項

- **禁止在 dish.md 確認完成前產生 spec 文件**
- 已產出 guideline：`guideline/10-txn-flow.md`、`guideline/10-txn-flow-zh-tw.md`
