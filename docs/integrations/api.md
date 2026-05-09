# Server-to-Server / REST Integration

Use this when there's no human in the loop, or when you're building a custom UI you want to keep on your own backend.

## Auth

API-key authentication. Include on every request:

```
apiKey: <your-api-key>
```

Plus the `Origin` header your HTTP client sets automatically. The agent's allow-list controls which Origins (or IPs, if you set `apikey.ip`) can use the key.

Get the key from your tenant admin: **Admin → Agents → your TableAI agent → API Keys → Create**.

Full auth detail: [api/auth.md](../api/auth.md).

## Lifecycle

Same as every other surface — read [concepts.md](../concepts.md) for the model.

```
1. init  ──►  GET  /emp/1/api/tableai/init   (header: remoteSessionId)
              ◄── { sessionData: { sessionId, fileAvailable } , data: [history] }

2. if !fileAvailable:
   upload ──► POST /emp/1/api/tableai/upload (multipart file=csv, header sessionId)

3. chat  ──►  POST /emp/1/api/tableai/chat   (body { sessionId, query, stream:true })
              ◄── SSE: TEXT* CHART? DONE     (or { answer, chartData } if stream:false)

4. dashboard (optional)
   ──► POST /emp/1/api/dashboard/spec        (body { sessionId })
   ──► POST /emp/1/api/dashboard/widget/data (body { sessionId, widgetId, offset, limit })
```

Full request/response reference: [api/endpoints.md](../api/endpoints.md).

## Worked example — Node.js

```javascript
const fetch = require('node-fetch');
const FormData = require('form-data');
const crypto = require('crypto');

const BASE   = 'https://yourcompany.antrika.com';
const APIKEY = process.env.TABLEAI_API_KEY;

const HEADERS = {
  apiKey: APIKEY,
  Origin: 'https://your-server.example.com'
};

async function ask(csv, query) {
  const remoteSessionId = 'tai-srv-' +
    crypto.createHash('sha256').update(csv).digest('hex').slice(0, 32);

  // 1. init
  const init = await fetch(
    `${ BASE }/emp/1/api/tableai/init`,
    { headers: { ...HEADERS, remoteSessionId, pt: '50', pn: '0' } }
  ).then(r => r.json());

  if (!init?.sessionData?.sessionId) {
    throw new Error(init?.ed || 'init failed');
  }
  const sessionId = init.sessionData.sessionId;

  // 2. upload if needed
  if (!init.sessionData.fileAvailable) {
    const form = new FormData();
    form.append('file', Buffer.from(csv, 'utf-8'), 'data.csv');
    const up = await fetch(
      `${ BASE }/emp/1/api/tableai/upload`,
      { method: 'POST', headers: { ...HEADERS, sessionId }, body: form }
    ).then(r => r.json());
    if (up?.s === false) throw new Error(up.ed || 'upload failed');
  }

  // 3. chat (non-streaming for simplicity here)
  const ans = await fetch(
    `${ BASE }/emp/1/api/tableai/chat`,
    {
      method: 'POST',
      headers: { ...HEADERS, 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionId, query, stream: false })
    }
  ).then(r => r.json());

  return ans;
}

(async () => {
  const csv = 'Employee,Hours\nAlice,8\nBob,7\nCarol,9';
  console.log(await ask(csv, 'Who worked the most?'));
})();
```

## Streaming version

When you want token-by-token output (e.g. relaying to a UI):

```javascript
const res = await fetch(`${ BASE }/emp/1/api/tableai/chat`, {
  method: 'POST',
  headers: { ...HEADERS, 'Content-Type': 'application/json',
             'Accept': 'text/event-stream, application/json' },
  body: JSON.stringify({ sessionId, query, stream: true })
});

const reader = res.body.getReader();
const decoder = new TextDecoder('utf-8');
let buf = '';
let answer = '';

for (;;) {
  const { value, done } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });

  let i;
  while ((i = buf.indexOf('\n\n')) !== -1) {
    const event = buf.slice(0, i);
    buf = buf.slice(i + 2);
    for (const line of event.split('\n')) {
      if (!line.startsWith('data:')) continue;
      const raw = line.slice(5).trim();
      if (!raw) continue;
      let parsed;
      try { parsed = JSON.parse(raw); } catch { continue; }
      switch (parsed.contentType) {
        case 'TEXT':  answer += parsed.content || ''; break;
        case 'CHART': /* render or store parsed.data */ break;
        case 'ERROR': throw new Error(parsed.content || 'stream error');
        case 'DONE':  return answer;
      }
    }
  }
}
```

Stream framing reference: [api/sse-protocol.md](../api/sse-protocol.md).

## Worked example — Java

```java
HttpClient http = HttpClient.newHttpClient();
String base = "https://yourcompany.antrika.com";
String apiKey = System.getenv("TABLEAI_API_KEY");

String csv = "Employee,Hours\nAlice,8\nBob,7\nCarol,9";

// remoteSessionId
MessageDigest md = MessageDigest.getInstance("SHA-256");
byte[] hash = md.digest(csv.getBytes(StandardCharsets.UTF_8));
String remoteSessionId = "tai-srv-" + HexFormat.of().formatHex(hash).substring(0, 32);

// 1. init
HttpRequest initReq = HttpRequest.newBuilder()
    .uri(URI.create(base + "/emp/1/api/tableai/init"))
    .header("apiKey", apiKey)
    .header("remoteSessionId", remoteSessionId)
    .header("pt", "50").header("pn", "0")
    .GET().build();
HttpResponse<String> initRes = http.send(initReq, HttpResponse.BodyHandlers.ofString());
JsonObject init = JsonParser.parseString(initRes.body()).getAsJsonObject();
String sessionId = init.getAsJsonObject("sessionData").get("sessionId").getAsString();
boolean fileAvailable = init.getAsJsonObject("sessionData").get("fileAvailable").getAsBoolean();

// 2. upload (if needed) — skipped here for brevity; use a multipart library

// 3. chat (non-streaming)
String body = "{\"data\":{\"sessionId\":\"" + sessionId + "\","
            + "\"query\":\"Who worked the most?\",\"stream\":false}}";
HttpRequest chatReq = HttpRequest.newBuilder()
    .uri(URI.create(base + "/emp/1/api/tableai/chat"))
    .header("apiKey", apiKey)
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
HttpResponse<String> chatRes = http.send(chatReq, HttpResponse.BodyHandlers.ofString());
System.out.println(chatRes.body());
```

## Idempotency and resume

- Calling `init` with the same `remoteSessionId` always returns the same `sessionId`. Safe to retry.
- Calling `upload` against an existing `fileAvailable=true` session is a no-op (the second file is rejected). Always check `init.sessionData.fileAvailable` first.
- Calling `chat` against a session with `fileAvailable=false` returns `{ ed: "File not available" }`. Upload first.

## Rate limits

Per-tenant, defined by the customer's plan. Hitting the limit returns `429`. Plan-bound features are also gated — base TableAI access plus a separate API access flag must both be enabled on your tenant.

If you get `403 "This feature is not available at the moment"`, your tenant's plan needs the relevant feature enabled. Contact your account manager.
