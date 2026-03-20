# PCI DSS 合規規範

PCI DSS（Payment Card Industry Data Security Standard）是國際支付卡產業資料安全標準，所有儲存、處理或傳輸持卡人資料的系統均須遵循。WillCard 作為信用卡交易處理系統，須符合相關要求。

---

## 1. 持卡人資料定義

**持卡人資料（Cardholder Data，CHD）：**

| 資料類型 | 說明 | 是否允許儲存 |
|---|---|---|
| PAN（主帳號，卡號） | 16 位數信用卡號 | 允許，但須加密或遮罩 |
| 持卡人姓名 | 卡片上的姓名 | 允許 |
| 到期日 | 卡片到期月年 | 允許 |
| CVV / CVC | 卡片背面三位數驗證碼 | **絕對禁止儲存**，授權後立即清除 |
| PIN | 個人識別碼 | **絕對禁止儲存** |

---

## 2. 系統各層的 PCI DSS 處理方式

### 2.1 網路層（Network Segmentation）

```
外網
  └── BFF（DMZ 區）
        └── 內網 / k8s cluster
              ├── Domain Services（不對外暴露）
              ├── Core Services（不對外暴露）
              └── Middleware（獨立 VM，最小化存取權限）
```

- Domain Service、Ledger、Orchestrator 所有 port 僅在 k8s cluster 內網可達
- Middleware VM（MySQL、Redis、Kafka）僅允許 k8s worker node IP 連線
- 所有對外通訊強制 TLS 1.2+

---

### 2.2 卡片資料加密（Data Encryption）

**NCCC 加密流程：**

在台灣，持卡人卡片資訊由 NCCC 接收後使用 combineKey 加密，WillCard 收到的 `encryptedCard` 已是密文。WillCard 與 NCCC 協商並共用 combineKey，加密責任在 NCCC 端，WillCard 只執行解密。

**combineKey 產生與管理：**

三個 sourceKey 儲存於同一張 `card_key_parts` table 的三個欄位，各欄位以 **AES-256 加密**儲存（可還原）。加密 sourceKey 所用的 AES 金鑰由環境變數注入或 KMS 管理，不儲存於 DB。任一欄位單獨洩漏均無法還原完整 combineKey。

> **TODO：** 補充 combineKey 衍生演算法說明（XOR / HKDF 等），詳見金鑰管理章節。

> **TODO：** Prod 環境金鑰管理規範（KMS 選型與實作）詳見金鑰管理章節。本文件僅說明原則：prod 強制使用 KMS，環境變數僅允許用於 local / dev。

Card Service 啟動時（`ApplicationReadyEvent`）從 DB 讀取並解密三個 part，組合並衍生出 combineKey，常駐 JVM heap。combineKey 明文不序列化、不落地、不出現在任何網路封包。

**解密後卡片資料處理：**

```
NCCC 加密的 encryptedCard
  → Card Service 以記憶體內 combineKey 解密
  → 解密結果僅存 JVM heap
  → 授權完成後立即從記憶體清除
  → CVV 絕不寫入任何持久化儲存
```

**資料庫中卡片資料儲存規範：**

| 欄位 | 儲存方式 |
|---|---|
| 卡號 PAN | 僅儲存遮罩格式：`****-****-****-1234` |
| 到期日 | AES-256 加密後儲存 |
| 持卡人姓名 | AES-256 加密後儲存 |
| CVV | **不儲存**，授權後立即丟棄 |

---

### 2.3 傳輸安全（Data in Transit）

- 所有對外 API 強制 HTTPS，TLS 1.2 最低版本，建議 TLS 1.3
- 服務間通訊在 k8s cluster 內網，建議啟用 mTLS（可使用 Istio service mesh）
- Kafka、Redis 連線配置 TLS

---

### 2.4 Log 規範（Log Sanitization）

所有 application log 禁止出現以下資料：

| 禁止記錄的資料 | 處理方式 |
|---|---|
| 完整卡號 PAN | 僅記錄 `****-****-****-1234` |
| CVV / CVC | 不得出現在任何 log |
| combineKey 或任何加密金鑰 | 不得出現在任何 log |
| 原始 OTP 碼 | 只記錄 `OTP_REQUESTED` 事件，不記錄碼值 |
| JWT 完整內容 | 只記錄 `jti`（JWT ID） |

---

### 2.5 Kafka Event Payload 規範

Kafka 事件中的持卡人資料欄位規範：

| 欄位 | 規範 |
|---|---|
| 卡號 | 僅傳遞 `maskedCardNumber`（`****-****-****-1234`） |
| 金額 | 允許傳遞 |
| userId | 允許傳遞 |
| merchantId | 允許傳遞 |
| CVV | **禁止出現在任何 event payload** |

---

### 2.6 稽核軌跡（Audit Trail）

- 所有信用卡相關操作（OTP 請求、OTP 驗證、交易授權、卡片狀態變更）均寫入 Audit Log
- Audit Log 為 append-only，不得修改或刪除
- 保留期限最少 12 個月線上 + 12 個月離線歸檔

> **TODO：** Audit Log 儲存位置、表結構與 append-only 實作方式詳見 Audit Trail 章節。

---

### 2.7 存取控制（Access Control）

> **TODO：** 詳見存取控制章節。規劃方向：
> - 各 service 使用獨立 DB 帳號，不共用，最小權限原則
> - 僅 card-service 有權存取 card 相關資料表，其他服務透過 API 呼叫
> - K8s Secret 管理 DB 帳號密碼，不寫入程式碼或 config 檔

下一章: [Spec 撰寫規範](/guideline/7-spec-guideline-zh-tw.md)