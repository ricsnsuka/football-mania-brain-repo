# Security Audit Report
**Date:** 2025-07-16
**Performed By:** Security Auditor Agent (Phase 7)
**Version Audited:** v4.x.x — `POST /api/players/{id}/link-me` feature addition
**Scope:** Targeted audit of the self-link endpoint and its supporting service layer.
  Full codebase CVE scan deferred (no new dependencies added).

---

## Executive Summary

The `POST /api/players/{id}/link-me` implementation is **secure**. All six primary security
concerns (IDOR, privilege escalation, ADMIN bypass, race condition, authentication
enforcement, and broader controller authorization) returned **PASS**. One low-severity
API-contract finding was identified and fixed: a missing handler for
`DataIntegrityViolationException` that would have caused concurrent race-condition
failures to surface as HTTP 500 instead of HTTP 409.

---

## CVE Findings

> **Not applicable for this audit.** No new dependencies were added.
> Previous full CVE scan: `SECURITY-AUDIT-2025-07-09.md`.

---

## Targeted Security Findings

### a. IDOR — User ID Injection ✅ PASS

| Property | Value |
|---|---|
| User ID source | `(UserPrincipal) authentication.getPrincipal()` — exclusively from JWT |
| Request body | None (`@RequestBody` absent) |
| Path variable | Only `{id}` = player ID — cannot influence `userId` used in service |

**Assessment:** There is no mechanism for a caller to inject a foreign `userId`. The
`authentication.getPrincipal()` is populated by `JwtAuthenticationFilter` from the
validated JWT and the live `UserDetails` record loaded from the database. The `principal.id()`
used in `PlayerService.linkMe()` cannot be tampered with via URL, request params, or request
body.

---

### b. Privilege Escalation ✅ PASS

**Assessment:** `linkMe()` calls `player.setUser(linkedUser)` and
`player.setUpdatedBy(principal.username())` only. The `AppUser.role` field is never read
for modification — it is read only to enforce the ADMIN block. Linking a player does not
change any role, grant any new authority, or affect the JWT claims used for subsequent
requests (the principal is re-loaded from the DB on every request).

---

### c. ADMIN_USER Bypass ✅ PASS

The guard is:
```java
boolean isAdmin = principal.authorities().stream()
    .anyMatch(a -> "ROLE_ADMIN_USER".equals(a.getAuthority()));
if (isAdmin) {
    throw BusinessException.forbidden("ADMIN_USER accounts cannot be linked to a player");
}
```

**Analysis:**
- Authorities originate from `UserDetailsServiceImpl.loadUserByUsername()`, which reads the
  role fresh from the database on every request (called inside `JwtAuthenticationFilter`).
- A token whose user has been promoted to ADMIN since issuance will be blocked on the very
  next request — no stale-principal bypass is possible.
- The string `"ROLE_ADMIN_USER"` exactly matches the authority string produced by
  `new SimpleGrantedAuthority("ROLE_" + user.getRole().name())` for `ADMIN_USER`.
- `MASTER_USER` is intentionally **not** blocked — consistent with spec (MASTER_USER is an
  elevated manager role but is a valid participant).

---

### d. Race Condition ✅ DB-SAFE / 🔧 FIXED (API contract)

**DB Level:** The `players.user_id` column carries a `UNIQUE` constraint declared in
`V1__initial_schema.sql` (line 33):
```sql
user_id BIGINT UNIQUE REFERENCES users(id) ON DELETE SET NULL
```
Two concurrent transactions that both pass the application-level duplicate checks will not
both succeed — the database guarantees only one write wins.

**Finding (LOW):** The `DataIntegrityViolationException` thrown by the losing concurrent
transaction had no handler in `GlobalExceptionHandler`. It would have fallen through to the
generic `Exception` handler and returned HTTP 500, leaking an internal error message to the
caller.

**Fix Applied:** Added `DataIntegrityViolationException` handler to `GlobalExceptionHandler`:
```java
@ExceptionHandler(DataIntegrityViolationException.class)
public ResponseEntity<ApiError> handleDataIntegrity(
        DataIntegrityViolationException ex, HttpServletRequest req) {
    log.warn("Data integrity violation at {} {}: {}",
        req.getMethod(), req.getRequestURI(), ex.getMostSpecificCause().getMessage());
    return error(HttpStatus.CONFLICT, "Request conflicts with existing data", req, null);
}
```
This also benefits any future endpoint that may encounter DB-level constraint violations.

---

### e. Authentication Enforcement ✅ PASS

| Layer | Rule |
|---|---|
| `SecurityConfig` | `anyRequest().authenticated()` — global catch-all |
| `PlayerController.linkMe` | `@PreAuthorize("isAuthenticated()")` — explicit |

**Assessment:** The `@PreAuthorize` annotation is technically redundant with the global
`anyRequest().authenticated()` rule; however it is not a security weakness. Explicit
method-level annotation is preferred for defense-in-depth — it makes authorization intent
self-documenting and keeps the endpoint protected if the global config is ever relaxed.
No unauthenticated access is possible.

---

### f. Broader PlayerController Authorization Review ✅ PASS

| Endpoint | Annotation | Required Role | Status |
|---|---|---|---|
| `GET /api/players` | `@PreAuthorize("isAuthenticated()")` | Any auth user | ✅ |
| `GET /api/players/{id}` | `@PreAuthorize("isAuthenticated()")` | Any auth user | ✅ |
| `POST /api/players` | `@PreAuthorize("hasRole('ADMIN_USER') or hasRole('MASTER_USER')")` | ADMIN or MASTER | ✅ |
| `PATCH /api/players/{id}` | `@PreAuthorize("hasRole('ADMIN_USER') or hasRole('MASTER_USER')")` | ADMIN or MASTER | ✅ |
| `PATCH /api/players/{id}/status` | `@PreAuthorize("hasRole('ADMIN_USER') or hasRole('MASTER_USER')")` | ADMIN or MASTER | ✅ |
| `DELETE /api/players/{id}` | `@PreAuthorize("hasRole('ADMIN_USER')")` | ADMIN only | ✅ |
| `POST /api/players/{id}/link-me` | `@PreAuthorize("isAuthenticated()")` | Any auth user (ADMIN blocked in service) | ✅ |

All 7 endpoints in `PlayerController` have correct, appropriate `@PreAuthorize` protection.
No unguarded mutating operations were found.

---

## Full Findings Summary

| # | Area | Severity | Finding | Status |
|---|---|---|---|---|
| 1 | IDOR | — | userId sourced exclusively from JWT principal | ✅ PASS |
| 2 | Privilege Escalation | — | `linkMe()` does not modify user role | ✅ PASS |
| 3 | ADMIN Bypass | — | Authority check correct; loaded fresh from DB | ✅ PASS |
| 4 | Race Condition (API contract) | 🟢 LOW | `DataIntegrityViolationException` unmapped → HTTP 500 | ✅ FIXED |
| 5 | Auth Enforcement | — | Dual-layer is defense-in-depth, not a gap | ✅ PASS |
| 6 | Controller Auth Breadth | — | All 7 PlayerController endpoints properly guarded | ✅ PASS |

---

## Files Changed

| File | Change |
|---|---|
| `src/main/java/pt/rics/demo/football/exception/GlobalExceptionHandler.java` | Added `DataIntegrityViolationException` → HTTP 409 handler |

---

## Recommendations

1. 🟢 **LOW** *(Fixed)* — `DataIntegrityViolationException` now maps to HTTP 409 Conflict,
   preventing 500 responses on concurrent `link-me` race conditions. No further action needed.

2. 🟢 **INFO** — Consider reducing JWT token lifetime or implementing a token revocation
   mechanism (previously noted in `SECURITY-AUDIT-2025-07-09.md`). No change for this release.

---

## Resolved Items (from previous audit)

*No items from `SECURITY-AUDIT-2025-07-09.md` are applicable to this feature addition.*

---

## Conclusion

The `POST /api/players/{id}/link-me` endpoint is **safe to document and release**.
All critical and high-severity security vectors are closed. One low-severity API-contract
issue was found and fixed during this audit.

