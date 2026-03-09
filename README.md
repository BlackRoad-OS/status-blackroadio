# status.blackroad.io — System Status Page

> **BlackRoad OS, Inc.** — Production-grade system status page deployed on Cloudflare Workers at [status.blackroad.io](https://status.blackroad.io).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Installation](#4-installation)
5. [Local Development](#5-local-development)
6. [Deployment](#6-deployment)
7. [Environment Variables & Secrets](#7-environment-variables--secrets)
8. [npm Package](#8-npm-package)
9. [Stripe Integration](#9-stripe-integration)
10. [End-to-End (E2E) Testing](#10-end-to-end-e2e-testing)
11. [CI/CD — GitHub Actions](#11-cicd--github-actions)
12. [API Reference](#12-api-reference)
13. [Project Structure](#13-project-structure)
14. [License](#14-license)

---

## 1. Overview

`status-blackroadio` is the official system status page for the BlackRoad OS platform. It is an edge-deployed [Cloudflare Worker](https://workers.cloudflare.com/) that:

- Polls the BlackRoad OS health API (`/health`) every 30 seconds.
- Displays real-time agent count, repository count, and platform health.
- Auto-refreshes every 30 seconds in-browser via `<meta http-equiv="refresh">`.
- Serves globally from Cloudflare's edge network with zero cold starts.

**Live URL:** `https://status.blackroad.io`

---

## 2. Architecture

```
Browser / curl
     │
     ▼
Cloudflare Edge (status.blackroad.io)
     │
     ├─ src/index.js  ← Cloudflare Worker (this repo)
     │       │
     │       ├─ GET /health  → blackroad-os-api.amundsonalexa.workers.dev
     │       └─ renders HTML status page
     │
     └─ Response: text/html — cache TTL 30s
```

| Layer | Technology |
|---|---|
| Runtime | Cloudflare Workers (V8 isolates) |
| Bundler / CLI | Wrangler v3 |
| Node.js (dev only) | v20 LTS |
| CI/CD | GitHub Actions |
| Domain | `status.blackroad.io` (zone: `blackroad.io`) |

---

## 3. Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Node.js | `≥ 20 LTS` | Local dev & Wrangler CLI |
| npm | `≥ 10` | Dependency management |
| Wrangler CLI | `^3.0.0` (installed via `npm`) | Build, preview, deploy |
| Cloudflare account | — | Production deployment |
| `CLOUDFLARE_API_TOKEN` | — | Scoped API token (see §7) |

---

## 4. Installation

```bash
# Clone the repository
git clone https://github.com/BlackRoad-OS/status-blackroadio.git
cd status-blackroadio

# Install dependencies (Wrangler dev tooling only)
npm install
```

---

## 5. Local Development

```bash
# Start the local Wrangler dev server (hot-reload)
npm run dev
```

Wrangler will start a local preview at `http://localhost:8787`. The Worker fetches live data from the production health API, so an active internet connection is required.

```bash
# Stream real-time logs from the production worker
npm run tail
```

---

## 6. Deployment

Deployment is handled automatically by GitHub Actions on every push to `main` (see §11). Manual deployment:

```bash
# Deploy to production (status.blackroad.io)
npm run deploy
```

This runs `wrangler deploy`, which:
1. Bundles `src/index.js`.
2. Uploads the Worker to Cloudflare.
3. Routes `status.blackroad.io/*` to the Worker.

> **Note:** `CLOUDFLARE_API_TOKEN` must be set in your environment before running a manual deploy. See §7.

---

## 7. Environment Variables & Secrets

| Variable | Where set | Required | Description |
|---|---|---|---|
| `CLOUDFLARE_API_TOKEN` | GitHub Secrets / local env | ✅ Yes | Cloudflare API token with *Worker Scripts:Edit* and *Zone:Read* permissions |
| `CLOUDFLARE_ACCOUNT_ID` | `wrangler.toml` / GitHub Actions | ✅ Yes | Cloudflare Account ID (`848cf0b18d51e0170e0d1537aec3505a`) |
| `ENVIRONMENT` | `wrangler.toml` (`[vars]`) | ✅ Yes | Set to `"production"` |

### Generating a Cloudflare API Token

1. Log in to the [Cloudflare Dashboard](https://dash.cloudflare.com/profile/api-tokens).
2. Click **Create Token → Custom Token**.
3. Grant `Account > Workers Scripts > Edit` and `Zone > Zone > Read` permissions.
4. Scope to account `BlackRoad OS, Inc.`.
5. Copy the token and add it as a GitHub secret named `CLOUDFLARE_API_TOKEN`.

---

## 8. npm Package

The package is published under the **`@blackroad`** scope:

```
@blackroad/status-blackroadio   v2.0.0
```

> This Worker is currently marked `"private": true` in `package.json`. To publish to the npm registry, update `package.json`:

```jsonc
{
  "name": "@blackroad/status-blackroadio",
  "version": "2.0.0",
  "private": false,          // ← change to false for npm publish
  "description": "BlackRoad OS — System Status Page Worker",
  "main": "src/index.js",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "tail": "wrangler tail"
  }
}
```

Then publish:

```bash
npm login --scope=@blackroad
npm publish --access public
```

---

## 9. Stripe Integration

> **Status:** Ready for integration. The Worker is production-deployed on Cloudflare's global edge, making it a suitable backend hook for Stripe webhooks and billing status endpoints.

### Webhook Handler (example pattern)

To receive Stripe webhooks via this Worker, extend `src/index.js`:

```js
import Stripe from 'stripe';   // npm install stripe

const stripe = new Stripe(env.STRIPE_SECRET_KEY);

// Inside export default { fetch(request, env, ctx) { ... } }
if (request.method === 'POST' && url.pathname === '/webhooks/stripe') {
  const sig   = request.headers.get('stripe-signature');
  const body  = await request.text();
  const event = await stripe.webhooks.constructEventAsync(
    body, sig, env.STRIPE_WEBHOOK_SECRET
  );
  // Handle event.type …
  return new Response('OK', { status: 200 });
}
```

Add the following secrets to Cloudflare Workers (via Dashboard or `wrangler secret put`):

```bash
wrangler secret put STRIPE_SECRET_KEY
wrangler secret put STRIPE_WEBHOOK_SECRET
```

---

## 10. End-to-End (E2E) Testing

E2E tests validate the full request → Worker → response cycle.

### Smoke Test (curl)

```bash
# Test production endpoint
curl -I https://status.blackroad.io
# Expect: HTTP/2 200, content-type: text/html;charset=UTF-8

# Assert page content
curl -s https://status.blackroad.io | grep -q "System Status Page" && echo "PASS" || echo "FAIL"
```

### Local E2E (Wrangler dev)

```bash
# Terminal 1 — start local worker
npm run dev

# Terminal 2 — run assertions
curl -s http://localhost:8787 | grep -q "System Status Page" && echo "PASS" || echo "FAIL"
curl -s http://localhost:8787 | grep -q "live-badge" && echo "PASS: live badge" || echo "FAIL: live badge"
```

### Automated E2E with Playwright (recommended)

```bash
npm install --save-dev @playwright/test
npx playwright install chromium
```

Create `e2e/status.spec.js`:

```js
import { test, expect } from '@playwright/test';

test('status page loads and shows live badge', async ({ page }) => {
  await page.goto('https://status.blackroad.io');
  await expect(page).toHaveTitle(/System Status Page/);
  await expect(page.locator('.live-badge')).toBeVisible();
  await expect(page.locator('.stat .val').first()).not.toBeEmpty();
});
```

Run:

```bash
npx playwright test
```

---

## 11. CI/CD — GitHub Actions

The workflow at `.github/workflows/deploy.yml` automatically deploys on every push to `main`:

```
push → main
  └─ actions/checkout@v4
  └─ actions/setup-node@v4  (Node 20)
  └─ npm install
  └─ npx wrangler deploy    (uses CLOUDFLARE_API_TOKEN secret)
```

**Required GitHub Secrets:**

| Secret | Description |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token (see §7) |

---

## 12. API Reference

### `GET /`

Returns the rendered HTML status page.

| Header | Value |
|---|---|
| `Content-Type` | `text/html;charset=UTF-8` |
| `Cache-Control` | `public, max-age=30` |
| `X-BlackRoad-Worker` | `status-blackroadio` |
| `X-BlackRoad-Version` | `2.0.0` |

The page auto-refreshes every 30 seconds.

### Upstream: `GET /health`

The Worker polls `https://blackroad-os-api.amundsonalexa.workers.dev/health` to surface:

| Field | Description |
|---|---|
| `status` | `"ok"` \| `"degraded"` \| `"down"` |
| `agents` | Number of agents currently online |

---

## 13. Project Structure

```
status-blackroadio/
├── .github/
│   └── workflows/
│       └── deploy.yml        # GitHub Actions CI/CD
├── src/
│   └── index.js              # Cloudflare Worker entry point
├── LICENSE                   # Proprietary — All Rights Reserved
├── package.json              # npm manifest (@blackroad/status-blackroadio v2.0.0)
├── README.md                 # This file
└── wrangler.toml             # Cloudflare Workers configuration
```

---

## 14. License

**Proprietary — All Rights Reserved**

© 2024–2026 BlackRoad OS, Inc. Unauthorized use, reproduction, or distribution is strictly prohibited. See [LICENSE](./LICENSE) for full terms.

---

<p align="center">
  <a href="https://blackroad.io">blackroad.io</a> ·
  <a href="https://dashboard.blackroad.io">Dashboard</a> ·
  <a href="https://agents.blackroad.io">Agents</a> ·
  <a href="https://docs.blackroad.io">Docs</a> ·
  <a href="https://status.blackroad.io">Status</a>
</p>
