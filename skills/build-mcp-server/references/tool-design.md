# Tool Design — Writing Tools Hosts Use Correctly

Tool schemas and descriptions are part of the MCP contract. They are visible to
models and hosts, so vague descriptions or loose schemas become runtime bugs.

## Baseline rules

- Give every tool a stable, short, snake_case name.
- Add a human-readable `title`.
- Add a precise `description`.
- Use the narrowest practical `inputSchema`.
- Reject invalid input at the MCP seam.
- Split read and write operations into separate tools.
- Mark read tools with `readOnlyHint: true`.
- Mark destructive or overwrite/delete tools with `destructiveHint: true`.
- Mark retry-safe write tools with `idempotentHint: true` only when that is
  actually true.
- Do not use tool descriptions to override system instructions or tell the model
  how to behave globally.

Annotations are hints for clients and user interfaces, not access control. Server
authorization must still enforce the real policy.

---

## Descriptions

Write descriptions like one-line manpage entries plus disambiguating details.

Good:

```text
search_issues — Search issues by keyword across title and body. Returns up to
limit results ranked by recency. Does not search comments or pull requests.
```

Bad:

```text
search_issues — Searches for issues.
```

When two tools are similar, each description should say when to use the other:

```text
get_user           — Fetch a user by ID. If you only have an email, use find_user_by_email.
find_user_by_email — Look up a user by email address. Returns a not-found tool error if absent.
```

---

## Parameter schemas

Every constraint in the schema is one fewer bad call the handler has to recover
from.

| Instead of | Use |
|---|---|
| `z.string()` for an ID | `z.string().regex(/^usr_[a-z0-9]{12}$/)` |
| `z.number()` for a limit | `z.number().int().min(1).max(100).default(20)` |
| `z.string()` for a choice | `z.enum(["open", "closed", "all"])` |
| optional with no hint | `.optional().describe("Defaults to the caller's workspace")` |

Describe parameters. The description text is visible in the schema the host sees.

```typescript
{
  query: z.string().min(1).describe("Keywords to search for. Supports quoted phrases."),
  status: z.enum(["open", "closed", "all"]).default("open")
    .describe("Filter by issue status. Use all to include closed issues."),
  limit: z.number().int().min(1).max(50).default(10)
    .describe("Maximum results. Hard cap is 50."),
}
```

---

## Return shapes

Make outputs parseable and useful for follow-up calls.

Do:

- return JSON for structured data
- return short confirmations for mutations
- include IDs needed for follow-up calls
- include counts and pagination/truncation notes
- return `structuredContent` when the SDK/version supports it
- include a text fallback for hosts that do not read structured output

Do not:

- return raw HTML unless the tool is explicitly an HTML fetcher
- return megabytes of unfiltered API output
- return `"ok"` after creating a record without the created ID
- leak credentials, tokens, private URLs, or internal stack traces

---

## How many tools?

| Tool count | Guidance |
|---|---|
| 1-15 | One tool per action. Sweet spot. |
| 15-30 | Still workable. Audit near-duplicates. |
| 30+ | Prefer search + execute. Optionally promote the top 3-5 actions. |

This is not a protocol limit. It is model-context and host-UX economics: every
tool schema costs attention and tokens.

---

## Tool errors

Return expected failures as MCP tool results rather than crashing the transport.

```typescript
if (!item) {
  return {
    isError: true,
    content: [{
      type: "text",
      text: `Item ${id} was not found. Use search_items to find valid IDs.`,
    }],
  };
}
```

Use transport/protocol errors for malformed MCP requests, auth failures, or
server defects. Use tool errors for domain failures the model can recover from.

---

## Tool annotations

Tool annotations are host/UI hints.

| Annotation | Meaning |
|---|---|
| `readOnlyHint: true` | Tool does not modify its environment |
| `destructiveHint: true` | Tool may delete, overwrite, or otherwise destroy data |
| `idempotentHint: true` | Repeating same args has no additional effect |
| `openWorldHint: true` | Tool talks to external systems outside the server |

Example:

```typescript
server.registerTool("delete_file", {
  title: "Delete file",
  description: "Delete one file by path. This permanently removes the file.",
  inputSchema: { path: z.string().describe("Path inside the approved root") },
  annotations: {
    readOnlyHint: false,
    destructiveHint: true,
    idempotentHint: false,
  },
}, handler);
```

Mark every read-only tool explicitly with `readOnlyHint: true`.

---

## Structured output

Plain JSON in `content[].text` works broadly, but modern MCP supports typed
output with `outputSchema` and `structuredContent`.

```typescript
server.registerTool("get_weather", {
  title: "Get weather",
  description: "Get current weather for one city.",
  inputSchema: { city: z.string().describe("City name") },
  outputSchema: { temp: z.number(), conditions: z.string() },
  annotations: { readOnlyHint: true, destructiveHint: false },
}, async ({ city }) => {
  const data = await fetchWeather(city);
  return {
    content: [{ type: "text", text: JSON.stringify(data, null, 2) }],
    structuredContent: data,
  };
});
```

Always include a text fallback unless the target host is known to consume
structured output.

---

## Content types beyond text

Tools can return more than text:

| Type | Use for |
|---|---|
| `text` | Default text/JSON response |
| `image` | Screenshots, charts, diagrams |
| `audio` | Audio output |
| `resource_link` | Pointer to a resource the host may fetch later |
| embedded `resource` | Small resource content that is always needed |

Use `resource_link` for large payloads or optional follow-up context. Embed small
content only when it is needed immediately.
