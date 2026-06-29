# Auth for MCP Servers

Auth has two separate planes. Keep them separate in design and code.

1. **MCP client -> MCP server**: who may call the MCP endpoint.
2. **MCP server -> upstream service**: what credentials the MCP server uses when
   it calls the API, database, filesystem, or service it wraps.

Many broken MCP servers collapse these into one vague "API key" and accidentally
serve every user through the same upstream account. Do not do that unless the
server is explicitly a private single-tenant tool.

---

## MCP client -> MCP server auth

### Authless

Use only for public data or local prototypes.

Authless remote servers must still validate inputs and protect against abuse.
Never expose private records, local filesystem access, write tools, or secrets
through an authless remote server.

### Static bearer token

Simple and useful for private/team/internal servers.

The MCP client sends:

```http
Authorization: Bearer <token>
```

Server responsibilities:

- verify the token before handling MCP JSON-RPC
- bind the token to a user, team, tenant, workspace, or allowed scope
- reject unknown tokens with `401`
- never log raw bearer tokens
- rotate/revoke tokens outside the MCP transport

This is often the right first path for a product that already has API keys for
agents or automation.

### OAuth-backed MCP auth

Use when users need browser consent, per-user authorization, or public remote
server onboarding.

For HTTP MCP servers, MCP authorization is based on OAuth 2.1 patterns:

- The MCP server acts as an OAuth protected resource.
- The authorization server issues tokens for this MCP server.
- Clients discover the authorization server through OAuth Protected Resource
  Metadata or the `WWW-Authenticate` header.
- Clients use PKCE.
- Client ID Metadata Documents are the preferred open-ecosystem registration
  path when supported.
- Dynamic Client Registration remains a practical compatibility path for current
  hosts when supported.

For product-backed MCP servers, the human login provider and the agent
credential are usually separate. WorkOS, Cloudflare Access, OIDC, or another
provider can prove "this human logged in"; the product should issue the MCP
token that answers "which user/tenant/workspace/scopes did this agent get?"
Do not hand MCP clients browser cookies, upstream identity-provider tokens, or
Cloudflare Access assertions as their long-lived agent credential.

Server responsibilities:

1. Return useful `401` responses for unauthenticated MCP requests.
2. Provide OAuth Protected Resource Metadata for the MCP endpoint.
3. Ensure issued/accepted tokens are meant for this MCP server.
4. Validate bearer token signature, expiry, issuer, audience/resource, and
   scopes.
5. Map the validated token subject to the server's user/tenant/workspace model.

Do not accept "any valid token from the identity provider." Validate that the
token was minted for this MCP server.

Refresh behavior is part of auth UX, not only security hardening. If the client
should stay connected, test with real hosts after access-token expiry. If you
support OAuth and bearer/API-key fallback, test those paths separately; bearer
headers can cause some hosts to skip OAuth discovery. See
`references/target-client-compatibility.md`.

---

## MCP server -> upstream service auth

The MCP server may need credentials to call the service it wraps.

Common shapes:

- **Server-owned credential**: one env/secret-store API key used by a private
  internal MCP server. Simple, but all callers share the same upstream account.
- **Per-user OAuth token**: MCP user authorizes the upstream service, and the MCP
  server stores/refreshes a token bound to that user.
- **Per-tenant credential**: admin configures a credential for a workspace/team.
- **Token exchange**: MCP token is exchanged for an upstream token with narrower
  audience/scope.

Token passthrough is not acceptable: do not receive a token for the MCP server
and forward that same token to a different upstream API. If the server calls
another service, use a credential or token intended for that upstream service.

---

## URL-mode elicitation for upstream credentials

If the MCP server needs an upstream API key, payment credential, password, or
OAuth authorization, do not ask for it through form-mode elicitation.

Use one of:

- OAuth authorization flow
- URL-mode elicitation that sends the user to the MCP server's trusted web page
- admin configuration outside MCP

For URL-mode elicitation, bind the elicitation request to the MCP user identity
and verify the same user completes the browser flow before accepting credentials.

---

## Token storage

| Deployment | Store tokens in |
|---|---|
| Remote stateless bearer-only | Nowhere; validate bearer per request |
| Remote OAuth/per-user upstream tokens | DB/secret store encrypted or access-controlled by user/tenant |
| Local stdio | OS keychain/keyring when available; avoid plaintext files |

Never put tokens in tool results, logs, traces, prompt text, exceptions, or test
snapshots.

---

## SDK/helper notes

MCP SDK auth APIs are version-sensitive.

For current SDK v2 split-package docs:

- Resource Server helpers such as bearer validation and protected-resource
  metadata are in runtime/framework packages such as `@modelcontextprotocol/express`.
- Authorization Server helpers from the old core SDK are deprecated/frozen under
  `@modelcontextprotocol/server-legacy/auth`.
- New production Authorization Server code should use a dedicated OAuth/IdP
  library or provider, not legacy SDK AS helpers.

For stable SDK v1 projects, confirm the exact package docs before copying auth
examples. Do not mix old core-SDK auth examples with v2 split-package imports.

---

## Auth checklist

- [ ] Decide whether the MCP endpoint is authless, static bearer, or OAuth-backed.
- [ ] Parse and validate auth before dispatching MCP requests.
- [ ] Keep MCP client auth separate from upstream service credentials.
- [ ] Validate token audience/resource, not just signature.
- [ ] Scope calls to tenant/workspace/user before returning data.
- [ ] For OAuth, test initial login and post-expiry refresh with target hosts.
- [ ] Do not expose secrets through tool output, resources, prompts, logs, or errors.
- [ ] If using URL-mode elicitation, bind the URL flow to the same MCP user.
- [ ] Add smoke checks for unauthenticated, invalid-token, and valid-token calls.
