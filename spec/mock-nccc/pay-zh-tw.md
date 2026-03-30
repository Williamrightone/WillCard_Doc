# 付款 — Mock NCCC Spec

> **服務：** mock-nccc
> **環境：** 僅限開發環境
> **Contract Module：** mock-nccc-contract
> **呼叫方：** [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md)
> **最後更新：** 2026-03

---

## 1. 概述

模擬 NCCC（財金聯合信用卡處理中心）卡組織在付款發起腿的行為。

接收來自 ORCH 的付款請求，向 card-service 內部明文端點取得卡片資料，使用 `MOCK_COMBINE_KEY`（AES-256-GCM）組裝 `encryptedCard`，再呼叫 card-service auth/authorize 執行驗卡與 OTP 生成。

回傳 `challengeRef` 給 ORCH。

> **僅限開發：** `MOCK_COMBINE_KEY` 必須與 card-service 的 `combineKey`（三把 source key XOR 合成）一致。金鑰從 `application-dev.yml` 在啟動時載入至 `MockCombineKeyHolder`——不需要資料庫。

---

## 2. 跨規格介面契約

| 介面 | 規格 |
|------|------|
| 呼叫方（ORCH） | [txn-orch/card-pay-zh-tw.md](../txn-orch/card-pay-zh-tw.md) |
| Mock NCCC → card-service（明文卡片資料） | [card-service/card-plain-zh-tw.md](../card-service/card-plain-zh-tw.md) |
| Mock NCCC → card-service（auth/authorize） | [card-service/card-auth-authorize-zh-tw.md](../card-service/card-auth-authorize-zh-tw.md) |
| 第二腿（OTP 確認） | [mock-nccc/pay-confirm-zh-tw.md](pay-confirm-zh-tw.md) |

---

## 3. API 定義 — mock-nccc

### Endpoint

| 欄位 | 值 |
|------|---|
| Method | POST |
| context-path | /mock-nccc |
| Controller mapping | /pay |
| 實際接收路徑 | /mock-nccc/pay |
| 呼叫方 | ORCH（MockNcccFeignClient） |

### Request（`mock-nccc.contract.dto.MockNcccPayRq`）

| 欄位 | 型別 | 必填 | 驗證 | 說明 |
|------|------|------|------|------|
| cardId | String | ✅ | @NotBlank | 卡片 Snowflake ID（String 格式） |
| amount | Long | ✅ | @NotNull; @Min(1) | 原始交易金額（分） |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 幣別代碼 |
| txnTwdBaseAmount | Long | ✅ | @NotNull; @Min(1) | FX 換算後的台幣基準金額；由 ORCH 計算 |
| merchantId | String | ✅ | @NotBlank | 商戶識別碼 |

### Response（`mock-nccc.contract.dto.MockNcccPayRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| challengeRef | String | UUID；OTP Session 綁定；由 card-service auth/authorize 回傳 |

---

## 4. Service UseCase Flow

**UseCase：** `MockNcccPayUseCase.pay(MockNcccPayRq)`（mock-nccc）

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DOMAIN]` | 從 `MockCombineKeyHolder` 取得 `MOCK_COMBINE_KEY`；為 null → 拋出 503 MOCK_KEY_UNAVAILABLE |
| 2 | `[FEIGN]` | `CardServiceFeignClient.getPlainCard(cardId)` → `GET /card-service/internal/cards/{cardId}/plain`；Response：`{ pan, expiryMMYY, cvvItoken }`；見 [card-service/card-plain-zh-tw.md](../card-service/card-plain-zh-tw.md) |
| 3 | `[DOMAIN]` | AES-256-GCM 加密 `{ pan, expiryMMYY, cvvItoken }` 使用 `MOCK_COMBINE_KEY` → `encryptedCard`（try-finally：加密後清零 `pan` 位元組陣列） |
| 4 | `[FEIGN]` | `CardServiceFeignClient.authorize(CardAuthAuthorizeRq)` → `POST /card-service/auth/authorize`；Request：`{ encryptedCard, amount, currency, txnTwdBaseAmount, merchantId }`；Response：`{ challengeRef }`；見 [card-service/card-auth-authorize-zh-tw.md](../card-service/card-auth-authorize-zh-tw.md) |
| 5 | `[RETURN]` | 回傳 `MockNcccPayRs { challengeRef }` |

> Step 4 可能拋出 CA000xx 錯誤（驗卡失敗或限額超過），將透傳至 ORCH。

---

## 5. 啟動設定 — MockCombineKeyHolder

**觸發：** `ApplicationReadyEvent`

| 步驟 | 說明 |
|------|------|
| 1 | 讀取 `application-dev.yml` 中的 `willcard.security.mock-combine-key`（64 位十六進位字元 = 256-bit AES 金鑰） |
| 2 | 驗證：必須為恰好 64 個有效十六進位字元 |
| 3 | 解碼十六進位字串 → 32 位元組陣列 → 儲存至 `MockCombineKeyHolder` |
| 4 | 若為空或格式無效 → `MockCombineKeyHolder.key = null`；後續所有 UseCase 呼叫回傳 503 MOCK_KEY_UNAVAILABLE |

**`application-dev.yml` 設定：**

```yaml
willcard:
  security:
    mock-combine-key: "0000000000000000000000000000000000000000000000000000000000000000"
    # 64 位十六進位（256-bit）；必須與 card-service combineKey（三把 source key XOR 結果）相同
```

> `MOCK_COMBINE_KEY` 為開發環境密鑰。不得出現在正式環境設定中，也不得在未使用環境特定 override 的情況下提交至版本控制。

---

## 6. 錯誤碼

| HTTP 狀態 | 錯誤碼 | 說明 | 觸發條件 |
|---------|--------|------|---------|
| 503 | MOCK_KEY_UNAVAILABLE | Mock combine key 未載入 | 啟動時 `MockCombineKeyHolder.key = null` |
| 409 | CA00002 | CARD_NOT_ACTIVE | 透傳自 card-service auth/authorize |
| 409 | CA00003 | CARD_EXPIRED | 透傳自 card-service auth/authorize |
| 409 | CA00004 | CARD_FROZEN | 透傳自 card-service auth/authorize |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | 透傳自 card-service auth/authorize |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | 透傳自 card-service auth/authorize |
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | 透傳自 card-service（combineKey 未載入） |
| 500 | INTERNAL_ERROR | 系統錯誤 | 未預期例外 |

---

## 7. Changelog

### v1.0 — 2026-03 — 初始版本
