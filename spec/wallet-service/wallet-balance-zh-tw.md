# 查詢錢包餘額 — wallet-service Spec

> **服務：** wallet-service
> **Contract 模組：** wallet-contract
> **呼叫方：** bff → [mobile-bff/wallet-balance-zh-tw.md](../mobile-bff/wallet-balance-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 BFF 透過 Feign 轉發的餘額查詢請求。
讀取請求用戶的 `wallet_account` 記錄，回傳 `availableBalance` 與 `reservedBalance`。

---

## 2. API Definition — wallet-service

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| context-path | `/wallet` |
| Controller mapping | `/balance` |
| 實際接收路徑 | `/wallet/balance` |
| 呼叫方 | BFF（`WalletFeignClient`） |

### Request

無請求 body。用戶身份由 BFF 注入的 `X-Member-Id` header 傳遞。

| Header | 必填 | 說明 |
|--------|------|------|
| X-Member-Id | ✅ | 會員 ID（Long，以 String 傳遞） |

### Response（`wallet.contract.dto.WalletBalanceRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| availableBalance | Long | 可用點數餘額（TWD 分） |
| reservedBalance | Long | 進行中授權的鎖定點數（TWD 分） |

---

## 3. Service UseCase Flow

**UseCase：** `WalletBalanceUseCase.getBalance(Long userId)`（wallet-service）
**UseCase Impl：** `WalletBalanceUseCaseImpl`
**Repository：** `WalletAccountJpaRepository`（Spring Data JPA）

| 步驟 | 類型 | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 查詢 `wallet_account` WHERE `user_id = ?`；不存在 → 拋出 `WALLET_NOT_FOUND`（WA00001） | `WalletQueryService.findByUserId()` |
| 2 | `[RETURN]` | 組裝並回傳 `WalletBalanceRs` | — |

---

## 4. 資料庫

### MySQL

| 資料表 | 操作 | 說明 |
|--------|------|------|
| `wallet_account` | READ | 以 `user_id` 查詢 |

### Redis

> 本端點不進行任何 Redis 操作。

### 資料表 Schema

#### wallet_account（本操作相關欄位）

| 欄位 | 型別 | 說明 |
|------|------|------|
| user_id | BIGINT | PK |
| available_balance | BIGINT | 回傳為 `availableBalance` |
| reserved_balance | BIGINT | 回傳為 `reservedBalance` |

> 完整 Schema：[wallet_db.md](../../database/wallet_db.md)

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 404 | WA00001 | WALLET_NOT_FOUND | 該 `user_id` 無對應的 `wallet_account` 記錄 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
