# Route Filtering and Summarization

* Distribution lists, prefix lists, AS_PATH filters, route maps, aggregate address
* All can filter in and out updates
* Peer group processes update once, not per peer
* Cannot apply to single neighbour in peer group
* Matching logic is on BGP update
* Clear required to take effect
* Clear can be soft

* **neighbor distribute-list** - Standard ACL, match prfx and wildcard mask
* **neighbor distribute-list** - Ext ACL, prefix, length and WC mask
* **neighbor prefix-list** - Exact or first N of prefix, plus range of lengths
* **neighbor filter-list** - AS_PATH
* **neighbor route-map** - Prefix, length, AS_PATH, and/or PA

## Filtering on NLRI

* Use Ext ACL
* Most IGPs have standard ACL
* Prefix and length

### Route Map Rules for NLRI filtering

* Update compare to route-map, filtered or not based on clause
* **deny** filters route, deny in ACL/list doesnt match route

### Soft Reconfig

* **clear ip bgp { * | neighbor-address | peer-group} {soft [in | out]}**
* Soft applies policy config in and out, and can be direct
* Soft default for sending updates
* Enable for inbound, **neighbor x.x.x.x soft-reconfiguration inbound**
 * Received updates stored

* Config changes for local injection cant be soft \
 * Just reprocesses updates

## Comparing BGP prefix lists, dist lists and route maps

* Dist lists can match BGP NLRI
* Prefix lists more flexible
* Route maps add nothing for filtering NLRI, but can manipulate
 * Also combine match criteria

## Filtering Subnets of a Summary using agg command

* **summary-only** key word
* **supress-map**
 * Route map, any subnets with route-map permit suppressed
 * Suppressed in advertisement only

## Filtering updates by matching AS PATH PA

1. **ip as-path access-list NUMBER permit/deny REGEX**
2. **neighbor x.x.x.x filter-list AS-PATH-ACL { in | out }**

### AS PATH and segment types

* AS SEQ most common, most recently added first ASN
* AS_SET comma delimited, enclosed with {}
* AS_CONFED_SEQ space delimited, enclosed with ()
* AS_CONFED_SET, comma delimiter, enclosed with {}

### Regex path matching

1. Regex of first line applied to AS_PATH of each route
2. For match NLRIs, action permit or deny
3. For unmatched, step 1 and 2 for lines
4. Any NLRI not matched is filtered

**Regex characters**
* ^ - Start of line
* $ - EOL
* | - Logical OR
* _ - Any delimiter (blank, comma, SOL, EOL)
* . - Any character
* ? - Zero or one
* * - Zero or more
* + - 1 or more
* (string) - Make string single entity
* [string] - Wildcard for any single character

* For regex on BGP route, IOS searchs AS_PATH for first instance of first item in regex, then rest of path sequentially

* **show ip bgp neighbor x.x.x.x advertised-routes** - after filtering
* **show ip bgp neighbor x.x.x.x received-routes** - before filtering

* Match AS_CONFED with [(] and [)], as ( and ) are Regex characters

# PAs and Decision Process

## Generic terms and characteristics

* Well known or optional
* Mandatory or discretionary
 * ATOMIC_AGGREGATE - Well known discretionary
 * AS_PATH - Well known mandatory
* Transitive - Silent forward to other routers, even if unknown to self
* Nontransitive - Remove PA and not propagate

|Name|Description|Type|
|----|-----------|----|
|AS_PATH|Transitted ASNs|Well-known mandatory|
|NEXT_HOP|Next Hop of NLRI|Well-known mandatory|
|AGGREGATOR|RID and ASN of summarizing router|Optional transitive|
|ATOMIC_AGGREGATE|Tags NLRI as being a summary|Well knon discretionary|
|ORIGIN|Where route injected|Well known mandatory|
|ORIGINATOR_ID|RRs for RID of original route|Optional Nontransitive|
|CLUSTER_LIST|Cluster IDs of RRs|Optional nontransitive|

## BGP Decision Process

1. Next hop reachable
2. Highest admin weight, higher better
3. Local Pref - Higher better
4. Locally injected routes (network, redist or summarization)
5. Shortest AS path, ignore with **bgp bestpath as-path ignore**
6. Origin - IGP over EGP over ?
7. Smallest MED
8. Neighbour type - eBGP over iBGP (confed eBGP is still iBGP)\
9. IGP metric to next hop

* Above done before looking at maximum paths

### Final tiebreakers

10. Oldest route
11. Smallest neighbor RID (only if **bgp bestpath compare-routerid** configured)
12. Smallest neighbor ID (means router has two neighbour relationships to same touer)

## Multiple routes to IP routing table

* If best path done in first 9, only one added
* If best path after step 9, considers multiple
* Even if multiple added, best path chosen and only one advertised

# Configuring BGP policies

## NEXT_HOP reachable

* **next-hop-self**
* **next-hop-unchanged**

## Weight

* 0 through 65535
* Dfault 0 for learned, 32768 for local injection
* Route map or **neighbor weight**
 * Route map takes preference

## Highest local pref

* Default 100
* **bgp default local-preference <0-4294967295>**

## Locally injected routes with Origin

* With local injection weight, automatically used
* For routes that might happen, would need local injection, advertisement and a route-map assigning weight
* Or same NLRI from different sources (i.e. network and redistribute connected)

## Shortest AS_PATH

* AS_SET - Counts as single ASN always
* Confeds - Don't count
* Aggregate address - See agg rules
* **neighbor remove-private-as** - By router attached to private AS
* **neighbor local-as no-prepend** - Can use different AS than in neighbour command
* AS_PATH prepending
* **bgp bestpath as-path ignore**

### Private ASNs

In IOS: -
* Private ASNs only removed for eBGP updates
* If current AS_SEQ has priv and public, private ASNs not removed
* If ASN of eBGP peer in current AS_PATH, private ASN not removed

### Prepending and route aggregation

* Can prepend any ASN
* Route agg can decrease path length

## Best Origin PA

* **set origin**

## Smallest MED

* Default 0
* Sent to one AS, no further
* **bgp bestpath med missing-as-worst**
* Range of 0 through 2^32 -1
* **bgp always-compared-med** - MED can discriminate before AS path with this (put on all routers)
* **bgp deterministic-med** - Processes routes per adjacent AS, then picks best from them
 * Without this, just goes sequential

## eBGP over iBGP

* External should always be preferred

## Smallest IGP Metric

* Find shortest path to next hop

## Maximum paths

* Which route is best - tiebreakers
* Whether to add multiple paths - **maximum-paths** command

## Lowest BGP route ID (with one exception)

1. Examine eBGP routes only, picking routes with lowest RID
2. If only iBGP routes, lowest RID

* Exception when BGP has best route to NLRI, but learned new info from other routes
 * Including BGP route to reach previously known prefix
* Router then goes through decision process again

* If gone through process and gets here again, If existing route is eBGP route, do not replacee exisitng best, even if new has smaller RID
* Stops flaps
* **bgp bestpath compare-routerid** - eBGP routes only

## Lowest neighbour ID

* Does consider all routes again if it gets to this step first

## Max paths

* Defaults to 1

Rules for eBGP
1. Must have had to use tiebreaker
2. Max paths above 1
3. Only eBGP routes where adjacent ASN same is candidate
4. If more candidates exist than can be used, tiebreakers on these

Rules for iBGP
1. Same as rule 1
2. maximum-paths ibgp command for this
3. Only those with differing next hops considered
4. Same as rule 4

* **maximum eibgp** exist, but this is MPLS only

# BGP Communities

* Optional transitive
* **neighbor send-community**
* **ip community-list**

## Match with communty lists

* Were a 32 bit decimal
* When added to BGP standard RFC 1997, formatted AA:NN, still 32 bit
* **ip bgp-community new-format**
* Multiple entries with set or set additive
* Multiple values on same community-list
 * Must included all values (unordered)
* Extended can use regex
* No more than 16 lines in standard
* Many in extended

## Remove community values

* **set community none**
* **set comm-list COMMUNITY-LIST delete**

## Special values

* Can match and also **match exact**

* NO_EXPORT - FFFF:FF01 - Not out of this AS, can be to confeds
* NO_ADVERT - FFFF:FF02 - Not to any other peer
* LOCAL_AS - FFFF:FF03 - Not out the local confed sub-AS (also known as NO_EXPORT_SUBCONFED)

# Fast Convergence Enhancements

* Updates peers every 5 seconds for iBGP, 30 for eBGP
* Also relies on IGP

## Fast External Neighbour Loss Detection

* eBGP direct neighbours immediately torn down if connected subnet 
* Immediate route flush
* Immediate alternative routes
* Enabled by default in IOS 10.0+

## Internal Neighbour Loss Detection

* Since 12.0
* **neighbor fall-over** means keepalive traffic isn't required to signal pull down neighbour quickly
 * Moment IP of peer removed from table, session torn down
 * IGP must be able to find BGP peer immeditely
 * If interrupted, BGP session already disconnected
 * Holdown/delay in BGP session deactivaiton not used

## EBGP Fast Session Deactivation

* Works on all BGP session, quick detects failures of EBGP to loopbacks, or when external fallover disabled
* Per neighbour
 * Disable fast external with **no bgp fast-external fallover** - retains quick response to interface failure
* Identical to iBGP use case described
* Inmplemented through fall-over command
* Can reflect rule that BGP peer has to be correctly connected, with route map command matching only connected subnets

