# Configuration

JSON config file (`CONFIG_PATH`) plus a small set of env overrides, validated
with zod at startup. **Invalid or incomplete config prevents startup.** Secrets
are resolved separately (env or mounted files) and never placed in the config
object or logs.

## Groups

### server
| Field | Env | Default |
|---|---|---|
| `host` | `SERVER_HOST` | `0.0.0.0` |
| `port` | `SERVER_PORT` | `8080` |

### consent
| Field | Env | Notes |
|---|---|---|
| `baseUrl` | `CONSENT_BASE_URL` | Consent server URL including the org path, e.g. `https://consent.scrambleid.com/orgs/<org>/api/v1`. |
| `eventStreamPath` | `CONSENT_EVENT_STREAM_PATH` | Default `/consent-requests/events`. |
| `tenant` | `CONSENT_TENANT` | Label for logs/metrics; identity still comes from the token. |

### auth
| Field | Env | Notes |
|---|---|---|
| `mode` | `AUTH_MODE` | `static` (default), `clientCredentials`, or `privateKeyJwt`. |
| `scope` | `AUTH_SCOPE` | Default `consent.outcome.read`. |
| `tokenEndpoint` | `AUTH_TOKEN_ENDPOINT` | Required for `clientCredentials` and `privateKeyJwt`. |
| `clientId` | `AUTH_CLIENT_ID` | Required for `clientCredentials` and `privateKeyJwt`. |
| `issuer` | `AUTH_ISSUER` | `privateKeyJwt` only. Assertion `aud`; defaults to `tokenEndpoint` when omitted. |
| `signingAlg` | `AUTH_SIGNING_ALG` | `privateKeyJwt` only. `RS256` (default) or `PS256`. |
| `keyId` | `AUTH_KID` | `privateKeyJwt` only. Optional JWK `kid` for the assertion header. |
| `assertionTtlSeconds` | `AUTH_ASSERTION_TTL_SECONDS` | `privateKeyJwt` only. Assertion lifetime, `1..3600` (default `60`). |
| `audience` | `AUTH_AUDIENCE` | `privateKeyJwt` only. Optional `audience`/`resource` request param. |

Secrets (never in config):
| Secret | Env (inline) | Env (file, preferred) |
|---|---|---|
| Static bearer token | `UPSTREAM_BEARER_TOKEN` | `UPSTREAM_TOKEN_FILE` |
| OAuth2 client secret | `AUTH_CLIENT_SECRET` | `AUTH_CLIENT_SECRET_FILE` |
| Private key (privateKeyJwt) | `AUTH_PRIVATE_KEY_PEM` | `AUTH_PRIVATE_KEY_FILE` |

### callbackPolicy
| Field | Default | Notes |
|---|---|---|
| `allowedHosts` | `[]` | Exact or suffix (".internal") match; `"*"` allows any. Empty = deny all. |
| `allowedCidrs` | `[]` | Resolved callback address must fall within these. Metadata/loopback/link-local always blocked. |
| `requireHttps` | `true` | |
| `allowLoopback` | `false` | Allow loopback destinations. Local testing only. |
| `defaultTimeoutMs` | `10000` | Per-callback request timeout. |

### callback authentication
An **authenticated** callback carries a **dedicated** callback bearer token (minted from a
dedicated callback credential — see `callback.auth` below) in an `Authorization: Bearer`
header. The proxy **never** reuses the upstream ScrambleID M2M token for callbacks. With no
mechanism configured, **no bearer is sent** (no shared secret, no header).

| Field | Env | Notes |
|---|---|---|
| callback auth origin | `CALLBACK_AUTH_ORIGIN` | Exact receiver origin (e.g. `https://workflow.internal`). The bearer is attached **only** when the callback URL's origin matches this; validated at startup (must be a clean `http(s)` origin, no path/query/creds). |

The receiver validates the token against the issuer's **JWKS URL** (signature + standard
claims) before acting. The header is only attached when the proxy actually holds a dedicated
callback token and the origin matches; otherwise no `Authorization` header is sent. Setting
`CALLBACK_AUTH_ORIGIN` **without** a callback mechanism **fails startup** (the proxy refuses
to silently deliver unauthenticated — it will not fall back to the upstream token).

### callback.auth
How the callback bearer token is obtained. **Optional** — absent, or `mechanism = "none"`,
means **no callback authentication**: no bearer is attached, and the upstream M2M token is
**never** reused. Set a mechanism to mint a **dedicated** callback token from a ScrambleID or
your-own auth server. Non-secret fields only; secrets are mounted as files (below).

| Field | Env | Mechanism | Default | Notes |
|---|---|---|---|---|
| `mechanism` | `CALLBACK_AUTH_MECHANISM` | — | `none` | `none` \| `privateKeyJwt` \| `clientCredentials`. |
| `tokenEndpoint` | `CALLBACK_AUTH_TOKEN_ENDPOINT` | both | — | The OAuth2 token endpoint (ScrambleID or your own). One of `tokenEndpoint` / `discoveryUrl` is required for the dedicated mechanisms. **When both are set, `tokenEndpoint` wins (override).** |
| `discoveryUrl` | `CALLBACK_AUTH_DISCOVERY_URL` | both | — | OIDC discovery document URL (`.../.well-known/openid-configuration`). Alternative to `tokenEndpoint` — the token endpoint is read from the document once, cached. HTTPS-only + SSRF-guarded. Must be an absolute `https` URL. |
| `jwksUri` | `CALLBACK_AUTH_JWKS_URI` | both | — | Optional public-key (JWKS) endpoint. Informational/override only — the proxy does not validate callback tokens (the receiver does). Must be an absolute `https` URL when set. |
| `issuer` | `CALLBACK_AUTH_ISSUER` | privateKeyJwt | — | The **expected authorization-server issuer** (the `iss` from the auth server's OIDC discovery document / the issuer the receiver trusts), populated from discovery when available. Used as the `private_key_jwt` client-assertion `aud`. When omitted, the proxy uses the discovered `issuer` if discovery is configured, otherwise falls back to the resolved `tokenEndpoint` **only as the assertion `aud`** (a pragmatic fallback many auth servers accept — not a claim that the issuer equals the token endpoint). When both `issuer` and the document's `issuer` are present they must match exactly. Keep this distinct from the assertion `iss` claim (which equals `clientId`) and from the access-token issuer the receiver validates. |
| `clientId` | `CALLBACK_AUTH_CLIENT_ID` | both | — | Required for the dedicated mechanisms. The dedicated callback client id. |
| `scope` | `CALLBACK_AUTH_SCOPE` | both | — | Optional requested scope for the issued token. |
| `audience` | `CALLBACK_AUTH_AUDIENCE` | both | — | Optional `audience`/`resource` request param placed in the token-request body — the intended audience of the **issued access token** (what the receiver checks). Set it only when the receiver/auth server requires it (some servers need it to mint a correctly-scoped token; others ignore it). **Not** the `private_key_jwt` client-assertion `aud` (that is the token endpoint / auth-server issuer — see `issuer`). |
| `signingAlg` | `CALLBACK_AUTH_SIGNING_ALG` | privateKeyJwt | `RS256` | `RS256` \| `PS256`. |
| `keyId` | `CALLBACK_AUTH_KID` | privateKeyJwt | — | Optional JWK `kid` for the assertion header. |
| `assertionTtlSeconds` | `CALLBACK_AUTH_ASSERTION_TTL_SECONDS` | privateKeyJwt | `60` | Assertion lifetime, `1..3600`. |
| `clientAuthMethod` | `CALLBACK_AUTH_CLIENT_AUTH_METHOD` | clientCredentials | `client_secret_basic` | `client_secret_basic` \| `client_secret_post`. |

**Grant vs client-authentication method.** Both dedicated mechanisms use the OAuth2
`client_credentials` grant; they differ only in the **client-authentication method** layered
on it. `private_key_jwt` is a client-authentication method (not a separate grant):
`mechanism = "privateKeyJwt"` **is** `client_credentials` authenticated with `private_key_jwt`
(and conditionally requires a private key), while `mechanism = "clientCredentials"` is
`client_credentials` authenticated with a shared secret via `clientAuthMethod`
(`client_secret_basic` / `client_secret_post`). `private_key_jwt` is therefore **not** a value
of `clientAuthMethod`; select it with `mechanism = "privateKeyJwt"`.

The proxy does **not** validate callback tokens — the receiver does. On each delivered access
token the receiver should verify the signature (against the issuing auth server's JWKS), the
issuer, the audience, the expiration, and any required scopes. Obtain the expected issuer and
JWKS from the issuing auth server's OIDC discovery document (`issuer` / `jwks_uri`) — the same
server named by `discoveryUrl` / `issuer` / `jwksUri`.

Fail-closed: `privateKeyJwt` requires `clientId`, one of `tokenEndpoint` **or**
`discoveryUrl`, and a resolved callback private key; `clientCredentials` requires
`clientId`, one of `tokenEndpoint` **or** `discoveryUrl`, and a resolved callback client
secret. `discoveryUrl` and `jwksUri`, when set, must be absolute `https` URLs. **Setting
`CALLBACK_AUTH_ORIGIN` while `mechanism = "none"` (or absent) fails startup** — an origin
signals intent to authenticate, but `none` attaches no bearer and the upstream token is never
reused. Invalid/incomplete callback auth config prevents startup.

**Discovery vs explicit endpoint (precedence).** Discovery applies to both dedicated
mechanisms. When `discoveryUrl` is set the token endpoint is resolved from the OIDC
discovery document (fetched once, cached, HTTPS-only, SSRF-guarded). An explicit
`tokenEndpoint` always overrides discovery: if both are present, the proxy uses
`tokenEndpoint` and never fetches the document. Optional discovery metadata
(`token_endpoint_auth_methods_supported`, `grant_types_supported`, `scopes_supported`,
etc.) is never required — only `token_endpoint` must be present in the document.

Callback auth secrets (never in config):
| Secret | Env (inline) | Env (file, preferred) |
|---|---|---|
| Callback private key (privateKeyJwt) | `CALLBACK_AUTH_PRIVATE_KEY_PEM` | `CALLBACK_AUTH_PRIVATE_KEY_FILE` |
| Callback key passphrase (optional) | `CALLBACK_AUTH_PRIVATE_KEY_PASSPHRASE` | `CALLBACK_AUTH_PRIVATE_KEY_PASSPHRASE_FILE` |
| Callback client secret (clientCredentials) | `CALLBACK_AUTH_CLIENT_SECRET` | `CALLBACK_AUTH_CLIENT_SECRET_FILE` |

### callbackRetry
| Field | Default |
|---|---|
| `baseMs` | `1000` |
| `maxMs` | `30000` (backoff cap) |
| `totalBudgetMs` | `120000` |

### sse
| Field | Default |
|---|---|
| `heartbeatTimeoutMs` | `45000` |
| `reconnectBaseMs` | `1000` |
| `reconnectMaxMs` | `30000` |

### operations
| Field | Env | Default |
|---|---|---|
| `logLevel` | `LOG_LEVEL` | `info` |
| `gracefulShutdownTimeoutMs` | `GRACEFUL_SHUTDOWN_TIMEOUT_MS` | `15000` |

## Example
See `config.example.json` and `.env.example` in the deployment templates. Examples only — do not
hard-code customer-specific values into the image.
