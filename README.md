# tableArth.ai Documentation

tableArth.ai lets users ask natural-language questions of any tabular dataset
and get back text, charts, and a generated dashboard. It can be embedded in any
web product, dropped into Google Chrome, or called server-to-server.

This repo is the integration handbook. If you're shipping tableArth.ai inside
your own surface, start here.

> The brand is **tableArth.ai**; the HTTP API keeps the original `tableai` path
> slug (e.g. `/emp/1/api/tableai/init`). Both are correct — one's the name, one's
> the wire.

## Pick your path

| Surface | Best for | Doc |
|---|---|---|
| **JS widget** | Embedding alongside an existing report or table in your own web app. | [docs/integrations/widget.md](docs/integrations/widget.md) |
| **Chrome extension** | Letting end users analyse any table on any third-party page. | [docs/integrations/chrome-extension.md](docs/integrations/chrome-extension.md) |
| **Server-to-server REST** | Programmatic ingestion / batch / custom UI. | [docs/integrations/api.md](docs/integrations/api.md) |

If you're not sure which one you need, the rule of thumb:

- **My users already log into my product** → JS widget.
- **My users browse other people's products** → Chrome extension.
- **No human in the loop** → REST API.

## Read before integrating

- [Concepts](docs/concepts.md) — agents, sessions, `remoteId`, the upload→chat lifecycle. Five minutes; saves hours later.
- [Auth modes](docs/api/auth.md) — which credential goes where, and why the widget and the API authenticate differently.
- [Agent setup](docs/admin/agent-setup.md) — what an admin needs to do once before any integration works.

## Reference

- [Endpoint reference](docs/api/endpoints.md) — every public route, headers, request/response shapes.
- [SSE protocol](docs/api/sse-protocol.md) — how the chat stream is framed.
- [Troubleshooting](docs/troubleshooting.md) — common 401/403/CORS pitfalls.

## Getting help

If something blocks your integration, contact your tenant administrator or the
tableArth.ai support channel with:

- The surface (widget / Chrome extension / REST).
- The full response body you received (`s`, `ed`, `msg`).
- The browser and version, if it's a browser-side issue.
