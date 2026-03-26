# 刷卡付款確認 — BFF Spec

## 概述

接收會員的 OTP 輸入，並轉發至 Mock NCCC Service 進行最終授權驗證。
核准時回傳授權碼、交易 ID 及點數結算明細；拒絕時回傳拒絕原因代碼。
本端點為兩階段刷卡 Saga 的第二步（最終步）。

---

## Operation Log 等級

**等級：L1**
**觸發時機：** 每次呼叫，無論成功或失敗均發送。
**Kafka Topic：** `operation-log.card.pay-confirm`

| OperationLogEvent 欄位 | 值來源 |
|----------------------|--------|
| service | `bff` |
| action | `card.pay-confirm` |
| userId | 從 JWT 解析 |
| userAccount | — |
| ip | 用戶端 IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗描述；成功為 null |
| errorCode | 失敗錯誤碼；成功為 null |
| beforeSnapshot | — |
| afterSnapshot | — |
| requestId | HTTP 關聯 ID |
| txnId | 核准時為 `CardPayConfirmRs.txnId`；其他情況為 null |

**發送時機：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| JWT 驗證失敗 | FAIL | `UNAUTHORIZED` / — |
| 請求驗證失敗 | FAIL | `INVALID_REQUEST` / — |
| 工作階段已過期（Step 3） | FAIL | `SESSION_EXPIRED` / — |
| OTP 驗證失敗（Step 3） | FAIL | `OTP_FAILED` / — |
| 卡片已鎖定（Step 3） | FAIL | `CARD_LOCKED` / — |
| 付款核准（Step 3） | SUCCESS | — |

---

## API 定義 — BFF

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| 用戶端 URL | `/api/v1/card/pay/confirm` |
| BFF 接收路徑 | `/card/pay/confirm` |
| 是否需要認證 | 是（JWT Bearer Token） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | `Bearer {accessToken}` |

### Request Body

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| challengeRef | String | ✅ | @NotBlank | 由 `/card/pay` 回傳的 OTP 工作階段參考碼 |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 簡訊收到的 6 位數 OTP |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| status | String | `APPROVED` / `DECLINED` |
| authCode | String | 授權碼；`APPROVED` 時存在 |
| txnId | String | Snowflake 交易 ID；`APPROVED` 時存在 |
| pointsUsed | Integer | 實際扣除的點數；`APPROVED` 時存在 |
| cardAmount | BigDecimal | 最終刷卡金額（TWD）；`APPROVED` 時存在 |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED`；`DECLINED` 時存在 |
| attemptsRemaining | Integer | 本次工作階段剩餘 OTP 嘗試次數；`declineReason = OTP_FAILED` 時存在 |

---

## BFF UseCase Flow

**UseCase：** `CardPayConfirmUseCase.confirm(CardPayConfirmRq)`

| Step | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[VALIDATE]` | 驗證 JWT；解析 `X-User-Id` | Token 無效或缺少 → 401 UNAUTHORIZED；發送 L1 log |
| 2 | `[VALIDATE]` | 驗證 Request Body（`challengeRef`、`otp` 格式） | 驗證失敗 → INVALID_REQUEST；發送 L1 log |
| 3 | `[FEIGN]` | 呼叫 Mock NCCC Service → [spec/mock-nccc/pay-confirm.md](../../mock-nccc/pay-confirm.md) | 對映下游 DECLINED 回應至 BFF 錯誤碼；發送 L1 log |
| 4 | `[KAFKA]` | 非同步發送 `operation-log.card.pay-confirm` 事件 | — |
| 5 | `[RETURN]` | 回傳 `{ status, authCode?, txnId?, pointsUsed?, cardAmount?, declineReason?, attemptsRemaining? }` | — |

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 400 | INVALID_REQUEST | 請求格式錯誤或欄位缺失 | `challengeRef` 為空或 `otp` 非 6 位數 |
| 401 | UNAUTHORIZED | 身份驗證失敗 | JWT 無效或缺少 |
| 422 | SESSION_EXPIRED | OTP 工作階段已過期 | Redis key `otp:{challengeRef}` 不存在（TTL 180s） |
| 422 | OTP_FAILED | OTP 驗證失敗 | OTP 輸入錯誤；工作階段仍有效 |
| 422 | CARD_LOCKED | 卡片因連續失敗暫時鎖定 | 每張卡 OTP 失敗門檻（15 分鐘內 5 次）已超過 |
| 500 | INTERNAL_ERROR | 未預期的系統錯誤 | 未處理例外 |
