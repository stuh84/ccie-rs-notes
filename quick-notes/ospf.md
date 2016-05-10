# Messages

* Hello - Keepalive, neigh discov, 2 way
* DBD - LSA headers with versions
* LSR - Request for LSAs
* LSU - Full LSA details
* LSAck

# Timers

* Hellos every 10s or 30s (network type)
* Dead 4 times hello default
* LSRefresh default 30m
* 60m Max Age

# Trivia

* RID mandatory before messages sent, manual config, highest IP on loopback, highest IP on int
* Multiple processes try unique RID
* int can be down down
* IP Proto 89
* Fully adj if both sides unnumbered
* After bidi hellos, much have same auth, primary subnet and mask, area, area type, no duplicate RIDs, equal hello and dead
* MTU match, not part of hello though

## DB Exchange - Master/Slave
* Flags - Master, More, Init
* Lowest RID becomes slave
* MS, M and I on exstart
 * Slave sends DBD with I and MS cleared, seq to masters seq (seen in DBD)

## Requesting, Getting and Acking LSAs

* -2^31 to 2^31 (detects wrap around)
* When wrap reached, flush LSA and back to 0x80000001
* LSU acks by exact LSU back to sender, or LSAck listing ACK'd headers

## DRs

* LSA to DR then flooded to segment
* Creates type 2 LSAs
* When new router boots, syncs only to DR and BDR
* Other routers LSAck to DR when new LSUs arrive
 * 224.0.0.5 when no DR, otherwise 224.0.0.6 for sending LSUs

### Election

* Time period to wait is OSPF wait time (waiting to elect DR, DR of 0.0.0.0)
* If hello lists DR RID already, election
* No preememption
* 0 priority means not DR
* Done on local data
* If during wait, hello with a claimed BDR, or DR claimed and no BDR, begins
 * Only for roles not claimed
* Highest priority as DR, second as BDR, highest RID as tie break
* If DR fails, BDR to DR, new BDR election

### DR on WAN and OSPF network Types

* DR only on B'Cast and NBMA
* Hellos of 10 on B'Cast and p2p
* 30s on others
* Static neigh on NB types
* More than two hosts on any but p2p
* P2P default for FR P2P
* P2MP default for FR phy and MP ints

### Network types over NBMA

* Make sure hello and dead match
* If DR for one side and not other, no LSA exchange
* If DR and BDR, must have PVCs to other routers (otherwise can't gen type 2)
* Can set static config on one side only

### Neighbours over different ints
* Priority in neighbour command not for DR and BDR
 * Hellos only to nonzero first
 * DR and BDR then elected, then hellos to others
 * If neighbour priority mathces int priority, DR/BDR with them first
 * If no priority, all neighbours contacted immediately
 * If set to 0 in ip ospf priority, neighbour statements removed

## Areas

* Separate LSDB per area, isolated
* Only ABR can translate
* SPF in each LSDB separately
* Can summarize/filter at ABRs and ASBRs

## Path Selection

* Intra > Intra > External (E1 > E2)
* E1 has path metric, E2 not

## LSA types

1. Router - One per router, RID and all ints, represents stubs
2. Network - DR originated, represents subnets/ints to subnet
3. Net summary - By ABR, networks present in one area, with cost. Flooded only within area of origin, reoriginated by ABRs
4. ASBR Summ - Like type 3, host route to each ASBR, within area of origin, reorig on BAR
5. AS External - By ASBRs, flooded to all regular areas
6. Group Membership - MOSPF, no IOS support
7. NSSA External - By ASBRs inside NSSA, converted to type 5s by ABRs, flooded in area
8. External attributes - By ASBRs, during BGP to OSPF redist, not in OSPF
9 through 11. Future expansion, type 10 is TE

* Transit network - Two or more router neighbours elected DR so traffic can transit, P2P treated as combination of P2P and stub on this link
* Stub - no neighbour relationship

**Type 1**
* List of neigh routers
* Defined by LSID, equal to RID

**Type 2**
* Transit subnet which DR elected
* LSID is DRs int ip on subnet
* None for networks without DR, enough info already
* Can be thought of as Pseudonode
* Has RIDs of all neighbours of DR

* For down networks, 1 and 2 reorig with disconnected network removed from LSA, or Aged (3600s and flood)

**Type 3 and Inter-Area Costs**
* Type 3s into one area
* Describe interarea dest with subnet, mask and ABRs cost to subnet
* If network disappears, ABR removes type 3 for the networks
 * If another type 3 doesn't exist in DB, removed from routing table
* Can update metric to remove, preferred to premature age
* Partial calc not dependent on summary routes
* Type 3s don't cross area boundaries (so get originated at ABR to an area, based on networks from another area)
* Type 3s only from backbone, do not use them from outside backbone, other than forwarding back into nonbackbone area
* Only intra-area routes from non-backbone into backbone
* Inter and intra from backbone

**Type 4 and 5**
* Lists metric and type
* Flooded through all regular areas
* Type 1 and 2 within area for cost to ASBR, Type 4 in others
* Type 4 only flooded in other areas

**Stubby**
* ABR with highest RID performs type 7 to 5 translation
 * When router forced to pick forward address for type 7, preference to routers internal address, otherwise routers active stub network address
 * avoids extra hops that may happen when transit networks address used
 * Forward Address seen in LSA Output
 * Newest enabled OSPF loopback IP, then newest enabled non loopback for transit stub
* NSSA does not default auto gen, `area ID nssa default-information-originate`
* NSSA-TS does

**ABR Loop Prevention**
* DV bw areas
* Split Horizon for many LSA types - makes sure info from LSA not adv into one nonbackbone and back into backbone
* No inter-area routes form nonbackbone to backbone (so no transit to backbone via nonbackbone)

## Demand Circuit

* suppressed periodic hellos and LSARefresh
* DC option bit exchanged
 * If neg'd, sets bit in LSA age called DoNotAge
 * LSARfresh - only sent on TC or router that doesn't understand it
* If ABR sees router that cannot process demand cricuits, tells originating router to not set DNA bit

## Filtering

* Filters routes, not LSAs
 * Distribute list
 * area range no-advertise - prevents particular type 3 from ABR

### Distribute list

* Inbound - results of SPF
* Outbound - only on redistributed and ASBR
* If incoming int added, checks as if outgoing int of route

### Type 3 filtering

* Filters at point type 3 created
* `area 1 filter-list prefix NAME {in | out}`
 * in - prefixes going into area
 * out - prefixes going out of area

### Area Range

* Uses summarization but with not-advertise

## Virtual Link

* Through regular area only
* Packets forwarded based on true dest, so transit area must know all networks

## Classic OSPF Auth

* None, clear text, MD5 (type 0, 1 and 2)
* `ip ospf authentication` on interface
* Default type 0
* Change default with `area authentication` under process
* Keys always under int
* Multiple MD5 keys with different IDs allowed
* Sent packets always use key added as last one to int
* Rx'd uses key id

**Key migration**
* Send all packets as many times as keys config'd
* Ends when all neighbours use same key

## Extended Crypto Auth

* 15.4(1)T up
* SHA supported
* Key migration like EIGRP

## TTL Security

* `ip ospf ttl-security` per int
* `ttl-security all-interface` per process
* `ip ospf ttl-security disable`
* All have `hops hop-count` to specify
* 254 disables check
* Value of 1 default
* Only VLs or SLs higher TTL, rest should be link local
* `area virtual-link ttl-security hops` - hops mandatory

## Tuning

### SPF Scheduling and Throttling

* Cisco default 5 second SPF run after updated LSA
* If LSA arrives aftere above, subsequent delay grows by up to 10 sec
 * SPF Throttling
* spf-start - Initial wait before SPF
* spf-hold - Wait between runs
* spf-max-wait - Max time between runs, and time network must be stable for wait to be back to SPF start
* If network stable in last spf-hold, but not spf-max-wait, wait returns to spf start, but subsequent wait twice previous spf-hold
* `timers throttle spf spf-start spf-hold spf-max` - ms
* LSA rx'd, waits start time and runs SPF, expects nothing in next hold.
 * If LSA rx'd in next hold, waits until hold finished then runs
 * Next hold is doubled (so if previous hold was 5 seconds, is now 10 seconds)
 * Process repeats until hold gets to spf-max-wait
 * Hold stays at max-wait until stability (next hold after instability, plus another of no LSAs)
 * If above achieved, spf-hold back to original value

### LSA Origination and Throttling

* After LSA not updated for more than max, created and flooded after start-interval
* Next wait set to hold
* If same LSA updated, flooding postponed until wait
* after above, hold/wait doubled
* If no update in next wait, wait back to start, hold at higher value still
* If same LSA updated after current wait but before max, flooding done at start, but next wait twice hold
* Hold reset if LSA not required during entire max
* Default start of 0
* Hold and max 5000ms (no progressive increase)
* `timers throttle lsa all start-internval hold-interval max-interval`
* Can ignore an LSA upon arrival if too often, `timers lsa arrival MS`
* Ignore if same LSA in that time
* Default is 1000ms
* Should be smaller than initial hold in LSA throttling, otherwise sending sooner than accepted

## Incremental ISPF

* Can be enabled per router, `ispf`
* Only affected part of SPF tree recalc'd

## OSPFv2 Prefix Suppression

* Suppresses transit link prefixes, not links themselves
* All stubs in type 1 corresponding to prefixes used on P2Ps can be suppressed
* Omits stubs for p2p ints to other routers
* For type 2s, IP prefix computed by bitwise AND of LSID and netmask
* Routers not implementing RFC install host route to DR
* `prefix-suppression` - all ints except loopback
* `ip ospf-prefix-suppression [disable]` - per int

## Stub Router Config

* Makes router a non transit
* 12.2(4)T and UP
* Can be set to advertise on time period or completion of BGP convergence
* `max-metric router-lsa on-startup TIME - seconds`
* `max-metric router-lsa on-start wait-for-bgp` - Waits for BGP convergence of 10m

## Graceful Restart

* RFC 3623
* NSF before GR (cisco prop)
* Router in GR mode, neighbours in helper
* Lack of hellos in grace ignored
* Considers fully adjacent
* Still seen as DR
* every IOS router can be helper
* Mainly chassis as GR
* Can continue forwarding assuming line cards use last FIB version, OSPF process uses grace LSA (type 9) with grace period, restart reason and IP, LSDB stable during restart, all neighbours support and config'd as helper
* Fully adj neighbours must be helper

## Graceful Shutdown

**Under process**
* Drop all OSPF adj, max age LSAs, hellos with 0.0.0.0 DR BDR, empty neighbour list, stop OSPF packets

**Under Int**
* As above but withour DR and BDR stuff

## OSPFv3

* Advertises multiple networks per int
* RID Must be set (no automatic choosing)
* Link LSA link local, AS scope for AS external, rest area scope
* Multiple instances per link
* Auth done by v6 itself
* Only type 1 and type 2 trigger spf, type 8 and 9 do not

### V3 in nbma

* As per v2, other end must be link-local, other addresses rejected in `ipv6 ospf neighbour` command

### v3 over FR

* No InverseARP so need v6/DLCI mappings
* Link local needs broadcast keyword

### Auth and Encryption

* Enable AH with ipv6 ospf authentication, ipv6 ospf encryption for ESP
* ESP or AH, not both, ESP does auth and encryp
* Need to define ipsec SPI to use
* Can uyse auth key chains, but they do not provide encryption
* Same SPI, AH/ESP mode, algorithm and keys to match

## AF Support

* AF Support possible
* 0 base v6 unicast, 32 base v6 multicast, 64 v4 unicast, 96 v4 multicast, up from each is local policy
* Uses base ID if not config'd
* SPF and calc independent of one another
* If packet seen in AF instance with no AF bit, other side doesn't support it, so never establishes adj
* Encap is v6, must have it to transport
* VLs only for v6 unicast AFs

### Prefix suppression

* Omits transit prefixes in type 8s and 9s
* Can be per AF or global

### Graceful Shut

* Hellos with router priority of 0
* Stops accepting hellos
* Flush all originated LSAs except type 1s
* Flush type 1s with too high cost
* After dead interval and neighbours dead, flush own type 1

## Fast Hellos

* Set with `ip ospf dead-interval minimal hello-multiplier N`
* Sets hellos to under a second
* Can send btetween 3 and 20 every second (i.e. every 50ms)
* Hellos adv as 0
* Dead interval must be consistent on segment

## Flooding Reduction

* ELiminates need for LSA refresh
* Does not suppress hellos
* All routers still need to support demand circuits
* `ip ospf flood-reduction`

## MAX LSA

* Limit non-self genned LSAs with max-lsa NUMBER

## Redistribution

* redistribute maximum-prefix NUMBER PERCENT-THRESHOPLD [warning-only]

## LSA Pacing

* Instead of refreshing moment half life age, awaits pacing interval to group several LSas
* Usually short than 30m, default 240s
* timers pacing flood lsa-group retransmission
* Retrans - every time router needs to retransmis unack'd LSA, waits to group
* Flood - Similar except controls interface LSA flood list - pacing on an int, grouping what could go on int, default 33ms, retrans 66ms

# Processes

## Neighbourship

* Send hello, init state, seen null, RID 1.1.1.1
 * Other router goes init
* Other router 2way, sends hello, seen 1.1.1.1, RiD 2.2.2.2
* First router 2 way, seen 1.1.1.1, 2.2.2.2, RID 1.1.1.1 hello sent
* DR Election if needed
* R1 to R2, R1 exstart, DD sent
* R2 to R1 - R2 exstart, DD sent
* R1 and R2 exchange with DD
* R1 and R2 loading with LS messages
* Both go full

### States

* Down - Initial, neighbour torn down, implies router knows about other
* Attempted - NBMA or P2MP-NB, back to down in dead time
* Init - Hello seen, no own rid in seen
* 2 way - Hello seen, own RID seen
* Exstart - Master/Slave, Empty DBDs compare RIDs, agree on start seq
* Exchange - DBDs
* Loading LSAs
* Full

# Config

```
int Fa0/0
 ip ospf dead-interval minimal hello-multiplier 4 <--- Minimal makes dead timer 1s, must set hello-multiplier (Fast Hellos)
 ip ospf priority 254 <--- For DR/BDR election
 ip ospf cost BLA
 ip ospf 1 area 0.0.0.0 secondaries none

router ospf 1
  log-adjacency-changes detail
  auto-cost reference-bandwidth MBPS UNITs
  neighbor X.X.X.X cost VALUE - per neighbour cost
  distribute-list prefix-list-1 in Fa0/0
  distribute-list route-map rm-1 in

route-map rm-1 deny 10
  match ip address 48
  match ip route-source 51 # SPECIFY Route source, eg permit 2.2.2.2
```

## AUTH

```
int Fa0/0
 ip ospf authentication null

int Fa0/1
 ip ospf authentication
 ip ospf authentication-key DAVE

int Fa0/2
 ip ospf authentication message-digest
 ip ospf message-digest-key NUMBER md5 VALUE

router ospf 1
 area 0 authentication
 area 1 authentication message-digest
 area 2 virtual-link X.X.X.X authentication [{null | message-digest}]
```

## Crypto Auth

```
key chain ospf 
 key 1 
  cryptographic-algoritihm hmac-sha-1
  send-lifetime 10
  accept-lifetime 10
  key-string dave

int Gi0/0
 ip ospf authentication key-chain ospf

router ospf 1
 area 0 virtual-link x.x.x.x key-chain NAME
```
## Demand circuit

```
int Se0
 ip ospf demand circuit
```

# Verificaiton

