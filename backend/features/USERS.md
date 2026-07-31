# Users

**Added in:** v1.0.0  
**Date:** May 15, 2026  
**Status:** ✅ Released

---

## Overview

A **User** represents a person who can log in to the Football Management System. Users are
the authentication layer of the application. They may optionally be linked to a **Player**
profile, which is managed from the Player entity side. If a user holds the `ADMIN_USER` role
they cannot be linked to a player.

---

## Business Rules

- A user must have a unique **username** (3–50 chars) and a unique **email**.
- A user can log in using **either** their username or their email address.
- Passwords must be at least **8 characters** and are stored BCrypt-hashed.
- There are three roles: `BASIC_USER`, `MASTER_USER`, `ADMIN_USER`.
- `ADMIN_USER` accounts **cannot** be linked to a player profile.
- **Deactivation is soft** — setting `isActive = false` prevents login without deleting data.
- On first login, `forcePasswordChange = true` signals the client to prompt a password change.
- Only the account owner (or an admin) can change a password; current password must be verified.
- Only admins can change a user's role or active status.
- Owners can update their own `firstName`, `lastName`, and `email` (email must remain unique).

---

## Roles

| Role          | Capabilities                                                              |
|---------------|---------------------------------------------------------------------------|
| `ADMIN_USER`  | Full system access. Cannot own a player profile.                         |
| `MASTER_USER` | Creates and manages matches, seasons. Can link to a player profile.      |
| `BASIC_USER`  | Views own stats and match history. Can link to a player profile.         |

---

## API Endpoints

Base path: `/api/users` · Auth endpoints: `/api/auth`

| Method | Path                            | Description                        | Auth Required      |
|--------|---------------------------------|------------------------------------|--------------------|
| POST   | `/api/auth/login`               | Obtain JWT token                   | Public             |
| GET    | `/api/users`                 | List all users (paginated)         | `ADMIN_USER`       |
| GET    | `/api/users/me`              | Get own user profile               | Any authenticated  |
| GET    | `/api/users/{id}`            | Get user by ID                     | `ADMIN_USER` or own|
| POST   | `/api/users`                 | Create a new user                  | `ADMIN_USER`       |
| PATCH  | `/api/users/{id}`            | Update profile (name, email)       | `ADMIN_USER` or own|
| PATCH  | `/api/users/{id}/role`       | Update role / active status        | `ADMIN_USER`       |
| ~~PATCH~~ | ~~`/api/users/{id}/reactivate`~~ | **Removed** — use `PATCH /{id}/role` with `{"isActive": true}` | `ADMIN_USER` |
| DELETE | `/api/users/{id}`            | Deactivate user (soft delete)      | `ADMIN_USER`       |
| POST   | `/api/users/{id}/change-password` | Change own password           | `ADMIN_USER` or own|

---

## DTOs

### `LoginRequestDTO` — POST /api/auth/login (Request)

| Field        | Type   | Required | Constraints  | Notes                          |
|--------------|--------|----------|--------------|--------------------------------|
| `identifier` | String | ✅       | not blank    | Username **or** email          |
| `password`   | String | ✅       | not blank    |                                |

### `LoginResponseDTO` — POST /api/auth/login (Response)

| Field                 | Type    | Notes                                               |
|-----------------------|---------|-----------------------------------------------------|
| `token`               | String  | HMAC-SHA256 JWT bearer token                        |
| `userId`              | Long    | Database ID of the authenticated user               |
| `username`            | String  |                                                     |
| `email`               | String  |                                                     |
| `role`                | String  | `BASIC_USER` / `MASTER_USER` / `ADMIN_USER`         |
| `forcePasswordChange` | boolean | `true` → client must redirect to change-password UI |

### `UserDTO` — Read view

| Field                 | Type    | Notes                                   |
|-----------------------|---------|-----------------------------------------|
| `id`                  | Long    |                                         |
| `username`            | String  |                                         |
| `email`               | String  |                                         |
| `firstName`           | String  | nullable                                |
| `lastName`            | String  | nullable                                |
| `role`                | String  | `BASIC_USER` / `MASTER_USER` / `ADMIN_USER` |
| `isActive`            | boolean |                                         |
| `forcePasswordChange` | boolean |                                         |
| `createdAt`           | Instant | ISO-8601 UTC timestamp                  |

### `UserCreateDTO` — POST /api/users (Request — ADMIN)

| Field       | Type   | Required | Constraints         |
|-------------|--------|----------|---------------------|
| `username`  | String | ✅       | 3–50 chars          |
| `email`     | String | ✅       | valid email format  |
| `password`  | String | ✅       | 8–100 chars         |
| `firstName` | String | ❌       | max 100 chars       |
| `lastName`  | String | ❌       | max 100 chars       |
| `role`      | String | ✅       | `BASIC_USER` / `MASTER_USER` / `ADMIN_USER` |

### `UserUpdateDTO` — PATCH /api/users/{id} (Request — owner or ADMIN)

| Field       | Type   | Required | Constraints        | Notes           |
|-------------|--------|----------|--------------------|-----------------|
| `firstName` | String | ❌       | max 100 chars      | null = no change|
| `lastName`  | String | ❌       | max 100 chars      | null = no change|
| `email`     | String | ❌       | valid email format | null = no change|

### `AdminUserUpdateDTO` — PATCH /api/users/{id}/role (Request — ADMIN only)

| Field      | Type    | Required | Notes                |
|------------|---------|----------|----------------------|
| `role`     | String  | ❌       | null = no change     |
| `isActive` | Boolean | ❌       | null = no change     |

### `ChangePasswordDTO` — POST /api/users/{id}/change-password (Request)

| Field             | Type   | Required | Constraints  |
|-------------------|--------|----------|--------------|
| `currentPassword` | String | ✅       | not blank    |
| `newPassword`     | String | ✅       | 8–100 chars  |

---

## Implementation Details

### Login Flow

```
Client
  │
  ├─ POST /api/auth/login  { identifier, password }
  │
  └─► AuthController
        └─► AuthService.login()
              ├─ identifier contains "@"?
              │     YES → UserRepository.findByEmail() → get username
              │     NO  → use identifier as username directly
              │
              ├─ AuthenticationManager.authenticate(username, password)
              │     └─► UserDetailsServiceImpl.loadUserByUsername()
              │               ├─ identifier contains "@" → findByEmail
              │               └─ else                   → findByUsername
              │
              ├─ JwtTokenProvider.generateToken(authentication)
              │
              └─ return LoginResponseDTO { token, userId, username, email, role, forcePasswordChange }
```

### Caching

- `UserService.getUser(id)` → cached under key `"user-{id}"` in the `"users"` Caffeine cache (10 min TTL)
- `UserService.listUsers(pageable)` → cached under key `"page-{pageable}"`
- All write methods (`createUser`, `updateUser`, `adminUpdateUser`, `changePassword`, `deleteUser`) trigger `@CacheEvict(allEntries = true)` on the `"users"` cache

### Security

- All `/api/auth/**` paths are public (no token required)
- Endpoint-level access control is enforced via `@PreAuthorize` Spring Security SpEL
- "Own record" checks use `#id == authentication.principal.id` in SpEL — the principal is a `UserPrincipal` Java Record

### Password Handling

- Passwords encoded with BCrypt (strength 12)
- `changePassword` verifies `currentPassword` via `PasswordEncoder.matches()` before encoding the new one
- Successful password change sets `forcePasswordChange = false`

---

## Examples

### Example: Login with username

**Request:**
```http
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{
  "identifier": "johndoe",
  "password": "myPassword123"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "userId": 5,
  "username": "johndoe",
  "email": "john@example.com",
  "role": "BASIC_USER",
  "forcePasswordChange": false
}
```

---

### Example: Login with email

**Request:**
```http
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{
  "identifier": "john@example.com",
  "password": "myPassword123"
}
```

**Response:** *(identical to username login)*

---

### Example: Create a user (admin)

**Request:**
```http
POST /api/users HTTP/1.1
Content-Type: application/json
Authorization: Bearer <admin-token>

{
  "username": "janedoe",
  "email": "jane@example.com",
  "password": "securePass1",
  "firstName": "Jane",
  "lastName": "Doe",
  "role": "BASIC_USER"
}
```

**Response `201 Created`:**
```json
{
  "id": 6,
  "username": "janedoe",
  "email": "jane@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "role": "BASIC_USER",
  "isActive": true,
  "forcePasswordChange": false,
  "createdAt": "2026-05-15T10:30:00Z"
}
```

---

### Example: Change password

**Request:**
```http
POST /api/users/6/change-password HTTP/1.1
Content-Type: application/json
Authorization: Bearer <own-token>

{
  "currentPassword": "securePass1",
  "newPassword": "newSecurePass2"
}
```

**Response:** `204 No Content`

---

### Example: Reactivate an inactive user (admin)

Restores a deactivated user account by setting `isActive = true`. The companion
deactivation endpoint is `DELETE /api/users/{id}` (soft delete — sets `isActive = false`).

> ⚠️ The dedicated `PATCH /api/users/{id}/reactivate` endpoint **was removed**. It set exactly
> the field `AdminUserUpdateDTO` already carries, so it was a second way to do one thing — and a
> second place for the authorisation rules to drift. Reactivate via `/role` as shown below.

| Detail | Value |
|--------|-------|
| **Method** | `PATCH` |
| **Path** | `/api/users/{id}/role` |
| **Auth** | `ADMIN_USER` only |
| **Request body** | `AdminUserUpdateDTO` — `{"isActive": true}` |
| **Success response** | `200 OK` — `UserDTO` with `isActive: true` |

**Request:**
```http
PATCH /api/users/6/role HTTP/1.1
Authorization: Bearer <admin-token>
Content-Type: application/json

{"isActive": true}
```

**Response `200 OK`:**
```json
{
  "id": 6,
  "username": "janedoe",
  "email": "jane@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "role": "BASIC_USER",
  "isActive": true,
  "forcePasswordChange": false,
  "createdAt": "2026-05-15T10:30:00Z"
}
```

**Error cases:**

| Status | Condition | Message |
|--------|-----------|---------|
| `403 Forbidden` | Caller is not `ADMIN_USER` | Forbidden |
| `404 Not Found` | User ID does not exist | `User with id {id} not found` |
| `409 Conflict` | User is already active | `User is already active` |

> ℹ️ To deactivate a user use `DELETE /api/users/{id}` (soft delete). Both endpoints return
> the full `UserDTO` (or `204` for deactivation) so the frontend can update its local state
> immediately.

---

## Error Responses

| Scenario                           | Status | Error Message                              |
|------------------------------------|--------|--------------------------------------------|
| Wrong password at login            | 401    | Bad credentials                            |
| Inactive user login attempt        | 401    | User is inactive                           |
| Duplicate username on create       | 409    | `Username already taken: {username}`       |
| Duplicate email on create/update   | 409    | `Email already registered: {email}`        |
| Invalid role string                | 400    | `Invalid role: {role}. Must be one of: ...`|
| Wrong current password on change   | 400    | `Current password is incorrect`            |
| User not found                     | 404    | `User with id {id} not found`              |
| Non-admin accessing admin endpoint | 403    | Forbidden                                  |
| Non-owner accessing own endpoint   | 403    | Forbidden                                  |

---

## Known Limitations

- No password reset via email (not yet implemented — future feature)
- No account registration self-service — only admins can create users via `POST /api/users`
- JWT expiry defaults to 24 hours (`app.jwt.expiration-ms=86400000`); no refresh token mechanism yet

