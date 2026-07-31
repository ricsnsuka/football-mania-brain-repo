# Rankings & Leaderboards — API Contract

**Date:** 2026-07-29
**Version:** v1.0.0 (Roadmap Phase 3)
**Status:** APPROVED — backend complete (both read endpoints, cache wiring, eviction)

---

## Scope

Two read-only endpoints over data that already exists. Purely additive — no migration, no new
table, no change to any existing endpoint, DTO or status code.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/rankings` | The league table — one ordered list by `skillRating` | Any authenticated |
| `GET` | `/api/leaderboards` | Category tops — goals, assists, MVPs, longest streak | Any authenticated |

Neither returns contact details. They report name, rating and match record only, so unlike
`/api/players` there is no PII redaction step and no role-dependent response shape.

---

## Design decisions

### These are two features, not one

The roadmap's phrasing blurred them. `/api/rankings` answers *"where do I stand?"* and is ordered
by rating; `/api/leaderboards` answers *"who is best at X?"* and is ordered by a count. They are
kept apart because they are cached separately (`CacheConfig.RANKINGS` / `CacheConfig.LEADERBOARDS`)
and because merging them would force one response shape onto two different questions.

### Almost nothing new is computed

`skillRating`, `totalGoals`, `totalAssists`, `totalMatchesPlayed`, `currentStreak` and
`longestStreak` are already maintained on `Player` by `CalculationService` on every match
completion. Both endpoints are ordering queries over one table.

The one exception is **W/D/L**, counted from `player_stats.match_result` by a single grouped query
for the whole table. No W/D/L columns were added to `Player`: they are derivable, and a fourth
denormalised counter is a fourth thing that can drift out of step with the matches it summarises.

### Minimum appearances — decided up front, on purpose

Ranking on `skillRating` alone puts a player with **one match** at the top. Every player starts at
5.00, so a single good result is enough to lead the table.

A player needs **3 completed matches** to be given a position. Below that they are still listed —
a new player has to be able to find themselves — but with `qualified: false` and **no `rank` field
at all**, and they sort after every ranked player regardless of rating.

> This threshold exists in v1 rather than being added later **deliberately**. Retrofitting it would
> move everybody's rank at once and be indistinguishable from a bug.

The threshold is reported in the response as `minimumMatchesToQualify` rather than left for the
frontend to hard-code, so changing it does not silently make the UI's explanation wrong.

That has stopped being a hypothetical: **the threshold is admin-configurable**, so `3` is the
default rather than a constant, and it can differ between installs and change between two requests.
Reading it from the response is now the only correct way to render "needs N more to qualify".

**It does not apply to leaderboards.** Goals and assists are counting stats, not rates: five goals
in one match is a real five goals, and filtering them would make the top-scorer list disagree with
the match records it is derived from.

### All-time only — and no parameter that pretends otherwise

There is **no `seasonId` parameter**. `Player`'s totals are career-wide, so a season-scoped table
cannot be built from them; it has to go through `skill_rating_history` joined to `match.season_id`,
which is a different query with a different W/D/L path.

All-time-only is the deliberate v1. A `seasonId` parameter that was accepted and quietly ignored
would be worse than not having one — it would look supported and return the wrong table.

Adding it later is additive: a new optional parameter, unchanged default behaviour.

### Ordering is fully deterministic

`skillRating DESC, totalMatchesPlayed DESC, name ASC, id ASC`.

Ties on rating are not an edge case — every new player sits on exactly 5.00 — and a table whose
rows swapped places between two identical requests reads as a bug. Preferring the player with more
matches on a tie is the same judgement the qualification threshold makes: the longer record is the
better-evidenced one.

Ranks are sequential (`1, 2, 3, …`), not shared on a tie. The tie-break chain above is what makes
that assignment stable.

### Active vs. inactive differs between the two, and should

| | Deactivated players |
|---|---|
| `/api/rankings` | **Excluded** by default (`?includeInactive=true` to include) |
| `/api/leaderboards` | **Always included** |

The league table describes the group as it is now. A record board is the opposite: whoever scored
the most goals still scored them after they left.

### `mostMvps` counts the admin's pick

It counts `player_stats.is_mvp` on completed matches. When crowd MOTM voting lands it will be a
**separate** fact with separate authority (`matches.crowd_mvp_player_id`) and must not be folded
into this count — see `docs/plans/PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM.md`.

---

## Endpoints

### GET /api/rankings

**Auth:** `isAuthenticated()`. **Success:** `200`.

| Query param | Required | Default | Notes |
|-------------|----------|---------|-------|
| `includeInactive` | — | `false` | Include deactivated players in the table |

Not paginated. A recreational group's roster is small enough to send whole, and a league table
that arrives in pages cannot be sorted or filtered client-side without refetching.

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
      "rank": 2, "qualified": true,
      "playerId": 11, "playerName": "Ana Costa",
      "skillRating": 7.10, "played": 9,
      "wins": 4, "draws": 1, "losses": 4,
      "goals": 3, "assists": 6,
      "currentStreak": 0, "longestStreak": 3
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

Note the third entry: a 9.90 rating from one match, listed **below** both ranked players. That is
the threshold doing its job, not a sorting error.

> ⚠️ **`rank` is absent from that entry, not `null`.** The application sets
> `spring.jackson.default-property-inclusion: non_null`, so every nullable field is omitted from
> the JSON rather than serialised as `null`. In TypeScript the field arrives as `undefined`, and a
> client testing `rank === null` would classify every unranked player as ranked. **Branch on
> `qualified`**, which is a boolean and always present.

| Field | Notes |
|-------|-------|
| `rank` | **Absent** (not `null`) exactly when `qualified` is `false`. Never `0` |
| `qualified` | `played >= minimumMatchesToQualify` |
| `qualifiedCount` | Number of entries with a rank — also the highest rank issued |
| `totalCount` | Length of `entries` |
| `played` | `Player.totalMatchesPlayed` — career completed matches |
| `wins` / `draws` / `losses` | Counted from `player_stats.match_result` on completed matches. All zero for a player with no completed matches |
| `goals` / `assists` | Career totals. Own goals are **not** counted in `goals` |
| `longestStreak` | A record — it does not reset when the run ends |

`wins + draws + losses` may be **less than** `played` if a completed match was recorded without a
result. It is never more.

#### Error responses

| Status | Trigger |
|--------|---------|
| `403` | Unauthenticated *(Spring Security — no body)* |

---

### GET /api/leaderboards

**Auth:** `isAuthenticated()`. **Success:** `200`.

| Query param | Required | Default | Notes |
|-------------|----------|---------|-------|
| `limit` | — | `5` | Entries **per category**. Clamped to `1`–`25` out of the box |

`limit` is **clamped, not rejected** — entries-per-card is a display preference and a `400` for
asking for too many is friction with no upside. On a default install `limit=5000` returns 25 and
reports `"limit": 25`. Clamping happens before the cache is consulted, so the same response is not
stored under every oversized number a caller tries.

> **The default and the ceiling are admin-configurable** and are no longer guaranteed to be 5 and
> 25. An admin can change either through the settings API, so a client must not hard-code them.
> **Read the value back from the response's `limit` field** rather than assuming the request was
> honoured — that field is what was actually applied. The lower bound of `1` is fixed.

```json
{
  "limit": 5,
  "topScorers": [
    { "rank": 1, "playerId": 10, "playerName": "Joao Silva", "value": 30 },
    { "rank": 2, "playerId": 12, "playerName": "New Player", "value": 3 }
  ],
  "topAssists": [
    { "rank": 1, "playerId": 11, "playerName": "Ana Costa", "value": 25 }
  ],
  "mostMvps": [
    { "rank": 1, "playerId": 10, "playerName": "Joao Silva", "value": 6 }
  ],
  "longestStreaks": [
    { "rank": 1, "playerId": 11, "playerName": "Ana Costa", "value": 9 }
  ]
}
```

- All four lists are **always present**, ordered best-first, and empty rather than absent when
  nothing has been recorded in that category. A frontend rendering four cards never has to
  null-check one.
- `value` is whatever the category counts: goals, assists, MVP awards, or streak length.
- **Zero-valued players are omitted.** A "top scorers" list padded with people who have never
  scored is noise, and it makes a young season look busier than it is.
- Ranks are numbered per list. The same player can be 1st in one category and 4th in another.
- Each list may be shorter than `limit` — that means the category ran out of non-zero entries, not
  that the page ended.

#### Error responses

| Status | Trigger |
|--------|---------|
| `403` | Unauthenticated *(Spring Security — no body)* |

---

## Caching and eviction

Both caches were declared in `CacheConfig` before either endpoint existed. They are now read.

| Cache | Key | TTL |
|-------|-----|-----|
| `rankings` | `table-<includeInactive>` | 10 min (Caffeine default) |
| `leaderboards` | `boards-<limit>` | 10 min |

**Both are evicted together, everywhere.** The two are derived from the same writes, and evicting
one without the other is exactly the failure that shows up days later as "the leaderboard is
stale", far from its cause.

| Evicted by | When |
|------------|------|
| `MatchEventListener` | After a successful post-completion recalculation |
| `MatchService` | `POST /api/matches/{id}/recalculate` and the bulk endpoint |
| `PlayerService` | Any player write — create, update, status, delete, link, unlink |
| `PrivacyService` | Erasure (`eraseByUser` / `erasePlayer`) |

> `MatchEventListener` previously evicted `RANKINGS` only. `LEADERBOARDS` was added here: a
> completed match moves goal, assist, MVP and streak totals just as surely as it moves ratings.

`PlayerService` evicts on *every* write rather than the subset that demonstrably changes the
derived views. The interesting cases (a deactivation removes someone from the table, a rename
changes what it says) are already in scope, and deciding per method which player fields reach which
view is a judgement that has to be revisited whenever either side changes.

**Multi-node caveat** applies as documented in `CacheConfig`: Caffeine is per-process, so on more
than one instance a write evicts the writing node and the others serve their copy until TTL. Ten
minutes of stale league table is an accepted trade-off, not an oversight.

---

## Privacy

Neither endpoint exposes anything not already visible to an authenticated user on
`/api/players` — names and match statistics, no email or phone.

GDPR erasure anonymises rather than deletes: the name becomes a tombstone and the player is
deactivated. The consequences here are deliberate and worth stating:

- **`/api/rankings`** — the erased player drops out, because erasure deactivates them and the table
  excludes inactive players by default. They reappear under the tombstone name with
  `?includeInactive=true`.
- **`/api/leaderboards`** — the erased player **keeps their entry**, under the tombstone name. That
  is the documented behaviour of erasure (the statistics survive, the person behind them does not)
  rather than a gap: removing the entry would retroactively alter other players' match records.

Erasure evicts both caches, so the real name is not served from cache after the request is
actioned. See [PRIVACY_AND_DATA_PROTECTION.md](../features/PRIVACY_AND_DATA_PROTECTION.md).

No new personal-data table was introduced, so the three-place checklist in that document (data
table, `PrivacyService`, `PrivacyServiceTest`) does not apply to this change.

---

## Frontend Migration Notes

### Rendering the table

```ts
const { minimumMatchesToQualify, entries } = await apiFetch('/api/rankings').then(r => r.json());
```

- **Branch on `qualified`, never on `rank`.** `rank` is *omitted* for unqualified players (see the
  warning above), so it is `undefined` rather than `null` and `rank === null` is always false.
- **Render the missing rank as `—`**, not as `0` or a blank cell. The response tells you why:
  `minimumMatchesToQualify - played` more matches to go.
- **Take the threshold from the response**, not a constant. `"Needs ${minimumMatchesToQualify -
  played} more match(es) to be ranked"` stays correct if it changes.
- Unqualified entries are already sorted last — do not re-sort the array by `skillRating`, or they
  will jump to the top and undo the whole point.
- `wins + draws + losses` can be less than `played`. Do not derive one from the others.

### Rendering the boards

- Render from the four named lists; each may be empty and each may be shorter than `limit`.
- A short list means the category ran out, not that there is another page. There is no pagination.
- `value` has a different unit per category — label it per card ("goals", "assists", "MVPs",
  "matches"), not once for the whole component.

### Caching

Responses are cached server-side for up to 10 minutes and evicted on the writes listed above.
Client-side polling faster than that gets the same bytes; refetching after a match completes is the
useful trigger.

---

## Breaking Changes

- [x] **None.** Two new endpoints, no new table, no migration. No existing endpoint, DTO, field or
  status code changes.

Behavioural changes to existing code, both additive:

- `MatchEventListener` now also evicts `leaderboards` after a successful recalculation.
- `PlayerService` and `PrivacyService` writes now also evict `rankings` and `leaderboards`.
