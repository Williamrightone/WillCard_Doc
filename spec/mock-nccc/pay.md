# Pay — Mock NCCC Spec

> **Service:** mock-nccc
> **Environment:** Development only
> **Contract Module:** mock-nccc-contract
> **Caller:** [txn-orch/card-pay.md](../txn-orch/card-pay.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Simulates NCCC (National Credit Card Center) card network behavior for the payment initiation leg.

Receives the payment request from ORCH, fetches plaintext card data from card-service's internal plain endpoint, assembles `encryptedCard` using `MOCK_COMBINE_KEY` (AES-256-GCM), then forwards to card-service auth/authorize for card validation and OTP generation.

Returns `challengeRef` to ORCH.

> **Dev-only:** `MOCK_COMBINE_KEY` must match card-service's `combineKey` (the XOR result of the three source key parts). Key is loaded from `application-dev.yml` at startup into `MockCombineKeyHolder` — no database involved.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (ORCH) | [txn-orch/card-pay.md](../txn-orch/card-pay.md) |
| Mock NCCC → card-service (plain card data) | [card-service/card-plain.md](../card-service/card-plain.md) |
| Mock NCCC → card-service (auth/authorize) | [card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md) |
| Leg 2 (OTP confirmation) | [mock-nccc/pay-confirm.md](pay-confirm.md) |

---

## 3. API Definition — mock-nccc

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /mock-nccc |
| Controller mapping | /pay |
| Effective received path | /mock-nccc/pay |
| Caller | ORCH (MockNcccFeignClient) |

### Request (`mock-nccc.contract.dto.MockNcccPayRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Card Snowflake ID as String |
| amount | Long | ✅ | @NotNull; @Min(1) | Original transaction amount in cents |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 currency code |
| txnTwdBaseAmount | Long | ✅ | @NotNull; @Min(1) | FX-converted TWD base amount; computed by ORCH |
| merchantId | String | ✅ | @NotBlank | Merchant identifier |

### Response (`mock-nccc.contract.dto.MockNcccPayRs`)

| Field | Type | Description |
|-------|------|-------------|
| challengeRef | String | UUID; OTP session binding; returned from card-service auth/authorize |

---

## 4. Service UseCase Flow

**UseCase:** `MockNcccPayUseCase.pay(MockNcccPayRq)` (mock-nccc)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DOMAIN]` | Retrieve `MOCK_COMBINE_KEY` from `MockCombineKeyHolder`; if null → throw 503 MOCK_KEY_UNAVAILABLE |
| 2 | `[FEIGN]` | `CardServiceFeignClient.getPlainCard(cardId)` → `GET /card-service/internal/cards/{cardId}/plain`; response: `{ pan, expiryMMYY, cvvItoken }`; see [card-service/card-plain.md](../card-service/card-plain.md) |
| 3 | `[DOMAIN]` | AES-256-GCM encrypt `{ pan, expiryMMYY, cvvItoken }` with `MOCK_COMBINE_KEY` → `encryptedCard` (try-finally: zero out `pan` byte array after encryption) |
| 4 | `[FEIGN]` | `CardServiceFeignClient.authorize(CardAuthAuthorizeRq)` → `POST /card-service/auth/authorize`; request: `{ encryptedCard, amount, currency, txnTwdBaseAmount, merchantId }`; response: `{ challengeRef }`; see [card-service/card-auth-authorize.md](../card-service/card-auth-authorize.md) |
| 5 | `[RETURN]` | Return `MockNcccPayRs { challengeRef }` |

> Step 4 may throw CA000xx errors (card validation or limit failures) which are propagated to ORCH.

---

## 5. Startup Configuration — MockCombineKeyHolder

**Trigger:** `ApplicationReadyEvent`

| Step | Description |
|------|-------------|
| 1 | Read `willcard.security.mock-combine-key` from `application-dev.yml` (64 hex chars = 256-bit AES key) |
| 2 | Validate: must be exactly 64 valid hexadecimal characters |
| 3 | Decode hex string → 32-byte array → store in `MockCombineKeyHolder` |
| 4 | If blank or invalid format → `MockCombineKeyHolder.key = null`; all UseCase calls will return 503 MOCK_KEY_UNAVAILABLE |

**`application-dev.yml` configuration:**

```yaml
willcard:
  security:
    mock-combine-key: "0000000000000000000000000000000000000000000000000000000000000000"
    # 64 hex chars (256-bit); must equal card-service combineKey (XOR of 3 source key parts)
```

> `MOCK_COMBINE_KEY` is a dev-environment secret. It must never appear in production configuration or be committed to source control without environment-specific override.

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 503 | MOCK_KEY_UNAVAILABLE | Mock combine key not loaded | `MockCombineKeyHolder.key = null` at startup |
| 409 | CA00002 | CARD_NOT_ACTIVE | Propagated from card-service auth/authorize |
| 409 | CA00003 | CARD_EXPIRED | Propagated from card-service auth/authorize |
| 409 | CA00004 | CARD_FROZEN | Propagated from card-service auth/authorize |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | Propagated from card-service auth/authorize |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | Propagated from card-service auth/authorize |
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | Propagated from card-service (card-service combineKey not loaded) |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 7. Changelog

### v1.0 — 2026-03 — Initial spec
