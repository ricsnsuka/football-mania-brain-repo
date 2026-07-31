# Player Feature

**Added in:** v1.1.0 (Unreleased)  
**Date:** 2026-05-15  
**Status:** 🚧 In Progress (merged to main, pending release tag)

---

## Overview

The **Player** entity is the central domain object in the Football Management System. It represents a physical person who participates in football matches. A player can optionally be linked to an `AppUser` account (enabling authentication), but players and users are deliberately separate concepts — not every player has a login, and `ADMIN_USER` accounts can **never** be players.

Players carry skill ratings, win-streak counters, and an audit trail. The Player API exposes full CRUD operations with role-based write protection: reads are open to all authenticated users, writes require `ADMIN_USER` or `MASTER_USER`.

---

## Data Model

### `players` Table (V1 baseline + V2 audit columns)

| Column              | Type           | Nullable | Notes                                                 |
|---------------------|----------------|----------|-------------------------------------------------------|
| `id`                | BIGSERIAL (PK) | No       |                                                       |
| `name`              | VARCHAR(100)   | No       | Display name                                          |
| `phone_number`      | VARCHAR(20)    | Yes      |                                                       |
| `is_active`         | BOOLEAN        | No       | Default `true`; inactive players excluded from teams  |
| `skill_rating`      | DOUBLE         | No       | 1.0–10.0; evolves via match performance algorithm     |
| `base_skill_rating` | INTEGER        | No       | 1–10; set at creation; **never** modified by algorithm|
| `current_streak`    | INTEGER        | No       | Consecutive wins                                      |
| `longest_streak`    | INTEGER        | No       | All-time best streak                                  |
| `user_id`           | BIGINT (FK)    | Yes      | FK → `users.id`; null if unlinked                    |
| `created_by`        | VARCHAR(50)    | No       | **Added in V2** — username of creator                 |
| `updated_by`        | VARCHAR(50)    | No       | **Added in V2** — username of last updater            |
| `created_at`        | TIMESTAMP      | No       | Set on INSERT                                         |
| `updated_at`        | TIMESTAMP      | No       | Updated on every PATCH                                |

> **Note:** `email` is **not** stored in the `players` table. It is derived at query time from the linked `AppUser.email`. If no user is linked, `email` is `null` in the response.

### DB Migration History

| Migration | File | Description |
|-----------|------|-------------|
| V1 | `V1__initial_schema.sql` | Baseline schema — includes `players` table (without audit columns) |
| V1 | `V1__initial_schema.sql` | Creates `players`, including the `created_by` / `updated_by` audit columns |

---

## PlayerDTO (Response Shape)

All read and write endpoints return a `PlayerDTO`:

| Field             | Type    | Nullable | Notes                                         |
|-------------------|---------|----------|-----------------------------------------------|
| `id`              | Long    | No       |                                               |
| `name`            | String  | No       |                                               |
| `email`           | String  | Yes      | Sourced from linked `AppUser`; `null` if none |
| `isActive`        | boolean | No       |                                               |
| `skillRating`     | double  | No       | 1.0–10.0                                      |
| `baseSkillRating` | int     | No       | 1–10; immutable after creation                |
| `phoneNumber`     | String  | Yes      |                                               |
| `currentStreak`   | int     | No       |                                               |
| `longestStreak`   | int     | No       |                                               |
| `linkedUserId`    | Long    | Yes      | ID of the linked `AppUser`; null if unlinked  |
| `createdBy`       | String  | No       | Username of creator (audit)                   |
| `updatedBy`       | String  | No       | Username of last updater (audit)              |
| `createdAt`       | Instant | No       | ISO-8601 UTC                                  |
| `updatedAt`       | Instant | No       | ISO-8601 UTC                                  |

---

## Request DTOs

### PlayerCreateDTO — `POST /api/players`

| Field             | Type    | Required | Validation                        |
|-------------------|---------|----------|-----------------------------------|
| `name`            | String  | ✅ Yes   | `@NotBlank @Size(min=2, max=100)` |
| `baseSkillRating` | Integer | ✅ Yes   | `@NotNull @Min(1) @Max(10)`       |
| `phoneNumber`     | String  | No       | `@Size(max=20)`                   |
| `isActive`        | Boolean | No       | Defaults to `true` in service     |
| `userId`          | Long    | No       | Links to `AppUser`; `ADMIN_USER` accounts rejected |

### PlayerUpdateDTO — `PATCH /api/players/{id}`

| Field         | Type    | Required | Validation              | Notes                                |
|---------------|---------|----------|-------------------------|--------------------------------------|
| `name`        | String  | No       | `@Size(min=2, max=100)` | `null` = no change                   |
| `phoneNumber` | String  | No       | `@Size(max=20)`         | `null` = no change                   |
| `userId`      | Long    | No       | —                       | New user to link; `null` = no change |
| `unlinkUser`  | Boolean | No       | —                       | `true` = explicitly remove user link |

> ⚠️ **PATCH Semantics & the `unlinkUser` Flag**  
> Standard PATCH semantics treat `null` as "no change". This creates an ambiguity: how do you *remove* a user link by sending `null` for `userId` when `null` also means "leave it alone"?  
> The `unlinkUser: true` flag resolves this. To unlink a user from a player, send `{ "unlinkUser": true }`. Sending `{ "userId": null }` without this flag is a no-op.  
> This flag was introduced in the Phase 3 compliance review to prevent accidental user unlinking.

### PlayerStatusDTO — `PATCH /api/players/{id}/status`

| Field      | Type    | Required | Validation |
|------------|---------|----------|------------|
| `isActive` | Boolean | ✅ Yes   | `@NotNull` |

---

## API Endpoints

Base path: `/api/players`

| Method | Path | Auth Required | Description | Response |
|--------|------|---------------|-------------|----------|
| GET | `/api/players` | Any authenticated | List players (paginated) | `Page<PlayerDTO>` (200) |
| GET | `/api/players/{id}` | Any authenticated | Get player by ID | `PlayerDTO` (200) |
| POST | `/api/players` | `ADMIN_USER` or `MASTER_USER` | Create new player | `PlayerDTO` (201) |
| PATCH | `/api/players/{id}` | `ADMIN_USER` or `MASTER_USER` | Partial update | `PlayerDTO` (200) |
| PATCH | `/api/players/{id}/status` | `ADMIN_USER` or `MASTER_USER` | Activate / deactivate | `PlayerDTO` (200) |
| POST | `/api/players/{id}/link-me` | Any authenticated (ADMIN_USER rejected) | Link own account to player — no body | `PlayerDTO` (200) |
| DELETE | `/api/players/{id}` | `ADMIN_USER` only | Hard delete (no stats) | `204 No Content` |

### Authorization Matrix

| Action | `BASIC_USER` | `MASTER_USER` | `ADMIN_USER` |
|--------|:---:|:---:|:---:|
| Read player list | ✅ | ✅ | ✅ |
| Read player by ID | ✅ | ✅ | ✅ |
| Create player | ❌ | ✅ | ✅ |
| Update player | ❌ | ✅ | ✅ |
| Activate / deactivate | ❌ | ✅ | ✅ |
| Delete player | ❌ | ❌ | ✅ |
| Link self as player | ✅ | ✅ | ❌ (blocked at service — 403) |

---

## Request / Response Examples

### Create a Player

**Request:**
```http
POST /api/players HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "João Silva",
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "isActive": true,
  "userId": 5
}
```

**Response** `201 Created`:
```json
{
  "id": 42,
  "name": "João Silva",
  "email": "joao.silva@example.com",
  "isActive": true,
  "skillRating": 7.0,
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "currentStreak": 0,
  "longestStreak": 0,
  "linkedUserId": 5,
  "createdBy": "admin",
  "updatedBy": "admin",
  "createdAt": "2026-05-15T10:30:00Z",
  "updatedAt": "2026-05-15T10:30:00Z"
}
```

### Partial Update (rename + change phone)

**Request:**
```http
PATCH /api/players/42 HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "João M. Silva",
  "phoneNumber": "+351987654321"
}
```

**Response** `200 OK`:
```json
{
  "id": 42,
  "name": "João M. Silva",
  "email": "joao.silva@example.com",
  "isActive": true,
  "skillRating": 7.0,
  "baseSkillRating": 7,
  "phoneNumber": "+351987654321",
  "currentStreak": 0,
  "longestStreak": 0,
  "linkedUserId": 5,
  "createdBy": "admin",
  "updatedBy": "master_user",
  "createdAt": "2026-05-15T10:30:00Z",
  "updatedAt": "2026-05-15T11:00:00Z"
}
```

### Unlink a User from a Player

**Request:**
```http
PATCH /api/players/42 HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "unlinkUser": true
}
```

**Response** `200 OK` — `linkedUserId` and `email` become `null`.

### Deactivate a Player

**Request:**
```http
PATCH /api/players/42/status HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "isActive": false
}
```

**Response** `200 OK` — player returned with `"isActive": false`.

### Delete a Player

```http
DELETE /api/players/42 HTTP/1.1
Authorization: Bearer <token>
```

**Success:** `204 No Content`  
**Blocked (has stats):** `409 Conflict` — player has match statistics and cannot be deleted.

### Self-Link: Link Your Own Account to a Player

**Request:**
```http
POST /api/players/42/link-me HTTP/1.1
Authorization: Bearer <token>
```
*(No request body.)*

**Response** `200 OK`:
```json
{
  "id": 42,
  "name": "João Silva",
  "email": "joao@example.com",
  "isActive": true,
  "skillRating": 7.0,
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "currentStreak": 3,
  "longestStreak": 5,
  "linkedUserId": 12,
  "createdBy": "admin",
  "updatedBy": "joao",
  "createdAt": "2026-01-10T08:00:00Z",
  "updatedAt": "2026-06-29T10:00:00Z"
}
```

**Conflict responses (409):**
- `"Player is already linked to your account"` — calling user is already the linked user for this player
- `"Player is already linked to another user"` — player already has a different user linked
- `"You are already linked to another player"` — calling user already linked to a different player

---

### Admin Unlink User from a Player

Removes the link between a player and their associated `AppUser` account. The player record is preserved; only the `user_id` FK is set to `null`.

**Request:**
```http
DELETE /api/players/42/user-link HTTP/1.1
Authorization: Bearer <admin-token>
```
*(No request body.)*

**Response** `200 OK`:
```json
{
  "id": 42,
  "name": "João Silva",
  "email": null,
  "isActive": true,
  "skillRating": 7.0,
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "currentStreak": 3,
  "longestStreak": 5,
  "linkedUserId": null,
  "createdBy": "admin",
  "updatedBy": "admin",
  "createdAt": "2026-01-10T08:00:00Z",
  "updatedAt": "2026-07-01T12:00:00Z"
}
```

> Both `linkedUserId` and `email` become `null` after a successful unlink.

**Error responses:**
- `404 Not Found` — player does not exist
- `409 Conflict` — `"Player is not linked to any user"` — player has no user to unlink
- `403 Forbidden` — caller is not `ADMIN_USER`

---

## Business Rules

1. **Skill rating range** — `skillRating` is always between **1.0 and 10.0**. `baseSkillRating` is an integer 1–10 set at creation and **never** modified by the rating algorithm.
2. **ADMIN_USER cannot be a player** — the service layer rejects any attempt to link an `ADMIN_USER` account to a player profile (`403 Forbidden`). This applies to `POST /api/players`, `PATCH /api/players/{id}`, and `POST /api/players/{id}/link-me`.
3. **Inactive players excluded from teams** — during balanced team generation, only `isActive: true` players are eligible.
4. **Hard delete guard** — `DELETE /api/players/{id}` is blocked (`409 Conflict`) if the player has any records in `player_stats`. Use deactivation instead.
5. **Email is derived** — `email` is read from the linked `AppUser` at query time; it is not persisted in `players`. If no user is linked, `email` is `null`.
6. **Audit trail** — every create/update records the authenticated principal's username in `created_by` / `updated_by`.
7. **Safe PATCH** — `null` fields in `PlayerUpdateDTO` mean "no change". The explicit `unlinkUser: true` flag is required to remove a user link.
8. **Self-link uniqueness** — a player can be linked to at most one user, and a user can be linked to at most one player. `POST /{id}/link-me` enforces both constraints, returning `409 Conflict` with a descriptive message for each violation. A `DataIntegrityViolationException` (concurrent race) also resolves to `409`.
9. **Admin unlink** — `DELETE /api/players/{id}/user-link` is restricted to `ADMIN_USER`. Returns `409 Conflict` if the player has no linked user (`"Player is not linked to any user"`). Returns the updated `PlayerDTO` with `linkedUserId: null` and `email: null` on success.

---

## Caching Strategy

| Cache Name      | Used By | Evicted When |
|-----------------|---------|--------------|
| `players`       | `GET /api/players` (paginated list) | Any write to a player |
| `playerProfile` | `GET /api/players/{id}` (single player) | Update or delete of that player |

Both caches use **Caffeine** (local, no Redis) with a **10-minute TTL** and **500 max entries**.

Cache annotations in `PlayerService`:
- `@Cacheable("playerProfile")` on `getPlayer(id)`
- `@CacheEvict(value = {"players", "playerProfile"}, allEntries = true)` on create/update/delete

---

## Security Notes

### JWT_SECRET Environment Variable (Mandatory)

The application **will not start** without the `JWT_SECRET` environment variable set. This secret is used to sign and validate all HMAC-SHA256 JWT tokens. There is no default fallback — this is an intentional security measure.

```bash
# Required in production
JWT_SECRET=<min-32-char-random-secret>

# Example (dev only — never reuse in prod)
JWT_SECRET=dev-local-secret-do-not-use-in-production
```

### ADMIN_USER Cannot Be Linked to Players

Enforced at the `PlayerService` layer. Any `POST /api/players` or `PATCH /api/players/{id}` request that attempts to link an `ADMIN_USER` account will receive `403 Forbidden`.

Rationale: Admin accounts are system management accounts, not participant profiles.

### Audit Trail

Every player record stores `created_by` (set on INSERT) and `updated_by` (updated on every PATCH). These are populated from the authenticated principal's username via Spring Security context. This provides a full audit trail of who created and last modified each player.

### PostgreSQL CVE Fix

The PostgreSQL JDBC driver was pinned to **42.7.11** during the security audit (Phase 7) to resolve two HIGH-severity CVEs present in older versions. The `build.gradle` dependency is explicitly version-pinned.

---

## Implementation Details

- **Entity:** `Player.java` — JPA entity with Lombok `@Getter`/`@Setter`/`@NoArgsConstructor`
- **Repository:** `PlayerRepository.java` — Spring Data JPA; includes `findAllByIsActive(boolean, Pageable)` for filtered listing
- **Mapper:** `PlayerMapper.java` — MapStruct compile-time mapper; maps `Player` → `PlayerDTO` with `email` derived from linked user
- **Service:** `PlayerService.java` — business logic, cache management, ADMIN guard, delete guard
- **Controller:** `PlayerController.java` — REST layer with `@PreAuthorize` for role-based access
- **DTOs:** `PlayerDTO`, `PlayerCreateDTO`, `PlayerUpdateDTO`, `PlayerStatusDTO` — all Java Records

### Test Coverage

- **48 tests total** — 24 service unit tests + 24 controller slice tests
- **Service tests:** `@ExtendWith(MockitoExtension.class)` with mocked `PlayerRepository` and `UserRepository`
- **Controller tests:** `@WebMvcTest(PlayerController.class)` with mocked `PlayerService`
- **Build status:** ✅ BUILD SUCCESSFUL

---

## Known Limitations

- `GET /api/players` does not currently support server-side filtering by `name` or `skillRating` range. Filtering is limited to `isActive`.
- `skillRating` is initialised to `baseSkillRating` (as a double) on creation. The match-performance algorithm that evolves it is implemented in a separate `calculation-service` pipeline.
- Streak counters (`currentStreak`, `longestStreak`) are updated by the match result processing flow, not by the Player API directly.

