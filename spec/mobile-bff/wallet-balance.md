# Get Wallet Balance — BFF Spec

> **Service:** bff
> **Biz spec:** [wallet-service/wallet-balance.md](../wallet-service/wallet-balance.md)
> **Last updated:** 2026-03

---

## 1. Overview

Receives the client's request to query the current wallet balance.
Forwards to wallet-service, which reads `wallet_account` and returns `availableBalance` and `reservedBalance`.

> **Operation level:** L3 — read-only query; EFK log only, no Kafka event.

---

## 2. Operation Log Level

**Level: L3**
**Behavior:** EFK log only — no Kafka event, no Audit DB write.

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| Client URL | `/api/v1/wallet/balance` |
| BFF receives | `/wallet/balance` |
| Auth required | Yes (Bearer JWT — Spring Security) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | ✅ | Bearer {accessToken} |

### Response Body

| Field | Type | Description |
|-------|------|-------------|
| availableBalance | Long | Points available for use (TWD cents) |
| reservedBalance | Long | Points locked for in-flight authorizations (TWD cents) |

---

## 4. BFF UseCase Flow

**UseCase:** `WalletBalanceUseCase.getBalance()`
**UseCase Impl:** `WalletBalanceUseCaseImpl`
**Adapter:** `WalletFeignAdapter` implements `WalletClientPort`
**Feign:** `WalletFeignClient` (uses `wallet-contract`'s `WalletBalanceRs`)

| Step | Type | Description | Failure Handling |
|------|------|-------------|-----------------|
| 1 | `[FEIGN]` | `WalletFeignAdapter` calls wallet-service → [wallet-service/wallet-balance.md](../wallet-service/wallet-balance.md) | System error → throw `INTERNAL_ERROR` |
| 2 | `[RETURN]` | Relay `WalletBalanceRs` to client | — |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 401 | UNAUTHORIZED | Not authenticated | JWT missing or invalid |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
