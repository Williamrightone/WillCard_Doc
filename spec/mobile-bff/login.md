# Login — BFF Spec

> **Service:** bff  
> **Related biz spec:** [auth-service/login.md](../auth-service/login.en.md)  
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the client's username/password login request.  
Performs account lock checking, then forwards the request to auth-service for authentication.  
On success, sets the JWT in the Response Header and asynchronously publishes an Operation Log event.  
The BFF does not perform password verification or token issuance — it is responsible only for flow coordination and header assembly.

---

## 2. Operation Log Level

**Level: L1**  
**Trigger:** Every call to the login API must be recorded, regardless of success or failure.  
**Kafka Topic:** `operation-log.auth.login`

| Field | Type | Description |
|-------|------|-------------|
| memberId | BIGINT | Snowflake ID on success; null on failure |
| account | VARCHAR | Login account |
| result | ENUM | SUCCESS / FAIL |
| failReason | VARCHAR | Failure reason; null on success |
| ip | VARCHAR | Client IP address |
| timestamp | DATETIME | Event occurrence time |

---

## 3. API Definition — BFF

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | /api/v1/auth/login |
| BFF Received Path | /auth/login |
| Auth Required | No (Public endpoint — Spring Security permitAll) |

### Request Body (`application.api.dto.LoginRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| account | String | ✅ | @NotBlank | Login account |
| passwd | String | ✅ | @NotBlank | Login password |

### Response Body (`application.api.dto.LoginRs`)

| Field | Type | Description |
|-------|------|-------------|
| memberId | String | Snowflake ID (serialized as String to prevent JavaScript precision loss) |
| account | String | Login account |
| name | String | Display name |

### Response Header

| Header | Description |
|--------|-------------|
| Authorization | Bearer {accessToken} |

---

## 4. BFF UseCase Flow

**UseCase:** `LoginUseCase.login(LoginRq)`  
**UseCase Impl:** `LoginUseCaseImpl`  
**Adapter:** `AuthClientAdapter` implements `AuthClientPort`  
**Feign:** `AuthFeignClient` (uses `LoginRq` / `LoginRs` from `auth-contract`)

### Conversion Note

The BFF `LoginRq` (`application.api.dto`) and the auth-contract `LoginRq` (`auth.contract.dto`) share the same class name but belong to different packages.  
`LoginAssembler.toAuthRq(LoginRq)` handles the conversion inside `LoginUseCaseImpl`, preventing both same-named classes from appearing directly in the same method scope.

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[REDIS READ]` | Check if `login:lock:{account}` exists | Key exists → throw `ACCOUNT_LOCKED`, abort flow |
| 2 | `[VALIDATE]` | `LoginAssembler.toAuthRq(rq)` converts BFF LoginRq to auth-contract LoginRq | — |
| 3 | `[FEIGN]` | `AuthClientAdapter` calls auth-service → [auth-service/login.en.md](../auth-service/login.en.md) | Business error → proceed to step 4; system error → throw `INTERNAL_ERROR` |
| 4 | `[REDIS WRITE]` | On login failure: increment `login:attempt:{account}`, TTL 1200s; write `login:lock:{account}` when count reaches 3, TTL 1200s | — |
| 5 | `[HEADER]` | On login success: extract `accessToken` from auth-contract LoginRs, set `Authorization: Bearer {token}` | — |
| 6 | `[KAFKA]` | Publish `operation-log.auth.login` event (async — failure does not affect main flow) | Log the error, do not throw exception |
| 7 | `[RETURN]` | `LoginAssembler.toLoginRs(authRs)` assembles BFF LoginRs (without accessToken) and returns | — |

### Rate Limit Design

| Redis Key | Operation | TTL | Description |
|-----------|-----------|-----|-------------|
| `login:attempt:{account}` | READ / WRITE | 1200s | Sliding window failure counter |
| `login:lock:{account}` | READ / WRITE | 1200s | Account lock flag; key expiry = automatic unlock |

**Default Threshold:** 3 failures within 20 minutes triggers a 20-minute lock.  
**Configurable via:** `rate-limit.login.max-attempts`, `rate-limit.login.window-seconds`

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | account or passwd is blank |
| 401 | INVALID_CREDENTIALS | Invalid credentials | auth-service returned an authentication failure |
| 403 | ACCOUNT_LOCKED | Account is locked | `login:lock:{account}` key exists |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |