# Tenancy Enforcement (Runtime Isolation) — Technical Specification

**Date:** 2026-08-01
**Status:** ✅ **BUILT 2026-08-02** — Phase 5a-2, all six steps of §13. Ships dark; not yet
deployed. See §15 for what was decided differently from this spec and what was deliberately left
**Priority:** HIGH
**Estimated Effort:** L (≈4–5 days backend; no user-visible frontend)
**Depends on:** `TENANCY-SCHEMA-PLAN.md` — enforces a boundary that must already exist in the schema. **Build that first.**
**Depended on by:** `TENANT-PRIVACY-PLAN.md`, `GROUP-ONBOARDING-PLAN.md`
**Contract:** `docs/api/TENANCY-API-CONTRACT.md` (new — the `X-Group-Id` protocol and 404 semantics; updated in the same backend commit)

---

## 1. Requirement Summary

The schema rung made cross-tenant rows unrepresentable. This rung makes cross-tenant **requests**
impossible: every read and write is resolved to a verified active group, every list and aggregate
is scoped to it, every id-keyed lookup asserts ownership, every cache key carries the tenant, and
every asynchronous consumer knows which tenant it is working for.

**This rung also ships dark.** Production still has exactly one organization, and the frontend
does not yet send the new header — the server falls back to the caller's only membership. The
acceptance test is unchanged behaviour on the entire pre-existing suite, plus a new
`TenantIsolationIT` proving two seeded organizations cannot see each other at any layer.

---

## 2. Scope

| In | Out |
|----|-----|
| `X-Group-Id` resolution + membership verification | Group picker/switcher UI (`GROUP-ONBOARDING-PLAN.md`) |
| `TenantContext` per-request (MDC, finally-cleared) | Group creation/invites |
| Hibernate `@Filter` on tenant-owned entities | Erasure semantics fork (`TENANT-PRIVACY-PLAN.md`) |
| `TenantGuard.assertOwned()` + `@PrePersist` stamping | Billing/entitlement checks |
| Tenant-prefixed cache keys; scoped eviction | Redis (named escape hatch only — §7) |
| `tenantId` in event payloads; async context propagation | Any change to Reminder/MVP scheduler *model* |
| SSE/draft ownership checks; `exportForPlayer` IDOR fix | Multi-node SSE (pre-existing, unchanged) |
| Platform-admin grant (minimal surface) | Platform operator console (`GROUP-BILLING-PLAN.md`) |
| V28 `DROP TABLE user_roles` | |

---

## 3. Model decision: `X-Group-Id` verified per request — not a tenant claim in the JWT

**Chosen:** the client sends `X-Group-Id: <organization id>` on every authenticated request.
`JwtAuthenticationFilter` — which already re-loads the user from the database on every request —
verifies an ACTIVE membership for `(user, group)` and puts the tenant on `UserPrincipal` and into
`TenantContext`. **Missing header:** if the caller has exactly one active membership, it is used
(the dark-launch compatibility rule — today's frontend keeps working untouched); with several,
`400` with a body naming the header. **Header naming a group without membership: `404`.**

**Rejected: a tenant claim inside the JWT.** It pins one group per login, so switching groups
means re-issuing tokens; and it goes stale on revocation — a removed member keeps a valid claim
until expiry (24h). The per-request DB verification this codebase *already pays for* makes the
header check nearly free and revocation instant. The V18 role-migration plan made the same call
about roles ("the tokens themselves need no handling at all") — this extends it to tenancy.

**The SSE exception.** `EventSource` cannot set request headers. Draft SSE endpoints therefore
take **no client-asserted tenant at all**: the tenant is derived from the resource (the draft
session row's own `tenant_id` — session ids are globally unique), and the caller's membership in
that tenant is asserted; mismatch → `404`. A query-parameter tenant was rejected as
client-asserted and redundant — the resource already knows who owns it.

## 4. Model decision: three independent layers — not filter-only

**Chosen:** (1) a Hibernate `@Filter("tenantFilter")` on every tenant-owned entity, enabled
per-session from `TenantContext`, scoping every JPQL/criteria query; (2) an explicit
`TenantGuard.assertOwned(entity)` after **every** `findById`-style load; (3) an `@PrePersist`
listener stamping `tenant_id` from context on new rows — with the schema rung's composite FKs
underneath as the layer that cannot be bypassed.

**Rejected: trusting the filter alone.** Documented Hibernate behaviour: `@Filter` does **not**
apply to `em.find()`/`findById`, to `@ElementCollection` tables, or to native queries
(`PlayerRepository.countPlayerStats` is the codebase's one native query). The codebase is
saturated with id-keyed reads behind endpoints that today carry no ownership predicate — a filter
gives them false confidence, an assert gives them a decision. One missed call site under three
layers is a 404-shaped bug; under filter-only it is a data leak.

**BR: cross-tenant access answers `404`, never `403`.** A 403 confirms the id exists — an
existence oracle across tenants. 404 is what "not in your world" looks like.

## 5. Where the context lives

`HttpRequestLoggingFilter` (`@Order(HIGHEST_PRECEDENCE + 10)`, already ahead of the security
chain, already MDC-disciplined with a `finally` clear) gains the `TenantContext` set/clear —
which also puts `tenantId` in **every log line for free**. The security filter *verifies* what
the logging filter provisionally parsed. `TenantContext` is a `ThreadLocal` with an explicit
`runAs(tenantId, Runnable)` for non-request callers (§8).

## 6. The sweep: queries and endpoints

Every whole-table read from the schema inventory becomes tenant-scoped (the filter does most
mechanically; these are re-verified by name in `TenantIsolationIT`):

- `PlayerRepository`: scarcity averages (`findAverageGoalsPerMatch`/`Assists` — **the
  cross-tenant rating-contamination pair**, same argument as V21's guest guards one level up),
  rankings both variants, top scorers/assisters/streaks, `findAllByActive`.
- `PlayerStatRepository.countResultsByPlayer` / `countMvpsByPlayer`.
- `MatchFeeService.balancesByPlayer` + `allBalances` (`findAll()` *is* the group definition
  today); `CalculationService.endSeason` (global mean!); `DraftSessionService.getAll*`;
  `PlayerService.listPlayers`; `UserService.listUsers` → membership-scoped;
  `AppSettingsService.getAll/getOverrides`; `MatchRepository`/`MatchPlanRepository` search paths.
- Every `{id}` endpoint gains `assertOwned` via its service load. Named specially:
  `PrivacyService.exportForPlayer(playerId)` — **the most dangerous IDOR in the app** under
  multi-tenancy (an admin of group A exporting group B's player by id) — asserts tenant before
  building anything.
- `SystemHealthService`: per-group counters (subscriptions of members, badges, matches) scoped;
  VAPID/version stay platform facts, visible to platform admin (§9) and inert for group admins.

## 7. Model decision: Caffeine stays; keys gain the tenant in this same change — not Redis, not later

**Chosen:** every `@Cacheable` key gains a `TenantContext.currentTenant()` prefix (the
app-settings key already carries it from the schema rung). `RANKINGS` (two possible keys
deployment-wide today) and `LEADERBOARDS` (~25) are the proof of necessity: a cache hit bypasses
the Hibernate filter entirely, so **un-prefixed keys make the filter worthless on the hottest
read paths**. Same change, same commit — booked as a BR.

`@CacheEvict(allEntries = true)` (~15 sites) becoming a cross-tenant flush is **accepted for 5a**
— wasteful, never wrong. `maximumSize` moves to configuration and scales with group count.
`AdminController.evictCaches` becomes platform-admin only; group admins get a tenant-scoped
evict. **Rejected: Redis** — ADR-003's no-new-infra stance holds; its own escape hatch (swap in
`CacheConfig` only) is the named trigger if flush storms become measurable with real group counts.

## 8. Model decision: the tenant travels in event payloads — not a TaskDecorator

**Chosen:** `MatchCompletedEvent` and `DraftStateChangedEvent` gain `tenantId`; every async
consumer (`MatchEventListener`, `PushNotificationService` ×2, `DraftSseEmitterRegistry`) wraps its
work in `TenantContext.runAs(event.tenantId(), …)` with try/finally. Schedulers keep their
**global sweep, per-row claim** model unchanged — each claimed row carries its tenant, and the
dispatcher sets context per row. `BadgeBackfillService` alone changes model: it becomes scoped to
the triggering admin's group (a tenant admin must not replay every tenant's history).

**Rejected: a context-propagating `TaskDecorator`** on the executor. It captures ambient state
implicitly — unauditable, and simply absent for scheduler-originated work where there *is* no
request context to capture. Explicit payload beats ambient magic; this is the same reasoning
ADR-002 used for putting work after commit explicitly rather than hoping.

## 9. Model decision: platform admin is a platform-level grant — not group ADMIN, not a special org

**Chosen:** a `platform_admins(user_id)` table (or boolean on `users` — implementer's choice,
table recommended for auditability), checked by a `@PreAuthorize("@platformGuard.isOperator(...)")`
style predicate. Surface this rung: global cache evict, org listing, platform health. **"ADMIN is
not a superset" survives Phase 5**: group ADMIN administers a group — and *every founder will
hold it* — so it cannot carry operator powers; a membership in a magic "platform org" was
rejected because it breaks the tenant invariants everywhere else.

## 10. Keycloak: parked, explicitly

**Chosen:** HMAC JWT + per-request DB membership resolution (which §3 depends on) stays; the
Keycloak scaffolding — a realm predating V18 roles, and **no `@Profile("keycloak")` security
chain at all** (activating the profile today removes all security) — is declared out of scope and
flagged for removal as cleanup. **Rejected: adopting Keycloak for Phase 5** — realm-per-tenant vs
single-realm is a real design effort purchasing nothing this phase needs, at the price of the
first piece of mandatory new infrastructure.

## 11. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-E1 | Every authenticated request resolves to exactly one verified ACTIVE membership | Header; single-membership fallback keeps 5a dark |
| BR-E2 | Cross-tenant anything → `404` | No existence oracle |
| BR-E3 | Cache keys carry the tenant, same commit as the filter | The filter is worthless without it |
| BR-E4 | Async work declares its tenant in the payload | No ambient inheritance |
| BR-E5 | Platform grant ≠ group ADMIN | Flat, separate, auditable |
| BR-E6 | SSE derives tenant from the resource, never the client | EventSource cannot send headers anyway |

## 12. Test plan

| Area | Cases |
|------|-------|
| **`TenantIsolationIT`** (Testcontainers; `GuestIsolationIT` is the template; `application` package per CI invariant) | Two orgs seeded with deliberately skewed data; (a) every list/aggregate returns own rows only — asserted **numerically** for scarcity averages, `endSeason` mean, fee balances, so contamination *changes a number*, not just a count; (b) every `{id}` endpoint cross-tenant → 404; (c) cache isolation: prime rankings as A, fetch as B; (d) SSE subscribe/pick cross-tenant → 404; (e) `exportForPlayer` cross-tenant → 404; (f) event-driven paths (match completion) touch only their tenant |
| Unit tier (H2 — cannot see SQL constraints) | `TenantContext`/`TenantGuard`, header resolution incl. fallback + 400 + 404, cache-key composition, event payload propagation, platform-guard predicate |
| Dark-launch acceptance | Pre-existing suites pass unmodified with no header sent (single-membership fallback) |
| Regression guard (optional, recommended) | ArchUnit/grep CI rule: no new repository method without a tenant predicate, outside a named allowlist (login lookup, scheduler sweeps, version/health) |

## 13. Order of work

1. `TenantContext` request resolution + header protocol + contract doc.
2. Filters + `assertOwned` sweep + `@PrePersist` stamping (one PR; mechanical but wide).
3. Cache keys + eviction scoping (same PR as filters — BR-E3).
4. Event payloads + async `runAs` + BadgeBackfill scoping.
5. SSE ownership + `exportForPlayer` fix + platform grant + V28 drop `user_roles`.
6. `TenantIsolationIT` green in CI **before** the visibility flip is even scheduled.

## 14. Breaking changes

- [x] **None for today's clients.** The header is optional while every account holds one
      membership; every existing request resolves as before.
- [ ] **Deliberate API posture change:** id-keyed endpoints that once answered 200/403 across
      what will become tenant lines answer 404. Unobservable at one tenant; the contract doc
      states it from day one.

---

## 15. What was built, and where it departs from the above

**Written 2026-08-02, after the fact.** The spec above is left as it was written; this section is
the honest diff. Steps 1–3 landed in `a9a513a`/`05d05aa`/`9c9a231`, steps 4–6 in `1a028e7`.

### Departures

**Explicit predicates instead of a Hibernate `@Filter` (§4).** The filter needs either a new AOP
dependency with reordered transaction advice, or Hibernate 6's `@TenantId` — which takes
insert-time stamping away from the `@PrePersist` listener the schema rung had just built. Every
whole-table read named in §6 now carries a `tenantId` parameter instead. The layered posture the
section argued for is intact and arguably stronger: predicates, `assertOwned`, stamping, composite
FKs — and unlike a filter, a predicate covers the codebase's one native query, which §4 named as
the reason not to trust a filter alone.

The cost is honest: a predicate can be forgotten where a filter is ambient. §12's optional grep/
ArchUnit regression guard is therefore **no longer optional**, and is the top follow-up.

**`TenantContext.currentTenant()` throws when unbound.** Not in the spec. A query against
`tenant_id = null` matches nothing and reads as "no data" rather than as the bug it is. It broke
393 tests on first run; every one was a genuine unbound path.

**V28 is the platform-admin table, not `DROP TABLE user_roles` (§13 step 5).** The drop is the
contract half of V23's expand/contract and is due after a release of soak. V22–V27 are merged but
**not deployed**, so the soak has not started — dropping now would remove the way back from a
rollback. The drop becomes V29.

This was the single largest deviation from §13, and it was raised for exactly that reason rather
than made quietly. **The owner confirmed it on 2026-08-02**, so §13 step 5 should now be read as
superseded: the drop is V29, gated on V22–V27 having been live for a release, and a session that
finds it unwritten is looking at a decision rather than a gap.

**The SSE path needed a resolver exemption (§3).** §3 said the SSE endpoints take no client tenant
and derive it from the resource, which is what they do. What it did not anticipate: the resolver
sits in the filter chain and would have answered `400` to a multi-group caller *before* the handler
could derive anything. `TenantResolver` therefore recognises the one path and leaves it unbound
rather than refusing. A header sent on that path anyway is still verified.

**Bulk id loads were scoped uniformly.** §6 named `findById`-style loads. `findAllById` — eight
sites taking ids from requests and from already-owned parents alike — was not named, and is the
same hole with a collection in it. All eight became `findAllByIdInAndTenantId`, including the ones
whose ids provably come from a row the caller owns, because "these ids came from somewhere safe"
stops being true the moment somebody adds a caller.

**`listUsers` scoping needed a join, not a column (§6).** `AppUser` is platform-level by design, so
there is no `tenant_id` to filter on. The listing joins `memberships`, and the id-keyed user
operations assert an ACTIVE membership rather than using `TenantGuard`, which only understands
`TenantOwned`.

### Deliberately not done

- **`DELETE /api/users/{id}` still deactivates the account**, not the membership. Under several
  groups that is more than one group's admin should decide — but narrowing it is the erasure fork
  `TENANT-PRIVACY-PLAN.md` owns. This rung only stops an admin reaching an account that was never
  theirs.
- **`scrubCreatedBy` stays global.** It runs while the account itself is being deleted, so a
  username-wide scrub is coherent; half-scoping it would be worse than either alternative. 5a-3's
  fork is where it gets decided properly.
- **Keycloak scaffolding not removed.** §10 flagged it as cleanup; it stayed out of a change this
  wide.
- **`@CacheEvict(allEntries = true)` is still a cross-tenant flush** at ~15 sites, as §7 accepted
  for 5a. The *admin-triggered* evict is now scoped, because that one is reachable by a customer.

### Follow-ups this rung created

1. **The regression guard is now load-bearing** (§12) — a CI rule that no repository method gains a
   whole-table read without a tenant predicate, outside a named allowlist (login lookup, scheduler
   sweeps, `PlatformService`).
2. **`PlatformService` is the one deliberately unscoped service.** It says so in its own javadoc;
   the allowlist above must name it, or the guard will fight it every release.
3. **The group-scoped cache evict walks Caffeine's keyspace** to find this tenant's prefix — linear
   in cache size, fine at these sizes, and the named trigger for the Redis escape hatch in §7 if
   group counts ever make it measurable.
