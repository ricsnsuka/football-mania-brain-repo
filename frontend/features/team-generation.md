# Team Generation

The Team Generation feature allows privileged users to generate teams for a confirmed match plan. It supports two modes:

- **Auto Generate** — algorithmic generation (Balanced, Random, Snake Draft)
- **Captain Pick Draft** — an interactive turn-based draft where two captains pick players alternately in real time

Access is restricted to **MANAGER** roles for full management. A member with no roles can also reach this page, but only in read-only spectator mode and only when there is an active (`OPEN` or `COMPLETED`) draft pick session in progress. If no active session exists, Basic Users see a "no active sessions" message.

---

## Which plans are selectable

Both modes offer only plans where the API says **`generatable`** — confirmed, not yet expired, and
not already generated from.

**Filter on that flag; do not re-derive the rule.** It is sent precisely so the UI and the server
cannot drift apart.

This used to be "any `CONFIRMED` plan", so every plan ever confirmed stayed in the list forever and
it grew with each match played. Two changes fixed it:

- A plan that produces a match becomes **`GENERATED`**, a terminal status. That is what stops one
  plan being used twice.
- A plan whose kickoff has passed is **expired**, derived by the server from the clock rather than
  stored — there is no event to record, and a stored flag would need a job whose only output is
  "time passed".

Generated teams are named after each side's highest-rated player (`Team Ricardo`), except in
captain-pick drafts, which use the captains' names. `Team A` / `Team B` told you nothing and read
identically on every match ever played.

## Auto Generate Mode

### User Flow

1. **Select a match plan** — Choose a confirmed plan that has not expired or been used.
2. **See confirmed players** — Shows all players who confirmed attendance with skill ratings.
3. **Choose generation type** — `BALANCED`, `RANDOM`, or `SNAKE_DRAFT`.
4. **Generate preview** — API returns a `MatchPreviewDTO` showing proposed team compositions.
5. **Confirm or regenerate** — Review the preview, then confirm to create the match.

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/match-plans/{id}/generate?generationType=` | Preview team generation |
| `POST` | `/api/match-plans/{id}/generate/confirm?generationType=` | Create match from preview |

### Validation Rules

| Format | Required confirmed players |
|--------|---------------------------|
| `FIVE_A_SIDE` | 10 |
| `SEVEN_A_SIDE` | 14 |
| `ELEVEN_A_SIDE` | 22 |

---

## Captain Pick Draft Mode

A live, interactive draft where two captains take turns selecting players from the available pool until all positions are filled.

### User Flow

1. **Select a confirmed match plan** in the Captain Pick section.
2. Optionally **choose captains** (Team A and Team B). If omitted, the backend auto-selects the two highest-rated confirmed players.
3. **Start Draft** — creates a `DraftSession` via `POST /api/draft-sessions`.
4. The **CaptainPickBoard** renders a 3-column layout: Team A | Available Pool | Team B.
5. The active captain clicks a player in the pool to pick them (`POST /api/draft-sessions/{id}/pick`).
6. The board polls every 3 seconds — turn indicators update live.
7. When all players are picked (`status === 'COMPLETED'`), the **Create Match** button appears.
8. Confirm converts the draft to a real match (`POST /api/draft-sessions/{id}/confirm`).
9. At any time during OPEN or COMPLETED states, **Cancel Draft** is available.

### Draft Session Status Flow

```
OPEN → (all players picked) → COMPLETED → (confirm) → CONVERTED
                                         ↘ (cancel) → CANCELLED
OPEN → (cancel) → CANCELLED
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/draft-sessions` | Create a new draft session |
| `GET` | `/api/draft-sessions/{id}` | Fetch session state (polled every 3 s) |
| `POST` | `/api/draft-sessions/{id}/pick` | Submit a captain's pick |
| `POST` | `/api/draft-sessions/{id}/confirm` | Convert session to a match |
| `DELETE` | `/api/draft-sessions/{id}` | Cancel session |

### Draft Session DTO

```ts
interface DraftSessionDTO {
  id: number;
  matchPlanId: number;
  matchPlanTitle: string;
  status: 'OPEN' | 'COMPLETED' | 'CANCELLED' | 'CONVERTED';
  captainA: DraftPlayerDTO;    // always the first captain
  captainB: DraftPlayerDTO;
  currentTurn: 'A' | 'B' | null;
  teamA: DraftPlayerDTO[];
  teamB: DraftPlayerDTO[];
  remaining: DraftPlayerDTO[]; // players not yet picked
  totalPlayers: number;
  picksRemaining: number;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

---

## Draft Sessions Admin (GROUP_ADMIN only)

A separate admin-only page at `/draft-sessions` that lists **all** draft sessions regardless of status, with controls to cancel or permanently delete them.

### Route

`/draft-sessions` — guarded to `GROUP_ADMIN` only (redirects non-admins to `/dashboard`).

### User Flow

1. Admin navigates to `/draft-sessions`.
2. All sessions are listed in a table, newest-first.
3. Filter buttons (All | Open | Completed | Cancelled | Converted) narrow the list client-side.
4. For **OPEN** sessions — a **Cancel** button (soft-cancel via `DELETE /api/draft-sessions/{id}`) sets status to `CANCELLED`.
5. For all non-`CONVERTED` sessions — a **Purge** button (hard-delete via `DELETE /api/draft-sessions/{id}/purge`) permanently removes the DB row.
6. Both actions require an inline confirmation step before executing.
7. `CONVERTED` sessions (linked to a real match) cannot be purged — the purge button is hidden and the backend enforces this with a `409 Conflict`.

### API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/draft-sessions/summary` | `GROUP_ADMIN` | Lightweight list of all sessions |
| `DELETE` | `/api/draft-sessions/{id}/purge` | `GROUP_ADMIN` | Hard-delete a session |
| `DELETE` | `/api/draft-sessions/{id}` | `GROUP_ADMIN` / `MANAGER` | Soft-cancel (already existed) |

### DraftSessionSummaryDTO

```ts
interface DraftSessionSummaryDTO {
  id: number;
  matchPlanId: number;
  matchPlanTitle: string;
  status: 'OPEN' | 'COMPLETED' | 'CANCELLED' | 'CONVERTED';
  captainAName: string;
  captainBName: string;
  currentTurn: 'A' | 'B' | null;   // null unless OPEN
  totalPlayers: number;
  picksRemaining: number;
  createdBy: string | null;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

## Files Involved

| File | Role |
|------|------|
| `src/app/(app)/team-generation/page.tsx` | Next.js route — renders `TeamGenerationPage` |
| `src/app/(app)/draft-sessions/page.tsx` | Next.js route — admin-only draft sessions admin page |
| `src/features/teamGeneration/TeamGenerationPage.tsx` | Main page — auto generate + captain pick sections |
| `src/features/teamGeneration/CaptainPickSection.tsx` | Setup form for starting a draft session |
| `src/features/teamGeneration/CaptainPickBoard.tsx` | Live interactive draft board |
| `src/features/teamGeneration/DraftSessionsAdminPage.tsx` | Admin table with cancel/purge actions |
| `src/services/draftService.ts` | API calls for draft-sessions endpoints |
| `src/hooks/draft/useDraft.ts` | TanStack Query hooks for draft state + mutations |
| `src/types/draft.ts` | TypeScript types: `DraftSessionDTO`, `DraftSessionSummaryDTO`, etc. |
| `src/tests/draft/useDraft.test.ts` | Hook unit tests |
| `src/tests/draft/CaptainPickBoard.test.tsx` | Board component tests |
| `src/tests/draft/CaptainPickSection.test.tsx` | Setup section component tests |
| `src/tests/draft/DraftSessionsAdminPage.test.tsx` | Admin page component tests |

---

## i18n Keys

All user-facing strings live under the `teamGeneration` and `captainPick` namespaces in `src/locales/*/common.json`.

Key groups:
- `teamGeneration.*` — page title, setup form, player count, preview, confirm
- `captainPick.setup.*` — setup form labels and actions
- `captainPick.board.*` — draft board labels, turn indicators, actions
- `captainPick.status.*` — status badge text
- `captainPick.pick|create|confirm|cancel` — toast notification messages
