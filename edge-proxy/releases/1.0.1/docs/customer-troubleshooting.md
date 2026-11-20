# ScrambleID Edge Proxy — Troubleshooting Runbook

**Audience:** the team operating the proxy, and ScrambleID support.
**Companion:** `customer-deployment.md`.

Work top-down: start at the symptom, run the **Confirm** command, apply the **Fix**. Logs are
structured JSON; grep for the quoted strings. Replace `8080` with your `EDGE_PROXY_HOST_PORT` if you
changed it, and `127.0.0.1` with your `EDGE_PROXY_BIND_IP`.

Quick triage:
```bash
docker ps --filter name=edge-proxy
docker inspect --format '{{.State.Health.Status}}' edge-proxy
docker logs --tail=200 edge-proxy
curl -s http://127.0.0.1:8080/health/ready
curl -s http://127.0.0.1:8080/health/status
curl -s http://127.0.0.1:8080/metrics
```

---

## 1. Container won't start / exits immediately

**Confirm**
```bash
docker logs edge-proxy | grep -i "Invalid configuration\|cannot be read\|lacks\|token request failed"
```

**Causes & fixes**
- `Invalid configuration:` and a list → every wrong or missing setting, at once. Fix them all in
  `.env` and recreate.
- `... cannot be read (ENOENT)` → the key or secret file is not mounted where the setting says.
  Check the mount in `docker-compose.yml`.
- `... is not RSA` or `too small` → use an RSA key of 2048 bits or more, PKCS8.
- Key not readable (`EACCES`) → owner must be uid 1000, `chmod 0400` (deployment §5).
- `the account's token lacks policy.sync` (or `consent.outcome.read`) → the account has no scope for
  a feature that is on. Add it, or turn the feature off.
- `the account's token also carries ...` → a warning. The account has a scope no feature here uses.
  Prefer one account per feature.
- `token request failed at start` → the issuer was not reachable or refused the key or secret. It
  retries. Check `EDGE_PROXY_TOKEN_URL`, the client id, `EDGE_PROXY_AUTH_MECHANISM` and the
  registered public key or secret.

---

## 2. `/health/ready` stays 503

`curl /health/status` shows the state and `lastError`. Match the log:

| Log | Cause and fix |
|---|---|
| `event stream failed: ... answered 404` | ScrambleID is older than this proxy and has no combined stream. Ask for the server to be updated. |
| `event stream answered 200 with content-type "text/html"` | A proxy or captive portal answered instead of ScrambleID. Allow the host through, or fix the corporate proxy settings. |
| `no event for 15s, reconnecting` | A device between you and ScrambleID holds the stream back. Allow server-sent events and long-lived responses. |
| `event stream answered 401 (TOKEN_EXPIRED)` | The token expired. It renews and reconnects by itself. |
| `event stream answered 401` or `403`, `token request answered 400/401/403` (`refused: true`) | The account was refused: wrong key, JWKS, issuer, audience or scope. Retries slow to one every 10 minutes (`stream.nextRetryAt`). After the fix, **restart the proxy** to connect at once. |
| `event stream answered 403 (FORBIDDEN)` | The account lacks the scope of a feature. |
| TLS errors | A corporate CA: mount it and set `NODE_EXTRA_CA_CERTS` (deployment §10). |

A reconnect about every 15 minutes (the token's life) is normal. A failure that repeats is logged
once every 5 minutes, with `repeats`.

Test egress from the host:
```bash
curl -sI https://consent.scrambleid.com | head -1
curl -sI <EDGE_PROXY_TOKEN_URL> | head -1
```

---

## 3. Decisions

| Symptom | Cause and fix |
|---|---|
| `503 POLICY_NOT_LOADED` | No copy yet. It waits `EDGE_PROXY_FIRST_LOAD_WAIT_MS`, then answers this. Check the stream is open (§2). |
| `503 POLICY_STALE` | No heartbeat for `MAX_STALENESS_SECONDS`, behind ScrambleID for `MAX_APPLY_LAG_SECONDS`, or the clock is off (`reason: clock`). See `sync.lastError`. Goes back to fresh by itself. |
| `503 POLICY_INCOMPATIBLE` | ScrambleID's policies need a newer proxy. Upgrade it. |
| `401 UNAUTHORIZED` | The caller's token is not signed by a key in `EDGE_PROXY_JWKS_URL`, or its issuer or audience is not accepted. |
| `403 FORBIDDEN` | The token lacks `policy.evaluate`, or is for another org. |
| `400 CONSENT_NOT_SUPPORTED` | A call that spends a consent goes to ScrambleID, not here. |
| Answers differ from ScrambleID's | Never expected. Send the decision, the `appliedRev` from `/health/status` and the version. |

---

## 4. Connected, but no callbacks arrive

**Confirm**
```bash
curl -s http://127.0.0.1:8080/health/status       # callbacks: queued, inFlight, dropped
curl -s http://127.0.0.1:8080/metrics | grep -E "outcomes_received|callbacks_"
docker logs edge-proxy | grep -i "callback rejected\|callback gave up"
```

**Causes & fixes**
- `outcomes_received_total` is 0 → nothing is coming in. The agent's `proxyId` names another proxy,
  or the request had no callback URL.
- `callback rejected, not retried` with:
  - `callback host ... is not allowed` → add the host to `EDGE_PROXY_CALLBACK_ALLOWED_HOSTS`.
  - `... is outside the allowed ranges` or `loopback address` → the name resolves outside
    `EDGE_PROXY_CALLBACK_ALLOWED_CIDRS`, or to loopback/link-local/metadata.
  - `receiver answered 4xx` → your receiver refused it. Final, not retried (§5).
- `callback gave up: retry budget used` → your receiver kept failing for
  `EDGE_PROXY_CALLBACK_RETRY_BUDGET_SECONDS`. ScrambleID resends after the next reconnect or restart.
- `outcome queue is full` → deliveries are slower than outcomes arrive. Raise
  `EDGE_PROXY_CALLBACK_CONCURRENCY`. What was refused is resent.

---

## 5. Callbacks sent but your receiver rejects them

- **Receiver returns 401/403** → if you use callback authentication, check the log
  `callback token: ... answered 401 (invalid_client)`. Your own auth server refused the callback
  credential. It is not the ScrambleID one. Check client id, key or secret, and that
  `EDGE_PROXY_CALLBACK_AUTH_ORIGIN` equals the callback URL's origin. The token is attached only to
  that origin.
- **Receiver rejects the token** → validate it against your auth server's JWKS, issuer and audience
  (deployment §15).
- **Retries.** `408`, `429`, `5xx` and network errors retry for `RETRY_BUDGET_SECONDS`. Other `4xx`
  and `3xx` are final.

---

## 6. Duplicate callbacks

Expected by design (at-least-once delivery, resend after a reconnect or a lost ack). Two proxies on
one account each deliver the same outcome. De-duplicate on **`x-callback-id`** (stable per outcome).
`x-event-id` is its number at ScrambleID. This is not an error.

---

## 7. Metrics to watch / alert on

```bash
curl -s http://127.0.0.1:8080/metrics
```
- `sse_connected` (gauge, 1/0) — alert if `0` for a sustained period.
- `sse_reconnects_total` — alert on churn.
- `callbacks_failed_total` — alert on sustained increase.
- `outcomes_queue_full_total` — rising means deliveries cannot keep up.
- `acks_failed_total` — rising means acks aren't landing (outcomes may be resent; dedup still
  protects the receiver).
- `token_mint_failures_total` — rising means the proxy can't get a token (§1, §2).
- `/health/ready` — alert on `503`. Take the proxy out of rotation. Do not restart it.

---

## 8. Support bundle (send this to ScrambleID)

Collect:
```bash
docker logs --tail=500 edge-proxy > proxy-logs.txt
curl -s http://127.0.0.1:8080/health/status > proxy-status.txt
curl -s http://127.0.0.1:8080/metrics > proxy-metrics.txt
docker inspect --format '{{.Config.Image}} {{.State.Health.Status}}' edge-proxy > proxy-version.txt
```
Also include your `.env` **with secrets removed**.

**Never send:** a private key, a client secret, a raw access token, or any file contents from
`secrets/`.
