# AGENTS.md

## Overview

Everything a customer needs to deploy ScrambleID's consent proxies, shared by all customers.

## Current Status

Holds the Edge Proxy 1.0.1 (and 1.0.0) and, archived, the callback proxy (Consent Notification Proxy) 1.0.4: guides,
templates, checksums and SBOM.

## Key Decisions

- One repo for all customers, so nothing customer-specific goes here.
- This repo is about deploying the proxy. Consent and decision API docs stay on the consent server at `/docs`, the one source of truth, and are linked, never copied.
- Images live in public GHCR packages under `scramble-ext`, so customers need no credentials. Never in this repo or the internal registry.
- One top-level folder per maintained proxy (`edge-proxy/`). Proxies no longer developed move, unchanged, to `archive/<proxy>/`; `archive/README.md` lists them. Everything for a version is in `releases/<version>/`: `docs/`, `deploy/`, `SHA256SUMS` and SBOM. Guides and templates change between versions, so a customer reads the ones for the image they run.
- A published version's folder is frozen. A new release gets a new folder (copy the previous one, then apply that release's changes). Only wording fixes may go into a published folder, noted in `CHANGELOG.md`.
- The README links to the latest version of each proxy. Update those links when releasing.
- The edge proxy replaces the Policy Decision Proxy, which is removed. Its image is `scrambleid-edge-proxy` and its settings are `EDGE_PROXY_*`.

## Architecture

Docs and templates only. The private source repos stay the source of truth. This repo gets copies.

## Recent Changes

- Archived the callback proxy (`archive/callback-proxy/`).
- Added Edge Proxy 1.0.1. Its SBOM is not here yet: the release SBOM names the image by an internal registry address, so it waits for one generated against the GHCR name.
- Moved guides and templates into per-version folders (`releases/<version>/`).
- Added the Edge Proxy 1.0.0 (decisions and callbacks in one image).
- Removed the Policy Decision Proxy.

## Next Steps

- Add `edge-proxy/releases/1.0.1/sbom.spdx.json` once one exists that names the image by its GHCR address.

## Notes for Next Agent

- Everything here is read by customers. Keep out internal hostnames, internal repo names, tokens,
  customer names and incident notes.
- Edit proxy guides and templates in the proxy's source repo and copy them here. Do not edit them only here.
- Templates stay generic. Tenant-specific bundles go to that customer, not here.
- Use LF line endings.

## File Structure

- `README.md`: landing page.
- `CHANGELOG.md`: repo releases.
- `edge-proxy/releases/<version>/`: `docs/` (guides), `deploy/` (compose file and `.env` template), `SHA256SUMS` (offline tarball checksum, image digest) and SBOM, copied from the release.
- `archive/callback-proxy/releases/1.0.4/`: the archived callback proxy, same layout (`deploy/` also has the config template). Frozen.
- `LICENSE`: terms of use.

## Environment Setup

None. There is nothing to build or run.
