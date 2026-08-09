# Retire the organiser's MVP pick — Technical Specification

**Date:** 2026-08-08
**Status:** 📋 **SPECIFIED, not built.** Written the day the counting change shipped, because that
change is what created the problem this one solves.
**Priority:** MEDIUM — nothing is broken, but the match sheet now shows a control that lies
**Estimated Effort:** Rung 1 — S (≈½ day backend, ≈½ day frontend). Rung 2 — M, and gated on a
decision about historical data that nobody has taken yet
**Depends on:** the MVP counting change — **written 2026-08-08, not yet released**; see §1
**Contract:** `docs/api/MATCH-LINEUP-API-CONTRACT.md` and `MOTM-API-CONTRACT.md` in the backend
repo, updated in the same commit as the code, per repo convention

---

## 1. Why this exists

On 2026-08-08 the owner settled a question the codebase had answered the other way for months:

> **Man of the Match is decided by the crowd, not by an algorithm or an admin decision.**

The counting change that followed is **written but not yet released** — backend only, awaiting a
commit, a merge to `next`, and a release. Everything below assumes it lands; if it does not, none of
this applies. What it does:

- `mostMvps` and `FIRST_MVP` count `matches.crowd_mvp_player_id` — the vote.
- `player_stats.is_mvp` counts **only on matches that never ran a poll**: everything completed
  before crowd voting shipped, which has no crowd answer and never will. Without that fallback the
  board would have lost years of history on the day the rule changed. The owner chose the fallback
  explicitly.
- Where a poll ran, a tie counts for nobody and the organiser's pick does not stand in for the
  result the group failed to reach.

That leaves the organiser's MVP toggle on the scoresheet **feeding no total on any match played
since crowd voting shipped**, which is every match anybody is currently recording. It still renders
a star next to a player's name. A control that looks like it names the Man of the Match and does
not is worse than no control: the next person to find it will report it as a bug, and they will be
right about the symptom and wrong about the cause.

**This document is the follow-up the owner deferred on 2026-08-08**, when asked whether to retire
the toggle as part of the counting fix. The answer was: fix the counts now, write the retirement up
separately, because it touches the match stats API, the record form and the scoresheet, and it is
too big to bolt onto a bug fix.

---

## 2. The trap, stated first

`player_stats.is_mvp` is **the only record of who was Man of the Match for every pre-voting
match**, and the leaderboard now depends on it for exactly those matches.

Dropping the column deletes that history — silently, because the board would simply get shorter and
no error would be raised. Any plan that starts with the migration has the order backwards.

This is why the work below is two rungs with a decision between them, rather than one change.

---

## 3. Scope

| In | Out |
|----|-----|
| Removing the toggle from the record form and the star from the scoresheet | Changing anything about how the crowd vote works |
| Removing the write path: `PlayerStatUpdateDTO.isMvp` and the rules that police it | The `mostMvps` / `FIRST_MVP` counting rule — shipped, and settled |
| Deciding what happens to pre-voting MVP history (§5) | Backfilling votes for matches nobody voted in |
| Optionally, dropping the column once §5 is settled | The season awards — `CROWD_FAVOURITE` never read `is_mvp` and is unaffected |

---

## 4. Rung 1 — retire the control, keep the column

**This is the whole of the visible problem, and it needs no migration and loses no data.**

The column stops being writable and becomes what it already is in practice: a frozen record of what
organisers marked before the group had a vote. The leaderboard's pre-vote half keeps working
untouched.

### Backend

| Change | File | Note |
|---|---|---|
| Drop `isMvp` from the stat update DTO | `dto/PlayerStatUpdateDTO.java` | A field that is accepted and ignored is worse than one that is refused |
| Remove the "at most one MVP across both teams" validation | `service/MatchService.java` (~line 469) | It exists only to keep the write path consistent |
| Remove the clear-every-other-stat pass when `isMvp=true` is set | `service/MatchService.java` (~line 639) | Same |
| Stop applying it in the stat patch | `service/MatchService.java` (~line 1118) | |
| Keep `PlayerStatDTO.isMvp` on **reads** | `dto/PlayerStatDTO.java` | Pre-voting matches still have one, and a client showing an old match should still be able to say so |

Reads keeping the field is the deliberate half of this rung: the fact is still true of those
matches, and the frontend needs it to render them honestly (§4, frontend).

### Frontend

| Change | File |
|---|---|
| Remove the MVP toggle button | `features/matches/matchModal/RecordMatchForm.tsx` (~line 276) |
| Remove `isMvp` from the submit payload and the local form state | `RecordMatchForm.tsx`, `matchModal/utils.ts` |
| Stop sending it in the scoresheet's diff | `matchModal/MatchScoresheet.tsx` (~line 123) |
| Show the star **only on matches with no crowd poll**, relabelled | `MatchScoresheet.tsx` (~line 269) |

That last row is the one worth thinking about. On a pre-voting match the star is a real historical
fact and should stay, but it should not say "MVP" while the MOTM panel next to it says something
else for newer matches. Label it *organiser's pick*, in all three locales, and gate it on the
absence of a poll — the match DTO already carries enough to tell (`mvpVotingClosesAt` is null
exactly when no poll ran).

### Consequence to accept

Two matches side by side will show different things: an old one with a star, a new one with a
crowd result. That is correct. They *are* different — one was decided by a person, the other by
the group — and pretending otherwise is what got us here.

---

## 5. The decision between the rungs

**Question for the owner: what happens to pre-voting MVP history?**

| Option | What it means | Cost |
|---|---|---|
| **A. Keep the column forever** *(recommended)* | Stop at rung 1. `is_mvp` stays as read-only history and keeps feeding the pre-vote half of the leaderboard | None. One dormant column, documented as frozen |
| **B. Backfill, then drop** | Copy each pre-voting match's `is_mvp` winner into `crowd_mvp_player_id`, then drop the column | The board keeps its history, but the data now claims the group voted for people it never voted for. Needs a marker column to stay honest, which is a new column to avoid an old one |
| **C. Accept the loss, then drop** | Drop the column and the fallback query with it | Every MVP won before crowd voting disappears from the board and from `FIRST_MVP`. Irreversible once the column is gone |

A is recommended because the cost of B and C is paid by the historical record and the benefit is
tidiness. The column is not in anybody's way: it has no index, no constraint, and one query reads
it. **Do not take B or C without the owner saying so in writing** — C is unrecoverable after the
migration.

---

## 6. Rung 2 — drop the column (only after §5 lands on B or C)

Gated. Listed so the cost is visible when the decision is taken, not discovered afterwards.

- Migration: drop `player_stats.is_mvp`.
- `PlayerStatRepository.countAdminMvpAwardsWithoutAPoll` and `countAdminMvpsWithoutAPoll` are
  deleted; `LeaderboardService.mvpCounts` collapses back to a single query, and its merge, its
  sort, and the whole "two disjoint sets of matches" explanation go with it.
- `BadgeService` stops adding two numbers.
- `MvpCountIT` loses the fallback cases and keeps the crowd ones.
- **The privacy export changes shape.** `PersonalDataExportDTO` carries `mvp` per match stat
  (`service/PrivacyService.java` ~line 451). Removing a field from a GDPR export is a contract
  change and needs `PRIVACY-API-CONTRACT.md` updated in the same commit.
- `MATCH_FEATURE.md` in this repo documents the column and the DTO field in four places.

---

## 7. The middle option, if rung 1 is too much

Keep the toggle and stop calling it MVP: relabel it *manager's pick*, show it beside the crowd
result rather than competing with it, and leave the write path alone. Frontend-only, an afternoon,
and it removes the lie without removing the feature.

Worth taking if organisers turn out to actually use the toggle for something — several groups mark
their own pick before the vote closes, and finding that out is cheaper than guessing. Rung 1 can
follow later; the two are not exclusive.

---

## 8. What must not be lost when this is built

- **The fallback is a deliberate owner decision, not an accident.** Anybody reading
  `countAdminMvpsWithoutAPoll` and thinking "why is this still here" should land on §2 before
  deleting it.
- **A tie counts for nobody, and does not fall back.** The organiser's pick standing in for a tied
  vote would quietly reintroduce exactly the authority this change removed.
- **The board moves a day after the match**, when the window closes, not at completion —
  `MvpResolutionScheduler` clears the leaderboard cache and re-runs the badge sweep at that point.
  A future change that resolves polls elsewhere has to carry both.

---

## 9. Related

- `docs/api/LEADERBOARDS-API-CONTRACT.md`, `MOTM-API-CONTRACT.md`, `BADGES-API-CONTRACT.md`
  (backend repo) — all three record the counting rule as it now stands.
- [PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM.md](PHASE3_HANDOFF_LEADERBOARDS_AND_MOTM.md) §"Keep the
  crowd result separate from `isMvp`" — the original decision, now superseded on the question of
  *counting* and still standing on the question of *storage*. The two columns remain two columns.
- [frontend/features/motm-voting.md](../../frontend/features/motm-voting.md),
  [frontend/features/badges.md](../../frontend/features/badges.md).
