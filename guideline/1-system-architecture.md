# System Architecture

This chapter describes the system architecture of WillCard, including the architectural principles that govern service design, a complete microservice inventory with each service's layer and responsibility, and a high-level topology diagram illustrating how requests flow from clients through the BFF layer into domain services.

## Architecture Principles

### 1. two types of microservice

- **BFF (Backend-for-Frontend)**: Single external entry point — JWT validation, idempotency control, request aggregation, header forwarding.
- **Domain Services**: Each service owns its bounded context; no direct cross-domain service calls.

### 2. Distributed Transactions

- **Saga Orchestration**: Cross-service transactions coordinated by the Transaction Orchestrator; compensating transactions ensure consistency.
- **Event-Driven**: Services decouple via Kafka event bus; each consumer group operates independently.
- **Immutable Ledger**: Journal entries are INSERT-only — no UPDATE or DELETE ever

## Microservice Inventory

| # | Service | Layer | Responsibility |
| --- | --- | --- | --- |
| 1 | BFF | Presentation | Request aggregation, JWT validation, rate limiting, idempotency, header forwarding |
| 2 | Card Service | Domain | Virtual card management, OTP generation/validation, card state machine, combineKey |
| 3 | Wallet Service | Domain | TWD points balance, top-up commands |
| 4 | Points Service | Domain | Points account, reward calculation, redemption, expiry management |
| 5 | FX Service | Utility | Exchange rate query and locking (stateless calculator — no account ownership) |
| 6 | Transaction Orchestrator | Core | Saga coordination, compensating transactions, idempotency control |
| 7 | Ledger Service | Core | Double-entry ledger, journal entries, immutable |
| 8 | Reconciliation Service | Core | T+0 real-time reconciliation, T+1 batch settlement, discrepancy reports |
| 9 | Notification Service | Infra | OTP SMS, transaction push notifications, email receipts |
| 10 | Auth Service | Domain | Login, Oauth, user authentication and authrozation |

> **FX Service positioning:**
> FX Service is a stateless exchange rate utility. It holds no accounts and participates in no Saga state management. Its sole responsibility is rate query, locking, and conversion calculation.

```mermaid
graph TB
    subgraph Clients["Clients"]
        MA[Mobile App]
        WA[Web App]
        POS[Merchant POS]
    end

    subgraph BFF_Layer["BFF Layer (Spring Security + JWT)"]
        BFF[BFF<br/>JWT Validation · Rate Limit · Idempotency<br/>Header Forwarding]
    end

    subgraph Domain_Layer["Domain Services (trust BFF headers — no Security filter)"]
        CARD[Card Service<br/>vCard · OTP · combineKey]
        WALLET[Wallet Service<br/>Balance · Top-up]
        POINTS[Points Service<br/>Earn · Redeem]
    end

    subgraph Utility_Layer["Utility Service"]
        FX[FX Service<br/>Rate Query · Lock · Calculate<br/>Stateless — no account ownership]
    end

    subgraph Core_Layer["Core Engine"]
        ORCH[Transaction Orchestrator<br/>Saga · Compensate · Idempotency]
        LEDGER[Ledger Service<br/>Double-entry · Immutable]
        RECON[Reconciliation<br/>T+0 · T+1 Batch]
    end

    subgraph Event_Bus["Kafka Event Bus"]
        KAFKA[(Kafka)]
    end

    subgraph Notification_Layer["Notification"]
        NOTIFY[Notification Service]
        MQ[(RabbitMQ<br/>OTP SMS · Push · Email)]
    end

    MA --> BFF
    WA --> BFF
    POS --> BFF

    BFF -->|X-User-Id, X-User-Role header| CARD
    BFF -->|X-User-Id, X-User-Role header| WALLET
    BFF -->|X-User-Id, X-User-Role header| POINTS
    BFF -->|X-User-Id, X-User-Role header| FX
    BFF -->|X-User-Id, X-User-Role header| ORCH

    ORCH --> CARD
    ORCH --> WALLET
    ORCH --> FX
    ORCH --> LEDGER
    ORCH --> KAFKA

    KAFKA --> POINTS
    KAFKA --> LEDGER
    KAFKA --> RECON
    KAFKA --> NOTIFY

    NOTIFY --> MQ
    CARD --> MQ
```

Next Chapter: [Infrastructure Configuration](/guideline/2-infrastructure.md)
