# Group Billing (Stripe Subscriptions) — Technical Specification

**Date:** 2026-08-01
**Status:** DRAFT (2026-08-01) — Phase 5b, **design only by owner decision**: 5a ships with all groups free; nothing here is scheduled until 5a-4 has real groups on it
**Priority:** MEDIUM (the commercial step, deliberately after the product step)
**Estimated Effort:** L (≈4–5 days backend, ≈3 days frontend) — held loosely at design stage
**Depends on:** `GROUP-ONBOARDING-PLAN.md` — you cannot bill groups that cannot exist
**Contract:** `docs/api/BILLING-API-CONTRACT.md` (new, when implemented)

---

## 1. Requirement Summary — and the boundary, first

**No money moves through this application, and billing does not change that.** The match-fee
ledger records debts between friends and money that moved over MB WAY between phones; that stays
exactly as V19 built it. What this spec adds is the **operator's own revenue**: each group pays
the platform operator a subscription, via Stripe, for the service of hosting their group. The
roadmap warned "don't conflate them" — they share no tables, no services, no UI, and the words
"charge"/"payment" keep meaning the ledger; billing says "subscription"/"invoice".

Groups from 5a are **grandfathered free** (BR-B1) — early groups took the product risk with us.

---

## 2. Scope (when implemented)

| In | Out |
|----|-----|
| Stripe subscription per organization; Checkout + customer portal | Any card data touching our servers (Stripe-hosted surfaces only) |
| `org_subscriptions` + entitlement state on the org | Match-fee ledger changes of any kind |
| One enforcement point: org status → AuthGuard gate 4 billing wall | Per-seat pricing (group-sized tiers first; revisit with data) |
| Webhook endpoint (ADR-002 after-commit pattern, signature-verified) | Stripe Connect / player-to-group payments (explicitly not this — that is the roadmap's separate warning) |
| Free tier + grandfathering + dunning states | In-app tax/invoice rendering (Stripe portal does it) |
| Creation codes → promo/trial codes | Per-group theming and other plan-gated features (a later tier decision) |
| Platform operator console (orgs, subscriptions, codes) | |

## 3. Model decision: subscription state lives on the organization, enforced at membership resolution — not per-endpoint checks

**Chosen:** `org_subscriptions(org_id, stripe_customer_id, stripe_subscription_id, status,
current_period_end, …)` (V30+, prose headers as ever), mirrored into a coarse
`organizations.status` (`ACTIVE` / `PAST_DUE` / `SUSPENDED`). **The single enforcement point is
where the request already resolves the membership** (enforcement plan §3): a suspended org
resolves with a `suspended` flag → API answers 402-family on writes, frontend AuthGuard gate 4
shows the billing wall (reads stay available in grace — losing your own match history because a
card expired is hostage-taking, not dunning).

**Rejected: per-endpoint entitlement checks** — a scattering of gates that drifts; the
membership resolution runs on every request already, so the org's status rides along for free.
**Cache note:** entitlement state is deliberately **not** in Caffeine — membership resolution is
a per-request DB read today (the design D4 bought), so suspension takes effect on the next
request, not after a 10-minute staleness window (ADR-003's accepted staleness is fine for league
tables, wrong for entitlements — this is why the check lives where it lives).

## 4. Model decision: Stripe-hosted everything — not Elements, not stored cards

Checkout Sessions for subscribe, the customer portal for card/plan/cancel, webhooks for truth.
The operator becomes a merchant for **subscriptions only**; the app never sees a PAN, and PSD2/
SCA is Stripe's problem. Webhooks follow the ledger plan §14's own sketch: a signature-verified
unauthenticated endpoint (joins `PUBLIC_PATHS` with verification, the one new public path),
idempotent on `UNIQUE(event_id)`, processed after-commit in the ADR-002 shape, updating
`org_subscriptions` then the org mirror. Frontend needs CSP additions (`connect-src`/`frame-src`
for Stripe) — noted against the build-time CSP constraint.

## 5. Dunning, tiers, codes

- **States:** `TRIALING → ACTIVE → PAST_DUE (grace, writes ok, banner) → SUSPENDED (reads only)
  → CANCELED (export honoured forever — erasure rights don't lapse with payment)`.
- **Tiers at launch:** Free (grandfathered + trials) and one paid tier. Feature-gating by tier is
  deliberately deferred; the AI-match-reports cost ceiling (ADR-004's ~$130/yr per hundred
  groups figure — the repo's one unit-economics number) is the obvious first paid-tier candidate.
- **Creation codes become promo/trial codes** — the 5a-4 gate converts rather than dies.
- **Platform console** (platform-admin grant): org list with subscription state, code minting,
  manual comp/extend. This is where the operator surface deferred since the enforcement plan
  finally lands.

## 6. Privacy

Billing data is personal data (the house checklist gains its first guidance for it): Stripe ids
and subscription state enter the data table; export includes the org's billing state for org
ADMINs; **Stripe joins the sub-processor list the privacy plan created empty** — the privacy
page's "shared with nobody" claim dies the day this ships, by design, in writing. Erasure: an
org's deletion schedules Stripe customer deletion; a person's erasure is untouched (billing
attaches to orgs, not people).

## 7. Test plan (sketch — firmed at implementation)

| Area | Cases |
|------|-------|
| Webhooks | Signature reject; idempotent replay; out-of-order events; each transition updates org mirror |
| Enforcement | PAST_DUE writes + banner; SUSPENDED read-only + wall; reactivation next-request |
| Grandfathering | 5a orgs stay ACTIVE with no subscription row |
| Isolation | Billing state of org A invisible to org B (extends `TenantIsolationIT`) |
| Frontend | Gate 4 states; portal round-trip (Stripe test clock) |

## 8. Order of work (when scheduled)

1. V30+ migrations + webhook skeleton behind a feature flag (ADR-004's flag pattern).
2. Checkout/portal round-trip in test mode; enforcement point; gate 4.
3. Codes conversion + console; grandfathering sweep; privacy/docs updates ship in the same
   release (the STATUS.md lesson: the follow-through *is* the feature).

## 9. Breaking changes

- [x] **None at introduction** — every existing group is grandfathered ACTIVE.
- [ ] **The deliberate one:** new groups require a subscription or trial after this ships. That
      is the point.
