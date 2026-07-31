# System Architecture — Football Management System

**Version:** 1.0.0  
**Date:** May 2026

---

## Overview

The Football Management System is a **monolithic REST API** built on Java 21 and
Spring Boot 3.4.5. It deliberately avoids microservices complexity — the domain is
small and consistent, all data lives in a single PostgreSQL database, and the team
is small. The design leverages Java 21's Virtual Threads for I/O concurrency,
Caffeine for local caching, and MapStruct for zero-reflection DTO mapping.

---

## Technology Stack

| Concern         | Technology                   | Version   | Notes                                          |
|-----------------|------------------------------|-----------|------------------------------------------------|
| Language        | Java                         | 21        | Records, virtual threads, pattern matching, ZGC|
| Framework       | Spring Boot                  | 3.4.5     | WebMVC with virtual thread executor            |
| Database        | PostgreSQL (prod), H2 (test) | 17+ / 2.x | Flyway migrations                              |
| Caching         | Caffeine                     | 3.2.0     | Local in-process cache — no Redis              |
| Security        | Spring Security + JJWT       | 6.x / 0.12.6 | HMAC-SHA256 JWT (default) or Keycloak profile |
| Mapping         | MapStruct                    | 1.6.3     | Compile-time, no runtime reflection            |
| Boilerplate     | Lombok                       | 1.18.38   | Entities & configs only; DTOs use Records      |
| API Docs        | SpringDoc OpenAPI            | 2.8.6     | Swagger UI at `/swagger-ui.html`               |
| Testing         | JUnit 5 · Testcontainers · H2 | —        | H2 (unit/controller), PostgreSQL (E2E)         |
| Build           | Gradle                       | 8          | Foojay toolchain resolver (auto-downloads JDK) |

---

## Package Structure

```
src/main/java/pt/rics/demo/football/
├── FootballApplication.java          # @SpringBootApplication, @EnableCaching, @EnableAsync
├── config/
│   ├── CacheConfig.java              # Caffeine cache definitions (10 min TTL, 500 max entries)
│   ├── OpenApiConfig.java            # Swagger/OpenAPI config
│   └── SecurityConfig.java          # @Profile("!keycloak") — HMAC JWT filter chain
├── controller/
│   ├── AuthController.java           # POST /api/auth/login
│   ├── HealthController.java         # GET /api/health
│   ├── MatchController.java          # /api/matches/**
│   ├── MatchPlanController.java      # /api/match-plans/**
│   ├── PlayerController.java         # /api/players/**
│   ├── UserController.java           # /api/users/**
│   └── VersionController.java        # GET /api/version
├── dto/                              # Java Records ONLY — immutable, zero Lombok
├── event/
│   └── MatchCompletedEvent.java      # Spring ApplicationEvent for post-match rating
├── exception/
│   ├── ResourceNotFoundException.java  # factory: .of("Player", id)
│   ├── BusinessException.java          # factory: .conflict(), .badRequest(), etc.
│   └── GlobalExceptionHandler.java     # @RestControllerAdvice — ApiError record
├── mapper/                           # MapStruct @Mapper interfaces
│   ├── MatchMapper.java
│   ├── MatchPlanMapper.java
│   ├── PlayerMapper.java
│   └── UserMapper.java
├── model/                            # JPA entities — Lombok @Getter/@Setter
│   ├── AppUser.java
│   ├── Match.java + MatchTeam.java
│   ├── MatchPlan.java + PlayerConfirmation.java
│   ├── Player.java
│   ├── PlayerStat.java + Goal.java
│   ├── Season.java + SkillRatingHistory.java
│   └── Enums: MatchType, GenerationType, MatchResult, ConfirmationStatus, PlanStatus
├── repository/                       # Spring Data JPA interfaces
├── security/
│   ├── JwtTokenProvider.java         # HMAC-SHA256 sign/validate
│   ├── JwtAuthenticationFilter.java  # @Profile("!keycloak") — Bearer token extraction
│   └── UserPrincipal.java            # Java Record implementing UserDetails
└── service/
    ├── AuthService.java
    ├── CalculationService.java       # Core rating engine — sole source of ratings
    ├── MatchEventListener.java       # Listens to MatchCompletedEvent
    ├── MatchPlanService.java
    ├── MatchService.java
    ├── PlayerService.java
    ├── UserDetailsServiceImpl.java
    ├── UserService.java
    └── teamgeneration/
        ├── TeamGenerationStrategy.java         (interface)
        ├── TeamGenerationContext.java          (record — input)
        ├── TeamGenerationResult.java           (record — output)
        ├── TeamGenerationStrategyFactory.java  (@Component — resolves by GenerationType)
        ├── BalancedGenerationStrategy.java
        ├── RandomGenerationStrategy.java
        └── SnakeDraftGenerationStrategy.java
```

---

## Request Lifecycle

```
HTTP Request
     │
     ▼
JwtAuthenticationFilter  (reads Bearer token, populates SecurityContext)
     │
     ▼
DispatcherServlet
     │
     ▼
Controller  (@PreAuthorize SpEL checked here via AOP)
     │  @Valid — JSR-380 bean validation on DTO
     ▼
Service  (business logic, cache annotations, @Transactional)
     │
     ├── Repository  (Spring Data JPA → Hibernate → HikariCP → PostgreSQL)
     │
     └── MapStruct Mapper  (entity → DTO, compile-time)
     │
     ▼
ResponseEntity<DTO>  (Jackson serialises Record fields to JSON)
```

---

## Security Architecture

### Default Profile: HMAC JWT

```
Client
  │
  ├─ POST /api/auth/login  { identifier, password }
  │         │
  │         ▼
  │   AuthService.login()
  │         ├── AuthenticationManager.authenticate()
  │         │         └── UserDetailsServiceImpl.loadUserByUsername()
  │         │                  ├── by email (if "@" present)
  │         │                  └── by username
  │         └── JwtTokenProvider.generateToken() → HMAC-SHA256 JWT
  │         └── return LoginResponseDTO { token, userId, username, role, ... }
  │
  └─ All subsequent requests:
        Authorization: Bearer <token>
              │
              ▼
        JwtAuthenticationFilter
              ├── Extract token from header
              ├── JwtTokenProvider.validateToken() → true/false
              ├── JwtTokenProvider.getUsername()
              ├── UserDetailsServiceImpl.loadUserByUsername()
              └── Set SecurityContextHolder with UserPrincipal
```

**`JWT_SECRET` env var is mandatory** — the application fails fast on startup if absent.
Minimum 64 characters. No default fallback exists.

### Keycloak Profile

When `spring.profiles.active=keycloak`, a separate `SecurityConfig` bean configures
OAuth2 Resource Server with JWKS validation. Keycloak runs at `http://localhost:8180`
(local) or a dedicated Keycloak instance in production.

### Roles

| Role          | Description                                                       |
|---------------|-------------------------------------------------------------------|
| `ADMIN_USER`  | Full system access. Cannot own a player profile.                 |
| `MASTER_USER` | Creates/manages matches and plans. Can own a player profile.     |
| `BASIC_USER`  | Views stats. Can self-confirm availability. Can own a player.    |

---

## Database Architecture

### Schema Overview

```
users ──────────────────────────────────────┐
   │ (user_id FK nullable)                  │
   ▼                                        │
players ─────────────────────────────────── │
   │                                        │
   │ (player_id FK)                         │
   ▼                                        │
player_stats ◄── match_teams ◄── matches ──┤season_id FK
                                   │        │
                            winning_team_id │
                                            │
match_plans ◄──────────────────────────────┘
   │ (match_plan_id FK)
   ▼
player_confirmations

skill_rating_history (player_id FK, match_id FK nullable)
goals (match_id FK, scorer_id FK, assister_id FK nullable)
seasons
```

### Migration History

| Migration | File                                | Description                                     |
|-----------|-------------------------------------|-------------------------------------------------|
| V1        | `V1__initial_schema.sql`            | Complete baseline schema (all tables, indexes, seed) |
| V2        | `V2__player_stats_goal_types.sql`   | Adds `solo_goals`, `assisted_goals`, `penalty_goals` to `player_stats` |
| V3        | `V3__player_aggregate_stats.sql`    | Adds `total_matches_played`, `total_goals`, `total_assists` to `players` (+ backfill) |
| V4        | `V4__draft_sessions.sql`            | Draft-session tables for the interactive Captain Pick feature |
| V5        | `V5__optimistic_lock_and_season_constraints.sql` | `players.version` optimistic-lock column; `matches.season_id` set `NOT NULL` |
| V6        | `V6__draft_session_version.sql`     | `draft_sessions.version` optimistic-lock column |
| V7        | `V7__history_applied_stats.sql`     | `skill_rating_history.goals_applied` / `assists_applied` — the contribution each rating application consumed, so a recalculation reverses exactly even after a stat amendment |
| V8        | `V8__missing_fk_indexes.sql`        | Indexes for previously unindexed FKs (`player_confirmations.player_id`, `goals.match_team_id`, `goals.assister_id`, `matches.winning_team_id`) + composite for the confirmation-ordered query |
| V9        | `V9__match_needs_recalc.sql`        | `matches.needs_recalc` — durable marker for a failed asynchronous rating recalculation |
| V10       | `V10__single_current_season.sql`    | `ux_seasons_single_current` — partial unique index over `is_current = TRUE`, so at most one season can be current. Previously a second current row made `findByCurrentTrue()` raise, 500-ing every season-resolving request |
| V11       | `V11__player_anonymized_at.sql`     | `players.anonymized_at` — records a GDPR erasure. Erasure anonymises in place rather than deleting, because `players(id)` cascades into `player_stats`, `goals` and `skill_rating_history` and a delete would rewrite other players' records |
| V12       | `V12__push_subscriptions.sql`       | `push_subscriptions` + `notification_mutes` — Web Push registrations and per-category opt-outs. Both cascade from `users`, so GDPR erasure removes them |
| V13       | `V13__match_plan_reminder_guards.sql` | `match_plans.deadline_reminder_sent_at` / `match_reminder_sent_at` — conditional-update guards so the reminder scheduler is safe on more than one instance and across restarts |

> ⚠️ **Never modify existing migration files.** Always add new numbered migrations.

### Key Design Decisions

- `players.email` is **not stored** — derived at query time from the linked `AppUser.email`
- `matches.version` enables **optimistic locking** for concurrent completion prevention
- `match_plans.confirmed_count` is **denormalised** for read performance at the cost of
  careful increment/decrement in service layer
- `player_stats.goals` is a **denormalised total** (`solo + assisted + penalty`) kept
  for backwards-compatible reporting queries

---

## Caching Architecture

**Provider:** Caffeine (local in-process cache — no Redis dependency)

| Cache Name      | TTL        | Max Entries | Used By                                  |
|-----------------|------------|-------------|------------------------------------------|
| `players`       | 10 minutes | 500         | Player list (paginated)                  |
| `playerProfile` | 10 minutes | 500         | Player by ID                             |
| `matches`       | 10 minutes | 500         | Match list + by ID                       |
| `rankings`      | 10 minutes | 500         | Rankings / leaderboards (future)         |
| `leaderboards`  | 10 minutes | 500         | Reserved                                 |
| `seasons`       | 10 minutes | 500         | Season data (future)                     |
| `users`         | 10 minutes | 500         | User list + by ID                        |

Eviction policy: writes always trigger `allEntries = true` eviction on the affected
cache(s). Completing a match evicts both `matches` and `players` (skill ratings change).

---

## Virtual Threads (Java 21)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Spring Boot 3.4+ maps all Tomcat request threads to Java Virtual Threads when this is
enabled. Virtual threads are JVM-managed, lightweight (≈1KB stack vs 512KB for OS threads),
and park (not block) during I/O operations.

**Effect:** The application can handle thousands of concurrent requests on a single
Tomcat instance without an explicit thread-pool tuning. The HikariCP pool is kept
bounded at 20 connections — virtual threads will park waiting for a DB connection
without consuming OS threads.

---

## Performance Targets (Production)

| Metric              | Target                   |
|---------------------|--------------------------|
| P99 response time   | < 200ms                  |
| GC pause (ZGC)      | < 5ms                    |
| Cache hit rate      | > 80%                    |
| DB connection pool  | 20 connections (HikariCP)|
| Thread model        | Virtual Threads (JDK 21) |

### Production JVM Flags

```
-XX:+UseZGC -XX:+ZGenerational -XX:+UseStringDeduplication
```

---

## Configuration Reference

### Required Environment Variables

| Variable               | Required | Description                                              |
|------------------------|----------|----------------------------------------------------------|
| `JWT_SECRET`           | **Yes**  | HMAC-SHA256 signing key. Minimum 64 chars. App fails without it. |
| `DATABASE_URL`         | No       | JDBC URL. Default: `jdbc:postgresql://localhost:5432/football` |
| `DATABASE_USERNAME`    | No       | Default: `postgres`                                      |
| `DATABASE_PASSWORD`    | No       | Default: `postgres`                                      |
| `CORS_ALLOWED_ORIGINS` | No       | Default: `http://localhost:3000`                         |

### Profile Activation

| Profile     | Description                                         |
|-------------|-----------------------------------------------------|
| `dev`       | Local development — verbose logging, relaxed security |
| `prod`      | Production — strict security, optimised settings    |
| `test`      | H2 in-memory database for unit/controller tests     |
| `keycloak`  | Replace HMAC JWT with OAuth2 Resource Server        |

### Generate a JWT Secret (Development)

```powershell
# PowerShell
./scripts/generate-jwt-secret.ps1
```

```bash
# Bash
openssl rand -base64 64
```

---

## API Design Conventions

### URL Patterns

```
/api/{resource}                              (collection)
/api/{resource}/{id}                         (single resource)
/api/{resource}/{id}/{sub-resource}          (sub-collection)
/api/{resource}/{id}/{sub-resource}/{subId}  (single sub-resource)
```

### HTTP Semantics

| Method   | Semantics                 | Response            |
|----------|---------------------------|---------------------|
| `GET`    | Read — never modifies     | `200 OK` + DTO      |
| `POST`   | Create resource           | `201 Created` + DTO |
| `PUT`    | Full replace              | `200 OK` + DTO      |
| `PATCH`  | Partial update (safe)     | `200 OK` + DTO      |
| `DELETE` | Remove resource           | `204 No Content`    |

**Safe PATCH:** `null` fields in PATCH requests mean "no change". Explicit flags (e.g.
`unlinkUser: true` in `PlayerUpdateDTO`) are used when `null` would be ambiguous.

### Error Response

All errors return:

```json
{
  "timestamp": "2026-05-22T10:00:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Player with id 42 not found",
  "path": "/api/players/42",
  "violations": []
}
```

Validation failures populate `violations`:

```json
{
  "violations": [
    { "field": "name", "message": "must not be blank", "rejectedValue": "" }
  ]
}
```

---

## Testing Strategy

| Test Type          | Profile   | Database | Scope                                         |
|--------------------|-----------|----------|-----------------------------------------------|
| Unit (service)     | `test`    | None     | `@ExtendWith(MockitoExtension.class)`          |
| Controller (slice) | `test`    | H2       | `@WebMvcTest` + `@MockBean` service            |
| Integration (E2E)  | `integration`| PostgreSQL (Testcontainers) | Full stack  |

### Test Counts (v1.0.0)

| Component          | Tests |
|--------------------|-------|
| UserService        | ~30   |
| PlayerService      | 24    |
| PlayerController   | 24    |
| MatchService       | 40    |
| MatchController    | 40    |
| MatchPlanService   | ~60   |
| MatchPlanController| ~45   |
| CalculationService | 37    |
| **Total**          | **~300** |

---

## Local Development Quick Start

```bash
# 1. Start PostgreSQL (Docker)
docker-compose up -d postgres

# 2. Set required env var
$env:JWT_SECRET = "dev-local-secret-do-not-use-in-production-minimum-64-chars-here"

# 3. Run the app
./gradlew bootRun --args='--spring.profiles.active=dev'

# 4. Swagger UI
# http://localhost:8080/swagger-ui.html

# 5. Login with default admin
# POST http://localhost:8080/api/auth/login
# { "identifier": "admin", "password": "Admin@1234" }
# (force_password_change = true — change password before using other endpoints)
```

---

## Agent-Assisted Development

All development tasks go through AI agents in `.github/agents/`. The **orchestrator**
agent is the single entry point — it classifies the task and delegates to specialist agents:

| Agent                    | Domain                                 |
|--------------------------|----------------------------------------|
| `orchestrator`           | Master — classifies and delegates everything |
| `requirements-analyst`   | Feature decomposition, impact analysis |
| `api-designer`           | Endpoint contracts, DTO schemas        |
| `db-migration`           | Flyway SQL, schema evolution           |
| `dev-assistant`          | Production code implementation         |
| `phase3-compliance`      | Architecture review                    |
| `test-engineer`          | Unit/controller/integration tests      |
| `security-auditor`       | CVE scanning, auth review              |
| `documentation-writer`   | API docs, feature guides               |
| `calculation-service`    | Rating formula refactoring             |
| `version-updater`        | Version bumps, CHANGELOG               |
| `deployment-engineer`    | Docker, Heroku, deployment             |
| `postman-engineer`       | Postman collection management          |

