# Authentication Modes

TableAI exposes the same lifecycle (init → upload → chat) under two auth schemes, each suited to a different surface. Pick the one that matches your integration.

## Overview

| Mode | Header(s) | Customer resolved from | Used by |
|---|---|---|---|
| **API Key** | `apiKey: <raw key>` | The key's tenant binding | Chrome extension, Excel add-in, server-to-server |
| **JWT in header** | `agentId: <id>` + `ssoToken: <jwt>` | JWT claims (`sub`, `companies`) | JS widget |

Both modes share the same agent-level controls:

- Agent must be **ACTIVE**.
- Customer must be **ACTIVE**.
- Tenant plan must include the relevant TableAI features (base + per-mode).
- `Origin` header must match the agent's allow-list (when configured).

## API Key

The simplest mode. Issued per agent, scoped by Origin and/or IP allow-list.

### Request shape

```
POST /emp/1/api/tableai/chat HTTP/1.1
Host:    yourcompany.antrika.com
apiKey:  ak_live_4f8c2e7d9b1a6c3e5f8d2b9a4e7c1f6d
Origin:  https://your-app.example.com
Content-Type: application/json

{ "data": { "sessionId": "…", "query": "…", "stream": true } }
```

### Failure responses

| Code | Body | Cause |
|---|---|---|
| 401 | `{ ed: "Missing API key" }` | No `apiKey` header. |
| 401 | `{ ed: "Invalid API key" }` | Key not found. |
| 403 | `{ ed: "Not authorised" }` | Origin/IP not in allow-list. |
| 403 | `{ ed: "Agent is not active" }` | Agent status ≠ ACTIVE. |
| 403 | `{ ed: "Customer is not active" }` | Tenant suspended. |
| 403 | `{ ed: "This feature is not available…" }` | Plan missing the required TableAI feature. |

### Origin allow-list

- **Empty list + empty IP** → any Origin is accepted (avoid in production).
- **List populated** → exact match required, `*` glob supported. Examples:
  - `https://app.example.com` — exact
  - `https://*.example.com` — any subdomain
  - `https://*.staging.*.example.com` — multi-glob
- **Server IP set** → server-side calls from that IP are accepted regardless of Origin.

## JWT in `ssoToken` header

Used by the JS widget. The host server mints a short-lived JWT per visitor; the widget passes it on every request. No third-party cookies, no `syncUser` round-trip.

### Request shape

```
POST /emp/1/api/tableai/widget/chat HTTP/1.1
Host:     yourcompany.antrika.com
agentId:  agt_xxx
ssoToken: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c3JfMTIzIiw…
Origin:   https://your-app.example.com
Content-Type: application/json

{ "sessionId": "…", "query": "…", "stream": true }
```

### Token contract

Algorithm: **HS256**.
Signing key: the agent's shared secret (admin → Agents → *Show secret*).

Required claims:

| Claim | Type | Description |
|---|---|---|
| `sub` | string | Stable user ID in your system. |
| `name` | string | Human-readable user name (display). |
| `email` | string | User email. |
| `companies` | array | At least one `{ id, name }` — maps to backend Account. |
| `iat` | number | Issued-at unix seconds. |
| `exp` | number | Expiry unix seconds. **≤ 1 hour recommended.** |

Optional:

| Claim | Description |
|---|---|
| `customFields` | Object of tenant-defined user attributes (mapped to backend user fields by name). |
| `companies[].customFields` | Same, per-company. |

The backend validates the signature, parses the claims, and creates or fetches the matching Account and User records — so subsequent sessions can be filtered by user/account in admin.

### Why JWT not cookies

- Cross-origin embeds work (no third-party cookie).
- File uploads (multipart) don't fight CORS preflight.
- Host can rotate per-user tokens without server-side state.
- Plays well with edge / serverless deployments.

### Trust boundary

The agent's shared secret is **server-only**. Never embed it in the widget bundle or HTML. Mint the JWT on your server and ship just the token to the page.

Examples:

- [Node.js](../examples/jwt-mint-node.md)
- [Java](../examples/jwt-mint-java.md)

## Picking a mode

```
Server with no UI?
  └─ API Key

Browser surface, embedded in YOUR product?
  └─ JWT in header (JS widget)

Browser surface, embedded in OTHER people's products?
  └─ Chrome extension (which uses API Key under the hood)
```

## Quick comparison

| | API Key | JWT in header |
|---|---|---|
| Per-user history | No (key is service-wide) | Yes |
| Host setup cost | Low (paste a key) | Low (mint a JWT) |
| Cross-origin friendly | Yes | Yes |
| File upload friendly | Yes | Yes |
| Rotation | Recreate the key | Per-token expiry |
| Audit who asked what | Approximately (via session metadata) | Yes |
