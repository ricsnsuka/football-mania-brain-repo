# Matches Feature

Server-side paginated card list of all matches with filtering and role-gated creation.

## Page defaults

| Setting | Value |
|---------|-------|
| Default page size | **3** per page |
| Available page sizes | 3, 5, 10 |
| Default filter | All (upcoming + completed) |

## Filters

| Filter | API param |
|--------|-----------|
| All | `completed` omitted |
| Upcoming | `completed=false` |
| Completed | `completed=true` |

Changing the filter resets to page 0.  
Changing the page size resets to page 0.

## Page size selector

A `<select>` rendered inside the pagination bar (`matches-page-size-select`) lets users switch between 3, 5, and 10 matches per page. It is visible whenever the page contains at least one match (`totalElements > 0`).

## Role gates

| Action | Roles |
|--------|-------|
| View matches | All |
| Create match | `MANAGER` |
| Bulk recalculate ratings | `ADMIN` |
| Single recalculate ratings | `ADMIN` |

## Match creation

**One FAB, one modal.** There used to be two stacked buttons — `+` and `⚙` — opening near-identical
forms. The only real difference was that one enforced the match type's exact player count while the
other accepted any two equal teams. That is a validation nuance, not a second feature, and it left
the page offering two buttons nobody could tell apart.

What survives is the flexible one: both teams need at least one player and the same number, with no
match-type count enforced. `generationType` is omitted from the payload and the backend sets
`MANUAL`.

> The strict endpoint still exists at `POST /api/matches` and is unchanged. Nothing in the UI calls
> it — the surviving modal posts to `/api/matches/manual`.

## Bulk rating recalculation (admin only)

The `RecalculateMatchesPanel` is rendered inside `MatchesPage` for `ADMIN` only. It allows re-running the rating engine on completed matches.

### Scope modes

| Mode | Payload |
|------|---------|
| All completed | `{}` |
| By match IDs | `{ matchIds: number[] }` |
| By season | `{ seasonId: number }` |

Constraints: max 500 match IDs per request. IDs are parsed from a comma/space-separated text input.

### Results display

After running, a summary row (total / succeeded / failed) and a per-match results table are shown inline. Each row has a status chip (`SUCCESS`, `FAILED`, `SKIPPED`).

## Single match rating recalculation (admin only)

In the `MatchModal` scoresheet view, an admin-only zone (`match-modal-admin-zone`) is rendered below the stats table with a **Recalculate Ratings** button (`match-modal-recalc-btn`). On success, a toast notifies how many player ratings were updated.

## File map

| Layer | File |
|-------|------|
| Page component | `src/features/matches/MatchesPage.tsx` |
| Match card | `src/features/matches/MatchCard.tsx` |
| Match detail modal | `src/features/matches/matchModal/MatchModal.tsx` |
| Modal parts | `matchModal/TeamScoresheet.tsx`, `TeamsRosterView.tsx`, `RecordMatchForm.tsx`, `MotmVotePanel.tsx` |
| Modal helpers | `src/features/matches/matchModal/utils.ts` (`getMode`, `countdownTo`) |
| Create modal | `src/features/matches/CreateMatchModal.tsx` |
| Bulk recalculate panel | `src/features/matches/RecalculateMatchesPanel.tsx` |
| Data hook | `src/hooks/match/useMatches.ts` |
| Service | `src/services/matchService.ts` |
| Types | `src/types/match.ts` |

## i18n keys (matches namespace)

All strings live under the `matches` key in each `locales/<lang>/common.json`.

Notable keys:

- `matches.pagination.prev` / `matches.pagination.next` — prev/next button labels
- `matches.pagination.of` — "of N pages"
- `matches.pagination.perPage` — "per page" label in the size selector

## Match modal

The `MatchModal` has three display modes depending on match state:

| Mode | Condition | Content |
|------|-----------|---------|
| `upcoming` | Not due yet | Kickoff countdown in the header; squads side by side with ratings |
| `record` | Past, not completed | Admin/master: live stat-entry form. Basic user: awaiting-result message |
| `scoresheet` | Completed | Scoreboard, then **tabs**: Scoresheet / Squads / MOTM |

### Post-match content is tabbed

Everything used to render in one column — scoreboard, both scoresheets, the MOTM panel and the
admin tools. On a phone that is a long scroll to reach something you already knew you wanted. Three
tabs put each answer one tap away.

The selected tab is stored **with the match id it belongs to**, so opening a different match falls
back to the scoresheet without an effect resetting it. Doing that in an effect trips
`react-hooks/set-state-in-effect` and renders one frame on the stale tab first — open a match on
MOTM, open another, and the second match's MOTM panel flashes before the reset lands.

Admin tools sit outside the tabs behind a collapsed toggle. Recalculation rewrites every player's
rating, so it should take a deliberate second click rather than sitting under the thumb of someone
scrolling a scoresheet.

### Pre-match shows how fair the teams look

`TeamsRosterView` was a bare list of names. Names answer "who is playing" and nothing else; the
question people actually open an upcoming match for is whether the sides are balanced. Each player
now carries their skill rating and each side its average, with the gap stated outright rather than
left to be worked out by comparing two numbers.

Ratings come from the canonical players query via `select` — **not** from the match. `PlayerStatDTO`
carries the rating a player *earned in that match*, which does not exist yet before it is played.

The header gains a countdown (`countdownTo` in `utils.ts`). It is pure and takes `now` as an
argument rather than reading the clock, so two renders of the same match cannot disagree — the same
reason `react-hooks/purity` objects to `Date.now()` inside a component. `MatchModal` snapshots the
time once on open; the modal is short-lived, so not ticking is not a problem.

### CSS class groups

| Group | Prefix | Purpose |
|-------|--------|---------|
| Shell | `match-modal` | Dialog, header, close, subtitle |
| Scoreboard | `match-modal-score*` | Score display shared across modes |
| Roster | `match-modal-roster-*` | Upcoming-match player lists |
| Record form | `match-record-*` | Live goal entry, player rows, actions bar |
| Stats table | `match-stats-*` | Completed-match per-player table |

The stats table uses `min-width: 340px` with `overflow-x: auto` on its parent so it scrolls horizontally on narrow screens instead of wrapping.

The record form score row is `match-record-score-row`; the goal entry widget is `match-record-goal-widget` and its children use BEM `__` elements.
