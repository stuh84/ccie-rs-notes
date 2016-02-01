* IOS/IEC standard 10587:2002, initially for OSI networks
* Dijkstra/SPF
* Low dependence on ISO protocols
* Uses NSAP for router identity, area membership, adj
* Provides L1 and L2 routing
* Messages in data-link frames
* Adj and address info in TLVs
* For a new AF, define new TLV with existing topology
* RFC 1195 for IP routing along with OSI in single instance (Integrated IS-IS)

# OSI Network Layer and addressing

* ISO created protocol specs for each OSI layer
* Many used in telecoms equipment and aviation
* End System (ES) - Host
* Intermediate System - Router
* System - Network node
* Circuit - Interface
* Domain - AS

* ES-ES uses connectionless and connection mode network layer comms
* Connectionless like IP (datagram service without session establishments)
* L3 proto between ES is CLNP (Connectionless Network Protocol)
 * ISO/IEC8473-1:1998
 * Services by CLNP are CLNS
* Connection orientated adapted from X.25
* NSAP address for both modes, IOS/IEC8348
* NSAP for entire network node
* No interface subnets

* Basic address format is IDP (Initial Domain Part) and DSP (Domain Specific Part)
* Variable format and length (application dependent)
* Address has variable length

**IDP and AFI**
* Two fields, Authority and Format Identifer (AFI) and Initial Domain Identifier (IDI)
* AFI is format of address fields
* IDI variable length based on AFI
* AFI and IDI indicate routing domain for node

**DSP**
* Depends on address format
* Variable length HO-DSP (High Order DSP)
 * Area/part node is in
* Subfields of this
 * System ID - unique to node, 1 to 8 octets long, but usually fixed at 6
 * SEL (NSAP Selector/NSELF), 1 octet, service/network layer on node that should process packet
* 49 is locally defined
* Usually, AFI of 49, length and meaning of HO-DSP admi ndefined
* Minimum NSAP is 8 octets, with AFI, Sys ID and SEL. Max is 20
* If SEL 0, entire address identifiers dest node
* NSAP with SEL 0 is Network Entity Title (NET)
* Dots or no dots doesnt matter
 * Cisco sets to 49.XXXX.XXXX.XXXX.XXXX.00 by default
* L2 of interface is SNAP (Sub Network Point Of Attachment)
* IS enumerates interfaces with local 1 octet number, local circuit ID, increments by 1 for every int, begins at 0 for cisco

## OSI Routing Levels

* 0 - ES-ES or ES-IS
* 1 - ES in single area
* 2 - ES in different areas
* 3 - ES in different domains

* Period hellos from ES to IS to discover nearest IS
 * ESH for ES
 * ISH for IS

* L1 - Intra area routing
* Collects list of ESs attached, advertises list to each other

* L2 - Inter area
* No list of connected ES
* Area prefixes exchanged to learn how to reach areas
* If IS sees packet of ES in different area, forwards to closest L2 capable IS
* Packet forwarded by L2-capable IS until destination

* L1 is routing by system ID
* L2 routing by area

* L2 is backbone of domain

* L3 similar to BGP. As BGP-MP exists, and can carry NSAP, OSI IDRP (Interdomain Routing Protocol) replaced with BGP

## Metrics, Levels, Adj

* Default - Required by all IS-IS imps, usually bandwidth, higher worse
* Delay - Transit delay on link
* Expense - monetary
* Error - Residual bit error
* All above calc 4 different SPF trees
* Default usually only one supported
* Default metric 10, no matter bandwidth
* **isis metric** *metric [level]* - Per int

* Original spec defined link and metric to be 6 bytes wide
* Range of 1-63, path metrics 10 bits wide (1-1023)
* Wide metrics expand, 24 bits per int, 32 per path

* Separate LSDB per routing level
* For each level, router originates LSP (Link State PDU)
* Similar to LSU
* L1 and L2 LSPs describe adj at each level
* LSPs never leak between L1 and L2 databases
 * Can inject routes in a controlled way between the two

## IS-IS Packet Types

* Hello
* LSP
* CSNP
* PSNP

### Hello

* IIH
* Separate sent for L1 and L2 adj on b'cast
* L1L2 (or P2P) hello on P2P links
* 10s hello time default, 1-65535 range
* **isis hello-interval seconds [level]**
* Hold down multiplier of hello, default 3
* **isis hello-multiplier multiplier [level]**
* Don't need to match between neighbours
* DIS is one third of config'd timers
 * 3 1/3s hello
 * 10 hold
 * Detects DIS outage quicker

### LSPs

* Advertising routing info
* Smallest standlone element of LSDB is entire LSP
* No different types, distinct TLVs inside LSP instead

Identified by: -
* System ID of originator (6 octets)
* Pseudonode ID, differentiates between LSP describing router, and LSP for multiaccess networks in which router a DIS (1 octet)
* LSP number - Fragment number of LSP (1 octet)
* Triplet is LSPID
* For LSPs describing router themselves, PSID is 0

* Seq nuymber is 32 bit
* Start 0X00000001
* End 0xFFFFFFFF
* Modification increments
* Highest seq most recent
* Similar to OSPF, but no wrapover
* Originating router would need to turn off for LSP to expire, or change Sys ID
* Would take 136 years for an LSP every second changing to reach this

* LSP remaining lifetime
* 1200 seconds default (20m)
* IS-IS routers refresh LSPs every 15m
* LSP body deleted after lifetime
* Advertises empty LSP with lifetime 0 (LSP purge)
* No flushing yet, ZeroAgeLifetime of 60s first
* Ensures header retains until all neighbours have seen it
* Ciscos hold empty LSP header for another 20m

* MTU limits payload size
* Frag required
* Same Sys and PSID
* Frag number increments, starts at 0
* Frag only by originating router
 * Means end to end MTU must be no smaller than routers into network

Inside LSP: -
* Adj to neighbour routers or networks
* Intra/Inter Area prefixes
* External prefixes
* Address info about all in LSP of each router connected to that network

* Topology info about network and connected routers from Pseudonode LSP
* Gen'd by DIS on ma network
* IS-IS router on a level gen's 1 LSP describing itself and topology info, plus a PS LSP for each network its a DIS in

* Unique ID of LSP as a whole
* Can be flooded, requested, ack'd, refreshed, aged and flushed
* When any topology or addressing change, LSP needs to be regend
* Very few LSPs rather than multiple LSAs, so few required to refresh

* All topology/address info in TLVs
* Routers process those they know, ignore those it doesnt

### CSNP and PSNP

* Syncs LDPs
* Seq number refers to range of LSPID values about set of LSPs that packets carry inf about
* Not seq number of individual LSPs

* CSNPs like DBDs
* Complete list of LSPs in senders LSDB
* Rx router compares own LSDB to CSNP and acts (floods missing, or requests missing)

* Multiple CSNPs sent if filled MTU
* Individual LSPIDs advertised sorted as integer numbers
* Each CSNP has start LSPID and End LSPID
* Full range starts with 0000.0000.0000.00-00 (first 3 octets are sys ID, next is PSID, octet after dash is LSPID)
* Ends with FFFF.FFFF.FFFF.FF-FF
* If more than one, ends as last in CSNP

* PSNP similar to LSR and LSA
* Requests an LSP or acks arrival
* Can req or ack multiple LSPs
* No start or end
* On P2P, req's and acks
* On b'cast, PSNP request, ack'd with CSNP by DIS

* Native on b'cast and P2P
* Try to use P2P instead of hub-spoke or p2mp

* Broadcast links known as multiaccess links in IS-IS
* Does not care about data link capability
* Just wants mulitple neighbours to be reached over same interface

**Three Adj States**
* Down - No IIH from neighbour
* Initializing - IIH from neighbour, not certan neighbours receiving this routers IIH
* Up - IIH form nei, certain neighbour receiving own IIH

### Operation over P2P

* Expects single neighbor
* Brings up adj
* Syncs LSDB

* Original spec assumed adj possible moment P2P IIH rx'd
* No bidi check
* RFC 3373 (then 5303) added three way handshake

* Each router has LCIDs
* LCID on P2P appear only in IIH, detects change in identity on other end
* On b'cast, LCID is PSID if router is DIS on interface
* LCID unique only for b'cast
* LCIDs of b'cast and p2p do not clash

* 256 LCIDS were available, three way added Extended LCID (4 octets long)
* Auto assigned, used only for three way handshake

Adj State TLV has following fields: -
* Adj Three Way State - State as seen by sender
* Extended LCID - Sending router ints LCID
* Neighbour Sys ID - Received IIH sys ID
* Neighbour Extended LCID

Early implementation: -
1. If router receives IIH with Three way state down, Router A hears B, B might not hear A. State send of Initializing in IIH
2. B sees state initializing, knows own IIH successful, sends with state up
3. When A sees IIH with Up, bidi confirmed, seends IIH with up

Default o Cisco, configured with **isis three-way-handshake cisco** int

* If on FR or ATM VC, VC could be switched towards another neighbour with sub int going down
* Rx and Tx could be independent

**Other three fields help, as such: -**
* If no IIH from B, A sending IIH with Extended LCID only, set to Router As outgoing int ID, no other fields present
* When IIH arrives from B and accepted, A puts B's sys ID in neighbour SysID field, Bs Extended LCID in Neighbour Ext LCID field, sent back to B

IIH carrying three way adj state TLV accepted only if one of following: -
* Neighbour Sys ID and Ext LCID not present
* Nei Sys ID matches own Sys ID, Nei Ext LCID mathes own int ID
* If not, IIH silently dropped

* **isis three-way-handshake ietf** - Backwards compat with cisco

* After adj, routers sync LSDB
* Marks all LSPs for flooding over p2p link
* CSNPs sent to each other
* CSNPs could be exchanged before LSP transmission takes place (periodic scheduling)
* If CSNP has LSP that marked for sending, router unmarks LSP
* Only missing sent
* Requests sent for any unknown with PSNP

* Above unnecessary as all LSPs flooded without CSNP and PSNP aid on initilization
* Every LSP sent over P2P link must be ack'd (even during initial sync)
* Each sent LSP acked by corresponding PSNP from neighbour
* If neighbour also sends periodic CSNPs, also valid ACKs
* Some vendors do do this on P2Ps
* Ciscos dont, enable with **isis csnp-interval interval [level]**

### Operation over B'Cast

* Adj, sync db, keep syncd
* Packets encaped in 802.2 LLCs with DSAP and SSP set to 0xFE
* All L1 packets multicasted to 0180.c200.0014, L2 to 0180.c200.0015

* IIH detects neighbours
* Lists neighbouring routers on b'cast int in IIH it sends out
* If router sees own SNPA in IIH, bidi vis and adj to up
* If not, adj in initializing state

Every b'cast has DIS. No backup dis. Elected as such: -
* Router with highest int priority
* Highest SNPA
* If SNPAs not comparable, highest system ID (use on FR and ATM phy and mp)
* On above, each router sees itself and neighbour on same VC, ID'd by same DLCI/VPI/VCI, multiple VCs terminated on interface, ambigirous which VCID for SNPA

* Priority in range 0-127
* **isis priority priority [level]**
* 0 excludes router from DIS
* DIS election preemptive
* Performed on each rx'd IIH
* All routers on b'cast segement fully adj
* Every router sends LSP on b'cast link
* All other can accept

**DIS Purpose**
* Help routers on segment to sync
* Represent segment in LSDB as standalone object - pseudonode
* Sync where DIS creates and sends CSNP in regular interval (10s by default)
* CSNP lists all LSPs in DIS's LSDB
* Results of receiving this could be
 * Same LSP and LSPID and seq, no action
 * Does not have LSP with LSPID or seq on DIS higher, router sends PSNP
 * Router has same LSP with same LSPID, higher Seq. Router floods LSP on network immediately
* Reference point of LSPs
* No explicit ack of LSPs, just seen in next CSNP from DIS
* PSNPs only for LSP request
* Represents b'cast in LSDB for simpler topology model
 * Each router claims connectivity to b'vast netwtork
 * Network claims connectivity to each router

* DIS originates PS LSP
* Each LSP has SysID, PSID and LSP frag number
* For network LSPs, Sys ID is of DIS< PSID is LCID of DIS's int on network
* Shorter hello and hold (one third of config'd)
* Fails quicker
* If DIS fails, router elected in place
* ONly thing needed is to replace PS LSP with new one, remaining routers update LSPs to point towards new PSLSP

## Areas

* Routers in single area only due to one NSAP
* Three NSAps in single IS-IS possible, providing SysID identical, and only AreaID different
* Only LSDB mantainained, with config'd ares merged (useful for splitting/joining or renumbering areas)
* Add second NSAP with new AreaID, and old NSAP removed, no adj flaps

* During stable operation, should be one NSAP per IS-IS process
* Entire high order part of NSAP up to SysID is area ID
* Makes sure all routers uses same format

* Different LSDBs for L1 and L2

* Each L1 router advertises directly connected IP nets in L1 LSP
* Neighbouring L1s in diff areas never adj
* L1 routers keep intra-area info contained

* L2 routing only converned with Area IDs
* l2 routers form backbone of domain
 * Must be contiguous
 * Pervades all areas within domain
* IP has no embedded area info
* Each L2 router advertises directly connected IP networks to achieve contiguous IP backbone connectivity
* Also avdertises all other L1 routes from own area
* While LSPs not leaked on L2 routers, IP routing info computed from L1 LSDB injected into L2 LSP
* No opposite flow
* L2 domain knows all IP network sin domain
* L1 similar to OSPF Totally Stubby, or at least NSSA-TS if external routes present

* L2 routers ignore area boundaries for adj and flooding, any area can form adj
* Default of L1L2 operation in IOS

**FLags in show isis database**

* ATT (Attached) - When L1L2 router performs L2 calc, and can reach other area besides own, sets ATT flag in LSP to indicate working attachment to other areas. L1 routers create default route towards any router with ATT bit set
* P (Partition Repair) - Feature that allows healing partitioned area over L2 subdomain (like Virtual Link), not on Cisco routers
* O (Overload) - Signals router unable to store all LSPs in memory, so don't use as routed path. Ignored when computing SPF. Still takes directly attached networks though.

* O bit can be used for maintenance, or bringing new router in network
* Allows settling of BGP table for example
 * Can be set for on reboot, or BGP signals convergence

* Suummarization should be config'd on each L1L2 router in area
* **summary-address** - Under process
* Applies euqually to intra-area networks from L1 to L2, and redist

# Auth in IS-IS

* IIH authed indepdnent of LSP, CSNP and PSNP
* Auth added as additional TLV
* LSPs must not be modded, so for L1 LSPs within area, must use same area password
* All L2 routers within L2 subdomain must use same domain password to auth LSPs

* LAN IIH - L1 - **isis auth mode {text | md5 } level 1, isis auth key-chain name level-1** - Interface Commands
* LAN IIH - L2 - **isis auth mode {text | md5 } level 2, isis auth key-chain name level-2** - Interface Commands
* P2P IIH - **isis auth mode {text | md5}, isis auth key-chain name**
* LSP, CSNP, PSNP - Level 1 - **auth mode {text | md5} level-1, auth key-chain name level-1** - Process command
* LSP, CSNP, PSNP - Level 2 - **auth mode {text | md5} level-2, auth key-chain name level-2** - Process command

* Can be activated for either, both or none
* IIH auth only direct neighborus
* If IIH fails, no adj
* If IIH Pass but non IIH fails, Up but no LSDB sync
* MD5 auth added in RFC3567 (then 5394)
* Key IDs not carried, so only key-string needs to match

# V6 support

* Identical rules for v6 as v4
* No different as no L3 protocol carrying packets
* Different AFs in a single instance

# Configure ISIS

R1
```
key chain ISISAuth 
 key 1
  key-string DaveLikesToRoute

int lo0
 ip address 10.1.1.1 255.255.255.0
 ip router isis

int Se0/0/0
 desc to R2
 ip address 10.1.12.1 255.255.255.0
 ip router isis
 isis authentication mode md5
 isis authentication key-chain ISISAuth
 isis three-way-handshake ietf

router isis
 net 49.0001.0000.0000.0001.00
 is-type level-1
 authentication mode md5
 authentication key-chain ISISAuth
 metric-style wide
 log-adjacency-changes all
```

R2

```
int lo0
 ip address 10.1.2.1 255.255.255.0

int Se0/0/0
 desc To R3
 ip address 10.12.23.2 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
 isis metric 100 level-2

int Se0/0/1
 desc to R1
 ip address 10.1.12.2 255.255.255.0
 ip router isis
 isis authentication mode md5
 isis authentication key-chain ISISAuth
 isis three-way-handshake ietf

router isis
 net 49.0001.0000.0000.0002.00
 authentication mode md5 level-1
 authentication key-chain ISISAuth level-1
 metric-style wide
 log-adjacency-changes all
 summary-address 10.1.0.0 255.255.0.0
 passive-interface Lo0 # This will advertise lo0's IP out
```

R3 with v6 config
```
int lo0
 ip addr 10.2.3.1 255.255.255.0
 ip router isis 
 ipv6 address 2001:DB8:2:3::1/64
 ipv6 router isis

int Se0/0/0
 desc To R4
 ip address 10.2.34.4 255.255.255.0
 ip router isis
 ipv6 address FE80::3 link-local
 ipv6 address 2001:DB8:2:34::3/64
 ipv6 router isis

int Se0/0/1
 desc to R2
 ip address 10.12.23.3 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
 isis metric 100 level-2

router isis
 net 49.0002.0000.0000.0003.00
 metric-style wide
 log-adjacency-changes all
 summary-address 10.2.0.0 255.255.255.0
 address-family ipv6
  summary-prefix 2001:DB8:2::/32
 exit-address-family
```

Check with

* show clns - Info about routers NET and Integrated IS-IS mode
* show clns is-neighbors - Neighbor info
* show clns neighbors - Shows SNPA of neighbor (for HDLC and PP, text description shown)
* show clns interface
* show isis neighbors
* show isis database detail
* show ip router isis
