# Source Key 更新 — card-service Spec

> **Service：** card-service
> **Contract Module：** card-contract
> **呼叫方：** 操作人員管理工具（僅限內部）
> **最後更新：** 2026-03

---

## 1. 概述

用於注入或輪換三把 Source Key 之一的內部 API，這三把 Source Key 共同組合成 combineKey。

接受以 64 個十六進位字元表示的 Source Key（256-bit 金鑰），以及其對應的 Key Check Value（KCV）。
透過獨立計算 KCV 來驗證提供的金鑰是否正確。
驗證成功後，以 Master Key（來自 env/KMS）加密 Source Key，並以 Upsert 方式更新 `card_key_parts` 對應的資料列。
完成後從 DB 重新讀取三個 Part，並重建記憶體中的 combineKey。

> 此端點僅限內部網路存取。Nginx 封鎖所有來自外部的 `/internal/` 路徑請求。

完整的 combineKey 架構與 KCV 計算規格，請參閱 [guideline/14-3ds-combine-key-zh-tw.md](../../guideline/14-3ds-combine-key-zh-tw.md)。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|----|
| Method | PUT |
| context-path | `/card` |
| Controller mapping | `/internal/keys/source-key/{partSeq}` |
| 實際接收路徑 | `/card/internal/keys/source-key/{partSeq}` |
| 呼叫方 | 操作人員管理工具 |

### Path Variable

| 變數 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| partSeq | Integer | ✅ | 1、2 或 3 | Source Key Part 序號 |

### Request（`card.contract.dto.SourceKeyUpdateRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| sourceKeyHex | String | ✅ | @NotBlank、長度 = 64、有效十六進位 | 256-bit Source Key，以 64 個十六進位字元表示（大小寫不拘） |
| kcv | String | ✅ | @NotBlank、長度 = 6、有效十六進位 | Key Check Value — `AES-256-ECB(sourceKey, 0x00×16)` 前 3 位元組的 6 個十六進位字元 |

### Response（`card.contract.dto.SourceKeyUpdateRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| partSeq | Integer | 已更新的 Part 序號 |
| kcv | String | 新存入金鑰的 KCV（6 個大寫十六進位字元；由服務計算） |
| updatedAt | String | ISO-8601 格式的更新時間戳 |

---

## 3. Service UseCase Flow

**UseCase：** `SourceKeyUpdateUseCase.update(int partSeq, SourceKeyUpdateRq rq)`（card-service）
**UseCase Impl：** `SourceKeyUpdateUseCaseImpl`
**Repository：** `CardKeyPartsJpaRepository`（Spring Data JPA）

| Step | Type | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[VALIDATE]` | 驗證 `partSeq` ∈ {1, 2, 3}；不合法 → 拋出 `INVALID_KEY_FORMAT`（CA00012） | — |
| 2 | `[DOMAIN]` | 解析 `sourceKeyHex` → `byte[32]`；非十六進位字元或長度 ≠ 64 → 拋出 `INVALID_KEY_FORMAT`（CA00012） | `KeyParseService.hexToBytes()` |
| 3 | `[DOMAIN]` | 計算 KCV：`AES-256-ECB(sourceKeyBytes, byte[16]{0x00})[0..2]` → `kcvComputed`（6 個大寫十六進位字元） | `KcvService.compute()` |
| 4 | `[VALIDATE]` | 比較 `kcvComputed` 與 `rq.kcv`（大小寫不拘）；不符 → 清零 `sourceKeyBytes` 後拋出 `KCV_MISMATCH`（CA00011） | — |
| 5 | `[DOMAIN]` | 以 `MASTER_KEY`（AES-256-GCM）加密 `sourceKeyBytes` → `encryptedPart` | `CardEncryptionService.encryptKeyPart()` |
| 6 | `[DB WRITE]` | Upsert `card_key_parts`：`part_seq = partSeq`、`encrypted_part = encryptedPart`、`updated_at = now()` | `CardKeyPartsJpaRepository.upsert()` |
| 7 | `[DOMAIN]` | 重新載入 combineKey：讀取 `card_key_parts` 全部 3 筆，以 `MASTER_KEY` 逐一解密，逐位元組 XOR → 更新 `CombineKeyHolder` | `CombineKeyService.reload()` |
| 8 | `[DOMAIN]` | 清零所有中間明文位元組陣列（`sourceKeyBytes`、所有解密後的 Part、中間 XOR 緩衝區） | `KeyZeroizer.zeroFill()` |
| 9 | `[RETURN]` | 回傳 `SourceKeyUpdateRs`（`partSeq`、`kcv = kcvComputed`、`updatedAt = now()`） | — |

> **Step 4 失敗路徑：** 拋出 `KCV_MISMATCH` 之前，必須先清零 `sourceKeyBytes`。
> 注入失敗的非法金鑰不得殘留於 Heap 記憶體中。

> **Step 7 原子性：** `CombineKeyHolder` 對 `SecretKey` 參考執行記憶體內原子性替換。
> 輪換期間的並發解密操作可能短暫使用舊的 combineKey — 此為 Phase 5 的已知行為（參見 Guideline 14 Known Gaps）。

---

## 4. Database

### MySQL

| Table | Operation | 說明 |
|-------|-----------|------|
| `card_key_parts` | WRITE | 以 `part_seq` Upsert Source Key Part |
| `card_key_parts` | READ | Step 7 讀取全部 3 筆以重建 combineKey |

### Redis

> 此端點不執行任何 Redis 操作。

### Table Schema

#### card_key_parts

| Column | Type | Nullable | Default | 說明 |
|--------|------|----------|---------|------|
| part_seq | INT | NO | — | PK：Source Key 序號（1、2 或 3） |
| encrypted_part | VARCHAR(512) | NO | — | AES-256-GCM 加密的 256-bit Source Key；以 env/KMS 的 Master Key 加密 |
| updated_at | DATETIME(3) | NO | — | 最近一次注入或輪換的時間戳 |

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|---------|
| 400 | CA00012 | INVALID_KEY_FORMAT | `partSeq` 不在 {1, 2, 3} 中；或 `sourceKeyHex` 非 64 個有效十六進位字元 |
| 422 | CA00011 | KCV_MISMATCH | 提供的 `kcv` 與計算出的 KCV 不符 |
| 500 | INTERNAL_ERROR | 非預期系統錯誤 | 未處理的例外（如 MASTER_KEY 解密失敗、DB 錯誤） |
