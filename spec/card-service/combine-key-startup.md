# CombineKey Startup Loader — card-service Spec

> **Service:** card-service
> **Component Type:** Application Lifecycle — `ApplicationReadyEvent` handler
> **Last Updated:** 2026-03

---

## 1. Overview

On application startup, card-service reads the three encrypted source key parts from `card_key_parts`,
decrypts each one with the master key, XORs them together to produce the `combineKey`, and stores
the result as a `SecretKey` in the singleton `CombineKeyHolder` bean.

The `combineKey` lives exclusively in JVM heap memory and is used by Phase 7
(`POST /card-service/auth/authorize`) to decrypt NCCC-encrypted card data.

On JVM shutdown (`@PreDestroy`), the `CombineKeyHolder` zero-fills the key's backing byte array
to prevent the key from remaining in a memory dump.

> **No HTTP endpoint.** This spec describes a lifecycle component, not an API.
> See [guideline/14-3ds-combine-key.md](../../guideline/14-3ds-combine-key.md) for the full
> key architecture context.

---

## 2. Component Definition

### CombineKeyHolder

A singleton Spring bean that holds the assembled `combineKey` for the lifetime of the application.

| Attribute | Value |
|-----------|-------|
| Bean scope | Singleton |
| Class | `card.service.security.CombineKeyHolder` |
| Interface | Implements `DisposableBean` (for `@PreDestroy` zero-fill) |
| Thread safety | `combineKey` field is `volatile`; atomic reference swap used during hot-reload |

```
CombineKeyHolder
  ─ volatile SecretKey combineKey          // javax.crypto.spec.SecretKeySpec, algorithm "AES"
  ─ void set(SecretKey key)                // called by startup loader and by SourceKeyUpdateUseCase
  ─ SecretKey get()                        // called by Phase 7 decrypt logic; throws if null
  ─ void destroy()                         // @PreDestroy — zero-fills backing byte array
```

### CombineKeyStartupLoader

An `ApplicationListener<ApplicationReadyEvent>` that executes the load sequence after the Spring
context and all datasource connections are fully ready.

| Attribute | Value |
|-----------|-------|
| Class | `card.service.security.CombineKeyStartupLoader` |
| Trigger | `ApplicationReadyEvent` |
| Dependencies | `CardKeyPartsJpaRepository`, `CardEncryptionService`, `KcvService`, `CombineKeyHolder` |

---

## 3. Startup UseCase Flow

**Component:** `CombineKeyStartupLoader.onApplicationEvent(ApplicationReadyEvent)`

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | `SELECT * FROM card_key_parts ORDER BY part_seq`; collect all rows | `CardKeyPartsJpaRepository.findAllOrderByPartSeq()` |
| 2 | `[VALIDATE]` | If row count < 3: log `ERROR "card_key_parts has {n} rows; expected 3 — combineKey unavailable"` → `CombineKeyHolder.set(null)` → **return** (service starts degraded; see Section 5) | — |
| 3 | `[DOMAIN]` | For each of the 3 rows: decrypt `encrypted_part` with `MASTER_KEY` (AES-256-GCM) → `sourceKeyBytes[n]` (byte[32]); on decryption failure for any part → log `ERROR "failed to decrypt part {partSeq}"` → zero-fill any already-decrypted buffers → `CombineKeyHolder.set(null)` → **return** | `CardEncryptionService.decryptKeyPart()` |
| 4 | `[DOMAIN]` | Compose: `combineKeyBytes[i] = sourceKey1[i] XOR sourceKey2[i] XOR sourceKey3[i]` (loop over 32 bytes) | `CombineKeyService.xorCompose()` |
| 5 | `[DOMAIN]` | Wrap: `SecretKey = new SecretKeySpec(combineKeyBytes, "AES")` | — |
| 6 | `[DOMAIN]` | Zero-fill all intermediate byte arrays: `sourceKey1Bytes`, `sourceKey2Bytes`, `sourceKey3Bytes`, `combineKeyBytes` (the `SecretKeySpec` retains its own internal copy) | `KeyZeroizer.zeroFill()` |
| 7 | `[DOMAIN]` | `CombineKeyHolder.set(secretKey)` — atomic volatile write | — |
| 8 | `[DOMAIN]` | Log `INFO "combineKey loaded successfully (parts: 1, 2, 3)"` — **no key bytes in log** | — |

---

## 4. Shutdown Behavior (`@PreDestroy`)

**Component:** `CombineKeyHolder.destroy()` (via `DisposableBean.destroy()`)

| Step | Type | Description |
|------|------|-------------|
| 1 | `[DOMAIN]` | If `combineKey == null`: log `INFO "combineKey was null at shutdown — nothing to clear"` → return |
| 2 | `[DOMAIN]` | Obtain backing byte array from `SecretKeySpec` via `getEncoded()` |
| 3 | `[DOMAIN]` | Zero-fill the byte array: `Arrays.fill(keyBytes, (byte) 0)` |
| 4 | `[DOMAIN]` | Set `combineKey = null` |
| 5 | `[DOMAIN]` | Log `INFO "combineKey zeroed on shutdown"` |

> **Note:** `javax.crypto.spec.SecretKeySpec.getEncoded()` returns a copy of the internal byte array.
> Zero-filling the copy does not zero-fill the internal array. To reliably zero the JVM's internal
> copy, use `sun.security.util.Password.destroy(key)` or an equivalent `Destroyable.destroy()` call
> if available in the target JRE. Document this limitation in code comments.

---

## 5. Degraded-Start Behavior

If startup loading fails (Step 2 or Step 3 above), `CombineKeyHolder.combineKey` remains `null`.
The service continues to start normally — other endpoints (card CRUD, KCV verify, source key update)
remain available. Only authorization-path decryption is affected.

| Endpoint | Behavior when combineKey is null |
|----------|----------------------------------|
| `POST /card-service/auth/authorize` | `CombineKeyHolder.get()` throws `CombineKeyUnavailableException` → mapped to HTTP 503 with error code `CA00013 COMBINE_KEY_UNAVAILABLE` |
| All other card-service endpoints | Unaffected |

> **Operational note:** A degraded start should trigger an alerting rule on the ERROR log line
> `"combineKey unavailable"`. Operators should inject or verify source keys using
> `PUT /internal/keys/source-key/{partSeq}` and then restart the service (or call the reload
> path triggered by a successful source key update).

---

## 6. Hot-Reload (triggered by `SourceKeyUpdateUseCase`)

When an operator successfully updates a source key via the Source Key Update API, that UseCase
calls `CombineKeyService.reload()` at Step 7, which executes the same compose logic as Steps 3–7
above (reading fresh DB state) and calls `CombineKeyHolder.set(newSecretKey)`.

This replaces the in-memory `combineKey` without restarting the service.

| Step | Type | Description | Domain Service |
|------|------|-------------|----------------|
| 1 | `[DB READ]` | Read all 3 rows from `card_key_parts` | `CardKeyPartsJpaRepository.findAllOrderByPartSeq()` |
| 2 | `[DOMAIN]` | Decrypt each part with `MASTER_KEY` → 3 × `byte[32]` | `CardEncryptionService.decryptKeyPart()` |
| 3 | `[DOMAIN]` | XOR compose → `combineKeyBytes` | `CombineKeyService.xorCompose()` |
| 4 | `[DOMAIN]` | `CombineKeyHolder.set(new SecretKeySpec(combineKeyBytes, "AES"))` | — |
| 5 | `[DOMAIN]` | Zero-fill all intermediate byte arrays | `KeyZeroizer.zeroFill()` |

---

## 7. Database

### MySQL

| Table | Operation | Description |
|-------|-----------|-------------|
| `card_key_parts` | READ | Read all 3 rows on startup and on each hot-reload |

### Redis

> This component performs no Redis operations.

### Table Schema

#### card_key_parts

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| part_seq | INT | NO | PK: 1, 2, or 3 |
| encrypted_part | VARCHAR(512) | NO | AES-256-GCM encrypted 256-bit source key; master key from env/KMS |
| updated_at | DATETIME(3) | NO | Last injection or rotation timestamp |

---

## 8. Error Codes

> No HTTP error codes — this component does not expose an HTTP endpoint.
> Startup failures are reported via ERROR log lines and surface at runtime only through
> `POST /card-service/auth/authorize` returning HTTP 503.

| Condition | Behavior | Log Level |
|-----------|----------|-----------|
| `card_key_parts` has fewer than 3 rows | `combineKey = null`; service starts degraded | ERROR |
| Decryption failure for any part | `combineKey = null`; service starts degraded | ERROR |
| `CombineKeyHolder.get()` called when `null` | Throws `CombineKeyUnavailableException` → CA00013 | — |

| HTTP Status | Error Code | Description | Trigger Condition |
|-------------|------------|-------------|-------------------|
| 503 | CA00013 | COMBINE_KEY_UNAVAILABLE | `combineKey` is null at the time `auth/authorize` is called |
