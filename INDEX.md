# Documentation Index

Every document in this repo. ⚠️ marks one with known drift — see
[STATUS.md](STATUS.md#known-documentation-drift).

---

## Curated (written for this repo)

| Document | What it is |
|---|---|
| [README](README.md) | What this repo is and the source-of-truth rule |
| [STATUS](STATUS.md) | Where the project stands, live hazards, the documentation drift register |
| [CONTRIBUTING](CONTRIBUTING.md) | How to keep this true, and the ship checklist |
| [product/roadmap](product/roadmap.md) | The five-phase improvement and mobile roadmap — the single copy |
| [product/feature-status](product/feature-status.md) | Every feature × backend × frontend × docs |
| [architecture/system-overview](architecture/system-overview.md) | How the two repos fit together; the conventions that bite |
| [architecture/database-migrations](architecture/database-migrations.md) | Complete V1–V19 history + deployment state |

---

## Backend

### API

| Document | What it is |
|---|---|
| [API_REFERENCE](backend/api/API_REFERENCE.md) ⚠️ | The whole API surface in one file. Missing Push and Payments sections |
| [player-API-CONTRACT](backend/api/player-API-CONTRACT.md) | Player management, incl. self-edit and PII gating |
| [PLAYER-LINK-ME-API-CONTRACT](backend/api/PLAYER-LINK-ME-API-CONTRACT.md) | Linking your own account to a player |
| [ROLES-API-CONTRACT](backend/api/ROLES-API-CONTRACT.md) | Composable roles (V18) |
| [ADMIN-API-CONTRACT](backend/api/ADMIN-API-CONTRACT.md) | Runtime settings and system endpoints |
| [PAYMENTS-API-CONTRACT](backend/api/PAYMENTS-API-CONTRACT.md) | The match fee ledger |
| [PUSH-API-CONTRACT](backend/api/PUSH-API-CONTRACT.md) | Subscriptions, per-category preferences, the seven categories |
| [MOTM-API-CONTRACT](backend/api/MOTM-API-CONTRACT.md) | Crowd Man-of-the-Match voting |
| [BADGES-API-CONTRACT](backend/api/BADGES-API-CONTRACT.md) | Achievement badges |
| [LEADERBOARDS-API-CONTRACT](backend/api/LEADERBOARDS-API-CONTRACT.md) | League table and category tops |
| [MATCH-RATING-RECALCULATION-API-CONTRACT](backend/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md) | Idempotent admin recalculation |
| [DRAFT-SESSION-ADMIN-API-CONTRACT](backend/api/DRAFT-SESSION-ADMIN-API-CONTRACT.md) | Admin control over a running draft |
| [DRAFT-SESSION-RESUME-API-CONTRACT](backend/api/DRAFT-SESSION-RESUME-API-CONTRACT.md) | SSE reconnect and resume |
| [PRIVACY-API-CONTRACT](backend/api/PRIVACY-API-CONTRACT.md) | GDPR export and erasure |

### Architecture & features

| Document | What it is |
|---|---|
| [ARCHITECTURE](backend/architecture/ARCHITECTURE.md) ⚠️ | Layering, packages, schema. Migration table stops at V13 |
| [ARCHITECTURE_DECISIONS](backend/next-release/ARCHITECTURE_DECISIONS.md) | ADR-001 optimistic locking · ADR-002 async completion · ADR-003 Caffeine · ADR-004 LLM vendor |
| [MICROSERVICES_ARCHITECTURE](backend/features/MICROSERVICES_ARCHITECTURE.md) | Why it is still one deployable |
| [CALCULATION_SERVICE](backend/features/CALCULATION_SERVICE.md) | The skill-rating engine, formula versions and worked examples |
| [MATCH_FEATURE](backend/features/MATCH_FEATURE.md) | Match lifecycle, completion, amendment |
| [MATCH_PLANS_FEATURE](backend/features/MATCH_PLANS_FEATURE.md) ⚠️ | Plans and the availability poll. **Stale since 2026-05-27** |
| [PLAYER_FEATURE](backend/features/PLAYER_FEATURE.md) | Roster, aggregates, soft delete |
| [USERS](backend/features/USERS.md) | Accounts, authentication, roles |
| [SEASON_FEATURE](backend/features/SEASON_FEATURE.md) | Seasons and the single-current invariant |
| [TEAM_GENERATION_FEATURE](backend/features/TEAM_GENERATION_FEATURE.md) | The six strategies |
| [TEAM_GENERATION_DESIGN](backend/features/TEAM_GENERATION_DESIGN.md) | How they were evaluated against each other |
| [DRAFT_SESSION_FEATURE](backend/features/DRAFT_SESSION_FEATURE.md) | Interactive captain pick |
| [PRIVACY_AND_DATA_PROTECTION](backend/features/PRIVACY_AND_DATA_PROTECTION.md) | What personal data exists, and both GDPR paths |
| [MATCH_TEAM_SPEC](backend/next-release/MATCH_TEAM_SPEC.md) | Match & team feature specification |

### Integration, operations, history

| Document | What it is |
|---|---|
| [DRAFT_SESSION_SSE_GUIDE](backend/frontend/DRAFT_SESSION_SSE_GUIDE.md) | How the frontend consumes the draft SSE stream |
| [FRONTEND_ENDPOINT_CHANGES](backend/frontend/FRONTEND_ENDPOINT_CHANGES.md) | Endpoint changes the frontend must react to |
| [HEROKU_DEPLOYMENT_GUIDE](backend/deployment/HEROKU_DEPLOYMENT_GUIDE.md) | Deploying the backend |
| [CHANGELOG](backend/CHANGELOG.md) ⚠️ | **Twelve commits behind** |
| [RELEASE_NOTES](backend/next-release/RELEASE_NOTES.md) | Accumulated notes for the next release |
| [INCIDENT 2026-05-26](backend/fixes/INCIDENT_2026-05-26_Java_Version_Mismatch.md) | Java 21 vs IntelliJ's 17 |
| [SECURITY-AUDIT 2025-07-09](backend/security/SECURITY-AUDIT-2025-07-09.md) · [2025-07-16](backend/security/SECURITY-AUDIT-2025-07-16.md) | Two audit passes |
| [PROJECT-README](backend/PROJECT-README.md) · [docs README](backend/README.md) | The backend's own front pages |

### Plans and handoffs

| Document | Status |
|---|---|
| [ROLE-MODEL-MIGRATION-PLAN](backend/plans/ROLE-MODEL-MIGRATION-PLAN.md) | ✅ complete, both repos |
| [MATCH-FEE-LEDGER-PLAN](backend/plans/MATCH-FEE-LEDGER-PLAN.md) | ✅ shipped — §14 covers a future payment integration |
| [PLAYER-LINK-ME-PLAN](backend/plans/PLAYER-LINK-ME-PLAN.md) | ✅ shipped |
| [PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM](backend/plans/PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM.md) | 🟡 all done except AI match reports |
| [PHASE0_FRONTEND_HANDOFF](backend/plans/PHASE0_FRONTEND_HANDOFF.md) | ✅ done |
| [REVIEW_HANDOFF_2026-07-27](backend/plans/REVIEW_HANDOFF_2026-07-27.md) | ✅ backlog cleared |
| [ORCHESTRATOR_SESSION](backend/plans/ORCHESTRATOR_SESSION.md) ⚠️ | Session log — **no entries since 2026-07-28** |

### Conventions

| Document | What it is |
|---|---|
| [CONVENTIONS-copilot-instructions](backend/CONVENTIONS-copilot-instructions.md) | The backend's coding and review conventions |
| [AGENT-PIPELINE](backend/AGENT-PIPELINE.md) | What the agent pipeline is. The 14 agent definitions stay in the code repo |

---

## Frontend

| Document | What it is |
|---|---|
| [PROJECT-README](frontend/PROJECT-README.md) | Front page — **push does not work under `npm run dev`** |
| [architecture/overview](frontend/architecture/overview.md) | Tech stack, folder structure, data flow |
| [INDEX](frontend/INDEX.md) ⚠️ | The frontend's own index — omits seven of its files |
| [CONVENTIONS-AGENTS](frontend/CONVENTIONS-AGENTS.md) | "This is NOT the Next.js you know" — read before writing Next.js code here |

### Features

| Document | What it is |
|---|---|
| [login](frontend/features/login.md) | Auth flow, change-password |
| [dashboard](frontend/features/dashboard.md) | Role-based overview |
| [players](frontend/features/players.md) · [matches](frontend/features/matches.md) | The two main tables |
| [team-generation](frontend/features/team-generation.md) | Generation UI and balance-at-a-glance |
| [rankings](frontend/features/rankings.md) | League table and category cards |
| [motm-voting](frontend/features/motm-voting.md) · [badges](frontend/features/badges.md) | Crowd MOTM and achievements |
| [roles](frontend/features/roles.md) | Composable roles in the UI — `ADMIN` is **not** a superset |
| [settings](frontend/features/settings.md) | The settings home |
| [push-notifications](frontend/features/push-notifications.md) · [notifications](frontend/features/notifications.md) | Web Push, and the in-app toast widget |
| [pwa](frontend/features/pwa.md) | Installability, service worker, offline |
| [privacy](frontend/features/privacy.md) | Export, deletion, public policy page |
| [language-switcher](frontend/features/language-switcher.md) · [theme](frontend/features/theme.md) | Locale and light/dark |
| *payments* ⚠️ | **Missing.** The ledger UI shipped in `a3efac0` with no feature doc |

### Guides

| Document | What it is |
|---|---|
| [getting-started](frontend/guides/getting-started.md) | Local setup |
| [component-conventions](frontend/guides/component-conventions.md) | How to write components here |
| [shared-components](frontend/guides/shared-components.md) | The shared primitives |
| [styling](frontend/guides/styling.md) | Semantic colour tokens and control surfaces |
| [i18n](frontend/guides/i18n.md) | en / pt / es — **every user-visible string needs all three** |
| [testing](frontend/guides/testing.md) | Unit and visual regression |
| [netlify-deployment](frontend/guides/netlify-deployment.md) | Deploying the frontend |
