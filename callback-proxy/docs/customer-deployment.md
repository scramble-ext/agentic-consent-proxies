# Consent Notification Proxy — Customer Deployment Runbook

**Audience:** the team deploying and operating the proxy inside your own network.
**Companion docs:** `customer-overview.md` (what it is and why), `customer-troubleshooting.md`
(when something is wrong). Deployment templates are in `callback-proxy/deploy/` of the
`agentic-consent-proxies` repo.

The proxy is a small, stateless container. It makes **one outbound connection** to the
ScrambleID consent-server and delivers each consent decision to your callback endpoint.
No inbound connections are required.

---

## 1. Prerequisites

- A Linux host you control, with **Docker** and **Docker Compose v2** (`docker compose`).
- **Outbound HTTPS** egress from that host to:
  - the ScrambleID consent-server (`consent.baseUrl`),
  - the OIDC token endpoint (`auth.tokenEndpoint`),
  - your own callback endpoint.
- The **private key** (PEM) of the proxy's service account (`privateKeyJwt` mode). You hold
  this. It never passes through ScrambleID.

> No inbound firewall rules are needed for the proxy's core function. Health/metrics bind to
> loopback by default.

---

## 2. Who provides what

| Provided by ScrambleID | Provided by you (customer) |
|---|---|
| Container image (registry access **or** offline tarball) + its checksum/digest | The host, Docker, and egress firewall rules |
| `consent.baseUrl`, `tenant` | The proxy's service account, and its private key (`secrets/private.pem`) |
| `auth.tokenEndpoint`, `auth.issuer` | Your callback endpoint |
| The **JWKS URL** for your receiver, only if ScrambleID issues your callback token (§13) | The callback allowlist (`allowedHosts`, `allowedCidrs`) |
| The pinned image version to run | Running/operating the container |

### Create the proxy's service account

Do this first. The proxy cannot start without a client id.

1. Create a service account for the proxy through the ScrambleID M2M flow, with the scope
   `consent.outcome.read`.
2. Use its client id as `auth.clientId`, and its private key as `secrets/private.pem` (§5).

The proxy receives every outcome in your organization. If you run more than one proxy, see
"Outcomes to your Proxy" in the [consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

---

## 3. Get the image

Pin to the **exact version** ScrambleID gives you (an immutable tag like `1.0.0`, or a
`@sha256:...` digest). Do not use `latest` in production. Verify the checksum/digest ScrambleID
publishes before running.

### Option A — pull from the registry (connected sites)
The image is public, so no login is needed.
```bash
docker pull ghcr.io/scramble-ext/consent-proxy:<version>
# Optional: verify the digest matches what ScrambleID published
docker image inspect ghcr.io/scramble-ext/consent-proxy:<version> --format '{{index .RepoDigests 0}}'
```

### Option B — offline tarball (air-gapped / locked-down networks)
```bash
# Verify the file against the checksum ScrambleID provided, then load it:
sha256sum consent-proxy-<version>.tar.gz        # compare to the published value
docker load -i consent-proxy-<version>.tar.gz   # prints the loaded image tag
```

Set the resulting image reference in `.env` as `CONSENT_PROXY_IMAGE`.

---

## 4. Lay out the stack directory

Create a directory (e.g. `/opt/consent-proxy`) and copy the templates from
`callback-proxy/deploy/` into it:

```
/opt/consent-proxy/
├── docker-compose.yml          # from callback-proxy/deploy/docker-compose.yml
├── .env                        # from callback-proxy/deploy/.env.example (fill in)
├── config/
│   └── config.json             # from callback-proxy/deploy/config.example.json (fill in)
└── secrets/
    └── private.pem             # YOUR private key
```

```bash
mkdir -p /opt/consent-proxy/config /opt/consent-proxy/secrets
cp callback-proxy/deploy/docker-compose.yml   /opt/consent-proxy/docker-compose.yml
cp callback-proxy/deploy/.env.example         /opt/consent-proxy/.env
cp callback-proxy/deploy/config.example.json  /opt/consent-proxy/config/config.json
```

If ScrambleID sent you a delivery bundle, use its `docker-compose.yml`, `.env` and
`config/config.json` instead. They are already filled in for your organization.

---

## 5. Provide the private key (bring-your-own)

Place your PEM at `secrets/private.pem`, then set ownership to the container's user
(**uid 1000**, the `node` user) and lock the permissions. You do this with your own host
privileges — no container-as-root trick needed.

```bash
cp /path/to/your/private.pem /opt/consent-proxy/secrets/private.pem
sudo chown 1000:1000 /opt/consent-proxy/secrets/private.pem
sudo chmod 0400       /opt/consent-proxy/secrets/private.pem
chmod 700 /opt/consent-proxy/secrets
```

The key is mounted **read-only** at `/run/consent-proxy-secrets/private.pem`. It is never
baked into the image and never logged.

> Encrypted PEM? Also mount a passphrase file and set `AUTH_PRIVATE_KEY_PASSPHRASE_FILE`
> in `.env`.

---

## 6. Configure

### `.env` (non-secret)
Fill in at least:
- `CONSENT_PROXY_IMAGE` — the pinned image from step 3.
- `LOG_LEVEL=info` (leave `PRINT_OUTCOMES` unset in production).
- `AUTH_PRIVATE_KEY_FILE=/run/consent-proxy-secrets/private.pem` (already set in the template).

Leave `CALLBACK_AUTH_ORIGIN` unset unless you turn on callback authentication (§13). Setting it
on its own **stops the proxy from starting**.

### `config/config.json` (non-secret)
Fill in the values ScrambleID gave you, your service account's client id, and your allowlist:
- `consent.baseUrl` (it must end in `/orgs/<org>/api/v1`), `consent.tenant`.
- `auth.tokenEndpoint`, `auth.issuer`, `auth.clientId` (keep `mode: "privateKeyJwt"`).
- `callbackPolicy.allowedHosts` — the hostname(s) of your callback endpoint.
- `callbackPolicy.allowedCidrs` — the IP range(s) your callback host resolves to. The
  proxy checks the **resolved address**; if your receiver is on a public IP, add that
  range (loopback/link-local/metadata are always blocked). `requireHttps` stays `true`.

> Using `clientCredentials` or `static` instead? See the notes in `config.example.json` and
> `.env.example` for the fields and the secret file to mount.

> **Callback authentication is optional.** By default deliveries carry **no** `Authorization`
> header, so secure your receiver another way, such as network controls or mTLS. To authenticate
> them with a **dedicated** callback token (from a ScrambleID *or* your own auth server), see
> **§13. Callback authentication**.

Invalid or incomplete config **fails startup** (by design) — the proxy will not run with a
bad configuration.

---

## 7. Run

```bash
cd /opt/consent-proxy
docker compose up -d
```

The compose file pins the image via `${CONSENT_PROXY_IMAGE}`, mounts your config and key
read-only, drops all capabilities, runs read-only + non-root, and binds health/metrics to
loopback by default.

---

## 8. Verify

```bash
# Liveness (process up)
curl -s http://127.0.0.1:8080/health/live
# Readiness — 200 with "connected" once the SSE stream is up
curl -s http://127.0.0.1:8080/health/ready
# Metrics
curl -s http://127.0.0.1:8080/metrics | head
# Container health
docker inspect --format '{{.State.Health.Status}}' consent-proxy
```

Expected: `/health/ready` returns `{"status":"ok", ... "stream": {"ok":true, "detail":"connected"}}`
and the container reports `healthy`. In the logs you should see the proxy obtain its token
and then `SSE stream connected`.

If `/health/ready` stays `503`, see `customer-troubleshooting.md` (most commonly a token/JWKS
or egress issue).

---

## 9. Operate

### Logs
```bash
docker logs --tail=100 -f consent-proxy
```
Structured JSON; tokens, secrets, and callback bodies are never logged.

### Upgrade
```bash
# set CONSENT_PROXY_IMAGE in .env to the new pinned version, then:
cd /opt/consent-proxy
docker compose pull
docker compose up -d --force-recreate
docker image prune -f
```

### Rollback
```bash
# set CONSENT_PROXY_IMAGE back to the previous version, then:
docker compose up -d --force-recreate
```

### Rotate the private key
```bash
cp /path/to/new-private.pem /opt/consent-proxy/secrets/private.pem
sudo chown 1000:1000 /opt/consent-proxy/secrets/private.pem
sudo chmod 0400       /opt/consent-proxy/secrets/private.pem
docker compose up -d --force-recreate   # recreate so the new key is picked up
```

### Stop
```bash
docker compose down          # graceful stop; nothing persisted
```

---

## 10. Network / egress summary

Open **outbound** from the host:
- → ScrambleID consent-server host (`consent.baseUrl`), HTTPS.
- → OIDC token endpoint host (`auth.tokenEndpoint`), HTTPS.
- → your callback endpoint.

No inbound exposure is required. Behind a corporate proxy or TLS interception, set
`HTTPS_PROXY` / `NO_PROXY` and `NODE_EXTRA_CA_CERTS` in `.env`.

If your receiver or forward proxy uses a certificate from an internal CA, copy the CA file to
`secrets/corp-ca.pem`, uncomment its mount in `docker-compose.yml`, and set
`NODE_EXTRA_CA_CERTS=/run/consent-proxy-secrets/corp-ca.pem` in `.env`.

---

## 11. Security posture (for your review)

- **No secret in the image.** The private key is mounted read-only at runtime; you own it.
- **Non-root, read-only filesystem, all capabilities dropped, `no-new-privileges`, no volume.**
- **Outbound-only.** The one inbound surface (health/metrics) binds to loopback by default.
- **Least privilege on delivery.** Every callback is re-validated (HTTPS, allowlist, CIDR,
  SSRF) before each send; redirects are never followed; a callback bearer token, when
  configured, is sent only to `CALLBACK_AUTH_ORIGIN`.
- **No sensitive data at rest or in logs.** Decisions carry request id + status, not user
  content or secrets.
- **Stateless.** Nothing to back up; restarts reconnect and catch up automatically.

---

## 12. If you need support

Collect and send (see `customer-troubleshooting.md` §"Support bundle"): a recent
`docker logs` excerpt, a `/metrics` snapshot, your `config.json` and `.env` **with secrets
removed**, and the image version. Do **not** send the private key or any token.

---

## 13. Callback authentication

This section is **self-contained** and **optional**. It controls how the proxy
authenticates the delivery it POSTs to **your** receiver — specifically, how it obtains the
`Authorization: Bearer <token>` it attaches (only) to `CALLBACK_AUTH_ORIGIN`.

### 13.0 Default behavior (no callback authentication)
If you configure **nothing** in this section (no `callback.auth` block and no
`CALLBACK_AUTH_*` mechanism var, or `CALLBACK_AUTH_MECHANISM=none`), the proxy sends **no
`Authorization` header** on callbacks. It does **not** reuse its upstream ScrambleID token:
that token authenticates the proxy to the ScrambleID consent-server, and reusing it on a
delivery to **your** receiver would present the wrong audience, grant excessive scope, and
create an unintended ScrambleID↔receiver trust. To authenticate deliveries you **must**
configure a **dedicated** credential — a mechanism (`privateKeyJwt` or `clientCredentials`)
issued by a ScrambleID auth server **or your own auth server**.

If you set `CALLBACK_AUTH_ORIGIN` **without** a mechanism, the proxy **fails startup** with:

```
CALLBACK_AUTH_ORIGIN is set but callback.auth.mechanism is none: the proxy will not reuse
the upstream ScrambleID token for callbacks. Configure a dedicated callback credential
(CALLBACK_AUTH_MECHANISM=privateKeyJwt or clientCredentials) or unset CALLBACK_AUTH_ORIGIN
to send unauthenticated callbacks.
```

### 13.1 Environment variables

| Var | Mechanism | Required / Optional | Default |
|---|---|---|---|
| `CALLBACK_AUTH_MECHANISM` | — | optional | `none` |
| `CALLBACK_AUTH_TOKEN_ENDPOINT` | privateKeyJwt, clientCredentials | **required** (dedicated) unless `CALLBACK_AUTH_DISCOVERY_URL` is set; **wins** when both set | — |
| `CALLBACK_AUTH_DISCOVERY_URL` | privateKeyJwt, clientCredentials | optional (alternative to token endpoint) | — |
| `CALLBACK_AUTH_JWKS_URI` | privateKeyJwt, clientCredentials | optional (informational/override only) | — |
| `CALLBACK_AUTH_CLIENT_ID` | privateKeyJwt, clientCredentials | **required** (dedicated) | — |
| `CALLBACK_AUTH_ISSUER` | privateKeyJwt | optional | the expected authorization-server issuer; when omitted, the discovered `issuer` (if discovery is configured) is used as the assertion `aud`, otherwise the resolved token endpoint is used as the assertion `aud` |
| `CALLBACK_AUTH_SCOPE` | privateKeyJwt, clientCredentials | optional | — |
| `CALLBACK_AUTH_AUDIENCE` | privateKeyJwt, clientCredentials | optional | — |
| `CALLBACK_AUTH_SIGNING_ALG` | privateKeyJwt | optional | `RS256` (or `PS256`) |
| `CALLBACK_AUTH_KID` | privateKeyJwt | optional | — |
| `CALLBACK_AUTH_ASSERTION_TTL_SECONDS` | privateKeyJwt | optional | `60` (max `3600`) |
| `CALLBACK_AUTH_CLIENT_AUTH_METHOD` | clientCredentials | optional | `client_secret_basic` (or `client_secret_post`) |
| `CALLBACK_AUTH_PRIVATE_KEY_FILE` | privateKeyJwt | **required** (dedicated) | — |
| `CALLBACK_AUTH_PRIVATE_KEY_PASSPHRASE_FILE` | privateKeyJwt | optional (encrypted PEM only) | — |
| `CALLBACK_AUTH_CLIENT_SECRET_FILE` | clientCredentials | **required** (dedicated) | — |

Non-secret fields may instead live in `config.json` under `callback.auth.*`; the env vars
override them. The secret `*_FILE` vars are env-only (same model as the upstream key). The
proxy **fails startup** if a mechanism is set without its client id, without either a token
endpoint or a discovery URL, or without its secret — **and** if `CALLBACK_AUTH_ORIGIN` is set
while the mechanism is `none`/absent (an origin without a mechanism can never attach a bearer,
since the upstream token is never reused — see §13.0 and §13.5). Conversely, a mechanism set
**without** `CALLBACK_AUTH_ORIGIN` loads fine but mints a token that is **never attached**
(the origin gate has nothing to match — see §13.5).

### 13.1a Discovery vs explicit endpoint

Both dedicated mechanisms accept **either** an explicit `CALLBACK_AUTH_TOKEN_ENDPOINT`
**or** a `CALLBACK_AUTH_DISCOVERY_URL`. Discovery lets the proxy read the token endpoint from
your auth server's OIDC discovery document (`.../.well-known/openid-configuration`) instead
of you configuring it by hand.

- **Discovery applies to both mechanisms** (privateKeyJwt and clientCredentials).
- **Explicit endpoint overrides discovery.** If both are set, the proxy uses
  `CALLBACK_AUTH_TOKEN_ENDPOINT` and never fetches the document.
- **HTTPS-only, SSRF-guarded, cached.** The discovery URL must be `https`; its host is
  resolved and checked against the same SSRF guard as callback delivery (metadata/loopback/
  link-local are blocked); the document is fetched **once** and cached for the process
  lifetime (not per delivery).
- **Optional metadata is not required.** Only `token_endpoint` must be present in the
  document. If the document publishes `issuer` and you also set `CALLBACK_AUTH_ISSUER`, they
  must match exactly. For `privateKeyJwt`, when you do not set `CALLBACK_AUTH_ISSUER` the
  assertion `aud` is taken from the discovered `issuer` (falling back to the token endpoint).
- `CALLBACK_AUTH_JWKS_URI` is accepted for documentation/override but the proxy does not use
  it to validate tokens — your receiver does.

### 13.1b Grant vs client-authentication method

It helps to keep two OAuth2 concepts separate, because they are **orthogonal** — not three
competing grants:

- The **grant** is *how* the client asks for a token. Both dedicated mechanisms here use the
  **`client_credentials` grant** (a machine-to-machine, no-user flow).
- The **client-authentication method** is *how* the client proves who it is to the token
  endpoint. `private_key_jwt` (a signed client assertion, RFC 7523 §2.2) is one such method;
  `client_secret_basic` and `client_secret_post` (a shared secret) are others.

So `private_key_jwt` is a client-authentication method **layered on** the `client_credentials`
grant, not a separate grant. The compatible combinations are:

- `client_credentials` + `client_secret_basic`
- `client_credentials` + `client_secret_post`
- `client_credentials` + `private_key_jwt`

In **this proxy** those combinations map onto `CALLBACK_AUTH_MECHANISM` as follows:

- `CALLBACK_AUTH_MECHANISM=privateKeyJwt` **is** `client_credentials` authenticated with
  `private_key_jwt`. It is the single way to do `private_key_jwt`, and it conditionally
  requires a **private key** (`CALLBACK_AUTH_PRIVATE_KEY_FILE`), not a client secret.
- `CALLBACK_AUTH_MECHANISM=clientCredentials` **is** `client_credentials` authenticated with
  a **shared secret**, where `CALLBACK_AUTH_CLIENT_AUTH_METHOD` selects
  `client_secret_basic` (default) or `client_secret_post`.

Note that `CALLBACK_AUTH_CLIENT_AUTH_METHOD` only takes `client_secret_basic` or
`client_secret_post` — `private_key_jwt` is **not** a value of that variable; you select it by
setting `CALLBACK_AUTH_MECHANISM=privateKeyJwt`.

### 13.1c Three distinct "issuer" concepts

When reading this section, keep three separate notions of "issuer" apart — they are **not**
interchangeable:

- **(a) The authorization-server issuer** — the `iss` published in that auth server's OIDC
  discovery document, i.e. the issuer your receiver trusts. This is what `CALLBACK_AUTH_ISSUER`
  denotes. When discovery is configured the proxy populates it from the discovered `issuer`.
- **(b) The JWT client-assertion `iss` claim** (`private_key_jwt` only) — this equals your
  `CALLBACK_AUTH_CLIENT_ID` (the proxy sets the assertion `iss` and `sub` to the client id).
  It exists **only** to authenticate the proxy to the token endpoint; it is unrelated to (a).
- **(c) The access-token issuer the receiver validates** — the `iss` your receiver checks on
  each delivered access token. It is the authorization-server issuer of whichever server
  minted that token (so it aligns with (a) when that same server both issues the token and is
  named by `CALLBACK_AUTH_ISSUER`).

**On the `CALLBACK_AUTH_ISSUER` default.** When `CALLBACK_AUTH_ISSUER` is omitted, the proxy
uses the discovered `issuer` if discovery is configured; otherwise it falls back to the
resolved **token endpoint** value — but **only** as the client-assertion `aud` (concept (a)
used as the assertion audience), which is a pragmatic fallback many authorization servers
accept. This fallback is about the assertion's audience; it is **not** a claim that the
authorization-server issuer equals the token endpoint URL.

### 13.2 Mount the callback secret(s)

Mount the secret file(s) read-only and lock them to the container user (**uid 1000**,
`node`), exactly as in **§5**:

```bash
# privateKeyJwt: your dedicated callback signing key
cp /path/to/callback-private.pem /opt/consent-proxy/secrets/callback-private.pem
sudo chown 1000:1000 /opt/consent-proxy/secrets/callback-private.pem
sudo chmod 0400       /opt/consent-proxy/secrets/callback-private.pem

# clientCredentials: your dedicated callback client secret (single line, no trailing newline)
printf '%s' 'YOUR_CALLBACK_CLIENT_SECRET' > /opt/consent-proxy/secrets/callback-client-secret
sudo chown 1000:1000 /opt/consent-proxy/secrets/callback-client-secret
sudo chmod 0400       /opt/consent-proxy/secrets/callback-client-secret

# optional: passphrase for an encrypted callback PEM
printf '%s' 'YOUR_PEM_PASSPHRASE' > /opt/consent-proxy/secrets/callback-passphrase
sudo chown 1000:1000 /opt/consent-proxy/secrets/callback-passphrase
sudo chmod 0400       /opt/consent-proxy/secrets/callback-passphrase
```

Then uncomment the matching read-only mount(s) in `docker-compose.yml`:

```yaml
    volumes:
      - ./config:/etc/consent-proxy/config:ro
      - ./secrets/private.pem:/run/consent-proxy-secrets/private.pem:ro
      # privateKeyJwt:
      - ./secrets/callback-private.pem:/run/consent-proxy-secrets/callback-private.pem:ro
      # - ./secrets/callback-passphrase:/run/consent-proxy-secrets/callback-passphrase:ro
      # clientCredentials:
      # - ./secrets/callback-client-secret:/run/consent-proxy-secrets/callback-client-secret:ro
```

### 13.2a Service-account registration (privateKeyJwt)

When you choose `CALLBACK_AUTH_MECHANISM=privateKeyJwt`, the proxy authenticates to a token
endpoint with a signed assertion to mint the callback token. That requires a client (a
"service account") registered at the authorization server that will **issue** the callback
token. Work through the following with whoever owns that authorization server.

- **Who creates the client.** The client is registered at the authorization server that
  **issues** the callback token. That can be **your own auth server** (when you issue the
  token) **or** ScrambleID (when a ScrambleID auth server issues it) — this is configurable,
  because `CALLBACK_AUTH_TOKEN_ENDPOINT` / `CALLBACK_AUTH_DISCOVERY_URL` may point at either.
  The dedicated callback credential is independent of the upstream credential, so it may live
  at a different authorization server.
- **Registering the public key.** You generate the callback key pair and register the
  **public** key (as a JWK or PEM) with that authorization server for the callback client.
  The **private** key is mounted to the proxy **only** (read-only, locked to uid 1000, as in
  §5 and §13.2) and never leaves your environment.
- **Client id mapping.** `CALLBACK_AUTH_CLIENT_ID` must equal the client/service-account
  identifier registered at that authorization server. The proxy also uses it as the
  assertion `iss` and `sub` (they are the same value).
- **`kid` mapping.** Set `CALLBACK_AUTH_KID` to the `kid` of the registered public JWK so the
  authorization server selects the right key to verify the assertion. Omit it if the
  authorization server does not key on `kid`.
- **Scopes / permissions.** Grant the callback client whatever scope your receiver expects
  (set it via `CALLBACK_AUTH_SCOPE`) plus any audience or permission the authorization server
  requires in order to issue the token.
- **Key-rotation coordination.** Register the **new** public key at the authorization server
  first (ideally under a new `kid`), then swap the mounted private key and
  `CALLBACK_AUTH_KID` on the proxy and recreate the container — the same mechanics as the
  private-key rotation steps in §9. Keep the old and new public keys registered
  **simultaneously** at the authorization server during the cutover so in-flight assertions
  signed with either key still verify, avoiding downtime.

> **Two different keys — do not confuse them.** The **dedicated callback-authentication key**
> described here (the `privateKeyJwt` callback credential) is used **only** to mint the token
> the proxy attaches to deliveries to your receiver. It is a different credential from the
> **upstream service-account key** (`secrets/private.pem` / `AUTH_PRIVATE_KEY_FILE`, §5),
> which authenticates the proxy's **SSE connection to the ScrambleID consent-server**. The
> two keys serve different purposes and are registered at (potentially) different
> authorization servers. **Do not reuse one for the other.**

### 13.3 Walkthrough A — Private Key JWT

1. Place and lock the signing key (§13.2) and uncomment its mount.
2. In `.env`:
   ```bash
   CALLBACK_AUTH_MECHANISM=privateKeyJwt
   CALLBACK_AUTH_TOKEN_ENDPOINT=https://auth.example.com/oauth2/token
   CALLBACK_AUTH_ISSUER=https://auth.example.com
   CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   CALLBACK_AUTH_SCOPE=callback.write
   CALLBACK_AUTH_PRIVATE_KEY_FILE=/run/consent-proxy-secrets/callback-private.pem
   # CALLBACK_AUTH_KID=key-1
   # CALLBACK_AUTH_SIGNING_ALG=RS256
   # CALLBACK_AUTH_ASSERTION_TTL_SECONDS=60
   # CALLBACK_AUTH_PRIVATE_KEY_PASSPHRASE_FILE=/run/consent-proxy-secrets/callback-passphrase
   ```
   **Discovery variant** — replace `CALLBACK_AUTH_TOKEN_ENDPOINT` (and, if you like,
   `CALLBACK_AUTH_ISSUER`) with the discovery URL; the token endpoint (and the assertion
   `aud`, when `CALLBACK_AUTH_ISSUER` is unset) come from the document:
   ```bash
   CALLBACK_AUTH_MECHANISM=privateKeyJwt
   CALLBACK_AUTH_DISCOVERY_URL=https://auth.example.com/.well-known/openid-configuration
   CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   CALLBACK_AUTH_SCOPE=callback.write
   CALLBACK_AUTH_PRIVATE_KEY_FILE=/run/consent-proxy-secrets/callback-private.pem
   ```
3. `docker compose up -d --force-recreate`, then verify per §8. On the first delivery the
   logs show `obtained callback access token (private_key_jwt)` (no secrets, no token). With
   discovery you also see one `resolved OIDC discovery document` line (issuer + token
   endpoint only).

### 13.4 Walkthrough B — Client Credentials

1. Place and lock the client secret file (§13.2) and uncomment its mount.
2. In `.env`:
   ```bash
   CALLBACK_AUTH_MECHANISM=clientCredentials
   CALLBACK_AUTH_TOKEN_ENDPOINT=https://auth.example.com/oauth2/token
   CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   CALLBACK_AUTH_SCOPE=callback.write
   CALLBACK_AUTH_CLIENT_SECRET_FILE=/run/consent-proxy-secrets/callback-client-secret
   # CALLBACK_AUTH_CLIENT_AUTH_METHOD=client_secret_basic   # or client_secret_post
   # CALLBACK_AUTH_AUDIENCE=https://your-receiver.internal
   ```
   **Discovery variant** — replace `CALLBACK_AUTH_TOKEN_ENDPOINT` with the discovery URL;
   the token endpoint comes from the document:
   ```bash
   CALLBACK_AUTH_MECHANISM=clientCredentials
   CALLBACK_AUTH_DISCOVERY_URL=https://auth.example.com/.well-known/openid-configuration
   CALLBACK_AUTH_CLIENT_ID=your-callback-client-id
   CALLBACK_AUTH_SCOPE=callback.write
   CALLBACK_AUTH_CLIENT_SECRET_FILE=/run/consent-proxy-secrets/callback-client-secret
   ```
3. `docker compose up -d --force-recreate`, then verify per §8. On the first delivery the
   logs show `obtained callback access token (client_credentials)`. With discovery you also
   see one `resolved OIDC discovery document` line.

### 13.5 Interaction with `CALLBACK_AUTH_ORIGIN`
`CALLBACK_AUTH_ORIGIN` is **still required** to attach any bearer. The dedicated callback
token is attached **only** when the callback URL's origin matches `CALLBACK_AUTH_ORIGIN`;
otherwise no `Authorization` header is sent. The mechanism decides *which* dedicated token is
minted; the origin gate decides *whether* it is attached. The upstream ScrambleID token is
**never** used for callbacks. Two edge cases follow from this:

- **Mechanism `none` + `CALLBACK_AUTH_ORIGIN` set → startup failure.** An origin signals
  intent to authenticate, but `none` produces no token and the proxy will not reuse the
  upstream token, so it refuses to start (see §13.0 for the exact message).
- **Mechanism set + `CALLBACK_AUTH_ORIGIN` unset → token minted but not attached.** The
  dedicated credential is valid and a token is obtained, but with no origin to match the
  origin gate attaches nothing (no bearer on the delivery).

### 13.5a Audience & receiver token validation

- **Choosing `CALLBACK_AUTH_AUDIENCE`.** This is an **optional token-request parameter**
  (`audience`/`resource`) placed in the body of the request to the token endpoint. Set it to
  whatever your agent/receiver expects as the issued token's audience. Some authorization
  servers require it to mint a correctly-scoped token; others ignore it entirely. **Set it
  only when your receiver or authorization server requires it**, and leave it unset
  otherwise.
- **Do not confuse it with the client-assertion audience.** `CALLBACK_AUTH_AUDIENCE` is the
  intended audience of the **issued access token** (what your receiver checks). It is **not**
  the JWT **client-assertion** `aud`, which for `private_key_jwt` is generally the token
  endpoint / authorization-server issuer (see §13.1c above). One
  value describes where the issued token is meant to be used; the other describes where the
  signed assertion is presented to obtain that token. They are different things and are
  configured differently (`CALLBACK_AUTH_AUDIENCE` vs `CALLBACK_AUTH_ISSUER`).
- **What your receiver must validate** on every delivered access token:
  - the **signature**, against the issuing authorization server's JWKS;
  - the **issuer** (`iss`);
  - the **audience** (`aud`);
  - the **expiration** (`exp`), and
  - any **required scopes**.
- **Where to obtain the expected issuer and JWKS.** Both come from the authorization server
  that **issues** the callback token, via its OIDC discovery document — the `issuer` and
  `jwks_uri` fields of `.../.well-known/openid-configuration`. That is the same authorization
  server referenced by `CALLBACK_AUTH_DISCOVERY_URL` / `CALLBACK_AUTH_ISSUER` /
  `CALLBACK_AUTH_JWKS_URI`. When **ScrambleID** issues the token, ScrambleID provides these
  values; when **your own** authorization server issues it, you already have them.

### 13.6 Validation & troubleshooting
- **Startup fails with `callback.auth.<mechanism> requires callback.auth.clientId and one of
  callback.auth.tokenEndpoint or callback.auth.discoveryUrl`** — set `CALLBACK_AUTH_CLIENT_ID`
  and **either** `CALLBACK_AUTH_TOKEN_ENDPOINT` or `CALLBACK_AUTH_DISCOVERY_URL` (plus the
  secret file var).
- **Startup fails with `callback.auth.discoveryUrl must be an absolute https URL`** (or
  `jwksUri`) — the URL is missing, malformed, or not `https`. Fix the value; discovery and
  JWKS URLs must be absolute `https`.
- **`unusable private key`** — the PEM is malformed or the passphrase is wrong; re-check the
  file and `CALLBACK_AUTH_PRIVATE_KEY_PASSPHRASE_FILE`.
- **`resolved OIDC discovery document` never appears / discovery errors** — common causes:
  - *`discovery document returned HTTP <code>`* or *`discovery document request failed`* —
    the discovery URL is wrong or egress to the auth server is blocked; add it to allowed
    egress (§10).
  - *`discovery document missing token_endpoint`* — the document at that URL is not a valid
    OIDC discovery document.
  - *`discovery host blocked`* — the discovery host resolves to a metadata/loopback/
    link-local address (SSRF guard). Point it at the real auth server.
  - *`discovery issuer mismatch`* — your `CALLBACK_AUTH_ISSUER` does not equal the document's
    `issuer`. Remove it or correct it to match exactly.
- **`token endpoint returned HTTP <code>`** — the dedicated client/scope/endpoint is wrong or
  the egress to the token endpoint is blocked; add it to your allowed egress (see §10) and
  verify the client registration. The proxy never logs the token or secret.
- **Receiver rejects the token** — validate it at the receiver against the correct issuer's
  JWKS (ScrambleID's for ScrambleID-issued tokens, your issuer's for your-own-auth tokens)
  and confirm scope/audience expectations.
- See `customer-troubleshooting.md` for the general token/egress playbook.

---

## 14. What your receiver gets

The proxy sends one `POST` to the request's `callbackUrl` for each final outcome, with
`Content-Type: application/json`. The body is the outcome exactly as the consent server sends
it. Its fields are in "Outcomes to your Proxy" in the [consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

The proxy adds these headers:

| Header | Meaning |
|---|---|
| `x-callback-id` | The same on every delivery of one outcome. Use it to drop duplicates. |
| `x-event-id` | The outcome's position in the stream. |
| `Authorization` | `Bearer <token>`, only with a dedicated callback credential (§13). |

Return a `2xx` to confirm receipt. The proxy retries `408`, `429`, `5xx` and network errors
within `callbackRetry.totalBudgetMs`. It treats `3xx` and other `4xx` responses as permanent.
