# 確認扣款 / 釋放預留 — wallet-service Spec

> **服務：** wallet-service
> **Contract Module：** wallet-contract
> **呼叫方：** [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

最終確認由 `reserve` 端點建立的 `PENDING` 預留。兩種結果：

- **Confirm Deduct**（確認扣款）— 交易核准；永久扣除預留點數。更新預留狀態為 `CONFIRMED`，遞減 `reserved_balance`。
- **Release**（釋放）— 交易拒絕或取消；將預留點數還原至原始批次。更新預留狀態為 `RELEASED`，遞增 `available_balance` 並還原各批次 `remaining_balance`。

兩個操作均透過狀態守衛保障安全性：對已完成（`CONFIRMED` 或 `RELEASED`）的預留呼叫任一端點將回傳 `WA00004`。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方 — ORCH 第二腿 APPROVED 路徑 | [txn-orch/card-pay-confirm-zh-tw.md](../txn-orch/card-pay-confirm-zh-tw.md) |
| 預留建立方 | [wallet-service/reserve-zh-tw.md](reserve-zh-tw.md) |

---

## 3. API 定義 — wallet-service

### Endpoint A — 確認扣款（Confirm Deduct）

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations/{reservationId}/confirm |
| 實際接收路徑 | /wallet-service/reservations/{reservationId}/confirm |
| 呼叫方 | ORCH（`WalletFeignClient.confirmDeduct(reservationId)`） |

### Endpoint B — 釋放（Release）

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations/{reservationId}/release |
| 實際接收路徑 | /wallet-service/reservations/{reservationId}/release |
| 呼叫方 | ORCH（`WalletFeignClient.release(reservationId)`） |

### Path Variable（兩個端點共用）

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| reservationId | Long | ✅ | 欲最終確認的 `wallet_reservation` Snowflake ID |

### Response（兩個端點）

成功時 HTTP 200，空 Response Body。

---

## 4. Service UseCase Flow — 確認扣款

**UseCase：** `WalletConfirmDeductUseCase.confirm(Long reservationId)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | SELECT `wallet_reservation` WHERE `reservation_id = reservationId`；找不到 → `WA00003` |
| 2 | `[DOMAIN]` | 驗證 `status == 'PENDING'`；否則 → `WA00004` |
| 3 | `[DB WRITE]` | UPDATE `wallet_reservation SET status = 'CONFIRMED', updated_at = NOW(3)` |
| 4 | `[DB WRITE]` | UPDATE `wallet_account SET reserved_balance -= reservation.total_amount, updated_at = NOW(3)` WHERE `user_id = reservation.user_id` |
| 5 | `[RETURN]` | HTTP 200 |

> **點數已永久扣除** — `available_balance` 在 `reserve()` 時已遞減。Confirm 僅清除 `reserved_balance`。
>
> **不更新批次：** `point_reward_batch.remaining_balance` 在 `reserve()` 時已減少，此處不還原。

---

## 5. Service UseCase Flow — 釋放

**UseCase：** `WalletReleaseUseCase.release(Long reservationId)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | SELECT `wallet_reservation` WHERE `reservation_id = reservationId`；找不到 → `WA00003` |
| 2 | `[DOMAIN]` | 驗證 `status == 'PENDING'`；否則 → `WA00004` |
| 3 | `[DB READ]` | SELECT `wallet_reservation_item` WHERE `reservation_id = reservationId` |
| 4 | `[DB WRITE]` | 每筆 item：UPDATE `point_reward_batch SET remaining_balance += item.amount, updated_at = NOW(3)` WHERE `batch_id = item.batch_id` |
| 5 | `[DB WRITE]` | UPDATE `wallet_reservation SET status = 'RELEASED', updated_at = NOW(3)` |
| 6 | `[DB WRITE]` | UPDATE `wallet_account SET available_balance += reservation.total_amount, reserved_balance -= reservation.total_amount, updated_at = NOW(3)` WHERE `user_id = reservation.user_id` |
| 7 | `[RETURN]` | HTTP 200 |

> **原子性：** Steps 4–6 須在同一資料庫 Transaction 內執行。
>
> **稽核紀錄：** `wallet_reservation_item` 列永不刪除，保留為不可變的稽核紀錄（即使釋放後）。
>
> **批次狀態：** 還原的批次維持 `CONFIRMED` 狀態；`remaining_balance` 僅被遞增還原，不更改批次狀態。

---

## 6. 資料庫

### 確認扣款

| 表 | 操作 | 條件 |
|----|------|------|
| `wallet_reservation` | SELECT | `reservation_id = ?` |
| `wallet_reservation` | UPDATE | `status = 'CONFIRMED'` |
| `wallet_account` | UPDATE | `reserved_balance -= total_amount` |

### 釋放

| 表 | 操作 | 條件 |
|----|------|------|
| `wallet_reservation` | SELECT | `reservation_id = ?` |
| `wallet_reservation_item` | SELECT | `reservation_id = ?` |
| `point_reward_batch` | UPDATE（每筆 item） | `remaining_balance += item.amount` |
| `wallet_reservation` | UPDATE | `status = 'RELEASED'` |
| `wallet_account` | UPDATE | `available_balance += total_amount`，`reserved_balance -= total_amount` |

---

## 7. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | WA00003 | RESERVATION_NOT_FOUND | 找不到對應 `reservationId` 的 `wallet_reservation` |
| 409 | WA00004 | INVALID_RESERVATION_STATUS | 預留狀態非 `PENDING` |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 8. Changelog

### v1.0 — 2026-03 — 初始版本
