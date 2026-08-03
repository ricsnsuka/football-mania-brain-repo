# Review Handoff — Football Management System (2026-07-27)

> **For Claude Code (CLI).** This file hands off an in-progress code-review-and-fix pass that
> was started in a remote Cowork session which had **no access to Gradle/Docker** (it could edit
> files but not build or test). Your job: **verify the applied fixes build & pass**, add the
> regression tests noted below, then continue the backlog. Follow the project's own pipeline in
> `.github/agents/orchestrator.agent.md` and conventions in `.github/copilot-instructions.md`.

---

## 0. How to use this handoff

```bash
cd <this repo>
claude "Read docs/plans/REVIEW_HANDOFF_2026-07-27.md and start with section 2 (verify the build)."
```

The orchestrator session log has also been updated: see the two `2026-07-27` entries in
`docs/plans/ORCHESTRATOR_SESSION.md`. The `CHANGELOG.md` `[Unreleased]` section lists every change.

---

## 1. What was done — applied & written to disk (NOT build-verified)

Two batches of fixes were applied directly to the working tree. Nothing was committed. All edits
were surgical (exact-match) and re-read after writing, but **no compile/test was run** — that's the
first task for you.

### Batch 1 — correctness & concurrency
| ID | Fix | Files |
|----|-----|-------|
| H2 | `completeMatch` now assigns `matchResult` to **every** `PlayerStat` by team membership, not only players present in the request DTO. (Omitted players were kept `matchResult=null` → rated as neutral, dropped from streaks.) | `service/MatchService.java` |
| H3 | Added `@Version` to `Player`; `MatchEventListener` retries the idempotent recalculation up to 3× on `ObjectOptimisticLockingFailureException`. (Concurrent completions previously lost skillRating/aggregate/streak updates.) | `model/Player.java`, `service/MatchEventListener.java`, migration `V5` |
| M2 | `matches.season_id` set `NOT NULL` (entity mapped it mandatory; column was nullable + `ON DELETE SET NULL` → season delete orphaned matches). | migration `V5` |
| M1 | Added an **opt-in** `integrationTest` Gradle task + `MigrationSchemaValidationIT` that runs the real Flyway migrations on a PostgreSQL Testcontainer with `ddl-auto: validate`, catching entity/migration drift the H2 `create-drop` unit tests cannot. Excluded from the default `test` task. | `build.gradle`, `src/test/java/.../MigrationSchemaValidationIT.java`, `src/test/resources/application-integration.yml` |

### Batch 2 — robustness
| ID | Fix | Files |
|----|-----|-------|
| H1 | `BALANCED` + `FORM_BASED` greedy split now caps each team at half the pool (guarantees equal teams); `confirmGeneration` asserts both teams == `matchType/2` before persisting. | `service/teamgeneration/BalancedGenerationStrategy.java`, `FormBasedGenerationStrategy.java`, `service/MatchPlanService.java` |
| M3 | `GlobalExceptionHandler` maps standard MVC exceptions to 4xx: `HttpMessageNotReadable`→400, `MethodArgumentTypeMismatch`→400, `MissingServletRequestParameter`→400, `HttpRequestMethodNotSupported`→405, `HttpMediaTypeNotSupported`→415. (Were falling through to the catch-all → 500.) | `exception/GlobalExceptionHandler.java` |
| M6 | `@Version` on `DraftSession` (+ migration `V6`) — concurrent picks now 409 instead of lost pick / double-assign. | `model/DraftSession.java`, migration `V6` |
| M7 | `confirmDraft` + draft read path validate all referenced players still exist → 422 instead of NPE / null-FK when a drafted player was hard-deleted. | `service/DraftSessionService.java` |
| M10 | `MatchCompleteDTO.playerStats` cascades `@Valid`. | `dto/MatchCompleteDTO.java` |
| M13 | `adminUpsertConfirmation` rejects mutation on a CANCELLED plan. | `service/MatchPlanService.java` |

### New files
- `src/main/resources/db/migration/V5__optimistic_lock_and_season_constraints.sql`
- `src/main/resources/db/migration/V6__draft_session_version.sql`
- `src/test/java/pt/rics/demo/football/MigrationSchemaValidationIT.java`
- `src/test/resources/application-integration.yml`

### Docs updated
- `CHANGELOG.md` (two review-pass sections under `[Unreleased]`)
- `docs/architecture/ARCHITECTURE.md` (migration history table now V1–V6 — this also fixed doc-drift item **D1**)
- `docs/plans/ORCHESTRATOR_SESSION.md` (two session entries)

---

## 2. VERIFY NOW (first task)

```bash
./gradlew clean test          # existing suite — H2 create-drop; should stay GREEN
./gradlew integrationTest     # new drift check — REQUIRES DOCKER running
./gradlew spotbugsMain        # static analysis (part of `check`)
```

**Expectations / troubleshooting:**
- `test` should pass unchanged. The two new migrations add `players.version` and
  `draft_sessions.version`; the create-drop test schema generates those columns from the
  `@Version` fields automatically, so unit tests should be unaffected. If a `DraftSessionServiceTest`
  or `MatchService`/`CalculationService` test fails, most likely an assertion needs the new
  optimistic-lock field accounted for — inspect, don't blindly delete.
- `integrationTest` is **new and was never executed**. If it fails:
  - Docker not running → start Docker, re-run. It is intentionally NOT wired into `check`, so this
    won't block the normal build.
  - If it fails on Hibernate **schema validation**, that is the test doing its job — it found real
    entity/migration drift (e.g. the known `Player.user` `@ManyToOne` vs `user_id UNIQUE`, report
    item **L5**, still open). Capture the mismatch and fix the entity or add a migration.
- If anything doesn't compile, the culprit is one of the edited files in section 1 — re-read the
  method; all edits were verified structurally but not compiled.

After green: run the mandatory pipeline stages for changed service/repository/schema code —
`phase3-compliance` + `test-engineer` (per `orchestrator.agent.md`). No endpoints or DTO shape
changed, so `postman-engineer` is not required (M10 only adds `@Valid`).

---

## 3. Regression tests to ADD (test-engineer stage)

These target the bugs fixed above and did not exist before:
1. **H2:** `completeMatch` with a valid payload that **omits** a player who has a `PlayerStat` row
   → assert that omitted player's `matchResult` is set (WIN/LOSS/DRAW by team), and that they are
   included in streak recomputation.
2. **H3:** concurrency/idempotency — two matches sharing a player recalculated; assert aggregates
   are not lost. (At minimum, a test that a recalc after an `ObjectOptimisticLockingFailureException`
   retries and converges.)
3. **H1:** a `BALANCED` (and `FORM_BASED`) generation with a deliberately skewed rating pool →
   assert `teamA.size() == teamB.size() == required/2`. Add a `confirmGeneration` test asserting the
   unbalanced-teams guard throws 422 if a strategy returns lopsided teams (mock the strategy).
4. **M3:** controller slice tests — malformed JSON body → 400; `/api/users/abc` (type mismatch) →
   400; unsupported method → 405.
5. **M6:** two concurrent `pick` calls on one OPEN session → one 409 (optimistic lock).
6. **M7:** `getDraftSession`/`confirmDraft` when a drafted player id no longer resolves → 422.
7. **M13:** `adminUpsertConfirmation` on a CANCELLED plan → 409.

---

## 4. Deferred — need a decision or careful build-verified work

- **M5 — amend-stat aggregate reconciliation.** `MatchService.amendPlayerStat` rewrites a completed
  match's stat but never updates `Player.totalGoals/totalAssists`, `skillRating`, or the stored match
  score, so denormalised totals drift. The clean fix runs through the `reverse-then-reapply` recalc
  path (`CalculationService.recalculateSingleMatch`) and also needs a decision on whether amend may
  change the scoreline. Was NOT changed — do this with tests in front of you.
- **M12 — surplus confirmations.** A plan can reach `CONFIRMED` with more confirmations than the
  match needs (`updatePlanStatus` allows `>= required`), but generation requires **exactly**
  `required`, so the plan gets stuck. Needs a product decision: allow surplus + a drop mechanism, or
  forbid surplus at the CONFIRMED transition. Not changed.

## 5. Pending USER decisions (do not change behaviour without confirmation)

- **M8 — PII exposure.** `GET /api/players` + `/api/players/{id}` return `email` + `phoneNumber`
  (from linked `AppUser`) to ANY authenticated user (`isAuthenticated()`). A `BASIC_USER` can harvest
  every member's contact info. Confirm whether this is intended; if not, gate those fields to
  GROUP_ADMIN/MASTER (or the player's own linked user) in `PlayerMapper`/`PlayerDTO`.
- **M9 — draft authZ.** `POST /api/draft-sessions/{id}/pick` is `isAuthenticated()` with no check
  that the caller is a captain/creator/participant, so any user can drive any open draft. Javadoc
  says "captains can be anyone", so this may be intended — confirm before locking down.

---

## 6. Remaining backlog (from the review, NOT yet done)

**Performance:** M11 (`CalculationService.applyMatchRatings` calls `playerRepository.findAll()` per
match → O(matches×players) in bulk recalc; replace with an aggregate query), M14 (N+1s:
`getConfirmations` lazy player, `getAllDraftSessions` per-session loads, `DraftSession` EAGER
`@ElementCollection`s), L6 (missing indexes: `player_confirmations.player_id`, `goals.match_team_id`,
`goals.assister_id`, `matches.winning_team_id`).

**Durability:** M4 (async completion failure is swallowed — no retry/needs-recalc flag beyond the
H3 retry; a full fix likely wants a `needs_recalc` column).

**Low / hardening:** L1 (enforce JWT secret ≥64 chars at startup), L2 (`/api/auth/**` wildcard →
narrow to `/login`), L3 (swagger/actuator exposure in prod), L4 (`HttpRequestLoggingFilter` logs
`user=anonymous` — runs before security), L5 (`Player.user` `@ManyToOne` vs `UNIQUE` → `@OneToOne`),
L7 (DTO validation gaps: `@Valid` on `MatchCreateDTO`/`ManualMatchCreateDTO.teams`,
`@Positive` on `MatchPlanCreateDTO.minPlayersRequired`, constrain `AdminUserUpdateDTO.role`), L8
(`DraftSession.expiresAt` never set — dead expiry branch), L9 (SSE subscribe race + only catches
`IOException`), L10 (empty file `service/PlayerHistoricalStats.java` — delete), L11
(`UserController.adminUpdateUser` missing `@Valid`).

**Doc drift (D1 done):** D2 (`CHANGELOG` references non-existent `V2__add_player_audit_columns.sql`),
D3 (`copilot-instructions` controller example uses `/api/v2/players` — actual base is `/api/`),
D4 (`copilot-instructions` "Matches & Teams" says `generationType balanced|manual` + "exactly 14
players" — actually 6 strategies + 5/7/11-a-side), D5 (`copilot-instructions` overview still mentions
removed "peer evaluations"), D6 (`Goal.java`/`PlayerStat.java` javadoc mis-attribute migration
history).

---

## 7. Environment caveats from the remote session

- The working tree already had **many uncommitted changes** before this pass (unrelated to it). To
  review only this pass's work, `git diff` the specific paths in section 1, or check `CHANGELOG.md`.
- The remote shell **could not delete files**, so a throwaway probe file was moved to a new
  `_to_delete/` folder at the repo root — **you can delete `_to_delete/`**.
- Recommended commit grouping: (1) batch-1 correctness/concurrency + `V5` + M1 harness;
  (2) batch-2 robustness + `V6`; keep docs with their batch. Run the build before committing.
