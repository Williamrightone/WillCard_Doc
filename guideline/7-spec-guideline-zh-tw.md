# Spec Guideline

本文件定義 WillCard 後端所有 API Spec 的撰寫規範與閱讀方式。  
所有 API Spec 文件須遵循本指引的結構、命名與分層規則。

---

## 1. 文件位置與命名

### 1.1 目錄結構

```cmd
guideline/
├── 7-spec-guideline.md       ← 本文件（撰寫規範）
└── spec/
    ├── bff/
    │     └── login.md        ← BFF spec（對外 API 入口，連結到各 biz spec）
    ├── auth-service/
    │     └── login.md        ← biz spec（contract 定義、UseCase、DB）
    ├── card-service/
    │     └── ...
    └── wallet-service/
          └── ...
```

### 1.2 命名規則

- 文件名稱使用小寫 kebab-case，例如 `login.md`、`top-up.md`
- 每個 service 一個資料夾，資料夾名稱與 Maven module 名稱一致
- 一個 Use Case 對應一份 BFF spec，BFF spec 再連結到它所呼叫的一份或多份 biz spec
- 同一個 biz spec 可被多個 BFF spec 連結（contract 重用）

### 1.3 文件職責邊界

| 文件類型 | 職責 |
|----------|------|
| **BFF spec** | 對外 API 定義、BFF UseCase Flow、Operation Log Level、BFF 層 Error Codes |
| **biz spec** | Service 內部 API 定義（含 contract Rq/Rs）、Service UseCase Flow、Database、Service 層 Error Codes |

---

## 2. BFF Spec 文件結構

```cmd
1. Overview
2. Operation Log Level
3. API Definition — BFF
4. BFF UseCase Flow
5. Error Codes
```

---

## 3. Biz Spec 文件結構

```cmd
1. Overview
2. API Definition — {service-name}
3. Service UseCase Flow
4. Database
5. Error Codes
```

---

## 4. 各章節撰寫規則

### 4.1 Overview

簡述這個 API 的業務目的、誰呼叫它、核心行為是什麼。  
不超過 5 行。不寫實作細節。  
BFF spec 與 biz spec 的 Overview 分別描述各自層的職責。

```markdown
## Overview

（BFF spec 範例）
接收 Client 的帳號密碼登入請求，進行頻率限制檢查後，轉發至 auth-service 進行身份驗證。
驗證通過後將 JWT 設入 Response Header 回傳，並發布 Operation Log 事件。

（biz spec 範例）
驗證使用者帳號密碼，簽發 JWT，並建立 Redis Session。
Token 回傳至 BFF，不直接暴露給 Client。
```

---

### 4.2 Operation Log Level（僅 BFF spec）

標注此 API 的 Operation Log 等級，說明觸發條件與 Kafka Topic。

| 等級 | 定義 | Log 行為 |
|------|------|----------|
| L1 | 涉及金流、身份驗證、安全操作 | BFF publish Kafka event → Audit Consumer 寫 DB |
| L2 | 資料變更（非金流） | BFF publish Kafka event → Audit Consumer 寫 DB |
| L3 | 一般查詢 | 只寫 EFK，不進 Kafka，不入 DB |

```markdown
## Operation Log Level

**等級：L1**  
觸發時機：每次呼叫 login API，無論成功或失敗皆須記錄。  
Kafka Topic：`operation-log.auth.login`  
儲存欄位：userId（失敗時為 null）、userAccount、result（SUCCESS / FAIL）、failReason、ip、timestamp
```

---

### 4.3 API Definition — BFF

描述 BFF 對外暴露的 API。

**路徑規則：**

- Client 送出：`/api/v{n}/{resource}/{action}`
- Nginx strip `/api/v{n}` 後，BFF 實際接收：`/{resource}/{action}`
- BFF 無 context-path，版本號統一由 Nginx 管理
- **Controller 內永遠只寫無版本號的路徑**，版本由 Nginx 路由決定

**版本升級流程：**

正常狀態下，Controller 只有一個路徑：

```cmd
Controller: /auth/login        ← Nginx v1 指向此路徑
```

需要 breaking change 時，暫時並存兩個路徑：

```cmd
Controller: /auth/login        ← 繼續服務舊版 App（Nginx v1）
Controller: /auth/login-v2     ← 新版暫用命名（Nginx v2）
```

v2 測試完畢、舊版 App 下架後，完成切換：

```cmd
Controller: /auth/login        ← 改回標準命名（Nginx v1 / v2 皆指向此路徑）
舊 /auth/login-v2              ← 刪除
```

> 版本升級屬於過渡期行為，`-v2` 後綴為臨時命名，不應長期存在於 codebase。

```markdown
## API Definition — BFF

### Endpoint
| 項目 | 內容 |
|------|------|
| Method | POST |
| Client URL | /api/v1/auth/login |
| BFF 接收路徑 | /auth/login |
| Auth Required | No（Public endpoint） |

### Request Body
| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| userAccount | String | ✅ | @NotBlank | 使用者帳號 |
| passwd | String | ✅ | @NotBlank | 使用者密碼 |

### Response Body
| 欄位 | 型別 | 說明 |
|------|------|------|
| userId | String | 使用者 ID |
| userAccount | String | 使用者帳號 |
| name | String | 使用者姓名 |

### Response Header
| Header | 說明 |
|--------|------|
| Authorization | Bearer {accessToken} |
```

---

### 4.4 API Definition — {service-name}（biz spec）

描述 BFF 透過 Feign 呼叫的 Service 內部 API，同時定義 contract module 的 Rq/Rs。

**路徑規則：**

- 各 service 設定 `server.servlet.context-path: /{service-name}`
- Controller mapping 只含動詞，不含 service 名稱
- 實際接收路徑 = `/{service-name}/{action}`
- 版本號由 Nginx 統一管理，service 層不含版本號

```markdown
## API Definition — auth-service

### Endpoint
| 項目 | 內容 |
|------|------|
| Method | POST |
| context-path | /auth |
| Controller mapping | /login |
| 實際接收路徑 | /auth/login |
| 呼叫方 | BFF（FeignClient） |

### Request（LoginRq — auth-contract）
| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| userAccount | String | ✅ | @NotBlank | 使用者帳號 |
| passwd | String | ✅ | @NotBlank | 使用者密碼 |

### Response（LoginRs — auth-contract）
| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| userId | String | 使用者 ID | |
| userAccount | String | 使用者帳號 | |
| name | String | 使用者姓名 | |
| accessToken | String | JWT Token | @JsonIgnore，僅供 BFF 讀取，不序列化至 Response Body |
```

---

### 4.5 BFF UseCase Flow

描述 BFF UseCase 的執行步驟。  
`[FEIGN]` 步驟須直接連結到對應的 biz spec 文件。

**行為類型標籤：**

| 標籤 | 說明 |
|------|------|
| `[VALIDATE]` | 輸入驗證 |
| `[REDIS READ]` | 讀取 Redis |
| `[REDIS WRITE]` | 寫入 Redis |
| `[FEIGN]` | 呼叫下游 Service（須附連結） |
| `[HEADER]` | 設定 Response Header |
| `[KAFKA]` | 發布 Kafka Event |
| `[RETURN]` | 回傳 Response |

```markdown
## BFF UseCase Flow

**UseCase：** `LoginUseCase.login(LoginRq)`

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[REDIS READ]` | 檢查 `login:lock:{userAccount}` 是否存在 | key 存在 → 拋出 ACCOUNT_LOCKED |
| 2 | `[FEIGN]` | 呼叫 auth-service → [auth-service/login.md](../auth-service/login.md) | 回傳錯誤 → 執行步驟 3 |
| 3 | `[REDIS WRITE]` | 登入失敗：`login:attempt:{userAccount}` +1，TTL 1200s；達 3 次寫入 `login:lock:{userAccount}`，TTL 1200s | — |
| 4 | `[HEADER]` | 從 LoginRs 取出 accessToken，設入 `Authorization: Bearer {token}` | — |
| 5 | `[KAFKA]` | Publish `operation-log.auth.login` event（非同步，失敗不影響主流程） | — |
| 6 | `[RETURN]` | 回傳 LoginRs（不含 accessToken） | — |
```

---

### 4.6 Service UseCase Flow（biz spec）

描述 Service UseCase 的執行步驟，定義到 Domain Service 層的介面呼叫。  
Domain Service 內部實作細節不在 spec 範圍內，由 PG 自行決定。

**行為類型標籤：**

| 標籤 | 說明 |
|------|------|
| `[DB READ]` | 讀取 MySQL |
| `[DB WRITE]` | 寫入 MySQL |
| `[REDIS WRITE]` | 寫入 Redis |
| `[DOMAIN]` | 呼叫 Domain Service 方法 |
| `[RETURN]` | 回傳 Response |

```markdown
## Service UseCase Flow

**UseCase：** `LoginUseCase.login(LoginRq)`（auth-service）

| 步驟 | 類型 | 說明 | 負責的 Domain Service |
|------|------|------|-----------------------|
| 1 | `[DB READ]` | 依 userAccount 查詢使用者 | `UserQueryService.findByAccount()` |
| 2 | `[DOMAIN]` | Bcrypt 驗證 passwd 與 DB 雜湊值 | `AuthService.verifyPassword()` |
| 3 | `[DOMAIN]` | 生成 JWT（payload：jti、userId、userAccount、role、exp） | `TokenService.generate()` |
| 4 | `[REDIS WRITE]` | 寫入 `usession:{userId}`（jti、userId、userAccount、role、loginAt），TTL 3600s | `SessionService.createSession()` |
| 5 | `[RETURN]` | 回傳 LoginRs（含 accessToken，供 BFF 讀取） | — |
```

---

### 4.7 Database（biz spec）

列出此 API 會讀寫的 Table 與 Redis key，並附上 Table Schema。  
完整 DDL 請參閱 `4-data-mgmt.md`。

```markdown
## Database

### MySQL

| Table | 操作 | 說明 |
|-------|------|------|
| `user` | READ | 依 userAccount 查詢使用者基本資料與密碼雜湊 |
| `operation_log` | WRITE | 由 Audit Consumer 監聽 Kafka 後非同步寫入 |

### Redis

| Key Pattern | 操作 | TTL | 說明 |
|-------------|------|-----|------|
| `usession:{userId}` | WRITE | 3600s | 使用者 Session |

### Table Schema

#### user
| 欄位 | 型別 | Nullable | 說明 |
|------|------|----------|------|
| id | BIGINT | NO | PK，自增 |
| user_id | VARCHAR(36) | NO | UUID，業務用 ID，Unique |
| user_account | VARCHAR(100) | NO | 登入帳號，Unique |
| passwd | VARCHAR(255) | NO | Bcrypt 雜湊值 |
| name | VARCHAR(100) | NO | 使用者姓名 |
| role | VARCHAR(20) | NO | 使用者角色（USER / ADMIN） |
| status | VARCHAR(20) | NO | 帳號狀態（ACTIVE / DISABLED） |
| created_at | DATETIME | NO | 建立時間 |
| updated_at | DATETIME | NO | 最後更新時間 |
```

---

### 4.8 Error Codes

**BFF spec** 列出對外回傳的錯誤碼（Client 可見）。  
**biz spec** 列出 Service 內部拋出的錯誤碼（BFF 收到後可能轉換或直接透傳）。  
格式需與 `5-exception.md` 的錯誤碼規範一致。

```markdown
## Error Codes

（BFF spec 範例）
| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | userAccount 或 passwd 為空 |
| 401 | INVALID_CREDENTIALS | 帳號或密碼錯誤 | auth-service 回傳驗證失敗 |
| 403 | ACCOUNT_LOCKED | 帳號已鎖定 | login:lock key 存在 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |

（biz spec 範例）
| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 404 | USER_NOT_FOUND | 使用者不存在 | userAccount 查無資料 |
| 401 | INVALID_CREDENTIALS | 密碼驗證失敗 | Bcrypt 不符合 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |
```

---

## 5. 冪等設計（Idempotency）

### 5.1 適用範圍

所有**產生副作用的金融操作** POST 請求（交易、儲值、兌換等）**必須**帶入 `Idempotency-Key` header。
非金融或唯讀的 POST（如 login）不適用。

### 5.2 機制

前端生成 UUID v4，放入 `Idempotency-Key` request header。
BFF 在執行業務邏輯前檢查 Redis：

```
前端生成 UUID v4 → 放入 Idempotency-Key header

BFF 收到請求後：
  1. SETNX  idempotency:{key}  →  { status: PROCESSING, createdAt }（短 TTL，如 30–60 秒）
  2. 若 SETNX 失敗且 status = COMPLETED  → 直接回傳快取 response（不重複執行）
  3. 若 SETNX 失敗且 status = PROCESSING  → 拋出 409 Conflict
  4. 若 SETNX 成功                        → 執行業務邏輯
  5. 執行成功 → 更新為 COMPLETED，快取 response（較長 TTL，如 24 小時）
  6. 執行失敗 → 刪除 key（允許 client 用同一個 Idempotency-Key 重試）
```

> **SETNX（SET NX）** 確保原子性——同一時間只有一個請求能取得該 key，防止重複請求同時到達時的競態條件。

### 5.3 Redis Key 與 Value

| 項目 | 內容 |
|------|------|
| Key pattern | `idempotency:{Idempotency-Key}` |
| PROCESSING TTL | 短（如 30–60 秒）——防止 server crash 後 key 永久鎖定 |
| COMPLETED TTL | 較長（如 24 小時）——決定快取回應的重試窗口 |

**Value 結構：**

| 欄位 | 型別 | 說明 |
|------|------|------|
| status | String | `PROCESSING` / `COMPLETED` |
| httpStatus | int | 快取回應的原始 HTTP status code |
| responseBody | String | 序列化的回應 JSON |
| createdAt | long | key 建立時間（epoch millis） |

### 5.4 失敗處理

| 場景 | 行為 |
|------|------|
| 業務邏輯成功 | 更新 status 為 `COMPLETED`，快取 response |
| 業務邏輯失敗 | **刪除 key**——client 可用同一個 `Idempotency-Key` 重試 |
| PROCESSING 狀態被另一請求命中 | 回傳 **409 Conflict** |
| PROCESSING key 過期（TTL） | key 自動釋放——下一個請求重新開始 |

### 5.5 Idempotency-Key vs txnRef

| | `Idempotency-Key` | `txnRef` |
|---|---|---|
| 用途 | 防止 BFF 層 HTTP 重複請求 | OTP 兩步驟流程中 Saga 的 session 識別符 |
| 產生方 | Client（UUID v4） | Server（`card-pay` 第一步產生） |
| 作用範圍 | 單一 HTTP 請求 | 整個 Saga 生命週期（如 `card-pay` → `card-pay/confirm`） |
| 儲存位置 | Redis（`idempotency:{key}`） | Saga orchestrator state |

### 5.6 BFF Spec 撰寫指引

當 BFF spec 需要冪等時：

1. **API Definition** — 在 Request Header 表格中加入 `Idempotency-Key`：

```markdown
### Request Header
| Header | 必填 | 說明 |
|--------|------|------|
| Idempotency-Key | ✅ | UUID v4，由前端生成 |
```

2. **BFF UseCase Flow** — 將冪等檢查作為第一個步驟：

```markdown
| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | [REDIS WRITE] | SETNX idempotency:{Idempotency-Key}，標記 PROCESSING | key 已存在且 COMPLETED → 回傳快取；key 已存在且 PROCESSING → 拋出 409 |
| ... | ... | （後續業務邏輯步驟） | ... |
| N | [REDIS WRITE] | 更新 idempotency:{Idempotency-Key} 為 COMPLETED，快取 response | 業務失敗 → 刪除 key |
```

---

## 6. 快速檢查清單

### BFF Spec

- [ ] Overview 說明 BFF 層的業務目的
- [ ] Operation Log Level 已標注等級、Kafka Topic 與儲存欄位
- [ ] API Definition 路徑符合 Nginx strip 規則（無版本號）
- [ ] BFF UseCase Flow 每步驟有標籤與失敗處理
- [ ] `[FEIGN]` 步驟已連結到對應的 biz spec
- [ ] Error Codes 列出對外錯誤碼
- [ ] 需要冪等的 API 已在 Request Header 宣告 `Idempotency-Key`，且 UseCase Flow 包含冪等檢查步驟

### Biz Spec

- [ ] Overview 說明 Service 層的業務職責
- [ ] API Definition 路徑符合 context-path 規則（無版本號）
- [ ] contract Rq/Rs 欄位定義完整，`@JsonIgnore` 欄位已備註
- [ ] Service UseCase Flow 定義到 Domain Service 介面，不寫實作細節
- [ ] Database 章節包含 Table 清單、Redis key 清單與 Table Schema
- [ ] Error Codes 列出 Service 內部錯誤碼

---

## 7. 閱讀建議

**開發者（BFF）閱讀順序：**

1. `bff/{feature}.md` → Overview、API Definition、BFF UseCase Flow
2. 跟著 `[FEIGN]` 連結 → 閱讀對應的 biz spec 確認 contract 定義

**開發者（biz service）閱讀順序：**

1. `{service}/{feature}.md` → Overview、API Definition（contract Rq/Rs）
2. Service UseCase Flow → Database

**架構師 / Review 閱讀順序：**

1. `bff/{feature}.md` → 全覽流程與職責邊界
2. 各 biz spec → 確認 contract 定義與 DB 設計
