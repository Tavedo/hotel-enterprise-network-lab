# Hotel Enterprise Network — Cisco CML Lab

A resilient enterprise network for a hotel, built by hand in Cisco CML:
**dual-core HSRP high availability**, segmented VLANs with default-deny
security, **dual-WAN / dual-ISP failover**, and a site-to-site IPsec VPN.

The focus is **high availability and segmentation** done the way a real
property would need it: if a core switch, a WAN link, or an ISP dies, guests
keep their Wi-Fi, the front desk keeps working, and the door locks keep talking.

Every device was configured and verified individually, then the design was
extended with the HA and redundancy layers below.

> Full build walkthrough — design, configs, and verification, including the HA
> & redundancy section: [`docs/hotel-cml-lab.html`](docs/hotel-cml-lab.html)
> (open in a browser).

---

## What it demonstrates

| Area | Design |
|---|---|
| **Core HA** | Dual L3 cores with **HSRPv2** per VLAN — virtual IP gateways, priority + preempt, **interface/object tracking** so the gateway follows the live uplink |
| **L2 resilience** | `rapid-pvst`, STP root aligned to the HSRP active core (priority 4096 / 8192) so Layer 2 and Layer 3 agree on the primary |
| **Segmentation** | Separate VLANs for Guest, Voice, and IoT / door-locks, inter-VLAN routing on SVIs, native VLAN 999 |
| **Security** | **Default-deny ACLs** (GUEST-IN / VOICE-IN / IOT-IN) — guests reach the internet but not each other or hotel systems; voice and IoT are locked down |
| **WAN HA** | Dual edge routers, dual ISP, **IP SLA + tracked static routes** for fast failover, NAT |
| **Site-to-site** | IPsec VPN to a remote site |

---

## The HA ideas worth knowing

- **HSRP gateway tracking** — each core owns the virtual gateway IP for its VLANs with higher priority; an object tracks the uplink, and if it drops the priority decrements below the standby, so the gateway fails over *before* traffic black-holes.
- **STP ↔ HSRP alignment** — the HSRP active core is also the STP root, so the Layer 2 forwarding path and the Layer 3 gateway are the same device. Misaligning them sends traffic across the inter-switch link on every packet.
- **The ACL-eats-HSRP gotcha** — a default-deny ACL silently drops HSRP hellos (`224.0.0.102`, UDP 1985) and both cores think they're active (split-brain). The fix is an explicit permit before the denies — debugged live.
- **Tracked-static WAN failover** — `ip sla` pings the primary next-hop; a tracked static route withdraws when it fails, floating a backup route via the second ISP.

---

## Layout

```
hotel-enterprise-network-lab/
├── README.md
└── docs/
    └── hotel-cml-lab.html        # full build guide (design + HA + verification)
```

---

## Notes

- Built and verified by hand in Cisco CML, device by device.
- Credentials in the guide are shown as `CHANGE_ME` placeholders — set your own; nothing sensitive is committed.
- Automation (config-as-code) is a planned next step, to be added once built and validated.
