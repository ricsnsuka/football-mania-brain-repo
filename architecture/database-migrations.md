# Database Migration History

**Complete: every version from `V1` to `V44` has a row**, verified against
`src/main/resources/db/migration/` on 2026-08-27. `V40` and `V41` were missing until then — the gap
was recorded on 2026-08-26 rather than filled, because the rationale was not to hand and inventing
one is worse than a hole. It was filled from the migrations' own headers when the backend repo was
next open beside this one, which is the only honest way to close a gap like that.

**Production is applied through `V41`**, which is where 2.0.0 left it; 2.1.0 added no migration.
**`V42`, `V43` and `V44` are cut into 2.2.0 and not deployed** — they sit on `next` in the backend
repo as of 2026-08-27. Verify against the platform rather than against this line: `heroku pg:psql -c
'select version from flyway_schema_history order by installed_rank desc limit 1'`.

**V37, V38 and V39 have all been applied to a real PostgreSQL and validated against the entity
mappings**, and V37/V38 shipped in `1.7.0`. V39 shipped in `1.8.0` on 2026-08-09 — ⚠️ *pushed to
`main`, with the deploy itself unconfirmed*; see [STATUS.md](../STATUS.md).

The chain was run for real rather than inferred: no Docker was available, but the PostgreSQL 16
server binaries were, so a throwaway cluster was enough to apply all 39 migrations to an empty
database, pass `ddl-auto: validate`, and boot the application. **That is what caught V39's
`CHAR(64)`** — see the V39 row below.

✅ **V36 shipped and is in production**, along with the rest of the chain through V41. This
paragraph used to say it was "written, not merged and not deployed", sitting on a branch and never
run against a real database. That was true when it was written and stopped being true several
releases ago; it is corrected here rather than left, because a migration table that understates how
far the schema has moved is the one thing this file exists not to be.

This supersedes the table in [backend/architecture/ARCHITECTURE.md](../backend/architecture/ARCHITECTURE.md),
which stops at V13.

Flyway runs these in order on startup. The version number is the contract: **never renumber, never
edit an applied migration.** A migration that has run somewhere is history, and history is not
editable — correct it with a new one.

| # | File | What it does |
|---|------|--------------|
| V1 | `V1__initial_schema.sql` | Complete baseline schema (all tables, indexes, seed) |
| V2 | `V2__player_stats_goal_types.sql` | `solo_goals`, `assisted_goals`, `penalty_goals` on `player_stats` |
| V3 | `V3__player_aggregate_stats.sql` | `total_matches_played`, `total_goals`, `total_assists` on `players` (+ backfill) |
| V4 | `V4__draft_sessions.sql` | Draft-session tables for the interactive Captain Pick feature |
| V5 | `V5__optimistic_lock_and_season_constraints.sql` | `players.version` optimistic-lock column; `matches.season_id` set `NOT NULL` |
| V6 | `V6__draft_session_version.sql` | `draft_sessions.version` optimistic-lock column |
| V7 | `V7__history_applied_stats.sql` | `skill_rating_history.goals_applied` / `assists_applied` — the contribution each rating application consumed, so a recalculation reverses exactly even after a stat amendment |
| V8 | `V8__missing_fk_indexes.sql` | Indexes for previously unindexed FKs (`player_confirmations.player_id`, `goals.match_team_id`, `goals.assister_id`, `matches.winning_team_id`) + a composite for the confirmation-ordered query |
| V9 | `V9__match_needs_recalc.sql` | `matches.needs_recalc` — durable marker for a failed asynchronous rating recalculation |
| V10 | `V10__single_current_season.sql` | `ux_seasons_single_current` — partial unique index over `is_current = TRUE`. Previously a second current row made `findByCurrentTrue()` raise, 500-ing every season-resolving request |
| V11 | `V11__player_anonymized_at.sql` | `players.anonymized_at` — records a GDPR erasure. Erasure anonymises in place rather than deleting, because `players(id)` cascades into `player_stats`, `goals` and `skill_rating_history`, and a delete would rewrite other players' records |
| V12 | `V12__push_subscriptions.sql` | `push_subscriptions` + `notification_mutes` — Web Push registrations and per-category opt-outs. Both cascade from `users`, so GDPR erasure removes them |
| V13 | `V13__match_plan_reminder_guards.sql` | `match_plans.deadline_reminder_sent_at` / `match_reminder_sent_at` — conditional-update guards so the reminder scheduler is safe on more than one instance and across restarts |
| V14 | `V14__match_mvp_votes.sql` | `match_mvp_votes`, plus `matches.crowd_mvp_player_id` / `mvp_voting_closes_at` / `mvp_resolved_at`. The crowd's pick is a **different fact** from `player_stats.is_mvp`, which is the admin's, and the two are kept apart on purpose. Includes a partial index over matches pending resolution |
| V15 | `V15__player_badges.sql` | `player_badges` — a badge is a record that a threshold was crossed, derived from aggregates that already exist. No new counters |
| V16 | `V16__app_settings.sql` | `app_settings` — values that were compile-time constants (MOTM voting window, ranking qualification threshold, leaderboard sizes). They are competition rules, and changing one should not be a redeploy |
| V17 | `V17__match_plan_kickoff_and_lifecycle.sql` | `match_plans.proposed_date` `DATE` → `TIMESTAMPTZ`, so a plan can record a kickoff **time**; and a `GENERATED` terminal status (the status check constraint is dropped and rebuilt). Not backfilled precisely, because `Match` has no FK to `MatchPlan` |
| V18 | `V18__composable_roles.sql` | `user_roles` join table, and **`ALTER TABLE users DROP COLUMN role`**. One role per user became a set, so running the matches, handling the money and administering the system can be held by different people — which the fee ledger requires |
| V19 | `V19__match_fee_ledger.sql` | `match_plans.total_cost_cents` (`NULL` = not recorded, `0` = the match was free), plus append-only `player_charges` and `player_payments`. Balance is derived as `SUM(payments) − SUM(charges)`; there is no allocation table. **No backfill** — inventing figures for past matches would produce debts nobody agreed to |
| V20 | `V20__payment_delegation.sql` | `payment_delegations` — a standing debtor → payer mapping recording **who the organiser chases**, plus `player_payments.paid_by_player_id` for who physically handed money over. No charge or payment moves: `uq_player_charges_plan` forbids a second charge on the payer for the same plan, charge amounts are frozen at creation, and the per-player breakdown is the requirement. Ended, never deleted (with one carve-out: the delegations of a hard-deleted never-played guest are deleted with them — ending one would leave a row referencing a removed player, which is exactly how guest removal first broke in production); one active payer per debtor via a partial unique index. [Plan](../backend/plans/PAYMENT-DELEGATION-PLAN.md) |
| V21 | `V21__guest_players.sql` | `players.is_guest` (state, cleared on promotion) + `players.invited_by_player_id` (provenance, kept) + `chk_guest_has_no_account`, which fixes the ordering promote → register → link. `DEFAULT FALSE` means no backfill: every existing player is a member by definition. [Plan](../backend/plans/GUEST-PLAYERS-PLAN.md) |
| V22 | `V22__create_organizations.sql` | `organizations`, and the row that turns "the whole players table" into organization #1. Name is a deliberate placeholder — the founder renames it at onboarding. [Plan](../backend/plans/TENANCY-SCHEMA-PLAN.md) |
| V23 | `V23__create_memberships_backfill_org1.sql` | `memberships` + `membership_roles`; every account joins org #1 with its V18 roles copied across. **`user_roles` is frozen from here** (expand/contract): auth reads the membership, the role-update endpoint dual-writes both, and the drop waits for a release of soak |
| V24 | `V24__add_tenant_id_everywhere.sql` | `tenant_id BIGINT NOT NULL` (backfilled `1`) + FK + index on all 15 owned tables. `users`, `push_subscriptions` and `notification_mutes` stay platform-level — an account and its devices belong to a person, not a group |
| V25 | `V25__composite_tenant_fks.sql` | ~29 child FKs recreated as `(tenant_id, parent_id)`, making a cross-tenant reference unrepresentable at the SQL layer rather than merely unwritten by the application |
| V26 | `V26__rescope_unique_constraints.sql` | Per-tenant uniques: one player per account **per group**, season names per group, and one current season per group |
| V27 | `V27__app_settings_tenant_pk.sql` | `app_settings` PK → `(tenant_id, setting_key)`. Absence of a row still means "on the default", so a new group needs zero seeded rows — V16's design paying off |
| V28 | `V28__platform_admins.sql` | `platform_admins` — the operator grant, flat and separate from the role model, because every founder will hold group `GROUP_ADMIN`. Ships **empty**: no backfill and no "promote the first admin", since a migration that silently grants platform-wide powers is the opposite of a dark launch. [Plan](../backend/plans/TENANCY-ENFORCEMENT-PLAN.md) §9 |
| V29 | `V29__drop_user_roles.sql` | **`DROP TABLE user_roles`** — the contract half of V23's expand/contract, written once the soak had actually started rather than at the V28 the plan pencilled in. **The point of no return:** pre-V22 code resolves authorities from `users.roles` → `user_roles`, so past here a rollback to before V22 means restoring a backup, not redeploying an old jar. No data is lost — V23 copied every row and the dual-write kept them level; the migration carries the `EXCEPT` query to prove it before running. [Plan](../backend/plans/TENANCY-ENFORCEMENT-PLAN.md) |
| V30 | `V30__group_invites.sql` | `group_invites` — server-issued, single-use, expiring tokens, **not** shareable role-bearing URLs, which would be credentials with no revocation and no audit. The grants belong to the invite rather than its author, so an GROUP_ADMIN may mint a MANAGER invite and a stolen token grants exactly what it says, once. Completes the guest arc GUEST-PLAYERS-PLAN §2 deferred: guest plays → manager promotes → invite → person registers and accepts → links to the player row. [Plan](../backend/plans/GROUP-ONBOARDING-PLAN.md) |
| V31 | `V31__group_creation_codes.sql` | `group_creation_codes` — founding a group needs a single-use operator-issued code. Registration stays open and free; a *group* is what the code buys. Open self-serve was rejected: with no billing, no metering and no abuse controls it would open an unbounded free-rider window in the very release that makes the product visible, and a code keeps the flip reversible with no deploy. **The table ships empty.** When billing lands these become promo/trial codes rather than dead weight. [Plan](../backend/plans/GROUP-ONBOARDING-PLAN.md) |
| V32 | `V32__membership_status_suspended.sql` | Widens `chk_memberships_status`, which V23 wrote as `IN ('ACTIVE')` because that was the only status then. **Fixes a group administrator being able to end a member's access to every group**: removing somebody was `DELETE /api/users/{id}`, which writes `users.is_active` — the flag governing login everywhere — while the guard said `GROUP_ADMIN`, a per-membership grant since V23. `SUSPENDED` is the group-scoped answer: it writes one tenant's membership, so every other group and the account are bit-identical. Chosen over deleting the row because that is what the erasure fork already does, and it anonymises the `Player` with it — right for "erase me", far too much for "not this season". The constraint is **widened rather than dropped** so a typo stays a write-time violation instead of a membership that silently resolves nowhere. No data migration; every existing row is and stays `ACTIVE`. Shipped in `1.2.0`. |
| V33 | `V33__rename_admin_to_group_admin.sql` | Renames the stored grant `ADMIN` → `GROUP_ADMIN`. **The name was the root cause of what V32 fixed**: roles have been per-membership since V23, so it always meant "administrator *of one group*", but `hasRole('ADMIN')` on an endpoint writing a platform-wide field read as correct to everyone who looked at it. A guard reading `hasRole('GROUP_ADMIN')` on that same write looks wrong at a glance. V23's CHECK constraint lists the permitted values, so it comes off before the `UPDATE` and back on after; the migration ends by asserting no row still holds `ADMIN`, because a survivor would resolve to **no authority at all** — an administrator silently demoted. **There is deliberately no `PLATFORM_ADMIN`**: every value here is granted per membership, so a platform-level constant would be grantable by any group administrator inside their own group. ⚠️ **Rollback boundary** — see below. Shipped in `1.2.0`. |

| V34 | `V34__platform_settings.sql` | `platform_settings` — figures that belong to the **deployment**, not to a group, the first being how far ahead a recurring match plan may be scheduled. **No `tenant_id`, deliberately**, which is the whole difference from `app_settings`: there is no tenant to scope to, and a query here carrying a tenant predicate would be the mistake. It sits behind `PlatformGuard`, not `GROUP_ADMIN` — a cap in `app_settings` would have been self-service for exactly the people it constrains. Same shape as pre-V27 `app_settings` (a row exists only where an operator overrode something, the enum default is the fallback, deleting a row is the reset), because re-deriving that differently would mean two things to learn instead of one. **Ships empty.** [Recurring plans](../backend/features/MATCH_PLANS_FEATURE.md#weekly-recurring-runs) |
| V35 | `V35__platform_admin_exclusivity.sql` | **Comment-only — no schema change, no data change.** States the invariant that an account is *either* a platform operator *or* a member of groups, never both; somebody who needs both holds two logins. Worth a migration rather than a convention because an account that is both is a session sometimes bounded by a tenant and sometimes not, and every "which group is this" branch would carry the ambiguity forever. It has already cost one production bug — `GET /api/users/me` answered 404 to any group-less account, because the read asked "is this user a member of the current group" about a user reading *themselves*. Enforced in `PlatformGuard.assertNotOperator`, called from the only two paths that grant a membership, **before** the single-use token is consumed. [Incident](../backend/fixes/INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md) |
| V36 | `V36__player_positions_and_keeper.sql` | `player_positions` (an `@ElementCollection`) plus `players.goalkeeper_willingness`. **Two fields because they answer two questions** — where somebody likes to play is a preference and can have several answers at once, so it is a set; whether they will go in goal is a willingness, and the honest kickabout answer is usually "if nobody else will", which a boolean cannot hold. **No `tenant_id` on `player_positions`**, following the exception V24 made for the `draft_session_team_*` tables: Hibernate owns the table and the parent `players` row carries tenancy. **No backfill** — an existing player has answered neither question and an empty set says exactly that. One invariant is deliberately *not* in the schema: `GOALKEEPER` among the positions implies `HAPPY_TO`, which spans a table and a column and is held by `Player.reconcileGoalkeeperWillingness()` instead of a trigger. [Players](../backend/features/PLAYER_FEATURE.md) |
| V37 | `V37__season_awards.sql` | `season_awards` — the honours board: who won what in which season, and by how much. **Written once when a season is finalised and never updated**, which is the whole design: computing awards on read would be a second aggregation over `player_stats` that disagrees with the first the moment a completed match is amended, and a Golden Boot that quietly changes hands two months after the medal was handed out is worse than no medal. **`value` is stored**, not just the winner — "top scorer" without the number is a claim nobody can check, and the number is exactly what becomes irrecoverable. **Ties get a row each**: the unique constraint is `(tenant_id, season_id, award, player_id)` rather than `(tenant_id, season_id, award)`, because two players on five goals both won it and picking one would be the software inventing a result. Composite FKs per V25; `ON DELETE CASCADE` on the player so erasure does not leave the honours board naming somebody who asked to be forgotten — an *anonymised* player keeps their row and their awards, which is correct, since the season happened. **No backfill**: no season has ever been finalised through an API that did not exist until the release before this one. [Seasons](../backend/features/SEASON_FEATURE.md) |
| V38 | `V38__ballon_dor_poll.sql` | `ballon_dor_polls` and `ballon_dor_votes`, plus V37's award CHECK widened with `BALLON_DOR`. **The window is measured in matches, not days** — `baseline_matches` records the completed matches *outside* the voted season when the poll opened, and the close condition is recompared against the matches table on every completion. A counter incremented by the async listener would be wrong forever, and silently, if it ever failed; a baseline makes a missed call cost only a delay. `closes_after_matches` is stored per poll rather than read from a constant, so changing the rule does not retroactively close or reopen every poll in flight. **The voter is an account**, not a player — this is the group's opinion of a season, unlike the MOTM vote, which is restricted to the players who appeared in the match. Winner FK is `ON DELETE SET NULL`: erasing the winner must not delete the record that the vote happened, while the `season_awards` row carrying their name goes with V37's cascade. Widening the CHECK means dropping and recreating it — there is no ALTER for the predicate — which is safe here because one code path writes the table and no existing row can violate the wider constraint. [Seasons](../backend/features/SEASON_FEATURE.md) |
| V39 | `V39__api_tokens.sql` | `api_tokens` — a long-lived, narrowly-scoped, revocable credential for something that is not a browser. **`token_hash CHAR(64)` and never the token**: the table is the thing that leaks, and a backup on a laptop must not yield working credentials. **SHA-256 rather than bcrypt, and unsalted**, which is deliberate twice over — the value is 32 bytes of CSPRNG output, so there is no dictionary to attack and nothing for a slow hash to defend, while the hash *is* computed on every authenticated request; and a per-row salt would defeat the lookup outright, because the row is found **by** the hash and so it has to be deterministic. `token_prefix` stores eight characters in clear so a person can tell two of their own tokens apart without the secret. **`tenant_id` is on the row, not derived**: a token is minted for one group and cannot leave it, so the group is a stored fact rather than something re-read per request from a header the caller controls — a contradicting `X-Group-Id` is a 404. `ON DELETE CASCADE` on the user because GDPR erasure must take the tokens with it: a live credential belonging to a deleted account works and nobody owns it. `expires_at` is `NOT NULL` — there is no non-expiring option, since a credential with no end date is one nobody ever revisits. Revocation tombstones (`revoked_at`) rather than deleting, so the list can still show what existed and when it was last used. **`token_hash` is VARCHAR(64) and not CHAR(64)**, which is how it was first written and which broke every integration test at once: PostgreSQL's `char(n)` is `bpchar`, Hibernate maps `String` to `varchar`, and `ddl-auto: validate` therefore refused to build the EntityManagerFactory — so all 86 tests failed without reaching what they test. It was the only CHAR in the schema. VARCHAR is the right column anyway: `char(n)` blank-pads, which for a column that exists to be looked up by equality is a latent correctness hazard rather than a style question. **No backfill**, and nothing to backfill: no token has ever existed. [API tokens contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API-TOKENS-API-CONTRACT.md) · [plan](../backend/plans/SCOPED-API-TOKENS-PLAN.md) |
| V40 | `V40__match_plan_drop_description_and_backfill_deadlines.sql` | **`match_plans.description` dropped**, and `confirmation_deadline` backfilled where it was null. ⚠️ **This is a hard rollback boundary and the drop is not reversible** — the previous jar's entity maps a column that no longer exists, so `ddl-auto: validate` refuses to start it. Same shape as the V22/V29 contract steps: rolling back past V40 means restoring the column first, and it comes back **empty**. The drop is the point — `description` was free text beside a title, a location, a kickoff and a match type, and it never carried anything those four did not already say. A second free-text field next to a title is an invitation to put the real information in the wrong one, and once somebody does, the list view (which shows the title) stops telling the truth about the plan. **The backfill fixes a live defect, not a tidiness one**: `MatchPlanMapper` reads a null deadline as *poll open*, so every plan without one had a poll that never closed, taking confirmations through kickoff and long after the final whistle. Five minutes matches `DEFAULT_DEADLINE_LEAD_MINUTES`. **Past plans are included on purpose** — skipping them would leave exactly the rows whose polls are still wrongly open, and a March plan with a null deadline is the worst case rather than the harmless one; a deadline five minutes before a kickoff that has already happened is in the past, which closes the poll, which is the right answer for a match that has been played. The column **stays nullable**: `NOT NULL` would be a promise the schema cannot keep, since the deadline is derived from `proposed_date` at write time by `MatchPlanService` and a row inserted by any other path has no business being refused by the database for a rule that lives in the service. [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md) |
| V41 | `V41__chat_and_presence.sql` | Five tables for group chat — `chat_conversations`, `chat_participants`, `chat_messages`, plus the moderation and presence tables — shipped in `2.0.0` on 2026-08-23 and **the latest migration in production**. **Built around deletion being the normal case**: messages are destroyed 24 hours after they are written, and an ad-hoc `GROUP` chat dies after 12 hours of silence and takes its surviving messages with it. **There is no `deleted_at` anywhere, deliberately** — "permanently deleted" was explicit in the request, and a soft-deleted row is still a row that a future query, an export or a backup can surface. Retention is `DELETE`. **Roles grant nothing here, and the schema is what makes that cheap**: an administrator who is in a chat is an ordinary participant, one who is not cannot see that the conversation exists, and authorization asks exactly one question — is the caller a participant? — which is a row lookup. No column records a role, a grant or a visibility flag, because the moment one exists somebody will read it. Three conversation kinds that differ in more than name: `DIRECT` never dies and persists as an empty container once its messages expire, which is what stops "message X again" opening a second thread; `GROUP` dies on inactivity or when its creator deletes it; `EVERYONE` is the one permanent group-wide channel, created on first use and never deleted. **`EVERYONE`'s membership is not `chat_participants`** — materialising a row per member would need maintenance on every join, suspension and departure and would be wrong between those events, so the authority is the thing already correct by construction, an `ACTIVE` membership in the tenant; participant rows exist for it as read cursors (`last_read_at`), not as permission. ⚠️ **The SSE layer this feeds is a per-JVM in-memory registry**, which is safe only because production is `web=1` on Heroku Basic — a tier that cannot scale horizontally at all. That assumption stops being guaranteed the day the app moves to Standard or Performance. [Chat plan](../backend/plans/GROUP-CHAT-PLAN.md) |
| V42 | `V42__push_subscription_account_scoping.sql` | `push_subscriptions.active` + key change from `UNIQUE (endpoint)` to `UNIQUE (user_id, endpoint)`, with a **partial unique index over `active`** so at most one registration per endpoint is live. V12 keyed a registration on the endpoint alone, which conflated two different things: the endpoint identifies a browser *install* — the Push API mints one per origin and hands back the same one whoever is signed in — so it is the identity of the **channel**, not of a **registration**. On any device that saw two accounts the row kept pointing at whoever registered first, so their notifications were delivered to and displayed on a device somebody else was now using (the service worker renders whatever arrives; it has no session and cannot check), while the second account had no registration and no way to get one — the frontend asked the browser "is there a subscription?" rather than the server "is it mine?", so its toggle already read *on* and it never called `/subscribe`. **Ownership moves but is never shared**, because there is only one channel; what is not forced is what happens to the loser's row, and it is kept **dormant rather than deleted**, so switching back to an account on that device restores its choice instead of asking for consent twice. Only active rows are sent to — a dormant row is not a quieter subscription, it is no subscription. **No backfill**: existing rows are all live registrations and the old constraint guaranteed one per endpoint, so `DEFAULT TRUE` is already correct and the new partial index cannot collide. [Push contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PUSH-API-CONTRACT.md) |
| V43 | `V43__password_reset_tokens.sql` | `password_reset_tokens` — the first way back into an account whose password nobody remembers. Before it, recovery was `UPDATE users SET password = …` by hand, which made whoever runs the deployment the support desk and needed a bcrypt hash produced off-system. **Only the hash is stored**, following `api_tokens` (V39) rather than `group_invites` (V12), which holds its token in clear: this table is the thing that leaks — a backup, a screenshot of a query — and a leaked table must not yield working credentials. SHA-256 unsalted for V39's reasons, and **`VARCHAR(64)` not `char(64)`**, which is V39's documented mistake: `char(n)` is `bpchar`, Hibernate maps `String` to `varchar`, and `ddl-auto: validate` therefore refuses to build the EntityManagerFactory and fails every test at once. **Three states, not one flag** — `used_at` and `superseded_at` are separate columns because "this link was redeemed" and "you asked again, so this one was retired" are different events, and telling somebody the first when the second happened reads as *another person got into my account*. Requesting again supersedes whatever came before it, so clicking "forgot password" three times while hunting through an inbox does not leave three working keys to the account behind. `issued_by` is `ON DELETE SET NULL` rather than a cascade: erasing the admin who issued a reset must not delete the record that it happened to somebody else. **No `tenant_id`**, following V24's carve-out for `users` and `push_subscriptions` — an account is platform-level and so is the ability to sign into it, even when a group admin is the one who issued the link. **No backfill**: no reset has ever existed. [Password reset contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PASSWORD-RESET-API-CONTRACT.md) |
| V44 | `V44__goal_events.sql` | `goals.event_id` and `goals.occurred_at`, both nullable, plus `UNIQUE (tenant_id, event_id)`. **The `goals` table has existed since `V1` and nothing had ever written to it** — every rating the engine has produced came from the `PlayerStat` counters, and the timing-weighted branch that reads `Goal` rows was unreachable code. This is the migration that fills it. **`event_id` is the client's idempotency key, not the server's**, because the caller this exists for captures goals where there is no network and drains a queue later: a repeat carrying a known key answers `200` with the goal already recorded rather than `409`, since an error would stop the drain and stop it on the same item every time afterwards, jamming the queue permanently on a goal that was never lost. Unique **per tenant** rather than globally — two groups cannot see each other's keys, so a collision between them is not a duplicate, and a global constraint would let one group's key silently reject another's goal. **`occurred_at` is the client's clock and the server never substitutes its own**: a goal queued at 15:12 and posted at 16:40 belongs at 15:12, and stamping it on receipt would put every delayed goal at the end of the match, where it reads as a scoring bug rather than a clock bug. `minute` stays unused — it needs a kickoff to count from, and `matches.match_date` is a schedulable, backdatable date rather than one — so the goal ordering query sorts on `occurred_at` ahead of `created_at`, because a queue drained in one burst shares a `created_at` to the second. **`NULL` for every pre-existing row**, which is the whole point: additive and nullable, so the previous jar starts against the same schema and rollback is a redeploy. [Goal events contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MATCH-GOAL-EVENTS-API-CONTRACT.md) |
| V45–V49 ⚠️ | `V45__match_clock.sql` · `V46__user_token_version.sql` · `V47__player_fee_reminder.sql` · `V48__notification_preference_enabled.sql` · `V49__match_chat.sql` | **Shipped, not yet described here** — found missing when V50 was added (2026-09-04). Each file opens with its own reasoning comment; read that until these rows are written. Registered in the drift table |
| V50 | `V50__rank_ladder.sql` | **The rank ladder's columns, and nothing that reads them.** Five on `players` — `rank_tier`, `rank_division`, `form_points`, `rank_shield_left`, `season_placements_played` — and nine on `skill_rating_history`: the tier, division and points *before and after* the row, the shield and placement count before it, and a `rank_event`. **A snapshot, not a delta**, because the ladder is not additive: a promotion resets the points inside the new division, so "what this row added" is not a number the reverse can subtract, and it restores the five-field state instead. `fp_before IS NULL` marks a row from before this migration; the first whole-set replay fills it. **Nothing backfilled here** — the movement for a match is computed inside the same forward pass as its rating, and that pass already exists (the 3.4.2 replay), so the backfill is `POST /api/matches/recalculate` with an empty body, once per group, after deploy. `INTEGER` rather than `SMALLINT` for the counters because `ddl-auto: validate` maps `Integer` to `integer`. Additive and nullable-or-defaulted, so the previous jar starts against it: **not a rollback boundary**. Step 1 of [RANK-LADDER-PLAN](../backend/plans/RANK-LADDER-PLAN.md), [FootMania-Back#275](https://github.com/ricsnsuka/FootMania-Back/pull/275) |

---

## Deployment state

⚠️ **V42 moves the rollback boundary, and it is the newest one.** It drops
`uq_push_subscriptions_endpoint` — the constraint the 2.1.0 jar relies on for "one registration per
browser". That jar *starts* against the V42 schema and no longer has the uniqueness it assumes: it
can write a second live row for an endpoint and then fail its next insert against the new partial
unique index, which is a failure at the point of subscribing rather than at boot, where nothing is
watching. **V43 and V44 in the same release are not boundaries** — V43 only adds a table nothing
before it reads, and V44's two `goals` columns are additive and nullable. So rolling the backend
back *within* 2.2.0's chain is a redeploy; rolling back *past* it is a restore.

⚠️ **V40 is a harder boundary than V42, and it is already in production.** It **drops**
`match_plans.description`, so the previous jar does not merely lose a guarantee — it does not
start: the old entity maps a column that no longer exists and `ddl-auto: validate` refuses to build
the EntityManagerFactory. Rolling back past V40 means restoring the column first, and it comes back
**empty**, because the drop discarded what was in it. Same shape as the V22/V29 contract steps.

⚠️ **V33 moves the rollback boundary again.** Past it the previous jar cannot parse `GROUP_ADMIN`
— `Role.valueOf` throws for every membership carrying it, so authorities fail to resolve for
exactly the accounts with the most to lose. Rolling the code back past V33 means running the
inverse `UPDATE` first. Anything forward of it is fine. Same shape as V29's boundary, and recorded
here for the same reason.

**Live sessions survive V33**, which is worth knowing before anyone plans a maintenance window:
authorities are re-derived from `membership_roles` on every request rather than read from the JWT's
`roles` claim, so tokens issued before the rename keep working and nobody is logged out. The
frontend's *stored* copy is the stale one, and it is migrated client-side on rehydrate.

✅ **V32 shipped in `1.2.0` on 2026-08-04**, so the deployed schema accepts `SUSPENDED`. While it
was merged and undeployed, the deployed schema still refused the value and the code writing it would
have failed on the CHECK constraint — which is how the integration suite found the constraint in the
first place, and why **the backend deploys before the frontend that offers suspension**, per
[CONTRIBUTING](../CONTRIBUTING.md#deployment-order).

**The whole chain through V31 was applied in production on 2026-08-02**; V32–V35 followed on
2026-08-04 with `1.2.0` and `1.3.0`. V22–V28, the Phase 5a schema and the platform grant, went out as one
release, dark; V29–V31 followed the same day. The notes below record the state at each step,
newest first.

**V29–V31 were applied on the same day.** The 5a-4 rung — the `user_roles` drop plus invites and
creation codes — is in production, which means **the whole chain through V31 is live** and Phase 5a
is complete rather than partial.

**The rollback boundary moved with it.** Past V29 there is no `user_roles`, so a pre-V22 release
would resolve no authorities at all and lock every administrator out of their own group. Rolling
back to before V22 now means restoring a backup; anything back to the current release is fine,
because `membership_roles` has been the source of truth since V23. Intended end state of an
expand/contract, written down because the next person reaching for a rollback deserves to know
which kinds still work.

V31 shipped `group_creation_codes` **empty**, and it stays empty until an operator issues one.
Deploying this chain did not make anything self-serve: the endpoint existing is not the flip, a
code existing is.

**The chain through V21 was applied as of 2026-08-01** — the guest-players
feature was observed live the day V20/V21 merged, which means V17–V19 (including V18's column
drop) went through cleanly too. The long-standing "never run against a real database" hazard is
closed; STATUS.md records it as resolved.

Two things survive it:

- ~~`GuestIsolationIT` and the rest of the integration suite remain opt-in and still unexecuted.~~
  **Resolved 2026-08-02.** `integrationTest` runs on every pull request and the whole tier —
  `GuestIsolationIT`, `TenancySchemaIT`, `TenantIsolationIT`, `MigrationSchemaValidationIT` — has
  now executed against a real PostgreSQL container. It is still not part of `check`, so a local
  `./gradlew build` proves nothing about the constraints; CI is the gate.
- The day-one production defect in guest removal (see the guest plan §12b) was a flush-ordering
  bug no mocked test could see. Real-persistence coverage for delete paths is now the house rule.

### Reserved — specced not written

| # | Concern | Plan |
|---|---------|------|
| V32+ | `org_subscriptions` + entitlements | [GROUP-BILLING-PLAN](../backend/plans/GROUP-BILLING-PLAN.md), design-only — **on hold** |

*V29 and V30 were reserved here and are now written (above). V31 went to
`group_creation_codes` rather than to billing, which is why the billing reservation moved to V32+.*

**Why the `user_roles` drop is V29 and not V28 — a decision, now history.** The enforcement plan
pencilled the drop in at V28 on the assumption the soak would have elapsed by then. It had not:
V22–V27 were merged and undeployed at the time, so dropping the table would have removed the way
back from a rollback to pre-V22 code. V28 became the platform-admin grant instead and the drop took
a later number. Proposed by the implementer 2026-08-02 and confirmed by the owner the same day.

**Both are now deployed** (2026-08-02), so this section records why the numbering is what it is,
not a constraint still to be observed.

V22–V28 ship as **one release** (the deploy is the migration) behind a DB-backup gate and a green
`integrationTest` — now wired into CI, and green: `TenantIsolationIT` and `TenancySchemaIT` both
executed for the first time on 2026-08-02. The constraint work in this chain is invisible to the
H2 unit tier, so that tier passing means nothing here.

The pre-flight for the next migration is unchanged:

```bash
./gradlew integrationTest    # real Flyway chain on a PostgreSQL Testcontainer,
                             # with ddl-auto: validate to catch entity/migration drift
```

It is deliberately **not** wired into `check`, so it only runs when asked locally. It requires
Docker. CI runs it on every pull request, which is what made the first execution happen at all —
worth remembering the next time something is "opt-in for now".

## Conventions

- One concern per migration; the filename says which.
- Every migration in the tree opens with a comment block explaining **why**, not what — the SQL
  already says what. Keep that up; it is the reason this table could be reconstructed accurately.
- Entity and migration must agree. `MigrationSchemaValidationIT` is what proves it, by running the
  real chain with Hibernate validation on.
