# Introduction to Dynamic Routing

**Distance Vector**

* Distance = metric/feasibility
* Vector = unidimensional array
* Messages contain arrays of network, and distance to it
* Cisco only advertise route if placed into routes routing table
* No topology exchange info
* Path vector for BGP (ASNs)

**Link State**

* Individual objects ad their interconnection
* Routrs, MA networks, border touers, ASBRs
* IP prefixes treate as attributes
* Routers and links what matter
* No modification to messages
* Route summarization difficult

# RIPv2 Basics

* Classless
* DV
* Timer-driven
* UDP port 520
* Hop count maxed at 15
* 16 hops infinite
* No hellos, only routing updates
* 224.0.0.9
* 30s updates
* Full pdates each interval
* On demand circuits can send full update once, then silent until changes (RFC 2091)
* Triggered updates
* Simple or MD5 auth
* Route tags allowed
* Next hop can be rewritten
* Updates sent on each interface every update timer
* Advetises connected and routes in routing table
* No neighbourship
* Use as broadcast with **ip rip v2-broadcast** per interface (rare)

Two message types, request and response. Message format same: -

* 48-bit
* Command field - 1 req, 2 resp
* Version field - 2 for v2
* Octet 3 and 4 must be zero
* Address Family ID and Route Tags
* IP, Subnet, Next Hop, Metric
* 25 routes in single message

**Request**
* Requests full or partial update rather than waiting
* Full update req'd by setting a route with AF ID of 0 and metric 16
* Otherwise, update on networks sent
* Sent on RIP startup for fulls
* Also for RIP int up and **clear ip route**
* No partiuals on cisco
* Metric is cumilative, update on send of update, not receipt
* 4 ECMP routes installed by default, can be between 1 and 16, or 32 (platform dependent)
* **maximum-paths**

## Convergence And Loop Prevention

**Counting To Infinity**
* If next hop with suddenly increase metric, accept and update our metric
* If reaches infinity, stop using next hop

**Split Horizon**
* No sending on received/outgoing interface for route
* With Poisoned Reverse - If outgoing int matches interface in update, advertised with infinite metric

**Route Poisoning**
* Send infinite metric on route failure

**Triggered Update**
* Send moment change occurs
* Only changed network sent

**Update Timer**
* Default 30s

**Invalid After Timer**
* Per route timer
* Default 180s
* Reset after update
* If updates cease, timer expires, route invalid
* Holddown then starts

**Holddown**
* Default 180s
* Being after invalid timer hit
* Routing entry stays in routing table until holddown expires

**Flushed After**
* Default 240s
* Reset after update about network from next hop
* If update about route from next hop cease
* Flushed after time reaches limit, router removes route

Networks chosen by least total metric. All others with higher metric ignored. One time this doesnt take place is COunting To Infinity

Split Horizon enabled by default except FR and ATM, verify with **show ip interface**

Route poisoning means backup path can be found quickly. While route removed, still appears in internal database, marked as possibly down. Allows routes to be repeatedly advertised as unreachable.

Triggered Updates sent moment change detected. Also known as Flash Updates.

Tiiggered Extensions ar On demand circuit behaviour.

Holddown useful when making sure updates received can be believed (others may not know its down yet). Can happen in following: -
* Updates lost in transit
* Next hop router crashed
* next hop router sees router as next hop, split horizon into effect
* RIP removed from next hop
* Summarization/filtering/passive int
* Next hop doesnt support Poisoning (i.e. route will just stop being advertised)


Lack of info about network tolerable for limited time (UDP). If it exceeds a time, "something happened", hence updates not accepted during this.

After route invalid: -

* Router declares network invalud (seen as is possibly down in **show ip route**)
* Holddown started, infinite metric announced for route
* No updates accepted until Holddown expires
* After holddown, entry unlocked, convergence

Cisco version of flushed after verified only after route moved to invalid stat. At that point, route could need to be flushed instantly.

### Converged SSO

* Show ip protocols - Shows RIP settings, timers, version, neighbours and route sources
* show ip route shows route age

### Convergence Extras

* Tune timers
* clear ip route rmoeves all routes in table along with timers

## RIPv2 Config

```
router rip
 version 2
 network 172.31.0.0
```

* Can use classful beaviour
* Disable per interface with following
* Sending updates - **passive-interface**
* Receiving - Filter incoming routes with distribute list, updates with ACL
* Advertising connected subnet - Filter outbout advertisements and interfaces as above
* Limit using neighbor command (unicast updates)
* Autosummarizaiton is default, **no auto-summary**

**Authentication**
* Multiple keys allowed in key chain
* Per interface
* **ip rip authentication key-chain name**, lowest seq number used
* **ip rip authentication mode { text | md5 }
* 24 routes per update now, 20 bytes of auth data in first route

### Next Hop and Split Horizon

* **ip split-horizon** - Not on FR and ATM by default

* Next-hop feature means v2 router can advertise different next hop
* RFC2453
* Not available on Ciscos
* Used for nonzero address when on NBMA (spoke-to-spoke routing)

### Offset lits

* Adds route metric, before or after sending update
* ACL matches route, then add offset
* Specify with in or out


### Route filtering

* **distribute-list** referencing ACL or prefix
* In or out, interface keyword available
* If interface ommitted, applied to all updates

# RIPng for IPv6

* Few changes made
* UDP 521
* FF02::9
* Metric handling increments on receipt
* Route entries based on v6 MTU
* No multiprotocol capability (AF ID ommitted)
* Next hop field specified by separate route entry containing v6 next hop (Link Local) in v6 prefix field with metric of 255, route tag and prefix length 0
* All following updates for that next hop
* Route entry of :: reverts send of message to next hop
* Auth by IPsec

On Ciscos: -
* Auth or encryption not supported
* Split horizon per process only
* Passive ints not supported
* no static neighbours

Following improvements
* Multiple processes can be run (4 seems to be limit), distinguished by name (locally significant)
* Route poisoning activated per process
* Ints can have metric-offset value, applied to all updates (i.e. RIP now has link costs)
* Def route originated per interface, can surpress updates with it

```
ipv6 unicast-routing
ipv6 cef

int Fa0/0
 ipv6 address 2001:DB8:1::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 default-information only

int Se0/0
 ipv6 address 2001:DB8:2::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 metric-offset 3

ipv6 router rip 1
 poison-reverse
```
