# Consent Notification Proxy — Troubleshooting Runbook

**Audience:** the team operating the proxy, and ScrambleID support.
**Companion:** `customer-deployment.md`.

Work top-down: start at the symptom, run the **Confirm** command, apply the **Fix**. Logs are
structured JSON; grep for the quoted strings. Replace `8080` with your `CONSENT_PROXY_HOST_PORT`
if you changed it.

Quick triage:
```bash
docker ps --filter name=consent-proxy
docker inspect --format '{{.State.Health.Status}}' consent-proxy
docker logs --tail=200 consent-proxy
curl -s http://127.0.0.1:8080/health/ready
curl -s http://127.0.0.1:8080/metrics
```

---

## 1. Container won't start / exits immediately

**Confirm**
```bash
docker logs consent-proxy | grep -i "Fatal startup error\|Invalid configuration\|EADDRINUSE"
```

**Causes & fixes**
- `Invalid configuration: ...` → a required/!malformed config value. The proxy fails closed
  on bad config. Fix the field named in the message in `config/config.json` (or the matching
  env var) and recreate.
- `listen EADDRINUSE ... :8080` → the host port is already in use. Change
  `CONSENT_PROXY_HOST_PORT` in `.env` (e.g. `8090`) and `docker compose up -d`.
- Key not readable → see §4 (permissions / uid 1000).

---

## 2. `/health/ready` stays 503 (not ready)

503 means the SSE stream to the consent-server is **not connected**. Find out why:

```bash
docker logs --tail=100 consent-proxy | grep -i "SSE stream\|access token\|token endpoint\|401\|disconnected"
```

Branch on what you see:

### 2a. `SSE connect returned HTTP 401` (looping)
The proxy obtained a token but the consent-server **rejected it**. The token itself is the
problem — almost always a **JWKS / issuer / audience** mismatch.

**Confirm** — decode the token the proxy mints (header + payload are not secret):
- Capture the access token (or mint one the same way via your service account) and inspect
  `iss`, `aud`, `exp`, `sub`, `scope`, and the signing `kid`.
- Compare the `kid` against the keys published at the consent-server's configured **JWKS URL**.

**Fix**
- The consent-server's JWKS must include the issuer that signed your token. If your `kid`
  isn't in that JWKS → ScrambleID must add your issuer's JWKS (or you're pointed at the wrong
  environment). This is a configuration alignment between your `auth.issuer`/`auth.tokenEndpoint`
  and the consent-server's trusted issuers — raise it with ScrambleID.
- Verify `consent.baseUrl` and `auth.*` point at the **same environment** (don't mix a QA
  issuer with a production consent host).
- A missing scope would be `403`, not `401`.

### 2b. Token mint failing (never reaches 401)
```bash
docker logs consent-proxy | grep -i "privateKeyJwt\|token endpoint\|token_mint_failures"
```
- `unusable private key` → wrong/corrupt PEM or wrong passphrase. Re-check `secrets/private.pem`.
- `token endpoint returned HTTP 4xx/5xx` → wrong `auth.tokenEndpoint`, wrong `clientId`, the
  assertion rejected, or clock skew on the host (check NTP; the assertion `exp` is short).
- Request failed → egress blocked (see §2c).

### 2c. Egress blocked
```bash
# From the host (or `docker exec consent-proxy`), test reachability:
curl -sS -o /dev/null -w "%{http_code}\n" https://<consent-host>/
curl -sS -o /dev/null -w "%{http_code}\n" https://<token-endpoint-host>/
```
- Connection refused/timeout → firewall is not allowing outbound to that host.
- TLS errors behind a corporate proxy → set `HTTPS_PROXY`/`NO_PROXY` and
  `NODE_EXTRA_CA_CERTS` in `.env`, recreate.

---

## 3. Connected, but no callbacks arrive

First confirm outcomes are even being received:
```bash
curl -s http://127.0.0.1:8080/metrics | grep -E "outcomes_received_total|callbacks_"
```

### 3a. `callback rejected (validation/SSRF)`
```bash
docker logs consent-proxy | grep "validation/SSRF"
```
- `callback host not in allowlist: <host>` → add `<host>` to `callbackPolicy.allowedHosts`
  (exact, or a `.suffix`). Empty list = deny all.
- `Address <ip> is outside the destination's allowed ranges` → the resolved IP isn't in
  `allowedCidrs`. Add the correct range (e.g. the receiver's public `/32` or its subnet).
- `Address <ip> is in a blocked range` → loopback/link-local/metadata are always blocked.
  For local testing only, `allowLoopback: true` permits loopback.
- `callback URL must be HTTPS` → either make the receiver HTTPS or set `requireHttps: false`
  (not recommended in production).

Recreate the container after editing `config.json`.

### 3b. `outcomes_received_total` is 0 (nothing coming in)
- A wrong `consent.baseUrl` or `clientId` means your proxy connects but never receives your
  organization's events. Verify both against your service account.
- The request may name a different proxy. See "Outcomes to your Proxy" in the
  [consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

---

## 4. Private key permission errors

**Confirm**
```bash
ls -ln /opt/consent-proxy/secrets/private.pem   # expect owner 1000, mode 0400
docker logs consent-proxy | grep -i "unusable private key\|EACCES\|permission"
```

**Fix**
```bash
sudo chown 1000:1000 /opt/consent-proxy/secrets/private.pem
sudo chmod 0400       /opt/consent-proxy/secrets/private.pem
docker compose up -d --force-recreate
```
The container runs as uid 1000; the key must be readable by that uid.

---

## 5. Callbacks sent but your receiver rejects them

### 5a. Receiver returns 401/403
- This applies only with a dedicated callback credential (runbook §13). Your receiver must
  validate the `Authorization: Bearer <token>` against the JWKS of the auth server that issued
  it (signature + standard claims). Check its JWKS config.
- The bearer is attached **only** when the callback URL's origin equals `CALLBACK_AUTH_ORIGIN`.
  If they differ, no token is sent. Make `CALLBACK_AUTH_ORIGIN` exactly match your receiver's
  origin (scheme + host + port).

### 5b. Retry behavior
```bash
docker logs consent-proxy | grep -E "callback delivered|callback failed"
```
- `408 / 429 / 5xx / network` → retried with backoff within `callbackRetry.totalBudgetMs`
  (honors `Retry-After` on 429). Transient receiver issues self-heal.
- `3xx / other 4xx` → treated as permanent, not retried. Fix the receiver response.
- Budget exhausted → counted in `callbacks_failed_total`; the decision can be reconciled via
  the consent-server status API.

---

## 6. Duplicate callbacks

Expected by design (at-least-once delivery + reconnect catch-up). De-duplicate on the
**`x-callback-id`** header (stable per decision across retries/replays). `x-event-id` is the
stream position. This is not an error.

---

## 7. Metrics to watch / alert on

```bash
curl -s http://127.0.0.1:8080/metrics
```
- `sse_connected` (gauge, 1/0) — alert if `0` for a sustained period.
- `callbacks_failed_total` — alert on sustained increase.
- `sse_reconnects_total` — alert on churn.
- `acks_failed_total` — rising means acks aren't landing (outcomes may be replayed; dedup
  still protects the receiver).
- `token_mint_failures_total` — rising means the proxy can't get a token (see §2b).

---

## 8. Support bundle (send this to ScrambleID)

Collect:
```bash
docker logs --tail=500 consent-proxy > proxy-logs.txt
curl -s http://127.0.0.1:8080/metrics > proxy-metrics.txt
docker inspect --format '{{.Config.Image}} {{.State.Health.Status}}' consent-proxy > proxy-version.txt
```
Also include your `config.json` and `.env` **with secrets removed**, and (if a 401/JWKS
issue) the **decoded token claims** (`iss`, `aud`, `exp`, `sub`, `scope`, `kid`) — header and
payload only.

**Never send:** the private key, the raw access token, or any file contents from `secrets/`.
