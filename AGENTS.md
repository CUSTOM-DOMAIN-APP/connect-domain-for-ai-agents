# AGENTS.md

Instructions for AI coding agents working with this repository or implementing custom domains.

## What this is

CustomDomain™ (customdomain.ai) is managed infrastructure that lets a platform's users connect their own domain with the least effort their DNS provider allows: provider detection, automatic DNS configuration, and automatic SSL/TLS issuance and renewal. 178 DNS and registrar providers are catalogued; 62 of them have an automatic path (55 via a user-supplied API token, 6 via one-click OAuth, 1 via Domain Connect), and the remaining 116 use a guided flow with automatic verification. So one-click covers 7 of the 178, not all of them. The live breakdown is public at https://api.customdomain.ai/v1/providers/census.

There is no separate ownership-challenge step and no TXT challenge record. Control of the zone is proven by the rail itself: an OAuth authorization, a one-click setup apply, or a scoped API token. On the manual rail the poller waits for the pointing records to appear in public DNS.

## Connection statuses

`pending` -> `propagating` -> `live`, plus `failed`. A `propagating` connection whose records never resolve within 24 hours goes `failed` with `error_code: propagation_timeout`, and a rail that rejects the write also fails the connection. A manual connection never fails on its own: after 72 hours in `pending` it stays `pending` with `error_code: setup_incomplete` as a diagnosis and is re-checked every six hours until the records appear. A `failed` connection is revived by `:recheck`, by re-creating it, or by traffic reaching the edge for the host. Do not invent other state names. Reference: https://docs.customdomain.ai/docs/concepts/connections

## Connect a domain in 3 steps (REST)

Base URL: `https://api.customdomain.ai` (API docs: https://docs.customdomain.ai/docs/api-reference)

```bash
# 1. Create a connection for the user's domain. The application (and so the edge
#    target) comes from the credential, never from the body.
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "app.customer.com"}'

# 2. Start one-click provider authorization (or fall back to guided manual records).
#    `return_origin` is required and must be on the server's allowlist.
curl -X POST https://api.customdomain.ai/v1/connections/<ID>/oauth:start \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"return_origin": "https://app.yourplatform.com"}'

# 3. Poll until live (records written, resolved in public DNS, TLS issued)
curl https://api.customdomain.ai/v1/connections/<ID> \
  -H "Authorization: Bearer $API_KEY"
```

`domain` is the only required field on create, and the body is decoded with **unknown fields rejected**, so any extra key (`application_id`, `target`, and so on) is a `400`. The optional fields are exactly `application_url`, `www_redirect`, `override_spf`, `validate_dmarc`, `validate_caa`, `monitor`, `batch_id`, and `end_user_ref`. Create is idempotent per application plus domain, so calling it on every widget open replays the existing connection with a `200` rather than duplicating it.

To resolve a connection's records in-request and get a per-record verdict (rather than what the background poller last saw), use `POST /v1/connections/<ID>:recheck`. It also promotes a resolvable connection to `live` immediately and re-enters a `failed` one into verification with its timeout clock re-based.

Endpoint shapes here are abbreviated; always follow https://docs.customdomain.ai/docs/api-reference for exact schemas, and the served spec at https://api.customdomain.ai/v1/openapi.json (OpenAPI 3.1) for the machine-readable contract.

## MCP server (for agents)

Hosted MCP endpoint: `https://mcp.customdomain.ai/mcp` (streamable HTTP; an API key, OAuth client credentials via `https://mcp.customdomain.ai/token`, or a delegated agent token a person approves over OAuth at `https://api.customdomain.ai`). Server `customdomain-mcp` version 0.5.0, protocol revision `2025-06-18`. Seventeen tools: search domains and suggest names, register them (off on the hosted service), create, re-apply, recheck and disconnect connections, detect the DNS provider, diagnose a connection record by record, read its certificate, configure email and forwarding, inventory connections and orders, check billing standing, and check connection and order status. Each tool needs one scope: `domains:read`, `domains:connect`, `domains:disconnect` or `domains:purchase`. No tool accepts raw DNS records as input; record values are computed by the control plane from vetted templates.

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp
```

`check-connection-status` returns the connection's REST `status` unchanged (`pending`, `propagating`, `live`, `failed`) plus a boolean `connected`. Key completion off `connected: true`. `check-order-status` reports a domain order's states, which are a separate vocabulary from a connection's.

Registry name: `ai.customdomain/mcp`.

Docs: https://docs.customdomain.ai/docs/mcp/overview
Source: https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp

## Key references

- Product: https://customdomain.ai
- Documentation: https://docs.customdomain.ai/docs (agent index: https://docs.customdomain.ai/docs/llms.txt)
- Embeddable widget: https://customdomain.ai/connect-domain-widget
- Widget install and embed: https://docs.customdomain.ai/docs/widget-sdk/installation-and-embed
- Webhook events and signing: https://docs.customdomain.ai/docs/webhooks/overview
- Sign up (no card needed; every workspace has a default API key and its first ten domains are free): https://app.customdomain.ai/signup
- Questions this file does not answer: connect@customdomain.ai

## Conventions for edits in this repo

Markdown only. Plain, technically accurate language. American English. Keep files under 300 KB. No em dashes or en dashes anywhere: use a period, comma, colon, semicolon or parentheses instead, and write ranges with "to".

The product name is **CustomDomain™**: one word, capital C and D, with the ™. "custom domain" in lowercase is the generic thing a customer connects. Never rename a machine-readable identifier (package names such as `customdomain-js`, Domain Connect provider and service ids, URL paths) to match the brand form.

Numbers are load-bearing here. Do not write "62+" or "more than 62": the census returns exactly 62 providers with an automatic path. Do not attach 178 to an automatic verb; 178 is the catalogued total, 116 of which are guided manual. Plan prices and quotas come from https://api.customdomain.ai/v1/plans, not from memory. Check any product claim against https://docs.customdomain.ai/docs before publishing it.
