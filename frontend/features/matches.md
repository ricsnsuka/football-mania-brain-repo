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

A `<select>` inside the shared pagination control (`components/ui/Pagination`, class
`pagination__size-select`) switches between 3, 5 and 10 matches per page. It is visible whenever the
page contains at least one match (`totalElements > 0`).

**Three is the default because a match card is tall** — scoreline, both squads, venue — so three of
them is about a screenful without scrolling. The larger sizes are for scanning back through a
season rather than reading the last few.

> Corrected 2026-08-05. This page had specified 3 from 3/5/10 since it was written and the code
> shipped a default of 10 from 5/10/20; the code was changed to match. The class named here was
> `matches-page-size-select`, which no component had used since pagination became a shared
> primitive — the rule that was still in `globals.css` has been deleted.

## Role gates

| Action | Roles |
|--------|-------|
| View matches | All |
| Create match | `MANAGER` |
| Bulk recalculate ratings | `GROUP_ADMIN` |
| Single recalculate ratings | `GROUP_ADMIN` |

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

The `RecalculateMatchesPanel` is rendered inside `MatchesPage` for `GROUP_ADMIN` only. It allows re-running the rating engine on completed matches.

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

### The scoresheet is one table, not two

Until `1.4.0` each team rendered **its own table**, sized by its own contents — so the rating column
and the amend button landed at different x-positions for the two sides, and the column heads were
printed twice in one panel. A reader comparing a Reds player against a Blues one was reading across
two grids that only looked alike. `MatchScoresheet` measures every column once and makes the team a
band inside the table rather than a second table around it.

The figures carry **no emoji**. A ball before the goals and a boxed red A before the assists were
decoration standing in a data cell — the red one read as an error state, and both render differently
on every platform. MVP and the winning side are chips; ratings carry `/10`, and colour marks the
best rating of the match rather than every rating, which only repeated what the column head said.

**No team colour anywhere.** The squads tab used to give each side a coloured dot assigned by team
*order*, so a team called "Reds" got a blue one. Team names are free text: any palette can contradict
the name beside it, so the band and the winner chip separate the sides instead.

### Three ways to correct a match

All added in `1.4.0`–`1.4.1`, and each is a different scope with a different grant:

| Control | Where | Grant | Writes |
|---|---|---|---|
| **Edit details** | Modal header | `MANAGER` | `PATCH /api/matches/{id}` — description, kickoff, location |
| **Edit sheet** | Scoresheet toolbar | `GROUP_ADMIN` | One `PATCH …/stats/{statId}` per changed row, on Save |
| **Edit lineup** | Squads tab | `GROUP_ADMIN` | `POST …/lineup/swap` or `…/lineup/replace` |
| **Delete match** | Below the tabs | `GROUP_ADMIN` | `DELETE /api/matches/{id}` |

**Each edit control sits in the surface it governs**, not in the tab bar. Edit sheet spent
`1.4.0`–`2.5.0` as a fifth child of the tablist: a button that is not a tab, counted as one by
assistive technology, in a row that neither wraps nor scrolls — so on a phone, where the four tab
labels already fill it, the button was pushed off the right edge and clipped mid-word. It now has
its own toolbar at the top of the scoresheet panel, matching the one the squads tab has carried
since `2.2.0`. The tab row scrolls rather than clips, which the translated labels need anyway.

**Editing the sheet stages its changes.** Each row's save used to fire its own request, and each one
replayed the rating engine over the whole match — so correcting three players meant three
reconciliations. Changes are now staged, changed rows marked while unsaved, and Save issues one
request per changed row **sequentially**, because two reconciliations at once would be two engines
writing the same rows.

It also sends **only the fields that were touched**. The previous form seeded the three goal-type
fields to zero and sent all six every time, so correcting somebody's assists wiped their goals — and
the scoreline, which the server derives from the stats, dropped with them.

Goals are stored as three types and returned only as a total, so the app cannot round-trip them: the
goals cell opens the three inputs on request, sends nothing when untouched, and says outright that
what you enter replaces the total shown.

**Editing the lineup is select-then-target.** Tap a player, then tap somebody on the other side to
swap them, or choose a replacement from everybody not already in the match. Tapping the same side
re-selects rather than erroring — far likelier to be a change of mind, and there is no one-way move
to offer, because the sides must stay the size they were.

**Deleting a played match says what it costs first.** The control used to be offered only for
matches that had *not* been played, which is the wrong half. The confirmation names what happens —
every player gets their rating, goals, assists and appearance back, their later matches are
recalculated, and none of it is reversible — and the server's report is repeated afterwards rather
than swallowed. Anything left needing recalculation is raised as its own error, because it is a job
somebody has to go and do.

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
| Scoreboard | `match-modal-score*` | Score display, result line, muted losing figure |
| Scoresheet | `match-sheet__*` | The single table, its team bands, chips, and edit-mode inputs |
| Details form | `match-details-form*` | Editing description, kickoff and location |
| Squads | `match-squads__*` | Side-by-side lists, and lineup editing |
| Record form | `match-record-*` | Live goal entry, player rows, actions bar |

Two focus rings are `focus-visible` rather than `focus` — the close button and the tabs.
`showModal()` moves focus to the first control, so every match used to open with a grey ring boxing
the ✕, and clicking a tab left a blue box around it, marking the selected tab twice.
| Stats table | `match-stats-*` | Completed-match per-player table |

The stats table uses `min-width: 340px` with `overflow-x: auto` on its parent so it scrolls horizontally on narrow screens instead of wrapping.

The record form score row is `match-record-score-row`; the goal entry widget is `match-record-goal-widget` and its children use BEM `__` elements.
