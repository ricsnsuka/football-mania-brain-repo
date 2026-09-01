# Dashboard Feature

An overview of what is happening and what needs you. One page for every role.

> **It used to be three dashboards, and they were a second navbar.** `AdminDashboard`,
> `MasterDashboard` and `BasicDashboard` each rendered a grid of `DashboardNavCard`s linking to
> Players, Matches, Match Plans, Team Generation and Users — every one of which the navbar already
> offers. A lot of screen, no information. All four components are gone.

## What it answers

| Question | Widget |
|----------|--------|
| Is anything waiting on me? | `AwaitingYou` |
| When is the next match, and am I in it? | `NextMatchSection` (plan → `NextMatchCard`, settled/live → `SettledMatchCard`) |
| How am I doing? | `YourStats` |
| What happened recently? | `RecentResults` |

None of that is role-specific, which is why there is one component rather than three. The only
role-dependent piece left is the admin panel of unlinked players — genuinely an admin's outstanding
work rather than a link to a page.

## Widgets

### AwaitingYou

Renders **nothing at all** when nothing is pending, rather than an empty "you're all caught up"
card. A panel that is always present stops being read; one that appears only when it means
something gets looked at.

It can carry three things, in this order:

1. **Not linked to a player.** First, and phrased as a consequence rather than an instruction —
   until it is fixed, that user's stats, badges and MOTM vote all silently do nothing, and nothing
   else on the page can ever fill in. Links to `/settings`.
2. **An open MOTM vote**, with its deadline. The deadline is *why* it is urgent, so it is shown
   rather than implied.
3. **An open draft session.**

### NextMatchSection (the match spotlight)

One section, three shapes, chosen by `pickSpotlight()` (pure, unit-tested) in `useDashboardData`:

- **A plan is next** (`NextMatchCard`): the chronologically next upcoming plan, **`PENDING` or
  `CONFIRMED`**. It used to be `PENDING` only, which treated confirming a plan as if it had already
  become a match — confirming Friday's game made the dashboard skip ahead to some later pending
  plan. RSVP is answerable **here**, not only on the Match Plans page: the point of an overview is
  that the obvious next action does not require navigating somewhere else first. `pollOpen: false`
  replaces the buttons with an explanation — a plan can stop taking responses while still being
  pending, and live buttons then would be a lie.
- **A settled match is next** (`SettledMatchCard`): teams generated, `Match` row waiting to be
  played. Match details — kickoff, venue, match type, team names — instead of an RSVP card;
  confirmations are settled business by then. Settled matches and plans compete on kickoff time
  alone; a dead heat goes to the settled match as the more decided fact.
- **A match is live**: the section becomes a horizontal snap carousel with dots — the live match
  first (title **Live match**, live chip, running score), the next fixture behind it. Auto-rotates
  every 5 s; each panel carries its own title, so the heading switches with the panel. Paused on
  hover/focus, interval restarts on a manual swipe, honors `prefers-reduced-motion`. "Live" is
  `isLive()` in `types/match.ts`: uncompleted, and either a running score or a recorded kickoff
  (`kickedOffAt`) without a recorded full time.

The tour's `data-tour="next-match"` anchor is on the section wrapper, not on any card.

### YourStats

Matches, goals, assists, streak, and up to four badges. **Renders nothing when unlinked**: a row of
dashes reads as "you have played nothing" rather than "we do not know who you are", and
`AwaitingYou` covers that case with something actionable.

Badges are silent when empty, which is the normal state until an admin has run the backfill — see
[badges](badges.md).

### RecentResults

The last three completed matches, scores only. Gives the page something to say when there is no
upcoming match, which is most of the week. Who played and how they rated is one click away;
repeating it here would make this a second Matches page.

## Data

All reads live in `useDashboardData`, not in the widgets. Every widget needs a slice of the same few
queries, and having each one call them independently made the fetch pattern impossible to see.

| Source | Notes |
|--------|-------|
| `usePlayers` | Shaped with `select` off the one canonical query — no extra request |
| `useMatchPlans({ status: 'PENDING,CONFIRMED', timeframe: 'upcoming', size: 5 })` | Server windows and sorts ascending — the old unbounded `PENDING`-only query could page-truncate the real next plan behind stale ones |
| `useMatches({ completed: false, size: 50, sort: ['matchDate,asc'] })` | Group-wide open matches: the live one, and settled fixtures. `useMatches` polls every 30 s while any listed match is live |
| `useMatches({ completed: true, size: 3 })` | |
| `useDraftSessions` | |
| `useMvpVote` | **Most recent completed match only** — see below |

> **Only the newest completed match is checked for an open MOTM poll.** Vote state is per match and
> would otherwise be one request per match on every dashboard load. With a window measured in hours
> two matches can be open at once, and this shows only the newer one; the full list is on the
> Matches page. Catching the common case beats N requests.

`Date.now()` is snapshotted once on mount via `useState(() => Date.now())` rather than read during
render. Reading the clock while rendering is non-deterministic — two renders of the same data could
disagree about which plan is "next" — which `react-hooks/purity` correctly rejects. The cost is that
a dashboard left open past a kickoff keeps showing that match until something re-mounts, which beats
silently retiring the card while somebody is looking at it.

## File map

| Layer | File |
|-------|------|
| Route | `src/app/(app)/dashboard/page.tsx` |
| Page component | `src/features/dashboard/DashboardOverview.tsx` |
| Reads | `src/features/dashboard/useDashboardData.ts` (incl. `pickSpotlight`) |
| Widgets | `AwaitingYou.tsx`, `NextMatchSection.tsx`, `NextMatchCard.tsx`, `SettledMatchCard.tsx`, `YourStats.tsx`, `RecentResults.tsx` |
| Admin panel | `src/features/dashboard/UnlinkedPlayersPanel.tsx`, `LinkPlayerModal.tsx` |

`LinkedPlayerBanner` and `SelfLinkPlayerModal` moved to `src/features/settings/` — managing your
player link is an account action. See [settings](settings.md).

## Route guard

`AuthGuard` only. Every role sees the same page; the widgets decide what is worth showing.

## What is deliberately *not* here

- **Nav cards.** The navbar does this.
- **Notification and privacy settings.** They rendered below the role dashboards; they are account
  concerns and live on `/settings` now. Having account deletion at the bottom of the page everybody
  lands on was the wrong place for it.
- **Create-player and create-match shortcuts.** The Players and Matches pages have FABs.

## CSS

`.overview*` and `.awaiting*` in `globals.css` — including `.overview-carousel*` (the live
carousel: hidden-scrollbar snap track, 44 px dot buttons) and `.overview-match__*` (the settled
card's teams row). `.dashboard-greeting` and `.dashboard-role-badge` survive from the old page and
are still used by the header.
