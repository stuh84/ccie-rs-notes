# IP Forwarding

* Router rx's frame, checks FCS, drops if errors
* If no errors, check ethertype, extract packet, remove datalink header and trailer
* For v4, header checksum verified. Not in v6
* If dest IP on router, packet arrived, Analyses protocol field to pass to upper drivers
* If not, TTL must be greater than 1, otherwise ICMP Time Exceeded
* Check routing table
* Get interface and next hop router, build new data link frame
* IP header TTL/hop count updated, v4 checksum rebuilt
* New datalink header and trailer

# Processing Switching, Fast Switching, CEF

## Process

All in CPU

## Fast Switching

* 1st packet proc switched
* Above adds to route cache, organized for fst lookups
* Cache has dest ip, next hop and data link header info
* Can overload CPU with lots of packets before cache updated
* Loadbalance per dest only
* Support ended in 12.2(25)S and 12.4(20)T

## CEF

* L2 headers preconstructed (due to same fields always being used for many packets)
* Constructed as routing table constructed (IPs of next hops and ARP/l3 to l2 mapping tables)
* Dest prefixes go in FIB, optimized for fast look ups
* Each entry in FIB has pointer to entry in adj table
* Recursion resolved when creating FIB
* FIB entries for same next hop point to same entry
* Routing table no longer used unless complex processing required
* Routing table is data source for FIB and ADJ
* RIB is master copy of routing info
* If next hop changes, only pointer in FIB needs changing
* FIB contains all known dest prefixes
* FIB contains additional specific entries organized as mtrie (multiway prefix tree)
* CEF in software on lower end routers
* In hardware using TCAM storing FIB contents on higher platforms
* TCAM matches on entire contents in parallel
* CEF distributed to individual line cards
* v4 and v6 have different entries in adj (Proto/ether type different), so different preconstructed headers

**ip cef** - Global enable
**ipv6 cef** - Global enable, v4 cef must also be enabled
**no ip route-cache cef** - Disable per interface

### CEF Load Sharing

* Per Packet - Distributed packet-per-packet
* Per Destination - Source and dest IP and other fields optionally, hashed to identify packet ath

Per dest is default (some hardware CEF doesn't support per packet)

* Per dest has pseudo load share table
* Sits beween Fib and Adj
* Contains up to 16 pointers to entries in adj
* Individual load share entries populate so ratio of adj to cost of parallel routes (so 8 per path when 2 ECMP routes, 5 per path when 3 ECMP routes)
* Per interface command **ip load-share { per-destination | per-packet }

CEF Polarization
* Multiple routers down a pat causing issues
* One router loadshares between two routers
* Next router could hash down one downlink (all met same criteria)
* Combatted with 4B-long number called universal ID
* Univ ID seeds hashing function
* Different routers have different ID (different hashing results)

Load-sharing algorithm multiple options
* Original - Unseeded
* Universal - Seeded
* Tunnel - Smaller outer source/dest parts, so optimized for this
* L4 port - Takes l4 info into account, based on universal

ID and algoritihm specified in **ip cef load-sharing algorithm** and **ipv6 cef load-sharing algorithm**

Cat6500 and others have own workarounds. Set with **mls ip cef load-sharing**. Options are: -
* Default - Source, dest IP and universal ID (if supported in hardware)
* Full (**mls ip cef load-sharing full**)- Source IP/Port and Dest IP/Port, prone to polarization. Only equal across paths if pths are odd.
* Simple - Source an Dest IP, no universal ID
* Full Simple - Not using universal ID, differs from full in all parallel paths recive equal weight, ferwer adj entiresin hardware

# Multilayer Switching

## MLS Logic

* VLAN ints, routed ints and L3 PortChannels
* VLAN SVIs, presented in routing table as egress ints

SVI States
* Admin Down/Line Proto down - SVI int shut
* Down/LP Down - VLAN doesnt exist or in active state
* Up/Down - VLAN exists, but not allowed and in STP forwarding state
* Up/Up

Avoid up/down with either VLAN across at least one trunk, not pruned and STP forwarding, or at least one access/voice port config'd

Switch finds next-hop MACs in CAM

## Routed Ports and port-channels in MLS

* Int uses internal VLAN
* On most cat platforms, can't have sub ints
* No L2 switching info on port
* L3 settings like a router
* Aj table lists outgoing int/port channel, l2 logic not required

VLANs allocated 1006 up, or 4094 down based upon **vlan internal allocation policy { ascending | descending }**

**show vlan internal usage**

These VLANs not in VTP domain, can be different after reboot

```
vlan 11

vlan 12

ip routing

ipv6 unicast-routing distributed

int Gi0/1
 no switchport
 no ip address
 channel-group 1 mode desirable

int Po1
 ip address 192.168.1.1 255.255.255.0

int Fa0/1
 no switchport
 ip address 192.168.2.1 255.255.255.0

int vlan 11
 ip address 192.168.3.1 255.255.255.0
 standby 23 ip 192.168.3.254
 standby 23 piorirty 90
 standby 23 preempt
 standby 23 track Fa0/1
```

# Policy Routing

* Use **ip policy** or **ipv6 policy** on interface
* Route map specifes action
* No match or deny means forwarded by routing
* Match with **match ip address**, **match ipv6 address** or **match length**
* set ip next hop or set ipv6 next hop - Must be in connected subnet, forwars to first address in list for which associated int is up
* set ip(v6) default next hop - As above, except standard routing first (default route ignored)
* set interface - Recommended on p2p only
* set default int - as above, routes first
* set ip df
* set ip(v6) precedenc
* set ip tos

Evaluated in order of set ip next hop, st int, set default ip, set default interface

v6 set interface stills checks for matching route first

On Cat3550, 3560 and 3750, CAM needs repartitioning. Use templates of routing, access or dual-ipv4-and-ipv6-routing. Check with show sdn prefer.

For 3650 and 3850, advanced template

Change template with **sdm prefer template**

# Routing Protocol Changes and migrations

1. Plan strategy
2. Activate new protocol, higher AD than current
3. 
