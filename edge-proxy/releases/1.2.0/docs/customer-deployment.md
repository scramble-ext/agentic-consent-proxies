# ScrambleID Edge Proxy — Customer Deployment Runbook

**Audience:** the team deploying and operating the proxy inside your own network.
**Companion docs:** `customer-overview.md` (what it is and why), `configuration.md` (every
setting), `customer-troubleshooting.md` (when something is wrong). Deployment templates are in
the `deploy/` folder beside these docs, under `edge-proxy/releases/1.2.0/` of the
`agentic-consent-proxies` repo. Use the folder of the version you run.

The proxy is a small, stateless container. It makes **one outbound connection** to the ScrambleID
consent-server. With `decisions` on it answers your gateway's decision calls. With `callbacks` on it
delivers each consent outcome to your callback endpoint. No inbound connections from the internet
are needed.

---

## 1. Prerequisites

- A Linux host you control, with **Docker** and **Docker Compose v2** (`docker compose`).
- **Outbound HTTPS** from that host to:
  - the ScrambleID consent-server (`EDGE_PROXY_SERVER_URL`),
  - the token endpoint (`EDGE_PROXY_TOKEN_URL`),
  - your gateway's JWKS URL (`decisions`),
  - your callback receiver (`callbacks`).
- The **private key** of the proxy's service account (or its client secret). You hold it. It never
  passes through ScrambleID.
- Decide the features (`decisions`, `callbacks`, or both) and whether they share one service
  account or each gets its own (§12).

> Only `decisions` needs an inbound port, and only from your gateway.

---

## 2. Who provides what

| Provided by ScrambleID | Provided by you (customer) |
|---|---|
| Container image (tarball on the release page) + its checksum | The host, Docker, and firewall rules |
| `EDGE_PROXY_SERVER_URL` | The proxy's service account, and its private key (`secrets/private.pem`) |
| `EDGE_PROXY_TOKEN_URL`, `EDGE_PROXY_TOKEN_ISSUER` | Your gateway's JWKS URL (`decisions`) |
| The pinned image version to run | Your callback allow list (`callbacks`) |
| | Running and operating the container |

### Create the proxy's service account

Do this first. The proxy cannot start without a client id.

1. Create a service account for the proxy through the ScrambleID M2M flow, with only the scope of
   each feature you turn on:

   | Feature | Scope |
   |---|---|
   | `decisions` | `policy.sync` |
   | `callbacks` | `consent.outcome.read` |

2. Register its **public** key (§5), or a client secret for `oidc_client_creds`.
3. Use its client id as `EDGE_PROXY_CLIENT_ID`.

For `callbacks`, an agent sends its outcomes to this proxy by naming it in its token's `proxyId`
claim, set to the service account's client id. A request that names no proxy goes to every proxy
connected for your org. See "Outcomes to your Proxy" in the
[consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

---

## 3. Get the image

The image is a file, not a registry pull. Use the **exact version** ScrambleID gives you, never `latest`.

1. Open the release page of your version:
   `https://github.com/scramble-ext/agentic-consent-proxies/releases/tag/edge-proxy-v1.2.0`
2. Download `scrambleid-edge-proxy-1.2.0.tar.gz` and `SHA256SUMS`. Put them in one directory.
3. Check and load:
   ```bash
   sha256sum -c SHA256SUMS --ignore-missing                # must print: scrambleid-edge-proxy-1.2.0.tar.gz: OK
   docker load -i scrambleid-edge-proxy-1.2.0.tar.gz   # prints: Loaded image: scrambleid-edge-proxy:1.2.0
   ```

`EDGE_PROXY_IMAGE` in `.env` must be that loaded image name. A missing image stops `docker compose up`
with "image not found": load it first.

---

## 4. Lay out the stack directory

Create a directory (e.g. `/opt/edge-proxy`) and copy the templates from `edge-proxy/releases/1.2.0/deploy/`:

```
/opt/edge-proxy/
├── docker-compose.yml          # from edge-proxy/releases/1.2.0/deploy/docker-compose.yml
├── .env                        # from edge-proxy/releases/1.2.0/deploy/.env.example (fill in)
└── secrets/
    └── private.pem             # YOUR private key
```

```bash
mkdir -p /opt/edge-proxy/secrets
VERSION=1.2.0   # the version you run
cp edge-proxy/releases/$VERSION/deploy/docker-compose.yml  /opt/edge-proxy/docker-compose.yml
cp edge-proxy/releases/$VERSION/deploy/.env.example        /opt/edge-proxy/.env
```

If ScrambleID sent you a delivery bundle, use its `docker-compose.yml` and `.env` instead. They are
already filled in for your organization.

---

## 5. Make the private key (on the host)

Set `EDGE_PROXY_AUTH_MECHANISM` to how the proxy proves itself to your issuer:
`oidc_pvt_key_jwt` (a key pair, below) or `oidc_client_creds` (a client secret).

For `oidc_pvt_key_jwt`, create the key pair on the host and lock the private half to the
container's user (**uid 1000**, the `node` user):

```bash
cd /opt/edge-proxy
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out secrets/private.pem
openssl rsa -in secrets/private.pem -pubout -out secrets/public.pem
sudo chown 1000:1000 secrets/private.pem
sudo chmod 0400      secrets/private.pem
chmod 700 secrets
```

RSA 2048 bits or more, PKCS8. Register **only `public.pem`** on the service account. The private key
never leaves the host, is never in the image and is never logged. It is mounted read-only at
`/run/edge-proxy-secrets/private.pem`.

> Encrypted PEM? Also mount a passphrase file and set `EDGE_PROXY_PRIVATE_KEY_PASSPHRASE_FILE`.

For `oidc_client_creds`, put the secret alone in `secrets/client-secret` (owner 1000, `chmod 0400`),
mount it in `docker-compose.yml` instead of `private.pem`, and set `EDGE_PROXY_CLIENT_SECRET_FILE`.

---

## 6. Configure

### `.env` (non-secret)
Always set:
- `EDGE_PROXY_IMAGE` — the pinned image from §3.
- `EDGE_PROXY_FEATURES` — `decisions`, `callbacks` or `decisions,callbacks`.
- `EDGE_PROXY_SERVER_URL`, `EDGE_PROXY_TOKEN_URL`, `EDGE_PROXY_CLIENT_ID`,
  `EDGE_PROXY_AUTH_MECHANISM`.
- `EDGE_PROXY_BIND_IP` — the internal IP your gateway reaches (`decisions`), or `127.0.0.1`.

Per feature:
- **decisions:** `EDGE_PROXY_JWKS_URL`, the key set that signs your gateway's tokens.
- **decisions, with GCP Agent Gateway** (optional, §11): `EDGE_PROXY_ADAPTERS=ext_proc`, the TLS
  files (`EDGE_PROXY_ADAPTER_TLS_CERT_FILE`, `..._KEY_FILE`, mounted) and the gRPC port mapping in
  `docker-compose.yml`.
- **callbacks:** `EDGE_PROXY_CALLBACK_ALLOWED_HOSTS`, your receiver's host. Add
  `EDGE_PROXY_CALLBACK_ALLOWED_CIDRS` to pin it to your network's ranges (the proxy checks the
  **resolved address**). `EDGE_PROXY_CALLBACK_REQUIRE_HTTPS` stays `true`.

Every setting is in `configuration.md`. A missing or wrong one **stops the container at start** and
the log names every problem at once.

> **Callback authentication is optional.** By default deliveries carry **no** `Authorization`
> header, so secure your receiver another way, such as network controls or mTLS. To authenticate
> them, see **§15**.

---

## 7. Run

```bash
cd /opt/edge-proxy
docker compose up -d
docker logs edge-proxy
```

The compose file pins the image via `${EDGE_PROXY_IMAGE}`, mounts your key read-only, drops all
capabilities, runs read-only and non-root, and publishes the port on `EDGE_PROXY_BIND_IP`.

---

## 8. Verify

```bash
curl -s http://<bind-ip>:8080/health/live
curl -s http://<bind-ip>:8080/health/ready     # 200 when it would do its job
curl -s http://<bind-ip>:8080/health/status    # org, revision, queue, counts
curl -s http://<bind-ip>:8080/metrics | head
docker inspect --format '{{.State.Health.Status}}' edge-proxy
```

In the logs you should see `edge proxy started`, then `event stream open`. If a feature's scope is
missing from the account the container stops and names it.

- **decisions:** ready means the copy is fresh. Send your gateway's call to
  `POST http://<bind-ip>:8080/orgs/<org>/api/v1/decisions`. The path, body and answer are the same
  as ScrambleID's, `consentToken` included. `/health/status` shows `consentTokens.loaded: true` once
  ScrambleID's consent token keys are fetched.
- **gateway adapter:** `/health/status` shows `adapters` with the adapters that are on.
  Call a tool through your gateway: an allowed one
  reaches the MCP server, one that needs consent comes back as a JSON-RPC error with
  `error.data.decision = consent_required`, and the log has an `adapter decision` line for each.
- **callbacks:** ready means it is connected. Create a consent request with a callback URL and
  cancel it. Your receiver should get the `CANCELLED` outcome within seconds.

If `/health/ready` stays `503`, see `customer-troubleshooting.md`.

---

## 9. Operate

### Logs
```bash
docker logs --tail=100 -f edge-proxy
```
Structured JSON. Tokens, secrets, policies and callback bodies are never logged.

### Upgrade
```bash
# set EDGE_PROXY_IMAGE in .env to the new pinned version, then:
cd /opt/edge-proxy
docker compose pull
docker compose up -d --force-recreate
docker image prune -f
```

### Rollback
```bash
# set EDGE_PROXY_IMAGE back to the previous version, then:
docker compose up -d --force-recreate
```

A restart costs nothing: policies are fetched again, and outcomes not acknowledged are sent again.

From the version with gateway adapters on, `POST /decisions` accepts a `consentToken` (before, it
answered `400 CONSENT_NOT_SUPPORTED`), and the proxy fetches ScrambleID's consent token keys at start.
Nothing else changes for an existing setup: no adapter is on until `EDGE_PROXY_ADAPTERS` names one.

### Rotate the private key
Register the new public key on the service account, then:
```bash
cp /path/to/new-private.pem /opt/edge-proxy/secrets/private.pem
sudo chown 1000:1000 /opt/edge-proxy/secrets/private.pem
sudo chmod 0400      /opt/edge-proxy/secrets/private.pem
docker compose up -d --force-recreate   # recreate so the new key is picked up
```

### Stop
```bash
docker compose down          # drains deliveries in flight; nothing persisted
```

---

## 10. Network / egress summary

Open **outbound** from the host:
- → ScrambleID consent-server (`EDGE_PROXY_SERVER_URL`), HTTPS. Allow long-lived streaming responses.
  Its consent token keys (`/.well-known/jwks.json`) are fetched from the same host.
- → token endpoint (`EDGE_PROXY_TOKEN_URL`), HTTPS.
- → your gateway's JWKS URL (`decisions`).
- → your callback receiver (`callbacks`).

Inbound: only your gateway, to the published port (`decisions`), and to the gRPC port with the
`ext_proc` adapter. The gRPC port has no caller token: allow only your gateway's
addresses.

### Corporate network

Behind a corporate proxy or TLS interception, set `HTTPS_PROXY`, `NO_PROXY` and
`NODE_USE_ENV_PROXY=1` in `.env`. This covers the calls to ScrambleID and your token issuer.
Callbacks go straight to your receivers.

If your network re-signs TLS, copy the CA file to `secrets/corp-ca.pem`, uncomment its mount in
`docker-compose.yml`, and set `NODE_EXTRA_CA_CERTS=/run/edge-proxy-secrets/corp-ca.pem`.

---

## 11. Pointing your gateway at it (`decisions`)

Your gateway keeps its token and its request. Only the base URL changes:
`http://<bind-ip>:<port>/orgs/{orgCode}/api/v1/decisions`.

Two things the gateway has to handle:

1. **A call that carries a consent.** A `consentToken` for exactly the call turns a `consent` answer
   into `allow`, as on ScrambleID, checked here against ScrambleID's public keys. It is not spent
   here: it is accepted again until it expires (15 minutes at most). A `consentRequestId` without a
   token is `400 CONSENT_NOT_SUPPORTED`: send the token, or send that call to ScrambleID's
   `POST /decisions`.
2. **A `503`** (`POLICY_NOT_LOADED`, `POLICY_STALE`, `POLICY_INCOMPATIBLE`, `CONSENT_KEYS_UNAVAILABLE`)
   or no answer at all.

### Fallback to ScrambleID (recommended)

In Apigee, a TargetServer load balancer:

- each edge proxy container as a TargetServer;
- `https://consent.scrambleid.com` as an `IsFallback` server;
- a HealthMonitor on `/health/ready`, so a stale proxy leaves rotation;
- `ServerUnhealthyResponse` with `503`, so a call that hits a proxy between health checks retries
  on the next server instead of failing.

The token and request are the same on both, and the answers are equal. The proxy never forwards a
call itself. Without a fallback, every decision fails while no proxy is ready. Treat it as `deny`.
One difference with a consent token: ScrambleID spends it the first time it accepts it, so a later
call with the same token that reaches the fallback is refused (`CONSENT_USED`); the proxy accepts it
again until it expires.

Run **2 containers on separate hosts** so one can restart while the other serves. With one
container, every restart sends traffic to the fallback.

### Checking the fallback by hand

In staging:

1. Send a decision through the gateway. `/health/status` `counters.decisions` goes up.
2. Block the proxy's access to ScrambleID for longer than `EDGE_PROXY_MAX_STALENESS_SECONDS`.
   `/health/ready` turns `503` and the HealthMonitor takes it out.
3. Send the same decision again. ScrambleID answers it, with the same body.
4. Unblock. Within seconds `/health/ready` is `200` and the proxy is back in rotation.

### GCP Agent Gateway

GCP Agent Gateway hands each request to an external authorization service instead of calling
`POST /decisions`. Set `EDGE_PROXY_ADAPTERS=ext_proc`, mount the TLS files and publish the gRPC port
(`9443` in the container) in `docker-compose.yml`. Steps: [customer-gcp-agent-gateway.md](customer-gcp-agent-gateway.md).

On this path there is no fallback to ScrambleID: the adapter refuses a tool call when the policies
are not fresh (`503`). Run two containers.

---

## 12. Running two edge proxies (separate accounts)

To keep the two features apart, copy the directory and give each its own `.env`, key,
`container_name` and host port. One has `EDGE_PROXY_FEATURES=decisions` and the other `callbacks`,
each with its own service account holding only its scope.

---

## 13. Moving from the older proxies

The Policy Decision Proxy and the Consent Notification Proxy keep working. To switch:

1. Run the edge proxy next to them. Settings are renamed to `EDGE_PROXY_*`, and the callback allow
   list is a comma list in `.env` instead of a JSON file.
2. For callbacks, agents must name the edge proxy's client id in `proxyId` (or reuse the old
   proxy's account for the edge proxy and change nothing in the agents).
3. Move your gateway to the edge proxy's address, watch `/health/status`, then stop the old
   container.

---

## 14. Security posture (for your review)

- **No secret in the image.** Keys are mounted read-only at run time; you own them.
- **Non-root, read-only filesystem, all capabilities dropped, `no-new-privileges`, no volume.**
- **Outbound to ScrambleID only.** The one inbound surface is the published port.
- **Policies in memory only.** Never written to disk.
- **Never decides from a copy that may be wrong.** A stale copy is a `503`, not a decision.
- **Least privilege on delivery.** Every callback is re-validated (https, allow list, CIDR, SSRF)
  before each send. Redirects are never followed. A callback bearer token, when configured, goes
  only to `EDGE_PROXY_CALLBACK_AUTH_ORIGIN`.
- **No sensitive data in logs.** Ids, states and counts only.

---

## 15. Callback authentication (`callbacks`, optional)

Controls how the proxy authenticates the delivery it POSTs to **your** receiver. Off by default
(`none`): no `Authorization` header. The proxy never reuses its ScrambleID token for this. Use a
**separate** credential from your own auth server.

| Variable | Notes |
|---|---|
| `EDGE_PROXY_CALLBACK_AUTH_MECHANISM` | `none` (default), `privateKeyJwt` or `clientSecret` |
| `EDGE_PROXY_CALLBACK_AUTH_ORIGIN` | **Required** with a mechanism. `scheme://host[:port]`, https, one of the allowed hosts. Only this origin gets the token |
| `EDGE_PROXY_CALLBACK_AUTH_TOKEN_URL` or `..._DISCOVERY_URL` | One is **required**. The token URL wins if both are set |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_ID` | **Required** |
| `EDGE_PROXY_CALLBACK_AUTH_SCOPE`, `..._AUDIENCE` | Optional token request fields |
| `EDGE_PROXY_CALLBACK_AUTH_PRIVATE_KEY_FILE` | `privateKeyJwt`. Also `..._PASSPHRASE_FILE`, `..._SIGNING_ALG`, `..._KEY_ID`, `..._ISSUER` |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_SECRET_FILE` | `clientSecret`. File holding only the secret |
| `EDGE_PROXY_CALLBACK_AUTH_CLIENT_AUTH_METHOD` | `clientSecret`: `basic` (default) or `post` |

A bad or incomplete setting **stops the container at start** and the log names it.

> **Two different keys.** The callback key only mints the token your receiver checks. It is not the
> ScrambleID service-account key (§5). Do not reuse one for the other.

### Walkthrough A — Private Key JWT

1. Create the key pair, register the public key at your auth server, and lock the private key as in
   §5 (`secrets/callback-private.pem`, uid 1000, `0400`).
2. Uncomment its mount in `docker-compose.yml`.
3. In `.env`:
   ```bash
   EDGE_PROXY_CALLBACK_AUTH_MECHANISM=privateKeyJwt
   EDGE_PROXY_CALLBACK_AUTH_ORIGIN=https://receiver.internal
   EDGE_PROXY_CALLBACK_AUTH_TOKEN_URL=https://auth.internal/oauth2/token
   EDGE_PROXY_CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   EDGE_PROXY_CALLBACK_AUTH_SCOPE=callback.write
   EDGE_PROXY_CALLBACK_AUTH_PRIVATE_KEY_FILE=/run/edge-proxy-secrets/callback-private.pem
   ```
4. `docker compose up -d --force-recreate`, then verify per §8.

### Walkthrough B — Client Secret

1. Put the secret alone in `secrets/callback-client-secret` (uid 1000, `0400`) and uncomment its
   mount.
2. In `.env`:
   ```bash
   EDGE_PROXY_CALLBACK_AUTH_MECHANISM=clientSecret
   EDGE_PROXY_CALLBACK_AUTH_ORIGIN=https://receiver.internal
   EDGE_PROXY_CALLBACK_AUTH_TOKEN_URL=https://auth.internal/oauth2/token
   EDGE_PROXY_CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   EDGE_PROXY_CALLBACK_AUTH_SCOPE=callback.write
   EDGE_PROXY_CALLBACK_AUTH_CLIENT_SECRET_FILE=/run/edge-proxy-secrets/callback-client-secret
   ```
3. `docker compose up -d --force-recreate`, then verify per §8.

### What your receiver must validate

On every delivered access token: the **signature** against your auth server's JWKS, the **issuer**
(`iss`), the **audience** (`aud`), the **expiration** (`exp`) and any required **scopes**. Both
issuer and JWKS come from your auth server's discovery document.

---

## 16. What your receiver gets

The proxy sends one `POST` to the request's `callbackUrl` for each final outcome, with
`Content-Type: application/json`. The body is the outcome exactly as the consent server sends it.
Its fields are in "Outcomes to your Proxy" in the
[consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

The proxy adds these headers:

| Header | Meaning |
|---|---|
| `x-callback-id` | The same on every delivery of one outcome. Use it to drop duplicates. |
| `x-event-id` | The outcome's number at ScrambleID. |
| `Authorization` | `Bearer <token>`, only with callback authentication (§15). |

Return a `2xx` to confirm receipt. The proxy retries `408`, `429`, `5xx` and network errors within
`EDGE_PROXY_CALLBACK_RETRY_BUDGET_SECONDS`. It treats `3xx` and other `4xx` as permanent.

---

## 17. If you need support

Collect and send (see `customer-troubleshooting.md` §"Support bundle"): a recent `docker logs`
excerpt, a `/health/status` and `/metrics` snapshot, your `.env` **with secrets removed**, and the
image version. Do **not** send a private key, a secret or any token.
