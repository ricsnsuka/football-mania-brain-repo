# Players Feature

Client-side paginated table of all registered players with filtering and role-gated creation.

> **This is the roster view, not the league table.** It answers "who is in the group, who is
> active, what is their skill rating" — an administrative question. Competitive standing lives
> on the [Rankings](rankings.md) page, which computes real W-D-L from completed matches rather
> than ordering by skill rating. The two used to overlap: this page carried a `#` column with
> gold/silver/bronze medals derived from skill-rating order, which read as a ranking while
> measuring something else entirely. That column is gone.

> **Pagination is client-side, not server-side.** `usePlayers()` fetches the full player
> dataset once (`{ size: 10000 }`) under one canonical query key (`['players', 'all']`);
> `PlayersPage` sorts, filters, and slices that array in memory (`useMemo`-keyed on
> `[data, sortBy, sortDir]` and `[sortedPlayers, page, pageSize]` so it doesn't re-run on
> every render). This is a deliberate decision, not a bug: the backend's `GET /api/players`
> only documents `page`, `size`, and `active` — no `sort` param — so server-side sort/paging
> isn't available to the frontend today. See the query-layer pattern in
> [Architecture Overview](../architecture/overview.md) for why player data is fetched this
> way (one canonical query, consumed via `select` by every screen that needs players).

## Page defaults

| Setting | Value |
|---------|-------|
| Default page size | **10** per page |
| Available page sizes | 5, 10, 20 |
| Default filter | All (active + inactive) |
| Default sort | **Name, ascending** |

The default sort is alphabetical because this is a lookup view: you arrive knowing which player
you want. Sorting by skill rating instead would re-assert the ranking reading that the `#`
column was removed to end.

## Filters

| Filter | Applied as |
|--------|-----------|
| All | no filter |
| Active | `player.isActive === true` |
| Inactive | `player.isActive === false` |

Filtering happens client-side over the same fetched dataset via `usePlayers({ select })` —
there is no `active` query param sent to the backend for this page.

Changing the filter resets to page 0.  
Changing the page size resets to page 0.

## Page size selector

The shared `Pagination` component (`src/components/ui/Pagination.tsx` — see
[Shared UI Primitives](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/shared-components.md)) renders a page-size `<select>`
letting users switch between 5, 10, and 20 rows per page. It is visible whenever the page
contains at least one player (`totalElements > 0`).

## Table columns

| # | Column | Data source | Notes |
|---|--------|-------------|-------|
| 1 | Name | `player.name` | Sortable, `localeCompare` so accented names order correctly |
| 2 | Status | `player.isActive` | Active / Inactive chip |
| 3 | Skill Rating | `player.skillRating` | Dynamic rating (not base); sortable |
| 4 | Played | `player.totalMatchesPlayed` | `—` when absent; sortable |
| 5 | Goals | `player.totalGoals` | `—` when absent; sortable |
| 6 | Assists | `player.totalAssists` | `—` when absent; sortable |
| 7 | Streak | `player.currentStreak` | |

Sortable columns expose `aria-sort`. Name is the only non-numeric sort, so it branches to
`localeCompare` while every other key subtracts; absent numeric values sort as `-1`.

> **Note:** `totalMatchesPlayed`, `totalGoals`, and `totalAssists` are optional fields in
> `PlayerDTO`. They render `—` until the backend exposes aggregate stats on the player
> endpoint.

## Player detail modal

| Field | Data source |
|-------|-------------|
| Name | `player.name` (heading) |
| Status | `player.isActive` chip |
| Skill Rating | `player.skillRating` |
| Current Streak | `player.currentStreak` |
| Best Streak | `player.longestStreak` |
| Played Matches | `player.totalMatchesPlayed` (`—` if absent) |
| Goals | `player.totalGoals` (`—` if absent) |
| Assists | `player.totalAssists` (`—` if absent) |

The stat chips render in a `grid-cols-2 sm:grid-cols-3` grid (6 chips, 2 rows on desktop).

### Position and keeper chips

Beside the status and guest chips in the header, a neutral chip per entry in
`player.preferredPositions`, and — **only when it is not `NEVER`** — one amber chip for
`player.goalkeeperWillingness`.

A "never" chip on most of the roster would be noise. The question the chip answers is *who can I
put in goal*, and a row of negatives makes that harder to read, not easier. The willing chip is
solid and the "if needed" one is outlined, because that difference is the entire reason the field
is three values and not a boolean.

Positions get neutral slate rather than a colour: they sit next to the status chip (green/grey) and
the guest chip (violet), and a third colour family in the same row stops any of them meaning
anything.

## Editing positions and keeper willingness

`PositionFields` renders both controls and is shared by the create and edit modals — one component,
because the rule below is one thing to get wrong and two copies would be two places to forget it.

**Positions are checkboxes.** A player may prefer several, and a single-choice control would make
them pick a favourite and quietly lose the fact that they will also play at the back.

**Picking goalkeeper locks the willingness control to "happy to" and disables it.** The server
overwrites the field in that case whatever the form sends, so an enabled control would silently
discard the choice — a control that ignores you is worse than one that shows you it is not yours
to set. The hint text changes to say why. The value is raised in the same handler that ticks the
box, so the form never submits a stale `NEVER` from under a control that has just gone grey.

Unticking goalkeeper re-enables the control but does **not** lower the value, mirroring the server:
there is no record of what it was before, and `HAPPY_TO` is not wrong about somebody who was happy
in goal last week.

**The edit modal sends both fields on every save, even untouched.** `[]` is how the server is told
to clear the set and an omitted field means "leave alone", so sending only what changed would make
removing your last position impossible.

Both are editable on the self path as well as the manager one — the same component, the same
endpoint choice `EditPlayerModal` already makes for name and phone.

## Role gates

| Action | Roles | Enforced by |
|--------|-------|-------------|
| Reach the route | Any authenticated user | `AuthGuard` in `src/app/(app)/players/page.tsx` |
| See the navbar entry | `MANAGER` | `NAV_LINKS` in `Navbar.tsx` |
| Create a player | `MANAGER` | `canCreate` in `PlayersPage.tsx` |

**Discoverability and access are gated separately, and only the first excludes anyone.** The
navbar entry is hidden from `a member with no roles` because for a player this page is a strictly worse
[Rankings](rankings.md) page — same names, less information, no competitive standing — so
offering both invites the wrong one to be clicked. The route behind it is guarded for
authentication only: a bookmark, a deep link, or a nav entry restored later all keep working,
and nothing is shown here that a `a member with no roles` cannot already reach elsewhere.

That distinction is deliberate, so don't "fix" it by adding a role check to the route. If this
page ever grows something a `a member with no roles` genuinely must not see, the gate belongs on the data,
not on the link — hiding a nav entry is a usability decision and has never been a security
boundary.

## File map

| Layer | File |
|-------|------|
| Page component | `src/features/players/PlayersPage.tsx` |
| Player detail modal | `src/features/players/PlayerModal.tsx` |
| Create modal | `src/features/players/CreatePlayerModal.tsx` |
| Edit modal (self + manager) | `src/features/players/EditPlayerModal.tsx` |
| Position / keeper fields | `src/features/players/PositionFields.tsx` |
| Data hook | `src/hooks/player/usePlayers.ts` |
| Service | `src/services/playerService.ts` |
| Types | `src/types/player.ts` |

## i18n keys (players namespace)

All strings live under the `players` key in each `locales/<lang>/common.json`.

**Column keys** (`players.columns.*`): `name`, `status`, `skillRating`, `playedMatches`, `goals`, `assists`, `streak`

**Position and keeper keys**: `players.positions.label`, `players.positions.hint`,
`players.positions.values.*` (four), `players.keeper.label`, `players.keeper.hint`,
`players.keeper.lockedHint`, `players.keeper.values.*` (three), `players.keeper.chip.*` (three —
`NEVER` is present and empty, since that chip is never rendered).

The `t()` fallbacks for both enums are **spelled-out English**, held in `PLAYER_POSITION_LABELS` and
`GOALKEEPER_WILLINGNESS_LABELS` in `src/types/player.ts`, not the enum value. A runtime-built key
falling back to itself is exactly how `FEE_CHARGED` and `MVP_VOTE_OPEN` reached production as screen
text. Writing the first test for `PositionFields` caught it immediately — it asserted `Goalkeeper`
and got `GOALKEEPER` — which is the argument for the fallbacks existing at all.

There is deliberately no `players.columns.rank`. The identically named `rankings.columns.rank`
does exist and is in use — they are different keys in different namespaces, so a search hit on
one is not evidence the other went missing by mistake.

**Modal keys** (`players.modal.*`): `close`, `skillRating`, `currentStreak`, `longestStreak`, `playedMatches`, `goals`, `assists`

**Pagination**: the players page uses the shared `Pagination` component and its top-level
`pagination.*` keys (`position`, `totalItems`, `pageSize`, `previous`, `next`) — see
[Shared UI Primitives](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/shared-components.md) — not a players-namespaced
`players.pagination.*` set.

**Empty/error states**: there are no players-namespaced keys for these. Empty, error and
loading all go through the shared `DataStateMessage` / `TableSkeleton` components and their
`states.*` / `common.loading` keys — see
[Shared UI Primitives](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/shared-components.md).
