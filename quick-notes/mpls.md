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


# Processes

# Config

# Verification

```
show mpls ldp bindings ROUTE - Shows LIB entries, remote and local
show mpls forwarding table ROUTE - local entry, outgoing tag and int
show ip cef ROUTE internal - FIB entry
```
