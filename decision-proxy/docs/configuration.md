# Configuration

Environment variables only, in `.env` (the compose file loads it). Validated at startup.
**A wrong or missing setting prevents startup**, and the log lists every problem at once. The
private key is a mounted file and never placed in `.env`, an environment variable or the logs.

URLs must be `https` (`http` only on `localhost`) with no credentials in them.

## Groups

### container
Read by `docker-compose.yml`, not by the proxy.

| Variable | Required | Default | Notes |
|---|---|---|---|
| `DECISION_PROXY_IMAGE` | yes | — | The pinned image (tag or `@sha256` digest). |
| `DECISION_PROXY_BIND_IP` | yes | — | Internal IP your gateway reaches. Not `0.0.0.0`. |
| `DECISION_PROXY_HOST_PORT` | | `8080` | Host port. |

### server
Pinned in `docker-compose.yml`; leave them there.

| Variable | Default |
|---|---|
| `DECISION_PROXY_HOST` | `0.0.0.0` |
| `DECISION_PROXY_PORT` | `8080` (`0..65535`) |

### sync
How the proxy reaches ScrambleID and signs in.

| Variable | Required | Notes |
|---|---|---|
| `DECISION_PROXY_SERVER_URL` | yes | `https://consent.scrambleid.com`. No org path: the org is the `orgCode` on the M2M token. |
| `DECISION_PROXY_TOKEN_URL` | yes | Where the proxy gets its own `policy.sync` token, e.g. `https://<env>.scrambleid.com/oidc/<org>/token`. |
| `DECISION_PROXY_CLIENT_ID` | yes | The M2M service account whose public key is registered at the issuer. |
| `DECISION_PROXY_TOKEN_ISSUER` | | The issuer, as the audience of the signed assertion. Default: `DECISION_PROXY_TOKEN_URL`. |
| `DECISION_PROXY_SIGNING_ALG` | | `RS256` (default) or `PS256`. |
| `DECISION_PROXY_KEY_ID` | | The `kid` of the registered public key, if it has one. |

Key files (paths inside the container; never the key itself):

| Variable | Required | Notes |
|---|---|---|
| `DECISION_PROXY_PRIVATE_KEY_FILE` | yes | `/run/policy-decision-proxy-secrets/private.pem`, matching the mount. RSA, 2048 bits or more, PEM (PKCS8 or PKCS1). |
| `DECISION_PROXY_PRIVATE_KEY_PASSPHRASE_FILE` | | Passphrase file, for an encrypted key. Mount it too. |

### gateway tokens
How the proxy checks the token your gateway sends with each decision.

| Variable | Required | Notes |
|---|---|---|
| `DECISION_PROXY_JWKS_URL` | yes | The key set that signs your gateway's `policy.evaluate` tokens. |
| `DECISION_PROXY_ISSUERS` | | Comma-separated. Empty = any issuer of that key set. |
| `DECISION_PROXY_AUDIENCE` | | Required audience, if your gateway's tokens have one. |

### freshness
When the proxy stops deciding (`503`) because its copy may be out of date.

| Variable | Default | Range | Notes |
|---|---|---|---|
| `DECISION_PROXY_MAX_STALENESS_SECONDS` | `30` | `1..3600` | No word from ScrambleID for this long = stale. |
| `DECISION_PROXY_MAX_APPLY_LAG_SECONDS` | `5` | `1..3600` | Behind a change for this long = stale. |
| `DECISION_PROXY_MAX_CLOCK_SKEW_MS` | `2000` | `1..3600000` | Host clock off ScrambleID's by more than this = stale. |
| `DECISION_PROXY_STREAM_IDLE_SECONDS` | `15` | `1..3600` | A silent stream is reopened after this. |
| `DECISION_PROXY_FIRST_LOAD_WAIT_MS` | `1500` | `0..10000` | A call right after a start waits this long for the first copy. `0` = answer `503` at once. Keep it under your gateway's target timeout. |

### operations
| Variable | Default |
|---|---|
| `DECISION_PROXY_LOG_LEVEL` | `info` |

### corporate network
Standard Node variables, all optional.

| Variable | Notes |
|---|---|
| `NODE_EXTRA_CA_CERTS` | Your CA, for TLS inspection: `/run/policy-decision-proxy-secrets/corp-ca.pem`. Uncomment its mount in `docker-compose.yml`. |
| `NODE_USE_ENV_PROXY` | Set to `1` to use `HTTPS_PROXY`. Without it, Node ignores `HTTPS_PROXY`. |
| `HTTPS_PROXY` | Your forward proxy, e.g. `http://proxy.internal:3128`. |
| `NO_PROXY` | Hosts that bypass it, e.g. `localhost,127.0.0.1`. |

## Example
See `.env.example` in the deployment templates. Example only — do not hard-code
customer-specific values into the image.
