# AGENTS.md

Instructions for AI coding agents working with this repository or implementing custom domains.

## What this is

CustomDomain (customdomain.ai) is managed infrastructure that lets a platform's users connect their own domain with the least effort their DNS provider allows: provider detection, automatic DNS configuration, and automatic SSL/TLS issuance and renewal. 63 DNS and registrar providers are catalogued; 25 of them have an automatic path (17 via a user-supplied API token, 6 via one-click OAuth, 2 via Domain Connect), and the remaining 38 use a guided flow with automatic verification. So one-click covers 8 of the 63, not all of them. The live breakdown is public at https://api.customdomain.ai/v1/providers/census.

There is no separate ownership-challenge step and no TXT challenge record. Control of the zone is proven by the rail itself: an OAuth authorization, a one-click setup apply, or a scoped API token. On the manual rail the poller waits for the pointing records to appear in public DNS.

## Connection statuses

`pending` -> `propagating` -> `live`, plus `failed`. A `propagating` connection whose records never resolve within 24 hours goes `failed` with `error_code: propagation_timeout`; a manual connection whose records are never added goes `failed` with `setup_incomplete` after 72 hours. Both clear automatically if the connection later verifies. Do not invent other state names. Reference: https://docs.customdomain.ai/docs/concepts/connections

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

Hosted MCP endpoint: `https://mcp.customdomain.ai/mcp` (streamable HTTP, OAuth client credentials via `https://mcp.customdomain.ai/token`). Twelve tools: search and register domains, create, re-apply and disconnect connections, detect the DNS provider, configure email and forwarding, inventory connections, and check verification and TLS status. No tool accepts raw DNS records as input; record values are computed by the control plane from vetted templates.

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp
```

`check-connection-status` reports the MCP job vocabulary (`pending`, `propagating`, `completed`, `failed`, `error`, `expired`), which is one word off from the REST `status` field: `completed` there is `live` here. Branch on the vocabulary of whichever surface you called rather than normalizing by hand.

Registry name: `ai.customdomain/mcp`.

Docs: https://docs.customdomain.ai/docs/mcp/overview
Source: https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp

## Key references

- Product: https://customdomain.ai
- Documentation: https://docs.customdomain.ai/docs (agent index: https://docs.customdomain.ai/docs/llms.txt)
- Embeddable widget: https://customdomain.ai/connect-domain-widget
- Widget install and embed: https://docs.customdomain.ai/docs/widget-sdk/installation-and-embed
- Webhook events and signing: https://docs.customdomain.ai/docs/webhooks/overview
- Sign up (free tier): https://app.customdomain.ai/signup

## Conventions for edits in this repo

Markdown only. No em dashes or en dashes anywhere. Plain, technically accurate language. American English. Keep files under 300 KB.

The brand is **CustomDomain**, one word. "Custom Domain" with a space is the generic category. Never rename a machine-readable identifier (package names, Domain Connect provider and service ids, URL paths) to match.

Numbers are load-bearing here. Do not write "25+" or "more than 25": the census returns exactly 25 providers with an automatic path. Do not attach 63 to an automatic verb; 63 is the catalogued total, 38 of which are guided manual. Plan prices and quotas come from https://api.customdomain.ai/v1/plans, not from memory. Check any product claim against https://docs.customdomain.ai/docs before publishing it.
