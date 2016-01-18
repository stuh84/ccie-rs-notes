# RJ45 Pinouts

*T568A*

Pair 1 - 4 5
Pair 2 - 3 6
Pair 3 - 1 2
Pair 4 - 7 8

*T568B*

Pair 1 - 4 5
Pair 2 - 1 2
Pair 3 - 3 6
Pair 4 - 7 8

Pairs 2 and 3 twisted

Straightthrough is A to A, B to B

X-over is A to B

Automdix swaps Pairs 2 and 3 if wrong cable used

# Autoneg, Speed and Duplex

Detected with: -

* Fast link pulses
* Electic signal (if autoneg disabled)

Duplex with autoneg only, Hdx chosen as default on 10/100, fdx on gig

# CSMA/CD

1) Listen until ethernet nost busy (no carrier sign on Ethernet)
2) When not busy, send
3) Listens for no collisions
4) If collisions, all send jamming signal
5) After jamming, each sender of collided frame starts random timer, sends after time
6) After timers, step 1

# Collision domains and switch buffering

* Hubs at L1
* Repeat electric signal
* Forward signal from one port to all

Switches buffer frames (prevents collisions).

NICs uses loopback circuitry for hdx mode, transmitted frames loop back to receive on NIC

# Basic switch config

```
speed { auto | 10 | 100 | 1000 }
duplex { auto | half | full }
```

# Ethernet L2 Framing And Addressing

DEC Intel and Xerox defined ethernet specs

802.3 MAC standard formalised details, others in 802.2 LLC

1 byte DSAP in 802.2 LLC too small. So SNAP formalised, goes after LLC header

**READ BOOK FOR HEADER FIELDS, LOCATION 1026**

# Types of Ethernet Addresses

6 byte macs, 3 main types

I/G = least sig bit

* Unicast - I/G bit set to 0
* Broadcast - Always hex of FFFFFFFFFFFF
* Multicast - I/G set to 1

Unicast has senders mac in source field, other devices mac in dest

# Ethernet address formats

First half of MAC is OUI, vender assigns unique lower 3 bits

Ethernet frame transmitted in user order for bytes, bits in reverse. FCS only exception

Individual/Group bit - Least sig, first bit received

Universal/Local bit - Next to least sig, 0 means vendor assigned address, 1 admin assigned (1 not always supported)

# Protocol types and 802.3 length field

Type used to use same 2 bytes as length. Ethernet type field new header begins at 1536, data limited to 1500 or less

Can defined IP, IPX etc

Protocol Type - dix v2 type field, values by IEEE

DSAP - 1 byte, 2 high order bits reserved

SNAP - 2 bytes, same as protocol type, uses DSAP of 0xAA

# Switching and Bridging Logic

Known unicast - frame to single port with destination

Unknown - out all ports except received

Broadcast - As above

M'cast - As above unless snooping enabled

Switches learn macs on source MAC field in frame

# SPAN, RSPAN and ERSPAN

SPAN - Local
RSPAN - Remote dest, vlan based
ERSPAN - GRE est

## Core concept of SPAN, RSPAN and ERSPAN

SPAN - Create span source and span dest on same switch

RSPAN - Same source, destiantion VLAN, at RSPN dest, RSPAN vlan is source

ERSPAN - Encap'd RSPAN, GRE used, available on IOS-XE, Cat6500, 7600 and Nexus. Monitoring sources are Port Channel, FE and GigE

## Restrictions and conditions

* Dest ports original config overwritten, restored when removing
* Dest ports removed from etherchannel
* Dest don't support port security 802.1x or PVLANs
* Dest ports don't support CDP, STP, VTP etc

Number of conditions to work: -

* Source can be one or more ports or a vlan, not a mix
* 64 span dest max on a switch
* Switches or routed ports can be source or dest
* Don't overload ports (gig mirrored to 10mb)
* Single SPAN cannot deliver traffic to dest when source is mix of SPAN, RSPAN or ERSPAN
* Span dest cannot be source
* Only one dest session per source
* Span dest passes only span traffic
* Trunks can be source
* Traffic routed from VLAN to a source VLAN cannot be monitored (traffic within switch for example)

Default both directions. Restrictions for in one direction: -

RX - Each frame transported with modification
TX - All modifications take place before transmit (eg QoS, ACL filtering etc). Some L2 frames can be exempt using encapsulation command

## Basic SPAN Config

```
monitor session 1 source interface <SOURCE-IF>
monitor session 1 destination interface <DESTINATION-IF>
```

## Complex SPAN Config

```
monitor session 1 source interface Fa0/18 rx
monitor session 1 source interface Fa0/19 tx
monitor session 1 filter vlan 1-3, 229 <--- VLANs not to monitor
monitor session 1 destination interface Fa0/24 encapsulation replicate
```

## RSPAN Config

*Source*

```
vlan 199
 remote span

monitor session 3 source vlan 66-68 rx
monitor session 3 destination remote vlan 199
```

*Intermediate*

```
vlan 199
 remote span

monitor session 23 source vlan 9
monitor session 23 destination remote vlan 199
```

*Destination*

```
vlan 199
 remote span

monitor session 63 source remote vlan 199
monitor session 63 destination interface fa0/24
```

Multiple session IDs can go across. Range of VLANs must be 1-66

## ERSPAN Config

*Source*

```
monitor session 1 type erspan-source
 source interface gig0/1/0 rx
 no shutdown
 destination
  erspan-id 101
  ip address 10.1.1.1
  origin ip address 172.16.1.1
```

*Destination*

```
monitor session 2 type erspan-destination
 destination interface Gi2/2/1
 no shutdown
 source
  erspan-id 101
  ip address 10.1.1.1
```

Verify with show monitor session 1

# VSS

In Cat 6500 and 4500s running IOS-XE

## Virtual Switching System

Multiple physical, one logical

## VSS Active and Standby

Switches have roles in VSS, contend for active and standby

VSS Active controls VSS, runs L2 and l3 protocols and management functions. Packet forwarding done locally, control traffic to active

## Virtual Switch Link

Carries control and data b/w switches. Usually etherchannel, up to 8 links. Higher priority to control and management over link

Data traffic load shared using standard etherchannel algorithm

## Multichassic Etherchannel

Max 256 etherchannels across VSS stack

## Basic VSS config

VSS domain must e between 1 and 255

*Switch 1*

```
switch virtual domain 10
 switch 1
```

*Switch 2*

```
switch virtual domain 10
 switch 2
```

*Switch 1 VSL*

```
int port-chan 5
 switchport
 switch virtual link 1
```

*Switch 2 VSL*

```
int port-chan 10
 switchport
 switch virtual link 2
```

Up/down until reboot

Convert switches with **switch convert mode virtual**, converted config goes to system bootflash

## Verificaiton

show switch virtual - Switch domain, number and role

All consoles of standby's disabled

show switch virtual role - Peer 0 is local switch

show switch virtual link

show siwtvh cirtual link port-channel

# IOS-XE

Same look and feel as IOS, runs linux OS. Single daemon, additional functionality as isolated processes within OS

* Symetrical multiprocesisng allowed.
* Control Plane seperate from Forwarding
* Processes can fail without taking out host
* Can build drivers and use APIs to interact, rather than recording entire OS
* Can iolsate load between planes, i.e. blades in modular chassis
* Different driver for each bay/slot, so no impact on other bays or chassis
* Individual driver patching possible
* FFM - Forwarding and Feature Manager
* FED - Forwarding Engine Driver
* FFM - APIs for control plane processes, programes FED, maintains forwarding states
* FED - Drivers affect data planes
