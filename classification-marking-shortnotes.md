# QoS Marking

* IP Header
* LAN Trunking Header
* FR Header
* ATM Cell header

## IP Header and DSCP compared

* RFC791, 1-byte tos field
* ToS byte divided, high order 3 bits are IPP field

|Name|IPP|Binary|
|----|---|------|
|Routine|0|000|
|Priority|1|001|
|Immediate|2|010|
|Flash|3|011|
|Flash Override|4|100|
|Critical|5|101|
|Internetwork Control|6|110|
|Network Control|7|111|

* Bits 3 to 6 are flags, 7 not defined
* ToS byte renamed to DiffServ
* IPP replaced by 6 bit field (high order 0-5)
* Lower order 2 bits for QoS ECN
* Replaces Precedence and TOS
* ToS byte and DS field same length

## DSCP Settings and Terminology

* Decimal 46 - EF

### Class Selector PHB and DSCP values

* RFC2475 defines these
* IPP compatible


|Name|DSCP Binary|IPP Binary|IPP Name|
|----|-----------|----------|--------|
|Default CS0|000000|000|Routine|
|CS1|001000|001|Priority|
|CS2|010000|010|Immediate|
|CS3|011000|011|Flash|
|CS4|100000|100|Flash Override|
|CS5|101000|101|Critical|
|CS6|110000|110|Internetwork Control|
|CS7|111000|111|Network Control|

### AF PHB and DSCP Values

* Three levels of drop probabality
* 4 classes
* AFxy, x being queue, y being priority
* Higher x - better treatment
* Higher y - worse drop treatment


|Queue|AFx1/Decimal/Binary|AFx2|AFx3|
|-----|-------------------|----|----|
|1|AF11/10/001010|AF12/12/001100|AF13/14/001110|
|2|AF21/18/001010|AF22/20/010100|AF23/22/010110|
|3|AF31/26/011010|AF32/28/011100|AF33/30/011110|
|4|AF41/34/100010|AF42/36/100100|AF43/38/100110|

* IPP can still react, but no drop prob
* Decimal from name with 8x + 2y

### EF PHB and DSCP

* Queue EF for low latency
* Police to not starve
* Decimal 46, binary 101110

## Non-IP Header Marking Fields

### Ethernet LAN CoS

* 3 bit QoS field
* Only with .1q or ISL
* 3 most sig bits in tag control, user priority bits - .1q
* 3 least sig from 1 byte user field, called CoS - ISL

### WAN Marking Fields

* FR and ATM have single bit
* For drop prob
* Discard Eligibility - FR
* Cell Loss Priority - ATM
* Can be set by router or ATM/FR switch
* MPLS EXP 3 bit field, remarks DSCP or IPP usually

## Locations for Marking and Matching

* For classification - ingress only, only if int supports header field
* For marking - egress only, as above for support

# Cisco MQC

* All MQC tools are "Cklass Based"

## Mechanics of MQC

* Class map defines matching parameters
* Policy map - PHP actions
* service-policy - bind to interface

## Classification using class-maps

* Match can match things like qos fields, acls, macs etc
* Case sensitive names
* match protocol is nbar
* match any - all packets

### Multiple match commands

* Up to four IPP/8 DSCP in a single match cos, match precedence or match DSCP
 * Matches any, not all
* With multiple match, can define match-any or match-all (default match-all)
* Match class matches anotehr class map (has to match both to match then)

```
class-map match-all to-nest
 match access-group 102
 match precedence 5

class-map match-any nested
 match class to-nest
 match cos 5
```

## Classification with NBAR

* Can refer to host name, URL, mime type etc
* Citrix
* CB marking
* **match protocol**
* RTP matching on even number only, allowing for classification of payload (odd numbers are control traffic)

# Classification and Marking Tools

## CB Marking Config

* Requires CEF
* MQC class maps match packets
* MQC policy refers to 1 or more class maps
* Enabled on itnerface with **serviice-policy in | out**
* Processed sequentially, once matched goes no further
* Multiple sets in one class allowed
* Class-default for unmatched
* With no set, packets in that class not mark

Set example: -
* **set [ip] precedence]** - v4 and v6, unless ip option, then v4 only
* **set [ip] dscp** - as above
* **set cos**
* **set qos-group**
* **set atm-clp**
* **set fr-de**

* **show policy-map NAME** - config
* **show policy-map INTERFACE input/out class CLASS-NAME** - show stats about policy map on interface

```
ip cef

class-map voip-rtp
 match protocol rtp audio

class-map http-impo
 match http url "*important*"

class-map http-not
 match protocol http url "*not-so*"

class-map match-any NetMeet
 match protocol rtp payload-type 4
 match protocol rtp payload-type 34

policy-map laundry-list
 class voip-rtp
  set ip dscp EF
 class NetMeet
  set ip dscp AF41
 class http-impo
  set ip dscp AF21
 class http-not
  set ip dscp AF23
 class class-default
  set ip dscp default

int fa0/0
 service-policy input laundry-list
```

* **show policy-map interface NAME [vc [vpi]/vci] [dlci dlci] [input | output] [class NAME]**
* Load interval sub command defines how often IOS measures packets/bit rates on interface, lower is quicker
 * Default 5 mins, lowest 30s

### CB Marking of CoS and DSCP

```
class-map match-any EF
 match dscp EF

class-map AF11
 match dscp AF11

class-map COS1
 match cos 1

policy-map map-cos-to-dscp
 class cos1
  set dscp AF11
 class cos5
  set ip dscp ef
```

* Cannot apply cos on dot1q native ints

## NBAR

* **ip nbar protocol discovery** on interface
* **show ip nbar protocol-discover interface Fa0/0 stats packet-count top-n 5**
* From 12.2T/12.3 protocol discovery command not required
* Upgrade with PDLMs (Packet Description Language Modules)
 * Download, copy to flash, aadd with **ip nbar pdlm NAME**

## CB Marking Design Choices

* Try and mark as close to ingress as possible
 * So long as device is trusted to make markings

Traffic Recommendations

|Type|CoS|IPP|DSCP|
|----|---|---|----|
|Voice Payload|5|5|EF|
|Video Payload|4|4|AF41|
|Voice and Video Signalling|3|3|CS3|
|Mission Critical Data|3|3|AF31, 32 and 33|
|Transactional data|2|2|AF21, 22, 23|
|Bulk data|1|1|AF11, 12, 13|
|Best Effort|0|0|BE|
|Scavenger|0|0|2, 4, 6|

* Try not to use four or five service classes at most

## Marking Using Policiers

* Can mark based upon thresholds/contracts
* Then drops if network gets full

# QoS Pre-Classification

* For traffic to be encrypted
* ToS byte copied into tunnel header (in IPSec transport, tunnel and GRE)
* NBAR cannot work on this

* IOS can do pre classification instead, keeping original traffic in memory until QoS actions taken
* Enable in tunnel config, VT or crypto map
 * **qos pre-classify**
* See effects with **show interface** and **show crypto-map**

* int tunnel - GRE and IPIP
* int virtual-tempalte - L2F and L2TP
* crypto map - IPSec

# Policy Routing for Marking

1. Examine pacekts on ingress
2. matches subet of packets
3. Mark IPP or TOS
4. Might define route with set command (not required)

# AutoQoS

* Macro deployment

## AutoQoS for VoIP

* Supported on most switches and routers
* Enabled per interface
* Creates global and int config
* CDP detects phone prescence (soft or hardware)
* On uplinks/trunks, trusts CoS or DSCP received and sets up itnerface qos

### Switches

* Assumes access or uplink
* **auto qos voip {cisco-phone | cisco-softphone}**
* If no phone found, DSCP 0 for all traffic
* If phone, trusts QoS markings

On ingress, following in priority queue: -
* Voice/video control
* Real time video
* Voice
* Routing traffic
* STP BPDUs

* All others in normal ingress queue
* On egress, voice placed in priority queue, rest among others

* For uplink, **auto qos voip trust**, trusts DSCP (L3) and CoS (L2)

Config created is: -
* Globally enables QoS
* CoS To DSCP and reverse, maps created
* Priority of expedite in/out queues
* CoS values mapped to queues/thresholds
* As above, DSCP
* Class maps and policy maps for identifying, prioritizing and policing voice

### Routers

* **auto qos voip [trust]**
* Config int bandwidth first
* Config differs based on interface
 * Compression and frag on 768kbps links or lower
 * Traffic shaping and service policy regardless of bandwidth
* On less than 768kps, encap of PPP, PPP multilink created, LFI enabled
* If trust, class maps group traffic on DSCP values
* If not, ACLs created to match voice, data and control
 * Anything else to DSCP 0

### Verifying

* **show auto qos** - shows int  AutoQoS commands
* **show mls qos** - Modifiers to display queueing and CoS/DSCP mappings
* **show policy-map interface**

## AutoQoS for Enterprise

### Discovering Traffic

* **auto discovery qos [trust]**
* CEF required, bandwidth configured
* Trust for traffic marked already
* NBAR discovers types and amounts
* Show run it long enough, then apply

Classification into one of then classes: -
* Routing - CS6, EIGRP, OSPF
* VoIP - EF, RTP Voice Media
* Interactive Video - AF41, RTP video media
* streamng video - CS4, real audio, netshow
* control - cs4, rtcp, h323, sip
* transaction - af21, sap, citrix, telnet, ssh
* bulk - af11, ftp, smtp, pop3, exchange
* scavenger - cs1, p2p
* management - cs2, snmp syslog, dhcp, dns
* best effort - all others

### Generating AutoQoS config

* **auto qos** on interface
* In case of DLCI, applies policy map to FR map class, applies class to DLCI
* Can turn off NBAR with **no auto discovery qos**
* **show auto discover qos** - list types  and amounts of traffic
* **show auto qos**
* **show policy-map interface**
