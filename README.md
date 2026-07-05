# Raspberry Pi 4B Router Build (Homelab / CCNA Study Project)

Building a Raspberry Pi 4B into a full router replacement, using a USB Wi-Fi adapter for the AP/LAN side instead of a second wired NIC. Built as a hands-on companion project while studying for the CCNA — this document covers real setup steps, the actual failures encountered, and how each was diagnosed and fixed.

## Hardware

- Raspberry Pi 4B
- EDUP USB Wi-Fi adapter (MediaTek **mt7921u** chipset, confirmed via `lsusb`)
- Onboard Ethernet (`eth0`) — WAN/uplink side
- Onboard Wi-Fi (`wlan0`) — unused in this build
- USB Wi-Fi adapter (`wlan1`) — LAN/AP side

## OS

Raspberry Pi OS, **Trixie** (Debian 13-based). Notably:
- No `dhcpcd` — network config is handled by **NetworkManager** by default
- No `/etc/sysctl.conf` by default — sysctl settings go in `/etc/sysctl.d/` drop-in files instead

This matters because most existing Pi-router tutorials online assume the older `dhcpcd`-based stack and a second wired NIC. Significant adaptation was required.

## Architecture

```
Internet <-> Modem <-> Existing Router <-> [eth0] Raspberry Pi [wlan1] <-> Wi-Fi Clients
```

Currently running **behind an existing router** (double NAT) as a live-fire test bed before cutting over to a direct modem connection. This lets the build be validated over several weeks without risking loss of internet if something breaks.

## Setup Steps

### 1. Packages
```bash
sudo apt update
sudo apt install -y hostapd dnsmasq iptables-persistent netfilter-persistent
```

### 2. Static IP on wlan1 (via systemd-networkd, not dhcpcd)
Took `wlan1` off NetworkManager and assigned it manually:
```bash
sudo tee /etc/systemd/network/10-wlan1.network > /dev/null <<'EOF'
[Match]
Name=wlan1

[Network]
Address=192.168.4.1/24
ConfigureWithoutCarrier=yes
EOF

sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
```
`ConfigureWithoutCarrier=yes` was required — systemd-networkd won't apply static IPv4 config to a wireless interface until it detects carrier, and an AP interface with no hostapd/client running yet never has one.

### 3. hostapd (the access point)
```bash
sudo tee /etc/hostapd/hostapd.conf > /dev/null <<'EOF'
interface=wlan1
driver=nl80211
ssid=REDACTED
hw_mode=a
channel=149
country_code=US
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
auth_algs=1
wpa=2
wpa_passphrase=REDACTED
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
EOF

sudo sed -i 's|#DAEMON_CONF=.*|DAEMON_CONF="/etc/hostapd/hostapd.conf"|' /etc/default/hostapd
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

**Channel choice:** used channel 149 (5GHz, non-DFS) instead of a DFS channel like 36/52, to avoid radar-detection startup delays.

### 4. dnsmasq (DHCP + DNS for AP clients)
```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo tee /etc/dnsmasq.conf > /dev/null <<'EOF'
interface=wlan1
bind-interfaces
dhcp-range=192.168.4.10,192.168.4.100,255.255.255.0,24h
dhcp-option=3,192.168.4.1
dhcp-option=6,192.168.4.1
EOF

sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

### 5. IP forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl -p /etc/sysctl.d/99-ip-forward.conf
```

### 6. NAT / forwarding rules (no firewall configured yet)
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan1 -o eth0 -j ACCEPT
sudo netfilter-persistent save
```

### 7. Making the unmanaged wlan1 setting permanent
`nmcli device set wlan1 managed no` is only in-memory and doesn't survive a reboot. Made it permanent to prevent NetworkManager from ever reclaiming the interface from hostapd:
```bash
sudo tee /etc/NetworkManager/conf.d/99-unmanaged-wlan1.conf > /dev/null <<'EOF'
[keyfile]
unmanaged-devices=interface-name:wlan1
EOF
sudo systemctl restart NetworkManager
```

> **Note:** the rules above provide NAT and basic forwarding only — default `iptables` chain policies were left at `ACCEPT`, so this does not constitute an actual firewall. A proper default-deny firewall configuration will be documented separately.

## Issues Encountered & Fixes

| Issue | Root Cause | Fix |
|---|---|---|
| `dhcpcd.service not found` | Trixie uses NetworkManager, not dhcpcd | Used `nmcli` / `systemd-networkd` instead |
| `wlan1` showed "unavailable" | Wi-Fi + Bluetooth soft-blocked via `rfkill`, likely due to unset country code | `sudo rfkill unblock wifi`; set country/timezone via `raspi-config` |
| `wlan1` had an IP link but no IPv4 address after static config | `systemd-networkd` doesn't apply static config to an interface with no carrier by default | Added `ConfigureWithoutCarrier=yes` |
| `sudo sed ... /etc/sysctl.conf`: No such file | Trixie doesn't ship a default `sysctl.conf` | Used a drop-in file in `/etc/sysctl.d/` instead |
| `wlan1` reverted to NetworkManager-managed after reboot | `nmcli device set managed no` is not persistent | Added a permanent `unmanaged-devices` rule in NetworkManager's config |
| SSID disappeared mid-session, `ip link show wlan1` hung | USB adapter (mt7921u) hit a string of `vendor request failed:-110` (timeout) errors in `dmesg` and underwent a firmware reset; hostapd's internal state didn't notice the hardware reset | `sudo systemctl restart hostapd` to resync hostapd with the reinitialized hardware |

## Design Decisions

- **WPA2 over WPA3**: chosen for compatibility with older devices on the network rather than WPA3/WPA3-transition, given this is a home network with legacy client devices.
- **Channel 149 over DFS channels (36/52/etc)**: avoids radar-detection startup delays and potential silent hostapd failures on some drivers.
- **Testing behind the existing router (double NAT) before full cutover**: validates the whole stack — hostapd, dnsmasq, NAT, persistence across reboot — without risking loss of internet if a config error surfaces mid-build.

## Next Steps

- [ ] Multi-week stability soak test (currently in progress)
- [ ] Monitor for recurring `failed:-110` USB timeout errors; if they recur, move the EDUP adapter to a USB 2.0 port (documented community reports of mt7921u power/negotiation quirks on Pi 4's USB 3.0 controller)
- [ ] Cut over `eth0` from existing router to modem for full router replacement
- [ ] Check ISP MAC-binding requirements before cutover
- [ ] Consider WPA2/WPA3 transition mode (`wpa_key_mgmt=WPA-PSK SAE`, `ieee80211w=1`) once legacy-device compatibility is no longer a constraint
