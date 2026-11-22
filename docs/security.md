# Segmentation & Hardening

This is where the lab becomes a real hotel network and not just a routing
exercise. Four classes of device, four different policies — and the
difference between them is a genuine design insight, not a copy-paste.

## The four-tier policy

| From | Gateway | VLAN 40 (servers) | Other internal | Internet |
|---|---|---|---|---|
| **Guest (30)** | ✓ | ✗ | ✗ | ✓ |
| **Voice (60)** | ✓ | ✓ PBX | ✗ | ✗ |
| **IoT / lock (80)** | ✓ | ✓ lock server | ✗ | ✗ |
| **F&B / POS (50)** | ✓ | ✓ PMS / payment | ✗ | ✓ card processor |

> *"Guests get internet but no LAN; appliances get their server but no internet;
> nothing moves laterally."* That one sentence shows you understand **why** you
> segment, not just how. Door locks must reach the lock controller but never the
> internet — compromised IoT is a top hotel breach vector.
>
> **F&B / POS breaks the pattern on purpose.** Point-of-sale terminals need
> *both* their PMS/payment server on VLAN 40 **and** the internet (to reach the
> card processor) — but still no lateral movement. So it's neither "guest"
> (internet, no LAN) nor "appliance" (LAN, no internet); it's a fourth policy. In
> production you'd narrow its internet permit to the processor's address — that's
> PCI segmentation in miniature.

### The design rule that's easy to get wrong
A guest-isolation ACL belongs **only** on the guest VLAN. Applying that same
ACL inbound on the voice and IoT SVIs makes those VLANs **deny traffic to their
own subnet** — the door lock pulled DHCP fine but couldn't ping its own gateway
(internet still worked, because that request fell through to `permit ip any
any`). Appliances get their **own default-deny** policy instead: reach the
gateway and their server, nothing else.

---

## GUEST-IN — on Vlan30 only
```
ip access-list extended GUEST-IN
 remark ALLOW DHCP
 permit udp any eq bootpc any eq bootps
 remark BLOCK ALL INTERNAL SEGMENTS
 deny ip any 192.168.10.0 0.0.0.255
 deny ip any 192.168.20.0 0.0.0.255
 deny ip any 192.168.40.0 0.0.0.255
 deny ip any 192.168.50.0 0.0.0.255
 deny ip any 192.168.60.0 0.0.0.255
 deny ip any 192.168.80.0 0.0.0.255
 deny ip any 192.168.99.0 0.0.0.255
 remark BLOCK CORE-EDGE TRANSIT
 deny ip any 10.0.0.0 0.0.0.3
 remark ALLOW INTERNET
 permit ip any any
!
interface Vlan30
 ip access-group GUEST-IN in
```

## VOICE-IN — on Vlan60 (default-deny)
```
ip access-list extended VOICE-IN
 permit udp any eq bootpc any eq bootps
 remark GATEWAY + PHONE-TO-PHONE MEDIA
 permit ip 192.168.60.0 0.0.0.255 192.168.60.0 0.0.0.255
 remark CALL MANAGER / PBX IN HOTEL SERVICES
 permit ip 192.168.60.0 0.0.0.255 192.168.40.0 0.0.0.255
 remark DEFAULT DENY (no lateral, no internet)
 deny ip any 192.168.0.0 0.0.255.255
 deny ip any 10.0.0.0 0.0.0.3
 deny ip any any
!
interface Vlan60
 ip access-group VOICE-IN in
```

## IOT-IN — on Vlan80 (default-deny)
```
ip access-list extended IOT-IN
 permit udp any eq bootpc any eq bootps
 remark GATEWAY + LOCK-TO-LOCK
 permit ip 192.168.80.0 0.0.0.255 192.168.80.0 0.0.0.255
 remark LOCK CONTROLLER / PMS IN HOTEL SERVICES
 permit ip 192.168.80.0 0.0.0.255 192.168.40.0 0.0.0.255
 remark DEFAULT DENY
 deny ip any 192.168.0.0 0.0.255.255
 deny ip any 10.0.0.0 0.0.0.3
 deny ip any any
!
interface Vlan80
 ip access-group IOT-IN in
```

## FB-IN — on Vlan50 (POS: server + payment, no lateral)
```
ip access-list extended FB-IN
 permit udp any eq bootpc any eq bootps
 remark POS-TO-POS LOCAL
 permit ip 192.168.50.0 0.0.0.255 192.168.50.0 0.0.0.255
 remark PMS / PAYMENT SERVER IN HOTEL SERVICES
 permit ip 192.168.50.0 0.0.0.255 192.168.40.0 0.0.0.255
 remark BLOCK ALL OTHER INTERNAL (no lateral)
 deny ip any 192.168.0.0 0.0.255.255
 deny ip any 10.0.0.0 0.0.0.3
 remark ALLOW INTERNET — the card processor (narrow to its IP in production)
 permit ip any any
!
interface Vlan50
 ip access-group FB-IN in
```
The order is the whole point: the two `permit` lines let POS reach its own
subnet and the PMS/payment server **first**, then the `deny 192.168.0.0/16`
blocks every other internal hop, and only **after** that does `permit ip any
any` let card traffic out to the internet. Flip the internet permit above the
deny and POS could suddenly reach the guest VLAN.

> **Build the ACLs before you bind them** — the `ip access-group` lines
> reference ACLs that must already exist. And never `no interface Vlan60`: that
> deletes the whole SVI, gateway IP and all. To drop just a policy, enter the
> interface and `no ip access-group ... in`.

---

## Verified live — CML ping evidence

Pulled from the running CML console logs. This is the difference between "here
is my ACL" and "here is my ACL enforcing, captured from the lab."

**Voice phone — VOICE0 (192.168.60.21):**

| VOICE0 → target | Result | Why |
|---|---|---|
| 192.168.60.1 (gateway) | ✓ 2/2 | own gateway |
| 192.168.40.1 (PMS / PBX) | ✓ 2/2 | VOICE-IN permit to services |
| 192.168.80.1 (IoT) | ✗ 100% loss | no lateral movement |
| 198.51.100.2 (internet) | ✗ 100% loss | default-deny, no internet |

**Door lock — DOOR-LOCK0 (192.168.80.22), before → after the ACL fix:**

| DOOR-LOCK0 → target | Before (GUEST-IN on Vlan80) | After (IOT-IN) |
|---|---|---|
| 192.168.80.1 (gateway) | ✗ 134 lost | ✓ 2/2 |
| 198.51.100.2 (internet) | ✓ reachable | ✗ 100% loss |
| 192.168.40.1 (PMS) | — | ✓ 2/2 |
| 192.168.10.1 (front desk) | — | ✗ 176 lost |

> Internet went **working → blocked** and the gateway went **blocked → working**
> at the exact moment `GUEST-IN` was swapped for `IOT-IN` on Vlan80. That single
> flip is the whole lesson: a guest-isolation ACL on an appliance VLAN denies the
> appliance its own subnet. On `GUEST-IN`, `show access-lists` shows the deny
> counters climbing — isolation you can watch work.

---

## Device hardening

### SSH with a management ACL (login local)
```
username netadmin privilege 15 secret <ADMIN_SECRET>
ip domain-name hotel.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip access-list standard SSH-MANAGEMENT
 permit 192.168.20.0 0.0.0.255
 permit 192.168.99.0 0.0.0.255
 deny any
line vty 0 15
 login local
 transport input ssh
 access-class SSH-MANAGEMENT in
 exec-timeout 10 0
 logging synchronous
```
`transport input ssh` refuses Telnet entirely (Telnet sends the admin password
in plaintext). The `access-class` means only the management and admin VLANs can
even open an SSH session.

### Port-security + unused ports (SW-ADMIN)
```
interface range GigabitEthernet1/0 - 1
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
! park unused ports in the black-hole native VLAN and shut them
interface range GigabitEthernet2/0 - 3
 description UNUSED
 switchport mode access
 switchport access vlan 999
 shutdown
```
> **Not on the AP ports** — a bridged AP presents many MACs (every wireless
> client), which would trip `violation`. BPDU Guard there is fine (the AP is an
> end host).

### Rogue-switch defence — BPDU Guard
```
spanning-tree portfast bpduguard default   ! access ports only — safe on trunks
```
BPDU Guard err-disables any access port that receives a BPDU (which only
switches send) — so a guest plugging a switch/router into a room jack shuts the
port instead of joining the spanning tree. **Never on a trunk**: trunks
legitimately exchange BPDUs, and guarding one would take the uplink down.
