# payment-gateway
Payment gateway with microservice architecture built with Java, Spring Framework, Kafka, PostgreSQL, Redis, Docker. Supports payments, card binding, recurring charges, refunds, webhook notifications, and receipt generation.

## API Documentation
https://sergeevoa.github.io/payment-gateway/

## Authorization
All API endpoints require a bearer token in the `Authorization` header.
```http
Authorization: Bearer <your_api_key>
```
Api keys are managed through the Merchant Portal - a separate application for merchant registration and credential management.
🚧 Merchant Portal is currently under development.