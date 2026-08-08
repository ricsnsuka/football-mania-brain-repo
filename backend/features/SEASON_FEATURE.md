# Season Feature

**Added in:** v1.0.0  
**Date:** May 2026  
**Status:** ✅ Released (Model + Seed). **Write API added 2026-08-07** — `SeasonController`,
`SeasonService`, `SeasonDTOs`, and **no migration**. Contract:
[SEASONS-API-CONTRACT](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md).
The screen that drives it is [frontend/features/seasons.md](../../frontend/features/seasons.md).

---

## Overview

A **Season** provides a time-bounded context for matches and player skill-rating history.
Every match belongs to a season. When a season ends, an admin triggers the
**season-end skill-rating transition** which adjusts all players' ratings based on
their performance across the season.

~~There is currently **no dedicated `/api/seasons` CRUD endpoint**~~ — **there is, as of
2026-08-07.** For the first eighteen months there was not, and seasons were managed via the
database seed alone: every group ran on the "Season 1" `GroupService` creates at founding, current
forever, never closed. `GET`/`POST /api/seasons` and the two lifecycle endpoints close that.

The `seasonId` field on `MatchCreateDTO` still associates a match with a specific season; when
omitted, the current season is resolved automatically — which is the flag
`POST /api/seasons/{id}/start` moves.

---

## Data Model

### `seasons` Table

| Column       | Type           | Nullable | Notes                                               |
|--------------|----------------|----------|-----------------------------------------------------|
| `id`         | BIGSERIAL (PK) | No       |                                                     |
| `name`       | VARCHAR(100)   | No       | Unique. e.g. `"Season 1"`, `"Spring 2026"`          |
| `start_date` | DATE           | No       | Start of the season                                 |
| `end_date`   | DATE           | Yes      | Null until the season is closed                     |
| `is_current` | BOOLEAN        | No       | Default `false`. Only one season should be current  |
| `created_at` | TIMESTAMPTZ    | No       |                                                     |

### Default Seed

On a fresh database (via `V1__initial_schema.sql`), a default season is created:

```sql
INSERT INTO seasons (name, start_date, is_current)
VALUES ('Season 1', CURRENT_DATE, TRUE);
```

This ensures there is always a "current season" so that `POST /api/matches` can resolve
a season without requiring an explicit `seasonId` on every request.

---

## How Seasons Are Used

### Match Association

When creating a match via `POST /api/matches`:

- If `seasonId` is provided → the match is linked to that season.
- If `seasonId` is omitted → the service queries `seasons WHERE is_current = true`.
  If no current season is found, the request fails with `400 Bad Request`:
  > `"No active season found; please specify a seasonId"`

### Filtering Matches by Season

`GET /api/matches` supports a `seasonId` query parameter:

```
GET /api/matches?seasonId=1&completed=true
```

### Season-End Transition

Triggered by calling `CalculationService.endSeason(seasonId)` (programmatically by an admin).
See [CALCULATION_SERVICE.md](CALCULATION_SERVICE.md) for the full formula.

The transition reads `skill_rating_history` entries linked to the season's matches to
compute average and starting ratings for each player, then adjusts `skillRating` accordingly.

---

## Season Lifecycle

```
                   DB seed / admin creates Season
                              │
                              ▼
                       [ ACTIVE SEASON ]
                        is_current = TRUE
                              │
                  Matches reference this season
                  SkillRatingHistory entries accumulate
                              │
                              ▼
          Admin: POST /api/seasons/{id}/finalise
                              │
                              │  1. Compute season-end ratings for all players
                              │  2. Persist SkillRatingHistory entries
                              │     (reason = "Season end: {name}")
                              │  3. Player skillRating values updated
                              │  4. end_date set, is_current cleared (the endpoint,
                              │     not endSeason — see limitation 4)
                              │
                              ▼
                       [ SEASON CLOSED ]
                        end_date set, is_current set to FALSE

        The group now has NO current season, and four paths
        refuse to work until POST /api/seasons/{id}/start
```

---

## Skill Rating History — Season Link

Every `skill_rating_history` row is linked to a match (for per-match updates) or has
`match_id = NULL` (for season-end entries). Season association is inferred from the match:
`skill_rating_history.match_id → matches.season_id`.

The `historyRepository.findAllByPlayerAndSeason(playerId, seasonId)` query fetches
all history entries for a player during a given season to compute:

- `startRating` — first entry's `rating_before`
- `avgRating` — average of all entries' `rating_after`
- `matchesPlayed` — count of entries (one per match participated in)

---

## Current Limitations

1. ~~**No `/api/seasons` REST API**~~ — **resolved 2026-08-07.** Four endpoints: list, define,
   start, finalise. Reads are open to any member, writes are `GROUP_ADMIN`. See
   [SEASONS-API-CONTRACT](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md).

2. **No automatic season-end trigger** — and deliberately still none. `endSeason()` runs when an
   administrator calls `POST /api/seasons/{id}/finalise`, not on `end_date` and not on a schedule.
   A job that closed a season by the calendar would re-rate a whole group at midnight on a date
   somebody typed weeks earlier, with nobody watching; the irreversible step stays a decision
   somebody makes.

3. ~~**`is_current` is not enforced to be unique**~~ — **fixed in V10**
   (`ux_seasons_single_current`, a partial unique index over the `is_current = TRUE` rows).

   The original note here also misdescribed the failure: the application did **not** "pick the
   first match if multiple exist". `SeasonRepository.findByCurrentTrue()` returns
   `Optional<Season>`, so a second current row made Spring Data raise
   `IncorrectResultSizeDataAccessException` — a 500 on every request that resolves a season,
   which is `POST /api/matches`, match-plan confirmation and draft-session creation. A partial
   index rather than `UNIQUE(is_current)` because the latter would also forbid a second *closed*
   season, which is what the table is mostly full of.

4. **`end_date` is not automatically set** on `endSeason()` — the column exists but the
   current implementation does not update it. It has been in the schema since V1 and **has never
   been written by anything.**

   **`POST /api/seasons/{id}/finalise` now sets it** — the endpoint, not `endSeason()`, which
   keeps the service method callable from a replay or a test without closing the season underneath
   it. That endpoint is the only thing in any version that has ever written this column.

   It stopped being cosmetic when the write API landed: a season's state is derived from
   `is_current` and `end_date` alone, so without the date a finalised season is indistinguishable
   from one nobody has started.

---

## Planned Improvements

| Feature                      | Notes                                                          |
|------------------------------|----------------------------------------------------------------|
| ~~`GET /api/seasons`~~ | ✅ **Done 2026-08-07** — newest `startDate` first, tie-broken by id, with both match counts from one `GROUP BY` |
| ~~`POST /api/seasons`~~ | ✅ **Done** — defines a season **not current**. Duplicate name is 409, checked case-insensitively because the index is not |
| ~~`POST /api/seasons/{id}/start`~~ | ✅ **Done** — clears the old flag with `saveAndFlush` before setting the new one. Idempotent when already current |
| ~~`POST /api/seasons/{id}/finalise`~~ | ✅ **Done** — runs `endSeason()`, **sets `end_date`** (see limitation 4), clears `is_current`, evicts the rating caches. Refuses a season that was never started |
| ~~`PATCH /api/seasons/{id}/end`~~ | Superseded. Three verbs rather than one patch: only one of them is irreversible, and a shared endpoint gives all three the same shape, audit line and permission |
| ~~Unique `is_current` constraint on DB~~ | ✅ Done in V10 — see Current Limitations #3     |
| `GET /api/seasons/{id}/leaderboard`  | Player rankings for a specific season. Still unspecified — `GET /api/rankings` is all-time by design, and season-scoping it is its own piece of work |

All four are written up in full — request shapes, failure codes, the derived status and what the
client does with each — in
[SEASONS-API-CONTRACT](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md),
with the screen's half in [frontend/features/seasons.md](../../frontend/features/seasons.md).
**None of them needed a migration**: every field is an existing column or a count over
`matches.season_id`.

> A season write API must open and close seasons in **one transaction** — the V10 index makes
> "mark the new season current" fail while the old one still is. Clear the old flag first, or the
> rollover deadlocks against its own constraint.

