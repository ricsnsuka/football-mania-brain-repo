# Groups — the frontend

How a person gets into a group, moves between groups, and how the two people who hand out access
do it. The schema and enforcement behind this are in
[architecture/multi-tenancy.md](../../architecture/multi-tenancy.md) and the Phase 5a plans; this
is the part with buttons.

**Added:** 2026-08-02 — choke points in `fcc6c59`, the surfaces below in `4c5c012`, creation codes
and the whole-page baselines in `18e4348`
**Backend contracts:** [TENANCY](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/TENANCY-API-CONTRACT.md) ·
[GROUP-INVITES](https://github.com/ricsnsuka/FootMania-Back/blob/master/docs/api/GROUP-INVITES-API-CONTRACT.md)

---

## The shape of it

An account and a group are separate things. Registering gets you the first and none of the second:
`POST /api/users/register` deliberately creates no membership, so a fresh account arrives with
`memberships: []` and belongs nowhere. There are exactly two ways in.

| Route in | Who starts it | Screen |
|---|---|---|
| An invite link | a group `ADMIN` | `/join/<token>` |
| A creation code | the platform operator | `/groups`, "I have a creation code" |

Both are deliberate acts by somebody who already has standing. There is no open self-serve path,
and `group_creation_codes` ships empty — **the endpoints existing is not the flip; a code existing
is.**

---

## `/groups` — the picker

`AuthGuard`'s third gate sends anybody with no active group here, and it sits outside the `(app)`
route group because that layout renders the Navbar, which shows the name of the group they do not
have yet.

Three states, one screen: several memberships means pick one; none means wait for an invite or
redeem a code; exactly one never arrives here at all, because `resolveActiveGroup` selects it
silently. That last case is why onboarding was not a breaking change — every account that predates
it holds exactly one membership and sees none of this.

It re-reads the membership list on mount rather than trusting login's snapshot. The two ways that
list goes stale — being invited into a group, being removed from one — both happen while somebody
is looking at this exact screen.

## `/join/<token>` — accepting an invite

**Told first, asked second.** The preview is fetched before anything else, including for somebody
with no account, because `GET /api/invites/{token}` is public precisely so that being asked to
register before being told what you are joining is not the first thing that happens. It discloses
the group's name and the grants on offer, and nothing about the group's size, members or contents.

Signed in, one button accepts, re-reads the memberships and makes the new group active — somebody
accepting an invitation meant to go there. Signed out, it routes to `/login?next=/join/<token>` so
accepting resumes on the invite rather than dropping the person on a dashboard holding a link they
may no longer have.

> **`?next=` is guarded.** Only a same-origin path is honoured; `//host`, `https://host` and
> `javascript:` all fall back to `/dashboard`. A login URL that redirects wherever the query string
> says is an open redirect, and one that genuinely begins with this app's own login URL is exactly
> what makes a phishing link look trustworthy.

A token fails in four distinguishable ways — never issued, already used, expired, already a member
— and only the server knows which. Each has a different next step, so **its message is shown
verbatim** rather than replaced with one generic refusal.

## The switcher

With more than one membership, the Navbar brand slot becomes a `<select>` of the groups. What
matters to somebody mid-session is which group they are acting in, not which product they are
using; the platform name recedes to the login screen and the footer. With one membership it is
simply the group's name, which is what every pre-onboarding account sees.

Switching navigates to the dashboard rather than staying put: the current page may be one the
caller has no grant for in the group they just moved to, and landing on a permission redirect is a
worse answer than starting from home. The store erases the query cache on the way.

**Roles are per group, and the UI already knew that.** `user.roles` is kept equal to the active
membership's grants, so roughly thirty `hasRole(user, 'ADMIN')` call sites mean something narrower
than they used to without one of them being touched — an administrator of the Tuesday lot acting in
the Sunday league holds nothing there, and every one of those sites now says so.

---

## Handing out access

### Invites — Users page, group `ADMIN`

They live there because that page answers "who is in this group", and bringing somebody new in is
the same question in a different tense.

The **token is shown on every listing**, not only at creation. An admin who cannot re-read the link
they just made cannot send it twice, and the workaround for that is minting another, which leaves
live tokens behind. Copy writes the whole `/join/<token>` link and confirms by changing its own
label for two seconds — a toast per copy is noise, and an administrator minting three invites in a
row would collect three of them.

The grants belong to the **invite**, not the inviter — an `ADMIN` may mint a `MANAGER` invite, and
a stolen link grants exactly what it says, once. Ticking nothing sends `roles: []`, which the
server reads as a plain member.

An accepted invite offers no withdraw button. The membership it created still exists, and deleting
the row would erase how somebody came to be in the group; the server refuses it with `409`.

### Leaving — Settings → Privacy

The other end of membership, and 5a-3's half of it: one row per group with a leave button, beside
the export and erasure controls rather than on a screen of its own, because leaving one group and
erasing yourself from the platform are the same question at two scales.

Leaving does not name a replacement group. The store re-resolves, so leaving the group you were
acting in either auto-selects the single one you have left or drops you at the picker — staying
pointed at a group you are no longer in would 404 every request with nothing to act on.

### Creation codes — Settings, platform operator

Behind `user.platformAdmin`, which is **a different grant, not a stronger one**. Every founder will
hold group `ADMIN`; deciding that another group may exist is a different job with a different blast
radius, so V28 gave it its own flat platform-level grant.

The section is hidden for everybody else and the endpoints refuse them regardless — a hidden
control is a usability decision and never a security one. A redeemed code cannot be withdrawn: the
group it produced still exists, and the row is the only record of who authorised it.

---

## What is not here

- **Removing somebody else** — changing another member's grants, or taking their membership away,
  is the Users page's existing edit modal. Only *leaving, yourself* is covered above.
- **Deep links per group** — a push notification from group B while group A is active opens in the
  active group. An accepted 5a limitation; the payload carries an advisory `groupId` for later.
- **Billing** — `GROUP-BILLING-PLAN`, on hold by owner decision. When it lands, creation codes
  become promo/trial codes rather than dead weight.
