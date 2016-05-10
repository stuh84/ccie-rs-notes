# Messages

* Open - Establishes neighbours and parameters
* Keepalive - Maintains neighbours
* Update - info
* Notification - Error, takes neighbour down

## Update

|Description|Size|
|-----------|----|
|Length in bytes of Withdrawn routes|2 bytes|
|Withdrawn routes|Variable|
|Length in bytes of PA section|2 bytes|
|PA|Variable|
|Prefix Length and Prefix Variable|2 bytes|
|Above as many times as NLRI||

* Withdraws inform neighbours
* Separate updates if each NLRI with different PA

# Timers

* Hold Time - 180 by default
* Keepalive - 60 by default
* `bgp timers keepalive holdtime` - Global
* `neighbor timers` - Per neighbour
* Fast convergence - Update peers every 5 seconds for iBGP, 30 for eBGP, also relies on IGP

# Trivia

* Auto summ off by default
* One set of updates per peer group
* Loopbacks hop inside router, therefore TTL higher (`ebgp-multihop`)
* No routes advertised, PAs with NLRI that share PA sent
* Network command - Origin of I, no mask = classful
* Redistribution means ? origin, any metric assigned to MED, matches any connected routes IGP matches with network command

## Summaries

* AS_PATH PA
 * AS_SEQ - Sequence of ASs if ASs of agg'd routes same, null if differ
 * AS_SET - Unordered list of ASs - `aggregate-address 192.168.0.0 255.255.255.0 summary-only as-set`, only creates if AS_SEQ null
 * AS_CONFED_SEQ - As above but for confeds
 * AS_CONFED_SET - As above but for confeds
* Next hop of 0.0.0.0 in BGP table
 * Update source when advertised
* To eBGP peer, ASN prepended to AS_SEQ
* Suppress Map used to advertise component routes
 * Route-map, any subnets with route-map permit suppressed
 * Suppressed in advertisement only

## Defaults

* Network command - 0.0.0.0/0 must be in routing table
* Redistribution - Requires `default-information originate`
* Default originate - To neighbour, can be conditional, `neighbor X.X.X.X default-originate [route-map name]` 

## Routes

* Don't include if routes aren't best, denied in filtering, and for iBGP, any IBGP learned routes (unless RR or Confed)
* Without any routing policy usually goes Shortest AS_PATH, eBGP over iBGP, lowest IGP metric, lowest RID for iBGP
* Next hop must be reachable
 * next-hop-self
 * next-hop-unchanged
* Valid route within show ip bgp neighbor outputs means candidate for use

## eBGP routes to IP routing table

* Must be best
* AD
 * `distance bgp *external internal local*`
 * `distance VALUE 192.168.0.0 0.0.255.255 [ip-standard-list] [ext-acl]`
  * IP and Mask refer to neighbour, not next hop
* Next Hop must be in routing table

### Back Door

* Makes BGP local route (AD 200)
* Not advertised by BGP downstream

## iBGP routes to IP Routing Table

* BGP Sync as well as other requirements (sync disabled means same as eBGP)
* Static routes dont work with sync
* When sync used with OSPF, OSPF RID must match BGP RID of advertising route

## Confeds

* RFC 5065
* Confed AS path for loop prevention
* Confed eBGP - TTL normal, NEXT_HOP retained
* Confed ASs not used outside Confed (i.e. AS Path length)
 * Removed on exit

## RRs

* Clients and non clients unaware of RR

**Rules for advertisement**

|Route from|To client|To Nonclient|
|----------|---------|------------|
|Client|Yes|Yes|
|Non-client|Yes|No|
|eBGP|Yes|Yes|

* Multiple clusters make sense only when physical redundancy
 * Must have one peer from RR cluster to another RR cluster
* All RRs peer directly (mesh*
* Non-clients fully mesh with all RRs in all clusters

### Loop avoidance

* CLUSTER_LIST - If own CLUSTER_LIST on RR - drop update
* ORIGINATOR_ID - If own RID seen (by client/non-client), drop it
* Only advertises best routes

## MP-BGP

* With L3VPNs, VPNv4, Label info and standard comms transmitted
* Capabilites in Open message

### Two nontransitive attributes

* MP_REACH_NLRI
 * Reachable prefixes with next hop info
 * Contains AFI
 * Contains Next hop info - next router in path to destination
 * NLRI - must be in same AF
* MP_UNREACH_NLRI
 * Revokes routes

* `no bgp default ipv4-unicast`
* Activate peers in AFI
* Extended Communities by default only

## Neighbor Disable-connected

* Allows peering of eBGP neighbour with a loopback without changing TTL
* Effectively avoids rewriting IP TTL
* Does mean neighbour must be 1 hop away

## Multisession Transport per AF

* Separate TCP session per AF
* Multiple topologies
* `neighbor X.X.X.X transport multisession`
* Uses scope hierarchy

## Third Party next Hop
* Sets it correctly in local NGP table
* eBGP changes it
* If all peers shae same segment, next hop unchanged

## BGP Link Bandwidth

* Adv bw of AS exit link as ext-comm
* Propagated to iBGP peers
* Used for BGP multipath and unequal BW lb
* CEF required
* Only under v4 and vpnv4
* Only for DC'd eBGP neighbours
* Enabled with `bgp dmz link-bw`
* Two paths seen as same for LB if weight, LP, AS PATH, med and IGP costs same
* Will be seen in show ip bgp PREFIX

## BGP over GRE

* Need to change next hop if iBGP other network never knows to use it
* GRE TTL decrements

# Processes

## Neighbour establishment

* TCP 179
* No neighbour config = request rejected

### States

* Idle - Refused incoming connections
* Connect - Waits for TCP to complete --> Goes to OpenSent if successful, after sending OPEN message
 * Failure goes to Active, Connect or Idle
* Active - Attempting TCP connection
 * Failure goes Active or Idle
* OpenSent - OPEN Message sent, waits for Open reply. If OPEN received, OpenConfirm and KeepAlives sent
* OpenConfirm - Waits for Keepalives, Established if received, Idle if not
* Established

### Prechecks

* Source of packet in neighbour config
* ASN must match (unless confed)
* RID not same
* MD5 must pass

# Attributes

## Origin

* IGP, EGP, Incomplete
 * I - Network, default-originate, aggregate (if as-set not used, or if all components of as-set i)
 * ? - Redist, agg (at least one component ?), default-information originate

# Filtering

* Peer groups process update once, not per peer
* Cannot apply to single neighbour in group
* Matching logic on BGP update
* Distribute list
 * Standard ACL - Matches prefix and wildcard mask
 * Extended ACL - Prefix, length and wildcard mask
* Prefix list - Exact or first N of prefix, and range of lengths
* Filter list - AS paths
* Route MAP - Prefix, length, AS-PATH, PAs
 * Deny filters route, deny in ACL doesn't match route

## AS PATH filtering

* AS_SEQ most common, first ASN most recent
* AS_SET comma delimited, enclosed with `{}`
* AS_CONFED_SEQ space, enclosed with `()`
* AS_CONFED_SET comma, enclosed with `{}`

## Regex

* Regex of first line applied to each
* Permit or deny
* Implicit deny

```
^ - Start of line
$ - EOL
| - Logical OR
_ - Any delimiter (blank, comma, SOL, EOL)
. - Any character
? - Zero or one
* - Zero or more
+ - One or more
(string) - Single entity
[String] - Wildcard for any single character
{} - Amount of matches
```
* First item matched, rest of path sequentially
* Match AS_CONFED with [(] and [)], brackets are regex characters otherwise

## Decision process

* Well known - Supported by every BGP implementation
* Optional - Opposite of above
* Mandatory - Has to be in update
* Discretionary - Not required in update
* Transitive - Silently forwarded, even if unknown to self
* Nontransitive - Remove and do not propagate
* Well known mandatory - AS-PATH, NEXT_HOP, ORIGIN
* Well known discretionary - ATOMIC_AGGREGATE
* Optional Transitive - Aggregate
* Optional Nontransitive - ORIGINATOR_ID, CLUSTER_LIST

1. Next hop reachable
2. Weight - Highest
3. Local Pref - Highest
4. Locally injected (network, redistr, or summarization)
5. AS Path - Shortest, bgp bestpath as-path ignore
6. Origin - IGP > EGP > ?
7. MED - Smallest
8. Neighbour type - eBGP > iBGP
9. IGP Metric
10. Oldest route
11. Smallest neighbour RID (if bgp bestpath compare-routerid configured)
12. Smallest neighbour ID (two relationships to same router to pass 11)

* Multiple routes into table after 9
* Even if multiple added, best path advertised only

## Policies

* NEXT_HOP - self or unchanged
* Weight - 0 through 65535, default 0 for learned 32768 for local injection
 * Set with Route Map or neighbor weight, route map takes pref
* Local pref - Default 100, `bgp default local-preference VALUE`
* Locally injected routes with Origin 
 * Local injection weight auto used
 * Might happen if local injection plus Weight route map manipulation
 * Same NLRI from different sources
* Shortest AS-PATH - AS-Sets count as 1, confeds don't count
 * remove-private-as - by router attached to private AS
 * local-as no-prepend - Different than in neighbour command
 * bgp bestpath as-path ignore
 * Private ASNs only removed for eBGP updates
 * If AS_SEQ priv and pub, priv not removed
 * If ASN of eBGP peer in current AS_PATH, private ASN not removed
 * Route agg can decrease path length
* Best origin - set origin in route map
* MED - Default 0, sent to one AS, no further
 * bgp bestpath med missing-as-worst
 * Range of 0 through 2^32 -1
 * bgp always-compared-med - puts before AS path
 * bgp deterministic-med - Process per adjacent AS, then picks best
* Maximum Paths - `maximum-paths` command
 * Defaults to 1
* Lowest Route ID - Examines eBGP routes only, picks routes with lowest RID
 * if only iBGP routes, lowet RID
 * If BGP has best route to NLRI, but new info learned from other routes
  * Including BGP route to reach previously known prefix
  * Goes through decision process again
 * If gone through process and gets here again, does not replace exisitng best
 * stops flaps
 * `bgp bestpath compare-routerid` - eBGP routes only

### Max Paths

* eBGP - Tiebreaker, max paths above 1, adjacent ASN must be same, tiebreaker if more candidates than can be used
* iBGP - Tiebreaker, maximum-paths ibgp, only with differing next hops

## Communities

* Optional transitive
* Must include all values in same community list (un ordered)
* Extended uses regex
* 16 lines in standard
* Many in extended
* set community none
* set comm-list LIST delete
* match and match exact
* FFFF:FF01 - NO_EXPORT - Not out of this AS but can be to cofeds
* FFFF:FF02 - Not to any other peer
* FFFF:FF03 - Not out of local confed sub-AS (NO_EXPORT_SUBCONFED)

## Fast convergence

### Fast External Neighbour Loss Detection

* eBGP direct neighbours torn down if connected subnet goes - immediately
* Immediate route flush
* Immediate alternates
* Default from IOS 10.0 and up

### Internal Neighbour Loss Detection

* 12.0 up
* Neighbor fall-over - Keepalive traffic not required to signal pull down quickly
 * Moment IP of peer removed, session goes
 * IGP must find BGP peer immediately, or BGP session disconnetion
 * Holddown/delay in BGP session not used

### eBGP Fast Session Deactivation

* Quick detects failures of eBGP to loopbacks, or when external fallover disabled
* `no bgp fast-external fallover` - Disables, retains quick response to interface failure
* Identical to internal
* Can relect rule that BGP peer has to be correctly connected, route map matching connected subnets


# Config

## Confederations

```
router bgp 65001 <--- SUB-AS
 bgp confederation identifier AS <--- Real AS
 bgp confederation peers SUB-AS <--- What other ASs are in confed
```

## Conditional Advertisement

```
router bgp 65412
 neighbor X.X.X.X advertise-map RM1 non-exist-map RM2

route-map RM1
 match ip address 65

route-map RM2
 match ip address 60
```

* Non exist - says what networks shouldn't exist
* Adv map - If networks match by above do not exist, advertise from advertise map matched networks

## Dynamic neighbours

```
router bgp 65001
 bgp listen range SUBNET peer-group NAME
 neighbor GROUP parameters
```

## Max AS Limit

```
bgp max-as limit NUMBER <--- As's above number, discards, 1-2000
```
