# Connect a Domain for AI Agents

AI agents can already generate a codebase, provision infrastructure, and deploy a working app in minutes. Then they stall at the last step: connecting a real custom domain. This repository explains how domain registration, DNS configuration, control verification, and TLS issuance actually work, and shows the practical ways to let an agent finish that step programmatically. It is maintained by [CustomDomain](https://customdomain.ai), which operates a [hosted MCP server](https://customdomain.ai/mcp-server) and a [REST API](https://customdomain.ai/custom-domain-api) built for exactly this workflow.

## What this repository is

A field guide for people building agent products that ship websites and apps: site-generating agents, coding agents with a deploy step, and workflow agents that spin up landing pages, stores, and docs sites. It covers the problem space first (DNS, control proof, TLS, propagation), then the ways records can get into a customer's DNS zone, then the concrete implementation paths (widget, REST API, MCP tools). Deep dives live in [docs/](docs/):

| Deep dive | What it covers |
|---|---|
| [Agents that ship websites need domains](docs/01-agents-that-ship-websites-need-domains.md) | The agent-provisions-everything pattern and exactly where the domain step breaks it |
| [An MCP server for domains](docs/02-mcp-server-for-domains.md) | The hosted MCP tool surface: the twelve tools, their arguments, and config snippets for Claude and Cursor |
| [Programmatic domain connection over REST](docs/03-programmatic-domain-connection-api.md) | The full API flow end to end, real request and response shapes, failure handling, and an agent tool definition |
| [Security for agent-driven DNS](docs/04-security-for-agent-driven-dns.md) | Scoped credentials, human approval points, and why agents should never hold registrar passwords |

## The problem: agents ship everything except the domain

An agent that builds and deploys a site produces a preview URL on a platform subdomain. The user's next message is predictable: "make it live on acmebakery.com." At that moment the agent needs three things to happen in someone else's infrastructure: routing records written into a DNS zone it does not control, a demonstration that the user actually controls that zone, and a TLS certificate for the hostname. Then it has to wait for resolvers worldwide to return the new answers.

Humans already struggle here. The [knowledge base of the Domain Connect project](https://github.com/Domain-Connect/knowledge-base) (published under CC0 1.0) reports that approximately 50% of users who attempt manual DNS configuration fail and abandon the process, and documents one major productivity suite whose email onboarding requires 7 to 15 hand-created records and 16 help sites, 10 of them registrar-specific. Those users had already paid. They simply could not translate "add a CNAME" into the right clicks at their particular provider.

For an agent the failure mode is worse, because every workaround available to a human is unavailable or dangerous for an agent:

- **No hands.** The registrar's DNS panel is a browser UI, often behind 2FA. Driving it with browser automation is brittle and, with real registrar credentials in the loop, genuinely risky.
- **No safe credentials.** The obvious shortcut, "just give the agent my registrar password," hands an autonomous system full control of the zone, email routing, transfer locks, and billing. [Don't do this.](docs/04-security-for-agent-driven-dns.md)
- **No durable state.** DNS propagation can outlive an agent session. Without a resumable, queryable state machine, the agent cannot pick up where it left off.
- **No feedback.** A human squints at a help article and retries. An agent needs deterministic statuses, explicit error codes, and webhooks or pollable endpoints.

The result is the autonomy cliff: a fully automated build pipeline that ends with "now open your registrar dashboard and paste these records." The rest of this README is about removing that cliff.

## How domain connection actually works

Four mechanisms, in order. If you internalize these, every provider quirk becomes explainable.

### 1. Routing: DNS records point the name at your edge

A subdomain routes with a CNAME. A zone apex (the bare domain) cannot legally carry a CNAME alongside its other records, so it needs an A/AAAA record or a provider-specific ALIAS/ANAME that flattens to one:

```dns
; subdomain: CNAME to the platform edge
www.acmebakery.com.    3600  IN  CNAME  edge.customdomain.ai.

; apex: ALIAS/ANAME where the provider supports it, otherwise A
acmebakery.com.        3600  IN  A      203.0.113.10
```

CustomDomain's record set carries a provider-agnostic `APEXCNAME` type for this case, realized to the host's native ALIAS, ANAME, or flattened CNAME on write, and rendered as `ALIAS` in manual instructions. Providers that cannot host any CNAME-like record at the zone root are caught before a connection is created: `POST /v1/domains:check` returns `apex_supported: false` with an `apex_message`, rather than letting the connection fail at apply time.

Getting these exactly right, at the right provider, in the right zone, is the step that generates most support tickets. See the plain-language walkthrough at [How to set up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain) and the [custom domain vs subdomain glossary entry](https://customdomain.ai/glossary/custom-domain-vs-subdomain) for why a platform subdomain is not a substitute.

### 2. Proof: the rail is the proof

Before a platform serves traffic for a hostname, it must establish that the requesting user controls that DNS zone. Many platforms do this with a separate scoped TXT challenge. CustomDomain does not, and it is worth understanding why, because it changes what an agent has to orchestrate.

Control is proven by whatever mechanism wrote the records:

- An **OAuth authorization** at the DNS provider. Only someone who can log into the zone can grant it.
- A **one-click template apply** at a Domain Connect provider. Same argument, provider-hosted.
- A **scoped provider API token** the customer created and supplied.
- In the **manual** flow, the records appearing in public DNS is itself the proof. Nobody else can put a CNAME on the customer's hostname.

The practical consequence for an agent: there is no verification step to poll separately, and no `_challenge` record to relay to a user. There is one record set, and either it resolves or it does not. If you are reading older copy of ours that describes a TXT ownership challenge, that copy was wrong. See [Connections](https://docs.customdomain.ai/docs/concepts/connections) for the authoritative description.

### 3. Security: TLS issuance and renewal

Once the records resolve, a certificate is issued for the hostname from a publicly trusted certificate authority and installed at a reverse-proxy edge that terminates TLS. On the hosted service, issuance happens on the next TLS handshake for that host, and renewal is scheduled automatically. Your platform never generates a CSR, never stores the private key, and never wakes up to an expired certificate. Hosts whose records do not resolve never reach the edge and so never get a certificate.

### 4. Waiting: propagation and TTLs

"Propagation" is not a push; it is caches expiring. Every resolver that previously looked up the name keeps its answer until the record's TTL runs out. Records written correctly on the first attempt, with sensible TTLs, resolve quickly. Records written wrong, then corrected, inherit the old TTL on the wrong answer, which is why manual setups "take 24 to 48 hours" in folklore and automated ones usually do not. When records are written through provider authorization, a domain is typically live in about 30 seconds. A background poller re-resolves every `pending` and `propagating` connection on a one-minute interval, with value checking: a CNAME must resolve to the expected target, an A or AAAA must contain the exact address.

## Three ways to get records into the customer's zone

Every domain connection product, and every homegrown flow, reduces to one of these. CustomDomain implements all of them across 63 catalogued DNS and registrar providers. 25 of the 63 have an automatic path; the remaining 38 use guided manual setup with automatic verification. The split below is the live census at `GET https://api.customdomain.ai/v1/providers/census`, counted 2026-08-19. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for the provider-level view.

| Method | Providers | How it works | User effort | Typical time to live |
|---|---|---|---|---|
| **One-click provider authorization** | 8 (6 OAuth into the provider, 2 provider-hosted Domain Connect) | The user approves a scoped change at their DNS provider; correct records are applied for them | One approval click | About 30 seconds |
| **API token** | 17 | The user pastes a scoped DNS API token from their provider; records are written through the provider's API | Create and paste one token | Minutes |
| **Guided manual with automatic verification** | 38 | The user is shown the exact records for their detected provider and a poller waits for them to resolve, then proceeds automatically | Copy and paste records | Minutes to hours, TTL dependent |

Be honest with yourself about that third row: it is 38 of 63, the largest group. The fallback chain is the point, not a footnote. Detection fingerprints the user's provider from the domain's existing DNS, offers the best available method, and degrades to guided manual for the long tail. The agent's job stays the same in all three cases: create the connection, surface the next required human action if there is one, and poll or subscribe until the status reaches `live`.

## The connection state machine

An agent needs to branch on state, so here is the complete vocabulary. There are exactly four values.

| Status | Meaning | Agent's move |
|---|---|---|
| `pending` | Created; records not yet written or not yet observed | Surface the next human action if the chosen rail needs one |
| `propagating` | Records were written by a rail and are being checked against public DNS | Wait. Poll, or subscribe to `connection.live` |
| `live` | Every desired record resolves to its intended value; the edge serves TLS for the host | Report success, store the connection id |
| `failed` | Records never appeared inside the window | Read `error_code`, surface the concrete fix, then `POST /v1/connections/{id}:recheck` |

The failure windows are asymmetric on purpose: 24 hours from `propagating` on an automatic rail (a write that succeeded should resolve quickly), and 72 hours from `pending` on a manual connection, which is long enough for a human to get around to it. A `failed` connection carries `error_code` and `error_message`. The codes an agent will actually see include `propagation_timeout` (records never resolved), `setup_incomplete` (manual records were never added), `apex_not_supported` (the provider cannot host a CNAME-like record at the zone root), `domain_already_connected`, and `dns_write_failed`. Both fields clear automatically when the connection next verifies, and `:recheck` on a `failed` connection re-enters it into verification with the timeout clock re-based, so a retry does not immediately re-fail.

Because the connection is a durable resource and `POST /v1/connections` is idempotent per application plus domain, a connection created in one agent session is resumed from any later session with a single GET, or by simply re-posting the same domain.

## Implementation paths: widget, API, or MCP

| Path | Best for | Integration shape |
|---|---|---|
| [Connect widget and SDK](https://customdomain.ai/connect-domain-widget) | Agent products with a web UI where the end user completes the connection themselves | Embed a prebuilt flow; the widget handles provider detection, authorization, and status. `customdomain-js` on npm, with `@customdomain/react` as the React wrapper |
| [REST API](https://customdomain.ai/custom-domain-api) | Backend orchestration, existing job queues, platforms that already run a provisioning pipeline | Connections, DNS records, TLS, monitoring, webhooks, registrar search and purchase. The served OpenAPI 3.1 spec at [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json) describes 67 paths and 79 operations (counted 2026-08-19) |
| [Hosted MCP server](https://customdomain.ai/mcp-server) | Agents that operate through tool calls: coding assistants, autonomous builders, chat-native products | Streamable HTTP at `mcp.customdomain.ai/mcp`, API-key or OAuth client-credentials auth, twelve tools. Listed in the official MCP registry as `ai.customdomain/mcp` |

A minimal REST call to start a connection. `domain` is the only required field, and the body is decoded with unknown fields rejected, so an extra key is a `400`. The application, and therefore the edge target, comes from the credential:

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com"}'
```

And the one-liner that gives a Claude Code agent the full domain toolset:

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer sk_live_YOUR_KEY"
```

The browser path, for products where the end user should drive the flow:

```bash
npm install customdomain-js     # or: npm install @customdomain/react
```

Mint a short-lived widget JWT on your server (never ship an API key to the browser), then call `window.customdomain.open({ applicationId, token })`. The token lifecycle is documented at [Widget tokens](https://docs.customdomain.ai/docs/authentication/widget-tokens) and the embed at [Installation and embed](https://docs.customdomain.ai/docs/widget-sdk/installation-and-embed).

Full walkthroughs: [REST flow end to end](docs/03-programmatic-domain-connection-api.md) and [the MCP tool surface](docs/02-mcp-server-for-domains.md). Complete reference lives at [docs.customdomain.ai](https://docs.customdomain.ai/docs).

## Decision guide

| Your situation | Start with |
|---|---|
| Users click "connect my domain" in your product's UI | Widget, with the API for status and webhooks |
| Your agent runs a provisioning pipeline server-side | REST API |
| Your agent is tool-calling (MCP-native, chat-driven) | MCP server |
| Users arrive without a domain at all | MCP or API registrar search and purchase, then auto-connect |
| You need human sign-off before money moves | All paths: purchases are fail-closed behind an authorization callback |

## What it costs, and where the free tier stops

Prices and quotas below come from `GET https://api.customdomain.ai/v1/plans`, read 2026-08-19. That endpoint is the source of truth; treat any prose that disagrees with it, including ours, as stale.

| Plan | Price | Domain connections included | Hard cap in the catalog? | Beyond Connect |
|---|---|---|---|---|
| Free | $0 | 10 per year (1 per month) | Yes | none |
| Startup | $149/mo | 600 per year (50 per month) | No | none |
| Growth | $649/mo | 600 per year (50 per month) | No | Power (reverse-proxy origin) and Secure (the `/v1/ssl*` certificate-management surface) |
| Premium | Contact sales | 12,000 per year | No | adds Monitor (preview) |
| Enterprise | Contact sales | 12,000 per year | No | adds white-label |

Two honest notes on that table. First, Free is the only tier carrying a hard cap, and even that needs a deployment-level enforcement switch that the hosted service currently leaves off, so no tier is refusing connections today. Paid tiers are never hard-capped by design: the refusal would land on your end user mid-connect rather than on your admin, so overage is recorded and surfaced instead. Read `quota_enforced` on `GET /v1/billing/usage` if you need to know whether a `402` is actually possible right now; `quota_state: at_limit` means "over the line," not "blocked." Second, "free tier" does not mean the whole product. Automatic TLS is included everywhere, but the `/v1/ssl*` management surface (import your own certificate, force a renew, read provisioning detail) and the `/v1/connections/{id}/power*` reverse-proxy surface both return `402 plan_upgrade_required` below Growth. If you have seen us write "pricing starts at $0 with the full product," that was an overclaim.

## How this compares

Worth stating plainly, because anyone searching "domain connection for AI agents" is evaluating options rather than reading a brochure.

**Against Entri.** Entri is the established product in this category and the honest comparison is mixed. Fetched 2026-08-19, Entri's Startup tier is $249/mo for 600 domains a year, and Growth, Premium and Enterprise are all "Talk to Sales"; there is no free tier. CustomDomain's Startup is $149/mo for the same 600 a year, Growth is a published $649/mo, and there is a free tier at 10 connections a year. Where Entri is ahead: template coverage in the open Domain Connect ecosystem. In the upstream [Domain-Connect/Templates](https://github.com/Domain-Connect/Templates) repository, Entri (goentri.com) ships 77 templates and CustomDomain ships 18, second place out of 696 provider domains. More upstream templates means more providers where a one-click apply is possible without any bilateral integration. If provider-hosted one-click breadth is your deciding factor, that gap is real and it is theirs.

**Against building it yourself.** The DNS and ACME parts are not the hard part. The hard part is the long tail: 63 provider fingerprints, per-provider apex behavior, SPF merge rather than SPF replace, CAA records that block issuance, a poller with value checking rather than "did the name resolve at all," and a support path for the 38 providers where a human still pastes records. If you have one or two DNS providers to support because you also sell the domains, build it. If your users arrive with whatever registrar they already had, do not.

**Where a managed layer is the wrong call.** If your product needs arbitrary DNS records written on the customer's behalf, this is not the tool. No API or MCP tool here accepts a raw record as input, by design (see [the security deep dive](docs/04-security-for-agent-driven-dns.md)). The one narrow exception is `PUT /v1/connections/{id}/records`, for record sets that cannot be derived from the domain, such as Amazon SES verification tokens, and a connection whose records came in that way cannot use the Domain Connect rail afterward.

## FAQ

**Can the agent buy the domain too, not just connect it?**
Yes. Registrar search and purchase are part of both the REST API (`GET /v1/registrar/search`, `POST /v1/registrar/quote`, `POST /v1/registrar/checkout`, `POST /v1/registrar/fulfill`) and the MCP server (`search-domain-availability`, `create-domain-order`, `check-order-status`). The flow is ordered so you cannot pay for a domain you do not get: authorize the card, register the domain, and only then capture. If registration fails, the authorization is released. A domain bought this way auto-connects, because its zone is created pointing at the edge from the first second. Purchases are fail-closed: MCP orders require an explicit purchase-authorization callback, so an agent cannot spend money on its own.

**Does the agent need my customer's registrar login?**
No, and it never should. One-click provider authorization keeps credentials at the DNS provider, where the user approves a scoped change inside their own authenticated session. The API-token path uses a token limited to DNS, written once and then discarded unless you explicitly opt into remembering it. The agent itself only ever holds a scoped, revocable CustomDomain key. The reasoning is laid out in [Security for agent-driven DNS](docs/04-security-for-agent-driven-dns.md).

**How long until the domain is actually live?**
Through provider authorization, typically about 30 seconds. Through guided manual, it depends on how fast the user pastes records and on existing TTLs. The poller runs every minute and flips the status the moment every record resolves.

**What happens if the agent session ends mid-connection?**
Nothing is lost. A connection is a durable resource, and `POST /v1/connections` is idempotent per application plus domain, so re-posting the same domain replays the existing connection with a `200` rather than creating a duplicate. Any later session, or any webhook consumer, can `GET /v1/connections/{id}` and resume.

**What should the agent do when a connection fails?**
Read `error_code`, not the message text. `setup_incomplete` means the human never pasted the records, so re-render them and ask again. `propagation_timeout` means they were written but never resolved, which usually means they were written in the wrong zone or overwritten. `apex_not_supported` means the provider cannot serve this hostname at all and the user needs a subdomain or a different DNS host. Then call `POST /v1/connections/{id}:recheck`, which resolves every record in-request, returns a per-record verdict, and re-enters a failed connection into verification.

**What if the user's DNS provider is not supported for automation?**
Then it is one of the 38, and the flow degrades to guided manual: exact records rendered for the detected provider, with automatic verification polling. The agent gets the same statuses and the same webhooks; only the human step changes.

**Can a compromised or prompt-injected agent write hostile DNS records?**
Not through the MCP server: there is deliberately no tool that accepts a DNS record. Record values are computed by the control plane from vetted templates, so the worst outcome of a bad tool call is a failed connection, not a hijacked zone. Details in [the security deep dive](docs/04-security-for-agent-driven-dns.md).

**Is there anything advertised that is not actually built?**
Yes, and it is worth knowing before you plan around it. Delegated agent tokens (short-lived, scope-enforced tokens for a human delegating access to an agent) are built and tested but not enabled on the hosted service. Drift webhooks (`domain.record_missing` and `domain.record_restored`) are produced by the Monitor sweep but not delivered unless the deployment enables them, which the hosted default does not. SSO and SCIM are not built and no tier grants them.

## Attribution

Parts of the problem framing in this repository draw on the [knowledge base of the Domain Connect project](https://github.com/Domain-Connect/knowledge-base), published under CC0 1.0. The Domain Connect protocol is an open standard maintained by a community of developers across multiple companies and is referenced here as third-party prior art, not as a product name.

## About CustomDomain

This repository is maintained by [CustomDomain](https://customdomain.ai), a product of EverJust Company. CustomDomain lets a platform's users connect their own domain with the least effort their DNS provider allows: 63 catalogued DNS and registrar providers, 25 of them with an automatic path (8 one-click, 17 through a scoped API token) and 38 with guided manual setup and automatic verification, plus automatic TLS issuance and renewal, on a managed control plane with a reverse-proxy edge and strict multi-tenant isolation. Surfaces include an [embeddable connect widget](https://customdomain.ai/connect-domain-widget), a [full REST API](https://customdomain.ai/custom-domain-api), and a [hosted MCP server for AI agents](https://customdomain.ai/mcp-server). The free tier is $0 for 10 domain connections a year.

- [CustomDomain for AI agents](https://customdomain.ai/for/ai-agents)
- [Custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)
- [For site builders](https://customdomain.ai/for/site-builders) and [for agencies and white-label](https://customdomain.ai/for/agencies-white-label)
- [Developer docs](https://docs.customdomain.ai/docs) and the [agent index](https://docs.customdomain.ai/docs/llms.txt)
- [MCP client examples on GitHub](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp)
- [Create a free account](https://app.customdomain.ai/signup)

Sibling field guides in this organization: [for website builders](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-website-builders), [for agencies](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-agencies), [for email platforms](https://github.com/CUSTOM-DOMAIN-APP/connect-domain-for-email-platforms), and the curated [awesome-custom-domains](https://github.com/CUSTOM-DOMAIN-APP/awesome-custom-domains) list.
