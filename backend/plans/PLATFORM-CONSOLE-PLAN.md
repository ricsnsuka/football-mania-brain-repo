# Platform Operator Console — Plan

**Status: DRAFT — nothing here is authorised, and no code has been written.** Written 2026-08-03
against backend `next` (`47cb80a`). Requested as "what could we come up with", so the ordering is a
recommendation and the last section is the argument against parts of it.

Assumes the `GROUP_ADMIN` rename lands first — this document uses the new name throughout.

---

## The finding that reorders everything

**Most of what was asked for is already built and simply has no screen.**

`PlatformService` already answers two of the three questions, and has since 5a-2:

| Endpoint | Returns | Surfaced in the UI? |
|---|---|---|
| `GET /api/admin/platform/organizations` | id, name, status, createdAt, **memberCount**, playerCount, matchCount — for every group | ❌ **no** |
| `GET /api/admin/platform/health` | organizationCount, activeOrganizationCount, accountCount, subscriptionCount, completedMatchCount, pushConfigured, version | ❌ **no** |
| `POST /api/admin/platform/caches/evict` | deployment-wide flush | ❌ no |
| `…/creation-codes` (issue, list, revoke) | the code funnel | ✅ **the only thing the screen shows** |

`platformService.ts` in the frontend contains three functions, all about creation codes. The
`/platform` route exists and is already gated on `user.platformAdmin`.

So "which groups are out there" and "how many members each has" are **a rendering job, not a
feature**. Only "who administers each group" needs new backend work, and it is one query.

---

## Rung 0 — Render what exists

No backend work. No new grants. No decisions.

- A groups table on `/platform`: name, status, created, members, players, matches.
- A counters strip from `health`: organizations, accounts, completed matches, push configured,
  build version.

**Worth doing on its own merits even if nothing below is ever built**, and worth doing first for a
reason beyond cost: it will immediately show whether these numbers are *useful*. A counter nobody
looks at twice is a counter to delete, and that is cheaper to learn now than after building six more.

One thing it fixes straight away: **`version` answers "what is actually deployed"** — the question
that came up repeatedly this week and was answered by `heroku releases` each time, because the app
itself would not say.

---

## Rung 1 — The questions the data can nearly answer

Each is a query, not a subsystem.

### Who administers each group *(the explicit ask)*

`memberCount` exists; the administrators do not. One query over `membership_roles` for
`GROUP_ADMIN` per tenant, returning username and email per group.

**Why it matters operationally**: it is the answer to "who do I contact about this group", and it
is the only way to spot **a group with no administrator** — which the leave-group and suspend
guards refuse to create, but which a direct database edit or a future bug could still produce. A
group in that state cannot appoint one, because only an administrator may grant the role.

### Liveness, not size

**The single most valuable addition, and the one nobody asks for.** `memberCount` cannot tell a
thriving group from an abandoned one — twelve members and no match since March looks identical to
twelve members playing weekly.

- Last completed match, per group.
- Last login by any member, per group.
- Matches in the last 30 days.

Everything needed is already stored. This is what turns a list of groups into a picture of the
product, and it is the number that should decide Phase 4's App Store gate rather than intuition.

### The creation-code funnel

Issued / redeemed / expired / revoked, and time-to-redemption. The rows already exist; nothing
aggregates them.

Answers the question the gate exists to raise: **is the code actually the constraint?** If codes
are issued and never redeemed, the gate is not what is limiting growth and the hold on self-serve
is protecting nothing.

### Push health

Now that VAPID is configured, `subscriptionCount` becomes meaningful — and per-group, plus
**delivery failure counts**, becomes operational. A push pipeline that has silently stopped
delivering looks exactly like a quiet week.

Note the platform split this exposes: subscriptions belong to *accounts*, not groups, so
"subscriptions per group" is a join through membership and one person with two groups counts twice.
Worth deciding what the number should mean before rendering it.

---

## Rung 2 — Actions, and the line worth not crossing

Everything above is read-only. These are not, and each deserves its own decision.

### Suspending a group

`organizations.status` **exists, is displayed, and is enforced nowhere.** `PlatformService` counts
`ACTIVE` rows and `GroupService` returns the value; no request is refused because a group is not
active. So this is not "add a button" — it is adding enforcement to a field that currently
decorates, which means deciding what a suspended group means for its members mid-season.

The membership-suspension work (`V32`) is the shape to copy: one status, enforced in one resolver,
reversible, and refusing the state that cannot be recovered from.

### An operator audit trail

The operator grant is the most powerful thing in the deployment and currently leaves **no record of
anything it does**. Issuing a creation code is a business decision; a global cache evict is an
operational one; suspending a group is both. None of them are recoverable from the data afterwards.

If any of rung 2 gets built, this comes with it rather than after — an audit table added later
starts empty and is worth nothing for the period that most needs explaining.

### Cross-group user lookup — **the one to think hardest about**

"This person emailed support, which groups are they in?" is a genuine operator need and technically
trivial.

It is also the exact capability the `404`-not-`403` rule spends effort denying group administrators,
and the reason `PlatformGuard`'s surface was kept deliberately tiny: *"a grant that grows by default
is how ADMIN became a superset the first time."*

If it is built: name the person, name their groups, and **stop there** — no group content, ever.
The operator needs to know where someone is, not what they did. Anything more turns the console into
a legitimate reason to read another group's data, and that reason will not go away once it exists.

---

## Worth considering, deliberately not recommended yet

- **Per-group storage.** Avatars are BLOBs. Real before billing, invisible before it.
- **Error/exception surfacing.** A dashboard that duplicates Heroku's logs badly.
- **Impersonation ("view as this group").** The single fastest route to an operator reading group
  data routinely. If support pressure ever makes it necessary, it needs consent, an audit entry and
  a time limit — three things easier to design now than to retrofit.
- **Billing dashboards.** The billing rung is on hold by owner decision. Not now, and not partially.

---

## The argument against

The console is for an audience of **one**, and today there is **one group and no creation code
issued**. Every screen here reports on a population that does not yet exist, and the numbers are all
`1` or `0`.

Rung 0 is still worth doing, because it costs a screen and answers "what is deployed". **Rung 1 is
worth doing when there are enough groups that a list stops being memorisable** — call it five. Rung 2
is worth doing when something has gone wrong and the absence of an audit trail is felt.

The honest sequencing is: **build rung 0, issue the first codes, and let the questions you actually
find yourself asking pick from rung 1.** A console designed before the operating experience exists
will measure what was easy to measure.

---

## What this must not become

`PlatformGuard`'s javadoc sets the constraint and it is worth restating, because a console is
exactly the pressure it was written against:

> Three things this rung: the global cache evict, the organization listing, and the platform
> counters. Everything else an operator might eventually want — suspending a group, reading another
> group's data, the operator console — belongs to later rungs and to a **deliberate decision each
> time**. A grant that grows by default is how "admin" became a superset the first time.

Read-only aggregates are safe to add. Row-level access to another group's content is not, and the
distinction should stay explicit in the code rather than in anyone's memory.
