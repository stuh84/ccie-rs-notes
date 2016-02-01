# Route Maps, Prefix Lists and AD

## Config of route maps

* If then else logic
* Sequential order
* Use no match command to match all

**Rules**
* Each command same name, all with name in same map
* Permit/deny action
* Delete/insert with seq
* For redist, routes from current routing table
* After route matched, no processing beyond
* If permitted, route redist'd
* When denied, route not redist'd

**When redisting**
* Permit means route redist'd, or leave route in list of routes to be examined in next clause
* Deny filters route or leaves in list for next clause
* If using an ACL, acl deny just means not matched
* Implicit deny at end

## Match commands for redist

* If more than one applied, has to match all
* match int - outgoing int of route
* match ip address - Route and length (acl or pfx list)
* match ip next hop - Use ACL
* match ip route-source - Match advertising routers IP, acl
* match metric - Exact, or range (plus/minus for deviation)
* match route-type - internal, external, E1/N1, E2/N2, l1, l2
* match tag

## Set commands

* set level
* set metric *value* - OSPF, RIP and IS-IS
* set metric *bandwidth delay reliability loading mtu* - IGRP/EIGRP
* set metric type - internal, external, type1 or 2, for ISIS or OSPF
* set tag

## Prefix Lists

* Redistribute must use route map with prefix list inside
* Seq numbers
* Logic is route's prefix must be within range of addresses implied by commands
* Prefix length must match range of prefixes

## Administrative Distance

* EIGRP summary is 5

**Change AD**
* **distance** distance - RIP
* **distance eigrp** internal-dist external-dist
* **distance ospf {[intra-area dist1] [inter-area dist2] [external dist3]}**

# Route Redistribution

## Mechanics of Redistribute Command

Full syntax is

```
redistribute protocol [process-id] [level-1 | level-1-2 | level-2] [as-number] [ metric value] [metric-type type-value] [match { internal | external 1 | external 2} ] [ tag value] [ route-map map-tag] [subnets]
```

## With default settings

* Subnets allows subnets into OSPF
* OSPF default cost 20 when from IGP, 1 from BGP
* Only redists from current routing table

Following logic from a particular IGP
* Take all routes in routing table learned by routing protocol from which routes redist'd from
* Take all connected subnets matched by network commands

* Subnets there, otherwise only classful taken in
* Auto-summary for each network shows just classful networks

### Setting Metrics, Types and Tags

1. Route map (per route)
2. Metric option on command (all redist from protocol)
3. Default metric command (all redist routes not matched by above)

* RIP - no external routes concept, no default metric
* EIGRP - no default metric, external type
* OSPF - 20/1 (IGP/BGP), default as E2, can be E1/E2
* ISIS - 0, default type of l1, Can be L1, L2, L1/L2 or Ext

### Mutual Redist at Multiple Routers

* Routers use AD for best route, but could be suboptimal
* RIP route could be shortest path, but OSPF used
* Need to make routers aware where routes came from 
* Solve by filtering or changing AD

### Preventing suboptimal routes with AD

* Higher AD for redist routes
* eg of **distance ospf external 180**
* Can apply per route with **distance { distance-value ip-address {wildcard-mask} [ip-standard-acl] [ip-extended-list]**
* IP address is advertising router
 * For RIP, EIGRP and IS-IS, advertising router's neighbour interface address
 * For OSPF, matches RID creating LSA

### Preventing suboptimal with Route Tags

* Distribute list can match tags, eg

```
distribute-list route-map check-tag-9999 in
redistribute ospf 1 route-map tag-ospf-9999 in
```

### Metrics and Types for Redist routes

Metric preferences: -
* RIP - None
* EIGRP - Internal > External
* OSPF - Intra > Inter, E1, E2, tiebreaked E2 with cost to ASBR
* IS-IS - L1, L2, External

# Route Summarization

Default workings: -
* Same metric as lowest-metric component
* No component subnet advertising
* Not advertised if no components
* LOcal summary to null0 installed
* Reduce routing tables and DB size
* Decrease specific info

## EIGRP

* **ip summary-address eigrp** asn network-address mask [admin-distance] - interface
* Distance not advertised, used on local router, determines if null placed in routing table

## OSPF

* Between areas only (no LSDB differences)
* ABR or ASBR
* ASBR - **summary-address** {{ip-address mask} {prefix-mask}} [not-advertise] [tag value]
* ABR - **area 1 range ip-address mask [advertise | not-advertise] [cost value]**
* For ABR, this is area where component subnets are

## Default Routes

* With IP classless, routes normal
* No IP classless, router checks if part of destinations classful network in table, avoids default

**Five basic methods**
* Redistribution
* Static route to 0.0.0.0 with redistribute static - EIGRP, RIP
* **default-information-originate** - RIP, OSPF
* **ip default-network** - RIP, EIGRP
* Summary routes - EIGRP

### Static routes with redist

* RIP and EIGRP
* Both commands needs to be on same router
* Metric must be default or set
* Redist can have route map
* EIGRP treats default as external by default

### default-information-originate

* OSPF
* Any defaults in routing table
* Can set metric and type directly, defaults of cost 1, E2
* Always keyword means advertised regardless of routing table
* Supported in RIP but differences
 * Creates and advertises if either no default exists, or default from another protocol
 * If a static route, no injection as done with redistribution

### ip default-network

* RIP and EIGRP
* Local router must match a classful network
* Classful network must be in routing table (by any means)
* For EIGRP, must be advertised by local router into EIGRP
* For EIGRP< flagged as candidate
* RIP no flag, does not need to advertise it to use it

### Route Summarization

* EIGRP only
* Creates a null route on local, should be avoided
* Summary to others as AD 90 (standard EIGRP)
* Should set higher distance to not blackhole traffic

# Performance Routing (PfR)

* Routing based on load/bandwidth
* OER original, prefix based optimizations
 * Not application specific

## Operation Phases

* Runs after OER configured
* Profile Phase - learns flows with high latency, traffic profiled/learned is traffic class, list of classes is MTC (monitored traffic classes) list
* Measure Phase - Collect/compute performance
* Apply Policy - Apply low and high thresholds
* COntrol phase - Influence traffic with routing manip/PFR
* Verify phase - OER verifies OOP event performance, makes adjustments to bring back in policy

## Concepts

Interfaces: -
* Internal - Connects to internal network, comms with device in infrastructure designated as control plane manager for PfR (Master Controller)
* External - Transmitting packets out of network, must be at least two
* Local - For forming control plane, defines source to communicate to master controller

## Auth

* Mandated, not optional
* Key-chain auth
* Global config

## Operational Roles

### Master Controller

* Maintains comms and auths border routers
* Monitors flows
* Applies policies for prefixes and exit links
* MC not always in forwarding path
* Must be reachable by BRs
* Single MC supports 10 BRs or 20 exit ints
* Can run MC and BR on same device
 * Still needs auth
 * Standalone preferred

### Border Router

* Router with one or more exit links
* All policy descisions/routing changes enforced
* Reports prefix and transit link measurements to MC
* MC inejects preferred route to alter flow

## Basic Config

### Config of MC

**Auth**
```
key chain PFR_AUTH
 key 1
  key-string DAVEPERFORMS
```

**Enable process**
```
pfr master
```

**Designate internal/external ints**
```
pfr master
 border 2.2.2.2 key-chain PFR_AUTH
  interface Se0/0.21 internal
  interface Fa0/0 external
 border 3.3.3.3 key-chain PFR_AUTH
  interface Se0/0.31 interval
  interface Fa0/0 external
```

show oer master border

### BR config

**Auth**
```
key chain PFR_AUTH
 key 1
  key-string DAVEPERFORMS
```

**Process**
```
pfr border
 master 4.4.4.4 key-chain PFR_AUTH
```

**Specify local interface**
```
pfr border
 local loopback 0
```

* Can use logging, and change port
 * **logging** under pfr border
 * **port 3950** under pfr border

* Seen as MC Active on MC when both BRs up

# Troubleshooting complex L3 issues

## Troubleshooting process

**Problems at other layers**
* MTU mismatch
* Unidirectional link
* Duplex
* Error rate
* L2 config
* ACL
* Security policy
* TTL too low
* Two or more l3 subnets in same VLAN

**CHeck fields in IP header**
* Mismatch subnet masks
* Too short TTL for adj (eBGP multihop)
* MTU dropping large packets
* Multicast not supported/dsiabled/rate limited
* Overloaded link
* QoS config 

**Routing Problems**
* Incorrect split horizon - some routes adv, others not
* Incorrect redistribution - filttered, or routing loops
* Protocols not adv routes when they should
* Protocols no redisting routes when they should
* Incorrect route filtering (masks)
* EIGRP SIA
* Incorrect summarizatoin
* AD manip superceeding correct routing rules
* Metric calc different on different routers (eg auto cost, EIGRP k)
* Metric manipulation
* NAT
* PBR
* Interface dampening
* Mismatched timers dropping adj

## Commands

* show ip protocols
* show interfaces
* show ip interfaces
* show ip nat trans
* show ip access-list
* show ip int brief
* show dampening
* show logging
* show policy-map
* traceroute
* ping and extended ping
* show route-map
* show standby
* show vrrp
* show track
* show ip route
