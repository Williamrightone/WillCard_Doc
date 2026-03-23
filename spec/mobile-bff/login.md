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
**Trigger:** Every call, regardless of success or failure.
**Kafka Topic:** `operation-log.auth.login`

| OperationLogEvent Field | Value Source |
|------------------------|--------------|
| service | `bff` |
| action | `auth.login` |
| userId | `LoginRs.memberId` (null on failure) |
| userAccount | `Request.account` |
| ip | Client IP |
| result | `SUCCESS` / `FAIL` |
| failReason | Error description on failure; null on success |
| errorCode | Error code on failure; null on success |
| beforeSnapshot | — |
| afterSnapshot | — |
| requestId | HTTP correlation ID |
| txnId | — |

**Publish Triggers:**

| Scenario | result | failReason / errorCode |
|----------|--------|------------------------|
| Account locked (Step 1) | FAIL | `ACCOUNT_LOCKED` / — |
| Credential failure (Step 5) | FAIL | From auth-service errorCode |
| Login success (Step 7) | SUCCESS | — |

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
**Adapter:** `AuthFeignAdapter` implements `AuthClientPort`
**Feign:** `AuthFeignClient` (uses `LoginRq` / `LoginRs` from `auth-contract`)

### Execution Steps

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[REDIS READ]` | Check if `login:lock:{account}` exists | Key exists → publish FAIL event (failReason=`ACCOUNT_LOCKED`, async) → throw `ACCOUNT_LOCKED`, abort flow |
| 2 | `[FEIGN]` | `AuthFeignAdapter` calls auth-service → [auth-service/login.md](../auth-service/login.md) | Business error → proceed to step 3; system error → throw `INTERNAL_ERROR` |
| 3 | `[REDIS WRITE]` | **[On failure]** Increment `login:attempt:{account}`, TTL 1200s; write `login:lock:{account}` when count reaches 3, TTL 1200s | — |
| 4 | `[KAFKA]` | **[On failure]** Publish FAIL event (errorCode from auth-service, async) → throw `INVALID_CREDENTIALS` | Log the error, do not throw exception |
| 5 | `[REDIS WRITE]` | **[On success]** Write `session:{memberId}` = accessToken, TTL aligned with JWT exp (3600s); overwrites any existing token — enforces single active session per member | — |
| 6 | `[HEADER]` | **[On success]** Extract `accessToken` from auth-contract LoginRs, set `Authorization: Bearer {token}`; assemble identity headers (see Header Forwarding) | — |
| 7 | `[KAFKA]` | **[On success]** Publish SUCCESS event (async) | Log the error, do not throw exception |
| 8 | `[RETURN]` | Assemble BFF `LoginRs` from auth-contract `LoginRs` (exclude `accessToken`) and return | — |

### Header Forwarding

On successful login, the BFF extracts identity data from the auth-contract LoginRs and injects it as request headers for use in subsequent requests.  
Downstream services read these headers via `MemberContextFilter` (auto-configured by `common-biz`) — no manual parsing required.

| Header | Source Field | Description |
|--------|--------------|-------------|
| `X-Member-Id` | `LoginRs.memberId` | Snowflake ID (String) |
| `X-Member-Account` | `LoginRs.account` | Login account |
| `X-Member-Role` | `LoginRs.role` | MEMBER / ADMIN |

> **Note:** These headers are injected by the BFF on every authenticated request, not only during login. Login establishes the Session; header forwarding is the shared mechanism for all subsequent APIs. See `spec/common-biz/member-context.md`.

### Rate Limit Design

| Redis Key | Operation | TTL | Description |
|-----------|-----------|-----|-------------|
| `login:attempt:{account}` | READ / WRITE | 1200s | Sliding window failure counter |
| `login:lock:{account}` | READ / WRITE | 1200s | Account lock flag; key expiry = automatic unlock |
| `session:{memberId}` | WRITE | JWT exp (3600s) | Active token store — new login overwrites previous token (single active session per member) |

**Default Threshold:** 3 failures within 20 minutes triggers a 20-minute lock.
**Configurable via:** `rate-limit.login.max-attempts`, `rate-limit.login.window-seconds`

> **Session ownership rationale:** User session state ("which token is currently valid for this member") is a channel-layer concern managed by the BFF. auth-service is stateless and only validates credentials.

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | account or passwd is blank |
| 400 | ACCOUNT_LOCKED | Account is locked | `login:lock:{account}` key exists |
| 400 | INVALID_CREDENTIALS | Invalid credentials | auth-service returned an authentication failure |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |