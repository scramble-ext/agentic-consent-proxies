# Changelog

## Unreleased

- Archived the Consent Notification Proxy (callback proxy): its folder moved to
  `archive/callback-proxy/`, unchanged. The edge proxy with `callbacks` on replaces it.
- ScrambleID Edge Proxy 1.0.1: same features and settings as 1.0.0. The offline image tarball and the
  registry image are now the same image, with one digest in `SHA256SUMS`. Use 1.0.1 rather than
  1.0.0, whose tarball and registry image were built separately. The 1.0.1 SBOM will be added here
  shortly.
- Each version now has its own folder with its guides, templates, checksums and SBOM:
  `edge-proxy/releases/<version>/` and `callback-proxy/releases/<version>/`. The top-level `docs/`
  and `deploy/` folders are gone. Use the folder of the version you run. The deployment runbooks
  (edge proxy 1.0.0, callback proxy 1.0.4) now point to these paths; nothing else in them changed.
- Edge proxy 1.0.0: `SHA256SUMS` now also lists the checksum of the offline image tarball.
- ScrambleID Edge Proxy 1.0.0: guides, deployment templates, checksums and SBOM. It does the work of
  both the decision proxy and the callback proxy.
- Consent Notification Proxy 1.0.4: guides, deployment templates, checksums and SBOM.
- Removed the Policy Decision Proxy. The edge proxy replaces it.
