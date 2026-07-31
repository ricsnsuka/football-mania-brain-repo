# Phase 0 — Frontend Handoff: Replace the Placeholder Assets

**Date:** 2026-07-28
**Target repo:** `FootballMania/front/football` (separate git repository, branch `v1.0.0`)
**Blocks:** Phase 1 (PWA manifest), Phase 2 (push notification icon), Phase 4 (store listings)

> ## ✅ Applied — 2026-07-28, frontend commit `949ddb8`
>
> Delivered alongside Phase 1, which needed the icons before its manifest could reference
> them. **The frontend repo's `docs/features/pwa.md` is now the live reference**; this file is
> kept as the record of what was handed over and why.
>
> **One instruction below was wrong** and is corrected in what shipped: step 5 puts
> `themeColor` in the `metadata` export. It has been deprecated there since Next.js 14 and
> belongs in a separate `viewport` export — in `metadata` it produces a build warning and no
> meta tag. The shipped version also splits the value by colour scheme (`#ffffff` light /
> `#0a0a0a` dark), which the manifest's single `theme_color` cannot express.
>
> Also changed in flight: the manifest is `app/manifest.ts` rather than hand-written JSON, so
> it is type-checked against `MetadataRoute.Manifest`; and the optional navbar-emoji swap in
> step 6 was left undone, since it is cosmetic and touches the two brand rules in
> `globals.css`.

This is the one Phase 0 item that lives outside the backend repository. It is written to be
applied without re-deriving anything — the mark is below in full, the pipeline is a copy-paste,
and the Next.js conventions that make most of the usual `<link rel="icon">` boilerplate
unnecessary are called out where they apply.

---

## Current state

```
public/
  file.svg      ← create-next-app placeholder
  globe.svg     ← create-next-app placeholder
  next.svg      ← create-next-app placeholder
  vercel.svg    ← create-next-app placeholder
  window.svg    ← create-next-app placeholder
src/app/
  favicon.ico   ← the Next.js default, not a brand mark
```

No manifest, no app icon set, no Apple touch icon. `src/app/layout.tsx` sets
`metadata = { title: 'Football Mania', description: 'Football Mania app' }` and nothing else.

The roadmap is right that this blocks rather than polishes: a PWA with no icon is not installable,
a push notification with no icon renders as a grey square, and neither store accepts a listing
without one.

---

## The mark

A football abbreviated to what survives at 16 pixels: a white ball on the app's existing
`blue-600` (`#2563eb`), which is already the primary across buttons, links, active nav and FABs —
so the icon matches the product rather than introducing a second brand colour.

The pentagon and five spokes are the minimum that reads as a football. Below roughly 20px they
merge into a white disc, which still reads correctly in a browser tab. The ball is 300px across
inside a 512px canvas, comfortably within the 80% safe zone Android's maskable icons crop to, so
one file serves every purpose.

Save as **`src/app/icon.svg`**:

```svg
<svg width="512" height="512" viewBox="0 0 512 512" fill="none" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Football Mania">
  <rect width="512" height="512" rx="112" fill="#2563EB"/>
  <circle cx="256" cy="256" r="150" fill="#FFFFFF"/>
  <g stroke="#2563EB" stroke-width="22" stroke-linecap="round">
    <path d="M256 204 V106"/>
    <path d="M305.5 239.9 L398.7 209.6"/>
    <path d="M286.6 298.1 L344.2 377.4"/>
    <path d="M225.4 298.1 L167.8 377.4"/>
    <path d="M206.5 239.9 L113.3 209.6"/>
  </g>
  <path d="M256 204 L305.5 239.9 L286.6 298.1 L225.4 298.1 L206.5 239.9 Z" fill="#2563EB"/>
</svg>
```

It is one flat colour plus white — no gradients, no strokes below 22px, nothing that degrades when
downscaled or flattened to a 16×16 `.ico`. Swapping in a designed logo later means replacing this
one file and re-running the script below; nothing else references the artwork directly.

> **Dark mode:** the mark carries its own background, so it needs no dark variant. Do not make the
> square transparent — an icon that inherits the tab or launcher background disappears against blue.

---

## What Next.js gives you for free

The App Router resolves icon files by convention and injects the `<link>` tags itself. Do **not**
hand-write `<link rel="icon">` / `<link rel="apple-touch-icon">` into `layout.tsx` — it duplicates
what the framework already emits.

| File | Emits |
|------|-------|
| `src/app/icon.svg` | `<link rel="icon" type="image/svg+xml">` |
| `src/app/apple-icon.png` (180×180) | `<link rel="apple-touch-icon">` |
| `src/app/manifest.ts` | `<link rel="manifest">` (Phase 1) |

Only the maskable PWA icons referenced from the manifest need to live in `public/`.

---

## Steps

**1. Add the mark** — save the SVG above as `src/app/icon.svg`.

**2. Generate the raster sizes.** `sharp` is not currently a dependency; add it as a dev one and
script the conversion so regenerating after a logo change is one command rather than a manual
export:

```bash
npm i -D sharp
```

`scripts/generate-icons.mjs`:

```js
import sharp from 'sharp';
import { readFile, mkdir } from 'node:fs/promises';

const svg = await readFile('src/app/icon.svg');
await mkdir('public/icons', { recursive: true });

const targets = [
  // Apple touch icon — App Router picks this up by filename, no <link> needed.
  ['src/app/apple-icon.png', 180],
  // Referenced from the manifest in Phase 1. 192 and 512 are the two Android requires;
  // 512 is also what the install prompt and the splash screen use.
  ['public/icons/icon-192.png', 192],
  ['public/icons/icon-512.png', 512],
];

for (const [out, size] of targets) {
  await sharp(svg, { density: 384 }).resize(size, size).png().toFile(out);
  console.log(`${out} (${size}×${size})`);
}
```

```bash
node scripts/generate-icons.mjs
```

Add it to `package.json` scripts as `"icons": "node scripts/generate-icons.mjs"`.

> `density: 384` matters — `sharp` rasterises SVG at 72 DPI by default, so the 512px output would
> be upscaled from a 512pt render and come out soft. This renders at native resolution first.

**3. Replace the favicon.** Delete `src/app/favicon.ico`. The `icon.svg` covers every browser that
matters; add a 32×32 `.ico` only if analytics show meaningful legacy traffic.

**4. Delete the placeholders — after checking.** They are almost certainly unreferenced, but the
default `page.tsx` imports several of them and a stray import breaks the build:

```bash
grep -rn "file.svg\|globe.svg\|next.svg\|vercel.svg\|window.svg" src/ --include="*.tsx" --include="*.ts"
rm public/file.svg public/globe.svg public/next.svg public/vercel.svg public/window.svg
```

**5. Fill in the metadata.** `description: 'Football Mania app'` is the placeholder the generator
wrote; it becomes the store listing subtitle and the link preview text. In `src/app/layout.tsx`:

```ts
export const metadata: Metadata = {
  title: 'Football Mania',
  description: 'Organise your football group: plan matches, confirm availability, '
    + 'generate balanced teams and track player ratings.',
  applicationName: 'Football Mania',
  themeColor: '#2563eb',
};
```

`themeColor` tints the Android address bar and must match the manifest's value in Phase 1.

**6. Optional — replace the navbar emoji.** `.navbar-brand-icon` and `.login-brand__icon` in
`globals.css` are sized for a text glyph (`text-2xl` / `text-4xl`). If the emoji is swapped for the
SVG, those two rules need width/height instead of font-size. Cosmetic; not blocking.

---

## Definition of done

- [ ] `src/app/icon.svg` present, `src/app/apple-icon.png` at 180×180
- [ ] `public/icons/icon-192.png` and `icon-512.png` generated
- [ ] `scripts/generate-icons.mjs` committed with the `npm run icons` script
- [ ] All five `create-next-app` placeholders deleted, `npm run build` still green
- [ ] `metadata.description` and `themeColor` set
- [ ] Tab icon shows the mark in light **and** dark browser themes

Phase 1 then adds `src/app/manifest.ts` pointing at `public/icons/*` with
`purpose: 'maskable'` — at which point the Lighthouse PWA audit has real icons to check against.
