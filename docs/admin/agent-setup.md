# Agent Setup (Admin)

Before any tableArth.ai integration works, a tenant admin needs to create an AI
Agent of type *data query* and configure its access controls. This page is for the
admin doing that one-time setup.

## Prerequisites

Your tenant plan must include the base module plus the per-surface flag:

| Capability | Plan flag | Needed for |
|---|---|---|
| Base data-query | `agent-data-query` | all surfaces |
| API access | `apiAccess` | Chrome extension, server-to-server |
| Widget integration | `widgetIntegration` | JS widget |
| Single sign-on | `singleSignOn` | JS widget, `/v2` API routes |

If a flag is missing, integration calls return `403` (e.g. *"API access is not
enabled for your plan."* or the generic *"This feature is not available at the
moment. Please contact your admin."*). Contact your account manager.

## Step 1 — Create the agent

1. Admin → **AI Agents** → **New Agent**.
2. Pick type **data query** (tableArth.ai).
3. Name it after the surface it'll serve (e.g. *Reports — Production*,
   *Reports — Staging*). One agent per environment is the cleanest split.
4. Pick the model + system prompt. Defaults are fine for first-pass.
5. Save → status defaults to **DRAFT**. Flip to **ACTIVE** before integrating.

The agent's **`agentId`** is the public identifier you'll paste into integrations
(the JS widget calls it `widgetId` / `widget_id`).

## Step 2 — Origin allow-list

Restricts which web origins (or server IPs) can use this agent. The Chrome
extension reads this list to decide where its badges appear.

| Pattern | Matches |
|---|---|
| `https://app.example.com` | exact origin |
| `https://*.example.com` | any subdomain |
| `https://*.staging.*.example.com` | multi-glob |
| (empty list, no IP) | **anything** — avoid in production |

Origin matching is exact unless `*` appears; each `*` matches any sequence of
characters within the URL.

For server-to-server, set the **Server IP** field instead (or in addition).

## Step 3 — Issue credentials

Pick the credential type for each integration:

### A. API Key (Chrome extension, server-to-server)

1. Agent → **API Keys** → **Create Key**.
2. Name it after the consumer (`chrome-ext`, `etl-job`). Helps in audit.
3. Optionally restrict the key to a subset of the agent's Origins / IPs.
4. Copy the raw key once; it's not shown again.

### B. SSO signing key (JS widget)

The widget identifies visitors via an SSO session, not an API key. Its integrator
needs the tenant's **SSO signing key**:

1. Admin → **SSO settings** → copy (or create) the **signing key**. This is the
   HS256 key the host server uses to mint the `syncUser` token.
2. **Server-side only.** Never embed it in HTML or JS bundles.
3. The widget's own per-request signing secret is provisioned automatically — the
   bundle fetches it; the integrator doesn't handle it.

SSO token contract: see [auth.md → Widget SSO](../api/auth.md#sso-token-contract).

## Step 4 — Quick sanity check

Validate an API key from the command line before handing it over:

```bash
curl -i \
  -X POST 'https://yourcompany.tableArth.ai/emp/1/api/tableai/extension/validate' \
  -H 'apiKey: ak_live_…' \
  -H 'Origin: https://app.example.com' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

Expected: `200 { "s": true }`.

Common failures and what they mean: [troubleshooting.md](../troubleshooting.md).

## Step 5 — Hand off

Give the integrating developer:

- The base URL (e.g. `https://yourcompany.tableArth.ai`).
- The **agent ID** (`agentId` / `widgetId`).
- For API-key consumers (extension, server-to-server): the **raw key**.
- For the JS widget: the **tenant / customer ID** and tell them to ask their
  backend team to fetch the **SSO signing key** from SSO settings and configure
  their token minter.

Then point them at the integration doc for the surface they're building:

- [Widget](../integrations/widget.md)
- [Chrome extension](../integrations/chrome-extension.md)
- [Server-to-server](../integrations/api.md)

## Auditing

Each session records the **source** — `API`, `Widget`, `Extension`, or `Admin` —
plus the optional `webpage` / `tableName` / `customerId` / `userId` the client
sent. Use admin → **Sessions** to filter.

Per-message reactions (👍 / 👎) flow into the same view, scoped to the session and
user.

## Operational tips

- **One agent per environment** keeps prod/stage Origin allow-lists separate.
- **One key per consumer** lets you revoke without breaking unrelated clients.
- **Short SSO token expiries** (≤ 1 hour) limit blast radius when a token leaks.
- **Rotate the SSO signing key** after any suspected compromise — old tokens stop
  validating immediately, forcing your servers to mint with the new key.
