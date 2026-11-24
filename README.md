# ScrambleID Agentic Consent Proxies

Everything you need to deploy ScrambleID's consent proxies in your own network. For the consent and
decision APIs themselves, see the
[ScrambleID Consent documentation](https://consent.scrambleid.com/docs).

## Edge proxy

The ScrambleID Edge Proxy. It's optional. One container, one outbound connection. It answers your
gateway's decisions inside your network and delivers consent outcomes to your callback endpoint.
Turn on either job or both. It replaces the Policy Decision Proxy and the Consent Notification Proxy.

Latest version: **1.2.0**. Guides for that version:

| Guide | Read it if |
|---|---|
| [Overview](edge-proxy/releases/1.2.0/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](edge-proxy/releases/1.2.0/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](edge-proxy/releases/1.2.0/docs/configuration.md) | You need every setting. |
| [Troubleshooting](edge-proxy/releases/1.2.0/docs/customer-troubleshooting.md) | Something isn't working. |
| [GCP Agent Gateway](edge-proxy/releases/1.2.0/docs/customer-gcp-agent-gateway.md) | You use GCP Agent Gateway. |

## Archived proxies

The Consent Notification Proxy (callback proxy, last version 1.0.4) is archived. The edge proxy with
`callbacks` on does its job. Its guides stay in [archive/](archive/) for deployments that still run it.

## What's here

```
README.md                  this page
CHANGELOG.md               what changed in each release of this repo
edge-proxy/releases/<version>/
  docs/                    guides for that version
  deploy/                  compose file and .env template
  SHA256SUMS, sbom.spdx.json   (up to 1.0.1; later versions have them on the release page)
archive/                   proxies no longer developed (callback proxy)
LICENSE                    terms of use
```

Each version has its own folder, so the guides and templates always match that version's image.
Use the folder of the version you run. A published version's folder does not change.

## Getting the image

The edge proxy image is a file, not a registry pull. Each version has a
[release page](https://github.com/scramble-ext/agentic-consent-proxies/releases) (`edge-proxy-v<version>`)
with the image tarball, `SHA256SUMS` and the SBOM. Download, check and load them as the version's
deployment runbook says.

The archived callback proxy's image stays on GitHub Container Registry
(`ghcr.io/scramble-ext/consent-proxy`, public, no login).

## Staying up to date

Watch this repo's releases. [CHANGELOG.md](CHANGELOG.md) lists each release.

## Support

Contact your ScrambleID representative.
