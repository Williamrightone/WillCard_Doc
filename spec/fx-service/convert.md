# Convert Amount — fx-service Spec

> **Service:** fx-service
> **Contract Module:** fx-contract
> **Caller:** Transaction Orchestrator
> **Last Updated:** 2026-03

---

## 1. Overview

Accepts a transaction amount and currency pair, retrieves the current exchange rate, and returns the converted amount.
This API is called by the Transaction Orchestrator during Saga execution to calculate the settlement amount for foreign currency transactions (e.g., credit card cross-border purchases).
fx-service is a **stateless Utility Service**: it holds no account data, does not participate in Saga state management, and performs no MySQL writes.
A live rate strategy is used (no rate locking) — each call uses the current cached rate, consistent with traditional credit card behavior.

---

## 2. API Definition — fx-service

### Endpoint

| Item | Value |
|------|-------|
| Method | POST |
| context-path | /fx |
| Controller mapping | /convert |
| Effective path | /fx/convert |
| Caller | Transaction Orchestrator |

### Request (`fx.contract.dto.FxConvertRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | Amount to convert (in source currency) |
| fromCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | Source currency (ISO 4217, uppercase, e.g. USD) |
| toCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | Target currency (ISO 4217, uppercase, e.g. TWD) |

### Response (`fx.contract.dto.FxConvertRs`)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| originalAmount | BigDecimal | Original amount (source currency) | |
| fromCurrency | String | Source currency (ISO 4217) | |
| convertedAmount | BigDecimal | Converted amount (target currency) | Rounded to 2 decimal places |
| toCurrency | String | Target currency (ISO 4217) | |
| rate | BigDecimal | Applied exchange rate | 10 decimal places |
| rateTimestamp | Long | Rate data timestamp (epoch millis) | For Orchestrator to record in transaction log |

---

## 3. Service UseCase Flow

**UseCase:** `ConvertUseCase.convert(FxConvertRq)`
**UseCase Impl:** `ConvertUseCaseImpl`

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[DOMAIN]` | Validate that `fromCurrency` and `toCurrency` are in the supported currency list (`CurrencyService.getCurrencies()`) | Unsupported currency → throw `UNSUPPORTED_CURRENCY` |
| 2 | `[REDIS READ]` | Read `fx:raw-rates`; cache hit → proceed to step 4 | Cache miss → proceed to step 3 |
| 3 | `[DOMAIN]` | Call `BotExchangeRateClient.fetchParsedData()` to fetch and parse the full BOT CSV; write the raw map to Redis via `FxRatePort.cacheRawRates()` (TTL 10800s) | Bank API failure or unexpected exception → throw `RATE_UNAVAILABLE` |
| 4 | `[DOMAIN]` | Compute rate via `BotExchangeRateClient.computeRate(from, to, rawRates)` applying the calculation strategy in the get-rate spec §3.2 | Computation failure (e.g., missing currency in map) → throw `RATE_UNAVAILABLE` |
| 5 | `[DOMAIN]` | Calculate converted amount: `convertedAmount = amount × rate`, rounded to 2 decimal places (`FxCalculationService.convert()`) | Calculation error → throw `INTERNAL_ERROR` |
| 6 | `[RETURN]` | Return FxConvertRs (originalAmount, fromCurrency, convertedAmount, toCurrency, rate, rateTimestamp) | — |

---

## 4. Database

### MySQL

> fx-service is a stateless Utility Service — **no MySQL reads or writes are performed**.

### Redis

| Key | Operation | TTL | Description |
|-----|-----------|-----|-------------|
| `fx:raw-rates` | READ | — | Entire BOT raw rate map (all supported currencies), stored as one JSON blob |

> **Rate cache writes are handled by the scheduled task (FX Rate Scheduler).** This API triggers an on-demand update via `FxRatePort.cacheRawRates()` only on a cache miss.

### Redis Value Structure (`fx:raw-rates`)

Stored as a JSON serialization of `Map<String, BotRateRow>`, where each key is the ISO 4217 currency code and the value is a `BotRateRow` object:

| Field | Type | Description |
|-------|------|-------------|
| `{currencyCode}` (map key) | String | Foreign currency code (e.g. `USD`, `JPY`) |
| `spotBuy` | String | Spot Buy rate against TWD (BigDecimal serialized as string) |
| `spotSell` | String | Spot Sell rate against TWD (BigDecimal serialized as string) |

> See `get-rate` spec §5 for the full Redis value example.

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 400 | UNSUPPORTED_CURRENCY | Unsupported currency | `fromCurrency` or `toCurrency` not in supported currency list |
| 400 | INVALID_AMOUNT | Invalid amount | `amount` less than 0.01 or malformed |
| 503 | RATE_UNAVAILABLE | Rate data unavailable | External rate API failed and no Redis cache available |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
