# Occupify API Gateway - Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & System Entry Point:**
> - External Gateway Port: `8180`
> - Downstream Services:
>   - Identity: `http://localhost:8181`
>   - Core: `http://localhost:8182`
>   - Payment: `http://localhost:8183`
>   - Notification: `http://localhost:8184`
>   - Administration: `http://localhost:8185`

---

## 1. Overview & Core Responsibilities

The **Occupify API Gateway** (`occupify-gateway`) is built on **Spring Cloud Gateway (WebFlux)** and serves as the single public reverse-proxy and security gatekeeper for the platform.

### Core Functions:
1. **JWT Authentication & Validation:** Parses Bearer tokens from incoming `Authorization` headers using HMAC-SHA512 (`HS512`). Validates token signatures and expiration.
2. **Context Propagation (Header Ingestion):** Injects downstream headers:
   - `X-User-Id`: User UUID string
   - `X-User-Role`: User role string (`USER` or `ADMIN`)
   - `X-User-Email`: Authenticated user email
3. **Correlation Tracking:** Generates and appends `X-Correlation-Id` across all requests and responses for end-to-end distributed tracing.
4. **Resilience & Fault Tolerance:** Employs Resilience4j Circuit Breakers and Retry policies for all downstream HTTP calls. Requests failing or timing out are redirected to internal fallback endpoints.
5. **Cross-Origin Resource Sharing (CORS):** Manages pre-flight `OPTIONS` requests and configures allowed origins (`http://localhost:3000`).

---

## 2. Gateway Fallback Endpoints (`/fallback/*`)

When a downstream service instance crashes, becomes unresponsive, or trips its circuit breaker, the gateway catches the failure and returns a structured `ErrorResponse`.

### 2.1 Common Fallback
- **Endpoint:** `GET|POST|PUT|DELETE|PATCH /fallback/common`
- **Trigger:** Generic fallback when downstream route fails.
- **Response:** `503 SERVICE_UNAVAILABLE` (`SYS_008`) or `504 GATEWAY_TIMEOUT` (`SYS_009`).

### 2.2 Service-Specific Fallback
- **Endpoint:** `GET|POST|PUT|DELETE|PATCH /fallback/{service}`
- **Supported Path Variables:** `core`, `payment`, `notification`, `administration`.
- **Response Example (`503 SERVICE_UNAVAILABLE`):**
  ```json
  {
    "statusCode": 503,
    "message": "Core marketplace service is temporarily unavailable. Please try again later.",
    "errorCode": "SYS_008"
  }
  ```
- **Response Example (`504 GATEWAY_TIMEOUT`):**
  ```json
  {
    "statusCode": 504,
    "message": "Payment service request timed out. Please try again later.",
    "errorCode": "SYS_009"
  }
  ```

---

## 3. Path Authorization Policies

| Path Pattern | Security Rule | Required Role / Condition |
| :--- | :--- | :--- |
| `/auth/**` | Public / Whitelisted | No token required |
| `/projects` (GET), `/projects/{id}` (GET) | Public / Whitelisted | No token required |
| `/actuator/**`, `/v3/api-docs/**`, `/swagger-ui/**`, `/scalar/**` | Whitelisted | Monitoring / Documentation |
| `/fallback/**` | Internal / Whitelisted | Internal Gateway fallback |
| `/admin/**`, `/audit-logs/**`, `/reports/**` | Role Restricted | Must have `X-User-Role: ADMIN` |
| All other paths (`/projects` POST/PUT/PATCH, `/users/**`, `/notifications/**`, `/contracts/**`) | Authenticated | Valid Bearer JWT token required |
