# Playbook: Amazon Echo Dot

Echo devices are aggressive: persistent MQTT TLS to AWS even when idle, hardcoded fallback DNS to 8.8.8.8. Locking them down properly requires both DNS blocking AND IP-level drops.

## Pre-flight

1. Decide your scope:
   - **Keep Alexa working** but kill ads/telemetry → Tier 1 only.
   - **Disable Alexa entirely** (just use as Bluetooth speaker) → Tier 1 + Tier 2 + disconnect Amazon account.
2. If you have multiple Echos, each gets its own registry entry — different rooms = different routing.

## Step 1 — Find the Echo

```bash
bin/nosey discover
```

Look for vendor `Amazon` (OUI `ac:63:be:*`, `f0:81:73:*`, others). Note IP + MAC.

## Step 2 — Add + enable

```bash
bin/nosey add echo-kitchen \
  --ip 192.168.0.180 \
  --mac F0:81:73:AA:BB:CC \
  --owner family \
  --template echo-dot
bin/nosey enable echo-kitchen
bin/nosey apply
```

## Step 3 — Configure Echo's network

Unlike TVs and phones, **Echo devices don't have a UI to set static gateway**. You have two options:

### Option A — DHCP reservation with static gateway (cleanest)

In your router (or Pi-hole DHCP if you use it):
1. Reserve the Echo's MAC to its current IP.
2. Push static gateway = Nosey gateway IP, DNS = Pi-hole IP via DHCP options.
3. Power-cycle the Echo to pick up new lease.

### Option B — Drop the Echo's hardcoded DNS at the gateway

If your router can't set per-device DHCP, the Echo will still use the router as gateway BUT its hardcoded fallback DNS (8.8.8.8) is dropped by the Nosey nft rules. Pi-hole catches the rest.

**Important caveat**: under Option B, the Echo's traffic does NOT flow through the Nosey gateway (Zeek won't see it). You only get the DNS blocking layer.

## Step 4 — Drop persistent MQTT IPs (optional, aggressive)

Echo keeps TCP/8883 alive to AWS even when idle. If you went Option A and the Echo IS routing through the gateway:

```bash
ssh nosey-gateway 'grep "8883" /opt/zeek/logs/current/conn.log | head'
```

Common Echo AWS destinations (EU region):
```
52.86.62.49
52.86.62.50
54.221.x.x range
```

Add to `/etc/nftables.conf`:
```nft
ip daddr { 52.86.62.49, 52.86.62.50 } tcp dport 8883 drop
```

⚠️ This breaks Alexa entirely — voice queries fail silently. Only do this if Alexa is decommissioned.

## Step 5 — Verify

```bash
bin/nosey status echo-kitchen
```

Expected after 5 minutes:
- Some `*.amazon.com` traffic (legit).
- Constant 8883 connections to AWS if Alexa is alive.
- Pi-hole queries log shows blocked `device-metrics-us.amazon.com`, `dcape-na.amazon.com`, etc.

If Alexa stops responding when you intended for it to work:
- Re-check you didn't block `apl-alexa.com` or `avs.amazon.dev` accidentally.
- Verify `apns.courier.push.apple.com`-style critical service hosts are NOT in the block list.
- `bin/nosey disable echo-kitchen` + `apply` = back to default.
