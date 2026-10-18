# Occupify Identity Error Codes Specification

This document details all business and system error codes thrown by `occupify-identity`.

## Error Codes Reference Table

| Error Code | HTTP Status | Category | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **`AUTH_000`** | `400 BAD_REQUEST` | Authentication | **Invalid Email Format** | Thrown when email format does not adhere to standard RFC pattern. |
| **`AUTH_001`** | `409 CONFLICT` | Authentication | **Email Already Exists** | Thrown when attempting to register with an email already taken. |
| **`AUTH_002`** | `400 BAD_REQUEST` | Authentication | **Password Too Short** | Thrown when password is less than 8 characters long. |
| **`AUTH_003`** | `401 UNAUTHORIZED` | Authentication | **Invalid Credentials** | Thrown when email or password does not match database record. |
| **`AUTH_004`** | `400 BAD_REQUEST` | Authentication | **Refresh Token Required** | Thrown when refresh token request is missing the token body. |
| **`AUTH_005`** | `401 UNAUTHORIZED` | Authentication | **Invalid Refresh Token** | Thrown when token signature or payload is invalid. |
| **`AUTH_006`** | `401 UNAUTHORIZED` | Authentication | **Expired / Revoked Token** | Thrown when refresh token expired or was revoked during sign-out. |
| **`AUTH_007`** | `429 TOO_MANY_REQUESTS` | Rate Limiting | **OTP Rate Limit Exceeded** | Thrown when requesting OTP resend before cooldown window expires. |
| **`AUTH_008`** | `400 BAD_REQUEST` | Authentication | **OTP Expired or Missing** | Thrown when OTP code does not exist in Redis or expired. |
| **`AUTH_009`** | `400 BAD_REQUEST` | Authentication | **Invalid OTP Code** | Thrown when provided OTP code does not match active OTP. |
| **`AUTH_010`** | `429 TOO_MANY_REQUESTS` | Rate Limiting | **Max OTP Attempts Exceeded** | Thrown when user exceeds maximum allowed failed OTP verification attempts. |
| **`AUTH_011`** | `500 INTERNAL_SERVER_ERROR` | System | **Notification Dispatch Failure** | Thrown when RabbitMQ message broker fails to queue notification event. |
| **`AUTH_012`** | `404 NOT_FOUND` | User Account | **User Not Found** | Thrown when email address is not found in user repository. |
| **`AUTH_013`** | `400 BAD_REQUEST` | Authentication | **Current Password Mismatch** | Thrown during password change when existing password is incorrect. |
| **`AUTH_014`** | `400 BAD_REQUEST` | Authentication | **Identical Password** | Thrown when new password is identical to previous password. |
| **`AUTH_015`** | `400 BAD_REQUEST` | Authentication | **Account Already Activated** | Thrown when attempting to verify OTP on an already active account. |
| **`AUTH_016`** | `400 BAD_REQUEST` | Authentication | **Invalid Reset Token** | Thrown when reset token is invalid or expired. |
| **`AUTH_017`** | `400 BAD_REQUEST` | Authentication | **Password Confirmation Mismatch**| Thrown when confirmation password does not match new password. |
| **`USER_001`** | `404 NOT_FOUND` | User Account | **User Account Not Found** | Thrown when user UUID lookup yields no database record. |
| **`USER_002`** | `410 GONE` | User Account | **User Account Deleted** | Thrown when attempting operations on a soft-deleted user. |
| **`USER_007`** | `403 FORBIDDEN` | User Account | **User Inactive or Locked** | Thrown when user account has not yet verified OTP. |
| **`USER_008`** | `403 FORBIDDEN` | User Account | **User Account Banned** | Thrown when user account is permanently suspended by admin. |
| **`EXP_404`** | `404 NOT_FOUND` | User Profile | **Experience Record Not Found** | Thrown when experience ID lookup fails. |
| **`EDU_404`** | `404 NOT_FOUND` | User Profile | **Education Record Not Found** | Thrown when education ID lookup fails. |
| **`INST_404`** | `404 NOT_FOUND` | Catalog | **Institution Not Found** | Thrown when referencing non-existent educational institution. |
| **`QUAL_404`** | `404 NOT_FOUND` | User Profile | **Qualification Not Found** | Thrown when qualification ID lookup fails. |
| **`SKILL_404`**| `404 NOT_FOUND` | Catalog | **Skill Not Found** | Thrown when referencing a skill ID not in canonical catalog. |
| **`USKILL_404`**| `404 NOT_FOUND` | User Profile | **User Skill Not Found** | Thrown when attempting to update or delete a non-associated skill. |
| **`USKILL_409`**| `409 CONFLICT` | User Profile | **Skill Already Added** | Thrown when attempting to add a duplicate skill to profile. |
| **`DATE_400`** | `400 BAD_REQUEST` | User Profile | **Invalid Date Range** | Thrown when end date is earlier than start date. |
| **`STORAGE_001`**| `400 BAD_REQUEST`| Object Storage | **File Empty or Missing** | Thrown when uploaded file contains 0 bytes. |
| **`STORAGE_002`**| `400 BAD_REQUEST`| Object Storage | **Invalid Image Type** | Thrown when avatar is not JPEG, PNG, WEBP, or GIF. |
| **`STORAGE_003`**| `400 BAD_REQUEST`| Object Storage | **File Too Large** | Thrown when avatar exceeds 5MB limit. |
| **`STORAGE_004`**| `500 INTERNAL_SERVER_ERROR`| Object Storage | **Upload Failed** | Thrown on object storage write failure. |
| **`STORAGE_005`**| `500 INTERNAL_SERVER_ERROR`| Object Storage | **Delete Failed** | Thrown on object storage delete failure. |
| **`STORAGE_006`**| `400 BAD_REQUEST`| Object Storage | **Invalid Document Type** | Thrown when proof document is not PDF, JPEG, PNG, or WEBP. |
| **`STORAGE_007`**| `400 BAD_REQUEST`| Object Storage | **Document Too Large** | Thrown when document exceeds allowed upload size limit. |
| **`SYS_001`** | `500 INTERNAL_SERVER_ERROR` | System | **Internal Server Error** | Unexpected server failure. |
| **`SYS_002`** | `400 BAD_REQUEST` | System | **Validation Error** | Request DTO validation failure. |
| **`SYS_003`** | `500 INTERNAL_SERVER_ERROR` | System | **Database Error** | Low-level database constraint or connectivity error. |
| **`SYS_004`** | `404 NOT_FOUND` | System | **Resource Not Found** | Generic resource lookup failure. |
| **`SYS_005`** | `405 METHOD_NOT_ALLOWED` | System | **Method Not Allowed** | Unsupported HTTP method. |
| **`SYS_006`** | `401 UNAUTHORIZED` | System | **Unauthorized Access** | Authentication required but not supplied. |
| **`SYS_007`** | `403 FORBIDDEN` | System | **Access Denied** | Insufficient role permissions. |
