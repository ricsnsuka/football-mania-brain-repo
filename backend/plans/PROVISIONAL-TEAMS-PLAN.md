# Provisional Teams — Plan

**Date:** 2026-09-24
**Status:** 📋 **DECIDED, not built.** The owner took every decision below on 2026-09-24. Nothing is
implemented and no epic is filed yet.
**Effort:** L — three backend PRs and two frontend PRs, with C (fewest-moves repair) following on its own
**Depends on:** nothing. It touches `MatchPlanService`, `DraftSessionService`, `MatchFeeService`,
chat and push.
**Contracts (to write with the code):** a new `docs/api/MATCH-PLAN-PAIRINGS-API-CONTRACT.md`, plus
changes to the chat, push, payments and poll-window contracts.
**Design page:** the owner's comparison and design page, "Squads After the Draw"
(https://claude.ai/artifact/78JG55MEGvnNExpGj2k2rz). It is private to the owner, and this file is the
record.

---

## 1. The problem

Generating teams ends a match plan. `POST /generate/confirm` does four things in one transaction:
the plan becomes `GENERATED` (terminal), the poll shuts (`MatchPlan.acceptsAnswers()` excludes
`GENERATED`), the match and its teams are created, and the pitch fee is charged. A manager who draws
early, and then hears that someone can't come, has no way back. They can swap or replace on the match
(`GROUP_ADMIN` only, with no balance guidance and no fee change), or delete the match (the plan stays
`GENERATED`), or start a new plan and add everyone again, which is what actually happens.

Filling the gap alone breaks the balance the draw was for. In the worked example on the design page,
a reserve filling a strong starter's slot took a Balanced split from Δ 0.04 to Δ 0.56.

## 2. What was decided, and what was rejected

Five approaches were compared on 2026-09-24. The owner chose **D, provisional teams saved on the
plan**, with **C, a fewest-moves repair**, later.

| Rejected | Why |
|---|---|
| A. Undo the draw (GENERATED → CONFIRMED) | Deletes and recreates the match, so links and the match chat go. Fees are voided and charged again, and players still can't drop out in the app while the plan is GENERATED |
| B. Keep the squad open after the match exists | GENERATED stops being terminal for the squad, and every change has to edit the match and move a fee. Worth it only if the match must exist early |
| E. Scheduled automatic draw | Doesn't solve a withdrawal after the draw. Decision 7 covers the other thing it would have fixed |

The owner's requirements, in their numbering:

1. **Teams are saved against the plan, temporarily.** A manager opens the plan's pairings and picks
   one to create the match.
2. **Team generation creates provisional teams, except captain-pick drafts.** A draft still converts
   straight into a match. Drafts keep the late-replacement gap, and that is accepted.
3. **The provisional teams are deleted when the match is created.**
4. **On a replacement, the provisional teams are flagged**, and regenerated on demand with the model
   they were created with.
5. **On match creation, a message in the match chat with both teams, and a push to each player with
   their team and its members.**
6. **The plan shows a pill while provisional teams exist.**

And the answers to the open questions:

| # | Question | Decision |
|---|---|---|
| 1 | C in the first release? | **Later.** Regenerate ships first, and C follows in its own PR |
| 2 | Who posts the teams in the chat? | **A system sender**, not the manager |
| 3 | Announce draft-picked matches too? | **Yes.** The same event from `confirmDraft` |
| 4 | Who sees the pill? | **Everyone.** Only managers open the pairings (§5 records the reading) |
| 5 | Pill wording | **"Provisional teams"**, translated: "Equipas provisórias", "Equipos provisionales" |
| 6 | The reserve cutoff | **In this epic** |
| 7 | Nobody creates the match before kickoff | **Allow it** for a grace window after kickoff |
| 8 | Who is charged at match creation? | **Starters only**, meaning the team sheet. A roster change after the match is the organiser's to fix with the existing void and add-charge tools |
| 9 | The draft that ignores withdrawals | **A separate bug:** [FootMania-Back#352](https://github.com/ricsnsuka/FootMania-Back/issues/352) |
| 10 | "What's new" highlight | **Yes, with role restrictions** |

## 3. How a plan's week goes

1. **Save.** On team generation the manager previews Balanced, Random, Optimal or a manual split, and
   saves the ones worth keeping. The plan stays `CONFIRMED`, so the poll, the waitlist and the
   `RESERVE_PROMOTED` push all keep working unchanged. The plan shows "Provisional teams".
2. **Someone drops out.** The first reserve is promoted exactly as today. Every pairing now names
   someone who isn't a starter, so every pairing reads as flagged.
3. **Regenerate.** The manager opens the plan's pairings and regenerates one. The stored type and
   params run again on the current starters.
4. **Create the match.** The manager picks a pairing. The server checks it against the current
   starters, creates the match, marks the plan `GENERATED`, charges the team sheet, deletes every
   pairing of the plan, and after commit posts the teams in the match chat and pushes each player
   their team.

`GENERATED` keeps its meaning (terminal, and one plan means one match). That is the main reason D
won over B: the guard, the docs and the frontend types don't change.

## 4. Backend

### Data (V61, additive, not a rollback boundary)

- `match_plan_pairings`: id, tenant, plan, generation type, params, who saved it, when, and when it
  was last regenerated. It uses the composite tenant FK every table has carried since V25, and
  `ON DELETE CASCADE` from the plan.
- `match_plan_pairing_players`: pairing, player, side (1 or 2), cascading from the pairing.
- **Flagged is derived, never stored.** A pairing is flagged when its player set is not exactly the
  plan's current starters. Who left and who joined is the difference between the two sets. This is
  the rule `MatchPlan.isExpired()` and the waitlist positions already follow. It needs no hook in the
  confirmation paths, and it gets the withdraw-then-return case right without extra code.
- At most **five pairings per plan**, a constant like the guest cap.

### Endpoints (`MANAGER`)

| Endpoint | Does |
|---|---|
| `POST /api/match-plans/{id}/pairings` | Save from a preview: type, params and the two previewed lists, since the preview is what gets saved (as `/generate/confirm` already does since FootMania-Back#324). 201. 409 at the cap |
| `GET /api/match-plans/{id}/pairings` | The pairings with sides, averages, Δ, `flagged`, and who left and joined. **`MANAGER` only**, 403 for everyone else |
| `POST …/pairings/{pairingId}/regenerate` | Re-run the stored type and params on the current starters. For MANUAL there's nothing to re-run: the newcomer takes the leaver's side and the manager edits from there |
| `DELETE …/pairings/{pairingId}` | Discard one. 204 |
| `POST …/pairings/{pairingId}/match` | Create the match. 409 when flagged. The same guards as confirm apply: generatable (with the grace window below), equal sides, and every listed player still a starter, otherwise 422. Deletes every pairing of the plan in the same transaction. 201 with the `MatchDTO` |

- `MatchPlanDTO.provisionalTeams: { count, flagged }`, sent to everyone (decision 4) and omitted when
  there are none, under `non_null`.
- `POST /generate` stays a stateless preview. **`/generate/confirm` stays** until the frontend release
  that stops calling it is live, because the backend deploys first. It is removed in a later release.
- Cancelling a plan deletes its pairings. Deleting one cascades.
- The match-building half of `confirmGeneration` becomes one method, shared by the legacy confirm and
  the pairing path.

### Grace window (decision 7)

`MatchPlan.isGeneratable()` today requires kickoff to still be ahead. It becomes `CONFIRMED` and kickoff
plus a grace window still ahead, so preview, save, regenerate and create all keep working for a short
time after kickoff. **Proposed: 6 hours**, as a constant, not yet confirmed by the owner. `expired`
does not change, and neither does the poll: `acceptsAnswers()` still closes at kickoff, so the
starters can't move during the grace window. Clients branch on `generatable` and never re-derive it,
so the frontend follows without a change.

### Charges (decision 8)

`MatchFeeService.generateChargesFor` today splits the total across **every CONFIRMED confirmation**,
reserves included. Twelve confirmed means twelve people each pay a twelfth, two of them for a match
they weren't picked for. No test pinned it.

It becomes **the team sheet of the plan's match**, split by the sheet's size, on both paths (pairing
and draft). The manual `POST /api/match-plans/{id}/charges` ("for when the cost was not known then")
charges the same sheet, and refuses with 409 while no match exists, because under D there is nothing
to charge before the pick. A lineup change after the match (swap, replace) moves no charge. The owner
decided that is the organiser's to fix with the existing void and add-charge tools. Guests keep their
auto-delegation to the inviter. CHANGELOG, under Fixed: reserves are no longer charged.

### The announcement (point 5, decisions 2 and 3)

Published as a `TeamsAnnouncedEvent` from both match-creating paths (the pairing and `confirmDraft`),
and consumed **after commit** like every push in `service/push`.

- **Push:** a new `NotificationCategory.TEAMS_ANNOUNCED`, on by default and mutable. One message per
  player on the sheet with an account: "Team Nuno · Friday 5-a-side" / "You're with Nuno, Hugo, Diogo
  and Luís.", with the kickoff instant so it renders in the reader's zone, and linking to the match.
  Guests get nothing, as with every push. The text is written in English on the server, like the
  existing pushes. The draft path keeps its `DRAFT_COMPLETED` "Teams are set" at completion, which
  doesn't say which team. `TEAMS_ANNOUNCED` follows at confirm.
- **Chat:** today the match chat exists only after somebody presses to open it
  (`MatchChatService.open`). Match creation now creates it through `MatchChatProvisioner`, with the
  roster the press would compute (players on the sheet with accounts, plus the group's ORGANIZERs),
  and posts the teams. If fewer than two people on the roster have accounts, the chat is skipped and
  only the push goes out, the same rule the press applies.
- **System sender (V62):** `chat_messages.sender_user_id` is NOT NULL (V41). It becomes nullable, and
  a `kind` column (`USER` or `SYSTEM`) is added. **This is a rollback boundary, but a short one.**
  The previous release cannot map a message with no sender, but every message is deleted 24 hours
  after it is written, so a day after the last system message the previous jar reads the table again.
  The CHANGELOG says so. A system message can't be reported or replied to.
- **No double push:** a new message sends `CHAT_MESSAGE` to every participant. `ChatPushNotifier`
  must skip `SYSTEM` messages, or everyone gets two pushes for one event.
- **It won't stay long, and that's accepted:** messages live 24 hours and a quiet match chat is removed
  after 12 hours of silence. The push and the match page are the lasting copies.

### The reserve cutoff (decision 6)

Close to kickoff, a reserve shouldn't be pulled in, because they can't get ready and get there. Today
a withdrawal ten minutes before kickoff still promotes a reserve and pushes "You're in".

- **Setting:** `AppSetting.PLAN_RESERVE_CUTOFF_MINUTES` (`plan.reserve.cutoff.minutes`), per group,
  next to the guest cap. 0 turns it off and restores today's behaviour. **Proposed default: 120, range
  0–1440**, not yet confirmed by the owner. It is edited on the group settings screen that already
  edits the guest cap.
- **Holding the place:** the starting list is recomputed from the queue on every read, so after the
  cutoff a withdrawal has to keep its place or the first reserve still slides in, on paper and into
  the fees. A late withdrawal is recorded as DECLINED with `player_confirmations.withdrawn_late_at`
  set, and **keeps its `confirmed_at`**, so it counts for queue position and is excluded from teams and
  fees. It is **a nullable column and not a new status**: `status` has a CHECK constraint from V1, so a
  new value would be a migration and a rollback boundary. The column is additive (V63). The previous
  release ignores it and promotes the reserve, which is today's behaviour.
- **Leaving stays open after the cutoff**, as `isWithdrawalOpen()` argues: the app should record that
  someone can't come.
- **Releasing the place:** a manager action clears the hold, so the row becomes an ordinary
  withdrawal, the first reserve moves up and gets the normal push. It's for when a reserve says they
  can make it after all.
- **The empty place:** sides must be equal (starter selection, generation and the rating engine all
  assume it), so a held place flags every pairing, regenerate is refused for lack of starters, and the
  match can't be created until the manager releases the place or adds someone (a guest, or a player
  confirmed on their behalf). A side playing one short is out of scope.
- **Telling the manager:** a new `NotificationCategory.LATE_WITHDRAWAL` to the group's MANAGERs:
  "Hugo dropped out 40 minutes before kickoff. No reserve moved up." It earns its place because
  without it the manager finds out at the pitch.
- Drafts are untouched (#352 is separate).

## 5. Frontend

- **Team generation:** previewing works as today. "Confirm" becomes "Save as provisional teams", and the
  plan's saved pairings are listed under the plan picker. The captain-pick section is unchanged.
- **The pill** on the plan list and in the plan modal reads "Provisional teams" / "Equipas provisórias"
  / "Equipos provisionales" **for everyone** (decision 4). The reading taken: everyone sees the pill,
  and only a `MANAGER` can open the pairings. For a manager it turns amber when any pairing is flagged,
  with the flagged count in its accessible label and in the pairings view.
- **The pairings view** in the plan modal (MANAGER): each pairing's sides and `BalanceGauge`, the flag
  banner ("Tiago dropped out · Filipe is in"), and Regenerate, Discard and Create match. A flagged
  pairing can't create a match. Repair arrives with C.
- **Chat:** render `SYSTEM` messages as a system line, with no sender, report or reply.
- **Settings:** toggles for `TEAMS_ANNOUNCED` and `LATE_WITHDRAWAL`, and the cutoff on the group
  settings screen.
- **Late withdrawals:** the plan shows a held place, and the manager gets "Release the place".
- **What's new (decision 10):** two rows in `src/releases/unreleased.ts`, following
  `docs/guides/release-highlights.md`. `audience: ['MANAGER']` for provisional teams (save, regenerate,
  create from the plan) and the late-withdrawal hold. `audience: 'everyone'` for the team push and chat
  message, and the pill. The wording is written in the frontend PR.
- Every string in en, pt and es. CHANGELOG entries in both repos.

## 6. Delivery

Every PR bases on `next`. The backend release goes out before the frontend.

| # | Repo | PR |
|---|---|---|
| 1 | Back | Provisional teams: V61, pairings service and endpoints, derived flag, create match from a pairing with point 3, `provisionalTeams`, the grace window, charges from the team sheet. Contract, `FRONTEND_ENDPOINT_CHANGES.md`, CHANGELOG, `integrationTest` |
| 2 | Back | Announcement: `TeamsAnnouncedEvent` from both paths, chat created at match creation, V62 system messages, `ChatPushNotifier` skip, `TEAMS_ANNOUNCED`. Chat and push contracts |
| 3 | Back | Reserve cutoff: V63 `withdrawn_late_at`, the setting, the hold in starter selection, release-the-place, `LATE_WITHDRAWAL`. Poll-window contract |
| 4 | Front | Provisional teams screens, the pill, the What's new rows |
| 5 | Front | System chat line, the two notification toggles, the cutoff setting, the held place and release |
| — | Both, later | C: a pure swap evaluator beside `DraftBalanceEvaluator`, suggestions and apply, the Repair button |
| — | Back, later | Remove `/generate/confirm` once PR 4 is live |

Migration numbers are assigned at merge time. They are V61–V63 only if nothing else lands first.

## 7. Traps found while designing

- **The chat has no system sender** (`sender_user_id NOT NULL`), and **a new message fires
  `CHAT_MESSAGE`**. Both are covered in §4. Miss the second and every player gets two pushes.
- **The match chat doesn't exist at match creation** unless the server creates it.
- **`confirmations.status` has a CHECK constraint from V1**, which is why the late withdrawal is a
  column.
- **The manual charge endpoint shares `generateChargesFor`**, so the starters-only change covers it too.
- **A draft snapshots its pool and never re-checks it:** #352. Pairings need exactly that check, so
  don't copy the draft's `confirmDraft` as a template.
- **The live frontend still calls `/generate/confirm`.** Removing it with PR 1 would break production
  between the backend and frontend deploys.

## 8. Out of scope

C in the first release. A side playing one short. Members seeing the pairings themselves. Scheduled
automatic draws. Changing captain-pick drafts beyond announcing them.
