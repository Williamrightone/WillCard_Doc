# 點數預留 — wallet-service Spec

> **服務：** wallet-service
> **Contract Module：** wallet-contract
> **呼叫方：** [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

為進行中的交易鎖定指定數量的點數。由 Transaction Orchestrator 在付款第一腿（OTP Session 建立後）呼叫。

以 FIFO 方式從最早到期的 `CONFIRMED` 批次中扣除，建立 `wallet_reservation` 標頭與 `wallet_reservation_item` 明細，並以原子操作更新餘額快取。

回傳 `reservationId` 供 ORCH 在第二腿執行 `confirmDeduct` 或 `release`。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（ORCH 第一腿） | [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md) |
| 預留確認 / 釋放 | [wallet-service/confirm-deduct-zh-tw.md](confirm-deduct-zh-tw.md) |

---

## 3. API 定義 — wallet-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /wallet-service |
| Controller mapping | /reservations |
| 實際接收路徑 | /wallet-service/reservations |
| 呼叫方 | ORCH（`WalletFeignClient`） |

### Request（`wallet.contract.dto.WalletReserveRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| userId | Long | ✅ | @NotNull | 會員 Snowflake ID |
| amount | Long | ✅ | @NotNull; @Min(1) | 欲預留的點數（台幣分） |
| idempotencyKey | String | ❌ | @Size(max=64) | 呼叫方提供的去重複鍵（ORCH 傳入 `challengeRef`）。若相同 key 的 PENDING reservation 已存在，直接回傳既有 `reservationId`（冪等成功），不建立新預留。 |

### Response（`wallet.contract.dto.WalletReserveRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| reservationId | Long | 建立的 `wallet_reservation` Snowflake ID；回傳給 ORCH 供第二腿使用 |

---

## 4. Service UseCase Flow

**UseCase：** `WalletReserveUseCase.reserve(WalletReserveRq)`

| 步驟 | 類型 | 說明 |
|------|------|------|
| 0 | `[DB READ]` | **冪等性查詢：** 若 `rq.idempotencyKey` 有值 → SELECT `wallet_reservation` WHERE `idempotency_key = rq.idempotencyKey AND status = 'PENDING'`；存在 → 直接回傳 `{ reservationId }`（冪等成功） |
| 1 | `[DB READ]` | SELECT `wallet_account` WHERE `user_id = rq.userId`；找不到 → `WA00001` |
| 2 | `[DOMAIN]` | 驗證 `available_balance >= rq.amount`；不足 → `WA00002` |
| 3 | `[DB READ]` | SELECT `point_reward_batch` WHERE `user_id = rq.userId AND status = 'CONFIRMED' AND remaining_balance > 0 AND expires_at > NOW()` ORDER BY `expires_at ASC`（FIFO） |
| 4 | `[DOMAIN]` | 以 FIFO 順序遍歷批次，累計扣除直至滿足 `rq.amount`；若所有批次合計不足 → `WA00002`；建立 `allocationList`（batchId, drawAmount）清單 |
| 5 | `[DOMAIN]` | 產生 Snowflake `reservationId` |
| 6 | `[DB WRITE]` | INSERT `wallet_reservation { reservation_id, user_id, total_amount: rq.amount, status: 'PENDING', idempotency_key: rq.idempotencyKey, created_at, updated_at }` |
| 7 | `[DB WRITE]` | INSERT `wallet_reservation_item` — allocationList 中每筆 (batchId, drawAmount) 一列；各自產生 Snowflake `item_id` |
| 8 | `[DB WRITE]` | 每筆 item：UPDATE `point_reward_batch SET remaining_balance -= drawAmount` WHERE `batch_id = item.batchId` |
| 9 | `[DB WRITE]` | UPDATE `wallet_account SET available_balance -= rq.amount, reserved_balance += rq.amount, updated_at = NOW(3)` WHERE `user_id = rq.userId` |
| 10 | `[RETURN]` | 回傳 `WalletReserveRs { reservationId }` |

> **原子性：** Steps 6–9 須在同一資料庫 Transaction 內執行。任何失敗觸發完整 rollback，不留部分狀態。
>
> **FIFO 策略：** 優先從最早到期的批次扣除，避免較早取得的點數因未被使用而過期。
>
> **餘額真實來源：** `point_reward_batch.remaining_balance` 為各批次的權威餘額。`wallet_account.available_balance` 為衍生快取，在 Step 9 原子更新。

---

## 5. 資料庫

**讀取表：**

| 表 | 操作 | 條件 |
|----|------|------|
| `wallet_account` | SELECT | `user_id = rq.userId` |
| `point_reward_batch` | SELECT（FIFO） | `user_id, status='CONFIRMED', remaining_balance>0, expires_at>NOW() ORDER BY expires_at ASC` |

**寫入表（單一 Transaction）：**

| 表 | 操作 | 說明 |
|----|------|------|
| `wallet_reservation` | INSERT | 新預留標頭（`status = PENDING`，儲存 `idempotency_key`） |
| `wallet_reservation_item` | INSERT | FIFO 分配中每個批次一列 |
| `point_reward_batch` | UPDATE | 每個分配批次的 `remaining_balance -= drawAmount` |
| `wallet_account` | UPDATE | `available_balance -= amount`，`reserved_balance += amount` |

---

## 6. 排程清理 — 孤立 PENDING Reservation

當 ORCH Saga State TTL 到期，Leg 1 建立的 `wallet_reservation` 將因 `reservationId` 遺失而永遠無法被 confirm 或 release，造成 `available_balance` 永久偏低。

排程任務規格：

| 欄位 | 值 |
|------|---|
| 觸發週期 | 每 5 分鐘（可設定） |
| 目標資料 | `wallet_reservation` WHERE `status = 'PENDING' AND created_at < NOW() - INTERVAL 10 MINUTE` |
| 執行動作 | 對每筆孤立 reservation 執行與 `release()` 相同邏輯：還原各批次 `remaining_balance`、更新 `wallet_account.available_balance += total_amount, reserved_balance -= total_amount`、將 `status` 設為 `'RELEASED'` |
| 原子性 | 每筆 reservation 使用獨立 DB Transaction |

> **為何設 10 分鐘？** Saga TTL 為 180s（約 3 分鐘）。10 分鐘閾值確保不會因時鐘偏差或處理延遲，誤釋放仍在進行中的交易預留。

---

## 7. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | WA00001 | WALLET_NOT_FOUND | 找不到對應 `userId` 的 `wallet_account` |
| 422 | WA00002 | INSUFFICIENT_BALANCE | `available_balance < amount` 或 CONFIRMED 批次合計不足 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 8. Changelog

### v1.1 — 2026-03 — 新增冪等性 + 排程清理
- Request 新增 `idempotencyKey`（選填）；Step 0 執行冪等查詢。
- 新增 § 排程清理，處理 ORCH Saga State 過期後的孤立 PENDING reservation。
- `wallet_reservation` INSERT 現在儲存 `idempotency_key`。

### v1.0 — 2026-03 — 初始版本
