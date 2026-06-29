# Physical Network Infrastructure — Structured Cabling Run

## Overview

As part of relocating my home lab router (a Raspberry Pi) from the garage to the living room, I planned and executed a structured cabling installation through my home. The goal was to establish a reliable, low-latency wired backbone between my fiber modem and the Pi's new location, eliminating heat and coverage issues associated with the garage environment.

---

## Problem Statement

My ISP's fiber connection enters the house through the garage — a poor location for network equipment. In addition to providing inadequate whole-home Wi-Fi coverage, the garage reaches temperatures that are unsafe for sustained Raspberry Pi operation. Rather than accept a degraded setup or rely on a long Wi-Fi hop, I designed and installed a wired run to bring connectivity into the living room.

---

## Materials & Equipment

| Item | Specification |
|---|---|
| Ethernet cable | Cat6A UTP, 30 ft (ordered; discovered to be STP on termination) |
| Connectors | RJ45 pass-through, 8P8C |
| Crimping tool | CableMatters pass-through crimper |
| Cable tester | Siemens continuity tester |
| Wall outlet | Cat5e keystone punchdown jack |
| Cable management | Cable staples |
| Miscellaneous | Wall outlet covers (×2), drywall tools |

---

## Design Decisions

- **Cat6A** was selected to future-proof the run for 10GbE speeds, even though current devices operate at 1GbE. Cat6A's improved alien crosstalk rejection also makes it appropriate for longer runs in a home environment.
- **T568B** termination standard was used consistently on both ends to ensure a straight-through configuration appropriate for connecting end devices to network infrastructure.
- **18 inches of service loop** was left on the garage side to allow for re-termination if a connector fails or needs to be redone in the future — a standard practice in structured cabling installations.
- **Pass-through RJ45 connectors** were chosen over standard connectors to simplify wire alignment and produce more consistent terminations.

---

## Installation Process

### 1. Cable Routing — Garage
Cable was routed along the garage wall and ceiling using cable staples for physical management. Staples were spaced to keep the cable secure without pinching the jacket, which can degrade signal quality in Cat6A due to its tighter tolerances.

### 2. Wall Penetration
A hole was cut through the shared wall between the garage and living room. Outlet covers were threaded onto the cable before the run was pulled through, allowing them to be seated flush against the drywall on both sides once the cable was at the correct length.

### 3. Termination — Living Room (Keystone Jack)
The cable was cut to length on the living room side, the jacket was stripped, and the individual pairs were punched down into a Cat5e keystone jack using T568B pair orientation. The keystone was seated in the wall outlet and the cover plate mounted.

### 4. Termination — Garage (RJ45 Connector)
The garage end was terminated with a pass-through RJ45 connector using T568B orientation and crimped with the CableMatters tool.

### 5. Testing
A short patch cable was fabricated to bridge the keystone jack on the living room side to the Siemens cable tester. The tester confirmed continuity and correct pin mapping across all eight conductors — no opens, shorts, or miswires detected.

### 6. Final Connections
Short patch cables were fabricated to connect the Raspberry Pi to the living room wall outlet, completing the link between the Pi and the fiber modem in the garage.

---

## Lessons Learned

**STP vs. UTP identification:** The cable ordered as UTP Cat6A turned out to be STP (shielded twisted pair) upon termination — identifiable by the foil shielding around the pairs beneath the outer jacket. In this installation the shielding was removed and the cable was terminated as standard UTP. In a production environment, STP requires proper grounding at both ends; terminating STP without grounding can actually introduce noise rather than eliminate it. Going forward I will verify my orders on arrival rather than trusting they are what I expect.

**Connector compatibility matters:** Early in termination I attempted to reuse an RJ45 connector from a different Cat6A STP cable. That connector had an internal support beam designed for the shielded cable's construction, which repeatedly displaced the wire pairs during insertion. Switching to connectors matched to the cable type resolved the issue immediately. This cost roughly 20 minutes of troubleshooting — a small lesson in not mixing connector types across cable specs.

**T568B wire orientation:** Achieving clean, consistent T568B orientation in pass-through connectors requires deliberate pair untwisting and careful sequencing before insertion. Rushing this step was the primary source of re-dos during this installation.

---

## Outcome

The Raspberry Pi is now operating in the living room on a clean wired connection to the fiber modem. Cable continuity tested clean on all eight pins. The installation provides a stable physical foundation for the broader home lab build, including VLAN segmentation, Linux kernel routing, and SIEM/syslog configuration documented separately.

---

## Skills Demonstrated

- Structured cabling installation (Cat6A)
- RJ45 pass-through termination (T568B)
- Keystone jack punchdown
- Cable routing and physical management
- Continuity testing and fault identification
- Drywall work and cable penetration
- STP vs. UTP identification and handling
