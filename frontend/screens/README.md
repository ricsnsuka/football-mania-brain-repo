# Screen captures

Every routed screen and every dialog in the frontend, photographed full-page.

| | Desktop 1280 | Mobile 390 |
|---|---|---|
| **Screens** (23) | light + dark | dark |
| **Modals** (28, in `modals/`) | dark | dark |

They feed two places: the **Screens** board in Notion, where each screen has a page describing what
it is for and who reaches it, and the **Football Mania — Screens (dark)** Figma file, where each
capture is a named frame at its real pixel size. Both keep their own copy of every image, so these
files are the source rather than the thing either one serves.

`figma-frames.json` says which capture belongs in which Figma frame. All 102 are in place; the
mapping stays because re-uploading a regenerated capture is then one call per frame rather than a
layout job done again. It needs network access to `mcp.figma.com`, which is not in the default
allowlist for a cloud session.

## What is in them

Nothing real. Every request is answered from the frontend's visual-test fixtures
(`e2e/fixtures.ts`), so the group is "Sunday League", the players are Ana Costa and Bruno Alves,
and the addresses are `example.com`. No deployment, no database and no account is involved, which
is also why these can live in a public repository.

The session behind each capture is the one that makes the screen worth looking at: an organiser on
payments and match plans, an administrator on members, the draft queue and moderation, an operator
on the platform console, an account with no group on the three onboarding screens — and, for the
self-link dialog, an account with no player of its own, because that is the only state that offers
it.

Both themes come from the same run, twice over: the theme is seeded into `localStorage` before
the first script executes, and the capture waits on the `dark` class actually being applied, which
is also what proves the client has taken over from the server render. Dark is not a filter over
the light capture — every screen is re-rendered under it, which is the point of having both. The
same goes for width: 390 is a real re-render, which is why the roster is a list of cards there and
a table at 1280.

## What they do not cover

- `/` — the entry redirect renders nothing; it decides where to send you.
- Mobile light, and light-theme modals. The same run produces them; they are simply not committed.
- Chat's live indicator reads "Reconnecting…", because the event stream is stubbed like every
  other request. On a real deployment it reads "Live".
- The "What's new" dialog shows its empty state, since no release highlights are pending.

## Regenerating them

There is no committed script: the capture run is written when it is needed and deleted afterwards,
because a checked-in copy of `visual.spec.ts` that nobody runs is a copy that rots. What it does is
small enough to state.

**For the screens**, a Playwright spec beside `e2e/visual.spec.ts` that imports `stubApi` and
`seedSession` from `e2e/fixtures.ts`, walks the routes, and calls `page.screenshot()` instead of
`toHaveScreenshot()` — nothing here is a baseline and nothing here should become one. Run it once
per theme and width.

**For the modals**, the same setup plus an `open()` per dialog, because a page at rest cannot
reach one. Screenshot the dialog rather than the page, and prefer its `__panel` child where it has
one: a full-bleed dialog is mostly scrim.

Five things the run has to get right, each of which cost a wrong screenshot first:

1. **Block the service worker** (`test.use({ serviceWorkers: 'block' })`). A production build
   registers one, and the requests it makes are invisible to `page.route`, so the stubs are
   bypassed and the page renders as if the API were down.
2. **Mark the guided tours done** in `localStorage` (`tour:<page>:v1` = `done`), or the
   first-visit overlay dims the screen being photographed.
3. **Grow the viewport before capturing.** The shell pins `body` to the viewport and scrolls
   inside `.app-main`, so `fullPage` on its own photographs the first 900px and nothing else. A
   dialog taller than the viewport is clipped the same way — capture modals at ~2200px tall.
4. **Stub the routes the shared fixtures answer with `{}`.** Badges, suspended members, honours,
   chat, the plan behind a card (`/api/match-plans/{id}`, which the list stub does not match) and
   a player's own ledger (`/api/players/{id}/payments`) all throw into the error boundary
   otherwise. A page showing "Something went wrong" is what an unstubbed route looks like.
5. **Click what is visible.** At phone width the roster's table is still in the DOM, hidden, beside
   the card list — so a row selector finds the invisible one. Match on `[aria-label]:visible`.

Seed `GROUP_ADMIN` rather than `ADMIN` when a capture needs an administrator: the fixture's type
still carries the old name, and the stale one rehydrates to no admin entries in the nav and a raw
role chip in the members table.

**Look at the pictures before committing them.** The same warning is in the frontend's
`e2e/README.md` and for the same reason: a capture run goes green whatever it photographed.
