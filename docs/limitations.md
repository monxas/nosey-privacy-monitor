# Limitations (honest)

Nosey is **not** a magic bullet. Things it does NOT solve:

## 1. SNI-only visibility

We see the **hostname** during TLS handshake. We do **not** see HTTP request bodies, query params, or payload content. If a tracker exfiltrates data inside a TLS-encrypted POST to `legit-looking-cdn.com`, Nosey can't tell.

## 2. ECH (Encrypted Client Hello) — coming for SNI

Modern browsers and increasingly mobile apps support [ECH](https://blog.cloudflare.com/handshake-encryption-endgame-an-ech-update/), which encrypts the SNI too. Once a destination supports ECH, you only see "TLS to <IP>" with no hostname.

In May 2026 ECH adoption is still under 15% of mobile traffic. By 2028 this technique will degrade significantly. We mention it explicitly because every privacy tool that uses SNI inspection has this same fundamental limit — anyone telling you otherwise is selling something.

## 3. Cellular fallback

If a device falls back to 4G/5G (your kid's phone, your work laptop, an LTE-equipped TV [yes, these exist]), Nosey sees nothing. It's a LAN tool.

For mobile devices that can't switch off cellular reasonably, the only workaround is an always-on VPN/WireGuard tunnel back to your network. That's Phase 2 of this project (not yet implemented — see [ADR-0001](adr/0001-l3-transparent-gateway.md)).

## 4. IP pinning

Some apps don't use DNS at all. They have hardcoded IP addresses for their backend. DNS blocking doesn't help. You need to identify the IPs in Zeek `conn.log` and add nft DROP rules. The README example for LG TVs (`54.74.67.39`, `52.209.196.19`, `13.66.142.97`) is exactly this pattern.

## 5. Encrypted DNS

If a device uses DoH (DNS over HTTPS) to `1.1.1.1:443` or DoT to `853`, your Pi-hole sees nothing. The `templates/doh-doq-killer.yaml` template addresses this — you drop the well-known IPs at the gateway, forcing devices to fall back to plaintext DNS.

But: it's a cat-and-mouse game. Some apps embed DoH endpoints in code that aren't on the well-known list.

## 6. Family privacy is human, not technical

The hardest part isn't blocking trackers. It's blocking them for your partner / kids / housemate without surveilling them. Nosey's `consent_doc:` requirement is an attempt to make this explicit in the tool. **It cannot replace an actual conversation.**

## 7. False sense of security

Every blocked domain is a small win. But:

- 100 trackers blocked + 1 not blocked = leaked data anyway.
- Mobile carriers do their own tracking (CNAM, location, browsing, depending on country).
- Apps the user is logged into (Instagram, TikTok, etc.) are tracking you with full consent inside their own walls.
- Operating systems track at levels lower than TLS (telemetry baked into the OS).

Nosey makes one piece of the privacy landscape better. The rest is on you.

## 8. Maintenance burden

Trackers move. Apple Migrates services. Google deprecates endpoints. Samsung adds new ones. **Templates go stale.** That's why this is on GitHub — collective updates beat any single person's adlist.

If you find your TV phoning home to a new domain not in the templates, open a PR.
