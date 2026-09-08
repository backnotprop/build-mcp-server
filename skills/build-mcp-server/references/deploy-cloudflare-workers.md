# Deploy a Modern MCP Server to Cloudflare Workers

Use the official TypeScript SDK v2 web-standard handler for a
`2026-07-28`-era Worker. A Cloudflare `McpAgent`/Durable Object template that
implements protocol sessions may still be useful for 2025-era compatibility,
but it must not be presented as the current stateless transport without
explicit modern-protocol support.

## Contents

- Bootstrap and Worker entry
- Listen and application state
- Host/Origin policy
- Authorization
- Compatibility gate

## Bootstrap

Create a TypeScript Worker, then install a matched v2 SDK line and Zod:

```bash
npm create cloudflare@latest -- my-mcp-server
cd my-mcp-server
npm install @modelcontextprotocol/server zod
```

Check `@modelcontextprotocol/server` package tags first. This reference targets
the stable v2 line.

## `src/index.ts`

```typescript
import {
  createMcpHandler,
  hostHeaderValidationResponse,
  McpServer,
  originValidationResponse,
} from "@modelcontextprotocol/server";
import * as z from "zod/v4";

interface Env {
  ALLOWED_HOSTS: string;
  ALLOWED_ORIGIN_HOSTS: string;
}

const ITEMS = [
  { id: "item_001", title: "First item" },
  { id: "item_002", title: "Second item" },
];

function csv(value: string): string[] {
  return value
    .split(",")
    .map((part) => part.trim())
    .filter(Boolean);
}

function buildServer() {
  const server = new McpServer(
    { name: "my-service", version: "0.1.0" },
    {
      instructions:
        "Use search_items to discover IDs before requesting details.",
      cacheHints: {
        "tools/list": { ttlMs: 60_000, cacheScope: "public" },
      },
    },
  );

  server.registerTool(
    "search_items",
    {
      title: "Search items",
      description:
        "Search items by keyword and return IDs for follow-up calls.",
      inputSchema: z.object({
        query: z.string().min(1).describe("Keywords to search for"),
        limit: z.number().int().min(1).max(50).default(10),
      }),
      annotations: {
        readOnlyHint: true,
        destructiveHint: false,
        openWorldHint: false,
      },
    },
    async ({ query, limit }) => {
      const needle = query.toLowerCase();
      const results = ITEMS.filter((item) =>
        item.title.toLowerCase().includes(needle),
      ).slice(0, limit);
      return {
        content: [{ type: "text", text: JSON.stringify({ results }) }],
        structuredContent: { results },
      };
    },
  );

  return server;
}

const handler = createMcpHandler(buildServer, {
  legacy: "reject",
  maxSubscriptions: 0,
});

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname !== "/mcp") {
      return new Response("Not found", { status: 404 });
    }

    const rejected =
      hostHeaderValidationResponse(request, csv(env.ALLOWED_HOSTS)) ??
      originValidationResponse(request, csv(env.ALLOWED_ORIGIN_HOSTS));
    if (rejected) return rejected;

    return handler.fetch(request);
  },
};
```

`createMcpHandler` creates a fresh server per request. No Durable Object,
protocol session, session routing, GET stream, DELETE endpoint, or SSE replay is
needed for the modern core protocol. Use Durable Objects only for actual
application state or multi-instance notification fan-out, with explicit state
handles and authorization, not to recreate `Mcp-Session-Id`.

This minimal Worker explicitly rejects `subscriptions/listen` by setting
`maxSubscriptions: 0`; it must not open SSE when the application has no event
source. To enable listen, supply a real `ServerEventBus`, a positive bounded
capacity, cancellation/reconnect tests, and cross-isolate fan-out. An in-memory
bus is insufficient across Worker isolates; a Durable Object or another shared
event backbone can own that application concern.

## Configure and deploy

Set `ALLOWED_HOSTS` to the deployed Worker hostname. The SDK
`originValidationResponse` helper is a hostname allowlist and ignores scheme and
port, so `ALLOWED_ORIGIN_HOSTS` intentionally contains bare hostnames. Include
the local hostname in a development environment only when needed. If policy
must distinguish HTTPS from HTTP or production from a development port,
validate the complete serialized `Origin` value before calling the hostname
helper.

```bash
npx wrangler dev
npx wrangler deploy
```

Store upstream secrets with Worker secrets, not `vars` committed to source:

```bash
npx wrangler secret put UPSTREAM_API_KEY
```

## Authorization

Workers are a web-standard `fetch` host, so use the web-standard gate from
`@modelcontextprotocol/server` rather than a framework middleware:

```typescript
import { createMcpHandler, requireBearerAuth } from "@modelcontextprotocol/server";

const gate = requireBearerAuth({ verifier, requiredScopes: ["mcp"] });

// inside fetch, after the Host/Origin checks:
const auth = await gate(request);
if (auth instanceof Response) return auth;
return handler.fetch(request, { authInfo: auth });
```

The gate resolves to the verified `AuthInfo` or to a ready-to-return challenge
`Response`. Handlers read the caller as `ctx.http.authInfo`, which is
`undefined` when the same server runs over stdio, so guard the read if it serves
both. `requiredScopes` is endpoint-wide; a scope only some tools need is a check
inside the handler returning `isError: true`, so the model reads the refusal
instead of losing the call.

You supply `verifyAccessToken`: raw token in, `AuthInfo` out. Always populate
`expiresAt`, or the gate answers `401 invalid_token` for every request.

Serve OAuth Protected Resource Metadata at the path-aware RFC 9728 location and
point the `WWW-Authenticate` challenge at it. `oauthMetadataResponse` from
`@modelcontextprotocol/server` serves both well-known documents from a `fetch`
handler and falls through when the path does not match. Note that it ships
permissive, reflecting CORS; if you want a fixed allow-headers policy, serve the
documents yourself.

An authorization-server library can issue tokens, but it does not replace the
MCP resource-server checks: validate expiry, issuer, audience/resource, scopes,
and caller/tenant binding. Client ID Metadata Documents are the preferred open
registration path in `2026-07-28`; Dynamic Client Registration is deprecated
and should exist only as a tested compatibility fallback.

If the Worker is also the authorization server, everything above is the
resource-server half only. The rest, including the endpoint inventory, the CORS
rules for browser-hosted clients, registration hardening, consent, and
revocation, is in `authorization-server.md`. Two Workers-specific notes: a
per-IP rate limiter on the registration and token doors is a binding you must
declare, and the limiter must run before any parse or storage write, which is
also what makes it assertable in a test.

## Compatibility gate for Cloudflare-specific wrappers

Before substituting `McpAgent` or another wrapper, verify that its installed
version supports:

- `server/discover` and the per-request metadata envelope
- POST-only modern Streamable HTTP and required MCP headers
- required `resultType`, `ttlMs`, and `cacheScope` fields
- MRTR `input_required` instead of direct server-to-client JSON-RPC requests
- an explicit `subscriptions/listen` behavior: tested rejection, or a real
  shared event bus with capacity and cancellation
- no modern dependency on session IDs, GET/DELETE, or `Last-Event-ID`

If it does not, keep the web-standard handler above or label the wrapper route
as a separate legacy endpoint.
