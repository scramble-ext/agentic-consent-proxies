# ScrambleID Agentic Consent Proxies

Everything you need to deploy ScrambleID's consent proxies in your own network. For the consent and
decision APIs themselves, see the
[ScrambleID Consent documentation](https://consent.scrambleid.com/docs).

## Edge proxy

The ScrambleID Edge Proxy. It's optional. One container, one outbound connection. It answers your
gateway's decisions inside your network and delivers consent outcomes to your callback endpoint.
Turn on either job or both. It replaces the Policy Decision Proxy and the Consent Notification Proxy.

Latest version: **1.0.1**. Guides for that version:

| Guide | Read it if |
|---|---|
| [Overview](edge-proxy/releases/1.0.1/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](edge-proxy/releases/1.0.1/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](edge-proxy/releases/1.0.1/docs/configuration.md) | You need every setting. |
| [Troubleshooting](edge-proxy/releases/1.0.1/docs/customer-troubleshooting.md) | Something isn't working. |

## Callback proxy

The Consent Notification Proxy. It's optional. It receives consent outcomes over one outbound
connection and delivers them to your callback endpoint. The edge proxy does this job too.

Latest version: **1.0.4**. Guides for that version:

| Guide | Read it if |
|---|---|
| [Overview](callback-proxy/releases/1.0.4/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](callback-proxy/releases/1.0.4/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](callback-proxy/releases/1.0.4/docs/configuration.md) | You need every setting. |
| [Troubleshooting](callback-proxy/releases/1.0.4/docs/customer-troubleshooting.md) | Something isn't working. |

## What's here

```
README.md                  this page
CHANGELOG.md               what changed in each release of this repo
edge-proxy/releases/<version>/
  docs/                    guides for that version
  deploy/                  compose file and .env template
  SHA256SUMS, sbom.spdx.json
callback-proxy/releases/<version>/
  docs/                    guides for that version
  deploy/                  compose file, .env and config templates
  SHA256SUMS, sbom.spdx.json
LICENSE                    terms of use
```

Each version has its own folder, so the guides and templates always match that version's image.
Use the folder of the version you run. A published version's folder does not change.

## Container images

Images are not stored in this repo. They are public on GitHub Container Registry, so no login is
needed:

- edge proxy: `ghcr.io/scramble-ext/scrambleid-edge-proxy:<version>`
- callback proxy: `ghcr.io/scramble-ext/consent-proxy:<version>`

## Staying up to date

Watch this repo's releases. [CHANGELOG.md](CHANGELOG.md) lists each release.

## Support

Contact your ScrambleID representative.
