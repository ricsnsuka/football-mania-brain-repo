# Player Self-Link — API Contract
**Date:** 2026-05-15
**Version:** v4.x.x
**Status:** APPROVED

---

## HTTP Verb Decision — POST vs PATCH

**Decision: `POST /api/players/{id}/link-me` → `200 OK`**

| Option | For | Against |
|--------|-----|---------|
| `PATCH /{id}/link-me` | Mirrors `PATCH /{id}/status` pattern | Every other PATCH in this project has a request body; this one has none. PATCH with no body is semantically odd. |
| `POST /{id}/link-me` ✅ | Standard REST pattern for named action sub-resources with no body (e.g. `/activate`, `/complete`). Explicitly requested by the original requirement. | `POST` normally returns `201` — but `200` is correct here because no new resource is created. |

**Justification in full:**

1. `PATCH /{id}/status` has a request body (`PlayerStatusDTO.isActive`). The partial-update semantics
   require a body to indicate *what* to change. `link-me` has no body — the only input is the
   path variable and the JWT principal. Applying PATCH to a bodyless request is non-standard.
2. Named action sub-resource endpoints (`/complete`, `/link-me`, `/activate`) are idiomatically
   expressed with `POST` in REST APIs when they carry no request body.
3. The user's original requirement explicitly specifies `POST`.
4. `200 OK` (not `201 Created`) is correct because no new resource is created; an existing
   `Player` entity is mutated.

---

## Endpoints

### POST /api/players/{id}/link-me

**Description:** Links the calling user's own account to the specified player profile.
The user ID is extracted exclusively from the JWT principal — it is never accepted in a
request body or path variable. On success the player's `user_id` FK is set and the updated
`PlayerDTO` is returned.

**Authorization:** `@PreAuthorize("isAuthenticated()")`
> Role enforcement for `ADMIN` is additionally performed inside the service method
> (role check before any DB access) to ensure it is always enforced even if the annotation
> is bypassed in tests.

**Tags:** `Players`

---

#### Request

**Path Variables:**

| Name | Type | Required | Description      |
|------|------|----------|------------------|
| `id` | Long | Yes      | Target player ID |

**Query Parameters:** None

**Request Body:** None

**Headers:**

| Header          | Value                   | Required |
|-----------------|-------------------------|----------|
| `Authorization` | `Bearer <JWT-token>`    | Yes      |

---

#### Response

**Success:** `200 OK`

**Body:** `PlayerDTO`

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
  "updatedBy": "ricardo",
  "createdAt": "2026-01-01T00:00:00Z",
  "updatedAt": "2026-05-15T10:00:00Z"
}
```

> `linkedUserId` will reflect the newly linked user ID.  
> `updatedBy` will be stamped with `principal.username()`.  
> `email` will be populated from the linked `AppUser.email`.

---

#### Error Responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `401 Unauthorized` | No `Authorization` header / invalid JWT | *(handled by Spring Security filter — no body from this controller)* |
| `403 Forbidden` | Caller has `ROLE_ADMIN` | `"ADMIN accounts cannot be linked to a player"` |
| `404 Not Found` | Player `{id}` does not exist | `"Player with id {id} not found"` |
| `409 Conflict` | Player already linked to **this** calling user | `"Player is already linked to your account"` |
| `409 Conflict` | Player already linked to a **different** user | `"Player is already linked to another user"` |
| `409 Conflict` | Calling user already linked to a **different** player | `"You are already linked to another player"` |

All errors follow the project's standard `ApiError` shape:

```json
{
  "timestamp": "2026-05-15T10:30:00Z",
  "status": 409,
  "error": "Conflict",
  "message": "Player is already linked to your account",
  "path": "/api/players/7/link-me",
  "violations": []
}
```

---

## New / Modified DTOs

**No new DTOs.** This endpoint uses no request body and returns the existing `PlayerDTO`.

| DTO | Change |
|-----|--------|
| `PlayerDTO` | None — all required fields (`linkedUserId`, `email`) already present |
| `PlayerCreateDTO` | None |
| `PlayerUpdateDTO` | None |

---

## Service Method Signature

```java
/**
 * Links the calling user's own account to the given player profile.
 *
 * @param playerId  the target player ID (from path variable)
 * @param principal the authenticated caller (from JWT)
 * @return the updated PlayerDTO with linkedUserId and email populated
 * @throws BusinessException (403) if caller is ADMIN
 * @throws ResourceNotFoundException (404) if player not found
 * @throws BusinessException (409) if any link conflict exists
 */
@Caching(evict = {
    @CacheEvict(value = "players",       allEntries = true),
    @CacheEvict(value = "playerProfile", allEntries = true)
})
public PlayerDTO linkMe(Long playerId, UserPrincipal principal)
```

---

## Controller Method Signature

```java
@PostMapping("/{id}/link-me")
@Operation(
    summary     = "Link the calling user to a player profile (self-link)",
    description = "Allows any authenticated non-admin user to associate their own account "
                + "with an existing player record. User ID is derived from the JWT principal — "
                + "no request body is accepted. Returns the updated PlayerDTO on success."
)
@ApiResponse(responseCode = "200",  description = "Player successfully linked to calling user")
@ApiResponse(responseCode = "403",  description = "Caller is ADMIN or unauthenticated")
@ApiResponse(responseCode = "404",  description = "Player not found")
@ApiResponse(responseCode = "409",  description = "Link conflict — see message for detail")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<PlayerDTO> linkMe(
        @Parameter(description = "Player ID to link to", required = true)
        @PathVariable Long id,
        @AuthenticationPrincipal UserPrincipal principal) {
    return ResponseEntity.ok(playerService.linkMe(id, principal));
}
```

---

## Service Business Rule Execution Order

The following checks are performed in `PlayerService.linkMe()` in this exact order
(each check that fails throws immediately — no further checks are performed):

| Step | Check | Exception |
|------|-------|-----------|
| 1 | `principal.authorities()` contains `ROLE_ADMIN` | `BusinessException.forbidden("ADMIN accounts cannot be linked to a player")` |
| 2 | `playerRepository.findById(playerId)` returns empty | `ResourceNotFoundException.of("Player", playerId)` |
| 3 | `player.getUser() != null` AND `player.getUser().getId().equals(principal.id())` | `BusinessException.conflict("Player is already linked to your account")` |
| 4 | `player.getUser() != null` (different user) | `BusinessException.conflict("Player is already linked to another user")` |
| 5 | `playerRepository.findByUserId(principal.id())` returns a **different** player | `BusinessException.conflict("You are already linked to another player")` |
| 6 | Load `AppUser` via `userRepository.findById(principal.id())` | `ResourceNotFoundException.of("User", principal.id())` *(defensive — should never occur if JWT is valid)* |
| 7 | `player.setUser(linkedUser)`, `player.setUpdatedBy(principal.username())`, `playerRepository.save(player)` | — |
| 8 | `return playerMapper.toDto(savedPlayer)` | — |

---

## Caching Strategy

Identical to `updatePlayer` — both write operations that mutate the `Player` entity:

```java
@Caching(evict = {
    @CacheEvict(value = "players",       allEntries = true),
    @CacheEvict(value = "playerProfile", allEntries = true)
})
```

**Rationale:** `linkMe` changes `player.user` which affects both the list cache (`players`)
and the profile cache (`playerProfile`) since `PlayerDTO.email` and `PlayerDTO.linkedUserId`
are derived from the linked user. Both caches must be invalidated.

---

## Breaking Changes

- [x] **No breaking changes** — new endpoint added; no existing endpoints, DTOs, or response
  shapes are modified. The feature is purely additive.

---

## Frontend Migration Notes

New endpoint for the frontend onboarding flow:

```
POST /api/players/{id}/link-me
Authorization: Bearer <JWT>
(no request body)

→ 200 OK  { PlayerDTO }
→ 403     ADMIN accounts cannot link
→ 404     Player not found
→ 409     Conflict (see message)
```

**Usage pattern:**
1. User browses the player list (`GET /api/players`)
2. User identifies their player profile by name
3. User calls `POST /api/players/{id}/link-me` with their auth token
4. On `200` → update local state with returned `PlayerDTO` (note `linkedUserId` and `email` now populated)
5. On `409` → display the `message` field from `ApiError` body to the user

---

## Security Notes

| Concern | Resolution |
|---------|------------|
| IDOR (horizontal privilege escalation) | User ID is **only** taken from `UserPrincipal.id()` (JWT) — never from path or body. A caller cannot link another user's account. |
| CSRF | Not applicable — API is stateless JWT; CSRF is disabled in `SecurityConfig`. |
| ADMIN bypass | Role check is in the **service layer**, not only in `@PreAuthorize`, so it cannot be bypassed by test setup or annotation stripping. |
| Unauthenticated access | `@PreAuthorize("isAuthenticated()")` on the controller method. Spring Security filter rejects before controller is reached if no valid JWT. |

