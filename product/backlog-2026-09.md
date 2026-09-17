# The 2026-09 batched backlog

**Opened 2026-09-02 against `f1b25a6`. Closed 2026-09-03 with 3.3.0** — six bugs and two features,
batched into one release rather than a version per fix — **and reopened the same day for 3.4.0**,
which shipped two more of its features.

## The artifacts

Rendered, shareable versions. They are the presentation; this file is the record, and where the two
disagree this file is newer.

| | |
|---|---|
| [Bug backlog + feature shortlist](https://claude.ai/code/artifact/85d243aa-db3c-433c-8a4b-e96f1b527ee6) | The whole batch — all six bugs and four features, with the corrections written in place |
| [FEAT-1 · Date polling spec](https://claude.ai/code/artifact/3a046dc4-8384-4ef6-829b-b531eb5f270a) | **Not built.** Decide *when* to play, in the app |
| [FEAT-5 · Rating history chart spec](https://claude.ai/code/artifact/d7e37060-db6d-4870-ba5e-72f3a950a2e6) | **Shipped in 3.4.0**, 2026-09-03 — Back#265, Front#136; **dates and ordering fixed in 3.4.1** (Back#269). Skill rating over matches, on the profile card |
| [FEAT-6 · Match chat spec](https://claude.ai/code/artifact/93e823b5-78b5-4305-ac76-bd751ad72880) | **Shipped in 3.4.0**, 2026-09-03 — Back#266, Front#137. A chat opened from a match |

> ⚠️ **FEAT-6's spec calls its migration `V46`. That number was taken** — `V46` is session
> `token_version`, shipped in 3.3.0. It was built as **`V49`**.

## What shipped in 3.3.0

| | | |
|---|---|---|
| BUG-1 | Drafted matches never retired their plan — so they **never billed anybody** | Back#258 |
| BUG-2 | Cost controls vanished the moment kickoff passed, which is when the cost is known | Front#129 |
| BUG-3 | `FULL_TIME` stamped a time and left the match open | Back#261 |
| BUG-5 | Session tokens could not be revoked; no logout endpoint existed | Back#257, Front#128 |
| FEAT-3 | A promoted reserve was never told | Back#260, Front#133 |
| FEAT-4 | Nothing ever chased anybody for money | Back#259, Front#132 |

Plus two that were **never broken**, and one that was found on the way:

- **BUG-4** (roster leaking contact details) — already fixed in `5bf515b`. The audit read
  `PlayerDTO` and the `@PreAuthorize`, and never opened `PlayerPiiPolicy`, which sits one line below
  the annotation it read.
- **BUG-6** (no security headers on the app origin) — already fixed in `c9c7876`. The audit read
  `netlify.toml`, correctly found no `[[headers]]`, and did not open `next.config.ts` — which the
  finding itself named as the alternative place to look.
- **`tomcat-embed-core` on three CRITICAL advisories**, newly published mid-release. Not caused by
  the release; live in production on 3.1.0 the whole time. Raised to 10.1.59 in Back#263.

## What this batch got wrong about itself

Worth more than the fixes. Two entries were wrong about their own subject, in opposite directions:

**BUG-3 overstated the cost.** It said `completeMatch` needs an absolute scoresheet the clock does
not carry, and that deriving one from goal events was the real design decision. Nothing needed
deriving: `recordGoalEvent` already writes into the `PlayerStat` rows and the running-score
recompute already keeps the scoreline current, so the scoresheet exists before the whistle blows.

**FEAT-3 understated it.** Called "the cheapest thing on the list" — one notification, one call
site. It was silently blocked: `assertPollOpen` required `PENDING`, so the poll closed when a plan
was *confirmed*, and the late dropout the whole feature exists for could not be recorded at all. The
waitlist machinery was correct and unreachable. Fixing it meant changing the poll model.

**The lesson, alongside the "check every place a missing thing could live" one:** a claim that
something *needs a design decision* deserves the same scrutiny as a claim that something is
missing. Both were assertions about code nobody had read closely, and both cost more to unlearn
than to fix.

## Decisions taken, so they are not silently re-litigated

- **FEAT-4 reverses a written decision.** `FEE_CHARGED`'s javadoc said "chasing is the organiser's
  job". The argument for reversing it is in the new `FEE_REMINDER` javadoc and the push contract,
  not left implicit. It ships **off by default** — the only such category.
- **FEAT-3 reframes `CONFIRMED`.** It means *this match is happening*, not *the squad is settled*.
  The squad settles at team generation. Past the deadline the window narrows to withdraw-only.
- **BUG-3 completes an empty scoresheet 0–0.** Owner's decision, against the recommendation. A
  recorder who never used the app completes on a fiction and moves everyone's rating, silently.
  Recoverable via amend + `POST /api/matches/{id}/recalculate` — manually, and nothing announces it.
- **Releasing without a browser check is deliberate, and these documents should stop counting it.**
  Owner's decision, 2026-09-03: the group has one user, who reads the app on a phone and is content
  to find things in production. There is no preview environment and none is planned for this, so
  "went out dark" describes the normal case rather than a hazard — 3.3.0 and both 3.4.0 features
  were flagged that way three times over, which is three warnings about a choice already made.
  **What still belongs in the record is what a look actually found**, as the FEAT-5 section does:
  the bug is history worth keeping, the missing preview environment is not a finding.

## FEAT-5, built 2026-09-03 — what the spec did not know

Backend [Back#265](https://github.com/ricsnsuka/FootMania-Back/pull/265), frontend
[Front#136](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/136), both merged into `next`
on 2026-09-03, backend first — `095d553` and `eab6f90` — and **shipped in 3.4.0** the same night
(see [STATUS.md](../STATUS.md) for what was and was not confirmed). No migration, as
the spec predicted. Its four open decisions all went the way it recommended: hand-drawn SVG (D1),
career by default with `?seasonId=` available (D2), `isAuthenticated()` (D3).

Three things the spec got wrong or did not reach, all found by building it:

- **"Evict it wherever `MATCHES` is" is not sufficient.** `SeasonService.finalise()` writes a
  transition row per player and touches no match, so it evicts `RANKINGS`/`LEADERBOARDS`/
  `PLAYERS`/`PLAYER_PROFILE` and never `MATCHES`. Following the rule to the letter would have
  left the chart missing the one point that explains the largest movement of the year, for up to
  ten minutes. The eviction is now explicit there. **The general lesson matches the one 3.3.0
  already learned:** a rule of the form "do X wherever Y is" is only as good as whether Y actually
  covers every writer, and nobody had checked.

- **`role="img"` on the chart would have made it unreadable to a screen reader.** `img` is a
  leaf role: it removes every descendant from the accessibility tree, including the focusable
  points the spec's §09 requires to announce their date, result, goals, assists and movement. The
  acceptance test caught it. It ships as `role="group"`.

- **D4 was not implementable as written.** It recommended the tooltip name "result and scoreline",
  but the DTO in §04 carries no scoreline field, and §09 requires the response carry no field the
  chart does not draw. Built to §04 — result, no scoreline. Adding one is a new field and a join
  to `Match`, on a read the same decision argues should stay cheap. **Open**, if anybody wants it.

Also worth recording: the ordering needed an `id` tie-break that the spec did not mention. A bulk
recalculation writes a whole career's rows inside the same instant, so `created_at` alone does not
determine the order the line is drawn in. `findAllByPlayerAndSeason` gained the same tie-break —
its javadoc already promised the first entry is the season's start rating.

⚠️ **Neither half was opened in a browser.** No preview environment; the animation, the tooltip
placement and the dark-theme colours want a local look before release — the same gap 3.3.0 shipped
with, and the reason it is written down here rather than left implied.

### Found on the first real look, 2026-09-03 — fixed in 3.4.1 (Back#269)

**The first time anybody opened the chart was in production, and it was wrong twice.** Every point
in a career was dated the same day, and segments were drawn climbing where the tooltip said the
rating fell. One cause under both.

`skill_rating_history.created_at` is when a **row was written**. A recalculation deletes a match's
rows and inserts fresh ones, so a movement from a match played months ago comes back stamped with
today — and a bulk recalculation replays a whole career in one pass, giving every row in it one
instant. That is the dates. A **season transition is the one row nothing rewrites**, so it keeps
its original timestamp and sorted in front of every recalculated match, however long ago that
season ended. That breaks the chain the line is drawn through — a segment is positioned by the
previous row's `ratingAfter` and coloured by its own `change`, and those agree only in application
order. That is the colours. The chart opened on two season ends and then contradicted itself.

Now dated and ordered by `COALESCE(match.matchDate, history.createdAt)`, tie-broken by `id`, in
one place (`SkillRatingHistory.occurredAt()`) and for every read of that table that means "career
order" — including `CalculationService`, which took the season's first entry as where a player
stood when the season began and could get another night's rating instead.

⚠️ **The `id` tie-break was added for exactly the right reason and fixed the wrong half.** The
contract stated the hazard correctly — *"a bulk recalculation writes a whole career's rows inside
the same instant, and they are drawn in the order they happened only if the id decides it"* — and
breaking a tie only makes the order deterministic **among rows that share an instant**. It says
nothing about the comparison between rows that do not, which is precisely the transitions.

**That is this feature's second bug of one shape**, after "evict it wherever `MATCHES` is" missed
`SeasonService.finalise()`. Both were rules that were right about the thing they named and silent
about whether it covered the cases — and both were written down as *decisions*, which is what made
them look settled. The eviction one was caught while building. This one needed somebody to look at
the screen.

## FEAT-6, built 2026-09-03 — what the spec did not know

Backend [Back#266](https://github.com/ricsnsuka/FootMania-Back/pull/266), frontend
[Front#137](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/137), both merged into `next`
on 2026-09-03, backend first — `1552c31` and `a309003` — and **shipped in 3.4.0** the same night
(see [STATUS.md](../STATUS.md) for what was and was not confirmed).
One migration (`V49`, not the spec's `V46`), one column, one endpoint — and one
more endpoint the spec recommended alongside it. All four open decisions went the way it
recommended: any match with a team sheet gets the button (D1), organisers are seeded and
`DELETE /api/chat/conversations/{id}/me` ships in the same change so they can leave (D2),
`ORGANIZER` only, not `GROUP_ADMIN` (D3), and only the match's players and the organisers may
press it — a bystander is a `403`, not a `404`, because the match was never a secret (D4).

Two things the spec did not reach, both found by building it:

- **The race has a second half the spec's `everyoneChannel` analogy does not have.** The
  group-wide channel is one row; a match chat is one row *and its roster*. Committing the
  conversation in its own transaction and seeding the participants afterwards — the obvious
  spelling — leaves a window in which the loser re-reads a committed chat with no participants,
  joins it with one row for themselves, and then the winner's seed arrives carrying that same
  person: `uq_chat_participants` refuses the winner's insert and takes the *winner's* request
  down. `MatchChatProvisioner` therefore commits the roster together with the conversation, so
  the loser only ever sees a chat that is already fully seeded. The lesson is the same shape as
  FEAT-5's: an existing pattern is only as reusable as the invariant it protects, and this one
  protected one row, not two tables.

- **The provisioner's entity cannot be handed back across the transaction boundary.** It carries
  a lazy proxy for the match, bound to a persistence context that has already closed; describing
  it would throw. Ids in, id out, and the caller re-reads in its own transaction. Not visible in a
  unit test with mocked repositories, which is one more reason `MatchChatIT` exists.

Also worth recording:

- **`MembershipRepository.findActiveUserIdsWithRole` was not "genuinely new".** FEAT-4 had already
  added `findActiveUserIdsHolding(tenantId, roles)` for the fee-reminder digest. Reused.
- **The frontend already avoids `useSearchParams`** (the login form reads `window.location`
  inside a handler to stay out of a Suspense boundary). `/chat?conversation=9` follows that
  precedent: read once in the state initialiser, and never rendered directly, so the server's
  `null` and the client's id cannot disagree on screen.
- **The button is gated client-side on the same two facts the server checks** — a linked player
  on the sheet, or `ORGANIZER` — so it never `403`s on use. A member with no linked player is not
  "on the sheet", whatever their name, and sees no button.
- **`playersWithoutAccount` counts suspended members too.** The spec's worked example (14 → 11
  linked → 10 active → "4 can't be added") already did; the field's name under-describes it, and
  the DTO javadoc says so.

⚠️ **Neither half was opened in a browser.** No preview environment — the third feature in a row
shipped this way. ⚠️ **The integration tests were not run locally either**: Docker is not available
in the session that built this, so `MatchChatIT` (nine cases, including the race and the retention
clock) and the two new `ChatSchemaIT` cases are proved by CI's Testcontainers job alone — which
**passed on Back#266** ([run 33705595030](https://github.com/ricsnsuka/FootMania-Back/actions/runs/33705595030)),
as did every other check on both PRs.

## Still open after 3.3.0

- **Plans drafted before BUG-1 are not backfilled** — still `CONFIRMED`, still unbilled.
  `POST /api/match-plans/{id}/charges` is how they get their charges; their *status* needs a call.
- **A revoked session surfaces as `403`, and the web app only redirects on `401`** — so it shows an
  error instead of bouncing to login. Pre-existing for expired tokens; revocation makes it
  reachable more often. Not fixable by treating `403` as logout: `403` also means "you lack that
  role", and signing people out of pages they merely cannot access is worse. Needs the two cases
  distinguished on the API side.
- **Shipping without a browser check is settled, and is no longer listed here as a gap** — see
  the decision below. What is still open is narrower and worth keeping: **no test caught the
  rating-chart bug because every test that touched the ordering supplied the list already
  ordered.** A mock returns what it was handed, in the order it was handed; the `ORDER BY` was the
  thing under test and the only tier that can hold an opinion about it is the one with a database.
  `RatingHistoryOrderIT` is that test now.
- **FEAT-1, FEAT-2** are unbuilt. **FEAT-5 and FEAT-6 shipped in 3.4.0** — see above. FEAT-2 (three teams / winner-stays-on) remains
  the largest job on the page and touches generation, `Match`/`MatchTeam`, scoring, stats and the
  rating engine.
