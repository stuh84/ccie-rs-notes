# Messages

# Timers

## HSRP

* 3s Hello, 10s dead

# Trivia

## ARP

* Dest of 255.255.255.255
* Same place as IP header for source and dest IP

## HSRP

* Preemption disabled by default
* 255 groups per int
* 0000.0C07.ACXX (XX is hex of group)
* Clear text and MD5 auth

## VRRP

* 0000.5E00.01XX
* IOS Object Tracking rather than internal
* Preempts by default
* Master rather than active
* Group IP an interface on one of the routers

## GLBP

* AVG assigns each router unique MAC
* 4 virtual MACs max
* 0008.B40X.xxyy, X.xx (10 bit GLBP number), YY per router
* 1024 groups per int

## NTP

* NTP client mode on most routers/switches
* NTP defines messages used and algs to adjust
* Symmetric active - ntp peer (syncs with another host)

## SNMP

* Get - one variable request
* GetNext - Next singlemib leaf variable in tree
* GetBulk - Multiple MIB variables in one request
* Response to any Gets and Sets
* Set
* Trap - unsolicited
* Inform - between managers

### Security

* v3 added auth and encryption
* SHA and MD5 for message digest of each message
* AES and DES encryption

## Syslog

* Cisco's dont log to NVRAM by default
 * `logging buffered`
* UDP 514 by default
* `logging host`
* `logging trap` then 0-7 for severity levels

## WCCP

* Routers use caching engines
* Hosts unaware of proxying
* Can cluster engines
* Pool uses WCCP messages
* 32 with one router in v1
* Lowest IP as lead engine
* Lead content engine distributes traffic
* only 1 router in v1
* v2 multiple routers

### V2 over V1

* TCP and UDP outside of 80
* Segment caches
* M'cast support
* Multiple routers (32 per cluster)
* MD5 security
* Default in IOS

## IP SLA

* Can measure delay, jitter, packet loss, sequencing, path, connectivity, download type and MOS

## RITE

* Exports packets to VLAN or LAN
* Only for traffic rx'd on multiple WAN/LAN Ints simultaneously, used for IDS
* Copies packets to MAC of IDS

## RMON

* Thresholds based on SNMP objects
* Events track, set rising and falling thresholds
* For trap, SNMP community of server

## BFD

* Requires CEF
* Async mode - contrl packets sent between two systems (i.e. must be set on both sides)
* Enabled at int level, applied to routing protocols
* Interval negotiation
* QoS of CoS 6 by default
* ip route static bfd INT NH-TO-MONITOR
 * says to monitor a next hop over an interface, for use with static routes
* Echo mode
 * Enabled by default
 * works with async
 * Echoes by forwarding, sent back on same path
 * "without assymetry"
 * `no bfd echo`
 * `no ip redirects` required
 * Used to forward in hardware quickly
* BFD Template - can apply to PW classes etc

### Multihop

* Uses template (keyword multihop)
* Uses BFD map
* Associated mode - static route auto linked to bfd static if next hop matches destination
* Unassociated - not added to any routes automatically

### Dampening

* Done in tempalte, takes down until network stablisies
* Can auth with key chains

### Slow timers
* For control packet,s can bring down timers, for checking liveliness when echo mode enabled

## Netflow

**v5**
* Packet format fixed
* Flows calced in int ingress
* Outbound based on inbound of other ints
 * Needs to be on all ints
* Ingress int (snmp index), source ip, dest ip, ip proto, source + dest port, IP ToS

**v7**
* Exclusive to Netflow feature card

**v8**
* Route Based aggregation

**v9**
* Dynamic format
* egress flows
* Uses templates
* Supports m'cast, IPSec, MPLS
* Sampled netflow support
* Can send BGP next hop info (peer or origin AS as well as next hop)
* IP TTL field from IP Header
* ID field
* Packet elngth
* ICMP type and code
* Source + Dest Mac
* VLAN Id on Tx and Rx frames

## EEM

* Create applet
* Syslog event detector
* Variables
* Point to TCL script
* Runs EEM policy when specified event occurs
 * System - cisco policy
 * User - User defined, eg event manager policy FILENAME
* Can detect cli events, coutners, track objects, int counters, OIR, resource events, app events, SNMP, syslog, timer, watchdog
* Core event publishes to EEM server, event subscriber
* EEM Policy director receives from EEM applet and EEM script

## NTP

* SNTP - client only, cannot provide time
* v4 has DNS support for ipv6
* v4 allows mc'ast for v6 NTP sync
* No sync to servers not already sync'd
* NTP Leap - only allowed within month before leap is to happen

## Performance Monitor

* monitors packet flows in network
* similar to netflow and flexible netflow
* Requires cef
* Components
 * Interface - service policy
 * Policy - type performance monitor
 * Class map - matching criteria
 * Flow monitor - type performance monitor
 * Flow record - type performance monitor
 * Flow exporter
* Configure flow record, specify fields, exporter sends to dest
* Then flow monitor, class map, policy map with 1 or more classes, then apply to int

# Processes

## WCCP

1. Client sends HTTP get
2. Router redirects to engine
3. Content engine sees if cachced, if so HTTP response back, if not, to original server
4. Original server replies (if it went there)

## IP SLA

1. Config SLA type
2. Config threshold conditions
3. Config responder
4. Schedule/start
5. Review results

# Config

## DHCP Server

```
int eth1 
 ip address 10.1.1.1 255.255.255.0
 ip helper address 10.1.2.202

ip dhcp excluded-address 10.1.1.0 10.1.1.20

ip dhcp pool subnet1
 network 10.1.1.0 255.255.255.0
 dns-server 10.1.2.203
 default-router 10.1.1.1
 lease 0 0 20
```

## NTP

**Server**
```
int Fa0/0
 ntp broadcast

ntp authentication-key 1 MD5 12948902348 7
ntp authenticate
ntp trusted-key 1
ntp master 7
```

* If 127.127.7.1 seen in show ntp asso, this router lock source

**Static client**

```
ntp server 10.1.1.1
```

**Broadcast client**

```
int E0/0
 ntp broadcast client
```

**Symmetric Active**

```
ntp peer 10.1.1.1
```

## WCCP

```
ip wccp web-cache group-address 239.128.1.100 password Cisco

int Fa0/0
 ip wccp web-cache redirect out

int Fa0/1
 ip wccp redirect exclude in
```

## RITE

```
ip traffic-export profile export-this
 int Fa0/0
 bidirectional 
 mac-address 0018.0fad.df30
 incoming sample one-in-every 30
 outgoing sample one-in-every 100

int Fa0/0
 ip traffic-export apply export-this
```

## RMON

```
rmon event 1 log trap public description Fa0.0RisingErrors owner config
rmon event 2 log trap public description Fa0.0FallingErrors owner config

rmon alarm 11 ifInerrors.1 60 delta rising-threshold 10 1 falling-threshold 5 2 owner config
```

## FTP and TFTP

```
ip ftp username DAVE
ip ftp password DAVETHEFISH
ip ftp passive
exception protocol ftp
exception region-size 65536
exception dump 172.30.19.63
```

```
tftp-server flash:startup-config myconfig.file 11
```

## SCP

```
ip scp server enable
```

## HTTP and HTTPS

```
ip http server
ip http port
ip http access-class
ip http client username
ip http client password
ip http authentication [ aaa | local | enable | tacacs ]

ip http secure-server <--- on 12.4 or later, disables HTTP - can specify cipher suite
```

## BFD

```
iint vlan 10
 bfd interval 50 min_rx 50 multiplier 5

router eigrp 
 bfd all-interfaces

router ospf/isis
 bfd all-interaces

router bgp 49182
 neighbor x.x.x.x fall-over bfd
```

**MUltihop**
```
bfd-template multihop DAVE
 interval min_tx 50 min_rx 50 multiplier 5

bfd map ipv4 4.4.4.0/24 1.1.1.0/24 DAVE <--4.4.4.0 = dest, 1.1.1.0 = source

router bgp 49182
 neighbor x.x.x.x fall-over bfd multihop

ip route static bfd <mh-dest> <mh-source> [unassociate]
``` 
