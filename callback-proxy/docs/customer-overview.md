# ScrambleID Consent Notification Proxy — Customer Overview

**Audience:** customer platform, security, and operations teams evaluating or hosting the
proxy inside their own network.

This document explains what the Consent Notification Proxy is, why it is deployed in your
environment, how it works, what it needs from you, and how it behaves in production. It is
written for a general technical audience — no prior knowledge of ScrambleID internals is
assumed.

---

## 1. What it is, in one paragraph

The Consent Notification Proxy is a small, self-contained service that you run inside your
own network. Its only job is to **listen for consent decisions from the ScrambleID
consent-server and deliver each decision to an application endpoint of yours** (your
"callback URL"). It is the piece that lets your internal systems be notified the moment a
consent request is approved, denied, expired, or cancelled — without those internal systems
ever being exposed to the public internet.

Think of it as a one-way doorbell: ScrambleID rings it, and the proxy forwards the message
to the right internal address you configured. It never lets anyone ring your internal
systems from the outside.

---

## 2. Why it is deployed in your network

Your workflow systems that need to react to consent decisions typically live on private
addresses inside your network. There are two ways to connect them to ScrambleID:

1. Open those internal systems to inbound traffic from ScrambleID (requires firewall holes,
   public DNS, and exposes your systems).
2. Run a small outbound-only relay inside your network that pulls decisions and hands them
   to your internal endpoint locally.

The proxy is option 2. It is deployed in **your** network specifically so that:

- **No inbound connections are required.** The proxy makes a single outbound connection to
  the ScrambleID consent-server. Your firewall only needs to allow outbound traffic; nothing
  needs to be opened inbound.
- **Your internal callback endpoints stay private.** The final delivery hop (proxy →
  your endpoint) happens entirely inside your network. The callback URL never leaves your
  perimeter.
- **You keep control.** You decide which internal hosts the proxy is allowed to deliver to,
  and you hold the credentials. ScrambleID cannot reach arbitrary systems in your network.

One proxy instance is deployed per customer (per tenant).

---

## 3. How it works

```
   Your network                                   ScrambleID
 ┌───────────────────────────────────┐          ┌───────────────────┐
 │                                    │  outbound │                   │
 │   Consent Notification Proxy  ─────┼──────────▶│  Consent Server   │
 │            │                       │   (SSE,   │                   │
 │            │  decision arrives     │  held open)│                  │
 │            ▼                       │          └───────────────────┘
 │   Your callback endpoint           │
 │   (e.g. https://workflow.internal) │
 └───────────────────────────────────┘
```

Step by step:

1. **Connect.** On startup the proxy opens one authenticated, long-lived connection to the
   ScrambleID consent-server using a technology called **Server-Sent Events (SSE)** — a
   standard, HTTP-based, one-way stream. The connection is **outbound only** and stays open.

2. **Wait.** The proxy holds the connection open and waits. ScrambleID sends periodic
   heartbeats so both sides know the link is healthy.

3. **Receive a decision.** When a consent request reaches a final state, the consent-server
   pushes a small `consent.outcome` message over the stream. The decision is one of:
   `APPROVED`, `DENIED`, `EXPIRED`, or `CANCELLED`. The message contains the request id,
   the status, the tenant, and the callback URL to deliver to. **It contains no secrets, no
   passwords, and no raw user data.**

4. **Validate the destination.** Before delivering, the proxy re-checks the callback URL
   against the allowlist you configured (see §6). If the destination is not explicitly
   permitted, the proxy refuses to send. This is what prevents the proxy from ever being
   turned into an open relay.

5. **Deliver.** The proxy makes a local HTTP `POST` to your callback endpoint with the
   decision in the body. Your endpoint does whatever it needs to do and returns a success
   response.

6. **Acknowledge.** Once your endpoint confirms receipt (a `2xx` response), the proxy tells
   the consent-server "delivered," so the same decision is not re-sent on the next
   reconnect.

If anything fails along the way, the proxy retries briefly and then moves on (see §5).

---

## 4. What the proxy does **not** do

To set clear expectations about its blast radius:

- It does **not** accept inbound business traffic. The only inbound endpoints it exposes are
  local health and metrics checks (see §7) — no API to create, approve, or cancel anything.
- It does **not** store data. There is no database, queue, or disk-backed state. Nothing to
  back up, nothing that persists decisions.
- It does **not** forward to arbitrary addresses. Destinations must be on your allowlist.
- It does **not** receive secrets or user content in the decision messages.
- It does **not** keep your internal systems reachable from outside — it is a one-way relay.

---

## 5. Reliability and delivery guarantees

Delivery is **best-effort with at-least-once semantics**. In plain terms:

- When your endpoint is reachable and returns success, the decision is delivered promptly.
- If your endpoint is briefly unavailable (timeout, `429 Too Many Requests`, or a `5xx`
  error), the proxy retries automatically a few times with increasing delay, honoring any
  `Retry-After` your endpoint sends, up to a configurable total budget (default ~2 minutes).
- If delivery still fails, or the proxy restarts mid-flight, the decision is not lost
  forever: the consent-server keeps the record, and the proxy catches up on reconnect.
- Because retries and reconnect catch-up can both happen, **your endpoint may occasionally
  receive the same decision more than once.** This is normal and expected for at-least-once
  delivery.

### Handling duplicates on your side
Every delivery carries two headers your endpoint can use to recognize repeats:

- `x-event-id` — the stream position of the decision.
- `x-callback-id` — a stable, unique id for that specific decision that is **identical on
  every retry and replay**.

Your endpoint should treat delivery as **idempotent**: if it sees an `x-callback-id` it has
already processed, it can safely acknowledge and ignore it. This is the recommended way to
make your integration duplicate-proof.

> **Recovery path:** if your endpoint was down long enough to miss a decision entirely, your
> application can always read the current state directly from the consent-server status API
> (`GET /consent-requests/:id/status`). The proxy is a fast-path notifier, not the system of
> record.

---

## 6. What you need to provide

To run the proxy, you supply a small amount of configuration and one credential. All of it
is provided to the proxy through a config file and environment variables; secrets are read
from mounted files or environment variables and are **never written to logs**.

| What | Example | Purpose |
|---|---|---|
| Consent-server base URL | `https://consent.scrambleid.com/orgs/<org>/api/v1` | Where the proxy connects out to. Provided by ScrambleID. |
| Tenant label | `your-tenant` | Identifies your organization. Provided by ScrambleID. |
| Upstream credential | the private key of the proxy's service account | How the proxy authenticates to the consent-server. **You create the service account** through the ScrambleID M2M flow. |
| Callback allowlist | hosts + IP ranges (CIDRs) | The internal destinations the proxy is permitted to deliver to. **You define this.** |
| Callback endpoint | `https://workflow.internal/consent/outcome` | Your application endpoint that receives decisions. **You run this.** |
| Receiver origin | `https://workflow.internal` | The exact origin the proxy may attach its bearer token to (`CALLBACK_AUTH_ORIGIN`). Required only when you configure a dedicated callback credential. **You define this.** |
| Callback credential (optional) | a dedicated client id + key/secret | Mints the token attached to deliveries. Only when you enable callback authentication (`privateKeyJwt`/`clientCredentials`). Issued by a ScrambleID **or** your own auth server. |
| JWKS URL (for your endpoint) | from the issuing auth server's OIDC discovery | Your endpoint uses this to validate the callback token. ScrambleID provides it for ScrambleID-issued tokens; for your-own-auth tokens it is your issuer's JWKS. |

### The callback allowlist (your main control)
This is the most important setting you own. It has three parts:

- **Allowed hosts** — which hostnames the proxy may deliver to (exact names, or a suffix
  like `.internal` to permit a whole internal domain).
- **Allowed IP ranges (CIDRs)** — the resolved address of the destination must fall inside
  these ranges (for example, your private `10.0.0.0/8` space).
- **HTTPS requirement** — on by default; the proxy refuses plaintext destinations.

Regardless of what you allowlist, the proxy **always** blocks delivery to loopback,
link-local, and cloud-metadata addresses, and it **never follows redirects**. This is a
deliberate anti-SSRF (server-side request forgery) safeguard so the relay cannot be tricked
into reaching somewhere it shouldn't.

### Authenticating the delivery to your endpoint
When you configure a dedicated callback credential (see the options below), each delivery
carries an `Authorization: Bearer <token>` header; your endpoint validates that token to
confirm the request genuinely came from the proxy, then proceeds. With no credential
configured, deliveries carry no bearer and you must secure your endpoint another way.

You choose, per deployment, **how that callback token is obtained** — one setting with three
options:

- **`none` (the default).** **No callback authentication:** the proxy attaches **no**
  `Authorization` header to deliveries. It does **not** reuse its upstream ScrambleID token —
  that token authenticates the proxy to ScrambleID, and reusing it on a delivery to your
  receiver would carry the wrong audience, grant excessive scope, and create an unintended
  trust between ScrambleID and your receiver. If you leave this at `none` you must secure your
  receiver by other means (e.g. network controls or mTLS). If you set a receiver origin
  without choosing a mechanism, the proxy refuses to start (see below).
- **`privateKeyJwt`.** The proxy mints a **dedicated** callback token using a private key you
  hold. There is no shared secret: the proxy signs a short-lived assertion with the key and
  exchanges it for a token at an auth server.
- **`clientCredentials`.** The proxy mints a **dedicated** callback token using a client id
  and a client secret you provide, exchanged at an auth server.

For the two dedicated options, the token may be issued by a **ScrambleID auth server or your
own auth server** — you point the proxy at whichever token endpoint you want, and you choose
the scope and audience the token should carry. This lets the callback credential be fully
separate from the upstream ScrambleID credential if your security model calls for it. The
dedicated callback credential is also **distinct from the upstream credential** the proxy uses
to connect to ScrambleID — a different key or secret, potentially at a different auth server,
and not to be reused for the other.

Both dedicated options use the same machine-to-machine flow; they differ only in **how the
proxy proves its identity** when fetching a token — a signed assertion from a private key
(`privateKeyJwt`) or a shared client secret (`clientCredentials`). They are not separate
"grants"; they are two authentication methods for the same flow.

Rather than configuring the token endpoint by hand, you may instead give the proxy your auth
server's **OIDC discovery URL** (its `.well-known/openid-configuration`) and it will read the
token endpoint from there automatically. An explicitly configured token endpoint always takes
precedence if you provide both.

For the two dedicated options:
- The token is short-lived and rotates automatically.
- The token is attached **only** to the one receiver origin you designate, never to any other
  allowlisted host.
- With `privateKeyJwt` there is **no shared secret** to exchange; `clientCredentials` uses a
  client secret you hold and mount as a file.
- The upstream ScrambleID connection token is **never** used for callbacks.

What you need on your side: validate the bearer token your receiver gets — verify the
signature against the issuer's **JWKS URL** (for ScrambleID-issued tokens, the one we
provide; for your-own-auth-server tokens, your issuer's JWKS) and check the standard claims
(issuer, audience, expiry, and any required scopes), then accept the request. You obtain the
expected issuer and JWKS from the auth server that issues the callback token — from its OIDC
discovery document. The token's **audience** is whatever your receiver expects; for the
dedicated options you can have the auth server stamp that audience into the token.

---

## 7. Security posture

The proxy is designed to be a low-risk component to host:

- **Outbound-only.** No inbound firewall rules needed for its core function. It initiates
  the one connection to ScrambleID.
- **Authenticated connection.** The link to the consent-server is authenticated with a
  short-lived token; the proxy transparently refreshes/reconnects when it expires. Optional
  mutual TLS (mTLS) is supported for the upstream link.
- **Least privilege on delivery.** Every delivery is re-validated against your allowlist and
  the SSRF guard *before* each send — not just once at startup.
- **Verifiable delivery auth, no shared secret.** Deliveries are authenticated **only when
  you configure a dedicated callback credential** — a short-lived bearer token minted from
  that credential, attached only to your configured receiver origin, which your endpoint
  verifies against the JWKS URL. The proxy never reuses its ScrambleID connection token for
  deliveries. Without a dedicated credential, no bearer is attached (secure your receiver by
  other means, e.g. network controls or mTLS).
- **No secrets in transit or logs.** Decision messages carry no secrets. The proxy never
  logs tokens, credentials, or the bodies of your callbacks. Secret-shaped fields are
  redacted.
- **Hardened runtime.** The shipped container runs as a non-root user with a read-only
  filesystem, all Linux capabilities dropped, `no-new-privileges`, and no persistent volume.
- **Enterprise-friendly egress.** It supports a corporate forward proxy (`HTTPS_PROXY` /
  `NO_PROXY`) and a custom CA bundle (`NODE_EXTRA_CA_CERTS`) for environments that
  intercept or inspect TLS.

**Network summary:** allow outbound HTTPS from the proxy to the ScrambleID consent-server
URL, and allow the proxy to reach your internal callback endpoint(s). Nothing else.

---

## 8. Running and operating it

The proxy ships as a Docker container and is operated like any small stateless service.

### Deployment model
- One container per tenant.
- Stateless, so you can run more than one replica for availability. Both replicas receive
  the same decisions; your endpoint's idempotency handling (see §5) absorbs the duplicates.
- Nothing to back up.

### Health checks
- `GET /health/live` — the process is running.
- `GET /health/ready` — returns `200` when the stream to ScrambleID is connected, `503`
  otherwise. Brief reconnects can flip this momentarily; it self-heals, so don't restart on a
  short blip.

### Metrics
A `GET /metrics` endpoint exposes Prometheus-style counters you can scrape and alert on,
including:

- `sse_connected` — is the upstream stream currently connected (1/0).
- `outcomes_received_total` — decisions received from ScrambleID.
- `callbacks_delivered_total` / `callbacks_failed_total` — delivery outcomes.
- `callback_latency_ms` — delivery timing.
- `sse_reconnects_total` — stream reconnect churn.

**Suggested alerts:** `sse_connected == 0` sustained; a rising `callbacks_failed_total`; or
excessive reconnect churn.

### Graceful shutdown
On a stop signal, the proxy stops accepting new work, finishes any in-flight delivery within
a grace period, and exits cleanly.

---

## 9. Common questions

**Does the proxy see our user data?**
No. Decision messages contain identifiers and status, not user content or secrets.

**Can ScrambleID reach into our network through it?**
No. The proxy makes an outbound connection and only ever delivers to destinations you
allowlisted. There is no inbound path to your systems through it.

**What happens if our callback endpoint is down for a while?**
The proxy retries briefly, then moves on. The decision is retained by ScrambleID; your
application can reconcile via the status API, and the proxy catches up on reconnect. You may
see the decision delivered later or more than once — handle it idempotently.

**What if the proxy itself crashes or is redeployed?**
It holds no state, so it simply reconnects and resumes. Any decisions missed while it was
down are replayed by the consent-server after it reconnects.

**Will our endpoint ever get duplicate deliveries?**
Possibly, by design (at-least-once). Deduplicate using the `x-callback-id` header.

**How does our endpoint know the delivery is genuinely from the proxy?**
When you configure a **dedicated callback credential** (`privateKeyJwt` or
`clientCredentials`), each delivery carries an `Authorization: Bearer <token>` header holding
a short-lived token minted from that credential. Validate it against the issuing auth server's
JWKS URL (signature + standard claims: issuer, audience, expiry, scopes). The proxy never
reuses its upstream ScrambleID token for deliveries. If you configure no mechanism, no bearer
is attached and you must authenticate the delivery another way (e.g. network controls/mTLS).

**What ports does it need open?**
Outbound HTTPS to the ScrambleID consent-server, and network reach to your internal callback
endpoint. No inbound exposure is required for its core function (health/metrics are local).

---

## 10. Quick glossary

- **SSE (Server-Sent Events):** a standard, long-lived, one-way HTTP stream the server uses
  to push messages to the proxy.
- **Callback URL / endpoint:** your internal application address that receives decisions.
- **Allowlist:** the explicit set of hosts and IP ranges the proxy is permitted to deliver
  to.
- **Idempotent:** safe to receive the same message more than once without double-processing.
- **SSRF guard:** protection that stops the relay from being tricked into reaching internal
  or sensitive addresses it shouldn't.
- **M2M (machine-to-machine) token:** a short-lived access token used to prove a service's
  identity. The proxy uses one **upstream** to authenticate to ScrambleID; a **separate**,
  dedicated callback token (when configured) authenticates deliveries to your endpoint. The
  upstream token is never reused for callbacks.
- **JWKS URL:** a standard, public endpoint listing the signing keys your endpoint uses to
  verify the bearer token on each delivery.
- **Tenant:** your organization's identity within ScrambleID.

---

*For deployment and day-to-day operations see `customer-deployment.md`; for every setting see
`configuration.md`; when something is wrong see `customer-troubleshooting.md`.*
