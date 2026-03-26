# 卡片管理

本文件說明 WillCard 虛擬卡的生命週期管理、設計原則與合規規範。
涵蓋卡種、狀態機、PCI DSS 合規、加密策略與限額管理。
實作細節（API contract、DB schema、UseCase flow）請參閱 `spec/card-service/` 與 `spec/mobile-bff/` 下對應的 spec 文件。

---

## 1. 卡種

WillCard 提供四種虛擬信用卡。每種卡型擁有不同的回饋率、交易限額與特約優惠。

| 卡種 | 名稱 | 國內回饋 | 國外回饋 | 跨國手續費 | 單筆上限 | 單日上限 |
|------|------|---------|---------|-----------|---------|---------|
| CLASSIC | WillCard 經典卡 | 1% | 0% | 1.5% | NT$100,000 | NT$200,000 |
| OVERSEAS | WillCard 海外卡 | 0% | 5% | 1% | NT$100,000 | NT$200,000 |
| PREMIUM | WillCard 貴賓卡 | 2% | 5% | 1% | NT$200,000 | NT$500,000 |
| INFINITE | WillCard 無限卡 | 2% | 5% | 1% | 無上限 | 無上限 |

卡種屬性儲存於 `card_type_limit` 表作為**種子資料**（Flyway DML），**Runtime 唯讀**，不提供任何 API 修改。

**每人每卡種限一張：** 每位用戶每種卡型最多持有一張有效卡。若該用戶已有 PENDING、ACTIVE 或 FROZEN 狀態的相同卡種，申請將被拒絕。此限制由**應用層**執行（Service 層重複檢查），而非 DB 唯一約束，原因是條件為 `status != 'CANCELLED'`。

---

## 2. 卡片網路

每張卡可選擇三種卡組織之一：

| card_network | BIN 前綴 |
|-------------|---------|
| VISA | 4（例：4xxx-xxxx-xxxx-xxxx） |
| MASTERCARD | 51–55（例：5xxx-xxxx-xxxx-xxxx） |
| JCB | 35（例：35xx-xxxx-xxxx-xxxx） |

PAN 產生流程：依所選卡組織取 BIN 前綴，以 `SecureRandom` 填充剩餘位數，並以 **Luhn 演算法**驗證完整 16 位卡號。此為**模擬實作**，僅適用於開發與測試環境。

---

## 3. 卡片生命週期（狀態機）

```
PENDING ──► ACTIVE ──► FROZEN
               ▲          │
               └──────────┘
               │
ACTIVE  ──► CANCELLED
FROZEN  ──► CANCELLED
```

| 狀態 | 說明 |
|------|------|
| PENDING | 申請已收到，尚未啟用（預留給未來的審核流程） |
| ACTIVE | 卡片可正常使用 |
| FROZEN | 卡片暫時凍結，不允許新授權交易 |
| CANCELLED | 卡片永久取消，不可再啟用 |

**合法狀態轉移：**

| 來源 | 目標 | 觸發條件 |
|------|------|---------|
| PENDING | ACTIVE | 申請核准（MVP：直接核卡，省略審核） |
| ACTIVE | FROZEN | 用戶執行凍結 |
| FROZEN | ACTIVE | 用戶執行解凍 |
| ACTIVE | CANCELLED | 用戶或系統取消卡片 |
| FROZEN | CANCELLED | 用戶或系統取消卡片 |

`activated_at`：首次轉為 ACTIVE 時寫入，不可覆寫。
`cancelled_at`：轉為 CANCELLED 時寫入，不可覆寫。
兩欄位均保留作稽核用途，符合 PCI DSS 要求。

---

## 4. PCI DSS 合規原則

card-service 是唯一處理持卡人資料（CHD）的服務。其他所有服務僅使用遮罩或雜湊格式的資料。

### 靜態加密

下列欄位在寫入資料庫前以 **AES-256-GCM** 加密：

| 欄位 | 是否加密 | 原因 |
|------|---------|------|
| `pan_encrypted` | ✅ | 完整卡號 — PCI DSS 要求靜態加密 |
| `pan_masked` | ❌（直接儲存） | 已遮罩，僅保留末 4 碼 |
| `pan_hash` | ❌（HMAC-SHA256） | 不可逆，用於 PAN-based 查詢 |
| `cardholder_name_zh` | ✅ | 個人資料 |
| `cardholder_name_en` | ✅ | 個人資料（卡面印刷） |
| `expiry_date` | ✅ | 卡片資料（MMYY 格式） |

### 金鑰管理

- AES-256 加密金鑰及 `combineKey` 主金鑰透過**環境變數或 KMS** 注入。
- 金鑰**絕不**儲存在資料庫或程式碼中。
- `combineKey` 於服務啟動時載入 JVM Heap（`ApplicationReadyEvent`），關閉時清除（`@PreDestroy`）。
- 完整金鑰管理設計請參閱 Phase 5（combineKey 基礎設施）。

### 解密範圍

- 解密**僅在 card-service 內部**執行。
- 解密後的明文僅存在於 **JVM Heap 記憶體**中，不序列化、不記錄 Log、不傳入 Kafka payload。
- BFF 與其他服務絕不接收解密後的 PAN 或卡片資料。

### Log 過濾

- Logback filter 攔截所有 Log 輸出，以遮罩字串取代符合完整 PAN 或 CVV 格式的內容。
- DB 中的 `pan_encrypted` 不會出現在 Log，風險在於解密後的明文，filter 是最後一道防線。

---

## 5. CVV 政策

CVV 為申請卡片時產生的 3 位安全驗證碼。

| 規則 | 政策 |
|------|------|
| 產生方式 | 申請時以 `SecureRandom` 在 card-service JVM 中產生 |
| 儲存 | **絕不儲存** — 不儲存於 DB、Redis、Log 或任何持久層 |
| 交付 | 僅在 `card-apply` Response 中回傳**一次** |
| 補查 | 初次申請 Response 後無法再查詢 CVV |
| 授權流程 | CVV 包含於加密卡片 payload 中；僅在 card-service JVM Heap 解密；授權決策完成後立即清除 |

此政策確保符合 PCI DSS 要求（Requirement 3.2.1：授權後不得儲存敏感認證資料）。

---

## 6. 遮罩卡號慣例

所有顯示情境與 Event payload 均使用遮罩卡號格式：

```
****-****-****-1234
```

`pan_masked` 於申請時由完整卡號衍生並直接儲存（顯示時不需解密）。
Kafka Event payload 與 Log 記錄必須只使用 `maskedPan`。完整卡號絕不出現在 Kafka Event、Log 或 API Response 中（授權流程內部使用的 `pan_encrypted` 除外）。

---

## 7. PAN Hash 供授權查詢使用

Phase 7（Issuer 授權）需以 PAN 識別持卡人（「由 PAN 識別會員（pan_hash 查詢）」）。
為避免儲存明文 PAN，card-service 於申請時產生 `pan_hash = HMAC-SHA256(pan, hmacKey)`。

- `pan_hash` 儲存於 `card` 表，建立**唯一索引**。
- 授權時，NCCC 提供加密後的 PAN；card-service 在 JVM Heap 解密後計算 hash，再執行 DB 查詢。
- `hmacKey` 的管理方式與 AES 加密金鑰相同（環境變數 / KMS；不存入 DB）。

---

## 8. 回饋率結構

交易回饋率採三層疊加設計：

```
final_rate = base_rate + mcc_bonus + merchant_bonus
```

| 層次 | 來源資料表 | 定義於 |
|------|----------|-------|
| `base_rate` | `card_type_limit.domestic_reward_rate` 或 `overseas_reward_rate` | Phase 1 |
| `mcc_bonus` | `reward_plan`（依 MCC，可限定卡種） | Phase 3 |
| `merchant_bonus` | `card_type_merchant_benefit`（特約商家額外加成） | Phase 1 |

`card_type_merchant_benefit` 儲存特定 `card_type` 搭配 `merchant_id` 或 `mcc_code` 的額外回饋率，作為**種子資料**（Flyway DML）管理，Runtime 唯讀。

---

## 9. 交易限額管理

| 限額類型 | 儲存位置 | 範圍 | 備注 |
|---------|---------|------|------|
| 單筆交易限額 | `card_type_limit.single_txn_limit`（DB，BIGINT 分） | 每卡種 | NULL = 無上限 |
| 單日累計限額 | `card:daily:{cardId}:{yyyyMMdd}`（Redis，BIGINT 分） | 每張卡每自然日 | TTL = 當日結束（86400s） |

- **INFINITE** 卡種的兩項限額均為 NULL，跳過所有限額檢查。
- 每日限額追蹤在授權時（Phase 7）以原子方式累加。Phase 1 僅定義 Redis Key 結構，卡片管理階段不進行寫入。

---

## 10. 冪等性與 CVV 衝突

`card-apply` **不採用**標準 `Idempotency-Key` 機制。

**原因：** Apply Response 中的一次性 CVV 不可快取於 Redis（`idempotency:{key}` 的 value 包含 `responseBody`），將 CVV 儲存於 Redis 違反 PCI DSS Requirement 3.2.1。

**重複防護：** 服務端唯一性驗證（`CARD_ALREADY_EXISTS`，CA00009）已防止同卡種重複申請。Client 必須將 `CARD_ALREADY_EXISTS` 視為終止性失敗，不得重試 apply 請求。

其他卡片管理端點（freeze、unfreeze）依 spec guideline 定義不屬於金融操作，亦不需要 `Idempotency-Key`。

---

## 11. 已知待補項目（Phase 1 範圍外）

下列項目超出 Phase 1 範圍，應於後續 Phase 補充：

| 待補項目 | 說明 |
|---------|------|
| 卡片取消 API | 狀態機包含 CANCELLED，但 Phase 1 未定義取消 API |
| 卡片清單 API | 缺少 `GET /api/v1/card/list`；Client 須自行保存 apply response 中的 `cardId` |
| 持卡人姓名來源 | Phase 1 要求 Client 在申請時提供雙語姓名；未來應由會員 Profile 自動帶入 |

---

下一章：[交易流程](/guideline/10-txn-flow.md)
