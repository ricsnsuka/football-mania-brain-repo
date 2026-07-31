# Component Conventions

See [Architecture Overview](../architecture/overview.md) for folder structure.

## Rules

1. **Server Components only where they cost nothing** — in practice this app is a
   client-rendered SPA behind `AuthGuard` (see [Architecture Overview](../architecture/overview.md)):
   the root layout, the `(app)` route-group layout, and `(app)/loading.tsx` are genuine
   Server Components, but every route's `page.tsx` and everything it renders is a Client
   Component, because each page needs `useAppStore`/`AuthGuard` (hooks) and most feature
   components need interactivity or TanStack Query. Add `'use client'` to any component
   using hooks, events, or browser APIs — which, today, is nearly all of them. Don't reach
   for a Server Component just to match this rule; write `'use client'` when the component
   needs it, same as before.
2. **No Tailwind in JSX** — semantic class names only; styles live in `src/app/globals.css`
3. **No hardcoded strings** — always `t('key')` via `useTranslation`
4. **Named exports only** — no default exports from component files
5. **One component per file**
6. **Types from `src/types/`** — never inline type definitions in component files

## Atomic Hierarchy

```
Page       → composes Feature components only
Feature    → orchestrates data fetching + layout
Organism   → self-contained UI section (PlayerCard, MatchRow)
Molecule   → logical grouping of atoms (SearchBar, PlayerBadge)
Atom       → single-purpose element (Button, Avatar, Badge)
```

Shared atoms/molecules → `src/components/ui/`  
Feature-specific → `src/features/<feature>/components/`

## Class Naming

BEM-inspired: `block__element--modifier`

```tsx
// ✅ correct
<div className="player-card">
  <span className="player-card__name player-card__name--highlighted">...</span>
</div>

// ❌ wrong — no Tailwind in JSX
<div className="flex items-center gap-2 rounded-lg">
```
