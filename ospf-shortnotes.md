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

## OSPF Costs and Clearing OSPF Process

* **clear ip ospf process** - All processes
* **log-adjacency-changes detail** - Messages at each state change
* Default cost is 100,000 kbps / bandwidth
* **ip ospf cost** on interface
* **auto-cost reference-bandwidth**, mbps units, default 100
* **neighbor X.X.X.X cost value** - Per neighbour

Order chosen
* Neighbor cost (only valid on P2MP-NB
* IP ospf cost
* Default based on int bandwidth
* auto-cost

Int costs in IOS kbps, auto-cost mbps

## Network command alternative

* From 12.3(11)T
* **ip ospf 1 area area-id** on interface
* All secondaries matched rather than explicit with network commands
* Avoid with **secondaries none** keyword

## OSPF Filtering

* Filtering routes, not LSAs - **distribute list**, filters routes from SPF to routing table, doesn't affect LSDB
* ABR type 3s - Prevent ABR creating a type 3
* **area range no-advertise** - Prevents particular type 3 from ABR

### Distribute list

Following rules: -

* Inbound direction - Results of SPF, not prior
* Outbound - Only on redistribtued and ASBR
* Inbound does not filter inbound LSAs
* If incoming interface added, checks as if it were outgoing int of route


```
router ospf 1
 distribute-list prefix prefix-list-1 in Serial 0.2

router ospf 1
 distribute-list route-map rm-1 in

route-map rm-1 deny 10
 match ip address 48
 match ip route-source 51 # Specify route source, eg permit 2.2.2.2
```

Earlier IOS allowed replacing with next best route, 12.4 and beyond don't ad the route to routing table

### ABR LSA type 3 filtering

* Identical LSDBs in each area still met
* Filters at point type 3s created (ABR)
* **area 1 filter-list prefix name { in | out }

* in - Prefixes going into area
* out - Oposite of above

### Area Range

Route summarization at ABR, but with **not-advertise** keyword, meaning no summary either

## Virtual Link

* Connects areas to backbone that go via none backbone
* Targetted OSPF session
* Not a tunnel for data
* Makes two remote routers within single area fully adjacent, syncs LSDBs
* VL goes throug regular area only
* Packets forwarded based on true dest address, meaning transit area must know all networks

```
R1

router ospf 1
 area 3 virtual-link 3.3.3.3

R3

router ospf 1
 area 3 virtual-link 1.1.1.1
```

**show ip ospf virtual-links**

## Classic OSPF auth

* None, clear text and MD5
* SHA-1 exists, but different config

Rules: -
* Type 0 (none), type 1 (clear text), type 2 (MD5)
* **ip ospf authentication** - Enables on int
* Default type 0
* Default changed with **area authentication** router ospf command
* Keys always under int
* Multiple MD5 keys with different key IDs allowed per int

* Sent packets always use key added as last one to interface
* Rx'd use key ID
* Key migration phgase if different key number seen than this router
 * Sends all packets as many times as keys config'd, each with different key
 * Phase ends when all neighbours use same key
* Key rollover procedure, not available for clear text

**None**

```
int Fa0/0
 ip ospf authenticaiton null
```

**Clear text**

```
int fa0/0
 ip ospf authentication
 ip ospf authentication-key key-value
```

**MD5**

```
int Fa0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key key-number md5 key-value
```

* **area authentication** required prior to IOS 12.0
* No area auth means type 0 default
* **area 0 authentication** - type 1
* **area 0 authenticaiton message-digest** - type 2

* Keys in clear text unless using **service password-encryption**

Virtual links: -

```
area 0 virtual-link x.x.x.x authentication null
area 0 virtual-link x.x.x.x authentication
area 0 virtual-link x.x.x.x authenticaiton message-digest
```

## Extended Crypto Auth

* IOS 15.4(1)T up
* SHA-HMAC supported
* Key chains used
* Crypto algorithm per key
* Must use **cryptographic-algorithm** command, otherwise not used
* **send-lifetime** and **accept-lifetime** possible
* Highest ID used if multiple
* Key rollover not used, migration like EIGRP instead
* Enabled per int with **ip ospf authentication key-chain key-chain-name**
* Virtual links - **area 0 virtual-link x.x.x.x key-chain key-chain-name**
* MD5 auth supported like this, ignores **ip ospf message-digest-key** if config'd already

```
key chain ospf 
 key 1
  cryptographic-algorithm hmac-sha-1/256/384/512/md5
  key-string DAVE

int Gi0/0
 ip ospf authentication key-chain ospf
```

**show ip ospf interface** - Shows crypto auth enabled

## Protecting routers with TTL security check

* Avoids unicast OSPF packets across network
* Besides virtual/sham links, comms should be direct, ttl shouldn't decrement
* **ip ospf ttl-security** - per int
* **ttl-scurity all-interface** - per process
* **ip ospf ttl-security disable** - per int
* All have **hops hop-count** for relaxing TTL security check. 
 * Stting to 100 would say from 255 to 155
 * 254 means effectively disabling check
 * Above useful for migration to TTL security
* Value of 1 is default
* IOS based routers decrement TTL of received OSPF packets before handing to OSPF process

* Enable on VLs or SLs with **area virtual-link ttl-security hops** and **area sham-link ttl-security hops** - Hops mandatory

## Tuning OSPF Performance

### SPF scheduling with SPF throttling

* Used if links flapping
* Cisco default runs SPF 5 seconds after updated LSA
* If LSA arrives after above run, subsequent delay grows by up to 10 seconds
* Controlled by SPF throttling
 * Defines variable wait interval between two SPF runs

* spf-start - Initial wait before SPF
* spf-hold - Wait between SPF runs
* spf-max-wait - Max time between two SPF runs, and time network must be stabl for wait interval to be set back to spf start and hold to its preconfig'd value

* If network stable in last-spf hold, but not spf-max-wait, wait returns to spf start, but subsequent wait twice previous spf-hold

* **timers throttle spf spf-start spf-hold spf-max**, all in ms
* Current values in **show ip ospf**
* **debug ip ospf spf statistic** - verifies current and next wait intervals

### Tuning LSA origination with LSA throttling

* Rate at which an LSA might be reoriginated by its router
* After LSA not updated for more than max-interval, created and flooded after star-interval
* Next wait interval set to hold-interval
* If same LSA updated within wait since last reorigination, origination/flooding postponed until wait expires
* After above, then creating and flooding, hold doubled
* If no updated in next wait, wait back to start. Hold still at higher value
* If same LSA updated after current wait but before max, reorigination and flooding done based on start, but next wait twice hold
* Hold reset to configured value if LSA not required to be updated for entire max-interval

Defaults
* Start - 0 (LSA updates immediately)
* Hold and max - 5000ms (subsequent origination 5 seconds, no progressive increase)

* Set with **timers throttle lsa all start-interval hold-interval max-interval** - ms
* Show above in **show ip ospf**
* Can ignore an LSA upon arrival if too often, **timers lsa arrival millisconds**
* Ignored if same LSA in that time
* Default arival is 1000ms
* Should be smaller than inital hold in LSA throttling, otherwise neighbour sending sooner than would be accepted

### Incremental ISPF

* Only affected part of SPF tree recalc'd
* Further away change occurs, bigger effect
* **ispf** command
* Can be enabled per router, not required through entire network

### OSPFv2 Prefix Suppression

* Most hosts located in nontransit networks
* Most devices have loopbacks for mgmt
* Maintaing transit link prefixes in this state is pointless
* RFC 6860 hides/suppresses transit link prefixes
* Suppresses prefixes, not links themselves
* Transit links are connections be routers, must be advrtised

Type 1 for router and its adjacencies, four possible link types in type 1
* P2P to another router - Transit link pointing to other routers RID, no addresing info, not influenced by prefix suppression
* Link to transit network - Points to transit networks DR IP, no addressing info like network max, only DR IP, not influenced
* Stub network - All stubs in type 1 corresponding to prefixes used on P2Ps can be supressed
* Virtual link - Virtual transit p2p, virtual adj, no address info, no suppression

* Omits stub network entires for p2p ints to other routers

* For type 2s, IP Prefix computed by bitswise AND of LSID of LSA and netmask in payload. 
* RFC 6860 says settings netmask to 255.255.255.255
* Routers supporting it see signal of LSA containing no prefix info
* Routers not implementing RFC install host route to DR

* **prefix-suppression** - works on all ints except loopbacks
* **ip ospf-prefix-suppression [disable]** - per int

### Stub Router Config

* RFC 3137, obsoleted by 6987
* IOS 12.2(4)T and up
* Router temporarily or permanently not a transit
* Advertises own type ones with infinite metrics for all trasnit type adj (p2p, transit links, VLs)
* Stub network adj advertised with real int metrics
* Can be set to advertise for a time period, or completion of BGP convergence

**max-metric router-lsa on-startup announce-time** - Seconds

**max-metric router-lsa on-start wait-for-bgp** - Waits until BGP signals convergence or 10m pass

### Graceful Restart

* RFC 3623
* For platforms forwarding packets while process stops
* NSF before GR (NSF Cisco proprietary)
* Router is in GR mode, its neighbours in heper mode
* Lack of hellos for grace period ignored
* Considers fully adjacent, reports as so in type 1/2 LSAs
* Still seen as DR
* Every IOS router can be helper if supported
* Routers with specific hardware required for GR
* NSF-Aware (helpers), NSF-Capable (helpers and GR)
* NSF capable are 6500, 7200, 7300, 7600, 10000, 12000, ASR and CRS

Can continue forwarding without loops assuming following: -
* Routers hardware can go while line cards use last FIB version
* OSPF processes must let neighbours know with grace LSA (type 9)
 * Contains estimated restart duration (grace period)
 * Restart reason
 * IP of router
* LSDB stable during restart
* All neighbours support and config'd as helper
* Restarts within grace
* Fully adj neighbours must operate in helper

* CEF used uring restart
* OSPF rebuild RIBs
* Cisco and IET awareness enabled by default in IOS
* **nsf [ cisco | ietf ] helper disable

### Graceful Shutdown

Using shutdown command in process
* Drops all OSPF adj
* Flush all LSAs (3600 max age)
* Hellos sent with DR/BDR to 0.0.0.0, emtpy neighbour list (adj fall to init)
* Stop OSPF packets

Per int: -
* Drop Adj on int
* Flood LSAs that no longer include that int and adj through other ints (if any)
Hellos as above
* Stop OSPF packets on that int
* **ip ospf shutdown**

# OSPFv3

* RFC 5340

## Diffs between v2 and v3

* Interface commands, eg **ipv6 ospf 1 area 2**
 * **ipv6 router ospf 1**
* Advertises multiple networks on interface - All advertised if on int
* RID must be set - Cannot automatically choose RID if no v4 set (if v4 set, same rules as v2)
* Flooding scope
 * Link-local scope - Link LSA
 * Area scope - Flooed through area (Router, Network, Inter-Area Prefix, Inter-Area Router, Intra-Area Prefix)
 * AS Scope - AS External
* Multiple instances per link
* v3 uses links, v2 networks
* Source packets from link-local, VLs from globally coped v6
* Auth - v6 covers rather than protocol
* Networks in LSAs - prefix, prefix length, rather than address, mask

## LSA Types

|Type|Name|Description|Flooding Scope|
|----|----|-----------|--------------|
|1|Router|Routers and links to neighbors within area|Area|
|2|Network|From DR, represents multiaccess transit network and connections to member routers|Area|
|3|Inter-Area Prefix|From ABRs, networks in other areas|Area|
|4|Inter-Area Router|From ABRs, shows ASBRs in other areas|Area|
|5|AS External|From ASBR, networks from other protocols|AS|
|7|NSSA|ASBR in NSSA, as per 5|Area|
|8|Link|Link local address and prefix of router to all other routers on link and option info, sent only if more than one router on link|Link|
|9|Intra-Area-Prefix|List of v6 prefixes with transit network (points to Network LSA) or v6 prefixes with a router (points to Router LSA)|Area|

* Type 1 and Type 2 have have prefix info placed into Type 8s and 9s instead
* Type 1 and 2 only carry topology adj
* Type 9 has prefixes of a router, carried with area scope
* With link local next hops, each router advertises a type 8 out particular int
* Only type 1 and 2 updates trigger SPF
* Type 8 and 9 originated and flooded, no SPF run

## v3 in NBMA

* Per interface **ipv6 ospf neighbor**
* Other end must be link-local, other addresses rejected
* Other v2 rules apply

## v3 over FR

* No InverseARP
* All v6/DLCI mappings manual
* Mappings needed for link-local and global
* Link-local need broadcast keyword (although in NB or P2MP NB)

```
int Se0/0
 frame-relay map ipv6 FE80::1 708 broadcast
 frame-relay map 2001:207::1 08
```

Change link local address with **ipv6 address ADDRESS link-local**

## Enabling and Configuring v3

```
ipv6 unicast-routing
ipv6 cef

int lo0
 ipv6 address 3001:0:3::/64 eui-64
 ipv6 ospf 1 area 704

int lo1
 ip address 10.3.3.6 255.255.255.0

int lo2
 ipv6 address 3001:0:3:2::/64 eui-64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 0

int Fa0/0
 ipv6 address 2001:0:3::/64 eui-64
 ipv6 ospf 1 area 704

int Se0/0
 bandwidth 128
 encapsulation frame-relay
 ipv6 address 2001::/64 eui-64
 ipv6 ospf neighbor FE80::2
 frame-relay map ipv6 FE80::1 807 broadcast
 frame-relay map ipv6 2001:207::1 807

ipv6 router ospf 1
```

Verify with **show ipv6 interface brief**, **show ipv6 protocols**, **show ipv6 ospf interface**, **show ipv6 router ospf**

* Redistributing same as v2

## v3 Auth and Encryption

* 12.3(4)T - AH supported
* 12.4(9)T - ESP supporte

* Enable AH with **ipv6 ospf authentication**
* Encryption with **ipv6 ospf encryption**
* Cannot have both on one int, ESP does both auth and encryp
* Supply slected crypto algorithms for has and encryp and keys
* Defines how packets proected by IPSec
* SA identified by SPI (Security Parameter Index)
* Each v3 packet proected by IPSec carries SPI number of SA to protect it
* RX router uses SPI to identify SA

```
int Fa0/0
 ipv6 ospf auth ipsec spi 1000 sha1 KEY

int Se1/0
 ipv6 ospf encryption ipsec spi 1001 esp aes-cbc 128 KEY

ipv6 router ospf 1
 area 1 authenticaiton ipsec spi 1002 md5 KEY
 area 2 encryption ipsec spi 1003 esp 3des KEY md5 KEY
```

* Can use authentication trailer, configured like v2 extended auth
* 15.4(2)T support
* Refer to **ospf v3 authentication key-chain*
* IPSec not compatible with above
* Does not provide encryption

* Per link auth and encryp preffered as more layers of security
* Auth trailer an AH for auth
* ESP for auth and encrypt
* Same SPI number, AH/ESP mode, algoritihm and keys required
* For auth trailer, same crypto algorithms, key IDs and strings

## Address Family Support

* Prefix info in new LSA types
* AF support possible
* Instance IDs split into different categories for different AF families

* 0 - Base v6 unicast
* 1-31 - v6 unicast with local policy
* 32 - Base v6 multicast
* 33-63 - v6 multicast local policy
* 64 - v4 unicast base
* 65-95 - v4 unicast local policy
* 96 - v4 multicast base
* 97-127 - v4 multicast local policy
* 128-191 - Unassigned
* 192-255 - Reserved for private use

* Use base instance ID if not config'd
* Packet exchange, LSA flood/originate, SPF computation and routing tables independent of one another
* Options v3 bitfield in v3 Hellos, DBDs and LSAs AF-bit efined
* Set if sending router supports AF extension and this instance isn't v6 unicast
* If packet seen in AF instance with no AF bit, implies other side doesnt support AFs
* Router drops packets, never establishes adj
* Encap of v3 packets stays same (v6)
* Interface must have v6 addressing on to allow AF operation
* v3 packets for v4 still in v6
* VLs only for v6 unicast AFs

Config as such: -

```
int lo0
 ipv6 address 2001:DB8:0:FFFF::1/128
 ip address 10.255.255.1 255.255.255.255
 ospfv3 1 ipv6 area 0 
 ospfv3 1 ipv4 area 0

int Fa0/0
 ipv6 address 2001:DB8:1:1::1/64
 ip address 10.1.1.1 255.255.255.0
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ospfv3 1 ipv4 area 1

int Se0/0/0
 ipv6 address 2001:DB8:0:1::1/64
 ip address 10.0.1.1 255.255.255.0
 ospfv3 hello-interval 1
 ospfv3 1 ipv6 area 0
 ospfv3 1 ipv4 area 0

router ospfv3 1
 address-family ipv4
  area 1 range 10.1.0.0 255.255.0.0
 address-family ipv6
  area 1 range 2001:DB8:1::/48
```

## Prefix Suppression

* Omits suppressed transit link prefixes from type 8s and 9s
* Per process with **prefix suppression**
* Per int with **ipv6 ospf prefix-suppression** or **ospfv3 prefix-suppression**
* Can be done per AF, or outside (global process)

## Graceful Shutdown

Modified process
* Hellos with router priority of 0 (drops DR/BDR if applicable)
* Stops accepting hellos
* Flushs all originated LSAs except type 1s
* Flood type 1s with links in that LSA with cost 65535
* After dead interval expires and neighbours dead, flush own type 1
* STop ospfv3 packets

More gradual, over declared dead interval
