# Changelog

All notable changes documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning is [SemVer](https://semver.org/).

## [Unreleased]

## [0.3.0] - 2026-05-17

### Added
- **`nosey capture <device>`** — passive forensic mode (no MITM, no custom CA).
  Pulls Zeek logs from the L3 gateway (via a configurable shell command),
  filters on the device's IP, and emits a markdown report with:
  - Summary: connection / TLS / QUIC / plaintext-HTTP counts, bytes up/down,
    unique destinations and SNIs.
  - Top 25 SNIs the device asked for.
  - Top 15 destination IPs (with bytes, packets, ports, protocols, SNIs seen).
  - TLS posture: versions, ciphers, key exchange curves
    (flags `X25519MLKEM768` as iOS 18+ / macOS 15+ post-quantum hybrid).
  - Cadence per minute (heartbeat detection) + connection duration percentiles.
  - QUIC / HTTP/3 top SNIs.
  - Plaintext HTTP requests (anything here = a leaky endpoint).
- `gateway.zeek_fetch_cmd` config option — shell command template with `{log}`
  placeholder. Works with direct SSH, `pct exec` (Proxmox), `kubectl exec`, etc.
- `docs/playbooks/forensic-capture.md` would be a natural follow-up.

### Notes
- Capture mode is **complementary to MITM**: gives ~90% of forensic value
  with zero device cooperation (no CA install, no certificate pinning issues).
- Designed for the case "I want to see what the iPhone is *actually* doing
  in the next 30 minutes without installing anything on the iPhone."

## [0.2.0] - 2026-05-17

### Added
- **Pi-hole bridge**: extended `PiholeBackend` with v6 REST API methods
  (`stats_summary`, `top_domains`, `top_clients`, `query_history`,
  `list_domains`, `add_domain`, `remove_domain`).
- **`nosey serve`** subcommand — local web UI (FastAPI + HTMX, single file, no build step):
  - Live Pi-hole stats dashboard (24h queries / blocked / clients / block rate).
  - Per-device card with query + block counts pulled from Pi-hole client stats.
  - Toggle enable/disable from the UI (consent enforcement applies).
  - **Template diff modal**: shows which domains would be added/removed/kept
    in the Pi-hole group, with `tier`, `why_block`, and `breaks_if_blocked`
    reasoning per entry. Confirm → pushes changes via API.
- Helper `diff_template_vs_pihole()` used by both UI and future automation.
- README: dedicated "Pi-hole bridge" section + "How it compares to Pi-hole alone" table.

### Changed
- Auto-application of template domains to Pi-hole groups (the TODO from v0.1.0
  `Known limitations`) is now resolved via `nosey serve` template-diff/apply flow.
- README features list now highlights the UI bridge.

### Notes
- `nosey serve` only adds deps if you actually run it
  (`pip install "fastapi[standard]" uvicorn` — lazy imported).
- UI binds to `127.0.0.1:8765` by default. Don't expose it to LAN without auth.

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
