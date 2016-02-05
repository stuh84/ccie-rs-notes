# Traffic-Shaping Concepts

* Shapers monitor rate data sent
* If rate exceeded, delays packets in shaping queue
* Packets then released over time
* Solves issue of SP dropping traffic (shape before it reaches them)
* Solves egress blocking (choke point in the network)

## Shaping Terminology

* Routers only send at clock rate
* Lower rates alternate between sending and silence
* For IOS Shaping, static time interval set (Tc)
* Number of bits per Tc interval calc'd to match shaping rate
* Bits per Tc = Bc (committed burst)
* Tc = Bc/CIR, measure in Ms
* Bc = Committed burst in bits
* Shapred rate - Bps
* Be - Excess burst, number of bits above Bc sent after period of inactivity

###  Shaping with excessive burst

* More than Bc bits in one or more ints if link has been quiet

## Underlying Shaping Mechanics

Tc = Bc/shape rate

* Token bucket model
* Bucket filled at start of Tc
* Each token is for 1 bit
* At start of Tc, shaping can release Bc bits

Actions in shaping bucket
* Refill at Tc start
* Spend tokens for packet forwarding

* Overflow - Only sents amount in interval
* Be exists to use overflow
 * Bc tokens added at each Tc still
 * If more tokens available than Bc, can send more at next Tc

## Generic Traffic Shaping

* GTS in older IOS
* Supported on most router ints
* Not used with flow switching
* Could shape all traffic leaving, or ACL'd for subset
* **traffic-shape rate** *shaped-rate [Bc] [Be] [buffer-limit]*
* Shaped rate - bps
* Bc and Be - bits
* Buffer is maximum queue buffer in bps
* Only shaped rate required, Bc and Be would then be quarter of shaped by default
* **traffic-shape group** *access-list-number shaped-rate* **{BC} {BE}**
* **show traffic-shape INTERFACE**
* **show traffic-shape STATISTICS**
* **show traffic-shape QUEUE**

## Class-Based Shaping

* Allows queueing tools for packets delayed by shaping
* Classification of packets for different shape rates
* Command in policy map is **shape [average | peak]** *mean-rate [[burst-size][excess-burst-size]]*
* Apply with service policy as normal
* Tc cannot be set directly
* Bc and Be dont need setting
* Calcaultes values of abvoe based on

|Variable| Rate le 320kbps|Rate gt 320kbps|
|--------|----------------|---------------|
|Bc|8000 bits|Bc = shape rate by Tc|
|Be|Be=Bc=8000 bits|Be=Bc|
|Tc|Tc=Bc/shape rate|25ms|

### Tuning for Voice

* Use small Tc so packets less delayed

```
policy-map queue-voip 
 class voip-packets
  priority 32
 class class-default
  fair-queue

policy-map shape-all
 class class-default
  shape average 96000 960
  service-policy queue-voip
```

* Shaping only when single packet above contract

### Configuring Shaping by Bandwidth percent

* Can be done as perents

Keep in mind following: -
* Shape percent ses bw of int or subint on which enabled
* Sub ints dont inherit physical bandwidth, 1544 default
* Bc and Be configured as Ms (bits sent at configured rate in time period)
* Tc set to configure Bc (in ms)
* **shape average 50 125 ms**
 * 50 is shaper rate, 125 ms is Bc
 * Ms required or command rejected

### CB Shaping at Peak Rate

* **shape peak MEAN-RATE**
* Calcs Bc, Be and Tc same way as average
* Refills Bc + Be for time intervals
* Logic means CB shaping can send Bc and Be per time period
* Shaping rate becomes
 * configured_rate x (1 + Be/Bc)
 * 64 (1 + 8000/8000) = 128

### Adaptive Shaping

* **shape adaptive MIN-RATE** under shape command in class config

# Policing Concepts and Config

* Different internal process than older IOS policer

## CB Policing Concepts

* Enabled on ingress or egress
* Monitors bit rate of combined packets
* When above, takes action
* Actions are
 * Drop
 * **set-dscp-transmit**
 * **set-prec-transmit**
 * **set-qos-transmit**
 * **set-clp-transmit**
 * **transmit**
* Packets conform, exceed or violate

### Single Rate, Two Colour (One Bucket)

* No excess burst
* Conform and exceed only
* Single token bucket
* Over time, policer refils bucket according to policing rate
* Token is a byte, so for 96 kbps over a second, bucket filled with 12,000 tokens (12KBps)
* Not refilled on time interval
* Instead, reacts to arrival of packet, replenishes prorate number of tokens into bucket
* Defined by: -
 * ((Current_packet_arrival_time - Previous_packet_arrival_time) x Police_rate) / 8
* Policer then decides if newly arrived packets conform or exceeding contract
* Number of bytes (Xp)
* Number of tokens in bucket (Xb)

|Category|Requirements|Tokens drained from bucket|
|--------|------------|--------------------------|
|Conform|If Xp <= Xb|Xp tokens|
|Exceed|If Xp>Xb|None|

* Over time, tokens back into packet, so some packets conform
* After bit rate lowers, all conform

### Single-Rate, Three Colour (Two Buckets)

* First bucket filled like single
* If Bc bucket overlows, these fill Be
* After filling buckets, another option
 * If Xbe>=Xp>Xbc, tokens from Be bucket

### Two-Rate, Three Colour Policer

* Second rate is PIR (Peak Information Rate)
* Packets under CIR conform
* Packets below PIR exceed
* Beyond is violating
* Sustained excess buristing allowed in this model
* Be bucket not relying on spillage
* If one bucket was 128kbps, other 256kbps, in 0.1s 1600 tokens into BC, 3200 into Be

Consumed like so: -

|Category|Requirements|Tokens Drained from Bucket|
|--------|------------|--------------------------|
|Conform|Xp <= Xbc| Xp tokens from Bc and Xp from Be|
|Exceed| Xbc < Xp <= Xbe|Xp tokens from Be|
|Violate|Xp > Xbc and Xp > Xbe|None|

## CB Policing Config

* **police** *bps burst-normal burst-max* **conform-action** *action* **exceed-action** *action* **[violate-action** *action*

**Single Rate Three Colour**
```
policy-map police-all
 class class-default
  police cir 96000 bc 12000 be 6000 conform-action transmit exceed-action set-dscp-transmit 0 violate-action drop
```

**Policing subset**
```
class-map match-all match-web
 match protocol http

policy-map police-web
 class match-web
  police cir 80000 bc 10000 bc 5000 conform-action transmit exceed-action transmit violate-drop
 class class-default
  police cir 16000 bc 2000 be 1000 conform-action transmit exceed-action set-dscp-transmit 0 violate-action set-dscp-transmit 0
```

### CB Policing defaults for Bc and Be

* If Bc not config'd, equivalent in bytes for 1/4 send at police rate
 * Bc = ((CIR * 0.25s)/8 bits = CIR/32

Default Be based on policy: -

|Type of config|Defaults|
|--------------|--------|
|Single rate, two colour|Bc = CIR/32, Be = 0|
|Single rate, three colour|Bc = CIR/32, Be = Bc|
|Dual Rate three colour|Bc=Cir/32, Be = PIR/32|

### Config for Dual Rate

* **police {cir CIR} [bc CONFORM-BURST] {pir PIR} [be PEAK-BURST] [conform-action ACTION exceed-action ACTION [violate-action ACTION]]**
* Requires PIR and CIR to be confirmed, can set with only these two

### Multi action policing

* Mark multiple fields in same packet
* Slightly different syntax, places in policing subconfig mode

```
policy-map testpol1
 class class-default
  police 128000 256000
   conform-actionb transmit
   exceed-action transmit
   violate-action set-dscp-transmit
   violate-action set-frde-transmit
```

### Policing by percentage

* Bc and Be config'd as number of Ms
* IOS calcs Bc and Be based on how many bits sent in that many ms

Dual rate example:- 

```
policy-map test-pol6
 class class-default
  police cir percent 25 bc 500 ms pir percent 50 be 500 ms conform transmit exceed transmit violate drop
```

## Committed Access Rate

* Single rate two colour
* Set rate in bps, Bc and Be bytes
* CAR differs from CB policing
* Uses rate-limit command
* Cascaded/nested rate limits (multiple on interface)
* Does support Be, but no violate categeory
* When Be config'd, internal logic for monitoring differs from CB
* **rate-limit {input | output| [access-group [rate-limit]** *acl-index] bps burst-normal burst-max* **conform-action** *action* **exceed-action** *action*

* Can use normal ACL or rate limit ACL, which can match MPLS EXP, IPP or MAC. For other fields, ACL

```
int Se0/0
 rate-limit input 496000 62000 62000 conform-action continue exceed-action drop
 rate-limit input access-group 101 400000 50000 50000 conform-action transmit exceed-action drop
 rate-limit input access-group 102 160000 20000 20000 conform-action transmit exceed-action drop
 rate-limit input access-group 101 200000 25000 25000 conform-action transmit exceed-action drop
```
* Continue means go through and potentially match others

# Hierarchical Queuing Framework

* MQC mechanism used for HQC for queueing and shaping
* Local engine to support QoS features
* Tree structure built using policy maps
* When data through interface using HQF, data classified so it traveres tree branches
* Data arrives at top of tree, classified on one of leaves

```
policy-map class
 class c1
  bandwidth 14
 class c2
  bandwidth 18

policy-map map1
 class class-default
  shape average 64000
  service-policy class

policy-map map2
 class class-default
  shape average 96000

map-class frame-relay fr1
 service-policy output map1

map-class frame fr2
 service-policy output map2

int Se4/1
 encapsulation frame-relay
 frame-relay interface-dlci 16
  class fr1
 frame-relay interface-dlci 17
  class fr2
```

* Fast deployment of QoS queueing and shaping
* Consisitent queueing behaviour with common MQC
* HQF supports distributed and non-distributed implementations
* Includes levels of scheduling and support for integrated CB shaping and queueing
* Placement of hierarchical policies and queueing features at every level of structure
 * means can apply queueing to any traffic class in parent or child level of policy
 * discrete service levels for different sessions/subscribers

```
policy-map class
 class c1
  bandwidth 14
 class c2
  bandwidth 18

policy-map map1
 policy-map child
  class child-c1
   bandwidth 400
  class child-c2
   bandwidth 400

policy-map parent
 class parent-c1
  bandwidth 1000
  service-policy child
 class parent-c2
  bandwidth
  service-policy child
```

## Flow-based Fair Queueing in Class-Default

* Rather than WFQ, flow based used
* Flow queues scheduled equally instead of weight based (ipp or dscp)

## Default Queueing Implementation for Class-Default

* Default FIFO when no policy-map
* Can use bandwidth, fair queue or service policy to change

### Class-Default and Bandwidth

* Bandwidth assigned is unused int bandwidth not in other user-defined classes
* Minimum 1% of int by default

### Default Queuing for Shape Class

* Default FIFO rather than WFQ for Shape Class

### Policy Map and Int Bandwidth

* Can reserve 100% of int bandwidth
* If no explciti bandwidth guarantee in class-default, can assign max of 99% of int bandwidth
* Error appears when going to 12.4(20)T if 100% to user classes

## Per-Flow Queue Limit in Fair Queue

* When fair queue enabled, default per flow queue limit is 1/4 of class queue limit
* If not enabled in class, default is 16 packets

## Oversubscription Support for Multiple Policies on logical interfaces

* When shaping policy added to multiple logical ints (including sub int), and sum of all above physical in tbandwidth
 * Congestion at physical gives back pressure to each logical int policy
 * Each policy reduces output rate to its fair share of int bandwidth

## Shaping on GRE Tunnel

* In HQF, shapping after Encap with hierarchical service policy
* When shape in parent policy applied to tunnel, can use class-default only
* Cannot configure user defined class in parent
* Can apply service policy with queueing at tunnel/virtual int and service policy with queueing at physical, but not at same time

## Nested Policy and Reference Bandwidth for Child Policy

* When nested policy config'd with child queueing policy under parent
 * Ref bw for child taken from following
  * Minimum (parent shaper rate, parent class's implicit/explicit bandwidth guarantee)
 * when not defined for parent, int bandwidth devices among all parents as implicit bandwidth guarantee

## Handling congestion on int with policy map

* If int configurured with policy map full of heavy traffic, 
* Implicitly defined polcier allows traffic as defined in bandwidth statement at each class
* Policer activated whenevr traffic congestion on interface

# QoS Troubleshooting and COmmands

* Lack of planning for qos requirements
* Failure to track changes in apps and traffic
* Lack of good documentation

## Troubleshooting Slow App Response

* Ensure enough bandwidth for application
* Check bw, latency and drop requirements
* IP SLA can help, **show ip sla statistics**
* **show policy-map**
* **show class-map**
* **show policy-map interface**

## Troubleshooting Voice and Video

* **show mls qos**
* **show policy-map**
* **show class-map**
* **show policy-map interface**
* IP SLA

Switch techniques
* **show mls qos input-queue** ingress
* **show mls qos INTERFACE queuing** egress
* **show mls qos maps cos-input-q** - mapping for CoS to Queue
* **show mls qos maps cos-output-q**
* **show mls qos maps cos-dscp**
* As above but dscp-cos

Rotuers also have
* **show mls qos maps** - COS and DSCP mappings
* If traffic shaping enabled, tune Tc to 10ms
* SP may have different service levels, check mapping

## Other tips

* Remove config if something like a printer on an interface that used to be a user (eg AutoQoS for voice)
* default interface INTERFACE

## Approach to resolve

|Problem|Approach|Helpful IOS Commands|
|-------|--------|--------------------|
|Troubleshooting QoS misconfig |Verify QoS is enabled, class map config, policy map config, and service policy operation |show mls qos, show class-map, show policy-map, show policy-map interface|
|Possible switch QoS misconfig |show commands to determine how input/egress queueing configured |show mls qos input-queue, show mls qos interface interface queueing, show mls qos maps cos-input-q, show mls qos maps cos-output-q, show mls qos maps cos-dscp, show mls qos maps dscp-cos|
Possible router Qos |show commands to determine how queueing configured |show mls qos maps, show traffic-shape|
 


