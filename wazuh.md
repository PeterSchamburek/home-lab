# Wazuh SIEM Deployment — Homelab

Deployed Wazuh (manager, indexer, and dashboard) as a security monitoring layer over the [Raspberry Pi router](./README.md) and its [firewall](./firewall.md), with the Wazuh agent installed directly on the Pi.

## Architecture Decision: Separating the Manager from the Monitored Device

The most important decision in this build happened before any installation: **the Wazuh manager/indexer/dashboard does not run on the Pi.** It runs on a separate Hyper-V VM, on different physical hardware entirely (an old gaming PC, i7/32GB RAM).

Two reasons drove this:

1. **Resource requirements.** Wazuh's indexer (OpenSearch, Java-based) officially recommends 4GB RAM and 8 CPU cores minimum for even a small deployment. The Pi 4B (4GB RAM, 4 cores) is already at or below that floor on its own — before accounting for it also running a router, firewall, and DNS resolver. The lightweight Wazuh **agent** (~35MB RAM), by contrast, runs on the Pi with no meaningful overhead.
2. **Security architecture.** Security monitoring should not live on the same device it's monitoring. If the router is ever compromised, a manager living on the same box could be blinded or tampered with at the same time — defeating the purpose of the monitoring. Running the manager on physically separate hardware keeps the "watcher" independent of the "watched."

## VM Specification

| Setting | Value | Reasoning |
|---|---|---|
| Hypervisor | Hyper-V | Already available on the host (Windows) |
| Guest OS | Ubuntu Server (standard, not minimized) | Wazuh's installer expects a standard package environment |
| Generation | 2 | UEFI support, better performance |
| vCPUs | 3 | Between Wazuh's 2-core floor and 8-core recommendation; workload is a single low-volume agent, not production scale |
| Memory | 8GB, **fixed** (Dynamic Memory disabled) | OpenSearch (Java/JVM) performs unpredictably with memory that resizes at runtime; a stable floor was prioritized over host RAM flexibility |
| Disk | 80–100GB, LVM | Indexer data grows over time; LVM allows the volume to be expanded later without a rebuild |
| Network | External virtual switch | Bridges the VM directly onto the physical LAN, giving it its own independent DHCP lease/IP rather than sharing the host's identity |
| Secure Boot | Disabled | Default Gen 2 setting blocks Linux boot |

## Networking

The VM received a DHCP-assigned IP (`<WAZUH_MANAGER_IP>`) through the External switch, then converted to a **static reservation on the home router** (binding the VM's MAC address to that IP) rather than a static config on the VM itself — keeps the VM's own network config simple (still just DHCP) while guaranteeing the address never changes, which the Wazuh agent depends on to always find the manager.

## Installation

### Manager / Indexer / Dashboard (on the VM)

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The `-a` flag performs an all-in-one install: manager, indexer, and dashboard together on a single host. The installer generates self-signed certificates and prints a one-time admin password for the dashboard at the end — captured immediately, since it is not retrievable from the terminal after the fact (though it remains recoverable from `wazuh-install-files.tar` if needed later).

### Agent (on the Raspberry Pi)

```bash
# Add the Wazuh GPG key and repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list
sudo apt update

# Install, pointed at the manager's static IP
sudo WAZUH_MANAGER="<WAZUH_MANAGER_IP>" apt-get install wazuh-agent

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Because installation went through `apt`, the correct architecture (`arm64`) was pulled automatically for the Pi — no manual package selection needed, despite the Pi and the manager VM running on entirely different CPU architectures.

## Issues Encountered & Fixes

| Issue | Root Cause | Fix |
|---|---|---|
| GPG key import failed with `Permission denied` | `sudo` was applied to `chmod` but not to the `gpg --import` command itself, so the key was never actually written to `/usr/share/keyrings/` | Re-ran the import with `sudo` on the `gpg` command directly |
| `apt update` refused the Wazuh repo as unsigned | Downstream effect of the above — the `.list` file referenced a keyring file that didn't exist | Removed the broken `.list` entry, redid the GPG import correctly, then re-added the repository |
| Dashboard login repeatedly failed (`401`, confirmed via `wazuh-dashboard` logs showing `Failed authentication: Error: Authentication Exception`) | Password was being retyped digit-by-digit from a screenshot rather than copied — easy to mistype ambiguous characters in a long random string | Extracted the password directly from `wazuh-install-files.tar` and copied it via SSH (with a real terminal supporting copy/paste) instead of transcribing by hand |
| `tar` command failed with `Options '-[0-7][lmh]' not supported` | Typed `-0` (the number zero) instead of `-O` (capital O) — visually near-identical in some terminal fonts | Re-ran with the correct flag; also used a simpler `tar -xvf` + `cat` approach afterward to avoid the ambiguous flag entirely |
| Dashboard briefly "unreachable" from a browser | Local Wi-Fi outage on the network, unrelated to any Wazuh/VM configuration | Confirmed once Wi-Fi was restored — a reminder to rule out basic connectivity before deeper troubleshooting |

## Verification

Confirmed end-to-end via the agent's own log on the Pi:

```
wazuh-agentd: INFO: Trying to connect to server ([<WAZUH_MANAGER_IP>]:1514/tcp).
wazuh-agentd: INFO: (4102): Connected to the server ([<WAZUH_MANAGER_IP>]:1514/tcp).
wazuh-syscheckd: INFO: Agent is now online. Process unlocked, continuing...
```

And confirmed the agent appears and shows as active in the Wazuh dashboard's Agents view.

On first startup, the agent automatically began:
- **File Integrity Monitoring (FIM)** — scanning on a 43,200-second (12-hour) interval
- **Security Configuration Assessment (SCA)** — evaluated the CIS benchmark policy for Debian 13 immediately on startup
- **Log collection** — monitoring `dpkg.log`, active-response logs, and system journal entries
- **Rootcheck** — a baseline rootkit/anomaly scan

## Next Steps

- [ ] Let the agent run for an extended period and review what SCA/FIM findings actually surface
- [ ] Review and tune Wazuh's default ruleset — decide which alerts are relevant signal vs. noise for a home network
- [ ] Consider forwarding Pi firewall logs (currently not logged — see firewall doc's next steps) into Wazuh for correlation
- [ ] Revisit VLAN segmentation later; if implemented, evaluate whether additional Wazuh agents are warranted on other segmented devices
- [ ] Document a basic incident-response runbook: what an analyst (i.e., future me) should do when a specific alert type fires
