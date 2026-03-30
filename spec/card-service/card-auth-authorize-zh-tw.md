# 卡片授權 — card-service Spec

> **服務：** card-service
> **Contract Module：** card-service-contract
> **呼叫方：** [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Mock NCCC 組裝的加密卡片資料（`encryptedCard`），使用記憶體內的 `combineKey` 解密，執行驗卡與交易限額核查，產生一次性 OTP，並發布通知事件由 notify-service 透過會員已登記的 OTP 頻道發送。

僅回傳 `{ challengeRef }` 給 Mock NCCC。所有 Saga 協調（Wallet reserve/confirm/release、Kafka 發布）由 Transaction Orchestrator 統一負責。

> **無跨服務呼叫。** card-service 不呼叫 FX Service 或 Wallet Service。`txnTwdBaseAmount` 由 ORCH 預先計算並透過 Mock NCCC 轉發。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（Mock NCCC） | [mock-nccc/pay-zh-tw.md](../mock-nccc/pay-zh-tw.md) |
| 明文卡片資料來源 | [card-service/card-plain-zh-tw.md](card-plain-zh-tw.md) |
| OTP 發送消費者 | [notify-service/otp-delivery-zh-tw.md](../notify-service/otp-delivery-zh-tw.md) |
| OTP 驗證（第二腿） | [card-service/card-auth-verify-challenge-zh-tw.md](card-auth-verify-challenge-zh-tw.md) |
| CombineKey 啟動 | [card-service/combine-key-startup-zh-tw.md](combine-key-startup-zh-tw.md) |

---

## 3. API 定義 — card-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /card-service |
| Controller mapping | /auth/authorize |
| 實際接收路徑 | /card-service/auth/authorize |
| 呼叫方 | Mock NCCC（`CardServiceFeignClient`） |

### Request（`card.service.contract.dto.CardAuthAuthorizeRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| encryptedCard | String | ✅ | @NotBlank | Mock NCCC 以 `MOCK_COMBINE_KEY` 組裝的 AES-256-GCM 加密資料（`{ pan, expiryMMYY, cvvItoken }`） |
| amount | Long | ✅ | @NotNull; @Min(1) | 原始交易金額（分） |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 幣別代碼 |
| txnTwdBaseAmount | Long | ✅ | @NotNull; @Min(1) | 由 ORCH 預先計算的台幣等值；用於限額核查 |
| merchantId | String | ✅ | @NotBlank | 商戶識別碼 |

### Response（`card.service.contract.dto.CardAuthAuthorizeRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| challengeRef | String | 識別 Redis OTP Session 的 UUID；有效期 180 秒 |

---

## 4. Service UseCase Flow

**UseCase：** `CardAuthAuthorizeUseCase.authorize(CardAuthAuthorizeRq)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DOMAIN]` | 從 `CombineKeyHolder` 取得 `combineKey`；null → 拋出 503 `COMBINE_KEY_UNAVAILABLE` |
| 2 | `[DOMAIN]` | AES-256-GCM 以 `combineKey` 解密 `encryptedCard` → `{ pan, expiryMMYY, cvvItoken }`；以 try-finally 保證方法結束時清零 `pan` 位元組陣列 |
| 3 | `[DB READ]` | HMAC-SHA256(`pan`) → `panHash`；`SELECT card WHERE pan_hash = panHash`；找不到 → `CA00001` |
| 4 | `[DOMAIN]` | 比對 `cvvItoken`：received == `card.cvv_itoken`；不符 → `CA00014` |
| 5 | `[DOMAIN]` | 驗證卡片 `status`：僅 `ACTIVE` 通過；`DISABLED/PENDING` → `CA00002`；`FROZEN` → `CA00004` |
| 6 | `[DOMAIN]` | 驗證有效期：解密 `card.expiry_date` → `expiryMMYY`；已過當月 → `CA00003` |
| 7 | `[DOMAIN]` | 單筆限額：若 `card_type != INFINITE` 且 `txnTwdBaseAmount > card_type_limit.single_txn_limit` → `CA00006` |
| 8 | `[REDIS READ]` | GET `card:daily:{cardId}:{yyyyMMdd}`；若 `(currentTotal + txnTwdBaseAmount) > card_type_limit.daily_limit` 且 `daily_limit != null` → `CA00007` |
| 9 | `[DOMAIN]` | 產生 `challengeRef`（UUID v4）+ `otp`（6 位數，`SecureRandom`） |
| 10 | `[MQ PUBLISH]` | 發布至 RabbitMQ exchange `notification`，routing key `otp.send`；payload：`{ userId: card.user_id, otp }`；見 [notify-service/otp-delivery-zh-tw.md](../notify-service/otp-delivery-zh-tw.md) |
| 11 | `[REDIS WRITE]` | SET `otp:{challengeRef}` = OTP Session（見下方）；TTL 180 秒 |
| 12 | `[DOMAIN]` | 清零 `pan` 位元組陣列（由 Step 2 的 try-finally 保證執行） |
| 13 | `[RETURN]` | 回傳 `CardAuthAuthorizeRs { challengeRef }` |

> **Step 4（cvvItoken）：** 查詢以 PAN Hash 解析出的 `card_id` 進行，而非直接以 iToken 查詢。
> 原始 CVV 從不儲存；`cvv_itoken` 為單向 HMAC-SHA256 Token（見 `card_db.md`）。
>
> **Step 12：** Step 2 的 try-finally 保證即使 Step 3–11 拋出例外，PAN 位元組也一定會被清零。

---

## 5. Redis — OTP Session

**Key：** `otp:{challengeRef}`
**TTL：** 180 秒
**寫入：** Step 11（本 UseCase）
**讀取 / 刪除：** `card-service/card-auth-verify-challenge`

```json
{
  "userId": 123456789,
  "cardId": 987654321,
  "txnAmount": 10000,
  "txnCurrency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "otpValue": "123456"
}
```

| 欄位 | 型別 | 來源 | 說明 |
|------|------|------|------|
| userId | Long | `card.user_id` | 卡片持有人 Snowflake ID |
| cardId | Long | `card.card_id` | 卡片 Snowflake ID |
| txnAmount | Long | Request `amount` | 原始交易金額（分） |
| txnCurrency | String | Request `currency` | ISO 4217 幣別代碼 |
| txnTwdBaseAmount | Long | Request `txnTwdBaseAmount` | 台幣等值（由 ORCH 預先計算） |
| isOverseas | Boolean | `currency != "TWD"` | 是否為國外交易 |
| otpValue | String | Step 9 | 6 位 OTP 值；由 verify-challenge 讀取 |

> **注意：** `merchantId`、`pointsToUse`、`reservationId` 屬於 ORCH Saga State 管理，不儲存於此。

---

## 6. Redis — 每日累計計數器（本 UseCase 僅讀取）

**Key：** `card:daily:{cardId}:{yyyyMMdd}`
**本 UseCase 讀取：** Step 8（僅核查，不寫入）
**更新方：** `card-auth-verify-challenge` Step 3（OTP 通過後 INCR txnTwdBaseAmount）

---

## 7. 資料庫

**讀取表：**

| 表 | 欄位 | 用途 |
|----|------|------|
| `card` | `pan_hash`, `cvv_itoken`, `status`, `expiry_date`, `user_id`, `card_id`, `card_type`, `pan_masked` | 卡片查詢與驗證 |
| `card_type_limit` | `single_txn_limit`, `daily_limit` | 交易限額核查（Step 7–8） |

---

## 8. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | 啟動時 `CombineKeyHolder.combineKey` 為 null |
| 404 | CA00001 | CARD_NOT_FOUND | `card` 表中找不到對應 PAN Hash |
| 422 | CA00014 | CVV_ITOKEN_MISMATCH | 收到的 `cvvItoken` 與 `card.cvv_itoken` 不符 |
| 409 | CA00002 | CARD_NOT_ACTIVE | 卡片 `status` 非 `ACTIVE` |
| 409 | CA00003 | CARD_EXPIRED | 卡片有效期已過 |
| 409 | CA00004 | CARD_FROZEN | 卡片 `status` 為 `FROZEN` |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | `txnTwdBaseAmount` 超過 `single_txn_limit` |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | 當日累計加上本次交易將超過 `daily_limit` |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 9. Changelog

### v1.0 — 2026-03 — 初始版本
