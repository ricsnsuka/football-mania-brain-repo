# GitHub Copilot Instructions — Football Management System

> **Version:** 1.0.0
> **Stack:** Java 21 · Spring Boot 3.4.5 · PostgreSQL · Caffeine · MapStruct
> **Last Updated:** May 15, 2026

---

## 🧭 Project Overview

A **Football Management System** REST API that manages players, matches, teams, seasons,
match planning with availability polls, and skill ratings. Built from scratch on Java 21 to fully leverage
**Virtual Threads**, **Records**, **Pattern Matching**, and **ZGC** for production performance.

---

## 🏗️ Technology Stack

| Concern     | Technology                           | Notes                                      |
|-------------|--------------------------------------|--------------------------------------------|
| Language    | Java 21                              | Records, virtual threads, pattern matching |
| Framework   | Spring Boot 3.4.5                    | WebMVC with virtual thread executor        |
| Database    | PostgreSQL (prod), H2 (test)         | Flyway migrations                          |
| Caching     | Caffeine                             | Local cache — no Redis                     |
| Security    | Spring Security + JJWT 0.12.6        | HMAC JWT (default) or Keycloak profile     |
| Mapping     | MapStruct 1.6.3                      | Compile-time, no runtime reflection        |
| Boilerplate | Lombok 1.18.38                       | Entities & configs only; DTOs use Records  |
| API Docs    | SpringDoc OpenAPI 2.8.6              | Swagger UI at `/swagger-ui.html`           |
| Testing     | JUnit 5, Testcontainers, H2          | H2 (unit/controller), PG (E2E)             |
| Build       | Gradle 8 + Foojay toolchain resolver | Auto-downloads Java 21 toolchain           |

---

## 📁 Package Structure

```
src/main/java/pt/rics/demo/football/
├── FootballApplication.java          # @SpringBootApplication, @EnableCaching, @EnableAsync
├── config/
│   ├── CacheConfig.java              # Caffeine cache definitions
│   ├── OpenApiConfig.java            # Swagger/OpenAPI config
│   └── SecurityConfig.java          # @Profile("!keycloak") — HMAC JWT mode
├── controller/
│   └── HealthController.java        # GET /api/health
├── dto/                              # Java Records ONLY — no classes
├── exception/
│   ├── ResourceNotFoundException.java
│   ├── BusinessException.java
│   └── GlobalExceptionHandler.java  # @RestControllerAdvice, ApiError record
├── mapper/                           # MapStruct @Mapper interfaces
├── model/                            # JPA entities (@Entity, Lombok @Getter/@Setter)
├── repository/                       # Spring Data JPA interfaces
├── security/
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java  # @Profile("!keycloak")
│   └── UserPrincipal.java            # Java Record implementing UserDetails
└── service/                          # Concrete @Service classes — NO interfaces
```

```
src/main/resources/
├── application.yml                   # Virtual threads, HikariCP, HTTP/2, compression
├── application-dev.yml
├── application-prod.yml
├── application-test.yml              # H2 in-memory, PostgreSQL compatibility mode
└── db/migration/
    └── V1__initial_schema.sql        # Complete baseline schema
```

---

## ⚙️ Core Java 21 Conventions

### 1. DTOs are Java Records — Always

```java
// ✅ Correct
public record PlayerCreateDTO(
                @NotBlank @Size(min = 2, max = 100) String name,
                @NotNull @Min(1) @Max(10) Integer baseSkillRating,
                @Size(max = 20) String phoneNumber,
                Boolean isActive
        ) {
}

// ❌ Wrong — never use classes for DTOs
public class PlayerCreateDTO { ...
}
```

### 2. No Service Interfaces

```java
// ✅ Correct — inject concrete class
@Service
@RequiredArgsConstructor
public class PlayerService {
    private final PlayerRepository playerRepository;
}

// ❌ Wrong — no pointless interface/impl split
public interface PlayerService { ...
}

public class PlayerServiceImpl implements PlayerService { ...
}
```

### 3. Entities Use Lombok — Not Records

```java
// ✅ Correct — Lombok on JPA entities
@Entity
@Table(name = "players")
@Getter
@Setter
@NoArgsConstructor
public class Player {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

### 4. Use Modern Stream/Collection Methods

```java
// ✅ Java 21
list.stream().

filter(...).

toList();          // not Collectors.toList()
Map.

of("key",value);                        // not new HashMap<>()

// ✅ Pattern matching
if(obj instanceof
String s){...}
        switch(shape){
        case
Circle c  ->c.

radius();
    case
Rectangle r ->r.

width();
}
```

### 5. Exception Factory Methods

```java
// ✅ Use factory methods — never `new` directly
throw ResourceNotFoundException.of("Player",id);
throw ResourceNotFoundException.

of("Player","email",email);
throw BusinessException.

conflict("Player name already exists");
throw BusinessException.

forbidden("Cannot modify admin user");
throw BusinessException.

badRequest("Confirmed player count must be at least 14 for SEVEN_A_SIDE");
```

---

## 🔒 Security

### JWT Mode (default — `@Profile("!keycloak")`)

- `SecurityConfig` configures the filter chain
- `JwtTokenProvider` signs/validates tokens using HMAC-SHA256
- `JwtAuthenticationFilter` extracts `UserPrincipal` from bearer token
- `UserPrincipal` is a **Java Record** implementing `UserDetails`
- ⚠️ **`JWT_SECRET` env var is mandatory** — app fails fast on startup if not set; no default exists

### Keycloak Mode (`spring.profiles.active=keycloak`)

- OAuth2 Resource Server with JWKS validation
- `SecurityConfig` replaced by keycloak-specific config

### Roles

| Role          | Description                               |
|---------------|-------------------------------------------|
| `ADMIN_USER`  | System administrator — cannot be a player |
| `MASTER_USER` | Match organizer/manager                   |
| `BASIC_USER`  | Regular player                            |

---

## 💾 Database

### Migrations

- All schema changes via **Flyway** in `src/main/resources/db/migration/`
- Naming: `V{number}__{description}.sql` (e.g., `V2__add_player_avatar.sql`)
- H2 with `MODE=PostgreSQL` for tests — use PostgreSQL-compatible SQL only
- **Never modify existing migration files** — always add new ones

### Key Tables (from V1)

| Table                  | Description                             |
|------------------------|-----------------------------------------|
| `users`                | Authentication accounts                 |
| `players`              | Player profiles (optional link to user) |
| `seasons`              | Season periods                          |
| `matches`              | Match records                           |
| `match_teams`          | Two teams per match                     |
| `player_stats`         | Per-player per-match stats — includes `solo_goals`, `assisted_goals`, `penalty_goals` |
| `goals`                | Goal events with scorer/assister        |
| `skill_rating_history` | Audit trail of skill rating changes     |
| `match_plans`          | Pre-match planning                      |
| `player_confirmations` | RSVPs for match plans                   |

---

## 🗃️ Caching

**Provider:** Caffeine (local, no Redis)

| Cache Name      | TTL        | Max Entries |
|-----------------|------------|-------------|
| `players`       | 10 minutes | 500         |
| `playerProfile` | 10 minutes | 500         |
| `matches`       | 10 minutes | 500         |
| `rankings`      | 10 minutes | 500         |
| `leaderboards`  | 10 minutes | 500         |
| `seasons`       | 10 minutes | 500         |
| `users`         | 10 minutes | 500         |

**Rules:**

- Use `@Cacheable`, `@CacheEvict`, `@CachePut` annotations only
- Evict related caches on writes (e.g., evict `players` + `rankings` on player update)
- Never cache security-sensitive data

---

## 🌐 API Conventions

### URL Pattern

```
/api/{resource}
/api/{resource}/{id}
/api/{resource}/{id}/{sub-resource}
```

### HTTP Semantics

| Method   | Semantics                   | Response            |
|----------|-----------------------------|---------------------|
| `GET`    | Read — never modifies state | `200 OK` + DTO      |
| `POST`   | Create resource             | `201 Created` + DTO |
| `PUT`    | Full replace                | `200 OK` + DTO      |
| `PATCH`  | Partial update              | `200 OK` + DTO      |
| `DELETE` | Remove resource             | `204 No Content`    |

### Controller Pattern

```java

@RestController
@RequestMapping("/api/players")
@RequiredArgsConstructor
@Tag(name = "Players", description = "Player management")
public class PlayerController {

    private final PlayerService playerService;

    @GetMapping("/{id}")
    @Operation(summary = "Get player by ID")
    public ResponseEntity<PlayerDTO> getPlayer(@PathVariable Long id) {
        return ResponseEntity.ok(playerService.getPlayer(id));
    }

    @PostMapping
    @Operation(summary = "Create player")
    public ResponseEntity<PlayerDTO> createPlayer(@Valid @RequestBody PlayerCreateDTO dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(playerService.createPlayer(dto));
    }
}
```

### Error Response Shape

```json
{
  "timestamp": "2026-05-15T10:30:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Player with id 42 not found",
  "path": "/api/players/42",
  "violations": []
}
```

Validation errors populate `violations`:

```json
{
  "violations": [
    {
      "field": "name",
      "message": "must not be blank",
      "rejectedValue": ""
    },
    {
      "field": "baseSkillRating",
      "message": "must be between 1 and 10",
      "rejectedValue": 15
    }
  ]
}
```

---

## 📊 Business Rules

### Players

- Skill ratings range **1.0 – 10.0**
- `baseSkillRating` set at creation (integer, 1–10) — never modified by algorithm
- `skillRating` (double) evolves via match performance
- `ADMIN_USER` accounts **cannot** be linked to players
- Inactive players excluded from team generation
- `email` is sourced from `AppUser.email` — never stored in the `players` table; null if no linked user
- Hard delete blocked if player has any `player_stats` records
- **Self-link** (`POST /api/players/{id}/link-me`): user ID from JWT only; ADMIN_USER → 403; player already linked to self → 409; player linked to other user → 409; caller already linked to another player → 409

### Matches & Teams

- Match types are 5-, 7- and 11-a-side, needing exactly **10 / 14 / 22** confirmed players
  respectively (both teams combined) → 2 equal teams of 5 / 7 / 11
- A plan may hold **more** confirmations than the match needs; generation takes the first N in
  confirmation order and the rest are reserves
- Manual creation: provide `teams` list with player IDs
- `generationType` (see `GenerationType`): `MANUAL`, `BALANCED`, `RANDOM`, `SNAKE_DRAFT`,
  `FORM_BASED`, `CAPTAIN_PICK` — plus `STREAK_AWARE`, which is declared but not yet implemented

### Skill Rating Calculation (Rating Model v2.1)

Ratings are **not** a fixed additive score. Each player first accumulates an **unbounded RAW
score**, then the whole match is **normalized proportionally** against the top performer using
a **stats-dependent ceiling**. Only goals (solo/assisted/penalty), assists, and own goals are
tracked — no other per-player stats exist.

```
# Per-goal-type points (SOLO 3.0 > ASSISTED 2.0 > PENALTY 1.0; ASSIST 1.5; OWN_GOAL −2.0)
# Goal-timing impact (when Goal rows exist, ordered by minute ASC NULLS LAST then createdAt):
#   seqFactor = 1 + 0.50 × (index / (goalCount−1))         # later goals matter more
#   impact    = seqFactor + 0.40 (go-ahead) + 0.25 (equalizer)
#   assister receives 50% of the timing uplift
# Graceful FLAT fallback to per-type PlayerStat weights when no Goal rows exist.

raw = RAW_BASE_POINTS (7.5 — v2.1: elevated from 1.0)
    + statPoints                                            # timing-weighted OR flat, minus ownGoals×2.0
    + min(decisiveness, 1.5) × 1.0
    + WIN_BONUS (+0.4 — v2.1: reduced from 1.0) | LOSS_PENALTY (−2.2 — v2.1: increased from 0.75) | 0 (draw)
    ± min(|goalDiff| × 0.10, 0.50)                          # goal-diff nudge, cap 0.50

# Match-wide proportional normalization (two-pass, in recalculateMatchRatings):
# v2.1 compressed mapping to [RATING_FLOOR, ceiling] instead of [0, ceiling]
ceiling     = 8.0 + 1.5 × clamp(topStatPoints / 9.0, 0, 1)  # band [8.0, 9.5] (v2.1: compressed from [6.5, 10.0])
finalRating = 4.0 + (raw / topRaw) × (ceiling − 4.0)        # v2.1: compressed mapping
finalRating = clamp(finalRating, 1.0, 10.0)
```

- **decisiveness** = `(scoreShare×0.60 + involvementRatio×0.40) × (1 + 1/max(|goalDiff|,1))`, capped `1.5`.
- The ceiling scales with the **top performer's absolute stat quality**: a 1 goal + 1 assist top
  performance ≈ **8.75** (never 9.5); a 3g+2a WIN ≈ **9.5**.
- **Edge cases (v2.1):** single-player match → compressed ceiling; 0-0 no-stats draw → **8.0**
  (`CEILING_MIN` after compressed mapping); all RAW ≤ 0 → floor **1.0**. Own goals credit the
  opponent's scoreboard and are never rewarded.
- **Realistic distributions (v2.1):** non-contributor on WIN → ~6.1-6.5 (>6.0 ✅); non-contributor
  on LOSS → ~5.0-5.5; 1-goal contributor on WIN → ~7.0-7.5 (was 4.1 in v2 — FIXED).
- **RAW vs. normalized:** the public `computeMatchRating()` overloads return the RAW flat-clamped
  value used transiently by `MatchService`; the **authoritative normalized** rating is written
  asynchronously by `recalculateMatchRatings` after completion and drives `skillRating`.
- **Live preview** (`PATCH /api/matches/{id}/stats/live`) stays **bonus-free**: no WIN/LOSS, no
  goal-diff, no scarcity, no normalization.
- New dependency: `GoalRepository.findByMatchIdOrderByTiming(matchId)`.

All weight constants live in `CalculationService`. `computeMatchRating()` is the sole source of
ratings — clients **never submit a rating value**. See
[`docs/features/CALCULATION_SERVICE.md`](../docs/features/CALCULATION_SERVICE.md) for full detail.

### Season Transitions

```
newRating = (endRating × 0.45) + (avgRating × 0.25) +
            (startRating × 0.10) + (meanRating × 0.20) +
            activityAdjustment
```

Bounds: 1.0 – 10.0 · Max change per season: ±2.0

---

## 🧪 Testing

### Test Profiles

| Profile       | Database                      | Use Case                      |
|---------------|-------------------------------|-------------------------------|
| `test`        | H2 in-memory                  | Unit, controller, slice tests |
| `integration` | PostgreSQL via Testcontainers | E2E, full-stack tests         |

### Conventions

```java
// Controller slice
@WebMvcTest(PlayerController.class)
@ActiveProfiles("test")
class PlayerControllerTest {
    @Autowired
    MockMvc mockMvc;
    @MockBean
    PlayerService playerService;
}

// Service unit test
@ExtendWith(MockitoExtension.class)
class PlayerServiceTest {
    @Mock
    PlayerRepository playerRepository;
    @InjectMocks
    PlayerService playerService;
}
```

### Rules

1. ❌ Never `@Transactional` on `@BeforeEach`
2. ❌ Never `Collectors.toList()` — use `.toList()`
3. ✅ `@ParameterizedTest` for boundary checks
4. ✅ `@Nested` classes for scenario grouping
5. Target **90% branch coverage** on service classes

---

## 📝 Validation Reference

| Constraint    | Annotation                           |
|---------------|--------------------------------------|
| Required      | `@NotNull`, `@NotBlank`, `@NotEmpty` |
| String length | `@Size(min=X, max=Y)`                |
| Number range  | `@Min(X)`, `@Max(Y)`                 |
| Email         | `@Email`                             |
| Positive      | `@Positive`, `@PositiveOrZero`       |
| Future date   | `@Future`, `@FutureOrPresent`        |
| Skill rating  | `@Min(1) @Max(10)` on Integer fields |

---

## 📁 Documentation Standards

### Version Folder Structure

```
docs/
  vX.Y.Z/
    README.md
    RELEASE_NOTES.md
    CHANGELOG.md
    implementation/
    api/
    deployment/
```

### What Goes Where

| Content                       | Location                      |
|-------------------------------|-------------------------------|
| Feature implementation guides | `docs/vX.Y.Z/implementation/` |
| API reference                 | `docs/vX.Y.Z/api/`            |
| Migration / deployment guide  | `docs/vX.Y.Z/deployment/`     |
| Session notes / scratch       | `docs/development/`           |
| Orchestration plans           | `docs/plans/`                 |
| Incident / hotfix logs        | `docs/fixes/`                 |

### ⚠️ Never commit to root:

`*_SUMMARY.md`, `*_FIX.md`, `SESSION_*.md`, `*.ps1`, `*.dump`, `uploads/`

---

## 📐 Player API — DTO Definitions & Endpoint Contracts

### Player DTOs (Java Records)

#### PlayerDTO (response)

| Field            | Type    | Nullable | Notes                                       |
|------------------|---------|----------|---------------------------------------------|
| `id`             | Long    | No       |                                             |
| `name`           | String  | No       |                                             |
| `email`          | String  | Yes      | From linked `AppUser.email`; null if no link|
| `isActive`       | boolean | No       |                                             |
| `skillRating`    | double  | No       | 1.0–10.0, evolves with matches              |
| `baseSkillRating`| int     | No       | 1–10, set at creation, never auto-modified  |
| `phoneNumber`    | String  | Yes      |                                             |
| `currentStreak`  | int     | No       |                                             |
| `longestStreak`  | int     | No       |                                             |
| `linkedUserId`   | Long    | Yes      |                                             |
| `createdBy`      | String  | No       | Audit: username of creator                  |
| `updatedBy`      | String  | No       | Audit: username of last updater             |
| `createdAt`      | Instant | No       |                                             |
| `updatedAt`      | Instant | No       |                                             |

#### PlayerCreateDTO (POST /api/players)

| Field             | Type    | Required | Validation                 |
|-------------------|---------|----------|----------------------------|
| `name`            | String  | Yes      | `@NotBlank @Size(min=2, max=100)` |
| `baseSkillRating` | Integer | Yes      | `@NotNull @Min(1) @Max(10)` |
| `phoneNumber`     | String  | No       | `@Size(max=20)`             |
| `isActive`        | Boolean | No       | Defaults to `true` in service |
| `userId`          | Long    | No       | Links to AppUser; ADMIN_USER rejected |

#### PlayerUpdateDTO (PATCH /api/players/{id})

| Field         | Type    | Required | Validation              | Notes                                             |
|---------------|---------|----------|-------------------------|---------------------------------------------------|
| `name`        | String  | No       | `@Size(min=2, max=100)` | `null` = no change                                |
| `phoneNumber` | String  | No       | `@Size(max=20)`         | `null` = no change                                |
| `userId`      | Long    | No       | —                       | `null` = no change (use `unlinkUser` to remove)   |
| `unlinkUser`  | Boolean | No       | —                       | `true` = explicitly remove user link (safe PATCH) |

> ⚠️ **Safe PATCH**: `null` fields mean "no change". To remove a user link, set `unlinkUser: true` — do **not** rely on `userId: null`.

#### PlayerStatusDTO (PATCH /api/players/{id}/status)

| Field      | Type    | Required | Validation  |
|------------|---------|----------|-------------|
| `isActive` | Boolean | Yes      | `@NotNull`  |

### Player Endpoints

| Method   | Path                           | Authorization                           | Description                                                       |
|----------|--------------------------------|-----------------------------------------|-------------------------------------------------------------------|
| `GET`    | `/api/players`                 | `isAuthenticated()`                     | List players (paginated, filter by active)                        |
| `GET`    | `/api/players/{id}`            | `isAuthenticated()`                     | Get player by ID                                                  |
| `POST`   | `/api/players`                 | `hasRole('ADMIN_USER') or hasRole('MASTER_USER')` | Create player                                         |
| `PATCH`  | `/api/players/{id}`            | `hasRole('ADMIN_USER') or hasRole('MASTER_USER')` | Partial update                                        |
| `PATCH`  | `/api/players/{id}/status`     | `hasRole('ADMIN_USER') or hasRole('MASTER_USER')` | Activate / deactivate                                 |
| `POST`   | `/api/players/{id}/link-me`    | `isAuthenticated()` (ADMIN_USER → 403 in service) | Self-link: associate calling user's account to player |
| `DELETE` | `/api/players/{id}`            | `hasRole('ADMIN_USER')`                 | Hard delete (no match history only)                               |

---

## 📐 Match API — DTO Definitions & Endpoint Contracts

### Match Endpoints

| Method   | Path                                               | Authorization                                     | Description                                      |
|----------|----------------------------------------------------|---------------------------------------------------|--------------------------------------------------|
| `POST`   | `/api/matches`                                     | `ADMIN_USER` or `MASTER_USER`                     | Create match with manual teams                   |
| `GET`    | `/api/matches`                                     | Any authenticated                                 | List matches (paginated)                         |
| `GET`    | `/api/matches/{id}`                                | Any authenticated                                 | Get match by ID                                  |
| `PATCH`  | `/api/matches/{id}`                                | `ADMIN_USER` or `MASTER_USER`                     | Update description / date / location             |
| `PATCH`  | `/api/matches/{id}/complete`                       | `ADMIN_USER` or `MASTER_USER`                     | Complete match; compliance check; server ratings |
| `PATCH`  | `/api/matches/{id}/stats/live`                     | `ADMIN_USER` or `MASTER_USER`                     | Live stat update; returns preview ratings        |
| `GET`    | `/api/matches/{id}/teams`                          | Any authenticated                                 | Get both teams                                   |
| `PATCH`  | `/api/matches/{id}/teams/{teamId}/stats/{statId}`  | `ADMIN_USER`                                      | Amend a single player stat post-completion       |
| `POST`   | `/api/matches/{id}/recalculate`                    | `ADMIN_USER`                                      | Idempotently recalc ratings for one completed match (200 + `RecalculationResultDTO`) |
| `POST`   | `/api/matches/recalculate`                         | `ADMIN_USER`                                      | Bulk recalc (matchIds / seasonId / all completed); per-match summary (200 + `BulkRecalculationResponseDTO`) |
| `DELETE` | `/api/matches/{id}`                                | `ADMIN_USER`                                      | Delete non-completed match                       |

### PlayerStatUpdateDTO (within MatchCompleteDTO / MatchLiveUpdateDTO / stat amend)

| Field           | Type    | Required | Notes                              |
|-----------------|---------|----------|------------------------------------|
| `playerStatId`  | Long    | **Yes**  | Must match an existing stat row    |
| `soloGoals`     | Integer | No       | Null = no change                   |
| `assistedGoals` | Integer | No       | Null = no change                   |
| `penaltyGoals`  | Integer | No       | Null = no change                   |
| `assists`       | Integer | No       | Null = no change                   |
| `ownGoals`      | Integer | No       | Null = no change                   |
| `isMvp`         | Boolean | No       | Null = no change                   |

> ⚠️ `goals` and `rating` fields **do not exist** on this DTO — never add them.

### PlayerStatDTO (response — inside MatchTeamDTO.playerStats)

| Field           | Type    | Nullable | Notes                                          |
|-----------------|---------|----------|------------------------------------------------|
| `id`            | Long    | No       |                                                |
| `playerId`      | Long    | No       |                                                |
| `playerName`    | String  | No       |                                                |
| `soloGoals`     | Integer | No       |                                                |
| `assistedGoals` | Integer | No       |                                                |
| `penaltyGoals`  | Integer | No       |                                                |
| `assists`       | Integer | No       |                                                |
| `ownGoals`      | Integer | No       |                                                |
| `rating`        | Double  | Yes      | Null before completion; server-computed after  |
| `isMvp`         | Boolean | No       |                                                |
| `matchResult`   | String  | Yes      | Null before completion; `WIN`/`LOSS`/`DRAW`    |

### PlayerMatchRatingDTO (inside MatchLiveUpdateResponseDTO.ratings)

| Field          | Type   | Notes                                              |
|----------------|--------|----------------------------------------------------|
| `playerStatId` | Long   | FK to `player_stats`                               |
| `playerId`     | Long   | FK to `players`                                    |
| `playerName`   | String | Display name                                       |
| `matchRating`  | double | Preview rating — no WIN/LOSS/goal-diff bonuses     |

---

## 📐 Match Rating Recalculation — DTO Definitions (Admin, v4.2.0)

> Idempotent admin recalculation of match ratings. `CalculationService.recalculateMatchRatings` is
> refactored to **reverse-then-reapply** so re-runs never double-count aggregates, streaks, EMA
> skillRating, or `skill_rating_history` rows. See
> [`docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md`](../docs/api/MATCH-RATING-RECALCULATION-API-CONTRACT.md).

### RecalculationResultDTO (response — single & per-item in bulk)

| Field            | Type   | Nullable | Notes                                              |
|------------------|--------|----------|----------------------------------------------------|
| `matchId`        | Long   | No       |                                                    |
| `status`         | String | No       | `SUCCESS` \| `FAILED` \| `SKIPPED` (no player stats) |
| `ratingsUpdated` | int    | No       | PlayerStat rows recomputed; `0` on failure         |
| `message`        | String | No       | Human-readable outcome                             |

### BulkRecalculationRequestDTO (POST /api/matches/recalculate)

| Field      | Type       | Required | Validation                          | Notes                                   |
|------------|------------|----------|-------------------------------------|-----------------------------------------|
| `matchIds` | List<Long> | No       | `@Size(max=500)`, elems `@Positive` | Explicit list                           |
| `seasonId` | Long       | No       | `@Positive`                         | All completed matches in season         |

> Precedence: `matchIds` → `seasonId` → all completed. Both present ⇒ 400. Empty/absent body ⇒ all completed.

### BulkRecalculationResponseDTO (response — bulk)

| Field            | Type                          | Nullable | Notes                                     |
|------------------|-------------------------------|----------|-------------------------------------------|
| `totalRequested` | int                           | No       | Matches the request resolved to           |
| `succeeded`      | int                           | No       | `status == SUCCESS` count                 |
| `failed`         | int                           | No       | `status == FAILED` count                  |
| `results`        | List<RecalculationResultDTO>  | No       | One entry per processed match             |

> Bulk returns **200 OK** even when individual matches fail; each match runs in its own transaction
> (partial-failure semantics). `succeeded + failed` may be `< totalRequested` when `SKIPPED` items exist.

All development tasks go through the **orchestrator** first.

### Agent Pipeline

```
 Phase 1          Phase 2         Phase 3          Phase 4
Requirements  →  API Design   →  DB Migration  →  Implementation
analyst            designer        engineer         dev-assistant

 Phase 5          Phase 6         Phase 7          Phase 8
Compliance    →  Testing      →  Security      →  Documentation
phase3-           test-            security-         documentation-
compliance        engineer         auditor           writer

 Phase 9          Phase 10        ✦ Cross-cutting
Version &    →   Deployment       calculation-service
Release           engineer        documentation-organizer
version-updater                   postman-engineer
```

### Agent File Locations

```
.github/agents/
├── orchestrator.agent.md            # 🎯 Master entry point — start here
├── requirements-analyst.agent.md
├── api-designer.agent.md
├── db-migration.agent.md
├── dev-assistant.agent.md
├── phase3-compliance.agent.md
├── test-engineer.agent.md
├── security-auditor.agent.md
├── documentation-writer.agent.md
├── documentation-organizer.agent.md
├── version-updater.agent.md
├── deployment-engineer.agent.md
├── calculation-service.agent.md
└── postman-engineer.agent.md
```

### Quick Agent Selection

| Task                   | Start With                |
|------------------------|---------------------------|
| Any task (recommended) | `orchestrator`            |
| New feature            | `orchestrator`            |
| Known bug fix          | `dev-assistant`           |
| API design question    | `api-designer`            |
| DB schema change       | `db-migration`            |
| Write / fix tests      | `test-engineer`           |
| Security / CVE scan    | `security-auditor`        |
| Docs update            | `documentation-writer`    |
| Docs reorganisation    | `documentation-organizer` |
| Release preparation    | `version-updater`         |
| Deploy to prod         | `deployment-engineer`     |
| Postman collection     | `postman-engineer`        |

---

## 🚫 Common Mistakes

| ❌ Don't                                        | ✅ Do instead                                     |
|------------------------------------------------|--------------------------------------------------|
| Use class for DTO                              | Use Java Record                                  |
| Create service interface + impl                | Single concrete `@Service` class                 |
| Use `Collectors.toList()`                      | Use `.toList()`                                  |
| Add Redis dependency                           | Use Caffeine cache                               |
| Create teams with ≠ 14 players (balanced mode) | Validate 14 players before generation            |
| Link `ADMIN_USER` to a player                  | Admin role is mutually exclusive with player     |
| Target Java 17 in any new code                 | Java 21 — records, sealed classes, pattern match |
| Use `new HashMap<>()` when immutable is fine   | Use `Map.of()`, `List.of()`                      |

---

## 🔑 Default Credentials (Development Only)

| Field    | Value                                         |
|----------|-----------------------------------------------|
| Username | `admin`                                       |
| Password | `Admin@1234`                                  |
| Role     | `ADMIN_USER`                                  |
| Note     | `force_password_change = true` on first login |

---

## ⚡ Performance Targets (Production)

| Metric             | Target                   |
|--------------------|--------------------------|
| P99 response time  | < 200ms                  |
| GC pause (ZGC)     | < 5ms                    |
| Cache hit rate     | > 80%                    |
| DB connection pool | 20 connections           |
| Thread model       | Virtual Threads (JDK 21) |

---

## 📦 Version & Deployment

- **JAR name:** `football-{version}.jar`
- **JVM flags (prod):** `-XX:+UseZGC -XX:+ZGenerational -XX:+UseStringDeduplication`
- **Java runtime:** `21` (Heroku `system.properties`)
- **Docker:** `docker-compose.yml` — postgres + keycloak services only (no Redis)
- **Active Spring profiles:** `dev` | `prod` | `test` | `keycloak`
