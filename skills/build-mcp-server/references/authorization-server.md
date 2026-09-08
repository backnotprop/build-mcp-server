# Acting as Your Own Authorization Server

Read `auth.md` first for the two authorization planes and the protected-resource
rules. This reference covers the choice underneath them: where the token comes
from, and what you must build and prove if the answer is "from us".

The guidance is the `2026-07-28` authorization specification plus the operating
experience of one production MCP server on Cloudflare Workers that mints its own
tokens over the product's existing browser sign-in.

## Contents

- Three ways to satisfy the authorization requirement
- Endpoints and documents
- The 401 challenge
- CORS for browser-hosted clients
- Registration hardening
- Redirect URI matching
- Consent and remembered grants
- Authorization codes and the token exchange
- One credential model
- Revocation
- Error bodies
- Proving the auth surface works
- Known gaps to plan for

## Three ways to satisfy the authorization requirement

The specification says the MCP server is a protected resource and that some
authorization server issues its tokens. It does not say who runs that
authorization server. Three shapes satisfy it. Choose one explicitly and record
why.

| Shape                       | The MCP server is                              | Choose when                                                                                                       | What it costs                                                                                        |
| --------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Delegate to an external AS  | Protected resource only                        | You already run an identity provider that speaks OAuth 2.1 with PKCE, resource indicators, and metadata documents  | Provider configuration, most of it console work somebody must actually perform                       |
| Act as your own AS          | Protected resource and authorization server    | The product already signs people in through a browser and you do not want a second identity system                | Every endpoint below, plus consent UI, plus the hardening and tests here                             |
| Accept a long-lived API key | Protected resource with a non-OAuth credential | Automation, CI, and scripts, or as a labelled fallback beside a browser lane                                      | No interoperable browser flow; hosts that expect OAuth show the user a paste-a-token step or nothing |

Two warnings from experience.

Delegation looks cheapest on a diagram and is often not cheapest in calendar
time. One team's delegated lane sat unshipped for over a month because it needed
four provider console steps (turn metadata documents on, turn registration on,
set a default resource indicator, read back the issuer domain) that no code
change could perform. Treat provider configuration as scheduled work with an
owner, not as a detail of the implementation slice.

The API-key lane is not a substitute for the browser lane when humans connect
through hosts that expect OAuth. It is the right answer for automation and the
wrong answer as the only answer.

## Endpoints and documents

A server acting as its own authorization server serves all of these. The paths
are conventional; the metadata documents are what make them discoverable.

| Path                                               | Method    | Purpose                                                                  |
| -------------------------------------------------- | --------- | ------------------------------------------------------------------------ |
| `/.well-known/oauth-protected-resource`            | GET       | RFC 9728 document at the root                                            |
| `/.well-known/oauth-protected-resource/<mcp-path>` | GET       | RFC 9728 document at the path-aware location; clients try this one first |
| `/.well-known/oauth-authorization-server`          | GET       | RFC 8414 authorization server metadata                                   |
| `/oauth/register`                                  | POST      | RFC 7591 registration, if you keep the deprecated mechanism              |
| `/oauth/authorize`                                 | GET, POST | Sign-in bounce, consent page, and the Allow submission                   |
| `/oauth/token`                                     | POST      | Authorization code plus PKCE verifier exchanged for a token              |
| `/oauth/revoke`                                    | POST      | RFC 7009 revocation                                                      |
| `/mcp`                                             | POST      | The MCP endpoint itself                                                  |

Serve the protected-resource document at both locations. Clients prefer the
`resource_metadata` URL from the `WWW-Authenticate` challenge, then probe the
path-aware well-known, then the root. Some client SDKs also probe a path-aware
authorization-server metadata URL, so serving that variant costs one route and
removes a failure mode.

A protected-resource document that works:

```json
{
  "resource": "https://mcp.example.com/mcp",
  "authorization_servers": ["https://mcp.example.com"],
  "bearer_methods_supported": ["header"],
  "scopes_supported": ["mcp"]
}
```

- `resource` is the MCP endpoint URL, not the bare origin, and it must equal the
  `resource` value you validate at the token endpoint.
- `authorization_servers` is ordered. At least one widely used client SDK takes
  `authorization_servers[0]` without choosing, so if you list more than one, put
  the one you want used first.
- Some SDK builders omit `bearer_methods_supported`. Add it.
- Keep `scopes_supported` minimal. It is meant to be the small set needed for
  basic functionality, with more requested through step-up, not a catalog.

An authorization-server document that works:

```json
{
  "issuer": "https://mcp.example.com",
  "authorization_endpoint": "https://mcp.example.com/oauth/authorize",
  "token_endpoint": "https://mcp.example.com/oauth/token",
  "registration_endpoint": "https://mcp.example.com/oauth/register",
  "revocation_endpoint": "https://mcp.example.com/oauth/revoke",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["none"],
  "scopes_supported": ["mcp"],
  "client_id_metadata_document_supported": true,
  "authorization_response_iss_parameter_supported": true
}
```

Every field there earns its place:

- `code_challenge_methods_supported` is not optional in practice. A conforming
  client must refuse to proceed if it is absent, because that is the only way to
  discover PKCE support.
- `client_id_metadata_document_supported` is how a client knows it may use a
  CIMD instead of registering. Clients fall back to `registration_endpoint` when
  it is absent or false. **Do not set it until CIMD actually works and is
  tested.** Clients read metadata literally.
- Omitting `offline_access` from `scopes_supported` is how you say you issue no
  refresh tokens. The specification also tells protected resources not to
  advertise `offline_access`, because refresh is a client and AS concern.
- `authorization_response_iss_parameter_supported` goes with actually emitting
  `iss` on redirects, including error redirects. That pair is the mix-up attack
  mitigation.

Rules that are easy to get wrong:

- **Never derive the issuer origin from the `Host` header.** Take it from
  configuration. A request-controlled issuer lets a caller point discovery at a
  host you do not control.
- **Gate the whole lane behind one configuration flag**, compare it exactly
  (`=== "true"`, so that `"yes"` is off), check it first in the middleware and
  again first in each handler, and make an unconfigured deployment answer an
  ordinary 404 with no OAuth-shaped headers anywhere. A half-on authorization
  server is worse than none.
- `/oauth/token` takes `application/x-www-form-urlencoded`. `/oauth/register`
  takes `application/json`. Reject the wrong content type with 415 rather than
  guessing.
- Answer `OPTIONS`, `GET`, and `HEAD` on the metadata paths and `405` with an
  accurate `Allow` header on everything else, before any cache lookup or
  upstream fetch.
- If you mirror an upstream authorization server's document, cache it and answer
  `503` with `Retry-After` while the cache is cold rather than blocking or
  guessing. Then prove with a test that the hot MCP path makes no outbound calls.

## The 401 challenge

The unauthenticated MCP answer is what starts the whole flow:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="Example MCP",
                         resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource/mcp"
```

- Point `resource_metadata` at the path-aware location.
- The specification says servers **SHOULD** also include `scope` with the scopes
  required for the operation, and **SHOULD** return `403` with
  `error="insufficient_scope"` plus the full scope set when a valid token is
  merely under-privileged. Do that if you have more than one scope. A server with
  a single scope has nothing to step up to, and one fixed challenge string for
  every failure has the compensating property that it is not an oracle for which
  check rejected.
- Keep the 401 body stable and boring, and keep it the same for every failure
  reason. If some SDK helper would change the body shape depending on the
  credential type, write the challenge yourself.
- Validate `Origin` on the MCP endpoint and answer `403` before authentication
  when it is present and wrong. That is the DNS-rebinding guard, and it belongs
  ahead of the credential check.

## CORS for browser-hosted clients

A desktop or CLI host makes these requests from a process, where CORS does not
apply. A browser-hosted client makes them from a page, where it does. If you
want browser-hosted clients to work, exactly four door classes need CORS, and
the rest must never have it.

| Door                                            | CORS                             | Allowed methods      | Why                                                     |
| ----------------------------------------------- | -------------------------------- | -------------------- | ------------------------------------------------------- |
| Protected-resource metadata (both locations)    | `Access-Control-Allow-Origin: *` | `GET, HEAD, OPTIONS` | Public document by RFC 9728                             |
| Authorization-server metadata                   | `Access-Control-Allow-Origin: *` | `GET, HEAD, OPTIONS` | Public document by RFC 8414                             |
| `POST /oauth/register`                          | `Access-Control-Allow-Origin: *` | `POST, OPTIONS`      | Unauthenticated by RFC 7591; reads no credential        |
| `POST /oauth/token`                             | `Access-Control-Allow-Origin: *` | `POST, OPTIONS`      | Authenticates by `client_id` plus PKCE, not by a cookie |
| `GET`/`POST /oauth/authorize`, the consent page | **None, on every path**          | n/a                  | Reads the signed-in user's session cookie               |
| `POST /oauth/revoke`                            | **None, on every path**          | n/a                  | A native-client door; acts on the caller's credential   |
| `/mcp`                                          | **None, on every path**          | n/a                  | The whole point of the bearer token                     |

The rule in one line: **never put a CORS header on `/oauth/authorize`, on the
consent page, on revoke, or on the MCP endpoint itself.** The authorize step is
a top-level browser navigation, not a fetch, so it never needs one anyway.

What makes the wildcard safe on the other four is one property you must hold
deliberately:

- Send `Access-Control-Allow-Credentials` **nowhere**. With no allow-credentials
  the browser attaches no cookie and no HTTP auth to these cross-origin calls,
  and refuses to expose the response if a server ever paired `*` with
  credentials. The wildcard therefore cannot become ambient authority.
- Confirm by reading the code, not by intent: none of those four handlers may
  read a cookie or an `Authorization` header. Grep for it. A door that reads
  ambient authority does not get a wildcard.

With that held, the wildcard grants a browser page exactly what a `curl` from
the same network position already has: read public metadata, register a public
client, and exchange an authorization code it already holds together with the
verifier only it knows. That is the entire threat model.

Details worth copying:

- Use a **fixed** `Access-Control-Allow-Headers` list rather than reflecting the
  request's `Access-Control-Request-Headers`. A fixed list needs no `Vary`, and
  it is a stricter policy than the reflecting default some SDK metadata helpers
  ship. This list covers the official client SDK:

  ```text
  authorization, content-type, accept, mcp-protocol-version
  ```

  `mcp-protocol-version` is the header that forces the preflight at all, because
  the SDK stamps it on both discovery fetches. `content-type` is required
  (JSON on register, form-urlencoded on token). `accept` costs nothing.
  `authorization` is worth allowing even though your token door does not read
  it: a third-party client that sends client auth anyway then reaches the
  handler and gets a readable JSON OAuth error instead of an opaque preflight
  failure.

- Answer preflight with `204` and **no body**, carrying only allow-origin,
  allow-methods, allow-headers, and max-age.
- Set an explicit `Access-Control-Max-Age`. Ten minutes is a reasonable value:
  Chromium caps at two hours and WebKit at ten minutes, so a larger number buys
  nothing portable.
- **Do stamp the header on the door's own error answers.** A browser client
  needs to read a `400 invalid_grant` body; without the header it gets an opaque
  failure and cannot tell the user anything.
- **Do not stamp it on methods the door does not accept.** If the middleware is
  registered for a path with method `ALL`, a `GET /oauth/token` falls through to
  the framework's generic 404 and comes back carrying
  `Access-Control-Allow-Origin: *`, advertising a header on a method the
  preflight says the door refuses. Guard on method after the OPTIONS branch and
  before the stamp.
- Register the middleware **before** the route handlers if your framework
  composes matched handlers in registration order.
- Check that a thrown error cannot pick up the header. Stamping after
  `await next()` means a throw skips the stamp, which is the safe direction, but
  confirm rather than assume.
- Probe prefix and case behavior in your router: `/oauth/tokenx`,
  `/oauth/token/`, `/oauth/token/x`, `/oauth/TOKEN`, `/oauth//token`, and
  percent-encoded spellings should each be either the same door or a
  header-free 404.

## Registration hardening

Dynamic Client Registration is deprecated in `2026-07-28` in favor of Client ID
Metadata Documents. Keep DCR only when a host you must support needs it, and
harden it, because it is an unauthenticated write endpoint that creates rows.

Apply these in order, cheapest first:

1. **Config gate**, then **rate limit per IP before any parse or row.** Ten
   registrations per minute per client IP is a workable start. The limiter must
   run before the body is read.
2. **Require `application/json`.** Answer 415 otherwise, and 400 on unparseable
   JSON.
3. **Validate redirect URIs strictly.** One to ten of them, each within a length
   cap such as 2048 characters, each parseable, each with **no userinfo and no
   fragment**, and each one of:
   - `https:` anywhere;
   - `http:` **only** on a loopback host (`localhost`, `127.0.0.1`, `::1`,
     `[::1]`);
   - a private-use scheme (RFC 8252) that is not http(s) and is not
     browser-handled. Refuse `javascript:`, `data:`, `blob:`, `file:`, `about:`,
     `vbscript:`, `ftp:`, `ws:`, `wss:`, `view-source:`, and extension schemes
     such as `chrome-extension:` and `moz-extension:`.
4. **Force `token_endpoint_auth_method` to `none`.** These are public clients.
   Advertise `token_endpoint_auth_methods_supported: ["none"]`, refuse anything
   else rather than silently downgrading, and mint no client secret. PKCE is the
   secret.
5. **Bound every stored string.** A client name capped at roughly 120 characters
   and stripped of control and formatting characters keeps a hostile name from
   deforming the consent page.
6. **Require an appropriate OIDC `application_type`** if your server is
   OIDC-shaped. Omitting it defaults to `web` under OIDC, which conflicts with
   loopback redirect URIs.
7. **Narrow in the response, not the request.** Accept what the client sent,
   then answer with what you will actually honor: `grant_types:
   ["authorization_code"]`, `response_types: ["code"]`,
   `token_endpoint_auth_method: "none"`, your single scope. Refusing a client
   because it asked for `refresh_token` in `grant_types` breaks nearly every
   client, since the official client SDK asks for it by default.
8. **Echo back only the caller's own validated metadata**, with
   `Cache-Control: no-store`.
9. **Omit absent optional fields; never emit `null` for them.** This is not a
   style preference. A major host rejected a registration response outright with
   `Server OAuth metadata invalid: client_uri / Invalid input: expected string,
   received null`, before the browser ever opened. Absent means absent.

### Unused-registration eviction

This is the failure mode teams find last. If the only thing that removes client
rows is a sweep of registrations older than the client TTL, and you also cap the
number of live rows, then the cap is a denial-of-service surface: a single IP at
a ten-per-minute limit fills a thousand-row cap in under two hours, after which
every new registration fails until rows age out. The attack needs no browser and
no response body, so CORS does not change it.

Give **never-used** registrations a much shorter horizon than used ones. Sweeping
rows created and never exchanged for a code after an hour is the small fix; the
long TTL then applies only to registrations that did something. Mark a
registration used when it first mints a code.

### Client ID Metadata Documents

CIMD is the preferred mechanism, and the fetch it requires makes your
authorization server an HTTP client that attackers can aim. Fence it:

- `https` only, a non-root path, no userinfo, query, or fragment, the default
  port, and a length cap.
- Refuse IP literals, `localhost`, and `*.local`, `*.internal`, `*.localhost`
  hosts.
- Check resolved addresses against loopback, RFC 1918, link-local, CGNAT, and
  unique-local ranges, and dial the pinned address you checked so a TTL-zero
  rebind cannot slip past the check.
- `GET` with `Accept: application/json`, redirects **not** followed (any 3xx
  fails), a short abort timeout, a body size cap, and a content-type check.
- The document's `client_id` must string-equal the URL exactly.
- Cache with a clamped TTL, and refresh only where the fetch is safe to make.
  Fetch at the authorize step, never at the token endpoint: the token endpoint
  should make no outbound calls at all.
- Cap CIMD client rows separately from DCR rows, with a separate rate-limit key.
- Decide a trust policy and write it down: allowlist domains for a protected
  server, accept any HTTPS `client_id` for an open one, or something in between.

## Redirect URI matching

Match the presented `redirect_uri` against the registered one **byte for byte**,
with at most one deliberate relaxation, because real clients need it:

- When **both** the registered and the presented URI are loopback `http:`,
  ignore the port. Command-line hosts bind an ephemeral callback port at connect
  time and cannot register it.
- When the registered URI has no query string, allow the presented one to carry
  one. At least one host appends its own correlation parameter.

Host and path stay exact in both cases, and a registered query must match byte
for byte. The relaxation must never reach `https`, private-use schemes, or
non-loopback hosts, and it must apply identically at authorize and at token.

Test the relaxation and its edges together: the ephemeral-port case, the
appended-query case, a different path, a different loopback host, and an
`https` URI with an added query all in one table.

## Consent and remembered grants

A consent page is not decoration. It is the mitigation the specification names
for the confused deputy problem: a server holding a static client identity
upstream must obtain user consent per client before forwarding.

### Order of checks on `GET /oauth/authorize`

The order is load-bearing. Do it in this sequence:

1. **Required parameters and a size bound.** Everything that will ride the
   sealed consent cookie (`state`, `redirect_uri`, `client_id`,
   `code_challenge`) needs a combined byte cap, because a cookie over about 4 KB
   is silently dropped. Answer these **inline**, never as a redirect.
2. **The session gate.** An anonymous, bearer-shaped, or stale-cookie caller gets
   a redirect to the product's normal sign-in with a return-to, and nothing
   below runs. Do not invent a second login. Putting this before step 3 means an
   anonymous caller triggers no client lookup and no outbound CIMD fetch.
3. **Client resolution and redirect URI proof.** An unknown client or an
   unregistered redirect URI answers **inline with a 400**, never a redirect.
   This is the open-redirect defense: you may only redirect to a URI you have
   already proven belongs to a registered client.
4. **Everything else** (`response_type`, PKCE presence and method, scope,
   `resource`) now redirects to the proven URI with `error`, `state`, and `iss`.
5. **The remembered-grant check**, then the consent page.

`GET` must have no side effects. Assert in a test that rendering the page mints
zero authorization codes.

### The consent handshake

- Seal the consent state (a unique `jti`, the client id and name, the user id,
  the redirect URI, the resource, the code challenge, the client's `state`, and
  an expiry) into a cookie. The seal must carry a **time to live**; a zero or
  missing TTL is the bug that turns a sealed blob into a permanent one.
- Cookie attributes: `Path=/oauth/authorize`, a short `Max-Age` such as ten
  minutes, `HttpOnly`, `Secure`, `SameSite=Lax`. `Lax` is correct because the
  consent submission is a same-site form post and a cross-site post must not
  carry it. The path scope means the sealed blob rides only the two requests that
  need it, which is a better property than the `__Host-` prefix buys here; with
  a cryptographic seal a planted cookie simply fails to open and the worst
  outcome is an expired-consent page, not a forged consent.
- Only the opaque `state` rides the form. Comparing it against the sealed copy
  with a constant-time compare is the double-submit CSRF guard.
- On POST: clear the cookie on **every** outcome, re-verify the live session and
  require the same user, then **burn the `jti` in a redemption table before
  honoring any decision**, so a replayed cookie and form pair loses on Allow and
  on Deny alike. Then re-check the client row and the sealed redirect URI.
- Mint the authorization code in exactly one shared helper used by both the
  Allow path and the remembered-grant path, so the two cannot drift.

### What the page must show

- The client name, and **where the name came from**: "verified by
  `<cimd-host>`" for a Client ID Metadata Document, or an explicit "name
  declared by the app, not verified" for a dynamically registered client. A
  registered client can call itself anything.
- Who is signing in.
- That the token acts as that person.
- **The redirect target host and port**, with a warning that varies by kind:
  loopback ("any program on this computer could have asked for this"),
  private-use scheme ("this hands the code to whatever app owns `scheme://` on
  this computer"), or plain https. A CIMD proves control of a domain; nothing
  proves which local process is listening on a loopback port, and displaying the
  redirect hostname is the countermeasure the specification asks for.
- Nothing else. No script, and no sealed blob in the HTML.

Response headers for the page: `Cache-Control: private, no-store`,
`X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, and a content security
policy such as
`default-src 'none'; style-src 'unsafe-inline'; frame-ancestors 'none'; base-uri 'none'`.
Note that `form-action` is deliberately **omitted**: browsers enforce it across
the post-submit redirect chain, so including it breaks the flow.

### Remembered grants

Show consent the first time a client asks for a given user's approval, then
remember it so the second connection does not re-prompt.

- Key the grant on **(approving user, `client_id`)**.
- You may not need a grant table. If the artifact of approval is a credential row
  owned by that user and tagged with that client, then "a live row exists" is the
  grant, and revoking the row restores the consent page automatically. One less
  thing to keep in sync.
- Put the silent path behind its own rate-limit key, so a looping client cannot
  turn one session cookie into a code-minting amplifier. Use a **different** key
  from the first-consent path, so a tripped bucket can never block a human's
  Allow click.
- Test the leakage cases explicitly: a different user with the same client, and
  a different client with the same user, must both still see the page.

## Authorization codes and the token exchange

Bind the code at mint time to everything the exchange must match: `client_id`,
the approving user, `redirect_uri`, `resource`, and `code_challenge`.

At mint:

- Generate from a cryptographic random source, 32 bytes is a reasonable floor,
  and give the value a distinct prefix so logs and branches are readable.
- **Store only a hash of the code**, the same way you would store a token.
- Give it a short TTL. Five minutes is a common choice.

At the token endpoint:

- Require `application/x-www-form-urlencoded`, else 415.
- **Rate limit before touching any row**, and prove it: a test that trips the
  limiter should find the code's consumed marker still unset.
- **Consume the code atomically before verifying PKCE.** One conditional update
  that sets the consumed marker where the hash matches, the marker is null, and
  the expiry is in the future, returning the row. Two racing exchanges then have
  exactly one winner, and a wrong verifier still burns the code. Assert that
  second property directly: wrong verifier fails, then the correct verifier on
  the same code also fails.
- Compare the consumed row's `client_id`, `redirect_uri`, and `resource` against
  the request, and answer all of unknown, expired, already-consumed, binding
  mismatch, and PKCE mismatch with **one** `invalid_grant` and one description.
- Verify PKCE with `S256` only. Reject `plain` at the authorize step as a
  downgrade. Constrain challenge and verifier to 43 to 128 characters of
  `[A-Za-z0-9._~-]`.
- **Derive the expected `resource` from configuration**, never accept it as
  configuration, and compare URL-normalized values. Validate it at authorize and
  at token, and answer a mismatch with `invalid_target`. Clients must send
  `resource` whether or not you support it, so a server that ignores it throws
  away the audience binding that stops a token minted for one resource being
  replayed at another.
- Make no outbound calls here at all. No CIMD fetch, no upstream metadata fetch.
- Require the client row to still exist, and answer `invalid_client` if not.

### Always send `expires_in`

The token response should carry `access_token`, `token_type`, `expires_in`, and
`scope`, with `Cache-Control: no-store`.

`expires_in` is not decoration, and omitting it is not neutral. Hosts disagree
about what a missing value means: some read it as one hour, and with no refresh
token they then re-run the entire browser consent flow every hour; others read it
as never expiring. Send the real number.

## One credential model

The most useful structural decision one team made: **the token the authorization
server mints is a first-class API key row**, owned by the user who approved it,
stamped with how it was created and which OAuth client it belongs to, and stored
as a hash.

Consequences worth the trade:

- The MCP endpoint accepts one credential type. The OAuth door mints exactly
  what the door already verifies. There is no second verification path to keep
  in sync.
- Attribution, audit events, and per-owner reach already work, because the row
  already has an owner. The owner is the human who **approved**, never the
  presenter of the code.
- Revocation is one operation, reachable from the product's own credential
  screen and from `/oauth/revoke`.
- The remembered grant can be the existence of a live row, as described above.

The honest cost: an API key's lifetime is not an access token's lifetime. That
team shipped a very long expiry and no refresh token, which is defensible for a
first version and is a real gap. A client expecting rotation gets none. If you
take this shape, either issue short-lived tokens with refresh from the start, or
write the limitation where the next person will read it.

### Two arms, no fallthrough

When a server accepts more than one credential shape, discriminate on a token
prefix and commit to the branch:

```text
Authorization: Bearer <token>
  starts with the API-key prefix -> API key resolver, and only that
  otherwise                      -> OAuth token verifier, and only that
```

A claimed-but-invalid bearer must 401. It must **never** fall through to a
weaker branch that might accept it. Distinct prefixes per credential type make
this cheap and make logs readable. Both arms must fail identically, so the
response is not an oracle for which arm ran.

Related rule: **bearer credentials win over ambient cookies, and a cookie-only
request to the MCP endpoint is rejected.** Make that an explicit test. A browser
session is not an MCP credential.

## Revocation

- Answer `200` regardless of whether the token existed, per RFC 7009. Anything
  else is a validity oracle. Prove it with a garbage token.
- **Scope what the unauthenticated door can reach.** It should only be able to
  match credentials this OAuth lane minted. A key the user created through the
  product's own settings screen must be unreachable from an unauthenticated
  revoke endpoint. Prove that too: revoke it, get `200`, and confirm it still
  serves.
- **Delete the grant's unconsumed authorization codes as part of revoking.** A
  code minted seconds before the revoke otherwise survives for its full TTL and
  hands the just-revoked client a fresh credential. Scope the deletion to that
  (client, user) pair, and add a control test proving another client's
  outstanding code for the same user still works.
- Revocation must actually stop service. Prove it end to end: mint through the
  real flow, call a tool, revoke, call the same tool, assert the 401.
- If you issue refresh tokens, rotate them, and treat reuse of a rotated refresh
  token as compromise of the whole family: revoke the family.

## Error bodies

Every error on the registration and token endpoints should be
`{ error, error_description }` with `Cache-Control: no-store` and a **fixed
string** description. Do not interpolate caller input into an error body, and do
not distinguish causes the caller is not entitled to distinguish.

Use RFC 6749 token error codes. `slow_down` is a device-flow code and is the
wrong choice for a rate-limited token or registration request; prefer a
`429` with `Retry-After` and an error code that belongs to this endpoint.

## Proving the auth surface works

An OAuth surface is proven by exact assertions, not by a successful login.

**Assert values, not shapes.** Compare whole documents with `toEqual` against a
literal, and assert key order where a consumer depends on it. Assert each CORS
header value individually, including that `Access-Control-Allow-Credentials` is
absent. Assert error codes as exact strings, not with regexes or "contains".
Assert the minted credential's database row as one whole object. Assert the token
response's exact key set, and assert that `refresh_token` is absent if you do not
issue one.

**Prove the off state by byte-comparison against a control.** Fetch a route that
was never implemented, then assert that every gated path matches that control's
status, content type, cache control, and full body. Do it through the real
worker or server entry, not only through the app object, so routing and asset
fallbacks are covered too.

**Write a fixture self-test.** If your tests sign their own tokens, prove that a
token signed with a foreign key does **not** verify while an identical token
signed with the right key does. Without that, every downstream "bad token is
rejected" test can pass for the wrong reason, and the whole
audience/issuer/expiry matrix becomes theater.

**Prove absence of work, not just presence of results.** Assert that the hot MCP
path makes zero outbound calls. Assert that caching happened by comparing the
recorded call list to an expected list rather than counting. Assert that a
rate-limited request touched no rows.

The cases to cover:

- The bare MCP 401: status, the full `WWW-Authenticate` value, the error body.
- Both metadata documents, exactly, plus `authorization_servers` ordering.
- Every preflight, and every door that must **not** have CORS, including its
  redirect and error paths, asserted to a null CORS header.
- A `GET` on each POST-only door: the plain 404, body-compared to a control, with
  no CORS header.
- Registration refusals, table-driven: each hostile redirect scheme, non-loopback
  `http`, a fragment, userinfo, an over-long URI, too many URIs, and a
  `token_endpoint_auth_method` other than `none`.
- Redirect URI matching, including the loopback relaxation and its edges.
- Code burn: same code twice, and a wrong verifier burning the code.
- Consent forgery: a tampered or expired sealed cookie does not approve; a
  replayed `jti` fails on Allow and on Deny.
- Remembered grant as a state transition: page, then no page, then page again
  after revocation, with cross-user and cross-client controls.
- Revocation stopping service, plus the three negative controls above.
- Cookie-only MCP request rejected.
- Real client shapes by name: the callback URI forms your target hosts actually
  register, and their exact registration bodies including unknown members.

Two disciplines make those tests worth having:

- **Red-proof them.** Restore the pre-change source, run the new tests, and
  confirm they fail. A test that passes against the old code is testing nothing.
- **Run a scriptable verifier against the deployed environment**, not a manual
  click-through: health, both metadata documents, the unauthenticated challenge,
  cookie-only rejection, registration success, bad-redirect rejection, the
  fallback credential lane, then browser authorize, token exchange, refresh if
  you have it, revoke, and revoked-token rejection.

Then connect a real host and complete a real browser approval. An authorization
server no client has finished a flow against is not finished.

## Known gaps to plan for

These are what the shipping team found missing after the flow worked. Plan them
rather than discovering them:

- **Refresh tokens.** A long-lived opaque token works and is not what a client
  expecting rotation will do well with.
- **Production configuration.** The lane is usually enabled per environment. A
  flag set only in staging means the production endpoint answers 404 to
  discovery, and nobody notices until launch.
- **Browser-hosted clients.** Without the CORS rules above they cannot register
  or exchange, even though every other host works.
- **Cookie prefixes and address fences.** `__Host-` on cookies, stripping
  formatting characters from client names, and covering NAT64 and 6to4 forms in
  a loopback address check are all small and all easy to defer past launch.
- **The setup guide.** See `target-client-compatibility.md`. Whatever text the
  guide prints is what every agent does, so a guide that still hands out a pasted
  key means the browser flow you built goes unused.
