# Keeping this repo true

A source of truth is only worth the trust people put in it. One wrong table read as authoritative
does more damage than the same table in a repo nobody believed.

## The rules

**1. This repo is canonical.** When a document here and one in a code repo disagree, this is the
one to fix and to trust.

**2. Per-change API contracts still ship with the code.** A change that touches the API updates its
contract in `docs/api/` in the *same commit* — that is what keeps contracts honest, because the
person changing the endpoint is the person who knows what changed. Bring it here as part of the
same work.

**3. Write why, not what.** The code already says what. These documents earn their keep by
recording the decision, the alternative that was rejected, and the trap that was hit. The migration
history could be reconstructed accurately in an afternoon precisely because every migration opens
with a comment explaining why it exists.

**4. State the status, and date it.** Any document describing work in progress gets a `Status:` line
that says whether it is a draft, shipped, or abandoned. A plan that shipped and still says "DRAFT —
not implemented" sends the next reader to build it again.

**5. Never silently narrow.** If a document is wrong and you cannot fix it now, add it to the drift
table in [STATUS.md](STATUS.md) rather than leaving it to be discovered.

**6. Link, don't copy.** The roadmap was duplicated in two repos and both copies drifted. One file,
many links.

## When something ships

- [ ] Contract updated in the code repo, same commit as the code
- [ ] Feature doc here updated — or written, if the feature is new
- [ ] [product/feature-status.md](product/feature-status.md) row updated
- [ ] Migration added to [architecture/database-migrations.md](architecture/database-migrations.md)
      if the schema moved
- [ ] User-visible strings exist in **all three locales** — `en`, `pt`, `es`. A missing key falls
      back to the raw enum name and reaches production looking like `FEE_CHARGED`
- [ ] Backend deployed *before* the frontend that depends on it, and confirmed running — see
      [Deployment order](#deployment-order). Merged is not deployed
- [ ] CHANGELOG entry says what actually happened — a release that shipped does not still read
      "awaiting deployment"
- [ ] [STATUS.md](STATUS.md) still accurate

## Branches and releases

Both code repos use the same two permanent branches, and nothing else is permanent.

| Branch | Means |
|---|---|
| `main` | production. For the frontend, pushing it **is** a deploy |
| `next` | integration. Every pull request bases here |
| `vX.Y.Z` | a tag, not a branch — immutable, one per release |
| `release/X.Y.x` | cut from a tag **only** when a shipped release needs a patch and `next` has moved on |

**Releases are tags because branches were tried and failed.** The backend accumulated thirteen
version branches — `v3.0.0` through `v4.1.0` — and not one of them held a commit that was not
already in `master`. They were tags all along, with the added cost that every clone listed them
forever. A branch nobody commits to after the release is a tag with worse ergonomics.

**The version lives in `build.gradle` and `package.json`, never in a branch name.** A branch named
for a version has to be renamed every release, and renaming touches the default branch, both CI
trigger lists, every open pull request's base, and every clone. It also drifts: the branch called
`v1.0.0` was shipping `1.1.0` for two months before it was renamed.

**Renaming a branch is a deployment event, not bookkeeping.** Two things do not follow a GitHub
rename and will fail silently:

- **Netlify** deploys the frontend from a branch named in its site settings. Change that setting
  *before* renaming the branch, or production deploys stop — or worse, resume from the wrong
  branch. See hazard 5 in [STATUS.md](STATUS.md) for what that already cost once.
- **Heroku** deploys from `main` automatically since 2026-08-05, but it still keeps its own
  `master` internally. Renaming the branch it watches breaks that connection silently, and the
  manual fallback if it ever comes to that is `git push heroku main:master` — pushing `main` alone
  creates a branch Heroku never builds and reports success having deployed nothing.

CI trigger lists are the third: a workflow still filtering on a branch that no longer exists does
not fail, it stops running, and an unchecked pull request looks exactly like a passing one.

### Cutting a release

`next` accumulates finished work for as long as it needs to. Releasing is a decision someone makes,
not something that follows automatically from work being done — so the steps below start when you
have decided, not when `next` is green.

1. **Bump the version.** Backend: `build.gradle` *and* the jar name in `Procfile` — the Version
   Check job fails the build if they disagree. Frontend: `package.json`. Bump at release time, not
   while work accumulates.
2. **Close the CHANGELOG section.** `[Unreleased]` becomes `[X.Y.Z] — date`, and the note says what
   shipped, including whether it went out dark.
3. **Merge `next` into `main`** in both repos. Fast-forward if it can. **The release branch merges
   into `next` first** — it is a pull request like any other, and merging it straight into `main` is
   what broke this on 2026-08-05 (see below).

   Since 2026-08-31 the release PR into `main` does **not** re-run the test pipeline — full CI
   runs on the `next` side, where green still gates something. The PR runs only a **Release
   Gate** (plus the backend's Version Check): a job asserting `main` is an ancestor of the PR
   head, which is what makes skipping the re-run safe rather than assumed. A red gate means
   `main` holds a commit that never went through `next` — the merge result would be a tree
   nobody tested. Bring that commit into `next` first, or run the workflow manually via
   `workflow_dispatch`. Step 7 done properly is exactly what keeps this gate green.
4. **The backend deploys itself** from `main` — automatic deployment was turned on 2026-08-05, so
   step 3 already shipped it. **Confirm it anyway**: `heroku releases -a footmania` for the commit
   that actually landed, and `/api/version` for what the running process says it is. A deploy that
   started is not a deploy that booted the right jar. See [Deployment order](#deployment-order) for
   why the backend still goes first.
5. **The frontend deploys itself too**, on the same merge to `main`. Confirm the Netlify deploy id
   rather than assuming the push published.
6. **Tag `vX.Y.Z` in both repos, at the commit that was deployed** — not at the branch tip, unless
   they are the same. Annotate the tag with how you established that.
7. **Make `next` identical to `main` again**, and check it rather than assuming it:

   ```bash
   git log --oneline next..main   # must print nothing, in both repos
   ```

8. **Update [STATUS.md](STATUS.md)** with the release, the running commit, and the platform's own
   release number.

**Step 7 exists because step 3 was done backwards on 2026-08-05.** A frontend release branch cut
from `next` was merged straight into `main`. The version bump therefore existed only on `main`,
`next` sat two commits behind production for the rest of the day, and every branch cut from `next`
afterwards started from a base missing what had shipped. Nothing failed and nothing said so — a
branch that is behind looks exactly like a branch that is not.

**"Every pull request bases on `next`" was already written here and did not prevent it.** The rule
was true and incomplete: it says where work enters, and says nothing about the one merge that goes
the other way, or about the branch that is left behind by it. A release is the only time `main`
moves, and it is the only time the two branches can disagree — so it is the only place the check
belongs.

Both code repos now carry a short copy of this pointing back here — frontend
[`AGENTS.md`](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/AGENTS.md), backend
[`.github/copilot-instructions.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/.github/copilot-instructions.md).
That is a deliberate exception to "link, don't copy": the rule governs the moment somebody is inside
one of those repos about to open a pull request, and a rule they have to leave the repo to read is
one they will not read. Each copy is four sentences and names this file as canonical.

**Step 6 is the one that goes wrong.** A release tag is a claim about production, so it is only as
good as the evidence behind it. Placing `v1.1.0` from a status document put it three deploys early;
the platform's release list put it right. Ask the platform, then tag.

### Naming a working branch

Everything that is not `main` or `next` is temporary and named `type/what-it-does`, kebab-case:

```
feat/group-invite-expiry      fix/cors-x-group-id
docs/changelog-1.1.0-shipped  chore/rename-branches-to-main-next
refactor/tenant-resolver      test/guest-removal-flush
```

Types are `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`. Delete the branch when its
pull request merges — GitHub does it for you if you let it, with **one thing to check first**: see
[Deleting the branch under a stacked pull request](#deleting-the-branch-under-a-stacked-pull-request).

**Name the branch after the change, not after who or what made it.** The repos accumulated
`copilot/fix-jarfile-access-error`, `copilot/fix-jarfile-access-error-again`, and
`claude/handoff-continuation-axeqfs`. The first two tell you a build broke twice and nothing about
what was wrong; the third tells you nothing at all. Six months later the author is the least useful
fact about a branch, and a generated suffix is worse than useless — it cannot be guessed, typed, or
recognised. This applies to branches an agent opens too: the agent picks the name, so the agent
follows the convention.

**One branch, one reason to exist.** A branch that fixes CORS *and* reorganises docs cannot be
reviewed as either, and cannot be reverted without taking the other with it.

#### Deleting the branch under a stacked pull request

**Retarget the child first.** Deleting the base branch of an open pull request makes GitHub
**auto-close** that pull request — and a closed one cannot be reopened or retargeted once its base
is gone. The work is not lost, but the review, its comments and its number are: the only way back is
a new pull request from the same commit.

That is not hypothetical. [Front #80](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/80)
was the locale key-parity test, stacked on
[#78](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/78) for the `MANUAL` key it needed.
Merging #78 with `--delete-branch` closed #80 on the spot, and
[#82](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/82) is that same commit (`9f7145d`)
opened again against `next`. **The lesson is about deleting the base, not about stacking** — stacking
is fine, and is often the honest way to say "this needs that first".

So, in order: **retarget the child at `next` first, then merge the parent, then delete.**

⚠️ **"Automatically delete head branches" makes this automatic rather than a choice somebody makes.**
It is worth turning on — it is the only thing that stops the branch list growing by one per release,
and the backend has some forty stale ones to show for not having it. But it removes the pause in
which somebody would have noticed the stacked child. With it on, retargeting first stops being good
practice and becomes the only protection there is.

## Deployment order

**The backend goes first, and *deployed* means confirmed — not merged.**

| | Trigger | Lag |
|---|---|---|
| Frontend | merging `main` — Netlify builds every push | minutes, automatic |
| Backend | merging `main` — Heroku builds every push, **since 2026-08-05** | minutes, automatic |

The asymmetry this section was written for is gone: both sides now deploy themselves, so a backend
change can no longer sit merged and unshipped while the frontend that needs it goes live. What
remains is the ordering and the confirmation.

For any change where the frontend depends on new backend behaviour:

1. Merge the backend release. That **is** the deploy.
2. Confirm the running dyno actually has it — `heroku releases -a footmania` for the commit that
   landed, `/api/version` for what the process says it is. Automatic is not the same as arrived,
   and a build can fail after a merge succeeds.
3. Then merge the frontend, which deploys itself on the same terms.

**This is written down because the obvious order caused a partial outage on 2026-08-02**, back when
the backend needed a human to push it. The
`X-Group-Id` CORS fix was merged on the backend and the frontend that needed it was pushed the same
day. The frontend deployed itself in minutes; the backend sat merged and undeployed. Every
authenticated request failed preflight, and because each screen degraded to its empty state rather
than erroring, it did not look like an outage — it looked like missing data. It was found as "the
competition rules section is empty".

**Migrations tighten this further.** A backend deploy that carries schema changes cannot be rolled
back by redeploying the previous jar once a contract step has dropped something — see hazard 3 in
[STATUS.md](STATUS.md), where pre-`V22` code would come back with every account holding no grants
at all. Deploy the backend on its own, confirm the chain applied, and only then ship the frontend
that assumes it.

**When the order genuinely cannot hold**, ship the frontend so it tolerates both shapes — feature
detection or a graceful empty state that says so — rather than shipping something that assumes a
backend that is not there yet.

## Structure

Curated documents live at the top level and in `product/` and `architecture/`. `backend/` and
`frontend/` mirror each code repo's `docs/` tree — keep that shape, because the relative links
inside those documents depend on it. A new document about one side goes in that side's tree; a
document about how the two fit together goes in `architecture/`.
