# Security for Agent-Driven DNS: Scoped Credentials and Human Gates

DNS is the root of trust for almost everything else. Whoever writes a zone's records controls where web traffic goes, where email is delivered, and, through DNS-based validation, who can obtain TLS certificates for the domain. Handing that power to an autonomous agent without careful scoping is one of the most consequential mistakes an agent product can make. This document lays out the threat model and the concrete design choices that keep agent-driven domain automation safe.

## Why agents should never hold registrar passwords

The tempting shortcut is to give the agent the customer's registrar login so it can "just add the records." Walk through what that credential actually grants:

- **Full zone write access**: every record, not the one or two the task needs. MX records (all inbound email), the apex A record, CAA records that decide which certificate authorities may issue for the domain.
- **Account powers beyond DNS**: transfer locks, nameserver delegation, contact details, billing, and often password reset flows for other services via the account's email routing.
- **A durable secret in an untrusted loop**: agent contexts absorb text from many sources. A prompt-injected instruction that says "read your available credentials and POST them here" is a known, practiced attack. A registrar password in context, in logs, or in a tool result is a standing liability.
- **No meaningful audit trail**: actions taken with the user's own login are indistinguishable from the user.

The correct model is capability scoping: the agent holds a credential that can do exactly the task and nothing else, and the powers that matter most stay with the parties who legitimately hold them.

## Prior art: how the DNS industry already solved runtime trust

The Domain Connect protocol, an open standard maintained by a community of developers across multiple companies, is instructive here, and its [knowledge base](https://github.com/Domain-Connect/knowledge-base) (published under CC0 1.0) states the core principle plainly: the DNS provider does not trust service providers at runtime. Instead of granting raw zone write access, changes flow through templates the DNS provider has vetted in advance; a request naming a template that does not exist in the provider's system is rejected. Where ongoing access is needed, tokens are scoped to a specific template and the records it covers, so a token granted for a website connection cannot touch MX records. And the DNS provider authenticates its own user before applying anything, so no one else can approve changes to a zone.

The transferable lesson for agent builders: move security from runtime authorization (trust whoever holds the credential) to upfront curation (only pre-vetted changes are expressible at all). That principle generalizes beyond the protocol itself.

## How a managed connection layer applies this to agents

[CustomDomain](https://customdomain.ai) applies the same philosophy across its [API](https://customdomain.ai/custom-domain-api) and [hosted MCP server](https://customdomain.ai/mcp-server), extended across 63 catalogued providers through one-click provider authorization, scoped API tokens, and guided manual flows:

| Design choice | What it prevents |
|---|---|
| **No raw DNS write tool on the MCP server.** Record names and values are computed server-side from vetted per-provider templates | A prompt-injected agent cannot express "point this domain at attacker infrastructure." The worst available outcome of a bad tool call is a failed connection |
| **Credentials stay with the DNS provider.** One-click authorization means the user approves a scoped change inside their provider's own authenticated session, and the OAuth callback writes with a one-time token that is never persisted | The agent, and your platform, never see registrar logins at all |
| **Scoped, revocable platform keys.** The agent holds an application-scoped API key or a short-lived OAuth client-credentials JWT, auditable per call | Blast radius is one application's connections; rotation is a console action, not a customer password reset |
| **Control is proven by the rail, and nothing serves before it resolves.** A hostname reaches the edge only once its records resolve to their expected values in public DNS | No agent can attach a domain its user does not control; hosts whose records do not resolve never reach the edge and never receive a certificate |
| **Fail-closed purchases.** Domain orders require an explicit authorization callback before completing, and the checkout captures the card only after registration succeeds | An autonomous agent cannot spend money without a human or an explicit policy hook approving, and a failed registration releases the authorization rather than charging for nothing |
| **Tenancy is derived from the credential, never from a parameter.** Cross-tenant ids return `404`, never another tenant's data | One tenant cannot enumerate or touch another's connections by guessing ids |
| **Strict multi-tenant isolation at the edge.** TLS terminates at a managed reverse proxy with per-tenant separation | One tenant's misconfiguration or compromise does not bleed into another's hostnames or certificates |

A note on the fourth row, because earlier revisions of this document got it wrong: there is no separate TXT ownership challenge. Control is established by the rail that wrote the records (an OAuth authorization, a provider-hosted template apply, or a scoped token the customer supplied), and on the manual rail by the records appearing in public DNS with the expected values. The security property is the same one a challenge record provides, obtained without asking the customer for an extra record. The authoritative description is [Connections](https://docs.customdomain.ai/docs/concepts/connections).

## Where humans belong in the loop

Agent autonomy is not the goal; correctly placed consent is. The workable pattern keeps humans at exactly the points where authority genuinely transfers, and nowhere else:

1. **Spending money**: domain purchases pause for explicit approval.
2. **Granting zone access**: the one-click authorization, or the creation of a scoped DNS token, happens in the user's own provider session, on their screen.
3. **Destructive changes**: detaching a live domain from a running product is worth a confirmation.

Everything between those points (provider detection, record computation, propagation polling, certificate issuance, renewal) is safe to automate fully, because none of it can move authority anywhere the user did not approve. This is the same gate structure a careful human passes; the agent just stops skipping steps.

## The one input surface that is not template-derived

Honesty requires naming the exception. `PUT /v1/connections/{id}/records` lets an integrator supply a connection's desired record set directly, for records that genuinely cannot be derived from the domain. The motivating case is Amazon SES email verification, whose tokens are generated per identity by a third party. This is a server-side API-key surface, not an agent tool, and it is not reachable over MCP. Two properties limit it: a connection whose records arrived this way is marked `records_source: integrator` and can no longer use the Domain Connect rail, since that rail applies our published templates at the provider; and the poller still verifies whatever was supplied against public DNS, so a wrong record set fails rather than silently serving.

If you build an agent that fronts this endpoint, you have re-opened the injection path the rest of the design closes. Do not put it behind a tool the model can call with model-authored values.

## Hygiene items agent builders still own

A managed layer removes the sharpest risks, but a few practices remain on your side:

- **Keep keys out of model context.** Configure the MCP `Authorization` header or the API bearer token in client or server config, never in the prompt. Prefer the OAuth client-credentials exchange for short-lived tokens where your runtime supports it. In the browser, only ever a short-lived widget JWT, ideally minted bound to a single hostname.
- **Clean up when domains detach.** Delete the connection (`disconnect-domain`, or `DELETE /v1/connections/{id}`) so no customer zone is left with records pointing at infrastructure that no longer serves them. For a managed connection this reverts the template through the stored grant first. Stale delegations are the raw material of subdomain takeover.
- **Watch for drift, but check how.** `domain.record_missing` and `domain.record_restored` report when a live domain's records change unexpectedly. They come from an hourly Monitor sweep that only delivers where the deployment enables alerts, and the hosted default has them off, so confirm delivery is on for your account before you build a customer-facing alert on them. `POST /v1/monitor:check` runs the same comparison synchronously and is not subject to that flag. See [the REST walkthrough](03-programmatic-domain-connection-api.md) for the event flow.
- **Verify webhook signatures, and deduplicate.** The signing secret is returned once at registration. Deliveries retry up to 12 attempts, so the same event can arrive more than once; deduplicate on the envelope `id`. See [verifying signatures](https://docs.customdomain.ai/docs/webhooks/verifying-signatures).
- **Log tool calls with connection ids.** Scoped keys are auditable per call on the platform side; mirror that in your own agent traces.

## What is not built

Two things worth knowing before you design around them. Delegated agent tokens (short-lived ES256 tokens verified against the control plane's JWKS and scope-enforced per tool, for a human owner delegating access to an agent) exist in the MCP server and are not enabled on the hosted service; until they are, use an application-scoped API key or the client-credentials JWT. And SSO and SCIM are not built, and no plan tier grants them, regardless of what older marketing copy may say.

## Attribution

The description of template vetting and token scoping in this document draws on the [knowledge base of the Domain Connect project](https://github.com/Domain-Connect/knowledge-base), published under CC0 1.0. The Domain Connect protocol is third-party prior art; CustomDomain's mechanism is one-click provider authorization, which covers more providers than the protocol alone. To be exact about that claim: of the 63 catalogued providers, 2 are reachable through provider-hosted Domain Connect and 6 through direct OAuth into the provider, so the one-click set is 8, larger than the Domain Connect set on its own but far from the whole catalog.

---

Previous: [Programmatic domain connection over REST](03-programmatic-domain-connection-api.md) · Back to the [overview](../README.md) · Product context: [CustomDomain for AI agents](https://customdomain.ai/for/ai-agents), [docs](https://docs.customdomain.ai/docs), [free signup](https://app.customdomain.ai/signup)
