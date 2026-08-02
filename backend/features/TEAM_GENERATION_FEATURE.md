# Team Generation Feature

**Added in:** v4.0.0 (Phase 1) · v4.1.0 (Phase 2 FORM_BASED) · v4.2.0 (Phase 3 CAPTAIN_PICK) · v4.3.0 (Phase 4 Draft Session)
**Date:** May 16, 2026  
**Status:** ✅ Released (BALANCED, RANDOM, SNAKE_DRAFT, FORM_BASED, CAPTAIN_PICK) · ⚠️ Pending (STREAK_AWARE) · 🔮 Future (POSITION_BASED)  
**Depends on:** Match Plans & Availability Poll, Player.skillRating, PlayerStat history (FORM_BASED)

---

## Overview

**Team Generation** is the process of automatically distributing confirmed players into two
balanced (or intentionally unbalanced) teams before a match. It sits between the availability
poll phase and actual match creation.

There are **two ways** to generate teams:

1. **Stateless Preview + Confirm** (all types except CAPTAIN_PICK interactive mode) — call
   `POST /api/match-plans/{id}/generate` to get a preview `MatchPreviewDTO`, inspect it,
   then call `POST /api/match-plans/{id}/generate/confirm` to create the `Match` record.
   This is the primary flow for all server-computed generation types.

2. **Interactive Draft Session** (`/api/draft-sessions`) — a human-driven, real-time pick
   session where captains select players one at a time via API with SSE event streaming for
   spectators. Results in a persisted `Match` when confirmed.

The generation type is stored on the `Match` entity as `generationType` and a human-readable
summary is stored in `generationNotes` for every non-MANUAL match.

---

## Quick Reference

| Type | Status | Auth | Data Required | When to Use | Extra Params |
|------|:------:|------|---------------|-------------|--------------|
| `MANUAL` | ✅ Active | ADMIN / MASTER | None — caller provides IDs | Full control over team composition | None |
| `BALANCED` | ✅ Active | ADMIN / MASTER | `skillRating` only | Default go-to for casual matches | None |
| `RANDOM` | ✅ Active | ADMIN / MASTER | None | Fun/casual games where fairness is not the goal | None |
| `SNAKE_DRAFT` | ✅ Active | ADMIN / MASTER | `skillRating` only | Transparent, explainable distribution players enjoy | None |
| `FORM_BASED` | ✅ Active | ADMIN / MASTER | `skillRating` + match history | Regular groups playing weekly where recent form matters | `formWindow` (int, default 5) |
| `STREAK_AWARE` | ⚠️ Pending | ADMIN / MASTER | `skillRating` + streak data | Preventing momentum clusters — not yet available | None |
| `CAPTAIN_PICK` | ✅ Active | ADMIN / MASTER | `skillRating` | Social engagement, when captains are meaningful | `captainAId`, `captainBId` (Long, optional) |

> **MANUAL** is used via `POST /api/matches` directly (the caller provides `playerIds` per team).
> All other types use the `POST /api/match-plans/{id}/generate` flow.

---

## Prerequisites — Player Counts

The match plan must have **exactly** the following number of `CONFIRMED` players for its `MatchType` before generation can proceed:

| MatchType | Players per Team | Total Confirmed Required |
|-----------|:----------------:|:------------------------:|
| `FIVE_A_SIDE` | 5 | 10 |
| `SEVEN_A_SIDE` | 7 | 14 |
| `ELEVEN_A_SIDE` | 11 | 22 |

If player count does not match, the generate endpoint returns `400 Bad Request`.

---

## The Generation Flow

### Step-by-Step: Stateless Preview + Confirm

```
① POST /api/match-plans          → create a match plan (status: PENDING)
        │
② PATCH /api/match-plans/{id}/players/{playerId}/confirmation
        │                        → players confirm availability
        │                          (repeat until required count confirmed)
        │
③ PATCH /api/match-plans/{id}/status  { "status": "CONFIRMED" }
        │                        → lock the plan (optional but recommended)
        │
④ POST /api/match-plans/{id}/generate?generationType=BALANCED
        │                        → returns MatchPreviewDTO (not persisted)
        │                          can be called multiple times with different types
        │
⑤ inspect MatchPreviewDTO        → review both team compositions and avg ratings
        │
⑥ POST /api/match-plans/{id}/generate/confirm?generationType=BALANCED
                                 → creates and persists the Match record
                                   returns MatchDTO (201 Created)
```

> Steps ④ and ⑤ can be repeated as many times as needed — the preview is completely
> stateless and does not affect any persisted data. Only the `/confirm` call creates a Match.

### Step-by-Step: Interactive Draft Session

```
① (same as above through ③)

④ POST /api/draft-sessions       → create a DraftSession linked to a CONFIRMED match plan
        │                          returns DraftSessionDTO with captains and available players
        │
⑤ GET  /api/draft-sessions/{id}/events
        │                        → SSE stream — spectators subscribe here for real-time updates
        │
⑥ POST /api/draft-sessions/{id}/pick  { "playerId": 42 }
        │                        → captain makes a pick; SSE PICK_MADE event fires
        │                          (alternating between Team A and Team B captains)
        │
    (repeat until all players picked — SSE SESSION_COMPLETED fires)
        │
⑦ POST /api/draft-sessions/{id}/confirm
                                 → persists the Draft result as a Match record
                                   returns MatchDTO (201 Created)
```

See [`docs/features/DRAFT_SESSION_FEATURE.md`](DRAFT_SESSION_FEATURE.md) and
[`docs/frontend/DRAFT_SESSION_SSE_GUIDE.md`](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) for the
full interactive draft documentation.

---

## API Endpoint Reference

### Stateless Generation — `/api/match-plans`

**Auth:** Bearer JWT · Role: `ADMIN_USER` or `MASTER_USER`

#### Step 1 — Preview (not persisted)

```
POST /api/match-plans/{id}/generate
```

| Parameter | Location | Type | Default | Description |
|-----------|----------|------|---------|-------------|
| `id` | Path | Long | — | ID of the match plan |
| `generationType` | Query | String | `"BALANCED"` | One of: `BALANCED`, `RANDOM`, `SNAKE_DRAFT`, `FORM_BASED`, `STREAK_AWARE`, `CAPTAIN_PICK` |
| `params[formWindow]` | Query | int | `5` | **FORM_BASED only** — number of recent matches to consider |
| `params[captainAId]` | Query | Long | auto | **CAPTAIN_PICK only** — player ID for Team A captain |
| `params[captainBId]` | Query | Long | auto | **CAPTAIN_PICK only** — player ID for Team B captain |

**Response:** `200 OK` → `MatchPreviewDTO`

```json
{
  "matchPlanId": 7,
  "generationType": "BALANCED",
  "generationNotes": "BALANCED (greedy): avgA=7.53 avgB=7.50 Δ=0.03",
  "teamA": {
    "players": [
      { "playerId": 1, "playerName": "João Silva", "skillRating": 9.1 },
      { "playerId": 4, "playerName": "Rui Ferreira", "skillRating": 8.2 }
    ],
    "avgRating": 7.53
  },
  "teamB": {
    "players": [
      { "playerId": 2, "playerName": "Miguel Santos", "skillRating": 9.0 },
      { "playerId": 3, "playerName": "André Costa", "skillRating": 8.7 }
    ],
    "avgRating": 7.50
  }
}
```

#### Step 2 — Confirm (creates the Match)

```
POST /api/match-plans/{id}/generate/confirm
```

| Parameter | Location | Type | Default | Description |
|-----------|----------|------|---------|-------------|
| `id` | Path | Long | — | ID of the match plan |
| `generationType` | Query | String | `"BALANCED"` | Same options as preview |
| `params[...]` | Query | Map | — | Same algorithm-specific params as preview |

**Response:** `201 Created` → `MatchDTO`

> The same algorithm is re-run at confirm time to produce the final team assignment.
> The preview and confirm calls are idempotent for deterministic types (BALANCED,
> SNAKE_DRAFT, FORM_BASED, CAPTAIN_PICK with fixed captains). RANDOM will produce
> a different shuffle each time.

---

### Interactive Draft Session — `/api/draft-sessions`

**Auth:** Bearer JWT

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/draft-sessions` | Any authenticated | List all draft sessions |
| `POST` | `/api/draft-sessions` | ADMIN / MASTER | Create a new draft session for a CONFIRMED match plan |
| `GET` | `/api/draft-sessions/{id}` | Any authenticated | Get session state by ID |
| `GET` | `/api/draft-sessions/{id}/events` | Any authenticated | Subscribe to real-time SSE events |
| `POST` | `/api/draft-sessions/{id}/pick` | Any authenticated | Submit a pick for the current captain's turn |
| `POST` | `/api/draft-sessions/{id}/confirm` | ADMIN / MASTER | Confirm completed draft → creates Match (201) |
| `DELETE` | `/api/draft-sessions/{id}` | ADMIN / MASTER | Cancel an OPEN or COMPLETED session |

**Session States:** `OPEN` → `COMPLETED` → *(confirmed → Match created)*

**SSE Event Types:**

| Event | When Fired | Payload |
|-------|-----------|---------|
| `CONNECTED` | On subscribe | Full current session snapshot |
| `PICK_MADE` | After each successful pick | Updated team lists + whose turn next |
| `SESSION_COMPLETED` | All players picked | Final team A and B compositions |
| `SESSION_CANCELLED` | Session deleted | Cancellation notice |

---

## GenerationType Deep-Dives

---

### 🟢 MANUAL

**Status:** ✅ Active  
**Flow:** Direct match creation via `POST /api/matches` — no match plan required.

The caller explicitly provides `playerIds` for each team in `MatchCreateDTO`. The server
validates player count matches the `matchType` but does not reorder or rebalance.

**When to use:** When an admin knows exactly who goes on which team and wants full control.

**generationNotes:** `null` (no algorithm was applied)

**Example:**

```bash
curl -X POST http://localhost:8080/api/matches \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Training Match",
    "matchType": "SEVEN_A_SIDE",
    "generationType": "MANUAL",
    "seasonId": 1,
    "teams": [
      { "name": "Team A", "playerIds": [1, 2, 3, 4, 5, 6, 7] },
      { "name": "Team B", "playerIds": [8, 9, 10, 11, 12, 13, 14] }
    ]
  }'
```

---

### 🟢 BALANCED

**Status:** ✅ Active — Phase 1  
**Extra Params:** None

#### Algorithm

Greedy bin-packing by `skillRating`. Players are sorted descending by skill rating, then each
player is assigned to whichever team currently has the **lower cumulative rating sum**.
Tie-break: Team A.

```
Players sorted DESC: [9.5, 8.2, 7.8, 7.1, 6.5, 6.0, 5.8, 5.5, 5.1, 4.9, 4.7, 4.5, 4.2, 4.0]

Iteration:
  9.5 → Team A (both empty, A wins tie)   sumA=9.5   sumB=0.0
  8.2 → Team B (lower sum)               sumA=9.5   sumB=8.2
  7.8 → Team B (lower sum)               sumA=9.5   sumB=16.0
  7.1 → Team A (lower sum)               sumA=16.6  sumB=16.0
  6.5 → Team B (lower sum)               sumA=16.6  sumB=22.5
  6.0 → Team A (lower sum)               sumA=22.6  sumB=22.5
  ...

Result: avgA=7.53  avgB=7.50  Δ=0.03 ✅
```

**Complexity:** O(n log n) — sort dominates  
**Worst-case imbalance:** ≤ `maxRating / (n+1)`. For n=14 players and max rating 10.0: ≤ 0.67 rating points. In practice, typically < 0.10.

**When to use:** The default choice for most casual or competitive matches. Fast, deterministic,
and produces tightly balanced teams purely from current skill ratings.

**Limitations:** Does not account for recent form, winning streaks, or player positions.

**Example curl:**

```bash
# Preview
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=BALANCED" \
  -H "Authorization: Bearer <token>"

# Confirm
curl -X POST "http://localhost:8080/api/match-plans/7/generate/confirm?generationType=BALANCED" \
  -H "Authorization: Bearer <token>"
```

**generationNotes output:**
```
BALANCED (greedy): avgA=7.53 avgB=7.50 Δ=0.03
```

---

### 🟢 RANDOM

**Status:** ✅ Active — Phase 1  
**Extra Params:** None

#### Algorithm

All confirmed players are shuffled randomly using `Collections.shuffle()`. The first half of
the shuffled list becomes Team A; the second half becomes Team B. No skill balancing is
applied — the result is deliberately unpredictable.

```
Confirmed players: [P1, P2, P3, P4, P5, P6, P7, P8, P9, P10, P11, P12, P13, P14]
After shuffle:     [P8, P3, P11, P1, P13, P6, P4, P10, P2, P7, P14, P5, P12, P9]
Team A (first 7):  [P8, P3, P11, P1, P13, P6, P4]
Team B (last 7):   [P10, P2, P7, P14, P5, P12, P9]
```

**Complexity:** O(n)  
**Balance guarantee:** None. Rating difference could be 3.0+ points on a bad shuffle.

**When to use:** Casual / fun matches where "let fate decide" is part of the game. Great for
holiday games, parties, or when players explicitly want the "chaos" element.

> ⚠️ **Note:** Because RANDOM uses a fresh shuffle on each call, the preview result will differ
> from the confirm result. If you need the preview composition to match the confirmed match,
> use BALANCED, SNAKE_DRAFT, FORM_BASED, or CAPTAIN_PICK with fixed captain IDs.

**Example curl:**

```bash
# Preview (new shuffle each time)
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=RANDOM" \
  -H "Authorization: Bearer <token>"

# Confirm (produces a DIFFERENT shuffle — this is expected)
curl -X POST "http://localhost:8080/api/match-plans/7/generate/confirm?generationType=RANDOM" \
  -H "Authorization: Bearer <token>"
```

**generationNotes output:**
```
RANDOM: avgA=6.80 avgB=7.23 (no balance guarantee)
```

---

### 🟢 SNAKE_DRAFT

**Status:** ✅ Active — Phase 1  
**Extra Params:** None

#### Algorithm

Players are sorted descending by `skillRating`. Picks are assigned using a **snake pattern** —
alternating which team leads each pair of picks:

```
Pick #:   1   2   3   4   5   6   7   8   9  10  11  12  13  14
Team:     A   B   B   A   A   B   B   A   A   B   B   A   A   B
```

Pattern rule: picks are grouped in pairs (1-2, 3-4, 5-6, …). Even pair groups (0, 2, 4…)
are led by Team A; odd pair groups (1, 3, 5…) are led by Team B.

**Visual example with 14 players:**
```
Sorted by rating: P1(9.8), P2(9.1), P3(8.7), P4(8.2), P5(7.9), P6(7.5),
                  P7(7.1), P8(6.9), P9(6.5), P10(6.2), P11(5.8), P12(5.5),
                  P13(5.1), P14(4.9)

Team A picks: P1(1st), P4(4th), P5(5th), P8(8th), P9(9th), P12(12th), P13(13th)
Team B picks: P2(2nd), P3(3rd), P6(6th), P7(7th), P10(10th), P11(11th), P14(14th)

Result: avgA=7.06  avgB=7.11  Δ=0.05 ✅
```

**Complexity:** O(n log n) — sort dominates  
**Balance:** Mathematically near-optimal; the snake pattern distributes top and bottom
players fairly across both teams.

**When to use:** When players want a transparent distribution they can follow and reason about.
Popular with groups that understand or enjoy the concept of a "draft." Produces slightly
different team compositions than BALANCED while achieving similar balance.

**Example curl:**

```bash
# Preview
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=SNAKE_DRAFT" \
  -H "Authorization: Bearer <token>"

# Confirm
curl -X POST "http://localhost:8080/api/match-plans/7/generate/confirm?generationType=SNAKE_DRAFT" \
  -H "Authorization: Bearer <token>"
```

**generationNotes output:**
```
SNAKE_DRAFT: avgA=7.06 avgB=7.11 Δ=0.05
```

---

### 🟢 FORM_BASED

**Status:** ✅ Active — Phase 2  
**Extra Params:** `params[formWindow]` (int, ≥ 1, default: `5`)

#### Algorithm

Instead of using raw `skillRating`, FORM_BASED computes a **form score** for each player
from their recent match history, then applies the same BALANCED greedy algorithm using form
scores as the balancing metric.

**Form score formula (linear decay):**

```
For each player, fetch last formWindow completed rated matches (newest → oldest):
  ratings = [r₀, r₁, r₂, ..., rₙ₋₁]   (r₀ = most recent)

  weightedSum = Σ( rᵢ × (ratings.size() - i) )
  totalWeight = Σ( ratings.size() - i )  for i in 0..ratings.size()-1

  formScore = weightedSum / totalWeight
```

The most recent match has the highest weight; weights decrease linearly towards older matches.

**Example — N=3 window, ratings=[8.5, 7.8, 7.0] (newest→oldest):**
```
weights     = [3, 2, 1]
weightedSum = (8.5 × 3) + (7.8 × 2) + (7.0 × 1) = 25.5 + 15.6 + 7.0 = 48.1
totalWeight = 3 + 2 + 1 = 6
formScore   = 48.1 / 6 ≈ 8.02
```

**Fallback:** If a player has no rated match history, their `skillRating` is used as the
form score directly. This means new players are treated as per their static rating for
the purpose of team formation.

**DB Impact:** Fetches up to `formWindow` rows per player from `player_stats` via
`PlayerStatRepository.findRecentRatedByPlayerId(playerId, Pageable)`. For 14 players at
`formWindow=5`: up to 70 rows. Mitigated by `Pageable` — only the required rows are read.

**Complexity:** O(n log n + n × formWindow DB reads)

**When to use:** Regular groups where the same players compete weekly and recent form is
a better predictor of match outcome than the historical `skillRating`. Prevents the
"always balanced on paper, but one team keeps winning" problem.

**Limitations:** Players with fewer than `formWindow` completed matches use their `skillRating`
as a substitute. The `formWindow` is per-request — different values produce different results.

**Example curl:**

```bash
# Preview with default window (5)
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=FORM_BASED" \
  -H "Authorization: Bearer <token>"

# Preview with custom window of 3
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=FORM_BASED&params[formWindow]=3" \
  -H "Authorization: Bearer <token>"

# Confirm with window of 3
curl -X POST "http://localhost:8080/api/match-plans/7/generate/confirm?generationType=FORM_BASED&params[formWindow]=3" \
  -H "Authorization: Bearer <token>"
```

**generationNotes output:**
```
FORM_BASED (window=5): avgFormA=7.81 avgFormB=7.78 Δ=0.03
```

> **Note:** The notes show average **form scores** (`avgFormA`/`avgFormB`), not average skill
> ratings. These may differ from what `GET /api/players` returns for `skillRating`.

---

### ⚠️ STREAK_AWARE

**Status:** ⚠️ Placeholder — returns `422 Unprocessable Entity`  
**Extra Params:** None (planned)

#### What it Should Do

STREAK_AWARE is designed as a two-pass algorithm:

1. **Pass 1:** Generate teams using the BALANCED greedy algorithm (by `skillRating`).
2. **Pass 2:** Apply a streak separation post-pass. Define "hot" as `currentStreak >= +3`.
   If hot players cluster on one team, attempt to swap a hot player with a cold player from
   the other team, *provided* the rating delta from the swap is ≤ 0.5.

**Planned swap logic:**
```
hotA = count of hot players in Team A
hotB = count of hot players in Team B

While |hotA - hotB| > 1:
    Pick the team with more hot players
    Find a hot player H (rating = rH) on that team
    Find the closest-rated cold player C (rating = rC) on the other team
    If |rH - rC| ≤ 0.5:
        Swap H and C
        Recalculate hotA, hotB
    Else:
        Break — accept imbalance (no valid swap within threshold)
```

**Blocking issue:** `CalculationService` must be reliably populating
`players.current_streak` before this algorithm has meaningful input data.
Until that prerequisite is confirmed stable, any call to STREAK_AWARE returns:

```json
HTTP 422 Unprocessable Entity
{
  "message": "STREAK_AWARE generation is not yet available. It will be enabled once CalculationService is live."
}
```

**When to use (once active):** Complements BALANCED when a group tracks win/loss streaks
and wants to prevent hot players from stacking on the same team, making matches more competitive.

---

### 🟢 CAPTAIN_PICK

**Status:** ✅ Active — Phase 3 (server-side simulation)  
**Extra Params:** `params[captainAId]` (Long, optional) · `params[captainBId]` (Long, optional)

> **Important distinction:** This is a **server-side snake draft simulation** — it runs instantly and
> returns a complete team assignment, similar to other generation types. It is NOT the same as
> the interactive `DraftSession` flow (which involves real-time human picks via `/api/draft-sessions`).

#### Algorithm

**Step 1 — Resolve Captains:**
```
If params[captainAId] is supplied:
    Captain A = player with that ID from the confirmed pool
    (throws 400 if not found in pool)
Else:
    Captain A = highest skillRating player in pool

If params[captainBId] is supplied:
    Captain B = player with that ID from the confirmed pool
    (throws 400 if not found in pool)
Else:
    Captain B = highest skillRating player in pool, excluding Captain A
```

**Step 2 — Seed teams:**
```
teamA = [captainA]
teamB = [captainB]
```

**Step 3 — Snake draft from remaining players (sorted DESC by skillRating):**
```
remaining = pool − {captainA, captainB}, sorted by skillRating DESC

for i in 0..remaining.size()-1:
    round       = i / 2          (integer division, 0-indexed)
    posInRound  = i % 2
    aLeadsRound = (round % 2 == 0)

    assign to Team A if: (aLeadsRound AND posInRound == 0) OR (NOT aLeadsRound AND posInRound == 1)
    assign to Team B otherwise

Pattern for 12 remaining players (7-a-side with 2 captains already assigned):
  i:    0  1  2  3  4  5  6  7  8  9 10 11
  team: A  B  B  A  A  B  B  A  A  B  B  A
```

**Example for SEVEN_A_SIDE (14 players total, 2 captains + 12 remaining):**
```
Captain A auto-selected: João Silva (skillRating=9.5)
Captain B auto-selected: Miguel Santos (skillRating=9.1)
Remaining sorted: [8.7, 8.2, 7.9, 7.5, 7.1, 6.9, 6.5, 6.2, 5.8, 5.5, 5.1, 4.9]

Team A: Captain(9.5) + picks[0,3,4,7,8,11] → avg = 7.43
Team B: Captain(9.1) + picks[1,2,5,6,9,10] → avg = 7.21
Δ = 0.22
```

**Complexity:** O(n log n)

**Error cases:**

| Scenario | Response |
|----------|----------|
| `captainAId` not in confirmed pool | `400 Bad Request` |
| `captainBId` not in confirmed pool | `400 Bad Request` |
| Same player specified for both captains | `captainBId` is ignored; Captain B auto-selects next-highest |

**When to use:** When the group has natural "captain" figures and enjoys a draft-narrative, but
you want it resolved instantly by the server rather than waiting for human input. The captain
identities are reflected in `generationNotes`, so players can see "their" captain's name.

**Example curl:**

```bash
# Auto-select captains (top 2 by rating)
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=CAPTAIN_PICK" \
  -H "Authorization: Bearer <token>"

# Specify both captains
curl -X POST "http://localhost:8080/api/match-plans/7/generate?generationType=CAPTAIN_PICK&params[captainAId]=1&params[captainBId]=2" \
  -H "Authorization: Bearer <token>"

# Confirm with auto-captains
curl -X POST "http://localhost:8080/api/match-plans/7/generate/confirm?generationType=CAPTAIN_PICK" \
  -H "Authorization: Bearer <token>"
```

**generationNotes output:**
```
CAPTAIN_PICK: captainA=João Silva captainB=Miguel Santos avgA=7.43 avgB=7.21 Δ=0.22
```

---

## generationNotes Field Reference

Every generated `MatchDTO` (and `MatchPreviewDTO`) includes a `generationNotes` field
(String, nullable) that documents how the teams were formed.

| GenerationType | Format | Example |
|---|---|---|
| `MANUAL` | `null` | *(no value)* |
| `BALANCED` | `BALANCED (greedy): avgA=X avgB=Y Δ=Z` | `BALANCED (greedy): avgA=7.53 avgB=7.50 Δ=0.03` |
| `RANDOM` | `RANDOM: avgA=X avgB=Y (no balance guarantee)` | `RANDOM: avgA=6.80 avgB=7.23 (no balance guarantee)` |
| `SNAKE_DRAFT` | `SNAKE_DRAFT: avgA=X avgB=Y Δ=Z` | `SNAKE_DRAFT: avgA=7.06 avgB=7.11 Δ=0.05` |
| `FORM_BASED` | `FORM_BASED (window=N): avgFormA=X avgFormB=Y Δ=Z` | `FORM_BASED (window=5): avgFormA=7.81 avgFormB=7.78 Δ=0.03` |
| `CAPTAIN_PICK` | `CAPTAIN_PICK: captainA=<name> captainB=<name> avgA=X avgB=Y Δ=Z` | `CAPTAIN_PICK: captainA=João Silva captainB=Miguel Santos avgA=7.43 avgB=7.21 Δ=0.22` |
| `STREAK_AWARE` | N/A — returns 422 | *(not yet active)* |

> **Stored on the Match entity** in the `generation_notes VARCHAR(500)` column (added in V6 Flyway migration).
> Frontend can display this string directly as an informational label on match cards.

---

## Edge Cases & Error Responses

| Scenario | HTTP Status | Message / Behaviour |
|----------|:-----------:|---------------------|
| Match plan not found | 404 | `Match plan with id {id} not found` |
| Not enough CONFIRMED players | 400 | `Not enough confirmed players for match type {type}: required {n}, found {m}` |
| `STREAK_AWARE` called | 422 | `STREAK_AWARE generation is not yet available. It will be enabled once CalculationService is live.` |
| `captainAId` not in confirmed pool | 400 | Descriptive message indicating invalid player ID |
| `captainBId` not in confirmed pool | 400 | Descriptive message indicating invalid player ID |
| FORM_BASED player has zero match history | — | Silently falls back to `skillRating` for that player |
| `formWindow` ≤ 0 | 400 | Validation error on the `params` map |
| All players same `skillRating` | — | Any distribution is valid; BALANCED alternates starting with Team A |
| Inactive player in confirmed pool | — | `MatchService` auto-activates the player; generation uses the full pool |
| Duplicate player IDs in pool | 400 | Caught by MatchService at match creation, not at generation |
| Not ADMIN/MASTER role | 403 | Forbidden — Bearer token present but insufficient authority |
| No Bearer token | 401 | Unauthorized |

---

## Choosing the Right Type

Use this guide to select the appropriate generation type:

```
Is this a fun/casual game where balance doesn't matter?
    └── YES → RANDOM

Is player history / CalculationService data reliable?
    ├── NO  → Do players need to see a transparent draft narrative?
    │           ├── YES → SNAKE_DRAFT
    │           └── NO  → BALANCED  ← (recommended default)
    │
    └── YES → Has the match plan been running for 3+ match cycles?
                ├── NO  → BALANCED (use skillRating; form data is sparse)
                └── YES → Do recent performances matter more than career ratings?
                            ├── YES → FORM_BASED (use formWindow=3 for short-term form,
                            │                     formWindow=8 for longer trend)
                            └── NO  → Are there "captain" figures in the group?
                                        ├── YES → CAPTAIN_PICK (server-side simulation)
                                        │          or Interactive Draft Session
                                        └── NO  → BALANCED or SNAKE_DRAFT
```

**Summary recommendation table:**

| Use Case | Recommended Type |
|----------|-----------------|
| First match of a season / new group | `BALANCED` |
| Casual or fun game | `RANDOM` |
| Group wants transparency / "draft feeling" | `SNAKE_DRAFT` |
| Regular weekly group with match history | `FORM_BASED` |
| Group with designated captains | `CAPTAIN_PICK` or Interactive Draft Session |
| Preventing win-streak stacking | `STREAK_AWARE` *(once available)* |
| Admin knows best team composition | `MANUAL` |

---

## Algorithm Comparison Matrix

| Algorithm | Balance Quality | Requires History | Deterministic | Social Engagement | Complexity |
|-----------|:--------------:|:----------------:|:-------------:|:-----------------:|:----------:|
| `MANUAL` | User-defined | ❌ | ✅ | ⭐ | None |
| `BALANCED` | ⭐⭐⭐⭐⭐ | ❌ | ✅ | ⭐⭐ | O(n log n) |
| `RANDOM` | ⭐ | ❌ | ❌ | ⭐⭐⭐⭐⭐ | O(n) |
| `SNAKE_DRAFT` | ⭐⭐⭐⭐⭐ | ❌ | ✅ | ⭐⭐⭐ | O(n log n) |
| `FORM_BASED` | ⭐⭐⭐⭐ | ✅ (N matches) | ✅ | ⭐⭐⭐ | O(n log n + n×W) |
| `STREAK_AWARE` | ⭐⭐⭐⭐ | ✅ (streaks) | ✅ | ⭐⭐⭐ | O(n log n) |
| `CAPTAIN_PICK` | ⭐⭐⭐ (human-flavoured) | ❌ | ✅* | ⭐⭐⭐⭐⭐ | O(n log n) |

*Deterministic if captain IDs are fixed; non-deterministic if auto-selected and ratings are tied.

---

## Phase Roadmap

| Phase | Features | Status |
|-------|----------|:------:|
| Phase 1 | `BALANCED`, `RANDOM`, `SNAKE_DRAFT` strategies · Strategy pattern infrastructure · `MatchPreviewDTO` · `POST /api/match-plans/{id}/generate` endpoints · V6 migration (`generation_notes` column) | ✅ Done |
| Phase 2 | `FORM_BASED` strategy · `PlayerStatRepository.findRecentRatedByPlayerId` · `STREAK_AWARE` placeholder (422) | ✅ Done (STREAK_AWARE pending) |
| Phase 3 | `CAPTAIN_PICK` server-side snake draft simulation (`CaptainPickGenerationStrategy`) | ✅ Done |
| Phase 4 | Interactive `DraftSession` API (`/api/draft-sessions`) · SSE event streaming · `V4__draft_sessions.sql` migration | ✅ Done |
| Phase 5 | `STREAK_AWARE` full implementation (blocked on CalculationService `current_streak` stability) | ⏳ Pending |
| Future | `POSITION_BASED` — requires player position data (new Flyway migration + position enum on `players` table) | 🔮 Planned |

---

## Implementation Notes

> These notes are for backend developers and architects. Skip if you are a frontend consumer.

### Architecture — Strategy Pattern

All generation algorithms implement a common `TeamGenerationStrategy` interface. A
`TeamGenerationStrategyFactory` Spring component holds all implementations (auto-injected)
and resolves the correct one by `GenerationType` at request time.

```
service/teamgeneration/
├── TeamGenerationStrategy.java         ← interface: type() + generate(context)
├── TeamGenerationContext.java          ← input record: matchType, players, params
├── TeamGenerationResult.java           ← output record: teamA, teamB, algorithmNotes
├── BalancedGenerationStrategy.java     ← BALANCED
├── RandomGenerationStrategy.java       ← RANDOM
├── SnakeDraftGenerationStrategy.java   ← SNAKE_DRAFT
├── FormBasedGenerationStrategy.java    ← FORM_BASED
├── CaptainPickGenerationStrategy.java  ← CAPTAIN_PICK (server-side)
├── StreakAwareGenerationStrategy.java  ← STREAK_AWARE (placeholder — throws 422)
└── TeamGenerationStrategyFactory.java  ← resolves strategy by GenerationType
```

Adding a new generation algorithm requires only: implementing the interface, annotating
with `@Component`, and adding the value to the `GenerationType` enum. No changes to
`MatchService` or `MatchPlanService` are required.

### Related Files

| File | Role |
|------|------|
| `MatchPlanController.java` | REST endpoints for generate + confirm |
| `MatchPlanService.java` | Orchestrates strategy call + creates Match from result |
| `TeamGenerationStrategyFactory.java` | Resolves strategy by GenerationType |
| `MatchPreviewDTO.java` | Response DTO for the stateless preview step |
| `PlayerStatRepository.java` | `findRecentRatedByPlayerId` used by FORM_BASED |
| `DraftSessionController.java` | Interactive draft session endpoints |
| `DraftSessionService.java` | Pick management, SSE event publishing, session lifecycle |

### Related Documentation

- [`docs/features/MATCH_PLANS_FEATURE.md`](MATCH_PLANS_FEATURE.md) — Full match plan & availability poll docs
- [`docs/features/MATCH_FEATURE.md`](MATCH_FEATURE.md) — Match entity, DTOs, lifecycle
- [`docs/features/DRAFT_SESSION_FEATURE.md`](DRAFT_SESSION_FEATURE.md) — Interactive draft session full reference
- [`docs/frontend/DRAFT_SESSION_SSE_GUIDE.md`](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) — Frontend SSE integration guide
- [`docs/features/TEAM_GENERATION_DESIGN.md`](TEAM_GENERATION_DESIGN.md) — Original design evaluation with algorithm analysis and architecture decision records
- [`docs/features/CALCULATION_SERVICE.md`](CALCULATION_SERVICE.md) — How `skillRating` and `currentStreak` are computed (prerequisite for FORM_BASED / STREAK_AWARE)

