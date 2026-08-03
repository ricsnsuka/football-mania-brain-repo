# Calculation Service — Skill Rating Engine

**Component:** `CalculationService.java`  
**Added in:** v1.0.0  
**Enhanced in:** v1.0.0 (2026-05-23) — Scarcity, Decisiveness & Dynamic Learning Rate  
**Rating Model v2:** (unreleased) — Goal-type weights, goal-timing impact, RAW score & proportional stats-dependent ceiling  
**Rating Model v2.1:** (unreleased) — Realistic 6.0-base distribution via compressed mapping [4.0, ceiling]  
**Idempotent recalculation:** (unreleased) — reverse-then-reapply; manually triggerable via the admin recalculation endpoints  
**Status:** ✅ Released

---

## Overview

`CalculationService` is the **sole source of all rating computations** in the Football
Management System. It is responsible for:

1. **Computing per-player match ratings** after a match is completed.
2. **Updating player `skillRating`** using an exponential moving average with a **dynamic
   learning rate** that reacts to the gap between performance and current rating.
3. **Updating win/loss/draw streaks** (`currentStreak`, `longestStreak`).
4. **Recording a `SkillRatingHistory` audit trail** for every change.
5. **Incrementing career aggregates** (`totalMatchesPlayed`, `totalGoals`, `totalAssists`).
6. **Applying season-end transitions** when an admin closes a season.

> ⚠️ **Clients never submit a rating value.** `PlayerStatUpdateDTO` has no `rating` field.
> All ratings are computed server-side by this service.

> 📊 **The system tracks only goals (solo / assisted / penalty), assists, and own goals.**
> No other per-player statistics (shots, passes, tackles, minutes, etc.) exist — every
> formula below is derived exclusively from those counters plus the match result and score.

---

## Why Rating Model v2? (And v2.1)

The original additive model started every player at a fixed `BASE_MATCH_RATING = 5.0` and
added small per-stat bonuses, then clamped to `[1.0, 10.0]`. In practice this **compressed
strong attacking displays**: a decisive **WIN with 1 goal + 3 assists previously scored only
~6.8**, barely above an average appearance, because the small additive bonuses could never
escape the mid-band.

**Rating Model v2** fixed this by separating two concerns:

1. An **unbounded RAW score** that accumulates goal-type-weighted, timing-aware points with
   no artificial ceiling — so a genuinely dominant performance produces a genuinely large
   number.
2. A **match-wide proportional normalization** against the best performer, using a **ceiling
   that scales with the absolute stat quality** of that top performance.

However, v2's `raw / topRaw × ceiling` mapping produced **unrealistic distributions**:
- 1-goal contributor in a 3-1 victory → **4.1** (too low)
- 3g+2a top performer → **10** (correct)
- Non-contributor on winning team → **1.8** (unrealistically harsh)

**Rating Model v2.1** (current) fixes the distribution problem via **compressed-range mapping**
and rebalanced constants:

1. **Elevated RAW_BASE_POINTS** (`1.0 → 7.5`) — anchors non-contributors near a realistic 6.0 base.
2. **Compressed normalization formula**: `RATING_FLOOR + (raw/topRaw) × (ceiling − RATING_FLOOR)`
   maps the proportional scale to **[4.0, ceiling]** instead of **[1.0, ceiling]**.
3. **Compressed ceiling band**: `[8.0, 9.5]` (was `[6.5, 10.0]`) — eliminates perfect 10s and
   extreme outliers.
4. **Smaller WIN_BONUS** (`1.0 → 0.4`), **larger LOSS_PENALTY** (`0.75 → 2.2`) — balanced
   relative to the new elevated base.

**Net effect (v2.1):**
- 1-goal contributor in a 3-1 victory → **~7.0** (was 4.1 — FIXED ✅)
- 3g+2a top performer → **~9.5** (was 10)
- Non-contributor on winning team → **~6.1-6.5** (was 1.8 — FIXED ✅)
- Non-contributor on losing team → **~5.0-5.5** (was <2.0 — FIXED ✅)
- Neutral non-contributor (draw) → **~6.0** (realistic baseline)

---

## Trigger: When Does Calculation Run?

### On Match Completion

When `PATCH /api/matches/{id}/complete` is called:

1. `MatchService.completeMatch()` validates goal compliance, sets `matchResult` on every
   `PlayerStat`, then calls `CalculationService.computeMatchRating()` synchronously to build
   the transient completion response.
2. `MatchService` publishes a `MatchCompletedEvent` (Spring Application Event).
3. `MatchEventListener` picks up the event and calls
   `CalculationService.recalculateMatchRatings(matchId)`, which performs the **authoritative,
   match-wide proportional normalization**, updates player `skillRating`, streaks, and career
   aggregates, and writes the history trail.

> 🔑 **RAW vs. normalized — read this.** The public `computeMatchRating(...)` overloads return
> the **RAW pre-normalization score** clamped to `[1.0, 10.0]`. This is used only transiently
> by `MatchService` for the immediate completion response. The **authoritative** value that is
> persisted on `player_stats.rating` and drives `skillRating` is the **normalized** value
> written by `recalculateMatchRatings` once the match transaction commits. Where the two
> differ, the normalized value wins. See [RAW vs. Normalized](#raw-vs-normalized--two-values-one-truth).

### On Season End

`CalculationService.endSeason(seasonId)` applies the season-end transition formula to
**all** players (active and inactive).

### On Manual Admin Recalculation

An administrator can re-run the engine on demand — **synchronously** — via two `ADMIN_USER`-only
endpoints:

- `POST /api/matches/{id}/recalculate` → recalculates one completed match.
- `POST /api/matches/recalculate` → bulk recalculation (by `matchIds`, by `seasonId`, or all
  completed matches), each match in its own transaction.

These call `recalculateSingleMatch(matchId)` directly (not via the completion event). Because
recalculation is now **idempotent** (see below), re-running never double-counts. See
[`MATCH_FEATURE.md`](./MATCH_FEATURE.md#rating-recalculation-admin) and
[`../api/MATCH-RATING-RECALCULATION-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md).

---

## Idempotent Recalculation — Reverse-then-Reapply

`recalculateMatchRatings(matchId)` (and its result-returning sibling
`recalculateSingleMatch(matchId)`) are **idempotent**. Before applying a match's effect they first
**reverse** any previously-recorded effect for that match, then re-apply the normal Pass-1/Pass-2
computation. On a match's first-ever calculation the reverse phase is a no-op (no history rows yet),
so the automatic completion path behaves exactly as before.

**Reverse phase** (for players with a prior `skill_rating_history` row for this match):
`skillRating -= changeAmount`; `totalMatchesPlayed -= 1`; `totalGoals`/`totalAssists` decremented by
this match's contribution; this match's history rows deleted; streaks recomputed from the chain
(never reversed arithmetically). **Reapply phase** then runs the existing normalization → EMA →
aggregate → history logic verbatim.

**Guarantees (run-twice = run-once):**

| Side effect | Guarantee |
|-------------|-----------|
| `PlayerStat.rating` | **Exact** every run |
| `totalGoals` / `totalAssists` / `totalMatchesPlayed` | **Exact** (net-zero for unchanged stats) |
| `skill_rating_history` | **No duplicates** (replace, not append) |
| Streaks | **Exact** (recomputed from the ordered chain) |
| `skillRating` (EMA) | **Exact for a player's most-recent match**; approximate for a mid-chain historical match |

> ⚠️ **Mid-chain caveat.** `skillRating` is a cumulative EMA over the player's matches in
> chronological order, so reversing a single mid-chain match cannot perfectly restore every later
> match's baseline. The single-match endpoint is exact for the **latest** match in a player's chain;
> to fully reconcile a season use the **bulk** endpoint, which replays affected matches chronologically
> (`matchDate ASC NULLS LAST, id ASC`) via `PlayerStatRepository.findCompletedByPlayerIdChronological`.
> No schema change was required.

---

## 1. Per-Goal-Type Points

Every goal is weighted by **difficulty / impact**, and assists are meaningfully rewarded:

| Contribution      | Constant                  | Points | Rationale                          |
|-------------------|---------------------------|--------|------------------------------------|
| Solo goal         | `SOLO_GOAL_POINTS`        | `3.0`  | Unassisted — hardest, top reward   |
| Assisted goal     | `ASSISTED_GOAL_POINTS`    | `2.0`  | Created by a team-mate             |
| Penalty goal      | `PENALTY_GOAL_POINTS`     | `1.0`  | Easiest — set-piece, lowest reward |
| Assist            | `ASSIST_POINTS`           | `1.5`  | Below a scored goal, above penalty |
| Own goal          | `OWN_GOAL_PENALTY_POINTS` | `−2.0` | Subtracted as a penalty            |

> Ordering: **SOLO (3.0) > ASSISTED (2.0) > PENALTY (1.0)**, with **ASSIST (1.5)** slotted
> between assisted and penalty goals.

**Own goals are never rewarded.** An own goal credits the *opponent's* scoreboard and applies
a `−2.0` point penalty to the scorer's RAW score. It contributes no positive timing points.

---

## 2. Goal-Timing Impact

When `Goal` rows exist for the match, `GoalRepository.findByMatchIdOrderByTiming(matchId)`
loads them ordered by **`minute` ASC (NULLS LAST)** then **`createdAt` ASC** (the stable
tie-breaker — goals are registered in time order). The service walks them in that order,
reconstructs the scoreboard, and applies a per-goal impact multiplier:

```
seqFactor = 1 + LATE_GAME_IMPACT_WEIGHT × (index / (goalCount − 1))   // later goals matter more
impact    = seqFactor
          + GO_AHEAD_GOAL_BONUS    (+0.40, if the goal put the scoring team ahead)
          + EQUALIZER_GOAL_BONUS   (+0.25, if the goal levelled the game)

scorerPoints += goalTypePoints × impact
assistPoints += ASSIST_POINTS × (1 + (impact − 1) × ASSIST_TIMING_SHARE)   // assister gets 50% of the uplift
```

| Constant                 | Value | Meaning                                                    |
|--------------------------|-------|------------------------------------------------------------|
| `LATE_GAME_IMPACT_WEIGHT`| `0.50`| Max uplift for the last goal relative to the first         |
| `GO_AHEAD_GOAL_BONUS`    | `0.40`| Extra impact for a goal that puts the team ahead           |
| `EQUALIZER_GOAL_BONUS`   | `0.25`| Extra impact for a goal that levels the game               |
| `ASSIST_TIMING_SHARE`    | `0.50`| Fraction of a goal's timing uplift that flows to the assister |

- `seqFactor` ranges from `1.0` (first goal) to `1.5` (last goal) across the match.
- The **assister receives 50% of the timing uplift** applied to the goal they created.
- **Own goals** are skipped for positive attribution — they only move the reconstructed
  scoreboard (crediting the opposing team) and are penalised on the scorer's stat record.

### Graceful FLAT Fallback

Older or manually-entered matches may have **no `Goal` rows**. In that case the timing model
is skipped entirely and the service falls back to **flat per-type weights** taken directly
from the `PlayerStat` counters (`soloGoals`, `assistedGoals`, `penaltyGoals`, `assists`,
`ownGoals`) — with the penalty goal getting only **half** the goal-scarcity bonus. Nothing
breaks; the match is still rated. The public `computeMatchRating` / `computePreviewMatchRating`
overloads always use this flat path.

---

## 3. RAW Per-Player Score (unbounded, pre-normalization)

```
rawStatPoints = (timing-weighted goal + assist points  OR  flat per-type points)
              − ownGoals × OWN_GOAL_PENALTY_POINTS

raw = RAW_BASE_POINTS                                          (7.5 — v2.1: elevated from 1.0)
    + rawStatPoints
    + min(decisiveness, DECISIVENESS_CAP) × DECISIVENESS_WEIGHT
    + RAW_WIN_BONUS         (+0.4, if WIN — v2.1: reduced from 1.0)
    − RAW_LOSS_PENALTY      (−2.2, if LOSS — v2.1: increased from 0.75)   (0 for DRAW)
    ± goalDiffBonus         (min(|goalDiff| × 0.10, 0.50))     (+ if winning/drawing, − if losing)
```

There is intentionally **no upper roof** on `raw`.

| Constant               | Value (v2.1) | Description                                              |
|------------------------|--------------|----------------------------------------------------------|
| `RAW_BASE_POINTS`      | `7.5`        | Elevated base — non-contributors anchor near 6.0 final   |
| `DECISIVENESS_WEIGHT`  | `1.0`        | Scales the (capped) decisiveness into the RAW score      |
| `DECISIVENESS_CAP`     | `1.5`        | Cap applied to decisiveness before scaling               |
| `RAW_WIN_BONUS`        | `0.4`        | Added for every player on the winning team               |
| `RAW_LOSS_PENALTY`     | `2.2`        | Subtracted for every player on the losing team           |
| `RAW_GOAL_DIFF_PER_GOAL`| `0.10`      | Per goal of margin                                       |
| `RAW_MAX_GOAL_DIFF`    | `0.50`       | Cap on the goal-difference nudge (unchanged from v1)     |

---

## 4. Proportional Normalization & Stats-Dependent Ceiling (v2.1 — Compressed Mapping)

After every player's RAW score is known (a match-wide first pass inside
`recalculateMatchRatings`), the **top RAW performer** defines the scale:

```
topRaw        = max(raw across all players)
topStatPoints = the stat-points of that same top performer

ceiling     = CEILING_MIN + (CEILING_MAX − CEILING_MIN)
                          × clamp(topStatPoints / DOMINANCE_FULL_POINTS, 0, 1)

finalRating = RATING_FLOOR + (raw / topRaw) × (ceiling − RATING_FLOOR)
            = 4.0 + (raw / topRaw) × (ceiling − 4.0)

finalRating = clamp(finalRating, MIN_RATING, MAX_RATING)
```

| Constant               | Value (v2.1) | Description                                                     |
|------------------------|--------------|-----------------------------------------------------------------|
| `RATING_FLOOR`         | `4.0`        | NEW — lower bound of compressed mapping (realistic floor)      |
| `CEILING_MIN`          | `8.0`        | Lowest possible ceiling (was 6.5 — v2.1: raised)                |
| `CEILING_MAX`          | `9.5`        | Highest possible ceiling (was 10.0 — v2.1: reduced)             |
| `DOMINANCE_FULL_POINTS`| `9.0`        | Stat-points at which the top performer reaches `CEILING_MAX`    |
| `MIN_RATING`           | `1.0`        | Hard rating floor (unchanged)                                   |
| `MAX_RATING`           | `10.0`       | Hard rating ceiling (unchanged)                                 |

**Key v2.1 changes:**

1. **Compressed mapping formula**: instead of `raw/topRaw × ceiling` (which maps `[0, topRaw]`
   to `[0, ceiling]`), v2.1 uses `RATING_FLOOR + (raw/topRaw) × (ceiling − RATING_FLOOR)`,
   which maps `[0, topRaw]` to **`[4.0, ceiling]`**.
2. **Compressed ceiling band**: `[8.0, 9.5]` instead of `[6.5, 10.0]` — eliminates perfect 10s
   and reduces extreme-score variance.
3. **Result:** the worst performer in a match lands on `RATING_FLOOR = 4.0` (instead of a
   proportionally-tiny value), and the top performer reaches the ceiling (now max 9.5, not 10.0).

**Behavioral examples (v2.1):**

- A top performer with **9.0 stat-points** (e.g., 3 solo goals) → ceiling = `8.0 + 1.5 × 1.0 = 9.5`
  → as the sole top performer: `finalRating = 4.0 + 1 × (9.5 − 4.0) = 9.5`.
- A top performer with **4.5 stat-points** (e.g., 1 goal + 1 assist) → ceiling = `8.0 + 1.5 × 0.5 = 8.75`
  → as the sole top performer: `finalRating = 4.0 + 1 × (8.75 − 4.0) = 8.75`.
- A **non-contributor on a WIN** (RAW ≈ 7.9 base + 0.4 bonus = 8.3; if topRaw = 15.0 → ratio ≈ 0.55)
  → `finalRating ≈ 4.0 + 0.55 × 5.5 ≈ 7.0` (realistic!).

### Guard: non-scalable matches

If **every** RAW score is non-positive (e.g. an all-own-goals or heavy-loss scenario where
`topRaw ≤ 0`), proportional scaling is undefined and **every player lands on `MIN_RATING`
(1.0)**.

---

## RAW vs. Normalized — Two Values, One Truth

| Aspect            | RAW (public `computeMatchRating`)      | Normalized (`recalculateMatchRatings`)          |
|-------------------|-----------------------------------------|-------------------------------------------------|
| When              | Synchronously, during completion        | Asynchronously, after the match commits (event) |
| Scope             | One player in isolation                 | Whole match, two-pass                           |
| Ceiling           | Flat clamp to `[1.0, 10.0]`             | Stats-dependent ceiling `[6.5, 10.0]`           |
| Goal timing       | No (flat per-type weights)              | Yes (when `Goal` rows exist)                    |
| Persisted?        | No — transient response only            | **Yes** — written to `player_stats.rating`      |
| Authoritative?    | No                                      | **Yes** — this drives `skillRating`             |

> For **isolated numeric reasoning** about a single player's stat line, reason about the
> **RAW** value. The persisted rating is always the normalized value.

---

## Match Decisiveness

> **Core idea:** A goal in a 1-0 win matters more to the result than the 5th goal in a 5-0
> rout. Decisiveness blends the player's share of the team's goals with their overall
> involvement, then amplifies it by how close the game was.

```
scoreShareRatio    = playerGoals / max(teamGoals, 1)                  [0.0 if team scored 0]
contributionRatio  = playerInvolvements / max(teamTotalInvolvements, 1)
closeness          = 1.0 / max(|goalDiff|, 1)                          [draws → 1.0]

decisiveness = (scoreShareRatio × SCORE_SHARE_WEIGHT
              + contributionRatio × CONTRIBUTION_RATIO_WEIGHT)
              × (1.0 + closeness)

decisiveness = min(decisiveness, DECISIVENESS_CAP)                     [capped at 1.5]
```

| Constant                  | Value | Fraction of decisiveness from…              |
|---------------------------|-------|---------------------------------------------|
| `SCORE_SHARE_WEIGHT`      | `0.60`| Score-share (player goals / team goals)     |
| `CONTRIBUTION_RATIO_WEIGHT`| `0.40`| Involvement ratio (goals + assists / team)  |

`playerInvolvements = soloGoals + assistedGoals + penaltyGoals + assists` (own goals are
**never** counted as involvements).

---

## Goal / Assist Scarcity Multipliers

> **Core idea:** A defensive player who rarely scores deserves more credit for a rare goal
> than a prolific striker's routine finish.

Scarcity is derived from each player's **career aggregates** (`totalGoals /
totalMatchesPlayed`, `totalAssists / totalMatchesPlayed` — columns on the `players` table)
compared against the group average. These aggregates are incremented **after** rating
computation, so scarcity always reflects the **pre-match** profile.

```
scarcityRatio      = clamp(1.0 − (playerPerMatch / (groupAvgPerMatch × 2.0)), 0.0, 1.0)
goalScarcityMult   = 1.0 + scarcityRatio × GOAL_SCARCITY_WEIGHT      ∈ [1.0, 1.40]
assistScarcityMult = 1.0 + scarcityRatio × ASSIST_SCARCITY_WEIGHT    ∈ [1.0, 1.25]
```

| Constant                | Value | Range        |
|-------------------------|-------|--------------|
| `GOAL_SCARCITY_WEIGHT`  | `0.40`| `[1.0, 1.40]`|
| `ASSIST_SCARCITY_WEIGHT`| `0.25`| `[1.0, 1.25]`|

- Player scores at **double the group average** → `mult = 1.0` (no bonus).
- Player **never scores** in a scoring group → `mult = 1.40` (max bonus).
- **No historical data** (new player) → `mult = 1.0`.
- Penalties receive only **half** the goal-scarcity bonus in the flat path
  (`penaltyScarcityMult = 1.0 + (goalScarcityMult − 1.0) × 0.5`).

---

## Worked Examples (v2.1 — Compressed Mapping)

> All examples below assume **neutral scarcity** (`mult = 1.0`) and treat the described
> player as the **top RAW performer** in the match. For a single-player (or clear top)
> performer, `raw / topRaw = 1`, so the final rating equals `RATING_FLOOR + (ceiling − RATING_FLOOR)`.

### Example A — 3-1 Victory: Rating Distribution Fix

Imagine a 3-1 victory with these contributions:
- Player 1: 3 goals + 2 assists (top performer)
- Player 2: 1 goal
- Player 3: 0 stats (non-contributor on winning team)

**Player 1 (top performer):**
```
Stat points (flat): 3 × 3.0 + 2 × 1.5 = 9.0 + 3.0 = 12.0
RAW = 7.5 (base) + 12.0 (stats) + 0.4 (win) = 19.9
topStatPoints = 12.0 → ceiling = 8.0 + 1.5 × (12.0 / 9.0) = 8.0 + 1.5 = 9.5 (clamped to 9.5)
finalRating = 4.0 + (19.9 / 19.9) × (9.5 − 4.0) = 4.0 + 5.5 = 9.5 ✅
```

**Player 2 (1 goal):**
```
Stat points (flat): 1 × 3.0 = 3.0
RAW = 7.5 + 3.0 + 0.4 = 10.9
finalRating = 4.0 + (10.9 / 19.9) × 5.5 = 4.0 + 0.548 × 5.5 ≈ 4.0 + 3.0 = 7.0 ✅
(v2 old: 4.1 — FIXED!)
```

**Player 3 (non-contributor, WIN):**
```
Stat points = 0
RAW = 7.5 + 0 + 0.4 = 7.9
finalRating = 4.0 + (7.9 / 19.9) × 5.5 = 4.0 + 0.397 × 5.5 ≈ 4.0 + 2.18 = 6.18 ✅
(v2 old: 1.8 — FIXED! Now > 6.0 as required)
```

**Non-contributor on losing team (hypothetical):**
```
RAW = 7.5 + 0 − 2.2 = 5.3
finalRating = 4.0 + (5.3 / 19.9) × 5.5 = 4.0 + 0.266 × 5.5 ≈ 4.0 + 1.46 = 5.46 ✅
(v2 old: <2.0 — FIXED! Now realistic ~5.0-5.5)
```

### Example B — Ceiling guard: 1 goal + 1 assist never reaches 10 (or even 9.5)

```
Stat points (flat): 1 solo goal × 3.0 + 1 assist × 1.5 = 3.0 + 1.5 = 4.5

topStatPoints = 4.5
ceiling = 8.0 + 1.5 × clamp(4.5 / 9.0, 0, 1)
        = 8.0 + 1.5 × 0.5
        = 8.0 + 0.75
        = 8.75

As the top performer: finalRating = 4.0 + 1 × (8.75 − 4.0) = 8.75
```

Even as the single best performer in the match, a 1 goal + 1 assist line **cannot reach 9.5** —
its absolute stat quality caps the ceiling at 8.75.

### Example C — A dominant display approaches CEILING_MAX

```
Stat points (flat): 3 solo goals × 3.0 = 9.0   (≥ DOMINANCE_FULL_POINTS)

topStatPoints = 9.0
ceiling = 8.0 + 1.5 × clamp(9.0 / 9.0, 0, 1) = 8.0 + 1.5 = 9.5

As the top performer: finalRating = 4.0 + 1 × (9.5 − 4.0) = 9.5
```

---

## Edge Cases & Behavioral Notes

1. **Single-player matches always normalize to the compressed ceiling.** With one player (or a clear
   solo top performer) `raw / topRaw = 1`, so `finalRating = RATING_FLOOR + (ceiling − RATING_FLOOR)`.
   Use the **RAW** value for isolated numeric reasoning.
2. **A 0-0 no-stats draw normalizes to `8.0` (`CEILING_MIN`), not 6.5 or 5.0.** With no stat points,
   `topStatPoints = 0` → `ceiling = 8.0`, and every RAW is equal → `raw/topRaw = 1` → all players
   land on `4.0 + (8.0 − 4.0) = 8.0` (v2.1 compressed mapping).
3. **Goal-diff bonus cap is `0.50`** (unchanged from v2); the **own-goal penalty is `−2.0` points**
   on the RAW score (unchanged).
4. **No `Goal` rows** → graceful FLAT fallback (per-type `PlayerStat` weights); nothing breaks.
5. **All RAW scores non-positive** (`topRaw ≤ 0`) → proportional scaling undefined → everyone
   gets `MIN_RATING` (1.0).
6. **Draws** apply neither WIN bonus nor LOSS penalty; `closeness = 1.0` (max) for decisiveness.
7. **v2.1 realistic distributions:**
   - Non-contributor on **WIN**: ~6.1-6.5 (> 6.0 ✅)
   - Non-contributor on **LOSS**: ~5.0-5.5 (realistic)
   - Non-contributor on **DRAW**: ~6.0 (neutral baseline)
   - 1-goal contributor on **WIN**: ~7.0-7.5 (was 4.1 — FIXED ✅)

---

## Preview Rating (Live Updates)

`computePreviewMatchRating()` is called during `PATCH /api/matches/{id}/stats/live`.

A match in progress has **no final scoreboard**, so the preview is **bonus-free**: no
WIN/LOSS influence, no goal-diff nudge, no scarcity, and **no match-wide normalization**. It
uses the same flat per-type points as the full model so the live indicator stays consistent:

```
previewRating = RAW_BASE_POINTS (1.0)
              + flatStatPoints(stat, scarcity = 1.0, 1.0)
              + contributionRatio × DECISIVENESS_WEIGHT
```

Clamped to `[1.0, 10.0]`. Preview ratings differ from the final rating because they exclude
goal timing, the result bonus, the goal-diff nudge, scarcity, and the proportional ceiling.

---

## Skill Rating Update — Dynamic Learning Rate

> **Goal:** Great performances should accelerate a player's growth; mediocre performances
> should not anchor an elite player's rating.

The persistent `skillRating` is updated via an exponential moving average (EMA) whose
**learning rate scales with the gap** between the (normalized) match rating and current rating:

```
performanceDelta = |matchRating − currentSkillRating|

dynamicRate = clamp(
                BASE_LEARNING_RATE + (performanceDelta / (MAX_RATING − MIN_RATING)) × PERFORMANCE_RATE_BOOST,
                BASE_LEARNING_RATE, MAX_LEARNING_RATE
              )

newSkillRating = clamp(
                   currentSkillRating × (1 − dynamicRate) + matchRating × dynamicRate,
                   MIN_RATING, MAX_RATING
                 )
```

| Constant                | Value | Description                                  |
|-------------------------|-------|----------------------------------------------|
| `BASE_LEARNING_RATE`    | `0.20`| Minimum rate applied to every match          |
| `PERFORMANCE_RATE_BOOST`| `0.15`| Max additional rate for extreme performances |
| `MAX_LEARNING_RATE`     | `0.35`| Hard ceiling on the dynamic rate             |

### Worked Example

Player with `skillRating = 5.0` earns a normalized `matchRating = 9.5`:

```
delta       = |9.5 − 5.0| = 4.5
dynamicRate = clamp(0.20 + (4.5/9) × 0.15, 0.20, 0.35) = 0.275
newSkillRating = 5.0 × 0.725 + 9.5 × 0.275 = 3.625 + 2.6125 ≈ 6.24
```

---

## Player Aggregate Tracking

After computing each player's match rating (pass 2), `recalculateMatchRatings` increments the
`Player` career aggregates: `totalGoals += playerGoals`, `totalAssists += assists`,
`totalMatchesPlayed += 1`. The increment happens **after** scarcity is computed so the
multipliers always reflect the **pre-match** profile.

---

## Streak Rules

| Result | Rule                                                             |
|--------|------------------------------------------------------------------|
| WIN    | If `currentStreak >= 0` → `currentStreak++`. Else → set to `1`   |
| LOSS   | If `currentStreak <= 0` → `currentStreak--`. Else → set to `-1`  |
| DRAW   | `currentStreak = 0`                                              |

`longestStreak` is updated if the new positive streak exceeds the historical peak.

---

## Season-End Transition Formula

When `endSeason(seasonId)` is called, ALL players (active and inactive) receive a new skill
rating:

```
newRating = (endRating   × 0.45)
          + (avgRating   × 0.25)
          + (startRating × 0.10)
          + (meanRating  × 0.20)
          + activityAdjustment
```

Where:
- `endRating` = player's current `skillRating` at season end
- `avgRating` = average `ratingAfter` across this player's season history (falls back to `endRating`)
- `startRating` = `ratingBefore` of the first history entry (falls back to `endRating`)
- `meanRating` = mean `skillRating` across ALL players (regression-to-mean anchor)
- `activityAdjustment` = `clamp((participationRate − 0.5) × ACTIVITY_WEIGHT, −0.25, +0.25)`

### Season Bounds

| Constraint            | Value |
|-----------------------|-------|
| Max change per season | ±2.0  |
| Min final rating      | 1.0   |
| Max final rating      | 10.0  |

---

## Skill Rating History

Every rating change is persisted to `skill_rating_history`:

| Column          | Type          | Notes                                                   |
|-----------------|---------------|---------------------------------------------------------|
| `id`            | BIGSERIAL     |                                                         |
| `player_id`     | BIGINT (FK)   | Player whose rating changed                             |
| `match_id`      | BIGINT (FK)   | Null for season-end transitions                         |
| `rating_before` | NUMERIC(5,2)  |                                                         |
| `rating_after`  | NUMERIC(5,2)  |                                                         |
| `change_amount` | NUMERIC(5,2)  | `ratingAfter − ratingBefore`                            |
| `reason`        | VARCHAR(255)  | e.g. `"Match id=5 (WIN)"` or `"Season end: Season 1"`   |
| `created_at`    | TIMESTAMPTZ   |                                                         |

---

## Implementation Details

- **Service:** `CalculationService.java` — all constants `private static final`
- **New repository:** `GoalRepository.findByMatchIdOrderByTiming(matchId)` — loads a match's
  goals in scoresheet order (`minute` ASC NULLS LAST, then `createdAt` ASC), eagerly fetching
  `scorer`, `assister`, and the scoring `matchTeam` for scoreboard reconstruction.
- **Recalculation finders (idempotent recalc):** `MatchRepository.findCompletedOrdered` /
  `findCompletedBySeasonOrdered` (chronological `matchDate ASC NULLS LAST, id ASC`) and
  `PlayerStatRepository.findCompletedByPlayerIdChronological(playerId)`.
- **Internal type:** `TimingContribution` record — per-player scorer/assist point maps built
  by walking the match goals; `hasData()` gates the timing model vs. the flat fallback.
- **Event:** `MatchCompletedEvent.java` — Spring Application Event carrying `matchId`.
- **Listener:** `MatchEventListener.java` — calls `recalculateMatchRatings(matchId)`.
- **History:** `SkillRatingHistory.java` entity + `SkillRatingHistoryRepository.java`.

### Key Methods

| Method                                                                                      | Called By           | Description                                                              |
|---------------------------------------------------------------------------------------------|---------------------|--------------------------------------------------------------------------|
| `recalculateMatchRatings(matchId)`                                                          | `MatchEventListener`| **Authoritative** idempotent two-pass normalization + EMA + streaks + aggregates + history (reverse-then-reapply) |
| `recalculateSingleMatch(matchId)`                                                           | `MatchService` (admin endpoints) | Result-returning idempotent recalc → `RecalculationResultDTO` (`SUCCESS`/`SKIPPED`; `409` if not completed, `404` if missing) |
| `computeMatchRating(stat, goalDiff, teamTotalInv)`                                          | `MatchService`      | 3-arg compat overload → RAW value, neutral scarcity                      |
| `computeMatchRating(stat, goalDiff, teamTotalInv, teamGoals, goalScarcMult, assistScarcMult)`| `MatchService`/tests| 6-arg RAW value (flat weights, no timing), clamped `[1.0, 10.0]`         |
| `computePreviewMatchRating(stat, teamTotalInv)`                                             | `MatchService`      | Bonus-free live preview (no WIN/LOSS, goal-diff, scarcity, normalization)|
| `endSeason(seasonId)`                                                                       | Admin action        | Season-end transition for all players                                    |

> The public `computeMatchRating` overloads return the **RAW** flat-clamped value used
> transiently by `MatchService`; the **normalized** value from `recalculateMatchRatings` is
> the persisted, authoritative rating.

---

## Testing Notes

- `CalculationServiceTest` — 62 tests, ~89% branch coverage. All use
  `@ExtendWith(MockitoExtension.class)`:
  - **Goal-type & timing tests**: verify per-type weights and go-ahead / equalizer / late-game uplift with `Goal` rows present.
  - **FLAT fallback tests**: verify identical behavior when no `Goal` rows exist.
  - **Normalization tests**: verify the stats-dependent ceiling, the 6.8→~9.4 fix, and the 1g+1a→~8.25 cap.
  - **Edge-case tests**: single-player → ceiling; 0-0 no-stats draw → 6.5; all-non-positive RAW → 1.0 floor.
  - **DynamicLearningRateTests**: extreme performances shift rating more than moderate ones.
  - **StreakUpdateTests**: WIN/LOSS/DRAW streak logic.
  - **EndSeasonTests**: season transition formula, activity adjustment, ±2.0 cap.
