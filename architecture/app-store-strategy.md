# Putting FootMania on the Apple App Store

**Status: IN PROGRESS, 2026-11-06 — the native iPhone app chosen on 2026-11-02 (below) is built
through Phase 5 and is on TestFlight at build 6, with push working in production and the Apple Watch
app working end to end on real devices. It is not on the App Store yet: the listing is left. The
Capacitor route this document first drafted was not taken; its obstacles still apply.** Written
2026-08-03 against backend `2981bd3` and frontend `ac831ee`, both at `1.1.0`. This is roadmap
[Phase 4](../product/roadmap.md), which is deliberately gated on real usage data rather than on
enthusiasm. Read [When not to do this](#when-not-to-do-this) before the rest.

**A throwaway rung-1 try-out ran on 2026-10-31**, against `3.15.3`, and the app ran in the iOS
Simulator. Nothing from it is merged. It found that the static export no longer works unchanged —
[the record](#the-rung-1-try-out-2026-10-31) supersedes the "wrap it" claims below where they
disagree.

**Decision, 2026-11-02: a native SwiftUI app, not the Capacitor wrap.** After the try-out's verdict
("a website in a frame"), the owner chose to rebuild the frontend natively in Swift and SwiftUI for
iPhone, as a new client of the existing backend. This supersedes the Capacitor recommendation in
[The short answer](#the-short-answer). The obstacles below still apply to it: native push, guideline
4.2, the payments rule, and the release and version discipline. **Android is undecided**; if it
comes, it is a separate Kotlin app. Nothing is built yet. The phased plan, the screen-by-screen map
and the decisions still open live in the owner's private guide
[Football Mania in Swift](https://claude.ai/artifact/CoGWNyawucDLJ1pkPR8Cy2).

**Phase 1 spike, 2026-11-02: passed.** The owner ran a throwaway SwiftUI app (log in, see the next
match, mark availability; [the spike guide](https://claude.ai/artifact/Fd4wzRVTfdibJwqmdDm9De))
against a local backend in the Simulator. Its six files compiled and ran exactly as written on
Xcode 27.2, although no compiler had seen them before the owner ran them. A session on the owner's
Mac then rebuilt it with `xcodebuild` and compared the local backend's real JSON with the spike's
models: login, the match-plans page and `confirmations/me` all decode, and `createdAt`, `updatedAt`
and `joinedAt` carry six-digit fractional seconds, which is why the custom date decoder exists. The
verdict: **"it feels like a proper app"**, against "a
website in a frame" for the Capacitor wrap on the same backend. That clears the gate the plan set for
Phase 1. It proves the feel, not the cost: the spike is one flow in about 450 lines, and none of
the expensive parts (the design system, translations, live chat and drafts, offline goal entry,
push) were touched. Phase 2 waits on the eight decisions in the guide.

**Phase 2, the foundation: done 2026-11-03**, in six pull requests to FootMania-iOS, each built and
tested on the owner's Mac before merging (121 tests by the last):

| PR | What it added |
|---|---|
| #3 | Log in, stay logged in (Keychain), choose a group, log out |
| #4 | The web's colours in light and dark, the shared building blocks |
| #5 | The app's wording copied from the web's locale files by a script |
| #6 | Every feature from the home screen, as the web's own page inside the app (with web #289) |
| #7 | The minimum-build check and update screen (with backend #383) |
| #8 | Forgot your password, from the login screen |

That meets the guide's exit: the app logs in, switches groups, shows every feature (most as web
screens) and looks like Football Mania. **Phase 3 is next**: the screens every player uses, built
natively into the tab bar settled below.

**Phase 3, what every player uses: done 2026-11-04**, in sixteen pull requests to FootMania-iOS
(#9 to #25; #15 is an issue, not a pull request). Each was built and tested on the owner's Mac,
unpiped and with Xcode closed, and checked in the Simulator by the owner before merging. There were
369 tests by the last. The first ten (#9 to #19) were written in a cloud session with no compiler
and built on the Mac, apart from #16's paging commit, which the Mac session wrote. From #20 on, the
Mac session wrote them too, after the cloud session handed the work over.

| PR | What it made native |
|---|---|
| #9 | The tab bar: Home, Matches, Rankings, Chat, More |
| #10 | Home: your next match and your answer |
| #11 | Matches → Upcoming: the plan list and a plan's screen |
| #12 | The saved teams and the team vote; a plan's screen in two parts, Squad and Teams |
| #13 | Matches → Results: the match list, the scoresheet, and the squads with skill ratings |
| #14 | The man of the match vote on a played match |
| #16 | Rankings: the league table (paged, 5, 7 or 10 a page) and the leaderboards |
| #17 | A player's profile, from the league table |
| #18 | A player's profile from the match screens too |
| #19 | Your balance, in More → Payments |
| #20 | "I've paid", one report per person, from your credit or not |
| #21 | Settings, your data: a copy of it, leaving a group, and deleting your account |
| #22 | Settings: your profile, password, theme and language |
| #23 | Chat: the conversations, reading and writing, the live stream, the unread badge |
| #24 | Chat: starting a conversation, who's around, reporting, leaving, deleting; a match's chat |
| #25 | Rankings → Honours: the week, the month, the season's awards, the Ballon d'Or roll |

**Against the exit ("a player never needs the web screens"): met for everything a player does
every week, with five links left.** The app still opens these web pages for a player:

- **More → Profile**: your own player profile. The native profile exists (#17); More doesn't open it yet.
- **Settings on the website**: notifications and the linked player.
- **A match on the web**: the goal timeline, and recording the result.
- **A plan on the web**: making and running plans.
- **Home → Dashboard**.

None blocks Phase 4; the first is the obvious next small change.

Settled during Phase 3, 2026-11-03 to 2026-11-04:

- **The league table pages on the phone: 5, 7 or 10 a page, 5 unless the reader picks another.**
  The owner's call, iPhone only: the web keeps its 10, 25 or 50. The whole table still arrives in
  one answer; a page is a slice of it (FootMania-iOS#16).
- **A player's profile opens from the match screens too**: the scoresheet, the squads and the man
  of the match count. That goes beyond the web, whose match dialog doesn't link players (#18).
- **The language follows the iPhone's own per-app setting**, which Settings opens, rather than a
  picker in the app: iOS offers it for every app, and the app then speaks it everywhere, the
  system's text included. **The theme is chosen in the app**, System, Light or Dark, and kept on
  the device, as the web keeps it in the browser (#22).
- **API tokens stay on the web**, on purpose. The owner's plan for an Apple Watch app that records a
  match offline (kickoff, full time, each goal with its scorer, assister and time) is better served
  by the iPhone app's own login than by a token. The Watch work belongs after the TestFlight beta.
  *Corrected 2026-11-05: the login isn't handed to the Watch either. See
  [the Watch app](#phase-7-the-apple-watch-app-planned-2026-11-05).*
- **Changing your password keeps you signed in.** The backend ends every session on a change, this
  phone's too, which the web doesn't handle; the app logs straight back in with the new password
  (#22).
- **Chat has no push notifications until Phase 5**, because the backend sends Web Push only.
  The Chat tab's badge counts what's addressed to you, direct and group chats, not the Everyone
  channel, as the web's navbar does (#23).
- **Honours is a screen of its own**, opened from a row under the table and the leaderboards,
  rather than a third segment: three wouldn't fit a phone's navigation bar in every language (#25).
- **"I've paid" has no duplicate warnings yet**, because the web doesn't (FootMania-Simple-Front#223,
  brain #88). The app copies the web's rules rather than inventing its own.
- **The goal timeline needs revisiting.** It was held back because the web shows it only for a match
  recorded goal by goal. But the owner does record goals that way, with timestamps, through a
  Shortcuts and Reminders workaround, so the timeline has real data behind it. It belongs with match
  day in Phase 4.

**Phase 0 decisions, all eight, 2026-11-02.** The guide holds the reasoning for each.

- **Android:** decided after the iPhone app ships, with real usage to go on.
- **Apple account:** the owner, personally ([open question 4](#open-questions)).
- **Paid:** yes, a subscription per group paid by its group admins ([open question 1](#open-questions)).
  The billing rung stays on hold.
- **Minimum supported version:** version 1 ships the check and an update screen. It needs one small
  backend setting the app reads at launch.
- **Admin tools stay on the web:** the platform console, moderation, user administration, API
  tokens and founding a group. *Changed since: moderation, members and founding a group went native
  on the owner's word (#49, #51, #54). The platform console and API tokens stay on the web.*
- **Screens not yet native** open the existing web screen inside the app, so it has every feature
  from the first beta.
- **API code:** written by hand in the spike. Apple's OpenAPI generator is tried at the start of
  Phase 2 and kept only if its output is readable. *Tried 2026-11-03 and set aside: see
  [the generator trial](#the-openapi-generator-trial-2026-11-03).*
- **Name and identifier:** "Football Mania", `pt.rics.footballmania`. The identifier is permanent
  once published. *The name changed during Phase 6: "Football Mania" was taken on the App Store, so
  the listing is "SportsMania - Football" and the Home Screen name "SportsMania". On 2026-11-06 the
  owner extended the rename to the web and inside the app, so SportsMania is the brand everywhere
  users see it. The identifier and other stored or contract values keep their old names.*

Settled the same day while setting up the Xcode project (FootMania-iOS#2), beyond the eight:

- **Lowest supported iOS: 18.0.** Xcode's default was the latest (27.2), which would have shut out
  every iPhone not yet on it. The code needs iOS 17 at least, for `@Observable`.
- **Swift 6 language mode**, so data races are checked at compile time, switched on while the
  project was still the empty template.
- **iPhone only for version 1.** The App Store lets an update add iPad support but not take it away
  once released, and an iPad version means its own layouts and screenshots.

Settled while laying the design foundation (FootMania-iOS#4):

- **Lettering: the iPhone's system font, not the web's Geist.** The owner's call. The colours match
  the web token for token; the type does not, on purpose. The system font follows each person's
  text-size setting without extra work and is what every other app on the phone uses; Geist would
  mean bundling the font file and wiring every text style to scale with it.

Settled 2026-11-03, before the in-app web screens (Phase 2, slice 4):

- **The web gets an entry made for the app; the app doesn't write into the web's storage.** The
  owner's call. The app opens `/app/open?to=<path>` and hands over its token, group and language
  by script, never in a URL; the web signs in from the token as a reload would. Writing straight
  into the web's `sessionStorage` would have needed no web change, but that format is the web's
  private business and has changed before (`migrateStoredUser`). Installed apps can't follow a
  deploy, so the next change would have broken every installed copy's web screens until an App
  Store update. The contract lives with the code, in FootMania-Simple-Front's
  `docs/guides/native-app.md`.
- **Inside the app, the web hides its own header, footer and install hint.** The owner's call. The
  app draws its own navigation, and a second one inside the page would let people wander the whole
  website from a screen, and log out from two places.

Settled 2026-11-03, at the end of Phase 2:

- **The tab bar: Home, Matches, Rankings, Chat, More.** The owner's call, keeping the guide's
  suggestion. The web shows six entries to everyone (Dashboard, Chat, Matches, Match plans,
  Rankings, Payments) and an iPhone tab bar holds five comfortably, so upcoming plans and results
  share **Matches**. **Home** is the next match, **More** holds payments, profile, settings and
  privacy, with a Manage section for managers and group admins. Phase 3 builds its screens into
  these five.
- **The minimum build is a platform setting, and the check fails open.** `IOS_MINIMUM_BUILD`, read
  publicly at `GET /api/clients/ios` (FootMania-Back#383), compared with the app's build number,
  which is one whole number that only goes up. No answer means carry on: a check that could lock
  every installed copy out during an outage is worse than an old build running a little longer, so
  the check is never a reason to skip the release order in FootMania-iOS's `AGENTS.md`.
- **API code stays hand-written.** The owner had the generator tried, and its output, while tidy,
  describes a contract less exact than the hand-written models: every field optional, every
  failure undocumented. [The trial](#the-openapi-generator-trial-2026-11-03) has the evidence and
  what would change the answer.

**Phase 4, what managers and organisers use: done 2026-11-04**, in thirteen pull requests to
FootMania-iOS. A match can now be planned, drawn, played and recorded in the app. The live captains'
draft was left for later and went native in #52.

| PR | What it made native |
|---|---|
| #26 | Match day: goals recorded as they're scored, kept on the phone until sent, and the timeline |
| #27 | Making plans: one-off and weekly, editing, confirming, cancelling, deleting, cost and charge |
| #28 | Running the squad: answering for players, held places, guests, the team vote switch |
| #29 | Drawing teams by every method, the preview, saving, and creating the match |
| #30 | Regenerating, repairing and discarding saved teams |
| #31 | Completing a match from match day, and editing its details |
| #32 | A group admin's corrections: the sheet, the lineup, recalculating, deleting |
| #33 | A match without a plan |
| #34 | The organiser's Owed list, review queue and Record payment |
| #35 | Balances and a player's money |
| #36 | The roster, with the manager's controls |
| #37 | A group admin's part of the roster, and linking an account |
| #38 | The group's seasons |

**Phase 5, push: done 2026-11-05.** The owner joined the Apple Developer Program on 2026-11-04.
The backend half is FootMania-Back#385, shipped in 3.17.0, and the app's is FootMania-iOS#39. Permission
is asked after a first plan answer, with a screen saying why, and Settings holds the switches.
Production first refused every send with `403 BadEnvironmentKeyInToken`, because the APNs key wasn't
enabled for production. With a new key on Heroku, a notification reached the owner's phone on
2026-11-05. **Tapping a notification opens its screen** since #55.

**Phase 6, TestFlight: under way.** #40 and #41 made the app ready: the icon, production addresses in
Release builds, and the Home Screen name. **TestFlight is at build 6, as of 2026-11-06.** The owner
uploads from Xcode.

**The rest of the web, made native, 2026-11-05**, listed by the owner's order once TestFlight was up:

| PR | What it made native |
|---|---|
| #42, #44 | Your own profile, your details, and rating, ladder and head-to-head charts |
| #43 | A fix: logging in lands on Home, not on Chat |
| #45, #46 | The player of the season vote, and Waiting on you, on Home |
| #47 | Joining a group with an invite, and linking yourself to your player |
| #48 | Creating an account in the app |
| #49, #50, #51 | Members, invites and the moderation queue, for group admins |
| #52, #53 | The captains' draft and draft sessions |
| #54 | Group settings, founding a group, the forced password change |
| #55 | Notifications open their screen, and invite links open the app (with web #296) |

API tokens stay on the web, as the owner chose, and Settings is the one screen left that links to the
web, for those and the platform's settings. A review on 2026-11-06 found the plan and match links had
nothing left to offer, and the dashboard held only the Ballon d'Or result, which came over to Home.
#63 removed all three, and the owner chose to remove Upcoming's link to the plans page too, so
browsing past plans by week stays on the website.

### Phase 7, the Apple Watch app: planned and built 2026-11-05

Replaces the Reminders and Shortcuts workaround. The owner plays in the matches they record, so the
app is for a wrist, mid-game: kick-off, goals, full time and nothing else. It was added as a phase on
the owner's word, after TestFlight, as agreed. The minimum is watchOS 11, which covers the owner's
Series 7, and it ships inside the iPhone app's upload.

**Built and working on real devices.** On TestFlight build 6, the owner recorded a match on their
Series 7, reviewed it on their iPhone and sent it to production, and the app reported it done.

| PR | What it built |
|---|---|
| #57 | The Watch target, and Send to Watch |
| #58 | Recording on the Watch, with the workout |
| #59 | The hand-over to the iPhone, the review screen and sending |
| #62 | A fix: the review screen opened black on build 5, because its `.task` sat on an empty view and never ran |

**The Watch is offline and only ever talks to the iPhone.** The owner's call. It has no internet
use, no login and no token. The iPhone app is the only thing that talks to the server. A late score
doesn't matter, the owner says, because everyone who cares is on the pitch.

**This reverses two earlier answers, and the reasons matter:**

- *"The Watch borrows the iPhone's login."* Rejected once the contracts were read: the session lasts
  24 hours with no refresh, so a login handed to the Watch would die between matches.
- *A scoped API token on the Watch, calling the server itself.* Proposed and agreed the same day,
  then dropped when the owner settled that only the iPhone sends: a credential on the wrist then
  buys nothing. It also removes a real question, raised by the owner: a token revoked by logging out
  mid-match would strand the recording. With no token, logging out or an expired login only delays
  sending until the next login. A token can still be added later on the iPhone alone.

**How it works:**

- **The iPhone sends the match.** A **Send to Watch** button shows for a manager on a match with both
  teams filled, until kick-off, with a Watch paired and the app on it. It's the owner's idea, and it
  means the Watch never has to find a match. A plan has no match id, so **Create match** comes first.
  It's iPhone only for now: the web can't reach a Watch without a new path through the backend.
- **The Watch holds everything until full time.** Sending on reconnect would push goals out
  mid-match whenever the phone sits in a bag by the pitch. Holding means any goal can be fixed until
  the whistle, by a 20-second Undo after each goal (the owner's number, from experience) or from a
  list.
- **Assists go on the goal,** as `assisterId` on `POST /events`, never as `PATCH /stats/live`, which
  writes no goal row and so nothing for the timeline.
- **The iPhone reviews, then sends.** The match arrives as a card with the score and goals, and a
  **Send** button. Full time completes the match, runs the ratings and opens the vote, and can't be
  moved once recorded, so this is the last chance to catch a mistake. Delivery to a closed iPhone
  app isn't promised, and the app has to be open anyway.
- **The order is kick-off, then goals, then full time.** The clock contract allows any order, but
  ratings don't: completion reads both whistles for pace, and a goal behind full time meets a `409`.
  A refused goal stops full time from going.
- **What's waiting belongs to the account that recorded it.** Today's `GoalQueue` checks only the
  group, so somebody else logging in on that phone would send it as them.
- **A Soccer workout runs from kick-off to full time** and is saved to Fitness, which keeps the app on
  screen when the wrist comes up. The owner doesn't use Apple's Workout app, so nothing conflicts. It
  needs a HealthKit permission and a line in the privacy policy and App Store labels.

**A delete-goal endpoint: dropped by the owner on 2026-11-06.** It was planned for after the Watch,
because no path removes a goal row. The owner's reason: the Undo button stays long enough to take a
mistake back. The gap stays for a goal found wrong after sending: amending the counters leaves the
goal on the timeline, and `goalEventsAreComplete` then silently rates the match on flat weights.

### What's left

| Phase | What | Exit |
|---|---|---|
| 6 | The listing: privacy labels, screenshots, age rating, a support address; expect the first submission back once | The app is in the store |
| 7 | The Apple Watch app, above: recording works, and the owner's checks on a real match are left (taking a goal off in review, Discard, the wrist coming down, the workout in Fitness, wet buttons) | A match is recorded from the wrist, and the Reminders and Shortcuts workaround is retired |

Still open, beside the phases:

- **Sign in with Apple**: the backend's part was outlined on 2026-11-05 and not started.
- **Testing on the owner's iPhone against the Mac**: it reaches the Mac over plain `http` only once
  a Debug-only Info.plist allows local networking. Declined for now on 2026-11-04.
- **The secret man of the match ballot** (brain #104): after the vote closes, show the runner-up
  counts, or only the winner's?
- **Small cleanups.** The Results list's paging repeats Upcoming's. Home's requests could run side by
  side. `sync-strings` can't handle a plural with named placeholders. The tier colours on rungs are
  neutral until `Palette` has tokens for them.
- **Group billing stays on hold** until the owner says otherwise.

Lives in `architecture/` because it is not a frontend job. It needs an iPhone app in its own
repository (FootMania-iOS), an APNs path in the backend, the production addresses, and a set of
decisions that are neither. (It was written for a Capacitor shell in the frontend, the route not
taken.)

---

## The short answer

**Wrap the existing Next.js frontend in Capacitor. Do not fork a React Native app.** One frontend
stays one frontend, and the PWA work already done is most of the way there.

The wrapper is the easy part and takes about a day. What actually costs time is the three things
Apple demands that a PWA never has to satisfy: **native push**, **a defensible reason to exist as
an app at all**, and **the payments rule**. Budget for those, not for Capacitor.

---

## What already makes this cheap

Verified against the code on 2026-08-03, not assumed:

| Fact | Why it matters | Evidence |
|---|---|---|
| **No API routes, no middleware, no server actions** | A Capacitor app ships a *static bundle*. Any of these would make `output: 'export'` impossible and force a rewrite | `src/app/**/route.ts` → 0 files; no `middleware.ts`; no `'use server'` |
| **No dynamic server APIs** | `cookies()`, `headers()`, `searchParams`, `force-dynamic` and `revalidate` all block static export. None are used | grep across `src/app` returns nothing |
| **Data is fetched client-side** | The API is called from the browser through `apiClient.ts`, so pages do not need a server at runtime | React Query + `apiClient.ts` |
| **The PWA already exists** | Manifest, service worker, offline cache, generated icon set — all reusable | [pwa.md](../frontend/features/pwa.md), `app/manifest.ts`, `public/sw.js`, `scripts/generate-icons.mjs` |
| **iOS and standalone detection already written** | The Capacitor shell needs the same branching, and `pushSupport.ts` already has it, tested | `isIos()`, `isStandalone()`, `canUsePush()` |
| **In-app account deletion exists** | Apple **5.1.1(v)** requires it for any app with account creation. Already built for GDPR | `PrivacySettings.tsx`, `privacyService.ts`, [privacy.md](../frontend/features/privacy.md) |
| **Email/password only, no social login** | Apple **4.8** requires offering Sign in with Apple *only* if you offer other third-party logins. Not triggered | no OAuth/Keycloak/Google in the frontend |
| **A privacy policy and GDPR export/erasure exist** | Mandatory for a listing, and usually the thing that stalls a first submission | Phase 0, [PRIVACY_AND_DATA_PROTECTION.md](../backend/features/PRIVACY_AND_DATA_PROTECTION.md) |

That is an unusually good starting position. The static-export finding is the important one — it is
the difference between "wrap it" and "rebuild it", and it holds today only because nobody reached
for a server feature. **It is worth protecting deliberately**: the first API route or `cookies()`
call added to the frontend silently ends the cheap path.

---

## The four real obstacles

### 1. The app ships static; production ships SSR

Netlify builds the frontend with `@netlify/plugin-nextjs`, which runs it server-side. Capacitor has
no server — it bundles files into the `.ipa` and serves them from a local WebView origin.

Two ways out, and only one is viable:

**Build twice from one codebase.** Keep the Netlify build exactly as it is, and add a second build
with `output: 'export'` that produces the bundle Capacitor packages. Same source, same components,
two outputs.

The one snag is `/join/[token]`, the invite route. Static export needs `generateStaticParams`, and
invite tokens are issued at runtime and unguessable by design — there is no set to pre-render.
It already reads its parameter client-side via `useParams`, so the fix is a shell route rather than
a rewrite: export one page for the path and let the client read the token. Worth confirming early,
because it is on the critical path of the whole onboarding arc. *(It was the only route with this
shape when this was written; 2.2.0 added `/reset-password/[token]`, and the shell-route fix works for
both — confirmed in [the try-out](#the-rung-1-try-out-2026-10-31).)*

**Point Capacitor at the live site** (`server.url`). Rejected. It turns the app into a browser in a
costume — which is exactly what guideline 4.2 exists to reject — and it throws away the offline
behaviour the PWA already has. It also makes every release a silent server-side change to a
shipped app, which is the opposite of the release discipline in [CONTRIBUTING](../CONTRIBUTING.md).

### 2. Web Push does not work inside the app

This is the largest piece of real work, and it is mostly backend.

Today push is **VAPID Web Push**, and on iOS it only works for a PWA added to the home screen —
Safari 16.4+, documented in [push-notifications.md](../frontend/features/push-notifications.md) and
gated in `canUsePush()`. Inside a Capacitor app the page runs in a **WKWebView, which has no Push
API at all**. The existing subscription flow will not degrade gracefully; it will simply never
return a subscription.

Native push means **APNs**, via `@capacitor/push-notifications`. What that implies:

- **The backend gains a second subscription kind.** `push_subscriptions` currently stores a Web
  Push endpoint plus `p256dh`/`auth` keys. An APNs device token is one opaque string with none of
  that shape. A `type` discriminator with nullable columns is the honest modelling — a device token
  is not an endpoint and pretending otherwise will produce a table where half the columns are
  meaningless for half the rows.
- **A second sender.** Web Push is signed with VAPID; APNs needs a `.p8` key, a key ID and the team
  ID, and it speaks HTTP/2. The dispatch layer gains a branch, and the notification *content*
  should stay shared — the fan-out logic is the valuable part and should not be duplicated.
- **Both paths must keep working.** The PWA is not being retired. A user with the web app on one
  device and the store app on another has two subscriptions of different kinds and expects both to
  fire.
- **Permission timing changes.** iOS shows the system prompt once, ever. Asking on first launch
  before the user knows what the app does is the reliable way to get a permanent denial.

### 3. Guideline 4.2 — "Minimum Functionality"

Apple rejects apps that are a website in a wrapper. This is the most common rejection for exactly
this architecture, and it is a judgement call made by a human reviewer, so it is not something to
argue with after the fact.

What answers it here — all of which are real product value, not compliance theatre:

- **Native push** (obstacle 2) is the strongest single answer, and match-day notifications are the
  genuine reason someone wants this on a phone.
- **Offline** already works via the persisted React Query cache. Squad lists and fixtures readable
  on a pitch with no signal is a real capability a browser tab does not have.
- **Home-screen presence and app-switcher identity** — weak on its own, fine alongside the rest.
- Worth considering, not required: share sheet for match results, calendar integration for
  kickoff times, biometric unlock.

The submission should say plainly what the app does that the site cannot. Reviewers read that.

### 4. Guideline 3.1.1 — In-App Purchase, and the billing rung

**This is the strategic decision, and it is the one that should be made before any code.**

If the app ever unlocks features, capacity or content for money, Apple requires **In-App Purchase**
and takes its commission — and forbids linking out to your own checkout. Today this is fine:

- The match fee ledger is **bookkeeping only; no money moves through the application**, by design
  ([MATCH-FEE-LEDGER-PLAN](../backend/plans/MATCH-FEE-LEDGER-PLAN.md)). Recording what people owe
  each other is not a digital purchase.
- The **billing rung of Phase 5a is on hold by owner decision** ([STATUS.md](../STATUS.md)).

The trap is doing these in the wrong order. **If group billing ships first as a web subscription
and the app arrives afterwards, the App Store submission inherits a payment model Apple will not
accept** — and the fix at that point is either IAP with a second pricing model, or pulling the app.
Deciding "are we ever charging for this?" is cheaper now than after both exist.

---

## The route, in three rungs

Each rung is independently useful, and stopping after any of them leaves something that works.

### Rung 1 — Prove the static build, ship nothing

No Capacitor, no Apple account, no cost.

- Add a second build with `output: 'export'` alongside the Netlify build.
- Resolve `/join/[token]` as a client-resolved shell route.
- Confirm every screen works from the static bundle served as plain files.

**This rung is worth doing on its own merits even if the app is never built.** It converts
"we think we could wrap this" into a fact, and it puts a guard around the static-export property so
that a future server feature is a deliberate choice rather than an accident. It is also the only
rung that can be done without spending money.

### Rung 2 — The shell and native push

- `@capacitor/core`, `@capacitor/ios`, `@capacitor/push-notifications`; `npx cap add ios`.
- Backend: APNs sender and the subscription-type discriminator (obstacle 2).
- **Add the Capacitor origin to CORS.** In the WebView the page origin becomes
  `capacitor://localhost` — not the site's `https://` origin. `CorsConfig` uses
  `setAllowedOrigins`, an **exact-match** allowlist, together with `setAllowCredentials(true)`,
  which forbids a `*` wildcard. Production currently allows exactly one origin,
  `https://amadora-foot.netlify.app`, set through the `CORS_ALLOWED_ORIGINS` config var on Heroku.
  Without the new origin, **every authenticated request from the app fails preflight while the
  website keeps working perfectly** — the precise shape of the 2026-08-02 outage, which no test
  tier caught because CORS is enforced only by a browser. See hazard 4 in
  [STATUS.md](../STATUS.md). Add it before the first device test, not after.
- **Revisit the CSP.** `next.config.ts` sets `default-src 'self'` and derives `connect-src` from
  `NEXT_PUBLIC_API_URL`. "Self" means something different under `capacitor://localhost`, and a CSP
  that silently blocks the API produces the same symptom as the CORS failure.
- **Universal Links** so invite links open the app rather than Safari. The invite arc is the app's
  main entry point for new members and is worth getting right.

### Rung 3 — Submission

Everything here is administrative and none of it is quick the first time.

- **Apple Developer Program, $99/year**, and enrolment can take days — start it before you need it.
- **Privacy nutrition labels.** You collect email, phone numbers and device tokens; declare them.
  They must match what the app actually does, and the GDPR documentation already describes it.
- Screenshots at every required size, for iPhone and — if the app is universal — iPad.
- Age rating, support URL, marketing description.
- **TestFlight first.** Push on a real device behaves differently from push in a simulator.
- Expect the first submission to be rejected. Budget a round trip; 4.2 is the likely reason.

---

## Version and release discipline

The app is a third artifact deployed on a fourth timeline, and the existing model already covers
it: work lands on `next`, releases are cut deliberately and tagged
([CONTRIBUTING](../CONTRIBUTING.md#cutting-a-release)).

Two things change:

- **A shipped app cannot be rolled back.** Netlify redeploys in minutes and Heroku is one command,
  but a bad build in the App Store is fixed only by shipping another and waiting for review. The
  slowest artifact should set the pace of anything the three share.
- **Old app versions live in the wild indefinitely.** Users do not update. Any API change must
  stay compatible with the oldest version still installed, which is a constraint the two
  auto-updating clients have never imposed. This deserves its own decision — a minimum supported
  version with a forced-upgrade screen is the usual answer, and it is much easier to add in
  version 1 than to retrofit.

---

## The rung-1 try-out, 2026-10-31

**Run by the owner on a throwaway frontend branch, `try/capacitor-ios`, never merged.** It answers
"does the wrap still work, and what does it look like", and nothing past that. The step-by-step guide
it followed is the owner's private artifact
[Football Mania iOS Try-out](https://claude.ai/artifact/79T4XhrVQCRwYkjKjV66SS).

### The static export no longer works unchanged

The table above was verified at `1.1.0`. At `3.15.3` (`next` @ `6441043`) the export build failed
three times before it passed — each failure reproduced and each fix built, on a scratch copy, before
the owner applied them:

| Build failure | Fix on the try-out branch |
|---|---|
| `Page "/join/[token]" is missing "generateStaticParams()"` | The client page moves to `TokenRoute.tsx`; a new server `page.tsx` exports one placeholder param (`_`) and renders it. The client still reads the real token with `useParams` |
| The same error for `/reset-password/[token]` | The same fix |
| `dynamic = "force-static"` not configured on `/manifest.webmanifest` | `export const dynamic = "force-static"` in `app/manifest.ts` |

`output: "export"` sits behind `CAPACITOR_BUILD=1` in `next.config.ts`, so the Netlify build is
untouched — both builds were run. The next.config `headers()` and `redirects()` only **warn**: the
CSP, the other security headers and the `/draft-sessions` redirect simply do not exist inside the
app. Result: 28 routes, 5.1 MB in `out/`.

### What it ran on

macOS 26.6 (Apple Silicon), Xcode 27.2 (27B52), the iOS 27.2 Simulator, Node 26.13, Next.js 16.3.8,
Capacitor 8.6.0. Capacitor 8 generates a **Swift Package Manager** project (iOS 15+) — no CocoaPods.
The app's origin is `capacitor://localhost` (Capacitor's `CAPInstanceDescriptor.swift`), so the
backend ran with `--app.cors.allowed-origins=http://localhost:3000,capacitor://localhost`: the dev
profile's allowlist is hard-coded and does not read `CORS_ALLOWED_ORIGINS`. Production was not
touched.

### What the owner saw, all as predicted

- **Logged out on every cold start.** Auth persists to `sessionStorage` (`appStore.ts`), which dies
  with the app's session. A real app needs persistent storage for the token.
- **The header sits under the status bar and notch.** `viewportFit: 'cover'` is set, but nothing in
  `src/` uses `env(safe-area-inset-*)`.
- **No way to turn on notifications.** WKWebView has no Push API, so `canUsePush()` is false.
- **A reload or a 401 lands on `/login` or `/dashboard`, not where the user was.** Capacitor's iOS
  router serves the root `index.html` for every path without a file extension (`Router.swift`), and
  the root page redirects by auth state. Degrades, does not break.
- **Invite and reset links open Safari.** No Universal Links, and in the app those two routes are
  placeholders only.

### The owner's verdict: "it felt like a website in a frame"

That is guideline 4.2's rejection reason in the owner's own words, and it is what
[obstacle 3](#3-guideline-42--minimum-functionality) predicts for a bare wrap: the try-out added
nothing a browser tab lacks. Two consequences:

- **The wrap is not a submission.** As built, expect it to fail review under 4.2. Native push is
  still the strongest answer, and it is the one piece the try-out could not include.
- **"Feels like an app" is its own budget line**, separate from the about-a-day wrapper and from
  push. Part of the verdict comes from the gaps above (the notch, logging in on every launch), which
  are fixable; how much is inherent to a web UI is not yet known.

### What it did not answer

Native push (the paid program plus an APNs sender), guideline 4.2, the payments question, and the
gate itself. **Rung 2 still waits on the iOS-versus-Android push subscription numbers, and nothing
measures them yet** — `/api/admin/platform/health` reports one total `subscriptionCount` with no
platform split ([PLATFORM-CONSOLE-PLAN](../backend/plans/PLATFORM-CONSOLE-PLAN.md), push health).

### Before rung 1 counts as done

The three fixes go into `next` as a proper pull request with its changelog entry, plus a CI step that
runs `CAPACITOR_BUILD=1 next build`. That check is what "worth protecting deliberately" above asks
for: today the cheap path breaks silently, as these three failures show.

---

## The OpenAPI generator trial, 2026-11-03

**Run on the owner's Mac in a throwaway package, never added to the app.** Apple's
`swift-openapi-generator` 1.14.3 (with `swift-openapi-runtime` 1.13.3 and
`swift-openapi-urlsession` 1.4.0, Swift 6.4.2), `namingStrategy: idiomatic`, against the backend's
own `/v3/api-docs` from `next` @ `707f480`, cut down to the 8 operations the app uses and the 13
schemas they reference. Two runs: **A**, the spec as served; **B**, the same with every `*/*` media
type replaced by `application/json`.

### What came out

| | Run A | Run B |
|---|---|---|
| Builds | ✅ no generator diagnostics; one compiler warning (`role` is deprecated) | ✅ the same |
| Generated lines for 8 operations | 2,204 across 11 files | 2,204 |
| A `200` body | `.any(HTTPBody)`: **raw bytes**, decoded by hand | `.json(Components.Schemas.…)`: typed |
| Response fields | **all optional** | all optional |

B differs from A only in the six JSON responses, so **one line of backend config
(`springdoc.default-produces-media-type: application/json`) turns raw bodies into typed ones** and
changes nothing else. The rest is the spec's, and no generator setting fixes it:

- **No response field is required.** Only 43 of the full spec's 1,054 properties are, all of them
  in request bodies. So `LoginResponseDTO.token`, `MembershipSummaryDTO.groupId` and
  `ClientRequirementsDTO.minimumBuild` all come out optional, and integers come out `Int32` or
  `Int64`. The shortest call that compiles is
  `Int(try await client.ios().ok.body.json.minimumBuild ?? 0)`, where the hand-written model knows
  the field is always there.
- **Only `200` is documented, on all 182 operations.** Logout and the reset request answer `204`,
  which the generated client calls `.undocumented`, and so is every `400`, `403`, `404` and `429`.
  The app's `403` handling (`explainForbidden`) and its error messages would stay hand-written
  either way.
- **37 properties are enums elsewhere in the spec.** The generator is expected to make them closed
  Swift enums, so a value the backend adds later would fail to decode in every installed copy. The
  hand-written models read such fields as strings on purpose. None of the 8 trial operations has
  one, so this is inferred, not seen in the output.
- **Paginated lists need a `Pageable` on every call.** springdoc publishes it as a required query
  object. The generator accepted it and encodes it as form-exploded `page`, `size` and `sort`;
  whether the nested `sort` array goes out the way Spring reads it was not checked.
- **Dates were not exercised.** The backend's mix of whole-second and six-digit fractional
  timestamps will most likely need a custom `DateTranscoder`, the same job `JSONCoding.swift` does
  now.

### The verdict: hand-written, for now

The output is readable, but it describes a contract less exact than the hand-written models:
optional where the backend is certain, `200` where it says `204`, nothing at all where it refuses.
Adopting it would mean either wrapping every generated type in the hand-written one it replaces, or
fixing the spec first. **Fixing the spec is the real cost, and the risky part lands in installed
apps.** Marking about 1,000 response fields required means knowing, field by field, which ones the
backend omits when null (`non_null`), and a field marked required that is sometimes absent fails to
decode in every installed copy until the next App Store update.

**What would change the answer:** the backend publishing an exact contract for its own reasons
(required and nullable fields, real status codes, JSON media types), after which the generator is
worth a second trial on a Phase 3 surface. The media-type line is cheap and harmless on its own, and
can go in whenever someone touches the backend's OpenAPI setup; it was not made as part of this
trial.

**Costs it would also have carried:** three Apple packages and their own dependencies, which
FootMania-iOS's `AGENTS.md` lets in only after a decision recorded here; about 275 generated lines
per operation, so an estimated tens of thousands for the full spec, compiled into every build.

---

## When not to do this

The roadmap gates Phase 4 on evidence, and that gate is worth respecting:

> Don't start this speculatively — do it once you have real usage data suggesting the PWA install
> rate is low or users are explicitly asking for "the app" in a store.

The honest case against: the PWA already installs, already works offline, and already does push on
iOS once it is on the home screen. The App Store adds discovery, a trust signal, reliable push
without the install dance, and a permanent $99/year plus a review cycle on every release.

**The strongest argument for doing it is the iOS push install dance**, not the store listing.
Web push on iPhone requires users to find Share → "Add to Home Screen" unprompted — a flow
[pwa.md](../frontend/features/pwa.md) documents as undiscoverable enough to need a bespoke banner.
If the numbers show iPhone users are not getting through it, that is the trigger, and it is
measurable: compare push subscription rates between iOS and Android.

**Do not start on the strength of one person asking for it.**

---

## Open questions

Decide these before rung 2, not during:

1. **Is anything ever going to be paid?** Determines whether IAP is in scope and constrains the
   billing rung's design. The most expensive question to answer late.

   **Answered 2026-11-02: yes.** Group admins will pay a subscription per group, eventually. That is
   a direction, not a start: the [billing rung](../backend/plans/GROUP-BILLING-PLAN.md) stays ⛔ on
   hold until the owner lifts it in their own words. What Apple's App Review Guidelines say, read
   from the published text on 2026-11-02:

   - **3.1.1**: unlocking "features or functionality within your app" requires in-app purchase. A
     group's subscription that decides whether the app works for that group is exactly that.
   - **3.1.3(b)**: the app may honour a subscription bought on the website only "provided those
     items are also available as in-app purchases within the app".
   - **3.1.3(c)** covers apps sold "to organizations or groups for their employees or students", and
     **3.1.3(f)** free companions to paid "VoIP, Cloud Storage, Email Services, or Web Hosting"
     tools. Neither describes a football group, so neither exempts it.
   - **3.1.1(a)**: a link to buying elsewhere needs a regional entitlement, except on the United
     States storefront, where buttons and links to outside purchase need none. That makes
     [obstacle 4](#4-guideline-311--in-app-purchase-and-the-billing-rung)'s "forbids linking out to
     your own checkout" out of date for the US; it still describes Portugal and the rest of the EU
     closely enough, where a link-out needs an entitlement and sits beside in-app purchase.

   **Consequence for the billing design:** a group's paid status has to be an entitlement on the
   organization that more than one payment source can grant (Stripe on the web, App Store in-app
   purchase in the app), not a Stripe subscription the backend reads directly. The plan's §3
   already keeps subscription state on the organization, but its columns are Stripe's; it needs a
   second source (Apple's server notifications) designed in before the hold lifts, not after.
2. **Android at the same time?** Capacitor makes it nearly free, Play review is far gentler, and
   Android has no equivalent of the iOS push install problem — which weakens the main argument for
   doing iOS first. **Answered 2026-11-02: decided after the iPhone app ships.** The route is now
   native SwiftUI, so Android would be a separate Kotlin app rather than nearly free.
3. **Minimum supported app version**, and whether version 1 ships the forced-upgrade screen.
   **Answered 2026-11-02: version 1 ships it.**
4. **Who owns the Apple account?** It is a business identity with renewal obligations, and moving
   it later is unpleasant.

   **Answered 2026-11-02: the owner, personally**, as an individual enrollment. Expect the App
   Store to show the owner's own name as the seller, and in-app purchase revenue to be paid to
   them personally; both are worth confirming during enrollment rather than assuming. Converting
   to an organization later is the move this question warned about.
