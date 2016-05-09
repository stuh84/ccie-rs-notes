# Messages

* Open - Establishes neighbours and parameters
* Keepalive - Maintains neighbours
* Update - info
* Notification - Error, takes neighbour down

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
* Default originate - To neighbour, can be conditional, `neighbor X.X.X.X default-originate [route-map name]  


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

# Config
