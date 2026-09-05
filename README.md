# Lab – OSPF Routing with FortiGate LAN/WAN/DMZ Zones

## Objective
Route between a LAN zone (two routers with loopback interfaces behind a switch), a DMZ zone (a third router), and a WAN zone (a router + end host), using a FortiGate firewall as the OSPF backbone for the LAN and DMZ segments, with explicit firewall policies controlling LAN↔DMZ traffic.

## Topology
![Topology](topology.png)

| Device | Zone | Interface | IP | Loopbacks |
|---|---|---|---|---|
| R1 | LAN | f0/0 | 172.16.10.10/24 | Lo1 1.1.1.1/24, Lo2 2.2.2.2/24 |
| R3 | LAN | f0/0 | 172.16.10.20/24 | Lo1 11.11.11.11/24, Lo2 12.12.12.12/24 |
| IOU1 | LAN | e0/0, e0/1, e0/2 | L2 switch connecting R1, R3, FortiGate Port2 |
| FortiGate-VM64-KVM | LAN | Port2 | 172.16.10.1/24 | — |
| FortiGate-VM64-KVM | WAN | Port4 | 192.168.1.1/24 | — |
| FortiGate-VM64-KVM | DMZ | Port3 | 192.168.2.1/24 | — |
| FortiGate-VM64-KVM | MGMT | Port1 | 192.168.229.134/24 | — |
| R4 | DMZ | f0/0 | 192.168.2.2/24 | — |
| R2 | WAN | f0/0, f0/1 | 192.168.1.2/24, 10.10.1.2/24 | — |
| PC3 | WAN | e0 | 10.10.1.1 | — |

## FortiGate Configuration

**Interfaces**
- LAN (port2): `172.16.10.1/255.255.255.0`
- DMZ (port3): `192.168.2.1/255.255.255.0`
- WAN (port4): `192.168.1.1/255.255.255.0`
- MGMT (port1): `192.168.229.134/255.255.255.0`

**OSPF**
- Router ID: `100.100.100.100`
- Area `0.0.0.0` (Regular, no authentication)
- Networks advertised into area 0:
  - `172.16.10.0/24` (LAN)
  - `192.168.2.0/24` (DMZ)

The WAN segment (`192.168.1.0/24`) is intentionally kept outside OSPF — routing to/from the WAN zone is handled separately, and the firewall policy layer controls what's actually allowed between zones regardless of how a route was learned.

**Firewall Policy**
| Name | Source → Destination | Service | Action | NAT |
|---|---|---|---|---|
| DMZ to LAN | DMZ (port3) → LAN (port2), all/all | ALL | ACCEPT | Disabled |
| LAN to DMZ | LAN (port2) → DMZ (port3), all/all | ALL | ACCEPT | Disabled |
| Implicit Deny | all → all | ALL | DENY | — |

![FortiGate OSPF config](fortigate-ospf-config.png)
![FortiGate firewall policy](fortigate-firewall-policy.png)
![FortiGate interfaces](fortigate-interfaces-1.png)
![FortiGate WAN interface](fortigate-interfaces-2-wan.png)

## Router Configuration

**R1**
```
interface Loopback1
 ip address 1.1.1.1 255.255.255.0
interface Loopback2
 ip address 2.2.2.2 255.255.255.0
interface FastEthernet0/0
 ip address 172.16.10.10 255.255.255.0
router ospf 1
 network 172.16.10.0 0.0.0.255 area 0
 network 1.1.1.0 0.0.0.255 area 0
 network 2.2.2.0 0.0.0.255 area 0
```

**R3**
```
interface Loopback1
 ip address 11.11.11.11 255.255.255.0
interface Loopback2
 ip address 12.12.12.12 255.255.255.0
interface FastEthernet0/0
 ip address 172.16.10.20 255.255.255.0
router ospf 1
 network 172.16.10.0 0.0.0.255 area 0
 network 11.11.11.0 0.0.0.255 area 0
 network 12.12.12.0 0.0.0.255 area 0
```

**R4 (DMZ)**
```
interface FastEthernet0/0
 ip address 192.168.2.2 255.255.255.0
router ospf 1
 log-adjacency-changes
 network 192.168.2.0 0.0.0.255 area 0
```

**R2 (WAN)** — static routing, outside the OSPF domain
```
interface FastEthernet0/0
 ip address 192.168.1.2 255.255.255.0
interface FastEthernet0/1
 ip address 10.10.1.2 255.255.255.0
ip route 0.0.0.0 0.0.0.0 FastEthernet0/0
ip route 1.1.1.0 255.255.255.0 192.168.1.1
ip route 172.16.10.0 255.255.255.0 192.168.1.1
```

## Verification
```
R1# show ip ospf neighbor
R3# show ip ospf neighbor
R4# show ip ospf neighbor
R1# show ip route ospf
FortiGate # get router info ospf neighbor
FortiGate # get router info routing-table ospf
PC3> ping 172.16.10.10
```

## Key Takeaways
- Used the FortiGate itself as an OSPF speaker (area 0) to tie together the LAN and DMZ subnets, rather than relying purely on static routes across the firewall.
- Kept the WAN zone outside the OSPF domain and used static routing there instead — a common real-world pattern where you don't want to run a full IGP across a boundary to an external-facing segment.
- Controlled LAN↔DMZ reachability with explicit firewall policies (not just routing), so even though OSPF makes DMZ and LAN mutually reachable at layer 3, actual traffic is still gated by policy.

## What I'd Do Differently
- Scope the DMZ↔LAN firewall policies to specific services/sources instead of `all/all/ALL`.
- Add explicit static (or OSPF-redistributed) routes on R2 for R3's and R4's subnets so PC3 can reach the full LAN/DMZ address space, not just R1's loopback and the LAN subnet.
