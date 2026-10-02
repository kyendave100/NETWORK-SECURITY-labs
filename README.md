# Hub vs Switch Comparison + Inter-LAN Routing Lab

## Overview

A small lab comparing how a hub and a switch handle traffic differently at Layer 2, then connecting two separate LANs together using a router to test Layer 3 routing between them.

## Problem / Scenario

I wanted to see, hands-on, why switches replaced hubs in real networks — not just know the definition, but actually watch the difference. I built two identical 4-PC LANs: one connected through a hub, one through a switch, each with its own subnet I calculated by hand. After confirming both worked internally, I had two isolated networks with no way to reach each other — so I added a router to connect them and tested whether a device on one LAN could reach a device on the completely different LAN.

## Objectives

- Build two separate LANs, one using a hub and one using a switch
- Calculate subnet addressing by hand for both networks
- Observe and compare how a hub and a switch actually forward traffic
- Connect both LANs using a router, configured as the default gateway for each
- Confirm full connectivity with a cross-network ping

## Network Architecture

- **LAN 1 (hub-based):** 4 PCs connected via straight-through cables to a hub, subnet `192.168.245.0/29`
- **LAN 2 (switch-based):** 4 PCs connected via straight-through cables to a switch, subnet `192.122.87.32/29`
- **Router:** connects both LANs, one interface per network, acting as default gateway for each
  - `GigabitEthernet0/0` → `192.168.245.5` (hub LAN)
  - `GigabitEthernet0/1` → `192.122.87.38` (switch LAN)

## Technologies & Tools

- Cisco Packet Tracer
- Static IP addressing
- Manual subnet calculation (binary, CIDR)
- Cisco IOS CLI (router configuration)

## Implementation

Since I needed 4 usable host addresses on each LAN, I worked out that a `/29` mask (6 usable addresses per subnet) was the right fit — more than enough room for 4 devices, without wasting excessive address space the way a larger subnet would. For each LAN, I calculated the network address, broadcast address, usable range, and subnet mask by hand before touching any configuration screen.

## Configuration

Each PC was configured individually: Desktop → IP Configuration → Static, assigning one address from the usable range, with subnet mask `255.255.255.248`. Hubs and switches don't get IP addresses themselves — only the end devices needed addressing.

For the router, I configured two interfaces (`GigabitEthernet0/0` facing the hub LAN, `GigabitEthernet0/1` facing the switch LAN), each given one of the remaining free addresses from its respective LAN's subnet, to act as the default gateway for that side. Once addressed, I set the default gateway field on every PC to point to the correct router interface for its LAN.

## Testing & Validation

- Pinged between PCs on the hub-side LAN to confirm the hub forwards traffic to every port (seen by watching the packet animation reach all devices, not just the intended one).
- Pinged between PCs on the switch-side LAN to confirm the switch only floods the very first packet, then sends directly to the correct port once it learns the MAC address.
- After configuring the router and setting default gateways on all 8 PCs, pinged from a PC on the hub-side LAN to a PC on the switch-side LAN, confirming traffic successfully crossed from one subnet, through the router, into the completely separate subnet.

## Troubleshooting

**Switch ports stayed orange instead of green:** after connecting the four PCs to the switch, the link lights stayed orange and wouldn't turn green, even after waiting a while. This is caused by Spanning Tree Protocol (STP), which makes a switch port pause for a few seconds before it starts forwarding traffic — hubs don't do this, so I hadn't run into it yet. Switching Packet Tracer to Simulation mode and back to Realtime mode made the ports turn green right away.

**Setting the router's IP address didn't print anything:** after typing the command to give the router's interface an IP address, nothing showed up on screen, so it looked like the command had failed. Running `show ip interface brief` showed the address was actually there — Cisco routers just don't print a message when this command works, so no output actually meant it worked.

**The interface was addressed but still not working:** even with the IP address set, the interface showed as "administratively down." Router interfaces are turned off by default, even after you give them an address, so I had to run `no shutdown` to actually turn it on. After that, `show ip interface brief` showed the interface as fully up.

## Security Considerations

This lab makes the hub-vs-switch security difference concrete rather than theoretical. A hub sends every request and every reply to all connected devices, meaning any device on that network can see traffic that wasn't meant for it — a real confidentiality risk, since no real control exists over who actually receives what. A switch, once it has learned where each device is, sends traffic only to the intended port, keeping other devices on the network from seeing communication that isn't theirs. This is a practical reason switches replaced hubs almost entirely in real networks, beyond just performance.

## Results

Both LANs worked internally, and the hub/switch behavior difference was directly observed, not just assumed. After configuring the router with one interface on each network and setting default gateways on every PC, a ping from a hub-side PC successfully reached a switch-side PC — confirming full routing between two independently addressed subnets.

## Screenshots / Evidence

*(Add screenshots here — suggested: full topology view, `show ip interface brief` output, successful cross-network ping result)*

- `evidence/topology.png`
- `evidence/router-interfaces.png`
- `evidence/cross-network-ping.png`

## What I Learned

- How to configure a router with multiple interfaces, each acting as the default gateway for a different LAN
- The difference between straight-through and crossover cables, and when each is actually needed
- The real, observable difference between how hubs and switches handle traffic — not just the definition, but watching it happen

## Challenges

Early on, while calculating one of the `/29` subnets, I first wrote out the usable range incorrectly (stopping two addresses short of the actual range) before catching and correcting the error by re-checking the block boundaries.

## Improvements / Future Work

If this were a real organization's network, I'd want to add internet access and deploy actual servers (web, email, file sharing) so the network does real work, not just connects PCs to each other. I'd also add firewalls to control and filter traffic between zones, and look at encrypting communications and securing device configurations, rather than leaving everything in plain, unprotected form.
