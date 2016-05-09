# Messages

# Packet formats

* IP header
* 802.1p
* FR Header
* ATM Cell Priority

## IP Header and DSCP

* RFC791 defined 1 byte ToS field
 * High order 3 bytes IPP
  * Routine, Priority, Immediate, Flash, Flash Override, Critical, Internetwork Control, Network Control
 * 3 to 6 - flags, 7 not defined
* ToS byte renamed DiffServ
 * 6 bit field instead of IPP (0-5 bits)
  * Replaces precedence and ToS
  * ToS byte and DS field same length

### DSCP

* Decimal 46 - EF
* Class Selector
 * RFC 2475
 * CS0 - Binary 000000
 * CS1 - Binary 001000
 * CS2 - Binary 010000
 * Etc up to CS7
* AF PHB
 * Three levels of drop prob
 * 4 Classes
 * 4xy, X being queue, Y being priority
  * Higher X, better treatment
  * Higher Y worse drop
  * AF11, 12 and 13 - 001010, 001100, 001110
  * AF21, 22 and 23 - 010010, 010100, 010110
  * AF31, 32 and 33 - 011010, 011100, 011110
  * AF41, 42 and 43 - 100010, 100100, 100110
  * Decimal from name - 8x + 2y
  * Still IPP compatible

### LAN CoS

* 3 bit QoS Field
* Only with .1q or ISL
* 3 most sig bits in tag control, user priority in 1q
* 3 least sig from 1 byte user field, called CoS in ISL

### WAN

* FR and ATM have single bit for drop probablity
* DE for FR, CLP for ATM
* Can be set by rotuer or ATM/FR switch
* MPLS EXP bit, remarks DSCP or IPP usually

# Trivia

* Classify on ingress only, mark on egress only

## MQC

* All MQC tools "Class Based"
* Class map matches
 * Can match QoS Fields, ACLs, macs
 * Case sensitive
 * match protocol - nbar
 * Match any - all packets
 * Up to four IPP or 8 DSCP in single match CoS, Prec or DSCP
  * matches any, not all
 * Can define to match any or match all for multiple matches
 * Match class matches another class map (so can match all in one, then match any in another)
* Policy map applies actions
* service policy binds to interface

### NBAR

* Can refer to hostname, URL, mime type etc
* Citrix
* RTP matching on even number only, can classify payload (odd numbers are contro)
* `ip nbar protocol discovery` - Not required after 12.2T/12.3
* Upgrade PDLMs, add with `ip nbar pdlm NAME`


### CB Marking

* CEF required
* Processed sequentially, onced match no further
* Multiple sets allowed
* Class-default for unmatch
* Load interval sub command defines how often IOS measures packets/rates on interface
 * lower quick, default 5 mins, lowest 30s

### Pre-Classification

* For traffic to be encrypted
* ToS byte to tunnel header (in IPSec transport, tunnel and GRE)
* Can't use NBAR for this
* Pre-classification instead, keeps traffic in memory
* Use `qos pre-classify` on tunnel, VT or crypto map
* See with show interface and show crypto-map
 * Int tunnel for GRE and IPIP
 * VT - L2F and L2TP
 * Crypto map - IPSec

### Policy routing

* Mark IPP or ToS on ingress packets

## Auto QoS

### VoIP
**Switches**
* Enabled per int
* CDP detects phone (soft or hardware)
 * auto qos voip {cisco-phone | cisco-softphone}
 * If no phone found, DSCP 0
 * If phone, trust QoS markings
 * Voice/video control, real time video, voice, routing and STP BPDUs in priority queue
 * All others in normal
 * On egress, voice in priority, rest on others
* On uplinks/trusts, trusts CoS/DSCP, sets up int qos
 * auto qos voip trust - Trusts DSCP and CoS
* Globally enables QoS
* CoS-DSCP and reverse maps created
* Priority queues
* CoS/DSCP values mapped to queues/thresholds
* Class maps and policy maps identify/prioritize/police voice

**Routers**
* auto qos voip [trust]
* Need int bandwidth setting
 * Below 768kbps - compression and frag, PPP encap, PPP multilink, LFI
 * Traffic shaping and service policy at all bandwidths
* If trust, class maps group traffic on DSCP
* If not, ACLs match voice, data and control, rest DSCP0

### Enterprise

* auto discovery qos [trust]
* CEF required, bandwidth config'd
* Trust for traffic marked already
* NBAR discovery
* Run it long enough, then apply
 * Routing - CS6 - Routing protos
 * VoIP - EF - RTP
 * Interactive Video - AF41 - RTP
 * Streaming video - CS4 - real audio, netshow
 * Control - CS4 - RTCP, H323, SIP
 * Transaction - AF21 - SAP, Citrix, telnet, SSH
 * Bulk - AF11 - FTP, SMTP, POP3, exchange
 * Scavenger - CS1 - P2P
 * Management - CS2 - SNMP, SYSLOG, DHCP, DNS
 * Best Effort - Everything else
* `auto qos` in interface
 * DLCI - Applies to FR map class, class to DLCI
 * Turn off NBAR with `no auto discovery qos`

# Processes

# Config

# Verification

```
show policy-map NAME

show policy-map INTERFACE input/out class NAME

show ip nbar protocol-discovery interface Fa0/0 stats packet-count top-n 5

show auto qos

show mls qos

show auto discover qos
```
