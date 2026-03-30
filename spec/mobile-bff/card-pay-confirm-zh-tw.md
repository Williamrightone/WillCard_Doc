# 刷卡付款確認 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 App 提交的 OTP 確認（**交易流程第二腿**）。
發布 L1 操作日誌，透過 Feign 將 OTP 轉發至 Mock NCCC。
回傳最終交易結果（`APPROVED` 或 `DECLINED`）給 App。

> `challengeRef` 由[第一腿 — card-pay](card-pay-zh-tw.md) 建立；OTP Session 在 **180 秒**後失效，逾時提交將回傳 `DECLINED + SESSION_EXPIRED`。

---

## 2. 跨規格介面契約

此端點為第二腿，以下欄位契約須與第一腿及下游規格保持一致。

| 介面 | 規格 |
|------|------|
| BFF → Transaction Orchestrator 請求／回應 | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| ORCH → Mock NCCC 請求／回應 | [mock-nccc/pay-confirm-zh-tw.md](../mock-nccc/pay-confirm-zh-tw.md) |
| ORCH → Wallet Service（confirmDeduct / release） | [wallet-service/confirm-deduct-zh-tw.md](../wallet-service/confirm-deduct-zh-tw.md) |
| Mock NCCC → card-service auth/verify-challenge | [card-service/card-auth-verify-challenge-zh-tw.md](../card-service/card-auth-verify-challenge-zh-tw.md) |
| 第一腿（建立 challengeRef） | [mobile-bff/card-pay-zh-tw.md](card-pay-zh-tw.md) |

**跨規格欄位契約（所有相關規格必須一致）：**

| 欄位 | 型別 | 流向 | 說明 |
|------|------|------|------|
| `challengeRef` | String | App → BFF → ORCH → MockNCCC → CS | 第一腿產生的 UUID；識別 OTP Session |
| `otp` | String | App → BFF → ORCH → MockNCCC → CS | 使用者輸入的 6 位 OTP |
| `result` | String | CS → MockNCCC → ORCH → BFF → App | `APPROVED` 或 `DECLINED` |
| `txnId` | String | CS → MockNCCC → ORCH → BFF → App | Snowflake txnId；由 card-service 分配；僅 `APPROVED` 時存在 |
| `reason` | String | CS → MockNCCC → ORCH → BFF → App | 拒絕原因碼；僅 `DECLINED` 時存在 |

**拒絕原因值（由 card-service verify-challenge 定義）：**

| reason | 原因 |
|--------|------|
| `SESSION_EXPIRED` | Redis OTP Session 不存在（TTL 已過） |
| `OTP_FAILED` | OTP 錯誤；Session 仍有效；回傳 `remainingAttempts` |
| `SESSION_VOIDED` | 同一 Session 連續錯誤 ≥ 3 次；Session 作廢；Reservation 釋放 |
| `CARD_LOCKED` | 同一卡片 15 分鐘內連續錯誤 ≥ 5 次；卡片 FROZEN；Reservation 釋放 |

---

## 3. 操作日誌等級

**等級：L1**
**觸發：** 每次呼叫，無論成功或失敗。
**Kafka Topic：** `operation-log.card.pay-confirm`

| OperationLogEvent 欄位 | 值來源 |
|------------------------|--------|
| service | `bff` |
| action | `card.pay-confirm` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | 用戶端 IP |
| result | `SUCCESS`（APPROVED）/ `FAIL`（DECLINED 或例外） |
| failReason | 拒絕原因或錯誤說明；APPROVED 時 null |
| errorCode | — |
| beforeSnapshot | — |
| afterSnapshot | `{ "txnId": "...", "challengeRef": "..." }`（僅 APPROVED） |
| requestId | HTTP correlation ID |
| txnId | verify-challenge 回傳的 Snowflake txnId（僅 APPROVED） |

**發布觸發條件：**

| 情境 | result | failReason |
|------|--------|------------|
| 驗證失敗（Step 1） | FAIL | `INVALID_REQUEST` |
| DECLINED（Step 4） | FAIL | 對應 reason 碼 |
| APPROVED（Step 4） | SUCCESS | — |

---

## 4. API 定義 — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| Client URL | `/api/v1/card/pay/confirm` |
| BFF 接收路徑 | `/card/pay/confirm` |
| 需要認證 | 是（Bearer JWT） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body（`bff.contract.dto.CardPayConfirmBffRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| challengeRef | String | ✅ | @NotBlank | 第一腿回傳的 UUID |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 6 位 OTP |

### Response Body（`bff.contract.dto.CardPayConfirmBffRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| result | String | `APPROVED` 或 `DECLINED` |
| txnId | String | Snowflake txnId（僅 result = `APPROVED` 時存在） |
| reason | String | 拒絕原因碼（僅 result = `DECLINED` 時存在）；見第 2 節 |
| remainingAttempts | Integer | OTP 剩餘輸入次數（僅 reason = `OTP_FAILED` 時存在） |

---

## 5. BFF UseCase Flow

**UseCase：** `CardPayConfirmUseCase.confirm(CardPayConfirmBffRq)`（bff）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[VALIDATE]` | 驗證請求欄位（@Valid）；失敗 → throw 400 INVALID_REQUEST |
| 2 | `[HEADER]` | 從 Security Context 取得 `X-Member-Id` |
| 3 | `[KAFKA]` | 發布 `operation-log.card.pay-confirm`（L1）— result 在 Step 4 結果確認後決定 |
| 4 | `[FEIGN]` | `TxnOrchFeignClient.confirm(CardPayConfirmOrchRq)` → `POST /txn-orch/card/pay/confirm`；傳遞 `X-Member-Id` header；見 [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| 5 | `[RETURN]` | 回傳 `CardPayConfirmBffRs { result, txnId?, reason?, remainingAttempts? }` |

> `DECLINED` 業務結果以 **HTTP 200** 回傳，不拋出例外。
> Step 4 系統錯誤（非業務失敗）→ 重新發布 operation-log（result = `FAIL`），再將錯誤回傳給用戶端。

---

## 6. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | — | INVALID_REQUEST | 請求驗證失敗 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

> OTP 相關拒絕（`SESSION_EXPIRED`、`OTP_FAILED`、`SESSION_VOIDED`、`CARD_LOCKED`）以 **HTTP 200**、`result = DECLINED` 回傳——屬業務結果，非錯誤。

---

## 7. Changelog

### v1.0 — 2026-03 — 初始版本
