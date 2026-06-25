# Elicitation — user input during an MCP request

Elicitation lets an MCP server ask the user for input through the MCP host while
a request is in progress. Use it only when the server genuinely needs more user
input to complete the current operation.

There are two modes:

- **Form mode**: in-band structured input for non-sensitive data.
- **URL mode**: sends the user to a URL for sensitive or out-of-band flows.

Always check client capabilities before sending an elicitation request. Hosts
may support form mode, URL mode, both, or neither.

---

## Form mode

Use form mode for simple non-sensitive input:

- confirmation
- enum choice
- short text
- date/date-time
- simple flat form

Form-mode schemas are deliberately limited:

- flat objects only
- primitive properties only
- no nested objects
- no arrays of objects
- supported string formats include `email`, `uri`, `date`, and `date-time`
- use `title` and `description` for form labels/help text

Example shape:

```typescript
const caps = getClientCapabilitiesSomehow();
if (!caps?.elicitation?.form && !caps?.elicitation) {
  return {
    isError: true,
    content: [{ type: "text", text: "This operation requires user confirmation, but this MCP client does not support elicitation." }],
  };
}

const result = await elicitInputSomehow({
  mode: "form",
  message: "Delete this item? This cannot be undone.",
  requestedSchema: {
    type: "object",
    properties: {
      confirm: { type: "boolean", title: "Confirm deletion" },
    },
    required: ["confirm"],
  },
});

if (result.action !== "accept" || result.content?.confirm !== true) {
  return { content: [{ type: "text", text: "Cancelled." }] };
}

await deleteItem();
return { content: [{ type: "text", text: "Deleted." }] };
```

The exact SDK API is version-specific. Current SDK v2 uses request-context APIs
such as `ctx.mcpReq.elicitInput(...)`; older SDK examples may use different
names. Do not copy an elicitation snippet across SDK versions without checking
the installed SDK docs.

---

## URL mode

Use URL mode for sensitive or out-of-band flows:

- API key entry
- OAuth with an upstream service
- payment details
- credentials
- account connection pages

Form mode must not request passwords, API keys, access tokens, payment
credentials, or similar secrets.

URL mode returns a URL for the user to open. The MCP client should not receive
the sensitive data. The server handles the web flow on its own trusted domain.

Important security rule: bind the URL-mode elicitation to the MCP user identity.
When the user opens the URL, verify the browser/session user is the same user
who triggered the MCP request before accepting credentials or completing the
flow.

Example shape:

```typescript
const caps = getClientCapabilitiesSomehow();
if (!caps?.elicitation?.url) {
  return {
    isError: true,
    content: [{ type: "text", text: "This operation requires opening a secure setup URL, but this MCP client does not support URL elicitation." }],
  };
}

const elicitationId = crypto.randomUUID();
await storePendingElicitation({
  elicitationId,
  mcpUserId,
  purpose: "connect_upstream_api",
  expiresAt: Date.now() + 10 * 60_000,
});

await elicitInputSomehow({
  mode: "url",
  message: "Connect your upstream account.",
  elicitationId,
  url: `https://mcp.example.com/connect?elicitationId=${elicitationId}`,
});
```

The connect page must:

1. authenticate the browser user
2. load the pending elicitation by `elicitationId`
3. verify it belongs to that same MCP user
4. collect credentials or complete OAuth
5. store resulting upstream credentials server-side
6. never send the secret back through MCP content

---

## Response actions

Elicitation responses use three actions:

| Action | Meaning |
|---|---|
| `accept` | User submitted the form or completed the requested action |
| `decline` | User explicitly declined |
| `cancel` | User dismissed or abandoned the request |

Treat `decline` and `cancel` differently when it matters. `decline` is an
intentional answer; `cancel` may be accidental.

---

## When not to use elicitation

Do not use elicitation for:

- ordinary required tool arguments that can be part of `inputSchema`
- rich custom interfaces
- long multi-step workflows
- background account setup that can happen before tool calls
- secrets in form mode

If the data is required every time, make it a tool argument. If it is account
configuration, prefer OAuth, a settings page, or URL-mode elicitation.
