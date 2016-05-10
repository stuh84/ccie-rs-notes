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

## IPv6 FHRP

### HSRP

* v2 required
* RAs advertised available devices to hosts
* RSs ask for RAs
* RAs periodically m'cast
* Periodic RAs for int link local stop after final RA sent, while at least one virtual IPv6 link local addr conf'd
* MAC of 0005.73a0.0000
* UDP port 2029

### VRRP
* FF02:0:0:0:0:0:0:12
* IP Proto 112
* v3 allows ms timers

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

### Probes

* active, synthetic traffic
* visibility of processing time on device vs transit or on wire
* Near ms precision

* SNMP traps based on threshold/trigger condition
* Histortical data storage
* Set a target with ip sla responder, adds in/out timestamps in payload, measuring cpu time
* ICMP Path echo - traceroute first, then measures responde from each hop, path jitter similar
* UDP Jitter - per direction jitter
* Time sync required
* Shadow router - dedicate source of IP SLA measurement

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

### Top Talkers

* Sorted by either total num of packets or total bytes
* Does not require a collector, placed in a special cache

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

## Enhanced Object Tracking

* Tracking using boolean (track NUMBER list BOOLEAN)
* Threshold weights
* Can track ip route reachability or metric threshold, resolution, timer

## KRON

* Scheduler
* Clock must be set
* Use of conditional means executions tops if error occurs

## Autoinstall over LAN

* Staging router - Must have IP helper on for DNS and TFTP
* Default router - One that Autoinstall requests go to
* Config files - Host specific ("hostname-config.cfg"), default ("router-config"), network ("network-config")
 * Order - network-config, cisconet.cfg, router-config, router.cfg, ciscortr.cfg
* DHCP Server sends siaddr for TFTP in Autoinstall requests
 * Name of file
 * IP of TFTP server
 * Hostname of TFTP server
 * IP of DNS servers
 * IP of staging router
* RARP 
 * RARP requests link local MAC to RARP server
 * Needs to be DC'd to RARP server for Auto install

## Debug Sanity

* Every buffer used, sanity checked when allcoated and when freed

## Buffer Tuning

* show buffers
* buffer middle permanent value
* buffer middle min-free percent
* buffer middle max-free percent
* Reserves buffers on certain platforms

## DHCP On Demand Pool

* ODAP Server
 * Grants addresses as subnets
 * can config size of subnets w prefix length
 * When 1 runs out, another dynamically created
 * Binding tracks use
  * Associated with ODAP manager
  * Binding destroyed when not in used and rreturned to pool
* ODAP manager
 * Allcoates DHCP addesses from DHCP ODAP server out to ODAP client
 * Requests new from server
* Client - Normal client
* DHCP ODAP can work with VPn to assign some sbnets to different VRFs

## DHCP Server Radius Proxy
* Address allocation with radius auth of leases
* Supports DHCP option 60 and 121
* Passes client info to radius
* Response with all required attributes
* Server translates them to options
* Binding synced after Radius auths client
* Can assign from local pool and auth pool for different clients
* Enhancement allows classmane and other optional info (session-timeout, session duration etc)

## DHCP Info Option 82
* relay goes in as giaddr, saying choice of pool
* Option 82 refines, slects sub range of pool
* DHCP snooping use zero for giaddr
* Can be used to define classes

## DHCP Authorized Arp
* Disables dynamic arp learning on int
* Limitation in supporting accurate 1 minute billing
* Static config overrides auth arp

## Cisco Authorative NS

* Listens on port 53
* Answers suing perm/cached entries
* No zone transfers
* Will forward on if not zone authority and ip domain lookup enabled

## ICMP Router Discovery Protocol (IRDP)

* Allows hosts to locate routers as a gw to reach IP based devices on other networks
* IRDP device as router, rotuer discovery apckets gen, as host rx'd

## Other bits

* Small-serevrs - small servers for diag usage, echo back from telnet sessions
 * service tcp-small-servers
 * service udp-small-servers
* Chargen - gens stream of ascii (telnet X.X.X.X chargen)
* Discard - throws away (telnet X.X.X.X discard)
* Daytime - returns sysdate and tim (telnet X.X.X.X daytime)
* UDP all byt day time


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

## Config Archive, Replace and Rollback

* Can be files in flash, filestores, can now be in archive

```
archive
 path URL <-- disk0:myconfig for example
 maximum <1-14> -- default 10
 time-period TIME - optional, auto saves, in minutes
```

* archive config - exec command
* configure replace URL [nolock] [list] [force] [ignorecase] [reverttrigger [error] [timer mins]] [time mins]
 * List - displays list of commands being applied
 * Force - w/o prompt
 * Time - must enter config confirm in this time, or auto reversed
 * Nolock, prevents lock of running config to other uses during operation
* configure revert { now | timer { minutes | idle MINs}}
* show archive

## KRON

```
kron policy-list NAME [conditional]
 cli COMMAND

kron occurence NAME [username] {in [[numdays:]numhours]nummin | at hours:min[[mount[day-ofmonth]} {oneshot | recurring | system-startup}
 policy-list name
```

* show kron schedule

## Autoinstall

**LAN Config for staging rtr**
```
int Fa0/0
 ip address ....
 ip helper-address - separate for each server if required
```

**Cisco RARP server**
```
ip rarp-server ip
arp 192.168.7.19 0800.0900.1834 arpa
```

**Cisco DHCP server**
```
service dhdp
ip dhcp pool ID
 host ADDR MASK/LENGTH
 hardware-address ADDRESS TYPE
 bootfile NAME 
 or 
 option 15 ip address - TFTP
 or
 next-server ADDRESS - siaddr
 option 66 ascii NAME - tftp server name
 default-router in
```

## TCP Keep alives

```
service tcp-keepalive [in] [out]
```
* For telnet

## Coredumps

```
exception dump ip
exception protocol FTP/RCP <---FTP requires FTP un and pw
exception flash <PROCMEM|IOMEM|all> device <erase|no_erase>
exception memory minimum SIZE
exception memory fragment SIZE
```

## IPv6 HSRP

```
int Fa0/0
 standby version 2
 standy NUM upv6 {link-loc-addr | autoconfig}
 standby NUM preempt
 standby NUM priority
```

## VRRP v3 (for v4 and v6

```
int Fa0/0
 vrrp NUM address family {ipv4 | ipv6}
  address ADDR {primary | secondary}
  description DEScf
  match address <--- matches 2nd in advertsiement packet against conf'd addr
  preemept delay min SEC
  priority value
  timers advertise MS
  vrrpv2 - v2 compat
  vrrs leader VRRS-LEADER-NAME
```

## DHCP ODAP

```
ip address-pool dhcp-pool

int Fa0/0 
 peer default ip address dhcp-pool NAME

ip dhcp pool pool-name
 vrf NAME
 origin {dhcp | aaa | ipcp} [ subnet SIZE initial SIZE [autogrow SIZE]]
 utilization mark low percentage
 utilization mark high percentage
```

* Obtain with IPCP 

```
ip dhcp pool NAME
 import all 
 origin ipcp
```

## DHCP RADIUS PROXY

```
service dhcp
aaa new model
aaa group server radius NAME
  server IP auth-port PORT acct-port PORT
aaa auth network NAME group NAME
aaa account network NAME start-stop group NAME

int Fa0/0
 ip addres ....

ip dhcp use class [aaa]
ip dhcp pool NAME
 accounting NAME
 auth NAME
 auth shared-password STRING
 auth username STRING
```

**Enhanced**

```
network number mask [secondary]/prefix length [secondary]
class NAME
address range START END
```

## DHCP Authorized Arp

```
int Fa0/0
 arp authorized
 arp timeout seconds {don't set less than 30s}
```

## Cisco Authorative NS

```
ip dns server
ip name-server ADDRESS --- forwarders or other NS
ip dns server quueue LIMIT {forwarder LIMIT}
ip host [vrf NAME] [view NAME] hostname address
ip dns primary DOMAIN soa
 primary-server-name mail-box-name [refresh retry expire-ttl min-ttl]
ip host NAME ns SERVER-NAME
ip dns sppofing IP - responds to DNS queries with conf'd IP when queried for anything but itself
```

## Netflow top talkers

```
ip flow cache entries <1024-524288>
ip flow cache timeout active mins VALUE <1-60, default 30>
ip flow-cache timeout inactive (15s default, 10-600)

ip flow-top-talkers
 top num (1-200)
 sort-by [bytes | packets]
 cache-timeout MS
 match source-address IP/NN|ip-mask
```

* show ip flow top-talkers

## IRDP

```
no ip routing
ip gdp irdp

int Fa0/0
 ip irdp
 ip irdp multicast (sent to 224.0.0.1)
 ip irdp holdtime NUM
 ip irdp max-advert-interval NUM
 ip irdp min-advert-interval NUM
 ip irdp preference NUM
 ip irdp address XXX num (address + pref to proxy advertise)
```

## ICMP Settings

```
[no] ip unreachables
ip icmp rate-limit unreachable [df] ms
ip redirects
ip mask-reply - can send mask of network
```

## TCP performance parameters

```
ip tcp syn-wait-time SEC (default 30s)
ip tcp path-mtu-discovery [age-timer {minutes|ifninte}] - default 10 mins
ip tcp selective-ack - Allows acking multiple packets in window (some lost, not all)
ip tcp timestamp
ip tcp chunk-size characters - max read size for telnet or rlogin
ip tcp window-size BYTES <--- must be more than 65535 for scaling support, otherwise 4128 default
ip tcp ecn
ip tcp queuemax packets

int Fa0/0
 ip tcp adjust-mss SIZE
 ip mtu BYTES
```
