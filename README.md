# Hotel Enterprise Network — Cisco CML Lab

A resilient enterprise network for a hotel, built by hand in Cisco CML:
a **Layer-3 core doing inter-VLAN routing**, **four-tier segmentation** (guest
/ voice / IoT / F&B) enforced with ACLs, a **NAT internet edge**, and a
**dual-core HSRP high-availability** layer with **tracked-static WAN failover**.

The focus is **segmentation and high availability** the way a real property
needs it: guests get internet but never the hotel LAN, appliances reach their
server but never the internet, nothing moves laterally — and if a core switch,
an uplink, or an ISP dies, the network keeps running with no human touch.

Every device was configured and verified individually, then the design was
extended with the security and HA layers. The gotchas in
[`docs/lessons-learned.md`](docs/lessons-learned.md) were all hit and fixed on
live IOS.

---

## Final design — VLANs & addressing (L3-core)

| VLAN | Name | Subnet | Gateway (VIP) | DHCP |
|---|---|---|---|---|
| 10 | FRONT_DESK | 192.168.10.0/24 | 192.168.10.1 | Yes |
| 20 | MANAGEMENT | 192.168.20.0/24 | 192.168.20.1 | Yes |
| 30 | GUEST_WIFI | 192.168.30.0/24 | 192.168.30.1 | Yes |
| 40 | HOTEL_SERVICES (PMS) | 192.168.40.0/24 | 192.168.40.1 | Static (.10) |
| 50 | F&B / POS | 192.168.50.0/24 | 192.168.50.1 | Yes |
| 60 | VOICE | 192.168.60.0/24 | 192.168.60.1 | Yes |
| 80 | IOT / door locks | 192.168.80.0/24 | 192.168.80.1 | Yes |
| 99 | NETWORK_MGMT | 192.168.99.0/24 | 192.168.99.1 | Static (sw .11/.12/.13) |
| 999 | UNUSED_NATIVE | — | — | native on all trunks |

Gateways are **HSRP virtual IPs** (`.1`) floating across two cores; the physical
cores hold `.2` (active) and `.3` (standby). Transit to the edge is `10.0.0.0/30`
(Core-1) and `10.0.0.4/30` (Core-2); the WAN is `203.0.113.0/30`.

---

## Topology

```
  internet ── ISP-R1 ── EDGE-R1 ──┬─ G0/1 ─► L3-CORE        NAT (PAT),
                                  └─ G0/2 ─► L3-CORE-2       dual transit

         L3-CORE ◄═══ core trunk (G1/0 ↔ G1/0) ═══► L3-CORE-2
         (active)                                   (standby)

  Each access switch has one uplink to EACH core (dual-homed).
  The access switches do NOT connect to each other.

     SW-ADMIN    → L3-CORE G0/2  +  L3-CORE-2 G1/2    (V10/20/40/50, F&B on G2/0)
     SW-FLOOR1   → L3-CORE G0/0  +  L3-CORE-2 G0/0    (APs V30, voice V60, lock V80)
     SW-FLOOR2   → L3-CORE G0/1  +  L3-CORE-2 G0/1    (APs V30, voice V60, lock V80)
```

Every switch — and the edge — reaches **both cores**; the **core-to-core trunk**
carries every VLAN and is the L2 path HSRP peers over. The exact uplinks:

| Link | A side | B side |
|---|---|---|
| Core trunk | L3-CORE G1/0 | L3-CORE-2 G1/0 |
| EDGE transit 1 | EDGE-R1 G0/1 | L3-CORE G0/3 |
| EDGE transit 2 | EDGE-R1 G0/2 | L3-CORE-2 G0/2 |
| SW-ADMIN | G0/0 → L3-CORE G0/2 | G1/2 → L3-CORE-2 G1/2 |
| SW-FLOOR1 | G0/2 → L3-CORE G0/0 | G1/0 → L3-CORE-2 G0/0 |
| SW-FLOOR2 | G0/2 → L3-CORE G0/1 | G1/0 → L3-CORE-2 G0/1 |

---

## What it demonstrates

| Area | Design |
|---|---|
| **L3 core** | Multilayer switch, SVIs per VLAN, `ip routing`, routed `/30` to the edge |
| **Segmentation** | Four-tier ACL model — guest (internet-only), voice & IoT (server-only), F&B/POS (PMS + payment, no lateral), all default-deny, proven with live pings |
| **Internet edge** | EDGE-R1 PAT behind one public IP; ISP-R1 + public server as "the internet" |
| **Core HA** | **HSRPv2** per VLAN, priority + preempt, **object tracking** on the uplink; STP root aligned to the active core |
| **WAN HA** | Dual transit to the edge + **IP SLA + tracked static** failover (a bare static won't fail over when the link stays up) |
| **Hardening** | SSH-only with a management ACL, port-security, BPDU Guard, unused ports parked in a black-hole VLAN |
| **Roadmap** | EtherChannel (LACP), dual-ISP, a second edge router for the last SPOF |

---

## Docs

- [`docs/configuration.md`](docs/configuration.md) — the full device configs (L3-CORE, switches, EDGE, ISP, hosts)
- [`docs/security.md`](docs/security.md) — four-tier segmentation, hardening, and the live ping evidence
- [`docs/high-availability.md`](docs/high-availability.md) — HSRP dual-core, tracked-WAN failover, the resilience roadmap, and the failover demo
- [`docs/lessons-learned.md`](docs/lessons-learned.md) — the real gotchas, diagnosed and fixed on live IOS

---

## Security

No real credentials are committed. Device secrets, the enable secret, the Wi-Fi
PSK and the AP password are shown as placeholders (`<ADMIN_SECRET>`,
`<ENABLE_SECRET>`, `<WIFI_PSK>`, `<AP_PASSWORD>`); set your own at deploy time.
The live passwords exist only in the running CML lab.
