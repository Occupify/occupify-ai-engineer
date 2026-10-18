# Occupify Administration - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8185` (Internal)
> - Gateway Port: `8180` (External entry point)
> - Gateway Routes: `/admin/**`, `/audit-logs/**`, `/reports/**`
> - Database Port: PostgreSQL `5534` (`occupify_administration`)
> - Authorization: Strictly restricted to `X-User-Role: ADMIN`. Public and standard `USER` requests are blocked by Gateway with `403 FORBIDDEN` (`SYS_007`).
> - Status: *Scaffolding / Architecture Specification Phase*

---

## 1. Domain Responsibilities

The **Administration Service** (`occupify-administration`) provides governance, dispute mediation, compliance tracking, and platform auditing:
1. **User Moderation (`/admin/users/**`):** Suspending, banning, or reinstating abusive accounts.
2. **Audit Logging (`/audit-logs/**`):** Centralized immutable audit trails for sensitive events (login attempts, financial releases, contract terminations, admin actions).
3. **Dispute Resolution & Reports (`/reports/**`):** Processing user dispute complaints against contracts, freelancers, or clients.
4. **Platform Analytics (`/admin/metrics`):** Aggregated metrics on gross marketplace volume, active jobs, registration trends.

---

## 2. Planned API Endpoints

### 2.1 User Moderation (`/admin/users`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/admin/users` | Admin (`@Authenticated`) | Retrieves paginated list of all platform users with filter by status (`ACTIVE`, `LOCKED`, `BANNED`). |
| `PATCH`| `/admin/users/{id}/status` | Admin | Updates user account status (e.g. lock or ban an account). |

### 2.2 Audit Logs (`/audit-logs`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/audit-logs` | Admin | Queries immutable system audit trail with filters on timestamp, actor UUID, action, and target resource. |
| `GET` | `/audit-logs/{id}` | Admin | Retrieves specific audit log event details with full before/after snapshot payloads. |

### 2.3 Dispute Mediation & Reports (`/reports`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/reports` | Admin | Lists active dispute tickets and user report claims. |
| `GET` | `/reports/{id}` | Admin | Inspects dispute evidence and message history between disputing parties. |
| `POST`| `/reports/{id}/resolve` | Admin | Issues an administrative verdict on a dispute (refund, release, or dismissal). |
