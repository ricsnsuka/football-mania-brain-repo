# Tenant Privacy (Erasure Fork & Controller Split) — Technical Specification

**Date:** 2026-08-01
**Status:** ✅ **BUILT 2026-08-02** — Phase 5a-3, all five steps of §9. Ships dark; **this
unblocks `GROUP-ONBOARDING-PLAN.md`**. Two departures from this spec, both deliberate: the
membership row is *deleted* rather than tombstoned (a tombstone retains the datum being erased and
would need a migration), and the per-group leave *list* in the UI waits for 5a-4's picker, which is
the endpoint that can enumerate memberships — the service call and copy are in place
**Priority:** HIGH — gates onboarding: the current erase path destroys cross-group data the moment two memberships exist
**Estimated Effort:** M (≈2 days backend, ≈½ day frontend)
**Depends on:** `TENANCY-ENFORCEMENT-PLAN.md` — a tenant-scoped erasure needs tenant scoping to exist. **Build that first.**
**Depended on by:** `GROUP-ONBOARDING-PLAN.md` (hard gate)
**Contract:** `docs/api/PRIVACY-API-CONTRACT.md` (updated in the same backend commit)

---

## 1. Requirement Summary

`PRIVACY_AND_DATA_PROTECTION.md` booked exactly this for Phase 5: *"If Phase 5 makes this a
self-serve product, this section needs revisiting with the tenant boundary in mind — the operator
of a hosted service and the group organiser are then different parties, and the controller/
processor split has to be written down."* This spec is that revisit — and it is a **feature**, not
a paragraph, because today's machinery is wrong for multiple memberships in ways that destroy
data:

- `eraseByUser` **deletes the `users` row**. With two memberships, erasure requested in group A
  cascades away the person's memberships, push subscriptions and mutes in group B — group B's
  member vanishes because group A processed a request.
- `exportForUser` assumes **one player per account** (`findByUserId` → single `Optional`) —
  mechanically patched in the schema rung, semantically redesigned here.
- "Another administrator must action this" (the admin self-erasure guard) must come to mean
  another administrator **of the same group**.

This rung ships dark: with one organization, leave-group and erase-platform coincide, and the new
fork is behaviourally today's behaviour.

---

## 2. Scope

| In | Out |
|----|-----|
| Erasure fork: leave-group vs erase-platform | Group creation/joining flows (`GROUP-ONBOARDING-PLAN.md`) |
| Membership-iterating export | Billing data privacy (`GROUP-BILLING-PLAN.md` — its tables, its checklist entries) |
| Same-tenant admin guards | FK-ifying audit columns (out of scope — see §5) |
| Controller/processor split, written down | Log-retention policy decision (flagged, operator task) |
| Two-tier privacy page (frontend) | DPA legal template drafting (operator + counsel) |

---

## 3. Model decision: erasure forks into leave-group and erase-platform — not per-request account deletion

**Chosen:** two operations with distinct semantics:

- **Leave-group** (`DELETE /api/privacy/me/memberships/{groupId}`, and the admin-actioned
  equivalent per player): tenant-scoped. Anonymises the person's `Player` row *in that tenant*
  (the existing anonymise-in-place machinery, unchanged — stats survive as the group's history,
  the person behind them does not), ends that membership, scrubs that tenant's audit references.
  Everything in every other group is untouched — **bit-identical**, and the test plan asserts it.
- **Erase-platform** (the existing `DELETE /api/privacy/me` reshaped): the account and everything
  hanging off it (push subscriptions, mutes, memberships) plus leave-group semantics in every
  group the person belongs to. **Only permitted when it is the last membership** — otherwise the
  API answers 409 listing the groups to leave first, and the UI offers the loop. Auto-offered as
  the natural final step of leaving your last group.

**Rejected:** keeping one erase-everything operation and documenting it. A person leaving one
five-a-side group has not asked to vanish from three others; GDPR's own scope is the processing
context. One operation with two blast radii chosen by circumstance is exactly the bug class the
guest-removal incident taught us to model explicitly.

## 4. Model decision: export iterates memberships — one document, per-group sections

`exportForUser` builds one export with a section per membership (group name, roles, joined date,
then that group's player profile, matches, goals, availability, ledger, delegations, badges,
votes — each resolved under that tenant's context), plus a platform section (account, devices,
mutes). `exportForPlayer` stays admin-actioned and **tenant-asserted** (the enforcement rung's
IDOR fix), producing only that group's section — a group admin can export what their group
processes, nothing more. Counterpart names inside a section (paid-by, delegation counterparts,
MVP votee) are by construction same-tenant now; the votes-received-are-counts-only rule survives
unchanged.

## 5. Audit columns stay usernames — with the fork made safe

Audit columns (`created_by`, `recorded_by`, `voided_by`, …) are `VARCHAR(50)` usernames, not FKs.
**Chosen:** keep them; leave-group scrubs them *within the leaving tenant's rows only*
(scrub queries gain a tenant predicate); erase-platform scrubs across all tenants — which is
correct precisely because usernames are globally unique (schema plan D3). **Rejected:** FK-ifying
audit columns — a schema-wide rewrite of append-only tables for a problem the tenant predicate
solves.

## 6. The controller/processor split, written down

- **Per group:** the group (via its ADMIN/ORGANIZER holders) is the **controller** of its
  competition data — it decides who is on the roster, what gets recorded, when someone is erased.
  The platform operator is the **processor**, acting on the group's instructions through the
  product's features. This inverts today's single-deployment framing.
- **Platform-level:** the operator is the **controller** of account data (credentials, email,
  push endpoints, memberships) and of platform operations (logs, backups).
- Deliverables: this analysis lands in `PRIVACY_AND_DATA_PROTECTION.md` (replacing the
  "you joined the group" lawful-basis section with the two-party version — noting the guest
  plan's data-minimisation argument survives, since a guest still joined nothing); a DPA
  template placeholder for the operator (legal review is an operator task, flagged not drafted);
  the sub-processor list section is created **empty** now so billing has somewhere to add Stripe.
- **Frontend:** `privacy/page.tsx` becomes two-tier — platform policy (operator identity, the
  resolved `[operator email]` placeholder, account data, sub-processors) + per-group controller
  framing ("your group's organisers control the competition data; direct sports-data requests to
  them, account requests to us"). Strings ×3 locales.

## 7. Business rules

| # | Rule | Notes |
|---|------|-------|
| BR-P1 | Leave-group touches exactly one tenant; other groups bit-identical | Asserted by test, not asserted by hope |
| BR-P2 | Erase-platform only on last membership; 409 otherwise, listing remaining groups | The UI turns the 409 into a guided loop |
| BR-P3 | Admin self-erasure guard is per-tenant | "Another administrator" = same group; last-admin-of-group check |
| BR-P4 | Export sections are membership-scoped; admin export is single-tenant | IDOR fix inherited from enforcement |
| BR-P5 | Anonymise-in-place survives unchanged inside each tenant | The group's history stays coherent — the V11 rule, per group |
| BR-P6 | Every new personal-data table = data-table row + export + erase + test | The house rule, restated for the chain |

## 8. Test plan

| Area | Cases |
|------|-------|
| Real persistence (house rule — both are delete paths) | Leave-group with two memberships: target group anonymised + membership gone, **other group's rows byte-compared unchanged** (incl. subscriptions/mutes survive); erase-platform on last membership: account + devices gone, per-group anonymisation applied |
| Fork guards | Erase-platform with two memberships → 409 + group list; leave-last-group offers/permits platform erase |
| Admin guards | Same-tenant admin can action; cross-tenant admin → 404; last-admin-of-group self-erase → 403 naming the rule |
| Export | Two-membership export has two group sections + platform section; admin export contains only their tenant's section |
| Dark acceptance | Single-org deployment: new endpoints behave as today's; suite unmodified |

## 9. Order of work

1. Backend fork (leave-group service + reshaped erase-platform) + guards + scrub predicates.
2. Export restructure.
3. Contract + `PRIVACY_AND_DATA_PROTECTION.md` rewrite (lawful basis, controller split, DPA + sub-processor placeholders).
4. Frontend privacy page two-tier + locales; PrivacySettings gains per-group leave actions.
5. Real-persistence tests green → **this unblocks `GROUP-ONBOARDING-PLAN.md`**.

## 10. Breaking changes

- [x] **None at one organization** — the fork degenerates to today's semantics.
- [ ] **Deliberate semantic change, stated:** once two memberships exist, "erase me" from inside
      a group means leave-group, and the old whole-account erase requires being down to one
      membership. The privacy page says so in plain words.
