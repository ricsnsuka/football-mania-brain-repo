# Crowd MOTM Voting — API Contract

**Date:** 2026-07-29
**Version:** v1.0.0 (Roadmap Phase 3)
**Status:** APPROVED — backend complete (voting, configurable window, resolution job, notification, GDPR)

---

## Scope

The players who appeared in a match vote for one of their own. Two endpoints, one new table, one
migration (V14). No existing endpoint, DTO or status code changes.

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/matches/{id}/mvp-vote` | The poll's state, from the caller's point of view | Any authenticated |
| `POST` | `/api/matches/{id}/mvp-vote` | Cast or change your vote | Any authenticated |

**The voter is always the caller's linked player, taken from the JWT.** There is deliberately no
path or body parameter naming a voter — one would let anybody vote as somebody else, and the entire
value of the result rests on it being one vote per person who actually played.

---

## Design decisions

### The crowd result is not `isMvp`

`player_stats.is_mvp` is the **administrator's** pick and is untouched by everything here. The
crowd's answer lives in `matches.crowd_mvp_player_id`.

They are two different facts with two different authorities. Merging them would make it impossible
to say *"the admin picked X, the players picked Y"*, and revising one would destroy the other. The
leaderboard's `mostMvps` category still counts `is_mvp` only.

### The one-vote rule is a database constraint

`uq_match_mvp_votes_voter UNIQUE (match_id, voter_player_id)`.

Checking for an existing vote in the service and then inserting **races**: two concurrent requests
both read "no vote yet" and both insert. The database is the only place this can be decided
correctly. The service's lookup only chooses between INSERT and UPDATE; losing that race surfaces
as a constraint violation, which is returned as `409` telling the caller to retry.

### A vote can be changed while the window is open

Voting again replaces the previous choice rather than adding one — the unique constraint still
guarantees one *effective* vote per voter, so permitting a correction costs nothing and forgives a
misclick. Once the window shuts, nothing moves.

### Only players who appeared may vote

Checked against `player_stats`, the record of who actually turned out. This is not merely a
correctness rule: **it is what stops the vote being brigadable** by anyone holding an account.

Self-voting is refused with `400`, and is also a CHECK constraint in the schema so the rule survives
a bulk import or a future second code path.

### Ties produce no winner

Decided up front, as the design demanded. When two or more players are level at the top,
`crowd_mvp_player_id` stays `null` and the match is still marked resolved.

Picking between them would mean inventing a rule the voters never agreed to, and every candidate
rule is bad in a way people notice: *highest rating* systematically hands ties to the strongest
player already; *earliest vote* is invisible from the outside and reads as arbitrary. "The players
could not agree" is an honest answer that needs no defending.

> With 8–14 voters per match, **ties will be reasonably common**. The UI should present a tie as a
> normal outcome, not an error or a missing value.

### Resolution is a single claimed write, never computed on read

`MvpResolutionScheduler` runs hourly and hands each due match to `MvpResolutionDispatcher`, which
**claims before counting** via a conditional update — `SET mvp_resolved_at = :now WHERE id = :id AND
mvp_resolved_at IS NULL` — exactly the V13 reminder pattern.

Two readers counting independently could disagree, because votes can still be arriving as the
window closes, and a result already shown to somebody must not change. `mvp_resolved_at` is both the
claim and the marker, so a tie is a *final answer* rather than a poll left hanging — otherwise the
scheduler would pick it up again on every tick forever.

`MvpResolutionDispatcher` is a **separate bean** from the scheduler for the same mechanical reason
`ReminderDispatcher` is: `@Transactional` is proxy-based, so a same-bean call would drop it silently
and the `@Modifying` claim would fail outright.

### The window is anchored to completion, not `matchDate`

`matches.mvp_voting_closes_at` is set to `now + <window>` when the match is completed, and only if
it is not already set — so re-completing or amending a match never reopens a counted poll.

The window defaults to 24 hours and is **admin-configurable** (1–168). Because the closing time is
stamped onto the match at completion rather than computed on read, **a change never applies
retroactively**: polls already open keep the window they were opened with, so shortening the setting
cannot close a vote somebody is part-way through casting. Clients must read `mvpVotingClosesAt` from
the match rather than adding 24 hours to anything.

It gets its own column because neither existing timestamp works: `match_date` can be **backdated**
when recording an old match (a window derived from it could already have expired before anyone could
vote), and `updated_at` moves on any later edit.

**Matches completed before V14 have `mvp_voting_closes_at = NULL`** and are permanently closed.
Backfilling would open a poll on a match everyone has forgotten.

---

## Endpoints

### GET /api/matches/{id}/mvp-vote

**Auth:** `isAuthenticated()`. **Success:** `200`.

Readable by anyone signed in — the result is part of the match record. Only `canVote` depends on who
is asking.

```json
{
  "matchId": 42,
  "votingOpen": true,
  "votingClosesAt": "2026-07-30T16:00:49Z",
  "resolved": false,
  "crowdMvpPlayerId": null,
  "crowdMvpPlayerName": null,
  "tied": false,
  "canVote": true,
  "myVotePlayerId": 7,
  "totalVotes": 5,
  "tally": [
    { "playerId": 7, "playerName": "Ricardo Nsuka", "votes": 3 },
    { "playerId": 4, "playerName": "João Silva", "votes": 2 }
  ]
}
```

| Field | Notes |
|-------|-------|
| `votingOpen` | Completed, has a window, and the window has not passed |
| `votingClosesAt` | **Absent** when null — see the Jackson note below |
| `resolved` | The count has been done. True even when it produced no winner |
| `crowdMvpPlayerId` | Absent when unresolved **or** when tied. `resolved` + `tied` disambiguate |
| `tied` | True only when resolution found players level at the top |
| `canVote` | The caller played in this match **and** the window is open |
| `myVotePlayerId` | Absent if the caller has not voted. Drives the selected state on the ballot |
| `tally` | Most votes first, ties broken by name. **Leading is not winning** until `resolved` |

> ⚠️ **Nullable fields are omitted, not sent as `null`.** This API sets
> `spring.jackson.default-property-inclusion: non_null`, so `crowdMvpPlayerId`, `myVotePlayerId`
> and `votingClosesAt` are **absent** from the JSON when they have no value — they arrive as
> `undefined` in TypeScript, and `x === null` is false for all of them. Branch on the booleans
> (`resolved`, `tied`, `canVote`), which are always present. This exact mistake was made and
> corrected once already in `LEADERBOARDS-API-CONTRACT.md`.

---

### POST /api/matches/{id}/mvp-vote

**Auth:** `isAuthenticated()`. **Success:** `200` with the full updated summary — a ballot has to
reconcile against the server anyway, and returning the state saves a follow-up request.

```json
{ "votedForPlayerId": 7 }
```

**Idempotent in effect:** posting the same choice twice leaves one vote. Posting a different choice
replaces it.

#### Error responses

| Status | Trigger | `message` |
|--------|---------|-----------|
| `400` | `votedForPlayerId` missing | *(validation)* |
| `400` | Voting for yourself | `"You cannot vote for yourself"` |
| `400` | Candidate did not play in this match | `"That player did not appear in this match"` |
| `400` | Caller's account has no linked player | `"No player linked to your account"` |
| `403` | Caller did not play in this match | `"Only players who appeared in this match may vote"` |
| `403` | Unauthenticated | *(Spring Security — no body)* |
| `404` | No such match | *(standard)* |
| `409` | Match not completed | `"Voting opens when the match is completed"` |
| `409` | Match predates the feature | `"This match has no MOTM vote"` |
| `409` | Window has closed | `"Voting for this match has closed"` |
| `409` | Two concurrent first-votes from the same person | `"Your vote was submitted twice at once — try again"` |

Note `403` for a non-participant versus `400` for an ineligible candidate: the first is *you may
not do this*, the second is *that input is wrong*.

---

## Notifications

One new category, `MVP_VOTE_OPEN`, sent to everyone who appeared, when the match completes.

It earns its place by **asking for something** rather than reporting something — it is the only
category that needs the recipient to act, and with a window of about a day somebody who is not told simply
misses it.

> **This is a second notification at the same moment as `MATCH_COMPLETED`**, and that is a real
> cost worth naming. They stay separate because they ask different things — one reports a result,
> one requests an action — and separate categories are what let somebody mute the request while
> keeping the report. If the pair proves annoying in practice, folding the prompt into the
> completion message is a one-line change, at the price of that independent mute.

There is deliberately **no "the result is in" notification**. The outcome is on the match screen for
anyone who cares, and a second interruption per match for information nobody is waiting on is
exactly what gets a channel muted wholesale.

Adding a category is free by design: `notification_mutes` stores opt-outs, so `MVP_VOTE_OPEN` ships
enabled with no backfill.

---

## Privacy

`match_mvp_votes` holds personal data about **two** people per row — the voter and the votee — and
the export treats the directions differently on purpose.

- **Votes cast** (`mvpVotesCast`) are reported **with the player chosen**. This is the one place the
  export names another player, and it is justified: the fact being reported is the subject's own
  action, and a vote with its choice withheld would not be a copy of it. The votee's name is already
  visible to every authenticated user; what the no-other-names rule protects is other people's
  *statistics*, and none appear.
- **Votes received** (`mvpVotesReceived`) are reported as a **count per match only**. The voters are
  not named. Who someone voted for is *their* personal data, not the subject's, and a ballot the
  winner could unmask by requesting an export would not be secret at all.

**Erasure leaves votes untouched in both directions.** The table cascades from `players`, but
erasure anonymises rather than deletes, so votes survive with a now-anonymous voter — the same
reasoning that keeps `player_stats`. Deleting votes cast would retroactively change other matches'
results; deleting votes received would destroy other people's records.

All three places required by `PRIVACY_AND_DATA_PROTECTION.md` were updated: the data table, the
export and erasure paths in `PrivacyService`, and the proving cases in `PrivacyServiceTest`.

---

## Frontend Migration Notes

1. **Poll `GET` after completing a match** — the window opens at completion, not at `matchDate`.
2. **Render a tie explicitly.** `resolved: true` with no `crowdMvpPlayerId` and `tied: true` means
   *the players could not agree*, which is a normal outcome and will not be rare in a small group.
   Do not render it as "no result yet" — that is what `resolved: false` means.
3. **Do not treat the top of `tally` as the winner** while `resolved` is false. It is a running
   count and the lead can change until the window shuts.
4. **Hide the ballot when `canVote` is false**, and say why — not played, or voting closed. The
   flag already accounts for both.
5. **Show `myVotePlayerId` as selected**, and allow re-submitting: changing a vote is supported
   while the window is open.
6. **Remember the omitted-null rule** in the warning above. Branch on booleans.

---

## Breaking Changes

- [x] **None for existing endpoints.** Two new endpoints, one new table, three new nullable columns
  on `matches`.

Two additive changes to be aware of:

- `PersonalDataExportDTO` gained `mvpVotesCast` and `mvpVotesReceived`. Both are always present,
  empty rather than absent.
- `NotificationCategory` gained `MVP_VOTE_OPEN`, so `GET /api/push/preferences` returns one more
  entry in `categories`. A frontend rendering from that list — as `PUSH-API-CONTRACT.md` instructs —
  needs no change.
