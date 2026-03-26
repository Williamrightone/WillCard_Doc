# Mock NCCC Pay — Biz Spec

## Overview

**Mock NCCC is a standalone Spring Boot microservice** (Maven module: `mock-nccc`,
deployed only in the development Kubernetes cluster). It simulates NCCC's inbound
authorization call, enabling end-to-end testing without a real NCCC connection.

Receives the App's payment request (via BFF Feign), retrieves the member's encrypted
card data from Card Service, decrypts it locally, re-encrypts it with the mock
combineKey (simulating NCCC's encryption role), and forwards the `encryptedCard` payload
to the Card Service Issuer Authorization API.
Returns `CHALLENGE_REQUIRED` with `challengeRef` and `pointsPreview` on success.

**Deployment notes:**
- Maven module: `mock-nccc` — independent deployable jar, separate from `card-service`
- `server.servlet.context-path: /mock-nccc`
- K8s: single-replica Deployment in `dev` namespace only; no HPA, no production manifest
- mock combineKey injected via K8s Secret: `${MOCK_NCCC_COMBINE_KEY}`

> **Scope:** Development and integration testing only. Must not be deployed to production.

---

## API Definition — mock-nccc

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/mock-nccc` |
| Controller mapping | `/pay` |
| Effective received path | `/mock-nccc/pay` |
| Caller | BFF (FeignClient) |

### Request (MockNcccPayRq — mock-nccc-contract)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Member's virtual card identifier (Snowflake ID as String) |
| amount | BigDecimal | ✅ | @NotNull, @DecimalMin("0.01") | Transaction amount in original currency |
| currency | String | ✅ | @NotBlank, @Size(min=3,max=3) | ISO 4217 currency code, e.g. `TWD`, `USD` |
| merchantId | String | ✅ | @NotBlank | Merchant identifier |

### Response (MockNcccPayRs — mock-nccc-contract)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| status | String | `CHALLENGE_REQUIRED` / `DECLINED` | |
| challengeRef | String | OTP session reference | Present when `CHALLENGE_REQUIRED` |
| pointsPreview.pointsToUse | Integer | Server-computed points offset | Present when `CHALLENGE_REQUIRED` |
| pointsPreview.cardAmount | BigDecimal | Amount charged to card after offset (TWD) | Present when `CHALLENGE_REQUIRED` |
| declineReason | String | Decline reason code | Present when `DECLINED` |

---

## Service UseCase Flow

**UseCase:** `MockNcccPayUseCase.pay(MockNcccPayRq)`

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[FEIGN]` | Call Card Service internal dev endpoint `GET /card/internal/raw-data/{cardId}` to retrieve `{ panEncrypted, expiryDateEncrypted }` — dev-only; not exposed in production → [spec/card-service/card-internal-raw-data.md](../card-service/card-internal-raw-data.md) | — |
| 2 | `[DOMAIN]` | Decrypt `panEncrypted` and `expiryDateEncrypted` using AES-256 (shared Card Service key injected via env var in dev) | `MockCardDecryptionService.decrypt()` |
| 3 | `[DOMAIN]` | Construct mock card payload `{ pan, expiry, cvv="000" }` — mock CVV is a fixed dev value; assemble and encrypt with mock combineKey (AES-256 GCM) → `encryptedCard` | `MockCombineKeyService.encrypt()` |
| 4 | `[DOMAIN]` | Resolve `merchantMCC` from `merchantId` using in-memory mock MCC mapping (e.g. `Map<merchantId, mccCode>` seeded at startup) | `MockMerchantMccResolver.resolve()` |
| 5 | `[FEIGN]` | Call Card Service `POST /card/auth/authorize` with `{ encryptedCard, amount, currency, merchantId, merchantMCC }` → [spec/card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md) | — |
| 6 | `[RETURN]` | Pass through the `CardAuthAuthorizeRs` response as `MockNcccPayRs` | — |

---

## Database

### MySQL

_(None — Mock NCCC is stateless; all state is managed by Card Service)_

### Redis

_(None)_

---

## Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 422 | `CARD_NOT_FOUND` | Card not found | Passed through from Card Service CA00001 |
| 422 | `CARD_NOT_ACTIVE` | Card not active | Passed through from Card Service CA00002 |
| 422 | `CARD_EXPIRED` | Card expired | Passed through from Card Service CA00003 |
| 422 | `CARD_FROZEN` | Card is frozen | Passed through from Card Service CA00004 |
| 422 | `CARD_CANCELLED` | Card cancelled | Passed through from Card Service CA00005 |
| 422 | `SINGLE_TXN_LIMIT_EXCEEDED` | Single transaction limit exceeded | Passed through from Card Service CA00006 |
| 422 | `DAILY_LIMIT_EXCEEDED` | Daily limit exceeded | Passed through from Card Service CA00007 |
| 422 | `INSUFFICIENT_FUNDS` | Wallet balance insufficient | Passed through from Card Service |
| 500 | `INTERNAL_ERROR` | Unexpected system error | Unhandled exception |
