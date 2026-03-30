# Card Plain Data — card-service Spec (Internal / Dev Only)

> **Service:** card-service
> **Environment:** Development only
> **Contract Module:** card-service-contract (internal)
> **Caller:** [mock-nccc/pay.md](../mock-nccc/pay.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Internal-only endpoint that returns decrypted card data for Mock NCCC to assemble `encryptedCard` during dev-environment transaction simulation.

Decrypts `pan_encrypted` and `expiry_date` from the `card` table and returns `cvv_itoken` (the HMAC-SHA256 token — raw CVV is never stored per PCI DSS) for card-service `auth/authorize` to verify.

> **Dev profile only.** This endpoint must never be exposed in production. It must be guarded by a Spring profile condition (`@Profile("dev")` or equivalent) so that the route is unavailable in non-dev deployments.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (Mock NCCC) | [mock-nccc/pay.md](../mock-nccc/pay.md) |
| How returned data is used | [card-service/card-auth-authorize.md](card-auth-authorize.md) |

---

## 3. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| context-path | /card-service |
| Controller mapping | /internal/cards/{cardId}/plain |
| Effective received path | /card-service/internal/cards/{cardId}/plain |
| Caller | Mock NCCC (`CardServiceFeignClient`) |
| Profile restriction | `dev` only — route inactive in other profiles |

### Path Variable

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| cardId | String | ✅ | Card Snowflake ID (String format) |

### Response (`card.service.internal.dto.CardPlainRs`)

| Field | Type | Description |
|-------|------|-------------|
| pan | String | Full plaintext PAN (16 digits); **must not be logged** |
| expiryMMYY | String | Expiry in MMYY format, e.g. `0329`; **must not be logged** |
| cvvItoken | String | HMAC-SHA256 of CVV (stored token); raw CVV is never returned |

---

## 4. Service UseCase Flow

**UseCase:** `CardPlainInternalUseCase.getPlainCard(String cardId)`

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DB READ]` | SELECT `card` WHERE `card_id = cardId`; not found → 404 `CARD_NOT_FOUND` |
| 2 | `[DOMAIN]` | Decrypt `card.pan_encrypted` (AES-256-GCM, combineKey) → `pan`; decrypt `card.expiry_date` (AES-256-GCM) → `expiryMMYY` |
| 3 | `[RETURN]` | Return `CardPlainRs { pan, expiryMMYY, cvvItoken: card.cvv_itoken }` |

> `pan` and `expiryMMYY` exist only in JVM heap during this call. The `CardPlainRs` DTO must **never** be serialized to any log, metrics, or persistent storage (PCI DSS).
> `cvvItoken` is the HMAC-SHA256 token stored in `card.cvv_itoken`; it is returned as-is.

---

## 5. Database

**Table read:** `card`

| Column | Usage |
|--------|-------|
| `card_id` | Lookup key |
| `pan_encrypted` | AES-256-GCM decrypted → `pan` |
| `expiry_date` | AES-256-GCM decrypted → `expiryMMYY` |
| `cvv_itoken` | Returned as-is in response |

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | CA00001 | CARD_NOT_FOUND | No card found for given `cardId` |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 7. Changelog

### v1.0 — 2026-03 — Initial spec
