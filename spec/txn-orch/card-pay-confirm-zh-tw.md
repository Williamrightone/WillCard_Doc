# 刷卡付款確認 — Transaction Orchestrator Spec

> **服務：** txn-orch
> **Contract Module：** txn-orch-contract
> **呼叫方：** [mobile-bff/card-pay-confirm-zh-tw.md](../mobile-bff/card-pay-confirm-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

刷卡付款 Saga 的**第二腿**。讀取第一腿儲存的 Saga State，將 OTP 轉發至 Mock NCCC（呼叫 card-service auth/verify-challenge），並完成 Saga：

- **APPROVED**：確認 Wallet 預留、發布 `txn.card.authorized` Kafka 事件、刪除 Saga State。
- **DECLINED + SESSION_VOIDED / CARD_LOCKED**：釋放 Wallet 預留、刪除 Saga State。
- **DECLINED + OTP_FAILED**：不執行 Wallet 操作，Saga State 保留（Session 仍有效）。

回傳最終交易結果給 BFF。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（BFF） | [mobile-bff/card-pay-confirm-zh-tw.md](../mobile-bff/card-pay-confirm-zh-tw.md) |
| ORCH → Mock NCCC（OTP 驗證） | [mock-nccc/pay-confirm-zh-tw.md](../mock-nccc/pay-confirm-zh-tw.md) |
| ORCH → Wallet Service（confirmDeduct / release） | [wallet-service/confirm-deduct-zh-tw.md](../wallet-service/confirm-deduct-zh-tw.md) |
| 第一腿（建立 Saga State + challengeRef） | [txn-orch/card-pay-zh-tw.md](card-pay-zh-tw.md) |

---

## 3. API 定義 — txn-orch

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /txn-orch |
| Controller mapping | /card/pay/confirm |
| 實際接收路徑 | /txn-orch/card/pay/confirm |
| 呼叫方 | BFF（TxnOrchFeignClient） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| X-Member-Id | ✅ | 已驗證的使用者 Snowflake ID（BFF 轉發） |

### Request（`txn-orch.contract.dto.CardPayConfirmOrchRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| challengeRef | String | ✅ | @NotBlank | 第一腿回傳的 UUID；識別 OTP Session |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 使用者輸入的 6 位 OTP |

### Response（`txn-orch.contract.dto.CardPayConfirmOrchRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| result | String | `APPROVED` 或 `DECLINED` |
| txnId | String | card-service 分配的 Snowflake txnId（僅 APPROVED） |
| reason | String | 拒絕原因碼（僅 DECLINED）；見下表 |
| remainingAttempts | Integer | OTP 剩餘輸入次數（僅 reason = `OTP_FAILED` 時存在） |

**拒絕原因值（由 card-service verify-challenge 定義）：**

| reason | 原因 |
|--------|------|
| `SESSION_EXPIRED` | Redis 中找不到 Saga State（TTL 在確認呼叫前已過期） |
| `OTP_FAILED` | OTP 錯誤；card-service Session 仍有效；回傳 `remainingAttempts` |
| `SESSION_VOIDED` | 同一 Session 連續錯誤 ≥ 3 次；Session 作廢；Reservation 釋放 |
| `CARD_LOCKED` | 同一卡片 15 分鐘內連續錯誤 ≥ 5 次；卡片 FROZEN；Reservation 釋放 |

---

## 4. Service UseCase Flow

**UseCase：** `CardPayConfirmOrchUseCase.confirm(CardPayConfirmOrchRq)`（txn-orch）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[REDIS READ]` | GET `saga:{challengeRef}`；不存在 → 立即回傳 `{ result: DECLINED, reason: SESSION_EXPIRED }` |
| 2 | `[FEIGN]` | `MockNcccFeignClient.confirm(MockNcccPayConfirmRq)` → `POST /mock-nccc/pay/confirm`；Request：`{ challengeRef, otp }`；Response：`{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }`；見 [mock-nccc/pay-confirm-zh-tw.md](../mock-nccc/pay-confirm-zh-tw.md) |

**若 `result = APPROVED`：**

| 步驟 | 類型 | 說明 |
|------|------|------|
| 3 | `[FEIGN]` | 若 `sagaState.reservationId != null`：`WalletFeignClient.confirmDeduct(reservationId)` — 見 [wallet-service/confirm-deduct-zh-tw.md](../wallet-service/confirm-deduct-zh-tw.md) |
| 4 | `[KAFKA]` | 發布 `txn.card.authorized`（見下方 Kafka 事件章節） |
| 5 | `[REDIS WRITE]` | DEL `saga:{challengeRef}` |
| 6 | `[RETURN]` | 回傳 `{ result: APPROVED, txnId }` |

**若 `result = DECLINED` + reason = `SESSION_VOIDED` 或 `CARD_LOCKED`：**

| 步驟 | 類型 | 說明 |
|------|------|------|
| 7 | `[FEIGN]` | 若 `sagaState.reservationId != null`：`WalletFeignClient.release(reservationId)` — 見 [wallet-service/confirm-deduct-zh-tw.md](../wallet-service/confirm-deduct-zh-tw.md) |
| 8 | `[REDIS WRITE]` | DEL `saga:{challengeRef}` |
| 9 | `[RETURN]` | 回傳 `{ result: DECLINED, reason }` |

**若 `result = DECLINED` + reason = `OTP_FAILED`：**

| 步驟 | 類型 | 說明 |
|------|------|------|
| 10 | `[RETURN]` | 回傳 `{ result: DECLINED, reason: OTP_FAILED, remainingAttempts }`；**不**執行 release；Saga State 保留；Session 仍有效 |

> 所有 `DECLINED` 結果以 **HTTP 200** 回傳——屬業務結果，非錯誤。

---

## 5. Kafka 事件 — `txn.card.authorized`

**Topic：** `txn.card.authorized`
**發布方：** ORCH（Step 4，僅 APPROVED 路徑）
**Event Class：** `txn-orch.event.TxnCardAuthorizedEvent`

| 欄位 | 型別 | 來源 | 說明 |
|------|------|------|------|
| txnId | String | verify-challenge response | card-service 分配的 Snowflake txnId |
| userId | Long | Saga State | 使用者 Snowflake ID |
| cardId | Long | Saga State | 卡片 Snowflake ID |
| cardType | String | verify-challenge response | `CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE` |
| maskedPan | String | verify-challenge response | 遮罩 PAN（例如 `****1234`）；用於顯示與稽核 |
| merchantId | String | Saga State | 商戶識別碼 |
| txnAmount | Long | Saga State | 原始交易金額（分） |
| txnCurrency | String | Saga State | ISO 4217 幣別代碼 |
| txnTwdBaseAmount | Long | Saga State | FX 換算後的台幣基準金額 |
| isOverseas | Boolean | Saga State | 是否為國外交易 |
| pointsToUse | Long | Saga State | 折抵點數（分）；`0` 表示不折抵 |
| txnTimestamp | String | ORCH 發布時間 | ISO-8601 時間戳（例如 `2026-03-31T10:00:00.000+08:00`） |

---

## 6. Saga State（Redis）

**Key：** `saga:{challengeRef}` — 由 [txn-orch/card-pay-zh-tw.md](card-pay-zh-tw.md) 寫入
**生命週期：**

| 事件 | 動作 |
|------|------|
| 第二腿 — APPROVED | wallet confirmDeduct + Kafka 發布後 DEL（Step 5） |
| 第二腿 — SESSION_VOIDED / CARD_LOCKED | wallet release 後 DEL（Step 8） |
| 第二腿 — OTP_FAILED | 保留（Session 與 Reservation 仍有效） |
| TTL 到期（180s） | Redis 自動刪除 |

> **殭屍 Reservation 風險：** Saga State 因 TTL 到期被刪除後，Leg 1 已建立的 `wallet_reservation` 將永久停留在 `PENDING` 狀態——`reservationId` 隨 Saga State 一同消失，ORCH 無法追蹤。wallet-service 必須以排程任務定期釋放超過 10 分鐘仍為 PENDING 的 reservation（詳見 `reserve.md` § Scheduled Cleanup）。

---

## 7. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

> `DECLINED` 結果（`SESSION_EXPIRED`、`OTP_FAILED`、`SESSION_VOIDED`、`CARD_LOCKED`）以 **HTTP 200** 回傳——屬業務結果，非錯誤。

---

## 8. Changelog

### v1.1 — 2026-03 — 新增殭屍 Reservation 備注
- 新增設計備注：Saga State TTL 到期後，wallet reservation 仍為 PENDING，需 wallet-service 排程清理。

### v1.0 — 2026-03 — 初始版本
