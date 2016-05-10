# Messages

## Header and label

* 4 byte
* Before IP Header
* 20 bit label field
* Label, EXP, Bottom Of Stack and TTL

## Hellos

* Multicast on 224.0.0.2
* UDP 646
* Has LSRs LDP ID - 32 bit dotted decimal, 2 byte label space number (0 for frame based)
* Transport address transmitted if set
* TCP after neighbour discovery
* After TCP UP, adv all loc bindings and prefixes
* LDP ID like router ID

# Timers

# Trivia

* CEF creates FIB, entry for each dest IP prefix
* CEF adj lists data link header
* LSR uses CEF FIB and LFIB for forwarding
* Label info in both, outgoing int and next hop
* FIB and LFIB differ (incoming unlabelled and labelled)
* TTL - LSRs decrement, Ingress E-LSR drops IP ttl, and copies to MPLS, Egress E-LSR drops MPLS ttl, pops MPLS header, copies to IP header
* Disabling TTL propagation - Ingress E-LSR, MPLS TTL to 255, egress IP TTL unchanged
* `no mpls ip propagate ttl`
* Labels advertised back to router rx'd from

##  MPLS LIB Feeding FIB and LFIB

* Best label chosen and outgoing int
 * populates into FIB and LFIB, both have best labels
* LIB has all labels
* Routing protocol loop prevention

* CEF required

## VPNs

* Outer - forwarding, inner Bottom of stack 1, identifies egress VRF
* Three components of VRF tables, RIB, CEF FIB (based on VRFs RIB), instances/process of routing protocol to CE
* RD is 64 bits long
 * First 2 bytes defines format
 * IOS infers first 2 bytes based on 6 bytes of rd command
* RTs 8 bytes

## FEC

* Set of packets receiving same treatment by single LSR
 * MPLS QoS could be different from another for same prefix
 * MPLS TE - fec is tunnel

## 6PE and 6VPE

* Allows v6 over v4 network
* For single label per prfix, 4000 max per box
* Edge routers dual stack
* Only static routes and BGP for v6 in VRF context
* PEs use v4-mapped v6 address for v6 prefix
* Next hop advertised by PE for both is v4 for v4 L3vpn routes, but with ::FFFF: prepended

### 6PE

* Customers v6 prefixes inside global
* v6 labels/prefixes exhanged using labelled v6 over v4 BGP between PEs

### 6VPE

* customer v6 prefixes in vrf
* v6 labels and precies exchanged via VPNv6

# Processes

# Config

## 6PE

```
ip cef
ipv6 cef
ipv6 unicast routing

router bgp 1000
 no sync
 no bgp default ipv4-unicast
 neighbor 10.108.1.12 remote-as 65200
 neighbour 10.108.1.12 update-source Lo0
 address-family ipv6 
  neighbor 10.108.1.12 activate
  neighbor 10.108.1.12 send-label
```

## 6VPE

```
router bgp 1000
 neighbor 2001::1 remote as 65202
 address family ipv6 vrf VPE1
  neighbor 2001::1 activate
 address family vpnv6
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
``` 

# Verification

```
show mpls ldp bindings ROUTE - Shows LIB entries, remote and local
show mpls forwarding table ROUTE - local entry, outgoing tag and int
show ip cef ROUTE internal - FIB entry
```
