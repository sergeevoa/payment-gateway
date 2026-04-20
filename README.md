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

High-level component diagram showing services, protocols, and data flows:
```mermaid
flowchart TB
    %% External actors
    Merchant([Merchant Backend])
    Bank([Acquiring Bank])
    OFD([ОФД])

    %% Internal services
    subgraph Gateway_Cluster[" Payment Gateway "]
        direction TB
        GW[api-gateway<br/><i>Spring Cloud Gateway</i>]

        subgraph Core[" Core Services "]
            direction TB
            MS[merchant-service<br/><i>Spring Boot</i>]
            PS[payment-service<br/><i>Spring Boot</i>]
            CUS[customer-service<br/><i>Spring Boot</i>]
            CARD[card-service<br/><i>Spring Boot</i>]
            REC[receipt-service<br/><i>Spring Boot</i>]
            NS[notification-service<br/><i>Spring Boot</i>]
        end

        %% Infrastructure
        PG[(PostgreSQL)]
        RD[(Redis)]
        KAFKA{{Kafka}}
        MINIO[(MinIO)]
    end

    %% External → Gateway
    Merchant -->|REST + HMAC| GW

    %% Gateway → Core (sync, hot path)
    GW -->|gRPC| PS
    GW -->|gRPC| CARD
    GW -->|gRPC| CUS
    GW -->|gRPC| MS
    GW -->|gRPC| REC

    %% Inter-service sync calls
    PS -->|gRPC| MS
    PS -->|gRPC| CUS
    PS -->|gRPC| CARD
    CARD -->|gRPC| CUS
    CARD -->|gRPC| MS
    REC -->|gRPC| MS

    %% Outbound to external systems
    PS -->|REST| Bank
    REC -->|REST| OFD
    NS -->|REST + HMAC webhooks| Merchant

    %% Persistence
    MS --- PG
    PS --- PG
    CUS --- PG
    CARD --- PG
    REC --- PG
    NS --- PG

    %% Cache & shared state
    GW -.->|rate limit, idempotency| RD
    PS -.->|idempotency, locks| RD
    CARD -.->|state cache| RD
    MS -.->|config cache| RD

    %% Async events via Kafka
    PS -.->|outbox→publish| KAFKA
    CARD -.->|outbox→publish| KAFKA
    REC -.->|outbox→publish| KAFKA
    KAFKA -.->|consume| NS
    KAFKA -.->|consume| REC

    %% Object storage
    REC --- MINIO
    NS --- MINIO

    %% Styling
    classDef external fill:#e8e8e8,stroke:#555,stroke-width:2px,color:#000
    classDef service fill:#dbeafe,stroke:#1e40af,stroke-width:2px,color:#000
    classDef gateway fill:#fef3c7,stroke:#b45309,stroke-width:2px,color:#000
    classDef datastore fill:#dcfce7,stroke:#166534,stroke-width:2px,color:#000
    classDef broker fill:#fce7f3,stroke:#9f1239,stroke-width:2px,color:#000

    class Merchant,Bank,OFD external
    class MS,PS,CUS,CARD,REC,NS service
    class GW gateway
    class PG,RD,MINIO datastore
    class KAFKA broker

    %% Subgraph styling — transparent fill, thin border
    style Gateway_Cluster fill:none,stroke:#666,stroke-width:1.5px,stroke-dasharray: 5 5
    style Core fill:none,stroke:#999,stroke-width:1px,stroke-dasharray: 3 3
```

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

**Language & Framework:** Java 17, Spring Boot 3, Spring Cloud Gateway, Spring Data JPA, Spring Security  
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
