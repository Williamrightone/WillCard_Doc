# 刷卡付款 — BFF Spec

## 概述

接收 App 的信用卡付款請求，驗證冪等鍵後，將請求轉發至 Mock NCCC Service。
Phase 1 授權成功時回傳 `challengeRef` 與 `pointsPreview`，讓 App 顯示 OTP 輸入畫面及點數折抵預覽。
本端點為兩階段刷卡 Saga 的第一步。

---

## Operation Log 等級

**等級：L1**
**觸發時機：** 每次呼叫，無論成功或失敗均發送。
**Kafka Topic：** `operation-log.card.pay`

| OperationLogEvent 欄位 | 值來源 |
|----------------------|--------|
| service | `bff` |
| action | `card.pay` |
| userId | 從 JWT 解析（`X-User-Id` header） |
| userAccount | — |
| ip | 用戶端 IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗描述；成功為 null |
| errorCode | 失敗錯誤碼；成功為 null |
| beforeSnapshot | — |
| afterSnapshot | — |
| requestId | HTTP 關聯 ID |
| txnId | — （Phase 2 confirm 步驟才產生） |

**發送時機：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| JWT 驗證失敗（Step 1） | FAIL | `UNAUTHORIZED` / — |
| 冪等衝突（Step 2） | FAIL | `CONFLICT` / — |
| 請求驗證失敗（Step 3） | FAIL | `INVALID_REQUEST` / — |
| 下游服務錯誤（Step 4） | FAIL | 從下游錯誤碼對映 |
| 回傳 CHALLENGE_REQUIRED（Step 4） | SUCCESS | — |

---

## API 定義 — BFF

### 端點

| 欄位 | 值 |
|------|-----|
| Method | POST |
| 用戶端 URL | `/api/v1/card/pay` |
| BFF 接收路徑 | `/card/pay` |
| 是否需要認證 | 是（JWT Bearer Token） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | `Bearer {accessToken}` |
| Idempotency-Key | ✅ | UUID v4，由用戶端產生 |

### Request Body

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| cardId | String | ✅ | @NotBlank | 會員虛擬卡識別碼（Snowflake ID 以字串傳遞） |
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | 交易金額 |
| currency | String | ✅ | @NotBlank, @Size(min=3,max=3) | 幣別代碼，如 `TWD`、`USD` |
| merchantId | String | ✅ | @NotBlank | 商家識別碼 |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| status | String | `CHALLENGE_REQUIRED` / `DECLINED` |
| challengeRef | String | OTP 工作階段參考碼；`status = CHALLENGE_REQUIRED` 時存在 |
| pointsPreview.pointsToUse | Integer | 伺服端計算之點數折抵數（僅供顯示；準備金已於 Phase 1 凍結） |
| pointsPreview.cardAmount | BigDecimal | 折抵後實際刷卡金額（TWD） |
| declineReason | String | 拒絕原因代碼；`status = DECLINED` 時存在 |

---

## BFF UseCase Flow

**UseCase：** `CardPayUseCase.pay(CardPayRq)`

| Step | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[VALIDATE]` | 驗證 JWT；解析並設定 `X-User-Id` header 給下游 | Token 無效或缺少 → 401 UNAUTHORIZED；發送 L1 log |
| 2 | `[REDIS WRITE]` | SETNX `idempotency:{Idempotency-Key}`，狀態為 `PROCESSING`（TTL 60s） | Key 存在且 `COMPLETED` → 回傳快取回應；Key 存在且 `PROCESSING` → 409 CONFLICT；發送 L1 log |
| 3 | `[VALIDATE]` | 驗證 Request Body 欄位 | 驗證失敗 → INVALID_REQUEST；刪除冪等鍵；發送 L1 log |
| 4 | `[FEIGN]` | 呼叫 Mock NCCC Service → [spec/mock-nccc/pay.md](../../mock-nccc/pay.md) | 下游錯誤 → 對映 BFF 錯誤碼；刪除冪等鍵；發送 L1 log |
| 5 | `[KAFKA]` | 非同步發送 `operation-log.card.pay` 事件（失敗不影響主流程） | — |
| 6 | `[REDIS WRITE]` | 更新 `idempotency:{Idempotency-Key}` 為 `COMPLETED`，快取回應（TTL 24h） | — |
| 7 | `[RETURN]` | 回傳 `{ status, challengeRef?, pointsPreview?, declineReason? }` | — |

---

## 錯誤碼

| HTTP Status | 錯誤碼 | 說明 | 觸發條件 |
|-------------|--------|------|----------|
| 400 | INVALID_REQUEST | 請求格式錯誤或欄位缺失 | Request Body 驗證失敗 |
| 401 | UNAUTHORIZED | 身份驗證失敗 | JWT 無效或缺少 |
| 409 | CONFLICT | 重複進行中的請求 | `Idempotency-Key` 狀態為 `PROCESSING` |
| 422 | CARD_NOT_FOUND | 卡片不存在 | Card Service CA00001 |
| 422 | CARD_NOT_ACTIVE | 卡片非 ACTIVE 狀態 | Card Service CA00002 |
| 422 | CARD_EXPIRED | 卡片已過期 | Card Service CA00003 |
| 422 | CARD_FROZEN | 卡片已凍結 | Card Service CA00004 |
| 422 | CARD_CANCELLED | 卡片已註銷 | Card Service CA00005 |
| 422 | SINGLE_TXN_LIMIT_EXCEEDED | 超過單筆交易上限 | Card Service CA00006 |
| 422 | DAILY_LIMIT_EXCEEDED | 超過單日累計上限 | Card Service CA00007 |
| 422 | INSUFFICIENT_FUNDS | 錢包餘額不足（點數折抵後） | Wallet Service 錯誤 |
| 500 | INTERNAL_ERROR | 未預期的系統錯誤 | 未處理例外 |
