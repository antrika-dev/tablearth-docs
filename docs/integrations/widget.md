# JS Widget Integration

Embed tableArth.ai next to any table in your web product. The user clicks an AI
button → a chat dialog opens against the report's data.

## TL;DR — minimum integration

```html
<!-- 1. load the bundle -->
<link  href="https://<your-widget-host>/app.<hash>.css" rel="stylesheet">
<script src="https://<your-widget-host>/app.<hash>.js"></script>

<!-- 2. one div per table -->
<div class="antrika-table-ai-widget"
     data-id="WIDGET_ID"
     data-report-name="Attendance Yesterday"></div>

<!-- 3. authenticate the visitor, then mount -->
<script>
  // (a) establish the SSO session for the signed-in visitor
  window.antrika('syncUser', {
    customerId: 'YOUR_TENANT_ID',
    token:      'eyJhbGciOiJIUzI1Ni…'   // JWT your server mints (see "Auth")
  }, function () {
    // (b) mount the widget once the session exists
    window.antrika('renderTableAI', {
      widgetId:        'WIDGET_ID',
      remoteSessionId: 'report:attendance:2026-05-08',
      reportName:      'Attendance Yesterday'
    });
  });
</script>
```

Three things: the bundle, a div, and a `syncUser` → `renderTableAI` pair.

## What the host provides

### On the mount `<div>`

| Attribute | Required? | What it's for |
|---|---|---|
| `class="antrika-table-ai-widget"` | yes | Marks the element the widget mounts into. |
| `data-id` | yes | The agent / widget ID. Must equal the `widgetId` you pass to `renderTableAI`. |
| `data-report-name` | optional | Dialog title. Falls back to the `reportName` render argument. |

### To `renderTableAI(options)`

| Field | Required? | What it's used for |
|---|---|---|
| `widgetId` | yes | Tells the backend which agent + tenant. Matched against each div's `data-id`. |
| `remoteSessionId` | recommended | Stable session key — same value resumes the chat. See [concepts](../concepts.md#remoteid--remotesessionid). |
| `csvFile` | optional | Raw CSV string. Used to derive the session key when `remoteSessionId` is absent. |
| `reportName` | optional | Dialog title (or set `data-report-name`). |
| `theme` | optional | `'LIGHT'` (default) or `'DARK'`. |

> There is **no `ssoToken` argument** to `renderTableAI`. The visitor's identity
> comes from the SSO session established by `syncUser` (below), not from a token
> you hand the render call.

Advanced options (`reportContext`, `chatContext`, `sessionId`, `selectedModule`,
`btnStyle`, `tableRef`, `reportTableJson`) exist for tighter host integrations;
most embeds don't need them.

If both `remoteSessionId` and `csvFile` are omitted, chat resume after navigation
is lost — pass at least one.

## Auth

The widget authenticates in two steps. Your server mints a short-lived JWT; the
page hands it to `syncUser`; the backend turns it into an SSO session (a cookie)
that the widget rides on every request.

```
your server                browser (page)                 tableArth.ai backend
     │                           │                                 │
  mint JWT  ──── token ────►  syncUser(customerId, token) ───────► verify JWT
                               │                          ◄─ Set-Cookie: antrikaSSO
                               │                                 │
                          renderTableAI(...)                     │
                               │   widget signs each request     │
                               │   (widget_id + per-widget token │
                               │    + antrikaSSO cookie) ───────► init / upload / chat
```

The per-request signing is internal to the bundle — **you don't compute it.**
Your only job is to (1) mint the JWT and (2) call `syncUser`.

### Step 1 — mint the SSO token (server-side)

The token is an **HS256 JWT** signed with your tenant's **SSO signing key**
(admin → SSO settings). Never embed that key in the page.

Required claims:

```json
{
  "id":    "user-id-in-your-system",
  "name":  "Full Name",
  "email": "user@example.com",
  "companies": [
    { "id": "your-account-id", "name": "Acme Corp" }
  ],
  "iat":   1746816000,
  "exp":   1746819600
}
```

| Claim | Required | Notes |
|---|---|---|
| `id` | yes | Stable user ID in your system (the user identifier — **not** `sub`). |
| `name` | yes | Display name. |
| `email` | yes | User email. |
| `companies` | yes | ≥ 1 `{ id, name }` — maps to a backend Account. |
| `iat` / `exp` | yes | Issued-at / expiry, unix seconds. Keep TTL short (≤ 1 hour). |
| `thumb` | optional | Avatar URL. |
| `timezone` | optional | IANA tz, e.g. `Asia/Kolkata`. |
| `tags` | optional | Array of strings (user licenses / labels). |
| `customFields` | optional | `{ key: value }`; also valid per-company as `companies[].customFields`. |

Examples: [Node.js](../examples/jwt-mint-node.md) · [Java](../examples/jwt-mint-java.md).

### Step 2 — establish the session

```js
window.antrika('syncUser', {
  customerId: 'YOUR_TENANT_ID',   // your tenant / customer id (from admin)
  token:      mintedJwt            // the JWT from step 1
}, onReady);
```

`syncUser` posts to the production host; use **`syncUserLocal`** for local/dev
tenants. On success the backend creates/fetches the User + Account from the
claims and sets the `antrikaSSO` cookie. Mount the widget in the callback so the
cookie exists before the first request.

### Refreshing

The SSO session lives in the `antrikaSSO` cookie. To refresh before the visitor's
token expires, mint a fresh JWT and call `syncUser` again — the cookie is
re-issued. There is no separate `setSsoToken` call.

## How `remoteSessionId` works

Read [concepts.md](../concepts.md#remoteid--remotesessionid) for the full picture.
For the widget:

- Pass a **stable identity** when you have one:
  - `report:attendance:2026-05-08:cust-123`
  - `dashboard:sales-pipeline:weekly:cust-123`
- Re-key (mint a new value) when the dataset changes meaningfully (new time
  window, changed filters, refreshed data).
- If you only have the CSV, omit `remoteSessionId` and pass `csvFile` — the widget
  hashes it:

```
remoteSessionId = sha256(csvFile).slice(0, 48)   // 48 hex chars, no prefix
```

## Lifecycle inside the widget

```
mount  ─► render AI button (no network yet)
click  ─► open dialog
        ─► widget/init   (widget_id + signed request + antrikaSSO cookie)
            ├ fileAvailable=true  → render past messages, wait for user
            └ fileAvailable=false → widget/upload (multipart csv) → continue
type a query, press Send
        ─► widget/chat   (body { data: { sessionId, query, stream:true } })
            stream chunks: STATUS TEXT* CHART? DONE
optional dashboard tab
        ─► widget/dashboard/spec
        ─► widget/dashboard/widget/data
```

The widget uses its **own** dashboard routes (`/widget/dashboard/*`), not the
API-key `/api/dashboard/*` ones. Endpoint reference:
[api/endpoints.md](../api/endpoints.md#widget-endpoints).

> The widget `chat` body is wrapped in a top-level `data` field
> (`{ "data": { … } }`). The API-key `chat` route is **not** — see
> [endpoints.md](../api/endpoints.md#post-emp1apitableaichat).

## Multiple tables on one page

Repeat the div per table. The widget mounts independently for each `data-id`
match:

```html
<div class="antrika-table-ai-widget" data-id="WIDGET_ID"
     data-report-name="Attendance"></div>

<!-- ...other markup, another table... -->

<div class="antrika-table-ai-widget" data-id="WIDGET_ID"
     data-report-name="Leaves"></div>

<script>
  // pass a distinct remoteSessionId per table via your own mount logic,
  // or let each derive its key from the csvFile you hand it.
  window.antrika('renderTableAI', { widgetId: 'WIDGET_ID' });
</script>
```

`renderTableAI` is idempotent — it skips divs already marked
`data-antrika-mounted="1"`.

## Theming

Pick `LIGHT` or `DARK` via the `theme` argument. To override colours, the widget
reads CSS variables it sets on `:root`; the most useful:

```css
:root {
  --widget-body-primary:      #036DCE;   /* primary colour: icons / send button */
  --widget-body-primary-dark: #00218D;
  --widget-text-primary:      #1a1a1a;
}
```

## Branding

A "Powered by" footer shows by default. Tenants on a plan that includes the
`hidePoweredBy` feature can disable it (admin setting `hidePoweredByAntrika`),
which also removes the watermark from PDF exports.

## Gotchas

- **No SSO session, no data.** If `syncUser` hasn't run (or the `antrikaSSO`
  cookie is missing/expired), `init`/`chat` fail. Mount in the `syncUser` callback.
- **Origin must be in the agent's allow-list.** A request from
  `https://app.example.com` to an agent that only allows `https://example.com`
  returns `403`. Globs work: `https://*.example.com`.
- **Sign the JWT with the SSO key, not the agent API key.** The two are different
  credentials.
- **`widget_id` must be set.** Requests without it skip signing and fail auth —
  always pass `widgetId` (and the matching `data-id`).
- **CSV upload is multipart**, so it triggers a CORS preflight. The widget
  endpoint handles `OPTIONS` automatically — no host config needed.

## Worked example

```html
<!doctype html>
<html>
<head>
  <link href="https://<your-widget-host>/app.css" rel="stylesheet">
</head>
<body>
  <h1>Daily Reports</h1>

  <section>
    <h2>Attendance — Yesterday</h2>
    <table id="att-yest">…</table>
    <div class="antrika-table-ai-widget"
         data-id="agt_xxx"
         data-report-name="Attendance Yesterday"></div>
  </section>

  <script src="https://<your-widget-host>/app.js"></script>
  <script>
    // your endpoint returns a JWT minted with the tenant SSO key (see Auth)
    fetch('/me/tableai-token', { credentials: 'include' })
      .then(r => r.json())
      .then(({ token, customerId }) => {
        window.antrika('syncUser', { customerId, token }, () => {
          window.antrika('renderTableAI', {
            widgetId:        'agt_xxx',
            remoteSessionId: 'report:attendance:2026-05-08:cust-123'
          });
        });
      });
  </script>
</body>
</html>
```

The `/me/tableai-token` endpoint is yours — it returns a JWT minted as in the
[auth section](#step-1--mint-the-sso-token-server-side).
