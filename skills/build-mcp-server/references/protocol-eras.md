# MCP Protocol Eras

The released `2026-07-28` protocol begins a new wire-protocol era. Revisions
through `2025-11-25` share an initialization/session-oriented shape and are
called the legacy era here. A server may support both, but it must implement the
rules of the selected revision rather than mixing behaviors.

## Contents

- Modern `2026-07-28` invariants
- Legacy era through `2025-11-25`
- Official TypeScript SDK v2 negotiation behavior
- Deprecation timing
- Compatibility decision

## Modern `2026-07-28` invariants

### Stateless requests

- There is no `initialize` / `notifications/initialized` handshake.
- Every request includes these fields in `params._meta`:
  - `io.modelcontextprotocol/protocolVersion`
  - `io.modelcontextprotocol/clientCapabilities`
- Clients should also include `io.modelcontextprotocol/clientInfo`.
- Servers should include `io.modelcontextprotocol/serverInfo` in result `_meta`.
- All successful core wire results include `resultType`: normally `complete`,
  or `input_required` for an MRTR interim result. Negotiated extensions may
  define additional discriminators, such as Tasks' `task` result.
- Connection or process identity is not a session, user, conversation, or
  capability boundary.

Servers must implement `server/discover`. It returns supported protocol
versions, server capabilities, server identity, optional instructions, and
cache hints. Clients may call it first or send another request and handle
`UnsupportedProtocolVersion` (`-32022`).

Modern servers must emit `resultType`. A client reading an earlier-protocol
response without it treats the result as `complete`; that compatibility rule is
for consuming legacy results, not permission for a modern server to omit it.

### Streamable HTTP

The modern endpoint accepts POST. Each POST contains exactly one JSON-RPC
request or notification. For a JSON-RPC request POST, the standard headers are:

- `Accept: application/json, text/event-stream`
- `MCP-Protocol-Version`, matching the body `_meta`
- `Mcp-Method`, matching the JSON-RPC method
- `Mcp-Name` for `tools/call`, `resources/read`, and `prompts/get`, matching
  `params.name` or `params.uri`

The core `2026-07-28` revision does not define the MCP-header requirements for
client-to-server notification POSTs. Do not present the request-POST rules above
as notification requirements unless documenting an explicitly stricter
implementation convention.

The server returns either one JSON object or an SSE stream scoped to that
request. HTTP GET and DELETE, `Mcp-Session-Id`, `Last-Event-ID`, SSE replay,
and session affinity are not part of this revision. GET/DELETE on a
modern-only MCP endpoint should return 405.

Servers must validate `Origin` when present. If infrastructure routes or
authorizes using mirrored MCP headers, the component that parses the body must
also verify that header and body values match. A mismatch returns HTTP 400 with
`HeaderMismatch` (`-32020`).

An accepted client notification receives HTTP 202 with no body. An unknown RPC
method receives HTTP 404 with JSON-RPC `Method not found` (`-32601`). A modern
request missing required metadata or headers receives HTTP 400 rather than
falling through to a legacy interpretation.

Tool parameters annotated with JSON Schema `x-mcp-header` are mirrored as
`Mcp-Param-*` headers on HTTP. Use this only for primitive routing/policy values,
never secrets or PII, and let a conforming SDK enforce the encoding and
header/body checks.

### Server-to-client interaction

The server does not send JSON-RPC requests to the client. If `tools/call`,
`resources/read`, or `prompts/get` needs elicitation, roots, or sampling, it
returns an `InputRequiredResult`:

- `resultType: "input_required"`
- `inputRequests`: keyed embedded requests supported by the client's declared
  capabilities
- optional `requestState`: an opaque string echoed on retry

The client gathers responses and retries the original method with a new
JSON-RPC ID, `inputResponses`, and the exact `requestState`. Treat returned
state as attacker-controlled: integrity-protect it and bind it to the principal,
method/arguments, and a short expiry when tampering or replay matters.

### Change notifications

The modern protocol replaces the HTTP GET stream and
`resources/subscribe`/`resources/unsubscribe` with `subscriptions/listen`.
The client explicitly selects tool-list, prompt-list, resource-list, and/or
resource-URI events. The first stream message acknowledges the accepted filter;
all subsequent messages carry
`_meta.io.modelcontextprotocol/subscriptionId`.

Progress and the deprecated MCP logging notifications are request-scoped and
stay on the response stream of the request that caused them. They do not use the
subscription stream.

Core `ping`, `logging/setLevel`, and `notifications/roots/list_changed` were
removed. Use an ordinary transport/application health check, per-request
`_meta.io.modelcontextprotocol/logLevel` when maintaining deprecated MCP
logging, and application-specific invalidation for configured paths.

### Caching and schemas

Successful `server/discover`, `tools/list`, `prompts/list`, `resources/list`,
`resources/templates/list`, and `resources/read` results require:

- `ttlMs`: a non-negative freshness hint in milliseconds
- `cacheScope`: `public` or `private`

Use `private` whenever a result depends on caller identity or authorization.
List pages are cached independently, and a change notification invalidates the
relevant cached response.

MCP defaults JSON Schema to 2020-12. Implementations must not fetch network
`$ref` targets automatically. If opt-in fetching exists, use strict SSRF,
timeout, size, host, and recursion defenses.

### Protocol-defined errors

The modern MCP-specific server-error range starts at `-32020`:

| Code     | Meaning                                                                      |
| -------- | ---------------------------------------------------------------------------- |
| `-32020` | HTTP header/body mismatch or required header missing/malformed               |
| `-32021` | Required client capability was not declared on the request                   |
| `-32022` | Requested protocol version is unsupported; response lists supported versions |

`-32000` through `-32019` are the legacy implementation-defined MCP subrange.
New codes MUST NOT be allocated there, and new implementations SHOULD NOT emit
codes from it. In particular, modern servers must not emit the historical
`-32002` resource-not-found or `-32042` URL-elicitation errors. The entire
`-32020` through `-32099` range is reserved for future specification-defined
errors. Use standard JSON-RPC errors for malformed requests and methods; expose
expected tool/domain failures as tool results. If an implementation-specific
JSON-RPC error is unavoidable, choose a code outside both MCP-reserved ranges.

### Trace context

The `_meta` keys `traceparent`, `tracestate`, and `baggage` carry OpenTelemetry
trace context using the W3C formats. Propagate them according to policy without
copying credentials, tool payloads, or private content into spans.

### Deprecations

New `2026-07-28` servers should not adopt:

- Roots: pass paths/resources through arguments, URIs, or configuration.
- Sampling: call an LLM provider directly from the server.
- MCP Logging: use stderr for stdio and OpenTelemetry for observability.
- Dynamic Client Registration: prefer Client ID Metadata Documents; keep DCR
  only as a tested compatibility fallback.
- HTTP+SSE: use Streamable HTTP.

Tasks moved out of core into the `io.modelcontextprotocol/tasks` extension.
Extensions are negotiated through the `extensions` maps in client/server
capabilities and need an explicit fallback when the peer does not support them.
The Tasks extension uses `resultType: "task"`, `tasks/get`, `tasks/update`, and
`tasks/cancel`; it removed the old blocking `tasks/result` and `tasks/list`.

## Legacy era through `2025-11-25`

Legacy versions use an `initialize` handshake and different transport behavior.
Depending on the selected revision they may use protocol-level HTTP sessions,
GET/DELETE on the Streamable HTTP endpoint, server-to-client JSON-RPC requests,
resource subscription methods, or SSE replay.

Keep those behaviors inside a legacy SDK path. A dual-era implementation should
use an entry point that classifies or negotiates the era and then applies the
complete rules for that era. Do not infer a modern client's capabilities from a
legacy initialization result or vice versa.

The official TypeScript v2 `createMcpHandler` serves modern and stateless legacy
HTTP requests from one endpoint by default, and `serveStdio` selects an era for
the connection. Use `legacy: "reject"` for modern-only serving. A sessionful
legacy HTTP implementation still needs a separate legacy handler in front of a
strict modern entry.

Normatively, a well-formed modern response can prove that the server understands
the modern protocol even when it reports an error. A dual-era client should
correct the modern request or version rather than treating every HTTP `400` as
permission to fall back. Clients should cache the selected era for the server
process or HTTP origin and re-probe when the assumption fails. A legacy-only
client has no fall-forward mechanism, so a modern-only server should name its
supported versions when rejecting `initialize`.

Servers may serve both eras concurrently, but each request/connection stays
inside one complete behavior family.

## Official TypeScript SDK v2 negotiation behavior

Do not confuse the normative rule above with the stable v2 client's conservative
automatic probe classifier. In `@modelcontextprotocol/client@2.0.0`, a valid
`server/discover` result or a well-formed `UnsupportedProtocolVersion`
(`-32022`) response is modern evidence. `HeaderMismatch` (`-32020`) and
`MissingRequiredClientCapability` (`-32021`) can still classify as legacy
fallback.

The v2 client also defaults to legacy behavior unless `versionNegotiation` is
set explicitly. Therefore:

- pin `2026-07-28` in modern integration tests
- test the raw modern wire shape independently of automatic negotiation
- test automatic dual-era fallback as a separate compatibility behavior
- record any dependence on this SDK-specific classifier in the version ledger

## Deprecation timing

Deprecated does not mean removed. Roots, Sampling, Logging, and DCR first become
eligible for removal in a revision released on or after `2027-07-28`; actual
removal is a later Core Maintainer decision. Sampling's
`includeContext: "thisServer"` and `"allServers"` values are also deprecated;
omit the field or use `"none"`. HTTP+SSE follows its separate registry timeline.
Always check the current deprecated-features registry when planning support.

## Compatibility decision

| Required clients                                          | Recommendation                                                               |
| --------------------------------------------------------- | ---------------------------------------------------------------------------- |
| All support `2026-07-28`                                  | Modern-only server                                                           |
| Mix of modern and 2025-era clients                        | SDK-supported dual-era server; test both paths                               |
| Only legacy clients and stability outweighs modernization | Maintain the legacy target explicitly and plan migration                     |
| Client support is unknown                                 | Probe/test the real host before choosing; do not guess from marketing claims |

The protocol specification is the authority for wire behavior. SDK documentation
is the authority for how a particular SDK realizes that behavior. When they
appear to conflict, first confirm the target protocol revision: a correct 2025
example is still wrong for a `2026-07-28` endpoint.
