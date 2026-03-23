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
**觸發時機：** 每次呼叫，無論成功或失敗皆須記錄。
**Kafka Topic：** `operation-log.auth.login`

| OperationLogEvent 欄位 | 值來源 |
|------------------------|--------|
| service | `bff` |
| action | `auth.login` |
| userId | `LoginRs.memberId`（失敗時為 null） |
| userAccount | `Request.account` |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | 失敗描述；成功時為 null |
| errorCode | 失敗錯誤碼；成功時為 null |
| beforeSnapshot | — |
| afterSnapshot | — |
| requestId | HTTP correlation ID |
| txnId | — |

**Publish 觸發點：**

| 情境 | result | failReason / errorCode |
|------|--------|------------------------|
| 帳號鎖定（Step 1） | FAIL | `ACCOUNT_LOCKED` / — |
| 密碼驗證失敗（Step 5） | FAIL | 取自 auth-service errorCode |
| 登入成功（Step 7） | SUCCESS | — |

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
**Adapter：** `AuthFeignAdapter` implements `AuthClientPort`
**Feign：** `AuthFeignClient`（使用 `auth-contract` 的 `LoginRq` / `LoginRs`）

### 執行步驟

| 步驟 | 類型 | 說明 | 失敗處理 |
|------|------|------|----------|
| 1 | `[REDIS READ]` | 檢查 `login:lock:{account}` 是否存在 | key 存在 → publish FAIL event（failReason=`ACCOUNT_LOCKED`，非同步）→ 拋出 `ACCOUNT_LOCKED`，中止流程 |
| 2 | `[FEIGN]` | `AuthFeignAdapter` 呼叫 auth-service → [auth-service/login.md](../auth-service/login.md) | auth-service 回傳業務錯誤 → 執行步驟 3；系統錯誤 → 拋出 `INTERNAL_ERROR` |
| 3 | `[REDIS WRITE]` | **【失敗路徑】** `login:attempt:{account}` 計數 +1，TTL 1200s；計數達 3 時寫入 `login:lock:{account}`，TTL 1200s | — |
| 4 | `[KAFKA]` | **【失敗路徑】** Publish FAIL event（errorCode 取自 auth-service，非同步）→ 拋出 `INVALID_CREDENTIALS` | 記錄錯誤 log，不拋出例外 |
| 5 | `[REDIS WRITE]` | **【成功路徑】** 寫入 `session:{memberId}` = accessToken，TTL 與 JWT exp 一致（3600s）；覆蓋既有 token，實現後踢前（同一帳號同一時間只有一個有效 Session） | — |
| 6 | `[HEADER]` | **【成功路徑】** 從 auth-contract LoginRs 取出 `accessToken`，設入 `Authorization: Bearer {token}`；組裝身份 header 注入下游（見 Header Forwarding） | — |
| 7 | `[KAFKA]` | **【成功路徑】** Publish SUCCESS event（非同步） | 記錄錯誤 log，不拋出例外 |
| 8 | `[RETURN]` | 從 auth-contract `LoginRs` 組裝 BFF `LoginRs`（不含 `accessToken`）回傳 | — |

### Header Forwarding

登入成功後，BFF 從 auth-contract LoginRs 取出身份資料，注入為 Response Header 供後續請求使用。  
下游 service 透過 `MemberContextFilter`（`common-biz` 自動配置）從 header 取得身份資料，無需自行解析。

| Header | 來源欄位 | 說明 |
|--------|----------|------|
| `X-Member-Id` | `LoginRs.memberId` | Snowflake ID（String） |
| `X-Role` | `LoginRs.role` | MEMBER / ADMIN |
| `X-Name` | `LoginRs.name` | 顯示名稱 |

> **注意：** 這三個 header 是 BFF 在每次通過驗證的請求中注入的，不僅限於 login。login 是建立 Session 的時機，header forwarding 是所有後續 API 的共通機制，詳見 `spec/common-biz/member-context.md`。

### Rate Limit 設計

| Redis Key | 操作 | TTL | 說明 |
|-----------|------|-----|------|
| `login:attempt:{account}` | READ / WRITE | 1200s | 滑動視窗失敗計數器 |
| `login:lock:{account}` | READ / WRITE | 1200s | 帳號鎖定旗標，key 自然過期即解鎖 |
| `session:{memberId}` | WRITE | JWT exp（3600s） | 有效 Token 存儲，新登入覆蓋舊值，實現後踢前 |

**預設閾值：** 20 分鐘內累計失敗 3 次觸發鎖定，鎖定 20 分鐘。
**可透過設定檔覆寫：** `rate-limit.login.max-attempts`、`rate-limit.login.window-seconds`

> **Session 歸屬說明：** 使用者 Session（「這個 member 目前哪個 token 有效」）是通道層（channel layer）的狀態，屬於接入層 BFF 的關注點。auth-service 維持無狀態，僅負責驗證帳密。

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 400 | INVALID_REQUEST | 請求格式錯誤 | account 或 passwd 為空 |
| 401 | INVALID_CREDENTIALS | 帳號或密碼錯誤 | auth-service 回傳驗證失敗 |
| 403 | ACCOUNT_LOCKED | 帳號已鎖定 | `login:lock:{account}` key 存在 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |