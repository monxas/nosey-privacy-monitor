# Changelog

All notable changes documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning is [SemVer](https://semver.org/).

## [Unreleased]

## [0.1.0] - 2026-05-17

### Added
- First public release.
- `bin/nosey` CLI: list / add / enable / disable / apply / status / discover / wizard.
- Pi-hole 6 backend support.
- Standalone (no observability) and observability (Loki/Grafana) modes.
- Curated templates: `lg-tv`, `iphone-ios`, `echo-dot`, `samsung-tv`, `roku-tv`, `doh-doq-killer`.
- Setup guides for Proxmox LXC and Raspberry Pi.
- Playbooks for LG TV and iPhone end-to-end.
- `docs/what-i-found.md` — real observed trackers.
- `docs/limitations.md` — honest description of what Nosey does NOT do.
- Consent enforcement (`consent_doc:` required for non-self devices).
- Bootstrap installer (`scripts/install.sh`).
- GitHub Actions lint + smoke test.

### Known limitations
- AdGuard Home backend is a stub (PRs welcome).
- WireGuard routing mode is registered but not yet implemented (Phase 2).
- Auto-application of blocklist domains to Pi-hole groups not yet automated — `nosey apply` creates the group and assigns the client, but you still need to add the template's domains via Pi-hole UI manually (or scripted via Pi-hole's own API in the meantime).
