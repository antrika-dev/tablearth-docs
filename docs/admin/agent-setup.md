# Agent Setup (Admin)

Before any TableAI integration works, a tenant admin needs to create an AI Agent of type *TableAI* and configure its access controls. This page is for the admin doing that one-time setup.

## Prerequisites

Your tenant plan must include:

- **Base TableAI** — required for all surfaces.
- One or more of the per-mode add-ons:
  - **Widget access** — JS widget embeds.
  - **API access** — API-key endpoints (Chrome extension, Excel add-in, server-to-server).

If any of these are missing, integration calls return `403 "This feature is not available at the moment. Please contact your admin."` Contact your account manager.

## Step 1 — Create the agent

1. Admin → **AI Agents** → **New Agent**.
2. Pick type **TableAI**.
3. Name it after the surface it'll serve (e.g. *Reports — Production*, *Reports — Staging*). One agent per environment is the cleanest split.
4. Pick the model + system prompt. Defaults are fine for first-pass.
5. Save → status defaults to **DRAFT**. Flip to **ACTIVE** before integrating.

The agent's **`agentId`** is the public identifier you'll paste into integrations (e.g. as `widgetId` for the JS widget).

## Step 2 — Origin allow-list

Restricts which web origins (or server IPs) can use this agent.

| Pattern | Matches |
|---|---|
| `https://app.example.com` | exact origin |
| `https://*.example.com` | any subdomain |
| `https://*.staging.*.example.com` | multi-glob |
| (empty list, no IP) | **anything** — avoid in production |

Origin matching is exact unless `*` appears; each `*` matches any sequence of characters within the URL.

For server-to-server, set the **Server IP** field instead (or in addition).

## Step 3 — Issue credentials

Pick the credential type for each integration:

### A. API Key (Chrome ext, Excel add-in, server-to-server)

1. Agent → **API Keys** → **Create Key**.
2. Name it after the consumer (`chrome-ext`, `excel-addin`, `etl-job`). Helps in audit.
3. Optionally restrict the key to a subset of the agent's Origins / IPs.
4. Copy the raw key once; it's not shown again.

### B. JWT secret (JS widget)

1. Agent → **Show secret**. This is the HS256 signing key your server will use to mint per-user JWTs.
2. **Server-side only.** Never embed in HTML or JS bundles.
3. Rotate via *Regenerate secret* — old tokens become invalid immediately.

JWT contract: see [auth.md → JWT in `ssoToken` header](../api/auth.md#jwt-in-ssotoken-header).

## Step 4 — Quick sanity check

Before handing the agent ID and key over to a developer, validate from the command line:

```bash
curl -i \
  -X POST 'https://yourcompany.antrika.com/emp/1/api/tableai/apikey/validate' \
  -H 'apiKey: ak_live_…' \
  -H 'Origin: https://app.example.com' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

Expected: `200 { "s": true }`.

Common failures and what they mean: [troubleshooting.md](../troubleshooting.md).

## Step 5 — Hand off

Give the integrating developer:

- The base URL (e.g. `https://yourcompany.antrika.com`).
- The **agent ID** (`agentId` / `widgetId`).
- For API-key consumers: the **raw key**.
- For widget JWT consumers: tell them to ask their own backend team to fetch the **agent secret** from admin and configure their JWT minter.

Then point them at the integration doc for the surface they're building:

- [Widget](../integrations/widget.md)
- [Chrome ext](../integrations/chrome-extension.md)
- [Excel add-in](../integrations/excel-addin.md)
- [Server-to-server](../integrations/api.md)

## Auditing

Each session records the source (`API` / `WIDGET` / `EXTENSION`) plus the optional `webpage` / `tableName` / `customerId` / `userId` headers the client sent. Use admin → **Sessions** to filter.

Per-message reactions (👍 / 👎) flow into the same view, scoped to the session and user.

## Operational tips

- **One agent per environment** keeps prod/stage Origin allow-lists separate.
- **One key per consumer** lets you revoke without breaking unrelated clients.
- **Short JWT expiries** (≤ 1 hour) limit blast radius when a token leaks.
- **Rotate the agent secret** after any suspected compromise — JWT signing breaks immediately for old tokens, forcing your servers to mint with the new secret.
