# Agents That Ship Websites Need Domains

The most useful thing an AI agent can do with a website is finish it. Agents today can take a prompt to a deployed, working app without a human touching a keyboard, and then hand back a preview URL on a platform subdomain. This document looks closely at that pattern, and at the exact points where the custom domain step breaks it.

## The agent-provisions-everything pattern

A modern build agent runs a loop that looks like this:

1. Interpret the request ("a booking site for my bakery").
2. Generate or modify the codebase.
3. Provision infrastructure: hosting, database, environment config.
4. Deploy, test, iterate.
5. Return a URL.

Steps 1 through 4 are fully tool-mediated. Every resource the agent touches has an API, a deterministic response, and a way to check state. Step 5 returns something like `acme-bakery-7f3a.platform.app`, and the user immediately asks for `acmebakery.com`.

That request is reasonable. A real domain is the difference between a demo and a business: it carries the brand, it is what customers type and trust, it is where email lives, and it is the stable address that search engines index. The [custom domain vs subdomain glossary entry](https://customdomain.ai/glossary/custom-domain-vs-subdomain) covers the practical differences; the short version is that no serious product launch ends on a platform subdomain.

## Where the loop breaks

The domain step violates every assumption the agent loop depends on.

**It happens in someone else's system.** The records must be written into a DNS zone at whichever of dozens of registrars and DNS hosts the user happens to use. There is no single API, and the interfaces range from clean REST to a web panel from 2009.

**It traditionally requires a human with credentials.** The classic instructions are "log into your registrar and add these records." The human-facing version of this already fails about half the time, per the CC0 knowledge base of the Domain Connect Association. The agent-facing version is worse: an agent has no registrar login, and giving it one is a serious mistake (see [Security for agent-driven DNS](04-security-for-agent-driven-dns.md)).

**It has hidden preconditions.** Apex domains cannot take a CNAME next to their other records, so the agent must know whether this provider supports ALIAS flattening or needs an A record. Existing records can conflict. Email routing can break if an MX record is disturbed. A correct answer for one provider is wrong for another.

**Its timescale outlives the session.** DNS answers are cached by resolvers until TTLs expire, and TLS issuance can only start after ownership verification. A user who pastes records two hours after the agent asked has, in most architectures, an agent that is long gone. Without a durable state to resume from, the workflow simply dies.

**It gives no structured feedback.** A human retries after reading a help article. An agent needs machine-readable state: is this pending because the record has not resolved, because the wrong value was pasted, or because verification has not been attempted?

The outcome is familiar to anyone building in this space: a beautifully autonomous pipeline that ends with a wall of manual instructions and a support queue.

## What the domain step needs to look like for agents

Working backwards from the failure points, an agent-compatible domain flow needs five properties:

| Property | Why it matters to an agent |
|---|---|
| API or tool-call surface for everything | No browser automation, no screen scraping, no clipboard |
| Deterministic state machine | `pending`, `verified`, `live` lets the agent branch, report, and retry rationally |
| Durable, resumable resources | Any later session can query the connection and continue |
| Push and pull status | Webhooks for pipelines, pollable status for chat loops |
| Scoped credentials with human gates | The agent holds a revocable key, never a registrar password; money and ownership changes require explicit approval |

This is precisely the shape of a managed domain connection layer. [Custom Domain](https://customdomain.ai) exposes it three ways: an [embeddable widget](https://customdomain.ai/connect-domain-widget) when the end user should click through the flow themselves, a [REST API](https://customdomain.ai/custom-domain-api) when a backend orchestrates it, and a [hosted MCP server](https://customdomain.ai/mcp-server) when the agent itself is the caller. Provider detection fingerprints the user's DNS host and picks the best of three connection methods automatically: one-click provider authorization (typically live in about 30 seconds), a scoped API token, or a guided manual flow with automatic verification. Coverage spans 63 DNS and registrar providers, with more than 25 fully auto-configured.

## Closing the loop entirely: agents that buy the domain

Many users arrive without a domain at all. The complete agent flow then becomes: search availability, suggest names, purchase (behind an explicit human authorization, since purchases are fail-closed), and connect. A domain bought this way is the easy case, because its DNS is under management from the first second, so there is nothing to detect and no records for anyone to paste. The end-to-end sequence, as tool calls, is walked through in [An MCP server for domains](02-mcp-server-for-domains.md); the same lifecycle over HTTP is in [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md).

## Who is building this way

The pattern is not limited to coding agents. Site builders adding an AI assistant ([for site builders](https://customdomain.ai/for/site-builders)), agencies shipping white-label client sites at volume ([for agencies and white-label](https://customdomain.ai/for/agencies-white-label)), and SaaS platforms giving every tenant a branded hostname ([custom domains for SaaS](https://customdomain.ai/custom-domains-for-saas)) all hit the same wall, and all benefit from the same fix: make the domain step a set of tool calls with verifiable state, and keep the human in the loop only where consent genuinely matters.

---

Next: [An MCP server for domains](02-mcp-server-for-domains.md) · Back to the [overview](../README.md)
