# Draft Session — Real-Time SSE Integration Guide

**Feature:** Captain Pick Draft Session — Live Event Stream  
**Endpoint:** `GET /api/draft-sessions/{id}/events`  
**Protocol:** Server-Sent Events (SSE) over HTTP/1.1 or HTTP/2  
**Added in:** v1.0.0 (2026-05-27)

---

## Overview

When a draft session is in progress, multiple users (both captains and observers) need to see picks happening in real time. Rather than polling `GET /api/draft-sessions/{id}` repeatedly, the frontend subscribes once to the SSE stream and receives push updates automatically.

```
┌─────────────────────────────────────────────────────────┐
│  FE subscribes to GET /api/draft-sessions/{id}/events   │◄── JWT in header
│                                                          │
│  Server sends:                                           │
│    → CONNECTED  (current state, immediately on connect)  │
│    → PICK        (after every pick, while OPEN)          │
│    → COMPLETED   (when all picks are done)               │
│    → CANCELLED   (session cancelled by admin)            │
│    → CONVERTED   (match confirmed, session done)         │
│                                                          │
│  FE submits picks via  POST /api/draft-sessions/{id}/pick│
│  (SSE is read-only — it receives, never sends)           │
└─────────────────────────────────────────────────────────┘
```

---

## SSE Event Reference

Every event arrives as a `DraftSessionDTO` JSON object:

```
event: <EVENT_TYPE>
data: { ...DraftSessionDTO... }
```

### Event Types

| Event | Trigger | Status in payload | Action |
|-------|---------|-------------------|--------|
| `CONNECTED` | Immediately on subscribe | Current status | Render initial state |
| `PICK` | After each pick (while remaining > 0) | `OPEN` | Update teams + remaining pool |
| `COMPLETED` | When last player is picked | `COMPLETED` | Show "Confirm draft" UI |
| `CANCELLED` | Admin cancels | `CANCELLED` | Show error / redirect |
| `CONVERTED` | Admin confirms → match created | `CONVERTED` | Navigate to match list |

> **`OPENED` exists on the backend but never reaches an SSE client.** It is published when a
> session is created, purely so the push notifier can tell the first captain it is their turn
> (every later turn arrives on `PICK`). Nobody can be subscribed to a session that did not exist
> a moment earlier, so the registry drops it. Do **not** add it to `DraftEventType` — it is not
> part of the SSE contract, and a handler for it would be dead code.

### `CANCELLED` and `CONVERTED` are terminal events
The server automatically closes the SSE stream after sending either of these. The client should close the `EventSource` on receipt.

---

## DraftSessionDTO Payload Shape

```json
{
  "id": 1,
  "matchPlanId": 5,
  "matchPlanTitle": "Friday Night Match",
  "status": "OPEN",
  "captainA": { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 },
  "captainB": { "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 },
  "currentTurn": "A",
  "teamA": [
    { "playerId": 42, "playerName": "João Silva", "skillRating": 9.2 }
  ],
  "teamB": [
    { "playerId": 17, "playerName": "Miguel Santos", "skillRating": 8.8 }
  ],
  "remaining": [
    { "playerId": 7, "playerName": "Carlos Matos", "skillRating": 8.5 }
  ],
  "totalPlayers": 10,
  "picksRemaining": 8,
  "expiresAt": null,
  "createdAt": "2026-05-27T10:00:00Z",
  "updatedAt": "2026-05-27T10:05:00Z"
}
```

| Field | Notes |
|-------|-------|
| `currentTurn` | `"A"` or `"B"` while `OPEN`; `null` in other statuses |
| `remaining` | Available pool ordered by `skillRating DESC` |
| `picksRemaining` | `remaining.length` — use this for progress bar |
| `expiresAt` | Optional session expiry; `null` if no time limit set |

---

## Authentication

SSE requires a valid JWT, exactly like every other API endpoint. The browser's native `EventSource` API **does not support custom headers**, so you must use the `Fetch API` (or a library like `@microsoft/fetch-event-source`) to attach the `Authorization: Bearer <token>` header.

> ⚠️ **Do NOT use `new EventSource(url)` directly** — it cannot send the JWT.

---

## Implementation Examples

### Vanilla JavaScript (Fetch-based SSE)

```javascript
/**
 * Subscribes to a draft session's real-time event stream.
 *
 * @param {number}   sessionId  - The draft session ID
 * @param {string}   jwtToken   - Bearer token from login
 * @param {Function} onEvent    - Called with (eventType, DraftSessionDTO) on each event
 * @returns {AbortController}   - Call .abort() to close the connection
 */
async function subscribeToDraftSession(sessionId, jwtToken, onEvent) {
  const controller = new AbortController();

  const response = await fetch(
    `/api/draft-sessions/${sessionId}/events`,
    {
      headers: {
        Authorization: `Bearer ${jwtToken}`,
        Accept: 'text/event-stream',
      },
      signal: controller.signal,
    }
  );

  if (!response.ok) {
    throw new Error(`SSE connection failed: ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  (async () => {
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop(); // Keep incomplete line

        let eventType = null;
        let dataLine = null;

        for (const line of lines) {
          if (line.startsWith('event:')) {
            eventType = line.slice(6).trim();
          } else if (line.startsWith('data:')) {
            dataLine = line.slice(5).trim();
          } else if (line === '' && eventType && dataLine) {
            const payload = JSON.parse(dataLine);
            onEvent(eventType, payload);

            // Close connection on terminal events
            if (eventType === 'CANCELLED' || eventType === 'CONVERTED') {
              controller.abort();
              return;
            }

            eventType = null;
            dataLine = null;
          }
        }
      }
    } catch (err) {
      if (err.name !== 'AbortError') {
        console.error('SSE stream error:', err);
      }
    }
  })();

  return controller;
}

// ── Usage ────────────────────────────────────────────────────────────────────

const abort = await subscribeToDraftSession(1, token, (type, session) => {
  switch (type) {
    case 'CONNECTED':
      console.log('Connected — current state:', session);
      renderDraftBoard(session);
      break;
    case 'PICK':
      console.log(`Pick made — it is now Team ${session.currentTurn}'s turn`);
      renderDraftBoard(session);
      break;
    case 'COMPLETED':
      console.log('All picks done — waiting for admin to confirm');
      showConfirmBanner();
      break;
    case 'CANCELLED':
      console.warn('Draft session was cancelled');
      showError('This draft session was cancelled.');
      break;
    case 'CONVERTED':
      console.log('Draft confirmed — match created!');
      navigateToMatchList();
      break;
  }
});

// To disconnect manually:
abort.abort();
```

---

### React Hook (TypeScript)

```typescript
// types.ts
interface DraftPlayerDTO {
  playerId: number;
  playerName: string;
  skillRating: number;
}

interface DraftSessionDTO {
  id: number;
  matchPlanId: number;
  matchPlanTitle: string;
  status: 'OPEN' | 'COMPLETED' | 'CANCELLED' | 'CONVERTED';
  captainA: DraftPlayerDTO;
  captainB: DraftPlayerDTO;
  currentTurn: 'A' | 'B' | null;
  teamA: DraftPlayerDTO[];
  teamB: DraftPlayerDTO[];
  remaining: DraftPlayerDTO[];
  totalPlayers: number;
  picksRemaining: number;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}

// Deliberately excludes the backend's 'OPENED' — see the note under the event table above.
type DraftEventType = 'CONNECTED' | 'PICK' | 'COMPLETED' | 'CANCELLED' | 'CONVERTED';
```

```typescript
// useDraftSession.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseDraftSessionOptions {
  sessionId: number;
  token: string;
  onTerminal?: (type: 'CANCELLED' | 'CONVERTED', session: DraftSessionDTO) => void;
}

interface UseDraftSessionResult {
  session: DraftSessionDTO | null;
  connected: boolean;
  error: string | null;
  reconnect: () => void;
}

export function useDraftSession({
  sessionId,
  token,
  onTerminal,
}: UseDraftSessionOptions): UseDraftSessionResult {
  const [session, setSession] = useState<DraftSessionDTO | null>(null);
  const [connected, setConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const abortRef = useRef<AbortController | null>(null);

  const connect = useCallback(() => {
    // Cancel any existing connection
    abortRef.current?.abort();
    const controller = new AbortController();
    abortRef.current = controller;

    setError(null);

    (async () => {
      try {
        const response = await fetch(
          `/api/draft-sessions/${sessionId}/events`,
          {
            headers: {
              Authorization: `Bearer ${token}`,
              Accept: 'text/event-stream',
            },
            signal: controller.signal,
          }
        );

        if (!response.ok) {
          setError(`Connection failed: ${response.status}`);
          return;
        }

        setConnected(true);
        const reader = response.body!.getReader();
        const decoder = new TextDecoder();
        let buffer = '';

        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          buffer += decoder.decode(value, { stream: true });
          const lines = buffer.split('\n');
          buffer = lines.pop() ?? '';

          let eventType: DraftEventType | null = null;
          let dataLine: string | null = null;

          for (const line of lines) {
            if (line.startsWith('event:')) {
              eventType = line.slice(6).trim() as DraftEventType;
            } else if (line.startsWith('data:')) {
              dataLine = line.slice(5).trim();
            } else if (line === '' && eventType && dataLine) {
              const payload: DraftSessionDTO = JSON.parse(dataLine);
              setSession(payload);

              if (eventType === 'CANCELLED' || eventType === 'CONVERTED') {
                onTerminal?.(eventType, payload);
                controller.abort();
                setConnected(false);
                return;
              }

              eventType = null;
              dataLine = null;
            }
          }
        }
      } catch (err: unknown) {
        if ((err as Error).name !== 'AbortError') {
          console.error('[useDraftSession] Stream error:', err);
          setError('Stream disconnected. Reconnecting...');
          setConnected(false);
          // Auto-reconnect after 3 seconds (mirrors server retry hint)
          setTimeout(() => connect(), 3_000);
        }
      }
    })();
  }, [sessionId, token, onTerminal]);

  useEffect(() => {
    connect();
    return () => {
      abortRef.current?.abort();
    };
  }, [connect]);

  return { session, connected, error, reconnect: connect };
}
```

```tsx
// DraftBoard.tsx
import { useDraftSession } from './useDraftSession';
import { useNavigate } from 'react-router-dom';

export function DraftBoard({ sessionId }: { sessionId: number }) {
  const token = useAuthToken(); // Your auth hook
  const navigate = useNavigate();

  const { session, connected, error } = useDraftSession({
    sessionId,
    token,
    onTerminal: (type) => {
      if (type === 'CONVERTED') navigate('/matches');
      if (type === 'CANCELLED') navigate('/draft-sessions');
    },
  });

  if (!session) return <p>Connecting to draft...</p>;
  if (error) return <p className="error">{error}</p>;

  return (
    <div className="draft-board">
      <header>
        <span className={`status ${session.status.toLowerCase()}`}>
          {session.status}
        </span>
        <span>{connected ? '🟢 Live' : '🔴 Reconnecting...'}</span>
      </header>

      <progress value={session.totalPlayers - session.picksRemaining} max={session.totalPlayers} />
      <p>{session.picksRemaining} picks remaining</p>

      {session.status === 'OPEN' && (
        <p className="turn-indicator">
          ⏳ Team {session.currentTurn}'s turn to pick
        </p>
      )}

      <div className="teams">
        <TeamColumn
          label={`Team A — ${session.captainA.playerName}`}
          players={session.teamA}
          highlight={session.currentTurn === 'A'}
        />
        <TeamColumn
          label={`Team B — ${session.captainB.playerName}`}
          players={session.teamB}
          highlight={session.currentTurn === 'B'}
        />
      </div>

      <PlayerPool
        players={session.remaining}
        currentTurn={session.currentTurn}
        sessionId={sessionId}
        token={token}
      />
    </div>
  );
}
```

---

### Submitting a Pick

Picks are submitted via REST, not through the SSE stream:

```typescript
async function submitPick(sessionId: number, playerId: number, token: string): Promise<void> {
  const response = await fetch(`/api/draft-sessions/${sessionId}/pick`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ playerId }),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(err.message ?? 'Pick failed');
  }
  // No need to update local state — the SSE PICK event will arrive shortly and update everyone
}
```

> ℹ️ After submitting a pick, **do not update local state manually** — the SSE `PICK` event will arrive within milliseconds and update every connected client simultaneously (including the one who just picked). This prevents race conditions.

---

## Connection Flow

```
Client                                        Server
  │                                              │
  │  GET /api/draft-sessions/{id}/events         │
  │  Authorization: Bearer <token>               │
  │  Accept: text/event-stream          ────────►│
  │                                              │
  │◄──── HTTP 200 Content-Type: text/event-stream│
  │                                              │
  │  event: CONNECTED                            │
  │  data: {current DraftSessionDTO}    ◄────────│
  │                                              │
  │  (Session is OPEN — captains take turns)     │
  │                                              │
  │  POST /api/draft-sessions/{id}/pick ─────────│  (Captain A picks)
  │  { "playerId": 7 }                           │
  │  ◄─────────── 200 DraftSessionDTO ───────────│
  │                                              │
  │  event: PICK                                 │
  │  data: {updated DraftSessionDTO}   ◄─────────│  (all watchers receive this)
  │                                              │
  │  (last pick happens...)                      │
  │                                              │
  │  event: COMPLETED                            │
  │  data: {DraftSessionDTO status=COMPLETED}  ◄─│
  │                                              │
  │  (Admin calls POST /{id}/confirm)            │
  │                                              │
  │  event: CONVERTED                            │
  │  data: {DraftSessionDTO status=CONVERTED}  ◄─│
  │                                              │
  │  [Server closes SSE stream]                  │
  │  [Client closes connection]                  │
  │                                              │
```

---

## Reconnection Strategy

The server sets a `retry: 3000` hint on every event (3 second reconnect delay). The browser's native `EventSource` handles this automatically. When using the Fetch API approach, reconnect manually on stream end:

```typescript
// Inside the stream reading loop — on stream close (not abort):
if (done) {
  // Stream ended normally (e.g., server timeout after 5 min)
  // Reconnect ONLY if the last-known status is non-terminal.
  // Terminal sessions (CANCELLED/CONVERTED) close themselves — do NOT reconnect.
  if (session?.status === 'OPEN' || session?.status === 'COMPLETED') {
    setTimeout(() => connect(), 3_000);
  }
  break;
}
```

> ✅ **Gate auto-reconnect on `status ∈ {OPEN, COMPLETED}`.** After a terminal event (`CANCELLED` / `CONVERTED`), the server closes the stream and you must **not** reconnect. Rely on the server's terminal event — not a status check — to know when to close for the current connection.

### Terminal-on-Reconnect (no more hanging streams)

Every (re)connect to `/events` first sends an authoritative `CONNECTED` snapshot. What happens next depends on the session status **at the moment you connect**:

| Status at (re)connect | Event sequence | Stream after |
|-----------------------|----------------|--------------|
| `OPEN` | `CONNECTED` → *(live `PICK` / `COMPLETED` / `CANCELLED` / `CONVERTED`)* | stays open |
| `COMPLETED` | `CONNECTED` → *(later `CANCELLED` / `CONVERTED`)* | stays open |
| `CANCELLED` | `CONNECTED` → `CANCELLED` → **close** | closes immediately |
| `CONVERTED` | `CONNECTED` → `CONVERTED` → **close** | closes immediately |

> 🆕 **Behavioral fix (2026-07-02):** Previously, reconnecting to an *already-terminal* session (one that was `CANCELLED` / `CONVERTED` while you were disconnected) left the stream hanging with no close signal until the 5-minute timeout. Now the server sends `CONNECTED` (authoritative terminal snapshot) → the matching terminal event (`CANCELLED` or `CONVERTED`) → and **closes the stream immediately**. No client change is required — the existing terminal handlers (which already close the `EventSource` on `CANCELLED`/`CONVERTED`) handle this automatically, whether the terminal event arrives mid-session or right after `CONNECTED` on reconnect.

### Resuming a dropped session

Because all draft session state is persisted server-side, a dropped SSE stream (5-minute timeout, tab reload, network blip, laptop sleep, etc.) can be **resumed** simply by reconnecting to the same `/events` endpoint — there is **no dedicated resume endpoint**. The `CONNECTED` event is itself the resume/rehydrate primitive.

```
Client                                                    Server
  │  (stream dropped: timeout / network / reload)            │
  │                                                          │
  │  GET /api/draft-sessions/{id}/events   (reconnect) ─────►│
  │  Authorization: Bearer <token>                           │
  │                                                          │
  │◄──── 200 text/event-stream                               │
  │  event: CONNECTED                                        │
  │  data: <DraftSessionDTO> (authoritative state) ◄─────────│
  │                                                          │
  ├─ status OPEN | COMPLETED:                                │
  │     render from CONNECTED snapshot; keep stream open;    │
  │     resume picking via POST /api/draft-sessions/{id}/pick│
  │                                                          │
  ├─ status CANCELLED | CONVERTED:                           │
  │     event: <STATUS>  data: <DraftSessionDTO>    ◄─────────│
  │     [server completes stream] → close EventSource        │
  │     (CANCELLED → notice/redirect, CONVERTED → matches)   │
```

**Client rules for resume:**
1. On reconnect, always wait for `CONNECTED` and treat it as the authoritative baseline. A separate `GET /api/draft-sessions/{id}` call is **not** required — though that endpoint remains available for a manual full-state rehydrate at any time.
2. Handle a terminal event (`CANCELLED` / `CONVERTED`) that arrives immediately after `CONNECTED` exactly like one arriving mid-session: run your terminal handling, then close the stream.
3. Only auto-reconnect while the last-known status is `OPEN` or `COMPLETED`. After a terminal event, **do not** reconnect.

### SSE Timeout

The server closes the SSE connection after **5 minutes** of inactivity. This is a standard HTTP keepalive limit. The client should reconnect:
- Automatically after receiving the stream-end signal
- Only if the session is still `OPEN` or `COMPLETED` (`CANCELLED` / `CONVERTED` are terminal — the server closes those streams itself and sends the terminal event first, so reconnecting to a terminal session no longer hangs until the timeout)

---

## Error Handling

| HTTP Status | Cause | Resolution |
|-------------|-------|------------|
| `401` | Missing or expired JWT | Re-authenticate, then reconnect |
| `403` | Not authenticated | Log in first |
| `404` | Session ID not found | Redirect to session list |
| `200 → stream closes` | Server timeout (5 min) | Reconnect if session still active |
| `200 → CANCELLED event` | Admin cancelled | Show message, redirect |
| `200 → CONVERTED event` | Match created | Navigate to match list |

```typescript
// Handle HTTP-level errors before reading the stream
const response = await fetch(url, { headers, signal });

switch (response.status) {
  case 401:
    await refreshToken();
    connect(); // retry
    return;
  case 403:
    navigate('/login');
    return;
  case 404:
    setError('Draft session not found.');
    return;
}
```

---

## CORS Considerations

The API already has CORS configured. For SSE specifically:

- `Authorization` header — included in CORS allowed headers ✅
- `Content-Type: text/event-stream` — supported by the server ✅
- **Preflight:** SSE `GET` requests send a `preflight OPTIONS` request. The server handles `OPTIONS /**` with `permitAll()` ✅

No additional CORS configuration is needed on the frontend.

---

## Important Notes

### The CONNECTED event is your authoritative baseline

On subscribe, the server immediately sends a `CONNECTED` event with the full current state. Always wait for this event before rendering the draft board — do not call `GET /api/draft-sessions/{id}` separately on subscribe.

### Why not WebSocket?

Picks are submitted via REST (`POST /api/draft-sessions/{id}/pick`). The only missing piece was server-to-client push — not bidirectional communication. SSE handles this with less setup, no extra dependencies, and natural HTTP semantics (auth headers, standard error codes, load balancer compatibility).

### Virtual Thread scalability

The server uses Java 21 virtual threads. Each SSE connection holds a virtual thread (not an OS thread), so hundreds of simultaneous observers have negligible performance impact.

### Multi-instance deployments

In single-instance deployments (current Heroku setup), SSE works out of the box. If you horizontally scale to multiple JVM instances, enable **sticky sessions** on your load balancer so each client always reaches the same instance. Alternatively, a broker-backed SSE solution (e.g., Redis Pub/Sub + reactive streaming) would be needed for stateless multi-instance deployments.

---

## Quick Testing with curl

```bash
# Subscribe to events (replace TOKEN and SESSION_ID)
curl -N \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/event-stream" \
  https://your-api/api/draft-sessions/1/events
```

Expected output:
```
event: CONNECTED
data: {"id":1,"status":"OPEN","currentTurn":"A",...}

event: PICK
data: {"id":1,"status":"OPEN","currentTurn":"B",...}

event: COMPLETED
data: {"id":1,"status":"COMPLETED","currentTurn":null,...}
```

> Browser DevTools → Network tab → Filter by `EventStream` type also shows all SSE events in real time.

---

## Related Documentation

- [Draft Session Feature Guide](./DRAFT_SESSION_FEATURE.md)
- [API Reference — Draft Sessions](../api/API_REFERENCE.md#draft-sessions)
- [Frontend Endpoint Changes](./FRONTEND_ENDPOINT_CHANGES.md)

