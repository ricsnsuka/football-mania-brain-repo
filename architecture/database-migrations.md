# Database Migration History

Complete as of **V19**, verified against `src/main/resources/db/migration/` on 2026-07-31.

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

### Reserved — specced, not yet written

| # | File | Spec |
|---|------|------|
| V20 | `V20__payment_delegation.sql` | [PAYMENT-DELEGATION-PLAN](../backend/plans/PAYMENT-DELEGATION-PLAN.md) — `payment_delegations` (standing debtor → payer responsibility) + `player_payments.paid_by_player_id` |
| V21 | `V21__guest_players.sql` | [GUEST-PLAYERS-PLAN](../backend/plans/GUEST-PLAYERS-PLAN.md) — `players.is_guest` + `players.invited_by_player_id` + `chk_guest_has_no_account`. Ships after V20 |

Both stack on top of V17–V19, which have never run against a real database (below) — run
`./gradlew integrationTest` before writing either, and move each row into the main table when it
merges.

---

## Deployment state — read before deploying

**V17, V18 and V19 have never run against a real database.** They are merged and green against
Testcontainers only. V18 **drops a column**, so a failed run there is not a small problem.

The pre-flight that exists:

```bash
./gradlew integrationTest    # real Flyway chain on a PostgreSQL Testcontainer,
                             # with ddl-auto: validate to catch entity/migration drift
```

It is deliberately **not** wired into `check`, so it only runs when asked. It requires Docker.

## Conventions

- One concern per migration; the filename says which.
- Every migration in the tree opens with a comment block explaining **why**, not what — the SQL
  already says what. Keep that up; it is the reason this table could be reconstructed accurately.
- Entity and migration must agree. `MigrationSchemaValidationIT` is what proves it, by running the
  real chain with Hibernate validation on.
