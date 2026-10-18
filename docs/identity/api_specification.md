# Occupify Identity - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8181` (Internal)
> - Gateway Port: `8180` (External entry point)
> - Gateway Routes: `/auth/**`, `/account/**`, `/users/**`, `/user/**`
> - Database Port: PostgreSQL `5530` (`occupify_identity`), Redis `6379`, RabbitMQ `5672`

---

## 1. Authentication Endpoints (`/auth`)

### 1.1 Register Account
- **Endpoint:** `POST /auth/register`
- **Description:** Registers a new user account with email and password. Generates a registration OTP and publishes `UserRegisteredEvent` to RabbitMQ (`occupify.notification.exchange` with routing key `user.registered`).
- **Authorization:** Public
- **Request Body (`application/json`):**
  - `email` (String, required): Valid email address.
  - `password` (String, required): Minimum 8 characters.
  - `fullName` (String, optional): Full display name.
- **Response:** `201 CREATED` with message "Registration initiated. Verification OTP dispatched."

### 1.2 Resend Registration OTP
- **Endpoint:** `POST /auth/resend-otp`
- **Description:** Generates and dispatches a fresh OTP if previous one expired or was not received. Rate-limited to prevent abuse (`AUTH_007`).
- **Authorization:** Public
- **Request Body:** `{ "email": "user@example.com" }`

### 1.3 Verify Registration OTP
- **Endpoint:** `POST /auth/verify-otp`
- **Description:** Validates 6-digit verification OTP and activates the user account (`status = ACTIVE`).
- **Authorization:** Public
- **Request Body:** `{ "email": "user@example.com", "otpCode": "123456" }`

### 1.4 User Login
- **Endpoint:** `POST /auth/login`
- **Description:** Authenticates user via email and password. Returns JWT access token and refresh token.
- **Authorization:** Public
- **Request Body:** `{ "email": "user@example.com", "password": "securePassword123" }`
- **Response Body (`SingleResponse<AuthResponse>`):**
  - `accessToken` (String): HS512 JWT access token (valid 1 hour).
  - `refreshToken` (String): Long-lived refresh token (valid 7 days).
  - `user` (Object): User profile preview (`id`, `email`, `role`, `fullName`, `avatarUrl`).

### 1.5 Refresh Access Token
- **Endpoint:** `POST /auth/refresh-token`
- **Description:** Issues a new JWT access token using a valid, non-revoked refresh token.
- **Authorization:** Public
- **Request Body:** `{ "refreshToken": "..." }`

### 1.6 Sign Out / Revoke Session
- **Endpoint:** `POST /auth/sign-out`
- **Description:** Revokes active refresh token and blacklists active JWT session in Redis.
- **Authorization:** Authenticated
- **Request Body:** `{ "refreshToken": "..." }`

### 1.7 Forgot Password Request
- **Endpoint:** `POST /auth/forgot-password`
- **Description:** Dispatches a password reset OTP to user's registered email via RabbitMQ (`PasswordResetRequestedEvent`).
- **Authorization:** Public
- **Request Body:** `{ "email": "user@example.com" }`

### 1.8 Reset Password With OTP
- **Endpoint:** `POST /auth/reset-password`
- **Description:** Resets password using verification OTP.
- **Authorization:** Public
- **Request Body:** `{ "email": "...", "otpCode": "...", "newPassword": "...", "confirmPassword": "..." }`

### 1.9 Change Password
- **Endpoint:** `POST /auth/change-password`
- **Description:** Allows authenticated user to update their password by confirming their existing password.
- **Authorization:** Authenticated (`@Authenticated UUID userId`)
- **Request Body:** `{ "currentPassword": "...", "newPassword": "...", "confirmPassword": "..." }`

---

## 2. User Account Management (`/users`)

### 2.1 Get Current User Preview
- **Endpoint:** `GET /users/me`
- **Authorization:** Authenticated
- **Response:** Basic user identity details (`id`, `email`, `role`, `status`, `avatarUrl`).

### 2.2 Get Current User Full Profile
- **Endpoint:** `GET /users/me/detail`
- **Authorization:** Authenticated
- **Response:** Comprehensive user profile including work experiences, educations, certifications, and skills.

### 2.3 Get Public User Profile
- **Endpoint:** `GET /users/{id}/detail`
- **Authorization:** Public
- **Path Parameter:** `id` (UUID of user)
- **Response:** Public freelancer/client profile.

### 2.4 Update User Profile
- **Endpoint:** `PUT /users/me`
- **Authorization:** Authenticated
- **Request Body:** `{ "fullName": "...", "phoneNumber": "...", "bio": "...", "location": "..." }`

### 2.5 Delete User Account
- **Endpoint:** `DELETE /users/me`
- **Authorization:** Authenticated
- **Description:** Soft-deletes user account (`deleted_at` timestamp).

### 2.6 Upload Profile Avatar
- **Endpoint:** `POST /users/me/avatar`
- **Consumes:** `multipart/form-data`
- **Authorization:** Authenticated
- **Request Parameter:** `file` (Multipart file: JPEG, PNG, WEBP, GIF, max 5MB).

### 2.7 Delete Profile Avatar
- **Endpoint:** `DELETE /users/me/avatar`
- **Authorization:** Authenticated

---

## 3. Profile Sub-Resources (`/users/me/*`)

### 3.1 Work Experience
- `POST /users/me/experiences`: Add new experience (`title`, `company`, `location`, `startDate`, `endDate`, `description`).
- `PUT /users/me/experiences/{id}`: Update experience.
- `DELETE /users/me/experiences/{id}`: Remove experience.

### 3.2 Education Records
- `POST /users/me/educations` (`multipart/form-data`): Add education with degree, field of study, and optional proof document.
- `PUT /users/me/educations/{id}` (`multipart/form-data`): Update education record.
- `DELETE /users/me/educations/{id}`: Remove education record.

### 3.3 Qualifications / Certifications
- `POST /users/me/qualifications` (`multipart/form-data`): Add professional certificate or qualification.
- `PUT /users/me/qualifications/{id}` (`multipart/form-data`): Update qualification.
- `DELETE /users/me/qualifications/{id}`: Remove qualification.

### 3.4 User Skills
- `GET /users/me/skills`: Retrieve list of skills associated with user profile.
- `POST /users/me/skills`: Associate canonical skill from catalog (`skillId`, `proficiencyLevel`).
- `PUT /users/me/skills/{skillId}`: Update skill level.
- `DELETE /users/me/skills/{skillId}`: Remove skill association.

### 3.5 Educational Institutions Catalog
- `GET /user/institutions` or `GET /users/institutions`: Search recognized educational institutions.
