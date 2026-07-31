# Draft Session Resume (SSE Reconnect) — API Contract
**Date:** 2026-05-08
**Version:** v1.x.x (patch — behavioral enhancement, no schema change)
**Status:** DRAFT

## Overview

Enable a client to **resume** a draft session after its SSE stream drops (inactivity, the
5-minute `SSE_TIMEOUT_MS`, network blip, tab reload, etc.). All session state is already
persisted, and the existing `GET /api/draft-sessions/{id}/events` endpoint already sends a
`CONNECTED` snapshot on every (re)connect, so OPEN/COMPLETED sessions already resume via the
documented client auto-reconnect.

**The gap being closed:** when a client reconnects to a session that reached a **TERMINAL**
state (`CANCELLED` / `CONVERTED`) *while it was disconnected*, the terminal broadcast already
fired to the now-gone emitters and `DraftSseEmitterRegistry` removed the registry entry. On
reconnect, `subscribeToEvents` sends `CONNECTED` (showing the terminal status) and then returns
an emitter that is **never completed** → the stream hangs until `SSE_TIMEOUT_MS` (5 min) and the
client never receives a close signal.

This contract makes reconnection robust for **all** statuses, with **no new REST path**.

> **Scope guardrails (from orchestrator):**
> - `@EnableScheduling` is **not** present (only `@EnableAsync`) → heartbeats/keep-alive are **out of scope**.
> - No new endpoint unless clearly justified — see the [decision](#new-endpoint-decision) below (conclusion: **not needed**).

---

## Endpoints

### GET /api/draft-sessions/{id}/events  *(behavior enhanced — path & signature unchanged)*

**Description:** Subscribe to the real-time draft session event stream via Server-Sent Events.
On every (re)connect the server sends an authoritative `CONNECTED` snapshot. When the session is
already in a terminal state at connect time, the server now also emits the matching terminal event
and closes the stream immediately — so a reconnecting client terminates cleanly instead of hanging.
**Authorization:** `@PreAuthorize("isAuthenticated()")` *(unchanged)*
**Tags:** `Draft Sessions`
**Produces:** `text/event-stream` *(unchanged)*

#### Request

**Path Variables:**
- `{id}` — Long — draft session ID

No query parameters. No request body. JWT bearer token required (as with every endpoint).

#### Response

**Success:** `200 OK`, `Content-Type: text/event-stream`
**Body:** a stream of SSE events; each `data:` payload is a full `DraftSessionDTO` JSON object.

Every event carries `retry: 3000` (3s client reconnect hint) — unchanged from current behavior.

**Error Responses:**
| Status | Trigger |
|--------|---------|
| 401 | Missing / expired JWT |
| 403 | Not authenticated |
| 404 | Session `{id}` does not exist (fast-fail **before** any emitter is created — **preserved**) |

---

## Enhanced `/events` Behavior — Event Ordering per Status

The status is read from the initial snapshot (`draftSessionService.getDraftSession(id)`).
`DraftStatus` values: `OPEN`, `COMPLETED`, `CANCELLED`, `CONVERTED`.

### 1. Non-existent session (any state) — **unchanged**
```
draftSessionService.getDraftSession(id)  → throws ResourceNotFoundException → HTTP 404
```
Fast-fail happens **before** any emitter is created or registered. No SSE stream opens.

### 2. OPEN / COMPLETED (non-terminal) — **unchanged**
```
1. register emitter in broadcast registry  (sseEmitterRegistry.subscribe(id))
2. send  event: CONNECTED   data: <DraftSessionDTO snapshot>
3. stream stays OPEN — future PICK / COMPLETED / CANCELLED / CONVERTED
   events arrive via the registry broadcast (AFTER_COMMIT).
```
Registering **before** sending `CONNECTED` is retained so no concurrently-committed event is missed.

### 3. CANCELLED / CONVERTED (already terminal at (re)connect) — **NEW**
```
1. create a standalone emitter — do NOT add it to the broadcast registry
2. send  event: CONNECTED   data: <DraftSessionDTO snapshot>   (authoritative terminal state)
3. send  event: <STATUS>    data: <same DraftSessionDTO snapshot>
         where <STATUS> is exactly "CANCELLED" or "CONVERTED" (matches session.status())
4. emitter.complete()   → stream closes immediately; client observes the terminal event + EOF
```

**Strict ordering guarantee:** `CONNECTED` is always sent **first** (so the client has the
authoritative baseline exactly as in the non-terminal path), then the single terminal event,
then completion. The client never has to special-case "CONNECTED with a terminal status but no
terminal event" — it will always receive the terminal event next and can close deterministically.

> **Why not register terminal reconnects in the broadcast registry?**
> A terminal session will never emit another `DraftStateChangedEvent`, so registration buys
> nothing and risks leaving a stale entry. Creating a standalone emitter and completing it
> keeps `DraftSseEmitterRegistry` clean and side-effect-free for terminal reconnects.

### Ordering summary table

| Status at connect | Emitter registered? | Event sequence | Stream after |
|-------------------|---------------------|----------------|--------------|
| *(missing)* | No | — (HTTP 404) | never opens |
| `OPEN` | Yes | `CONNECTED` → *(live events)* | stays open |
| `COMPLETED` | Yes | `CONNECTED` → *(later `CANCELLED`/`CONVERTED`)* | stays open |
| `CANCELLED` | **No** | `CONNECTED` → `CANCELLED` → *complete* | closes immediately |
| `CONVERTED` | **No** | `CONNECTED` → `CONVERTED` → *complete* | closes immediately |

---

## <a id="new-endpoint-decision"></a>New Endpoint Decision

**Decision: NO new endpoint is warranted.** Resume is fully satisfied by existing paths plus
the terminal-on-reconnect enhancement above:

- **`GET /api/draft-sessions/{id}`** — full-state rehydrate (heavy `DraftSessionDTO`), available
  to any authenticated user. A client can call this at any time to rebuild UI state without a stream.
- **`GET /api/draft-sessions/{id}/events`** — reconnect handles resume for live sessions
  (`CONNECTED` snapshot) and now cleanly terminates for already-terminal sessions.

Adding a dedicated `/resume` endpoint would duplicate the `CONNECTED` snapshot semantics that the
SSE stream already provides and violate the project's "no path unless justified" bias. The
`CONNECTED` event is itself the resume/rehydrate primitive.

---

## Frontend Resume / Rehydrate Flow (to document in the SSE guide)

```
Client                                                    Server
  │  (stream dropped: 5-min timeout / network / reload)      │
  │                                                          │
  │  GET /api/draft-sessions/{id}/events   (reconnect) ─────►│
  │  Authorization: Bearer <token>                           │
  │                                                          │
  │◄──── 200 text/event-stream                               │
  │                                                          │
  │  event: CONNECTED                                        │
  │  data: <DraftSessionDTO> (authoritative state) ◄─────────│
  │                                                          │
  ├─ if status == OPEN | COMPLETED:                          │
  │     render board from CONNECTED snapshot;                │
  │     keep stream open; resume picking via                 │
  │     POST /api/draft-sessions/{id}/pick                   │
  │                                                          │
  ├─ if status == CANCELLED | CONVERTED:                     │
  │     event: <STATUS>                                      │
  │     data: <DraftSessionDTO>                     ◄─────────│
  │     [server completes stream]                            │
  │     client closes EventSource; show terminal UI          │
  │     (CANCELLED → notice/redirect, CONVERTED → matches)   │
```

**Client rules:**
1. On reconnect, always wait for `CONNECTED` and treat it as the authoritative baseline (no
   separate `GET /{id}` call is required, though `GET /{id}` remains available for manual rehydrate).
2. Handle a terminal event (`CANCELLED` / `CONVERTED`) that arrives **immediately after**
   `CONNECTED` exactly like one arriving mid-session: run terminal handling, then close the stream.
3. Only auto-reconnect while the last-known status is `OPEN` or `COMPLETED`. After a terminal
   event, **do not** reconnect (existing guidance — reinforced).

---

## Terminal-Event-on-Reconnect Edge Cases

- **Event name = status string.** The terminal SSE event name is exactly `session.status()`
  (`"CANCELLED"` or `"CONVERTED"`), matching `DraftSseEmitterRegistry.TERMINAL_EVENTS` and the
  live-broadcast naming. No new event type is introduced.
- **No double-broadcast to live subscribers.** The terminal reconnect emitter is a standalone
  emitter that is **not** added to the registry, so replaying `CONNECTED` + terminal to the
  reconnecting client does not fan out to any other live subscriber and publishes no new
  `DraftStateChangedEvent`.
- **404 fast-fail preserved.** The existing state load runs first; a missing session returns 404
  before any emitter is created.
- **COMPLETED is NOT terminal.** A `COMPLETED` session keeps the stream open (it can still be
  `CONVERTED` via confirm or `CANCELLED`), so it follows the non-terminal path — unchanged.
- **Idempotent / repeatable.** Reconnecting to a terminal session any number of times yields the
  same deterministic `CONNECTED` → terminal → close sequence.
- **Send failure.** If sending `CONNECTED` or the terminal event throws `IOException`, complete
  the emitter with the error (consistent with the current `completeWithError` handling) — the
  client will observe stream end and must not reconnect for a terminal status.

---

## New / Modified DTOs

**None.** `DraftSessionDTO` (existing record) is reused unchanged as the payload for both the
`CONNECTED` event and the terminal event. No request bodies are added.

---

## Breaking Changes

- [x] **No breaking changes.**
  - Path, HTTP method, auth rule, and `text/event-stream` content type are unchanged.
  - `CANCELLED` / `CONVERTED` event names and `DraftSessionDTO` payload are unchanged.
  - The only observable difference: a client reconnecting to an *already-terminal* session now
    receives the terminal event immediately (then EOF) instead of a hanging stream. This is
    strictly additive and matches behavior clients already implement for mid-session terminals.

## Frontend Migration Notes

- No mandatory FE code change. Clients that already close on `CANCELLED` / `CONVERTED` (per the
  existing SSE guide) automatically benefit — they now close on reconnect instead of waiting for
  the 5-minute timeout.
- Recommended: update `docs/frontend/DRAFT_SESSION_SSE_GUIDE.md` "Reconnection Strategy" section
  to note that reconnecting to a terminal session yields `CONNECTED` → `<STATUS>` → close, and
  that auto-reconnect must be gated on `status ∈ {OPEN, COMPLETED}` (already the guidance).

---

## Summary for Next Agent

**Implement in `DraftSessionController.subscribeToEvents(Long id)` (design only — no code here):**

1. Load snapshot via `draftSessionService.getDraftSession(id)` first → preserves **404 fast-fail**.
2. Determine terminality from `snapshot.status()`: terminal ⇔ `"CANCELLED"` or `"CONVERTED"`
   (reuse the `DraftSseEmitterRegistry.TERMINAL_EVENTS` set; consider exposing it, or add a small
   helper on the registry, to avoid duplicating the constant).
3. **Non-terminal (`OPEN`, `COMPLETED`)** — behavior unchanged:
   - `emitter = sseEmitterRegistry.subscribe(id)` → send `CONNECTED` (snapshot) → return; stream stays open.
4. **Terminal (`CANCELLED`, `CONVERTED`)** — NEW path:
   - Create a **standalone** `SseEmitter` (do **not** call `subscribe(id)` / do not register in the broadcast registry).
   - Send `event: CONNECTED`, data = snapshot (authoritative baseline) — **first**.
   - Send `event: <status>` (`"CANCELLED"` or `"CONVERTED"`), data = same snapshot — **second**.
   - `emitter.complete()` — **third** (stream closes immediately).
   - On `IOException` while sending, `emitter.completeWithError(e)` (matches existing handling).
5. **Event ordering is strict:** `CONNECTED` always precedes the terminal event, which always
   precedes completion.
6. **No new endpoint, no new DTO, no schema change.** `GET /{id}` (rehydrate) + enhanced `/events`
   (reconnect) fully satisfy "resume".
7. **Edge cases to honor:** terminal event name == status string; standalone emitter → no
   double-broadcast and no new `DraftStateChangedEvent`; `COMPLETED` stays open (not terminal);
   heartbeats out of scope (no `@EnableScheduling`).

**Testing hints for later stages:** `@WebMvcTest(DraftSessionController.class)` asserting the SSE
event sequence per status; verify terminal reconnect does **not** increment
`sseEmitterRegistry.subscriberCount(id)`; verify 404 when session missing.

