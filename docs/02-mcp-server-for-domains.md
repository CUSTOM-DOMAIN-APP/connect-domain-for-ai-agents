# An MCP Server for Domains: Search, Register, Connect, and Secure

The Model Context Protocol (MCP) is the standard for giving AI agents tools they can call directly. This document describes what a hosted MCP server for the domain lifecycle exposes, using the [CustomDomain MCP server](https://customdomain.ai/mcp-server) as the concrete example: domain search and registration, connection, propagation status, email routing, and forwarding, all as first-class tools an agent can call without wrapper code.

## Endpoint and authentication

The server is hosted; there is nothing to run or deploy on your side. It is listed in the official MCP registry as `ai.customdomain/mcp`.

| Item | Value |
|---|---|
| Endpoint | `https://mcp.customdomain.ai/mcp` |
| Transport | Streamable HTTP, JSON-RPC 2.0 |
| Server identity | `customdomain-mcp`, protocol revision `2025-06-18`, advertises only the `tools` capability |
| Auth, option 1 | API key from the console (`sk_live_...` or `sk_test_...`), scoped to one application |
| Auth, option 2 | OAuth client credentials: exchange an application id and client secret for a short-lived JWT |
| Auth, option 3 | The `mcp-remote` shortcut form `CLIENT_ID:CLIENT_SECRET`, which the server exchanges for a JWT on the fly |

The OAuth exchange, for agents that should never see a long-lived key:

```bash
curl -X POST https://mcp.customdomain.ai/token \
  -u "APPLICATION_ID:CLIENT_SECRET" \
  -d "grant_type=client_credentials"
# → { "access_token": "eyJ…", "token_type": "Bearer", "expires_in": 3600 }
```

The server does not validate credentials cryptographically itself. It forwards the caller's credential to the control plane, which enforces the same application-scoped permissions as the REST API. Revoke the key in the console and the agent loses access on the next call. OAuth authorization-server metadata is published at `https://mcp.customdomain.ai/.well-known/oauth-authorization-server`.

## The tool surface

Twelve tools cover the lifecycle, registered in a fixed order. Arguments are listed because agent authors need them to write prompts and schemas that do not guess.

| Phase | Tool | Arguments | What it does |
|---|---|---|---|
| Discover and register | `search-domain-availability` | `{ domain }` | Availability with live purchase and renewal pricing |
| | `generate-domain-suggestions` | `{ keywords, limit? }` | Available-to-register name ideas, each priced; default 5, max 20 |
| | `create-domain-order` | `{ domain }` | Start a registration through the resolved registrar |
| | `check-order-status` | `{ orderId }` or `{ jobId }` | Read the live status of a registration order; supply exactly one |
| Connect | `discover-provider` | `{ domain }` | Detect where DNS is hosted and which automated rails are available. Read-only |
| | `connect-domain` | `{ domain }` | Start a guided DNS configuration flow. Returns a `link` for the user and a `jobId` to poll |
| | `check-connection-status` | `{ jobId }` | Read the live status of a connection job |
| | `reapply-connection` | `{ connectionId }` | Re-push a managed connection's DNS through its stored grant |
| | `disconnect-domain` | `{ connectionId }` | Revert the DNS through the grant, then delete the grant and connection |
| | `list-connections` | `{ status? }` | Inventory every connection on the account, optionally filtered. Read-only |
| Route mail and traffic | `add-email` | `{ domain?, connectionId?, provider?, mxHost?, spfInclude?, dkimSelector?, dkimTarget?, dmarcRua? }` | Configure MX, SPF, DKIM and DMARC in one call via server-side templates. `provider` (`google`, `microsoft365`, `zoho`) fills in the standard MX and SPF |
| | `forward-domain` | `{ target, domain?, connectionId? }` | Permanent 301 forward, via a managed one-click setup flow |

One design decision matters more than the tool list: **no tool accepts DNS records as input.** Record names and values are computed by the control plane from vetted per-provider templates. An agent can ask for a domain to be connected, or for Google Workspace mail to be configured; it cannot ask for `acmebakery.com` to be pointed at an arbitrary IP through a crafted record. If a prompt injection reaches the agent, the worst available DNS outcome is a failed connection, not a hijacked zone. The full reasoning is in [Security for agent-driven DNS](04-security-for-agent-driven-dns.md).

If you have read older copy of ours describing these tools as "manage DNS records," that description was wrong in a way that matters, because it invites an integrator to build exactly the injection path the design closes.

Purchases are similarly guarded: `create-domain-order` places a paid order only after a successful call to the integrator's purchase-authorization callback. If that callback is not configured, every purchase is denied. That is the safe default, not a bug.

One more thing worth knowing before you write prompt text: `generate-domain-suggestions` is deterministic, not a model call. Candidates are keyword joins plus a few common affixes, expanded across TLDs by registrar search and filtered to what is actually available. The same keywords always produce the same candidates, which is what makes the tool testable, and also means it will not surprise you with a clever name.

## Client configuration

**Claude Code** (one command):

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer sk_live_YOUR_KEY"
```

**Claude Desktop** (`claude_desktop_config.json`, via `mcp-remote`):

```json
{
  "mcpServers": {
    "customdomain": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://mcp.customdomain.ai/mcp",
        "--header", "Authorization: Bearer sk_live_YOUR_KEY"
      ]
    }
  }
}
```

**Cursor** (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "customdomain": {
      "url": "https://mcp.customdomain.ai/mcp",
      "headers": { "Authorization": "Bearer sk_live_YOUR_KEY" }
    }
  }
}
```

Working client examples live in the [customdomain-mcp repository on GitHub](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp), and the full tool reference is at [docs.customdomain.ai/docs/mcp/overview](https://docs.customdomain.ai/docs/mcp/overview).

## What a session actually looks like

A typical "make it live on a real domain" exchange, as tool calls:

1. `search-domain-availability` for `acmebakery.com`. Taken. `generate-domain-suggestions` returns available alternatives; the user picks one.
2. `create-domain-order`. The order waits on the purchase-authorization callback; the user approves. `check-order-status` confirms completion.
3. Nothing to connect by hand. A domain registered this way has its zone created pointing at the edge, so the connection starts propagating immediately.
4. `check-connection-status` until the job reports `completed`.

For a domain the user already owns, step 3 is the interesting one: `discover-provider` fingerprints the DNS host and reports which rails are available, then `connect-domain` returns a `link` the agent hands to the user and a `jobId` the agent polls. Behind that link the flow takes the best rail available at that provider: one-click authorization (typically about 30 seconds to live), a scoped API token, or guided manual where the records are rendered and a poller waits for them. `check-connection-status` returns the same vocabulary in all three cases, so the agent's control flow does not care which rail ran. If the user fixes something on their side, `reapply-connection` re-pushes a managed connection without starting over.

### Status vocabulary, and the one place it differs from REST

`check-connection-status` reports the job vocabulary: `pending`, `propagating`, `completed`, `failed`, `error`, `expired`. The REST `status` field on a connection reports `pending`, `propagating`, `live`, `failed`. They are the same lifecycle with one word different: the MCP job's `completed` is the connection's `live`. If your agent reads both surfaces, branch on the vocabulary of whichever one you called rather than normalizing them by hand and getting it backwards. The REST lifecycle, including the `error_code` values on a failure, is documented in [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md).

## Why hosted MCP instead of custom wrappers

You could wrap the [REST API](03-programmatic-domain-connection-api.md) in your own function-calling schema, and for backend pipelines that is often right. The hosted MCP server earns its place when the agent is the caller: tool descriptions, input schemas, and status semantics are maintained server-side and improve without you redeploying anything, and any MCP-capable client (Claude Code, Claude Desktop, Cursor, or your own product's agent runtime) gets the whole surface from one config block.

The honest counter-case: the MCP surface is deliberately narrower than the REST API. Twelve intent-shaped tools against 67 paths means several things are simply not reachable over MCP, including webhook registration, the `/v1/ssl*` certificate-management surface, the `/v1/connections/{id}/power*` reverse-proxy surface, `POST /v1/monitor:check`, and the integrator-supplied record set at `PUT /v1/connections/{id}/records`. If your pipeline needs those, it needs HTTP, and mixing the two is fine: both hit the same control plane and the same connection ids.

Discovery metadata for agents is published, though not all of it on the MCP host. The agent index is at [docs.customdomain.ai/docs/llms.txt](https://docs.customdomain.ai/docs/llms.txt), the OpenAPI 3.1 description of the REST API is served at [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json), and the MCP host serves OAuth metadata at `https://mcp.customdomain.ai/.well-known/oauth-authorization-server`. The MCP host does not serve an `/llms.txt` of its own; if you have seen us claim otherwise, use the docs URL above.

One capability is built but not yet on: delegated agent tokens, meaning short-lived ES256 tokens verified locally against the control plane's JWKS and scope-enforced per tool, for a human owner delegating access to an agent. It is implemented and tested and not enabled on the hosted service. Until it is, use an application-scoped API key or the client-credentials JWT.

The positioning page at [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents) summarizes the same story from the product side, and there is a walkthrough guide at [connect domains with AI agents](https://customdomain.ai/guides/connect-domains-with-ai-agents).

---

Next: [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md) · Previous: [Agents that ship websites need domains](01-agents-that-ship-websites-need-domains.md) · Back to the [overview](../README.md)
