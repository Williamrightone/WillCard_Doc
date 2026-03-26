# Card Benefits — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** bff → [mobile-bff/card-benefits.md](../mobile-bff/card-benefits.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives a merchant benefit query from BFF via Feign.
Queries `card_type_merchant_benefit` for all entries matching the given `card_type` where the benefit is currently active (today falls within `[effective_from, effective_to]`, or `effective_to IS NULL`).
Returns the full list of matching benefit records.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | GET |
| context-path | `/card` |
| Controller mapping | `/benefits` |
| Effective received path | `/card/benefits` |
| Caller | BFF (`CardFeignClient`) |

### Request (`card.contract.dto.CardBenefitsRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardType | String | ✅ | @NotBlank | CLASSIC / OVERSEAS / PREMIUM / INFINITE |

> **Note:** GET request — field is passed as a query parameter (`?cardType=CLASSIC`).

### Response (`card.contract.dto.CardBenefitsRs`)

| Field | Type | Description |
|-------|------|-------------|
| cardType | String | Card type queried |
| benefits | List\<CardBenefitItem\> | Active benefit records; empty list if none |

**CardBenefitItem:**

| Field | Type | Description |
|-------|------|-------------|
| benefitId | String | Snowflake ID (String) |
| merchantId | String | Merchant ID; null = applies to entire MCC |
| mccCode | String | MCC code; null = applies to all merchants |
| bonusRate | BigDecimal | Additional reward rate (e.g., 0.0300) |
| descriptionZh | String | Chinese description |
| descriptionEn | String | English description |
| effectiveFrom | String | `YYYY-MM-DD` |
| effectiveTo | String | `YYYY-MM-DD`; null = indefinite |

---

## 3. Service UseCase Flow

**UseCase:** `CardBenefitsUseCase.getBenefits(CardBenefitsRq)` (card-service)
**UseCase Impl:** `CardBenefitsUseCaseImpl`
**Repository:** `CardBenefitJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query `card_type_merchant_benefit` WHERE `card_type = ?` AND `effective_from <= today` AND (`effective_to IS NULL` OR `effective_to >= today`); order by `effective_from ASC` | `CardBenefitQueryService.findActiveByType()` |
| 2 | `[RETURN]` | Assemble and return `CardBenefitsRs` | — |

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card_type_merchant_benefit` | READ | Query active benefits by `card_type` and date range |

### Redis

> This endpoint performs no Redis operations.

### Table Schema

#### card_type_merchant_benefit (seed data, read-only)

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| benefit_id | BIGINT | NO | PK, Snowflake ID |
| card_type | VARCHAR(20) | NO | CLASSIC / OVERSEAS / PREMIUM / INFINITE |
| merchant_id | VARCHAR(50) | YES | Specific merchant; null = applies to entire MCC |
| mcc_code | VARCHAR(10) | YES | MCC category; null = applies to all MCCs |
| bonus_rate | DECIMAL(5,4) | NO | Additional reward rate on top of base + MCC bonus |
| description_zh | VARCHAR(255) | NO | Benefit description (Chinese) |
| description_en | VARCHAR(255) | NO | Benefit description (English) |
| effective_from | DATE | NO | Benefit start date |
| effective_to | DATE | YES | Benefit end date; null = indefinitely active |
| created_at | DATETIME(3) | NO | — |
| updated_at | DATETIME(3) | NO | — |

> **Query condition:** `effective_from <= today AND (effective_to IS NULL OR effective_to >= today)`

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | CA00008 | CARD_TYPE_NOT_FOUND | Unrecognized `cardType` value |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
