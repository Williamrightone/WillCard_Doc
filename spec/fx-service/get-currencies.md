# Get Supported Currencies — fx-service Spec

> **Service:** fx-service
> **Contract Module:** fx-contract
> **Caller:** BFF (`FxFeignClient`)
> **Last Updated:** 2026-03

---

## 1. Overview

Returns the list of currencies currently supported by the system, for use by BFF to populate frontend dropdown menus.
The currency list is statically configured and cached in Redis to reduce repeated read overhead.
fx-service is a **stateless Utility Service**: it holds no account data, does not participate in Saga state management, and performs no MySQL writes.

---

## 2. API Definition — fx-service

### Endpoint

| Item | Value |
|------|-------|
| Method | GET |
| context-path | /fx |
| Controller mapping | /currencies |
| Effective path | /fx/currencies |
| Caller | BFF (`FxFeignClient`) |

### Request

No request body or query parameters.

### Response (`fx.contract.dto.FxCurrencyRs`)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| currencies | List\<CurrencyItem\> | Supported currency list | Sorted alphabetically by currency code |

#### CurrencyItem

| Field | Type | Description |
|-------|------|-------------|
| code | String | Currency code (ISO 4217, uppercase, e.g. USD) |
| name | String | Currency name in English (e.g. US Dollar) |
| nameZhTw | String | Currency name in Traditional Chinese (e.g. 美元) |
| symbol | String | Currency symbol (e.g. $, ¥, NT$) |

---

## 3. Service UseCase Flow

**UseCase:** `GetCurrenciesUseCase.getCurrencies()`
**UseCase Impl:** `GetCurrenciesUseCaseImpl`

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[REDIS READ]` | Read `fx:currencies`; cache hit → proceed to step 3 | Cache miss → proceed to step 2 |
| 2 | `[DOMAIN]` | Load supported currency list from application config (`application.yml`), write to `fx:currencies` (TTL 86400s) | Load failure → throw `INTERNAL_ERROR` |
| 3 | `[RETURN]` | Return FxCurrencyRs (currencies list, sorted by code alphabetically) | — |

---

## 4. Database

### MySQL

> fx-service is a stateless Utility Service — **no MySQL reads or writes are performed**.

### Redis

| Key Pattern | Operation | TTL | Description |
|-------------|-----------|-----|-------------|
| `fx:currencies` | READ | — | Read cached supported currency list |

> Currency list is low-frequency change data; TTL is set to 86400s (24 hours).
> Cache is written on first query (or after TTL expiry), sourced from `application.yml` config.

### Redis Value Structure (`fx:currencies`)

Stored as a serialized JSON array; each element has the following structure:

| Field | Type | Description |
|-------|------|-------------|
| code | String | Currency code (ISO 4217) |
| name | String | Currency name in English |
| nameZhTw | String | Currency name in Traditional Chinese |
| symbol | String | Currency symbol |

### Supported Currency List (initial config)

| code | name | nameZhTw | symbol |
|------|------|----------|--------|
| TWD | New Taiwan Dollar | 新台幣 | NT$ |
| USD | US Dollar | 美元 | $ |
| EUR | Euro | 歐元 | € |
| JPY | Japanese Yen | 日圓 | ¥ |
| GBP | British Pound | 英鎊 | £ |
| HKD | Hong Kong Dollar | 港元 | HK$ |
| CNY | Chinese Yuan | 人民幣 | ¥ |
| AUD | Australian Dollar | 澳元 | A$ |
| SGD | Singapore Dollar | 新加坡元 | S$ |
| KRW | South Korean Won | 韓元 | ₩ |

> This list represents the initial configuration. The authoritative source is `fx.supported-currencies` in `application.yml`.

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 500 | INTERNAL_ERROR | System error | Currency list config failed to load or unexpected exception |
