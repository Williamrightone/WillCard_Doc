# 刷卡付款 — Transaction Orchestrator Spec

> **服務：** txn-orch
> **Contract Module：** txn-orch-contract
> **呼叫方：** [mobile-bff/card-pay-zh-tw.md](../mobile-bff/card-pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

刷卡付款 Saga 的**第一腿**。接收 BFF 發起的付款請求，執行 FX 換算、計算折抵點數、轉發至 Mock NCCC（負責驗卡與 OTP 生成），若有折抵則預留 Wallet 點數，並將 Saga State 儲存供第二腿使用。

回傳 `challengeRef`、`pointsToUse` 及 `estimatedTwdAmount` 給 BFF。

> ORCH 統一負責所有 Saga 協調：FX 換算、Wallet reserve/confirm/release、Saga State 管理、Kafka 發布。card-service 不發起任何跨服務呼叫。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（BFF） | [mobile-bff/card-pay-zh-tw.md](../mobile-bff/card-pay-zh-tw.md) |
| ORCH → FX Service（外幣換算） | [fx-service/convert-zh-tw.md](../fx-service/convert-zh-tw.md) |
| ORCH → Wallet Service（餘額查詢 + 預留） | [wallet-service/reserve-zh-tw.md](../wallet-service/reserve-zh-tw.md) |
| ORCH → Mock NCCC（驗卡 + OTP） | [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md) |
| 第二腿（OTP 確認） | [txn-orch/card-pay-confirm-zh-tw.md](card-pay-confirm-zh-tw.md) |

---

## 3. API 定義 — txn-orch

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /txn-orch |
| Controller mapping | /card/pay |
| 實際接收路徑 | /txn-orch/card/pay |
| 呼叫方 | BFF（TxnOrchFeignClient） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| X-Member-Id | ✅ | 已驗證的使用者 Snowflake ID（BFF 從 Security Context 轉發） |

### Request（`txn-orch.contract.dto.CardPayOrchRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 扣款卡片（Snowflake ID as String） |
| amount | Long | ✅ | @NotNull; @Min(1) | 交易金額（分） |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 幣別代碼（例如 `TWD`、`USD`） |
| merchantId | String | ✅ | @NotBlank | 商戶識別碼 |
| usePoints | Boolean | ✅ | @NotNull | 是否套用點數折抵；僅由 ORCH 處理，不轉發至下游 |

### Response（`txn-orch.contract.dto.CardPayOrchRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| challengeRef | String | UUID；OTP Session 綁定；傳入第二腿確認端點；有效期 180s |
| pointsToUse | Long | 將折抵的點數（分）；usePoints = false 或餘額不足時為 `0` |
| estimatedTwdAmount | Long | FX 換算後的台幣等值（= txnTwdBaseAmount）；未扣除點數 |

---

## 4. Service UseCase Flow

**UseCase：** `CardPayOrchUseCase.initiate(CardPayOrchRq)`（txn-orch）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[FEIGN]` | **FX 換算：** 若 `currency = TWD` → `txnTwdBaseAmount = amount`；否則呼叫 `FxServiceFeignClient.convert(amount, currency, "TWD")` → `POST /fx-service/convert` → `txnTwdBaseAmount`；見 [fx-service/convert-zh-tw.md](../fx-service/convert-zh-tw.md) |
| 2 | `[DOMAIN]` | `isOverseas = (currency != "TWD")` |
| 3 | `[FEIGN]` | **點數計算：** 若 `usePoints = true` → `WalletFeignClient.getBalance(userId)` → `availableBalance`；`pointsToUse = min(availableBalance, txnTwdBaseAmount)`；否則 `pointsToUse = 0`；見 [wallet-service/reserve-zh-tw.md](../wallet-service/reserve-zh-tw.md) |
| 4 | `[FEIGN]` | `MockNcccFeignClient.pay(MockNcccPayRq)` → `POST /mock-nccc/pay`；Request：`{ cardId, amount, currency, txnTwdBaseAmount, merchantId }`；Response：`{ challengeRef }`；見 [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md) |
| 5 | `[FEIGN]` | **Wallet 預留：** 若 `pointsToUse > 0` → `WalletFeignClient.reserve(userId, pointsToUse)` → `reservationId`；否則 `reservationId = null`；見 [wallet-service/reserve-zh-tw.md](../wallet-service/reserve-zh-tw.md) |
| 6 | `[REDIS WRITE]` | SET `saga:{challengeRef}` = SagaState（TTL 180s）；見 Saga State 章節 |
| 7 | `[RETURN]` | 回傳 `CardPayOrchRs { challengeRef, pointsToUse, estimatedTwdAmount: txnTwdBaseAmount }` |

> **Step 4 失敗**（驗卡失敗、限額超過）→ 直接回傳錯誤；無需清理（Step 5 尚未執行）。
> **Step 5 失敗**（wallet reserve 失敗）→ 回傳錯誤；card-service 的 OTP Session 將自然 TTL 過期（180s）。

---

## 5. Saga State

**Redis Key：** `saga:{challengeRef}`
**TTL：** 180 秒
**寫入：** Step 6（第一腿）
**讀取：** 第二腿 Step 1
**刪除：** 第二腿 — APPROVED 路徑（Step 5）與 DECLINED SESSION_VOIDED / CARD_LOCKED 路徑（Step 8）

```json
{
  "userId": 123456789,
  "cardId": 987654321,
  "amount": 10000,
  "currency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "merchantId": "MERCHANT_001",
  "pointsToUse": 5000,
  "reservationId": 111222333
}
```

| 欄位 | 型別 | 來源 | 說明 |
|------|------|------|------|
| userId | Long | X-Member-Id header | 已驗證的使用者 Snowflake ID |
| cardId | Long | Request | 卡片 Snowflake ID |
| amount | Long | Request | 原始交易金額（分） |
| currency | String | Request | ISO 4217 幣別代碼 |
| txnTwdBaseAmount | Long | Step 1 | FX 換算後的台幣金額（TWD 交易則等於 amount） |
| isOverseas | Boolean | Step 2 | 是否為國外交易 |
| merchantId | String | Request | 商戶識別碼 |
| pointsToUse | Long | Step 3 | 將折抵的點數（分）；`0` 表示不折抵 |
| reservationId | Long | Step 5 | Wallet 預留 Snowflake ID；pointsToUse = 0 時為 `null` |

---

## 6. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 409 | CA00002 | CARD_NOT_ACTIVE | 卡片非 ACTIVE（透傳：card-service → Mock NCCC → ORCH） |
| 409 | CA00003 | CARD_EXPIRED | 卡片已過期（透傳） |
| 409 | CA00004 | CARD_FROZEN | 卡片已凍結（透傳） |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | 超過單筆交易限額（透傳） |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | 超過當日累計限額（透傳） |
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | card-service combineKey 未載入（透傳） |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 7. Changelog

### v1.0 — 2026-03 — 初始版本
