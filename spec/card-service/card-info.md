# Card Info — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** bff → [mobile-bff/card-info.md](../mobile-bff/card-info.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives a card info query from BFF via Feign.
Validates that the card exists and belongs to the requesting user.
Decrypts `expiry_date`, `cardholder_name_zh`, and `cardholder_name_en` in JVM heap, then assembles and returns the card record.
`pan_encrypted` is never included in the response — only `pan_masked` is returned.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| context-path | `/card` |
| Controller mapping | `/info` |
| Effective received path | `/card/info` |
| Caller | BFF (`CardFeignClient`) |

### Request (`card.contract.dto.CardInfoRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Target card Snowflake ID |

> **Note:** GET request — field is passed as a query parameter (`?cardId=...`).

### Response (`card.contract.dto.CardInfoRs`)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| cardId | String | Snowflake ID (String) | |
| maskedPan | String | `****-****-****-1234` | Returned as stored; no decryption needed |
| cardType | String | Card type | |
| cardNetwork | String | Card network | |
| expiryDate | String | `MM/YY` format | Decrypted from `expiry_date` (stored as encrypted MMYY) |
| cardholderNameZh | String | Chinese name | Decrypted from `cardholder_name_zh` |
| cardholderNameEn | String | English name | Decrypted from `cardholder_name_en` |
| status | String | ACTIVE / FROZEN / CANCELLED | |
| activatedAt | String | ISO-8601 timestamp | null if not yet activated |

---

## 3. Service UseCase Flow

**UseCase:** `CardInfoUseCase.getInfo(CardInfoRq)` (card-service)
**UseCase Impl:** `CardInfoUseCaseImpl`
**Repository:** `CardJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query `card` by `card_id`; not found → throw `CARD_NOT_FOUND` (CA00001) | `CardQueryService.findById()` |
| 2 | `[DOMAIN]` | Validate `card.user_id == X-Member-Id`; mismatch → throw `CARD_NOT_FOUND` (CA00001, do not expose ownership info) | `CardAuthorizationService.assertOwnership()` |
| 3 | `[DOMAIN]` | Decrypt `expiry_date` (MMYY) → format as `MM/YY`; decrypt `cardholder_name_zh`; decrypt `cardholder_name_en` | `CardEncryptionService.decrypt()` |
| 4 | `[RETURN]` | Assemble and return `CardInfoRs` (maskedPan from DB as-is; decrypted fields from Step 3) | — |

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card` | READ | Query card record by `card_id` |

### Redis

> This endpoint performs no Redis operations.

### Table Schema

#### card (key columns for this operation)

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| card_id | BIGINT | NO | PK |
| user_id | BIGINT | NO | Ownership check: must equal X-Member-Id |
| card_type | VARCHAR(20) | NO | — |
| card_network | VARCHAR(10) | NO | — |
| status | VARCHAR(20) | NO | — |
| pan_masked | VARCHAR(20) | NO | Returned as-is |
| cardholder_name_zh | VARCHAR(512) | NO | AES-256-GCM encrypted; decrypted in JVM heap |
| cardholder_name_en | VARCHAR(512) | NO | AES-256-GCM encrypted; decrypted in JVM heap |
| expiry_date | VARCHAR(512) | NO | AES-256-GCM encrypted MMYY; decrypted and formatted as MM/YY |
| activated_at | DATETIME(3) | YES | — |

> Full card table schema: see [card-apply.md](./card-apply.md#table-schema).

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | CA00001 | CARD_NOT_FOUND | Card does not exist, or `user_id` does not match `X-Member-Id` |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
