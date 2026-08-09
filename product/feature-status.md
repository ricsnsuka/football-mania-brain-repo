# Feature Status

What exists, in which repo, and where it is written down. Commit refs are the change that
introduced the feature, not the last touch. Verified 2026-07-31.

Legend: ✅ shipped · 🟡 partial · ⬜ not started · — not applicable

---

## Core

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Players & roster | ✅ | ✅ | [PLAYER_FEATURE](../backend/features/PLAYER_FEATURE.md) · [player contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/player-API-CONTRACT.md) · [FE players](../frontend/features/players.md) |
| Accounts & auth (JWT) | ✅ | ✅ | [USERS](../backend/features/USERS.md) · [FE login](../frontend/features/login.md) |
| Account ↔ player self-link | ✅ | ✅ | [link-me contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PLAYER-LINK-ME-API-CONTRACT.md) · [plan](../backend/plans/PLAYER-LINK-ME-PLAN.md) |
| Matches — create, complete, live stats, amend | ✅ | ✅ | [MATCH_FEATURE](../backend/features/MATCH_FEATURE.md) · [FE matches](../frontend/features/matches.md) |
| Skill ratings & recalculation | ✅ | — | [CALCULATION_SERVICE](../backend/features/CALCULATION_SERVICE.md) · [recalc contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md) |
| Seasons — model, seed, `seasonId` filter | ✅ | ✅ | [SEASON_FEATURE](../backend/features/SEASON_FEATURE.md) |
| Seasons — define / start / finalise (admin) | ✅ 2026-08-07 | ✅ 2026-08-07 | [SEASONS contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md) · [FE seasons](../frontend/features/seasons.md). Four endpoints, **no migration** — every field is an existing column or a count over `matches.season_id`. The admin surface is a section of the settings group tab, not a page. `seasons.end_date` is written for the first time in any version |
| Season-scoped stats — rankings, dashboard, matches, player modal | ✅ 2026-08-08 | ✅ 2026-08-08 | [rankings contract §season-bounded](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/LEADERBOARDS-API-CONTRACT.md) · [FE rankings](../frontend/features/rankings.md#the-season-scope) · [FE seasons](../frontend/features/seasons.md#what-having-seasons-changed-elsewhere). Streaks stay **all-time in every scope**, by design |
| Seasonal awards — POTS, Golden Boot, Playmaker, Iron Man, Most improved, Crowd favourite | ✅ 2026-08-08 **V37** | ✅ 2026-08-08 | [SEASONS contract §awards](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md#get-apiseasonsidawards) · [FE seasons §honours](../frontend/features/seasons.md#the-honours-board) · [V37](../architecture/database-migrations.md). Computed at finalise and **stored**, ties get a row each, and only the two rating awards carry an appearance threshold |
| Ballon d'Or — season-change poll | ✅ 2026-08-08 **V38** | ✅ 2026-08-08 | [BALLON-DOR contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/BALLON-DOR-API-CONTRACT.md) · [FE seasons §Ballon d'Or](../frontend/features/seasons.md#the-ballon-dor) · [V38](../architecture/database-migrations.md). Window measured in **matches, not days**; no running tally is ever sent; a tie writes no award, and nobody voting is not a tie |
| Team generation (7 strategies) | ✅ | ✅ | [TEAM_GENERATION_FEATURE](../backend/features/TEAM_GENERATION_FEATURE.md) · [design](../backend/features/TEAM_GENERATION_DESIGN.md) · [OPTIMAL plan](../backend/plans/OPTIMAL-PARTITION-PLAN.md) · [FE](../frontend/features/team-generation.md) |
| Player positions & keeper willingness | ✅ V36 | ✅ | [PLAYER_FEATURE](../backend/features/PLAYER_FEATURE.md) · [FE players](../frontend/features/players.md) |
| Draft sessions (captain pick, SSE) | ✅ | ✅ | [DRAFT_SESSION_FEATURE](../backend/features/DRAFT_SESSION_FEATURE.md) · [SSE guide](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/frontend/DRAFT_SESSION_SSE_GUIDE.md) |
| Match plans — RSVP, deadline, waitlist | ✅ `57844eb` | ✅ `3e8aa79` | [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md) — rewritten against the code 2026-08-05 · [FE match-plans](../frontend/features/match-plans.md) |
| Kickoff time + plan lifecycle (V17) | ✅ `4120ad1` | ✅ `eff49bf` | [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md) — the instant kickoff, `GENERATED`, and the derived `expired`/`generatable`/`cancellable` flags |
| Match editing — details, scoresheet, lineup (1.4.0) | ✅ `e69e604` | ✅ `b55274f` | [lineup contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MATCH-LINEUP-API-CONTRACT.md) · [MATCH_FEATURE](../backend/features/MATCH_FEATURE.md) · [FE matches](../frontend/features/matches.md) |
| Delete a match, unwinding it (1.4.1) | ✅ `aa86d24` | ✅ `0f226d3` | [deletion contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MATCH-DELETION-API-CONTRACT.md) · [MATCH_FEATURE §8](../backend/features/MATCH_FEATURE.md) · [FE matches](../frontend/features/matches.md) |
| Weekly recurring match plans (V34 horizon) | ✅ `5aff478` | ✅ `4dec174` | [recurring contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/RECURRING-MATCH-PLANS-API-CONTRACT.md) · [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md#weekly-recurring-runs) · [FE match-plans](../frontend/features/match-plans.md) |
| Match plans filtered to one week (Mon–Sun) | ✅ | ✅ | [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md#from--to--an-explicit-kickoff-window) · [endpoint changelog](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/frontend/FRONTEND_ENDPOINT_CHANGES.md) · [FE match-plans](../frontend/features/match-plans.md#the-week-filter) |
| Scoped API tokens (V39) | ✅ 1.8.0 | ✅ 1.8.0 | [plan](../backend/plans/SCOPED-API-TOKENS-PLAN.md) · [contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API-TOKENS-API-CONTRACT.md). A revocable, group-bound personal token — the prerequisite for **any** automation route, including a watch shortcut. Shipped in **1.8.0**, 2026-08-09 (`FootMania-Back#197`, `FootMania-Simple-Front#64`). The V1–V39 chain has been applied and validated against a real PostgreSQL 16. ⚠️ The backend *deploy* was not read back from the platform — see [STATUS.md](../STATUS.md) |
| Paged rankings, ledger and matches lists | — | ✅ | [FE rankings](../frontend/features/rankings.md#paging) · [FE payments](../frontend/features/payments.md) · [FE matches](../frontend/features/matches.md#page-size-selector) |

## Roadmap Phase 0–3

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Phase 0 — branding, season gap, GDPR posture | ✅ | ✅ | [PHASE0 handoff](../backend/plans/PHASE0_FRONTEND_HANDOFF.md) |
| Privacy / GDPR export & erasure | ✅ | ✅ `ce9ab08` | [PRIVACY_AND_DATA_PROTECTION](../backend/features/PRIVACY_AND_DATA_PROTECTION.md) · [contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PRIVACY-API-CONTRACT.md) · [FE](../frontend/features/privacy.md) |
| Phase 1 — PWA (installable, offline, SW) | — | ✅ | [FE pwa](../frontend/features/pwa.md) |
| Phase 2 — Web Push (VAPID, 7 categories) | ✅ `80e9b5f`, `aaff369` | ✅ | [PUSH contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PUSH-API-CONTRACT.md) · [FE push](../frontend/features/push-notifications.md) |
| Balance at a glance | — | ✅ `f8798cc` | [FE team-generation](../frontend/features/team-generation.md) |
| Leaderboards & rankings | ✅ `2e3018e` | ✅ `6d27431` | [LEADERBOARDS contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/LEADERBOARDS-API-CONTRACT.md) · [FE rankings](../frontend/features/rankings.md) |
| Crowd MOTM voting (V14) | ✅ `cf3240e` | ✅ `139e0ed` | [MOTM contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MOTM-API-CONTRACT.md) · [FE motm-voting](../frontend/features/motm-voting.md) |
| Achievement badges (V15) | ✅ `45fff7e` | ✅ `1af7973` | [BADGES contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/BADGES-API-CONTRACT.md) · [FE badges](../frontend/features/badges.md) |
| **AI match reports** | ⬜ | ⬜ | ADR-004 picks the vendor; nothing built. **The last open Phase 3 item** |

## Built alongside the roadmap

| Feature | Backend | Frontend | Documentation |
|---|---|---|---|
| Runtime-configurable competition rules (V16) | ✅ `c4d6312` | ✅ | [GROUP_ADMIN contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/ADMIN-API-CONTRACT.md) · [FE settings](../frontend/features/settings.md) |
| Admin settings & system endpoints | ✅ `c7d3554` | ✅ `2b961a3` | [GROUP_ADMIN contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/ADMIN-API-CONTRACT.md) |
| Composable roles (V18) | ✅ `d6b908f` | ✅ `418256e` | [ROLES contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/ROLES-API-CONTRACT.md) · [migration plan](../backend/plans/ROLE-MODEL-MIGRATION-PLAN.md) · [FE roles](../frontend/features/roles.md) |
| Match fee ledger (V19) | ✅ `828db3b` | ✅ `a3efac0` | [PAYMENTS contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PAYMENTS-API-CONTRACT.md) · [ledger plan](../backend/plans/MATCH-FEE-LEDGER-PLAN.md) · [FE payments](../frontend/features/payments.md) — written 2026-08-05 |
| Payment delegation (V20) | ✅ `d3b3339` | ✅ `722335c` | [PAYMENTS contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PAYMENTS-API-CONTRACT.md) · [plan](../backend/plans/PAYMENT-DELEGATION-PLAN.md) · deployed 2026-08-01 |
| Guest players (V21) | ✅ `d3b3339`, `2eac528` + fix `3159812` | ✅ `722335c` + fix `c041898` | [GUEST-PLAYERS contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/GUEST-PLAYERS-API-CONTRACT.md) · [plan](../backend/plans/GUEST-PLAYERS-PLAN.md) §12b — guest removal broke in production on day one, fixed same day · ⚠️ `GuestIsolationIT` still never executed |
| Pitch cost on `MatchPlanDTO` | ✅ `e87d624` | ✅ `4f47bf5` | [MATCH_PLANS_FEATURE](../backend/features/MATCH_PLANS_FEATURE.md) — the column, and that absent ≠ zero. Still missing from `API_REFERENCE.md` |
| Platform settings, operator-only (V34) | ✅ `5aff478` | ✅ | [platform settings contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PLATFORM-SETTINGS-API-CONTRACT.md) · [migrations](../architecture/database-migrations.md) |
| i18n — en / pt / es | — | ✅ | [FE i18n guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/i18n.md) · [language switcher](../frontend/features/language-switcher.md) |
| Theme (light / dark) | — | ✅ | [FE theme](../frontend/features/theme.md) |

## Not started

| Feature | State |
|---|---|
| Phase 4 — Capacitor wrap, app stores | ⬜ Gated on real PWA install data. Needs Apple ($99/yr) + Play ($25) accounts and a public privacy policy page |
| Phase 5 — multi-tenancy + billing | 🟡 **Two rungs built.** [schema](../backend/plans/TENANCY-SCHEMA-PLAN.md) ✅ 2026-08-01 (V22–V27) → [enforcement](../backend/plans/TENANCY-ENFORCEMENT-PLAN.md) ✅ 2026-08-02 (`X-Group-Id`, predicates, async payloads, SSE ownership, V28 platform grant) → [privacy](../backend/plans/TENANT-PRIVACY-PLAN.md) 📝 → [onboarding](../backend/plans/GROUP-ONBOARDING-PLAN.md) 📝 **the visibility flip, owner-gated** → [billing](../backend/plans/GROUP-BILLING-PLAN.md) ⛔ **ON HOLD (owner)**. Both built rungs are **deployed and running dark** as of 2026-08-02 — one organization, an optional header, no behaviour change. Owner decisions locked: multi-membership, picker addressing, code-gated creation, tenancy before billing |
| Tiered invitations (core vs extended) | ⬜ Overlaps Phase 5 — the roadmap warns against over-building it first |
| **Real in-app payments** | ⬜ **Deliberately out of scope.** The ledger records who owes what; money moves over MB WAY between phones. Collecting money makes somebody a merchant of record with PSP, KYC and PSD2 obligations |

---

## Where the gaps are

Two of the three gaps that stood here on 2026-07-31 are closed: the plan lifecycle and
`totalCostCents` are now written down, in a MATCH_PLANS_FEATURE rewritten against the code on
2026-08-05. **The frontend ledger UI still has no feature doc** — `a3efac0` shipped the whole "what
you owe" screen and nothing describes it. That, and whatever else is open, is in the drift table in
[../STATUS.md](../STATUS.md), which is the list to work from.
