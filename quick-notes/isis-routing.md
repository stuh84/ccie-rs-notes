# Messages

## Packet Types

**Hello**
* IIH - separate for L1 and L2 on b'cast
* L1L2 on P2P

**LSP**
* Smallest standalone element in LSDB is entire LSP
* No different types, all TLVs
* IDd by Sys ID of originator (6 octets)
* Pseudonode ID - differentiates from LSP describing router, and LSP for MA networks in which router is DIS (1 octet)
* LSP Number - Fragnumber of LSP (1 octet)
* Triplet is LSPID
* For LSPs describing themselves, PSID 0
* Seq 32 bit, starts 0x00000001, end 0xFFFFFFFF, modificaiton increments, highest most recent, no wrapover, woudl need to turn off to expire or change sys ID
* MTU limits payload size
* Frag required, same sys, psid, frag number increments, starts at 0
* frag only by originator (so end to end MTU but be no smaller than router)
* Inside LSP
 * adj to neighbour routers/networks
 * Intra/Inter are prefix
 * Ext prefix
 * Addr info about all in LSP of each router in network
 * Topology info about network and connected rtrs from PS LSP
  * Gen'd by DIS
 * IS-IS rtr on a level gen's 1 LSP for itself, plus a PS LSP for each network a DIS in
 
**CSNP**
* Syncs LSPs
* Seq refers to range of LSPIDs 
* Like DBDs
 * Complete list of LSPs in LSDB
 * Rx router compares own
* Multiple senf if MTU filled
* Full range start swith 0000.0000.0000.00-00 (Sys ID, PSID, LSP-ID)
* Ends with FFFF.FFFF.FFFF.FF-FF

**PSNP**
* Liek LSR and LSA
* Requests an LSP or acks arrival
* On P2P, req's and acks
* B'cast, PSNP req, ack'd with CSNP
* Native on b'cast and P2P

* Try to use P2P instead of hub-spoke or p2mp
* B'cast links are mcast links in IS-IS

# Timers

* 10s hello default, 1-65535 range
 * `isis hello-interval seconds [level]`
* Hold down multiplier of hello, default 3
 * `isis hello-multiplier MULTIPLIER [level]`
* Don't need to match b/w neighs
* DIS 1/3 of timers
* LSP remaining lifetime is 1200 sec by default (20m)
* LSPs refreshed every 15m
 * Deleted after lifetime, empty LSP with lifetime 0 (LSP purge)
 * No flushing yet, ZeroAgeLifetime of 60s first


# Trivia

* Messages in data link frames, so no need for IP, TCP etc as transport
* Adj and addr info n TLVs
* Define new TLV in existing topology for new AF
* NSAP for entire node
* Basic address is IDP (Initial Domain Part) and DSP (Domain Specific Part)
* Variable length
* AFI and IDI (Authority and Format ID, and Initial Domain ID)
 * Indicate routing domain for node
* DSP
 * HO-DSP - Area/part node is in
 * Subfields - System ID - unique to Node, 1 to 8 octets long, usually 6
 * SEL (NSAP Selector/NSELF) - 1 octet
* Usually AFI of 49
* Minimum NSAP 8 octets with AFI, Sys Id and SEL, max 20
* If SEL 0, entire address IDs dest node
* NSAP with Sel 0 is NET
* 49.XXXX.XXXX.XXXX.XXXX.00
* L2 int is Sub Network Point of Attachment
* IS enumerates ints with local 1 octet num, local circuit ID, increments 1 per int, begins at 0 for cisco

## Metrics, Levels, Adj

* Default - required by all IS-IS imps
 * Delay, expense error, and default, all calc 4 different SPF trees
* Default usually only  one supported
* Default metric 10 - isis metric METRIC [LEVEL] to set
* Originally 6 bytes wide, range of 1 to 63 for ints, path 1 to 1023 (10 bits)
* Wide metrics make 24 bits per int, 32 per path
* LSP for each level
* L1 and L2 LSPs describe adj at each level
* LSPs never leak between L1 and L2

### Adj stats

* Down - No IIH from neigh
* Initializing - IIH from neigh, not certain neigh rx'd own IIH
* Up - IIH from nei, knwon neigh sees IIH

## Over P2P

* No bidi check initally
* Added 3 way handshake
* Router has LCIDs (on p2p only in IIH)
* On b'cast, LCID is PSID if router is DIS
* Extended LCID (4 octets long) over 256 lcid originally
 * Auto assigned
* Adj state TLV has following
 * Adj 3 way state
 * Extended LCID
 * Neighbour Sys ID
 * Neighbour Extended LCID
* IETF handshake - tlv accept if: -
 * Neigh SysID and Ext LCID not present
 * Nei Sys ID matches own Sys ID, same for neigh ext lcid
* All LSPs marked for flooding over P2P
* CSNPs sent
 * Could be exchanged before LSP transmission takes place (periodic scheduling)
* Unmarks LSPs already has
* Don't need above as usually done - every LSP must be ack'd
 * Can send CSNPs to ack with `isis csnp-internval INTERVAL [level]`

## Over B'cast

* 802.2 LLC, DSAP and SSP set to 0xFE
* L1 packets m'cast to 0180.c200.0014, L2 to 0015
* IIH detects neigh
 * Lists neighbouring routers on b'cast int in IIH
* If own SNPA in IIH, bidi
* If not, initializing

### DIS

* No backup DIS
* Router with highest int priority
 * Highest SNPA
 * Highest Sys ID (if on FR, ATM)
* Priority in 0-127 range
 * `isis priority VALUE [level]`
 * 0 excludes
* Preemptive
* Performed on each rx'd IIH
* All routers fully adjancent
* Every router sends LSP on b'cast link
* Helps routers sync
* Represents in LSDB as standalone object (pseudonode)
* 10s CSNP
* Reference point of LSPs
* No ack of LSPs, just seen in next CSNP
* PSNP only for LSP req
* DIS originates PS LSP
* Remaining routers update LSPs to point to new PSLSP if it fails

## Areas

* Nieghbouring L1s in diff areas never adj
* L2 routers advertise DC'd networks and all other L1 routers from own area
* IP routing info compute from L1 LSDB injected into LSP (LSPs not directly leaked)
* L1 like OSPF Tottally Stubby
* Flags in show isis database
 * ATT - For L1 router to point def route to (for when L2 Calc'd and can reach other areas besides own)
 * P (Partition Repair) - Heals paritioned area over L2 subdomain
 * O - Unable to store all LSPs in memory, so dont use computed path
  * Can be set on reboot, or BGP signals convergence
* summary-address - under process

## Auth

* IIH independent of LSP, CSNP and PSNP
* Auth a TLV
* LSPs not modded, so all within same area have same PW
* Can pass IIH auth but fail LSP/CSNP/PSNP auth
* Key IDs not carried, so only key-string needs to match

## v6 support

* No different from v4 (as ISIS not v4 specific anyway)




# Processes

## P2P 3 way handshake - Cisco

`isis three-way-handshake cisco`
1. If router rx's IIH with 3way state down, Router A hears B, b might not hear A, state send of initializing in IIH
2. B sees initializing, knows own IIH worked, sends up state
3. When A sees IIH up, bidi confirmed

## P2P IETF 3 way

`isis three-way-handshake ietf`
* If no IIH from B, A sending IIH with extended LCID only, set to Router As outgoing int ID
* When IIH arrives and accepted, puts B's sys ID in neigh sysid field, Ext LCID in Neighbour Ext LCID field


# Config

## Auth

**LAN IIH**
```
isis auth mode {text | md5 } level {1 | 2}
isis auth key-chain name level-{1|2}
```

**P2P IIH**
```
isis auth mode {text | md5}
isis auth key-chain name
```

**LSP, CSNP, PSNP**
```
auth mode {text | md5} level-{1 | 2}
auth key-chain name level-{1|2}
```

## IPv4 and IPv6 config

```
int lo0
 ip address 1.1.1.1 255.255.255.255
 ip route isis
 ipv6 address 2001:DB8:2:3::1/128
 ipv6 router isis

int Se0/0/0
 ip address 10.2.34.4 255.255.255.0
 ip router isis
 ipv6 address FE80::3 link-local
 ipv6 address 2001:db8:2:34::3/64
 ipv6 router isis
 isis authentication mode md5
 isis authentication key-chain ISISAuth
 isis three-way-handshake ietf

int Se0/0/1
 ip address 10.12.23.3 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
 isis metric 100 level-2

router isis
 net 49.0002.0000.0000.0003.00
 metric-style wide
 authentication mode md5 level-1
 authenticaiton key-chain ISISAuth level-1
 summary-address 10.2.0.0 255.255.255.0
 address-family ipv6
  summary-prefix 2001:db8:2::/32
 exit-address-family
```

# Verification

```
show clns - Info abotu routers NET and Integrated ISIS mode
show clns is-neighbors - Neighbour info 
show clns neighbors - SNPA of neighbour (for HDLC and PPP, text description shown
show clns interface
show isis neighbors
show isis database detail
show ip router isis
```
