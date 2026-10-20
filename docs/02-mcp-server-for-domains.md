# An MCP Server for Domains: Search, Register, Connect, and Secure

The Model Context Protocol (MCP) is the standard for giving AI agents tools they can call directly. This document describes what a hosted MCP server for the domain lifecycle exposes, using the [CustomDomain™ MCP server](https://customdomain.ai/mcp-server) as the concrete example: domain search and registration, connection, propagation status, email routing, and forwarding, all as first-class tools an agent can call without wrapper code.

## Endpoint and authentication

The server is hosted; there is nothing to run or deploy on your side. It is listed in the official MCP registry as `ai.customdomain/mcp`.

| Item | Value |
|---|---|
| Endpoint | `https://mcp.customdomain.ai/mcp` |
| Transport | Streamable HTTP, JSON-RPC 2.0 |
| Server identity | `customdomain-mcp` version 0.5.0, protocol revision `2025-06-18`, advertises only the `tools` capability |
| Auth, option 1 | API key from the console (`cd_live_...` or `cd_test_...`; older `sk_live_...` keys keep working), which acts for the whole workspace |
| Auth, option 2 | OAuth client credentials: exchange an application id and client secret for a JWT that lasts 15 minutes |
| Auth, option 3 | The `mcp-remote` shortcut form `CLIENT_ID:CLIENT_SECRET`, which the server exchanges for a JWT on the fly |
| Auth, option 4 | A delegated agent token: the client follows the `401` to the OAuth metadata, a person signs in at `https://api.customdomain.ai` and approves scopes (authorization code with PKCE), and the server checks the tool's scope on every call |

The OAuth exchange, for agents that should never see a long-lived key:

```bash
curl -X POST https://mcp.customdomain.ai/token \
  -u "APPLICATION_ID:CLIENT_SECRET" \
  -d "grant_type=client_credentials"
# → { "access_token": "eyJ…", "token_type": "Bearer", "expires_in": 900 }
```

For the first three, the server does not validate credentials cryptographically itself. It forwards the caller's credential to the control plane, which enforces the same permissions as the REST API. Revoke the key in the console and the agent loses access on the next call. Delegated agent tokens are the exception: the server verifies them locally against the control plane's JWKS and refuses a call whose tool needs a scope the person did not approve. The protected-resource metadata is at `https://mcp.customdomain.ai/.well-known/oauth-protected-resource`, and the `client_credentials` authorization-server metadata at `https://mcp.customdomain.ai/.well-known/oauth-authorization-server`.

## The tool surface

Seventeen tools cover the lifecycle, registered in a fixed order. Arguments are listed because agent authors need them to write prompts and schemas that do not guess. Each tool needs one scope, which matters when a person approves a delegated agent: `domains:read` looks and changes nothing, `domains:connect` sets a domain up, `domains:disconnect` takes one offline, and `domains:purchase` spends money.

| Phase | Tool | Arguments | Scope | What it does |
|---|---|---|---|---|
| Discover and register | `search-domain-availability` | `{ domain }` | read | Availability with live purchase and renewal pricing |
| | `generate-domain-suggestions` | `{ keywords, limit? }` | read | Available-to-register name ideas, each priced; default 5, max 20 |
| | `create-domain-order` | `{ domain, confirm_domain }` | purchase | Start a registration through the resolved registrar. Off on the hosted service |
| | `check-order-status` | `{ orderId }` or `{ jobId }` | read | Read the live status of a registration order; supply exactly one |
| | `list-domain-orders` | `{ limit? }` | read | The account's domain purchases, newest first |
| Connect | `discover-provider` | `{ domain }` | read | Detect where DNS is hosted and which setup rails can run for it |
| | `connect-domain` | `{ domain, applicationId? }` | connect | Attach a domain the user owns. Returns the exact `records`, a `link` for the user and a `connectionId` to poll (also sent as `jobId`) |
| | `check-connection-status` | `{ connectionId }` or `{ jobId }` | read | Read the live status of a connection and its records |
| | `diagnose-connection` | `{ connectionId }` | read | Explain, record by record, what public DNS shows (`found`, `missing`, `mismatch`) and what the user still has to do |
| | `recheck-connection` | `{ connectionId }` | connect | Re-run the DNS check now instead of waiting for the next scheduled one |
| | `check-certificate-status` | `{ connectionId }` | read | Issuer, expiry, days left, and whether renewal is failing |
| | `reapply-connection` | `{ connectionId }` | connect | Re-push a managed connection's DNS through its stored grant |
| | `disconnect-domain` | `{ connectionId, confirm }` | disconnect | Take the domain offline. `confirm` is the domain the person confirmed. Records come out only where a kept grant wrote them |
| | `list-connections` | `{ status?, limit?, cursor? }` | read | Inventory every connection on the account, page by page |
| Route mail and traffic | `add-email` | `{ domain?, connectionId?, provider?, mxHost?, spfInclude?, dkimSelector?, dkimTarget?, dmarcRua? }` | connect | Configure MX, SPF, DKIM and DMARC in one call via server-side templates. `provider` (`google`, `microsoft365`, `zoho`) fills in the standard MX and SPF |
| | `forward-domain` | `{ target, domain?, connectionId? }` | connect | Permanent 301 forward, via a managed one-click setup flow |
| Account | `check-billing-status` | none | read | Whether the workspace may connect new domains right now, with the plan and this month's usage |

One design decision matters more than the tool list: **no tool accepts DNS records as input.** Record names and values are computed by the control plane from vetted per-provider templates. An agent can ask for a domain to be connected, or for Google Workspace mail to be configured; it cannot ask for `acmebakery.com` to be pointed at an arbitrary IP through a crafted record. If a prompt injection reaches the agent, the worst available DNS outcome is a failed connection, not a hijacked zone. The full reasoning is in [Security for agent-driven DNS](04-security-for-agent-driven-dns.md).

If you have read older copy of ours describing these tools as "manage DNS records," that description was wrong in a way that matters, because it invites an integrator to build exactly the injection path the design closes.

Purchases are similarly guarded: `create-domain-order` places a paid order only after a successful call to the integrator's purchase-authorization callback. If that callback is not configured, every purchase is denied. That is the safe default, not a bug. On the hosted service the direct purchase route behind the tool is also switched off, so purchases through MCP are off there today; buying a domain is live through the console's checkout.

One more thing worth knowing before you write prompt text: `generate-domain-suggestions` is deterministic, not a model call. Candidates are keyword joins plus a few common affixes, expanded across TLDs by registrar search and filtered to what is actually available. The same keywords always produce the same candidates, which is what makes the tool testable, and also means it will not surprise you with a clever name.

## Client configuration

**Claude Code** (one command):

```bash
claude mcp add --transport http customdomain https://mcp.customdomain.ai/mcp \
  --header "Authorization: Bearer $CUSTOMDOMAIN_API_KEY"
```

**Claude Desktop**: add a custom connector (**Customize > Connectors > Add custom connector**) with the URL `https://mcp.customdomain.ai/mcp`. A desktop that only starts local programs can still bridge through `mcp-remote`:

```json
{
  "mcpServers": {
    "customdomain": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://mcp.customdomain.ai/mcp",
        "--header", "Authorization: Bearer cd_live_YOUR_KEY"
      ]
    }
  }
}
```

**Cursor** (the user-level `~/.cursor/mcp.json`, so the key stays out of the project):

```json
{
  "mcpServers": {
    "customdomain": {
      "url": "https://mcp.customdomain.ai/mcp",
      "headers": { "Authorization": "Bearer cd_live_YOUR_KEY" }
    }
  }
}
```

**VS Code and GitHub Copilot cloud agent**: see [MCP setup](../README.md#mcp-setup-for-claude-cursor-vs-code-and-github-copilot) in the README. Copilot cloud agent cannot sign in with OAuth, so it sends the credential from an Agents secret.

Working client examples live in the [customdomain-mcp repository on GitHub](https://github.com/CUSTOM-DOMAIN-APP/customdomain-mcp), and the full tool reference is at [docs.customdomain.ai/docs/mcp/overview](https://docs.customdomain.ai/docs/mcp/overview).

## What a session actually looks like

A typical "make it live on a real domain" exchange, as tool calls:

1. `search-domain-availability` for `acmebakery.com`. Taken. `generate-domain-suggestions` returns available alternatives; the user picks one.
2. `create-domain-order` where a deployment has purchases on: the order waits on the purchase-authorization callback, the user approves, and `check-order-status` confirms completion. On the hosted service purchases through MCP are off, so the user buys the domain in the console instead.
3. Nothing to connect by hand. A domain registered this way has its zone created pointing at the edge, so the connection starts propagating immediately.
4. `check-connection-status` until it returns `connected: true` (status `live`).

For a domain the user already owns, step 3 is the interesting one: `discover-provider` fingerprints the DNS host and reports which rails are available, then `connect-domain` returns the records, a `link` the agent hands to the user and a `connectionId` the agent polls. Behind that link the flow takes the best rail available at that provider: one-click authorization (typically about 30 seconds to live), a scoped API token, or guided manual where the records are rendered and a poller waits for them. `check-connection-status` returns the same vocabulary in all three cases, so the agent's control flow does not care which rail ran. If a connection stalls, `diagnose-connection` says which record is missing or wrong, `recheck-connection` confirms the fix without waiting for the next scheduled check, and `reapply-connection` re-pushes a managed connection without starting over.

### Status vocabulary

`check-connection-status` returns the connection's REST `status` unchanged: `pending`, `propagating`, `live`, or `failed`. Alongside it comes a boolean `connected`, which is the reliable completion signal, so key the agent's "done" branch off `connected: true` rather than the string. `check-order-status` is the one tool with a different vocabulary, because it reports a domain order, not a connection. The REST lifecycle, including the `error_code` values on a failure, is documented in [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md).

## Why hosted MCP instead of custom wrappers

You could wrap the [REST API](03-programmatic-domain-connection-api.md) in your own function-calling schema, and for backend pipelines that is often right. The hosted MCP server earns its place when the agent is the caller: tool descriptions, input schemas, and status semantics are maintained server-side and improve without you redeploying anything, and any MCP-capable client (Claude Code, Claude Desktop, Cursor, or your own product's agent runtime) gets the whole surface from one config block.

The honest counter-case: the MCP surface is deliberately narrower than the REST API. Seventeen intent-shaped tools against 129 paths (counted 2026-10-20) means several things are simply not reachable over MCP, including webhook registration, the `/v1/ssl*` certificate-management surface, the `/v1/connections/{id}/power*` reverse-proxy surface, `POST /v1/monitor:check`, and the integrator-supplied record set at `PUT /v1/connections/{id}/records`. If your pipeline needs those, it needs HTTP, and mixing the two is fine: both hit the same control plane and the same connection ids.

Discovery metadata for agents is published, though not all of it on the MCP host. The agent index is at [docs.customdomain.ai/docs/llms.txt](https://docs.customdomain.ai/docs/llms.txt), the OpenAPI 3.1 description of the REST API is served at [api.customdomain.ai/v1/openapi.json](https://api.customdomain.ai/v1/openapi.json), and the MCP host serves OAuth metadata at `https://mcp.customdomain.ai/.well-known/oauth-protected-resource` and its version and tool list at `https://mcp.customdomain.ai/.well-known/agent/mcp.json`. The MCP host does not serve an `/llms.txt` of its own; if you have seen us claim otherwise, use the docs URL above.

Delegated agent tokens are live on the hosted service: short-lived ES256 tokens, verified locally against the control plane's JWKS and scope-enforced per tool, for a human owner delegating access to an agent. A client that cannot run the OAuth flow (Copilot cloud agent, a backend job) uses an API key or the client-credentials JWT instead; prefer the JWT, because it acts for one application rather than the whole workspace.

The positioning page at [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents) summarizes the same story from the product side, and there is a walkthrough guide at [connect domains with AI agents](https://customdomain.ai/guides/connect-domains-with-ai-agents).

---

Next: [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md) · Previous: [Agents that ship websites need domains](01-agents-that-ship-websites-need-domains.md) · Back to the [overview](../README.md)
