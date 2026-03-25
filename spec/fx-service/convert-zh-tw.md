# 換算金額 — fx-service Spec

> **所屬服務：** fx-service
> **Contract Module：** fx-contract
> **呼叫方：** Transaction Orchestrator
> **最後更新：** 2026-03

---

## 1. Overview

接收交易金額與幣別對，查詢即時匯率後回傳換算後金額。
此 API 由 Transaction Orchestrator 在 Saga 流程中呼叫，用於計算外幣交易的入帳金額（如信用卡境外消費結算）。
fx-service 為**無狀態 Utility Service**：不持有帳戶資料，不參與 Saga 狀態管理，不寫入 MySQL。
使用即時匯率策略（不鎖定），每次呼叫取用當下快取匯率，符合傳統信用卡行為。

---

## 2. API Definition — fx-service

### Endpoint

| 項目 | 內容 |
|------|------|
| Method | POST |
| context-path | /fx |
| Controller mapping | /convert |
| 實際接收路徑 | /fx/convert |
| 呼叫方 | Transaction Orchestrator |

### Request（`fx.contract.dto.FxConvertRq`）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | 欲換算的金額（來源幣別） |
| fromCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | 來源幣別（ISO 4217，大寫，如 USD） |
| toCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | 目標幣別（ISO 4217，大寫，如 TWD） |

### Response（`fx.contract.dto.FxConvertRs`）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| originalAmount | BigDecimal | 原始金額（來源幣別） | |
| fromCurrency | String | 來源幣別（ISO 4217） | |
| convertedAmount | BigDecimal | 換算後金額（目標幣別） | 四捨五入至小數點後 2 位 |
| toCurrency | String | 目標幣別（ISO 4217） | |
| rate | BigDecimal | 套用的匯率 | 精度 10 位小數 |
| rateTimestamp | Long | 匯率資料時間戳記（epoch millis） | 供 Orchestrator 記錄於交易紀錄 |

---

## 3. Service UseCase Flow

**UseCase：** `ConvertUseCase.convert(FxConvertRq)`
**UseCase Impl：** `ConvertUseCaseImpl`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[DOMAIN]` | 透過 `CurrencyService.getCurrencies()` 驗證 `fromCurrency` 與 `toCurrency` 是否為系統支援幣別 | 不支援的幣別 → 拋出 `UNSUPPORTED_CURRENCY` |
| 2 | `[REDIS READ]` | 讀取 `fx:raw-rates`；快取命中 → 進入步驟 4 | 快取未命中 → 進入步驟 3 |
| 3 | `[DOMAIN]` | 呼叫 `BotExchangeRateClient.fetchParsedData()` 取得完整原始匯率 Map；透過 `FxRatePort.cacheRawRates()` 寫入 Redis（TTL 10800s） | 銀行 API 呼叫失敗或非預期例外 → 拋出 `RATE_UNAVAILABLE` |
| 4 | `[DOMAIN]` | 呼叫 `BotExchangeRateClient.computeRate(from, to, rawRates)`，依 get-rate spec §3.2 計算策略換算匯率 | 換算失敗（如幣別不在 Map 中）→ 拋出 `RATE_UNAVAILABLE` |
| 5 | `[DOMAIN]` | 計算換算金額：`convertedAmount = amount × rate`，四捨五入至小數點後 2 位（`FxCalculationService.convert()`） | 計算異常 → 拋出 `INTERNAL_ERROR` |
| 6 | `[RETURN]` | 回傳 FxConvertRs（originalAmount、fromCurrency、convertedAmount、toCurrency、rate、rateTimestamp） | — |

---

## 4. Database

### MySQL

> fx-service 為無狀態 Utility Service，**不進行任何 MySQL 讀寫**。

### Redis

| Key | 操作 | TTL | 說明 |
|-----|------|-----|------|
| `fx:raw-rates` | READ | — | 所有支援幣別的原始匯率 Map，整包存為一筆 JSON |

> **匯率快取寫入由排程任務（FX Rate Scheduler）負責**，本 API 於快取未命中時透過 `FxRatePort.cacheRawRates()` 觸發即時更新。

### Redis Value 結構（`fx:raw-rates`）

儲存格式為 `Map<String, BotRateRow>` 的 JSON 序列化結果，Map Key 為幣別代碼（ISO 4217），Value 為 `BotRateRow` 物件：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `{幣別代碼}`（Map Key） | String | 外幣代碼（如 `USD`、`JPY`） |
| `spotBuy` | String | 對 TWD 的即期買入匯率（BigDecimal 序列化為字串） |
| `spotSell` | String | 對 TWD 的即期賣出匯率（BigDecimal 序列化為字串） |

> 完整 Redis Value 範例請參閱 get-rate spec §5。

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 400 | UNSUPPORTED_CURRENCY | 不支援的幣別 | `fromCurrency` 或 `toCurrency` 不在支援幣別清單中 |
| 400 | INVALID_AMOUNT | 無效金額 | `amount` 小於 0.01 或格式錯誤 |
| 503 | RATE_UNAVAILABLE | 匯率資料無法取得 | 外部匯率 API 失敗且 Redis 無快取 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |
