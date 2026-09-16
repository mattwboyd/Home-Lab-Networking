# Home Lab — Networking

Hands-on documentation of segmenting a flat home network into three isolated
VLANs (Main / IoT / Guest) using a TP-Link Omada stack — covering VLAN and
subnet design, per-VLAN SSIDs, a stateful gateway ACL trust model, and
cross-VLAN service discovery.

Written up as a lab record: what was built, why each decision was made, and
how isolation was validated.

## Contents

- **[omada-vlan-segmentation.md](omada-vlan-segmentation.md)** — full lab
  writeup: VLAN/subnet design, SSID-to-VLAN mapping, gateway ACLs and the
  resulting trust matrix, mDNS/SSDP discovery notes, and a validation
  checklist.

## Environment

- TP-Link Omada Software Controller (v6.2.x)
- 1 gateway (ER605), 1 managed switch (TL-SG2008P), 2 APs (EAP650)

## Key concepts covered

- VLAN segmentation via 802.1Q tagging on shared physical infrastructure
- Omada gateway ACL evaluation (top-down, first-match, implicit allow-all)
  vs. Cisco's implicit deny
- Stateful firewalling to build one-way trust with a single deny rule
- Trunk/tagged vs. access/untagged traffic to APs
- mDNS/Bonjour vs. SSDP/DIAL discovery, and why multicast needs a repeater
  to cross VLANs

## Status

Networking build complete. A NAS/storage project is planned separately.
This repo will be rebuilt in the future with screenshots.
