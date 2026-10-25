# Phase 5 — Validation

**Goal:** Prove the segmentation actually works, not just that the settings were entered.
Testing the *blocked* paths matters as much as testing the allowed ones — and every "blocked"
result needs a baseline showing the target *does* answer when it should.

---

## Results

| Check | Test | Result |
|---|---|---|
| Clients on the right subnets | Omada **Clients** page | ✅ Main → `192.168.0.x`, IoT → `192.168.20.x`, Guest → `192.168.30.x` |
| Roku placement | Clients page | ✅ `55TCLRokuTV-Office` on `192.168.0.126` / `RedLeader` (Main) |
| Guest → Internet | `ping 8.8.8.8` from `RedLeader_Guest` | ✅ 0% loss |
| Guest → Main / IoT | `ping 192.168.0.1` and `192.168.20.1` from `RedLeader_Guest` | ✅ 100% loss — blocked |
| Main → Roku (baseline) | `ping 192.168.0.126` from `RedLeader` | ✅ 0% loss — target answers pings |
| IoT → Internet | `ping 8.8.8.8` from `RedLeader_IoT` | ✅ 0% loss |
| IoT → Main device | `ping 192.168.0.126` from `RedLeader_IoT` | ✅ 100% loss — blocked |
| IoT → gateway's other interfaces | `ping 192.168.0.1` and `192.168.30.1` from `RedLeader_IoT` | ⚠️ **Replies** — see [finding](#finding--iot-could-reach-the-gateway-itself-fixed) |
| IoT → gateway management page | Browse `https://192.168.0.1` from `RedLeader_IoT` | ⚠️ Login page loaded → 🔧 fixed with Rule 5 → ✅ times out on `.0.1`, `.20.1` and `.30.1` |

---

## Evidence

### Clients landed on the right subnets

![Omada Clients page showing devices on RedLeader (192.168.0.x) including the Roku TV, and IoT devices on RedLeader_IoT (192.168.20.x)](../screenshots/05-clients-main-iot.png)

*Main and IoT side by side. Each SSID hands out addresses from its own VLAN: `RedLeader` →
`192.168.0.x`, `RedLeader_IoT` → `192.168.20.x`. Note `55TCLRokuTV-Office` on
`192.168.0.126` / `RedLeader` — the Roku sitting on Main, per the
[Phase 4 decision](04-mdns-and-discovery.md). MAC-address client names are blurred.*

![Omada Clients page showing an iPhone on RedLeader_Guest at 192.168.30.100 alongside IoT devices](../screenshots/05-clients-guest-iot.png)

*A guest device (`iPhone`) on `RedLeader_Guest` received `192.168.30.100` — the first address
in the Guest pool — while the smart-home devices stay on `192.168.20.x`.*

### Guest isolation

![Laptop on RedLeader_Guest: ipconfig shows 192.168.30.101, ping to 8.8.8.8 succeeds, pings to 192.168.0.1 and 192.168.20.1 time out](../screenshots/05-guest-blocked.png)

*A laptop on `RedLeader_Guest` gets `192.168.30.101` (gateway `192.168.30.1`), reaches the
internet (`8.8.8.8`, 0% loss), but gets **100% loss** to both Main (`192.168.0.1`) and IoT
(`192.168.20.1`).*

### IoT isolation — with a baseline

A timeout on its own proves nothing: some devices simply ignore ping. So the same target was
tested from both sides.

![Laptop on RedLeader (192.168.0.109) pinging the Roku at 192.168.0.126 with 0% loss](../screenshots/05-roku-reachable-from-main.png)

*Baseline — from Main (`192.168.0.109`), the Roku at `192.168.0.126` replies. It answers
pings.*

![Laptop on RedLeader_IoT (192.168.20.113) pinging the Roku at 192.168.0.126 with 100% loss](../screenshots/05-iot-blocked.png)

*Same laptop, same target, now on `RedLeader_IoT` (`192.168.20.113`): **100% loss**. The only
thing that changed is the VLAN, so the block is Rule 1 (Block IoT to Main) doing its job.*

---

## Finding — IoT could reach the gateway itself (fixed)

### What was found

While testing from IoT, pings to `192.168.0.1` and `192.168.30.1` — the gateway's addresses
on Main and Guest — **replied**, even though pings to a real Main device were blocked.

![Laptop on RedLeader_IoT: 8.8.8.8 replies, 192.168.0.1 and 192.168.30.1 reply, 192.168.0.126 times out](../screenshots/05-iot-gateway-reachable.png)

*From IoT: the gateway answers on its Main and Guest addresses, but the Roku behind it does
not. Note the replies come back with **TTL=64** — the starting value — meaning no router hop
decremented it. The gateway answered directly; nothing crossed into another VLAN.*

Following that lead, browsing to the gateway from IoT loaded its **admin login page**:

![Laptop on RedLeader_IoT loading the ER605 gateway login page at https://192.168.0.1](../screenshots/05-iot-gateway-login-before.png)

*Before the fix — the ER605's management login, reachable from the IoT network.*

### Why it happened

`192.168.0.1`, `.20.1` and `.30.1` are all interfaces on the *same device* — the gateway.
The Gateway ACLs are **LAN→LAN** rules, which filter traffic the gateway *routes between*
subnets. A request to one of the gateway's own addresses isn't routed anywhere; it's traffic
*to* the gateway, so those rules never see it — even though `192.168.0.1` sits inside the
`Main-Net` IP Group.

**Why Guest didn't show the same thing:** `RedLeader_Guest` has Omada's **Guest Network**
option enabled, which blocks guests from reaching private addresses — gateway included. IoT
doesn't have that option, so it only had the LAN→LAN rules.

**Risk:** host-to-host isolation was intact — IoT couldn't reach any device on Main or Guest.
The exposure was the gateway itself: a compromised IoT device could reach the login page of
the box that runs the whole network and attempt to brute-force it.

### Fix options considered

| Option | Effect | Decision |
|---|---|---|
| Enable **Guest Network** on `RedLeader_IoT` | Blocks all private addresses — but also turns on client isolation (IoT devices can't talk to each other), may break local control from Main, and only covers Wi-Fi | Rejected — too broad |
| Gateway ACL with destination **Gateway Management Page** | Blocks only the management interface; DHCP, DNS and internet untouched | **Chosen** — fixes exactly what was exposed |

### Fix — Rule 5

![Gateway ACL list with Rule 5: Block IoT to Gateway Management, LAN to LAN, Deny, IoT-Net to Gateway Management Page](../screenshots/05-acl-rule5-gateway-mgmt.png)

*Rule 5 — `Block IoT to Gateway Management`: LAN→LAN, Deny, all protocols, `IoT-Net` →
**Gateway Management Page**.*

### Re-test

From `RedLeader_IoT`, the management page now times out on **all three** gateway addresses —
`https://192.168.0.1`, `https://192.168.30.1`, and `https://192.168.20.1` (IoT's own gateway,
the address the old LAN→LAN rules never covered at all).

![Browser on RedLeader_IoT: 192.168.20.1 took too long to respond, ERR_CONNECTION_TIMED_OUT](../screenshots/05-iot-gateway-login-after.png)

*After the fix — the gateway's management page is unreachable from IoT, even on IoT's own
gateway address.*

**Status: ✅ fixed.** The gateway may still answer ping from IoT — that's expected and fine.
IoT devices need the gateway for DHCP, DNS and internet routing; the goal was to stop them
*managing* it, not reaching it.

### Lessons

- **Pinging a gateway address is a misleading isolation test.** Test against a real host on
  the far side, with a baseline from the trusted side.
- **Inter-VLAN rules don't protect the router itself.** Traffic *to* the gateway needs its own
  controls, separate from traffic *through* it.

---

## Concepts reinforced

- **VLAN segmentation** — separate broadcast domains / subnets on shared physical
  infrastructure via 802.1Q tagging
- **ACL evaluation** — top-down first match; implicit allow (Omada gateway) vs. implicit deny
  (Cisco)
- **Routed vs. to-the-device traffic** — LAN→LAN rules filter traffic *through* the gateway,
  not traffic addressed *to* it; management-plane access needs its own rule
- **TTL as a clue** — a reply at TTL=64 came straight from the device, not across a router hop
- **Testing method** — every "blocked" result needs a baseline proving the target responds
- **Stateful firewalling** — return traffic for established sessions is auto-permitted,
  enabling one-way trust with deny rules alone
- **Service discovery protocols** — mDNS/Bonjour vs. SSDP/DIAL, and why multicast discovery
  needs a repeater to cross VLANs

---

**Previous:** [Phase 4 — mDNS & discovery](04-mdns-and-discovery.md) · **Back to:** [README](../README.md)
