# Football Mania Development Learnings

**Status:** living document, brought up to date 2026-09-25, while FootMania-Simple-Front#260 (the
rankings split, step 9a) was in review. It began as a Claude Docs page on the same day; this file is
the copy to keep, and the page is a snapshot.

What the September 2026 screen-audit work (FootMania-Simple-Front #241–#260, FootMania-Back #361–#362) taught: the steps that kept every PR green on its first CI run, and the mistakes not to repeat. Hand this page to any future session before it starts work. PR numbers without a repo name are FootMania-Simple-Front's.

## Context to load first

Read these before touching code. Each settles a question a session otherwise guesses, and most of the mistakes below came from guessing.

| Repo | Read first | What it settles |
| --- | --- | --- |
| FootMania-Simple-Front | `AGENTS.md`, `.claude/rules/components.md`, `docs/guides/` (conventions, shared UI, styling, i18n, testing, release highlights) | Branches, paperwork, BEM classes in `globals.css` (no Tailwind in JSX), semantic tokens, 44px targets, named exports, types in `src/types` |
| FootMania-Simple-Front | `node_modules/next/dist/docs/` | This Next.js is newer than training data; its router behaves differently (see the URL race below) |
| FootMania-Back | `AGENTS.md`, `.github/copilot-instructions.md`, `docs/api/*-API-CONTRACT.md`, `docs/frontend/FRONTEND_ENDPOINT_CHANGES.md` | One contract per surface; frontend notes are append-only; the build includes SpotBugs |
| football-mania-brain-repo | `STATUS.md`, `CONTRIBUTING.md` (branches and releases) | Where the project stands; the canonical release rule |
| This work | [Screen responsibility audit](https://claude.ai/artifact/Q5sDaUKhwi6Ff6d9Kn1cEe) and this page | What was changed, in which PR, and why |

Three rules that are easy to miss:

- `main` is production on both sides: Netlify and Heroku deploy on every push. Every PR, release PRs included, targets `next`, and after a release `git log --oneline next..main` must print nothing.
- The backend deploys before the frontend. So a deep link is a query (`/match-plans?plan=12`, `/matches?match=40&tab=motm`), never a new path: an older frontend ignores the query and shows the list, where a new path would be "not found".
- The in-app What's new (`src/releases/unreleased.ts`) is written only on the owner's explicit yes. Say a change looks like a candidate and ask; no answer means no highlight.

## Steps of one change

Every PR in this run followed the same ten steps, and the ones that skipped a step were the ones that needed a second push.

1. **Start from the latest `next`.** After each merge: `git fetch origin next && git checkout -B <branch> origin/next && git push -f origin <branch>`. The force push is safe only because the branch holds nothing but merged history.
2. **Read before writing.** The component, its tests, its CSS, and the backend contract for any request or response shape (`docs/api/`). Find every caller with a grep before changing a component's props.
3. **Baseline first, for a refactor.** If the change is meant to move nothing on screen, first add screenshot captures that reach every state the code can show, and take their baselines on the old code (#256 added the group-admin player dialog for this).
4. **Split, then change.** One PR moves code with no visual change (#245 plan dialog, #252 match dialog, #256 player dialog); the next ones change behaviour. Each stays small enough to review.
5. **Paperwork in the same commit.**
   - Frontend: strings in `en`, `pt` and `es`; a `CHANGELOG.md` entry under `[Unreleased]` for anything a user sees; tests under `src/tests/<domain>/`.
   - Backend: the contract in `docs/api/`, an appended note in `FRONTEND_ENDPOINT_CHANGES.md`, and the changelog.
   - Remove what the change made dead: strings, CSS classes, comments describing old behaviour.
6. **Verify locally, unpiped** (commands under Verifying): lint, types, locales, releases, unit tests, build, then the Docker screenshot run.
7. **Read your own diff as a reviewer would.** Stale comments, a doc line that now lies, an unused class, a test that passes for the wrong reason.
8. **Open the PR into `next`** with the repo's template headings, trigger `visual.yml`, subscribe to the PR's activity and schedule a check-in about 40 minutes out.
9. **Merge only on the owner's say-so.** When paired, merge the backend PR first. `expectedHeadSha` must be the full 40-character SHA (`git rev-parse HEAD`). Then reset the branch (step 1).
10. **Report in plain terms:** what users will see, what was checked, what is the owner's decision, and what is still open.

## Mistakes made, and the rule for each

None of these reached production, but each cost a failed run, a second push or a correction. They come from a scan of the whole session, the provisional-teams epic (Back #358–#360, Front #239–#240) included.

**Process and tooling**

| What happened | Rule |
| --- | --- |
| `merge_pull_request` refused a short SHA, three times (Back #361, Front #252, #253) | Pass the full 40-character SHA from `git rev-parse HEAD`, every time |
| A background task reported exit 0 while the command inside failed (`npm ci` with EUSAGE; a Docker run whose log said `EXIT=1`) | Read the inner exit code in the log, never the task's status. CI uses `npm install`, so do the same |
| "No Docker" was assumed for hours; `dockerd` was installed but not running, and it died again after container restarts | Check `which dockerd`, start it, and guard every run with `docker info \|\| dockerd &` |
| An audit finding was called a bug without checking the server ("the draft picks from every confirmed player", "Delete shows on played matches"); the owner corrected both | Check the backend's `@PreAuthorize` and service rules before calling anything a bug |
| "Merge when green" for one PR was stretched to the next one | Permission to merge covers the PR it was given for; write that into each check-in message |
| A check that was red on every branch was nearly blamed on #239 | Run the same workflow on `next` first; #241 then fixed the pre-existing failure |
| Republishing the audit page without its `url` created a duplicate page | Always publish with `url`, after reading the live version |
| `git add -A` in the backend, where a leaked VAPID key once came from and the pre-commit hook is not executable | Stage files by name, and grep the staged diff for keys before committing |
| Scripted edits went wrong silently: `sed` with `\n` did nothing, a splice inserted nothing | Assert each replacement matched exactly once, then look at the result |
| A throwaway spec left in `e2e/` broke the Playwright web server, because `next build` type-checks `e2e/` | Delete throwaway specs and `test-results/` before committing |
| Maven Central answered 429 mid-build | Mirror it with a Gradle init script outside the repo; environment workarounds are never committed |
| A new test passed on code that was wrong: the What's new anchor check read the release files' own `[data-tour="…"]` selectors as definitions, so a misspelt anchor went unnoticed (#260) | Break the code on purpose and watch every new test fail once before trusting it |
| A new screenshot capture showed "Something went wrong": the page read an endpoint the fixtures did not stub, and the `{}` default is a shape no list endpoint sends (#260) | Stub every endpoint a new capture reaches, in its real shape, before taking the baseline |

**In the code**

| What happened | Rule |
| --- | --- |
| Migration V61 declared `SMALLINT` for an `Integer` field; all 210 integration tests failed on validate (#358) | SQL types match the Java field types exactly; run `integrationTest` for every migration |
| The plan link opened the list without the plan: the URL was read once, before Next had written it (#251) | Read the URL with `useSyncExternalStore`; reproduce such races in a unit test that fails on the old code |
| A delete confirm left open on one player was open on the next, because the dialog stays mounted (#243) | Reset per-item state by keying on the item id |
| After linking an account, the player dialog kept offering to link one: it showed a stale prop (#257) | Show the item from the canonical query, not the copy a page passed |
| The scoresheet and the leaderboards named different men of the match after a tie (#255) | A rule the backend applies lives once, in a helper that mirrors it, with a table test |
| "Remove guest" showed when both sides of `invitedByPlayerId === myPlayer?.id` were undefined (#244) | Guard for null before comparing ids |
| Draft lists and saved pairings went stale after writes (#243, #240) | Every mutation invalidates every list that shows its result |
| Two siblings keyed `match.id` kept stale copies on screen (#252) | Sibling keys must differ: prefix them |
| The draft's live "converted" event undid the board's own navigation to the new match (#254) | When the action is local, suppress the echo from the stream (a ref) |
| Going Back to close a dialog while navigating elsewhere raced the navigation | Leaving for another page uses `forget()`, never `close()` |
| One new hook (`useMvpVote` in `PlayedView`) failed all 63 match-dialog tests; an empty users mock silently swallowed a select change | Mock a new hook in every test file that renders it, and give selects real options |
| Focus rings boxed the ✕ and a tab after mouse clicks | `focus-visible`, not `focus`, on anything a dialog focuses on open |
| A `### Fixed` heading in the backend's `[Unreleased]` would have relabelled every paragraph below it (#358) | The backend changelog is prose paragraphs, no subheadings |

## Code patterns to reuse

Each of these replaced a bug or a tangle during the run; reach for them before inventing another.

| Pattern | Where it lives | Why |
| --- | --- | --- |
| Item in the URL as a query, read with `useSyncExternalStore` over `window.location.search` | `src/hooks/useIdInUrl.ts` (plans, matches, players) | Next writes the URL after the new page renders; reading it once into state saw the old page. `pushState` fires no event, so writes announce themselves; Back closes the item; `forget()` when leaving for another page |
| Per-item state resets by `key` on the item id, or by storing state with the id it belongs to | `PlayerDetails key={player.id}`; `MatchModal` `taskState {matchId, task}` | Dialogs stay mounted between items; a confirm left open carried to the next item (#243). The React Compiler lint forbids `setState` in an effect |
| Read the live item from the canonical query, not the copy passed as a prop | `usePlayers({ select })` in `PlayerDetails` | The prop never learns about a write: after linking, the dialog kept offering to link. Every write invalidates the one list query |
| One ⋯ menu, one task at a time, the task in place of the body | `ActionMenu` in `PlanActions`, `MatchActions`, `PlayerActions` | Controls were scattered in 3–5 places; a half-edited sheet could be lost by switching tab. Destructive items go last; the menu hides while a task is open |
| No dialog on a dialog: extract the form, render it in place | `EditPlayerForm` (wrapped by `EditPlayerModal` for self-edits) | Edit details used to open a second modal over the player |
| A rule the backend also applies lives in one helper with a table test | `manOfTheMatch.ts`; `usePlanCapabilities`, `useMatchPermissions`, `usePlayerPermissions` | The scoresheet named a man of the match the leaderboards did not count, because it used its own rule |
| Big dialog = a shell plus single-job parts in a folder | `planModal/`, `matchModal/`, `playerModal/` | 500–600-line files became parts of 40–160 lines, each changed alone later |
| Show a tab only when it has content; hold a link-requested tab while its read loads | `PlayedView` | Timeline and vote tabs were usually empty; a `&tab=motm` link must not fall back before the poll arrives |
| Share CSS by grouped selectors, never a copy | `.match-modal-menu__*, .player-modal-menu__*` | One change restyles both; the match screenshots proved it moved nothing |
| One panel per tab, mounted only while open; state that must outlive a tab switch lives in the page | `LeagueTablePanel`, `LeaderboardsPanel`, `useLeagueTableView` | Each tab fetches only when looked at, and the table's page and filter survive a look at the other tab |
| One component for a cell two layouts draw | `RankBadge` (the rankings table and the phone cards) | The two copies checked a missing rank differently, and the phone printed "undefined" |
| A tour or What's new anchor on a shared component goes through a prop, never a wrapper | `ActionMenu`'s `tour` prop | The guide wants the anchor on the element itself; a test checks every step's anchor exists |

## Verifying

`npm run build` is the frontend's verdict and `./gradlew build --console=plain` the backend's. Never pipe either into `tail` or `head`: the pipe's exit code replaces the build's, and a red build reads green. Write to a log and print the exit code instead.

```bash
# Frontend, in this order
npm run lint > lint.log 2>&1; echo "lint $?"
npx tsc --noEmit > tsc.log 2>&1; echo "tsc $?"
node scripts/check-locales.mjs; node scripts/check-releases.mjs
npx vitest run > vt.log 2>&1; echo "vitest $?"
npm run build > build.log 2>&1; echo "build $?"

# Backend (test alone skips SpotBugs); integrationTest when a migration or entity mapping changes
./gradlew build --console=plain > gradle.log 2>&1; echo "gradle $?"
# Failures are in build/test-results/test/*.xml: read them there, do not re-run to see them
```

**Screenshots run in Playwright's Docker image**, so one set of baselines serves every machine. Run the whole suite before and after any `globals.css` change. In the cloud container that meant the command below (restart the daemon with `dockerd > dockerd.log 2>&1 &` if `docker info` fails).

```bash
docker run --rm --ipc=host --network host \
  -v "$PWD":/work -v footmania-visual-node_modules:/work/node_modules -v footmania-visual-next:/work/.next \
  -v /root/.ccr/ca-bundle.crt:/ccr-ca.crt:ro -w /work \
  -e CI=1 -e PLAYWRIGHT_IN_DOCKER=1 -e HTTPS_PROXY -e https_proxy -e NO_PROXY -e no_proxy \
  -e NODE_EXTRA_CA_CERTS=/ccr-ca.crt -e SSL_CERT_FILE=/ccr-ca.crt \
  mcr.microsoft.com/playwright:v1.62.0-noble \
  bash -lc 'npm install --no-audit --no-fund > /tmp/npm.log 2>&1; npx playwright test [-g "<tests>"] [--update-snapshots]'
# Afterwards: git checkout -- package-lock.json && rm -rf test-results
```

- **Accepting a baseline:** open every `*-actual.png` (and `*-diff.png`) in `test-results/` first. Update only the tests you reviewed, with `-g`, then run the whole suite once more. A baseline update is how a regression becomes normal.
- **Reproduce before fixing:** write the test, stash the fix, watch it fail, restore. The URL race and the record form's star were both proven this way.
- **jsdom never opens a `<dialog>`**, so role queries treat its contents as hidden. Use `getByText`, or `{ hidden: true }`, and `fireEvent.click` on the text inside a menu trigger.
- **`vi.clearAllMocks()` keeps implementations.** Reset any `mockImplementation` a test set in `beforeEach`, or it leaks into the next test.
- **Mock the hook module, not react-query.** A component that gains a query (for example `useMvpVote` in `PlayedView`) breaks every test file without a provider; add it to that file's mocks.
- **Seed the role the screen needs.** `seedSession(page, theme, true, roles)` defaults to MANAGER, which never sees a GROUP\_ADMIN item such as Delete.

## The owner's calls, and what is open

Merging, What's new and product trade-offs belong to the owner. "Merge when green" covers the PRs open when it is said, not the next ones, and a change that removes or restricts something people use is flagged in its PR so it can be reversed before merge.

| Call | Where | State |
| --- | --- | --- |
| What's new highlight for the plan link | #249 | Yes, added (owner, 2026-09-25) |
| What's new for the match link, the match menu, man of the match, the player menu, the player link, the account search | #253–#260 | Yes (owner, 2026-09-25): four rows, added in #260. #256 and #260 moved nothing on screen, so they have no row of their own |
| Record form's ⭐ removed: the pick was never counted once voting existed | #255 | Flagged; merged; reversible |
| Rankings open a player read-only | #257 | Flagged; merged |
| The leaderboards load when their tab is first opened, not with the table | #260 | Flagged; in review |
| Any manager may confirm or cancel a captains' draft (was captain or admin only) | #243 | Flagged; merged |
| Promote to member stays in view for a guest, not in the menu | #257 | Explained in the PR |

Open on 2026-09-25:

- [ ] Chat page `?conversation=` probably has the same URL race the plan link had; move it onto `useIdInUrl`.
- [ ] A dialog opened straight from a link shows a focus ring on its first control (the match ⋯). The fix belongs to where the dialog puts initial focus.
- [x] Player dialog: search in the account picker (#259, merged) and a link per player, `/players?player=4` (#258, merged).
- [ ] Rankings, the last screen in the audit. The split is in review as #260. Still to do: honours reached from one place instead of three, the tab and season in the URL, a "jump to me", the season selector's label, and Team of the week opening as a dialog on a dialog.
- [ ] The provisional-teams What's new step was reworded; confirm it still reads right.
- [ ] An older duplicate of the audit page exists; the current one is shared as "Anyone with the link".
- [ ] The audit's plan (steps 1–8) lives only on the audit page; per the repos' rules it belongs in the brain repo, with this page.
