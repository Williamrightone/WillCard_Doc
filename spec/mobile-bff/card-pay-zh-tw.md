# 刷卡付款 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 App 發起的刷卡付款請求（**交易流程第一腿**）。
驗證請求、執行冪等控制、發布 L1 操作日誌，然後透過 Feign 轉發至 Mock NCCC。
回傳 `challengeRef` 及預估金額給 App；App 須在 **180 秒**內呼叫 `POST /api/v1/card/pay/confirm` 完成交易。

---

## 2. 跨規格介面契約

此端點啟動跨多個服務的交易鏈。以下規格必須使用相同的欄位名稱與型別。

| 介面 | 規格 |
|------|------|
| BFF → Transaction Orchestrator 請求／回應 | [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md) |
| ORCH → Mock NCCC 請求／回應 | [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md) |
| ORCH → FX Service | [fx-service/convert-zh-tw.md](../fx-service/convert-zh-tw.md) |
| ORCH → Wallet Service（reserve） | [wallet-service/reserve-zh-tw.md](../wallet-service/reserve-zh-tw.md) |
| Mock NCCC → card-service 內部明文端點 | [card-service/card-plain-zh-tw.md](../card-service/card-plain-zh-tw.md) |
| Mock NCCC → card-service auth/authorize | [card-service/card-auth-authorize-zh-tw.md](../card-service/card-auth-authorize-zh-tw.md) |
| 第二腿 OTP 確認 | [mobile-bff/card-pay-confirm-zh-tw.md](card-pay-confirm-zh-tw.md) |

**跨規格欄位契約（所有相關規格必須一致）：**

| 欄位 | 型別 | 流向 | 說明 |
|------|------|------|------|
| `cardId` | String | App → BFF → ORCH → MockNCCC | Snowflake ID 序列化為 String |
| `amount` | Long | App → BFF → ORCH → MockNCCC → CS | 交易金額（分） |
| `currency` | String | App → BFF → ORCH → MockNCCC → CS | ISO 4217（例如 `TWD`、`USD`） |
| `merchantId` | String | App → BFF → ORCH → MockNCCC → CS | 商戶識別碼 |
| `usePoints` | Boolean | App → BFF → ORCH | 由 ORCH 處理；不轉發至 MockNCCC |
| `txnTwdBaseAmount` | Long | ORCH → MockNCCC → CS | FX 換算後的台幣金額；由 ORCH 計算，透過 MockNCCC 傳入 CS |
| `challengeRef` | String | CS → MockNCCC → ORCH → BFF → App | UUID；OTP Session 綁定；TTL 180s |
| `pointsToUse` | Long | ORCH → BFF → App | 由 ORCH 計算（min(wallet餘額, txnTwdBaseAmount)）；不折抵時為 `0` |
| `estimatedTwdAmount` | Long | ORCH → BFF → App | = `txnTwdBaseAmount`；FX 換算後的台幣等值金額 |

---

## 3. 操作日誌等級

**等級：L1**
**觸發：** 每次呼叫，無論成功或失敗。
**Kafka Topic：** `operation-log.card.pay`

| OperationLogEvent 欄位 | 值來源 |
|------------------------|--------|
| service | `bff` |
| action | `card.pay` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | 用戶端 IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗說明；成功時 null |
| errorCode | 失敗錯誤碼；成功時 null |
| beforeSnapshot | — |
| afterSnapshot | `{ "cardId": "...", "amount": 10000, "currency": "USD", "merchantId": "...", "usePoints": true }`（僅 SUCCESS） |
| requestId | HTTP correlation ID |
| txnId | —（第一腿尚未分配 txnId） |

**發布觸發條件：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| 驗證失敗（Step 1） | FAIL | `INVALID_REQUEST` / — |
| 冪等衝突（Step 3） | FAIL | `DUPLICATE_REQUEST` / — |
| 下游回傳 CA000xx（Step 6） | FAIL | 來自下游 errorCode |
| 發起成功（Step 7） | SUCCESS | — |

---

## 4. API 定義 — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| Client URL | `/api/v1/card/pay` |
| BFF 接收路徑 | `/card/pay` |
| 需要認證 | 是（Bearer JWT） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |
| Idempotency-Key | ✅ | 由用戶端產生的 UUID v4；60 秒內防重複提交 |

### Request Body（`bff.contract.dto.CardPayBffRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 扣款卡片（Snowflake ID as String） |
| amount | Long | ✅ | @NotNull; @Min(1) | 交易金額（分） |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 幣別代碼 |
| merchantId | String | ✅ | @NotBlank | 商戶識別碼 |
| usePoints | Boolean | ✅ | @NotNull | 是否套用點數折抵 |

### Response Body（`bff.contract.dto.CardPayBffRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| challengeRef | String | UUID；傳入 `/card/pay/confirm`；有效期 180s |
| pointsToUse | Long | 將折抵的點數（分）；不折抵時為 `0` |
| estimatedTwdAmount | Long | 台幣等值（外幣已換算；未扣除點數） |

---

## 5. BFF UseCase Flow

**UseCase：** `CardPayUseCase.initiate(CardPayBffRq)`（bff）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[VALIDATE]` | 驗證請求欄位（@Valid）；失敗 → throw 400 INVALID_REQUEST |
| 2 | `[HEADER]` | 從 Security Context 取得 `X-Member-Id` |
| 3 | `[REDIS READ]` | GET `bff:pay:idem:{userId}:{idempotencyKey}`；Key 存在 → throw 409 DUPLICATE_REQUEST |
| 4 | `[REDIS WRITE]` | SET `bff:pay:idem:{userId}:{idempotencyKey}` = `1` EX 60 |
| 5 | `[KAFKA]` | 發布 `operation-log.card.pay`（L1）— result 在 Step 6 結果確認後決定 |
| 6 | `[FEIGN]` | `TxnOrchFeignClient.pay(CardPayOrchRq)` → `POST /txn-orch/card/pay`；傳遞 `X-Member-Id` header；見 [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md) |
| 7 | `[RETURN]` | 回傳 `CardPayBffRs { challengeRef, pointsToUse, estimatedTwdAmount }` |

> Step 6 拋出例外時，重新發布 operation-log（result = `FAIL`），再將錯誤回傳給用戶端。

---

## 6. Redis Key

| Key | TTL | 用途 |
|-----|-----|------|
| `bff:pay:idem:{userId}:{idempotencyKey}` | 60s | 冪等保護；防止重複發起付款 |

---

## 7. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | — | INVALID_REQUEST | 請求驗證失敗 |
| 409 | — | DUPLICATE_REQUEST | Idempotency-Key 於 60s 內重複提交 |
| 409 | CA00002 | CARD_NOT_ACTIVE | 卡片非 ACTIVE（透傳：card-service → MockNCCC → ORCH → BFF） |
| 409 | CA00003 | CARD_EXPIRED | 卡片已過期 |
| 409 | CA00004 | CARD_FROZEN | 卡片已凍結 |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | 超過單筆交易限額 |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | 超過當日累計限額 |
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | card-service combineKey 未載入 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 8. Changelog

### v1.0 — 2026-03 — 初始版本
