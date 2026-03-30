# CombineKey 啟動載入器 — card-service Spec

> **Service：** card-service
> **元件類型：** 應用程式生命週期 — `ApplicationReadyEvent` 處理器
> **最後更新：** 2026-03

---

## 1. 概述

應用程式啟動時，card-service 從 `card_key_parts` 讀取三個加密的 Source Key Part，以 Master Key 逐一解密，XOR 合成 `combineKey`，並將結果以 `SecretKey` 形式存入單例 `CombineKeyHolder` Bean 中。

`combineKey` 完全常駐於 JVM Heap 記憶體中，供 Phase 7（`POST /card-service/auth/authorize`）解密 NCCC 加密的卡片資料使用。

JVM 關閉時（`@PreDestroy`），`CombineKeyHolder` 清零金鑰的底層位元組陣列，防止金鑰殘留於記憶體傾印檔案中。

> **無 HTTP 端點。** 本 spec 描述的是生命週期元件，而非 API。
> 完整金鑰架構背景請參閱 [guideline/14-3ds-combine-key-zh-tw.md](../../guideline/14-3ds-combine-key-zh-tw.md)。

---

## 2. 元件定義

### CombineKeyHolder

常駐應用程式整個生命週期的單例 Spring Bean，持有組合完成的 `combineKey`。

| 屬性 | 值 |
|------|----|
| Bean scope | Singleton |
| Class | `card.service.security.CombineKeyHolder` |
| Interface | 實作 `DisposableBean`（供 `@PreDestroy` 清零使用） |
| 執行緒安全 | `combineKey` 欄位為 `volatile`；熱更新時使用原子性參考替換 |

```
CombineKeyHolder
  ─ volatile SecretKey combineKey          // javax.crypto.spec.SecretKeySpec，演算法 "AES"
  ─ void set(SecretKey key)                // 由啟動載入器及 SourceKeyUpdateUseCase 呼叫
  ─ SecretKey get()                        // 由 Phase 7 解密邏輯呼叫；若為 null 則拋出例外
  ─ void destroy()                         // @PreDestroy — 清零底層位元組陣列
```

### CombineKeyStartupLoader

實作 `ApplicationListener<ApplicationReadyEvent>`，在 Spring Context 與所有資料來源連線完全就緒後執行載入序列。

| 屬性 | 值 |
|------|----|
| Class | `card.service.security.CombineKeyStartupLoader` |
| 觸發時機 | `ApplicationReadyEvent` |
| 依賴元件 | `CardKeyPartsJpaRepository`、`CardEncryptionService`、`KcvService`、`CombineKeyHolder` |

---

## 3. 啟動 UseCase Flow

**元件：** `CombineKeyStartupLoader.onApplicationEvent(ApplicationReadyEvent)`

| Step | Type | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | `SELECT * FROM card_key_parts ORDER BY part_seq`；讀取所有資料列 | `CardKeyPartsJpaRepository.findAllOrderByPartSeq()` |
| 2 | `[VALIDATE]` | 若資料列數 < 3：log `ERROR "card_key_parts 僅有 {n} 筆；預期 3 筆 — combineKey 無法建立"` → `CombineKeyHolder.set(null)` → **return**（服務以降級模式啟動；參見第 5 節） | — |
| 3 | `[DOMAIN]` | 逐一處理 3 筆資料：以 `MASTER_KEY`（AES-256-GCM）解密 `encrypted_part` → `sourceKeyBytes[n]`（byte[32]）；任一 Part 解密失敗 → log `ERROR "無法解密 Part {partSeq}"` → 清零已解密緩衝區 → `CombineKeyHolder.set(null)` → **return** | `CardEncryptionService.decryptKeyPart()` |
| 4 | `[DOMAIN]` | 合成：`combineKeyBytes[i] = sourceKey1[i] XOR sourceKey2[i] XOR sourceKey3[i]`（對 32 個位元組逐一 XOR） | `CombineKeyService.xorCompose()` |
| 5 | `[DOMAIN]` | 包裝：`SecretKey = new SecretKeySpec(combineKeyBytes, "AES")` | — |
| 6 | `[DOMAIN]` | 清零所有中間位元組陣列：`sourceKey1Bytes`、`sourceKey2Bytes`、`sourceKey3Bytes`、`combineKeyBytes`（`SecretKeySpec` 自行保留內部副本） | `KeyZeroizer.zeroFill()` |
| 7 | `[DOMAIN]` | `CombineKeyHolder.set(secretKey)` — 原子性 volatile 寫入 | — |
| 8 | `[DOMAIN]` | log `INFO "combineKey 載入成功（parts: 1, 2, 3）"` — **不記錄任何金鑰位元組** | — |

---

## 4. 關閉行為（`@PreDestroy`）

**元件：** `CombineKeyHolder.destroy()`（透過 `DisposableBean.destroy()`）

| Step | Type | 說明 |
|------|------|------|
| 1 | `[DOMAIN]` | 若 `combineKey == null`：log `INFO "combineKey 於關閉時為 null — 無需清零"` → return |
| 2 | `[DOMAIN]` | 透過 `getEncoded()` 取得 `SecretKeySpec` 的底層位元組陣列 |
| 3 | `[DOMAIN]` | 清零位元組陣列：`Arrays.fill(keyBytes, (byte) 0)` |
| 4 | `[DOMAIN]` | 設定 `combineKey = null` |
| 5 | `[DOMAIN]` | log `INFO "combineKey 已於關閉時清零"` |

> **注意：** `javax.crypto.spec.SecretKeySpec.getEncoded()` 回傳的是內部陣列的**副本**。
> 清零副本並不會清零 JVM 內部的原始陣列。若目標 JRE 支援，建議呼叫 `sun.security.util.Password.destroy(key)` 或等效的 `Destroyable.destroy()` 以可靠清零。請在程式碼中以註解說明此限制。

---

## 5. 降級啟動行為

若啟動載入失敗（Step 2 或 Step 3），`CombineKeyHolder.combineKey` 將維持 `null`。
服務仍正常啟動 — 其他端點（卡片 CRUD、KCV 驗證、Source Key 更新）維持可用。僅授權路徑的解密受影響。

| 端點 | combineKey 為 null 時的行為 |
|------|----------------------------|
| `POST /card-service/auth/authorize` | `CombineKeyHolder.get()` 拋出 `CombineKeyUnavailableException` → 對應 HTTP 503，錯誤碼 `CA00013 COMBINE_KEY_UNAVAILABLE` |
| 其他所有 card-service 端點 | 不受影響 |

> **維運注意：** 降級啟動應觸發 `"combineKey unavailable"` ERROR log 的告警規則。
> 操作人員應透過 `PUT /internal/keys/source-key/{partSeq}` 注入或驗證 Source Key，
> 然後重啟服務（或等待 Source Key 成功更新後自動觸發的熱更新路徑）。

---

## 6. 熱更新（由 `SourceKeyUpdateUseCase` 觸發）

操作人員透過 Source Key 更新 API 成功更新某個 Part 後，該 UseCase 在 Step 7 呼叫 `CombineKeyService.reload()`，執行與 Step 3–7 相同的組合邏輯（讀取 DB 最新狀態），並呼叫 `CombineKeyHolder.set(newSecretKey)`。

此操作在不重啟服務的情況下替換記憶體中的 `combineKey`。

| Step | Type | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[DB READ]` | 讀取 `card_key_parts` 全部 3 筆 | `CardKeyPartsJpaRepository.findAllOrderByPartSeq()` |
| 2 | `[DOMAIN]` | 以 `MASTER_KEY` 逐一解密 → 3 × `byte[32]` | `CardEncryptionService.decryptKeyPart()` |
| 3 | `[DOMAIN]` | XOR 合成 → `combineKeyBytes` | `CombineKeyService.xorCompose()` |
| 4 | `[DOMAIN]` | `CombineKeyHolder.set(new SecretKeySpec(combineKeyBytes, "AES"))` | — |
| 5 | `[DOMAIN]` | 清零所有中間位元組陣列 | `KeyZeroizer.zeroFill()` |

---

## 7. Database

### MySQL

| Table | Operation | 說明 |
|-------|-----------|------|
| `card_key_parts` | READ | 啟動時及每次熱更新時讀取全部 3 筆資料列 |

### Redis

> 此元件不執行任何 Redis 操作。

### Table Schema

#### card_key_parts

| Column | Type | Nullable | 說明 |
|--------|------|----------|------|
| part_seq | INT | NO | PK：1、2 或 3 |
| encrypted_part | VARCHAR(512) | NO | AES-256-GCM 加密的 256-bit Source Key；以 env/KMS 的 Master Key 加密 |
| updated_at | DATETIME(3) | NO | 最近一次注入或輪換的時間戳 |

---

## 8. 錯誤說明

> 本元件無 HTTP 端點，故無 HTTP 錯誤碼。
> 啟動失敗透過 ERROR log 回報，執行時僅於呼叫 `POST /card-service/auth/authorize` 時以 HTTP 503 顯現。

| 狀況 | 行為 | Log 等級 |
|------|------|---------|
| `card_key_parts` 資料列數 < 3 | `combineKey = null`；服務以降級模式啟動 | ERROR |
| 任一 Part 解密失敗 | `combineKey = null`；服務以降級模式啟動 | ERROR |
| `CombineKeyHolder.get()` 在 `null` 狀態被呼叫 | 拋出 `CombineKeyUnavailableException` → CA00013 | — |

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|---------|
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | 呼叫 `auth/authorize` 時 `combineKey` 為 null |
