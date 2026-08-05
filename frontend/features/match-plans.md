# Match Plans Feature

Server-side paginated card list of match plans, with the availability poll, the waitlist, guests,
the pitch cost and plan creation — including a **weekly run**, created in one request.

Backend: [MATCH_PLANS_FEATURE](../../backend/features/MATCH_PLANS_FEATURE.md) ·
[recurring contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/RECURRING-MATCH-PLANS-API-CONTRACT.md)

## Page defaults

| Setting | Value |
|---------|-------|
| Default page size | **20** per page |
| Available page sizes | 10, 20, 50 |
| Default timeframe | **Upcoming** |
| Default status filter | All |

## Two filter rows, not one

The timeframe (`upcoming` / `past`) and the status filter (`all` / `PENDING` / `CONFIRMED` /
`CANCELLED`) are separate bars. Merging them would offer "Past + Confirmed" as a peer of "Upcoming",
which reads as a status.

Both are sent to the API rather than applied here — the list is paginated, so filtering after the
page is built shows however many rows survived while `totalElements` keeps counting the ones that
did not. Changing either resets to page 0.

**Upcoming is the default** because a plan whose kickoff has passed can no longer be confirmed,
cancelled or generated from, and listing it beside next Friday's put dead entries above live ones —
the longer a group plays, the more of them there are. Ordering is the server's: nearest first in
both directions, since `1.3.0`.

## Role gates

| Action | Roles |
|--------|-------|
| View plans, RSVP for yourself | All |
| Bring / remove a guest | Any member with a linked player, while the poll is open and a spot is unfilled — **state, not role** |
| Create a plan, or a weekly run | `MANAGER` |
| Edit, confirm, cancel, override another player's availability | `MANAGER` |
| Record the pitch cost | `MANAGER` or `ORGANIZER` |
| Delete a plan | `GROUP_ADMIN` |

Deleting is the one control gated above the rest, which is worth knowing before assuming the modal
is uniformly `MANAGER`.

> The create FAB was `MASTER_USER`-only for a while, which hid it from administrators the endpoint
> had always accepted. It reads `MANAGER` now — the same thing the server gates on.

## Plan creation

One FAB, one modal, and the **same modal creates a run**. `CreateMatchPlanModal` posts to
`/api/match-plans` normally, and to `/api/match-plans/recurring` when **Repeat weekly** is ticked.

A second modal was the obvious alternative and would have been wrong: every field except the count
means the same thing for one plan and for eight, which is exactly how the endpoint is designed. Two
modals would have been two ways to say one thing.

### The weekly run

| Control | Behaviour |
|---------|-----------|
| **Repeat weekly** checkbox | Reveals the rest. Unticked, nothing about the form changes |
| **How many weeks** | 2–120, defaulting to 4. Validated in the resolver, not coerced — a number input yields `""` when cleared, and a coercing schema turns that into `0`, failing a minimum the manager never tripped |
| Preview line | "8 plans, the last one on Fri, Oct 23, 2026" |
| Deadline hint | Shown only when a confirmation deadline has been set, since that is when the lead-time rule means anything |
| Submit button | Becomes "Create 8 Plans" |

**The count is echoed in the button, and the last kickoff shown before submitting,** because one
click writes every plan in the run. That is the last moment to notice it says twelve rather than two.

**The preview steps by seven days of absolute time**, which is what the server does — including
across a daylight-saving boundary, where a 19:00 fixture becomes 18:00 locally. Adding weeks in
local time would be friendlier and would show a date the backend is not going to write.

### When the run is refused

The horizon (`MATCH_PLAN_RECURRENCE_MAX_MONTHS`) sits behind the platform-operator grant, so **the
client cannot read it** and does not try to pre-empt it. A run that overruns it comes back `400`
with a message naming how many occurrences would have fitted, and that message is shown **verbatim**
in the notification and in the form's error strip. A generic "failed to create" would leave the
manager guessing at a number the server has already counted. Non-400 failures fall back to a
translated string.

## Plan detail modal

`MatchPlanDetailModal` re-reads the plan by id (`useMatchPlan`) and prefers that over the card's copy,
so the modal reflects a plan somebody else just changed. It carries: your own RSVP, the confirmation
list with starters and reserves, guests, the pitch cost section, the plan-status actions, inline edit,
and delete.

The starter/reserve split, `expired`, `generatable` and `cancellable` all come **from the server** —
the UI branches on the flags rather than re-deriving them, which is what keeps it from drifting from
the server's rule.

## File map

| Layer | File |
|-------|------|
| Page component | `src/features/matchPlans/MatchPlansPage.tsx` |
| Plan card | `src/features/matchPlans/MatchPlanCard.tsx` |
| Create modal (single **and** weekly run) | `src/features/matchPlans/CreateMatchPlanModal.tsx` |
| Detail modal | `src/features/matchPlans/MatchPlanDetailModal.tsx` |
| Inline edit form | `src/features/matchPlans/MatchPlanEditForm.tsx` |
| Guests | `src/features/matchPlans/AddGuestModal.tsx` |
| Pitch cost | `src/features/matchPlans/PlanCostSection.tsx` |
| Data hooks | `src/hooks/matchPlan/useMatchPlans.ts` |
| Service | `src/services/matchPlanService.ts` |
| Types | `src/types/matchPlan.ts` |
| Tests | `src/tests/matchPlans/` |

`useCreateRecurringMatchPlans` invalidates the whole `['match-plans']` key rather than seeding rows
into it: a run lands anywhere across a paginated, timeframe-filtered, kickoff-ordered list, so there
is no single page an insert would be correct for. The server evicts its own plan cache wholesale for
the same reason.

## i18n keys (matchPlans namespace)

All strings live under `matchPlans` in each `locales/<lang>/common.json` — `en`, `pt` **and** `es`. A
missing key reaches production as the raw key.

Notable groups:

- `matchPlans.form.*` — the create modal, single and recurring
- `matchPlans.form.recurring.*` — toggle, count label, preview line, submit label, success and error
- `matchPlans.form.errors.occurrences_range` — sits with the other resolver errors, because the form
  renders those by key: `t('matchPlans.form.errors.' + error.message)`
- `matchPlans.detail.*` — the detail modal, RSVP and status actions
- `matchPlans.guests.*` · `matchPlans.waitlist.*` — guests, starters and reserves
