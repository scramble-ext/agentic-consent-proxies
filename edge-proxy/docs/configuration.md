# Configuration

Environment variables, set in `.env`. **Invalid or incomplete config prevents startup**, and the log
lists every problem at once. Secrets are never variables: keys and secrets are files you mount,
named by path. A setting of a feature that is off does nothing, and the proxy says so at start.

## Groups

### container
| Variable | Default | Notes |
|---|---|---|
| `EDGE_PROXY_IMAGE` | required | Pinned image, e.g. `ghcr.io/scramble-ext/scrambleid-edge-proxy:<version>`. |
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
| `/health/status` | Org, revision, state, last error, queue, counts. |
| `/metrics` | Prometheus text. No ids, tenants or URLs. |

## Example
See `.env.example` and `docker-compose.yml` in the deployment templates. Examples only — do not
hard-code customer-specific values into the image.
