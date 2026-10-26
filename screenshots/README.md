# Screenshots

Drop proof screenshots here using these filenames, then un-comment the matching
`<!-- Screenshot: ... -->` line in the phase doc so the image renders on GitHub.
The verification shots (Phase 5) matter most.

| Filename | What it should show | Referenced in |
|---|---|---|
| `00-topology.png` ✅ | Omada topology — WAN → ER605 → TL-SG2008P → APs / controller (MACs + device names removed) | README |
| `01-vlan-list.png` ✅ | Omada VLAN list — Default (1), RedLeader_IoT (20), RedLeader_Guest (30) with subnets | Phase 1 |
| `02-ssid-list.png` ✅ | SSID list — RedLeader, RedLeader_IoT, RedLeader_Guest | Phase 2 |
| `02-ssid-vlan-binding.png` ✅ | RedLeader_IoT → Advanced Settings → VLAN bound by network to RedLeader_IoT | Phase 2 |
| `03-gateway-acl-rules.png` ✅ | Gateway ACL list with the four deny rules | Phase 3 |
| `05-clients-main-iot.png` ✅ | Clients page — Main (incl. Roku) on 192.168.0.x, IoT on 192.168.20.x | Phase 5 |
| `05-clients-guest-iot.png` ✅ | Clients page — guest iPhone on 192.168.30.x | Phase 5 |
| `05-roku-reachable-from-main.png` ✅ | Baseline — Roku (192.168.0.126) replies to ping from Main | Phase 5 |
| `05-iot-blocked.png` ✅ | Same Roku ping from IoT (192.168.20.113) — 100% loss | Phase 5 |
| `05-iot-gateway-reachable.png` ✅ | Finding — gateway's 0.1 / 30.1 addresses reply from IoT | Phase 5 |
| `05-iot-gateway-login-before.png` ✅ | Before fix — ER605 login page loads from IoT (profile picture blurred) | Phase 5 |
| `05-acl-rule5-gateway-mgmt.png` ✅ | Attempt 1 — Rule 5 with all protocols | Phase 5 |
| `05-rule5-dns-blocked.png` ✅ | Diagnosis — DNS via gateway times out, via 8.8.8.8 works | Phase 5 |
| `05-rule6-attempt-config.png` ✅ | Attempt 2 — IP-Port Group rule config | Phase 5 |
| `05-rule6-attempt-login-loads.png` ✅ | Attempt 2 — login page still loads from IoT | Phase 5 |
| `05-rule5-tcp-config.png` ✅ | Attempt 3 — final Rule 5, TCP only | Phase 5 |
| `05-rule5-tcp-dns-ok.png` ✅ | Regression check — DNS and gateway ping work from IoT | Phase 5 |
| `05-iot-gateway-login-after.png` ✅ | After final fix — 192.168.20.1 management page times out from IoT | Phase 5 |
| `05-acl-final.png` ✅ | Final Gateway ACL list (Rules 1–5) | Phase 5 |
| `05-guest-blocked.png` ✅ | Guest device failing to reach Main/IoT | Phase 5 |

**Tips**
- PNG keeps text crisp; crop to just the relevant panel.
- Black out real SSID names, public IPs, MAC addresses and device names you'd rather not publish.
- If you rename a file, update the matching reference in the phase doc.
