# Occupify Ecosystem — Centralized Architecture & API Documentation

Welcome to the centralized documentation repository for the **Occupify** microservices ecosystem. This repository serves as the single source of truth for service ports, API routing, authentication propagation, asynchronous event topology, and standardized error specifications across all services.

---

## 1. Centralized Service Port Matrix

| Service | Type | App Port | Database / Broker | Database URL / Connection | Status |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **`occupify-gateway`** | API Gateway (Spring Cloud Gateway) | `8180` | None | N/A | Active |
| **`occupify-identity`** | Auth & Identity Service | `8181` | PostgreSQL, Redis, RabbitMQ | `jdbc:postgresql://localhost:5530/occupify_identity`<br>`redis://localhost:6379`<br>`amqp://guest:guest@localhost:5672` | Active |
| **`occupify-core`** | Marketplace & Projects Service | `8182` | PostgreSQL, RabbitMQ | `jdbc:postgresql://localhost:5531/occupify_core`<br>`amqp://guest:guest@localhost:5672` | Active |
| **`occupify-payment`** | Contracts, Wallets & Billing | `8183` | PostgreSQL | `jdbc:postgresql://localhost:5532/occupify_payment` | Scaffolding |
| **`occupify-noti`** | Mail & Push Notification Service | `8184` | PostgreSQL, RabbitMQ, SMTP | `jdbc:postgresql://localhost:5533/occupify_noti`<br>`amqp://guest:guest@localhost:5672`<br>`smtp://smtp.gmail.com:587` | Active |
| **`occupify-administration`**| Admin & Audit Service | `8185` | PostgreSQL | `jdbc:postgresql://localhost:5534/occupify_administration` | Scaffolding |
| **`occupify-agentic`** | AI Agentic & Match/Search Service | `8186` | PostgreSQL (pgvector), RabbitMQ | `postgresql+asyncpg://postgres:postgres@localhost:5535/occupify_agentic`<br>`amqp://guest:guest@localhost:5672` | Active |
| **`occupify-web`** | Frontend Web (Next.js / React) | `3000` | N/A | Calls Gateway (`http://localhost:8180`) | Active |

---

## 2. API Gateway Routing Matrix (`occupify-gateway` :8180)

All external traffic from clients and web applications enters exclusively through the API Gateway on port `8180`. Downstream services should not be exposed directly to public clients.

| Route ID | Predicate Path | Downstream Target Service | Default Upstream URL | Resilience4j Circuit Breaker |
| :--- | :--- | :--- | :--- | :--- |
| `auth-service` | `/auth/**`, `/account/**`, `/users/**`, `/user/**` | `occupify-identity` | `http://localhost:8181` | Direct pass-through |
| `core-service` | `/projects/**`, `/applications/**`, `/reviews/**`, `/bids/**` | `occupify-core` | `http://localhost:8182` | `coreCircuitBreaker` (`forward:/fallback/core`) |
| `payment-service` | `/contracts/**`, `/wallets/**`, `/payments/**` | `occupify-payment` | `http://localhost:8183` | `paymentCircuitBreaker` (`forward:/fallback/payment`) |
| `notification-service` | `/notifications/**` | `occupify-noti` | `http://localhost:8184` | `notificationCircuitBreaker` (`forward:/fallback/notification`) |
| `administration-service` | `/admin/**`, `/audit-logs/**`, `/reports/**` | `occupify-administration` | `http://localhost:8185` | `adminCircuitBreaker` (`forward:/fallback/administration`) |
| `ai-service` | `/ai/**` | `occupify-agentic` | `http://localhost:8186` | Direct pass-through (No timeout, supports SSE streaming) |

---

## 3. Authentication & Header Propagation Contract

### Inbound Client Request
Clients send standard JWT Bearer tokens to Gateway:
```http
Authorization: Bearer <jwt_access_token>
```

### Gateway Token Inspection & Header Injection
`occupify-gateway` validates the JWT token, extracts claims, and injects trusted identity headers to downstream services:
- **`X-User-Id`**: UUID string representing the authenticated user.
- **`X-User-Role`**: User role string — strictly either `USER` or `ADMIN`.
- **`X-User-Email`**: Authenticated user's verified email address.

### Downstream Ingestion
Downstream services (e.g., `occupify-core`, `occupify-noti`) resolve these headers automatically via custom parameter resolvers (such as `@Authenticated UserContext userContext` or `@Authenticated UUID userId`).

---

## 4. Asynchronous Event Topology (RabbitMQ)

The system relies on RabbitMQ topic exchanges for cross-service eventual consistency.

```
[occupify-identity]
        |
        | (routing key: user.registered)
        v
[occupify.notification.exchange] (Topic)
        |
        +-----> [occupify.core.user.queue] ---------> [occupify-core] (Auto-sync User entity)
        |          (DLX: occupify.core.dlx -> occupify.core.user.dlq)
        |
        +-----> [occupify.notification.user.queue] -> [occupify-noti] (Send registration OTP email)

[occupify-core]
        |
        | (routing keys: project.*, job.#)
        v
[occupify.core.exchange] (Topic)
        |
        +-----> [occupify.agent.job.sync.queue] ----> [occupify-agentic] (Real-time pgvector sync)
                   (DLX: occupify.agent.dlx -> occupify.agent.job.sync.dlq)
```

| Event Record | Publisher | Exchange | Routing Key | Consumers / Queues | Business Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `UserRegisteredEvent` | `occupify-identity` | `occupify.notification.exchange` | `user.registered` | 1. `occupify.core.user.queue` (`occupify-core`)<br>2. `occupify.notification.user.queue` (`occupify-noti`) | Auto-creates `User` in `occupify-core` and dispatches welcome/verification OTP email in `occupify-noti`. |
| `PasswordResetRequestedEvent` | `occupify-identity` | `occupify.notification.exchange` | `user.password-reset` | `occupify.notification.user.queue` (`occupify-noti`) | Sends password reset email with temporary OTP. |
| `ProjectCreatedEvent` | `occupify-core` | `occupify.notification.exchange` | `project.created` | `occupify.notification.queue` (`occupify-noti`) | Broadcasts new project listing notifications to matching freelancers. |
| `ProjectClosedEvent` | `occupify-core` | `occupify.notification.exchange` | `project.closed` | `occupify.notification.queue` (`occupify-noti`) | Notifies applicants and workers of project termination. |
| `ProjectCreatedDomainEvent` | `occupify-core` | `occupify.core.exchange` | `project.created` | `occupify.agent.job.sync.queue` (`occupify-agentic`) | Syncs newly published jobs into pgvector job search index. |
| `ProjectUpdatedDomainEvent` | `occupify-core` | `occupify.core.exchange` | `project.updated` | `occupify.agent.job.sync.queue` (`occupify-agentic`) | Updates title, description, skills, budget in vector search index. |
| `ProjectClosedDomainEvent` | `occupify-core` | `occupify.core.exchange` | `project.closed` | `occupify.agent.job.sync.queue` (`occupify-agentic`) | Marks job as closed/unsearchable in vector search index. |
| `JobStatusChangedDomainEvent` | `occupify-core` | `occupify.core.exchange` | `job.status.changed` | `occupify.agent.job.sync.queue` (`occupify-agentic`) | Updates active status of job in search index. |

---

## 5. Microservice Specifications Directory

| Service | API Specification | Error Codes Specification |
| :--- | :--- | :--- |
| **API Gateway** | [Gateway Specs](gateway/api_specification.md) | [Gateway Error Codes](gateway/error_code.md) |
| **Identity Service** | [Identity Specs](identity/api_specification.md) | [Identity Error Codes](identity/error_code.md) |
| **Core Marketplace** | [Core Specs](core/api_specification.md) | [Core Error Codes](core/error_code.md) |
| **Notification Service** | [Notification Specs](notification/api_specification.md) | [Notification Error Codes](notification/error_code.md) |
| **Payment Service** | [Payment Specs](payment/api_specification.md) | [Payment Error Codes](payment/error_code.md) |
| **Administration Service**| [Administration Specs](administration/api_specification.md) | [Administration Error Codes](administration/error_code.md) |
| **AI Agentic Service** | [Agentic Specs](agentic/api_specification.md) | [Agentic Error Codes](agentic/error_code.md) |
