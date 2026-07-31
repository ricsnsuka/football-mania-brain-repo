# Football Management System

A **Football Management System** REST API built with **Java 21** and **Spring Boot 3.4.5**.

The web client lives in a **separate repository** (`FootballMania/front/football`, Next.js 16) —
this is not a monorepo, so an API change and its frontend change are two commits in two places.
Every API-touching change here ships with its contract doc updated in the same commit.

## What it does

| Area | Summary |
|------|---------|
| **Players & users** | Roster, accounts, three roles, self-service linking of an account to a player |
| **Matches** | Creation, completion, live stat updates, admin amendment, rating recalculation |
| **Match plans** | Proposed fixtures with RSVP, a confirmation deadline and waitlisting |
| **Team generation** | Balanced auto-generation, plus captain-draft sessions |
| **Ratings** | Skill rating recalculated per match, with full history |
| **Rankings & leaderboards** | League table with a qualification threshold; category tops |
| **Crowd MOTM** | The players who appeared vote for one of their own, resolved on a schedule |
| **Achievement badges** | Milestones derived from existing aggregates, awarded on completion |
| **Push notifications** | Web Push (VAPID), per-category preferences, reminder scheduling |
| **Privacy / GDPR** | Data export and erasure, acting on the caller only |
| **Admin settings** | Competition rules configurable at runtime, system health, maintenance actions |

## ⚡ Why Java 21?

| Feature | Benefit |
|---------|---------|
| **Virtual Threads** | Handles thousands of concurrent requests without thread-pool exhaustion |
| **Records** for DTOs | Immutable, GC-friendly, zero-boilerplate data containers |
| **Sealed classes** | Exhaustive domain modelling, no impossible states |
| **Pattern matching** | Cleaner switch expressions across service layer |
| **ZGC / Generational ZGC** | Sub-millisecond GC pauses in production |

## 🚀 Quick Start

```bash
# 1. Start PostgreSQL
docker-compose up -d postgres

# 2. Run the app
./gradlew bootRun --args='--spring.profiles.active=dev'

# 3. Open Swagger UI
open http://localhost:8080/swagger-ui.html
```

## 🧪 Run Tests

```bash
./gradlew test          # all 835

# CI runs these four in parallel instead:
./gradlew testControllers testServices testSecurity testApplication
```

> **The four CI tasks must together cover everything `test` runs.** They filter by *package*, so a
> test in a new package runs locally and is **silently skipped in CI**. After adding a package,
> check the totals still sum — currently 296 + 535 + 3 + 1 = 835. `build.gradle` documents this
> next to the task definitions.

## 🏗️ Architecture

```
src/main/java/pt/rics/demo/football/
  FootballApplication.java    ← Entry point, Virtual Threads enabled
  config/                     ← Spring configuration (Security, Cache, OpenAPI)
  controller/                 ← REST controllers
  service/                    ← Business logic (concrete classes, no thin interfaces)
  repository/                 ← Spring Data JPA repositories
  model/                      ← JPA entities (@Entity)
  dto/                        ← Java Records (immutable, no Lombok needed)
  mapper/                     ← MapStruct compile-time mappers
  exception/                  ← Exceptions + GlobalExceptionHandler
  security/                   ← JWT filter, token provider, UserPrincipal record
```

## 🗄️ Database

- **Production**: PostgreSQL 17
- **Test**: H2 in-memory (PostgreSQL compatibility mode)
- **Migrations**: Flyway (`src/main/resources/db/migration/`), currently through **V16**

`V16__app_settings.sql` is worth knowing about: it is created **empty** and holds a row only where
an admin has overridden a default. Defaults live in the `AppSetting` enum, so changing one is an
ordinary code change, adding a setting needs no migration, and deleting a row is the reset path.

## ⚙️ Runtime-configurable rules

Four values that used to be Java constants, editable by an admin via `PATCH /api/admin/settings`:

| Setting | Default | Range |
|---------|---------|-------|
| `MVP_VOTING_WINDOW_HOURS` | 24 | 1–168 |
| `RANKING_MINIMUM_MATCHES` | 3 | 1–50 |
| `LEADERBOARD_DEFAULT_LIMIT` | 5 | 1–100 |
| `LEADERBOARD_MAX_LIMIT` | 25 | 1–100 |

Clients must read these back from responses (`minimumMatchesToQualify`, `mvpVotingClosesAt`,
`limit`) rather than hard-coding them. The MOTM window is stamped onto a match at completion, so
changing it **never** affects a poll that is already open.

## 📚 API docs

`docs/api/API_REFERENCE.md` is the full endpoint list. Per-feature contracts sit alongside it —
[admin](docs/api/ADMIN-API-CONTRACT.md), [leaderboards](docs/api/LEADERBOARDS-API-CONTRACT.md),
[MOTM](docs/api/MOTM-API-CONTRACT.md), [badges](docs/api/BADGES-API-CONTRACT.md),
[push](docs/api/PUSH-API-CONTRACT.md), [privacy](docs/api/PRIVACY-API-CONTRACT.md) and others.

> ⚠️ **Nullable fields are omitted, not sent as `null`.** `spring.jackson.default-property-inclusion:
> non_null` means a field with no value is *absent from the JSON*. Clients receive `undefined`, and
> `x === null` is false for all of them. In tests, `jsonPath().doesNotExist()` passes for both absent
> and present-but-null — use `doesNotHaveJsonPath()` when you mean absent.

## 🐳 Docker

```bash
# Dev (PostgreSQL only)
docker-compose up -d postgres

# Full stack (Postgres + Keycloak)
docker-compose --profile keycloak up -d
```

## 🔐 Authentication

Two modes controlled by Spring profile:

| Profile | Mode | Description |
|---------|------|-------------|
| (default) | HMAC JWT | `app.jwt.secret` in `application.yml` |
| `keycloak` | OAuth2 Resource Server | Keycloak at `http://localhost:8180` |

## 🧰 Agent Ecosystem

Development tasks are handled by specialized AI agents in `.github/agents/`.
Start with the **orchestrator** for any task:

```
.github/agents/orchestrator.agent.md  ← Master entry point
```

## 🪤 Things that have caught people out

- **`@Transactional` and `@Async` are proxy-based**, so a same-bean call silently drops the
  annotation. That is why `ReminderScheduler`/`ReminderDispatcher`,
  `MvpResolutionScheduler`/`MvpResolutionDispatcher` and `BadgeBackfillService`/`BadgeService` are
  separate beans. Do not merge them back together.
- **Idempotency is enforced by UNIQUE constraints**, not by checking first — checking then inserting
  races. `uq_match_mvp_votes_voter` and `uq_player_badges` are load-bearing, because badge
  evaluation is *replayed* routinely by bulk rating recalculation.
- **Multi-instance work claims before acting** via a conditional update
  (`SET x = :now WHERE id = ? AND x IS NULL`), so two nodes cannot both process the same row.
- **Caffeine is per-node.** A write evicts the local cache only; another pod serves its copy until
  the 10-minute TTL. Accepted trade-off, documented in `CacheConfig`.

## 🔔 Push notifications

VAPID keys are supplied via environment variables; `VapidKeyGenerator` will mint a pair.

> The `aud` claim is written with `.claim("aud", ...)` rather than JJWT's `audience().add(...)` on
> purpose. JJWT models `aud` as a collection and serialises even a single value as a one-element
> array — legal per RFC 7519, but FCM answers `403 permission denied: invalid aud claim` and every
> notification fails. The comment in `VapidSigner` says so; please leave it there.

## 📋 Version

**Current:** `1.0.0` — Java 21 rewrite (May 2026)
