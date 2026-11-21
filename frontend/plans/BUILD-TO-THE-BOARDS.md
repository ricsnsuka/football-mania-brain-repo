# Building an app to its boards: the iPhone record, and the playbook for the web

Written 2026-11-21, when every live iPhone board was built and the differences review was done. It
records how the iPhone was taken from approved boards to a matching app (iOS#145 and the PRs after
it), every rule and decision the owner gave on the way, and how to run the same process for **web
desktop** and **web phone**, which come next.

## Where things are

| What | Where |
|---|---|
| The boards | Design canvas **sidekick-redesign**, https://claude.ai/artifact/9ELuLvS4nzeyBaqy9qtPHx. iPhone on the page "iPhone · made one"; web desktop and web phone on their own pages |
| A board's id and state | In its title: `<name> · <Id>  <mark>`. Marks: `○ Proposal`, `✓ Approved YYYY-MM-DD`, `✕ Superseded`, and `● Built` once the app matches it |
| Which code builds which board | `~/.claude/skills/mock-check/screens.json`, read with `screens.py` (`list`, `resolve`, `check`) |
| Recorded departures from a board | `docs/design/MOCK_DEVIATIONS.md` in the repo that built the screen. Append only; a settled entry gets `**Resolved:** date, reason` |
| The iPhone's shared parts | The canvas page "iPhone components", kept up to date as components change |
| Web desktop's record of what is done | The doc "Sidekick web-desktop redesign ledger", https://claude.ai/artifact/WgH2zhp2DTW8tGHb93wXfs |

## The process, as it ended up

It took one false start to get here. Steps 1–19 of the iPhone build (2026-11-20) were checked **by
reading the code and the board's words**, and 35 boards were marked built. The owner then opened
Compare and found it unlike its board. Every mark was withdrawn and the work was redone, checked by
picture. **A screen is checked by looking at it beside its board, never by words.**

1. **A board first.** Nothing is built, and nothing visible changes, without an owner-approved board.
   A new layout, a new control, or a design of the builder's own is drawn as `○ Proposal` and waits
   for "approve it, build it". Pagination, the compare chart, the AI report card, the Live Activity
   and the Reports filter all went this way.
2. **Check the board itself before building.** Run `board-check.mjs` (centred pill words, stray `<`,
   empty password fields, unclosed tags), then look at it beside a sibling of the same kind. A fault on
   the board is fixed on the board, not built into the app. Found this way: SignUp's terms box was
   not drawn; Forgot and Forced password had a white lead line where siblings use grey; the preview's
   "vs" had the wrong size; Compare said "Today" instead of a date; the ladder said "Placing" where the
   app says "Placement 3 of 5".
3. **Build exactly the board.** Words come from the board (`app.*` keys) over the web's shared strings.
   Reuse a component before writing a screen's own; a new one goes on the components page.
4. **Picture it with fixtures.** iPhone: the opt-in `BoardShots` tests
   (`TEST_RUNNER_FM_SHOTS=<dir> xcodebuild … -only-testing:'FootballManiaTests/BoardShots/<test>()'`)
   render each screen against a fake backend, 393 wide, dark. Render the board with the web repo's
   Playwright, put the two side by side, and compare part by part: what's there, order, words,
   colours, sizes, spacing, states.
5. **Mark `● Built` only when the pictures match.** Data differences (names, dates) don't count.
6. **A difference the builder can't resolve is recorded, not asked about each time.** It goes into
   `MOCK_DEVIATIONS.md` with what the board shows, what was built and why. Owner, 2026-11-20: "review
   all of them together after every board is built; don't ask per step".
7. **The owner's answer is drawn onto the board.** The board stays the spec, so the next check finds
   no difference. A board redrawn only to show the owner's choice is set to
   `✓ Approved <today>  ● Built` at once, with no separate approval round.
8. **Commit and push each step**, with its changelog entry. Run the full suite once per batch.
9. **The differences review at the end**: every open entry, each with a recommendation, answered by
   number. See below.

### Practical notes

- **Don't check light mode** unless asked (owner, 2026-11-21). IosTableLightOne is the one iPhone board
  left unbuilt for that reason.
- **The Simulator's test runner can hang** ("hung before establishing connection") after manual boots.
  `xcrun simctl shutdown all; killall -9 com.apple.CoreSimulator.CoreSimulatorService` clears it.
- **The owner reviews pages on his own Simulator**, so don't spend a session on Simulator captures.
- **Never pipe a build or a test run**; redirect to a file and read the exit code.
- **A Live Activity can't be fully pictured here.** This Mac has no Simulator app to lock the screen:
  the compact island was photographed from a real activity, the Lock Screen card only as a rendered
  view.

## The owner's standing rules (all platforms unless noted)

- **An approved board is the spec**; build exactly it, and check by picture.
- **Board wording wins** over the web's shared strings (iPhone: `app.*` keys).
- **Think in components.** Reuse before writing; keep the components page current.
- **A screen-wide filter or scope is never in the bar or inside the list.** It is a card of its own
  ("Season · Season 2 ⌄", "Show · All 6 ⌄", "Include ones looked at").
- **Every page title is in the bar, except Home's** (iPhone, 2026-11-21).
- **A ⋯ whose contents nobody decided is removed**, not filled with guesses: Match day, Match created,
  Balances (everyone and one player's), the group-wide chat.
- **The losing side's score and name are grey** on a finished match; a draw greys both; live and
  unrecorded stay white. Every screen, every platform.
- **The tab bar shows on every screen**, so forms are pushed screens, not sheets (iPhone).
- **Dates and times are picked in place**, a calendar or wheel opening under its row, never a pop-up
  (iPhone).
- **Every tab opens on its first screen**, not where it was left; Matches reopens on Coming up and
  Table on League (iPhone, 2026-11-21).
- **Short weekdays are three letters in Portuguese and Spanish** ("qua", "mié"), never the whole word
  (iPhone, 2026-11-21).
- **Lists of matches show five a page**, with a "‹ Page 1 of 4 ›" stepper; a list of five or fewer has
  no pager (iPhone Matches, 2026-11-21).
- **"Added by" and "Started by" show usernames**, not names (2026-11-21).
- **The two-tone wordmark**: SIDE in ink, KICK in the accent, everywhere.
- **The losing-side rule, the score header and the figure face** (Barlow Condensed for figures) are
  shared across iPhone and Watch.

## Decisions taken while building the iPhone

By area, with the date. Smaller ones are in the commit messages of iOS#145.

- **Look (2026-11-15 to 20):** the web's colours but native iOS parts; figures in bundled Barlow
  Condensed (five sizes: 72, 40, 30, 24, 20); a system tab bar tinted lime; every board redrawn
  natively (glass round bar buttons, inset grouped lists, capsule switches, pull-down menus, swipe
  actions, context menus).
- **Home:** a bell at the top right opening a sheet; the waiting card's rows padded; the
  player-of-the-season line remembered on the device.
- **Matches:** search over Coming up; swipe left to answer; drawn plans shown as their match's row;
  Played's season as its own card; swipe to Record or open the sheet, press and hold to Share; the
  "Ratings stale" chip kept and drawn; five a page.
- **A match:** one score header everywhere (names under their numbers, the when line always kept);
  match chat in the ⋯; the AI report card folds from its heading, remembered per kind; before kickoff
  the score header with "vs", then "Ones to watch", then the squads.
- **Match day and the Live Activity:** no ⋯; the Live Activity shows the score and the last goal (the
  iPhone has no match clock), starts with match day, moves only with goals recorded on that phone,
  stands an hour at full time with the loser grey, and opens match day when tapped.
- **Plans:** the pinned answer kept; Bring a guest and Answer for others in the ⋯; Charge players kept
  under the cost field; no guest limit (below).
- **Teams and the draft:** the preview shows each side's players under the averages; the draft's rung
  in its tier colour with no bar; iOS 26 popover dialogs kept; averages hidden from non-managers with
  the ladder on (the server withholds ratings).
- **Table:** the season as its own card on every part; with the ladder on, each row a card tinted in
  its tier, never lime; Honours one week and one month at a time, empty weeks grey.
- **Players and profiles:** one profile screen however it's reached; Compare with a rating chart (you
  lime, them grey, the last nine matches) above the season switch.
- **Balances:** full pages, not sheets; one "Pay" button; three per page on Mine; credit counts as
  settled; void only by press and hold, no reason asked; picking ledger lines to record or void.
- **Chat:** swipe to leave or delete; Block from Who's around; no "Add people" on a group conversation
  (it isn't a priority, and the server can't yet); no ⋯ on the group-wide channel.
- **Settings:** Group settings keeps a "Matches" heading and the AI disclosure; notifications in the
  web's order; Reports' filter as its own card.
- **Sign-in:** no passkey line until passkeys exist (brain#151).
- **What's new** is off until further notice (iPhone and web).

## The differences review (2026-11-21)

Twelve open differences, each with a recommendation; the owner answered by number.

| # | Difference | Decision |
|---|---|---|
| 1 | "Ratings stale" chip on Played rows | Keep; drawn on IosPlayedOne |
| 2 | Teams preview lists each side | Keep; already drawn |
| 3 | Charge players on a plan's cost | Keep; the board's note says where it sits |
| 4 | Draft rung: colour and bar | Colour it, no bar at all |
| 5 | "Guests per member" on Plan a match | **Retire the guest limit entirely**: brain#153 with Back#458 (backend first) and Front#390; the iPhone hides it now; row off the board |
| 6 | "Guest · brought by Ana" | Built from data the app already had (`invitedByPlayerId`); no backend change needed |
| 7 | "Add people" on a group conversation | Hidden until the server supports it; not a priority |
| 8 | "Share of the pitch" under ledger charges | The app shows it under every charge from a plan. A match made by hand gets it once the server sends the charge's `matchId` (stored, not yet sent); the app already reads it |
| 9 | One-word initials ("R" vs "RI") | Two letters, as the board |
| 10 | "Added by" / "Started by" | Usernames; boards redrawn |
| 11 | Draft averages for non-managers, ladder on | Keep the server's privacy rule; noted on the board |
| 12 | ⋯ on the group-wide channel | None, as already drawn |

Also fixed without a decision: the lead card padded 18 as the boards draw it (Honours' week cards
16). A guest limit of 0 used to mean "members only"; retiring the limit removes that option too.

## Running it for web desktop and web phone

The process is the same; the tools differ.

- **Boards:** the web desktop and web phone pages of the same canvas. Web desktop has its own ledger
  doc (above): read it before changing a screen and update it after, in the same turn.
- **Pictures:** the web's visual scenes, `e2e/scenes.ts`, run with
  `npm run test:visual:host -- --grep <scene>` for a look. Their pixels are not the Docker baselines.
- **The catalogue:** `screens.py list web-desktop` / `web-phone`; `resolve` prints each screen's route
  and files.
- **Departures:** the web repo's own `docs/design/MOCK_DEVIATIONS.md`.
- **Carry over the rules above** that apply to the web: filters as their own card, removing ⋯ with no
  decided contents, the losing side grey, the wordmark, board wording, components first. The
  iPhone-only ones (tab bar, in-place pickers, tabs at root) don't apply unless the owner says so.
- **Start from what the iPhone decided.** Where an iPhone decision changed a behaviour both apps share
  (no guest limit, usernames for "added by", the three-letter weekday), check the web does the same,
  and file it if not.

## Still open

- The server sends `matchId` on a charge from a match made by hand, so "Share of the pitch" shows there too.
- The board's Epic field needs an option for brain#153, which the owner adds by hand.
- The "iPhone components" page lacks the newest parts: Pager, RatingCompareChart, StatePage,
  PasswordFields, the Live Activity views.
- IosTableLightOne, until light mode is checked.
- Back#458 and Front#390 (the guest limit), passkeys (brain#151).
