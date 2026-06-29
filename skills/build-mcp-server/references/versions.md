# Version-Sensitive Claims

Check this file before updating scaffolds or protocol guidance.

| Claim | Where used | Last checked |
|---|---|---|
| Stable TypeScript production scaffold targets `@modelcontextprotocol/sdk` v1.x unless the repo has explicitly adopted v2 split packages | `SKILL.md`, `remote-http-scaffold.md` | 2026-06 |
| SDK v2 development branch uses split packages: `@modelcontextprotocol/server`, `@modelcontextprotocol/node`, `@modelcontextprotocol/express`, `@modelcontextprotocol/hono` | `SKILL.md`, `auth.md`, `remote-http-scaffold.md` | 2026-06 |
| `StreamableHTTPServerTransport` v1 becomes `NodeStreamableHTTPServerTransport` under `@modelcontextprotocol/node` in v2 | `remote-http-scaffold.md` | 2026-06 |
| Current MCP HTTP transport requires `Origin` validation when `Origin` is present, and unsupported protocol versions must be rejected by the SDK/middleware/route code | `SKILL.md`, `remote-http-scaffold.md` | 2026-06 |
| Form-mode elicitation must not request secrets; URL-mode elicitation is the protocol path for sensitive/out-of-band flows | `elicitation.md`, `auth.md` | 2026-06 |
| Authorization Server helpers are deprecated/frozen under legacy SDK packages in current v2 docs; new production AS code should use a dedicated OAuth/IdP library | `auth.md` | 2026-06 |
| Cloudflare `agents/mcp` and `McpAgent` template APIs are Cloudflare-specific and must be checked before copying | `deploy-cloudflare-workers.md` | 2026-06 |
| Codex Streamable HTTP OAuth discovery can be masked by configured bearer auth; path-scoped authorization-server metadata for `/mcp` is exercised in Codex OAuth tests | `target-client-compatibility.md` | 2026-06 |

## How to verify

```bash
# Stable SDK line used by a target project
npm view @modelcontextprotocol/sdk version

# Current split packages if a repo has adopted SDK v2
npm view @modelcontextprotocol/server version
npm view @modelcontextprotocol/node version
npm view @modelcontextprotocol/express version
npm view @modelcontextprotocol/hono version

# Cloudflare template/package existence
npm view agents version
```

Also check the local MCP references when available:

- `research/mcp/references/modelcontextprotocol/specification/`
- `research/mcp/references/typescript-sdk/`

For real-host compatibility work, also check current host docs or source. Useful
targets include Codex, Claude Code, Cursor, ChatGPT, and the user's custom host.
