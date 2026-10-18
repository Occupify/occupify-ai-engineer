# Occupify Administration Error Codes Specification

This document details all business and system error codes reserved for `occupify-administration`.

## Error Codes Reference Table

| Error Code | HTTP Status | Category | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **`ADM_001`** | `404 NOT_FOUND` | User Moderation | **User Not Found** | Thrown when attempting to moderate a user ID that does not exist. |
| **`ADM_002`** | `400 BAD_REQUEST` | User Moderation | **Illegal Status Transition** | Thrown when attempting an invalid status transition (e.g. banning an already banned user). |
| **`ADM_003`** | `404 NOT_FOUND` | Audit | **Audit Log Not Found** | Thrown when referenced audit log ID does not exist. |
| **`ADM_004`** | `404 NOT_FOUND` | Dispute | **Dispute Report Not Found** | Thrown when referenced dispute ticket ID does not exist. |
| **`ADM_005`** | `400 BAD_REQUEST` | Dispute | **Dispute Already Resolved** | Thrown when attempting to issue a verdict on a closed dispute. |
| **`SYS_001`** | `500 INTERNAL_SERVER_ERROR` | System | **Internal Server Error** | Unexpected administrative processing failure. |
| **`SYS_002`** | `400 BAD_REQUEST` | System | **Validation Error** | Request parameter validation failure. |
| **`SYS_007`** | `403 FORBIDDEN` | Security | **Admin Privileges Required** | Non-admin user attempted to access administrative routes. |
