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
| Week filter | Off until chosen; opens on the current week |

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

## The week filter

A third tab beside Upcoming and Past. Choosing it reveals a stepper — `‹`, the week's range, `›`,
and a disabled-when-you-are-already-there "This week" — and filters the list to one Monday-to-Sunday
week.

**It is not a timeframe the API knows about.** There is no `timeframe=week`. The page computes the
window itself and sends two instants:

```
GET /api/match-plans?from=2026-08-03T00:00:00.000Z&to=2026-08-10T00:00:00.000Z
```

**Because a week boundary is a local-calendar fact.** "This week" starts at a different instant in
Lisbon than in São Paulo, and only the browser knows which of them is asking. A server-resolved
`week` value would hand everybody Lisbon's week. `from` is inclusive and `to` exclusive, so
consecutive weeks tile exactly — a plan kicking off at the boundary belongs to the week starting
there and to no other.

**Choosing a week drops `timeframe` rather than combining with it.** The API intersects the two, so
`upcoming` plus this week would mean *the rest of* this week — quietly hiding the half already
played, from a control the reader used to ask for the whole of it. Switching back to Upcoming or
Past drops the window in turn.

### The daylight saving trap

`src/lib/week.ts` steps with `Date#setDate`, **never** by adding 7 × 24 hours. Across a daylight
saving change a week is 167 or 169 hours long, and the arithmetic version drifts the boundary an
hour into the neighbouring day — which, at exactly midnight, is a different day.

`src/tests/lib/week.test.ts` pins `Europe/Lisbon` rather than running in the machine's timezone: in
UTC every one of those assertions passes for the broken version too, so a suite that did not pin a
DST zone would be green and wrong. One test asserts the spring-forward week spans exactly 167 hours.

### Formatting the range

The label uses `Intl.DateTimeFormat.formatRange` (`formatDateRange` in `src/lib/formatDate.ts`),
not two formatted dates joined with a dash. Which parts collapse, and in what order, is the
locale's business: en-GB elides to `3 – 9 Aug 2026` and en-US to `Aug 3 – 9, 2026`. Hand-joining
produced `3 – Aug 9, 2026` — day-first output spliced into a month-first locale.

The label has a `min-width` so the arrows either side do not shuffle sideways as the range changes
width — `3–9 Aug 2026` against `28 Dec – 3 Jan 2027`. Rule 8 in the frontend
[styling guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/styling.md).

> ICU separates a range with THIN SPACE + EN DASH + THIN SPACE, not spaces and a hyphen. An
> assertion written with ordinary spaces fails with `expected '3 – 9 Aug 2026' to be
> '3 – 9 Aug 2026'`. The tests compare on collapsed whitespace.

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
| Week arithmetic | `src/lib/week.ts` — `startOfWeek`, `addWeeks`, `endOfWeek`, `weekWindow`, `isSameWeek` |
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
- `matchPlans.timeframeWeek` · `matchPlans.week.*` — the week tab, the stepper labels and the
  "Monday to Sunday" hint
