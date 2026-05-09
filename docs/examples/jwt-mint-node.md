# JWT Minting — Node.js

A minimal example of minting the `ssoToken` JWT the JS widget will pass on every backend request. Keep this on your **server**; never ship the agent secret to the browser.

## Install

```bash
npm install jsonwebtoken
```

## Mint a token

```javascript
const jwt = require('jsonwebtoken');

// From admin → AI Agents → your agent → Show secret. Server-only.
const AGENT_SECRET = process.env.TABLEAI_AGENT_SECRET;

function mintTableAiToken(user, account, ttlSeconds = 3600) {
  const now = Math.floor(Date.now() / 1000);
  const payload = {
    sub:   user.id,
    name:  user.name,
    email: user.email,
    companies: [
      {
        id:           account.id,
        name:         account.name,
        // optional — maps to backend account custom fields by name
        customFields: account.customFields || {}
      }
    ],
    // optional — maps to backend user custom fields by name
    customFields: user.customFields || {},
    iat: now,
    exp: now + ttlSeconds
  };

  return jwt.sign(payload, AGENT_SECRET, { algorithm: 'HS256' });
}

module.exports = { mintTableAiToken };
```

## Expose to your page

A simple Express endpoint your front-end can hit on page load:

```javascript
const express = require('express');
const { mintTableAiToken } = require('./tableai-token');

const app = express();

app.get('/me/tableai-token', requireAuth, (req, res) => {
  // requireAuth has already populated req.user / req.account
  const token = mintTableAiToken(req.user, req.account);
  res.json({ token, expiresIn: 3600 });
});
```

## Front-end

```html
<div class="antrika-table-ai-widget"
     data-id="agt_xxx"
     data-remote-id="report:attendance:2026-05-08:cust-123"></div>

<script src="https://widget.antrika.com/app.js"></script>
<script>
  fetch('/me/tableai-token', { credentials: 'include' })
    .then(r => r.json())
    .then(({ token }) => {
      window.antrika('renderTableAI', {
        widgetId: 'agt_xxx',
        ssoToken: token
      });
    });
</script>
```

## Refreshing before expiry

Mint with `ttlSeconds = 3600` (1h). Refresh proactively when ~5 min remain:

```javascript
function startRefresh() {
  let token, exp;

  async function refresh() {
    const res = await fetch('/me/tableai-token', { credentials: 'include' }).then(r => r.json());
    token = res.token;
    exp   = Date.now() + (res.expiresIn - 300) * 1000;  // refresh 5 min early
    window.antrika('setSsoToken', { widgetId: 'agt_xxx', ssoToken: token });
  }

  refresh();
  setInterval(() => Date.now() >= exp && refresh(), 60_000);
}
```

## Don't

- **Don't** put `AGENT_SECRET` in any front-end bundle, env var consumed by the browser, HTML, or query string.
- **Don't** mint long-lived tokens ("never expires") — short TTL + refresh is the whole point.
- **Don't** sign with a different algorithm — backend expects HS256.

## Verify locally

```javascript
const { mintTableAiToken } = require('./tableai-token');
console.log(mintTableAiToken(
  { id: 'u_42', name: 'Test User', email: 't@x.com' },
  { id: 'a_1', name: 'Test Co' }
));
```

Drop the resulting JWT into [jwt.io](https://jwt.io) → header should show `{ alg: HS256, typ: JWT }`, payload should match what you passed in. Signature won't verify in the public tool unless you paste the secret.
