# Card Unfreeze — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** bff → [mobile-bff/card-unfreeze.md](../mobile-bff/card-unfreeze.md)
> **Last Updated:** 2026-03

---

## 1. Overview

Receives the unfreeze request from BFF via Feign.
Validates that the card exists, belongs to the requesting user, and is currently `FROZEN`.
Transitions the card status from `FROZEN` to `ACTIVE`.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/unfreeze` |
| Effective received path | `/card/unfreeze` |
| Caller | BFF (`CardFeignClient`) |

### Request (`card.contract.dto.CardUnfreezeRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Target card Snowflake ID |

### Response (`card.contract.dto.CardStatusRs`)

| Field | Type | Description |
|-------|------|-------------|
| cardId | String | Card Snowflake ID (String) |
| status | String | `ACTIVE` |

---

## 3. Service UseCase Flow

**UseCase:** `CardUnfreezeUseCase.unfreeze(CardUnfreezeRq)` (card-service)
**UseCase Impl:** `CardUnfreezeUseCaseImpl`
**Repository:** `CardJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Query `card` by `card_id`; not found → throw `CARD_NOT_FOUND` (CA00001) | `CardQueryService.findById()` |
| 2 | `[DOMAIN]` | Validate `card.user_id == X-Member-Id`; mismatch → throw `CARD_NOT_FOUND` (CA00001) | `CardAuthorizationService.assertOwnership()` |
| 3 | `[DOMAIN]` | Validate `card.status == FROZEN`; otherwise → throw `CARD_FROZEN` (CA00004) | `CardStateService.assertFrozen()` |
| 4 | `[DB WRITE]` | Update `card.status = 'ACTIVE'`, `updated_at = now()` | `CardJpaRepository.save()` |
| 5 | `[RETURN]` | Return `CardStatusRs` (cardId, status = `ACTIVE`) | — |

> **Note:** CA00004 (`CARD_FROZEN`) is repurposed here as "card is frozen (already in this state, cannot freeze again)" for the unfreeze flow — the condition being checked is `status != FROZEN`, so the error indicates the card is not in the expected FROZEN state. BFF maps this to `CARD_NOT_FROZEN`.

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card` | READ | Query card by `card_id` |
| `card` | WRITE | Update `status` to `ACTIVE` |

### Redis

> This endpoint performs no Redis operations.

### Table Schema

#### card (key columns for this operation)

| Column | Type | Description |
|--------|------|-------------|
| card_id | BIGINT | PK |
| user_id | BIGINT | Ownership check |
| status | VARCHAR(20) | Validated (must be FROZEN); updated to ACTIVE |
| updated_at | DATETIME(3) | Updated by BaseTimeEntity |

> Full card table schema: see [card-apply.md](./card-apply.md#table-schema).

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 404 | CA00001 | CARD_NOT_FOUND | Card does not exist, or `user_id` does not match `X-Member-Id` |
| 409 | CA00004 | CARD_FROZEN | Card status is not `FROZEN` (already ACTIVE or CANCELLED) |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |
