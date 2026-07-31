# Match Plans & Availability Poll Feature

**Added in:** v1.0.0  
**Date:** May 17, 2026  
**Status:** ✅ Released

---

## Overview

A **Match Plan** is a pre-match organisation tool. Before a match can be created, an
admin or master user:

1. Creates a **Match Plan** with a proposed date, location, match type, and confirmation deadline.
2. The plan opens an **availability poll** — players confirm or decline participation.
3. Once enough players have confirmed, the admin **generates a team preview** using one of
   three algorithms (BALANCED, RANDOM, SNAKE_DRAFT).
4. The admin reviews the preview and **confirms** it — which creates the actual `Match`.

This decouples "planning who will play" from "running the match stats".

---

## Domain Model

### `match_plans` Table

| Column                  | Type           | Nullable | Notes                                                                 |
|-------------------------|----------------|----------|-----------------------------------------------------------------------|
| `id`                    | BIGSERIAL (PK) | No       |                                                                       |
| `title`                 | VARCHAR(100)   | No       | Default `'Unnamed Plan'`                                              |
| `proposed_date`         | DATE           | No       | Must be in the future at creation (@Future)                           |
| `location`              | VARCHAR(255)   | Yes      |                                                                       |
| `description`           | VARCHAR(500)   | Yes      |                                                                       |
| `match_type`            | VARCHAR(20)    | No       | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`                    |
| `status`                | VARCHAR(20)    | No       | `PENDING` / `CONFIRMED` / `CANCELLED`                                |
| `confirmed_count`       | INTEGER        | No       | Denormalised count of `CONFIRMED` player confirmations                |
| `min_players_required`  | INTEGER        | No       | Default `14` for SEVEN_A_SIDE                                         |
| `confirmation_deadline` | TIMESTAMPTZ    | Yes      | After this time, `pollOpen` becomes `false`                           |
| `created_by`            | VARCHAR(50)    | Yes      | Username of the admin/master who created the plan                     |
| `created_at`            | TIMESTAMPTZ    | No       |                                                                       |
| `updated_at`            | TIMESTAMPTZ    | No       |                                                                       |

### `player_confirmations` Table

| Column          | Type           | Nullable | Notes                                                |
|-----------------|----------------|----------|------------------------------------------------------|
| `id`            | BIGSERIAL (PK) | No       |                                                      |
| `match_plan_id` | BIGINT (FK)    | No       | FK → `match_plans.id` (CASCADE DELETE)               |
| `player_id`     | BIGINT (FK)    | No       | FK → `players.id` (CASCADE DELETE)                   |
| `status`        | VARCHAR(20)    | No       | `CONFIRMED` / `DECLINED` / `PENDING`                 |
| `notes`         | VARCHAR(500)   | Yes      | Optional note from player (e.g. "5 minutes late")    |
| `confirmed_at`  | TIMESTAMPTZ    | Yes      | Timestamp when player confirmed                      |

Unique constraint: `(match_plan_id, player_id)` — one confirmation record per player per plan.

---

## Plan Status Lifecycle

```
      POST /api/match-plans
              │
              ▼
          [ PENDING ]   ← poll is open (if deadline not passed)
              │
              ├─── PATCH /api/match-plans/{id}/status  { status: "CONFIRMED" }
              │         ▼
              │     [ CONFIRMED ]
              │           │
              │           ├─── PATCH .../status  { status: "CANCELLED" }
              │           │         ▼
              │           │     [ CANCELLED ]
              │           │
              │           └─── keeps active (match may be generated from this)
              │
              └─── PATCH /api/match-plans/{id}/status  { status: "CANCELLED" }
                        ▼
                    [ CANCELLED ]
```

Status transitions:
- `PENDING` → `CONFIRMED` ✅
- `PENDING` → `CANCELLED` ✅
- `CONFIRMED` → `CANCELLED` ✅
- Any other transition → `400 Bad Request`

---

## Team Generation Flow

```
GET  /api/match-plans/{id}/confirmations   (review who confirmed)
          │
          ▼
POST /api/match-plans/{id}/generate?generationType=BALANCED
          │
          ▼
    MatchPreviewDTO  (NOT persisted — preview only)
    {
      teamARatingAvg: 7.14,
      teamBRatingAvg: 7.12,
      ratingDelta: 0.02,
      teams: [ { name: "Team A", players: [...] }, { name: "Team B", players: [...] } ]
    }
          │
          │  ← admin reviews, can regenerate with different algorithm
          │
          ▼
POST /api/match-plans/{id}/generate/confirm?generationType=BALANCED
          │
          ▼
    MatchDTO (201 Created — the actual Match is now persisted)
```

Available `generationType` values:
| Value         | Description                                             |
|---------------|---------------------------------------------------------|
| `BALANCED`    | Greedy skill-rating equalizer (default, most fair)      |
| `RANDOM`      | Pure random shuffle                                     |
| `SNAKE_DRAFT` | Alternating top-pick (snake order by skill rating)      |

---

## API Endpoints

Base path: `/api/match-plans`

| Method  | Path                                            | Auth                          | Description                                                    |
|---------|-------------------------------------------------|-------------------------------|----------------------------------------------------------------|
| `POST`  | `/api/match-plans`                              | `ADMIN_USER` or `MASTER_USER` | Create plan and open poll                                      |
| `GET`   | `/api/match-plans`                              | Any authenticated             | List plans (paginated, optional `status` filter)               |
| `GET`   | `/api/match-plans/{id}`                         | Any authenticated             | Get plan by ID                                                 |
| `PATCH` | `/api/match-plans/{id}`                         | `ADMIN_USER` or `MASTER_USER` | Update plan details (PENDING only)                             |
| `PATCH` | `/api/match-plans/{id}/status`                  | `ADMIN_USER` or `MASTER_USER` | Transition status                                              |
| `DELETE`| `/api/match-plans/{id}`                         | `ADMIN_USER`                  | Delete a PENDING plan                                          |
| `GET`   | `/api/match-plans/{id}/confirmations`           | Any authenticated             | List all confirmations (optional `status` filter)              |
| `POST`  | `/api/match-plans/{id}/confirmations/me`        | Any authenticated             | Self-confirm or decline availability                           |
| `GET`   | `/api/match-plans/{id}/confirmations/me`        | Any authenticated             | Get your own confirmation entry                                |
| `PATCH` | `/api/match-plans/{id}/confirmations/{playerId}`| `ADMIN_USER` or `MASTER_USER` | Admin override: set any player's confirmation status           |
| `POST`  | `/api/match-plans/{id}/generate`                | `ADMIN_USER` or `MASTER_USER` | Preview generated teams (stateless, not persisted)             |
| `POST`  | `/api/match-plans/{id}/generate/confirm`        | `ADMIN_USER` or `MASTER_USER` | Confirm preview and create the match                           |

### Authorization Matrix

| Action                           | `BASIC_USER` | `MASTER_USER` | `ADMIN_USER` |
|----------------------------------|:---:|:---:|:---:|
| Read plans / confirmations       | ✅ | ✅ | ✅ |
| Self-confirm availability        | ✅ | ✅ | ✅ |
| Create plan                      | ❌ | ✅ | ✅ |
| Update plan details              | ❌ | ✅ | ✅ |
| Change plan status               | ❌ | ✅ | ✅ |
| Override any player confirmation | ❌ | ✅ | ✅ |
| Generate team preview            | ❌ | ✅ | ✅ |
| Confirm generation (create match)| ❌ | ✅ | ✅ |
| Delete plan                      | ❌ | ❌ | ✅ |

---

## DTOs

### `MatchPlanDTO` (Response)

| Field                  | Type    | Nullable | Notes                                                              |
|------------------------|---------|----------|--------------------------------------------------------------------|
| `id`                   | Long    | No       |                                                                    |
| `title`                | String  | No       |                                                                    |
| `matchType`            | String  | No       | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`                 |
| `location`             | String  | Yes      |                                                                    |
| `proposedDate`         | String  | No       | ISO-8601 date: `"2026-05-23"`                                      |
| `confirmationDeadline` | Instant | Yes      |                                                                    |
| `status`               | String  | No       | `PENDING` / `CONFIRMED` / `CANCELLED`                             |
| `confirmedCount`       | int     | No       | Number of players who CONFIRMED                                    |
| `minPlayersRequired`   | int     | No       | Minimum needed to generate teams                                   |
| `description`          | String  | Yes      |                                                                    |
| `createdBy`            | String  | Yes      | Username of creator                                                |
| `playersNeeded`        | int     | No       | `max(0, minPlayersRequired - confirmedCount)` — computed           |
| `pollOpen`             | boolean | No       | `true` when `status == PENDING` and deadline has not passed        |
| `createdAt`            | Instant | No       |                                                                    |
| `updatedAt`            | Instant | No       |                                                                    |

### `MatchPlanCreateDTO` (POST request)

| Field                  | Type    | Required | Validation                                              |
|------------------------|---------|----------|---------------------------------------------------------|
| `title`                | String  | Yes      | `@NotBlank`                                             |
| `matchType`            | String  | Yes      | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`       |
| `location`             | String  | No       |                                                         |
| `proposedDate`         | String  | Yes      | `@Future` — must be a future date, format `YYYY-MM-DD`  |
| `confirmationDeadline` | Instant | No       |                                                         |
| `description`          | String  | No       |                                                         |
| `minPlayersRequired`   | Integer | No       | Default `14`                                            |

### `MatchPlanUpdateDTO` (PATCH request — all fields optional)

| Field                  | Type    | Notes                       |
|------------------------|---------|-----------------------------|
| `title`                | String  | Null = no change            |
| `location`             | String  | Null = no change            |
| `proposedDate`         | String  | Null = no change            |
| `confirmationDeadline` | Instant | Null = no change            |
| `description`          | String  | Null = no change            |

> ⚠️ Updates are only allowed when the plan is in `PENDING` status.

### `MatchPlanStatusDTO` (PATCH /status request)

| Field    | Type   | Required | Validation            |
|----------|--------|----------|-----------------------|
| `status` | String | Yes      | `@NotBlank`           |

### `ConfirmationUpsertDTO` (POST /confirmations/me and PATCH /confirmations/{playerId})

| Field    | Type   | Required | Validation              | Notes                                       |
|----------|--------|----------|-------------------------|---------------------------------------------|
| `status` | String | Yes      | `@NotBlank`             | `CONFIRMED` / `DECLINED` / `PENDING`        |
| `notes`  | String | No       | `@Size(max=500)`        | Optional note (e.g. "Will be 5 min late")   |

### `PlayerConfirmationDTO` (Response)

| Field         | Type    | Nullable | Notes                                          |
|---------------|---------|----------|------------------------------------------------|
| `id`          | Long    | No       |                                                |
| `matchPlanId` | Long    | No       |                                                |
| `playerId`    | Long    | No       |                                                |
| `playerName`  | String  | No       |                                                |
| `status`      | String  | No       | `CONFIRMED` / `DECLINED` / `PENDING`           |
| `notes`       | String  | Yes      |                                                |
| `confirmedAt` | Instant | Yes      | Set when player first confirms                 |

### `MatchPreviewDTO` (POST /generate response)

| Field              | Type                         | Notes                                               |
|--------------------|------------------------------|-----------------------------------------------------|
| `matchPlanId`      | Long                         |                                                     |
| `matchType`        | String                       |                                                     |
| `generationType`   | String                       |                                                     |
| `location`         | String                       |                                                     |
| `proposedDate`     | String                       |                                                     |
| `generationNotes`  | String                       | Algorithm explanation                               |
| `teamARatingAvg`   | double                       | Average skill rating of Team A                      |
| `teamBRatingAvg`   | double                       | Average skill rating of Team B                      |
| `ratingDelta`      | double                       | `|teamARatingAvg - teamBRatingAvg|` → fairness gauge|
| `teams`            | List\<PreviewTeamDTO\>       | Two teams with players and their ratings            |

**`PreviewTeamDTO`:**

| Field        | Type                                | Notes                      |
|--------------|-------------------------------------|----------------------------|
| `name`       | String                              | `"Team A"` / `"Team B"`    |
| `ratingAvg`  | double                              | Average skill rating        |
| `players`    | List\<PreviewPlayerDTO\>            | Players in this team        |

**`PreviewPlayerDTO`:**

| Field         | Type   | Notes                  |
|---------------|--------|------------------------|
| `playerId`    | Long   |                        |
| `playerName`  | String |                        |
| `skillRating` | double |                        |

---

## Business Rules

1. **`proposedDate` must be in the future** — validated with `@Future`. Updating a plan to
   a past date is rejected.

2. **Only `PENDING` plans can be updated** — `PATCH /{id}` returns `409 Conflict` if the
   plan is `CONFIRMED` or `CANCELLED`.

3. **Only `PENDING` plans can be deleted** — `DELETE /{id}` returns `409 Conflict` for
   non-pending plans.

4. **Confirmations are upsert** — `POST /confirmations/me` creates the record if it
   doesn't exist, or updates it if it does.

5. **`confirmedCount` is maintained automatically** — the service increments/decrements
   `confirmedCount` on the `MatchPlan` record whenever a confirmation status changes
   to/from `CONFIRMED`.

6. **`pollOpen` is computed** — `true` when `status == PENDING` and either
   `confirmationDeadline` is null or it is in the future. It is **not** a stored column.

7. **`playersNeeded` is computed** — `max(0, minPlayersRequired - confirmedCount)`.
   Useful for UI display ("Still need 3 more players").

8. **Team generation uses confirmed players only** — only players with `status == CONFIRMED`
   are included in the generated teams. The confirmed pool must match the team
   size requirements of the `matchType` (e.g. 14 for SEVEN_A_SIDE).

9. **Generation is stateless** — `POST /generate` never persists anything. The preview
   is computed in memory. Call it multiple times with different `generationType` values
   to compare distributions.

10. **`generate/confirm` creates the match** — each call to `POST /generate/confirm`
    creates one `Match`. Calling it multiple times creates multiple matches from the same
    plan (admins should confirm only once).

---

## Request / Response Examples

### Create a Match Plan

```http
POST /api/match-plans HTTP/1.1
Content-Type: application/json
Authorization: Bearer <admin-token>

{
  "title": "Friday Night — Week 21",
  "matchType": "SEVEN_A_SIDE",
  "location": "Central Park Pitch 2",
  "proposedDate": "2026-05-30",
  "confirmationDeadline": "2026-05-29T20:00:00Z",
  "description": "Bring bibs",
  "minPlayersRequired": 14
}
```

**Response `201 Created`:**
```json
{
  "id": 5,
  "title": "Friday Night — Week 21",
  "matchType": "SEVEN_A_SIDE",
  "location": "Central Park Pitch 2",
  "proposedDate": "2026-05-30",
  "confirmationDeadline": "2026-05-29T20:00:00Z",
  "status": "PENDING",
  "confirmedCount": 0,
  "minPlayersRequired": 14,
  "description": "Bring bibs",
  "createdBy": "admin",
  "playersNeeded": 14,
  "pollOpen": true,
  "createdAt": "2026-05-22T10:00:00Z",
  "updatedAt": "2026-05-22T10:00:00Z"
}
```

### Player Self-Confirms

```http
POST /api/match-plans/5/confirmations/me HTTP/1.1
Content-Type: application/json
Authorization: Bearer <player-token>

{
  "status": "CONFIRMED",
  "notes": "Will be 5 minutes late"
}
```

**Response `200 OK`:**
```json
{
  "id": 33,
  "matchPlanId": 5,
  "playerId": 42,
  "playerName": "João Silva",
  "status": "CONFIRMED",
  "notes": "Will be 5 minutes late",
  "confirmedAt": "2026-05-22T11:00:00Z"
}
```

### Generate Team Preview (BALANCED)

```http
POST /api/match-plans/5/generate?generationType=BALANCED HTTP/1.1
Authorization: Bearer <admin-token>
```

**Response `200 OK`:**
```json
{
  "matchPlanId": 5,
  "matchType": "SEVEN_A_SIDE",
  "generationType": "BALANCED",
  "location": "Central Park Pitch 2",
  "proposedDate": "2026-05-30",
  "generationNotes": "Teams balanced by skill rating using greedy bin-packing",
  "teamARatingAvg": 7.14,
  "teamBRatingAvg": 7.12,
  "ratingDelta": 0.02,
  "teams": [
    {
      "name": "Team A",
      "ratingAvg": 7.14,
      "players": [
        { "playerId": 1, "playerName": "João Silva", "skillRating": 9.0 },
        { "playerId": 3, "playerName": "Carlos M.", "skillRating": 7.5 }
      ]
    },
    {
      "name": "Team B",
      "ratingAvg": 7.12,
      "players": [
        { "playerId": 2, "playerName": "Pedro Costa", "skillRating": 8.5 }
      ]
    }
  ]
}
```

### Confirm Generation (Creates the Match)

```http
POST /api/match-plans/5/generate/confirm?generationType=BALANCED HTTP/1.1
Authorization: Bearer <admin-token>
```

**Response `201 Created`:** Full `MatchDTO` (same shape as `POST /api/matches` response).

---

## Caching Strategy

Match plans use the `matches` Caffeine cache (shared with the Match feature):

| Cache Name | Populated By         | Evicted When          |
|------------|----------------------|-----------------------|
| `matches`  | `GET /api/match-plans/{id}` | Any write to match plans |

---

## Implementation Details

- **Entity:** `MatchPlan.java`, `PlayerConfirmation.java` — JPA entities with Lombok
- **Service:** `MatchPlanService.java` — business logic, confirmation management, team generation dispatch
- **Controller:** `MatchPlanController.java` — REST layer; `@AuthenticationPrincipal UserPrincipal` used for self-service confirmation
- **Team generation:** Delegated to `TeamGenerationStrategyFactory` → `BalancedGenerationStrategy` / `RandomGenerationStrategy` / `SnakeDraftGenerationStrategy`
- **DTOs:** `MatchPlanCreateDTO`, `MatchPlanDTO`, `MatchPlanUpdateDTO`, `MatchPlanStatusDTO`, `ConfirmationUpsertDTO`, `PlayerConfirmationDTO`, `MatchPreviewDTO` — all Java Records

---

## Error Reference

| Scenario                                       | Status | Message                                                  |
|------------------------------------------------|--------|----------------------------------------------------------|
| Plan not found                                 | 404    | `MatchPlan with id {id} not found`                       |
| Updating a non-PENDING plan                    | 409    | `Match plan cannot be updated in status {status}`        |
| Deleting a non-PENDING plan                    | 409    | `Match plan cannot be deleted in status {status}`        |
| Invalid status transition                      | 400    | `Invalid status transition from {from} to {to}`          |
| Not enough confirmed players for generation    | 400    | `Not enough confirmed players: need {N}, have {M}`       |
| `CAPTAIN_PICK` or unsupported algorithm        | 422    | `CAPTAIN_PICK is not yet available`                      |
| Player not found for admin confirmation        | 404    | `Player with id {id} not found`                          |
| proposedDate in the past                       | 400    | Validation violation on `proposedDate`                   |

