# Programmatic Domain Connection: The REST Flow End to End

When your agent product runs a server-side provisioning pipeline, the natural integration is HTTP, not tool calls. This document walks the full domain connection lifecycle over the [CustomDomain REST API](https://customdomain.ai/custom-domain-api): create a connection, read the authoritative record set, get the records applied through a rail, watch propagation, handle failure, and get notified when the domain goes live. The complete reference is at [docs.customdomain.ai](https://docs.customdomain.ai/docs), and the served OpenAPI 3.1 spec is at [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json).

## The resource model

Everything hangs off the **connection**: a durable resource that maps one customer hostname to your application's edge target. Its `id` doubles as the `jobId` the widget and webhooks report. Every connection carries an authoritative **desired record set** and a `status`. Authentication is a bearer credential, scoped and revocable: an `sk_` API key for server-side use, or a short-lived widget JWT when the browser is involved.

Connections move through exactly four statuses.

| Status | Meaning | Agent's move |
|---|---|---|
| `pending` | Created; records not yet written or not yet observed | Surface the next human action if the chosen rail needs one |
| `propagating` | Records were written by a rail and are being checked against public DNS | Wait; poll or subscribe to `connection.live` |
| `live` | Every desired record resolves to its intended value; the edge serves TLS for the host | Report success, store the connection id |
| `failed` | Records never appeared inside the window: 24 hours from `propagating` on an automatic rail, 72 hours from `pending` on manual | Read `error_code`, surface the concrete fix, then `POST /v1/connections/{id}:recheck` |

There is no separate verification status, because there is no separate ownership challenge. Control of the zone is proven by the rail that wrote the records, or in the manual case by the records appearing at all. See [Connections](https://docs.customdomain.ai/docs/concepts/connections).

Because the resource is durable and creation is idempotent, a connection created in one agent session is resumed from any later session with a single GET, or by re-posting the same domain. Nothing depends on the agent staying alive through DNS propagation.

## Step 0 (optional but worth it): check the domain first

```bash
curl -X POST https://api.customdomain.ai/v1/domains:check \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com"}'
```

Read-only detection. It returns the detected `provider`, the chosen `setup_type`, `supports_automatic`, `oauth_available`, `domain_connect`, capability flags (`ns_support`, `wildcard_support`, `cname_flattening`, `spf_override_support`, `caa_support`), any `record_conflicts` already visible in public DNS, and the server's authoritative Public Suffix List parse (`subdomain`, `registrable_domain`, `public_suffix`) which clients should adopt instead of computing their own apex boundary.

The field to check before an apex connection is `apex_supported`. If the provider cannot host a CNAME-like record at the zone root, this call returns `apex_message` and refuses here, before a connection exists, rather than letting it fail at apply time. An agent that calls `domains:check` first can tell the user "your DNS host cannot serve this at the bare domain, use `www` or move DNS" in the same turn, instead of 24 hours later.

## Step 1: create the connection

`domain` is the only required field. The body is decoded with unknown fields **rejected**, so any extra key is a `400`; the application, and therefore the edge target, comes from the credential, not from the body. The optional fields are `application_url`, `www_redirect`, `override_spf`, `validate_dmarc`, `validate_caa`, `monitor`, `batch_id`, and `end_user_ref`.

```bash
curl -X POST https://api.customdomain.ai/v1/connections \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain": "shop.acmebakery.com", "end_user_ref": "tenant_8842"}'
```

Response (`201` on create, `200` replaying an existing non-failed connection for the same application plus domain):

```json
{
  "id": "con_01j9x4",
  "application_id": "app_7t2k",
  "domain": "shop.acmebakery.com",
  "host": "shop",
  "provider_id": "cloudflare",
  "setup_type": "automatic",
  "status": "pending",
  "created_at": "2026-07-15T14:03:00Z",
  "records": [
    { "type": "CNAME", "host": "shop.acmebakery.com", "value": "edge.customdomain.ai", "ttl": 3600 }
  ]
}
```

`setup_type` is one of `automatic`, `manual`, `async`, or `mcp`, chosen from provider detection; it steers which rail is offered. `records` is the authoritative desired set, synthesized server-side before any rail has written anything, and re-fetchable at any time from `GET /v1/connections/{id}/records`. Note the record types you may see: `CNAME`, `A`, `AAAA`, `TXT`, `MX`, and `APEXCNAME`, which is the provider-agnostic CNAME-at-the-apex type realized to the host's native ALIAS, ANAME, or flattened CNAME on write and shown to users as `ALIAS`.

Nothing in this response asks the customer for nameserver access. The connection needs one record set on one hostname, and no more than that.

## Step 2: get the records applied through a rail

A connection reaches `live` through one of four rails. `POST /v1/domains:check` tells you which are available; the widget picks automatically.

| Rail | Endpoint | How records get written |
|---|---|---|
| OAuth into the provider | `POST /v1/connections/{id}/oauth:start` | The customer authorizes in a popup at their DNS provider; the callback writes records with a one-time token that is never stored |
| One-click setup (Domain Connect) | `POST /v1/connections/{id}/domainconnect:start` | Returns a provider-hosted apply URL; the provider applies a signed template. No server-to-server token at all |
| API token (bring your own) | `POST /v1/connections/{id}/apply` | You pass the customer's scoped provider token; it writes once and is discarded, unless you opt into remembering it |
| Manual | none | Render `GET /v1/connections/{id}/records` and let the poller wait |

The OAuth start call requires `return_origin`, the web origin the callback page may `postMessage` the outcome to, and that origin must be on the server's allowlist:

```bash
curl -X POST https://api.customdomain.ai/v1/connections/con_01j9x4/oauth:start \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"return_origin": "https://app.yourplatform.com"}'
# → { "authorize_url": "https://dash.cloudflare.com/oauth2/auth?..." }
```

For the manual rail, render the record set:

```bash
curl https://api.customdomain.ai/v1/connections/con_01j9x4/records \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN"
# → { "connection_id": "con_01j9x4", "records": [ { "type": "CNAME", "host": "…", "value": "…", "ttl": 3600, "applied": false } ] }
```

`applied` on each record is `true` once it has been written through a provider. Rendering exactly these values, for the detected provider, is what an agent should relay to a user instead of pasting a generic help article. See [one-click DNS setup](https://customdomain.ai/one-click-dns-setup) for how rail selection looks from the user's side.

## Step 3: wait without polling loops (webhooks)

A background poller re-resolves every `pending` and `propagating` connection on a one-minute interval, with value checking, and flips the status the moment every record resolves. Your pipeline can poll `GET /v1/connections/con_01j9x4`, but for anything beyond a demo, register a webhook:

```bash
curl -X POST https://api.customdomain.ai/v1/webhooks \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://hooks.yourplatform.example/customdomain",
       "events": ["connection.applied", "connection.live", "connection.failed"]}'
# → { "id": "whk_…", "secret": "whsec_…", … }
```

The `secret` is returned once. Store it and verify signatures ([how](https://docs.customdomain.ai/docs/webhooks/verifying-signatures)). An empty `events` array subscribes to everything.

The connection-lifecycle events are `connection.created`, `connection.applied`, `connection.live`, `connection.failed`, `connection.records.updated`, `connection.reapply.failed`, `connection.disconnected`, and `connection.records_outdated`. Purchases add `domain.purchased`, `purchase.error`, and `purchase.confirmation.expired`. Drift adds `domain.record_missing` and `domain.record_restored`. Provisioning adds `secure_status` and `power_status`. That list is the complete vocabulary; an unknown event string is accepted at registration but can never be delivered. The full catalog is in [Webhooks](https://docs.customdomain.ai/docs/webhooks/overview).

The payload when a domain goes live is a flat envelope:

```json
{
  "id": "evt_01j9x9",
  "type": "connection.live",
  "domain": "shop.acmebakery.com",
  "subdomain": "shop",
  "provider": "cloudflare",
  "setup_type": "automatic",
  "propagation_status": "success",
  "data": {
    "records_propagated": [
      { "type": "CNAME", "host": "shop.acmebakery.com", "value": "edge.customdomain.ai" }
    ],
    "records_non_propagated": []
  },
  "created_at": "2026-07-15T14:03:41Z"
}
```

Two delivery properties an agent pipeline should be built around. Deliveries retry with exponential backoff for up to 12 attempts spanning about a day before being dead-lettered, and a non-transient `4xx` other than `429` is not retried at all, because a retry cannot fix a misconfiguration. And the same event can arrive more than once, so your endpoint must be idempotent: deduplicate on the envelope `id`, or on `domain` plus `type`. Inspect what actually happened with `GET /v1/webhook-deliveries`.

One caveat on the drift events specifically: they are produced by an hourly Monitor sweep that only delivers where the deployment enables alerts, and the hosted default has them off. The sweep runs in shadow mode regardless. Do not design a customer-facing alert on `domain.record_missing` without confirming delivery is on for your account; use the synchronous `POST /v1/monitor:check` if you need drift detection you control.

## Step 4: when it fails

This is the part most integration guides skip, including earlier revisions of this one.

A connection is marked `failed` when its records never appear inside the window. On an automatic rail that is 24 hours from `propagating`, because a write that the provider accepted should resolve long before then. On a manual connection it is 72 hours from `pending`, which is long enough for a human to get to it. The connection then carries `error_code` and `error_message`.

| `error_code` | What actually happened | What to tell the user |
|---|---|---|
| `setup_incomplete` | A manual connection's records were never added | Re-render the record set and ask again. Nothing is broken |
| `propagation_timeout` | Records were written but never resolved | Usually the wrong zone, a conflicting record, or an overwrite. Check `records_conflicts` from `domains:check` |
| `apex_not_supported` | The provider cannot host a CNAME-like record at the zone root | Use a subdomain, or move DNS to a provider with ALIAS/ANAME support |
| `domain_already_connected` | The hostname is attached elsewhere | Disconnect the other connection first |
| `dns_write_failed`, `ProviderAuthenticationError` | The foreground apply through a token or OAuth failed at the provider | Re-authorize, or supply a token with DNS scope |
| `spf_merge_failed` | An existing SPF record could not be merged | Set `override_spf` deliberately, or fix the record by hand |
| `forwarding_not_supported` | The provider cannot express the requested forward | Use a different rail or a different provider |

Request validation failures (`400`) are deliberately not recorded in `error_code`: a malformed request is not a connection failure. Errors come back as a `{ code, title, details }` envelope, where `code` is the stable machine-readable string to branch on.

The retry verb is `:recheck`:

```bash
curl -X POST https://api.customdomain.ai/v1/connections/con_01j9x4:recheck \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN"
```

It resolves every record the connection is verified against **in-request** against live public DNS and returns the full connection plus the check's findings: `checked` (a per-record verdict with `resolved` true or false), `records_resolved` (true only when every entry resolved), `retried`, and `previous_status`. It also advances the connection: a `pending` or `propagating` connection whose records are all visible is promoted to `live` right there instead of on the poller's next pass, and a `failed` connection is re-entered into verification with its timeout clock re-based off the retry. Without that re-basing a retry would re-fail immediately, since the connection is `failed` precisely because it aged past the window.

It is idempotent and safe to call repeatedly. One subtlety worth building on: `:recheck` never transitions a `live` connection, so `records_resolved: false` on a `live` connection is a drift signal rather than a failure.

## Step 5: TLS, observable but hands-off

Certificates are issued for the hostname once the records resolve, on the next TLS handshake at the managed reverse-proxy edge, and renewal is scheduled automatically. You never generate a CSR, never store a private key, and never track renewal dates. For dashboards and support tooling the mirrored status is queryable:

```bash
curl https://api.customdomain.ai/v1/connections/con_01j9x4/ssl \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN"
# → { "certificate": { … } }   (404 when no certificate is tracked yet)
```

Note the path is `/ssl`, not `/tls`, and note the entitlement: the `/v1/ssl*` surface (including `ssl:provision`, `ssl:renew`, and `ssl:import` for bringing your own certificate) requires the `Secure` entitlement, which starts on the Growth plan. An unentitled tenant gets `402 plan_upgrade_required`. Automatic issuance as part of going live is included on every plan; it is the management surface that is gated.

## Registrar search and purchase in the same flow

When the customer has no domain yet, the same API closes that gap:

```bash
curl "https://api.customdomain.ai/v1/registrar/search?q=acmebakery&suggest=1" \
  -H "Authorization: Bearer $CUSTOMDOMAIN_TOKEN"
```

With `suggest=1` each result carries a `Kind` of `exact`, `tld`, or `variant` so you can group the exact match, the same name on other endings, and close variants. `POST /v1/registrar/quote` prices a specific name; `POST /v1/registrar/checkout` opens a Stripe session; `POST /v1/registrar/fulfill` registers the domain, captures the charge, and auto-connects it.

The ordering is the safety property: authorize the card, register server-side driven by the Stripe webhook, and only then capture. If registration fails, the authorization is released and nobody pays. Fulfillment is idempotent per checkout session, so a reload cannot register or charge twice. A purchased domain auto-connects, because its zone is created pointing at the edge, so there is no detection step and no records for anyone to paste. Purchasing is fail-closed: where the registrar credential or Stripe key is not configured, purchase returns `503` rather than taking an order it cannot fulfill, and MCP-driven orders additionally require the integrator's purchase-authorization callback (see [Security for agent-driven DNS](04-security-for-agent-driven-dns.md)).

## Exposing this to a Claude agent as tools

If your product's agent is the caller and you would rather define your own tools than adopt the [hosted MCP server](02-mcp-server-for-domains.md), the wrapping is small. Two tools carry most of the value: start a connection, and read its state. Keep the record set out of the tool inputs, the same way the MCP server does.

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();
const CD = "https://api.customdomain.ai/v1";
const auth = { Authorization: `Bearer ${process.env.CUSTOMDOMAIN_TOKEN}` };

const connectDomain = betaZodTool({
  name: "connect_domain",
  description:
    "Start connecting a customer's domain to our platform. Idempotent: calling it " +
    "again for the same domain returns the existing connection. Returns the " +
    "connection id, its status, and the DNS records the customer may need to add.",
  inputSchema: z.object({
    domain: z.string().describe("Hostname to connect, e.g. shop.acmebakery.com"),
  }),
  run: async ({ domain }) => {
    const res = await fetch(`${CD}/connections`, {
      method: "POST",
      headers: { ...auth, "Content-Type": "application/json" },
      body: JSON.stringify({ domain }),
    });
    return JSON.stringify(await res.json());
  },
});

const checkDomainStatus = betaZodTool({
  name: "check_domain_status",
  description:
    "Re-resolve a connection's DNS right now and return its status. Status is one " +
    "of pending, propagating, live, failed. On failed, read error_code before " +
    "telling the user anything.",
  inputSchema: z.object({
    connectionId: z.string().describe("Connection id returned by connect_domain"),
  }),
  run: async ({ connectionId }) => {
    const res = await fetch(`${CD}/connections/${connectionId}:recheck`, {
      method: "POST",
      headers: auth,
    });
    return JSON.stringify(await res.json());
  },
});

const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-5",
  max_tokens: 16000,
  tools: [connectDomain, checkDomainStatus],
  messages: [
    { role: "user", content: "Put my site live on shop.acmebakery.com" },
  ],
});
```

Two things to get right in the descriptions, because they are what the model actually reads. Say that `connect_domain` is idempotent, or the model will avoid re-calling it after an interruption and invent a resume path instead. And name the four statuses explicitly, or it will guess a fifth. Do not add a tool that takes DNS record values as arguments: that is the one shape that turns a prompt injection into a zone change, and the API will not accept it anyway.

## Practical notes for agent pipelines

- **Idempotency is server-side, not a header you send.** `POST /v1/connections` is idempotent per application plus domain: an existing non-failed connection for that domain is replayed with `200` instead of a duplicate being created. Safe to call on every widget open and on every pipeline retry.
- **Tenancy is derived from the credential, never from a parameter.** An `sk_` key sees all its tenant's connections; a widget JWT sees only its application's, and a domain-bound JWT can only act on the hostname it was minted for. Cross-tenant ids return `404`, never another tenant's data.
- **Error surfacing.** Branch on `error_code` on a connection and on the `code` field of the error envelope. Both are stable strings; `title` and `error_message` are for humans.
- **Cleanup.** `DELETE /v1/connections/{id}` when a customer detaches a domain. For a managed connection this reverts the template through the stored grant first, which is what keeps a customer's zone from being left pointing at an edge that no longer serves them.
- **Published rate limits are not documented.** Treat `429` as retryable with backoff and do not build a pipeline that assumes a specific ceiling.
- **Tool-calling agents.** If the caller is the agent itself rather than your backend, the same lifecycle is exposed as MCP tools; see [An MCP server for domains](02-mcp-server-for-domains.md).

---

Next: [Security for agent-driven DNS](04-security-for-agent-driven-dns.md) · Previous: [An MCP server for domains](02-mcp-server-for-domains.md) · Back to the [overview](../README.md)
