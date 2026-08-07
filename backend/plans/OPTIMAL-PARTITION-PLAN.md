# `OPTIMAL` team generation — exact partition search with a pluggable objective

> **Status:** 📋 **Specified, not built.** No code exists. Written 2026-08-07.
> **Repo:** backend (`FootMania-Back`) — one new strategy class, no migration. Frontend follows.
> **Supersedes nothing.** Ships alongside the six existing strategies; none are removed.
> **Related:** [TEAM_GENERATION_DESIGN.md](../features/TEAM_GENERATION_DESIGN.md) (the original
> six), [TEAM_GENERATION_FEATURE.md](../features/TEAM_GENERATION_FEATURE.md) (as built).

---

## 1. Why this exists

All six existing strategies are one algorithm in six costumes: **reduce each player to a scalar,
sort, split.** `BALANCED` and `FORM_BASED` are the identical greedy loop over a different scalar;
`SNAKE_DRAFT` and `CAPTAIN_PICK` are the identical snake over a different seed.

That shape has a hard ceiling, and it is *not* the one you would guess.

**The ceiling is not accuracy.** Greedy bin-packing on sorted input is already excellent at
minimising the mean gap — [TEAM_GENERATION_DESIGN.md §7.1](../features/TEAM_GENERATION_DESIGN.md)
proves a `≤ P_max/(n+1)` bound, and hand-checking six adversarial 6-player pools against exhaustive
search found greedy hitting the exact optimum every time. **Replacing greedy with exact search to
get better mean balance would be a waste of effort.**

**The ceiling is expressiveness.** Greedy's correctness argument depends on the quantity being
minimised being an additive sum over players. The moment the objective involves anything else —
the *shape* of a team, who played together last week, a keeper on each side — greedy has no
justification and no mechanism. You cannot bolt those onto a sort.

Exhaustive search has no such constraint, and at these pool sizes it is free:

| Format | Pool | Distinct splits¹ | Search cost² |
|---|---:|---:|---|
| `FIVE_A_SIDE` | 10 | **126** | microseconds |
| `SEVEN_A_SIDE` | 14 | **1,716** | microseconds |
| `ELEVEN_A_SIDE` | 22 | **352,716** | ~5–15 ms |

¹ `C(n−1, n/2−1)` — fixing the lowest-ID player to Team A removes the mirror-image duplicate.
² Single-threaded, ~n integer ops per candidate. Pure CPU, no I/O.

So the deliverable is not "a better balancer". It is **one class that turns team generation from a
sorting problem into a scoring problem**, after which shape balance, rotation, hard constraints and
streak separation are each *a term in a function* rather than a new strategy class.

---

## 2. What v1 ships

`OPTIMAL` — enumerate every legal split, score each with

```
score = |Δmean| + λ · |Δspread|
```

and keep the minimum. Two terms, one tunable. That is the whole algorithm.

### 2.1 Why the spread term is the point

Today's `Δ` is blind to team *shape*:

```
Team A: [9.0, 9.0, 9.0, 4.0, 4.0]   mean 7.00   spread 2.45
Team B: [7.0, 7.0, 7.0, 7.0, 7.0]   mean 7.00   spread 0.00
                                     Δmean = 0.00  ✅ "perfectly balanced"
```

`BalanceGauge` renders that green. Three strong players beat five average ones in 5-a-side most
weeks, and the group knows it — this is the single most common "the algorithm is rubbish" complaint
a balanced-teams feature gets.

`|Δspread|` names the defect directly. Note it targets the *difference* in shape, not low shape:
two equally top-heavy teams are a fair match and score 0, correctly.

### 2.2 Why λ mostly acts as a tie-break

With 126–352,716 candidates and ratings on a 1–10 scale at two decimals, **many splits tie on mean
to within a hundredth of a point.** The mean term picks that near-optimal set; the spread term
chooses within it. λ therefore buys shape balance at almost no cost in mean balance — which is
exactly why exact search is required. Greedy returns *one* answer and cannot choose among equals.

**Default λ = 0.5.** A 0.1 worsening in mean balance must buy a 0.2 improvement in shape.
Recommended range `[0, 5]`, clamped. `λ = 0` reproduces pure mean optimisation.

### 2.3 The metric is a parameter, not a strategy

The objective consumes one scalar per player. Which scalar is orthogonal:

| `metric` | Scalar | Equivalent to |
|---|---|---|
| `skill` *(default)* | `Player.skillRating` | `BALANCED`, but exact and shape-aware |
| `form` | linearly-weighted last-N rating | `FORM_BASED`, but exact and shape-aware |

`form` reuses `FormBasedGenerationStrategy.computeFormScore()` verbatim, including its fallback to
`skillRating` for players with no rated history. **Extract that method to a shared
`PlayerMetrics` component** rather than duplicating or subclassing — two copies of a scoring rule
drift, and this one already has locked semantics and tests.

`FORM_BASED` and `BALANCED` stay. Existing `matches` rows reference them, and re-running a strategy
that no longer exists must never become possible.

---

## 3. Determinism — the hard requirement

**`MatchPlanService.confirmGeneration()` does not persist the preview. It re-runs the strategy from
scratch** (`MatchPlanService.java:951`). Anything non-deterministic silently gives the admin
different teams from the ones they approved. `RANDOM` does this knowingly and says so. A strategy
advertised as deterministic doing it is a correctness bug.

`OPTIMAL` must be a **pure function of `(player IDs, metric values, params)`**. Three rules:

1. **Sort the pool by `Player.id` ASC before assigning bit indices.** Do not trust incoming order —
   see §7.1; it is not stable.
2. **Enumerate masks in ascending numeric order, strict `<` on score, first-best wins.** No epsilon
   comparison: `a < b` on doubles is total and reproducible, whereas "within ε" is not transitive
   and can pick differently depending on visit order.
3. **Team A is the team containing the lowest player ID.** Falls out of fixing bit 0 to Team A;
   state it anyway so it survives refactors.

Float summation is reproducible because bits are always visited in the same order for a given mask.

Two mandatory regression tests: *same pool shuffled → identical teams*, and *two consecutive calls →
identical teams*. Neither passes today for the existing strategies (§7.1).

---

## 4. Algorithm

```
INPUT   players (size n, validated even and equal to 10/14/22 upstream)
        params  metric ∈ {skill, form}, shapeWeight λ, formWindow

 1. pool ← players sorted by id ASC
 2. v[i] ← metric value for pool[i]            // one array, computed once
 3. half ← n / 2
 4. best ← +∞ ; bestMask ← undefined
 5. for mask in 0 .. 2^n − 1:
        if (mask & 1) == 0            continue   // bit 0 pinned to Team A
        if bitCount(mask) ≠ half      continue
        (meanA, sdA) ← moments of v over set bits
        (meanB, sdB) ← moments of v over clear bits
        s ← |meanA − meanB| + λ · |sdA − sdB|
        if s < best: best ← s ; bestMask ← mask
 6. teamA ← pool[i] where bit i set ; teamB ← the rest
 7. notes ← formatted summary (§5)
```

`spread` is the **population** standard deviation (divide by `half`, not `half−1`) — the team is the
whole population, and the choice must simply be consistent.

**Complexity** `O(2^n)` masks, `O(n)` work per surviving candidate. Bounded and tiny at every legal
`n`; `Integer.bitCount` compiles to `POPCNT`.

**Both moments in one pass** via `Σv` and `Σv²` — a second pass per candidate is unnecessary and at
352,716 candidates it is the difference between 8 ms and 16 ms.

### 4.1 Guard against pool explosion

`2^n` is fine to 22 (4.2 M masks) and catastrophic by 30 (1.07 B). `MatchService.PLAYERS_PER_TEAM`
caps the pool at 22 today, so this is unreachable — but **a strategy that hangs a request thread is
worse than one that degrades**, and the cap lives in a different class that could change without
anyone thinking about this one.

> **If `n > 22`: delegate to the injected `BalancedGenerationStrategy` and say so in the notes.**

Inject the bean rather than reimplementing greedy — it is already tested.

---

## 5. The notes string — and using it as the observability channel

`matches.generation_notes` is `VARCHAR(500)` (`V1__initial_schema.sql:65`). **`BalanceGauge` does
not parse it**; the gauge reads the typed DTO fields `teamARatingAvg` / `teamBRatingAvg` /
`ratingDelta`, which `MatchPlanService` computes uniformly from `skillRating` whatever strategy ran.
The notes are prose. Format is therefore free — length is not.

```
OPTIMAL (metric=skill, λ=0.50): 1716 splits searched; avgA=7.14 avgB=7.11 Δmean=0.03;
sdA=1.22 sdB=1.25 Δsd=0.03; score=0.045
```

~150 characters, and bounded — unlike `CAPTAIN_PICK`, which interpolates two player names into a
500-char column and has no guard.

**Echo the *effective* parameters, always.** Not the requested ones — the ones that actually ran,
after clamping and defaulting. This is the cheapest possible defence against §7.2, where a parameter
has been silently ignored in production for months with nothing on screen to show it.

**`metric=form` will legitimately disagree with the gauge**, which always shows skill-rating
balance. That is the pre-existing `FORM_BASED` situation already explained in `BalanceGauge.tsx`'s
header comment. Naming the metric in the notes is what makes the discrepancy self-explaining rather
than looking like a bug.

---

## 6. Parameters and API surface

| Param | Type | Default | Validation |
|---|---|---|---|
| `metric` | `skill` \| `form` | `skill` | unknown value → **400** |
| `shapeWeight` | double | `0.5` | clamped to `[0, 5]`; non-numeric → **400** |
| `formWindow` | int | `5` | `metric=form` only; clamped to `≥ 1`; non-numeric → **400** |

**Reject garbage, clamp out-of-range.** This deliberately departs from
`FormBasedGenerationStrategy.resolveFormWindow()`, which swallows `"notAnInt"` and returns the
default. Silent fallback is precisely what let §7.2 hide. A clamped value is still the value the
user meant; an unparseable one is a caller bug and should say so.

No new endpoints. Both existing routes carry it:

```
POST /api/match-plans/{id}/generate?generationType=OPTIMAL&...
POST /api/match-plans/{id}/generate/confirm?generationType=OPTIMAL&...
```

**Read §7.2 before wiring any of these — the parameter channel does not currently work.**

---

## 7. Defects found while writing this spec

Three pre-existing problems surfaced. Each is independent of `OPTIMAL` and each is worth its own
commit — but two of them will silently break `OPTIMAL` if shipped around rather than fixed.

### 7.1 🐛 Deterministic strategies are not deterministic on tied ratings

`playerRepository.findAllByIdInAndTenantId(ids, tenantId)` is a derived query with **no `ORDER BY`**
(`PlayerRepository.java:50`). Row order is whatever Postgres returns. Every strategy then sorts with
`Comparator.comparingDouble(Player::getSkillRating)`, and `Stream.sorted()` is **stable** — so
players on equal ratings keep *database* order, and that order decides which team they land on.

This is not theoretical. `base_skill_rating` is an integer 1–10, `skill_rating` defaults to `5.0`,
and guests are created at the default — ties are the common case in a young group, not the edge
case. And because `CalculationService` `UPDATE`s `skill_rating` after every match, Postgres MVCC
rewrites those rows to new heap positions, **so the return order genuinely changes over time.**

Consequence: preview shows one split, `confirmGeneration` re-runs and can persist a different one.
Affects `BALANCED`, `SNAKE_DRAFT`, `FORM_BASED`, `CAPTAIN_PICK` — everything except `RANDOM`, which
is allowed to differ.

**Fix:** add `.thenComparing(Player::getId)` to the comparator in all four. Four lines. It makes the
sort a total order, after which input order cannot matter.

### 7.2 🐛 `params[...]` never reaches any strategy — `formWindow` has always been ignored

**Verified empirically**, not inferred. A probe added to `MatchPlanControllerTest`, run and then
reverted, captured what the controller actually binds:

```
POST /api/match-plans/1/generate?generationType=FORM_BASED&params[formWindow]=3
  → bound map = {generationType=FORM_BASED, params[formWindow]=3}
```

`@RequestParam(required = false) Map<String, String> params` (`MatchPlanController.java:232`) binds
**every** query parameter under its **literal** name. Spring MVC does not decompose `a[b]` bracket
syntax for a `Map` target — that is `@ModelAttribute` indexed-property behaviour, and this is not
that. So:

- `params.get("formWindow")` → `null` → `FormBasedGenerationStrategy` **always uses the default 5**.
  The frontend's form-window control (`TeamGenerationPage.tsx:248`) has never done anything.
- `params.get("captainAId")` / `("captainBId")` → `null`. `CAPTAIN_PICK` always auto-selects the top
  two rated players; the documented override is dead.
- `generationType` leaks *into* the params map, so strategies receive a key that is not theirs.

Why it went unnoticed: every strategy test constructs `TeamGenerationContext` and calls
`generate()` **directly** (`FormBasedGenerationStrategyTest.java:175`), so the parameters are
correct by construction and the controller is never in the picture. No controller test passes a
`params[...]` key — all thirteen `generate` tests send `generationType` alone.
`docs/api/API_REFERENCE.md:723` documents the broken syntax as though it works.

**Fix — pick one:**

| Option | Change | Cost |
|---|---|---|
| **A. Flatten the prefix** *(recommended)* | Keep the wire format; strip the `params[…]` wrapper server-side and drop `generationType` | Controller-only. **No frontend change, no API-doc change** — the documented contract starts being true |
| B. Flat query keys | `?formWindow=3`; filter out `generationType` | Frontend + API doc + Postman |
| C. JSON request body | `POST` with a body | Cleanest long-term, breaks every caller |

Whichever is chosen, **the fix needs a controller-level test asserting the strategy receives the
key** — the gap that hid this is the missing test, not the missing line.

> ⚠️ `OPTIMAL` takes `metric` and `shapeWeight` through this same channel. **Ship the fix first, or
> ship `OPTIMAL` with both parameters permanently stuck at their defaults.**

### 7.3 🧹 `STREAK_AWARE` is a registered bean that throws

`StreakAwareGenerationStrategy.generate()` unconditionally throws `unprocessable` with the message
*"will be enabled once CalculationService is live"*. `CalculationService` has been live for a long
time; `players.current_streak` is maintained after every match. The stated blocker is gone.

It is **not** currently reachable — `TeamGenerationPage.tsx:30` offers only `BALANCED`, `RANDOM`,
`SNAKE_DRAFT`, `FORM_BASED` — but it is a Spring bean the factory will happily resolve, advertising
a capability that does not exist, with a locked spec (threshold `+3`, tolerance `0.5`) nobody built.

**Resolve it in one of three ways, but resolve it.** Note that under `OPTIMAL`, "separate the hot
streaks" is naturally `+ w·|Δ(hot player count)|` — a term, not a class — which argues for deleting
the stub and the enum value rather than implementing it.

---

## 8. Edge cases

| # | Case | Behaviour |
|---|---|---|
| 1 | Pool smaller than the format needs | Already thrown by `selectStarterPlayerIds` before any strategy runs |
| 2 | Pool size odd | Unreachable — `TOTAL_PLAYERS_NEEDED` is even. Assert `n % 2 == 0` defensively; `IllegalStateException`, not a business error |
| 3 | Pool > 22 | Delegate to `BalancedGenerationStrategy`, note the fallback (§4.1) |
| 4 | **Every player identically rated** | Every split scores exactly `0.0`; first mask wins → "lowest 7 IDs vs highest 7". Deterministic and correct, but it **looks** like a bug to an admin, and repeats every week for a group that has not played yet. Say so in the notes: `all metric values equal — split is arbitrary`. This degenerate case is also the strongest argument for the rotation term (§9.2) |
| 5 | New player, no rated history, `metric=form` | Falls back to `skillRating` — inherited from `computeFormScore` |
| 6 | Nobody in the pool has history, `metric=form` | Every value falls back → identical to `metric=skill`. Notes should say so rather than implying form was used |
| 7 | Guest players (`is_guest`) | No special handling — they carry the default `5.0` and no history, so they are case 5. They make case 4 substantially more likely |
| 8 | Anonymized player (`anonymized_at`) | Rating survives anonymisation, so the maths is unaffected. Only the display name changes, and that is `MatchPlanService`'s business |
| 9 | Inactive player in the pool | `confirmGeneration` reactivates in a batch write (`MatchPlanService.java:1038`). Nothing for the strategy to do |
| 10 | `skillRating` null | Impossible — `NOT NULL DEFAULT 5.0` with a `CHECK` between 1.0 and 10.0 |
| 11 | Duplicate player in the pool | Prevented upstream: one confirmation per player, plus `validateAllPlayersFound` |
| 12 | Rating spread so wide nothing balances | Returns the best available and reports the achieved Δ. **Never throws** — an unbalanceable pool is a fact about the players, not an error |
| 13 | Withdrawal between preview and confirm | Pool changes, so the result legitimately changes. Pre-existing and deliberate (a reserve is promoted); not this strategy's problem, but the admin should re-preview |
| 14 | Two players tied on the objective *and* on ID | Impossible — ID is the primary key |
| 15 | λ = 0 | Pure mean optimisation. Valid, documented, and the reference case in tests |
| 16 | λ enormous (clamped to 5) | Shape dominates; mean balance may visibly suffer and the gauge may read "uneven". Correct behaviour for what was asked — the clamp is what stops it going further |
| 17 | Notes exceeding 500 chars | Impossible for `OPTIMAL` (fixed-width, no names). Flagged only because `CAPTAIN_PICK` interpolates two unbounded names into the same column with no guard — a separate latent bug |

---

## 9. Designed-for, not built

Each of these is **a term added to §4 step 5**, not a new class. Listed so v1 does not accidentally
foreclose them.

### 9.1 Hard constraints → a feasibility filter

`must-be-together`, `must-be-apart`, `both keepers must not share a side`. Insert `if (!feasible(mask)) continue;`
before scoring. Cost is one bitmask test per candidate.

**On infeasibility: relax and report, do not throw.** A group that has over-constrained itself wants
teams and a warning, not a 422 twenty minutes before kickoff.

Needs either a `player_pair_constraints` table or throwaway per-request params.

### 9.2 Rotation → a penalty term

The complaint no current strategy can hear: *"I've been on João's team nine weeks running."*
Deterministic strategies **cause** this — same pool, same ratings, same teams, every week.
`BALANCED` is repeatable by design, which is a feature mathematically and a bug socially.

The data already exists: `player_stats → match_team → match` gives every past pairing. Add
`+ w · Σ(pairs sharing a team) recentTogetherCount`. One query, no migration. It also resolves case
4 above, turning an arbitrary tie-break into a useful one.

### 9.3 Goalkeeper coverage → one boolean

[TEAM_GENERATION_DESIGN.md §3.G](../features/TEAM_GENERATION_DESIGN.md) scopes `POSITION_BASED` as a
full GK/DEF/MID/FWD model at "🔴 High" effort. In 5- and 7-a-side almost none of that matters. **One
`can_keep` boolean on `players`** captures the only positional failure that actually ruins a match —
both keepers on one side. One-column migration, one line in §9.1's filter.

### 9.4 Streak separation → the term that retires §7.3

`+ w · |Δ(players with currentStreak ≥ 3)|`.

---

## 10. Test plan

Mirrors the existing `*GenerationStrategyTest` style — plain JUnit against the strategy, plus the
controller-level test that §7.2 proves is missing.

**Contract** (per `MatchType`)
1. Returns exactly two teams of `n/2`
2. Output is a partition: every input player appears exactly once, nothing invented

**Optimality** — the tests that justify the whole exercise
3. With `λ=0`, output equals an independent brute-force minimiser of `|Δmean|` written in the test
4. Property test, ~1000 random pools: `OPTIMAL(λ=0)` is **never worse** than `BALANCED` on `|Δmean|`
5. Hand-built `[9,9,9,4,4]` vs `[7,7,7,7,7]` fixture: `λ=0` may pick either; `λ=0.5` picks the
   shape-balanced split

**Determinism** (§3 — these are the regressions)
6. Same pool, shuffled input order → byte-identical teams
7. Two consecutive calls → identical teams
8. Pool where several players share a rating → still identical across shuffles

**Metric**
9. `metric=form` calls `findRecentRatedByPlayerId` with the right `Pageable`
10. `metric=form`, no history → falls back to `skillRating`
11. `metric=form`, nobody has history → identical result to `metric=skill`

**Params**
12. Unknown `metric` → 400; non-numeric `shapeWeight` → 400
13. `shapeWeight` out of range → clamped, and the notes echo the **effective** value
14. Notes stay under 500 characters at `ELEVEN_A_SIDE`

**Edge**
15. All-equal ratings → deterministic, equal-size, notes flag the arbitrariness (case 4)
16. `n > 22` → greedy fallback, notes say so
17. `TeamGenerationStrategyFactory` resolves `OPTIMAL`

**Performance**
18. `ELEVEN_A_SIDE` completes under 100 ms — a smoke ceiling, ~10× the expected cost, not a benchmark

**Controller** (the §7.2 gap)
19. A `params[...]`-style request results in the strategy receiving the parsed key

---

## 11. Rollout

**No migration.** `generation_type` is `VARCHAR(20)` with no `CHECK` constraint
(`V1__initial_schema.sql:59` — only `match_type`, `score_*` and `team_order` are constrained on that
table). `"OPTIMAL"` is 7 characters, and `GenerationTypeConverter` upper-cases on read.

Order matters, per [CONTRIBUTING §Deployment order](../../CONTRIBUTING.md):

1. **§7.2 first** — otherwise `OPTIMAL` ships with dead parameters. Independently valuable; it also
   revives `formWindow` and the `CAPTAIN_PICK` overrides
2. **§7.1** — four-line comparator fix, own commit, own regression test
3. **Backend `OPTIMAL`** — `OptimalPartitionStrategy` + `PlayerMetrics` extraction + enum value +
   tests. Deploy and confirm running
4. **Frontend** — `'OPTIMAL'` into the `GenerationType` union **and** the zod enum
   (`src/types/match.ts:5` and `:77` — both, they are separate declarations), plus
   `GENERATION_TYPES` (`TeamGenerationPage.tsx:30`), plus a `shapeWeight` control
5. **i18n in all three locales** — `en`, `pt`, `es`. A missing key renders the raw enum name, so an
   untranslated build ships a dropdown reading `OPTIMAL`
6. **Docs, same commit as the code**: `API_REFERENCE.md` (backend repo — and correct the
   `params[...]` rows while there), Postman collection, this plan's status line,
   `TEAM_GENERATION_DESIGN.md`, `product/feature-status.md`

**`BALANCED` stays the default.** Ship `OPTIMAL` as an option, let a few weeks of real matches show
whether the shape term changes anything anyone notices, and promote it only then. Changing the
default is a separate decision with its own evidence, not a footnote to this one.

---

## 12. Open decisions

| # | Question | Recommendation |
|---|---|---|
| 1 | Default λ | `0.5` — mostly a tie-break among mean-equal splits (§2.2) |
| 2 | Fix §7.2 via A, B or C | **A** — no frontend change, and it makes the published API doc true |
| 3 | `OPTIMAL` as the default type? | **Not yet.** Ship alongside; revisit with evidence |
| 4 | Strict param validation, departing from `FORM_BASED`'s silent fallback? | **Yes** — silent fallback is how §7.2 hid |
| 5 | `STREAK_AWARE`: build, delete, or leave? | **Delete** the class and the enum value; it returns as §9.4 |
| 6 | Fix §7.1 across the four existing strategies, or only in `OPTIMAL`? | **All four** — they have the same latent bug |
| 7 | Expose `metric=form` in the UI, given `FORM_BASED` already exists? | Ship `metric=skill` only in the UI at first; two overlapping form controls will confuse before they help |
| 8 | Batch the `metric=form` reads? | Not now — `n` queries is the existing `FORM_BASED` cost, not a regression. Worth doing when someone complains |

---

## 13. Files

**New** — `service/teamgeneration/OptimalPartitionStrategy.java`,
`service/teamgeneration/PlayerMetrics.java` (extracted),
`test/.../OptimalPartitionStrategyTest.java`

**Changed** — `model/GenerationType.java` (add `OPTIMAL`, remove `STREAK_AWARE` per decision 5),
`FormBasedGenerationStrategy.java` (delegate to `PlayerMetrics`), `MatchPlanController.java` (§7.2),
`BalancedGenerationStrategy` / `SnakeDraft` / `FormBased` / `CaptainPick` (§7.1 comparator),
`MatchPlanControllerTest.java` (§7.2 test), `docs/api/API_REFERENCE.md`

**Frontend** — `src/types/match.ts`, `src/features/teamGeneration/TeamGenerationPage.tsx`,
`src/locales/{en,pt,es}/*`

**Deleted** — `service/teamgeneration/StreakAwareGenerationStrategy.java` (decision 5)
