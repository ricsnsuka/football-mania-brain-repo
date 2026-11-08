# Dependency advisories — 2026-11-03

**Status:** frontend fixed and merged into `next`
([FootMania-Simple-Front #290](https://github.com/ricsnsuka/FootMania-Simple-Front/pull/290)); backend
**set aside until 2027-02-01**, with **Spring Boot 4 due by then**
([FootMania-Back #379](https://github.com/ricsnsuka/FootMania-Back/pull/379)).

**Update 2026-11-08: the backend side is fixed on `next`.**
[FootMania-Back#412](https://github.com/ricsnsuka/FootMania-Back/pull/412) moved to Spring Boot 4.1.2 with
Spring Framework 7.0.10, past the 7.0.9 that fixes both advisories. `config/cve-allowlist.json` is empty,
and `dependencyCveScan` reports nothing. The JSON contract was pinned on Boot 3 first
([#411](https://github.com/ricsnsuka/FootMania-Back/pull/411)) and is unchanged. **It reaches production
with 4.0.0**, the next release, well inside the deadline.

**Follows** [DEPENDENCY-CVE-SCAN 2026-08-24](DEPENDENCY-CVE-SCAN-2026-08-24.md), whose CI checks did
their job here: they went red. Nobody acted on red for about a month, and that is the finding.

---

## How it surfaced

Both repos' dependency gates had failed **every pull request** since at least 2026-10-28; the
frontend's since 2026-10-08 on two advisories, and on eleven by 2026-10-28. The failures were read as
GitHub Actions limits. That was true for one *step* on the backend, the artifact upload, which hit
the storage quota *after* the scan had already failed. On the frontend it wasn't true at all. Both
came to light on 2026-11-03, when the iPhone app's web PR (#289) went red on the same gate.

**Lesson:** a red gate's log is read to its first failing step before it is explained. "Actions
limits" was a reasonable guess that cost a month.

---

## Frontend: fixed

Next.js 16.3.8 → **16.4.4** closes two **critical** advisories: GHSA-2464-fhr4-w9xw, remote code
execution in the image optimizer with SVGs, and GHSA-673q-mjm6-r5xr, remote code execution in `use
cache` functions. It also closes one high, GHSA-p3m5-59mp-jv5r. The site uses neither `next/image`
nor `use cache`, so neither critical one is known to have been reachable. `sharp`, `zod`, `undici`,
`vite` and `source-map-js` moved to their fixed versions too.

- npm 10.9.4 crashes on this tree for `npm audit fix` and `npm update` (`edgesOut` of null); npm 11
  did the update.
- The lockfile is in step with `package.json` again, so `npm ci` works.
- The audit allowlist is empty.
- The visual suite gave 84 passing and 10 failing, the same 10 that fail on a clean `next`, so the
  upgrade changes nothing visible. Those 10 are dashboard and plan-modal captures, probably
  date-dependent, and **deserve their own look**.

## Backend: set aside, with a deadline

Two **critical** advisories, against the `spring-webmvc` 6.2.19 that Spring Boot 3.5.16 brings:

| Advisory | Needs | Here |
|---|---|---|
| GHSA-pc63-qcmh-9cmg, CVE-2026-47884 (path traversal in `XsltView`, SSRF or RCE) | an `XsltView`, a `"/**"` mapping that renders a view, and no explicit view name | **No**: the API renders no views, only JSON |
| GHSA-j9f9-w8pj-32f8, CVE-2026-47890 (corrupted SSE streams) | view fragments sent over Server-Sent Events, with data an attacker controls | **No**: chat and the draft send JSON over `SseEmitter`, never view fragments |

**No open-source fix exists on Spring 6.2.** 6.2.19 is the newest on Maven Central, and 3.5.16 the
newest Boot 3.5. Spring lists 6.2.20 as Enterprise Support only. The fix is **Spring Framework
7.0.9, which needs Spring Boot 4**.

**Decision, the owner's, 2026-11-03.** Both advisories are set aside in the backend scan until
**2027-02-01**, through `config/cve-allowlist.json`, which #379 adds. Each entry names the advisory,
the package, the date and the evidence. An exception holds through its date and **fails the scan the
day after**, so that date is the Boot 4 deadline, enforced by CI rather than by memory. The rejected
alternative was migrating to Boot 4 at once: days of high-regression work, with every backend
release blocked meanwhile, to fix a condition this code doesn't have.

**The risk accepted:** Spring's page doesn't say outright that plain JSON SSE is unaffected. The
reading that it is hasn't been proven. The deadline bounds that risk.

## Still open

- ~~**Spring Boot 4 migration, due before 2027-02-01**~~
  ([FootMania-Back#382](https://github.com/ricsnsuka/FootMania-Back/issues/382)). Done on `next` on
  2026-11-08 by #411 and #412, allowlist emptied. It ships in 4.0.0.
- **Releases.** The frontend fix reaches users only with a release to `main` (3.15.4, frontend-only).
  The backend's exception unblocks #378 (group timezones, `V65`), which ships in a paired release,
  backend first.
- **The backend's artifact storage quota**, which also failed the 2026-10-28 run's upload steps.
