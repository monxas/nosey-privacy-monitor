# Playbook: iPhone (iOS 17–18)

## Prerequisites

- Nosey gateway + Pi-hole running.
- Your iPhone WILL be in scope. (For someone else's iPhone, see [Consent and ethics](#consent-and-ethics).)
- ~5 minutes.

## Pre-flight (do this BEFORE Nosey)

These are settings on the iPhone itself that determine whether Nosey can see anything:

1. **Settings → Wi-Fi → tap the (i) on your home network**:
   - **Private Wi-Fi Address: OFF**. Otherwise the iPhone changes its MAC each session and your Pi-hole group assignment becomes unreliable.
2. **Settings → [your name] → iCloud → Private Relay: OFF** (at least for this WiFi).
   - With Private Relay ON, iOS routes DNS via Apple's encrypted relay. Your Pi-hole sees no queries at all from the iPhone.
3. **Settings → Privacy & Security → Tracking → "Allow Apps to Request to Track" → OFF**. Different layer (this is App Tracking Transparency, baked into iOS) but worth it.

## Step 1 — Find the iPhone's current IP

```bash
bin/nosey discover --subnet 192.168.0.0/24
```

Or on the iPhone itself: Settings → Wi-Fi → tap the (i) → "IP Address" field.

## Step 2 — Add to registry

```bash
bin/nosey add my-iphone \
  --ip 192.168.0.50 \
  --mac AA:BB:CC:DD:EE:FF \
  --owner self \
  --template iphone-ios
```

(The MAC must be the **stable one**, after you disabled Private Wi-Fi Address.)

## Step 3 — Enable + apply

```bash
bin/nosey enable my-iphone
bin/nosey apply
```

## Step 4 — Configure the iPhone

1. Settings → Wi-Fi → ⓘ next to your network.
2. Scroll to **"Configure IP" → tap → choose Manual**.
3. Fill in:
   - **IP Address**: same as before (or reserve in DHCP).
   - **Subnet Mask**: `255.255.255.0` (your subnet).
   - **Router**: `<Nosey gateway IP>`.
4. Scroll to **"Configure DNS" → tap → choose Manual → Add Server**.
5. Add `<Pi-hole IP>`. Remove other servers.
6. Tap **Save**.
7. Toggle WiFi off and on (to force re-DHCP and re-association with new config).

## Step 5 — Verify

After 60 seconds, browse anything (open Safari → news site is a good test).

```bash
bin/nosey status my-iphone
```

You should see:
- Hundreds of SNI hits in the first few minutes (iOS is chatty).
- Top services: Apple background, iCloud, your installed apps.

Trackers (Marfeel, Adobe Analytics, etc.) will appear in Pi-hole's **Query Log** as
**Blocked by Gravity** or **Denylist**. They do not appear in Zeek SSL log because
they never get past the DNS lookup.

## What you'll observe in the first 24 hours

| Pattern | What it means |
|---|---|
| ~270 distinct SNIs/hour | Normal for an active iPhone |
| `gateway.icloud.com` topping the list | iCloud sync — expected |
| `gdmf.apple.com`, `mesu.apple.com` constant | Apple device management / updates — expected |
| `mask-api.icloud.com` | Apple Private Relay (you should have turned this OFF) |
| `gs-loc.apple.com` ~17/hr | Background location queries — see template Tier 3 |
| Reddit / Slack / WhatsApp / Gmail | Your normal apps |

## Rollback

If iOS starts complaining ("No Internet connection" loop) or apps misbehave:

1. Settings → Wi-Fi → ⓘ → Configure IP → **Automatic**.
2. Configure DNS → **Automatic**.
3. Toggle WiFi.
4. Back to normal in 30 seconds.

The Nosey registry entry doesn't need to be deleted — the iPhone is just no longer routing through the gateway.

## Consent and ethics

If this iPhone is **not yours**:

```bash
bin/nosey enable partner-iphone
# ⛔ 'partner-iphone' (owner='partner') requires consent_doc or --force.
```

Create `docs/consent-log-partner.md` with:
- Date of consent conversation.
- Scope (what data is captured: SNI hostnames, DNS queries, IPs, NOT message contents).
- Kill-switch (how to disable in 30s).
- Signature line.

Reference it in `devices.yaml`:

```yaml
partner-iphone:
  owner: partner
  consent_doc: docs/consent-log-partner.md
  ...
```

Now `nosey enable` works.

This is friction *on purpose*. Privacy tooling for households fails when one
adult silently monitors another. Nosey ships with a soft brake.
