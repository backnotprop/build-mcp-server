# Authorization for MCP Servers

Keep two independent authorization planes in the design and code:

1. **MCP client -> MCP server**: who may call the MCP HTTP endpoint and with
   which scopes.
2. **MCP server -> upstream service**: which credential the MCP server uses for
   the API, database, or service it wraps.

Never accept a token intended for the first plane and forward it into the
second. Token passthrough crosses audiences and is forbidden.

The MCP OAuth framework applies to HTTP transports. Stdio servers should
normally receive configuration or credentials through the process environment,
OS keychain, or another local mechanism rather than running the HTTP OAuth flow.

## Contents

- MCP client to MCP server: authless, bearer, and OAuth
- Who runs the authorization server
- MCP server to upstream service
- URL-mode setup and token storage
- Spec security requirements
- SDK notes and checklist

## MCP client -> MCP server

### Authless

Use only for deliberately public data or local development.

An authless remote server still needs validation, rate limits, abuse controls,
Origin/Host protection, and strict tool/resource boundaries. Do not expose
private records, filesystem access, mutations, or secrets through a public
authless endpoint.

### Static bearer token

A private/internal deployment may use a product-specific bearer token:

```http
Authorization: Bearer <token>
```

Verify it before MCP dispatch, bind it to a principal/tenant/scopes, reject
unknown or expired tokens with 401, and never log it. This can be a practical
closed-system mechanism, but it is not the interoperable browser-consent flow
defined by the MCP OAuth profile. It is the right lane for automation, CI, and
scripts, and the wrong lane as the only one when humans connect through hosts
that expect the browser flow.

If the server accepts both an API key and an OAuth token, discriminate on a
token prefix and commit to that branch. A claimed-but-invalid bearer must return
401; it must never fall through to a weaker credential branch that might accept
it, and both branches must fail identically so the response is not an oracle.

### OAuth-protected MCP server

In the MCP authorization model:

- the MCP server is an OAuth protected resource
- the MCP client is an OAuth client
- an authorization server authenticates/authorizes the resource owner and
  issues tokens for the MCP resource

The authorization server may be colocated with the MCP server, but these are
separate roles. A server that only acts as the protected resource should not
grow a home-built token issuer merely because it needs to verify bearer tokens.

## Who runs the authorization server

The specification does not say. Three shapes satisfy it, and the choice drives
most of the remaining work:

| Shape                       | Choose when                                                                                              | Read                        |
| --------------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------- |
| Delegate to an external AS  | You already run an identity provider that speaks OAuth 2.1 with PKCE, resource indicators, and metadata   | The rest of this file       |
| Act as your own AS          | The product already signs people in through a browser and a second identity system is not worth it        | `authorization-server.md`   |
| Accept a long-lived API key | Automation and CI, or a labelled fallback beside a browser lane                                           | Static bearer token, above  |

Delegation looks cheapest and is often not cheapest in calendar time: provider
consoles need metadata documents enabled, registration enabled, and a default
resource indicator set, and none of that is a code change. Schedule it with an
owner.

If you run your own authorization server, or if you want browser-hosted clients
to work at all, read `authorization-server.md`. It covers the endpoint
inventory, the CORS rules (wildcard on the metadata documents, registration, and
token; never on authorize, consent, revoke, or the MCP endpoint), registration
hardening, consent and remembered grants, code and token handling, revocation,
and the assertions that prove it.

#### Protected-resource responsibilities

1. Serve OAuth Protected Resource Metadata (RFC 9728). Its
   `authorization_servers` field must contain at least one issuer.
2. On unauthenticated requests, return 401 with a useful
   `WWW-Authenticate: Bearer` challenge. Include `resource_metadata`; include
   the scopes required for the operation when useful.
3. Validate token signature or introspection result, expiry, issuer,
   audience/resource, scopes, and revocation state as applicable.
4. Map the validated subject/client to the application user, tenant, workspace,
   and policy context before MCP dispatch.
5. Return 403 plus `error="insufficient_scope"` and the complete scope set for
   the current operation when a valid token lacks permission.
6. Apply authorization again when dereferencing application state handles;
   possession of a handle is not authorization.

Clients must send the RFC 8707 `resource` parameter in authorization and token
requests. The value should be the most specific canonical URI for the MCP
server, such as `https://mcp.example.com/mcp`. The protected resource must only
accept tokens intended for that resource.

#### Discovery

For an MCP endpoint at `https://example.com/public/mcp`, support path-aware
Protected Resource Metadata at:

```text
https://example.com/.well-known/oauth-protected-resource/public/mcp
```

Also support the root fallback where appropriate. Clients prefer the
`resource_metadata` URL from `WWW-Authenticate`, then probe the RFC 9728
well-known locations.

The Protected Resource Metadata names the authorization-server issuer. Clients
then discover that issuer's metadata through RFC 8414 or OpenID Connect
Discovery. For an issuer with a path, the first OAuth metadata candidate uses
path insertion:

```text
issuer:   https://auth.example.com/tenant1
metadata: https://auth.example.com/.well-known/oauth-authorization-server/tenant1
```

Do not derive authorization-server metadata paths from the MCP endpoint unless
the MCP origin is itself the authorization-server issuer.

#### Client registration

The `2026-07-28` priority is:

1. pre-registered client credentials when available
2. Client ID Metadata Documents (CIMD) when the authorization server advertises
   support
3. Dynamic Client Registration (DCR) only as a compatibility fallback
4. user-entered pre-registration details when no automated path works

DCR is deprecated. If retained for real hosts, require the client to send an
appropriate OIDC `application_type`, and key persisted registrations by the
authorization-server issuer. Never reuse credentials with a different issuer.

CIMD authorization servers fetch an HTTPS client metadata URL. Implement exact
`client_id` and redirect URI validation, JSON/schema validation, bounded
caching, timeouts, response-size limits, redirect restrictions, DNS/IP checks,
and SSRF defenses. Treat logos and other remote metadata as untrusted.

#### Authorization-code and refresh security

MCP clients use PKCE and must verify that the authorization server advertises a
supported code-challenge method. Authorization servers should include `iss` in
authorization responses and advertise that behavior. Clients validate a
present `iss` by exact comparison with the issuer recorded from validated
metadata before sending the code to a token endpoint.

If the authorization server issues refresh tokens to public clients, rotate
them and protect them in transit and storage. `offline_access` is a client/AS
concern; the MCP protected resource should not advertise it as a resource scope
in Protected Resource Metadata or a `WWW-Authenticate` challenge.

Initial login is not enough. Test token expiry, refresh, reconnect, scope
step-up, issuer changes, revocation, and re-authorization with each target host.

## MCP server -> upstream service

Common shapes:

- **Server-owned credential**: one secret-store API key for a single-tenant or
  tightly controlled internal server. All callers share the upstream identity.
- **Per-user upstream OAuth token**: the MCP user separately authorizes the
  upstream service; the server stores/refreshes a token bound to that user.
- **Per-tenant credential**: an administrator configures one credential for a
  workspace or customer.
- **Token exchange**: the MCP credential is exchanged for a distinct upstream
  token with the correct audience and narrower scope.

For upstream OAuth, the MCP server acts as an OAuth client to the upstream
authorization server. Those upstream tokens are distinct from the bearer token
accepted at the MCP endpoint and must never be returned through MCP.

## URL-mode elicitation for upstream setup

If a tool needs an upstream API key, password, payment credential, or third-party
OAuth grant, do not request it through form elicitation.

Use one of:

- third-party OAuth through a trusted server-hosted web flow
- URL-mode elicitation that sends the user to that web flow
- administrator configuration outside MCP

In the modern protocol, URL elicitation is an embedded request inside an
`input_required` result. Encode correlation in integrity-protected
`requestState` or server storage; `elicitationId` is not part of
`2026-07-28`. Authenticate the browser user and prove it is the same principal
who triggered the MCP request before accepting credentials.

## Token storage

| Deployment                      | Store tokens in                                                                              |
| ------------------------------- | -------------------------------------------------------------------------------------------- |
| Remote bearer verification only | Nowhere when local verification is sufficient; otherwise a bounded introspection/cache layer |
| Remote per-user upstream tokens | Encrypted/access-controlled database or secret store partitioned by user/tenant              |
| Local stdio                     | OS keychain/keyring when available; avoid plaintext files                                    |

Never put tokens in source, URLs, tool results, resources, prompts, logs,
traces, exceptions, or test snapshots.

## Spec security requirements

The `2026-07-28` security guidance names specific attacks. Restated as server
obligations:

- **Confused deputy.** A server that proxies to a third-party API with one
  static upstream client identity must obtain user consent for each MCP client
  before forwarding. Without it, a consent cookie from an earlier legitimate flow
  lets an attacker's crafted authorization request skip the consent screen and
  redirect the code to a URI they registered. See `authorization-server.md` for
  the consent-page shape.
- **Token passthrough.** Never accept a token that was not issued for this MCP
  server, and never forward the token you received to an upstream API. The
  upstream token is a separate token from a separate issuer. Passthrough breaks
  audience boundaries, defeats rate limits and request validation that key on
  audience, and makes audit trails misattribute callers.
- **State handle hijacking.** This is what replaced session hijacking now that
  the protocol has no sessions. A handle is not authentication. Generate handles
  from a secure random source, bind them server-side to the verified principal
  (key stored state as principal plus handle, with the principal derived from
  the token, never from an argument), expire them, and reject a handle presented
  by anyone else.
- **SSRF.** Any URL the server fetches on a caller's behalf, including a Client
  ID Metadata Document and any opt-in JSON Schema `$ref`, needs scheme and host
  allowlisting, resolved-address checks against private ranges, no redirect
  following, timeouts, and body size caps.
- **Authorization URL handling.** Only `http` and `https` authorization URLs,
  with `http` limited to loopback in development. Never open a URL through a
  shell.
- **Localhost redirect impersonation.** A Client ID Metadata Document proves
  control of a domain and proves nothing about which local process listens on a
  loopback port. Display the redirect hostname during authorization and warn on
  loopback-only redirect URIs.
- **Scope minimization.** Keep `scopes_supported` to the minimum needed for
  basic functionality and elevate through targeted `WWW-Authenticate` challenges.
  Do not publish every scope, do not use wildcard or omnibus scopes, and do not
  bundle unrelated privileges to preempt future prompts. A token's claimed scopes
  are never a substitute for server-side authorization.

## SDK notes

In the official TypeScript SDK v2 split packages, the resource-server half is
supplied and the authorization-server half is not.

| Helper                                   | Package                          | Does                                                                               |
| ---------------------------------------- | -------------------------------- | ---------------------------------------------------------------------------------- |
| `requireBearerAuth`                      | `@modelcontextprotocol/server`   | Web-standard gate: takes a `Request`, returns verified `AuthInfo` or a 401/403      |
| `requireBearerAuth`                      | `@modelcontextprotocol/express`  | The same core as Express middleware, attaching `req.auth`                          |
| `getOAuthProtectedResourceMetadataUrl`   | `@modelcontextprotocol/express`  | Builds the path-aware RFC 9728 URL for the `resource_metadata` challenge parameter |
| `oauthMetadataResponse`                  | `@modelcontextprotocol/server`   | Serves both well-known documents from a `fetch` handler                            |
| `mcpAuthMetadataRouter`                  | `@modelcontextprotocol/express`  | The Express equivalent, mounting both well-known routes                            |

You supply one function: take a raw token string and return an `AuthInfo`. Local
JWT verification, RFC 7662 introspection, or a provider call all fit behind it.
Always populate `expiresAt`; the gate answers `401 invalid_token` when it is
unset. Throw the SDK's `OAuthError` with the invalid-token code for a rejection;
any other exception becomes a `500`.

Two caveats:

- `requiredScopes` on the gate is endpoint-wide. A scope only some tools need is
  a check inside the handler, returning `isError: true` so the model reads the
  refusal, rather than a 403 that drops the call.
- The SDK's metadata response helper ships permissive, reflecting CORS. If you
  care about the exact policy, serve the documents yourself with a fixed
  allow-headers list. See `authorization-server.md`.

The v1 Authorization Server helpers (`mcpAuthRouter`, `ProxyOAuthServerProvider`
and friends) are frozen under `@modelcontextprotocol/server-legacy/auth`. The
SDK never issues tokens; a new authorization server uses a dedicated identity
provider or the design in `authorization-server.md`.

SDK helpers are version-sensitive. Confirm the package docs and do not mix v1
core-SDK auth examples with v2 imports.

## Checklist

- [ ] Choose authless, closed-system bearer, or OAuth-protected HTTP explicitly.
- [ ] Name who runs the authorization server: an external provider, this server,
      or nobody because the lane is API keys.
- [ ] Authenticate before MCP dispatch and authorize every tool/resource access.
- [ ] Discriminate multiple credential shapes by prefix, with no fallthrough.
- [ ] Serve CORS on the metadata documents (and on registration and token if you
      run the AS), and on nothing else.
- [ ] Keep MCP authorization separate from upstream credentials.
- [ ] Validate token issuer, audience/resource, expiry, and scopes, not only its signature.
- [ ] Partition every read, write, cache, and state handle by the verified principal/tenant.
- [ ] Publish and validate path-aware Protected Resource Metadata and AS metadata.
- [ ] Test PKCE, `resource`, `iss`, 401, 403 scope step-up, and issuer changes.
- [ ] Prefer pre-registration/CIMD; keep DCR only for tested compatibility.
- [ ] Test post-expiry refresh/reconnect and revoke/re-auth with target hosts.
- [ ] Bind URL elicitation and upstream credentials to the same verified MCP user.
- [ ] Keep secrets out of every MCP-visible or observable channel.
