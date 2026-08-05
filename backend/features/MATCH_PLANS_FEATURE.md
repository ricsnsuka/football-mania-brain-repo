# Match Plans & Availability Poll Feature

**Added in:** v1.0.0  
**Date:** May 17, 2026 · **rewritten against the code on 2026-08-05**  
**Status:** ✅ Released — weekly recurring runs added in `1.3.0`

> **Why the rewrite.** This document was last touched on 2026-05-27 and had been carrying a drift
> entry ever since, because it was *wrong* rather than merely old: `proposed_date` was documented as
> a `DATE` two months after V17 made it an instant, `GENERATED` was missing from the lifecycle it
> terminates, the grants were written in role names V33 renamed, and neither the waitlist, the pitch
> cost, the past/upcoming split nor guests appeared at all. Everything below is checked against the
> entities, the migrations and `MatchPlanController` at that date.

---

## Overview

A **Match Plan** is a pre-match organisation tool. Before a match can be created, a `MANAGER`:

1. Creates a **Match Plan** with a kickoff instant, location, match type, and confirmation deadline.
2. The plan opens an **availability poll** — players confirm or decline participation, and anyone
   with a linked player may bring a guest to fill a spot that is still empty.
3. Once enough players have confirmed, they **generate a team preview** using one of the
   generation strategies (BALANCED, RANDOM, SNAKE_DRAFT, …).
4. The admin reviews the preview and **confirms** it — which creates the actual `Match`.

This decouples "planning who will play" from "running the match stats".

A group that plays the same fixture every week can create the whole run in one request — see
[Weekly recurring runs](#weekly-recurring-runs). The plans it writes are ordinary plans; a run is a
faster way to reach the same rows, not a new kind of thing.

---

## Domain Model

### `match_plans` Table

| Column                  | Type           | Nullable | Notes                                                                 |
|-------------------------|----------------|----------|-----------------------------------------------------------------------|
| `id`                    | BIGSERIAL (PK) | No       |                                                                       |
| `tenant_id`             | BIGINT (FK)    | No       | The owning group (V24). Every read is scoped to it                    |
| `title`                 | VARCHAR(100)   | No       | Default `'Unnamed Plan'`                                              |
| `proposed_date`         | TIMESTAMPTZ    | No       | **Kickoff instant, not a date** — V17. Must be in the future at creation (@Future) |
| `location`              | VARCHAR(255)   | Yes      |                                                                       |
| `description`           | VARCHAR(500)   | Yes      |                                                                       |
| `match_type`            | VARCHAR(20)    | No       | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`                    |
| `status`                | VARCHAR(20)    | No       | `PENDING` / `CONFIRMED` / `CANCELLED` / `GENERATED` (V17)             |
| `confirmed_count`       | INTEGER        | No       | Denormalised count of `CONFIRMED` player confirmations                |
| `min_players_required`  | INTEGER        | No       | Default `14` for SEVEN_A_SIDE                                         |
| `confirmation_deadline` | TIMESTAMPTZ    | Yes      | After this time, `pollOpen` becomes `false`                           |
| `total_cost_cents`      | INTEGER        | Yes      | What the pitch cost (V19). **Absent ≠ zero** — nobody recorded it, versus the match was free |
| `deadline_reminder_sent_at` | TIMESTAMPTZ | Yes     | Conditional-update guard (V13) so the reminder scheduler is safe across instances and restarts |
| `match_reminder_sent_at`| TIMESTAMPTZ    | Yes      | The same guard for the pre-match reminder                             |
| `created_by`            | VARCHAR(50)    | Yes      | Username of the manager who created the plan                          |
| `created_at`            | TIMESTAMPTZ    | No       |                                                                       |
| `updated_at`            | TIMESTAMPTZ    | No       |                                                                       |

**There is no recurrence column, deliberately.** A weekly run writes independent rows and keeps no
series — see [Weekly recurring runs](#weekly-recurring-runs).

### `player_confirmations` Table

| Column          | Type           | Nullable | Notes                                                |
|-----------------|----------------|----------|------------------------------------------------------|
| `id`            | BIGSERIAL (PK) | No       |                                                      |
| `match_plan_id` | BIGINT (FK)    | No       | FK → `match_plans.id` (CASCADE DELETE)               |
| `player_id`     | BIGINT (FK)    | No       | FK → `players.id` (CASCADE DELETE)                   |
| `status`        | VARCHAR(20)    | No       | `CONFIRMED` / `DECLINED` / `PENDING`                 |
| `notes`         | VARCHAR(500)   | Yes      | Optional note from player (e.g. "5 minutes late")    |
| `confirmed_at`  | TIMESTAMPTZ    | Yes      | Timestamp when player confirmed                      |

Plus `tenant_id` (V24), as everywhere else.

Unique constraint: `(match_plan_id, player_id)` — one confirmation record per player per plan.

**The waitlist is not stored.** `confirmationRank`, `isStarter` and `waitlistPosition` are derived
on read from confirmation order against `min_players_required`: rank ≤ required is a starter, and
everyone past it gets `rank - required` as their place in the queue. Storing them would mean
rewriting every later row each time somebody drops out.

**Guests are `players`, not confirmations.** `is_guest` and `invited_by_player_id` are columns on
`players` (V21), so a guest holds an ordinary confirmation and the ranking queries exclude them with
one predicate.

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
- `CONFIRMED` → `GENERATED` ✅ — written by `POST /generate/confirm`, not by a client
- Any other transition → `400 Bad Request`

**`GENERATED` is terminal** (V17). Teams were drawn and a `Match` exists, so nothing transitions out
of it — that is what stops one plan producing two matches.

**There is no `EXPIRED` status**, and adding one would be a mistake: a plan goes out of date because
the clock moved, not because anything happened to it, so no write exists to make. The API derives
`expired` per request instead, alongside `generatable` and `cancellable`. **Clients must branch on
those flags rather than re-deriving them**, or the UI drifts from the server's rule.

---

## Weekly recurring runs

`POST /api/match-plans/recurring` (`MANAGER`) writes a whole weekly run in one request. It shipped
in `1.3.0`; the frontend reaches it from the same create form, as a **Repeat weekly** checkbox plus a
count. The [API contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/RECURRING-MATCH-PLANS-API-CONTRACT.md)
is authoritative on the request shape. What is worth recording here is why it looks like this.

**The plans are independent — there is no series.** Every occurrence is an ordinary plan the moment
it is written: its own poll, its own lifecycle, edited, cancelled, generated from or deleted on its
own. A series id was considered and rejected — it buys "cancel them all" and costs a second
lifecycle to reason about, including what cancelling a run means for the plans in it that already
have people confirmed or a match generated. Every existing rule therefore applies unchanged, which
is why this feature added no column.

**Weekly only.** Each occurrence is exactly seven days after the one before, so a run holds the same
weekday at the same time — the "Friday 5-a-side" a group actually plays. Fortnightly and monthly were
left out rather than guessed at: monthly needs a rule for what "the 31st" means in a 30-day month,
and *a rule nobody asked for is a rule nobody checked*.

**A count, not an end date.** `occurrences` is 2–120 inclusive of the first, because "the next eight
Fridays" is the thing a manager knows. One occurrence is not a recurrence — `POST /api/match-plans`
already does that — and the 120 ceiling is structural, stopping arithmetic on an absurd count before
the horizon check runs. It is not the real limit.

**The deadline is a lead time, not an instant.** Each later plan's `confirmationDeadline` moves by
the same seven days its kickoff does. A run of thirteen plans all sharing one deadline in week one
would close twelve polls before anybody could answer them.

**The horizon is a platform setting, not a group one.** `PlatformSetting.MATCH_PLAN_RECURRENCE_MAX_MONTHS`
(default 3, range 1–24) bounds how far ahead a run may reach, and it sits behind the operator grant
in `platform_settings` (V34) rather than in a group's own `app_settings`. One call writes every row
in the run, so the horizon bounds what a single request can do to the deployment — a group able to
raise its own ceiling would not be bounded at all. The horizon is measured **from now**, not from the
first kickoff; otherwise a run starting in eleven months could reach fourteen.

**Refused whole, never truncated.** A run that overruns the horizon is rejected with `400`, and the
message names how many occurrences *would* have fitted — counted, not estimated, so the number
offered back is one that will be accepted. Silently writing the nine that fit out of twelve would
answer `201`, look like success, and leave the manager to discover the gap weeks later.

**Known limitation: seven days is seven days.** Occurrences step by seven days of *absolute* time, so
across a daylight-saving boundary the local kickoff moves by an hour — a 19:00 fixture becomes 18:00.
Adding weeks in the group's own timezone is the correct fix and the schema stores no timezone
anywhere: kickoffs are instants, and the frontend renders them in whatever zone the browser is in.
Hard-coding one would be silently wrong for every group not in it, which is worse than an hour.
Recorded rather than papered over — and the frontend's "last kickoff" preview steps the same way, so
it shows the date the backend will actually write.

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

| Method  | Path                                            | Auth                | Description                                                    |
|---------|-------------------------------------------------|---------------------|----------------------------------------------------------------|
| `POST`  | `/api/match-plans`                              | `MANAGER`           | Create plan and open poll                                      |
| `POST`  | `/api/match-plans/recurring`                    | `MANAGER`           | Create a weekly run in one request — `201` with the plans in kickoff order |
| `GET`   | `/api/match-plans`                              | Any authenticated   | List plans (paginated; optional `status`, `timeframe` and `from`/`to` filters) |
| `GET`   | `/api/match-plans/{id}`                         | Any authenticated   | Get plan by ID                                                 |
| `PATCH` | `/api/match-plans/{id}`                         | `MANAGER`           | Update plan details (PENDING only)                             |
| `PATCH` | `/api/match-plans/{id}/status`                  | `MANAGER`           | Transition status                                              |
| `DELETE`| `/api/match-plans/{id}`                         | `GROUP_ADMIN`       | Delete a PENDING plan                                          |
| `GET`   | `/api/match-plans/{id}/confirmations`           | Any authenticated   | List all confirmations (optional `status` filter)              |
| `POST`  | `/api/match-plans/{id}/confirmations/me`        | Any authenticated   | Self-confirm or decline availability                           |
| `GET`   | `/api/match-plans/{id}/confirmations/me`        | Any authenticated   | Get your own confirmation entry                                |
| `POST`  | `/api/match-plans/{id}/guests`                  | Any authenticated   | Bring an outsider to fill a spot — the gate is state, not role  |
| `DELETE`| `/api/match-plans/{id}/guests/{playerId}`       | Any authenticated   | Take a guest off — the inviter while the poll is open, or a `MANAGER` |
| `PATCH` | `/api/match-plans/{id}/confirmations/{playerId}`| `MANAGER`           | Override any player's confirmation status                      |
| `POST`  | `/api/match-plans/{id}/generate`                | `MANAGER`           | Preview generated teams (stateless, not persisted)             |
| `POST`  | `/api/match-plans/{id}/generate/confirm`        | `MANAGER`           | Confirm preview and create the match                           |

### `timeframe`, and the ordering that goes with it

`timeframe` is `upcoming` (kickoff still ahead), `past` (kickoff behind), or omitted for both. It is
a **server-side** filter because the list is paginated — dropping rows after the page is built shows
however many survived while `totalElements` keeps counting the ones that did not.

Ordering runs **away from now** unless the client sends its own `sort`, which still wins:

| `timeframe` | Default order |
|---|---|
| `upcoming` | `proposedDate ASC, id ASC` — the next match is first |
| `past` | `proposedDate DESC, id DESC` — the one just played is first |
| omitted | ascending, as the plainer reading of "chronological" |

Opposite directions, one rule: whatever happens or happened nearest is what somebody is looking for.
The `id` tie-break matters — without it, rows sharing a `proposedDate` reintroduce the pagination
instability the ordering exists to fix. **Until `1.3.0` the query had no `ORDER BY` at all**, so
Postgres was free to return any order it liked.

### `from` / `to` — an explicit kickoff window

Two optional ISO-8601 instants bounding `proposedDate`: `from` **inclusive**, `to` **exclusive**.
They filter in the query for the same reason `timeframe` does — the list is paginated.

**Half-open on purpose.** Consecutive windows tile exactly, so a plan kicking off at the boundary
belongs to the window starting there and to no other. An inclusive upper bound has to name a last
representable instant, and whatever it names, a kickoff in the gap after it belongs to no window at
all.

**There is deliberately no `timeframe=week`.** A week runs Monday to Sunday *in the reader's
timezone*, and the server has no way to know whose Monday is meant — resolving it here would give
every caller the server's own zone. The caller computes the window and sends two instants; the
frontend does exactly this for its "By week" tab.

**The window intersects with `timeframe` rather than replacing it**, so `upcoming` plus this week
means the rest of this week. The service takes the later of the two lower bounds and the earlier of
the two upper bounds. An empty intersection — last week's window with `timeframe=upcoming` — is an
honest empty page.

| Condition | Result |
|---|---|
| `from` on or after `to` | `400 from must be before to` |
| Unparseable instant | `400 Parameter 'from' has an invalid value` (Spring type conversion) |
| Either omitted | That end is unbounded |

A zero-length window is refused rather than silently matching nothing: `>= from AND < to` with
`from == to` can never match, so it is a caller mistake rather than a filter that legitimately found
none.

Ordering is unchanged — a window with no `timeframe` takes the ascending default, which is what
reading a week Monday-first wants.

### Authorization Matrix

Role names are post-V33: `GROUP_ADMIN` administers **one group** (it was called `ADMIN` and the name
was the root cause of a cross-group bug), `MANAGER` runs matches, and every grant is held per
membership.

| Action                           | Member | `MANAGER` | `GROUP_ADMIN` |
|----------------------------------|:---:|:---:|:---:|
| Read plans / confirmations       | ✅ | ✅ | ✅ |
| Self-confirm availability        | ✅ | ✅ | ✅ |
| Bring / remove a guest           | ✅ | ✅ | ✅ |
| Create plan                      | ❌ | ✅ | ✅ |
| Create a weekly run              | ❌ | ✅ | ✅ |
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
| `proposedDate`         | Instant | No       | Kickoff, full instant: `"2026-05-23T19:00:00Z"` (date-only until V17) |
| `confirmationDeadline` | Instant | Yes      |                                                                    |
| `status`               | String  | No       | `PENDING` / `CONFIRMED` / `CANCELLED` / `GENERATED`               |
| `totalCostCents`       | Integer | Yes      | Pitch cost. **Omitted when unrecorded**, which is not zero          |
| `confirmedCount`       | int     | No       | Number of players who CONFIRMED                                    |
| `minPlayersRequired`   | int     | No       | Minimum needed to generate teams                                   |
| `description`          | String  | Yes      |                                                                    |
| `createdBy`            | String  | Yes      | Username of creator                                                |
| `playersNeeded`        | int     | No       | `max(0, minPlayersRequired - confirmedCount)` — computed           |
| `pollOpen`             | boolean | No       | `true` when `status == PENDING` and deadline has not passed        |
| `expired`              | boolean | No       | Kickoff has passed. Derived per read, never stored                 |
| `generatable`          | boolean | No       | `CONFIRMED` and not expired. **Do not re-derive** — a `GENERATED` plan is `false` here, which is what stops one plan producing two matches |
| `cancellable`          | boolean | No       | Not generated from, and has not already happened                   |
| `createdAt`            | Instant | No       |                                                                    |
| `updatedAt`            | Instant | No       |                                                                    |

Nullable fields are **omitted** rather than sent as `null` — the API sets
`spring.jackson.default-property-inclusion: non_null`.

### `MatchPlanCreateDTO` (POST request)

| Field                  | Type    | Required | Validation                                              |
|------------------------|---------|----------|---------------------------------------------------------|
| `title`                | String  | Yes      | `@NotBlank`                                             |
| `matchType`            | String  | Yes      | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`       |
| `location`             | String  | No       |                                                         |
| `proposedDate`         | Instant | Yes      | `@Future` — the kickoff instant, not a date             |
| `confirmationDeadline` | Instant | No       | Must be before `proposedDate`                           |
| `description`          | String  | No       | `@Size(max=500)`                                        |
| `minPlayersRequired`   | Integer | No       | Defaults to the minimum for the match type (10 / 14 / 22) |

### `RecurringMatchPlanCreateDTO` (POST /recurring request)

Every field of `MatchPlanCreateDTO`, meaning exactly what it means there and applied to each plan in
the run, plus one:

| Field         | Type    | Required | Validation                                                        |
|---------------|---------|----------|-------------------------------------------------------------------|
| `occurrences` | int     | Yes      | `2`–`120`, **inclusive of the first**; further bounded by the horizon |

`proposedDate` is the **first** kickoff. Validating it is enough for the whole run: every later
kickoff is strictly further into the future, so a valid first one makes the rest valid on those
checks. The response is a plain array in kickoff order, with **no `Location` header** — the run has
no single resource to point at, and naming the first plan would imply the others hang off it.

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

10. **`generate/confirm` creates the match** — it moves the plan to `GENERATED`, which is terminal.
    That is the guard against one plan producing two matches; branch on `generatable` rather than
    on status.

11. **A weekly run is refused whole or written whole** — the horizon check runs before anything is
    written, so a rejected run leaves no partial rows behind.

12. **A run's plans are independent from the moment they exist** — no series id, no cascade, no
    "cancel them all". Whatever is true of a plan created one at a time is true of every plan in a
    run.

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
  "proposedDate": "2026-05-30T19:00:00Z",
  "confirmationDeadline": "2026-05-29T20:00:00Z",
  "description": "Bring bibs",
  "minPlayersRequired": 14
}
```

> Sending a date-only `"2026-05-30"` used to be accepted, and the API took the date and discarded
> the rest — so every plan kicked off at midnight. Send the instant.

**Response `201 Created`:**
```json
{
  "id": 5,
  "title": "Friday Night — Week 21",
  "matchType": "SEVEN_A_SIDE",
  "location": "Central Park Pitch 2",
  "proposedDate": "2026-05-30T19:00:00Z",
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

### Create Eight Weeks of It

```http
POST /api/match-plans/recurring HTTP/1.1
Content-Type: application/json
Authorization: Bearer <manager-token>

{
  "title": "Friday 5-a-side",
  "matchType": "FIVE_A_SIDE",
  "location": "Pavilhão Municipal",
  "proposedDate": "2026-08-07T19:00:00Z",
  "confirmationDeadline": "2026-08-06T19:00:00Z",
  "occurrences": 8
}
```

**Response `201 Created`:** an array of eight `MatchPlanDTO`s in kickoff order — 7 Aug, 14 Aug, …,
25 Sep — each with its deadline a day before its own kickoff.

**Response `400 Bad Request`** when the run overruns the horizon:

```json
{
  "status": 400,
  "message": "A weekly run of 20 would reach 2026-12-18T19:00:00Z, beyond the 3-month recurrence horizon. At most 12 occurrence(s) fit from this start date."
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

Match plans have their own Caffeine cache, `matchPlans` (`CacheConfig.MATCH_PLANS`) — not the
`matches` cache this document used to name:

| Cache Name  | Populated By                | Evicted When                                |
|-------------|-----------------------------|---------------------------------------------|
| `matchPlans`| `GET /api/match-plans/{id}` | Any write to a plan, its confirmations, its guests or its cost — `allEntries = true` |

Keys carry the tenant (`TenantContext.currentTenant() + ':plan-' + id`), so one group's cached plan
can never be served to another. Creating a **run** evicts the whole cache for the same reason every
other write does: a run lands anywhere in the list, and no single-key eviction describes that.

---

## Implementation Details

- **Entity:** `MatchPlan.java`, `PlayerConfirmation.java` — JPA entities with Lombok
- **Service:** `MatchPlanService.java` — business logic, confirmation management, team generation dispatch
- **Controller:** `MatchPlanController.java` — REST layer; `@AuthenticationPrincipal UserPrincipal` used for self-service confirmation
- **Team generation:** Delegated to `TeamGenerationStrategyFactory` → `BalancedGenerationStrategy` / `RandomGenerationStrategy` / `SnakeDraftGenerationStrategy`
- **DTOs:** `MatchPlanCreateDTO`, `RecurringMatchPlanCreateDTO`, `MatchPlanDTO`, `MatchPlanUpdateDTO`, `MatchPlanStatusDTO`, `ConfirmationUpsertDTO`, `PlayerConfirmationDTO`, `GuestCreateDTO`, `MatchPreviewDTO` — all Java Records
- **Recurrence horizon:** `PlatformSettingsService` reading `PlatformSetting.MATCH_PLAN_RECURRENCE_MAX_MONTHS` — a separate bean from `AppSettingsService` on purpose, because the two answer to different grants
- **Frontend:** the create modal's **Repeat weekly** checkbox — see [FE match plans](../../frontend/features/match-plans.md)

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
| Deadline not before kickoff                    | 400    | `confirmationDeadline must be before proposedDate`       |
| `minPlayersRequired` below the type's minimum  | 400    | `minPlayersRequired (N) cannot be less than the minimum required for <type> (M players)` |
| First kickoff already past the horizon         | 400    | `The first kickoff is beyond the N-month recurrence horizon; no plan in this run could be created` |
| Weekly run overruns the horizon                | 400    | `A weekly run of N would reach <instant>, beyond the M-month recurrence horizon. At most K occurrence(s) fit from this start date.` |
| Creating a run without `MANAGER`               | 403    | —                                                        |

