# One job per screen: the September 2026 screen audit

Audited 2026-09-24 on FootMania-Simple-Front `next` (7ac0b45); worked through 2026-09-24 to
2026-09-25 in FootMania-Simple-Front #241–#267, with FootMania-Back #361–#362. **Done**: every step
below is merged into `next`, and nothing of it is released yet. What the work taught is in
[development-learnings](../../development-learnings.md).

This page is the plan and its outcome. The interactive version, with the plan-dialog x-ray and the
mock-ups, is a private claude.ai page; this is the copy that stays with the repos.

---

## The rule

- **One question, one audience.** A component's job can be named without saying "and". "Am I
  playing?" is one job; "shows the plan and lets you answer and edit it and set the cost and draw
  teams" is five.
- **Permissions decided once.** Roles become named abilities (`canRelease`, `canSetCost`) in one
  hook per screen. Components receive the abilities and never read roles themselves.
- **Same structure for everyone.** A role adds actions to what is on screen. It does not swap in a
  different list or layout, so somebody holding two roles, and the tests, see one shape.

## What the audit found

The largest screens by line count, and the pieces each held:

| Screen | Lines | Pieces | Verdict |
|---|---|---|---|
| Match plan dialog (`MatchPlanDetailModal`) | 918 | 13 blocks, 12 data hooks, 12 permission and state flags, one of 3 player lists by role and phase | Split first |
| Team generation (`TeamGenerationPage`) | 663 | Two ways to make teams stacked on one page, two plan pickers, a role router, a page that turns into the live draft board | Split |
| Rankings (`RankingsPage`) | 639 | Season scope, table and cards, leaderboards, honours reached from three places | Tidy |
| Player dialog (`PlayerModal`) | 540 | The profile, 5 admin mutations, 3 destructive areas, an Edit dialog on top | Split |
| Match dialog (`MatchModal`) | 514 (1,573 with its parts) | 3 modes with 3 layouts, 3 edit modes with buttons in 3 places, 4 tabs once played | Split |
| Captains' draft board (`CaptainPickBoard`) | 470 | The stream, turns, picks, confirm and cancel, 4 helpers inside the file | Tidy |
| Players page (`PlayersPage`) | 444 | Three filters in three styles, a table and cards that repeat each other, an unlabelled "+" | Tidy |
| Match plans list (`MatchPlansPage`) | 263 | Finding a plan | Mostly fine |

Six bugs, checked in the code and fixed in #243: a plan with teams drawn got the "cancelled" chip;
the player dialog carried open forms from one player to the next; a manager could start a captains'
draft they could not finish; starting a draft did not refresh the list of drafts; recording a result
saved each goal at once and failed silently; the captain pickers offered reserves the server refuses.
A seventh finding, Delete on a played match, turned out to be intended.

## The ten patterns

1. **Show the phase.** A stepper says where a plan or match is and what comes next.
2. **One list, actions by role.** One structure for everyone; roles add row actions behind "⋯".
3. **Permissions in one hook per screen.** `usePlanCapabilities`, `useMatchPermissions`,
   `useDraftRole`, `usePlayerPermissions`, each table-tested.
4. **One primary action, one danger zone.** The phase's main action in a footer; destructive
   actions last and apart.
5. **No dialog on a dialog.** Edit details, Bring a guest and Team of the week each opened on top of
   another dialog.
6. **A URL for anything people link to.** Plans, matches and players, so notifications land on the
   item and Back works.
7. **Say why a button is greyed out.** The reason in a line under it, not a tooltip.
8. **One meaning per word.** "Confirmed" was both the plan's status and a player's answer.
9. **Name the create buttons.** No bare "+".
10. **Teams made in one place.** Drawn from the plan, both methods behind one switch, one path to a
    match.

## The steps, as they went

PR numbers are FootMania-Simple-Front's unless named.

1. **A screenshot baseline** — #241, #242. Forty reviewed baselines, then fourteen captures of the
   screens the split would touch, so every later step could prove what it did and did not move.
2. **The six bugs** — #243.
3. **The permission hooks** — #244. Four pure functions, 681 table cases, no screenshot moved.
4. **The plan dialog split without changing how it looks** — #245.
5. **The plan dialog changed** — #246 phase stepper and footer; #247 Squad, Teams and Money tabs;
   #248 one squad list with row menus and "Answer for someone"; #249 with FootMania-Back#361 a link
   per plan, `/match-plans?plan=12`, which the plan notifications open. A query rather than a route:
   the backend deploys first, and an older frontend ignores a query but answers an unknown path with
   "not found".
6. **Team-making in the plan** — #250 one plan, one pool, Generate and Captains' draft behind one
   switch, a finished draft asking before it creates the match; #251 the plan's Teams tab hosts it,
   and Team generation lists the plans that need teams.
7. **Match dialog, player dialog, rankings** — the same recipe for each:
   - Match: #252 split; #253 one "⋯" menu and one edit at a time; #254 with FootMania-Back#362 a link
     per match, `/matches?match=40&tab=motm`, opened by the teams, result and vote notifications;
     #255 the man of the match named once, by the leaderboards' rule.
   - Player: #256 split; #257 one menu, each task in place of the profile, the player read live;
     #258 a link per player; #259 one account picker, a search over the members.
   - Rankings: #260 split; #261 the tab and season in the address and "Find me".
8. **The two Tidy screens** — Players: #262 split; #263 labelled create buttons, one filter style,
   "Clear filters", fuller phone cards. Draft board: #264 split with its first captures; #265 the
   turn line, the turn chip and the page heading those captures showed wrong.
9. **What needed no owner decision** — #266: bringing a guest in place of the squad instead of a
   second dialog; the reason under a greyed-out Generate, Start the draft or Create match; an empty
   week naming the next plan; Match chat opening its conversation; no focus ring on a dialog opened
   by a link; "+ New Match".
10. **The owner's calls** — #267: the honours as the rankings' third tab (the week, the month, the
    season's awards and the Ballon d'Or, newest first); "On" for a plan that is on and "In" for a
    player's answer; the draft board's pick arrows pointing up and down on a phone. Brain repo #94
    moved the seasons doc with it.

## Decisions the owner made

- **What's new** rows for #253–#260 (match menu, man of the match, match and player links, player
  menu), for the plan link (#249), and for the Honours tab (#267). None for the rest.
- **Honours as a tab** (2026-09-25), where 3.13.0 had placed them in three places. The 3.13.0
  walk-through was re-pointed at the tab.
- **"On" and "In"** (2026-09-25), the wording the audit proposed.
- **Any manager can finish a captains' draft** (#243), as the server always allowed; the old
  captain-or-admin rule was dropped.
- **The manager keeps the squad as the plan dialog's default view** (#248).

## Not done, on purpose

- **A page per plan, match or player.** The audit drew `/match-plans/12`; each is a query on its list
  page instead, for the deploy-order reason in step 5.
- **The match plans list** was left alone apart from its create button and the empty week: it
  already did one job.
