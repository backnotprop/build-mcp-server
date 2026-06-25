# Remote Streamable HTTP MCP Server — Scaffold

This reference gives small remote MCP server scaffolds. Start with one read-only
tool, verify the transport, then add real tools/resources.

## Version boundary

The TypeScript scaffold below targets the stable v1 SDK package:

```bash
npm install @modelcontextprotocol/sdk zod express
```

The SDK v2 development branch uses split packages such as
`@modelcontextprotocol/server`, `@modelcontextprotocol/node`, and
`@modelcontextprotocol/express`. If the target repo already uses v2/split
packages, adapt the imports and transport class before coding. Do not mix v1 and
v2 examples.

---

## TypeScript SDK + Express

```bash
npm init -y
npm install @modelcontextprotocol/sdk zod express
npm install -D typescript @types/express @types/node tsx
```

**`src/server.ts`**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express, { type NextFunction, type Request, type Response } from "express";
import { z } from "zod";

const ITEMS = [
  { id: "item_001", title: "First item", body: "Example item for local smoke tests." },
  { id: "item_002", title: "Second item", body: "Another example item." },
];

function createServer() {
  const server = new McpServer(
    { name: "my-service", version: "0.1.0" },
    { instructions: "Use search_items to discover IDs before calling get_item." },
  );

  server.registerTool(
    "search_items",
    {
      title: "Search items",
      description: "Search items by keyword. Returns up to limit matches with IDs for follow-up calls.",
      inputSchema: {
        query: z.string().min(1).describe("Search keywords"),
        limit: z.number().int().min(1).max(50).default(10).describe("Maximum number of matches"),
      },
      annotations: { readOnlyHint: true, destructiveHint: false },
    },
    async ({ query, limit }) => {
      const q = query.toLowerCase();
      const results = ITEMS.filter(
        (item) => item.title.toLowerCase().includes(q) || item.body.toLowerCase().includes(q),
      ).slice(0, limit);

      return {
        content: [{ type: "text", text: JSON.stringify({ results }, null, 2) }],
      };
    },
  );

  server.registerTool(
    "get_item",
    {
      title: "Get item",
      description: "Fetch one item by ID. Use search_items first if you do not know the ID.",
      inputSchema: { id: z.string().regex(/^item_[0-9]{3}$/).describe("Item ID, such as item_001") },
      annotations: { readOnlyHint: true, destructiveHint: false },
    },
    async ({ id }) => {
      const item = ITEMS.find((candidate) => candidate.id === id);
      if (!item) {
        return {
          isError: true,
          content: [{ type: "text", text: `Item ${id} was not found. Use search_items to find valid IDs.` }],
        };
      }
      return {
        content: [{ type: "text", text: JSON.stringify(item, null, 2) }],
      };
    },
  );

  return server;
}

function parseCsv(value: string | undefined): Set<string> {
  return new Set((value ?? "").split(",").map((part) => part.trim()).filter(Boolean));
}

const allowedOrigins = parseCsv(process.env.ALLOWED_ORIGINS);

function validateOrigin(req: Request, res: Response, next: NextFunction) {
  const origin = req.header("origin");
  // Origin is often absent for non-browser MCP clients. If it is present, enforce
  // the allow-list. Production browser-accessible deployments must set ALLOWED_ORIGINS.
  if (origin && allowedOrigins.size === 0 && process.env.NODE_ENV === "production") {
    res.status(403).json({
      jsonrpc: "2.0",
      error: { code: -32000, message: "Origin allow-list is not configured" },
      id: null,
    });
    return;
  }
  if (origin && allowedOrigins.size > 0 && !allowedOrigins.has(origin)) {
    res.status(403).json({
      jsonrpc: "2.0",
      error: { code: -32000, message: "Invalid Origin header" },
      id: null,
    });
    return;
  }
  next();
}

function requireBearerToken(req: Request, res: Response, next: NextFunction) {
  const expected = process.env.MCP_BEARER_TOKEN;
  if (!expected) {
    next(); // authless local/dev mode only; require auth before exposing private data
    return;
  }
  const auth = req.header("authorization") ?? "";
  if (auth !== `Bearer ${expected}`) {
    res.status(401).json({
      jsonrpc: "2.0",
      error: { code: -32001, message: "Unauthorized" },
      id: null,
    });
    return;
  }
  next();
}

const app = express();
app.use(express.json({ limit: "1mb" }));
app.use(validateOrigin);
app.use(requireBearerToken);

app.post("/mcp", async (req, res) => {
  const server = createServer();
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless; no resumability
  });

  res.on("close", () => {
    transport.close();
  });

  try {
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } catch (error) {
    console.error("MCP request failed", error);
    if (!res.headersSent) {
      res.status(500).json({
        jsonrpc: "2.0",
        error: { code: -32603, message: "Internal server error" },
        id: null,
      });
    }
  }
});

// Stateless servers do not open a long-lived SSE stream. Handle the method
// explicitly so clients get an MCP-shaped response rather than an HTML 404.
app.get("/mcp", (_req, res) => {
  res.status(405).json({
    jsonrpc: "2.0",
    error: { code: -32000, message: "GET is not supported by this stateless MCP server" },
    id: null,
  });
});

app.delete("/mcp", (_req, res) => {
  res.status(405).json({
    jsonrpc: "2.0",
    error: { code: -32000, message: "DELETE is not supported by this stateless MCP server" },
    id: null,
  });
});

app.get("/healthz", (_req, res) => res.json({ ok: true }));

app.listen(Number(process.env.PORT ?? 3000), () => {
  console.log("MCP server listening on http://localhost:3000/mcp");
});
```

**Stateless vs stateful:** this creates a fresh transport per request and sets
`sessionIdGenerator: undefined`. That is fine for most API-wrapping servers. If
you need resumability, server-initiated streams, or shared per-session state, use
the SDK's stateful Streamable HTTP example and implement `GET /mcp` and
`DELETE /mcp` for stream/resume/session termination.

**Auth:** `MCP_BEARER_TOKEN` is only a simple private-server example. Real
multi-user products should validate a user/team/workspace-scoped token or
OAuth-issued access token before dispatching MCP requests.

---

## FastMCP 3.x (Python)

```bash
pip install fastmcp
```

**`server.py`**

```python
from fastmcp import FastMCP

mcp = FastMCP(
    name="my-service",
    instructions="Use search_items to discover IDs before calling get_item.",
)

ITEMS = [
    {"id": "item_001", "title": "First item", "body": "Example item for smoke tests."},
    {"id": "item_002", "title": "Second item", "body": "Another example item."},
]

@mcp.tool(annotations={"readOnlyHint": True, "destructiveHint": False})
def search_items(query: str, limit: int = 10) -> dict:
    """Search items by keyword. Returns matches with IDs for follow-up calls."""
    q = query.lower()
    results = [
        item for item in ITEMS
        if q in item["title"].lower() or q in item["body"].lower()
    ][:limit]
    return {"results": results}

@mcp.tool(annotations={"readOnlyHint": True, "destructiveHint": False})
def get_item(id: str) -> dict:
    """Fetch one item by ID. Use search_items first if you do not know the ID."""
    for item in ITEMS:
        if item["id"] == id:
            return item
    raise ValueError(f"Item {id} was not found")

if __name__ == "__main__":
    mcp.run(transport="http", host="127.0.0.1", port=3000)
```

FastMCP derives JSON schema from type hints. Keep docstrings terse and
action-oriented; they are visible to models and hosts.

---

## Search + execute pattern

When wrapping dozens or hundreds of operations, do not register every endpoint as
a tool. Use a catalog internally and expose discovery plus execution:

```typescript
const CATALOG = loadActionCatalog(); // { id, description, paramSchema }[]

server.registerTool(
  "search_actions",
  {
    title: "Search actions",
    description: "Find available actions matching an intent. Returns action IDs, descriptions, and parameter schemas.",
    inputSchema: { intent: z.string().describe("What you want to do, in plain English") },
    annotations: { readOnlyHint: true, destructiveHint: false },
  },
  async ({ intent }) => {
    const matches = rankActions(CATALOG, intent).slice(0, 10);
    return { content: [{ type: "text", text: JSON.stringify({ matches }, null, 2) }] };
  },
);

server.registerTool(
  "execute_action",
  {
    title: "Execute action",
    description: "Execute an action by ID. Get the ID and params schema from search_actions first.",
    inputSchema: {
      action_id: z.string(),
      params: z.record(z.unknown()),
    },
    annotations: { readOnlyHint: false, destructiveHint: true },
  },
  async ({ action_id, params }) => {
    const action = CATALOG.find((candidate) => candidate.id === action_id);
    if (!action) {
      return {
        isError: true,
        content: [{ type: "text", text: `Unknown action ${action_id}. Call search_actions first.` }],
      };
    }
    const parsed = parseParams(params, action.paramSchema);
    const result = await dispatch(action, parsed);
    return { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] };
  },
);
```

Start with keyword matching for `rankActions`. Upgrade to embeddings only if the
catalog is large enough to need it.

---

## Test it

Use MCP Inspector for local interactive and scripted checks.

```bash
# Interactive UI
npx @modelcontextprotocol/inspector
# Select Streamable HTTP, paste http://localhost:3000/mcp, connect.
```

Scripted checks:

```bash
npx @modelcontextprotocol/inspector --cli http://localhost:3000/mcp \
  --transport http --method tools/list

npx @modelcontextprotocol/inspector --cli http://localhost:3000/mcp \
  --transport http --method tools/call --tool-name search_items --tool-arg query=item
```

If auth is enabled, configure the target host or Inspector with the expected
`Authorization` header.

---

## Deploy

The Express scaffold runs on any ordinary Node host: Render, Railway, Fly.io,
Kubernetes, a VPS, or a container platform.

Deployment basics:

- set `MCP_BEARER_TOKEN` or real auth before exposing private data
- set `ALLOWED_ORIGINS` for browser-accessible deployments
- serve over HTTPS
- add a separate `/healthz`
- log request failures without logging tokens or tool arguments that may contain
  secrets

For Cloudflare Workers, use `deploy-cloudflare-workers.md`; the runtime and
transport wiring are different from this Express scaffold.
