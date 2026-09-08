# Server Capabilities and Cross-Cutting Features

Protocol revision `2026-07-28` makes server capabilities discoverable and
client capabilities per-request. Add optional features only when the server's
use case needs them, and never rely on a capability the peer did not declare.

SDK APIs are version-sensitive; confirm the installed version before copying
method names.

## Contents

- Discovery, instructions, caching, and pagination
- Subscriptions, progress, cancellation, and completion
- Extensions and deprecated features
- Capability checklist

## `server/discover`

Every modern server must implement `server/discover`. Its complete result
advertises:

- supported protocol versions
- server capabilities
- server identity in result `_meta`
- optional instructions
- `ttlMs` and `cacheScope`

A modern SDK serving entry should install this automatically from the server
factory. Test it directly; a working `tools/list` alone does not prove modern
discovery or version negotiation.

Capability and identity are not security claims. Authorize from verified HTTP
credentials, not self-reported `clientInfo` or `serverInfo`.

## Instructions

Instructions describe stable cross-tool workflows and constraints:

```typescript
const server = new McpServer(
  { name: "my-server", version: "1.0.0" },
  {
    instructions:
      "Use search_items to discover IDs before get_item. Read current state before mutations.",
  },
);
```

Keep the opening self-contained: what the server does, its discovery-first
workflow, and critical limits. Do not duplicate every tool description or try
to override host/system behavior.

Treat the string as a first-class artifact, not a comment. It is the one place
that states the intended loop across tools, and a model reads it before it reads
any individual description. A useful instructions string covers, in order:

1. What the server is for, and the starting ladder (list, then list, then read)
   for a caller who was given no identifiers.
2. The lifecycle verbs that create and destroy, with the ones that are
   irreversible or owner-only named as such.
3. The one or two counter-intuitive data rules that a caller will otherwise get
   wrong, stated as rules rather than described.
4. Restraint: when to propose a change rather than make it.
5. Concurrency discipline: which read to do before which write, and which
   version or hash field feeds the next call.
6. What to hand a human, and what to fetch out of band rather than through a
   tool result.
7. Identity: ask who you are before acting as someone.
8. **An escape hatch.** Say plainly that these tools are one door, name the
   other surfaces (the REST API and its contract, a CLI, a git endpoint), and
   point at a fetchable guide on the same origin. Tell the caller to read it
   before working around a gap in the tools. Agents route around missing tools;
   this decides whether they route somewhere correct.

Set cache hints honestly. If `server/discover` or `tools/list` results depend on
the caller's identity or authorization, they are `cacheScope: "private"`, and a
`ttlMs` of zero is the correct answer when the list can differ per caller.

## Caching

Modern complete results from these methods require `ttlMs` and `cacheScope`:

- `server/discover`
- `tools/list`
- `prompts/list`
- `resources/list`
- `resources/templates/list`
- `resources/read`

Use `ttlMs: 0` for immediately stale data. Use `cacheScope: "private"` for
anything dependent on a user, tenant, token, or granted scope. `public` permits
sharing across authorization contexts, even from an authenticated endpoint.

TTL is a freshness hint checked when the result is needed, not a polling
interval or a guarantee that data cannot change. Keep `cacheScope` identical
across every page of the same paginated list.

An input-required result is not cacheable. A request retry carrying
`inputResponses` or `requestState` must not be cached.

## Pagination

List methods use opaque cursors. Do not parse them or assume a fixed page size;
an empty string is still a valid cursor. Treat a missing `nextCursor` as the end.
Each page has its own cache hints. If a cursor becomes invalid, discard the
cached page chain and restart from the first page.

## Subscriptions

`subscriptions/listen` is the modern path for server-to-client change events.
The client opens a long-lived request and explicitly selects:

- `toolsListChanged`
- `promptsListChanged`
- `resourcesListChanged`
- `resourceSubscriptions` for named URIs

The server's first message acknowledges the accepted filter. Every message on
that stream carries `_meta.io.modelcontextprotocol/subscriptionId`, equal to the
listen request ID. Never send a notification type the client did not request.

On Streamable HTTP, the listen response is SSE and ends when the client closes
it. On stdio, cancellation uses `notifications/cancelled`. Re-establish
subscriptions after transport/process reconnect.

Make support an explicit serving decision. The official TypeScript v2
`createMcpHandler` installs a listen router even when the application advertises
no change capability. Capability advertising alone does not disable it:

- without an application notification path, set `maxSubscriptions: 0` or
  reject `subscriptions/listen` before SDK dispatch and assert it never opens
  SSE
- with support, provide a real event bus, bounded capacity, cancellation and
  reconnect behavior, and event fan-out tests
- for multi-process/isolate deployments, use a shared bus; the SDK's default
  in-memory bus cannot fan out across instances

This replaces the modern HTTP GET stream and
`resources/subscribe`/`resources/unsubscribe`. Those older behaviors belong
only to a legacy path.

## Progress

When the client includes a unique `progressToken` in request `_meta`, the server
may send request-scoped `notifications/progress` before the final response.

- progress must increase
- total is optional
- messages should be concise
- stop notifications after completion/cancellation
- rate limit updates
- do not fail the operation merely because progress cannot be emitted

On HTTP, progress stays on that request's SSE response stream. It is not a
subscription event.

## Cancellation

Long-running handlers should propagate the SDK's abort signal into fetch,
database, and child operations and check it inside long loops.

- Modern Streamable HTTP: closing the request's SSE response stream is
  cancellation. Do not POST `notifications/cancelled` for that request.
- Stdio: the client sends `notifications/cancelled` with the request ID.

Stop work, free resources, and send no later response when cancellation wins.
Handle completion/cancellation races without treating them as server faults.

## Completion

Completion offers suggestions for prompt arguments and resource template
variables through `completion/complete`.

Use it when users choose among many workspace IDs, project names, labels,
tables, or document paths. Suggestions should be relevance-sorted,
access-controlled, rate-limited, and capped. Skip it when discovery is already
easy.

## Extensions

Optional extensions are negotiated in the `extensions` maps of client and
server capabilities. Extension identifiers require a vendor-style prefix and
each extension defines its own settings object.

If the peer does not advertise an extension, fall back to core behavior or
return an appropriate error as the extension specifies. Do not put extension
messages on the core path without negotiation.

Tasks are no longer a core protocol feature. Modern task support uses the
official `io.modelcontextprotocol/tasks` extension. It adds
`resultType: "task"`, durable handles, polling through `tasks/get`, mid-flight
input through `tasks/update`, cooperative `tasks/cancel`, and optional task
notifications through `subscriptions/listen`. It removes the old blocking
`tasks/result` and `tasks/list` methods. Do not return a task unless the client
declared the extension, and provide a synchronous/core fallback when practical.

Extensions evolve independently of core MCP. Verify the installed SDK and the
extension's own version/settings rather than inferring support from core
`2026-07-28` conformance.

## Deprecated features

The following remain in `2026-07-28` during their deprecation windows but new
servers should not adopt them.

Roots, Sampling, Logging, and DCR first become eligible for removal in a
revision released on or after `2027-07-28`; eligibility is not a guaranteed
removal date. HTTP+SSE follows the separate date in the live deprecated-features
registry.

### Roots

Migration: pass directories/files through tool arguments, resource URIs, or
server configuration. Roots were only guidance, never an enforced filesystem
sandbox. Validate and authorize paths independently.

### Sampling

Migration: integrate directly with an LLM provider from the server when the
feature is truly needed. Do not assume the connected host will run model work
for a new server. The `includeContext` values `"thisServer"` and `"allServers"`
are also deprecated; omit the field or use `"none"` while maintaining this
surface.

### MCP Logging

Migration: log to stderr for stdio and use OpenTelemetry for structured remote
observability. Never write non-MCP data to stdout. In the deprecated modern
logging surface, log level is per request in
`_meta.io.modelcontextprotocol/logLevel`; `logging/setLevel` is gone and the
server must not emit `notifications/message` when the field is absent.

### DCR and HTTP+SSE

Dynamic Client Registration is deprecated in favor of Client ID Metadata
Documents, though tested host compatibility may require a fallback. HTTP+SSE is
deprecated in favor of Streamable HTTP.

### Removed core methods

Do not implement modern health or lifecycle logic with `ping`,
`logging/setLevel`, or `notifications/roots/list_changed`; they were removed
from the core revision. Use a transport/application health endpoint,
per-request `logLevel` only for deprecated MCP logging, and explicit
application configuration/invalidation instead.

## Capability checklist

| Feature                 | Declaration/opt-in                                                                      | Default fallback                                         |
| ----------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| tools/resources/prompts | Server advertises in `server/discover`                                                  | method unavailable                                       |
| list/resource changes   | Client opens `subscriptions/listen` with exact filter and server explicitly supports it | TTL refresh or explicit listen rejection                 |
| progress                | Client provides `progressToken`                                                         | complete silently                                        |
| elicitation             | Client declares form and/or URL per request                                             | return useful failure or require ordinary argument/setup |
| completion              | Server advertises completions                                                           | no suggestions                                           |
| extensions              | Both peers advertise the identifier                                                     | core behavior or documented error                        |
| roots/sampling/logging  | Deprecated capability plus MRTR/request opt-in as applicable                            | use migration path                                       |
