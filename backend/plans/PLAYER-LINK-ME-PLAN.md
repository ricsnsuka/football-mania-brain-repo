# Player Self-Link Endpoint — Technical Specification
**Date:** 2026-05-15
**Status:** APPROVED
**Priority:** MEDIUM
**Estimated Effort:** S

---

## 1. Requirement Summary

Introduce `POST /api/players/{id}/link-me` so that any authenticated non-admin user can
self-link their own account to an existing player record, without requiring an GROUP_ADMIN/MASTER
operator to perform the link on their behalf. This improves onboarding autonomy for
`BASIC_USER` accounts.

---

## 2. Business Rules & Constraints

| # | Rule | Source |
|---|------|--------|
| BR-1 | `ADMIN_USER` cannot be linked to any player — mutual exclusion | copilot-instructions.md |
| BR-2 | A player can be linked to at most one user (`players.user_id` is unique) | DB schema (V1) |
| BR-3 | A user can be linked to at most one player | `PlayerRepository.existsByUserId` |
| BR-4 | The caller can only link their own account — user ID is taken from the JWT `UserPrincipal`, never from the request body | New (this feature) |
| BR-5 | If the player is already linked to the **calling** user → `409 Conflict` with descriptive message | New (this feature) |
| BR-6 | If the player is already linked to a **different** user → `409 Conflict` | Derived from BR-2 |
| BR-7 | If the calling user is already linked to a **different** player → `409 Conflict` | Derived from BR-3 |
| BR-8 | Skill ratings and calculated fields are never touched by this operation | copilot-instructions.md |

### New Business Rules Introduced
- **BR-4**: A self-link operation derives the user identity exclusively from the JWT principal.
  There is no request body, so there is zero risk of a caller supplying a foreign `userId`.
- **BR-5**: The "already linked to self" case is explicitly a `409`, not a silent `200`.
  This is intentional: the endpoint is not designed to be silently idempotent
  (a `200` response would hide the fact that no state changed and could mask bugs in
  client retry logic).

---

## 3. Impact Analysis

### Affected Components

| Component | Type | Change Needed |
|-----------|------|---------------|
| `PlayerController` | Controller | NEW endpoint `POST /api/players/{id}/link-me` |
| `PlayerService` | Service | NEW method `linkMe(Long playerId, UserPrincipal principal)` |
| `PlayerRepository` | Repository | **No change** — `findByUserId` and `existsByUserId` already exist |
| `UserRepository` | Repository | **No change** — already injected into `PlayerService` |
| `PlayerMapper` | Mapper | **No change** — returns existing `PlayerDTO` |
| `PlayerDTO` | DTO | **No change** — all required fields already present (`linkedUserId`, `email`) |
| DB migration | Migration | **Not needed** — `players.user_id` FK column already exists (V1) |
| `PlayerServiceTest` | Test | NEW `@Nested` class `LinkMeTests` |
| `PlayerControllerTest` | Test | NEW `@Nested` class `LinkMeTests` |
| `FRONTEND_ENDPOINT_CHANGES.md` | Docs | NEW endpoint entry |
| `copilot-instructions.md` | Docs | Add endpoint to Player Endpoints table |

### Risk Assessment

- **Data Migration Risk:** NONE — no schema changes
- **Breaking Change:** NO — new endpoint added; no existing endpoints or DTOs modified
- **Performance Impact:** LOW — one additional `findById` (player) + one `findByUserId` (player
  lookup by userId) + one `findById` (user, for entity reference) + one `save`. All within a
  single transaction. Caches are evicted on write, consistent with all other write operations.

---

## 4. API Contract (Preliminary)

### New Endpoint

```
POST /api/players/{id}/link-me
Authorization: Bearer <JWT>    (any authenticated user)
Request body: (none)
```

### Success Response — 200 OK

```json
{
  "id": 7,
  "name": "Ricardo Costa",
  "email": "ricardo@example.com",
  "isActive": true,
  "skillRating": 6.5,
  "baseSkillRating": 7,
  "phoneNumber": null,
  "currentStreak": 3,
  "longestStreak": 5,
  "linkedUserId": 42,
  "createdBy": "admin",
  "updatedBy": "admin",
  "createdAt": "2026-01-01T00:00:00Z",
  "updatedAt": "2026-05-15T10:00:00Z"
}
```

### Error Responses

| HTTP Status | Trigger | `message` example |
|-------------|---------|-------------------|
| `403 Forbidden` | Caller is `ADMIN_USER` | `"ADMIN_USER accounts cannot be linked to a player"` |
| `404 Not Found` | Player `{id}` does not exist | `"Player with id 99 not found"` |
| `409 Conflict` | Player already linked to this calling user | `"Player is already linked to your account"` |
| `409 Conflict` | Player already linked to a different user | `"Player is already linked to another user"` |
| `409 Conflict` | Calling user already linked to a different player | `"You are already linked to another player"` |

> ⚠️ **HTTP method / project convention note for api-designer:**
> The project's `copilot-instructions.md` maps `POST` to 201 Created. This endpoint uses
> `POST` with `200 OK` because no new resource is created — a link association on an existing
> entity is mutated. The api-designer should confirm whether to keep `POST + 200` as specified,
> or prefer `PATCH /api/players/{id}/link-me` (which aligns with "partial update → 200 OK").
> Either is acceptable; PATCH is semantically cleaner. **This decision must be made before
> dev-assistant implements the controller.**

---

## 5. Database Schema Changes

**None.** The `players.user_id` nullable FK to `users.id` already exists from `V1__initial_schema.sql`.
The latest migration file is `V4__draft_sessions.sql`. No new migration file is required.

---

## 6. Task Breakdown

### Phase 2 — API Design
- [ ] Confirm HTTP method: `POST + 200 OK` vs `PATCH + 200 OK` for `/api/players/{id}/link-me`
- [ ] Finalise OpenAPI `@Operation` summary and response description

### Phase 3 — Database Migration
- [ ] *(Skipped — no schema changes required)*

### Phase 4 — Implementation

#### `PlayerService` — new method
```
linkMe(Long playerId, UserPrincipal principal) → PlayerDTO
```

Execution order inside the method:
1. Check `principal.authorities()` for `ROLE_ADMIN_USER` → throw `BusinessException.forbidden(...)` *(avoids extra DB call)*
2. `playerRepository.findById(playerId)` → throw `ResourceNotFoundException.of("Player", playerId)` if absent
3. If `player.getUser() != null && player.getUser().getId().equals(callerUserId)` → throw `BusinessException.conflict("Player is already linked to your account")`
4. If `player.getUser() != null` (different user) → throw `BusinessException.conflict("Player is already linked to another user")`
5. `playerRepository.findByUserId(callerUserId)` — if present (and different player) → throw `BusinessException.conflict("You are already linked to another player")`
6. `userRepository.findById(callerUserId)` → load `AppUser` entity reference for FK
7. `player.setUser(linkedUser)` + `player.setUpdatedBy(principal.username())` + `playerRepository.save(player)`
8. Return `playerMapper.toDto(savedPlayer)`

Cache: `@Caching(evict = { @CacheEvict("players", allEntries=true), @CacheEvict("playerProfile", allEntries=true) })`

- [ ] Task: Add `linkMe` method to `PlayerService`

#### `PlayerController` — new endpoint
```java
@PostMapping("/{id}/link-me")
@Operation(summary = "Link the calling user to a player (self-link)")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<PlayerDTO> linkMe(
        @PathVariable Long id,
        @AuthenticationPrincipal UserPrincipal principal) {
    return ResponseEntity.ok(playerService.linkMe(id, principal));
}
```

- [ ] Task: Add `linkMe` endpoint to `PlayerController`

### Phase 5 — Architecture Compliance
- [ ] Verify no skill rating calculations were added or bypassed
- [ ] Verify `CalculationService` is untouched
- [ ] Verify caching annotations match the pattern in `updatePlayer`

### Phase 6 — Testing

#### `PlayerServiceTest` — new `@Nested class LinkMeTests`
- [ ] `linkMe_success` — BASIC_USER, unlinked player, unlinked user → returns PlayerDTO, saves player
- [ ] `linkMe_adminUser_throwsForbidden` — ADMIN_USER caller → 403
- [ ] `linkMe_playerNotFound_throwsNotFound` — unknown player id → 404
- [ ] `linkMe_playerAlreadyLinkedToSelf_throwsConflict` — player.user == caller → 409
- [ ] `linkMe_playerAlreadyLinkedToOtherUser_throwsConflict` — player.user != null, != caller → 409
- [ ] `linkMe_callerAlreadyLinkedToOtherPlayer_throwsConflict` — caller linked elsewhere → 409
- [ ] `linkMe_masterUser_success` — MASTER_USER should also be allowed to self-link

#### `PlayerControllerTest` — new `@Nested class LinkMeTests`
- [ ] `linkMe_authenticated_returns200` — mock service returns PlayerDTO → assert 200 + body
- [ ] `linkMe_unauthenticated_returns403` — no principal → assert 403
- [ ] `linkMe_adminForbidden_returns403` — service throws `BusinessException.forbidden(...)` → assert 403
- [ ] `linkMe_playerNotFound_returns404` — service throws `ResourceNotFoundException` → assert 404
- [ ] `linkMe_playerAlreadyLinked_returns409` — service throws `BusinessException.conflict(...)` → assert 409
- [ ] `linkMe_callerAlreadyLinked_returns409` — service throws `BusinessException.conflict(...)` → assert 409

### Phase 7 — Security
- [ ] Verify `@PreAuthorize("isAuthenticated()")` is the correct expression (all roles except unauthenticated)
- [ ] Verify ADMIN_USER enforcement is in the **service layer** (not just `@PreAuthorize`) to ensure it is always enforced, even if the annotation is bypassed in tests
- [ ] Confirm no CSRF risk (API is stateless JWT — CSRF disabled in `SecurityConfig`)
- [ ] Confirm no IDOR risk: user ID **must** come from `UserPrincipal`, never from request body or path variable

### Phase 8 — Documentation
- [ ] Add endpoint row to Player Endpoints table in `copilot-instructions.md`
- [ ] Create or update `FRONTEND_ENDPOINT_CHANGES.md` with the new endpoint
- [ ] Update `API_REFERENCE.md` if it exists

### Phase 9 — Release
- [ ] Add entry to `RELEASE_NOTES.md` for the current version

---

## 7. Open Questions / Resolved Ambiguities

| # | Question | Resolution |
|---|----------|------------|
| Q1 | HTTP method: `POST` or `PATCH`? | **Flag for api-designer.** `POST + 200` (as specified) or `PATCH + 200` (more RESTful for a partial update). Both are acceptable; PATCH is recommended. |
| Q2 | "Player already linked to calling user" → 409 or 200? | **409 Conflict** per the requirement ("idempotent-friendly *or* 409"). Chosen 409 to avoid silently masking client bugs. |
| Q3 | Should the method load `AppUser` from DB to verify role, or trust `UserPrincipal.authorities()`? | **Trust `UserPrincipal.authorities()`** for the role check (avoids extra DB roundtrip). Load `AppUser` entity only when setting the FK on the Player. |
| Q4 | What `updatedBy` value should be stamped on the player after linking? | **`principal.username()`** — consistent with all other write operations. |
| Q5 | Should inactive players be linkable? | **Yes** — the requirement does not restrict this. A player's active status is independent of their account link status. |

---

## 8. Acceptance Criteria

- [ ] `POST /api/players/{id}/link-me` returns `200 OK` with full `PlayerDTO` on success
- [ ] Unauthenticated requests receive `403 Forbidden`
- [ ] `ADMIN_USER` receives `403 Forbidden` with message `"ADMIN_USER accounts cannot be linked to a player"`
- [ ] Non-existent player receives `404 Not Found`
- [ ] Player already linked to the calling user receives `409 Conflict`
- [ ] Player already linked to a different user receives `409 Conflict`
- [ ] Calling user already linked to a different player receives `409 Conflict`
- [ ] User ID is **never** accepted from the request body — only from the JWT principal
- [ ] `players` and `playerProfile` caches are evicted on successful link
- [ ] `PlayerDTO.linkedUserId` reflects the newly linked user ID in the response
- [ ] All new service branches have unit tests with ≥ 90% branch coverage
- [ ] All new controller branches have slice tests
- [ ] No DB migration file is added

---

## Next Steps

- Phase 2 (Design): Run **api-designer** agent — confirm POST vs PATCH, finalise OpenAPI spec
- Phase 3 (DB Migration): **Skipped** — no schema changes
- Phase 4 (Implementation): Run **dev-assistant** agent
- Phase 5 (Compliance): Run **phase3-compliance** agent
- Phase 6 (Testing): Run **test-engineer** agent
- Phase 7 (Security): Run **security-auditor** agent
- Phase 8 (Documentation): Run **documentation-writer** agent
- Phase 9 (Release): Run **version-updater** agent
- Phase 10 (Deployment): **Not required** — no infrastructure changes

---

## Summary for Next Agent

### Files to Create / Modify

| File | Action |
|------|--------|
| `src/main/java/pt/rics/demo/football/controller/PlayerController.java` | MODIFY — add `linkMe` endpoint |
| `src/main/java/pt/rics/demo/football/service/PlayerService.java` | MODIFY — add `linkMe(Long, UserPrincipal)` method |
| `src/test/java/pt/rics/demo/football/service/PlayerServiceTest.java` | MODIFY — add `@Nested class LinkMeTests` |
| `src/test/java/pt/rics/demo/football/controller/PlayerControllerTest.java` | MODIFY — add `@Nested class LinkMeTests` |
| `.github/copilot-instructions.md` | MODIFY — add endpoint row to Player Endpoints table |

### Key Decisions Made

1. **No DB migration** — `players.user_id` FK already exists.
2. **No new DTO** — response is the existing `PlayerDTO`; user ID is sourced from `UserPrincipal.id()`.
3. **ADMIN_USER check in service layer** using `UserPrincipal.authorities()` (no extra DB call); `AppUser` entity loaded only to set FK.
4. **409 for "already linked to self"** — not a silent 200; aligns with requirement wording.
5. **HTTP method open for api-designer**: `POST /api/players/{id}/link-me + 200` (as specified) vs `PATCH /api/players/{id}/link-me + 200` (project convention). Recommend PATCH.
6. **Cache eviction**: evict both `players` and `playerProfile` caches, matching `updatePlayer` pattern.
7. **`updatedBy`** stamped with `principal.username()`.
8. **Inactive players** may be linked — no restriction on `isActive` for this operation.

### Anything the Next Agent Must Know

- `UserPrincipal` is a **Java Record** — access user ID via `principal.id()` and role via `principal.authorities()`.
- `PlayerRepository` already has both `existsByUserId(Long)` and `findByUserId(Long)` — no repository changes needed.
- `UserRepository` is already injected into `PlayerService` — no constructor changes needed.
- The controller must import `org.springframework.security.core.annotation.AuthenticationPrincipal` and `pt.rics.demo.football.security.UserPrincipal`.
- Existing controller tests use `SecurityMockMvcRequestPostProcessors.user(...)` for authentication — follow the same pattern.
- `BusinessException.forbidden()` → HTTP 403; `BusinessException.conflict()` → HTTP 409 (verified in `BusinessException.java`).
- The **check order in `linkMe` service method matters**: role check → player lookup → player-already-linked-to-self → player-already-linked-to-other → caller-already-linked-elsewhere → load user entity → save.

