# JS Widget Integration

Embed TableAI next to any table in your web product. The user clicks an AI button → a chat dialog opens against the report's data.

## TL;DR — minimum integration

```html
<!-- 1. one div per table -->
<div class="antrika-table-ai-widget"
     data-id="WIDGET_ID"
     data-remote-id="report:attendance:2026-05-08"
     data-report-name="Attendance Yesterday"></div>

<!-- 2. the bundle -->
<link  href="https://widget.antrika.com/app.<hash>.css" rel="stylesheet">
<script src="https://widget.antrika.com/app.<hash>.js"></script>

<!-- 3. boot the widget -->
<script>
  window.antrika('renderTableAI', {
    widgetId: 'WIDGET_ID',           // = the agent ID issued by your admin
    ssoToken: 'eyJhbGciOiJIUzI1Ni…', // JWT minted by your server (see "Auth")
    remoteId: 'report:attendance:2026-05-08'
  });
</script>
```

That's it for the host page. Three things: a div, the bundle, one JS call.

## What the host needs to provide

| Field | Required? | Where it comes from | What it's used for |
|---|---|---|---|
| `widgetId` | yes | Admin → Agents → copy the agent ID | Tells the backend which agent + tenant |
| `ssoToken` | yes | Your server mints a JWT signed with the agent's secret | Identifies the visitor (account / user) |
| `remoteId` | recommended | Your code | Stable session key — same value resumes the chat |
| `csv` | optional | Your code | Used to derive the session key when `remoteId` is absent |
| `reportName` | optional | Your code or `data-report-name` | Shows in the dialog title |
| `theme` | optional | `'LIGHT'` (default) or `'DARK'` | Widget appearance |

If both `remoteId` and `csv` are omitted the widget falls back to a hash of `widgetId + URL`. Functional but you'll lose chat resume after navigation.

## Auth — minting the SSO token

The widget itself never holds tenant credentials. Your server mints a short-lived JWT once per page load (or per session) using the agent's shared secret (visible in admin → Agents → *Show secret*).

Required claims:

```json
{
  "sub":   "user-id-in-your-system",
  "name":  "Full Name",
  "email": "user@example.com",
  "companies": [
    { "id": "your-account-id", "name": "Acme Corp" }
  ],
  "iat":   1746816000,
  "exp":   1746819600
}
```

- Algorithm: **HS256**.
- Recommended TTL: **≤ 1 hour**.
- Sign on your server. Never embed the agent secret in the page or in any JS bundle.

Examples:

- [Node.js](../examples/jwt-mint-node.md)
- [Java](../examples/jwt-mint-java.md)

The widget passes the JWT verbatim as the `ssoToken` header on every backend request. The backend validates it with the agent's secret and binds the resulting account/user to the session.

## How `remoteId` works

Read [concepts.md](../concepts.md#remoteid--remotesessionid) for the full picture. Quick rules for the widget:

- Pass a **stable identity** when you have one. Examples that work well:
  - `report:attendance:2026-05-08:cust-123`
  - `dashboard:sales-pipeline:weekly:cust-123`
  - `kpi-board:executive:cust-123:rev=42`
- Re-key (mint a new `remoteId`) when the dataset changes meaningfully:
  - New time window
  - User changed filters
  - Backing data refreshed
- If you only have the CSV in hand, omit `remoteId` and pass `csv` — the widget hashes it.

The widget converts whatever you give it to a `remoteSessionId` like:

```
remoteSessionId = 'tai-w-' + (remoteId ?? sha256(csv).slice(0, 32))
```

## Lifecycle inside the widget

```
mount  ─► render AI button (no network yet)
click  ─► open dialog
        ─► init   (sends widgetId, ssoToken, remoteSessionId)
            ├ fileAvailable=true  → render past messages, wait for user
            └ fileAvailable=false → upload (multipart csv) → continue
type a query, press Send
        ─► chat   (body { sessionId, query, stream:true })
            stream chunks: TEXT* CHART? DONE
optional dashboard tab
        ─► dashboard/spec
        ─► dashboard/widget/data
```

Endpoint reference: [api/endpoints.md](../api/endpoints.md).

## Multiple tables on one page

Repeat the div per table. The widget mounts independently for each match against `data-id`:

```html
<div class="antrika-table-ai-widget"
     data-id="WIDGET_ID"
     data-remote-id="report:attendance:2026-05-08"></div>

<!-- ...other markup, another table... -->

<div class="antrika-table-ai-widget"
     data-id="WIDGET_ID"
     data-remote-id="report:leaves:2026-05-08"></div>

<script>
  window.antrika('renderTableAI', { widgetId: 'WIDGET_ID', ssoToken: '…' });
</script>
```

The `renderTableAI` call is idempotent — it skips divs that have already been mounted.

## Theming

Pick `LIGHT` or `DARK` via the `theme` argument. To override colours fully, the widget reads a small set of CSS variables:

```css
:root {
  --widget-body-primary:      #036DCE;   /* primary blue used for icons / send button */
  --widget-body-primary-dark: #00218D;
  --widget-brand-danger:      #E34F45;
}
```

Set these on `:root` of the host page or under any selector that wraps the widget container.

## Refreshing the SSO token

JWTs expire. Refresh proactively (e.g. when ~5 minutes remain) using the runtime helper:

```js
window.antrika('setSsoToken', { widgetId: 'WIDGET_ID', ssoToken: 'newJwt…' });
```

The widget uses the new token on the next request. In-flight requests are unaffected.

## Gotchas

- **Origin must be in the agent's allow-list.** A request from `https://app.example.com` to an agent that only allows `https://example.com` returns `403 Not authorised`. Globs work: `https://*.example.com`.
- **Same `remoteId` across tenants would share a session.** Include a tenant prefix (`cust-123:`) in your `remoteId` if your code paths could possibly collide.
- **JWT must be valid when sent.** Refresh before the `exp` claim using the helper above.
- **CSV upload is multipart**, so it triggers a CORS preflight. The widget endpoint handles `OPTIONS` automatically — no host config needed.

## Worked example

A full HTML page that renders two reports with their own AI buttons:

```html
<!doctype html>
<html>
<head>
  <link href="https://widget.antrika.com/app.css" rel="stylesheet">
</head>
<body>
  <h1>Daily Reports</h1>

  <section>
    <h2>Attendance — Yesterday</h2>
    <table id="att-yest">…</table>
    <div class="antrika-table-ai-widget"
         data-id="agt_xxx"
         data-remote-id="report:attendance:2026-05-08:cust-123"
         data-report-name="Attendance Yesterday"></div>
  </section>

  <section>
    <h2>Leaves — Yesterday</h2>
    <table id="lvs-yest">…</table>
    <div class="antrika-table-ai-widget"
         data-id="agt_xxx"
         data-remote-id="report:leaves:2026-05-08:cust-123"
         data-report-name="Leaves Yesterday"></div>
  </section>

  <script src="https://widget.antrika.com/app.js"></script>
  <script>
    fetch('/me/tableai-token').then(r => r.json()).then(({ token }) => {
      window.antrika('renderTableAI', {
        widgetId: 'agt_xxx',
        ssoToken: token
      });
    });
  </script>
</body>
</html>
```

The `/me/tableai-token` endpoint is yours — it returns a JWT minted as in the [auth section](#auth--minting-the-sso-token).
