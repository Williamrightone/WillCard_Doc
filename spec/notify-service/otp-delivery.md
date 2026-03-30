# OTP Delivery — notify-service Spec

> **Service:** notify-service
> **Component Type:** RabbitMQ Consumer + OTP Notification Strategy
> **Message Source:** [card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Consumes `notification.otp.send` messages published by card-service during transaction authorization, looks up the member's active OTP delivery channels from auth-service, and dispatches the OTP using the appropriate strategy.

OTP delivery is implemented via a **Strategy Pattern** (`OtpNotificationPort` interface), enabling multiple delivery channels without modifying the consumer logic. Current phase: Telegram is fully implemented; SMS is a stub interface only.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Message publisher | [card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md) |
| Member OTP channel lookup | auth-service internal endpoint `GET /auth-service/internal/members/{memberId}/otp-channels` *(spec: TBD)* |

---

## 3. Message Consumer Definition

### RabbitMQ Binding

| Field | Value |
|-------|-------|
| Exchange | `notification` (topic) |
| Routing Key | `otp.send` |
| Queue | `notify.otp` |
| Consumer class | `notify.service.consumer.OtpNotificationConsumer` |
| Ack mode | Manual (ack after successful dispatch; nack with requeue on retriable failure) |

### Message Payload (`notify.service.consumer.dto.OtpNotificationMessage`)

| Field | Type | Description |
|-------|------|-------------|
| userId | Long | Card owner member Snowflake ID; used to look up OTP channels |
| otp | String | 6-digit OTP to deliver; **must not be logged** |

---

## 4. OTP Notification Strategy Design

### Interface

```
OtpNotificationPort
  ─ void send(OtpDeliveryCommand command)   // throws OtpDeliveryException on failure
  ─ ChannelType channelType()               // returns the channel type this adapter handles
```

```
OtpDeliveryCommand
  ─ String contactValue    // Telegram chat_id or E.164 phone number
  ─ String maskedValue     // log-safe display value
  ─ String otp             // 6-digit OTP; never logged
```

### Adapters

| Adapter | Channel Type | Status | Behavior |
|---------|-------------|--------|----------|
| `TelegramOtpAdapter` | `TELEGRAM` | **Implemented** | Sends message via Telegram Bot API using `contactValue` as `chat_id`; message template: `您的 WillCard 交易驗證碼為 {otp}，有效期限 3 分鐘，請勿告知他人。` |
| `SmsOtpAdapter` | `SMS` | **Stub only** | Logs a `WARN`-level entry (`"SMS OTP delivery not implemented — userId: {userId}, masked: {maskedValue}"`) and returns without sending. Does not throw. |

### Strategy Context

`OtpNotificationContext` holds all `OtpNotificationPort` implementations injected as a `List<OtpNotificationPort>` bean. On dispatch, it selects the adapter whose `channelType()` matches the member's active channel.

If no matching adapter is found (e.g., channel type in DB has no registered adapter), it falls back to `SmsOtpAdapter` behavior (WARN log, no send).

---

## 5. Service UseCase Flow

**UseCase:** `OtpNotificationConsumer.consume(OtpNotificationMessage)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[FEIGN]` | `AuthServiceFeignClient.getOtpChannels(userId)` → `GET /auth-service/internal/members/{userId}/otp-channels?status=ACTIVE`; response: list of `{ channelType, contactValue, maskedValue, isPrimary }` *(auth-service internal spec TBD)* |
| 2 | `[DOMAIN]` | Select primary active channel from result (filter `isPrimary = true`, first match); if empty list → log WARN and skip delivery (no channel registered) |
| 3 | `[DOMAIN]` | Resolve adapter via `OtpNotificationContext.resolve(channelType)` |
| 4 | `[DOMAIN]` | Build `OtpDeliveryCommand { contactValue, maskedValue, otp }` |
| 5 | `[DOMAIN]` | `adapter.send(command)`; on `OtpDeliveryException` → NACK with requeue (retriable); on stub (SMS) → no-op |
| 6 | `[RETURN]` | ACK message |

> **OTP security:** `otp` must never appear in any log statement. Log only `maskedValue` for audit trail.
> **No DB writes:** notify-service does not persist OTP delivery records in this phase.

---

## 6. Telegram Adapter — Configuration

| Config Key | Description |
|-----------|-------------|
| `willcard.notify.telegram.bot-token` | Telegram Bot API token; injected from env/KMS |
| `willcard.notify.telegram.api-url` | Telegram Bot API base URL (default: `https://api.telegram.org`) |

**Telegram Bot API call:**

```
POST https://api.telegram.org/bot{botToken}/sendMessage
Body: { "chat_id": "{contactValue}", "text": "您的 WillCard 交易驗證碼為 {otp}，有效期限 3 分鐘，請勿告知他人。" }
```

---

## 7. Error Handling

| Condition | Behavior |
|-----------|----------|
| auth-service Feign call fails | NACK with requeue; retry up to RabbitMQ dead-letter policy |
| Telegram API call fails (network / 4xx / 5xx) | NACK with requeue |
| No channel registered for member | WARN log; ACK (non-retriable; user has no channel configured) |
| SMS channel selected | WARN log (stub); ACK (non-retriable; by design not implemented) |

---

## 8. Changelog

### v1.0 — 2026-03 — Initial spec; Telegram implemented, SMS stub
