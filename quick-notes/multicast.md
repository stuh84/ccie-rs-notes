# Messages

* 8-Octet IGMP message
 * Types
  * Membership Query - 0x11, group field of 0.0.0.0 for Group Membership Query, Group Specific to group being queried. Sent by router after receiving IGMPv2 leave from host
  * v1 Membership report - for v2 hosts, backwards compat
  * v2 membership report - By group member, says at least one group member on subnet
  * Leave group - If last to send membership report
 * MRT - 8 bit field in queries, units 1/10 of a second, 100 default, between 1 and 255
 * Checksum - 16 bit over payload
 * Group Address
 * v2 back compat with v1, 0x11 and 0x12 match v1 queries and reports
 * v2 hosts and routers recgnize v1 messages

## PIM-DM

* Hello - forms neighbours, monitors Adj, elects PIM DR on MA networks
* Prune
* State refresh
* Assert
* Prune override
* Graft/Graft-Ack 

# Timers

* IGMP General Query v2 - every 60s

# Trivia

## Addressing

* Class D
* First 4 bits of first octet - 1110
* 224.0.0.0/4
* 224.0.0.0 to 224.0.1.255 - permanent
* SSM - 232/8
* GLOP - 233/8
 * Anyone with ASN gets 256 global m'cast addresses
 * First octet 233, second and third ASN in binary (first 8 bits for second octet, second 8 bits for third)
* Priv - 239/8

## IP to MAC

* 01005E, binary 0, then last 23 bits of M'Cast IP
* Could get same MAC for 238.10.24.5 and 228.10.24.5

## IGMP

* Hosts listen to m'cast MAC as well as BIA
* Session Description Protocol and Service Advertise Protocol dscribe m'cast events and adv them
* IP TTL of 1
* IGMP messages in IP, proto 2
* IGMP auto enabled when m'cast routing and PIM config'd
* Default v2

### v2

* Leave group messages
* Group specific queries
* MRT - Tunes response time for host membership report
* Querier election process - Selects preferred router for sending queries
* Reduces surged in solicited reports by change Query Response Interval
* Host can IGMP report in response to query or when app comes up
* Host membership query functions - Any group members on ints, dest 224.0.0.1/01.00.5e.00.00.01, source IP and MAC of routers IP and BIA
* Host membership report - Replies to queries - To query (solicited), informing (unsolicited)
 * Solicited - report suppression, v2 MRT timer set, query resopnse interval, maximum of MRT to send report to receive traffic for group, host picks random time, when expires, sends, but only if not heard from others
* Leave - Drops latency
* GS Query - Prevents routers from stopping forwarding traffic when another host leaves (sent when a leave received from segment)
* In v1, router takes 3 mins to conclude no hoss on int, 3s in v2
* Last Member Query Interval and LMQ Count - says how long for no hostsleft on lan (MRT of 10 for GS queries, i.e 1s)
 * GS Query sent, if no report in LMQI, repeat LMQC times, default LMQC is 2
* Querier - Election for multi routers, gen query to 224.0.0.1, lowest IP of router groups querier, other routers monitor. Two consec query ints and one half Query Response Interval = dead
 * Default 255s
 * v2 query int default 125s
 * Default query response 10s

### V3

* SSM
* 224.0.0.22
* SOURCE-INCLUDE in v3 membership report, 0x22 message type

### v1 and v2 Interop

**v2 hosts, v1 routers**
* v2 report message type 0x16, v1 ignores
* v2 looks at MRT in periodic query, 0 = v1, non zero is v2
* Marks int as v1, stops v2 messages
* 400s v1 router present timer on v1 query

**v1 hosts and v2 routers**
* v1 reports
* IGMP hosts response as queries similar format
* If v2 hosts on same subnet, send v2 reports, no report supp
* v2 router knows v1 on LAN
* Ignores leaves and GS
* Router suspends leave optimization
* Ignore until IGMPv1-Host-Present-Countdown timer expires

## LAN Multicast Optimizations

### CGMP

* CGMP - l2 protocol
* On routers and switches
* Routers communicate L2 info to switches
* Routers know MAC of hosts and groups
* Switches modify CAM entries
* Routers produce CGMP messages, switches listen
* Enable on both ends (ip cgmp on interface)
* Daddr of 0100.0CDD.DDDD
* Flooded through all ports
* GDA and USA (Group Dest, Unicast Source)

|Type|GDA|USA|Meaning|
|----|---|---|-------|
|Join|Group MAC|Host MAC|Add USA port to group|
|Leave|Group MAC|Host MAC|Delete USA port from Group|
|Join|Zero|Router MAC|Learn of CGMP router on port|
|Leave|Zero|Router MAC|Release CGMP router port|
|Leave|Group MAC|Zero|Deletes group from CAM|
|Leave|Zero|Zero|Deletes all groups from CAM|

### IGMP Snooping

* Switch exaines conversations
* Detects multiple routers on same subnet (IGMP gen query, OSPF, PIM messages, HSRP, DVMRP etc)
 * Router port added to all GDAs in that VLAN
* IGMP report to switch - looks at GDA, creates CAM for GDA - adds port to it and router port
* When IGMP leave - remoes port, if last nonrouter port, switch discards leave, otherwise sends to router
* Enabled by default on 3560s and most other L3 switches
* Less efficient in maintaining group info
* Gen queries sent to 224.0.0.1, forwarded through all ports in VLAN
* Hosts dont see each others IGMP reports (report suppression)

## Router-Port Group Manage Protocol

* L2 Proto
* Router tells switch which groups it wants traffic for
* Helps routers redunce overhead
* `ip rgmp`
* Enable turns off CGMP
* Works with IGMP snooping
* IGMP helps switches control distribution of m'cast traffic for hosts, but not routers
* 224.0.0.25
* RGMP Hello every 30s, switch stops forwrding all m'cast traffic on prt
* RGMP Join G
* RGMP Leave G
* RGMP Bye - disabled RGMP

## IGMP Filtering

* Filters IGMP traffic
* Manages IGMP snooping
* With v1 and v2, entire packet discarded
* v3 - packet rewritten
* Restrict groups that can be joined on port, max groups, and IGMP version
* Use policy applied to L3 svi, l2 port or VLAN
* Only works with snooping
* Three filters
 * IGMP Group and channel access control
 * Several igmp groups and channels limit
 * IGMP min ver

## IGMP Proxy

* Enables Unidirectional link routing environment to join m'cast group from upstream network
* Not directly connected to downstream
* Hosts can join source
* User 1 sends IGMP membership report to join G
* Router sends PIM Join to RP
* RP Receives PIM Join, forwrds entry for Group G on LAN
* RP looks at mroute, proxies IGMP report to upstream UDL 
* Proxy creates and maintains forwarding entry on UDL

## RPF Check

* Looks at source of m'cast packet
 * Source int must be IGP dest for source IP

## Scoping

### TTL

* Default TTL on interface of 0
* Can configure higher for a boundary
* Applies to all m'cast packets

### Admin scope

* Apply filter
* 239.0.0.0/24 - private addr

## PIM-DM

* RPF Check, floods until prunes
* v2 sends hellos every 30s
* IP Protocol 103
* 224.0.0.13
* Hold time is three times hello
* v1 used PIM Queries (IP proto 2, 224.0.0.2)
* PIM messages sent only on ints with known active pim neighbours

### Source-Based Distribution Trees

* If RPF passed, packet forwarded to all PIM neighbours except packet source
* Also known as shortest path tree
* Path from source host to all subnets requiring it
* Different SBDT for each source and m'cast group, differs on location
* (S, G)

### Prune

* For subnets not needing it
* Default 3 minute prune timer, resets

**Failed Link**
* If unicast table changes, RPF int can change
* Different ints in outgoing list
* INts pruned may go forwarding

**Pruning rules**
* Packet on non-RPF int
* No hosts wanting it
* IGMP Leave
* Prune reference SPT out RPF int
* show ip mroute has P flag (prune)
* C flag and RPF neigh of 0.0.0.0 means connected device is source
* Prune and Join same message (group in join or prune field)

**Steady-State Operation and State Refresh**
* v2 for State refresh
* Prevents auto unpruning
* Default timer 60s (not tied to prune expiration)

**Graft**
* Unprunes
* Message upstream
* Graft Acks downstream
* Individual grafters per router if many pruned

### LAN Specific issues in PIM-DM and PIM-SM

**Prune Override**
* Join on b'cast segment, sent to 224.0.0.13
* sent before 3-second time expires

**Assert**
* Routers neg, winner forwards onto LAN
* Winner on routing protocol and metric to find route to reach unicast metric
 * If tie, metric wins
 * If tie, highest IP on LAN wins

**Designated Router**
* PIM hellos elect DR on MA 
* Router with highest IP DR on link
* Applies for v1, as no querier mechanism
 * in v1, PIM DR IGMP querier
 * in v2, directly elects
* Querier and Assert likely different routers
* Querier uses lowest IP, Assert has higehst IP as breaker

## PIM SM

* Elects DR on MA network
* Prune overrides
* Assert elects designated forwarder

## Joining Shared Tree

* Root-Path Tree
* RP is root
* One tree for each active group
* PIM Joins or IGMP Membership Reports
* S flag in show ip mroute
* Source Path Tree to RP, RPT to receivers

## SSO with Send Joins

* Periodic PIM-SM join every 60s
* Must be at least one IGMP report/Join in response to gen query
* 3m Prune timer

## Analsying Mroute table

* If incoming int null, router is root (RP)
* RPF neighbour listed as 0.0.0.0 for same reason
* T is SPT, source listed at beginning of line
 * Incomcing int shown and RPF neigh
* RP uses SPT to pull traffic from source

## SPT Switchover

* Avoids inefficient paths
* Switches overs, prunes to upstream of shared tree
* RPT joined first as source unknown



## Pruning from Shared Tree

* RPT may no longer be needed
* PIM-SM Prune to RP, references S,G SPT, identifying source

## Dynamically finding RPs and using Redundant RPs

* Unicast
* AutoRP
* BSR

### Auto-RP

* RP-Announce to 224.0.1.39
 * Router advertises groups its RP for
 * Sent every minute
* Need a mapping agent
* Learns RPs and groups they support
* RP Discovery, to 224.0.1.40
* RP discovery for MA to decide which RP for each group
* MA selects Highest IP as RP for group
* If router with PIM-SM and Auto_RP configd, auto joins 224.0.1.40 group
* Learns Group-to-RP mappings
* ip pim sparse-dense-mode or ip pim autorp listener for dense propagation of these groups

### BSR

* PIMv2
* RP sends messages to another router collecting group-to-RP mappings
* Router distrbutes mappings (similar to MA)
* MA does not pick best RP for each group, all mappings sent to PIM routers instead
* Routers pick current best RP with same hashing info
* BSR floods mappings to ALL-PIM-ROUTERS (224.0.0.13)
* Floods out all non-RPF ints
* If BS on non-rpf int, drops
* Each cRP informs BSRs groups it supports
* C-RP unicasts messages to BSR, with IP used and groups
* BS message contains all c-RPs
* For multiple, c-BSRs send BS with priority and its IP
 * Highest priority then highest IP
* Winniner BSR sends BSR message, others monitor
* ACL can limit what groups router is RP for
* Can specify priority for multiple BSRs

### Anycast RP with MSDP

* Can use any RP config
* Multiple RPs acting as RP
* Will partition group, so MSDP peers up to make sure that mappings that either RP learns are passed over

## MSDP for Interdomain

* RP uses MSDP to send messages to peer RPs
* Source Active - IP of each srouce for each m'cast group
* Unicast TCP connection
* Static config
* BGP used for main routing
* Receiver in other domain joints SPT of source, as this is whats in SA
* If RP No receives for gorup caches them for later
* SAs every 60s
 * Lists groups nd sources
 * SA request for new list
 * SA response back
* Config Auto-RP or BSR first

## Bidi PIM

* PIM-SM inefficient for large groups of senders and receivers
* RP builds shared tree with root
* When sources send m'casts, router rx does not use PIM register, instead forwards in opposite direction of shared tree to RP
* RP Forwards through shared tree
* All packets forwarded as per step 2. 
* RP Does not join source tree for source, leaf do not join SPT either

## SSM

* Subscribe to S,G with both source and group
* IGMPv3
* Only edge routers nearest host need SSM

## v6 Multicast Pim

* ipv6 multicast-routing - enables on all ints, assumes v6 PIM, always sparse
* no ipv6 pim per int
* Tunnels formed for routing
* Show int tunnel shows pim/ipv6
* show ipv6 pim neighbours - Formed with link locals, DRs still elected, values manipulated as per v4

### DR priority manipulation

* `ipv6 pim dr-priority <0-4294967295> - Higher is better`

### PIMv6 hello

* every 30s
* When all neighbours replied, DR chosen
 * Highest priority, then highest IP
* Hold time 3.5 times hello
* Auth with MD5 hash

### IPv6 Sparse-Mode Multicast

* For RP, static, v6 BSR and embeeded RP
* Static as before

## Multicast Listener Discovery

* `ipv6 mld join-group ADDRESS`
* Replaces IGMP
 * v1 like IGMPv2
 * v2 like v3, supports SSM
* Queriers elected through MLD
* ICMP messages carry inside, link local in scope, router alert option set
* Three messages, query, report done
 * Done like leave, triggers query to check more receivers on segment
* Options - `ipv6 mld limit` limits number of recipients
* ipv6 multicast-routing auto enables MLD

## Embedded RP

* RP part of m'cast group addr
* Extra RP identity and uses it for shared tree immediately
* Explcit config on device that is RP

Accomplish with following rules
* Always start with FF70::/12
* Scopes are
 * 1 - Int local
 * 2 - link local 
 * 4 - admin local 
 * 5 - site local
 * 8 - org local
 * E - global
* Isolated three values from RP
 * RP Int ID, prefix length in hex, RP prefix
 * `FF7<SCOPE>:0<RP Interface ID><Hex prefix length>:<64-bit RP prefix>:<32 bit group ID>:<1-F>`
 * Eg, RP of 2001:2:2:2::2/64
  * RP int is 2 (taken from ::2)
  * Prefix length 64 (40 in hex)
  * RP prefix 2001:2:2:2
  * Global scope
  * 32 bit group ID commonly 0
  * FF7E:0240:2001:2:2:2:0:1
* Make sure router knows its an RP (ipv6 pim rp-address)
* Use embedded for group joins on others

# Processes

## Sources sending packets to RP

1. Source sends packet to RP
2. RP sends m'cast packet to all routers/hosts in group, shared tree
3. Routers with local hosts that IGMP join can join source specific S,G SPT
4. Routers on same subnet as soure register with RP
5. RP acces registration only if knows it is needed

**With no requests**

1. Hosts sends m'cast to group, router receives m'ast to same LAN
2. Router sends PIM Register to RP
3. RP sends unicast Register-Stop
 * PIM register encaps first m'cast packet
4. When register stop rx'd, 1m Register-Suppression
5. 5s before timer expires, another register with Null-Register bit set, without encap'd packet
 * Another register stop (back to 4)
 * Doesn't reply, timer expires, R1 sends encap'd m'cast packets in PIM register

## Completion of source registration

1. Host sends m'cast to group
2. Router encaps inside register
3. RP de-encaps and sends
4. RP joints SPT, PIM-SM Join for group S,G to source
5. When source router receives Join, forwards traffic to RP
6. RP sends unicast Register-stop 

## SPT Swithcover

1. Source sends m'cast to first hop
2. First hop to RP
3. RP to router in shared tree
4. PIM-SM Join out preferred int (if better unicast path than RPF to RP)
5. First hop router places another int in forwarding

**J flag (join)** - traffic switches from RPT to SPT

# Config

## IGMP Proxy

**Upstream**
```
int Gi0/0
 ip address 10.1.1.1 255.255.255.0
 ip pim dense-mode

int Gi1/0/0
 ip address 10.2.1.1 255.255.255.0
 ip igmp unidirectional-link
 ip pim dense-mode
```

**Downstream**
```
ip pim rp-address 10.5.1.1 5
access-list 5 permit 239.0.0.0 0.255.255.255

int lo0
 ip address 10.7.1.1 255.255.255.0
 ip pim dense-mode
 ip igmp help-address udl Gi0/0/0 <--- Any reports to this interface can be sent over the UDL interface
 ip igmp proxy-service <--- Checks mroute table for any (*, G) entries with interfaces config'd with ip igmp mroute-proxy, creates and receives one IGMP report on this interface

int Gi0/0/0
 ip address 10.2.1.2 255.255.255.0
 ip pim dense-mode
 ip igmp unidirectional-link

int Gi1/0/0
 ip address 10.5.1.1 255.255.255.0
 ip pim sparse-mode
 ip igmp mroute-proxy lo0 <--- Allows forwarding of IGMP reports to an interface with the ip igmp proxy-service, for all (*, G) groups
```

## AutoRP

**Mapping Agent**

```
ip multicast-routing

ip pim send-rp-discovery scope 10

int Se0
 ip pim sparse-mode
```

**RP**

```
int lo0
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim send-rp-announce loopback0 scope 10
```

## BSR

**BSR**
```
ip multicast routing

int lo0
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim bsr-candidate lo0 0 # 0 is priority, default
```

**RP**

```
ip multicast-routing

int lo0
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim rp-candidate lo0
```

## MSDP

```
int lo2
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

ip multicast-routing

ip pim rp-candidate lo2
ip pim msdp peer 172.16.1.1
```

```
int lo0
 ip address 172.16.1.1 255.255.255.255
 ip pim sparse-mode

ip multicast-routing
ip pim rp-candidate lo0
ip msdp peer 10.1.10.3 connect-source Lo0
```

## SSM

```
ip multicast-routing
ip pim ssm {default | range ACL} - Default is 232.0.0.0/24

int Fa0/0
 ip pim sparse-mode
 ip igmp version 3
```

## IPv6 BSR

```
ipv6 pim bsr candidate bsr 2001:2:2:2:2::2
ipv6 pim bsr candidate rp 2001:1:1:1::1
ipv6 pim bsr candidate rp 2001:3:3:3::3
```


# Verification

```
show ip msdp peer

show ip pim rp

show ip igmp snooping

show ip pim rp mapping

show ipv6 pim bsr rp-cache - shows cache from RPs

show ipv6 bsr candidate-rp

show ipv6 pim interface

show ipv6 mld interface

show ipv6 pim traffic

show ipv6 pim group-map
```


