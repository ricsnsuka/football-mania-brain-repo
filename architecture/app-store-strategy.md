# Putting FootMania on the Apple App Store

**Status: DRAFT — nothing here is implemented, and starting it is not yet authorised.** Written
2026-08-03 against backend `2981bd3` and frontend `ac831ee`, both at `1.1.0`. This is roadmap
[Phase 4](../product/roadmap.md), which is deliberately gated on real usage data rather than on
enthusiasm. Read [When not to do this](#when-not-to-do-this) before the rest.

Lives in `architecture/` because it is not a frontend job. It needs a Capacitor shell in the
frontend, an APNs path in the backend, one config change in production, and a set of decisions that
are neither.

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
because it is the only route in the app with this shape and it is on the critical path of the whole
onboarding arc.

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
2. **Android at the same time?** Capacitor makes it nearly free, Play review is far gentler, and
   Android has no equivalent of the iOS push install problem — which weakens the main argument for
   doing iOS first.
3. **Minimum supported app version**, and whether version 1 ships the forced-upgrade screen.
4. **Who owns the Apple account?** It is a business identity with renewal obligations, and moving
   it later is unpleasant.
