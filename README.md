# Hotel Enterprise Network — Cisco CML Lab

A resilient enterprise network for a hotel, built by hand in Cisco CML:
a **Layer-3 core doing inter-VLAN routing**, **three-tier segmentation** (guest
/ voice / IoT) enforced with ACLs, a **NAT internet edge**, and a
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
                      ┌──────────┐      ┌──────────┐
   internet  ◄──────► │  ISP-R1  │◄────►│  EDGE-R1 │  NAT (PAT)
   198.51.100.0/24    └──────────┘ /30  └────┬─┬───┘
                                   203.0.113  │ │ 10.0.0.0/30 + 10.0.0.4/30
                                     ┌────────┘ └────────┐
                                ┌────┴────┐         ┌────┴────┐
                                │ L3-CORE │◄═══════►│ L3-CORE2│   HSRP pair
                                │ (active)│ core-trunk (standby)│  STP root 4096/8192
                                └──┬───┬──┘         └──┬───┬──┘
                        ┌──────────┘   └──────┬────────┘   │
                   ┌────┴────┐          ┌─────┴───┐   ┌────┴────┐
                   │ SW-ADMIN│          │SW-FLOOR1│   │SW-FLOOR2│  (dual-homed)
                   └────┬────┘          └────┬────┘   └────┬────┘
              Front desk/Mgmt/PMS       Guest APs V30   Guest APs V30
              V10 / V20 / V40 (.10)
```

Access switches are **dual-homed** to both cores; the **core-to-core trunk**
carries every VLAN and is the L2 path HSRP peers over.

---

## What it demonstrates

| Area | Design |
|---|---|
| **L3 core** | Multilayer switch, SVIs per VLAN, `ip routing`, routed `/30` to the edge |
| **Segmentation** | Three-tier ACL model — guest (internet-only), voice & IoT (server-only, default-deny), proven with live pings |
| **Internet edge** | EDGE-R1 PAT behind one public IP; ISP-R1 + public server as "the internet" |
| **Core HA** | **HSRPv2** per VLAN, priority + preempt, **object tracking** on the uplink; STP root aligned to the active core |
| **WAN HA** | Dual transit to the edge + **IP SLA + tracked static** failover (a bare static won't fail over when the link stays up) |
| **Hardening** | SSH-only with a management ACL, port-security, BPDU Guard, unused ports parked in a black-hole VLAN |
| **Roadmap** | EtherChannel (LACP), dual-ISP, a second edge router for the last SPOF |

---

## Docs

- [`docs/configuration.md`](docs/configuration.md) — the full device configs (L3-CORE, switches, EDGE, ISP, hosts)
- [`docs/security.md`](docs/security.md) — three-tier segmentation, hardening, and the live ping evidence
- [`docs/high-availability.md`](docs/high-availability.md) — HSRP dual-core, tracked-WAN failover, the resilience roadmap, and the failover demo
- [`docs/lessons-learned.md`](docs/lessons-learned.md) — the real gotchas, diagnosed and fixed on live IOS

---

## Security

No real credentials are committed. Device secrets, the enable secret, the Wi-Fi
PSK and the AP password are shown as placeholders (`<ADMIN_SECRET>`,
`<ENABLE_SECRET>`, `<WIFI_PSK>`, `<AP_PASSWORD>`); set your own at deploy time.
The live passwords exist only in the running CML lab.
