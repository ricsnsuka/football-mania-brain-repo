# Netlify Deployment Guide

This guide covers deploying the Football Mania frontend to Netlify using the official Next.js runtime plugin.

---

## Prerequisites

- A [Netlify account](https://netlify.com)
- The repository pushed to GitHub, GitLab, or Bitbucket
- The backend API deployed and accessible at a stable URL

---

## 1. Install the Netlify Next.js Plugin

Run this in the project root before pushing your deploy branch:

```bash
npm install -D @netlify/plugin-nextjs
```

This plugin handles server-side rendering, API routes, image optimisation, and middleware for Next.js App Router on Netlify's edge infrastructure.

---

## 2. Add `netlify.toml`

Create a `netlify.toml` file at the project root:

```toml
[build]
  command   = "npm run build"
  publish   = ".next"

[[plugins]]
  package = "@netlify/plugin-nextjs"
```

> **Do not set a `functions` directory manually.** The plugin configures Netlify Functions for SSR automatically.

---

## 3. Connect the Repository on Netlify

1. Go to **Netlify Dashboard → Add new site → Import an existing project**.
2. Authorise Netlify to access your Git provider and select this repository.
3. Netlify will auto-detect the framework. Confirm or set:
   - **Build command:** `npm run build`
   - **Publish directory:** `.next`
4. Click **Deploy site** — this first deploy will likely fail because the environment variable is not yet set. That is expected.

---

## 4. Configure Environment Variables

In **Site configuration → Environment variables**, add the following:

| Variable | Required | Description | Example value |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | Yes | Full base URL of the backend API (no trailing slash) | `https://api.footballmania.example.com` |

> `NEXT_PUBLIC_*` variables are embedded into the client bundle at build time. Changing them requires a new deploy — they are **not** read at runtime.

After adding the variable, trigger a new deploy via **Deploys → Trigger deploy → Deploy site**.

---

## 5. Verify the Deploy

Once the deploy succeeds:

1. Open the Netlify-provided URL (e.g. `https://football-mania.netlify.app`).
2. Confirm the login page loads and the API can be reached (attempt to log in).
3. Check **Deploys → Function logs** if any SSR pages return 500 errors.

---

## 6. Configure a Custom Domain (Optional)

1. **Site configuration → Domain management → Add a domain**.
2. Enter your domain and follow the DNS instructions (CNAME or nameserver delegation).
3. Netlify provisions a free TLS certificate via Let's Encrypt automatically once DNS propagates.

---

## 7. Branch Deploys and Preview URLs

Netlify creates a unique preview URL for every pull request and branch push by default.

To control which branches get deployed:

- **Site configuration → Build & deploy → Branches and deploy contexts**
- Set **Production branch** to `main` (or `v1.0.0`).
- Enable **Branch deploys** for feature branches if you want preview environments per branch.

Each preview deploy needs the same `NEXT_PUBLIC_API_URL` value unless you use different backend environments per branch (configurable via deploy contexts in `netlify.toml`):

```toml
[context.production]
  environment = { NEXT_PUBLIC_API_URL = "https://api.footballmania.example.com" }

[context.deploy-preview]
  environment = { NEXT_PUBLIC_API_URL = "https://staging-api.footballmania.example.com" }
```

---

## 8. Redirects for Direct URL Access

The Next.js plugin handles routing automatically. No manual `[[redirects]]` rules are needed for App Router pages.

If you add a custom `_redirects` file or additional redirect rules, place them under `public/` — Netlify merges them with plugin-generated rules.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build fails with `Cannot find module '@netlify/plugin-nextjs'` | Plugin not installed | Run `npm install -D @netlify/plugin-nextjs` and push |
| Pages load but all API calls fail (network error) | `NEXT_PUBLIC_API_URL` not set or wrong | Verify the env var value and redeploy |
| Pages return 500 on server-rendered routes | SSR runtime error | Check **Function logs** in the Netlify dashboard |
| CORS errors in browser console | Backend does not allow the Netlify origin | Add the Netlify domain to the backend CORS allowed-origins list |
| Old env var value still active after update | Variable cached in previous build | Trigger a **clear cache and deploy** from the Deploys tab |

---

## Related

- [Getting Started](getting-started.md) — local development setup
- [Architecture Overview](../architecture/overview.md) — how services and hooks are structured
- [Environment Variables](getting-started.md#environment-variables) — all supported variables
