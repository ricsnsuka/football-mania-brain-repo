# Admin Match Rating Recalculation — API Contract
**Date:** 2026-05-15
**Feature version:** v4.2.0 (proposed)
**Status:** ✅ IMPLEMENTED — matches shipped code (all tests green)
**Owner agent:** api-designer → dev-assistant → documentation-writer

> **Resolution of the §8 open decisions (as implemented):**
> 1. **Idempotency depth:** reverse-then-reapply everywhere; the **bulk** endpoint replays matches
>    chronologically (`matchDate ASC NULLS LAST, id ASC`) so `skillRating`/streaks reconcile across a
>    span. Single-match recalc is exact for a player's most-recent match, approximate mid-chain.
> 2. **Not-completed status code:** **409 Conflict** at the endpoint boundary
>    (`BusinessException.conflict`), defensive **400** kept inside the engine guard.
> 3. **`status` type:** kept as `String` (`"SUCCESS"` / `"FAILED"` / `"SKIPPED"`).
> 4. **Batch cap:** `@Size(max = 500)` on `matchIds`.
> 5. **Completion ordering:** existing `matchDate` (NULLS LAST) + `id` used — **no schema change**
>    (`matches.completed_at` not added).
> 6. **New finders added:** `MatchRepository.findCompletedOrdered` / `findCompletedBySeasonOrdered`,
>    `PlayerStatRepository.findCompletedByPlayerIdChronological`.


Adds **ADMIN-only** endpoints to re-run the rating engine
(`CalculationService.recalculateMatchRatings`) for a single completed match or for many
matches in bulk, **without double-counting** career aggregates, streaks, skill-rating EMA, or
skill-rating history.

---

## 1. Design Constraints Carried From the Existing Architecture

| Constraint | Source | Impact on this design |
|-----------|--------|-----------------------|
| Only **COMPLETED** matches have ratings | copilot-instructions "Matches & Teams" | Recalc must reject non-completed matches with a business error |
| `recalculateMatchRatings(Long)` was built to run **exactly once**, `AFTER_COMMIT`, fired async by `MatchEventListener` | orchestrator context + `CalculationService` L216-410 | Re-running it as-is **double-counts**: (a) EMA re-blends from an already-moved `skillRating`, (b) `totalGoals/totalAssists/totalMatchesPlayed` re-incremented, (c) streaks re-advanced, (d) duplicate `skill_rating_history` rows inserted |
| DTOs are **Java records only** | copilot-instructions §1 | All new DTOs are records |
| Exceptions via **factory methods** | copilot-instructions §5 | `ResourceNotFoundException.of("Match", id)` (404), `BusinessException.badRequest/conflict` |
| Action-style POST that mutates an existing resource returns **200 OK + result DTO** | orchestrator context | Both recalc endpoints return `200 OK`, not `201` |
| ADMIN-only ⇒ `@PreAuthorize("hasRole('ADMIN')")` | copilot-instructions Roles | Both endpoints |
| Error shape = `ApiError` from `GlobalExceptionHandler` | `GlobalExceptionHandler.ApiError` | No new error shape introduced |
| No new deps / no schema change if avoidable | orchestrator task | Achieved — see §6. One optional schema addition is **flagged**, not required |

---

## 2. Endpoints

### 2.1 `POST /api/matches/{id}/recalculate`

**Description:** Re-run the rating engine for a single completed match. Idempotent: repeated calls
converge to the same persisted state (see §6). Intended for admin correction after a stat amend
(`PATCH /api/matches/{id}/teams/{teamId}/stats/{statId}`) or an engine constant change.

**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`
**Tags:** `Matches`
**OpenAPI `@Operation` summary:** `"Recalculate ratings for a single completed match (ADMIN only)"`

#### Request

**Path Variables:**
- `{id}` — `Long` — Match ID.

**Query Parameters:** none.
**Request Body:** none.

#### Response

**Success:** `200 OK`
**Body:** `RecalculationResultDTO`
```json
{
  "matchId": 42,
  "status": "SUCCESS",
  "ratingsUpdated": 14,
  "message": "Recalculated 14 player ratings"
}
```

**Error Responses (via `ApiError`):**
| Status | Trigger | Thrown by |
|--------|---------|-----------|
| 400 | — reserved (no body validation on this endpoint) | — |
| 403 | Caller is not `ADMIN` | Spring Security → `AccessDeniedException` |
| 404 | Match `{id}` does not exist | `ResourceNotFoundException.of("Match", id)` |
| 409 | Match exists but is **not completed** | `BusinessException.conflict(...)` |
| 409 | Concurrent recalc/complete on the same match | `ObjectOptimisticLockingFailureException` (already handled) |

> **Design note — precondition status code.** The existing `recalculateMatchRatings` throws
> `BusinessException.badRequest` ("not completed yet") → 400. For the admin endpoint the "not
> completed" state is a **conflict with the resource's current state**, so this contract specifies
> **409 Conflict** via `BusinessException.conflict(...)`. dev-assistant should throw the 409 in the
> controller-facing service method **before** delegating, and keep the internal engine guard as a
> defensive 400. (Open decision — see §8.)

---

### 2.2 `POST /api/matches/recalculate`

**Description:** Bulk re-run of the rating engine across many completed matches. Processes each match
**independently** — one bad match never fails the whole batch. Returns a per-match summary.

**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`
**Tags:** `Matches`
**OpenAPI `@Operation` summary:** `"Bulk-recalculate ratings for completed matches (ADMIN only)"`

#### Request

**Body:** `BulkRecalculationRequestDTO` (optional — an empty body / `{}` means "all completed matches").

Selection precedence (documented, deterministic):
1. If `matchIds` is non-empty → recalc exactly those IDs (each validated independently).
2. Else if `seasonId` is non-null → recalc **all completed** matches in that season.
3. Else → recalc **all completed** matches, **chronologically ordered** (see §6 ordering note).

`matchIds` and `seasonId` are **mutually exclusive**; supplying both → `400` (`BusinessException.badRequest`).

```json
{
  "matchIds": [42, 43, 44],
  "seasonId": null
}
```

#### Response

**Success:** `200 OK` (returned even if some/all individual matches failed — inspect the summary)
**Body:** `BulkRecalculationResponseDTO`
```json
{
  "totalRequested": 3,
  "succeeded": 2,
  "failed": 1,
  "results": [
    { "matchId": 42, "status": "SUCCESS", "ratingsUpdated": 14, "message": "Recalculated 14 player ratings" },
    { "matchId": 43, "status": "SUCCESS", "ratingsUpdated": 14, "message": "Recalculated 14 player ratings" },
    { "matchId": 44, "status": "FAILED",  "ratingsUpdated": 0,  "message": "Match not completed" }
  ]
}
```

**Error Responses (whole-request failures only):**
| Status | Trigger | Thrown by |
|--------|---------|-----------|
| 400 | Both `matchIds` and `seasonId` supplied | `BusinessException.badRequest(...)` |
| 400 | `matchIds` contains a null/`< 1` element (bean validation) | `MethodArgumentNotValidException` → `violations[]` |
| 403 | Caller is not `ADMIN` | Spring Security |
| 404 | `seasonId` supplied but season does not exist | `ResourceNotFoundException.of("Season", seasonId)` |

> **Partial-failure semantics:** a missing match ID, a non-completed match, or an engine error on one
> match is recorded as a `FAILED` entry in `results[]` — it does **not** roll back the whole batch and
> does **not** change the HTTP status. Each match is recalculated in its **own transaction** so a
> failure never corrupts a sibling. (See §6 + §8.)

---

## 3. New DTOs (Java Records)

All live in `src/main/java/pt/rics/demo/football/dto/`.

### 3.1 `RecalculationResultDTO` (response — single match & per-item in bulk)

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `matchId` | `Long` | No | The match this result refers to |
| `status` | `String` | No | `"SUCCESS"` \| `"FAILED"` \| `"SKIPPED"` (see enum note) |
| `ratingsUpdated` | `int` | No | Count of `PlayerStat` rows whose `rating` was recomputed; `0` on failure |
| `message` | `String` | No | Human-readable outcome (`"Recalculated 14 player ratings"`, `"Match not completed"`, `"Match not found"`, `"No player stats — skipped"`) |

```java
public record RecalculationResultDTO(
        Long matchId,
        String status,
        int ratingsUpdated,
        String message
) {
    public static RecalculationResultDTO success(Long matchId, int ratingsUpdated) {
        return new RecalculationResultDTO(matchId, "SUCCESS", ratingsUpdated,
                "Recalculated " + ratingsUpdated + " player ratings");
    }
    public static RecalculationResultDTO failed(Long matchId, String message) {
        return new RecalculationResultDTO(matchId, "FAILED", 0, message);
    }
    public static RecalculationResultDTO skipped(Long matchId, String message) {
        return new RecalculationResultDTO(matchId, "SKIPPED", 0, message);
    }
}
```

> **`status` values.** `SUCCESS` = ratings recomputed. `FAILED` = precondition/engine error (not
> found, not completed, exception). `SKIPPED` = completed match with **no** `player_stats` rows (the
> engine currently `log.warn`s and returns early at `CalculationService` L237-240) — surface this
> honestly rather than reporting a false success. dev-assistant may model `status` as a small enum
> `RecalculationStatus { SUCCESS, FAILED, SKIPPED }` serialized as its name; keeping it a `String` in
> the DTO is acceptable and avoids an extra type. (Open decision — §8.)

### 3.2 `BulkRecalculationRequestDTO` (request body — `POST /api/matches/recalculate`)

| Field | Type | Required | Validation | Notes |
|-------|------|----------|-----------|-------|
| `matchIds` | `List<Long>` | No | `@Size(max = 500)`, elements `@Positive` (see note) | Explicit list; empty/null = not selected by list |
| `seasonId` | `Long` | No | `@Positive` | Recalc all completed matches in this season |

```java
public record BulkRecalculationRequestDTO(
        @Size(max = 500, message = "cannot recalculate more than 500 matches per request")
        List<@NotNull @Positive Long> matchIds,
        @Positive Long seasonId
) {}
```

Notes:
- Both fields omitted / body absent ⇒ "recalc all completed matches".
- `matchIds` **and** `seasonId` both present ⇒ `400` (enforced in the service, not by bean validation).
- `@Size(max = 500)` is a guardrail against an unbounded admin batch; tune with orchestrator (§8).
- Element-level `@Positive`/`@NotNull` requires Hibernate Validator container-element validation
  (already on classpath) — no new dependency.

### 3.3 `BulkRecalculationResponseDTO` (response — `POST /api/matches/recalculate`)

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `totalRequested` | `int` | No | Number of matches the request resolved to |
| `succeeded` | `int` | No | Count with `status == SUCCESS` |
| `failed` | `int` | No | Count with `status == FAILED` |
| `results` | `List<RecalculationResultDTO>` | No | One entry per processed match, in processing order |

```java
public record BulkRecalculationResponseDTO(
        int totalRequested,
        int succeeded,
        int failed,
        List<RecalculationResultDTO> results
) {
    public static BulkRecalculationResponseDTO from(List<RecalculationResultDTO> results) {
        int ok  = (int) results.stream().filter(r -> "SUCCESS".equals(r.status())).count();
        int bad = (int) results.stream().filter(r -> "FAILED".equals(r.status())).count();
        return new BulkRecalculationResponseDTO(results.size(), ok, bad, results);
    }
}
```

> `succeeded + failed` may be `< totalRequested` when `SKIPPED` items exist. This is intentional and
> documented; consumers should not assume `succeeded + failed == totalRequested`.

---

## 4. Authorization Summary

| Endpoint | `@PreAuthorize` |
|----------|-----------------|
| `POST /api/matches/{id}/recalculate` | `hasRole('ADMIN')` |
| `POST /api/matches/recalculate` | `hasRole('ADMIN')` |

`MANAGER` is intentionally **excluded** — recalculation rewrites historical `skillRating`,
career aggregates, and audit history, which is a system-administration action.

---

## 5. Controller Placement & OpenAPI

Both methods go into the existing `MatchController` (`@RequestMapping("/api/matches")`, `@Tag("Matches")`).
Follow the established pattern (constructor injection, `@Operation`, `@PreAuthorize` per method,
`ResponseEntity.ok(...)`).

```java
@PostMapping("/{id}/recalculate")
@Operation(summary = "Recalculate ratings for a single completed match (ADMIN only)")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<RecalculationResultDTO> recalculateMatch(@PathVariable Long id) {
    return ResponseEntity.ok(matchService.recalculateMatch(id));
}

@PostMapping("/recalculate")
@Operation(summary = "Bulk-recalculate ratings for completed matches (ADMIN only)")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<BulkRecalculationResponseDTO> recalculateMatches(
        @Valid @RequestBody(required = false) BulkRecalculationRequestDTO request) {
    return ResponseEntity.ok(matchService.recalculateMatches(request));
}
```

`@RequestBody(required = false)` lets an empty body mean "all completed matches"; the service must
null-guard the request into an effective "all" selection.

> **Route ordering:** Spring MVC matches the literal `/recalculate` and the templated
> `/{id}/recalculate` unambiguously (different path depth), so no ordering conflict with existing
> `/{id}` routes. Verified against the current `MatchController` route table.

---

## 6. ⭐ Idempotency Strategy (the critical decision)

**Problem.** `recalculateMatchRatings` is a *forward-only accumulator*. Re-running it against a match
that was already processed applies its side effects a second time:

| Side effect | Location | Re-run behaviour if unchanged | Idempotent? |
|-------------|----------|-------------------------------|-------------|
| `PlayerStat.rating` (per-match) | L351-352 | **Overwritten** with recomputed value | ✅ already idempotent |
| `Player.skillRating` (EMA) | L354-377 | Re-blends from the *already-moved* value → drifts further each run | ❌ |
| `totalGoals/totalAssists/totalMatchesPlayed` | L371-373 | Incremented **again** | ❌ |
| `currentStreak/longestStreak` | L407 / L632-649 | Advanced **again** | ❌ |
| `skill_rating_history` rows | L380-388 | Duplicate row inserted per player per run | ❌ |

### 6.1 Chosen approach — **"Reverse-then-Reapply" per match** (no schema change)

Refactor the engine so a recalc first **undoes this match's previously-recorded effect** using data
that already exists, then re-applies the effect exactly like a first-time calculation.

**Step-by-step (single match, one `@Transactional`):**

1. **Load & guard.** Load `Match` (→ 404). If not completed → 409. Load `List<PlayerStat>` for the
   match. If empty → return `SKIPPED` (mirror the current early-return warning).
2. **Detect prior run.** Load `skill_rating_history` rows for this match via
   `SkillRatingHistoryRepository.findAllByMatchId(matchId)` (already exists, L29). Index by `playerId`.
3. **Reverse phase** — for each player that has a prior history row for **this** match:
   - `skillRating -= history.changeAmount` → restores the pre-match `skillRating`
     (`ratingBefore`). *(Exactness caveat below.)*
   - `totalMatchesPlayed -= 1`
   - `totalGoals   -= (soloGoals + assistedGoals + penaltyGoals)` of this match's `PlayerStat`
   - `totalAssists -= assists` of this match's `PlayerStat`
   - Streaks: **recompute** (see §6.3) — do not attempt an arithmetic reverse.
   - **Delete** the prior `skill_rating_history` rows for this match
     (`historyRepository.deleteAll(rowsForMatch)`).
4. **Reapply phase** — run the *existing* Pass-1/Pass-2 logic verbatim (raw → proportional
   normalization → EMA blend → re-increment aggregates → insert fresh history rows). Because the
   aggregates were reversed in step 3 and stats are unchanged, the re-increment nets to the same
   values → **exactly idempotent** for aggregates + history + `PlayerStat.rating`.
5. **Return** `RecalculationResultDTO.success(matchId, stats.size())`.

The **first-ever** calculation is just this flow with an empty "prior run" set — so
`MatchEventListener`'s completion hook can call the **same** method and stay correct.

### 6.2 Idempotency guarantees by side effect

| Side effect | Mechanism | Guarantee |
|-------------|-----------|-----------|
| `PlayerStat.rating` | Recomputed + overwritten | **Exact** every run |
| `totalGoals/Assists/MatchesPlayed` | Reversed from this match's stats, then re-added | **Exact** (net-zero for unchanged stats) |
| `skill_rating_history` | Delete-this-match rows, then re-insert | **No duplicates** (replace, not append) |
| `Player.skillRating` | `-= changeAmount`, then re-EMA | **Exact for the player's most-recent match**; approximate for mid-chain historical matches — see §6.3 |
| Streaks | Recompute from ordered results | **Exact** if §6.3 recompute is implemented; otherwise a known limitation |

### 6.3 The honest caveat: `skillRating` EMA & streaks for **mid-chain** matches

`skillRating` is a cumulative EMA over the player's matches in chronological order. Reversing a single
match via `-= changeAmount` perfectly restores the pre-match value **only if no later match has moved
`skillRating` since** (i.e. it is the player's latest match). For an older match in the middle of the
chain, reversing one link and re-blending produces a value that diverges from a true replay, because
every subsequent match's `ratingBefore` is now stale. Streaks have the same ordering dependency and
cannot be reversed from a single match at all.

Two implementable resolutions — **dev-assistant/orchestrator must pick one (§8):**

- **Option A (MVP, per-match reverse-reapply as above).**
  Exact for aggregates, history, and `PlayerStat.rating`. `skillRating`/streaks exact for the latest
  match, approximate for mid-chain matches. Simplest; no schema change. Recommended default for the
  single-match endpoint's typical use (correcting the most recent match after a stat amend).

- **Option B (fully correct: per-player chronological replay).**
  Determine the set of **affected players**, then for each rebuild `skillRating`, streaks, and
  aggregates deterministically by replaying **all** their completed matches in order:
  - Reset baseline: `skillRating = baseSkillRating` (as double), `totalGoals/Assists/MatchesPlayed = 0`,
    `currentStreak = longestStreak = 0`.
  - Delete all **match-linked** `skill_rating_history` rows for the player (preserve season-transition
    rows where `match IS NULL`).
  - Replay each completed match (ordered by completion time) through the same per-match math,
    re-inserting history.
  This is the "recompute `skillRating` from the full history chain" option. Correct for all cases,
  including streaks and mid-chain edits, with **no schema change**, but heavier and it must interleave
  correctly with season-transition history rows. Recommended when bulk-recalculating an entire season
  or after changing engine constants.

**Recommendation:** implement the engine so the single-match endpoint uses **Option A** (reverse-reapply)
and the bulk endpoint, when selecting "all" or "by season", uses **Option B** (per-affected-player
replay) so a full-season recompute is exact. dev-assistant should extract a private
`recalculateSingleMatch(matchId)` (Option A) and a `replayPlayers(Set<Long>)` helper (Option B) and
compose them. If time-boxing forces one path, ship Option A everywhere and document the mid-chain
approximation as a known limitation.

### 6.4 Ordering & the "match completion time" question

Option B replay requires a deterministic chronological order of a player's completed matches. The
`Match` entity's completion ordering must be confirmed by dev-assistant (candidates: a
`completedAt`/`updatedAt` timestamp, or `matchDate`, or `id` as a monotonic proxy). **If no reliable
completion-order column exists, that is the one place a schema addition (`matches.completed_at`) may be
justified — FLAGGED for the orchestrator (§8).** Option A does **not** need this.

### 6.5 Transaction boundaries

- **Single-match endpoint:** the whole reverse-reapply runs in one `@Transactional` method → all-or-nothing.
- **Bulk endpoint:** iterate match IDs and call the single-match service method **per match in its own
  transaction** (`REQUIRES_NEW`, or a self-invocation-safe separate bean) so one failure is caught,
  recorded as a `FAILED` result, and never rolls back siblings. The bulk orchestration method itself
  must **not** be `@Transactional`, or must swallow per-match exceptions outside the inner transaction.
  dev-assistant: beware Spring self-invocation — the per-match transactional method must be invoked
  through a proxy (separate bean or `TransactionTemplate`).

### 6.6 Cache eviction

Recalculation mutates `players`, rankings, leaderboards, and match ratings. dev-assistant must evict
the `players`, `playerProfile`, `matches`, `rankings`, and `leaderboards` caches after a
recalc (per the caching rules in copilot-instructions). Bulk: evict once at the end.

### 6.7 Async vs sync

The existing completion path is async (`AFTER_COMMIT` via `MatchEventListener`). The admin endpoints are
**synchronous request/response** so the admin sees the result immediately. This is a deliberate
difference — the endpoints call the engine directly, not via the event.

---

## 7. Breaking Changes

- [x] **No breaking changes to existing FE-visible contracts.** Two brand-new endpoints and three new
  response/request DTOs; no existing endpoint, DTO field, or status code is modified.
- ⚠️ **Behavioural (internal):** `CalculationService.recalculateMatchRatings` must be refactored to be
  idempotent (reverse-then-reapply). The completion path keeps identical *first-run* behaviour, so no
  externally observable change for normal match completion. test-engineer must add a
  "run-twice-equals-run-once" regression test.

### Frontend migration notes
No FE changes required to existing screens. New admin tooling may add "Recalculate" (per match) and
"Recalculate season / all" (bulk) actions, both ADMIN-gated.

---

## 8. Open Decisions for dev-assistant / orchestrator

1. **Idempotency depth (§6.3):** Option A everywhere (simpler, mid-chain approximate) **vs** A for
   single + B (per-player replay) for bulk (exact). **Recommended: A + B split.**
2. **Not-completed status code (§2.1):** 409 Conflict (this contract's choice) vs the engine's current
   400. Recommend **409** at the endpoint boundary.
3. **`status` type:** `String` in DTO (simplest) vs a `RecalculationStatus` enum. Recommend enum
   serialized by name; DTO field may stay `String`.
4. **Batch cap:** `@Size(max = 500)` on `matchIds` and behaviour of an unbounded "all" recalc
   (consider a hard ceiling / async job if match count is very large). Confirm with orchestrator.
5. **Completion ordering for Option B (§6.4):** confirm a reliable chronological column exists; if not,
   **FLAGGED**: add `matches.completed_at` via a new Flyway migration (`db-migration` agent) — the only
   potential schema change, and only if Option B is chosen and no ordering column exists.
6. **New repository methods needed** (dev-assistant to add):
   - `List<Match> findAllByCompletedTrueOrderBy...` (bulk "all", ordered) — only a `Page` variant exists today.
   - `List<Match> findAllBySeasonIdAndCompletedTrue(Long seasonId)` (bulk by season, unpaged list).
   - Existing reusable: `SkillRatingHistoryRepository.findAllByMatchId`, `PlayerStatRepository.findAllByMatchId`,
     `MatchRepository.findById`, `SeasonRepository.findById`.
7. **Season existence check:** bulk with `seasonId` should 404 if the season doesn't exist, even when
   it has zero completed matches (return an empty-but-valid summary otherwise).

---

## Summary for Next Agent

**Endpoints (final):**

| Method | Path | Auth | Success | Errors |
|--------|------|------|---------|--------|
| `POST` | `/api/matches/{id}/recalculate` | `hasRole('ADMIN')` | `200 OK` + `RecalculationResultDTO` | 403; 404 (match missing); 409 (not completed / optimistic lock) |
| `POST` | `/api/matches/recalculate` | `hasRole('ADMIN')` | `200 OK` + `BulkRecalculationResponseDTO` | 400 (both `matchIds`+`seasonId`, or invalid element); 403; 404 (season missing). Per-match failures are reported in `results[]`, **not** as HTTP errors |

**DTO records (all in `dto/`):**
```java
public record RecalculationResultDTO(Long matchId, String status, int ratingsUpdated, String message) { /* success/failed/skipped factories */ }

public record BulkRecalculationRequestDTO(
        @Size(max = 500) List<@NotNull @Positive Long> matchIds,
        @Positive Long seasonId) {}

public record BulkRecalculationResponseDTO(int totalRequested, int succeeded, int failed,
        List<RecalculationResultDTO> results) { /* from(results) factory */ }
```
`status` ∈ `{ "SUCCESS", "FAILED", "SKIPPED" }`. Bulk selection precedence: `matchIds` → `seasonId` → all completed. `matchIds` + `seasonId` together = 400. Empty/absent body = all completed.

**Chosen idempotency strategy (step-by-step), single match, one transaction:**
1. Load match (404), assert completed (409), load `PlayerStat`s (empty ⇒ `SKIPPED`).
2. Load prior `skill_rating_history` for the match (`findAllByMatchId`), index by player.
3. **Reverse** each prior effect: `skillRating -= changeAmount`; `totalMatchesPlayed -= 1`;
   `totalGoals -= match goals`; `totalAssists -= match assists`; delete this match's history rows;
   recompute streaks (don't reverse arithmetically).
4. **Reapply** the existing Pass-1/Pass-2 engine (recompute `PlayerStat.rating` via proportional
   normalization → EMA blend from restored `skillRating` → re-increment aggregates → insert fresh
   history rows → update streaks).
5. Evict `players/playerProfile/matches/rankings/leaderboards` caches; return
   `RecalculationResultDTO.success(matchId, count)`.

Guarantees: `PlayerStat.rating`, aggregates, and history are **exactly idempotent**. `skillRating` &
streaks are exact for the player's latest match; for **mid-chain** matches use **Option B**
(per-affected-player chronological replay from `baseSkillRating`) — recommended for the bulk/season path.

**Bulk transaction rule:** process each match in its **own** transaction (`REQUIRES_NEW` / separate
bean / `TransactionTemplate`); catch per-match errors → `FAILED` result; the outer loop is not a single
transaction. Beware Spring self-invocation.

**Open decisions dev-assistant must make:** (1) Option A-only vs A+B split [rec: A+B]; (2) 409 vs 400
for not-completed [rec: 409]; (3) `String` vs enum `status`; (4) batch cap; (5) **FLAGGED** — Option B
needs a reliable match completion-order column; add `matches.completed_at` (new Flyway migration) only
if none exists; (6) add `List` repository finders for "all completed" and "completed by season".

🔜 Ready for: **dev-assistant** (implementation) and, only if decision #5 requires it, **db-migration**
(`matches.completed_at`). Also update `.github/copilot-instructions.md` Match endpoint table + DTO
sections (done in this stage).


