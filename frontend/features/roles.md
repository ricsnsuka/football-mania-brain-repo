# Roles

A user holds a **set** of capability grants, not one role. The set can be empty.

| Role | Grants |
|------|--------|
| `ORGANIZER` | See all balances, record payments, set fees *(the fee ledger — backend not built yet)* |
| `MANAGER` | Create plans and matches, manage the roster, generate teams, record results |
| `GROUP_ADMIN` | System settings, user accounts, rating recalculation, GDPR actions, purge |

Two things that are **not** roles:

- **Authenticated** — matches, plans, rankings, the league table, your own profile.
- **Being a player** — a linked player record. Confirm attendance, vote MOTM, be drafted.

Holding nothing is the ordinary case for most of the group. It renders as **"Member"**.

---

## Always go through `hasRole`

```ts
import { hasRole, hasAnyRole } from '@/lib/roles';

const canCreate = hasRole(user, 'MANAGER');
const canSeeRoster = hasAnyRole(user, ['MANAGER', 'GROUP_ADMIN']);
```

Never write `user.roles.includes('GROUP_ADMIN')` at a call site. Nineteen components branch on roles; each
one reimplementing the null guard is how the check drifts out of agreement with the server.

`hasRole` is safe on `null`, `undefined` and a missing `roles` array — all of which occur before
hydration finishes.

## `GROUP_ADMIN` does not imply `MANAGER`

The roles are **flat**. An account holding only `GROUP_ADMIN` is refused every manager-gated control, on
the client and on the server alike.

This is invisible day to day, because V18's backfill granted every existing administrator
`{ORGANIZER, MANAGER, GROUP_ADMIN}` — so they hold both and everything works. It becomes visible the
moment somebody is granted `GROUP_ADMIN` alone. Gate on the role the endpoint actually checks:

```ts
// Wrong — the endpoint checks MANAGER, so this hides a control the user can actually use,
// or shows one whose every request will 403.
const canCreate = hasRole(user, 'GROUP_ADMIN');

// Right
const canCreate = hasRole(user, 'MANAGER');
```

`AuthGuard` takes one role or an array, and admits on **any** match:

```tsx
<AuthGuard requiredRole="GROUP_ADMIN">…</AuthGuard>
<AuthGuard requiredRole={['MANAGER', 'GROUP_ADMIN']}>…</AuthGuard>
```

## Rendering roles

`RoleChips` — one component, used by the users table, the account menu, the dashboard header and
profile settings. It renders "Member" for an empty set rather than nothing, because a blank cell
reads as missing data instead of as an answer.

```tsx
<RoleChips roles={user.roles} />                              // default chip styling
<RoleChips roles={user.roles} className="account-menu__role" /> // plain labels
```

Each chip gets a `--organizer` / `--manager` / `--admin` / `--member` modifier appended to the base
class, so a new role needs a CSS rule and nothing else.

## Editing roles

`EditUserModal` submits the **complete intended set**, not a diff. `PATCH /api/users/{id}/role`
replaces wholesale: `[]` strips every grant, and omitting `roles` leaves them alone. There is no
grant/revoke pair, so two administrators editing at once cannot interleave into a combination
neither of them chose.

---

## Two compatibility shims, both temporary

Delete these together once the backend has stopped sending `role` and every live session has turned
over.

**`rolesFromLegacy`** maps a pre-V18 role name to a set, mirroring the V18 backfill exactly:
`ADMIN_USER` → all three, `MASTER_USER` → `['MANAGER']`, anything else → `[]`.

1. **Session hydration** (`appStore.migrateStoredUser`). Sessions outlive deploys — anyone signed in
   when this shipped has `{"role": "ADMIN_USER"}` in `sessionStorage` and no `roles` key. Without
   the migration an administrator would watch their own controls vanish until they next logged out.
2. **Login response** (`useLogin`). Covers this frontend reaching production ahead of the backend,
   when `roles` is not being sent yet.

`loginResponseSchema` therefore marks **both** role fields optional. Requiring either would turn a
deploy-ordering mismatch between two independently released repositories into a total login failure.

---

## Files

| File | Role |
|------|------|
| `src/lib/roles.ts` | `hasRole`, `hasAnyRole`, `ALL_ROLES`, `rolesFromLegacy` |
| `src/types/auth.ts` | `UserRole`, `userRoleSchema`, `AuthUser.roles` |
| `src/components/ui/RoleChips.tsx` | Renders a grant set |
| `src/components/auth/AuthGuard.tsx` | Route guard |
| `src/store/appStore.ts` | `migrateStoredUser` on hydration |
| `src/features/users/EditUserModal.tsx` | The checkbox picker |

Backend contract: `docs/api/ROLES-API-CONTRACT.md` in the backend repository.
