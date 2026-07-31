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

Below 640px the table is replaced by cards (`.rankings-cards`), which show the record as a compact
`W-D-L` triple.

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
