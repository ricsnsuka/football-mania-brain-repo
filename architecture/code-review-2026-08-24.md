# Code review — 2026-08-24

**Status:** complete. Findings recorded, none fixed — every entry below is open work.
**Scope:** both code repos at `main` — backend `7f015ca`, frontend `097916a`, i.e. `2.0.0` as
deployed and confirmed on 2026-08-23.
**Read alongside:** [STATUS.md](../STATUS.md) — the live hazards there are *not* repeated here, and
nothing below contradicts one.

This is a whole-codebase read rather than a diff review: 375 Java files and 324 TypeScript files,
with depth spent where a defect is expensive — the tenancy boundary, money, the two SSE hooks, and
the newest feature (`2.0.0` group chat, which has had the least time to be wrong in public).

---

## What is in good shape, said once so the rest can be short

Three things were checked hard and came back clean, and they are worth naming because the findings
below are all *narrow* — they are narrow because the structure around them holds.

**Tenant scoping is explicit everywhere and there is no global Hibernate filter.** That is the
harder discipline and the right one: `grep` for `@Filter` returns nothing, so every scoped read
says so at the call site instead of relying on ambient state a future `runAs` could quietly widen.
There are ten `@Cacheable` methods; **nine carry the tenant in the key** and the tenth is
`PlatformSettingsService`, whose values are platform-level and correctly are not scoped. Every
controller method with no `@PreAuthorize` is one of the six anonymous endpoints the filter chain
permits — the five in `PUBLIC_PATHS` that have a controller, plus the `GET /api/invites/*` preview
its own matcher allows — with no exceptions in either direction.

**The public/authenticated split is enforced twice** — once in `SecurityConfig`'s allow-list and
once per method — and the two agree exactly. `/api/auth/login` being listed literally rather than as
`/api/auth/**` is the kind of decision that pays back the day somebody adds password reset.

**Locale key parity is currently perfect**, which closes half of [suggested next step
6](../STATUS.md#suggested-next-steps) with evidence rather than hope. All three locale files hold
**1390 keys** and the sets are identical — no key present in one and missing from another, in either
direction. Every value of `NotificationCategory` (9), `AppSetting` (6), `Role` (3), `Badge` (9),
`MatchType` (3), `ConfirmationStatus` (3), `SeasonAwardType` (7), `ChatConversationKind` (3) and
`ApiTokenScope` (3) resolves to a key in `en`, `pt` **and** `es`.

That is not an argument against writing the parity test — it is the argument *for* writing it. The
one enum-derived gap this review did find (finding 7 below) was found by doing by hand exactly what
that test would do automatically, and it is in the one family nobody thought to check.

---

## Findings

Ordered by what it would cost to be wrong about them, not by how interesting they are.

| # | Where | What | Severity |
|---|---|---|---|
| 1 | backend | `TenantContext` is not cleared for three request paths | Medium — latent |
| 2 | backend | A rule stated once in `EveryoneChannelProvisioner` is broken in three other services | Medium |
| 3 | backend | A group-less account gets a **500** from every tenant-scoped endpoint | Medium |
| 4 | frontend | `matchDTOSchema` rejects every manually-recorded match | Medium |
| 5 | frontend | `useDraftSessionSSE` leaks a self-perpetuating reconnect loop past unmount | Medium |
| 6 | backend | `page` is the one unclamped paging parameter, and a negative one is a 500 | Low |
| 7 | both | `GenerationType.MANUAL` has no i18n key, and the TS union spells it lowercase | Low |
| 8 | backend | `parseGenerationType`'s "valid values" list has drifted from the enum | Low |

---

### 1. `TenantContext` is not cleared for `/actuator`, `/v3/api-docs` and `/swagger-ui`

**`HttpRequestLoggingFilter.java:52-58`** — the skip loop returns *before* the `try/finally` that
owns the clear.

```java
for (String prefix : SKIP_PREFIXES) {
    if (path.startsWith(prefix)) {
        filterChain.doFilter(request, response);
        return;                              // ← never reaches the finally below
    }
}
```

The comment in that `finally` states the invariant this breaks, in the file that breaks it:

> This filter owns the clear because it is the outermost one: it still runs when the security chain
> rejects the request, and when a handler throws.

It is the outermost one. It does not always run. In production `app.security.public-api-docs` is
`false`, so `/v3/api-docs` is *authenticated* — which means `JwtAuthenticationFilter` binds a real
tenant to that request and nothing ever unbinds it. The container thread goes back to the pool
carrying somebody's group.

**Today this is latent, and the reason is worth writing down because it is not the reason you would
guess.** It is not that the skipped paths are harmless. It is that `TenantContext.set(null)` calls
`ThreadLocal.remove()`, so *every* JWT-authenticated request re-binds correctly regardless of what
the thread was carrying. The stale value therefore only survives into a request that binds nothing —
which means an anonymous one, which means one of the six public paths. Those were each checked:
`registerUser` deliberately writes no tenant-owned rows (and says so in a comment), and
`InviteService.preview` reads by token without touching `TenantContext` at all.

**So the exposure is entirely in the future, and it has a specific shape.** `TenantStamping` fills
`tenant_id` from `currentTenant()` on any `TenantOwned` entity persisted with a null tenant. The
listener exists precisely so creation sites do not have to remember — which means the day a public
endpoint persists a tenant-owned row, it will stamp whatever the previous request left on the
thread, silently and correctly-looking, instead of throwing the `IllegalStateException` that the
whole design is built to produce.

**Live today, minor:** `MDC.clear()` is skipped too, so log lines emitted while serving `/actuator`
carry the *previous* request's `requestId`, `user` and `tenantId`. The audit trail says the wrong
thing rather than nothing.

**Fix:** move the skip decision inside the `try`, or wrap the early return in its own
`try { … } finally { TenantContext.clear(); MDC.clear(); }`. Three lines. A test that asserts
`TenantContext.isBound()` is false after a filter pass over `/actuator/health` would pin it.

---

### 2. Catching `DataIntegrityViolationException` and carrying on inside the same transaction

The codebase already knows this rule, states it better than most textbooks, and states it in
exactly one place — **`EveryoneChannelProvisioner.java`**:

> That re-read cannot happen in the transaction that hit the constraint — a
> `DataIntegrityViolationException` marks it rollback-only, so every subsequent query in it fails
> too. `Propagation.REQUIRES_NEW` gives the insert its own transaction, so a violation leaves the
> caller's intact.

That class does it right, including the part everyone gets wrong (a separate bean, because a
self-invoked `@Transactional` method is not proxied). Four other places catch the same exception.
**One of them is correct and three are not.**

| Site | Behaviour after the catch | Verdict |
|---|---|---|
| `ChatService.everyoneChannel` | `REQUIRES_NEW` in a separate bean, then re-reads | ✅ correct |
| `MvpVoteService.java:101` | rethrows as a 409 — the transaction rolls back, which is what it wants | ✅ correct |
| `MatchFeeService.generateChargesFor:124` | **continues the loop**, then publishes notifications | ❌ |
| `BadgeService.awardForMatch:94` | **continues the loop** | ❌ |
| `ChatPresenceService.heartbeat:76` | returns, and the method commits | ❌ |

Both entities are `GenerationType.IDENTITY`, so `save()` issues the INSERT immediately and the
violation lands inside the `try` rather than at commit. On PostgreSQL the transaction is aborted
from that statement onward: every later `save()` in the loop fails with `25P02`, and the commit
raises `UnexpectedRollbackException`.

**`MatchFeeService` is the one that matters**, because its javadoc promises the opposite of what
happens:

> **Re-running is safe.** `uq_player_charges_plan` is the guarantee and the pre-check below is only
> an optimisation: two concurrent calls can both pass it, and the loser's
> `DataIntegrityViolationException` is swallowed because the player ends up charged exactly once
> either way.

The player does not end up charged exactly once. **Nobody ends up charged at all**, and there are two
ways to get there depending on where in the roster the duplicate falls:

- **Duplicate mid-loop** — the *next* `save()` fails against the aborted transaction with something
  that is not a `DataIntegrityViolationException`, so nothing catches it, it propagates out of
  `generateChargesFor`, and the whole batch rolls back. A 500, no charges, no notifications.
- **Duplicate on the last player** — the loop ends normally and `notifyCharged` runs. It calls
  `PushNotificationService.notifyPlayer`, which is `@Async` and therefore dispatched on a thread
  that never touches this transaction. So **the `FEE_CHARGED` pushes go out**, and then the commit
  raises `UnexpectedRollbackException` and rolls the charges back behind them.

The second is the expensive one: players are told what they owe for a debt that does not exist, and
the ledger and the notifications disagree with no record of why.

`BadgeService` has the same shape and a genuinely reachable race: `MatchEventListener` (async, after
match completion) and `MvpResolutionScheduler` (hourly) both call `awardForMatch(matchId)` for the
same match, and `FIRST_MVP` is awarded on the second path.

`ChatPresenceService.heartbeat` is the mildest — the method returns immediately, so only the commit
fails. From the SSE keep-alive that is swallowed at `debug` and the heartbeat is silently lost; from
`POST /api/chat/presence/heartbeat` it is a 500.

**Fix:** the pattern is already written. Push each insert into a `REQUIRES_NEW` method on a separate
bean, as `EveryoneChannelProvisioner` does — or, better for the two loops, drop the read-then-insert
entirely and use `INSERT … ON CONFLICT DO NOTHING`, which needs no exception at all. Whichever is
chosen, the reasoning belongs where the rule is already written down, not restated per site.

---

### 3. A group-less account gets a 500 from every tenant-scoped endpoint

Two components hold defensible positions that do not compose.

`TenantResolver.resolve` returns `null` for an account with no membership and calls it, correctly,
*a legitimate state* — "they authenticate and hold nothing … so platform-level endpoints still
answer while **tenant-scoped ones refuse**".

`TenantContext.currentTenant()` throws when unbound, and calls that, also correctly, a programming
error — "an unbound read is a programming error — most likely async or scheduled work that forgot to
declare its tenant".

Nothing sits between them. So "refuse", the stated intent, is delivered as `IllegalStateException`
→ no `@ExceptionHandler` matches → `handleGeneric` → **HTTP 500**, across **100 `currentTenant()`
call sites in 21 services**. `GET /api/chat/presence` is the shortest reproduction: line one of
`ChatPresenceService.roster()`.

**This is the 2026-08-04 incident's second half, and the fix that shipped only closed the first.**
[`INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts`](../backend/fixes/INCIDENT_2026-08-04_Users_Me_404_For_Groupless_Accounts.md)
found exactly this on `UserService` — the `@Cacheable` key called `currentTenant()` and blew up
before the method body ran — and fixed it there by moving to `currentTenantOrNull()`. The
accompanying rule, "an account is either a platform operator or a member of groups, never both"
(`V35`), closes the *operator* case for good. It does not close the ordinary one:
`POST /api/users/register` deliberately creates an account with no membership, and that account can
log in and hold nothing until it accepts an invite or founds a group.

`PlatformOperatorAccountIT`'s own javadoc predicted this: *"a fix without the rule just waits for
the next endpoint."* The rule it adopted was narrower than the bug.

The UI does not expose it — `ChatPage` returns early on `activeGroupId === null`, and `AuthGuard`
routes group-less users to onboarding — so this is an API-contract defect, not a user-visible one.
That is exactly the state the `X-Group-Id` CORS outage was in the day before it was found.

**Fix:** one `@ExceptionHandler(IllegalStateException.class)` is the wrong answer — it would turn
genuine async-forgot-its-tenant bugs into quiet 4xx and lose the stack trace the design wants.
Better: refuse at the boundary. A group-less caller reaching a path that is neither public nor
tenant-agnostic can be answered `409` (or `403`) by the filter chain, naming the state, before any
service reads the context. That preserves the loud failure for the case it was built for.

---

### 4. `matchDTOSchema` rejects every manually-recorded match

Backend, `MatchMapper.java:29-30`:

```java
@Mapping(target = "generationType", expression = "java(match.getGenerationType().name())")
```

`name()` on `GenerationType.MANUAL` is `"MANUAL"`. Frontend, `src/types/match.ts:122-130`, accepts:

```ts
generationType: z.enum(['BALANCED','RANDOM','SNAKE_DRAFT','FORM_BASED','CAPTAIN_PICK','OPTIMAL','manual'])
```

Lowercase. Every match created through `POST /api/matches/manual` — which
`MatchService.createManualMatch` forces to `GenerationType.MANUAL` — fails validation.

**The blast radius is exactly one call**, and it is worth saying so rather than leaving it to be
assumed: `matchDTOSchema` is passed to `apiFetch` in one place, `matchService.ts:40`
(`GET /api/matches/{id}`). The list endpoints send no schema, so they are unaffected. What this
costs is one screen — opening a hand-recorded match's detail modal — and a console error nobody is
reading.

**The consequence is not the failure, it is what the failure does.** `apiFetch` fails open by
design, and the reasoning is sound:

> Fail open in BOTH environments: a schema/backend disagreement about one field must never brick a
> screen.

But failing open returns `json as T` — the **raw** response, not `result.data` — so the schema's
transforms are skipped for the *entire object*, not just the offending field. `nullableField` is
`schema.nullish().transform(v => v ?? null)`, and `application.yml:58` sets
`default-property-inclusion: non_null`, so the backend omits null fields altogether. The net effect
on a manual match is that `location`, `scoreTeamA`, `scoreTeamB`, `finalScore`, `winningTeamId`,
`generationNotes`, `seasonId` and `version` arrive as **`undefined` where the type says `null`**.

Any `=== null` check, any `in` test, any `Object.keys` walk over that object is now wrong, and
TypeScript cannot see it because the declared type still says `null`. The fail-open behaviour is
right; what makes this bite is that its blast radius is the whole payload while the drift is one
field.

**Fix:** change `'manual'` to `'MANUAL'` in **two** places — the Zod enum at `match.ts:122` and the
hand-written `GenerationType` union at `match.ts:5-12`, which carries the same lowercase spelling
and is what makes the mistake type-check. While there, `'STREAK_AWARE'` is absent from both —
harmless today because `StreakAwareGenerationStrategy` throws rather than persisting anything, but
it is the same omission waiting for the day that strategy ships.

**Worth considering separately:** make the fail-open path narrower, so a single unknown enum member
does not discard normalisation for every other field.

---

### 5. `useDraftSessionSSE` leaks a self-perpetuating reconnect loop past unmount

`src/hooks/draft/useDraft.ts:173` and `:208`:

```ts
setTimeout(() => connectRef.current(), 3_000);
```

Neither call checks whether the hook is still alive. The cleanup aborts the current controller and
nothing else:

```ts
return () => { abortRef.current?.abort(); };
```

The `catch` branch is safe by accident — an abort rejects the pending read with `AbortError` and the
handler skips it. **The `done` branch is not.** Spring closes a draft SSE stream on its own timeout,
which is the ordinary end of every long-lived draft connection, and that path schedules the
reconnect unguarded. Unmount within those three seconds and the timer still fires: it calls
`connect()`, which opens a **new** `fetch` on a dead component, writes `setError` / `setConnected` /
`setSession` after unmount, and holds a stream that will close and reschedule again. Navigating away
from a draft at the wrong moment leaves a connection reopening itself for as long as the tab lives.

**The fix already exists in this repo, one directory over.** `useChatStream` — whose header says it
is *"Modelled on `useDraftSessionSSE`"* — was written with the guard on both ends:

```ts
function scheduleReconnect(controller: AbortController) {
  if (controller.signal.aborted) return;
  setTimeout(() => { if (!controller.signal.aborted) connectRef.current(); }, RECONNECT_DELAY_MS);
}
```

Back-port it verbatim. The same commit should take the other correction the newer hook made and the
older one still carries: `useDraft` calls `setError(null)` synchronously inside the effect and
suppresses the lint rule for it (`// eslint-disable-next-line react-hooks/set-state-in-effect` at
`:220`), where `useChatStream` moved that line inside the async body instead and needed no
suppression.

**The shape is the lesson, and it is [hazard 7](../STATUS.md#live-hazards)'s again in a new
costume.** When a fix is discovered while writing the *second* implementation of something, the
first one does not learn about it. Nothing failed, nothing said so, and the newer file reads as
though it were merely more carefully written.

---

### 6. `page` is the one unclamped paging parameter

`ChatController.java:177` takes `@RequestParam(defaultValue = "0") int page` and hands it to
`ChatService.messages`, which validates `size` and not `page`:

```java
if (size < 1 || size > MAX_PAGE_SIZE) { throw BusinessException.badRequest(...); }
Slice<ChatMessage> slice = messageRepository.findSurviving(
        conversationId, Instant.now(), PageRequest.of(page, size));
```

`PageRequest.of(-1, 50)` throws `IllegalArgumentException`, which no handler matches, so
`?page=-1` is a **500** where `?size=-1` is a well-shaped 400 from the line above it.

This is the only place it happens, which is the interesting part: `LeaderboardService` has a
`clampLimit` and the controller calls it, `MatchPlanService` rebuilds the `Pageable` it was given,
and every other paging site takes a Spring-bound `Pageable` that validates itself. Chat is the one
endpoint that took the integers raw.

**Fix:** extend the existing guard to `page < 0`. One condition, in the line above.

---

### 7. `GenerationType.MANUAL` has no i18n key, in any locale

The same drift as finding 4, from the other side. `teamGeneration.generationType.MANUAL` is absent
from `en`, `pt` and `es` — as are `CAPTAIN_PICK` and `STREAK_AWARE`.

`CAPTAIN_PICK` and `STREAK_AWARE` are defensible: the picker's `GENERATION_TYPES` array lists only
the five that have keys, captain pick has its own screen with its own strings, and streak-aware does
not work yet. `MANUAL` is not, because it is not something a user *picks* — it is what the server
stamps on a match somebody records by hand, and it comes back on every read of that match.

This is [the recurring class STATUS.md already names](../STATUS.md#known-documentation-drift): a name
built at runtime and looked up somewhere untyped, so nothing fails when the two sides drift and the
only detector is a person noticing. Third sighting after the `--group_admin` chip and the two
settings labels.

**Fix:** add the key in all three locales. Then write the parity test from suggested next step 6 and
point it at `GenerationType` as well as the three enums that entry names — this family is the one it
would have caught, and the one nobody listed.

---

### 8. `parseGenerationType`'s "valid values" list has drifted from the enum

`MatchPlanService.java:1213-1214`:

```java
throw BusinessException.badRequest("Unknown generationType: " + value +
        ". Valid values: BALANCED, RANDOM, SNAKE_DRAFT, FORM_BASED, STREAK_AWARE");
```

The enum holds eight. The message names five, and gets two of them wrong in opposite directions: it
**omits `OPTIMAL` and `CAPTAIN_PICK`**, both shipped and both reachable from the UI, and it
**advertises `STREAK_AWARE`**, which is the one value guaranteed to fail — `StreakAwareGenerationStrategy`
throws "not yet available" for anybody who takes the advice.

So a caller debugging a typo is told to use the one thing that cannot work and not told about the
two that can. `parseMatchType` directly above it has the same hardcoded shape and happens to still
be right, which is luck rather than a difference in kind.

**Fix:** derive both from `values()`. The message then cannot be wrong again, and the fix is smaller
than the string it replaces.

---

## Two things this review looked for and did not find

Recorded because a negative result costs the next reader nothing to read and a lot to reproduce.

**No cross-tenant read path.** Every `@Cacheable` key carries the tenant; `TenantGuard.assertOwned`
is applied at the id-keyed loads; `PlayerPiiPolicy` is applied *after* the cache rather than inside
it, with the reasoning written down. `ChatSseEmitterRegistry.broadcast` fans out by user id without
re-checking the subscription's tenant — but the recipient list it is handed comes from
`recipientsOf`, which is scoped to the conversation's tenant, so the widest consequence is that a
member of two groups may receive a group-B message over a stream opened for group A. They are
entitled to both, the client no-ops on it (conversation ids are globally unique, so the
`setQueryData` finds no cache entry), and the only visible cost is a redundant refetch of group A's
conversation list. Worth knowing; not worth a finding.

**No hardcoded secret in either repo.** `app.jwt.secret` has no fallback and refuses to boot below
64 bytes; CORS origins are environment-driven in every profile; VAPID keys default to blank and
disable push rather than shipping a key.

---

## What this review did not cover

- **Neither test suite was executed.** The frontend has no `node_modules` in this environment and
  the backend has no Gradle build directory; installing either needs network the session did not
  have. Every finding above is from reading the code, and each one names the file and line so it can
  be checked without taking this document's word for it. **Findings 4, 6 and 7 in particular deserve
  a runtime confirmation before anybody spends time on them** — they are the three where a test run
  would settle it in seconds.
- **No dependency/CVE scan.** The last one on record is
  [SECURITY-AUDIT-2025-07-09](../backend/security/SECURITY-AUDIT-2025-07-09.md), which is thirteen
  months old and predates the Next.js 16 / React 19 / Tailwind 4 upgrades.
- **The visual suite was not touched.** It cannot be meaningfully run off Windows —
  [hazard 7](../STATUS.md#live-hazards) — and [hazard 8](../STATUS.md#live-hazards) makes running it
  on Linux actively misleading.
- **`CalculationService` (1119 lines) was read for structure, not verified numerically.** The
  rating formulas have their own document and worked examples; checking the arithmetic against them
  is a separate exercise and a worthwhile one.

---

## Suggested order

Grouped by what a single branch could reasonably hold, since
[one branch, one reason to exist](../CONTRIBUTING.md#naming-a-working-branch) applies to fixes as
much as to features.

1. **`fix/generation-type-manual`** — findings 4, 7 and 8 together. One drift, three symptoms, two
   repos; the backend half deploys first as usual. Smallest, most visible, and it makes the
   `GET /api/matches/{id}` console error stop.
2. **`fix/draft-sse-reconnect-guard`** — finding 5. A back-port of code that already exists and is
   already tested in `useChatStream`.
3. **`fix/tenant-context-clear-on-skipped-paths`** — finding 1, with the assertion test.
4. **`fix/charge-generation-race`** — finding 2. The largest, because it wants a real decision
   (`REQUIRES_NEW` per insert, or `ON CONFLICT DO NOTHING` and no exception at all) rather than a
   patch, and because the money path deserves an `integrationTest` against real PostgreSQL —
   H2 will not reproduce the aborted-transaction behaviour that is the whole defect.
5. **`fix/groupless-account-refusal`** — finding 3, once somebody has decided what the refusal
   should be. Finding 6 can ride along; it is one condition in the same area of the map.

Then **write the locale parity test**. It is the only item here that prevents a class rather than an
instance, [suggested next step 6](../STATUS.md#suggested-next-steps) has been open through three
production sightings, and finding 7 is the fourth.
