# Version-Sensitive Claims

Check this ledger before changing scaffolds or protocol guidance. Protocol
revision and SDK major are independent choices.

| Claim                                                                                                                                                                                                                                                  | Where used                                                     | Last checked |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- | ------------ |
| The current released MCP revision is `2026-07-28`; its TypeScript `schema/2026-07-28/schema.ts` is the protocol schema source of truth                                                                                                                 | all references                                                 | 2026-09-08   |
| `2026-07-28` removes `initialize`, protocol sessions, HTTP GET/DELETE streams, server-initiated JSON-RPC requests, and SSE replay; it adds per-request metadata and mandatory `server/discover`                                                        | `SKILL.md`, `protocol-eras.md`, scaffolds                      | 2026-09-08   |
| Modern Streamable HTTP uses POST; JSON-RPC request POSTs require `MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name` where applicable, with header/body validation. This revision does not define notification-POST header requirements               | `SKILL.md`, `protocol-eras.md`, scaffold                       | 2026-09-08   |
| Modern complete results require `resultType`; cacheable discover/list/read results require `ttlMs` and `cacheScope`                                                                                                                                    | `SKILL.md`, `protocol-eras.md`, capability/resource references | 2026-09-08   |
| Elicitation uses MRTR `input_required`; `elicitationId` and the completion notification are removed                                                                                                                                                    | `elicitation.md`, `auth.md`                                    | 2026-09-08   |
| Roots, Sampling, Logging, DCR, and HTTP+SSE are deprecated; Tasks moved to the `io.modelcontextprotocol/tasks` extension                                                                                                                               | `SKILL.md`, `server-capabilities.md`, `auth.md`                | 2026-09-08   |
| Core `ping`, `logging/setLevel`, and `notifications/roots/list_changed` are removed; the Tasks extension uses `resultType: "task"`, `tasks/get`, `tasks/update`, and `tasks/cancel` while removing `tasks/result` and `tasks/list`                     | `protocol-eras.md`, `server-capabilities.md`, migration guide  | 2026-09-08   |
| Roots, Sampling, Logging, and DCR first become eligible for removal in a revision on or after `2027-07-28`; eligibility is not a guaranteed removal date                                                                                               | `protocol-eras.md`, `server-capabilities.md`, migration guide  | 2026-09-08   |
| The published official TypeScript SDK v2 split packages are stable at `2.0.0` and expose the modern serving model; a nearby source checkout still contained `2.0.0-beta.4` metadata, so published tags and source checkouts must be checked separately | `SKILL.md`, `remote-http-scaffold.md`, Workers reference       | 2026-09-08   |
| Official TypeScript v2 `createMcpHandler` and `serveStdio` serve both modern and legacy eras by default; `legacy: "reject"` makes them modern-only, while sessionful legacy HTTP needs an isolated handler                                             | `SKILL.md`, `protocol-eras.md`, scaffold, migration guide      | 2026-09-08   |
| `createMcpHandler` installs the modern listen router even when the application advertises no notification path. Use `maxSubscriptions: 0` for explicit rejection, or configure a real event bus, capacity, cancellation, and multi-instance fan-out    | HTTP and Workers scaffolds, capability reference               | 2026-09-08   |
| `@modelcontextprotocol/client@2.0.0` defaults to legacy unless `versionNegotiation` is explicit. Its auto-probe treats valid discovery or well-formed `-32022` as modern evidence, while `-32020`/`-32021` can still fall back                         | `protocol-eras.md`, migration and client references            | 2026-09-08   |
| The official v2 handler supplies modern wire bookkeeping and conservative cache defaults; application handlers return neutral result shapes                                                                                                            | `remote-http-scaffold.md`, `tool-design.md`                    | 2026-09-08   |
| Official TypeScript v1 `@modelcontextprotocol/sdk` examples target the legacy era and must not be presented as current-protocol scaffolds                                                                                                              | `SKILL.md`, `remote-http-scaffold.md`                          | 2026-09-08   |
| The v2 Authorization Server helpers from v1 are frozen under `@modelcontextprotocol/server-legacy/auth`; new production AS code should use a dedicated provider/library                                                                                | `auth.md`                                                      | 2026-09-08   |
| SDK/framework/host support for `2026-07-28` must be verified; package popularity or "Streamable HTTP" support alone does not prove the modern era                                                                                                      | all implementation references                                  | 2026-09-08   |
| Anthropic announced that `2026-07-28` support was rolling out across Claude products “soon”; this is not a claim that every Claude surface/version already supports the modern era or every extension                                                  | `target-client-compatibility.md`, migration guide              | 2026-07-31   |
| The official SDK's resource-server helpers are `requireBearerAuth` (in `@modelcontextprotocol/server` for web-standard hosts, `@modelcontextprotocol/express` for Express), `getOAuthProtectedResourceMetadataUrl`, `oauthMetadataResponse`, and `mcpAuthMetadataRouter`; the SDK issues no tokens | `auth.md`, Workers reference                                   | 2026-09-08   |
| `oauthMetadataResponse` ships permissive, reflecting CORS on the metadata documents; a fixed allow-headers policy means serving the documents yourself                                                                                                | `auth.md`, `authorization-server.md`, Workers reference        | 2026-09-08   |
| Claude Code adds an HTTP server with `claude mcp add --transport http <name> <url>` and authenticates from `/mcp`; it also has `--header`, `--client-id`, and `--callback-port`. Cursor uses `~/.cursor/mcp.json` `mcpServers`; Codex uses `~/.codex/config.toml` `[mcp_servers.<name>]` with `url` | `target-client-compatibility.md`                               | 2026-09-08   |
| Hosts disagree on a missing `expires_in` in a token response: some assume one hour and re-run consent, others assume no expiry. Always send it                                                                                                        | `authorization-server.md`, `target-client-compatibility.md`    | 2026-09-08   |
| No shipped `2026-07-28` mechanism moves a user's file into a tool call; `resources` is a server capability only and roots carry no read method. A File Uploads WG is chartered on SEP-2356, which is still an open proposal                            | `tool-design.md`                                               | 2026-09-08   |

## Sources to check

Protocol repository:

```text
schema/2026-07-28/schema.ts
docs/specification/2026-07-28/
docs/specification/2026-07-28/changelog.mdx
docs/specification/2026-07-28/deprecated.mdx
```

Official TypeScript SDK repository:

```text
docs/protocol-versions.md
docs/migration/support-2026-07-28.md
docs/serving/authorization.md
docs/serving/http.md
docs/serving/sessions-state-scaling.md
docs/serving/stdio.md
docs/servers/input-required.md
packages/client/src/client/probeClassifier.ts
packages/server/src/server/createMcpHandler.ts
packages/server/src/server/listenRouter.ts
```

Package checks:

```bash
npm view @modelcontextprotocol/server version dist-tags --json
npm view @modelcontextprotocol/client version dist-tags --json
npm view @modelcontextprotocol/core version dist-tags --json
npm view @modelcontextprotocol/node version dist-tags --json
npm view @modelcontextprotocol/express version dist-tags --json
npm view @modelcontextprotocol/hono version dist-tags --json
npm view @modelcontextprotocol/fastify version dist-tags --json
npm view @modelcontextprotocol/sdk version dist-tags --json
```

Release and host-rollout context:

```text
https://blog.modelcontextprotocol.io/posts/2026-07-28/
https://modelcontextprotocol.io/specification/2026-07-28/changelog
https://claude.com/blog/bringing-mcp-2026-07-28-to-claude
```

For Python, Cloudflare-specific wrappers, Inspector, and target hosts, inspect
the installed version's release notes, protocol-version constants, modern
conformance results, and source. Verify these behaviors directly:

- `server/discover`
- required per-request metadata and HTTP headers
- required result/cache fields
- MRTR and `subscriptions/listen`
- absence of modern session/GET/resumability assumptions

Never infer current-protocol support from a package's major number alone.
