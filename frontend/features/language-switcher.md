# Language Switcher Feature

The app supports three locales: **English** (`en`), **European Portuguese** (`pt`), and **Spanish** (`es`).

## How it works

| Layer | File | Responsibility |
|-------|------|----------------|
| Type | `src/types/locale.ts` | `Locale = 'en' \| 'pt' \| 'es'`; `LOCALES` array; `LOCALE_META` display record |
| Store | `src/store/appStore.ts` | `locale` state + `setLocale()`, persists to `localStorage`, hydrated on init |
| Provider | `src/providers/I18nProvider.tsx` | Syncs `locale` from store into `i18next` on every change |
| Config | `src/lib/i18n.ts` | i18next configured with `supportedLngs: ['en', 'pt', 'es']` |
| Locale files | `src/locales/{en,pt,es}/common.json` | Translation strings per locale |
| Widget | `src/components/ui/LanguageWidget.tsx` | Dropdown showing current locale flag + code; opens menu listing all 3 options |
| CSS | `src/app/globals.css` | `.lang-widget`, `.lang-menu`, `.lang-menu-item`, `.lang-menu-item--active` |

## Usage

`LanguageWidget` is rendered inside the `Footer`. No additional wiring is needed.

To read or change the locale from any component:

```ts
import { useAppStore } from '@/store/appStore';
import type { Locale } from '@/types/locale';

const locale = useAppStore((s) => s.locale);        // 'en' | 'pt' | 'es'
const setLocale = useAppStore((s) => s.setLocale);  // persists to localStorage

setLocale('pt');
```

## Adding a new locale

1. Add the new code to `Locale` in `src/types/locale.ts` and update `LOCALES` + `LOCALE_META`.
2. Create `src/locales/{code}/common.json` with all translation keys.
3. Add the code to `supportedLngs` in `src/lib/i18n.ts`.

## Translation keys

All locale files (`common.json`) must include a `language` block:

```json
"language": {
  "en": "...",
  "pt": "...",
  "es": "...",
  "select": "..."
}
```

`language.select` is used as the accessible `aria-label` for the toggle button.

## Locale metadata

`LOCALE_META` in `src/types/locale.ts` maps each locale to:

| Field | Purpose |
|-------|---------|
| `flag` | Flag emoji shown in the toggle button and menu |
| `label` | English display name (for logging/admin) |
| `nativeLabel` | Name in that language — shown in the menu to help users who don't read the current UI language |
