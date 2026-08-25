---
name: app-mcp-composability
description: Exposing an operation as a Model Context Protocol tool (mcp.description/inputSchema), the one aggregated /__zanix-mcp endpoint, allowedCallers-based auth reused from app-to-app calls (permissive by default once a tool is exposed), and real spec gaps (no listChanged, no true 202 response). Use when exposing an operation to an AI agent, or reviewing an existing MCP-exposed tool's authorization.
---

Covers `@zanix/app`'s Model Context Protocol integration — one aggregated,
process-wide MCP endpoint exposing opted-in `operations` as tools. For the
underlying app-to-app call/auth mechanism this reuses, see
`app-remote-calls-and-control-plane`. File:line references point at
`~/Documents/Development/ZanixLibraries/app` — read the real code there
before assuming this summary is still accurate.

## Golden rule (token savings)

- `mcp` is opt-in per operation — an operation with no `mcp` field is
  invisible to the endpoint regardless of `allowedCallers`. Check that
  first before investigating any MCP-exposure question.
- Once an operation IS exposed via `mcp`, treat `allowedCallers` as the
  real gate — its absence means any authenticated MCP client can invoke it.

## Exposing an operation as a tool

```ts
operations: {
  refundOrder: {
    handler: async (payload, ctx) => ({ refunded: true }),
    mcp: {
      description: 'Refunds a customer order given an order id.',
      inputSchema: { type: 'object', properties: { orderId: { type: 'string' } }, required: ['orderId'] },
    },
    allowedCallers: ['agent:claude-desktop'],
  },
}
```

`registerMcpServer()` is idempotent (a module-level guard) and registers a
dedicated `Application` named `__zanix-mcp`: `POST
/__zanix-mcp/service-token` wraps `@zanix/auth`'s
`exchangeServiceCredential`; `POST /__zanix-mcp` is protected by
`@AuthTokenValidation({type: 'api'})`, then routes into
`handleMcpRequest(body, callerAppName)` where `callerAppName` is the
exchanged token's `sub` claim.

Supported JSON-RPC methods: `initialize` (returns `protocolVersion:
'2025-06-18'`, `capabilities: {tools: {}}`), `notifications/initialized`,
`tools/list` (no pagination — returns every exposed tool at once),
`tools/call` (tool name format `${appName}.${operationName}`).

## Authorization: the same `allowedCallers` mechanism, applied to the agent's identity

The MCP client's own `serviceId` (e.g. `agent:claude-desktop`) is checked
against a tool's `allowedCallers` via the identical mechanism
app-to-app calls use (`app-remote-calls-and-control-plane`) — same env
config (`JWK_PUB_<serviceId>`/`SERVICE_PERMISSIONS_<serviceId>`).

**Security-relevant, permissive default**: **omitting `allowedCallers` on a
tool that IS exposed lets any authenticated MCP client invoke it.** This
mirrors the ordinary app-to-app default (opt-in restriction, not
opt-in access) — don't assume exposing an operation to MCP is itself a
narrowing of who can call it; it's the opposite unless `allowedCallers` is
set explicitly.

**Honest limitation**: authorization here is app-to-app-shaped, applied to
the calling agent's own identity — it says nothing about which human
triggered the call through that agent.

## What's not implemented (real spec gaps, not planned omissions to route around)

Checked against the MCP spec (`2025-06-18`) directly — not implemented:
`resources`/`prompts`/`logging` capabilities, pagination, `Mcp-Session-Id`
session management, SSE streaming (single non-streaming `application/json`
body only).

**`listChanged` is not implemented** — a real staleness footgun for a
long-lived agent session: an agent has no way to learn that a hot
install/uninstall changed the available tool set until it re-calls
`tools/list` itself. Don't assume an agent's cached tool list stays current
across a hot install (`app-hot-install-and-multitenancy`).

**A real, documented deviation from the notification-response spec**: the
spec calls for an HTTP 202 with no body for a notification, but this
framework's own handler contract has no shape for "202, empty" — the real
code returns a 200 with an empty JSON object instead of true 202. An MCP
client that strictly checks for 202 could misbehave against this endpoint.

## Checklist before exposing/reviewing an MCP-exposed operation

- [ ] Does the operation's `mcp.inputSchema` actually match what `handler`
      expects — hand-authored, since `@zanix/validator`'s `BaseRTO`
      decorators are imperative, not introspectable into a JSON Schema?
- [ ] Is `allowedCallers` set deliberately for any tool with real
      side-effects — not left implicitly open to every authenticated MCP
      client?
- [ ] Does anything consuming this endpoint assume `listChanged`
      notifications or a true 202 response — neither is implemented, and
      both would need a workaround (manual `tools/list` re-polling, HTTP
      200 handling) on the client side?
