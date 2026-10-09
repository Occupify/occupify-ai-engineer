# The 4 Resilience Tiers in Microservices Architecture

In distributed microservice ecosystems, failures are inevitable. A common architectural fallacy is believing that configuring a Circuit Breaker and Retry mechanism exclusively at the **API Gateway** makes the entire system resilient.

The API Gateway protects only **North-South ingress traffic** (client $\rightarrow$ gateway $\rightarrow$ service). It has zero control or visibility over internal synchronous calls, external third-party payment gateways, or asynchronous message broker event streams.

To achieve true enterprise-grade fault tolerance, **Occupify** enforces a **4-Tier Defense-in-Depth** resilience model.

---

## 1. System Resilience Topology

```mermaid
flowchart TD
    Client["Client App (React Web / Mobile)"] -->|North-South HTTP| Gateway["Tier 1: API Gateway (:8181)\n(Spring Cloud Gateway + Resilience4j)"]
    
    subgraph IngressLayer ["Ingress & Perimeter Boundary"]
        Gateway
    end
    
    subgraph DomainLayer ["Internal Microservices Domain"]
        Core["occupify-core (:8182)"]
        Payment["occupify-payment (:8183)"]
        Noti["occupify-noti (:8184)"]
        Admin["occupify-administration (:8185)"]
    end
    
    Gateway -->|Route + Fallback 503/504| Core
    Gateway -->|Route + Fallback 503/504| Payment
    Gateway -->|Route + Fallback 503/504| Noti
    Gateway -->|Route + Fallback 503/504| Admin
    
    Payment -.->|Tier 2: East-West HTTP\n(@CircuitBreaker + @Retry)| Core
    Admin -.->|Tier 2: East-West HTTP\n(@CircuitBreaker + @Retry)| Core
    
    Payment -->|Tier 3: 3rd-Party API\n(@Bulkhead + @TimeLimiter)| Stripe["Stripe / VNPay Payment Gateways"]
    Core -->|Tier 4: Domain Events| RabbitBroker["RabbitMQ Topic Exchange"]
    RabbitBroker -->|Queue with DLX / DLQ| Noti
    Noti -->|SMTP with Retry| MailServer["SMTP Mail Gateway (Gmail / SES)"]
    
```

---

## 2. The 4 Resilience Tiers Comparison Matrix

| Tier | Traffic Scope | What Can Go Wrong? | Why Gateway Alone Cannot Help | Enforced Production Solution |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1: API Gateway** *(Ingress Perimeter)* | **North-South**<br>Client $\rightarrow$ API Gateway $\rightarrow$ Microservice | A microservice crashes or experiences latency spikes, exhausting Gateway Netty/worker threads and taking down the entire ingress entry point. | The Gateway protects itself and client perceptions, but has zero visibility into internal service-to-service calls or background jobs. | **Spring Cloud Gateway + Resilience4j Reactive**:<br>• Automatic fail-fast via Circuit Breaker<br>• Retry with exponential backoff strictly for idempotent methods (`GET`, `HEAD`)<br>• Uniform `ErrorResponse` fallback (503/504)<br>• Fallback whitelist exemption |
| **Tier 2: Inter-Service** *(East-West)* | **East-West**<br>Microservice $\rightarrow$ Microservice (e.g. `Payment` $\rightarrow$ `Core`) | When `Payment` synchronously calls `Core` to verify a project status, and `Core` hangs, `Payment`'s thread pool is consumed and crashes too. | Internal HTTP calls occur directly between microservice containers on the private network, bypassing the API Gateway entirely. | **Resilience4j Annotations on `RestClient`**:<br>• `@CircuitBreaker(name = "coreService")`<br>• `@Retry(name = "coreService")`<br>• Typed fallback methods returning degraded default responses or domain exceptions |
| **Tier 3: 3rd-Party Integrations** | **Outbound External**<br>Microservice $\rightarrow$ External Vendor (Stripe, VNPay, PayPal, Cloudinary) | Payment provider experiences latency spikes or downtime. Unbounded HTTP requests exhaust local web threads, freezing unrelated user endpoints. | External outbound traffic originates inside the microservice container, outside Gateway jurisdiction. | **Thread Pool Bulkhead + TimeLimiter**:<br>• `@Bulkhead(type = THREADPOOL)` isolates external call threads<br>• `@TimeLimiter` bounds timeouts (e.g. 5s)<br>• Idempotency keys prevent duplicate credit charges |
| **Tier 4: Event-Driven & Async** | **Asynchronous AMQP**<br>RabbitMQ Consumers & SMTP Email | 1. Poison pill messages throw unhandled exceptions, causing infinite redelivery loops.<br>2. Transient SMTP socket drop loses OTP activation emails. | Asynchronous messaging brokers and SMTP transports bypass HTTP gateway filters entirely. | **RabbitMQ DLX/DLQ + Consumer Idempotency**:<br>• Container retry (3 attempts with backoff)<br>• Dead Letter Exchange (`DLX`/`DLQ`) on exhausted retries<br>• Redis key deduplication (`setIfAbsent`)<br>• Spring `@Retryable` on mail sender |

---

## 3. Tier-by-Tier Implementation Specifications

### Tier 1: API Gateway (Ingress Fault Tolerance)

- **Service**: `occupify-identity` (:8181)
- **Primary Mechanism**: Spring Cloud Gateway + `spring-cloud-starter-circuitbreaker-reactor-resilience4j`
- **Key Parameters**:
  - `slidingWindowSize: 10` (count-based)
  - `minimumNumberOfCalls: 5`
  - `failureRateThreshold: 50%`
  - `slowCallDurationThreshold: 3000ms`
  - `waitDurationInOpenState: 10000ms` (auto-transitions to HALF-OPEN)
  - `timeoutDuration: 5000ms` (TimeLimiter)
- **Safe Retry Rules**:
  - Automatically retry only idempotent methods (`GET`, `HEAD`) on `502`, `503`, and `504` status codes.
  - Never blindly retry non-idempotent operations (`POST`, `PATCH`, `DELETE`) without distributed idempotency tokens.
- **Fallback Handling**:
  - Failed or timed-out requests forward to `GatewayFallbackController` (`/fallback/{service}`).
  - Returns unified `ErrorResponse` (HTTP 503 `SYS_008` / HTTP 504 `SYS_009`) with preserved `correlationId`.
  - Whitelist `/fallback/**` in `SecurityProperties` to guarantee error dispatch without token checks.

---

### Tier 2: Inter-Service Communication (East-West Isolation)

- **Target Services**: `occupify-payment`, `occupify-core`, `occupify-administration`
- **Primary Mechanism**: `resilience4j-spring-boot3` annotations on Spring 6 `RestClient`
- **Pattern**:
  ```java
  @Service
  @RequiredArgsConstructor
  public class CoreServiceClientImpl implements CoreServiceClient {

      private final RestClient restClient;

      @CircuitBreaker(name = "coreService", fallbackMethod = "getProjectFallback")
      @Retry(name = "coreService")
      public ProjectResponse getProjectById(UUID projectId) {
          return restClient.get()
                  .uri("/projects/{id}", projectId)
                  .retrieve()
                  .body(ProjectResponse.class);
      }

      private ProjectResponse getProjectFallback(UUID projectId, Throwable ex) {
          log.error("Core service call failed for projectId [{}]: {}", projectId, ex.getMessage());
          throw new ServiceUnavailableException("Core service is temporarily unreachable", ex);
      }
  }
  ```

---

### Tier 3: Third-Party External Integrations (Vendor Shielding)

- **Target Integrations**: Stripe, VNPay, PayPal, Cloudinary, AWS S3
- **Primary Mechanism**: Resilience4j `@Bulkhead` (Thread Pool Isolation) + `@TimeLimiter`
- **Why It Matters**: If Stripe slows down from 200ms to 8,000ms, your web server threads will back up and freeze the entire payment service for all users. A Thread Pool Bulkhead restricts external calls to a dedicated, bounded pool (e.g., 10 threads). If the pool fills up, excess requests fast-fail immediately without affecting local database queries or account lookups.

---

### Tier 4: Event-Driven & Async Processing (Broker Resilience)

- **Target Integrations**: RabbitMQ consumers, Spring MailSender
- **Primary Mechanism**: Spring AMQP Container Retry, Dead Letter Exchange (`DLX`/`DLQ`), Redis Deduplication
- **Poison Pill Elimination**:
  1. A malformed event reaches a consumer queue (`occupify.notification.queue`).
  2. The consumer retries up to 3 times with exponential backoff (1s $\rightarrow$ 2s $\rightarrow$ 4s).
  3. If still failing, the message is rejected (`requeue = false`) and automatically rerouted to `occupify.notification.dlx` $\rightarrow$ `occupify.notification.dlq`.
  4. Active queues remain unblocked and consumers never crash in infinite loops.
- **Consumer Deduplication**:
  ```java
  @RabbitHandler
  public void handleUserRegistered(UserRegisteredEvent event) {
      String dedupKey = "event:dedup:" + event.eventId();
      Boolean isNew = redisTemplate.opsForValue().setIfAbsent(dedupKey, "PROCESSED", Duration.ofDays(7));
      if (Boolean.FALSE.equals(isNew)) {
          log.warn("Duplicate event [{}] ignored", event.eventId());
          return;
      }
      emailService.sendRegistrationOtp(event.email(), event.otpCode());
  }
  ```

---

## 4. Observability & Health Monitoring

All tiers export runtime metrics and states to Spring Boot Actuator for Prometheus/Grafana dashboards:

- `GET /actuator/health` — Shows overall health and per-circuit breaker state (`UP`, `CIRCUIT_OPEN`).
- `GET /actuator/circuitbreakers` — Inspects all active circuit breaker instances and their sliding windows.
- `GET /actuator/circuitbreakerevents` — Emits real-time state transition and failure events.
- `GET /actuator/metrics/resilience4j.circuitbreaker.calls` — Quantifies total successful, failed, and short-circuited call volumes.
