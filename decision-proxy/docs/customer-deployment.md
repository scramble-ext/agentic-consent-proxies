# Policy Decision Proxy — Customer Deployment Runbook

**Audience:** the team deploying and operating the proxy inside your own network.
**Companion docs:** `customer-overview.md` (what it is and why), `configuration.md` (every
setting), `customer-troubleshooting.md` (when something is wrong). Deployment templates are in
`decision-proxy/deploy/` of the `agentic-consent-proxies` repo.

The proxy answers your gateway's `POST /decisions` from inside your network, from a copy of
your policies it keeps in memory. It connects out to ScrambleID; only your gateway connects in.

---

## 1. Prerequisites

- A Linux host you control, with **Docker** and **Docker Compose v2** (`docker compose`).
  Two hosts for availability (§12).
- **Outbound HTTPS (443)** from that host to:
  - ScrambleID (`https://consent.scrambleid.com`),
  - your token issuer (`DECISION_PROXY_TOKEN_URL`),
  - your gateway's JWKS URL (`DECISION_PROXY_JWKS_URL`).
- **Inbound** only from your gateway, on the port you publish.
- An **M2M service account** with the scope `policy.sync` (§2).
- The host clock on **NTP**. More than 2 seconds off ScrambleID's stops decisions.

Nothing is stored on disk except the key file.

---

## 2. Who provides what

| Provided by ScrambleID | Provided by you (customer) |
|---|---|
| Container image (registry **or** offline tarball) + its checksum/digest | The host, Docker, and firewall rules |
| `DECISION_PROXY_SERVER_URL`, `DECISION_PROXY_TOKEN_URL`, `DECISION_PROXY_TOKEN_ISSUER` | The private key (`secrets/private.pem`), made on the host |
| The pinned image version to run | `DECISION_PROXY_JWKS_URL`: the key set that signs your gateway's tokens |
| | `DECISION_PROXY_BIND_IP`: the internal IP your gateway reaches |
| | Your gateway's load balancer and fallback (§11) |

### Create the proxy's service account

Do this first. The proxy cannot start without a client id.

1. Create a service account for the proxy through the ScrambleID M2M flow, with the scope
   `policy.sync`. It belongs to your organization: the proxy serves the `orgCode` in its token,
   so there is no org setting.
2. It signs in with a **private key** (`private_key_jwt`, no client secret). Make the key on the
   proxy's host (§5) and register only its **public key** on the service account.
3. Use its client id as `DECISION_PROXY_CLIENT_ID`, and the registered key's id, if it has one,
   as `DECISION_PROXY_KEY_ID`.

The account may be on another issuer than your gateway's. Then set that issuer's JWKS URL under
**Policy decision proxy** on your dashboard's JWKS configuration page.

---

## 3. Get the image

Pin the **exact version** ScrambleID gives you (a tag like `1.0.0`, or a `@sha256:...` digest).
Never `latest`.

### Option A — pull from the registry
The image is public, so no login is needed.
```bash
docker pull ghcr.io/scramble-ext/policy-decision-proxy:<version>
docker image inspect ghcr.io/scramble-ext/policy-decision-proxy:<version> --format '{{index .RepoDigests 0}}'
```
Compare the digest to the one in `decision-proxy/releases/<version>/SHA256SUMS` of the
`agentic-consent-proxies` repo.

### Option B — offline tarball
```bash
sha256sum policy-decision-proxy-<version>.tar.gz     # compare to the checksum ScrambleID sent
docker load -i policy-decision-proxy-<version>.tar.gz
```

Set the image reference in `.env` as `DECISION_PROXY_IMAGE`.

---

## 4. Lay out the stack directory

```
/opt/policy-decision-proxy/
├── docker-compose.yml   # from decision-proxy/deploy/docker-compose.yml
├── .env                 # from decision-proxy/deploy/.env.example, filled in (no secrets)
└── secrets/
    └── private.pem      # YOUR private key (§5)
```

Run these and §5 as root (or with `sudo`):

```bash
git clone https://github.com/scramble-ext/agentic-consent-proxies.git
cd agentic-consent-proxies
mkdir -p /opt/policy-decision-proxy/secrets && chmod 700 /opt/policy-decision-proxy/secrets
cp decision-proxy/deploy/docker-compose.yml /opt/policy-decision-proxy/docker-compose.yml
cp decision-proxy/deploy/.env.example       /opt/policy-decision-proxy/.env
```

If ScrambleID sent you a delivery bundle, use its `docker-compose.yml` and `.env` instead.
They are already filled in for your organization.

---

## 5. Make the private key

Make it on this host, so the private half never travels:

```bash
cd /opt/policy-decision-proxy
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out secrets/private.pem
openssl pkey -in secrets/private.pem -pubout -out decision-proxy.pub   # send only this file
sudo chown 1000:1000 secrets/private.pem && sudo chmod 0400 secrets/private.pem
```

- Send ScrambleID (or your issuer admin) **only `decision-proxy.pub`**, to register on the
  service account. Never email, chat or ticket the private key.
- RSA, 2048 bits or more, PEM (PKCS8 or PKCS1). The proxy refuses anything else at start.
- The compose file mounts it read-only at `/run/policy-decision-proxy-secrets/private.pem`. It
  is never in the image, in `.env` or in an environment variable, and the proxy never logs it.
- The proxy reads it once at start. Keep it out of git and out of backups other people can
  read. A file that group or others can read starts with a warning.
- An encrypted key also needs its passphrase as a file: uncomment its mount in
  `docker-compose.yml` and set `DECISION_PROXY_PRIVATE_KEY_PASSPHRASE_FILE`.
- If the key leaks or the host is lost, remove the public key at the issuer. Until then,
  whoever holds it can read your policies (approver emails). Then rotate (§9).

---

## 6. Configure

All settings are environment variables in `.env`. No secret goes in it: the key is the file
above. Fill in at least:

- `DECISION_PROXY_IMAGE` — the pinned image from §3.
- `DECISION_PROXY_BIND_IP` — the internal IP your gateway reaches. Not `0.0.0.0`.
- `DECISION_PROXY_SERVER_URL` — `https://consent.scrambleid.com`.
- `DECISION_PROXY_TOKEN_URL`, `DECISION_PROXY_TOKEN_ISSUER` — the values ScrambleID gave you
  (§2).
- `DECISION_PROXY_CLIENT_ID`, and `DECISION_PROXY_KEY_ID` if the key has an id — the service
  account (§2).
- `DECISION_PROXY_JWKS_URL` — the key set that signs your gateway's `policy.evaluate` tokens:
  the same one ScrambleID checks them against (**Decision** on your dashboard's JWKS
  configuration page).
- `DECISION_PROXY_PRIVATE_KEY_FILE` — already set in the template to match the mount.

Every setting is in `configuration.md`. A wrong setting stops the container at start and lists
every problem (`docker logs policy-decision-proxy`).

---

## 7. Run

```bash
cd /opt/policy-decision-proxy
docker compose up -d
docker compose logs -f
```

The compose file pins the image via `${DECISION_PROXY_IMAGE}`, mounts your key read-only,
drops all capabilities, and runs read-only and non-root with no volume.

---

## 8. Verify

```bash
curl -s http://<bind-ip>:8080/health/ready     # 200 {"state":"fresh"}
curl -s http://<bind-ip>:8080/health/status
docker inspect --format '{{.State.Health.Status}}' policy-decision-proxy
```

Replace `8080` with your `DECISION_PROXY_HOST_PORT` if you changed it.

`/health/status` shows:

- `state`: `cold`, `fresh`, `stale` or `incompatible`;
- `reason` when stale: `heartbeat`, `behind` or `clock`;
- `appliedRev` and `serverRev`: these should be equal;
- `lagMs`, `skewMs`, and sync and decision counters;
- `cedar` (the policy engine): `loading`, `ready` or `failed`;
- `sync.lastError`: the last stream or snapshot failure, with the status and error code the
  server gave.

In the logs you should see `policy decision proxy started`, then `policy state changed` to
`fresh`. Then send one decision through your gateway and check that `counters.decisions` goes
up.

| Probe | Path | Passes when |
|---|---|---|
| startup | `/health/startup` | It has a copy of the policies (even one gone stale), and Cedar loaded. |
| readiness | `/health/ready` | It may decide right now, and is not shutting down. Point your gateway's health check here. |
| liveness | `/health/live` | The process answers. Docker's healthcheck uses this. |

The probes need no token. Let only your gateway reach the published port.

If `/health/ready` stays `503`, see `customer-troubleshooting.md`.

---

## 9. Operate

### Logs
```bash
docker logs --tail=100 -f policy-decision-proxy
```
Structured JSON. Logs never carry a policy, a parameter, a token or the key. Log lines carry
your `x-client-request-id` as `reqId`.

### Upgrade
Upgrade after ScrambleID changes how a policy is read. A proxy that no longer agrees answers
`POLICY_INCOMPATIBLE` (and `/health/ready` fails) until it is upgraded.
```bash
# set DECISION_PROXY_IMAGE in .env to the new pinned version, then:
cd /opt/policy-decision-proxy
docker compose pull      # skip for a tarball; docker load it instead
docker compose up -d
```
With two containers, upgrade one, wait for `/health/ready` to return 200, then the other.

### Rollback
```bash
# set DECISION_PROXY_IMAGE back to the previous version, then:
docker compose up -d
```
There is no state to migrate.

### Rotate the private key
1. Make the new key (§5) and send its public half to be registered on the service account
   **next to the old one**.
2. Replace `secrets/private.pem` (same owner and mode), set `DECISION_PROXY_KEY_ID` if the new
   key has another `kid`, and run `docker compose up -d --force-recreate`. A second container
   keeps serving meanwhile.
3. Check `/health/ready` is 200, then remove the old public key at the issuer.

### Stop
```bash
docker compose down          # nothing persisted
```
On `SIGTERM`, ready fails first and the container closes 5s later, so the gateway moves away
before it stops. Keep the stop grace period above 5s (`docker-compose.yml` sets 20s).

---

## 10. Network / egress summary

Open **outbound** HTTPS from the host:
- → ScrambleID (`DECISION_PROXY_SERVER_URL`).
- → your token issuer (`DECISION_PROXY_TOKEN_URL`).
- → your gateway's JWKS URL (`DECISION_PROXY_JWKS_URL`).

Open **inbound** only from your gateway, to `DECISION_PROXY_BIND_IP:DECISION_PROXY_HOST_PORT`.

### Corporate network
- **TLS inspection:** copy your CA to `secrets/corp-ca.pem`, uncomment its mount in
  `docker-compose.yml`, and set `NODE_EXTRA_CA_CERTS=/run/policy-decision-proxy-secrets/corp-ca.pem`.
- **Forward proxy:** set `NODE_USE_ENV_PROXY=1` and `HTTPS_PROXY`. Without
  `NODE_USE_ENV_PROXY`, Node ignores `HTTPS_PROXY`.
- The event stream is long-lived. A proxy that buffers or cuts idle connections makes the
  decision proxy go stale. Allow streaming to `consent.scrambleid.com`.

---

## 11. Pointing your gateway at it

Your gateway keeps its token and its request. Only the base URL changes:
`http://<bind-ip>:<port>/orgs/{orgCode}/api/v1/decisions`.

Two things the gateway has to handle:

1. **A call that carries a consent** (`consentToken` or `consentRequestId`) is `400
   CONSENT_NOT_SUPPORTED` here. Check the consent token yourself ("Validating the consent
   token" in the [consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent)),
   or send that one call to ScrambleID's `POST /decisions`.
2. **A `503`** (`POLICY_NOT_LOADED`, `POLICY_STALE`, `POLICY_INCOMPATIBLE`) or no answer at all.

### Fallback to ScrambleID (recommended)

In Apigee, a TargetServer load balancer:

- each decision proxy container as a TargetServer;
- `https://consent.scrambleid.com` as an `IsFallback` server;
- a HealthMonitor on `/health/ready`, so a stale decision proxy leaves rotation;
- `ServerUnhealthyResponse` with `503`, so a call that hits a decision proxy between health
  checks retries on the next server instead of failing.

The token and request are the same on both, and the answers are equal. Decisions come from
ScrambleID until a decision proxy is ready again. The decision proxy never forwards a call
itself.

Without a fallback, every decision fails while no decision proxy is ready. Treat it as `deny`.

### Checking the fallback by hand

In staging, with the decision proxy and the fallback configured:

1. Send a decision through the gateway. `/health/status` `counters.decisions` goes up.
2. Block the decision proxy's access to ScrambleID for longer than
   `DECISION_PROXY_MAX_STALENESS_SECONDS`. `/health/ready` turns `503`, and the HealthMonitor
   takes it out.
3. Send the same decision again. ScrambleID answers it, with the same body.
4. Unblock. Within a few seconds `/health/ready` is `200` and the decision proxy is back in
   rotation.
5. Restart the container with `docker compose restart`. Decisions keep working through the
   fallback.

---

## 12. Availability

The copy is in memory. A restart or upgrade starts cold until the first fetch (usually under a
second). A call that arrives meanwhile waits up to `DECISION_PROXY_FIRST_LOAD_WAIT_MS` for it,
then decides. Past that, it is `503 POLICY_NOT_LOADED`. A stale copy is never waited for: it
is `503` at once.

- Run **2 containers on separate hosts** so one can restart while the other serves.
- With one container, every restart sends traffic to the fallback.

---

## 13. Security posture (for your review)

- **No secret in the image.** The private key is made on your host and mounted read-only at
  run time; only its public half leaves the host.
- **Non-root, read-only filesystem, all capabilities dropped, `no-new-privileges`, no volume.**
- **Outbound to ScrambleID only.** The one inbound surface is the published port, for your
  gateway.
- **Policies in memory only.** Never written to disk; a restart fetches them again.
- **Never decides from a copy that may be wrong.** A stale copy is a `503`, not a decision.

---

## 14. If you need support

Collect and send (see `customer-troubleshooting.md` §"Support bundle"): a recent `docker logs`
excerpt, a `/health/status` snapshot, your `.env` **with secrets removed**, and the image
version. Do **not** send the private key or any token.
