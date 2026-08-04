# Incident Report — `GET /api/users/me` returned 404 to accounts with no group

**Date:** 2026-08-04
**Severity:** HIGH (an entire class of account could not use the application)
**Status:** RESOLVED — `FootMania-Back#160`, `FootMania-Simple-Front#18`
**Reporter:** Owner, from production

---

## 🚨 Issue Summary

A platform operator logged in successfully and then received **404** from `GET /api/users/me`.

The endpoint was telling a user that they do not exist — **while they were reading their own
record**.

---

## 🔍 Root Cause

`UserService.getUser` routed through `findOrThrow`:

```java
AppUser user = userRepository.findById(id).orElseThrow(...);   // found — the row exists

if (!membershipRepository.existsActiveByUserIdAndTenantId(
        id, TenantContext.currentTenantOrNull())) {            // null tenant → matches nothing
    throw ResourceNotFoundException.of("User", id);             // → 404
}
```

`findOrThrow` answers *"may this group see this account"*. That is the right question for
`GET /api/users/{id}` about **somebody else** — it is the check that stands in for a `tenant_id`
column on an account, and it is what stops one group's administrator reading another group's people.

It is the **wrong question entirely** when the subject is the caller. An account is always entitled
to read its own record, and routing `/me` through the group check made the answer depend on a group
the caller may not have.

### The second failure, which the first one masked

```java
@Cacheable(value = "users",
           key = "T(...TenantContext).currentTenant() + ':' + 'user-' + #id")
```

`currentTenant()` **throws** when unbound (deliberately — see its javadoc). Cache keys are evaluated
by Spring's cache interceptor **before the method body runs**, so this would have raised
`IllegalStateException` from outside anything the service could catch.

**Fixing only `findOrThrow` would have turned the 404 into a 500.** The membership check happened to
be reached first only because... it was not: the throw would have come first in production, where
caching is on. The 404 was reported because the reporter's path hit the branch that returns before
the cached call. Either way both had to be fixed together.

---

## 🧨 Who was affected

| Account | Affected | For how long |
|---|---|---|
| Platform operator | Yes | Permanently — the grant is defined as belonging to no group |
| Registered, not yet in a group | Yes | Until they joined or founded one |
| Member of exactly one group | No | Single-membership fallback binds a tenant |
| Member of several groups | Yes, until they picked one | The picker paths are tenant-agnostic; `/me` is not |

`POST /api/users/{id}/change-password` carried the identical bug for one's own account — an operator
could not change their own password.

---

## ✅ Resolution

`UserService.getSelf` — uncached, no group check — now serves `/me`, `/users/{id}` when the id is the
caller's own, and change-password for one's own account. The cache key on `getUser` moved to
`currentTenantOrNull()` regardless, because a throwing key expression is a 500 nobody can catch.

Not cached, deliberately: it has no tenant to key by, and keying it by account would put a second key
shape in a cache whose last key expression was the bug. The saving would be one primary-key lookup.

---

## 🧠 The lesson, and what was done about it

**The bug is not the endpoint. It is the ambiguity.**

An account that is sometimes tenant-bound and sometimes not means every question the code asks about
a caller — *which group is this*, *what may they see here* — has a second answer depending on which
hat they were wearing. `/me` was the first place that surfaced. It would not have been the last.

So `V35` states the rule that removes the ambiguity rather than only patching the symptom:

> **An account is either a platform operator or a member of groups, never both.**

Somebody who needs both holds two accounts. Enforced in `PlatformGuard.assertNotOperator` on the only
two paths that grant a membership, and checked *before* the single-use creation code or invite is
consumed so a refusal costs neither.

The reverse direction is not enforceable in the application — the operator grant is issued by
hand-written SQL by design (V28), so there is no endpoint to guard. The `platform_admins` table
comment now carries the check for whoever runs that `INSERT`.

---

## 🧪 Why the regression test is an integration test

`PlatformOperatorAccountIT` — and it had to be, for the cache half.

Cache keys are evaluated by an interceptor that only exists on a **proxied bean in a real context**.
Every unit test in the suite is Mockito `@InjectMocks`, which constructs the service directly and
never builds the proxy; the `test` profile sets `spring.cache.type=none` on top of that. A unit test
passes with the key expression present, absent, or misspelled.

The test was verified to actually catch the bug: reverting the key to `currentTenant()` makes it
fail, and it fails on the *right* assertion — `ResourceNotFoundException` rather than
`IllegalStateException`. A bare "does not throw" would have been satisfied by neither, and
`isInstanceOf(Exception)` by both.

---

## 📌 Follow-ups

- **`GET /api/users/me` has no contract file of its own.** It is covered by `API_REFERENCE.md` and now
  by `PLATFORM-OPERATOR-ACCOUNTS-API-CONTRACT.md`, but the user surface predates the per-surface
  contract convention. Worth folding in when `docs/api/` is next tidied.
- **`PLATFORM-CONSOLE-PLAN.md` rung 0 is now partly built** — the console is an operator's landing
  screen rather than a corner of Settings. The plan's own advice still stands: issue the first codes
  and let the questions actually asked pick from rung 1.
