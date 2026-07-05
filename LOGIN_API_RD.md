# Login API RD

## 1. Purpose

The Login API authenticates an existing HMS user and returns tokens needed to access protected APIs. It must support login using username, phone number, or email address, and it must map the logged-in user to the correct organisation and role.

## 2. Scope

### In Scope

- Login with username, phone number, or email address.
- Password authentication for an existing active user.
- Organisation-aware login using the user's stored `orgId`.
- Role-aware login for HMS roles.
- Access token generation.
- Refresh token generation, storage, rotation, and cookie handling.
- Device-aware refresh token management.
- Failed login lockout using organisation-specific maximum attempts.
- Customer/patient self-service forgot-password flow using email OTP and new password confirmation.
- Admin-managed login policy configuration.
- Development and testing checklist for login behavior.

### Out of Scope

- Creating a new user during login.
- OTP login.
- Frontend page design, except required organisation logo/name behavior.

## 3. Actors and Roles

Current roles available in the system:

| Role | Notes |
| --- | --- |
| ADMIN | Organisation admin / privileged user. |
| DOCTOR | Medical staff user. |
| NURSE | Nursing staff user. |
| PHARMACIST | Pharmacy staff user. |
| LAB_STAFF | Lab staff user. |
| BILLING_STAFF | Billing staff user. |
| OTHER_STAFF | Other organisation staff. |
| PATIENT | Patient self-registration user. |
| CUSTOMER | Pharmacy customer user. |

## 4. Pre-Conditions

- User must already exist before login.
- HMS staff and patient users must belong to a valid organisation through `users.org_id`.
- Pharmacy `CUSTOMER` users must belong to a valid organisation through `pharmacyusers.org_id`.
- User must have a valid role.
- User account must be active.
- User account must not be locked.
- Password must already be stored as an encoded password.
- Frontend should resolve and display the organisation name/logo before login when the login page is organisation-specific.

## 5. Functional Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-001 | User can login with username, phone number, or email address using a single `loginId` field. | Must |
| FR-002 | User must provide password. | Must |
| FR-003 | User should be created/register first before login is allowed. | Must |
| FR-004 | Only active users can login. | Must |
| FR-005 | Login response must include access token, refresh token, user ID, username, first name, last name, role, orgId, organisationName, and logoUrl. | Must |
| FR-006 | Login token claims must include username, role, and organisation ID. | Must |
| FR-007 | Refresh token must be saved against user and device ID. | Must |
| FR-008 | Re-login from the same device must replace the previous refresh token for that user/device. | Must |
| FR-009 | Browser clients must receive refresh token as an HttpOnly cookie. | Must |
| FR-010 | Refresh API must rotate refresh token and issue a new access token. | Must |
| FR-011 | Refresh must use the same device ID used during login. | Must |
| FR-012 | Login should not expose password, password hash, or internal DB data. | Must |
| FR-013 | Login page should display the respective organisation name/logo before authentication when the organisation is selected/resolved. | Should |
| FR-014 | Common login API must support all user roles, including pharmacy `CUSTOMER`; access is still isolated by `orgId` and downstream role permissions. | Must |
| FR-015 | Self-registered `PATIENT` and `CUSTOMER` accounts become active after successful email OTP verification, not admin approval. | Must |
| FR-016 | Failed login attempts must lock the account when the organisation's configured limit is reached. Default limit is 3. | Must |
| FR-017 | Admin users must be able to update max failed login attempts per organisation. | Must |
| FR-018 | A user can have at most 3 active device sessions. Same-device login replaces the existing session. | Must |
| FR-019 | Pharmacy `CUSTOMER` self-registration must create the account only in `pharmacyusers`, not in the normal `users` table. | Must |
| FR-020 | Login, refresh token, token blacklist, logout, lockout, and forgot-password flows must support standalone `pharmacyusers` customer accounts. | Must |
| FR-021 | Locked `PATIENT` and `CUSTOMER` accounts can use forgot-password email OTP to verify ownership, set a new password, clear failed attempts, and unlock the account. | Must |
| FR-022 | Locked admin/staff accounts must remain admin-managed and must not be self-unlocked through forgot-password. | Must |

## 6. API Contract

### 6.1 Login

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/login` | Public | Authenticate user and create access/refresh token pair. |

Request body:

```json
{
  "loginId": "superadmin",
  "password": "Admin@123",
  "deviceId": "web-chrome-01"
}
```

`loginId` may be:

- Username
- Email address
- Phone number

Success response:

```json
{
  "accessToken": "<jwt-access-token>",
  "refreshToken": "<refresh-token>",
  "userId": 1,
  "username": "superadmin",
  "firstName": "Super",
  "lastName": "Admin",
  "role": "ADMIN",
  "orgId": "ORG-001",
  "organisationName": "City Pharmacy",
  "logoUrl": "https://cdn.example.com/logo.png"
}
```

Cookie behavior:

- Set `refresh_token` as HttpOnly cookie.
- Cookie path: `/`
- SameSite: `Lax`
- Max age: 7 days.
- Secure cookie should be enabled only for secure non-localhost requests.

### 6.2 Refresh Token

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/refresh` | Refresh token | Rotate refresh token and issue new access token. |

Request body:

```json
{
  "accessToken": "<old-access-token>",
  "refreshToken": "<refresh-token>",
  "deviceId": "web-chrome-01"
}
```

Rules:

- Browser clients may omit `refreshToken` in JSON if `refresh_token` cookie is present.
- Swagger/API testers can pass `refreshToken` in the request body.
- `deviceId` must match the device ID stored with the refresh token.
- Old refresh token must be deleted after rotation.
- Old access token should be blacklisted when provided.
- Refresh response returns the same user and organisation convenience fields as login response.

### 6.3 Admin Login Policy

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| PATCH | `/v1/api/admin/login-policy/organisations/{orgId}` | ADMIN | Update maximum failed login attempts for an organisation. |
| PATCH | `/v1/api/admin/login-policy/users/{userId}/unlock` | ADMIN | Unlock a user account after failed-login lockout. |

Update request:

```json
{
  "maxLoginAttempts": 3
}
```

### 6.4 Forgot Password / Password Reset

This flow is public but only available for `PATIENT` and `CUSTOMER` accounts. Admin and staff accounts must be managed by an admin.

#### Send OTP

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/otp/send` | Public | Send email OTP for password reset. |

Request body:

```json
{
  "email": "customer@example.com",
  "purpose": "PASSWORD_RESET"
}
```

#### Verify OTP

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/otp/verify` | Public | Verify password reset OTP and issue short-lived reset token. |

Request body:

```json
{
  "email": "customer@example.com",
  "purpose": "PASSWORD_RESET",
  "otp": "123456"
}
```

Success response:

```json
{
  "message": "OTP verified successfully. Please set a new password.",
  "email": "customer@example.com",
  "purpose": "PASSWORD_RESET",
  "verified": true,
  "resetToken": "<one-time-reset-token>"
}
```

#### Reset Password

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/otp/reset-password` | Public reset token | Replace the account password after OTP verification. |

Request body:

```json
{
  "email": "customer@example.com",
  "resetToken": "<one-time-reset-token>",
  "newPassword": "Newpass@123",
  "confirmPassword": "Newpass@123"
}
```

Rules:

- `resetToken` is generated only after successful password-reset OTP verification.
- `resetToken` must be stored as a hash and must expire after 10 minutes.
- `newPassword` and `confirmPassword` must match.
- New password must meet password strength rules.
- Password is updated in `users.password` for `PATIENT` accounts.
- Password is updated in `pharmacyusers.password` for `CUSTOMER` accounts.
- Successful reset must clear `account_locked` and reset `failed_login_attempts` to 0.
- Reset token must be single-use.

## 7. Organisation Requirements

- User record must contain `orgId`.
- Login token must include the user's `orgId`.
- Backend DB user should be mapped with the correct organisation.
- Login response must include the user's organisation ID, organisation name, and logo URL for frontend convenience.
- A pharmacy-only package and enterprise package can share the same login API; package separation is enforced through different organisations and role-based authorization.
- Public organisation resolution is available through:

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/v1/api/public/organisations/resolve?key={key}` | Resolve active organisation by ID, name, or slug-like key. |

Frontend requirement:

- Before login, frontend should resolve the organisation key from URL/selection.
- Login page should show the organisation name and logo for the resolved organisation.
- Login must not allow the user to switch into another organisation by changing request body data. Organisation comes from the stored user record.

## 8. Database Requirements

### Users Table

Required fields for login:

| Field | Rule |
| --- | --- |
| `user_id` | Primary key. |
| `username` | Unique login identifier. |
| `email` | Unique login identifier, case-insensitive lookup. |
| `phone` | Unique login identifier. |
| `password` | Encoded password. |
| `role` | Must match a supported role. |
| `org_id` | Required organisation mapping. |
| `is_active` | Must be true for login. |
| `failed_login_attempts` | Number of consecutive failed password attempts. |
| `account_locked` | Must be false for login. |

### Pharmacy Users Table

Required fields for standalone pharmacy customer login:

| Field | Rule |
| --- | --- |
| `pharmacy_user_id` | Primary key and login response ID for `CUSTOMER` accounts. |
| `user_id` | Nullable legacy/optional link; self-registered customers must not require a `users` row. |
| `username` | Unique login identifier. |
| `email_address` | Unique login identifier, case-insensitive lookup. |
| `phone_number` | Unique login identifier. |
| `password` | Encoded password for customer login. |
| `role` | Must be `CUSTOMER`. |
| `org_id` | Required organisation mapping. |
| `is_active` | Must be true after email OTP account confirmation. |
| `failed_login_attempts` | Number of consecutive failed password attempts. |
| `account_locked` | Must be false for login. |

### Organisation Table

Required fields for login policy and frontend convenience:

| Field | Rule |
| --- | --- |
| `id` | Organisation ID used as `orgId`. |
| `organisation_name` | Returned in login and organisation resolve responses. |
| `logo_url` | Returned in login and organisation resolve responses. |
| `max_login_attempts` | Failed login limit for users in the organisation. Default is 3. |

### Refresh Token Table

Required fields:

| Field | Rule |
| --- | --- |
| `token` | Random refresh token value. |
| `user` | Linked HMS user for normal users. Nullable for standalone pharmacy customers. |
| `pharmacy_user` | Linked pharmacy customer user. Nullable for normal users. |
| `device_id` | Device identifier from request. |
| `device_name` | Captured from User-Agent. |
| `device_type` | WEB, ANDROID, IOS, or MOBILE. |
| `ip_address` | Captured from request. |
| `expiry_date` | Current time plus 7 days. |

### Token Blacklist Table

Used to invalidate old access tokens during refresh, logout, or same-device re-login when an old bearer token is provided.

### Email OTP Table

Required fields for account confirmation and password reset:

| Field | Rule |
| --- | --- |
| `email` | OTP recipient. |
| `purpose` | `ACCOUNT_CONFIRMATION` or `PASSWORD_RESET`. |
| `otp_hash` | Encoded OTP value. |
| `verification_reference` | Unique reference for audit/tracking. |
| `attempt_count` | Number of failed OTP attempts. |
| `max_attempts` | OTP attempt limit. |
| `expires_at` | OTP expiry time. |
| `consumed_at` | Set when OTP is verified. |
| `password_reset_token_hash` | Encoded one-time reset token for password reset. |
| `password_reset_token_expires_at` | Reset token expiry time. |
| `password_reset_used_at` | Set when password reset is completed. |

## 9. Validation Rules

- `loginId` is required.
- `password` is required.
- `deviceId` should be required for reliable device-specific token control.
- User must exist by username, email, or phone.
- User must be active.
- User must not be locked.
- Failed password attempts must increment `failed_login_attempts`.
- When failed attempts reach `organisation.max_login_attempts`, account must be locked.
- Successful login must reset failed login attempts.
- New-device login must be rejected when the user already has 3 active refresh-token device sessions.
- Refresh token must exist.
- Refresh token must not be expired.
- Refresh token device ID must match the request device ID.
- Token response must not include password or password hash.
- Forgot-password OTP send must reject unknown accounts.
- Forgot-password OTP send and verify must reject admin/staff roles for self-service reset.
- Password reset must require verified OTP, valid reset token, matching passwords, and strong password format.

## 10. Error Handling

| Status | Scenario | Expected Rule |
| --- | --- | --- |
| 400 | Missing required field | Return validation error. |
| 400 | Invalid refresh token or expired refresh token | Reject refresh request. |
| 401 | Invalid login ID/password | Reject login. |
| 403 | Account disabled | Reject login. |
| 423 | Account locked for admin/staff roles | Reject login until admin unlock. |
| 423 | Account locked for `PATIENT`/`CUSTOMER` roles | Reject login and direct user to forgot-password email verification. |
| 429 | Maximum active devices reached | Reject new-device login. |
| 404 | Organisation key not found on public resolve | Return organisation not found or inactive. |

Implementation note: current service throws runtime exceptions for some cases. For consistent API behavior, map these exceptions to stable error responses through a global exception handler.

## 11. Security Rules

- Password must never be logged or returned.
- Access token should be short-lived.
- Refresh token must expire after 7 days.
- Refresh token must be rotated on each refresh.
- Refresh token cookie must be HttpOnly.
- Access token should be sent as `Authorization: Bearer <accessToken>` for protected APIs.
- JWT must include role and organisation ID claims.
- Role checks for protected APIs must use the role claim.
- Same-device re-login should invalidate the previous refresh token.
- Users may login on up to 3 active devices.
- Failed login lockout limit is organisation-specific and defaults to 3.
- Admins can update an organisation's max login attempts and unlock locked accounts.
- Forgot-password reset tokens must be one-time use, hashed at rest, and short-lived.
- Customer password reset must update `pharmacyusers.password`, not `users.password`.

## 12. Development Notes

Existing implementation references:

- Controller: `src/main/java/com/hms/login/controller/LoginController.java`
- Service: `src/main/java/com/hms/login/service/impl/LoginServiceImpl.java`
- Request DTO: `src/main/java/com/hms/login/dto/LoginRequest.java`
- Response DTO: `src/main/java/com/hms/login/dto/LoginResponse.java`
- User repository: `src/main/java/com/hms/Register/repository/UserRepository.java`
- User entity: `src/main/java/com/hms/Register/entity/User.java`
- Organisation resolver: `src/main/java/com/hms/organisation/service/OrganisationResolverService.java`
- Pharmacy customer repository: `src/main/java/com/hms/Register/repository/PharmacyUserRepository.java`
- OTP controller/service: `src/main/java/com/hms/common/otp/controller/OtpController.java`, `src/main/java/com/hms/common/otp/service/EmailOtpService.java`

Suggested development improvements:

- Add `@Valid` to login request handling if validation should be enforced by Spring.
- Make `deviceId` required in `LoginRequest`.
- Standardize role values using `UserRole` enum wherever possible.
- Add global exception mapping for login and refresh errors.

## 13. Test Checklist

### Unit Tests

- Login succeeds with username.
- Login succeeds with email.
- Login succeeds with phone number.
- Login fails with wrong password.
- Login fails for inactive user.
- Login locks account after configured failed attempts.
- Login rejects locked account.
- Login creates refresh token for device.
- Login replaces old refresh token for same user/device.
- Login rejects new-device login after 3 active device sessions.
- Login response includes access token, refresh token, user ID, username, first name, last name, role, orgId, organisationName, and logoUrl.
- Access token includes username, role, and orgId claims.
- Refresh succeeds with valid token and matching device ID.
- Refresh fails with expired token.
- Refresh fails with invalid token.
- Refresh fails when device ID does not match.
- Refresh deletes old refresh token and saves new refresh token.
- Customer registration stores account only in `pharmacyusers`.
- Customer login succeeds from `pharmacyusers` without a linked `users` row.
- Customer failed login attempts lock `pharmacyusers.account_locked`.
- Password reset OTP verification returns reset token.
- Password reset updates encoded password, clears failed attempts, unlocks the account, and marks reset token as used.
- Admin/staff password reset attempt is rejected from self-service flow.

### API / Integration Tests

- `POST /v1/api/login` returns 200 for valid credentials.
- `POST /v1/api/login` sets `refresh_token` HttpOnly cookie.
- `POST /v1/api/refresh` accepts refresh token from JSON body.
- `POST /v1/api/refresh` accepts refresh token from cookie.
- `GET /v1/api/public/organisations/resolve` resolves active organisation by ID.
- `GET /v1/api/public/organisations/resolve` resolves active organisation by name.
- `GET /v1/api/public/organisations/resolve` rejects inactive or unknown organisation.
- `PATCH /v1/api/admin/login-policy/organisations/{orgId}` updates max login attempts for admin users.
- `PATCH /v1/api/admin/login-policy/users/{userId}/unlock` unlocks a locked user for admin users.
- `POST /v1/api/otp/send` sends password reset OTP for `PATIENT`/`CUSTOMER`.
- `POST /v1/api/otp/verify` verifies password reset OTP and returns reset token.
- `POST /v1/api/otp/reset-password` replaces password and unlocks `PATIENT`/`CUSTOMER`.

### Manual Swagger Testing

1. Create or confirm an active user exists.
2. Call `/v1/api/login` with `loginId`, `password`, and `deviceId`.
3. Copy `accessToken`.
4. Use Swagger Authorize with only the token value, without `Bearer`.
5. Call a protected API for the user's role.
6. Call `/v1/api/refresh` with the refresh token and same device ID.
7. Confirm the old refresh token no longer works.
8. Trigger 3 failed logins for a `CUSTOMER` account and confirm the UI navigates to forgot password.
9. Complete email OTP verification, set a new password, and confirm login succeeds with the new password.

## 14. Resolved Product Decisions

- Common login API is used for all users, including pharmacy `CUSTOMER`; organisation/package separation is enforced through `orgId` and authorization rules.
- Self-registered `PATIENT` and `CUSTOMER` accounts are activated by email OTP verification, not admin approval.
- Login and refresh responses include `userId`, `orgId`, `organisationName`, and `logoUrl` for frontend convenience.
- Failed login attempts lock the account at the organisation-configured limit; default maximum is 3 attempts.
- Admin users can change the max failed-attempt policy per organisation and unlock locked accounts.
- Users may have up to 3 active device sessions.
- Pharmacy `CUSTOMER` self-registration stores customer accounts only in `pharmacyusers`.
- `PATIENT` and `CUSTOMER` users can self-reset password through email OTP; admin/staff roles remain admin-managed.
- Forgot-password is a two-step flow: OTP verification creates a short-lived reset token, then reset-password replaces the encoded password in the correct table.
