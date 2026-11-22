 Configuration

The paste-ready configs for the final L3-core build. Passwords are shown as
placeholders — set your own at deploy time.

Wiring:

| A side | B side | Role |
|---|---|---|
| L3-CORE G0/3 | EDGE-R1 G0/1 | routed /30 — 10.0.0.0/30 |
| L3-CORE G0/2 | SW-ADMIN G0/0 | trunk (10,20,40,50,99,999) |
| L3-CORE G0/0 | SW-FLOOR1 G0/2 | trunk (30,99,999) |
| L3-CORE G0/1 | SW-FLOOR2 G0/2 | trunk (30,99,999) |
| SW-ADMIN G0/1 | HOTEL-SERVER | access VLAN 40 |
| SW-ADMIN G0/2-3 | MANAGER / OFFICE PC | access VLAN 20 |
| SW-ADMIN G1/0-1 | FRONT-DESK PCs | access VLAN 10 |
| SW-ADMIN G2/0 | F&B / POS terminal | access VLAN 50 |
| SW-FLOOR1/2 G0/0-1 | guest APs | access VLAN 30 |
| EDGE-R1 G0/0 | ISP-R1 G0/1 | WAN /30 — 203.0.113.0/30 |
| ISP-R1 G0/0 | PUBLIC-SERVER | 198.51.100.0/24 |

---

## L3-CORE (multilayer switch)

### Base, VLANs, STP root
```
hostname L3-CORE
no ip domain-lookup
enable secret <ENABLE_SECRET>
service password-encryption
banner motd # AUTHORIZED ACCESS ONLY #
spanning-tree mode rapid-pvst
line console 0
 logging synchronous
!
vlan 10
 name FRONT_DESK
vlan 20
 name MANAGEMENT
vlan 30
 name GUEST_WIFI
vlan 40
 name HOTEL_SERVICES
vlan 50
 name F_AND_B
vlan 60
 name VOICE
vlan 80
 name IOT_LOCKS
vlan 99
 name NETWORK_MGMT
vlan 999
 name UNUSED_NATIVE
! make the core the STP root
spanning-tree vlan 10,20,30,40,50,60,80,99,999 root primary
```

### Trunks (encapsulation before mode)
```
interface GigabitEthernet0/2
 description TRUNK_TO_SW-ADMIN
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,40,99,999
 switchport nonegotiate
 no shutdown
!
interface GigabitEthernet0/0
 description TRUNK_TO_SW-FLOOR1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 30,99,999
 switchport nonegotiate
 no shutdown
!
interface GigabitEthernet0/1
 description TRUNK_TO_SW-FLOOR2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 30,99,999
 switchport nonegotiate
 no shutdown
```

### Routing, SVIs, routed link, default route
```
ip routing
interface Vlan10
 description FRONT_DESK_GATEWAY
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface Vlan30
 ip address 192.168.30.1 255.255.255.0
 no shutdown
interface Vlan40
 ip address 192.168.40.1 255.255.255.0
 no shutdown
interface Vlan50
 description F_AND_B_GATEWAY
 ip address 192.168.50.1 255.255.255.0
 no shutdown
interface Vlan60
 ip address 192.168.60.1 255.255.255.0
 no shutdown
interface Vlan80
 ip address 192.168.80.1 255.255.255.0
 no shutdown
interface Vlan99
 ip address 192.168.99.1 255.255.255.0
 no shutdown
interface Vlan1
 shutdown
!
interface GigabitEthernet0/3
 description ROUTED_LINK_TO_EDGE-R1
 no switchport
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.0.1
```
> If `no switchport` is rejected on your IOSvL2 image, make a small transit VLAN with an SVI on the core and an access port to EDGE instead.

### DHCP
```
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20
ip dhcp excluded-address 192.168.50.1 192.168.50.20
ip dhcp pool FRONT_DESK
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 198.51.100.2
 domain-name hotel.local
ip dhcp pool MANAGEMENT
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 198.51.100.2
ip dhcp pool GUEST_WIFI
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 198.51.100.2
ip dhcp pool F_AND_B
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 dns-server 198.51.100.2
```

---

## SW-ADMIN (access switch)
```
hostname SW-ADMIN
no ip domain-lookup
enable secret <ENABLE_SECRET>
service password-encryption
banner motd # AUTHORIZED ACCESS ONLY #
spanning-tree mode rapid-pvst
line console 0
 logging synchronous
!
vlan 10
 name FRONT_DESK
vlan 20
 name MANAGEMENT
vlan 40
 name HOTEL_SERVICES
vlan 50
 name F_AND_B
vlan 99
 name NETWORK_MGMT
vlan 999
 name UNUSED_NATIVE
!
interface GigabitEthernet0/0
 description TRUNK_TO_L3-CORE
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,40,50,99,999
 switchport nonegotiate
 no shutdown
! front desk PCs - VLAN 10
interface range GigabitEthernet1/0 - 1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
! management PCs - VLAN 20
interface range GigabitEthernet0/2 - 3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
! hotel PMS server - VLAN 40
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
! F&B / POS terminal - VLAN 50
interface GigabitEthernet2/0
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface Vlan99
 ip address 192.168.99.11 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1
interface Vlan1
 shutdown
```

---

## SW-FLOOR1 / SW-FLOOR2 (guest-floor access)

Identical except hostname and mgmt IP (`.12` / `.13`). Guest AP ports carry
VLAN 30 — **no port-security** (a bridged AP presents many MACs), but BPDU
Guard is fine (the AP is an end host).

```
hostname SW-FLOOR1
no ip domain-lookup
enable secret <ENABLE_SECRET>
service password-encryption
spanning-tree mode rapid-pvst
!
vlan 30
 name GUEST_WIFI
vlan 99
 name NETWORK_MGMT
vlan 999
 name UNUSED_NATIVE
!
interface GigabitEthernet0/2
 description TRUNK_TO_L3-CORE
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 30,99,999
 switchport nonegotiate
 no shutdown
! AP ports - VLAN 30
interface range GigabitEthernet0/0 - 1
 description GUEST_ACCESS_POINTS
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface Vlan99
 ip address 192.168.99.12 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1
interface Vlan1
 shutdown
```

---

## EDGE-R1 (NAT internet edge)
```
hostname EDGE-R1
no ip domain-lookup
enable secret <ENABLE_SECRET>
service password-encryption
!
interface GigabitEthernet0/1
 description TO_L3-CORE-SW
 ip address 10.0.0.1 255.255.255.252
 ip nat inside
 no shutdown
interface GigabitEthernet0/0
 description TO_ISP-R1
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shutdown
!
ip route 192.168.10.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.0.2
ip route 192.168.30.0 255.255.255.0 10.0.0.2
ip route 192.168.40.0 255.255.255.0 10.0.0.2
ip route 192.168.50.0 255.255.255.0 10.0.0.2
ip route 192.168.99.0 255.255.255.0 10.0.0.2
ip route 0.0.0.0 0.0.0.0 203.0.113.1
!
ip access-list standard NAT_INSIDE
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255
 permit 192.168.40.0 0.0.0.255
 permit 192.168.50.0 0.0.0.255
 permit 192.168.99.0 0.0.0.255
ip nat inside source list NAT_INSIDE interface GigabitEthernet0/0 overload
```
> The per-VLAN static routes here are replaced by a tracked `/16` summary once the HA layer is added — see [high-availability.md](high-availability.md).

---

## ISP-R1 ("the internet")
```
hostname ISP-R1
no ip domain-lookup
interface GigabitEthernet0/1
 description TO_EDGE-R1
 ip address 203.0.113.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/0
 description PUBLIC_INTERNET_NETWORK
 ip address 198.51.100.1 255.255.255.0
 no shutdown
```
> ISP-R1 needs **no route back** to the hotel — EDGE's PAT hides every `192.168.x` behind `203.0.113.2`, so the public server only ever sees that one address. That's exactly how a real hotel's internet connection works.

---

## Linux nodes

**HOTEL-SERVER** (static, VLAN 40):
```
sudo ifconfig eth1 192.168.40.10 netmask 255.255.255.0 up
sudo route add default gw 192.168.40.1 dev eth1
echo "nameserver 198.51.100.2" | sudo tee /etc/resolv.conf
```

**PUBLIC-SERVER** (DNS/web, on the ISP side) — runs dnsmasq + nginx; `ext-conn-0`
gives it real internet for package installs and the dnsmasq upstream.

**PCs (Alpine)** — DHCP via `/etc/network/interfaces` (`iface eth1 inet dhcp`),
then `rc-service networking restart`.

> Confirm each Linux node's data port with `ip link` — if the wired link isn't `eth1`, use `eth0`. Most common CML Alpine/TCL gotcha.
