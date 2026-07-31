# Documentation Index

> **Rule**: Every new doc file must be linked here. The documentation-agent is responsible for keeping this file up to date.

## Structure

```
docs/
├── INDEX.md              ← you are here
├── architecture/         ← system design, tech stack rationale
├── features/             ← one file per product feature
├── guides/               ← developer how-to references
├── technical/            ← ADRs and low-level technical decisions
└── api/                  ← API domain docs (frontend perspective)
```

---

## Architecture

| Document | Description |
|----------|-------------|
| [Overview](architecture/overview.md) | Tech stack, folder structure, data flow |

---

## Features

| Document | Description |
|----------|-------------|
| [Login](features/login.md) | Login form, auth flow, change-password modal |
| [Notifications](features/notifications.md) | Toast notification widget — usage, types, layout |
| [Language Switcher](features/language-switcher.md) | Locale selection widget — en, pt, es; store persistence; adding new locales |
| [Dashboard](features/dashboard.md) | Role-based dashboards — Admin, Master, Basic; nav cards; linked player detection |
| [Players](features/players.md) | Paginated players table — page size selector, filters, rank calculation |
| [Matches](features/matches.md) | Paginated matches list — page size selector, filters, role gates |
| [PWA](features/pwa.md) | Installability, manifest, service worker and caching strategy, offline fallback, iOS install hint |
| [Push Notifications](features/push-notifications.md) | Web Push subscription flow, iOS install requirement, per-category preferences, service worker handlers |
| [Privacy & Your Data](features/privacy.md) | GDPR export and account deletion, the admin on-behalf flow, and the public privacy policy page |

---

## Developer Guides

| Document | Description |
|----------|-------------|
| [Getting Started](guides/getting-started.md) | Local setup and dev workflow |
| [Component Conventions](guides/component-conventions.md) | How to write components in this project |
| [Shared UI Primitives](guides/shared-components.md) | `Pagination`, `TableSkeleton`, `DashboardSkeleton`, `DataStateMessage` — canonical pagination/loading/empty-state components |
| [Testing](guides/testing.md) | How to write and run tests |
| [i18n](guides/i18n.md) | How to add and manage translations |
| [Styling](guides/styling.md) | CSS architecture and class naming |
| [Netlify Deployment](guides/netlify-deployment.md) | Step-by-step guide to deploying to Netlify |

---

## Technical Decisions (ADRs)

| Document | Description |
|----------|-------------|
| _(none yet)_ | Added here as decisions are recorded |

---

## API Reference

| Document | Description |
|----------|-------------|
| _(none yet)_ | Added per domain as API layer is built |
