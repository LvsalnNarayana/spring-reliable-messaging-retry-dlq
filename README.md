
# Spring-Reliable-Messaging-Retry-Dlq

## Overview

This project is a **multi-microservice demonstration** of **reliable asynchronous messaging** using **RabbitMQ** and **Spring AMQP**, focusing on production-grade **retry mechanisms**, **exponential backoff**, and **Dead Letter Queue (DLQ)** routing.

It showcases how to build fault-tolerant message-driven systems where temporary failures do not result in lost messages, and permanent failures are safely isolated for investigation.

## Real-World Scenario

In payment gateways like **Stripe** or **PayPal**:
- Payment authorization requests are sent asynchronously to processors or third-party providers.
- Failures can occur due to network timeouts, rate limits, or temporary unavailability.
- Critical business events **must not be lost** — they need automatic retries.
- After exhausting retries, failed messages go to a **Dead Letter Queue** for manual review, alerting, or compensatory actions.

This ensures **reliability**, **auditability**, and **operational visibility** in high-stakes financial workflows.

## Microservices Involved

| Service                   | Responsibility                                                                 | Port  |
|---------------------------|--------------------------------------------------------------------------------|-------|
| **eureka-server**         | Service discovery (Netflix Eureka)                                             | 8761  |
| **payment-gateway-service**| Receives payment requests, publishes to processing queue                       | 8081  |
| **payment-processor-service**| Consumes messages, simulates processing with injected failures & retries     | 8082  |
| **dlq-auditor-service**   | Consumes from DLQ, persists failures, allows manual reprocessing               | 8083  |
| **notification-service**  | Sends alerts on success or DLQ routing                                         | 8084  |

All messaging flows through **RabbitMQ** with dedicated queues for main, retry, and DLQ.

## Tech Stack

- Spring Boot 3.x
- Spring AMQP + RabbitMQ
- Spring Retry + Exponential Backoff
- Dead Letter Exchange/Queue configuration
- PostgreSQL (persistence of payments and DLQ records)
- Spring Cloud Netflix Eureka
- Micrometer + Actuator (metrics)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: payments
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  payment-gateway-service:
    build: ./payment-gateway-service
    depends_on:
      - rabbitmq
      - eureka-server
    ports:
      - "8081:8081"

  payment-processor-service:
    build: ./payment-processor-service
    depends_on:
      - rabbitmq
      - postgres
      - eureka-server
    ports:
      - "8082:8082"

  dlq-auditor-service:
    build: ./dlq-auditor-service
    depends_on:
      - rabbitmq
      - postgres
      - eureka-server
    ports:
      - "8083:8083"

  notification-service:
    build: ./notification-service
    depends_on:
      - rabbitmq
      - eureka-server
    ports:
      - "8084:8084"
```

Run with: `docker-compose up --build`

## Retry & DLQ Strategy

| Component                | Configuration Details                                                  |
|--------------------------|------------------------------------------------------------------------|
| **Main Queue**           | `payments.process` — TTL, dead-letter-exchange                         |
| **Retry Policy**         | Exponential backoff: initial 1s → multiplier 2 → max 60s               |
| **Max Attempts**         | 5 retries before routing to DLQ                                        |
| **DLQ**                  | `payments.dlq` — separate exchange and queue                           |
| **Message Enrichment**   | Headers: `x-retry-count`, `x-original-queue`, `x-exception-message`   |
| **Idempotency**          | Payment ID as business key — prevents duplicates on reprocessing      |

## Key Features

- Configurable retry with exponential backoff
- Automatic DLQ routing after exhaustion
- Message headers enriched with failure details
- Idempotent processing using payment ID
- Manual reprocessing endpoint from DLQ
- Metrics: retry count, DLQ volume, success rate
- RabbitMQ Management UI for queue monitoring
- Notification on final success or DLQ entry

## Expected Endpoints

### Payment Gateway Service (`http://localhost:8081`)

| Method | Endpoint                  | Description                                      |
|--------|---------------------------|--------------------------------------------------|
| POST   | `/api/payments`           | Initiate payment → publish to queue              |
| GET    | `/api/payments/{id}`      | Check payment status                             |

### Payment Processor Service (`http://localhost:8082`)

| Method | Endpoint                  | Description                                      |
|--------|---------------------------|--------------------------------------------------|
| GET    | `/api/metrics/retries`    | View retry counters                              |

### DLQ Auditor Service (`http://localhost:8083`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/dlq/payments`             | List failed payments in DLQ                      |
| POST   | `/api/dlq/reprocess/{id}`       | Manually republish a failed payment               |
| GET    | `/api/dlq/stats`                | DLQ volume and reasons                           |

### Notification Service (`http://localhost:8084`)

| Method | Endpoint                  | Description                                      |
|--------|---------------------------|--------------------------------------------------|
| GET    | `/api/notifications`      | Recent notifications (success/DLQ alerts)        |

### RabbitMQ Management
- URL: `http://localhost:15672` (admin/admin)

## Architecture Overview

```
Clients
   ↓
Payment Gateway → RabbitMQ (payments.process queue)
   ↓
Payment Processor ← retries with backoff
   ↓ (on failure after max)
Dead Letter Queue → DLQ Auditor
   ↓
Notification Service (alerts)
   ↓
PostgreSQL (source of truth)
```

**Failure Flow**:
1. Message published → Processor consumes
2. Temporary failure → NACK + requeue with delay
3. After 5 attempts → routed to DLQ
4. DLQ Auditor persists + notifies
5. Manual reprocess → republish to main queue

## How to Run

1. Clone repository
2. Start Docker
3. `docker-compose up --build`
4. Access RabbitMQ UI: `http://localhost:15672`
5. Initiate payment: POST `/api/payments` (simulate failure via header)
6. Observe retries in logs → eventual DLQ routing

## Testing Reliability

1. Send payment with `X-Simulate-Failure: transient` → see retries
2. Send with `X-Simulate-Failure: permanent` → direct to DLQ after max retries
3. Check RabbitMQ UI → messages move to DLQ
4. Use auditor to reprocess → succeeds on retry

## Skills Demonstrated

- Advanced Spring AMQP retry configuration
- Exponential backoff and DLQ patterns
- Message header enrichment and idempotency
- Operational visibility (queues, metrics)
- Fault-tolerant asynchronous architecture
- Manual intervention workflows for failed messages

## Future Extensions

- Delayed retries using RabbitMQ Delayed Message Plugin
- Distributed tracing with Zipkin
- Outbox pattern for transactional consistency
- Integration with external dead-letter alerting (Slack/PagerDuty)
