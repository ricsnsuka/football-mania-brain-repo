# The 2026-09 batched backlog

**Opened 2026-09-02 against `f1b25a6`. Closed 2026-09-03 with 3.3.0.** Six bugs and two features,
batched into one release rather than a version per fix.

## The artifacts

Rendered, shareable versions. They are the presentation; this file is the record, and where the two
disagree this file is newer.

| | |
|---|---|
| [Bug backlog + feature shortlist](https://claude.ai/code/artifact/85d243aa-db3c-433c-8a4b-e96f1b527ee6) | The whole batch — all six bugs and four features, with the corrections written in place |
| [FEAT-1 · Date polling spec](https://claude.ai/code/artifact/3a046dc4-8384-4ef6-829b-b531eb5f270a) | **Not built.** Decide *when* to play, in the app |
| [FEAT-5 · Rating history chart spec](https://claude.ai/code/artifact/d7e37060-db6d-4870-ba5e-72f3a950a2e6) | **Not built.** Skill rating over matches, on the profile card |
| [FEAT-6 · Match chat spec](https://claude.ai/code/artifact/93e823b5-78b5-4305-ac76-bd751ad72880) | **Not built.** A chat opened from a match |

> ⚠️ **FEAT-6's spec calls its migration `V46`. That number is taken** — `V46` is session
> `token_version`, shipped in 3.3.0. Renumber to `V49` or later when it is built.

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

## Still open after 3.3.0

- **Plans drafted before BUG-1 are not backfilled** — still `CONFIRMED`, still unbilled.
  `POST /api/match-plans/{id}/charges` is how they get their charges; their *status* needs a call.
- **A revoked session surfaces as `403`, and the web app only redirects on `401`** — so it shows an
  error instead of bouncing to login. Pre-existing for expired tokens; revocation makes it
  reachable more often. Not fixable by treating `403` as logout: `403` also means "you lack that
  role", and signing people out of pages they merely cannot access is worse. Needs the two cases
  distinguished on the API side.
- **3.3.0 shipped without a browser check.** No preview environment; three user-facing surfaces
  changed — the logout button, the cost panel on a kicked-off plan, and the dashboard card.
- **FEAT-1, FEAT-2, FEAT-5, FEAT-6** are unbuilt. FEAT-2 (three teams / winner-stays-on) remains
  the largest job on the page and touches generation, `Match`/`MatchTeam`, scoring, stats and the
  rating engine.
