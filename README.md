# Custom domains for AI agents: the CustomDomain™ field guide

Custom domains for AI agents: a hosted MCP server and REST API that let an agent search for a domain, connect it to
an app, verify it and get HTTPS, with DNS, verification and TLS handled by the control plane.

**Status:** Maintained guide · live API numbers re-verified on 2026-10-20 · public

[![docs](https://img.shields.io/badge/docs-docs.customdomain.ai-1c1917?style=flat)](https://docs.customdomain.ai/docs)
[![license](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](./LICENSE)

[Website](https://customdomain.ai) · [Docs](https://docs.customdomain.ai/docs) · [Console](https://app.customdomain.ai) · [MCP server](https://customdomain.ai/mcp-server) · [For AI agents](https://customdomain.ai/for/ai-agents)

|  |  |
|---|---|
| **What it is** | Implementation guide for connecting a real domain from inside an AI agent |
| **Who it's for** | Teams building site-generating agents, coding agents with a deploy step, workflow agents |
| **Live at** | [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents) · docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| **Stack** | Markdown guide · hosted MCP server (version 0.5.0) over streamable HTTP · REST API, OpenAPI 3.1 |
| **Status** | Maintained · 178 providers, 5 plans and 17 MCP tools re-counted from the live service 2026-10-20 |

An agent can generate a codebase, provision infrastructure and deploy a working app in minutes, then stall on
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

Give a Claude Code agent the full domain toolset (seventeen tools, streamable HTTP), or drive the same control
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
`domains:purchase`). Keep keys in an environment variable or a secret store, never in a file you commit.

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
a domain offline; `"tools": ["*"]` enables every tool without asking.

## What it does

- **Explains the four mechanics** of a connection: routing records, proof of control, TLS issuance, propagation. [docs/01](docs/01-agents-that-ship-websites-need-domains.md)
- **Documents the seventeen MCP tools**, their arguments and scopes, and config for Claude, Cursor, VS Code and GitHub Copilot. [docs/02](docs/02-mcp-server-for-domains.md)
- **Walks the REST flow end to end** with real request and response shapes and failure handling. [docs/03](docs/03-programmatic-domain-connection-api.md)
- **Argues the security model**: scoped credentials, human approval points, no registrar passwords. [docs/04](docs/04-security-for-agent-driven-dns.md)
- **Publishes the numbers with their sources**, so you can re-count them yourself before trusting them.

## How it works

Records have to reach a DNS zone the agent does not control. There are exactly three rails for that, and every
domain-connection product reduces to them. The split below is the live census at `GET
https://api.customdomain.ai/v1/providers/census`, counted 2026-10-20.

| Rail | Providers (of 178) | What the human does | Typical time to live |
|---|---|---|---|
| One-click provider authorization | 7 (6 provider OAuth, 1 provider-hosted Domain Connect) | Approves a scoped change inside their own DNS provider session | About 30 seconds |
| Scoped API token | 55 | Pastes one DNS-scoped token, used once and discarded by default | Minutes |
| Guided manual with automatic verification | 116 | Copies the exact records; a poller waits for them to resolve | TTL dependent |

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
| `live` | Every record resolves to its intended value; the edge serves TLS | Report success, store the connection id |
| `failed` | Records a rail wrote never resolved within 24 hours, or the rail rejected the write | Read `error_code`, fix, `POST /v1/connections/{id}:recheck` |

Only a rail can time out: 24 hours in `propagating` without the records resolving is `failed` with
`propagation_timeout`. A manual connection never fails on its own. After 72 hours in `pending` it stays `pending`,
carries `error_code: setup_incomplete` as a diagnosis, and is re-checked every six hours until the records
appear. Codes seen in practice: `propagation_timeout`, `setup_incomplete`, `apex_not_supported`,
`domain_already_connected`, `dns_write_failed`. `:recheck` re-resolves every record in-request and re-bases the
clock, so a retry does not immediately re-fail.

## Repository layout

```text
.
├── README.md  # the problem, the three rails, the state machine, pricing
├── AGENTS.md  # machine-readable brief for coding agents working in this repo
├── docs/      # 01 where the agent pipeline breaks · 02 the seventeen MCP tools
│              # 03 the REST flow end to end · 04 credential scoping and approval gates
└── LICENSE    # MIT
```

Markdown only, no build step. Surfaces referenced from here: MCP at `mcp.customdomain.ai/mcp` (registry id
`ai.customdomain/mcp`), REST at `api.customdomain.ai` (OpenAPI 3.1: 129 paths, 152 operations, counted
2026-10-20), and the browser widget `customdomain-js` on npm.

## Pricing, and where this loses

Read from `GET https://api.customdomain.ai/v1/plans` on 2026-10-20.

| Plan | Price | Connections included | Beyond Connect |
|---|---|---|---|
| Starter | $10/mo | 10/yr, 1/month, hard capped | none |
| Startup | $149/mo | 600/yr, 50/month | none |
| Growth | $649/mo | 600/yr, 50/month, metered overage | Power (reverse proxy) and Secure (`/v1/ssl*`) |
| Premium / Enterprise | Contact sales | 12,000/yr | adds Monitor, then white-label |

The self-serve plans start with a 14 day free trial, and a new workspace connects its first ten domains
without a card. Starter replaced the Free plan on 2026-09-29.

Entri is the established vendor here and the comparison is mixed. Read 2026-08-19, its entry tier is $249/mo for
the same 600 connections a year, with no free tier. Where Entri is ahead: upstream one-click coverage. In
[Domain-Connect/Templates](https://github.com/Domain-Connect/Templates), goentri.com ships 77 templates and
customdomain.ai ships 18 (counted 2026-09-26), the two largest sets. That gap is real and it is theirs.

## Limits and known gaps

- **No tool accepts a raw DNS record**, by design. Record values are computed server-side from vetted templates, so a prompt-injected agent cannot write a hostile record. The one narrow exception is `PUT /v1/connections/{id}/records`, for sets that cannot be derived from the domain (Amazon SES DKIM tokens). If you need arbitrary DNS writes, this is the wrong tool.
- **SSO and SCIM are not built at all**, and no tier grants them.
- **Purchases through MCP are off on the hosted service.** `create-domain-order` is built, but the direct purchase route it calls is switched off there, so it answers `success: false`. Buy in the console, then connect the domain over MCP or REST.
- **Drift webhooks** (`domain.record_missing`, `domain.record_restored`) are produced by the monitor sweep but not delivered unless a deployment enables them, which the hosted default does not. Use `POST /v1/monitor:check` instead.

## Corrections

Every number here traces to a live endpoint or a public repository, named at the point of use. If one does not,
that is a bug: open an issue with the file and line.

Problem framing draws on the [Domain Connect knowledge base](https://github.com/Domain-Connect/knowledge-base)
(CC0 1.0); Domain Connect is an open standard maintained by a community across multiple companies, referenced
here as prior art, not as a product name.

## Related

- [connect-domain-for-agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies): custom domains for agencies, the field guide
- [connect-domain-for-email-platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms): custom sending domains for email platforms, the field guide
- [connect-domain-for-website-builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders): custom domains for website builders, the field guide
- [customdomain-mcp](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp): the hosted MCP server, its seventeen tools and client configs
- [customdomain-sdk](https://github.com/CUSTOM-DOMAIN-APP/customdomain-sdk): the browser SDK `customdomain-js` and the React wrapper `@customdomain/react`
- [docs](https://github.com/CUSTOM-DOMAIN-APP/docs): the CustomDomain™ documentation source, rendered at [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains): the curated list of the category, including the alternatives to this product

## Support

- **Docs:** [docs.customdomain.ai](https://docs.customdomain.ai/docs)
- **Questions and ideas:** [GitHub Discussions](https://github.com/CUSTOM-DOMAIN-APP/docs/discussions)
- **Bugs and corrections:** [open an issue](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-ai-agents/issues) on this repository
- **Service status:** [status.customdomain.ai](https://status.customdomain.ai)
- **Account and billing:** connect@customdomain.ai
- **Security:** report privately to security@customdomain.ai, never in a public issue. Policy: [app.customdomain.ai/security](https://app.customdomain.ai/security)

## License

[MIT](./LICENSE). CustomDomain™ is a product of EverJust Company.
