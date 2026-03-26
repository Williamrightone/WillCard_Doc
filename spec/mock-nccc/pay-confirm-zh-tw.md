# Mock NCCC 刷卡確認 — Biz Spec

## 概述

**Mock NCCC 為獨立的 Spring Boot 微服務**（Maven module：`mock-nccc`，
僅部署於開發環境的 Kubernetes cluster）。本端點為 Mock NCCC 付款模擬的第二步。
接收 App（透過 BFF Feign）提交的 OTP 及 `challengeRef`，直接轉送至 Card Service
發卡機構授權 API Phase 2，並透傳最終 `APPROVED` 或 `DECLINED` 結果。

> **適用範圍：** 僅限開發與整合測試環境，不得部署至正式環境。

---

## API 定義 — mock-nccc

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| context-path | `/mock-nccc` |
| Controller mapping | `/pay/confirm` |
| 實際接收路徑 | `/mock-nccc/pay/confirm` |
| 呼叫方 | BFF（FeignClient） |

### Request（MockNcccPayConfirmRq — mock-nccc-contract）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| challengeRef | String | ✅ | @NotBlank | 由 `/mock-nccc/pay` 回傳的 OTP 工作階段參考碼 |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 持卡人提交的 6 位數 OTP |

### Response（MockNcccPayConfirmRs — mock-nccc-contract）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| status | String | `APPROVED` / `DECLINED` | |
| authCode | String | 授權碼 | `APPROVED` 時存在 |
| txnId | String | Snowflake 交易 ID | `APPROVED` 時存在 |
| pointsUsed | Integer | 實際扣除的點數 | `APPROVED` 時存在 |
| cardAmount | BigDecimal | 最終刷卡金額（TWD） | `APPROVED` 時存在 |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED` | `DECLINED` 時存在 |
| attemptsRemaining | Integer | 本次工作階段剩餘嘗試次數 | `declineReason = OTP_FAILED` 時存在 |

---

## Service UseCase Flow

**UseCase：** `MockNcccPayConfirmUseCase.confirm(MockNcccPayConfirmRq)`

| Step | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[FEIGN]` | 呼叫 Card Service `POST /card/auth/verify-challenge`，帶入 `{ challengeRef, otp }` → [spec/card-service/card-auth-verify-challenge.md](../card-service/card-auth-verify-challenge.md) | — |
| 2 | `[RETURN]` | 將 `CardAuthVerifyChallengeRs` 直接作為 `MockNcccPayConfirmRs` 回傳 | — |

---

## 資料庫

### MySQL

_（無 — Mock NCCC 為無狀態服務）_

### Redis

_（無）_

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 422 | `SESSION_EXPIRED` | OTP 工作階段已過期 | 從 Card Service 透傳 |
| 422 | `OTP_FAILED` | OTP 驗證失敗 | 從 Card Service 透傳 |
| 422 | `CARD_LOCKED` | 卡片因連續失敗暫時鎖定 | 從 Card Service 透傳 |
| 500 | `INTERNAL_ERROR` | 未預期的系統錯誤 | 未處理例外 |
