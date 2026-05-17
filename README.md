<div align="center">

# 🔍 Nosey

**Find out what your smart devices are really talking to.**

Plug-and-play L3 traffic monitor + privacy auditor for your LAN. Catch the trackers,
silence the telemetry, keep the streaming. Bring your own Pi-hole.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

[**Quick Start**](#quick-start) · [**What you'll find**](docs/what-i-found.md) · [**Templates**](templates/) · [**Setup guides**](docs/setup/) · [**Playbooks**](docs/playbooks/)

</div>

---

## Why

Your TV. Your phone. Your Echo. Your smart bulb. **Most of them phone home dozens of times per minute** — to ad networks, telemetry servers, model-training clouds. You bought the device, they keep the data.

**Nosey** routes selected devices through a transparent L3 gateway, watches their traffic with Zeek, and applies per-device blocklists curated from real observation (not stale public adlists from 2018).

After one week with Nosey on an LG TV:

- 12 distinct SNI destinations remaining (all Netflix / Amazon Prime — legit)
- 100% of LG telemetry endpoints blocked (`lgtvcommon`, `lgtvsdp`, `lgeapi`, `lgtvonline`)
- Alexa-built-in voice search killed at the right modern endpoints (`*.avs.amazon.dev`, `*.apl-alexa.com` — **most public adlists target the old AVS hosts that were deprecated in 2024**)

This repo is the tooling + the curated blocklists. Bring your own Pi-hole or AdGuard.

---

## Features

- 📋 **Device registry** as YAML — one file, one source of truth.
- 🌐 **Web UI bridge** — `nosey serve` launches a local dashboard (FastAPI + HTMX) that pulls Pi-hole stats and **pushes template rules to Pi-hole groups with a diff/confirm flow**. No more copy-pasting blocklists into the Pi-hole admin.
- 🔬 **Forensic capture mode** — `nosey capture <device>` pulls Zeek logs from the gateway and emits a markdown report with top SNIs, IPs, bytes, TLS posture, and cadence (heartbeat detection). No MITM, no custom CA — passive observation gives you ~90% of the forensic value.
- 🎯 **Curated blocklist templates** per device class (LG TV, Samsung TV, Echo Dot, iPhone, Android, ...) with **comments explaining every domain** (you decide what to block).
- 🛡 **Consent enforcement** — refuses to enable monitoring for devices you don't own unless a `consent_doc:` is referenced. Family-friendly defaults.
- 🔍 **Auto-discovery** — `nosey discover` scans the LAN, identifies devices by MAC vendor, suggests templates.
- 📊 **Standalone mode** — works with just Pi-hole + nft, no Loki/Grafana required (observability is opt-in).
- 🧱 **Multi-backend DNS** — Pi-hole, AdGuard Home, Unbound.
- 🔥 **DoH/DoT killer** — drops known encrypted DNS endpoints (1.1.1.1, 8.8.8.8, Quad9, Apple Private Relay) so devices can't bypass your blocklists.
- 📈 **Optional Grafana dashboard** if you have the stack.
- 🐧 Runs on **Proxmox LXC, Raspberry Pi, Docker, bare metal**.

---

## Quick Start

```bash
# 1. Clone
git clone https://github.com/monxas/nosey-privacy-monitor.git
cd nosey-privacy-monitor

# 2. Run interactive bootstrap (asks about your setup)
./scripts/install.sh

# 3. Add your first device (interactive)
bin/nosey wizard

# 4. Apply config
bin/nosey apply

# 5. On the device: configure WiFi → Router = <gateway IP>, DNS = <Pi-hole IP>
#    (Nosey prints the exact instructions after `apply`)

# 6. Watch what your device is doing
bin/nosey status my-device

# 7. Optional — launch the web UI bridge to Pi-hole
pip install "fastapi[standard]" uvicorn
bin/nosey serve         # → http://127.0.0.1:8765
```

That's it. No Kubernetes. No Docker. No cloud account. ~50 MB of Python + Zeek.

---

## What "monitoring" means here

Nosey **does NOT** intercept TLS (no MITM, no custom CA, no installing profiles on your iPhone). What it sees:

- ✅ **Server Name Indication (SNI)** — the hostname your device asks for (visible in cleartext during TLS handshake).
- ✅ **DNS queries** (via Pi-hole logs).
- ✅ **IPs, ports, byte counts** (Zeek `conn.log`).
- ✅ **Plaintext HTTP headers** (if any device still does HTTP — rarer every year).

What it does NOT see:

- ❌ HTTPS request bodies / API payloads.
- ❌ Encrypted DNS responses (DoH/DoT) — but Nosey blocks those at the gateway so devices fall back to plain DNS, where Pi-hole catches them.

This is enough to identify ~95% of trackers and most telemetry endpoints. Read [docs/what-i-found.md](docs/what-i-found.md) for examples of real discoveries.

---

## How it works (1 paragraph)

You designate one host on your LAN as the "monitoring gateway" — a tiny machine (Proxmox LXC, Raspberry Pi, old laptop) running Linux with `nftables` + Zeek. Devices you opt in are configured (manually in iOS/macOS, or via WireGuard) to use that host as their **default gateway**. The gateway masquerades their traffic out to the internet, Zeek sniffs the flows in JSON, optional promtail ships logs to Loki/Grafana. DNS goes to your Pi-hole (or AdGuard), grouped per-device so each device gets its own blocklist policy. Devices you don't opt in are unaffected.

Detailed diagrams: [docs/architecture.md](docs/architecture.md).

---

## The Pi-hole bridge (`nosey serve`)

Run `nosey serve` and open `http://127.0.0.1:8765` — a local-only dashboard that:

- **Pulls live Pi-hole stats** (queries/blocked/clients last 24h) via the Pi-hole 6 REST API.
- **Lists every registered device** with its per-client query count + block %.
- **Toggle enable/disable** per device — consent check enforced, just like the CLI.
- **"Template diff" button** per device — shows exactly which domains the template would **add** / **remove** / **keep** in the Pi-hole group, with the `why_block` and `breaks_if_blocked` reason for each one. Confirm → it pushes those changes directly into Pi-hole via API.

So you finally get answers to *"is my LG TV blocklist actually applied in Pi-hole right now?"* without clicking through 5 admin screens.

The UI is a single Python file. Tailwind + HTMX from CDNs. No build step, no Node, no Docker. ~600 LOC total.

```bash
pip install "fastapi[standard]" uvicorn
bin/nosey serve --host 127.0.0.1 --port 8765
```

> **Security**: by default `nosey serve` binds to localhost only. If you bind to `0.0.0.0`, put a reverse proxy with auth in front of it (Caddy + basicauth, Tailscale Funnel, etc.).

---

## How it compares to Pi-hole / AdGuard alone

**Nosey is not a replacement for Pi-hole — it's a bridge that gives Pi-hole eyes and per-device curation.**

|                                                                   | Pi-hole alone                | Nosey + Pi-hole                                  |
| ----------------------------------------------------------------- | ---------------------------- | ------------------------------------------------ |
| DNS blocking by domain                                            | ✅                            | ✅ (Pi-hole stays as the engine)                  |
| Device hardcodes `8.8.8.8` → bypasses your blocker                | ❌ invisible                  | ✅ nft DNAT redirects to Pi-hole                  |
| App uses DoH (`1.1.1.1:443`, `dns.google:443`)                    | ❌ invisible                  | ✅ DROPed at gateway, app falls back to OS DNS    |
| TV connects to a cached IP (no DNS query)                         | ❌ never seen                 | ✅ nft IP DROP rules                              |
| **What is each device *actually* doing right now?** (SNI per IP)  | ❌ only DNS, no post-resolve  | ✅ Zeek `ssl.log` shows real TLS hostnames        |
| Per-device blocklist with `breaks_if_blocked` for every domain    | ⚠️ groups+adlists, no notes  | ✅ curated YAML templates with reasoning          |
| Find brand-new trackers (Marfeel, Sensic, Adobe DTM)              | ❌ only if in a public adlist | ✅ from your own SSL logs in 1–2 hours of capture |
| Apply a template (12 domains) to a device group                   | ❌ manual UI clicks           | ✅ one button + diff/confirm in `nosey serve`     |

**One-liner**: Pi-hole tells you which queries it blocked. Nosey tells you what each device is doing when Pi-hole isn't looking.

---

## Privacy ethics built-in

Nosey is deliberately **opt-in per device** with a consent check:

```yaml
# In devices.yaml
ramons-iphone:
  owner: ramon           # "self" — proceeds
  enabled: true

partner-iphone:
  owner: gloria          # NOT self
  consent_doc: docs/consent-log-gloria.md  # REQUIRED to enable
  enabled: false
```

If you try `nosey enable partner-iphone` without `consent_doc`, the tool refuses with a clear message pointing to a sample consent log template. Family privacy isn't a side note here.

---

## Project status

**v0.1** — works for the author. Code is small (~400 LOC Python), tested only on Debian 12 + Pi-hole 6 + Proxmox LXC. PRs welcome especially for:

- Setup scripts for Raspberry Pi OS, Ubuntu Server, NixOS.
- AdGuard Home / Unbound backends.
- More device templates (Roku, Apple TV, Samsung Frame, Google Home, Sonos).
- WireGuard companion mode (Phase 2 — for mobile devices that can't set static gateway).

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## What this is NOT

- ❌ **Not** a PSA replacement for [Pi-hole](https://pi-hole.net/) / [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) / [Blocky](https://0xerr0r.github.io/blocky/) — Nosey **complements** them with per-device routing and curated templates.
- ❌ **Not** a NIDS (no signature-based threat detection) — use [Suricata](https://suricata.io/) or [Crowdsec](https://crowdsec.net/) for that.
- ❌ **Not** going to defeat the most determined trackers — IP-pinned services, ECH-encrypted handshakes, and cellular fallback all degrade SNI inspection. Read [docs/limitations.md](docs/limitations.md).

---

## License

MIT. Use it, fork it, do what you want. If you find new tracking domains worth blocking, please open a PR against the templates — that's the whole point.

---

<sub>Built because privacy adlists from 2018 don't catch what TVs are doing in 2026.</sub>
