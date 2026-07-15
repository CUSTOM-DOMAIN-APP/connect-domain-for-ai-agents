# Programmatic Domain Connection: The REST Flow End to End

When your agent product runs a server-side provisioning pipeline, the natural integration is HTTP, not tool calls. This document walks the full domain connection lifecycle over the [Custom Domain REST API](https://customdomain.ai/custom-domain-api): create a connection, read the required records, verify ownership, watch TLS issuance, and get notified when the domain goes live. The complete reference is at [app.customdomain.ai/docs](https://app.customdomain.ai/docs).

## The resource model

Everything hangs off the **connection**: a durable resource that maps one customer hostname to one platform target. The API surface covers connections, DNS records, verification, TLS lifecycle, monitoring, webhooks, and registrar search and purchase. Authentication is a bearer token, scoped and revocable per application.

Connections move through a deterministic state machine:

| State | Meaning | Agent's move |
|---|---|---|
| `pending` | Created; ownership not yet proven | Surface the next human action if any; poll or wait for webhook |
| `verified` | Ownership challenge resolved | Nothing; TLS issuance starts automatically |
| `live` | Records resolve, certificate issued, traffic serving | Report success, store the connection ID |

Because the resource is durable, a connection created in one agent session can be resumed from any later session with a single GET. Nothing depends on the agent staying alive through DNS propagation.

## Step 1: create the connection

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com", "target": "edge.yourplatform.com"}'
```

Response:

```json
{
  "id": "con_01j9x4",
  "domain": "shop.acmebakery.com",
  "status": "pending",
  "verification": {
    "method": "dns_txt",
    "record": {
      "type": "TXT",
      "name": "_cd-challenge.shop.acmebakery.com",
      "value": "cd-verify=7f3a9b"
    }
  },
  "tls": { "status": "awaiting_verification" },
  "created_at": "2026-07-15T14:03:00Z"
}
```

Note what verification asks for: one scoped TXT challenge. The customer never hands over nameservers and never grants more access than the proof requires.

## Step 2: get records applied

How the routing and challenge records reach the customer's zone depends on their provider, and the platform picks the best of three methods automatically across 63 supported DNS and registrar providers. With one-click provider authorization (available at more than 25 auto-configured providers), the user approves once and correct records are written for them; the connection is typically live in about 30 seconds. With an API token, records are written through the provider's API. With guided manual, your UI renders the exact records for the detected provider:

```bash
curl https://api.customdomain.ai/v1/connections/con_01j9x4/records \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN"
```

This returns both the required records and what currently exists in DNS, so an agent can tell the user precisely what is missing or mismatched instead of pasting a generic help article. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for how the method selection works from the user's side.

## Step 3: wait without polling loops (webhooks)

Verification polls DNS continuously on the platform side and flips the state the moment the challenge resolves. Your pipeline can poll `GET /v1/connections/con_01j9x4`, but for anything beyond a demo, register a webhook:

```bash
curl -X POST https://api.customdomain.ai/v1/webhooks \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://hooks.yourplatform.com/customdomain", "events": ["connection.verified", "connection.live", "tls.renewed"]}'
```

The payload when a domain goes live:

```json
{
  "event": "connection.live",
  "data": {
    "connection_id": "con_01j9x4",
    "domain": "shop.acmebakery.com",
    "tls": { "status": "issued", "renews_at": "2026-09-13" }
  }
}
```

Events cover the connection lifecycle plus TLS issuance, renewal, and drift detection, so you also hear about it when a customer's records change out from under a live domain.

## Step 4: TLS, observable but hands-off

Certificates are requested automatically after verification and terminate at the managed reverse-proxy edge. You never generate a CSR, never store a private key, and never track renewal dates. For dashboards and support tooling, the status is queryable:

```bash
curl https://api.customdomain.ai/v1/connections/con_01j9x4/tls \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN"
```

This reports certificate status, issuer details, and the renewal schedule.

## Registrar search and purchase in the same flow

When the customer has no domain yet, the same API closes that gap:

```bash
curl "https://api.customdomain.ai/v1/registrar/search?q=acmebakery" \
  -H "Authorization: Bearer $CUSTOM_DOMAIN_TOKEN"
```

Search returns availability inside your product flow; purchase completes behind an explicit authorization step (purchases are fail-closed, a property that matters for agent-driven flows; see [Security for agent-driven DNS](04-security-for-agent-driven-dns.md)). A domain bought this way connects itself, because its DNS is under platform management from the first second: no detection, no pasting, no waiting on foreign TTLs.

## Practical notes for agent pipelines

- **Idempotency**: treat the connection ID as the source of truth. Re-running a pipeline should GET before it POSTs.
- **Error surfacing**: the records endpoint's required-versus-existing diff is the right thing to show a user, or to feed back into an agent's next message.
- **Cleanup**: `DELETE /v1/connections/:id` when a customer detaches a domain, which also avoids leaving stale records pointing at your edge.
- **Tool-calling agents**: if the caller is the agent itself rather than your backend, the same lifecycle is exposed as MCP tools; see [An MCP server for domains](02-mcp-server-for-domains.md).

---

Next: [Security for agent-driven DNS](04-security-for-agent-driven-dns.md) · Previous: [An MCP server for domains](02-mcp-server-for-domains.md) · Back to the [overview](../README.md)
