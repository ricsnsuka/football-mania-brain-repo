# Password Recovery

Routes: `/forgot-password`, `/reset-password/<token>` · Auth: **public** ·
Admin surface: `EditUserModal` → *Send a password reset link*

**Added in:** 2.2.0 · **Status:** ⚠️ **cut on `next`, not deployed** ·
**Backend contract:** [`docs/api/PASSWORD-RESET-API-CONTRACT.md`](https://github.com/ricsnsuka/FootMania-Back/blob/main/docs/api/PASSWORD-RESET-API-CONTRACT.md)

Before 2.2.0 there was no way back into an account whose password nobody remembered. The only
recovery was `UPDATE users SET password = …` by hand, which made whoever runs the deployment the
support desk and needed a bcrypt hash produced off-system.

---

## Two ways in, one way through

Recovery splits into **issuance**, **delivery** and **redemption**, and only delivery needs email.
That is why there are two ways to mint a token and one way to redeem it:

| Who | Endpoint | Auth |
|---|---|---|
| The account holder | `POST /api/auth/password-reset/request` | public |
| A group admin, on their behalf | `POST /api/users/{id}/password-reset` | authenticated |
| Either link, redeemed | `POST /api/auth/password-reset/redeem` | public |

**The admin path is not a workaround for a missing mail server so much as for a missing mailbox.**
Plenty of accounts here carry an address that was wrong on the day, or has not been read since.

`GET /api/auth/password-reset/availability` reports `emailEnabled`, and the login page reads it —
because a "Forgot password?" link to a form that silently does nothing is worse than no link. It
still shows the link either way; see [below](#the-link-is-always-offered).

---

## Files

```
src/app/(auth)/forgot-password/page.tsx        Request a link
src/app/(auth)/reset-password/[token]/page.tsx Redeem one
src/components/auth/ForgotPasswordForm.tsx     The request form
src/components/auth/ResetPasswordForm.tsx      The redeem form, and the pre-check
src/components/auth/LoginForm.tsx              Carries the entry point
src/features/users/IssuePasswordReset.tsx      The admin surface
src/hooks/auth/usePasswordReset.ts             React Query wrappers
src/services/passwordResetService.ts           API calls — all but one `skipAuth`
src/types/passwordReset.ts                     DTOs
```

**Everything except the admin call is `skipAuth`**, and that is not merely tidiness: the whole
premise is a caller who cannot log in. Sending a stale token would at best be ignored, and at worst
trip the 401 handler that wipes the store out from under a page somebody is mid-way through.

---

## The token is a path segment, not a query parameter

`/reset-password/<token>`, following `/join/<token>`. Reading it with `useParams` also keeps the
route out of the Suspense boundary `useSearchParams` would force on a statically prerendered page —
for a value the form needs at first render.

---

## Three rules this UI has to keep

Each one is a server property surfacing, so breaking it here breaks the property.

### It never claims an email was sent

The confirmation says **"if that account exists"**, and the *same* confirmation is shown when the
request fails. The server answers `204` for an unknown account, a deactivated account, a request
inside the cooldown and a mail server that is down — identically, and asynchronously, so it is not
identical-but-slower either. That is what stops the endpoint being an oracle for which addresses are
registered, and **a UI that distinguished the two would give it away regardless of what the server
does**.

### It checks the link before showing the form

`GET /api/auth/password-reset/{token}` is called first, and **checking a link does not spend it** —
which matters because mail clients and link scanners prefetch URLs.

Asking somebody to compose a password and only *then* telling them the link expired is the worst
version of this screen: they have done the work, lost it, and the message arrives looking like their
password was rejected.

The status says nothing about whose account it is. A page reading *"set a new password for
someone@example.com"* hands that address to anybody the link reached.

### It shows the server's policy messages verbatim

They name the rule that failed. A generic string leaves people guessing at a password policy they
cannot see.

---

## `USED` and `SUPERSEDED` are rendered as different things

`PasswordResetTokenReason` is `EXPIRED | USED | SUPERSEDED | UNKNOWN`, and the middle two are
deliberately distinct all the way from the `used_at` / `superseded_at` columns in `V43` to the
string on the screen.

**"Already used" reads as *somebody got into my account*.** "A newer link replaced this one" is what
actually happened when you clicked the older of two emails — which is the ordinary outcome of
clicking "forgot password" twice while hunting through an inbox. Collapsing them raises a false
alarm about a compromise that did not occur.

---

## The success screen says the API tokens are gone

Redeeming **revokes the account's API tokens**. A reset is often the answer to a suspected
compromise, and a token minted under the old password is a standing credential no password change
would otherwise touch.

It is surprising, so the screen says so. Finding out later, in silence, is worse — particularly for
the one population that has these tokens, which is people running a watch shortcut on a match day.

⚠️ **Existing JWTs are *not* invalidated.** They are stateless and there is no revocation list. The
contract states it rather than glossing over it, and so does this page.

**Redeeming does not sign anybody in**, either. A forwarded link, or one sitting in a mail gateway's
log, must be a spent credential and not a session — so the page sends people to `/login`.

---

## The admin surface

*Send a password reset link* lives in `EditUserModal`, **behind a confirmation**. It mints a
credential that takes over an account and kills its API tokens, which should not be one stray click
away in a modal whose other buttons rename people.

A group admin may only reset an account when **every** group it belongs to is one they administer,
and never a platform operator's — enforced server-side. Without that rule, the admin of one group
could take over an account administering another: the privilege escalation across the tenancy
boundary that `V25`'s composite FKs exist to make unrepresentable, arriving through the front door.

The link comes back **once**, and is shown in full and **selectable as well as copyable** — the
clipboard API is unavailable in plenty of contexts, and a link you can see but not copy is still a
link you can read out. `emailed` says whether a copy also went to the account's address, which the
issuer needs before deciding whether relaying it is optional.

---

## The link is always offered

The login page shows *Forgot password?* even where the server cannot send email. That page is also
where somebody is told to ask an admin instead — **hiding the link would leave them nowhere to read
that**.

---

## Tests

| File | Covers |
|------|--------|
| `src/tests/auth/passwordReset.test.tsx` | The identical confirmation on success and failure, the pre-check before the form renders, `USED` vs `SUPERSEDED` as distinct messages, verbatim policy errors, the API-token warning on the success screen, and the admin flow's confirmation step |
