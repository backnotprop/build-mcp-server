# Server Capabilities

MCP has features beyond tools/resources/prompts. Add them only when the server
needs them.

SDK APIs for these features are version-sensitive. Confirm the installed SDK
before copying method names.

---

## Instructions

Server instructions describe cross-tool relationships, workflow patterns, or
constraints. Hosts may include them in the model context.

Use instructions for stable server-level guidance that does not belong in one
tool description.

```typescript
const server = new McpServer(
  { name: "my-server", version: "1.0.0" },
  { instructions: "Use search_items to discover IDs before calling get_item." },
);
```

Some hosts use server instructions during startup and tool discovery. Keep the
opening self-contained: what tasks this server handles, the normal discovery
workflow, and important constraints such as rate limits or "read before write."
Avoid burying critical guidance after a long preamble.

Do not use instructions to override host/system behavior or duplicate every tool
description.

---

## Sampling

Sampling lets a server request model output from the connected host. Use it when
tool logic needs summarization, classification, extraction, or generation but
you do not want the MCP server to own an LLM provider integration.

Rules:

- check client capabilities before requesting sampling
- provide a fallback if unsupported
- do not send secrets or unnecessary private data
- bound token/output size

---

## Roots

Roots let a local-capable host tell a server which directories or workspace roots
the user approved.

Use roots for local stdio servers that need filesystem boundaries. Do not
hardcode broad filesystem access when roots are available.

Roots are version-sensitive and may be deprecated in newer protocol versions, so
prefer explicit configuration, tool parameters, or resource URIs when that is
cleaner.

---

## Logging

MCP logging sends structured messages to the client. Prefer it over writing
everything to stderr for remote servers.

Use logging for:

- progress details that are useful to a user/developer
- integration warnings
- non-secret diagnostic context

Do not log bearer tokens, upstream credentials, full private documents, or raw
tool arguments that may contain secrets.

---

## Progress

Long-running tools can emit progress notifications when the client supplies a
progress token.

Rules:

- treat progress as optional
- do not fail the tool if progress cannot be emitted
- keep progress messages concise
- still support cancellation

---

## Cancellation

Long-running tools should honor the request cancellation signal provided by the
SDK/runtime.

For TypeScript handlers, pass the SDK `AbortSignal` to fetch/database clients
when possible and check it in long loops.

```typescript
async (_args, ctx) => {
  for (const item of items) {
    if (ctx.signal?.aborted) {
      throw new Error("Cancelled");
    }
    await process(item);
  }
}
```

The exact signal location differs by SDK version. Check the installed SDK docs.

---

## Completion

Completion provides autocomplete suggestions for prompt arguments or resource
template variables.

Use it when users must choose from many valid values, such as table names,
workspace IDs, projects, labels, or document paths.

Skip it for the first version unless argument discovery is painful.

---

## Capability checklist

| Feature | Needs client capability? | Safe fallback |
|---|---|---|
| instructions | no explicit client capability | omit if not useful |
| logging | server capability + client support varies | server logs |
| progress | client sends a progress token | silently skip progress |
| sampling | yes | return a tool error or use server-owned LLM if explicitly chosen |
| elicitation | yes | return a tool error or ask for input as a normal argument |
| roots | yes | explicit config/tool parameter |
| completion | yes | no autocomplete |
