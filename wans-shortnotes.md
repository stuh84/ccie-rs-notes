# L2 Protocols

* For P2P links, most popular are HDLC and PPP
* ISO standard for HDLC has no type field
* 2-byte type added in IOS for multiple protocols
* HDLC default on serial

## HDLC

* Encap not shown in config
* With back to back serial, router connected to DCE end of cable
 * Provides clcok signal for serial
* **clockrate**
* **show controllers** - Verify DCE or DTE

## PPP

* RFC 1661

|Feature|HDLC|PPP|
|-------|----|---|
|Error Detection|yes|yes|
|Error Recovery|no|Yes (IOS defaults to not use reliable PPP feature|
|Standard Protocol Type Field|No|Yes|
|Default on IOS Serial links|Yes|no|
|Supports sync and async links|No|yes|

* RFC 1662
 * Defines PPP Framing using HDLC header and trailer for most parts
 * Adds protocol field and optional padding field
 * Padding field ensures even number of bytes

### PPP Link Control Protocol

* Controls features independent of L3 protocol
* Each L3 protocol has an NCP (network control protocol)
 * IPCP defines dynamic IP assignment etc

* When PPP comes up (i.e. router sends a Clear TO Send, Data Send Read and Data Carrier Detect to bring up physical)
* LCP parameter negotiation
* Auth methods controlled by LCP, in what order
* After LCP negotiation done, considered up
* L3 CP then begins

LCP Features

* Link Quality Monitoring (LQM) - Exchange statistics about percentage errors, link dropped if below a config'd threshold
* Looped link detection - Random magic number picked, if sees own, link looped, might take down
* Layer 2 load balancing - MLP frags each frame into one per link
* Auth - CHAP and PAP

**Basic LCP and PPP config**
```
username R4 password 0
rom 838

int Se0/1/0
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp quality 80
 ppp authenticaiton chap
```

### Multilink PPP

* Original for multiple ISDN B-channels
* Can balance across any p2p serial link
* Frag each data link frame
 * Based on link number or config'd frag delay
* Frags sent across differet links
* Header added including seq number, and flags with beginning and endging fragment
* Can use multilink ints or VTs

```
int Multilink 1
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp multilink
 ppp multilink group 1

int Se0/1/0
 no ip address
 encapsulation ppp
 ppp multilink group 1

int Se0/1/1
 no ip address
 encapsulation ppp
 ppp multilink group 1
```

* show int multilink1

### MLP LFI

* Cisco QoS tool
* Small delay-sensitive packets interleaved into big packets
* Big packets fragged, small packets after portion of longer
* Key elements fragmentation, interleave possibility, and queueing scheduler

MLP supports LFI: -
* **ppp multilink interleave** - Allows interleaving, per int
* **ppp multilink fragment-delay x** - Frag size indirectly (delay in ms, size = x * bandwidth)
* Can be used on one link or multiple
* Queue scheduler determines next packet, often uses LLQ

```
int multilink 1
 bandwidth 256
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp multilink group 1
 ppp multilink fragment-delay 10
 ppp multilink interleave
 service-policy out queue-on-dscp
```

### PPP Compression

* Can neg to use L2 payload compression, or TCP/RTP header compression
* Payload better on larger packets
* Header better n smaller, header compression achieves big ratios (10:1 to 20:1 compression)

**PPP L2 Payload Compression**

* Three options, LZS (Lempel-Ziv stacker), Microsoft P2P Compression (MPPC) and Predictor
* First two use same LZ algorithm
* Predictors algorithm is predictor
* LZ more cpu use, better ratio

* When ATM-FR interworking used, MLPS must be used, so all payload compression types supported by PPP supported for interworking

|Feature|Stack|MPPC|Predictor|
|-------|-----|----|----------|
|Uses LZ|Yes|Yes|no|
|Predictor|No|no|Yes|
|HDLC|Yes|no|no|
|PPP|yes|yes|yes|
|FR|Yes|no|no|
|ATM and ATM-to-FR interworking|Yes|yES|Yes|

* Use **compression** command on each interface
* After config'd, PPP starts Compression CP, negs and manages compression

**Header Compression**

* Two styles, TCP and RTP
* 40 bytes of G.279 packets usually headers (packet 60 bytes)
* Above reduces packet by more than 50%
* For TCP, 40 bytes to 3 or 5, helps on smaller payloads

* Legacy commands
 * **ip tcp header-compression [passive]**
 * **ip rtp header-compression [passive]**
 * Passive awaits negotiation, otherwise tries to enable it
 * All flows compressed when added

* MQC

```
policy-map cb-compression
 class voice
  bandwidth 82
  compress header ip rtp
 class critical
  bandwidth 110
  compression header ip tcp

int Multilink 1
 bandwidth 256
 service-policy out cb-compression
```

## PPPoE

* connects hosts to remote aggregator
* Allows Network Service provider to own customer
* Emulates PPP link across shared medium

### Server config

* Broadband Aggregate (bba) group createdm handles incoming PPPoE config

```
bba-group pppoe BBA-GROUP
 virtual-template 1
 session per-mac limit 2
```
* Above has few macs per session (allows a drop and reconnect with a different mac, and thats it)

```
int virtual-template 1
 ip address 10.0.0.1 255.255.255.0
 peer default ip address pool PPPOE_POOL
```

* When client initiates session, router dynamically creates virtual int

```
ip local pool PPPOE_POOL 10.0.0.2 10.0.0.254
```

* Enable group on int to client
```
int Fa0/0
 no ip address
 pppoe enable group MyGroup
 no shutdown
```

### Client Config

```
int dialer 1
 dialer pool 1
 encapsulation ppp
 ip address negotiated
```

* 8 byte header overhead, so drop MTU of dialer to 1492

```
int Dialer 1
 mtu 1492

int Fa0/0
 no ip address
 pppoe-client dial-pool-number 1
 no shutdown
```

* **show pppoe session**

### Auth

* PAP or CHAP, latter preferred

```
username PPP password PPPpassword

int virtual-template 1
 ppp authentication chap callin
```

```
int dialer 1
 ppp chap password MyPassword
```

# Ethernet WAN

* Many solutions (VPLS, MPLS, AToM, QinQ, Metro-E)

## VPLS

* Brings various WAN connections as logical ethernet network
* Multipoing ethernet LAN services, referred to as Transparent LAN service (TLS)
* CE communicates directly with all other CE nodes
* Usually in ATM< CE node designated to be hub for all spokes

* IETF VPLS links ethernet bridges with Pseudo-Wires
* Operation same as IEEE 802.1 Bridges
 * Self learns source MAP, frames forwarded on dest
 * Unknown macs flooded on all ports

Other functions: -
* Autodiscovery of PE associated with particular VPLS instance
* Signalling of PWs to interconenction VPLS VSIs
* Loop avoidance
* MAC address withdrawal

## Metro-Ethernet

* Ethernet on MAN can be used as Ethernet, EoMPLS, or Eo Dark Fibre

* MPLS-based Metro_E use MPLS in SPs network
* Subscriber gets ethernet interface
* Packets transported over MPLS network
* Ethernet underlying technology transporting MPLS
* LDP signals site-to-site inner label
* RSVP-TE or LDP for outer label
* Metro-E has star or mesh topology
