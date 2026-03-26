# Card Freeze — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** bff → [mobile-bff/card-freeze.md](../mobile-bff/card-freeze.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the freeze request from BFF via Feign.
Validates that the card exists, belongs to the requesting user, and is currently `ACTIVE`.
Transitions the card status from `ACTIVE` to `FROZEN`.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/freeze` |
| Effective received path | `/card/freeze` |
| Caller | BFF (`CardFeignClient`) |

### Request (`card.contract.dto.CardFreezeRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Target card Snowflake ID |

### Response (`card.contract.dto.CardStatusRs`)

| Field | Type | Description |
|-------|------|-------------|
| cardId | String | Card Snowflake ID (String) |
| status | String | `FROZEN` |

---

## 3. Service UseCase Flow

**UseCase:** `CardFreezeUseCase.freeze(CardFreezeRq)` (card-service)
**UseCase Impl:** `CardFreezeUseCaseImpl`
**Repository:** `CardJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query `card` by `card_id`; not found → throw `CARD_NOT_FOUND` (CA00001) | `CardQueryService.findById()` |
| 2 | `[DOMAIN]` | Validate `card.user_id == X-Member-Id`; mismatch → throw `CARD_NOT_FOUND` (CA00001) | `CardAuthorizationService.assertOwnership()` |
| 3 | `[DOMAIN]` | Validate `card.status == ACTIVE`; otherwise → throw `CARD_NOT_ACTIVE` (CA00002) | `CardStateService.assertActive()` |
| 4 | `[DB WRITE]` | Update `card.status = 'FROZEN'`, `updated_at = now()` | `CardJpaRepository.save()` |
| 5 | `[RETURN]` | Return `CardStatusRs` (cardId, status = `FROZEN`) | — |

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card` | READ | Query card by `card_id` |
| `card` | WRITE | Update `status` to `FROZEN` |

### Redis

> This endpoint performs no Redis operations.

### Table Schema

#### card (key columns for this operation)

| Column | Type | Description |
|--------|------|-------------|
| card_id | BIGINT | PK |
| user_id | BIGINT | Ownership check |
| status | VARCHAR(20) | Validated (must be ACTIVE); updated to FROZEN |
| updated_at | DATETIME(3) | Updated by BaseTimeEntity |

> Full card table schema: see [card-apply.md](./card-apply.md#table-schema).

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | CA00001 | CARD_NOT_FOUND | Card does not exist, or `user_id` does not match `X-Member-Id` |
| 409 | CA00002 | CARD_NOT_ACTIVE | Card status is not `ACTIVE` (already FROZEN or CANCELLED) |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
