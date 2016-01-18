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

Test
