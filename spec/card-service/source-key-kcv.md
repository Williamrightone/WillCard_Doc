# Source Key KCV Verify — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** Operator admin tool (internal only)
> **Last Updated:** 2026-03

---

## 1. Overview

Internal API for verifying that a given Key Check Value matches the source key currently stored
for a given part sequence.

The operator provides the expected KCV (6 hex chars). The service decrypts the stored source key,
independently computes its KCV, and returns a boolean indicating whether they match.

The raw source key and its computed KCV are never exposed in the response — only the boolean result
is returned. This allows an operator to confirm "is the key I expect currently loaded?" without any
key material leaving the service.

> This endpoint is only accessible from the internal network.
> Nginx blocks all `/internal/` paths from external traffic.

See [guideline/14-3ds-combine-key.md](../../guideline/14-3ds-combine-key.md) for the KCV
computation specification.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | POST |
| context-path | `/card` |
| Controller mapping | `/internal/keys/source-key/{partSeq}/kcv/verify` |
| Effective received path | `/card/internal/keys/source-key/{partSeq}/kcv/verify` |
| Caller | Operator admin tool |

### Path Variable

| Variable | Type | Required | Validation | Description |
|----------|------|----------|------------|-------------|
| partSeq | Integer | ✅ | 1, 2, or 3 | Source key part sequence number |

### Request (`card.contract.dto.SourceKeyKcvVerifyRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| kcv | String | ✅ | @NotBlank, length = 6, valid hex | Expected KCV to verify against the currently stored key |

### Response (`card.contract.dto.SourceKeyKcvVerifyRs`)

| Field | Type | Description |
|-------|------|-------------|
| partSeq | Integer | Source key part sequence number |
| matched | Boolean | `true` if the provided KCV matches the stored key; `false` otherwise |
| updatedAt | String | ISO-8601 timestamp of the last key update for this part |

---

## 3. Service UseCase Flow

**UseCase:** `SourceKeyKcvVerifyUseCase.verify(int partSeq, SourceKeyKcvVerifyRq rq)` (card-service)
**UseCase Impl:** `SourceKeyKcvVerifyUseCaseImpl`
**Repository:** `CardKeyPartsJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[VALIDATE]` | Validate `partSeq` ∈ {1, 2, 3}; invalid value → throw `INVALID_KEY_FORMAT` (CA00012) | — |
| 2 | `[DB READ]` | `SELECT * FROM card_key_parts WHERE part_seq = partSeq`; not found → throw `KEY_PART_NOT_FOUND` (CA00010) | `CardKeyPartsJpaRepository.findByPartSeq()` |
| 3 | `[DOMAIN]` | Decrypt `encrypted_part` with `MASTER_KEY` (AES-256-GCM) → `sourceKeyBytes` | `CardEncryptionService.decryptKeyPart()` |
| 4 | `[DOMAIN]` | Compute KCV: `AES-256-ECB(sourceKeyBytes, byte[16]{0x00})[0..2]` → `kcvComputed` (6 uppercase hex) | `KcvService.compute()` |
| 5 | `[DOMAIN]` | Zero-fill `sourceKeyBytes` | `KeyZeroizer.zeroFill()` |
| 6 | `[DOMAIN]` | `matched = kcvComputed.equalsIgnoreCase(rq.kcv)` | — |
| 7 | `[RETURN]` | Return `SourceKeyKcvVerifyRs` (`partSeq`, `matched`, `updatedAt` from DB row) | — |

> **Step 5:** `sourceKeyBytes` must be zero-filled in a `try-finally` block to guarantee cleanup
> regardless of whether Step 4 succeeds or fails.
>
> **No error on mismatch:** A KCV mismatch (`matched = false`) is a valid business outcome, not an
> exception. The caller receives HTTP 200 with `matched: false`. Only structural errors (invalid
> `partSeq`, missing row, decryption failure) result in non-200 responses.

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card_key_parts` | READ | Fetch the encrypted source key row by `part_seq` |

### Redis

> This endpoint performs no Redis operations.

### Table Schema

#### card_key_parts

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| part_seq | INT | NO | — | PK: source key sequence (1, 2, or 3) |
| encrypted_part | VARCHAR(512) | NO | — | AES-256-GCM encrypted 256-bit source key; master key from env/KMS |
| updated_at | DATETIME(3) | NO | — | Last injection or rotation timestamp |

---

## 5. Error Codes

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 400 | CA00012 | INVALID_KEY_FORMAT | `partSeq` not in {1, 2, 3}; or `kcv` is not 6 valid hex characters |
| 404 | CA00010 | KEY_PART_NOT_FOUND | No `card_key_parts` row exists for the given `partSeq` |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception (e.g. MASTER_KEY decryption failure) |
