# Card Management

This document describes the lifecycle management, design principles, and compliance rules for WillCard virtual cards.
It covers card types, state machine, PCI DSS compliance, encryption strategy, and limit management.
Implementation details (API contracts, DB schema, UseCase flows) are in the corresponding specs under `spec/card-service/` and `spec/mobile-bff/`.

---

## 1. Card Types

WillCard issues four types of virtual credit cards. Each type has distinct reward rates, transaction limits, and applicable benefits.

| Card Type | Name | Domestic Reward | Overseas Reward | FX Fee | Single Limit | Daily Limit |
|-----------|------|----------------|-----------------|--------|--------------|-------------|
| CLASSIC | WillCard Classic | 1% | 0% | 1.5% | NT$100,000 | NT$200,000 |
| OVERSEAS | WillCard Overseas | 0% | 5% | 1% | NT$100,000 | NT$200,000 |
| PREMIUM | WillCard Premium | 2% | 5% | 1% | NT$200,000 | NT$500,000 |
| INFINITE | WillCard Infinite | 2% | 5% | 1% | Unlimited | Unlimited |

Card type attributes are stored in the `card_type_limit` table as **seed data** (Flyway DML). These values are **read-only at runtime** — no API modifies them.

**One Card Per Type Per User:** Each user may hold at most one active card of each type. A new application for a card type is rejected if the user already has a PENDING, ACTIVE, or FROZEN card of that type. This constraint is enforced at the **application layer** (service-level duplicate check) before the DB write — not via a DB-level unique constraint, because the constraint is conditional on `status != 'CANCELLED'`.

---

## 2. Card Networks

Each card may be issued on one of three card networks:

| card_network | BIN Prefix |
|-------------|-----------|
| VISA | 4 (e.g., 4xxx-xxxx-xxxx-xxxx) |
| MASTERCARD | 51–55 (e.g., 5xxx-xxxx-xxxx-xxxx) |
| JCB | 35 (e.g., 35xx-xxxx-xxxx-xxxx) |

PAN generation uses the BIN prefix of the chosen network, fills the remaining digits with `SecureRandom`, and validates the full 16-digit number with the **Luhn algorithm**. This is a **mock implementation** suitable for development and testing environments only.

---

## 3. Card Lifecycle (State Machine)

```
PENDING ──► ACTIVE ──► FROZEN
               ▲          │
               └──────────┘
               │
ACTIVE  ──► CANCELLED
FROZEN  ──► CANCELLED
```

| Status | Description |
|--------|-------------|
| PENDING | Application received; awaiting activation (reserved for future approval workflows) |
| ACTIVE | Card is usable for transactions |
| FROZEN | Card is temporarily suspended; no new authorizations allowed |
| CANCELLED | Card is permanently closed; cannot be reactivated |

**Legal transitions:**

| From | To | Trigger |
|------|----|---------|
| PENDING | ACTIVE | Application approved (MVP: immediate approval, no review step) |
| ACTIVE | FROZEN | User initiates freeze |
| FROZEN | ACTIVE | User initiates unfreeze |
| ACTIVE | CANCELLED | User or system cancels card |
| FROZEN | CANCELLED | User or system cancels card |

`activated_at` is written when status first transitions to ACTIVE. It is never overwritten.
`cancelled_at` is written when status transitions to CANCELLED. It is never overwritten.
Both are retained for audit and PCI DSS compliance.

---

## 4. PCI DSS Compliance Principles

Card-service is the **only service** that handles sensitive cardholder data (CHD). All other services operate with masked or hashed representations only.

### Encryption at Rest

The following fields are encrypted with **AES-256-GCM** before being persisted to the database:

| Field | Encrypted | Rationale |
|-------|-----------|-----------|
| `pan_encrypted` | ✅ | Full PAN — PCI DSS requires encryption at rest |
| `pan_masked` | ❌ (stored as-is) | Already sanitized; last 4 digits only |
| `pan_hash` | ❌ (HMAC-SHA256) | Non-reversible; used for PAN-based lookup |
| `cardholder_name_zh` | ✅ | Personal data |
| `cardholder_name_en` | ✅ | Personal data (card face printing) |
| `expiry_date` | ✅ | Card data (MMYY format) |

### Key Management

- AES-256 encryption keys and the `combineKey` master key are injected via **environment variables or KMS**.
- Keys are **never** stored in the database or source code.
- The `combineKey` is loaded into JVM heap at startup (`ApplicationReadyEvent`) and cleared on shutdown (`@PreDestroy`).
- See Phase 5 (`combineKey` infrastructure) for the full key management design.

### Decryption Scope

- Decryption occurs **only inside card-service**.
- Decrypted plaintext exists only in **JVM heap memory** — it is not serialized, logged, or passed in Kafka payloads.
- The BFF and other services never receive decrypted PAN or card data.

### Log Sanitization

- Logback filter intercepts all log output and replaces any pattern matching a full PAN or CVV with a masked placeholder.
- `pan_encrypted` in DB does not appear in logs. The risk is in the decrypted value — the filter is the last-line defense.

---

## 5. CVV Policy

CVV is a 3-digit security code generated at card application time.

| Rule | Policy |
|------|--------|
| Generation | `SecureRandom` at application time, in card-service JVM |
| Storage | **Never stored** — not in DB, Redis, logs, or any persistent layer |
| Delivery | Returned in the `card-apply` response **once only** |
| Re-query | CVV cannot be retrieved after the initial apply response |
| Authorization use | CVV is part of the encrypted card payload during authorization; decrypted in card-service JVM heap only; cleared immediately after the authorization decision |

This policy ensures full PCI DSS compliance for CVV handling (Requirement 3.2.1: do not store sensitive authentication data after authorization).

---

## 6. Masked PAN Convention

All display contexts and event payloads use the masked PAN format:

```
****-****-****-1234
```

`pan_masked` is derived from the full PAN at application time and stored as plain text (no decryption needed for display).
Kafka event payloads must use `maskedPan` only. Full PAN must never appear in Kafka events, logs, or API responses (except the one-time `pan_encrypted` used internally during authorization).

---

## 7. PAN Hash for Authorization Lookup

Phase 7 (Issuer Authorization) identifies the cardholder by PAN — "由 PAN 識別會員（pan_hash 查詢）".
To support this without storing plain PAN, card-service generates `pan_hash = HMAC-SHA256(pan, hmacKey)` at application time.

- `pan_hash` is stored in the `card` table with a **unique index**.
- During authorization, NCCC provides the encrypted PAN; card-service decrypts it in JVM heap and computes the hash to perform a DB lookup.
- `hmacKey` is managed identically to the AES encryption keys (env var / KMS; never in DB).

---

## 8. Reward Rate Structure

Transaction reward rates are layered. The final rate for a transaction is the sum of three components:

```
final_rate = base_rate + mcc_bonus + merchant_bonus
```

| Layer | Source Table | Defined in Phase |
|-------|-------------|-----------------|
| `base_rate` | `card_type_limit.domestic_reward_rate` or `overseas_reward_rate` | Phase 1 |
| `mcc_bonus` | `reward_plan` (MCC-based, may be scoped to a card type) | Phase 3 |
| `merchant_bonus` | `card_type_merchant_benefit` (merchant-specific extra bonus) | Phase 1 |

`card_type_merchant_benefit` stores optional extra reward rates scoped to a specific `card_type` and optionally a `merchant_id` or `mcc_code`. It is managed as **seed data** (Flyway DML) and is read-only at runtime.

---

## 9. Transaction Limit Management

| Limit Type | Storage | Scope | Notes |
|------------|---------|-------|-------|
| Single-transaction limit | `card_type_limit.single_txn_limit` (DB, BIGINT cents) | Per card type | NULL = unlimited |
| Daily cumulative limit | `card:daily:{cardId}:{yyyyMMdd}` (Redis, BIGINT cents) | Per card per calendar day | TTL = end of day (86400s) |

- **INFINITE** card type has NULL limits — all limit checks are skipped for this type.
- Daily limit tracking is incremented atomically at authorization time (Phase 7). Phase 1 only defines the Redis key structure; no writes occur during card management.

---

## 10. Idempotency and CVV Conflict

`card-apply` does **not** use the standard `Idempotency-Key` mechanism.

**Reason:** The one-time CVV in the apply response cannot be cached in Redis (`idempotency:{key}` value contains `responseBody`). Storing CVV in Redis would violate PCI DSS Requirement 3.2.1.

**Duplicate prevention:** Server-side uniqueness validation (`CARD_ALREADY_EXISTS`, CA00009) prevents duplicate applications for the same card type. Clients must treat `CARD_ALREADY_EXISTS` as a terminal failure and not retry the apply request.

Other card management endpoints (freeze, unfreeze) are **not financial operations** per the spec guideline definition and therefore also do not require `Idempotency-Key`.

---

## 11. Known Gaps (Phase 1 Scope)

The following items are out of scope for Phase 1 and should be addressed in later phases:

| Gap | Notes |
|-----|-------|
| Card cancellation API | State machine includes CANCELLED, but no API is defined in Phase 1 |
| Card list API | No `GET /api/v1/card/list` endpoint; clients must store `cardId` from apply response |
| Cardholder name source | Phase 1 requires client to supply both names in the apply request; a future phase should auto-fill from member profile |

---

Next Chapter: [FX Service](/guideline/10-txn-flow.md)
