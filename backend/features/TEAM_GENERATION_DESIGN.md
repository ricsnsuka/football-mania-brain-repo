# Team Generation — Design Evaluation
> **Status:** ✅ Approved — Ready for Implementation
> **Context:** Football Management System — Java 21 · Spring Boot 3.4.5
> **Created:** 2026-05-16 · **Decisions locked:** 2026-05-16
> **Depends on:** MatchPlan (confirmed player pool), Player.skillRating, PlayerStat history

---

## ✅ Implementation Decisions (Locked 2026-05-16)

All open questions resolved. The following are firm requirements for the dev-assistant:

| # | Decision | Locked Value |
|---|----------|-------------|
| 1 | Phase 1 algorithms | `BALANCED`, `RANDOM`, `SNAKE_DRAFT` — all three in one PR |
| 2 | `formWindow` for FORM_BASED | Per-request DTO param — `Map<String,String> generationParams`, key `"formWindow"`, default `5` |
| 3 | Hot streak threshold (`STREAK_AWARE`) | `currentStreak >= 3` |
| 4 | Swap tolerance (`STREAK_AWARE`) | `0.5` rating points max delta when swapping for streak balance |
| 5 | `generationNotes` in response | Add `String generationNotes` field to `MatchDTO` and `Match` entity (new V6 migration column) |
| 6 | Re-generate endpoint | `POST /api/match-plans/{id}/generate` — can be called multiple times; returns a preview each time |
| 7 | Generation flow | **Preview first** → `POST /api/match-plans/{id}/generate` returns `MatchPreviewDTO` (not persisted); admin confirms with `POST /api/match-plans/{id}/generate/confirm` which creates the `Match` |
| 8 | `CAPTAIN_PICK` | Server-side snake draft simulation — auto-selects top-2 rated players as captains (or use `captainAId`/`captainBId` params). Full interactive DraftSession API remains a future Phase 4 item. |

### Phase 1 Scope (this sprint)

```
✅ Strategy pattern infrastructure
   ├── TeamGenerationStrategy (interface)
   ├── TeamGenerationContext (record)
   ├── TeamGenerationResult (record)
   └── TeamGenerationStrategyFactory (@Component)

✅ Phase 1 strategy implementations
   ├── BalancedGenerationStrategy
   ├── RandomGenerationStrategy
   └── SnakeDraftGenerationStrategy

✅ GenerationType enum expansion
   BALANCED, RANDOM, SNAKE_DRAFT, FORM_BASED (Phase 2), STREAK_AWARE* (Phase 2 TBD), CAPTAIN_PICK (Phase 3)
   * STREAK_AWARE still throws BusinessException.unprocessable (pending CalculationService)

✅ MatchPreviewDTO (record) — returned by POST /match-plans/{id}/generate (stateless)
✅ POST /api/match-plans/{id}/generate         → MatchPreviewDTO (not persisted)
✅ POST /api/match-plans/{id}/generate/confirm  → MatchDTO (creates the Match)

✅ V6 migration: ADD COLUMN generation_notes VARCHAR(500) to matches
✅ Match entity: add generationNotes field
✅ MatchDTO: add generationNotes field

✅ MatchService: wire strategy factory for all non-MANUAL generation types
```

### Phase 2 Scope (after CalculationService is live)

```
✅ FormBasedGenerationStrategy   — reads player_stats history, formWindow param
🔲 StreakAwareGenerationStrategy  — reads players.current_streak, threshold=3, tolerance=0.5
✅ PlayerStatRepository: findRecentRatedByPlayerId(playerId, Pageable)
```

### Phase 3 Scope (future)

```
✅ CaptainPickGenerationStrategy  — server-side snake draft simulation — `CaptainPickGenerationStrategy` (no DraftSession, stateless).
   Auto-selects top-2 rated players as captains (or use captainAId/captainBId params).
🔮 PositionBasedGenerationStrategy + player position Flyway migration
```

### Phase 4 Scope

```
✅ Phase 4 — Interactive Draft Session API
   ├── DraftSession entity + DraftSessionRepository
   ├── DraftSessionService (5 operations: create, get, pick, confirm, cancel)
   ├── DraftSessionController (5 endpoints at /api/draft-sessions)
   └── V4__draft_sessions.sql migration (4 tables: draft_sessions, draft_session_team_a,
       draft_session_team_b, draft_session_remaining)
```

---

## Table of Contents

1. [Available Player Data](#1-available-player-data)
2. [Current State](#2-current-state)
3. [Algorithm Catalogue](#3-algorithm-catalogue)
4. [Algorithm Comparison Matrix](#4-algorithm-comparison-matrix)
5. [Code Architecture — Strategy Pattern](#5-code-architecture--strategy-pattern)
6. [Updated GenerationType Enum](#6-updated-generationtype-enum)
7. [Algorithm Deep-Dives](#7-algorithm-deep-dives)
8. [Edge Cases & Shared Constraints](#8-edge-cases--shared-constraints)
9. [Recommendation & Phased Rollout](#9-recommendation--phased-rollout)
10. [Open Questions](#10-open-questions)

---

## 1. Available Player Data

Before choosing algorithms, let's map exactly what data is available per player at generation time:

| Field | Source | Type | Notes |
|-------|--------|------|-------|
| `skillRating` | `players.skill_rating` | `double` 1.0–10.0 | Dynamic — updated by `CalculationService` after each match |
| `baseSkillRating` | `players.base_skill_rating` | `int` 1–10 | Static — admin-set at creation, never auto-modified |
| `currentStreak` | `players.current_streak` | `int` | Positive = win streak, negative = loss streak (pending CalculationService) |
| `longestStreak` | `players.longest_streak` | `int` | Historical peak |
| Per-match stats | `player_stats` | rows per match | goals, assists, own_goals, rating (0–10), match_result (WIN/LOSS/DRAW), is_mvp |
| Confirmed? | `player_confirmations` | status | CONFIRMED players from a MatchPlan |

### What we **don't** have (yet)

| Missing Data | Impact |
|--------------|--------|
| Player **positions** (goalkeeper, defender…) | Positional balancing algorithms not possible without a migration |
| **Chemistry** between specific players | No relationship graph between players |
| **Availability preferences** (time, location) | Not applicable — MatchPlan confirmation covers this |
| **Fatigue** (matches played in last N days) | Could derive from `player_stats` join, but no direct field |

---

## 2. Current State

The `GenerationType` enum currently has two values:

```java
public enum GenerationType {
    BALANCED,   // intended: equalize team skill ratings
    MANUAL      // admin provides explicit team lists
}
```

`BALANCED` is **declared but not fully implemented** — it has no backing algorithm class.
`MANUAL` works: the caller provides player IDs per team in `MatchCreateDTO`.

`MatchService.PLAYERS_PER_TEAM` already enforces correct player counts per `MatchType`:
```java
FIVE_A_SIDE   → 5 players per team (10 total)
SEVEN_A_SIDE  → 7 players per team (14 total)
ELEVEN_A_SIDE → 11 players per team (22 total)
```

---

## 3. Algorithm Catalogue

### A. `BALANCED` — Greedy Rating Equalizer *(replace placeholder)*

**Concept:** Sort players by `skillRating` descending. Assign each player to whichever team
currently has the lower cumulative rating (greedy bin-packing).

**Visual example with 6 players (FIVE_A_SIDE omitting 1 spare):**
```
Players sorted: [9.5, 8.2, 7.8, 7.1, 6.5, 6.0]

Iteration:
  9.5 → Team A (both empty, A chosen)   A=9.5   B=0
  8.2 → Team B (lower sum)              A=9.5   B=8.2
  7.8 → Team B (lower sum)              A=9.5   B=16.0
  7.1 → Team A (lower sum)              A=16.6  B=16.0
  6.5 → Team B (lower sum)              A=16.6  B=22.5
  6.0 → Team A (lower sum)              A=22.6  B=22.5

Result:  Team A avg=7.53  Team B avg=7.50  Δ=0.03 ✅
```

**When to use:** Default go-to for any casual match. Simple, fast, deterministic.

**Pros:** `O(n log n)`, predictable, easy to explain to players.

**Cons:** Doesn't account for recent form, streaks, or positional needs.

---

### B. `RANDOM` — Pure Shuffle

**Concept:** Randomly shuffle the confirmed player list, cut in half.

**Visual example:**
```
Players: [A, B, C, D, E, F, G, H, I, J, K, L, M, N]
Shuffle: [H, C, L, A, N, G, D, J, B, F, K, M, I, E]
Team 1: [H, C, L, A, N, G, D]
Team 2: [J, B, F, K, M, I, E]
```

**When to use:** Casual/fun matches where fairness is explicitly not the goal ("let fate decide").
Great for parties or low-stakes games.

**Pros:** Trivially simple, no data dependency, adds "luck" element.

**Cons:** Can produce wildly unbalanced teams. Rating difference could be 3.0+ points.

---

### C. `SNAKE_DRAFT` — Alternating Top-Pick

**Concept:** Sort players by `skillRating` descending. Team A picks 1st, Team B picks
next 2, Team A picks next 2, alternating in snake pattern (like a real draft).

**Visual example with 14 players:**
```
Sorted by rating: P1(9.8), P2(9.1), P3(8.7), P4(8.2), P5(7.9), P6(7.5),
                  P7(7.1), P8(6.9), P9(6.5), P10(6.2), P11(5.8), P12(5.5),
                  P13(5.1), P14(4.9)

Round 1 →A:  A=[P1]          B=[]
Round 1 →B:  A=[P1]          B=[P2]
             A=[P1]          B=[P2, P3]      ← B gets 2
Round 2 →A:  A=[P1,P4]       B=[P2,P3]
             A=[P1,P4,P5]    B=[P2,P3]       ← A gets 2
Round 2 →B:  A=[P1,P4,P5]    B=[P2,P3,P6]
             A=[P1,P4,P5]    B=[P2,P3,P6,P7]
...and so on

Team A avg: (9.8+8.2+7.9+6.9+6.2+5.5+4.9)/7 = 7.06
Team B avg: (9.1+8.7+7.5+7.1+6.5+5.8+5.1)/7 = 7.11  Δ=0.05 ✅
```

**When to use:** When players want a transparent, fair-feeling distribution and enjoy seeing 
the "draft logic."

**Pros:** Very balanced (mathematically near-optimal), transparent and explainable.

**Cons:** Slightly more complex than greedy. Ties in skill rating need a tie-breaker.

---

### D. `FORM_BASED` — Recent Performance Weight

**Concept:** Compute a **form score** from each player's last N matches (configurable, default 5).
Use this form-adjusted score instead of raw `skillRating` as the balancing input.

**Form Score formula:**
```
formScore = AVG(match_rating over last N matches)
          × (1 + WIN_RATIO bonus)
          × recencyWeight

recencyWeight: most recent match = 1.0, previous = 0.9, 0.8, 0.7, 0.6 ...

If player has < N matches → fall back to skillRating
```

**Example:**
```
Player A: skillRating=7.2, last 5 match ratings=[8.0, 7.5, 8.2, 7.8, 8.5], 4 wins
  → formScore = 8.0 × 1.15 = 9.2 (hot player)

Player B: skillRating=7.0, last 5 match ratings=[5.5, 6.0, 5.8, 6.2, 5.9], 1 win
  → formScore = 5.88 × 0.86 = 5.05 (cold player)

Balanced teams using formScore instead of skillRating → accounts for current form
```

**When to use:** Regular groups where the same players play weekly and form matters.
Prevents "always balanced on paper but one team always wins" effect.

**Pros:** Adapts to real current performance, rewards in-form players.

**Cons:** Requires `player_stats` history. New players (< N matches) fall back to
`skillRating`. The form window N should be configurable per match.

**Input from DTO:** `formWindow` (int, optional, default 5).

---

### E. `STREAK_AWARE` — Streak Separation

**Concept:** After generating balanced teams (using BALANCED greedy), apply a
**streak separation pass**: try to ensure players on hot win streaks (currentStreak >= +3)
are not all on the same team. Swap pairs if it improves streak balance without
degrading rating balance beyond a threshold (e.g. ±0.3 rating delta allowed).

**Visual:**
```
After BALANCED:
  Team A: [P1★★★(streak+5), P4★★(streak+3), P7, P9, P11, P13, P15]
  Team B: [P2, P3, P6, P8, P10, P12, P14]

Streak check: Team A has 2 hot players, Team B has 0
→ Attempt swap P4(streak+3, rating=8.2) ↔ P6(streak-1, rating=8.0)
→ Rating delta: 0.2 (within 0.3 threshold) → SWAP applied

Result Team A: [P1★★★, P6, P7, P9, P11, P13, P15]   avg=7.12
       Team B: [P2, P3, P4★★, P8, P10, P12, P14]    avg=7.09
→ Hot streaks distributed: A=1, B=1
```

**When to use:** Complements BALANCED when the group tracks streaks and wants to break
momentum clusters, making matches more competitive.

**Pros:** Reduces "everyone hot on the same team" problem. Small adjustment on top of BALANCED.

**Cons:** Swap logic adds complexity. Threshold for streak/rating trade-off is a tunable parameter.

---

### F. `CAPTAIN_PICK` — Human Draft (Interactive)

**Concept:** The system selects two captains (highest `skillRating` pair, or admin-designated),
then enables an interactive alternating pick via API. Each captain picks one player
per round until all players are assigned.

**API Flow:**
```
POST /api/match-plans/{id}/generate?type=CAPTAIN_PICK
→ 200 { draftSessionId, captainA: {playerId, name}, captainB: {playerId, name}, availablePlayers: [...] }

POST /api/draft/{sessionId}/pick   { playerId }  (Captain A picks)
→ 200 { nextPickCaptain: B, remainingPlayers: [...] }

POST /api/draft/{sessionId}/pick   { playerId }  (Captain B picks)
→ ...repeat...

POST /api/draft/{sessionId}/confirm
→ 201 MatchDTO (match created from completed draft)
```

**When to use:** Socially engaging — players enjoy watching the draft live. Natural for 
groups where the captains "know" the players' real-world abilities beyond the rating.

**Pros:** Maximum social engagement, captains can factor in things the algorithm doesn't know
(injuries, energy levels, personal rivalries).

**Cons:** Requires a separate "draft session" concept with state management (Redis or DB).
Blocking — someone must pick each round. Out of scope for current feature scope but
an excellent future feature.

**Complexity:** High — needs new `DraftSession` entity, dedicated service, timeout handling.

---

### G. `POSITION_BASED` *(Future — requires DB migration)*

**Concept:** Assign positions to players (GK, DEF, MID, FWD). Ensure each team has the
required positional distribution (e.g., 7-a-side: 1 GK, 2 DEF, 3 MID, 1 FWD).
Then balance by rating within position groups.

**Requires:**
- New `position` column on `players` (Flyway migration + enum)
- Position counts per `MatchType`
- Position-aware pairing algorithm

**When to use:** Serious groups that care about team structure, not just overall rating.

**Complexity:** Medium-High. Positional constraint satisfaction is a constrained optimization problem.

---

## 4. Algorithm Comparison Matrix

| Algorithm | Rating Balance | Effort to Implement | Requires History | Interactive | Social Fun | Deterministic |
|-----------|:--------------:|:-------------------:|:----------------:|:-----------:|:----------:|:-------------:|
| `BALANCED` (greedy) | ⭐⭐⭐⭐⭐ | 🟢 Low | ❌ No | ❌ No | ⭐⭐ | ✅ Yes |
| `RANDOM` | ⭐ | 🟢 Trivial | ❌ No | ❌ No | ⭐⭐⭐⭐⭐ | ❌ No |
| `SNAKE_DRAFT` | ⭐⭐⭐⭐⭐ | 🟡 Medium | ❌ No | ❌ No | ⭐⭐⭐ | ✅ Yes |
| `FORM_BASED` | ⭐⭐⭐⭐ | 🟡 Medium | ✅ Yes (N matches) | ❌ No | ⭐⭐⭐ | ✅ Yes |
| `STREAK_AWARE` | ⭐⭐⭐⭐ | 🟡 Medium | ✅ Yes (streak) | ❌ No | ⭐⭐⭐ | ✅ Yes |
| `CAPTAIN_PICK` | ⭐⭐⭐ (human) | 🟡 Medium | ❌ No | ❌ No (server-side sim) | ⭐⭐⭐⭐⭐ | ❌ No |
| `POSITION_BASED` | ⭐⭐⭐⭐⭐ | 🔴 High | ❌ No | ❌ No | ⭐⭐⭐ | ✅ Yes |

---

## 5. Code Architecture — Strategy Pattern

All algorithms should live behind a **Strategy interface** so they can be selected at
runtime from the `GenerationType` parameter. This keeps `MatchService` clean and
makes adding new algorithms trivial.

```
service/
└── teamgeneration/
    ├── TeamGenerationStrategy.java         ← interface
    ├── TeamGenerationContext.java          ← input record (players + config)
    ├── TeamGenerationResult.java           ← output record (List<List<Player>>)
    ├── BalancedGenerationStrategy.java     ← BALANCED (greedy)
    ├── RandomGenerationStrategy.java       ← RANDOM
    ├── SnakeDraftGenerationStrategy.java   ← SNAKE_DRAFT
    ├── FormBasedGenerationStrategy.java    ← FORM_BASED
    ├── CaptainPickGenerationStrategy.java  ← CAPTAIN_PICK (server-side simulation)
    └── StreakAwareGenerationStrategy.java  ← STREAK_AWARE
```

### Interface contract

```java
/**
 * All team generation algorithms implement this interface.
 * Spring's bean registry + a factory resolves the correct strategy at runtime.
 */
public interface TeamGenerationStrategy {

    /** The GenerationType this strategy handles. */
    GenerationType type();

    /**
     * Generate two balanced teams from the provided player pool.
     *
     * @param context  all players confirmed for the match + algorithm config params
     * @return         a result containing exactly two teams of equal size
     * @throws BusinessException if the player pool size does not match matchType requirements
     */
    TeamGenerationResult generate(TeamGenerationContext context);
}
```

### Input/Output records

```java
/** Input to any TeamGenerationStrategy. */
public record TeamGenerationContext(
    MatchType matchType,
    List<Player> players,               // confirmed pool — already validated for count
    Map<String, String> params          // algorithm-specific params (e.g. "formWindow" = "5")
) {}

/** Output of any TeamGenerationStrategy — exactly two groups. */
public record TeamGenerationResult(
    List<Player> teamA,
    List<Player> teamB,
    String algorithmNotes               // human-readable explanation of how teams were formed
) {}
```

### Strategy Factory

```java
@Component
@RequiredArgsConstructor
public class TeamGenerationStrategyFactory {

    private final List<TeamGenerationStrategy> strategies; // Spring auto-injects all implementations

    public TeamGenerationStrategy resolve(GenerationType type) {
        return strategies.stream()
                .filter(s -> s.type() == type)
                .findFirst()
                .orElseThrow(() -> BusinessException.badRequest(
                        "No team generation strategy found for type: " + type));
    }
}
```

### Usage in MatchService

```java
// In MatchService.createMatch() or new generateTeams() method:
TeamGenerationContext ctx = new TeamGenerationContext(matchType, players, dto.generationParams());
TeamGenerationResult result = strategyFactory.resolve(generationType).generate(ctx);
// → result.teamA() and result.teamB() become the two MatchTeams
```

---

## 6. Updated GenerationType Enum

Current enum only has `BALANCED` and `MANUAL`. Proposed expansion:

```java
public enum GenerationType {
    MANUAL,         // caller provides explicit player IDs per team
    BALANCED,       // greedy rating equalizer — implemented Phase 1
    RANDOM,         // pure shuffle — implemented Phase 1
    SNAKE_DRAFT,    // alternating top-pick — implemented Phase 1
    FORM_BASED,     // last-N-match rating weighted — implemented Phase 2
    STREAK_AWARE,   // BALANCED + streak separation post-pass — implemented Phase 2
    CAPTAIN_PICK    // server-side snake draft simulation — implemented Phase 3
}
```

**No DB migration needed for the enum values** — they are stored as `VARCHAR(20)` strings
already in the `matches` table (no CHECK constraint on allowed values in V1). Adding
new values just needs the Java enum updated.

> ⚠️ **Note:** `CAPTAIN_PICK` is implemented as a server-side snake draft simulation (Phase 3, `CaptainPickGenerationStrategy` — stateless, no DraftSession).
> Full interactive draft session (real-time human picks via API) is now also available as Phase 4: see `DraftSessionController` at `/api/draft-sessions` and `docs/features/DRAFT_SESSION_FEATURE.md`.

---

## 7. Algorithm Deep-Dives

### 7.1 BALANCED — Greedy Bin Packing

```
Input:  players sorted by skillRating DESC
Output: two teams of equal size, minimised Δ(avgA, avgB)

Algorithm:
    teamA = [], teamB = []
    for each player p in sorted order:
        if sum(teamA) <= sum(teamB):
            assign p → teamA
        else:
            assign p → teamB

Complexity: O(n log n) for sort + O(n) for assignment = O(n log n) total
Worst case Delta: ≈ skillRating[n/2] / n  (the "middle" player's impact spread)

Tie-breaking (when sums are equal): alternate assignment (A first for odd positions).
```

**Proof of near-optimality:** For a sorted sequence this greedy approach achieves
≤ P_max / (n+1) imbalance, where P_max is the top player rating. For n=14, P_max=10:
max possible imbalance ≤ 10/15 = 0.67 rating points. In practice much smaller.

---

### 7.2 SNAKE_DRAFT — Alternating Pick

```
Input:  players sorted by skillRating DESC, indexed 0..n-1
Output: two teams using snake pattern assignment

Pattern for n=14:
  Pick #:  1  2  3  4  5  6  7  8  9 10 11 12 13 14
  Team:    A  B  B  A  A  B  B  A  A  B  B  A  A  B

General rule:
  i (1-indexed): teamA if ceil(i/2) is odd for odd pick positions
  Simpler: A gets picks {1,4,5,8,9,12,13}, B gets {2,3,6,7,10,11,14}

Complexity: O(n log n) sort only
```

---

### 7.3 FORM_BASED — Weighted Recent Rating

```
Input:  players, formWindow N (default 5, min 1)

For each player p:
    recentStats = findRecentRatedByPlayerId(p.id, PageRequest.of(0, N))
                  — JPQL navigates ps.matchTeam.match to get match date order
    ratings = recentStats filtered to non-null, non-zero rating values

    if ratings.isEmpty():
        formScore = p.skillRating           ← no history → fall back

    else:
        // Linear decay: i=0 is most recent, weight_i = N - i
        weightedSum = Σ( ratings[i] × (ratings.size() - i) )
        totalWeight = Σ( ratings.size() - i )  for i in 0..ratings.size()-1
        formScore   = weightedSum / totalWeight
        // No WIN_RATIO bonus; no clamping applied beyond natural weighted average

// Example for N=3, ratings=[8.5, 7.8, 7.0] (newest→oldest):
//   weights = [3, 2, 1], weightedSum = 8.5×3 + 7.8×2 + 7.0×1 = 25.5+15.6+7.0 = 48.1
//   totalWeight = 6
//   formScore = 48.1 / 6 ≈ 8.02

Then apply BALANCED greedy using formScore instead of skillRating.

DB query used:
    PlayerStatRepository.findRecentRatedByPlayerId(playerId, Pageable)
    — JPQL: SELECT ps FROM PlayerStat ps
             WHERE ps.player.id = :playerId
               AND ps.rating IS NOT NULL
               AND ps.rating > 0
             ORDER BY ps.matchTeam.match.createdAt DESC
```

**Trade-off:** Adds N × |players| DB rows fetched. For 14 players, N=5: up to 70 rows.
Mitigated by the `Pageable` limit — only the required rows are fetched per player.

---

### 7.4 STREAK_AWARE — Balanced + Streak Post-Pass

```
Step 1: Generate teams using BALANCED greedy (by skillRating)
Step 2: Define "hot" threshold = currentStreak >= +3
Step 3: Count hotA = hot players in teamA, hotB = hot players in teamB
Step 4: While |hotA - hotB| > 1:
    Find candidate swap:
        hotPlayer from the team with more hot players (rating = rH)
        coldPlayer from the other team with closest rating (|rC - rH| < SWAP_THRESHOLD)
        SWAP_THRESHOLD = 0.5 (configurable)
    If valid swap found:
        Execute swap
        Recalculate hotA, hotB
    Else:
        Break (no valid swap within threshold — accept imbalance)

Complexity: O(n log n) + O(h × n) post-pass where h = hot players (≤ n/2 typically)

Note: This is a post-pass decoration on BALANCED, not a standalone algorithm.
```

---

### 7.6 CAPTAIN_PICK — Server-Side Snake Draft Simulation

> ⚠️ This is a **server-side simulation**, not the full interactive draft originally designed.
> Full real-time human picks via API (with `DraftSession` entity, pick endpoints, timeout handling)
> remains a **Phase 4** item.

```
Input:  players, optional captainAId (Long), optional captainBId (Long)

Step 1 — Resolve captains:
    If captainAId param provided → find player with that ID in pool (throws IllegalArgumentException if not found)
    Else → auto-select highest-rated player in pool as Captain A

    If captainBId param provided → find player with that ID in pool (throws IllegalArgumentException if not found)
    Else → auto-select highest-rated player in pool excluding Captain A as Captain B

Step 2 — Seed teams:
    teamA = [captainA]
    teamB = [captainB]

Step 3 — Snake draft from remaining:
    remaining = pool.remove(captainA, captainB), sorted by skillRating DESC

    for i in 0..remaining.size()-1:
        round    = i / 2         (0-indexed pair)
        posInRound = i % 2       (0=first pick of pair, 1=second)
        aLeadsRound = (round % 2 == 0)

        if (aLeadsRound && posInRound==0) || (!aLeadsRound && posInRound==1):
            teamA.add(remaining[i])
        else:
            teamB.add(remaining[i])

    Pattern for 12 remaining players:
      i:    0  1  2  3  4  5  6  7  8  9 10 11
      team: A  B  B  A  A  B  B  A  A  B  B  A

Step 4 — Build notes:
    n = totalPlayers / 2
    notes = "CAPTAIN_PICK: captainA=<name> captainB=<name> avgA=<x> avgB=<y> Δ=<z>"

Complexity: O(n log n)
```

**Supported params:**
- `captainAId` — `Long` player ID for Team A captain (optional). Auto-selects top-rated if omitted.
- `captainBId` — `Long` player ID for Team B captain (optional). Auto-selects 2nd-highest if omitted.

**Notes field example:**
```
CAPTAIN_PICK: captainA=João Silva captainB=Miguel Santos avgA=7.43 avgB=7.21 Δ=0.22
```

---

## 8. Edge Cases & Shared Constraints

All generation algorithms must handle these cases:

| Scenario | Required Behaviour |
|----------|--------------------|
| Player count < required for matchType | Throw `BusinessException.badRequest(...)` (already enforced in MatchService) |
| All players have same skillRating | Any distribution is valid; BALANCED distributes alternately |
| Odd number of players | Only possible in MANUAL; all auto-gen algorithms require exact even count |
| Player with `skillRating = null` | Not possible — `skillRating` has `NOT NULL DEFAULT 5.0` in DB |
| Single player with extreme rating (e.g. 10.0 vs rest 5.0) | BALANCED handles it optimally (10.0 player in team, rest distributed by greedy) |
| FORM_BASED with a player who has 0 match history | Falls back to `player.skillRating` for that player |
| `captainAId`/`captainBId` param references unknown player ID | Throws `IllegalArgumentException` with descriptive message |
| CAPTAIN_PICK with valid params | Works — auto-selects captains if params omitted, or uses specified IDs |
| Duplicate player IDs in pool | Already validated in `MatchService.createMatch()` |
| Inactive player in confirmed pool | `MatchService` auto-activates; generation uses the full pool |

---

## 9. Recommendation & Phased Rollout

### Phase 1 — Implement Now (Low/Medium effort, high value)

| Algorithm | Reason |
|-----------|--------|
| ✅ `BALANCED` | Core feature — replaces the placeholder. O(n log n), deterministic |
| ✅ `RANDOM` | Trivial to add alongside; useful for casual games |
| ✅ `SNAKE_DRAFT` | Near-identical complexity to BALANCED; gives a transparent alternative |

All three share the Strategy interface and require no extra data beyond `skillRating`.
Implement as a **single PR** with the Strategy pattern infrastructure.

### Phase 2 — Implement Next Sprint (Depends on CalculationService being live)

| Algorithm | Prerequisite |
|-----------|-------------|
| ✅ `FORM_BASED` | `CalculationService` must be populating `player_stats.rating` reliably |
| ✅ `STREAK_AWARE` | `CalculationService` must be updating `players.current_streak` reliably |

These read from `player_stats` — only meaningful once match history exists.

### Phase 3 — Implemented (server-side simulation)

| Algorithm | Notes |
|-----------|-------|
| ✅ `CAPTAIN_PICK` | Server-side snake draft simulation implemented. Auto-selects top-2 rated as captains (or use `captainAId`/`captainBId` params). Full interactive DraftSession API (real-time human picks) remains Phase 4. |

### Phase 4 — ✅ Implemented

| Feature | Notes |
|---------|-------|
| ✅ `CAPTAIN_PICK` (interactive) | `DraftSession` entity, session state management (DB-backed), 5 REST endpoints at `/api/draft-sessions`, `V4__draft_sessions.sql` migration |

### Phase 5 — Future (High complexity, optional)

| Algorithm | Notes |
|-----------|-------|
| 🔮 `POSITION_BASED` | Requires player position data (new Flyway migration + enum) |

---

## 10. Open Questions — ✅ All Resolved

| # | Question | ✅ Answer |
|---|----------|---------:|
| 1 | Phase 1 includes BALANCED, RANDOM, SNAKE_DRAFT? | **Yes — all three** |
| 2 | formWindow for FORM_BASED: per-request or admin config? | **Per-request param, default 5** |
| 3 | Hot streak threshold for STREAK_AWARE? | **currentStreak >= 3** |
| 4 | Swap tolerance (rating delta) for STREAK_AWARE? | **0.5 rating points** |
| 5 | Return `algorithmNotes` in MatchDTO? | **Yes — `generationNotes` field added to MatchDTO + V6 migration** |
| 6 | Re-generate endpoint allowed? | **Yes — POST /api/match-plans/{id}/generate (stateless preview)** |
| 7 | Generate flow: immediate or preview-first? | **Preview first — confirm with second call** |
| 8 | CAPTAIN_PICK scope? | **Implemented as server-side snake draft simulation — Phase 3. Full interactive DraftSession API is Phase 4.** |

---

*Decisions locked 2026-05-16 — implementation begins immediately.*




