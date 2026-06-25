---
name: build-mcp-server
description: Design and scaffold Model Context Protocol servers. Use when the user asks to build an MCP server, create an MCP integration, expose an API or data source through MCP tools/resources/prompts, or choose an MCP transport/auth/deployment shape.
metadata:
  version: "0.2.0"
---

# Build an MCP Server

You are helping design and build an MCP server. Keep the work focused on the
server: protocol primitives, transport, auth, validation, tests, and deployment.

Your first job is discovery, not code. MCP servers are small when the transport,
auth, and tool/resource shape are chosen correctly; they become painful when
those choices are guessed.

Before scaffolding, answer the Phase 1 questions. If the user's request already
answers them, state the inferred choices and proceed.

---

## Phase 1 — Interrogate the use case

Ask the questions in one concise batch. Adapt wording to what the user already
said.

### 1. What does the server expose?

| It exposes... | Likely direction |
|---|---|
| A cloud API, SaaS app, database, or service | Remote Streamable HTTP |
| Local files, a local process, localhost service, or desktop state | Local stdio |
| Pure computation with no user-local state | Remote Streamable HTTP by default |
| A private internal service for one team | Remote Streamable HTTP or local stdio, based on network access |

### 2. Who connects to it?

- One developer or a local automation script: local stdio is acceptable.
- A team, organization, or external users: remote Streamable HTTP.
- A self-hosted customer deployment: remote Streamable HTTP inside their
  deployment boundary.

### 3. What primitives does it need?

- Tools: model-invoked actions. Use for searches, reads with model-chosen
  parameters, writes, mutations, and calls into APIs.
- Resources: read-only context the host can browse or attach, such as docs,
  files, records, logs, schemas, or versioned content.
- Prompts: reusable user-invoked workflows.

Most servers start with tools. Use resources when the server has browsable
context that is valuable independent of a single tool call.

### 4. How many actions are there?

- Under roughly 15 actions: one tool per action.
- 15-30 actions: still workable, but audit near-duplicates.
- Dozens to hundreds of actions: use a search + execute pattern.

### 5. Does a tool need mid-call user input?

- Simple non-sensitive input: use elicitation, with capability checks and a
  fallback.
- Sensitive input such as API keys, passwords, access tokens, or payment
  credentials: do not use form elicitation. Use OAuth or URL-mode elicitation
  with identity binding. See `references/elicitation.md`.

### 6. What auth is required?

Separate two auth planes:

- MCP client -> MCP server: who is allowed to call `/mcp`.
- MCP server -> upstream service: what credentials the MCP server uses when it
  calls the API/database/service it wraps.

Do not collapse those into one vague "API key" answer. See `references/auth.md`.

---

## Phase 2 — Recommend a server shape

Recommend one path.

### Remote Streamable HTTP server

Default for cloud APIs, team servers, self-host deployments, and anything meant
to be reachable over the network.

Why it wins:

- one deployed URL serves all clients
- ordinary HTTP auth works
- OAuth and browser redirects are possible
- upgrades happen server-side
- works with the protocol's remote transport

Use `references/remote-http-scaffold.md` for a portable TypeScript or Python
scaffold. Use `references/deploy-cloudflare-workers.md` only when the user
explicitly wants a Cloudflare Workers deploy target.

### Local stdio server

Use when the server must access user-local files, user-local processes, local
hardware, or local desktop state.

Keep local stdio simple:

- validate and confine filesystem paths
- avoid broad shell execution
- avoid plaintext secrets on disk
- document exactly what local access the server needs

This skill can design the server and the stdio command, but it does not cover
runtime packaging.

---

## Phase 3 — Pick a tool design pattern

Tool schemas and descriptions are part of the runtime contract. They are visible
to models and hosts, so they should be precise, small, and stable.

### Pattern A: One tool per action

Use for small surfaces.

```text
create_issue    — Create a new issue. Params: title, body, labels[]
update_issue    — Update an existing issue. Params: id, title?, body?, state?
search_issues   — Search issues by query string. Params: query, limit?
add_comment     — Add a comment to an issue. Params: issue_id, body
```

Why it works: each tool has a tight name, description, input schema, output
shape, and safety annotation.

### Pattern B: Search + execute

Use for large API surfaces where listing every operation would flood the model
context.

```text
search_actions  — Given a natural-language intent, return matching actions
                  with IDs, descriptions, and parameter schemas.
execute_action  — Run an action by ID with a params object.
```

The server owns the full action catalog internally. The model searches, selects,
then executes. Optionally promote the 3-5 most common actions to dedicated
tools.

See `references/tool-design.md` for schema and return-shape guidance.

---

## Phase 4 — Pick an implementation stack

Prefer the user's existing stack. If there is no preference:

| Stack | Use when |
|---|---|
| Official TypeScript SDK | Default for TypeScript/JavaScript servers and Hono/Express/Workers environments. |
| FastMCP 3.x for Python | User prefers Python or is wrapping Python libraries. |

Version rule for TypeScript:

- Stable production scaffolds should use the current stable SDK line unless the
  target repo has already chosen the SDK v2 split packages.
- The SDK v2 `main` branch uses split packages such as
  `@modelcontextprotocol/server`, `@modelcontextprotocol/node`,
  `@modelcontextprotocol/express`, and `@modelcontextprotocol/hono`.
- Before implementation, confirm the package version and import paths. Do not
  mix v1 and v2 examples in one codebase.

See `references/versions.md`.

---

## Phase 5 — Scaffold and verify

Once deployment model, primitive shape, framework, and auth are chosen:

1. Scaffold a minimal server with one read-only tool.
2. Add auth before adding private or mutating operations.
3. Add resources/prompts only if they solve a concrete context problem.
4. Add tests or smoke checks for:
   - initialize
   - tools/list
   - tools/call
   - resources/list or resources/read, if resources exist
   - unauthorized request
   - invalid input
   - a tool error returned as an MCP tool result, not a crashed transport
5. Run MCP Inspector against the local server.
6. Test with the actual target MCP host.

---

## Server primitives reference

| Primitive | Who triggers it | Use when |
|---|---|---|
| Tools | Model through the host | Actions, searches, reads with parameters, writes |
| Resources | Host application | Browsable/read-only context |
| Prompts | User | Reusable workflows or message templates |
| Elicitation | Server mid-request | Simple user input or URL handoff, with client capability support |
| Sampling | Server mid-request | Server asks the host model to generate/classify/summarize |

Read:

- `references/tool-design.md`
- `references/resources-and-prompts.md`
- `references/elicitation.md`
- `references/server-capabilities.md`

---

## Deployment checklist

Before calling a server ready:

- [ ] `/mcp` uses the correct transport for the target: Streamable HTTP or stdio.
- [ ] Streamable HTTP endpoint handles the methods required by the chosen mode
      (`POST`, and `GET`/`DELETE` where stateful sessions or resumability are in
      play).
- [ ] `Origin` is validated for HTTP requests when present.
- [ ] Local HTTP servers bind to localhost or validate Host headers.
- [ ] Unsupported `MCP-Protocol-Version` values are rejected by the SDK,
      middleware, or explicit route code.
- [ ] Auth is implemented before private data or mutations are exposed.
- [ ] MCP client auth is separate from upstream-service auth.
- [ ] Tool schemas reject invalid input at the MCP seam.
- [ ] Mutating tools are separated from read tools.
- [ ] Tools include useful `title`, `description`, `inputSchema`, and safety
      annotations.
- [ ] Errors return MCP results/errors, not HTML 500s.
- [ ] Secrets come from config/env/secret stores, never hardcoded.
- [ ] Inspector and target-host smoke checks pass.

---

## Reference files

- `references/remote-http-scaffold.md` — portable Streamable HTTP scaffold
- `references/deploy-cloudflare-workers.md` — Cloudflare Workers server target
- `references/tool-design.md` — tool naming, schemas, annotations, returns
- `references/auth.md` — MCP server auth and upstream auth
- `references/resources-and-prompts.md` — read-only context and prompt templates
- `references/elicitation.md` — mid-call user input and URL handoff
- `references/server-capabilities.md` — instructions, sampling, roots, logging,
  progress, cancellation
- `references/versions.md` — version-sensitive claims ledger
