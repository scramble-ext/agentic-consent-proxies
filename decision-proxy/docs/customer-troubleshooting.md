# Policy Decision Proxy — Troubleshooting Runbook

**Audience:** the team operating the proxy, and ScrambleID support.
**Companion:** `customer-deployment.md`.

Work top-down: start at the symptom, run the **Confirm** command, apply the **Fix**. Logs are
structured JSON; grep for the quoted strings. Replace `8080` with your
`DECISION_PROXY_HOST_PORT` if you changed it.

Quick triage:
```bash
docker ps --filter name=policy-decision-proxy
docker inspect --format '{{.State.Health.Status}}' policy-decision-proxy
docker logs --tail=200 policy-decision-proxy
curl -s http://<bind-ip>:8080/health/ready
curl -s http://<bind-ip>:8080/health/status
```

A `503` body has `error.code`, `state`, `reason`, `org`, `appliedRev`, `serverRev` and
`lastSyncAt`. State changes are logged once per change, with `org`, `from`, `to` and `reason`.
Log lines carry your `x-client-request-id` as `reqId`.

---

## 1. Container won't start / exits immediately

**Confirm**
```bash
docker logs policy-decision-proxy | grep -i "Invalid configuration\|EADDRINUSE"
```

**Causes & fixes**
- `Invalid configuration: ...` → a missing or malformed setting. Every problem is listed. Fix
  each named variable in `.env` (see `configuration.md`) and `docker compose up -d`.
- `DECISION_PROXY_PRIVATE_KEY_FILE ... cannot be read` → the key is missing or not readable by
  uid 1000. See §6.
- `... is not a usable private key` → not a PEM key, or the passphrase is wrong or missing.
- `... is an RSA key of <n> bits` or `... is a <type> key` → make an RSA key of 2048 bits or
  more (runbook §5).
- `listen EADDRINUSE` → the host port is in use. Change `DECISION_PROXY_HOST_PORT` in `.env`.
- Compose refuses to start with `Set DECISION_PROXY_BIND_IP ...` or `Set DECISION_PROXY_IMAGE
  ...` → set that variable in `.env`.

---

## 2. `/health/ready` stays 503, `POLICY_NOT_LOADED`

The proxy has no copy of your policies yet.

```bash
docker logs --tail=100 policy-decision-proxy | grep -i "orgCode\|serves\|fetch failed\|stream failed\|event stream answered"
curl -s http://<bind-ip>:8080/health/status    # read sync.lastError
```

### 2a. `policy snapshot fetch failed` / `policy event stream failed`
Read `err`:
- `token request answered 401 (invalid_client)` → the public key is not registered on the
  service account, or `DECISION_PROXY_KEY_ID`, `DECISION_PROXY_TOKEN_ISSUER` or
  `DECISION_PROXY_CLIENT_ID` is wrong.
- `answered 403` → the service account lacks `policy.sync`.
- `fetch failed: UNABLE_TO_GET_ISSUER_CERT_LOCALLY` → a proxy re-signs TLS. Mount its CA
  (runbook §10, Corporate network).
- Connection refused or timeout → egress is blocked (§2d).

### 2b. `M2M token has no valid orgCode claim`
The service account is not in an organization. Ask ScrambleID or your issuer admin to set its
`orgCode`.

### 2c. `M2M token is for org <b>, this decision proxy serves <a>`
The service account's organization changed while the proxy was running. It never serves one
organization from another's copy. Restart the container to serve the new one.

### 2d. `event stream answered 200 with content-type "text/html"`
A proxy or login page answers instead of ScrambleID. Allow `DECISION_PROXY_SERVER_URL` through.

Check reachability from the host:
```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://consent.scrambleid.com/
curl -sS -o /dev/null -w "%{http_code}\n" https://<token-issuer-host>/
```

---

## 3. `/health/ready` is 503, `POLICY_STALE`

The proxy has a copy but may not decide from it. `/health/status` `reason` says why.

| `reason` | Log line | Cause and fix |
|---|---|---|
| `heartbeat` | `no event for 15s, reconnecting` | No word from ScrambleID. Check the network, and that your forward proxy does not buffer or cut long-lived responses. |
| `behind` | `policy snapshot fetch failed` | An edit could not be fetched in time. Read `err` (§2a). |
| `clock` | `policy state changed` with `skewMs` | The host clock is off ScrambleID's. Fix NTP. A proxy that delays the stream does not cause this. |

The proxy recovers by itself once the cause is gone. Do not restart it for this: a restart
throws away the copy.

---

## 4. `POLICY_INCOMPATIBLE`

**Confirm**
```bash
docker logs policy-decision-proxy | grep "server policy versions differ"
```

ScrambleID and this image no longer agree on how a policy is read (`server` and `local` in the
line). Upgrade the image (runbook §9). Decisions go to the fallback until then.

---

## 5. Decision calls fail with another code

| Code | Log line | Cause and fix |
|---|---|---|
| `503 POLICY_UNAVAILABLE` | `policy check failed` | A decision failed inside the proxy. If `/health/status` says `cedar: failed`, see `cedar runtime failed to load`: the image is damaged or out of memory. It retries every 10s. |
| `503 AUTH_KEYS_UNAVAILABLE` | `token keys could not be fetched` | The proxy cannot reach `DECISION_PROXY_JWKS_URL`. Logged at most once a minute. |
| `401` | — | The gateway's token does not verify against `DECISION_PROXY_JWKS_URL`, or fails `DECISION_PROXY_ISSUERS` / `DECISION_PROXY_AUDIENCE`. |
| `403 FORBIDDEN` | — | The gateway's token lacks `policy.evaluate`, or is for another organization than the path or than the one this proxy serves. |
| `400 CONSENT_NOT_SUPPORTED` | — | The call carries a consent. Send it to ScrambleID (runbook §11). |

---

## 6. Private key permission errors

**Confirm**
```bash
ls -ln /opt/policy-decision-proxy/secrets/private.pem   # expect owner 1000, mode 0400
docker logs policy-decision-proxy | grep -i "cannot be read\|readable by group or others"
```

**Fix**
```bash
sudo chown 1000:1000 /opt/policy-decision-proxy/secrets/private.pem
sudo chmod 0400       /opt/policy-decision-proxy/secrets/private.pem
docker compose up -d --force-recreate
```
The container runs as uid 1000; the key must be readable by that uid.
`private key file is readable by group or others` is a warning, not a failure.

---

## 7. What to watch / alert on

There is no `/metrics` endpoint. Use your gateway's health check and `/health/status`:

- `/health/ready` not `200` for a sustained period, on every proxy at once.
- `state` other than `fresh`; `appliedRev` not equal to `serverRev` for long.
- `counters.unavailable` rising.
- `sync.lastError` set.

---

## 8. Support bundle (send this to ScrambleID)

Collect:
```bash
docker logs --tail=500 policy-decision-proxy > decision-proxy-logs.txt
curl -s http://<bind-ip>:8080/health/status > decision-proxy-status.txt
docker inspect --format '{{.Config.Image}} {{.State.Health.Status}}' policy-decision-proxy > decision-proxy-version.txt
```
Also include your `.env`.

**Never send:** the private key, any token, or any file from `secrets/`.
