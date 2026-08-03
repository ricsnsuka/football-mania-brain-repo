# Composable Role Model — Technical Specification

**Date:** 2026-07-31
**Status:** **COMPLETE** — backend (853 tests) and frontend (428 unit, 20 visual) both green.
**Priority:** HIGH — unblocks `MATCH-FEE-LEDGER-PLAN.md`
**Estimated Effort:** L — delivered
**Contract:** `docs/api/ROLES-API-CONTRACT.md` · frontend notes: `docs/features/roles.md` (frontend repo)

> **Not yet deployed.** V18 has never run: Flyway is disabled in tests, so the backfill and the
> CHECK constraint execute for the first time against a real database, and `DROP COLUMN users.role`
> has no rollback but a restore. Verify on staging with the queries in §5 before production.

---

## 1. Requirement Summary

Replace the single-valued `AppUser.role` with a **set** of independent capability grants, so that
running the matches, handling the money, and administering the system can be held by different
people in any combination.

The immediate driver is the fee ledger: whoever receives the payments need not be whoever creates
the matches, and today's model cannot say one without the other.

**This ships as a pure refactor.** No user gains or loses a capability. New behaviour arrives with
the ledger, in a separate commit.

---

## 2. The role set

Three roles. Flat, independent, additive — **not a hierarchy**.

| Role | Grants |
|------|--------|
| `ORGANIZER` | See all balances, record payments, set fees, void and waive charges |
| `MANAGER` | Create plans and matches, manage the roster, generate teams, record results |
| `GROUP_ADMIN` | System settings, user administration, rating recalculation, GDPR actions, purge |

Everything else is derived from two things that are not roles:

| Fact | Source | Grants |
|------|--------|--------|
| Authenticated | A valid token | See matches, plans, rankings, league table, own profile |
| **Is a player** | A linked `Player` row (`players.user_id`) | Confirm attendance, vote MOTM, be drafted, see **own** balance |

### Why there is no `PLAYER` role

"Is a player" is already a fact in the schema — `players.user_id` is UNIQUE and `Player.user` is
`@OneToOne`, established through `linkMe`. A `PLAYER` role would be a second source of truth for
the same fact, free to disagree with the first:

- role but no link → the app believes you play, with nowhere to record your goals
- link but no role → you are in the squad but cannot confirm attendance

Both fail silently, and no constraint can catch either because they live in different tables.

*"A manager who isn't interested in playing football"* needs no role to express it. It is a user
with no linked player — exactly what an administrator is today.

### Why there is no hierarchy

A player-organiser is not "below" a manager, so the roles cannot be ordered. Making them flat and
independent removes the whole category of implication bugs ("does GROUP_ADMIN imply MANAGER?"), at the
cost of one thing that must be handled deliberately: **`GROUP_ADMIN` does not imply `MANAGER`.** See §4.

### `BASIC_USER` disappears

It is replaced by the **empty set**. A user with no roles is authenticated and nothing more, which
is what `BASIC_USER` always meant. Spring Security is content with an empty authority collection;
`isAuthenticated()` still passes.

The UI renders an empty set as "Member".

---

## 3. Naming

`_USER` is dropped and `MASTER` becomes `MANAGER`. Authorities become `ROLE_ADMIN`, `ROLE_MANAGER`,
`ROLE_ORGANIZER`.

Every authorisation site is being rewritten anyway, so this rename is close to free now and
expensive at any later date. "Master" never named a capability.

---

## 4. The backfill is the whole risk

Today `ADMIN_USER` passes every `hasRole('ADMIN_USER') or hasRole('MASTER_USER')` check — twelve
sites covering match creation, plan management and draft confirmation. Under flat roles it no
longer does.

**So the backfill must grant administrators `MANAGER` as well, or every administrator silently
loses the ability to create a match the moment this deploys.**

| Old | New |
|-----|-----|
| `ADMIN_USER` | `{GROUP_ADMIN, MANAGER, ORGANIZER}` |
| `MASTER_USER` | `{MANAGER}` |
| `BASIC_USER` | `{}` — no rows |

`ORGANIZER` is included for administrators because it is inert until the ledger ships and it means
the ledger arrives with somebody able to use it. It is **not** granted to managers: that would be
handing out a new capability under cover of a refactor.

---

## 5. Schema — `V18__composable_roles.sql`

> **The ledger migration moves to V19.** `MATCH-FEE-LEDGER-PLAN.md` §6 has been updated.

```sql
CREATE TABLE user_roles (
    user_id BIGINT      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role    VARCHAR(20) NOT NULL,
    PRIMARY KEY (user_id, role),
    CONSTRAINT user_roles_role_check CHECK (role IN ('ORGANIZER', 'MANAGER', 'GROUP_ADMIN'))
);

INSERT INTO user_roles (user_id, role)
SELECT id, r.role
FROM users
CROSS JOIN LATERAL (VALUES ('GROUP_ADMIN'), ('MANAGER'), ('ORGANIZER')) AS r(role)
WHERE users.role = 'ADMIN_USER';

INSERT INTO user_roles (user_id, role)
SELECT id, 'MANAGER' FROM users WHERE users.role = 'MASTER_USER';

-- BASIC_USER deliberately receives no rows.

ALTER TABLE users DROP COLUMN role;
```

The primary key doubles as the uniqueness guarantee, so a role cannot be granted twice and the
grant operation is naturally idempotent.

**The column is dropped in the same migration, not kept for a release.** A retained `users.role`
would stop being written the moment the entity changes, so within a day it would claim
`MASTER_USER` for somebody who is now organiser-only — and being a plausible lie, somebody would
eventually trust it.

> Flyway is disabled in tests (`ddl-auto: create-drop`), so **neither the CHECK constraint nor the
> backfill is exercised by the suite**. The backfill runs exactly once, against real data, and is
> not reversible. Run it on `staging` first and compare counts:
>
> ```sql
> SELECT role, count(*) FROM users GROUP BY role;          -- before
> SELECT role, count(*) FROM user_roles GROUP BY role;     -- after
> ```
>
> Expect `GROUP_ADMIN`, `MANAGER` and `ORGANIZER` each ≥ the old `ADMIN_USER` count, and
> `MANAGER` = `ADMIN_USER` + `MASTER_USER`.

---

## 6. Entity

```java
public enum Role { ORGANIZER, MANAGER, GROUP_ADMIN }

@ElementCollection(fetch = FetchType.EAGER)
@CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
@Column(name = "role", nullable = false, length = 20)
@Enumerated(EnumType.STRING)
@Builder.Default
private Set<Role> roles = new HashSet<>();

public boolean hasRole(Role role) {
    return roles.contains(role);
}
```

`EAGER` on a collection of at most three elements: `UserDetailsServiceImpl` reads it on **every
request**, and `UserService.listUsers` would otherwise issue one query per user.

`HashSet`, not `EnumSet` — Hibernate replaces the instance with its own `PersistentSet` on load, so
an `EnumSet` default survives only until the first read and creates a type that varies by code path.

---

## 7. Authorisation sites

`UserDetailsServiceImpl:41-42` becomes:

```java
var authorities = user.getRoles().stream()
        .map(r -> new SimpleGrantedAuthority("ROLE_" + r.name()))
        .toList();
```

### Expression mapping

| Current | Becomes | Sites |
|---------|---------|-------|
| `hasRole('ADMIN_USER') or hasRole('MASTER_USER')` | `hasRole('MANAGER')` | 17 |
| `hasRole('ADMIN_USER')` | `hasRole('GROUP_ADMIN')` | 20 |
| `hasRole('ADMIN_USER') or #id == authentication.principal.id` | `hasRole('GROUP_ADMIN') or #id == …` | 3 |
| `isAuthenticated()` | unchanged | 26 |

**66 sites in total, 40 of them changed** — the counts above are the measured inventory, not the
estimate this plan originally carried (12/11/12), which was taken from a truncated search.

The compound expressions collapse to a single term — which is only correct **because** the backfill
grants administrators `MANAGER` (§4). That one line of SQL is what makes twelve authorisation
sites simplify rather than regress.

Affected: `MatchController`, `MatchPlanController`, `DraftSessionController`, `AdminController`,
`PlayerController`, `UserController`, `PrivacyController`.

### `PlayerPiiPolicy`

```java
private static final Set<String> PRIVILEGED_ROLES =
        Set.of("ROLE_ADMIN", "ROLE_MANAGER", "ROLE_ORGANIZER");
```

`ORGANIZER` is **added**, and this is not incidental: contact details include the phone number, and
a phone number is how an MB WAY payment is addressed. An organiser who cannot see it cannot do the
job.

---

## 8. The GROUP_ADMIN-cannot-be-a-player rule is removed

> **Confirm this reading.** It reverses an invariant documented in three places. One line each to
> restore if it is not what was wanted.

Deleted from all three enforcement sites:

| Site | What it does today |
|------|--------------------|
| `PlayerService:86` | Rejects linking on player **create** |
| `PlayerService:138` | Rejects linking on player **update** |
| `PlayerService:201-206` | Rejects `linkMe` when the caller holds `ROLE_ADMIN_USER` |

Reasoning: under composable roles `GROUP_ADMIN` is a capability grant, not an identity, so "holds GROUP_ADMIN"
says nothing about whether the person kicks a ball. The rule reads as a separation-of-duties
control but never functioned as one — managers already edit scoresheets, name the MVP *and* play,
so the conflict of interest it appears to prevent has always been present one role down. Its only
reliable effect was forcing whoever runs the system to keep a second account in order to play,
which is the exact friction this change exists to remove.

### What is **not** removed

`PrivacyService:364` guards **self-erasure** — an administrator cannot erase their own account,
another administrator must action it. Unrelated to player linking, and it stays:

```java
if (user.hasRole(Role.GROUP_ADMIN)) { ... }
```

A find-and-replace across `AppUser.Role.ADMIN_USER` will otherwise take a real safety control with
it. Four sites match; three are deleted and this one is rewritten.

### Documentation to correct

- `Player.java` class javadoc — the invariant list
- `PLAYER-LINK-ME-PLAN.md` — BR-1
- `docs/api/PLAYER-LINK-ME-API-CONTRACT.md` — the 403 case
- `docs/api/player-API-CONTRACT.md`
- Tests asserting the 403 (they become tests that the link **succeeds**)

---

## 9. DTOs

| DTO | Change |
|-----|--------|
| `LoginResponseDTO` | `String role` → `Set<String> roles` **+ compatibility shim, §10** |
| `UserDTO` | `String role` → `Set<String> roles` |
| `UserCreateDTO` | `@NotNull String role` → `Set<String> roles`, may be empty |
| `AdminUserUpdateDTO` | `@Pattern` on a scalar → element-level validation |
| `UserRegisterDTO` | Javadoc: role forced to `BASIC_USER` → **no roles granted** |
| `PersonalDataExportDTO` | Line 68 `role` → `roles` |

```java
public record AdminUserUpdateDTO(
        Set<@Pattern(regexp = "ORGANIZER|MANAGER|GROUP_ADMIN",
                     message = "must be one of ORGANIZER, MANAGER, GROUP_ADMIN") String> roles,
        Boolean isActive
) {}
```

Container-element validation, so each entry is checked individually and the error path stays a
field-level validation failure rather than a parse error — preserving the reason the `@Pattern` was
added in the first place.

> ⚠️ **`PersonalDataExportDTO` has two unrelated fields called `role`.** Line 68 is the user's
> role. Line 115 is a goal involvement — `SCORER` or `ASSISTER`. Only the first changes. This is
> the single most likely mis-edit in the whole migration.

`UserService:94` (`.role(AppUser.Role.BASIC_USER)` on registration) becomes no grant at all.

---

## 10. Deploy compatibility

The two repos deploy independently, so `LoginResponseDTO` losing `role` would black out every
role-gated control in an already-deployed frontend — everyone would look like a basic user until
the frontend caught up.

**Emit both for one release:**

```java
// Deprecated. Highest-precedence single role, for frontends predating composable roles.
// Delete once the frontend ships `roles`.
String role,          // "ADMIN_USER" | "MASTER_USER" | "BASIC_USER"
Set<String> roles     // ["GROUP_ADMIN", "MANAGER", "ORGANIZER"]
```

Precedence: `GROUP_ADMIN` → `ADMIN_USER`, else `MANAGER` → `MASTER_USER`, else `BASIC_USER`. That maps
exactly onto the old vocabulary, so an un-updated frontend behaves identically.

**The tokens themselves need no handling at all.** `JwtAuthenticationFilter:43-44` puts only the
username in the JWT and re-loads authorities from the database on every request. Nobody is logged
out, no token is invalidated, and a role change takes effect on the next request rather than the
next login. For a security refactor this is as forgiving as it gets.

---

## 11. Frontend

Separate repo. 237 matches across 66 files, though locale strings and CSS `[role=]` selectors
inflate that — the real work is concentrated:

| File | Change |
|------|--------|
| `src/types/auth.ts` | `role: string` → `roles: string[]` |
| `src/store/appStore.ts` | User shape |
| `src/lib/roles.ts` *(new)* | `hasRole(user, 'GROUP_ADMIN')` helper |
| `src/components/auth/AuthGuard.tsx` | Accept a required-role set |
| `src/features/users/EditUserModal.tsx` | Single `<select>` → checkbox group |
| `src/features/users/UsersPage.tsx` | Render role chips; empty set as "Member" |

Then every `user?.role === 'ADMIN_USER'` becomes `hasRole(user, 'GROUP_ADMIN')`. **Introduce the helper
first** — sixty files each writing `user.roles.includes(...)` is how the check drifts.

Sites that branch on role today: `Navbar`, `AccountMenu`, `SettingsPage`, `SystemSettings`,
`DashboardOverview`, `MatchesPage`, `MatchModal`, `MatchCard`, `RecalculateMatchesPanel`,
`MatchPlansPage`, `MatchPlanCard`, `MatchPlanDetailModal`, `PlayersPage`, `PlayerModal`,
`TeamGenerationPage`, `CaptainPickBoard`, `RankingsPage`, `draft-sessions/page`, `users/page`.

Test fixtures mocking `{ role: 'ADMIN_USER' }` all need updating, plus `e2e/fixtures.ts`
(`seedSession(page, theme, authenticated, role)` → a role array) and the visual baselines, since
the navbar contents change per role.

> i18n: `roles.organizer` / `roles.manager` / `roles.admin` / `roles.member` in en, pt and es.
> **Splice, do not `JSON.stringify`** — pt and es use compact one-line objects and a rewrite
> produces an 800-line diff.

---

## 12. Test plan

| Area | Cases |
|------|-------|
| `UserDetailsServiceImpl` | Empty set → no authorities, still authenticated; three roles → three authorities |
| Authorisation matrix | For each of `{}`, `{MANAGER}`, `{ORGANIZER}`, `{GROUP_ADMIN}`, `{GROUP_ADMIN,MANAGER}`: one endpoint per tier |
| Regression guard | `{GROUP_ADMIN}` alone is **rejected** by a manager endpoint — proves flatness is real and not accidentally re-implied |
| Linking | An `GROUP_ADMIN` user can now be linked to a player, and can `linkMe` |
| Self-erasure | An `GROUP_ADMIN` still cannot erase their own account |
| DTO validation | `roles: ["WIZARD"]` → 400 field error, not a parse failure |
| Compatibility | `{GROUP_ADMIN,MANAGER}` serialises `role: "ADMIN_USER"` |

> **CI invariant:** `build.gradle` asserts
> `testControllers + testServices + testSecurity + testApplication == test`. New security tests
> must land in a counted package or the build fails on the count.

---

## 13. Order of work

1. `V18__composable_roles.sql` — table, backfill, drop column
2. `Role` enum, `AppUser.roles`, `hasRole`
3. `UserDetailsServiceImpl` → authority list
4. All 23 `@PreAuthorize` expressions + `PlayerPiiPolicy`
5. Remove the three player-linking checks; rewrite the self-erasure check
6. DTOs + the compatibility shim
7. Backend tests green
8. Frontend: types, store, `hasRole`, then the call sites
9. Docs: this file to APPROVED, plus the five contract docs in §8

Steps 1–7 are independently shippable — the shim means the current frontend keeps working
untouched.

### What the frontend actually needed beyond this plan

Two things this plan did not anticipate, both found during implementation:

- **Sessions outlive deploys.** `appStore` persists `AuthUser` to `sessionStorage`, so everyone
  signed in at deploy time carries `{"role": "ADMIN_USER"}` with no `roles` key. Every gate would
  read them as unprivileged — an administrator watching their own controls disappear — until they
  happened to log out. `migrateStoredUser` derives the set on hydration using the same
  `rolesFromLegacy` mapping as the login fallback.
- **`loginResponseSchema` had to make both role fields optional.** It validates at the `apiFetch`
  boundary, so requiring `roles` would turn "frontend deployed before backend" into a total login
  failure rather than a degraded one.

One latent inconsistency surfaced and was fixed rather than preserved: the match-plans create
button was gated on `MASTER_USER` alone, hiding it from administrators even though
`POST /api/match-plans` has always accepted them. It now gates on `MANAGER`, which is what the
server checks.

---

## 14. Breaking changes

- [x] **`LoginResponseDTO.role`** — mitigated by the shim in §10; genuinely breaking once removed
- [x] **`UserDTO.role`, `UserCreateDTO.role`, `AdminUserUpdateDTO.role`** — admin surfaces only
- [x] **`users.role` column dropped** — no rollback but a database restore
- [x] **GROUP_ADMIN may now be linked to a player** — deliberate, §8

Contract docs to update in the same commit, per repo convention: `player-API-CONTRACT.md`,
`PLAYER-LINK-ME-API-CONTRACT.md`, `ADMIN-API-CONTRACT.md`, `PRIVACY-API-CONTRACT.md`,
`API_REFERENCE.md`.
