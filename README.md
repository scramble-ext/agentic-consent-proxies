# ScrambleID Agentic Consent Proxies

Everything you need to deploy ScrambleID's consent proxies in your own network. For the consent and
decision APIs themselves, see the
[ScrambleID Consent documentation](https://consent.scrambleid.com/docs).

## Edge proxy

The ScrambleID Edge Proxy. It's optional. One container, one outbound connection. It answers your
gateway's decisions inside your network and delivers consent outcomes to your callback endpoint.
Turn on either job or both. It replaces the Policy Decision Proxy and the Consent Notification Proxy.

| Guide | Read it if |
|---|---|
| [Overview](edge-proxy/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](edge-proxy/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](edge-proxy/docs/configuration.md) | You need every setting. |
| [Troubleshooting](edge-proxy/docs/customer-troubleshooting.md) | Something isn't working. |

## Callback proxy

The Consent Notification Proxy. It's optional. It receives consent outcomes over one outbound
connection and delivers them to your callback endpoint. The edge proxy does this job too.

| Guide | Read it if |
|---|---|
| [Overview](callback-proxy/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](callback-proxy/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](callback-proxy/docs/configuration.md) | You need every setting. |
| [Troubleshooting](callback-proxy/docs/customer-troubleshooting.md) | Something isn't working. |

## What's here

```
README.md                  this page
CHANGELOG.md               what changed in each release of this repo
edge-proxy/docs/           guides
edge-proxy/deploy/         compose file and .env template
edge-proxy/releases/       checksums and SBOM per version
callback-proxy/docs/       guides
callback-proxy/deploy/     compose file, .env and config templates
callback-proxy/releases/   checksums and SBOM per version
LICENSE                    terms of use
```

## Container images

Images are not stored in this repo. They are public on GitHub Container Registry, so no login is
needed:

- edge proxy: `ghcr.io/scramble-ext/scrambleid-edge-proxy:<version>`
- callback proxy: `ghcr.io/scramble-ext/consent-proxy:<version>`

## Staying up to date

Watch this repo's releases. [CHANGELOG.md](CHANGELOG.md) lists each release.

## Support

Contact your ScrambleID representative.
