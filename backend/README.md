# Documentation — Football Management System

**Version:** 1.0.0  
**Last Updated:** May 22, 2026

---

## 📁 Structure

```
docs/
  api/
    API_REFERENCE.md                  ← Complete endpoint catalog (all resources)
    player-API-CONTRACT.md            ← Player endpoint contract & DTO reference
  architecture/
    ARCHITECTURE.md                   ← System architecture, stack, package structure, dev guide
  features/
    USERS.md                          ← User entity, auth login flow, roles, DTOs, examples
    PLAYER_FEATURE.md                 ← Player entity, skill rating, streaks, user link, status
    MATCH_FEATURE.md                  ← Match lifecycle, teams, player stats, completeMatch flow
    MATCH_PLANS_FEATURE.md            ← Match plans, availability poll, team generation flow
    CALCULATION_SERVICE.md            ← Skill rating engine — all formulas, constants, season-end
    SEASON_FEATURE.md                 ← Season model, match association, season-end transition
    TEAM_GENERATION_DESIGN.md         ← Team generation algorithm design decisions (Phase 1–3)
    MICROSERVICES_ARCHITECTURE.md     ← Microservices migration reference (educational)
  frontend/
    FRONTEND_ENDPOINT_CHANGES.md      ← Cumulative FE changelog (append-only)
  next-release/
    RELEASE_NOTES.md                  ← Accumulated notes for the next release
    ARCHITECTURE_DECISIONS.md         ← Key architectural decisions log
    MATCH_TEAM_SPEC.md                ← Match & team domain specification
  plans/
    ORCHESTRATOR_SESSION.md           ← Orchestration session log
  security/
    SECURITY-AUDIT-2025-07-09.md      ← Security audit report (CVE findings, auth review)
```

---

## 🚀 Quick Links

| Document | Description |
|---|---|
| [Architecture Overview](architecture/ARCHITECTURE.md) | Full system architecture, tech stack, package layout, dev guide |
| [API Reference](api/API_REFERENCE.md) | All endpoints, request/response shapes, auth levels |
| [Player API Contract](api/player-API-CONTRACT.md) | Player DTO schemas and endpoint contracts |
| [Frontend Changes](frontend/FRONTEND_ENDPOINT_CHANGES.md) | Cumulative changelog for frontend integration |
| [Feature: Users](features/USERS.md) | User entity — auth, roles, DTOs, login flow |
| [Feature: Players](features/PLAYER_FEATURE.md) | Player entity — skill rating, streaks, DTOs |
| [Feature: Matches](features/MATCH_FEATURE.md) | Match lifecycle, teams, stats, completion, live updates |
| [Feature: Match Plans](features/MATCH_PLANS_FEATURE.md) | Pre-match planning, availability poll, team generation |
| [Feature: Calculation Service](features/CALCULATION_SERVICE.md) | Skill rating formulas, constants, season-end transition |
| [Feature: Seasons](features/SEASON_FEATURE.md) | Season model, match association, current limitations |
| [Team Generation Design](features/TEAM_GENERATION_DESIGN.md) | Algorithm evaluation & Phase 1 implementation |
| [Microservices Architecture](features/MICROSERVICES_ARCHITECTURE.md) | Bounded contexts & migration path (educational) |
| [Release Notes](next-release/RELEASE_NOTES.md) | What's going into the next release |
| [Architecture Decisions](next-release/ARCHITECTURE_DECISIONS.md) | ADRs — virtual threads, caching, security choices |
| [Match & Team Spec](next-release/MATCH_TEAM_SPEC.md) | Match domain specification |
| [Session Log](plans/ORCHESTRATOR_SESSION.md) | Orchestration session log |
| [Security Audit](security/SECURITY-AUDIT-2025-07-09.md) | CVE scan results and auth review |

---

## 📊 Implementation Progress

| Entity / Feature        | Status      | Docs                                  | Notes                                        |
|-------------------------|-------------|---------------------------------------|----------------------------------------------|
| Users                   | ✅ Done      | [USERS.md](features/USERS.md)        | CRUD, roles, password change, JWT auth       |
| Authentication          | ✅ Done      | [USERS.md](features/USERS.md)        | Login via username or email                  |
| Players                 | ✅ Done      | [PLAYER_FEATURE.md](features/PLAYER_FEATURE.md) | CRUD, skill rating, streaks, user link |
| Matches                 | ✅ Done      | [MATCH_FEATURE.md](features/MATCH_FEATURE.md) | Create, complete, score, team stats   |
| Player Stats            | ✅ Done      | [MATCH_FEATURE.md](features/MATCH_FEATURE.md) | Per-match stats (goals, assists, MVP) |
| Live Stats Update       | ✅ Done      | [MATCH_FEATURE.md](features/MATCH_FEATURE.md) | Preview ratings during match          |
| Match Plans             | ✅ Done      | [MATCH_PLANS_FEATURE.md](features/MATCH_PLANS_FEATURE.md) | Poll, confirmations, preview |
| Team Generation         | ✅ Done      | [MATCH_PLANS_FEATURE.md](features/MATCH_PLANS_FEATURE.md) | BALANCED, RANDOM, SNAKE_DRAFT |
| Skill Rating Engine     | ✅ Done      | [CALCULATION_SERVICE.md](features/CALCULATION_SERVICE.md) | Match rating + skill update |
| Season-End Transition   | ✅ Done      | [CALCULATION_SERVICE.md](features/CALCULATION_SERVICE.md) | Weighted formula, ±2.0 max |
| Seasons Model           | ✅ Done      | [SEASON_FEATURE.md](features/SEASON_FEATURE.md) | DB model, seed, current limitations |
| Rankings / Leaderboards | 🔲 Pending  | —                                     | Cache keys defined; endpoints TBD           |
| Season CRUD API         | 🔲 Pending  | [SEASON_FEATURE.md](features/SEASON_FEATURE.md) | No `/api/seasons` endpoint yet    |
