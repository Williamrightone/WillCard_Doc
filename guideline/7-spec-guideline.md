# Spec Guideline

This document defines the writing conventions and reading guide for all API Spec documents in the WillCard backend.  
All API Spec documents must follow the structure, naming, and layering rules described here.

---

## 1. File Location & Naming

### 1.1 Directory Structure

```cmd
guideline/
├── 7-spec-guideline.md       ← This document (writing conventions)
└── spec/
    ├── bff/
    │     └── login.md        ← BFF spec (external API entry point, links to biz specs)
    ├── auth-service/
    │     └── login.md        ← biz spec (contract definition, UseCase, DB)
    ├── card-service/
    │     └── ...
    └── wallet-service/
          └── ...
```

### 1.2 Naming Rules

- File names use lowercase kebab-case, e.g. `login.md`, `top-up.md`
- Each service has its own folder; folder name matches the Maven module name
- One Use Case maps to one BFF spec; the BFF spec links to one or more biz specs it calls
- The same biz spec may be linked from multiple BFF specs (contract reuse)

### 1.3 Document Responsibility Boundary

| Document Type | Responsibility |
|---------------|----------------|
| **BFF spec** | External API definition, BFF UseCase Flow, Operation Log Level, BFF-layer Error Codes |
| **biz spec** | Internal Service API definition (incl. contract Rq/Rs), Service UseCase Flow, Database, Service-layer Error Codes |

---

## 2. BFF Spec Document Structure

```cmd
1. Overview
2. Operation Log Level
3. API Definition — BFF
4. BFF UseCase Flow
5. Error Codes
```

---

## 3. Biz Spec Document Structure

```cmd
1. Overview
2. API Definition — {service-name}
3. Service UseCase Flow
4. Database
5. Error Codes
```

---

## 4. Section Writing Rules

### 4.1 Overview

Briefly describe the business purpose of this API, who calls it, and what its core behavior is.  
No more than 5 lines. Do not include implementation details.  
BFF spec and biz spec Overviews each describe the responsibility of their own layer.

```markdown
## Overview

(BFF spec example)
Receives the client's username/password login request, performs rate limit checks,
then forwards to auth-service for authentication. On success, sets the JWT in the
Response Header and publishes an Operation Log event.

(biz spec example)
Validates the user's credentials, issues a JWT, and creates a Redis Session.
The token is returned to the BFF only and is never directly exposed to the client.
```

---

### 4.2 Operation Log Level (BFF spec only)

Declare the Operation Log level for this API, along with its trigger condition and Kafka Topic.

| Level | Definition | Log Behavior |
|-------|------------|--------------|
| L1 | Financial, authentication, or security operations | BFF publishes Kafka event → Audit Consumer writes to DB |
| L2 | Non-financial data mutations | BFF publishes Kafka event → Audit Consumer writes to DB |
| L3 | General read queries | EFK only — no Kafka, no DB write |

```markdown
## Operation Log Level

**Level: L1**
Trigger: Every call to the login API must be recorded, regardless of success or failure.
Kafka Topic: `operation-log.auth.login`
Stored Fields: userId (null on failure), userAccount, result (SUCCESS / FAIL), failReason, ip, timestamp
```

---

### 4.3 API Definition — BFF

Describes the API exposed externally by the BFF.

**Path Rules:**

- Client sends: `/api/v{n}/{resource}/{action}`
- Nginx strips `/api/v{n}`; BFF receives: `/{resource}/{action}`
- BFF has no context-path; version management is handled entirely by Nginx
- **Controllers always use version-free paths** — the version is determined by Nginx routing

**Version Upgrade Flow:**

Under normal conditions, a Controller has a single path:

```cmd
Controller: /auth/login        ← Nginx v1 routes here
```

When a breaking change is needed, two paths temporarily coexist:

```cmd
Controller: /auth/login        ← Continues serving old App users (Nginx v1)
Controller: /auth/login-v2     ← Temporary name for the new version (Nginx v2)
```

After v2 testing is complete and the old App is retired:

```cmd
Controller: /auth/login        ← Renamed back to standard (Nginx v1 / v2 both route here)
Old /auth/login-v2             ← Deleted
```

> Version upgrades are transitional. The `-v2` suffix is a temporary naming convention and must not persist long-term in the codebase.

```markdown
## API Definition — BFF

### Endpoint
| Field | Value |
|-------|-------|
| Method | POST |
| Client URL | /api/v1/auth/login |
| BFF Received Path | /auth/login |
| Auth Required | No (Public endpoint) |

### Request Body
| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| userAccount | String | ✅ | @NotBlank | User account |
| passwd | String | ✅ | @NotBlank | User password |

### Response Body
| Field | Type | Description |
|-------|------|-------------|
| userId | String | User ID |
| userAccount | String | User account |
| name | String | User display name |

### Response Header
| Header | Description |
|--------|-------------|
| Authorization | Bearer {accessToken} |
```

---

### 4.4 API Definition — {service-name} (biz spec)

Describes the internal API called by the BFF via Feign, and defines the contract module's Rq/Rs.

**Path Rules:**

- Each service sets `server.servlet.context-path: /{service-name}`
- Controller mapping contains only the action verb — no service name prefix
- Effective received path = `/{service-name}/{action}`
- Version management is handled by Nginx; the service layer carries no version number

```markdown
## API Definition — auth-service

### Endpoint
| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /auth |
| Controller mapping | /login |
| Effective received path | /auth/login |
| Caller | BFF (FeignClient) |

### Request (LoginRq — auth-contract)
| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| userAccount | String | ✅ | @NotBlank | User account |
| passwd | String | ✅ | @NotBlank | User password |

### Response (LoginRs — auth-contract)
| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| userId | String | User ID | |
| userAccount | String | User account | |
| name | String | User display name | |
| accessToken | String | JWT Token | @JsonIgnore — read by BFF only, not serialized into Response Body |
```

---

### 4.5 BFF UseCase Flow

Describes the execution steps of the BFF UseCase.  
Every `[FEIGN]` step must include a link to the corresponding biz spec.

**Action Type Tags:**

| Tag | Description |
|-----|-------------|
| `[VALIDATE]` | Input validation |
| `[REDIS READ]` | Read from Redis |
| `[REDIS WRITE]` | Write to Redis |
| `[FEIGN]` | Call a downstream service (link required) |
| `[HEADER]` | Set Response Header |
| `[KAFKA]` | Publish a Kafka event |
| `[RETURN]` | Return the Response |

```markdown
## BFF UseCase Flow

**UseCase:** `LoginUseCase.login(LoginRq)`

| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | `[REDIS READ]` | Check if `login:lock:{userAccount}` exists | Key exists → throw ACCOUNT_LOCKED |
| 2 | `[FEIGN]` | Call auth-service → [auth-service/login.md](../auth-service/login.md) | Error returned → proceed to step 3 |
| 3 | `[REDIS WRITE]` | On login failure: increment `login:attempt:{userAccount}`, TTL 1200s; write `login:lock:{userAccount}` if count reaches 3, TTL 1200s | — |
| 4 | `[HEADER]` | Extract accessToken from LoginRs, set `Authorization: Bearer {token}` | — |
| 5 | `[KAFKA]` | Publish `operation-log.auth.login` event (async — failure does not affect main flow) | — |
| 6 | `[RETURN]` | Return LoginRs (without accessToken) | — |
```

---

### 4.6 Service UseCase Flow (biz spec)

Describes the execution steps of the Service UseCase, defined down to the Domain Service interface level.  
Domain Service implementation details are out of scope — left to the developer's discretion.

**Action Type Tags:**

| Tag | Description |
|-----|-------------|
| `[DB READ]` | Read from MySQL |
| `[DB WRITE]` | Write to MySQL |
| `[REDIS WRITE]` | Write to Redis |
| `[DOMAIN]` | Call a Domain Service method |
| `[RETURN]` | Return the Response |

```markdown
## Service UseCase Flow

**UseCase:** `LoginUseCase.login(LoginRq)` (auth-service)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query user by userAccount | `UserQueryService.findByAccount()` |
| 2 | `[DOMAIN]` | Bcrypt-verify passwd against stored hash | `AuthService.verifyPassword()` |
| 3 | `[DOMAIN]` | Generate JWT (payload: jti, userId, userAccount, role, exp) | `TokenService.generate()` |
| 4 | `[REDIS WRITE]` | Write `usession:{userId}` (jti, userId, userAccount, role, loginAt), TTL 3600s | `SessionService.createSession()` |
| 5 | `[RETURN]` | Return LoginRs (including accessToken for BFF to read) | — |
```

---

### 4.7 Database (biz spec)

List all Tables and Redis keys this API reads from or writes to, along with the Table Schema.  
Full DDL is maintained in `4-data-mgmt.md`.

```markdown
## Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `user` | READ | Query user credentials and profile by userAccount |
| `operation_log` | WRITE | Written asynchronously by Audit Consumer after Kafka event |

### Redis

| Key Pattern | Operation | TTL | Description |
|-------------|-----------|-----|-------------|
| `usession:{userId}` | WRITE | 3600s | User Session |

### Table Schema

#### user
| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT | NO | PK, auto-increment |
| user_id | VARCHAR(36) | NO | UUID, business-facing ID, Unique |
| user_account | VARCHAR(100) | NO | Login account, Unique |
| passwd | VARCHAR(255) | NO | Bcrypt hash |
| name | VARCHAR(100) | NO | Display name |
| role | VARCHAR(20) | NO | User role (USER / ADMIN) |
| status | VARCHAR(20) | NO | Account status (ACTIVE / DISABLED) |
| created_at | DATETIME | NO | Record creation timestamp |
| updated_at | DATETIME | NO | Last update timestamp |
```

---

### 4.8 Error Codes

**BFF spec** lists error codes returned to the client (externally visible).  
**biz spec** lists error codes thrown internally by the service (BFF may remap or pass through).  
Format must conform to the error code conventions in `5-exception.md`.

```markdown
## Error Codes

(BFF spec example)
| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | INVALID_REQUEST | Malformed request | userAccount or passwd is blank |
| 401 | INVALID_CREDENTIALS | Invalid credentials | auth-service returned an auth failure |
| 403 | ACCOUNT_LOCKED | Account is locked | login:lock key exists |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

(biz spec example)
| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | USER_NOT_FOUND | User does not exist | No record found for userAccount |
| 401 | INVALID_CREDENTIALS | Password verification failed | Bcrypt mismatch |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
```

---

## 5. Idempotency

### 5.1 Scope

All POST requests that produce side effects on financial operations (transactions, top-ups, redemptions, etc.) **must** carry an `Idempotency-Key` header.
Non-financial or read-only POST endpoints (e.g. login) are excluded.

### 5.2 Mechanism

The client generates a UUID v4 and sends it as the `Idempotency-Key` request header.
The BFF checks Redis before executing business logic:

```
Client generates UUID v4 → Idempotency-Key header

BFF receives request:
  1. SETNX  idempotency:{key}  →  { status: PROCESSING, createdAt }   (short TTL, e.g. 30–60 s)
  2. If SETNX fails and status = COMPLETED  → return cached response (no re-execution)
  3. If SETNX fails and status = PROCESSING  → throw 409 Conflict
  4. If SETNX succeeds                       → execute business logic
  5. On success → update to COMPLETED, cache response (longer TTL, e.g. 24 h)
  6. On failure → delete key (allow client to retry with the same Idempotency-Key)
```

> **SETNX (SET NX)** ensures atomicity — only one request can claim a given key, preventing race conditions when duplicate requests arrive simultaneously.

### 5.3 Redis Key & Value

| Item | Value |
|------|-------|
| Key pattern | `idempotency:{Idempotency-Key}` |
| PROCESSING TTL | Short (e.g. 30–60 s) — prevents permanent lock on server crash |
| COMPLETED TTL | Longer (e.g. 24 h) — defines the retry window for cached responses |

**Value structure:**

| Field | Type | Description |
|-------|------|-------------|
| status | String | `PROCESSING` / `COMPLETED` |
| httpStatus | int | Original HTTP status code of the cached response |
| responseBody | String | Serialized JSON of the response |
| createdAt | long | Epoch millis when the key was created |

### 5.4 Failure Handling

| Scenario | Behavior |
|----------|----------|
| Business logic succeeds | Update status to `COMPLETED`, cache response |
| Business logic fails | **Delete the key** — client may retry with the same `Idempotency-Key` |
| PROCESSING key hit by another request | Return **409 Conflict** |
| PROCESSING key expires (TTL) | Key is auto-released — next request starts fresh |

### 5.5 Idempotency-Key vs txnRef

| | `Idempotency-Key` | `txnRef` |
|---|---|---|
| Purpose | Prevent duplicate HTTP requests at the BFF layer | Saga session identifier for multi-step OTP flows |
| Generated by | Client (UUID v4) | Server (first step of `card-pay`) |
| Scope | Single HTTP request | Entire Saga lifecycle (e.g. `card-pay` → `card-pay/confirm`) |
| Storage | Redis (`idempotency:{key}`) | Saga orchestrator state |

### 5.6 BFF Spec Writing Guide

When a BFF spec requires idempotency:

1. **API Definition** — add `Idempotency-Key` to the Request Header table:

```markdown
### Request Header
| Header | Required | Description |
|--------|----------|-------------|
| Idempotency-Key | ✅ | UUID v4, generated by client |
```

2. **BFF UseCase Flow** — insert the idempotency check as the first step(s):

```markdown
| Step | Type | Description | Failure Handling |
|------|------|-------------|------------------|
| 1 | [REDIS WRITE] | SETNX idempotency:{Idempotency-Key} with PROCESSING status | Key exists & COMPLETED → return cached response; Key exists & PROCESSING → throw 409 |
| ... | ... | (remaining business logic steps) | ... |
| N | [REDIS WRITE] | Update idempotency:{Idempotency-Key} to COMPLETED, cache response | On biz failure → delete key |
```

---

## 6. Pre-Publish Checklist

### BFF Spec

- [ ] Overview describes the BFF-layer business purpose
- [ ] Operation Log Level declares the level, Kafka Topic, and stored fields
- [ ] API Definition path follows the Nginx strip rule (no version number)
- [ ] Every BFF UseCase Flow step has a type tag and failure handling
- [ ] Every `[FEIGN]` step links to the corresponding biz spec
- [ ] Error Codes lists externally visible error codes
- [ ] APIs requiring idempotency declare `Idempotency-Key` in Request Header and include idempotency check steps in UseCase Flow

### Biz Spec

- [ ] Overview describes the Service-layer business responsibility
- [ ] API Definition path follows the context-path rule (no version number)
- [ ] Contract Rq/Rs fields are fully defined; `@JsonIgnore` fields are annotated
- [ ] Service UseCase Flow is defined down to the Domain Service interface — no implementation details
- [ ] Database section includes Table list, Redis key list, and Table Schema
- [ ] Error Codes lists internal Service error codes

---

## 7. Reading Guide

**Developer (BFF) reading order:**

1. `bff/{feature}.md` → Overview, API Definition, BFF UseCase Flow
2. Follow `[FEIGN]` links → Read the corresponding biz spec for contract details

**Developer (biz service) reading order:**

1. `{service}/{feature}.md` → Overview, API Definition (contract Rq/Rs)
2. Service UseCase Flow → Database

**Architect / Reviewer reading order:**

1. `bff/{feature}.md` → Full flow overview and responsibility boundaries
2. Each biz spec → Verify contract definitions and DB design

---
