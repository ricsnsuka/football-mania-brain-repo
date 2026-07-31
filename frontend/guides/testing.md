# Testing Guide

## Stack

- **Runner**: Vitest v4 (jsdom environment, globals enabled)
- **Rendering**: `@testing-library/react`
- **User events**: `@testing-library/user-event`
- **Matchers**: `@testing-library/jest-dom`

## File Locations

Tests live under `src/tests/`, grouped by **feature/domain area** rather than mirroring the
`src/` category folders (`hooks/`, `services/`, `features/`) 1:1 — a component test, its
hook's test, and its feature-page test for the same domain sit in the same folder:

```
src/tests/
├── setup.ts                         # jest-dom import
├── app/                              # error.tsx / loading.tsx / not-found.tsx
├── auth/                             # AuthGuard, LoginForm, useLogin, ChangePasswordModal, ...
├── components/                       # shared src/components/{ui,layout}/ pieces
│                                      # (Pagination, TableSkeleton, DashboardSkeleton,
│                                      #  DataStateMessage, Navbar, NotificationWidget, ...)
├── dashboard/                        # AdminDashboard, MasterDashboard, BasicDashboard, DashboardPage
├── draft/                            # useDraft, CaptainPickBoard/Section, DraftSessionsAdminPage
├── lib/                              # formatDate, and other src/lib/ helpers
├── matches/                          # MatchCard, MatchModal, CreateMatchModal, MatchesPage
├── matchPlans/                       # MatchPlanCard, CreateMatchPlanModal, MatchPlanDetailModal, ...
├── players/                          # PlayersPage, PlayerModal, CreatePlayerModal
└── store/                            # appStore
```

A hook test (e.g. `useDraft.test.ts`) lives with its feature/domain (`tests/draft/`), not in
a standalone `tests/hooks/` folder — there is no such folder. When adding a test, put it
next to the other tests for the same domain rather than inventing a new category folder.

## Running Tests

```bash
npm run test          # watch mode
npm run test:run      # single run
npm run test:coverage # coverage report
# run a specific file:
npm run test:run -- src/tests/features/players/PlayerCard.test.tsx
```

## Standard Mocks

```typescript
// react-i18next
// Include `i18n.language` (+ a stub `changeLanguage`) alongside `t` — components widely
// call `i18n.language` for locale-aware date formatting via `src/lib/formatDate.ts`, and a
// mock missing it will throw ("Cannot read properties of undefined") in any component that
// formats a date.
vi.mock('react-i18next', () => ({
  useTranslation: () => ({
    t: (key: string) => key,
    i18n: { language: 'en', changeLanguage: vi.fn() },
  }),
}));

// TanStack Query wrapper
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
const createWrapper = () => {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={qc}>{children}</QueryClientProvider>
  );
};
```

## Rules

- Query by role/text/label — not by `data-testid`
- No snapshot tests
- Use `userEvent` not `fireEvent`
- One behavior per `it` block
- Tests go in `src/tests/` — never co-located with source files
