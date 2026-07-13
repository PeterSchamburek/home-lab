# Firewall Configuration — Raspberry Pi Router

Default-deny firewall built on top of the [Raspberry Pi router](./README.md), replacing an open-by-default `INPUT`/`FORWARD` policy with explicit least-privilege rules.

## Starting Point

The initial router build had NAT and forwarding working (`MASQUERADE`, plus two `FORWARD` rules), but the default chain policies for `INPUT` and `FORWARD` were both still `ACCEPT`. That meant NAT/routing worked, but nothing was actually restricting traffic — any packet not explicitly matched by an existing rule was allowed through by default. That's routing with permit rules layered on top, not a firewall.

## Goal

Flip `INPUT` and `FORWARD` to **default-deny** (`DROP`), and add back only the specific traffic that should be allowed — deny everything, explicitly permit what's needed.

## The Risk: Remote Lockout

Management access to the Pi is via SSH over `eth0` (the home-network side), remotely. Setting the default policy to `DROP` before an explicit SSH allow rule exists would silently drop the very next packet back to the Pi — locking out the only remote access path, recoverable only by physically unplugging/replugging the Pi.

### Safety net: an automatic rollback

Before touching the live ruleset, a dead-man's-switch job was scheduled using `at`/`atd`, set to automatically revert both chains to fully open if something went wrong:

```bash
sudo apt install -y at
sudo systemctl enable --now atd

echo "sudo iptables -P INPUT ACCEPT; sudo iptables -P FORWARD ACCEPT; sudo iptables -F INPUT; sudo iptables -F FORWARD" | at now + 15 minutes
```

This was tested for real: an earlier attempt used a 5-minute window that ran out mid-testing, and the job fired automatically, reverting everything to open with no manual intervention needed — confirming the safety net actually works before relying on it for the real change.

## Ruleset

### 1. Allow rules — added *before* changing the default policy

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -i eth0 -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i wlan1 -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i wlan1 -p udp --dport 67 -j ACCEPT
sudo iptables -A INPUT -i wlan1 -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -i wlan1 -p tcp --dport 53 -j ACCEPT
```

| Rule | Purpose |
|---|---|
| `-i lo -j ACCEPT` | Always allow loopback — the Pi talking to itself internally |
| `state ESTABLISHED,RELATED -j ACCEPT` | Allow return traffic for connections the Pi itself initiated (e.g. its own DNS lookups, `apt` updates) |
| `-i eth0 -p tcp --dport 22` | SSH allowed **only** from the trusted home-network side (`eth0`), not from the AP side |
| `-i eth0/wlan1 -p icmp --icmp-type echo-request` | Ping allowed from both sides, for troubleshooting |
| `-i wlan1 -p udp --dport 67` | DHCP requests from AP clients (dnsmasq handing out leases) |
| `-i wlan1 -p udp/tcp --dport 53` | DNS queries from AP clients (dnsmasq / Pi-hole resolving lookups) |

### 2. Flip the default policies — only after the allow rules exist

```bash
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
```

### 3. Re-affirm forwarding rules under the new default-deny FORWARD policy

```bash
sudo iptables -A FORWARD -i eth0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan1 -o eth0 -j ACCEPT
```

Keeps NAT/internet-sharing intact — return traffic from the internet to AP clients, and outbound traffic from AP clients to the internet — while any other cross-interface traffic is dropped by default.

## Testing Procedure

1. Added all allow rules first, flipped default policies second (never the reverse)
2. Opened a **second, independent** SSH session without closing the first, to confirm remote access still worked
3. Verified ping from a device on the network
4. Verified a phone connected to the Pi's AP still had working internet
5. Only after all of the above passed: cancelled the pending `at` rollback job and made the ruleset permanent

```bash
atq                          # confirm/find the scheduled rollback job
sudo atrm <job_number>       # cancel it once rules are confirmed working
sudo netfilter-persistent save
```

6. Rebooted the Pi and re-verified the ruleset persisted (rather than silently reverting to open) by re-running `sudo iptables -L -v -n` after reconnecting

## Verification Output (post-reboot)

```
Chain INPUT (policy DROP 369 packets, 29608 bytes)
 pkts bytes target     prot opt in     out     source               destination
   12  2467 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
  534 49800 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    1    60 ACCEPT     tcp  --  eth0   *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     icmp --  eth0   *       0.0.0.0/0            0.0.0.0/0            icmptype 8
    0     0 ACCEPT     icmp --  wlan1  *       0.0.0.0/0            0.0.0.0/0            icmptype 8
    0     0 ACCEPT     udp  --  wlan1  *       0.0.0.0/0            0.0.0.0/0            udp dpt:67
    0     0 ACCEPT     udp  --  wlan1  *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
    0     0 ACCEPT     tcp  --  wlan1  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  eth0   wlan1   0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 ACCEPT     all  --  wlan1  eth0    0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 593 packets, 60781 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

Rules and default-deny policies persisted identically across the reboot, confirming the configuration is durable rather than session-only.

## Design Principles Demonstrated

- **Default-deny / least privilege**: nothing is permitted unless explicitly allowed, rather than blocking a list of known-bad traffic on top of an open baseline
- **Change management with rollback**: used a scheduled, automatic revert mechanism (`at`/`atd`) as a safety net before applying a change with real lockout risk — tested the rollback mechanism itself before relying on it
- **Ordered application of changes**: allow rules were added before the restrictive default policy, specifically to avoid a self-inflicted lockout window
- **Verification, not assumption**: every change was tested live (parallel SSH session, ping, AP client internet) before being made permanent, and re-verified after a reboot rather than assumed to persist

## Next Steps

- [ ] Restrict `OUTPUT` chain (currently fully open) once normal traffic patterns are better understood, to avoid breaking legitimate outbound needs (`apt`, NTP, DNS, Wazuh agent communication)
- [ ] Add logging rules (`LOG` target) for dropped packets to aid future troubleshooting and to feed into Wazuh once the SIEM stack is in place
- [ ] Revisit VLAN-specific firewall rules if/when VLAN segmentation is implemented later
