# 付款確認 — Mock NCCC Spec

> **服務：** mock-nccc
> **環境：** 僅限開發環境
> **Contract Module：** mock-nccc-contract
> **呼叫方：** [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

模擬 NCCC 卡組織在 OTP 確認腿的行為。

接收來自 ORCH 的 `challengeRef` 與 `otp`，直接轉發至 card-service auth/verify-challenge，並將完整結果（包含 `cardType` 與 `maskedPan` 供 ORCH 組裝 Kafka payload）回傳給 ORCH。

此腿為純透傳——不需要加解密或卡片資料查詢。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（ORCH） | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| Mock NCCC → card-service（OTP 驗證） | [card-service/card-auth-verify-challenge-zh-tw.md](../card-service/card-auth-verify-challenge-zh-tw.md) |
| 第一腿 | [mock-nccc/pay-zh-tw.md](pay-zh-tw.md) |

---

## 3. API 定義 — mock-nccc

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /mock-nccc |
| Controller mapping | /pay/confirm |
| 實際接收路徑 | /mock-nccc/pay/confirm |
| 呼叫方 | ORCH（MockNcccFeignClient） |

### Request（`mock-nccc.contract.dto.MockNcccPayConfirmRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| challengeRef | String | ✅ | @NotBlank | 第一腿產生的 UUID；識別 card-service 中的 OTP Session |
| otp | String | ✅ | @NotBlank; @Size(min=6,max=6); @Pattern([0-9]{6}) | 使用者輸入的 6 位 OTP |

### Response（`mock-nccc.contract.dto.MockNcccPayConfirmRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| result | String | `APPROVED` 或 `DECLINED` |
| txnId | String | card-service 分配的 Snowflake txnId（僅 APPROVED） |
| cardType | String | 卡片種類：`CLASSIC` / `OVERSEAS` / `PREMIUM` / `INFINITE`（僅 APPROVED；供 ORCH Kafka payload 使用） |
| maskedPan | String | 遮罩 PAN，例如 `****1234`（僅 APPROVED；供 ORCH Kafka payload 使用） |
| reason | String | 拒絕原因碼（僅 DECLINED）；見下表 |
| remainingAttempts | Integer | OTP 剩餘輸入次數（僅 reason = `OTP_FAILED` 時存在） |

**拒絕原因值（由 card-service verify-challenge 定義）：**

| reason | 原因 |
|--------|------|
| `SESSION_EXPIRED` | Redis OTP Session 不存在（TTL 180s 已過） |
| `OTP_FAILED` | OTP 錯誤；Session 仍有效 |
| `SESSION_VOIDED` | 同一 Session 連續錯誤 ≥ 3 次；Session 作廢 |
| `CARD_LOCKED` | 同一卡片 15 分鐘內連續錯誤 ≥ 5 次；卡片 FROZEN |

---

## 4. Service UseCase Flow

**UseCase：** `MockNcccPayConfirmUseCase.confirm(MockNcccPayConfirmRq)`（mock-nccc）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[FEIGN]` | `CardServiceFeignClient.verifyChallenge(CardAuthVerifyChallengeRq)` → `POST /card-service/auth/verify-challenge`；Request：`{ challengeRef, otp }`；Response：`{ result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }`；見 [card-service/card-auth-verify-challenge-zh-tw.md](../card-service/card-auth-verify-challenge-zh-tw.md) |
| 2 | `[RETURN]` | 回傳 `MockNcccPayConfirmRs { result, txnId?, cardType?, maskedPan?, reason?, remainingAttempts? }` |

> 所有 `APPROVED` / `DECLINED` 結果以 **HTTP 200** 回傳——為 card-service 透傳的業務結果。

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

> OTP 相關結果（`SESSION_EXPIRED`、`OTP_FAILED`、`SESSION_VOIDED`、`CARD_LOCKED`）以 **HTTP 200**、`result = DECLINED` 回傳。

---

## 6. Changelog

### v1.0 — 2026-03 — 初始版本
