# Project Status

**Snapshot: 2026-08-02.** Update it when the answers change, not on a schedule — a status page
nobody trusts is worse than none.

| | Backend | Frontend |
|---|---|---|
| Branch | `master` | `v1.0.0` |
| Head | `7b9fe59` — *Merge #142: bump to 1.1.0* | `f42575c` — *Fix roster row overlap on mobile…* |
| Version | `1.1.0` — cut 2026-08-02 for the Phase 5a release, changelog written | — |
| Working tree | clean | clean |
| Tests | 958 unit + integrationTest green in CI | 470 unit + 20 visual regression |
| Latest migration | `V28__platform_admins.sql` — in `master`, **not deployed**; production is on V21 | — |

⚠️ **Merged ≠ deployed, and this is now the only gap.** As of 2026-08-02 the whole Phase 5a
schema-and-enforcement chain is in `master` — V22 through V28, plus the code that enforces it. It
has run against a PostgreSQL container in CI and has **never touched the production database**,
which still stands at V21. Everything below marked "shipped" for Phase 5 means *in master*, not
*live*.

**The next action on Phase 5 is a deploy**, not more code: one release, behind a database backup.
The chain is dark end to end — nothing user-visible changes, and the observable difference is seven
rows in `flyway_schema_history` plus a version that reads `1.1.0`. It is also what starts the soak
clock V29 depends on.

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
| **5** — multi-tenancy + billing | 🟡 **two rungs merged, neither deployed.** 5a-1 (schema, V22–V27) 2026-08-01; 5a-2 (enforcement + V28) 2026-08-02, [PR #141](https://github.com/ricsnsuka/FootMania-Back/pull/141). Both ship **dark** — one organization, an optional header, no behaviour change. Remaining: 5a-3 privacy fork, 5a-4 onboarding (**the visibility flip**, owner-gated). ⛔ **The billing rung is ON HOLD by owner decision — do not start it, in any session, until the owner lifts the hold in their own words** |

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

**2. V22–V28 have never run against the production database.** The successor to hazard 1, and the
reason that one is worth remembering rather than deleting. The chain is green in CI against a
PostgreSQL container — which is genuinely more than V17–V19 ever got before their deploy — but CI
seeds an empty schema and production has two years of rows. V24's `NOT NULL` backfill and V25's ~29
recreated composite FKs are where a real dataset differs from a fresh one. Take the backup, run it
behind the gate, and expect V25 to be the slow one.

**3. `user_roles` must survive until V22–V27 have been live for a release.** *(Owner-confirmed
2026-08-02.)* V23 froze it as the expand half of an expand/contract; auth reads `membership_roles`
and the role-update endpoint dual-writes both. The drop is **V29**, not the V28 the enforcement
plan pencilled in, and it **must not be run early** — while V22–V27 are undeployed, that table is
the only thing that makes a rollback to pre-V22 code survivable.

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
| [backend/architecture/ARCHITECTURE.md](backend/architecture/ARCHITECTURE.md) | Migration table stops at **V13** of 19 | Superseded by [architecture/database-migrations.md](architecture/database-migrations.md) — reconcile or point at it |
| [backend/api/API_REFERENCE.md](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/API_REFERENCE.md) | No Push section, no Payments/match-fee section; `totalCostCents` on `MatchPlanDTO` documented nowhere | Contracts exist standalone ([PUSH](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PUSH-API-CONTRACT.md), [PAYMENTS](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PAYMENTS-API-CONTRACT.md)); the reference needs the sections and the field |
| [backend/features/MATCH_PLANS_FEATURE.md](backend/features/MATCH_PLANS_FEATURE.md) | Last touched 2026-05-27 — predates kickoff time, lifecycle/expiry, waitlist, past-plan split and pitch cost | Rewrite against current behaviour |
| [backend/plans/MATCH-FEE-LEDGER-PLAN.md](backend/plans/MATCH-FEE-LEDGER-PLAN.md) | Header read "DRAFT — not implemented"; it shipped in `828db3b` | ✅ corrected here on import; **the copy in the backend repo is still wrong** |
| [backend/plans/ORCHESTRATOR_SESSION.md](backend/plans/ORCHESTRATOR_SESSION.md) | Last entry 2026-07-28, though `orchestrator.agent.md` still mandates an entry per session | Resume it, or retire the convention deliberately |
| Postman collection (not imported) | Last updated 2026-07-10: 60 requests against 84 controller mappings. Missing entirely: Payments (9), Push (5), Admin settings/system (5), Privacy (4), MOTM (2), rankings, leaderboards, badges, and now the three platform-operator endpoints. Its changelog still describes a 37-request collection | Regenerate; roughly 31 endpoints of work. Every request also needs an optional `X-Group-Id` |
| [frontend/INDEX.md](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/v1.0.0/docs/INDEX.md) | States "Every new doc file must be linked here" and then omits seven of its own: badges, motm-voting, rankings, roles, settings, team-generation, theme | Relink |
| frontend — missing file | No `features/payments.md`, though `a3efac0` shipped the whole "what you owe" UI and every other feature has one | Write it |

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
2. **Add the repository-predicate regression guard.** Phase 5a-2 chose explicit tenant predicates
   over a Hibernate filter, which is deterministic and covers the native query — but a predicate
   can be forgotten where a filter is ambient. A grep/ArchUnit CI rule ("no whole-table read
   without a tenant predicate, outside a named allowlist: login lookup, scheduler sweeps,
   `PlatformService`") is what keeps that trade honest. See the enforcement plan §15.
3. **Decide when V22–V28 deploy.** They are the gate on everything else in Phase 5, they have never
   met a real dataset, and `user_roles` cannot be dropped until they have been live for a release.
4. ~~Backfill the changelog~~ ✅ done 2026-08-02 — fifteen sections written from the commit
   messages and cut as `[1.1.0]`. Keep fixing the rest of the drift table above.
5. Regenerate the Postman collection — now with `X-Group-Id`.
6. Add the locale key-parity test, and a controller test asserting `totalCostCents` serialises.
7. Then 5a-3 (privacy fork), or Phase 3's last item — AI match reports. 5a-4 is the visibility flip
   and is owner-gated; billing is on hold.
