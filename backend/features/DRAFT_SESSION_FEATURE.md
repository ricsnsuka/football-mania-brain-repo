# Draft Session — Interactive Captain Pick

**Added in:** v1.0.0 (Phase 4)  
**Date:** 2026-05-20  
**Status:** ✅ Released

---

## Overview

The Draft Session feature adds a **stateful, interactive Captain Pick** flow to the Football Management System. Rather than having the server silently simulate who each captain would pick (`CaptainPickGenerationStrategy`), this API allows two human captains to take actual turns picking players via real API calls, producing a live draft experience.

A draft session is created from a **CONFIRMED** match plan, proceeds through alternating picks, and concludes when an admin confirms it — at which point a real `Match` entity is created with `generationType=CAPTAIN_PICK`.

---

## Lifecycle

```
                    ┌─────────────────────┐
                    │   MatchPlan         │
                    │   status=CONFIRMED  │
                    └──────────┬──────────┘
                               │  POST /api/draft-sessions
                               ▼
                    ┌─────────────────────┐
                    │   DraftSession      │
                    │   status=OPEN       │◄─── picks ongoing (alternating A→B→A…)
                    └──────────┬──────────┘
              ┌────────────────┼────────────────┐
              │ all players    │                │
              │ picked         │                │
              ▼                │            DELETE /{id}
  ┌──────────────────────┐     │         ┌────────────────┐
  │   status=COMPLETED   │     │         │ status=CANCELLED│
  └──────────┬───────────┘     │         └────────────────┘
             │                 │                ▲
    POST /{id}/confirm         │         DELETE /{id} (also valid
             │             DELETE /{id}   from COMPLETED)
             ▼
  ┌──────────────────────┐
  │   Match created      │
  │   status=CONVERTED   │
  └──────────────────────┘
```

### Status Values

| Status | Meaning |
|--------|---------|
| `OPEN` | Draft in progress — picks are being submitted |
| `COMPLETED` | All remaining players picked — awaiting confirm |
| `CANCELLED` | Cancelled before or after completion |
| `CONVERTED` | Match has been created from this draft (terminal) |

> ⚠️ `CONVERTED` and `CANCELLED` are terminal — cancellation of a `CONVERTED` session returns `409 Conflict`.

---

## API Endpoints

Base path: `/api/draft-sessions`

| Method | Path | Description | Auth | Request Body | Response |
|--------|------|-------------|------|-------------|----------|
| GET | `/api/draft-sessions` | List all draft sessions | Any authenticated | — | `List<DraftSessionDTO>` (200) |
| POST | `/api/draft-sessions` | Create draft session from a CONFIRMED match plan | `ADMIN_USER` or `MASTER_USER` | `DraftSessionCreateDTO` | `DraftSessionDTO` (201) |
| GET | `/api/draft-sessions/{id}` | Get current draft state | Any authenticated | — | `DraftSessionDTO` (200) |
| GET | `/api/draft-sessions/{id}/events` | Subscribe to real-time draft events via SSE | Any authenticated | — | `text/event-stream` (200) |
| POST | `/api/draft-sessions/{id}/pick` | Submit a captain's pick for the current turn | Any authenticated | `DraftPickDTO` | `DraftSessionDTO` (200) |
| POST | `/api/draft-sessions/{id}/confirm` | Finalize COMPLETED draft and create a Match | `ADMIN_USER` or `MASTER_USER` | — | `MatchDTO` (201) |
| DELETE | `/api/draft-sessions/{id}` | Cancel an OPEN or COMPLETED draft session (soft-cancel → `CANCELLED`) | `ADMIN_USER` or `MASTER_USER` | — | `204 No Content` |
| GET | `/api/draft-sessions/summary` | List **all** draft sessions (all statuses) as lightweight summaries, newest-first | `ADMIN_USER` only | — | `List<DraftSessionSummaryDTO>` (200) |
| DELETE | `/api/draft-sessions/{id}/purge` | **Hard-delete** a draft session row (irreversible) | `ADMIN_USER` only | — | `204 No Content` |

### Admin Endpoints (added 2026-07-02)

Two **ADMIN_USER-only** management endpoints were added on top of the existing flow. Both are additive — the existing `GET /api/draft-sessions` (heavy `DraftSessionDTO`) and soft-cancel `DELETE /{id}` are unchanged.

#### `GET /api/draft-sessions/summary`

Returns a lightweight overview of **every** draft session (all statuses: `OPEN`, `COMPLETED`, `CANCELLED`, `CONVERTED`), sorted newest-first (`createdAt` DESC). Unlike `GET /api/draft-sessions` it does **not** carry the heavy `teamA`/`teamB`/`remaining` player arrays — it only exposes counts and the associated match plan name plus session status, making it suitable for an admin management table.

- **Auth:** `@PreAuthorize("hasRole('ADMIN_USER')")`
- **Response:** `200 OK` + `List<DraftSessionSummaryDTO>`
- **Errors:** `401` (not authenticated), `403` (not `ADMIN_USER`)

#### `DELETE /api/draft-sessions/{id}/purge` — hard-delete vs. soft-cancel

There are now **two distinct deletion semantics** for a draft session:

| Action | Endpoint | Auth | Effect | Reversible? |
|--------|----------|------|--------|-------------|
| **Soft-cancel** | `DELETE /api/draft-sessions/{id}` | `ADMIN_USER` or `MASTER_USER` | Sets `status = CANCELLED`; row remains in the DB | Row preserved |
| **Hard-purge** | `DELETE /api/draft-sessions/{id}/purge` | `ADMIN_USER` only | Permanently removes the row and its `@ElementCollection` child rows | **No — irreversible** |

`purge` is intended for admin cleanup of stale/cancelled sessions. It may be applied to `OPEN`, `COMPLETED`, or `CANCELLED` sessions, but a `CONVERTED` session **cannot** be purged — it is linked to a created `Match`, so the request is rejected with `409 Conflict`.

- **Auth:** `@PreAuthorize("hasRole('ADMIN_USER')")`
- **Success:** `204 No Content`
- **Errors:** `401` (not authenticated), `403` (not `ADMIN_USER`), `404` (no session with that `id`), `409` (session status is `CONVERTED`)

---

## Real-Time Events & Resume-on-Reconnect (SSE)

`GET /api/draft-sessions/{id}/events` streams live draft updates via Server-Sent Events. On every (re)connect it first emits an authoritative `CONNECTED` snapshot (a full `DraftSessionDTO`), which doubles as the **resume/rehydrate primitive** — all session state is persisted, so a dropped stream (5-minute timeout, tab reload, network blip) is recovered simply by reconnecting to the same endpoint. There is **no dedicated resume endpoint**.

Reconnection is robust for every status:

| Status at (re)connect | Event sequence | Stream after |
|-----------------------|----------------|--------------|
| *(missing session)* | — (`404` fast-fail before any stream) | never opens |
| `OPEN` | `CONNECTED` → *(live `PICK` / `COMPLETED` / `CANCELLED` / `CONVERTED`)* | stays open |
| `COMPLETED` | `CONNECTED` → *(later `CANCELLED` / `CONVERTED`)* | stays open |
| `CANCELLED` | `CONNECTED` → `CANCELLED` → *complete* | closes immediately |
| `CONVERTED` | `CONNECTED` → `CONVERTED` → *complete* | closes immediately |

> 🆕 **Resume enhancement (2026-07-02, non-breaking):** Reconnecting to a session that reached a **terminal** state (`CANCELLED` / `CONVERTED`) while the client was disconnected now sends `CONNECTED` → the matching terminal event → and **closes the stream immediately**. Previously such a reconnect hung until the 5-minute timeout with no close signal. Clients gate auto-reconnect on `status ∈ {OPEN, COMPLETED}` and rely on the terminal event to close. See the [SSE Integration Guide](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) and the [Resume API Contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/DRAFT-SESSION-RESUME-API-CONTRACT.md) for the full frontend flow.

---

## DTOs

### DraftSessionCreateDTO (request)

```json
{
  "matchPlanId": 5,
  "captainAId": 42,
  "captainBId": 17
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `matchPlanId` | `Long` | **Yes** | Must reference a `CONFIRMED` match plan |
| `captainAId` | `Long` | No | Optional — auto-selects highest-rated confirmed player if absent |
| `captainBId` | `Long` | No | Optional — auto-selects second-highest-rated if absent; must differ from `captainAId` |

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
    { "playerId": 7, "playerName": "Carlos Matos", "skillRating": 8.5 },
    { "playerId": 3, "playerName": "Rui Ferreira", "skillRating": 7.9 }
  ],
  "totalPlayers": 14,
  "picksRemaining": 12,
  "expiresAt": null,
  "createdAt": "2026-05-20T10:00:00Z",
  "updatedAt": "2026-05-20T10:05:00Z"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | `Long` | Auto-generated session ID |
| `matchPlanId` | `Long` | Source match plan |
| `matchPlanTitle` | `String` | Title from the match plan |
| `status` | `String` | `"OPEN"` \| `"COMPLETED"` \| `"CANCELLED"` \| `"CONVERTED"` |
| `captainA` | `DraftPlayerDTO` | Fixed Team A captain |
| `captainB` | `DraftPlayerDTO` | Fixed Team B captain |
| `currentTurn` | `String` | `"A"` or `"B"` when `OPEN`; `null` in all other statuses |
| `teamA` | `List<DraftPlayerDTO>` | Players picked for Team A so far (captain always first) |
| `teamB` | `List<DraftPlayerDTO>` | Players picked for Team B so far (captain always first) |
| `remaining` | `List<DraftPlayerDTO>` | Available pool, ordered by `skillRating` DESC |
| `totalPlayers` | `int` | Total confirmed players (both teams + remaining) |
| `picksRemaining` | `int` | `remaining.size()` — how many picks left |
| `expiresAt` | `Instant` | Optional expiry timestamp; `null` if no expiry set |
| `createdAt` | `Instant` | Session creation timestamp |
| `updatedAt` | `Instant` | Last modification timestamp |

### DraftPlayerDTO (embedded)

```json
{ "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 }
```

| Field | Type | Notes |
|-------|------|-------|
| `playerId` | `Long` | Player ID |
| `playerName` | `String` | Display name |
| `skillRating` | `double` | Current skill rating at time of response |

### DraftPickDTO (request)

```json
{ "playerId": 7 }
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `playerId` | `Long` | **Yes** | Must be in the `remaining` pool |

### DraftSessionSummaryDTO (response — admin summary)

Lightweight projection returned by `GET /api/draft-sessions/summary`. A Java `record` in `pt.rics.demo.football.dto` — no `DraftPlayerDTO` lists, only counts and identifiers.

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

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `id` | `Long` | No | Draft session ID |
| `matchPlanId` | `Long` | No | Source match plan ID |
| `matchPlanTitle` | `String` | No | Associated match plan name |
| `status` | `String` | No | `"OPEN"` \| `"COMPLETED"` \| `"CANCELLED"` \| `"CONVERTED"` |
| `captainAName` | `String` | No | Team A captain display name |
| `captainBName` | `String` | No | Team B captain display name |
| `currentTurn` | `String` | **Yes** | `"A"` or `"B"` only when `status = OPEN`; `null` otherwise |
| `totalPlayers` | `int` | No | teamA + teamB + remaining sizes |
| `picksRemaining` | `int` | No | Remaining pool size |
| `createdBy` | `String` | **Yes** | Creator username; may be `null` |
| `expiresAt` | `Instant` | **Yes** | Optional expiry; `null` if no expiry set |
| `createdAt` | `Instant` | No | Session creation timestamp |
| `updatedAt` | `Instant` | No | Last modification timestamp |

---

## Business Rules

The following 11 rules are enforced by `DraftSessionService`:

1. **CONFIRMED plan required** — `matchPlanId` must point to a `MatchPlan` with `status=CONFIRMED`. Returns `400` otherwise.
2. **No duplicate open sessions** — Only one `OPEN` draft session may exist per match plan at a time. Returns `409` if one already exists.
3. **Exact player count** — The confirmed player count must exactly match the format requirement: `FIVE_A_SIDE=10`, `SEVEN_A_SIDE=14`, `ELEVEN_A_SIDE=22`. Returns `400` otherwise.
4. **Captain A auto-select** — If `captainAId` is absent, the player with the highest `skillRating` in the confirmed pool is assigned.
5. **Captain B auto-select** — If `captainBId` is absent, the player with the highest `skillRating` excluding Captain A is assigned.
6. **Captains must be distinct** — `captainAId == captainBId` returns `400`.
7. **Captains must be in the pool** — Explicitly provided captain IDs that are not confirmed for the match plan return `400`.
8. **Remaining pool seeded skill-descending** — After captains are extracted, remaining players are sorted by `skillRating` DESC so frontend can show a ranked pick list.
9. **Pick must be in remaining** — Submitting a `playerId` not in the `remaining` pool returns `400`.
10. **Session must be OPEN to pick** — Picks on `COMPLETED`, `CANCELLED`, or `CONVERTED` sessions return `409`.
11. **Expiry auto-cancel on pick** — If `expiresAt` is set and `Instant.now()` is past expiry, the session is automatically cancelled and `409` is returned.
12. **Last pick completes session** — When `remaining` becomes empty after a pick, `status` transitions to `COMPLETED` automatically.
13. **Confirm requires COMPLETED** — `POST /{id}/confirm` on a non-`COMPLETED` session returns `409`.
14. **Cancel terminal-state guard** — `DELETE /{id}` on a `CONVERTED` or already `CANCELLED` session returns `409`.

---

## Turn Order

Captain A always picks first. After each pick, the turn alternates.

```
Session start (SEVEN_A_SIDE — 12 picks needed after captains seeded):

 Turn:    A  B  A  B  A  B  A  B  A  B  A  B
 Pick #:  1  2  3  4  5  6  7  8  9  10 11 12

Team A after all picks: captainA + picks 1,3,5,7,9,11  → 7 players ✅
Team B after all picks: captainB + picks 2,4,6,8,10,12 → 7 players ✅
```

> **Note:** This is a simple alternating pattern (`A → B → A → B…`), not the snake-draft double-pick pattern used by `CaptainPickGenerationStrategy` (server-side). The interactive draft gives each captain exactly one pick per turn.

---

## Session Creation Flow

```
1. Load MatchPlan (must be CONFIRMED)
2. Guard: no existing OPEN session for this plan
3. Load all CONFIRMED player IDs from player_confirmations
4. Validate player count == required for matchType
5. Resolve Captain A (explicit captainAId or highest skillRating)
6. Resolve Captain B (explicit captainBId or second-highest, must differ from A)
7. Build remaining pool = all confirmed players minus captainA and captainB
   → sorted by skillRating DESC
8. Persist DraftSession:
   - teamA = [captainA.id]
   - teamB = [captainB.id]
   - remaining = [sorted rest...]
   - currentTurn = "A"
   - status = OPEN
9. Return DraftSessionDTO
```

---

## Confirm Flow (COMPLETED → CONVERTED + Match created)

```
1. Load DraftSession (must be COMPLETED)
2. Load all Player entities for both team lists
3. Load MatchPlan (for description, proposedDate, location, matchType)
4. Resolve current active Season (throws 400 if none active)
5. Build Match entity:
   - description  = plan.title
   - matchDate    = plan.proposedDate at 00:00 UTC
   - location     = plan.location
   - matchType    = plan.matchType
   - generationType = CAPTAIN_PICK
   - generationNotes = "CAPTAIN_PICK (interactive): captainA=<name> captainB=<name>"
6. Create 2 MatchTeam rows (Team A, Team B)
7. For any player with isActive=false → reactivate (set isActive=true)
8. Create PlayerStat rows for every player in both teams
9. Set DraftSession.status = CONVERTED
10. Return MatchDTO
```

---

## Error Cases

| Scenario | HTTP Status | Message |
|----------|-------------|---------|
| Match plan not found | `404 Not Found` | `MatchPlan with id X not found` |
| Match plan not CONFIRMED | `400 Bad Request` | `Match plan must be in CONFIRMED status…` |
| OPEN session already exists | `409 Conflict` | `A draft session is already open for this match plan` |
| Wrong confirmed player count | `400 Bad Request` | `Confirmed player count must be exactly N for TYPE, got M` |
| `captainAId` not in pool | `400 Bad Request` | `CaptainA id=X is not in the confirmed player pool` |
| `captainBId` not in pool | `400 Bad Request` | `CaptainB id=X is not in the confirmed player pool` |
| CaptainA == CaptainB | `400 Bad Request` | `CaptainA and CaptainB cannot be the same player` |
| Pick on non-OPEN session | `409 Conflict` | `Draft session is not open` |
| Session expired | `409 Conflict` | `Draft session has expired` |
| Picked player not in remaining | `400 Bad Request` | `Player id=X is not available in this draft` |
| Confirm on non-COMPLETED session | `409 Conflict` | `Draft session must be in COMPLETED status before confirming…` |
| Cancel on CONVERTED/CANCELLED session | `409 Conflict` | `Cannot cancel draft session with status=X` |
| No active season on confirm | `400 Bad Request` | `No active season; cannot create match` |
| Draft session not found | `404 Not Found` | `DraftSession with id X not found` |

---

## Example Workflow (SEVEN_A_SIDE, step-by-step)

### Step 1 — Create the Draft Session

**Request:**
```http
POST /api/draft-sessions HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{
  "matchPlanId": 5
}
```

**Response (201 Created):**
```json
{
  "id": 1,
  "matchPlanId": 5,
  "matchPlanTitle": "Friday Night Match",
  "status": "OPEN",
  "captainA": { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 },
  "captainB": { "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 },
  "currentTurn": "A",
  "teamA": [{ "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 }],
  "teamB": [{ "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 }],
  "remaining": [
    { "playerId": 7, "playerName": "Carlos Matos", "skillRating": 8.5 },
    { "playerId": 3, "playerName": "Rui Ferreira", "skillRating": 7.9 },
    { "playerId": 9, "playerName": "André Lima", "skillRating": 7.6 },
    { "playerId": 11, "playerName": "Pedro Costa", "skillRating": 7.3 },
    { "playerId": 5, "playerName": "Tiago Nunes", "skillRating": 6.8 },
    { "playerId": 13, "playerName": "Bruno Lopes", "skillRating": 6.5 },
    { "playerId": 8, "playerName": "Fábio Gomes", "skillRating": 6.2 },
    { "playerId": 2, "playerName": "Luís Pinto", "skillRating": 5.9 },
    { "playerId": 6, "playerName": "Marco Vieira", "skillRating": 5.6 },
    { "playerId": 10, "playerName": "Nuno Alves", "skillRating": 5.3 },
    { "playerId": 4, "playerName": "David Rocha", "skillRating": 4.8 },
    { "playerId": 14, "playerName": "Sérgio Campos", "skillRating": 4.5 }
  ],
  "totalPlayers": 14,
  "picksRemaining": 12,
  "expiresAt": null,
  "createdAt": "2026-05-20T10:00:00Z",
  "updatedAt": "2026-05-20T10:00:00Z"
}
```

### Step 2 — Captain A picks (turn = A)

**Request:**
```http
POST /api/draft-sessions/1/pick HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{ "playerId": 7 }
```

**Response (200 OK):**
```json
{
  "id": 1,
  "status": "OPEN",
  "currentTurn": "B",
  "teamA": [
    { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 },
    { "playerId": 7, "playerName": "Carlos Matos", "skillRating": 8.5 }
  ],
  "teamB": [{ "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 }],
  "remaining": [ ... 11 players ... ],
  "picksRemaining": 11,
  ...
}
```

### Step 3–13 — Continue alternating picks

*Repeat `POST /api/draft-sessions/1/pick` with the desired `playerId`, alternating between captains A and B.*

### Step 14 — Final pick (last remaining player)

After the 12th pick, the response will show:

```json
{
  "status": "COMPLETED",
  "currentTurn": null,
  "remaining": [],
  "picksRemaining": 0
}
```

### Step 15 — Confirm draft and create the Match

**Request:**
```http
POST /api/draft-sessions/1/confirm HTTP/1.1
Authorization: Bearer <token>
```

**Response (201 Created):** Full `MatchDTO` with `generationType=CAPTAIN_PICK`.

---

## Relationship to Server-Side CAPTAIN_PICK Simulation

The system supports **two distinct modes** for captain-pick team generation:

| Mode | How to Invoke | State | Output |
|------|---------------|-------|--------|
| **Server-side simulation** (Phase 3) | `POST /api/match-plans/{id}/generate?generationType=CAPTAIN_PICK` | Stateless | `MatchPreviewDTO` (not persisted until `/confirm`) |
| **Interactive Draft Session** (Phase 4) | `POST /api/draft-sessions` + sequential picks | Stateful (DB) | `DraftSessionDTO` → `MatchDTO` on confirm |

The `CaptainPickGenerationStrategy` class handles the server-side simulation and is still fully available. It uses a snake-draft **double-pick pattern** (A → BB → AA →…), whereas the interactive session uses a simple alternating pattern (A → B → A →…).

Choose the non-interactive flow for quick automated balancing; use the interactive `DraftSession` API for live social drafting events.

---

## Database Schema

`V4__draft_sessions.sql` creates 4 tables:

| Table | Purpose |
|-------|---------|
| `draft_sessions` | Master record — status, captains, current turn, expiry |
| `draft_session_team_a` | Ordered list of picked players for Team A (position = insertion order) |
| `draft_session_team_b` | Ordered list of picked players for Team B |
| `draft_session_remaining` | Players still available, ordered by skill rating DESC at creation |

All child tables cascade delete from `draft_sessions`. Indexes on `match_plan_id` and `status` for efficient lookup.

---

## Known Limitations

- ⚠️ **No background job to auto-expire sessions** — `expiresAt` is checked lazily on the next pick attempt. A session past its `expiresAt` remains `OPEN` in the DB until a pick is submitted against it.
- ⚠️ **No real-time push notifications** — There is no WebSocket/SSE integration. Clients must poll `GET /api/draft-sessions/{id}` to see the latest state.
- ⚠️ **Draft history / audit endpoint not exposed** — There is no endpoint to list all past draft sessions for a match plan. Can be queried directly from DB via `draft_sessions` table.
- ⚠️ **Single OPEN session per plan** — You cannot run multiple simultaneous drafts for the same match plan. Cancel the existing one first.
- ⚠️ **No pick undo** — Once a player is picked, the pick cannot be undone without cancelling the whole session.

