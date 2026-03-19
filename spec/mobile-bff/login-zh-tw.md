# Login — BFF Spec

> **所屬服務：** bff  
> **關聯 biz spec：** [auth-service/login.md](../auth-service/login.md)  
> **最後更新：** 2026-03

---

## 1. Overview

接收 Client 的帳號密碼登入請求。  
依序執行帳號鎖定檢查、轉發至 auth-service 進行身份驗證。  
驗證通過後，將 JWT 設入 Response Header 回傳給 Client，並非同步發布 Operation Log 事件。  
BFF 本身不執行密碼驗證，不簽發 Token，僅負責流程協調與 Header 組裝。

---

## 2. Operation Log Level

**等級：L1**  
**觸發時機：** 每次呼叫 login API，無論成功或失敗皆須記錄。  
**Kafka Topic：** `operation-log.auth.login`

| 儲存欄位 | 型別 | 說明 |
|----------|------|------|
| memberId | BIGINT | 登入成功時的 Snowflake ID；失敗時為 null |
| account | VARCHAR | 登入帳號 |
| result | ENUM | SUCCESS / FAIL |
| failReason | VARCHAR | 失敗原因；成功時為 null |
| ip | VARCHAR | Client IP |
| timestamp | DATETIME | 事件發生時間 |

---

## 3. API Definition — BFF

### Endpoint

| 項目 | 內容 |
|------|------|
| Method | POST |
| Client URL | /api/v1/auth/login |
| BFF 接收路徑 | /auth/login |
| Auth Required | No（Public endpoint，Spring Security permitAll） |

### Request Body（`application.api.dto.LoginRq`）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| account | String | ✅ | @NotBlank | 登入帳號 |
| passwd | String | ✅ | @NotBlank | 登入密碼 |

### Response Body（`application.api.dto.LoginRs`）

| 欄位 | 型別 | 說明 |
|------|------|------|
| memberId | String | Snowflake ID（轉為 String 防止前端精度遺失） |
| account | String | 登入帳號 |
| name | String | 顯示名稱 |

### Response Header

| Header | 說明 |
|--------|------|
| Authorization | Bearer {accessToken} |

---

## 4. BFF UseCase Flow

**UseCase：** `LoginUseCase.login(LoginRq)`  
**UseCase Impl：** `LoginUseCaseImpl`  
**Adapter：** `AuthClientAdapter` implements `AuthClientPort`  
**Feign：** `AuthFeignClient`（使用 `auth-contract` 的 `LoginRq` / `LoginRs`）

### 轉換說明

BFF 的 `LoginRq`（`application.api.dto`）與 auth-contract 的 `LoginRq`（`auth.contract.dto`）同名但 package 不同。  
`LoginAssembler.toAuthRq(LoginRq)` 負責在 UseCaseImpl 內進行轉換，避免兩個同名類別直接出現在同一個方法內。

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[REDIS READ]` | 檢查 `login:lock:{account}` 是否存在 | key 存在 → 拋出 `ACCOUNT_LOCKED`，中止流程 |
| 2 | `[VALIDATE]` | `LoginAssembler.toAuthRq(rq)` 將 BFF LoginRq 轉換為 auth-contract LoginRq | — |
| 3 | `[FEIGN]` | `AuthClientAdapter` 呼叫 auth-service → [auth-service/login.md](../auth-service/login.md) | auth-service 回傳業務錯誤 → 執行步驟 4；系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 4 | `[REDIS WRITE]` | 登入失敗時：`login:attempt:{account}` 計數 +1，TTL 1200s；計數達 3 時，另寫入 `login:lock:{account}`，TTL 1200s | — |
| 5 | `[HEADER]` | 登入成功：從 auth-contract LoginRs 取出 `accessToken`，設入 `Authorization: Bearer {token}` | — |
| 6 | `[KAFKA]` | Publish `operation-log.auth.login` event（非同步，失敗不影響主流程） | 記錄錯誤 log，不拋出例外 |
| 7 | `[RETURN]` | `LoginAssembler.toLoginRs(authRs)` 組裝 BFF LoginRs（不含 accessToken）回傳 | — |

### Rate Limit 設計

| Redis Key | 操作 | TTL | 說明 |
|-----------|------|-----|------|
| `login:attempt:{account}` | READ / WRITE | 1200s | 滑動視窗失敗計數器 |
| `login:lock:{account}` | READ / WRITE | 1200s | 帳號鎖定旗標，key 自然過期即解鎖 |

**預設閾值：** 20 分鐘內累計失敗 3 次觸發鎖定，鎖定 20 分鐘。  
**可透過設定檔覆寫：** `rate-limit.login.max-attempts`、`rate-limit.login.window-seconds`

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | account 或 passwd 為空 |
| 401 | INVALID_CREDENTIALS | 帳號或密碼錯誤 | auth-service 回傳驗證失敗 |
| 403 | ACCOUNT_LOCKED | 帳號已鎖定 | `login:lock:{account}` key 存在 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |