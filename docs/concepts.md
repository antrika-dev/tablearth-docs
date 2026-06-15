# Concepts

A five-minute mental model. Read this before any of the integration docs.

> **Naming:** the product is **tableArth.ai**. The HTTP API paths keep the
> original `tableai` slug (e.g. `/emp/1/api/tableai/init`) — that's the wire
> contract, not the brand. You'll see both: `tableArth.ai` in prose, `tableai`
> in URLs.

## The four nouns

### Agent

An **AI Agent** of type *data query* (a "tableArth.ai agent") is the unit of
configuration: the LLM model, prompt, tool list, allowed Origins, and
credentials. An agent is created once by a tenant admin (see
[agent setup](admin/agent-setup.md)). Every integration talks *to a specific
agent* — its `agentId` is the public identifier you put on the host page or in
the API call. The JS widget calls the same value `widgetId` / `widget_id`.

A tenant can have many agents (e.g. one per product surface, one per department).

### Session

A **session** holds the chat history and the materialised dataset for one
conversation. The dataset is stored as a parquet file derived from the CSV the
client uploaded. Once a session has a file, asking new questions does *not*
re-upload anything — the agent re-runs queries against the cached parquet.

Sessions are owned by a tenant (`customerId`). Optionally they can be tagged
with an Account (`accountId`) and User (`userId`) so admins can filter
conversations. Each session also records a **source** — `API`, `Widget`,
`Extension`, or `Admin` — so admins can see where a conversation came from.

### `remoteSessionId` / `remoteId`

The **session lookup key**. The client (widget / extension / your code)
supplies it; the backend uses it to find an existing session or create a new one.

```
remoteSessionId = '<optional prefix>' + <stable identifier or content hash>
```

Two ways to derive it, in order of preference:

1. **Stable business identity** — when the host knows what the report *is*.
   Example: `report:attendance:2026-05-08:customerX`. Best for live tables that
   paginate or sort, because the identity stays put even when row order changes.
2. **Hash of the data** — when the host only knows the *contents*. Example:
   `sha256(csv).slice(0,32)`. Same contents → same session. One row added → new
   session, fresh conversation.

Conventions in the wild:

| Surface | What it sends |
|---|---|
| Chrome extension | `tai-ext-` + `sha256(csv).slice(0,32)` |
| Server-to-server | `tai-srv-` + your hash or stable id (your convention) |
| JS widget | `sha256(csv).slice(0,48)` — **no prefix** — or the `remoteSessionId` you pass |

The prefix is just a soft namespace; the backend doesn't enforce it. Pick one
if you build a new client.

### File

The CSV (uploaded as multipart) becomes a parquet on the server.
`sessionData.fileAvailable === true` means "you can chat without uploading
again." `false` means "upload first."

## The lifecycle

```
client                                            backend
  │                                                  │
  │── init(remoteSessionId) ──────────────────────►  │
  │                                                  │  (find or create session)
  │  ◄──────── { sessionData, data: [history] }      │
  │                                                  │
  │  if !fileAvailable:                              │
  │── upload(sessionId, csv) ─────────────────────►  │
  │  ◄──────── { tableFileId, dashboardSpec }        │  (parse → parquet)
  │                                                  │
  │── chat(sessionId, query, stream:true) ────────►  │
  │  ◄──── SSE: STATUS TEXT* CHART? DONE             │
  │                                                  │
  │  optional dashboard:                             │
  │── dashboard/spec(sessionId) ──────────────────►  │
  │  ◄──────── { ready, dashboardSpec: "<json>" }    │
  │── dashboard/widget/data(widgetId, offset, limit)►│
  │  ◄──────── { data, total, hm }                   │
```

### Resuming

Re-opening with the same `remoteSessionId` returns the same `sessionId` and
`fileAvailable=true` (assuming the file is still there) — no re-upload. The chat
history rendered on screen comes from the `data` array `init` returns.

### Invalidating

Mint a new `remoteSessionId` whenever the underlying data changes in a way that
should reset the conversation. Hashing the CSV does this for free. Stable IDs
need to be versioned by you (e.g. include a date or revision suffix).

## Auth at a glance

Two ways in, each suited to a different surface. Full detail in
[api/auth.md](api/auth.md).

| Mode | How | Surface |
|---|---|---|
| **API key** in the `apiKey` header | Tenant-issued key + Origin/IP allow-list | Chrome extension, server-to-server |
| **Widget SSO** | Host establishes an SSO session (`syncUser` → `antrikaSSO` cookie); the widget signs each request with `widget_id` + a per-widget token | JS widget |

The widget does **not** take a host-minted token as a render argument. The host
authenticates the visitor once via `window.antrika('syncUser', …)`, which mints
the SSO session the widget then rides. See [widget.md](integrations/widget.md#auth).

> API clients that need per-user identity (not just the service-wide key) can
> additionally pass a host-minted `ssoToken` JWT on the `/v2` API routes — see
> [auth.md](api/auth.md#per-user-identity-on-api-routes-v2).

## What the agent emits during a chat

The chat stream is Server-Sent Events. Each `data:` line is a JSON object with a
`contentType`:

- `STATUS` — a short "working on it" status line, usually emitted first.
- `TEXT` — token-stream of the answer.
- `CHART` — a chart payload (`chartConfig` + `chartData`) to render inline.
- `ERROR` — terminal; message in `content`.
- `DONE` — terminal; signals the answer is complete.

A few other `contentType`s exist (`SESSION`, `ACTION_CONFIRMATION`,
`ACTION_INPUT`). Ignore any you don't handle. Full framing in
[api/sse-protocol.md](api/sse-protocol.md).

## What the auto dashboard contains

The first upload triggers a background job that builds a dashboard spec:

- **KPI cards** — single-number summaries (`count`, `avg`, `max`, etc.).
- **Charts** — bars / lines / pies pulled from the data.
- **Table** — a paginated view of the source data (rows fetched per page from
  `dashboard/widget/data`).

`dashboard/spec` returns `{ ready: false }` until the job finishes. The spec
itself comes back as a **JSON string** in `dashboardSpec` — parse it client-side.
Show a placeholder; poll or re-fetch on tab switch.

## Common gotchas

- **`remoteSessionId` collisions** — two unrelated tables that happen to hash to
  the same value would share history. Including a tenant or report-type prefix in
  the input avoids this.
- **CSV size** — keep individual uploads under your tenant's plan limit. Larger
  sets need to be sliced upstream.
- **Origin must be exact** — `https://app.example.com` and `https://example.com`
  are different origins. The agent's allow-list supports `*` globs (e.g.
  `https://*.example.com`).
- **Session belongs to a customer** — you can't move a session between tenants.
  If you re-key your tenants, you re-init.

Next: pick an integration surface from the [README](../README.md#pick-your-path).
