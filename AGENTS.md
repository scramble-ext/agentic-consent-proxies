# AGENTS.md

## Overview

Everything a customer needs to deploy ScrambleID's two consent proxies, shared by all customers.

## Current Status

Holds the callback proxy (Consent Notification Proxy) 1.0.4 and the decision proxy (Policy Decision Proxy) 1.0.0: guides, templates, checksums and SBOM.

## Key Decisions

- One repo for all customers, so nothing customer-specific goes here.
- This repo is about deploying the proxy. Consent and decision API docs stay on the consent server at `/docs`, the one source of truth, and are linked, never copied.
- Images live in public GHCR packages under `scramble-ext`, so customers need no credentials. Never in this repo or the internal registry.
- One top-level folder per proxy: `callback-proxy/` and `decision-proxy/`. Each has `docs/`, `deploy/` and `releases/`.
- Customers call the second one the Policy Decision Proxy. Its image is `policy-decision-proxy` and its settings are `DECISION_PROXY_*`.

## Architecture

Docs and templates only. The private source repos stay the source of truth. This repo gets copies.

## Recent Changes

- First edition of the repo, with the Consent Notification Proxy 1.0.4.
- Added the Policy Decision Proxy 1.0.0.

## Next Steps

- None.

## Notes for Next Agent

- Everything here is read by customers. Keep out internal hostnames, internal repo names, tokens,
  customer names and incident notes.
- Edit proxy guides and templates in the proxy's source repo and copy them here. Do not edit them only here.
- Templates stay generic. Tenant-specific bundles go to that customer, not here.
- Use LF line endings.

## File Structure

- `README.md`: landing page.
- `CHANGELOG.md`: repo releases.
- `callback-proxy/docs/`: guides.
- `callback-proxy/deploy/`: compose file, `.env` and config templates.
- `callback-proxy/releases/<version>/`: `SHA256SUMS` and SBOM, copied from the release.
- `decision-proxy/docs/`, `decision-proxy/deploy/` (compose file and `.env` template), `decision-proxy/releases/<version>/`: the same for the decision proxy.
- `LICENSE`: terms of use.

## Environment Setup

None. There is nothing to build or run.
