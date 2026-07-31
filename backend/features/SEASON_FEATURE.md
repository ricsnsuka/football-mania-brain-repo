# Season Feature

**Added in:** v1.0.0  
**Date:** May 2026  
**Status:** ✅ Released (Model + Seed; dedicated API TBD)

---

## Overview

A **Season** provides a time-bounded context for matches and player skill-rating history.
Every match belongs to a season. When a season ends, an admin triggers the
**season-end skill-rating transition** which adjusts all players' ratings based on
their performance across the season.

There is currently **no dedicated `/api/seasons` CRUD endpoint** — seasons are managed
directly via the database seed and future admin tooling. The `seasonId` field on
`MatchCreateDTO` allows associating matches with a specific season; when omitted,
the current active season is used automatically.

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
               Admin calls CalculationService.endSeason()
                              │
                              │  1. Compute season-end ratings for all players
                              │  2. Persist SkillRatingHistory entries
                              │     (reason = "Season end: {name}")
                              │  3. Player skillRating values updated
                              │
                              ▼
                       [ SEASON CLOSED ]
                        end_date set, is_current set to FALSE

               New season created with is_current = TRUE
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

1. **No `/api/seasons` REST API** — there is no endpoint to list, create, or close seasons.
   Season management is done directly via DB or programmatic service calls.

2. **No automatic season-end trigger** — `endSeason()` must be called manually by an
   admin (programmatic call). There is no scheduled job.

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
   current implementation does not update it. Future improvement.

---

## Planned Improvements

| Feature                      | Notes                                                          |
|------------------------------|----------------------------------------------------------------|
| `GET /api/seasons`           | List all seasons with match counts and player stats summary    |
| `POST /api/seasons`          | Admin creates a new season                                     |
| `PATCH /api/seasons/{id}/end`| Admin closes a season, triggers rating transition automatically |
| ~~Unique `is_current` constraint on DB~~ | ✅ Done in V10 — see Current Limitations #3     |
| `GET /api/seasons/{id}/leaderboard`  | Player rankings for a specific season                |

> A season write API must open and close seasons in **one transaction** — the V10 index makes
> "mark the new season current" fail while the old one still is. Clear the old flag first, or the
> rollover deadlocks against its own constraint.

