# Why do you need multicasting?

* 1991 concept (Multicast Routing in datagram network)

## Problems with Unicast and Broadcast Methods

* Unicast - one copy of packet per group member
* Broadcast transmits wants, but goes on all links to everyone

## How Multicasting Provides Scalable and Manageable Solution

Requirements for multicast are as follows: -

* Designated L3 range for M'cast
* Multicast as dest, not source addr
* Multicast app insalled on all hosts who received traffic, same L3
  address as server (joining group)
* All hosts on LAN use standard method to calc L2 multicast address from
  L3 m'cast
* Dynamically indicate whether wants to receive traffic (IGMP/CGMP)
* Multicast routing protocols (DVMRP, MOSPF, PIM-DM, PIM-SM)
* Multicast UDP based, lac of TCP windowing and slow starts can congest
  links

# IP Addressing

* Address for group
* Dest IP for packet
* Multicast address never assigned to device (never source)
* Source always unicast

## Multicast Address Range and Structure

* IANA assigned Class Ds
* First 4 bits of first octet 1110
* 224.0.0.0 to 239.255.255.255
* No subnet mask

## Well-Known adresses

* IANA assigns addresses only with good technical justication
* Permanent multicast groups - 224.0.0.0-224.0.1.255
* SSM - 232.0.0.0/8
* GLOP - 233.0.0.0/8
* Private - 239.0.0.0/8

### Multicast Addresses for permanent groups

* Local non routed is 224.0.0.0/24
 * 224.0.0.5 - OSPF All Routers
 * 224.0.0.6 - OSPF All DRs
 * 224.0.0.1 - All multicast capabale hosts
 * 224.0.0.2 - All m'cast routers
* 224.0.1.0/24 routed
 * 224.0.1.39 and 40 used for Cisco Auto-RP

### Multicast Addresses for SSM

* Selects source of group
* Routing more efficient
* Selects better quality source
* Minimizes m'cast DoS

### Multicast Addresses for GLOP

* RFC 3180
* Anyone with ASN gets 256 global m'cast addresses
* IANA reserves for global uniqueness
* First octet 233, second and third ASN in binary (first 8 bits for
  second octet, second 8 bits for third)

### M'cast Add for Transient Groups

* Entire internet shares them
* Must be dynamically allocated and released when no longer used
* No standard method for using

## Mapping IP M'cast address to MAC addr

* IEEE-registered OUI of 01005E, then binary 0, then last 23 bits of
  Multicast IP
 * Identical for Ethernet and FDDI

Steps to do so are: -

1. Convert IP to binary
2. Replace first 4 bits 1110 of IP with 01-00-5E
3. Next 5 bits of binary IP with one binary 0
4. Last 23 bits of binary IP with last-23 bit of m'cast mac
5. Convert last 24 bits of mulciast MAC to 6 hex digits

Could get same MAC for 238.10.24.5 and 228.10.24.5

# Managing Distribution of M'cast Traffic with IGMP

* Router needs to know answer to 
 * Any hosts want this traffic?
 * If none, why should I forward it?
 * If any hosts interested, where?

* Switch needs to know ports to forward traffic
* Without IGMP, default to flood (as unknown destination, no source MACs
  for IP)
* CGMP, IGMP snooping, RGMP

## Joining a group

* Host listens to m'cast MAC as well as BIA
* Session Description Protocol and Service Advertising Protocol describe
  m'cast events and advertise them

## IGMP

* Evolved from Host Member Protocol
* RFC 1112 - v1
* RFC 2236 - v2
* RFC 3376 - v3
* IGMP messages in IP datagrams, protocol 2
* IP TTL 1
 * Passes only on LAN
* Two goals
 * Inform local m'cast router hosts wants m'cast traffic for group
 * As above, but leaving group
* M'cast routers use IGMP to known interface to send for groups and whic
  hosts
* IGMP enabled automatically when m'cast routing and PIM config'd
* Default version 2, can be changed per int

## IGMP v2

* 8-Octet IGMP message
* Type - 8 bit field, one of four message types
 * Membership Query 0x11 - Discovers group members on subnet. Group
   Membership Query has group field to 0.0.0.0. Group specfic sets to
group being queried. Sent by router after receiving IGMPv2 leave group
from host
 * v1 Membership report 0x12 - for v2 hosts, backwards compatibility
 * v2 membership report 0x16 - By group member, informs router that at
   least one group member on subnet
 * Leave Group 0x17 - By group member if last to send membership report
* Maximum Response Time - 8 bit field only in queries, units 1/10 of a
  second, 100 default, between 1 to 255
* Checksum - 16 bit over entire IP payload
* Group address - 0.0.0.0 in general queries, group address in group
  specific. Membership has group being reported, leave has group leaving

* v2 backwards compat with v1, 0x11 and 0x12 match codes for v1 Queries
  and reports
* V2 hosts and routers recognise v1 messages

* Better leave mechanism in v2

Following featueS: -
* Leave group messages - Notifies router that leaving group
* Group-specific queries - Specific rather than for all groups
* MRT - For queries, tunes response time for host membership report
* Querier Election Process - Selects preferred router for sending
  queries
* Reduces surged in v2 solicited report messages from hosts in response
  to queries by changing Query Response Interval. Spreads messages over
longer period
* Host can send IGMP report in response to query, or when host app comes
  up
* General query from v2 router every 60 seconds

## IGMPv2 Host Membership Query Functions

* Out LAN interfaces
* Any group member on interface?
* Routers send every Query Interval (defalt 60s)
* Dest 224.0.0.1/01-00-5E-00-00-01
* Source IP and MAC of routers IP and BIA
* TTL 1

## IGMPv2 Host Membership report Functions

* Reply to queries
* Communicates groups to receive
* Send under two conditions
 * v2 query from router received, supposed to send report for all groups
   its wants traffic for (solicited)
 * When host joins new group, sends to inform it wants traffic
   (Unsolicited)

### v2 Solicited Host Membership Report

* Report suppression helps with not having multiple reports
* v2 MRT timer set, Query Response Interval
* Maximum of MRT to send report to receive traffic for group
* Host picks random time between 0 and MRT
* When expires, sends HM report, only if no others heard for group

### v2 Unsolicited HM Report
* Sent when app launched


## v2 Leave Group and Group-Specific Query Messages

* LG drops latency to leave
* GS Query prevents router from stopping forwarding packets when another
  host leaves group
* In v2, leave sent when host leaves group
* v2 receives leave, sends group specific query to group
* In v1, router takes by default 3 mins to conclude if no hosts on int
* In v2, 3 seconds
* RFC 2236 recommends LG sent if leaving member last to send report in
  response to query
* Most vendors always send an LG when leaving

* Last Member Query Interval and LMQ Count says how long router says no
  hosts left on LAN. By default, MRT of 10 for GS Queries (1s)
* Router should receive response in this time, taken as LMQI

1. Send GS Query after rx of Leave
2. If no report in LMQI, step 1
3. Repeat step 1 LMQC times

* Default LMQC is 2, concludes no members seen

## v2 Querier

* Election when multiple routers on subnet
* General query to 224.0.0.1 sent
* When routers receive them, compares source IP to own int IP
* Lowest IP is groups querier
* Nonqueriers send no queries, monitor how often queries sent
* When two consecutive Query Intervals plus one half Query Response
  Interval, considered dead
* RFC 2236 says is Other Querier Present Interval
 * Default 255s
 * Default v2 query interval 125s
 * Default query response 10s

## v2 Timers

|Timer|Usage|Default Value|
|-----|-----|-------------|
|Query Interval|Period between general queries sent by a router|60s|
|Query Response Interval|MRT for hosts to response to periodic general queries|10s, can be between 0.1 to 25.5|
|Group Membership Interval|Time where if router gets no IGMP report, no members on subnet|260s|
|Other Querier Present Interval|Querier considered dead if no queries seen|255s|
|Last member query interval|MRT in GS queries and time period between two consecutive GS queries by same group|1s|
|v1 Router Present Timeout|If v2 host does not see v1 Query, v1 host says no v1 routers present, sends v2 messages|400s|

# IGMP version 3

* RFC 3376
* Major protocol revision
* Last-hop routers needed updating
* Host OSs modified, apps rewritten

* In v2, router forwards traffic to group regardless of source
* Malicious traffic/useless traffic (music streaming) could be sent to
  all members

* v3 hosts can specify source
 * Source Specific Multicast (SSM)
* Host indicates interest in receiving packets from certain sources, or
  to avoid certain sources, for group

* Membership report prcoess is
 * v3 report sent to 224.0.0.22 (IANA v3 membership report address)
 * Message type 0x22
 * Has SOURCE-INCLUDE <addresses>
* v3 compataible with v1 and v2

* Cisco designed URL Rendezvous Discovery and IGMP v3 lite for new
  features in v2 until v3 available in apps and OS

# v1 and v2 Interop

## v2 hosts and v1 routers

* v2 report message type 0x16, v1 says invalid message and ignores
* v2 hosts looks at MRT of periodic general IGMP query
 * Field 0 in v1
 * Non-zero in v2
* Marks interface as v1 int, stops sending v2 messages
* 400s v1 router present timer started on v1 query
 * If expires, send v2 messages

## v1 hosts and v2 routers

* v1 reports (0x12)
* IGMP hosts responds normally to v2 queries as very similar format to
  v1s (except second octet)
* If v2 hosts on same subnet, send v2 reports
 * v1 dont understand these, no report suppression
* v2 report would receive both v1 report and v2 in response to general
  query
* v2 router knows v1 on LAN
* Ignores leaves and GS
* Router suspends optimizations that reduce leave latency
* Ignored until IGMPv1-Host-Present-Countdown timer expires
* 2236 says routers sets if receivng v1 report
* Equal to group membership interval (180s in v1, 260 in v2)

# Comparison of v1, v2 and v3

Feature comparison

|Feature|v1|v2|v3|
|-------|-----|----|00--|
|First Octet for Query|0x11|0x11|0x11|
|Group Addr for General Query|0.0.0.0|0.0.0.0|0.0.0.0|
|D-Addr for General Query|224.0.0.1|224.0.0.1|224.0.0.1|
|Dflt Query Interval|60s|125s|125s|
|First Octet for report|0x12|0x16|0x22|
|Group addr for report|Joining M'cast Group addr|as per v1|Joining m'cast group addr and source address|
|D-Addr for report|Joining m'cast group addr|as v1|224.0.0.22|
|Report Suppression|Yes|yes|no|
|Can MRT be configured|No, 10s fixed|Yes, 0 to 25.5s|Yes, 0 to 53m|
|Can host send leave|No|Yes|yes|
|Leave destination|None|224.0.0.2|224.0.0.22|
|Can router send GS Query|No|Yes|yes|
|Can host send source and GS reports|No|No|yes|
|Can router send source and GS queries|No|No|Yes|
|Rules for electing querier|None, depends on M'cast Routing
Protocol|Lowest IP on subnet|as per v2|
|Compatauble with other versions|No|Yes, v1|Yes, v1 and v2|


# LAN Multicast Optimizations

## Cisco Group Management Protocol

* IGMP at L3
* Switches don't understand IGMP messages
* CGMP - L2 protocol
* Config'd on routers and switches
* Permits router to communicate L2 info learned from IGMP to switches
* Routers knows MACs of hosts and groups they're in
* With CGMP messages, switches can dynamically modify CAM entires
* Routers produce CGMP messages, switches listen
* Enable both ends of link
* L3 switches are routers for CGMP
* Server as CGMP servers only
* On these switches, CGMP can be enabled only on L3 ints that connect to
  L2 switches

```
int Fa0/1
 ip cgmp
```

* D-Addr of 0100.0CDD.DDDD
* Flooded through all ports, all switches get messages
* Useful info in messages is one or more pairds of MACs
 * GDA (Group Destination Address)
 * USA (Unicast Source Address)

1. CGMP router connected to switch, CGMP Join sent, GDA of 0, USA of own
MAC. CGMP knows m'cast router on port. Message repeat every 60s. CGMP
can be sent, same USA and GDA as Join.
2. IGMP join from host. Router examines L2 dest and source MAC of join.
   CGMP Join generated with M'cast MAC of group (GDA field) MAC of host
(USA field)
3. When switches receive above, search CAM for port with USA MAC.
   Creates new CAM entry (or uses existing if alreayd created) for
m'cast MAC in GDA. Adds port with host MAC in USA, forwards traffic to
port
4. IGMP Leave from host. Sees USA and group left. CGMP leave with
   multicast MAC for GDA of group, and unicast MAC in USA
5. Switch gets CGMP leave, looks for port with host MAC, removes from
   CAM entry for m'cast MAC in GDA

* IGMP messages optimized, switches know groups they go to (IGMP reports
  for example)

Combination of GDA and USA in CGMP messages mean: -

|Type|GDA|USA|Meaning|
|Join|Group MAC|Host MAC|Add USA port to group|
|Leave|Group MAC|Host MAC|Delete USA port from Group|
|Join|Zero|Router MAC|Learn of CGMP router on port|
|Leave|Zero|Router MAC|Release CGMP router port|
|Leave|Group MAC|Zero|Deletes group from CAM|
|Leave|Zero|Zero|Deletes all groups from CAM|

* Last message used with things like **clear ip cgmp**

## IGMP Snooping

* Multivendor
* Switch examines IGMP conversations
* Learns locations of routers and group members

General Process: -
1. Detects whether multiple routers on same subnet. Following messages
   determine which ports routers connected: -
 * IGMP general query - 01-00-5E-00-00-01
 * OSPF message - 01-00-5E-00-00-05 and 01-00-5E-00-00-06
 * PIMv1 and HSRP hellos - 01-00-5E-00-00-02
 * PIMv2 hellos - 01-00-5E-00-00-0D
 * DVMRP probes - 01-00-5E-00-00-04
 * When above seen, router port added to port list of all GDAs in that
   VLAN
2. When switch gets IGMP report, looks at GDA, creates CAM for GDA. Adds
   port to it and router port.
3. When IGMP leave, removes port from group in CAM. If last nonrouter
   port, switch discards leave, otherwise sends to router

* Requires hardware filtering support in switch (so can see IGMP reports
  during standard mc'ast traffic)
* Forwarding should be done in forwarding ASICs
* IGMP snooping enabled by default on 3560s and most other L3 switches

```
ip igmp snooping
no ip igmp snooping vlan 20
ip igmp snooping last-member-query-interval 500
ip igmp snooping vlan 22 immediate-leave
```

* Less efficient in maintaing group info
* General queries sent to 224.0.0.1 and forwarded through all ports in
  VLAN
* Hosts do not see each others IGMP reports (breaks report suppression)
 * Switch sends only one IGMP report per group to router

# Router-Port Group Manage Protocol

* L2 protocol
* Enables router to tell switch which groups it wants traffic for
* Helps routers reduce overhead when attached to highspeed LAN backbones
* Enable on interface **ip rgmp**
* Not compatible with CGMP
 * When enabled, CGMP silently disabled, and vice versa
* RFC 3488 exists, but is Cisco Proprietary
* Works with IGMP snooping
* IGMP helps switches control distribution of m'cast traffic for hosts,
  but not for routers

Four messages, sent to m'cast IP of 224.0.0.25: -
* When RGMP enabled, RGMP hello every 30s. When Switch receives it,
  stops forwarding all m'cast traffic on port
* To receive specific groups, RGMP Join G (G is m'cast address)
* RGMP Leave G to stop traffic
* RGMP Bye when RGMP disabled. Switch sends all IP m'cast traffic on
  port again

# IGMP Filtering

* Filters on SVI, port, or per-port per-VLAN basis
* Filters IGMP traffic
* Manages IGMP snooping
* Defines whether IGMP packet discarded or allowed
* with v1 and v2, entire packet discarded
* v3, packet rewritten, removing message elements denied
* Can restrict on groups that can be joined on port, maximum numbr of
  groups on interface, and IGMP version
* User policy applied to L3 SVI, L2 port or VLAN on trunk
 * L2 port can be access or trunk
* Only works if snooping enabled
* Three filters
 * IGMP group and channel access control
 * several IGMP groups and channels limit
 * IGMP minimum version

# IGMP Proxy

* Enables Unidirectional link routing environment to join m'cast group
  from upstream network
 * Not directly connected to downstream router

* Enables hosts to jon group source
 * Situations like internet gapped, no direct message

1. User 1 sends IGMP membership report to join group G
2. Router sends PIM Join to RP
3. RP receives PIM join, forwards entry for Group G on LAN
4. RP looks at mroute, proxies IGMP membership report to upstream UDL
   device
5. Proxy creates and mantains forwarding entry on UDL

**Config on upstream**
```
int Gi0/0
 ip address 10.1.1.1 255.255.255.0
 ip pim dense-mode

int Gi1/0/0
 ip address 10.2.1.1 255.255.255.0
 ip igmp unidirectional-link
 ip pim dense-mode
```

**Config on downstream**

```
ip pim rp-address 10.5.1.1 5
access-list 5 permit 239.0.0.0 0.255.255.255

int lo0
 ip address 10.7.1.1 255.255.255.0
 ip pim dense-mode
 ip igmp help-address udl ethernet 0
 ip igmp proxy-service

int Gi0/0/0
 ip address 10.2.1.2 255.255.255.0
 ip pim dense-mode ip igmp unidirectional-link

int Gi1/0/0
 ip address 10.5.1.1 255.255.255.0
 ip pim sparse-mode
 ip igmp mroute-proxy lo0
```
