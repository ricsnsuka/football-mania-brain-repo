# What's new — release notes inside the app

**Status:** on `next` since 2026-09-17, ships in 3.11.1 · frontend only · epic
[#81](https://github.com/ricsnsuka/football-mania-brain-repo/issues/81), sub-issues
FootMania-Simple-Front #203 #204 #205 (built), #206 #207 (filed, later).
Design proposal with an interactive mock: https://claude.ai/artifact/SnoSqNLuSQkBbiwiASwkJ6.

## What the person sees

The first visit after a deploy opens **What's new in x.y.z**: one row per highlight of the
release, in their language, with a **Show me** on each. Show me closes the dialog, goes to the
feature's page and lights the feature up in place with the same spotlight and popover the page
tours use, one step at a time; when several highlights were listed, the last popover of each
offers the next. **Got it** remembers the version on this device. **Later** (also Esc and the
backdrop) keeps quiet for the visit. Releases somebody skipped are grouped into one dialog,
newest first, three at most.

It reopens on demand from Settings → Tutorials (*Show what's new*, a second row beside *Replay
tutorials*) and from the version number in the footer, which is now a button.

## Who gets it, and when

- **Only a device that has finished at least one page tour** gets the dialog on its own. A
  newcomer is greeted by the page tours; greeting them with release notes at the same time would
  be two things at once. The running version is stamped silently instead, so the dialog appears
  from the next release on. *Replay tutorials* clears the tour flags and therefore closes this
  gate again until a tour is finished; it never clears the seen version.
- **A device with no seen version but a finished tour** is an existing member from before this
  feature: they get the most recent releases (capped at three), which is what let the dialog show
  something on its own first day — 3.11.0's three payments features were backfilled for it.
- **A highlight that needs a backend version** (`needsBackend`) is held back while `/api/version`
  reports an older one, and nothing is stamped in that case, so the dialog appears once the
  backend has caught up. This covers the minutes between the two deploys and a frontend that
  ships alone.
- **Role.** `audience` filters against the roles held in the active group before the dialog
  renders; a member never sees Show me for an organiser's panel.
- **A signed-in account without a group** (the picker) never gets it.

## How it works

- The app knows its own version at build time: `next.config.ts` bakes `package.json`'s
  `version` into `NEXT_PUBLIC_APP_VERSION`. Netlify builds from `main`, so the bundle always
  carries the released number. The footer keeps showing the backend's version, which is the one
  people quote; the frontend's rides in the tooltip.
- `whatsnew:lastSeen` (localStorage) is the version last stamped; it only ever moves forward.
  `whatsnew:snoozed` (sessionStorage) is Later. `whatsnew:<version>:<id>` marks a walked
  highlight. Every access is wrapped, as the tours' are: a private window simply never shows it.
- The decision is made **once per page load**, in a module store rather than a component ref,
  because the shell that hosts the dialog remounts when the route leaves the `(app)` group for
  `/settings` and a walk-through that navigates there would otherwise meet the dialog twice.
- Show me writes a queue (`whatsnew:queue`, sessionStorage, ids only) and navigates. A runner
  mounted once in the shell, before the page, waits for the highlight's first anchor with a
  MutationObserver and an 8 s timeout, then drives the tour. If the anchor never appears (the
  organiser's review panel exists only while something is pending) it shows one centred popover
  with the highlight's `fallbackKey` line and moves on. Closing mid-way drops the queue and
  snoozes; finishing the last one stamps the version.
- Page tours and the dialog never overlap: opening the dialog, or driving a Show me, snoozes
  the current page's own tour for the session. That tour comes back next visit.
- The tour builder is shared: `usePageTour` and the runner both call `buildTour`, so the popover,
  buttons, keys, the finished/dismissed split and the top-layer trick for modals are one thing.

## The author's part

The pull request that ships a feature appends its highlight to `src/releases/unreleased.ts` in
the same commit as the code, next to its changelog entry, with strings in the three locales.
The frontend's `docs/guides/release-highlights.md` is the how-to; `AGENTS.md` lists it with the
rest of the same-commit paperwork. The release skill renames the file to `v<version>.ts` when it
stamps `[Unreleased]`, and `scripts/check-releases.mjs` (CI) holds every stamped file to a
matching changelog section and resolvable keys. A release with no highlights is legitimate.

## Decisions taken 2026-09-17, not to be re-opened

1. All five issues filed; 1 to 3 built first, 4 (Try it steps) and 5 (service-worker update
   prompt) later.
2. The seen version lives in localStorage only, per device, like the tours. No backend value.
3. Only accounts with a finished page tour get the dialog on its own (above).
4. **No scripted clicks.** A feature behind a tab or a modal gets a "Try it" step (#206): the
   person opens it and the walk-through continues when the target appears. The ghost cursor in
   the design mock is not built.
5. Existing tour keys are not bumped for a new card; `v1` → `v2` stays for a redesigned page.

## Out of scope

A per-account seen version on the backend; video or GIFs in the dialog; anything outside the app
(`CHANGELOG.md` stays the written record).
