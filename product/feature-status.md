# Feature Status

What exists, in which repo, and where it is written down. Commit refs are the change that
introduced the feature, not the last touch. Verified 2026-07-31.

Legend: ✅ shipped · 🟡 partial · ⬜ not started · — not applicable

---

## Core

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Players & roster | ✅ | ✅ | [PLAYER_FEATURE](../backend/features/PLAYER_FEATURE.md) · [player contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/player-API-CONTRACT.md) · [FE players](../frontend/features/players.md) |
| Accounts & auth (JWT) | ✅ | ✅ | [USERS](../backend/features/USERS.md) · [FE login](../frontend/features/login.md) |
| Account ↔ player self-link | ✅ | ✅ | [link-me contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PLAYER-LINK-ME-API-CONTRACT.md) · [plan](../backend/plans/PLAYER-LINK-ME-PLAN.md) |
| Matches — create, complete, live stats, amend | ✅ | ✅ | [MATCH_FEATURE](../backend/features/MATCH_FEATURE.md) · [FE matches](../frontend/features/matches.md) |
| Skill ratings & recalculation | ✅ | — | [CALCULATION_SERVICE](../backend/features/CALCULATION_SERVICE.md) · [recalc contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md) |
| Seasons | ✅ | 🟡 | [SEASON_FEATURE](../backend/features/SEASON_FEATURE.md) — season **write** API still deferred |
| Team generation (6 strategies) | ✅ | ✅ | [TEAM_GENERATION_FEATURE](../backend/features/TEAM_GENERATION_FEATURE.md) · [design](../backend/features/TEAM_GENERATION_DESIGN.md) · [FE](../frontend/features/team-generation.md) |
| Draft sessions (captain pick, SSE) | ✅ | ✅ | [DRAFT_SESSION_FEATURE](../backend/features/DRAFT_SESSION_FEATURE.md) · [SSE guide](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) |
| Match plans — RSVP, deadline, waitlist | ✅ `57844eb` | ✅ `3e8aa79` | [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md) ⚠️ **stale since 2026-05-27** |
| Kickoff time + plan lifecycle (V17) | ✅ `4120ad1` | ✅ `eff49bf` | ⚠️ only in the migration and the changelog gap |

## Roadmap Phase 0–3

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Phase 0 — branding, season gap, GDPR posture | ✅ | ✅ | [PHASE0 handoff](../backend/plans/PHASE0_FRONTEND_HANDOFF.md) |
| Privacy / GDPR export & erasure | ✅ | ✅ `ce9ab08` | [PRIVACY_AND_DATA_PROTECTION](../backend/features/PRIVACY_AND_DATA_PROTECTION.md) · [contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PRIVACY-API-CONTRACT.md) · [FE](../frontend/features/privacy.md) |
| Phase 1 — PWA (installable, offline, SW) | — | ✅ | [FE pwa](../frontend/features/pwa.md) |
| Phase 2 — Web Push (VAPID, 7 categories) | ✅ `80e9b5f`, `aaff369` | ✅ | [PUSH contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PUSH-API-CONTRACT.md) · [FE push](../frontend/features/push-notifications.md) |
| Balance at a glance | — | ✅ `f8798cc` | [FE team-generation](../frontend/features/team-generation.md) |
| Leaderboards & rankings | ✅ `2e3018e` | ✅ `6d27431` | [LEADERBOARDS contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/LEADERBOARDS-API-CONTRACT.md) · [FE rankings](../frontend/features/rankings.md) |
| Crowd MOTM voting (V14) | ✅ `cf3240e` | ✅ `139e0ed` | [MOTM contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/MOTM-API-CONTRACT.md) · [FE motm-voting](../frontend/features/motm-voting.md) |
| Achievement badges (V15) | ✅ `45fff7e` | ✅ `1af7973` | [BADGES contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/BADGES-API-CONTRACT.md) · [FE badges](../frontend/features/badges.md) |
| **AI match reports** | ⬜ | ⬜ | ADR-004 picks the vendor; nothing built. **The last open Phase 3 item** |

## Built alongside the roadmap

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Runtime-configurable competition rules (V16) | ✅ `c4d6312` | ✅ | [ADMIN contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/ADMIN-API-CONTRACT.md) · [FE settings](../frontend/features/settings.md) |
| Admin settings & system endpoints | ✅ `c7d3554` | ✅ `2b961a3` | [ADMIN contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/ADMIN-API-CONTRACT.md) |
| Composable roles (V18) | ✅ `d6b908f` | ✅ `418256e` | [ROLES contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/ROLES-API-CONTRACT.md) · [migration plan](../backend/plans/ROLE-MODEL-MIGRATION-PLAN.md) · [FE roles](../frontend/features/roles.md) |
| Match fee ledger (V19) | ✅ `828db3b` | ✅ `a3efac0` | [PAYMENTS contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PAYMENTS-API-CONTRACT.md) · [ledger plan](../backend/plans/MATCH-FEE-LEDGER-PLAN.md) · ⚠️ **no FE feature doc** |
| Payment delegation (V20) | ✅ `d3b3339` | ✅ `722335c` | [PAYMENTS contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/PAYMENTS-API-CONTRACT.md) · [plan](../backend/plans/PAYMENT-DELEGATION-PLAN.md) · deployed 2026-08-01 |
| Guest players (V21) | ✅ `d3b3339`, `2eac528` + fix `3159812` | ✅ `722335c` + fix `c041898` | [GUEST-PLAYERS contract](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/GUEST-PLAYERS-API-CONTRACT.md) · [plan](../backend/plans/GUEST-PLAYERS-PLAN.md) §12b — guest removal broke in production on day one, fixed same day · ⚠️ `GuestIsolationIT` still never executed |
| Pitch cost on `MatchPlanDTO` | ✅ `e87d624` | ✅ `4f47bf5` | ⚠️ **documented nowhere** |
| i18n — en / pt / es | — | ✅ | [FE i18n guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/v1.0.0/docs/guides/i18n.md) · [language switcher](../frontend/features/language-switcher.md) |
| Theme (light / dark) | — | ✅ | [FE theme](../frontend/features/theme.md) |

## Not started

| Feature | State |
|---|---|
| Phase 4 — Capacitor wrap, app stores | ⬜ Gated on real PWA install data. Needs Apple ($99/yr) + Play ($25) accounts and a public privacy policy page |
| Phase 5 — multi-tenancy + billing | 📝 **Specced 2026-08-01** as a five-plan dark-launch chain: [schema](../backend/plans/TENANCY-SCHEMA-PLAN.md) → [enforcement](../backend/plans/TENANCY-ENFORCEMENT-PLAN.md) → [privacy](../backend/plans/TENANT-PRIVACY-PLAN.md) → [onboarding](../backend/plans/GROUP-ONBOARDING-PLAN.md) → [billing](../backend/plans/GROUP-BILLING-PLAN.md), + [architecture/multi-tenancy](../architecture/multi-tenancy.md). Owner decisions locked: multi-membership, picker addressing, code-gated creation, tenancy before billing. Pre-work: `integrationTest` into CI |
| Tiered invitations (core vs extended) | ⬜ Overlaps Phase 5 — the roadmap warns against over-building it first |
| **Real in-app payments** | ⬜ **Deliberately out of scope.** The ledger records who owes what; money moves over MB WAY between phones. Collecting money makes somebody a merchant of record with PSP, KYC and PSD2 obligations |

---

## Where the gaps are

Three shipped features have no or stale documentation — plan lifecycle, the frontend ledger UI, and
`totalCostCents`. All three are in the drift table in [../STATUS.md](../STATUS.md), which is the
list to work from.
