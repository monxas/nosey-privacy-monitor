# What I Found

> The whole reason this project exists. Real discoveries from running this stack on a household LAN in 2026.

## TL;DR

In one month with Nosey on an LG TV, an iPhone, and an Echo Dot:

- **~30,000 tracking queries blocked per week** by per-device Pi-hole groups.
- **4 unknown trackers caught in 2 minutes** when I first opened a news app on the iPhone (Marfeel, Adobe, Sensic, Branch).
- **The LG TV tried to contact `nudge.lgtvcommon.com` every 30 seconds** even when off (standby).
- **Alexa-built-in moved hosts and most public adlists missed it.** This is the biggest finding.

## #1 — Alexa moved hosts and the privacy community didn't notice

Public blocklists (Pi-hole defaults, StevenBlack, OISD, etc.) target older Alexa endpoints:

```
avs-alexa-na.amazon.com
pitangui.amazon.com
dcape-na.amazon.com
arc.msh.amazon.dev
dp-gw-na-js.amazon.com
ais-fe.amazon.com
```

**Most of these were deprecated by Amazon in 2024.** Modern Alexa-built-in (in LG webOS, Samsung Tizen, Fire TV, and the Alexa app itself) uses:

```
*.avs.amazon.dev          ← TLD .dev !
*.apl-alexa.com           ← APL = Alexa Presentation Language
sonic.xapp.avs.amazon.dev
arl.assets.apl-alexa.com
alexa.amazon.{com,es,...}
```

I confirmed this by:
1. Capturing 3 days of SSL SNI logs while pressing the Alexa button on an LG remote.
2. Cross-referencing with what `pitangui` was getting (nothing — it never showed up).
3. After adding the new regex rules to Pi-hole, the Alexa mic light flashes blue and stays blue → "I can't connect to Alexa right now."

Confirmed dead — block at any tier ≥ 2.

## #2 — LG TVs are MQTT chatty even when off

Standby ≠ off for an LG webOS TV. Three persistent TCP/8883 (MQTT TLS) sessions stay alive:

| Destination | Cloud | Purpose (guessed) |
|---|---|---|
| `54.74.67.39` | AWS Dublin | LG ThinQ telemetry |
| `52.209.196.19` | AWS Dublin | LG account / EPG |
| `13.66.142.97` | Azure | LG MQTT bus |

These are IP-pinned and **survive DNS blocking** of the corresponding domains. To actually kill them you need nft DROP rules at the gateway. The TV doesn't seem to notice — Netflix and YouTube continue to work fine.

## #3 — `avsxappcaptiveportal.com` is not what it looks like

When I first saw HTTP traffic to `avsxappcaptiveportal.com/generate_204` from the TV, I assumed it was a fake captive-portal-style tracker. It pinged seven different US-East IPs in five minutes.

It turns out to be a **legitimate Amazon AVS connectivity probe** — the same kind of thing `connectivitycheck.gstatic.com` is for Google. Returns HTTP 204 with empty body. No data filtered.

But if you don't use Alexa, you can safely block it. Listed in `templates/lg-tv.yaml` at Tier 2.

## #4 — iPhone trackers found in the first 2 minutes

Opened the iPhone's default News-like apps after enabling monitoring. In **two minutes**, these tracking domains hit Pi-hole and got blocked:

| Domain | Vendor | What |
|---|---|---|
| `marfeelexperimentsexperienceengine.mrf.io` | Marfeel | A/B testing |
| `sdk.mrf.io` | Marfeel | SDK loader |
| `edge.adobedc.net` | Adobe | Analytics SDK |
| `es-config.sensic.net` | Sensic (GfK) | Audience measurement |

None of these are essential. None of these are in the typical "block these endpoints" guide. They're in `templates/iphone-ios.yaml`.

## #5 — Reddit is 35% of an iPhone's traffic

After 4 hours of normal use: 1,091 HTTPS connections, 159 distinct SNIs. Distribution:

| Category | Connections | % |
|---|---|---|
| Reddit (`cf.gql-fed.reddit.com`, `e.reddit.com`, etc.) | 367 | 33.6% |
| iCloud sync | 127 | 11.6% |
| Slack | 33 | 3.0% |
| Apple background services (gdmf, mesu, bag, gs-loc) | 128 | 11.7% |
| Gmail IMAP | 23 | 2.1% |
| WhatsApp | 17 | 1.6% |
| LLMs (ChatGPT iOS + Grok) | 25 | 2.3% |
| Everything else | 371 | 34.0% |

Not surprising once you see it, but very different from a TV's profile (where streaming dominates).

## #6 — `gs-loc.apple.com` ran 17 queries per hour

The Apple location service polled every ~3.5 minutes from a stationary iPhone. This is:
- Find My iPhone heartbeat
- Significant Locations background scanning
- Weather widget geocoding

If you really care, you can block it (Tier 3 in the template) — but expect:
- Find My less accurate
- Weather may stale
- Significant Locations stops working

I chose NOT to block — that's a fair trade for the user-facing features. Listed
as Tier 3 so it's explicitly opt-in.

## #7 — iCloud Private Relay broke my Pi-hole

When iCloud Private Relay is on, iOS routes DNS over Apple's encrypted relay.
**Your Pi-hole sees zero queries from the device.**

Three options:
1. Disable Private Relay in iOS settings (works, slightly less private from Apple).
2. Block `mask-api.icloud.com` in your DNS blocker (forces fallback, but iOS may show a warning).
3. nft drop the Apple Private Relay IPs at the gateway (most aggressive).

I chose #1. The whole point of Nosey is that **you** see what's happening on your network, not Apple's proxy.

## #8 — Marfeel is the dirtiest mobile tracking I've seen

Spent a couple of hours of news-app browsing on an iPhone with full Zeek capture. The top tracker domains by frequency:

```
marfeelexperimentsexperienceengine.mrf.io   — A/B test variants
sdk.mrf.io                                  — SDK loader
mrfd.mrf.io                                 — Performance metrics
m.exelator.com                              — Identity resolution
adservice.google.com                        — Google ads
```

Every single Spanish news site appears to use Marfeel. Block once, every news app gets quieter.

## Want to contribute findings?

Open a PR against `templates/` with new domains + your evidence (a few Zeek SSL log lines is enough). The point of this project is **shared discovery** — your TV/phone/whatever may have phone-home patterns mine doesn't.
