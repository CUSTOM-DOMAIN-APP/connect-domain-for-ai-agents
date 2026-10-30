# Custom domains for AI agents

A hosted MCP server and REST API that let an agent connect a customer's own domain to the app it built, then
report when the domain is `live`. No tool accepts raw DNS records, and the agent never holds a registrar password.

[Custom domains for AI agents on customdomain.ai](https://customdomain.ai/gh/ai-agents) · [The MCP server in the docs](https://docs.customdomain.ai/docs/mcp/overview) · [Start free: first ten domains, no card](https://app.customdomain.ai/signup)

An agent can generate a codebase, provision infrastructure and deploy a working app, then stall on
one step: putting it on a domain the customer already owns. This repository explains what that step actually
requires (DNS records, proof of control, TLS, propagation) and shows the three ways an agent can finish it
without ever holding a registrar password. It is maintained by [CustomDomain™](https://customdomain.ai), which
runs a hosted MCP server and REST API for the job.

## The problem: agents ship everything except the domain

An agent that builds and deploys a site hands back a preview URL on a platform subdomain. The next message is
always the same: make it live on acmebakery.com. At that moment three things have to happen inside
infrastructure the agent does not control. Routing records have to be written into someone else's DNS zone.
Control of that zone has to be demonstrated. A TLS certificate has to be issued for the hostname. Then
everything waits on caches expiring.

Humans already fail at this. The Domain Connect knowledge base reports that roughly half of the users who
attempt manual DNS configuration abandon it, and documents a major productivity suite whose email onboarding
needs 7 to 15 hand-created records and 16 help sites, 10 of them specific to one registrar. Those users had
already paid; they could not turn "add a CNAME" into the right clicks at their own provider. For an agent it is
worse, because every human workaround is either unavailable or dangerous:

- **No hands.** The DNS panel is a browser UI, often behind 2FA. Driving it with browser automation is brittle, and risky with real registrar credentials in the loop.
- **No safe credentials.** Handing an autonomous system a registrar password gives it the zone, mail routing, transfer locks and billing. Do not do this.
- **No durable state.** Propagation outlives the session. Without a resumable, queryable resource the agent cannot pick up where it stopped.
- **No feedback.** A human squints at a help article and retries. An agent needs deterministic statuses and explicit error codes.

That is the autonomy cliff: a fully automated pipeline ending in "now open your registrar dashboard and paste
these records." Everything below is about removing it.

## Quickstart

Give a Claude Code agent the full domain toolset (twenty tools, streamable HTTP), or drive the same control
plane from a backend job. `domain` is the only required field; the edge target comes from the credential.

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer $CUSTOMDOMAIN_API_KEY"

curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com"}'
```

Unknown fields are rejected with a `400`, so do not invent keys. Create is idempotent per application plus
domain: re-posting replays the existing connection with a `200`. Signing up at
[app.customdomain.ai/signup](https://app.customdomain.ai/signup) needs no card: every workspace has a default
API key on the **API & SDK** page, and its first ten domains connect without one.

## MCP setup for Claude, Cursor, VS Code and GitHub Copilot

Every client points at the same endpoint, `https://mcp.customdomain.ai/mcp` (streamable HTTP, registry name
`ai.customdomain/mcp`). It accepts a Bearer credential in a header: an API key (`cd_live_...`) from the
console's **API & SDK** page, or an application's `APPLICATION_ID:CLIENT_SECRET` pair. A client that follows
the server's `401` to its OAuth metadata can instead sign a person in through the browser, and that person
approves the scopes the agent gets (`domains:read`, `domains:connect`, `domains:disconnect`,
`domains:purchase`). Buying a domain through MCP is off on the hosted service, whatever the scope. Keep keys in an
environment variable or a secret store, never in a file you commit.

**Claude Code**:

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer $CUSTOMDOMAIN_API_KEY"
```

**Claude Desktop and claude.ai**: open **Customize > Connectors**, choose **Add custom connector** and enter
`https://mcp.customdomain.ai/mcp`. To use a key instead of a sign-in, choose **No sign-in** and add
`Bearer cd_live_...` as the `authorization` value under **Request headers** (a beta that not every
organization has yet).

**Cursor**: merge into your user-level `~/.cursor/mcp.json`, not a project file that gets committed:

```json
{
  "mcpServers": {
    "customdomain": {
      "url": "https://mcp.customdomain.ai/mcp",
      "headers": { "Authorization": "Bearer cd_live_YOUR_API_KEY" }
    }
  }
}
```

**VS Code**: add the server to `.vscode/mcp.json` (or use the one-click install badge in
[customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp)). The entry holds no key; the
server's `401` points VS Code to the sign-in, and the customdomain-mcp README shows the key variant:

```json
{
  "servers": {
    "customdomain": { "type": "http", "url": "https://mcp.customdomain.ai/mcp" }
  }
}
```

**GitHub Copilot cloud agent** (the coding agent): it does not support remote MCP servers that sign in with
OAuth, so the credential goes in a header. In the repository's **Settings > Copilot > MCP servers**, save:

```json
{
  "mcpServers": {
    "customdomain": {
      "type": "http",
      "url": "https://mcp.customdomain.ai/mcp",
      "headers": { "Authorization": "$COPILOT_MCP_CUSTOMDOMAIN_AUTH" },
      "tools": ["discover-provider", "connect-domain", "check-connection-status", "diagnose-connection", "recheck-connection"]
    }
  }
}
```

Then add an Agents secret `COPILOT_MCP_CUSTOMDOMAIN_AUTH` whose value is `Bearer ` followed by the API key or
the `APPLICATION_ID:CLIENT_SECRET` pair. Listing tools keeps the agent away from `disconnect-domain`, which takes
a domain offline, and from the nameserver moves; `"tools": ["*"]` enables every tool without asking.

## In this guide

- **Explains the four mechanics** of a connection: routing records, proof of control, TLS issuance, propagation. [docs/01](docs/01-agents-that-ship-websites-need-domains.md)
- **Documents the twenty MCP tools**, their arguments and scopes, and config for Claude, Cursor, VS Code and GitHub Copilot. [docs/02](docs/02-mcp-server-for-domains.md)
- **Walks the REST flow end to end** with real request and response shapes and failure handling. [docs/03](docs/03-programmatic-domain-connection-api.md)
- **Argues the security model**: scoped credentials, human approval points, no registrar passwords. [docs/04](docs/04-security-for-agent-driven-dns.md)

## How it works

Records have to reach a DNS zone the agent does not control. There are exactly three rails for that, and every
domain-connection product reduces to them. The split below is the live census at `GET
https://api.customdomain.ai/v1/providers/census`, counted 2026-10-30.

| Rail | Providers (of 189) | What the human does |
|---|---|---|
| One-click provider authorization | 7 (6 provider OAuth, 1 provider-hosted Domain Connect) | Approves a scoped change inside their own DNS provider session |
| Scoped API token | 66 | Pastes one DNS-scoped token, used once and discarded by default |
| Guided manual with automatic verification | 116 | Copies the exact records. A poller waits for them to resolve |

Guided manual is the largest group, for every vendor in this category, so the quality of that path matters more
than a coverage headline. Control of the zone is proven by whichever rail wrote the records: no separate TXT
challenge to relay, no verification step to poll on its own
([connections](https://docs.customdomain.ai/docs/concepts/connections)).

## Connection states and error codes

A connection is a durable resource, so a session that dies mid-flow loses nothing. Four states, and an agent
should branch on all four.

| Status | Meaning | The agent's move |
|---|---|---|
| `pending` | Created; records not written or not yet observed | Surface the next human action if the rail needs one |
| `propagating` | Records written, being checked against public DNS by value | Wait. Poll, or subscribe to `connection.live` |
| `live` | Every record resolves to its intended value. On Growth and up, the CustomDomain™ edge serves HTTPS | Report success, store the connection id |
| `failed` | Records a rail wrote never resolved within 24 hours, or the rail rejected the write | Read `error_code`, fix, `POST /v1/connections/{id}:recheck` |

Only a rail can time out: 24 hours in `propagating` without the records resolving is `failed` with
`propagation_timeout`. A manual connection never fails on its own. After 72 hours in `pending` it stays `pending`,
carries `error_code: setup_incomplete` as a diagnosis, and is re-checked every six hours until the records
appear. Codes seen in practice: `propagation_timeout`, `setup_incomplete`, `apex_not_supported`,
`domain_already_connected`, `dns_write_failed`. `:recheck` re-resolves every record in-request and re-bases the
clock, so a retry does not immediately re-fail.

## Where this loses

Plan prices and allowances are in the facts block below. Entri is the established vendor here and the comparison
is mixed. Read 2026-08-19, its entry tier is $249/mo for 600 connections a year, with no free tier. Where Entri
is ahead: upstream one-click coverage. In [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates),
goentri.com ships 77 templates and customdomain.ai ships 18 (counted 2026-09-26), the two largest sets. That gap
is real and it is theirs.

## Limits and known gaps

- **No tool accepts a raw DNS record**, by design. Record values are computed server-side from vetted templates, so a prompt-injected agent cannot write a hostile record. The one narrow exception is `PUT /v1/connections/{id}/records`, for sets that cannot be derived from the domain (Amazon SES DKIM tokens). If you need arbitrary DNS writes, this is the wrong tool.
- **SSO and SCIM are not built at all**, and no tier grants them.
- **Purchases through MCP are off on the hosted service.** `create-domain-order` is built, but the direct purchase route it calls is switched off there, so it answers `success: false`. Buy in the console, then connect the domain over MCP or REST.
- **Drift webhooks** (`domain.record_missing`, `domain.record_restored`) are produced by the monitor sweep but not delivered unless a deployment enables them, which the hosted default does not. Use `POST /v1/monitor:check` instead.

<!-- facts:start -->
## Facts and sources

Read from the live endpoints on 2026-10-30. Every number in this README traces to a row here or to a source named
where it is used. If one does not, open an issue with the file and line.

| Fact | Value | Source |
|---|---|---|
| DNS providers known | 189 | [`/v1/providers/census`](https://api.customdomain.ai/v1/providers/census) |
| Automatic path | 73: 66 by scoped API key, 6 by provider sign-in (OAuth), 1 by one-click setup (Domain Connect) | census `mode` |
| Exact records and provider steps | 116 | census `mode` |
| Going live | No TXT ownership record. A domain goes `live` when every record resolves. Statuses: `pending`, `propagating`, `live`, `failed` | [Connections](https://docs.customdomain.ai/docs/concepts/connections) |
| HTTPS on the CustomDomain™ edge | Growth and up. On Starter and Startup, domains point at your servers and you serve TLS | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Plans | Starter $10 a month, 10 domains a year, hard cap. Startup $149 a month, 600 a year. Growth $649 a month, 600 a year. Premium and Enterprise by contract, 12,000 a year. 14 day trial on Starter, Startup and Growth | [`/v1/plans`](https://api.customdomain.ai/v1/plans) |
| Free to start | A new workspace connects its first ten domains free. No card needed to sign up | [Plans and quotas](https://docs.customdomain.ai/docs/billing/plans-and-quotas) |
| REST API | 163 operations on 140 paths, OpenAPI 3.1.0 | [`/v1/openapi.json`](https://api.customdomain.ai/v1/openapi.json) |
| MCP server | `customdomain-mcp` 0.6.0, 20 tools. Buying a domain through MCP is off on the hosted service | [`mcp.json`](https://mcp.customdomain.ai/.well-known/agent/mcp.json) |

Check the provider split yourself:

```sh
curl -s https://api.customdomain.ai/v1/providers/census \
  | python3 -c 'import json,sys,collections; d=json.load(sys.stdin); print(d["count"], collections.Counter(p["mode"] for p in d["providers"]))'
# 189 Counter({'manual': 116, 'api': 66, 'oauth': 6, 'dc': 1})
```
<!-- facts:end -->

## Related

- [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp): the hosted MCP server, its twenty tools and client configs
- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders): the field guide for the publish flow an agent's app usually ends in
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, including other domain MCP servers

## Support and license

Problem framing draws on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0).

Questions go to [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions) and corrections to
[this repository's issues](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents/issues). Account and billing:
connect@customdomain.ai. Report security issues privately to security@customdomain.ai, never in a public issue
([policy](https://app.customdomain.ai/security)).

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
