# PCI DSS Compliance

PCI DSS (Payment Card Industry Data Security Standard) is the international security standard for all systems that store, process, or transmit cardholder data. As a credit card transaction processing system, WillCard must comply with all relevant requirements.

---

## 1. Cardholder Data Definition

**Cardholder Data (CHD):**

| Data Type | Description | Storage Permitted |
|---|---|---|
| PAN (Primary Account Number) | 16-digit credit card number | Permitted, but must be encrypted or masked |
| Cardholder Name | Name as printed on the card | Permitted |
| Expiry Date | Card expiration month and year | Permitted |
| CVV / CVC | 3-digit verification code on the back of the card | **Strictly prohibited** — must be discarded immediately after authorization |
| PIN | Personal Identification Number | **Strictly prohibited** |

---

## 2. PCI DSS Controls by Layer

### 2.1 Network Segmentation

```
Public Internet
  └── BFF (DMZ)
        └── Internal Network / k8s cluster
              ├── Domain Services (not exposed externally)
              ├── Core Services (not exposed externally)
              └── Middleware (dedicated VMs, least-privilege access)
```

- All ports for Domain Services, Ledger, and Orchestrator are reachable only within the k8s cluster internal network
- Middleware VMs (MySQL, Redis, Kafka) only accept connections from k8s worker node IPs
- All external communication enforces TLS 1.2 minimum

---

### 2.2 Card Data Encryption

**NCCC Encryption Flow:**

In Taiwan, cardholder card information is received by NCCC and encrypted using a combineKey before being forwarded. The `encryptedCard` received by WillCard is already ciphertext. WillCard and NCCC negotiate and share the combineKey — encryption responsibility lies with NCCC, and WillCard only performs decryption.

**combineKey Generation and Management:**

Three sourceKeys are stored in three separate columns of the same `card_key_parts` table, each encrypted with **AES-256** (reversible). The AES key used to encrypt the sourceKeys is injected via environment variables or managed by a KMS — it is never stored in the database. A single column leak alone is insufficient to reconstruct the full combineKey.

> **TODO:** Document the combineKey derivation algorithm (XOR / HKDF etc.). See the Key Management chapter.

> **TODO:** Prod environment key management specification (KMS selection and implementation). See the Key Management chapter. Principle: KMS is mandatory in prod; environment variables are only permitted for local / dev environments.

At startup (`ApplicationReadyEvent`), Card Service reads and decrypts the three parts from the database, combines and derives the combineKey, and keeps it resident in the JVM heap. The combineKey plaintext is never serialized, never persisted, and never appears in any network packet.

**Post-decryption Card Data Handling:**

```
NCCC-encrypted encryptedCard
  → Card Service decrypts using in-memory combineKey
  → Decrypted result exists only in JVM heap
  → Cleared from memory immediately after authorization
  → CVV is never written to any persistent storage
```

**Database Storage Rules for Card Data:**

| Field | Storage Method |
|---|---|
| PAN (Card Number) | Stored in masked format only: `****-****-****-1234` |
| Expiry Date | Stored encrypted with AES-256 |
| Cardholder Name | Stored encrypted with AES-256 |
| CVV | **Not stored** — discarded immediately after authorization |

---

### 2.3 Data in Transit

- All external APIs enforce HTTPS with a minimum of TLS 1.2; TLS 1.3 is recommended
- Inter-service communication within the k8s cluster internal network; mTLS is recommended (e.g. via Istio service mesh)
- Kafka and Redis connections must be configured with TLS

---

### 2.4 Log Sanitization

The following data must never appear in any application log:

| Prohibited Data | Handling |
|---|---|
| Full card number (PAN) | Log masked format only: `****-****-****-1234` |
| CVV / CVC | Must not appear in any log |
| combineKey or any encryption key | Must not appear in any log |
| Raw OTP value | Log the `OTP_REQUESTED` event only — never log the OTP value itself |
| Full JWT content | Log the `jti` (JWT ID) only |

---

### 2.5 Kafka Event Payload Rules

Cardholder data field rules for Kafka event payloads:

| Field | Rule |
|---|---|
| Card Number | Transmit `maskedCardNumber` only (`****-****-****-1234`) |
| Amount | Permitted |
| userId | Permitted |
| merchantId | Permitted |
| CVV | **Prohibited in any event payload** |

---

### 2.6 Audit Trail

- All credit card-related operations (OTP request, OTP verification, transaction authorization, card status change) must be written to the Audit Log
- Audit Log is append-only — modification and deletion are prohibited
- Retention period: minimum 12 months online + 12 months offline archive

> **TODO:** Audit Log storage location, table schema, and append-only implementation. See the Audit Trail chapter.

---

### 2.7 Access Control

> **TODO:** See the Access Control chapter. Planned direction:
> - Each service uses a dedicated DB account — no shared credentials; least-privilege principle applies
> - Only card-service has direct access to card-related tables; all other services access card data via API calls
> - DB credentials are managed via K8s Secrets — never hardcoded in source code or config files

Next Chapter: [Spec Guideline](/guideline/7-spec-guideline.md)