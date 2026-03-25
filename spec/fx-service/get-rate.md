# Get Exchange Rate — fx-service Spec

> **Service:** fx-service
> **Contract Module:** fx-contract
> **Caller:** BFF (`FxFeignClient`), Transaction Orchestrator
> **Last Updated:** 2026-03

---

## 1. Overview

Accepts source and target currency codes from the caller and returns the current exchange rate with a timestamp.

Rate data is sourced from the **Bank of Taiwan (BOT) public CSV API** — no API key required. The system fetches all supported currency pair rates every 3 hours via a scheduled task and writes them to Redis in batch (TTL 10800s).
On query, Redis cache is checked first; if the cache is missing (e.g., on first startup), the bank API is called in real time and the result is written to cache (**Cache-Aside fallback**).

fx-service is a **stateless Utility Service**: it holds no account data, does not participate in Saga state management, and performs no MySQL writes.
BFF calls this API to display rates to users; Transaction Orchestrator calls this API to retrieve the live rate for settlement calculations.

---

## 2. API Definition — fx-service

### Endpoint

| Item | Value |
|------|-------|
| Method | GET |
| context-path | /fx |
| Controller mapping | /rate |
| Effective path | /fx/rate |
| Caller | BFF (`FxFeignClient`), Transaction Orchestrator |

### Request (`fx.contract.dto.FxRateRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| fromCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | Source currency (ISO 4217, uppercase, e.g. USD) |
| toCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | Target currency (ISO 4217, uppercase, e.g. TWD) |

> **Note:** GET request — fields are passed as query parameters (`?fromCurrency=USD&toCurrency=TWD`).

### Response (`fx.contract.dto.FxRateRs`)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| fromCurrency | String | Source currency (ISO 4217) | |
| toCurrency | String | Target currency (ISO 4217) | |
| rate | BigDecimal | Exchange rate (fromCurrency → toCurrency) | 10 decimal places |
| rateSource | String | Rate data source identifier | Fixed value `BOT` (Bank of Taiwan) |
| rateTimestamp | Long | Rate data timestamp (epoch millis) | Cache write time (BOT does not provide quote time; fetch time is used instead) |
| cachedAt | Long | Cache write timestamp (epoch millis) | Redis cache write time |

---

## 3. Rate Data Architecture

### 3.1 Partner Bank API (Bank of Taiwan)

| Item | Value |
|------|-------|
| Bank | Bank of Taiwan (BOT) |
| Endpoint | `https://rate.bot.com.tw/xrt/flcsv/0/day` |
| HTTP Method | GET |
| Response Format | `text/csv` (UTF-8) |
| Update Frequency | Hourly during business hours (Mon–Fri 09:00–17:00) |
| API Key | Not required (public access) |
| Component | `BotExchangeRateClient` (`RestTemplate` wrapper) |
| Config Location | `application.yml` → `fx.bank.bot.url` |

**CSV field structure (each row represents one currency, quoted against TWD):**

| Field Index | Field Name | Description |
|-------------|------------|-------------|
| 0 | Currency | Foreign currency code (e.g. USD, JPY) |
| 1 | Cash Buy | Bank cash buying rate |
| 2 | Cash Sell | Bank cash selling rate |
| 3 | Spot Buy | Bank spot buying rate |
| 4 | Spot Sell | Bank spot selling rate |
| 5–16 | Forward rates (10/30/60/90/120/180-day buy/sell) | Not used |

> **CSV example (excerpt):**
> ```
> Currency,Cash Buy,Cash Sell,Spot Buy,Spot Sell,...
> USD,31.36,31.86,31.56,31.66,...
> JPY,0.2062,0.2142,0.2082,0.2102,...
> ```

### 3.2 Rate Calculation Strategy

The BOT API uses **TWD as the base currency** (1 unit of foreign currency = N TWD), so each currency pair is calculated as follows:

| Scenario | Calculation | Field Used | Description |
|----------|-------------|------------|-------------|
| Foreign → TWD | Direct lookup | Spot Buy (index 3) | Bank "buys" foreign currency from cardholder; cardholder sells foreign currency for TWD |
| TWD → Foreign | `1 / Spot Sell` | Spot Sell (index 4) | Bank "sells" foreign currency to cardholder; cardholder buys foreign currency |
| Foreign → Foreign | Cross rate | Spot mid of both against TWD | `rate(A→B) = rate(A→TWD) / rate(B→TWD)` |

> **Credit card settlement convention:** Cross-border transactions are settled using the **Spot Buy rate** (bank buying the cardholder's foreign currency).

### 3.3 Scheduled Task (FxRateScheduler)

| Item | Value |
|------|-------|
| Component | `FxRateScheduler` (Spring `@Scheduled`) |
| Schedule | Every 3 hours (`cron = "0 0 */3 * * *"`) |
| Behavior | Calls BOT CSV API, parses all supported currencies in batch, writes each pair to Redis with TTL 10800s |
| On Startup | `@PostConstruct` triggers one immediate run to ensure Redis is pre-populated |
| Failure Handling | If a single currency pair fails to parse, log an Error and continue with remaining pairs (do not abort the batch) |

---

## 4. Service UseCase Flow

**UseCase:** `GetRateUseCase.getRate(FxRateRq)`
**UseCase Impl:** `GetRateUseCaseImpl`

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[DOMAIN]` | Validate that `fromCurrency` and `toCurrency` are supported; if both are the same, return `rate=1` directly | Unsupported currency → throw `UNSUPPORTED_CURRENCY` |
| 2 | `[REDIS READ]` | Read `fx:rate:{fromCurrency}:{toCurrency}`; cache hit → proceed to step 4 | Cache miss → proceed to step 3 (Cache-Aside fallback) |
| 3 | `[DOMAIN]` | Call `BotExchangeRateClient.fetchRate(fromCurrency, toCurrency)` for a live rate (parse CSV, apply calculation strategy), write to Redis with TTL 10800s (`FxRateService.fetchAndCache()`) | Bank API failure → throw `RATE_UNAVAILABLE` |
| 4 | `[RETURN]` | Return FxRateRs (fromCurrency, toCurrency, rate, rateSource, rateTimestamp, cachedAt) | — |

---

## 5. Database

### MySQL

> fx-service is a stateless Utility Service — **no MySQL reads or writes are performed**.

### Redis

| Key Pattern | Operation | TTL | Description |
|-------------|-----------|-----|-------------|
| `fx:rate:{fromCurrency}:{toCurrency}` | READ | 10800s | Read cached rate for a specific currency pair |

> TTL 10800s = 3 hours, aligned with the scheduler cycle. TTL is reset on every scheduler write.
> Cache-Aside fallback writes (step 3) also use 10800s TTL.
> Key examples: `fx:rate:USD:TWD`, `fx:rate:JPY:TWD`

### Redis Value Structure (`fx:rate:{fromCurrency}:{toCurrency}`)

| Field | Type | Description |
|-------|------|-------------|
| fromCurrency | String | Source currency (ISO 4217) |
| toCurrency | String | Target currency (ISO 4217) |
| rate | String | Exchange rate (BigDecimal serialized as string) |
| rateSource | String | Data source, fixed value `BOT` |
| rateTimestamp | Long | Rate fetch timestamp (epoch millis) |
| cachedAt | Long | Cache write timestamp (epoch millis) |

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger |
|-------------|------------|-------------|---------|
| 400 | UNSUPPORTED_CURRENCY | Unsupported currency | `fromCurrency` or `toCurrency` not in supported currency list |
| 503 | RATE_UNAVAILABLE | Rate data unavailable | No Redis cache and live BOT API call failed |
| 500 | INTERNAL_ERROR | System error | Unexpected exception |
