# Organizations & Memberships (Tenancy Schema) — Technical Specification

**Date:** 2026-08-01
**Status:** ✅ **SHIPPED 2026-08-01** — Phase 5a-1, merged in `77c3ca8`. V22–V27 applied by
`integrationTest` in CI; **not yet deployed to production.** The 939-test unit suite passed with
only mechanical fixture changes, which was the dark-launch acceptance criterion
**Priority:** HIGH — every Phase 5 deliverable stacks on this
**Estimated Effort:** L (≈3–4 days backend; no frontend beyond keeping tests green)
**Depends on:** `GUEST-PLAYERS-PLAN.md` *(shipped — this spec honours its membership promise, §4)*
**Depended on by:** `TENANCY-ENFORCEMENT-PLAN.md` → `TENANT-PRIVACY-PLAN.md` → `GROUP-ONBOARDING-PLAN.md` → `GROUP-BILLING-PLAN.md`
**Contract:** none — this rung is deliberately invisible on the wire. The first API changes land in `TENANCY-ENFORCEMENT-PLAN.md`.

---

## 1. Requirement Summary

Phase 5 turns one deployment serving one football group into a product serving many. The owner's
decisions: **multiple groups per account** (an account joins several groups, each with its own
roles), **tenancy before billing**, **picker-style group addressing** (one app, one URL, an active
group per session), and **invite-code-gated group creation** at launch.

This first spec is the ground floor: the `organizations` and `memberships` tables, a `tenant_id`
on **every** tenant-owned table with composite foreign keys, the re-scoping of every uniqueness
constraint that currently means "per deployment", and the backfill that turns the existing
production database into **organization #1** with nobody noticing.

**This rung ships dark.** After the deploy, the app behaves byte-identically: one organization,
every existing user a member with their existing roles, every query returning what it returned
yesterday. The acceptance test is that the pre-existing test suites pass unmodified.

Vocabulary: the schema and code say `tenant_id` / `organizations` (greppable, unambiguous); every
user-facing word is **group**.

---

## 2. Scope

| In | Out |
|----|-----|
| `organizations`, `memberships`, `membership_roles` tables | Any request-level tenant resolution (`TENANCY-ENFORCEMENT-PLAN.md`) |
| `tenant_id` on every tenant-owned table, composite FKs | Group creation, invites, picker UI (`GROUP-ONBOARDING-PLAN.md`) |
| Constraint re-scoping (players/user link, seasons, app_settings) | Erasure semantics rework (`TENANT-PRIVACY-PLAN.md`) |
| Backfill: this deployment becomes org #1 | Billing, entitlements (`GROUP-BILLING-PLAN.md`) |
| Membership-backed `UserDetailsServiceImpl` | Dropping `user_roles` (V28, next rung — expand/contract) |
| `TenantContext` as a constant org-#1 resolver | Hibernate filters, ownership asserts, cache keys (next rung) |
| Privacy house rule for the two new personal-data tables | Keycloak (parked — see §12 of the enforcement plan) |

---

## 3. Model decision: row-level tenancy with `tenant_id` on every table — not transitive FKs, not schema-per-tenant

**Chosen:** one database, one schema, a `tenant_id BIGINT NOT NULL REFERENCES organizations(id)`
column on every tenant-owned table, and composite foreign keys `(tenant_id, parent_id)` wherever
one tenant-owned row references another.

**Rejected: transitive tenancy** (tenant only on root tables — players, seasons, match_plans —
letting children inherit through NOT NULL parents). The schema has **eight-plus references that a
plain FK lets cross tenants silently**: `players.invited_by_player_id`,
`player_payments.paid_by_player_id`, `draft_sessions.captain_a_id/captain_b_id`,
`match_mvp_votes.voter/voted_for`, `payment_delegations.debtor/payer`,
`player_confirmations.player_id`, `matches.crowd_mvp_player_id`. No constraint can say "same
tenant as my other parent" across a plain FK. With `tenant_id` denormalised onto every table and
composite FKs, a cross-tenant reference is **unrepresentable in the database** — the class of bug
dies at the schema, independent of any ORM behaviour. The denormalisation is the price, and it is
the standard multi-tenant answer.

**Also rejected: schema-per-tenant / database-per-tenant.** Requires a
`MultiTenantConnectionProvider`, per-tenant Flyway orchestration, and breaks the "a deploy *is*
the migration" Heroku posture. The house rule is no new infrastructure unless a feature cannot
work without it; row-level tenancy works without any.

---

## 4. Model decision: membership as a first-class entity — not a `user_roles` PK extension

**Chosen:** `memberships (id, tenant_id, user_id, status, created_at, UNIQUE(tenant_id, user_id))`
plus `membership_roles (membership_id, role)`. A user's roles become **per-membership**: ADMIN of
group A, plain member of group B. Roles stay flat and composable exactly as V18 left them — a
membership *holds* a set of flat grants; nothing becomes a hierarchy.

**Rejected:** extending `user_roles`' primary key to `(user_id, tenant_id, role)`. Three reasons:
it keeps roles as an EAGER `@ElementCollection` on the hottest path in the app (loaded on every
authenticated request), and element collections are permanently invisible to Hibernate filters —
a standing leak hazard the enforcement rung could never close; it has no home for membership
**status** (`ACTIVE` now; `INVITED` in onboarding; `SUSPENDED` in billing); and it cannot
represent "joined, zero roles yet", which is the normal state of a new member (the V18 "empty set
renders as Member" model, per group).

### The guest promise, honoured literally

`GUEST-PLAYERS-PLAN.md` §3 promised: *"a guest is simply a player with no membership row."* This
spec makes that the definition:

- A **member** of a group = an `AppUser` with an ACTIVE membership row for that tenant.
- A **guest** = a `Player` row (`is_guest = TRUE`, no linked account) in that tenant — no
  membership row exists or can exist, because memberships attach to accounts and
  `chk_guest_has_no_account` forbids a guest having one.
- **Promotion stays exactly as cheap as promised**: flip `is_guest`, and later, when the person
  registers and links, their membership row is created by the join flow — promote → register →
  link → member, the V21 ordering preserved with one new final step that
  `GROUP-ONBOARDING-PLAN.md` owns.
- `isCore` tiering remains deliberately untouched, as it has been since the roadmap first warned
  about it.

---

## 5. Model decision: username and email uniqueness stay global — not per-tenant

**Chosen:** `users_username_key` and `users_email_key` remain deployment-wide. Accounts are
**platform-level**; membership is the per-tenant thing.

**Rejected:** per-tenant username/email. Login resolves username-or-email with no tenant in hand
(`UserDetailsServiceImpl` — the identifier is all the login form sends), and the owner's picker
model means authentication happens *before* group selection by design. Per-tenant identity would
make login unresolvable without a tenant discriminator the UX deliberately doesn't collect.

**Corollary:** `players_user_id_key UNIQUE(user_id)` — today "one player per account, globally" —
becomes `UNIQUE(tenant_id, user_id)`: one player per account **per group**, which
multiple-membership requires. Every `findByUserId(...)` caller is updated in this rung
(§8) because the single-`Optional` shape stops being true the moment a second membership exists.

---

## 6. Schema — V22 through V27, one release

> **One release, six migrations, one concern each** (house rule). Flyway runs on boot, so the
> deploy *is* the migration — the Order of work (§11) gates it on a database backup and a green
> `integrationTest` run. Numbers assume V21 is still head at implementation time; renumber if
> anything lands first.

### `V22__create_organizations.sql`

```sql
-- Every row in this database is about to get an owner. This is the owner.
CREATE TABLE organizations (
    id            BIGSERIAL    PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    -- ACTIVE now; billing adds SUSPENDED etc. via its own migration. VARCHAR, not enum:
    -- the set is going to grow and a CHECK is cheaper to change than a type.
    status        VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    -- Group creation is gated at launch: an operator-issued code, single-use.
    -- Lives here rather than its own table until billing turns codes into promo/trial codes.
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- The existing deployment becomes organization #1. The name is a placeholder the founder
-- renames during onboarding (GROUP-ONBOARDING-PLAN) — inventing a real name here would be
-- guessing on the group's behalf.
INSERT INTO organizations (name) VALUES ('My Group');
```

### `V23__create_memberships_backfill_org1.sql`

```sql
CREATE TABLE memberships (
    id          BIGSERIAL   PRIMARY KEY,
    tenant_id   BIGINT      NOT NULL REFERENCES organizations(id),
    user_id     BIGINT      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_membership_tenant_user UNIQUE (tenant_id, user_id)
);

CREATE TABLE membership_roles (
    membership_id BIGINT      NOT NULL REFERENCES memberships(id) ON DELETE CASCADE,
    role          VARCHAR(20) NOT NULL,
    PRIMARY KEY (membership_id, role),
    CONSTRAINT membership_roles_role_check CHECK (role IN ('ORGANIZER','MANAGER','ADMIN'))
);

-- Every existing account joins org #1 — including zero-role accounts: they registered at this
-- deployment, so this deployment's group is the one they were joining. Roles copy over from
-- user_roles, which is FROZEN from this migration on (still read by nothing after the same
-- release's code change) and dropped in V28 after a release of soak — expand/contract, the
-- same posture V18 took with users.role.
INSERT INTO memberships (tenant_id, user_id)
SELECT 1, id FROM users;

INSERT INTO membership_roles (membership_id, role)
SELECT m.id, ur.role
FROM user_roles ur JOIN memberships m ON m.user_id = ur.user_id AND m.tenant_id = 1;
```

### `V24__add_tenant_id_everywhere.sql`

One concern: the ownership column exists and is populated. For **every** tenant-owned table —
`players`, `seasons`, `matches`, `match_teams`, `player_stats`, `goals`, `skill_rating_history`,
`match_plans`, `player_confirmations`, `draft_sessions`, `push-adjacent tables stay user-owned
(see below)`, `match_mvp_votes`, `player_badges`, `app_settings`, `player_charges`,
`player_payments`, `payment_delegations`:

```sql
ALTER TABLE players ADD COLUMN tenant_id BIGINT;
UPDATE players SET tenant_id = 1;
ALTER TABLE players ALTER COLUMN tenant_id SET NOT NULL;
ALTER TABLE players ADD CONSTRAINT fk_players_tenant
    FOREIGN KEY (tenant_id) REFERENCES organizations(id);
CREATE INDEX idx_players_tenant ON players (tenant_id);
-- …repeated per table; identical shape.
```

**Deliberately not tenant-owned:** `users` (platform-level, §5), `push_subscriptions` and
`notification_mutes` (they hang off the *account*: one browser has one push endpoint per origin —
`uq_push_subscriptions_endpoint` — so a subscription belongs to the person, and notification
fan-out resolves memberships at send time; per-account mutes are one setting for one person, a
deliberate simplification), `flyway_schema_history`.

### `V25__composite_tenant_fks.sql`

One concern: cross-tenant references become unrepresentable.

```sql
-- Parents gain a composite key target…
ALTER TABLE players ADD CONSTRAINT uq_players_tenant_id UNIQUE (tenant_id, id);
-- …and every reference between tenant-owned rows re-points at it:
ALTER TABLE players DROP CONSTRAINT players_invited_by_player_id_fkey;
ALTER TABLE players ADD CONSTRAINT fk_players_invited_by
    FOREIGN KEY (tenant_id, invited_by_player_id)
    REFERENCES players (tenant_id, id) ON DELETE SET NULL;
-- …repeated for: paid_by, delegation debtor/payer, confirmations (plan+player), draft captains,
-- mvp voter/voted_for, crowd_mvp, stats/goals/history parents, teams→matches, matches→seasons,
-- charges→plans, sessions→plans. Same shape throughout.
```

(`ON DELETE` behaviours are preserved exactly — SET NULL stays SET NULL, CASCADE stays CASCADE.)

### `V26__rescope_unique_constraints.sql`

```sql
-- One player per account PER GROUP (was: globally) — required by multiple memberships.
ALTER TABLE players DROP CONSTRAINT players_user_id_key;
ALTER TABLE players ADD CONSTRAINT uq_players_tenant_user UNIQUE (tenant_id, user_id);

-- Every group will call their first season "Season 1".
ALTER TABLE seasons DROP CONSTRAINT seasons_name_key;
ALTER TABLE seasons ADD CONSTRAINT uq_seasons_tenant_name UNIQUE (tenant_id, name);

-- ONE CURRENT SEASON *PER GROUP*. The V10 global index is the single loudest boot-blocker in
-- Phase 5: SeasonRepository.findByCurrentTrue() returns Optional and is called from four
-- services — the moment a second tenant had a current season it would 500 on every
-- season-resolving request. Safe to land now: with exactly one tenant the new index is
-- trivially satisfied, and the repository signature changes in this same release (§8) so no
-- unscoped call site survives to the day a second tenant exists.
DROP INDEX ux_seasons_single_current;
CREATE UNIQUE INDEX ux_seasons_tenant_single_current
    ON seasons (tenant_id) WHERE is_current = TRUE;
```

### `V27__app_settings_tenant_pk.sql`

```sql
-- Competition rules become per-group. Existing override rows become org #1's overrides.
-- The friendliest table in the schema: absence-of-row means "on the default" (V16), so a new
-- group inherits every default with ZERO seeded rows — nothing to copy at group creation.
ALTER TABLE app_settings DROP CONSTRAINT app_settings_pkey;
ALTER TABLE app_settings ADD COLUMN tenant_id BIGINT NOT NULL DEFAULT 1
    REFERENCES organizations(id);
ALTER TABLE app_settings ALTER COLUMN tenant_id DROP DEFAULT;
ALTER TABLE app_settings ADD PRIMARY KEY (tenant_id, setting_key);
```

*(V28 — drop `user_roles` — belongs to the enforcement rung, after a release of soak on
membership-backed auth. V29 — group invites — belongs to onboarding.)*

---

## 7. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-T1 | Every tenant-owned row carries `tenant_id`, NOT NULL, stamped at creation | Enforcement rung adds the `@PrePersist` stamp; this rung's code sets it explicitly |
| BR-T2 | A reference between tenant-owned rows is composite `(tenant_id, id)` | Cross-tenant references unrepresentable — the DB is the last line of defence, and the first |
| BR-T3 | Accounts are platform-level; membership is per-tenant; roles are per-membership and stay flat | `ADMIN` of one group says nothing about any other group, and nothing about the platform |
| BR-T4 | A guest is a player with no membership row | The V21 promise, now definitional. `chk_guest_has_no_account` unchanged |
| BR-T5 | One player per account per group | `uq_players_tenant_user` |
| BR-T6 | One current season per group | `ux_seasons_tenant_single_current` |
| BR-T7 | A new group needs **zero** seeded settings/mutes/roles rows | Absence-means-default is preserved everywhere it exists today |
| BR-T8 | `user_roles` is frozen at V23 and dropped only after a release of soak (V28) | Expand/contract — the rollback story for the hottest table in the app |

---

## 8. Entities, repositories, and the minimum code to stay dark

- **`Organization`**, **`Membership`** (+ `@ElementCollection` roles on Membership — a small,
  per-membership collection loaded with the membership, not on every request), in `model/`.
- **`TenantContext`** — a static holder with exactly one implementation this rung: `currentTenant()
  returns 1L`. It exists so every touched call site compiles against the final shape; the
  enforcement rung swaps the constant for request resolution. No filters, no request protocol yet.
- **`UserDetailsServiceImpl`** re-reads authorities from `membership_roles` for org #1 instead of
  `user_roles` (the V23 freeze point). Same authorities out, same empty-set-is-member semantics.
- **`SeasonRepository.findByCurrentTrue()` → `findCurrentByTenantId(Long)`** — all four callers
  (`MatchService` ×2, `MatchPlanService`, `DraftSessionService`) updated; a post-merge grep for
  `findByCurrentTrue` returning zero hits is a checklist item.
- **`PlayerRepository.findByUserId` → `findByTenantIdAndUserId`** (and `existsBy…` likewise) —
  every caller updated (`MatchPlanService` self-RSVP/getMy, `MvpVoteService`,
  `PlayerService.updateMe/getMe/linkMe`, `PaymentController` self-endpoints, `PrivacyService`).
  `PrivacyService.exportForUser`'s single-player assumption is *mechanically* patched here
  (current tenant); its semantic redesign is `TENANT-PRIVACY-PLAN.md`.
- **`AppSettingValue`** moves to `@EmbeddedId (tenant_id, setting_key)`; `AppSettingsService.get`
  keys the cache `TenantContext.currentTenant() + ':' + setting.name()` (a constant prefix this
  rung — deliberately landed now so the enforcement rung's cache work is already half done).
- Entity `@ManyToOne` mappings gain their `tenant_id` columns via `@JoinColumn` updates where
  composite FKs demand it; everything else is untouched.

---

## 9. Privacy

Two new tables hold personal data, so the house rule applies — all three places:

1. **Data table** in `PRIVACY_AND_DATA_PROTECTION.md`: `memberships` (which groups a person
   belongs to — associational data; retention: life of the membership),
   `membership_roles` (what they may do there).
2. **`PrivacyService`**: export gains a `memberships` section (group name, roles, joined date —
   per group); erasure deletes membership rows with the account (CASCADE) exactly as
   `user_roles` behaves today. **The leave-one-group vs erase-platform fork is explicitly NOT
   this spec** — it needs enforcement's tenant scoping to exist first and is the whole subject of
   `TENANT-PRIVACY-PLAN.md`. Until then the erase path remains all-or-nothing, which is exactly
   what it is today, and onboarding (the first moment two memberships can exist) depends on that
   plan, not this one.
3. **`PrivacyServiceTest`**: export includes memberships; erasure removes them with the account.

The controller/processor split the privacy doc books for Phase 5 is likewise
`TENANT-PRIVACY-PLAN.md` material.

---

## 10. Test plan

| Area | Cases |
|------|-------|
| Migration chain | `MigrationSchemaValidationIT` green on V1→V27; backfill leaves org #1 with every user membered, roles identical to `user_roles`, every row `tenant_id = 1` |
| Constraints (integrationTest ONLY — H2 create-drop cannot see SQL constraints) | Composite FKs reject a cross-tenant `invited_by`/`paid_by`/`captain`/`vote`/`confirmation` insert; `uq_players_tenant_user` allows same account in two orgs, rejects twice in one; `ux_seasons_tenant_single_current` allows two orgs a current season each, rejects a second in one; app_settings composite PK |
| Auth equivalence | Membership-backed `UserDetailsServiceImpl` yields identical authorities for every V23-backfilled fixture, including zero-role accounts |
| Dark-launch acceptance | **The entire pre-existing suite passes unmodified** (bar mechanical fixture arity). 938+ unit tests, `GuestLifecycleTest`, e2e |
| Repository renames | Grep gate: zero call sites of `findByCurrentTrue`/`findByUserId(` (player) post-merge |

> **Pre-work (blocking, cheap): wire `integrationTest` into CI.** `GuestIsolationIT` has never
> executed anywhere; from this spec on, the entire tenancy safety story lives in that tier.
> Landing V22–V27 with the constraint tests unrun would be theatre.

---

## 11. Order of work

1. **Pre-work:** `integrationTest` into CI (and run `GuestIsolationIT` for the first time);
   reconcile the stale USERS.md/login.md registration drift so onboarding doesn't build on it.
2. Migrations V22–V27 + entities + `TenantContext` (constant) + repository renames, one PR.
3. `MigrationSchemaValidationIT` + the new constraint cases in a `TenancySchemaIT`.
4. **Deploy gate:** database backup; `./gradlew integrationTest` green in CI; then deploy (the
   deploy is the migration).
5. Soak one release; V28 (`DROP TABLE user_roles`) rides with the enforcement rung.

## 12. Breaking changes

- [x] **None on the wire, none in behaviour.** That is the definition of this rung: schema and
      code change together, the app answers every request as before, and the only observable
      difference is six new rows in `flyway_schema_history`.
- [ ] **One operational change, stated plainly:** rollback after this deploy requires the V28
      deferral (old code still reads `user_roles`, which V23 froze but kept) plus the pre-deploy
      backup. Roll forward is the plan; the backup is the parachute.
