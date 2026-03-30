# Source Key Update — card-service Spec

> **Service:** card-service
> **Contract Module:** card-contract
> **Caller:** Operator admin tool (internal only)
> **Last Updated:** 2026-03

---

## 1. Overview

Internal API for injecting or rotating one of the three source keys used to compose the combineKey.

Accepts the source key as a 64-hex-character string (256-bit key) along with its Key Check Value (KCV).
Verifies the KCV by independently computing it from the provided key material.
On verification success, encrypts the source key with the master key (from env/KMS) and upserts
the corresponding row in `card_key_parts`. Then reloads all three parts from DB and reconstructs
the in-memory combineKey.

> This endpoint is only accessible from the internal network.
> Nginx blocks all `/internal/` paths from external traffic.

See [guideline/14-3ds-combine-key.md](../../guideline/14-3ds-combine-key.md) for the full combineKey
architecture and KCV computation specification.

---

## 2. API Definition — card-service

### Endpoint

| Field | Value |
|-------|-------|
| Method | PUT |
| context-path | `/card` |
| Controller mapping | `/internal/keys/source-key/{partSeq}` |
| Effective received path | `/card/internal/keys/source-key/{partSeq}` |
| Caller | Operator admin tool |

### Path Variable

| Variable | Type | Required | Validation | Description |
|----------|------|----------|------------|-------------|
| partSeq | Integer | ✅ | 1, 2, or 3 | Source key part sequence number |

### Request (`card.contract.dto.SourceKeyUpdateRq`)

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| sourceKeyHex | String | ✅ | @NotBlank, length = 64, valid hex | 256-bit source key encoded as 64 hex characters (case-insensitive) |
| kcv | String | ✅ | @NotBlank, length = 6, valid hex | Key Check Value — first 3 bytes of AES-256-ECB(sourceKey, 0x00×16) as 6 hex chars |

### Response (`card.contract.dto.SourceKeyUpdateRs`)

| Field | Type | Description |
|-------|------|-------------|
| partSeq | Integer | Updated part sequence number |
| kcv | String | KCV of the newly stored key (6 uppercase hex chars; computed by the service) |
| updatedAt | String | ISO-8601 timestamp of the update |

---

## 3. Service UseCase Flow

**UseCase:** `SourceKeyUpdateUseCase.update(int partSeq, SourceKeyUpdateRq rq)` (card-service)
**UseCase Impl:** `SourceKeyUpdateUseCaseImpl`
**Repository:** `CardKeyPartsJpaRepository` (Spring Data JPA)

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[VALIDATE]` | Validate `partSeq` ∈ {1, 2, 3}; invalid value → throw `INVALID_KEY_FORMAT` (CA00012) | — |
| 2 | `[DOMAIN]` | Parse `sourceKeyHex` → `byte[32]`; non-hex characters or length ≠ 64 → throw `INVALID_KEY_FORMAT` (CA00012) | `KeyParseService.hexToBytes()` |
| 3 | `[DOMAIN]` | Compute KCV: `AES-256-ECB(sourceKeyBytes, byte[16]{0x00})[0..2]` → `kcvComputed` (6 uppercase hex) | `KcvService.compute()` |
| 4 | `[VALIDATE]` | Compare `kcvComputed` with `rq.kcv` (case-insensitive); mismatch → throw `KCV_MISMATCH` (CA00011); zero-fill `sourceKeyBytes` on failure | — |
| 5 | `[DOMAIN]` | Encrypt `sourceKeyBytes` with `MASTER_KEY` (AES-256-GCM) → `encryptedPart` | `CardEncryptionService.encryptKeyPart()` |
| 6 | `[DB WRITE]` | Upsert `card_key_parts`: `part_seq = partSeq`, `encrypted_part = encryptedPart`, `updated_at = now()` | `CardKeyPartsJpaRepository.upsert()` |
| 7 | `[DOMAIN]` | Reload combineKey: read all 3 rows from `card_key_parts`, decrypt each with `MASTER_KEY`, XOR byte-by-byte → update `CombineKeyHolder` | `CombineKeyService.reload()` |
| 8 | `[DOMAIN]` | Zero-fill all intermediate plaintext byte arrays (`sourceKeyBytes`, all decrypted parts, intermediate XOR buffers) | `KeyZeroizer.zeroFill()` |
| 9 | `[RETURN]` | Return `SourceKeyUpdateRs` (`partSeq`, `kcv = kcvComputed`, `updatedAt = now()`) | — |

> **Step 4 failure path:** `sourceKeyBytes` must be zero-filled before throwing `KCV_MISMATCH`.
> The invalid key must not remain in heap memory after a failed injection attempt.

> **Step 7 atomicity:** `CombineKeyHolder` performs an atomic in-memory swap of the `SecretKey`
> reference. Concurrent decrypt operations during reload may briefly use the old combineKey — this
> is accepted behavior in Phase 5 (see Known Gaps in guideline 14).

---

## 4. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card_key_parts` | WRITE | Upsert source key part by `part_seq` |
| `card_key_parts` | READ | Read all 3 rows in Step 7 to reload combineKey |

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
| 400 | CA00012 | INVALID_KEY_FORMAT | `partSeq` not in {1, 2, 3}; or `sourceKeyHex` is not 64 valid hex characters |
| 422 | CA00011 | KCV_MISMATCH | Provided `kcv` does not match the computed KCV of `sourceKeyHex` |
| 500 | INTERNAL_ERROR | Unexpected system error | Unhandled exception (e.g. MASTER_KEY decryption failure, DB error) |
