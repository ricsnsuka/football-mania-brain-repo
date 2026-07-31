# API Reference — Football Management System

**Version:** 1.0.0  
**Base URL:** `/api` (authenticated endpoints) · `/api/auth` (public)  
**Auth:** Bearer JWT — obtain via `POST /api/auth/login`  
**Date:** May 18, 2026 · **Updated:** June 29, 2026 (audited and corrected against implementation)
· July 29, 2026 (rankings & leaderboards, crowd MOTM voting, achievement badges added)

---

## Authentication

Base path: `/api/auth`

| Method | Path              | Description           | Auth   | Request Body      | Response                 |
|--------|-------------------|-----------------------|--------|-------------------|--------------------------|
| POST   | `/api/auth/login` | Login — get JWT token | Public | `LoginRequestDTO` | `LoginResponseDTO` (200) |

### LoginRequestDTO

```json
{
  "identifier": "joao",
  // required — username OR email
  "password": "secret123"
  // required
}
```

### LoginResponseDTO

```json
{
  "token": "<JWT>",
  "userId": 1,
  "username": "joao",
  "email": "joao@example.com",
  "role": "BASIC_USER",
  // DEPRECATED — highest-precedence legacy name, emitted for one release only
  "roles": [],
  // every grant held, sorted; empty for an unprivileged account
  "forcePasswordChange": false
}
```

> Read `roles`. `role` exists so a frontend deployed before this change keeps working, and will be
> removed — see `ROLES-API-CONTRACT.md`.

---

## Users

Base path: `/api/users`

| Method | Path                              | Description                         | Auth                | Request Body         | Response              |
|--------|-----------------------------------|-------------------------------------|---------------------|----------------------|-----------------------|
| GET    | `/api/users`                      | List all users (paginated)          | `ADMIN`        | —                    | `Page<UserDTO>` (200) |
| GET    | `/api/users/me`                   | Get own profile                     | Any authenticated   | —                    | `UserDTO` (200)       |
| GET    | `/api/users/{id}`                 | Get user by ID                      | `ADMIN` or own | —                    | `UserDTO` (200)       |
| POST   | `/api/users`                      | Create user                         | `ADMIN`        | `UserCreateDTO`      | `UserDTO` (201)       |
| POST   | `/api/users/register`             | Self-register new account           | Public              | `UserRegisterDTO`    | `UserDTO` (201)       |
| PATCH  | `/api/users/{id}`                 | Update profile (name, email)        | `ADMIN` or own | `UserUpdateDTO`      | `UserDTO` (200)       |
| PATCH  | `/api/users/{id}/role`            | Update role / active status         | `ADMIN`        | `AdminUserUpdateDTO` | `UserDTO` (200)       |
| DELETE | `/api/users/{id}`                 | Deactivate user (soft delete)       | `ADMIN`        | —                    | `204 No Content`      |
| POST   | `/api/users/{id}/change-password` | Change password                     | `ADMIN` or own | `ChangePasswordDTO`  | `204 No Content`      |

### UserDTO (response)

```json
{
  "id": 1,
  "username": "joao",
  "email": "joao@example.com",
  "firstName": "João",
  "lastName": "Silva",
  "roles": ["MANAGER"],
  "isActive": true,
  "forcePasswordChange": false,
  "createdAt": "2026-05-15T10:30:00Z"
}
```

### UserCreateDTO

```json
{
  "username": "joao",
  // required, 3-50 chars
  "email": "joao@example.com",
  // required, valid email
  "password": "secret123",
  // required, 8-100 chars
  "firstName": "João",
  // optional, max 100 chars
  "lastName": "Silva",
  // optional, max 100 chars
  "roles": ["MANAGER", "ORGANIZER"]
  // optional — any of "ORGANIZER" | "MANAGER" | "ADMIN"; omit or [] for an account
  // that is authenticated and nothing more
}
```

### UserRegisterDTO (public registration — never grants a role)

```json
{
  "username": "joao",
  // required, 3-50 chars
  "email": "joao@example.com",
  // required, valid email
  "password": "secret123",
  // required, 8-100 chars
  "firstName": "João",
  // optional, max 100 chars
  "lastName": "Silva"
  // optional, max 100 chars
}
```

### UserUpdateDTO

```json
{
  "firstName": "João",
  // optional, max 100 chars
  "lastName": "Silva",
  // optional, max 100 chars
  "email": "new@example.com"
  // optional, valid email
}
```

> All fields optional (safe PATCH). Only the owner or `ADMIN` can call this endpoint.

### AdminUserUpdateDTO

```json
{
  "roles": ["MANAGER", "ORGANIZER"],
  // optional — REPLACES the user's grants wholesale. [] strips them all;
  // omit the field entirely to leave them alone.
  "isActive": false
  // optional
}
```

### ChangePasswordDTO

```json
{
  "currentPassword": "old-secret",
  // required
  "newPassword": "new-secret-123"
  // required, 8-100 chars
}
```

---

## Players

Base path: `/api/players`

| Method | Path                          | Description                                                            | Auth                                    | Request Body      | Response                |
|--------|-------------------------------|------------------------------------------------------------------------|-----------------------------------------|-------------------|-------------------------|
| GET    | `/api/players`                | List players (paginated); filter by `?active=true/false`               | Any authenticated                       | —                 | `Page<PlayerDTO>` (200) |
| GET    | `/api/players/{id}`           | Get player by ID                                                       | Any authenticated                       | —                 | `PlayerDTO` (200)       |
| POST   | `/api/players`                | Create player                                                          | `MANAGER`           | `PlayerCreateDTO` | `PlayerDTO` (201)       |
| PATCH  | `/api/players/{id}`           | Partial update (name, phone, user link)                                | `MANAGER`           | `PlayerUpdateDTO` | `PlayerDTO` (200)       |
| PATCH  | `/api/players/{id}/status`    | Activate or deactivate player                                          | `MANAGER`           | `PlayerStatusDTO` | `PlayerDTO` (200)       |
| POST   | `/api/players/{id}/link-me`   | Self-link calling user to a player profile (no body; user ID from JWT) | Any authenticated | —                 | `PlayerDTO` (200)       |
| GET    | `/api/players/{id}/badges`    | Achievement badges earned, oldest first                                | Any authenticated                       | —                 | `List<PlayerBadgeDTO>` (200) |
| DELETE | `/api/players/{id}/user-link` | Unlink player from their associated user account                       | `ADMIN` only                       | —                 | `PlayerDTO` (200)       |
| DELETE | `/api/players/{id}`           | Hard delete (blocked if player has match stats)                        | `ADMIN` only                       | —                 | `204 No Content`        |

### PlayerDTO (all read/write responses)

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
  "totalMatchesPlayed": 12,
  "totalGoals": 8,
  "totalAssists": 5,
  "linkedUserId": 5,
  "createdBy": "admin",
  "updatedBy": "master_user",
  "createdAt": "2026-05-15T10:30:00Z",
  "updatedAt": "2026-05-15T11:00:00Z"
}
```

> `email` is sourced from the linked `AppUser` — not stored in `players`. `null` if no user is linked.  
> `totalMatchesPlayed`, `totalGoals`, and `totalAssists` are career totals maintained by `CalculationService` on every
> match completion.

### PlayerCreateDTO

```json
{
  "name": "João Silva",
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "isActive": true,
  "userId": 5
}
```

> `userId` is optional — links an `AppUser` to the player. `ADMIN` accounts are rejected (`403`).

### PlayerUpdateDTO

```json
{
  "name": "João M. Silva",
  "phoneNumber": "+351987654321",
  "userId": 5,
  "unlinkUser": true
}
```

> All fields optional. `unlinkUser: true` explicitly removes the user link. Sending `"userId": null` without
`unlinkUser: true` is a **no-op** (safe PATCH semantics).

### PlayerStatusDTO

```json
{
  "isActive": false
}
```

---

## Matches

Base path: `/api/matches`

| Method | Path                                              | Description                                                                             | Auth                          | Request Body           | Response                           |
|--------|---------------------------------------------------|-----------------------------------------------------------------------------------------|-------------------------------|------------------------|------------------------------------|
| POST   | `/api/matches`                                    | Create a match with manual teams (player count must match `matchType`)                  | `MANAGER` | `MatchCreateDTO`       | `MatchDTO` (201)                   |
| POST   | `/api/matches/manual`                             | Create a match with freely-sized teams (equal sizes only, no match-type count enforced) | `MANAGER` | `ManualMatchCreateDTO` | `MatchDTO` (201)                   |
| GET    | `/api/matches`                                    | List matches (filter: `seasonId`, `completed`, `matchType`)                             | Any authenticated             | —                      | `Page<MatchDTO>` (200)             |
| GET    | `/api/matches/{id}`                               | Get match by ID                                                                         | Any authenticated             | —                      | `MatchDTO` (200)                   |
| PATCH  | `/api/matches/{id}`                               | Update description / date / location                                                    | `MANAGER` | `MatchUpdateDTO`       | `MatchDTO` (200)                   |
| PATCH  | `/api/matches/{id}/complete`                      | Mark match completed — server computes ratings; compliance validation required          | `MANAGER` | `MatchCompleteDTO`     | `MatchDTO` (200)                   |
| PATCH  | `/api/matches/{id}/stats/live`                    | Live stat update (preview ratings, no win/loss bonuses)                                 | `MANAGER` | `MatchLiveUpdateDTO`   | `MatchLiveUpdateResponseDTO` (200) |
| PATCH  | `/api/matches/{id}/teams/{teamId}/stats/{statId}` | Amend a single player stat                                                              | `ADMIN`                  | `PlayerStatUpdateDTO`  | `PlayerStatDTO` (200)              |
| POST   | `/api/matches/{id}/recalculate`                   | Idempotently recalc ratings for one completed match                                     | `ADMIN`                  | —                      | `RecalculationResultDTO` (200)     |
| POST   | `/api/matches/recalculate`                        | Bulk recalc (matchIds / seasonId / all completed); per-match summary                    | `ADMIN`                  | `BulkRecalculationRequestDTO` (optional) | `BulkRecalculationResponseDTO` (200) |
| DELETE | `/api/matches/{id}`                               | Delete non-completed match                                                              | `ADMIN`                  | —                      | `204 No Content`                   |

### MatchDTO (response)

```json
{
  "id": 1,
  "description": "Friendly Friday",
  "matchDate": "2026-05-20T19:00:00Z",
  "location": "Central Park",
  "matchType": "SEVEN_A_SIDE",
  "isCompleted": false,
  "scoreTeamA": null,
  "scoreTeamB": null,
  "finalScore": null,
  "winningTeamId": null,
  "generationType": "BALANCED",
  "generationNotes": "BALANCED (greedy): avgA=7.14 avgB=7.12 Δ=0.02",
  "seasonId": 1,
  "teams": [
    {
      "id": 1,
      "name": "Team A",
      "teamOrder": 1,
      "playerStats": []
    },
    {
      "id": 2,
      "name": "Team B",
      "teamOrder": 2,
      "playerStats": []
    }
  ],
  "createdAt": "2026-05-17T10:00:00Z",
  "updatedAt": "2026-05-17T10:00:00Z"
}
```

### MatchCreateDTO

```json
{
  "description": "Friendly Friday",
  "matchDate": "2026-05-20T19:00:00Z",
  "location": "Central Park",
  "matchType": "SEVEN_A_SIDE",
  "generationType": "MANUAL",
  "seasonId": 1,
  "teams": [
    {
      "name": "Team A",
      "playerIds": [
        1,
        2,
        3,
        4,
        5,
        6,
        7
      ]
    },
    {
      "name": "Team B",
      "playerIds": [
        8,
        9,
        10,
        11,
        12,
        13,
        14
      ]
    }
  ]
}
```

> `matchType`: `"FIVE_A_SIDE"` | `"SEVEN_A_SIDE"` | `"ELEVEN_A_SIDE"`  
> `generationType`: `"MANUAL"` | `"BALANCED"` | `"RANDOM"` | `"SNAKE_DRAFT"` | `"FORM_BASED"` | `"CAPTAIN_PICK"` —
> always uppercase  
> `teams`: exactly 2 entries required (MANUAL only); each `playerIds` list must match the match type player count.  
> For algorithm-generated matches use the `POST /api/match-plans/{id}/generate/confirm` flow instead.

### MatchCompleteDTO

```json
{
  "scoreTeamA": 3,
  "scoreTeamB": 2,
  "winningTeamId": 1,
  "playerStats": [
    {
      "playerStatId": 10,
      "soloGoals": 1,
      "assistedGoals": 1,
      "penaltyGoals": 0,
      "assists": 1,
      "ownGoals": 0,
      "isMvp": true
    },
    {
      "playerStatId": 11,
      "soloGoals": 0,
      "assistedGoals": 0,
      "penaltyGoals": 0,
      "assists": 0,
      "ownGoals": 1,
      "isMvp": false
    }
  ]
}
```

> `winningTeamId` is optional — omit for a draw.  
> ⚠️ **`rating` is no longer accepted** — it is computed server-side by `CalculationService`.  
> ⚠️ **Compliance validation:** `(teamA soloGoals + assistedGoals + penaltyGoals) + teamB ownGoals` must equal
`scoreTeamA`, and vice versa for teamB. Returns `400` on mismatch.  
> After completion, `CalculationService` automatically updates all player skill ratings and streaks. Per-player `rating`
> is returned in the response `PlayerStatDTO`.

### MatchLiveUpdateDTO

```json
{
  "playerStats": [
    {
      "playerStatId": 10,
      "soloGoals": 1,
      "assistedGoals": 1,
      "penaltyGoals": 0,
      "assists": 1,
      "ownGoals": 0,
      "isMvp": false
    },
    {
      "playerStatId": 11,
      "soloGoals": 0,
      "assistedGoals": 0,
      "penaltyGoals": 0,
      "assists": 0,
      "ownGoals": 1,
      "isMvp": false
    }
  ]
}
```

> Fields in each `PlayerStatUpdateDTO` entry are **all optional except `playerStatId`** — only the provided fields are
> updated. Null = no change (safe PATCH semantics).  
> Live updates persist to DB but ratings returned are **preview only** (no win/loss bonus or goal-diff bonus applied).

### MatchLiveUpdateResponseDTO

```json
{
  "ratings": [
    {
      "playerStatId": 10,
      "playerId": 42,
      "playerName": "João Silva",
      "matchRating": 6.3
    },
    {
      "playerStatId": 11,
      "playerId": 43,
      "playerName": "Pedro Costa",
      "matchRating": 4.6
    }
  ]
}
```

> `matchRating` is a **preview** — does not include WIN_BONUS / LOSS_PENALTY or goal-diff bonus. Final ratings are
> computed at `PATCH /complete`.

### PlayerMatchRatingDTO

| Field          | Type   | Description                                        |
|----------------|--------|----------------------------------------------------|
| `playerStatId` | Long   | FK to `player_stats`                               |
| `playerId`     | Long   | FK to `players`                                    |
| `playerName`   | String | Display name                                       |
| `matchRating`  | double | Server-computed preview rating (no WIN/LOSS bonus) |

### ManualMatchCreateDTO

```json
{
  "description": "Casual 3v3 match",
  "matchDate": "2026-07-10T18:00:00Z",
  "location": "Park pitch B",
  "matchType": "SEVEN_A_SIDE",
  "seasonId": 1,
  "teams": [
    {
      "name": "Team A",
      "playerIds": [
        1,
        2,
        3
      ]
    },
    {
      "name": "Team B",
      "playerIds": [
        4,
        5,
        6
      ]
    }
  ]
}
```

> Unlike `MatchCreateDTO`, there is **no requirement** that `playerIds` count matches the `matchType` player count.  
> Enforced constraints:
> - Exactly 2 teams (`@Size(min=2, max=2)`)
> - Both teams must have the **same** number of players
> - At least 1 player per team (`@NotEmpty`)
> - No duplicate player IDs across teams
> - All player IDs must exist in the database
>
> `generationType` is always set to `"MANUAL"` by the server — not accepted as input.  
> `matchType`: `"FIVE_A_SIDE"` | `"SEVEN_A_SIDE"` | `"ELEVEN_A_SIDE"` (required, `@NotNull`)  
> `seasonId` is optional — falls back to current active season.

| Field         | Type                       | Required | Validation                                                  |
|---------------|----------------------------|----------|-------------------------------------------------------------|
| `description` | String                     | Yes      | `@NotBlank @Size(max=255)`                                  |
| `matchDate`   | Instant                    | Yes      | `@NotNull` — ISO-8601 UTC datetime                          |
| `location`    | String                     | No       | `@Size(max=255)` — venue name                               |
| `matchType`   | String                     | Yes      | `@NotNull` — `FIVE_A_SIDE`, `SEVEN_A_SIDE`, `ELEVEN_A_SIDE` |
| `seasonId`    | Long                       | No       | Falls back to current active season if null                 |
| `teams`       | `List<MatchTeamCreateDTO>` | Yes      | `@Size(min=2, max=2)` — exactly 2 teams                     |

**Error responses:**

| Status | Condition                                                                                                 |
|--------|-----------------------------------------------------------------------------------------------------------|
| 400    | `description` blank, `matchDate` null, `matchType` null/invalid, unequal team sizes, duplicate player IDs |
| 403    | Caller lacks `MANAGER` role                                                           |
| 404    | One or more player IDs not found                                                                          |

### MatchUpdateDTO

```json
{
  "description": "Friday Night Match",
  "matchDate": "2026-05-20T20:00:00Z",
  "location": "Old Trafford"
}
```

> All fields optional (safe PATCH). Only pre-completion changes allowed.

### PlayerStatDTO (within MatchTeamDTO)

```json
{
  "id": 10,
  "playerId": 42,
  "playerName": "João Silva",
  "goals": 2,
  "assists": 1,
  "ownGoals": 0,
  "rating": 8.5,
  "isMvp": true,
  "matchResult": "WIN"
}
```

> `goals` is the total goals contributed by this player (solo + assisted + penalty, stored as a single value).  
> `rating` is `null` until the match is completed. After completion it holds the server-computed value.

### PlayerStatUpdateDTO (PATCH .../stats/{statId} and within MatchCompleteDTO / MatchLiveUpdateDTO)

| Field           | Type    | Required | Notes                           |
|-----------------|---------|----------|---------------------------------|
| `playerStatId`  | Long    | **Yes**  | Must match an existing stat row |
| `soloGoals`     | Integer | No       | Null = no change                |
| `assistedGoals` | Integer | No       | Null = no change                |
| `penaltyGoals`  | Integer | No       | Null = no change                |
| `assists`       | Integer | No       | Null = no change                |
| `ownGoals`      | Integer | No       | Null = no change                |
| `isMvp`         | Boolean | No       | Null = no change                |

> ⚠️ **`rating` has been removed** — ratings are now always computed server-side.

---

## Match Rating Recalculation (Admin)

Two **`ADMIN`-only** endpoints re-run the rating engine over already-completed matches
**idempotently** (reverse-then-reapply). Use them after amending a stat post-completion, or after
a rating-model change, to bring persisted ratings back in sync. Both return `200 OK`.

Non-admin callers — `MANAGER`, `ORGANIZER`, or no roles at all — and unauthenticated callers both receive `403`.

### POST `/api/matches/{id}/recalculate` — single match

Recalculates one completed match. Idempotent for the target match's `PlayerStat.rating`, career
aggregates (`totalGoals` / `totalAssists` / `totalMatchesPlayed`), `skill_rating_history` rows,
and streaks (run-twice = run-once).

| Status | Trigger |
|--------|---------|
| `200`  | Recalculated (`status: "SUCCESS"`) **or** completed match with no player stats (`status: "SKIPPED"`) |
| `403`  | Caller is not `ADMIN` (or unauthenticated) |
| `404`  | Match `{id}` does not exist |
| `409`  | Match exists but is **not completed** (also optimistic-lock conflict) |

**Request:** no body.

**Response `200 OK` — `RecalculationResultDTO`:**

```json
{
  "matchId": 42,
  "status": "SUCCESS",
  "ratingsUpdated": 14,
  "message": "Recalculated 14 player ratings"
}
```

A completed match with no stats returns `200` with:

```json
{ "matchId": 42, "status": "SKIPPED", "ratingsUpdated": 0, "message": "No player stats — skipped" }
```

> ⚠️ **Idempotency caveat (skillRating & streaks).** `skillRating` is an EMA over the player's
> chronological match chain. The single-match endpoint is **exact** for a player's **most-recent**
> match, but only an **approximation** for a match in the middle of a player's chain (later matches'
> baselines become stale). `PlayerStat.rating`, career aggregates, and history are always exact. To
> fully reconcile a season, use the **bulk** endpoint (chronological replay).

### POST `/api/matches/recalculate` — bulk

Selection precedence: **`matchIds` → `seasonId` → all completed**. The batch processes each match in
its **own transaction**, in chronological order (`matchDate ASC NULLS LAST, id ASC`), so a per-match
failure never rolls back a sibling and the call **still returns `200 OK`**.

| Status | Trigger |
|--------|---------|
| `200`  | Batch executed — inspect `results[]` (per-match `SUCCESS` / `FAILED` / `SKIPPED`) |
| `400`  | Both `matchIds` **and** `seasonId` supplied, or a `matchIds` element is null / `< 1` |
| `403`  | Caller is not `ADMIN` (or unauthenticated) |
| `404`  | `seasonId` supplied but the season does not exist |

> Per-match issues are **not** HTTP errors: a non-existent explicit `matchId`, a non-completed match,
> or an engine error appears as a `FAILED` entry in `results[]`; a completed match with no stats
> appears as `SKIPPED`. `succeeded + failed` may be `< totalRequested` when `SKIPPED` items exist.

**Request — `BulkRecalculationRequestDTO` (body optional; `{}` / absent = all completed matches):**

```json
{
  "matchIds": [42, 43, 44],
  "seasonId": null
}
```

**Response `200 OK` — `BulkRecalculationResponseDTO`:**

```json
{
  "totalRequested": 3,
  "succeeded": 2,
  "failed": 1,
  "results": [
    { "matchId": 42, "status": "SUCCESS", "ratingsUpdated": 14, "message": "Recalculated 14 player ratings" },
    { "matchId": 43, "status": "SUCCESS", "ratingsUpdated": 14, "message": "Recalculated 14 player ratings" },
    { "matchId": 44, "status": "FAILED",  "ratingsUpdated": 0,  "message": "Match not found" }
  ]
}
```

### RecalculationResultDTO (single match & per-item in bulk)

| Field            | Type   | Nullable | Notes                                                            |
|------------------|--------|----------|------------------------------------------------------------------|
| `matchId`        | Long   | No       | The match this result refers to                                  |
| `status`         | String | No       | `"SUCCESS"` \| `"FAILED"` \| `"SKIPPED"`                          |
| `ratingsUpdated` | int    | No       | `PlayerStat` rows recomputed; `0` on `FAILED` / `SKIPPED`        |
| `message`        | String | No       | Human-readable outcome                                           |

### BulkRecalculationRequestDTO

| Field      | Type         | Required | Validation                                | Notes                             |
|------------|--------------|----------|-------------------------------------------|-----------------------------------|
| `matchIds` | List\<Long\> | No       | `@Size(max=500)`, elements `@NotNull @Positive` | Explicit list; empty/absent = not selected |
| `seasonId` | Long         | No       | `@Positive`                               | All completed matches in the season |

> Mutually exclusive — supplying both `matchIds` and `seasonId` → `400`.

### BulkRecalculationResponseDTO

| Field            | Type                            | Nullable | Notes                            |
|------------------|---------------------------------|----------|----------------------------------|
| `totalRequested` | int                             | No       | Matches the request resolved to  |
| `succeeded`      | int                             | No       | Count with `status == SUCCESS`   |
| `failed`         | int                             | No       | Count with `status == FAILED`    |
| `results`        | List\<RecalculationResultDTO\>  | No       | One entry per processed match    |

> See [`MATCH-RATING-RECALCULATION-API-CONTRACT.md`](./MATCH-RATING-RECALCULATION-API-CONTRACT.md)
> for the full idempotency strategy and the mid-chain reconciliation rationale.

---

## Match Plans

Base path: `/api/match-plans`

| Method | Path                                             | Description                                                          | Auth                          | Request Body            | Response                            |
|--------|--------------------------------------------------|----------------------------------------------------------------------|-------------------------------|-------------------------|-------------------------------------|
| POST   | `/api/match-plans`                               | Create plan and open availability poll                               | `MANAGER` | `MatchPlanCreateDTO`    | `MatchPlanDTO` (201)                |
| GET    | `/api/match-plans`                               | List plans (filter: `status`)                                        | Any authenticated             | —                       | `Page<MatchPlanDTO>` (200)          |
| GET    | `/api/match-plans/{id}`                          | Get plan by ID                                                       | Any authenticated             | —                       | `MatchPlanDTO` (200)                |
| PATCH  | `/api/match-plans/{id}`                          | Update plan details                                                  | `MANAGER` | `MatchPlanUpdateDTO`    | `MatchPlanDTO` (200)                |
| PATCH  | `/api/match-plans/{id}/status`                   | Transition status (PENDING→CONFIRMED/CANCELLED, CONFIRMED→CANCELLED) | `MANAGER` | `MatchPlanStatusDTO`    | `MatchPlanDTO` (200)                |
| DELETE | `/api/match-plans/{id}`                          | Delete a PENDING plan                                                | `ADMIN`                  | —                       | `204 No Content`                    |
| GET    | `/api/match-plans/{id}/confirmations`            | List all player confirmations (filter: `status`)                     | Any authenticated             | —                       | `List<PlayerConfirmationDTO>` (200) |
| POST   | `/api/match-plans/{id}/confirmations/me`         | Self-confirm or decline availability                                 | Any authenticated             | `ConfirmationUpsertDTO` | `PlayerConfirmationDTO` (200)       |
| GET    | `/api/match-plans/{id}/confirmations/me`         | Get your own confirmation entry                                      | Any authenticated             | —                       | `PlayerConfirmationDTO` (200)       |
| PATCH  | `/api/match-plans/{id}/confirmations/{playerId}` | Admin override: set a player's confirmation status                   | `MANAGER` | `ConfirmationUpsertDTO` | `PlayerConfirmationDTO` (200)       |
| POST   | `/api/match-plans/{id}/generate`                 | Preview generated teams (not persisted)                              | `MANAGER` | — (query params)        | `MatchPreviewDTO` (200)             |
| POST   | `/api/match-plans/{id}/generate/confirm`         | Confirm preview and create the match                                 | `MANAGER` | — (query params)        | `MatchDTO` (201)                    |

> **Generate query params:**
>
> | Param | Type | Default | Description |
> |-------|------|---------|-------------|
> | `generationType` | String | `BALANCED` | Algorithm to use — see table below |
> | `params[formWindow]` | int (≥1) | `5` | `FORM_BASED` only — number of recent rated matches to consider per player |
> | `params[captainAId]` | Long | auto | `CAPTAIN_PICK` only — player ID for Team A captain; auto = highest-rated |
> | `params[captainBId]` | Long | auto | `CAPTAIN_PICK` only — player ID for Team B captain; auto = 2nd-highest |
>
> | `generationType` | Status | Data Required | Notes |
> |------------------|--------|---------------|-------|
> | `BALANCED` | ✅ Active | `skillRating` | Greedy bin-packing — default, recommended |
> | `RANDOM` | ✅ Active | none | Pure random shuffle — preview and confirm produce **different** shuffles |
> | `SNAKE_DRAFT` | ✅ Active | `skillRating` | Alternating snake pick by rating rank |
> | `FORM_BASED` | ✅ Active | `player_stats` history | Linearly-weighted recent match ratings; falls back to `skillRating` for new players |
> | `STREAK_AWARE` | ⚠️ Pending | `currentStreak` | Returns `422` — will be enabled once CalculationService populates streaks |
> | `CAPTAIN_PICK` | ✅ Active | `skillRating` | Server-side snake draft; auto-selects top-2 captains unless overridden via params |
>
> See [`docs/features/TEAM_GENERATION_FEATURE.md`](../features/TEAM_GENERATION_FEATURE.md) for full algorithm details.

### MatchPlanDTO (response)

```json
{
  "id": 5,
  "title": "Friday Night Match",
  "matchType": "SEVEN_A_SIDE",
  "location": "Central Park",
  "proposedDate": "2026-05-23",
  "confirmationDeadline": "2026-05-22T18:00:00Z",
  "status": "PENDING",
  "confirmedCount": 8,
  "minPlayersRequired": 14,
  "description": "Bring boots",
  "createdBy": "admin",
  "playersNeeded": 6,
  "pollOpen": true,
  "createdAt": "2026-05-17T10:00:00Z",
  "updatedAt": "2026-05-17T10:00:00Z"
}
```

> `playersNeeded` = `minPlayersRequired - confirmedCount` (floored at 0)  
> `pollOpen` = `true` when `status == PENDING` and deadline has not passed

### MatchPlanCreateDTO

```json
{
  "title": "Friday Night Match",
  "matchType": "SEVEN_A_SIDE",
  "location": "Central Park",
  "proposedDate": "2026-05-23",
  "confirmationDeadline": "2026-05-22T18:00:00Z",
  "description": "Bring boots",
  "minPlayersRequired": 14
}
```

> `proposedDate` must be in the **future** (`@Future`). `matchType`: `"FIVE_A_SIDE"` | `"SEVEN_A_SIDE"` |
`"ELEVEN_A_SIDE"`.

### MatchPlanUpdateDTO

```json
{
  "title": "Updated Title",
  "location": "New Venue",
  "proposedDate": "2026-05-24",
  "confirmationDeadline": "2026-05-23T18:00:00Z",
  "description": "Updated notes"
}
```

> All fields optional (safe PATCH). Only allowed when plan is `PENDING`.

### ConfirmationUpsertDTO

```json
{
  "status": "CONFIRMED",
  "notes": "Will be 10 minutes late"
}
```

> `status`: `"CONFIRMED"` | `"DECLINED"` | `"PENDING"`. `notes` optional, max 500 chars.

### PlayerConfirmationDTO (response)

```json
{
  "id": 33,
  "matchPlanId": 5,
  "playerId": 42,
  "playerName": "João Silva",
  "status": "CONFIRMED",
  "notes": "Will be 10 minutes late",
  "confirmedAt": "2026-05-17T12:00:00Z",
  "confirmationRank": 11,
  "isStarter": false,
  "waitlistPosition": 1
}
```

**Starter vs reserve.** A plan may hold more confirmations than the match needs. Team generation
takes the first `N` in confirmation order — 10 / 14 / 22 by match type — and the rest are
reserves. These three fields expose that ordering, which the backend has always used but never
reported:

| Field | Meaning |
|-------|---------|
| `confirmationRank` | 1-based place among **CONFIRMED** players, in `confirmedAt` order |
| `isStarter` | `true` when `confirmationRank` is within the required count |
| `waitlistPosition` | 1-based place in the reserve queue — **1 means next to come in** |

All three are `null` for anyone not `CONFIRMED`: a pending or declined player holds no place in
the queue, and a `0` would invite a client to render one. `waitlistPosition` is also `null` for
starters.

They are derived from the same ordering team generation uses, not stored, so the badge a player
sees cannot drift from the side they end up on. Withdrawing clears `confirmedAt`, which removes
that player from the ordering entirely and moves everyone behind them up — so a reserve becoming
a starter needs no separate promotion step.

### MatchPreviewDTO (team generation preview)

```json
{
  "matchPlanId": 5,
  "matchType": "SEVEN_A_SIDE",
  "generationType": "BALANCED",
  "location": "Central Park",
  "proposedDate": "2026-05-23",
  "generationNotes": "BALANCED (greedy): avgA=7.14 avgB=7.12 Δ=0.02",
  "teamARatingAvg": 7.14,
  "teamBRatingAvg": 7.12,
  "ratingDelta": 0.02,
  "teams": [
    {
      "name": "Team A",
      "ratingAvg": 7.14,
      "players": [
        {
          "playerId": 1,
          "playerName": "João Silva",
          "skillRating": 8.5
        }
      ]
    },
    {
      "name": "Team B",
      "ratingAvg": 7.12,
      "players": [
        {
          "playerId": 2,
          "playerName": "Pedro Costa",
          "skillRating": 7.0
        }
      ]
    }
  ]
}
```

---

## Draft Sessions

Base path: `/api/draft-sessions`

> Interactive Captain Pick draft session. Allows two human captains to take turns picking players via API calls,
> producing a live draft experience. Created from a `CONFIRMED` match plan; on confirmation a real `Match` is created with
`generationType=CAPTAIN_PICK`.

| Method | Path                               | Description                                                                   | Auth                          | Request Body            | Response                             |
|--------|------------------------------------|-------------------------------------------------------------------------------|-------------------------------|-------------------------|--------------------------------------|
| GET    | `/api/draft-sessions`              | List all draft sessions                                                       | Any authenticated             | —                       | `List<DraftSessionDTO>` (200)        |
| POST   | `/api/draft-sessions`              | Create draft session from a CONFIRMED match plan                              | `MANAGER` | `DraftSessionCreateDTO` | `DraftSessionDTO` (201)              |
| GET    | `/api/draft-sessions/{id}`         | Get current draft state                                                       | Any authenticated             | —                       | `DraftSessionDTO` (200)              |
| GET    | `/api/draft-sessions/{id}/events`  | Subscribe to real-time draft events via SSE                                   | Any authenticated             | —                       | `text/event-stream` (200)            |
| POST   | `/api/draft-sessions/{id}/pick`    | Submit a captain's pick for the current turn                                  | Any authenticated             | `DraftPickDTO`          | `DraftSessionDTO` (200)              |
| POST   | `/api/draft-sessions/{id}/confirm` | Finalize COMPLETED draft and create the Match                                 | `MANAGER` | —                       | `MatchDTO` (201)                     |
| DELETE | `/api/draft-sessions/{id}`         | Cancel an OPEN or COMPLETED draft session (soft-cancel → `CANCELLED`)         | `MANAGER` | —                       | `204 No Content`                     |
| GET    | `/api/draft-sessions/summary`      | List all draft sessions (all statuses) as lightweight summaries, newest-first | `ADMIN` only             | —                       | `List<DraftSessionSummaryDTO>` (200) |
| DELETE | `/api/draft-sessions/{id}/purge`   | Hard-delete a draft session row (irreversible)                                | `ADMIN` only             | —                       | `204 No Content`                     |

> **Admin endpoints (added 2026-07-02):**
> - `GET /api/draft-sessions/summary` returns **all** sessions (all statuses) sorted `createdAt` DESC, without the heavy
    player arrays. Errors: `401`, `403`.
> - `DELETE /api/draft-sessions/{id}/purge` **hard-deletes** the row (and its `@ElementCollection` child rows), distinct
    from the soft-cancel `DELETE /{id}`. Errors: `401`, `403`, `404` (session not found), `409` (session status is
    `CONVERTED` — linked to a match, cannot be purged).

### Session Lifecycle

```
OPEN → (all players picked) → COMPLETED → (confirm) → CONVERTED
OPEN or COMPLETED → (cancel) → CANCELLED
```

`CONVERTED` and `CANCELLED` are terminal states.

### DraftSessionCreateDTO (request)

```json
{
  "matchPlanId": 5,
  "captainAId": 42,
  "captainBId": 17
}
```

| Field         | Type   | Required | Notes                                                                                 |
|---------------|--------|----------|---------------------------------------------------------------------------------------|
| `matchPlanId` | `Long` | **Yes**  | Must reference a `CONFIRMED` match plan                                               |
| `captainAId`  | `Long` | No       | Optional — auto-selects highest-rated confirmed player if absent                      |
| `captainBId`  | `Long` | No       | Optional — auto-selects second-highest-rated if absent; must differ from `captainAId` |

### DraftPickDTO (request)

```json
{ "playerId": 7 }
```

| Field      | Type   | Required | Notes                                                  |
|------------|--------|----------|--------------------------------------------------------|
| `playerId` | `Long` | **Yes**  | Must be present in the `remaining` pool of the session |

### DraftSessionDTO (response)

```json
{
  "id": 1,
  "matchPlanId": 5,
  "matchPlanTitle": "Friday Night Match",
  "status": "OPEN",
  "captainA": { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 },
  "captainB": { "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 },
  "currentTurn": "A",
  "teamA": [
    { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 }
  ],
  "teamB": [
    { "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 }
  ],
  "remaining": [
    { "playerId": 7, "playerName": "Carlos Matos", "skillRating": 8.5 }
  ],
  "totalPlayers": 14,
  "picksRemaining": 12,
  "expiresAt": null,
  "createdAt": "2026-05-20T10:00:00Z",
  "updatedAt": "2026-05-20T10:00:00Z"
}
```

| Field            | Type                   | Notes                                                       |
|------------------|------------------------|-------------------------------------------------------------|
| `id`             | `Long`                 | Auto-generated session ID                                   |
| `matchPlanId`    | `Long`                 | Source match plan ID                                        |
| `matchPlanTitle` | `String`               | Title from the match plan                                   |
| `status`         | `String`               | `"OPEN"` \| `"COMPLETED"` \| `"CANCELLED"` \| `"CONVERTED"` |
| `captainA`       | `DraftPlayerDTO`       | Fixed Team A captain (id, name, skillRating)                |
| `captainB`       | `DraftPlayerDTO`       | Fixed Team B captain                                        |
| `currentTurn`    | `String`               | `"A"` or `"B"` when status is `OPEN`; `null` otherwise      |
| `teamA`          | `List<DraftPlayerDTO>` | Players picked for Team A (captain always first)            |
| `teamB`          | `List<DraftPlayerDTO>` | Players picked for Team B (captain always first)            |
| `remaining`      | `List<DraftPlayerDTO>` | Still-available pool, sorted by `skillRating` DESC          |
| `totalPlayers`   | `int`                  | Total confirmed players (both teams + remaining)            |
| `picksRemaining` | `int`                  | Number of picks still required (`remaining.size()`)         |
| `expiresAt`      | `Instant`              | Optional expiry — `null` if no expiry set                   |
| `createdAt`      | `Instant`              | Session creation timestamp                                  |
| `updatedAt`      | `Instant`              | Last modification timestamp                                 |

> `POST /{id}/confirm` returns a full `MatchDTO` (see Matches section) with `generationType=CAPTAIN_PICK` and
`generationNotes` set to `"CAPTAIN_PICK: captainA=<name> captainB=<name> avgA=<x> avgB=<y> Δ=<z>"`.

### DraftPlayerDTO (embedded)

| Field         | Type     | Notes                         |
|---------------|----------|-------------------------------|
| `playerId`    | `Long`   | Player ID                     |
| `playerName`  | `String` | Display name                  |
| `skillRating` | `double` | Player's current skill rating |

### DraftSessionSummaryDTO (response — admin summary)

Returned by `GET /api/draft-sessions/summary`. Lightweight projection — no player arrays.

```json
{
  "id": 42,
  "matchPlanId": 7,
  "matchPlanTitle": "Friday Night 7-a-side",
  "status": "OPEN",
  "captainAName": "Alice",
  "captainBName": "Bob",
  "currentTurn": "A",
  "totalPlayers": 14,
  "picksRemaining": 8,
  "createdBy": "admin",
  "expiresAt": "2026-05-08T20:00:00Z",
  "createdAt": "2026-05-08T18:30:00Z",
  "updatedAt": "2026-05-08T18:45:00Z"
}
```

| Field            | Type      | Nullable | Notes                                                       |
|------------------|-----------|----------|-------------------------------------------------------------|
| `id`             | `Long`    | No       | Draft session ID                                            |
| `matchPlanId`    | `Long`    | No       | Source match plan ID                                        |
| `matchPlanTitle` | `String`  | No       | Associated match plan name                                  |
| `status`         | `String`  | No       | `"OPEN"` \| `"COMPLETED"` \| `"CANCELLED"` \| `"CONVERTED"` |
| `captainAName`   | `String`  | No       | Team A captain display name                                 |
| `captainBName`   | `String`  | No       | Team B captain display name                                 |
| `currentTurn`    | `String`  | **Yes**  | `"A"`/`"B"` only when `OPEN`; `null` otherwise              |
| `totalPlayers`   | `int`     | No       | teamA + teamB + remaining sizes                             |
| `picksRemaining` | `int`     | No       | Remaining pool size                                         |
| `createdBy`      | `String`  | **Yes**  | Creator username; may be `null`                             |
| `expiresAt`      | `Instant` | **Yes**  | Optional expiry; `null` if unset                            |
| `createdAt`      | `Instant` | No       | Session creation timestamp                                  |
| `updatedAt`      | `Instant` | No       | Last modification timestamp                                 |

### Player Count Requirements

| Match Type      | Total Confirmed Players Required |
|-----------------|----------------------------------|
| `FIVE_A_SIDE`   | 10                               |
| `SEVEN_A_SIDE`  | 14                               |
| `ELEVEN_A_SIDE` | 22                               |

---

## Privacy (GDPR data-subject rights)

Base path: `/api/privacy`

| Method | Path                              | Description                                                  | Auth              | Request Body | Response                       |
|--------|-----------------------------------|--------------------------------------------------------------|-------------------|--------------|--------------------------------|
| GET    | `/api/privacy/me/export`          | Download a copy of your own personal data (Art. 15 / 20)     | Any authenticated | —            | `PersonalDataExportDTO` (200)  |
| DELETE | `/api/privacy/me`                 | Erase your own personal data (Art. 17). Irreversible          | Any authenticated | —            | `204 No Content`               |
| GET    | `/api/privacy/players/{id}/export`| Export another subject's data — for requests actioned on their behalf | `ADMIN` only | —      | `PersonalDataExportDTO` (200)  |
| DELETE | `/api/privacy/players/{id}`       | Erase another subject's data. Irreversible                    | `ADMIN` only | —            | `204 No Content`               |

The `/me` endpoints take their subject from the authenticated principal, never from a request
parameter — there is no value a caller can change to reach another person's record. The
`/players/{id}` endpoints exist because a player added by an admin may have no account at all and
therefore no way to make the request themselves; they are `ADMIN`-only rather than
`MANAGER` because one reads a full personal record and the other cannot be undone.

Both export responses carry `Content-Disposition: attachment` — the portability right is that the
subject walks away with a file.

**Error responses**

| Status | When |
|--------|------|
| `403 Forbidden` | An `ADMIN` attempting to erase their own account (another administrator must action it), or a non-admin calling a `/players/{id}` endpoint |
| `404 Not Found` | No such user or player |
| `409 Conflict` | The record has already been erased (`players.anonymized_at` is set) |

### PersonalDataExportDTO (response)

`account` is `null` for a player who never had a login; `player` is `null` for an account not yet
linked to a player. At least one is always present. The four lists are always arrays, empty rather
than absent.

```json
{
  "generatedAt": "2026-07-28T12:00:00Z",
  "account": {
    "id": 10, "username": "j.silva", "email": "j.silva@example.com",
    "firstName": "Joao", "lastName": "Silva", "roles": [],
    "active": true, "createdAt": "2026-01-01T00:00:00Z"
  },
  "player": {
    "id": 5, "name": "Joao Silva", "phoneNumber": "+351912345678", "active": true,
    "skillRating": 7.4, "baseSkillRating": 7,
    "currentStreak": 2, "longestStreak": 5,
    "totalMatchesPlayed": 30, "totalGoals": 12, "totalAssists": 9,
    "createdAt": "2026-01-01T00:00:00Z", "updatedAt": "2026-07-01T00:00:00Z"
  },
  "matches": [
    {
      "matchId": 100, "matchDescription": "Friday night",
      "matchDate": "2026-05-01T19:00:00Z", "location": "Pitch 2",
      "teamName": "Reds", "goals": 2, "assists": 1, "ownGoals": 0,
      "rating": 7.8, "mvp": true, "result": "WIN"
    }
  ],
  "goals": [
    {
      "matchId": 100, "matchDate": "2026-05-01T19:00:00Z",
      "role": "SCORER", "minute": 12, "ownGoal": false, "penalty": false,
      "description": null
    }
  ],
  "ratingHistory": [
    {
      "matchId": 100, "ratingBefore": 7.2, "ratingAfter": 7.4,
      "changeAmount": 0.2, "reason": "Match 100 result",
      "createdAt": "2026-05-01T21:00:00Z"
    }
  ],
  "availability": [
    {
      "matchPlanId": 20, "matchPlanTitle": "Friday 7-a-side",
      "proposedDate": "2026-05-01", "status": "CONFIRMED",
      "notes": "Can only make the second half",
      "confirmedAt": "2026-04-28T10:00:00Z"
    }
  ]
}
```

The export is scoped to *data about this person*, not *data this person can see*: a match appears
as the subject's own line in it rather than the full scoresheet, and `role` names the subject's
part in a goal without naming the counterpart. `password` is deliberately absent — a bcrypt hash is
not portable data.

See [PRIVACY_AND_DATA_PROTECTION.md](../features/PRIVACY_AND_DATA_PROTECTION.md) for what erasure
changes, what it deliberately leaves alone, and why it anonymises instead of deleting.

---

## Achievement Badges

| Method | Path | Description | Auth | Request Body | Response |
|--------|------|-------------|------|--------------|----------|
| GET | `/api/players/{id}/badges` | Badges this player has earned, oldest first | Any authenticated | — | `List<PlayerBadgeDTO>` (200) |

Awarding has no endpoint — it happens in `MatchEventListener` after a match's ratings are
recalculated, so thresholds are read from career totals the match is already folded into.

**Awards are permanent.** No revocation path: a later downward stat amendment does not remove one.
`uq_player_badges UNIQUE (player_id, badge)` plus insert-only awarding is what makes that safe,
and it is load-bearing — the listener retries on optimistic-lock conflicts and a bulk recalculation
re-runs every completed match, so re-evaluation is routine rather than exceptional.

**Existing players are not backfilled**, so the first completed match after deployment awards each
participant everything they already qualify for at once.

```json
[
  { "badge": "FIRST_MATCH", "displayName": "First match",
    "awardedAt": "2026-03-01T20:00:00Z", "matchId": 80 },
  { "badge": "FIRST_GOAL", "displayName": "First goal",
    "awardedAt": "2026-03-08T20:00:00Z" }
]
```

Catalogue: `FIRST_MATCH`, `TEN_MATCHES`, `FIFTY_MATCHES`, `FIRST_GOAL`, `TEN_GOALS`,
`FIFTY_GOALS`, `FIRST_ASSIST`, `WIN_STREAK_5` (from `longestStreak`), `FIRST_MVP` (the
administrator's `is_mvp`, not the crowd MOTM result).

`displayName` is sent so a new badge needs no frontend change; switch on `badge`. `matchId` is
**omitted when absent** (`non_null` inclusion), not null. An unknown player id returns `[]` rather
than 404.

> See [`BADGES-API-CONTRACT.md`](./BADGES-API-CONTRACT.md) for the idempotency argument and the
> privacy treatment.

---

## Crowd MOTM Voting

| Method | Path | Description | Auth | Request Body | Response |
|--------|------|-------------|------|--------------|----------|
| GET | `/api/matches/{id}/mvp-vote` | Poll state for this match, from the caller's point of view | Any authenticated | — | `MvpVoteSummaryDTO` (200) |
| POST | `/api/matches/{id}/mvp-vote` | Cast or change your vote | Any authenticated | `MvpVoteCreateDTO` | `MvpVoteSummaryDTO` (200) |

**Separate from `player_stats.is_mvp`**, which stays the administrator's pick. The crowd's answer is
`matches.crowd_mvp_player_id`. Keeping them apart is what allows "the admin picked X, the players
picked Y" — and the leaderboards' `mostMvps` still counts `is_mvp` only.

**Only players who appeared may vote, and not for themselves.** Eligibility is checked against
`player_stats`; that is what stops the vote being brigadable by anyone with an account. One vote per
voter is a UNIQUE constraint, not a service check — a check-then-insert races. Voting again while
the window is open replaces the previous choice.

**The window runs from completion, 24h by default** (`mvp_voting_closes_at`, set at completion — not
derived from `matchDate`, which can be backdated). The length is admin-configurable, and because it
is stamped at completion a change never affects a poll that is already open. Read
`mvpVotingClosesAt` rather than assuming 24 hours. Matches completed before V14 have no window and
are permanently closed.

**A tie produces no winner.** `crowd_mvp_player_id` stays unset and the match is still marked
resolved; `tied: true` says which it was. With 8–14 voters this will not be rare, and the UI should
present it as a normal outcome.

```json
{
  "matchId": 42,
  "votingOpen": true,
  "votingClosesAt": "2026-07-30T16:00:49Z",
  "resolved": false,
  "tied": false,
  "canVote": true,
  "myVotePlayerId": 7,
  "totalVotes": 5,
  "tally": [
    { "playerId": 7, "playerName": "Ricardo Nsuka", "votes": 3 },
    { "playerId": 4, "playerName": "João Silva", "votes": 2 }
  ]
}
```

`crowdMvpPlayerId`, `crowdMvpPlayerName`, `myVotePlayerId` and `votingClosesAt` are **omitted when
null** (`default-property-inclusion: non_null`) — branch on `resolved`, `tied` and `canVote`, which
are always present. `tally` is ordered but **leading is not winning** until `resolved`.

**Errors:** `400` self-vote / candidate not in the match / no linked player; `403` caller did not
play, or unauthenticated; `404` no such match; `409` not completed, no window, window closed, or a
concurrent duplicate first vote.

> See [`MOTM-API-CONTRACT.md`](./MOTM-API-CONTRACT.md) for the resolution job, the claim pattern and
> the privacy treatment of votes.

---

## Rankings & Leaderboards

Two read-only endpoints over data `CalculationService` already maintains. Neither returns contact
details, so unlike `/api/players` there is no PII redaction and no role-dependent response shape.

| Method | Path                | Description                                                          | Auth              | Request Body | Response                |
|--------|---------------------|----------------------------------------------------------------------|-------------------|--------------|-------------------------|
| GET    | `/api/rankings`     | League table by `skillRating` (query: `includeInactive`, default `false`) | Any authenticated | —            | `RankingsDTO` (200)     |
| GET    | `/api/leaderboards` | Category tops — goals, assists, MVPs, streaks (query: `limit`, clamped; default `5` and ceiling `25` are admin-configurable) | Any authenticated | —            | `LeaderboardsDTO` (200) |

**All-time only — there is no `seasonId` parameter.** `Player`'s totals are career-wide; a
season-scoped table would have to go through `skill_rating_history`. A parameter that accepted a
season and quietly ignored it would be worse than not having one.

**A player needs 3 completed matches by default to be given a rank.** Below that they are still
listed — `qualified: false`, with no `rank` field — and sort after every ranked player regardless of
rating. Otherwise a single lucky match tops the table. The threshold is **admin-configurable**, so
`3` is a default and not a constant; it is reported as `minimumMatchesToQualify` in every response,
which is the only correct source for it. It does **not** apply to leaderboards: goals and assists
are counting stats, not rates.

Deactivated players are excluded from the table (the group as it is now) but included in the
leaderboards (records, which outlast the player).

### RankingsDTO (response)

Not paginated. Ordering is `skillRating DESC, totalMatchesPlayed DESC, name ASC, id ASC` — fully
deterministic, because every new player sits on exactly 5.00 and a table that reorders between
identical requests reads as a bug.

```json
{
  "minimumMatchesToQualify": 3,
  "qualifiedCount": 2,
  "totalCount": 3,
  "entries": [
    {
      "rank": 1, "qualified": true,
      "playerId": 10, "playerName": "Joao Silva",
      "skillRating": 8.25, "played": 12,
      "wins": 7, "draws": 2, "losses": 3,
      "goals": 9, "assists": 4,
      "currentStreak": 2, "longestStreak": 5
    },
    {
      "qualified": false,
      "playerId": 12, "playerName": "New Player",
      "skillRating": 9.90, "played": 1,
      "wins": 1, "draws": 0, "losses": 0,
      "goals": 3, "assists": 0,
      "currentStreak": 1, "longestStreak": 1
    }
  ]
}
```

**`rank` is absent — not `null` — exactly when `qualified` is `false`**, because this API sets
`spring.jackson.default-property-inclusion: non_null` and omits every null field. Clients see
`undefined` and must branch on `qualified`, which is always present. `wins + draws + losses` may be
less than `played` when a completed match carries no result; it is never more. `goals` excludes own
goals.

### LeaderboardsDTO (response)

All four lists are always present, ordered best-first, and empty rather than absent. Players with a
zero count are omitted, so a list may be shorter than `limit` — that means the category ran out,
not that there is another page.

```json
{
  "limit": 5,
  "topScorers":     [ { "rank": 1, "playerId": 10, "playerName": "Joao Silva", "value": 30 } ],
  "topAssists":     [ { "rank": 1, "playerId": 11, "playerName": "Ana Costa",  "value": 25 } ],
  "mostMvps":       [ { "rank": 1, "playerId": 10, "playerName": "Joao Silva", "value": 6 } ],
  "longestStreaks": [ { "rank": 1, "playerId": 11, "playerName": "Ana Costa",  "value": 9 } ]
}
```

`value` carries a different unit per category — label it per card, not once for the component.
`mostMvps` counts the admin-assigned `is_mvp`; crowd MOTM voting will be a separate fact and must
not be folded into it.

**Error responses:** `403 Forbidden` when unauthenticated (Spring Security, no body). An
out-of-range `limit` is clamped rather than rejected.

> See [`LEADERBOARDS-API-CONTRACT.md`](./LEADERBOARDS-API-CONTRACT.md) for the cache-eviction
> matrix, the privacy consequences of erasure in each endpoint, and frontend migration notes.

---

## Admin — settings & system

| Method  | Path                        | Description                                              | Auth         | Request                | Response                  |
|---------|-----------------------------|----------------------------------------------------------|--------------|------------------------|---------------------------|
| GET     | `/api/admin/settings`       | Every configurable setting with bounds and provenance    | `ADMIN` | —                      | `AppSettingDTO[]` (200)   |
| PATCH   | `/api/admin/settings`       | Change one or more settings (keys are enum names)        | `ADMIN` | `AppSettingsUpdateDTO` | `AppSettingDTO[]` (200)   |
| GET     | `/api/admin/system-health`  | Push configuration and data counts                       | `ADMIN` | —                      | `SystemHealthDTO` (200)   |
| POST    | `/api/admin/badges/backfill`| Award badges across the whole completed match history    | `ADMIN` | —                      | `BackfillResult` (200)    |
| POST    | `/api/admin/caches/evict`   | Clear cached reads **on the node serving the request**   | `ADMIN` | —                      | eviction summary (200)    |

**`MANAGER` is refused here, not merely unlisted** — master runs the squad, these are system
concerns. Four settings are configurable: `MVP_VOTING_WINDOW_HOURS` (24, 1–168),
`RANKING_MINIMUM_MATCHES` (3, 1–50), `LEADERBOARD_DEFAULT_LIMIT` (5, 1–100) and
`LEADERBOARD_MAX_LIMIT` (25, 1–100).

`app_settings` holds a row **only where a default has been overridden**, so `overridden: false` means
"still on the default" and `updatedAt`/`updatedBy` are absent. Clients must read `min`, `max` and
`defaultValue` from the response rather than hard-coding them.

`system-health` **never returns key material**; the VAPID public key is included because every
subscribing browser receives it anyway. Badge backfill is idempotent — `uq_player_badges` makes a
repeat run a no-op.

> See [`ADMIN-API-CONTRACT.md`](./ADMIN-API-CONTRACT.md) for the cross-field validation rule, the
> cache-eviction consequences of a settings change, and why an unknown setting name is a `400`.

---

## Health

| Method | Path          | Description                                           | Auth   | Response               |
|--------|---------------|-------------------------------------------------------|--------|------------------------|
| GET    | `/api/health` | Application health check — returns status and version | Public | `HealthResponse` (200) |

### HealthResponse

```json
{
  "status": "UP",
  "version": "1.0.0",
  "timestamp": "2026-05-18T08:30:00Z"
}
```

---

## Version

| Method | Path           | Description                            | Auth   | Response                |
|--------|----------------|----------------------------------------|--------|-------------------------|
| GET    | `/api/version` | Application version and build metadata | Public | `VersionResponse` (200) |

### VersionResponse

```json
{
  "application": "football",
  "version": "1.0.0",
  "group": "pt.rics.demo",
  "buildTime": "2026-05-18T08:30:00Z"
}
```

> All fields are sourced from the compiled `build-info.properties` — always in sync with `build.gradle`.

---

## Error Format

All errors return the following JSON shape:

```json
{
  "timestamp": "2026-05-15T10:30:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "User with id 42 not found",
  "path": "/api/users/42",
  "violations": []
}
```

Validation errors populate `violations`:

```json
{
  "violations": [
    {
      "field": "username",
      "message": "must not be blank",
      "rejectedValue": ""
    },
    {
      "field": "email",
      "message": "must be a well-formed email address",
      "rejectedValue": "not-an-email"
    }
  ]
}
```

