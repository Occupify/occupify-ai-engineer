# Spring Boot 3 + Java 25 Architecture & System Prompts

This document defines the operational directives, architectural rules, code hygiene standards, and behavioral boundaries for agents consulting under `plugins/backend/`.

---

## 1. Top-Level Architectural Directives

When advising on or generating code for Spring Boot microservices, agents must adhere to these inviolable architectural principles:

1. **Layered Clean Architecture & Strict Boundaries**:
   - Organize code into clean layers: `controller`, `service`, `repository`, `dto`, `mapper`, `exception`, `constant`, `config`, `event`.
   - Controllers must ONLY handle HTTP transport concerns (path mapping, parameter binding, DTO validation with `@Valid`, calling service, wrapping response in `ResponseFactory`). Absolutely NO business logic or SQL queries in controllers.
   - Services encapsulate ALL business logic, transaction boundaries (`@Transactional`), domain validations, and event publishing.
   - Repositories encapsulate database access using Spring Data JPA. Never leak entity persistence operations to controllers.

2. **Standardized API Contracts & ResponseFactory Envelopes**:
   - Every public REST endpoint must return standard response envelopes via `ResponseFactory`:
     - Single object retrieval / update: `ResponseEntity<SingleResponse<T>>` via `ResponseFactory.ok(data, message)`
     - Creation: `ResponseEntity<CreatedResponse<T>>` via `ResponseFactory.created(data, message)`
     - Void / mutation with no payload: `ResponseEntity<SuccessResponse>` via `ResponseFactory.success(message)`
     - Paginated list: `ResponseEntity<PageResponse<T>>` via `ResponseFactory.page(pageData, message)`
     - Errors: Handled centrally by `@RestControllerAdvice` returning `ResponseEntity<ErrorResponse>` via `ResponseFactory.error(status, code, message, details)`.
   - Never return raw Java entities, bare maps, or arbitrary unstructured JSON from controllers.

3. **Flyway Migrations: STRICTLY Sequential Increment Numbering**:
   - Every database migration script must follow the deterministic sequential format:
     `V1__<description>.sql`, `V2__<description>.sql`, `V3__<description>.sql`.
   - **Timestamp-based migration filenames (e.g., `V20261008...`) are STRICTLY PROHIBITED.** Timestamp versioning creates version drift across team branches, breaks clean sequential execution, and violates the project standard.
   - Always verify the current highest migration number in `src/main/resources/db/migration/` and increment by exactly 1 (`V{N+1}__`).
   - All migrations must be non-destructive (Expand and Contract pattern) and written in lowercase snake_case.

4. **DTO Immutability via Java 25 Records & MapStruct**:
   - All Request and Response DTOs must be declared as immutable Java 25 `record` types.
   - DTOs must carry `jakarta.validation.constraints` annotations (`@NotBlank`, `@NotNull`, `@Size`, `@Email`, etc.).
   - Entity-to-DTO and DTO-to-Entity conversions must be performed via type-safe MapStruct mappers (`@Mapper(componentModel = "spring")`). Never perform manual repetitive setter mapping in service code.

5. **Resilient Event Messaging (RabbitMQ)**:
   - Event contracts must be defined as immutable records in the `event` package.
   - Every event must include metadata: `correlationId`, `timestamp`, `eventType`, `sourceService`.
   - Consumers must be idempotent: verify message deduplication keys before executing business state transitions.
   - All consumer queues must be bound to a Dead Letter Exchange (`DLX`) and Dead Letter Queue (`DLQ`) with configurable retry policies.

6. **Microservice Security & Context Propagation**:
   - Downstream microservices sit behind Spring Cloud Gateway. The Gateway handles token validation and injects identity headers (`X-User-Id`, `X-User-Role`, `X-User-Email`).
   - Microservices extract this identity via custom `@Authenticated UserContext` method argument resolvers or filters. Never bypass or re-invent unvalidated token parsing.

7. **Zero-Failure Build & Slice Testing Gate**:
   - Every implementation must be covered by slice tests:
     - `@WebMvcTest` for Controller endpoint contracts, validation status codes, and `ResponseFactory` structure.
     - `@DataJpaTest` for custom repository query methods.
     - Pure unit tests with Mockito for Service domain logic.
   - `./mvnw clean test` MUST pass 100% with 0 failures before any code is marked as complete.

8. **Fault Tolerance & Defense-in-Depth Resilience**:
   - Ingress Gateway: All downstream routes must declare Spring Cloud Gateway `Retry` (restricted to idempotent `GET`, `HEAD` with exponential backoff on 502/503/504) and Resilience4j `CircuitBreaker` with fallback forwarding. Tripped requests must return standard `ErrorResponse` (HTTP 503 `SYS_008` / HTTP 504 `SYS_009`).
   - Microservices: Direct inter-service HTTP calls must enforce Resilience4j `@CircuitBreaker` and `@Retry`. Third-party APIs must use bulkhead thread isolation.
   - Consumers: RabbitMQ listeners must configure container retry with exponential backoff and route unrecoverable failures to DLX/DLQ.

---

## 2. Prohibited Antipatterns & Enforced Alternatives

| # | Antipattern | Why it is Prohibited | Enforced Production Alternative |
| :- | :--- | :--- | :--- |
| **1** | **Timestamp Flyway Migrations**<br>`V20261008120000__create_table.sql` | Causes version ordering drift, merge conflicts, and breaks deterministic deployment sequence across team environments. | **Sequential Increment Versioning**:<br>`V1__create_users_table.sql`, `V2__add_email_index.sql`. Check existing `V{N}` and increment by 1. |
| **2** | **Raw Entity Exposure in API**<br>`public ResponseEntity<User> getUser(...)` | Leaks internal database schema, sensitive columns (passwords, salts), triggers Jackson infinite recursion on JPA relations. | **Java 25 Record DTO + MapStruct**:<br>`public ResponseEntity<SingleResponse<UserResponse>> getUser(...)` mapped via `@Mapper`. |
| **3** | **Unwrapped API Responses**<br>`return ResponseEntity.ok(userDto);` | Inconsistent client response envelope; frontend cannot rely on unified status codes, timestamps, and message envelopes. | **ResponseFactory Envelope**:<br>`return ResponseFactory.ok(userResponse, "User retrieved successfully");` |
| **4** | **Business Logic in Controllers**<br>Writing validation, calculations, or entity mutations in `@RestController`. | Violates Single Responsibility, hinders unit testability, and prevents reusing logic across channels (e.g. messaging, CLI). | **Service Layer Encapsulation**:<br>Controller delegates entirely to `@Service`. Controller only handles HTTP protocol and DTO validation. |
| **5** | **Direct Cross-Service Database Access**<br>Microservice A connecting directly to Microservice B's database. | Breaks bounded contexts, causes tight coupling, and corrupts independent deployment capabilities. | **API Gateway / RabbitMQ Events**:<br>Communicate asynchronously via RabbitMQ topic exchanges or synchronously via REST through Gateway. |
| **6** | **Non-Transactional DB Mutations**<br>Modifying multiple entities across service calls without `@Transactional`. | Causes partial writes and corrupts database state if intermediate step fails or throws an exception. | **Explicit `@Transactional` Boundaries**:<br>Annotate mutation methods with `@Transactional`. Use `@Transactional(readOnly = true)` for query methods. |
| **7** | **Destructive Migrations (`DROP TABLE`, `ALTER DROP COLUMN`)**<br>Directly dropping columns in production migrations. | Breaks running instances of the previous version during rolling zero-downtime deployments. | **Expand and Contract Pattern**:<br>Deprecate column in V1, make nullable, deploy new app writing to new column, drop legacy column in V{N+2}. |
| **8** | **Unprotected RabbitMQ Consumers**<br>Processing queue messages without retry limits, DLQ, or deduplication. | Poison pills cause infinite retry loops, crashing consumer pods and starving the broker queue. | **Idempotent Consumer + DLX/DLQ**:<br>Declare DLX with dead-letter-routing-key; verify idempotency key in Redis/DB before processing; NACK with requeue=false on unrecoverable errors. |
| **9** | **Hardcoded Magic Strings & Paths**<br>`@RequestMapping("/api/v1/noti")` with inline magic values. | Leads to broken links, typos, and drift between documentation and code. | **Centralized Constants & Configuration**:<br>Define route prefixes, topic exchange names, and message keys in `constant` classes or `application.yml`. |
| **10** | **Lombok `@Data` on JPA Entities**<br>Using `@Data` on `@Entity` classes. | Generates `equals()`, `hashCode()`, and `toString()` that evaluate lazy associations, triggering LazyInitializationException or infinite loops. | **Selective Lombok Annotations**:<br>Use `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor` on entities. Override `equals` and `hashCode` based solely on business key or ID. |
| **11** | **Bypassing Gateway Identity Headers**<br>Parsing JWT tokens inside every downstream microservice independently. | Duplicates auth code, leaks cryptographic keys, and violates single source of truth for auth verification. | **Central Gateway Verification + `@Authenticated`**:<br>Gateway validates token, forwards `X-User-Id` / `X-User-Role`. Service injects via custom `HandlerMethodArgumentResolver`. |
| **12** | **Single-Agent Unverified Delivery**<br>Pushing code without running `./mvnw clean test` and multi-agent audit. | Leaves failing tests, broken migrations, or contract regressions undetected until CI pipeline fails. | **Automated Test Gate & Dual-QA Signoff**:<br>Must execute `./mvnw clean test` locally and obtain signoff from `backend-quality-auditor` and `backend-qa-tester`. |
| **13** | **Unprotected Gateway Routes / Blind Non-Idempotent Retries**<br>Forwarding routes without circuit breakers, or blindly retrying non-idempotent `POST` requests. | Downstream failure cascades into gateway thread exhaustion; retrying `POST` without idempotency causes duplicate charges or records. | **Circuit Breaker + Idempotent Retry**:<br>Declare Resilience4j circuit breaker with fallback on all gateway routes; restrict automatic retry strictly to safe/idempotent methods (`GET`, `HEAD`). |

---

## 3. Scope Discipline (Scaffolding vs Implementation)

When instructed to **"init service"**, **"scaffold microservice"**, or **"setup auth"**:
1. Create ONLY the foundational architecture:
   - Root `pom.xml` with dependencies (Web, Data JPA, Flyway, Postgres, Validation, MapStruct, Lombok, RabbitMQ/Actuator as needed).
   - `application.yml` and `application-test.yml` with placeholder environment variable substitutions (`${DB_HOST:localhost}`).
   - Base package structure: `config`, `constant`, `controller`, `dto`, `event`, `exception`, `mapper`, `repository`, `service`.
   - Core baseline classes: `AbstractBaseController`, `ResponseFactory`, `SingleResponse`, `CreatedResponse`, `PageResponse`, `ErrorResponse`, `GlobalExceptionHandler`, `BaseException`.
   - Baseline migration `V1__init_schema.sql` ONLY if the initial schema was explicitly requested.
2. DO NOT generate speculative business entities, mock controllers, or extra tables unless explicitly instructed in the user prompt.

---

## 4. Coding Standards by Layer

### 4.1 Controller Layer
```java
@RestController
@RequestMapping("/api/v1/properties")
@RequiredArgsConstructor
public class PropertyController extends AbstractBaseController {

    private final PropertyService propertyService;

    @PostMapping
    public ResponseEntity<CreatedResponse<PropertyResponse>> createProperty(
            @Valid @RequestBody CreatePropertyRequest request,
            @Authenticated UserContext userContext) {
        PropertyResponse response = propertyService.createProperty(request, userContext.userId());
        return ResponseFactory.created(response, "Property created successfully");
    }

    @GetMapping("/{id}")
    public ResponseEntity<SingleResponse<PropertyResponse>> getProperty(@PathVariable UUID id) {
        PropertyResponse response = propertyService.getPropertyById(id);
        return ResponseFactory.ok(response, "Property retrieved successfully");
    }

    @GetMapping
    public ResponseEntity<PageResponse<PropertyResponse>> getProperties(
            @Valid PropertyFilterRequest filter,
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
        Page<PropertyResponse> page = propertyService.getProperties(filter, pageable);
        return ResponseFactory.page(page, "Properties retrieved successfully");
    }
}
```

### 4.2 DTO Contracts (Java 25 Records)
```java
public record CreatePropertyRequest(
    @NotBlank(message = "Title is required")
    @Size(max = 200, message = "Title must not exceed 200 characters")
    String title,

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be strictly positive")
    BigDecimal price,

    @NotNull(message = "Address is required")
    @Valid
    AddressDto address
) {}
```

### 4.3 Service & Transaction Layer
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PropertyServiceImpl implements PropertyService {

    private final PropertyRepository propertyRepository;
    private final PropertyMapper propertyMapper;
    private final EventPublisher eventPublisher;

    @Override
    @Transactional
    public PropertyResponse createProperty(CreatePropertyRequest request, UUID ownerId) {
        Property property = propertyMapper.toEntity(request);
        property.setOwnerId(ownerId);
        property.setStatus(PropertyStatus.PENDING_REVIEW);

        Property saved = propertyRepository.save(property);
        eventPublisher.publishPropertyCreatedEvent(new PropertyCreatedEvent(saved.getId(), ownerId, Instant.now()));

        return propertyMapper.toResponse(saved);
    }
}
```

### 4.4 Flyway Migrations (Strict Sequential Increment)
```sql
-- src/main/resources/db/migration/V1__init_property_schema.sql
CREATE TABLE IF NOT EXISTS properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL,
    title VARCHAR(200) NOT NULL,
    price NUMERIC(14, 2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING_REVIEW',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_properties_owner_id ON properties(owner_id);
CREATE INDEX IF NOT EXISTS idx_properties_status ON properties(status);
```

---

## 5. Auditor Verification Checklist

Before certifying any backend change:
- [ ] Are all Flyway migrations named using **sequential increment numbers** (`V1__`, `V2__`, `V3__`)? Verify NO timestamps exist.
- [ ] Are all controller responses wrapped in `ResponseFactory` (`SingleResponse`, `CreatedResponse`, `PageResponse`, `ErrorResponse`)?
- [ ] Are all DTOs implemented as immutable Java 25 `record`s with `jakarta.validation` annotations?
- [ ] Are entities annotated with `@Getter`, `@Setter`, `@NoArgsConstructor` instead of `@Data`?
- [ ] Are read operations marked `@Transactional(readOnly = true)` and write operations marked `@Transactional`?
- [ ] Are RabbitMQ consumers protected with idempotency checks and bound to DLX/DLQ?
- [ ] Does `./mvnw clean test` execute and pass with 0 failures, 0 errors, 0 skipped regressions?
