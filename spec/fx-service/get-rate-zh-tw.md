# 查詢匯率 — fx-service Spec

> **所屬服務：** fx-service
> **Contract Module：** fx-contract
> **呼叫方：** BFF（`FxFeignClient`）、Transaction Orchestrator
> **最後更新：** 2026-03

---

## 1. Overview

接收呼叫方提供的來源幣別與目標幣別，回傳當前匯率與時間戳記。

匯率資料來源為**台灣銀行（Bank of Taiwan）公開匯率 CSV API**，無需 API Key，系統以排程任務每 3 小時主動抓取一次完整 CSV，將所有支援幣別解析為一份原始匯率 Map，整包序列化為 JSON 後寫入 Redis（Key `fx:raw-rates`，TTL 10800s）。
查詢時讀取 Redis 中的原始匯率 Map，並在讀取時即時換算匯率；若快取不存在（如首次啟動），則即時呼叫銀行 API 並寫入快取（**Cache-Aside 補漏機制**）。

fx-service 為**無狀態 Utility Service**：不持有帳戶資料，不參與 Saga 狀態管理，不寫入 MySQL。
BFF 呼叫此 API 以展示匯率給使用者；Transaction Orchestrator 呼叫此 API 以取得即時匯率代入換算計算。

---

## 2. API Definition — fx-service

### Endpoint

| 項目 | 內容 |
|------|------|
| Method | GET |
| context-path | /fx |
| Controller mapping | /rate |
| 實際接收路徑 | /fx/rate |
| 呼叫方 | BFF（`FxFeignClient`）、Transaction Orchestrator |

### Request（`fx.contract.dto.FxRateRq`）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| fromCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | 來源幣別（ISO 4217，大寫，如 USD） |
| toCurrency | String | ✅ | @NotBlank, @Size(min=3, max=3) | 目標幣別（ISO 4217，大寫，如 TWD） |

> **注意：** GET 請求，欄位以 Query Parameter 傳遞（`?fromCurrency=USD&toCurrency=TWD`）。

### Response（`fx.contract.dto.FxRateRs`）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| fromCurrency | String | 來源幣別（ISO 4217） | |
| toCurrency | String | 目標幣別（ISO 4217） | |
| rate | BigDecimal | 匯率（fromCurrency → toCurrency） | 精度 10 位小數 |
| rateSource | String | 匯率資料來源識別 | 固定值 `BOT`（Bank of Taiwan） |
| rateTimestamp | Long | 匯率資料時間戳記（epoch millis） | 匯率換算時間（台灣銀行未提供報價時間，以換算時間代替） |
| cachedAt | Long | 快取寫入時間戳記（epoch millis） | Redis 快取寫入時間 |

---

## 3. 匯率資料架構

### 3.1 合作銀行 API（台灣銀行）

| 項目 | 內容 |
|------|------|
| 銀行 | 台灣銀行（Bank of Taiwan） |
| Endpoint | `https://rate.bot.com.tw/xrt/flcsv/0/day` |
| HTTP Method | GET |
| 回傳格式 | `text/csv`（UTF-8） |
| 更新頻率 | 營業時間（週一至五 09:00–17:00）每小時更新 |
| API Key | 不需要（公開存取） |
| 呼叫元件 | `BotExchangeRateClient`（`RestTemplate` 封裝） |
| 設定位置 | `application.yml` → `fx.bank.bot.url` |

**CSV 欄位結構（每行代表一個幣別，以 TWD 為計價基準）：**

| 欄位索引 | 欄位名稱 | 說明 |
|----------|----------|------|
| 0 | 幣別 | 外幣代碼（如 USD, JPY） |
| 1 | 現金匯率-本行買入 | 銀行買入現金價格 |
| 2 | 現金匯率-本行賣出 | 銀行賣出現金價格 |
| 3 | 即期匯率-本行買入 | 銀行買入即期價格 |
| 4 | 即期匯率-本行賣出 | 銀行賣出即期價格 |
| 5–16 | 遠期匯率（10/30/60/90/120/180天 買入/賣出） | 不使用 |

> **CSV 範例（擷取）：**
> ```
> 幣別,現金匯率(本行買入),現金匯率(本行賣出),即期匯率(本行買入),即期匯率(本行賣出),...
> USD,31.36,31.86,31.56,31.66,...
> JPY,0.2062,0.2142,0.2082,0.2102,...
> ```

### 3.2 匯率計算策略

台灣銀行 API 以 **TWD 為計價基準**（1 單位外幣 = N 元 TWD），因此各幣別對的計算方式如下：

| 情境 | 計算方式 | 使用欄位 | 說明 |
|------|----------|----------|------|
| 外幣 → TWD | 直接取得 | 即期匯率-本行買入（index 3） | 銀行向持卡人「買入」外幣，即持卡人賣出外幣換 TWD |
| TWD → 外幣 | `1 / 即期賣出` | 即期匯率-本行賣出（index 4） | 銀行向持卡人「賣出」外幣，即持卡人買入外幣 |
| 外幣 → 外幣 | 透過 TWD 均值交叉匯率 | 兩者各自對 TWD 的即期均值 | `rate(A→B) = mid(A→TWD) / mid(B→TWD)`，其中 `mid = (即期買入 + 即期賣出) / 2` |

> **信用卡結算慣例：** 境外消費入帳以**即期匯率-本行買入**換算（銀行買入持卡人的外幣）。

### 3.3 排程任務（FxRateScheduler）

| 項目 | 內容 |
|------|------|
| 元件 | `FxRateScheduler`（Spring `@Scheduled`） |
| 排程 | 每 3 小時執行一次（`cron = "0 0 */3 * * *"`） |
| 行為 | 呼叫台灣銀行 CSV API，將所有支援幣別解析為原始 `Map<String, BotRateRow>`，整包序列化為 JSON 後寫入 Redis（Key `fx:raw-rates`，TTL 10800s） |
| 啟動時 | `@PostConstruct` 啟動後立即執行一次，確保 Redis 有初始資料 |
| 失敗處理 | 單一幣別列解析失敗時（如欄位非數字），記錄 Warning Log 並跳過該列，其餘幣別仍正常寫入快取 |

---

## 4. Service UseCase Flow

**UseCase：** `GetRateUseCase.getRate(FxRateRq)`
**UseCase Impl：** `GetRateUseCaseImpl`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[DOMAIN]` | 驗證 `fromCurrency` 與 `toCurrency` 是否在支援幣別清單中（`CurrencyService.getCurrencies()`）；若相同幣別（`fromCurrency == toCurrency`）則直接回傳 `rate=1` | 不支援的幣別 → 拋出 `UNSUPPORTED_CURRENCY` |
| 2 | `[REDIS READ]` | 讀取 `fx:raw-rates`；快取命中 → 進入步驟 4 | 快取未命中 → 進入步驟 3（Cache-Aside 補漏） |
| 3 | `[DOMAIN]` | 呼叫 `BotExchangeRateClient.fetchParsedData()` 取得完整原始匯率 Map；透過 `FxRatePort.cacheRawRates()` 寫入 Redis（TTL 10800s） | 銀行 API 呼叫失敗或非預期例外 → 拋出 `RATE_UNAVAILABLE` |
| 4 | `[DOMAIN]` | 呼叫 `BotExchangeRateClient.computeRate(from, to, rawRates)`，依 §3.2 計算策略換算匯率 | 換算失敗（如幣別不在 Map 中） → 拋出 `RATE_UNAVAILABLE` |
| 5 | `[RETURN]` | 回傳 `FxRateRs`（fromCurrency、toCurrency、rate、rateSource=`BOT`、rateTimestamp、cachedAt） | — |

---

## 5. Database

### MySQL

> fx-service 為無狀態 Utility Service，**不進行任何 MySQL 讀寫**。

### Redis

| Key | 操作 | TTL | 說明 |
|-----|------|-----|------|
| `fx:raw-rates` | READ / WRITE | 10800s | 所有支援幣別的原始匯率 Map，整包存為一筆 JSON |

> TTL 10800s = 3 小時，與排程週期對齊。排程任務每次寫入時重置 TTL。
> Cache-Aside 補漏（步驟 3）寫入時同樣使用 10800s TTL。

### Redis Value 結構（`fx:raw-rates`）

儲存格式為 `Map<String, BotRateRow>` 的 JSON 序列化結果，Map Key 為幣別代碼（ISO 4217），Value 為 `BotRateRow` 物件：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `{幣別代碼}`（Map Key） | String | 外幣代碼（如 `USD`、`JPY`） |
| `spotBuy` | String | 對 TWD 的即期買入匯率（BigDecimal 序列化為字串） |
| `spotSell` | String | 對 TWD 的即期賣出匯率（BigDecimal 序列化為字串） |

> **範例：**
> ```json
> {
>   "USD": { "spotBuy": "31.56", "spotSell": "31.66" },
>   "JPY": { "spotBuy": "0.2082", "spotSell": "0.2102" }
> }
> ```

---

## 6. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 400 | UNSUPPORTED_CURRENCY | 不支援的幣別 | `fromCurrency` 或 `toCurrency` 不在支援幣別清單中 |
| 503 | RATE_UNAVAILABLE | 匯率資料無法取得 | Redis 無快取且即時呼叫台灣銀行 API 失敗，或匯率換算失敗 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |
