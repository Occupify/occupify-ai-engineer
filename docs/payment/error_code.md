# Occupify Payment Error Codes Specification

This document details all business and system error codes reserved for `occupify-payment`.

## Error Codes Reference Table

| Error Code | HTTP Status | Category | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **`PAY_001`** | `404 NOT_FOUND` | Contract | **Contract Not Found** | Thrown when referenced contract ID does not exist. |
| **`PAY_002`** | `403 FORBIDDEN` | Contract | **Contract Access Forbidden** | Thrown when user is not a party (client or freelancer) to the contract. |
| **`PAY_003`** | `400 BAD_REQUEST` | Contract | **Illegal Contract State** | Thrown when attempting an illegal action (e.g. signing an already completed contract). |
| **`PAY_004`** | `400 BAD_REQUEST` | Escrow | **Insufficient Wallet Balance** | Thrown when client account lacks funds to escrow milestone. |
| **`PAY_005`** | `400 BAD_REQUEST` | Escrow | **Milestone Not Releasable** | Thrown when attempting to release escrow on an incomplete milestone. |
| **`PAY_006`** | `404 NOT_FOUND` | Wallet | **Wallet Not Found** | Thrown when wallet record does not exist for user. |
| **`PAY_007`** | `400 BAD_REQUEST` | Wallet | **Invalid Withdrawal Amount** | Thrown when withdrawal exceeds available balance or below minimum threshold. |
| **`PAY_008`** | `400 BAD_REQUEST` | Webhook | **Invalid Webhook Signature** | Thrown when payment provider callback HMAC signature verification fails. |
| **`SYS_001`** | `500 INTERNAL_SERVER_ERROR` | System | **Internal Server Error** | Unexpected payment processing failure. |
| **`SYS_002`** | `400 BAD_REQUEST` | System | **Validation Error** | Request parameter validation failure. |
