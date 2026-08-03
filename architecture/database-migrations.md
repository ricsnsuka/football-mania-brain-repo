# Database Migration History

Complete as of **V33**, verified against `src/main/resources/db/migration/` on 2026-08-03.
**Production is applied through V31.** V32 and V33 are merged to `next` and **not deployed**.

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
| V32 ⏳ | `V32__membership_status_suspended.sql` | Widens `chk_memberships_status`, which V23 wrote as `IN ('ACTIVE')` because that was the only status then. **Fixes a group administrator being able to end a member's access to every group**: removing somebody was `DELETE /api/users/{id}`, which writes `users.is_active` — the flag governing login everywhere — while the guard said `GROUP_ADMIN`, a per-membership grant since V23. `SUSPENDED` is the group-scoped answer: it writes one tenant's membership, so every other group and the account are bit-identical. Chosen over deleting the row because that is what the erasure fork already does, and it anonymises the `Player` with it — right for "erase me", far too much for "not this season". The constraint is **widened rather than dropped** so a typo stays a write-time violation instead of a membership that silently resolves nowhere. No data migration; every existing row is and stays `ACTIVE`. **Not deployed.** |
| V33 ⏳ | `V33__rename_admin_to_group_admin.sql` | Renames the stored grant `ADMIN` → `GROUP_ADMIN`. **The name was the root cause of what V32 fixed**: roles have been per-membership since V23, so it always meant "administrator *of one group*", but `hasRole('ADMIN')` on an endpoint writing a platform-wide field read as correct to everyone who looked at it. A guard reading `hasRole('GROUP_ADMIN')` on that same write looks wrong at a glance. V23's CHECK constraint lists the permitted values, so it comes off before the `UPDATE` and back on after; the migration ends by asserting no row still holds `ADMIN`, because a survivor would resolve to **no authority at all** — an administrator silently demoted. **There is deliberately no `PLATFORM_ADMIN`**: every value here is granted per membership, so a platform-level constant would be grantable by any group administrator inside their own group. ⚠️ **Rollback boundary** — see below. **Not deployed.** |

---

## Deployment state

⚠️ **V33 moves the rollback boundary again.** Past it the previous jar cannot parse `GROUP_ADMIN`
— `Role.valueOf` throws for every membership carrying it, so authorities fail to resolve for
exactly the accounts with the most to lose. Rolling the code back past V33 means running the
inverse `UPDATE` first. Anything forward of it is fine. Same shape as V29's boundary, and recorded
here for the same reason.

**Live sessions survive V33**, which is worth knowing before anyone plans a maintenance window:
authorities are re-derived from `membership_roles` on every request rather than read from the JWT's
`roles` claim, so tokens issued before the rename keep working and nobody is logged out. The
frontend's *stored* copy is the stale one, and it is migrated client-side on rehydrate.

⏳ **V32 is merged and undeployed.** It rides on `next` with the fix that needs it, so until the
next release the deployed schema still refuses `SUSPENDED` — the code that writes it would fail on
the CHECK constraint, which is how the integration suite found the constraint in the first place.
**Deploy the backend before the frontend that offers suspension**, per
[CONTRIBUTING](../CONTRIBUTING.md#deployment-order).

**The whole chain through V31 is applied in production as of 2026-08-02** — every migration in the
table above except V32. V22–V28, the Phase 5a schema and the platform grant, went out as one
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
