# Changelog

All notable changes to the Football Management System are documented here.

## [Unreleased]

### 🔔 Phase 2 — Web Push: subscription and send path

Everything needed to deliver a notification, minus the triggers that decide to send one.

**No dependency (`WebPushCrypto`, `VapidSigner`, `WebPushSender`)**

- RFC 8291 payload encryption and RFC 8292 VAPID auth implemented against the JDK. The
  roadmap suggested `nl.martijndwars:web-push`, but it has been unmaintained since 2022 and
  transitively pulls a CLI argument parser, Apache HttpAsyncClient and the whole Netty stack —
  a second and third HTTP client for an app that already has Spring's and the JDK's — while
  still needing BouncyCastle added by hand. SunEC, `HmacSHA256`, `AES/GCM/NoPadding` and the
  existing `jjwt` cover every primitive.
- Hand-written crypto is only defensible when it is checked, so `encrypt()` takes the ephemeral
  keypair and salt as parameters and the test drives it with the **published RFC 8291 §5 vector**,
  asserting the output byte for byte. `VapidSignerTest` parses each token back with the public
  key the header advertises — signing something no push service can verify is the failure that
  would otherwise surface asynchronously, where nobody is watching.
- One trap worth naming: `BigInteger.toByteArray()` is variable-length, so writing EC
  coordinates straight out yields a wrong-length point for roughly one key in 128. Coordinates
  are left-padded into a fixed 32-byte field; a test generates 200 keys so that path executes.

**Subscriptions and preferences (`V12`)**

- `push_subscriptions` keyed by a **unique endpoint**, because the endpoint is the identity of a
  browser install. Browsers re-subscribe whenever the push service rotates keys, so subscribe is
  an upsert — otherwise one person collects a row per re-subscription and receives that many
  copies of every notification. It also reassigns ownership, which is what stops a shared
  browser delivering the previous user's notifications to the next one.
- `notification_mutes` stores **opt-outs, not preferences**: every category is on by default, so
  a new category ships enabled with no backfill and the table stays empty for anyone who never
  changes anything. Shipped with the feature rather than after it, per the roadmap's warning
  that retrofitting notification preferences is harder and later than doing it up front.
- `GET|POST|DELETE /api/push/subscribe`, `GET|PUT /api/push/preferences`. Every one acts on the
  calling user from the JWT principal — no parameter names a subject. `/api/push/public-key` is
  the one public route, listed explicitly rather than opening `/api/push/**`.
- Subscription keys are validated at registration: a `p256dh` that is not a 65-byte point on the
  curve, or an `auth` that is not 16 bytes, is rejected there rather than failing every send
  forever afterwards.

**Sending (`PushNotificationService`)**

- One entry point for trigger sites. Push not configured, player has no account, category muted,
  no registered device — all resolved in one place so a trigger reads as a line and cannot get
  the checks subtly wrong.
- Never throws. Runs on the ADR-002 async pattern behind an already-committed transaction: a
  push service being unreachable must not undo a saved match or reach a caller who is gone.
- A `404`/`410` means the browser is permanently gone, so the subscription is pruned — keeping
  it would retry a guaranteed failure on every future notification, and it is personal data with
  nothing left justifying it. A `429` or `5xx` is transient and the row is kept.
- `notifyPlayers` is annotated in its own right rather than looping over `notifyPlayer`: that
  would be a self-invocation, which does not pass through the Spring proxy, silently dropping
  both `@Async` and `REQUIRES_NEW` and running the whole fan-out inline.

**Privacy** — push subscriptions are personal data, so they land in both paths at the same time
as the table, as `PRIVACY_AND_DATA_PROTECTION.md` required. Erasure needs no new code (both
tables cascade from `users`). The export reports device label, push service **host**,
registration and last-notified time — but withholds the endpoint and both keys, which together
are a working capability to push to that browser rather than a fact about the person.

**Operations** — `./gradlew generateVapidKeys` prints a keypair using the application's own
encoding. Blank keys disable push cleanly: sends are skipped, `/public-key` reports
`enabled: false`, and the app boots normally — the default for local development and CI.

**Triggers — all five wired (`V13`)**

- **`DRAFT_YOUR_TURN` / `DRAFT_COMPLETED`** — `DraftPushNotifier`, a second listener on the
  existing `DraftStateChangedEvent` rather than a branch inside `DraftSseEmitterRegistry`. The
  two do unrelated things with the same event (open SSE streams vs closed browsers) and a
  failure in either must not affect the other. Only the captain now on the clock is notified;
  other captains' picks are already visible to anyone watching the stream. Participants are
  deduped, since captains appear in their own team lists.
  A new `OPENED` event is published on session creation so the first captain is told
  it is their turn — every later turn arrives on a `PICK`, so without it the person who
  picks first was the only participant never notified. It never reaches an SSE client:
  nobody can be subscribed to a session that did not exist a moment earlier.
- **`MATCH_COMPLETED`** — `MatchEventListener`, **after** the rating recalculation succeeds.
  Notifying when the match is merely saved would announce a rating change and then land the
  person on a screen still showing the old value, and on the optimistic-lock retry path a
  notification sent before a failed attempt could not be taken back.
- **`CONFIRMATION_DEADLINE` / `MATCH_REMINDER`** — `ReminderScheduler`, hourly on the hour.
  Deadline reminders go only to players still `PENDING`; match reminders only to `CONFIRMED`.
  Chasing someone who has already answered is exactly the noise that gets a channel muted.
- **Safe on more than one instance.** The scheduler claims each reminder with a conditional
  update (`SET sent_at = now WHERE id = ? AND sent_at IS NULL`), so the database picks the
  winner and the loser sends nothing. Read-then-write would leave a window where both instances
  see `NULL`. The same guard stops a restart re-notifying on every tick for a whole day.
  Reminders are consequently at-most-once, which is the right trade — a missed reminder is a
  small annoyance, a duplicate at 3am is why people disable notifications.
- `ReminderDispatcher` is a separate bean from `ReminderScheduler` for a mechanical reason:
  `@Transactional` is proxy-based, so a call from another method of the same bean bypasses it,
  and the `@Modifying` claim would fail outright without a transaction.
- `@EnableScheduling` added to `FootballApplication` — the first scheduled work in the project.

### 🛡️ Phase 0 — Groundwork Before the Mobile Roadmap

Implements Phase 0 of `docs/plans/Football App - Improvement and Mobile Roadmap.md`: the decisions
and cleanup that get harder once Phase 5 multiplies the blast radius.

**One current season, enforced (`V10`)**

- `ux_seasons_single_current`, a partial unique index over the `is_current = TRUE` rows. A partial
  index rather than `UNIQUE(is_current)`, which would also forbid a second *closed* season — what
  the table is mostly full of.
- The old note in `SEASON_FEATURE.md` misdescribed the failure it was documenting: the application
  did not "pick the first match if multiple exist". `SeasonRepository.findByCurrentTrue()` returns
  `Optional<Season>`, so a second current row raised `IncorrectResultSizeDataAccessException` — a
  500 on `POST /api/matches`, match-plan confirmation and draft-session creation alike. Corrected
  there.
- `SeasonCurrentConstraintIT` proves the index fires against real PostgreSQL. The H2 unit tests
  build their schema from the entities and a partial index is not expressible in JPA, so as far as
  they are concerned the constraint does not exist.

**GDPR: data export and erasure (`V11`)**

- `GET /api/privacy/me/export` and `DELETE /api/privacy/me` for the caller's own data;
  `GET|DELETE /api/privacy/players/{id}` for an ADMIN actioning a request on behalf of someone who
  cannot make it themselves — a player added by an admin may have no account at all.
- **Erasure anonymises in place rather than deleting.** `player_stats`, `goals` and
  `skill_rating_history` all reference `players(id)` with `ON DELETE CASCADE`, so deleting a player
  would delete their appearances and the goals they scored — silently rewriting *other* players'
  records, whose scorelines would stop matching their goal lists. Name, phone number and the entire
  login account go; the anonymous statistical residue stays. `players.anonymized_at` records that
  it happened, guards against a second attempt, and holds no personal data itself.
- Usernames are scrubbed from the `created_by` audit columns on match plans and draft sessions — a
  username identifies a person as surely as a name, and leaving it would make the erasure cosmetic.
- The export is scoped to *data about this person*, not *data this person can see*: a match is
  represented by their own line in it, never the full scoresheet, and goals name the subject's role
  but not the counterpart. Otherwise one person's export becomes a way to harvest the roster.
- An `ADMIN_USER` cannot erase their own account (403) — the request would irreversibly delete the
  credentials that administer the deployment, potentially the last set, from a self-service
  endpoint.
- `docs/features/PRIVACY_AND_DATA_PROTECTION.md` records the posture, what is stored and why, and
  what erasure deliberately does not reach (application logs — retention is the backstop).

**Decisions recorded, no code**

- **ADR-004** fixes the LLM vendor for Phase 3's AI match reports: Anthropic Claude via the official
  Java SDK on `claude-opus-5`, with the call shape, the ≈$0.025-per-report arithmetic, and the
  guards (one report per match ever, a daily ceiling, a default-off feature flag) settled up front.
- `docs/plans/PHASE0_FRONTEND_HANDOFF.md` carries the branding work, which lives in the separate
  frontend repository: the app mark as inline SVG, the icon-generation script, and the Next.js
  App Router file conventions that make most of the usual `<link rel="icon">` boilerplate
  unnecessary.

### 🧹 Review Backlog Cleared (review pass, batch 6)

Closes the remaining review items: M4, M11, M14, L1–L11 and the D2–D6 doc drift.

**Durability**

- **M4 — failed post-completion work is now recorded.** Rating recalculation runs asynchronously
  after `completeMatch` commits, and the handler deliberately swallows failures so they cannot
  reach a caller whose match is already saved. That left a log line as the only trace: the match
  looked complete while its participants' ratings quietly were not. `matches.needs_recalc` (`V9`)
  records the state so it survives a restart and is queryable
  (`findByNeedsRecalcTrueOrderByMatchDateAsc`). The flag is cleared inside
  `CalculationService`, so *any* successful recalculation resolves it — admin single, admin bulk,
  or a later retry — and marking is itself failure-tolerant.

**Performance**

- **M11 — bulk recalculation no longer loads the roster per match.** The scarcity pass called
  `playerRepository.findAll()` for every match, making a season replay O(matches × players) in
  rows fetched. Replaced with two scalar aggregates computed in the database.
- **M14 — N+1s removed.** `getConfirmations` uses fetch-joined finders (the DTO reads
  `getPlayer()`, so the lazy association cost a query per confirmation);
  `getAllDraftSessions` does one batched player lookup for the whole listing instead of one per
  session; `DraftSession`'s three eager `@ElementCollection`s carry `@BatchSize`.
- **L6 — indexes for unindexed foreign keys** (`V8`): `player_confirmations.player_id`,
  `goals.match_team_id`, `goals.assister_id`, `matches.winning_team_id`, plus a composite
  `(match_plan_id, status, confirmed_at)` serving the confirmation-ordered query.

**Security**

- **L1 — JWT secret is validated at startup.** `Keys.hmacShaKeyFor` only rejects secrets below 32
  bytes, so a short secret silently weakened signing rather than failing. Now a secret under 64
  bytes (or missing) refuses to start.
- **L2 — `/api/auth/**` narrowed to `/api/auth/login`.** The wildcard would have made any future
  endpoint on that controller public the day it was written.
- **L3 — API docs and actuator restricted in production.** `app.security.public-api-docs` is
  false in `application-prod.yml` (docs then require authentication), and actuator exposure is
  narrowed to `health` only.
- **L4 — request log no longer reports every caller as anonymous.** The filter runs ahead of the
  security chain so a `requestId` exists for auth failures too, which meant the user was resolved
  before authentication had happened. Identity is now resolved on the way out, for the completion
  entry that actually matters for auditing writes.

**Correctness / hygiene**

- **L5 — `Player.user` mapped `@OneToOne`.** `players.user_id` is UNIQUE, so the previous
  `@ManyToOne` described a relationship the database forbids.
- **L7 — validation gaps closed:** `@Valid` cascades into `MatchCreateDTO.teams` and
  `ManualMatchCreateDTO.teams`; `@Positive` on `MatchPlanCreateDTO.minPlayersRequired`;
  `AdminUserUpdateDTO.role` constrained to the known roles.
- **L8 — draft expiry wired up, opt-in.** `pick()` always cancelled an expired session but
  nothing ever set `expiresAt`, so the branch was unreachable. `app.draft.ttl-hours` defaults to
  `0` (no expiry, preserving today's behaviour); a positive value activates it.
- **L9 — two SSE defects.** `trySend` caught only `IOException`, but `SseEmitter.send` throws
  `IllegalStateException` for an already-completed emitter — an unchecked exception that escaped
  the broadcast loop, so one stale subscriber stopped the event reaching everyone after it.
  Separately, subscribe/unsubscribe were not atomic: a new subscriber could be added to a list
  that removal was concurrently dropping from the map, leaving it silently receiving nothing.
- **L11 — `@Valid` on `adminUpdateUser`**; **L10 —** deleted the empty `PlayerHistoricalStats.java`.

**Docs (D2–D6)** — removed references to a `V2__add_player_audit_columns.sql` that never existed
(the columns are in the V1 baseline); corrected `/api/v2/players` → `/api/players`; replaced the
"balanced generation requires exactly 14 players" and `balanced|manual` claims with the real match
types and generation strategies; dropped the removed "peer evaluations" from the overview; and
replaced the migration attributions in `Goal`/`PlayerStat` javadoc, which named migrations that
never did what they claimed.

> **Also:** `testSecurity` joins the CI test tasks. The package-filtered split silently skipped
> the new `security` package, so tests there ran locally but not in CI; `build.gradle` now
> documents the invariant and how to re-check it.

---

### 🔒 Player Contact Details Restricted (review pass, batch 5)

**M8 — PII exposure.** `GET /api/players` and `/api/players/{id}` are `isAuthenticated()` and
returned `email` and `phoneNumber` (sourced from the linked `AppUser`) to any logged-in user, so
a `BASIC_USER` could page through the roster and harvest every member's contact details. Those
two fields are now visible only to:

- `ADMIN_USER` and `MASTER_USER`, who administer the roster; and
- the player's **own linked account**.

Everyone else receives the same payload with both fields null. Nothing else is withheld — names,
ratings and career stats remain visible to authenticated users, and the write endpoints were
already ADMIN/MASTER-only.

Redaction is applied in the controller, deliberately **outside** the cache: `getPlayer` and
`listPlayers` are `@Cacheable` on keys that do not include the caller, so redacting inside them
would have served one caller's view to the next. The cache keeps the full record and each
response is filtered for its own caller.

**M9 — draft authorization: confirmed as intentional, left open.** Any authenticated user may
submit a pick, and the caller is not matched against the captain whose turn it is. Captaincy has
no account requirement — a captain can be any player, including one with no linked user — so
there is no identity to enforce against. This is now documented at `DraftSessionService.pick`
rather than left looking like an oversight, along with a note on where the check would belong if
it is ever locked down. Turn order and pick validity are still enforced.

#### Files
- `security/PlayerPiiPolicy.java` — new; visibility rule + redaction
- `controller/PlayerController.java` — applies the policy to both read endpoints
- `service/DraftSessionService.java` — records the M9 decision

---

### 🔄 Amend Reconciliation & Surplus Confirmations (review pass, batch 4)

Resolves the two deferred items that needed a product decision.

**M5 — amending a completed match now reconciles everything.** `amendPlayerStat` previously
rewrote the single `PlayerStat` row and stopped, leaving the scoreline, the winner, every
player's WIN/LOSS/DRAW and all denormalised rating state stale. It now:

- **re-derives the scoreline from the stats**, using the same rule `completeMatch` already
  enforces — a team's score is its own goals plus the opposition's own goals. The stats are the
  source of truth after an amendment, so an amendment may legitimately change the result,
  including flipping the winner or turning a decided match into a draw;
- **reassigns `matchResult` on every stat** in the match, since a flipped result changes the win
  bonus / loss penalty for all of them;
- **replays the rating engine** via `CalculationService.recalculateSingleMatch` (reverse-then-
  reapply), so `skillRating`, career aggregates, streaks and `skill_rating_history` all converge.

  An in-progress match has no scoreline to reconcile, so the amendment there behaves as before.

**Exact reversal (V7).** The reverse phase restored `skillRating` from the history row but
decremented `totalGoals`/`totalAssists` from the *current* `PlayerStat`. That is only correct
while the stat is unchanged between apply and reverse — precisely what an amendment breaks,
leaving career totals permanently off by the amendment delta. Each application now records the
contribution it made (`skill_rating_history.goals_applied` / `assists_applied`) and the reversal
subtracts that. Rows predating V7 have no record and fall back to the previous behaviour.

**M12 — surplus confirmations are allowed, with automatic reserve promotion.** A plan may now
hold more confirmations than the match needs, instead of becoming un-generatable:

- confirmations are ranked by `confirmedAt`, so the queue is **order of entry**;
- generation (both team-generation and the captain-pick draft) takes the **first 10/14/22** in
  that order; the remainder are reserves;
- a withdrawal clears `confirmedAt` and removes the player from the CONFIRMED set, so the next
  generation re-derives its starters and **the first reserve takes the freed slot** — no separate
  bookkeeping, and re-confirming later takes a fresh timestamp, correctly going to the back of
  the queue;
- the previous "Too many confirmed players" rejection is gone; too *few* still fails with
  `at least N`.

#### Files
- `service/MatchService.java` — amend reconciliation + scoreline derivation
- `service/CalculationService.java` — record and reverse the applied contribution
- `model/SkillRatingHistory.java`, `db/migration/V7__history_applied_stats.sql` — new columns
- `repository/PlayerConfirmationRepository.java` — confirmation-ordered finder (`JOIN FETCH` on player)
- `service/MatchPlanService.java` — `selectStarterPlayerIds`, used by preview and confirm
- `service/DraftSessionService.java` — draft seeded from the first N in confirmation order

> **Not included:** starters vs reserves are not surfaced in the confirmations API. Players
> currently cannot see whether they are in the XI or on the reserve list — worth a DTO field,
> but that is an API contract change and was left out of this pass.

---

### 🧪 Build Verification & Test-Suite Repair (review pass, batch 3)

Build-verification pass over batches 1–2. Two defects found in the applied fixes, plus a
pre-existing red suite that predates this review.

- **M7 guard false positive:** `DraftSessionService.validateAllPlayersResolved` gated on
  `found.size() == distinctIds.size()` and only then computed the missing ids, so whenever the
  lookup returned more rows than were requested it threw `422` with an empty list
  (`references player(s) that no longer exist: []`). The check now derives the missing ids first
  and throws only when that list is non-empty — same protection, correct predicate.
- **`integrationTest` never ran (M1):** the task set no `testClassesDirs`/`classpath`, so it
  reported `NO-SOURCE` / `BUILD SUCCESSFUL` while executing zero tests. Once wired up it failed
  against Docker Engine 25+ with a misleading "Could not find a valid Docker environment" — the
  daemon actually replies `HTTP 400` because Testcontainers 1.20.6's docker-java negotiates an API
  version below the daemon's minimum. Pinned via `systemProperty 'api.version'`, overridable with
  `-PdockerApiVersion=`. The migration/entity drift check now genuinely runs: V1–V6 apply to a real
  PostgreSQL container and Hibernate `validate` passes.
- **CalculationServiceTest realigned to Rating Model v2.1 (33 tests):** the v2.1 rewrite landed with
  its implementation and docs updated but 33 test expectations left on the v1/v2 model, so the suite
  was already red before this review pass. Every expectation was re-derived from the documented v2.1
  formula (`RAW_BASE_POINTS` 7.5, `RATING_FLOOR` 4.0, ceiling band `[8.0, 9.5]`) rather than from
  observed output; all 33 matched the implementation exactly, confirming stale tests rather than a
  regression. Notable semantic changes captured in the tests:
  - streaks are recomputed by replaying the chronological chain, so tests must stub
    `findCompletedByPlayerIdChronological` — the stored `currentStreak` is ignored;
  - the v2.1 mapping is **affine**, not linear: proportionality holds as
    `(rating − 4.0) / (ceiling − 4.0) = raw / topRaw`;
  - a lone player in a match always maps to the ceiling, so "poor performance" tests need a second,
    dominant player to create the scale.

- **CI test tasks ran nothing:** `testControllers`, `testServices` and `testApplication` had the
  same missing-`testClassesDirs`/`classpath` defect as `integrationTest` — all three were
  `NO-SOURCE`, so CI executed zero tests and reported success. Now wired up: 248 + 381 + 1 = 630
  tests, matching the `test` task exactly (no gap between the three package filters).

#### Files
- `service/DraftSessionService.java` — M7 guard predicate
- `build.gradle` — classpath wiring for `integrationTest` + the three CI tasks; Docker API version pin
- `src/test/java/pt/rics/demo/football/service/CalculationServiceTest.java` — realigned to v2.1
- removed `_to_delete/` (throwaway probe from the remote session)

---

### ✅ Regression Tests for the Review-Pass Fixes

15 tests covering the bugs fixed in batches 1–2; none of these paths were previously exercised.

| Item | Test | Location |
|------|------|----------|
| H1 | Skewed rating/form pool still yields equal teams (uncapped greedy gave 1 v 9) | `BalancedGenerationStrategyTest`, `FormBasedGenerationStrategyTest` |
| H1 | `confirmGeneration` rejects lopsided teams with 422 and persists nothing | `MatchPlanServiceTest` |
| H2 | `completeMatch` sets `matchResult` on players omitted from the payload, and persists them | `MatchServiceTest` |
| H3 | Recalculation retries after an optimistic-lock conflict and converges; gives up after 3; no retry on other failures | `MatchEventListenerTest` |
| M3 | Type mismatch → 400, malformed JSON → 400, bad method → 405, bad content type → 415 | `UserControllerTest` |
| M6 | `pick` propagates the optimistic-lock conflict (→ 409) instead of losing a pick | `DraftSessionServiceTest` |
| M7 | `getDraftSession` / `confirmDraft` return 422 for a hard-deleted drafted player; nothing persisted | `DraftSessionServiceTest` |
| M13 | `adminUpsertConfirmation` on a CANCELLED plan → 409, before any write | `MatchPlanServiceTest` |

Status codes are asserted via `BusinessException.getStatus()` rather than exception type alone.

---

### 🛠 Correctness & Concurrency Hardening (review pass)

First-pass fixes from a full-codebase review. No API contract change; no new dependency.

- **completeMatch — match result for all players (H2):** `matchResult` (WIN/LOSS/DRAW) is now
  assigned to **every** `PlayerStat` of the match by team membership, not only the players present
  in the request payload. Previously an omitted player kept `matchResult = null`, so the rating
  engine applied no win bonus / loss penalty and the match was silently dropped from that player's
  streak recomputation.
- **Player optimistic locking (H3):** added a `@Version` column to `Player`. The async
  post-completion recalculation does read-modify-write on shared player rows (`skillRating`, career
  aggregates, streaks); concurrent match completions could previously lose updates (last-writer-wins).
  `MatchEventListener` now retries the idempotent recalculation up to 3× on
  `ObjectOptimisticLockingFailureException`.
- **matches.season_id NOT NULL (M2):** the entity maps season as mandatory but the column was
  nullable with `ON DELETE SET NULL`; deleting a season could orphan matches. `V5` makes the column
  `NOT NULL` (which also prevents orphaning).
- **Migration/entity drift guard (M1):** added an opt-in `integrationTest` Gradle task and
  `MigrationSchemaValidationIT` that runs the real Flyway migrations against a PostgreSQL
  Testcontainer with `ddl-auto: validate`. The default `test` task (H2 create-drop) builds the
  schema from the entities and so cannot catch entity↔migration drift; this can. Requires Docker;
  excluded from the default build.

#### Files
- `model/Player.java` — `@Version` field
- `service/MatchService.java` — assign `matchResult` to all stats
- `service/MatchEventListener.java` — retry recalc on optimistic-lock conflict
- `src/main/resources/db/migration/V5__optimistic_lock_and_season_constraints.sql` — new
- `src/test/java/pt/rics/demo/football/MigrationSchemaValidationIT.java` — new
- `src/test/resources/application-integration.yml` — new
- `build.gradle` — `integrationTest` task; exclude `*IT` from `test`

---

### 🧩 Robustness Fixes (review pass, batch 2)

Second-pass hardening. No API contract change; no new dependency.

- **Team generation equal-size guarantee (H1):** the greedy `BALANCED` and `FORM_BASED` strategies
  now cap each team at half the pool, so a skewed rating/form distribution can no longer produce
  lopsided teams. `MatchPlanService.confirmGeneration` also asserts both teams equal `matchType/2`
  before persisting.
- **Proper 4xx for bad input (M3):** `GlobalExceptionHandler` now maps `HttpMessageNotReadable`
  (malformed JSON → 400), `MethodArgumentTypeMismatch` (400), `MissingServletRequestParameter` (400),
  `HttpRequestMethodNotSupported` (405) and `HttpMediaTypeNotSupported` (415) — previously these fell
  through to the catch-all and returned 500.
- **Draft pick concurrency (M6):** added `@Version` to `DraftSession` (+ `V6` column). Racing picks
  on an OPEN session now collide detectably (409; the client re-syncs via SSE) instead of silently
  losing a pick or double-assigning a player.
- **Draft deleted-player guard (M7):** `confirmDraft` and the read path validate that every
  referenced player still exists, returning a clear 422 instead of an NPE / null-FK failure when a
  player was hard-deleted while the draft was open.
- **Nested completion validation (M10):** `MatchCompleteDTO.playerStats` now cascades `@Valid`, so a
  bad element (e.g. null `playerStatId`) is rejected at the edge rather than deep in the service.
- **Admin confirmation guard (M13):** `adminUpsertConfirmation` rejects mutations on a CANCELLED plan
  (was unguarded, could desync `confirmed_count`).

#### Files
- `service/teamgeneration/BalancedGenerationStrategy.java`, `FormBasedGenerationStrategy.java`
- `service/MatchPlanService.java` — size post-condition + CANCELLED guard
- `service/DraftSessionService.java` — deleted-player guard; `model/DraftSession.java` — `@Version`
- `exception/GlobalExceptionHandler.java` — standard MVC exception handlers
- `dto/MatchCompleteDTO.java` — `@Valid` cascade
- `src/main/resources/db/migration/V6__draft_session_version.sql` — new

**Deferred:** M12 (a plan can reach CONFIRMED with a confirmation count generation can't use — needs
a product decision on surplus handling), M5 (amend-stat aggregate reconciliation — touches the
recalc reversal path), M8/M9 (await behaviour decision).

---

### ⚡ Rating Model v2.1 — Realistic 6.0-Base Distribution

Refined the match rating model to produce realistic distributions. Rating Model v2 introduced
proportional normalization but produced unrealistic scores (1-goal contributor → 4.1,
non-contributor on WIN → 1.8). v2.1 fixes this via compressed-range mapping and rebalanced constants.

**Constants changed (v2 → v2.1):**
- `RAW_BASE_POINTS`: `1.0 → 7.5` (elevated base)
- `RAW_WIN_BONUS`: `1.0 → 0.4` (smaller relative to elevated base)
- `RAW_LOSS_PENALTY`: `0.75 → 2.2` (larger to balance losing team ratings)
- `RATING_FLOOR`: **new** `4.0` (lower bound of compressed mapping)
- `CEILING_MIN`: `6.5 → 8.0`, `CEILING_MAX`: `10.0 → 9.5` (compressed ceiling band)

**Formula:** changed from `raw/topRaw × ceiling` to `4.0 + (raw/topRaw) × (ceiling − 4.0)`,
mapping to range `[4.0, ceiling]` instead of `[0, ceiling]`.

**Results (3-1 victory worked example):**
- 1-goal contributor: ~7.0 (was 4.1 — FIXED ✅)
- Non-contributor on WIN: ~6.1 (was 1.8 — FIXED ✅; >6.0 requirement met)
- Non-contributor on LOSS: ~5.3 (was <2.0 — realistic)
- 3g+2a top: ~9.5 (was 10.0 — compressed)

**No API contract change.** All tests green (62 tests, ~89% branch coverage).

#### File Changed
- `src/main/java/pt/rics/demo/football/service/CalculationService.java`

---

### 🆕 Admin Match Rating Recalculation

Two `ADMIN_USER`-only endpoints to re-run the rating engine on demand, **idempotently**
(reverse-then-reapply — re-running never double-counts aggregates, streaks, EMA `skillRating`, or
`skill_rating_history` rows). No DB schema change.

- `POST /api/matches/{id}/recalculate` → `RecalculationResultDTO` (200; `SUCCESS`/`SKIPPED`;
  `404` if missing, `409` if not completed).
- `POST /api/matches/recalculate` → `BulkRecalculationResponseDTO` (always 200; per-match
  `results[]`). Selection precedence `matchIds` → `seasonId` → all completed; both supplied → `400`;
  unknown `seasonId` → `404`. Each match runs in its own transaction, chronologically ordered.
- **Idempotency guarantees:** `PlayerStat.rating`, career aggregates, history rows, and streaks are
  **exact**. `skillRating` (EMA) is exact for a player's most-recent match and approximate mid-chain —
  use the bulk endpoint's chronological replay to fully reconcile a season.

#### New Files
- `src/main/java/pt/rics/demo/football/dto/RecalculationResultDTO.java`
- `src/main/java/pt/rics/demo/football/dto/BulkRecalculationRequestDTO.java`
- `src/main/java/pt/rics/demo/football/dto/BulkRecalculationResponseDTO.java`

#### New Repository Finders
- `MatchRepository.findCompletedOrdered` / `findCompletedBySeasonOrdered`
- `PlayerStatRepository.findCompletedByPlayerIdChronological`

---

### ⚡ Match Ratings — Rating Model v2 (Realistic Scoring)

Overhauled the match/overall rating model for more realistic scores. Replaces the old additive
`BASE 5.0` formula with an unbounded RAW score + match-wide proportional normalization against a
stats-dependent ceiling.

- **Goal-type weights:** SOLO 3.0 > ASSISTED 2.0 > PENALTY 1.0; ASSIST 1.5; OWN_GOAL −2.0.
- **Goal-timing impact:** late-game sequence uplift, go-ahead (+0.40) / equalizer (+0.25) bonuses;
  assister gets 50% of the uplift. Graceful FLAT fallback when no `Goal` rows exist.
- **Stats-dependent ceiling:** `6.5 + 3.5 × clamp(topStatPoints/9.0, 0, 1)`; a WIN with 1 goal +
  3 assists now scores ~9.4 (was ~6.8); a 1 goal + 1 assist top performance caps at ~8.25 (never 10).
- Authoritative normalized rating written by `recalculateMatchRatings`; live preview stays bonus-free.

#### New Files
- `src/main/java/pt/rics/demo/football/repository/GoalRepository.java` — `findByMatchIdOrderByTiming(matchId)`

#### Testing
- **62 tests** — ~89% branch coverage. ✅ BUILD SUCCESSFUL

---

### 🆕 Player Entity — Full CRUD API

Complete Player management layer including REST API, MapStruct mapper, Flyway migration, and comprehensive test suite.

#### New Files
- `src/main/java/pt/rics/demo/football/model/Player.java` — JPA entity
- `src/main/java/pt/rics/demo/football/repository/PlayerRepository.java` — Spring Data JPA repository
- `src/main/java/pt/rics/demo/football/mapper/PlayerMapper.java` — MapStruct compile-time mapper
- `src/main/java/pt/rics/demo/football/service/PlayerService.java` — Business logic + cache management
- `src/main/java/pt/rics/demo/football/controller/PlayerController.java` — REST controller
- `src/main/java/pt/rics/demo/football/dto/PlayerDTO.java` — Response record
- `src/main/java/pt/rics/demo/football/dto/PlayerCreateDTO.java` — Create request record
- `src/main/java/pt/rics/demo/football/dto/PlayerUpdateDTO.java` — Partial update record (safe PATCH with `unlinkUser` flag)
- `src/main/java/pt/rics/demo/football/dto/PlayerStatusDTO.java` — Activate/deactivate record
- `players.created_by` / `players.updated_by` audit columns — part of the `V1__initial_schema.sql`
  baseline. (This entry previously named a `V2__add_player_audit_columns.sql` migration; no such
  file has ever existed, and the real V2 is `V2__player_stats_goal_types.sql`.)

#### Endpoints Added

| Method | Path | Auth |
|--------|------|------|
| GET | `/api/players` | Any authenticated |
| GET | `/api/players/{id}` | Any authenticated |
| POST | `/api/players` | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/players/{id}` | `ADMIN_USER` or `MASTER_USER` |
| PATCH | `/api/players/{id}/status` | `ADMIN_USER` or `MASTER_USER` |
| DELETE | `/api/players/{id}` | `ADMIN_USER` only |

#### Key Business Rules
- `ADMIN_USER` accounts **cannot** be linked to player profiles (enforced at service layer)
- Hard delete blocked if player has match statistics (`409 Conflict`)
- `email` derived from linked `AppUser` — not stored in `players` table
- Audit trail: `created_by` + `updated_by` populated from authenticated principal

#### Testing
- **48 tests** — 24 service unit tests (Mockito) + 24 controller slice tests (`@WebMvcTest`)
- ✅ BUILD SUCCESSFUL

#### Security
- PostgreSQL JDBC driver pinned to **42.7.11** — resolves 2 HIGH-severity CVEs
- `JWT_SECRET` env var is now mandatory — application fails-fast on startup if not set
- Hardcoded JWT secret removed from codebase

---

## [1.0.0] — 2026-05-15

### 🆕 Initial Release — Java 21 Rewrite

Complete rewrite from the ground up, driven by production performance requirements.

#### Tech Stack
- **Java 21** (upgraded from Java 17)
- **Spring Boot 3.4.5**
- **Virtual Threads** (`spring.threads.virtual.enabled=true`) — primary performance driver
- **ZGC with Generational mode** in production — sub-millisecond GC pauses

#### Architecture Decisions
- **DTOs as Java Records** — immutable, zero-boilerplate, GC-friendly
- **No thin service interfaces** — concrete `@Service` classes injected directly (lesson from v4.x Phase E)
- **Removed Redis** — Caffeine local cache is sufficient and eliminates a network dependency
- **Clean Flyway baseline** — V1 schema covers all domain entities from day one
- **Single `application.yml`** with profile overlays (`-dev`, `-prod`, `-test`)

#### What's Included in Scaffold
- `FootballApplication` — virtual threads + caching + async enabled
- `SecurityConfig` — profile-gated HMAC JWT (`!keycloak`) / OAuth2 Resource Server (`keycloak`)
- `JwtTokenProvider` + `JwtAuthenticationFilter` — HMAC JWT auth
- `UserPrincipal` — Java Record replacing old UserDetails class
- `GlobalExceptionHandler` — unified `ApiError` record response
- `BusinessException` + `ResourceNotFoundException` — typed exceptions with factory methods
- `CacheConfig` — Caffeine with named caches and stats
- `OpenApiConfig` — Swagger UI with Bearer auth
- `HealthController` — public `/api/health` endpoint
- `V1__initial_schema.sql` — complete database schema (users, players, matches, teams, goals, evaluations, seasons, plans, absences, rating config)

#### Default Admin User
- Username: `admin` / Password: `Admin@1234`
- `force_password_change = true` — must change on first login

---

*Previous versions (v2.x → v4.x) have been archived. The codebase has been reset.*

