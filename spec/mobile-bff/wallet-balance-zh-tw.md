# 查詢錢包餘額 — BFF Spec

> **服務：** bff
> **對應 biz spec：** [wallet-service/wallet-balance-zh-tw.md](../wallet-service/wallet-balance-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

接收 Client 查詢目前錢包餘額的請求。
轉發至 wallet-service，由其讀取 `wallet_account` 後回傳 `availableBalance` 與 `reservedBalance`。

> **L3 操作：** 唯讀查詢，不發布 Kafka 事件，僅 EFK 記錄。

---

## 2. 操作日誌等級

**等級：L3**
**行為：** 僅 EFK 記錄 — 不發布 Kafka 事件，不寫入 Audit DB。

---

## 3. API Definition — BFF

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | GET |
| Client URL | `/api/v1/wallet/balance` |
| BFF 接收路徑 | `/wallet/balance` |
| 需要認證 | 是（Bearer JWT — Spring Security） |

### Request Header

| Header | 必填 | 說明 |
|--------|------|------|
| Authorization | ✅ | Bearer {accessToken} |

### Response Body

| 欄位 | 型別 | 說明 |
|------|------|------|
| availableBalance | Long | 可用點數餘額（TWD 分） |
| reservedBalance | Long | 進行中授權的鎖定點數（TWD 分） |

---

## 4. BFF UseCase Flow

**UseCase：** `WalletBalanceUseCase.getBalance()`
**UseCase Impl：** `WalletBalanceUseCaseImpl`
**Adapter：** `WalletFeignAdapter` implements `WalletClientPort`
**Feign：** `WalletFeignClient`（使用 `wallet-contract` 的 `WalletBalanceRs`）

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|---------|
| 1 | `[FEIGN]` | `WalletFeignAdapter` 呼叫 wallet-service → [wallet-service/wallet-balance-zh-tw.md](../wallet-service/wallet-balance-zh-tw.md) | 系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 2 | `[RETURN]` | 將 `WalletBalanceRs` 中繼回傳給 Client | — |

---

## 5. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 401 | UNAUTHORIZED | 未認證 | JWT 缺失或無效 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |
