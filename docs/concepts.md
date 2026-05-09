# Concepts

A five-minute mental model. Read this before any of the integration docs.

## The four nouns

### Agent

An **AI Agent** of type *TableAI* is the unit of configuration: the LLM model, prompt, tool list, allowed Origins, optional API key. An agent is created once by a tenant admin (see [agent setup](admin/agent-setup.md)). Every integration of TableAI talks *to a specific agent* — its `agentId` is the public identifier you put on the host page or in the API call.

A tenant can have many agents (e.g. one per product surface, one per department).

### Session

A **session** holds the chat history and the materialised dataset for one conversation. The dataset is stored as a parquet file derived from the CSV the client uploaded. Once a session has a file, asking new questions does *not* re-upload anything — the agent re-runs queries against the cached parquet.

Sessions are owned by a tenant (`customerId`). Optionally they can be tagged with an Account (`accountId`) and User (`userId`) so admins can filter conversations.

### `remoteId` / `remoteSessionId`

The **session lookup key**. The client (widget / extension / your code) supplies it; the backend uses it to find an existing session or create a new one.

```
remoteSessionId = '<prefix>-' + <stable identifier>
```

Two ways to derive it, in order of preference:

1. **Stable business identity** — when the host knows what the report *is*. Example: `report:attendance:2026-05-08:customerX`. Best for live tables that paginate or sort, because the identity stays put even when row order changes.
2. **Hash of the data** — when the host only knows the *contents*. Example: `sha256(csv).slice(0,32)`. Same contents → same session. One row added → new session, fresh conversation.

Conventions in the wild:

| Surface | Prefix |
|---|---|
| Chrome extension | `tai-ext-` |
| Excel add-in | `tai-xl-` |
| JS widget | `tai-w-` |

The prefix is just a soft namespace; the backend doesn't enforce it. Pick one if you build a new client.

### File

The CSV (uploaded as multipart) becomes a parquet on the server. `sessionData.fileAvailable === true` means "you can chat without uploading again." `false` means "upload first."

## The lifecycle

```
client                                            backend
  │                                                  │
  │── init(remoteSessionId) ──────────────────────►  │
  │                                                  │  (find or create session)
  │  ◄──────── { sessionId, fileAvailable, history } │
  │                                                  │
  │  if !fileAvailable:                              │
  │── upload(sessionId, csv) ─────────────────────►  │
  │  ◄──────── { ok }                                │  (parse → parquet)
  │                                                  │
  │── chat(sessionId, query, stream:true) ────────►  │
  │  ◄──── SSE: TEXT* CHART? ERROR? DONE             │
  │                                                  │
  │  optional dashboard:                             │
  │── dashboard/spec(sessionId) ──────────────────►  │
  │  ◄──────── { ready, spec: {kpis, charts, table} }│
  │── dashboard/widget/data(widgetId, offset, limit)►│
  │  ◄──────── { rows, total, hm }                   │
```

### Resuming

Re-opening with the same `remoteSessionId` returns the same `sessionId` and `fileAvailable=true` (assuming the file is still there) — no re-upload. The chat history rendered on screen comes from the `data` array `init` returns.

### Invalidating

Mint a new `remoteSessionId` whenever the underlying data changes in a way that should reset the conversation. Hashing the CSV does this for free. Stable IDs need to be versioned by you (e.g. include a date or revision suffix).

## Auth at a glance

Two authentication modes, each used for different integration surfaces. Full detail in [api/auth.md](api/auth.md).

| Mode | How | Surface |
|---|---|---|
| **API key** in `apiKey` header | Tenant-issued key + Origin allow-list | Chrome extension, Excel add-in, server-to-server |
| **JWT in `ssoToken` header** | Host server signs a short-lived JWT with the agent's secret | JS widget |

## What the agent emits during a chat

The chat stream is Server-Sent Events. Each `data:` line is a JSON object with a `contentType`:

- `TEXT` — token-stream of the answer.
- `CHART` — a chart payload (data + spec) to render inline.
- `ERROR` — terminal; message in `content`.
- `DONE` — terminal; signals the answer is complete.

Full framing in [api/sse-protocol.md](api/sse-protocol.md).

## What the auto dashboard contains

The first chat answer triggers a background job that builds a dashboard spec:

- **KPI cards** — single-number summaries (`count`, `avg`, `max`, etc.).
- **Charts** — bars / lines / pies pulled from the data.
- **Table** — a paginated view of the source data (rows fetched per page from `dashboard/widget/data`).

`dashboard/spec` returns `{ ready: false }` until the job finishes. Show a placeholder; poll or re-fetch on tab switch.

## Common gotchas

- **`remoteSessionId` collisions** — two unrelated tables that happen to hash to the same value would share history. Including a tenant or report-type prefix in the input avoids this.
- **CSV size** — keep individual uploads under your tenant's plan limit (default 25 MB). Larger sets need to be sliced upstream.
- **Origin must be exact** — `https://app.example.com` and `https://example.com` are different origins. The agent's allow-list supports `*` globs (e.g. `https://*.example.com`).
- **Session belongs to a customer** — you can't move a session between tenants. If you re-key your tenants, you re-init.

Next: pick an integration surface from the [README](../README.md#pick-your-path).
