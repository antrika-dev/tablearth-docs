# Contributing to tableArth.ai Docs

These docs describe a real product surface; readers will paste code from them into production. Edits should hold to that.

## Ground rules

1. **Document only what is publicly available.** If a feature isn't shipped or isn't part of the public contract, don't describe it. Public docs should never reference internal repos, file paths, class names, or in-flight work.
2. **Show the failure modes.** Every endpoint or auth flow doc should list the realistic ways it can fail and what the user sees. The [troubleshooting](docs/troubleshooting.md) page is the catch-all but per-page sections should mention the top three.
3. **Worked examples beat prose.** Wherever possible, show the request/response pair, the curl, the code snippet. Diagrams of state machines beat paragraphs about them.
4. **No marketing copy.** Skip "powerful", "seamless", "world-class". Say what it does.

## File layout

```
README.md                         landing page + path picker
CHANGELOG.md
CONTRIBUTING.md                   this file
docs/
├── concepts.md                   the mental model — read first
├── integrations/
│   ├── widget.md
│   ├── chrome-extension.md
│   └── api.md
├── api/
│   ├── auth.md
│   ├── endpoints.md
│   └── sse-protocol.md
├── admin/
│   └── agent-setup.md
├── examples/
│   ├── jwt-mint-node.md
│   └── jwt-mint-java.md
└── troubleshooting.md
```

Add new integration surfaces under `docs/integrations/`. Add new auth modes under `docs/api/auth.md` (don't fork it). Add new examples under `docs/examples/`.

## Style

- **Voice**: direct, second-person ("you upload the CSV"), present tense.
- **Headings**: sentence case ("Auth modes" not "Auth Modes").
- **Code**: fenced blocks with the language tag. JSON examples annotated with `jsonc` if they have comments.
- **Tables** for anything with three or more parallel options (auth modes, error codes, surfaces).

## Cross-references

Use relative paths from the file you're editing. Examples:

- From `docs/integrations/widget.md` → `[concepts](../concepts.md)`
- From `docs/api/auth.md` → `[endpoints](endpoints.md)`
- From `README.md` → `[widget](docs/integrations/widget.md)`

## Reviewing changes

A PR is mergeable when:

- Curl / code snippets actually work against the public API.
- Cross-document links resolve.
- New endpoints are listed in [`docs/api/endpoints.md`](docs/api/endpoints.md) **and** in the relevant integration page.
- The CHANGELOG has a one-line entry for any user-visible change.

Spelling and tone fixes don't need a CHANGELOG entry.
