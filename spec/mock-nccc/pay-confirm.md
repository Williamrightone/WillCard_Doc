# Mock NCCC Pay Confirm — Biz Spec

## Overview

**Mock NCCC is a standalone Spring Boot microservice** (Maven module: `mock-nccc`,
deployed only in the development Kubernetes cluster). This endpoint is the second step
of the Mock NCCC payment simulation. Receives the cardholder's OTP submission and
`challengeRef` from the App (via BFF Feign), and forwards them directly to the Card
Service Issuer Authorization API Phase 2.
Returns the final `APPROVED` or `DECLINED` result transparently.

> **Scope:** Development and integration testing only. Must not be deployed to production.

---

## API Definition — mock-nccc

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/mock-nccc` |
| Controller mapping | `/pay/confirm` |
| Effective received path | `/mock-nccc/pay/confirm` |
| Caller | BFF (FeignClient) |

### Request (MockNcccPayConfirmRq — mock-nccc-contract)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| challengeRef | String | ✅ | @NotBlank | OTP session reference from the `/mock-nccc/pay` response |
| otp | String | ✅ | @NotBlank, @Pattern(regexp="^\\d{6}$") | 6-digit OTP submitted by cardholder |

### Response (MockNcccPayConfirmRs — mock-nccc-contract)

| Field | Type | Description | Notes |
|-------|------|-------------|-------|
| status | String | `APPROVED` / `DECLINED` | |
| authCode | String | Authorization code | Present on `APPROVED` |
| txnId | String | Snowflake transaction ID | Present on `APPROVED` |
| pointsUsed | Integer | Points actually deducted | Present on `APPROVED` |
| cardAmount | BigDecimal | Final amount charged to card (TWD) | Present on `APPROVED` |
| declineReason | String | `OTP_FAILED` / `SESSION_EXPIRED` / `CARD_LOCKED` | Present on `DECLINED` |
| attemptsRemaining | Integer | Remaining attempts for this session | Present when `declineReason = OTP_FAILED` |

---

## Service UseCase Flow

**UseCase:** `MockNcccPayConfirmUseCase.confirm(MockNcccPayConfirmRq)`

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[FEIGN]` | Call Card Service `POST /card/auth/verify-challenge` with `{ challengeRef, otp }` → [spec/card-service/card-auth-verify-challenge.md](../card-service/card-auth-verify-challenge.md) | — |
| 2 | `[RETURN]` | Pass through the `CardAuthVerifyChallengeRs` response as `MockNcccPayConfirmRs` | — |

---

## Database

### MySQL

_(None — Mock NCCC is stateless)_

### Redis

_(None)_

---

## Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 422 | `SESSION_EXPIRED` | OTP session expired | Passed through from Card Service |
| 422 | `OTP_FAILED` | Incorrect OTP | Passed through from Card Service |
| 422 | `CARD_LOCKED` | Card temporarily locked | Passed through from Card Service |
| 500 | `INTERNAL_ERROR` | Unexpected system error | Unhandled exception |
