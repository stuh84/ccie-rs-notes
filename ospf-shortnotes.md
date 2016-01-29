* Link state means data about each link and state
* Above is LSDB
* Dijkstra calcs best routes to subnet

# DB Exchange

* Five different messages
* LSA exchange same for single or multiple areas

## Router IDs

* Mandatory before messages sent
* 32-bit ID
* Order is manual config, highest IP on loopback, highest IP on interface (nonshutdown for both)
* Multiple processes try unique RIDs
* No networ command match required
* Int can be down/down
* No route required for advertisment of RID
* No reachability in routing table required
* Changes on process restart or config
* Change causes other routers to reprocess SPF (same as new router)
* If set, never changes

## Becoming Neighbours, DB exchange, adj

* IP proto 89

**Messages**
* Hello - Neighbor discovery, 2 way state, keepalive
* DBD - LSA Headers in initial topology exchange, with versions
* LSR - Request for LSAs
* LSU - Full LSA details (LSR or TC triggered)
* LSAck - Ack's LSU

* LSAs not a message, just a structure

**Down to Full process**

* R1 -> R2 - R1 init, Hello (seen null, RID 1.1.1.1) sent, R2 goes init
* R2 -> R1 - R2 2-way, Hello (seen 1.1.1.1, RID 2.2.2.2) sent
* R1 -> R2 - R1 2-way, Hello (Seen 1.1.1.1, 2.2.2.2, RID 1.1.1.1) sent
* DR election if needed, Hello, DR=z.z.z.z
* R1 -> R2 - R1 Exstart, DD sent
* R2 -> R1 - R2 exstart, DD sent
* R1 + R2 exchange with DD
* R1 + R2 Loading with LSR, LSU and LSAcks exchanged
* Both go Full 

## OSPF Neighbour States

* Only how router sees other side, not adj state
* Both could be in different state, must arrive at same

**Down**
* Initial
* Neighbour torn down
* Manual neighbour no replying to hellos
* Implies router knows about neighbours IP

**Attempted**
* NBMA and P2MP-NB, immediately in attempt state
* Hellos at normal timers
* If no response in dead time, back to down
* Could reduce hello rate

**Init**
* Hello seen
* No own RID in "Seen routers"

**2-Way**
* Hello seen
* Own RID in seen
* Stable if not full adj

**ExStart**
* Master/Slave established
* Empty DBDs compare RIDs
* Agree on common starting seq for future DBD ACKs

**Exchange**
* DBDs exchanged

**Loading**
* LSAs downloaded, stays in this until download complete

**Full**
* FInished loading
* Stable state between fully adj

## Becoming Neighbours: Hello Process

* Discovers routers on subnets
* Check config parameters
* Bi-di vis
* Health check

* 224.0.0.5 on OSPF-enabled ints
* Hellos from primary IP on int
* Fully adj if one or both sides unnumbered

After bidi hellos, must check: -
* Auth
* Same primary subnet and mask
* Same area
* Same area type
* No duplicate RIDs
* Equal hello and dead

* MTU should match so DDs can process, but not part of hello process
* Variable size of routers seen on segment

* Default 10s hello on Broadcast/P2P, 30s on NB and P2MP
* Default 4xhello dead

## Transmitting LSA Headers to Neighbours

* DBD contain seq numbers
* Ack'd by a DVD from neighbour with identical seq
* Window size of one packet, waits for ack before next DBD

## DB Exchange: Master/Slave

* Determined at ExStart
* Master can send DBD of own accord
* Slave only in response, must use same seq number

**Three FLags**
* Master (MS) Flag - Set in DBD by master, clear in slave packets
* More (M) - Set when router needs to send more DBDs
* Init (I) - Inital DBD in exchange, subsequent clear flag

Rules for sending DBDs: -

1. Each DBD sent from master must be replied by slave
2. Slave only sends DBD in response to DBD

* Empty DBD sent if no more to send
* M flag helps for Slave
* M flag also says when to stop (if not set, no more DBDs)

* Lowest RID changes to Slave
* In Exstart, MS, M and I flags set to 1 on both
* Slave sneds DBD with I and MS cleared, seq to seq of masters DBD

## Request, Getting and Acking LSAs

* Seq number of LSA in DBD determines if new
* Seq up on every change/reorigination

**Seq Numbers**
* -2^31 to 2^31 -1
* 2^31 used to detect wrap around
* Negative numbers in Hex go 0x80000001 (-2^31 +1) to 0xFFFFFFFF (-1)
* 0x00000000 (0)
* Finish at 0x7FFFFFFF (2^31`-1)
* When wrap reached, flush LSA and reoriginate back at 0x80000001

* LSU acknowedged by sending exact LSU back to sender or LSAck listing ack'd LSA headers

# Designated Routers on LANs

* Optimize LSA flooding
* DR and BDR on LAN/MA
* Flooding through DR
* LSA goes to DR then flooded to segment
* Double floooding isn't an optimization
* When new router boots up, only sync's with DR and BDR (useful during initial DB sync)
* Create type 2 LSAs (most important part) on MA segment

## DR optimizations on LANs

* If DR and BDR sends an update, sends LSU with new LSA in 224.0.0.5
* Updates towards DR and BDR to 224.0.0.6
* DR and BDR store LSA in LSU in LSDB and flood to 224.0.0.5
* No acks sent, as flooding indicates receipt
* Other routers unicast LSAck to DR
* Without DR< LSUs usually to 224.0.0.5
* DROther for no DR and BDR
* DR Others stop at 2-way

## DR Election on LANs

* DR of 0.0.0.0 means waiting to elect DR (occurs after LAN failure usually)
* Time period to wait is OSPF wait time, same as dead
* If hellos already list DR RID, election process
 * Usually router lost connection to LAN, but others up
* Newly connected routers do not attempt to elect new DR (assume DR listed is current)

**DR Election**
* 1-255 priority can take part, 0 cant
* Done local based on data and algorithm
* Priority and RIDs collected during wait
* If during wait, hello with a claimed BDR, or if DR claimed and no BDR, router begins process
* Only for roles not claimed
* Highest priority as DR, second as BDR, if same priority, highest RID as tie break
* No preemption
* If DR fails, BDR to DR, new election for BDR

If different result achieved, usually means network partitioned (eg STP), reenter election phase

## DR on WAN and OSPF Network Types

Network type dicates: -
* DR or none
* Static neighbour config
* More than two neighbours on same subnet

* DR/BDR - B'cast, NBMA
* Hello of 10s - B'cast, P2P
* Hello of 30s - NBMA, p2MP, P2MP NB
* Static neighbour - NBMA, P2MP NB
* More than two hosts - All but P2P

Loopback also network type, only on lo ints

* P2P default for FR p2p subints
* p2MP default for FR phy and MP subints

## Network types over NBMA networks

If OSPF network types don't match over FR: -

* Make sure hello/dead match
* If DR for one side, other does't need, no full LSA exchange
* If DR used, DR and BDR must have PVC to other routers in subnet (can't learn LSUs, and type 2 can't generate)
* Can set static neighbour config on one side only

* Best to use P2P subints
* If not possible, use **ip ospf network point-to-multipoint**

If different neighbours reachable over different ints, following applies
* Priority in neighbor command not used in DR and BDR
* If multiple neighbour statements and one or more with priority, hellos only to those neighbours with nonzero
* Only after DR and BDR elected will hellos go to others
* If neighbour priority matches int priority, router engage in DR/BDR with them neighbours first
* If no priority, all neighbours contacted imediately
* If set to 0 in **ip ospf priority** command, neighbour statements removed from config

# SPF Calculation

* LSAs have info for math equivalent of figure of network
* Routers, links, costs and admin status
* SPF constructs least cost paths to all possible destination
* Costs summed, least total cost taken
* Cost per interface
* Outgoing int cost used

## SSO

Following true in steady state: -

* Hellos send per hello interval
* Hellos in dead time, or neighbour failed
* LSAs reflooded (with seq increment), LSRefresh interval (per LSA), default 30m
* MaxAge for LSA is 60m

# OSPF Design and LSAs

## OSPF Design Terms

**ABRs**
* Between areas
* Must connect to area 0
* Cisco says must connect for it to be be an ABR 


**ASBR**
* External routes

* Separate LSDB for each area
* Per-area LSDB isolated
* Only ABR can translate/carry info between
* SPF ran in each LSDB separately

Benefits of areas: -
* Smaller per-area LSDB
* Faster SPF calc
* Link failure only partial SPF calc in other areas
* Summarize/filter at ABRs and ASBRs
* LSDB srhinks as type 3 LSAs (sparser than type 1 and 2s)

## Path selection

* Intra > Inter > External
* Ignore type 3 LSAs in nonbackbone area during SPF calc
 * Prevents ABR going into nonbackbone and back in to backbone elsewhere

## LSA types

* Only modified by original router
* Processed and flooded with in flooding scope
* Never blocked, dropped (unless lifetime expires or changed)

|Type|Name|Description|
|----|----|-----------|
|1|Router|One per router per area, lists RID and all int IPs in area, represents stubs, flooded within area|
|2|Network|One per transit network, DR originated, represents subnets/router ints connected to subnet, within area|
|3|Net Summary|By ABR, represents networks present in on area, defines subnets in area and cost, nothing else. Flooded only within its area of origin, reoriginated on ABRs|
|4|ABSR Summary|Like type 3, host routed to each ASBR, within area of origin, reoriginated on ABRs|
|5|AS External|Created by ASBRs for external routes, flooded to all regular areas|
|6|Group Membership|For MOSPF, no IOS support|
|7|NSSA External|By ASBRs inside NSSA, flooded in area, converted to type 5 by ABRSs|
|8|External Attributes|Created by ASBRs, during BGP-to-OSPF redist, preserves BGP attributes, not in IOS|
|9-11|Opaque|Future expansion, Type 10 is MPLS TE, Type 9 link local, type 10 area local, type 11, as flooded|



