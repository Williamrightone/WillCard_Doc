# 3DS combineKey 加解密（Phase 5）

本文件描述 WillCard 如何建構與管理 combineKey，用於在 3DS 授權流程中解密 NCCC 傳入的卡片資料。
涵蓋金鑰架構、Key Check Value（KCV）機制、DB Schema、啟動時組合、執行時解密、Source Key 輪換，以及 Mock 環境設定。
實作細節（API 合約、UseCase 流程）請參閱 `spec/card-service/` 下的對應 spec 文件。

> **Phase 5 範疇：** Source Key 儲存（新增 / 輪換）、啟動時 combineKey 組合，以及 KCV 驗證。
> combineKey 在 Phase 7（Issuer Authorization）時用於解密 NCCC 提供的 `encryptedCard` payload。
> Mock NCCC 的使用方式定義於 Phase 8。

---

## 1. 概述

在 3DS 流程中，NCCC 的 iframe 直接在瀏覽器端接收持卡人的 PAN、有效期限與 CVV。
在將這些資料轉發至 Issuer Authorization API（`POST /card-service/auth/authorize`）之前，
NCCC 會以 **combineKey** 加密卡片資料 — combineKey 是由三把 Source Key 組合而成的 256-bit AES 金鑰。

card-service 將 combineKey 完全保存於 JVM Heap 記憶體中。三把 Source Key 以 Master Key（來自 KMS / 環境變數）加密後存放於 `card_key_parts` 表。啟動時，card-service 讀取並解密三個 part，XOR 後重建 combineKey，以 `SecretKey` 物件形式常駐記憶體。

> **與 Phase 1 卡片落地加密的區別：**
> Phase 1 使用 `CARD_ENCRYPTION_KEY`（來自 env/KMS）加密 `card` 表中的 PAN、有效期限與持卡人姓名。
> combineKey 為 NCCC 通道專用，僅存在於記憶體中，永遠不寫入任何儲存媒介。

---

## 2. 金鑰架構

```
MASTER_KEY（來自 KMS / 環境變數，不寫入 DB）
  ├── AES-256-GCM encrypt(sourceKey1) → card_key_parts（part_seq = 1）
  ├── AES-256-GCM encrypt(sourceKey2) → card_key_parts（part_seq = 2）
  └── AES-256-GCM encrypt(sourceKey3) → card_key_parts（part_seq = 3）

combineKey = sourceKey1 XOR sourceKey2 XOR sourceKey3   （僅存於 JVM Heap，永遠不落地）
```

| 金鑰 | 長度 | 儲存位置 | 生命週期 |
|------|------|---------|---------|
| `MASTER_KEY` | 256-bit AES | KMS / 環境變數（僅此） | 長期；僅透過 Ops/KMS 輪換 |
| `sourceKey1/2/3` | 256-bit AES | `card_key_parts`（加密靜置） | 由 NCCC 定期輪換 |
| `combineKey` | 256-bit AES | JVM Heap（僅此） | 啟動時及每次 Source Key 更新後重建；`@PreDestroy` 時清零 |

> **分散持有（Split Knowledge）：** 任何單一方都不擁有全部三把 Source Key。
> 這符合支付卡產業的金鑰持有者實務 — 單一人員無法單獨重建 combineKey。

---

## 3. Key Check Value（KCV）

KCV 是一個 3 位元組的金鑰指紋，透過以該金鑰加密一組零位元組來計算。
它讓操作人員驗證金鑰輸入是否正確，而不需要暴露金鑰本身。

### 計算方式

```
kcv = AES-256-ECB( sourceKey, byte[16]{ 0x00 } )[0..2]
    → 以 6 個大寫十六進位字元表示
```

| 參數 | 值 |
|------|----|
| 演算法 | AES-256-ECB（無 IV；確定性） |
| 明文 | 16 位元組全 `0x00` |
| 金鑰 | 待驗證的 256-bit Source Key |
| 輸出 | 前 3 位元組 → 6 個大寫十六進位字元（例：`A3F2B1`） |

> 此方式遵循 ISO/TR 21188 KCV 語意，適用於 AES-256 金鑰，與 HSM 金鑰注入實務一致。

### 在金鑰輪換中的使用

NCCC 輪換 Source Key 時，操作人員會從 NCCC 取得新金鑰（64 hex 字元）及其預期 KCV。注入流程如下：

1. 操作人員呼叫 `PUT /card-service/internal/keys/source-key/{partSeq}`，傳入 `sourceKeyHex` 與 `kcv`。
2. card-service 計算 `kcv_computed = AES-256-ECB(sourceKey, zeros)[0:3]`。
3. `kcv_computed == kcv_provided` → 金鑰接受，以 MASTER_KEY 加密後存入 DB。
4. `kcv_computed != kcv_provided` → 拒絕（`KCV_MISMATCH`）；金鑰不寫入 DB。

### KCV 查詢

操作人員也可透過 `GET /card-service/internal/keys/source-key/{partSeq}/kcv` 查詢當前存放金鑰的 KCV，在不取得金鑰本身的情況下確認目前 DB 中的金鑰內容。

---

## 4. DB Schema

### `card_key_parts`

> **Source migration：** `V5.0.0__create_card_key_parts_table.sql`
> **最後更新：** 2026-03

```sql
CREATE TABLE `card_key_parts` (
  `part_seq`        INT           NOT NULL COMMENT 'PK：Source Key 序號（1、2 或 3）',
  `encrypted_part`  VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM 加密的 256-bit Source Key；以 env/KMS 的 Master Key 加密',
  `updated_at`      DATETIME(3)   NOT NULL COMMENT '最近一次金鑰注入或輪換的時間戳',
  PRIMARY KEY (`part_seq`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> `part_seq` 為自然 PK（固定恰好 3 筆資料：1、2、3），無需代理鍵。
> `updated_at` 提供每把 Source Key 最後輪換時間的稽核記錄。
> 無 `created_at`：資料列於部署時播種一次，後續以 Upsert 方式就地更新。

### 初始播種

透過獨立的 DML migration（`V5.0.1__seed_card_key_parts.sql`）插入初始 3 筆資料。
正式環境種入 NCCC 提供的初始 Source Key（以 MASTER_KEY 加密後）。
開發 / 測試環境使用固定的 Mock 值（參見第 8 節）。

---

## 5. 啟動行為

由 `ApplicationReadyEvent` 觸發（在 Spring Context、JPA 及 DB 連線就緒後執行）：

```
Step 1：[DB READ]   SELECT * FROM card_key_parts ORDER BY part_seq
        → 若 row count < 3：
              log ERROR "card_key_parts 資料不足 3 筆；combineKey 無法建立"
              設定 CombineKeyHolder.combineKey = null
              （授權請求將收到 DECRYPTION_KEY_NOT_AVAILABLE）

Step 2：[DOMAIN]   逐一解密：以 MASTER_KEY（AES-256-GCM）解密 encrypted_part
                   → sourceKey1Bytes, sourceKey2Bytes, sourceKey3Bytes
        → 解密失敗：
              log ERROR "無法解密 Source Key part {partSeq}"
              設定 CombineKeyHolder.combineKey = null

Step 3：[DOMAIN]   combineKey[i] = sourceKey1[i] XOR sourceKey2[i] XOR sourceKey3[i]
                   （對 32 位元組逐位元組 XOR）

Step 4：[DOMAIN]   CombineKeyHolder.set(new SecretKeySpec(combineKeyBytes, "AES"))
                   → 包裝為 SecretKeySpec 後，立即清零 sourceKey1/2/3Bytes 及 combineKeyBytes 的位元組陣列

Step 5：[DOMAIN]   log INFO "combineKey 載入成功"（不記錄任何金鑰位元組）
```

**`@PreDestroy`（JVM 關閉時）：**

```
CombineKeyHolder.destroy()
  → 清零 SecretKey 內部的位元組陣列
  → 將 combineKey 參考設為 null
  → log INFO "combineKey 已清零"
```

---

## 6. 執行時解密（Phase 7）

card-service 收到 `POST /card-service/auth/authorize` 的 `encryptedCard` 時：

```
encryptedCard（NCCC 以 AES-256-GCM 加密的密文）
  → CombineKeyHolder.get()                        // 若為 null 則拋出例外
  → AES.decrypt(encryptedCard, combineKey, gcmIV) // GCM IV 前置於密文中
  → 明文：{ pan, expiryMMYY, cvv }
  → PAN 立即用於 HMAC-SHA256 查詢（pan_hash）
  → CVV 僅用於記憶體內驗證
  → 使用後立即清零明文位元組陣列
```

**PCI DSS 規定（解密明文）：**
- 僅存在於單一請求的 JVM Heap 中。
- 絕對不得出現於任何 log、序列化物件或持久化記錄。
- CVV 於 `verify-challenge` 完成後（或任何錯誤路徑上）立即清零。

---

## 7. Source Key 輪換

**輪換流程（一次更換一個 part）：**

1. NCCC 提供：新的 `sourceKeyHex`（64 hex 字元）、`kcv`（6 hex 字元），以及要更新的 `partSeq`。
2. 操作人員呼叫 `PUT /card-service/internal/keys/source-key/{partSeq}`。
3. card-service 驗證 KCV → 加密 Source Key → Upsert `card_key_parts` → 重新載入 combineKey。
4. 新的 combineKey 立即生效（記憶體中原子性替換 `CombineKeyHolder`）。

> **輪換順序：** 由於 combineKey 是三個 part 的 XOR，每輪換一個 part，combineKey 即改變一次。
> 請與 NCCC 協調新組合金鑰的生效時機（通常三個 part 在同一作業期間依序輪換完畢）。

> **飛行中的交易：** 輪換發生時，已在 NCCC iframe 中進行的交易可能是以舊 combineKey 加密的。
> 輪換後，該 `encryptedCard` 將解密失敗。用戶需重新發起交易，此為預期行為。

---

## 8. Mock / Dev 環境

在 `dev` 及 `local` Spring Profile 中：

- 固定的 `MOCK_COMBINE_KEY`（256-bit，全零或已知測試值）直接從 `application-dev.yml` 載入至 `CombineKeyHolder`，跳過 `card_key_parts` DB 查詢。
- `card_key_parts` 播入三個 part，其 XOR 等於 `MOCK_COMBINE_KEY`（僅供 Dev DB 一致性；DB 查詢路徑仍可在整合測試中驗證）。
- Mock NCCC Service 使用相同的 `MOCK_COMBINE_KEY` 加密卡片資料後呼叫 `POST /card-service/auth/authorize`。

```yaml
# application-dev.yml
willcard:
  security:
    mock-combine-key: "0000000000000000000000000000000000000000000000000000000000000000"
    # 64 hex 字元 = 256-bit 全零金鑰，僅供開發環境使用
```

> **正式環境或 Staging 嚴禁使用 Mock Key。** Dev Key 刻意設計為無安全性保護。

---

## 9. API 清單

### 9.1 更新 Source Key（內部）

| 欄位 | 值 |
|------|----|
| 操作 | `PUT /card-service/internal/keys/source-key/{partSeq}` |
| 呼叫方 | 操作人員管理工具（僅限內部） |
| 需要認證 | 否（內部；Nginx 封鎖 `/internal/` 對外流量） |
| 用途 | 注入或輪換一個 Source Key Part，含 KCV 驗證；並重新載入 combineKey |

Spec：[card-service/source-key-update-zh-tw.md](../spec/card-service/source-key-update-zh-tw.md)

### 9.2 驗證 Source Key KCV（內部）

| 欄位 | 值 |
|------|----|
| 操作 | `POST /card-service/internal/keys/source-key/{partSeq}/kcv/verify` |
| 呼叫方 | 操作人員管理工具（僅限內部） |
| 需要認證 | 否（內部） |
| 用途 | 操作人員傳入預期 KCV，服務回傳 `matched: Boolean` — 金鑰本體永遠不暴露於回應中 |

Spec：[card-service/source-key-kcv-zh-tw.md](../spec/card-service/source-key-kcv-zh-tw.md)

### 9.3 CombineKey 啟動載入器（應用程式生命週期）

| 欄位 | 值 |
|------|----|
| 觸發時機 | `ApplicationReadyEvent` |
| 元件 | `CombineKeyStartupLoader` → `CombineKeyHolder` |
| 用途 | 從 DB 解密 3 個 Source Key Part，XOR 合成 combineKey，存入 JVM Heap；`@PreDestroy` 時清零 |

Spec：[card-service/combine-key-startup-zh-tw.md](../spec/card-service/combine-key-startup-zh-tw.md)

---

## 10. 安全規範

| 規範 | 實作方式 |
|------|---------|
| Source Key 不得出現於 Log | 所有金鑰位元組陣列一律記錄為 `[REDACTED]`；`sourceKeyHex` 輸入欄位在稽核日誌中遮罩 |
| combineKey 不得序列化 | `CombineKeyHolder` 僅暴露 `SecretKey`；原始位元組在外部不可存取 |
| MASTER_KEY 不得存入 DB | 僅透過 KMS / 環境變數在啟動時注入 |
| 管理 API 傳輸安全 | 管理工具須透過內部網路呼叫；Nginx 封鎖外部對 `/internal/` 的存取 |
| 關閉時清零金鑰 | `@PreDestroy` 清零 combineKey 的位元組陣列 |
| CVV 授權後清零 | 在 `verify-challenge` 完成後或任何失敗路徑上立即清除（Phase 7） |
| KCV 不洩漏金鑰資訊 | 揭露 3 位元組 KCV 對 256-bit 金鑰的加密強度影響可忽略不計 |

---

## 11. 已知待辦（Phase 5 範疇）

| 待辦項目 | 說明 |
|---------|------|
| 輪換原子性 | combineKey 熱替換為 `CombineKeyHolder` 的單欄位取代；輪換期間的並發解密請求可能短暫使用過渡性金鑰 |
| MASTER_KEY 輪換 | 需重新加密全部 3 筆 `card_key_parts`；Phase 5 未定義任何工具 |
| HSM 整合 | Source Key 以 hex 字串形式透過管理 API 注入；Phase 5 不實作實體 HSM 金鑰注入 |
| 金鑰異動稽核 Log | `card_key_parts.updated_at` 記錄輪換時間；無專用的金鑰輪換稽核日誌表 |
| 雙人管制（Dual Control） | Phase 5 不強制實施分散持有注入（需兩位操作人員分別注入不同 part） |

---
