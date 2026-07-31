# Player Management — API Contract
**Date:** 2026-05-15
**Version:** v4.x.x
**Status:** APPROVED

---

## Endpoints

### GET /api/players

**Description:** List all players, paginated. Optionally filter by active status.
**Authorization:** `isAuthenticated()` — any logged-in user, including one holding no roles
**Tags:** `Player Management`

#### Request

**Query Parameters:**
| Name     | Type    | Required | Default | Description                         |
|----------|---------|----------|---------|-------------------------------------|
| `page`   | Integer | No       | 0       | Page index (0-based)                |
| `size`   | Integer | No       | 20      | Page size                           |
| `active` | Boolean | No       | (all)   | Filter by `isActive` when provided  |

#### Response

**Success:** `200 OK`
**Body:** `Page<PlayerDTO>`
```json
{
  "content": [
    {
      "id": 1,
      "name": "João Silva",
      "email": "joao@example.com",
      "isActive": true,
      "skillRating": 7.5,
      "baseSkillRating": 7,
      "phoneNumber": "+351911000001",
      "currentStreak": 3,
      "longestStreak": 8,
      "linkedUserId": 12,
      "createdBy": "admin",
      "updatedBy": "master",
      "createdAt": "2026-01-10T08:00:00Z",
      "updatedAt": "2026-05-10T14:30:00Z"
    }
  ],
  "totalElements": 42,
  "totalPages": 3,
  "page": 0,
  "size": 20
}
```

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401    | Not authenticated |

---

### GET /api/players/{id}

**Description:** Retrieve a single player by ID.
**Authorization:** `isAuthenticated()`
**Tags:** `Player Management`

#### Request

**Path Variables:**
- `{id}` — Long — Player ID

#### Response

**Success:** `200 OK`
**Body:** `PlayerDTO`
```json
{
  "id": 1,
  "name": "João Silva",
  "email": "joao@example.com",
  "isActive": true,
  "skillRating": 7.5,
  "baseSkillRating": 7,
  "phoneNumber": "+351911000001",
  "currentStreak": 3,
  "longestStreak": 8,
  "linkedUserId": 12,
  "createdBy": "admin",
  "updatedBy": "master",
  "createdAt": "2026-01-10T08:00:00Z",
  "updatedAt": "2026-05-10T14:30:00Z"
}
```

**Notes:**
- `email` is `null` when the player has no linked `AppUser`
- `linkedUserId` is `null` when the player has no linked `AppUser`

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401    | Not authenticated |
| 404    | Player not found |

---

### POST /api/players

**Description:** Create a new player. Optionally link to an existing AppUser.
**Authorization:** `hasRole('ADMIN') or hasRole('MANAGER')`
**Tags:** `Player Management`

#### Request

**Request Body:** `PlayerCreateDTO` (required)
```json
{
  "name": "João Silva",           // String, required, 2–100 chars
  "baseSkillRating": 7,           // Integer, required, 1–10
  "phoneNumber": "+351911000001", // String, optional, max 20 chars
  "isActive": true,               // Boolean, optional — defaults to true in service
  "userId": 12                    // Long, optional — links to AppUser (non-ADMIN only)
}
```

#### Response

**Success:** `201 Created`
**Headers:** `Location: /api/players/{id}`
**Body:** `PlayerDTO` (see above)

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 400    | Validation failure (blank name, rating out of range, etc.) |
| 403    | Caller lacks MANAGER role |
| 404    | `userId` provided but AppUser not found |
| 409    | Attempting to link an ADMIN account to a player |

---

### PATCH /api/players/{id}

**Description:** Partially update a player's name, phone number, or linked user.
**Authorization:** `hasRole('ADMIN') or hasRole('MANAGER')`
**Tags:** `Player Management`

#### Request

**Path Variables:**
- `{id}` — Long — Player ID

**Request Body:** `PlayerUpdateDTO` (required — send only fields to change)
```json
{
  "name": "João M. Silva",        // String, optional, 2–100 chars — null = no change
  "phoneNumber": "+351911000002", // String, optional, max 20 chars — null = no change
  "userId": null                  // Long — null = unlink current user; value = link to that user
}
```

**Important:** `userId` semantics:
- Field **absent / null** in JSON → unlinks any existing user association
- Provide a value → links player to that AppUser (ADMIN accounts rejected)

> ⚠️ To keep `userId` unchanged, the service must treat missing key differently from explicit null.
> Frontend must only include `userId` in the body when a change is intended.

#### Response

**Success:** `200 OK`
**Body:** `PlayerDTO`

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 400    | Validation failure |
| 403    | Caller lacks required role |
| 404    | Player not found; or `userId` not found |
| 409    | Attempting to link an ADMIN account |

---

### PATCH /api/players/me

**Description:** Update your own player record. **Name and phone number only.**
**Authorization:** `isAuthenticated()`
**Tags:** `Player Management`

The player is resolved from the authenticated principal, never from a path variable or the body,
so there is no id a caller could change to edit somebody else's record. That is what lets this be
`isAuthenticated()` instead of an ownership check that could be got wrong — the same shape as
`POST /api/players/{id}/link-me` and the "my confirmation" endpoints.

#### Request

**Request Body:** `PlayerSelfUpdateDTO` (both fields optional — null leaves that field unchanged)
```json
{
  "name": "Ricardo",              // 2–100 chars
  "phoneNumber": "+351911111111"  // max 20 chars
}
```

> ⚠️ **This is not `PlayerUpdateDTO`.** That one also carries `userId` and `unlinkUser`, and reusing
> it here would let anybody re-point their own player at another account, or unlink themselves,
> through an endpoint whose only check is "you are authenticated". Those fields are absent from
> the record, so Jackson drops them if they are sent. `baseSkillRating` and `isActive` are absent
> for the same reason — a rating you could set yourself would not be a rating.

#### Response

**Success:** `200 OK`
**Body:** `PlayerDTO`

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 400    | `name` shorter than 2 or longer than 100; `phoneNumber` longer than 20 |
| 403    | Unauthenticated *(Spring Security — no body)* |
| 409    | The caller has no linked player — there is nothing to edit |

Evicts the players, player-profile, rankings and leaderboards caches: a renamed player appears in
the league table and on every scoresheet, so leaving those warm would serve the old name for the
cache TTL.

---

### PATCH /api/players/{id}/status

**Description:** Activate or deactivate a player.
**Authorization:** `hasRole('ADMIN') or hasRole('MANAGER')`
**Tags:** `Player Management`

#### Request

**Path Variables:**
- `{id}` — Long — Player ID

**Request Body:** `PlayerStatusDTO` (required)
```json
{
  "isActive": false   // Boolean, required
}
```

#### Response

**Success:** `200 OK`
**Body:** `PlayerDTO`

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 400    | `isActive` is null |
| 403    | Caller lacks required role |
| 404    | Player not found |

---

### POST /api/players/{id}/link-me

**Description:** Self-link endpoint allowing any authenticated user to link their own account to a player profile. No request body required — the caller's user ID is derived from the JWT principal.
**Authorization:** `isAuthenticated()` — any authenticated user. Accounts holding ADMIN are no longer rejected — see ROLES-API-CONTRACT.md.
**Tags:** `Player Management`

#### Request

**Path Variables:**
- `{id}` — Long — Player ID to link to the calling user

**Request Body:** None

#### Response

**Success:** `200 OK`
**Body:** `PlayerDTO` (see above — `linkedUserId` will reflect the calling user's ID)

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
  "updatedAt": "2026-05-15T10:30:00Z"
}
```

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401    | Not authenticated |
| 403    | Caller has the `ADMIN` role |
| 404    | Player not found |
| 409    | Player is already linked to the calling user — `"Player is already linked to your account"` |
| 409    | Player is already linked to a different user — `"Player is already linked to another user"` |
| 409    | Calling user is already linked to a different player — `"You are already linked to another player"` |
| 409    | Concurrent DB constraint violation (race condition guard) |

---

### DELETE /api/players/{id}

**Description:** Hard-delete a player. Only permitted when the player has never participated in a match.
**Authorization:** `hasRole('ADMIN')`
**Tags:** `Player Management`

#### Request

**Path Variables:**
- `{id}` — Long — Player ID

#### Response

**Success:** `204 No Content`

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 403    | Caller is not ADMIN |
| 404    | Player not found |
| 409    | Player has match history — hard delete not allowed |

---

## New/Modified DTOs

### PlayerDTO (response)

```
id:             Long     (never null)
name:           String   (never null)
email:          String   (nullable — from linked AppUser.email)
isActive:       boolean
skillRating:    double   (1.0–10.0, evolves with matches)
baseSkillRating:int      (1–10, set at creation, never modified by algorithm)
phoneNumber:    String   (nullable)
currentStreak:  int
longestStreak:  int
linkedUserId:   Long     (nullable)
createdBy:      String   (audit — username of creator)
updatedBy:      String   (audit — username of last updater)
createdAt:      Instant
updatedAt:      Instant
```

### PlayerCreateDTO (request → POST)

```
name:            @NotBlank @Size(min=2, max=100) String   (required)
baseSkillRating: @NotNull @Min(1) @Max(10) Integer        (required)
phoneNumber:     @Size(max=20) String                     (optional, nullable)
isActive:        Boolean                                  (optional, nullable — service defaults to true)
userId:          Long                                     (optional, nullable — links to AppUser)
```

### PlayerUpdateDTO (request → PATCH /{id})

```
name:        @Size(min=2, max=100) String   (optional, nullable — null = no change)
phoneNumber: @Size(max=20) String           (optional, nullable — null = no change)
userId:      Long                           (optional, nullable — null = unlink; value = link)
```

### PlayerSelfUpdateDTO (request → PATCH /me)

```
name:        @Size(min=2, max=100) String   (optional, nullable — null = no change)
phoneNumber: @Size(max=20) String           (optional, nullable — null = no change)
```

Deliberately a subset of `PlayerUpdateDTO`, missing the linking fields — see the endpoint above.

### PlayerStatusDTO (request → PATCH /{id}/status)

```
isActive: @NotNull Boolean   (required)
```

---

## Business Rules

1. `ADMIN` accounts **cannot** be linked to a player — service throws `403 Forbidden`
2. `email` is sourced from `AppUser.email` — **not stored** in the `players` table
3. `baseSkillRating` is set at creation and **never modified** by the calculation algorithm
4. `skillRating` (double) evolves automatically via match performance calculation
5. Inactive players (`isActive = false`) are **excluded from team generation**
6. Hard delete (`DELETE`) is blocked if the player has any `player_stats` records
7. **Self-link** (`POST /{id}/link-me`): each player can be linked to at most one user, and each user can be linked to at most one player. Violations return `409 Conflict` with a descriptive message.
8. **Self-link is idempotent from the 409 side** — re-calling with the same user/player already linked returns a `409` with `"Player is already linked to your account"`, not a silent 200.

---

## Breaking Changes

- [ ] No breaking changes — this is a new resource with no prior FE contract

---

## Frontend Migration Notes

- New resource — implement from scratch
- `email` field in `PlayerDTO` may be `null` — guard in UI before display
- `linkedUserId` may be `null` — do not assume every player has a user account
- For `PATCH /api/players/{id}`: only include `userId` in the JSON body when explicitly changing the link; omit otherwise
- `DELETE` is restricted to `ADMIN` — hide delete button from MANAGER UI
- `POST /api/players/{id}/link-me` requires **no request body** — just send the POST with a valid Bearer token. Show this option to any player who does not yet have a linked profile, including administrators.
- Handle the three distinct `409` messages so the UI can show a helpful error: `"Player is already linked to your account"`, `"Player is already linked to another user"`, `"You are already linked to another player"`.

