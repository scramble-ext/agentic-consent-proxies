# ScrambleID Edge Proxy — GCP Agent Gateway

**Audience:** the team connecting GCP Agent Gateway to the Edge Proxy. You have already followed the
[deployment runbook](customer-deployment.md) (service account, key, image).

With this set up, every MCP `tools/call` your agents make through the gateway is checked against
your ScrambleID policies before it reaches the MCP server. A call that needs a person's approval
waits for it. For how to do each GCP step, follow the Google documentation linked below.

---

## 1. What you need

| What | Why |
|---|---|
| An **egress Agent Gateway** (Google managed, `AGENT_TO_ANYWHERE`) that your agents' MCP traffic goes through | Where each tool call is checked. [Set up Agent Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/set-up-agent-gateway) |
| The Edge Proxy on a **private address**, reachable from the gateway on **port 443 with TLS** | The gateway calls the proxy over private connectivity into your VPC (network attachment and DNS peering). A public address does not work: the gateway fails the call before it reaches the proxy |
| Your firewall allows **only the gateway** to reach port 443 | The gateway sends no caller identity on this call, so the network is the limit |
| Outbound HTTPS from the proxy to ScrambleID | Policy sync and consent token keys |
| Your MCP servers set up as services in ScrambleID, with policies | What is decided. Name each one by the URL your agents call, e.g. `https://mcp.acme.com/mcp/pay` |

---

## 2. Edge Proxy settings

Make a TLS key and certificate. Agent Gateway does not check the certificate, so a self-signed one is enough:

```bash
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -days 365 \
  -keyout gateway-tls.key -out gateway-tls.crt \
  -subj "/CN=edge-proxy.internal" -addext "subjectAltName=DNS:edge-proxy.internal"
```

Mount both next to your other secrets (readable by uid 1000 only) and add to `.env`:

```
EDGE_PROXY_FEATURES=decisions
EDGE_PROXY_ADAPTERS=ext_proc
EDGE_PROXY_ADAPTER_TLS_CERT_FILE=/run/edge-proxy-secrets/gateway-tls.crt
EDGE_PROXY_ADAPTER_TLS_KEY_FILE=/run/edge-proxy-secrets/gateway-tls.key
EDGE_PROXY_GRPC_HOST_PORT=443
```

Then, in `docker-compose.yml`, uncomment the gRPC `ports` line. `EDGE_PROXY_BIND_IP` must be the
private address the gateway reaches. `EDGE_PROXY_JWKS_URL` is still required with `decisions`, though
this path does not use `POST /decisions`. Every setting is in
[configuration.md](configuration.md#gateway-adapters-decisions-optional).

**Ready when** `/health/status` shows `state: fresh`, `consentTokens.loaded: true` and
`adapters.on: ["ext_proc"]`.

---

## 3. Connect Agent Gateway

Register the proxy as an **authorization extension** and attach it with an **authorization policy**:
[Delegate authorization with Service Extensions](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/delegate-authorization).
The values that matter:

| Resource | Field | Value |
|---|---|---|
| Authorization extension | `service` | The proxy's private DNS name, e.g. `edge-proxy.acme.internal` (a Cloud DNS private zone on the VPC) |
| | `timeout` | `1s` |
| | `failOpen` | `false` (the default): if the proxy cannot be reached, calls are refused |
| Authorization policy | `policyProfile` | **`CONTENT_AUTHZ`**. `REQUEST_AUTHZ` sends no body, so the proxy cannot see the tool |
| | `action` | `CUSTOM`, with the extension as `customProvider.authzExtension` |
| | `target` | Your Agent Gateway |

Attach it to your **MCP routes only**. A large non-MCP JSON request sent to the proxy is refused (`413`).
Do not rewrite the host or path before the check: the URL the gateway sees is the service.

---

## 4. Your agent

A refused call comes back from the gateway as a JSON-RPC error. Branch on `error.data.decision`:

| `decision` | What the agent does |
|---|---|
| `consent_required` | Ask ScrambleID for consent for the service, action and parameters in `error.data`, wait for approval, then send **the same call** again with the consent token in `x-consent-token`. |
| `deny` | Stop. No consent turns a deny into allow. `reason: unavailable` (503) means the proxy has no fresh policies: retry later. |

Do not branch on `error.code`: other MCP servers use `-32001` and `-32002` too. The token is
accepted for exactly those arguments, for up to 15 minutes. How to create the request and wait for
it: "Decision and Consent" in the [consent guide](https://consent.scrambleid.com/docs?guide=decision-and-consent).

---

## 5. Check it

Make three calls through the gateway: a tool your policy allows, one it denies, one that needs approval.

```bash
curl -s http://127.0.0.1:8080/health/status | jq '{state, adapters, consentTokens}'
docker logs edge-proxy | grep "adapter decision" | tail -20
```

Each call is one `adapter decision` line (tool, service, action, outcome, reason), never the
arguments or the token. The approved request also shows on your ScrambleID dashboard.

---

## 6. If it fails

| Symptom | Cause and fix |
|---|---|
| Calls fail and there is no `adapter decision` line | The gateway does not reach the proxy: not a private address, wrong DNS name in the extension, firewall to 443, or TLS missing. A public address makes the gateway answer gRPC `INTERNAL`. |
| `reason: body_missing` (411) | The policy uses `REQUEST_AUTHZ`. Use `CONTENT_AUTHZ`. |

Anything else: [customer-troubleshooting.md](customer-troubleshooting.md#3a-gateway-adapters).
