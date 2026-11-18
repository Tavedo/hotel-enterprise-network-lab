# Lessons Learned

Every one of these was hit and fixed on **live IOS** during the build — the kind
of real troubleshooting that answers an interview question far better than a
clean config does.

## The HA gotchas

| Symptom | Cause | Fix |
|---|---|---|
| HSRP stuck in **Init** | Core-2 SVIs were admin-down | `no shutdown` each SVI |
| `%IP-4-DUPADDR` on a VIP | Core-1 still held static `.1` while Core-2 ran HSRP for `.1` | Finish `.1`→`.2` + HSRP on **all** VLANs before Core-2 joins |
| VLAN 60/80 **split-brain** (both Active) | Default-deny VOICE-IN / IOT-IN dropped the HSRP hellos | `permit udp any host 224.0.0.102 eq 1985` before the denies |
| Native VLAN mismatch warning | New trunks defaulted to native VLAN 1 | `switchport trunk native vlan 999` on **both** ends |
| Internet dies on failover (gateway moved OK) | EDGE had no return route via Core-2 | Dual transit + tracked static |
| Route won't fail over to Core-2 | Static `/24` via Core-1 stayed (CML link stays up); stale specifics beat the summary | IP SLA track + floating `/16`; remove the `/24`s |

### The three worth telling in detail

**1 — Split-brain from my own ACL.**
After adding HSRP to the voice and IoT VLANs, `show standby brief` showed **both
cores Active** on VLANs 60 and 80. The cause was my own doing: the default-deny
`VOICE-IN` / `IOT-IN` ACLs were dropping the HSRP hello packets (destination
`224.0.0.102`, UDP `1985`), so the two cores couldn't see each other and each
assumed it was the only one. The fix is an explicit permit for the hellos
*before* the denies. *"I diagnosed HSRP split-brain from `show standby brief` and
found my own ACL was eating the hellos"* is a senior-level answer.

**2 — A bare static route won't fail over.**
HSRP moved the inside gateway to Core-2 cleanly, but the internet still died on
failover. Two problems: EDGE had no return path via Core-2, and — the subtle one
— a static route **doesn't withdraw when the next-hop device dies but the link
stays up** (which is exactly what CML does when you stop a node). The fix is
**IP SLA with object tracking**: EDGE actively pings the next-hop and pulls the
route when the pings stop, floating traffic to the backup. *"Bare statics don't
fail over when the device dies but the link stays up, so I used IP SLA with
object tracking."*

**3 — Stale specific routes beat the summary.**
Even with the tracked `/16` summary in place, failover didn't happen — because
old per-VLAN `/24` static routes via Core-1 were still in the config. A `/24` is
more specific than a `/16`, so longest-prefix match kept using the dead path.
Removing the stale `/24`s let the tracked summary do its job. Lesson:
**failover only works if nothing more specific overrides the route you're
floating.**

---

## The segmentation gotcha

**A guest-isolation ACL on an appliance VLAN denies the appliance its own
subnet.** Applying `GUEST-IN` inbound on the IoT (door-lock) VLAN meant the lock
pulled DHCP fine but **couldn't ping its own gateway** — yet the internet still
worked, because that traffic fell through to `permit ip any any`. The symptom is
backwards from what you'd expect ("internet works but the gateway doesn't"), and
it's a great reminder that **an inbound ACL filters traffic leaving the subnet,
toward everything — including its own gateway**. Appliances need their own
default-deny policy (gateway + their server, nothing else), not the guest ACL.

The proof was watching it flip live: swapping `GUEST-IN` for `IOT-IN` on the
door-lock VLAN turned the gateway ping from **blocked → working** and the
internet from **working → blocked** at the same instant.

---

## Smaller ones worth remembering

- **Encapsulation before mode** on IOSvL2 trunks: `switchport trunk encapsulation dot1q` must come before `switchport mode trunk`, or the mode command is rejected.
- **`no switchport`** for a routed port isn't supported on every IOSvL2 image — fall back to a transit VLAN + SVI if it's rejected.
- **Linux data port**: CML Alpine/TCL nodes aren't always `eth1` — check `ip link` first; it's the most common "my host won't get DHCP" cause.
- **Don't delete an SVI to remove a policy**: `no interface Vlan60` removes the gateway IP and everything else. Use `interface Vlan60` → `no ip access-group ... in`.
