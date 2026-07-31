# Notification Widget

Component: `NotificationWidget` · Placement: Root layout · Auth: Any

---

## Overview

A fixed, bottom-right toast notification system. Notifications are pushed to the Zustand store from anywhere in the app, displayed for 5 seconds with a slide-in/slide-out animation, and dismissed automatically or via a close button.

---

## Files

| File | Description |
|---|---|
| `src/types/notification.ts` | `NotificationType` union + `Notification` interface |
| `src/store/appStore.ts` | Notification slice: `notifications[]`, `addNotification`, `removeNotification` |
| `src/components/ui/NotificationWidget.tsx` | `'use client'` component — renders all active notifications |
| `src/app/globals.css` | `.notification-widget`, `.notification-item`, `.notification-item--{type}`, keyframe animations |
| `src/tests/components/NotificationWidget.test.tsx` | RTL + Vitest tests (5 cases) |
| `src/locales/en/common.json` | `notification.close` key |
| `src/locales/pt/common.json` | `notification.close` key (PT) |

---

## Usage

Call `addNotification` from anywhere that has access to the store:

```ts
import { useAppStore } from '@/store/appStore';

const addNotification = useAppStore((s) => s.addNotification);

// success | warning | info | error
addNotification('success', 'Profile saved.');
addNotification('error', 'Something went wrong.');
addNotification('warning', 'Your session will expire soon.');
addNotification('info', 'New version available.');
```

---

## Behavior

| Property | Value |
|---|---|
| Position | Fixed, bottom-right (`z-index: 9999`) |
| Auto-dismiss | 5 000 ms |
| Exit animation start | 4 700 ms (300 ms before removal) |
| Enter animation | Slide-in from right (`cubic-bezier(0.34, 1.56, 0.64, 1)`) |
| Exit animation | Slide-out to right (`ease-in`) |
| Manual dismiss | Close (✕) button |
| Multiple notifications | Stack vertically, each with its own independent timer |

---

## Colors

| Type | Background | Border | Text |
|---|---|---|---|
| `success` | `#f0fdf4` | `#22c55e` | `#166534` |
| `warning` | `#fefce8` | `#eab308` | `#854d0e` |
| `info` | `#eff6ff` | `#3b82f6` | `#1e40af` |
| `error` | `#fef2f2` | `#ef4444` | `#991b1b` |

---

## Layout Notes

The root layout (`src/app/layout.tsx`) uses `h-screen overflow-hidden` on `<body>` and `overflow-y-auto` on `<main>`, so:
- **Navbar** (`sticky top-0 z-50`) — always visible at the top
- **Footer** — always pinned at the bottom
- **Content** — scrolls independently inside `<main>`
- **NotificationWidget** — sits outside the scroll container, always on top (`z-index: 9999`)
