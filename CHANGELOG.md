# Changelog

## Unreleased

### 2026-06-15 — Accuracy pass against shipped code

- **Rebrand:** product is now **tableArth.ai** throughout (HTTP paths keep the
  `tableai` slug).
- **Removed the Excel add-in** docs — it is a pre-rebrand prototype, not a shipped
  surface.
- **Widget auth rewritten** to the shipped model: the host mints an SSO JWT and
  calls `window.antrika('syncUser', …)` to establish the `antrikaSSO` session; the
  widget signs each request internally with `widget_id`. Removed the non-existent
  `ssoToken` render argument and `setSsoToken` helper.
- **Endpoints corrected:** `apikey/validate` → `extension/validate` (+ new
  `extension/domains`); API-key `chat` body is flat (only the widget wraps it in
  `data`); upload returns `tableFileId` / `dashboardSpec` / `remoteSessionId`;
  history items use `message` (not `answer`); added the widget dashboard routes and
  the `/v2` per-user API routes.
- **SSE:** documented the `STATUS` event and the real `CHART` payload
  (`chartConfig` + `chartData`, JSON strings).
- **Auth/admin:** widget signs with the tenant **SSO signing key** (not the agent
  API key); feature flags documented (`agent-data-query`, `apiAccess`,
  `widgetIntegration`, `singleSignOn`); session `source` labels
  (`API`/`Widget`/`Extension`/`Admin`).
- **SSO token examples** (Node, Java) now mint the `syncUser` token (claim `id`,
  not `sub`).

### Initial set

- `README.md` with surface picker (widget / Chrome extension / REST).
- `docs/concepts.md` — agents, sessions, `remoteId`, lifecycle.
- Integration guides for the JS widget, Chrome extension, and server-to-server REST.
- API reference: auth modes, endpoint reference, SSE protocol.
- Admin guide for one-time agent setup.
- Troubleshooting and SSO-token-minting examples (Node, Java).
