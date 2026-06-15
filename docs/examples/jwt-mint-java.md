# SSO Token Minting — Java

A minimal example of minting the **SSO token** the JS widget needs. Your page hands
this token to `window.antrika('syncUser', …)`, which establishes the visitor's
session (see [widget auth](../integrations/widget.md#auth)). Server-side only —
never ship the SSO signing key to the browser.

This example uses the `io.jsonwebtoken:jjwt` library; any HS256-capable JWT library
will work.

## Dependency

Maven:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

## Mint a token

```java
package com.example.tableai;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.List;
import java.util.Map;

public final class SsoTokenMinter {

    // From admin → SSO settings. The tenant SSO signing key — NOT the agent API key.
    private final SecretKey signingKey;

    public SsoTokenMinter(String ssoSigningKey) {
        this.signingKey = Keys.hmacShaKeyFor(ssoSigningKey.getBytes(StandardCharsets.UTF_8));
    }

    public String mint(User user, Account account, long ttlSeconds) {
        Instant now = Instant.now();
        return Jwts.builder()
            // the user-id claim is `id`, NOT the standard `sub`
            .claim("id",    user.id())
            .claim("name",  user.name())
            .claim("email", user.email())
            .claim("companies", List.of(Map.of(
                "id",           account.id(),
                "name",         account.name(),
                // optional — maps to backend account custom fields by name
                "customFields", account.customFields()
            )))
            // optional — maps to backend user custom fields by name
            .claim("customFields", user.customFields())
            .issuedAt(java.util.Date.from(now))
            .expiration(java.util.Date.from(now.plusSeconds(ttlSeconds)))
            .signWith(signingKey, Jwts.SIG.HS256)
            .compact();
    }

    public record User(String id, String name, String email, Map<String, String> customFields) {}
    public record Account(String id, String name, Map<String, String> customFields) {}
}
```

## Expose to your page (Spring Boot)

Return the `customerId` (your tenant id) alongside the token — `syncUser` needs both:

```java
@RestController
@RequestMapping("/me")
public class SsoTokenController {

    private final SsoTokenMinter minter =
        new SsoTokenMinter(System.getenv("TABLEARTH_SSO_KEY"));
    private final String tenantId = System.getenv("TABLEARTH_TENANT_ID");

    @GetMapping("/tableai-token")
    public Map<String, Object> token(@AuthenticationPrincipal MyUser principal) {
        var user = new SsoTokenMinter.User(
            principal.getId(), principal.getName(), principal.getEmail(), Map.of()
        );
        var account = new SsoTokenMinter.Account(
            principal.getAccountId(), principal.getAccountName(), Map.of()
        );
        var jwt = minter.mint(user, account, 3600);
        return Map.of("token", jwt, "customerId", tenantId, "expiresIn", 3600);
    }
}
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
      window.antrika('syncUser', { customerId, token }, () => {
        window.antrika('renderTableAI', {
          widgetId:        'agt_xxx',
          remoteSessionId: 'report:attendance:2026-05-08:cust-123'
        });
      });
    });
</script>
```

> Use `syncUserLocal` instead of `syncUser` against a local/dev tenant.

## Refreshing

There is no `setSsoToken` call. Mint short (≤ 1 hour) and re-establish the session
before `exp` by minting a fresh token and calling `syncUser` again:

```javascript
async function refresh() {
  const { token, customerId, expiresIn } =
    await fetch('/me/tableai-token').then(r => r.json());
  window.antrika('syncUser', { customerId, token });
  setTimeout(refresh, (expiresIn - 300) * 1000);
}
refresh();
```

## Don't

- **Don't** load `TABLEARTH_SSO_KEY` from a config file shipped with the front-end build.
- **Don't** confuse the SSO signing key with the agent **API key** — different credentials.
- **Don't** sign with `none` / unsigned tokens — the backend rejects them.
- **Don't** hard-code the key in test fixtures committed to git.
