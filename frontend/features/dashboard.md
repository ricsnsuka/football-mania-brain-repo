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
role-dependent pieces left are the manager's count of unlinked players and the organiser's
payments to confirm: outstanding work for the one role that can clear it.

## One order, one visibility rule (web and iPhone)

Decided 2026-11-08 (gap IOS-09, FootMania-iOS#97). The web's dashboard and the iPhone's Home show
the same cards, in the same order, to the same people. Both read them from one request,
`GET /api/me/home` (backend `docs/api/HOME-API-CONTRACT.md`), so the selection rules live on the
server.

| # | Card | Shown to |
|---|------|----------|
| 1 | Greeting, with your roles | Everyone |
| 2 | Waiting on you | Everyone, when something is waiting |
| 3 | Ballon d'Or vote | Everyone in the group, while a poll is open |
| 4 | Ballon d'Or result | Everyone in the group, for 14 days after it closes |
| 5 | Live match, then next match or plan | A player, a `GROUP_ADMIN` or a `MANAGER` |
| 6 | Your numbers | A player |
| 7 | Recent results (your own) | A player, when there are any |
| 8 | Payments to confirm | An `ORGANIZER`, when there are any |

The iPhone also shows the Watch's inbox under the greeting, which has no web counterpart.

- **The votes sit above the next match** because they are the same kind of thing as "Waiting on
  you": something to do before a window closes. The iPhone used to put them below it.
- **The match card is group information**, so whoever runs the group sees it whether or not they
  play. An account that is none of player, admin or manager gets the "link a player" line instead.
  The iPhone used to show the card to everyone.
- **Inside "Waiting on you"**, the server decides the role lines: payments to confirm go to an
  `ORGANIZER`, and unlinked players to a `MANAGER`. The iPhone used to count them for
  `GROUP_ADMIN`. Who may link a player from that line is gap WEB-04 (FootMania-Simple-Front#327).
  The iPhone leaves guests out of the count; the server and the web include them, which is still
  to be settled.
- **The MOTM vote offered** is the one on your own latest completed match, on both clients.
- **A section the server could not read** (`unavailable`) says "couldn't be loaded" with a retry,
  in its card's place, never its empty state. A failed player read is not "not linked".

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

One section, three shapes, chosen by the server (`spotlight` in `GET /api/me/home`, `HomeServiceTest`):

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

One read, `useHome` over `GET /api/me/home`, shaped for the cards by `useDashboardData`
(FootMania-Simple-Front#337). It used to be four queries there and nine more in the cards, and the
skeleton waited only for the first four, so the rest popped in (gap WEB-03). Now the page is either
loading or whole.

- **Staying fresh.** Every successful mutation marks Home stale (a `MutationCache` in
  `QueryProvider`), and so does every push about the group (`queryKeysToInvalidate`). While a match
  is live, Home is re-read every 30 s.
- **Not in Home**, by the contract: the unread chat count, which follows chat's own stream.
- **The MOTM vote** is the one on your own latest completed match, chosen by the server. Only one
  vote is offered even when two are open; the full list is on the Matches page.

## File map

| Layer | File |
|-------|------|
| Route | `src/app/(app)/dashboard/page.tsx` |
| Page component | `src/features/dashboard/DashboardOverview.tsx` |
| Reads | `src/hooks/home/useHome.ts`, shaped by `src/features/dashboard/useDashboardData.ts` |
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
