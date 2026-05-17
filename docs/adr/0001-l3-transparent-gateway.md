# ADR-0001 — Use a Linux box as transparent L3 gateway for monitoring opt-in devices

| Field    | Value          |
|----------|----------------|
| Status   | Accepted       |
| Date     | 2026-05-04     |

## Context

We want to inspect and selectively block traffic from specific LAN devices (TV, optional mobile) without affecting the rest of the household network.

## Decision

A dedicated Linux box (LXC, Raspberry Pi, etc.) acts as the **monitoring gateway**. Devices that opt in are configured to use this box as their default WiFi gateway (and DNS server pointed at Pi-hole). The box:

- Masquerades their traffic out the normal upstream.
- Runs Zeek for SSL/DNS/connection logging.
- Applies nftables rules to drop DoT/DoQ/well-known DoH, and force-DNAT `:53` queries to Pi-hole.

## Alternatives considered

1. **Router-side filtering** — most consumer routers (Movistar R2 in our case) don't expose per-device DNS or PCAP. Rejected.
2. **Replace the router** — too invasive for a household; partner not interested in network downtime.
3. **WireGuard always-on per device** — works for phones/laptops but doesn't help for smart TVs that can't run a VPN client. Will be added as a Phase 2 supplementary mode.
4. **mitmproxy with custom CA** — would give plaintext visibility but requires installing a CA cert on every device. Closed-platform devices (LG webOS, Roku) can't accept it. Rejected as primary mechanism.
5. **Per-device VLAN** — needs VLAN-capable switches + a router that supports tagged VLANs. Movistar R2 doesn't. Could be a future improvement if the router is replaced.

## Consequences

**Positive**
- Opt-in per device.
- Works with any consumer router.
- No client-side software required (TV can't install our cert anyway — doesn't matter, SNI is plaintext).
- Failure mode is graceful: disable monitoring → device reverts to DHCP → back to normal.

**Negative**
- Single point of failure: if the gateway box dies, opted-in devices lose internet (until they switch back to DHCP, which requires manual action).
- IPv6 must be disabled on monitored devices (gateway only forwards IPv4 — IPv6 RA traffic goes direct).
- SNI inspection only — no payload visibility for TLS.

## Mitigations

- Health-check the gateway via Prometheus blackbox/HA + alert on outage.
- Document the "rollback in 2 minutes" procedure visibly in every playbook (`docs/playbooks/*.md`).
- IPv6 disable is mandatory in setup playbooks.
