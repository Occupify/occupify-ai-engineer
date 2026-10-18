# Occupify Notification Error Codes Specification

This document details all business and system error codes thrown by `occupify-noti`.

## Error Codes Reference Table

| Error Code | HTTP Status | Exception Class | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **`NOTI_001`** | `404 NOT_FOUND` | `NotificationException` | **Notification Not Found** | Thrown when attempting to access or mutate a notification ID that does not exist. |
| **`NOTI_002`** | `403 FORBIDDEN` | `NotificationException` | **Forbidden Access** | Thrown when an authenticated user attempts to read, mark, or delete a notification belonging to another user. |
| **`NOTI_003`** | `404 NOT_FOUND` | `NotificationException` | **User Not Found** | Thrown when retrieving notification preferences or records for a non-existent user. |
| **`NOTI_004`** | `400 BAD_REQUEST` | `NotificationException` | **Invalid Notification Payload** | Thrown when incoming notification message payload fails schema or content validation. |
| **`SYS_001`** | `500 INTERNAL_SERVER_ERROR` | `SystemException` / `Exception` | **Internal System Error** | Unhandled server error or mail server connectivity timeout. |
| **`SYS_002`** | `400 BAD_REQUEST` | `MethodArgumentNotValidException` | **Request Validation Error** | Request parameter validation failure. |
