# Seasons — the group admin's screen

**Status:** ✅ Both halves written 2026-08-07. The backend endpoints exist; deploy them first.

**A section of the group tab in Settings**, `GROUP_ADMIN` only. It was briefly a page at `/seasons`
with a navbar entry — see [Why settings and not a page](#why-settings-and-not-a-page).

> The contract is
> [`docs/api/SEASONS-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md)
> in the backend repo, per [CONTRIBUTING rule 2](../../CONTRIBUTING.md) — it ships in the same
> commit as the endpoints. This page keeps the UI half and the reasoning behind the shape.

---

## What was missing

Seasons have had a table, an entity and a repository since `V1`, and
`CalculationService.endSeason()` — the whole season-end rating transition — has been callable from
Java the entire time. None of it was ever reachable over HTTP.
[SEASON_FEATURE](../../backend/features/SEASON_FEATURE.md) has said so plainly under "Current
Limitations" since it was written: *no `/api/seasons` REST API*, *no automatic season-end trigger*,
*`end_date` is not automatically set*.

The practical effect: every group's season was the one `GroupService` seeded when the group was
founded — "Season 1", `is_current = true`, `start_date` the day the group was created, `end_date`
null — and nothing in the product could create a second one, switch to it, or close the first. A
group's ratings therefore never had a season boundary, which is the one thing the rating model was
designed to have.

---

## Why settings and not a page

This shipped first as `/seasons`, a route of its own with a `GROUP_ADMIN` navbar entry, and that was
the wrong shape twice over.

**The navbar is for places you move between during a session.** It was already eight items long for
an administrator, and a season is touched a handful of times a year — the row is the most expensive
real estate in the app and this is close to its least frequent destination.

**More to the point, a season belongs to the active group.** It changes when you switch groups,
which is precisely what the group tab in Settings is: the tab *titled with the group's name*,
holding the things that are true of that group rather than of your account. The competition's rules
already lived there. Its calendar being on the navbar instead made two halves of one subject sit in
two different places, and only one of them moved when you switched groups.

It sits immediately above the System section, so the two administrative blocks that describe how the
group actually competes — its calendar, then its rules — read as one thing.

The one cost is discoverability, and it is smaller than it looks: the warning that matters (no
current season) is only ever *caused* by finalising, which happens on this screen, so anybody who
can create the state is standing in the place that reports it.

---

## Three verbs, three endpoints

**Define, start and finalise are separate on purpose.** They are separate decisions made at
separate times, and they do wildly different amounts of damage.

| Verb | Endpoint | Touches | Reversible |
|---|---|---|---|
| Define | `POST /api/seasons` | one new row, **not** current | yes — nothing depends on it yet |
| Start | `POST /api/seasons/{id}/start` | `is_current` on two rows | yes — start the other one back |
| Finalise | `POST /api/seasons/{id}/finalise` | **every player's rating**, plus `end_date` and `is_current` | **no** |

A single `PATCH /api/seasons/{id}` would have given all three the same shape in the client, the
same audit line on the server and the same permission — and the only thing separating "renamed a
season" from "re-rated the whole group" would be which field of the body happened to be set.

### Define ≠ start

A season is usually written down weeks before it begins: the dates are known, the fixture list is
out, nobody has kicked a ball. A create that also made the new season current would silently move
every match recorded in that gap. So `POST /api/seasons` returns a season with `current: false`,
and the modal says so in the subtitle rather than leaving it to be discovered.

### Start ≠ finalise

**This is the sentence the whole screen exists to say.** Starting season 2 clears season 1's
`is_current` and stops there. Season 1 keeps a null `end_date`, its ratings are never taken, and
the season-end transition it was owed simply never runs — no error, no warning, and nothing
afterwards that can tell it happened apart from the rating history rows that are not there.

The server cannot refuse this, and should not: a group that has genuinely abandoned a season must
be able to move on from it. So the only place anybody will ever be warned is the confirmation, and
the confirmation names the displaced season rather than describing it. "The current season" is a
phrase an administrator has to resolve; "Season 1" is one they recognise.

---

## The contract

### `SeasonDTO`

```jsonc
{
  "id": 12,
  "name": "Season 2",
  "startDate": "2026-07-01",       // LocalDate — no time, no zone
  "endDate": null,                 // omitted entirely when null (non_null inclusion)
  "current": true,
  "matchCount": 6,                 // optional — see below
  "completedMatchCount": 5         // optional — see below
}
```

**No migration is required by any of this**, which is deliberate. Every field is either a column
that already exists on `seasons` or a count derivable from `matches.season_id`. A contract that
needs a schema change needs a deploy window and a place in
[database-migrations](../../architecture/database-migrations.md); this one needs neither, so the
backend half is a controller, a service and a DTO.

**`matchCount` and `completedMatchCount` are always sent** — the backend answers both from one
`GROUP BY` over `matches.season_id` for the whole list. The frontend still declares them
`nullableField` and renders an em dash when absent, which is not dead code: it is what makes the
screen readable against a backend one release behind, and *a season with no matches* and *a backend
that does not report matches* are different facts that "0" would collapse into one.

### Status is derived, not sent

```
current === true   → ACTIVE
endDate !== null   → FINALISED
otherwise          → PLANNED
```

There is no `status` field, and adding one would be a second copy of a fact `is_current` and
`end_date` already carry between them — the kind that disagrees with the first after some later
endpoint updates one and not the other.

`current` is checked **first**. A row carrying both a current flag and an end date is one the
server should never produce, but if a half-applied transition or a hand-run `UPDATE` ever makes
one, `current` is still the flag `POST /api/matches` resolves — so that is where matches are
landing, and reading it as finished would hide the very row an administrator needs to act on.
`src/tests/seasons/season.test.ts` pins that ordering.

`PLANNED` rather than `DRAFT`: a draft in this app is a captain-pick session, and two meanings for
one word in one admin surface is how somebody purges the wrong thing.

### The endpoints

| | Grant | Returns | Known failures |
|---|---|---|---|
| `GET /api/seasons` | member of the group | `SeasonDTO[]`, newest `startDate` first | 404 = **not deployed** (see below) |
| `POST /api/seasons` | `GROUP_ADMIN` | 201 + `SeasonDTO`, `current: false` | 409 name already used in this group |
| `POST /api/seasons/{id}/start` | `GROUP_ADMIN` | `SeasonDTO` | 409 season already finalised |
| `POST /api/seasons/{id}/finalise` | `GROUP_ADMIN` | `SeasonDTO` | 409 season already finalised |

No group id appears in any path. Seasons are `TenantOwned` and the server resolves them against
`X-Group-Id`, which `apiFetch` sends for whatever group is active — see
[multi-tenancy](../../architecture/multi-tenancy.md).

**The ordering is the server's and the client never re-sorts it.** Not for the reason the rankings
table does not — there is no qualification threshold to undo here — but because "newest first" is a
claim about `start_date` the client cannot make: the dates are `YYYY-MM-DD` strings, and two
seasons beginning on the same day have only their insertion order to separate them.

### Two things the backend had to get right, and did

1. **Start clears before it sets, in one transaction.** `ux_seasons_tenant_single_current` (V10,
   rescoped per group by V26) is a partial unique index over the `is_current = true` rows. Setting
   the new flag while the old one still stands violates it, and the request fails having changed
   nothing. `SeasonService.start` clears with `saveAndFlush` first — Hibernate orders updates of one
   entity type by its own action queue, not by how the code reads — and
   `SeasonCurrentConstraintIT` pins **both** directions against real PostgreSQL, because a partial
   index is not expressible in the JPA entity and the unit tier's generated schema therefore has
   no such constraint to violate.
2. **Finalise sets `end_date`.** `endSeason()` still does not — limitation 4 in SEASON_FEATURE, and
   deliberately still true, so the method stays callable from a replay without closing the season
   underneath it. The endpoint sets it. The column has existed since V1 and until now nothing had
   ever written it; without it there is no way to tell a finalised season from a planned one and
   the derivation above collapses.

One rule the frontend does **not** encode, because the server does: **only the current season can
be finalised**, 409 otherwise. The transition weights each player against the group mean by
participation, so run over a season with no history it re-rates everybody for no reason and leaves
nothing pointing back at the request that did it.

---

## The un-deployed state, which outlives the backend being written

Both halves ship together, so **the backend must be released and confirmed first** —
[Deployment order](../../CONTRIBUTING.md#deployment-order). The 404 branch is what the screen does
in the window where that has not happened yet.

That window is not hypothetical. On 2026-08-02 the frontend went live against a backend that was
merged and undeployed; every authenticated request failed preflight, every screen degraded to its
empty state, and the outage was reported as *"the competition rules section is empty"* — the
section immediately below this one.

So a 404 from `GET /api/seasons` renders its own state: *"Seasons are not available on this
deployment yet — the backend does not have the season endpoints"*, naming the endpoint, with the
"Define a season" button disabled. It is not the error state, and the difference is the point.
"Failed to load data" sends somebody to check their connection, the deployment and their group, in
that order, and only one of the three is wrong.

**A 404 here is unambiguous** because the list path carries no id. There is no season it could be
failing to find. Every other failure gets the ordinary `DataStateMessage` with a retry.

`useSeasons` also sets `retry: false`, against the app-wide default of three attempts with backoff.
A 404 will not become a 200 by asking again, and eight seconds of spinner before the same answer
defeats a state whose whole value is being immediate.

---

## No current season is the loudest thing this section can say

A group without one is a group where **four separate backend services throw**: `POST /api/matches`,
match-plan confirmation and draft-session creation all resolve `is_current` when no `seasonId` is
given, and none of them tolerate its absence. `GroupService` seeds one at group creation precisely
because of this.

Finalising is the first thing in the product that can *cause* that state — the transition clears
`is_current` and there is nothing to replace it. So:

- The finalise confirmation lists it as a consequence, in the warning treatment, before the click.
- The section shows a standing amber banner above the list whenever the group has no current
  season, whether or not the person came here to look at it.
- The banner waits for the list to arrive. Announcing it mid-request would be a warning about the
  network dressed up as a warning about the group.

---

## Finalising: typing the name

The confirm button is disabled until the season's name is typed exactly — trimmed, case-sensitive.

The two-click confirmations used elsewhere in the app (suspending a member, cancelling a draft,
purging a session) guard actions that undo. This one does not: the transition writes one
`skill_rating_history` row per player and moves every `skillRating` by up to ±2.0, and the ratings
it replaced are gone. Rebuilding the previous table means replaying every match through the bulk
recalculation endpoint. A pattern that means "are you sure" everywhere cannot also mean "this
cannot be taken back", so this borrows the bar from account erasure in
[privacy](privacy.md) — the only other irreversible thing a person can do here.

Case-insensitive was considered and rejected: the name is on screen directly above the field, so
matching it exactly costs nothing, and a looser comparison is a looser gate on the one action that
has no inverse. Whitespace **is** trimmed — a trailing space from a paste is not a different
answer.

The mutation is slow by nature, like the badge backfill: it returns when every player has been
re-rated, not when the request is accepted. The button stays disabled throughout so a second click
cannot start a second transition over the first.

---

## What finalising invalidates

Every player's rating moved, so everything derived from a rating is wrong the instant it returns —
and several of those caches survive a browser restart via the query persister.

`useFinaliseSeason` drops `['seasons']`, `['rankings']`, `['leaderboards']`, `['players']` and
`['player']`. The last one is singular on purpose: the modal that opens from a rankings row is
cached per player, and the rating on it moved too. Leaving these would show the pre-transition
league table until something else happened to invalidate it, which for the leaderboards is nothing
at all.

---

## Files

### Frontend

| | |
|---|---|
| Section | `src/features/seasons/SeasonSettings.tsx`, composed by `features/settings/SettingsPage.tsx` on the group tab, lazily and `GROUP_ADMIN`-gated |
| Card | `src/features/seasons/SeasonCard.tsx` — one control per state, never two |
| Modals | `CreateSeasonModal` · `StartSeasonModal` · `FinaliseSeasonModal` |
| Hooks | `src/hooks/season/useSeasons.ts` |
| Service | `src/services/seasonService.ts` |
| Types | `src/types/season.ts` — the DTO, the schemas and `seasonStatus()` |
| Tests | `src/tests/seasons/` — 18 across three files, plus the group-tab composition in `tests/settings/SettingsPage.test.tsx` |
| Visual | `e2e/fixtures.ts` stubs `/api/seasons` with one season of each state — a fixture with only the current one would screenshot a third of the section |

### Backend

| | |
|---|---|
| Controller | `SeasonController` — `/api/seasons`, reads open to the group, writes `GROUP_ADMIN` |
| Service | `SeasonService` — the three verbs, the rollover order, the cache evictions |
| DTOs | `SeasonDTOs` — `SeasonCreateDTO`, `SeasonDTO` |
| Repositories | `SeasonRepository` (list, name check) · `MatchRepository.countMatchesPerSeason` |
| Tests | `SeasonServiceTest`, `SeasonControllerTest`, and the rollover in `SeasonCurrentConstraintIT` |
| Contract | [`docs/api/SEASONS-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md) |

Cards rather than a table, which is the opposite choice to the draft-session admin next door. That
page lists dozens of near-identical rows and the reader is scanning for one; this one lists a
handful of long-lived things and the reader is deciding about each in turn. A table would also have
to put a destructive confirmation inside a cell — the exact defect that page had to grow a mobile
card layout to fix.

---

## Still open

- **Deleting a planned season.** There is no endpoint and no control. A season defined by mistake
  can be renamed around but not removed. Left out deliberately rather than forgotten: deleting one
  that has matches is a data question nobody has answered, and the gate would have to know.
- **The season filter that already exists.** `GET /api/matches?seasonId=` has worked the whole
  time, and the bulk recalculation panel still asks an administrator to **type a raw season id into
  a text box** (`RecalculateMatchesPanel`, `matches.recalculate.fields.seasonId`). Now that seasons
  can be listed, both should become a picker. Out of scope for "define, start, finalise" — one
  branch, one reason to exist — but it is the obvious next change.
- **Which season a match belongs to is still invisible.** `MatchDTO.seasonId` is on the wire and no
  screen renders it, which is what makes an accidental start undetectable until somebody finalises
  a season and finds it empty.
