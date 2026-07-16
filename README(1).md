# Home Network & Security Homelab

Turned a Raspberry Pi 4B into a router, then added a real firewall, a SIEM (Wazuh), and ran the physical cabling to move it out of the garage. Built while studying for the CCNA and prepping for a cybersecurity internship.

Each doc below covers what was built, why it was built that way, and the actual problems that came up — including the mistakes, not just the final config.

## Architecture

```
Fiber Modem (garage)
      │
      │  Cat6A structured cabling run (garage → living room)
      ▼
[eth0] Raspberry Pi 4B [wlan1 - EDUP USB Wi-Fi, mt7921u]
      │                          │
      │ default-deny firewall    │  hostapd AP
      │ NAT / forwarding         │
      ▼                          ▼
  (upstream network)      Wi-Fi clients
      │
      ▼
Wazuh agent (on the Pi) ──────► Wazuh manager / indexer / dashboard
                                  (separate Hyper-V VM, separate hardware)
```

Currently running behind an existing router (double NAT) as a test bed before cutting over to a direct modem connection.

## Projects in this repo

| Doc | What it covers |
|---|---|
| [`raspberry-router.md`](./raspberry-router.md) | Building the Pi into a router: hostapd, dnsmasq, NAT, adapting for a USB Wi-Fi adapter and a newer Debian networking stack (NetworkManager/systemd-networkd instead of dhcpcd) |
| [`firewall.md`](./firewall.md) | Rebuilding the router's iptables rules into an actual default-deny firewall, with a tested rollback so I couldn't lock myself out over SSH |
| [`wazuh.md`](./wazuh.md) | Deploying Wazuh, with the manager/indexer/dashboard kept off the Pi entirely and running on separate hardware |
| [`physical-cabling.md`](./physical-cabling.md) | Running Cat6A from the garage (modem) to the living room (Pi's new spot), termination, testing, and a cable-type mixup along the way |

## Skills

- Networking: NAT, DHCP, DNS, routing, 802.11 AP setup, structured cabling and T568B termination
- Security: default-deny firewalls, least-privilege rules, keeping monitoring separate from what it monitors, testing a rollback before making a risky change
- Linux admin: adapting to a changed networking stack (dhcpcd → NetworkManager/systemd-networkd), systemd, package management
- Troubleshooting: reading actual logs (`dmesg`, `journalctl`) instead of guessing, catching cases where a service reports itself as fine when the hardware underneath isn't
- Documentation: writing these up with the real failures included, not just the working end state

## Status

- ✅ Router built, stable, multi-week soak test in progress
- ✅ Default-deny firewall configured, tested, confirmed to survive a reboot
- ✅ Cabling run done, Pi moved to the living room
- ✅ Wazuh agent deployed and connected to the manager
- ⬜ VLANs (home / guest / tinkering) — holding off, single Wi-Fi radio means more SSIDs would fight each other for airtime
- ⬜ Cut over from existing router straight to the modem
- ⬜ Go through what Wazuh actually flags and tune the ruleset
- ⬜ SSH hardening, patch schedule, threat model doc

## Notes

Real SSID, passphrase, and any static IPs in these docs are redacted before publishing.
