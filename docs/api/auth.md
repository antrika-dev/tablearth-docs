# Authentication Modes

tableArth.ai exposes the same lifecycle (init → upload → chat) under two
authentication schemes, plus an optional per-user upgrade for API clients. Pick
the one that matches your surface.

## Overview

| Mode | Credential | Customer resolved from | Used by |
|---|---|---|---|
| **API key** | `apiKey` header | The key's tenant binding | Chrome extension, server-to-server |
| **Widget SSO** | `antrikaSSO` cookie + per-widget signed request | SSO token claims | JS widget |
| **Per-user on API (`/v2`)** | `apiKey` header **+** `ssoToken` JWT header | JWT claims | Server-to-server needing per-user identity |

All modes share the same agent-level controls:

- Agent must be **ACTIVE**.
- Customer must be **ACTIVE**.
- Tenant plan must include the relevant features (see [feature gating](#feature-gating)).
- `Origin` (and/or IP) must match the agent's allow-list when configured.

## API Key

The simplest mode. Issued per agent, scoped by Origin and/or IP allow-list.

### Request shape

```
POST /emp/1/api/tableai/chat HTTP/1.1
Host:    yourcompany.tableArth.ai
apiKey:  ak_live_4f8c2e7d9b1a6c3e5f8d2b9a4e7c1f6d
Origin:  https://your-app.example.com
Content-Type: application/json

{ "sessionId": "…", "query": "…", "stream": true }
```

> The API-key `chat` body is **flat** — `{ sessionId, query, stream }`. It is
> **not** wrapped in a `data` field. (Only the widget `chat` route wraps it.)

### Failure responses

| Code | Body | Cause |
|---|---|---|
| 401 | `{ ed: "Missing API key" }` | No `apiKey` header. |
| 401 | `{ ed: "Invalid API key" }` | Key not found / inactive. |
| 403 | `{ ed: "Not authorised" }` | Origin/IP not in allow-list. |
| 403 | `{ ed: "Agent is not active" }` | Agent status ≠ ACTIVE. |
| 403 | `{ ed: "Customer is not active" }` | Tenant suspended. |
| 403 | `{ ed: "Table AI subscription is not enabled for this account." }` | Plan missing the base module. |
| 403 | `{ ed: "API access is not enabled for your plan." }` | Plan missing API access. |
| 429 | `{ ed: "Too many requests…" }` | Rate-limited (notably the extension `validate`/`domains` routes). |

### Origin / IP allow-list

- **Empty list + empty IP** → any Origin is accepted (avoid in production).
- **Origins populated** → exact match required, `*` glob supported:
  - `https://app.example.com` — exact
  - `https://*.example.com` — any subdomain
  - `https://*.staging.*.example.com` — multi-glob
- **Server IP set** → requests from that IP are accepted regardless of Origin.
- A key may set origins, an IP, or both.

## Widget SSO

Used by the JS widget. There is **no host-minted token on the request itself** and
**no agent secret in the browser**. Instead:

1. Your server mints a short-lived **JWT** (HS256, signed with the tenant **SSO
   signing key** from admin → SSO settings).
2. The page calls `window.antrika('syncUser', { customerId, token })`. The backend
   verifies the JWT, creates/fetches the User + Account from its claims, and sets
   the **`antrikaSSO`** session cookie.
3. The widget then calls the `/emp/1/api/tableai/widget/*` routes, signing each
   request internally with the agent's `widget_id` and a per-widget token, and
   carrying the `antrikaSSO` cookie. The host does not compute this.

### SSO token contract

Algorithm: **HS256**. Signing key: the tenant's **SSO signing key** (admin → SSO
settings) — *not* the agent API key.

| Claim | Type | Description |
|---|---|---|
| `id` | string | Stable user ID in your system. (This is the user id claim — **not** `sub`.) |
| `name` | string | Display name. |
| `email` | string | User email. |
| `companies` | array | ≥ 1 `{ id, name }` — maps to a backend Account. |
| `iat` | number | Issued-at, unix seconds. |
| `exp` | number | Expiry, unix seconds. **≤ 1 hour recommended.** |

Optional: `thumb` (avatar URL), `timezone`, `tags` (string array),
`customFields` (object), and `companies[].customFields`.

### Request shape (issued by the widget)

```
GET /emp/1/api/tableai/widget/init?widget_id=agt_xxx&ssoToken=<cookie>&tlp-k=<sig>&tlp-t=<ts> HTTP/1.1
Host:     yourcompany.tableArth.ai
widget_id: agt_xxx
Origin:    https://your-app.example.com
Cookie:    antrikaSSO=eyJhbGciOiJIUzI1NiJ9…
```

> The widget `chat` body **is** wrapped: `{ "data": { sessionId, query, stream } }`.

### Why this shape

- Cross-origin embeds work without third-party cookies on *your* domain — the
  session cookie is on the tableArth.ai host.
- File uploads (multipart) don't fight CORS preflight.
- The host rotates per-user access by re-minting the SSO JWT and calling
  `syncUser` again.

Examples (mint the SSO JWT): [Node.js](../examples/jwt-mint-node.md) ·
[Java](../examples/jwt-mint-java.md).

## Per-user identity on API routes (`/v2`)

Server-to-server clients that want **per-user** attribution (instead of the
service-wide key alone) can use the `/v2` API routes, which accept a host-minted
`ssoToken` **JWT header** alongside the `apiKey`:

```
GET /emp/1/api/tableai/init/v2 HTTP/1.1
apiKey:   ak_live_…
ssoToken: eyJhbGciOiJIUzI1NiJ9…      // JWT identifying the end user
Origin:   https://your-server.example.com
```

- Routes: `/emp/1/api/tableai/{init,chat,upload}/v2`.
- The JWT carries the same user-claim shape as the [SSO token](#sso-token-contract)
  and is verified with the **API key's** secret.
- Requires the **single sign-on** feature on the plan; otherwise `403` with
  `ed: "This feature is not available at the moment. Please contact your admin."`

Most REST integrations don't need this — use plain [API key](#api-key) and tag
sessions with `customerId` / `userId` headers instead.

## Feature gating

The base module **must** be enabled (`agent-data-query`), plus the per-capability
flag for your surface:

| Capability | Plan flag | Surface |
|---|---|---|
| Base data-query | `agent-data-query` | all |
| API access | `apiAccess` | Chrome extension, server-to-server |
| Widget integration | `widgetIntegration` | JS widget |
| Single sign-on | `singleSignOn` | Widget chat/init, all `/v2` routes |

Missing flags return `403`. See [agent setup](../admin/agent-setup.md#prerequisites).

## Picking a mode

```
Server with no UI?
  └─ API Key   (add ssoToken on /v2 routes if you need per-user identity)

Browser surface, embedded in YOUR product?
  └─ Widget SSO (JS widget)

Browser surface, embedded in OTHER people's products?
  └─ Chrome extension (which uses API Key under the hood)
```

## Quick comparison

| | API Key | Widget SSO |
|---|---|---|
| Per-user history | No on its own (yes on `/v2`) | Yes |
| Host setup cost | Low (paste a key) | Low–medium (mint JWT + `syncUser`) |
| Cross-origin friendly | Yes | Yes |
| File upload friendly | Yes | Yes |
| Rotation | Recreate the key | Re-mint the SSO JWT |
| Audit who asked what | Approximate (session metadata) | Yes |
