# ScrambleID Edge Proxy — Customer Overview

**Audience:** customer platform, security, and operations teams evaluating or hosting the
proxy inside their own network.

This document explains what the Edge Proxy is, why it runs in your network, how it works, what it
needs from you, and how it behaves in production. No prior knowledge of ScrambleID internals is
assumed.

---

## 1. What it is, in one paragraph

The Edge Proxy is one small container you run inside your own network. It opens **one outbound
connection** to the ScrambleID consent-server and does up to two jobs over it:

| Feature | What it does | Who talks to it |
|---|---|---|
| `decisions` | Answers `POST /orgs/{org}/api/v1/decisions` from a copy of your org's policies, so a decision never leaves your network. It can also decide tool calls inside **GCP Agent Gateway**. | Your agent gateway |
| `callbacks` | Delivers each consent outcome (approved, denied, expired, cancelled) to the callback URL the agent gave. | Your callback receiver |

You turn on what you use with `EDGE_PROXY_FEATURES`: one, or both. It replaces the Policy Decision
Proxy and the Consent Notification Proxy.

---

## 2. Why it is deployed in your network

- **No inbound connections.** Your firewall only allows outbound traffic. Nothing is opened inbound
  for ScrambleID.
- **Decisions stay local.** A decision is answered inside your network, in milliseconds.
- **Your receivers stay private.** The last hop (proxy → your receiver) happens inside your
  network. ScrambleID never calls into it.
- **You keep control.** You choose which hosts may receive a callback, and you hold the keys.

One proxy serves one organization (the `orgCode` in its token).

---

## 3. How it works

```
                 outbound, one connection (HTTPS, SSE)
 ScrambleID  <-------------------------------------------  Edge Proxy  --- decisions --->  your gateway
 consent                                                       |
 server       policy changes and outcomes come back on it      +--------- callbacks ---->  your receiver
```

### Decisions

1. The proxy keeps a copy of your org's policies **in memory**. It never writes it to disk.
2. The stream carries a revision number, not policies. When the proxy is behind, it fetches the
   whole snapshot and swaps it in at once, so a decision never sees half a change. A policy change
   reaches it in about a second.
3. Your gateway sends the same call and token it would send to ScrambleID. The answer is the same:
   `allow`, `deny`, or `consent` (with the approvers).
4. If its copy may be out of date it answers `503` and says why. It never guesses. A stale copy is
   never a `deny`.
5. A call that carries a ScrambleID **consent token** for exactly that call turns a `consent` answer
   into `allow`. The proxy checks the token itself, against ScrambleID's public keys, without a call
   to ScrambleID.

### GCP Agent Gateway (optional, part of `decisions`)

GCP Agent Gateway hands each request to an external authorization service instead of calling
`POST /decisions`. Turn on the `ext_proc` adapter and the proxy answers it.

1. The gateway sends the agent's request as it is. The proxy reads the MCP `tools/call` in it and
   takes the URL the agent called as the ScrambleID service and the tool name as the action.
2. It decides from the same copy of the policies, and the gateway either forwards the request or
   answers the agent with a JSON-RPC error saying `deny` or `consent_required`.
3. On `consent_required` the agent asks ScrambleID for consent, and once approved, sends the same
   call again with the consent token. The proxy lets exactly that call through.

Setup: [customer-gcp-agent-gateway.md](customer-gcp-agent-gateway.md).

### Callbacks

1. When a consent request ends, ScrambleID sends the outcome down the connection.
2. The proxy `POST`s it to the request's `callbackUrl`, after checking the URL (§6).
3. It tells ScrambleID the outcome was delivered. Anything not acknowledged is sent again after a
   reconnect or restart.

---

## 4. What the proxy does **not** do

- It does **not** forward decision calls on `POST /decisions`. Your gateway does that. With an
  adapter, the gateway still does the forwarding; the proxy only says whether it may.
- It does **not** spend a consent. A `consentToken` is checked here and accepted for exactly its call
  until it expires (15 minutes at most), more than once, and ScrambleID does not show it as used. A
  consent revoked in ScrambleID is still accepted until then. A `consentRequestId` without a token is
  `400 CONSENT_NOT_SUPPORTED`: only ScrambleID has the records to check it.
- It does **not** turn a `deny` into `allow`, with or without a token.
- It does **not** decide from a copy that may be wrong.
- It does **not** store anything. No database, no volume. Nothing to back up.
- It does **not** send outcomes that have no `callbackUrl`.
- It does **not** reuse its ScrambleID token for your receiver.

---

## 5. Availability, fallback and delivery

### Decisions

The proxy answers `503` when it may not decide. The body names why:

| Code | Why |
|---|---|
| `POLICY_NOT_LOADED` | It has no copy yet (just started, or ScrambleID not reachable since). |
| `POLICY_STALE` | Its copy may be out of date. By default: no heartbeat for 30 seconds, behind a change for 5 seconds, or its clock differs from ScrambleID's by more than 2 seconds. |
| `POLICY_INCOMPATIBLE` | ScrambleID and this proxy version no longer agree on how a policy is read. Upgrade the proxy. |

A call right after a start waits up to 1.5 seconds (configurable) for the first copy. A stale copy
is never waited for: it is `503` at once. It goes back to fresh by itself once the cause clears.

**Run two containers on separate hosts** so one can restart while the other serves.

**Fallback (recommended).** Your gateway's load balancer lists the proxy as primary and
`https://consent.scrambleid.com` as the fallback, with a health check on `/health/ready`. The same
token and body work on both and the answers are equal. The proxy never forwards a call itself.
Without a fallback, every decision fails while no proxy is ready. Treat a `503`, a timeout or no
answer as `deny`.

### Callbacks

- Retries for up to 2 minutes (`408`, `429`, `5xx`, network errors). Other `4xx` and `3xx` are final.
- A receiver outage does not take decisions out of rotation.
- Delivery is **at least once**. Each delivery carries `x-callback-id` (same for every delivery of
  one outcome) and `x-event-id`. Drop duplicates on either.

---

## 6. What you need to provide

| What | Example | Purpose |
|---|---|---|
| M2M service account | client id with `policy.sync` and/or `consent.outcome.read` | How the proxy signs in to ScrambleID. |
| Private key | RSA 2048+ bits, made on the proxy's host (or a client secret file) | Signs the token request (`private_key_jwt`). **You hold it.** Only the public key is registered. |
| Token URL | `https://<env>.scrambleid.com/oidc/<org>/token` | Where the proxy gets its token. Provided by ScrambleID, or your issuer admin. |
| Gateway JWKS URL (`decisions`) | the key set that signs your gateway's tokens | The proxy checks `policy.evaluate` tokens against it. |
| Bind address (`decisions`) | an internal IP your gateway reaches | Where the proxy listens. |
| Callback allow list (`callbacks`) | `receiver.internal` | Hosts that may receive a callback. |
| TLS certificate and key (`ext_proc`) | PEM files; self-signed is enough for GCP Agent Gateway | The gateway connects to the adapter's gRPC port over TLS. |

### Scopes

Give each feature only the scope it needs. The proxy asks for the scopes of the features that are on.

| Feature | Scope |
|---|---|
| `decisions` | `policy.sync` |
| `callbacks` | `consent.outcome.read` |

- **One account for both** is simplest. Whoever holds the key can do both.
- **Two accounts** keep them apart: two proxies from the same image, one feature each, each with its
  own account and key (deployment runbook §12).
- A missing scope stops the proxy at start and names it. An extra scope is a warning.
- Gateway adapters and consent tokens are part of `decisions` and need no other scope.

### The callback allow list (your main control)

Agents choose the callback URL, so every URL is checked before every delivery: https, no
credentials in the URL, host on `EDGE_PROXY_CALLBACK_ALLOWED_HOSTS`, and the address the connection
really uses inside `EDGE_PROXY_CALLBACK_ALLOWED_CIDRS` (if set). Metadata and link-local addresses
are always blocked. Redirects are never followed.

### Authenticating the delivery to your receiver (optional)

Off by default: no `Authorization` header. To turn it on, make a **separate** credential at your own
auth server and set `EDGE_PROXY_CALLBACK_AUTH_*` (deployment runbook §15). The token goes only to
`EDGE_PROXY_CALLBACK_AUTH_ORIGIN`. It is never the ScrambleID token.

---

## 7. Security posture

- **Outbound-only to ScrambleID.** It also reaches your token issuer, your JWKS URL and your
  receivers. The inbound surface is the published port (health, metrics, decisions) and, with the
  `ext_proc` adapter, the gRPC port. The gRPC port has no caller token, as gateways do
  not send one: let only your gateway reach it, over TLS (mTLS where the gateway can present a
  certificate).
- **Adapters fail closed.** A tool call passes only when a fresh copy of the policies allows it, or
  a valid consent token covers exactly that call. A body it cannot read, a body the gateway cut or did
  not send, a stale copy, or consent keys it could not fetch: the call is refused.
- **No secret in the image.** Keys are mounted read-only at run time.
- **Hardened container.** Non-root, read-only filesystem, all capabilities dropped,
  `no-new-privileges`, no volume.
- **Least privilege.** One scope per feature. One org per proxy: tokens or outcomes of another org
  are refused or dropped.
- **Nothing at rest.** Policies are in memory only.
- **Quiet logs.** Ids, states and counts only. Never a policy, a request body, a callback path or a
  token.

---

## 8. Running and operating it

### Deployment model

One container per host, run with Docker Compose. Stateless: restarts reconnect and catch up. See
`customer-deployment.md`.

### Health checks

| Path | Meaning |
|---|---|
| `/health/live` | Process is up. Never depends on ScrambleID. |
| `/health/startup` | 200 once there is something to serve. |
| `/health/ready` | 200 when it would do its job: fresh policies (`decisions`), connected (`callbacks`). 503 while draining. |
| `/health/status` | Org, state, revision, last error, queue, counts. With `decisions`: the consent token keys (`consentTokens`). With adapters: `adapters`. |

Point readiness at `/health/ready` and liveness at `/health/live`. **Do not restart on ready
failing.** A stale proxy should leave rotation, not restart.

### Metrics

`/metrics` (Prometheus text): `sse_connected`, `outcomes_received_total`,
`callbacks_delivered_total`, `callbacks_failed_total`, `callback_latency_ms_*`, `acks_sent_total`,
`outcomes_queued`, `callbacks_in_flight`, `decisions_<answer>_total`. With adapters:
`adapter_calls_allowed_total`, `adapter_calls_denied_total`, `adapter_calls_consent_required_total`,
`adapter_decision_ms_*`. No ids, tenants or URLs.

### Graceful shutdown

On stop it drains deliveries in flight (`EDGE_PROXY_DRAIN_SECONDS`, default 15 s). What was not
delivered stays unacknowledged and is sent again later.

---

## 9. Common questions

**Do I need both features?** No. Turn on what you use.

**Can I run more than one?** Yes. For `decisions`, put them behind your gateway's balancer. For
`callbacks`, each proxy on the same account gets the same outcomes, so your receiver sees
duplicates: drop them on `x-callback-id`.

**What if my receiver is down?** Deliveries retry for 2 minutes, then ScrambleID resends after the
next reconnect or restart.

**What if ScrambleID is unreachable?** `decisions` goes stale and answers `503` (use the fallback).
`callbacks` waits and reconnects.

**What happens on upgrade?** Run the new version with the same settings. If ScrambleID changes
something an older proxy would answer differently, it answers `POLICY_INCOMPATIBLE` rather than
differ: upgrade it.

**I run the older proxies. Do I have to move?** No, they keep working. To switch, see the
deployment runbook §13.

**My gateway does not call `POST /decisions`. Can it still use ScrambleID?** For GCP Agent Gateway,
yes: turn on the `ext_proc` adapter ([guide](customer-gcp-agent-gateway.md)).

**Is a consent token used up?** Not here. ScrambleID marks a consent used when it decides the call
itself. The proxy checks the token locally, so the same call is accepted again until the token
expires (15 minutes at most).

---

## 10. Quick glossary

| Term | Meaning |
|---|---|
| Edge Proxy | This container. |
| Feature | `decisions` or `callbacks`. |
| Gateway adapter | A way for GCP Agent Gateway to ask `decisions` in its own protocol (`ext_proc`). |
| Consent token | Signed proof from ScrambleID that people approved one exact call. |
| Revision (`rev`) | Number that goes up on every policy change in your org. |
| Fresh / stale | Whether the policy copy may be used to decide. |
| Callback URL | Where the agent asked the outcome to be sent. |
| Allow list | The hosts and ranges a callback may go to. |
| M2M service account | The proxy's identity at ScrambleID. |
