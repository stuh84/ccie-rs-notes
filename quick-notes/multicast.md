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

# Processes

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
