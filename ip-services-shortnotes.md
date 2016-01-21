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
