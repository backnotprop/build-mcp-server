# Resources & Prompts

MCP servers can expose three server primitives:

- **Tools**: model-invoked actions.
- **Resources**: read-only context the host can browse or attach.
- **Prompts**: user-invoked message templates/workflows.

Most servers start with tools. Add resources or prompts when they make the
server easier to use or reduce repeated prompt boilerplate.

---

## Resources

A resource is read-only data identified by a URI. It is not called like a tool;
the host reads it through `resources/read`.

Use resources for:

- documents
- files
- schemas
- logs
- configs
- records that are useful as context
- versioned content

Use a tool instead when:

- the operation has side effects
- the model should choose parameters at call time
- the result is a computed action response rather than browsable context

### Static resources

```typescript
server.registerResource(
  "config",
  "config://app/settings",
  {
    title: "Application settings",
    description: "Current application configuration",
    mimeType: "application/json",
  },
  async (uri) => ({
    contents: [{
      uri: uri.href,
      mimeType: "application/json",
      text: JSON.stringify(config, null, 2),
    }],
  }),
);
```

### Dynamic resources

Resource templates let one registration serve many URI instances:

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

server.registerResource(
  "document",
  new ResourceTemplate("workspace://{workspaceId}/documents/{documentId}", { list: undefined }),
  {
    title: "Workspace document",
    description: "Read a document from a workspace",
    mimeType: "text/markdown",
  },
  async (uri, { workspaceId, documentId }) => {
    const doc = await readDocument(workspaceId, documentId);
    return {
      contents: [{
        uri: uri.href,
        mimeType: "text/markdown",
        text: doc.body,
      }],
    };
  },
);
```

If a resource is backed by a filesystem path, resolve the requested path and
prove it remains inside the intended root before reading. Reject traversal,
encoded traversal, and symlink escapes.

### Subscriptions

Some MCP versions/SDKs support resource change notifications. Only add
subscriptions when the host experience needs them. A simple read-only resource
surface should not start with subscriptions.

---

## Prompts

A prompt is a reusable message template. The host exposes it to the user as a
workflow, slash command, menu item, or equivalent UI.

Use prompts for workflows users run repeatedly:

- summarize a document
- review a changelog
- draft a reply
- compare two versions
- explain an error report

```typescript
server.registerPrompt(
  "summarize_document",
  {
    title: "Summarize document",
    description: "Create a concise summary of provided document text",
    argsSchema: {
      text: z.string(),
      max_words: z.string().optional(),
    },
  },
  ({ text, max_words }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Summarize in ${max_words ?? "100"} words:\n\n${text}`,
      },
    }],
  }),
);
```

Prompt handlers should be pure template builders. Do not mutate server state
inside a prompt handler.

---

## Decision table

| You want to... | Use |
|---|---|
| Let the model perform an action | Tool |
| Let the model query with parameters | Tool |
| Expose browsable/read-only context | Resource |
| Expose a dynamic family of readable objects | Resource template |
| Give users a reusable workflow | Prompt |
| Ask the user a question mid-operation | Elicitation |
