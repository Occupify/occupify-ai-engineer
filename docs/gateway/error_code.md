# Occupify API Gateway Error Codes Specification

This document details all error codes produced directly by `occupify-gateway`.

## Error Codes Reference Table

| Error Code | HTTP Status | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- |
| **`SYS_001`** | `500 INTERNAL_SERVER_ERROR` | **Unexpected Gateway Error** | Unhandled server error in reactive gateway filter pipeline. |
| **`SYS_002`** | `400 BAD_REQUEST` | **Validation Error** | Request parameter or header syntax failure. |
| **`SYS_003`** | `500 INTERNAL_SERVER_ERROR` | **Database / Internal Error** | Internal system fault. |
| **`SYS_004`** | `404 NOT_FOUND` | **Route Not Found** | Client requested a path that matches no route predicate. |
| **`SYS_005`** | `405 METHOD_NOT_ALLOWED` | **Method Not Allowed** | HTTP method is not supported for requested route. |
| **`SYS_006`** | `401 UNAUTHORIZED` | **Unauthorized Access** | Missing, malformed, or expired `Authorization: Bearer <token>` header on a protected endpoint. |
| **`SYS_007`** | `403 FORBIDDEN` | **Access Denied** | Authenticated user lacks `ADMIN` role when attempting to access `/admin/**`, `/audit-logs/**`, or `/reports/**`. |
| **`SYS_008`** | `503 SERVICE_UNAVAILABLE` | **Downstream Service Unavailable** | Circuit breaker is OPEN or downstream service instance is unreachable. |
| **`SYS_009`** | `504 GATEWAY_TIMEOUT` | **Downstream Gateway Timeout** | Downstream service took longer than the configured TimeLimiter timeout (5000ms) to respond. |
| **`AUTH_003`** | `401 UNAUTHORIZED` | **Invalid Token Credentials** | JWT signature verification failed or token has been revoked. |
| **`USER_007`** | `403 FORBIDDEN` | **Inactive / Locked Account** | JWT claims indicate account status is inactive or locked. |
| **`USER_008`** | `403 FORBIDDEN` | **Banned User Account** | JWT claims indicate account status is banned. |
