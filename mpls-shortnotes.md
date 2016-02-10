# MPLS Unicast IP Forwarding

* Forwarding on labels
* Only considers routes in unicast table
* Provides apps like VPNs and TE

## MPLS IP Forwarding Data Plane

* Relies on structure and logic of CEF

### CEF Review

* Routing protocols, statics and connected to create RIB
* CEF creates FIB, entry for each dest IP prefix
* Details next hop and int
* CEF adj lists data link header
* CEF optimizes FIB for smaller forwarding delay, higher PPS

### Overview of MPLS Unicast IP forwarding

* LSR - Any router pushing, pop, forwarding labelled packets
* Edge LSR - Processes labelled and unlabelled
* Ingress E-LSR - Rx unlabelled, tx labelled
* Egress E-LSR - Opposite of above
* ATM-LSR - MPLS control plane, sets up VCs, forwards labelled packets as ATM cells
* ATM E-LSR - Also performs ATM Segmentation And Reassembly (SAR) function

### MPLS Forwarding using FIB and LFIB

* LSR uses CEF FIB and LFIB for forwarding
* Label info in both, outgoing int nd next hop
* FIB and LFIB differ, one for incoming unlabelled, other labelled

## MPLS Header and Label

* 4 byte
* Before IP header
* 20-bit label field
 * Shim header
* Label, EXP, Bottom-Of-Stack (1 means bottom) and TTL (same as IP TTL)

### MPLS TTL Field and TTL Propagation

* TTL so LSRs can ignore IP header entirely
* LSRs decrement TTL field
* INgress E-LSR - drops IP TTL, adds label, copies TTL to MPLS TTL
* LSR - TTL dropped when label swapped
* Egress E-LSR - After MPLS TTL dropped, pops MPLS header, copies TTL to IP header

**Disabling TTL Propagation**
* Ingress E-LSR - MPLS TTL to 255
* Egress E-LSR - IP TTL unchanged
* Can disable it for two types of packets (disable for customers, leave on for SP routers)
* **no mpls ip propagate-ttl**

## MPLS IP Forwarding: Control Plane

### MPLS LDP Basics

* Adv label for each prefix in IP routing table
* LDP sends messages to neighbours, with IP prefix and corresponding label
* New route in table means new LDP adv
 * Local label assigned
* MPLS LSP - labels across path
* Even advertises labels back to router it received it from

### MPLS LIB feeding FIB and LFIB

* LSRs store labels and info inside LIB
* Best Label chosen and outgoing int
* Populates info FIB and LFIB
* FIB and LFIB have best labels
* LIB has all labels
* Uses routing protocols loop prevention, reacts to IGP choices

```
ip cef

int Gi1/0/1
 mpls ip

router eigpr 1
 network X.X.X.X
```

* **show mpls ldp bindings** *route* - Shows LIB entries, remote beindings and local bindings

### Examples of FIB and LFIB entires

* **show mpls forwarding table** *route* - Local entry, outgoing tag, outgoing int
* **show ip cef** *route* **internal** - FIB entry
* **show mpls ldp bindings** - LIB entries

## LDP Reference

* Uses Hellos
 * Multicasts on 224.0.0.2
 * UDP 646
 * List LSRs LDP ID
  * 32 bit dotted decimal, 2 byte label space number, always 0 for frame based
 * Transport address transmitted if set
  * IP LSR wants to use for LDP TCP connections
  * LDP ID if not set
* After nghbr discovery, TCP to each neighbour
 * Port 646
* Addresses must be reachable in unicast table
* After TCP up, adv all local bindings of labels and prefixes
* LDP ID chosen just like router ID (Config, Highest IP of loopback, Highest IP of int)

# MPLS VPNs

## Solution to Duplicate IP ranges

* VRFs - separates routes
* CE - no knowledge of MPLS protocols, no labelled packets
* PE - LSR linked to at least one CE
* P - Just forwards labelled packets
* Exchange to CE with eBGP, RIPv2, OSPF or EIGRP
* iBGP to exchange routes
* Two labels
 * Outer MPLS header - S-Bit = 0 - Forwarding
 * Inner - S-Bit = 1 - Identifies egress VRF for forwarding decision

## Control Plane

### VRF tables

**Three components**
* RIB
* CEF FIB, populated based on VRFs RIB
* Separate instance/process of routing protocol to CE, VRF support required

### MP-BGP and Route Distinguishers

* RD goes in front of original BGP NLRI
* Different number per customer, makes NLRI unique
* MP-BGP added RDs in RFC 4364
* 64-bits long, prepended onto v4 prefix (vpnv4)
* RD 8 byte, some quite formatting conventions
 * First 2 bytes - Defines format
* IOS can tell which used
* **rd** command only requires lst 6 bytes as integers, infers first 2 based on that

**Formats**
* 2-byte-integer:4-byte-integer
* 4-byte-integer:2-byte-integer
* 4-byte-dotted-decimal:2-byte-integer

* For all three, first should be ASN or v4 address
* Second can be anything

### Route Targets

* RTs are BGP Extended Community PA
* Generally 8 bytes in length
* Same basic format as RD
* One or more per prefix
* Determines which VRFs to place routes into
* Export and Import, Import says what to pull into VRFs RIB
* Usually single RT for import and export

### Overlapping VPNs

* Works using RT
* CE needs to be reachable in different VPNs
* Route leaking
* Multiple RTs

## MPLS VPN Configuration

```
ip vrf Cust-A
 rd 1:111
 route-target 1:100 both

ip vrf Cust-B
 rd 2:222
 route-target import 2:200
 route-target export 2:2000

int Fa0/1
 ip vrf forwarding Cust-A
 ip address 192.168.15.1 255.255.255.0

int Fa0/0
 ip vrf forwarding Cust-B
 ip address 192.168.16.1

router eigrp 65001
 address-family ipv4 vrf Cust-A
  autonomous-system 1
  network 192.168.15.1 0.0.0.0
  no auto-summary
  redistribute bgp 65001 metric 10000 1000 255 1 1500
 address-family ipv4 vrf Cust-B
  autonomous-system 1
  network 192.168.16.1 0.0.0.0
  no auto-summary
  redistribute bgp 65001 metric 5000 500 255 1 1500

router bgp 65001
 address-family ipv4 vrf Cust-A
  redistribute eigrp 1
 address-family ipv4 vrf Cust-B
  redistribute eigrp 1

router bgp 65001
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source loop0
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community
```

* Metric in BGP is MED, i.e. from IGP
* **show ip bgp vpnv4 all**
* **show ip route vrf**

## MPLS VPN Data Plane

* RDs allow unique preferences
* RTs says routes for VRFs
* Ingress PE has appropriate FIB entries
* Ps and PEs need LFIB entries

1. Unlabelled packet on VRF int, VRF FIB for forwarding decision
2. Ingress PE VRF FIB with outgoing int and label stack
3. P pops outer label, pushes own label for destination
4. When PE receives it, does two LFIB lookups, pops outer, sees inner is in LFIB

* Inner label for Egress PEs forwarding details, in particular outgoing int for unlabelled packet

### Building VPN label

* Inner label for each router added to each customers VRF
* New local labels associated with prefix, stored in LFIB
* Once local labels assigned, added to BGP table entry for routes
* Advertised in BGP update

### Creating LFIB Entries to Forward Packets to Egress PE

* iBGP routes list next hop
* LDP built for BGP next hop
* Must have route for next hop
* Label into LFIB

### Creating VRF FIB entries for Ingress PE

* Incoming packet on VRF int
* Forwarding using VRFs FIB

**2 labels in fib entry**
1. PE redists route from BGP in vrf
2. PE builds VRF FIB entry for route
3. New FIB entry has VPN label too and outer label for forwarding

### PHP

* Pops outer label before forwarding onto last hop
* Egress PE now only looks at inner label

# Other MPLS apps

* FEC (Forwarding Equivalency Class) - set of packets receiving same treatment b ysingle LSR
* For unicast, each v4 a FEC
* For VPNs, each prefix in VRF
* With QoS, one FEC different from another for same prefix potentially
* Label for each FEC, different labels for different forwarding details
* MPLS TE allows some packets over one LSP, some over another
* FEC in this is a TE tunnel
* For M'cast, extensions to PIM, exchanges FEC-to-Label Binding
* MPLS QoS, extensions to TDP/LDP

# Implementing Multi-VRF Customer Edge

* Multiple routing tables on single rotuer
* L3 separation
* Internetworks with overlapping IP space
* Same config commands as MPLS VPN

## VRF Lite, without MPLS

* Build VRF< associate interfaces
* Adds any routing protocols in VRF
* Multiple VRFs on single link need sub-ints

```
ip cef

ip vrf COI-1
 rd 11:11
 route-target both 11:11

ip vrf COI-2
 rd 22:22
 route-target both 22:22

int Se0/0/0
 encap frame-relay
 no shut
 desc To RouterLite2

int Se0/0/0.101 point-to-point
 frame-relay interface-dlci 101
 ip vrf forwarding COI-1
 ip address 192.168.4.1 255.255.255.252

int Se0/0/0.101 point-to-point
 frame-relay interface-dlci 101
 ip vrf forwarding COI-2
 ip address 192.168.4.5 255.255.255.252
```

* Usual config for rest

## VRF Lite with MPLS

* Multi-VRF CE
* Allows CE to have VRF awareness, remain in CE
* Could be multi-tenant unit
* Add VRFs to ints, routing protocols normal up sub-ints to PE
* PE then does MPLS
