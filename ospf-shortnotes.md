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

* Transit network - Two or more router neighbours and elected a DR so traffic can transit from one to another. P2P treated as a combination of P2P and stub on this link
* Stub - Subnet where no neighbour relationships formed

### Type 1 and 2

**Type 1**
* From router
* Describes router, ints in that area
* List of neighbouring routers in that area
* Defined by LSID, equal to RID

**Type 2**
* Transit subnet which DR elected
* LSID is DRs int IP on subnet
* None for networks without DR

* SPF from 1s and 2s creates topological graph of network
* Calculates best and chooses best routes

* Without DR, type 1 has enough info
* With DR< type 2 models subnet as node in SPF model
* Type 2 sometimes known as pseudonode
* Type 2 has RIDs of all neighbours of DR
* With Type 1s for each router in subnet, accurate network pictrue

**show ip ospf database** - Link ID should be LSID. LSID is unique ID for LSA, Link ID an entry in type 1s

* For down networks, type 1s and 2s reoriginated, disconnected ntwork removed from LSA, or entire LSA aged (3600s and flood)

### Type 3 and Inter-Area Costs

* ABRs dont forward Type 1s and 2s between areas
* Type 3s into one area
* Type 3 describes interarea dest with subnet, mask and ABRs cost to subnet
* Cost calc'd from router as ABR cost plus type 3 cost
* **show ip ospf database summary** - type 3 cost
* **show ip ospf border-routers** - ABR costs
* If network disappears, ABR removes type 3 for the networks
* If another type 3 doesn't exist in routers DBs, removed from routing tables
* Can update metric to 16777215
* Premature aging preferred (RFC2328)

* Partial calc not dependent on summary routes
* Type 3s flooded only within area into which they were originated by ABRs, dont cross area boundaries
* ABRs compute internal OSPF routing table for backbone area using all types of LSAs
* New type three for each intra and intra-area route, originated and flooded to nonbackbone areas

* ABRs use only type 3s received over backbone area in SPF calc
 * Non-backbone stored in LSDB, flooded within nonbackbone area
* When ABR creates and floods type 3s, only intra-area routes from non-backbones advertised into backbone
* Inter-and-intra from backbone

### LSA Type 4 and 5s, E1s and E2s

* E1 - External and internal metric
* E2 - External only (IOS Default)

* When external route injected, type 5 for subnet by ASBR
* Lists metric and type
* Flooded through all regular areas
* Processed depending on metric
* If E1, total cost is cost to ABSR plus E1 in LSA
* If multiple paths to same E1, least cost used
* E2 only external costs, internval viewed as negligible costs
* E1 > E2 routes
* Both need cost to ASBR
 * In same area, least-cost path with type 1s and 2s
 * Type 4 in other areas (contains ABSRs RID and ABR metric to reach it)
* Type 4 only flooded in other areas

### OSPF Design with LSA types

* Areas cut down SPF calc
* Link flaps less effect
* Summary routes reduce type 3s and 5s

### Stubby Areas

* Not all areas need to know about each external
* Packets must still go through an ABR in many cases, and no ASBR in current area
* Stubbys inject default route into area, so ABR defualt at all times
* If area is stubby, stops type 4 and 5s into area
* Every internal router in stubby area ignores type 5s, no origination itself
* ABR automatically injects default as type 3
* Visibility of intra-area and inter-area networks in stubby not affected
* Types 4s not mentioned, but useless anyway if no type 5s

Four types exist: -

|Type|Allowed LSAs|Ignored LSAs|Generated LSAs|
|----|------------|------------|--------------|
|Stubby|Type 3s|4 and 5|None|
|Totally Stubby|None|Type 3, 4 and 5|None|
|NSSA|Type 3s|4 and 5|7s|
|NSSA-TS|None|3, 4 and 5s|7s|

* TSs only allowed type 3 default
* NSSA for externals

```
area area-id nssa
area area-id nssa no-summary
area area-id stub
area area-id stub no-summary
```

* NSSAs for potential local breakout etc
* Type 7 changed to type 5 at ABR
* ABR with highest RID performs translation
* NSSA does not have default auto gen'd, need to do **area area-id nssa default-information-originate**
* NSSA-TS not required
* N1 and N2

## OSPF Path Choices, not cost

### Best Type

1. Intra-area
2. Inter-area
3. E1/N1
4. E2/N2

### ABR loop Prevention

* DV between areas
* Type 3s only have subnet, metric and ABR
* Split Horizon applied for many LSA types
 * Makes sure info from LSA not advertised into one nonbackbone area and back into backbone

* No inter-area routes from nonbackbone can go to backbone, means ABR does not go via non-backbone to reach backbone


# OSPF Config


```
R1
 
int Fa0/0
 ip address 10.1.1.1 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4

router ospf 1
 area 3 nssa no-summary
 area 4 stub no-summary
 area 5 stub 
 network 10.1.0.0 0.0.255.255 area 0
 network 10.3.0.0 0.0.255.255 area 3
 network 10.4.0.0 0.0.255.255 area 4
 network 10.5.0.0 0.0.255.255 area 5

R2

int Fa0/0
 ip address 10.1.1.2 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf 2 area 0

router ospf 2
 area 5 stub

R3

router ospf 1
 area 3 nssa no-summary
 network 10.0.0.0 0.255.255.255 area 3

R4

router ospf 1
 area 4 stub no-summary
 network 10.0.0.0 0.255.255.255 area 4

S1

int vlan 1 
 ip address 10.1.1.3 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4

router ospf 1
 router-id 7.7.7.7
 network 10.1.0.0 0.0.255.255 area 0

S2

int vlan 1
 ip address 10.1.1.4 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf priority 254
```

R3 and R4 don't require no summary, but better to have anyway for consistency.


