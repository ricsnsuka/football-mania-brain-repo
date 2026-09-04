# Documentation Index

Everything in this repo, plus pointers to the documents that deliberately stayed with the code.
⚠️ marks known drift — see [STATUS.md](STATUS.md#known-documentation-drift).
Why a document is here rather than there: [where-documents-live.md](where-documents-live.md).

---

## Start here

| Document | What it is |
|---|---|
| [README](README.md) | What this repo is and the source-of-truth rule |
| [STATUS](STATUS.md) | Where the project stands, live hazards, the drift register |
| [where-documents-live](where-documents-live.md) | Which repo owns which document, and why |
| [CONTRIBUTING](CONTRIBUTING.md) | How to keep this true, and the ship checklist |
| [product/roadmap](product/roadmap.md) | The five-phase improvement and mobile roadmap |
| [product/feature-status](product/feature-status.md) | Every feature × backend × frontend × docs |
| [product/backlog-2026-09](product/backlog-2026-09.md) | The batched backlog that became **3.3.0** — what shipped, the four unbuilt specs and where they live, what the batch got wrong about itself, and what is still open |
| [architecture/system-overview](architecture/system-overview.md) | How the two repos meet; the conventions that bite |
| [architecture/database-migrations](architecture/database-migrations.md) | Complete V1–V33 history + deployment state |
| [architecture/multi-tenancy](architecture/multi-tenancy.md) | Phase 5 cross-repo contract: tenant resolution, cache/key rules, the rollout ladder |
| [architecture/app-store-strategy](architecture/app-store-strategy.md) 📝 | Phase 4 draft: the Capacitor route, the four obstacles, and when *not* to start |
| [architecture/code-review-2026-08-24](architecture/code-review-2026-08-24.md) ⚠️ | Whole-codebase read of `2.0.0`, both repos. Eight open findings, ordered, with a suggested branch split |

---

## Backend

### Architecture and features

| Document | What it is |
|---|---|
| [ARCHITECTURE](backend/architecture/ARCHITECTURE.md) ⚠️ | Layering, packages, schema. Its migration table stops at V13 — use [database-migrations](architecture/database-migrations.md) |
| [ARCHITECTURE_DECISIONS](backend/next-release/ARCHITECTURE_DECISIONS.md) | ADR-001 optimistic locking · ADR-002 async completion · ADR-003 Caffeine · ADR-004 LLM vendor |
| [MICROSERVICES_ARCHITECTURE](backend/features/MICROSERVICES_ARCHITECTURE.md) | Why it is still one deployable |
| [CALCULATION_SERVICE](backend/features/CALCULATION_SERVICE.md) | The skill-rating engine — formulas, constants, worked examples |
| [MATCH_FEATURE](backend/features/MATCH_FEATURE.md) | Match lifecycle, completion, amendment |
| [MATCH_PLANS_FEATURE](backend/features/MATCH_PLANS_FEATURE.md) | Plans, the availability poll, the `timeframe` split and the `from`/`to` kickoff window. Rewritten against the code 2026-08-05 |
| [PLAYER_FEATURE](backend/features/PLAYER_FEATURE.md) | Roster, aggregates, soft delete |
| [USERS](backend/features/USERS.md) | Accounts, authentication, roles |
| [SEASON_FEATURE](backend/features/SEASON_FEATURE.md) | Seasons, the single-current invariant, and the write API added 2026-08-07 |
| [TEAM_GENERATION_FEATURE](backend/features/TEAM_GENERATION_FEATURE.md) · [DESIGN](backend/features/TEAM_GENERATION_DESIGN.md) | The six strategies, and how they were evaluated |
| [DRAFT_SESSION_FEATURE](backend/features/DRAFT_SESSION_FEATURE.md) | Interactive captain pick |
| [PRIVACY_AND_DATA_PROTECTION](backend/features/PRIVACY_AND_DATA_PROTECTION.md) | What personal data exists, and both GDPR paths |
| [MATCH_TEAM_SPEC](backend/next-release/MATCH_TEAM_SPEC.md) | Match & team feature specification |
| [RELEASE_NOTES](backend/next-release/RELEASE_NOTES.md) | Accumulated notes for the next release |

### History

| Document | What it is |
|---|---|
| [INCIDENT 2026-05-26](backend/fixes/INCIDENT_2026-05-26_Java_Version_Mismatch.md) | Java 21 vs IntelliJ's 17 |
| [DEPENDENCY-CVE-SCAN 2026-08-24](backend/security/DEPENDENCY-CVE-SCAN-2026-08-24.md) | Dependencies, both repos — and the CI checks that keep it current |
| [SECURITY-AUDIT 2025-07-09](backend/security/SECURITY-AUDIT-2025-07-09.md) · [2025-07-16](backend/security/SECURITY-AUDIT-2025-07-16.md) | Two audit passes — application security; superseded on **dependencies** by the scan above |
| [code-review 2026-08-24](architecture/code-review-2026-08-24.md) | Cross-repo, so it lives in `architecture/` rather than here. Linked from Start here |

### Plans and handoffs

| Document | Status |
|---|---|
| [ROLE-MODEL-MIGRATION-PLAN](backend/plans/ROLE-MODEL-MIGRATION-PLAN.md) | ✅ complete, both repos |
| [PAYMENT-DELEGATION-PLAN](backend/plans/PAYMENT-DELEGATION-PLAN.md) | ✅ shipped, both repos — V20, live 2026-08-01 |
| [GUEST-PLAYERS-PLAN](backend/plans/GUEST-PLAYERS-PLAN.md) | ✅ shipped, both repos — V21, live 2026-08-01. §12b: day-one removal bug, fixed same day. ⚠️ `GuestIsolationIT` never run |
| [TENANCY-SCHEMA-PLAN](backend/plans/TENANCY-SCHEMA-PLAN.md) | ✅ shipped — Phase 5a-1: organizations, memberships, `tenant_id` everywhere (V22–V27). **Live 2026-08-02** |
| [TENANCY-ENFORCEMENT-PLAN](backend/plans/TENANCY-ENFORCEMENT-PLAN.md) | ✅ shipped — 5a-2: `X-Group-Id`, predicates + asserts, tenant cache keys, async payloads, SSE ownership, platform grant (V28), `user_roles` dropped (V29). **Live 2026-08-02**. §15 records the departures |
| [TENANT-PRIVACY-PLAN](backend/plans/TENANT-PRIVACY-PLAN.md) | ✅ shipped — 5a-3: leave-group vs erase-platform, two-tier export, controller/processor split. **Live 2026-08-02** |
| [GROUP-ONBOARDING-PLAN](backend/plans/GROUP-ONBOARDING-PLAN.md) | ✅ shipped — 5a-4: creation codes, invites, picker/switcher, join page (V30–V31). **Live 2026-08-02**. ⚠️ **The visibility flip is not the deploy** — no code has been issued, so nothing is self-serve yet |
| [GROUP-BILLING-PLAN](backend/plans/GROUP-BILLING-PLAN.md) | ⛔ **ON HOLD (owner decision, 2026-08-01)** — design reference only, **do not implement** until the owner lifts the hold |
| [SCOPED-API-TOKENS-PLAN](backend/plans/SCOPED-API-TOKENS-PLAN.md) | ✅ shipped in **1.8.0** (V39) — a revocable, group-bound, narrowly-scoped personal token. **The prerequisite for every automation route**, including "Hey Siri, record a goal" from an Apple Watch, which needs no app changes and only a credential a shortcut can hold. Scopes *intersect* live grants rather than replacing them |
| [PLATFORM-CONSOLE-PLAN](backend/plans/PLATFORM-CONSOLE-PLAN.md) 📝 | DRAFT — operator monitoring. **Rung 0 is a rendering job**: the organization listing and platform counters already exist and have no screen |
| [MATCH-FEE-LEDGER-PLAN](backend/plans/MATCH-FEE-LEDGER-PLAN.md) | ✅ shipped — §14 covers a future payment integration |
| [PLAYER-LINK-ME-PLAN](backend/plans/PLAYER-LINK-ME-PLAN.md) | ✅ shipped |
| [OPTIMAL-PARTITION-PLAN](backend/plans/OPTIMAL-PARTITION-PLAN.md) 📋 | SPECIFIED, not built — exact-search team generation with a pluggable objective. **§7 records three pre-existing defects found while specifying**, including `params[…]` never reaching any strategy (verified), so `formWindow` and the `CAPTAIN_PICK` captain overrides have never worked |
| [RANK-LADDER-PLAN](backend/plans/RANK-LADDER-PLAN.md) 🟡 | **Step 1 built, in review** ([Back#275](https://github.com/ricsnsuka/FootMania-Back/pull/275)): the engine and V50, nothing reading them yet; steps 2–5 unbuilt — Iron→Master tiers of three 75-FP divisions in front of the raw rating, **Form Points** paid per match from result and performance, three placement matches per season, a soft reset that reads FP, and **Best of the Crop** for the top Masters. Owner decisions taken 2026-09-04; every other constant waits for a history replay (§8). Target `3.5.0` |
| [PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM](backend/plans/PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM.md) | 🟡 all done except AI match reports. ⚠️ **"Keep the crowd result separate from `isMvp`" is superseded in part (2026-08-08)** — the two columns stay separate, but the MVP counts now read the crowd's |
| [RETIRE-ORGANISER-MVP-PICK-PLAN](backend/plans/RETIRE-ORGANISER-MVP-PICK-PLAN.md) 📋 | SPECIFIED, not built — the MVP counts become the crowd's (written 2026-08-08, **not yet released**), leaving the organiser's toggle on the match form feeding no total. **§2 is the trap**: `is_mvp` is the only MVP record for every pre-voting match and the board now depends on it, so any plan starting with the migration has the order backwards. Rung 1 needs no migration; §5 is an open owner decision |
| [PHASE0_FRONTEND_HANDOFF](backend/plans/PHASE0_FRONTEND_HANDOFF.md) | ✅ done |
| [REVIEW_HANDOFF_2026-07-27](backend/plans/REVIEW_HANDOFF_2026-07-27.md) | ✅ backlog cleared |
| [ORCHESTRATOR_SESSION](backend/plans/ORCHESTRATOR_SESSION.md) ⚠️ | Session log — **no entries since 2026-07-28** |

---

## Frontend

| Document | What it is |
|---|---|
| [architecture/overview](frontend/architecture/overview.md) | Tech stack, folder structure, data flow |
| [login](frontend/features/login.md) | Auth flow, change-password |
| [password-recovery](frontend/features/password-recovery.md) | Forgot-password and reset screens, and the admin-issued link — **2.2.0, cut not deployed** |
| [dashboard](frontend/features/dashboard.md) | Role-based overview |
| [players](frontend/features/players.md) · [matches](frontend/features/matches.md) | The two main tables |
| [match-plans](frontend/features/match-plans.md) | The poll, the waitlist, guests — and creating a weekly run |
| [team-generation](frontend/features/team-generation.md) | Generation UI and balance-at-a-glance |
| [rankings](frontend/features/rankings.md) | League table and category cards |
| [motm-voting](frontend/features/motm-voting.md) · [badges](frontend/features/badges.md) | Crowd MOTM and achievements |
| [roles](frontend/features/roles.md) | Composable roles in the UI — `GROUP_ADMIN` is **not** a superset |
| [settings](frontend/features/settings.md) | The settings home |
| [push-notifications](frontend/features/push-notifications.md) · [notifications](frontend/features/notifications.md) | Web Push, and the in-app toast widget |
| [pwa](frontend/features/pwa.md) | Installability, service worker, offline |
| [privacy](frontend/features/privacy.md) | Export, deletion, public policy page |
| [language-switcher](frontend/features/language-switcher.md) · [theme](frontend/features/theme.md) | Locale and light/dark |
| [groups](frontend/features/groups.md) | Joining, founding, switching — and the two people who hand out access |
| [payments](frontend/features/payments.md) | Your balance, the interleaved ledger, the roster and delegation groups |
| [seasons](frontend/features/seasons.md) | Define, start and finalise, on the settings group tab — and why starting is not finalising |
| [api-tokens](frontend/features/api-tokens.md) | Minting, showing once and revoking the credential a shortcut carries — and why a per-group token is listed per account |
| *guests · payment-delegation* ⚠️ | **Missing.** Both shipped in the UI in `722335c` |

---

## Not here — and why

These stayed with their code because they change in the same commit as it does. Full reasoning in
[where-documents-live.md](where-documents-live.md).

| Document | Lives in |
|---|---|
| [API_REFERENCE](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API_REFERENCE.md) and the 15 per-surface contracts ⚠️ | [backend `docs/api/`](https://github.com/ricsnsuka/FootMania-Back/tree/main/docs/api) — *reference is missing Push and Payments sections; `TENANCY-API-CONTRACT` is the newest* |
| [FRONTEND_ENDPOINT_CHANGES](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/frontend/FRONTEND_ENDPOINT_CHANGES.md) · [draft SSE guide](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) | [backend `docs/frontend/`](https://github.com/ricsnsuka/FootMania-Back/tree/main/docs/frontend) |
| [Heroku deployment](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/deployment/HEROKU_DEPLOYMENT_GUIDE.md) | backend `docs/deployment/` |
| [Backend CHANGELOG](https://github.com/ricsnsuka/FootMania-Back/blob/main/CHANGELOG.md) ⚠️ | backend root — **twelve commits behind** |
| [copilot-instructions](https://github.com/ricsnsuka/FootMania-Back/blob/main/.github/copilot-instructions.md) · [the 14 agent definitions](https://github.com/ricsnsuka/FootMania-Back/tree/main/.github/agents) | backend `.github/` |
| [Postman collection](https://github.com/ricsnsuka/FootMania-Back/tree/main/postman) | backend `postman/` — generated from `/v3/api-docs` by `generate-collection.mjs`, 103 requests. Rerun the script after an API change |
| [Getting started](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/getting-started.md) · [component conventions](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/component-conventions.md) · [shared components](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/shared-components.md) · [styling](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/styling.md) · [i18n](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/i18n.md) · [testing](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/testing.md) · [Netlify](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/netlify-deployment.md) | [frontend `docs/guides/`](https://github.com/ricsnsuka/FootMania-Simple-Front/tree/main/docs/guides) |
| [AGENTS.md](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/AGENTS.md) — *"this is NOT the Next.js you know"* | frontend root |
