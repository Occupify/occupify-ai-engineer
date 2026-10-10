---
name: spring-boot
description: Production Spring Boot 3 + Java 25 microservice architecture and engineering consultation skill. Use when architecting enterprise microservices, enforcing layered clean architecture, standardizing DTOs and API responses (ResponseFactory, SingleResponse, CreatedResponse, PageResponse, ErrorResponse), writing deterministic sequential Flyway migrations (strictly V1__, V2__, V3__ increment numbers, never timestamps), designing resilient RabbitMQ event messaging (DLX/DLQ, idempotency), securing microservices with Spring Cloud Gateway and JWT (@Authenticated user context), and validating code with comprehensive slice testing (@WebMvcTest, @DataJpaTest) under a zero-failure build gate (./mvnw clean test).
---

# Production Spring Boot 3 + Java 25 Microservice Skill

This skill delivers enterprise-grade architecture consultation, design pattern implementation, database evolution safety, and code auditing for modern Spring Boot 3 microservices running on Java 25.

## Skill Specifications & Manifests

- **Configuration & Triggers**: [config.json](./config.json)  
  Defines triggers, agent roles (`spring-tech-lead`, `spring-core-dev`, `api-gateway-dev`, `backend-qa-tester`, `backend-quality-auditor`), source hierarchies, and architectural policies.
- **Architectural Directives & Rules**: [system_prompt.md](./system_prompt.md)  
  Contains operational directives, strict Java 25 constraints, prohibited antipatterns, and code generation guidelines.
- **Consultation & Execution Protocol**: [workflow.md](./workflow.md)  
  Specifies the 5-phase consultation lifecycle, audit checklist, and verification protocols.

## Progressive Knowledge References

Detailed architectural reference manuals are located in `references/`:

- **[Microservice Layered Architecture & Contracts](./references/microservice-architecture.md)**: Clean layered architecture (`controller`, `service`, `repository`, `dto`, `mapper`, `exception`, `constant`, `config`, `event`), standardized DTO records, `ResponseFactory` wrapper taxonomy (`SingleResponse`, `CreatedResponse`, `PageResponse`, `ErrorResponse`), `BaseException` hierarchy, and MapStruct mappings.
- **[Database Design & Flyway Sequential Migrations](./references/database-and-migrations.md)**: PostgreSQL schema design, JPA entity best practices, transaction boundaries (`@Transactional(readOnly = true)`), non-destructive expand-and-contract patterns, and **strictly sequential Flyway increment versioning** (`V1__`, `V2__`, `V3__` — timestamp versioning is strictly prohibited).
- **[Messaging & Event-Driven Resilience](./references/messaging-and-resilience.md)**: RabbitMQ event payloads, topic exchange routing, dead-letter exchanges and queues (DLX/DLQ), idempotent message processing, exponential retry backoff, and distributed correlation ID propagation.
- **[Fault Tolerance & Resilience Architecture](./references/fault-tolerance-and-resilience.md)**: 4-Tier Defense-in-Depth resilience model, Spring Cloud Gateway + Resilience4j Circuit Breaker & TimeLimiter, idempotent Gateway Retry backoff, unified FallbackController contracts, inter-service `@CircuitBreaker`, 3rd-party bulkhead thread isolation, and Actuator observability.
- **[Security & Gateway Integration](./references/security-and-gateway.md)**: Spring Cloud Gateway header propagation, JWT verification, custom `@Authenticated` argument resolvers for `X-User-Id` / `X-User-Role`, and OWASP defensive standards.
- **[Testing Strategy & Quality Gates](./references/testing-strategy.md)**: Test-Driven Development (TDD) workflow, slice testing (`@WebMvcTest`, `@DataJpaTest`), isolated Mockito boundaries, and the non-negotiable zero-failure build gate (`./mvnw clean test`).

## Quick Execution Summary

1. **Scope Calibration**: Distinguish between service bootstrapping (minimal scaffolding with pom.xml, health endpoint, base packages) and full business feature implementation. Never generate speculative or unrequested mock tables.
2. **Architecture Discovery**: Ingest domain requirements, determine service boundaries, define entity relationships, and enforce strict layered separation (Controllers handle HTTP only, Services encapsulate all transactions and business rules).
3. **API Contract & DTO First**: Define immutable Java 25 `record` DTOs with `jakarta.validation` annotations. Wrap every HTTP response in standard `ResponseFactory` envelopes (`SingleResponse`, `CreatedResponse`, `PageResponse`, `ErrorResponse`).
4. **Flyway Migration Discipline**: Enforce **strictly sequential increment numbers** (`V1__init_...sql`, `V2__add_...sql`, `V3__...`). Verify that every migration script is non-destructive, uses lowercase snake_case naming, and includes rollback safety considerations.
5. **Event-Driven Resilience**: When emitting or consuming RabbitMQ events, declare typed payload records, establish DLX/DLQ routing keys, and guarantee consumer idempotency using message unique identifiers.
6. **Slice Testing & Build Verification**: Write targeted slice tests (`@WebMvcTest` for controllers, `@DataJpaTest` for repositories, plain unit tests for services). Verify that `./mvnw clean test` passes with 100% success before committing.
7. **Mandatory Dual-Agent Review**: Enforce multi-agent verification (`backend-quality-auditor` for static compliance + `backend-qa-tester` for test execution) before final tech lead signoff.
