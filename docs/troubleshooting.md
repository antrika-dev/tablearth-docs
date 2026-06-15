# Troubleshooting

The most common integration issues and how to fix them. Symptoms first, causes second.

## Auth / 401 / 403

### `401 Missing API key`

You forgot the `apiKey` header. Browsers won't send it automatically — set it on every request, including preflight where applicable.

### `401 Invalid API key`

Key is wrong, deleted, or for a different tenant.

```bash
curl -i -X POST '$BASE/emp/1/api/tableai/extension/validate' \
  -H 'apiKey: ak_live_…' -H 'Origin: $ORIGIN' -d '{}'
```

`{ s: true }` → key is good. Anything else → admin gave you the wrong key.

### `403 Not authorised`

Origin (or IP) isn't in the agent's allow-list.

- Check the **exact Origin** the browser sends (DevTools → Network → request headers). `https://app.example.com` ≠ `https://app.example.com:443` ≠ `http://app.example.com`.
- If the integrator is behind a reverse proxy that rewrites Host, the Origin header may also be rewritten — fix the proxy or add the rewritten Origin to the allow-list.
- For local development, add `http://localhost:3000` (or whatever port) explicitly. `localhost` does **not** auto-match `127.0.0.1`.
- Use a glob if you need many subdomains: `https://*.example.com`.

### `403 Agent is not active`

Agent status is DRAFT or DISABLED. Admin → Agents → set to ACTIVE.

### `403 Customer is not active`

The tenant is suspended. Contact your account manager.

### `403 This feature is not available at the moment`

Plan-bound. The customer is missing one of the plan flags:

- `agent-data-query` — base module, required for any surface.
- `apiAccess` — required for the `/api/tableai/*` API-key routes.
- `widgetIntegration` — required for the `/api/tableai/widget/*` routes.
- `singleSignOn` — required for widget chat/init and all `/v2` routes.

Check with admin → Billing.

## Session / chat

### `{ ed: "Missing Session" }`

You called `chat` without a `sessionId` in the body. Always `init` first.

### `{ ed: "Invalid Session" }`

The `sessionId` doesn't belong to the same tenant as your `apiKey` / `agentId`. Likely causes:

- Re-using a `sessionId` from a different environment (staging vs prod).
- Tenant migration where session IDs were re-keyed.
- Typo / truncation.

Re-`init` to get a fresh `sessionId`.

### `{ ed: "File not available" }`

Session exists but no CSV was uploaded. `init` returned `fileAvailable=false` and you skipped the upload. Order is:

```
init → (if !fileAvailable) upload → chat
```

### Same `remoteSessionId` returns different sessions for different users

You're keying by user-specific data (e.g. URL with auth token). Either:

- Strip the volatile bits before hashing.
- Switch to a stable `remoteId` driven by report identity, not URL.

### Two unrelated reports return the *same* session

Hash collision or your stable IDs clash. Add a tenant prefix and a report-type discriminator: `cust-123:attendance:2026-05-08`.

### Chat history doesn't load

`init` returns messages **newest-first**. Reverse the array client-side before rendering chronologically. The mapping is `messageType: 'sent'` = assistant, anything else = user.

## CSV upload

### `413 Payload Too Large` / upload silently rejected

CSV exceeds the plan's per-file size limit (default 25 MB). Slice upstream — the agent answers against parquet, so trimming columns helps more than trimming rows.

### `{ ed: "File already uploaded" }`

You uploaded twice into the same session. The second upload is rejected to keep the parquet stable. To re-upload, use a new `remoteSessionId` (mint a new one — change the data hash or stable ID).

### CORS preflight fails on `OPTIONS /upload`

The upload endpoint requires:

- The `apiKey` to be present **as a query parameter** on the `OPTIONS` request (browsers can't send custom headers on preflight).
- A matching Origin in the allow-list.

If your client only puts `apiKey` in the header, switch to `?apiKey=…` for the upload URL specifically:

```javascript
fetch(`${ base }/emp/1/api/tableai/upload?apiKey=${ encodeURIComponent(key) }`, {
  method: 'POST',
  headers: { apiKey: key, sessionId },     // header still required for the POST itself
  body:    formData
});
```

## Streaming

### Stream stops mid-answer

Likely causes:

- Browser tab went to sleep / network changed.
- Reverse proxy (nginx, Cloudflare) buffered the response. SSE needs `proxy_buffering off;` (nginx) and stream-friendly upstream config.
- Client closed the `ReadableStream` early. Make sure you only `reader.cancel()` after seeing `DONE` or `ERROR`.

### Got a `data:` line that didn't parse

Don't fail the whole stream — `continue` on the bad line. The SSE protocol allows comment lines (starting with `:`) which look like data; ignore them. See [sse-protocol.md](api/sse-protocol.md).

### Tokens arrive duplicated ("TheThe", "Bob Bob")

You're processing the same SSE chunk twice. In a Chrome extension this happens when the relay both broadcasts via `chrome.runtime.sendMessage` and posts via `chrome.tabs.sendMessage`. Pick one.

## Dashboard

### `dashboard/spec` returns `ready: false`

The background generator hasn't finished. Show a placeholder; refetch on tab switch or every 10–15 seconds. The first session needs ~10–60s depending on data size.

### Widget data renders empty

If `body.data` is a string (legacy responses), parse it:

```javascript
if (typeof body.data === 'string') body.data = JSON.parse(body.data);
```

If still empty, check `body.s` and `body.ed`.

### "Tables only" pagination fields missing on KPI/chart widgets

Expected. KPI/chart widgets ignore `offset`/`limit` server-side and don't return `total`/`hm`. Don't render pagination UI for non-table widgets.

## Widget SSO

### Widget loads but `init` / `chat` fails with auth errors

The `antrikaSSO` session cookie is missing or expired. Make sure `syncUser` ran and
succeeded *before* you called `renderTableAI` (mount in its callback). If `syncUser`
returns an SSO error, the token is the problem — see below.

### `syncUser` rejected the token (SSO data incomplete / Sso token failed)

- Check expiry — `exp` claim is in unix seconds, not milliseconds.
- Check signing algorithm — must be `HS256` with the tenant **SSO signing key**
  (admin → SSO settings), **not** the agent API key.
- Check the user-id claim is `id` (not `sub`).
- Check the `companies[]` claim has at least one entry with `id` **and** `name`.
- Check you passed the correct `customerId` (the token is validated against that
  tenant's SSO settings).

### `widget_id` missing → requests fail auth

The widget signs each request using `widget_id`. Always pass `widgetId` to
`renderTableAI` and set the matching `data-id` on the mount div.

## Networking / infra

### Streaming works in Chrome but breaks in Safari

Safari requires the response to start within ~30s or it gives up. If your agent is slow on first turn, send a comment line (`:keepalive\n\n`) periodically to flush headers.

### Requests succeed locally but 404 in prod

Check you're hitting the right route for your auth mode:

- `/api/tableai/{init,upload,chat}` — API-key routes (Chrome extension, server-to-server).
- `/api/tableai/widget/{init,upload,chat}` — Widget SSO routes (JS widget).
- `/api/tableai/extension/{validate,domains}` — extension activation (replaces the
  old `apikey/validate`).

The route sets are not interchangeable — wrong route + right auth = 401 or 404.

## Still stuck?

Contact your tenant administrator or the tableArth.ai support channel with:

- The surface (widget / extension / api).
- The full request URL, method, and headers (redact the apiKey/JWT).
- The full response body (`ed`, `msg`, `s`).
- Browser + version (for browser-side issues).

Keep response bodies — they're the fastest way to triage.
