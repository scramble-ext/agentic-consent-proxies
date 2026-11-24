# AGENTS.md

## Overview

Everything a customer needs to deploy ScrambleID's consent proxies, shared by all customers.

## Current Status

Holds the Edge Proxy 1.2.0 (and 1.0.1, 1.0.0) and, archived, the callback proxy (Consent Notification Proxy) 1.0.4: guides,
templates, checksums and SBOM.

## Key Decisions

- One repo for all customers, so nothing customer-specific goes here.
- This repo is about deploying the proxy. Consent and decision API docs stay on the consent server at `/docs`, the one source of truth, and are linked, never copied.
- Edge proxy images ship as a tarball on each version's public release (`edge-proxy-v<version>`): tarball, `SHA256SUMS`, SBOM and the per-org delivery bundle zip. Customers cannot pull from a registry. Never put an image in this repo or the internal registry. Versions up to 1.2.0 are also on public GHCR, and the archived callback proxy stays there.
- One top-level folder per maintained proxy (`edge-proxy/`). Proxies no longer developed move, unchanged, to `archive/<proxy>/`; `archive/README.md` lists them. Everything for a version is in `releases/<version>/`: `docs/`, `deploy/`, `SHA256SUMS` and SBOM. Guides and templates change between versions, so a customer reads the ones for the image they run.
- A published version's folder is frozen. A new release gets a new folder (copy the previous one, then apply that release's changes). Only wording fixes may go into a published folder, noted in `CHANGELOG.md`.
- From 1.2.0, `SHA256SUMS` and the SBOM are not in the version folder. Releases made by the workflow carry them as release assets. Never copy hashes into the docs or a version folder.
- Customer guides cover only gateways that have been run (GCP Agent Gateway). Add another gateway only after testing it.
- The README links to the latest version of each proxy. The consent-server release workflow (`release-edge-proxy.yml`) pushes the new version folder to `main`, updates those links and `CHANGELOG.md`, then creates the release. Do not do it by hand.
- The edge proxy replaces the Policy Decision Proxy, which is removed. Its image is `scrambleid-edge-proxy` and its settings are `EDGE_PROXY_*`.

## Architecture

Docs and templates only. The private source repos stay the source of truth. This repo gets copies.

## Recent Changes

- Images now ship as a tarball on each version's release page, no registry. README "Container images" became "Getting the image".
- Added Edge Proxy 1.2.0: GCP Agent Gateway guide, no `SHA256SUMS` or SBOM in the folder (see Key Decisions).
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
- `edge-proxy/releases/<version>/`: `docs/` (guides), `deploy/` (compose file and `.env` template), `SHA256SUMS` and SBOM up to 1.0.1 (later versions: release assets), copied by the release workflow.
- `archive/callback-proxy/releases/1.0.4/`: the archived callback proxy, same layout (`deploy/` also has the config template). Frozen.
- `LICENSE`: terms of use.

## Environment Setup

None. There is nothing to build or run.
