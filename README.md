# Connect a Domain for AI Agents

AI agents can already generate a codebase, provision infrastructure, and deploy a working app in minutes. Then they stall at the last step: connecting a real custom domain. This repository explains how domain registration, DNS configuration, ownership verification, and TLS issuance actually work, and shows the practical ways to let an agent finish that step programmatically. It is maintained by [Custom Domain](https://customdomain.ai), which operates a [hosted MCP server](https://customdomain.ai/mcp-server) and a [REST API](https://customdomain.ai/custom-domain-api) built for exactly this workflow.

## What this repository is

A field guide for people building agent products that ship websites and apps: site-generating agents, coding agents with a deploy step, and workflow agents that spin up landing pages, stores, and docs sites. It covers the problem space first (DNS, verification, TLS, propagation), then the three ways records can get into a customer's DNS zone, then the concrete implementation paths (widget, REST API, MCP tools). Deep dives live in [docs/](docs/):

| Deep dive | What it covers |
|---|---|
| [Agents that ship websites need domains](docs/01-agents-that-ship-websites-need-domains.md) | The agent-provisions-everything pattern and exactly where the domain step breaks it |
| [An MCP server for domains](docs/02-mcp-server-for-domains.md) | The hosted MCP tool surface: search, register, connect, verify, monitor. Config snippets for Claude and Cursor |
| [Programmatic domain connection over REST](docs/03-programmatic-domain-connection-api.md) | The full API flow end to end, with example calls and webhook payloads |
| [Security for agent-driven DNS](docs/04-security-for-agent-driven-dns.md) | Scoped credentials, human approval points, and why agents should never hold registrar passwords |

## The problem: agents ship everything except the domain

An agent that builds and deploys a site produces a preview URL on a platform subdomain. The user's next message is predictable: "make it live on acmebakery.com." At that moment the agent needs four things to happen in someone else's infrastructure: routing records written into a DNS zone it does not control, proof that the user actually owns that zone, a TLS certificate issued for the hostname, and confirmation that resolvers worldwide are returning the new answers.

Humans already struggle here. The knowledge base of the Domain Connect Association (published under CC0) reports that roughly half of users who attempt manual DNS configuration fail and abandon the setup, and documents one major productivity suite whose email onboarding requires 7 to 15 hand-created records and 16 separately maintained registrar-specific help pages. Those users had already paid. They simply could not translate "add a CNAME" into the right clicks at their particular provider.

For an agent the failure mode is worse, because every workaround available to a human is unavailable or dangerous for an agent:

- **No hands.** The registrar's DNS panel is a browser UI, often behind 2FA. Driving it with browser automation is brittle and, with real registrar credentials in the loop, genuinely risky.
- **No safe credentials.** The obvious shortcut, "just give the agent my registrar password," hands an autonomous system full control of the zone, email routing, transfer locks, and billing. [Don't do this.](docs/04-security-for-agent-driven-dns.md)
- **No durable state.** DNS propagation can outlive an agent session. Without a resumable, queryable state machine, the agent cannot pick up where it left off.
- **No feedback.** A human squints at a help article and retries. An agent needs deterministic statuses, explicit error causes, and webhooks or pollable endpoints.

The result is the autonomy cliff: a fully automated build pipeline that ends with "now open your registrar dashboard and paste these records." The rest of this README is about removing that cliff.

## How domain connection actually works

Four mechanisms, in order. If you internalize these, every provider quirk becomes explainable.

### 1. Routing: DNS records point the name at your edge

A subdomain routes with a CNAME. A zone apex (the bare domain) cannot legally carry a CNAME alongside its other records, so it needs an A/AAAA record or a provider-specific ALIAS/ANAME that flattens to one:

```dns
; subdomain: CNAME to the platform edge
www.acmebakery.com.    3600  IN  CNAME  edge.yourplatform.com.

; apex: A record (or ALIAS/ANAME where the provider supports it)
acmebakery.com.        3600  IN  A      203.0.113.10
```

Getting these exactly right, at the right provider, in the right zone, is the step that generates most support tickets. See the plain-language walkthrough at [How to set up a custom domain](https://customdomain.ai/guides/how-to-set-up-a-custom-domain) and the [custom domain vs subdomain glossary entry](https://customdomain.ai/glossary/custom-domain-vs-subdomain) for why a platform subdomain is not a substitute.

### 2. Proof: ownership verification

Before a platform serves traffic or requests a certificate for a hostname, it must prove the requesting user controls that DNS zone. The standard mechanism is a scoped challenge record, typically TXT:

```dns
_cd-challenge.shop.acmebakery.com.  300  IN  TXT  "cd-verify=7f3a9b"
```

The platform polls until the challenge resolves, then marks the domain verified. Done correctly, verification requires no nameserver change and no access beyond that one record. This gate matters: it is what prevents anyone from pointing a domain they do not own at your infrastructure, and it is the same class of proof certificate authorities require.

### 3. Security: TLS issuance and renewal

Once ownership is proven, a certificate is requested through the ACME protocol from a publicly trusted certificate authority, validated against the live DNS, and installed at a reverse-proxy edge that terminates TLS. Renewal is scheduled automatically. In a managed setup the platform never generates a CSR, never stores the private key in its own application, and never wakes up to an expired certificate. Unverified hostnames never receive certificates and never serve traffic.

### 4. Waiting: propagation and TTLs

"Propagation" is not a push; it is caches expiring. Every resolver that previously looked up the name keeps its answer until the record's TTL runs out. Records written correctly on the first attempt, with sensible TTLs, resolve quickly. Records written wrong, then corrected, inherit the old TTL on the wrong answer, which is why manual setups "take 24 to 48 hours" in folklore and automated ones usually do not. When records are written through provider authorization, a domain is typically live in about 30 seconds.

## Three ways to get records into the customer's zone

Every domain connection product, and every homegrown flow, reduces to one of three methods. Custom Domain implements all three across 63 DNS and registrar providers, with more than 25 of them fully auto-configured. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for the provider-level view.

| Method | How it works | User effort | Typical time to live |
|---|---|---|---|
| **One-click provider authorization** | The user approves a scoped change at their DNS provider; correct records are applied automatically. Related in spirit to the Domain Connect protocol, an open standard from the Domain Connect Association, though provider authorization covers more providers than the protocol alone | One approval click | ~30 seconds |
| **API token** | The user pastes a scoped DNS API token from their provider; records are written through the provider's API | Create and paste one token | Minutes |
| **Guided manual with automatic verification** | The user is shown the exact records for their detected provider and the system polls DNS until they resolve, then proceeds automatically | Copy and paste records | Minutes to hours, TTL dependent |

The fallback chain is the point. Detection identifies the user's provider from the domain's existing DNS fingerprint, offers the best available method, and degrades gracefully to guided manual for the long tail. The agent's job stays the same in all three cases: initiate the connection, surface the next required human action if there is one, and poll or subscribe until the state reaches `live`.

## Implementation paths: widget, API, or MCP

| Path | Best for | Integration shape |
|---|---|---|
| [Connect widget + SDK](https://customdomain.ai/connect-domain-widget) | Agent products with a web UI where the end user completes the connection themselves | Embed a prebuilt flow; the widget handles provider detection, authorization, and status |
| [REST API](https://customdomain.ai/custom-domain-api) | Backend orchestration, existing job queues, platforms that already run a provisioning pipeline | Connections, DNS records, verification, TLS, monitoring, webhooks, registrar search and purchase |
| [Hosted MCP server](https://customdomain.ai/mcp-server) | Agents that operate through tool calls: coding assistants, autonomous builders, chat-native products | Streamable HTTP at `mcp.customdomain.ai/mcp` with API-key or OAuth client-credentials auth |

A minimal REST call to start a connection:

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com", "target": "edge.yourplatform.com"}'
```

And the one-liner that gives a Claude Code agent the full domain toolset:

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer sk_live_YOUR_KEY"
```

Full walkthroughs: [REST flow end to end](docs/03-programmatic-domain-connection-api.md) and [the MCP tool surface](docs/02-mcp-server-for-domains.md). Complete reference lives at [app.customdomain.ai/docs](https://app.customdomain.ai/docs).

## Decision guide

| Your situation | Start with |
|---|---|
| Users click "connect my domain" in your product's UI | Widget, with the API for status and webhooks |
| Your agent runs a provisioning pipeline server-side | REST API |
| Your agent is tool-calling (MCP-native, chat-driven) | MCP server |
| Users arrive without a domain at all | MCP or API registrar search and purchase, then auto-connect |
| You need human sign-off before money moves | All paths: purchases are fail-closed behind an authorization callback |

## FAQ

**Can the agent buy the domain too, not just connect it?**
Yes. Registrar search and purchase are part of both the REST API (`GET /v1/registrar/search`) and the MCP server (`search-domain-availability`, `create-domain-order`). A domain purchased this way connects itself, because its DNS is managed from the first second. Purchases are fail-closed: they require an explicit authorization callback, so an agent cannot spend money on its own.

**Does the agent need my customer's registrar login?**
No, and it never should. One-click provider authorization keeps credentials at the DNS provider, where the user approves a scoped change. The API-token path uses a token limited to DNS. The agent itself only ever holds a scoped, revocable Custom Domain key. The reasoning is laid out in [Security for agent-driven DNS](docs/04-security-for-agent-driven-dns.md).

**How long until the domain is actually live?**
Through provider authorization, typically about 30 seconds including TLS. Through guided manual, it depends on how fast the user pastes records and on existing TTLs; verification polls continuously and flips the state the moment records resolve.

**What happens if the agent session ends mid-connection?**
Nothing is lost. A connection is a durable resource with a deterministic state machine (`pending`, `verified`, `live`). Any later session, or any webhook consumer, can query `GET /v1/connections/:id` and resume from the current state.

**What if the user's DNS provider is not supported for automation?**
The flow degrades to guided manual: exact records rendered for the detected provider, with automatic verification polling. The agent still gets the same states and webhooks; only the human step changes.

**Can a compromised or prompt-injected agent write hostile DNS records?**
Not through the MCP server: there is deliberately no tool that writes raw DNS records. Record values are computed server-side from vetted configuration, so the blast radius of a bad tool call is a failed connection, not a hijacked zone. Details in [the security deep dive](docs/04-security-for-agent-driven-dns.md).

## Attribution

Parts of the problem framing in this repository draw on the knowledge base of the Domain Connect Association, published under CC0 1.0. The Domain Connect protocol is an open standard from the Domain Connect Association and is referenced here as third-party prior art, not as a product name.

## About Custom Domain

This repository is maintained by [Custom Domain](https://customdomain.ai). Custom Domain lets a platform's users connect their own domain with the least effort their DNS provider allows: coverage across 63 DNS and registrar providers, more than 25 of them fully auto-configured for one-click setup, plus domain ownership verification and automatic TLS issuance and renewal, on a managed control plane with a reverse-proxy edge and strict multi-tenant isolation. Surfaces include an [embeddable connect widget](https://customdomain.ai/connect-domain-widget), a [full REST API](https://customdomain.ai/custom-domain-api), and a [hosted MCP server for AI agents](https://customdomain.ai/mcp-server). Pricing starts at $0.

- [Custom Domain for AI agents](https://customdomain.ai/for/ai-agents)
- [Custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)
- [For site builders](https://customdomain.ai/for/site-builders) and [for agencies and white-label](https://customdomain.ai/for/agencies-white-label)
- [Developer docs](https://app.customdomain.ai/docs)
- [MCP client examples on GitHub](https://github.com/ever-just/customdomain-mcp)
- [Create a free account](https://app.customdomain.ai/signup)
