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

# Config

## Confederations

```
router bgp 65001 <--- SUB-AS
 bgp confederation identifier AS <--- Real AS
 bgp confederation peers SUB-AS <--- What other ASs are in confed
```


