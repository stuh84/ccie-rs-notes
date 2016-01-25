# Basics and Evolution

* IETF Draft exists (draft-savage-eigrp)
* In Quagga
* IP Protocol 88
* RTP for uni/m'cast delivery
* Hello and hold times
* Full updates on neighbour up
* Partial after
* MD5 or SHA auth
* Mask included in each route (VLSM/classless)
* Supports route tags
* Next hop field
* Summarize anywhere
* Multiprotocol (v4, v6)

## IGRP

* Cisco alternative to RIP v1
* Avoids hop count limits (up to 255)
* Composite metric
* 90s update period
* Unequal cost load sharing
* Interoperated with v4, ISO, CLNP, IPX, AppleTalk
* Request packet out all IGRP interfaces at start
* Sanity check on rx'd updates
* Vefifies source of packet belongs to subnet received on
* Classful DV
* Periodic full info broadcast
* Split-horizon
* Triggered updates
* Invalid After, holddown, flushed after
* Advertises summaries at boundaries

## IGRP to EIGRP

**IGRP Weaknesses**
* Full updates
* No VLSM
* Slow convergence
* Bad loop prevention

**EIGRP Advantages**
* Decoupled updates and adjacency building
* Event driven than timer driven
* Hello protocol
* RTP
* Allows complete routing info exchange, then changes only after (RTP verified)
* Loop free paths found with FC (packets not forwarded to neighbour that coudl loop)

**Diffusing Computation**
* FC could cause neighbour with lease cost path being ineligible
* Neighbours queried for their best path after topology change
* Replies with udpated distances
* Some neighbours respond straight away
* Some propagate query further

**DUAL**
* For multiple changes in one computation
* Controls computation run
* Processes replies, and how/when to insert routing table info

* Hop count limitation of 100 by default, 255 max **metric maximum-hops**
* Internal and external distance, 90, 170, **distance eigrp <internal> <external>**

