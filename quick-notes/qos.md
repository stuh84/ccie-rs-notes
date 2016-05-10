# Messages

## RSVP

* PATH - From originator to terminating GW
* RESVs - in response
* PATHs again to originator from terminator
* RESVs in response
* ResvConfs in response from terminator
* PHOP recorded (previous hop) put in Path messages to point return path for next device
* Periodic refresh messages

# Concepts

* TxQueue or Tx Ring on interface
 * FIFO
 * No effect from IOS tools
 * Shrinks to small length when tools exist
 * `tx-ring-limit 1`
* Queueing on Ints versus Subifs and VCs
 * Shaping function queues packets, by default in fifo queue  

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

## CBWFQ and LLQ

* CBWFQ reserves bandwidth for each queue
* WFQ in default queue
* LLQ - Priority with bw limit
* Bandwidth in CBWFQ
* priority in LLQ
* 64 queues/classes
* Max queue length can be changed
* Class-default always exists
* If all queues with large number of packets, percentage of bandwidth as implied by config
 * If some empty, distributed proportionally
* FIFO on 63 queues, WFQ on class-default (Cisco 7500 can be WFQ on all)
* Schedules among all - Percentages of guaranteed bandwidth

### Defining and limiting bandwidth

* Pre IOS 15 - checked to make sure too much not allocated, service policy rejected
 * Based on bandwidth and max-reserved-bandwidth
* No max-reserved in IOS bw, based on int-bw
* bandwidth percent - based on int-bw
* If remaining percent, percentage of classes reserved bandwidth (from bandwidth percent)
* Explicit bw - sum of values lt or eq to max-res x int-bw
* Percent - lt or eq to max res
* remaining - lt or eq to 100

### LLQ

* Scheduled first
* `priority {bandwidth-kbps | percentage PERCENTAGE} [burst]` - Guaranteed min and policed max
 * burst 20% by default
* Actual bw from LLQ and nonLLQ not exceed max-res x int-bw
* LLQ took from remaining bw, meaning rest can total 100

* Bandwidth under class default - reserved
 * if not set, whatevers left
* Congestion = full hw queue

## WRED

* Monitors queue length - discards more when longer
* Average queue depth compared to min and max threshold
* If Avrg lt min - NO DROP
* Min lt Avrg lt Max - Random Drop
* Avrg gt Max - Full drop 
 * Not tail drop, as max thresh might not be full queue size
* MPD - Mark Probability denominator
 * Discard percentage at max threshold is 1/MPD

* Min Thresh 20, max 40, MPD of 10
* Between min and max, random drops
* At max, 10% discard
* Over max, 100% discard

### Weighting

* Each profile has min, max and MPD
* Higher MPD = lower drops during time b/w thresholds
* Defaults - DSCP, Min Thresh, max Thresh, MPD, 1/MPD
 * AFx1 - 33, 40, 10, 10%
 * AFx2 - 28, 40, 10, 10%
 * AFx3 - 24, 40, 10, 10%
 * EF - 37, 40, 10, 10%

## MDRR

* 12000 only
* 7 round robins (0-6), 1 priority
* When none in priority, each queue used once per cycle
* When in priority, strict or alternate
 * Strict - served first always - can get more than config'd bw
 * Alternate, 0, P, 1 P etc
* Quantum Value and Deficit
* QV - number of bytes
* Any extra bytes are deficit (can't take half packets)
* Each queue gets a guaranteed bw

## LAN Switch Congestion Management and Avoidance

### Ingress

* 3560 - Two Ingress queues, one can be PQ
* WTD has discard thresholds per queue
* SRR schedules
* Rate packets from ingress to switch fabric
 * In shared, bw shared between two queues according to weights
 * Bw guaranteed, not limitied
* If one queue emptry, other gets all bw
* Relative weights rather than absolutes (ratios), like CBWFQ with percentages

### Congestion Avoidance

* WTD on by default when QoS on switch
* Three thresh per queue (CoS based) - tail drop when particular percentage
* Defaut drop at 100% of queue
 * works for priority
* Could do Thresh 1 drops traffic for values 0-3 when reach 40%, 2 with Cos 4 and 5 at 60%, 3 for 6 and 7, 100%
* Thresh 3 always 100%
* Separate from queues config

## Egress Queuing

* Four queues
* Configure CoS/DSCP to queue
* Weights
* Drop threshold
* PQ always queue 1
* WTD config'd like ingress
* Commands at int level, not global
* Can shape to slow egress traffic

## Traffic Shaping

* Queues packets to delay
* release oer time
* Routers only send at clock rate
* Lowers rate by alternating between sending and silence
* Static time interval set (Tc)
* Bits per Tc = Bc (committed burst)
* Tc = Bc/CIR (in ms)
* Bc = committed burst in bits
* Shaped rate - Bps
* Be - excess burst, bits over Bc after inactivity

### Mechanics

* Tc = BC/Shape rate
* Token buckets
* Bucket filled every Tc start
* Each token 1 bit
* At start, shaping can release Bc bits
* Overflow - Used for Be if exists (can send more at next Tc)

### Generic Traffic Shaping

* `traffic-shape rate RATE [bc] [be] [buffer-limit]`
* Buffer = max queue buffer in Bps
* Only shaped rate required, Bc and Be then quarter of Rate

### CB Shaping

* Class for different shape rates
* `shape [average | peak] MEAN-RATE [[burst-size][excess-burst-size]]`
* On 320kbps or less
 * 8000 bits Bc and BE
 * Tc = Bc/shape rate
* On more
 * BC = shape rate by tc
 * Be=Bc
 * Tc = 25ms

**Bandwidth Percent**
* Sees bw of int or subint
* Sub ints don't inherit phy bw, 1544 default
* Bc and Be config'd as Ms (bits sent at configd rate in time period)
* Tc set to Configure Bc
* Eg shape average 50 125ms
 * 50 is shaper rate
 * 125ms is bc

**Peak rate**
* Refills Bc + Be for time intervals
* Can send Bc and Be per time period
* Shaping rate becomes
 * Config'd rate x (1 + Be/Bc)
 * 64 (1 + 8000/8000) = 128

**Adaptive Shaping**
* `shape adaptive MIN-RATE`

## Policing

* Monitors bit rate of combined packets

### Single Rate, Two Colour

* No excess burst
* Policer refills bucket according to policing rate
* Token is a byte, 96kbps over a second, bucket filled with 12,000 tokens
* Not refilled on time interval
 * Reacts to arrival of packet, prorates tokens
 * ((Current packet arrival - Previous packet arrive) x Police_Rate) / 8
* Then decides if newly arrived packet conform or exceed
* Number of bytes (xp)
* Number of tokens in bucket (Xb)
* Conform - Xp lt or eq to Xb, takes Xp tokens
* Exceed - Xp gt Xb - None

### Single Rate Three Colour

* First bucket like before
* If Bc bucket oveflows, fills Be
* After filling buckets, another option
 * if Xbe gt or E Xp Gt Xbc, tokens from Be bucket only
* PIR (Peak information rate)
* Packets under CIR conform
* Packets under PIR exceed
* Conform - Xp lt or eq Xbc - Xp tokens from Bc and Xp from Be
* Exceed - Xbc lt Xp lt or Eq Xbe - Xp tokens from Be only
* Violate - Xp gt Xbc and Xp gt Xbe - none

### Defaults for Bc and Be

* If Bc not config'd, equivalent in bytes to 1/4 send at police rate
 * Bc = CIR/32 (i.e. bits to bytes)
* Single rate two colour - BC = CIR/32, Be = 0
* Single rate three colour - Bc = CIR/32, Be = Bc
* Dual rate three colour = Bc = Cir/32, Be = PIR/32

### Dual rate

* Has CIR and PIR
* `police {cir CIR} [bc CONFORM-BURST] {pir PIR} [be PEAK-BURST] [conform-action ACTION exceed ACTION [violate-action ACTION]]`

### Multi action

* Just go into policing sub config mode, and add multiple lines (eg multiple violates, multiple exceeds etc)

### Policing by percentage

* Config'd by num of ms
* IOS calcs Bc and Be by how many bits sent in that many ms

### Committed Access Rate

* Single rate two colour
* Set rate in bps, bc and be
* Can use ACLs
* Uses rate limit command
* Does support be, but no violate category

## Hierarchical Queuing Framework

* Tree structure using policy maps
* When data through interface using HQF, data classified so it traverses tree branches
* Idea is to apply service policies to service policies
 * Gives the ability to apply a global approach to traffic, while also saying some traffic will have other extended needs
 * Eg, shape everything to a rate, but then for certain traffic (eg HTTP) give it a different amount of bandwidth percentage
* Flow based Fair queueing in Classdefault rather than WFQ - scheduled equally rather than IPP or DSCP
* Default FIFO when no polucy map
* Class default abd bw - Minimum 1% of int by default
* Default queueing for Shape CLass is FIFO
* Policy maps can reserve 100% of int bw
 * No explcit guarantee in class default, can have max 99% of int bw
* In HQF, shaping after encap on re
* When shape in parent policy applied to tunnel, can use class-default only
 * Cannot configured user defined class in parent


# Processes

## LAN Switch Ingress Queueing

* Which traffic to put into queue (Cos 5 default queue 2, others queue 1)
 * Can use DSCP
* Priority traffic?
* Bw and buffer space per queue
* Whether WTD appropriate

## LAN Switch Egress Queueing

**When frame given internal DSCP and egress int, following logic for queue placement**

1. Frames internal DSCP to DSCP-to-CoS map
2. Per int CoS to Queue map

* Shared/Shaped SRR
 * Sharped rate limits won't exceed config'd percent of links bandwidth
 * `srr-queue bandwidth share WEIGHT1 WEIGHT2 WEIGHT3 WEIGHT4`
 * `srr-queue bandwidth shape WEIGHT1 WEIGHT2 WEIGHT3 WEIGHT4`
* Default weights 25
 * Shared keeps servicing one queue to int bandwidth
 * Shaped only gets config'd percent
* If Queue 1 (PQ) empty, others serviced
* If frame in PQ - serviced
 * Scheduler limits bw
 * Limits excess above queue, doesn't discard
* DSCP/CoS maps to queues global config (as per input)
* Each interface belongs to one of two egress queue sets
* BUffer and WTD done per queue set
* Bandwidth, shaped, shared and priority done on interface

## RSVP

* Unidirectional
* Flow based on dest IP, protocol ID, dest port


# Config

## CBWFQ

* bandwidth VALUE (in kbps) - Literal
* bandwidth percent - Percentage
* bandwidth remaining percent
* queue-limit - Max queue length
* fair-queue [queue-limit VALUE] - WFQ in class

```
int Se0/0
 max-reserved-bandwidth 85
 service-policy output NAME
```

## WRED

* Configured with Queue
* Not in LLQ, can be on Physical int or ATM VC
* `random-detect dscp DSCPVALUE MINTHRESH MAXTHRESH [MPD]`
* As above but precedence
* Change calc of queue with with Exponential Weight constant - Lower means old average smaller part of calc (so quick change average)
 * `random-detect exponential-weight-constant CONSTANT`

## Switch Ingress Queuing

### Priority Queue

* `mls qos srr-queue input priority-queue ID bandwidth WEIGHT` - Weight = percentage of links bandwidth
* Packets in priority served the moment current frame finished in non-priority

```
mls qos srr-queue input cos-map queue 2 6 <-- Queue value 2, CoS value 6
mls qos srr-queue input priority-queue 2 bandwidth 20 <--- Serve 20% of total bandwidth, before then sharing between queues
mls qos srr-queue input buffers PERCENT1 PERCENT2 <--- Ratio of interface buffers per queue (default 90% queue 1, 10% queue 2)
mls qos srr-queue input bandwidth WEIGHT1 WEIGHT2 <--- Frequency of scheduling from buffers, default 4 and 4
```

* Default settings - Queue 2 priority, Queue 2 10% bw, Cos 5 in queue 2

### Congestion avoidance

```
mls qos srr-queue input threshold QUEUEID threshold-percent1 threshold-percent2
mls qos srr-queue input cos-map threshold threshold-id cos1...cos8 <--- replace for DSCP

mls qos srr-queue input threshold 1 40 60
mls qos srr-queue input cos-map threshold 1 0 1 2 3
mls qos srr-queue input cos-map threshold 2 4 5 
mls qos srr-queue input cos-map threshold 3 6 7
```

## Switch Output Queueing

```
mls qos queue-set output 1 buffers 40 20 30 10
mls qos queue-set output 1 threshold 2 40 60 100 100 <--- QUEUE-ID THRESH1 THRESH2 RESERVEDBW MAXBW

int Fa0/1
 queue-set 1
 srr-queue bandwidth share 10 10 1 1
 srr-queue bandwidth shape 10 0 20 20 <-- Shape overrides share
 priority-queue out
```
* In above, thresh1 and thresh2 defined, thresh3 always 100%
* Reserved BW - How much of the bandwidth is reserved for the queue
* Max BW - How much the queue can expand if no other traffic present - range of 1  to 3200%
* 0 in Shape means no limit (so queue 2 would not be shaped)
mls qos queue-set 1 threshold 1 138 138 92 138
mls qos queue-set 1 threshold 1 138 138 92 138
* Q1 can either be prioritized or shaped

## RSVP

* Allocates bw per flow
* Bw per int allowed

```
int fa0/0
 ip rsvp bandwidth TOTAL-KBPS SINGLE-FLOW-KBPS <--- 75 percent of int-bw default total
 ip rsvp signalling dscp VALUE - DSCP for rsvp messages
 ip rsvp resource-provider none <--- Disables RSVP WFQ when LLQ exists
 ip rsvp data-packet classification none <--- Stops RSVP processing all packets
```

* Flow of 0 means full amount
* RSVP reserves voice for voice calls
* Gateways place in priority queue
* When using LLQ, PQ size includes l2 overhead
* RSVP bw does not - set RSVP bw equal to PQ minus L2 overhead

## CB Policing

```
policy-map police-all
 class class-default
  police cir 96000 bc 12000 be 6000 conform-action transmit exceed-action set-dscp-transmit 0 violate-action
```

* Apply as above per class for different treatment of different traffic

# Verification

```
show policy-map NAME

show policy-map INTERFACE input/out class NAME

show ip nbar protocol-discovery interface Fa0/0 stats packet-count top-n 5

show auto qos

show mls qos

show auto discover qos

show controllers <-- tx_limited is Tx_ring

show mls qos input-queue
Distribution1#show mls qos input-queue
Queue     :       1       2
----------------------------------------------
buffers   :      90      10
bandwidth :       4       4
priority  :       0      10
threshold1:      40     100
threshold2:      60     100

To achieve above: -

mls qos srr-queue input buffers 90 10 
mls qos srr-queue input bandwidth 4 4
mls qos srr-queue input threshold 1 40 60
mls qos srr-queue input threshold 2 100 100

show mls qos queue-set

SW-3560#sh mls qos queue-set 
Queueset: 1
Queue     :       1       2       3       4
----------------------------------------------
buffers   :      10      10      26      54
threshold1:     138     138      36      20
threshold2:     138     138      77      50
reserved  :      92      92     100      67
maximum   :     138     400     318     400
Queueset: 2
Queue     :       1       2       3       4
----------------------------------------------
buffers   :      16       6      17      61
threshold1:     149     118      41      42
threshold2:     149     118      68      72
reserved  :     100     100     100     100
maximum   :     149     235     272     242

mls qos queue-set 1 threshold 1 138 138 92 138
mls qos queue-set 1 threshold 2 138 138 92 400
mls qos queue-set 1 threshold 3 36 77 100 318
mls qos queue-set 1 threshold 4 20 50 67 400
mls qos queue-set 1 buffers 10 10 26 54

mls qos queue-set 2 threshold 1 149 149 100 149
mls qos queue-set 2 threshold 2 118 118 100 235
mls qos queue-set 2 threshold 3 41 68 100 272
mls qos queue-set 2 threshold 4 42 72 100 242
mls qos queue-set 2 buffers 16 6 17 61

SW-3560#sh mls qos maps cos-output-q 
   Cos-outputq-threshold map:
              cos:  0   1   2   3   4   5   6   7  
              ------------------------------------
  queue-threshold: 4-3 4-2 3-3 2-3 3-3 1-3 2-3 2-3 

CoS 0 Mapped to Queue 4, threshold 3
Cos 1 Mapped to Queue 4, threshold 2
etc etc

eg 

mls qos srr-queue output cos-map queue 4 threshold 3 0

SW-3560#sh mls qos maps dscp-output-q 
   Dscp-outputq-threshold map:
     d1 :d2    0     1     2     3     4     5     6     7     8     9 
     ------------------------------------------------------------
      0 :    04-03 04-03 04-03 04-03 04-03 04-03 04-03 04-03 04-01 04-02 
      1 :    04-02 04-02 04-02 04-02 04-02 04-02 03-03 03-03 03-03 03-03 
      2 :    03-03 03-03 03-03 03-03 02-03 02-03 02-03 02-03 02-03 02-03 
      3 :    02-03 02-03 03-03 03-03 03-03 03-03 03-03 03-03 03-03 03-03 
      4 :    01-03 01-03 01-03 01-03 01-03 01-03 01-03 01-03 02-03 02-03 
      5 :    02-03 02-03 02-03 02-03 02-03 02-03 02-03 02-03 02-03 02-03 
      6 :    02-03 02-03 02-03 02-03 

00 - DSCP decimal 00
46 - DSCP EF - mapped to queue 1 threshold 3
10 - AF11 - mapped to Queue 4 threshold 2

eg

mls qos srr-queue output dscp-map queue 1 threshold 3 46
mls qos srr-queue output dscp-map queue 4 threshold 2 10


show ip rsvp interface
show ip rsvp interface detail
```
