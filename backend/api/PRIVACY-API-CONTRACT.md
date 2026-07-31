# Privacy (GDPR Data-Subject Rights) — API Contract

**Date:** 2026-07-28
**Version:** v1.0.0 (Phase 0)
**Status:** APPROVED — implemented

---

## Scope

Four endpoints serving the two data-subject rights that need code rather than a policy document:
**access/portability** (GDPR Art. 15 & 20) and **erasure** (Art. 17). Purely additive — no
existing endpoint, DTO or response shape changes.

Base path: `/api/privacy`

| Method | Path | Right served | Auth |
|--------|------|--------------|------|
| `GET` | `/api/privacy/me/export` | Access, portability | Any authenticated |
| `DELETE` | `/api/privacy/me` | Erasure | Any authenticated |
| `GET` | `/api/privacy/players/{id}/export` | Access, portability (on behalf of) | `ADMIN` only |
| `DELETE` | `/api/privacy/players/{id}` | Erasure (on behalf of) | `ADMIN` only |

---

## Design decisions

### Why `/me` and `/players/{id}` are separate paths, not one endpoint with a role check

| Option | For | Against |
|--------|-----|---------|
| One endpoint, `?playerId=` optional, role-gated | Fewer routes | The subject becomes a **request parameter**. Every future refactor has to re-derive that omitting it means "me" and supplying it means "someone else", and getting that wrong is an IDOR that leaks a full personal record. |
| Two paths ✅ | On `/me` the subject *is* the authenticated principal — there is no parameter a caller can tamper with. The ADMIN-only rule sits on a separate route where it is the only thing that route does. | Two handlers instead of one. |

### Why the admin endpoints exist at all

A player created by an admin may have **no linked account**. That person has the same rights and
no way to log in and exercise them, so someone must action the request on their behalf. Without
these routes the rights would be unservable for exactly the people most likely to ask.

### Why `ADMIN` and not `MANAGER`

`MANAGER` can create and edit players. It cannot read a complete personal record of another
individual, nor perform an irreversible erasure. Both are `ADMIN`-only.

### Why erasure is `DELETE` returning `204` and not `POST /erase`

It is a deletion of personal data from the caller's point of view — `DELETE` is the honest verb —
and it is idempotent in effect at the HTTP level even though a repeat returns `409` (see the error
table). No body is returned because there is nothing left to return.

---

## Endpoints

### GET /api/privacy/me/export

**Description:** Returns everything the system holds about the calling user, as a single JSON
document offered as a file download. The subject is taken **exclusively** from the JWT principal
(`UserPrincipal.id()`) — it is never accepted as a path variable, query parameter or body field.

**Authorization:** `@PreAuthorize("isAuthenticated()")`

**Tags:** `Privacy`

#### Request

**Path Variables:** None
**Query Parameters:** None
**Request Body:** None

**Headers:**

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer <JWT-token>` | Yes |

#### Response

**Success:** `200 OK`
**Body:** `PersonalDataExportDTO` (see below)

**Response headers:**

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `Content-Disposition` | `attachment; filename="personal-data-export.json"` |

> `Content-Disposition: attachment` is deliberate: the point of Art. 20 is that the subject walks
> away with a file. See [Frontend Migration Notes](#frontend-migration-notes) — this changes how
> the frontend should trigger the request.

#### Error Responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `403 Forbidden` | No / invalid JWT | *(Spring Security filter — no body from this controller)* |
| `404 Not Found` | The JWT's user id no longer exists | `"User not found with id: {id}"` |

---

### DELETE /api/privacy/me

**Description:** Erases the calling user. The login account is deleted outright; the linked player
record, if any, is **anonymised in place**. Irreversible.

**Authorization:** `@PreAuthorize("isAuthenticated()")`

#### Request

Path variables, query parameters and body: none. `Authorization` header required.

#### Response

**Success:** `204 No Content` (empty body)

#### Error Responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `403 Forbidden` | Caller has `ROLE_ADMIN` | `"An ADMIN account cannot erase itself; another administrator must action this request"` |
| `403 Forbidden` | No / invalid JWT | *(Spring Security filter)* |
| `404 Not Found` | The JWT's user id no longer exists | `"User not found with id: {id}"` |
| `409 Conflict` | The linked player record has already been erased | `"Player id={id} has already been erased"` |

---

### GET /api/privacy/players/{id}/export

**Description:** Same document as `/me/export`, for an arbitrary player. For actioning a request
from someone who cannot make it themselves.

**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`

#### Request

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | Long | Yes | Target **player** id (not user id) |

#### Response

**Success:** `200 OK`, body `PersonalDataExportDTO`,
`Content-Disposition: attachment; filename="personal-data-export-{id}.json"`

#### Error Responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `403 Forbidden` | Caller is not `ADMIN` (includes `MANAGER` and users holding no roles) | *(Spring Security — no body)* |
| `404 Not Found` | Player `{id}` does not exist | `"Player not found with id: {id}"` |

---

### DELETE /api/privacy/players/{id}

**Description:** Erases the person behind a player record, and deletes their linked account if they
have one. Irreversible.

**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`

#### Response

**Success:** `204 No Content`

#### Error Responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `403 Forbidden` | Caller is not `ADMIN` | *(Spring Security — no body)* |
| `404 Not Found` | Player `{id}` does not exist | `"Player not found with id: {id}"` |
| `409 Conflict` | Already erased | `"Player id={id} has already been erased"` |

All errors follow the project's standard `ApiError` shape:

```json
{
  "timestamp": "2026-07-28T10:30:00Z",
  "status": 409,
  "error": "Conflict",
  "message": "Player id=5 has already been erased",
  "path": "/api/privacy/players/5",
  "violations": []
}
```

---

## New / Modified DTOs

| DTO | Change |
|-----|--------|
| `PersonalDataExportDTO` | **New** — response of both export endpoints |
| `PlayerDTO` | None |
| `UserDTO` | None |

### PersonalDataExportDTO

```json
{
  "generatedAt": "2026-07-28T12:00:00Z",
  "account": {
    "id": 10,
    "username": "j.silva",
    "email": "j.silva@example.com",
    "firstName": "Joao",
    "lastName": "Silva",
    "roles": [],
    "active": true,
    "createdAt": "2026-01-01T00:00:00Z"
  },
  "player": {
    "id": 5,
    "name": "Joao Silva",
    "phoneNumber": "+351912345678",
    "active": true,
    "skillRating": 7.4,
    "baseSkillRating": 7,
    "currentStreak": 2,
    "longestStreak": 5,
    "totalMatchesPlayed": 30,
    "totalGoals": 12,
    "totalAssists": 9,
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-07-01T00:00:00Z"
  },
  "matches": [
    {
      "matchId": 100,
      "matchDescription": "Friday night",
      "matchDate": "2026-05-01T19:00:00Z",
      "location": "Pitch 2",
      "teamName": "Reds",
      "goals": 2,
      "assists": 1,
      "ownGoals": 0,
      "rating": 7.8,
      "mvp": true,
      "result": "WIN"
    }
  ],
  "goals": [
    {
      "matchId": 100,
      "matchDate": "2026-05-01T19:00:00Z",
      "role": "SCORER",
      "minute": 12,
      "ownGoal": false,
      "penalty": false,
      "description": null
    }
  ],
  "ratingHistory": [
    {
      "matchId": 100,
      "ratingBefore": 7.2,
      "ratingAfter": 7.4,
      "changeAmount": 0.2,
      "reason": "Match 100 result",
      "createdAt": "2026-05-01T21:00:00Z"
    }
  ],
  "availability": [
    {
      "matchPlanId": 20,
      "matchPlanTitle": "Friday 7-a-side",
      "proposedDate": "2026-05-01",
      "status": "CONFIRMED",
      "notes": "Can only make the second half",
      "confirmedAt": "2026-04-28T10:00:00Z"
    }
  ]
}
```

#### Nullability rules — read these before writing the TypeScript types

| Field | Nullable | When |
|-------|----------|------|
| `account` | **Yes** | The player has no linked login (created by an admin). |
| `player` | **Yes** | The account is not linked to a player yet. |
| `matches`, `goals`, `ratingHistory`, `availability` | **No** | Always an array — `[]` when empty, never `null` or absent. A consumer should not have to distinguish "no matches" from "section omitted". |
| `player.phoneNumber` | Yes | Never supplied. |
| `matches[].rating` | Yes | Match not yet completed / not rated. |
| `matches[].result` | Yes | Match not yet completed. |
| `matches[].matchDate`, `location`, `matchDescription` | Yes | Optional on the match. |
| `goals[].minute`, `description` | Yes | Optional on the goal. |
| `ratingHistory[].matchId` | Yes | **Season-transition rows carry no match.** |
| `availability[].notes`, `confirmedAt` | Yes | No note written / not confirmed. |

At least one of `account` / `player` is always non-null.

#### Field notes

- `matches[].result` is the `MatchResult` enum: `WIN` \| `LOSS` \| `DRAW`.
- `availability[].status` is the `ConfirmationStatus` enum: `CONFIRMED` \| `DECLINED` \| `PENDING`.
- `account.roles` is an array of `ORGANIZER` \| `MANAGER` \| `ADMIN`, sorted; empty for an account holding none.
- `goals[].role` is `SCORER` \| `ASSISTER` — a player holds only one role per goal.
- `account.password` is **deliberately absent.** A bcrypt hash is not portable data.

#### Scoping rule the frontend should not try to work around

The export is scoped to **data about this person**, not **data this person can see**:

- A match is represented by the subject's own line in it — their team name, their goals, their
  rating. Never the full scoresheet, never the other participants.
- `goals[].role` names the subject's part; the counterpart scorer/assister is not named.

If a UI needs the full scoresheet, call `GET /api/matches/{id}` — that endpoint already applies
the roster's own visibility rules. Do not extend the export to carry it; one person's export must
not become a way to harvest everyone else's record.

---

## What erasure actually changes

Relevant to the frontend because **erased players remain visible in listings**.

| Field | After erasure |
|-------|---------------|
| `players.name` | `"Deleted player #{id}"` |
| `players.phone_number` | `null` |
| `players.is_active` | `false` |
| `players.user_id` | `null` |
| `players.anonymized_at` | Timestamp (not currently exposed on `PlayerDTO`) |
| `users` row | **Deleted** |
| Match results, goals, ratings, streaks, career totals | **Unchanged** |

Erasure anonymises rather than deletes because `player_stats`, `goals` and `skill_rating_history`
all reference `players(id)` with `ON DELETE CASCADE` — deleting the row would delete the goals that
player scored, and other players' scorelines would stop matching their goal lists.

**Consequences for the UI:**

- A `PlayerDTO` named `Deleted player #5` with `isActive: false` and `linkedUserId: null` is a
  normal, expected record. It will appear in `GET /api/players` unless filtered with `?active=true`.
- It still carries real `skillRating`, `totalGoals` etc. — those belong to the match history, not
  to a person, and leaderboards should keep counting them.
- Historical match detail views will show the tombstone name in team line-ups. That is correct.
- After a successful self-erasure the caller's JWT is still syntactically valid but the account is
  gone. **The frontend must clear the token and redirect to login immediately** — see below.

---

## Service Method Signatures

```java
@Transactional(readOnly = true)
public PersonalDataExportDTO exportForUser(Long userId);

@Transactional(readOnly = true)
public PersonalDataExportDTO exportForPlayer(Long playerId);

/**
 * @throws BusinessException (403) if the account is ADMIN
 * @throws BusinessException (409) if the linked player is already erased
 */
@Transactional
@Caching(evict = {
    @CacheEvict(value = "players",       allEntries = true),
    @CacheEvict(value = "playerProfile", allEntries = true)
})
public void eraseByUser(Long userId);

/** @throws BusinessException (409) if already erased */
@Transactional
@Caching(evict = {
    @CacheEvict(value = "players",       allEntries = true),
    @CacheEvict(value = "playerProfile", allEntries = true)
})
public void erasePlayer(Long playerId);
```

Both erasure methods evict `players` and `playerProfile` in full, matching every other
`Player`-mutating write in `PlayerService` — the tombstone name and cleared phone number must not
be served from a stale cache.

---

## Controller Method Signatures

```java
@GetMapping(value = "/me/export", produces = MediaType.APPLICATION_JSON_VALUE)
@PreAuthorize("isAuthenticated()")
public ResponseEntity<PersonalDataExportDTO> exportMyData(Authentication authentication);

@DeleteMapping("/me")
@PreAuthorize("isAuthenticated()")
public ResponseEntity<Void> eraseMyData(Authentication authentication);

@GetMapping(value = "/players/{id}/export", produces = MediaType.APPLICATION_JSON_VALUE)
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<PersonalDataExportDTO> exportPlayerData(@PathVariable Long id);

@DeleteMapping("/players/{id}")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Void> erasePlayerData(@PathVariable Long id);
```

---

## Business Rule Execution Order

`PrivacyService.eraseByUser()` — each failed check throws immediately:

| Step | Check | Exception |
|------|-------|-----------|
| 1 | `userRepository.findById(userId)` empty | `ResourceNotFoundException.of("User", userId)` |
| 2 | `user.getRole() == ADMIN` | `BusinessException.forbidden(...)` |
| 3 | Resolve linked player via `findByUserId` (may be absent) | — |
| 4 | `player.getAnonymizedAt() != null` | `BusinessException.conflict(...)` |
| 5 | Anonymise the player row and save | — |
| 6 | Scrub the username from `match_plans.created_by` and `draft_sessions.created_by` | — |
| 7 | Delete the `users` row | — |

`erasePlayer()` is steps 1 (by player), 4, 5, 6, 7 — no admin self-erasure check, because the
caller is not the subject.

Step 6 runs **before** step 7 deliberately: once the `users` row is gone the username survives only
in those audit columns, which is exactly what must not remain.

---

## Breaking Changes

- [x] **No breaking changes.** Four new endpoints and one new response DTO. No existing endpoint,
  DTO, field or status code is modified.

**One behavioural note that is not a breaking change but is easy to miss:** once erasure is
exposed in the UI, `GET /api/players` can start returning records named `Deleted player #{id}`.
Any frontend code that assumes a player name is a real human name — initials avatars, alphabetical
grouping, search — should be checked against that string.

---

## Frontend Migration Notes

### 1. Downloading the export

The response carries `Content-Disposition: attachment`, so **do not render it in the app**. Fetch
it and hand the browser a blob:

```ts
async function downloadMyData(token: string) {
  const res = await fetch('/api/privacy/me/export', {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!res.ok) throw new Error(`Export failed: ${res.status}`);

  const blob = await res.blob();
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'personal-data-export.json';
  a.click();
  URL.revokeObjectURL(url);
}
```

A plain `<a href="/api/privacy/me/export" download>` will **not** work — the endpoint needs the
`Authorization` header, which a navigation request cannot carry.

### 2. Self-erasure flow

```
DELETE /api/privacy/me
Authorization: Bearer <JWT>
(no request body)

→ 204   erased
→ 403   caller is ADMIN (show `message`)
→ 409   already erased (show `message`)
```

Required sequence:

1. **Two-step confirmation.** This is irreversible and cannot be undone by support. Require the
   user to type their username, not just click OK.
2. State plainly what is kept: *"Your match results and statistics stay in the group's history
   without your name attached. Your account, name and phone number are deleted."* Users who expect
   a full wipe should not discover the difference afterwards.
3. On `204`: **clear the stored JWT, reset the Zustand auth store, and redirect to login.** The
   token is still syntactically valid but the account behind it is gone — every subsequent request
   will fail in a way that looks like a bug.
4. On `403`: the caller is an admin. Show the message; there is no self-service path.

Suggested placement: a "Privacy & your data" section on the profile page with both actions,
export first.

### 3. Admin flow

`GET|DELETE /api/privacy/players/{id}` from the existing player modal, in a dedicated danger zone
alongside the existing delete/unlink zones (`.player-modal-delete-zone` is the established
pattern). Same confirmation requirements. Note the erase button should stay available for players
who already have match stats — unlike `DELETE /api/players/{id}`, which is blocked in that case,
erasure is specifically designed to work when history exists.

### 4. Rendering erased players

Erased players remain in `GET /api/players`. Detect them by name prefix (`Deleted player #`) or,
more robustly, by `isActive === false && linkedUserId === null && email === null`. Consider a
muted style in listings and suppressing them from selection UIs (match creation, draft captains) —
`?active=true` already does this for endpoints that support the filter.

### 5. Privacy policy page — still outstanding

Mandatory for both app stores in Phase 4 and for any public signup. It must state: controller
identity, what is collected, why, the lawful basis, retention, third parties (currently none),
the rights above and how to exercise them, and a contact address. Content requirements are in
`docs/features/PRIVACY_AND_DATA_PROTECTION.md`.

---

## Security Notes

| Concern | Resolution |
|---------|------------|
| IDOR on `/me` endpoints | The subject id comes only from `UserPrincipal.id()` (JWT). There is no path variable, query parameter or body field a caller could alter to reach another person's record. |
| Roster harvesting via export | The export is scoped to data *about* the subject. Other players are never named — no scoresheet, no counterpart on a goal. This mirrors the `PlayerPiiPolicy` rule on the player endpoints. |
| Privilege escalation on the admin endpoints | `hasRole('ADMIN')`, verified in `PrivacyControllerTest` against `MANAGER` and a role-less user for all four routes. |
| Last-admin lockout | An `ADMIN` cannot erase itself (403). The request would irreversibly delete the credentials that administer the deployment. |
| Password hash disclosure | Excluded from the export DTO by construction — the field does not exist on the record. |
| Replay of a valid token after erasure | The JWT remains syntactically valid until expiry but resolves to no user. Acceptable for a stateless JWT design; the frontend must clear the token on `204`. |
| CSRF | Not applicable — stateless JWT, CSRF disabled in `SecurityConfig`. |

---

## Test Coverage

| Test | Covers |
|------|--------|
| `PrivacyServiceTest` (18 cases) | Export shape for linked / unlinked / account-only subjects, goal role labelling, matchless rating rows, availability notes; erasure field-by-field, statistic preservation, audit-actor scrubbing, `createdBy` scrub only when self-created, double-erasure `409`, admin self-erasure `403`, both `404`s |
| `PrivacyControllerTest` (14 cases) | Principal-derived subject id, `Content-Disposition`, anonymous rejection, `MANAGER` and role-less rejection on all four admin routes, exception-to-status mapping (`403` / `404` / `409`) |
