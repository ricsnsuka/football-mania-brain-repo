# Match Feature

**Added in:** v1.0.0  
**Date:** May 17, 2026  
**Status:** ✅ Released

---

## Overview

The **Match** entity is the central event in the Football Management System. A match
records a single game between two teams, tracks per-player statistics (goals, assists,
own goals, MVP), and triggers automatic **skill-rating recalculation** on completion.

Matches are connected to a **Season** (falls back to the current active season if not
specified), contain exactly **two `MatchTeam` records**, and each team holds a list of
**`PlayerStat`** rows — one per participating player.

---

## Domain Model

### Entity Hierarchy

```
Match (1)
 ├── MatchTeam — "Team A" (teamOrder=1)
 │     ├── PlayerStat — player 1 (goals, assists, own_goals, rating, isMvp, matchResult)
 │     ├── PlayerStat — player 2
 │     └── ...
 └── MatchTeam — "Team B" (teamOrder=2)
       ├── PlayerStat — player 8
       └── ...
```

### `matches` Table

| Column             | Type           | Nullable | Notes                                                       |
|--------------------|----------------|----------|-------------------------------------------------------------|
| `id`               | BIGSERIAL (PK) | No       |                                                             |
| `description`      | VARCHAR(255)   | No       | Short title / name of the match                             |
| `match_date`       | TIMESTAMPTZ    | Yes      | When the match is scheduled / played                        |
| `location`         | VARCHAR(255)   | Yes      |                                                             |
| `is_completed`     | BOOLEAN        | No       | Default `false`. Set to `true` by `PATCH /complete`         |
| `match_type`       | VARCHAR(20)    | No       | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`            |
| `generation_type`  | VARCHAR(20)    | No       | `MANUAL` / `BALANCED` / `RANDOM` / `SNAKE_DRAFT`            |
| `generation_notes` | VARCHAR(500)   | Yes      | Human-readable explanation of team generation algorithm     |
| `score_team_a`     | INTEGER        | Yes      | Null until completed                                        |
| `score_team_b`     | INTEGER        | Yes      | Null until completed                                        |
| `final_score`      | VARCHAR(20)    | Yes      | Denormalised string, e.g. `"3-2"`. Null until completed     |
| `winning_team_id`  | BIGINT (FK)    | Yes      | FK → `match_teams.id`. Null for draws or pre-completion     |
| `season_id`        | BIGINT (FK)    | Yes      | FK → `seasons.id`. Falls back to current season on creation |
| `version`          | BIGINT         | No       | Optimistic locking version — prevents concurrent completion |
| `created_at`       | TIMESTAMPTZ    | No       |                                                             |
| `updated_at`       | TIMESTAMPTZ    | No       |                                                             |

### `match_teams` Table

| Column       | Type           | Nullable | Notes                                                     |
|--------------|----------------|----------|-----------------------------------------------------------|
| `id`         | BIGSERIAL (PK) | No       |                                                           |
| `match_id`   | BIGINT (FK)    | No       | FK → `matches.id` (CASCADE DELETE)                        |
| `name`       | VARCHAR(100)   | No       | e.g. `"Team A"`, `"Red"` — chosen when creating the match |
| `team_order` | INTEGER        | No       | `1` = Team A, `2` = Team B. Unique per match.             |
| `created_at` | TIMESTAMPTZ    | No       |                                                           |

### `player_stats` Table

| Column           | Type           | Nullable | Notes                                               |
|------------------|----------------|----------|-----------------------------------------------------|
| `id`             | BIGSERIAL (PK) | No       |                                                     |
| `player_id`      | BIGINT (FK)    | No       | FK → `players.id` (CASCADE DELETE)                  |
| `match_team_id`  | BIGINT (FK)    | No       | FK → `match_teams.id` (CASCADE DELETE)              |
| `goals`          | INTEGER        | No       | Denormalised total: `solo + assisted + penalty`     |
| `solo_goals`     | INTEGER        | No       | Solo goals (highest weight in rating formula)       |
| `assisted_goals` | INTEGER        | No       | Goals assisted by a team-mate                       |
| `penalty_goals`  | INTEGER        | No       | Penalty kicks scored                                |
| `assists`        | INTEGER        | No       | Assisted a goal scored by another player            |
| `own_goals`      | INTEGER        | No       | Own-goal count (penalised in rating formula)        |
| `rating`         | NUMERIC(4,2)   | Yes      | Server-computed match rating. Null until completion |
| `is_mvp`         | BOOLEAN        | No       | Default `false`. Admin-set at completion            |
| `match_result`   | VARCHAR(10)    | Yes      | `WIN` / `LOSS` / `DRAW`. Null until completion      |
| `created_at`     | TIMESTAMPTZ    | No       |                                                     |
| `updated_at`     | TIMESTAMPTZ    | No       |                                                     |

---

## API Endpoints

Base path: `/api/matches`

| Method   | Path                                              | Auth                          | Description                                                            |
|----------|---------------------------------------------------|-------------------------------|------------------------------------------------------------------------|
| `POST`   | `/api/matches`                                    | `ADMIN_USER` or `MASTER_USER` | Create a match with manual teams (player count must match `matchType`) |
| `POST`   | `/api/matches/manual`                             | `ADMIN_USER` or `MASTER_USER` | Create a match with freely-sized teams (equal sizes only)              |
| `GET`    | `/api/matches`                                    | Any authenticated             | List matches (paginated, filterable)                                   |
| `GET`    | `/api/matches/{id}`                               | Any authenticated             | Get match by ID                                                        |
| `PATCH`  | `/api/matches/{id}`                               | `ADMIN_USER` or `MASTER_USER` | Update description / date / location                                   |
| `PATCH`  | `/api/matches/{id}/complete`                      | `ADMIN_USER` or `MASTER_USER` | Complete match; run compliance check; compute ratings                  |
| `PATCH`  | `/api/matches/{id}/stats/live`                    | `ADMIN_USER` or `MASTER_USER` | Live stat update; returns preview ratings                              |
| `PATCH`  | `/api/matches/{id}/teams/{teamId}/stats/{statId}` | `ADMIN_USER`                  | Amend a single player stat post-completion                             |
| `POST`   | `/api/matches/{id}/recalculate`                   | `ADMIN_USER`                  | Idempotently recalc ratings for one completed match                    |
| `POST`   | `/api/matches/recalculate`                        | `ADMIN_USER`                  | Bulk recalc (matchIds / seasonId / all completed); per-match summary   |
| `DELETE` | `/api/matches/{id}`                               | `ADMIN_USER`                  | Delete a non-completed match                                           |

### Authorization Matrix

| Action                  | `BASIC_USER` | `MASTER_USER` | `ADMIN_USER` |
|-------------------------|:------------:|:-------------:|:------------:|
| List / read matches     |      ✅       |       ✅       |      ✅       |
| Create match            |      ❌       |       ✅       |      ✅       |
| Create manual match     |      ❌       |       ✅       |      ✅       |
| Update match details    |      ❌       |       ✅       |      ✅       |
| Complete match          |      ❌       |       ✅       |      ✅       |
| Live stats update       |      ❌       |       ✅       |      ✅       |
| Amend stat (post-comp.) |      ❌       |       ❌       |      ✅       |
| Recalculate ratings     |      ❌       |       ❌       |      ✅       |
| Delete match            |      ❌       |       ❌       |      ✅       |

---

## DTOs

### `MatchDTO` (Response)

All read endpoints return `MatchDTO`:

| Field             | Type                 | Nullable | Notes                                            |
|-------------------|----------------------|----------|--------------------------------------------------|
| `id`              | Long                 | No       |                                                  |
| `description`     | String               | No       |                                                  |
| `matchDate`       | Instant              | Yes      |                                                  |
| `location`        | String               | Yes      |                                                  |
| `matchType`       | String               | No       | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE` |
| `isCompleted`     | boolean              | No       |                                                  |
| `scoreTeamA`      | Integer              | Yes      | Null pre-completion                              |
| `scoreTeamB`      | Integer              | Yes      | Null pre-completion                              |
| `finalScore`      | String               | Yes      | `"3-2"` format. Null pre-completion              |
| `winningTeamId`   | Long                 | Yes      | Null for draw or pre-completion                  |
| `generationType`  | String               | No       |                                                  |
| `generationNotes` | String               | Yes      | Algorithm notes from team generation             |
| `seasonId`        | Long                 | Yes      |                                                  |
| `teams`           | List\<MatchTeamDTO\> | No       | Always 2 elements                                |
| `createdAt`       | Instant              | No       |                                                  |
| `updatedAt`       | Instant              | No       |                                                  |

### `MatchTeamDTO` (within `MatchDTO.teams`)

| Field         | Type                  | Notes                              |
|---------------|-----------------------|------------------------------------|
| `id`          | Long                  |                                    |
| `name`        | String                | e.g. `"Team A"`                    |
| `teamOrder`   | int                   | `1` = Team A, `2` = Team B         |
| `playerStats` | List\<PlayerStatDTO\> | Stats for each player in this team |

### `PlayerStatDTO` (within `MatchTeamDTO.playerStats`)

| Field           | Type    | Nullable | Notes                                          |
|-----------------|---------|----------|------------------------------------------------|
| `id`            | Long    | No       |                                                |
| `playerId`      | Long    | No       |                                                |
| `playerName`    | String  | No       |                                                |
| `soloGoals`     | Integer | No       |                                                |
| `assistedGoals` | Integer | No       |                                                |
| `penaltyGoals`  | Integer | No       |                                                |
| `assists`       | Integer | No       |                                                |
| `ownGoals`      | Integer | No       |                                                |
| `rating`        | Double  | Yes      | Null until completion; server-computed after   |
| `isMvp`         | Boolean | No       |                                                |
| `matchResult`   | String  | Yes      | `WIN` / `LOSS` / `DRAW`. Null until completion |

### `MatchCreateDTO` (POST request)

| Field            | Type                       | Required | Validation                                       |
|------------------|----------------------------|----------|--------------------------------------------------|
| `description`    | String                     | Yes      | `@NotBlank`                                      |
| `matchDate`      | Instant                    | No       |                                                  |
| `location`       | String                     | No       |                                                  |
| `matchType`      | String                     | Yes      | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE` |
| `generationType` | String                     | Yes      | `MANUAL` / `BALANCED` / `RANDOM` / `SNAKE_DRAFT` |
| `seasonId`       | Long                       | No       | Falls back to the current active season          |
| `teams`          | List\<MatchTeamCreateDTO\> | Yes      | Exactly 2 entries required                       |

**`MatchTeamCreateDTO`:**

| Field       | Type         | Required | Notes                                                    |
|-------------|--------------|----------|----------------------------------------------------------|
| `name`      | String       | Yes      | Team display name                                        |
| `playerIds` | List\<Long\> | Yes      | Must equal the required player count for the `matchType` |

> **Player counts per `matchType`:**
> - `FIVE_A_SIDE` → 5 players per team (10 total)
> - `SEVEN_A_SIDE` → 7 players per team (14 total)
> - `ELEVEN_A_SIDE` → 11 players per team (22 total)

### `ManualMatchCreateDTO` (POST /api/matches/manual request)

Used when an admin/master user wants to create a match with teams of any equal size, bypassing the per-`matchType`player
count requirement.

| Field         | Type                       | Required | Validation                                                    |
|---------------|----------------------------|----------|---------------------------------------------------------------|
| `description` | String                     | Yes      | `@NotBlank @Size(max=255)`                                    |
| `matchDate`   | Instant                    | Yes      | `@NotNull` — ISO-8601 UTC datetime                            |
| `location`    | String                     | No       | `@Size(max=255)` — venue name                                 |
| `matchType`   | String                     | Yes      | `@NotNull` — `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE` |
| `seasonId`    | Long                       | No       | Falls back to current active season if null                   |
| `teams`       | List\<MatchTeamCreateDTO\> | Yes      | `@Size(min=2, max=2)` — exactly 2 teams                       |

**`MatchTeamCreateDTO`** (same shape as above, but `playerIds` has relaxed count):

| Field       | Type         | Required | Notes                                                                                    |
|-------------|--------------|----------|------------------------------------------------------------------------------------------|
| `name`      | String       | Yes      | `@NotBlank @Size(max=100)` — team display name                                           |
| `playerIds` | List\<Long\> | Yes      | `@NotEmpty` — at least 1 player; no upper limit; both teams **must** have the same count |

> **Key difference from `MatchCreateDTO`:**  
> `POST /api/matches` requires each team to have exactly the `matchType`-defined number of players (5 / 7 / 11).
`POST /api/matches/manual` only enforces that **both teams have the same number of players** — any equal count is
> accepted (e.g., 3v3 on a SEVEN_A_SIDE `matchType`).  
> The `generationType` in the response is always `"MANUAL"` for this endpoint.

### `MatchUpdateDTO` (PATCH request)

All fields optional (safe PATCH — `null` = no change):

| Field         | Type    | Notes |
|---------------|---------|-------|
| `description` | String  |       |
| `matchDate`   | Instant |       |
| `location`    | String  |       |

> ⚠️ Can only update a match that is **not** yet completed.

### `MatchCompleteDTO` (PATCH /complete request)

| Field           | Type                        | Required | Notes                                              |
|-----------------|-----------------------------|----------|----------------------------------------------------|
| `scoreTeamA`    | Integer                     | Yes      | Final score for Team A (teamOrder=1)               |
| `scoreTeamB`    | Integer                     | Yes      | Final score for Team B (teamOrder=2)               |
| `winningTeamId` | Long                        | No       | Omit for a draw                                    |
| `playerStats`   | List\<PlayerStatUpdateDTO\> | Yes      | One entry per player stat that has stats to record |

### `PlayerStatUpdateDTO` (within MatchCompleteDTO and MatchLiveUpdateDTO)

| Field           | Type    | Required | Notes                       |
|-----------------|---------|----------|-----------------------------|
| `playerStatId`  | Long    | **Yes**  | Must reference existing row |
| `soloGoals`     | Integer | No       | Null = no change            |
| `assistedGoals` | Integer | No       | Null = no change            |
| `penaltyGoals`  | Integer | No       | Null = no change            |
| `assists`       | Integer | No       | Null = no change            |
| `ownGoals`      | Integer | No       | Null = no change            |
| `isMvp`         | Boolean | No       | Null = no change            |

> ⚠️ There is **no `rating` field** — ratings are always server-computed.

### `MatchLiveUpdateDTO` (PATCH /stats/live request)

| Field         | Type                        | Notes                               |
|---------------|-----------------------------|-------------------------------------|
| `playerStats` | List\<PlayerStatUpdateDTO\> | One or more player updates per call |

### `MatchLiveUpdateResponseDTO` (PATCH /stats/live response)

| Field     | Type                         | Notes                                  |
|-----------|------------------------------|----------------------------------------|
| `ratings` | List\<PlayerMatchRatingDTO\> | Preview rating for each updated player |

**`PlayerMatchRatingDTO`:**

| Field          | Type   | Notes                                           |
|----------------|--------|-------------------------------------------------|
| `playerStatId` | Long   | FK to `player_stats`                            |
| `playerId`     | Long   | FK to `players`                                 |
| `playerName`   | String |                                                 |
| `matchRating`  | double | Preview rating — no WIN/LOSS or goal-diff bonus |

---

## Match Lifecycle

```
          POST /api/matches
               │
               ▼
         [ CREATED ]  ←──── PATCH /api/matches/{id}   (update description/date/location)
          isCompleted=false
               │
               │  ← PATCH /api/matches/{id}/stats/live  (update stats, get preview ratings)
               │
               ▼
        PATCH /api/matches/{id}/complete
               │
               │  1. Compliance validation (goal counts must match declared score)
               │  2. Set matchResult (WIN/LOSS/DRAW) on each PlayerStat
               │  3. Compute server-side match ratings via CalculationService
               │  4. Set isCompleted = true; persist scores
               │  5. Publish MatchCompletedEvent
               │  6. CalculationService.recalculateMatchRatings() updates player skillRatings + streaks
               │
               ▼
         [ COMPLETED ]
          isCompleted=true
          rating populated on all PlayerStat rows

          ← PATCH /api/matches/{id}/teams/{teamId}/stats/{statId}  (ADMIN post-comp amendment)
```

---

### Business Rules

### 1. Team Size Validation

Each team must have exactly the number of players required by the `matchType`:

- `FIVE_A_SIDE` → 5 per team
- `SEVEN_A_SIDE` → 7 per team
- `ELEVEN_A_SIDE` → 11 per team

Both teams must have equal player counts.

> ⚠️ This rule applies to `POST /api/matches` only. See [Manual Match Creation](#manual-match-creation) for the
> unrestricted alternative.

### 2. No Duplicate Players

A player may not appear in both teams or appear twice in the same team.

### 3. Auto-Activation of Inactive Players

When a player marked `isActive = false` is included in a team, the service automatically
sets `isActive = true`. `isActive` is a participation flag (e.g. "unavailable this month"),
not a soft-delete.

### 4. Season Resolution

If `seasonId` is omitted from `MatchCreateDTO`, the service resolves the current active
season (`is_current = true` in `seasons`). If no current season exists, the request fails
with `400 Bad Request`.

### 5. Compliance Check on Completion

Before completing a match, the service validates goal-count consistency:

```
For Team A (teamOrder=1):
  (sum of soloGoals + assistedGoals + penaltyGoals for Team A players)
  + (sum of ownGoals for Team B players)
  must equal scoreTeamA

For Team B (teamOrder=2):
  (sum of soloGoals + assistedGoals + penaltyGoals for Team B players)
  + (sum of ownGoals for Team A players)
  must equal scoreTeamB
```

A mismatch returns `400 Bad Request` with a descriptive message.

### 6. Server-Side Rating Computation

The `rating` field on `PlayerStatDTO` is **always server-computed** by `CalculationService`.
Clients cannot submit a rating value — the field does not exist on `PlayerStatUpdateDTO`.

### 7. Optimistic Locking

The `Match` entity carries a `@Version` field. Concurrent attempts to complete the same
match will result in `409 Conflict` (`ObjectOptimisticLockingFailureException`).

### 8. Delete Restriction

A match can only be deleted if `isCompleted = false`. Attempting to delete a completed match
returns `409 Conflict`.

---

## Manual Match Creation

`POST /api/matches/manual` provides an **unrestricted** team creation path for admin/master users. It is designed for
scenarios where the standard player-count rules don't apply — ad-hoc matches, training games, or friendly fixtures with
non-standard squad sizes.

### Difference from Standard Match Creation

| Behaviour                    | `POST /api/matches`                 | `POST /api/matches/manual` |
|------------------------------|-------------------------------------|----------------------------|
| Player count per team        | Must match `matchType` (5 / 7 / 11) | Any count ≥ 1              |
| Both teams equal size        | ✅ Required                          | ✅ Required                 |
| No duplicate players         | ✅ Required                          | ✅ Required                 |
| Player IDs must exist        | ✅ Required                          | ✅ Required                 |
| `generationType` in response | Client-specified                    | Always `"MANUAL"`          |
| `matchDate`                  | Optional                            | **Required** (`@NotNull`)  |

### Lifecycle

Manual matches follow the exact same lifecycle as standard matches: they can receive live stat updates, be completed
with scores, have ratings computed, and be deleted (if not yet completed). The `matchType` field is still recorded and
used during completion if compliance checks apply.

### Example Request

```http
POST /api/matches/manual HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "description": "Casual 3v3 match",
  "matchDate": "2026-07-10T18:00:00Z",
  "location": "Park pitch B",
  "matchType": "SEVEN_A_SIDE",
  "seasonId": 1,
  "teams": [
    { "name": "Team A", "playerIds": [1, 2, 3] },
    { "name": "Team B", "playerIds": [4, 5, 6] }
  ]
}
```

### Example Response `201 Created`

```json
{
  "id": 12,
  "description": "Casual 3v3 match",
  "matchDate": "2026-07-10T18:00:00Z",
  "location": "Park pitch B",
  "matchType": "SEVEN_A_SIDE",
  "isCompleted": false,
  "scoreTeamA": null,
  "scoreTeamB": null,
  "finalScore": null,
  "winningTeamId": null,
  "generationType": "MANUAL",
  "generationNotes": null,
  "seasonId": 1,
  "teams": [
    {
      "id": 23,
      "name": "Team A",
      "teamOrder": 1,
      "playerStats": [
        {
          "id": 100,
          "playerId": 1,
          "playerName": "João Silva",
          "soloGoals": 0,
          "assistedGoals": 0,
          "penaltyGoals": 0,
          "assists": 0,
          "ownGoals": 0,
          "rating": null,
          "isMvp": false,
          "matchResult": null
        }
      ]
    },
    {
      "id": 24,
      "name": "Team B",
      "teamOrder": 2,
      "playerStats": []
    }
  ],
  "createdAt": "2026-07-09T14:00:00Z",
  "updatedAt": "2026-07-09T14:00:00Z"
}
```

The `Location` response header is set to `/api/matches/12`.

---

## Rating Recalculation (Admin)

**Added in:** unreleased · **Auth:** `ADMIN_USER` only (both endpoints).

Two admin-only endpoints re-run the rating engine over already-**completed** matches **without
double-counting**. They exist so an administrator can bring persisted ratings back in sync after
data or model changes, on demand and synchronously (the normal completion path stays event-driven).

| Method | Path | Response |
|--------|------|----------|
| `POST` | `/api/matches/{id}/recalculate` | `RecalculationResultDTO` (200) |
| `POST` | `/api/matches/recalculate` | `BulkRecalculationResponseDTO` (200) |

Non-admin (`BASIC_USER` / `MASTER_USER`) and unauthenticated callers both receive `403`.

### Use Cases

- **After amending a stat** via `PATCH /api/matches/{id}/teams/{teamId}/stats/{statId}` — the stat
  edit changes goals/assists but does not itself re-run the engine; recalculate the affected match to
  refresh `PlayerStat.rating`, career aggregates, streaks and history.
- **After a rating-model change** (engine constants / formula) — bulk-recalculate a season (or all
  completed matches) so historical ratings reflect the new model.
- **Data correction / backfill** — re-run older matches that were completed before an engine fix.

### Single Match — `POST /api/matches/{id}/recalculate`

No request body. Returns `RecalculationResultDTO`:

| Field | Type | Notes |
|-------|------|-------|
| `matchId` | Long | The recalculated match |
| `status` | String | `"SUCCESS"` \| `"FAILED"` \| `"SKIPPED"` |
| `ratingsUpdated` | int | `PlayerStat` rows recomputed; `0` on `FAILED`/`SKIPPED` |
| `message` | String | Human-readable outcome |

| Status | Trigger |
|--------|---------|
| `200` `SUCCESS` | Ratings recomputed |
| `200` `SKIPPED` | Completed match with **no** player stats (`"No player stats — skipped"`) |
| `403` | Not `ADMIN_USER` / unauthenticated |
| `404` | Match not found |
| `409` | Match exists but is **not completed** (also optimistic-lock conflict) |

```http
POST /api/matches/42/recalculate HTTP/1.1
Authorization: Bearer <admin-token>
```

```json
{ "matchId": 42, "status": "SUCCESS", "ratingsUpdated": 14, "message": "Recalculated 14 player ratings" }
```

### Bulk — `POST /api/matches/recalculate`

Optional body `BulkRecalculationRequestDTO`. Selection precedence: **`matchIds` → `seasonId` → all
completed**. An empty/absent body means "all completed matches". `matchIds` and `seasonId` are
mutually exclusive.

Each match is processed **independently, in its own transaction**, in chronological order
(`matchDate ASC NULLS LAST, id ASC`). One bad match never fails the batch — the call **always returns
`200 OK`** with a per-match summary.

| Status | Trigger |
|--------|---------|
| `200` | Batch executed — inspect `results[]` |
| `400` | Both `matchIds` **and** `seasonId` supplied, or a `matchIds` element is null / `< 1` |
| `403` | Not `ADMIN_USER` / unauthenticated |
| `404` | `seasonId` supplied but season does not exist |

Per-match outcomes: a non-existent explicit `matchId`, a non-completed match, or an engine error →
`FAILED`; a completed match with no stats → `SKIPPED`. `succeeded + failed` may be `< totalRequested`
when `SKIPPED` items exist.

```http
POST /api/matches/recalculate HTTP/1.1
Content-Type: application/json
Authorization: Bearer <admin-token>

{ "matchIds": [42, 43, 44], "seasonId": null }
```

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

### Idempotency Guarantees & Caveats

Recalculation is **idempotent** via a *reverse-then-reapply* strategy (see
[`CALCULATION_SERVICE.md`](./CALCULATION_SERVICE.md)). Running a recalc twice yields the same result
as running it once for:

- `PlayerStat.rating` (per-match) — **exact**
- Career aggregates `totalGoals` / `totalAssists` / `totalMatchesPlayed` — **exact**
- `skill_rating_history` rows — **no duplicates** (this match's rows are deleted then re-inserted)
- Streaks — **exact** (recomputed from the chain, not reversed arithmetically)

> ⚠️ **`skillRating` mid-chain caveat.** `skillRating` is an EMA over the player's chronological match
> chain. The **single-match** endpoint restores it **exactly** for a player's **most-recent** match,
> but only **approximately** for a match in the middle of the chain (later matches' baselines become
> stale). To fully reconcile a player/season, use the **bulk** endpoint, which replays the affected
> matches in chronological order.

**Bulk vs. single guidance:**

- **Single** → correcting the **latest** match after a stat amend (fast, exact for that case).
- **Bulk (season / all)** → after a model change, or to reconcile `skillRating` for mid-chain edits;
  chronological replay makes the whole span consistent.

> No DB schema change. Backed by repository finders `MatchRepository.findCompletedOrdered` /
> `findCompletedBySeasonOrdered` and `PlayerStatRepository.findCompletedByPlayerIdChronological`.

---

## Request / Response Examples

### Create a 7-a-side Match

**Request:**

```http
POST /api/matches HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "description": "Friday Night Match",
  "matchDate": "2026-05-23T19:00:00Z",
  "location": "Central Park",
  "matchType": "SEVEN_A_SIDE",
  "generationType": "manual",
  "seasonId": 1,
  "teams": [
    {
      "name": "Team A",
      "playerIds": [1, 2, 3, 4, 5, 6, 7]
    },
    {
      "name": "Team B",
      "playerIds": [8, 9, 10, 11, 12, 13, 14]
    }
  ]
}
```

**Response `201 Created`:**

```json
{
  "id": 5,
  "description": "Friday Night Match",
  "matchDate": "2026-05-23T19:00:00Z",
  "location": "Central Park",
  "matchType": "SEVEN_A_SIDE",
  "isCompleted": false,
  "scoreTeamA": null,
  "scoreTeamB": null,
  "finalScore": null,
  "winningTeamId": null,
  "generationType": "MANUAL",
  "generationNotes": null,
  "seasonId": 1,
  "teams": [
    {
      "id": 9,
      "name": "Team A",
      "teamOrder": 1,
      "playerStats": [
        {
          "id": 50,
          "playerId": 1,
          "playerName": "João Silva",
          "soloGoals": 0,
          "assistedGoals": 0,
          "penaltyGoals": 0,
          "assists": 0,
          "ownGoals": 0,
          "rating": null,
          "isMvp": false,
          "matchResult": null
        }
      ]
    },
    {
      "id": 10,
      "name": "Team B",
      "teamOrder": 2,
      "playerStats": []
    }
  ],
  "createdAt": "2026-05-22T10:00:00Z",
  "updatedAt": "2026-05-22T10:00:00Z"
}
```

### Live Stat Update (during the match)

```http
PATCH /api/matches/5/stats/live HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "playerStats": [
    { "playerStatId": 50, "soloGoals": 1, "assists": 0 },
    { "playerStatId": 51, "assists": 1 }
  ]
}
```

**Response `200 OK`:**

```json
{
  "ratings": [
    {
      "playerStatId": 50,
      "playerId": 1,
      "playerName": "João Silva",
      "matchRating": 5.83
    },
    {
      "playerStatId": 51,
      "playerId": 2,
      "playerName": "Pedro Costa",
      "matchRating": 5.28
    }
  ]
}
```

> `matchRating` here is a **preview** — no WIN/LOSS bonus or goal-diff bonus is applied.

### Complete a Match

```http
PATCH /api/matches/5/complete HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "scoreTeamA": 3,
  "scoreTeamB": 2,
  "winningTeamId": 9,
  "playerStats": [
    { "playerStatId": 50, "soloGoals": 2, "assistedGoals": 0, "penaltyGoals": 1, "assists": 0, "ownGoals": 0, "isMvp": true },
    { "playerStatId": 60, "soloGoals": 0, "assistedGoals": 0, "penaltyGoals": 0, "assists": 0, "ownGoals": 1, "isMvp": false }
  ]
}
```

> After completion, the `MatchDTO` response includes `rating` on every `PlayerStatDTO` (server-computed).
> Player skill ratings and streaks are updated asynchronously after the response returns.

---

## Caching Strategy

| Cache Name | When Populated                      | When Evicted                             |
|------------|-------------------------------------|------------------------------------------|
| `matches`  | `GET /api/matches/{id}` (key-based) | Any write (create/update/complete/amend) |

Completing a match also evicts `players` and `rankings` caches because:

- Player `skillRating` and streak values are updated
- Rankings leaderboard data must be refreshed

---

## Implementation Details

- **Entity:** `Match.java`, `MatchTeam.java`, `PlayerStat.java`, `Goal.java` — JPA entities with Lombok `@Getter`/
  `@Setter`
- **Repository:** `MatchRepository.java`, `MatchTeamRepository.java`, `PlayerStatRepository.java`
- **Mapper:** `MatchMapper.java` — MapStruct compile-time mapper
- **Service:** `MatchService.java` — business logic, compliance validation, cache eviction, event publishing
- **Controller:** `MatchController.java` — REST layer with `@PreAuthorize`
- **DTOs:** `MatchDTO`, `MatchCreateDTO`, `ManualMatchCreateDTO`, `MatchUpdateDTO`, `MatchCompleteDTO`, `MatchTeamDTO`,
  `MatchTeamCreateDTO`, `PlayerStatDTO`, `PlayerStatUpdateDTO`, `MatchLiveUpdateDTO`, `MatchLiveUpdateResponseDTO`,
  `PlayerMatchRatingDTO` — all Java Records

### Synchronous Completion Design

`completeMatch()` is intentionally **synchronous** — all stat writes, compliance checks,
rating computation via `CalculationService`, and skill-rating updates to the `players`
table happen within a single database transaction before the HTTP response is returned.

This ensures strong consistency: the caller's `PATCH /complete` response always contains
the final computed ratings. An async/event-driven approach would require eventual
consistency and is not warranted at the current scale.

### Optimistic Locking

`Match` has a `@Version` (BIGINT) column. If two requests try to complete the same match
simultaneously, one will get `ObjectOptimisticLockingFailureException`, which
`GlobalExceptionHandler` maps to `409 Conflict`.

---

## Error Reference

| Scenario                                | Status | Message                                                    |
|-----------------------------------------|--------|------------------------------------------------------------|
| Match not found                         | 404    | `Match with id {id} not found`                             |
| Wrong player count for match type       | 400    | `Match type X requires N players per team, got M`          |
| Both teams have different player counts | 400    | `Both teams must have the same number of players`          |
| Duplicate player across teams           | 400    | `Duplicate players detected across teams`                  |
| Player ID not found                     | 404    | `Player with ids {...} not found`                          |
| No active season                        | 400    | `No active season found; please specify a seasonId`        |
| Compliance check failure at completion  | 400    | `Score compliance failed: Team A goals (X) ≠ declared (Y)` |
| Completing an already-completed match   | 409    | Conflict / optimistic locking                              |
| Deleting a completed match              | 409    | `Cannot delete a completed match`                          |
| Amending stat: stat not in given team   | 404    | `PlayerStat with id {id} not found in team {teamId}`       |
| Not ADMIN/MASTER for write endpoints    | 403    | Forbidden                                                  |
| Recalculate: match not completed        | 409    | `Match id={id} is not completed`                           |
| Bulk recalc: both `matchIds`+`seasonId` | 400    | `matchIds and seasonId are mutually exclusive — supply only one` |
| Bulk recalc: `seasonId` not found       | 404    | `Season with id {id} not found`                            |
| Manual match — `matchDate` is null      | 400    | Validation: `matchDate` must not be null                   |
| Manual match — unequal team sizes       | 400    | `Both teams must have the same number of players`          |

