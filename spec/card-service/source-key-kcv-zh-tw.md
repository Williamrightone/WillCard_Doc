# Source Key KCV 驗證 — card-service Spec

> **Service：** card-service
> **Contract Module：** card-contract
> **呼叫方：** 操作人員管理工具（僅限內部）
> **最後更新：** 2026-03

---

## 1. 概述

用於驗證操作人員提供的 KCV 是否與指定 Part 當前已存 Source Key 相符的內部 API。

操作人員提供預期的 KCV（6 個十六進位字元）。服務解密已存的 Source Key，獨立計算其 KCV，並回傳布林值表示是否相符。

回應中永遠不暴露原始 Source Key 或其計算出的 KCV — 僅回傳比對結果。
操作人員可藉此確認「系統目前載入的是不是我預期的金鑰？」而無需任何金鑰內容離開服務。

> 此端點僅限內部網路存取。Nginx 封鎖所有來自外部的 `/internal/` 路徑請求。

KCV 計算規格請參閱 [guideline/14-3ds-combine-key-zh-tw.md](../../guideline/14-3ds-combine-key-zh-tw.md)。

---

## 2. API Definition — card-service

### Endpoint

| 欄位 | 值 |
|------|----|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/internal/keys/source-key/{partSeq}/kcv/verify` |
| 實際接收路徑 | `/card/internal/keys/source-key/{partSeq}/kcv/verify` |
| 呼叫方 | 操作人員管理工具 |

### Path Variable

| 變數 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| partSeq | Integer | ✅ | 1、2 或 3 | Source Key Part 序號 |

### Request（`card.contract.dto.SourceKeyKcvVerifyRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| kcv | String | ✅ | @NotBlank、長度 = 6、有效十六進位 | 欲驗證的預期 KCV |

### Response（`card.contract.dto.SourceKeyKcvVerifyRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| partSeq | Integer | Source Key Part 序號 |
| matched | Boolean | `true`：提供的 KCV 與已存金鑰相符；`false`：不符 |
| updatedAt | String | ISO-8601 格式的最後金鑰更新時間戳 |

---

## 3. Service UseCase Flow

**UseCase：** `SourceKeyKcvVerifyUseCase.verify(int partSeq, SourceKeyKcvVerifyRq rq)`（card-service）
**UseCase Impl：** `SourceKeyKcvVerifyUseCaseImpl`
**Repository：** `CardKeyPartsJpaRepository`（Spring Data JPA）

| Step | Type | 說明 | Domain Service |
|------|------|------|----------------|
| 1 | `[VALIDATE]` | 驗證 `partSeq` ∈ {1, 2, 3}；不合法 → 拋出 `INVALID_KEY_FORMAT`（CA00012） | — |
| 2 | `[DB READ]` | `SELECT * FROM card_key_parts WHERE part_seq = partSeq`；查無資料 → 拋出 `KEY_PART_NOT_FOUND`（CA00010） | `CardKeyPartsJpaRepository.findByPartSeq()` |
| 3 | `[DOMAIN]` | 以 `MASTER_KEY`（AES-256-GCM）解密 `encrypted_part` → `sourceKeyBytes` | `CardEncryptionService.decryptKeyPart()` |
| 4 | `[DOMAIN]` | 計算 KCV：`AES-256-ECB(sourceKeyBytes, byte[16]{0x00})[0..2]` → `kcvComputed`（6 個大寫十六進位字元） | `KcvService.compute()` |
| 5 | `[DOMAIN]` | 清零 `sourceKeyBytes` | `KeyZeroizer.zeroFill()` |
| 6 | `[DOMAIN]` | `matched = kcvComputed.equalsIgnoreCase(rq.kcv)` | — |
| 7 | `[RETURN]` | 回傳 `SourceKeyKcvVerifyRs`（`partSeq`、`matched`、`updatedAt` 來自 DB 資料列） | — |

> **Step 5：** `sourceKeyBytes` 必須在 `try-finally` 區塊中清零，確保無論 Step 4 成功或失敗都一定執行清零動作。
>
> **KCV 不符不拋例外：** `matched = false` 是正常業務結果，不是例外狀況。呼叫方收到 HTTP 200 加上 `matched: false`。只有結構性錯誤（`partSeq` 非法、查無資料、解密失敗）才回傳非 200 狀態碼。

---

## 4. Database

### MySQL

| Table | Operation | 說明 |
|-------|-----------|------|
| `card_key_parts` | READ | 以 `part_seq` 查詢加密的 Source Key 資料列 |

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
| 400 | CA00012 | INVALID_KEY_FORMAT | `partSeq` 不在 {1, 2, 3} 中；或 `kcv` 非 6 個有效十六進位字元 |
| 404 | CA00010 | KEY_PART_NOT_FOUND | 給定 `partSeq` 在 `card_key_parts` 中查無資料 |
| 500 | INTERNAL_ERROR | 非預期系統錯誤 | 未處理的例外（如 MASTER_KEY 解密失敗） |
