# SSE Streaming Protocol

The chat endpoint streams answers as Server-Sent Events when called with `stream:true`.

## Wire format

Standard SSE over HTTP. Each event is a `data:` line followed by `\n\n`:

```
data: {"contentType":"TEXT","content":"Bob "}

data: {"contentType":"TEXT","content":"worked "}

data: {"contentType":"TEXT","content":"the most "}

data: {"contentType":"CHART","data":{ … }}

data: {"contentType":"DONE"}
```

Notes:

- Each event is one JSON object per `data:` line.
- The double-newline separator is required; chunks may straddle TCP packets, so buffer until you see `\n\n`.
- The connection closes after `DONE` or `ERROR`.
- The server may send keep-alive `:` comment lines (ignore any line not starting with `data:`).

## Event types

| `contentType` | Payload | Meaning |
|---|---|---|
| `TEXT` | `content: string` | Token-stream of the answer. Append to the current bubble. |
| `CHART` | `data: object` | A chart spec + data to render inline. |
| `ERROR` | `content: string` | Terminal. Render error, stop reading. |
| `DONE` | (none) | Terminal. Connection closes immediately after. |

If your client encounters an unrecognised `contentType`, ignore that event and keep reading — new event types may be added in the future.

## Minimal client (browser)

```javascript
async function streamChat({ baseUrl, headers, body, onText, onChart, onError, onDone }) {
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
          case 'TEXT':  onText?.(p.content || ''); break;
          case 'CHART': onChart?.(p.data || null); break;
          case 'ERROR': onError?.(new Error(p.content || 'stream error')); return;
          case 'DONE':  onDone?.(); return;
          // ignore unknown contentTypes — the protocol may grow
        }
      }
    }
  }
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

Server stops generation when the client closes the connection — no extra "stop" call needed.

## Reconnects and idempotency

The protocol does **not** support resuming a partial stream. If the connection drops mid-answer:

- The partial text the client has is the partial text (display it as-is or discard).
- A retry sends the same query again — the agent will produce a new answer (possibly slightly different wording). It does **not** resume from where it left off.
- The session's chat history records each user turn; a duplicated retry will appear as a duplicated turn unless you de-dupe client-side before sending.

## Chart payload shape

The `CHART` event's `data` field is the renderable chart spec:

```jsonc
{
  "type":   "bar",        // bar | line | pie | scatter | …
  "title":  "Hours by employee",
  "x":      ["Alice", "Bob", "Carol"],
  "y":      [8, 9, 7],
  "series": "Hours",
  "config": { /* optional renderer hints */ }
}
```

Different chart types vary the shape (e.g. `pie` uses `labels` + `values`). Defensive renderers fall back to a data table when an unknown shape comes through.

## Why not WebSockets?

SSE was chosen because:

- It tunnels through any HTTP(S) infrastructure (load balancers, proxies, CDNs) without special config.
- It's one-way (server → client), which matches the chat-streaming use case.
- Browser support is universal — `fetch` + a `ReadableStream` reader is enough.

If you need bi-directional streaming (e.g. mid-answer user interrupts), open an issue.
