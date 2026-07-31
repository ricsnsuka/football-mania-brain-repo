# Release Notes — v1.0.0 (Unreleased)

> Accumulated notes for the next release. Append new entries below — never delete existing ones.

---

### ⚡ Enhancement: Rating Model v2.1 — Realistic 6.0-Base Distribution

**Date:** May 8, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`

Rating Model v2 introduced proportional normalization and stats-dependent ceilings, but produced unrealistic distributions: a 1-goal contributor in a 3-1 victory scored **4.1** (too low), and a non-contributor on a winning team scored **1.8** (unrealistically harsh). Rating Model v2.1 fixes this via **compressed-range mapping** and rebalanced constants.

**Constants changed (v2 → v2.1):**
- `RAW_BASE_POINTS`: `1.0 → 7.5` (elevated base anchors non-contributors near 6.0 final)
- `RAW_WIN_BONUS`: `1.0 → 0.4` (smaller relative to elevated base)
- `RAW_LOSS_PENALTY`: `0.75 → 2.2` (larger to ensure losing non-contributors land ~5.0-5.5)
- `RATING_FLOOR`: **new** `4.0` (lower bound of compressed mapping)
- `CEILING_MIN`: `6.5 → 8.0` (compressed ceiling band)
- `CEILING_MAX`: `10.0 → 9.5` (eliminates perfect 10s)

**Formula change:**
- **v2:** `finalRating = clamp(raw / topRaw × ceiling, 1.0, 10.0)` — maps `[0, topRaw]` to `[0, ceiling]`
- **v2.1:** `finalRating = 4.0 + (raw / topRaw) × (ceiling − 4.0)` then clamped `[1.0, 10.0]` — maps `[0, topRaw]` to **`[4.0, ceiling]`**

**Results (worked example: 3-1 victory):**
- 3g+2a top performer: **9.5** (was 10.0 — compressed)
- 1-goal contributor: **~7.0** (was 4.1 — FIXED ✅)
- Non-contributor on **WIN**: **~6.1** (was 1.8 — FIXED ✅; now >6.0 as required)
- Non-contributor on **LOSS**: **~5.3** (was <2.0 — realistic)
- Non-contributor on **DRAW**: **~6.0** (neutral baseline)

**No API contract change.** `PlayerStatDTO.rating` shape unchanged. All tests green (62 tests, ~89% branch coverage). See [`docs/features/CALCULATION_SERVICE.md`](../features/CALCULATION_SERVICE.md) for full detail.

---

### 🆕 New Feature: User Entity & Authentication

**Date:** May 15, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/AuthController.java`
- `src/main/java/pt/rics/demo/football/controller/UserController.java`
- `src/main/java/pt/rics/demo/football/service/AuthService.java`
- `src/main/java/pt/rics/demo/football/service/UserService.java`
- `src/main/java/pt/rics/demo/football/service/UserDetailsServiceImpl.java`
- `src/main/java/pt/rics/demo/football/mapper/UserMapper.java`
- `src/main/java/pt/rics/demo/football/dto/LoginRequestDTO.java`
- `src/main/java/pt/rics/demo/football/dto/LoginResponseDTO.java`
- `src/main/java/pt/rics/demo/football/dto/UserDTO.java`
- `src/main/java/pt/rics/demo/football/dto/UserCreateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/UserUpdateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/AdminUserUpdateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/ChangePasswordDTO.java`

Full user authentication and management layer. Supports login via username or email,
three access roles (ADMIN/MASTER/BASIC), paginated user listing, soft deletion,
and owner-scoped password change. 30 tests — all green.

---

### 🆕 New Feature: Player Entity & CRUD API

**Date:** May 15, 2026  
**Files Changed:**
- `src/main/resources/db/migration/V1__initial_schema.sql` (the `players` table and its
  `created_by` / `updated_by` audit columns; there is no separate audit-columns migration)
- `src/main/java/pt/rics/demo/football/model/Player.java`
- `src/main/java/pt/rics/demo/football/repository/PlayerRepository.java`
- `src/main/java/pt/rics/demo/football/mapper/PlayerMapper.java`
- `src/main/java/pt/rics/demo/football/service/PlayerService.java`
- `src/main/java/pt/rics/demo/football/controller/PlayerController.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerDTO.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerCreateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerUpdateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerStatusDTO.java`

Full Player management layer. Role-based access: reads open to all authenticated users, writes restricted to `ADMIN_USER`/`MASTER_USER`, deletes restricted to `ADMIN_USER`. Business guards: `ADMIN_USER` accounts cannot be linked to players; hard delete blocked if match statistics exist. Audit trail via `created_by`/`updated_by` columns (V2 migration). Safe PATCH semantics with explicit `unlinkUser` flag. 48 tests — all green.

---

### 🆕 New Feature: Match & Team Management

**Date:** May 17, 2026  
**Key Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/MatchController.java`
- `src/main/java/pt/rics/demo/football/service/MatchService.java`
- `src/main/java/pt/rics/demo/football/model/Match.java`, `MatchTeam.java`, `PlayerStat.java`, `Goal.java`
- `src/main/java/pt/rics/demo/football/dto/MatchDTO.java`, `MatchCreateDTO.java`, `MatchCompleteDTO.java`, `MatchUpdateDTO.java`, `MatchTeamDTO.java`, `MatchTeamCreateDTO.java`, `PlayerStatDTO.java`, `PlayerStatUpdateDTO.java`

Full match lifecycle: create with manual teams, list/filter, complete with scores and per-player stats (goals, assists, own goals, MVP, rating), amend stats post-completion (ADMIN only), delete non-completed matches.  
V3 migration: adds `match_type`, `score_team_a/b`, `team_order` (unique per match), `own_goals`; drops `player_stats.notes` and `goals.penalty_caused_by`. V4 migration: optimistic locking version column on `matches`. 80 new tests (40 service + 40 controller) — all green.

---

### 🆕 New Feature: Match Plans & Player Availability Poll

**Date:** May 17, 2026  
**Key Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/MatchPlanController.java`
- `src/main/java/pt/rics/demo/football/service/MatchPlanService.java`
- `src/main/java/pt/rics/demo/football/model/MatchPlan.java`, `PlayerConfirmation.java`
- `src/main/java/pt/rics/demo/football/dto/MatchPlanCreateDTO.java`, `MatchPlanDTO.java`, `MatchPlanUpdateDTO.java`, `MatchPlanStatusDTO.java`, `PlayerConfirmationDTO.java`, `ConfirmationUpsertDTO.java`, `MatchPreviewDTO.java`

Admin/Master creates a match plan with title, match type, location, and future-validated proposed date. Opens a player availability poll. Players self-confirm or decline via `POST /confirmations/me`. Admin can override any player's status. Plan transitions: `PENDING → CONFIRMED/CANCELLED`. Computed fields: `playersNeeded`, `pollOpen`. V5 migration: adds `title`, `match_type`, `updated_at`, `created_by` to `match_plans`. 105 new tests — all green.

---

### 🆕 New Feature: Team Generation (Phase 1)

**Date:** May 17, 2026  
**Key Files Changed:**
- `src/main/java/pt/rics/demo/football/service/BalancedGenerationStrategy.java`
- `src/main/java/pt/rics/demo/football/service/RandomGenerationStrategy.java`
- `src/main/java/pt/rics/demo/football/service/SnakeDraftGenerationStrategy.java`
- `src/main/java/pt/rics/demo/football/service/TeamGenerationStrategyFactory.java`
- `src/main/java/pt/rics/demo/football/service/TeamGenerationContext.java`, `TeamGenerationResult.java`

Preview-then-confirm flow: `POST /api/match-plans/{id}/generate` returns a `MatchPreviewDTO` (stateless, not persisted). Admin reviews team balance (`teamARatingAvg`, `teamBRatingAvg`, `ratingDelta`). `POST /api/match-plans/{id}/generate/confirm` creates the actual match.

| Strategy | Description |
|---|---|
| `BALANCED` | Sorts players by skill rating, interleaves alternately for minimum rating delta |
| `RANDOM` | Randomly shuffles confirmed players into two equal teams |
| `SNAKE_DRAFT` | Snake-order draft (1-2-2-2-...-1) for natural variance |

V6 migration: adds `generation_notes VARCHAR(500)` to `matches`.

---

### 🆕 New Feature: CalculationService — Skill Rating & Streaks

**Date:** May 17, 2026  
**Key Files Changed:**
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`
- `src/main/java/pt/rics/demo/football/model/SkillRatingHistory.java`
- `src/main/java/pt/rics/demo/football/repository/SkillRatingHistoryRepository.java`
- `src/main/java/pt/rics/demo/football/service/MatchEventListener.java`

Post-match skill rating recalculation triggered automatically via `MatchCompletedEvent` (Spring application event). Season-end transition callable by admin. All players included (active and inactive — `isActive` is a participation toggle, not soft-delete).

**Match rating formula:**
```
matchRating = BASE (5.0)
            + goals × GOAL_WEIGHT (0.5)
            + assists × ASSIST_WEIGHT (0.3)
            - ownGoals × OWN_GOAL_PENALTY (0.4)
            + WIN_BONUS (0.5) or - LOSS_PENALTY (0.3)
            + goalDiffBonus (capped)
```

**Season-end transition formula:**
```
newRating = (endRating × 0.45) + (avgRating × 0.25) + (startRating × 0.10) + (meanRating × 0.20) + activityAdjustment
```
Bounds: 1.0–10.0 · Max change per season: ±2.0. All changes logged to `skill_rating_history`. 37 new tests — all green.

---

### 🔧 Maintenance: Migration Consolidation & Docker Compliance

**Date:** May 17, 2026

**Migration consolidation:** V2–V6 merged into `V1__initial_schema.sql` (single baseline). All incremental migrations deleted. Schema is clean for fresh deployments.

**Docker fixes:**
- `Dockerfile`: Java 17→21 (builder + runtime), `gradle:8.5-jdk17` → `eclipse-temurin:21-jdk-alpine` + `./gradlew`, ZGC JVM flags added to ENTRYPOINT, health check corrected to `/api/health`
- `docker-compose.yml`: removed stray `- keycloak-db` from keycloak volumes, added healthcheck to `keycloak-db`, `depends_on` upgraded to `condition: service_healthy`
- `docker-compose.prod.yml`: fixed `_version` typo, removed nginx service (folder deleted), `JAVA_OPTS` corrected from G1GC to ZGC, removed non-existent `init.sql` mount, app port exposed, JWT expiration aligned to 24h

**Project cleanup:** removed `.vscode/`, `.junie/`, `k8s/`, `nginx/`, `backups/`, `uploads/`, `latest.dump`, `.gitlab-ci.yml`, `Jenkinsfile`, `Dockerfile.optimized`, `qodana.yaml`, stale fix scripts and Flyway repair SQLs.

---


**Date:** May 15, 2026  
**Files Changed:**
- `build.gradle`
- `src/main/resources/application.yml`

PostgreSQL JDBC driver pinned to **42.7.11** to resolve 2 HIGH-severity CVEs identified in security audit (Phase 7). Hardcoded JWT secret removed from codebase; `JWT_SECRET` environment variable is now mandatory — application fails fast on startup if not provided. No default fallback value exists.

---

### 🆕 New Feature: Match Live Stats & Rating Overhaul

**Date:** May 19, 2026  
**Files Changed:**
- `src/main/resources/db/migration/V2__player_stats_goal_types.sql`
- `src/main/java/pt/rics/demo/football/model/PlayerStat.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerStatUpdateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerStatDTO.java`
- `src/main/java/pt/rics/demo/football/dto/PlayerMatchRatingDTO.java`
- `src/main/java/pt/rics/demo/football/dto/MatchLiveUpdateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/MatchLiveUpdateResponseDTO.java`
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`
- `src/main/java/pt/rics/demo/football/service/MatchService.java`
- `src/main/java/pt/rics/demo/football/controller/MatchController.java`

**Goal type breakdown:** The single `goals` field on player stats has been replaced with three typed goal fields — `soloGoals`, `assistedGoals`, and `penaltyGoals` — each carrying a distinct rating weight (0.50 / 0.30 / 0.15 respectively). V2 migration adds `solo_goals`, `assisted_goals`, `penalty_goals` columns to `player_stats`.

**New live stats endpoint:** `PATCH /api/matches/{id}/stats/live` accepts `MatchLiveUpdateDTO` (a list of `PlayerStatUpdateDTO`) and returns `MatchLiveUpdateResponseDTO` containing per-player preview ratings. Live ratings are computed without WIN/LOSS bonus or goal-diff bonus — they represent an in-progress snapshot for admin display.

**Server-side rating computation:** `rating` is no longer submitted by the client in `PlayerStatUpdateDTO` or `MatchCompleteDTO`. `CalculationService.computeMatchRating()` is now the sole source of match ratings. The `MatchCompleteDTO` response (`MatchDTO`) contains server-computed `PlayerStatDTO.rating` for every player.

**Compliance validation on completion:** `MatchService.completeMatch()` enforces goal-count consistency before saving. For each team: `(teamGoals scored = soloGoals + assistedGoals + penaltyGoals summed across team players) + opponentOwnGoals` must equal the declared score. Returns `400 Bad Request` with a descriptive message if the check fails.

---

### 🆕 New Feature: Team Generation — Phase 2 & 3 Strategies

**Date:** 2026-05-20  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/service/teamgeneration/FormBasedGenerationStrategy.java`
- `src/main/java/pt/rics/demo/football/service/teamgeneration/CaptainPickGenerationStrategy.java`
- `src/main/java/pt/rics/demo/football/repository/PlayerStatRepository.java`
- `src/main/java/pt/rics/demo/football/model/GenerationType.java`

## Team Generation — Phase 2 & 3 Strategies

### FORM_BASED Generation ✅

Uses each player's recent match history instead of career `skillRating` for team balancing.

**Algorithm:**
1. Fetch last N completed matches per player from `player_stats` (default N=5, configurable via `formWindow` param)
2. Compute linearly-weighted form score: most recent match = weight N, oldest = weight 1
   - `formScore = Σ(rating_i × (N−i)) / Σ(N−i)` where i=0 is the most recent
3. Fall back to `player.skillRating` for players with no rated match history
4. Apply greedy bin-packing (same as BALANCED) using form scores

**New param:** `formWindow` (String, optional) in `TeamGenerationContext.params` — default `5`, min `1`

**New repo method:** `PlayerStatRepository.findRecentRatedByPlayerId(playerId, Pageable)`

---

### CAPTAIN_PICK Generation ✅ (server-side simulation)

Server-side snake draft simulation. Captains are auto-selected or explicitly designated.

**Algorithm:**
1. Resolve Captain A (highest-rated player, or `captainAId` param)
2. Resolve Captain B (second-highest-rated, or `captainBId` param)
3. Each captain seeds their own team
4. Remaining players sorted by `skillRating` DESC distributed via snake draft:
   `A picks → B picks | B picks → A picks | ...`

**Params (both optional):**
- `captainAId` — Long player ID for Team A captain
- `captainBId` — Long player ID for Team B captain

**Notes field example:**
`CAPTAIN_PICK: captainA=João Silva captainB=Miguel Santos avgA=7.43 avgB=7.21 Δ=0.22`

> Full interactive draft session (real-time human picks via API) remains a future Phase 4 item requiring a `DraftSession` entity and session API.

---

### 🆕 New Feature: Interactive Draft Session (Captain Pick Phase 4)

**Date:** 2026-05-20  
**Files Changed:**
- `src/main/resources/db/migration/V4__draft_sessions.sql`
- `src/main/java/pt/rics/demo/football/model/DraftSession.java`
- `src/main/java/pt/rics/demo/football/model/DraftStatus.java`
- `src/main/java/pt/rics/demo/football/repository/DraftSessionRepository.java`
- `src/main/java/pt/rics/demo/football/service/DraftSessionService.java`
- `src/main/java/pt/rics/demo/football/controller/DraftSessionController.java`
- `src/main/java/pt/rics/demo/football/dto/DraftSessionCreateDTO.java`
- `src/main/java/pt/rics/demo/football/dto/DraftSessionDTO.java`
- `src/main/java/pt/rics/demo/football/dto/DraftPickDTO.java`
- `src/main/java/pt/rics/demo/football/dto/DraftPlayerDTO.java`

Full stateful draft session API for interactive captain pick gameplay.

## Interactive Draft Session (Captain Pick Phase 4)

### New Endpoints
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/draft-sessions` | Create draft session from a CONFIRMED match plan |
| GET | `/api/draft-sessions/{id}` | Get current draft state |
| POST | `/api/draft-sessions/{id}/pick` | Submit a captain's pick |
| POST | `/api/draft-sessions/{id}/confirm` | Finalize draft and create Match |
| DELETE | `/api/draft-sessions/{id}` | Cancel draft session |

### Key Features
- Auto-selects highest/second-highest rated players as captains (or use explicit `captainAId`/`captainBId`)
- Both captains are pre-seeded in their respective teams at session creation
- Remaining players sorted by skill rating descending for fair pick order reference
- Alternating turns: A → B → A → B…
- Last pick automatically transitions status to `COMPLETED`
- Confirm draft creates a real `Match` entity with `generationType=CAPTAIN_PICK`
- Optional session expiry via `expiresAt` field
- Old server-side `CAPTAIN_PICK` simulation still available for non-interactive use

### New DB Migration
`V4__draft_sessions.sql` — 4 tables: `draft_sessions`, `draft_session_team_a`, `draft_session_team_b`, `draft_session_remaining`

38 new tests — 487 total. BUILD SUCCESSFUL.

---

### 🆕 New Feature: POST /api/players/{id}/link-me — BASIC_USER Self-Link Endpoint

**Date:** 2026-06-29  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/PlayerController.java`
- `src/main/java/pt/rics/demo/football/service/PlayerService.java`
- `src/main/java/pt/rics/demo/football/exception/GlobalExceptionHandler.java`

Dedicated self-link endpoint allowing any authenticated user (BASIC_USER, MASTER_USER) to link their own account to an existing, unlinked player profile. No request body is required — the calling user's ID is resolved from the JWT principal. Returns the updated `PlayerDTO` on success (200 OK).

**Authorization:** `isAuthenticated()` — ADMIN_USER callers are rejected with `403 Forbidden` at the service layer.

**Conflict handling (409 responses):**
- Player already linked to the calling user → `"Player is already linked to your account"`
- Player already linked to a different user → `"Player is already linked to another user"`
- Calling user already linked to a different player → `"You are already linked to another player"`
- Concurrent DB constraint violation (race condition) → `DataIntegrityViolationException` now handled in `GlobalExceptionHandler` → `409`

**Build:** SUCCESSFUL — 61 tests, 0 failures.

---

### 🆕 New Endpoint: PATCH /api/users/{id}/reactivate — Admin User Reactivation

**Date:** 2026-06-30  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/UserController.java`
- `src/main/java/pt/rics/demo/football/service/UserService.java`
- `src/test/java/pt/rics/demo/football/service/UserServiceTest.java`
- `src/test/java/pt/rics/demo/football/controller/UserControllerTest.java`

New `PATCH /api/users/{id}/reactivate` endpoint restricted to `ADMIN_USER`. Sets `isActive = true` on a deactivated account and returns the updated `UserDTO` (200 OK). Returns `404` if the user does not exist and `409 Conflict` with message `"User is already active"` if the account is not inactive. Complements the existing soft-delete endpoint `DELETE /api/users/{id}`. 9 new tests — all green.

---

### 🐛 Enhancement: Draft Session SSE Resume-on-Reconnect

**Date:** July 2, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/DraftSessionController.java`

Behavioral, **non-breaking** (patch-level) enhancement to `GET /api/draft-sessions/{id}/events`. Reconnecting to an already-terminal draft session (`CANCELLED` / `CONVERTED` reached while the client was disconnected) now sends `CONNECTED` (authoritative snapshot) → the matching terminal event → and closes the stream immediately, instead of hanging until the 5-minute SSE timeout with no close signal. `OPEN` / `COMPLETED` streams and the `404` fast-fail for missing sessions are unchanged. All session state is persisted, so a dropped stream is resumed simply by reconnecting — the `CONNECTED` snapshot is the resume/rehydrate primitive; no new endpoint, DTO, or schema change. See `docs/api/DRAFT-SESSION-RESUME-API-CONTRACT.md`.

---

### 🆕 New Feature: Manual Match Creation — POST /api/matches/manual

**Date:** July 9, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/dto/ManualMatchCreateDTO.java`
- `src/main/java/pt/rics/demo/football/service/MatchService.java`
- `src/main/java/pt/rics/demo/football/controller/MatchController.java`
- `src/test/java/pt/rics/demo/football/controller/MatchControllerTest.java`
- `src/test/java/pt/rics/demo/football/service/MatchServiceTest.java`

New `POST /api/matches/manual` endpoint restricted to `ADMIN_USER` and `MASTER_USER`. Allows creation of a match with teams of any equal size, bypassing the per-`matchType` player count restriction enforced by the standard `POST /api/matches` endpoint.

**Enforced constraints (same as standard, minus the count rule):**
- Exactly 2 teams
- Both teams must have the **same** number of players (at least 1 each)
- No duplicate player IDs across teams
- All player IDs must exist in the database

**Notable:** `matchDate` is **required** (`@NotNull`) on `ManualMatchCreateDTO` — unlike `MatchCreateDTO` where it is optional. The `generationType` in the response is always `"MANUAL"` (not user-configurable). Returns `201 Created` with standard `MatchDTO` and a `Location` header.

16 new tests (9 controller + 7 service) — all green.

---

### ⚡ Enhancement: Realistic Match Ratings — Rating Model v2

**Date:** May 8, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`
- `src/main/java/pt/rics/demo/football/repository/GoalRepository.java` (NEW)
- `src/test/java/pt/rics/demo/football/service/CalculationServiceTest.java`

The match/overall rating model has been overhauled to produce more realistic scores. The old
additive `BASE 5.0` formula compressed strong attacking displays — a decisive **WIN with 1
goal + 3 assists previously scored only ~6.8**. The new model:

- **Goal-type weights:** SOLO `3.0` > ASSISTED `2.0` > PENALTY `1.0`; ASSIST `1.5`; OWN_GOAL `−2.0`
  (own goals credit the opponent's scoreboard and are never rewarded).
- **Goal-timing impact** (when `Goal` rows exist, ordered by `minute` ASC NULLS LAST then
  `createdAt`): a late-game sequence uplift (`+0.50` max), `+0.40` go-ahead and `+0.25` equalizer
  bonuses; the assister receives 50% of the timing uplift. Graceful **FLAT fallback** to per-type
  `PlayerStat` weights when no `Goal` rows exist.
- **Unbounded RAW score** (`RAW_BASE 1.0` + stat points + capped decisiveness + WIN `+1.0` /
  LOSS `−0.75` + goal-diff nudge capped at `0.50`).
- **Match-wide proportional normalization** with a **stats-dependent ceiling**:
  `ceiling = 6.5 + 3.5 × clamp(topStatPoints/9.0, 0, 1)`; `finalRating = clamp(raw/topRaw × ceiling, 1.0, 10.0)`.

**Behavioral results:** the 1 goal + 3 assists WIN now scores **≈ 9.4**; a 1 goal + 1 assist top
performance tops out at **≈ 8.25** (never 10). A single-player match normalizes to the ceiling; a
0-0 no-stats draw normalizes to **6.5** (`CEILING_MIN`), not 5.0.

**Notable:** `computeMatchRating()` remains the sole source of ratings — its public overloads
return the RAW flat-clamped value used transiently by `MatchService`, while the authoritative
normalized value is written asynchronously by `recalculateMatchRatings` after completion. The
live-preview path (`PATCH /api/matches/{id}/stats/live`) stays **bonus-free**. New dependency:
`GoalRepository.findByMatchIdOrderByTiming(matchId)`.

No API contract change (`PlayerStatDTO.rating` shape unchanged). 62 tests green (~89% branch
coverage). See [`docs/features/CALCULATION_SERVICE.md`](../features/CALCULATION_SERVICE.md).

---

### 🆕 New Feature: Admin Match Rating Recalculation

**Date:** May 15, 2026  
**Files Changed:**
- `src/main/java/pt/rics/demo/football/controller/MatchController.java`
- `src/main/java/pt/rics/demo/football/service/MatchService.java`
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`
- `src/main/java/pt/rics/demo/football/repository/MatchRepository.java`
- `src/main/java/pt/rics/demo/football/repository/PlayerStatRepository.java`
- `src/main/java/pt/rics/demo/football/dto/RecalculationResultDTO.java` (NEW)
- `src/main/java/pt/rics/demo/football/dto/BulkRecalculationRequestDTO.java` (NEW)
- `src/main/java/pt/rics/demo/football/dto/BulkRecalculationResponseDTO.java` (NEW)

Two `ADMIN_USER`-only endpoints let an admin re-run the rating engine on demand (synchronously),
e.g. after amending a stat post-completion or after a rating-model change:

- `POST /api/matches/{id}/recalculate` → `RecalculationResultDTO` (200). `404` if the match is
  missing, `409` if it exists but is not completed; a completed match with no stats returns
  `200` with `status: "SKIPPED"`.
- `POST /api/matches/recalculate` → `BulkRecalculationResponseDTO` (always 200). Optional body
  `BulkRecalculationRequestDTO`; selection precedence `matchIds` → `seasonId` → all completed.
  Both supplied → `400`; unknown `seasonId` → `404`; non-positive ids → `400`. Per-match failures
  (including non-existent explicit `matchIds`) become `FAILED`/`SKIPPED` entries in `results[]`,
  never HTTP errors. Each match runs in its own transaction, chronologically ordered.

Non-admin (`BASIC_USER`/`MASTER_USER`) and unauthenticated callers both receive `403`.

**Idempotency:** `CalculationService.recalculateMatchRatings` is now idempotent via
reverse-then-reapply. `PlayerStat.rating`, career aggregates (`totalGoals`/`totalAssists`/
`totalMatchesPlayed`), `skill_rating_history` rows, and streaks are **exactly** idempotent.
`skillRating` (EMA) is exact for a player's most-recent match and an approximation for a mid-chain
historical match — the bulk endpoint's chronological replay fully reconciles a season. No DB schema
change. New finders: `MatchRepository.findCompletedOrdered`/`findCompletedBySeasonOrdered`,
`PlayerStatRepository.findCompletedByPlayerIdChronological`. All tests green (CalculationService
99%/90%, MatchService 99%/95%, MatchController 100% line). See
[`docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md`](../api/MATCH-RATING-RECALCULATION-API-CONTRACT.md).

