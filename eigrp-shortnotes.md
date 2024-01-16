# Basics and Evolution

* IETF Draft exists (draft-savage-eigrp)
* In Quagga
* IP Protocol 88
* RTP for uni/m'cast delivery
* Hello and hold times
* Full updates on neighbor up
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
* Loop free paths found with FC (packets not forwarded to neighbor that coudl loop)

**Diffusing Computation**
* FC could cause neighbor with lease cost path being ineligible
* neighbors queried for their best path after topology change
* Replies with udpated distances
* Some neighbors respond straight away
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
* Compares bandwidth advertised by neighbor to bandwidth advertised to neighbor
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
* Detect if neighbor supports it
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
* Ack - RTP, contains seq number of last packet from neighbor. Hello with nonzero ack treated as ack instead of hello (only non zero if packet itself is unicast, acks never unicast usually)
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
* Identifies neighbors
* Verifies if neighbors comptaible (common subnet, ASN, K values, Auth)
* Is keepalive
* 224.0.0.10 or FF02::A
* Static neighbors as unicast
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
* New adj buildup is unicast, unless multiple neighbors on one int in short time (multicast)
* Post full sync, m'cast
* If no ack from update, unicast to unresponsive neighbor
* Always unicast on p2p and static neighbors
* Reliable delivery
* Opcode 1

## Query

* Seach for best route to dest
* Reliable
* M'cast on multiaccess only when dynamic neighbors
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
* Verifies if neighbor is truly reachable and still computating
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
* If acks not rx'd in time, retransmits as unicast to unresponsive neighbor

### Conditional Receive

* Partitions neighbors on multiaccess interface
* Two groups, well behaved (ack messages) and lagging (failed to ack at least 1 reliable packet)
* Flag set on multicast packets for routers that ack'd all so far

Above accomplished by: -

* Hello sent with Sequence TLV and Next'Mcast Sequence TLV (sequenced hello)
* Next M'Cast - Upcoming seq number of next reliable message
* Sequence TLV - List of lagging neighbors, tells these to ignore next message
* neighbor puts itself in CR mode
* Sending router sends next m'cast with CR flag
* Routers in CR mode process as normal
* Routers not ignore it
* Routers not in CR-mode can catch up

* Ack wait time specified by multicast flow timer
* Time between subsequent unicast specified by RTO (Retransmission timeout)
* Flow and TRO calc'd for each neighbor from SRTT
* Smooth Round Trip Time - average elapsed time between reliable packet and ack, in ms

# Router Adjacencies

* Dynamic by default
* Can be static
* 224.0.0.10 or FF02::A
* Static usually on NBMA/none m'cast media (FR)
* Static stops m'cast on interface neighbor is reachable
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
* If hold expires, neighbor unreachable, dual informed
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

**show ip eigrp neighbors**
* H (handle) - Internal number assigned to neighbors, independent of addressing
* Address and Interface
* Hold - From value adv'd by neighbor
* Uptime
* SRTT
* RTO
* Q Cnt - Enqueued reliable packets, must be zero in stable network. Non-zero normal during db sync or convergence
* Seq number - Seq number of last reliable packet from neighbor

# Diffusing Update Algorithm

* Convergence algorithm
* Replaces Bellman-Ford algorithm in other DV
* Avoids loops by performing distributed SPF computation
* Maintains freedom from loops during calculation

# Topology Table

Stores routing info

* Prefix of known dests (address and mask)
* FD of network
* Address of neighboring router that advertised network (with egress int)
* Metrics of networks adv'd by neigh, plus resulting metric of path to dest
* State of dest network
* Other info (flags, type origin etc)
* Populated with connected networks (local or redist'd)
* Also pop'd with contents of Update, Query, Reply, SIA-Query and SIA-Reply
* neighbor with least cost path to destination AND loop free path chosen from topology table

Networks active or passive
* Passive - Shortest Path found
* Active - Searching for shortest path

Active for query packets being sent out
* Can't modify routing entry while queries still not replies to (i.e. no removing or changing next hop)
* Route loop free in this state
* If neighbor providing least-cost path can't guarantee loop free path, route in active state

**show ip eigrp topology all-links** - All routes, including FC check failures

# Computed, Reported and FDs, and FCs

**RD**
* Best distance of particular neighbor to destination

**CD**
* Total metric to destination via particular neighbor

(CD/RD) in **show ip eigrp topology**

**FD**
* Record of lowest distance known since last active to passive for route
* Not always equal to best CD
* Can only decrease (if CD goes below FD) or remain at current value (if current CD rises but route is passive)
* Never advertised
* Compared with RD, FD has at some point been at a certain value, meaning RD must be lower (nearer to destination)
* Shouldn't be a case if using a path with lower RD than FD that a routing loop forms

If current distance lower, packets never pass back to this router. Any neighbor closer to destination thatn this route has been since last time destination became Passive, cannot form loop. 

RD < FD = Feasibility Condition

FC not necessary for loop freedom. Means every neighbor meeting FC provides loop free path, but not every loop-free path satisfies FC>

All passing FC are FSs. Least CD to dest are Successors (can be multiple)

# Local and Diffusing Computations in EIGRP

* TC = Distance to network change or new neighbor online for that network
* Detected by receiving Update, Query, SIA-Query or SIA-Reply with info about network
* Also detected by local int metric change
* neighbor going down processed with CD/RD through neighbor set to inifnity

If new shortest path passes FC, FS, performs following: -

1. FS providing least CD is successor
2. If CD over new Successor < current FD, FD updated to new CD
3. Routing table update
4. If current distance to dest changed, update packet to neighbors

Above is local, info in topology table. Passive state through this.

If no FS after TC, could be a loop. Diffusing computation started: -
1. Route locked, can't be removed or change next hop
2. FD set to current CD through current Successor. If router needs to advertise its distance while in Active state, uses current CD through successor
3. Network placed in Active state, queries sent to neighbors

Queries contain network and routers current CD to it

Each neighbor updates topology table using distance in query, re-evaluates its Successors and FS. After, neighbor till has own FS or Successor with least-cost loop-free path, or needs to change current Successor.

If neighbor still has succesor, reply sent back with current distance. Route doesnt go active on this neighbor. Diff Comp boundary here.

If no successor, diff comp, own queries, own CD through current successor. Query through this part of network.

One destination active, must wait on replies. Once replies received, FC check skipped, and back to passive. FD becomes CD by selected neighbor. If router Active by receiving queries, it replies to queries, with distance it now has. Otherwise only updates

Main info in Update, Query, Reply and both SIAs is senders current distance. Computation started only if this changes things for a neighbor.

During successor failure: -
* When EIGRP detects change, records in topology table, updates RD and CD of neighbor advertising change, or influenced by (link metric)
* Identify least CD through other neighbors (that have updated CDs themselves)
* After least CD through neighbor found, verify neighbor meets FC and is FS. If yes, successor. If not, active.

# DUAL FSM

One passive, four active states

* A0 - Local origin with distance increase
* A1 - Local origin
* A2 - Multiple origins
* A3 - Successor origins

Rules are: -

* Passive if distance change means neighbor providing least CD doesnt meet FC
* If successor query fails to meet FC, enter A3. Queries sent, wait for replies. If no further distance increase while waiting, get last reply, go active. Change FD, choose new S
* If distance change from update, int metric, or neighbor loss, and neighbor distance fails FC, A1. Queries sent. If no distance increase or queries from S, back to passive
* If during A3 or A1, distance increase from something other than successors query, another change occured. As this cant be adv out, other routers may not know. State from A3 to A2, or A1 to A0. When last reply arrives, check least CD passes DC, using FD set when active. If FC passes, passive again. If not, return to previous state, another diff comp
* If during A1 or A0, a query from successor rx'd, another changed occured. State changes to A2, proceeds as above

Query origin flag stores active state

# Stuck in Active

* Single misbehaving router can cause to be SIA, never ending diff comp
 * CPU overload
 * Packet loss
 * Congestion
 * Large topology/complex, lot of prefixes from single node failure

Default active timer is 3 minutes, can be between 1 and 65535, **timers active-time**. If replies not received, router is SIA. Adj torn down to unresponsive neighbors. Diff Comp takes responds to be infinite metric.

SIA-Query and Reply exist to combat above aggressive behaviour
* If neighbor has no response within half of active, SIA-Query sent
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
* neighbor - static
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

# Additional and Advanced EIGRP features

## Router ID

* 4-byte
* Represents instance
* Each AF own router-id
* Multiple processes and AF families can use same RID
* Was used to prevent loops in redistribution (identified originating router)
* Now also includes internal routes
* **eigrp router-id**
 * If not set, highest IP of non-shut loopback
 * Highest ip on non-shut ints
* Not changed until EIGRP processed removed, RID configured or config'd RID removed
* Not allowed: 0.0.0.0 and 255.255.255.255
* RID change drops neighbors
* Only message if two routers with same RID, "Ignore Route, dup routerid" in **showe eigrp address-family ipv4 events**
* Seen in **show eigrp protocols** and **show ip protocols**

## Unequal cost load balancing

* Must be FS for present for loop free paths
* Paths through FS can be installed along with best
* **variance** command, says how many times worse than best path FS can be
* CD via successor < CD via Feasible Successor < V x CD successor
* If it does, will be installed. V of 1 means none
* Traffic amount is Highest Installed Metric / Path Metric
* Unequal cost paths installed into routing table count towards max parallel paths to destination, **maximum-paths**

## Add-Path

* Allows hub to advertise multiple equal-cost routes to destination
* Might require max paths command
* Split horizon must be disabled on tunnel
* Variance must be set to 1
* No **next-hop-self**, original NH needed
* **no-ecmp-mode** available on above
 * Above deactivates internal optimization
 * Walks over all equal-cost paths to dest in topology table
 * Any of these routes successors should be reachable over interface on which route is going to be readv
 * **next-hop-self** only applies to successor, not additional entires

## Stub Routing

* Configured on spokes
* TLV in hello of Spoke
* Does not propagate EIGRP routes to its neighbors, unless a **leak-map**
* Router can never be FS for remote networks (i.e. transit router)
* Advertises subset of own networks
 * set with **eigrp stub**
 * summary, connected, static, redistributed, receive-only
* neighbors that see stub never send query to stub

Stub handles queries as such: -
* Originating query packets - Same
* Processing queries - If any of stub networks above or leak-map, not modded
 * If known by stub but not allowed to advertise, processes with infinite distance

* Might receive queries from old IOS versions (no stub TLV support)
* Also if multiple stubs on common segment
* If stub router in common seg needs to send query, also send to other stubs
* Above supports multihomed branches
* Means can use other routers uplink if own uplink fails

In mixed scenarios
* Non-stub sent as unicast to nonstub, or conditional receive
* Above depends on number of nonstubs

* Protect against suboptimal routing (low speed as transit)
* Limits query propagation

Config: -
* eigrp stub connected - Connected routers adv
* eigrp stub leak-map - Dynamic prefixes in leak
* eigrp stub receive-only - Receive-only neighbor
* eigrp stub redistributed - allow redist
* eigrp stub static - static routes
* eigrp stub summary - Allow summries

For receive only, must have static routes, or NAT/PAT to reach networks behind it

**leak-map**
* Route map with ACLs or prefix lists
* Useful for multihomed

**Connected and static**
* Still need redistribute/network commands to do this
* Summary allows summary on interfaces

* Connected and summary assumed

**show ip protocols** shows if router a stub
**show ip eigrp neighbors detail** - show stub neighbors

Changing stub features drops adj

* Stub router has no impact on what hub advertises to spokes, still sends full

## Route SUmmarization

* Query boundary
* Queries inside summarized network propagate as normal
* If particular component route not in neighbors table, immediate reply with infinite distance
* Auto and manual
* Auto based on classful routing (major network as subnet on interface*
* Auto does nto apply to externals unless internal belongs to same major network as external
* In 15.0(1)M, aut-summ off by default

**Manual**
* Supernetting
* Can be a default route
* Overlapping summaries allowed
 * All advertised if components exist

**Classic Mode**
**ip summary-address eigrp asn address netmask [distance] [leak-map name]**

**Named Mode**
Under af-interface
**summary-address address netmask [ leak-map name]**

* Auto adds null0 into table
* AD of 5 for summary
* Above can cause issues, change distance in classic as shown
* In named, under topology base, **summary-metric address netmask distance admin-distance**
* CLassic way removed in recent releases

AD of 255 stops route going into table, still advertised (Older IOS). Newew don't advertise summary either.

* By default, lowest metric component is summary metric
* Changes if component routes change/new component is lower etc
* Use above command to define it

## Passive Interfaces

* Stops sending EIGRP packets on int
* In classic, **passive-interface** or **passive-interface default**
* In named, **passive-interface** under af-interface, or under af-interface default section

## Graceful Shutdown

* Can advertise being deactivated on an interface, AF or entire process
* Uses "goodbye" message (hello with K's of 255)
* classic only allows graceful shutdown of process in v6
* For v4, sent when shutting down ints, passive, or network statements
* In named, can be in router eigrp mode, under AF, or af-interface

## Authentication

* MD5 since start
* SHA-2 since 15.1(2)S and 15.2(1)T
* MD5 in classic or named
* SHA in named only
* One key chain minimum, key strings, and optionally validity timers
* Key-id and string must match
* SHA can configure password in interface config without key chains
* Above means transition of keys difficult

**Classic Commands**

**ip authentication mode eigrp**
**ip authentication key-chain eigrp**

**Named**

*Af-interface*
**authentication mode**
**authentication key-chain**
Can be under default section

```

key chain EIGRPKeys
 key 1
  key-string DAVE

router eigrp CCIE
 address-family ipv4 autonomous-system 1
  af-interface default
   authentication mode md5
   authentication key-chain EIGRPKeys
  af-interface Fa0/0
   authentication-mode hmac-sha-256 DAVESHA # DAVESHA is not the key used, it is a password set
   authentication key-chain EIGRPKeys
  af-interface F0/1
   authentication mode hmac-sha-256 DAVEISPW
   no authentication key-chain # Above password is now used as the key
  af-interface Se1/0
   no authentication mode
```

**show eigrp address-family ipv4 int detail** - Show used keychain

**send-lifetime** and **accept-lifetime** exists

Lowest key-id used for sending, received depends on ID. Can implement new keys like so: -

1. Add new key with higher ID to all routers
2. Change old-keys send-lifetime to past
3. Remove old key

## Default routing in EIGRP

* No dedicated command
* Redistribution or summarization
* Used to support **ip default9network** (flagging candidate route), network had to be classful and advertised in EIGRP
* If static route configured with only egress interface, IOS treats route as directly connected
 * Means 0.0.0.0 network command would be pulled in 
 * No effect on anything with next-hop set
 * above means all ints in eigrp

## Split Horizon

* Split horizon with poisoned reverse (advertises learned network out towards successor with inifnite metric)
* Can be activated for multipoint ints. Preferred to send default route to all spokes, but if not feasible, disable as such
 * **no { ip | ipv6 } split-horizon eigrp** - Classic
 * no split-horizon - Af interface

# EIGRP over ToP

* OTP for overlay multipooint VPNS between CEs running EIGRP
* Done using LISP

**LISP**
* Decouples host location from identity
* Mantain identity at all costs
* Mapping service maps identity to location
* TUnnel encaps packets between hosts
* End host identities in new packet destined to address representing end host locations
* LOcation of host could be different from locaiton (eg IPv6 and IPv4 reachable)

* LISP EID (Endpoint ID) never changes, v4 or v6 address
* Outside address of router in front of EIDs is RLOC (Routing Locator)
* Routers perform ingress/egress tunnelling of traffic between sites
* Also makes EID-to-RLOC registration and resolution

* LISP has control and data plane
* Control plane: Registraiton protocol and procedures; allows routers to register EIDs responsible for, along with RLOCs
* Data plane: tunnel encap

* For OTP, EIGRP replaces LISP control plane
* EIGRP routers running OTP target sessions, with IPs provided by SP as RLOCs
* Routes are EIDs
* LISP data plane reused in OTP
* Similar to DMVPN, SP never sees inside tunnel, but differences
 * No MP GRE in DMVPN
 * OTP uses LISP UDP encap for data plane, with native EIGRP
 * No tunnel int connfig required
 * Only mandatory static config, specify remote static neighbor
 * OTP can be protected yb GETVPN
 * NHRP in DMVPN in mappings, OTP uses EIGRP itself

```
R1

int lisp0
 bandwidth 1000000

int Gi0/0
 ip address 192.0.2.31 255.255.255.0

ip route 0.0.0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
 topology base
 exit-af-topology
 neighbor 198.51.100.62 Gi0/0 remote 100 lisp-encap
  network 10.0.1.0 0.0.0.255
  network 192.0.2.31 0.0.0.0

R2

int lisp0
 bandwidth 1000000

int Gi0/0
 ip address 198.51.100.52 255.255.255.0

ip route 0.0.0.0 0.0.0.0 198.51.100.1

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
 topology base
 exit-af-topology
 neighbor 192.0.2.31 Gi0/0 remote 100 lisp-encap
  network 10.0.2.0 0.0.0.255
  network 198.51.100.52 0.0.0.0
```

Seen as lisp0 for outgoing interface routes

**show ipv4 addres ipv4 nei** - neighbor on other side

**show ip cef X.X.X.X/X internal** - Shows lisp encap effect 

OTP neighbors can be built into a route reflector, config as such

```
int Lisp0
 bandwidth 1000000

int Gi0/0
 ip address 192.0.2.31 255.255.255.0

ip route 0.0.0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
  af-interface Gi0/0
   no next-hop-self
   no split-horizon
  exit-af-interface
  topology base
  exit-af-topology
  remote-neighbors Source Gi0/0 unicast-listen lisp-encap
  network 10.0.1.1 0.0.0.0
  network 192.0.2.31 0.0.0.0
```

Remote neighbors allows ACL to limit permitted RR clients

# EIGRP Logging and reporting

* Configed in router config mode
* If named mode used, commands in af family section
* **eigrp event-log size**
* **eigrp event-logging**
* **eigrp log-neighbor-changes**
* **eigrp log-neighbor-warnings**
* Event logging on by default, default size 500 lines
* ** show eigrp address-family {ipv4 | ipv6 events} **
* Size between 0-443604
* neighbor warnings on by default, default 10 second intervals

# Route Filtering

* Can be on inbound and outbound updates at any interface or af instance
* Use distribute list command (topology base for named, eigrp process for classic)
* Use ACLs, prefix lists and route-maps with distribute0list

* Does not directly limit query propagation
* Dist list out - All outoing updates, queries, replies and SIA messages, correct metric unless denied (in which case infinite)
* Dist list in - Permitted as normal, denied ignored for Updates, Replies and SIA replies. Queries and SIA-Queries not influenced

# Offset lists

* Adds metrics
* Adds to delay metric
* Any route not matched unchanged
* In or out, and match interface

# Clearing routing rtable

* Clear routes, but will keep in eigrp topology table
* No messages sent after command

* **clear eigrp address-family { ipv4 | ipv6 } neighbors** clears all neighborships
* Using soft on above does graceful restart, resyncs topology tables only
