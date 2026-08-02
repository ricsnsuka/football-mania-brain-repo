# Project Status

**Snapshot: 2026-08-02.** Update it when the answers change, not on a schedule — a status page
nobody trusts is worse than none.

| | Backend | Frontend |
|---|---|---|
| Branch | `master`, and `v1.0.0` rebased onto it | `v1.0.0` |
| Head | `013562c` — *Repoint the links the docs move left dangling* (`master` at `714924a`) | `286d6b7` — *Rewire this repo's agents to the brain repo too* |
| Version | `1.1.0` — **deployed 2026-08-02** | `1.1.0` — matched to the backend |
| Working tree | clean | clean |
| Tests | 1000 unit across 194 suites + integrationTest green in CI | 485 unit + 20 visual regression |
| Latest migration | `V31__group_creation_codes.sql` — **merged, not deployed** | — |
| Deployed through | `V28` (2026-08-02) | — |

✅ **Phase 5a is live.** V22–V28 were applied in production on 2026-08-02, and both repos are on
`1.1.0`. The deployment is dark by design: one organization, an optional header, no user-visible
change.

**This starts the soak clock.** `user_roles` is still frozen and still load-bearing — see hazard 3.
V29 drops it, and the rule is one release of soak, not one deployment event.

---

## Where we are on the roadmap

See [product/roadmap.md](product/roadmap.md) for the plan itself.

| Phase | State |
|---|---|
| **0** — branding, `is_current` season gap, GDPR posture | ✅ done |
| **1** — PWA groundwork | ✅ done (frontend) |
| **2** — Web Push | ✅ done, delivery observed end to end against a real push service |
| **3** — the "steal from Capo" set | 🟡 balance-at-a-glance, waitlist, leaderboards/rankings, MOTM voting and badges all shipped in both repos. **Remaining: AI match reports** |
| **4** — Capacitor + app stores | ⬜ not started, deliberately gated on real usage data |
| **5** — multi-tenancy + billing | 🟡 **two rungs live, 5a-4 authorised.** 5a-1 (schema, V22–V27) and 5a-2 (enforcement + V28) both deployed 2026-08-02. Running **dark** — one organization, an optional header, no behaviour change. 5a-3 privacy fork ✅ built 2026-08-02, which unblocked 5a-4. ✅ **5a-4 onboarding is built too** — `V29`–`V31`, group creation behind an operator-issued code, invites, and the last step of the guest arc. All of 5a is now written; `V29`–`V31` are **merged and not deployed**. ✅ **The 5a-4 owner gate was lifted on 2026-08-02** — the roadmap's "real engagement signal from push" precondition is explicitly waived. **Issuing creation codes is still not authorised**: that is the visibility flip itself and stays a separate, deliberate act, and `group_creation_codes` ships empty. ⛔ **The billing rung is ON HOLD by owner decision — do not start it, in any session, until the owner lifts the hold in their own words** |

Built alongside the roadmap, not on it: runtime-configurable competition rules, admin settings and
system endpoints, match-plan kickoff time and lifecycle, composable roles, and the match fee
ledger. The ledger is bookkeeping only — **no money moves through the application**, by design.
See [backend/plans/MATCH-FEE-LEDGER-PLAN.md](backend/plans/MATCH-FEE-LEDGER-PLAN.md) §14 for how a
real payment integration would bolt on later without any of it being wasted.

---

## Live hazards

**1. ~~V18 and V19 have never run against a real database.~~ Resolved 2026-08-01.** The whole
chain through **V21** is now applied in production — the guest-players feature (V20+V21, and
therefore everything before it) was observed live the day it merged. The lasting lesson stands,
twice over: `integrationTest` is the cheapest pre-flight and is still opt-in, and the first real
use of the guest feature immediately surfaced a defect the mocked suite could not see (guest
removal failed at Hibernate flush — see the guest plan's "what actually happened"). Run the real
chain, and prefer real-persistence tests for anything that deletes.

**2. ~~V22–V28 have never run against the production database.~~ Resolved 2026-08-02.** The whole
chain applied cleanly. What is worth keeping from it: unlike V17–V19, this chain had run against a
real PostgreSQL container in CI before it went anywhere near production — `integrationTest` stopped
being opt-in in time to matter. Two hazards in a row have now been closed by the same discipline,
and hazard 1's lesson is the reason this one was cheap.

**3. `user_roles` must survive until the deployed chain has been live for a release.**
⚠️ **The most live hazard on this page, as of the 2026-08-02 deploy.** *(Owner-confirmed
2026-08-02.)* V23 froze it as the expand half of an expand/contract; auth reads `membership_roles`
and the role-update endpoint dual-writes both. The drop is **V29**, not the V28 the enforcement
plan pencilled in, and it **must not be run early**. The soak began with the 2026-08-02 deploy and
ends when the *next* release ships — one release of running code, not one deployment event. Until
then that table is the only thing that makes a rollback to pre-V22 code survivable, and a rollback
is exactly what you need if the tenancy chain turns out to have a problem nobody has hit yet.

**V29 is now written and merged** — the hazard has moved from "do not write it" to "do not deploy
it early". Deploying `V29`–`V31` is the act that ends the way back: past V29 a rollback to before
V22 means restoring a backup, not redeploying an old jar, because pre-V22 code reads authorities
from a table that no longer exists and every account returns with no grants. Verify first with the
`EXCEPT` query in the migration's own header; zero rows means the copy is complete.

**4. A stale JVM will lie to you.** On 2026-07-31 a bug was reported where a match plan's pitch
cost saved correctly, notified players with the right amount, and still read "not set" in the UI.
The code was correct in all three layers. The Spring Boot process had been started *before* the
class was recompiled, so it was still serving the previous `MatchPlanDTO` — the running app's own
`/v3/api-docs` schema was missing the field. If a field is definitely in the DTO and definitely
absent from the response, check the process start time against the class file's mtime before
looking anywhere else.

---

## Known documentation drift

Found by auditing the imported documents against the code on 2026-07-31. Each line is a document
that is **wrong or incomplete**, not merely old. Fix the document, then delete its line.

| Where | Problem | Correction |
|---|---|---|
| [backend/architecture/ARCHITECTURE.md](backend/architecture/ARCHITECTURE.md) | Migration table stops at **V13** of 31, and the rest of it predates multi-tenancy | Superseded on migrations by [architecture/database-migrations.md](architecture/database-migrations.md); the layering and schema sections need a tenancy pass |
| [backend/api/API_REFERENCE.md](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/API_REFERENCE.md) | No Push section, no Payments/match-fee section; `totalCostCents` on `MatchPlanDTO` documented nowhere | Contracts exist standalone ([PUSH](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PUSH-API-CONTRACT.md), [PAYMENTS](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PAYMENTS-API-CONTRACT.md)); the reference needs the sections and the field |
| [backend/features/MATCH_PLANS_FEATURE.md](backend/features/MATCH_PLANS_FEATURE.md) | Last touched 2026-05-27 — predates kickoff time, lifecycle/expiry, waitlist, past-plan split and pitch cost | Rewrite against current behaviour |
| ~~[backend/plans/MATCH-FEE-LEDGER-PLAN.md](backend/plans/MATCH-FEE-LEDGER-PLAN.md)~~ | Header read "DRAFT — not implemented"; it shipped in `828db3b` | ✅ **Resolved.** Corrected here on import, and the stale backend copy went with the documentation split — there is one copy now, and it is this one |
| [backend/plans/ORCHESTRATOR_SESSION.md](backend/plans/ORCHESTRATOR_SESSION.md) | Last entry 2026-07-28, though `orchestrator.agent.md` still mandates an entry per session | Resume it, or retire the convention deliberately |
| Postman collection (not imported) | Last updated 2026-07-10: 60 requests against 84 controller mappings. Missing entirely: Payments (9), Push (5), Admin settings/system (5), Privacy (4), MOTM (2), rankings, leaderboards, badges, and now the three platform-operator endpoints. Its changelog still describes a 37-request collection | Regenerate; roughly 31 endpoints of work. Every request also needs an optional `X-Group-Id` |
| ~~[frontend/INDEX.md](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/v1.0.0/docs/INDEX.md)~~ | Omitted seven of its own files | ✅ **Resolved** by the documentation split — the feature docs it failed to link now live here, and what is left there is a pointer plus the seven guides |
| frontend — missing files | No `features/payments.md`, though `a3efac0` shipped the whole "what you owe" UI; and nothing for the group dimension (`fcc6c59`), guest players or payment delegation (`722335c`), all of which shipped in the UI | Write them. Every other frontend feature has a file |
| **this repo — no tenancy feature doc** | [architecture/multi-tenancy.md](architecture/multi-tenancy.md) and the plans carry the design, but `backend/features/` has nothing on organizations, memberships or per-membership roles, while nine older features each have a file | Write `backend/features/TENANCY.md`. It is the one thing a new session most needs and currently has to reconstruct from four plans |

A related failure, already fixed: the notification-settings screen rendered `MVP_VOTE_OPEN` and
`FEE_CHARGED` as raw enum names because both categories were added backend-side without anyone
adding the `en`/`pt`/`es` strings. Two separate commits missed it, and nothing tests that the
locale files cover `NotificationCategory`. **A key-parity test still does not exist.**

The shape of all of this is the same: the code shipped, the follow-through did not. That is the
reason this repo exists.

---

## Suggested next steps

1. ~~Wire `integrationTest` into CI and run it.~~ ✅ Done 2026-08-01/02. The tier is green on every
   pull request; `GuestIsolationIT`, `TenancySchemaIT` and `TenantIsolationIT` have all now
   executed. Worth naming as a win: `TenantIsolationIT` was written specifically so a leak would
   change a *number*, and it could only prove that once something ran it.
2. ~~Add the repository-predicate regression guard.~~ ✅ done 2026-08-02 —
   `RepositoryScopingGuardTest`, three checks in the unit tier. **It found two live cross-tenant
   defects the 5a-2 sweep had missed**, both inherited `JpaRepository` methods the sweep's grep
   could not see: `GET /api/players` with no `active` filter returned every group's roster, and the
   admin bulk recalculation could be pointed at another group's match ids — a cross-tenant
   *write*, since recalculation rewrites ratings. Both fixed in the same change. Each check was
   verified to fail when its defect is reintroduced, because a guard that passes for the wrong
   reason is worse than none.
3. ~~**Decide when V22–V28 deploy.**~~ ✅ Deployed 2026-08-02, dark and clean. The live question is
   now **when `V29`–`V31` deploy**: the soak ends with the next release, and that release is the
   point of no return described in hazard 3. Take a backup, run the `EXCEPT` check, then ship.
4. ~~Backfill the changelog~~ ✅ done 2026-08-02 — fifteen sections written from the commit
   messages and cut as `[1.1.0]`. Keep fixing the rest of the drift table above.
5. Regenerate the Postman collection — now with `X-Group-Id`.
6. Add the locale key-parity test, and a controller test asserting `totalCostCents` serialises.
7. Then 5a-3 (privacy fork), or Phase 3's last item — AI match reports. 5a-4 is the visibility flip
   and is owner-gated; billing is on hold.
