# Setup — Raspberry Pi

A Raspberry Pi 4 (2 GB) or Pi 5 makes a great dedicated Nosey gateway.

## Hardware

- **Recommended**: Raspberry Pi 4, 2 GB+ RAM, Gigabit Ethernet, microSD or USB SSD.
- **Minimum**: Pi 3 B+ (slower, but adequate for 1-2 monitored devices).
- Power supply: official PSU (Zeek + nft + promtail draws ~3W steady).

## OS

[Raspberry Pi OS Lite (64-bit)](https://www.raspberrypi.com/software/operating-systems/), Debian 12-based. Flash with rpi-imager, enable SSH + WiFi/hostname in the imager wizard.

## Setup

```bash
ssh pi@<your-pi-ip>
sudo apt update && sudo apt upgrade -y
sudo apt install -y nftables zeek python3-yaml python3-pip nmap git curl
```

From here, follow the [Proxmox LXC setup](proxmox-lxc.md) starting from **step 3 (Enable IP forwarding)**. The Zeek install is the same.

## Tips

- Put the Pi on a wired connection if you can. WiFi-only adds latency to every device using it.
- Reserve a static IP for the Pi in your router/Pi-hole DHCP.
- Use an SSD over USB instead of an SD card for `/opt/zeek/logs/` — SD cards die fast under continuous logging.
- Disable Bluetooth and WiFi if wired and unused (`/boot/config.txt` add `dtoverlay=disable-bt` `dtoverlay=disable-wifi`) to free up resources.

## Performance expected

For ~5 monitored devices, normal household traffic:
- Zeek: 3-5% CPU steady, 15% on streaming.
- nft: <0.5% CPU.
- Disk: ~50 MB Zeek logs per device per day.
- Network throughput: ~700 Mbps observed sustained on a Pi 4 with `iperf3`.
