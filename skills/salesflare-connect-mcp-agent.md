---
name: Connect an agent to the Salesflare hosted MCP server
description: >-
  Reach Salesflare as an agent over its first-party remote MCP server rather than
  the REST API — the OAuth handshake, what the grant actually covers, and where
  the REST contract still wins.
api: mcp/salesflare-mcp.yml
operations: []
generated: '2026-08-13'
method: generated
source: >-
  Live probes of https://mcp.salesflare.com/mcp and its RFC 9728 /RFC 8414
  discovery documents, plus
  https://integrations.salesflare.com/listings/model-context-protocol
---

# Connect an agent to Salesflare over MCP

Salesflare ships a **first-party hosted remote MCP server**. There is nothing to
install — no `npx`, no local process. An MCP client posts JSON-RPC straight to:

```
https://mcp.salesflare.com/mcp
```

Salesflare says it is "included in all Salesflare plans".

> A third-party npm package called `salesflare-mcp-server` exists. It is not
> Salesflare's. Use the hosted endpoint above.

## The authorization handshake

The endpoint is OAuth-gated. An anonymous `tools/list` returns:

```
HTTP/2 401
WWW-Authenticate: Bearer realm="MCP", error="invalid_token",
  error_description="See resource metadata",
  resource_metadata="https://mcp.salesflare.com/.well-known/oauth-protected-resource"
```

That is a correct RFC 9728 challenge, so a spec-compliant MCP client can complete
the flow with no manual configuration:

1. Fetch `https://mcp.salesflare.com/.well-known/oauth-protected-resource` →
   names the resource `https://mcp.salesflare.com/mcp` and the authorization
   server.
2. Fetch `https://api.salesflare.com/.well-known/oauth-authorization-server`
   (RFC 8414). Note: the OIDC Discovery path
   `/.well-known/openid-configuration` is **404** — use the RFC 8414 path.
3. Register dynamically at `https://api.salesflare.com/oidc/register`
   (RFC 7591) — no pre-registered `client_id` needed.
4. Authorization code + **PKCE S256** at `https://api.salesflare.com/oidc/auth`.
   PAR (`/oidc/request`) and DPoP (`ES256`, `EdDSA`) are both supported.
5. Exchange at `https://api.salesflare.com/oidc/token`. Request
   `offline_access` alongside `openid` if you need the connection to survive
   access-token expiry.
6. Send the token as `Authorization: Bearer …` — `bearer_methods_supported` is
   `["header"]` only.

## Know what the grant covers

The only scopes Salesflare advertises are `openid` and `offline_access`. Those
govern **identity and token lifetime, not permissions.** There is no read-only
scope and no per-object scope. Once a user approves the connection, the agent has
whatever that user has in the CRM, including writes.

So treat the consent as all-or-nothing and add your own guardrails: Salesflare's
own setup guide advises starting "with lookup and read-only questions" before
attempting modifications. Confirm with the human before any create, update or
delete.

## What the server can do

Salesflare publishes a capability list, not a tool list — `tools/list` is gated,
so the real tool names and input schemas only appear after you authenticate.
Published capabilities:

- Ask questions about your Salesflare data
- Look up accounts, contacts, opportunities, tasks, pipelines, tags, team members
- Create new accounts, contacts, opportunities, tasks
- Update CRM records and complete tasks
- Review account timelines (emails, messages, meetings, web visits, activities)
- Filter and sort CRM data to build lists, reports, workflows

`mcp/salesflare-tool-crosswalk.yml` binds each of those to the REST
`operationId`s that implement it.

## When to use REST instead

The MCP surface does not advertise deletes, custom-field administration, workflow
creation, meeting/call logging or internal-note writing. Those exist only on the
REST API — 44 of Salesflare's 71 operations have no advertised MCP capability. For
those, authenticate with an account API key (`Authorization: Bearer {APIKEY}`,
made under Settings > API keys) and call `https://api.salesflare.com` directly.
