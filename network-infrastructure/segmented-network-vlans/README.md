# Segmented Office Network: VLANs, Trunks, and Inter-VLAN Routing

## Overview

A small office network built in Cisco Packet Tracer where HR, Finance, and IT each get their own VLAN and subnet. I started with one switch and two VLANs, grew it to two switches joined by a trunk, and then made the departments able to talk to each other in two different ways: router-on-a-stick and a Layer 3 switch. I also broke things on purpose and fixed them, and kept the evidence.

## Problem / Scenario

In a flat network, every device shares one broadcast domain. An HR PC and a Finance PC sit on the same segment, hear each other's broadcasts, and can reach each other without any control. I wanted to see for myself how VLANs fix that, what stops working when you split a network up, and what you need to add to let the departments talk again.

## Objectives

- Split one switch into separate VLANs and prove they are isolated
- Carry several VLANs between two switches over one trunk
- Route between VLANs using a router (router-on-a-stick)
- Route between VLANs using a Layer 3 switch (SVIs)
- Troubleshoot VLAN and routing faults from evidence, not guesses

## Network Architecture

**Addressing plan**

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | HR | 192.168.10.0/24 | 192.168.10.1 |
| 20 | FINANCE | 192.168.20.0/24 | 192.168.20.1 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 |

I kept `.1` free in every subnet for the gateway, and gave PCs addresses from `.11` upward.

**Router-on-a-stick build (Day 32):** two 2960 switches, one 2901 router, nine PCs.

| Switch | Ports | VLAN |
|---|---|---|
| SW1 | Fa0/1-2 | 10 (HR) |
| SW1 | Fa0/3-4 | 20 (FINANCE) |
| SW1 | Fa0/5 | 30 (IT) |
| SW2 | Fa0/1 | 10 (HR) |
| SW2 | Fa0/2 | 20 (FINANCE) |
| SW2 | Fa0/3-4 | 30 (IT) |

Trunks: SW1 `Fa0/24` to SW2 `Fa0/24`, and SW1 `Gig0/1` to the router's `Gig0/0`. Only SW1 connects to the router, and SW2 reaches it through the trunk between the switches.

## Technologies & Tools

Cisco Packet Tracer, 2960 switches, a 2901 router, a 3560 multilayer switch, 802.1Q trunking, static IP addressing, Cisco IOS CLI.

## Files in this folder

| File | What it is |
|---|---|
| `day28-29-vlans.pkt` | one switch, two VLANs, and the troubleshooting drills |
| `day30-three-vlan-plan.pkt` | the three-VLAN plan built on one switch |
| `day31-trunk.pkt` | two switches joined by a trunk |
| `day32-inter-vlan.pkt` | router-on-a-stick |
| `day33-layer3-switch.pkt` | the Layer 3 switch version |

## Implementation

**Stage 1: one switch, two VLANs (Day 28).** I made HR (VLAN 10) and FINANCE (VLAN 20) on one switch with two PCs each. PCs in the same VLAN could ping each other, and PCs in different VLANs could not.

**Stage 2: a three-VLAN plan (Day 30).** Before building, I wrote a plan table: VLAN IDs, names, subnets, ports, PC addresses, and a gateway reserved for later. I then built HR, FINANCE, and IT with three PCs each on one switch.

**Stage 3: a trunk between two switches (Day 31).** I added a second switch with its own HR and Finance PCs and joined the two with a cable on `Fa0/24`. At first the ports were left on their default, and HR PCs on different switches could not ping each other. Setting the port to trunk fixed that.

**Stage 4: router-on-a-stick (Day 32).** A router joined to SW1 by one trunk, with one subinterface per VLAN. Each subinterface holds the gateway address for its VLAN.

**Stage 5: a Layer 3 switch (Day 33).** On a 3560, I created an SVI (Switch Virtual Interface) for each VLAN and let the switch do the routing itself, with no separate router. I haven't compared its speed against the router version.

## Configuration

**VLANs and access ports (same pattern on every switch)**

```
vlan 10
 name HR
vlan 20
 name FINANCE
vlan 30
 name IT
interface range fastEthernet 0/1 - 2
 switchport mode access
 switchport access vlan 10
```

**Trunk**

```
interface fastEthernet 0/24
 switchport mode trunk
```

**Router subinterfaces**

```
interface gigabitEthernet 0/0
 no shutdown
interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
```

The same pattern repeats for VLANs 20 and 30. The physical port gets no IP address. The `encapsulation` line has to come before `ip address`.

**Layer 3 switch (SVIs)**

```
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
```

On my Packet Tracer 3560, `ip routing` was already switched on by default. I confirmed it with `show running-config | include routing`.

## Testing & Validation

| Test | Result |
|---|---|
| Same VLAN, same switch | success, TTL 128 |
| Different VLANs, no router yet | timed out |
| Same VLAN across the trunk, before trunking | timed out |
| Same VLAN across the trunk, after trunking | success, TTL 128 |
| Router-on-a-stick: HR to Finance | success, TTL 127 |
| Router-on-a-stick: Finance to IT (two different pairs of PCs) | success both times, TTL 127, the first ping timing out each time |
| Router-on-a-stick: PC to its own gateway | success, TTL 255 |
| Layer 3 switch: PC to its SVI gateway | success, TTL 255 |
| Layer 3 switch: HR to Finance | success, TTL 127 |

TTL tells you how many routing hops a packet crossed. A PC starts at 128, so 128 means no router, 127 means one routing hop, and 255 means a ping to the router or SVI itself. In several routed tests, the first ping timed out while ARP resolved the addresses.

I also checked the build with `show vlan brief`, `show interfaces trunk`, `show ip interface brief`, and `show ip route`.

**The path between two PCs on the same switch (Simulation mode).** In the router-on-a-stick lab, I pinged between two PCs on SW2 that sit in different VLANs. In Simulation mode I watched the packet go from SW2 up to SW1, then to the router, then back to SW1 and down to SW2. So even though the two PCs are on the same switch, the traffic travels out to the router and back over the same links. A ping can't show this, because the TTL is 127 either way, and the capture is what proved it. The cost of this design is that all traffic between VLANs shares those links, and if SW1 or the router link fails, the VLANs can't reach each other.

**Layer 3 switch experiment.** I turned routing off with `no ip routing` and the cross-VLAN ping failed, and `show ip route` stopped listing the connected subnets. Turning it back on restored it. That made it clear that routing is what lets the VLANs talk.

## Troubleshooting

**1. A port shut down.** *Symptom:* a PC could not ping its neighbor, and its link light was red. *Test:* `show interfaces status`. *Evidence:* the port showed `disabled`. *Fix:* `no shutdown`. *Verification:* the ping succeeded.

**2. A PC in the wrong VLAN.** *Symptom:* a PC could not ping others in its own subnet. *Test:* `show vlan brief`. *Evidence:* its port was in the wrong VLAN. *Fix:* put the port back with `switchport access vlan`. *Verification:* ping succeeded.

**3. A VLAN deleted while ports still belonged to it.** After `no vlan 10`, the HR ports disappeared from `show vlan brief`, but `show interfaces status` still showed them as `connected` in VLAN 10, and pings failed. Recreating the VLAN brought the ports back. The lesson: a port can look fine and still be dead, so check that the VLAN exists.

**4. Wrong VLAN on a Finance port (Layer 3 switch lab).** *Symptom:* the Finance PC could not ping its gateway, a same-VLAN neighbor, or an HR PC. *Evidence:* `Fa0/3` was under VLAN 10 in `show vlan brief` and `connected` in `show interfaces status`. *Root cause:* the port was assigned to VLAN 10 instead of VLAN 20, so the PC sat away from its gateway and its neighbor. *Fix:* `switchport access vlan 20` on that port. *Verification:* `show vlan brief` showed the port under VLAN 20, and pings returned TTL 255 (gateway), 128 (neighbor), and 127 (other VLAN).

## Security Considerations

- **VLANs limit how far a broadcast travels,** which also limits how far problems like ARP spoofing can spread. They are not a complete security boundary on their own.
- **Trunk negotiation is a risk.** My switches defaulted to a mode where a port becomes a trunk if the other end asks. When I set only SW1 to trunk, SW2's port also turned into a trunk without my typing anything on it. A device pretending to be a switch could do the same. Setting a port to `switchport mode access` turned negotiation off in my test, which is why user ports should always be set explicitly.
- **VLAN 1 is the default for everything:** unused ports, the native VLAN, and the default management VLAN. I left it unchanged in this lab, and in a real network I'd move it out of use.
- **Inter-VLAN routing has no access control yet.** Once the router or Layer 3 switch is routing, every VLAN can reach every other VLAN. Restricting that needs ACLs, which I haven't built yet.

## Results

HR, Finance, and IT are isolated from each other by VLANs, share two switches through a trunk, and can talk through either a router or a Layer 3 switch. I confirmed each stage with `show` commands and pings, watched the router-on-a-stick traffic path in Simulation mode, and fixed four faults by following the evidence.

## Screenshots / Evidence

*TODO: screenshots still to be taken from the saved files, then added with `![description](screenshots/name.png)` once uploaded.*

| Screenshot | From which lab |
|---|---|
| Topology | `day32-inter-vlan.pkt` |
| `show vlan brief` on both switches | `day32-inter-vlan.pkt` |
| `show interfaces trunk` | `day32-inter-vlan.pkt` |
| `show ip interface brief` and `show ip route` on the router | `day32-inter-vlan.pkt` |
| Cross-VLAN ping with TTL 127 | `day32-inter-vlan.pkt` |
| The Simulation-mode capture of the same-switch path | `day32-inter-vlan.pkt` |
| Topology and `show ip route` with routing on and off | `day33-layer3-switch.pkt` |
| The wrong-VLAN fault and its fix | `day33-layer3-switch.pkt` |

## What I Learned

- A VLAN is a broadcast boundary, and a subnet has to match it. Fixing only one of them isn't enough.
- A trunk carries many VLANs by tagging frames with 802.1Q. The tag is added going out a trunk port and removed going out an access port.
- Routing between VLANs needs something that routes: a router with subinterfaces, or a Layer 3 switch with SVIs.
- With router-on-a-stick, two PCs on the same switch but in different VLANs still send their traffic out to the router and back.
- TTL is a useful clue, but it can't show the path a packet took. For that I needed Simulation mode.
- Two outputs can look fine while the network is broken, so I check the evidence from more than one command before trusting it.

## Challenges

The hardest part was telling similar faults apart. A wrong VLAN, a wrong gateway, and a shut-down port can all show up as a failed ping. What helped was running the pings in a fixed order (gateway, same-VLAN neighbor, other VLAN), because where it stops working points at the cause.

## Improvements / Future Work

- Add the baseline security I learned separately (hostnames, console and enable passwords, password encryption, warning banners). I did not set these on the devices in this lab
- Add DHCP, so PCs get their addresses automatically
- Use an unused VLAN as the native VLAN, and allow only the needed VLANs on each trunk
- Shut down unused ports and set them to access mode
- Add ACLs, so departments can only reach what they need
- Add a second path between switches, with spanning tree managing the loop
- Add a second core device, since one Layer 3 switch is a single point of failure
