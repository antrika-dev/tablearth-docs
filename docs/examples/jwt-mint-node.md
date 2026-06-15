# SSO Token Minting — Node.js

A minimal example of minting the **SSO token** the JS widget needs. Your page hands
this token to `window.antrika('syncUser', …)`, which establishes the visitor's
session (see [widget auth](../integrations/widget.md#auth)). Keep this on your
**server**; never ship the SSO signing key to the browser.

## Install

```bash
npm install jsonwebtoken
```

## Mint a token

```javascript
const jwt = require('jsonwebtoken');

// From admin → SSO settings. The tenant SSO signing key — NOT the agent API key.
const SSO_SIGNING_KEY = process.env.TABLEARTH_SSO_KEY;

function mintSsoToken(user, account, ttlSeconds = 3600) {
  const now = Math.floor(Date.now() / 1000);
  const payload = {
    id:    user.id,            // the user-id claim — note: `id`, not `sub`
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

  return jwt.sign(payload, SSO_SIGNING_KEY, { algorithm: 'HS256' });
}

module.exports = { mintSsoToken };
```

## Expose to your page

A simple Express endpoint your front-end can hit on page load. Return the
`customerId` (your tenant id) alongside the token — `syncUser` needs both:

```javascript
const express = require('express');
const { mintSsoToken } = require('./tableai-token');

const app = express();

app.get('/me/tableai-token', requireAuth, (req, res) => {
  // requireAuth has already populated req.user / req.account
  const token = mintSsoToken(req.user, req.account);
  res.json({ token, customerId: process.env.TABLEARTH_TENANT_ID, expiresIn: 3600 });
});
```

## Front-end

```html
<div class="antrika-table-ai-widget"
     data-id="agt_xxx"
     data-report-name="Attendance Yesterday"></div>

<script src="https://<your-widget-host>/app.js"></script>
<script>
  fetch('/me/tableai-token', { credentials: 'include' })
    .then(r => r.json())
    .then(({ token, customerId }) => {
      // 1. establish the SSO session
      window.antrika('syncUser', { customerId, token }, () => {
        // 2. mount the widget once the session exists
        window.antrika('renderTableAI', {
          widgetId:        'agt_xxx',
          remoteSessionId: 'report:attendance:2026-05-08:cust-123'
        });
      });
    });
</script>
```

> Use `syncUserLocal` instead of `syncUser` against a local/dev tenant.

## Refreshing before expiry

There is no `setSsoToken` call. To refresh, mint a fresh token and call `syncUser`
again — the session cookie is re-issued:

```javascript
async function refresh() {
  const { token, customerId, expiresIn } =
    await fetch('/me/tableai-token', { credentials: 'include' }).then(r => r.json());
  window.antrika('syncUser', { customerId, token });
  setTimeout(refresh, (expiresIn - 300) * 1000);   // refresh 5 min early
}
refresh();
```

## Don't

- **Don't** put `SSO_SIGNING_KEY` in any front-end bundle, browser env var, HTML,
  or query string.
- **Don't** confuse the SSO signing key with the agent **API key** — they're
  different credentials for different surfaces.
- **Don't** mint long-lived tokens ("never expires") — short TTL + refresh is the
  whole point.
- **Don't** sign with a different algorithm — the backend expects HS256.

## Verify locally

```javascript
const { mintSsoToken } = require('./tableai-token');
console.log(mintSsoToken(
  { id: 'u_42', name: 'Test User', email: 't@x.com' },
  { id: 'a_1', name: 'Test Co' }
));
```

Drop the resulting JWT into [jwt.io](https://jwt.io) → header should show
`{ alg: HS256, typ: JWT }`, payload should match what you passed in (note the `id`
claim). Signature won't verify in the public tool unless you paste the SSO key.
