# Custom Domain for AI Agents

Custom domains for AI agents — a hosted MCP server and REST API that let an agent search, buy and connect a
domain

**Status:** Maintained guide · every number re-verified against the live API on 2026-09-04 · public

[![docs](https://img.shields.io/badge/docs-docs.customdomain.ai-1c1917?style=flat)](https://docs.customdomain.ai/docs)
[![license](https://img.shields.io/badge/license-MIT-1c1917?style=flat)](./LICENSE)

|  |  |
|---|---|
| **What it is** | Implementation guide for connecting a real domain from inside an AI agent |
| **Who it's for** | Teams building site-generating agents, coding agents with a deploy step, workflow agents |
| **Live at** | [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents) · docs at [docs.customdomain.ai](https://docs.customdomain.ai/docs) |
| **Stack** | Markdown guide · hosted MCP server over streamable HTTP · REST API, OpenAPI 3.1 |
| **Status** | Maintained · 63 providers and 5 plans re-counted from the live API 2026-09-04 |

An agent can generate a codebase, provision infrastructure and deploy a working app in minutes, then stall on
one step: putting it on a domain the customer already owns. This repository explains what that step actually
requires (DNS records, proof of control, TLS, propagation) and shows the three ways an agent can finish it
without ever holding a registrar password. It is maintained by [Custom Domain](https://customdomain.ai), which
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

Give a Claude Code agent the full domain toolset (twelve tools, streamable HTTP), or drive the same control
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
domain: re-posting replays the existing connection with a `200`. Free-tier keys:
[app.customdomain.ai/signup](https://app.customdomain.ai/signup).

## What it does

- **Explains the four mechanics** of a connection: routing records, proof of control, TLS issuance, propagation. [docs/01](docs/01-agents-that-ship-websites-need-domains.md)
- **Documents the twelve MCP tools**, their arguments, and config for Claude and Cursor. [docs/02](docs/02-mcp-server-for-domains.md)
- **Walks the REST flow end to end** with real request and response shapes and failure handling. [docs/03](docs/03-programmatic-domain-connection-api.md)
- **Argues the security model**: scoped credentials, human approval points, no registrar passwords. [docs/04](docs/04-security-for-agent-driven-dns.md)
- **Publishes the numbers with their sources**, so you can re-count them yourself before trusting them.

## How it works

Records have to reach a DNS zone the agent does not control. There are exactly three rails for that, and every
domain-connection product reduces to them. The split below is the live census at `GET
https://api.customdomain.ai/v1/providers/census`, counted 2026-09-04.

| Rail | Providers (of 63) | What the human does | Typical time to live |
|---|---|---|---|
| One-click provider authorization | 8 (6 provider OAuth, 2 provider-hosted Domain Connect) | Approves a scoped change inside their own DNS provider session | About 30 seconds |
| Scoped API token | 17 | Pastes one DNS-scoped token, used once and discarded by default | Minutes |
| Guided manual with automatic verification | 38 | Copies the exact records; a poller waits for them to resolve | TTL dependent |

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
| `failed` | Records never appeared inside the window | Read `error_code`, fix, `POST /v1/connections/{id}:recheck` |

Windows are asymmetric on purpose: 24 hours from `propagating` on an automatic rail, 72 hours from `pending` on
a manual one. Codes seen in practice: `propagation_timeout`, `setup_incomplete`, `apex_not_supported`,
`domain_already_connected`, `dns_write_failed`. `:recheck` re-resolves every record in-request and re-bases the
clock, so a retry does not immediately re-fail.

## Repository layout

```text
.
├── README.md  # the problem, the three rails, the state machine, pricing
├── AGENTS.md  # machine-readable brief for coding agents working in this repo
├── docs/      # 01 where the agent pipeline breaks · 02 the twelve MCP tools
│              # 03 the REST flow end to end · 04 credential scoping and approval gates
└── LICENSE    # MIT
```

Markdown only, no build step. Surfaces referenced from here: MCP at `mcp.customdomain.ai/mcp` (registry id
`ai.customdomain/mcp`), REST at `api.customdomain.ai` (OpenAPI 3.1: 67 paths, 79 operations, counted
2026-09-04), and the browser widget `customdomain-js` on npm.

## Pricing, and where this loses

Read from `GET https://api.customdomain.ai/v1/plans` on 2026-09-04.

| Plan | Price | Connections included | Beyond Connect |
|---|---|---|---|
| Free | $0 | 10/yr, 1/month, hard capped | none |
| Startup | $149/mo | 600/yr, 50/month | none |
| Growth | $649/mo | 600/yr, 50/month, metered overage | Power (reverse proxy) and Secure (`/v1/ssl*`) |
| Premium / Enterprise | Contact sales | 12,000/yr | adds Monitor, then white-label |

Entri is the established vendor here and the comparison is mixed. Read 2026-08-19, its entry tier is $249/mo for
the same 600 connections a year, with no free tier. Where Entri is ahead: upstream one-click coverage. Across
696 provider domains in [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates), goentri.com
ships 77 templates and customdomain.ai ships 18. That gap is real and it is theirs.

## Limits and known gaps

- **No tool accepts a raw DNS record**, by design. Record values are computed server-side from vetted templates, so a prompt-injected agent cannot write a hostile record. The one narrow exception is `PUT /v1/connections/{id}/records`, for sets that cannot be derived from the domain (Amazon SES DKIM tokens). If you need arbitrary DNS writes, this is the wrong tool.
- **Delegated agent tokens** are built and tested but not enabled on the hosted service, and **SSO and SCIM are not built at all**. No tier grants them.
- **Drift webhooks** (`domain.record_missing`, `domain.record_restored`) are produced by the monitor sweep but not delivered unless a deployment enables them, which the hosted default does not. Use `POST /v1/monitor:check` instead.

## Corrections

Every number here traces to a live endpoint or a public repository, named at the point of use. If one does not,
that is a bug: open an issue with the file and line.

Sibling guides: [for agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies) · [for website
builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders) · [for email
platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms) ·
[awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains), which lists the
alternatives to this one. Problem framing draws on the [Domain Connect knowledge
base](https://github.com/Domain-Connect/knowledge-base) (CC0 1.0); Domain Connect is an open standard maintained
by a community across multiple companies, referenced here as prior art, not as a product name.

## License

[MIT](./LICENSE) © CustomDomain.ai, a product of EverJust Company.
