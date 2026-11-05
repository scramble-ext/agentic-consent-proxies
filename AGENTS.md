# AGENTS.md

## Overview

Everything a customer needs to deploy ScrambleID's two consent proxies, shared by all customers.

## Current Status

Holds the callback proxy (Consent Notification Proxy) 1.0.4 guides, templates, checksums and SBOM. The policy evaluation proxy isn't added yet.

## Key Decisions

- One repo for all customers, so nothing customer-specific goes here.
- This repo is about deploying the proxy. Consent and decision API docs stay on the consent server at `/docs`, the one source of truth, and are linked, never copied.
- Images live in public GHCR packages under `scramble-ext`, so customers need no credentials. Never in this repo or the internal registry.
- One top-level folder per proxy: `callback-proxy/` and, later, `policy-evaluation-proxy/`. Each has `docs/`, `deploy/` and `releases/`.

## Architecture

Docs and templates only. The private source repos stay the source of truth. This repo gets copies.

## Recent Changes

- First edition of the repo, with the Consent Notification Proxy 1.0.4.

## Next Steps

- Add `policy-evaluation-proxy/` when it ships.

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
- `LICENSE`: terms of use.

## Environment Setup

None. There is nothing to build or run.
