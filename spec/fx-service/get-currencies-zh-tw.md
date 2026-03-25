# 查詢支援幣別 — fx-service Spec

> **所屬服務：** fx-service
> **Contract Module：** fx-contract
> **呼叫方：** BFF（`FxFeignClient`）
> **最後更新：** 2026-03

---

## 1. Overview

回傳系統目前支援的幣別清單，供 BFF 提供給前端下拉選單使用。
幣別清單為靜態配置，快取於 Redis 以減少重複讀取開銷。
fx-service 為**無狀態 Utility Service**：不持有帳戶資料，不參與 Saga 狀態管理，不寫入 MySQL。

---

## 2. API Definition — fx-service

### Endpoint

| 項目 | 內容 |
|------|------|
| Method | GET |
| context-path | /fx |
| Controller mapping | /currencies |
| 實際接收路徑 | /fx/currencies |
| 呼叫方 | BFF（`FxFeignClient`） |

### Request

無 Request Body / Query Parameter。

### Response（`fx.contract.dto.FxCurrencyRs`）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| currencies | List\<CurrencyItem\> | 支援幣別清單 | 依幣別代碼字母排序 |

#### CurrencyItem

| 欄位 | 型別 | 說明 |
|------|------|------|
| code | String | 幣別代碼（ISO 4217，大寫，如 USD） |
| name | String | 幣別名稱（英文，如 US Dollar） |
| nameZhTw | String | 幣別名稱（繁體中文，如 美元） |
| symbol | String | 幣別符號（如 $、¥、NT$） |

---

## 3. Service UseCase Flow

**UseCase：** `GetCurrenciesUseCase.getCurrencies()`
**UseCase Impl：** `GetCurrenciesUseCaseImpl`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[REDIS READ]` | 讀取 `fx:currencies`；快取命中 → 直接進入步驟 3 | 快取未命中 → 進入步驟 2 |
| 2 | `[DOMAIN]` | 從應用程式設定檔（`application.yml`）載入支援幣別清單，寫入 `fx:currencies`（TTL 86400s） | 載入失敗 → 拋出 `INTERNAL_ERROR` |
| 3 | `[RETURN]` | 回傳 FxCurrencyRs（currencies 清單，依 code 字母排序） | — |

---

## 4. Database

### MySQL

> fx-service 為無狀態 Utility Service，**不進行任何 MySQL 讀寫**。

### Redis

| Key Pattern | 操作 | TTL | 說明 |
|-------------|------|-----|------|
| `fx:currencies` | READ | — | 查詢支援幣別清單快取 |

> 幣別清單屬於低頻變動資料，TTL 建議設為 86400s（24 小時）。
> 快取由首次查詢時（或快取過期後首次查詢）觸發寫入，資料來源為 `application.yml` 設定檔。

### Redis Value 結構（`fx:currencies`）

以 JSON Array 序列化存放，每個元素結構如下：

| 欄位 | 型別 | 說明 |
|------|------|------|
| code | String | 幣別代碼（ISO 4217） |
| name | String | 幣別英文名稱 |
| nameZhTw | String | 幣別繁體中文名稱 |
| symbol | String | 幣別符號 |

### 支援幣別清單（初始設定）

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

> 此清單為初始設定，實際以 `application.yml` 中的 `fx.supported-currencies` 設定為準。

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 500 | INTERNAL_ERROR | 系統錯誤 | 幣別清單設定檔載入失敗或非預期例外 |
