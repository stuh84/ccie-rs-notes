# Ethernet Basics

### Basic switch port config

```
speed { auto | 10 | 100 | 1000 }
duplex { auto | half | full }
```

## Basic SPAN Configuration

```

monitor session 1 source interface <SOURCE-IF>
monitor session 1 destination interface <DESTINATION-IF>
```

## Complex SPAN configuration

```
monitor session 1 source interface Fa0/18 rx
monitor session 1 source interface Fa0/19 tx
monitor session 1 filter vlan 1-3, 229
monitor session 1 destination interface Fa0/24 encapsulation replicate
```

## RSPAN Configuration

### Source switch

```
vlan 199
 remote span

monitor session 3 source vlan 66-68 rx
monitor session 3 destination remote vlan 199
```

### Destination switch

```
vlan 199
 remote span

monitor session 63 source remote vlan 199
monitor session 63 destination interface fa0/24
```

## ERSPAN configuration

### Source ASR

```
monitor session 1 type erspan-source
 source interface gig0/1/0 rx
 no shutdown
 destination
  erspan-id 101
  ip address 10.1.1.1
  origin ip address 172.16.1.1
```

### Destination 6509

```
monitor session 2 type erspan-destination
 destination interface Gi2/2/1
 no shutdown
 source
  erspan-id 101
  ip address 10.1.1.1
```

### Verification

```
show monitor session 1
```

## Basic VSS configuration

Same virtual switch domain needs creating, referenced by a number between 1 and 255. One switch must be switch 1, another switch 2.

### Switch 1:
```
switch virtual domain 10
switch 1
```

### Switch 2:

```
switch virtual domain 10
switch 2
```

### Switch VSL Port Channel

Switch 1

```
int port-channel 5
 switchport
 switch virtual link 1
```

Switch 2
```
int port-channel 10
 switchport
 switch virtual link 2
```

* Afterwards, interface will be up/down until reboot

Next, convert switches with

```
switch convert mode virtual
```

### VSS Verification

* show switch virtual - will show switch domain number, switch number and role
* show switch virtual role - Peer 0 is local switch
* show switch virtual link - Shows VSL info
* show switch virtual link port-channel - Show port channel info

# Virtual LANs and VLAN Trunking

## VLAN Database Mode

```
vlan database
 vlan 21
```

* show current - VLANs availble to IOS when switch in VTP server mode
* show proposed - VLANs waiting
* apply - Applies changes
* abort - Aborts changes
* reset - Don't make changes but stay in VLAN DB mode

## Config mode

````
int fa0/3
 switchport access vlan 22
```

* show vlan brief - Shows ports in VLANs (access only)

* switchport access vlan 31 would create VLAN 31
* vlan 32 - creates vlan 32

## Operational state of VLANs

* state suspend - Valid in db and config, suspends VLAN globally (i.e. vtp)
* shutdown - Shuts down locally

## Private VLANs

```

vlan 199
 private-vlan isolated

vlan 101
 private-vlan community

vlan 100
 private-vlan primary
 private-vlan association 101,199

show vlan private-vlan shows types of VLANs

int Fa0/1
 switchport mode private-vlan host
 switchport private-vlan host-association 100,101

int Fa0/13
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100,101,199

int vlan 100
 private-vlan mapping 101,199
 ip address 10.1.1.1 255.255.255.0
 ```

## VLAN Trunking

### ISL and 802.1q config

* switchport - toggles and interface to be switched or routed
* switchport mode - sets DTP negotiation parameters
* switchport trunk - Sets trunking parameters
* switchport access - Sets nontrunk parameters


* show int trunk - Summary of trunk info
* show int <int> trunk - Trunking details for particular interface
* show int <int> switchport - Trunking and nontrunking details for interface
* show dtp - Shows DTP information


### Allowd, Active and Pruned VLANs

* switchport trunk allowed - allows vlans
* VTP can prune VLANs
* show int trunk lists vlans that are:
* Allowed - Admistratively configured to be allowed (or all by default)
* Allowed and active - must be allowed, VLAN configured on switch and in active state. With PVST+, STP instance actively running on this trunk for VLANs in this list
* Active and not pruned - Subset of above, with any VTP pruned or VLANs considered blocked by PVST+ removed.

### Trunking config

* switchport mode and switchport nonegotiate define whether DTP attempts to trunk, and what rules when attempts made.
* switchport mode trunk - Always trunks this side, uses DTP to help other side trunk
* switchport mode trunk; switchport nonegotiate - Always trunks, no DTP send
* switchport mode dynamic desirable - Sends DTP messages hoping to trunk
* switchport mode dynamic auto - Prefers access but will trunk based on other side
* switchport mode access - Never trunks, sends DTP to help other side
* switchport mode access; switchport nonegotiate - Never trunks, no DTP sent
* switchport trunk encapsulation - sets trunking type, also includes option for negotiating the type

### Configuring trunks on routers

Use encapsulation dot1q vlan-id native on sub-int, allows to recognise both untagged and cos-marked frames with particular vlan-id

## QinQ Tunneling

```

int Fa0/1
 switchport mode dot1q-tunnel
 switchport access vlan 5
 l2protocol-tunnel cdp
 l2protocol-tunnel lldp
 l2protocol-tunnel stp
 l2protocol-tunnel vtp
```

show int fa0/1 would then show admin and operational mode of tunnel

## VLAN Trunking Protocol

* vtp domain <DOMAIN-NAME>
* show vtp status - shows domain, pruning mode, version, last updated etc
* vtp password - sets password, taken into account for MD5 hash of the VLAN database
* vtp mode - Server, transparent, client, off (v3 only)
* vtp version - 1 and 2 will apply to all switches in domain. v3 has to be done on each switch, and must have domain name set
* vtp pruning - enables/disables pruning
* vtp interface - Specifies identifier of updates, by default lowest number VLAN SVI

## Configuring PPPoE

```

int Fa0/0
 ip address 192.168.100.1 255.255.255.0
 ip nat inside

int Fa0/1
 no shutdown
 pppoe-client dial-pool-number 1

int dialer1
 mtu 1492
 ip tcp adjust-mss 1452
 encapsulation ppp
 ip address negotiated
 ppp chap hostname Username@ISP
 ppp chap password Password4ISP
 ip nat outside
 dialer pool 1

ip nat inside source list 1 interface dialer 1 overload

access-list 1 permit 192.168.100.0 0.0.0.255
ip route 0.0.0.0 0.0.0.0 dialer1
```

Verify with show pppoe session, debug with debug pppoe data/errors/events/packets

# Spanning Tree Protocol

## STP Config and Analysis

* show spanning-tree root - shows the root bridge, will also show "This bridge is the root" if on root switch
* spanning-tree vlan 1 priority 28672 - Changes root priority

```
int Fa0/1
 spanning-tree vlan 1 cost 100 - Changes port cost when done in port context
```

* Can also use the spanning-tree vlan vlan-id root { primary | secondary } [ diameter diameter] command. Diameter lowers Hello, ForwardDelay and MaxAge timers. This command is a macro though, not something placed into config.

Above command sets priority to 24576 if current root priority is larger than 24576 (or 24576 but higher mac). If current priority lower, set switches priority to 4096 below root. Secondary priority is always 28672

### MST configuration

```
spanning-tree mst configuration
 name <name>
 revision <number>
 instance <n> vlan <VLANS>
```

* show current will show current MST config when in this context, show pending shows future.

```
spanning-tree mode mst
spanning-tree mst 0 priority 0
spanning-tree mst 1 priority 4096
```

* change cost on port with spanning-tree cost mst

If VTPv3 used, domain and MST region can match.

```
vtp mode server mst
vtp primary mst
```

Above two commands mean that commands in spanning-tree mst config are mirrored on all switches in VTPv3 domain

### PortFast

```
spanning-tree portfast
spanning-tree portfast default
spanning-tree portfast disable
spanning-tree portfast trunk (For trunks connected to hosts)
```

### Root Guard, BPDU Guard, BPDU Filter

```
spanning-tree bpduguard enable
spanning-tree portfast bpduguard default

spanning-tree guard root

spanning-tree portfast bpdufilter default
spanning-tree portfast bpdufilter disable
```

Be careful with per port and global bpdufilter, global 10 hellos sent first then stopped. Port can still receive BPDUs. On an interface, bpdus stopped and received

### UDLD

```
udld { enable | aggressive } - global command
udld port [ aggressive] - per port
show udld neighbors
```

### Loop Guard

```
spanning-tree loopguard default
spanning-tree guard loop - Per port
```

### Bridge Assurance

```
spanning-tree bridge assurance - global
spanning-tree portfast network - per port
```

### Load Balancing across Port Channels

port-channel load-balance type - Set type of load balancing

### Port-Channel Discovery and Config

Must have same of the following: -

- Same speed and duplex settings
- Same operating mode (trunk, access, dynamic)
- If not trunking, same access VLAN
- If trunking, same trunk type, allowed VLANs and native VLAN
- No span ports

int Port Channel automatically added to config when Port Channel created. Inherits config of first interface added.

Config changes on Port Channel int only take effect on non-suspended members.

Following guidelines recommended: -


- Do not create port channel manually before bundling physical ports under it, let switch do it automatically
- Make sure to remove port channel interface from config so no issues when port channel with same number recreated later
- Physical port config needs to be identical
- Correct physical port config first, not port channel
- Port Channel int can be l2 or l3, depending on whether physical bundled ports configured as L2 or L3. Once port channel created, not possible to change it to other mode without recreating. Possible to combine L2 and L3 ports in a port channel
- When sorting out err-disable, shut down physical interfaces and port channel interface. Only then try to reactivate. If problem persists, remove port channel config altogether and recreate it

Configure ports to be in manual port channel with "channel-group number mode on"

```
 channel-group number auto/desirable - PAgP
 channel-group number active/passive - LOOPBACK0-IP

channel-protocol pagp/lacp makes only protocol psecific commands available
```

### LLDP

```
lldp run - globally enable
lldp transmit - per port
lldp receive - per port

lldp holdtime
lldp reinit
lldp timer
```

# IP Addressing

## Static NAT

```

int E0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside

int S0/0
 ip address 200.1.1.251 255.255.255.0
 ip nat outside

ip nat inside source static 10.1.1.2 200.1.1.2
ip nat inside source static 10.1.1.1 200.1.1.1
```

## Dynamic NAT

```

int E0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside

 int Se0/0
  ip address 200.1.1.251 255.255.255.0
  ip nat outside

ip nat pool fred 200.1.1.1 200.1.1.2 netmask 255.255.255.252
ip nat inside source list 1 pool
access-list 1 permit 10.1.1.0 0.0.0.255
```

## Dynamic PAT

As above but...

```
no ip nat inside source list 1 pool fred
ip nat inside source list 1 pool fred overload
```

## Dynamic v6 Tunneling config

```

R2
int tun23
 ipv6 address 23::2/64
 tunnel source lo0
 tunnel destination 3.3.3.3
 tunnel mode ipv6ip

R3
int tun32
 ipv6 address 23::3/64
 tunnel source lo0
 tunnel destination 2.2.2.2
 tunnel mode ipv6ip
```

# IP Services

## DHCP Helper

```
ip helper-address 10.1.2.202
```

## DHCP Server

```

int Eth1
 ip address 10.1.1.1 255.255.255.0
 ip helper-address 10.1.2.202

ip dhcp excluded-address 10.1.1.0 10.1.1.20

ip dhcp pool subnet1
 network 10.1.1.0 255.255.255.0
 dns-server 10.1.2.203
 default-router 10.1.1.1
 lease 0 0 20 # 0 days, 0 hours, 20 minutes
```

## HSRP

```

track 13 interface Se0/0.1 line-protocol

int Fa0/0
 ip address 10.1.1.1 255.255.255.0
  standby 21 ip 10.1.1.21
  standby 21 priority 105
  standby 21 preempt
  standby 21 track 13
  standby 22 ip 10.1.1.22
  standby 22 track 13

show standby shows state
```

## NTP

Configuration: -
* R1 - Server
* R2 - NTP static client
* R3 - NTP broadcast client
* R4 - NTP symmetric active mode

```
R1:
int Fa0/0
 ntp broadcast # Broadcasts NTP updates on this interface

ntp authentication-key 1 md5 15514141414 7
ntp authenticate
ntp trusted-key 1
ntp master 7 # CLock is syncs with stratum level 7

If 127.127.7.1 seen in show ntp associations, implies this router is NTP clock source

R2:

ntp authentication-key 1 md5 15514141414 7
ntp authenticate
ntp trusted-key 1
ntp clock-period 17208144 # Auto generated as part of sync process
ntp server 10.1.1.1

R3:

int E0/0
 ntp broadcast client

R4:

ntp authentication-key 1 md5 15514141414 7
ntp authenticate
ntp trusted-key 1
ntp clock-period 17208144 # Auto generated as part of sync process
ntp peer 10.1.1.1
```

## SNMP

```

access-list 33 permit 192.168.1.0 0.0.0.255
snmp-server community public RW33
snmp-server location B1
snmp-server contact helpdesk@dooda.com
snmp-server chassis-od 2511_AccessServer_Dave
snmp-server enable traps snmp
snmp-server enable traps hsrp
snmp-server enable traps bgp
snmp-server host 192.168.1.100 public
```

## syslog


Configure as such: -

1. Install syslog server
2. Configure to send on router with **logging host** command
3. Configure which severity levels to send with **logging trap** command, levels from 0-7

## WCCP

```

ip wccp web-cache group-address 239.128.1.100 password cisco # Service, group-address for communication and md5 password

int Fa0/0
 ip wccp web-cache redirect out
int fa0/1
 ip wccp redirect exclude in
 ```

## IP SLA

 MD5 auth supportd with **ip sla key-chain** command.

Global config command of **ip sla monitor responder** can be used. On originating router, do the following (example): -

```
ip sla monitor 1
 type udpEcho dest-ipaddr 200.1.200.9 dest-port 1330
 frequency 5
 exit
ip sla monitor schedule 1 life 86400 start-time now
```

Following commands for verification: -
```
show ip sla monitor statistics  - shows results of SLAs configured
show ip sla monitor configuration - Shows what has been configured
```

## Netflow

```

flow exporter ipv4flowexport
 destination 192.168.1.110
 dscp 8
 transport udp 1333

flow monitor ipv4flow
 description Monitors all IPv4 traffic
 record netflow ipv4 original-input
 cache timeout inactive 600
 cache timeout active 180
 cache entries 5000
 statistics packet protocol

interface Fa0/0
 ip address 192.168.39.9 255.255.255.0
 ip flow monitor ipv4flow input
```

## RITE

```

ip traffic-export profile export-this
 int Fa0/0
 bidirectional
 mac-address 0018.0fad.df30
 incoming sample one-in-every 20
 outgoing sample one-in-every 100

int fa0/1
 ip traffic-export apply export-this
```

## EEM

```

event manager applet CLI-cp-run-st
 event cli pattern "wr" sync yes
 action 1.0 syslog msg "$_cli_msg Command Executed"
 set 2.0 _exit_status 1
 end
```

## RMON

```

rmon event 1 log trap public description Fa0.0RisingErrors owner config

rmon event 2 log trap public description Fa0.0FallingErrors owner config

rmon event 3 log trap public description Se0.0RisingErrors owner config

rmon event 4 log trap public description Se0.0FallingErrors owner config

rmon alarm 11 ifInErrors.1 60 delta rising-threshold 10 1 falling-threshold 5 2 owner config

rmon alarm 20 ifInErrors.2 60 absolute rising-threshold 20 3 falling-threshold 10 4 owner config

```

Monitor activity with show rmon alarm and show rmon event

## FTP client

```

ip ftp username Dave
ip ftp password DaveTheFish
!
exception protocol ftp
exception region-size 65536
exception dump 172.30.19.63
```

## TFTP Server

Enable TFTP using **tftp-server** command, which has several arguments. Can specify memory region (typically flash), file name, ACL for which hosts have access to file. Example would be **tftp-server flash:c1700-advipservicesk9-mz.124-23.bin alias supersecretfile.bin 11**

## SCP Server

```
ip scp server enable
```

## HTTP and HTTPS

Enable HTTP with **ip http server**. Specify port with **ip http port**. Restrict with **ip http access-class. **Specify unique username and password with **ip http client username ** and **ip http client password** commands. Can also auth with others **ip http authentication [ aaa | local | enable | tacacs ]**

Enable HTTPS using **ip http secure-server. **When configured on 12.4 IOS or later, automatically disables HTTP access. Can specify cipher suite of choice too. The **show ip http server secure status** shows what is in use for cipher suites and other info.

## telnet

**login** command or a variation (eg login local) configured under VTY line. Can use rotary groups

## SSH


1. Configure a hostname using **hostname** command
2. Configure a domain name using **ip domain-name** command
3. Configure RSA keys using **crypto key generate rsa**
4. Configure terminal lines to permit SSH with **transport input ssh**


Can also use rotary lines like telnet.

# IP Forwarding

## CEF

- ip cef - Enables cef for all interfaces
- ipv6 cef - activates v6 CEF support, v4 CEF must be active to enable
- no ip route-cache cef - disables CEF on an interface

## CEF Load Sharing

 **ip load-share { per-destination | per packet }**. - Per interface

 ID (read on polarization) can be specified in **ip cef load-sharing algorithm** and **ipv6 load-sharing algorithm**. Also used to select algorithm.

 **mls ip cef load-sharing** on Cat6500 platforms

 ## VLAN Allocation policy

 On Cat switches support extended VLAN range, depending on setting of **vlan internal allocation policy { ascending | descending }**. If ascending, internal VLANS allocated from 1006 and up. If descending, 4094 and down. Important for routed interfaces (using internal VLANs on MLS)

  **show vlan internal usage**


## L3 Port Channel

Port channel can be L3, picks up the no switchport command from physical interfaces that are placed into bond. Cannot change once configured, need to remove all interfaces then reconfigure.

## Policy routing

The ** route-map **specified in this command is what decides on the routing.

Either no match or a deny in route-map statement causes packets to be forwarded by destination routing

Can match in route map with ** match ip address **or **match ipv6 address** or packet length (**match length**)


- set ip next-hop or ipv6 next hop - Next hop must be in connected subnet, forwards to first address in list for which associated interface is up
- set ip(v6) default next-hop - Same as above, except standard routing done first (default route ignored)
- set interface - Forwards on first interface in list that is up, recommended only for P2P interfaces
- set default interface - As above, tries to route first
- set ip df - Sets DF bit (0 or 1)
- set ip(v6) precedence - Set IPP bits
- set ip tos - Sets ToS bits, can be decimal value or ASCII name

# RIP

## Broadcast rather than multicast advertisement

**ip rip v2-broadcast**

## Split Horizon

Split Horizon enabled by default on Cisco RIPv2 interfaces, except FR and ATM. Verified with **show ip interface**, SH with Poisoned Reverse not in Cisco RIPv2

## Show RIP database

```
show ip rip database
```

## Enabling RIP and effects of autosummarization

```

router rip
 version 2
 network 172.31.0.0
```

Turn off auto summary with **no auto-summary**

## Authenticaiton

Authentication can be clear text or MD5, enabled per interface. Multiple keys allowed, grouped as a keychain, can make available at certain times. Enable with **ip rip authentication key-chain name**. Lowest sequence number used if multiple keys valid. Type chosen with **ip rip authentication mode { text | md5 }**


## Split Horizon

```
int Fa0/0
 ip split-horizon
```

Default on interfaces except FR and ATM when configured with an IP on their interfaces

## Offset lists

Adds a route metric, refers to ACL to match routes, adds specified offset, specified in/out direction, and optionally an interfaces

## Filtering routes

Use **distribute-list** under router ip, preference ACL or prefix-list. In or out, or per interface

## RIPng


- Auth or enryption by IPsec not supported
- Split horizon can only be disabled on a per-process basis
- Passive interfaces not supported
- No static neighbor definitions


```
ipv6 unicast-routing
ipv6 cef

int Fa0/0
 ipv6 address 2001:DB8:1::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 default-information only

int S0/0
 ipv6 address 2001:DB8:2::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 metric-offset 3

ipv6 router rip 1
 poison-reverse
```

# EIGRP

## Maximum hops

```
metric maximum-hops
```

Default of 100, can be upped to 255

## Distance

```
distance eigrp <internal> <external>
```
Default of 90 and 170

## EIGRP Wide Metrics

**show eigrp plugins** - Requires 8.0.0 eigrp-release
**show ip protocols** - K6, rib-scale of 128 and 64-bit wide metric if supported


## RIB-scale

The **metric rib-scale** command changes factor for downscaling, default 128, can be 1255. EIGRP still chooses best path, only downscaled when placed into rib

## Influencing path selection

Use delay, as bandwidth only other manually influenced component, and it has a knock on effect for other protocols. EIGRP also throttles based upon bandwidth, set it too high and interface could be swamped

## EIGRP packets

**show ip eigrp traffic** shows amount of packets received and what types

## Timers

**ip hello-interval eigrp 1 <time> ** - sets hello per interface
**ip hold-time eigrp 1 <time> **- sets hold per interface

## Neighbour verification


**show ip eigrp neighbours **

- H (Handle) - Shows internal number EIGRP assigns to each neighbour, internally identifies neighbours independent of addressing
- Address and Interface columns - Neighbors IP and routers interface towards neighbour
- Hold - Derived from value advertised by neighbour, decremented each second
- Uptime - Shows neighbour uptime
- SRTT - estimates turnover time between sending reliable packet to neighbour and receiving ack, show in ms
- RTO - Time router waits for ack of a retransmitted unicast packet after previously delivery not acknowledged, show in ms
- Q Cnt - Number of enqueued reliable packets prepared for sending and possibly not sent but for which no ack received, must be zero in stable network. Nonzero normal during router database sync or during network convergence
- Seq number - sequence number of last reliable packet received from neighbour

## All Links

**show ip eigrp topology all-links** - shows topology with all networks, including those who fail feasibility condition check

## Active timer

An active timer exists for a route. Default is 3 minutes, can be set between 1 and 65535 minutes (set with **timers active-time** under router eigrp). I

## Named Mode


- Address Family section - Address family command, specifies AF for which EIGRP instance shall be started. ASN part of this
- Per-AF interface section - Optional, af-interface, locate inside AF. One per-af-interface section created per each routed interface or subinterface. Can also use af-interface default for base settings.
- Per-af-topology section - Relates to MTR (Multi Topology Routing). Always present even if IOS has n support for MTR

```

router eigrp DAVE
 address-family ipv4 unicast autonomous-system 1
  af-interface default
   hello-interval 1
   hold-time 3
  exit-af-interface
  af-interface Loopback0
   passive-interface
  toplogy base
   maximum-paths 6
   variance 4
  exit-af-topology
  network 10.0.0.1 0.0.0.0
  network 10.255.255.1 0.0.0.0
 exit-address-family
 address-family ipv6 unicast autonomous-system 1
  af-interface default
   shutdown
  exit-af-interface
  af-interface Lo0
   no shutdown
  exit-af-interface
  af-interface Fa0/0
   no shutdown
  exit-af-interface
  topology base
   timers active-time 1
  exit-af-topology
 exit-address-family
```
### Address family config

- af-interface - Enter AF family config
- default - Set a command to its defaults
- eigrp - EIGRP Address Family specific commands
- exit-address-family - Exit AF config mode
- help - Description of interactive help system
- maximum-prefix - Limits prefixes allowed in aggregate
- metric - Modifies metrics and parameters for advertisement
- neighbour - Static neighbour config
- network - Enable routing on an network
- shutdown - shutdown AF
- timers - Adjust peering based timers
- topology - Topology config mode

### Per-Af-Interface


- add-paths - Advertise add paths
- authentication - Configure auth
- bandwidth-percent - Set percentage of bandwidth limit
- bfd - enable BFD
- dampening-change - Percent interface metric must change to cause update
- dampening-interval - Time in seconds to check interface metrics
- default - set a command to defaults
- exit-af-interface
- hello-interval
- hold-time
- next-hop-self
- passive-interface
- shutdown - Disables AF on interface
- split-horizon
- summary-address

### Per-AF-Topology


- auto-summary
- default
- default-information - Controls distribution of default info
- default-metric - Set metric of redistributed routes
- distance - Defines AD
- distribute-list - Filters entries in updates
- eigrp -
- exit-af-topology
- maximum-paths
- metric - modifies metric and parameters for advertisement
- offset-list - Add or subtract from EIGRP metrics
- redistribute
- snmp
- summary-metric - Metric for summary
- timers
- traffic-share - How to compute traffic share over alternate paths
- variance - COntrol load balancing variance


### Verification

**show eigrp address-family ipv4/ipv6 **used rather than show ip eigrp or sjow ipv6 eigrp (both still work, but not the new way, some features wont be shown)

## Router ID

```
eigrp router-id
```

Verify with show eigrp protocols and show ip protocols

## EIGRP Stub


- eigrp stub connected - Advertise connected routers
- eigrp stub leak-map - Allow dynamic prefixes based on leak map
- eigrp stub receive-only - Receive only neighbour
- eigrp stub redistributed - Allow redistributed routes
- eigrp stub static - Allow static routes
- eigrp stub summary - Allow summary routes

Use **show ip protocols** to show if a router is a stub, and **show ip eigrp neighbors detail** to see if neighbours are stub

By default, connected and summary assumed

## Route summarization

Classic mode
**ip summary-address eigrp ***asn address netmask [ distance ] *[ **leak-map ***name *]

Named mode

Under af-interface section

**summary-address ***address netmask *[ **leak-map *** name *]

topology base section
**summary-metric ***address netmask ***distance ***admin-distance.* - Useful for if summary would take over from other routes (as it has AD of 5)

## Passive interface

In classic mode, set either **passive-interface** and the interface, or **passive-interface default **to hit all interfaces. For named mode, **passive-interface** under af-interface section, or **passive-interface **under af-interface default section.

## Graceful Shutdown

Use shutdown command in following: -


- router eigrp mode (all AF instances deactivated)
- Under a particular AF, causing that family to be activated
- Under af-interface, ceasing operations for that AF on that interface

## authentication

CLassic mode commands: -

**ip authentication mode eigrp**
**ip authentication key-chain eigrp**

Cannot be done for all interfaces

Named mode: -

Under af-interface
**authentication mode**
**authentication key-chain**

Can also be done under af-interface default

```

key chain EIGRPKeys
 key 1
  key-string DAVE

router eigrp CCIE
 address-family ipv4 autonomous-system 1
  af-interface default
   authentication mode md5
   authentication key-chain EIGRPKeys
  af-interface Fa0/0
   authentication-mode hmac-sha-256 DAVESHA # DAVESHA is not the key used, it is a password set
   authentication key-chain EIGRPKeys
  af-interface F0/1
   authentication mode hmac-sha-256 DAVEISPW
   no authentication key-chain # Above password is now used as the key
  af-interface Se1/0
   no authentication mode
```

Use **show eigrp address-family ipv4 int detail ***interface*** **to see what key chain is used

Each key chain can have the **send-lifetime** set, and to authentication received packets in an **accept-lifetime**.


## Default Routing Using EIGRP

No dedicated command, either requires redistribution or summarization.

EIGRP used to support **ip default-network** command to flag a specific advertised route as candidate default route. Network had to be a classful network and advertised in EIGRP, and with candidate default flag set. Recent versions no longer honour the candidate default flag

If a static route configured with only egress interface, IOS treats route as directly connected network. Therefore if network 0.0.0.0 was used, this default would be pulled in. However this has no effect on anything with a next-hop set instead. Plus, all IPv4 enabled interfaces then become part of EIGRP

## Split Horizon

**no { ip | ipv6 } split-horizon eigrp** - Classic mode

**no split-horizon **- af-interface mode

## EIGRP over the ToP

```

R1

int LISP0
 bandwidth 1000000

int GI0/0
 ip address 192.0.2.31 255.255.255.0

ip route 0.0.0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
 topology base
 exit-af-topology
 neighbor 198.51.100.62 Gi0/0 remote 100 lisp-encap
  network 10.0.1.0 0.0.0.255
  network 192.0.2.31 0.0.0.0

R2

int LISP0
 bandwidth 1000000

int Gi0/0
 ip address 198.51.100.62 255.255.255.0

ip route 0.0.0.0 0.0.0.0 198.51.100.1

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
 topology base
 exit-af-topology
 neighbor 192.0.2.31 Gi0/0 remote 100 lisp-encap
  network 10.0.2.0 0.0.0.255
  network 198.51.100.62 0.0.0.0
```

Use show ip route to see outgoing interface as LISP0

show ip addr ipv4 nei - Shows neighbour on other side

show ip cef X.X.X.X/X internal - Shows LISP encapsulation in effect

OTP neighbours can be built into a route reflector, to stop the huge mesh of sessions. COnfig is as such: -

```

int LISP0
 bandwidth 1000000

int GI0/0
 ip address 192.0.2.31 255.255.255.0

ip route 0.0.0.0 0.0.0.0 192.0.2.2

router eigrp CCIE
 address-family ipv4 unicast autonomous-system 64512
  af-interface Gi0/0
   no next-hop-self # This is so this doesn't become the transit point
   no split-horizon # This is so routes can be advertised back on same as received interface (necessary for an RR)
  exit-af-interface
  topology base
  exit-af-topology
  remote-neighbors source GI0/0 unicast-listen lisp-encap
  network 10.0.1.1 0.0.0.0
  network 192.0.2.31 0.0.0.0
```

Remote-neighbours allow named-ACL to limit RRs

## EIGRP logging and reporting


- eigrp event-log-size - Set maximum event log entries
- eigrp event-logging - Log routing events
- eigrp log-neighbor-changes - Logs neighbor changes
- eigrp log-neighbor-warnings - Logs warnings of other neighbours

Viewed in **show eigrp address-family {ipv4 | ipv6} events**


The eigrp log-neighbor-warnings [seconds] is on by default, logging neighbour warning messages at 10-second intervals

## Route Filtering

Distribute lists can use ACL, prefix lists and route maps

## Offset lists

Can adjust metric, can be in, out and per interfaces

## Clear routing table

The **clear eigrp address-family { ipv4 | ipv6 } neighbors** command can clear all neighbourships and have router re-establish them. Using **soft** does a graceful restart, making topology tables resync but adjacencies stay up

# OSPF

## Router ID

```
router ospf 1
 router-id X.X.X.X
```

## Static neighbour config

Use **neighbor** command under ospf process. Can be set just one side, better to have on both.

## LSA Type 3 and Inter-Area costs

The **show ip ospf database summary ***link-id*** **shows cost, and **show ip ospf border-routers** shows cost to ABR.


## Stubby auth-pass-phrase-hashed

NSSA - **area ***area-id ***nssa**
Totally NSSA - **area ***area-id ***nssa no-summary**
Stubby - **area ***area-id ***stub**
Totally Stubby - **area ***area-id ***stub no-summary**

NSSA does not have a default route advertised automatically from ABRs. To do this, ABR must have **area ***area-id ***nssa default-information-originate. **NSSA-TS does not require this (automatic default exists)

## OSPF Config

```

R1

int Fa0/0
 ip address 10.1.1.1 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4

router ospf 1
 area 3 nssa no-summary
 area 4 stub no-summary
 area 5 stub
 network 10.1.0.0 0.0.255.255 area 0
 network 10.3.0.0 0.0.255.255 area 3
 network 10.4.0.0 0.0.255.255 area 4
 network 10.5.0.0 0.0.255.255 area 5

R2

int Fa0/0
 ip address 10.1.1.2 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf 2 area 0

router ospf 2
 area 5 stub

R3

router ospf 1
 area 3 nssa no-summary
 network 10.0.0.0 0.255.255.255 area 3

R4

router ospf 1
 area 4 stub no-summary
 network 10.0.0.0 0.255.255.255 area 4

S1

int vlan 1
 ip address 10.1.1.3 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4

router ospf 1
 router-id 7.7.7.7
 network 10.1.0.0 0.0.255.255 area 0

S2

int vlan 1
 ip address 10.1.1.4 255.255.255.0
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf priority 254
```

## Clearing Process

** clear ip ospf process ** - Clears all processess

**log-adjacency-changes detail** - Shows message at each state Change

## Interface costs

```
ip ospf cost

router ospf  1
 auto-cost-reference-bandwidth *mbps*
```

Set per neighbour with **neighbor** neighbor **cost value**


## Alternative to OSPF network command

** ip ospf process-id area area-id **

All secondaries matched too unless using ** secondaries none ** at end

## OSPF Filtering

### Distribute List


- Distribute list in inbound direction applies to results of SPF, not prior to it
- Distribute list in outbound applies only to redistributed routes and only on ASBR, selects which redistributed routes shall be advertised
- Inbound logic does not filter inbound LSAs, filters routes that SPF chooses to add to routing table
- If distribute list includes incoming interface, interface checked as if it were outgoing interface of the route. This means that the routes may have been flooded from multiple interfaces, so router checks outoing interface of route as if it had learned about routes through updates coming in that interface


```
router ospf 1
 distribute-list prefix prefix-list-1 in Serial 0.2

router ospf 1
 distribute-list route-map rm-1 in

route-map rm-1 deny 10
 match ip address 48
 match ip route-source 51 # Use an ACL to specify source of routes, eg permit 2.2.2.2
 ```

### ABR LSA Type 3 Filtering


**area ***number ***filter-list prefix ***name ***in | out **


- When direction in, prefixes filtered going into configured aresa
- When direction out, prefixes filtered coming out of configured area

### FIltering Type 3 with area range

Area range performs route summarization at ABRs, telling route to cease advertising smaller subnets in a particular address range, sending a single type 3 LSA with a summary.

When using the **not-advertise **keyword, summary route not advertised either.

## Virtual link

```

R1

router ospf 1
 area 3 virtual-link 3.3.3.3

R3

router ospf 1
 area 3 virtual-link 1.1.1.1
```

## Classic authentication

None


int Fa0/0
 ip ospf authentication null


Clear text


int Fa0/0
 ip ospf authentication
 ip ospf authentication-key key-value

MD5

int Fa0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key key-number md5 key-value


### Virtual link auth

**area ***area-id ***virtual-link ***router-id ***authentication null **means type 0.
**area ***area-id ***virtual-link ***router-id ***authentication** means type 1.
**area ***area-id ***virtual-link ***router-id ***authentication message-digest **means type 2

## Extended Crypto OSPF Auth

```

key chain ospf
 key 1
  cryptographic-algorithm hmac-sha-1/256/384/512/md5
  key-string DAVE

int Gi0/0
 ip ospf authentication key-chain ospf
```

## OSPF TTL Security

Enable per interface with **ip ospf ttl-security**, or per process using **ttl-security all-interfaces**, exempt from an interface with **ip ospf ttl-security disable.**

To enable on sham or virtual links, do **area virtual-link ttl-security hops** or **area sham-link ttl-security hops. **Hops is mandatory in this (should be based on longest possible intera area path).

## Tuning OSPF performance

### SPF Throttling (for SPF scheduling)

Configured with **timers throttle spf ***spf-start spf-hold spf-max-wait *under **router ospf**. All arguments in milliseconds. Current values shown in **show ip ospf**. Also, **debug ip ospf spf statistic** can verify current and next wait intervals.

### LSA Throttling (for LSA origination)

Configured with **timers throttle lsa all ***start-interval hold-interval max-interval. *All in milliseconds. Can be seen in show ip ospf**. **Router can be configured to ignore an LSA upon arrival if it arrives too often. Use **timers lsa arrival ***milliseconds. *Same LSA is accepted only if it arrives more than milliseconds after previous accepted one. Default 1000 ms, seen in show ip ospf. Should be smaller than neighbours initial hold in LSA throttling, otherwise neighbour allowed to send sooner than would be accepted.

## Incremental ISPF

Configure by applying **ispf** under router ospf context. Can be enabled per router, not needed through entire network to work.

## OSPFv2 Prefix Suppression

Enabled in v2 with **prefix-suppression** command (works on all OSPF interfaces except loopbacks). Cnfigure per interface with ** ip ospf prefix-suppression, **add the **disable **keyword to disable it per interface.

## OSPF stub router config

**max-metric router-lsa on-startup ***announce-time- *Done under router ospf, in seconds

**max-metric router-lsa on-startup wait-for-bgp ** - Waits until BGP signals convergence or until 10 minutes pass

## OSPF Graceful Restart

CEF handles forwarding during graceful restart, OSPF rebuilds RIB tables, provided conditions met. Cisco and IETF NSF awareness enabled by default in IOS. Disable with **nsf [ cisco | ietf ] helper disable**

## OSPF Graceful Shutdown

Use **shutdown** under process

## OSPFv3

### Over FR

```
int Se0/0
 frame-relay map ipv6 FE80::207:85FF:Fe80:7208 708 broadcast
 frame-relay map ipv6 2001::207:85FF:FE80:7208 708
```

### Config

```

ipv6 unicast-routing
ipv6 cef

int Lo0
 ipv6 address 3001:0:3::/64 eui-64
 ipv6 ospf 1 area 704

int Lo1
 ip address 10.3.3.6 255.255.255.0

int lo2
 ipv6 address 3001:0:3:2::/64 eui-64
 ipv6 ospf network point-to-point
 ipv6 ospf 1 area 0

int Fa0/0
 ipv6 address 2001:0:3::/64 eui-64
 ipv6 ospf 1 area 704

int Se0/0
 bandwidth 128
 encapsulation frame-relay
 ipv6 address 2001::/64 eui-64
 ipv6 ospf neighbor FE80::207:85FF:Fe80:71B8
 frame-relay map ipv6 FE80::207:85FF:FE80:71B8 807 broadcast
 frame-relay map ipv6 2001::207:85FF:FE80:71B8 807

ipv6 router ospf 1
```

Can verify config with show ipv6 interface brief, show ipv6 protocols (under ospf it will show interfaces and area), show ipv6 ospf interface, show ipv6 router ospf

### v3 Auth and Encryption

```

int Fa0/0
 ipv6 ospf auth ipsec spi 1000 sha1 <KEY>

int Se1/0
 ipv6 ospf encryption ipsec spi 1001 esp aes-cbc 128 <KEY>

ipv6 router ospf 1
 area 1 authentication ipsec spi 1002 md5 <KEY>
 area 2 encryption ipsec spi 1003 esp 3des <KEY> md5 <MD5-KEY>
 ```

### Address Family config

```

int lo0
 ipv6 address 2001:DB8:0:FFFF::1/128
 ip address 10.255.255.1 255.255.255.255
 ospfv3 1 ipv6 area 0
 ospfv3 1 ipv4 area 0

int Fa0/0
 ipv6 address 2001:DB8:1:1::1/64
 ip address 10.1.1.1 255.255.255.0
 ospfv3 network point-to-point
 ospfv3 1 ipv6 area 1
 ospfv3 1 ipv4 area 1

int Se0/0/0
 ipv6 address 2001:DB8:0:1::1/64
 ip address 10.0.1.1 255.255.255.0
 ospfv3 hello-interval 1
 ospfv3 1 ipv6 area 0
 ospfv3 1 ipv4 area 0

router ospfv3 1
 address-family ipv4
  area 1 range 10.1.0.0 255.255.0.0
 address-family ipv6
  area 1 range 2001:DB8:1::/48
```

### Prefix Suppression

Configured per process with **prefix-suppression**, or per interface with **ipv6 ospf prefix-suppression **or **ospfv3 prefix-suppression.** If configured outside of AF, affects all address families, or can be done per address family.

# IS-IS

## Metric

Default metric of 10 on all interfaces in IOS, regardless of Bandwidth. No automatic calculation. Can be defined on interface with **isis metric ***metric *[ *level *].

## Hellos

10 second hello time by default, can be set between 1 to 65535 per interface with **isis hello-interval ***seconds *[ *level *]. Hold time done as multiplier of hello. Default is 3. Can be changed with **isis hello-multiplier ***multiplier *[ *level *]. Timers do not need to match between neighbors.

## Three way handshake

** isis three-way-handshake cisco ** - per interface
** isis three-way-handshake ietf ** - per interface

## CSNPs

**isis csnp-interval ***interval *[ *level *]

## Interface priority

Interface priority in range of 0 to 127, configured with **isis priority ***priority *[ *level *]. Entire range usable. 0 excludes router from being a DIS

## Summarization

Multiple ares in a domain,primary created for summariztion. Summarization should be configured on each L1L2 router in the area. Done by adding **summary-address** command inside **router isis**. Applies equally to intra-area networks going from L1 to L2, and redistributed routes.

## IS-IS authentication

- LAN IIH - Level 1 - **isis auth mode { text | md5 } level 1**, **isis auth key-chain ***name ***level-1 - **Interface commands
- LAN IIH - Level 2 - **isis auth mode { text | md5 } level 2**, **isis auth key-chain ***name ***level-2 - **Interface commands
- P2P IIH - **isis auth mode { text | md5 }, isis auth key-chain ***name*
- LSP, CSNP, PSNP - Level 1 - **auth mode {text | md5} level-1 , auth key-chain ***name ***level-1** - IS-IS process
- LSP, CSNP, PSNP - Level 2 - **auth mode {text | md5} level-2, auth key-chain ***name ***level-2** - IS-IS process

## Config

R1 Config

```
key chain ISISAuth
 key 1
  key-string DaveLikesToRoute

int Lo0
 ip address 10.1.1.1 255.255.255.0
 ip router isis

int Se0/0/0
 desc TO R2
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


R2 Config

```
int Lo0
 ip address 10.1.2.1 255.255.255.0

int Se0/0/0
 desc To R3
 ip address 10.12.23.2 255.255.255.0
 ip router isis
 isis circuit-type level-2-only
 isis metric 100 level-2

int Se0/0/1
 desc To R1
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
 passive-interface Lo0 # Note that this will advertise out Lo0's IP, despite no ip router isis
```

Routers providing connectivity for areas must be L1L2 routers, otherwise L1 routers cant form adj with them.

Passive interface behaviour is so that a network can be advertised without necessarily needing to be active in IS-IS (i.e. not forming adj). Similar to EIGRP for v6 where passive-interface default causes all local interfaces networks to be advertised in is-is

R3 with v6 config

```
ipv6 unicast-routing

int lo0
 ip addr 10.2.3.1 255.255.255.0
 ip router isis
 ipv6 address 2001:DB8:2:3::1/64
 ipv6 router isis

int Se0/0/0
 desc to R4
 ip addr 10.2.34.4 255.255.255.0
 ip router isis
 ipv6 address Fe80::3 link-local
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

Commands to check are


- show clns - Shows info about routers NET and mode of Integrated IS-IS
- show clns is-neighbors - Displays neighbours info about them, use the detail word for more detailed info
- show clns neighbors - Can display SNPA of neighbour (for HDLC and PPP, text description shown, also can specify detail)
- show clns interface - Shows info about inferace
- show isis neighbors - Supports detail keyword
- show isis database detail
- show ip route isis


# IGP Route Redistribution, Route Summarization, Default Routing and Troubleshooting

## Route-Map match commands


- match interface - looks at outgoing interface of routes
- match ip address - Examines route prefix and prefix length (can use ACL or prefix list)
- match ip next hop - Examines routes next hop, use ACL
- match ip route-source - Match advertising routers IP, use acl
- match metric - Matches metric exactly, or optionally range of metrics (plus/minus configured deviation)
- match route-type - Matches route type (internval, external, E1/N1, E1/N2, level-1, level-2)
- match tag

## Set commands

- set level - Defined database into which route redist (l1, l2, l1l2, stub-area, backbone)
- set metric *metric-value* - Sets route metric OSPF, RIP and IS-IS
- set metric *bandwidth delay reliability loading mtu *- Sets IGRP/EIGRP metric
- set metric type - internal, external, type-1, type-2, for IS-IS and OSPF
- set tag

## Administrative distance

**distance ***distance *- RIP
**distance eigrp ***internal-dist external-dist*
**distance ospf {[intra-area ***dist1 ***] [ inter-area ***dist2***] [ external ***dist3 ***] }**

## Full Syntax redistribution

**redistribute ***protocol ***[ ***process-id ***] [ level-1 | level-1-2 | level-2 ] [ ***as-number ***] [ metric ***metric-value ***] [ metric-type ***type-value ***] [ match { internal | external 1 | external 2} ] [ tag ***tag-value ***] [ route-map ***map-tag ***] [ subnets ]**

### Notes

* Subnets causes subnets to be advertised into OSPF
* Default cost of 20 for OSPF from IGP, 1 from BGP
* Only redists routes in current IP routing table

By default, when redistributing into OSPF, only redistributes classful networks, hence **subnets** option. If **auto-summary** used, each redistributed network would show just the classiful networks.

## Distance per route

Can apply Distance to just a route, eg

**distance { ***distance-value ip-address ***{ ***wildcard-mask ***} [ ***ip-standard-list ***] [ ***ip-extended-list ***]**


## Route Tags

**distribute-list route-map check-tag-9999 in**
**redistribute ospf 1 route-map tag-ospf-9999 in**

## Route Summarization

### EIGRP

Place **ip summary-address eigrp ***as-number network-address subnet-mask *[ *admin-distance *] on an interface. Any component routes causes summary route to be sent out that interface.

### OSPF

- ASBR - **summary-address **{{* ip-address mask *} { *prefix-mask *}} **[not-advertise] [tag ***tag ***]**
- ABR - **area ***area-id ***range ***ip-address mask ***[advertise | not-advertise] [cost ***cost ***]**

For ABR command, this is the area for where component subnets reside. Can set cost of summary route rather than using lowest cost of all component routes.


## Default routes


- Static route to 0.0.0.0 with redistribute static command - EIGRP, RIP
- **default-information-originate **command - RIP, OSPF
- **ip default-network**- RIP, EIGRP
- Using summary routes - EIGRP


### Static routes with redistribute


- both commands need to be on same router
- Metric must be default or set
- Redistribute command can refer to route map, which examines all static routes
- EIGRP treats default route as external by default

### Default-information originate


- Redistributes any default route in table
- Can set metric and metric type directly, default cost of 1, type E2
- Allows use of always keyword, meaning default always exist even if not in table
- Supported in RIP but with differences

### IP Default network


- Local router must configure **ip default-network ***net-number*, with net-number being classful network number
- Classiful network must be in local routers IP routing table (by any means)
- For EIGRP, classiful network must be advertised by local router into EIGRP

### Route Summarization for defaults

- Local router creats local summary, dest null 0, using AD 5, when deciding whether its route is best one to add to local routing table
- Advertises summary to other ADs as 90
- Need to set higher distance in the **ip summary-address **command to not blackhole traffic


## PfR


## PfR Basic Configuration



### **Config of MC**



1. Create the authentication key chain


Auth required, uses MD5 keychain/keystring approach. Made under global config

R4 - MC

```
key chain PFR_AUTH
 key 1
  key-string DAVEPERFORMS
```

2. Enable PFR process
```
pfr master
```

3. Designate internal/external interfaces


MC must designate what interfaces on the BR are internal and external

```
pfr master
 border 2.2.2.2 key-chain PFR_AUTH
  interface Se0/0.21 internal
  interface Fa0/0 external
 border 3.3.3.3 key-chain PFR_AUTH
  interface Se0/0.31 internval
  interface Fa0/0 external
```

show oer master border


### **Config of BR**

1. Authentication key chain

```
key chain PFR_AUTH
 key 1
  key-string DAVEPERFORMS
```
2. Enable PfR process

```
pfr border
 master 4.4.4.4 key-chain PFR_AUTH
```

3. Specify the local itnerface

Need to specify a source, eg a loopback

```
pfr border
 local loopback 0
```

Can also use logging and change the port used as well (**logging **command under pfr border and **port 3950** under pfr border)


Will be seen as MC Active on MC when both BRs up (otherwise PfR is useless)


## Layer 3 Protocol Troubleshooting and Commands

- show ip protocols - lots of info about routing protocols
- show interfaces
- show ip interfaces - will show features like NAT< policy routing etc
- show ip nat trans
- show ip access-list
- show ip int brief
- show dampening
- show logging
- show policy-map
- traceroute
- ping (and extended ping)
- show route-map
- show standby
- show vrrp
- show track
- show ip route

# Fundamentals of BGP operations

## Timers

```
router bgp 65001
 bgp timers keepalive holdtime
 neighbor x.x.x.x timers keepalive holdtime [min-holdtime]
```

## Router ID

```
bgp router-id
```

## Basic Config

```
router bgp 123
 no sync
 bgp router-id 111.111.111.111
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 123
 neighbor 2.2.2.2 update-source Loopback1
 neighbor 3.3.3.3 remote-as 123
 neighbor 3.3.3.3 update-source Loopback1
 neighbor 3.3.3.3 password DAVE-LIKES-BGP
 no auto-summary
 ```

## External neighbours

Single links usually, **ebgp-multihop** if not

## Resetting Prefers

**neighbor shutdown** - Shuts down a connection from config

**clear ip bgp * - **Resets neighbour connection, closes TCP, removes all entries in table for that neighbour, begins process of rediscovering neighbour after

## Network command

no auto-summary implied since 12.3 mainline

**network **{ *network-number *[ **mask ***network-mask *]} [ **route-map ***map-tag *]

## aggregate

**aggregate-address ***prefix mask - *Will advertise out aggregate

**aggregate-address ***prefix mask ***summary-only ***- *Will advertise out aggregate only

**aggregate-address ***prefix mask ***summary-only as-set***- *Will advertise out aggregate only with all ASNs in component subnets as an AS-SET

## Default routes

- Use **network **command
- Redistribute in
- Use **neighbor ***neighbor-id ***default-originate [ route-map ***route-map-name *] command

default-information-originate required to get default route in when redistributing

## Next hop treatment

- iBGP - **next-hop-self** to change it
- eBGP - **next-hop-unchanged** to not change it

## Showing per neighbour routes

**show ip bgp neighbour advertised-routes**
**show ip bgp neighbour received-routes ** - Only works if **neighbor ***neighbor*** soft-reconfiguration inbound **enabled

The * valid part in **show ip bgp** just means the route is a candidate for use. Before route can be used and added, NEXT_HOP must also be reachable

## Admin distance

Change under BGP for all routes with **distance bgp ***extenal internal local***. **Or change for a route with **distance ***distance {* *ip-address *{ *wildcard-mask *}} [ *ip-standard-list *| *ip-extended-list *]

## Backdoor route

There is the **network backdoor** command, following occurs: -

- Makes BGP route a local route, hence 200 AD by default
- Does not advertise route with BGP downstream (received it via eBGP)

## Confederations

As ASN stated in **router bgp **will now be confederation AS, cant configure on existing kit without taking down BGP on this router

**router bgp ***sub-as*
**bgp confederation identifier ***asn - *Defines true AS
**bgp confederation peers ***sub-asn - *Identifies a neighbouring AS as another sub-AS

## Route Reflectors

**neighbor ***neighbor ***route-reflector-client**
**bgp cluster-id ***id*

## MP-BGP


## Config of MP BGP

When MP-BGP activated, automatically carries IPv4 unicast routes. This can be disabled with

```
router bgp 1
 no bgp default ipv4-unicast
```

Some configs can carry VPN-IPv4 routes, some only v4, other carry both. Type of BGP session controlled with address families, and activating the peer in that AF. Known as context based routing.

Default context becomes catch all where any non-VRF based or IPv4-specific session can be configured. Anything in here injected into global table

Standard v4 config

```
router bgp 1
 neighbor 194.22.15.3 remote-as 1
 neighbor 194.22.15.3 update-source lo0
 neighbor 194.22.15.3 activate
```

AF config

```
router bgp 1
 address-family vpnv4
  neighbor 194.22.15.3 activate
```

Another command required to support MP-BGP-specific extended communities

**neighbor ***neighbor ***send-community extended/standard/both. **Default sends only extended.

# BGP Routing Policies

## Filtering types


- **neighbor distribute-list** - Using standard ACL, can match prefix with wildcard mask
- **neighbor distribute-list **- Using extended ACL, can match prefix and length, with WC mask for each
- **neighbor prefix-list ** - Exact or first N bits pf prefix, plus range of prefix lengths
- **neighbor filter-list** - AS_PATH contents
- **neighbor route-map** - Prefix, prefix length, AS_PATH, and/or any PA matchable

## Filtering based on NLRI

### Route Map Rules for NLRI Filtering

**deny** as a route-map action will filter a route, whereas in a prefix-list or ACL is specifies whether it matches or doesnt match

### Soft reconfig

**clear ip bgp { * | neighbor-address | peer-group-name} [ soft [in | out ]]**

IOS supports soft reconfig for send update automatically, needs enabling for inbound. **neighbor ***neighbor-id ***soft-reconfiguration inbound**. This means updates received will be stored.

## Filtering based on aggregate-address command

Can allow none, all, or subset of summaries routes. This means filtering certain routes is an option. Filter all with **summary-only**, allow all with no **summary-only**, or use a supress map to allow certain ones through.

## AS PATH filtering

1. Configure AS_PATH filter using **ip as-path access-list ***number ***permit/deny ***regex*
2. Enable AS_PATH filter with **neighbor ***neighbor-id ***filter-list ***as-path-filter-number ***{ in | out }**

### Types to match

* AS_SEQ standard
* AS_SET has comma delimiter between ASNs, enclosing segment with {}
* AS_CONFED_SEQ has space delimiter between ASNs, enclosing segment with ()
* AS_CONFED_SET has comma delimiter between ASNs, enclosing segment with {}

### regex


1. Regex of first line in list applied to AS_PATH of each route
2. For matched NLRIs, NLRI passed/filtered based on that AS_ATH filters configured **permit **or **deny**
3. For unmatched, Step 1 and 2 repeated using next line in filter
4. Any NLRIs not matched explicitly is filtered

- ^ - Start of line
- $ - End of line
- | - Logical OR
- _ - Any delimiter (blank, comma, start of line, end of line)
- . - Any single character
- ? - Zero or one instance of character
- * - Zero or more instances of character
- + - One or more instance of character
- (string) - Combine enclose string characters as a single entity when used with other characters (eg (49182)+)
- [string] - Wildcard for which any single character in string can be used to match that position in AS_PATH

To match an AS_CONFED, need to enclose brackets like so, [(], as ( is a regex character already

## BGP Decision process

Shortest AS_PATH length can be ignored with **bgp bestpath as-path ignore**

First 9 steps done before ** maximum-paths** comes into play

Choose smallest neighbor RID, use route who next-hop router RID is smallest, only performed if **bgp bestpath compare-routerid **configured

## Configuring BGP Policies

### NEXT_HOP reachable

Can be changed with **next-hop-self **or **next-hop-unchanged**.

### Weight

* 0 through 65535
* Default 0 for learned, 32768 for locally injected
* Apply with route map or **neighbor weight** command, route map takes preference

### Highest Local Pref

* Default 100, change with bgp default local-preference
* Set with route-map

### Choose between locally injected routes based on ORIGIN PA

As BGP assigns a weight of 32768 to locally injected routes, automatically uses them.

To see routes where this might happen, a route would have to be injected AND advertised to neighbour, with a route-map assigning the weight. Another option is router injects routes through multiple methods, and same NLRI injected through two different sources. This would be the case with a **network** command and **redistribute connected **command. Same weights, same local pref.

### Shortest AS path

* AS_SET seen as one ASN
* **bgp bestpath as-path ignore**
* Confeds do not count in calculation
* neighbor remove-private-as - Removes private AS used by neighbor AS
* neighbor local-as no-prepend - Allows different AS

### Best Origin PA

i over e over ?, e never occurs today

**set origin** in route map

### Smallest MED

* Default 0, sent to one AS, no further
* **bgp bestpath med missing-as-worst **sets it to maximum value
* **bgp always-compare-med** - When multiple routes to different NLRI list different neighbouring ASNs, all routers in ASN would require this
* **bgp deterministic-med** - Stops sequential evaluation of routes to find best, processes routes per adjacent AS, picking best from that AS then comparing best from all "best" found

### EBGP over iBGP

eBGP > iBGP

### Smallest IGP Metric to next hop

As title

### Maximum paths

Above are done before maximum paths taken into account

Must decide which route is best based upon tiebreakers, and if to add multiple routes (**maximum-paths**)

### Lowest BGP RID

**bgp bestpath compare-routerid**.

Has caveats, see Notes

### Lowest Neighbor ID

As title

### Maximum paths

See notes

**maximum eibgp** - Only applies when MPLS in use


## Communities

**neighbor send-community - **needed to allow an Update to include community PA

**ip community-list** - used to match communities, no more than 16 lines in a standard list, more with extended

## display

 **ip bgp-community new-format** - Shows in AA:NN format rather than decimal

## Remove communities

**set community none**

**set comm-list list delete**

## Filtering

**match community list**, can also use exact (When exact keyword is spcified, match happen only when BGP updates have completely same communities value specified in the community list.)

## Internal Neighbor Loss detection

**neighbor fall-over**

## EBGP Fast Session Deactivation

This is a per neighbour setting. Can disable fast external fall-over with **no bgp fast-external fallover**

# Classification and Marking

## TOS values


- Routine - IPP 0 - 000
- Priority - IPP 1 - 001
- Immediate - IPP 2 - 010
- FLash - IPP 3 - 011
- Flash Override - IPP 4 - 100
- Critic/Critical - IPP 5 - 101
- Internetwork Control - IPP 6 - 110
- Network Control - IPP 7 - 111

## Class Selector PHB/dscp


- Default/CS0 - 000000 - 000 - Routine
- CS1 001000 - 001 - Priority
- CS2 010000 - 010 - Immediate
- CS3 011000 - 011 - Flash
- CS4 100000 - 100 - Flash Override
- CS5 101000 - 101 - Critical
- CS6 110000 - 110 - Internetwork Control
- CS7 111000 - 111 - Network Control

## AF PHB/DSCP


- 1 - AF11/10/001010, AF12/12/001100, AF13/14/001110
- 2 - AF21/18/010010, AF22/20/010100, AF23/22/010110
- 3 - AF31/26/011010, AF32/28/011100, AF33/30/011110
- 4 - AF41/34/100010, AF42/36/100100, AF43/38/100110

Formula to get to decimal from name is 8x + 2y, eg AF41 = 8*4 + 2*1 = 34

## EF

Decimal 46, binary 101110

## Match commands

```

class-map match-all to-nest
 match access-group 102
 match precedence 5

class-map match-any nested
 match class to-nest
 match cos 5
 ```

- Up to four Cos/IPP or eight DSCP values can be listed on a single match cos, match precedence or match dscp command. If any values found in packet, statement matched
- If class map has multiple match commands, match-any or match-all paramaeter on class-map defines how to match (default is match-all)
- match class *name* matches another class map, considered to match if refernced class map also results in match

## NBAR

** match protocol**, requires CEF

## CB Marking

- CB Marking requires CEF
- Packets classified based on logic in MQC class maps
- MQC policy prefers to one or more class maps
- CB Marking enabled for packets either entering or exiting interface using **service-policy in | out ***policy-map-name*
- CB Marking policy map processed sequentially, once matches doesnt got further
- Multiple sets in one class allowed
- Packets not expliticitly matched considered to match class-default
- For no set command in policy map, packets in that class not marked


- set [ip] precedence - for v4 and v6 if IP omitted, v4 only if ip stated
- set ip dscp - As above but for DSCP
- set cos - Marks CoS value
- set qos-group - Marks group identifier for QoS group
- set atm-clp
- set fr-de

**show policy-map ***policymap-name - *shows config

**show policy-map ***interface-spec ***input/output class ***class-name *- shows stats about policy map on an interface

```

ip cef

class-map voip-rtp
 match protocol rtp audio

class-map http-impo
 match http url "*important*"

class-map http-not
 match protocol http url "*not-so*"

class-map match-any NetMeet
 match protocol rtp payload-type 4
 match protocol rtp payload-type 34

policy-map laundry-list
 class voip-rtp
  set ip dscp EF
 class NetMeet
  set ip dscp AF41
 class http-impo
  set ip dscp AF21
 class http-not
  set ip dscp AF23
 class class-default
  set ip DSCP default

int Fa0/0
 service-policy input laundry-list
```

**show policy-map interface ***interface-name ***[ vc **[vpi/] vci**] [ dlci ***dlci *] [ **input | output ] [ class ***class-name*]

## CB Marking of CoS and DSCP

```

class-map match-any EF
 match dscp EF

class-map AF11
 match dscp AF11

class-map COS1
 match cos 1

policy-map map-cos-to-dscp
 class cos1
  set DSCP af11
 class cos5
  set ip DSCP EF
```

## NBAR

Need to make sure that **ip nbar protocol discovery **is either on an interface, or enabled by default

**show ip nbar protocol-discover interface Fa0/0 stats packet-count top-n 5**

From 12.2T/12.3, ip nbar protocol-discovery command no longer needed

NBAR can be upgraded with PDLMs (Packet Description Language Modules). Can download, copy to flash, and add with **ip nbar pdlm ***pdlm-name. *NBAR can then match on that protocol

## Cisco recommended traffic classes

Type, Cos, IPP, DSCP
- Voice Payload, 5, 5, EF
- Video Payload, 4, 4, AF41
- Voice, Video signalling, 3, 3, CS3
- Mission Critical data, 3, 3, AF31, AF32, AF33
- Transactional data, 2, 2, AF21, AF22, AF23
- Bulk data, 1, 1, AF11, AF12, AF13
- Best Effort 0, 0 BE
- Scavenger, 0, 0, 2, 4 ,6

## QoS Pre-Classification

Enable in tunnel config mode, virtual template or crypto map with **qos pre-classify**. Can see effects with **show interface **and **show crypto-map**


- interface tunnel - GRE and IPIP
- interface virtual-template - L2F and L2TP
- crypto map - IPsec

## Policy Routing for Marking


1. Packets examined as they enter interface
2. Route map matches subset of packets
3. Mark either IPP or entire ToS using set command
4. Might also define route with set command too (not required)

## AutoQoS for VoIP

Enabled at interface level with **auto qos voip { cisco-phone | cisco-softphone}. **

Enable on uplink with **auto qos voip trust**.

Enable on a router with **auto qos voip [ trust ]**. Make sure interface bandwidth configured before, as QoS config wont change later. When issuing on an individual data circuit, config differs based on interface. Compression and fragmentation enabled on links of 768kbps and lower. Not enabled on higher. Also configures traffic shaping and applies service policy regardless of bandwidth

show auto qos - displays interface AutoQoS commands
show mls qos - Several modifiers to display queueing and CoS/DSCP mappings
show policy-map interface

## AutoQoS for Enterprise

**auto discovery qos [ trust ], **issued at interface, DLCI or PVC level. CEF needs to be enabled and bandwidth configured. Trust keyword for traffic arriving already marked


Use **auto qos **on an interface. IN case of a DLCI, router applies policy map to FR map class and applies class to DLCI. Can turn off NBAR traffic collection with **no auto disvovery qos**

show auto discover qos - lists types and amounts of traffic

show auto qos

show policy map interface

# Congestion Management and Avoidance

## Hardware Queues


show controllers *interface* - tx_limited shows Tx_ring length
```
int s0/0
  tx-ring-limit 1
```

## CBWFQ basic features and config

**bandwidth ***bandwith-kbps/***percent ***percent - *Sets literal or percentage bandwidth for class
**bandwidth **{remaining percent *percent *} - Sets percentage of remaining bandwidth for class
**queue-limit ***queue-limit *- sets maximum length of queue
**fair-queue **[ queue-limit *queue-value - *Enables WFQ in class (class-default only)

```

class-map match-all voip-rtp
 match ip rtp 16384 16383

policy-map queue-voip
 class voip-rtp
  bandwidth 64
 class class-default
  fair-queue

int Se0/0
 bandwidth 128
 service-policy output queue-voip
```

## LLQ

LLQ is enabled in CBWFQ configuration by doing the following: -

**priority **{ *bandwidth-kbps *| **percent ***percentage *} [ *burst *]

```

policy-map queue-on-dscp
 class dscp-ef
  priority 58
 class dscp-af41
  bandwidth 22
 class dscp-af21
  bandwidth 20

int Se0/0  
 max-reserved-bandwidth 85
 serive-policy output queue-on-dscp
```

## WRED

### WRED Weighting

A list of the defaults for DSCP values is below: -

DSCP - Minimum Threshold - Maximum Threshold - MPD - 1/MPD


- AFx1, 33, 40, 10, 10%
- AFx2, 28, 40, 10, 10%
- AFx3, 24, 40, 10, 10%
- EF, 37, 40, 10, 10%

### Config

Most queue mechanisms do not support WRED, so can be configured in following locations: -

- Physical interface (with FIFO queueing)
- For non LLQ class in CBWFQ
- On ATM VC

 **random-detect** command enables WRED, either under interfaces or nder map. Can use **dscp-based** keyword to act on DSCP calues.


To changed WRED config from default wred profile, add this in same location as other random-detect ocmmand: -

**random-detect precedence ***precedence min-threshold max-threshold *[ *mark-prob-denominator *]

**random-detect dscp ***dscpvalue min-threshold max-threshold *[ *mark-probability-denominator *]

**random-detect exponential-weighting-constant ***constant* - Lower means old average small part of calculation (quicker changing average)

## LAN Switch QoS

### Creating priority queue

```
mls qos srr-queue input cos-map queue 2 6
mls qos srr-queue input priority-queue 2 bandwidth 20
mls qos srr-queue input buffers percentage1 percentage2
```

By default 90 percent of buffers assigned to queue 1, 10 to queue 2. Set frequency at which scheduler takes from buffers using **mls qos srr-queue input bandwidth ***weight1 weight2. *Default is 4 and 4 (evenly between two). Values are relative weightings, not strict values.


### Congestion Avoidance

Command to configure tail drop percentages for each threshold is: -

**mls qos srr-queue input threshold ***queue-id threshold-percentage1 threshold-percentage2*

If trusting CoS, map the CoS to a threshold with: -

**mls qos srr-queue input cos-map threshold ***threshold-id cos 1  cos 8*

As above but DSCP:

**mls qos srr-queue input dscp-map threshold ***threshold-id dscp 1  dscp 8*

```

mls qos srr-queue input buffers 80 20
mls qos srr-queue input bandwidth 3 1
mls qos srr-queue input threshold 1 40 60
mls qos srr-queue input cos-map threshold 1 0 1 2 3
mls qos srr-queue input cos-map threshold 2 4 5
mls qos srr-queue input cos-map threshold 3 6 7
```

All commands global for ingress QoS, so apply to all interfaces


### Egress queueing

**srr-queue bandwidth share ***weight1 weight 2 weight3 weight4*
**srr-queue bandwidth shape ***weight1 weight 2 weight3 weight4*

With default weights of 25, if all four queues contained frames, switch service each queue equally.

When queues not full though, shared scheduling keeps servicing single queue with that queue getting all bandwidth. With shapred, switch waits to servie queue, not sending any date out that interface so that queue only receives its configured percentage

```

mls qos queue-set output 1 buffers 40 20 30 10
mls qos queue-set output 1 threshold 2 40 60 100 100
int Fa0/1
 queue-set 1
 srr-queue bandwidth share 10 10 1 1
 srr-queue bandwidth shape 10 0 20 20
 priority-queue out
```


## RSVP

### Configuring

Enabled on an interace using **ip rsvp bandwdith ***total-kbps single-flow-kbps. *If no total specified, defaults to 75 percent of int-bw. If no flow value, any flow can reserve all bandwidth

DSCP value for RSVP controll messages set with **ip rsvp signalling dscp ***dscp-value*

### RSVP for Voice

When using LLQ with CBWFQ, disable RSVPs WFQ with **ip rsvp resource-provider none**. By default RSVP attempts to process every packet (not just voice). Turn this off with **ip rsvp data-packet classification none.**. LLQ and CBWFQ then configured as normal. RSVP then reserves bandwidth for voice calls, gateways QoS processes place voice traffic into priority queue

```
int S0/1/0
 ip rsvp bandwidth 128 64
 ip rsvp signalling dscp 40
 ip rsvp resource-provider none
 ip rsvp data-packet classification none
 service-policy output llq
```

Verify with

show ip rsvp interface
show ip rsvp interface detail

# Shaping, Policing and Link fragmentation

## Generic Traffic Shaping

Older, supported on router interfaces (not with flow switching)

**traffic-shape rate ***shaped-rate *[*Bc*] [*Be*] [ *buffer-limit*] - Shaprd rate is bps, bc and be bits, buffer limit in bps. Quart of shapred rate by default for be and bc

To limit with an acl, **traffic-shape group ***access-list-number shaped-rate *{**Bc**} {**Be**}

Verify with show traffic-shape *interface, *show traffic-shape statistics and show traffic-shape queue

## CB Shaping

**shape **[**average | peak**] *mean-rate *[[*burst-size*] [ *excess-burst-size*]]

```
policy-map shape-all
 class class-default
  shape average 64000

int Se0/0/0/.1
 service-policy output shape-all
```

CB shaping calculates values based on whether shaping rate exceeds 320kbps: -

Variable - Rate <= 320 kbps - Rate > 320kbps

- Bc - 8000 bits - Bc = shaping rate * Tc
- Be - Be = Bc = 8000 - Be = Bc
- Tc - Tc = Bc/shaping rate - 25 ms

### Tuning for Voice using LLQ and small Tc

```
class-map match-all voip-rtp
 match ip rtp 16384 16383

policy-map queue-voip
 class voip-rtp
  priority 32
 class class-default
  fair-queue

policy-map shape-all
 class class-default
  shape average 96000 960
  service-policy queue-voip

int Se0/0.1
 service-policy output shape-all
```

### Shaping by bandwidth percent

**shape average 50 125 ms** - 50 is shaper rate, 125 ms is the Bc with ms after. The ms required otherwise command rejected

### CB Shaping to peak rate

**shape peak ***mean-rate*

### Adaptive shaping

Just use **shape adaptive ***min-rate *under shape command in class configuration.

## Policing

**police ***bps burst-normal burst-max ***conform-action ***action ***exceed-action ***action *[ **violate-action ***action *]

### Single rate three colour

```
policy-map police-all
 class class-default
  police cir 96000 bc 12000 be 6000 conform-action transmit exceed-action set-dscp-transmit 0 violate-action drop
```

### Policing subset

```

class-map match-all match-web
 match protocol http

policy-map police-web
 class match-web
  police cir 80000 bc 10000 bc 5000 conform-action transmit exeed-action transmit violate-action drop
 class class-default
  police cir 16000 bc 2000 be 1000 conform-action transmit exceed-action set-dscp-transmit 0 violate-action set-dscp-transmit 0

```

### Dual rate Policing

**police **{ **cir ***cir *] [ **bc ***conform-burst *] { **pir ***pir*} [ **be ***peak-burst *] [**conform-action ***action ***exceed-action ***action *[ **violate-action ***action *]]

### Multi-action Policing

```

policy-map testpol1
 class class-default
  police 128000 256000
   conform-action transmit
   exceed-action transmit
   violate-action set-dscp-transmit0
   violate-action set-frde-transmit
```

### Policing by percentage

```
policy-map test-pol6
 class class-default
    police cir percent 25 bc 500 ms pir percent 50 be 500 ms conform transmit exceed transmit violate-drop
```

## Committed Access Rate

**rate-limit **{ **input | output **} [ **access-group **[ **rate-limit **] *acl-index *] *bps burst-normal burst-max ***conform-action ***action ***exceed-action ***action*

```
int Se0/0
 rate-limit input 496000 62000 62000 conform-action continue exceed-action drop
 rate-limit input access-group 101 400000 50000 50000 conform-action transmit exceed-action drop
 rate-limit input access-group 102 160000 20000 20000 conform-action transmit exceed-action drop
 rate-limit input access-group 103 200000 25000 25000 conform-action transmit exceed-action drop
```

The continue action means packets that conform continue through, and then potentially match against other services.

## Hierarchical Queuing Framework (HQF)

```

policy-map class
 class c1
  bandwidth 14
 class c2
  bandwidth 18

policy-map map1
 class class-default
  shape average 64000
  service-policy class

policy-map map2
 class class-default
  shape average 96000

map-class frame-relay fr1
 service-policy output map1

map-class frame fr2
 service-policy output map2

interface Se4/1
 encapsulation frame-relay
 frame-relay interface-dlci 16
  class fr1
 frame-relay interface-dlci 17
  class fr2
```

```

policy-map class
 class c1
  bandwidth 14
 class c2
  banwidth 18

policy-map map1
 policy-map child
 class child-c1
  bandwidth 400
 class child-c2
  bandwidth 400

policy-map parent
 class parent-c1
  bandwidth 1000
  service-policy child
 class parent-c2
  bandwidth 2000
  service-policy child
```


## Verification commands

Can verify response time with IP SLA between source and destination. Run **show ip sla statistics** to verify.

**show policy-map** - shows configured policy maps
**show class-map ** - displays associated class maps
**show policy-map interface** - what policies on an itnerface and actions being taken

** show mls qos**
** show mls qos input-queue**
** show mls qos maps cos-input-q**
** show mls qos maps cos-output-q**
** show mls qos maps cos-dscp**
** show mls qos maps dscp-cos**


- Troubleshooting QoS misconfig - Verify QoS is enabled, class map config, policy map config, and service policy operation - **show mls qos, show class-map, show policy-map, show policy-map interface**
- Pssible switch QoS misconfig - show commands to determine how input/egress queueing configured - **show mls qos input-queue, show mls qos interface ***interface ***queueing, show mls qos maps cos-input-q, show mls qos maps cos-output-q, show mls qos maps cos-dscp, show mls qos maps dscp-cos**
- Possible router Qos - show commands to determine how queueing configured - **show mls qos maps, show traffic-shape**

# Wide Area networks

## HDLC

With back-to-back serial, router connected to DCE (Data Communications Equipment) end of cable provides clock signal for serial link. This done with **clockrate** command. To see which end of cable interface is on, use **show controllers.**

## PPP

```

username R4 password 0
rom 838

int Se0/1/0
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp quality 80
 ppp authentcation chap
```

## MLPPP

```

int Multilink1
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp multilink
 ppp multilink group 1

int Se0/1/0
 no ip address
 encapsulation ppp
 ppp multilink group 1

int Se0/1/1
 no ip address
 encapsulation ppp
 ppp multilink group 1
```

Use show int multilink1 to show if multilink open

## MLPPP LFI

```

int Multilink 1
 bandwidth 256
 ip address 10.1.34.3 255.255.255.0
 encapsulation ppp
 ppp multilink group 1
 ppp multilink fragment-delay 10
 ppp multilink8 interleave
 service-policy output queue-on-dscp
```

## PPP Compression


### Payload

Use a matching **compression **command under each interface on each end of link.

### Header

Legacy commands are **ip tcp header-compression **[ **passive **] and **ip rtp header-compression **[ **passive **].

For MQC

```
policy-map cb-compression
 class voice
  bandwidth 82
  compress header ip rtp
 class critical
  bandwidth 110
  compression header ip tcp

int Multilink1
 bandwidth 256
 service-policy output cb-compression
```

## PPPoE

### Server config

BBA (broadband aggregation) group created to handle incoming PPPoE config

```
bba-group pppoe BBA-GROUP
 virtual-template 1
 sessions per-mac limit 2
```

The limit means not allow many macs to use the session (would allow a new session to be established immediately if prior session dropped when using 2 as the limit)

```
int virtual-template 1
 ip address 10.0.0.1 255.255.255.0
 peer default ip address pool PPPOE_POOL
```

When PPPoE client initiates a session with router, router dynamically creates virtual interface. Interface acts as placeholder for P2P connection spawned by this process.

The virtual template needs two components, an IP address and pool of IP addresses that is used to negotiate addresses to clients.

Pool needs defining to issues addresses in the pool.

```
ip local pool PPPOE_POOL 10.0.0.2 10.0.0.254
```

Final step to enable PPPoE group on interface facing the client: -

```
interface Fa0/0
 no ip address
 pppoe enable group MyGroup
 no shutdown
```

### Client config

Need to create dialer and then associate it with physical interface

```
int dialer1
 dialer pool 1
 encapsulation ppp
 ip address negotiated
```

PPP header adds 8 bytes of overhead to each frame. If ethernet using 1500 byte MTU, need to set MTU of diuialer to 1492 to avoid fragging

```
int dialer 1
 mtu 1492

int fa0/0
 no ip address
 pppoe-client dial-pool-number 1
 no shutdown
```

Verify with show ip int brief

show pppoe session  - Shows details of the PPPoE session


### Authentication


PPP can use PAP or CHAP to authentication clients, with latter preferred.

```
username PPP password PPPpassword

int virtual-template 1
 ppp authentication chap callin

int dialer 1
 ppp chap password MyPassword
```

# Introduction to IP Multicasting

## CGMP

```
int Fa0/1
 ip cgmp
```

## IGMP Snooping

```
ip igmp snooping
no ip igmp snooping vlan 20
ip igmp snooping last-member-query-interval 500
ip igmp snooping vlan 22 immediate-leave
```

## RGMP

Interface config of **ip rgmp**

## IGMP Proxy

Config on upstream: -

```
int Gi0/0
 ip address 10.1.1.1 255.255.255.0
 ip pim dense-mode

int Gi1/0/0
 ip address 10.2.1.1 255.255.255.0
 ip igmp unidrectional-link
 ip pim dense-mode
```

Config on downstream

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
 ip pim dense-mode
 ip igmp unidirectional-link

int Gi1/0/0
 ip address 10.5.1.1 255.255.255.0
 ip pim sparse-mode
 ip igmp mroute-proxy lo0
```

# Multicast Routing

## Dense Mode

Config required to enable is **ip multicast-routing** and **ip pim dense-mode** on all interfaces it is required on.

## Sparse Mode

```
ip multicast-routing
ip pim sparese-mode
ip pim rp-address X.X.X.X
```

## SPT to source-specific SPT

CXisco routers witch over from SPT to source-specific SPT after they receive first packet from shared tree.

Can change this with **ip pim spt-threshold ***rate*. Can be done on any router in a group, traffic exceeds rate in kbps to switch over.

## AutoRP

Normal router: -

```
ip multicast-routing

int Se0
 ip pim sparse-mode

ip pim autorp listener
```

Auto-RP Mapping Agent

```
ip multicast-routing

ip pim send-rp-discovery scope 10 (can designate a source intercace)

int Se0
 ip pim sparse-mode
```

Auto-RP RP

```
ip multicast routing
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim send-rp-announce loopback0 scope 10
```

## BSR

BSR

```
ip multicast-routing

int lo0
 ip pim sparse-mode

int Se0
 ip pim sparse-mode

ip pim bsr-candidate Lo0 0 (0 priority, the default)
```

On RP

```
ip multicast-routing

int lo2
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

ip pim rp-candidate Lo2
```

## Anycast RP with MSDP

Configure multiple routers with same address and use as RP address, anycasted

## Interdomain Multicast Routing with MSDP

```
int Lo2
 ip address 10.1.10.3 255.255.255.255
 ip pim sparse-mode

ip multicast-routing
ip pim rp-candidate Lo2
ip msdp peer 172.16.1.1
```

```
int lo0
 ip address 172.16.1.1 255.255.255.255
 ip pim sparse-mode

ip multicast-routing
ip pim rp-candidate Lo0
ip msdp peer 10.1.10.3 connect-source Lo0
```

**show ip msdp peer**

## SSM

SSM uses IGMPv3. Enable globally with **ip pim ssm { default | range ***access-list* **}. **Address range of 232.0.0.0/24 is SSM range (decreed by IANA)

```
ip multicast-routing

int Fa0/0
 ip pim sparse-mode
 ip igmp version 3

ip pim ssm default
```

## v6 PIM

Need to enable mcast routing for v6 through global config, **ipv6 multicast-routing.** his enables it on all interfaces. Default config of an in ipv6 interfaces assumes v6 pim, and does not appear in interface config. Always operates in sparse mode.

### DR Priority manipulation

```
int F0/0
 ipv6 pim dr-priority <0-4294967295> - Higher is better
```

### v6 Static RP

```
ipv6 pim rp-address 2001:2:2:2::2
```

### v6 BSR

```
ipv6 pim bsr candidate bsr 2001:2:2:2::2
ipv6 pim bsr candidate rp 2001:1:1:1::1
ipv6 pim bsr candidate rp 2001:3:3:3::3
```

Verify with **show ipv6 pim bsr rp-cache** to show whats in the Cache receives from RPs

**show ipv6 bsr candidate-rp**

### MLD

Statically join a group under an interface with **ipv6 mld join-group ***group-address*

IGMP replaced by MLD in v6. MLDv1 similar to v2, MLDv2 similar to v3. MLDv2 supports SSM in v6

**ipv6 mld limit** - Limit number of receivers
**ipv6 mld join-group** - Permanently subscribe an interface

The **ipv6 multicast-routing** not only enables PIM by default, but MLD auto config too.


**show ipv6 pim interface ** - Shows interfaces with PIM on, tunnels and DRs**
****
****show ipv6 mld interface - **Shows MLD timers, versions, activity, querying router etc

**show ipv6 pim traffic** shows PIM traffic traversing a router

### Embedded RP

FF7<scope>:0<RP interface ID><Hex prefix length>:<64-bit RP prefix>:<32 bit group ID>:<1-F>

Using 2001:2:2:2::2/64, RP interface is IS 2 (taken from ::2), prefix length is 64 (40 in hex), RP Prefix is 2001:2:2:2

Global scope, 32 bit group ID commonly 0.

FF7E:0240:2001:2:2:2:0:1

erify with **show ipv6 mroute, show ipv6 pim group-map**

Make sure a router knows it is an RP, can set with **ipv6 pim rp-address** then use above Embedded address style for group joins on other routers.

# Device and Network Security

## Simple Password Proection for CLI

```

line con 0
 login
 password dave

line vty 0 15
 login
 password barney
```

**service password-encryption** - Encrypts passwords in config

## Enable passwords

Define enable password with **enable password ***pw *or **enable secret ***pw. *If both defined, enable exec command only accepts password defined in secret.

## SSH

Telnet enable by default, SSH needs following: -


1. IOS SSH support, K9 image required
2. Configure a hostname
3. Configure a domain name
4. Configure a client auth method
5. Tell the router/switch to generate RSA keys to encrypt the session
6. Specify SSH version if v2 required
7. Disable telnet on VTY lines
8. Enable SSH on VTY lines

```
hostname R3

ip domain-name CCIE2b
username cisco password DAVE-LIKES-SSH
crypto key gen rsa

ip ssh version 2

line vty 0 4
 transport input none
 transport input ssh
```

## AAA Default Set of methods

```

enable secret 5 <MD5-HASH>
username cisco password 0 cisco

aaa new-model

aaa authentication enable default group radius local
aaa authentication login default group radius none

radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
```

## Multiple auth methods

Four methods per **aaa authentication** command.

Methods available

- **group radius** - Use configured RADIUS servers
- **group tacacs+ - **Use configured TACACs servers
- **aaa group server ldap** - Defines AAA server group with a group name and enters LDAP server group config mode
- **group ***name - *Use a defined group of either RADIUS or TACACS+ servers
- enable - Use enable password
- line - Use password command in line config (cannot be used with enable auth)
- local - Use username commands in local config, username case insensitive, password case sensitive
- local-case - As above, treats both as case sensitive
- none - No auth, user automatically authd

## Group of AAA servers

```

aaa group server radius fred
 server 10.1.1.3 auth-port 1645 acct-port 1646
 server 10.1.1.4 auth-port 1645 acct-port 1646

aaa new-model
aaa authentication enable default group fred local
aaa authentication login default group fred none
```

## Overriding defaults for login

```

aaa authentication login for-console group radius line
aaa authentication login for-vty group radius local
aaa authentication login for-aux group radius

line con 0
 password 7 1489247814
 login auth for-console

line aux 0
 login auth for-aux

line vty 0 4
 password 7 104D0000A0618
 login authentication for-vty
```

## PPP Security

Steps to use AAA for PPP are: -


1. Enable **aaa new-model**
2. Configure RADIUS and/or TACACS+ servers
3. **aaa authentication ppp default **
4. Use **aaa authentication ppp ***list-name method1 method2*
5. For groups use **ppp authentication ***protocol list-name eg *ppp authentication chap fred

## Switch Security best practices

Unused/user port: -

```

int fa0/0
 no cdp enable
 switchport mode access
 switchport nonegotiate
 spanning-tree guard root
 spanning-tree bpduguard enable
```

## Port Security

Port must be statically set to trunk or access, not dynamically learnt


- switchport port-security [maximum *value*] - Default is 1
- switchport port-security mac-address *mac-address *[ vlan { *vlan-id *| { access | voice }}] - Statically defines allowed MAC, for a particular VLAN (for trunking), and for either access or voice VLAN
- switchport port-security mac-address sticky - Switch remembers dynamically learned MAC
- switchport port-security [aging] [violation { protect | restrict | shutdown } ] - Defines aging timer and actions taken when violation occurs

Using **show port-security interface ***INTERFACE - *SecureUp means port is up and secured

## Dynamic Arp Inspection

DHCP snooping needs to be enabled before DAI can use DHCP snooping binding database. Also can configure static IPs, or perform additional validation (last 3 steps above) using **ip arp inspection validate**


- ip arp inspection vlan *vlan-range - *Global commands, enables DAI on this switch for specified VLANs
- [no] ip arp inspection trust - Interface sub command, defaults to enabled after above command added
- ip arp inspection filter *arp-acl-name *vlan *vlan-range *[static] - Refers to ARP ACL that defines static IP/MAC addresses to be checked by DAI for that VLAN
- ip arp inspection validate {[src-mac] [dst-mac] [ip]} - Additional option checking (as per above)
- ip arp inspection limit {rate *pps *[burst interval *seconds*] | none} - Limits ARP message rate to prevent DOS attacks carried out by sending a large number of ARPs

## DHCP Snooping

```

- ip dhcp snooping vlan *vlan-range * - Enables DHCP snooping on one or more VLANs
- [no] ip dhcp snooping turst - interface level command to enable or disable trust level, no trust by default
- ip dhcp snooping binding *mac-address *vlan *vlan-id ip-address *interface *interface-id *expiry *seconds - *Adds static entries to DHCP snooping database
- ip dhcp snooping verify mac-address - Adds optional check from step 3
- ip dhcp snooping limit rate *rate - *Maximum number of DHCP messages per second
```

## IP Source Guard

```
ip dhcp snooping

int Fa0/1
 switchport access vlan 3
 ip verify source
```

Check just source IP with **ip verify source**, check IP and MAC with **ip verify source port-security**. Can use **ip source binding ***mac-address ***vlan ***vlan-id ip-address ***interface ***interface-id* to create static entries in addition to database.

 **show ip dhcp snooping binding**

## 802.1x using EAP

```

aaa new model
aaa authentication dot1x default group radius
dot1x system-auth-control

radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
radius-server key cisco

int Fa0/1
 authentication port-control force-authorized

int Fa0/2
 authentication port-control force-authorized

int fa0/3
 authentication port-control auto

int fa0/4
 authentication port-control auto

int fa0/5
 authentication port-control force-unauthorized
```

## Storm Control

```

int Fa0/0
 storm-control broadcast level pps 100 50
 storm-control mutlciast level 0.50 0.40
 storm-control unicast level 80.00
 storm-control action trap
```

## ACLs

ip access-group adds to an itnerface
access-class adds to a line

ip access-list resqeuence can redefine sequence numbers for crowded ACL

show ip interface shows ACLs enabled on interface

show access-list - ACLs for all protocols

show ip access-list - IP acls only

## Smurf Attacks, Directed Boradcasts, RPF checks

As of Cisco IOS 12.0, **no ip directed-broadcast** exists, prevents routers from forwarding broadcast onto LAN. Also uRPF check could be enabled

**ip verify unicast source reachable-via {rx | any } [allow-default] [allow-self-ping] [ ***list *]

## TCP Intercept

```

ip tcp intercept list match-tcp-from-internet
ip tcp intercept mode watch
ip tcp intercept watch-timeout 20

ip access-list extended match-tcp-from-internet
 permit tcp any 1.0.0.0 0.255.255.255

 int Se0/0
```

## Cisco Classic Firewall with CBAC


1. Choose interface (inside or outside)
2. Configure ACL that denies all traffic to be inspected
3. Configure global timeouts and thresholds using **ip inspect**
4. Define inspection rule and optional rule-specific timeout value using **ip inspect name ***protocol *commands, eg **ip inspect name actionjackson ftp timeout 3600**
5. Apply inspection rule to an interface, **ip inspect actionjackson in**
6. Apply ACL to same interface as inspection rule, but in opposite direction

## ZBF


1. Decide zones and create them
2. Decide traffic between zones, and create zone-pairs
3. Create zclass maps to identify interzone traffic that must be inspected by fw
4. assign policies to traffic by creating policy maps and associating class maps with them
5. Assign policy maps to appropriate zone-pair
6. Assign interfaces to zones (interfaces can be in only one zone)

```
zone security LAN
 decription LAN zone

zone security WAN
 description WAN zone

zone-pair security Internal source LAN destination WAN

zone-pair security External source WAN destination LAN
```

```

ip access-list extended LAN_Subnet
 permit ip 10.1.1.0 0.0.0.255 any

ip access-list extended Web_Servers
 permit tcp 10.1.1.0 0.0.0.255 host 10.150.2.1
 permit tcp 10.1.1.0 0.0.0.255 host 10.150.2.2

class-map type inspect match-all Corp_Servers
 match access-group name Web_Servers
 match protocol http <---- NBAR

class-map type inspect Other_HTTP
 match protocol http
 match access-group name LAN_Subnet

class-map type inspect ICMP
 match protocol ICMP

class-map type inspect Other_Traffic
 match access-group name LAN_Subnet
```

Following actions cab be taken in policy maps when associated with class maps: -


- drop - drops packet
- Inspect - uses CBAC
- Pass - passes packet
- police - policies traffic
- service-policy - Use EPI Engine
- urlfilter - uses URL filtering engine

```

parameter-map type inspect Timeouts
 tcp idle-time 300
 udp idle-time 300

policy-map type inspect LAN2WAN
 class type inspect Corp_Servers
  inspect
 class type inspect Other_HTTP
  inspect
  police rate 1000000 burst 8000
 class type inspect ICMP
  drop
 class type inspect Other_Traffic
  inspect Timeouts
```

```

zone-pair security Internal source LAN destination WAN
 service-policy type inspect LAN2WAN

int Fa0/1
 zone-member security LAN

int Se1/0/1
 zone-member security WAN
```

**show zone-pair security**

## CoPP

```

Extended access-list BAD-STUFF
 10 permit tcp any any eq 554
 20 permit tcp any any eq 9996
 30 permit ip any any fragments

Exteed IP access list INTERACTIVE
 10 permit tcp 10.17.4.0 0.0.3.255 host 10.17.3.1 eq 22
 20 permit tcp 10.17.4.0 0.0.3.255 eq 22 host 10.17.3.1 established

Extended IP access list ROUTING
 10 permit tcp host 172.20.1.1 gt 1024 host 10.17.3.1 eq bgp
 20 permit tcp host 172.20.1.1 eq bgp host 10.17.3.1 gt 1024 established
 30 permit eigrp 10.17.4.0 0.0.3.255 host 10.17.3.1

Class Map match-all CoPP_ROUTING
 Match access-group name ROUTING

Class Map match-all CoPP_BAD_STUFF
 Match access-group name BAD_STUFF

Class Map match-all CoPP_INTERACTIVE
 Match access-group name INTERACTIVE

Policy Map CoPP
 Class CoPP_BAD_STUFF
  police cir 8000 bc 1500
   conform-action drop
   exceed-action drop
 Class CoPP_ROUTING
  police cir 200000 bc 6250
   conform-action transmit
   exceed-action transmit
 Class CoPP_INTERACTIVE
  police cir 10000 bc 1500
   conform-action transmit
   exceed-action transmit
 Class class-default
  police cir 10000 bc 1500
   conform-action transmit
   exceed-action transmit

control-plane
 service-policy input CoPP
```

## DMVPN

### Basic IP Config

```

R1

int Fa0/0
 ip address 192.168.123.1 255.255.255.0

int Lo0
 ip address 1.1.1.1 255.255.255.255

R2

int Fa0/0
 ip address 192.168.123.2 255.255.255.0

int Lo0
 ip address 2.2.2.2 255.255.255.255

R3

int Fa0/0
 ip address 192.168.123.3 255.255.255.0

int Lo0
 ip address 3.3.3.3 255.255.255.255
```

### GRE MP Tunnel

```

R1 (HUB)

int tun0
 ip address 172.16.123.1 255.255.255.0
 tunnel mode gre multipoint
 tunnel source fa 0/0
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp authentication cisco

R2 (SPOKE)

int tun0
 ip address 172.16.123.2 255.255.255.0
 tunnel mode gre multipoint
 ip nhrp authentication cisco
 ip nhrp map multicast dynamic
 ip nhrp map 172.16.123.1 192.168.123.1
 ip nhrp map multicast 192.168.123.1
 ip nhrp network-id 1
 ip nhrp nhs 172.16.123.1
 tunnel source Fa0/0

R3

int tun0
 ip address 172.16.123.3 255.255.255.0
 tunnel mode gre multipoint
 ip nhrp authentication cisco
 ip nhrp map multicast dynamic
 ip nhrp map 172.16.123.1 192.168.123.1
 ip nhrp map multicast 192.168.123.1
 ip nhrp network-id 1
 ip nhrp nhs 172.16.123.1
 tunnel source Fa0/0
```

### Config IPsec

```

All devices

crypto isakmp policy 1
 encryption aes
 hash md5
 authentication pre-share
 group 2
 lifetime 86400

cryto isakmp key 0 TEST address 0.0.0.0

crypto ipsec transform-set MYSET esp-aes esp-md5-hmac

crypto ipsec profile MGRE
 set security-association lifetime seconds 86400
 set transform-set MYSET

int Tun0
 tunnel protection ipsec profile MGRE
```

### DMVPN routing

```

R1 - HUB

router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
 network 172.16.123.0 0.0.0.255 area 0

int tun 0
 ip ospf network broadcast

R2

router ospf 1
 network 2.2.2.2 0.0.0.0 area 0
 network 172.16.123.0 0.0.0.255 area 0

int tun 0
 ip ospf network broadcast
 ip ospf priority 0

R3

router ospf 1
 network 3.3.3.3 0.0.0.0 area 0
 network 172.16.123.0 0.0.0.255 area 0

int tun 0
 ip ospf network broadcast
  ip ospf priority 0
```

## v6 Security

### Securing at First Hops

```

ipv6 acces-list ACCESS_PORT
 remark Block all traffic DHCP server -> client
 deny udp any eq 547 any eq 546
 remark Block Router Advertisements
 deny icmp any any router-advertisements
 permit any any

int Gi1/0/1
 switchport
 ipv6 traffic-filter ACCESS_PORT in
```

### RA Guard

```
ipv6 nd raguard policy POLICY-NAME
 device-role {host | router}

int Fa0/0
 ipv6 nd raguard attac-policy POLICY-NAME
```

### DHCPv6 Guard

```

ipv6 access-list acl1
 permit host FE80::A8BB:CCFF:FE01:F700 any

ipv6 prefix-list abc permit 2001:0DB8::/64 le 128

ipv6 dhcp guard policy pol1
 device-role server
 match server access-list acl1
 match reply prefix-list abc
 preference min 0
 preference max 255
 trusted-port

int Gi1/0/1
 switchport
 ipv6 dhcp guard attach policy pol1 vlan add 1

show ipv6 dhcp guard policy pol1
```

### DHCPv6 Guard and Binding Database

** show ipv6 neighbors binding**

```

ipv6 access-list dhcpv6_server
 permit host FE80::1 any
 ipv6 prefix-list dhcpv6_prefix permit 2001:DB8:1::/64 le 128

ipv6 dhcp guard policy dhcpv6guard_pol
 device-role server
 match server access-list dhcpv6_server
 match reply prefix-list dhcpv6_prefix
 vlan configuration 1
  ipv6 dhcp guard attach-policy dhcpv6guard_pol
```

```

ipv6 nd raguard policy ra_pol
 device-role router
 trusted-port

int Gi1/0/1
 ipv6 nd raguard attach-policy ra_pol
```


### IPv6 device tracking

```

ipv6 neighbor binding vlan 100 interface Gi1/0/1 reachable-lifetime 100
ipv6 neighbor binding max-entries 100
ipv6 neighbor binding logging
```

### IPv6 Neighor Discovery Inspection

```

ipv6 nd inspection policy example_policy
 device-role switch
 drop-unsecure
 limit address-count 1000
 tracking disable stale-lifetime infinite
 trusted port
 validate source-mac
 no validate source-mac
 default limit address-count
```

Verify above with show ipv6 nd inspection policy example_policy

Apply it to an interface with **ipv6 nd inspection attach-policy ***policy-name*

### IPv6 Source Guard

```

ipv6 source-guard policy example_policy
 deny global-autoconf < --- Denies data traffic from auto-config'd global addresses
 permit link-local <--- Allow data traffic that is sourced by a link-local address

int Gi1/0/1
 ipv6 source-guard attach-policy example_policy
```

## PACL

PACL processed first by switch IOS, then the VACL.

```
int Gi1/0/1
 ip access-group PACLIPList in
 mac access-group PACLMACList in
```

# Tunneling Technologies

## GRE Tunnel Config

```

int Lo0
 ip address 150.1.2.2 255.255.255.0

int Tun0
 ip address 192.168.201.2 255.255.255.0
 tunnel source Lo0
 tunnel destination 150.1.3.3
```

## DMVPN tunnels

### Phase 1

```

crypto isakmp policy 1
 encr 3des
 authentication pre-share
 group 2
 crypto isakmp key cisco123 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TSET esp-3des esp-sha-hmac
 mode transport <--- Transport mode decreases IPSec packet size

crypto ipsec profile DMVPN
 set transform-set TSET

int Tun0
 ip address 172.16.145.1 255.255.255.0
 ip mtu 1400
 ip nhrp authentication cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 12344
 no ip split-horizon eigrp 145 <--- Required so that protocol able to send routes gathered from one spoke to another
 tunnel source Fa0/0
 tunnel mode gre multipoint
 tunnel key 12345
 tunnel protection ipsec profile DMVPN
```

### Phase 2

DMVPN phase 2 introduces the direct spoke-to-spoke comms through DMVPN network. To allow this with EIGRP for example, use** no ip next-hop-self eigrp ***as, *which stops labelling routes for spokes as via the hub.

### Phase 3

```

crypto isakmp policy 10
 encr 3des
 authentication pre-share
 group 2
 crypto isakmp key cisco123 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TSET esp-3des esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN
 set transform-set TSET

int Tun0
 ip address 172.16.245.2 255.255.255.0
 ip mtu 1400
 ip nhrp authentication cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 123
 ip nhrp redirect
 tunnel source S0/1/0
 tunnel mode gre multipoint
 tunnel key 123
 tunnel protection ipsec profile DMVPN
 no ip split-horizon eigrp 245
```

## v6 Tunneling

### Manually Configured

```

int tun0
 no ip address
 ipv6 address 2001:DB8::1:1/64
 tunnel source Lo0
 tunnel destination 127.30.20.1
 tunnel mode ipv6ip
```

### Automatic v4-compatible tunnels

Tunnel destination automatically determined from low-order 32 bit of tunnel interface address. To use, use mode of **tunnel mode ipv6ip auto-tunnel**

### IPv6-over-v4-GRE

Only difference between this and manual config is using **tunnel mode gre ipv6**.

### Auto 6to4

```

int Fa0/0
 ipv6 address 2002:0a01:6401:1::1/64

int Fa0/1
 ipv6 address 2002:0a01:6401:2::1/64

int E2/0
 ip address 10.1.100.1 255.255.255.0

int tun0
 no ip address
 ipv6 address 2002:0a01:6401::1/64
 tunnel source Eth 2/0
 tunnel mode ipv6ip 6to4

ipv6 route 2002::/16 tunnel 0
```

### ISATAP

Tunnel mode used is **ipv6ip isatap**, and v6 address derived using EUI-64 method. EUI-64 for tunnels derives last 32 bits of interface ID from tunnel source interfaces v4 address.

Enable RAs using **no ipv6 nd suppress-ra.**

## L2VPNs

### AToM

```

R2

int Fa0/0
 xconnect 4.4.4.4 204 encapsulation mpls

R4

int fa0/0
 xconnect 2.2.2.2 204 encapsulation mpls
```

## GETVPN

```

ip domain-name cisco.com
crypto key gen rsa mod 1024

crypto isakmp policy 10
 authentication pre-share

crypto isakmp key GETVPN-R5 address 10.1.25.5
crypto isakmp key GETVPN-R4 address 10.1.24.4

crypto ipsec transform-set TSET esp-aes esp-sha-hmac

crypto ipsec profile GETVPN-PROF
 set transform-set TSET

crypto gdoi group GETVPN
 identity number 1
 server local
```

specify Rekey parameters. Rekey can be performed in two ways: -


- Unicast - When multicast not supported, KS sends down a Rekey packet to every GM it knows of
- Multicast - KS generates only one packet and sends it down to all GMs at once

```
rekey authentication mypubkey rsa R1.cisco.com
rekey transmit 10 number 2
rekey transport unicast

authorizationa address ipv4 GM-LIST
```

```

sa ipsec 1
 profile GETVPN-PROF
 match address ipv4 LAN-LIST
 replay counter window-size 64
 address ipv4 10.1.12.1

ip access-list standard GM-LIST
 permit 10.1.25.5
 permit 10.1.24.4

ip access-list extended LAN-LIST
 deny udp any eq 848 any eq 848
 permit ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
```

```

R5

crypto isakmp policy 10
 authentication pre-share

crypto isakmp key GETVPN-R5 address 10.1.12.1

crypto gdoi group GETVPN
 identity number 1
 server address ipv4 10.1.12.1

# Below ACL option, used if some traffic should be excluded (eg SSH)

ip access-list extended DO-NOT-ENCRYPT
 deny tcp 192.168.4.0 0.0.0.255 eq 22 192.168.5.0 0.0.0.255

crypto map CMAP-GETVPN 10 gdoi
 set group GETVPN
 match address DO-NOT-ENCRYPT

int Se0/1/0.52
 crypto map CMAP-GETVPN
```

Verify with **show crypto gdoi group ***group-name *and **show crypto gdoi ks policy, show crypto gdoi ks acl, show crypto gdoi ks members**

# MPLS


## MPLS config on LSRs for unicast IP support

```

ip cef

int type x/y/x
 mpls ip

router eigrp 1
 network ...
```
**show mpls ldp bindings ***route *will show LIB entries, remote bindings received, and local binding (label allocated by itself).

show mpls forwarding table *route * - Shows local entry, outgoing tag (label) and outgoing interface

show ip cef *route *internal - Shows FIB entry

show mpls ldp bindings - Shows LIB entries

## MPLS VPN-IPv4

```

ip vrf Cust-A
 rd 1:111
 route-target import 1:100
 route-target export 1:100

ip vrf Cust-B
 rd 2:222
 route-target import 2:200
 route-target export 2:2000

int Fa0/1
 ip vrf forwarding Cust-A
 ip address 192.168.15.1 255.255.255.0

int Fa0/0
 ip vrf forwarding Cust-B
 ip adress 192.168.16.1 255.255.255.0
```

```

CE config

router eigrp 1
 network 192.168.15.0
 network 10.0.0.0

PE config

router eigrp 65001
 address-family ipv4 vrf Cust-A
  autonomous-system 1
  network 192.168.15.1 0.0.0.0
 address-family ipv4 vrf Cust-B
  autonomous-system 1
  network 192.168.16.1 0.0.0.0
  no auto-summary
```

```

router bgp 65001
 address-family ipv4 vrf Cust-A
  redistribute eigrp 1
 address-family ipv4 vrf Cust-B
  redistribute eigrp 1

router eigrp 65001
 address-family ipv4 vrf Cust-A
  redistribute bgp 65001 metric 10000 1000 255 1 1500
 address-family ipv4 vrf Cust-B
  redistribute bgp 65001 metric 5000 500 255 1 1500
```

```

router bgp 65001
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 update-source loop0
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community
```

## VRF lite without MPLS

```

ip cef

ip vrf COI-1
 rd 11:11
 route-target both 11:11

ip vrf COI-2
 rd 22:22
 route-target both 22:22

int Se0/0/0
 encap frame-relay
 no shut
 desc to RouterLite2

int Se0/0/0.101 point-to-pint
 frame-relay interface-dlci 101
 ip vrf forwarding COI-1
 ip address 192.168.4.1 255.255.255.252

int Se0/0/0.101 point-to-pint
 frame-relay interface-dlci 101
 ip vrf forwarding COI-2
 ip address 192.168.4.5 255.255.255.252
```

