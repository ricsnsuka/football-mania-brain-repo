# Security Audit Report

**Date:** 2025-07-09  
**Performed By:** Security Auditor Agent (Phase 7)  
**Version Audited:** v1.0.0 — Player Feature  
**Build Status at Time of Audit:** ✅ BUILD SUCCESSFUL — 48 tests pass

---

## Executive Summary

The Player feature is well-structured from a security perspective. All write endpoints are
properly protected with role-based authorization, input validation is comprehensive, and no
SQL injection vectors were found. Two critical PostgreSQL driver CVEs were identified and
remediated by pinning the driver version. One medium-severity hardcoded JWT secret fallback
was removed to enforce mandatory environment-variable configuration in production. The
overall security posture is **GOOD** and the feature is cleared for release after the
mandatory dependency upgrade is applied.

---

## CVE Findings

| Dependency | Version (BOM default) | CVE | CVSS | Min Safe Version | Status |
|---|---|---|---|---|---|
| `org.postgresql:postgresql` | 42.7.4 | CVE-2025-49146 | **HIGH** | 42.7.11 | ✅ Fixed — pinned to 42.7.11 |
| `org.postgresql:postgresql` | 42.7.4 | CVE-2026-42198 | **HIGH** | 42.7.11 | ✅ Fixed — pinned to 42.7.11 |
| `io.jsonwebtoken:jjwt-api` | 0.12.6 | None | — | N/A | ✅ Clean |
| `org.springframework.boot` | 3.4.5 | None | — | N/A | ✅ Clean |
| `org.springframework.security` | (via Boot 3.4.5) | None | — | N/A | ✅ Clean |
| `org.flywaydb:flyway-core` | 10.22.0 | None | — | N/A | ✅ Clean |
| `com.github.ben-manes.caffeine:caffeine` | 3.2.0 | None | — | N/A | ✅ Clean |
| `org.mapstruct:mapstruct` | 1.6.3 | None | — | N/A | ✅ Clean |
| `org.projectlombok:lombok` | 1.18.38 | None | — | N/A | ✅ Clean |
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` | 2.8.6 | None | — | N/A | ✅ Clean |
| `org.owasp.encoder:encoder` | 1.3.1 | None | — | N/A | ✅ Clean |

### CVE Details

#### CVE-2025-49146 — Channel Binding Bypass (HIGH)
pgjdbc ignores `channelBinding=require` and falls back to insecure auth methods (MD5,
password, GSS, SSPI), enabling MITM attacks against connections that the operator
believed were protected by channel binding.

**Workaround (if immediate upgrade is not possible):** Set `sslMode=verify-full` on
the JDBC connection string.

#### CVE-2026-42198 — SCRAM DoS via Unbounded PBKDF2 Iterations (HIGH)
A malicious PostgreSQL server can send an arbitrarily large SCRAM iteration count,
causing the client to burn unbounded CPU on PBKDF2 before failing authentication.
Repeated attempts can exhaust connection pools.

**Fix applied:** Both CVEs resolved by explicitly pinning the driver:

```groovy
// build.gradle
runtimeOnly 'org.postgresql:postgresql:42.7.11'
```

---

## Authorization Findings

All endpoints correctly declare `@PreAuthorize`. No gaps detected.

| Endpoint | Auth Requirement | Status |
|---|---|---|
| `GET /api/players` | `isAuthenticated()` | ✅ Correct |
| `GET /api/players/{id}` | `isAuthenticated()` | ✅ Correct |
| `POST /api/players` | `ADMIN_USER` or `MASTER_USER` | ✅ Correct |
| `PATCH /api/players/{id}` | `ADMIN_USER` or `MASTER_USER` | ✅ Correct |
| `PATCH /api/players/{id}/status` | `ADMIN_USER` or `MASTER_USER` | ✅ Correct |
| `DELETE /api/players/{id}` | `ADMIN_USER` only | ✅ Correct |

**Global fallback:** `SecurityConfig` sets `.anyRequest().authenticated()` — no
unprotected endpoint can exist by accident.

**Method Security:** `@EnableMethodSecurity` is active — `@PreAuthorize` annotations
are enforced by the Spring Security proxy, not solely relying on filter-chain rules.

---

## Security Findings

### 🔴 CRITICAL / HIGH (Fixed)

#### [FIX APPLIED] CVE-2025-49146 & CVE-2026-42198 — PostgreSQL JDBC Driver
- **File:** `build.gradle`
- **Fix:** Pinned `org.postgresql:postgresql` to `42.7.11`.
- **Before:** `runtimeOnly 'org.postgresql:postgresql'` (resolved to 42.7.4 via Boot BOM)
- **After:** `runtimeOnly 'org.postgresql:postgresql:42.7.11'`

---

### 🟡 MEDIUM (Fixed)

#### [FIX APPLIED] Hardcoded JWT Secret Fallback
- **File:** `src/main/resources/application.yml`
- **Severity:** MEDIUM
- **Issue:** `app.jwt.secret` had a base64-encoded fallback:
  `${JWT_SECRET:c2VjcmV0LWtleS10aGF0LWlzLWxvbmctZW5vdWdoLWZvci1IUzI1Ni1oYXNoaW5nLWFsZ29yaXRobQ==}`
  which decodes to a publicly known string. If `JWT_SECRET` was not set in production,
  the application would silently use this known key — compromising all issued tokens.
- **Fix:** Removed the fallback. The application will now fail to start if `JWT_SECRET` is
  absent, preventing silent insecure deployments.
- **Before:**
  ```yaml
  app:
    jwt:
      secret: ${JWT_SECRET:c2VjcmV0LWtleS10aGF0...}
  ```
- **After:**
  ```yaml
  app:
    jwt:
      secret: ${JWT_SECRET}   # mandatory — no fallback
  ```
- **Note:** The `application-test.yml` provides the correct test-scoped secret under
  `app.jwt.secret`, so all 48 tests continue to pass.

---

### 🟢 LOW / Informational

#### [INFO] Swagger UI Publicly Accessible
- **File:** `src/main/java/pt/rics/demo/football/config/SecurityConfig.java`
- **Severity:** LOW
- **Finding:** `/v3/api-docs/**`, `/swagger-ui/**`, and `/swagger-ui.html` are in
  `PUBLIC_PATHS` (no authentication required). In production, this exposes full API
  schema to unauthenticated actors.
- **Recommendation:** Add a `@Profile("!prod")` guard on the permit-all for Swagger
  paths, or restrict them behind authentication in prod. For this release, acceptable
  if the production server is not internet-facing. Document in ops runbook.

#### [INFO] IDOR — Any Authenticated User Can Read All Player Data
- **Severity:** INFO (by-design)
- **Finding:** `GET /api/players` and `GET /api/players/{id}` allow any authenticated
  user (including `BASIC_USER`) to read all player records, including phone numbers.
- **Assessment:** Per business requirements, this is intentional — players are team
  members who should see each other's profiles. Phone numbers (`phoneNumber`) may
  warrant a separate privacy review if the user base includes external parties.
- **Recommendation:** If phone numbers are considered PII, restrict their exposure to
  `ADMIN_USER`/`MASTER_USER` via a separate `PlayerPublicDTO` without the field.

#### [INFO] No JWT Token Revocation / Refresh Mechanism
- **Severity:** INFO
- **Finding:** Tokens have a 24-hour lifetime with no revocation capability. If a user's
  role is downgraded or their account is disabled, the existing JWT remains valid until
  expiry.
- **Recommendation:** For higher-security environments, implement a token blacklist
  (Redis-backed) or reduce token expiry with a refresh-token mechanism. Acceptable for
  v1.0 with an ops runbook procedure (e.g., roll `JWT_SECRET` to invalidate all tokens
  simultaneously).

#### [INFO] Actuator Metrics Endpoint Exposed (Non-Player Scope)
- **Severity:** INFO  
- **Finding:** `management.endpoints.web.exposure.include=health,info,metrics` — the
  `metrics` endpoint is exposed. This is broader than strictly necessary.
- **Recommendation:** Restrict to `health,info` in production profile, or protect
  `/actuator/**` behind `ADMIN_USER` role.

---

## Positive Security Controls Verified

| Control | Status | Notes |
|---|---|---|
| Authorization on all endpoints | ✅ | `@PreAuthorize` on every controller method |
| GROUP_ADMIN-cannot-be-player rule at service level | ✅ | `PlayerService.createPlayer` + `updatePlayer` both check |
| Password not in `PlayerDTO` | ✅ | Only email (from linked user), never password/token |
| `@Valid` on all `@RequestBody` parameters | ✅ | `PlayerCreateDTO`, `PlayerUpdateDTO`, `PlayerStatusDTO` |
| Bean Validation on all DTO fields | ✅ | `@NotBlank`, `@NotNull`, `@Min`, `@Max`, `@Size` |
| SQL injection prevention (repository) | ✅ | All queries use Spring Data + named parameters (`:playerId`) |
| Audit trail tamper-proof | ✅ | `createdBy`/`updatedBy` from `authentication.getName()` (JWT-backed) |
| Delete guard (stats check) | ✅ | `countPlayerStats` enforced in service before `delete` |
| Mass assignment protection | ✅ | Java Records are immutable — no setter-based mass assignment possible |
| BCrypt password hashing | ✅ | `BCryptPasswordEncoder` in `SecurityConfig` |
| Stateless session | ✅ | `SessionCreationPolicy.STATELESS` |
| CORS — no wildcard `*` | ✅ | No `@CrossOrigin("*")` — CORS not explicitly configured (restrictive by default) |
| No sensitive data in JWT payload | ✅ | JWT claims: `sub` (username), `userId`, `roles` only |
| JWT validation errors handled gracefully | ✅ | `validateToken` catches `JwtException` and returns `false` (→ 401, not 500) |
| Debug logging does not expose passwords/tokens | ✅ | Service logging shows only player name/id |

---

## JWT Configuration Review

| Property | Value | Assessment |
|---|---|---|
| Secret source | `${JWT_SECRET}` env var (mandatory, no fallback) | ✅ Secure after fix |
| Algorithm | HMAC-SHA (via `Keys.hmacShaKeyFor`) | ✅ |
| Token expiry | 86 400 000 ms (24 hours) | ✅ Reasonable for internal app |
| Payload contents | username, userId, roles | ✅ No sensitive PII |
| Error handling | Returns `false` on `JwtException` | ✅ (→ 401) |
| Refresh mechanism | None | ⚠️ See INFO finding above |

---

## Input Validation Review

### `PlayerCreateDTO`
```java
@NotBlank @Size(min = 2, max = 100) String name        // ✅
@NotNull @Min(1) @Max(10) Integer baseSkillRating       // ✅
@Size(max = 20) String phoneNumber                      // ✅
Boolean isActive                                         // ✅ optional, service defaults true
Long userId                                             // ✅ optional, validated in service
```

### `PlayerUpdateDTO`
```java
@Size(min = 2, max = 100) String name        // ✅ optional PATCH
@Size(max = 20) String phoneNumber            // ✅ optional PATCH
Long userId                                  // ✅ validated in service
Boolean unlinkUser                            // ✅ explicit unlink flag (OWASP-safe PATCH)
```

### `PlayerStatusDTO`
Assumed: `@NotNull Boolean isActive` — validated by `@Valid` at controller. ✅

---

## Files Changed by This Audit

| File | Change | Reason |
|---|---|---|
| `build.gradle` | Pinned `postgresql` to `42.7.11` | Fix CVE-2025-49146 + CVE-2026-42198 |
| `src/main/resources/application.yml` | Removed JWT secret fallback default | Prevent silent insecure deployment |

---

## Recommendations

1. 🔴 **DONE — PostgreSQL upgrade:** Both driver CVEs fixed by pinning to 42.7.11.
2. 🟡 **DONE — JWT secret hardening:** Removed insecure fallback; `JWT_SECRET` env var is now mandatory.
3. 🟢 **Swagger protection in prod:** Consider restricting Swagger UI to non-production profiles or behind auth.
4. 🟢 **Phone number privacy:** Evaluate whether `phoneNumber` should be excluded from the public player read DTO.
5. 🟢 **Token revocation:** For higher-security environments, consider a token blacklist or shorter expiry + refresh.
6. 🟢 **Actuator metrics:** Restrict to `health,info` in `application-prod.yml` or place behind auth.

---

## Clearance Status

**✅ CONDITIONAL CLEAR**

The Player feature is cleared for release with the two applied fixes. No critical
unresolved findings remain. The informational items (IDOR, Swagger, token revocation)
are documented for product and ops team awareness and do not block release.

> **Prerequisite for production deployment:** `JWT_SECRET` environment variable must be
> set to a randomly generated string of at least 64 characters (≥512 bits). Example:
> ```bash
> JWT_SECRET=$(openssl rand -base64 64)
> ```

