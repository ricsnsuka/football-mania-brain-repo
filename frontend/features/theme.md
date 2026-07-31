# Theme Feature

The app supports **light**, **dark**, and **system** (follows OS preference) themes.

## How it works

| Layer | File | Responsibility |
|-------|------|----------------|
| Type | `src/types/theme.ts` | `Theme = 'light' \| 'dark' \| 'system'` |
| Store | `src/store/appStore.ts` | `theme` state + `setTheme()`, persists to `localStorage` |
| Provider | `src/providers/ThemeProvider.tsx` | Hydrates from `localStorage` on mount; applies/removes `.dark` class on `<html>`; listens for `prefers-color-scheme` changes when theme is `'system'` |
| Anti-FOUC | `src/app/layout.tsx` | Inline `<script>` in `<head>` applies `.dark` before first paint |
| Widget | `src/components/ui/ThemeWidget.tsx` | Cycles `light → dark → system` on each click |
| CSS | `src/app/globals.css` | Tailwind v4 `@custom-variant dark`; `.dark` CSS variable overrides; dark variants on all component classes |

## Usage

The `ThemeWidget` is rendered inside the `Footer`. No additional wiring is needed.

To read or change the theme from any component:

```ts
import { useAppStore } from '@/store/appStore';

const theme = useAppStore((s) => s.theme);        // 'light' | 'dark' | 'system'
const setTheme = useAppStore((s) => s.setTheme);  // persists to localStorage

setTheme('dark');
```

## Adding dark styles to new components

Tailwind's `dark:` variant is configured to respond to the `.dark` class on `<html>`. Use it in `@apply` blocks inside `globals.css`:

```css
.my-component {
  @apply bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100;
}
```

For plain CSS selectors (e.g. notification colours), prefix with `.dark`:

```css
.dark .my-component { background: #1a1a1a; }
```
