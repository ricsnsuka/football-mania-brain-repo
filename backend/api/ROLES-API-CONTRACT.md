# Composable Roles — API Contract

**Date:** 2026-07-31
**Version:** v1.0.0
**Status:** APPROVED — backend complete (migration, entity, authorisation, DTOs, tests)
**Supersedes:** the single-valued `role` field on every user-facing DTO

---

## Scope

`AppUser.role` — one of `BASIC_USER` / `MASTER_USER` / `ADMIN_USER` — is replaced by a **set** of
independent capability grants. One migration (V18), one new table, no new endpoints.

| Role | Grants |
|------|--------|
| `ORGANIZER` | See all balances, record payments, set fees, void and waive charges *(the fee ledger; inert until it ships)* |
| `MANAGER` | Create plans and matches, manage the roster, generate teams, record results |
| `ADMIN` | System settings, user administration, rating recalculation, GDPR actions, purge |

Two things that are **not** roles:

- **Authenticated** — sees matches, plans, rankings, the league table, own profile.
- **Is a player** — a linked `Player` row. Confirms attendance, votes MOTM, is drafted.

---

## Design decisions

### The roles are flat, and `ADMIN` does not imply `MANAGER`

A player-organiser is not "below" a manager, so the roles cannot be ordered. Making them
independent removes every implication rule, which is where role systems usually rot.

The consequence is real and deliberate: **an account holding only `ADMIN` is refused every
manager-gated endpoint.** `FlatRoleSemanticsTest` exists solely to pin this — every other test
authenticates as an administrator holding both, so all of them would keep passing if somebody
widened a manager endpoint back to `hasRole('ADMIN') or hasRole('MANAGER')`.

### There is no `PLAYER` role

`players.user_id` is UNIQUE and `Player.user` is `@OneToOne`, so "is a player" is already a fact in
the schema. A role asserting the same thing would be a second source of truth, free to disagree
with the first, and unguardable by any constraint because the two live in different tables — a
role with no linked row means the app believes you play but has nowhere to record your goals.

"Someone who does not play" needs no role to express it. It is a user with no linked player.

### `BASIC_USER` becomes the empty set

An empty authority collection still satisfies `isAuthenticated()`, which is all `BASIC_USER` ever
meant. Registration grants nothing; an administrator grants roles afterwards.

### The backfill grants administrators `MANAGER`

Every existing `ADMIN_USER` becomes `{ADMIN, MANAGER, ORGANIZER}`; every `MASTER_USER` becomes
`{MANAGER}`; every `BASIC_USER` gets no rows.

Without the `MANAGER` grant, every administrator would silently lose the ability to create a match
on deploy. That single line of SQL is also what allows seventeen compound `@PreAuthorize`
expressions to collapse to `hasRole('MANAGER')` rather than regress.

`ORGANIZER` is **not** granted to managers — that would hand out a new capability under cover of a
refactor.

### `ORGANIZER` sees contact details

`PlayerPiiPolicy` now treats `ROLE_ORGANIZER` as privileged alongside `ROLE_ADMIN` and
`ROLE_MANAGER`. Contact details include the phone number, and a phone number is how an MB WAY
payment is addressed — an organiser who cannot see it cannot do the job.

### An administrator may now be linked to a player

The rule forbidding it is **removed** from all three enforcement sites. Under composable roles
`ADMIN` is a capability grant, not an identity, so "holds ADMIN" says nothing about whether the
person kicks a ball. It never functioned as a separation-of-duties control either — managers
already edit scoresheets, name the MVP *and* play. Its only reliable effect was forcing whoever
runs the deployment to keep a second account in order to be in the squad.

> **Unchanged:** an account holding `ADMIN` still **cannot erase itself**
> (`PrivacyService.eraseByUser`). Unrelated rule — that one is about not deleting the credentials
> that administer the deployment, possibly the last set.

---

## Changed DTOs

| DTO | Was | Now |
|-----|-----|-----|
| `LoginResponseDTO` | `String role` | `String role` *(deprecated)* **+** `List<String> roles` |
| `UserDTO` | `String role` | `List<String> roles` |
| `UserCreateDTO` | `@NotNull String role` | `Set<String> roles` — optional, may be empty |
| `AdminUserUpdateDTO` | `String role` | `Set<String> roles` — replaces wholesale |
| `PersonalDataExportDTO.Account` | `String role` | `List<String> roles` |

Output lists are **sorted** (`ADMIN`, `MANAGER`, `ORGANIZER`). The entity holds a `HashSet`, whose
iteration order is not guaranteed; without sorting the array order would vary between responses for
the same user and both the frontend cache and the contract tests would flap.

> `PersonalDataExportDTO` has a **second, unrelated** field called `role` on its goal entries
> (`SCORER` / `ASSISTER`). That one is unchanged.

### `PATCH /api/users/{id}/role`

```json
{ "roles": ["MANAGER", "ORGANIZER"], "isActive": true }
```

`roles` **replaces** the user's grants rather than merging. There is no grant/revoke pair: sending
the intended end state means two administrators editing at once cannot interleave into a
combination neither of them chose.

An explicitly empty array strips every role. Only omitting the field means "leave alone".

| Status | Trigger |
|--------|---------|
| `400` | An entry outside `ORGANIZER` / `MANAGER` / `ADMIN` — a field-level validation error, not a parse failure |
| `403` | Caller does not hold `ADMIN` |
| `404` | No such user |

---

## Compatibility

**`LoginResponseDTO` emits both `role` and `roles` for one release.**

The frontend deploys from a separate repository. Dropping `role` outright would leave an
already-deployed client reading it as `undefined` and treating everybody — administrators
included — as unprivileged until the frontend caught up.

`role` carries the highest-precedence legacy name: `ADMIN` → `"ADMIN_USER"`, else `MANAGER` →
`"MASTER_USER"`, else `"BASIC_USER"`. That maps exactly onto the old vocabulary, so an un-updated
client behaves identically. Delete the field and `Role.legacyNameFor` once the frontend reads
`roles`.

**Tokens need no handling at all.** `JwtAuthenticationFilter` puts only the username in the JWT and
re-loads authorities from the database on every request. Nobody is logged out, no token is
invalidated, and a role change takes effect on the next request rather than the next login.

---

## Frontend migration notes

1. **Read `roles`, not `role`.** `role` is deprecated and will be removed.
2. **Introduce a `hasRole(user, 'ADMIN')` helper before touching call sites.** Sixty files each
   writing `user.roles.includes(...)` is how the check drifts.
3. **`ADMIN` no longer implies `MANAGER`.** A screen gated on "can manage matches" must test
   `MANAGER`, not `ADMIN`. Existing administrators hold both, so this is invisible until somebody
   is granted `ADMIN` alone.
4. **An empty array is a valid user**, not a loading state or an error. Render it as "Member".
5. **The role editor becomes multi-select**, and it submits the complete intended set.
6. **`PersonalDataExportDTO.account.role` is now `account.roles`** — an array.

---

## Breaking changes

- [x] **`UserDTO.role` → `roles`** — admin surfaces and `GET /api/users/me`
- [x] **`UserCreateDTO.role` → `roles`** — admin user creation
- [x] **`AdminUserUpdateDTO.role` → `roles`** — the role editor
- [x] **`PersonalDataExportDTO.Account.role` → `roles`** — GDPR export
- [x] **`users.role` column dropped** — no rollback but a database restore
- [x] **An `ADMIN` account may now be linked to a player** — deliberate
- [ ] `LoginResponseDTO.role` — **not yet breaking**, shimmed for one release

Full rationale and the migration runbook: `docs/plans/ROLE-MODEL-MIGRATION-PLAN.md`.
