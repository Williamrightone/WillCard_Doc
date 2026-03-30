# OTP 發送 — notify-service Spec

> **服務：** notify-service
> **元件類型：** RabbitMQ Consumer + OTP 通知策略
> **訊息來源：** [card-service/card-auth-authorize-zh-tw.md](../card-service/card-auth-authorize-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

消費 card-service 在交易授權期間發布的 `notification.otp.send` 訊息，從 auth-service 查詢會員已啟用的 OTP 發送頻道，並以對應策略完成 OTP 發送。

OTP 發送透過**策略模式**（`OtpNotificationPort` 介面）實作，支援多種頻道而無需修改消費者邏輯。本階段：Telegram 完整實作；SMS 僅建立 stub 介面，不實作。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 訊息發布方 | [card-service/card-auth-authorize-zh-tw.md](../card-service/card-auth-authorize-zh-tw.md) |
| 會員 OTP 頻道查詢 | auth-service 內部端點 `GET /auth-service/internal/members/{memberId}/otp-channels`（規格：待補） |

---

## 3. 訊息消費者定義

### RabbitMQ Binding

| 欄位 | 值 |
|------|---|
| Exchange | `notification`（topic） |
| Routing Key | `otp.send` |
| Queue | `notify.otp` |
| Consumer class | `notify.service.consumer.OtpNotificationConsumer` |
| Ack 模式 | Manual（成功發送後 ack；可重試失敗時 nack with requeue） |

### 訊息 Payload（`notify.service.consumer.dto.OtpNotificationMessage`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| userId | Long | 卡片持有人 Snowflake ID；用於查詢 OTP 頻道 |
| otp | String | 6 位 OTP；**不得記錄於 log** |

---

## 4. OTP 通知策略設計

### 介面

```
OtpNotificationPort
  ─ void send(OtpDeliveryCommand command)   // 失敗時拋出 OtpDeliveryException
  ─ ChannelType channelType()               // 回傳此 Adapter 負責的頻道類型
```

```
OtpDeliveryCommand
  ─ String contactValue    // Telegram chat_id 或 E.164 手機號碼
  ─ String maskedValue     // log 安全的遮罩顯示值
  ─ String otp             // 6 位 OTP；不得記錄於 log
```

### Adapter 清單

| Adapter | 頻道類型 | 狀態 | 行為 |
|---------|---------|------|------|
| `TelegramOtpAdapter` | `TELEGRAM` | **已實作** | 以 `contactValue` 作為 `chat_id` 呼叫 Telegram Bot API 發送訊息；範本：`您的 WillCard 交易驗證碼為 {otp}，有效期限 3 分鐘，請勿告知他人。` |
| `SmsOtpAdapter` | `SMS` | **僅 Stub** | 記錄一筆 `WARN` log（`"SMS OTP delivery not implemented — userId: {userId}, masked: {maskedValue}"`）後直接回傳，不發送，不拋出例外。 |

### 策略 Context

`OtpNotificationContext` 持有所有以 `List<OtpNotificationPort>` 注入的實作。發送時依會員頻道的 `channelType()` 選取對應 Adapter。

若無對應 Adapter（例如 DB 中的頻道類型尚未登記 Adapter），退回 `SmsOtpAdapter` 行為（記錄 WARN，不發送）。

---

## 5. Service UseCase Flow

**UseCase：** `OtpNotificationConsumer.consume(OtpNotificationMessage)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[FEIGN]` | `AuthServiceFeignClient.getOtpChannels(userId)` → `GET /auth-service/internal/members/{userId}/otp-channels?status=ACTIVE`；Response：`[{ channelType, contactValue, maskedValue, isPrimary }]`（auth-service 內部規格待補） |
| 2 | `[DOMAIN]` | 從結果中選取 `isPrimary = true` 的第一筆頻道；若結果為空 → 記錄 WARN，跳過發送（會員尚未設定頻道） |
| 3 | `[DOMAIN]` | 透過 `OtpNotificationContext.resolve(channelType)` 取得對應 Adapter |
| 4 | `[DOMAIN]` | 組裝 `OtpDeliveryCommand { contactValue, maskedValue, otp }` |
| 5 | `[DOMAIN]` | `adapter.send(command)`；發生 `OtpDeliveryException` → NACK with requeue（可重試）；SMS stub → 無操作 |
| 6 | `[RETURN]` | ACK 訊息 |

> **OTP 安全性：** `otp` 不得出現於任何 log 語句。稽核紀錄僅記錄 `maskedValue`。
> **無 DB 寫入：** 本階段 notify-service 不持久化 OTP 發送紀錄。

---

## 6. Telegram Adapter — 設定

| 設定鍵 | 說明 |
|--------|------|
| `willcard.notify.telegram.bot-token` | Telegram Bot API Token；從 env/KMS 注入 |
| `willcard.notify.telegram.api-url` | Telegram Bot API 基礎 URL（預設：`https://api.telegram.org`） |

**Telegram Bot API 呼叫：**

```
POST https://api.telegram.org/bot{botToken}/sendMessage
Body: { "chat_id": "{contactValue}", "text": "您的 WillCard 交易驗證碼為 {otp}，有效期限 3 分鐘，請勿告知他人。" }
```

---

## 7. 錯誤處理

| 情況 | 行為 |
|------|------|
| auth-service Feign 呼叫失敗 | NACK with requeue；依 RabbitMQ Dead Letter 策略重試 |
| Telegram API 呼叫失敗（網路 / 4xx / 5xx） | NACK with requeue |
| 會員未登記任何頻道 | 記錄 WARN；ACK（不可重試；使用者尚未設定頻道） |
| 選到 SMS 頻道 | 記錄 WARN（stub）；ACK（不可重試；設計上未實作） |

---

## 8. Changelog

### v1.0 — 2026-03 — 初始版本；Telegram 實作，SMS 為 Stub
