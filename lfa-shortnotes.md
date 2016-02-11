# EIGRP

## Restrictions for EIGRP LFA FRR

* Only paths reachable through p2p ints protection
* v6 not supported

## Repair Paths Overview

* Forward traffic during routing convergence
* Initially only neighbouring devices aware of failure
* others dont know
* Device adjacent to failure should use repair paths until failure communicated
* Repair paths precomputed

## LFA Computation

* Pre-computed next hop route, without loop back
* LFA in network failure
* LFA forwards without knowing of failure
* LFA computed two different ways
 * Per link - All prefixes through one link sheare same back up, protects next hop address
 * Per prefix - Protects destination address
* EIGRP does prefix-based LFAs

## LFA Tie Breaking Rules

* **interface-disjoint** - No LFA over same path as existing egress int
* **linecard-disjoint** - As above, but same linecard 
* **lowest-repair-path-metric** - If metric high, eliminate
* **SRLG-disjoint** - Shared Risk Link Group, eliminates in same group (could be common fibre)

## Configure

**LFA FRRs per Prefix**

```
router eigrp DAVE
 address-family ipv4 autonomous-system 65001
  topology base
   fast-reroute per-prefix { all | route-map NAME}
```

* **show ip eigrp topology frr**

**Disabling Load sharing among prefixes**

* When primary path ECMP with multiple LFAs
 * Prefixes among LFAs
 * When selection of LFAs tie breaking, disable load sharing among prefixes

```
router eigrp DAVE
 address-family ipv4 autonomous-system 65001
  topology base
   fast-reroute load-sharing disable
```

**Enabling Tie Breaking for EIGRP LFAs**

* Can assign priorities, lower better

```
router eigrp DAVE
 address-family ipv4 autonomous-system 65001
  topology base
   fast-reroute tie-break { interface-disjoint | linecard-disjoint | lowest-backup-path-metric | srlg-disjoint } priority-number
```

# OSPF

## Restrictions for LFA FRR

* Not supported on VL headends
* Only in global VPN VRF
* TE cannot be protected int
* TE tunnel can be in repair path, but won't verify placement
* Not all routes have repair paths

## Info about LFA FRR

### LFA Repair Paths

* Protecting router precomputes per-prefix repair paths
* Installed in RIB
* When primary fails, live traffic over stored repair path

### LFA Repair Path Attributes

Default policy: -

1. srlg
2. primary-path
3. interface-disjoint
4. lowest-metric
5. linecard-disjoint
6. node-protecting
7. broadcast-interface-disjoint

* SRLG - Only for locally configured groups\
 * Repair paths must have different SRLG ID
* Int Prottection - P2Ps have no alternate next hop
 * Prevents selection, protecting int
* Broadcast Int Protection - If computed on same int, but next-hops different, link not protected
* Node Protection - Can bypass primary path gateway router
* Downstream Path - Can specify metric path msut be lower than
* Linecard disjoint
* Metric - Repair path with lwoest metric
* ECMP Primary Paths - Can config primary attribute to specify an LFA repair path from ECMP set, or secondary to those not in ECMP set
* Candidate Repair-Path Lists - Usually keeps in local RIB only best among all candidates, can specify to keep all (more memory)

## Config

**Per-Prefix LFA FRR**

```
router ospf 1
 fast-reroute per-prefix enable prefix-priority LEVEL
```

* Low priority - All prefixes same eligibility
* High priority - Only high priority protected

**Specify prefixes protected**

```
route-map TEST permit
 match tag 11

router ospf 1
 prefix-priority high route-map TEST
```

**Selection policy**

```
router ospf 1
 fast-reroute per-prefix tie-break ATTRIBUTE [required] index LEVEL
```

**List of repair paths considered**

```
router ospf 1
 fast-reroute keep-all-paths
```

**Prohibiting interfaces to be used as next hop**

```
int Fa0/0
 ip ospf fast-reroute per-prefix candidate disable
```

# BGP

## PIC Edge

### Pre-Reqs

* BGP and IP/MPLS network up and running, site multihomed
* Backup/alternate has unique next hop
* BFD to detect link failures

### Restrictions

* For BGP Multipath, PIC already supported
* No BGP PIC for MPLS VPN Inter-AS Option B
* v4, v6, VPNv4 and VPNv6 NLRI
* If RR only in control plane, don't need BGP PIC (PIC is data plane)
* If two PEs are each others alternate path, traffic loops until TTL expires
* No support for NFS with SSO, ISSU is if Route Processors support it
* Solves traffic forwarding only for single network failure at edge and core
* Doesn't work with BGP Best External

### Benefits

* Additional paths for failover
* Constant convergence time
* From IOS XE 3.10S up, labelled PIC and LFA FRR can be togetehr on ASR 903

### Convergence improvements

**BGP Functionality**

* Second best path calc'd along with primary best
* Best and backup into BGP RIB

**RIB Functionality**

* Alternate per route installed if available
* With PIC, if RIB selects route with backup, installed backup with ebst path

**CEF Functionality**
* Stores alt path per prefix
* When primary lost, backup searched for in prefix independent manner
* CEF listens to BFD

**MPLS Functionality**
* Siumilar to CEF
* Stores alt path, switches if primary disappears


* When PIC enabled, backup in RIB, IP RIB and FIB
* Two type of failure
 * Core node/link failure (iBGP) - Failure detected through IGP convergence, detected through RIB to FIB
 * Local link/immediate node (eBGP) - BFD required, CEF looks for BFD events

**Convergence in Data Plane**

* CEF detects alt next hop for all prefixes affected by failure
* Data plane convergence subsecond

**Convergence in Control Plane**
* Learns through IGP/BFD, withdraws prefixes
* Calcs best and backups, advertises next best

### BGP FRRs role in BGP PIC

* FRR provides best and backup in BGP RIB and CEF
* Second best programemd into RIB and CEF, CEF programs linecard
* BGP PIC means CEF can switch to other egress ports if current next hop goes down

![BGP PIC](https://raw.githubusercontent.com/stuh84/ccie-rs-notes/master/images/pic.png)


