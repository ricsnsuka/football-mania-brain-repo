# Project Status

**Snapshot: 2026-08-07.** Update it when the answers change, not on a schedule — a status page
nobody trusts is worse than none.

| | Backend | Frontend |
|---|---|---|
| Branch | `main` (production), `next` (integration) | `main` (production), `next` (integration) |
| `next` head | `1b29ded` — content-identical to `main`; `next..main` lists the release **merge commit** and no changed file | `f700d07` — the same |
| Release | **`1.6.0`** — `build.gradle` | **`1.6.0`** — `package.json` |
| Running in production | **`1.6.0` at `69e4301`**, read back from the platform on 2026-08-07: Heroku **v61** `Deploy 69e43018`, and `/api/version` reporting `1.6.0` | **`3d577a7`, which calls itself `1.6.0` but is ten commits past the `v1.6.0` tag** — see below. Netlify deploy `6a762a22e8abd4000849509f`, `ready`, published 18:56:26Z, and the rule the release adds was **read back out of the deployed stylesheet** |
| `main` head | `69e4301` — 1.6.0 | `3d577a7` — 1.6.0 plus the spacing work and the own-row highlight |
| Working tree | clean | clean |
| Tests | 1104+ unit plus 28 new on `OPTIMAL`, SpotBugs clean on `spotbugsMain` | **732 unit, all passing** — green on Windows for the first time; 38/38 visual |
| Latest migration | **`V36__player_positions_and_keeper.sql`** — on `main` and applied | — |
| Deployed through | **`V36`**, applied on the 1.6.0 deploy | — |
| Tags | `v1.0.0` → **`v1.6.0`**, contiguous, and the tag is what is deployed | `v1.1.0` → **`v1.4.2`**, then **`v1.6.0`** — `1.5.x` deliberately skipped, and **the tag is now behind production** |

⚠️ **The frontend's `v1.6.0` tag no longer matches what is deployed.** A spacing pass shipped on
2026-08-07 at 16:40 **without a version bump, by owner decision** — `package.json` still reads
`1.6.0`, so production reports a version whose tag points at a different commit.

| | |
|---|---|
| `v1.6.0` tags | `f3ef62a` |
| Production runs | `3d577a7` |

**The tag was deliberately not moved.** A tag that changes what it points at is worse than one that
is merely behind: every clone that already fetched it keeps the old target silently, and the whole
value of step 6 is that a tag is an immutable claim. So **the deployed commit is recorded here
instead, and this row is the only place it exists.**

⚠️ **Two releases have now gone out under the same number, so the gap is no longer a curiosity.**
`v1.6.0` → production is ten commits and two deploys. **Cut `1.6.1` and tag it before the next
frontend release**, rather than after: the cost of the shortcut is that "what is in production" has
exactly one source, this file, and a status document is the thing step 6 exists to not rely on.

What shipped under `1.6.0` after the tag, in order:

1. **The spacing pass** (16:40) — 44px touch targets on 22 controls that were under the guide's
   stated minimum, one modal inset (`px-6`) instead of four, and 20px on both sides of every rule in
   every modal.
2. **The own-row highlight** (18:56) — the signed-in player's row marked in the league table, the
   scoresheet and the squads; regenerated visual baselines; and a green test suite.

Neither changed an API call, a dependency, or a user-visible string apart from one new `common.you`
label in all three locales.

✅ **1.6.0 shipped on 2026-08-07, both halves, backend first.** Production runs it and both platforms
were asked rather than assumed:

| | Deployed commit | Platform says |
|---|---|---|
| Backend | `69e4301` | Heroku release **v61**, `Deploy 69e43018`, 14:21:51 +0100 — and `/api/version` returns `{"version":"1.6.0","buildTime":"2026-08-07T13:21:32Z"}` |
| Frontend | `f3ef62a` | Netlify deploy **`6a75ddc0e859c70008082bc9`**, `ready`, `context: production`, published 13:30:33Z, 54s build |

Both tags are annotated with that evidence, so the claim each one makes about production can be
checked without this file. **Heroku's auto-deploy took about ten minutes to create the build, not
seconds** — long enough that it looked like it had not fired at all, and the builds API showed no
record for the commit while it was pending. Worth knowing before anybody reaches for
`git push heroku main:master` on the assumption the webhook is broken.

**The two repos are one version again.** They were two minors apart because `1.5.0` and `1.5.1` were
backend-only, a security pass and a correction to it, and neither moved an API surface. 1.6.0 spans
both halves, so the frontend went `1.4.2` → `1.6.0` and skipped `1.5.x` rather than carrying the
offset forward. Not `1.5.0`: a frontend `1.5.0` released now would not be the `1.5.0` the backend
shipped last week, and two different things under one number is worse than a number never used.

### What 1.6.0 carries

✅ **`V36` — preferred positions and goalkeeper willingness on a player.** **The migration has now
run against production**, on this deploy. It had already run against a real PostgreSQL in CI
(`🧪 Integration (Testcontainers)`), which is the `ddl-auto: validate` check that the migration and
the entity agree. New table, no backfill, nothing dropped — so unlike the V22/V29 contract steps,
redeploying the previous jar remains a viable rollback.

✅ **`OPTIMAL` team generation** — exhaustive partition search scored on
`|Δmean| + λ·|Δspread| + κ·keeperPenalty`, the seventh strategy and the first to consume V36's data.

✅ **Two defects that had to be fixed first**, each in its own commit. The `params[…]` binding
defect: every algorithm parameter the API reference documents had been silently discarded since it
was introduced, so `formWindow` and the `CAPTAIN_PICK` captain overrides work **for the first time**
in this release. And a tie-order determinism fix across the four strategies that claimed to be
deterministic and were not. See
[OPTIMAL-PARTITION-PLAN](backend/plans/OPTIMAL-PARTITION-PLAN.md), whose §7 records all three.

> The branch name says "match predictor" and the work is nothing of the sort. A match predictor was
> discussed in the same session and **declined** — nice to have, too much complexity for the value.
> The branch was already named by then. Read the pull requests, not the branch.

## 1.4.0, on 2026-08-05

**Backend first, then the frontend**, because the frontend calls two endpoints that ship in the
backend's half — the squads tab's swap and replace. Heroku took `9fa0134` as release **v56** before
`main` moved on the frontend at all.

What it carries:

- **The last-administrator guard.** `PATCH /api/users/{id}/role` refuses a write that leaves the
  group with no `GROUP_ADMIN`. This is the fix for a lockout that happened in production earlier the
  same day, and which no API call could undo — every route back is behind the grant that had just
  been given up, and an operator holds no membership to act with.
- **Lineup swap and replace**, plus the career-total reversal fix underneath them. `reverseMatchEffect`
  only ever ran for players still in the match, which was true of every caller until a replacement
  could take somebody out of one.
- **The match modal, rebuilt** — one scoresheet table instead of two that never lined up — and three
  ways to correct a match that previously had none: its details, its figures, and who played.
- **A data-loss bug closed**: the old amend form sent three zeroed goal fields on every save, so
  correcting somebody's assists wiped their goals and the scoreline derived from them. Worth checking
  whether any completed match lost goals that way before `1.4.0`.

## 1.4.1, later the same day

**Deleting a match now unwinds it.** A completed one could not be deleted at all; it can, and the
order is the design — reverse before deleting, because `skill_rating_history.match_id` is
`ON DELETE SET NULL` and deleting first strands the rows that record what to give back. Then delete,
rebuild streaks from what remains, and replay the players' later matches, each flagged
`needs_recalc` before it runs. The endpoint answers `200` with a report instead of `204`.

On the frontend the delete control moved to where it is useful — it was offered only for matches
that had *not* been played — and its confirmation names what deleting costs before it happens.

**Released as a patch by the owner's call**, though the status code moved: the only caller is this
project's own frontend, which shipped in the matching release. The frontend also learned to survive
a `204` from an older server, so the two halves no longer have to deploy in a fixed order for
anything but deleting a completed match.

✅ **The missing `v1.4.0` / `v1.4.1` tags were pushed, and the gap is closed.** This entry used to
warn that the session cutting the release could push branches but not tags, leaving `v1.3.1` as the
newest tag in both repos while production ran 1.4.x. Verified against both remotes on 2026-08-07:
the backend now carries `v1.0.0` → `v1.5.1` and the frontend `v1.1.0` → `v1.4.2`, contiguous in
both. Nothing is owed.

**Before this, both repos were fully deployed for the first time in a while — `main`, the tag and
the running process were the same commit on both sides.** Four releases went out in two days:
`1.2.0` (security), `1.3.0` (the group-less account fix plus password policy and login throttling),
and `1.3.1` (fixes for accounts that are not players).

🚀 **Both sides now deploy themselves from `main`.** Netlify always did; Heroku does since the owner
turned on automatic deployment on 2026-08-05. **Merging a release pull request is therefore the
deploy in both repos** — there is no separate `git push heroku main:master` step to forget, and no
window where `main` is ahead of production.

What that does *not* remove is the confirmation. A deploy that started is not a deploy that booted
the right jar, and this project has a release list showing three deploys in an afternoon that a
status file had recorded as one. Ask `heroku releases -a footmania` and `/api/version`; ask Netlify
for its deploy id.

🔒 **Work goes to `next`. `main` moves only when a release is cut.** Re-stated by the owner on
2026-08-04 and both branches re-synced to `f64fa92` / `242841f`, so `main` and `next` now agree at
`1.3.1` in both repos.

This had lapsed: between 1.2.0 and 1.3.1, five feature branches and three releases went straight to
`main`, leaving `next` two releases behind. **Do not infer the target branch from recent git
history — that history is the lapse.**

⚠️ **And it lapsed again on 2026-08-05, in the other direction.** Feature work went to `next`
correctly, then the release branch was merged straight into `main` — so the bump lived only on
`main` and `next` was behind production for the rest of the day. `next` has been fast-forwarded, and
[CONTRIBUTING](CONTRIBUTING.md#cutting-a-release) now carries the missing step: after the release
merge, `git log --oneline next..main` must print nothing. Both code repos carry a short copy of the
rule, because the brain repo is not open when somebody is about to open a pull request.

The flow:

| Step | Branch | Base |
|---|---|---|
| Ordinary work | `feat/…`, `fix/…`, `test/…` | **`next`** |
| Cutting a release | `release/x.y.z` off `next` — version bump + CHANGELOG section in one commit | **`main`** |
| After a release | fast-forward `next` to `main` so the two agree | — |

`git checkout next && git merge --ff-only main && git push origin next`

`main` is what production runs, and the frontend auto-deploys from it, so a merge to `main` **is** a
deploy. `next` is where finished work waits until shipping it is a deliberate decision rather than a
side effect of merging. If `next` is ever behind `main` again, that is drift to fix, not evidence
the convention is dead.

**Verify what is deployed against the platform, not against this file.** `heroku releases -a
footmania` names the commit, and the running app's own `/api/version` confirms it independently —
worth doing both, since that pair is what catches a deploy that succeeded but booted the wrong jar.
Netlify's project page names the frontend's. This table is a snapshot and can be stale: the tag
`v1.1.0` was first placed on `4e1856e` because an earlier snapshot named it as the head, and the
platform then showed three further deploys the same afternoon.

🔧 **`git.heroku.com` has been resetting connections.** `git push heroku main:master` failed
repeatedly on 2026-08-04 while the Heroku *API* stayed reachable (`heroku releases`, `apps:info` all
fine), so it is the git transport rather than auth. This gets through:

```
git -c http.postBuffer=524288000 -c http.version=HTTP/1.1 push heroku main:master
```

✅ **All of Phase 5a is live**, and so is everything after it. The schema deployment is no longer
dark: `1.3.0` made the platform-operator account a real thing with its own console.

**`user_roles` is gone.** That table was the thing that made a rollback to pre-V22 code
survivable; V29 dropped it. See hazard 3 for what that changes about recovery.

---

## Where we are on the roadmap

See [product/roadmap.md](product/roadmap.md) for the plan itself.

| Phase | State |
|---|---|
| **0** — branding, `is_current` season gap, GDPR posture | ✅ done |
| **1** — PWA groundwork | ✅ done (frontend) |
| **2** — Web Push | ✅ done, delivery observed end to end against a real push service |
| **3** — the "steal from Capo" set | 🟡 balance-at-a-glance, waitlist, leaderboards/rankings, MOTM voting and badges all shipped in both repos. **Remaining: AI match reports** |
| **4** — Capacitor + app stores | ⬜ not started, deliberately gated on real usage data |
| **5** — multi-tenancy + billing | 🟢 **5a is complete and entirely live.** All four rungs deployed 2026-08-02: schema (`V22`–`V27`), enforcement (`V28`), the privacy fork, and onboarding (`V29`–`V31`) — group creation behind an operator-issued code, invites, and the last step of the guest arc. Running **dark**: one organization, an optional header, no behaviour change yet. Both onboarding UIs exist too — `/join/{token}` and the picker for members, invite links for group `GROUP_ADMIN`, creation codes for the platform operator. **Issuing a creation code is still the flip, and is still a separate deliberate act**: the endpoint and the screen exist, and nothing is self-serve until a code is issued. *(Whether `group_creation_codes` is still empty was true on 2026-08-03 and is not re-verified here — check the platform console rather than this file.)* **Since `1.3.0` the operator surface is real rather than latent**: `/platform` is an operator-only account's landing screen with the deployment counters and the code funnel, `V34` added deployment-wide settings, and `V35` states the rule that an account is either a platform operator or a member of groups, never both. ⛔ **The billing rung is ON HOLD by owner decision — do not start it, in any session, until the owner lifts the hold in their own words** |

Built alongside the roadmap, not on it: runtime-configurable competition rules, admin settings and
system endpoints, match-plan kickoff time and lifecycle, composable roles, and the match fee
ledger. The ledger is bookkeeping only — **no money moves through the application**, by design.
See [backend/plans/MATCH-FEE-LEDGER-PLAN.md](backend/plans/MATCH-FEE-LEDGER-PLAN.md) §14 for how a
real payment integration would bolt on later without any of it being wasted.

---

## Live hazards

**1. ~~V18 and V19 have never run against a real database.~~ Resolved 2026-08-01.** The whole
chain through **V21** is now applied in production — the guest-players feature (V20+V21, and
therefore everything before it) was observed live the day it merged. The lasting lesson stands,
twice over: `integrationTest` is the cheapest pre-flight and is still opt-in, and the first real
use of the guest feature immediately surfaced a defect the mocked suite could not see (guest
removal failed at Hibernate flush — see the guest plan's "what actually happened"). Run the real
chain, and prefer real-persistence tests for anything that deletes.

**2. ~~V22–V28 have never run against the production database.~~ Resolved 2026-08-02.** The whole
chain applied cleanly. What is worth keeping from it: unlike V17–V19, this chain had run against a
real PostgreSQL container in CI before it went anywhere near production — `integrationTest` stopped
being opt-in in time to matter. Two hazards in a row have now been closed by the same discipline,
and hazard 1's lesson is the reason this one was cheap.

**3. ~~`user_roles` must survive until the deployed chain has been live for a release.~~ Closed
2026-08-02 — by the drop happening.** V23 froze that table as the expand half of an
expand/contract, and it was the only thing making a rollback to pre-V22 code survivable. **V29
dropped it in production on 2026-08-02**, so that is no longer a hazard to manage; it is a fact
about recovery.

⚠️ **What replaces it: rolling back past V22 now means restoring a backup, not redeploying an old
jar.** Pre-V22 releases resolve authorities from `users.roles` → `user_roles`. That table is gone,
so every account would come back with no grants at all, locking every administrator out of their
own group. Anything back to the current release is fine — `membership_roles` has been the source of
truth since V23. This is the normal, intended end state of an expand/contract rather than a
mistake, and it is written here because the next person reaching for a rollback deserves to know
which kinds still work.

**4. A stale process will lie to you — and it has, three ways now.** On 2026-07-31 a match plan's
pitch cost saved correctly, notified players with the right amount, and still read "not set" in
the UI: the local JVM predated the recompile and was serving the previous `MatchPlanDTO`. On
2026-08-02 the same shape fired twice more — Turbopack's dev server served pre-change CSS (new
markup, old stylesheet: "the tabs have no CSS"), and then, seriously, **the production dyno**: the
`X-Group-Id` CORS fix was merged on `master` but the running build predated it, while Netlify was
auto-deploying every frontend push — so production ran a frontend whose every authenticated
request failed preflight, and each screen showed its empty state ("the competition rules section
is empty"). When code is verifiably right and behaviour is wrong, **establish what build the
process is actually running before debugging the code** — process start time vs file mtime
locally, deployed commit vs branch tip in production.

**6. ~~A group admin could lock a member out of every group.~~ Resolved — deployed 2026-08-03 in
`1.2.0`.**
Removing somebody used `DELETE /api/users/{id}`, which writes `users.is_active`, the flag governing
login *everywhere*, while its guard said `ADMIN` — a per-membership grant since V23. One group's
administrator was therefore ending a person's access to groups they have nothing to do with, and
that person could not log in to reach any of them. Reach was never the problem: `findOrThrow`
already refused an account with no membership here, so this was one group's decision applied to
every other group's people, not a data leak.

Both account-level writes now require the platform operator grant, and the group-scoped answer is
`PATCH /api/groups/members/{userId}/status`, which suspends a membership and leaves everything else
bit-identical. Live in production since `1.2.0`.

Two things worth keeping from how this was found. The gap was **known and written down** — the
javadoc on `findOrThrow` said `deleteUser` still deactivated the account and handed the narrowing to
5a-3, which shipped without it; a deferral recorded only in a comment is a deferral nobody is
tracking. And **no test held one account in two groups**, so every tier passed. `TenantIsolationIT`
holds one now.

**The name was the root cause, and it has been fixed too (`V33`).** `ADMIN` became `GROUP_ADMIN`
across both repos on 2026-08-03. The grant always meant "administrator *of one group*"; the name
did not say so, and `hasRole('ADMIN')` guarding a platform-wide write read as correct to everyone
who looked at it. The same guard reading `hasRole('GROUP_ADMIN')` looks wrong at a glance, which is
the whole return on a migration.

**There is deliberately no `PLATFORM_ADMIN` role**, and it is worth knowing why before somebody
proposes one again: every value in the enum is granted *per membership*, so a platform-level
constant would be grantable by any group administrator inside their own group — the same escalation
this release closes, in a worse form. The platform grant stays flat and separate in
`platform_admins`, behind `PlatformGuard`. `TENANCY-ENFORCEMENT-PLAN` §9 rejected the role form
once already.

**5. Pushing either repo's `main` IS a production deploy.** Netlify's production site tracks that
branch for the frontend and builds every push; the backend has done the same on Heroku **since
2026-08-05** — the manual `git push heroku main:master` this hazard used to describe is now only
the fallback. *Corrected 2026-08-05; `7dc787c` changed the deployment guide and missed this line.*

The ordering survives the change. Merged is not deployed, and automatic is not the same as
arrived: a build can fail after a merge succeeds. Any frontend change that depends on backend
behaviour ships backend-first — *deployed* first, confirmed via `heroku releases -a footmania` and
`/api/version`, not merely merged. This is how the 2026-08-02 partial outage happened.

**7. The visual baselines are Windows-only. ~~Every one of them is stale.~~ Regenerated 2026-08-07.**
`e2e/` compares screenshots against committed `*-win32.png` files. On Linux or macOS Playwright
silently writes a *separate* set rather than failing, so **a green run on another platform proves
nothing about the committed baselines** — that half of the hazard is permanent and is why the suite
went unread long enough for the rest of this entry to happen.

Regenerate on Windows with `npm run test:visual:update` and **look at the images**. The e2e README
already warns why: a run once photographed the "you are not in a group yet" screen on all sixteen
pages and reported green.

✅ **Regenerated 2026-08-07** — all 38, on Windows, then re-run against themselves: 38 passed, zero
differing pixels. Every image was opened and checked for real content before the commit.

**The cause was not drift, and this file said it was for a few hours.** The first version of this
entry reasoned from `maxDiffPixels: 20` that 22 of 38 failing meant font and driver differences. The
regeneration disproved it: **six baselines came back byte-identical** — `login`, `join` and `privacy`
in both themes — and those are exactly the pages with no authenticated chrome. Identical bytes mean
this machine renders text the same as the machine that took the originals, so rendering cannot
explain the other 32.

They were **real changes that had shipped and were never recorded**:

- the rankings table gained a **pagination row**, absent from the old baseline entirely
- figure columns went from left-aligned to **right-aligned** — the "Columns of figures" rule the
  frontend styling guide documents

The size deltas agree: rankings and payments moved 7–11%, other authenticated pages 0–1%, the public
pages not at all.

**The failure mode is the lesson, and it is worse than drift would have been.** A suite that is red
for long enough stops being read, and a suite nobody reads records whatever was true when someone
last looked. Two shipped features sat uncaptured, and the committed `player-modal-light` baseline was
**accepting a real defect** — the "Edit details" button flush against the dialog's right edge,
because `.player-modal-edit-zone` had no horizontal padding at all.

So the rule is not "regenerate more often", it is **do not let the suite stay red**. Red for a real
reason and red for a stale reason look identical, and the second kind hides the first.

⚠️ **Two baselines went stale on 2026-08-07, knowingly, and need a Windows run.** Season
administration became a section of the settings group tab, so `settings-admin-light` and
`settings-admin-dark` gain it. `e2e/fixtures.ts` stubs `/api/seasons` with one season of each state
so the capture shows the section rather than its empty state.

It was briefly worse. The same feature first shipped as `/seasons` with a `GROUP_ADMIN` navbar
entry, which would have moved **six** baselines — the suite's session is an administrator, because
`seedSession` seeds the pre-V18 name `'ADMIN'` and the store's `renameLegacyAdmin` upgrades it to
`GROUP_ADMIN` on rehydrate. Moving the screen into settings for product reasons took `users` and
`settings-platform` back out of scope.

Not fixable here: regenerating off Windows writes a *separate* `-linux` set and proves nothing,
which is the permanent half of this hazard. Run `npm run test:visual:update` on Windows and **look
at the images** — see hazard 8 below, which the Linux runs made newly concrete.

**9. ~~Two frontend unit tests fail on Windows and pass in CI.~~ Resolved 2026-08-07.**
`formatDate.test.ts` expected `3 – 9 Aug 2026`; Node's ICU on the owner's machine produces
`3–9 Aug 2026`, same EN DASH, no spaces around it. Both are correct — the spacing is CLDR data and
moves between ICU versions, and `en-US` kept its spaces while `en-GB` lost them.

Fixed by widening the test's existing `plain()` normaliser to collapse the separator as well as the
three invisible space characters it already handled. `src/lib/formatDate.ts` was not touched, and
the assertions still test what the module decides — the locale's own field order, and a shared month
and year said once. **Verified non-vacuous**: reverting `formatDateRange` to a hand-built join of two
`formatDate` calls still fails three of the ten.

**The lesson is the shape, not the dash.** The suite was 730/2 for months, so a local run stopped
being a signal and every session learned to skip past two red lines — which is the same failure as
hazard 7 one row up, where a red visual suite went unread until it was accepting a real defect. A
test that asserts the platform rather than the code will eventually fail on somebody's machine, and
the cost is not the failure, it is that the suite stops being believed.

**8. The app's service worker bypasses Playwright's API stubs after first load.** `stubApi` routes
through `page.route`, which the service worker's own fetches do not pass through, so the *second*
request a test makes reaches the real Next server and 404s. The current visual suite never notices
because it only screenshots initial loads; the first interaction-driven e2e test will hit it, and
it presents as the app being broken rather than the harness. `serviceWorkers: 'block'` in
`playwright.config.ts` fixes it — deliberately not applied yet, because it may shift the baselines
in hazard 7 and those cannot be regenerated off Windows.

⚠️ **Observed for the first time on 2026-08-07, and it is not confined to interaction.** Two visual
tests were run on Linux while moving the seasons screen. `/api/me/memberships` was stubbed
correctly — the navbar carried the group's name — and **every subsequent stub was ignored**: the
System figures rendered as dashes, Competition rules came back empty, the linked-player block stayed
a skeleton, and `/users` rendered the error boundary outright. Identical with and without the change
under test, so it is the harness, not the app.

That makes accepting a regenerated baseline riskier than hazard 7 already says. If this reproduces
on Windows, `--update-snapshots` would commit a set of screenshots of empty states and report
green — which is the same failure that let the rankings pagination row and the `player-modal`
padding defect sit in the baselines unnoticed. **Look at the images**, and if they show empty
sections, fix this hazard before regenerating rather than after.

---

## Known documentation drift

Found by auditing the imported documents against the code on 2026-07-31. Each line is a document
that is **wrong or incomplete**, not merely old. Fix the document, then delete its line.

| Where | Problem | Correction |
|---|---|---|
| [backend/architecture/ARCHITECTURE.md](backend/architecture/ARCHITECTURE.md) | Migration table stops at **V13** of 31, and the rest of it predates multi-tenancy | Superseded on migrations by [architecture/database-migrations.md](architecture/database-migrations.md); the layering and schema sections need a tenancy pass |
| [backend/api/API_REFERENCE.md](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API_REFERENCE.md) | No Push section, no Payments/match-fee section; `totalCostCents` on `MatchPlanDTO` still missing (it is now described in [backend/features/MATCH_PLANS_FEATURE.md](backend/features/MATCH_PLANS_FEATURE.md), but not in the reference) | Contracts exist standalone ([PUSH](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PUSH-API-CONTRACT.md), [PAYMENTS](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PAYMENTS-API-CONTRACT.md)); the reference needs the sections and the field |
| ~~[backend/features/MATCH_PLANS_FEATURE.md](backend/features/MATCH_PLANS_FEATURE.md)~~ | Last touched 2026-05-27 — predated kickoff time, lifecycle/expiry, waitlist, past-plan split and pitch cost | ✅ **Resolved 2026-08-05.** Rewritten against the entities, the migrations and `MatchPlanController`: instant kickoff, `GENERATED` and the derived `expired`/`generatable`/`cancellable` flags, the derived waitlist, guests on `players`, `timeframe` and the ordering, pitch cost, post-V33 grants, and the weekly runs `1.3.0` added. The frontend side is now [frontend/features/match-plans.md](frontend/features/match-plans.md) |
| ~~[backend/plans/MATCH-FEE-LEDGER-PLAN.md](backend/plans/MATCH-FEE-LEDGER-PLAN.md)~~ | Header read "DRAFT — not implemented"; it shipped in `828db3b` | ✅ **Resolved.** Corrected here on import, and the stale backend copy went with the documentation split — there is one copy now, and it is this one |
| [backend/plans/ORCHESTRATOR_SESSION.md](backend/plans/ORCHESTRATOR_SESSION.md) | Last entry 2026-07-28, though `orchestrator.agent.md` still mandates an entry per session | Resume it, or retire the convention deliberately |
| ~~[API_REFERENCE.md](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/API_REFERENCE.md) §generate + [backend/features/TEAM_GENERATION_DESIGN.md](backend/features/TEAM_GENERATION_DESIGN.md) §7.3/§7.6~~ | Both documented `params[formWindow]`, `params[captainAId]` and `params[captainBId]` as working generation parameters. **They had never reached a strategy.** `@RequestParam Map<String,String>` binds query keys literally, so the map arrived as `{generationType=…, params[formWindow]=3}` and every `params.get("formWindow")` returned null | ✅ **Resolved 2026-08-07**, in `1.6.0`. Fixed the binding rather than the documents, so the published contract starts being true and no caller changed. `MatchPlanControllerTest` pins the server half and `generationQuery.test.ts` the frontend half — the absence of *either* is why this survived, since every strategy test builds its context directly and never involves the controller. **`1.6.0` is the first release in which `formWindow` and the captain overrides do anything** |
| ~~Postman collection~~ | Was 60 requests against a 103-operation API, and — the deeper find — **gitignored the whole time**, so every copy lived on one person's disk and none could be diffed | ✅ **Resolved 2026-08-02.** Now *generated* from the running app's `/v3/api-docs` by `postman/generate-collection.mjs`, committed to the repo (103 requests, 17 folders, `X-Group-Id` per the tenancy contract), and regeneration is one command. The hand-maintenance workflow in `postman-engineer.agent.md` is marked historical |
| ~~[frontend/INDEX.md](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/INDEX.md)~~ | Omitted seven of its own files | ✅ **Resolved** by the documentation split — the feature docs it failed to link now live here, and what is left there is a pointer plus the seven guides |
| frontend — missing files | Nothing for guest players or payment delegation (`722335c`) as features in their own right | Write them. ✅ `features/payments.md` **written 2026-08-05** — the balance, the interleaved ledger, the roster and the delegation groups; delegation is covered there rather than separately. ✅ The group dimension is covered by [frontend/features/groups.md](frontend/features/groups.md) |
| **this repo — no tenancy feature doc** | [architecture/multi-tenancy.md](architecture/multi-tenancy.md) and the plans carry the design, but `backend/features/` has nothing on organizations, memberships or per-membership roles, while nine older features each have a file | Write `backend/features/TENANCY.md`. It is the one thing a new session most needs and currently has to reconstruct from four plans |

A related failure, already fixed: the notification-settings screen rendered `MVP_VOTE_OPEN` and
`FEE_CHARGED` as raw enum names because both categories were added backend-side without anyone
adding the `en`/`pt`/`es` strings. Two separate commits missed it, and nothing tests that the
locale files cover `NotificationCategory`.

**That exact shape recurred twice more on 2026-08-04, and both reached production:**

- The administrator role chip lost its colour. `ADMIN` → `GROUP_ADMIN` renamed the role, three CSS
  selectors kept the old name, and `RoleChips` emitted `--group_admin` against rules that matched
  nothing. **No error, in CSS or anywhere else** — the chip kept its shape and silently lost its
  fill. It shipped, survived a release, and was found by somebody looking at the screen.
- `GUESTS_MAX_PER_INVITER` and `MATCH_DEFAULT_COST_CENTS` are editable group settings that rendered
  as their storage keys, because `settings.system.settings.*` had no entry for them. The owner
  reported the guest limit as a *missing feature*; it had been configurable all along and was simply
  unrecognisable.

The class is always the same: **a name is built at runtime and looked up somewhere untyped** — a CSS
class, an i18n key — so nothing fails when the two sides drift, and the only detector is a person
noticing. Type checking cannot see it (string concatenation), and rendering tests cannot either
(jsdom applies no stylesheet; i18next returns the key and the assertion is on the key).

✅ **One parity test now exists**: `tests/lib/modifierClassParity.test.ts` covers every runtime-built
CSS modifier against `globals.css`, in both directions, with documented exemptions. It was verified
to fail on a real break before being committed.

⚠️ **The locale key-parity test still does not exist** — nothing checks that the locale files cover
`NotificationCategory`, or `AppSetting`, or `Role`. The CSS one is a template for it; the settings
labels above are precisely what it would have caught.

The shape of all of this is the same: the code shipped, the follow-through did not. That is the
reason this repo exists.

---

## Suggested next steps

1. ~~Wire `integrationTest` into CI and run it.~~ ✅ Done 2026-08-01/02. The tier is green on every
   pull request; `GuestIsolationIT`, `TenancySchemaIT` and `TenantIsolationIT` have all now
   executed. Worth naming as a win: `TenantIsolationIT` was written specifically so a leak would
   change a *number*, and it could only prove that once something ran it.
2. ~~Add the repository-predicate regression guard.~~ ✅ done 2026-08-02 —
   `RepositoryScopingGuardTest`, three checks in the unit tier. **It found two live cross-tenant
   defects the 5a-2 sweep had missed**, both inherited `JpaRepository` methods the sweep's grep
   could not see: `GET /api/players` with no `active` filter returned every group's roster, and the
   admin bulk recalculation could be pointed at another group's match ids — a cross-tenant
   *write*, since recalculation rewrites ratings. Both fixed in the same change. Each check was
   verified to fail when its defect is reintroduced, because a guard that passes for the wrong
   reason is worse than none.
3. ~~**Decide when V22–V31 deploy.**~~ ✅ All deployed 2026-08-02. The whole Phase 5a chain is in
   production and the rollback boundary has moved — see hazard 3. The next deployment decision is
   a product one rather than a schema one: **issuing the first creation code.**
4. ~~Backfill the changelog~~ ✅ done 2026-08-02 — fifteen sections written from the commit
   messages and cut as `[1.1.0]`. Keep fixing the rest of the drift table above.
5. ~~Regenerate the Postman collection.~~ ✅ Done 2026-08-02 — and made a derived artefact, so this
   line cannot come back: `node postman/generate-collection.mjs` against a running app.
6. **Add the locale key-parity test**, and a controller test asserting `totalCostCents` serialises.
   Still open, and now with three production sightings behind it — see the drift section. The
   template exists: `modifierClassParity.test.ts` does exactly this for CSS modifiers. Point the
   same shape at `NotificationCategory`, `AppSetting` and `Role` against `en`/`pt`/`es`.
7. ~~**Decide what `next` is for.**~~ ✅ Answered 2026-08-04: **revived.** Work goes to `next`;
   `main` moves only when a release is cut. Both branches re-synced to `1.3.1` — see the top of this
   file for the flow.
8. ~~**Write the season endpoints.**~~ ✅ Done 2026-08-07, in the same change set as the screen —
   `SeasonController`, `SeasonService`, `SeasonDTOs`, no migration.
   [Contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/SEASONS-API-CONTRACT.md).
   What remains is a **deployment ordering job, not a coding one**: the backend is released and
   confirmed first, and the frontend degrades to a named "not deployed yet" state in the window
   between. Worth naming as a win: `SeasonCurrentConstraintIT` now pins the season rollover both
   ways round against real PostgreSQL, and it had to — a partial unique index is not expressible in
   the JPA entity, so an implementation that flushed both `is_current` updates together would pass
   every unit test in the build and fail on the first group to own two seasons.
9. Then Phase 3's last item — AI match reports. Billing is on hold.

---

## Recent incidents

- [`INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md`](backend/fixes/INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md)
  — `GET /api/users/me` answered **404** to a platform operator, because the read asked "is this
  account a member of the current group" about the caller reading *their own record*. Fixed in
  `1.3.0`, together with the rule that removes the ambiguity: **an account is either a platform
  operator or a member of groups, never both** (`V35`).
