# Messages

# Timers

# Trivia

## EIGRP

* Only paths reachable through P2P Ints protection
* No v6 support

### LFA Comp

* Pre-computed next hop route
* LFA forwards without knowing of failure
* Computed per link or per prefix (link only protects next hop, not destination)
* EIGRP does prefix

**Tie Breakers**
* interface-disjoint
* linecard-disjoint
* lowest-repair-path-metric - If metric high, eliminate
* SRLG-disjoint

## OSPF

* Not supported on VL head ends
* Only in global VRP
* TE cannot be protected int
* TE Tunnel can be in repair path, won't verify placement
* Not all routes have repair paths
* Precomputes per-prefix repair paths, installed in rib
 * When primary fails, live over stored repair path

### Attributes

1. slrg
2. primary-path
3. interface-disjoint
4. lowest-metric
5. linecard-disjoint
6. node-protecting
7. broadcast-interface-disjoint

* Usually keeps in local RIB only best among all candidates, can keep more (more memory)

## BGP

### PIC Edge

* BGP and IP MPLS up and running, site multihomed
* Backup/alternate has unique next hop
* BFD to detect link failures
* For BGP multipath, PIC already supported
* for v4, v6, vpnv4, vpnv6
* If RR only in control plane, dont need PIC (PIC is data plane)
* Doesn't support NFS with SSO, ISSU
* Only for single network failure
* Doesn't work with BGP Best External

### Convergence improvements

* Second best path calc'd, backup into BGP RIB
* Alternate route installed in RIB
* Stores alt path per prefix in CEF forwarding table
* CEF listens to BFD
* When primary lost, backup searched in prefix independent manner
* MPLS switches if primary disappears
* Two types of failure, core node/link (iBGP), detected through IGP convergence
 * Local link/immediate node - requids BFD
* Data plane convergence - CEF detects alt next hop for all prefixes affected by failure
 * Subsecond
* Control plane - Learns thorugh IGP/BFD, withdraws prefixes

### Improving MPLS VPN local convergence

* maintains local label for 5 minutes, ensures traffic uses backup/alternate
* `protection local-prefixes`
* VPNv4 AF - protects all VRFs
* VRF-IPv4 protects only v4 vrfs
* Router config mode protects global table

### CEF Forwarding Recursion

* Need to disable when using PIC as it searches all FIB entries
* Disabled for next hops with /32 mask and Next hops directly connected
* `bgp recursion host` - for host routes
 * Default enabled on vpnv4 v6, disabled on v4/v6 when PIC enabled
* Disable for DC'd hops with `disable-connected-check`
 
## BGP Add Paths

* Adv mutli paths for same prefix
* Adds path ID for each path in NLRI
* Similar to RD, except any AF
* ID unique to peering session
* Gen'd per network
* Stops overriding announcements
* Different update groups for negotiated capability

## BGP NHT

* Enabled by default on IOS
* Event drive
* Tracks prefixes when peers establish
* Picked up when RIB updates
* Best path calc run in bw scanner cycles, only next hop changes tracked and processed
* Scanner monitors next hop reachability, every 60s

### Selective

* Implement as part of selective tracking
* Route map defines routes to resolve bgp next hop
* `bgp nexthop` - Allows config length f prefix that applies NH attribute
* If next-hop route fails route-map, marked as unreachable
 * Match IP addr or source-protocol in RM

## BGP Support for Fast Peering Session deactivation

* Event driven, per neighbor, monitors session, adj changes detechted
* In between BGP scanning interval

### Selective

* Route map, neighbor fall-over - determines if peering session reset when route to peer changes
* Route map evaluates new route
* If deny return, session reset

# Processes

## BGP Add paths

1. Specify if device can send/rx or both, in AF or neighbour (capability negotiation)
2. Select candidate paths
3. Advertise for a neighbour


# Config

## EIGRP

**LFA FRRs per Prefix**
```
router eigrp dave
 address-family ipv4 unicast autonomous-system 65001
  topology base
   fast-reroute per-prefix { all | route-map NAME }
   fast-reroute load-sharing disable - When selection of LFAs tie breaking, disable load sharing
   fast-reroute tie-break { interface-disjoint | linecard-disjoint | lowest-backup-path-metric | srlg-disjoint } priority-number
```

## OSPF

```
router ospf 1
 fast-reroute per-prefix enable prefix-priority LEVEL - Low pri (all same eligibility), high pri (only high protected)
 prefix-priority high route-map TEST
 fast-reroute per-prefix tie-break ATTRIBUTE [required] index LEVEL
 fast-reroute keep-all-paths

route-map TEST permit
 match tag 11

int Fa0/0
 ip ospf fast-reroute per-prefix candidate disable
```

## BGP

**PIC**

```
router bgp 
 address-family BLA
  bgp additional-paths install
  bgp recursion host
  neighbor X.X.X.X fall-over bfd
```

**Add Paths**
```
router bgp 65000
 address-family ipv4/ipv6 unicast
  additional-paths receive
  additional-paths send
  additional-paths selection route-map NAME 
```

**Add Paths per neighbour**
```
router bgp 65000
 neighbor X.X.X.X remote-as 65001
  address-family ipv4/ipv6 unicast
   capability additional-paths receive [disable]
   capability additional-paths send [disable]
```

* Overrides AF

**Peer Policy**
```
router bgp 65000
 template peer-policy NAME
  capability additional-paths receive
  capability additional-paths send
 neighbor x.x.x.x remote-as 65001
  address-family ipv4 unicast
   inherit peer-policy NAME SEQ-NUMBER
```

**Filtering add paths**
* Match on prefix of add paths that are candidates
* set path-selection all advertise

**Selective Next-Hop Route Filtering**
```
router bgp 65000
 address-family ipv4 unicast
  bgp nexthop route-map CHECK-NEXTHOP
  bgp nexthop trigger delay TIMER - Max 100s, default 5s, full table walks to match IGP parameters
  no bgp nexthop trigger enable

ip prefix-list FILTER seq 5 permit 0.0.0.0/0 le 25

route-map CHECK-NEXTHOP deny 10 
 match ip address prefix-list FILTER

route-map CHECK-NEXTHOP permit 20
```

**Fast Session Deactivation**
*Per neighbour*
```
router bgp 65000
 address-family
  neighbor X.X.X.X remote-as 65001
  neighbor X.X.X.X fall-over
```

*Selective*
```
router bgp 65000
 neighbor X.X.X.X remote-as 65001
 neighbor X.X.X.X fall-over [route-map NAME]

ip prefix-list FILTER seq 5 permit 0.0.0.0/0 le 25

route-map CHECK-NEXTHOP deny 10
 match ip address prefix-list FILTER

route-map CHECK-NEXTHOP permit 20
```
