# Protocol

# Messages

## Packet format
* IP Proto 88
* MTU based length
* 20 byte header then TLVs
* TLVs of EIGRP and versions, k values, hold timers, control info for multicasting, route reachability
* Version field - 4 bits, always 2
* RTP done with flag seq and ack number
* Opcode for packet type (update, query, reply, hell/ack, sia query, sia reply, 1 3 4 5 10 11)
* Checksum - on entire packet
* Flags - Init, conditional rx, restart, end of table
* Seq, 32 bit, RTP
* Ack, RTP, hello with nonzero ack is ack (only nonzero if unicast)
* Virtual Router ID - Can be Unicast AF, multicast AF unicast service AF
* ASN 16 bits
* Multiple TLVs with Parameters, auth types, seq, version, next multicast seq, v4 internal routes, v4 external, then same for v6 and multiproto
 * Each int and ext TLV one route entry
 * Update, Query, Reply SIA query and SIA reply contain at least one TLV to advertise/query network

## Packets

* Hello - IDs neighbours, neighbour comptaability (common subnet, ASN, k values, auth), keepalive
 * 224.0.0.10 or FF02::A
 * Static neighbours unicast
 * Opcode 5
* Ack
 * Unicasted to sender
 * Hello with empty body, non-zero ack field
 * Can contain Ack to previous reliable packet, not necessary for standalone acks
 * Only unicasted packets have acks
* Update (reliable)
 * Routing info
 * Uni or mcast
 * New adj buildup is unicast, unless multiple neighbours on one int in short time
 * M'cast after first full sync
 * If no ack, unicast to unresponsive neighbour
 * Always unicast on p2p and static
 * Opcode 1
* Query (reliable)
 * Search for best route
 * Mcast on m'access only when dynamic neigh
 * Unicast retransmit
 * P2P and static unicast
 * Opcode 3
* Reply (reliable)
 * Query response
 * Senders current dist to dest (after TC)
 * Unicast to query originator
 * Opcode 4
* SIA-Query (reliable)
* SIA-Reply (reliable)
 * Both above during prolonged diff comp
 * SIA-reply if still computating
 * Timers govern max computation, extended on SIA reply
 * Q opcode 10
 * R opcode 11

* Seq - global value per EIGRP process - incremented on any packet originated by instance
* ACks can piggyback onto own rleiable packets
* Retransmits if acks not in time



# Timers

* Hello - 5s default, 60 on NBMA with less than 1544kbps
 * ip hello-interval eigrp 1
 * doesnt change hello
* Hold - 3x hello
* Active timer - default 3 minutes, can be 1 to 65535, timers active-time

# Trivia

* IP Proto 88
* RTP
* Full updates on neighbour up
* partial after
* MD5 or SHA Auth
* Mask in each route
* Supports route tags
* Next hop field
* Summarize anywhere
* Multiprotocol
* Hop count 100 by default, 255 max, metric maximum-hops
* 90, 170 AD, distance eigrp

## Metrics

* Classic Metrics
 * Bandwidth - per int, IOS assign if none, min along route taken, compares rx'd bw to tx'd bw, min taken, 1k to 10gig
 * Delay - per int, serialization delay, total delay used, tens of ms, range of 10 to 167772150 ms (top being infinite)
 * Reliablility - fraction of 255 - Tends to be ignored - updates not sent if changed, min used
 * LOad - Fraction of 255, different Tx and RX, max used, does not trigger updates
 * Min MTU carried, not used in selection
* K Values
 * K1 - Multiplied by bw
 * K2 - Load maxed
 * K3 - 256 * delays summed
 * K5 over k4 - Reliability min
* Usually only taken bw and delay
* Wide metrics
 * eigrp-release 8.0.0+
 * Scaled form carried (no rounding errors possible
 * K6 in show ip protocols, as well as rib scale 128, 65 bit metric
 * Throughput - like bw, 655.36 tbps max
 * Latency - Similar to delay, in ps
 * Reliability, Load, MTU, hop count as classic
 * Extended for future (Jitter, Energy, Quiescent Energy)
 * Named mode auto wide
 * Detected if neighbour supports
 * Wide preferred
 * both sent if mixed
 * RIB supports 32 bit metrics only, 128 default downscaling, can be 1 to 255
* Use delay to tweak


## Diffusing Computation

* Neighbours queried for best path after TC
* Replies with updated distance
* Some neighbours respond straight away
* Some propagate furhter

## Dual

* Multiple changes in one computation
* Processes replies

## Conditional Receive

* Two groups - well behaved and lagging (at least 1 failed ack)
* Next M'cast - upcoming seq
* Sequence TLV - list of lagging neighbours, tells to ignore next message
* Neighbour in CR mode
* Next m'cast with CR flag from router
* Routers in CR process as normal
* routers not ignore
* Rotuers not in CR catch up
* Ack wait specified by multicast flow imter
* Time between subsequenct unicast by RTO (retransmission timeout)
* Flow and TRO calc'd from SRTT
* SRTT - Average time between reliable packet and ack in ms

## Router Adj

* static usually on NBMA/none mcast
* Static stops m'cast on interface neighbour available by
* Need to have same auth, k-values, asn, primary addresses for relationships, same subnet
* On Rx of Hello, other side in pending
* Null updates after both in pending, init rx'd, then acked
* Once both inits sent and ack'd on both sides, both up and db sync'd
* Pending - no EIGRP messages with routing info allowed yet
 * Unreliable exchange

## DUAL

* Avoids loops with distribued SPF calc

## Topolgy table

* prefixes, FD, add of advertising router (with egress int), metrics adv (RD) and resulting path (FD), state, flags or origin type
* Populated with connected networks, update query, reply and SIAs
* Neighbour with least cost path and loop free chosen
* Can't modify routing entry while queries ongoing
 * Route loop free in this state
 * Active if no guaranteed loop free path

## Comptued, Reported, FD and FCs

* RD - best distance of neighbour to distance
* CD - Total metric to dest via neighbour
* (CD/RD) in topology table
* FD - record of lowest distance known since last active to passive for route
 * Not always equal to best CD
 * Can only decrease or remain at same (if current CD rises but route is passive)
 * never adv
 * RD must be loer than FD
* FC = RD lt FD
* FC not necessary for loop freedom - not every loop-free path satisfies FC
* All passing FCs are FSs, least CD to dest are Successors

## Local and Diff Comp

* TC - Distance to network change, or new neighbour online
* Detected - received U, Q, SIA Q or SIA R about network, or local metric change
* Neighbour going down - neighbour set to infinity
* If new shortest path passes FC, FS, does following: -
 * FS with least CD is successor
 * If CD over new success lt current FD, FD updated to new CD
 * Table update
 * if current distance to dest changed, update to neighbours
* If no FS after TC, could be loop
 * route locked - can't remove or change next hop
 * FD set to current CD through current successor
  * If needs to advertise, current CD used
 * Network n active
* Queries have network and routers current CD to it
* Distance in query updates topology table, revals Success and FS
 * If neighbour still has successor, reply back with CD - diff comp boundary
 * if no successor - diff comp, own queries, own CD through successor
* once rplies received, FC check skipped, FD becomes CD by selected neighbour
* If router active by receiving queries, replies, with current distance. Otherwise updates

## DUAL FSM

* A0 - Local origin with distance increase
* A1 - Local origin
* A2 - Multi origin
* A3 - Succesor origin

* If successory query fails to meet FC, A3. 
 * If no dist increase while waiting, go active. Change FD, choose new S
* If dist changed from update, metric or neighbour loss, and neighbour distance fails FC, A1. Qs sent. If no distance increase or queries from S, back to passive
* If during A3 or A1, distance increases from something other than successor, another change. State goes from A3 to A2 or A1 to A0. When last reply arrives, check least CD passes DC, using FD set when active. If FC passes, passive again. If not, return to previous state
* If during A1 or A0, query from successor rx'd, anotehr change occured. Change to A2

## SIA

* Adj torn down to unresponsive neighbours (after active time)
* SIA sents SIA-query in half active time
* SIA-Reply can say waiting on replies, or with metric
* Reply sent immediately
* Resets active timer
* Three SIA-Qs sent (half active time)
* If comp no finished by end plus one half active, adj tear down

## Aditional and advanced features

**RID**
* 4 byte, represents instance, each AF won router ID
* Can use same in different AFs and processes
* did prevent loops
* Now includes internal routes
* Set with eigrp router id, otherwise highest non shut loopback, highest non-shut int
* RID change drops neighbours
* Ignore Route duprouterid

**Unequal cost load balancing**
* Must be FSs
* Variance
* Max paths

**Add-Path**
* Allows hubs to advertise multiple equal-cost routes
* Might require max paths
* Split horizon disabled
* Variance must be 1
* no next-hop-self (require NH)
* no-ecmp-mode - Deactivates internal optimization, walks over all equal costpaths, any routes successors shoudl be reachable over int which route going to be readv, only applies to successor, not additional entires

**Stub**
* Configured on spokes
* TLV in hello of Spoke
* Does not propagate EIGRP routes to its neighbours, unless a leak map
* router can never be a Transit router (FS for remotes)
* Advertises subset of own networks
 * set with eigrp stub
 * summary, connected, static, redistributed, receive-only
* Might receive quries (no tlv support, or one son same segment), can use other routers too if own routes fail
* Leadk maps use ACLs and prefix lists
* seen in show ip protocols or show ip eigrp neighbors detail

**Route Summarization**

* Query boundary
* Immediate infinite distance reply
* ip summary address eigrp ASN ADDRESS NETMASK [distance] [leak-map NAME]
* summary address ADDRESS NETMASK [leak-map NAME]
 * topology base - summary-metric ADDRESS NETMASK distance admin-distance
* AD 5
* Null0 summary
* AD 255 stops going into table
* Lowest metric component usually summary

**Passive**

* stops sending EIGRP packets

**Graceful Shutdown**

* Can be deactivated on int, AF or process
* Hello with Ks of 255 - goodbye
* Classic only allows GS on v6 process
* v4 is shutting down ints, passive or network statements
* In named,all

**Auth**
* MD5 since start
* SHA 2 since 15.1
* MD5 in classic or named
* SHA in named only
* One key chain min
* SHA cna be on interface
* Send life and accept lifetime

1. Add new keys with higher IDs to all routers
2. Change old keys send lifetime to past
3. Remove old key

**Default routing**
* Redistribution or summarization
* ip default-network to flag candidarte route
* If static route configured with only egress, IOS treats as directly connected

**Split Horizon**
* With poisoned reverse

## EIGRP over TOP

* Uses LISPs, decouples host lcoation from identity
* Mapping service
* Tunnel encaps between hosts
* LISP EID never changes
* RLOC changes (outside address of router)
* Router does ingress/egress tunneling
* Also makes EID-to-RLOC registration and resolution
* OTP - EIGRP replaces LISP control plane
 * Run OTP targetted sessions, IPs as RLOCs
 * Routes are EIDs
 * Similar to DMVPN

## Route Filtering

* Dist list out - All outgoing updates, queries, replies and SIAs, correct metric unless denied (then infinite)
* Dist list in - permitted as normal, denied ignored for Updates, Replies and SIA replies. Queries and SIA-Queries not influenced
* Offset lits - add metrics, adds to delay metric, any route not mathced unchanged, in or out and match interface

# Processes

# Config

```
int lisp0
 bandwidth 10000

int Gi0/0 
 ip address 192.0.2.31 255.255.255.0
 ip route 0.0.0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 1000
  topology base
  exit-af-topology
  neighbor 198.51.100.62 Gi0/0 remote 100 lisp-encap
  network 10.0.1.0 0.0.0.255
  network 192.0.2.31 0.0.0.0
```

* lisp0 as outgoing interface

**LISP Route Reflector**

```
int Lisp0 
 bandwidth 1000

int Gi0/0
 ip address 192.0.2.31 255.255.255.0

ip route 0..0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 1000
  topology base
  exit-af-topology
  af-interface Gi0/0
   no next-hop-self
   no split-horizon
  exit-af-interface
  remote-neighbours Source Gi0/0 unicast-listen lisp-encap
   network 10.0.1.1 0.0.0.0
   network 192.0.2.31 0.0.0.0
```

# Verification


```
show ip eigrp topology all-links
```
