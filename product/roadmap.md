# Football App — Improvement & Mobile Roadmap

Prepared July 2026 · sequenced so each phase is shippable on its own and builds toward a mobile\-first, Capo\-competitive product.

* * *

## How to read this

Five phases, roughly in the order I'd tackle them. Phases 1–3 you can start immediately with zero new infrastructure. Phase 4 (app stores) and Phase 5 (multi\-tenancy/commercialization) are bigger decisions — I've flagged exactly what triggers each one, so you're not doing them speculatively.

Each feature item notes: what you already have that makes it cheap, what's actually new, and any gotcha to watch for.

* * *

## Phase 0 — Before you write any code

A short list of decisions and cleanup that will bite you later if skipped now.

| Item | Why it matters now |
| --- | --- |
| **Replace placeholder assets** | `public/` still has the default `file.svg`, `globe.svg`, `next.svg`, `vercel.svg` from `create-next-app`, and there's no favicon or app icon set. Every phase below (PWA manifest, app store listing, push notification icon) needs real branding. Do this first — it's a blocker for everything else, not just polish. |
| **Fix the `is_current` season gap** | Your own docs flag that the DB doesn't enforce a single current season. Low risk today at your scale, but do it before Phase 5 (multi\-tenant) multiplies the blast radius. |
| **Decide on GDPR posture now, not later** | You store phone numbers and emails, target `en`/`pt`/`es` locales (EU \+ Iberia/LatAm), and are about to add push subscriptions (device tokens) and possibly payments. That's enough personal data that you should have a privacy policy and a basic data\-export/delete path before any public signup — required for App Store/Play Store listings too (Phase 4), not optional. |
| **Pick your LLM vendor now if you want AI match reports (Phase 3)** | Not urgent to build, but worth deciding early since it affects the async job you'll wire in Phase 3. |

* * *

## Phase 1 — PWA groundwork (the mobile foundation)

**Goal:** installable app icon, works offline for cached views, no app\-store dependency yet. This is the cheapest possible "go mobile" step and reuses 100% of your existing Next.js code.

- Add `manifest.json` (name, icons in multiple sizes, theme color, `display: standalone`).
- Add a service worker — either hand\-rolled or via `next-pwa`/`@ducanh2912/next-pwa` (check current compatibility with Next.js 16's App Router; some PWA plugins lag behind major Next releases — confirm before committing).
- Cache\-first strategy for static assets, network\-first for API calls (you don't want stale player ratings served offline).
- Add an offline fallback page for when the API is unreachable.
- Run a Lighthouse PWA audit and fix whatever it flags (installability, viewport meta, etc.).

**Gotcha to be aware of:** iOS Safari's "Add to Home Screen" doesn't fire the `beforeinstallprompt` event the way Android/Chrome does. You'll need an iOS\-specific "tap Share → Add to Home Screen" hint UI, or users on iPhone (likely a big chunk of your base) won't discover installability at all.

**Effort:** days, not weeks. Ship this before anything else mobile\-related.

* * *

## Phase 2 — Push notifications (highest\-leverage single feature)

This is the one thing every competitor (Spond, Heja, TeamStats, Capo) has that you don't, and it's the natural next step given the SSE infrastructure you already built for the draft.

**Backend additions (all fit inside the existing monolith — no new services):**

- New `push_subscriptions` table (endpoint, keys, player/user FK).
- `POST /api/push/subscribe`, `DELETE /api/push/subscribe` endpoints.
- Use the raw Web Push protocol with VAPID keys (`web-push` library equivalent in Java) rather than Firebase Cloud Messaging — keeps you infra\-light and vendor\-neutral, consistent with your "no Redis, no Kafka" philosophy.
- Wire notification triggers into code paths you already have:
  - `MatchEventListener` → match completed / rating updated.
  - `DraftSessionService` → "it's your pick" on `PICK_MADE`, "draft complete" on `SESSION_COMPLETED`. This reuses the exact event points your SSE stream already emits.
  - A new lightweight `@Scheduled` job (Spring, no new infra) for availability\-poll deadline reminders and pre\-match reminders.

**Gotchas:**

- iOS push only works if the PWA has actually been *installed* to the home screen (Safari 16.4\+) — another reason Phase 1's install\-prompt UX matters, it's not just nice\-to\-have.
- Don't over\-notify. Every competitor review I found praised apps that use notifications sparingly (deadline, your turn, match reminder) and complained about noisy ones. Keep the trigger list short and let users mute per\-category from day one — retrofitting notification preferences is annoying.

* * *

## Phase 3 — The "steal from Capo" feature set

Here's what stood out from Capo, mapped against what you already have, so you can see exactly what's genuinely new work versus what's mostly a UI layer on data you already compute.

| Capo feature | What you already have | What's actually new |
| --- | --- | --- |
| **"Balance at a glance" score** | You already compute `avgA`, `avgB`, `Δ` in `generationNotes` for every algorithm | Pure frontend — a visual gauge/bar instead of a text string. Zero backend change. Do this first in this phase, it's nearly free. |
| **League tables / leaderboards** | Cache slots literally named `rankings` and `leaderboards` are already reserved in `CacheConfig` but unused | Build `GET /api/rankings` and `GET /api/leaderboards`. This also closes the gap your own docs flag under "Planned Improvements" for the Season API — do both together. |
| **Fantasy\-style stats & achievements/badges** | `skillRating`, `currentStreak`, `longestStreak`, `totalGoals`, `totalAssists`, `totalMatchesPlayed` all already exist | A badge\-rules table/enum \+ a check run inside `MatchEventListener` after each completed match. No new service — same event hook you already use for rating recalculation. |
| **MOTM (Man of the Match) voting** | `isMvp` exists but is admin\-set only | Add a peer\-voting window (e.g., open for 24h post\-match), one small `match_mvp_votes` table, simple majority resolves a "crowd MVP" — either merge into `isMvp` or keep as a separate field so admin intent and crowd result aren't conflated. |
| **Automatic waitlist on availability polls** | `player_confirmations` already models `CONFIRMED`/`DECLINED`/`PENDING` | Add a capacity cap to `MatchPlan` and auto\-promote logic in `MatchPlanService` when a confirmed player declines. Real business logic, not infra — a few days of focused work. |
| **Tiered invitations (core vs. extended players)** | Nothing yet | Smallest version: an `isCore` flag on `Player`, filter who gets invited/notified first. Note this starts to overlap with multi\-tenancy thinking (Phase 5) if you ever want "groups within a group" — don't over\-build this before deciding on Phase 5. |
| **AI\-written match reports / player profiles** | `MatchEventListener` already has an async hook (ADR\-002's pattern) | One outbound call to an LLM API, async, non\-blocking, store the result in a new `match_report` text column. This is the one item in this list that adds an external dependency and a recurring cost — budget for it and rate\-limit it before enabling for everyone. |
| **In\-app payments (organizer free, players pay a small fee)** | Nothing yet | This is not a "steal the feature" afternoon — it's a Stripe Connect integration with real compliance overhead (money movement, payouts, KYC). Treat as its own project, sequenced after Phase 5, not a Phase 3 item. Listed here only so you don't underestimate it when comparing yourself to Capo. |

**Recommended order inside this phase:** balance\-at\-a\-glance UI → leaderboards/rankings API → MOTM voting → waitlist logic → achievements/badges → AI match reports last, since it's the only one with an ongoing external cost.

* * *

## Phase 4 — App store presence (only once Phase 1–2 prove out)

Don't start this speculatively — do it once you have real usage data suggesting the PWA install rate is low or users are explicitly asking for "the app" in a store.

- Wrap the existing Next.js frontend with **Capacitor** rather than forking a separate React Native codebase — you keep one frontend to maintain.
- Add native push via Capacitor's push plugin (APNs/FCM), reusing the same `push_subscriptions` table with a device\-token variant alongside the Web Push endpoints.
- Store listing requirements to budget for: Apple Developer account ($99/yr), Google Play one\\\-time fee ($25), app icons/screenshots at required sizes, and — because you collect phone numbers and emails — a public privacy policy page is mandatory for both stores, not optional paperwork.

* * *

## Phase 5 — Commercialization / multi\-tenancy (parallel track)

This isn't mobile\-specific, but it's the thing that determines whether any of the above is a product or a personal tool, so it needs its own line here.

Right now there's no organization/tenant concept anywhere in the data model — one deployment serves one group's players and matches. Before any self\-serve public launch (and definitely before Phase 4's app\-store distribution, which implies multiple independent groups installing the same app), you need:

- A tenant/organization boundary threaded through the schema.
- Self\-serve signup (today there's a seeded admin credential, not a signup flow).
- Billing (Stripe subscriptions, separate from the Phase 3 in\-app match\-fee payments — different problem, don't conflate them).

**Sequencing recommendation:** do this after Phase 2 (push) validates real engagement, and before Phase 4 (app stores) since store distribution implies you're ready for strangers to sign up, not just your own group.

* * *

## Suggested overall order

1. Phase 0 cleanup (branding, season bug, privacy posture decision)
2. Phase 1 — PWA
3. Phase 2 — Push notifications
4. Phase 3 — Capo\-inspired features (balance UI → leaderboards → MOTM → waitlist → achievements → AI reports)
5. Phase 5 — Multi\-tenancy \+ billing (gate on real usage signal from Phase 2)
6. Phase 4 — Capacitor wrap \+ app stores

Phases 3 and 5 can run partly in parallel if you have the bandwidth — they don't block each other technically, only Phase 4 depends on both being reasonably mature.
