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
* PIM Joins for routers to request multicast traffic
* Must continually send Joins, otherwise go into prune state

## Similarities between PIM DM and SM

* Same RPF check mechanism
* PIM ND through Hellos
* Recalc of RPF int when routing table changes
* Election of DR on MA network
* Prune overrides
* Asserts elect designated forwarder

### Sources Sending Packets to RP

Steps for initial forwarding of m'cast with SM are: -

1. Source ends packets to RP
2. RP Sends m'cast packets to all routers/hosts registered to group.
This is a shared tree

* Routers with local hosts that IGMP Join for group can oin
  source-specific tree for S,G SPT
* Routers on same subnet as source register with RP
* RP accepts registration only if RP knows routers or hosts that need to
rx multicasts

```
ip multicast-routing
ip pim sparse-mode
ip pim rp-address X.X.X.X
```

Source Registratio process when RP has no requests for group: -

1. Host sends m'casts to group address, router receives m'cast as it
connects to same LAN
2. Router sends PIM register to RP
3. RP sends unicast Register-Stop message back, nothing wants traffic

* PIM Register encaps first m'cast packet
* Would be forwarded if anything in group
* Source host might keep sending m'casts
* When Register-Stop received, 1m Register-Suppression timer
* 5 seconds before timer expires, Router sends another Register, with
Null-Register bit set, without any encap'd m'cast packet

One of two things hapen
1. Another register-stop, resets register suppression
2. Doesn't reply, timer expires, R1 sends encap'd m'cast packets in PIM
register messages (i.e. host/router requires this traffic)

### Joining Shared Tree

* Root-Path Tree - alternate name
* Tree with RP as root
* Defines links m'cast forwards to reach routers
* One tree for each m'cast active group
* After m'cast packets sent by source to RP, RP forwards to group with
  RPT
* RPT created with PIM-SM router's PIM Joins to RP
* Sent under two conditions
 * PIM Join on any interface other than route to RP
 * IGMP Membership report from host on DC subnet
* Notation of (\*,G), any source to group

### Completion of Source Registration Process

* If register for an active group received, no Register-Stop
* De-encaps m'cast packet and forwards

Process goes through: -
* Host sends m'cast to group
* Router encaps m'cast inside Register to RP
* RP de-encaps and sends towards receiving hosts
* RP joins SPT for source of host and group, PIM-SM Join for group S,G
  to source
* When source router receives Join, forwards group traffic to RP
 * Still sending Register mesages with encap'd m'cast packets
* RP sends unicast Register-Stop to source router, stops above

### Shared Distribution Tree

* Traffic from RP to routers/hosts called shared distribution tree/root
  path tree
* If network has multiple sources, traffic to RP, then RPT to receives
* S flag in **show ip mroute** indicates PIM-SM

### Steady-State Operation by continuing to Send Joins

* Periodic, otherwise interface back to pruned
* Routers forward if Downstrem routers still send joins, or DC hosts
  respond to IGMP Querys with IGMP reports for group
* PIM-SM joins every 60s to upstream
* Prune timer 3m default, resets on join
* Must receive at lest one IGMP Report/Join in response to General
  Query, otherwise stops group traffic on int

### Analysing Mroute table

* If incoming int null, indicates router is root of tree (i.e. an RP)
* RPF neighbour listed as 0.0.0.0 for same reason
* T is entry for an SPT, source listed at beginning of same line
 * Incoming int shown
 * RPF neighbour shown
* RP uses SPT to pull traffic from source to itself, shared trees down
  to PIM-SM routers

### Shortest-Path Tree Switchover

* Any router can buil SPT between router and source DR, avoids
  inefficient path
* After router starts receiving group traffic over SPT, Prune to
  upstream of shared tree
* RFC 2362 says initiate switch to SP-tree after significant number of
  packets from a source. No defined amount
* Cisco switch from SPT to source-specific SPT after first packet from
  shared tree
* Change above with **ip pim spt-threshold** *rate*
 * Can be on any router in group
 * Rate is kbps, once over, switches
* RPT joined first as router doesnt know source
* After one packet, learns IP for source and switch to S,G

Process is: -

1. Source sends m'cast packet to first hop router
2. First hop forwards to RP
3. RP forwards to another router in shared tree, other router may have
better unicast path than its RPF int to RP
4. PIM-SM Join out preferred interface to first hop router, for SPT it
is for, travels hop-by-hop to source DR
5. First hop router places another int in forwarding for SPT

* J flag (Join) says traffic switched from RPT to SPT
* S,G entry forwarding to group

### Pruning from Shared Tree

* After above, RPT may no longer be required
* Stop RP from forwarding traffic with PIM-SM Prune to RP
* Prune references S,G SPT, identifying source
* This means "stop forwarding from lited source to listed group down
  RPT"

## Dynamically finding RPs and using Redundant RPs

* Unicast RP, statically config'd **ip pim rp-address** *address*
* Cisco-prop Auto-RP, designates RP, advertises ip to all PIM-SM routers
* Standard BSR, designates RP, advertises

Redunant RPs possible with: -
* Anycast RP with Multicast Source Discovery Protocol (MSDP)
* BSR

### Dynamically Finding RP using Auto-RP

* Sends RP-Announce to 224.0.1.39, stating is an RP
* Message allows router to advertise groups its RP for, allowing some
  load balancing
* Sent every minute
* Next, needs a router to be a mapping agent. Often same as RP, doesn't
  have to be.
* Learns RPs and groups they support
* Sends message called RP-Discovery, identifies RP for each range of
  groups
* Message to 224.0.1.40
* General router population now now which routers are RPs
* RP-Discovery so that Auto-RP mapping agent decides which RP for each
  group. Useful for RP redundancy, supports multiple RPs for a group
* Mapping agent selects router with highest IP as RP for group
* Can have multiple mapping agents
* If router with PIM-SM and Auto-RP config'd, automatically join
  224.0.1.40 CISCO-RP-DISCOVERY group
* Learns Group-to-RP mappings, maintains in cace
* When PIM-SM router gets IGMP or PIM-SM join, checks mapping in cache

Summarized steps: -

1. RP config'd with Auto-RP, announces itself and supported groups to
224.0.1.39
2. Auto-RP mapping agent gathers info about all RPs (RP Annoucnce
Messages)
3. Mapping Agent builds table of best RP for groups
4. RP-Discover from MA to 224.0.1.40 with mappings
5. All routers listen for packets to 224.0.1.40

* Problem is that PIM-SM routers need to send a join to RP they don't
  know yet
* Sparse-Dense Mode helps, makes a router dense if no RP known, SM when
  it does
* Dense long enough to learn mappings, then to sparse
* Configure per interface with **ip pim sparse-dense-mode**
* Can avoid unnecessary dense mode flooding with Auto-RP listener
* This means only Auto-RP traffic flooded out all SM interfaces
* **ip pim autorp listener**

Normal router: -
```
ip multicast-routing

int Se0
 ip pim sparse-mode

ip pim autorp listener
```

Auto-RP Mapping Agent
```
ip multicast-routing

ip pim send-rp-discovery scope 10 # Can designate source int

int Se0
 ip pim sparse-mode
```

Auto-RP RP
```
ip multicast routing

int lo0
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim send-rp-announce loopback0 scope 10
```

### Dynamically finding RP using BSR

* PIMv2 provides BSR
* Similar to AutoRP
* RP sends message to another router collecting group-to-RP mapping
* That router distributes mappings
* Once router is BSR (similar to mapping agent)

Differences from BSR to Mapping Agent: -
* Does not pick best RP for each group
* All mappings sent to PIM routers in bootstrap messages
* Routers pick current best RP by using same hash algorithm on info in
  bootstrap message
* BSR floods mapping info to ALL-PIM-ROUTERS (224.0.0.13)
* Flooding not required to have a known RP

* PIM-SM floods bootstraps out all non-RPF ints, meaning one copy of
  message to every router
* If BS message on non-RPF int, drop pacekt to prevent loops
* Each candidate RP (c-RP) informs BSR it is an RP and groups it
  supports
* All PIM routers know unicast IP of BSR due to earlier BS messages
* C-RPs unicast messages to BSR, with IP used by c-RP and groups
* BSR suports redundant RPs and BSRs
* BS messages contain all c-RPs
* For multiple BSRs, c-BSRs send BS with priority and its IP
 * Highest priority wins, then highest IP
* Winning BSR sends BSR messages, other BSRs monitor
 * If cease, others take over

* Minimum config is a cRP or cBSR, and source of messages
* ACL can limit what groups router will be RP for
* Can specify priority for multiple BSRs

BSR
```
ip multicast-routing

int lo0
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim bsr-candidate lo0 0 # 0 is priority, default
```

On RP
```
ip multicast- routing

int lo2
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

ip pim rp-candidate lo2
```

### Anycast RP with MSDP

* Anycast RP can use RP config, Auto-RP and BSR
* Without anycast RP - One router to be active for each group, load
  sharing for some groups, not others
* With Anycast RP - Multiple RPs acting as RP for same group

* Each RP uses same IP, /32 prefix with IGP
* All methods view multiple RPs as single RP
* Packets routed per IGP to closest RP
* If RP fails, just needs IGP convergence to change

# Interdomain Multicast Routing with MSDP

* Avoids issue when m'cast source might be in one side of netwokr, but
  not other
 * When anycast present and therefore one side of network doesnt see it
* RP uses MSDP to send messages to peer RPs
* Source Active messages list IP of each source for each m'cast group
* Unicast over TCP connection, maintained between RPs
* Static config
* RPs must have routes to each of their peers and to sources (BGP or
  M'cast BGP used for routing)
* RP in one domain could use MSDP to tell RP in another about multicast
  source for specific group at unicast IP (eg 226.1.1.1 known by
172.16.5.5)
* RP in another domain then floods into to any other MSDP peers
* Receiver in its domain joins SPT of source 172.16.5.5, group 226.1.1.1
* If RP no receivers for group, caches them for later

* MSDP RPs send SAs every 60s
* lists roups and sources
* RP can request new list with SA request
* SA response sent back
* Configure Auto-RP or BSR first
* If MSDP between routing domaings, then needs BGP
* MSDP peers specified on each router

```
int lo2
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

ip multicast-routing
ip pim rp-candidate lo2
ip pim msdp peer 172.16.1.1
```

```
int lo0
 ip address 172.16.1.1 255.255.255.255
 ip pim sparse-mode

ip multicast-routing
ip pim rp-candidate Lo0
ip msdp peer 10.1.10.3 connect-source Lo0
```

* Verify with **show ip msdp peer** (sender) and **show ip pim rp**
  (receiver)

# Bidirectional PIM

* PIM-SM inefficient with large number of sends and receivers

Steps for Bidi are

1. RP builds shared tree with root (same as SM)
2. When source sends m'casts, router receiving does not use PIM register. Instead, forwards packets in opposite direction of shared tree to RP
3. RP forwards through shared tree
4. All packets forwarded per step 2 and 3, RP does not join source tree for source, leaf do no join SPT either

# Comparison of DM and SM

* While SM more complex, more popular
* PIM-SM quickly moves to SPT when senders and receivers increase, same
  SPT PIM-DM would have derived

# Source-Specific Multicast

* Scenarios p to know using ISM (Internet Standard Multicast)
* No worrying about source
* Can lead to overlapping m'cast IPs (some streams using same addresses
  as address space not large)
* DoS attacks - Attack can be source, can interrupt stream or tax
  routers/switches
* Complexity - Complexity increases in large networks

* SSM receivers known unicast IP of source, specify it in group
* SSM receivers subscribe to S,G with both source and group address
* Hosts then only receive from specific sources
* Hard to DoS if not got source IP, and path needs to go through RPF
  checks
* RPs dont need to track which sources are active, as sources known
* Only edge routers nearest host need SSM

* Uses IGMPv3
* **ip pim ssm { default | range** *access-list* **}**. Addresses in
  232.0.0.0/24
* Default permits to forward all multicasts in that range
* Can limit groups with ACL and range keyword
* Need IGMPv3 under each interface

```
ip multicast-routing

int Fa0/0
 ip pim sparse-mode
 ip igmp version 3

ip pim ssm default
```

# Implementing v6 Multicast PIM

* **ipv6 multicast-routing**
 * Enables on all interfaces
 * Assumes v6 PIM, doesnt appear in config
 * Always sparse mode
* **no ipv6 pim** interface command
* Tunnels formed for multicast routing
 * Dynamically when above enabled
* Tunnel protocol in **show int tunnel** - PIM/IPv6
* **show ipv6 pim neighbors**
 * Formed using link locals
 * DRs still elected
 * Values maniped as per v4


