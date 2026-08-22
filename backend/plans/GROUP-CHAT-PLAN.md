# Ephemeral Group Chat & Presence — Technical Specification

**Date:** 2026-08-21, updated 2026-08-22
**Status:** **IN PROGRESS — steps 1–6 of 7 in review.** The eight questions in §7 are answered, and
so are §6's two privacy questions. Backend: schema
[#210](https://github.com/ricsnsuka/FootMania-Back/pull/210) → send-and-read
[#211](https://github.com/ricsnsuka/FootMania-Back/pull/211) → retention
[#212](https://github.com/ricsnsuka/FootMania-Back/pull/212) → reporting and moderation
[#213](https://github.com/ricsnsuka/FootMania-Back/pull/213) → SSE
[#214](https://github.com/ricsnsuka/FootMania-Back/pull/214) → presence
[#215](https://github.com/ricsnsuka/FootMania-Back/pull/215), **stacked in that order and to be
merged in it**. Frontend:
[FootMania-Simple-Front#72](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/72), carrying
both the stream and presence hooks, based on `release/2.0.0`. **Nothing is merged and nothing is
deployed** — and when it is, the backend goes first. The chat **screens** are built too —
[FootMania-Simple-Front#73](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/73), stacked on
#72 — which this plan never covered: every other frontend deliverable here is a hook. Only
**step 7 (push)** remains.
§4 carries the fifth table the moderation answer required; §7 records the answers rather than the
questions.
**Target release:** `2.0.0` — branches off `release/2.0.0` in both repos, per the freeze in
[CONTRIBUTING.md](../../CONTRIBUTING.md#branches-and-releases)
**Estimated effort:** L — the largest single feature since tenancy. Backend ≈3–4 days, frontend
≈3 days, and the two halves are not independent
**Depends on:** nothing new. It builds on tenancy (V22–V24), the SSE pattern already in
`DraftSseEmitterRegistry`, and the existing Web Push stack
**Contract:** `docs/api/CHAT-API-CONTRACT.md` — to be written in the same commit as the endpoints,
per rule 2 in [CONTRIBUTING.md](../../CONTRIBUTING.md)

---

## 1. The request, verbatim

Recorded exactly as given on 2026-08-21, because everything below it is an interpretation and the
original is the thing to check them against:

> Let's start developing a chat system within groups. The idea is to allow users to send messages
> to other users of the same group. These messages have a short life span, let's say 24 hours. In
> this system, users can send direct messages to a single user, all or even create a group chat.
> The group chat also will have a life span, in this case will be determined by inactivity time —
> after 12 hours of no activity the group is permanetely deleted, alongside all the still exisiting
> messages. Other way to delete group chats is to have the creator deleting it. Since we want to
> have privacy on these chats, roles are not taken in consideration — meaning that no admin can see
> anything unless they are part of the chat and if they are, they don't have more permissions than
> anyone else on the chat. Also, we need to have a place where we can see who's online and offline.

## 2. Requirement summary

1. **Chat is scoped to a group.** Both parties must hold an ACTIVE membership in the same
   organization. This is the tenancy rule the whole app already enforces, applied to new tables.
2. **Three conversation shapes** — direct (one other person), a chat with everyone in the group,
   and an ad-hoc group chat with a chosen subset.
3. **Messages expire after 24 hours** and are permanently deleted, not hidden.
4. **Ad-hoc group chats expire after 12 hours of inactivity**, taking their surviving messages with
   them, and can also be deleted outright by their creator.
5. **Roles grant nothing here.** Participation is the only authority. An administrator who is in a
   chat is an ordinary participant; one who is not sees nothing, not even that it exists.
6. **A presence view** — who is online and who is offline.

## 3. What the codebase already gives us

Read on 2026-08-21 and recorded so it does not have to be rediscovered.

| Need | What exists | Where |
|---|---|---|
| Real-time push to the browser | **SSE, already in production** for draft sessions — an in-memory `SseEmitter` registry fed by `@TransactionalEventListener(AFTER_COMMIT)`, with reconnect handling and per-subscriber failure isolation | `service/DraftSseEmitterRegistry.java`, consumed by `src/hooks/draft/useDraft.ts` |
| Scheduled cleanup | `@EnableScheduling` is on, with two existing schedulers to copy | `service/push/ReminderScheduler.java`, `MvpResolutionScheduler.java` |
| Tenant isolation | `TenantContext` + `TenantGuard` + `@TenantOwned`. **Scheduled work must call `TenantContext.runAs(...)`** — a scheduler that forgets has no tenant bound and throws | `tenancy/` |
| Group membership | `Membership` (tenant_id, user, status, roles). An ACTIVE row is the test for "in this group" | `model/Membership.java` |
| Notifications | Web Push implemented against the JDK, with categories and per-user mutes | `service/push/`, `model/NotificationCategory.java` |
| Caching | Caffeine, local. **No Redis, and deliberately so** | `config/CacheConfig.java` |

**There is no WebSocket dependency and no message broker.** The proposal below stays on SSE rather
than introducing either: the pattern is proven in this codebase, the frontend already knows how to
consume it, and chat is the same shape of problem the draft screen already solved.

⚠️ **The SSE registry is per-JVM.** Its own javadoc says so — multi-instance deployments need
sticky sessions or a broker. Chat makes that limitation matter far more than draft sessions did,
because chat is always-on rather than opened for one draft.

**Checked 2026-08-22: production is `web=1:Basic`,** and Heroku's Basic tier cannot scale
horizontally at all — so the design holds by construction rather than by luck. The thing to watch
is not the current count but the tier: moving to Standard or Performance makes a second dyno
possible, and on two dynos half the messages never arrive and nothing errors. See §7.7.

## 4. Proposed data model

**Five** tables, one migration (`V41`), as built. All tenant-owned. The fifth was not in the
original four: it is what answering §7's moderation question required.

```
chat_conversations
  id, tenant_id, kind (DIRECT | GROUP | EVERYONE), name, created_by_user_id,
  created_at, last_activity_at
  UNIQUE (tenant_id) WHERE kind = 'EVERYONE'      -- partial: one channel per group
  INDEX (tenant_id, last_activity_at DESC)
  INDEX (last_activity_at) WHERE kind = 'GROUP'   -- the 12-hour retention scan

chat_participants
  id, tenant_id, conversation_id, user_id, joined_at, last_read_at
  UNIQUE (conversation_id, user_id), INDEX (user_id, tenant_id)

chat_messages
  id, tenant_id, conversation_id, sender_user_id, body, created_at, expires_at
  INDEX (conversation_id, created_at DESC), INDEX (expires_at)

chat_message_reports                              -- the only durable table
  id, tenant_id, message_id, conversation_id, conversation_kind,
  reported_by_user_id, reported_user_id, body_snapshot, message_created_at,
  reason, created_at, reviewed_at, reviewed_by_user_id
  INDEX (tenant_id, created_at) WHERE reviewed_at IS NULL

user_presence
  user_id, tenant_id, last_seen_at
  PRIMARY KEY (user_id, tenant_id), INDEX (tenant_id, last_seen_at DESC)
```

`created_by_user_id` is `ON DELETE SET NULL`, not `CASCADE`: one person erasing their account must
not destroy a conversation belonging to everyone else in it. What is lost is the delete-my-own-chat
right, which nobody is left to exercise; the 12-hour clock still collects it.

**The group-wide channel's membership is not stored.** `chat_participants` is the authority for
`DIRECT` and `GROUP` only. For `EVERYONE` the authority is an ACTIVE `Membership` in the tenant —
the one definition that stays correct as people join, are suspended and leave. Participant rows
still appear for it, carrying `last_read_at` and nothing that grants anything.

**`expires_at` is a stored column, not `created_at + 24h` computed at read time.** The retention
window is a product decision that may change; rows written under the old window should keep the
lifespan they were written with rather than silently extending or shortening when a constant moves.
It also makes the delete a single indexed range scan.

**Deletion is real deletion.** `DELETE FROM chat_messages WHERE expires_at <= now()`, not a
`deleted_at` stamp. "Permanently deleted" was explicit in the request, and a soft-deleted row is
still a row that a future query, export or backup can surface.

### Two lifespans that interact

A message lives 24 hours. An ad-hoc group chat dies after 12 hours of silence **and takes its
messages with it**. So a message in a quiet group chat is destroyed at roughly half its advertised
lifespan — the conversation's clock, not the message's, is what ends it. This follows directly from
the request as written and is probably intended, but it is the kind of rule people notice only when
something they expected to still be there is gone. Flagged in §7.

## 5. Proposed API

All under `/api/chat`, all `@PreAuthorize("isAuthenticated()")` and **nothing more** — see §6.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/chat/conversations` | The caller's conversations. Only ever theirs |
| `POST` | `/api/chat/conversations` | Create — `kind`, optional `name`, participant user ids |
| `GET` | `/api/chat/conversations/{id}/messages` | Page of surviving messages |
| `POST` | `/api/chat/conversations/{id}/messages` | Send. Bumps `last_activity_at` |
| `DELETE` | `/api/chat/conversations/{id}` | Creator only, and only for `GROUP` |
| `POST` | `/api/chat/conversations/{id}/read` | Move `last_read_at` |
| `GET` | `/api/chat/events` | **One SSE stream per user**, carrying every conversation they are in |
| `GET` | `/api/chat/presence` | Roster of the group with online/offline |
| `POST` | `/api/chat/presence/heartbeat` | Refresh `last_seen_at` |

**One SSE stream per user, not per conversation.** A stream per conversation multiplies held
connections by however many chats somebody is in, and browsers cap concurrent connections per
origin. The draft registry keys emitters by session id; the chat registry keys by **user id** and
fans each message out to the emitters of every participant.

**Presence is a heartbeat plus a threshold, not a connection count.** Online means `last_seen_at`
within a small window (60s suggested), refreshed by the open SSE stream itself. Deriving it from
"has an open emitter" ties truth to one JVM's memory and reports everyone offline after a deploy
until they reconnect.

## 6. The privacy rule, concretely

The part most likely to be got wrong by accident, so it is written as a rule with teeth:

- **No chat endpoint may reference `Role` at all.** Not `GROUP_ADMIN`, not in a `@PreAuthorize`, not
  in a service branch. Authorization asks one question: *is the caller a participant in this
  conversation?*
- **`PlatformAdmin` gets nothing either.** Platform operators bypass tenancy elsewhere via
  `PlatformGuard`; chat must be explicitly excluded from that path, not merely left unmentioned.
- **The admin listing endpoints must not grow a chat section**, and `AdminController` should not
  gain a chat dependency.
- **Decide what the GDPR export does** before shipping. `PrivacyService` builds a data export, and
  "your messages are ephemeral and are not exported" is a defensible answer — but it has to be a
  decided answer written into `PRIVACY-API-CONTRACT.md`, not an omission.
- **Tests must include a negative one**: a `GROUP_ADMIN` who is not a participant gets `404`, not
  `403` — `403` confirms the conversation exists.

### The moderation trade-off — decided 2026-08-22: reporting is in

The trade-off was real: a group chat with no administrator visibility and 24-hour deletion means
harassment leaves no evidence and no lever — the group admin cannot read it, cannot remove it, and
cannot prove it happened. **The answer chosen is the middle option**, and it holds administrator
visibility at exactly zero. A participant can *report* one message, which copies that message out
of the ephemeral store into `chat_message_reports`. The decision moves to the only person entitled
to make it: someone who was in the conversation and read it.

Two consequences the schema had to be built around, both shipped in `V41`:

- **A report must outlive everything it points at.** Its foreign keys are `ON DELETE SET NULL`
  where the rest of chat cascades, and it carries `body_snapshot` rather than a reference. A report
  whose keys cascaded would be destroyed by the same retention job that collects the message it
  preserves — always before anybody read it, and silently. `ChatSchemaIT` asserts survival against
  deletion of the message, the conversation and the reporter.
- **The one cascade is the reported account.** Once that person is erased under GDPR the record's
  purpose is spent, and keeping their words in a table they can no longer reach is the retention an
  erasure request exists to stop.

**Reports are not reachable from chat.** No `/api/chat` endpoint reads them; they belong to a
separate moderation surface, because the rule that no chat endpoint may consult a `Role` has to
survive the arrival of a feature whose whole point is that an administrator acts on it. Who may
read the queue, and how long a report is kept, are decided in the PR that builds that surface —
`V41` deliberately does not answer either.

## 7. The eight questions, answered

Answered 2026-08-22 — four by the user, two by reading production and the dependency list, and two
by taking the assumption the draft had already written down. Recorded with the reasoning so a later
reader can tell a decision from a default.

1. **Does the 12-hour inactivity death apply to direct chats too? — No.** Confirmed as drafted:
   direct conversations persist as empty containers, and their messages still expire at 24h. That
   is also what makes messaging somebody again continue the same thread rather than opening a
   second one. `ChatConversation.expiresOnInactivity()` is the single place this lives.
2. **What is "all"? — A permanent group-wide channel.** One `EVERYONE` conversation per
   organization, created lazily on first use, never deleted and exempt from the inactivity clock.
   Enforced by a partial unique index rather than a service check, because lazy creation makes two
   people opening chat simultaneously a real race.
3. **Is the two-clock interaction intended? — Yes, as written.** A quiet group chat is destroyed at
   12 hours and takes messages that were promised 24 with it. Intended, and flagged in
   `ChatMessage`'s javadoc and in a test, because it is the kind of rule people notice only when
   something they expected is gone.
4. **Presence scope and visibility — per-group, visible to that group's members.** Taken as
   drafted. **No invisible opt-out** for 2.0.0; if one is wanted later it is a column on
   `user_presence` and a filter on the roster, not a redesign.
5. **Push notifications on new messages? — Yes**, as the final step. Chat is much less useful
   without it: a message that dies in 24 hours and is never seen is simply lost. The new
   `NotificationCategory` ships with **en, pt and es** strings in the same commit — a missing key
   reaches production looking like `CHAT_MESSAGE`.
6. **Read receipts and typing indicators — out of scope**, as drafted. `last_read_at` exists for
   unread counts and nobody but its owner ever sees the value.
7. **How many dynos? — one, and it cannot currently be more.** `heroku ps -a footmania` reports
   `web=1:Basic`. Heroku's Basic tier does not support horizontal scaling at all, so the per-JVM
   SSE registry is sound *by construction* rather than by luck. ⚠️ That guarantee is a property of
   the dyno tier, not of the app: moving to Standard or Performance and scaling past one dyno
   breaks delivery **silently** — half the events reach the wrong process and nothing errors.
   Sticky sessions or a broker have to come first. Recorded in `ChatConversation`'s javadoc and the
   `V41` header so it is found by somebody changing the dyno formation, who will not be reading
   this file.
8. **Message body — plain text, 2000 characters.** `org.owasp.encoder:encoder:1.3.1` was already on
   the dependency list as the draft assumed. Stored raw and encoded on the way *out*: escaping on
   the way in would corrupt the text for every non-HTML consumer (a push notification body, a
   future export) and double-encode as soon as anything else escaped it too.

## 8. Build order

Each step is a pull request into `release/2.0.0`, and each leaves the app working. Seven rather
than the original six: answering §7.1's moderation question added the reporting step.

1. ✅ **Schema and model** — `V41`, the five entities, repositories, tenancy annotations.
   [FootMania-Back#210](https://github.com/ricsnsuka/FootMania-Back/pull/210), in review. 11
   integration tests for what only the migration can express, 12 unit tests for the lifecycle rules.
2. ✅ **Send and read, no realtime** — six endpoints, `docs/api/CHAT-API-CONTRACT.md`, and
   `ChatPrivacyIT` asserting **404, not 403** against a real `GROUP_ADMIN` membership.
   [FootMania-Back#211](https://github.com/ricsnsuka/FootMania-Back/pull/211), stacked on #210.
   46 tests. Two things the build taught that the plan had not anticipated:
   - **The `EVERYONE` race needs its own bean.** The loser of the creation race cannot re-read
     inside the transaction that hit the constraint — `DataIntegrityViolationException` marks it
     rollback-only — so the insert needs `REQUIRES_NEW`, and Spring's proxy-based transactions mean
     a private method on `ChatService` would carry the annotation and silently ignore it. Hence
     `EveryoneChannelProvisioner`.
   - **The OWASP encoder is the wrong tool** and was never used: the API returns JSON, which
     Jackson escapes, to a React client that renders text. §7.8's answer stands, but the mechanism
     named in it does not — corrected in the code.
3. ✅ **Retention** — two sweeps on one fifteen-minute tick, with the clock passed in rather than
   read. [FootMania-Back#212](https://github.com/ricsnsuka/FootMania-Back/pull/212), stacked on
   #211. 17 tests. **The plan's `TenantContext.runAs` was not used, deliberately:** neither sweep
   reads a bound tenant — both are single bulk statements, and the windows are product constants
   rather than per-group settings — so binding one would change no row and looping the
   organizations would issue two statements per group to apply the same two constants. If a group
   ever gets its own retention window, this is the code that becomes a loop. `ChatRetentionIT`
   asserts the global behaviour across two tenants so that change fails loudly.
   - Also worth keeping: the cron had to become a property (`app.chat.retention.cron`, `-` to
     disable) because the sweep is global and absolute and would delete rows a test had just
     seeded — a flake that would have read as a bad assertion rather than as the scheduler working.
4. ✅ **Reporting and the moderation surface** —
   [FootMania-Back#213](https://github.com/ricsnsuka/FootMania-Back/pull/213), stacked on #212.
   36 tests. Filing lives on the chat surface (a participant's act, participation-authorized);
   reading the queue is `ModerationController` under `/api/moderation`, `GROUP_ADMIN` only. The
   payload carries **no conversation id, message id or participant list** — useless to an
   administrator anyway, but a conversation id would be a handle saying two reports came from the
   same private conversation. `ChatModerationIT` asserts the pair: the report arrives *and* the
   conversation is still a 404 to that administrator.
   - **§6's two open privacy answers, now decided.** *Nothing from chat is in a data export* —
     messages because nothing survives to export, reports under **Article 15(4)**, since in a
     twelve-person group disclosing that a report exists identifies the reporter about as reliably
     as naming them. *The reporter is shown* to the administrator rather than anonymised: among
     people who know each other, an anonymous report is easier to abuse than to hide behind.
   - **A gap chat had introduced:** `leaveGroup` left every chat row behind, because the `V41`
     foreign keys only fire on account *deletion*. Not an access hole — `TenantResolver` only binds
     a group the caller actively belongs to — but the participant rows would have restored old
     conversations to anybody who rejoined. Now cleared, direct threads first; reports about the
     departing member are deliberately kept.
5. ✅ **SSE** — [FootMania-Back#214](https://github.com/ricsnsuka/FootMania-Back/pull/214) and
   [FootMania-Simple-Front#72](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/72), the
   first step touching both repos. 29 tests. **The recipient list is the authorization boundary** —
   nothing in the delivery path re-checks participation, so recipients are resolved inside the
   sending transaction from the same two sources authorization uses, and the registry test asserts
   who does *not* receive a message.
   - **A keep-alive the draft stream never needed.** Heroku's router closes any connection that
     transmits nothing for 55 seconds. Draft events are frequent; an idle chat stream is silent for
     hours, so a comment frame every 25s is what stops every client reconnecting once a minute for
     as long as the app is open. Clients must skip lines starting with `:` rather than parse them.
   - **Inactivity deaths emit nothing**, deliberately: the sweep deletes in bulk and does not know
     which conversations it took. A vanished conversation is noticed as a 404, which the contract
     already called ordinary.
6. ✅ **Presence** — [FootMania-Back#215](https://github.com/ricsnsuka/FootMania-Back/pull/215) and
   the frontend hook on FootMania-Simple-Front#72. 16 tests. Presence is a **timestamp and a
   threshold**, never a connection count — deriving it from open emitters would report the whole
   group offline after every deploy. The open stream writes the timestamp (which is a different
   thing), so the registry now carries each subscription's tenant: presence is per-group and one
   stream would otherwise know somebody was here but not where.
   - **Two deliberate restraints, both asserted:** no last-seen timestamp (the request asked who is
     online; a log of somebody's habits visible to everyone they play with is a different feature
     nobody asked for) and no presence event on the SSE stream (a group of twenty would produce a
     constant trickle of arrive/depart frames to keep a dot the right colour). Poll at half the
     window instead — presence is stale by construction, so nothing is lost.
   - The roster is the **membership marked up**, not the presence table: somebody who has never
     opened chat is offline rather than absent.
7. ⬜ **Push** — the new category with all three locales in the same commit.

## 9. Related

- [CONTRIBUTING.md](../../CONTRIBUTING.md#branches-and-releases) — the `next` freeze; this work
  bases on `release/2.0.0`
- [architecture/multi-tenancy.md](../../architecture/multi-tenancy.md) — the isolation this feature
  must not become the exception to
- [backend/features/PRIVACY_AND_DATA_PROTECTION.md](../features/PRIVACY_AND_DATA_PROTECTION.md) —
  where the §6 export decision has to land
