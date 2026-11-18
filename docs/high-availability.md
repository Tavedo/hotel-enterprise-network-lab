# High Availability & Redundancy

A single core, a single uplink, a single ISP = single points of failure. This
layer removes them: a second core with a **floating gateway (HSRP)**, every
access switch **dual-homed**, and **two paths out** to the internet. The
principle is aggregation / redundancy — if one device dies, the other carries
the load with no human intervention.

## Single core → dual core

| Added | Purpose |
|---|---|
| **L3-CORE-2** | Mirror of Core-1 (same VLANs, SVIs, ACLs) — **minus DHCP** |
| **Core-to-core trunk** (Gi1/0 ↔ Gi1/0) | Carries all VLANs — the L2 path HSRP peers over |
| **Second uplink per access switch** | SW-FLOOR1/2 & SW-ADMIN reach both cores |
| **Second transit to EDGE** (10.0.0.4/30) | Core-2's own routed path out |

Core-2 is identical to Core-1 **except four things**: (1) no DHCP pools (two
servers = address conflict), (2) its own transit `10.0.0.6/30` + default route
via `10.0.0.5`, (3) STP priority `8192` (not `4096`), (4) HSRP physical IP `.3`
and priority `90`.

---

## HSRP — the floating gateway

Each VLAN gateway becomes a **virtual IP** that lives on whichever core is
active. Clients never change — they still point at `192.168.x.1`. Physical IPs:
Core-1 `.2`, Core-2 `.3`, VIP `.1`.

**L3-CORE (active, priority 110):**
```
track 1 interface GigabitEthernet0/3 line-protocol
!
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 track 1 decrement 30
! repeat per VLAN (20,30,40,60,80,99) with matching group numbers
spanning-tree vlan 10,20,30,40,60,80,99 priority 4096
```

**L3-CORE-2 (standby, priority 90):**
```
interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 90
 standby 10 preempt
! repeat per VLAN
spanning-tree vlan 10,20,30,40,60,80,99 priority 8192
```

**The object track is the point:** `track 1` watches the uplink to EDGE
(`Gi0/3`). If it drops, the active core's priority decrements by 30 (110 → 80),
falling below Core-2's 90, so the gateway fails over **before** traffic
black-holes — not just when the whole switch dies.

> **Order matters:** convert Core-1 first (it owns VIP `.1` as the sole member),
> then Core-2 joins as standby — otherwise Core-1's static `.1` collides with
> Core-2's VIP. Verify: `show standby brief` → every VLAN Core-1 **Active** /
> Core-2 **Standby**.

---

## North-south redundancy (dual transit + tracked static)

HSRP moves the inside gateway to Core-2, but **EDGE still needs a way to send
replies back through Core-2**. And a bare static route won't fail over — in CML
the link stays "up" even when the far node is stopped, so the route never
withdraws. The fix: **IP SLA + a tracked static** — EDGE actively pings Core-1
and pulls the route when it stops answering.

```
interface GigabitEthernet0/2
 description INSIDE-TO-CORE2
 ip address 10.0.0.5 255.255.255.252
 ip nat inside
 no shutdown
!
ip sla 1
 icmp-echo 10.0.0.2 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
!
ip route 192.168.0.0 255.255.0.0 10.0.0.2 track 1   ! primary, via Core-1
ip route 192.168.0.0 255.255.0.0 10.0.0.6 10        ! floating backup, via Core-2
```
> Remove any old per-VLAN `ip route 192.168.x.0 /24 via 10.0.0.2` — a `/24` is
> more specific than the `/16` summary and would never fail over. Both
> `ip nat inside` (on G0/1 **and** G0/2) and `ip nat outside` (G0/0) are required.

---

## Prove the failover (the trophy demo)

```
! 1. continuous ping from a guest/front-desk client
ping 198.51.100.2 -t
! 2. fully STOP L3-CORE in CML
! 3. watch: ~1-3 packets lost, then it keeps flowing
!    (RSTP unblocks the standby uplink + HSRP moves the VIP
!     + IP SLA withdraws the EDGE route to Core-2 — all at once)
! 4. confirm:
show standby brief            ! Core-2 now Active on all VLANs
show ip route 192.168.30.0    ! on EDGE: now via 10.0.0.6
! 5. start Core-1 -> it preempts back to Active; track recovers;
!    EDGE route reverts to via 10.0.0.2
```
Worth capturing for the portfolio: `show standby brief` (the Active/Standby
pair), `show spanning-tree vlan 30` (the `Altn BLK` standby uplink), and the
ping losing 1–2 packets through a core failure.

---

## Roadmap — next-level resilience

Every item follows one rule: **bundle or duplicate so one failure is
survivable.** Add them one at a time.

- **EtherChannel (LACP)** — bundle two links between a switch and a core into one `Port-channel`: more bandwidth, and losing a member doesn't reconverge STP. Highest ROI.
  ```
  interface range Gi0/2 - 3
   channel-group 1 mode active
  interface Port-channel1
   switchport trunk allowed vlan 30,60,80,99,999
  ```
- **Dual-ISP** — a second WAN uplink with two default routes, the primary tracked by IP SLA so it floats to the backup automatically (same pattern as the inside routes).
- **Second edge router** — the last SPOF is EDGE itself; a second edge with the cores pointing at a shared VIP (HSRP/VRRP on the transit) keeps the internet flowing even when one edge dies.

> For an interview, being able to **describe** these ("HSRP on gateways, LACP on
> uplinks, IP SLA for WAN failover, a second edge for the last SPOF") is worth as
> much as building every one.
