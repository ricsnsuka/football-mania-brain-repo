# Project Status

**Snapshot: 2026-08-05.** Update it when the answers change, not on a schedule — a status page
nobody trusts is worse than none.

| | Backend | Frontend |
|---|---|---|
| Branch | `main` (production), `next` (integration) | `main` (production), `next` (integration) |
| `next` head | `358285b` — level with `main`, checked after the release merge | `f343678` — level with `main`, checked after the release merge |
| Release | **`1.4.1` — merged to `main`** | **`1.4.1` — merged to `main`** |
| Running in production | `9fa0134` — Heroku release **v56** (1.4.0). **1.4.1 is merged and the Heroku push was not confirmed to this session** — ask `heroku releases` before believing any line about it | `f343678` merged to `main`, which is what Netlify publishes from — **deploy id not verified against the platform** |
| `main` head | `358285b` — 1.4.1, ahead of the confirmed deploy | `f343678` — 1.4.1 |
| Working tree | clean | clean |
| Tests | 1104 unit + 76 integration, SpotBugs clean | 677 unit |
| Latest migration | `V35__platform_admin_exclusivity.sql` | — |
| Deployed through | **`V35`** (2026-08-04) — 1.4.0 carried no migration | — |
| Tags | `v1.0.0` → `v1.3.1`; **`v1.4.0` and `v1.4.1` both missing** | `v1.1.0` → `v1.3.1`; **`v1.4.0` and `v1.4.1` both missing** |

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

⚠️ **The `v1.4.0` tags are written and not pushed, and `v1.4.1` has none either.** The session that cut the release could push
branches but not tags, so both annotations exist only in that session — they name `9fa0134` and
`4e87ddc`, and the backend one records that the SHA first reported for v56, `9fa0343`, matches no
object in the repository and is a transposition. **Until somebody pushes them, `v1.3.1` is the
newest tag in both repos while production runs 1.4.x — two releases of daylight.**

The four that are owed:

```bash
# backend                                     # frontend
git tag -a v1.4.0 9fa0134 …                   git tag -a v1.4.0 4e87ddc …
git tag -a v1.4.1 358285b …                   git tag -a v1.4.1 f343678 …
```

Place `v1.4.1` at whatever the platforms report rather than at these SHAs if they disagree.

**Before this, both repos were fully deployed for the first time in a while — `main`, the tag and
the running process were the same commit on both sides.** Four releases went out in two days:
`1.2.0` (security), `1.3.0` (the group-less account fix plus password policy and login throttling),
and `1.3.1` (fixes for accounts that are not players).

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

**5. Pushing the frontend's `main` IS a production deploy.** Netlify's production site tracks
that branch and builds every push; the backend is a **manual** `git push heroku main:master`, and
the running dyno can lag `main` by hours. Merged is not deployed. Any frontend change that
depends on backend behaviour ships backend-first — *deployed* first, not merged first. This is how
the 2026-08-02 partial outage happened.

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
| ~~Postman collection~~ | Was 60 requests against a 103-operation API, and — the deeper find — **gitignored the whole time**, so every copy lived on one person's disk and none could be diffed | ✅ **Resolved 2026-08-02.** Now *generated* from the running app's `/v3/api-docs` by `postman/generate-collection.mjs`, committed to the repo (103 requests, 17 folders, `X-Group-Id` per the tenancy contract), and regeneration is one command. The hand-maintenance workflow in `postman-engineer.agent.md` is marked historical |
| ~~[frontend/INDEX.md](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/INDEX.md)~~ | Omitted seven of its own files | ✅ **Resolved** by the documentation split — the feature docs it failed to link now live here, and what is left there is a pointer plus the seven guides |
| frontend — missing files | No `features/payments.md`, though `a3efac0` shipped the whole "what you owe" UI; and nothing for guest players or payment delegation (`722335c`) | Write them. Every other frontend feature has a file. ✅ The group dimension is now covered by [frontend/features/groups.md](frontend/features/groups.md) |
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
8. Then Phase 3's last item — AI match reports. Billing is on hold.

---

## Recent incidents

- [`INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md`](backend/fixes/INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md)
  — `GET /api/users/me` answered **404** to a platform operator, because the read asked "is this
  account a member of the current group" about the caller reading *their own record*. Fixed in
  `1.3.0`, together with the rule that removes the ambiguity: **an account is either a platform
  operator or a member of groups, never both** (`V35`).
