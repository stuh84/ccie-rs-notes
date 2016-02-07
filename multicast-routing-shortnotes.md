# Overview of Multicast Routing Protocols

## Using Dense Mode

* Assume group is popular, every subnet needs it
* Forwarded on all ints, except loop prevented ints
* M'cast never forwarded on receied int
* Do allow to ask to not receive traffic
* Traffic not wanted if no downstream routers need packets for that
  group, and no hosts on directly connected subnets
* Prune upstream to stop
* DVMRP, PIM-DM and MOSPF are DM

### RPF Check

* Looks at source of m'cast packet
* If route for source is on outgoing interface that packet received,
  passed. If not, do not replicated and forwarded
* Shortest path packets accepted, longer route packets discarded
* Can't use dest addresses to help routers forward packets

RPF Ints determined depending on protocol: -

* DVMRP has separate m'cast routing table, uses it for RPF check
* PIM and CBT (Core Based Tree) uses unicast table
* PIM and CBT can ue DVMRP, MP-BGP, or static configured m'cast routes
  for RPF check
* MOSPF doesn't use RPF, computes forward and reverse shortest path
  source-rooted trees

## Multicast Forwarding using Sparse Mode

* Doesn't forward unless request for traffic
* Requests on if router receives from downstream router, or direct host
  sends IGMP Join for group
* PIM SM fowards packets to RP
* When traffic at RP, RP forwars to those requesting for group
* RP address usually loopback
* Need to learn IP of RP, static or dynamically found
* RPF etermines source interface/route of RP, rather than source of
  packet

## Multicast Scoping

* Boundaries created

### TTL Scoping

* Compares TTL with TTL on outgoing interface
* If config'd TTL less than outgoing intrface, forwarded
* Default TTL on Ciscos of 0
* Difficult to guess correct TTL
* Applies to all multicast packets

### Administrative Scoping

* 239.0.0.0/24 private addresses
* Can set boundaries to limit
 * Requires manual config
* Can apply filter to stop traffic with private range entering and/or
  exiting an interface

# Dense-Mode Routing Protocols

## Operation of PIM-DM

* Was Cisco Proprietary
* Offered as experimental protocol in RFC2362, 3446 and 39873
* RPF check, flooding m'casts until prune, and SM logic of no forwarding
  until joins, all PIM rules
* Unicast IP routing table for RPF check

### Forming Pim Adj - Hellos

* v2 (current version) sends hellos every 30s by default
 * on every pim interface
* v2 Hellos IP Proto 103
* 224.0.0.13 (ALL-PIM-ROUTERS)
* Contains holdtime value (three times hello)
* v1 used PIM Queries instead of hellos
 * IP Proto 2
 * 224.0.0.2
* PIM messages sent only on ints with known active PIM neighbours

### Source-Based Distribution Trees

* When PIM-DM rx's packet, RPF check
* If passed, forwarded to all PIM neighbours except packet source
  neighbour
* Source-based distribution tree (or shortest path tree)
 * Defines path from source host and all subnets requiring it
 * Tree has source as root, router as nodes, subnets as branches, leaves of tree
* **ip multicasting routing** and **ip pim dense-mode** on each int
* Different SBDT for each source and m'cast group, SPT differs on source
  and hosts location
* (S,G) refers to particular SPT, S is source's IP, G m'cast group

### Prune Message

* SPT created on first m'cast packets to new group address
* SPT includes all ints except RPF ints
* Some subnets might not need it, Prune
* Prune from one router to other removes link from (S,G) SPT
* Default 3 minute Prune Timer on prune rx
* After 3 mins, sends traffic again, another prune required to stop

### PIM-DM: Reacting to failed link

* If unicast table changes, RPF int can change
* Different ints in outgoing list
* During process, ints pruned may go forwarding instead, incoming ints
  changed

### Rules for pruning

Routers send prunes for many reasons, main ones: -

* Packets rx'd on non-RPF interface
* No locally connected hosts or downstream routers in group
* IGMP Leave from host, query from router
* If none, prune referencing SPT (eg (10.1.1.10, 226.1.1.1)) out RPF int
* Continues until reaching something with hosts in group
* **show ip mroute** has P (prune flag), router pruned itself from (S,G)
  SPT
* Combination of C flag and RPF neighbour of 0.0.0.0 means connected
  device is source of group
* Single message for Prune and Join
 * Prune - group in prune field
 * Join - group in join field

### Steady-State Operation and State Refresh Message

* v2 introduced state refresh
* Prevents automatic unpruning
* State Refresh sent before Prune time expires
* Defalt timer is 60s, not tied to expiration of Prune
* Keeps S,G tree in steady state

### Graft Message

* Unprunes, rather than waiting
* Message to upstream, causing upstream to place traffic back on link
  for S,G SPT
* Graft Acks sent downstream (R1 to R2, R2 then sends to R3 etc)
* Individual grafts per router needed if many pruned

## LAN-Specific Issues with PIM-DM and PIM-SM

### Prune Override

* Some may want to prune, not others, on same segment
* Sent by router if it sees another prune on segment
* As prune sent to ALL-PIM-ROUTERS (224.0.0.13), other routers see it
* Prune override is just a join, sent before 3-second time expires

### Assert Message

* Routers negotiate, winner responsible for forwarding multicasts onto
  LAN
* Winner based on routing protocol and metic to find route to reach
  unicast address

1. Router with lowest AD to learn route wins
2. If tie, metric wins
3. If tie, Highest IP on LAN wins

### Designated Router

* PIM Hellos elect DR on multiaccess network
* Router with highest IP DR on link
* applies mainly for v1, as no querier mechanism
 * No way to device which routers sends IGMP queries
 * in v1, PIM DR IGMP querier
 * in v2, directly elects querier
* Querier and Assert likely diferent routers
* Querier uses lowet IP, Assert has highest IP as breaker

### Summary of PIM-DM Messages

* Hello - Forms neighbours, mnitors Adj, elects PIM DR on MA networks
* Prune - Asks neighbour to remove link for S,G SPT
* State refresh - Maintains prune state
* Assert - M'cast forwarder on LAN when multiple routers
* Prune override - Stops m'cast traffic being pruned on link, when only
  one router wants prune
* Graft/Graft-Ack - Prune link back up for S,G SPT, ack'd by RPF
  neighbour

## DVMRP

Diffs between PIM-DM and DVMRP

* No full IOS support for DVMRP, supports connectivity to a networ with
  it
* Own DV protocol similar to RIPv2, route updates 60s, 32 hops infinite,
  adds overhead
* Probes to find neighbours in ALL-DVMRP-ROUTERS 224.0.0.4
* Truncated broadcast tree, similar to SPT with some links pruned

## MOSPF

* RFC 1584
* extension to v2 routing protocol
* Group membership LSA (type 6) floods throughout originating routers
  area
* All MOSPF routers in area must have identical LSDBs
* SPT calc'd on demand, when first m'cast packet from group arrives
* All routers know where attached members are
* After SPF calc, entires into m'cast forwrding table
* SPT loop free, RPF check not required
* Only works with OSPF unicst protocol
* Suitablefor small networks, more hosts means higher Dijkstra runs
* IOS doesn't support it

# Sparse-Mode Routing Protocols

## Operation of PIM-SM

* Assumes no hosts want packet until they ask for it
