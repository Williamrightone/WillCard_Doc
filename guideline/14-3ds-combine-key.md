# 3DS combineKey Encryption (Phase 5)

This document describes how WillCard constructs and manages the combineKey used to decrypt card data
sent by NCCC during the 3DS authorization flow.
It covers the key hierarchy, Key Check Value (KCV) mechanism, DB schema, startup composition,
runtime decryption, source key rotation, and mock environment setup.
Implementation details (API contracts, UseCase flows) are in `spec/card-service/`.

> **Phase 5 scope:** Source key storage (insert / rotate), startup combineKey composition,
> and KCV verification. The combineKey is used by Phase 7 (Issuer Authorization) to decrypt
> NCCC-provided `encryptedCard` payloads. Mock NCCC usage is defined in Phase 8.

---

## 1. Overview

In the 3DS flow, NCCC's iframe receives the cardholder's PAN, expiry date, and CVV directly in the
browser. Before forwarding these to our Issuer Authorization API
(`POST /card-service/auth/authorize`), NCCC encrypts the card data using the **combineKey** —
a 256-bit AES key assembled from three source keys.

Card-service holds the combineKey exclusively in JVM heap memory. The three source keys are stored
encrypted (with a master key from KMS/env) in the `card_key_parts` table. On startup, card-service
reads and decrypts the three parts, XORs them to reconstruct the combineKey, and retains it as an
in-memory `SecretKey` object.

> **Distinction from Phase 1 card-at-rest encryption:**
> Phase 1 uses `CARD_ENCRYPTION_KEY` (from env/KMS) to encrypt PAN, expiry, and cardholder names
> in the `card` table. The combineKey is NCCC-channel-specific, entirely in-memory, and never
> written to any storage medium.

---

## 2. Key Architecture

```
MASTER_KEY (from KMS / env, never stored in DB)
  ├── AES-256-GCM encrypt(sourceKey1) → card_key_parts (part_seq = 1)
  ├── AES-256-GCM encrypt(sourceKey2) → card_key_parts (part_seq = 2)
  └── AES-256-GCM encrypt(sourceKey3) → card_key_parts (part_seq = 3)

combineKey = sourceKey1 XOR sourceKey2 XOR sourceKey3   (JVM heap only, never persisted)
```

| Key | Size | Storage | Lifecycle |
|-----|------|---------|-----------|
| `MASTER_KEY` | 256-bit AES | KMS / env var only | Permanent; rotated via Ops/KMS only |
| `sourceKey1/2/3` | 256-bit AES | `card_key_parts` (encrypted at rest) | Rotated periodically by NCCC |
| `combineKey` | 256-bit AES | JVM heap only | Rebuilt on startup and after each source key update; zero-filled on `@PreDestroy` |

> **Split knowledge:** No single party holds all three source keys. This mirrors the Payment Card
> Industry practice of key custodians each holding one component — no single operator can
> reconstruct the combineKey alone.

---

## 3. Key Check Value (KCV)

The KCV is a 3-byte fingerprint derived by encrypting a block of zeros with the key.
It lets an operator verify that a key was entered correctly without exposing the key itself.

### Computation

```
kcv = AES-256-ECB( sourceKey, byte[16]{ 0x00 } )[0..2]
    → expressed as 6 uppercase hexadecimal characters
```

| Parameter | Value |
|-----------|-------|
| Algorithm | AES-256-ECB (no IV; deterministic) |
| Plaintext | 16 bytes of `0x00` |
| Key | The 256-bit source key under test |
| Output | First 3 bytes → 6 uppercase hex chars (e.g., `A3F2B1`) |

> This follows ISO/TR 21188 KCV semantics adapted for AES-256 keys,
> consistent with HSM key-injection practices.

### Usage in Key Rotation

When NCCC rotates a source key, the operator receives the new key (as 64 hex chars) and its
expected KCV from NCCC. To inject the key:

1. Operator calls `PUT /card-service/internal/keys/source-key/{partSeq}` with `sourceKeyHex` and `kcv`.
2. card-service computes `kcv_computed = AES-256-ECB(sourceKey, zeros)[0:3]`.
3. `kcv_computed == kcv_provided` → key accepted, encrypted with MASTER_KEY, stored in DB.
4. `kcv_computed != kcv_provided` → rejected (`KCV_MISMATCH`); key is not stored.

### KCV Query

An operator can also query the KCV of any currently stored source key (without retrieving the key
itself) via `GET /card-service/internal/keys/source-key/{partSeq}/kcv`.
This confirms what is loaded without exposing sensitive key material.

---

## 4. DB Schema

### `card_key_parts`

> **Source migration:** `V5.0.0__create_card_key_parts_table.sql`
> **Last updated:** 2026-03

```sql
CREATE TABLE `card_key_parts` (
  `part_seq`        INT           NOT NULL COMMENT 'PK: Source key sequence (1, 2, or 3)',
  `encrypted_part`  VARCHAR(512)  NOT NULL COMMENT 'AES-256-GCM of the 256-bit source key; master key from env/KMS',
  `updated_at`      DATETIME(3)   NOT NULL COMMENT 'Timestamp of the last key injection or rotation',
  PRIMARY KEY (`part_seq`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_unicode_ci;
```

> `part_seq` is the natural PK (exactly 3 rows: 1, 2, 3). No surrogate key needed.
> `updated_at` provides an audit trail of when each source key was last rotated.
> There is no `created_at` — rows are seeded once at deployment and subsequently updated in-place.

### Initial Seed

Insert the initial 3 rows via a separate DML migration (`V5.0.1__seed_card_key_parts.sql`).
In production, seed values are the NCCC-provided initial source keys (encrypted with MASTER_KEY).
In dev/test, use the fixed mock values (see Section 8).

---

## 5. Startup Behavior

Triggered by `ApplicationReadyEvent` (fires after Spring context, JPA, and DB connections are ready):

```
Step 1: [DB READ]   SELECT * FROM card_key_parts ORDER BY part_seq
        → If row count < 3:
              log ERROR "card_key_parts has fewer than 3 rows; combineKey unavailable"
              set CombineKeyHolder.combineKey = null
              (auth requests will receive DECRYPTION_KEY_NOT_AVAILABLE)

Step 2: [DOMAIN]   For each row: decrypt encrypted_part with MASTER_KEY (AES-256-GCM)
                   → sourceKey1Bytes, sourceKey2Bytes, sourceKey3Bytes
        → Decryption error:
              log ERROR "Failed to decrypt source key part {partSeq}"
              set CombineKeyHolder.combineKey = null

Step 3: [DOMAIN]   combineKey[i] = sourceKey1[i] XOR sourceKey2[i] XOR sourceKey3[i]
                   (byte-by-byte XOR over 32 bytes)

Step 4: [DOMAIN]   CombineKeyHolder.set(new SecretKeySpec(combineKeyBytes, "AES"))
                   → Immediately zero-fill sourceKey1Bytes, sourceKey2Bytes, sourceKey3Bytes,
                     and combineKeyBytes after wrapping into SecretKeySpec

Step 5: [DOMAIN]   Log INFO "combineKey loaded successfully" (no key bytes in log)
```

**On `@PreDestroy` (JVM shutdown):**

```
CombineKeyHolder.destroy()
  → zero-fill the internal byte array backing the SecretKey
  → set combineKey reference to null
  → log INFO "combineKey zeroed"
```

---

## 6. Runtime Decryption (Phase 7)

When card-service receives `POST /card-service/auth/authorize` with `encryptedCard`:

```
encryptedCard (AES-256-GCM ciphertext from NCCC)
  → CombineKeyHolder.get()                         // throws if null
  → AES.decrypt(encryptedCard, combineKey, gcmIV)  // GCM IV is prepended to ciphertext
  → plaintext: { pan, expiryMMYY, cvv }
  → PAN used immediately for HMAC-SHA256 lookup (pan_hash)
  → CVV used immediately for in-memory verification only
  → Zero-fill plaintext byte array after use
```

**PCI DSS constraints on decrypted plaintext:**
- Exists only in JVM heap during the single request lifetime.
- Must not appear in any log, serialized object, or persisted record.
- CVV is zero-filled immediately after `verify-challenge` completes (or on any error path).

---

## 7. Source Key Rotation

**Rotation procedure (one part at a time):**

1. NCCC provides: new `sourceKeyHex` (64 hex chars), `kcv` (6 hex chars), and which `partSeq` to update.
2. Operator calls `PUT /card-service/internal/keys/source-key/{partSeq}` with the above values.
3. card-service verifies KCV → encrypts source key → upserts `card_key_parts` → reloads combineKey.
4. New combineKey takes effect immediately (in-memory atomic swap in `CombineKeyHolder`).

> **Rotation ordering:** Since the combineKey is the XOR of all 3 parts, rotating one part at a time
> means the combineKey changes after every individual rotation step. Coordinate with NCCC on when
> the new combined key takes effect (typically all 3 parts are rotated sequentially in one session).

> **In-flight transactions:** A transaction already in the NCCC iframe when a rotation occurs may
> have been encrypted with the old combineKey. After rotation, that `encryptedCard` will fail
> decryption. The user must re-initiate the transaction. This is expected behavior.

---

## 8. Mock / Dev Environment

In the `dev` and `local` Spring profiles:

- A fixed `MOCK_COMBINE_KEY` (256-bit, all zeros or a known test value) is loaded directly from
  `application-dev.yml` into `CombineKeyHolder`, bypassing `card_key_parts` DB lookup.
- `card_key_parts` is seeded with three parts whose XOR equals `MOCK_COMBINE_KEY`
  (for dev DB consistency only; the DB lookup path is still exercisable in integration tests).
- Mock NCCC service uses the same `MOCK_COMBINE_KEY` to encrypt card data before calling
  `POST /card-service/auth/authorize`.

```yaml
# application-dev.yml
willcard:
  security:
    mock-combine-key: "0000000000000000000000000000000000000000000000000000000000000000"
    # 64 hex chars = 256-bit all-zeros key for dev only
```

> **Never use the mock key in production or staging.** The dev key is intentionally trivial
> and offers no real security.

---

## 9. API List

### 9.1 Update Source Key (Internal)

| Field | Value |
|-------|-------|
| Operation | `PUT /card-service/internal/keys/source-key/{partSeq}` |
| Caller | Operator admin tool only |
| Auth | None (internal; Nginx blocks `/internal/` from external traffic) |
| Purpose | Inject or rotate one source key part with KCV verification; reloads combineKey |

Spec: [card-service/source-key-update.md](../spec/card-service/source-key-update.md)

### 9.2 Verify Source Key KCV (Internal)

| Field | Value |
|-------|-------|
| Operation | `POST /card-service/internal/keys/source-key/{partSeq}/kcv/verify` |
| Caller | Operator admin tool only |
| Auth | None (internal) |
| Purpose | Operator submits an expected KCV; service returns `matched: Boolean` — key material is never exposed |

Spec: [card-service/source-key-kcv.md](../spec/card-service/source-key-kcv.md)

### 9.3 CombineKey Startup Loader (Application Lifecycle)

| Field | Value |
|-------|-------|
| Trigger | `ApplicationReadyEvent` |
| Component | `CombineKeyStartupLoader` → `CombineKeyHolder` |
| Purpose | Decrypt 3 source key parts from DB, XOR-compose combineKey, store in JVM heap; zero-fill on `@PreDestroy` |

Spec: [card-service/combine-key-startup.md](../spec/card-service/combine-key-startup.md)

---

## 10. Security Requirements

| Requirement | Implementation |
|-------------|----------------|
| Source keys never in logs | All key byte arrays logged as `[REDACTED]`; `sourceKeyHex` input field masked in audit logs |
| combineKey never serialized | `CombineKeyHolder` exposes only `SecretKey`; raw bytes are inaccessible externally |
| MASTER_KEY never in DB | Injected from KMS / env var at startup only |
| Admin API transport security | Admin tool must call over internal network; Nginx blocks `/internal/` from outside |
| Key zero-fill on shutdown | `@PreDestroy` zero-fills combineKey backing byte array |
| CVV zero-fill after authorization | Cleared immediately after `verify-challenge` or on any failure path in Phase 7 |
| KCV does not reveal key material | Revealing the 3-byte KCV has negligible cryptographic impact on 256-bit keys |

---

## 11. Known Gaps (Phase 5 Scope)

| Gap | Notes |
|-----|-------|
| Rotation atomicity | combineKey hot-swap is single-field replace in `CombineKeyHolder`; concurrent decrypt calls during rotation may briefly use a transitional key |
| MASTER_KEY rotation | Requires re-encrypting all 3 `card_key_parts` rows; no tooling defined in Phase 5 |
| HSM integration | Source keys are provided as hex strings via the admin API; no physical HSM-based key injection |
| Key-event audit log | `updated_at` in `card_key_parts` records when rotation occurred; no dedicated key-rotation audit trail |
| Dual-control enforcement | Phase 5 does not enforce split-knowledge injection (requiring two separate operators for two parts each) |

---
