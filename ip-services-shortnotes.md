# ARP, Proxy ARP, Reverse ARP, BOOTP, DHCP

* Learn another hosts MAC - ARP, Proxy ARP
* Host discovers its own IP - RARP, BOOTP, DHCP

## ARP and Proxy ARP

* RFC826 - ARP
* RFC 1027 - Proxy ARP

**ARP**
* ARP destination of 255.255.255.255
* No IP header, source and dest IP in same relative position
* Protocol type 0x0806

**Proxy ARP**
* Same as ARP messages but for MAC not on local subnet
* Router can issue Proxy ARP request on behalf of target
* LAN broadcast, router replies with own MAC
* Before DHCP, proxy ARP relied on, hosts used default masks in networks

## RARP, BOOTP, DHCP

* For host to dynamically learn IP
* Broadcast to begin discovery
* All rely on server hearing request

### RARP

* ARP messages used
* ARP request lists its own MAC as target
* Target IP of 0.0.0.0
* Preconf'd RARP server (same subnet as client) receives request
* Server looks in config
* ARP reply with configured IP in source IP field

### BOOTP

* RFC 951 defined messages
* Commands encap'd in IP and UDP header
* Can go to other subnets
* Can assign subnet mask, def gw, DNS and IP of boot server
* Preconfig required

### DHCP

* Built on BOOTP format
* Messaging allows future changes
* No predefinition of MACs required
* Leasing of IPs
* Pooling of IPs
* Dynamic registration of client DNS FQDN
* LAN broadcasts forwarded to DHCP server by changing request's dest to match DHCP server (relay)
* In relay, routers own IP in gw IP address (giaddr)
* Source address change to LAN broadcast, so reply from server broadcast on LAN

Config DHCP relay with

```
ip helper-address 10.1.2.202
```

Router can be DHCP server
1. Config DHCP pool
2. Config excluded IPs (eg router IP)
3. Disable DHCP conflict logging, or configure DHCP database agent

Pool includes subnet, default gateway and lease time, can have DHCP domain name and options

DHCP address conflicts can log to server. Disabled with **no ip dhcp conflict-logging**, or confing DHCP db agent server with **ip dhcp database**

```
int eth1
 ip address 10.1.1.1 255.255.255.0
 ip helper-address 10.1.2.202

ip dhcp excluded-address 10.1.1.0 10.1.1.20

ip dhcp pool subnet1
 network 10.1.1.0 255.255.255.0
 dns-server 10.1.2.203
 default-router 10.1.1.1
 lease 0 0 20 (days, hours, minutes)
```

# HSRP, VRRP and GLBP

## HSRP

* Virtual IP and virtual MAC on active router
* Standby routers listen for hellos
* 3s hello, 10s dead by default
* Highest priority wins
* Premption disabled by default
* Default priority of 100
* Tracking
* 255 groups per int
* Virtual MAC of 0000.0C07.ACXX (XX is hex of group)
* Virtual IP must be in same subnet as int IP
* Virtual IP not on router int
* Clear text and MD5 auth
* One active router
* Load sharing with multiple groups
* Cisco proprietary

```
track 13 int Se0/0.1 line-protocol

int Fa0/0
 ip address 10.1.1.1 255.255.255.0
 standby 21 ip 10.1.1.10
 standby 21 priority 110
 standby 21 preempt
 standby 21 track 13
 standby 22 10.1.1.22
 standby 22 track 13
```

**show standby**

## VRRP

* RFC 3768
* Same as HSRP on Ciscos except the following points
* Multicast MAC of 0000.5E00.01xx (hex VRRP group)
* IOS obecting tracking rather than internal tracking
* Defaults to preempt
* Master rather than Active
* Group IP is interface IP on one of the routers

## GLBP

* Cisco proprietary
* Hosts target same IP, different virtual MAcs for different rotuers
* GLBP AVG (Active Virtual Gateway) assigns each rotuer unique MAC
* MAC of 0008.B40X.xxyy, X.xx being 10-bit GLBP group number, YY per router
* GLBP AVG replies to ARP with 1 of 4 virtual MACs
* 1024 groups per interface, four hosts per group

# NTP

* v3 - RFC 1305 syncs time of day clocks with common source
* NTP client mode on most routers/switches
* NTP defines messages used and algorithms to adjust
* Symmetric active (mutual syncing with another host)
* Servers reference others to find most accurate
* Atomic clocks and GPS provide stratum 1

Server config: -
```
int Fa0/0
 ntp broadcast

ntp authentication-key 1 MD5 13457348957 7
ntp authenticate
ntp trusted-key 1
ntp master 7

If 127.127.7.1 seen in show ntp ass, implies this router is clock source
```

Static client: -

```
ntp authentication-key 1 MD5 13457348957 7
ntp authenticate
ntp trusted-key 1
ntp clock-period 23420384 # Auto gen'd during sync
ntp server 10.1.1.1
```

Broadcast client: -
```
int E0/0
 ntp broadcast client
```

Symmetric active client: -
```
ntp authentication-key 1 MD5 13457348957 7
ntp authenticate
ntp trusted-key 1
ntp clock-period 23420384 # Auto gen'd during sync
ntp peer 10.1.1.1
```

# SNMP

* SNMP agents have a database (MIB), holds data how device operates

Four functional areas
* Data definition - Structure Of Management Information (SMI), how to define an agent or manager
* MIBs - COnform to SMI version, some standard, some proprietary
* Protocols - Messages
* Security and admin - How to secure data exchange

Versions: -
* 1 - SMIv1, simple auth with comminuties, used MIB-1 originally
* 2 - SMIv2 - removed requirement for communities, added GetBulk and Inform, began with MIB-II originally
* 2c - Same as v2 except communities
* 3 - Same as v2, but better security

## SNMP protocol messages

* RFC 3416 defines how manager/agents communicate
* Manager Uses three different messages to get data
* SNMP response from agent, supplies MIB data
* UDP
* SNMP response acks receipt

Messages: -

|Name|Initial Version|Reply With|Typically sent by|Description|
|----|---------------|------------|-----------------|-----------|
|Get|1|Response|Manager|Request for single variables value|
|GetNext|1|Response|Manager|Requests next single mib leaf variable in tree|
|GetBulk|2|Response|Manager|Multiple MIB variables in one request, helps with routing tabels etc|
|Response|1|Is a response|Any|Responds to Get and Set reqs|
|Set|1|Response|Manager|Set variable to a value|
|Trap|1|Response|Agent|Unsolicited info to manager, no reply|
|Inform|1|Response|Manager|Used between managers|

Successive GetNexts or GetBulks with MIB walk

## MIBs

* Standard generic MIBs in v1 and v2
* RFC 1156 - MIB-1
* RFC 1213 - MIB-2
* MIB2 created between release of v1 and v2
* After MIB-II, IETF stopped working on standard MIBs, set other groups to create MIBs for their tech (hundreds of standardized MIBs)
* RMON MIB (RFC2819) allows SNMP set, capite packets, stats, monitor thresholds etc

## SNMP Security

* v3 added auth and ecryption
* SHA and MD5 create message digest of each protocol message
* Encrypted with DES and AES (AES not in original v3 specs)

```
access-list 33 permit 192.168.1.0 0.0.0.255
snmp-server community public RW33
snmp-server location HERE
snmp-server contact FECK@FECK.com
snmp-server chassis-od 2511
snmp-server enable traps snmp
snmp-server enable traps hsrp
snmp-server enable traps bgp
snmp-server host 192.168.1.100 public
```

# Syslog

* Ciscos dont log to non-volatile memory by default
* Enable above with **logging buffered**
* RFC5424 for syslog
* Middle ground between manual log parsing and SNMP
* Real time event noficiation
* Sends messages to syslog servers
* UDP 514 default
* All events that enter log to syslog server by default
* Clear text

1. Install syslog server
2. Configure to send with **logging host**
3. Config severity levels with **logging trap** then 0-7

# WCCP

* Routers use caching engines
* Differs from web proxying (hosts unaware0

Following process
1. Client sents HTTP get
2. Router sees above, redirects to content engine
3. Content engine looks if cached
 a. If cached, HTTP response back
 b. If not, content engine sends Get to original server
4. If 3b taken, server replies to client

* UDP 2048
* Pool of content engines possble (cluster), they are aware of each other
* Pool communicates in WCCP messages
* 32 can communicate with one router using WCCPv1
* If more than one present, lowest IP elected as lead engine
* Info on cluster provided to engines by router in a list
* Lead content engine can use above to distribtue traffic
* If v1, only one router
* v2 has multiple routers and engines in a service group
* v1 for HTTP port 80 only, v2 others

v2 over v1
* Supports TCP and UDP outside of 80 (FTP, FTP proxy, Real Audio, video, telephony etc)
* Segment caches by protocol or protocols, priority system to decide how
* Multicast support
* Multiple routers (32 per cluster)
* MD5 security **ip wccp password**
* Load distribution
* Transparent error handling

v2 by default in IOS. Conf'd globally, affets all ints. Routers and engines can be in more than one service group

```
ip wccp web-cache group-address 239.128.1.100 password Cisco

int fa0/0
 ip wccp web-cache redirect out

int fa0/1
 ip wccp redirect exclude in
```

WCCP ACLs exist, can filter for certin clients **ip wccp web-cache redirect-list**. **ip wccp web-cache group-list** says what traffic router accepts from content engines

# IP SLA

* Used to be Service Assurance Agent (SAA), and previously RTR (Response Time Report)
* Probes network
* Built around source-responder

Can measure following: -

* Delay (one way and RTT)
* Jitter (directional)
* Packet loss (directional)
* Packet sequencing
* Path (per hop)
* Connectivity (UDP echo, ICMP echo, ICMP echo, TCP connect)
* Server or website download time
* Voice quality metrics (MOS)

Steps required: -

1. Config SLA type
2. Configure threshold conditions
3. Configure responder
4. Schedule/start
5. Review results

Must delete to reconfigure options, also deletes schedule.

Supports MD5 with **ip sla key-chain**

Set up responder with **ip sla monitor responder**

```
ip sla monitor 1
 type udpEcho dest-ipaddr 200.1.200.9 dest-port 1330
 frequency 5
 exit
ip sla monitor schedule 1 life 86400 start-time now
```

Verify with 
**show ip sla monitor statistics**  
**show ip sla monitor configuration**  

# Implementing Netflow

* Currentl on v9, renamed to Cisco Flexible Netflow
* Used to be seven fixed tuple identifying flow
* Now can have as mahy as you want

Components are: -

* Records - Key fields (predefined or user-defined), dest IP, source IP etc
* Flow monitors - Applied to int, includes records, cache and optional flow export
* Flow exporters - Export cached flow to outside systems
* Flow samplers - Reduces loads on Netflow devices, sample size definable, between 1:2 to 1:32768 packets

```
flow export ipv4flowexport
 destination 192.168.1.110
 dscp 8
 transport udp 1333

flow monitor ipv4flow
 description Monitors all v4 traffic
 record netflow ipv4 original-npurt
 cache timeout inactive 600
 cache timeout active 180
 cache entries 5000
 statistics packet protocol

int F0/0
 ip address 192.168.39.9 255.255.255.0
 ip flow monitor ipv4flow input
```

Verify: -
**show flow record**  
**show flow monitor**  
**show flow exporter**  
**show flow interface**  

# Router IP traffic Export (RITE)

* Exports packets to VLAN or LAN
* Only for traffic received on multiple WAN/LAN ints simultaneously (if device being targeted in DoS)
* Used for IDS
* Directly copies packets to MAC of IDS
* Forwards inbound by default, can do outbound or both
* Filter on number of forwarded packets (ACL, one-in-n packets)

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

# EEM

Detects events ,provides notification of events

Detectors supported:

* SNMP Objects
* Matching syslog patterns
* Monitoring coutners
* Timers (time-of-day, cron, watchdog etc)
* Screening CLI for match
* Hardware insertion/remvoal
* Routing table change
* IP SLA/netflow events
* Generic On-Line Diagnostic (GOLD) events
* Many others

Actions: -

* Generated prioritized syslog messages
* Reload router
* Switch to secondary sup
* Generate SNMP traps
* Set/modify counter
* Execute IOS command
* Send email
* Request system info
* Read or set track object
* IOS CLI or TCL

```
event manager applet CLI-cp-run-st
 event cli pattern "wr" sync yes
 action 1.0 syslog msg "$_cli_msg Command Execited"
 set 2.0 _exit_status 1
 end
```

# Implementing Remote Monitoring

* Configure thresholds based on SNMP objects (monitor device performance)
* Alarms and events
* Event is number, user config'd threshold for SNMP object
* Events track (CPU, errors etc), set rising and falling thresholds
* Tell RMON alarm to trigger when thresholds cross
* Alarm is what it does (logs event, sends trap)
* For trap, need SNMP community of server
* Event and alarm number locally signficiant

In the below: -

* Error counters
* For first, RMON looks for delta rise in 60 seconds, falling five errors per 60 seconds
* In second, thresholds absolute

```
rmon event 1 log trap public description Fa0.0RisingErrors owner config
rmon event 2 log trap public description Fa0.0FallingErrors owner config
rmon event 3 log trap public description Se0.0RisingErrors owner config
rmon event 4 log trap public description Se0.0FallingErrors owner config

rmon alarm 11 ifInerrors.1 60 delta rising-threshold 10 1 falling-threshold 5 2 owner config

rmon alarm 20 ifInerrors.2 60 absolute rising-threshold 20 3 falling-threshold 10 4 owner config
```

**show rmon alarm**  
**show rmon event**  

# FTP on a router

* Can be TFTP server
* Can't be FTP server
* Transfer files with **ip ftp** command and options
* **ip ftp username**
* **ip ftp password**
* **ip ftp source-interface**
* **copy startup-config ftp:**
* Can send dump in crash

```
ip ftp username Dave
ip ftp password DaveTheFish

exception protocol ftp
exception region-size 65536
exception dump 172.30.19.63

ip ftp passive # If required
```

# TFTP server

Enable with **tftp server**, can specify memory region, and ACL for hosts with access. 

**tftp-server flash:startup-config myconfig.file 11**


# SCP

* Requires AAA
* **ip scp server enable**

# HTTP and HTTPS

* **ip http server**
* **ip http port**
* **ip http access-class**
* **ip http client username**
* **ip http client password**
* **ip http authentication [ aaa | local | enable | tacacs ]**

* **ip http secure-server**
* On 12.4 or later, disables HTTP access
* Can specify cipher suite
* Show status and cipher suite with **show ip http server secure status**

# Implementing Telnet

* On VTY, login command (eg **login local**)
* Uses port 23, can be rotary (rotary 33 would be ports 3033, 4033 etc)

# Implementing SSH

1. Configure **hostname**
2. Configure **ip domain-name**
3. **crypto key generate rsa**
4. **transport input ssh**

Can uses rotaries too
