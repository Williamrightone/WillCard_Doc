# Login — auth-service Spec

> **所屬服務：** auth-service  
> **Contract Module：** auth-contract  
> **呼叫方：** bff → [bff/login.md](../bff/login.md)  
> **最後更新：** 2026-03

---

## 1. Overview

接收 BFF 透過 Feign 轉發的帳號密碼登入請求。
依序查詢使用者、驗證密碼、簽發 JWT。
`accessToken` 包含在 Response 內回傳給 BFF，由 BFF 設入 Response Header，不直接暴露給 Client。
auth-service 為**無狀態服務**：職責僅限於驗證帳密與簽發 JWT，不感知 Rate Limit，也不管理 Session。所有頻率控制與使用者 Session 管理由 BFF 負責。

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
| jti | JWT ID，UUID，本次 Token 簽發的唯一識別碼 |
| memberId | Snowflake ID |
| account | 登入帳號 |
| role | MEMBER / ADMIN |
| exp | 過期時間（簽發後 3600s） |

---

## 3. Service UseCase Flow

**UseCase：** `LoginUseCase.login(LoginRq)`
**UseCase Impl：** `LoginUseCaseImpl`
**Repository：** `MemberJpaRepository`（Spring Data JPA）

> **DataHelper 慣例：** 複雜 query 或跨多 table 的操作，應抽取獨立的 `*DataHelper` class，而非直接擴充 service。單一 table 的簡單查詢（如本例）可從 service 直接呼叫 `MemberJpaRepository`。

| 步驟 | 類型 | 說明 |
|------|------|------|
| 1 | `[DB READ]` | 透過 `MemberJpaRepository` 依 `account` 查詢 `member` |
| 2 | `[DOMAIN]` | 驗證 `passwd` 與 DB 內 Bcrypt 雜湊值 |
| 3 | `[DOMAIN]` | 生成 JWT（payload 含 jti、memberId、account、role、exp） |
| 4 | `[RETURN]` | 回傳 LoginRs（含 accessToken，供 BFF 讀取後設入 header） |

---

## 4. Database

### MySQL

| Table | 操作 | 說明 |
|-------|------|------|
| `member` | READ | 依 `account` 查詢會員資料與 Bcrypt 雜湊值 |

### Table Schema

#### member

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