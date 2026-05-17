# Contributing to Nosey

Thanks for the interest. The most useful contributions, in order:

## 1. Submit a new tracking domain

You found a smart device phoning home to a domain not in our templates? Open a PR adding it to the relevant `templates/<device>.yaml` with:

- The domain (exact or glob).
- **Why it's bloquable**: what feature breaks if you block it? Some endpoints are critical (OTA updates, account auth) — list the trade-off.
- Optional: SNI capture from Zeek log proving it appears.

Example:

```yaml
# templates/lg-tv.yaml
- domain: nudge.lgtvcommon.com
  tier: 2
  category: telemetry
  breaks_if_blocked: "LG Channels backbone — blocking removes the free ad-supported channels app"
  why_block: "Sends viewing analytics every 30s. Heavy nudge prompts on home screen."
  observed_via: "Zeek ssl.log between 2026-04-25 and 2026-05-01"
```

## 2. Setup guide for your platform

Did you set up Nosey on a Raspberry Pi / Ubuntu Server / NixOS / OpenWrt / pfSense?
Add `docs/setup/<your-platform>.md` with the **exact commands** that worked.

## 3. New device template

You have a smart device class not covered? Add `templates/<class>.yaml`:

- Roku TV
- Apple TV
- Samsung Frame TV  
- Google Nest / Home
- Amazon Echo Show
- Sonos
- Smart fridges / washers / cameras
- Tesla / Rivian mobile apps

## 4. Code

Nosey is intentionally small Python (~400 LOC). Don't add heavy dependencies. Stdlib first, `pyyaml` and `requests` allowed.

```bash
git clone https://github.com/monxas/nosey-privacy-monitor.git
cd nosey-privacy-monitor
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
ruff check bin/
pytest tests/
```

## Code of conduct

Be kind. Be honest about limitations. Privacy tooling lives in nuance — `partial protection > no protection`, but never oversell.

## License

By contributing you agree your work is released under MIT (same as the project).
