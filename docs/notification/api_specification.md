# Occupify Notification - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8184` (Internal)
> - Gateway Port: `8180` (External ingress entry point)
> - Gateway Routes: `/notifications/**`
> - Database Port: PostgreSQL `5533` (`occupify_noti`), RabbitMQ `5672`, SMTP `587`

---

## Standard Response Envelopes

Every API endpoint returns a standardized response envelope:

### Single Data Response (`SingleResponse<T>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
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
    "limit": 10,
    "total": 45,
    "totalPages": 5
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
  "statusCode": 404,
  "errorCode": "NOTI_001",
  "message": "Notification not found",
  "timestamp": "2026-10-18T22:30:00Z"
}
```

---

## 1. Notification Management Endpoints (`/notifications`)

All endpoints require authentication via `Authorization: Bearer <jwt_access_token>` or Gateway header injection (`X-User-Id: <uuid>`).

### 1.1 Get Notifications (Paginated)
- **Endpoint:** `GET /notifications`
- **Description:** Retrieves a paginated list of notifications for the authenticated user, ordered by creation date descending.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Headers:**
  | Name | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `Authorization` | String | Conditional | `Bearer <JWT>` |
  | `X-User-Id` | UUID | Conditional | Injected by Gateway |

#### Query Parameters
| Parameter | Type | Required | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `page` | Integer | No | `0` | Zero-based page index (must be $\ge 0$) |
| `limit` | Integer | No | `10` | Number of items per page |

#### Response Data (`PageResponse<NotificationResponse>`)
| Field | Type | Description |
| :--- | :--- | :--- |
| `data` | Array[`NotificationResponse`] | List of notifications on the current page |
| `data[].id` | UUID | Notification unique identifier |
| `data[].userId` | UUID | Recipient user ID |
| `data[].title` | String | Notification title |
| `data[].content` | String | Rendered notification message body |
| `data[].category` | String (Enum) | Category: `JOB`, `SYSTEM`, `MESSAGE`, `PAYMENT` |
| `data[].actionType` | String (Enum) | Template type (`PROPOSAL_VIEWED`, `PROPOSAL_SUBMITTED`, etc.) |
| `data[].isRead` | Boolean | Read status (`true` / `false`) |
| `data[].readAt` | String (DateTime) | Timestamp when marked as read, or null |
| `data[].createdAt` | String (DateTime) | Creation timestamp |
| `paging` | Object | Pagination metadata (`page`, `limit`, `total`, `totalPages`) |

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Notifications retrieved successfully",
  "data": [
    {
      "id": "e8d1a3c5-6789-4def-0123-456789abcdef",
      "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
      "title": "Đề xuất đã được xem",
      "content": "Nhà tuyển dụng TechCorp Global đã xem đề xuất của bạn cho công việc 'Senior Backend Developer'.",
      "category": "JOB",
      "actionType": "PROPOSAL_VIEWED",
      "isRead": false,
      "readAt": null,
      "createdAt": "2026-10-18T20:15:00"
    },
    {
      "id": "f9e2b4d6-7890-5ef0-1234-567890bcdefa",
      "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
      "title": "Đề xuất ứng tuyển mới",
      "content": "Ứng viên Alex Mercer vừa gửi đề xuất cho công việc 'Cloud Architect'.",
      "category": "JOB",
      "actionType": "PROPOSAL_SUBMITTED",
      "isRead": true,
      "readAt": "2026-10-18T19:00:00",
      "createdAt": "2026-10-18T18:45:00"
    }
  ],
  "paging": {
    "page": 0,
    "limit": 10,
    "total": 2,
    "totalPages": 1
  }
}
```

---

### 1.2 Get Unread Notification Count
- **Endpoint:** `GET /notifications/unread-count`
- **Description:** Returns the total count of unread notifications for the authenticated user (used for navbar badge display).
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Headers:** `Authorization: Bearer <token>` or `X-User-Id`

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Unread count retrieved successfully",
  "data": {
    "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
    "unreadCount": 5
  }
}
```

---

### 1.3 Mark Single Notification as Read
- **Endpoint:** `PUT /notifications/{notificationId}/read`
- **Description:** Marks a single notification as read. Returns `404` (`NOTI_001`) if not found or if the notification belongs to another user.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Path Parameter:** `notificationId` (UUID, required)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Notification marked as read"
}
```

---

### 1.4 Mark All Notifications as Read
- **Endpoint:** `PUT /notifications/read-all`
- **Description:** Marks all pending unread notifications for the authenticated user as read in bulk.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "All notifications marked as read"
}
```

---

### 1.5 Delete Notification
- **Endpoint:** `DELETE /notifications/{notificationId}`
- **Description:** Permanently deletes a specific notification from user's notification history.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Path Parameter:** `notificationId` (UUID, required)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Notification deleted successfully"
}
```

---

## 2. Asynchronous Event Messaging & AMQP Contracts (RabbitMQ)

`occupify-noti` consumes domain and notification events across two core exchanges:
1. `occupify.notification.exchange`: System lifecycle and transactional notification events.
2. `occupify.user.exchange`: User lifecycle events (e.g. account activation and profile synchronization).

```
[occupify-identity] --------(user.registered)---------> [occupify.notification.user.queue] (Send OTP Email)
[occupify-identity] --------(user.activated)----------> [occupify.notification.user.queue] (Sync User Record)
[occupify-identity] --------(user.password-reset)-----> [occupify.notification.user.queue] (Send Reset Email)
[occupify-core] ------------(notification.*)----------> [occupify.notification.queue]      (In-App Notification)
```

### 2.1 User Registered Event (`UserRegisteredEvent`)
- **Exchange:** `occupify.notification.exchange`
- **Routing Key:** `user.registered`
- **Queue:** `occupify.notification.user.queue`
- **Action:** Renders HTML template `templates/email/registration-otp.html` and dispatches registration OTP email via SMTP.

#### Payload Example (JSON)
```json
{
  "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "email": "candidate@occupify.work",
  "otpCode": "123456",
  "occurredAt": "2026-10-18T10:00:00Z"
}
```

---

### 2.2 User Activated Event (`UserActivatedEvent`)
- **Exchange:** `occupify.user.exchange`
- **Routing Key:** `user.activated`
- **Queue:** `occupify.notification.user.queue`
- **Action:** Synchronizes the user account record into the `occupify_noti` database (`users` table).

#### Payload Example (JSON)
```json
{
  "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "email": "candidate@occupify.work",
  "fullName": "Alex Mercer",
  "role": "USER",
  "status": "ACTIVE",
  "occurredAt": "2026-10-18T10:05:00Z"
}
```

---

### 2.3 Password Reset Requested Event (`PasswordResetRequestedEvent`)
- **Exchange:** `occupify.notification.exchange`
- **Routing Key:** `user.password-reset`
- **Queue:** `occupify.notification.user.queue`
- **Action:** Renders HTML template `templates/email/password-reset.html` and dispatches password reset OTP email via SMTP.

#### Payload Example (JSON)
```json
{
  "userId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "email": "candidate@occupify.work",
  "otpCode": "654321",
  "occurredAt": "2026-10-18T10:10:00Z"
}
```

---

### 2.4 In-App Notification Event (`NotificationEventPayload`)
- **Exchange:** `occupify.notification.exchange`
- **Routing Key:** `notification.#`
- **Queue:** `occupify.notification.queue`
- **Action:** Persists new notification in database and broadcasts via SSE stream.

#### Supported Templates & Actions
| Action Type | Category | Default Title | Content Format |
| :--- | :--- | :--- | :--- |
| `PROPOSAL_VIEWED` | `JOB` | Đề xuất đã được xem | `Nhà tuyển dụng %s đã xem đề xuất của bạn cho công việc '%s'.` |
| `PROPOSAL_SUBMITTED`| `JOB` | Đề xuất ứng tuyển mới | `Ứng viên %s vừa gửi đề xuất cho công việc '%s'.` |
| `PROPOSAL_ACCEPTED` | `JOB` | Đề xuất được chấp nhận | `Chúc mừng! Đề xuất cho công việc '%s' đã được chấp nhận bởi %s.` |
| `POST_LIKED` | `COMMUNITY` | Lượt thích mới | `%s đã thích bài viết '%s' của bạn.` |
| `POST_COMMENTED` | `COMMUNITY` | Bình luận mới | `%s đã bình luận về bài viết '%s': \"%s\".` |
| `POST_SHARED` | `COMMUNITY` | Lượt chia sẻ mới | `%s đã chia sẻ bài viết '%s' của bạn.` |
| `JOB_MATCH_POSTED` | `JOB` | Công việc phù hợp mới | `Công việc mới '%s' phù hợp với kỹ năng của bạn (Ngân sách: %s).` |

#### Payload Example (JSON)
```json
{
  "eventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipientUserId": "c1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "recipientEmail": "candidate@occupify.work",
  "recipientFullName": "Alex Mercer",
  "recipientAvatarUrl": "https://storage.occupify.work/avatars/c1a2b3c4-avatar.jpg",
  "category": "JOB",
  "actionType": "PROPOSAL_VIEWED",
  "customTitle": null,
  "customContent": null,
  "args": [
    "TechCorp Global",
    "Senior Backend Developer"
  ],
  "metadata": {
    "proposalId": "77777777-8888-9999-aaaa-bbbbbbbbbbbb",
    "jobId": "88888888-9999-aaaa-bbbb-cccccccccccc"
  },
  "occurredAt": "2026-10-18T20:15:00Z"
}
```
