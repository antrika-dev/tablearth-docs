# Endpoint Reference

Every public TableAI route, grouped by auth mode. All paths are relative to the tenant base URL (e.g. `https://yourcompany.antrika.com`).

## Quick index

| Path | Method | Auth | Used by |
|---|---|---|---|
| `/emp/1/api/tableai/apikey/validate` | POST | API key | Chrome extension, Excel add-in |
| `/emp/1/api/tableai/init` | GET | API key | Chrome extension, Excel add-in, server-to-server |
| `/emp/1/api/tableai/upload` | OPTIONS / POST | API key | same |
| `/emp/1/api/tableai/chat` | POST | API key | same |
| `/emp/1/api/tableai/widget/init` | GET | JWT (`agentId` + `ssoToken`) | JS widget |
| `/emp/1/api/tableai/widget/upload` | OPTIONS / POST | JWT (`agentId` + `ssoToken`) | JS widget |
| `/emp/1/api/tableai/widget/chat` | POST | JWT (`agentId` + `ssoToken`) | JS widget |
| `/emp/1/api/dashboard/spec` | POST | API key | Chrome extension, Excel add-in |
| `/emp/1/api/dashboard/widget/data` | POST | API key | Chrome extension, Excel add-in |

## API-key endpoints

Used by the Chrome extension, Excel add-in, and any direct REST client.

### `POST /emp/1/api/tableai/apikey/validate`

Lightweight ping for clients to validate their key before unlocking UI.

**Request**

```
apiKey:  <your key>
Origin:  <your origin>
Content-Type: application/json

{}
```

**Success** — `200 { s: true }` (plus standard envelope fields).

**Failure** — see [auth.md](auth.md#failure-responses).

### `GET /emp/1/api/tableai/init`

Open or resume a session by `remoteSessionId`.

**Request headers**

| Header | Required | Description |
|---|---|---|
| `apiKey` | yes | The activation key. |
| `Origin` | yes | Browser sets it; servers must set explicitly. |
| `remoteSessionId` | yes | Stable session key. See [concepts.md](../concepts.md#remoteid--remotesessionid). |
| `pt` | yes | Page total — number of past messages to load. |
| `pn` | yes | Page number. `0` for the first page. |
| `customerId` | optional | Tag the session with an Account ID. |
| `userId` | optional | Tag the session with a User ID. |
| `webpage` | optional | Source page URL — shows up in admin audit. |
| `tableName` | optional | Table label — shows up in admin audit. |

**Response**

```jsonc
{
  "s": true,
  "sessionData": {
    "sessionId":     "ses_abc123",
    "fileAvailable": true,
    "title":         "Attendance Yesterday"
  },
  "data": [ /* past messages, newest-first */
    {
      "id":          "msg_…",
      "messageType": "received",          // received = user, sent = assistant
      "message":     "Who worked the most?",
      "answer":      null,
      "chartData":   null,
      "chartConfig": null
    }
  ],
  "hm": false   // hasMore — whether older messages exist
}
```

**Notes**

- If no session exists for the `remoteSessionId`, one is created and returned with `fileAvailable=false`.
- Messages come back **newest-first**. Reverse if you display chronologically.

### `OPTIONS /emp/1/api/tableai/upload`

CORS preflight. Echoes Origin, allows POST + multipart headers. Always call before `POST` from a browser.

The preflight requires the `apiKey` to be present **as a query parameter** (browsers can't send custom headers on preflight). The `POST` itself can use either the header or the query parameter.

### `POST /emp/1/api/tableai/upload`

Upload the CSV into an existing session (multipart).

**Request**

```
apiKey:    <your key>
Origin:    <your origin>
sessionId: <from init response>

multipart/form-data:
  file: <csv blob; name=table.csv content-type=text/csv>
```

**Response**

```jsonc
{
  "s":        true,
  "fileId":   "fil_xyz",
  "rowCount": 42,
  "columns":  ["Employee", "Hours", …]
}
```

After this, `init` for the same `remoteSessionId` returns `fileAvailable=true`.

**Notes**

- Plan-bound size limit (default 25 MB).
- Re-uploading into a session that already has a file returns `{ ed: "File already uploaded" }`. Mint a new `remoteSessionId` to start fresh.

### `POST /emp/1/api/tableai/chat`

Ask a question. Streaming or non-streaming.

**Request**

```
apiKey:  <your key>
Origin:  <your origin>
Content-Type: application/json
Accept:       text/event-stream, application/json     // optional but recommended for stream

{ "data": { "sessionId": "ses_abc123", "query": "Who worked the most?", "stream": true } }
```

> The body must be wrapped in a top-level `data` field as shown.

**Response, `stream:false`**

```jsonc
{
  "s": true,
  "answer":      "Bob worked the most (9 hours).",
  "chartData":   null,
  "chartConfig": null
}
```

**Response, `stream:true`** — Server-Sent Events. See [sse-protocol.md](sse-protocol.md).

**Failure**

| Body | Cause |
|---|---|
| `{ ed: "Missing Session" }` | No `sessionId` in the body. |
| `{ ed: "Invalid Session" }` | Session not found for the tenant. |
| `{ ed: "File not available" }` | Session exists but no upload happened. |

## Widget endpoints

Used by the JS widget. Auth: `agentId` header + `ssoToken` JWT + Origin allow-list. Full auth detail in [auth.md](auth.md#jwt-in-ssotoken-header).

### `GET /emp/1/api/tableai/widget/init`

Same shape as the API-key `init` above, with these header changes:

| Header | Description |
|---|---|
| `agentId` | The agent ID (replaces `apiKey`). |
| `ssoToken` | JWT identifying the visitor (account + user from claims). |

Origin: comes from the browser; agent's allow-list applies.

### `OPTIONS / POST /emp/1/api/tableai/widget/upload`

Same as API-key `upload`; auth swapped (`agentId` + `ssoToken` instead of `apiKey`).

### `POST /emp/1/api/tableai/widget/chat`

Same body shape as API-key `chat`. Source recorded as `WIDGET` in admin (vs `API` for direct REST).

## Dashboard endpoints

Auth: `apiKey`.

### `POST /emp/1/api/dashboard/spec`

Returns the auto-generated dashboard spec for a session.

**Request**

```
apiKey: <your key>
Origin: <your origin>
Content-Type: application/json

{ "sessionId": "ses_abc123" }
```

**Response**

```jsonc
{
  "s":     true,
  "ready": true,             // false while the background job is still running
  "dashboardSpec": "<json string — parse client-side>"
}
```

The spec, once parsed, has roughly:

```jsonc
{
  "kpis":   [ { id, title, value, format, … } ],
  "charts": [ { id, title, type: "bar|line|pie", x, y, … } ],
  "tables": [ { id, title, columns: [ … ] } ]
}
```

`ready=false` is a normal initial state. Show a placeholder; refetch on tab switch or on a short interval.

### `POST /emp/1/api/dashboard/widget/data`

Fetch the data for a single widget in the dashboard spec. Charts/KPIs typically return everything in one call; tables paginate.

**Request**

```
apiKey: <your key>
Origin: <your origin>
Content-Type: application/json

{
  "sessionId": "ses_abc123",
  "widgetId":  "wgt_table_main",
  "offset":    0,        // tables only
  "limit":     50        // tables only
}
```

**Response**

```jsonc
{
  "s":     true,
  "data":  [ /* array of row objects, OR a JSON-encoded string of the same */ ],
  "total": 1234,         // tables only — total row count for pagination
  "hm":    true          // tables only — hasMore
}
```

> Some legacy responses encode `data` as a JSON string. Clients should detect-and-parse:
> ```js
> if (typeof body.data === 'string') body.data = JSON.parse(body.data);
> ```

## Pagination headers

Several endpoints (`/init`, `/widget/init`) accept generic pagination headers:

| Header | Description |
|---|---|
| `pt` | Page total (page size). |
| `pn` | Page number, zero-based. |

## Common envelope fields

Every response includes the standard fields:

| Field | Meaning |
|---|---|
| `s` | `true` on success, `false` or absent on failure. |
| `ed` | End-user error description. Show to users. |
| `msg` | Internal/debug message. Log; don't show. |
| `hm` | "Has more" — used wherever pagination applies. |
