# Tool Design

Tool names, descriptions, schemas, annotations, and results are runtime
contracts visible to models and hosts. Weak contracts become bad calls,
unnecessary confirmations, and authorization mistakes.

## Contents

- Baseline, descriptions, and input schemas
- Required arguments, discovery, and tool count
- Results, moving files, content blocks, and errors
- Stateful workflows and `x-mcp-header`

## Baseline

- Give every tool a stable, short name. Prefer `snake_case` for consistency,
  though the protocol also permits ASCII letters, digits, `_`, `-`, and `.`.
- Keep names within 1–128 characters and unique within the server.
- Pick one convention for the whole server and hold it. A namespace prefix
  (`service.verb_noun` or `service_verb_noun`) helps a model tell your tools
  apart from the other servers loaded in the same session. Mixed conventions in
  one fleet are their own defect.
- Declare each tool name as an exported constant and reference the constant from
  registration, tests, and any contract file. A name typed twice drifts.
- Bound agent authority in the tool set itself. A capability you do not want an
  agent to have is one you do not register, not one you guard with a
  description.
- Add a human-readable `title` and precise `description`.
- Use the narrowest practical `inputSchema` and validate at the MCP seam.
- Split reads and mutations into separate tools.
- Mark read tools with `readOnlyHint: true`.
- Mark delete/overwrite/destructive tools with `destructiveHint: true`.
- On a mutating tool, set `idempotentHint: true` only when repeating the same
  call has no additional effect. It is not meaningful for a read-only tool.
- Set `openWorldHint` accurately when a tool reaches external systems.
- Enforce authorization and policy in the handler. Annotations are untrusted
  hints for client UX, never access control.

Modern Streamable HTTP has no response-stream replay. If a stream breaks after
a mutation commits, the client reissues the operation as a new request with a
new JSON-RPC ID. Use application idempotency keys, conditional writes,
operation handles, or a status/read-after-write tool when duplicate execution
would be harmful. The JSON-RPC ID is not an application idempotency key.

Servers should return `tools/list` in deterministic order. This improves client
caching and model prompt-cache reuse.

## Descriptions

Write a one-line manpage entry plus the details that disambiguate the tool.

Good:

```text
search_issues: Search issue titles and bodies by keyword. Returns at most
limit results in descending update order. Does not search comments or pull requests.
```

Weak:

```text
search_issues: Searches for issues.
```

When tools overlap, explain the decision:

```text
get_user:           Fetch a user by ID. If only an email is known, use find_user_by_email.
find_user_by_email: Resolve one email to a user ID. Returns a not-found tool error if absent.
```

Do not use descriptions to override host/system behavior or issue global model
instructions.

## Input schemas

MCP defaults schemas without `$schema` to JSON Schema 2020-12. The modern
protocol permits all 2020-12 keywords, but a schema should still be small and
cheap to validate.

Every constraint encoded in the schema eliminates a class of bad calls:

| Instead of                | Use                                                          |
| ------------------------- | ------------------------------------------------------------ |
| `z.string()` for an ID    | `z.string().regex(/^usr_[a-z0-9]{12}$/)`                     |
| `z.number()` for a limit  | `z.number().int().min(1).max(100).default(20)`               |
| `z.string()` for a choice | `z.enum(["open", "closed", "all"])`                          |
| optional with no hint     | `.optional().describe("Defaults to the caller's workspace")` |

```typescript
const inputSchema = z.object({
  query: z
    .string()
    .min(1)
    .describe("Keywords to search for; quoted phrases are supported"),
  status: z
    .enum(["open", "closed", "all"])
    .default("open")
    .describe("Issue status; defaults to open"),
  limit: z
    .number()
    .int()
    .min(1)
    .max(50)
    .default(10)
    .describe("Maximum results; hard cap is 50"),
});
```

For a tool with no parameters, advertise an object schema. The protocol
recommends `{ "type": "object", "additionalProperties": false }` for an
explicitly empty argument object; an SDK may derive this when its input schema
is omitted.

Do not automatically fetch network `$ref` targets. Reject unresolved external
references by default. Any opt-in resolver needs strict host allowlists, SSRF
defenses, timeouts, byte limits, recursion/depth limits, and composition-cost
bounds.

## Required arguments and discovery

Make required fields and defaults obvious in descriptions. Do not rely on a
schema keyword alone for a choice a user/model cannot infer.

Good:

```text
kind: Required output format. Use "markdown" unless the user explicitly asks for HTML.
```

For agentic workflows, provide search/list tools so users do not have to know
opaque IDs. A useful server rarely exposes only
`read_document(workspace_id, document_id)`; it also offers a way to discover the
workspace and document IDs.

## Tool count

| Count | Guidance                                                      |
| ----- | ------------------------------------------------------------- |
| 1–15  | One tool per action is usually clearest.                      |
| 15–30 | Still workable; audit near-duplicates and naming.             |
| 30+   | Consider discovery + execution, with common actions promoted. |

This is model-context and host-UX guidance, not a protocol limit.

For a generic execute tool, return the selected operation's description,
parameter schema, and safety classification during discovery. Validate against
that exact schema at execution. A single generic tool annotation cannot safely
represent a catalog containing both reads and destructive writes, so keep
dangerous actions dedicated or require explicit confirmation/policy based on
the selected operation.

## Results

Make output easy to parse and useful for the next call:

- return `structuredContent` for machine-readable data
- define `outputSchema` when the shape is stable
- include a text fallback for hosts that do not consume structured output
- include IDs, versions, cursors, and optimistic-concurrency values needed next
- include counts and truncation/pagination notes
- return a short, specific confirmation after a mutation
- bound result size and link to a resource for large optional data

In `2026-07-28`, `structuredContent` may be any JSON value, not only an object.
When `outputSchema` exists, the server must return matching structured content
and the client should validate it.

```typescript
server.registerTool(
  "get_weather",
  {
    title: "Get weather",
    description: "Get current weather for one city.",
    inputSchema: z.object({ city: z.string().describe("City name") }),
    outputSchema: z.object({
      temperature_c: z.number(),
      conditions: z.string(),
    }),
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      openWorldHint: true,
    },
  },
  async ({ city }) => {
    const output = await fetchWeather(city);
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output,
    };
  },
);
```

The modern wire result also carries `resultType: "complete"`. Let the SDK
serving entry add wire bookkeeping rather than returning it from application
handlers unless that SDK explicitly requires it.

Do not return raw HTML unless the tool is intentionally an HTML fetcher,
megabytes of unfiltered API output, a bare `"ok"` without the created ID, or
credentials/internal stack traces.

## Moving files and large content

Do not carry file bytes through tool arguments. A model generating arguments
cannot reliably reproduce large content and cannot read binary content at all.
One team watched an agent truncate a 114 KB text file twice through an inline
`content` parameter, well under the declared size cap, and only caught it
because the store was content-addressed and the returned hash did not match the
local file's. Inline content is fine for small text and is a dead end for
anything else.

As of `2026-07-28` the protocol has no mechanism for this. `resources/read` is a
client asking the server for a resource the server holds, not a server asking the
client for a file: `resources` is a server capability, and the client capability
set has no resource service. Roots carry a URI and a name with no read method
and are informational, not access control. A File Uploads Working Group is
chartered to define declarative file inputs so hosts can present a native picker
and substitute the bytes, anchored on a SEP that is still an open proposal.
Design a seam for it; do not implement the draft.

Until then, the portable pattern is **MCP as the control plane, bytes out of
band**:

1. A `begin_upload` tool authenticates through MCP, authorizes the destination,
   and returns a short-lived one-time upload URL, the exact required headers, an
   expiry, and the declared size and hash it will check.
2. The agent's shell performs the transfer. The model carries the path, URL,
   headers, size, and hash, and never the file body. Every mainstream host has a
   shell.
3. A `commit_upload` tool re-checks authorization, then verifies the staged
   object's actual size and hash against what was declared, and publishes only on
   an exact match.

Rules that make it safe:

- Bind the upload capability narrowly: one principal, one destination, one
  method, one object, one declared size and hash, one use, short expiry.
- Re-validate every limit at commit, not only at issue time. If an outer layer
  advertises a looser cap than the storage layer enforces, staging success can
  otherwise bypass the real guardrail.
- Verify by hash end to end. That is what turns silent corruption into a failed
  commit.
- Accept the residual risk knowingly: the URL is visible to the model, so its
  scope and lifetime carry the security burden.
- Integration-test per host. Source capability is not permission; network policy,
  sandboxing, and approval settings can still block the transfer.

A local stdio companion is a fallback, not the answer: it adds install, upgrade,
and path-security surface, and MCP does not require a host to pass a remote
bearer token to a stdio child, so the credential handoff is host-specific.
Chunked inline upload keeps the model in the byte path and still cannot handle
binary; do not build it.

## Content blocks

Tool results may include:

| Type                | Use                                              |
| ------------------- | ------------------------------------------------ |
| `text`              | Default text or serialized JSON fallback         |
| `image`             | Image bytes with MIME type                       |
| `audio`             | Audio bytes with MIME type                       |
| `resource_link`     | URI for large or optional follow-up context      |
| embedded `resource` | Small resource content always needed immediately |

All content types may carry resource-style annotations such as audience,
priority, and last-modified time. Treat those as hints.

## Errors

Use a tool execution error for failures the model can understand and recover
from, including domain validation, upstream/API failures, range errors, or an
expired application handle:

```typescript
if (!item) {
  return {
    isError: true,
    content: [
      {
        type: "text",
        text: `Item ${id} was not found. Use search_items to find valid IDs.`,
      },
    ],
  };
}
```

Use JSON-RPC protocol errors for malformed MCP requests, unknown tools, invalid
protocol structure, or server defects. Authentication failures remain HTTP
401/403 before MCP dispatch.

Never turn an expected tool failure into a crashed transport or an HTML 500.

## Stateful workflows

Modern MCP has no protocol session. For carts, browser contexts, transactions,
or other state spanning calls:

1. return an opaque handle from a creation tool
2. accept it as an explicit argument on follow-up tools
3. authorize the caller against the handle on every use
4. document its lifetime and cleanup behavior
5. return a recoverable tool error for an unknown or expired handle

For an unauthenticated server, the handle itself is a bearer capability: use
high entropy, short lifetimes, least privilege, and revocation/cleanup.

MRTR `requestState` is separate: it correlates retries of the same logical
request and must be protected as described in `elicitation.md`.

For confirmation flows, request confirmation before the irreversible side
effect. If work must happen before an MRTR round, record a durable operation
phase and make every retry resumable; a new JSON-RPC ID does not imply a new
business operation.

## `x-mcp-header`

On modern Streamable HTTP, a tool schema may mark a primitive property with
`x-mcp-header` so conforming clients mirror the argument into
`Mcp-Param-{Name}`. Use this only when an intermediary genuinely needs a value
for routing or policy.

- only string, boolean, or safe-range integer properties
- values must be statically reachable through object `properties`
- header names must be valid and case-insensitively unique
- never annotate secrets, tokens, payment data, or PII
- let the SDK handle Base64 sentinel encoding and header/body validation

Clients using HTTP must exclude an invalidly annotated tool. Servers must reject
missing/mismatched mirrored values with HTTP 400 and `HeaderMismatch`
(`-32020`).
