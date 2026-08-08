# Crowd MOTM Voting

Players who appeared in a match vote for one of their own, for 24 hours after it completes. Lives
as a panel inside the completed-match modal (`MotmVotePanel`), not on its own route.

## Not the same thing as the MVP star — and now the one that counts

> **Written 2026-08-08, not yet released.** The counting change described here is backend-only and
> is still awaiting a release. Until it ships, "Most MVPs" counts the organiser's pick as it always
> did. Nothing in this file needs a frontend change either way.

`player_stats.isMvp` is the **organiser's** pick and is untouched by any of this. The crowd's
answer is `matches.crowd_mvp_player_id`, delivered through `GET /api/matches/{id}/mvp-vote`.

Two facts, two authorities, two columns — still. Merging the *storage* would make *"the organiser
picked X, the players picked Y"* unsayable, and revising one would destroy the other.

**What changed is which one is counted.** Man of the Match is decided by the players who played it,
so the leaderboards' "Most MVPs" card and the `FIRST_MVP` badge read `crowd_mvp_player_id`.
`isMvp` counts only on matches that never ran a poll — everything completed before crowd voting
shipped, which has no crowd answer and never will.

Two consequences that are visible on screen:

- **The MVP card does not move when a match completes.** It moves when the voting window closes, up
  to a day later, on the hourly resolver. A freshly completed match adding nothing is correct —
  nobody has been voted Man of the Match yet.
- **A tie adds nothing, ever**, and does not fall back to the organiser's pick. "The players could
  not agree" is the answer, and that match has no MVP.

The organiser's toggle is still on the record form and now feeds no total. Retiring it is specified
in [RETIRE-ORGANISER-MVP-PICK-PLAN.md](../../backend/plans/RETIRE-ORGANISER-MVP-PICK-PLAN.md).

## Three states that look alike and are not

| State | Response | Renders as |
|-------|----------|-----------|
| Open | `votingOpen: true` | The ballot |
| **Closed, not yet counted** | `votingOpen: false`, `resolved: false` | "Voting has closed" |
| Decided | `resolved: true`, a winner | 🏆 *X was voted Man of the Match* |
| **Tied** | `resolved: true`, `tied: true`, no winner | "The vote was tied — no Man of the Match this time" |

The middle one is real and easy to miss: the resolver runs **hourly**, so there is a genuine window
where voting has shut and nothing has been counted. Rendering that as "nobody voted" would be a lie.

**A tie is an outcome, not a missing value.** The backend marks a tied match resolved with no
winner, deliberately — every tie-break rule is bad in a way people notice, and "the players could
not agree" needs no defending. With a dozen voters this will not be rare, so it is styled amber
(a statement) rather than red (an error).

## The omitted-null trap

`crowdMvpPlayerId`, `crowdMvpPlayerName`, `myVotePlayerId` and `votingClosesAt` are **omitted**
when unset, not `null`. Every branch in the panel keys off the booleans — `resolved`, `tied`,
`canVote`, `votingOpen` — which are always present.

`src/tests/motm/mvpVoteSchema.test.ts` pins this with a payload whose nullable keys are simply
absent, which is exactly what the wire carries.

## The ballot

- **Only participants may vote**, and the API enforces it (403). This is anti-brigading, not just
  correctness.
- **The caller is left off their own ballot.** Self-votes are rejected with a 400 and a CHECK
  constraint, so offering the option would be a control that always fails. The caller's player is
  derived from `linkedUserId`, the same way the dashboards do it.
- **Voting again replaces the previous choice** while the window is open. The unique constraint
  guarantees one *effective* vote per voter, so a misclick is recoverable.
- `canVote` is false for a spectator, an admin who did not play, or an account with no linked
  player — and the panel says which, because a missing ballot with no explanation reads as broken.

## Notifications

One category, `MVP_VOTE_OPEN`, sent to participants when the match completes.

> This is a **second notification at the same instant as `MATCH_COMPLETED`**, which is a real cost.
> They stay separate because they ask different things — one reports a result, one requests an
> action within 24 hours — and separate categories are what let somebody mute the request while
> keeping the report. If the pair proves annoying, folding the prompt into the completion message
> is a one-line change, at the price of that independent mute.

There is deliberately no "the result is in" notification.

## File map

| Path | Role |
|------|------|
| `src/features/matches/matchModal/MotmVotePanel.tsx` | The panel |
| `src/hooks/mvp/useMvpVote.ts` | `useMvpVote`, `useCastMvpVote` |
| `src/services/mvpVoteService.ts` | `fetchMvpVote`, `castMvpVote` |
| `src/types/mvpVote.ts` | Types + zod schemas |

`MatchModal.test.tsx` **stubs** this panel rather than mocking its hooks — that file tests the
modal's layout, and a fetching child would otherwise need a `QueryClientProvider` for a component
it is not testing.

## i18n keys (`motm` namespace)

`title`, `prompt`, `changeHint`, `closes`, `winner`, `tied`, `noVotes`, `notAParticipant`,
`closedNoResult`, `voteFailed`, `totalVotes`.

Backend contract: `docs/api/MOTM-API-CONTRACT.md`.
