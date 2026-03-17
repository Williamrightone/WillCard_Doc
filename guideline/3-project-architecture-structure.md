# Project Architecture Guide

This document defines the architectural standards and conventions for the WillCard Backend project.

The system is built on Clean Architecture and Domain-Driven Design (DDD) principles, with each service structured to enforce a strict separation of concerns and a unidirectional dependency flow from infrastructure toward the domain core.

This guide is organized top-down — starting from the overall project layout and module structure, moving through package organization and inter-service dependencies, and finally covering individual object types, naming conventions, and design constraints.

## 1. Project Structure

### 1.1 Maven Multi-Project Overview

```cmd
willcard-parent
├── bff
├── card-service
├── card-contract
├── wallet-service
├── wallet-contract
├── points-service
├── points-contract
├── fx-service
├── fx-contract
├── transaction-service
├── transaction-contract
├── ...
└── pom.xml
```

### 1.2 Contract Module Principles

- Pure POJO — no framework dependencies (no Spring, JPA, etc.)
- Only allowed dependencies: `lombok`, `jackson-databind`, `spring-boot-starter-validation`
- Contains only Rq / Rs and Event schema definitions — no business logic
- Version management must be strict; breaking changes affect all consumers
- All modules that call a service must import the corresponding contract
- Must not import any internal common modules

### 1.3 Common Modules

Three shared library modules. All are pure libraries — they do not contain a Spring Boot entry point and must not run as standalone applications.

#### Dependency Overview

```cmd
      common-shared
    ↑               ↑
common-biz       common-bff
    ↑               ↑
*-service           bff
```

#### common-shared

The foundational infrastructure layer shared across all service modules. Provides universal utilities and base Spring configuration for the entire project.
 
| Content | Description |
|---|---|
| Exception base classes | `AbstractServiceException`, `BaseErrorType`, `IErrorLevel`, `ModelType` |
| General utilities | Date, string, number, and other pure utility classes |
| Spring base configuration | Web, Validation, Redis, AMQP, AOP starters |
| Observability | Actuator, Micrometer Tracing, OpenTelemetry, Prometheus |
| Logging | Logstash Logback Encoder |
 
#### common-biz
 
Shared business infrastructure for the service layer. Only `*-service` modules may import this — `bff` must not.
 
| Content | Description |
|---|---|
| JPA base configuration | `BaseEntity`, audit fields (`createdAt`, `updatedAt`) |
| Business utilities | Financial calculations, currency conversion, and other domain-aware utilities |
| Shared exception handler | `BizExceptionHandler` (`@ControllerAdvice`) |
 
#### common-bff
 
Shared infrastructure for the BFF layer. Only `bff` may import this — `*-service` modules must not.
 
| Content | Description |
|---|---|
| Response wrapper | Unified API response structure (`ApiResponse`) |
| Pagination | Shared pagination Rq / Rs definitions |
| Shared exception handler | `BffExceptionHandler` (`@ControllerAdvice`) |
 
#### Boundary Rules
 
- `*-contract` must not import any internal common module
- `*-service` may only import `common-biz` — must not import `common-bff`
- `bff` may only import `common-bff` — must not import `common-biz`
 
---
 
## 2. Object Naming Conventions
 
### 2.1 Naming Reference Table
 
| Type | Suffix | Example | Layer |
|---|---|---|---|
| API Request | `Rq` | `LoginRq` | `application.api.dto` / contract |
| API Response | `Rs` | `LoginRs` | `application.api.dto` / contract |
| JPA Entity | `Entity` | `UserEntity` | `adapter.persistence` |
| Domain Entity | `Model` | `UserCardModel` | `domain.model` |
| Value Object | `Vo` | `MoneyVo` | `domain.vo` |
| DTO | `Dto` | `AuthByPasswdDto` | `domain.dto` |
 
### 2.2 Type Definitions
 
**Rq / Rs**
- API request and response objects
- BFF Rq/Rs are owned exclusively by the BFF module, defined in `application.api.dto`
- All other services' Rq/Rs are defined in their corresponding contract module
 
**JPA Entity (suffix `Entity`)**
- JPA object that maps directly to a database table
- Used only within `adapter.persistence`
- Must not be returned as Rs to the frontend or other services
 
**Domain Entity (suffix `Model`)**
- Domain object with a unique ID and lifecycle
- Defined in `domain.model`
- Examples: `UserCardModel`, `WalletModel`
 
**Value Object (suffix `Vo`)**
- No ID, immutable, equality defined by value
- Defined in `domain.vo`
- Examples: `MoneyVo` (amount + currency), `CardNumberVo`
 
**DTO (suffix `Dto`)**
- Internal product of a Service method, typically named after the method
- Defined in `domain.dto`
- Assembled by the UseCase from multiple Dtos into a Rs
- Example: `AuthByPasswdDto`
 
### 2.3 Object Flow Example
 
```
LoginRq
  → Controller
  → LoginUseCase
  → AuthService.authByPasswd()  → AuthByPasswdDto
  → TokenService.generate()     → TokenDto
  → assembled into LoginRs
  ← returned by Controller
```
 
---
 
## 3. Package Structure
 
### 3.1 BFF Package Structure
 
```
bff
├── application
│     ├── api
│     │     └── dto                  # Rq / Rs (BFF-only)
│     ├── controller                 # Controller
│     ├── usecase                    # Use Case interfaces
│     ├── usecase.impl               # Use Case implementations
│     └── usecase.assembler          # Assemblers for large Use Cases
├── domain
│     ├── model                      # Domain Entities (UserCardModel...)
│     ├── vo                         # Value Objects (MoneyVo...)
│     ├── dto                        # Service-internal products (AuthByPasswdDto...)
│     ├── port                       # Abstract interfaces for external dependencies
│     ├── service                    # Domain Service interfaces
│     └── service.impl               # Domain Service implementations (selective)
├── adapter
│     ├── persistence                # JPA RepositoryImpl, Entity
│     ├── client                     # FeignClient implementing Port
│     └── messaging                  # Kafka / RabbitMQ Publisher implementing Port
└── bootstrap                        # Spring Boot entry point, ControllerAdvice
```
 
### 3.2 Domain Service Package Structure (fx-service example)
 
```
fx-service
├── domain
│     ├── model
│     ├── vo
│     ├── dto
│     ├── port
│     ├── service
│     └── service.impl
├── adapter
│     ├── persistence
│     ├── client
│     └── messaging
└── bootstrap
 
fx-contract                          # Standalone Maven Module — pure POJO
└── dto
      ├── FxRateRq
      ├── FxRateRs
      ├── FxLockRateRq
      └── FxLockRateRs
```
 
---
 
## 4. Design Principles & Constraints
 
### 4.0 Core Design Prerequisites
 
All rules in this chapter must be followed by every component described below.
 
#### 4.0.1 Controller and Use Case Naming
 
This project follows a Use Case–oriented API design. Every API corresponds to one explicit Use Case.
 
Naming rules:
- API endpoint, Controller method, and Use Case method must be semantically consistent
- Names are verb-first, representing a clear user or system action
- One API maps to exactly one Use Case — combining multiple behaviors is not allowed
 
Example:
 
```
POST /auth/guest/start
  → AuthController.start()
  → GenerateGuestTokenUseCase.start()
```
 
Purpose of this rule:
- Establish a one-to-one relationship between API and business behavior
- Reduce cognitive load when navigating and maintaining the system
- Ensure the Controller remains a Thin Adapter
 
#### 4.0.2 Request / Response (Rq / Rs) Package Strategy
 
Rq / Rs are data contracts for external or cross-module communication. Their location depends on the module's role.
 
| Module Type | Rq / Rs Location | Notes |
|---|---|---|
| BFF / Facade | `application.api.dto` | Owned exclusively by the facade module |
| Domain Service / Core | `{service}-contract` | Shared data contract for cross-module / cross-service use |
 
Forbidden: Facade-specific Rq/Rs must not be imported by other services.
 
#### 4.0.3 Selective Implementation Strategy
 
Not every Service, Repository, or Adapter is required to have a corresponding Impl class.
 
Decision criteria:
 
| Scenario | Interface Required? |
|---|---|
| Same interface has multiple implementations (e.g. Mock vs Real) | Yes |
| Cross-module dependency boundary | Yes |
| Single implementation with no expected replacement | No — use concrete class directly |
 
**Exception: Use Cases always require an interface**, regardless of the number of implementations.
 
---
 
### 4.1 Controller
 
The Controller is the external entry point of the system. It receives API requests and delegates to the corresponding Use Case.
 
**Responsibilities**
- Receive and parse the Request (Rq)
- Perform basic parameter validation (format, required fields)
- Invoke the corresponding Use Case
- Return the Response (Rs)
 
**Constraints**
- Must not contain any business logic
- Must not directly call Service, Repository, or Adapter
- Must not handle transactions or flow decisions
 
The Controller must remain a Thin Adapter.
 
---
 
### 4.2 Use Case
 
A Use Case represents one explicit user or system action. It is the core of the application layer.
 
**Responsibilities**
- Define the execution interface for a single business behavior
- Coordinate flow sequence and logic branching
- Invoke necessary Services or Adapters
 
**Rules**
- Must be an interface
- Named with a verb as the primary word
- One Use Case corresponds to one API behavior
 
---
 
### 4.3 Use Case Implementation
 
UseCaseImpl is the concrete implementation of a Use Case.
 
**Responsibilities**
- Implement application flow and logic orchestration
- Invoke Services, Repositories, or Adapters
- Assemble multiple Dtos into a Rs and return it
 
**Constraints**
- Must not contain infrastructure implementation details
- Must not directly operate Framework APIs
- Must not handle format conversions unrelated to business logic (e.g. JSON serialization)
- Object assembly that is part of business logic (Dto → Rs) is the responsibility of UseCaseImpl and is permitted
 
UseCaseImpl is the sole owner of application flow.
 
---
 
### 4.4 Assembler
 
The Assembler is used to decompose an oversized UseCase. It coordinates multiple Service calls and assembles the results into a Rs.
 
**Location:** `application.usecase.assembler`
 
**Responsibilities**
- Coordinate the invocation order of multiple Domain Services
- Merge multiple Dtos and assemble them into a Rs
 
**When to use**
- Consider extracting an Assembler when a UseCase coordinates more than three Services
 
**Constraints**
- Must not contain business decision logic — orchestration and assembly only
- Must not directly access Repository or Adapter
 
---
 
### 4.5 Service
 
The Service carries reusable business logic and sub-processes.
 
**Responsibilities**
- Encapsulate business rules
- Provide behaviors shared across multiple Use Cases
 
**Rules**
- Must not depend on Controller or Use Case
- May depend on Repository or Adapter Port interfaces
- Must not coordinate cross-Use Case flows
 
---
 
### 4.6 Repository
 
The Repository defines the abstract interface for data access.
 
**Responsibilities**
- Define data access methods
- Hide data source and implementation details
- Provide domain-centric access semantics
 
**Rules**
- Must be an interface
- Returns Domain Models — must not expose JPA Entities
- Must not expose ORM / SQL details
 
---
 
### 4.7 DataGripHelper
 
The DataGripHelper handles scenarios that require combining data from multiple Repositories. It encapsulates cross-table query and assembly logic.
 
**Location:** `adapter.persistence`
 
**Responsibilities**
- Inject multiple JPA Repositories
- Combine query results from multiple tables
- Convert results into Domain Models and return them to the Domain Service
 
**Rules**
- Implements a corresponding Port interface defined in `domain.port`
- Domain Service depends only on the Port interface — it must not be aware of DataGripHelper directly
- Return type must be a Domain Model — must not return JPA Entity or Vo
 
**Example**
 
```
domain.port
  └── CardWalletPort (interface) → returns CardWalletModel
 
adapter.persistence
  └── CardWalletDataGripHelper implements CardWalletPort
        ├── injects CardJpaRepository
        ├── injects WalletJpaRepository
        ├── assembles data
        └── converts and returns CardWalletModel
```
 
---
 
### 4.8 Port
 
A Port defines the abstract interaction interface with external systems or infrastructure.
 
**Responsibilities**
- Abstract HTTP, RPC, MQ, Cache, and other external dependencies
- Isolate the impact of third-party systems on the core business layer
 
**Rules**
- Must be an interface, defined in `domain.port`
- Implemented by Adapters
 
---
 
### 4.9 Repository Implementation
 
RepositoryImpl is the concrete implementation of a Repository.
 
**Responsibilities**
- Implement ORM, SQL, and Transaction technical details
- Convert JPA Entities into Domain Models before returning
 
**Constraints**
- Must not contain business logic
- Must not be directly depended on by Controller or Use Case
 
---
 
### 4.10 Adapter
 
The Adapter is the concrete implementation of a Port.
 
**Responsibilities**
- Implement actual interaction with external systems
- Handle communication, serialization, and error conversion
 
---
 
### 4.11 Data Models
 
#### Naming Conventions
 
| Type | Suffix | Example | Location |
|---|---|---|---|
| API Request | `Rq` | `LoginRq` | `application.api.dto` / contract |
| API Response | `Rs` | `LoginRs` | `application.api.dto` / contract |
| JPA Entity | `Entity` | `UserEntity` | `adapter.persistence` |
| Domain Entity | `Model` | `UserCardModel` | `domain.model` |
| Value Object | `Vo` | `MoneyVo` | `domain.vo` |
| DTO | `Dto` | `AuthByPasswdDto` | `domain.dto` |
 
#### Forbidden
 
- `Entity` must not be returned as Rs to the frontend or other services
- `Entity` must not appear outside of `adapter.persistence`
- `Model` must not map directly to a table structure (no JPA annotations)

---
 
## 5. Dependency Direction Overview
 
### 5.1 Clean Architecture Dependency Rule
 
```
xxx-service
  → application (Controller / UseCase / Assembler)
  → domain (Service / Port interface / Model / Vo / Dto)
  ← adapter (implements Port / Repository)
```
 
- Dependencies always point inward
- The `domain` layer has no dependency on any other module
- `adapter` depends on `domain` (to implement Port)
 
### 5.2 BFF Full Dependency Flow
 
```
Rq → Controller → UseCase → Assembler (optional)
                           → Domain Service A → Dto-A ┐
                           → Domain Service B → Dto-B ┤→ assembled into Rs
                  UseCase ←────────────────────────────┘
Controller ← Rs
```
 
### 5.3 Port / Adapter Dependency Inversion
 
```
Domain Service → Port (interface)
                      ↑
          Adapter implements Port (Dependency Inversion)
          ├── FeignClient      (HTTP call)
          ├── Kafka Publisher  (MQ)
          ├── RepositoryImpl   (JPA)
          └── DataGripHelper   (cross-table assembly)
```

Next Chapter: [Data Management](/guideline/4-data-mgmt.md)
