# Playbook: LG webOS TV

End-to-end setup for an LG smart TV. Tested on LG 43UQ80006LB / webOS p20.04.54.40.

## Prerequisites

- Nosey gateway running and reachable on your LAN (e.g. `192.168.0.210`).
- Pi-hole (or AdGuard) running and reachable (e.g. `192.168.0.204`).
- Nosey CLI configured (`config/nosey.yaml`, `config/devices.yaml`).

## Step 1 — Disable IPv6 in webOS

This is mandatory. The Nosey gateway only catches IPv4. If IPv6 is enabled, the TV's IPv6 traffic bypasses everything.

1. TV menu → **Settings → All Settings → Network → Wired/WiFi → Advanced**.
2. **IPv6 → Off**.

## Step 2 — Discover the TV on the LAN

```bash
bin/nosey discover --subnet 192.168.0.0/24
```

Look for a line like:

```
192.168.0.227   ac:5a:f0:8b:43:c2   LG Electronics              candidate: lg-tv
```

Note the IP and MAC.

## Step 3 — Add to registry

```bash
bin/nosey add living-room-tv \
  --ip 192.168.0.227 \
  --mac ac:5a:f0:8b:43:c2 \
  --owner family \
  --template lg-tv \
  --notes "LG 43UQ80006LB"
```

## Step 4 — Enable + apply

```bash
bin/nosey enable living-room-tv
bin/nosey apply
```

Apply will:
- Create Pi-hole group `living-room-tv`.
- Assign client IP to that group.
- Print on-screen instructions.

Now in your **Pi-hole admin → Domains**, add the entries from `templates/lg-tv.yaml`
to the new group `living-room-tv`. (Future enhancement: `nosey apply` will push
these automatically via Pi-hole API.)

## Step 5 — Configure the TV's network

1. TV menu → **Settings → All Settings → Network → Wired/WiFi → Edit**.
2. Choose **Manual** for IP configuration.
3. Set:
   - **IP**: keep what the TV currently has (or pick a static one).
   - **Subnet mask**: `255.255.255.0` (or yours).
   - **Gateway**: `<your Nosey gateway IP>` (e.g. `192.168.0.210`).
   - **DNS**: `<your Pi-hole IP>` (e.g. `192.168.0.204`).
4. Save and reconnect WiFi.

## Step 6 — Verify

After ~60 seconds:

```bash
bin/nosey status living-room-tv
```

You should see Zeek conn events and top SNIs. The first observed SSL handshakes
will typically be:
- `*.netflix.com`, `*.amazonvideo.com` — streaming (allowed)
- LG's own domains being blocked (matched, returned 0.0.0.0 by Pi-hole)

## Step 7 — Catch IP-pinned LG cloud connections

Some LG telemetry IPs are hardcoded in the TV firmware and survive DNS blocking.
Check Zeek logs for persistent connections to AWS Dublin and Azure IPs:

```bash
ssh nosey-gateway 'grep -E "id_resp_h.*(52\.209|54\.74|13\.66)" /opt/zeek/logs/current/conn.log | head'
```

If you see them, add to `/etc/nftables.conf`:

```nft
ip daddr { 52.209.196.19, 54.74.67.39, 52.16.104.93, 13.66.142.97 } drop
```

Apply with `nft -f /etc/nftables.conf`.

## Step 8 — Disable LG ThinQ (optional, aggressive)

If you don't use the ThinQ phone app to control the TV:

- Add Tier 3 entries from `templates/lg-tv.yaml` (lgthinq.com, lgtviot.com).
- Remove TV from LG account: TV menu → Settings → Account → Logout.

This kills nearly all LG cloud traffic. Tradeoff: no remote turn-on from phone, no OTA notifications (manual update only).

## Expected end-state

After a week:

- 10-15 distinct SNIs visible (all streaming services you actually use).
- 100% of LG telemetry blocked.
- Nudge banners gone from TV home screen.
- Alexa-built-in is disabled (intentional — Tier 2 blocks it).
- Streaming, video playback, picture quality: **unchanged**.

If something breaks unexpectedly, you can disable the entire setup quickly:

```bash
bin/nosey disable living-room-tv
bin/nosey apply
```

And revert the TV's WiFi to DHCP. Back to normal in 2 minutes.
