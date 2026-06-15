# SSE Streaming Protocol

The chat endpoint streams answers as Server-Sent Events when called with
`stream:true`.

## Wire format

Standard SSE over HTTP. Each event is a `data:` line followed by `\n\n`:

```
data: {"contentType":"STATUS","content":"Great question — on it!"}

data: {"contentType":"TEXT","content":"Bob "}

data: {"contentType":"TEXT","content":"worked "}

data: {"contentType":"TEXT","content":"the most "}

data: {"contentType":"CHART","data":{"chartConfig":"{…}","chartData":"{…}"}}

data: {"contentType":"DONE"}
```

Notes:

- Each event is one JSON object per `data:` line, with up to three fields:
  `contentType`, `content` (token text), and `data` (structured payload).
- The double-newline separator is required; chunks may straddle TCP packets, so
  buffer until you see `\n\n`.
- The connection closes after `DONE` or `ERROR`.
- The server may send keep-alive `:` comment lines (ignore any line not starting
  with `data:`).
- The streaming response also sets an **`X-Session-Id`** header so first-turn
  callers learn the session id.

## Event types

| `contentType` | Payload | Meaning |
|---|---|---|
| `STATUS` | `content: string` | A short "working on it" status, usually first. Display transiently or ignore. |
| `TEXT` | `content: string` | Token-stream of the answer. Append to the current bubble. |
| `CHART` | `data: { chartConfig, chartData }` | A chart to render inline. Both fields are JSON **strings** — parse them. |
| `ERROR` | `content: string` | Terminal. Render error, stop reading. |
| `DONE` | (none) | Terminal. Connection closes immediately after. |

Other `contentType`s may appear (`SESSION`, `ACTION_CONFIRMATION`, `ACTION_INPUT`).
If your client encounters an unrecognised `contentType`, ignore that event and keep
reading — the protocol may grow.

## Minimal client (browser)

```javascript
async function streamChat({ baseUrl, headers, body, onText, onChart, onError, onDone, onStatus }) {
  const res = await fetch(baseUrl + '/emp/1/api/tableai/chat', {
    method: 'POST',
    headers: {
      ...headers,
      'Content-Type': 'application/json',
      'Accept':       'text/event-stream, application/json'
    },
    body: JSON.stringify(body)
  });

  if (!res.ok || !res.body) {
    onError?.(new Error(`HTTP ${ res.status }`));
    return;
  }

  const reader  = res.body.getReader();
  const decoder = new TextDecoder('utf-8');
  let buffer = '';

  for (;;) {
    const { value, done } = await reader.read();
    if (done) { onDone?.(); break; }
    buffer += decoder.decode(value, { stream: true });

    let i;
    while ((i = buffer.indexOf('\n\n')) !== -1) {
      const event = buffer.slice(0, i);
      buffer = buffer.slice(i + 2);

      for (const line of event.split('\n')) {
        if (!line.startsWith('data:')) continue;
        const raw = line.slice(5).trim();
        if (!raw) continue;
        let p; try { p = JSON.parse(raw); } catch { continue; }

        switch (p.contentType) {
          case 'STATUS': onStatus?.(p.content || ''); break;
          case 'TEXT':   onText?.(p.content || ''); break;
          case 'CHART':  onChart?.(parseChart(p.data)); break;
          case 'ERROR':  onError?.(new Error(p.content || 'stream error')); return;
          case 'DONE':   onDone?.(); return;
          // ignore unknown contentTypes — the protocol may grow
        }
      }
    }
  }
}

// CHART payload fields are JSON-encoded strings.
function parseChart(data) {
  if (!data) return null;
  const j = s => (typeof s === 'string' ? JSON.parse(s) : s);
  return { config: j(data.chartConfig), data: j(data.chartData) };
}
```

## Aborting

Abort cleanly with an `AbortController`:

```javascript
const ctrl = new AbortController();
fetch(url, { …, signal: ctrl.signal });
// later:
ctrl.abort();
```

Server stops generation when the client closes the connection — no extra "stop"
call needed.

## Reconnects and idempotency

The protocol does **not** support resuming a partial stream. If the connection
drops mid-answer:

- The partial text the client has is the partial text (display it as-is or
  discard).
- A retry sends the same query again — the agent produces a new answer (possibly
  slightly different wording). It does **not** resume from where it left off.
- The session's chat history records each user turn; a duplicated retry appears as
  a duplicated turn unless you de-dupe client-side before sending.

## Chart payload shape

The `CHART` event's `data` object carries two JSON-encoded strings:

```jsonc
{
  "chartConfig": "{ \"type\": \"bar\", \"title\": \"Hours by employee\", … }",
  "chartData":   "{ \"x\": [\"Alice\",\"Bob\"], \"y\": [8,9], … }"
}
```

Parse both before rendering. `chartConfig` holds the chart type and presentation
hints; `chartData` holds the series. Defensive renderers fall back to a data table
when a shape is unknown.

## Why not WebSockets?

SSE was chosen because:

- It tunnels through any HTTP(S) infrastructure (load balancers, proxies, CDNs)
  without special config.
- It's one-way (server → client), which matches the chat-streaming use case.
- Browser support is universal — `fetch` + a `ReadableStream` reader is enough.

If you need bi-directional streaming (e.g. mid-answer user interrupts), open an
issue.
