# Configuration

Environment variables, set in `.env`. **Invalid or incomplete config prevents startup**, and the log
lists every problem at once. Secrets are never variables: keys and secrets are files you mount,
named by path. A setting of a feature that is off does nothing, and the proxy says so at start.

## Groups

### container
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_IMAGE` | required | Pinned image, e.g. `scrambleid-edge-proxy:1.2.0`. |
| `EDGE_PROXY_BIND_IP` | required | Host interface the port is published on. Internal IP your gateway reaches (`decisions`), else `127.0.0.1`. |
| `EDGE_PROXY_HOST_PORT` | `8080` | Host port. |

### server
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_FEATURES` | required | `decisions`, `callbacks` or both, comma-separated. |
| `EDGE_PROXY_HOST` / `EDGE_PROXY_PORT` | `0.0.0.0` / `8080` | Inside the container. Set by the compose file. |
| `EDGE_PROXY_SERVER_URL` | required | ScrambleID consent server, e.g. `https://consent.scrambleid.com`. |

### auth
How the proxy signs in to ScrambleID. URLs must be https (http only for localhost), with no
credentials. The scopes asked for come from the features: `policy.sync` for `decisions`,
`consent.outcome.read` for `callbacks`.

| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_TOKEN_URL` | required | Your service account's token endpoint. |
| `EDGE_PROXY_CLIENT_ID` | required | The service account. |
| `EDGE_PROXY_AUTH_MECHANISM` | required | `oidc_pvt_key_jwt` or `oidc_client_creds`. |
| `EDGE_PROXY_TOKEN_PARAMS` | | Extra token request fields, if your issuer needs them: URL-encoded `name=value` pairs joined by `&`. Never a secret. Cannot set `grant_type`, `client_id`, `client_secret`, `client_assertion*` or `scope`. |

#### `oidc_pvt_key_jwt`
Signed assertion (`private_key_jwt`). The public key is registered at the issuer.

| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_PRIVATE_KEY_FILE` | required | PKCS8 PEM, RSA 2048+ bits. |
| `EDGE_PROXY_PRIVATE_KEY_PASSPHRASE_FILE` | | For an encrypted key. |
| `EDGE_PROXY_SIGNING_ALG` | `RS256` | Or `PS256`. |
| `EDGE_PROXY_KEY_ID` | | `kid` registered at the issuer, if it has one. |
| `EDGE_PROXY_TOKEN_ISSUER` | the token URL | Audience of the signed assertion. |

#### `oidc_client_creds`
Client secret, sent to the token endpoint.

| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_CLIENT_SECRET_FILE` | required | File holding only the secret. |
| `EDGE_PROXY_CLIENT_AUTH_METHOD` | `basic` | `basic` = `Authorization` header. `post` = form body, for issuers that only read it there. |

A setting of the other mechanism does nothing, and the proxy says so at start.

### decisions
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_JWKS_URL` | required | Key set that signs your gateway's `policy.evaluate` tokens. |
| `EDGE_PROXY_ISSUERS` | any | Comma-separated issuers accepted. |
| `EDGE_PROXY_AUDIENCE` | any | Audience accepted. |

### consent tokens (`decisions`)
A decision of `consent` becomes `allow` when the call carries a ScrambleID consent token for exactly
that call: `consentToken` in the body of `POST /decisions`, or the consent header on a gateway
adapter. The proxy checks the token itself, against ScrambleID's public keys: signature, issuer,
audience, expiry, lifetime, and org, service, action and parameters. It is not spent, so it is
accepted again until it expires (15 minutes at most). A token never turns a deny into allow.

| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_CONSENT_TOKEN_JWKS_URL` | `<SERVER_URL>/.well-known/jwks.json` | ScrambleID's consent token keys. Fetched at start, and again when a token names an unknown key (at most every 30 s). |
| `EDGE_PROXY_CONSENT_TOKEN_ISSUER` | `https://consent.scrambleid.com` | `iss` a token must carry. |
| `EDGE_PROXY_CONSENT_TOKEN_AUDIENCE` | `https://gateway.scrambleid.com` | `aud` a token must carry. |
| `EDGE_PROXY_CONSENT_TOKEN_MAX_LIFETIME_SECONDS` | `900` | A token that lives longer is refused. |
| `EDGE_PROXY_CONSENT_TOKEN_CLOCK_TOLERANCE_SECONDS` | `30` | |

On `POST /decisions` the answer has ScrambleID's shape: `allow` with
`consent: { requestId, accepted: true }`; `consent: { accepted: false, reason: "not_needed" }` beside
an allow or deny that stands; or `403` with `CONSENT_TOKEN_INVALID`, `TOKEN_EXPIRED`,
`CONSENT_MISMATCH`, `CONSENT_PARAM_MISMATCH` or `CONSENT_NOT_APPROVED`. `503 CONSENT_KEYS_UNAVAILABLE`
means the keys could not be fetched yet. A `consentRequestId` without a token is
`400 CONSENT_NOT_SUPPORTED`: only ScrambleID has the records to check it.

### gateway adapters (`decisions`, optional)
`POST /decisions` is always on. For GCP Agent Gateway, the `ext_proc` adapter lets the gateway hand
the agent's request to the proxy instead: it reads the MCP `tools/call`, decides it from the same
copy of the policies, and answers in the gateway's protocol. Setup:
[customer-gcp-agent-gateway.md](customer-gcp-agent-gateway.md).

| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_ADAPTERS` | none | `ext_proc`: GCP Agent Gateway. |
| `EDGE_PROXY_ADAPTER_GRPC_HOST` / `..._GRPC_PORT` | `EDGE_PROXY_HOST` / `9443` | Where `ext_proc` listens (gRPC, HTTP/2). |
| `EDGE_PROXY_ADAPTER_TLS_CERT_FILE` / `..._TLS_KEY_FILE` | | PEM files, both or neither: TLS on the gRPC port. GCP Agent Gateway requires it. Without them it is plain text, reported at start. |
| `EDGE_PROXY_ADAPTER_TLS_CLIENT_CA_FILE` | | With it, the gateway must present a certificate signed by this CA (mTLS). |
| `EDGE_PROXY_ADAPTER_CONSENT_HEADER` | `x-consent-token` | Request header with the consent token. `params._meta.consentToken` in the tool call also works. |
| `EDGE_PROXY_ADAPTER_MAX_BODY_BYTES` | `262144` | A larger JSON POST body is refused (`413`). |

The gRPC port has no caller token: let only your gateway reach it. The certificate, key and CA are
files, like every other secret.

**What is decided.** Every `tools/call` in a POST the gateway sends is decided:

- **Service** = the URL the agent called, as the policy page has it: `https://mcp.acme.com/mcp/pay`.
  Host in lower case, no default port, no query, no trailing `/`. Put that URL in the policy's
  Service field (it is saved the same way).
- **Action** = the tool name. A name with characters outside `A-Z a-z 0-9 _ . : -` gets a short
  hash added, as in Agent Gateway.
- No policy for the service or tool, and no org default: deny. Set the org default or the service
  level to `allow` for an MCP server you want open.
- `initialize`, `tools/list`, notifications and `GET` pass.
- Send only your MCP routes to the edge proxy: attach the Agent Gateway extension to the MCP route
  only. A large non-MCP JSON POST sent here is refused (`413`).
- The URL is what the gateway sees before it rewrites host or path. Do not rewrite before the check.

**What the agent gets back.** Allow forwards the request unchanged. Otherwise the gateway answers the
agent itself and the MCP server is never called:

| Case | HTTP | JSON-RPC `error.code` | `error.data.decision` |
|---|---|---|---|
| Consent needed (no token, or a token that does not fit) | 403 | `-32001` | `consent_required` |
| Denied by policy, or a tool call that cannot be read | 403 (400, 411, 413 for bad bodies) | `-32002` | `deny` |
| Policies not fresh, or ScrambleID not reached yet | 503 | `-32002` | `deny` |

`error.data` also carries `reason`, `service`, `action`, `level` and, for consent, `approvers`.
Branch on `error.data.decision`, not on the code: other MCP servers use `-32001` and `-32002` too.

### freshness
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_MAX_STALENESS_SECONDS` | `30` | No heartbeat for this long = stale. |
| `EDGE_PROXY_MAX_APPLY_LAG_SECONDS` | `5` | Behind ScrambleID for this long = stale. |
| `EDGE_PROXY_MAX_CLOCK_SKEW_MS` | `2000` | Host clock may differ this much before decisions stop. |
| `EDGE_PROXY_FIRST_LOAD_WAIT_MS` | `1500` | A call right after start waits this long for the first copy. `0` = `503` at once. |

### callbacks
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_CALLBACK_ALLOWED_HOSTS` | required | Exact host, or `.suffix` for subdomains. No wildcard. |
| `EDGE_PROXY_CALLBACK_ALLOWED_CIDRS` | none | The receiver's address must be in one. Empty = no range limit. |
| `EDGE_PROXY_CALLBACK_REQUIRE_HTTPS` | `true` | `false` allows http and is reported at start. |
| `EDGE_PROXY_CALLBACK_ALLOW_LOOPBACK` | `false` | Testing only. Metadata and link-local stay blocked. |
| `EDGE_PROXY_CALLBACK_TIMEOUT_MS` | `10000` | Per attempt. |
| `EDGE_PROXY_CALLBACK_RETRY_BUDGET_SECONDS` | `120` | Then it gives up; ScrambleID resends later. |
| `EDGE_PROXY_CALLBACK_CONCURRENCY` | `8` | Deliveries at once. |
| `EDGE_PROXY_CALLBACK_QUEUE_SIZE` | `1000` | Waiting outcomes. Beyond it they wait at ScrambleID. |
| `EDGE_PROXY_CALLBACK_ACK` | `true` | Acknowledge delivered outcomes. Leave on. |

Each delivery is a `POST` of the outcome as JSON, with `x-event-id` and `x-callback-id`. `2xx` is
delivered. `408`, `429`, `5xx` and network errors are retried with a growing wait (and
`Retry-After`). Any other answer, a URL that fails the checks, or a blocked address is final.

### callback authentication
Off by default (`none`). Its own credential, never the ScrambleID one.

| Variable | Notes |
|---|---|
| `EDGE_PROXY_CALLBACK_AUTH_MECHANISM` | `none`, `privateKeyJwt` or `clientSecret`. |
| `EDGE_PROXY_CALLBACK_AUTH_ORIGIN` | Required with a mechanism. `scheme://host[:port]`, https, one of the allowed hosts. Only this origin gets the token. |
| `EDGE_PROXY_CALLBACK_AUTH_TOKEN_URL` or `..._DISCOVERY_URL` | One is required. The token URL wins if both are set. |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_ID` | Required. |
| `EDGE_PROXY_CALLBACK_AUTH_SCOPE`, `..._AUDIENCE` | Optional token request fields. |
| `EDGE_PROXY_CALLBACK_AUTH_PRIVATE_KEY_FILE` | `privateKeyJwt`. Also `..._PASSPHRASE_FILE`, `..._SIGNING_ALG`, `..._KEY_ID`, `..._ISSUER`. |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_SECRET_FILE` | `clientSecret`. File holding only the secret. |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_AUTH_METHOD` | `basic` (default) or `post`. |

### operations
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_LOG_LEVEL` | `info` | |
| `EDGE_PROXY_STREAM_IDLE_SECONDS` | `15` | No byte for this long = reconnect. |
| `EDGE_PROXY_DRAIN_SECONDS` | `15` | How long a stop waits for deliveries in flight. |

### corporate network
| Variable | Notes |
|---|---|
| `HTTPS_PROXY`, `NO_PROXY`, `NODE_USE_ENV_PROXY=1` | Egress proxy for the calls to ScrambleID and your token issuer. Callbacks go straight to your receivers. |
| `NODE_EXTRA_CA_CERTS` | Path (inside the container) to your CA file, if your network re-signs TLS. |

### health and metrics
| Path | Notes |
|---|---|
| `/health/live` | Process is up. Never depends on ScrambleID. |
| `/health/startup` | 200 once there is something to serve: a copy of the policies (`decisions`) or a first connection (`callbacks`). |
| `/health/ready` | 200 when it would do its job. 503 while draining. |
| `/health/status` | Org, revision, state, last error, queue, counts. With `decisions`: `consentTokens` (keys loaded, kids, last fetch). With adapters: `adapters` (which are on, gRPC port and TLS). |
| `/metrics` | Prometheus text. No ids, tenants or URLs. With adapters: `adapter_calls_allowed_total`, `adapter_calls_denied_total`, `adapter_calls_consent_required_total`, `adapter_requests_passed_total`, `adapter_consent_tokens_accepted_total`, `adapter_consent_tokens_refused_total`, `adapter_decision_ms_*`. |

Each tool call an adapter decides is logged as one `adapter decision` line (tool, service, action,
outcome, reason, time), never with its parameters or the token.

## Example
See `.env.example` and `docker-compose.yml` in the deployment templates. Examples only — do not
hard-code customer-specific values into the image.
