* AS Path is vector
* Shortest is best route

# Building BGP Neighbour Relationships

* TCP
* BGP Updates end goal
* Explicit neighbour config
* TCP 179
* If no config for neighbour, request rejected
* After TCP connection, BGP Open
* After opens, established
 * Updates then exchanged

* **bgp timers keepalive holdtime** - 60 and 180 default
* **neighbor timers** - per neighbour
* **bgp router-id**, then highest loopback, then highest IP
* Update source, or outgoing int IP
* Auto summ off by default
* Auth is MD5, **neighbor password**

## Internal Neighbours

```
router bgp 123
 no sync
 bgp router-id 111.111.111.111
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 123
 neighbor 2.2.2.2 updated-source loopback1
 neighbor 2.2.2.2 password DAVE-LIKES-BGP
 no auto-summary
```

```
 router bgp 123
 no sync
 bgp log-neighbor-changes
 neighbor my-as peer-grou
 neighbor my-as update-source Lo1
 neighbor my-as remote-as 123
 neighbor 1.1.1.1 peer-group my-as
 neighbor 2.2.2.2 peer-group my-as
 no auto-summ
```

* One set of updates per peer group

## External Neighbours

* Single link usually
* No update source
 * If they are, **ebgp-multihop**
 * Loopbacks are hop inside router (ttl of 1)

## Checks before becoming neighbours

1. Must have source of packet in neighbor config to accept packet
2. ASN must match config (unless confed)
3. RIDs must not be same
4. MD5 must pass

## Messages and Neighbour States

* Idle - Not listening for TCP
* Connect - Listens for TCP
* Active - Listens for TCP, initiates TCP
* Open Sent - Listens for TCP, Initiates TCP, TCP is up, Open Sent
* Open Confirm - Listens for TCP, Initiates TCP, TCP is up, Open Sent, Open Received
* Established - Listens for TCP, Initiates TCP, TCP is up, Open Sent, Open Received, Neighbor Up

## Message Types

* Open - Establishes neighbours and basic parameters
* Keepalive - Maintains neighbours
* Update - Routing info
* Notification - takes down neighbour on error

## Purposefully resetting BGP peers

* **neighbor shutdown** - In config
* **clear ip bgp** - Resets neighbour, closes TCP, entires in table removed for neighbour, rediscovers after

# Building BGP Table

* Topology table (RIB) holds learned NLRI and associated PAs
 * NLRI - IP PRefix and length
* Does not advertise routes, advertises PAs and NLRI sharing same PAs

## INjecting Routes/Prefixes into BGP table

### Network command

* Assumption of no auto-summary (default from 12.3 mainline)
* Looks in current routing table for matching networks
* If exists, puts NLRI into local BGP table
* Connected, static or IGP match
* When route withdraw, NLRI withdrawn and neighbours notified with Withdraw
* **network { network-number [ mask net-mask] } [ route-map name ]**
 * No mask - classful
 * Match with no auto-summ - Must match prefix and length
 * Match with auto-summ - If network lasts classful, matches any subnets of classful network exists
 * NEXT_HOP taken from next hop of IP route
 * Maximum number injected by one process - Limited by NVRAM and RAM

### Redistributing from IGP, Static or Connected Route

* No metric calc
* Examines various PAs for est route
* Apply route-map to manipulte PAs
* If metric is assigned, assigns to MED
* Takes in IGP routes and connected routes matched by IGP network commands
* next hop either next of redist, 0.0.0.0 fo connected and routes to null0

### Impact of AutoSumm on Redist and Network command

* Causes classful summary to be created if components exist
* Causes BGP to summarize only routes injected because of redist on router
* Does not look for classful network boundaries in topology
* Does not look at BGP table routes

**Redistribute**
* If subnets of classful would be redistributed, do not, redist a route for classful instead
* No subnets in redistribution

**Network**
* If network mentions classful, and any subnets of classful exist, inject route for classful network
* Still injects subnets as well as classful if command matches classful and components exist

### Manual Summaries and AS_PATH PA

* **aggregate-address** - Any in BGP table
* Does not always suppress advertisement
* Must carry AS_PATH PA
 * AS_SEQ
 * AS_SET
 * AS_CONFED_SEQ
 * AS_CONFED_SET

* Aggregate creates summary with AS_SEQ null
* If components have diff AS_SEQ values, AS_SEQ not aggregate
 * Uses AS_SET with unordered ASN lists instead

* **aggregate-address** prefix mask - Advertises out aggregate
* **aggregate-address** prefix mask **summary-only** - Only summary
* **aggregate-address** prefix mask **summary-only as-set** - Advertises aggregate only with all ASNs in components as AS-SET

Actions by aggregate command when summary route created: -
* No summary if no components in BGP
* If component subnets withdrawn, aggregate withdrawn
* NEXT_HOP of 0.0.0.0 in local BGP table
* NEXT_HOP is update source when advertised
* AS_SEQ same if all summary routes have same AS_SEQ
* Null if differ
* as-set creates AS_SET segment, but only if null
* If to eBGP peer, ASN prepended to AS_SEQ
* Summary only suppresses components, suppress-map allows specific components

* If static to null0 created and redist'd, effectively a summary. Does nt filter components

### Default Routes

* Network command
* Redistribute in
* **neighbor x.x.x.x default-originate [route-map name]**

**Network**
* 0.0.0.0/0 must be in IP routing table, any means
* Withdrawn if it disappears

**Redistribution**
* Requires **default-information originate**

**Neighbor**
* Advertises default to neighbor
* Can match on IPs in table (conditional advertisement) with route-map

## Origin PA

* IGP, EGP or Incomplete
* Seen in **show ip bgp**
* IGP - I - network, aggregate (in some cases) and neighbor default-originate
* EGP - e - Shouldn't be seen
* Incomplete - ? - redist, agg in some cases, default-information originate

**Aggs**
* If as-set option not used, origin i
* If as-set used, and all components code i, code i
* If as-set used and at least one component of ?, code ?

# Advertising BGP routes to Neighbours

## Update Message

**General Format**
* Length (bytes) of Withdrawn Routes (2 bytes)
* Withdrawn Routes (Variable)
* Length (bytes) of PA section (2 bytes)
* PA (variable)
* Prefix Length and Prefix Variable (2 bytes)
* Prefix Length and Prefix Variable (2 bytes)
* Prefix Length and Prefix Variable (2 bytes)
* Prefix Length and Prefix Variable (2 bytes)
* Prefix Length and Prefix Variable (2 bytes)
* ...

* Withdrawn routes for informing neighbours
* PA lists PA for every route (NH, AS_PATH etc)
* Prefix and Length fields define each NLRI
* Update Set of PAs, then send NLRI that matches PA
* Separate updates needed if each NLRI has different PA

### Determining Content of Updates

Rules for not including routes in updates: -
* Both iBGP and eBGP - Routes not best
* Both - Routes matched by deny in filtering
* iBGP - iBGP learned routes (unless RR or Confed)

Best route with any routing policies usually goes: -
1. Shortest AS_PATH
2. eBGP over iBGP
3. Lowest IGP metric to next hop
4. Lowest BGP RID for iBGP routes

* Routes must be locally injected or reachable recursively (i.e. next hop is valid)
* **next-hop-self** for iBGP usually
* **next-hop-unchanged** for eBGP usually

* Single RIB usually, notations to which entries were sent and received from each neighbour
* **show ip bgp neighbor advertised-routes**
* **show ip bgp neighbor received-routes** - Must have **soft-reconfiguration inbound** on neighbor config
* " valid" means route is candidate for use

# Building IP Routing Table

## Adding eBGP routes to IP Routing Table

* Must be best route
* AD must be lower (if learned by multiple protocols)
* Default AD is 20 (only EIRGP summaries lower as far as dynamic protocols go)
* 20, 200, 200 (eBGP, iBGP, Locally injected) ADs
* Change for all routes with **distance bgp external internal local**
* Change per route with **distance value [ip-address {wildcard-mask}} [ ip-standard-list] [ip-ext-list]**
 * IP and mask refer to neighbour, not rid or next hop, ACL mathces routes
* NEXT_HOP in IP routing table, even if not on local router (recursive lookups)

## Backdoor routes

* Used if learning routes via private link plus eBGP
* **network backdoor** command
 * Makes BGP route a "local" route (AD 200)
 * Does not advertise route with BGP downstream

## Adding iBGP routes to IP Routing Table

* BGP Sync as well as other two requirements
* With sync disabled, same as eBGP (AD and best route)

### Sync and Redistribution

* iBGP route in table can't be best unless exact prefix learned through IGP and in current routing table
* Static routes do not apply to above
* When sync used and OSPF is IGP, if OSPF RID different from BGP RID of advertising route, sync does not allow route

### Disabling sync and using BGP on all routers in AS

* Once all routers speak iBGP, sync not needed
* Full mesh requirement (or use confeds and RRs)
 * Avoids routing loops

### Confederations

* RFC 5065
* Routers go into sub-ASs
* Routers into same sub-AS are confed iBGP peers
* As above, different as, confed eBGP

* In single sub-AS, iBGP peers must be fully meshed, works like iBGP
* eBGP peers act like normal eBGP peers (can advertise to other confed sub-ASs)

* COnfed AS_PATH used inside as for loop prevention
* Confed adds sub-AS into AS_CONFED_SEQ, and AS_CONFED_SET (for aggs)
 * Usual AS PAth rules (avoid same sub-AS)

* 64512 to 65535 - Private ASNs

**Key Topics**
* Full mesh inside sub AS
* Normal eBGP for Confed eBGP
 * TTL normal
 * iBGP in other regards (eg NEXT_HOP)
 * Confed ASNs not used for best AS_PATH selection
 * Confed removes confed ASNs outside confederation

**Config**

* router bgp ASN is now of the sub-as
* Downtime required if converting

```
router bgp sub-as
 bgp confederation identifier as - true ASN
 bgp confederation peers sub-as - Identifies neighbouring AS as another sub-as
```

### Route Reflectors

* Servers and clients
* Servers allow iBGP routes from clients and advertise to other iBGP peers

**Core Logic**
* ONly RR uses modified rules, clients and non-clients unaware
* Prefix from client - to client and non-clients
* Prefix from non-client - to client, not to non-client
* From eBGP - to client and non-client

* Clusters of one or more RRs
* Multiple clusters make sense only when physical redundancy exists
 * Must have one peer from an RR cluster to another RR cluster
* All RRs peer directly, full mesh of iBGP RR peers
* Non clients need to fully mesh with all RRs in all clusters

**Avoid Routing Loops with**
* CLUSTER_LIST - RRs add their cluster ID into CLUSTER_LIST PA, if own ID in CLUSTER_LIST, drop update
* ORIGINATOR_ID - RID of first iBGP to advertise route, if own RID seen, drop it
* Only advertise best routes - Routes reflected only if considered best

* **neighbor x.x.x.x route-reflector-client**
* **bgp cluster-id value**

# MPBGP

* VPN routes from PE to CE
* Can be learned through BGP-4, EIGRP, OSPF, static routes etc
* Each MP-BGP session is internal session
* MP-iBGP required in MPLS VPN
* With L3 VPNs, VPNv4, label info and standard communities transmitted
* When Open sent, capabilities field lists what peer understands

**TWo nontransitive attributes**
* Multiprotocol Reachable NLRI (MP_REACH_NLRI) - new MP routes
 * COmmunicates set of reachable prefixes with next hop info
* Multiprotocol Unreachable NLRI (MP_UNREACH_NLRI) - Revokes routes previously announced
 * unreachable destinations

* When PE sends MP-BGP update to other PE routers, MP_REACH_NLRI contains
 * AF INformation - Network Layer Protocol being carried
 * Next-hop info - Next hop address of next router in path to destination
 * NLRI - Manages addition/withdrawal of MP routes and next hop, NLRI prefixes must be in same AF

## Config

* Disable IPv4 unicast wiht **no bgp default ipv4-unicast**
* Default context (i.e no address family) is catch all for non-vrf or v4 specific sessions
 * Anything into global table

**Standard Config**
```
router bgp 1
 neighbor 194.22.15.3 remote-as 1
 neighbor 194.22.15.3 update-source lo0
 neighbor 194.22.15.3 activate
```

**AF Config**
```
router bgp 1
 address-family vpnv4
  neighbor 194.22.15.3 activate
```

* **neighbor x.x.x.x send-community extended/standard/both** - Extended only by default

