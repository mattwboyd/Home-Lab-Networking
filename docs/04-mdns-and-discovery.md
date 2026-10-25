# Phase 4 — mDNS, Discovery & the Roku Decision

**Goal:** Deal with the most common side effect of segmentation: devices that can still be
*reached* across VLANs but can no longer be *discovered*.

---

## Key concepts

- **Discovery is multicast** — casting, AirPlay and printer discovery work by shouting on the
  local network. Multicast doesn't cross VLAN boundaries, even when the ACLs allow the
  unicast traffic that follows.
- **mDNS repeater** — re-advertises services from one VLAN into another so discovery works
  across the split.
- **Not all discovery is mDNS** — Bonjour/mDNS is one protocol family; **SSDP** (and
  DIAL/ECP) is another, used by Roku and many smart TVs.

---

## Research — would the mDNS repeater solve it?

Omada has an mDNS repeater (**Network Config → mDNS**) that advertises a *Service Network*'s
services into a *Client Network*. Before configuring it, I checked what it actually supports:
it bridges **Bonjour/mDNS services only** (AirPlay, Chromecast, Printers, Samba, SSH, etc.).

**Limitation found:** Roku uses **SSDP**, not mDNS. The Omada mDNS repeater doesn't bridge
SSDP, so it can't make a Roku on the IoT VLAN discoverable from Main.

Since the one device that needed cross-VLAN discovery was the Roku, and the repeater couldn't
help it, **no mDNS repeater was configured.** The problem was solved by placement instead
(below). If a Chromecast, AirPlay speaker or network printer is ever moved onto IoT, the
repeater is the tool to reach for.

---

## Design decision — where should the Roku live?

The TV is a Roku, controlled from a phone through the Roku app and used for casting.

| Option | How it works | Trade-off |
|---|---|---|
| **A. Roku on Main** *(chosen)* | Discovery and casting just work — Main can already reach everything | The Roku is trusted, but it's an actively-controlled device, not a set-and-forget gadget |
| B. Roku on IoT, control by IP | Use the Roku app's "connect by IP" against its 192.168.20.x address (allowed by the Main → IoT stateful path) | Needs a DHCP reservation; loses auto-discovery and third-party casting |

**Decision: Option A.** Segmentation effort is best spent on genuinely untrusted devices
(bulbs, plugs, sensors), which stay contained on VLAN 20. Forcing a streaming box that's used
every day onto IoT adds friction without a meaningful security gain.

This is the kind of call that comes up constantly in real environments: the most locked-down
design isn't automatically the best one if people end up working around it.

---

**Previous:** [Phase 3 — Gateway ACLs](03-gateway-acls.md) · **Next:** [Phase 5 — Validation](05-validation.md)
