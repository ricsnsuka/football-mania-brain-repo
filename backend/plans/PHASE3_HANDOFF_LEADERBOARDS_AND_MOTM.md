# Phase 3 Handoff — Leaderboards and MOTM Voting

**Date:** 2026-07-29
**Status:** Leaderboards & rankings, MOTM voting and badges **all done**, both repos. Remaining Phase 3: AI match reports.
**Repos:** backend `IdeaProjects/football` · frontend `FootballMania/front/football`

Written so the next session starts from a decision rather than a survey. Both features were
scoped and then deliberately not begun — a half-applied feature is worse than a clean boundary.

**Phase 3 progress so far:** balance-at-a-glance (`f8798cc`), waitlist visibility
(backend `57844eb`, frontend `3e8aa79`), leaderboards/rankings (backend `2e3018e`, frontend
`6d27431`), MOTM voting (backend `cf3240e`, frontend `139e0ed`) and badges (backend `45fff7e`,
frontend `1af7973`) are done. Remaining: AI match reports.

---

## 1. Leaderboards & Rankings — ✅ done

> **Built.** `GET /api/rankings` and `GET /api/leaderboards`, contract in
> [`docs/api/LEADERBOARDS-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/LEADERBOARDS-API-CONTRACT.md). The three
> decisions this section asked for were taken as: **all-time only** (no `seasonId` parameter at
> all, rather than one that ignores the season), **`qualified` flag at 3 matches** (everyone is
> listed; unqualified players get `rank: null` and sort last), and **Season write API deferred** to
> its own change so the V10 rollover hazard gets its own tests.
>
> The suspected eviction gap was real: `MatchEventListener` cleared `RANKINGS` but not
> `LEADERBOARDS`. Fixed, with a regression test. `PlayerService` and `PrivacyService` writes now
> evict both too — a deactivation must drop the player from the table, and an erasure must not
> leave the real name in a cached one.
>
> **Frontend done too** (`6d27431`): a `/rankings` route with the table above the four category
> cards, and a nav entry. It offers no sortable columns — sorting by rating client-side would pull
> unqualified players back to the top and undo the threshold.
>
> One thing worth carrying into the MOTM work: the backend sets
> `default-property-inclusion: non_null`, so **a null field is omitted from the JSON, not sent as
> `null`**. The first version of this contract documented `"rank": null` and would have had the
> frontend check for a value that never arrives. `jsonPath().doesNotExist()` cannot catch it —
> it passes for present-but-null as well as absent; `doesNotHaveJsonPath()` is the one that
> distinguishes them. Any nullable field in the MOTM response has the same problem.
>
> The original analysis follows, unchanged, as the record of why.

### The two endpoints are different things

The roadmap's phrasing blurs them. They are not one feature:

| Endpoint | Is | Answers |
|----------|-----|---------|
| `GET /api/rankings` | **The league table.** One ordered list of players by `skillRating`, with rank, played, W/D/L | "Where do I stand?" |
| `GET /api/leaderboards` | **Category tops.** Several short lists in one response — goals, assists, MVPs, longest streak | "Who is best at X?" |

Keep them separate: they cache differently and change at different rates.

### Almost nothing new needs computing

Everything is already on `Player`, maintained by `CalculationService` on every match
completion: `skillRating`, `totalGoals`, `totalAssists`, `totalMatchesPlayed`,
`currentStreak`, `longestStreak`. Both endpoints are ordering queries over one table, not new
aggregation.

**The exception is W/D/L**, which needs counting `PlayerStat.matchResult`. Either a projection
query grouping by player and result, or three counts. Do not add W/D/L columns to `Player` —
they are derivable, and a fourth denormalised counter is a fourth thing that can drift.

### The caches already exist

`CacheConfig.RANKINGS` and `CacheConfig.LEADERBOARDS` are declared and registered but unused —
this is what the roadmap noticed. Wire `@Cacheable` onto the read methods.

Eviction is half-built too: `MatchEventListener` already clears `RANKINGS` after every
successful recalculation. **Check whether `LEADERBOARDS` needs adding to that same evict** — it
almost certainly does, and it is the kind of omission that only shows up as "the leaderboard is
stale" days later.

### Two decisions to make before writing code

**1. Season scoping.** `GET /api/rankings?seasonId=` cannot use `Player`'s totals — those are
all-time. A season-scoped ranking has to go through `skill_rating_history` joined to
`match.season_id`. All-time-only is a legitimate v1; just do not add a `seasonId` parameter
that silently ignores the season.

**2. Whether to close the Season API at the same time.** The roadmap says do both together, and
`docs/features/SEASON_FEATURE.md` still lists `GET /api/seasons`, `POST /api/seasons`,
`PATCH /api/seasons/{id}/end` and `GET /api/seasons/{id}/leaderboard` as planned.

> ⚠️ If you build the season write API: **V10 enforces one current season** via a partial
> unique index. A rollover must clear the old `is_current` and set the new one **in one
> transaction**, or it deadlocks against its own constraint. Already documented in
> `SEASON_FEATURE.md`.

### The trap

Ranking by `skillRating` puts a player with **one match** at the top. Decide a
minimum-appearances threshold — or a separate `qualified` flag — **up front**. Retrofitting it
changes everyone's rank and looks like a bug rather than a fix.

---

## 2. MOTM (Man of the Match) Voting — ✅ done

> **Built.** `GET`/`POST /api/matches/{id}/mvp-vote`, migration V14, hourly resolution job,
> `MVP_VOTE_OPEN` notification, GDPR in all three places. Contract in
> [`docs/api/MOTM-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/MOTM-API-CONTRACT.md).
>
> The three decisions this section left open were taken as: **ties produce no crowd MVP** (the match
> is still marked resolved, so a tie is a final answer rather than a poll left hanging);
> **self-voting is refused** with a 400 and a CHECK constraint; **only the vote prompt is
> notified**, with no "result is in" follow-up.
>
> Two things this section did not anticipate:
>
> 1. **`Match` had no completion timestamp.** `matchDate` can be backdated and `updatedAt` moves on
>    any edit, so neither could anchor a 24h window. V14 adds `mvp_voting_closes_at`, set at
>    completion and only if unset — so amending a match never reopens a counted poll.
> 2. **The vote is personal data about *two* people**, which the export has to treat asymmetrically.
>    Votes cast name the player chosen; votes received are a **count only**. Naming the voters would
>    disclose their votes — their data, not the subject's — and would let a winner unmask the ballot
>    with an export request.
>
> **Frontend done too** (`139e0ed`): a panel in the completed-match modal with the ballot, the
> running tally and the outcome. Every branch keys off `resolved`/`tied`/`canVote` rather than the
> ids, because nullable fields are *omitted*, not `null` — the same trap the leaderboards hit, this
> time pinned by a schema test from the start. A tie renders as its own outcome, and
> closed-but-uncounted is distinguished from decided, since the resolver only runs hourly.
>
> The original analysis follows, unchanged, as the record of why.

### Keep the crowd result separate from `isMvp`

> **Superseded in part, 2026-08-08 — read this before acting on the section below.**
> *(The reversing change is written and tested but **not yet released**; see
> [RETIRE-ORGANISER-MVP-PICK-PLAN.md](RETIRE-ORGANISER-MVP-PICK-PLAN.md) §1.)*
>
> The *storage* decision stands and is still right: two columns, two authorities, and the analysis
> below is why. What was reversed is its last consequence — **which column the counts read.**
>
> The owner settled it: Man of the Match is decided by the crowd, not by an algorithm or an admin
> decision. `mostMvps` and `FIRST_MVP` now count `crowd_mvp_player_id`. `is_mvp` counts only on
> matches that never ran a poll, so the board keeps the years that predate the vote; a tie counts
> for nobody and does not fall back to the organiser's pick.
>
> See `LEADERBOARDS-API-CONTRACT.md`, `MOTM-API-CONTRACT.md` and `BADGES-API-CONTRACT.md` in the
> backend repo for the rule as it now stands, and
> [RETIRE-ORGANISER-MVP-PICK-PLAN.md](RETIRE-ORGANISER-MVP-PICK-PLAN.md) for the control this left
> stranded on the match form.
>
> The original analysis follows unchanged, as the record of why the columns are separate.

The roadmap raises this and the answer is: **do not merge**.

`isMvp` lives on `PlayerStat` and is admin intent. A crowd vote is a different fact with
different authority. Conflating them means you can never show *"the admin picked X, the players
picked Y"*, and you cannot undo one without destroying the other.

Add `crowd_mvp_player_id` to `matches`, alongside the existing `isMvp`.

### Schema

```sql
CREATE TABLE match_mvp_votes (
    id                  BIGSERIAL PRIMARY KEY,
    match_id            BIGINT NOT NULL REFERENCES matches(id) ON DELETE CASCADE,
    voter_player_id     BIGINT NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    voted_for_player_id BIGINT NOT NULL REFERENCES players(id) ON DELETE CASCADE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_match_mvp_votes_voter UNIQUE (match_id, voter_player_id)
);
```

**The UNIQUE constraint *is* the one-vote rule.** Enforce it in the database, not with a
check-then-insert, which races.

### Who may vote

**Only players who appeared in that match** — a `player_stats` lookup. This is not just
correctness: it is what stops the vote being brigadable.

### The window

24h post-match, per the roadmap. The machinery already exists: `ReminderScheduler` runs hourly
with a conditional-claim guard (`SET x = now WHERE id = ? AND x IS NULL`). Resolution is
another job in exactly that shape — reuse the claim pattern.

> **Do not resolve on read.** Two readers would resolve independently and can disagree. The
> resolution must be a single claimed write, like the reminders.

Remember `ReminderDispatcher` is a **separate bean** from `ReminderScheduler` because
`@Transactional` is proxy-based and a same-bean call bypasses it. Any new claim-and-act job
needs the same split.

### Ties

Decide up front: earliest vote, highest rating, or no crowd MVP. Silent arbitrary resolution is
exactly the thing people notice and lose trust over.

### Notifications

`NotificationCategory` is the right home for a "vote for MVP" prompt. Adding a category is free
by design — `notification_mutes` stores **opt-outs**, so a new category ships enabled with no
backfill.

### GDPR

`match_mvp_votes` holds personal data (who voted, and for whom). It cascades from `players`,
but erasure **anonymises rather than deletes**, so votes survive with a now-anonymous voter —
which is correct and worth stating explicitly.

Per `PRIVACY_AND_DATA_PROTECTION.md`, three places must be updated when a personal-data table
is added, and missing any one makes the export or erasure quietly incomplete:

1. The data table at the top of that document
2. `PrivacyService` — a section in the export, a step in `erase(...)`
3. `PrivacyServiceTest` — a case proving both

---

## Conventions worth carrying over

Patterns established this phase that will bite if forgotten:

- **API contract per change.** Any endpoint, DTO, status code or auth-rule change ships with
  `docs/api/<FEATURE>-API-CONTRACT.md` updated in the same commit — the frontend is a separate
  repo and the docs are the only interface. See `PUSH-API-CONTRACT.md` for the house format.
- **Self-invocation kills `@Async` and `@Transactional`.** Both are proxy-based. Calling one
  annotated method from another method of the same bean silently drops the annotation. This bit
  twice already (`PushNotificationService.notifyPlayers`, `ReminderScheduler`).
- **`useSyncExternalStore` for anything reading `navigator`/`window`** on the frontend. The
  React Compiler lint rejects `setState` in an effect body, and a distinct server snapshot is
  what prevents hydration mismatches. Used three times now.
- **Read `node_modules/next/dist/docs/` before writing Next.js code**, per the frontend's
  `AGENTS.md`. It has caught two things that were wrong from memory.
- **CI test tasks are package-filtered.** A test in a new package runs locally and is silently
  skipped in CI. `build.gradle` documents the invariant and how to re-check the counts.

---

## Still outstanding, independent of both features

| Item | Notes |
|------|-------|
| **Privacy policy operator details** | `src/app/privacy/page.tsx` says `[operator email]`. Must be real before any public launch or store submission |
| ~~**No push notification has ever been observed arriving**~~ ✅ **Resolved 2026-07-29** | Delivered end to end 925 ms after a match completion. Took four things at once, three of which were wrong: live VAPID keys; a **production** frontend build (`next dev` registers no service worker, so subscribing hangs forever at `navigator.serviceWorker.ready` with no error at all); Brave's *"Use Google services for push messaging"* enabled (off by default → `AbortError` at `pushManager.subscribe`); and a **fix to the `aud` claim** (`936d380`) — JJWT serialised it as a one-element array, which FCM rejects with `403 invalid aud claim`. Full account in `PUSH-API-CONTRACT.md` § *Delivery has been observed*. **`push_subscriptions.last_used_at` is the check** — written only on a 2xx from the push service |
| **Log retention policy** | Erasure cannot reach application logs, which carry resolved usernames. Retention is the only backstop and is currently the platform default |
| **Admins cannot create user accounts** | `POST /api/users` exists and is unused; there is no create-user UI. Self-registration is the only route to an account. Deliberate or not is a product call |
| **Waitlist standing is modal-only** | Shown in the match-plan detail modal, not the plans list — which may be where a player looks first |
| **`STREAK_AWARE`** | Declared in `GenerationType`, never implemented; `strategyFactory.resolve` throws if requested |
