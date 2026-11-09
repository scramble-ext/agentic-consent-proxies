# ScrambleID Agentic Consent Proxies

Everything you need to deploy ScrambleID's consent proxies in your own network. For the consent and
decision APIs themselves, see the
[ScrambleID Consent documentation](https://consent.scrambleid.com/docs).

## Callback proxy

The Consent Notification Proxy. It's optional. It receives consent outcomes over one outbound
connection and delivers them to your callback endpoint.

| Guide | Read it if |
|---|---|
| [Overview](callback-proxy/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](callback-proxy/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](callback-proxy/docs/configuration.md) | You need every setting. |
| [Troubleshooting](callback-proxy/docs/customer-troubleshooting.md) | Something isn't working. |

## Decision proxy

The Policy Decision Proxy. It's optional. It answers your gateway's decisions inside your
network, from a copy of your policies, so a call does not leave your network to be decided.

| Guide | Read it if |
|---|---|
| [Overview](decision-proxy/docs/customer-overview.md) | You're evaluating the proxy or reviewing its security. |
| [Deployment runbook](decision-proxy/docs/customer-deployment.md) | You're installing or operating it. |
| [Configuration](decision-proxy/docs/configuration.md) | You need every setting. |
| [Troubleshooting](decision-proxy/docs/customer-troubleshooting.md) | Something isn't working. |

## What's here

```
README.md                  this page
CHANGELOG.md               what changed in each release of this repo
callback-proxy/docs/       guides
callback-proxy/deploy/     compose file, .env and config templates
callback-proxy/releases/   checksums and SBOM per version
decision-proxy/docs/       guides
decision-proxy/deploy/     compose file and .env template
decision-proxy/releases/   checksums and SBOM per version
LICENSE                    terms of use
```

## Container images

Images are not stored in this repo. They are public on GitHub Container Registry, so no login is
needed:

- callback proxy: `ghcr.io/scramble-ext/consent-proxy:<version>`
- decision proxy: `ghcr.io/scramble-ext/policy-decision-proxy:<version>`

## Staying up to date

Watch this repo's releases. [CHANGELOG.md](CHANGELOG.md) lists each release.

## Support

Contact your ScrambleID representative.
