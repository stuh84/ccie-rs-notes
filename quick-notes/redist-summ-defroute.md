# Messages

# Timers

# Trivia

## Route Maps

* For redist, routes from current routing table

## Redistribution

* OSPF default cost 20 from IGP, 1 from BGP
* ISIS - 0, default type of l1

## Summarization

* Eigrp - ip summary-address eigrp ASN 192.168.0.0 255.255.255.0 DISTANCE - int, or done in af-interface, plus setting summary-metric within topology base
* OSPF - summary-address for ASBR, area 1 range for ABR, area is where component subnets area, not where to advertise

## Default routes

* Redistribution
* Static route to 0.0.0.0 with redist static - EIGRP and RIP
* default-information-originate - RIP, OSPF
 * OSPF - any defaults in routing table, default cost 1, E2
 * RIP creates and advertises if no default exists or from another protocol
  * If static, no injection as done with redist
* ip default-network - RIP,EIGRP
 * EIGRP - must be advertised by local router into EIGRP, flagged as candidate
 * RIP - no flag, no need to be advertised
* summary routes - EIGRP
 * Creates null route on local

## Performance Routing

* Interfaces
 * Internal - Connects to internal network, commes with PfR master controller
 * External - packets out network, needs at least two
 * Local - forms control plane, defines source to communicate to PfR MC
* Auth - not optional, key chain auth
* MC - Talks to and auths BRs, monitors flows, applies policies, single MC supports 10 BRs or 20 exit ints
 * Can run MC and BR on same device, still needs auth


# Processes

# Config

## AD change

* distance DISTANCE - RIP
* distance eigrp INTERNAL-DIST EXTERNAL-DIST
* distance ospf intra-area DIST inter-area DIST external DIST

## PFR

**MC**
```
key chain PFR_AUTH
 key 1
  key-string DAVE

pfr master
 border 2.2.2.2 key-chain PFR_AUTH
  int Se0/0.21 internal
  int Fa0/0 external
 border 3.3.3.3 key-chain PFR_AITH
  int Se0/0.31 internal
  int Fa0/0 external
```

**BR**
```
key chain PFR_AUTH
 key 1
  key-string DAVE

pfr border
 master 4.4.4.4 key-chain PFR_AUTH
 local loopback 0
 logging
 port 3950
```


# Verification

```
PFR

show oer master border <--- MC Active on MC when both BRs up 
```
