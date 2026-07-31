# Heroku Deployment Guide — Football Management System

**Version:** 1.0.0  
**Stack:** Java 21 · Spring Boot 3.4.5 · PostgreSQL · Flyway · Caffeine  
**Last Updated:** 2026-06-03

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Environment Variables Reference](#2-environment-variables-reference)
3. [Initial Heroku Setup](#3-initial-heroku-setup)
4. [JVM Configuration](#4-jvm-configuration)
5. [Database Setup](#5-database-setup)
6. [Build & Deploy](#6-build--deploy)
7. [Health Checks & Verification](#7-health-checks--verification)
8. [Keycloak Setup (Optional Profile)](#8-keycloak-setup-optional-profile)
9. [Logging & Monitoring](#9-logging--monitoring)
10. [Scaling](#10-scaling)
11. [Troubleshooting](#11-troubleshooting)
12. [Security Checklist](#12-security-checklist)
13. [Rollback Procedure](#13-rollback-procedure)

---

## 1. Prerequisites

### Local Tools Required

| Tool | Version | Install |
|------|---------|---------|
| Heroku CLI | Latest | https://devcenter.heroku.com/articles/heroku-cli |
| Java JDK | 21 (Temurin) | https://adoptium.net |
| Git | 2.x+ | https://git-scm.com |
| Gradle | Wrapper included (`./gradlew`) | — |
| PostgreSQL client (`psql`) | 14+ | For migration verification |

### Verify Heroku CLI Installation

```bash
heroku --version          # heroku/x.x.x
heroku login              # Opens browser for auth
heroku whoami             # Confirms logged-in user
```

### Verify Java Version

```bash
java -version             # Must output: openjdk version "21"
./gradlew -version        # Should report Gradle 8+
```

---

## 2. Environment Variables Reference

### Mandatory Variables (app fails to start if missing)

| Variable | Description | Example / How to Set |
|---------|-------------|----------------------|
| `JWT_SECRET` | HMAC signing secret. **Must be ≥ 64 chars (512 bits).** App performs a fail-fast check on startup. | `openssl rand -base64 64` |
| `DATABASE_URL` | PostgreSQL JDBC URL. Automatically set by the Heroku Postgres add-on. | `jdbc:postgresql://host:5432/db?sslmode=require` |
| `DATABASE_USERNAME` | DB username. Parsed from `DATABASE_URL` by the add-on config. | Set via `heroku config:set` |
| `DATABASE_PASSWORD` | DB password. | Set via `heroku config:set` |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed frontend origins. | `https://myfootballapp.com` |

### Optional Variables (have safe defaults)

| Variable | Default | Description |
|---------|---------|-------------|
| `SPRING_PROFILES_ACTIVE` | `prod` | Active Spring profile. Do **not** change unless you know what you're doing. |
| `JWT_EXPIRATION` | `86400000` | JWT token TTL in milliseconds (24 h). |
| `JAVA_OPTS` | *(see Procfile)* | Extra JVM flags appended to the Procfile baseline. |
| `PORT` | Set by Heroku | HTTP port to bind. **Never override manually.** |
| `LOGGING_LEVEL_ROOT` | `WARN` | Root log level. Use `INFO` for initial diagnosis, `WARN` for steady-state. |

### Generating a Secure JWT_SECRET

```bash
# Option 1 — openssl (recommended)
openssl rand -base64 64

# Option 2 — /dev/urandom
head -c 64 /dev/urandom | base64

# Option 3 — Node.js (if available)
node -e "console.log(require('crypto').randomBytes(64).toString('base64'))"
```

> ⚠️ **Never reuse a development secret in production.** The app explicitly has no default for
> `JWT_SECRET`; it will throw `IllegalStateException` and refuse to start if the variable is absent.

---

## 3. Initial Heroku Setup

### 3.1 Create the App

```bash
# Create a new Heroku app (region eu or us)
heroku create football-app-prod --region eu

# Or link an existing app
heroku git:remote -a football-app-prod
```

### 3.2 Set the Java Runtime

The `system.properties` file in the repository root already pins the JDK:

```properties
# system.properties
java.runtime.version=21
```

Heroku reads this file automatically during slug compilation. No manual step is needed.

### 3.3 Add Heroku Postgres

```bash
# Standard plan (recommended for production)
heroku addons:create heroku-postgresql:essential-1 -a football-app-prod

# Verify it was provisioned
heroku addons -a football-app-prod
heroku config -a football-app-prod | grep DATABASE_URL
```

> **Note:** Heroku provides `DATABASE_URL` in Postgres URL format (`postgres://user:pass@host:5432/db`).
> The app expects a JDBC URL. The Spring Boot prod profile reads `DATABASE_URL` directly — you must
> convert it. See [Section 5.1](#51-converting-the-heroku-database_url) below.

### 3.4 Set All Required Config Vars

```bash
# JWT — generate with: openssl rand -base64 64
heroku config:set JWT_SECRET="<your-64-char-base64-secret>" -a football-app-prod

# Spring profile
heroku config:set SPRING_PROFILES_ACTIVE=prod -a football-app-prod

# CORS — your production frontend URL
heroku config:set CORS_ALLOWED_ORIGINS="https://myfootballapp.com" -a football-app-prod

# Database credentials (parsed from DATABASE_URL or set explicitly)
heroku config:set DATABASE_USERNAME="<db-user>" -a football-app-prod
heroku config:set DATABASE_PASSWORD="<db-password>" -a football-app-prod
heroku config:set DATABASE_URL="jdbc:postgresql://<host>:<port>/<db>?sslmode=require" -a football-app-prod

# Optional: JWT expiration override (default 24 h)
heroku config:set JWT_EXPIRATION=86400000 -a football-app-prod

# Verify all vars
heroku config -a football-app-prod
```

### 3.5 Confirm the Procfile

The `Procfile` in the repository root is already production-ready:

```
web: java -Dserver.port=$PORT \
         -Dspring.profiles.active=${SPRING_PROFILES_ACTIVE:-prod} \
         -Xms256m -Xmx512m \
         -XX:MaxMetaspaceSize=128m \
         -XX:+UseZGC \
         -XX:+ZGenerational \
         -XX:+UseStringDeduplication \
         -XX:+UseContainerSupport \
         -XX:+ExitOnOutOfMemoryError \
         $JAVA_OPTS \
         -jar build/libs/football-1.0.0.jar
```

> **Version check:** Ensure `football-1.0.0.jar` matches the `version` in `build.gradle`
> (`version = '1.0.0'`). If you bump the version, update the `Procfile` accordingly before deploying.

---

## 4. JVM Configuration

### 4.1 Flags Explained

| Flag | Purpose |
|------|---------|
| `-XX:+UseZGC` | Z Garbage Collector — sub-5ms GC pauses, ideal for latency-sensitive REST APIs |
| `-XX:+ZGenerational` | Generational ZGC (Java 21+) — improves throughput on short-lived objects |
| `-XX:+UseStringDeduplication` | Reduces heap pressure from repeated strings (player names, team labels) |
| `-XX:+UseContainerSupport` | Reads cgroup memory/CPU limits correctly inside Heroku dynos |
| `-XX:+ExitOnOutOfMemoryError` | Forces a clean dyno restart instead of degraded OOM state |
| `-Xms256m -Xmx512m` | Tuned for Heroku Standard-1X (512 MB RAM). Adjust for larger dynos. |
| `-XX:MaxMetaspaceSize=128m` | Caps metaspace to avoid runaway class-loading overhead |

### 4.2 Virtual Threads

Virtual threads are enabled globally via `application.yml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

This replaces the default 200-thread platform thread pool with OS-level virtual threads, allowing
thousands of concurrent requests with minimal memory overhead — critical for Heroku's memory
constraints.

### 4.3 Memory Sizing per Dyno Type

| Dyno Type | RAM | Recommended `-Xmx` | Notes |
|-----------|-----|---------------------|-------|
| `basic` | 512 MB | `-Xmx256m` | Development / low-traffic only |
| `standard-1x` | 512 MB | `-Xmx384m` | Suitable for moderate traffic |
| `standard-2x` | 1 GB | `-Xmx768m` | Recommended for production |
| `performance-m` | 2.5 GB | `-Xmx2g` | High-traffic production |

To override JVM memory on a running dyno without changing the `Procfile`:

```bash
heroku config:set JAVA_OPTS="-Xms512m -Xmx768m" -a football-app-prod
```

---

## 5. Database Setup

### 5.1 Converting the Heroku DATABASE_URL

Heroku provides `DATABASE_URL` in the native Postgres format (`postgres://user:pass@host:port/db`).
Spring Boot's JDBC driver requires the `jdbc:postgresql://` scheme. Convert it once and store as the
app's `DATABASE_URL` config var:

```bash
# Retrieve the raw Heroku URL
RAW_URL=$(heroku config:get DATABASE_URL -a football-app-prod)
# It looks like: postgres://user:password@ec2-xx-xx.amazonaws.com:5432/dbname

# Extract parts and reformat
JDBC_URL=$(echo "$RAW_URL" | sed 's|postgres://\([^:]*\):\([^@]*\)@\(.*\)|\njdbc:postgresql://\3?sslmode=require|' | grep jdbc)
DB_USER=$(echo "$RAW_URL" | sed 's|postgres://\([^:]*\):.*|\1|')
DB_PASS=$(echo "$RAW_URL" | sed 's|postgres://[^:]*:\([^@]*\)@.*|\1|')

# Set as Heroku config vars
heroku config:set DATABASE_URL="$JDBC_URL" -a football-app-prod
heroku config:set DATABASE_USERNAME="$DB_USER" -a football-app-prod
heroku config:set DATABASE_PASSWORD="$DB_PASS" -a football-app-prod
```

Or use the one-liner helper script (save as `scripts/heroku-set-db.sh`):

```bash
#!/usr/bin/env bash
# Usage: ./scripts/heroku-set-db.sh football-app-prod
APP=$1
RAW=$(heroku config:get DATABASE_URL -a "$APP")
USER=$(echo "$RAW" | awk -F'[://@]' '{print $4}')
PASS=$(echo "$RAW" | awk -F'[://@]' '{print $5}')
HOST=$(echo "$RAW" | awk -F'[://@]' '{print $6}')
PORT=$(echo "$RAW" | awk -F'[://@]' '{print $7}')
DB=$(echo "$RAW" | awk -F'/' '{print $NF}')
heroku config:set \
  DATABASE_URL="jdbc:postgresql://${HOST}:${PORT}/${DB}?sslmode=require" \
  DATABASE_USERNAME="$USER" \
  DATABASE_PASSWORD="$PASS" \
  -a "$APP"
echo "✅ Database config vars updated for $APP"
```

### 5.2 Flyway Migrations

Flyway runs **automatically** on application startup when `spring.flyway.enabled=true` (the default
in `application-prod.yml`). No manual step is required — migrations are applied before the app
starts accepting traffic.

Current migration files:

| File | Description |
|------|-------------|
| `V1__initial_schema.sql` | Base schema — all core tables |
| `V2__player_stats_goal_types.sql` | Player statistics and goal type enums |
| `V3__player_aggregate_stats.sql` | Aggregate player statistics view/table |
| `V4__draft_sessions.sql` | Draft session management tables |

#### Verify Migrations Ran Successfully

```bash
heroku logs --tail -a football-app-prod | grep -i "flyway\|migration"
# Expected output:
# Flyway: Successfully applied 4 migrations to schema "public"
```

#### Key Flyway Settings (production)

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false    # Do NOT change — prevents accidental baseline on clean DBs
    out-of-order: false           # Enforce strict versioning
```

> ⚠️ **Never set `baseline-on-migrate: true` in production** unless you are migrating an existing
> schema that pre-dates Flyway adoption. Doing so will silently skip all prior migrations.

### 5.3 HikariCP Connection Pool

Production pool settings (from `application.yml`, overridable per dyno via config vars):

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # Heroku Postgres essential-1 allows 25 connections
      minimum-idle: 5
      connection-timeout: 20000    # 20 s — fail fast instead of queueing forever
      idle-timeout: 300000         # 5 min
      max-lifetime: 1200000        # 20 min (< Heroku's 20-min idle timeout)
      pool-name: FootballHikariPool
```

> **Important:** Heroku Postgres `essential-1` allows a maximum of **25 connections**. If you scale
> to multiple dynos, multiply `maximum-pool-size` by the number of dynos — total must stay ≤ 25.
> Upgrade to `standard-0` (120 connections) or higher for multi-dyno deployments.

| Plan | Max Connections | Recommended `maximum-pool-size` per dyno |
|------|----------------|------------------------------------------|
| `essential-0` | 20 | 8 (2 dynos max) |
| `essential-1` | 25 | 10 (2 dynos) / 5 (4 dynos) |
| `standard-0` | 120 | 20 (up to 5 dynos) |
| `standard-2` | 400 | 20 (up to 19 dynos) |

### 5.4 SSL Configuration

The `sslmode=require` parameter in the JDBC URL enforces TLS on all database connections.
Heroku Postgres requires SSL — connections without it are rejected.

---

## 6. Build & Deploy

### 6.1 Build the JAR Locally

```bash
# Full build (includes tests + SpotBugs)
./gradlew clean build --no-daemon

# Production-only build (no tests — use only if tests already passed in CI)
./gradlew buildProd --no-daemon
# Equivalent to: ./gradlew clean bootJar --no-daemon

# Verify the JAR was created
ls -lh build/libs/football-1.0.0.jar
```

### 6.2 Deploy to Heroku

Heroku builds the slug server-side by detecting the Gradle project via the Java buildpack.
The `Procfile` instructs the dyno to run the pre-built JAR.

```bash
# Ensure you are on the correct branch
git status
git log --oneline -5

# Add the Heroku remote (only needed once)
heroku git:remote -a football-app-prod

# Push to deploy
git push heroku main

# Monitor the build log
heroku logs --tail -a football-app-prod
```

### 6.3 Heroku Build Process

When you push, Heroku:

1. Detects Java/Gradle (from `build.gradle` + `gradlew`)
2. Runs `./gradlew build -x test` (test exclusion is Heroku default)
3. Packages the result as a **slug**
4. Releases the slug to all dynos
5. Dynos restart and execute the `Procfile` `web` command

### 6.4 Slug Size Considerations

Heroku slugs must be **< 500 MB** (compressed). The typical slug size for this app is ~120–150 MB.

If you exceed the limit:

```bash
# Check current slug size
heroku releases:info -a football-app-prod

# Common causes of slug bloat:
# - Build artifacts checked in to Git (build/ directory)
# - Test resources not excluded

# Verify .gitignore excludes build artifacts:
cat .gitignore | grep build
```

The `.gitignore` should contain:
```
build/
.gradle/
*.jar
```

### 6.5 Deploy from a Specific Git Branch

```bash
# Deploy a feature branch to staging
git push heroku feature/my-feature:main

# Deploy a tag
git push heroku v1.0.0:main
```

---

## 7. Health Checks & Verification

### 7.1 Application Health Endpoint

The app exposes a custom health endpoint at `GET /api/health`.

```bash
# Check via curl
curl -s https://football-app-prod.herokuapp.com/api/health | jq .

# Expected response (HTTP 200):
# { "status": "UP" }
```

### 7.2 Spring Boot Actuator

Actuator endpoints are available at `/actuator/*` (restricted to `health`, `info`, `metrics`
in production):

```bash
# Health (no auth required)
curl https://football-app-prod.herokuapp.com/actuator/health

# Info (app version, build time)
curl https://football-app-prod.herokuapp.com/actuator/info

# Metrics (requires authentication in prod)
curl -H "Authorization: Bearer <token>" \
     https://football-app-prod.herokuapp.com/actuator/metrics
```

### 7.3 Post-Deployment Verification Checklist

```bash
APP=https://football-app-prod.herokuapp.com

# 1. App is alive
curl -f "$APP/api/health"

# 2. Flyway migrations applied
heroku logs -n 200 -a football-app-prod | grep -i flyway

# 3. Authentication works — obtain a token
curl -X POST "$APP/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<password>"}'

# 4. Key API endpoints respond
TOKEN="<jwt-from-above>"
curl -H "Authorization: Bearer $TOKEN" "$APP/api/players"
curl -H "Authorization: Bearer $TOKEN" "$APP/api/seasons/current"

# 5. No ERROR-level logs in the first 5 minutes
heroku logs -n 500 -a football-app-prod | grep -c "ERROR"
# Should be 0
```

---

## 8. Keycloak Setup (Optional Profile)

Keycloak is an **optional** authentication provider activated by adding `keycloak` to the active
Spring profiles. The default authentication method is HMAC JWT (`prod` profile only).

> **Skip this section** unless you specifically need Keycloak for SSO / enterprise identity
> management.

### 8.1 When to Use Keycloak

- Enterprise SSO requirements
- Multi-tenant deployments
- OAuth2/OIDC integration with third-party clients

### 8.2 Deploying Keycloak to Heroku

Keycloak requires its own Heroku app (separate dyno). A production-optimised `Dockerfile.prod` is
provided in `keycloak/`:

```bash
# Create a separate Heroku app for Keycloak
heroku create football-keycloak-prod --region eu

# Set the stack to container
heroku stack:set container -a football-keycloak-prod

# Set Keycloak config vars
heroku config:set \
  KC_DB_URL="jdbc:postgresql://<host>:<port>/<db>?sslmode=require" \
  KC_DB_USERNAME="<user>" \
  KC_DB_PASSWORD="<password>" \
  KC_BOOTSTRAP_ADMIN_USERNAME="admin" \
  KC_BOOTSTRAP_ADMIN_PASSWORD="<secure-admin-password>" \
  KC_HOSTNAME="https://football-keycloak-prod.herokuapp.com" \
  KC_PROXY_HEADERS="xforwarded" \
  KC_HTTP_ENABLED="true" \
  KC_HOSTNAME_STRICT="false" \
  -a football-keycloak-prod
```

Heroku terminates TLS at the router, so Keycloak runs in HTTP mode internally (`KC_HTTP_ENABLED=true`,
`KC_PROXY_HEADERS=xforwarded`). The `entrypoint.sh` in `keycloak/` handles the `$PORT` binding.

### 8.3 Activate the Keycloak Profile in the App

```bash
heroku config:set \
  SPRING_PROFILES_ACTIVE="prod,keycloak" \
  KEYCLOAK_ISSUER_URI="https://football-keycloak-prod.herokuapp.com/realms/football" \
  -a football-app-prod
```

### 8.4 Realm Import

The realm configuration is pre-packaged in `keycloak/import/football-realm-prod.json` and is
imported automatically during Keycloak startup (handled in the Dockerfile build stage).

---

## 9. Logging & Monitoring

### 9.1 Real-Time Log Tailing

```bash
# Tail all logs
heroku logs --tail -a football-app-prod

# Filter for specific patterns
heroku logs --tail -a football-app-prod | grep -E "ERROR|WARN|Flyway"

# Last N lines
heroku logs -n 500 -a football-app-prod
```

### 9.2 Production Log Levels

Configured in `application-prod.yml`:

```yaml
logging:
  level:
    pt.rics.demo.football: INFO   # Application code
    root: WARN                    # Framework/library noise suppressed
```

To temporarily increase verbosity for diagnosis (without redeploying):

```bash
heroku config:set LOGGING_LEVEL_ROOT=INFO -a football-app-prod
heroku config:set LOGGING_LEVEL_PT_RICS_DEMO_FOOTBALL=DEBUG -a football-app-prod
# Remember to revert after diagnosis:
heroku config:unset LOGGING_LEVEL_PT_RICS_DEMO_FOOTBALL -a football-app-prod
```

### 9.3 Log Drains (Persistent Logging)

Heroku retains only the last **1,500 lines** of logs. For production you should add a log drain:

```bash
# Papertrail (free tier available)
heroku addons:create papertrail:choklad -a football-app-prod

# Logtail / Better Stack
heroku drains:add https://in.logtail.com/YOUR_SOURCE_TOKEN -a football-app-prod

# List active drains
heroku drains -a football-app-prod
```

### 9.4 Dyno Metrics

```bash
# View dyno memory usage
heroku ps -a football-app-prod

# Enable extended metrics dashboard (requires paid dyno)
heroku labs:enable log-runtime-metrics -a football-app-prod
```

---

## 10. Scaling

### 10.1 Scaling Dynos

```bash
# Scale to 2 web dynos
heroku ps:scale web=2 -a football-app-prod

# View current scale
heroku ps -a football-app-prod

# Scale down
heroku ps:scale web=1 -a football-app-prod
```

### 10.2 Dyno Type Recommendations

| Traffic Level | Dyno Type | Count | Notes |
|--------------|-----------|-------|-------|
| Development / Staging | `basic` | 1 | Sleeps after 30 min inactivity |
| Low production (< 100 req/s) | `standard-1x` | 1–2 | Always-on, 512 MB RAM |
| Moderate production (100–500 req/s) | `standard-2x` | 2–4 | 1 GB RAM, ZGC benefits shine |
| High production (500+ req/s) | `performance-m` | 2+ | 2.5 GB RAM, dedicated compute |

> ⚠️ **Never use `basic` or `eco` dynos for production.** They sleep after 30 minutes of inactivity,
> causing a cold-start delay (30–60 s) for the first request.

### 10.3 Multi-Dyno Database Connection Budget

When running multiple dynos, ensure the total connection pool does not exceed your Heroku Postgres
plan limit:

```
total_connections = dynos × maximum-pool-size
```

Example with `standard-1x` × 3 dynos and `essential-1` (25 connections):

```
3 × 8 = 24  ✅  (leaves 1 for admin tools)
3 × 10 = 30  ❌  (exceeds limit — connections will be refused)
```

Adjust via:

```bash
heroku config:set SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=8 -a football-app-prod
```

### 10.4 Cache Behaviour at Scale

This app uses **Caffeine** (local, in-process cache). Each dyno maintains its own independent cache.
This means:

- ✅ No external cache infrastructure required (no Redis add-on needed)
- ⚠️ Cache is **not shared** between dynos — a cache miss on dyno 2 won't hit dyno 1's cache
- ⚠️ Cache entries on one dyno will not reflect updates made via a request routed to another dyno

For most use cases this is acceptable. If strong cache consistency across dynos is required,
migrate to Redis (Heroku Data for Redis add-on) and update the cache configuration.

---

## 11. Troubleshooting

### 11.1 App Fails to Start — `JWT_SECRET` Not Set

**Symptom:** Dyno crashes immediately; logs show:

```
java.lang.IllegalStateException: JWT_SECRET environment variable must be set
```

**Fix:**

```bash
heroku config:set JWT_SECRET="$(openssl rand -base64 64)" -a football-app-prod
heroku ps:restart -a football-app-prod
```

### 11.2 App Fails to Start — Database Connection Refused

**Symptom:**

```
HikariPool-1 - Exception during pool initialization
com.zaxxer.hikari.pool.HikariPool$PoolInitializationException
```

**Diagnosis:**

```bash
# Verify DATABASE_URL is a JDBC URL (must start with jdbc:postgresql://)
heroku config:get DATABASE_URL -a football-app-prod

# If it starts with postgres:// (not jdbc:postgresql://) re-run the conversion:
# See Section 5.1
```

**Fix:** Re-run the URL conversion script from [Section 5.1](#51-converting-the-heroku-database_url).

### 11.3 Flyway Migration Failure

**Symptom:**

```
FlywayException: Validate failed: Migration checksum mismatch for migration version 2
```

**Cause:** A previously-applied migration file was modified after it ran.

**Fix:** Never modify an already-applied migration. Create a new version instead:

```sql
-- BAD: modifying V2__player_stats_goal_types.sql
-- GOOD: creating V5__fix_goal_type_column.sql
```

If you are absolutely sure the change is intentional (dev environment only):

```bash
# Connect to Heroku Postgres and repair the checksum
heroku pg:psql -a football-app-prod -c \
  "UPDATE flyway_schema_history SET checksum = <new_checksum> WHERE version = '2';"
```

### 11.4 R14 — Memory Quota Exceeded

**Symptom:** Heroku dashboard or logs show `Error R14 (Memory quota exceeded)`.

**Diagnosis:**

```bash
heroku logs --tail -a football-app-prod | grep "Memory"
```

**Fix options:**

1. Upgrade dyno type (`standard-2x` has 1 GB RAM):
   ```bash
   heroku ps:type standard-2x -a football-app-prod
   ```

2. Reduce heap size to leave room for Metaspace + JVM overhead:
   ```bash
   # For standard-1x (512 MB): keep -Xmx ≤ 384m
   heroku config:set JAVA_OPTS="-Xms128m -Xmx384m" -a football-app-prod
   ```

3. ZGC is the right GC for memory-constrained environments. Ensure `-XX:+UseZGC` is active.

### 11.5 H10 — App Crashed (Boot Timeout)

**Symptom:** Heroku returns HTTP 503; logs show `Error H10 (App crashed)`.

Heroku expects the app to bind to `$PORT` within **90 seconds**.

**Diagnosis:**

```bash
heroku logs -n 200 -a football-app-prod | grep -E "H10|ERROR|Exception"
```

**Common causes:**

| Cause | Fix |
|-------|-----|
| Missing env var causes fail-fast | Set the missing variable (see [Section 2](#2-environment-variables-reference)) |
| Flyway migration takes > 90 s | Break large migrations into smaller ones; pre-run via `heroku pg:psql` |
| DB not reachable | Check `DATABASE_URL` format and SSL settings |
| Wrong JAR name in `Procfile` | Verify `build.gradle` `version` matches `Procfile` |

### 11.6 CORS Errors from the Frontend

**Symptom:** Browser console shows `CORS policy: No 'Access-Control-Allow-Origin' header`.

**Fix:**

```bash
heroku config:set CORS_ALLOWED_ORIGINS="https://myfootballapp.com" -a football-app-prod
```

Multiple origins (comma-separated):

```bash
heroku config:set CORS_ALLOWED_ORIGINS="https://myfootballapp.com,https://www.myfootballapp.com" \
  -a football-app-prod
```

### 11.7 Slow First Request After Deployment

**Cause:** JVM JIT warm-up + Caffeine cache cold start.

**Expected behaviour:** First 10–20 requests are slower (50–200 ms) while the JIT compiles hot paths.
This is normal and resolves automatically.

**Mitigation for zero-downtime:** Use Heroku's preboot feature:

```bash
heroku features:enable preboot -a football-app-prod
```

Preboot starts new dynos before terminating the old ones, ensuring no request is served by a
cold JVM.

---

## 12. Security Checklist

Before going live, verify each item:

### Secrets & Credentials

- [ ] `JWT_SECRET` is ≥ 64 characters (512 bits), randomly generated, unique to production
- [ ] No secrets committed to Git (`.env` is in `.gitignore`)
- [ ] Database password is strong and not reused across environments
- [ ] Heroku Config Vars are set, not hardcoded in `application-prod.yml`

### Spring Security

- [ ] `spring.jpa.hibernate.ddl-auto=validate` — **never** `create`, `create-drop`, or `update`
- [ ] `spring.jpa.show-sql=false` — SQL must not leak to production logs
- [ ] `management.endpoint.health.show-details=never` — health endpoint does not reveal internals
- [ ] Swagger UI is disabled or access-restricted in production (review `springdoc` config)
- [ ] CORS origins are explicitly set (no wildcard `*` in production)

### Network & Transport

- [ ] Heroku Postgres SSL is enforced (`sslmode=require` in JDBC URL)
- [ ] App is served over HTTPS only (Heroku routes enforce HTTPS by default)
- [ ] HTTP → HTTPS redirect enabled in Heroku settings

### Dependencies

- [ ] PostgreSQL driver pinned to `42.7.11+` (fixes CVE-2025-49146 and CVE-2026-42198)
- [ ] No CRITICAL CVEs in `./gradlew dependencyCheckAnalyze` output
- [ ] SpotBugs passes: `./gradlew spotbugsMain`

### Operational

- [ ] Log drain configured — logs are not ephemeral
- [ ] Database backup policy in place (Heroku Postgres auto-backups for `standard` tier)
- [ ] Rollback procedure tested (see [Section 13](#13-rollback-procedure))
- [ ] Preboot enabled to prevent cold-start traffic exposure

---

## 13. Rollback Procedure

### 13.1 Application Rollback

```bash
# List the last 10 releases
heroku releases -n 10 -a football-app-prod

# Roll back to the previous release
heroku rollback -a football-app-prod

# Roll back to a specific release number
heroku rollback v42 -a football-app-prod

# Monitor logs after rollback
heroku logs --tail -a football-app-prod
```

### 13.2 Database Rollback

> ⚠️ Flyway migrations are **forward-only** by design. There is no automatic `flyway undo` in the
> community edition.

**Option A — Restore from backup (data loss window = time since last backup):**

```bash
# List available backups
heroku pg:backups -a football-app-prod

# Restore most recent backup
heroku pg:backups:restore -a football-app-prod

# Restore a specific backup
heroku pg:backups:restore b003 DATABASE_URL -a football-app-prod
```

**Option B — Apply a compensating migration (preferred for non-destructive changes):**

Create a new migration `V5__rollback_v4.sql` that reverses the schema changes and deploy it.

**Option C — Manual SQL (emergency only):**

```bash
heroku pg:psql -a football-app-prod
# Then execute compensating DDL/DML manually
```

### 13.3 Pre-Deployment Database Backup

Always create a manual backup before deploying migrations:

```bash
heroku pg:backups:capture -a football-app-prod
heroku pg:backups -a football-app-prod   # Confirm backup completed
```

---

## Appendix A — Quick Reference Commands

```bash
# Deploy
git push heroku main

# Restart dynos
heroku ps:restart -a football-app-prod

# Live logs
heroku logs --tail -a football-app-prod

# Open app in browser
heroku open -a football-app-prod

# Run one-off command (e.g., DB check)
heroku run "java -version" -a football-app-prod

# Connect to Postgres
heroku pg:psql -a football-app-prod

# View all config vars
heroku config -a football-app-prod

# Set a config var
heroku config:set KEY=VALUE -a football-app-prod

# Remove a config var
heroku config:unset KEY -a football-app-prod

# View dyno status
heroku ps -a football-app-prod

# Scale
heroku ps:scale web=2 -a football-app-prod

# Check slug size of latest release
heroku releases:info -a football-app-prod

# List recent releases
heroku releases -n 10 -a football-app-prod

# Rollback
heroku rollback v42 -a football-app-prod
```

---

## Appendix B — Environment Variables Quick-Set Script

Save as `scripts/heroku-configure.sh` and run once per environment:

```bash
#!/usr/bin/env bash
# Usage: ./scripts/heroku-configure.sh <app-name>
set -euo pipefail

APP="${1:?Usage: $0 <heroku-app-name>}"

echo "🔧 Configuring Heroku app: $APP"

heroku config:set \
  SPRING_PROFILES_ACTIVE="prod" \
  JWT_SECRET="$(openssl rand -base64 64)" \
  JWT_EXPIRATION="86400000" \
  CORS_ALLOWED_ORIGINS="https://myfootballapp.com" \
  LOGGING_LEVEL_ROOT="WARN" \
  -a "$APP"

echo "✅ Base config set. Don't forget:"
echo "   → Set DATABASE_URL / DATABASE_USERNAME / DATABASE_PASSWORD (see Section 5.1)"
echo "   → Verify: heroku config -a $APP"
```

---

*Guide maintained by the Football Management System deployment team.*  
*For questions, open an issue or consult the architecture docs in `docs/architecture/`.*

