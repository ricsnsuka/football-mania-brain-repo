# Rankings Feature

The competitive view: a league table ordered by skill rating, plus four all-time category
leaderboards. Route `/rankings`, any authenticated user.

> **This is not the Players page with more columns.** They answer different questions and are
> deliberately kept apart — see [Relationship to the Players page](#relationship-to-the-players-page)
> below, which is the section to read before adding anything to either.

## Two endpoints, two shapes

| Endpoint | Is | Answers |
|----------|-----|---------|
| `GET /api/rankings` | One ordered list of players | "Where do I stand?" |
| `GET /api/leaderboards` | Four short category lists | "Who is best at X?" |

They are separate calls with separate cache entries because they change at different rates. See
`docs/api/LEADERBOARDS-API-CONTRACT.md` in the backend repo.

## The season scope

**The default is the current season, not all-time**, and this is the only screen where that is the
right default: the table answers *"where do I stand"*, and "now" is a season. The selector offers
every season plus All time, and it is rendered only once there is more than one season to choose
between — a group in its first season has one answer, and a control with two identical outcomes is
clutter.

`GET /api/rankings?seasonId=` does the bounding; the response echoes `seasonId` back and that is
what the page branches on, because the two tables are the same shape.

**The streak columns stay all-time in every scope**, and the modal says so. A run of wins is a fact
about a player and closing a season is an administrative act — ending a streak because of one would
take away something nothing on the pitch did. This is the single place the season table disagrees
with its own heading, deliberately, and it is labelled rather than silent.

The query waits for the season list before firing. Without that the page fetches the all-time table
and then immediately replaces it with the current season's: two requests, and a visible switch
between two tables that look identical.

### The modal shows what the table showed

Clicking a row opens the player modal on **the row's own figures**, passed through as a
`PlayerStatsScope` rather than re-fetched. A modal that always showed career totals contradicted
the table the reader had just clicked — the same name with different goals beside it, one row
apart. The players page passes no scope and gets all-time, which is all that page has ever had.

Only four fields are scope-dependent: rating, appearances, goals, assists. Positions, keeper
willingness and the link state are properties of the person, and the streaks are all-time, so none
of them are in the scope.

## The order comes from the server and is never re-sorted

**There are no sortable columns, and that is the feature.** The backend returns qualified players
in rank order followed by unqualified ones, precisely so a player with one lucky match does not
appear at the top. Sorting this array client-side by `skillRating` would pull them straight back up
and undo the threshold entirely.

Free-form sorting lives on the Players page, which has no ranking semantics to break.

## Qualification

A player needs **3 completed matches** to be given a position. Below that they are still listed —
a newcomer must be able to find themselves — but rendered with:

- `—` instead of a rank
- a "N more to qualify" hint under their name
- the whole row dimmed (`.rankings-row--unranked`)

The threshold comes from `minimumMatchesToQualify` in the response, **not a frontend constant**, so
the hint stays truthful if the backend changes it.

## The omitted-null trap

`rank` is **absent** from an unqualified entry, not `null` — the API sets
`spring.jackson.default-property-inclusion: non_null`. It arrives as `undefined`, so
`rank === null` is false for every unranked player.

Every branch keys off **`qualified`**, a boolean that is always present. The zod schema normalises
the absence via `nullableField`, but `apiFetch` fails open on a mismatch and returns the
unvalidated body, so the boolean is the only thing safe to trust. `src/tests/rankings/` pins this.

## Table columns

| # | Column | Source | Notes |
|---|--------|--------|-------|
| 1 | `#` | `entry.rank` | 🥇🥈🥉 for the top three, `—` when unqualified |
| 2 | Name | `entry.playerName` | Carries the "N more to qualify" hint when unranked |
| 3 | P | `entry.played` | Career completed matches |
| 4/5/6 | W / D / L | counted from `player_stats.match_result` | May sum to **less** than `played` if a completed match carries no result |
| 7 | Goals | `entry.goals` | Own goals excluded |
| 8 | Assists | `entry.assists` | |
| 9 | Rating | `entry.skillRating` | Two decimals |

Columns 3–9 are all figures and are **right-aligned, headers included**, so the values line up on
the units digit. They carried `tabular-nums` from the start, which on its own buys nothing —
equal-width digits only pay off once the column has an edge, and left-aligned they still put the
units digit of `5` under the tens digit of `13`. Fixed 2026-08-05; see rule 8 in the frontend
[styling guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/styling.md).

Below 640px the table is replaced by cards (`.rankings-cards`), which show the record as a compact
`W-D-L` triple.

## Paging

Ten rows a page, from 10 / 25 / 50.

`GET /api/rankings` answers with the **whole** table — one cached document whose ranks are computed
against the full field — so this is a window onto an array that has already arrived, not a request
per page. Changing page fetches nothing and the server is never asked for a slice.

**Ranks stay the server's: page two starts at 11.** Renumbering per page would make its first row
"1", which is a different claim about that player.

The page index is **clamped, not reset**. Switching back to active-only shortens the list under the
reader, and an index past the end renders an empty table for somebody who has rows. Clamping keeps
them as close as possible to where they were; the two controls that change *which players exist* —
the active/inactive filter and the page size — do reset to page 0, because staying on page 4 would
land them in the middle of a different table.

One control serves both layouts. The table and the mobile cards are the same list at two widths and
only one of them is ever on screen.

## Leaderboards

Four cards: top scorers, top assists, most MVPs, longest streaks. Each renders from its own list
and is **always shown**, displaying "Nothing recorded yet" when empty — hiding an empty category
makes the grid jump around early in a season and leaves no clue the category exists.

`value` carries a different unit per card, so each labels its own ("goals", "assists", "MVPs",
"matches"). A list shorter than `limit` means the category ran out of non-zero entries, not that
there is another page.

**No qualification threshold applies here.** Goals and assists are counting stats, not rates — five
goals in one match is a real five goals — so an unqualified player can top the scorers list while
being unranked in the table above. That is correct and visible in the fixtures.

## Active vs inactive

| | Deactivated players |
|---|---|
| League table | **Excluded** by default; "Include inactive" toggle |
| Leaderboards | **Always included** |

The table describes the group as it is now; a record board is the opposite — whoever scored the
most goals still scored them after they left. GDPR erasure deactivates the player it anonymises, so
an erased person leaves the table by that filter rather than by any privacy-specific rule.

## Relationship to the Players page

| | Purpose |
|---|---|
| **Rankings** | The competitive view. Standings, form, category leaders. Read-only. |
| **Players** | The **roster**. Who is in the group, active/inactive, account links — and the route into a player's record. |

The Players page deliberately does **not** show medals or default to skill-rating sort: that is
this page's job, and having both do it made two lists look like the same thing. Rankings rows open
the player modal, so the richer page is not a dead end.

## File map

| Path | Role |
|------|------|
| `src/app/(app)/rankings/page.tsx` | Route, wrapped in `AuthGuard` + `PageContainer` |
| `src/features/rankings/RankingsPage.tsx` | Table, filter, mobile cards |
| `src/features/rankings/LeaderboardCard.tsx` | One category card |
| `src/hooks/ranking/useRankings.ts` | `useRankings`, `useLeaderboards` |
| `src/services/rankingService.ts` | `fetchRankings`, `fetchLeaderboards` |
| `src/types/ranking.ts` | Types + zod schemas |

## i18n keys (`rankings` namespace)

`title`, `tableTitle`, `filterActive`, `filterAll`, `summary`, `unranked`, `needsMore`,
`columns.*`, `leaderboards.*` (including `units.*` and `empty`).
