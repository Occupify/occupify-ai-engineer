# Occupify Notification - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8184` (Internal)
> - Gateway Port: `8180` (External entry point)
> - Gateway Routes: `/notifications/**`
> - Database Port: PostgreSQL `5533` (`occupify_noti`), RabbitMQ `5672`, SMTP `587`

---

## 1. Get Notifications
- **Endpoint:** `GET /notifications`
- **Description:** Retrieves a paginated list of user notifications, ordered by creation date descending.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Query Parameters:**
  - `page` (Integer, default `0`): Page index.
  - `limit` (Integer, default `10`): Number of notifications per page.
- **Response Data (`PageResponse<NotificationResponse>`):**
  - `items`: List of notifications (`id`, `userId`, `title`, `message`, `type`, `isRead`, `createdAt`).
  - `pageNumber`, `pageSize`, `totalElements`, `totalPages`.

---

## 2. Get Unread Notification Count
- **Endpoint:** `GET /notifications/unread-count`
- **Description:** Returns the total count of unread notifications for the authenticated user (used for navbar badge display).
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Response Data (`SingleResponse<UnreadCountResponse>`):**
  - `userId` (UUID): User ID.
  - `unreadCount` (Long): Total unread count.

---

## 3. Mark Single Notification as Read
- **Endpoint:** `PUT /notifications/{notificationId}/read`
- **Description:** Marks a specific notification as read. Fails with `NOTI_002` if the notification belongs to another user.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Path Parameters:**
  - `notificationId` (UUID, required): Unique ID of the notification.
- **Response:** `200 OK` (`SuccessResponse`).

---

## 4. Mark All Notifications as Read
- **Endpoint:** `PUT /notifications/read-all`
- **Description:** Marks all pending notifications for the authenticated user as read in bulk.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Response:** `200 OK` (`SuccessResponse`).

---

## 5. Delete Notification
- **Endpoint:** `DELETE /notifications/{notificationId}`
- **Description:** Deletes a specific notification from user's history.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Path Parameters:**
  - `notificationId` (UUID, required): Unique ID of the notification.
- **Response:** `200 OK` (`SuccessResponse`).

---

## 6. Asynchronous Event Consumers (RabbitMQ)

The Notification service listens on exchange `occupify.notification.exchange`:

| Queue | Routing Key | Handling Logic |
| :--- | :--- | :--- |
| `occupify.notification.user.queue` | `user.registered` | Renders HTML email template with OTP and dispatches verification email via SMTP. |
| `occupify.notification.user.queue` | `user.password-reset` | Renders password reset email template with OTP and dispatches email via SMTP. |
| `occupify.notification.queue` | `project.created` | Creates in-app notifications for matching candidates about new project openings. |
| `occupify.notification.queue` | `project.closed` | Informs applicants and contracted freelancers about project closure. |
