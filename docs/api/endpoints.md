# Endpoint Reference

Every public tableArth.ai route, grouped by auth mode. All paths are relative to
the tenant base URL (e.g. `https://yourcompany.tableArth.ai`). The brand is
tableArth.ai; the path slug stays `tableai`.

## Quick index

| Path | Method | Auth | Used by |
|---|---|---|---|
| `/emp/1/api/tableai/extension/validate` | POST | API key | Chrome extension |
| `/emp/1/api/tableai/extension/domains` | GET | API key | Chrome extension |
| `/emp/1/api/tableai/init` | GET | API key | Chrome extension, server-to-server |
| `/emp/1/api/tableai/upload` | OPTIONS / POST | API key | same |
| `/emp/1/api/tableai/chat` | POST | API key | same |
| `/emp/1/api/tableai/init/v2` | GET | API key + `ssoToken` | server-to-server (per-user) |
| `/emp/1/api/tableai/upload/v2` | OPTIONS / POST | API key + `ssoToken` | same |
| `/emp/1/api/tableai/chat/v2` | POST | API key + `ssoToken` | same |
| `/emp/1/api/tableai/widget/init` | GET | Widget SSO (`widget_id` + cookie) | JS widget |
| `/emp/1/api/tableai/widget/upload` | OPTIONS / POST | Widget SSO | JS widget |
| `/emp/1/api/tableai/widget/chat` | POST | Widget SSO | JS widget |
| `/emp/1/api/tableai/widget/dashboard/spec` | GET | Widget SSO | JS widget |
| `/emp/1/api/tableai/widget/dashboard/widget/data` | POST | Widget SSO | JS widget |
| `/emp/1/api/dashboard/spec` | POST | API key | Chrome extension, server-to-server |
| `/emp/1/api/dashboard/widget/data` | POST | API key | Chrome extension, server-to-server |

## API-key endpoints

Used by the Chrome extension and any direct REST client.

### `POST /emp/1/api/tableai/extension/validate`

Validate an activation key before unlocking UI. (Replaces the old
`apikey/validate` route.)

**Request**

```
apiKey:  <your key>
Origin:  <your origin>
Content-Type: application/json

{}
```

**Success** — `200 { s: true }` (plus standard envelope fields).
**Failure** — see [auth.md](auth.md#failure-responses). Rate-limited per IP (`429`).

### `GET /emp/1/api/tableai/extension/domains`

Returns the backend-managed Origin allow-list for this key, so a client can show
the user where it will activate.

**Response**

```jsonc
{
  "s": true,
  "allowAll": false,                       // true → key works on any origin
  "domains": [ "https://*.example.com" ]   // patterns; `*` is a glob
}
```

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
    "title":         "Attendance Yesterday",
    "source":        "Extension",          // API | Widget | Extension | Admin
    "readOnly":      true                  // true for any non-admin source
  },
  "data": [ /* past messages, newest-first */
    {
      "id":          "msg_…",
      "message":     "Who worked the most?",
      "messageType": "received",           // received = user, sent = assistant
      "chartData":   null,                 // present only on chart answers
      "chartConfig": null
    }
  ],
  "hm": false   // hasMore — whether older messages exist
}
```

**Notes**

- If no session exists for the `remoteSessionId`, one is created and returned with
  `fileAvailable=false`.
- Messages come back **newest-first**. Reverse if you display chronologically.
- The answer text is in **`message`** (there is no `answer` field on history
  items). `messageType: "sent"` = assistant, `"received"` = user.

### `OPTIONS /emp/1/api/tableai/upload`

CORS preflight. Echoes Origin, allows POST + multipart headers. Always call before
`POST` from a browser.

The preflight requires the `apiKey` **as a query parameter** (browsers can't send
custom headers on preflight). The `POST` itself can use either the header or the
query parameter.

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
  "s":               true,
  "tableFileId":     "fil_xyz",
  "remoteSessionId": "tai-ext-…",
  "dashboardSpec":   "<json string, or null while the dashboard job runs>"
}
```

After this, `init` for the same `remoteSessionId` returns `fileAvailable=true`.

**Notes**

- Plan-bound size limit.
- Re-uploading into a session that already has a file is rejected. Mint a new
  `remoteSessionId` to start fresh.

### `POST /emp/1/api/tableai/chat`

Ask a question. Streaming or non-streaming.

**Request**

```
apiKey:  <your key>
Origin:  <your origin>
Content-Type: application/json
Accept:       text/event-stream, application/json     // recommended for stream

{ "sessionId": "ses_abc123", "query": "Who worked the most?", "stream": true }
```

> The API-key body is **flat** — not wrapped in `data`. (The widget route wraps
> it; see below.)

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
The session id is echoed on the `X-Session-Id` response header.

**Failure**

| Body | Cause |
|---|---|
| `{ ed: "Missing Session" }` | No `sessionId` in the body. |
| `{ ed: "Invalid Session" }` | Session not found for the tenant. |
| `{ ed: "File not available" }` | Session exists but no upload happened. |

### `/v2` routes (per-user identity)

`/emp/1/api/tableai/{init,upload,chat}/v2` mirror the routes above but also accept
a host-minted `ssoToken` **JWT header** for per-user attribution. `init/v2`
requires `ssoToken`. Gated by the single-sign-on plan feature. See
[auth.md](auth.md#per-user-identity-on-api-routes-v2).

## Widget endpoints

Used by the JS widget. Auth: `widget_id` header/param + the `antrikaSSO` cookie +
per-widget request signing (handled inside the widget bundle). Full auth detail in
[auth.md](auth.md#widget-sso).

### `GET /emp/1/api/tableai/widget/init`

Same response shape as the API-key `init`. Identity (account + user) comes from the
SSO session, so it does **not** read `customerId` / `userId` headers.

### `OPTIONS / POST /emp/1/api/tableai/widget/upload`

Same as API-key `upload`; auth swapped to Widget SSO.

### `POST /emp/1/api/tableai/widget/chat`

Same semantics as API-key `chat`, but the body is **wrapped**:

```jsonc
{ "data": { "sessionId": "ses_abc123", "query": "…", "stream": true } }
```

Source recorded as `Widget` in admin.

### Widget dashboard

The widget has its **own** dashboard routes (distinct from `/api/dashboard/*`):

- `GET  /emp/1/api/tableai/widget/dashboard/spec` — `sessionId` via header.
- `POST /emp/1/api/tableai/widget/dashboard/widget/data` — body wrapped in `data`.

Response shapes match the [dashboard endpoints](#dashboard-endpoints) below.

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
  "ready": true,                 // false while the background job is still running
  "dashboardSpec": "<json string — parse client-side>"
}
```

The spec, once parsed, has roughly:

```jsonc
{
  "kpis":   [ { id, title, value, format, … } ],
  "charts": [ { id, title, type: "bar|line|pie", … } ],
  "tables": [ { id, title, columns: [ … ] } ]
}
```

`ready=false` is a normal initial state. Show a placeholder; refetch on tab switch
or on a short interval.

### `POST /emp/1/api/dashboard/widget/data`

Fetch the data for a single widget in the dashboard spec. Charts/KPIs typically
return everything in one call; tables paginate.

**Request**

```
apiKey: <your key>
Origin: <your origin>
Content-Type: application/json

{
  "sessionId": "ses_abc123",
  "widgetId":  "wgt_table_main",
  "offset":    0,        // tables only
  "limit":     50        // tables only (default 100)
}
```

**Response**

```jsonc
{
  "s":      true,
  "data":   "[ … ]",      // rows, often a JSON-encoded STRING — detect and parse
  "total":  1234,         // tables only — total row count for pagination
  "hm":     true,         // tables only — hasMore
  "capped": false         // true when the result set was truncated
}
```

> `data` is frequently a JSON string. Clients should detect-and-parse:
> ```js
> if (typeof body.data === 'string') body.data = JSON.parse(body.data);
> ```

## Pagination headers

`/init` and `/widget/init` accept generic pagination headers:

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
