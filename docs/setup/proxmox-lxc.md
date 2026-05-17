# Setup — Proxmox LXC

A small unprivileged LXC on your Proxmox node makes an excellent Nosey gateway.

## 1. Create the LXC

From the Proxmox web UI or shell:

```bash
pct create 110 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname nosey \
  --cores 2 \
  --memory 2048 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,firewall=0,ip=dhcp \
  --features nesting=1 \
  --onboot 1 \
  --unprivileged 1
pct start 110
```

> **Important**: `nesting=1` is required for Zeek (uses caps). `firewall=0` lets nft work without conflicting with Proxmox FW.

## 2. Enter and update

```bash
pct exec 110 -- bash
apt update && apt install -y nftables curl wget git python3-pip python3-yaml nmap
```

## 3. Enable IP forwarding

```bash
cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=2     # loose RPF for same-subnet asymmetry
net.ipv4.conf.all.accept_redirects=0
net.ipv6.conf.all.forwarding=0    # we explicitly DO NOT forward IPv6
EOF
sysctl -p
```

## 4. Apply nftables ruleset

Save as `/etc/nftables.conf`:

```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input   { type filter hook input priority filter; policy accept; }
    chain forward {
        type filter hook forward priority filter; policy accept;

        # Block DoT (TCP 853) and DoQ (UDP 853)
        tcp dport 853 counter drop comment "drop DoT"
        udp dport 853 counter drop comment "drop DoQ"

        # Block well-known DoH endpoints on :443
        ip daddr { 1.1.1.1, 1.0.0.1, 8.8.8.8, 8.8.4.4, 9.9.9.9, 149.112.112.112 } \
          tcp dport 443 counter drop comment "drop DoH TCP"
        ip daddr { 1.1.1.1, 1.0.0.1, 8.8.8.8, 8.8.4.4, 9.9.9.9, 149.112.112.112 } \
          udp dport 443 counter drop comment "drop DoH QUIC"
    }
    chain output  { type filter hook output priority filter; policy accept; }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        # Force any DNS query NOT aimed at Pi-hole to Pi-hole
        ip daddr != 192.168.0.204 udp dport 53 dnat to 192.168.0.204:53
        ip daddr != 192.168.0.204 tcp dport 53 dnat to 192.168.0.204:53
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "eth0" masquerade
    }
}

table ip6 filter {
    chain forward {
        type filter hook forward priority filter; policy drop;  # NO IPv6 forwarding
    }
}
```

Replace `192.168.0.204` with your Pi-hole IP.

```bash
systemctl enable --now nftables
nft list ruleset    # verify
```

## 5. Install Zeek

```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/Debian_12/ /' \
  > /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/Debian_12/Release.key \
  | gpg --dearmor -o /etc/apt/trusted.gpg.d/security_zeek.gpg
apt update && apt install -y zeek

# Configure standalone mode
sed -i 's/^interface=eth0/interface=eth0/' /opt/zeek/etc/node.cfg
# enable JSON output
echo "@load policy/tuning/json-logs.zeek" >> /opt/zeek/share/zeek/site/local.zeek

/opt/zeek/bin/zeekctl deploy
/opt/zeek/bin/zeekctl status
```

> **Note**: zeekctl will sometimes report "crashed" during the first 30 seconds. Wait, then `zeekctl status` should show "running". Check `ps aux | grep zeek` if unsure.

## 6. Optional: promtail for Loki

Only if you already have Loki running somewhere. Skip this section for standalone mode.

```bash
wget -O /usr/local/bin/promtail.gz \
  https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64.zip
unzip -o /tmp/promtail.zip -d /usr/local/bin/
chmod +x /usr/local/bin/promtail-linux-amd64
mv /usr/local/bin/promtail-linux-amd64 /usr/local/bin/promtail

# config
mkdir -p /etc/promtail
cat > /etc/promtail/config.yml <<'YAML'
server: { http_listen_port: 9080 }
clients:
  - url: http://<your-loki-host>:3100/loki/api/v1/push
positions: { filename: /var/lib/promtail/positions.yaml }
scrape_configs:
  - job_name: zeek
    static_configs:
      - targets: [localhost]
        labels:
          host: nosey
          job: zeek
          __path__: /opt/zeek/logs/current/*.log
YAML

systemctl enable --now promtail
```

## 7. Install Nosey itself

```bash
cd /opt
git clone https://github.com/monxas/nosey-privacy-monitor.git
cd nosey-privacy-monitor
pip3 install pyyaml  # if not already

cp config/nosey.example.yaml config/nosey.yaml
cp config/devices.example.yaml config/devices.yaml
# Edit both: set your gateway IP, Pi-hole IP/password, etc.

export PIHOLE_PASSWORD='your-pihole-password'
bin/nosey list   # should show empty registry
bin/nosey wizard # interactive add
```

## 8. Done

```bash
bin/nosey apply
```

Next, configure your first device's WiFi manually (see the [playbooks](../playbooks/)).
