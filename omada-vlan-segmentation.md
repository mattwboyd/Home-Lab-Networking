# Home Network Segmentation Lab — Omada VLANs, SSIDs, and Gateway ACLs

**Date:** 2026-08-23
**Environment:** TP-Link Omada Software Controller (v6.2.x) managing 1 gateway, 1 switch, 2 APs
**Objective:** Segment a flat home network into three isolated VLANs (Main, IoT, Guest) with per-VLAN SSIDs and a gateway ACL trust model, then validate isolation.

---

## 1. Overview

Started from a flat `192.168.0.0/24` network with all devices in one broadcast domain. Rebuilt it into a segmented design where low-trust devices (IoT, guests) are contained to their own subnets and cannot reach trusted devices, while trusted devices (Main) can reach everything.

The core design principle: **segment the noisy, low-trust devices; keep actively-controlled devices on the trusted network.** A streaming box (Roku) that is controlled constantly from a phone was intentionally left on Main rather than forced onto IoT — see the design note in section 6.

> Note: SSID and network names in this document are genericized (`Main-SSID`, `IoT-SSID`, `Guest-SSID`). Substitute your own names when applying.

---

## 2. VLAN / Subnet Design

| Purpose | VLAN ID | Subnet | Gateway | DHCP Pool | Omada Network Name |
|---------|---------|--------|---------|-----------|--------------------|
| Main (trusted) | 1 | 192.168.0.0/24 | 192.168.0.1 | .100–.199 | Default |
| IoT | 20 | 192.168.20.0/24 | 192.168.20.1 | .100–.199 | IoT-Net |
| Guest | 30 | 192.168.30.0/24 | 192.168.30.1 | .100–.199 | Guest-Net |

DHCP pools intentionally start at `.100` to leave `.1–.99` free for the gateway and any static/reserved assignments.

Each VLAN was created under **Network Config → VLAN → Add LAN**, as an *Interface* purpose network so the gateway routes for it, with its own DHCP scope (DNS and default gateway left on Auto).

---

## 3. Wireless (SSID → VLAN Mapping)

Three SSIDs, each bound to its VLAN via **Advanced Settings → VLAN → Custom → By Network**. Binding "by network" (rather than typing a raw VLAN ID) links the SSID directly to the LAN object, so subnet and DHCP stay consistent automatically.

| SSID | VLAN | Bands | Notes |
|------|------|-------|-------|
| Main-SSID | 1 (Main) | 2.4 + 5 GHz | Existing trusted network |
| IoT-SSID | 20 | 2.4 + 5 GHz | Both bands kept — TV streams video over Wi-Fi and needs 5 GHz throughput |
| Guest-SSID | 30 | 2.4 + 5 GHz | Guest Network toggle ON (wireless client isolation) |

The APs carry all three VLANs as tagged traffic over the trunk automatically — no per-AP configuration was needed. One SSID pushes to both APs with seamless roaming.

**Band note:** IoT SSIDs are often set 2.4 GHz-only so legacy smart-home gear onboards cleanly, but this deployment kept both bands because the TV (streaming) needs 5 GHz.

---

## 4. Gateway ACL Trust Model

ACLs applied under **Network Security → Gateway ACL**. Gateway ACL is the correct type because inter-VLAN traffic is *routed* by the gateway.

**Key behavior:** Omada Gateway ACLs are evaluated top-down, first-match-wins, with an **implicit ALLOW-all** at the bottom. Each rule therefore carves out a deny exception; anything not explicitly denied is permitted. (Contrast: Cisco ACLs use an implicit *deny*.)

### Rules (all Direction: LAN→LAN, Policy: Deny, Protocol: All)

| # | Description | Source | Destination |
|---|-------------|--------|-------------|
| 1 | Block IoT to Main | IoT-Net (192.168.20.0/24) | Main-Net (192.168.0.0/24) |
| 2 | Block IoT to Guest | IoT-Net | Guest-Net (192.168.30.0/24) |
| 3 | Block Guest to Main | Guest-Net | Main-Net |
| 4 | Block Guest to IoT | Guest-Net | IoT-Net |

Source/destination are defined as **IP Groups** (named subnets) created under Profiles/Groups, since this firmware references IP Groups rather than VLAN objects directly in ACLs.

### Resulting trust matrix

| From ↓ / To → | Main | IoT | Guest | Internet |
|---------------|------|-----|-------|----------|
| **Main** | ✔ | ✔ | ✔ | ✔ |
| **IoT** | ✘ | ✔ | ✘ | ✔ |
| **Guest** | ✘ | ✘ | ✔ | ✔ |

**Why no "allow Main→IoT" rule is needed:** the implicit allow-all already permits Main→IoT. The gateway firewall is **stateful**, so when a Main device (e.g. a phone) initiates a connection to an IoT device, the return traffic for that established session is automatically allowed — even though Rule 1 blocks IoT-*initiated* traffic to Main. This creates a one-way trust: Main can reach into IoT, IoT cannot reach back.

No rule shadows another because all four are denies with non-overlapping source/destination pairs, so ordering is not sensitive here.

---

## 5. mDNS / Discovery Across VLANs

Cross-VLAN service discovery (casting, AirPlay, printers) requires an **mDNS repeater** (Network Config → mDNS), because discovery multicast does not cross VLAN boundaries even when ACLs permit the unicast traffic.

Omada's mDNS repeater bridges **Bonjour/mDNS** services only (AirPlay, Chromecast, Printers, Samba, SSH, etc.). It advertises a *Service Network*'s services into a *Client Network*.

**Limitation encountered:** Roku uses **SSDP** (and DIAL/ECP), not mDNS. Omada's mDNS repeater does not bridge SSDP, so it cannot make a Roku on IoT discoverable from Main. See design note below.

---

## 6. Design Note — Roku Placement

The TV is a Roku, controlled from a phone via the Roku app and used for casting.

Roku discovery relies on SSDP, which the Omada mDNS repeater cannot bridge across VLANs. Options considered:

- **A. Roku on Main** *(chosen)* — simplest; discovery and casting work with zero extra config since Main→everywhere is already open. The Roku is an actively-controlled device, so keeping it trusted is reasonable. Genuine IoT devices (bulbs, plugs, sensors) remain contained on VLAN 20.
- **B. Roku on IoT, control by IP** — keep it segmented and use the Roku app's "connect by IP" against its 192.168.20.x address (allowed by the Main→IoT stateful path). Requires a DHCP reservation and loses auto-discovery/third-party casting.

**Decision:** Option A. Segmentation effort is best spent on genuinely untrusted devices; forcing an actively-used streaming box onto IoT adds friction without meaningful security gain.

---

## 7. Validation Checklist

- [ ] **Clients page** — Roku and phone on 192.168.0.x; IoT devices on 192.168.20.x; guests on 192.168.30.x
- [ ] **IoT isolation** — device on VLAN 20 reaches internet but cannot reach 192.168.0.1 (Main gateway/admin) — timeout = success
- [ ] **Guest isolation** — device on VLAN 30 reaches internet but not Main or IoT
- [ ] **Casting** — phone (Main) casts to Roku (Main) successfully
- [ ] **DHCP reservations** — fixed addresses set for devices that need stable IPs (Roku, lab hosts)

---

## 8. Concepts Reinforced

- **VLAN segmentation** — separate broadcast domains / subnets on shared physical infrastructure via 802.1Q tagging
- **ACL evaluation** — top-down first-match; implicit allow (Omada gateway) vs. implicit deny (Cisco)
- **Stateful firewalling** — return traffic for established sessions auto-permitted; enables one-way trust with a single deny rule
- **Trunk vs. access / tagged vs. untagged** — APs carry multiple VLANs tagged over a trunk to the controller/switch
- **Service discovery protocols** — mDNS/Bonjour vs. SSDP/DIAL, and why multicast discovery needs a repeater to cross VLANs
