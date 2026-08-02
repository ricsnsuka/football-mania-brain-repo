# Multi-Tenancy — How the Two Repos Meet

**Date:** 2026-08-01 · **Updated:** 2026-08-02 · **Status:** rungs 1–2 built, 3–5 specced
(`backend/plans/TENANCY-SCHEMA-PLAN.md` → `TENANCY-ENFORCEMENT-PLAN.md` →
`TENANT-PRIVACY-PLAN.md` → `GROUP-ONBOARDING-PLAN.md` → `GROUP-BILLING-PLAN.md`).
This doc owns what crosses the repo boundary; each plan owns its own internals; API contracts
stay in the backend repo's `docs/api/`, same commit as the code, as always.

---

## The one-paragraph model

One database, one deployment, many **groups** (`organizations` in the schema, `tenant_id` on
every owned row, composite FKs making cross-group references unrepresentable). **Accounts are
platform-level; membership is per-group; roles are per-membership and stay flat** — ADMIN of one
group says nothing about another group or the platform, whose operator holds a separate
platform-admin grant. A guest remains what the guest feature promised: a player row with no
membership row.

## The tenant-resolution contract

- Every authenticated API request carries **`X-Group-Id: <id>`**. The backend verifies an ACTIVE
  membership for (caller, group) **on every request** (it already reloads the user per request,
  so this is nearly free and revocation is instant — the reason a JWT tenant claim was rejected).
- **No header + exactly one membership → that membership.** This compatibility rule is what let
  the enforcement rung ship dark under the old frontend. No header + several → `400` naming the
  header.
- **Cross-group anything → `404`, never `403`.** A 403 is an existence oracle across tenants.
- **SSE is the exception**: `EventSource` cannot send headers, so draft-session endpoints accept
  no client-asserted tenant at all — the tenant comes from the resource row itself and membership
  in it is asserted. Nothing for the frontend to do; deliberately so.
- The frontend injects the header in exactly one place (`apiFetch`) plus the one non-fetch bypass
  (privacy blob download). Active group lives in the persisted auth slice.

## The cache-and-key convention (both repos, one rule)

A cache entry without a tenant in its key is a cross-tenant leak waiting for a hit — filters
don't apply to caches.

- **Backend:** every Caffeine key is prefixed `"<tenantId>:"`. Landed in the *same commit* as the
  Hibernate filter, because RANKINGS had literally two possible keys deployment-wide.
- **Frontend:** every TanStack Query key goes through one `groupKey()` factory →
  `['g', groupId, ...]`; named global exceptions only (`version`, `push/public-key`, `users/me`,
  memberships). **Group switch calls `eraseQueryCache()` unconditionally** — the same erase
  login/logout perform — because the persisted cache outlives a same-token switch. Both measures,
  not either: prefixes make stale entries *miss*; the erase keeps localStorage from carrying one
  group's data into another's session at all.

## Push, in one place

VAPID keys are global (they identify the application server; rotating per-tenant would orphan
every subscription). A browser subscription belongs to the **account** — one endpoint per origin
per the unique constraint — and notification fan-out resolves the account's memberships at send
time. Known 5a limitation, accepted in writing: a notification about group B taps into whatever
group the session has active; the payload carries an advisory `groupId` for a future
switch-on-tap.

## The rollout ladder

Each rung is a plan, independently shippable; **"dark" means the pre-existing test suites pass
unmodified.**

| Rung | Plan | Visible? | Acceptance |
|---|---|---|---|
| 1 | TENANCY-SCHEMA (V22–V27, org #1 backfill) | Dark | ✅ 2026-08-01. Suite unmodified; `TenancySchemaIT` green in CI |
| 2 | TENANCY-ENFORCEMENT (header, predicates, asserts, cache keys, async payloads, SSE, V28 platform grant) | Dark (single-membership fallback) | ✅ 2026-08-02. Suite unmodified **+ `TenantIsolationIT`** green — two seeded orgs invisible to each other at every layer, numerically for the rating and fee aggregates. Enforcement plan §15 records the departures, chiefly explicit predicates instead of a Hibernate filter |
| 3 | TENANT-PRIVACY (leave-group / erase-platform fork) | Dark (fork degenerates at one org) | ✅ built 2026-08-02. `TenantPrivacyIT` byte-compares the untouched group, per BR-P1 |
| 4 | GROUP-ONBOARDING (codes, invites, picker) | **The flip** | ✅ owner sign-off given 2026-08-02 — build authorised. The flip itself is still gated on the first creation codes being issued, which is a separate act |
| 5 | GROUP-BILLING (Stripe) | ⛔ **ON HOLD (owner)** | **Do not schedule or implement** — the owner lifts the hold explicitly or it stays |

~~Standing pre-work, before rung 1: `integrationTest` wired into CI.~~ ✅ Done 2026-08-01; the
whole tier has now executed against a real PostgreSQL container, which is what made rungs 1 and 2
provable rather than merely written.

**Every rung is live as of 2026-08-02.** Rungs 1 and 2 went out together as one dark release and
applied cleanly against real data; the privacy fork and onboarding (`V29`–`V31`) followed the same
day, so the whole chain through V31 is in production.

**`user_roles` is dropped.** It was frozen from V23 as the expand half of an expand/contract, and
V29 removed it — which closes the rollback path to anything before V22, since that would now need a
backup restore rather than an old jar. See STATUS.md hazard 3. What is left of Phase 5 is 5b
billing, on hold by owner decision.

## Bootstrapping the first platform operator

**There is no endpoint for this, on purpose, so it is a database act.** `platform_admins` ships
empty (V28 — a migration that silently grants platform-wide power is the opposite of a dark
launch), and `PlatformAdminRepository` is read-only: nothing in the API can create the first
operator, because such an endpoint would be reachable before any operator existed to guard it. A
deployment that predates tenancy therefore has no operator until its owner grants one by hand:

```sql
-- heroku pg:psql -a footmania   (or psql into whichever database)
SELECT id, username FROM users WHERE username = '<the account>';
INSERT INTO platform_admins (user_id, granted_by) VALUES (<id>, 'owner-bootstrap YYYY-MM-DD');
```

The grant takes effect immediately for API calls (the guard reads the table per request); the
frontend shows the Platform surfaces after the next **login**, because `platformAdmin` rides the
login response.

**The recommended shape is an operator-only account**: register a fresh account (registration
creates no membership), grant it, and keep it out of every group. Operating the deployment is a
different job from playing in a group — the reason the grant is flat and separate from group
`ADMIN` — and an account that can issue creation codes but appears on no roster is the cleanest
expression of that. The frontend supports it: a group-less operator gets a **Platform settings**
card on the picker leading to `/platform` (frontend `d0b7686`); an operator who is also a member
has the same controls as the Settings → Platform tab.

## What deliberately did not change

No Redis (ADR-003's CacheConfig-only swap remains the named escape hatch if cross-tenant
`allEntries` flushes become measurable), no Keycloak (parked — no security chain for that profile
ever existed; scaffolding flagged for removal), no subdomains or path routing (owner chose the
picker), no microservices (the monolith rule's conditions are organisational and unmet), and no
change to what the match-fee ledger is: **no money moves through the application** — billing is
the operator's subscription revenue, a different thing, kept different on purpose.
