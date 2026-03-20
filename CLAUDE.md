# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language Preference

Always respond in **Traditional Chinese (繁體中文)** by default when interacting with the user.

## What This Repository Is

This is the **documentation repository** for WillCard, an electronic payment wallet system. It contains architecture guidelines, development conventions, and API specifications — not application source code. The actual microservices code lives in a separate repository.

## Repository Structure

```
guideline/          # Architecture & development guidelines (numbered, bilingual)
spec/               # API specifications organized by service
  mobile-bff/       # BFF-layer specs (external-facing)
  auth-service/     # Auth service specs (internal)
  common/           # Shared spec templates
i18n/zh-TW/         # Traditional Chinese translations
```

## Bilingual Convention

Every document has two versions:
- English: `{name}.md`
- Traditional Chinese: `{name}-zh-tw.md`

When creating or editing documentation, **always maintain both language versions**.

## Guideline Documents (in reading order)

1. **System Architecture** — Microservice inventory (10 services), BFF vs Domain Service distinction, Saga orchestration, Kafka event-driven design
2. **Infrastructure** — 6-VM topology on VMware (k8s master, 2 workers, middleware, cache+DB, observability)
3. **Project Architecture & Structure** — Clean Architecture + DDD, Maven multi-module layout, contract module rules, object naming conventions (Rq/Rs/Entity/Model/Vo/Dto), package structure, dependency boundaries
4. **Data Management** — MySQL 8 (utf8mb4), Snowflake ID generation (custom epoch 2025-07-22), Redis-based pod ID leasing, Flyway migration naming, no DB-level foreign keys
5. **Exception Handling** — AbstractServiceException hierarchy, error code format `{modelType}{errorCode}` (e.g. AU00001), BffExceptionHandler vs BizExceptionHandler split
6. **PCI DSS** — Card data encryption rules, AES-256 for CHD, CVV never stored, log sanitization requirements, audit trail retention
7. **Spec Guideline** — The canonical reference for writing API specs (see below)

## Writing API Specifications

Specs follow strict conventions defined in `guideline/7-spec-guideline.md`. Key rules:

**Two spec types with different structures:**

| BFF Spec | Biz (Service) Spec |
|----------|-------------------|
| Overview | Overview |
| Operation Log Level (L1/L2/L3) | — |
| API Definition (Nginx strip rule) | API Definition (context-path rule) |
| BFF UseCase Flow | Service UseCase Flow |
| — | Database (tables + Redis keys) |
| Error Codes (external) | Error Codes (internal) |

**URL path rules:**
- BFF: Client sends `/api/v{n}/{resource}/{action}` → Nginx strips prefix → BFF controller receives `/{resource}/{action}`
- Services: `server.servlet.context-path: /{service-name}` + controller mapping = `/{service-name}/{action}`

**UseCase flow action tags:**
- BFF: `[VALIDATE]`, `[REDIS READ]`, `[REDIS WRITE]`, `[FEIGN]`, `[HEADER]`, `[KAFKA]`, `[RETURN]`
- Service: `[DB READ]`, `[DB WRITE]`, `[REDIS WRITE]`, `[DOMAIN]`, `[RETURN]`

**Every `[FEIGN]` step must link to the corresponding biz spec.**

**Operation Log Levels (BFF only):**
- L1: Financial, auth, security operations → Kafka → Audit DB
- L2: Non-financial data mutations → Kafka → Audit DB
- L3: Read queries → EFK only

## Tech Stack Reference

- **Java 17 + Spring Boot 3.4.5**, Microservices + BFF + Saga
- **Kubernetes** (kubeadm) on Ubuntu 22.04 VMs, physical host Windows Server 2019
- **MySQL 8** + Spring Data JPA, **Redis 7**, **Kafka** (events) + **RabbitMQ** (notifications)
- **Observability:** Jaeger + Prometheus + Grafana + EFK
- **Security:** Spring Security (BFF only), JWT, AES-256 card encryption, TLS 1.2+

## Key Architecture Rules

- **Contract modules** are pure POJOs — only lombok, jackson-databind, spring-boot-starter-validation allowed. No common module imports.
- **Dependency boundaries:** `*-service` imports `common-biz` only; `bff` imports `common-bff` only; contracts import nothing.
- **No DB-level foreign keys** — referential integrity at application layer, cross-service consistency via Saga.
- **Snowflake IDs** generated in application layer (no DB auto-increment). 64-bit: sign + 41-bit timestamp + 5-bit DC + 5-bit pod + 12-bit sequence.
- **CVV/PIN must never be stored or logged.** PAN stored only in masked format. Card encryption keys from env/KMS, never in DB.
