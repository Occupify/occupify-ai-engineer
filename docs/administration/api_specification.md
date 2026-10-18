# Occupify Administration - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8185` (Internal)
> - Gateway Port: `8180` (External ingress entry point)
> - Gateway Routes: `/admin/**`, `/audit-logs/**`, `/reports/**`
> - Database Port: PostgreSQL `5534` (`occupify_administration`), RabbitMQ `5672`
> - Authorization: Strictly restricted to `X-User-Role: ADMIN`. Public and standard `USER` requests are blocked by Gateway with `403 FORBIDDEN` (`SYS_007` / `USER_007`).
> - Status: *Architecture & API Contract Specification Phase*

---

## Standard Response Envelopes

Every API endpoint adheres to standardized Occupify response envelopes:

### Single Data Response (`SingleResponse<T>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": { ... }
}
```

### Resource Created Response (`CreatedResponse<T>`)
```json
{
  "statusCode": 201,
  "message": "Resource created successfully",
  "data": { ... }
}
```

### Paginated Data Response (`PageResponse<T>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": [ ... ],
  "paging": {
    "page": 0,
    "limit": 20,
    "total": 142,
    "totalPages": 8
  }
}
```

### Generic Action Success Response (`SuccessResponse`)
```json
{
  "statusCode": 200,
  "message": "Action completed successfully"
}
```

### Error Response (`ErrorResponse`)
```json
{
  "statusCode": 403,
  "errorCode": "SYS_007",
  "message": "Access denied: ADMIN role required",
  "timestamp": "2026-10-18T22:30:00Z"
}
```

---

## 1. Domain Responsibilities

The **Administration Service** (`occupify-administration`) provides administrative controls, compliance governance, audit trail verification, and dispute resolution:
1. **User Moderation (`/admin/users/**`):** Account lifecycle controls (locking, suspending, unbanning), review of flagged users, and administrative override.
2. **Platform Audit Trail (`/audit-logs/**`):** Immutable append-only event ledger capturing security-critical mutations, financial transactions, and administrative interventions.
3. **Dispute Mediation (`/reports/**`):** Handling submitted grievance claims between clients and freelancers on escrow contracts, evidence arbitration, and binding verdicts.
4. **Platform Analytics & Health (`/admin/metrics`):** Real-time aggregation of GMV (Gross Marketplace Volume), active contracts, registration velocity, and open disputes.

---

## 2. API Endpoints Specification

### 2.1 User Moderation (`/admin/users`)

#### 2.1.1 List All Users (Paginated)
- **Endpoint:** `GET /admin/users`
- **Description:** Queries platform users with filtering by status and search terms.
- **Authorization:** Admin (`X-User-Role: ADMIN`)
- **Query Parameters:**
  - `page`: Integer (default: `0`)
  - `limit`: Integer (default: `20`)
  - `status`: Enum (`ACTIVE`, `LOCKED`, `BANNED`)
  - `keyword`: String (email, full name match)

##### Response Example (`200 OK` - `PageResponse<AdminUserResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": [
    {
      "id": "22222222-3333-4444-5555-666666666666",
      "email": "freelancer@example.com",
      "fullName": "Jane Doe",
      "role": "USER",
      "status": "ACTIVE",
      "registeredAt": "2026-09-01T10:00:00Z",
      "lastLoginAt": "2026-10-18T18:30:00Z",
      "totalCompletedContracts": 12,
      "disputeCount": 0
    }
  ],
  "paging": {
    "page": 0,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

#### 2.1.2 Update User Account Status
- **Endpoint:** `PATCH /admin/users/{id}/status`
- **Description:** Suspends, bans, or restores an account. Emits `user.status.updated` event to invalidate sessions.
- **Authorization:** Admin (`X-User-Role: ADMIN`)
- **Headers:** `Content-Type: application/json`, `X-User-Id: <admin-uuid>`

##### Request Body
| Field | Type | Required | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| `status` | String (Enum) | Yes | `ACTIVE`, `LOCKED`, `BANNED` | New account state |
| `reason` | String | Yes | Max 500 chars, not blank | Justification for audit log |

##### Request Body Example
```json
{
  "status": "LOCKED",
  "reason": "Multiple reports of fraudulent contract delivery submitted by employers."
}
```

##### Response Example (`200 OK` - `SingleResponse<AdminUserStatusResponse>`)
```json
{
  "statusCode": 200,
  "message": "User status updated successfully",
  "data": {
    "userId": "22222222-3333-4444-5555-666666666666",
    "previousStatus": "ACTIVE",
    "newStatus": "LOCKED",
    "reason": "Multiple reports of fraudulent contract delivery submitted by employers.",
    "moderatedByAdminId": "99999999-0000-1111-2222-333333333333",
    "moderatedAt": "2026-10-19T09:00:00Z"
  }
}
```

---

### 2.2 Audit Trail (`/audit-logs`)

#### 2.2.1 Query Audit Logs
- **Endpoint:** `GET /audit-logs`
- **Description:** Queries the immutable audit trail for security events, financial operations, and configuration modifications.
- **Authorization:** Admin (`X-User-Role: ADMIN`)
- **Query Parameters:**
  - `page`: Integer (default: `0`)
  - `limit`: Integer (default: `20`)
  - `actorId`: UUID (optional actor filter)
  - `action`: String (e.g. `USER_LOCKED`, `ESCROW_RELEASED`, `DISPUTE_RESOLVED`)
  - `resourceType`: String (`USER`, `CONTRACT`, `PAYMENT`, `PROJECT`)

##### Response Example (`200 OK` - `PageResponse<AuditLogResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": [
    {
      "id": "aaaa1111-2222-3333-4444-555566667777",
      "actorId": "99999999-0000-1111-2222-333333333333",
      "actorEmail": "admin@occupify.app",
      "action": "USER_LOCKED",
      "resourceType": "USER",
      "resourceId": "22222222-3333-4444-5555-666666666666",
      "ipAddress": "192.168.1.50",
      "userAgent": "Mozilla/5.0 (X11; Linux x86_64)",
      "timestamp": "2026-10-19T09:00:00Z"
    }
  ],
  "paging": {
    "page": 0,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

#### 2.2.2 Get Audit Log Details
- **Endpoint:** `GET /audit-logs/{id}`
- **Description:** Retrieves the full audit record including before/after state diff snapshots.
- **Authorization:** Admin (`X-User-Role: ADMIN`)

##### Response Example (`200 OK` - `SingleResponse<AuditLogDetailResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": {
    "id": "aaaa1111-2222-3333-4444-555566667777",
    "actorId": "99999999-0000-1111-2222-333333333333",
    "actorEmail": "admin@occupify.app",
    "action": "USER_LOCKED",
    "resourceType": "USER",
    "resourceId": "22222222-3333-4444-5555-666666666666",
    "beforeSnapshot": {
      "status": "ACTIVE",
      "lockedReason": null
    },
    "afterSnapshot": {
      "status": "LOCKED",
      "lockedReason": "Multiple reports of fraudulent contract delivery submitted by employers."
    },
    "ipAddress": "192.168.1.50",
    "timestamp": "2026-10-19T09:00:00Z"
  }
}
```

---

### 2.3 Dispute Mediation & Reports (`/reports`)

#### 2.3.1 List Dispute Reports
- **Endpoint:** `GET /reports`
- **Description:** Lists user dispute claims and grievance tickets filed across contracts.
- **Authorization:** Admin (`X-User-Role: ADMIN`)
- **Query Parameters:**
  - `page`: Integer (default: `0`)
  - `limit`: Integer (default: `20`)
  - `status`: Enum (`OPEN`, `UNDER_REVIEW`, `RESOLVED`, `DISMISSED`)

##### Response Example (`200 OK` - `PageResponse<DisputeReportResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": [
    {
      "id": "cccc3333-4444-5555-6666-777788889999",
      "contractId": "77777777-8888-9999-aaaa-bbbbbbbbbbbb",
      "reporterId": "33333333-4444-5555-6666-777777777777",
      "reportedUserId": "22222222-3333-4444-5555-666666666666",
      "reason": "DELIVERABLE_NON_COMPLIANT",
      "status": "OPEN",
      "disputedAmount": 1000.00,
      "currency": "USD",
      "createdAt": "2026-10-19T08:00:00Z"
    }
  ],
  "paging": {
    "page": 0,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

#### 2.3.2 Get Dispute Report Details
- **Endpoint:** `GET /reports/{id}`
- **Description:** Retrieves complete dispute ticket with submission evidence links and chronological mediation timeline.
- **Authorization:** Admin (`X-User-Role: ADMIN`)

##### Response Example (`200 OK` - `SingleResponse<DisputeReportDetailResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": {
    "id": "cccc3333-4444-5555-6666-777788889999",
    "contractId": "77777777-8888-9999-aaaa-bbbbbbbbbbbb",
    "reporterId": "33333333-4444-5555-6666-777777777777",
    "reportedUserId": "22222222-3333-4444-5555-666666666666",
    "reason": "DELIVERABLE_NON_COMPLIANT",
    "description": "Freelancer failed to deliver RabbitMQ circuit breaker fallback and did not attend sprint review.",
    "evidenceUrls": [
      "https://occupify-storage.s3.ap-southeast-1.amazonaws.com/evidence/contract-7777/pr_diff.pdf"
    ],
    "status": "OPEN",
    "disputedAmount": 1000.00,
    "currency": "USD",
    "createdAt": "2026-10-19T08:00:00Z"
  }
}
```

---

#### 2.3.3 Resolve Dispute Report
- **Endpoint:** `POST /reports/{id}/resolve`
- **Description:** Issues an administrative binding decision on a dispute ticket. Automatically commands payment escrow disbursal according to verdict.
- **Authorization:** Admin (`X-User-Role: ADMIN`)
- **Headers:** `Content-Type: application/json`, `X-User-Id: <admin-uuid>`

##### Request Body
| Field | Type | Required | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| `verdict` | String (Enum) | Yes | `REFUND_CLIENT`, `RELEASE_FREELANCER`, `SPLIT_FUNDS`, `DISMISSED` | Binding decision |
| `adminNotes` | String | Yes | Max 2000 chars, not blank | Detailed explanation |
| `clientRefundAmount` | BigDecimal | Conditional | Required if `SPLIT_FUNDS` or `REFUND_CLIENT` | Escrow amount to return to client |
| `freelancerPayoutAmount` | BigDecimal | Conditional | Required if `SPLIT_FUNDS` or `RELEASE_FREELANCER` | Escrow amount to pay to freelancer |

##### Request Body Example
```json
{
  "verdict": "SPLIT_FUNDS",
  "adminNotes": "Milestone 1 work completed satisfactorily, but Milestone 2 was partially missing. Disputed escrow split 50/50.",
  "clientRefundAmount": 500.00,
  "freelancerPayoutAmount": 500.00
}
```

##### Response Example (`200 OK` - `SingleResponse<DisputeResolutionResponse>`)
```json
{
  "statusCode": 200,
  "message": "Dispute resolved successfully and escrow disbursement triggered",
  "data": {
    "reportId": "cccc3333-4444-5555-6666-777788889999",
    "contractId": "77777777-8888-9999-aaaa-bbbbbbbbbbbb",
    "status": "RESOLVED",
    "verdict": "SPLIT_FUNDS",
    "clientRefundAmount": 500.00,
    "freelancerPayoutAmount": 500.00,
    "resolvedByAdminId": "99999999-0000-1111-2222-333333333333",
    "resolvedAt": "2026-10-19T11:00:00Z"
  }
}
```

---

### 2.4 Platform Metrics (`/admin/metrics`)

#### 2.4.1 Get Platform Metrics
- **Endpoint:** `GET /admin/metrics`
- **Description:** Aggregates macro KPIs across user registrations, contracts, escrow volumes, and open tickets.
- **Authorization:** Admin (`X-User-Role: ADMIN`)

##### Response Example (`200 OK` - `SingleResponse<PlatformMetricsResponse>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": {
    "totalUsers": 12450,
    "activeUsers": 8320,
    "totalContracts": 3200,
    "activeContracts": 412,
    "grossMarketplaceVolume": 1285400.00,
    "currency": "USD",
    "openDisputes": 4,
    "calculatedAt": "2026-10-19T11:00:00Z"
  }
}
```
