# Card Pay — Transaction Orchestrator Spec

> **Service:** txn-orch
> **Contract Module:** txn-orch-contract
> **Caller:** [mobile-bff/card-pay.md](../mobile-bff/card-pay.md)
> **Last Updated:** 2026-03

---

## 1. Overview

**Leg 1** of the card payment Saga. Receives the payment initiation request from BFF, performs FX conversion, computes points-to-use, forwards the request to Mock NCCC (which handles card validation and OTP generation), reserves wallet points if applicable, and stores Saga State for Leg 2.

Returns `challengeRef`, `pointsToUse`, and `estimatedTwdAmount` to BFF.

> ORCH owns all Saga coordination: FX conversion, wallet reserve/confirm/release, Saga State management, and Kafka publishing. card-service has no cross-service calls.

---

## 2. Connected Specs

| Interface | Spec |
|-----------|------|
| Caller (BFF) | [mobile-bff/card-pay.md](../mobile-bff/card-pay.md) |
| ORCH → FX Service (foreign currency) | [fx-service/convert.md](../fx-service/convert.md) |
| ORCH → Wallet Service (balance query + reserve) | [wallet-service/reserve.md](../wallet-service/reserve.md) |
| ORCH → Mock NCCC (card validation + OTP) | [mock-nccc/pay.md](../mock-nccc/pay.md) |
| Leg 2 (OTP confirmation) | [txn-orch/card-pay-confirm.md](card-pay-confirm.md) |

---

## 3. API Definition — txn-orch

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | /txn-orch |
| Controller mapping | /card/pay |
| Effective received path | /txn-orch/card/pay |
| Caller | BFF (TxnOrchFeignClient) |

### Request Header

| Header | Required | Description |
|--------|----------|-------------|
| X-Member-Id | ✅ | Authenticated user Snowflake ID (forwarded by BFF from Security Context) |

### Request (`txn-orch.contract.dto.CardPayOrchRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| cardId | String | ✅ | @NotBlank | Card to charge (Snowflake ID as String) |
| amount | Long | ✅ | @NotNull; @Min(1) | Transaction amount in cents |
| currency | String | ✅ | @NotBlank; @Size(min=3,max=3) | ISO 4217 currency code (e.g. `TWD`, `USD`) |
| merchantId | String | ✅ | @NotBlank | Merchant identifier |
| usePoints | Boolean | ✅ | @NotNull | Whether to apply points offset; handled by ORCH only, not forwarded downstream |

### Response (`txn-orch.contract.dto.CardPayOrchRs`)

| Field | Type | Description |
|-------|------|-------------|
| challengeRef | String | UUID; OTP session binding; pass to Leg 2 confirm endpoint; valid 180s |
| pointsToUse | Long | Points to be deducted in cents; `0` if usePoints = false or wallet balance = 0 |
| estimatedTwdAmount | Long | TWD equivalent after FX conversion (= txnTwdBaseAmount); before points deduction |

---

## 4. Service UseCase Flow

**UseCase:** `CardPayOrchUseCase.initiate(CardPayOrchRq)` (txn-orch)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[FEIGN]` | **FX Conversion:** if `currency = TWD` → `txnTwdBaseAmount = amount`; else call `FxServiceFeignClient.convert(amount, currency, "TWD")` → `POST /fx-service/convert` → `txnTwdBaseAmount`; see [fx-service/convert.md](../fx-service/convert.md) |
| 2 | `[DOMAIN]` | `isOverseas = (currency != "TWD")` |
| 3 | `[FEIGN]` | **Points Calculation:** if `usePoints = true` → `WalletFeignClient.getBalance(userId)` → `availableBalance`; `pointsToUse = min(availableBalance, txnTwdBaseAmount)`; else `pointsToUse = 0`; see [wallet-service/reserve.md](../wallet-service/reserve.md) |
| 4 | `[FEIGN]` | `MockNcccFeignClient.pay(MockNcccPayRq)` → `POST /mock-nccc/pay`; request: `{ cardId, amount, currency, txnTwdBaseAmount, merchantId }`; response: `{ challengeRef }`; see [mock-nccc/pay.md](../mock-nccc/pay.md) |
| 5 | `[FEIGN]` | **Wallet Reserve:** if `pointsToUse > 0` → `WalletFeignClient.reserve(userId, pointsToUse)` → `reservationId`; else `reservationId = null`; see [wallet-service/reserve.md](../wallet-service/reserve.md) |
| 6 | `[REDIS WRITE]` | SET `saga:{challengeRef}` = SagaState (TTL 180s); see Saga State section |
| 7 | `[RETURN]` | Return `CardPayOrchRs { challengeRef, pointsToUse, estimatedTwdAmount: txnTwdBaseAmount }` |

> **Step 4 failure** (card validation failed, limit exceeded) → propagate error to BFF; no cleanup needed (Step 5 not yet executed).
> **Step 5 failure** (wallet reserve error) → return error to BFF; OTP session in card-service will expire naturally (TTL 180s).

---

## 5. Saga State

**Redis Key:** `saga:{challengeRef}`
**TTL:** 180 seconds
**Written at:** Step 6 (Leg 1)
**Read at:** Leg 2 Step 1
**Deleted at:** Leg 2 — APPROVED path (Step 5) and DECLINED SESSION_VOIDED / CARD_LOCKED path (Step 8)

```json
{
  "userId": 123456789,
  "cardId": 987654321,
  "amount": 10000,
  "currency": "USD",
  "txnTwdBaseAmount": 316000,
  "isOverseas": true,
  "merchantId": "MERCHANT_001",
  "pointsToUse": 5000,
  "reservationId": 111222333
}
```

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| userId | Long | X-Member-Id header | Authenticated user Snowflake ID |
| cardId | Long | Request | Card Snowflake ID |
| amount | Long | Request | Original transaction amount in cents |
| currency | String | Request | ISO 4217 currency code |
| txnTwdBaseAmount | Long | Step 1 | FX-converted TWD base amount (= amount if TWD) |
| isOverseas | Boolean | Step 2 | Whether this is an overseas transaction |
| merchantId | String | Request | Merchant identifier |
| pointsToUse | Long | Step 3 | Points to deduct in cents; `0` if not applicable |
| reservationId | Long | Step 5 | Wallet reservation Snowflake ID; `null` if pointsToUse = 0 |

---

## 6. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 409 | CA00002 | CARD_NOT_ACTIVE | Card is not ACTIVE (propagated: card-service → Mock NCCC → ORCH) |
| 409 | CA00003 | CARD_EXPIRED | Card has expired (propagated) |
| 409 | CA00004 | CARD_FROZEN | Card is frozen (propagated) |
| 422 | CA00006 | SINGLE_TXN_LIMIT_EXCEEDED | Transaction exceeds single-transaction limit (propagated) |
| 422 | CA00007 | DAILY_LIMIT_EXCEEDED | Daily cumulative limit exceeded (propagated) |
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | card-service combineKey not loaded (propagated) |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception |

---

## 7. Changelog

### v1.0 — 2026-03 — Initial spec
