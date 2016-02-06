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
|Query Response Interval|MRT for hosts to response to periodic general
queries|10s, can be between 0.1 to 25.5|
|Group Membership Interval|Time where if router gets no IGMP report, no
members on subnet|260s|
|Other Querier Present Interval|Querier considered dead if no queries
seen|255s|
|Last member query interval|MRT in GS queries and time period between
two consecutive GS queries by same group|1s|
|v1 Router Present Timeout|If v2 host does not see v1 Query, v1 host
says no v1 routers present, sends v2 messages|400s|

# IGMP version 3
