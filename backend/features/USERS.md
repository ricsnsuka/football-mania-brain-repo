# Users

**Added in:** v1.0.0  
**Date:** May 15, 2026  
**Status:** ✅ Released

---

## Overview

A **User** represents a person who can log in to the Football Management System. Users are
the authentication layer of the application. They may optionally be linked to a **Player**
profile, which is managed from the Player entity side — including accounts holding `GROUP_ADMIN`,
since V18 removed the rule that once forbade it.

---

## Business Rules

- A user must have a unique **username** (3–50 chars) and a unique **email**.
- A user can log in using **either** their username or their email address.
- Passwords must be at least **8 characters** and are stored BCrypt-hashed.
- There are three roles — `ORGANIZER`, `MANAGER`, `GROUP_ADMIN` — and since V18 they are **flat and
  independent**: a user holds a *set* of them, and `GROUP_ADMIN` implies neither of the others.
- An account may hold **no roles at all**. That is the normal state of a new account and the
  replacement for the old `BASIC_USER`: authenticated, and nothing more.
- Any account may be linked to a player profile, `GROUP_ADMIN` included.
- **Registration is self-service** — `POST /api/users/register` is public and always creates an
  account with no roles. Callers cannot choose their own grants.
- Since V23 the grants are held **per group**, on the membership rather than on the account:
  holding `GROUP_ADMIN` in one group says nothing about any other.
- **Deactivation is soft** — setting `isActive = false` prevents login without deleting data.
- On first login, `forcePasswordChange = true` signals the client to prompt a password change.
- Only the account owner (or an admin) can change a password; current password must be verified.
- Only admins can change a user's roles or active status.
- Owners can update their own `firstName`, `lastName`, and `email` (email must remain unique).

---

## Roles

| Role        | Capabilities                                                                     |
|-------------|----------------------------------------------------------------------------------|
| `GROUP_ADMIN`     | System: settings, user administration, rating recalculation, GDPR actions, purge. |
| `MANAGER`   | Matches: create plans and matches, manage the roster, generate teams, record results. |
| `ORGANIZER` | Money: see all balances, record payments, set fees, void and waive charges.       |
| *(none)*    | Authenticated, and nothing more.                                                  |

> **A set, not a ladder.** Holding `GROUP_ADMIN` grants system administration and says nothing about
> whether the holder may manage matches — grant `MANAGER` as well if they should. There is
> deliberately no `PLAYER` role: "is a player" is already a fact in the schema
> (`players.user_id` is UNIQUE), and a role asserting the same thing would be a second source of
> truth free to disagree with the first.

---

## API Endpoints

Base path: `/api/users` · Auth endpoints: `/api/auth`

| Method | Path                            | Description                        | Auth Required      |
|--------|---------------------------------|------------------------------------|--------------------|
| POST   | `/api/auth/login`               | Obtain JWT token                   | Public             |
| POST   | `/api/users/register`        | Self-service registration — always creates an account with **no roles** | Public |
| GET    | `/api/users`                 | List all users (paginated)         | `GROUP_ADMIN`            |
| GET    | `/api/users/me`              | Get own user profile               | Any authenticated  |
| GET    | `/api/users/{id}`            | Get user by ID                     | `GROUP_ADMIN` or own     |
| POST   | `/api/users`                 | Create a new user (roles may be set) | `GROUP_ADMIN`          |
| PATCH  | `/api/users/{id}`            | Update profile (name, email)       | `GROUP_ADMIN` or own     |
| PATCH  | `/api/users/{id}/role`       | Update roles / active status       | `GROUP_ADMIN`            |
| ~~PATCH~~ | ~~`/api/users/{id}/reactivate`~~ | **Removed** — use `PATCH /{id}/role` with `{"isActive": true}` | `GROUP_ADMIN` |
| DELETE | `/api/users/{id}`            | Deactivate user (soft delete)      | `GROUP_ADMIN`            |
| POST   | `/api/users/{id}/change-password` | Change own password           | `GROUP_ADMIN` or own     |

> `POST /api/users` and `POST /api/users/register` are **different endpoints with different
> payloads**. The first is admin-only and accepts `roles`; the second is public and has no
> `roles` field at all, because self-service role selection would be a privilege-escalation
> route.

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
| `role`                | String  | **Deprecated.** The single name this set would have carried before V18, emitted for one release so an already-deployed frontend does not read every user as unprivileged. Delete with `Role.legacyNameFor` once the frontend reads `roles` |
| `roles`               | String[] | Every grant held, sorted; **empty for an unprivileged account** |
| `forcePasswordChange` | boolean | `true` → client must redirect to change-password UI |

### `UserDTO` — Read view

| Field                 | Type    | Notes                                   |
|-----------------------|---------|-----------------------------------------|
| `id`                  | Long    |                                         |
| `username`            | String  |                                         |
| `email`               | String  |                                         |
| `firstName`           | String  | nullable                                |
| `lastName`            | String  | nullable                                |
| `roles`               | String[] | Sorted list — a `List` rather than a `Set` so the response is byte-for-byte stable; empty for an unprivileged account |
| `isActive`            | boolean |                                         |
| `forcePasswordChange` | boolean |                                         |
| `createdAt`           | Instant | ISO-8601 UTC timestamp                  |

### `UserCreateDTO` — POST /api/users (Request — GROUP_ADMIN)

| Field       | Type   | Required | Constraints         |
|-------------|--------|----------|---------------------|
| `username`  | String | ✅       | 3–50 chars          |
| `email`     | String | ✅       | valid email format  |
| `password`  | String | ✅       | 8–100 chars         |
| `firstName` | String | ❌       | max 100 chars       |
| `lastName`  | String | ❌       | max 100 chars       |
| `roles`     | String[] | ❌     | Any of `ORGANIZER` / `MANAGER` / `GROUP_ADMIN`. Empty or absent creates an account that is authenticated and nothing more |

### `UserRegisterDTO` — POST /api/users/register (Request — public)

| Field       | Type   | Required | Constraints        |
|-------------|--------|----------|--------------------|
| `username`  | String | ✅       | 3–50 chars         |
| `email`     | String | ✅       | valid email format |
| `password`  | String | ✅       | 8–100 chars        |
| `firstName` | String | ❌       | max 100 chars      |
| `lastName`  | String | ❌       | max 100 chars      |

> **There is no `roles` field, deliberately.** The account is created with no grants whatever the
> caller sends, so a hand-rolled request cannot escalate. An administrator grants roles afterwards
> via `PATCH /api/users/{id}/role`.

### `UserUpdateDTO` — PATCH /api/users/{id} (Request — owner or GROUP_ADMIN)

| Field       | Type   | Required | Constraints        | Notes           |
|-------------|--------|----------|--------------------|-----------------|
| `firstName` | String | ❌       | max 100 chars      | null = no change|
| `lastName`  | String | ❌       | max 100 chars      | null = no change|
| `email`     | String | ❌       | valid email format | null = no change|

### `AdminUserUpdateDTO` — PATCH /api/users/{id}/role (Request — GROUP_ADMIN only)

| Field      | Type     | Required | Notes                                                        |
|------------|----------|----------|--------------------------------------------------------------|
| `roles`    | String[] | ❌       | `null` = no change. When supplied it **replaces** the grants wholesale — send the intended end state, not a delta. Any of `ORGANIZER` / `MANAGER` / `GROUP_ADMIN` |
| `isActive` | Boolean  | ❌       | `null` = no change                                           |

> There is no grant/revoke pair by design: sending the end state means two administrators editing
> at once cannot interleave into a combination neither of them chose. An **explicitly empty set is
> therefore meaningful** — it strips every role, leaving the account authenticated and nothing
> more. Only `null` means "leave alone".

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
              └─ return LoginResponseDTO { token, userId, username, email,
                                           role (deprecated), roles, forcePasswordChange }
```

Authorities on the token's principal come from the caller's **membership** for the current group
since V23 — `user_roles` is frozen and no longer read. An account with no membership authenticates
and holds nothing, which is a legitimate state rather than an error.

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
  "role": null,
  "roles": [],
  "forcePasswordChange": false
}
```

*(This is an account with no grants — `roles` is empty and the deprecated `role` is `null`. An
administrator who also runs match day would show `"roles": ["GROUP_ADMIN", "MANAGER"]`.)*

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
  "roles": ["MANAGER"]
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
  "roles": ["MANAGER"],
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
| **Auth** | `GROUP_ADMIN` only |
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
  "roles": ["MANAGER"],
  "isActive": true,
  "forcePasswordChange": false,
  "createdAt": "2026-05-15T10:30:00Z"
}
```

**Error cases:**

| Status | Condition | Message |
|--------|-----------|---------|
| `403 Forbidden` | Caller does not hold `GROUP_ADMIN` | Forbidden |
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
| Unrecognised role in `roles`       | 400    | Field-level validation error — `must be one of ORGANIZER, MANAGER, GROUP_ADMIN`. The constraint sits on the set's *elements*, so a bad value is reported against the field rather than reaching `UserService` as a parse failure |
| Wrong current password on change   | 400    | `Current password is incorrect`            |
| User not found                     | 404    | `User with id {id} not found`              |
| Non-admin accessing admin endpoint | 403    | Forbidden                                  |
| Non-owner accessing own endpoint   | 403    | Forbidden                                  |

---

## Known Limitations

- No password reset via email (not yet implemented — future feature)
- JWT expiry defaults to 24 hours (`app.jwt.expiration-ms=86400000`); no refresh token mechanism yet

