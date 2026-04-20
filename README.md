# Payment gateway

> Self-hosted payment gateway built as a learning project, architecturally 
> modeled after Stripe, Adyen, and Tinkoff Acquiring. Focuses on distributed 
> systems patterns, payment domain modeling, and production-grade reliability.

## Overview

Payment gateway exposing a REST API to merchants for card payments, 
tokenization, recurring charges, refunds, and fiscal receipts. 
Communicates with merchants exclusively via signed webhooks.

**Public API:** [sergeevoa.github.io/payment-gateway](https://sergeevoa.github.io/payment-gateway/)

## Architecture

The system is described through three complementary views. Each view answers 
one specific question about the architecture.

### Synchronous communication (REST + gRPC)

Request-response paths triggered by merchant API calls.

```mermaid
flowchart TB
    Merchant([Merchant Backend])
    Bank([Acquiring Bank])
    OFD([ОФД])

    subgraph PGW[" Payment Gateway "]
        direction TB
        GW[api-gateway]

        subgraph Core[" Core Services "]
            direction TB
            PS[payment-service]
            CARD[card-service]
            CUS[customer-service]
            MS[merchant-service]
            REC[receipt-service]
            NS[notification-service]
        end
    end

    Merchant -->|REST + HMAC| GW

    GW -->|gRPC| PS
    GW -->|gRPC| CARD
    GW -->|gRPC| CUS
    GW -->|gRPC| MS
    GW -->|gRPC| REC

    PS -->|gRPC| MS
    PS -->|gRPC| CUS
    PS -->|gRPC| CARD
    CARD -->|gRPC| CUS
    CARD -->|gRPC| MS
    REC -->|gRPC| MS

    PS -->|REST| Bank
    REC -->|REST| OFD
    NS -->|REST + HMAC| Merchant

    classDef external fill:#e8e8e8,stroke:#555,stroke-width:2px,color:#000
    classDef service fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#000
    classDef gateway fill:#fef3c7,stroke:#b45309,stroke-width:2px,color:#000

    class Merchant,Bank,OFD external
    class PS,CARD,CUS,MS,REC,NS service
    class GW gateway

    style PGW fill:none,stroke:#666,stroke-width:1.5px,stroke-dasharray: 5 5
    style Core fill:none,stroke:#999,stroke-width:1px,stroke-dasharray: 3 3
```

- External API is REST with HMAC signatures — required for backward 
  compatibility and merchant SDK ecosystem
- **Synchronous** service-to-service calls use gRPC
- External integrations (acquiring bank, ОФД) use REST per their public contracts

### Event-driven communication (Kafka)

Domain events emitted by core services are consumed asynchronously to trigger 
webhooks, fiscal processing, and downstream workflows.

```mermaid
flowchart LR
    PS[payment-service]
    CARD[card-service]
    REC[receipt-service]
    NS[notification-service]
    Merchant([Merchant Backend])

    T1{{"payment.events"}}
    T2{{"card.events"}}
    T3{{"receipt.events"}}

    PS -->|outbox| T1
    CARD -->|outbox| T2
    REC -->|outbox| T3

    T1 --> NS
    T1 --> REC
    T2 --> NS
    T3 --> NS

    NS -->|REST + HMAC webhooks| Merchant

    classDef service fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#000
    classDef broker fill:#fce7f3,stroke:#9f1239,stroke-width:2px,color:#000
    classDef external fill:#e8e8e8,stroke:#555,stroke-width:2px,color:#000

    class PS,CARD,REC,NS service
    class T1,T2,T3 broker
    class Merchant external
```

### Data & infrastructure

Each service owns its PostgreSQL database — no shared schema, no cross-service 
joins. Shared infrastructure is limited to Redis and MinIO.

```mermaid
flowchart LR
    subgraph Services[" Services "]
        direction TB
        MS[merchant-service]
        PS[payment-service]
        CUS[customer-service]
        CARD[card-service]
        REC[receipt-service]
        NS[notification-service]
        GW[api-gateway]
    end

    MS_DB[(PG: merchant)]
    PS_DB[(PG: payment)]
    CUS_DB[(PG: customer)]
    CARD_DB[(PG: card)]
    REC_DB[(PG: receipt)]
    NS_DB[(PG: notification)]

    RD[(Redis)]
    MINIO[(MinIO)]

    MS --- MS_DB
    PS --- PS_DB
    CUS --- CUS_DB
    CARD --- CARD_DB
    REC --- REC_DB
    NS --- NS_DB

    GW -.-> RD
    PS -.-> RD
    CARD -.-> RD
    MS -.-> RD

    REC --- MINIO
    NS --- MINIO

    classDef service fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#000
    classDef gateway fill:#fef3c7,stroke:#b45309,stroke-width:2px,color:#000
    classDef datastore fill:#dcfce7,stroke:#166534,stroke-width:2px,color:#000

    class MS,PS,CUS,CARD,REC,NS service
    class GW gateway
    class MS_DB,PS_DB,CUS_DB,CARD_DB,REC_DB,NS_DB,RD,MINIO datastore

    style Services fill:none,stroke:#999,stroke-width:1px,stroke-dasharray: 3 3
```

- **PostgreSQL per service** — strict database-per-service boundary
- **Redis** — idempotency keys, rate limiting, config caching, distributed locks
- **MinIO** — object storage for fiscal receipt PDFs and webhook delivery logs

### Services

| Service | Responsibility
|---|---|
| `api-gateway` | TLS termination, HMAC validation, rate limiting, routing
| `merchant-service` | Merchants, terminals, API keys, webhook settings
| `payment-service` | Payment state machine, `Init`/`FinishAuthorize`/`Confirm`/`Cancel`/`Charge`
| `customer-service` | End-customer records scoped per merchant 
| `card-service` | Card tokenization, `RebillId`, card-binding state machine
| `receipt-service` | Fiscal receipts, OFD integration
| `notification-service` | Webhook delivery with retries and signing

## Tech Stack

**Language & Framework:** Java 21, Spring Boot 3, Spring Cloud Gateway, Spring Data JPA, Spring Security  
**Communication:** REST, gRPC, Kafka  
**Storage:** PostgreSQL, Redis, MinIO  
**Build & Test:** Gradle, JUnit 5, Mockito, Testcontainers  
**Infrastructure:** Docker  
**API Documentation:** OpenAPI 3.0 (Swagger)

## Authorization

All API endpoints require a bearer token in the `Authorization` header.
```http
Authorization: Bearer <your_api_key>
```
Api keys are managed through the Merchant Portal - a separate application for merchant registration and credential management.
🚧 Merchant Portal is currently under development.

## Project Status

🚧 Work in progress.
