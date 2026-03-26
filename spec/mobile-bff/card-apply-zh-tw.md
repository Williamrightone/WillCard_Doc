# 申請虛擬卡 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [card-service/card-apply-zh-tw.md](../card-service/card-apply-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 的虛擬卡申請請求。
驗證請求後轉發至 card-service 執行發卡流程。
成功後回傳新卡資訊，包含**一次性 CVV**。
BFF 不產生卡號亦不處理加密，僅負責流程協調、L1 操作日誌與回應組裝。

> ⚠️ **CVV 注意：** CVV 僅在本次回應中回傳**一次**，不儲存於任何地方，之後無法再查詢。Client 必須立即向用戶顯示並提示妥善保存。

> **不使用 Idempotency-Key：** 本端點不採用標準冪等機制，因為將含有 CVV 的回應快取至 Redis 違反 PCI DSS（Requirement 3.2.1）。重複申請由服務端唯一性驗證防護，Client 應將 `CARD_ALREADY_EXISTS` 視為終止性失敗。

---

## 2. 操作日誌等級

**等級：L1**
**觸發時機：** 每次呼叫，無論成功或失敗。
**Kafka Topic：** `operation-log.card.apply`

| OperationLogEvent 欄位 | 值來源 |
|----------------------|-------|
| service | `bff` |
| action | `card.apply` |
| userId | `X-Member-Id` header |
| userAccount | `X-Member-Account` header |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗時的錯誤說明；成功時為 null |
| errorCode | 失敗時的錯誤碼；成功時為 null |
| beforeSnapshot | — |
| afterSnapshot | `{ "cardId": "...", "cardType": "CLASSIC", "maskedPan": "****-****-****-1234", "status": "ACTIVE" }`（僅 SUCCESS；依 PCI DSS 排除 CVV） |
| requestId | HTTP 關聯 ID |
| txnId | — |

**發布時機：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| 驗證失敗（Step 1） | FAIL | `INVALID_REQUEST` / — |
| card-service 回傳 CARD_ALREADY_EXISTS（Step 2） | FAIL | `CARD_ALREADY_EXISTS` / CA00009 |
| card-service 回傳其他錯誤（Step 2） | FAIL | 來自 card-service 的錯誤碼 |
| 申請成功（Step 3） | SUCCESS | — |

---

## 3. API Definition — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| Client URL | `/api/v1/card/apply` |
| BFF 接收路徑 | `/card/apply` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Request Body

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardType | String | ✅ | @NotBlank；enum：CLASSIC / OVERSEAS / PREMIUM / INFINITE | 申請的卡種 |
| cardNetwork | String | ✅ | @NotBlank；enum：VISA / MASTERCARD / JCB | 卡片網路 |
| cardholderNameEn | String | ✅ | @NotBlank；@Size(max=26) | 英文持卡人姓名（印於卡面）；建議由會員 Profile 預填 |
| cardholderNameZh | String | ✅ | @NotBlank；@Size(max=50) | 中文持卡人姓名；建議由會員 Profile 預填 |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| cardId | String | Snowflake ID（序列化為 String，避免 JavaScript 精度問題） |
| maskedPan | String | 遮罩卡號（`****-****-****-1234`） |
| cardType | String | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| cardNetwork | String | VISA / MASTERCARD / JCB |
| expiryDate | String | 有效期限，`MM/YY` 格式 |
| cardholderNameEn | String | 英文持卡人姓名 |
| cvv | String | 3 位 CVV — **一次性；本回應後無法再取得** |
| status | String | `ACTIVE` |
| activatedAt | String | 啟用時間（ISO-8601） |

---

## 4. BFF UseCase Flow

**UseCase：** `CardApplyUseCase.apply(CardApplyRq)`
**UseCase Impl：** `CardApplyUseCaseImpl`
**Adapter：** `CardFeignAdapter` implements `CardClientPort`
**Feign：** `CardFeignClient`（使用 `card-contract` 的 `CardApplyRq` / `CardApplyRs`）

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[VALIDATE]` | 驗證 cardType 和 cardNetwork 為合法 enum 值；驗證持卡人姓名欄位不為空 | 驗證失敗 → 發布 FAIL 事件（async）→ 拋出 `INVALID_REQUEST` |
| 2 | `[FEIGN]` | `CardFeignAdapter` 呼叫 card-service → [card-service/card-apply-zh-tw.md](../card-service/card-apply-zh-tw.md) | 業務錯誤 → 進入 Step 3；系統錯誤 → 發布 FAIL 事件（async）→ 拋出 `INTERNAL_ERROR` |
| 3 | `[KAFKA]` | **【失敗時】** 發布 FAIL 事件（來自 card-service 的錯誤碼，async）→ 重拋映射後的 BFF 錯誤碼 | 記錄錯誤；不壓制原始例外 |
| 4 | `[KAFKA]` | **【成功時】** 發布 SUCCESS 事件，含 `afterSnapshot`（依 PCI DSS 排除 CVV，async） | 記錄錯誤；不拋出例外 |
| 5 | `[RETURN]` | 組裝 BFF `CardApplyRs`（來自 card-contract `CardApplyRs`）並回傳 | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | cardType / cardNetwork 無效；持卡人姓名為空 |
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 409 | CARD_ALREADY_EXISTS | 已存在有效卡片 | card-service 回傳 CA00009 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
