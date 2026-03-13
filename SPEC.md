# Web2Kindle — Spec

## Overview

Web2Kindle is a web app that converts URLs into PDFs and delivers them to a user's Kindle device via email. Users sign in with Google or GitHub, register their Kindle devices, and submit URLs through a simple form.

## Stack

| Layer | Technology |
|-------|------------|
| Runtime | Cloudflare Workers |
| Framework | Hono |
| Auth | Better Auth (Google + GitHub social login) |
| Database | Cloudflare D1 (SQLite) |
| DB access | Kysely + kysely-d1 |
| PDF rendering | @cloudflare/puppeteer (Browser binding) |
| Email | Resend |
| Validation | Zod |
| Frontend | Server-rendered HTML (Hono JSX) + minimal JS |

## Data Model

Better Auth creates its own tables (`user`, `session`, `account`, `verification`). We add one application table:

```sql
CREATE TABLE kindle_device (
  id        TEXT PRIMARY KEY,
  user_id   TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  name      TEXT NOT NULL,           -- e.g. "Living Room Kindle"
  email     TEXT NOT NULL,           -- e.g. "alice_abc123@kindle.com"
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE UNIQUE INDEX idx_device_user_email ON kindle_device(user_id, email);
```

## Auth

- Better Auth configured with Google and GitHub social providers
- Factory pattern: `createAuth(d1)` called per-request (CF Workers requirement — bindings aren't available at module scope)
- Hono middleware extracts session and sets `c.var.user` / `c.var.session`
- All routes except `/api/auth/*` and `/` require an active session

### OAuth Setup

Each provider needs a registered OAuth app with callback URL:
```
https://<domain>/api/auth/callback/google
https://<domain>/api/auth/callback/github
```

Environment secrets: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `RESEND_KEY`, `FROM_EMAIL`.

## Pages & Routes

### Public

| Route | Description |
|-------|-------------|
| `GET /` | Landing page with "Sign in with Google / GitHub" buttons |
| `GET/POST /api/auth/*` | Better Auth handler (login, callback, logout, session) |

### Authenticated

| Route | Description |
|-------|-------------|
| `GET /dashboard` | Main page: URL submission form + device picker + device list |
| `POST /devices` | Add a Kindle device (name + email) |
| `POST /devices/:id/delete` | Remove a Kindle device |
| `POST /send` | Submit a URL to be rendered and sent to the selected device |

## Core Flows

### 1. Sign Up / Sign In

1. User clicks "Sign in with Google" (or GitHub) on landing page
2. Better Auth redirects to provider → user authorizes → callback
3. Better Auth creates/updates `user` + `account` rows, sets session cookie
4. Redirect to `/dashboard`

### 2. Add a Kindle Device

1. On `/dashboard`, user fills in device name + Kindle email address
2. `POST /devices` validates and inserts into `kindle_device`
3. Redirect back to `/dashboard` showing updated device list
4. UI reminds user to add our sender address to their Kindle approved senders list

### 3. Send a URL to Kindle

1. User pastes a URL into the form, picks a device from dropdown, clicks "Send"
2. `POST /send` validates input, confirms device belongs to the user
3. Server launches Puppeteer, navigates to URL, dismisses cookie banners, renders PDF
4. Server emails the PDF to the device's Kindle email via Resend
5. Response confirms success or reports error

## Architecture Decisions

**No Durable Objects.** Launch a fresh browser per request. Simpler code, acceptable latency for a form submission the user is already waiting on.

**No Workflows.** Synchronous request: render → email → respond. If it fails, the user sees the error and can retry. No need for background orchestration.

**No KV cache.** Not worth the complexity. If the same URL is submitted twice, we render it again.

**No browser extension.** The web app *is* the interface.

**Server-rendered HTML.** Hono's JSX support renders pages on the server. Minimal client-side JS for form interactions only. No build step, no SPA framework.

## File Structure

```
src/
  index.ts          — Hono app, routes, middleware
  auth.ts           — createAuth() factory, Better Auth config
  db.ts             — Kysely helpers, device CRUD queries
  render.ts         — Puppeteer PDF rendering
  email.ts          — Resend email delivery
  pages/
    landing.tsx     — Landing / sign-in page
    dashboard.tsx   — Device management + URL submission form
    layout.tsx      — Shared HTML shell
wrangler.json       — Worker config: D1 binding, Browser binding, secrets
schema.sql          — D1 migration for kindle_device table
```

## What Gets Deleted

- `src/workflow.ts` — Workflow class
- `src/browser.ts` — Durable Object
- `src/types.d.ts`
- `public/extension.html`, `public/extension.js`, `public/manifest.json`
- `public/images/` — Extension icons
- `extension.sh` — Extension build script
- `web2kindle-starter.zip`
- `images/` — README screenshots
- KV namespace, DO, and Workflow config from `wrangler.json`
