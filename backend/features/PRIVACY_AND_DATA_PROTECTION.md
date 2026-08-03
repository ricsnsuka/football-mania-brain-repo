# Privacy & Data Protection

**Added in:** v1.0.0 (Phase 0)
**Date:** July 2026
**Status:** ✅ Backend implemented — privacy policy page outstanding (frontend)

---

## Why this exists now

The roadmap's Phase 0 flags GDPR posture as a decide-now item, and the reasoning holds up: the
system stores phone numbers and email addresses, the frontend targets `en`/`pt`/`es` (EU, Iberia
and LatAm), push subscriptions in Phase 2 add device tokens, and Phase 4 makes a public privacy
policy a hard requirement of both app stores. Retrofitting a data-export and erasure path onto a
live multi-tenant system is materially harder than building it into a single-tenant one, so it is
built now.

This document records the posture and points at the code that implements it. It is **not** legal
advice and it is not the privacy policy — see [What is still outstanding](#what-is-still-outstanding).

---

## What personal data is stored

| Data | Where | Why it is held |
|------|-------|----------------|
| Username, email, first/last name | `users` | Authentication and identifying who did what |
| Which groups you belong to, and when you joined | `memberships` | Knowing who is in a group. Associational data about one person; retained for the life of the membership, and removed with the account |
| What you are allowed to do in a group | `membership_roles` | Authorisation. Per group since V23 — a grant held in one group says nothing about any other |
| Password (bcrypt hash) | `users` | Authentication. Not personal data in a portable sense — a hash cannot be handed to another controller, so it is excluded from exports |
| Display name, phone number | `players` | Organising matches; contacting players about fixtures |
| Match appearances, goals, assists, own goals, MVP, per-match rating | `player_stats`, `goals` | The competitive record the group exists for |
| Skill rating and its full change history | `players`, `skill_rating_history` | Team generation and balance |
| Availability responses and free-text notes | `player_confirmations` | Running the availability poll |
| Username as audit actor | `players.created_by`/`updated_by`, `match_plans.created_by`, `draft_sessions.created_by` | Accountability for administrative actions |
| Push endpoint + encryption keys, device label | `push_subscriptions` | Delivering notifications to a specific browser install |
| Muted notification categories | `notification_mutes` | Respecting an explicit opt-out |
| Achievement badges and when they were earned | `player_badges` | Recognising milestones. Derived entirely from statistics already held, and a fact about one person only — so unlike the MOTM votes it needs no special handling in the export. Awards are permanent: there is no revocation path, and erasure anonymises rather than deletes |
| Crowd MOTM votes — who voted, and for whom | `match_mvp_votes` | Running the post-match Man of the Match poll. Personal data about **two** people per row, which is why the export treats the two directions differently: a subject's own votes are reported with the player they chose, but votes *received* are reported as a count only. Naming the voters would disclose their votes — their personal data, not the subject's — and would let a winner unmask the ballot by requesting an export |
| Request logs (method, path, status, duration, resolved username) | Application logs | Operational debugging and security auditing |
| Match fees charged, and payments received | `player_charges`, `player_payments` | Tracking who owes the group money for the pitch. Financial data about an identified person, but it names nobody else — what the pitch cost is not another player's personal data, and each row records one person's share. Keyed on the player rather than the account, so people who never registered are covered too. Erasure keeps the amounts and scrubs the audit columns |

**Not stored:** payment *instruments* of any kind — no card numbers, IBANs, MB WAY identifiers or
tokens. The money moves between phones and this application never sees it; a payment row is
somebody's word that it arrived, holding an amount and a date and nothing else. Also not stored:
location beyond a free-text pitch name, and biometric or special-category data of any kind.

---

## Who is responsible for what: the controller/processor split

**Revised in Phase 5a-3.** The previous version of this section assumed one deployment serving one
group, and booked its own revisit: *"the operator of a hosted service and the group organiser are
then different parties, and the controller/processor split has to be written down."* This is that.

There are two parties and two kinds of data, and conflating them is what makes multi-tenant
privacy go wrong.

| | Controller | Processor | What it covers |
|---|---|---|---|
| **A group's competition data** | the group, through whoever holds `ADMIN`/`ORGANIZER` in it | the platform operator | roster, matches, goals, ratings, availability, the fee ledger, badges, MOTM votes |
| **Account data** | the platform operator | — | credentials, email, push endpoints, notification preferences, memberships |
| **Platform operations** | the platform operator | — | logs, backups |

The practical consequences, which are what the code now enforces:

- **A group's administrators decide who is on their roster and when somebody is erased from it.**
  The operator acts on those instructions through the product's features. That is why
  `DELETE /api/privacy/players/{id}` anonymises the player *in that group* and ends that
  membership — it no longer deletes the account, which was never the group's to delete.
- **A group administrator's export gives them what their group processes, nothing more.** The
  admin-actioned export carries no platform tier: not the person's email, not their devices. A
  group has no standing to hand out account data it does not control.
- **Direct requests to the right party.** Sports-data questions ("delete my record from this
  group") go to the group's organisers. Account questions ("delete my account entirely") go to the
  operator, and the person can action that themselves.

### Lawful basis

**For a group's competition data:** **contract** (you joined the group; organising fixtures and
keeping the results is what the group does) and **legitimate interest** (retaining historical
match results so the competition's record stays coherent). Unchanged in substance — but it is now
the *group* relying on them, per group.

**For account data:** **contract** with the operator (you asked for an account, the operator
provides one).

Note that the guest-players data-minimisation argument survives this rewrite intact: a guest
joined nothing, has no account, and the group holds only a name and a skill estimate for them —
which is why guests were deliberately given no phone number.

### Sub-processors

*None at present.* This section exists and is deliberately empty so that the first one has an
obvious home. The billing rung would add a payment processor here; it is on hold.

### Data processing agreement

A DPA between the operator and each group is an **operator task requiring legal review**, flagged
here rather than drafted. This document is an engineering record of what the system does; it is
not legal advice and no template belongs in it.

---

## Data-subject rights

| Right | Article | How it is served |
|-------|---------|------------------|
| Access | 15 | `GET /api/privacy/me/export` |
| Portability | 20 | Same endpoint — JSON, offered as a file download |
| Erasure | 17 | `DELETE /api/privacy/me/groups/{groupId}` to leave one group; `DELETE /api/privacy/me` to erase the account once it is your last |
| Rectification | 16 | Existing `PATCH /api/players/{id}` and `PATCH /api/users/{id}` |
| Restriction / objection | 18, 21 | Manual — no automated route. Deactivating a player (`PATCH /api/players/{id}/status`) removes them from selection without erasing anything |

Implementation: [`PrivacyService`](https://github.com/ricsnsuka/FootMania-Back/blob/main/src/main/java/pt/rics/demo/football/service/PrivacyService.java),
[`PrivacyController`](https://github.com/ricsnsuka/FootMania-Back/blob/main/src/main/java/pt/rics/demo/football/controller/PrivacyController.java).

### Endpoints

| Method | Path | Access | Purpose |
|--------|------|--------|---------|
| `GET` | `/api/privacy/me/export` | Any authenticated user | Download your own data — platform tier plus one section per group |
| `DELETE` | `/api/privacy/me/groups/{groupId}` | Any authenticated user | **Leave one group.** Anonymises your player there, ends the membership, scrubs that group's audit trail. Everything else untouched |
| `DELETE` | `/api/privacy/me` | Any authenticated user | Erase your account from the platform. `409` while you belong to more than one group |
| `GET` | `/api/privacy/players/{id}/export` | `ADMIN_USER` | Action a request on someone's behalf — that group's section only, no platform tier |
| `DELETE` | `/api/privacy/players/{id}` | `ADMIN_USER` | Remove someone from **this group** and anonymise them. Does not touch their account |

The `/me` endpoints take their subject from the authenticated principal, never from a request
parameter — there is nothing a caller can change to read or erase another person's record. The
`/players/{id}` endpoints exist because **a player added by an admin may have no account at all**,
and that person has the same rights with no way to exercise them; they are `ADMIN_USER`-only, not
`MASTER_USER`, because one reads a full personal record and the other is irreversible.

---

## The export

**Two tiers since 5a-3.** The document is a platform section — the account and its registered
devices, which the operator controls — followed by one section per group the person belongs to.
Each group section carries that group's name, the roles held there, and the player profile,
matches, goals, rating history, availability, votes, badges and ledger *as that group holds them*,
resolved under that group's tenant.

A flat document was truthful only while a person could belong to one group. It cannot say which
group a goal was scored in, and merging two groups' records would describe a career the person
never had.

The admin-actioned export (`/players/{id}/export`) carries **one group section and no platform
tier** — see the controller split above.


One JSON document containing: the account, the player profile, the groups the subject belongs to
with their per-group grants, every match appearance, every goal scored or assisted, the full rating
history, and every availability response including the free-text notes the subject wrote.

Two deliberate constraints:

**Scope is "data about this person", not "data this person can see".** A match the subject played
in is represented by *their own line* in it — their team's name, their goals, their rating — never
by the full scoresheet. Goals name the subject's role (`SCORER` / `ASSISTER`) but not the
counterpart. Without that rule the export becomes a way for any member to harvest the whole
roster's record in one call, which is the same failure the `PlayerPiiPolicy` exists to prevent on
the player endpoints.

**Nothing is filtered.** Unlike the internal stat queries, the export query filters out neither
incomplete matches nor season-transition rating rows (which carry no match). A copy of someone's
data that quietly omitted categories would not be the complete copy Article 15 asks for.

---

## Erasure: anonymisation, not deletion

`player_stats`, `goals` and `skill_rating_history` all reference `players(id)` with
`ON DELETE CASCADE`. Deleting a player would therefore delete their appearances, the goals they
scored, and the rating history derived from both — which **silently rewrites other players'
records**: scorelines stop matching their goal lists, and career aggregates recomputed from the
surviving rows drift. One person exercising their rights must not corrupt everyone else's data.

So erasure strips what identifies the person and leaves the anonymous statistical residue:

| Field | After erasure |
|-------|---------------|
| `players.name` | `Deleted player #{id}` |
| `players.phone_number` | `NULL` |
| `players.is_active` | `false` |
| `players.user_id` | `NULL` |
| `players.anonymized_at` | Timestamp of the erasure |
| `players.updated_by` | `[erased]` |
| `players.created_by` | `[erased]` **only if** it was the subject's own username |
| `users` row | Deleted outright |
| `match_plans.created_by`, `draft_sessions.created_by` | `[erased]` wherever they held the subject's username |
| Match results, goals, ratings, streaks, career totals | **Unchanged** |
| `match_mvp_votes` — both votes cast and votes received | **Unchanged** (see below) |
| `player_badges` | **Unchanged** — an award attached to `Deleted player #5` attributes nothing to anybody, and removing it would rewrite achievement history other players can see |
| `player_charges`, `player_payments` — the amounts | **Unchanged** — they are the group's books, and a charge attached to `Deleted player #5` attributes nothing to anybody |
| `player_charges.created_by`/`voided_by`, `player_payments.recorded_by`/`voided_by` | `[erased]` wherever they held the subject's username. Usually an erased **organiser**, not an erased player — it is whoever wrote the rows down whose name sits in these columns |

This is what Article 17 asks for: once the data can no longer be attributed to an identified
person, it is no longer personal data and the retention question falls away. Usernames are scrubbed
from the audit columns for the same reason — a username identifies a person as surely as a name
does, and leaving it behind would make the erasure cosmetic.

`anonymized_at` (V11) is what makes the state durable: it is the guard against a second erasure
attempt, it distinguishes an erased record from one merely deactivated, and it is the evidence the
request was actioned. It holds no personal data itself.

### Crowd MOTM votes survive, deliberately

`match_mvp_votes` cascades from `players`, but erasure never deletes a player — so votes remain,
now cast by an anonymised person. That is the correct outcome in **both** directions, and neither
is an oversight:

- **Votes the subject cast.** Deleting them would change other matches' crowd MOTM results after
  the fact — a result that belongs to everyone who played, not to the voter. Anonymising the player
  is what removes the attribution: the row survives as "somebody voted for X", which is no longer
  personal data about an identified person. Exactly the argument that keeps `player_stats`.
- **Votes the subject received.** Those rows are *other people's* votes. One person exercising
  their rights gives no basis for destroying somebody else's record of what they did.

A tie or a decided result therefore stays stable across an erasure, which matters because
`matches.crowd_mvp_player_id` may point at the erased player — it then reads as the tombstone name,
the same way the league table does.

### Two guards

- **An erased record cannot be erased again** — `409 Conflict`.
- **The last administrator of a group cannot leave it** — `403 Forbidden`. Narrowed in 5a-3 from
  "an ADMIN cannot erase their own account", which meant "do not lock everybody out" when there was
  one deployment. The concern per group is sharper: only an administrator can grant roles, so a
  group at zero administrators cannot appoint one and needs operator intervention. Leaving is fine
  while somebody else can administer the group.
- **An account belonging to several groups cannot be erased from the platform** — `409 Conflict`,
  listing the groups. Leave each one first; the last one offers platform erasure as the natural
  final step. Somebody leaving their Tuesday five-a-side has not asked to vanish from two others,
  and the endpoint cannot tell which they meant.

### What erasure does not reach

Application **logs** contain resolved usernames on request-completion entries (see
`HttpRequestLoggingFilter`). Erasure does not rewrite them. Whatever log retention the deployment
uses is therefore the effective backstop, and a short retention window is the practical answer —
this should be set explicitly before public signup rather than left to the platform default.

---

## What is still outstanding

| Item | Owner | Blocking |
|------|-------|----------|
| **Public privacy policy page** | Frontend | Mandatory for both app stores (Phase 4) and for any public signup |
| **Log retention policy** | Deployment | Erasure cannot reach logs; retention is the backstop |
| **Cookie / local-storage disclosure** | Frontend | The app stores a JWT and a theme preference in `localStorage` — strictly necessary, so no consent banner is required, but it must be disclosed |
| **Controller/processor split** | Phase 5 | Only becomes a question once one deployment serves multiple independent groups |
| ~~**Push subscription data**~~ | ~~Phase 2~~ | ✅ Done — `push_subscriptions` and `notification_mutes` are in the export and cascade on erasure. See below |

The privacy policy needs to state, at minimum: who the controller is, what is collected (the table
at the top of this document), why, the lawful basis, how long it is kept, who it is shared with
(nobody today — no third-party processors, no analytics), the rights above and how to exercise them,
and a contact address.

---

## Push subscriptions

A push subscription is personal data — the endpoint identifies one specific browser install and
the keys are unique to it.

**Erasure** needs no special handling: both `push_subscriptions` and `notification_mutes`
reference `users(id)` with `ON DELETE CASCADE`, and erasure deletes that row. Unlike the player
record, there is nothing here worth anonymising and keeping — a subscription with its keys
removed is not a usable send target, just a row.

**Export** reports the device label, the push service **host**, the registration time and the
last notification time. Three fields are deliberately withheld:

| Withheld | Why |
|----------|-----|
| The full `endpoint` | It is a capability, not a fact about the person. Anyone holding it plus the keys can push to that browser |
| `p256dh` | Encryption key for that browser |
| `auth` | Subscription auth secret |

Article 15 gives a right to a copy of one's personal data, not to a working credential set. The
useful fact — *this browser on this push service was registered on this date* — is exported in
full; the material that would let a third party send to that device is not, because an export
file is precisely the sort of thing people forward on.

## When adding a new table that holds personal data

Three places need updating, and missing any one of them makes the export or erasure quietly
incomplete:

1. The table at the top of this document.
2. `PrivacyService` — a section in the export, and a step in `erase(...)`.
3. `PrivacyServiceTest` — a case proving both.

Phase 5's `memberships`/`membership_roles` were the most recent; like `user_roles` before them they
cascade with the `users` row, which is why the erasure table below does not list them. Phase 2's
`push_subscriptions` is the fullest worked example — see the section above for how it was handled.
