# ScrambleID Policy Decision Proxy — Customer Overview

**Audience:** customer platform, security, and operations teams evaluating or hosting the
proxy inside their own network.

This document explains what the Policy Decision Proxy is, why you would run it, how it works,
what it needs from you, and how it behaves in production. The decision API itself is in
"Decisions" and "Deciding inside your network" in the
[consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

---

## 1. What it is, in one paragraph

The Policy Decision Proxy is an optional service you run inside your own network. It answers
your gateway's `POST /decisions` itself, from a copy of your policies, so a call does not leave
your network to be decided. It only decides: it never forwards a call, creates a consent
request or spends a consent. Everything else (writing policies, consent, the consent token)
stays with ScrambleID, and your gateway changes only the base URL it asks for decisions.

---

## 2. Why it is deployed in your network

Without it, your gateway asks ScrambleID for a decision on every call. With it:

- **Decisions are made inside your network.** The call's service, action and parameters go to
  the proxy, not to ScrambleID.
- **ScrambleID never connects in.** The proxy opens the connections: one long-lived event
  stream and a fetch of your policies when they change. The only inbound traffic is from your
  gateway.
- **Same answer as ScrambleID.** Same token, same body, same answer as ScrambleID's
  `POST /decisions`, for the same policies.

One proxy serves one organization: the one on its M2M token.

---

## 3. How it works

```
agent ─▶ your gateway ─▶ policy decision proxy (your network) ─▶ allow | deny | consent
                              ▲ policy copy and change notices, outbound only
                         ScrambleID ◀── your gateway / agent, only for consent
```

Step by step:

1. **Load.** On start the proxy gets its own M2M token (scope `policy.sync`) and fetches the
   whole copy of your policies. Until then it is `cold` and cannot decide.
2. **Listen.** It holds an event stream open to ScrambleID. ScrambleID sends a heartbeat every
   5 seconds and a `rev` event as soon as any policy or the organization switch changes. The
   events carry only a revision number.
3. **Catch up.** When the proxy is not at that revision it fetches the whole copy again and
   swaps it in at once, so a decision always sees one revision. An edit reaches the proxy
   within a second or two, and the emergency switch is no slower.
4. **Decide.** Your gateway sends `POST /orgs/{orgCode}/api/v1/decisions` with its usual
   `policy.evaluate` token. The proxy checks the token against your gateway's JWKS URL and
   answers `allow`, `deny` or `consent`.
5. **Refuse when unsure.** If its copy may be out of date, it answers `503`, never a decision
   (§5).

The copy lives in memory only, never on disk, and a restart fetches it again.

---

## 4. What the proxy does **not** do

- It does **not** forward calls. Your gateway does that.
- It does **not** run consent. A body with `consentToken` or `consentRequestId` is
  `400 CONSENT_NOT_SUPPORTED`: only ScrambleID can spend a consent once. Send those calls to
  ScrambleID, or check the consent token yourself.
- It does **not** decide from a copy that may be wrong. A stale copy is a `503`, not a `deny`.
- It does **not** store anything. No database, no volume. Nothing to back up.
- It does **not** accept traffic from anyone but your gateway, if you bind it as described in
  the runbook.

---

## 5. Availability and fallback

The proxy answers `503` when it may not decide. The body names why:

| Code | Why |
|---|---|
| `POLICY_NOT_LOADED` | It has not received your policies yet (just started, or ScrambleID not reachable since). |
| `POLICY_STALE` | Its copy may be out of date. By default: no word from ScrambleID for 30 seconds, still behind a change after 5 seconds, or its clock differs from ScrambleID's by more than 2 seconds. |
| `POLICY_INCOMPATIBLE` | ScrambleID and this proxy version no longer agree on how a policy is read. Upgrade the proxy. |

A restart or upgrade starts cold until the first fetch (usually under a second). A call that
arrives meanwhile waits up to 1.5 seconds (configurable) for it, then decides. A stale copy is
never waited for: it is `503` at once.

**Run two containers on separate hosts** so one can restart while the other serves.

**Fallback (recommended).** Your gateway's load balancer lists the proxy as primary and
`https://consent.scrambleid.com` as the fallback, with a health check on `/health/ready`. The
same token and body work on both, and the answers are equal, so decisions come from ScrambleID
until a proxy is ready again. The proxy never forwards a call itself.

Without a fallback, every decision fails while no proxy is ready. Treat a `503`, a timeout or
no answer as `deny`.

---

## 6. What you need to provide

| What | Example | Purpose |
|---|---|---|
| M2M service account | client id with scope `policy.sync` | How the proxy authenticates to ScrambleID. It belongs to your organization: the proxy serves the `orgCode` in its token, so there is no org setting. |
| Private key | RSA, 2048 bits or more, made on the proxy's host | Signs the proxy's token requests (`private_key_jwt`, no client secret). **You hold it.** Only the public key is registered on the service account. |
| Token URL and issuer | `https://<env>.scrambleid.com/oidc/<org>/token` | Where the proxy gets its token. Provided by ScrambleID, or your issuer admin if the account is on your issuer. |
| Your gateway's JWKS URL | the key set that signs your gateway's tokens | The proxy checks the gateway's `policy.evaluate` tokens against it. **You provide this.** |
| Bind address | an internal IP your gateway reaches | Where the proxy listens. **You choose this.** |
| A load balancer with fallback | e.g. an Apigee TargetServer set | Takes a stale proxy out of rotation and sends decisions to ScrambleID meanwhile. **You run this.** |

The service account may be on another issuer than your gateway's. Then set that issuer's JWKS
URL under **Policy decision proxy** on your dashboard's JWKS configuration page.

---

## 7. Security posture

- **Outbound-only to ScrambleID.** It opens connections to ScrambleID, your token issuer and
  your gateway's JWKS URL. It accepts connections only from your gateway.
- **No secret in the image.** The private key is mounted read-only at run time, read once at
  start, and never logged. It is never in `.env` or an environment variable.
- **Your policies stay in memory.** The copy (approver emails included) is never written to
  disk.
- **Every decision call is authenticated** with your gateway's token, checked against your
  gateway's JWKS URL.
- **Health probes need no token.** Let only your gateway reach the published port.
- **Hardened runtime.** Non-root, read-only filesystem, all Linux capabilities dropped,
  `no-new-privileges`, no volume.
- **Enterprise-friendly egress.** It supports a corporate forward proxy (`HTTPS_PROXY`) and a
  custom CA (`NODE_EXTRA_CA_CERTS`).

**If the key leaks or the host is lost,** remove the public key at the issuer. Until then,
whoever holds the key can read your policies.

---

## 8. Running and operating it

The proxy ships as a Docker container.

### Deployment model
- Two containers on separate hosts, behind your gateway's load balancer.
- Stateless, so nothing to back up or migrate.

### Health checks

| Probe | Path | Passes when |
|---|---|---|
| startup | `/health/startup` | It has a copy of the policies (even one gone stale), and its policy engine loaded. |
| readiness | `/health/ready` | It may decide right now, and is not shutting down. Point your gateway's health check here. |
| liveness | `/health/live` | The process answers. Docker's healthcheck uses this. |

A stale proxy must leave your gateway's rotation, not be restarted: it recovers by itself once
ScrambleID is reachable.

`GET /health/status` shows the state, the revision it has and the server's, lag, clock
difference, the last sync error, and decision counters. There is no `/metrics` endpoint.

### Graceful shutdown
On `SIGTERM`, readiness fails first and the container closes 5 seconds later, so your load
balancer moves away before it stops.

---

## 9. Common questions

**Does the proxy send our call parameters to ScrambleID?**
No. It decides locally. ScrambleID only sends it your policies.

**What happens when we edit a policy?**
ScrambleID sends a `rev` event at once. The proxy fetches the new copy, usually within a second
or two. Until it has, and for at most 5 seconds by default, it keeps deciding; after that it
answers `503 POLICY_STALE`.

**What if ScrambleID is unreachable?**
After 30 seconds without a heartbeat (default) the proxy answers `503 POLICY_STALE`, and your
load balancer sends decisions to the fallback. If the fallback is also unreachable, treat the
call as `deny`.

**Can it handle consent?**
No. A call carrying a consent is `400 CONSENT_NOT_SUPPORTED`. Check the consent token yourself,
or send that call to ScrambleID.

**Why does the host clock matter?**
The emergency switch can be time-limited. A host clock more than 2 seconds off ScrambleID's
stops decisions instead of deciding wrongly about it. Keep the host on NTP.

**What ports does it need open?**
Outbound HTTPS to ScrambleID, your token issuer and your gateway's JWKS URL. Inbound only from
your gateway, on the port you publish.

---

## 10. Quick glossary

- **Copy:** the proxy's in-memory copy of your organization's policies.
- **Revision:** a number ScrambleID raises on every policy or switch change. The proxy decides
  only when its copy is at the server's revision, or was very recently.
- **Stale:** the proxy cannot be sure its copy is current, so it answers `503`.
- **Fallback:** ScrambleID's own `POST /decisions`, used by your load balancer while no proxy
  is ready.
- **M2M token:** a short-lived access token that proves a service's identity. The proxy uses
  one (scope `policy.sync`) to read your policies; your gateway uses its own
  (`policy.evaluate`) to ask for decisions.
- **JWKS URL:** a public endpoint listing the keys that sign a token issuer's tokens.

---

*For deployment and day-to-day operations see `customer-deployment.md`; for every setting see
`configuration.md`; when something is wrong see `customer-troubleshooting.md`.*
