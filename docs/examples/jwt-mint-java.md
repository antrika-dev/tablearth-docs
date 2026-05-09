# JWT Minting — Java

A minimal example of minting the `ssoToken` JWT the JS widget will pass on every backend request. Server-side only — never ship the agent secret to the browser.

This example uses the `io.jsonwebtoken:jjwt` library; any HS256-capable JWT library will work.

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

public final class TableAiTokenMinter {

    // From admin → AI Agents → your agent → Show secret. Server-only.
    private final SecretKey signingKey;

    public TableAiTokenMinter(String agentSecret) {
        this.signingKey = Keys.hmacShaKeyFor(agentSecret.getBytes(StandardCharsets.UTF_8));
    }

    public String mint(User user, Account account, long ttlSeconds) {
        Instant now = Instant.now();
        return Jwts.builder()
            .subject(user.id())
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

```java
@RestController
@RequestMapping("/me")
public class TableAiTokenController {

    private final TableAiTokenMinter minter =
        new TableAiTokenMinter(System.getenv("TABLEAI_AGENT_SECRET"));

    @GetMapping("/tableai-token")
    public Map<String, Object> token(@AuthenticationPrincipal MyUser principal) {
        var user = new TableAiTokenMinter.User(
            principal.getId(), principal.getName(), principal.getEmail(), Map.of()
        );
        var account = new TableAiTokenMinter.Account(
            principal.getAccountId(), principal.getAccountName(), Map.of()
        );
        var jwt = minter.mint(user, account, 3600);
        return Map.of("token", jwt, "expiresIn", 3600);
    }
}
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

## Refreshing

Mint short (≤ 1 hour). Refresh ~5 minutes before `exp`:

```javascript
async function refresh() {
  const { token, expiresIn } = await fetch('/me/tableai-token').then(r => r.json());
  window.antrika('setSsoToken', { widgetId: 'agt_xxx', ssoToken: token });
  setTimeout(refresh, (expiresIn - 300) * 1000);
}
refresh();
```

## Don't

- **Don't** load `TABLEAI_AGENT_SECRET` from a config file shipped with the front-end build.
- **Don't** sign with `none` / unsigned tokens — the backend rejects them.
- **Don't** hard-code the secret in test fixtures committed to git.
