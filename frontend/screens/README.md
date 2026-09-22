# Screen captures

Two full-page screenshots of every routed screen in the frontend — light and dark — 1280px wide.

They exist for the **Screens** board in Notion, where each screen has a page describing what it is
for, who reaches it and what is on it — the picture is what makes that page worth opening. Notion
stores its own copy of each image, so these files are the source rather than the thing Notion
serves.

## What is in them

Nothing real. Every request is answered from the frontend's visual-test fixtures
(`e2e/fixtures.ts`), so the group is "Sunday League", the players are Ana Costa and Bruno Alves,
and the addresses are `example.com`. No deployment, no database and no account is involved, which
is also why these can live in a public repository.

The session behind each capture is the one that makes the screen worth looking at: an organiser on
payments and match plans, an administrator on members, the draft queue and moderation, an operator
on the platform console, an account with no group on the three onboarding screens.

Both themes come from the same run, twice over: the theme is seeded into `localStorage` before
the first script executes, and the capture waits on the `dark` class actually being applied, which
is also what proves the client has taken over from the server render. Dark is not a filter over
the light capture — every screen is re-rendered under it, which is the point of having both.

## What they do not cover

- `/` — the entry redirect renders nothing; it decides where to send you.
- Modals. Everything here is a page at rest.
- Chat's live indicator reads "Reconnecting…", because the event stream is stubbed like every
  other request. On a real deployment it reads "Live".

## Regenerating them

There is no committed script: the capture run is written when it is needed and deleted afterwards,
because a checked-in copy of `visual.spec.ts` that nobody runs is a copy that rots. What it does is
small enough to state:

1. In the frontend repo, add a Playwright spec beside `e2e/visual.spec.ts` that imports `stubApi`
   and `seedSession` from `e2e/fixtures.ts`, walks the routes, and calls `page.screenshot()`
   instead of `toHaveScreenshot()` — nothing here is a baseline and nothing here should become one.
   Run it once per theme (`SHOT_THEME=light`, then `dark`).
2. Block the service worker (`test.use({ serviceWorkers: 'block' })`). A production build registers
   one, and the requests it makes are invisible to `page.route`, so the stubs are bypassed and the
   page renders as if the API were down.
3. Mark the guided tours done in `localStorage` (`tour:<page>:v1` = `done`), or the first-visit
   overlay dims the screen being photographed.
4. Grow the viewport before capturing: the shell pins `body` to the viewport and scrolls inside
   `.app-main`, so `fullPage` on its own photographs the first 900px and nothing else.
5. The shared fixtures answer every unlisted route with `{}`, which throws on any page that
   iterates the answer. Badges, suspended members, honours and the chat routes each need a stub of
   their own.

**Look at the pictures before committing them.** The same warning is in the frontend's
`e2e/README.md` and for the same reason: a capture run goes green whatever it photographed.
