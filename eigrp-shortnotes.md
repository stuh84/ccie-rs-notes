# Basics and Evolution

* IETF Draft exists (draft-savage-eigrp)
* In Quagga
* IP Protocol 88
* RTP for uni/m'cast delivery
* Hello and hold times
* Full updates on neighbour up
* Partial after
* MD5 or SHA auth
* Mask included in each route (VLSM/classless)
* Supports route tags
* Next hop field
* Summarize anywhere
* Multiprotocol (v4, v6)

## IGRP

* Cisco alternative to RIP v1
* Avoids hop count limits (up to 255)
* Composite metric
* 90s update period
* Unequal cost load sharing
* Interoperated with v4, ISO, CLNP, IPX, AppleTalk
* Request packet out all IGRP interfaces at start
* Sanity check on rx'd updates
* Vefifies source of packet belongs to subnet received on
* Classful DV
* Periodic full info broadcast
* Split-horizon
* Triggered updates
* Invalid After, holddown, flushed after
* Advertises summaries at boundaries

## IGRP to EIGRP

**IGRP Weaknesses**
* Full updates
* No VLSM
* Slow convergence
* Bad loop prevention

**EIGRP Advantages**
* Decoupled updates and adjacency building
* Event driven than timer driven
* Hello protocol
* RTP
* Allows complete routing info exchange, then changes only after (RTP verified)
* Loop free paths found with FC (packets not forwarded to neighbour that coudl loop)

**Diffusing Computation**
* FC could cause neighbour with lease cost path being ineligible
* Neighbours queried for their best path after topology change
* Replies with udpated distances
* Some neighbours respond straight away
* Some propagate query further

**DUAL**
* For multiple changes in one computation
* Controls computation run
* Processes replies, and how/when to insert routing table info

* Hop count limitation of 100 by default, 255 max **metric maximum-hops**
* Internal and external distance, 90, 170, **distance eigrp <internal> <external>**

# Metrics, Pacekts and Adj

## Classic Metrics

* Component metrics
* Bandwidth, delay reliability, load (combined for single number)
* MTU and hop count

### Bandwidth
* **bandwidth** per int command
* If none assigned, IOS assigns one (true speeds unless tunnel or serial)
* Minimum bandwidth along route taken
* Compares bandwidth advertised by neighbour to bandwidth advertised to neighbour
* 1kbps to 10Gbps

### Delay

* **delay** per int
* Realistically average delay
* Represents serialization delay
* Units in tens of microseconds
* Seen in show int
* Implicit delay if none set
* Total delay used
* Range of 10 to 167772150 microseconds. Delay of 16777215 tens means infinite distance

### Reliability

* Ratio of successfully received and all received frames
* Fraction of 255
* Dynamically updated by IOS
* Minimum reliability used
* Updates not sent if reliability changes
* Tends to be ignored, used from IGRP legacy

### Load

* Amount of traffic through interface
* Fraction of 255
* Computed exponentially, so smooths out burst
* Different in Tx and Rx
* Maximal Txload, higher value on link taken
* Does not trigger updates

### MTU

* Minimum MTU carried, not used in best path selection

### Hop count

* Number of hops
* Default limit of 100
* 1 to 255 range

### Calculating composite metric

* Each router computes its own
* Each component advertised (unless redistributed from another EIGRP process)
* Even with redistribution, only for diagnostics as components also carried
* K Values in formula (weight constants, 0 to 255), tweakble

* K1 - Multiplied by bandwidth
* K2 - BWs ((256 * 10^7)/Bw-Min) over 256) - load maxed
* K3 - 256 * delays summed
* K5 over k4 + Reliability min
* K5 0 by default (negating reliablity)

## Wide Metrics

* Bandwidth and delay carried in scaled form (so rounding errors possible)

Wide metrics support shown with: -
* **show eigrp plugins** - eigrp-release 8.0.0+
* **show eigrp tech-support**
* **show ip protocols** - Shows K6, rib scale 128, 64 bit metric

Consist of: -
* Throughput - Similar to bandwidth, 65536x10^7/Int-Bandwidth(kbps), max of 655.36 TBPS link
* Latency - Similar to delay, 65536xInterface Delay/10^6, in pico seconds
 * Interfaces operating on 1Gbps and no bandwidth set, IOS default delay in pico seconds
 * Over 1Gbps, 10^13/interface default bandwidth
 * Bw set and not delay - default delay in pico seconds
 * Explicitly set delay - converted to Picoseconds (10^7 delay)
* Reliability - As classic
* Load - As classic
* MTU - As classic
* Hop Count - As classic
* Extended, placeholders for future
 * Three already defined (Jitter, Energy, Quiescent Energy)
 * K6

* Named mode automatically wide metrics
* Detect if neighbour supports it
* Wide preferred
* If mixed, routers using wide use both metrics in messages, allowing non-wide support
* RIB only supports 32 bit metrics, so downscaled. Level of downscaling 1 to 255, default 128

## Tweak int metrics

Use delay as bandwidth can have other effects, also says amount EIGRP can use on interface.

# Packet FOrmat

* IP PAckets, proto 88
* MTU based length
* 20 byte header, then TLVs
* TLVs of EIGRP and TLV versions, k_values, hold timers, control info for multicasting and route reachability
* No RTP header, flag seq and ack number provide it

* Version field - 4 bit, set to 2 always
* Opcode - 4 bit packet type (1 = update, 3 = query, 4 = reply, 5 = Hello/Ack, 10 = SIA Query, 11 = SIA reply)
* Checksum - 24 bit on entire EIGRP packet
* Flags - 32 bits, 0x1 Init, 0x2 Conditional Received, 0x4 Restart, 0x8 End-Of-Table
* Seq - 32 bit, RTP
* Ack - RTP, contains seq number of last packet from neighbour. Hello with nonzero ack treated as ack instead of hello (only non zero if packet itself is unicast, acks never unicast usually)
* Virtual ROuter ID - 0x1 Unicast AF, 0x2, Multicast AF, 0x8000 UNicast Service AF
* ASN - 16 bit
* TLV - Route entries and EIGRP dual info
 * 0x0001 - EIGRP parameters (General TLV Types)
 * 0x0002 - Auth type (gen)
 * 0x0003 - Seq (gen)
 * 0x0004 - Software Version (gen)
 * 0x0005 - Next multicast seq (gen)
 * 0x0102 - v4 Internal Routes (IP Specific)
 * 0x0103 - v4 external (IP-spec)
 * 0x0402 - v6 internal (ip-spec)
 * 0x0403 - v6 external (ip-spec)
 * 0x0602 - Multi Proto internal (AFI-spec)
 * 0x0603 - Multi external (AFI-spec)

Each internal and external TLV only one route entry. Update, Query, Reply, SIA query and SIA reply contain at least one TLV to advertise a network or query it

List/array (vector) of Internal and External Route TLVs, is DV nature of EIGRP

# Packets

* Hello
* Ack
* Update (reliable)
* Query (reliable)
* Reply (reliable)
* SIA-Query (reliable)
* SIA-Reply (reliable)

**show ip eigrp traffic** packet stats

## Hello packets

* Periodic
* Identifies neighbours
* Verifies if neighbours comptaible (common subnet, ASN, K values, Auth)
* Is keepalive
* 224.0.0.10 or FF02::A
* Static neighbours as unicast
* Hello 5s default, 60 on NBMA with ints less than 1544kbps
* Opcode 5
* Not ack'd

## Acknowledgement

* For reliable packets
* Response to Update, Query, Reply, SIA-Query and SIA-Reply
* Unicasted to sender
* Format - Hello with empty body, common EIGRP headers, non-zero ack field
* Hello opcode
* Can contain Ack to previous reliable packet, not necessary for standalone acks
* Ack number field similar to TCP
* Only unicasted packets carry ack numbers

## Update

* Routing info
* Unicast or multicast
* New adj buildup is unicast, unless multiple neighbours on one int in short time (multicast)
* Post full sync, m'cast
* If no ack from update, unicast to unresponsive neighbour
* Always unicast on p2p and static neighbours
* Reliable delivery
* Opcode 1

## Query

* Seach for best route to dest
* Reliable
* M'cast on multiaccess only when dynamic neighbours
* If not ack'd in proper time, unicast retransmit
* P2p and static unicast
* Must be ack'd, although ack's receipt, not replies
* Opcode 3

## Reply

* Query response
* Senders current distance to dest (after topology change)
* Always unicast to Query originator
* Ack'd
* Opcode 4

## SIA-Query and SIA-Reply

* During prolonged diffusing computation
* Verifies if neighbour is truly reachable and still computating
* SIA-Reply if still computating
* Timers govern maximum computation, extended on SIA-Reply
* Unicast and reliable
* SIA-Q Opcode 10
* SIA-R Opcode 11

## RTP

* Reliable delivery of EIGRP packets
* All reliable packets have non-zero seq
* Seq number - global value, maintained per EIGRP process, incremented whenever packet originated by instance
* Acks sent in response to reliable packets, ack is rx'd seq
* Ack can piggyback onto own reliable packets
* If acks not rx'd in time, retransmits as unicast to unresponsive neighbour

### Conditional Receive

* Partitions neighbours on multiaccess interface
* Two groups, well behaved (ack messages) and lagging (failed to ack at least 1 reliable packet)
* Flag set on multicast packets for routers that ack'd all so far

Above accomplished by: -

* Hello sent with Sequence TLV and Next'Mcast Sequence TLV (sequenced hello)
* Next M'Cast - Upcoming seq number of next reliable message
* Sequence TLV - List of lagging neighbours, tells these to ignore next message
* Neighbour puts itself in CR mode
* Sending router sends next m'cast with CR flag
* Routers in CR mode process as normal
* Routers not ignore it
* Routers not in CR-mode can catch up

* Ack wait time specified by multicast flow timer
* Time between subsequent unicast specified by RTO (Retransmission timeout)
* Flow and TRO calc'd for each neighbour from SRTT
* Smooth Round Trip Time - average elapsed time between reliable packet and ack, in ms

# Router Adjacencies

* Dynamic by default
* Can be static
* 224.0.0.10 or FF02::A
* Static usually on NBMA/none m'cast media (FR)
* Static stops m'cast on interface neighbour is reachable
* Following need to match
 * Auth
 * K-Values
 * ASN
 * Primary addresses for relationships
 * Same subnet
* Timers dont need to match
* Hello 5s default
* Hello 60 on NBMA
* **ip hello-interval eigrp 1**
* Hold time default 3 times hello
* Changing hello doesn't change hold
* If hold expires, neighbour unreachable, dual informed
* **ip hold-time eigrp 1**

**Adjacency Process**

|Direction|Packet|State Following|
|---------|------|---------------|
|R1 to R2 | Hello | R2 puts R1 to pending|
|R2 to R1|Hello|R1 puts R2 to Pending (expedited allowing quick discovery)|
|R2 to R1|Null Update with INit, Seq=x|R1 declares Init rx'd from R2|
|R1 to R2|Null update with with Init, Seq=y, Ack=x|Init and Ack rx'd from R1, R2 puts R1 to up|
|R2 to R1|Ack, Ack=y|Ack rx'd from R2, R1 puts R2 to up|
|R1 <-> R2| Database Sync|Uses updates and Acks|

**Pending State**
* Doesn't send/accept EIGRP messages with routing info until connectivity in place
* Packets exchanged unreliable, other than reliable with Init flag
* Null update - no routing info

**show ip eigrp neighbours**
* H (handle) - Internal number assigned to neighbours, independent of addressing
* Address and Interface
* Hold - From value adv'd by neighbour
* Uptime
* SRTT
* RTO
* Q Cnt - Enqueued reliable packets, must be zero in stable network. Non-zero normal during db sync or convergence
* Seq number - Seq number of last reliable packet from neighbour

# Diffusing Update Algorithm

* Convergence algorithm
* Replaces Bellman-Ford algorithm in other DV
* Avoids loops by performing distributed SPF computation
* Maintains freedom from loops during calculation

# Topology Table

Stores routing info

* Prefix of known dests (address and mask)
* FD of network
* Address of neighbouring router that advertised network (with egress int)
* Metrics of networks adv'd by neigh, plus resulting metric of path to dest
* State of dest network
* Other info (flags, type origin etc)
* Populated with connected networks (local or redist'd)
* Also pop'd with contents of Update, Query, Reply, SIA-Query and SIA-Reply
* Neighbour with least cost path to destination AND loop free path chosen from topology table

Networks active or passive
* Passive - Shortest Path found
* Active - Searching for shortest path

Active for query packets being sent out
* Can't modify routing entry while queries still not replies to (i.e. no removing or changing next hop)
* Route loop free in this state
* If neighbour providing least-cost path can't guarantee loop free path, route in active state

**show ip eigrp topology all-links** - All routes, including FC check failures

# Computed, Reported and FDs, and FCs

**RD**
* Best distance of particular neighbour to destination

**CD**
* Total metric to destination via particular neighbour

(CD/RD) in **show ip eigrp topology**

**FD**
* Record of lowest distance known since last active to passive for route
* Not always equal to best CD
* Can only decrease (if CD goes below FD) or remain at current value (if current CD rises but route is passive)
* Never advertised
* Compared with RD, FD has at some point been at a certain value, meaning RD must be lower (nearer to destination)
* Shouldn't be a case if using a path with lower RD than FD that a routing loop forms

If current distance lower, packets never pass back to this router. Any neighbour closer to destination thatn this route has been since last time destination became Passive, cannot form loop. 

RD < FD = Feasibility Condition

FC not necessary for loop freedom. Means every neighbour meeting FC provides loop free path, but not every loop-free path satisfies FC>

All passing FC are FSs. Least CD to dest are Successors (can be multiple)

# Local and Diffusing Computations in EIGRP

* TC = Distance to network change or new neighbour online for that network
* Detected by receiving Update, Query, SIA-Query or SIA-Reply with info about network
* Also detected by local int metric change
* Neighbour going down processed with CD/RD through neighbour set to inifnity

If new shortest path passes FC, FS, performs following: -

1. FS providing least CD is successor
2. If CD over new Successor < current FD, FD updated to new CD
3. Routing table update
4. If current distance to dest changed, update packet to neighbours

Above is local, info in topology table. Passive state through this.

If no FS after TC, could be a loop. Diffusing computation started: -
1. Route locked, can't be removed or change next hop
2. FD set to current CD through current Successor. If router needs to advertise its distance while in Active state, uses current CD through successor
3. Network placed in Active state, queries sent to neighbours

Queries contain network and routers current CD to it

Each neighbour updates topology table using distance in query, re-evaluates its Successors and FS. After, neighbour till has own FS or Successor with least-cost loop-free path, or needs to change current Successor.

If neighbour still has succesor, reply sent back with current distance. Route doesnt go active on this neighbour. Diff Comp boundary here.

If no successor, diff comp, own queries, own CD through current successor. Query through this part of network.

One destination active, must wait on replies. Once replies received, FC check skipped, and back to passive. FD becomes CD by selected neighbour. If router Active by receiving queries, it replies to queries, with distance it now has. Otherwise only updates

Main info in Update, Query, Reply and both SIAs is senders current distance. Computation started only if this changes things for a neighbour.

During successor failure: -
* When EIGRP detects change, records in topology table, updates RD and CD of neighbour advertising change, or influenced by (link metric)
* Identify least CD through other neighbours (that have updated CDs themselves)
* After least CD through neighbour found, verify neighbour meets FC and is FS. If yes, successor. If not, active.

# DUAL FSM

One passive, four active states

* A0 - Local origin with distance increase
* A1 - Local origin
* A2 - Multiple origins
* A3 - Successor origins

Rules are: -

* Passive if distance change means neighbour providing least CD doesnt meet FC
* If successor query fails to meet FC, enter A3. Queries sent, wait for replies. If no further distance increase while waiting, get last reply, go active. Change FD, choose new S
* If distance change from update, int metric, or neighbour loss, and neighbour distance fails FC, A1. Queries sent. If no distance increase or queries from S, back to passive
* If during A3 or A1, distance increase from something other than successors query, another change occured. As this cant be adv out, other routers may not know. State from A3 to A2, or A1 to A0. When last reply arrives, check least CD passes DC, using FD set when active. If FC passes, passive again. If not, return to previous state, another diff comp
* If during A1 or A0, a query from successor rx'd, another changed occured. State changes to A2, proceeds as above

Query origin flag stores active state

# Stuck in Active

* Single misbehaving router can cause to be SIA, never ending diff comp
 * CPU overload
 * Packet loss
 * Congestion
 * Large topology/complex, lot of prefixes from single node failure

Default active timer is 3 minutes, can be between 1 and 65535, **timers active-time**. If replies not received, router is SIA. Adj torn down to unresponsive neighbours. Diff Comp takes responds to be infinite metric.

SIA-Query and Reply exist to combat above aggressive behaviour
* If neighbour has no response within half of active, SIA-Query sent
* SIA-Reply can say "waiting on replies" or "Done, this is metric"
* Reply send immediately
* Resets active timer
* Three SIA-Qs sent, each after half active. If Comp not finished by end plus one half of active, adj tear down

Two routers can't wait on each others reply. If during active, router gets another query for dest, reply sent back immediately with same distance in own query.

To avoid SIA, limit query propagation depth, and networks dependent on a link (passive int, route filtering, stub etc)

# EIGRP Named Mode

* IOS 15.0(1)M
* Old method is classic/as mode
* Named mode preferred
* Features go into named mode now
* Anything not in config mode ignored if named mode instance used

Three blocks
* AF section - Specifics AF for this EIGRP instance (ASN in here)
* Per-AF interface - Located inside AF, per interface or sub interface. Can set default settings. Optional section
* Per-af-topology - Multi-Topology-Routing, always present even if no support for MTR

```
router eigrp DAVE
 address-family ipv4 unicast autonomous-system 1
  af-interface default
   hello-interval 1
   hold-time 3
  exit-af-interface
  af-interface Loopback0
   passive-interface
  topology base
   maximum-paths 6
   variance 4
  exit-af-topology
  network 10.0.0.1 0.0.0.0
  network 10.255.255.1 0.0.0.0
 exit address-family
 address-family ipv6 unicast autonomous-system 1
  af-interface default
   shutdown
  exit-af-interface
  af-interface Loopback0
   no shutdown
  exit-af-interface
  af-interface Fa0/0
   no shutdown
  exit-af-interface
  topology base
   timers active-time 1
  exit-af-topology
 exit-address-family
```

* Multiple named processes
* Named not in messages
* Each process only a single instance for an AF
* Two or more processes cannot run same AF with same ASN
* Default for v6 is on all interfaces, even link-local only

## Address Family Section

* af-interface
* default - Set command to defaults
* eigrp - AF family commands
* help - interactive help
* maximum-prefix
* metric - metric and parameters for advertisement
* neighbour - static
* network
* shutdown - shutdown af
* timers
* topology

## Per-AF-Interface section

* add-paths - Advertise add paths
* authentication - Configure auth
* bandwidth-percent - Set percentage of bandwidth limit
* bfd
* damepning-change - Percent interface metric must change to update
* dampening-interval - Time in seconds to check int metric
* default
* hello-interval
* hold-time
* next-hop-self
* passive-interface
* shutdown
* split-horizon
* summary-address

## Per-AF-Topology section

* In MTr, defines subset of routers and links for a separate topology
* Entire network is base topology
* Any additional are class-specific, subset of base
* Each carries class of traffic and indepdnent of NLRI (maintains separate routing tables and FIBs)
* Can segregate different kinds of traffic or indepdnent v4 and v6 topologies

Commands are: -
* auto-summary
* default
* default-information - Distribution of default info
* default-metric - Set metric of redistributed routes
* distance - Defines AD
* distribute-list
* eigrp
* maximum-paths
* metric
* offset-list 
* redistribute
* snmp
* summary-metric
* timers
* traffic-share
* variance

**show eigrp address-family ipv4/ipv6** rather than show ip eigrp or show ipv6 eigrp (both work, but new features wont be shown)


