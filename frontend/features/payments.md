# Payments Feature

**Status:** ✅ Shipped · written 2026-08-05

What you owe, and — for an organiser — what everybody owes. Route `/payments`, any authenticated
user; the roster half is gated inside the page.

Backend semantics (what a charge is, when one is raised, how a balance is derived, what delegation
means to the ledger) live in the
[payments contract](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PAYMENTS-API-CONTRACT.md)
and the [ledger plan](../../backend/plans/MATCH-FEE-LEDGER-PLAN.md). **This document covers the
page only** — it was written to close the "no FE feature doc" gap in
[feature-status](../../product/feature-status.md), not to restate the contract.

## Your balance comes first, whoever you are

An organiser is usually also a player. Putting the roster table above their own figure buries the
one row that is about them, so the page is always: your balance, your history, then — if you have
`ORGANIZER` — everybody else's.

The balance card carries the sign in words rather than only in colour: **You owe** / **In credit** /
**All settled**, with the absolute amount beneath it. The hint says the app only keeps the record —
money moves by MB WAY, outside the system.

## 409 is a state, not a failure

The ledger is keyed on the *player*, so an account with no linked player has nothing to show rather
than something broken. `useMyLedger` answering `409` renders "Link your account to a player to see
your balance"; every other error renders the standard error state. `src/tests/payments/` pins the
distinction.

## The two sides are interleaved here, not on the server

`PlayerLedgerDTO` carries `charges` and `payments` as two whole arrays. `LedgerHistory` merges them
into one list sorted by date, descending.

**Dated on `paidAt` for a payment and `createdAt` for a charge.** A payment is recorded after the
fact, sometimes days later, so its creation time is not when it happened. Listing the two arrays one
after the other — which is what this did originally — put a payment made in March below a charge
from last week, and a ledger only reads correctly in the order the money actually moved.

**Paging is therefore client-side, five to a page.** There is no page to ask the server for: the two
sides would have to be paged in step to merge correctly and they are not the same length. Sizes are
5 / 10 / 25.

A row shows who handed the money over when it was somebody else (`paidByPlayerName`) — "I never paid
that" and "my mate paid that for me" are different readings of the same row. A voided entry is
struck through rather than hidden: *charged and then waived* is a different fact from *never
charged*, and the ledger is a record.

## The roster, and who answers for whom

`BalancesTable` lists everybody, most in debt first, behind the `ORGANIZER` gate. Settled players are
hidden unless "Show settled" is ticked — the list exists to be worked down.

**Players with no linked account are included and chipped `No account`.** They are charged like
everybody else and are exactly the people most likely to be forgotten; the chip is what tells the
organiser this debt needs a message rather than a notification. Amber, not red: it is a fact about
how to reach them, not a problem with them.

### Delegation groups

Where one player answers for another's fees, the covered debtors are shown **under their payer** in a
bordered card with a combined figure, because that is the amount actually being asked for. Each
debtor keeps their own row and their own number underneath — the breakdown is the whole point, and a
combined total that hid it would be a worse tool than the one it replaced.

`groupByPayer` (in `src/types/payment.ts`) does the grouping, and it runs over the **full** balance
list rather than the filtered one: a payer who is square themselves still answers for people who are
not, and dropping them would strand their debtors.

The group card is one grid — name, amount, action — so the payer's total sits directly above the
shares it is the sum of. See rule 8 in the frontend
[styling guide](https://github.com/ricsnsuka/FootMania-Simple-Front/blob/main/docs/guides/styling.md):
this card is where that rule was found, with three debtors owing an identical €4.30 showing it at
three different offsets.

## Money is integer cents everywhere

`formatCents` divides by 100 once, at the render edge, and everything upstream stays an integer.
Amounts are `tabular-nums` and right-aligned in a fixed column shared by the tables and the group
cards, so figures line up on the decimal point across the whole page.

## File map

| Layer | File |
|-------|------|
| Route | `src/app/(app)/payments/page.tsx` |
| Page component | `src/features/payments/PaymentsPage.tsx` |
| Ledger table + cards | `src/features/payments/LedgerHistory.tsx` |
| Roster and delegation groups | `src/features/payments/BalancesTable.tsx` |
| Record a payment | `src/features/payments/RecordPaymentModal.tsx` |
| Set / change a payer | `src/features/payments/SetDelegateModal.tsx` |
| Data hooks | `src/hooks/payment/usePayments.ts` — `useMyLedger`, `useBalances`, `useRecordPayment` |
| Types, `formatCents`, `amountOwed`, `groupByPayer` | `src/types/payment.ts` |
| Tests | `src/tests/payments/` |

Below 640px both tables are replaced by cards, the same pattern as the users and players lists.

## i18n keys (`payments` namespace)

All strings exist in `en`, `pt` **and** `es`. Notable groups:

- `payments.youOwe` · `inCredit` · `settled` · `payHint` · `noLinkedPlayer`
- `payments.columns.*` — date, detail, amount, player, balance, action
- `payments.delegate.*` — the grouping, `payingFor`, `ownShare`, `recordForGroup`, and the modal
- `payments.notContactable` — the "No account" chip and its hint
