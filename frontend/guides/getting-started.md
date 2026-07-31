# Getting Started

## Prerequisites

- Node.js v20+
- npm

## Setup

```bash
git clone <repo>
cd football
npm install
cp .env.example .env.local   # fill in NEXT_PUBLIC_API_URL
npm run dev
```

## Key Commands

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start dev server |
| `npm run build` | Production build |
| `npm run type-check` | TypeScript check |
| `npm run lint` | ESLint |
| `npm run format` | Prettier |
| `npm run test` | Vitest (watch) |
| `npm run test:run` | Vitest (single run) |
| `npm run test:coverage` | Coverage report |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | Yes | Base URL of the backend API |

## Further Reading

- [Component Conventions](component-conventions.md)
- [Testing Guide](testing.md)
- [i18n Guide](i18n.md)
- [Styling Guide](styling.md)
