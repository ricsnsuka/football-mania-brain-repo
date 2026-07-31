# Achievement Badges — API Contract

**Date:** 2026-07-29
**Version:** v1.0.0 (Roadmap Phase 3)
**Status:** APPROVED — backend complete (catalogue, awarding, endpoint, GDPR)

---

## Scope

One read endpoint over milestones derived from aggregates that already exist. One migration
(V15), one new table. No existing endpoint, DTO or status code changes.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/players/{id}/badges` | Badges this player has earned, oldest first | Any authenticated |

Awarding has no endpoint. It happens automatically in `MatchEventListener` after a match's ratings
are recalculated — the same async hook the roadmap pointed at, so there is no new service and no
new scheduled job.

---

## Design decisions

### Awards are permanent

There is no revocation path anywhere in the code. If an admin later amends a scoresheet downward,
the badge stays.

`awarded_at` records that a threshold was crossed at that moment, which stays true regardless of
later edits — and taking an achievement back because somebody fixed a typo is hostile. The
alternative (recomputing from current stats) would let a player silently lose a badge for a reason
that has nothing to do with them.

### Idempotency is the whole design, because replays are routine

`uq_player_badges UNIQUE (player_id, badge)` and insert-only awarding.

This matters more than it usually would. Awarding runs from `MatchEventListener`, which:

- **retries up to 3 times** on optimistic-lock conflicts, and
- is **re-run for every completed match** whenever an admin triggers a bulk rating recalculation.

So re-evaluation is normal operation, not an edge case. Because awarding only ever inserts, a
replay can add a badge that is newly deserved but can never disturb one already held. Without the
constraint, one bulk recalculation would shower the entire roster in duplicates.

> The "already held?" check in `BadgeService` is an **optimisation, not the guarantee**. Two
> concurrent completions can both pass it; the constraint decides, and the resulting
> `DataIntegrityViolationException` is swallowed because the player has the badge either way.

### Thresholds read career totals, not this match

By the time awarding runs, `CalculationService` has already folded the match into `Player`'s
aggregates. So "has 10 goals" is a property of the player, read from `totalGoals` — the badge rules
never re-count anything.

That is deliberate: a rule that summed goals itself would be a second implementation of the
counting, free to disagree with the first.

### No backfill for existing players

Every badge would land with today's timestamp, so `awarded_at` would claim the whole roster earned
everything at once — a worse lie than an empty table.

**Consequence worth expecting:** the first completed match after this ships awards each participant
everything they already qualify for, all at once. A ten-year veteran gets six badges from one
match, each citing that match. That is correct, and it is why there is no "recently earned"
notification.

### The streak badge reads `longestStreak`

Not `currentStreak`. The run happened, and it did not un-happen when it ended — a badge keyed on
the current streak would have to be revoked, and nothing here revokes.

---

## The catalogue

Nine badges, all computable from existing data. Adding one is an enum constant plus a threshold;
no new counters.

| Badge | Display name | Earned when |
|-------|--------------|-------------|
| `FIRST_MATCH` | First match | 1 completed match |
| `TEN_MATCHES` | 10 matches | 10 completed matches |
| `FIFTY_MATCHES` | 50 matches | 50 completed matches |
| `FIRST_GOAL` | First goal | 1 career goal |
| `TEN_GOALS` | 10 goals | 10 career goals |
| `FIFTY_GOALS` | 50 goals | 50 career goals |
| `FIRST_ASSIST` | First assist | 1 career assist |
| `WIN_STREAK_5` | 5-match streak | `longestStreak` reaches 5 |
| `FIRST_MVP` | First MVP | Named MVP by an admin at least once |

`FIRST_MVP` counts the **administrator's** `player_stats.is_mvp`, not the crowd MOTM result — two
different facts with two different authorities, as `MOTM-API-CONTRACT.md` sets out.

Declaration order in the `Badge` enum is the **display order**. Reordering is safe for the database
(the name is stored, not the ordinal) but silently rearranges every player's profile.

---

## GET /api/players/{id}/badges

**Auth:** `isAuthenticated()`. **Success:** `200`. Oldest first.

```json
[
  { "badge": "FIRST_MATCH", "displayName": "First match",
    "awardedAt": "2026-03-01T20:00:00Z", "matchId": 80 },
  { "badge": "FIRST_GOAL", "displayName": "First goal",
    "awardedAt": "2026-03-08T20:00:00Z" }
]
```

| Field | Notes |
|-------|-------|
| `badge` | Stable identifier. Switch on this, never on `displayName` |
| `displayName` | Sent by the server so a new badge appears without a frontend change — the same reasoning as `GET /api/push/preferences` returning its category list |
| `matchId` | The match that earned it. **Absent** when that match has since been deleted |

> ⚠️ **`matchId` is omitted, not null**, when unknown — `spring.jackson.default-property-inclusion:
> non_null` again. It arrives as `undefined` in TypeScript. Same rule as `rank` on the league table
> and the nullable fields on the MOTM poll.

An empty array is returned for a player who has earned nothing, and for a player who does not
exist — the endpoint does not 404, because "no badges" is the honest answer either way and probing
for which ids exist is not a capability worth handing out.

| Status | Trigger |
|--------|---------|
| `403` | Unauthenticated *(Spring Security — no body)* |

---

## Privacy

`player_badges` is personal data — an achievement is a fact about a person — but a simple case
compared to the MOTM votes, because a badge concerns **only its holder**. Nothing needs withholding.

- **Export** reports them under `badges` with the identifier, display name, timestamp and citing
  match.
- **Erasure leaves them untouched.** The table cascades from `players`, but erasure anonymises
  rather than deletes. A badge attached to `Deleted player #5` attributes nothing to anybody, and
  removing it would rewrite achievement history other players can see — the same reasoning that
  keeps `player_stats`.

All three places required by `PRIVACY_AND_DATA_PROTECTION.md` were updated: the data table, the
export and erasure paths in `PrivacyService`, and the proving cases in `PrivacyServiceTest`.

---

## Frontend Migration Notes

1. **Render from the response.** `displayName` is supplied; do not keep a local map of badge names,
   or a new badge will render as a raw enum until the frontend is redeployed.
2. **Switch on `badge`, not `displayName`** if you attach icons — the display name is copy and may
   be reworded.
3. **`matchId` may be absent.** Guard before linking to the match.
4. **Expect a burst on the first match after deployment.** Existing players are not backfilled, so
   everyone earns their whole history at once. A "new!" highlight based on `awardedAt` being recent
   would light up the entire profile that day.

---

## Breaking Changes

- [x] **None.** One new endpoint, one new table.

One additive change: `PersonalDataExportDTO` gained `badges`. Always present, empty rather than
absent.
