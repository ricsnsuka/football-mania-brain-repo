# Ephemeral Group Chat & Presence — Technical Specification

**Date:** 2026-08-21
**Status:** **DRAFT — not implemented.** Nothing has been built. This document exists so the work
can be picked up on another machine; it records the request verbatim, what the codebase already
provides, a proposed design, and the questions that must be answered before code is written.
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
because chat is always-on rather than opened for one draft. **Confirm the dyno count before
building**: on a single dyno this is fine; on two, half the messages never arrive and nothing
errors.

## 4. Proposed data model

Four tables, one migration (`V41`). All tenant-owned.

```
chat_conversations
  id, tenant_id, kind (DIRECT | GROUP | EVERYONE), name, created_by_user_id,
  created_at, last_activity_at

chat_participants
  id, conversation_id, user_id, joined_at, last_read_at
  UNIQUE (conversation_id, user_id)

chat_messages
  id, conversation_id, sender_user_id, body, created_at, expires_at
  INDEX (conversation_id, created_at), INDEX (expires_at)

user_presence
  user_id, tenant_id, last_seen_at
  INDEX (tenant_id, last_seen_at)
```

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

⚠️ **The moderation trade-off is real and should be an explicit decision.** A group chat with no
administrator visibility and 24-hour deletion means harassment inside a group leaves no evidence and
no lever: the group admin cannot read it, cannot remove it, and cannot prove it happened. That is
the direct consequence of the privacy requirement as stated, and it may well be the right call for a
five-a-side app between people who know each other. It should be a decision somebody made, not a
property nobody noticed. A middle option exists: participants can *report* a message, which copies
that one message out of the ephemeral store into a moderation record.

## 7. Open questions — answer before building

1. **Does the 12-hour inactivity death apply to direct chats too?** The request attaches it to group
   chats only. Assumed: direct conversations persist as empty containers; their messages still
   expire at 24h.
2. **What is "all"?** A permanent group-wide channel that always exists (`EVERYONE`, one per
   organization, undeletable), or an ad-hoc chat that happens to contain everyone? Assumed the
   former — it is the one people expect to find rather than create.
3. **Is the interaction in §4 intended** — a quiet group chat destroying 12-hour-old messages that
   were promised 24?
4. **Presence scope and visibility.** Online within *this group* to *its members*, or globally?
   Assumed per-group, visible to that group's members only. Is an "invisible" opt-out needed?
5. **Push notifications on new messages?** The infrastructure exists, including per-category mutes.
   A new `NotificationCategory` needs locale strings in **en, pt and es** — a missing key reaches
   production looking like `CHAT_MESSAGE`.
6. **Read receipts and typing indicators** — assumed out of scope for 2.0.0. `last_read_at` is in
   the schema for unread counts, which is a different thing.
7. **How many dynos?** See the warning in §3. This decides whether SSE is sufficient as designed.
8. **Message body limits** — length cap, and whether anything but plain text is allowed. Assumed
   plain text, 2000 characters, escaped through the OWASP encoder already on the dependency list.

## 8. Suggested build order

Each step is a pull request into `release/2.0.0`, and each leaves the app working.

1. **Schema and model** — `V41`, the four entities, repositories, tenancy annotations.
2. **Send and read, no realtime** — endpoints, service, participation-only authorization, the
   contract file, and the negative admin test. Polling is enough to prove it works.
3. **Retention** — the two scheduled jobs, `TenantContext.runAs`, and tests that advance a clock
   rather than sleeping.
4. **SSE** — the per-user registry, the event on commit, a frontend hook modelled on `useDraft`.
5. **Presence** — heartbeat, roster endpoint, the online/offline view.
6. **Push** — only if §7.5 is answered yes, and with all three locales in the same commit.

## 9. Related

- [CONTRIBUTING.md](../../CONTRIBUTING.md#branches-and-releases) — the `next` freeze; this work
  bases on `release/2.0.0`
- [architecture/multi-tenancy.md](../../architecture/multi-tenancy.md) — the isolation this feature
  must not become the exception to
- [backend/features/PRIVACY_AND_DATA_PROTECTION.md](../features/PRIVACY_AND_DATA_PROTECTION.md) —
  where the §6 export decision has to land
