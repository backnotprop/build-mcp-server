# Target Client Compatibility

Read this when building a remote Streamable HTTP server for real MCP hosts,
especially when OAuth is involved.

Spec compliance is necessary but not enough. Codex, Claude Code, Cursor, browser
apps, and custom clients can differ in discovery, registration, callback URLs,
token storage, refresh behavior, tool rendering, and whether server instructions
are used for tool selection.

## Client Inventory

Before coding OAuth, write down the target hosts and versions you care about.
For each host, verify:

- Streamable HTTP endpoint URL, usually `/mcp`
- auth mode: none, bearer token, OAuth, or both
- OAuth discovery paths it tries
- registration mode: Dynamic Client Registration, Client ID Metadata Document,
  preconfigured client ID, or a mix
- callback URL shape
- requested scopes
- whether it requests or preserves `resource`
- refresh-token behavior after access-token expiry
- how it displays tool names, descriptions, schemas, and server instructions

## OAuth Discovery

For remote OAuth, serve both:

- OAuth Protected Resource Metadata for the MCP endpoint
- Authorization Server Metadata for the authorization server

When the MCP endpoint is pathful, such as `/mcp`, also consider serving
path-scoped authorization-server metadata such as:

```text
/.well-known/oauth-authorization-server/mcp
```

Some clients discover path-scoped metadata for pathful MCP URLs. Codex source
checked in 2026-06 exercises this path in its Streamable HTTP OAuth tests.

If a client has `Authorization` headers or a bearer-token env var configured,
it may treat the server as bearer-token auth and skip OAuth discovery. Test the
OAuth flow with no bearer-token config. Test bearer-token fallback separately.

## DCR and CIMD

Treat registration as a target-client compatibility choice.

- Dynamic Client Registration is still a practical compatibility path for many
  hosts. Do not treat it as dead without testing target clients.
- Client ID Metadata Documents are cleaner where supported, but require outbound
  metadata fetches, SSRF controls, validation, timeouts, and caching.
- If supporting both, put "client resolution" behind a small seam so DCR rows
  and fetched metadata produce the same internal client record shape.

## Refresh and Reconnect

Do not stop at initial login. Test after the access token expires.

Practical server behavior:

- issue short-lived access tokens plus refresh tokens when the UX should stay
  logged in
- advertise the scopes needed for refresh if the target host derives requested
  scopes from metadata
- bind refresh tokens to the MCP resource/audience in server storage
- if a refresh request omits `resource`, derive it from the refresh token record
  rather than failing a legitimate client reconnect
- reject an explicit mismatched `resource`
- rotate refresh tokens for public clients
- reject stale rotated refresh tokens
- do not revoke the active grant on the first stale-token retry unless your
  threat model intentionally prefers security over possible client retry UX

The important smoke test is: log in, wait past access-token expiry, restart or
reconnect the client, and call one read tool. If that fails, users will see a
"not logged in" loop even though initial OAuth worked.

## Tool UX in Real Hosts

Models call the tool schema they see. Make it hard to make a bad call.

- Provide discovery tools so users do not need to know opaque IDs.
- Required parameters need clear descriptions and examples when choices are
  non-obvious.
- If a required enum has a common default, say so in the description instead of
  assuming the model will infer it.
- Mutating tools should include optimistic-concurrency fields when the API has
  them.
- Return IDs and versions needed for the next tool call.

Server instructions matter in hosts that use them for tool search or planning.
Keep the first few hundred characters self-contained: what the server does, the
normal discovery-first workflow, and any important constraints.

## Real-Host Smoke Matrix

For each target host:

- no bearer config: OAuth discovery reports login needed
- unauthenticated `/mcp` returns a useful challenge
- metadata endpoints validate
- registration works, if using DCR
- browser login and consent work
- initialize succeeds after OAuth
- tools/list shows expected tools and descriptions
- one discovery/read tool works without human-supplied IDs
- one write tool works only if writes are in scope
- reconnect after access-token expiry works
- revoke/logout leads to a clean re-auth path
- bearer-token fallback still works if supported
