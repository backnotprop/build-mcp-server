---
name: build-mcp-server
description: Design, scaffold, audit, and upgrade Model Context Protocol servers. Use when the user asks to build or migrate an MCP server, adopt MCP 2026-07-28, add dual-era support, create an MCP integration, expose an API or data source through MCP tools/resources/prompts, add OAuth or API-key auth to an MCP endpoint, run an authorization server for it, support browser-hosted MCP clients, or choose an MCP transport/auth/deployment shape.
---

# Build an MCP Server

Help design and build MCP servers. Keep the work focused on protocol primitives,
transport, authorization, validation, tests, and deployment.

Choose the operating mode before acting:

| Mode                 | Default behavior                                                                                                           |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Build or change      | Design, implement, and verify the requested server changes.                                                                |
| Audit or review      | Inspect and report with evidence. Do not edit files or mutate external systems unless the user explicitly asks.            |
| Migration or upgrade | Audit the current implementation first. Implement the migration only when the request includes authorization to change it. |

The first decision is the protocol era. The released `2026-07-28` protocol is
stateless and materially different from revisions through `2025-11-25`. Do not
mix their lifecycle or transport patterns. Read `references/protocol-eras.md`
before choosing an SDK or scaffold.

For an existing server, read `references/migrate-2026-07-28.md` and inventory
its lifecycle, transport, state, server-to-client requests, caching,
authorization, SDK entry points, and real client versions before editing it.

This skill is TypeScript-first: its ready-to-copy scaffold targets the official
TypeScript SDK v2. For Python or another SDK, verify every required modern wire
behavior in the selected release instead of transliterating TypeScript APIs.

Your first job is discovery, not code. MCP servers stay small when protocol
version, transport, authorization, and primitive shape are chosen explicitly.

Two standing rules for this work:

- **Assume your training data is stale on MCP.** The specification moves fast
  and revisions change wire behavior, not just wording. Verify every protocol
  claim against the versioned specification page for the revision you target and
  its changelog, and every SDK claim against the installed version.
- **MCP is a thin adapter over a good API, not the design center.** Adapt to the
  service layer that already owns validation, authorization, persistence, and
  events. Never duplicate route business logic inside a tool handler.

---

## Phase 1: Interrogate the use case

Answer these questions before scaffolding. If the request already answers them,
state the inferred choices and proceed.

### 1. What does the server expose?

| It exposes...                                                | Likely direction                                               |
| ------------------------------------------------------------ | -------------------------------------------------------------- |
| A cloud API, SaaS app, database, or service                  | Remote Streamable HTTP                                         |
| Local files, a local process, localhost service, or hardware | Local stdio                                                    |
| Desktop app with an existing in-process service or local API | Embedded loopback HTTP, optionally with a stdio shim           |
| Pure computation with no user-local state                    | Remote Streamable HTTP by default                              |
| A private internal service                                   | Remote Streamable HTTP or local stdio, based on network access |

For an existing application, identify the in-process service or local API that
already owns validation, authorization, persistence, and events. MCP handlers
should adapt to that layer rather than duplicate its business logic.

### 2. Who connects, and which protocol era do they support?

- One developer or a local automation script: local stdio is acceptable.
- A team, organization, or external users: remote Streamable HTTP.
- A self-hosted customer deployment: remote Streamable HTTP inside their
  deployment boundary.
- Name the target hosts and versions, such as Codex, Claude Code, Cursor,
  ChatGPT, a browser app, or a custom client.
- For a new server, target the modern `2026-07-28` era. Add 2025-era support
  only when a required host needs it, and test each era independently.
- Treat product-level rollout announcements as leads, not compatibility proof.
  Record the exact web, desktop, CLI, connector, or embedded host version tested.
- For remote OAuth servers, read
  `references/target-client-compatibility.md`.

### 3. What primitives does it need?

- Tools: model-invoked actions, searches, parameterized reads, and mutations.
- Resources: application-controlled, read-only context identified by URI.
- Prompts: user-invoked message templates or workflows.
- Elicitation: additional user input needed while resolving `tools/call`,
  `resources/read`, or `prompts/get`. In the modern era this uses a multi
  round-trip `input_required` result, not a direct server-to-client request.

Most servers start with tools. Use resources when context is useful independent
of a single tool call. Use prompts only for a genuinely reusable user workflow.
When selecting resources, templates, or prompts, read
`references/resources-and-prompts.md` now. When selecting subscriptions or an
extension, read `references/server-capabilities.md` now.

### 4. How many actions are there?

- Under roughly 15 actions: one tool per action.
- 15–30 actions: still workable, but audit near-duplicates.
- Dozens to hundreds of actions: consider a discovery-plus-execution pattern.

This is product guidance, not a protocol limit.

### 5. Does a request need more user input?

- Ordinary required input: put it in the tool/resource/prompt arguments.
- Simple non-sensitive input needed mid-operation: use form elicitation through
  the modern MRTR flow, with capability checks and a fallback.
- Secrets, API keys, passwords, payment credentials, or third-party OAuth: use
  URL-mode elicitation or a separate trusted setup flow. Never collect them in
  form mode. See `references/elicitation.md`.

### 6. What authorization is required?

Separate two authorization planes:

- MCP client -> MCP server: who may call the MCP endpoint.
- MCP server -> upstream service: which credential the server uses for the API,
  database, or service it wraps.

Do not collapse them into one vague "API key". See `references/auth.md`.

Then answer a second question the spec leaves open: **who runs the authorization
server?** Delegate to an external identity provider, act as your own, or accept
API keys and skip OAuth. Delegation is not automatically cheapest; provider
consoles need configuration nobody can do in code. If the answer is "we do", or
if browser-hosted clients must work, read `references/authorization-server.md`.

### 7. Does state span requests?

Modern MCP has no protocol session. If application state spans calls, mint an
opaque handle and pass it as an ordinary result/argument. Authorize the handle
on every use and give it a documented lifetime. For an MRTR retry, use an
integrity-protected `requestState` bound to the caller, operation, and expiry.

---

## Phase 2: Recommend a server shape

Recommend one primary path and name any compatibility path separately.

### Remote Streamable HTTP

Default for cloud APIs, team servers, self-hosted deployments, and anything
reachable over the network.

For `2026-07-28`:

- the MCP endpoint accepts POST; GET and DELETE are not part of the modern
  transport
- every request is self-contained and carries protocol version and client
  capabilities in `_meta`
- there is no `initialize` handshake, `Mcp-Session-Id`, session affinity, or SSE
  replay
- a response can be JSON or a request-scoped SSE stream
- long-lived change events use `subscriptions/listen`

Two different things are called "SSE" and the answer differs:

- **HTTP+SSE, the 2024-11-05 transport** with its separate SSE and POST
  endpoints, is deprecated and is in the deprecated-features registry. Do not
  adopt it. Migrate existing deployments to Streamable HTTP.
- **SSE as a response content type inside Streamable HTTP** is current. A server
  may answer any request POST with either `application/json` or a
  request-scoped `text/event-stream`, and a client must support both. A
  JSON-only server is valid; a client that only accepts JSON is not.

Do not hardcode a protocol version in product code. Let the SDK negotiate, and
keep version strings in tests and smoke scripts where a change is visible. A
version you observe a client negotiating is that client's choice, not your
server's maximum; read the maximum from `server/discover`.

Use `references/remote-http-scaffold.md` for the modern TypeScript scaffold.
Use `references/deploy-cloudflare-workers.md` only for a Cloudflare Workers
target.

### Local stdio

Use when the server must access user-local files, processes, hardware, or
desktop state.

Keep local stdio simple:

- write only MCP JSON-RPC messages to stdout; log to stderr
- validate and confine filesystem paths
- avoid broad shell execution
- avoid plaintext secrets on disk
- treat the process as a multiplexed transport, not a conversation or session
- document exactly what local access the server needs

### Embedded desktop listener

Use a loopback Streamable HTTP listener inside the desktop process when the app
already has an in-process service layer and HTTP is a better host boundary than
a child process. Keep one MCP registry/factory, adapt it to that existing
service, and add a stdio shim only for hosts that require stdio. Treat loopback
as a security boundary: bind locally, validate Host and Origin, authenticate the
host, and own listener startup/shutdown with the desktop lifecycle. See
`references/embedded-desktop.md`.

### Legacy compatibility

Revisions through `2025-11-25` use `initialize` and older Streamable HTTP
semantics. If a target host requires that era, use an SDK-supported dual-era
entry point or an explicitly separate legacy route. Never bolt legacy GET,
DELETE, session IDs, or direct server-to-client requests onto the modern path.

The official TypeScript v2 `createMcpHandler` and `serveStdio` entries support
both eras. For a new modern-only server, explicitly set `legacy: "reject"`;
enable the SDK's compatibility path only when a required client needs it.
Existing sessionful legacy HTTP deployments still need a deliberately isolated
legacy handler rather than session state inside the modern per-request factory.

---

## Phase 3: Pick a tool design pattern

Tool schemas and descriptions are runtime contracts visible to models and hosts.
Keep them precise, small, and stable.

### Pattern A: one tool per action

Use for small surfaces.

```text
create_issue:    Create a new issue. Params: title, body, labels[]
update_issue:    Update an issue. Params: id, title?, body?, state?
search_issues:   Search issues. Params: query, limit?
add_comment:     Add a comment. Params: issue_id, body
```

### Pattern B: discover + execute

Use selectively for very large API surfaces.

```text
search_actions:  Return matching actions with IDs, descriptions, safety, and schemas.
execute_action:  Execute one action by ID with validated parameters.
```

The server owns the full catalog. Never hide whether the selected action is a
read, write, or destructive operation behind a generic annotation. Consider
promoting the most common actions to dedicated tools.

See `references/tool-design.md`.

---

## Phase 4: Pick an implementation stack

Prefer the user's existing stack only if it supports the target protocol era.
Framework familiarity does not compensate for an incompatible wire protocol.

| Stack                                                  | Use when                                                                                                                                                                                     |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Official TypeScript SDK v2 split packages              | Default for a new `2026-07-28` TypeScript/JavaScript server. Use the modern `createMcpHandler` or `serveStdio` entry points.                                                                 |
| Official TypeScript SDK v1 `@modelcontextprotocol/sdk` | Maintaining a 2025-era server. Do not describe it as a current-protocol scaffold.                                                                                                            |
| Python/FastMCP or another SDK                          | The user prefers that ecosystem and its installed version is verified to support every required `2026-07-28` behavior. Otherwise target the older era explicitly or choose a supporting SDK. |

The official TypeScript v2 split packages are stable, but still confirm package
tags, imports, and supported protocol versions before implementation. Do not
mix v1 and v2 imports. See `references/versions.md`.

---

## Phase 5: Scaffold and verify

Once protocol era, deployment, primitives, framework, and authorization are
chosen:

1. Scaffold a minimal server with one read-only tool.
2. Add authorization before private data or mutations.
3. Add resources/prompts only when they solve a concrete context problem.
4. Add modern-protocol checks for:
   - `server/discover`
   - required per-request `_meta`
   - `tools/list` and `tools/call`
   - required wire `resultType`
   - required `ttlMs` and `cacheScope` on cacheable results
   - `resources/list`, `resources/read`, `resources/templates/list`, or
     `prompts/list`/`prompts/get` when exposed
   - malformed request and unsupported protocol version
   - a domain/input failure returned as a tool result, not a crashed transport
5. For Streamable HTTP, test POST request headers (`MCP-Protocol-Version`,
   `Mcp-Method`, and `Mcp-Name` where required) on JSON-RPC requests,
   header/body mismatch, Origin, unauthorized access, and every enabled server
   response mode. A JSON-only server is valid; clients must support JSON and
   request-scoped SSE. The current revision does not define notification-POST
   header requirements.
6. Run the current conformance suite and MCP Inspector version that support the
   target era.
7. Test with each actual target host.
8. If MCP OAuth is selected, verify login, scope step-up, reconnect/refresh
   after token expiry, revoke/re-auth, and registration behavior with each
   target host. Prove the auth surface with exact assertions, not a successful
   login: the full `WWW-Authenticate` value, whole metadata documents, each CORS
   header individually including the absence of allow-credentials, fixed error
   strings, code burn on a wrong verifier, and revocation actually stopping
   service. Red-proof those tests against the pre-change source. See
   `references/authorization-server.md`.
9. If supporting both eras, run the same functional tests against modern and
   legacy connections and assert that their transport behaviors stay separate.
10. For mutating tools, simulate a response stream failing after the operation
    commits. Verify that a retry with a new JSON-RPC ID cannot silently duplicate
    harmful work.
11. Make an explicit `subscriptions/listen` decision: either reject it before
    opening SSE and test that behavior, or provide a real event bus, fan-out,
    capacity, cancellation, and reconnect tests. Capability advertising alone
    does not disable the SDK listen router.

---

## Server primitives reference

| Primitive     | Controller                            | Use when                                                           |
| ------------- | ------------------------------------- | ------------------------------------------------------------------ |
| Tools         | Model through the host                | Actions, searches, parameterized reads, writes                     |
| Resources     | Host application                      | Browsable/read-only URI-addressed context                          |
| Prompts       | User                                  | Reusable workflows or message templates                            |
| Elicitation   | Server during a supported request     | Additional non-secret form input or a trusted URL handoff via MRTR |
| Subscriptions | Client                                | Opt in to list/resource change notifications                       |
| Extensions    | Negotiated client/server capabilities | Optional features such as MCP Apps or Tasks                        |

Roots, Sampling, Logging, and Dynamic Client Registration were deprecated in
`2026-07-28`. HTTP+SSE has been deprecated since `2025-03-26` and is now carried
in the same registry. All remain available only for compatibility during their
deprecation windows; new servers should use their documented migration paths.
Client ID Metadata Documents are DCR's migration path. See
`references/server-capabilities.md`.

---

## Deployment checklist

Before calling a modern server ready:

- [ ] `server/discover` advertises supported versions, capabilities, identity,
      instructions when useful, and cache hints.
- [ ] Every request is processed independently; no caller identity, capability,
      conversation, or application state is inferred from a connection.
- [ ] Streamable HTTP exposes POST at the MCP endpoint; GET/DELETE return 405
      unless they belong to a deliberately separate legacy route.
- [ ] Streamable HTTP validates `Origin` when present and local servers bind to
      localhost or validate Host.
- [ ] Required JSON-RPC request POST headers are present, safely decoded, and
      match the request body; unsupported versions return the specified error.
- [ ] Every enabled server response mode is tested. A JSON-only server is
      allowed; clients are verified to accept both JSON and request-scoped SSE.
- [ ] `subscriptions/listen` is either explicitly rejected without opening SSE,
      or backed by a tested event bus with fan-out, capacity, cancellation, and
      reconnect behavior.
- [ ] There are no protocol sessions, resumability, `Last-Event-ID`, or
      server-initiated JSON-RPC requests on the modern path.
- [ ] Auth precedes private data and mutations; MCP auth is separate from
      upstream-service auth.
- [ ] Multiple credential shapes are discriminated by prefix with no
      fallthrough; a claimed-but-invalid bearer is 401, never anonymous.
- [ ] If MCP OAuth is selected, Protected Resource Metadata,
      authorization-server discovery, `resource` audience binding, PKCE,
      issuer validation, and client registration behavior are tested.
- [ ] If the server runs its own authorization server: CORS is on the metadata
      documents, registration, and token, and on nothing else; no endpoint sends
      `Access-Control-Allow-Credentials`; consent is shown per client and
      remembered; revocation stops service and clears warm authorization codes.
- [ ] The credential lane the deployment serves is set in production, not only
      in staging, and the setup guide the server publishes matches it.
- [ ] No protocol version is hardcoded in product code.
- [ ] Tool/resource/prompt schemas reject invalid input and external `$ref`
      fetching is disabled by default.
- [ ] Tools include useful names, titles, descriptions, schemas, and accurate
      annotations; annotations are never treated as authorization.
- [ ] Tool lists are deterministic and cacheable results use correct `ttlMs`
      and `cacheScope` values.
- [ ] Expected domain/input failures return useful tool results; malformed MCP
      requests return protocol errors; HTTP failures never become HTML pages.
- [ ] Stateful workflows use explicit, authorized handles. MRTR state is
      integrity-protected and replay-bounded.
- [ ] Secrets never appear in source, results, resources, prompts, URLs, logs,
      traces, exceptions, or test snapshots.
- [ ] Conformance, Inspector, and real-host checks pass for every supported era.

---

## Reference routing

Use `references/protocol-eras.md` as the canonical wire-invariants reference;
context-specific references should point back to it rather than inventing a
second protocol model.

| Situation                                      | Read                                        |
| ---------------------------------------------- | ------------------------------------------- |
| Any era or wire-protocol decision              | `references/protocol-eras.md`               |
| Audit, migration, or dual-era rollout          | `references/migrate-2026-07-28.md`          |
| TypeScript remote HTTP implementation          | `references/remote-http-scaffold.md`        |
| Pinned runnable TypeScript HTTP example        | `assets/typescript-http/`                   |
| Embedded Electron or other desktop application | `references/embedded-desktop.md`            |
| Cloudflare Workers deployment                  | `references/deploy-cloudflare-workers.md`   |
| Tool names, schemas, annotations, or state     | `references/tool-design.md`                 |
| MCP OAuth or upstream authorization            | `references/auth.md`                        |
| Running the authorization server yourself      | `references/authorization-server.md`        |
| Browser-hosted MCP clients, OAuth CORS         | `references/authorization-server.md`        |
| Resources, URI templates, or prompts           | `references/resources-and-prompts.md`       |
| Form/URL elicitation or MRTR state             | `references/elicitation.md`                 |
| Discovery, caching, subscriptions, extensions  | `references/server-capabilities.md`         |
| Real-host negotiation, OAuth, or tool UX       | `references/target-client-compatibility.md` |
| Package/API/version-sensitive claim            | `references/versions.md`                    |
