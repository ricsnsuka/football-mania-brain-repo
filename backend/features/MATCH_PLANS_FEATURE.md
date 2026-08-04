# Match Plans & the Availability Poll

**Added in:** v1.0.0
**Status:** ✅ Released
**Rewritten against the code:** 2026-08-04 — the previous version described May's behaviour and was
wrong about the kickoff type, the status set, the roles, the team names and the cache. Every claim
below was read out of `MatchPlanService`, `MatchPlanController`, `MatchPlan`, `MatchFeeService` and
the migrations rather than carried over.

⏳ **Recurring runs and the chronological list order are on the recurring-plans branch, unmerged and
undeployed** (`V34` with them). They are documented here in the present tense with the rest; the
deployment state is in [STATUS.md](../../STATUS.md).

---

## Overview

A **match plan** is the thing that exists before a match does. Somebody proposes a game, players say
whether they are coming, and once enough have said yes the teams are drawn and a real `Match` is
created from it.

```
create plan ──▶ players confirm ──▶ enough confirmed ──▶ draw teams ──▶ Match exists
   PENDING          PENDING             CONFIRMED         GENERATED     (plan is done)
```

The split exists because organising who plays and recording what happened are different jobs with
different audiences. A plan is answered by everyone in the group; a match is written by whoever kept
the score.

Four things have been added since the first version and each changed the shape of the feature: a
plan now knows **what time** it kicks off, it knows **who is a reserve** rather than only who
confirmed, it can carry **what the pitch cost**, and a manager can create **a weekly run** of them in
one request.

---

## The plan itself

### `match_plans`

| Column                      | Type           | Null | Notes                                                              |
|-----------------------------|----------------|------|--------------------------------------------------------------------|
| `id`                        | BIGSERIAL (PK) | No   |                                                                    |
| `tenant_id`                 | BIGINT         | No   | The owning organization. Stamped at persist — `V24`                 |
| `title`                     | VARCHAR(100)   | No   |                                                                    |
| `proposed_date`             | **TIMESTAMPTZ**| No   | Kickoff, **with a time of day** since `V17`                        |
| `location`                  | VARCHAR(255)   | Yes  |                                                                    |
| `description`               | VARCHAR(500)   | Yes  |                                                                    |
| `match_type`                | VARCHAR(20)    | No   | `FIVE_A_SIDE` / `SEVEN_A_SIDE` / `ELEVEN_A_SIDE`                   |
| `status`                    | VARCHAR(20)    | No   | `PENDING` / `CONFIRMED` / `CANCELLED` / **`GENERATED`** — `V17`     |
| `confirmed_count`           | INTEGER        | No   | Denormalised; recomputed by a `COUNT` after every confirmation write|
| `min_players_required`      | INTEGER        | No   | Defaults to the match type's own total                             |
| `confirmation_deadline`     | TIMESTAMPTZ    | Yes  | After it passes, `pollOpen` is false                               |
| `total_cost_cents`          | INTEGER        | Yes  | What the pitch cost — `V19`. `CHECK (>= 0)`                        |
| `deadline_reminder_sent_at` | TIMESTAMPTZ    | Yes  | Reminder idempotency guard — `V13`                                 |
| `match_reminder_sent_at`    | TIMESTAMPTZ    | Yes  | Independent of the above — `V13`                                   |
| `created_by`                | VARCHAR(50)    | Yes  | Username. Scrubbed to a tombstone on erasure                       |
| `created_at` / `updated_at` | TIMESTAMPTZ    | No   |                                                                    |

#### Kickoff is an instant, and that was a bug fix

`proposed_date` was a `DATE`. The frontend's input has always been a `datetime-local` and has always
sent a time; the backend parsed it into a `LocalDate` and threw the time away. A bare date is parsed
by clients as UTC midnight, which renders as **01:00 in Lisbon** — so every plan in the app appeared
to kick off at one in the morning, and every match generated from one inherited it.

`V17` widened the column. **Existing rows converted to midnight UTC**, which is what they already
effectively meant; no plausible-looking 19:00 was invented for them, because a made-up kickoff hour
is indistinguishable from one somebody chose.

### `player_confirmations`

| Column          | Type           | Null | Notes                                          |
|-----------------|----------------|------|------------------------------------------------|
| `id`            | BIGSERIAL (PK) | No   |                                                |
| `tenant_id`     | BIGINT         | No   | Stamped at persist                             |
| `match_plan_id` | BIGINT (FK)    | No   | → `match_plans.id`, CASCADE DELETE             |
| `player_id`     | BIGINT (FK)    | No   | → `players.id`, CASCADE DELETE                 |
| `status`        | VARCHAR(20)    | No   | `CONFIRMED` / `DECLINED` / `PENDING`           |
| `notes`         | VARCHAR(500)   | Yes  | "Will be five minutes late"                    |
| `confirmed_at`  | TIMESTAMPTZ    | Yes  | Set on confirming, **cleared on withdrawing**  |

Unique on `(match_plan_id, player_id)` — one answer per player per plan.

`confirmed_at` doing double duty as the queue position is what makes the waitlist work without any
separate bookkeeping. See [Starters and reserves](#starters-and-reserves).

---

## Lifecycle

```
   POST /api/match-plans
            │
            ▼
      ┌───────────┐   PATCH /status {CONFIRMED}    ┌────────────┐
      │  PENDING  │ ─────────────────────────────▶ │ CONFIRMED  │
      └───────────┘   (needs enough confirmations) └────────────┘
            │                                            │
            │ PATCH /status {CANCELLED}                   │ POST /generate/confirm
            │                                            │
            ▼                    PATCH /status {CANCELLED}▼
      ┌───────────┐ ◀──────────────────────────────┌────────────┐
      │ CANCELLED │                                │ GENERATED  │  ◀── terminal
      └───────────┘                                └────────────┘
```

| From        | To          | Via                       | Conditions                                            |
|-------------|-------------|---------------------------|-------------------------------------------------------|
| `PENDING`   | `CONFIRMED` | `PATCH /{id}/status`      | `confirmedCount` ≥ `minPlayersRequired` **and** ≥ the match type's total |
| `PENDING`   | `CANCELLED` | `PATCH /{id}/status`      | kickoff has not passed                                |
| `CONFIRMED` | `CANCELLED` | `PATCH /{id}/status`      | kickoff has not passed                                |
| `CONFIRMED` | `GENERATED` | `POST /{id}/generate/confirm` | **not** reachable through the status endpoint     |

Everything else is a `409`.

### `GENERATED` is terminal, and that is the point

A plan whose teams had been drawn used to stay `CONFIRMED` forever. It therefore stayed selectable
for generation — one plan could produce two matches — and the "plans you can generate from" list only
ever grew. `GENERATED` records that this plan *became* a match, which is a fact about the past that
editing the plan cannot undo. Nothing transitions out of it, **including cancellation**: cancelling a
generated plan would be claiming that a match already on record was called off.

`V17` did **not** backfill existing plans to `GENERATED`. Nothing records which plan a match came
from — a match stores the players it was played with, not its origin — so a backfill could only guess
by pairing plans with same-day matches, and it would mark the wrong plan whenever two matches shared
a date. A plan wrongly frozen as `GENERATED` cannot be recovered through the UI. It was also
unnecessary: every already-used plan is in the past, and past plans are excluded from generation by
the expiry rule regardless of status.

### There is deliberately no `EXPIRED`

A plan goes out of date because **the clock moved**, not because anything happened to it. Storing
that would need a scheduled job whose entire output is "time passed", and every row the job had not
yet reached would be lying. It is derived on read instead:

| Derived flag  | Rule                                              | Sent on `MatchPlanDTO` as |
|---------------|---------------------------------------------------|---------------------------|
| `isExpired()` | `proposedDate` is before now                      | `expired`                 |
| `isGeneratable()` | status is `CONFIRMED` **and** not expired     | `generatable`             |
| `isCancellable()` | status is `PENDING` or `CONFIRMED` **and** not expired | `cancellable`    |
| `pollOpen`    | status is `PENDING` **and** deadline null or ahead | `pollOpen`               |

All four are serialised **so the UI does not re-derive them and drift**. `generatable` in particular
encodes the "one plan, one match" rule; a client computing it from `status` alone would get it wrong
the moment expiry mattered.

---

## The poll

Any authenticated user with a linked player answers for themselves:

```http
POST /api/match-plans/5/confirmations/me
{ "status": "CONFIRMED", "notes": "Straight from work, five minutes late" }
```

It is an upsert — the row is created on first answer and rewritten afterwards. **The poll must be
open**: status `PENDING`, and the confirmation deadline either unset or still ahead. Both failures
are `409`.

An account with no linked player gets `400 No player linked to your account`. That linked player is
the membership test throughout this feature, including for bringing guests: an account without one is
a spectator.

`confirmed_count` is recomputed with a `COUNT` after every write rather than incremented, so it
cannot drift from the rows it summarises.

### Manager override

`PATCH /{id}/confirmations/{playerId}` lets a `MANAGER` set anyone's answer — for the player who
texts the group instead of opening the app. It is refused **only** on a `CANCELLED` plan; a manager
can still correct confirmations on a `PENDING`, `CONFIRMED` or `GENERATED` one, and it ignores the
confirmation deadline, which is the difference between an override and self-service.

---

## Starters and reserves

A plan may collect more confirmations than the match needs, and for a long time nothing told the
players that. Somebody could confirm, see `CONFIRMED`, and be fourth in line with no way to know.

Every confirmation now carries its standing:

| Field              | Meaning                                                              |
|--------------------|----------------------------------------------------------------------|
| `confirmationRank` | 1-based place among `CONFIRMED` players, in confirmation order       |
| `isStarter`        | Whether that rank is within the match type's required count          |
| `waitlistPosition` | 1-based place in the reserve queue — 1 means next in                 |

All three are **null for anyone not `CONFIRMED`**. A pending or declined player holds no place in the
queue, and a `0` or `-1` would invite a frontend to render one.

**One ordering produces all of it.** Ranks, the starting selection, and the pool team generation
actually draws from are each read from `confirmedAt ASC NULLS LAST, id ASC`. Deriving them separately
would let the badge a player sees disagree with the team they end up in.

**Promotion needs no bookkeeping.** Withdrawing clears `confirmedAt` and drops the player out of the
`CONFIRMED` set entirely, so everyone behind them moves up and the first reserve lands in the freed
slot. It is re-evaluated on every read and again at generation — including between the preview and
the confirm, so a late withdrawal is reflected in the match that gets created rather than in the
preview somebody looked at a minute earlier.

This is why `GET /{id}/confirmations?status=DECLINED` loads the whole ordered list and filters
afterwards. Ranks are positions *within the confirmed set*; a query already narrowed to `DECLINED`
could not produce them, and running two queries would mean two results that can disagree.

---

## Creating a weekly run

A manager can create the next N Fridays in one request rather than thirteen.

```http
POST /api/match-plans/recurring
{
  "title": "Friday 5-a-side",
  "matchType": "SEVEN_A_SIDE",
  "proposedDate": "2026-09-04T19:00:00Z",
  "confirmationDeadline": "2026-09-03T20:00:00Z",
  "occurrences": 8
}
```

Every field except `occurrences` means what it does on the single-plan create and is applied to each
plan in the run. The response is `201` with the whole run in kickoff order, and **no `Location`
header**: a run has no single resource to point at, and naming the first plan would imply the others
hang off it.

### The plans are independent, and that is the design

Nothing links them once written. Each takes its own confirmations and is edited, cancelled,
generated from or deleted on its own, so the existing lifecycle, poll and generation rules apply
completely unchanged — a run is a faster way to reach rows a manager would otherwise create one at a
time, not a new kind of thing.

A series id was considered and rejected. It buys "cancel them all" and costs a second lifecycle to
reason about, starting with what cancelling a run does to the plans in it that already have people
confirmed or a match generated from them. There is no cheap answer to that, and the expensive one is
a second terminal-state problem.

### Weekly only

Every occurrence is exactly seven days after the one before, so a run holds the same weekday at the
same time. Fortnightly and monthly were left out rather than guessed at; monthly in particular needs
a rule for what "the 31st" means in a 30-day month, and a rule nobody asked for is a rule nobody
checked.

**A confirmation deadline is shifted with its own kickoff**, so every plan keeps the same lead time.
One shared deadline would close every poll but the first before anybody could answer it.

### The horizon is a platform limit

How far ahead a run may reach is `PlatformSetting.MATCH_PLAN_RECURRENCE_MAX_MONTHS` — **three months
by default**, range 1–24, stored in `platform_settings` (`V34`) and changed only through
`PATCH /api/admin/platform/settings` behind `PlatformGuard`.

**Not an `AppSetting`.** That catalogue has been keyed by tenant since `V27` and is edited by
`GROUP_ADMIN`, so a cap stored there would be self-service for the people it constrains. One call
writes every plan in the run, so the horizon is really a bound on how many rows a single request
creates — a limit on what one group can do to the deployment. A group that could raise its own
ceiling would not be bounded at all.

The horizon is measured from **now**, not from the first kickoff, so a run starting in eleven months
cannot reach fourteen.

**A run that overruns is refused, never truncated**, and the message names how many occurrences would
have fitted — counted rather than estimated, so the number offered back is one the retry will
actually accept. Writing the nine that fit out of the twelve asked for would return `201`, read as
success, and surface as three missing Fridays weeks later.

### A known limit: seven days is seven days

Occurrences step by seven days of **absolute** time. Across a daylight-saving boundary that moves the
*local* kickoff by an hour — a 19:00 fixture becomes 18:00 once the clocks change.

Fixing it properly means adding weeks in the group's own timezone, and **the schema stores no
timezone anywhere**: kickoffs are instants and the frontend renders them in whatever zone the browser
is in. Hard-coding one would be silently wrong for every group that is not in it, which is a worse
failure than an hour, and the product now onboards groups rather than serving one. Recorded rather
than papered over — see [known gaps](#known-gaps).

---

## Guests

Any member with a linked player may bring an outsider to fill an empty spot.

```http
POST /api/match-plans/5/guests
{ "name": "João (Rui's friend)", "baseSkillRating": 6, "notes": "coming straight from work" }
```

**It is deliberately not role-gated.** Everything that decides whether an invite is legitimate is
*state* — is the poll open, is there room, has this member already brought their allowance, is there
already a João on this plan — and none of it fits in a `@PreAuthorize` expression. The endpoint is
`isAuthenticated()` and the real checks live in `MatchPlanService`, the same conclusion
`DraftSessionService` reached about turn order.

The checks, in order:

| Check                                 | Failure                                                     |
|---------------------------------------|-------------------------------------------------------------|
| Caller has a linked player            | `400` — no player linked to your account                    |
| Poll is open                          | `409`                                                       |
| Plan is not already full              | `409` — guests fill spots, they never extend the queue      |
| Per-inviter cap                       | `409` — `guests.max.per.inviter`, default **2**, range 0–10 |
| No guest of that name on this plan     | `409` — two members adding "João" is probably the same João |

A guest never joins the waitlist. A member queues because a promotion later actually benefits them; a
stranger sitting fourth in line benefits nobody, and promising someone a game the group cannot give
them is worse than saying no. The cap is a per-group setting, resolved against the group the request
is acting in — a closed Sunday league and an open kickabout want different answers, and **zero is a
real policy** meaning members only. It is counted live, so removing a guest frees the slot again.

The name check is scoped to the plan, not the group: a group can perfectly well know three Joãos
across a season.

**Three rows in one transaction** — the guest `Player`, their already-`CONFIRMED` confirmation, and a
payment delegation making the inviter the person the organiser asks about the fee. A guest exists
*because* they are coming to this match, so a half-created one is not a state worth reaching. The
fee itself stays on the guest's own ledger row: `uq_player_charges_plan` forbids a second charge on
the inviter for the same plan, and the split is proven to sum to the plan's total. The delegation
carries the *responsibility*, which is a different thing from the debt.

There is **no phone number**, deliberately. A guest's details are typed by somebody else, about a
person who agreed to a football match but not to an app. A name and a rough skill guess are what a
team sheet needs. It also sidesteps a real gap: `PlayerPiiPolicy` has no concept of "the person who
invited them", so an inviting member could not have seen the number they themselves typed.

`baseSkillRating` is optional, 1–10, and **defaults to 5** — the same value a new player's computed
rating starts at, so generation has something to balance with. Insisting the inviter be precise about
a stranger would only produce confident-looking noise.

### Removing one

`DELETE /{id}/guests/{playerId}`. The **inviter** may remove their own while the poll is open — the
same window as changing their own answer. A **`MANAGER`** may remove any until the plan is
`GENERATED`. Everyone else gets `403`: a guest is somebody's specific arrangement, not communal
property.

What happens to the player row depends on whether there is anything to protect:

- **Never played** → hard-deleted, along with the delegation. The delegation must be *deleted* rather
  than ended — an ended row still references the player being removed and Hibernate rejects that at
  flush. This was found by the first real use of the feature, not by the mocked test suite.
- **Has history** → deactivated, keeping their record, exactly as `PlayerService.deletePlayer`
  refuses to delete anyone holding stats.

---

## What the pitch cost

`total_cost_cents` lives on the plan but is written through the **payments** API, because the
authorization question is a payments one:

| Endpoint                             | Roles                    |
|--------------------------------------|--------------------------|
| `PUT /api/match-plans/{id}/cost`     | `MANAGER` or `ORGANIZER` |
| `POST /api/match-plans/{id}/charges` | `MANAGER` or `ORGANIZER` |

Whoever booked the pitch knows what it cost, and that is the manager; whether money actually arrived
is only known to the person it was sent to, and that is the organizer.

**Null and zero are different.** Null means "nobody has recorded it yet" and generates no charges;
zero means the match was free, which happens. `totalCostCents` is serialised **non-null**, so the key
is *absent* rather than null when unset — without that the value could be saved and never read back,
and the field appeared to reset itself on every render.

**The total is stored, not a per-head figure.** The total is the number somebody actually knows when
they book. Per-head amounts are derived from it once, at charge generation, when the headcount is
finally settled — and each charge then carries its own frozen amount, so correcting the total
afterwards cannot rewrite what people already owe.

Charges are generated automatically at the `CONFIRMED → GENERATED` transition: the moment the squad
is final is the moment the pitch is committed, so it is the moment the fee is owed. Generation is a
no-op when there is no recorded cost, and idempotent — `POST /charges` exists for when the cost was
not known at generation time, and re-running it adds nothing.

---

## Team generation

```
POST /api/match-plans/{id}/generate?generationType=BALANCED          → MatchPreviewDTO (nothing persisted)
POST /api/match-plans/{id}/generate/confirm?generationType=BALANCED  → MatchDTO (201, the Match exists)
```

Both require `MANAGER` and both require `generatable` — `CONFIRMED` **and** kickoff still ahead. The
error message distinguishes the three ways that fails:

| State                | `400` message                                                |
|----------------------|--------------------------------------------------------------|
| `GENERATED`          | Teams have already been generated for this match plan        |
| expired              | Match plan kickoff has already passed                        |
| anything else        | Match plan must be in CONFIRMED status before generating teams |

**The preview persists nothing.** Call it as often as you like with different algorithms to compare
distributions. `RANDOM` will produce a different result each time, by design; `BALANCED` and
`SNAKE_DRAFT` are deterministic.

### Algorithms

| `generationType` | State                                                            |
|------------------|------------------------------------------------------------------|
| `BALANCED`       | Greedy rating equalizer. The default                             |
| `RANDOM`         | Pure shuffle                                                     |
| `SNAKE_DRAFT`    | Alternating top pick                                             |
| `FORM_BASED`     | Last-N-match linearly weighted form score                        |
| `CAPTAIN_PICK`   | Server-side simulation. The *interactive* version is a [draft session](DRAFT_SESSION_FEATURE.md) |
| `STREAK_AWARE`   | ⛔ Resolves, then throws `422` — awaiting `CalculationService`     |
| `MANUAL`         | Rejected here with `400` — it means "the caller supplied the teams", which is `POST /api/matches` |

### Who actually plays

The first `required` confirmations in confirmation order — 10, 14 or 22 by match type. Surplus
confirmations are reserves and are excluded from the draw. Fewer than `required` is a `400`, which is
why the `PENDING → CONFIRMED` transition enforces the match-type minimum as well as
`minPlayersRequired`: generation must not be able to fail after a plan has been declared confirmed.

Two defensive checks sit around the strategy, both `422`:

- A confirmed player who no longer exists — the response names the missing ids so the caller can
  clear the stale confirmations.
- A strategy returning teams that are not both exactly half the required size. A strategy bug must
  not be able to persist a lopsided match.

`confirmGeneration` additionally needs a **current season** (`400` if there is none), reactivates any
inactive player it drew, writes the `Match`, two `MatchTeam` rows and every `PlayerStat` in batches,
marks the plan `GENERATED`, and then generates the fee charges.

### Teams are named after their best player

Not "Team A" and "Team B" — those told you nothing and read identically on every match ever played.
Each side is named `Team {name of its highest-rated player}`, which gives it an identity people
recognise, and it is derived rather than stored, so the same squad always produces the same name.
**Ties break on player id**, not list order: two players on identical ratings is common on a small
roster, and without a tie-break the name would depend on whatever order the strategy happened to
emit, which changes between runs of the same generation.

`MatchPreviewDTO.teams[].name` carries these real names too, so the preview matches what gets
created. The field names `teamARatingAvg` / `teamBRatingAvg` / `ratingDelta` are positional — A is
`teams[0]`.

---

## Reminders

Two independent push notifications, both driven by `ReminderScheduler` hourly on the hour.

| Reminder      | Window        | Sent to                        | Category                |
|---------------|---------------|--------------------------------|-------------------------|
| Deadline near | next 24 hours | Everyone who has **not answered** | `CONFIRMATION_DEADLINE` |
| Match soon    | next 1 day    | Everyone **confirmed**         | `MATCH_REMINDER`        |

Only the people the message is for. Someone who already answered has done what the deadline asks, and
telling them again is exactly the noise that gets a whole notification channel switched off.

**Each send is claimed before it is sent**, with a conditional `UPDATE ... WHERE sent_at IS NULL`
(`V13`). The database picks the winner, so a second application instance updates zero rows and sends
nothing; a read-then-write would leave a window where both instances see null and both send. Claiming
*before* sending means a crash in between loses that reminder rather than re-sending it to everybody
on the next tick — a missed reminder is a small annoyance, a duplicated one is why people turn
notifications off. The guarantee is therefore **at most once**, not exactly once.

The sweep is global across every group — one query rather than waking once per group to ask the same
question — and the tenant is bound **per row**, from the plan the sweep found. A scheduler has no
request to inherit a group from, and each row already knows which one it belongs to.

---

## Tenancy

Every plan and every confirmation carries `tenant_id`, stamped at persist by `TenantStamping`.

- Single-row loads go through `findPlanOrThrow`, which loads by id and then `TenantGuard.assertOwned`
  — an id from another group is refused rather than returned.
- The paginated list query takes `tenantId` as a **non-optional, never-widened** parameter. Every
  other predicate in that query exists to narrow a list somebody asked to be narrowed; this one
  exists so the list was never wider than the caller's group to begin with.
- The cache key is prefixed with the current tenant.

Note the deliberate asymmetry in the list query: status and timeframe are expressed as *widened*
arguments — every status, and bounds that cannot exclude anything — rather than as nullable binds.
A null-valued bind on an enum or timestamp leans on driver type inference, and the `CAST` workaround
is dialect-specific. The same SQL runs every time, so nothing surprises on Postgres that did not
surprise on H2.

### Privacy

`created_by` stores a username, which identifies a person as surely as their name does. Erasure
replaces the attribution with a tombstone and keeps the plans:

- `scrubCreatedBy` — every tenant. Correct for erase-platform, where the account is going and
  usernames are globally unique.
- `scrubCreatedByInTenant` — one group. Somebody leaving their Tuesday five-a-side has not asked to
  be scrubbed from the audit trail of a different group they still play in.

---

## Caching

| Cache       | Populated by            | Evicted by                                            |
|-------------|-------------------------|-------------------------------------------------------|
| `matchPlans`| `GET /api/match-plans/{id}` | Every write in `MatchPlanService`, `allEntries` |

Its own Caffeine cache since the match-plan feature grew past sharing `matches` — 10-minute TTL,
500 entries, per node. `confirmGeneration` evicts `matches` as well, because it creates one.

The paginated list is **not** cached; only the single-plan read is.

---

## API

Base path `/api/match-plans`, except the two cost endpoints, which are served by `PaymentController`.

| Method   | Path                                 | Role                     | Notes                                        |
|----------|--------------------------------------|--------------------------|----------------------------------------------|
| `POST`   | `/`                                  | `MANAGER`                | `201`, `Location` header                     |
| `POST`   | `/recurring`                         | `MANAGER`                | `201` with the whole run, no `Location`      |
| `GET`    | `/`                                  | authenticated            | Paginated. `status`, `timeframe` params      |
| `GET`    | `/{id}`                              | authenticated            | Cached                                       |
| `PATCH`  | `/{id}`                              | `MANAGER`                | `PENDING` only. Null fields ignored          |
| `PATCH`  | `/{id}/status`                       | `MANAGER`                | See the transition table                     |
| `DELETE` | `/{id}`                              | **`GROUP_ADMIN`**        | `PENDING` only. `204`                        |
| `GET`    | `/{id}/confirmations`                | authenticated            | Optional `status` filter                     |
| `POST`   | `/{id}/confirmations/me`             | authenticated            | Upsert. Poll must be open                    |
| `GET`    | `/{id}/confirmations/me`             | authenticated            | Includes the caller's own waitlist standing  |
| `PATCH`  | `/{id}/confirmations/{playerId}`     | `MANAGER`                | Override. Refused only on `CANCELLED`        |
| `POST`   | `/{id}/guests`                       | authenticated            | `201`. State checks, not a role gate         |
| `DELETE` | `/{id}/guests/{playerId}`            | authenticated            | Inviter or `MANAGER`. `204`                  |
| `POST`   | `/{id}/generate`                     | `MANAGER`                | Preview. Persists nothing                    |
| `POST`   | `/{id}/generate/confirm`             | `MANAGER`                | `201`. Creates the `Match`                   |
| `PUT`    | `/{id}/cost`                         | `MANAGER` or `ORGANIZER` | `204`. `PaymentController`                   |
| `POST`   | `/{id}/charges`                      | `MANAGER` or `ORGANIZER` | `PaymentController`. Idempotent              |

The recurrence horizon is not on this controller at all — it is a deployment limit, read and written
through the operator surface:

| Method  | Path                            | Grant             | Notes                                  |
|---------|---------------------------------|-------------------|----------------------------------------|
| `GET`   | `/api/admin/platform/settings`  | platform operator | `AdminController`, behind `PlatformGuard` |
| `PATCH` | `/api/admin/platform/settings`  | platform operator | Keys are `PlatformSetting` names       |

`timeframe` is `upcoming` (kickoff ahead), `past` (kickoff behind), or omitted for both — anything
else is a `400`. It is filtered **in the query, not the client**, because the list is server
paginated: drop rows after the page is built and a page of twenty shows however many survived while
`totalElements` keeps counting the ones that did not. Separating the two is what stops a plan from
three weeks ago sitting above next Friday's.

### Ordering

Both halves run **away from now**:

| `timeframe`  | Order                              | So that                                  |
|--------------|------------------------------------|------------------------------------------|
| `upcoming`   | `proposedDate ASC, id ASC`         | the next match is first                  |
| `past`       | `proposedDate DESC, id DESC`       | the match just played is first           |
| omitted      | ascending                          | neither direction is right for a mixed list; ascending is the plainer reading of "chronological", and the frontend never reaches it |

Opposite directions, one rule: what happens or happened nearest is what somebody is looking for.

**There was no `ORDER BY` at all before this**, and the frontend sends no sort — so row order was
whatever Postgres returned. That is not merely unhelpful but *unstable*: with no total order,
`LIMIT`/`OFFSET` slice an ordering that is not there, and a row can appear on two pages or on none.

`id` is the tie-break and it is what makes the order total. Two plans can share a kickoff — a
recurring run creating a duplicate of an existing Friday does exactly that — and an unbroken tie
brings the pagination instability straight back for the rows that share one.

**A caller who supplies their own `sort` keeps it.** The default fills a gap rather than overruling
anybody; replacing an explicit sort would make `?sort=` silently inoperative on this endpoint alone.

### Roles

The names changed in `V33`: `ADMIN` → `GROUP_ADMIN`, and what used to be `MASTER_USER` / `ADMIN_USER`
is now the flat, per-membership set below. See [multi-tenancy](../../architecture/multi-tenancy.md).

| Action                          | member | `MANAGER` | `ORGANIZER` | `GROUP_ADMIN` |
|---------------------------------|:------:|:---------:|:-----------:|:-------------:|
| Read plans and confirmations    | ✅ | ✅ | ✅ | ✅ |
| Answer for yourself             | ✅ | ✅ | ✅ | ✅ |
| Bring a guest                   | ✅ | ✅ | ✅ | ✅ |
| Remove your own guest           | ✅ | ✅ | ✅ | ✅ |
| Remove any guest                | ❌ | ✅ | ❌ | ❌ |
| Create / update a plan          | ❌ | ✅ | ❌ | ❌ |
| Change status                   | ❌ | ✅ | ❌ | ❌ |
| Override a confirmation         | ❌ | ✅ | ❌ | ❌ |
| Generate teams                  | ❌ | ✅ | ❌ | ❌ |
| Set the pitch cost              | ❌ | ✅ | ✅ | ❌ |
| Create a weekly run             | ❌ | ✅ | ❌ | ❌ |
| **Delete a plan**               | ❌ | ❌ | ❌ | ✅ |
| **Set the recurrence horizon**  | ❌ | ❌ | ❌ | ❌ — platform operator only |

`GROUP_ADMIN` holds exactly one thing here, and holds it *only* — deleting a plan destroys everyone's
answers, so it sits with group administration rather than with running matches. A `GROUP_ADMIN` who
also organises matches holds `MANAGER` too, and says so.

---

## DTOs

### `MatchPlanCreateDTO` — `POST /`

| Field                  | Type      | Required | Validation                                    |
|------------------------|-----------|----------|-----------------------------------------------|
| `title`                | String    | Yes      | `@NotBlank`, 2–100                            |
| `matchType`            | String    | Yes      | one of the three                              |
| `location`             | String    | No       | ≤ 255                                         |
| `proposedDate`         | **Instant** | Yes    | `@FutureOrPresent`, **and** must be after now |
| `confirmationDeadline` | Instant   | No       | must be strictly before `proposedDate`        |
| `description`          | String    | No       | ≤ 500                                         |
| `minPlayersRequired`   | Integer   | No       | `@Positive`. Defaults to, and cannot be below, the match type's total |

Both the kickoff and the deadline checks are comparisons **between instants**. They used to widen the
kickoff to the start of its day, which let a plan be created for a kickoff that had already been and
gone this morning, and let a deadline land after the match had started.

### `RecurringMatchPlanCreateDTO` — `POST /recurring`

Every field of `MatchPlanCreateDTO`, applied to each plan in the run, plus:

| Field         | Type    | Required | Validation                                              |
|---------------|---------|----------|---------------------------------------------------------|
| `occurrences` | Integer | Yes      | `@Min(2)`, `@Max(120)` — and within the platform horizon |

Two is the floor because one occurrence is not a recurrence; `POST /api/match-plans` already does
that, and accepting it here would be a second way to do the same thing. The `@Max` is a
**structural** bound that stops arithmetic on an absurd count before the horizon check runs — it is
not the policy limit, which is the horizon and is enforced in `MatchPlanService`.

### `MatchPlanUpdateDTO` — `PATCH /{id}`

`title`, `location`, `proposedDate`, `confirmationDeadline`, `description`. All optional; **null
means no change**, so a field cannot be cleared through this endpoint. `PENDING` plans only (`409`
otherwise). A supplied deadline is validated against the kickoff being set in the same request, or
the stored one if it is unchanged.

`totalCostCents` is **not** here — see [what the pitch cost](#what-the-pitch-cost).

### `MatchPlanDTO` — response

`id`, `title`, `matchType`, `location`, `proposedDate`, `confirmationDeadline`, `status`,
`confirmedCount`, `minPlayersRequired`, `description`, `createdBy`, `createdAt`, `updatedAt`, plus:

| Field            | Derived from                                        |
|------------------|-----------------------------------------------------|
| `playersNeeded`  | `max(0, minPlayersRequired - confirmedCount)`        |
| `pollOpen`       | `PENDING` and the deadline has not passed            |
| `expired`        | kickoff is behind us                                 |
| `generatable`    | `CONFIRMED` and not expired                          |
| `cancellable`    | `PENDING`/`CONFIRMED` and not expired                |
| `totalCostCents` | **omitted entirely when unset** — absent ≠ `0`       |

### `PlayerConfirmationDTO` — response

`id`, `matchPlanId`, `playerId`, `playerName`, `status`, `notes`, `confirmedAt`, plus
`confirmationRank`, `isStarter`, `waitlistPosition` (see
[starters and reserves](#starters-and-reserves)) and:

| Field               | Notes                                                                 |
|---------------------|-----------------------------------------------------------------------|
| `isGuest`           | Carried on the confirmation, not looked up from the roster — the member view of a plan never loads the roster, and the row has to render its own chip |
| `invitedByPlayerId` | Null for members. What the remove affordance compares against         |

### `ConfirmationUpsertDTO` / `GuestCreateDTO` / `MatchPlanStatusDTO`

| DTO                     | Fields                                                            |
|-------------------------|--------------------------------------------------------------------|
| `ConfirmationUpsertDTO` | `status` (`@NotBlank`), `notes` (≤ 500)                            |
| `GuestCreateDTO`        | `name` (`@NotBlank`, 2–100), `baseSkillRating` (1–10, default 5), `notes` (≤ 500) |
| `MatchPlanStatusDTO`    | `status` (`@NotBlank`)                                             |

### `MatchPreviewDTO` — `POST /{id}/generate`

`matchPlanId`, `matchType`, `generationType`, `location`, `proposedDate`, `generationNotes`,
`teamARatingAvg`, `teamBRatingAvg`, `ratingDelta`, and `teams[]` of
`{ name, ratingAvg, players[{ playerId, playerName, skillRating }] }`.

---

## Errors

| Scenario                                        | Status | Message                                                        |
|-------------------------------------------------|--------|----------------------------------------------------------------|
| Plan not found, or belongs to another group     | `404`  | `MatchPlan with id {id} not found`                             |
| Caller has no linked player                     | `400`  | `No player linked to your account`                             |
| Kickoff not in the future                       | `400`  | `proposedDate must be in the future`                           |
| Deadline not before kickoff                     | `400`  | `confirmationDeadline must be before proposedDate`             |
| `minPlayersRequired` below the match-type total | `400`  | names both numbers                                             |
| Invalid `timeframe`                             | `400`  | `Must be 'upcoming' or 'past'`                                 |
| Run overruns the recurrence horizon             | `400`  | names the horizon and how many occurrences would fit           |
| First kickoff already beyond the horizon        | `400`  | `no plan in this run could be created`                         |
| `occurrences` below 2 or above 120              | `400`  | bean validation on the DTO                                     |
| Unknown enum value                              | `400`  | lists the valid ones                                           |
| Updating a non-`PENDING` plan                   | `409`  | `only PENDING plans can be updated`                            |
| Deleting a non-`PENDING` plan                   | `409`  | `only PENDING plans can be deleted`                            |
| Disallowed status transition                    | `409`  | `Cannot transition match plan from {from} to {to}`             |
| Cancelling after kickoff                        | `409`  | `Cannot cancel a match plan whose kickoff has already passed`  |
| Answering a closed poll                         | `409`  | not `PENDING`, or the deadline has passed                      |
| Confirming without enough players               | `422`  | names the shortfall, twice if the match-type minimum also fails |
| Plan full / guest cap / duplicate guest name    | `409`  | see [guests](#guests)                                          |
| Removing someone else's guest                   | `403`  | `Only the member who brought this guest, or a manager`         |
| Generating from a plan that is not generatable  | `400`  | one of three distinct messages                                 |
| Too few confirmed to generate                   | `400`  | `Confirmed player count must be at least {N}`                  |
| Confirmed player no longer exists               | `422`  | names the missing ids                                          |
| Strategy returned unbalanced teams              | `422`  | names both sizes                                               |
| `STREAK_AWARE`                                  | `422`  | not yet available                                              |
| No current season at generation                 | `400`  | `No active season; cannot create match`                        |

---

## Implementation

| Layer      | Files                                                                          |
|------------|--------------------------------------------------------------------------------|
| Entities   | `MatchPlan`, `PlayerConfirmation`, `PlanStatus`, `ConfirmationStatus`           |
| Platform   | `PlatformSetting` (catalogue), `PlatformSettingValue` (overrides), `PlatformSettingsService`, `PlatformSettingValueRepository` — the deployment's limits, deliberately a separate enum and service from `AppSetting`/`AppSettingsService` so the two cannot end up behind the same guard |
| Service    | `MatchPlanService` — the whole feature, including guests and the waitlist        |
| Controller | `MatchPlanController`; cost and charges on `PaymentController`                   |
| Mapper     | `MatchPlanMapper` (MapStruct). Waitlist fields are `ignore`d — they cannot be derived from one confirmation, so `MatchPlanService` fills them in |
| Repos      | `MatchPlanRepository` (search, reminder claims, privacy scrubs), `PlayerConfirmationRepository` (the ordering) |
| Fees       | `MatchFeeService`                                                                |
| Push       | `ReminderScheduler`, `ReminderDispatcher`                                        |
| Migrations | `V1` initial · `V13` reminder guards · `V17` kickoff + `GENERATED` · `V19` cost · `V20` delegation · `V24`/`V25` tenancy · `V34` platform settings |

Frontend: `src/features/matchPlans/` (page, card, detail modal, create modal, edit form, cost
section, guest modal), `useMatchPlans`, `matchPlanService`, `types/matchPlan.ts`, route
`/match-plans`. The repeat-weekly checkbox lives on `CreateMatchPlanModal`; the horizon control is a
Deployment limits section on `PlatformSettings.tsx`, above the creation codes, since these bound
every group that already exists whereas a code decides whether one more may.

---

## Known gaps

Recorded here rather than left to be discovered — see [CONTRIBUTING](../../CONTRIBUTING.md) rule 5.

1. ~~**Setting the cost does not evict the plan cache.**~~ **Fixed 2026-08-04, ⏳ not yet
   deployed.** `MatchFeeService.setPlanCost` wrote `total_cost_cents` and saved but carried no
   `@CacheEvict`, while `MatchPlanService.getPlan` is `@Cacheable` on `matchPlans` — so a save
   followed by the detail modal's refetch of `GET /api/match-plans/{id}` could be served the
   pre-write entry for up to the 10-minute TTL and show "not set". The frontend's own invalidation
   had already been fixed for this exact symptom (`useSetPlanCost` invalidates both plan query keys,
   which are not related by prefix), and that is why the server half survived: the client stopped
   showing its own stale copy and started refetching into the server's.

   `setPlanCost` now evicts, and `MatchPlanCostCacheIT` pins it. That test is an `*IT` rather than a
   unit test on purpose — the defect lived in an annotation, and neither the Mockito unit tier
   (which never builds the proxy that reads it) nor the `test` profile (`spring.cache.type=none`)
   can tell a correct eviction from a missing one. It asserts the priming read populated the cache
   before asserting the write cleared it, so it cannot pass by caching nothing.

   ⚠️ **It has not been executed.** The fix was written in an environment without Docker, so
   `./gradlew integrationTest` could not run. The unit tier is green. Run it before merging.
2. **A weekly run steps by absolute time, so daylight saving moves the local kickoff.** Seven days
   is seven days; a 19:00 fixture becomes 18:00 once the clocks change. Adding weeks in the group's
   own timezone is the fix, and there is no timezone to add them in — kickoffs are instants and the
   frontend renders them in the browser's zone. Hard-coding one would be silently wrong for every
   group that is not in it. Either a per-group timezone or an explicit "same local time" rule would
   close it; both are larger than the feature that surfaced it.

3. **`STREAK_AWARE` is reachable.** It parses, resolves to a real bean, and throws `422` from inside
   the strategy. Harmless, but the failure happens later than the other unsupported values.
4. **The `parsePlanStatus` error message omits `GENERATED`.** Sending it is accepted by the parser
   and then rejected by the transition table with a `409`, so the `400` never names it — but the
   message claims the valid values are `PENDING, CONFIRMED, CANCELLED`, which is not the enum.
5. **There is no `frontend/features/match-plans.md`.** Every other frontend feature has a file, and
   the match-plan UI is one of the larger ones — seven components and a route. The frontend types are
   unusually well commented, so the material exists.
6. **Unused `LocalDate` imports** remain in `MatchPlanDTO`, `MatchPlanCreateDTO` and
   `MatchPlanUpdateDTO` — leftovers from `V17`. Cosmetic. (`MatchPlanService`'s own was removed
   with the ordering change.)

7. **`V34` and the recurrence feature have never run against a real database.** The unit tier is
   green, but `integrationTest` needs Docker and was unavailable where this was written — so
   `MigrationSchemaValidationIT` has not checked `PlatformSettingValue` against the migration, and
   `MatchPlanCostCacheIT` (gap 1) is still unexecuted alongside it. Run the tier before merging.
