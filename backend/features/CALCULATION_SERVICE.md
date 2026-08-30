# Calculation Service — Skill Rating Engine

**Component:** `CalculationService.java`  
**Added in:** v1.0.0  
**Enhanced in:** v1.0.0 (2026-05-23) — Scarcity, Decisiveness & Dynamic Learning Rate  
**Rating Model v2:** (unreleased) — Goal-type weights, goal-timing impact, RAW score & proportional stats-dependent ceiling  
**Rating Model v2.1:** (unreleased) — Realistic 6.0-base distribution via compressed mapping [4.0, ceiling]  
**Rating Model v2.2:** — Escalating contribution ladder; goal/assist value climbs with each repeat  
**Rating Model v3:** (unreleased, branch `feat/rating-v3-absolute-curve`) — Absolute saturating curve replaces proportional normalization; collective team-defence term; true 10s reachable  
**Idempotent recalculation:** — reverse-then-reapply; manually triggerable via the admin recalculation endpoints  
**Status:** ✅ Released (v3 pending)

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

## Why Rating Model v2.2?

v2.1 solved the distribution but left the per-goal values too generous and, more importantly,
**linear**. A solo goal was worth a flat `3.0` against a base of `7.5`, so a single goal was nearly
enough to top the match on its own, and a brace was worth exactly two singles — the second goal
earned no recognition beyond the arithmetic. Scoring twice is not twice as good as scoring once.

**Rating Model v2.2** (current) replaces the flat per-type points with an **escalating ladder**
(§1). The first goal is deliberately cheap and each repeat is worth more than the one before, so
value now comes from *volume* rather than from a single strike. The goal-type ordering survives as
a variation on top of the rung: a penalty is a third of a goal, an assisted goal is `0.20` below a
solo one.

**Net effect (v2.2):**
- 1-goal top scorer → **8.30** (was 8.50, on a much flatter curve)
- Brace as the best performance → **8.90** (was 9.00, but now three singles' worth of ladder)
- Hat-trick → **9.50** (unchanged — the ceiling saturates)
- 1 goal + 1 assist → **8.47** (was 8.75)
- Non-contributors drift **up ~0.2-0.3**, an unavoidable consequence of the ratio normalization
  — see "The v2.2 passenger drift" in §4, and `STAT_POINTS_GAIN` as the dial for it.

## Why Rating Model v3?

Owner testing in a real amateur league (2026-08-31) exposed two structural failures of the
`raw / topRaw` proportional mapping — both consequences of rating players *relative to the
match's best performer* rather than against an absolute standard:

1. **Every dominant haul flattened onto exactly 9.5.** The ceiling saturates at
   `DOMINANCE_FULL_POINTS = 7.0` stat points — roughly three solo goals — and the top
   performer's ratio is `1.0` by definition. A hat-trick, a five-goal night and a seven-goal
   night were therefore **indistinguishable**, all printing the `CEILING_MAX` lid of 9.5.
2. **A player's rating depended on somebody else's night.** The same performance rated
   differently depending on how good the best player happened to be — and in a goal-heavy
   league (this league's normal), only ties at the top could produce two high ratings.

**Rating Model v3** keeps the entire points engine — ladder (§1), timing (§2), scarcity,
decisiveness, win/loss influence — and replaces only the mapping (§4):

1. **Absolute saturating curve.** `points` map directly to a rating through
   `BASE ± SPAN × (1 − e^(−|points|/SCALE))`. Nobody else's score is in the formula, the
   two-pass normalization collapses to a single pass, and several players can rate 9+ in the
   same goal-fest — a feature of the league, not an anomaly to normalize away.
2. **True 10s are back, and earned.** `UP_SPAN = 4.2` overshoots the 10.0 clamp so ≈16 points
   — five solo goals and change on a win — prints an actual 10. The v2.1 "no perfect 10s"
   rule is deliberately repealed.
3. **Collective team-defence term (§3b).** Computed from goals conceded and shared by the
   whole team: a clean sheet pays +0.8, fading to zero at three conceded, then a capped drag.
   This is what finally separates a passenger in a 2-0 win from a passenger in a 7-6 win.
   Deliberately **not** position-weighted: declared positions are unverifiable and gameable,
   and the scarcity multipliers already reward the rare scorer from observed behaviour.
4. **The transient and final ratings now share one scale.** `computeMatchRating(...)` runs the
   same curve, so the completion response no longer disagrees wildly with the persisted value
   (timing and scarcity refinements still apply only on the authoritative pass).

**Net effect (v3)** — lone scorer on a win, flat path, neutral scarcity:

| Performance | v2.2 | v3 |
|---|---|---|
| Passenger, 0-0 draw | ~8.0¹ | **6.59** |
| Passenger, 2-0 win | ~6.6 | **6.97** |
| Passenger, 7-6 win | ~6.6 | **~6.2** |
| Passenger, 0-2 loss | ~5.7 | **4.89** |
| 1 goal | 8.30 | **8.24** |
| Brace | 8.90 | **9.04** |
| Hat-trick | 9.50 | **9.54** |
| 4 goals | 9.50 | **9.84** |
| 5 goals | 9.50 | **10.00** |
| Own-goal catastrophe | 1.0² | **→ 4.0 asymptote** |

¹ v2's lone-passenger-on-a-draw mapped to the minimum ceiling (8.0) — itself an artefact of
the ratio being 1.0 for any sole player. ² v2's "all RAW non-positive" special case sent
everyone to 1.0; the curve needs no special case.

**Candidate v3.1 (deferred):** pace-aware timing — weighting the defence term by how long a
team held out between conceded goals, and the attack by scoring droughts. Blocked on a
denominator: match duration is not recorded, so "held out for 40 minutes" cannot be
normalized. Options if wanted: record kickoff/final-whistle, or assume nominal duration per
match type. Per-goal *impact* timing (late-game, go-ahead, equalizer) is already in v2.2+.

---

## Trigger: When Does Calculation Run?

### On Match Completion

When `PATCH /api/matches/{id}/complete` is called:

1. `MatchService.completeMatch()` validates goal compliance, sets `matchResult` on every
   `PlayerStat`, then calls `CalculationService.computeMatchRating()` synchronously to build
   the transient completion response.
2. `MatchService` publishes a `MatchCompletedEvent` (Spring Application Event).
3. `MatchEventListener` picks up the event and calls
   `CalculationService.recalculateMatchRatings(matchId)`, which performs the **authoritative
   calculation** (timing- and scarcity-aware points through the absolute curve), updates player
   `skillRating`, streaks, and career aggregates, and writes the history trail.

> 🔑 **Transient vs. authoritative — read this.** Since v3 the public
> `computeMatchRating(...)` overloads run the **same absolute curve** as the persisted path,
> so the transient completion response and the final rating share one scale. The
> **authoritative** value persisted on `player_stats.rating` is still the one written by
> `recalculateMatchRatings` once the match transaction commits — only that pass sees goal
> timing and scarcity — and where the two differ, it wins.

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
(never reversed arithmetically). **Reapply phase** then runs the existing points → curve → EMA →
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

## 1. The Escalating Contribution Ladder (v2.2)

**A goal is not worth a flat amount.** Until v2.1 every solo goal was worth `3.0`, which had two
consequences nobody wanted: one goal was nearly enough to top the match by itself, and a brace was
worth exactly two singles — scoring twice earned no recognition beyond the arithmetic.

v2.2 replaces the flat per-type points with a **ladder**. Each goal a player scores *in the same
match* takes the next rung, and the rungs climb: the first goal is deliberately cheap, and the
repeat is what pays.

| Rung | Goal | Running total | Assist | Running total |
|------|------|---------------|--------|---------------|
| 1st  | `0.70` | `0.70` | `0.40` | `0.40` |
| 2nd  | `1.40` | `2.10` | `0.80` | `1.20` |
| 3rd  | `1.50` | `3.60` | `0.85` | `2.05` |
| 4th  | `1.60` | `5.20` | `0.90` | `2.95` |
| 5th  | `1.70` | `6.90` | `0.95` | `3.90` |

So a brace is worth **three** singles, not two, and a hat-trick is worth **five**.

| Constant              | Value  | Meaning                                        |
|-----------------------|--------|------------------------------------------------|
| `GOAL_LADDER_FIRST`   | `0.70` | A player's first goal of the match             |
| `GOAL_LADDER_SECOND`  | `1.40` | The second                                     |
| `GOAL_LADDER_STEP`    | `0.10` | Added per goal beyond the second               |
| `ASSIST_LADDER_FIRST` | `0.40` | A player's first assist of the match           |
| `ASSIST_LADDER_SECOND`| `0.80` | The second                                     |
| `ASSIST_LADDER_STEP`  | `0.05` | Added per assist beyond the second             |

### Goal-type variations

The rung says *how many*. How the goal was scored then varies its value:

| Contribution  | Constant                   | Value at rung `v`      | 1st-goal value | Rationale                        |
|---------------|----------------------------|------------------------|----------------|----------------------------------|
| Solo goal     | —                          | `v`                    | `0.700`        | Unassisted — hardest, top reward |
| Assisted goal | `ASSISTED_GOAL_DISCOUNT`   | `v − 0.20`             | `0.500`        | A team-mate created it           |
| Penalty goal  | `PENALTY_GOAL_FRACTION`    | `v ÷ 3`                | `0.233`        | Easiest — a third of a goal      |
| Own goal      | `OWN_GOAL_PENALTY_POINTS`  | `−0.50` (never a rung) | `−0.500`       | Subtracted as a penalty          |

> Ordering at any rung: **SOLO > ASSISTED > PENALTY**, preserved from v2.1.

**Own goals are never rewarded** and never consume a ladder rung — a player who scores an own goal
and then a real goal still gets the first rung for the real one. An own goal credits the
*opponent's* scoreboard and subtracts `0.50` from the scorer's RAW score.

### Ladder units vs points units

The table above is authored on a human-readable scale where "a goal is worth 0.7". The points
total lives on a different scale, anchored by `RAW_LOSS_PENALTY` (2.2) and the curve's
`UP_SCALE` (5.3), so a single constant converts between them:

| Constant           | Value | Meaning                                     |
|--------------------|-------|----------------------------------------------|
| `STAT_POINTS_GAIN` | `2.0` | Multiplies ladder points into points units   |

Keeping the two scales separate means the ladder can be retuned without touching the
result-influence tuning, and vice versa. **`STAT_POINTS_GAIN` is also the dial for how far apart
contributors and passengers sit** — see the note at the end of §4.

**Ladder position is about volume only.** *When* a goal arrived and how much it mattered are priced
separately, by the timing impact in §2 and by decisiveness further down.

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

scorerPoints += typedGoalValue(goalLadderValue(nth goal by THIS player)) × STAT_POINTS_GAIN × impact
assistPoints += assistLadderValue(nth assist by THIS player) × STAT_POINTS_GAIN
                × (1 + (impact − 1) × ASSIST_TIMING_SHARE)                 // assister gets 50% of the uplift
```

Walking the goals in time order is also what fixes each player's **ladder position**: their nth
goal of the match takes the nth rung. The counters are tallied per player, so two team-mates who
score one goal each both sit on the first rung, however late they scored.

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

Older or manually-entered matches may have **no `Goal` rows** — and, since 2.2.0, matches that have
*some* but not all of them. In either case the timing model is skipped entirely and the service
falls back to the `PlayerStat` counters (`soloGoals`,
`assistedGoals`, `penaltyGoals`, `assists`, `ownGoals`) — with the penalty goal getting only
**half** the goal-scarcity bonus. Nothing breaks; the match is still rated. The public
`computeMatchRating` / `computePreviewMatchRating` overloads always use this flat path.

The counters say how many goals of each type a player scored but **not in which order**, so the
ladder cannot be walked rung by rung here. Every goal is instead priced at the player's **mean
rung** for that goal count:

```
meanRung = goalLadderSum(totalGoals) / totalGoals      // 2 goals → 2.10/2 = 1.05
```

This is order-independent — the fallback cannot invent an ordering it does not have — and it agrees
exactly with the timed path whenever all of a player's goals are the same type. Assists carry no
type variation, so their exact ladder sum is used.

### Partial goal events fall back too, and that is the whole point (2.2.0)

⚠️ **Cut into 2.2.0 on `next`, not deployed.**

**Timing weights are all-or-nothing.** When any `Goal` row exists with a scorer, `hasGoalData` is
true and a player's stat points come **only** from `TimingContribution` — the `PlayerStat` counters
are not read at all beyond own goals. So a match with three goal events and five counted goals did
not rate three players precisely and two approximately. It rated the two who scored the missing
goals at **zero stat points**, as though they had stood still all match.

The guard is therefore: use timing **only when the events account for every counted goal**.

That branch was unreachable for as long as nothing wrote `Goal` rows, which was every release up to
this one. [`POST /api/matches/{id}/events`](MATCH_FEATURE.md#goals-as-events-220) is what makes it
reachable, and **partial capture is the ordinary case there, not the failure case** — a flat
battery, a missed tap, a goal nobody saw go in. The counters are the number everybody agrees on,
because somebody sat down and totted them up; the events are a bonus when they happen to be
complete.

**So the fallback is deliberately to the *less precise* model**, which is the part worth
understanding before anybody "fixes" it. Flat weights lose the go-ahead bonus and the late-game
uplift: a rounding difference. Timing weights on partial data lose whole players: a wrong answer.

**Own goals count on both sides of the comparison** — they produce a `Goal` row like any other and
are counted in `PlayerStat.ownGoals` — even though they are attributed no timing points.
Miscounting them either way would silently disable timing on every match containing one, which is
why two of the three tests covering this are about own goals.

Logged at `WARN` when it fires. This is the difference between two rating models, and somebody
asking why a scorer rated badly needs to be able to find it.

---

## 3. Per-Player Points Total (v3 — unbounded, zero-centred)

```
statPoints = (timing-weighted ladder points  OR  mean-rung ladder points)
           − ownGoals × OWN_GOAL_PENALTY_POINTS × STAT_POINTS_GAIN

points = statPoints
       + min(decisiveness, DECISIVENESS_CAP) × DECISIVENESS_WEIGHT
       + RAW_WIN_BONUS         (+0.4, if WIN)
       − RAW_LOSS_PENALTY      (−2.2, if LOSS)   (0 for DRAW)
       ± goalDiffBonus         (min(|goalDiff| × 0.10, 0.50))  (+ if winning/drawing, − if losing)
       + teamDefenseTerm       (see §3b)
```

**Zero points is the reference performance: a passenger on a goalless draw** (before the
defence term, which lifts that exact case by its clean sheet). v2's `RAW_BASE_POINTS = 7.5`
is gone — it existed only to anchor the ratio mapping. There is intentionally **no upper
roof** on `points`.

| Constant               | Value | Description                                              |
|------------------------|-------|----------------------------------------------------------|
| `DECISIVENESS_WEIGHT`  | `1.0` | Scales the (capped) decisiveness into the points total   |
| `DECISIVENESS_CAP`     | `1.5` | Cap applied to decisiveness before scaling               |
| `RAW_WIN_BONUS`        | `0.4` | Added for every player on the winning team               |
| `RAW_LOSS_PENALTY`     | `2.2` | Subtracted for every player on the losing team           |
| `RAW_GOAL_DIFF_PER_GOAL`| `0.10`| Per goal of margin                                      |
| `RAW_MAX_GOAL_DIFF`    | `0.50`| Cap on the goal-difference nudge                         |

### 3b. Team-Defence Term (v3)

The attacking stats cannot see the defensive night: a passenger in a 2-0 win and a passenger
in a 7-6 win used to rate identically. Defence in amateur football is collective — no
per-player defensive stat exists — so the term is computed from the **team's goals conceded**
and shared equally by everyone on the team:

```
conceded ≤ 3:  +CLEAN_SHEET_BONUS × (1 − conceded / CLEAN_SHEET_FADE_GOALS)
conceded > 3:  −min((conceded − 3) × CONCEDED_DRAG_PER_GOAL, CONCEDED_MAX_DRAG)
```

| Constant                 | Value | Description                                          |
|--------------------------|-------|------------------------------------------------------|
| `CLEAN_SHEET_BONUS`      | `0.8` | Shared by the whole team at 0 conceded               |
| `CLEAN_SHEET_FADE_GOALS` | `3.0` | Conceded count at which the bonus reaches zero       |
| `CONCEDED_DRAG_PER_GOAL` | `0.1` | Drag per conceded goal beyond the fade point         |
| `CONCEDED_MAX_DRAG`      | `0.5` | Cap on the drag                                      |

Deliberately **not** weighted by declared favourite positions: those are self-reported,
unverifiable and gameable. The scarcity multipliers already reward the player who rarely
scores — from what they actually do rather than what their profile claims. When the match
scoreboard is unknown (null scores) the term stays neutral.

---

## 4. Absolute Saturating Curve (v3 — replaces proportional normalization)

The points total maps **directly** to a rating; nobody else's night is in the formula:

```
points ≥ 0:  rating = BASE_RATING + UP_SPAN   × (1 − e^(−points / UP_SCALE))
points < 0:  rating = BASE_RATING − DOWN_SPAN × (1 − e^(+points / DOWN_SCALE))
rating = clamp(rating, MIN_RATING, MAX_RATING)
```

| Constant      | Value  | Description                                                        |
|---------------|--------|--------------------------------------------------------------------|
| `BASE_RATING` | `6.0`  | The reference rating (passenger on a goalless draw, pre-defence)   |
| `UP_SPAN`     | `4.2`  | Overshoots the 10.0 clamp so ≈16 points prints a **true 10**       |
| `UP_SCALE`    | `5.3`  | Points at which ~63% of the upward span is earned                  |
| `DOWN_SPAN`   | `2.0`  | Floor asymptote at 6.0 − 2.0 = **4.0** (the v2.1 realistic floor)  |
| `DOWN_SCALE`  | `2.5`  | Negative points at which ~63% of the downward span is incurred     |
| `MIN_RATING`  | `1.0`  | Hard rating floor (unchanged)                                      |
| `MAX_RATING`  | `10.0` | Hard rating ceiling (unchanged)                                    |

Properties, each load-bearing:

- **Absolute** — a player's rating depends only on their own points, so several players can
  rate 9+ in the same goal-fest and a passenger's number never moves with a team-mate's haul.
- **Monotone** — within-match ordering always follows the points, exactly as v2 ordered raws.
- **Saturating** — mid-range nights stay well separated while the very top compresses; the
  down-curve approaches the 4.0 floor asymptotically, with **no special case** for v2's
  "all RAW non-positive" scenario.
- **Single-pass** — no `topRaw`, no ceiling, no match-wide first pass.

**Behavioral examples (v3)** — lone scorer on a 3-1 win, flat path, neutral scarcity
(decisiveness capped 1.5, +0.4 win, +0.2 gd, +0.533 defence for 1 conceded):

| Sole performer | Stat-points | Points | Rating |
|----------------|-------------|--------|--------|
| 1 solo goal    | `1.40`  | `4.03`  | **8.24** |
| 2 solo goals   | `4.20`  | `6.83`  | **9.04** |
| 3 solo goals   | `7.20`  | `9.83`  | **9.54** |
| 4 solo goals   | `10.40` | `13.03` | **9.84** |
| 5 solo goals   | `13.80` | `16.43` | **10.00** |
| 1 goal + 1 assist  | `2.20` | `4.83` | **8.51** |
| 1 goal + 3 assists | `5.50` | `8.13` | **9.29** |

Every haul is distinguishable — the v2 lid that flattened everything dominant onto exactly
9.5 is gone, and the five-goal night the league actually produces finally prints a 10.

---

## Transient vs. Authoritative — One Scale, One Truth

| Aspect            | Transient (public `computeMatchRating`) | Authoritative (`recalculateMatchRatings`)       |
|-------------------|------------------------------------------|-------------------------------------------------|
| When              | Synchronously, during completion         | Asynchronously, after the match commits (event) |
| Scope             | One player in isolation                  | Whole match, single pass                        |
| Curve             | Same absolute curve                      | Same absolute curve                             |
| Goal timing       | No (mean-rung ladder points)             | Yes (when `Goal` rows exist)                    |
| Scarcity          | Only if the caller passes multipliers    | Yes (pre-match career snapshot)                 |
| Persisted?        | No — transient response only             | **Yes** — written to `player_stats.rating`      |
| Authoritative?    | No                                       | **Yes** — this drives `skillRating`             |

> Since v3 the two share one scale and differ only by timing/scarcity refinements. Where they
> differ, the persisted value wins.

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

## Worked Examples (v3 — Absolute Curve)

> All examples assume **neutral scarcity** (`mult = 1.0`), the **flat** (mean-rung) path, and
> a 3-1 scoreboard unless stated. Every term the code applies is included — decisiveness,
> result, goal-diff, defence — and `CalculationServiceTest` pins Example A against the real
> implementation. There is no "top performer" special role any more: each player's rating is
> the curve over their own points.

### Example A — 3-1 Victory: four players, each rated on their own night

- Player 1: 3 goals + 2 assists │ Player 2: 1 goal │ Player 3: passenger (win) │ Player 4: passenger (loss)

**Player 1 (3g + 2a, WIN, conceded 1):**
```
Ladder: goals 0.70+1.40+1.50 = 3.60 ; assists 0.40+0.80 = 1.20  →  4.80 × 2.0 gain = 9.60
points = 9.60 + 1.175 (decis) + 0.4 (win) + 0.2 (gd) + 0.533 (defence) = 11.908
rating = 6.0 + 4.2 × (1 − e^(−11.908/5.3)) ≈ 9.76
```

**Player 2 (1 goal):**
```
points = 1.40 + 0.325 + 0.4 + 0.2 + 0.533 = 2.858 → rating ≈ 7.75
```

**Player 3 (passenger, WIN):**
```
points = 0.4 + 0.2 + 0.533 = 1.133 → rating ≈ 6.81
```

**Player 4 (passenger, LOSS, conceded 3 → no defence lift):**
```
points = −2.2 − 0.2 = −2.4 → rating = 6.0 − 2.0 × (1 − e^(−2.4/2.5)) ≈ 4.77
```

The ladder's gap survives (9.76 vs 7.75), the passenger quadrants separate (6.81 win vs 4.77
loss), and none of the four numbers would move if any other player's line changed.

### Example B — The curve's mid-range: 1 goal + 1 assist stays under 9

```
statPoints = (0.70 + 0.40) × 2.0 = 2.20 ; decisiveness 1.5 (sole contributor)
points = 2.20 + 1.5 + 0.4 + 0.2 + 0.533 = 4.833 → rating ≈ 8.51
```

### Example C — The ladder up the curve: a five-goal night prints a true 10

Lone scorer on a 3-1-style win (decisiveness capped, conceded 1):

| Goals | Points | Rating |
|-------|--------|--------|
| 1 | 4.03  | 8.24 |
| 2 | 6.83  | 9.04 |
| 3 | 9.83  | 9.54 |
| 4 | 13.03 | 9.84 |
| 5 | 16.43 | **10.00** |

Under v2 the last three rows were all exactly 9.5. `UP_SPAN = 4.2` overshoots the clamp
precisely so that last row can land.

---

## Edge Cases & Behavioral Notes

1. **A lone player rates from their own points, like everyone else.** v2's "sole player maps
   to the ceiling" artefact is gone; there is no ceiling.
2. **A 0-0 no-stats draw lands on ≈6.59, not 8.0.** Points = the clean sheet's `0.8` and
   nothing else — v2's `CEILING_MIN` artefact (a passenger printing 8.0 for a goalless draw)
   is gone.
3. **Goal-diff bonus cap is `0.50`** (unchanged from v2); the **own-goal penalty is `−0.50` ladder
   points** — `−1.0` on the RAW score after the gain (v2.2: rescaled from `−2.0`).
4. **No `Goal` rows** → graceful FLAT fallback (mean-rung ladder pricing); nothing breaks.
5. **All RAW scores non-positive** (`topRaw ≤ 0`) → proportional scaling undefined → everyone
   gets `MIN_RATING` (1.0). v2.2's smaller own-goal penalty means this now needs ~8+ own goals.
6. **Draws** apply neither WIN bonus nor LOSS penalty; `closeness = 1.0` (max) for decisiveness.
7. **The ladder is per player, per match.** It resets every match and is never shared between
   team-mates: two players who score one goal each both sit on the first rung.
8. **v2.2 realistic distributions:**
   - Non-contributor on **WIN**: ~6.6-7.3
   - Non-contributor on **LOSS**: ~5.7-6.2
   - Non-contributor on **DRAW**: ~6.5
   - 1-goal top scorer on **WIN**: ~8.3 (v2.1: 8.5)
   - Brace as the best performance: ~8.9 ; hat-trick: 9.5

---

## Preview Rating (Live Updates)

`computePreviewMatchRating()` is called during `PATCH /api/matches/{id}/stats/live`.

A match in progress has **no final scoreboard**, so the preview is **bonus-free**: no
WIN/LOSS influence, no goal-diff nudge, no scarcity, and **no match-wide normalization**. It
uses the same mean-rung ladder points as the flat path so the live indicator stays consistent:

```
previewRating = RAW_BASE_POINTS (7.5)
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
| `computeMatchRating(stat, goalDiff, teamTotalInv, teamGoals, goalScarcMult, assistScarcMult)`| `MatchService`/tests| 6-arg RAW value (mean-rung ladder, no timing), clamped `[1.0, 10.0]`     |
| `goalLadderValue(nth)` / `assistLadderValue(nth)`                                           | internal            | Value of a player's nth goal / assist of the match                       |
| `goalLadderSum(count)` / `assistLadderSum(count)`                                           | internal            | Cumulative ladder total, used for mean-rung pricing in the flat path     |
| `typedGoalValue(rung, penalty, assisted)`                                                   | internal            | Applies the penalty / assisted variation to a rung                       |
| `computePreviewMatchRating(stat, teamTotalInv)`                                             | `MatchService`      | Bonus-free live preview (no WIN/LOSS, goal-diff, scarcity, normalization)|
| `endSeason(seasonId)`                                                                       | Admin action        | Season-end transition for all players                                    |

> The public `computeMatchRating` overloads return the **RAW** flat-clamped value used
> transiently by `MatchService`; the **normalized** value from `recalculateMatchRatings` is
> the persisted, authoritative rating.

---

## Testing Notes

- `CalculationServiceTest` — 78 tests. All use `@ExtendWith(MockitoExtension.class)`:
  - **Ladder tests** (v2.2): sole-scorer ceiling per goal count (1→8.30, 2→8.90, 3→9.50); the
    second goal / assist adding more than the first; the penalty ⅓ and assisted −0.20 variations
    isolated at a fixed rung; the ladder being per player, not per match.
  - **Goal-type & timing tests**: verify the type ordering and go-ahead / equalizer / late-game uplift with `Goal` rows present.
  - **FLAT fallback tests**: verify mean-rung pricing agrees with the timed path when no `Goal` rows exist.
  - **Normalization tests**: verify the stats-dependent ceiling, the 6.8→~9.18 fix, and the 1g+1a→8.47 cap.
  - **Edge-case tests**: single-player → ceiling; 0-0 no-stats draw → 8.0; all-non-positive RAW → 1.0 floor.
  - **DynamicLearningRateTests**: extreme performances shift rating more than moderate ones.
  - **StreakUpdateTests**: WIN/LOSS/DRAW streak logic.
  - **EndSeasonTests**: season transition formula, activity adjustment, ±2.0 cap.
