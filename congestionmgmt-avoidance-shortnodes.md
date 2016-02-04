# Cisco Router Queuing Concepts

## Software and Hardware

* Queues in queueing tools software
* Queue scheduler picks from software to go into hardward FIFO
 * Tx queue or Tx ring

Hardware queues provide
* Next packet from hw queue encoded with software interrupt to CPU
* FIFO logic
* No effect from IOS queueing tools
* Shrinks to small length when queueing tool exists
 * Means software queues have more control

* Can only change length of Tx
 * **tx-ring-limit 1**
* **show controllers** - tx_limited is Tx_ring

## Queuing on Ints versus Subifs and VCs

* Shaping function queues packets, by default in fifo queue

# Comparing Queueing Tools

* Classification - looks at packet headers to choose queue
* Drop policy - Which packets to drop when filling
* Scheduling - Which packet next
* Maximum number of queues
* Maximum queue length

## CBWFQ and LLQ

* Roots in PQ, CQ and WQ
* CBWFQ reserves bandwidth for each queue
* WFQ in default queue option
* LLQ - priority queue, with bw limit
* **bandwidth** in CBWFQ
* **priority** in LLQ
* Supports 64 queues/classes
* Max queue length can be changed
 * Hardware dictates
* Both have class-default, exists even if not config'd

### CBWFQ features and config

* If all queues with large number of packets, get percentage of banwidth as implied by config
 * If some empty, distributed proportionally to others

Key features
* Classify on MQC
* Drop policy is Tail drop or WRED, decide per queue
* Scheduling inside queue - FIFO on 63, FIFO or WFQ on class-default
* Scheduling among all - Percentages of guaranteed bandwidth to each queue
* Cisco 7500 can have FIFO or WFQ in all queues

Commands
* **bandwidth KBPS/percent PERCENTAGE** - Literal or percentage for class
* **bandwidth { remanining percent PERCENT**} - Percentage of remaining
* **queue-limit LIMIT** - Max queue length
* **fair-queue [queue-limit VALUE]** - WFQ in class

```
class-map match-all voip-rtp
 match ip rtp 16384 16383

policy-map queue-voip
 class voip-rtp
  bandwidth 64
 class class-default
  fair-queue

int Se0/0
 bandwidth 128
 service-policy output queue-voip
```

### Defining and Limiting CBWFQ Bandwidth

* Pre-IOS 15 checked to make sure too much bandwidth not allocated
 * Checked whgen adding **service-policy output**, rejected if too much
 * Based on bandwidth and max-reserved-bandwidth
* In IOS 15, no **max-reserved-bandwidth**
 * Allocated based on **int-bw**
* **bandwidth percent** sets class's reserved based on **int-bw**
* If bandwidth remaining percent, percentage of class's reserved bandwidth (i.e. frm bandwidth percent)
* Explicit bandwidth - sum of values lt or eq to max-res x int-bw
* percent - lt or eq to max-res
* Remaining percent - lt or eq to 100

### LLQ

* Packets scheduled first always
* Sometimes called PQ
* Queue starvation doesnt happen, as has bandwidth value
 * Guaranteed minimum and policed max
* Enabled with **priority { bandwidth-kbps | percent PERCENTAGE } [burst]
 * Burst is 20 percent by default

```
policy-map queue-on-dscp
 class dscp-ef
  priority 58
 class dscp-af41
  bandwidth 22
 class dscp-af21
  bandwidth 20

int Se0/0
 max-reserved-bandwidth 85
 servcie-policy output queue-on-dscp
```

### Defining and limiting LLQ bandwidth

* Explicit and percentage values available in same policy map
* Limits amount of bandwidth in LLQ policy map
 * Actual bandwidth from LLQ and non-LLQ not exceed max-res x int-bw
* LLQ bandwidth subtracted from remaining-bandwidth, means rest of classes can total 100

### Misc CBWFQ/LLQ topics

* When bandwidth command under class-deault, class reserved that minimum
 * If none set, whatever is unassigned
* Congestion = hardware queue full in IOS

|Feature|CBWFQ|LLQ|
|-------|-----|---|
|Includes strict priority queue|No|Yes|
|Policies PQ|No|Yes|
|Reserves bw per queue|Yes|Yes|
|Classify on flows|Yes|Yes|
|Support RSVP|Yes|Yes|
|Max Queues|64|64|

# WRED

* Monitors queue length
* Discards percentage of packets in queue
* When queue longer, discards more
* Measures average queue depth, compared to min and max queue threshold
* If Average < min - No packets dropped, "NO DROP"
* If Min < Average < Max - Some dropped, "Random Drop"
* Average > Max, "Full Drop"
 * Not same as tail drop, macx threshold might not be full queue size
* MPD - Mark probability denominator
 * Discard percentage at max threshold is 1/MPD

Example
* Min threshold 20, max 40, MPD of 10 (IPP 0 defaults)
* Between min and max, random packets dropped, increases until max thresh, where 10% packets discarded
* Over max threshold, 100% discard

## How WRED Weights packets

* Preferences to IPP or DSCP values
* Different traffic profiles for these
* Each profile has min, max and MPD
* Higher MPD means lower drops during time between thresholds

Some defaults for DSCP values

|DSCP|Min Threshold|Max Threshold|MPD|1/MPD|
|----|-------------|-------------|---|-----|
|AFx1|33|40|10|10%|
|AFx2|28|40|10|10%|
|AFx3|24|40|10|10%|
|EF|37|40|10|10%|

## Config

* Must be configured along with queue
* Can be configured on physical interface, non LLQ in CBWFQ and ATM VC
 * On interface, all other queue disabled, single FIFO
* **random-detect** - Enables WRED, use **dscp-based**
* Change wred config from default wred profile with
 * **random-detect precedence** *precedence min-thresh max-thresh [mpd]*
 * **random-detect dscp** *dscp min-thresh max-thresh [mpd]*
* Change calc of average queue depth with the exponential weight constant, lower means old average small part of calc (quicker changing average)
 * **random-detect exponential-weight-constant CONSTANT**

# MDRR

* on 12000 routers only
 * They dont support CBWFQ and LLQ
* Classifes into seven round-robin queues (0-6) and one additional priority
* When no packets in priority, RR for each queue once per cycle
* When in priority, strict or alternate
 * Strict serves whatevers in queue, can get more than config'd bandwidth due to being serviced more than once per cycle
 * Alternate serves as such, 0, P, 1, P, 2, P etc, can cause jitter/latency
* Quantum Value and Deficit
* MDRR removes packets from queue until QV for that queue removed
 * QV - number of bytes (like byte count in CQ)
 * Repeats process for every queue sequentially
* Any extra bytes are the deficit
 * Can't take half a packet, if QV was 2000 and packet 1500, would end up taking two packets of 1500 and be 1000 in deficit
* Deficit feature means over time, each queue a guranteed bandwidth
 * based upon QV for Queue X/Sum of all QVs

# LAN Switch Congestion Management and Avoidance

## Switch ingress qaueueing

* 3560 have two ingress queues
 * one can be PQ
* Weighted tali drop sets discard thresholds per queue
* SRR schedules packets
* Controls rates packets sent from ingress queues to switch fabric
 * In shared mode, bandwidth shared between two queues according to weights
 * Bw guaranteed, not limited
* If one queue empty and other not, queue allowed all bandwidth
* SRR use relative weights rather than absoltues (ratios), like CBWFQ with percentages

Configure queueing like: -
* Which traffic to put into queue (Cos 5 default queue 2, all others queue 1)
 * Can be done with DSCP
* Priority traffic?
* Bandwidth and buffer space per queue
* Whether default WTD appropriate

## Creating priority queue

* **mls qos srr-queue input priority-queue ID bandwidth WEIGHT** - Weight is percentage of links bandwidth 
* Packets in priority served at moment they arrive, AFTER current frame finished in non-priority

```
mls qos srr-queue input cos-map queue 2 6
mls qos srr-queue input priority-queue 2 bandwidth 20
mls qos srr-queue input buffers PERCENT1 PERCENT2
```

* Defaults - 90% buffer to queue 1, 10% to queue 2
* Frequency of scheduling from buffers is **mls qos srr-queue input bandwidth WEIGHT1 WEIGHT2**
 * Default 4 and 4, values relative weightings

Default ingress settings
* Queue 2 priority
* Queue 2 10% bandwidth 
* Cos 5 in queue 2

## 3560 Congestion Avoidance

* WTD on by default when QoS on switch
* Three thresholds per queue (CoS based) for tail drop when queue reaches paricular pecentage
* Default is drop at 100% of queue
 * Makes sense for priority queue (i.e. UDP traffic)
* Could do A threshold 1 drops traffic for values of 0-3 when queue reaches 40 percent
 * Threshold 2 drops traffic with cos 4 and 5 at 60% full
 * Threshold 3 for 6 and 7 at 100% full
* Threshold 3 always has to be 100%, cannot be changed
* WTD configured separately from queues

* ** mls qos srr-queue input threshold** *queue-id threshold-percent1 threshold-percent2*
* If trusting CoS, map with **mls qos srr-queue input cos-map threshold** *threshold-id cos1 ... cos 8*
* Replace DSCP in above command for dscp

```
mls qos srr-queue input buffers 80 20
mls qos srr-queue input bandwidth 3 1
mls qos srr-queue input threshold 1 40 60
mls qos srr-queue input cos-map threshold 1 0 1 2 3
mls qos srr-queue input cos-map threshold 2 4 5
mls qos srr-queue input cos-map threshold 3 6 7
```

## 3560 Egress Queueing

* Similar to ingress, but four ints per queue
* Can configure which CoS and DSCP to which queue
* Weights of queues
* Drop thresholds
* PQ must be queue 1 if used
* WTD used for queues, config'd like ingress
* Commands on this at interface level, not global

* 3560 can shape to slow egress traffic
* Can prevent some DoS attacks
* Can implemented subrate speed for Metro-E
* Internal QoS decisions made on internal DSCP setting
 * Determined when frame forwarded

When frame givien internal DSCP and egress int, following logic for queue placement: -

1. Frames internal DSCP compared to global DSCP-to-CoS map
2. Per int CoS-to-Queue map

* Two versions of SRR
 * Shared
 * Shaped
* Shaped rate limits to not exceed config'd percent of links bandwidth

* **srr-queue bandwidth share** *weight1 weight2 weight3 weight4*
* **srr-queue bandwidth shape** *weight1 weight2 weight3 weight4*
* Default weights 25
 * If all four contained frames, serviced equally
* When not full, shared keeps servicing one queue to bandwidth
 * Shaped waits to service queue, shapes that queue (only gets config'd percentage)

* If queue 1 PQ and empty, others serviced
* If frames arrive in PQ, services current frame
 * Scheduler limits bandwidth for PQ
 * Limits excess above queue, rather than discarding
* If PQ still frames to send, but other queues empty, same as before

* Mapping DSCP or COS to queus done in global config, same way
* Each interface belongs to one of two egress queue sets
* Buffer and WTD configs done in global config for each queue set
* Bandwidth weights, shaped or shared, and priority done on interface

```
mls qos queue-set output 1 biuffers 40 20 30 10
mls qos queue-set output 1 threshold 2 40 60 100 100

int Fa0/1
 queue-set 1
 srr-queue bandwidth share 10 10 1 1
 srr-queue bandwidth shape 10 0 20 20
 priority-queue out
```

# RSVP

* Each router on path reserves bandwidth and requested QoS during flow duration
* Reservations flow-by-flow basis
 * each own reservation
* Unidirectional

## Process Overview

* Flow based on dest IP, protocol ID, dest port
* Reservation made from terminating to original gateway, and another in opposite direction

* PATH messages from originator along to terminating gw
* RESVs sent in response
* PATHs sent back from terminator to originator
* RESVs in response
* ResvConfs sent in response from terminator

* Path message contains IP of previous hop (PHOP) so returns go in same path
 * Also has bandwidth and QoS needs of traffic

1. GW1 -> RSVP PATH to dest ip
2. Next Hop -> Records PHOP, forwards PATH, its IP now previous HOP
3. If non-RSVP router, just forwards
4. Dest Router -> Gets PATH, replies with RESV to PHOP
 a. RESV requests needed QoS, if router can't guarantee it, returns error and discards RSVP
5. Dest Router -> PATH to GW1
6. Second router gets RESV, flow created, forwards to PHOP in PATH message earlier
7. When GW1 gets RESV, reservation succeeded. Completes for PATH from terminator, ResvConf confirms path in both directions

* Data transmission delayed during above
* Forwards on ResvConf rx'd
* Periodic fresh messages along path, adjusts to changes

## Config

* Allocate bandwidth per flow
* Bandwidth per interface allowed
* Per interface with **ip rsvp bandwidth** *total-kbps single-flow-kbps*
 * Default total of 75 percent of int-bw if none set
 * No flow means can use full amount
* DSCP value for RSVP messages - **ip rsvp signalling dscp** *dscp-value*
* Does not need to be on all routers


## RSVP for voice calls

* RSVP has own set of queues
 * puts reserved traffic in these bydefault
 * Low weights, not prioritized
 * WFQ for QoS
* Disabled RSVPs WFQ with **ip rsvp resource-provider none** when using LLQ with CBWFQ
* By default, RSVP attempts to process all packets
 * Turn off with **ip rsvp data-packet classification none**
* LLQ and CBWFQ then config'd normal
* RSVP reserves for voice calls
* Gateways QoS places voice traffic into priority queue

* When using LLQ, priority queue size includes L2 overhead
* RSVPs bandwidth does not include L2 overhead
 * Set RSVP bandwidth equal to PQ minus L2 overhead

```
int Se0/1/0
 ip rsvp bandwidth 128 64
 ip rsvp signalling dscp 40
 ip rsvp resource-provider none
 ip rsvp data-packet classification none
 service-policy output llq
```

* **show ip rsvp interface**
* **show ip rsvp interface detail**

