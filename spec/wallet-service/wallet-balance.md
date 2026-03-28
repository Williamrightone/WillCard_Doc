# Get Wallet Balance — wallet-service Spec

> **Service:** wallet-service
> **Contract module:** wallet-contract
> **Caller:** bff → [mobile-bff/wallet-balance.md](../mobile-bff/wallet-balance.md)
> **Last updated:** 2026-03

---

## 1. Overview

Receives the balance query forwarded by BFF via Feign.
Reads the `wallet_account` record for the requesting user and returns `availableBalance` and `reservedBalance`.

---

## 2. API Definition — wallet-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| context-path | `/wallet` |
| Controller mapping | `/balance` |
| Full path | `/wallet/balance` |
| Caller | BFF (`WalletFeignClient`) |

### Request

No request body. User identity is passed via the `X-Member-Id` header injected by BFF.

| Header | Required | Description |
|--------|----------|-------------|
| X-Member-Id | ✅ | Member ID (Long, as String) |

### Response (`wallet.contract.dto.WalletBalanceRs`)

| Field | Type | Description |
|-------|------|-------------|
| availableBalance | Long | Points available for use (TWD cents) |
| reservedBalance | Long | Points locked for in-flight authorizations (TWD cents) |

---

## 3. Service UseCase Flow

**UseCase:** `WalletBalanceUseCase.getBalance(Long userId)` (wallet-service)
**UseCase Impl:** `WalletBalanceUseCaseImpl`
**Repository:** `WalletAccountJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query `wallet_account` WHERE `user_id = ?`; not found → throw `WALLET_NOT_FOUND` (WA00001) | `WalletQueryService.findByUserId()` |
| 2 | `[RETURN]` | Assemble and return `WalletBalanceRs` | — |

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `wallet_account` | READ | Query by `user_id` |

### Redis

> No Redis operations for this endpoint.

### Table Schema

#### wallet_account (relevant columns)

| Column | Type | Description |
|--------|------|-------------|
| user_id | BIGINT | PK |
| available_balance | BIGINT | Returned as `availableBalance` |
| reserved_balance | BIGINT | Returned as `reservedBalance` |

> Full schema: [wallet_db.md](../../database/wallet_db.md)

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 404 | WA00001 | WALLET_NOT_FOUND | No `wallet_account` record for this `user_id` |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
