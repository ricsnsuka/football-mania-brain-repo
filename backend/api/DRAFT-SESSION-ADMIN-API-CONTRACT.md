# Draft Session Admin Endpoints — API Contract
**Date:** 2026-05-08
**Version:** v4.1.x
**Status:** DRAFT

## Overview

Two new **ADMIN-ONLY** endpoints on the existing `DraftSessionController`
(`@RequestMapping("/api/draft-sessions")`):

1. `GET /api/draft-sessions/summary` — lightweight list of **all** draft sessions (no player lists).
2. `DELETE /api/draft-sessions/{id}/purge` — **hard delete** (removes the DB row).

Both are gated by `@PreAuthorize("hasRole('ADMIN')")` — stricter than the existing
create/cancel/confirm endpoints which allow `MANAGER`.

> **Path-collision note:** The existing `GET /api/draft-sessions` (authenticated, HEAVY
> `DraftSessionDTO`) and `DELETE /api/draft-sessions/{id}` (soft-cancel → status `CANCELLED`)
> are **unchanged**. The new paths (`/summary`, `/{id}/purge`) are additive and do not collide.

---

## Endpoints

### GET /api/draft-sessions/summary

**Description:** Returns a lightweight summary of **every** draft session (all statuses),
suitable for an admin management/overview table. Excludes the heavy team/remaining player
lists carried by `DraftSessionDTO`.
**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`
**Tags:** `Draft Sessions`

#### Request

No path variables, no query parameters, no request body.

> Ordering recommendation: newest first (`createdAt` DESC). Non-paginated to match the
> existing `GET /api/draft-sessions` list behavior (repository `findAll`). If the dataset
> is expected to grow large, pagination can be added later as a non-breaking enhancement.

#### Response

**Success:** `200 OK`
**Body:** `List<DraftSessionSummaryDTO>`

```json
[
  {
    "id": 42,
    "matchPlanId": 7,
    "matchPlanTitle": "Friday Night 7-a-side",
    "status": "OPEN",
    "captainAName": "Alice",
    "captainBName": "Bob",
    "currentTurn": "A",
    "totalPlayers": 14,
    "picksRemaining": 8,
    "createdBy": "admin",
    "expiresAt": "2026-05-08T20:00:00Z",
    "createdAt": "2026-05-08T18:30:00Z",
    "updatedAt": "2026-05-08T18:45:00Z"
  }
]
```

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401 | Not authenticated |
| 403 | Authenticated but not `ADMIN` |

---

### DELETE /api/draft-sessions/{id}/purge

**Description:** **Hard-deletes** a draft session, permanently removing the row (and its
`@ElementCollection` child rows: `draft_session_team_a`, `draft_session_team_b`,
`draft_session_remaining`). This is distinct from the existing soft `DELETE /{id}` which only
sets status = `CANCELLED`. Intended for admin cleanup of stale/cancelled sessions.
**Authorization:** `@PreAuthorize("hasRole('ADMIN')")`
**Tags:** `Draft Sessions`

#### Request

**Path Variables:**
- `{id}` — `Long` — Draft session ID (required)

No request body.

#### Response

**Success:** `204 No Content` (empty body)

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401 | Not authenticated |
| 403 | Authenticated but not `ADMIN` |
| 404 | No draft session exists with the given `id` → `ResourceNotFoundException.of("DraftSession", id)` |
| 409 | Session status is `CONVERTED` (linked to a created Match) → `BusinessException.conflict(...)` — see business rule below |

#### Business Rules

- **BR-1 (recommended): Block hard-delete of `CONVERTED` sessions.** A `CONVERTED` session
  produced a real `Match`; purging it would destroy the audit trail linking the match back to
  its draft. Reject with `409 Conflict` via
  `BusinessException.conflict("Cannot purge a CONVERTED draft session (id=" + id + ") — it is linked to a created match")`.
  All other statuses (`OPEN`, `COMPLETED`, `CANCELLED`) may be purged.
  - Rationale: `OPEN`/`COMPLETED` purge = discard an in-progress/finished-but-unconfirmed draft;
    `CANCELLED` purge = normal cleanup. None of these have downstream `Match` rows.
- **BR-2 (recommended): Notify active watchers.** If the purged session had an SSE stream, publish
  a terminal event (reuse the existing `DraftStateChangedEvent` pattern, e.g. a `PURGED`/`DELETED`
  event type) so subscribed clients close their stream cleanly. This mirrors how `cancelDraftSession`
  publishes a `CANCELLED` event. *(Optional but consistent with existing behavior.)*
- **Deletion mechanics:** Use `draftSessionRepository.delete(session)` (or `deleteById` after the
  existence check). Because the collection tables are mapped via `@ElementCollection`, JPA removes
  the child rows automatically — no manual cleanup required.

---

## New DTO

### DraftSessionSummaryDTO

New Java **record** in `pt.rics.demo.football.dto`. Lightweight projection of `DraftSession`
(no `DraftPlayerDTO` lists).

```java
public record DraftSessionSummaryDTO(
        Long id,                 // never null
        Long matchPlanId,        // never null (matchPlan is @Column(nullable = false))
        String matchPlanTitle,   // never null — from MatchPlan.getTitle()
        String status,           // never null — DraftStatus.name(): OPEN|COMPLETED|CANCELLED|CONVERTED
        String captainAName,     // never null — Player.getName()
        String captainBName,     // never null — Player.getName()
        String currentTurn,      // NULLABLE — "A"/"B" when OPEN, null otherwise (matches DraftSessionDTO convention)
        int totalPlayers,        // teamA + teamB + remaining sizes
        int picksRemaining,      // remainingPlayerIds size
        String createdBy,        // NULLABLE — createdBy column has no NOT NULL constraint
        Instant expiresAt,       // NULLABLE — only set for time-boxed drafts
        Instant createdAt,       // never null
        Instant updatedAt        // never null
) {}
```

**Field notes / nullability:**
| Field | Type | Nullable | Source |
|-------|------|----------|--------|
| `id` | `Long` | no | `session.getId()` |
| `matchPlanId` | `Long` | no | `session.getMatchPlan().getId()` |
| `matchPlanTitle` | `String` | no | `session.getMatchPlan().getTitle()` |
| `status` | `String` | no | `session.getStatus().name()` |
| `captainAName` | `String` | no | `session.getCaptainA().getName()` |
| `captainBName` | `String` | no | `session.getCaptainB().getName()` |
| `currentTurn` | `String` | **yes** | `status == OPEN ? getCurrentTurn() : null` |
| `totalPlayers` | `int` | no | `teamA.size() + teamB.size() + remaining.size()` |
| `picksRemaining` | `int` | no | `session.getRemainingPlayerIds().size()` |
| `createdBy` | `String` | **yes** | `session.getCreatedBy()` |
| `expiresAt` | `Instant` | **yes** | `session.getExpiresAt()` |
| `createdAt` | `Instant` | no | `session.getCreatedAt()` |
| `updatedAt` | `Instant` | no | `session.getUpdatedAt()` |

> **Efficiency note for implementer:** The summary only needs captain names and the three list
> **sizes** (`teamAPlayerIds.size()`, `teamBPlayerIds.size()`, `remainingPlayerIds.size()`) — it does
> **not** need to batch-load every `Player` entity like `toDto(...)` does. Captain names come from the
> `captainA`/`captainB` associations. Prefer a dedicated lightweight mapper method
> (e.g. `toSummaryDto(DraftSession)`) rather than reusing the heavy `toDto` + `loadAllPlayers` path.

---

## Service & Controller changes (for dev-assistant)

**`DraftSessionService`** — add two methods (no service interface; use `.toList()`, exception factories):
```java
@Transactional(readOnly = true)
public List<DraftSessionSummaryDTO> getAllDraftSessionSummaries() { ... }   // findAll().stream().map(this::toSummaryDto).toList()

@Transactional
public void purgeDraftSession(Long id) {
    DraftSession session = draftSessionRepository.findById(id)
            .orElseThrow(() -> ResourceNotFoundException.of("DraftSession", id));
    if (session.getStatus() == DraftStatus.CONVERTED) {
        throw BusinessException.conflict("Cannot purge a CONVERTED draft session (id=" + id + ") — it is linked to a created match");
    }
    // optional: publish DraftStateChangedEvent(id, "PURGED"/"DELETED", ...) before delete
    draftSessionRepository.delete(session);
}
```

**`DraftSessionController`** — add two handlers:
```java
@GetMapping("/summary")
@Operation(summary = "List all draft sessions as lightweight summaries (ADMIN only)")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<List<DraftSessionSummaryDTO>> getAllDraftSessionSummaries() { ... }

@DeleteMapping("/{id}/purge")
@Operation(summary = "Hard-delete a draft session row (ADMIN only)")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Void> purgeDraftSession(@PathVariable Long id) {
    draftSessionService.purgeDraftSession(id);
    return ResponseEntity.noContent().build();
}
```

No new repository methods required (`findAll`, `findById`, `delete` already available).

---

## Breaking Changes

- [x] **No breaking changes.** Both endpoints are additive. Existing `GET /api/draft-sessions`
  (heavy DTO) and `DELETE /api/draft-sessions/{id}` (soft-cancel) are untouched.
- New DTO `DraftSessionSummaryDTO` is net-new; nothing depends on it yet.

## Frontend Migration Notes

- FE admin views can switch their draft-session **list/overview table** to
  `GET /api/draft-sessions/summary` for a lighter payload (no team/remaining player arrays).
- A new **"purge/permanently delete"** action should call `DELETE /api/draft-sessions/{id}/purge`
  and must be visually/UX-distinct from the existing **"cancel"** action (`DELETE /{id}`), since
  purge is irreversible and hard-deletes the row.
- Handle `409 Conflict` on purge for `CONVERTED` sessions (show "cannot delete — linked to a match").
- Both new endpoints require an `ADMIN` role — hide the controls from everyone else.

---

## Summary for Next Agent

**Endpoint 1 — list summaries**
- `GET /api/draft-sessions/summary`
- `@PreAuthorize("hasRole('ADMIN')")`
- Returns `200 OK` + `List<DraftSessionSummaryDTO>` (all statuses, recommend `createdAt` DESC, non-paginated)
- Errors: `401`, `403`

**Endpoint 2 — hard delete**
- `DELETE /api/draft-sessions/{id}/purge`
- `@PreAuthorize("hasRole('ADMIN')")`
- Returns `204 No Content`
- Errors: `401`, `403`, `404` (`ResourceNotFoundException.of("DraftSession", id)`), `409` (block `CONVERTED` via `BusinessException.conflict(...)`)
- Deletion via `draftSessionRepository.delete(session)`; `@ElementCollection` child rows auto-removed. Optional SSE `PURGED` event.

**New DTO — `DraftSessionSummaryDTO` (Java record, `pt.rics.demo.football.dto`)**
```
Long id (non-null), Long matchPlanId (non-null), String matchPlanTitle (non-null),
String status (non-null), String captainAName (non-null), String captainBName (non-null),
String currentTurn (nullable, only when OPEN), int totalPlayers, int picksRemaining,
String createdBy (nullable), Instant expiresAt (nullable), Instant createdAt (non-null), Instant updatedAt (non-null)
```
Add lightweight `toSummaryDto(DraftSession)` mapper — use captain associations + list **sizes**, do NOT batch-load all Players.

**Business rules:** BR-1 block purge of `CONVERTED` (409); BR-2 optional SSE terminal event on purge.
**Breaking changes:** none — additive; existing heavy `GET /api/draft-sessions` and soft-cancel `DELETE /{id}` unchanged.
**Ready for:** dev-assistant (implement DTO + service + controller). No DB migration needed (no schema change).

