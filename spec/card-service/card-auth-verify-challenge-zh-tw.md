# 卡片授權驗證挑戰 — card-service Spec

> **服務：** card-service
> **Contract Module：** card-service-contract
> **呼叫方：** [mock-nccc/pay-confirm-zh-tw.md](../mock-nccc/pay-confirm-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收來自 Mock NCCC 的 `challengeRef` 與 `otp`，對 Redis Session 中的 OTP 進行驗證，並回傳最終授權結果。

驗證通過：累計當日消費額、分配 Snowflake `txnId`，回傳 `{ result: APPROVED, txnId, cardType, maskedPan }` 供 ORCH 組裝 Kafka payload。

驗證失敗：同時遞增 per-session 嘗試計數器與 per-card 跨 Session 失敗計數器。卡片鎖定（15 分鐘內失敗 ≥ 5 次）優先於 Session 作廢（Session 內失敗 ≥ 3 次）。

> **不呼叫 Wallet Service，也不發布 `txn.card.authorized` Kafka。** 這兩個動作由 Transaction Orchestrator 統一負責（Phase 7b）。
> 此處唯一發布的 Kafka 事件為 `card.risk.otp-threshold-exceeded`——為卡片因多次 OTP 失敗遭凍結時觸發的風控事件。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（Mock NCCC） | [mock-nccc/pay-confirm-zh-tw.md](../mock-nccc/pay-confirm-zh-tw.md) |
| OTP Session 寫入方 | [card-service/card-auth-authorize-zh-tw.md](card-auth-authorize-zh-tw.md) |
| ORCH 第二腿（消費本端點回應） | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |

---

## 3. API 定義 — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /card-service |
| Controller mapping | /auth/verify-challenge |
| 實際接收路徑 | /card-service/auth/verify-challenge |
| 呼叫方 | Mock NCCC（`CardServiceFeignClient`） |

### Request（`card.service.contract.dto.CardAuthVerifyChallengeRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| challengeRef | String | ✅ | @NotBlank | 第一腿回傳的 UUID；識別 Redis 中的 OTP Session |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 使用者輸入的 6 位 OTP |

### Response（`card.service.contract.dto.CardAuthVerifyChallengeRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| result | String | `APPROVED` 或 `DECLINED` |
| txnId | String | card-service 分配的 Snowflake txnId（僅 APPROVED） |
| cardType | String | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE`（僅 APPROVED；供 ORCH Kafka payload 使用） |
| maskedPan | String | 遮罩 PAN，例如 `****1234`（僅 APPROVED；供 ORCH Kafka payload 使用） |
| reason | String | 拒絕原因碼（僅 DECLINED）；見下表 |
| remainingAttempts | Integer | 本 Session 剩餘 OTP 嘗試次數（僅 reason = `OTP_FAILED` 時存在） |

**拒絕原因值：**

| reason | 原因 |
|--------|------|
| `SESSION_EXPIRED` | Redis 中找不到 OTP Session（TTL 180s 已過或從未建立） |
| `OTP_FAILED` | OTP 錯誤；Session 仍有效 |
| `SESSION_VOIDED` | 同一 Session 連續錯誤 ≥ 3 次；Session 作廢 |
| `CARD_LOCKED` | 同一卡片 15 分鐘內連續錯誤 ≥ 5 次；卡片凍結 |

---

## 4. Service UseCase Flow

**UseCase：** `CardAuthVerifyChallengeUseCase.verify(CardAuthVerifyChallengeRq)`

### Step 1 — Session 查詢

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[REDIS READ]` | GET `otp:{challengeRef}`；找不到 → **RETURN** `{ result: DECLINED, reason: SESSION_EXPIRED }` |
| 2 | `[DOMAIN]` | 比對 `session.otpValue == rq.otp`；相符 → **OTP 正確路徑**（Step 3）；不符 → **OTP 錯誤路徑**（Step 8） |

---

### OTP 正確路徑

| 步驟 | 類型 | 說明 |
|------|------|------|
| 3 | `[REDIS WRITE]` | `INCRBY card:daily:{session.cardId}:{yyyyMMdd} session.txnTwdBaseAmount`；若為新 key（結果 = txnTwdBaseAmount）→ 設定 TTL 至當日 23:59:59 Asia/Taipei |
| 4 | `[DOMAIN]` | 分配 `txnId`（Snowflake） |
| 5 | `[DB READ]` | `SELECT card_type, pan_masked FROM card WHERE card_id = session.cardId` |
| 6 | `[REDIS WRITE]` | DEL `otp:{challengeRef}` |
| 7 | `[RETURN]` | `{ result: APPROVED, txnId, cardType: card.card_type, maskedPan: card.pan_masked }` |

---

### OTP 錯誤路徑

| 步驟 | 類型 | 說明 |
|------|------|------|
| 8 | `[REDIS WRITE]` | `INCR otp:card:fail:{session.cardId}`；若為新 key → 設定 TTL 900s；結果存為 `cardFailCount` |
| 9 | `[REDIS WRITE]` | `INCR otp:attempt:{challengeRef}`；若為新 key → 設定 TTL 180s；結果存為 `attemptCount` |
| 10 | `[DOMAIN]` | 若 `cardFailCount ≥ 5` → **卡片鎖定子路徑**（Steps 11–13） |
| 11 | `[DB WRITE]` | `UPDATE card SET status = 'FROZEN', updated_at = NOW(3) WHERE card_id = session.cardId` |
| 12 | `[REDIS WRITE]` | DEL `otp:{challengeRef}` |
| 13 | `[KAFKA]` | 發布 `card.risk.otp-threshold-exceeded`（見 Kafka 章節）；若發布失敗 → **記錄 ERROR，不拋出例外，不 rollback FROZEN 狀態**；**RETURN** `{ result: DECLINED, reason: CARD_LOCKED }` |
| 14 | `[DOMAIN]` | 若 `attemptCount ≥ 3` → **Session 作廢子路徑**：DEL `otp:{challengeRef}`；**RETURN** `{ result: DECLINED, reason: SESSION_VOIDED }` |
| 15 | `[RETURN]` | `{ result: DECLINED, reason: OTP_FAILED, remainingAttempts: 3 - attemptCount }` |

> **計數器順序（Steps 8–9）：** 兩個計數器先於任何條件判斷之前遞增。
> 確保 `otp:card:fail` 無論 Session 狀態如何都一定更新——防止已作廢的 Session 遮蔽卡片層級的失敗累計。
>
> **卡片鎖定優先於 Session 作廢（Step 10 在 Step 14 之前）：**
> 若第 3 次 OTP 錯誤同時觸發第 5 次卡片層級失敗，回傳 `CARD_LOCKED`（更嚴重）。
>
> **TTL 行為：** `INCR` 不影響既有 key 的 TTL。僅在 key 新建時（結果 = 1）設定 TTL。
>
> **CARD_LOCKED 時 Kafka 發布失敗（Step 13）：** 若 Kafka 發布失敗，僅記錄 ERROR，**不拋出例外，不 rollback FROZEN 狀態**。卡片安全鎖定的優先級高於風控事件通知。風控事件遺失應由監控告警補償，而非犧牲鎖卡一致性。

---

## 5. Kafka 事件 — `card.risk.otp-threshold-exceeded`

**Topic：** `card.risk.otp-threshold-exceeded`
**發布方：** card-service（Step 13，僅卡片鎖定子路徑）
**Event Class：** `card.service.event.CardRiskOtpThresholdExceededEvent`

| 欄位 | 型別 | 來源 | 說明 |
|------|------|------|------|
| cardId | Long | `session.cardId` | 被凍結的卡片 ID |
| userId | Long | `session.userId` | 卡片持有人 Snowflake ID |
| lockedAt | String | 系統時鐘 | 卡片被凍結的 ISO-8601 時間戳 |

> 此為**風控 / 安全事件**，與 ORCH 發布的 `txn.card.authorized` 屬不同性質的 Topic。
> 下游消費者（例如通知、稽核服務）可獨立訂閱此 Topic。

---

## 6. Redis Keys

### OTP Session（本 UseCase 讀取並刪除）

**Key：** `otp:{challengeRef}` — 由 `card-auth-authorize` 寫入；見 [card-auth-authorize-zh-tw.md](card-auth-authorize-zh-tw.md)

### 每日消費累計計數器

| 欄位 | 值 |
|------|---|
| Key | `card:daily:{cardId}:{yyyyMMdd}` |
| 操作 | `INCRBY txnTwdBaseAmount`（Step 3；僅 OTP 正確路徑） |
| TTL | 至當日 23:59:59 Asia/Taipei 的剩餘秒數 |
| 讀取方 | `card-auth-authorize` Step 8（限額預查） |

### Per-Session OTP 嘗試計數器

| 欄位 | 值 |
|------|---|
| Key | `otp:attempt:{challengeRef}` |
| 操作 | `INCR`（Step 9；OTP 錯誤路徑） |
| TTL | 180s（與 OTP Session 同壽命） |
| 觸發閾值 | ≥ 3 → SESSION_VOIDED |

### Per-Card 跨 Session 失敗計數器

| 欄位 | 值 |
|------|---|
| Key | `otp:card:fail:{cardId}` |
| 操作 | `INCR`（Step 8；OTP 錯誤路徑） |
| TTL | 900s（15 分鐘滾動視窗） |
| 觸發閾值 | ≥ 5 → CARD_LOCKED |

---

## 7. 資料庫

| 表 | 操作 | 欄位 | 條件 |
|----|------|------|------|
| `card` | READ（Step 5） | `card_type`, `pan_masked` | `card_id = session.cardId`；僅 OTP 正確路徑 |
| `card` | WRITE（Step 11） | `status = 'FROZEN'`, `updated_at` | `card_id = session.cardId`；僅卡片鎖定子路徑 |

---

## 8. 錯誤碼

所有 `DECLINED` 結果（含 `SESSION_EXPIRED`）以 **HTTP 200** 回傳——屬業務結果，非錯誤。

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 9. Changelog

### v1.1 — 2026-03 — 定義 CARD_LOCKED Kafka 失敗行為
- Step 13：Kafka 發布失敗時僅記錄 ERROR，不拋出例外，不 rollback FROZEN 狀態。

### v1.0 — 2026-03 — 初始版本
