# Frontend Endpoint Changes — Football Management System

> This file is **append-only**. Never delete or reorder existing entries.  
> Format: newest entry at the bottom.

---

## 2026-05-15 — User Entity & Authentication

### 📍 Endpoints Added

| Method | Path                                    | Change Type | Auth Required      |
|--------|-----------------------------------------|-------------|--------------------|
| POST   | `/api/auth/login`                       | NEW         | Public             |
| GET    | `/api/users`                         | NEW         | `ADMIN_USER`       |
| GET    | `/api/users/me`                      | NEW         | Any authenticated  |
| GET    | `/api/users/{id}`                    | NEW         | `ADMIN_USER` or own|
| POST   | `/api/users`                         | NEW         | `ADMIN_USER`       |
| PATCH  | `/api/users/{id}`                    | NEW         | `ADMIN_USER` or own|
| PATCH  | `/api/users/{id}/role`               | NEW         | `ADMIN_USER`       |
| DELETE | `/api/users/{id}`                    | NEW         | `ADMIN_USER`       |
| POST   | `/api/users/{id}/change-password`    | NEW         | `ADMIN_USER` or own|

### 📥 Request Bodies

**`LoginRequestDTO`** — POST /api/auth/login:
```json
{
  "identifier": "johndoe",
  "password": "myPassword123"
}
```
> `identifier` accepts a **username** or **email** — backend resolves automatically.

**`UserCreateDTO`** — POST /api/users:
```json
{
  "username": "janedoe",
  "email": "jane@example.com",
  "password": "securePass1",
  "firstName": "Jane",
  "lastName": "Doe",
  "role": "BASIC_USER"
}
```
> `role` must be exactly one of: `"BASIC_USER"`, `"MASTER_USER"`, `"ADMIN_USER"`.

**`UserUpdateDTO`** — PATCH /api/users/{id}:
```json
{
  "firstName": "Jane",
  "lastName": "Smith",
  "email": "jane.smith@example.com"
}
```
> All fields optional — send only the ones you want to change.

**`AdminUserUpdateDTO`** — PATCH /api/users/{id}/role:
```json
{
  "role": "MASTER_USER",
  "isActive": true
}
```
> Both fields optional — send only the ones you want to change.

**`ChangePasswordDTO`** — POST /api/users/{id}/change-password:
```json
{
  "currentPassword": "oldPassword1",
  "newPassword": "newPassword2"
}
```

### 📤 Response Shape

**`LoginResponseDTO`** — POST /api/auth/login:
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

**`UserDTO`** — all user read endpoints:
```json
{
  "id": 5,
  "username": "johndoe",
  "email": "john@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "role": "BASIC_USER",
  "isActive": true,
  "forcePasswordChange": false,
  "createdAt": "2026-05-15T10:30:00Z"
}
```

### 🔄 Integration Notes

- [ ] Store `token` from login response in secure storage (e.g., `httpOnly` cookie or memory — avoid `localStorage`)
- [ ] Send token as `Authorization: Bearer <token>` header on all authenticated requests
- [ ] Check `forcePasswordChange` after login — if `true`, redirect to change-password screen before allowing navigation
- [ ] `GET /api/users` returns a Spring `Page` object — handle `content`, `totalElements`, `totalPages`, `number` fields
- [ ] `DELETE /api/users/{id}` is a **soft delete** — user still exists in DB with `isActive: false`; frontend should reflect this state
- [ ] `createdAt` is ISO-8601 UTC (e.g., `"2026-05-15T10:30:00Z"`) — format locally per locale

### ⚠️ Breaking Changes

**NONE** — this is the initial implementation; all endpoints are new.

---

## 2026-05-15 — Player Entity & CRUD API

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| GET | `/api/players` | NEW | Any authenticated |
| GET | `/api/players/{id}` | NEW | Any authenticated |
| POST | `/api/players` | NEW | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/players/{id}` | NEW | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/players/{id}/status` | NEW | `ADMIN_USER` or `MASTER_USER` |
| DELETE | `/api/players/{id}` | NEW | `ADMIN_USER` only |

### 📥 Request Changes

**`PlayerCreateDTO`** — POST /api/players:
```json
{
  "name": "João Silva",
  "baseSkillRating": 7,
  "phoneNumber": "+351912345678",
  "isActive": true,
  "userId": 5
}
```
> `name` is required (2–100 chars). `baseSkillRating` is required (integer 1–10). All other fields optional.  
> `userId` links an existing `AppUser` to the player. Sending a `userId` belonging to an `ADMIN_USER` account returns `403 Forbidden`.

**`PlayerUpdateDTO`** — PATCH /api/players/{id}:
```json
{
  "name": "João M. Silva",
  "phoneNumber": "+351987654321",
  "userId": 10,
  "unlinkUser": true
}
```
> All fields are optional — send only what you want to change.  
> ⚠️ **Important:** to remove a player's user link, you **must** send `"unlinkUser": true`. Sending `"userId": null` alone is treated as "no change" (safe PATCH semantics). Sending both `userId` and `unlinkUser: true` results in `unlinkUser` taking precedence.

**`PlayerStatusDTO`** — PATCH /api/players/{id}/status:
```json
{ "isActive": false }
```
> `isActive` is required (`@NotNull`). Use this endpoint to activate or deactivate a player without touching other fields.

### 📤 Response Changes

**`PlayerDTO`** — all player endpoints return this shape:

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
  "linkedUserId": 5,
  "createdBy": "admin",
  "updatedBy": "master_user",
  "createdAt": "2026-05-15T10:30:00Z",
  "updatedAt": "2026-05-15T11:00:00Z"
}
```

`GET /api/players` returns a Spring `Page` wrapper:
```json
{
  "content": [ /* PlayerDTO array */ ],
  "totalElements": 25,
  "totalPages": 3,
  "number": 0,
  "size": 10
}
```

### 🔄 Migration Notes

What frontend developers must implement:

- [ ] Create TypeScript type for `PlayerDTO` — include all 14 fields; `email`, `phoneNumber`, `linkedUserId` are nullable
- [ ] Create TypeScript types for `PlayerCreateDTO`, `PlayerUpdateDTO`, `PlayerStatusDTO`
- [ ] Handle paginated response from `GET /api/players` — same `Page<T>` shape as `GET /api/users`
- [ ] `skillRating` is a `double` (e.g., `7.35`) — display with 1–2 decimal places
- [ ] `baseSkillRating` is an integer (1–10) — never changes after creation
- [ ] `email` may be `null` if the player has no linked user account — guard against null in templates
- [ ] `DELETE /api/players/{id}` may return `409 Conflict` if the player has match stats — display a user-friendly message
- [ ] To unlink a user from a player, send `PATCH /api/players/{id}` with body `{ "unlinkUser": true }` (not `{ "userId": null }`)
- [ ] `createdAt` and `updatedAt` are ISO-8601 UTC — format locally per locale

### ⚠️ Breaking Changes

**NONE** — these are all new endpoints.


---

## 2026-05-17 — Match & Team Feature

### 📍 Endpoints Added

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| POST | `/api/matches` | NEW | `ADMIN_USER` or `MASTER_USER` |
| GET | `/api/matches` | NEW | Any authenticated |
| GET | `/api/matches/{id}` | NEW | Any authenticated |
| PATCH | `/api/matches/{id}` | NEW | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/matches/{id}/complete` | NEW | `ADMIN_USER` or `MASTER_USER` |
| GET | `/api/matches/{id}/teams` | ~~NEW~~ **since removed** — `MatchDTO.teams` already carries this | Any authenticated |
| GET | `/api/matches/{id}/teams/{teamId}` | ~~NEW~~ **since removed** — read it from `MatchDTO.teams` | Any authenticated |
| PATCH | `/api/matches/{id}/teams/{teamId}/stats/{statId}` | NEW | `ADMIN_USER` only |
| DELETE | `/api/matches/{id}` | NEW | `ADMIN_USER` only |

### 📥 Request Changes

**`MatchCreateDTO`** — POST /api/matches:
```json
{
  "description": "Friendly Friday",
  "matchDate": "2026-05-20T19:00:00Z",
  "location": "Central Park",
  "matchType": "SEVEN_A_SIDE",
  "generationType": "manual",
  "seasonId": 1,
  "teams": [
    { "name": "Team A", "playerIds": [1, 2, 3, 4, 5, 6, 7] },
    { "name": "Team B", "playerIds": [8, 9, 10, 11, 12, 13, 14] }
  ]
}
```
> `matchType`: `"FIVE_A_SIDE"` | `"SEVEN_A_SIDE"` | `"ELEVEN_A_SIDE"`  
> `teams`: exactly 2 entries; each must have at least 1 player.

**`MatchCompleteDTO`** — PATCH /api/matches/{id}/complete:
```json
{
  "scoreTeamA": 3,
  "scoreTeamB": 2,
  "winningTeamId": 1,
  "playerStats": [
    { "playerStatId": 10, "goals": 2, "assists": 1, "ownGoals": 0, "rating": 8.5, "isMvp": true }
  ]
}
```
> `winningTeamId` is optional — omit for a draw.  
> After completion, player skill ratings and streaks are **automatically recalculated** via `CalculationService`.

**`MatchUpdateDTO`** — PATCH /api/matches/{id}:
```json
{
  "description": "Updated name",
  "matchDate": "2026-05-20T20:00:00Z",
  "location": "New Venue"
}
```
> All fields optional. Only allowed before match is completed.

**`PlayerStatUpdateDTO`** — PATCH /api/matches/{id}/teams/{teamId}/stats/{statId}:
```json
{ "playerStatId": 10, "goals": 1, "assists": 2, "ownGoals": 0, "rating": 7.5, "isMvp": false }
```

### 📤 Response Changes

**`MatchDTO`** — all match read endpoints:
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
  "generationType": "manual",
  "generationNotes": null,
  "seasonId": 1,
  "teams": [
    {
      "id": 1, "name": "Team A", "teamOrder": 1,
      "playerStats": [
        { "id": 10, "playerId": 42, "playerName": "João Silva",
          "goals": 0, "assists": 0, "ownGoals": 0,
          "rating": null, "isMvp": false, "matchResult": null }
      ]
    }
  ],
  "createdAt": "2026-05-17T10:00:00Z",
  "updatedAt": "2026-05-17T10:00:00Z"
}
```

### 🔄 Integration Notes

- [ ] `matchType` is always uppercase: `"FIVE_A_SIDE"`, `"SEVEN_A_SIDE"`, `"ELEVEN_A_SIDE"`
- [ ] `generationType` is always uppercase: `"MANUAL"`, `"BALANCED"`, `"RANDOM"`, `"SNAKE_DRAFT"`, `"FORM_BASED"`, `"CAPTAIN_PICK"`
- [ ] `scoreTeamA`, `scoreTeamB`, `finalScore`, `winningTeamId` are `null` until match is completed
- [ ] `PlayerStatDTO.matchResult` is `null` until completion, then `"WIN"` | `"LOSS"` | `"DRAW"`
- [ ] `PlayerStatDTO.rating` is `null` until provided at completion time
- [ ] `DELETE /api/matches/{id}` returns `409 Conflict` if match is already completed
- [ ] Completing a match triggers automatic skill rating recalculation — `skillRating` on `PlayerDTO` will have changed after `PATCH /api/matches/{id}/complete`
- [ ] `GET /api/matches` supports query params: `?seasonId=1`, `?completed=false`, `?matchType=SEVEN_A_SIDE`

### ⚠️ Breaking Changes

**NONE** — all new endpoints.

---

## 2026-05-17 — Match Plans & Team Generation

### 📍 Endpoints Added

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| POST | `/api/match-plans` | NEW | `ADMIN_USER` or `MASTER_USER` |
| GET | `/api/match-plans` | NEW | Any authenticated |
| GET | `/api/match-plans/{id}` | NEW | Any authenticated |
| PATCH | `/api/match-plans/{id}` | NEW | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/match-plans/{id}/status` | NEW | `ADMIN_USER` or `MASTER_USER` |
| DELETE | `/api/match-plans/{id}` | NEW | `ADMIN_USER` only |
| GET | `/api/match-plans/{id}/confirmations` | NEW | Any authenticated |
| POST | `/api/match-plans/{id}/confirmations/me` | NEW | Any authenticated |
| GET | `/api/match-plans/{id}/confirmations/me` | NEW | Any authenticated |
| PATCH | `/api/match-plans/{id}/confirmations/{playerId}` | NEW | `ADMIN_USER` or `MASTER_USER` |
| POST | `/api/match-plans/{id}/generate` | NEW | `ADMIN_USER` or `MASTER_USER` |
| POST | `/api/match-plans/{id}/generate/confirm` | NEW | `ADMIN_USER` or `MASTER_USER` |

### 📥 Request Changes

**`MatchPlanCreateDTO`** — POST /api/match-plans:
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
> `proposedDate` must be a **future date** (`LocalDate`). `confirmationDeadline` and `description` are optional. `minPlayersRequired` defaults to `14`.

**`ConfirmationUpsertDTO`** — POST /confirmations/me or PATCH /confirmations/{playerId}:
```json
{ "status": "CONFIRMED", "notes": "Will be 10 minutes late" }
```
> `status`: `"CONFIRMED"` | `"DECLINED"` | `"PENDING"`. `notes` optional.

**`MatchPlanStatusDTO`** — PATCH /api/match-plans/{id}/status:
```json
{ "status": "CONFIRMED" }
```
> Valid transitions: `PENDING→CONFIRMED`, `PENDING→CANCELLED`, `CONFIRMED→CANCELLED`.

**Team generation** — POST /api/match-plans/{id}/generate (query params only):
```
POST /api/match-plans/5/generate?generationType=BALANCED
POST /api/match-plans/5/generate?generationType=FORM_BASED&params[formWindow]=3
POST /api/match-plans/5/generate?generationType=CAPTAIN_PICK&params[captainAId]=42&params[captainBId]=17
```
> `generationType`: `BALANCED` (default) | `RANDOM` | `SNAKE_DRAFT` | `FORM_BASED` | `CAPTAIN_PICK`  
> `STREAK_AWARE` is declared but returns `422 Unprocessable Entity` — do not surface it to end users yet.  
> No request body — generation reads confirmed players from the plan.

### 📤 Response Changes

**`MatchPlanDTO`** — all match plan read endpoints:
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

**`PlayerConfirmationDTO`** — confirmation endpoints:
```json
{
  "id": 33, "matchPlanId": 5, "playerId": 42, "playerName": "João Silva",
  "status": "CONFIRMED", "notes": "Will be 10 minutes late",
  "confirmedAt": "2026-05-17T12:00:00Z"
}
```

**`MatchPreviewDTO`** — POST /api/match-plans/{id}/generate:
```json
{
  "matchPlanId": 5, "matchType": "SEVEN_A_SIDE", "generationType": "BALANCED",
  "location": "Central Park", "proposedDate": "2026-05-23",
  "generationNotes": "Teams balanced by skill rating",
  "teamARatingAvg": 7.14, "teamBRatingAvg": 7.12, "ratingDelta": 0.02,
  "teams": [
    { "name": "Team A", "ratingAvg": 7.14,
      "players": [{ "playerId": 1, "playerName": "João Silva", "skillRating": 8.5 }] },
    { "name": "Team B", "ratingAvg": 7.12,
      "players": [{ "playerId": 2, "playerName": "Pedro Costa", "skillRating": 7.0 }] }
  ]
}
```
> This is a **stateless preview** — nothing is persisted. Call `POST /generate/confirm` to create the actual match.

### 🔄 Integration Notes

- [ ] **Team generation flow**: `POST /generate` → show preview to admin → admin approves → `POST /generate/confirm` → `MatchDTO` returned (match created)
- [ ] `proposedDate` is a `LocalDate` string (`"2026-05-23"` — no time component)
- [ ] `pollOpen` is a computed boolean on `MatchPlanDTO` — when `false`, the confirm/decline button should be disabled
- [ ] `playersNeeded` is computed — show progress bar: `confirmedCount / minPlayersRequired`
- [ ] `status` values: `"PENDING"` | `"CONFIRMED"` | `"CANCELLED"`
- [ ] `GET /api/match-plans` supports `?status=PENDING` filter
- [ ] `GET /api/match-plans/{id}/confirmations` supports `?status=CONFIRMED` filter
- [ ] `DELETE /api/match-plans/{id}` returns `409` if plan is not `PENDING`
### ⚠️ Breaking Changes

**NONE** — all new endpoints.

---

## 2026-05-18 — Version Endpoint & Health Response Update

### 📍 Endpoints Added / Changed

| Method | Path           | Change Type | Auth    | Description                          |
|--------|----------------|-------------|---------|--------------------------------------|
| GET    | `/api/version` | NEW         | Public  | Returns application version metadata |
| GET    | `/api/health`  | CHANGED     | Public  | Response body `version` now sourced from build (was hardcoded) |

### 📥 Responses

**`GET /api/version`** — new endpoint:
```json
{
  "application": "football",
  "version": "1.0.0",
  "group": "pt.rics.demo",
  "buildTime": "2026-05-18T08:30:00Z"
}
```

**`GET /api/health`** — shape unchanged, `version` is now dynamic:
```json
{
  "status": "UP",
  "version": "1.0.0",
  "timestamp": "2026-05-18T08:30:00Z"
}
```

### 🔄 Integration Notes

- [ ] `/api/version` is **fully public** — no JWT required; safe to call before login (UI footer / about screen)
- [ ] `/api/health` is also public — use as a connectivity check on app startup
- [ ] `buildTime` is an ISO-8601 UTC instant — display as localised date if needed
- [ ] `version` on both endpoints is guaranteed to match the deployed JAR version

### ⚠️ Breaking Changes

**NONE** — `/api/health` response shape is unchanged; `version` field was already present.

---

## 2026-05-19 — Match Live Stats & Completion Overhaul

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| PATCH  | `/api/matches/{id}/complete` | MODIFIED | `ADMIN_USER` or `MASTER_USER` |
| PATCH  | `/api/matches/{id}/stats/live` | NEW | `ADMIN_USER` or `MASTER_USER` |

### 📥 Request Changes

**`PlayerStatUpdateDTO`** (used in `MatchCompleteDTO.playerStats`, `MatchLiveUpdateDTO.playerStats`, and `PATCH .../stats/{statId}`):

**Before:**
```json
{ "playerStatId": 10, "goals": 2, "assists": 1, "ownGoals": 0, "rating": 8.5, "isMvp": true }
```

**After:**
```json
{ "playerStatId": 10, "soloGoals": 1, "assistedGoals": 1, "penaltyGoals": 0, "assists": 1, "ownGoals": 0, "isMvp": true }
```

> `goals` has been **replaced** by three separate fields: `soloGoals`, `assistedGoals`, `penaltyGoals`.  
> `rating` has been **removed** — the server now computes all ratings.  
> All fields except `playerStatId` are optional (safe PATCH semantics — `null` = no change).

**`MatchCompleteDTO`** — PATCH /api/matches/{id}/complete:
```json
{
  "scoreTeamA": 3,
  "scoreTeamB": 2,
  "winningTeamId": 1,
  "playerStats": [
    { "playerStatId": 10, "soloGoals": 1, "assistedGoals": 1, "penaltyGoals": 0, "assists": 1, "ownGoals": 0, "isMvp": true },
    { "playerStatId": 11, "soloGoals": 0, "assistedGoals": 0, "penaltyGoals": 0, "assists": 0, "ownGoals": 1, "isMvp": false }
  ]
}
```

**`MatchLiveUpdateDTO`** — PATCH /api/matches/{id}/stats/live (NEW endpoint):
```json
{
  "playerStats": [
    { "playerStatId": 10, "soloGoals": 1, "assistedGoals": 0, "penaltyGoals": 0, "assists": 1, "ownGoals": 0, "isMvp": false }
  ]
}
```

### 📤 Response Changes

**`PlayerStatDTO`** (inside `MatchDTO.teams[].playerStats`) — new goal-type fields added:

**Before:**
```json
{ "id": 10, "playerId": 42, "playerName": "João Silva", "goals": 2, "assists": 1, "ownGoals": 0, "rating": null, "isMvp": false, "matchResult": null }
```

**After:**
```json
{ "id": 10, "playerId": 42, "playerName": "João Silva", "soloGoals": 1, "assistedGoals": 1, "penaltyGoals": 0, "assists": 1, "ownGoals": 0, "rating": 8.5, "isMvp": true, "matchResult": "WIN" }
```

> `rating` is `null` before completion, then populated by server after `PATCH /complete`.

**`MatchLiveUpdateResponseDTO`** — response for NEW `PATCH /api/matches/{id}/stats/live`:
```json
{
  "ratings": [
    { "playerStatId": 10, "playerId": 42, "playerName": "João Silva", "matchRating": 6.3 },
    { "playerStatId": 11, "playerId": 43, "playerName": "Pedro Costa", "matchRating": 4.6 }
  ]
}
```

> These are **preview ratings** — WIN/LOSS bonus and goal-diff bonus are not applied. Final ratings come from `PATCH /complete`.

### 🔄 Migration Notes

What frontend developers must update:

- [ ] **Remove `rating` from any stat submission forms** — field no longer accepted; will be ignored or cause validation error
- [ ] **Replace `goals` input with three separate fields:** `soloGoals`, `assistedGoals`, `penaltyGoals` — update all match completion UIs
- [ ] **Update TypeScript type for `PlayerStatUpdateDTO`**: remove `goals` and `rating`, add `soloGoals`, `assistedGoals`, `penaltyGoals`
- [ ] **Update TypeScript type for `PlayerStatDTO`**: remove `goals`, add `soloGoals`, `assistedGoals`, `penaltyGoals`; `rating` is now always server-sourced (read-only)
- [ ] **Add types for new DTOs**: `MatchLiveUpdateDTO`, `MatchLiveUpdateResponseDTO`, `PlayerMatchRatingDTO`
- [ ] **Implement live stats flow** (optional but recommended): call `PATCH /api/matches/{id}/stats/live` during match to get live preview ratings; display to admin; call `PATCH /complete` to finalize
- [ ] **Handle compliance validation error**: `PATCH /complete` returns `400 Bad Request` if submitted goal counts don't match `scoreTeamA`/`scoreTeamB`; show a user-friendly validation message

### ⚠️ Breaking Changes

**BREAKING — `PlayerStatUpdateDTO`:**
- `goals` field has been **removed** — replaced by `soloGoals`, `assistedGoals`, `penaltyGoals`
- `rating` field has been **removed** — server now computes all match ratings

**BREAKING — `PlayerStatDTO` (response):**
- `goals` field has been **removed** — replaced by `soloGoals`, `assistedGoals`, `penaltyGoals`

Any frontend type definitions or form submissions referencing `goals` or `rating` on player stat objects **will break** and must be updated.

**Rating weights (for display info only — computation is server-side):**

| Goal Type      | Weight |
|----------------|--------|
| Solo goal      | +0.50  |
| Assisted goal  | +0.30  |
| Penalty goal   | +0.15  |
| Assist         | +0.20  |
| Own goal       | -0.40  |
| Win bonus      | +0.50  |
| Loss penalty   | -0.30  |

---

## 2026-05-27 — Draft Session SSE Real-Time Events

### 📍 Endpoints Added

| Method | Path | Change Type | Auth Required | Produces |
|--------|------|-------------|---------------|----------|
| GET | `/api/draft-sessions/{id}/events` | NEW | Any authenticated | `text/event-stream` |

### ⚡ What This Enables

Instead of polling `GET /api/draft-sessions/{id}` for pick updates, the frontend subscribes once to the SSE stream and receives push events for every state change.

### 🔔 SSE Event Types

| Event | When | Payload | Terminal? |
|-------|------|---------|-----------|
| `CONNECTED` | Immediately on subscribe | Full `DraftSessionDTO` snapshot | No |
| `PICK` | After each pick (status=OPEN) | Full `DraftSessionDTO` | No |
| `COMPLETED` | All players picked (status=COMPLETED) | Full `DraftSessionDTO` | No |
| `CANCELLED` | Session cancelled | Full `DraftSessionDTO` | **Yes — close connection** |
| `CONVERTED` | Match confirmed | Full `DraftSessionDTO` (status=CONVERTED) | **Yes — close connection** |

### ⚠️ Breaking / Behaviour Change

None. Existing REST endpoints are unchanged. SSE is additive.

### 📌 Implementation Notes

- **Do not use `new EventSource(url)`** — it cannot send the JWT `Authorization` header. Use the Fetch API instead.
- **Do not manually update state after a pick** — the `PICK` SSE event arrives milliseconds later and updates all clients including the picker.
- **Reconnect on 5-minute timeout** — the server closes the connection after 5 min; reconnect if session is still OPEN/COMPLETED.
- Full integration guide: [`docs/frontend/DRAFT_SESSION_SSE_GUIDE.md`](./DRAFT_SESSION_SSE_GUIDE.md)

---

## 2026-05-28 — Team Generation Types Expansion

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| POST | `/api/match-plans/{id}/generate` | MODIFIED (new `generationType` values + `params` map) | `ADMIN_USER` or `MASTER_USER` |
| POST | `/api/match-plans/{id}/generate/confirm` | MODIFIED (same param expansion) | `ADMIN_USER` or `MASTER_USER` |

No response shape changes — `MatchPreviewDTO` and `MatchDTO` are unchanged.

### ⚡ What Changed

Four new values are now accepted for the `generationType` query parameter (expanding from 3 to 6 active values). A new `params` map query parameter enables per-algorithm configuration.

### 📥 Updated Request Params

Both `/generate` and `/generate/confirm` accept the same query params:

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `generationType` | String | `BALANCED` | Algorithm selector — see table below |
| `params[formWindow]` | int (≥1) | `5` | `FORM_BASED` only — recent match window per player |
| `params[captainAId]` | Long | auto | `CAPTAIN_PICK` only — Team A captain player ID |
| `params[captainBId]` | Long | auto | `CAPTAIN_PICK` only — Team B captain player ID |

#### All `generationType` Values

| Value | Status | Description | Extra Params |
|-------|--------|-------------|--------------|
| `BALANCED` | ✅ Active | Greedy bin-packing by `skillRating` — most balanced, deterministic | none |
| `RANDOM` | ✅ Active | Pure random shuffle — no balance guarantee, each call produces a different result | none |
| `SNAKE_DRAFT` | ✅ Active | Alternating snake pick by rating rank — near-optimal, deterministic | none |
| `FORM_BASED` | ✅ Active | Balances by linearly-weighted recent match ratings; falls back to `skillRating` for new players | `formWindow` |
| `STREAK_AWARE` | ⚠️ Not yet active | Returns `422 Unprocessable Entity` — pending CalculationService streak data | — |
| `CAPTAIN_PICK` | ✅ Active | Server-side snake draft simulation; auto-selects top-2 captains unless overridden | `captainAId`, `captainBId` |

#### Example Calls

```
# Default balanced split
POST /api/match-plans/5/generate?generationType=BALANCED

# Form-based using last 3 matches
POST /api/match-plans/5/generate?generationType=FORM_BASED&params[formWindow]=3

# Captain pick with explicit captains
POST /api/match-plans/5/generate?generationType=CAPTAIN_PICK&params[captainAId]=42&params[captainBId]=17

# Captain pick with auto-selected captains
POST /api/match-plans/5/generate?generationType=CAPTAIN_PICK
```

### 📤 Updated Response Fields

**`MatchPreviewDTO.generationNotes`** and **`MatchDTO.generationNotes`** now contain algorithm-specific information rather than a generic string:

| `generationType` | `generationNotes` format |
|------------------|--------------------------|
| `BALANCED` | `"BALANCED (greedy): avgA=7.53 avgB=7.50 Δ=0.03"` |
| `RANDOM` | `"RANDOM: avgA=6.80 avgB=7.23 (no balance guarantee)"` |
| `SNAKE_DRAFT` | `"SNAKE_DRAFT: avgA=7.06 avgB=7.11 Δ=0.05"` |
| `FORM_BASED` | `"FORM_BASED (window=5): avgFormA=7.81 avgFormB=7.78 Δ=0.03"` |
| `CAPTAIN_PICK` | `"CAPTAIN_PICK: captainA=João Silva captainB=Miguel Santos avgA=7.43 avgB=7.21 Δ=0.22"` |
| `MANUAL` | `null` |

> ⚠️ **`RANDOM` non-determinism:** the stateless preview (`POST /generate`) and the match creation call (`POST /generate/confirm`) run the shuffle independently. The teams shown in the preview **will differ** from the confirmed teams. If you want the admin to approve exactly the teams they saw, use `BALANCED`, `SNAKE_DRAFT`, `FORM_BASED`, or `CAPTAIN_PICK` (all deterministic).

### 🔄 Integration Notes

What frontend developers must implement or update:

- [ ] **Expand `generationType` selector UI** to include `BALANCED`, `RANDOM`, `SNAKE_DRAFT`, `FORM_BASED`, `CAPTAIN_PICK` — hide or grey out `STREAK_AWARE` until further notice
- [ ] **Add `formWindow` input** (integer spinner, 1–10, default 5) that appears only when `FORM_BASED` is selected
- [ ] **Add captain picker dropdowns** (`captainAId`, `captainBId`) that appear only when `CAPTAIN_PICK` is selected; populate from the confirmed player list; if left empty the server auto-selects
- [ ] **Parse `generationNotes` on `MatchPreviewDTO`** — display it below the team preview so admins understand how teams were formed (e.g., show Δ rating)
- [ ] **Parse `generationNotes` on `MatchDTO`** — show on match detail page as "How teams were generated"
- [ ] **Warn users about RANDOM non-determinism** — the preview is illustrative; confirm produces a fresh shuffle
- [ ] **Handle `422 Unprocessable Entity`** for `STREAK_AWARE` — display message: "Streak-aware generation is not yet available"
- [ ] **Handle `400 Bad Request`** when `captainAId`/`captainBId` reference a player not in the confirmed pool
- [ ] **Update TypeScript types** — `GenerationType` union should be: `'BALANCED' | 'RANDOM' | 'SNAKE_DRAFT' | 'FORM_BASED' | 'STREAK_AWARE' | 'CAPTAIN_PICK' | 'MANUAL'`

#### Suggested TypeScript Types

```typescript
export type GenerationType =
  | 'BALANCED'
  | 'RANDOM'
  | 'SNAKE_DRAFT'
  | 'FORM_BASED'
  | 'STREAK_AWARE'
  | 'CAPTAIN_PICK'
  | 'MANUAL';

export interface GenerationParams {
  formWindow?: number;    // FORM_BASED only — default 5
  captainAId?: number;   // CAPTAIN_PICK only — omit for auto-select
  captainBId?: number;   // CAPTAIN_PICK only — omit for auto-select
}

// Query params builder
function buildGenerationQuery(
  type: GenerationType,
  params?: GenerationParams
): string {
  const qs = new URLSearchParams({ generationType: type });
  if (params?.formWindow) qs.set('params[formWindow]', String(params.formWindow));
  if (params?.captainAId) qs.set('params[captainAId]', String(params.captainAId));
  if (params?.captainBId) qs.set('params[captainBId]', String(params.captainBId));
  return qs.toString();
}
```

### ⚠️ Breaking Changes

**NONE** — existing calls using `BALANCED`, `RANDOM`, or `SNAKE_DRAFT` without any `params` work exactly as before. The `STREAK_AWARE` enum value existed previously and already returned `422`.

**Behavioural clarification (not breaking):**
- `generationNotes` values are now algorithm-specific strings rather than a generic description. Any frontend code that displayed `generationNotes` verbatim continues to work; code that pattern-matched against `"Teams balanced by skill rating"` must be updated.

---

## 2026-07-01 — Admin Player User-Link Removal Endpoint

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| DELETE | `/api/players/{id}/user-link` | NEW | `ADMIN_USER` only |

### 📥 Request Changes

No request body. The player ID is supplied as a path variable.

### 📤 Response Changes

**Success `200 OK`** — returns the updated `PlayerDTO` with `linkedUserId` and `email` set to `null`:

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

### 🔄 Migration Notes

- [ ] No new TypeScript types required — response shape is the existing `PlayerDTO`
- [ ] After a successful call, refresh the player record — `linkedUserId` and `email` will be `null`
- [ ] Handle `409 Conflict` with message `"Player is not linked to any user"` — show a user-friendly error (e.g., "This player has no linked account")
- [ ] Handle `404 Not Found` — player does not exist
- [ ] This endpoint is **ADMIN_USER only** — hide or disable the "Unlink user" action for non-admin roles

### ⚠️ Breaking Changes

**NONE** — new endpoint; no existing behaviour changed.

---

## 2026-06-30 — Admin: Reactivate Inactive User

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| PATCH  | `/api/users/{id}/reactivate` | ~~NEW~~ **since removed** — reactivate via `PATCH /{id}/role` with `{"isActive": true}` | `ADMIN_USER` only |

### 📥 Request Changes

No request body. The user ID is supplied as a path variable.

### 📤 Response Changes

**Success `200 OK`** — returns the reactivated `UserDTO` with `isActive: true`:

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

### 🔄 Migration Notes

- [ ] No new TypeScript types required — response shape is the existing `UserDTO`
- [ ] After a successful call, refresh the user record — `isActive` will be `true`
- [ ] Handle `409 Conflict` with message `"User is already active"` — show a user-friendly error (e.g., "This account is already active")
- [ ] Handle `404 Not Found` — user does not exist
- [ ] This endpoint is **ADMIN_USER only** — show the "Reactivate" action only to admin-role sessions
- [ ] Companion deactivation endpoint: `DELETE /api/users/{id}` (sets `isActive: false`, returns `204 No Content`)

### ⚠️ Breaking Changes

**NONE** — new endpoint; no existing behaviour changed.

---

## 2026-07-02 — Admin: Draft Session Summary List & Hard-Purge

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| GET    | `/api/draft-sessions/summary` | NEW | `ADMIN_USER` only |
| DELETE | `/api/draft-sessions/{id}/purge` | NEW | `ADMIN_USER` only |

Both endpoints are additive. The existing `GET /api/draft-sessions` (heavy `DraftSessionDTO` list, any authenticated) and the soft-cancel `DELETE /api/draft-sessions/{id}` (`ADMIN_USER`/`MASTER_USER`) are **unchanged**.

### 📥 Request Changes

No request body for either endpoint.
- `GET /api/draft-sessions/summary` — no path variables, no query params.
- `DELETE /api/draft-sessions/{id}/purge` — `{id}` (`Long`) path variable identifies the session.

### 📤 Response Changes

**`DraftSessionSummaryDTO`** — array returned by `GET /api/draft-sessions/summary` (all sessions, all statuses, sorted newest-first by `createdAt` DESC):

```json
[
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
]
```

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `id` | `number` (Long) | No | Draft session ID |
| `matchPlanId` | `number` (Long) | No | Associated match plan ID |
| `matchPlanTitle` | `string` | No | Associated match plan name |
| `status` | `string` | No | `"OPEN"` \| `"COMPLETED"` \| `"CANCELLED"` \| `"CONVERTED"` |
| `captainAName` | `string` | No | Team A captain display name |
| `captainBName` | `string` | No | Team B captain display name |
| `currentTurn` | `string` | **Yes** | `"A"`/`"B"` only when `status = "OPEN"`; `null` otherwise |
| `totalPlayers` | `number` (int) | No | teamA + teamB + remaining sizes |
| `picksRemaining` | `number` (int) | No | Remaining pool size |
| `createdBy` | `string` | **Yes** | Creator username; may be `null` |
| `expiresAt` | `string` (ISO-8601) | **Yes** | Optional expiry; `null` if none set |
| `createdAt` | `string` (ISO-8601) | No | Session creation timestamp |
| `updatedAt` | `string` (ISO-8601) | No | Last modification timestamp |

> This is a **lightweight** DTO — it does **not** contain the `teamA`/`teamB`/`remaining` player arrays carried by `DraftSessionDTO`. Use it for admin overview tables; use `GET /api/draft-sessions/{id}` when you need the full player lists.

**`DELETE /api/draft-sessions/{id}/purge`** — returns `204 No Content` (empty body) on success.

### 🔀 Soft-Cancel vs. Hard-Purge

There are now **two different delete operations** on a draft session — expose them as distinct actions:

| Action | Endpoint | Auth | Effect | Reversible |
|--------|----------|------|--------|------------|
| **Cancel** | `DELETE /api/draft-sessions/{id}` | `ADMIN_USER` or `MASTER_USER` | Sets `status = "CANCELLED"` — row stays in DB, still appears in the summary list | Row preserved |
| **Purge** | `DELETE /api/draft-sessions/{id}/purge` | `ADMIN_USER` only | Permanently deletes the row — it disappears entirely | **No — irreversible** |

A `CONVERTED` session **cannot be purged** (it is linked to a created match) → the API responds `409 Conflict`. All other statuses (`OPEN`, `COMPLETED`, `CANCELLED`) can be purged.

### 🔄 Migration Notes

- [ ] Add a TypeScript type for `DraftSessionSummaryDTO` (fields above); `currentTurn`, `createdBy`, and `expiresAt` are nullable
- [ ] Point the admin draft-session **overview/list table** at `GET /api/draft-sessions/summary` for a lighter payload
- [ ] Add a **"Permanently delete / Purge"** action that calls `DELETE /api/draft-sessions/{id}/purge` — make it visually distinct from the existing **"Cancel"** action, and confirm with the user since it is irreversible
- [ ] Handle `409 Conflict` on purge (session is `CONVERTED`) — show e.g. "Cannot delete — this draft is linked to a match"
- [ ] Handle `404 Not Found` on purge — session does not exist
- [ ] Both new endpoints are **`ADMIN_USER` only** — hide these controls from `MASTER_USER`/`BASIC_USER` (note: soft-cancel remains available to `MASTER_USER`)

### ⚠️ Breaking Changes

**NONE** — both endpoints are new; existing draft-session endpoints are unchanged.

---

## 2026-07-02 — Draft Session SSE: Resume-on-Reconnect (terminal streams no longer hang)

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| GET | `/api/draft-sessions/{id}/events` | MODIFIED (behavioral, non-breaking) | Any authenticated |

Path, HTTP method, auth rule (`isAuthenticated()`), and `Content-Type: text/event-stream` are all **unchanged**. This is a behavioral enhancement only — no new endpoint, no new DTO, no schema change.

### 📥 Request Changes

No changes to request format. Path variable `{id}` only; no query params; no request body; JWT bearer token as usual.

### 📤 Response Changes

The event **payload** is unchanged — every `data:` is still a full `DraftSessionDTO`, and the `CANCELLED` / `CONVERTED` event names are unchanged. What changed is the **event ordering when you (re)connect to an already-terminal session**.

**Before** — reconnecting to a session that became `CANCELLED` / `CONVERTED` while you were disconnected:
```
event: CONNECTED
data: { ...status: "CANCELLED"... }

(stream then HANGS with no close signal until the 5-minute server timeout)
```

**After** — the server sends the terminal event and closes immediately:
```
event: CONNECTED
data: { ...status: "CANCELLED"... }

event: CANCELLED
data: { ...status: "CANCELLED"... }

(server completes the stream → EOF; client closes the EventSource)
```

Full ordering matrix per status at (re)connect:

| Status at (re)connect | Event sequence | Stream after |
|-----------------------|----------------|--------------|
| *(missing session)* | — (`404` fast-fail, no stream opens) | never opens |
| `OPEN` | `CONNECTED` → *(live `PICK` / `COMPLETED` / `CANCELLED` / `CONVERTED`)* | stays open |
| `COMPLETED` | `CONNECTED` → *(later `CANCELLED` / `CONVERTED`)* | stays open |
| `CANCELLED` | `CONNECTED` → `CANCELLED` → *close* | closes immediately |
| `CONVERTED` | `CONNECTED` → `CONVERTED` → *close* | closes immediately |

`CONNECTED` is **always** sent first as the authoritative baseline, then the single terminal event, then completion.

### 🔁 Resume Flow (dropped stream → reconnect)

Because all draft state is persisted, a dropped SSE stream (5-minute timeout, tab reload, network blip, laptop sleep) is resumed simply by reconnecting to the **same** `/events` endpoint. The `CONNECTED` snapshot is the resume/rehydrate primitive — **no dedicated resume endpoint exists**.

1. Reconnect → receive `CONNECTED` (authoritative `DraftSessionDTO`).
2. If `status ∈ {OPEN, COMPLETED}` → render from the snapshot, keep the stream open, resume picking via `POST /api/draft-sessions/{id}/pick`.
3. If `status ∈ {CANCELLED, CONVERTED}` → the terminal event arrives next; run terminal handling and close the `EventSource`.
4. `GET /api/draft-sessions/{id}` also returns full authoritative state for a manual rehydrate at any time.

### 🔄 Migration Notes

- [ ] **No mandatory change** — clients that already close the `EventSource` on `CANCELLED` / `CONVERTED` (per the SSE guide) automatically benefit: they now close on reconnect instead of waiting out the 5-minute timeout.
- [ ] Confirm auto-reconnect is gated on `status ∈ {OPEN, COMPLETED}`; do **not** reconnect after a terminal event.
- [ ] Treat a terminal event arriving immediately after `CONNECTED` the same as one arriving mid-session.

### ⚠️ Breaking Changes

**NONE** — the only observable difference is that reconnecting to an already-terminal session now receives the terminal event immediately (then EOF) instead of a hanging stream. This is strictly additive and matches behavior clients already implement for mid-session terminal events.

---

## 2026-07-09 — Manual Match Creation (Unrestricted Team Sizes)

### 📍 Endpoints Affected

| Method | Path | Change Type | Auth Required |
|--------|------|-------------|---------------|
| POST | `/api/matches/manual` | NEW | `ADMIN_USER` or `MASTER_USER` |

The existing `POST /api/matches` endpoint is **unchanged**.

### 📥 Request Changes

**`ManualMatchCreateDTO`** — POST /api/matches/manual:

```json
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

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `description` | `string` | Yes | `@NotBlank`, max 255 chars |
| `matchDate` | `string` (ISO-8601) | **Yes** | `@NotNull` — required (unlike `MatchCreateDTO` where it is optional) |
| `location` | `string` | No | Max 255 chars, nullable |
| `matchType` | `string` | Yes | `"FIVE_A_SIDE"` \| `"SEVEN_A_SIDE"` \| `"ELEVEN_A_SIDE"` |
| `seasonId` | `number` (Long) | No | Falls back to current active season if omitted |
| `teams` | `Array<MatchTeamCreateDTO>` | Yes | Exactly 2 entries |

Each team entry (`MatchTeamCreateDTO`):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | `string` | Yes | `@NotBlank`, max 100 chars |
| `playerIds` | `number[]` | Yes | `@NotEmpty` — at least 1 player; both teams must have the **same** count |

**Key difference vs `POST /api/matches`:**

| Rule | `POST /api/matches` | `POST /api/matches/manual` |
|------|---------------------|----------------------------|
| Players per team | Must equal `matchType` count (5 / 7 / 11) | Any count ≥ 1 |
| Equal team sizes | ✅ | ✅ |
| `matchDate` field | Optional | **Required** |
| `generationType` input | Client sends value | **Not accepted** — always `"MANUAL"` |

### 📤 Response Changes

Returns `201 Created` with the standard `MatchDTO` body. No new response fields — same shape as `POST /api/matches`.

Notable response values:
- `generationType` is always `"MANUAL"`
- `generationNotes` is always `null`
- `Location` header set to `/api/matches/{id}`

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
        { "id": 100, "playerId": 1, "playerName": "João Silva",
          "soloGoals": 0, "assistedGoals": 0, "penaltyGoals": 0,
          "assists": 0, "ownGoals": 0, "rating": null, "isMvp": false, "matchResult": null }
      ]
    },
    { "id": 24, "name": "Team B", "teamOrder": 2, "playerStats": [] }
  ],
  "createdAt": "2026-07-09T14:00:00Z",
  "updatedAt": "2026-07-09T14:00:00Z"
}
```

### 🔄 Migration Notes

- [ ] Add a **"Create Manual Match"** UI action visible to `MASTER_USER` and `ADMIN_USER` roles
- [ ] Create a TypeScript type `ManualMatchCreateDTO` — note `matchDate` is **required** (different from `MatchCreateDTO`)
- [ ] The `teams[].playerIds` array size is free-form; remove any client-side validation that restricts it to the `matchType` count on this form
- [ ] Both teams must have the same `playerIds.length` — add client-side validation and show a clear error
- [ ] After creation, the returned `MatchDTO` uses the same type as `POST /api/matches` — no new response types needed
- [ ] Handle `Location` response header if you want to navigate directly to the new match detail page

### ⚠️ Breaking Changes

**NONE** — new endpoint; `POST /api/matches` is unchanged.

