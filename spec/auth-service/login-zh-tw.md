# Login — auth-service Spec

> **所屬服務：** auth-service  
> **Contract Module：** auth-contract  
> **呼叫方：** bff → [bff/login.md](../bff/login.md)  
> **最後更新：** 2026-03

---

## 1. Overview

接收 BFF 透過 Feign 轉發的帳號密碼登入請求。  
依序查詢使用者、驗證密碼、簽發 JWT、建立 Redis Session。  
`accessToken` 包含在 Response 內回傳給 BFF，由 BFF 設入 Response Header，不直接暴露給 Client。  
auth-service 不感知 Rate Limit，所有頻率控制由 BFF 負責。

後續所有需要身份資料的 API，auth-service 透過 `MemberContextFilter`（`common-biz` 自動配置）從 BFF 注入的 `X-Member-Id`、`X-Role`、`X-Name` header 取得，無需自行解析或查詢 Redis。詳見 `spec/common-biz/member-context.md`。

---

## 2. API Definition — auth-service

### Endpoint

| 項目 | 內容 |
|------|------|
| Method | POST |
| context-path | /auth |
| Controller mapping | /login |
| 實際接收路徑 | /auth/login |
| 呼叫方 | BFF（`AuthFeignClient`） |

### Request（`auth.contract.dto.LoginRq`）

| 欄位 | 型別 | 必填 | 驗證規則 | 說明 |
|------|------|------|----------|------|
| account | String | ✅ | @NotBlank | 登入帳號 |
| passwd | String | ✅ | @NotBlank | 登入密碼（明文，傳輸層由 HTTPS 保護） |

### Response（`auth.contract.dto.LoginRs`）

| 欄位 | 型別 | 說明 | 備註 |
|------|------|------|------|
| memberId | String | Snowflake ID（轉為 String） | |
| account | String | 登入帳號 | |
| name | String | 顯示名稱 | |
| accessToken | String | JWT Token | `@JsonIgnore`，僅供 BFF 讀取，不序列化至 Response Body |

### JWT Payload

| 欄位 | 說明 |
|------|------|
| jti | JWT ID，UUID，用於 Redis Session 比對 |
| memberId | Snowflake ID |
| account | 登入帳號 |
| role | MEMBER / ADMIN |
| exp | 過期時間，與 Redis Session TTL 一致（3600s） |

---

## 3. Service UseCase Flow

**UseCase：** `LoginUseCase.login(LoginRq)`  
**UseCase Impl：** `LoginUseCaseImpl`  
**Repository：** `MemberRepository`（interface）→ `MemberRepositoryImpl`（JPA）

| 步驟 | 類型 | 說明 | 負責的 Domain Service |
|------|------|------|-----------------------|
| 1 | `[DB READ]` | 依 `account` 查詢 `card_member` | `MemberQueryService.findByAccount()` |
| 2 | `[DOMAIN]` | 驗證 `passwd` 與 DB 內 Bcrypt 雜湊值 | `AuthService.verifyPassword()` |
| 3 | `[DOMAIN]` | 生成 JWT（payload 含 jti、memberId、account、role、exp） | `TokenService.generate()` |
| 4 | `[REDIS WRITE]` | 寫入 `usession:{memberId}`，TTL 3600s | `SessionService.createSession()` |
| 5 | `[RETURN]` | 回傳 LoginRs（含 accessToken，供 BFF 讀取後設入 header） | — |

### Redis Session 內容

**Key：** `usession:{memberId}`  
**TTL：** 3600s（與 JWT exp 一致）  
**Value：**

| 欄位 | 說明 |
|------|------|
| jti | JWT ID，用於 Token 合法性比對，防止舊 Token 重放 |
| memberId | Snowflake ID |
| account | 登入帳號 |
| role | MEMBER / ADMIN |
| loginAt | Session 建立時間 |

> **單一 Session 策略：** Redis key 以 `memberId` 為單位，新登入會直接覆蓋舊 Session，舊 jti 自動失效。

---

## 4. Database

### MySQL

| Table | 操作 | 說明 |
|-------|------|------|
| `card_member` | READ | 依 `account` 查詢會員資料與 Bcrypt 雜湊值 |

### Redis

| Key Pattern | 操作 | TTL | 說明 |
|-------------|------|-----|------|
| `usession:{memberId}` | WRITE | 3600s | 使用者 Session，新登入覆蓋舊 Session |

### Table Schema

#### card_member

| 欄位 | 型別 | Nullable | 預設值 | 說明 |
|------|------|----------|--------|------|
| member_id | BIGINT | NO | — | PK，Snowflake ID，應用層產生 |
| account | VARCHAR(100) | NO | — | 登入帳號，Unique |
| passwd | VARCHAR(255) | NO | — | Bcrypt 雜湊值 |
| name | VARCHAR(100) | NO | — | 顯示名稱 |
| role | ENUM('MEMBER','ADMIN') | NO | 'MEMBER' | 使用者角色 |
| status | ENUM('ACTIVE','DISABLED') | NO | 'ACTIVE' | 帳號狀態 |
| created_at | DATETIME | NO | — | 建立時間 |
| updated_at | DATETIME | NO | — | 最後更新時間 |

> **注意：** `member_id` 為 Snowflake BIGINT，不使用 auto-increment。  
> 回傳給前端時轉為 `String`，防止 JavaScript 64-bit 整數精度遺失。

---

## 5. Error Codes

| HTTP Status | Error Code | 說明 | 觸發條件 |
|-------------|------------|------|----------|
| 404 | USER_NOT_FOUND | 使用者不存在 | `account` 查無資料 |
| 401 | INVALID_CREDENTIALS | 密碼驗證失敗 | Bcrypt 比對不符 |
| 500 | INTERNAL_ERROR | 系統錯誤 | 非預期例外 |