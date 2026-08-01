# Group Onboarding (Signup, Invites, Picker) — Technical Specification

**Date:** 2026-08-01
**Status:** DRAFT (2026-08-01) — Phase 5a-4, nothing built. **This is the visibility flip.**
**Priority:** HIGH
**Estimated Effort:** L (≈3 days backend, ≈4–5 days frontend)
**Depends on:** `TENANT-PRIVACY-PLAN.md` — a **hard gate**: the first moment a second membership exists, the old erase path destroys cross-group data. Also transitively on enforcement + schema.
**Depended on by:** `GROUP-BILLING-PLAN.md`
**Contract:** `docs/api/TENANCY-API-CONTRACT.md` (extended) + `docs/api/GROUP-INVITES-API-CONTRACT.md` (new), same backend commit

> **Owner gate, restated from the roadmap:** the roadmap gates Phase 5 on "a real engagement
> signal from push", which has never been formally evidenced. The three rungs before this one are
> pure hardening, valuable regardless. **This rung is the product launch** — the owner signs off
> here, not at 5a-1.

---

## 1. Requirement Summary

Everything before this rung is invisible. This rung makes it a product: an account can **create a
group** (gated by an operator-issued creation code), **join a group** via an invite link, hold
several memberships, and **switch** between them — one app, one URL, an active group per session,
per the owner's picker decision.

It also closes two loops this chain owns: the guest plan's deferred **invite links** land here,
and **guest → member promotion** completes its final step (promote → register → link → *member*,
where the last arrow is now "insert a membership row" — exactly what the schema plan defined
membership to make cheap).

---

## 2. Scope

| In | Out |
|----|-----|
| Create-group flow (creation-code-gated) + founder bootstrap | Open self-serve creation (billing-era decision — codes become promo/trial codes) |
| `group_invites` (V29): links to join a group | Guest self-serve accounts beyond the existing promote→register→link path |
| Group picker (AuthGuard gate 3) + Navbar switcher | Billing wall (AuthGuard gate 4, `GROUP-BILLING-PLAN.md`) |
| `X-Group-Id` injection; group-prefixed query keys; cache erase on switch | Subdomains / path routing (owner decision: picker) |
| Membership management UI (UsersPage → members of this group) | `isCore` tiering (**still** deliberately untouched) |
| Branding sweep + org-#1 founder rename | Per-group theming/logos (post-5b if ever) |
| `MAX_GUESTS_PER_INVITER` → per-tenant AppSetting | Push deep-link group targeting (limitation accepted, §9) |

---

## 3. Model decision: creation is code-gated at launch — not open

**Chosen:** `POST /api/groups` requires a single-use **creation code** issued by the platform
operator (platform-admin surface). Registration itself stays open and unchanged — an account is
free; a *group* costs a code. When billing lands, codes become promo/trial codes rather than dying.

**Rejected: fully open creation.** No billing, no abuse controls, an unbounded free-rider window,
and the roadmap's engagement gate unmet — a code is one column and keeps the flip reversible.
(Owner confirmed this choice explicitly.)

The founder flow: redeem code → name the group → the service creates the organization, an ACTIVE
membership with **ADMIN + MANAGER + ORGANIZER** (the three flat grants — a founder wears every
hat until they delegate; still no hierarchy), and **seeds 'Season 1' as current** — replicating
V1's seed, without which four season-resolving services 500. Org #1's placeholder name gets the
same rename treatment: first login by an org-#1 ADMIN after this ships is offered the rename step.

## 4. Model decision: invites are server-issued single-purpose tokens — not shareable role-bearing URLs alone

**Chosen (V29):**

```sql
CREATE TABLE group_invites (
    id            BIGSERIAL   PRIMARY KEY,
    tenant_id     BIGINT      NOT NULL REFERENCES organizations(id),
    token         VARCHAR(64) NOT NULL UNIQUE,        -- random, unguessable
    roles         VARCHAR(60) NOT NULL DEFAULT '',    -- grants on join; '' = plain member
    invited_by    BIGINT      NOT NULL,
    expires_at    TIMESTAMPTZ NOT NULL,
    used_at       TIMESTAMPTZ,
    used_by       BIGINT,
    FOREIGN KEY (tenant_id, invited_by) REFERENCES memberships (tenant_id, id)
);
```

`POST /api/groups/{id}/invites` (group ADMIN) → link like `/join/<token>`. Redeeming
(`POST /api/invites/{token}/accept`, authenticated) creates the membership with the listed roles;
single-use, expiring, revocable. An unauthenticated visitor hitting `/join/<token>` is walked
through register-then-accept. **Rejected:** multi-use open links as the only mechanism —
uncontrolled membership in a product that will meter by group; multi-use *capped* links are a
cheap later additive.

This is where `GUEST-PLAYERS-PLAN.md` §2's deferred "invite links" land. The complete guest
arc becomes: guest plays (no account, no membership) → manager promotes (`is_guest` flips; still
no membership) → manager sends an invite → person registers/accepts (membership row) → links to
their player row. Every V21 constraint (`chk_guest_has_no_account`, promote-before-link) is
untouched; the only new step is the last, and it is one row.

## 5. Backend API summary

| Method | Path | Auth |
|---|---|---|
| `POST` | `/api/groups` | authenticated + valid creation code |
| `PATCH` | `/api/groups/{id}` (rename) | group ADMIN |
| `GET` | `/api/me/memberships` | authenticated — the picker's data source |
| `POST` | `/api/groups/{id}/invites` · `DELETE .../invites/{inviteId}` | group ADMIN |
| `GET` | `/api/invites/{token}` (preview: group name, roles) | public |
| `POST` | `/api/invites/{token}/accept` | authenticated |
| Login response | gains `memberships: [{groupId, groupName, roles}]` | additive; zod schema fails open, so old frontends survive |

`MAX_GUESTS_PER_INVITER` moves from a compile-time constant to a per-tenant `AppSetting`
(`guests.max.per.inviter`, default 2, range 0–10) — the first group-policy knob that groups will
genuinely disagree about, and the app_settings design absorbs it with zero seeding.

---

## 6. Frontend — the group dimension arrives

**The two choke-point files first** (everything else hangs off them):

- `types/auth.ts`: `AuthUser` gains `memberships`; new `activeGroupId` in the persisted slice
  (with a `migrateStoredUser`-style shape migration — the established pattern). The login
  response schema addition is additive and fails open.
- `lib/roles.ts`: `hasRole`/`hasAnyRole` resolve against **the active membership's** roles; ~30
  call sites stay mechanically identical. `RoleChips` shows the active group's grants.

**The safety pair — both, not either** (BR-O3): a `setActiveGroup` store action that (a) calls
`eraseQueryCache()` unconditionally — the same erase `setAuth`/`clearAuth` perform, closing the
localStorage-rehydration leak — and (b) every query key gains the active group via one
`groupKey(...)` factory (`['g', groupId, ...rest]`), so even the two-tabs-two-groups localStorage
sharing degrades to cache misses, never wrong data. Platform-global keys (`['version']`,
`['push','public-key']`, `['users','me']`) are the named exceptions.

**Request plumbing:** `apiFetch` injects `X-Group-Id` from the store (the one choke point), plus
the two documented bypass sites: the privacy blob download adds the header; draft SSE needs
**nothing** — the enforcement plan made SSE resource-derived.

**AuthGuard gate 3:** authenticated but no active group → membership count 0 ⇒ onboarding
(create-with-code or "ask for an invite"); 1 ⇒ auto-select silently (the dark-compat path — every
existing user has exactly one); >1 ⇒ picker. Gate 4 (billing wall) is the next plan's.

**Navbar:** the brand slot becomes *group name + switcher* (memberships from the login payload);
platform brand recedes to login/footer. Switching = `setActiveGroup` → cache erase → navigate to
dashboard.

**Surface splits:** SystemSettings separates per-group competition rules (the AppSettings panel —
already per-tenant server-side, UI absorbs it) from platform facts (VAPID health, version — shown
only to platform admins via a new `platformAdmin` flag on the login payload); UsersPage becomes
"Members of {group}" backed by membership endpoints, editing per-membership roles; invite
management joins it.

**Branding sweep:** `appName` usages split into platform-name (login, PWA manifest, install
hint) vs `{{groupName}}` interpolation (~10 strings) — ×3 locales, spliced not re-serialized
(the house lesson).

## 7. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-O1 | A group exists only by creation code; the founder gets all three grants + a seeded current season | Four services assume a current season — seeding is load-bearing, tested |
| BR-O2 | Invites are single-use, expiring, revocable; roles-on-join are the invite's, not the inviter's | An ADMIN can mint a MANAGER invite; a stolen token grants once, then never |
| BR-O3 | Group switch = cache erase **and** group-prefixed keys | Belt and braces — the frontend inventory showed why each alone fails |
| BR-O4 | With one membership, everything auto-selects and the UI shows no group chrome beyond the name | Existing users wake up in the same app |
| BR-O5 | Guest promotion's final step is one membership row | The promise, kept end to end |

## 8. Test plan

| Area | Cases |
|------|-------|
| Backend | Create-group: code single-use/expired/invalid; founder grants; season seeded (then plan/match creation works immediately); rename ADMIN-only. Invites: accept creates membership+roles; reuse 409; expiry 410; revoke; cross-tenant invite admin → 404. Memberships list. Guest arc end-to-end: promote → register → accept invite → link |
| `TenantIsolationIT` extension | Two *real* groups created through the API (not seeded) can't see each other — the whole enforcement suite re-run against API-born tenants |
| Frontend unit | groupKey factory; setActiveGroup erases cache; roles resolve per active group; AuthGuard 0/1/n branches |
| Frontend e2e (one coordinated commit) | `seedSession` fixture gains memberships+activeGroupId; switcher e2e asserts **no stale cross-group data after switch**; all 20 visual baselines regenerated (win32) — budgeted, not discovered |
| i18n | Key parity ×3 locales for every new namespace |

## 9. Privacy & known limitations

- New personal-data table (`group_invites` — inviter, used_by): data-table row + export (invites
  I issued/accepted) + erase handling + test, per house rule.
- Push: subscriptions stay per-account (enforcement plan D9); a notification from group B while
  group A is active deep-links into the active group — **accepted 5a limitation**, payload gains
  an advisory `groupId` for the future; `tag: category` collapse across groups noted.
- Two-tab localStorage cache sharing: defused by prefixed keys (misses, not leaks); residual
  same-group staleness is pre-existing and documented.

## 10. Order of work

1. Backend: groups + creation codes + founder bootstrap + season seed; memberships endpoint; login payload.
2. Backend: invites (V29) + guest-arc completion; `MAX_GUESTS_PER_INVITER` setting.
3. Contracts.
4. Frontend: auth types + roles.ts + groupKey factory + setActiveGroup (the safety pair) + apiFetch header.
5. Frontend: AuthGuard gate 3 + picker/onboarding + switcher + surface splits + branding ×3 locales.
6. The coordinated e2e/visual commit.
7. **Owner sign-off checkpoint** → issue first creation codes. The flip is: codes exist.

## 11. Breaking changes

- [x] **None for existing users**: one membership auto-selects; same screens, plus their group's name.
- [ ] **Deliberate change:** `users`/user-management semantics move from deployment-wide to
      per-group; platform operations concentrate behind the platform-admin grant. Stated in the
      contract, invisible until a second group exists.
