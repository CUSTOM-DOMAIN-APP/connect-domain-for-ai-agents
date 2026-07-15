# An MCP Server for Domains: Search, Register, Connect, and Secure

The Model Context Protocol (MCP) is the emerging standard for giving AI agents tools they can call directly. This document describes what a hosted MCP server for the domain lifecycle exposes, using the [Custom Domain MCP server](https://customdomain.ai/mcp-server) as the concrete example: domain search and registration, connection, ownership verification, TLS status, and email routing, all as first-class tools an agent can call without wrapper code.

## Endpoint and authentication

The server is hosted; there is nothing to run or deploy on your side.

| Item | Value |
|---|---|
| Endpoint | `https://mcp.customdomain.ai/mcp` |
| Transport | Streamable HTTP |
| Auth, option 1 | API key from the console (`sk_live_...`), scoped to one application |
| Auth, option 2 | OAuth client credentials: exchange an application ID and secret for a short-lived JWT |

The OAuth exchange, for agents that should never see a long-lived key:

```bash
curl -X POST https://mcp.customdomain.ai/token \
  -d grant_type=client_credentials \
  -d client_id=YOUR_APP_ID \
  -d client_secret=YOUR_APP_SECRET
```

## The tool surface

Eleven tools cover the lifecycle, grouped by phase:

| Phase | Tools | What they do |
|---|---|---|
| Discover and register | `search-domain-availability`, `generate-domain-suggestions`, `create-domain-order`, `check-order-status` | Check names, suggest alternatives, purchase, track the order |
| Connect | `connect-domain`, `discover-provider`, `check-connection-status`, `reapply-connection`, `disconnect-domain` | Start a connection, fingerprint the user's DNS provider, poll the state machine, retry after a fix, clean up |
| Route mail and traffic | `add-email`, `forward-domain` | Email routing and domain forwarding on a connected domain |

One design decision matters more than the tool list: **there is no tool that writes raw DNS records.** Record names and values are computed server-side from vetted configuration for each provider. An agent can ask for a domain to be connected; it cannot ask for `acmebakery.com` to be pointed at an arbitrary IP through a crafted record. If a prompt injection reaches the agent, the worst available DNS outcome is a failed connection, not a hijacked zone. The full reasoning is in [Security for agent-driven DNS](04-security-for-agent-driven-dns.md).

Purchases are similarly guarded: `create-domain-order` is fail-closed and requires an explicit authorization callback before money moves, so an autonomous agent cannot complete a purchase alone.

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

Working client examples live in the [customdomain-mcp repository on GitHub](https://github.com/ever-just/customdomain-mcp), and the full tool reference is at [app.customdomain.ai/docs](https://app.customdomain.ai/docs).

## What a session actually looks like

A typical "make it live on a real domain" exchange, as tool calls:

1. `search-domain-availability` for `acmebakery.com`. Taken. `generate-domain-suggestions` returns alternatives; the user picks one.
2. `create-domain-order`. The order pauses for the human authorization callback; the user approves the purchase. `check-order-status` confirms completion.
3. `connect-domain` with the hostname and the platform target. Because the domain was just registered through the platform, DNS is already under management and there is nothing for anyone to paste.
4. `check-connection-status` until the state reaches `live`, which includes ownership verification and TLS issuance at the edge.

For a domain the user already owns, step 3 changes shape: `discover-provider` fingerprints the DNS host, and the connection proceeds through the best available method, one-click provider authorization (typically about 30 seconds to live), a scoped API token, or a guided manual flow where the agent relays the exact records and verification polls automatically. In every case `check-connection-status` returns the same deterministic states, so the agent's control flow does not care which method was used. If the user fixes something on their side, `reapply-connection` retries without starting over.

## Why hosted MCP instead of custom wrappers

You could wrap the [REST API](03-programmatic-domain-connection-api.md) in your own function-calling schema, and for backend pipelines that is often right. The hosted MCP server earns its place when the agent is the caller: tool descriptions, input schemas, and state semantics are maintained server-side and improve without you redeploying anything; any MCP-capable client (Claude Code, Claude Desktop, Cursor, or your own product's agent runtime) gets the whole surface from one config block; and the server publishes discovery metadata (`/llms.txt` and an OpenAPI 3.1 description) so agents can orient themselves.

The positioning page at [customdomain.ai/for/ai-agents](https://customdomain.ai/for/ai-agents) summarizes the same story from the product side: the domain step, as tool calls, with zero browser tabs in the loop.

---

Next: [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md) · Previous: [Agents that ship websites need domains](01-agents-that-ship-websites-need-domains.md) · Back to the [overview](../README.md)
