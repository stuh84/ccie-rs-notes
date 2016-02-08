# Router and Switch Device Security

## Simple Password Proection for CLI

```
line con 0
 login 
 password dave

line vty 0 15
 login
 password barney
```

* **service password-encryption** 
 * Weak encryption
 * Not changed in startup until copy run start or wr mem
* Not auto decrypted on **no service password-encryption**

## Better Protection of enable and username passwords

* **enable password** and **enable secret**
* Secret stored as MD5 hash, not affected by service password-encryption
* If both defined, precedence on secret

## Using SSH

* Auth with local un and pw, or AAA server
* v1 vulnerable to MITM attacks
* Telnet enabled by default

Steps for SSH: -

1. IOS SSH support, K9 image required
2. **hostname**
3. **ip domain-name**
4. Client auth method
5. Generate RSA keys
6. Specify SSH version if v2 required
7. Disable telnet on VTY
8. Enable SSH on vty

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

## User Mode and Privileged Mode AAA Auth

* Strongest method to protect CLI is TACACS+ or RADIUS (eg ACS)
* Router sends un and pw encrypted to server, awaits reply before
  rejecting

|Feature|Radius|TACACS+|
|-------|------|-------|
|Scope of encryption|Password Only|Entire Payload|
|L4 Protocol|UDP|TCP|
|Well-Known Port/IOS default for Auth|1812/1645|49/49|
|Standard or Prop|RFC2865|Prop|

### Using default set of auth methods

* Single auth method a way to auth user
* one could be a radius to auth
* Another to let router look at local
* List tried in order until an auth response (accepting or rejecting
  user)
* Simple AAA config defines default auth methods for all logins, and
  second set for enable
* Defined default login apply to all login session (Console/Telnet/Aux)

1. Enable AAA auth with **aaa new-model**
2. Define servers and keys, **radius-server host, radius-server key,
tacacs-server host, tacacs-server key**
3. Define default auth methods for all CLI access **aaa authentication
login default**
4. Define default set of auth methods for enable **aaa authentication
enable default**

```
enable secret 5 <MD5-HASH>
username cisco password 0 cisco

aaa new-model

aaa authentication enable default group radius local
aaa authentication login default group radius none
radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
```

### Using Multiple Auth Methods

* Four methods in command
* No practical limit to servers
* Logic is, try first method, if does not responde, move to next
 * First to respond with auth or reject stops process
 * If no response for any method, reject request

|Method|Meaning|
|------|-------|
|**group radius**|Use config'd RADIUS servers|
|**group tacacs+**|User config's TACACS sevrers|
|**aaa group server ldap**|AAA Server group with name and enters LDAP config mode|
|**group** *name*|Use defined group of either servers|
|**enable**|Use enable password|
|**line**|Use config in line (cant use with enable auth|
|**local**|Username commands in local config, username case insensitive, password sensitive|
|**local-case**|Above, but case sensitive username|
|**none**|No auth, user automatically auth'd|

### Groups of AAA Servers

* By default, all radius servers in RADIUS group, same for TACACSs
* Can create groups

```
aaa group server radius fred
 server 10.1.1.3 auth-port 1645 acct-port 1646
 server 10.1.1.4 auth-port 1645 acct-port 1646

aaa new-model
aaa authentication enable default group fred local
aaa authentication login default group fred none
```

### Overriding Defaults for Login Security

* Console VTY and Aux can override
* In line config, **login authentication** *name*, point to named config
  methods

```
aaa authentication login for-console group radius line
aaa authentication login for-vty group radius local
aaa authentication login for-aix group radius

line con 0
 password 7 48728934792
 login auth for-console

line aux 0 
 login auth for-aux

line vty 0 4
 password 7 78970952
 login authentication for-vty
```

## PPP Security

* PAP and CHAP can be for auth, useful for dial
* Default auth methods use local un and pw

AAA for PPP

1. **aaa new-model**
2. Config RAD/TAC servers
3. **aaa authentication ppp default**
4. **aaa authentication ppp** *list-name method1 method2*
5. **ppp authentication** *protocol list-name*, eg ppp authentication chap fred

# Layer 2 Security

* Unused - Not connected to devices
* User - Cabled to end evices, or cable drop to physically unprotected area
* Trusted or trunk - Fully trusted devices in areas of good phy security

Best practices on unused and user
* Disable unneeded dynamic proto (CDP, DTP)
* Access ports only
* BPDU and Root guard
* DAI or PVLANs to prevent frame sniffing
* Port security to limit macs, try to restrict to specific macs
* 802.1x
* DHCP snooping and IP source guard

Other recommendations
* For any port, use PVLANs
* Configure VTP auth globally
* Disabled unused ports and placed into dead VLAN
* Avoid VLAN 1
* Avoid native VLAN on trunks

Best PRactices on Unused/User ports
```
int Fa0/0
 no cdp enable
 switchport mode access
 switchport nonegotiate
 spanning-tree guard root
 spanning-tree bpduguard enable
```

### Port Security

* Can restrict number of macs on ports
* Can enforce certain macs on ports
* switch considers whether allowing, before adding source MAC to port

Supports following: -
* Limiting number of MACs
* Limiting actual macs
 * Static config
 * Dynamic learning up to defined maximum, dynamic entries lost on
   reload
 * Dynamic learning, saved in config (sticky)

* Attackers could claim same MAC as user
* Attacker could fill up mac tables

* Ports be be static trunk or access, not dynamic

* **switchport port-security [maximum-value]** - default 1
* **switchport port-security mac-address** *mac-address [vlan {vlan-id |
  {access | voice }}]* - Defines allowed MACs, for a VLAN (trunk) or for
access/voice
* **switchport port-security mac-address sticky** - Remembers dynamic
  learned MACs
* **switchport port-security [aging] [violation {protect | restrict |
  shutdown } ]** - Defines aging timer, and action taken

* First two required
* If config saved with sticky, they become static
* Protect tells switch to perform port security, restrict sends SNMP
  traps too, shutdown err-disables port, shut/no shut to recover

* **show port-security interface** - SecureUp means port up and secured

## Dynamic ARP Inspection

* MITM ca happen by gratuitous ARP
* GAs occur by sending ARP reply without ARP being sent, b'cast dest
* Can be used to claim IP of host
* DAI defeats this
* Examines ARP message, filters inapproriate messages
* Each port untrusted (default) or trusted
* DAI on untrusted

Message inappropriate if: -

1. ARP reply lists source IP not DHCP assigned to that device and port
2. As above, but with static defined IP/MAC combos
3. For received reply, source MAC in header checked against source MAC
in ARP. Should be equal in normal replies
4. Like Step 3, but dest MAC and target MAC
5. Check for unexpected IPs (0.0.0.0, 255.255.255.255, multicast)

* Requires DHCP snooping, for DHCP snooping binding db
* **ip arp inspection vlan** *vlan-range* - Global
* **[no] ip arp inspection trust**
* **ip arp inspection filter** *arp-acl-name* **vlan** *vlan-range
  [static]* - Refers to ARP ACL, defines static entries
* **ip arp inspection validate {[src-mac] [dst-mac] [ip]}** - Additional
  options
* **ip arp inspection limit {rate PPS [burst interval SECONDS] | none }** - Limits ARP rate to prevent DOS attacks
* Auto limit to 15 ARP messages per second

## DHCP Snooping

* Builds table of IP and port mappings, based on legitimate DHCP
  messages
* Table used by DAI and IP Source Guard
* DHCP clients should be on untrusted ports
* DHCP servers on trusted
* Examines client messages on untrusted ports
* Single device could pose as multiple sending different requests, with different hw addresses

Untrusted ports logic: -
1. Filter all messages sent by DHCP servers
2. Check DHCP release, declines against binding table. If IP in messages not in table, filtered
3. Optionally, compares DHCP client hw add with source MAC in frame

* **ip dhcp snooping vlan** *vlan-range*
* **[no] ip dhcp snooping trust** - Per interface
* **ip dhcp snooping binding** *mac-address* **vlan** *vlan-id ip-address* **interface** *interface-id* - Static entries
* **ip dhcp snooping verify mac-address** - Checks in step 3
* **ip dhcp snooping limit rate** *rate* - Max msgs per second

### IP Source Guard

* Adds to DHCP snooping
* Checks source of IP packets against snooping db
* Can check both source IP and MAC
* **ip verify source** - check source
* **ip verify source port-security** - IP and MAC
* **ip source binding** *mac-address* **vlan** *vlan-id ip-address* **interface** *interface-id* - static entry

```
ip dhcp snooping

int Fa0/1
 switchport access vlan 3
 ip verify source
```

* Verify with **show ip dhcp snooping binding**

## 802.1x Authentication using EAP

* Performs user auth
* Requires UN and PW
* Verified by RADIUS

* 802.1x defines some of detail of LAN user auth
* Also uses Extensible Authentication Protocol (EAP) for auth (RFC 3748)
 * Includes protocol messages to challnge for pw, and OTP flows (RFC 2289)
* First EAP message encap'd in ethernet frame
* Sent by supplicant to authenticator (switch)
* Frames are EAPoL (EAP over LAN)
* Switch translates to RADIUS message
* Switch and supp create OTP with temp key
* Switch forwards auth to auth server
* Switch aware of results, to enable port

Roles
* Supplicant - UN and PW prompt to suer, tx/rx EAPoL messages
* Authenticator - EAPoL/RADIUS translation, enables/disables ports
* Auth server - Stores UN and PWs, verifies RADIUS maessages

* 802.1x is another AAA option

1. **aaa new-model**
2. **radius-server host** and **radius-server key**
3. **aaa authenticaiton dot1x default** - radius, or **aaa authentication dot1x group** *name*
4. **dot1x system-auth-control** - Enables on switch
5. Set port role, **authenticaiton port-contol {auto | force-authorized | force-unauthorized }**

* Auto - 802.1x
* Int automatically authed (default)
* Int automatically unauthed

```
aaa new-model
aaa authenticaiton dot1x default group radius
dot1x system-auth-control

radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
radius-server key cisco

int Fa0/1
 authentication port-control force-authorized

int Fa0/2
 authentication port-control force-authorized

int Fa0/3
 authentication port-control auto

int Fa0/4
 authentication port-control auto

int Fa0/5
 authentication port-control force-unauthorized
```

* Only CDP, STP and EAPoL allowed when unauthed

## Storm Control

* Rate limits l2 traffic
* Can set rising and falling thresholds
 * per traffic type
* Can be per port
* Done on rate or percentage
* If no falling specified, same as rising
* If threshold passed, switch either rate limits by disccarding excess, rate limit and snmp trap, or shutdown

```
int Fa0/0
 storm-control broadcast level pps 100 50
 storm-control multicast level 0.50 0.40
 storm-control unicast level 80
 storm-control action trap
```

* Only supports physical ports
* Can be applied to etherchannels, but no effect

# General L2 Security Recommendations

* Avoid native VLANs
 * Avoids 802.1q headers being sent on access and hopping
 * Ineffective against ciscos anyway, but best practice
* PVLANs give security
* If PVLANs applied, if packet sent to dest MAC on promisc port, but dest IP not, this gets sidetraced (routes packet)
 * Use ACL on LAN int whos source and dest IP in same local connected subnet

# L3 Security

1. Enable SSH instead of telnet
2. SNMP security (v3 support)
3. Turn off all unncessary services
4. Turn on logging
5. Enable routing protocol auth
6. Enable CEF, avoids flow based paths

Other useful ones from RFC 2827 and 3704

1. Packets from own IP range should not come from internet into AS
2. Valid unicast sources - So not 127.0.0.1, b'cast, m'cast etc
3. Directed broadcast should not be allowed
4. RPF checks

# IP ACL Review

* **ip access-group** - Applies to interface
* **access-class** - To line
* **ip access-list resequence**
* **show ip interface** - Shows ACLs on interface
* **show access-list** - ACLs for all protocols
* **show ip access-list** - IP ACls only

## ACL Rule Summary

* ACE - individual entries
* Processed sequentially
* Can use eq, lt and gt for ports
* ne - not equal
* range x-y
* Can match ToS, established
* Log and  log-input (log-input is more info), one message on match, one every 5 minutes after
* Standard ACLs 1-99 1300-1999
* Extended ACLs 100-199 and 2000-2699

## Wildcard Masks

* Find by taking subnet mask and subtracting from 255.255.255.255

# General L3 Security Considerations

## Smurf, Direct Broadcasts, RPF Checks

* Smurf - Lots of ICMP Echo Reqs, atypical IP in packet
* Dest IP is subnet broadcast
* Routers forwarded these on normal routing
* Forwards to LAN as broadcast 
* Source IP of smurf is source IP of host being attacked
* **no ip direct-broadcast** since IOS 12.0
* Enable uRPF with **ip verify unicast source reachable-via { rx | any } [allow-default] [allow-self-ping]** *[list]*
 * CEF must be enabled
 * Strict - rx, incoming int must be same as outgoing int of route
 * Loose - any, checks route in routing table for source
 * Ignores def route
 * Can ping to verify connectivity
 * ACL limits what RPF enabled for
* Fraggles similar, except UDP echo

## Inapproriate IPs

* Should never forward packet with a source not from a registered range
* Bogus IPs (m'cast, b'cast as source, RFC 1918 from internet, bogons)

## TCP Syn Flood, Established Bit, TCP intercept

* Syn flood at servers
* Doesnt complete connections
* Servers fill up resources, load balancers don't balance equally
* Stateful firewalls can prevent
* Can prevent by filtering all pacekts that are first packet in TCP connection
* Established should not originate from client on oneside to server on other
* ACLs can match estab  (all TCP segments except very first)
* Above only works if no inbound traffic expected
* TCP Intercept
 * Watch Mode - Keeps state of TCP connections, if does not complete three way in time period, resets connection
  * If large number in a second (default of 1100 per second), router temporarily filters new TCP requests
 * Intercept - router replies instead of forwarding to server. If handshake completes, router merges connections

```
ip tcp intercept list match-tcp-from-internet
ip tcp intercept mode watch
ip tcp intercept watch-timeout 20

ip access-list extended match-tcp-from-internet
 permit tcp any 1.0.0.0 0.255.255.255
```

# Classic IOS Firewall

* Relies on CBAC (context based access control)
* Dynamically inspects traffic
* Based on protocol commands (eg ftp get)
* When session initiated on trusted network for protocol, which normally blocked by other methods, temporarily opens to permit corresponding inbound from untrusted
* Works on TCP and UDP traffic
* Config inspect protocols, ints, and direction

## TCP versus UDP with CBAC

* TCP easy to handle
* Recognizes FTP control channel commands for example
* UDP connectionless, so approximatees based on if source and dest and ports of UDP frames same as recent, and relative timing
 * Global idle timeout can be config'd

## Protocol support

Inspect on
* Any generic TCP session
* All UDP "sessions"
* FTP
* SMTP
* TFTP
* H.323
* JAva
* CU-SeeMe
* UNIX R
* RealAudio
* Sun RPC
* SQL Net
* Streamworks
* VDOLive

## Caveats

* Comes after ACLs
* Cant protect internal attacks
* Only on protocols config'd
* Named inspections rules on anything not TCP and UDP transported
* Does not inspect local destined or source traffic
* Restrictions on encrypted

## Config steps

1. Choose int (inside or outside)
2. Config ACL to deny all traffic that is to be inspected
3. Global timeouts and thresholds **ip inspect**
4. Inspection rule and timeout **ip inspect name** *protocol* eg **ip inspection actionjackson ftp timeout 3600**
5. Apply to int, **ip inspect actionjackson in**
6. Apply ACL in opposite direction

# IOSS ZBF

* 12.4(6)T onwards
* Ints in security zones
* Can travel between ints in same zone
* Blocked between zones
* Blocked between INTs assigned to a zone and those not
* Must apply policy to allow traffic between zones
* CB langueage used (like MQC)
* Inspect class and policy maps

Can inspect: -
* HTTP and HTTPS
* SMTP, ESMTP, POP3, IMAP
* P2P apps, can use heuristics for tracking port hopping
* IM apps
* RPC calls

Steps: -
1. Decide zones and create
2. Zone pairs - traffic between zones
3. Class maps - identify traffic
4. Polcies - assigned to traffic
5. Apply policies to zone pair
6. Assign ints to zones (interface in one zone only)

Example zone and pair config
```
zone security LAN
 description LAN zone

zone security WAN
 description WAN zone

zone-pair security Internal source LAN destination WAN

zone-pair security External source WAN destination LAN
```

* Auto creates default class, all traffic in it dropped

Class Maps

```
ip access-list extended LAN_Subnet
 permit ip 10.1.1.0 0.0.0.255 any

ip access-list extended Web_Servers
 permit tcp 10.1.1.0 0.0.0.255 host 10.150.2.1
 permit tcp 10.1.1.0 0.0.0.255 host 10.150.2.2

class-map type inspect match-all Corp_Servers
 match access-group name Web_Servers
 match protocol http

class-map type insppect Other_HTTP
 match protocol http
 match access-group name LAN_Subnet
```

* drop
* Inspect - use cbac
* Pass - passes packet
* police
* service-policy - EPI engine
* urlfilter - URL filtering engine

* TCP and UDP timers changed in parameter map

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

zone-pair security Internal source LAN destination WAN
 service-policy type inspect LAN2WAN

int Fa0/1
 zone-member security LAN

int Se1/0/1
 zone-member security WAN
```

* Verify with **show zone-pair security**

# Control-Plane Policing

* Uses MQC
* Rate limits or drops
* WOrks on most routers and MLS

## Preparing for CoPP Implementation

* Make sure rate limit is approriate
* Traffic grouped to form classes

Typical grouping
* Malicious traffic dropped (frag'd packets, malicious programs)
* Routing protocols not limited
* SSH and telnet limited to small amount, but ensures connectivity
* SNMP, FTP and TFTP
* Network apps like HSRP, DHCP, IGMP
* All other IP traffic
* Default class, including L2 protocols. ONly ARP in other class

* Most routers only support policing of inbound traffic

## Implementing CoPP

1. ACLs to classify
2. Class maps
3. Policy maps and associate class maps
4. Assign allowed bandwidth, conform and exceeds
5. Assign service policy to control plane

```
control-plane
 service-policy input CoPP
```

* **show policy-map control-plane**

# DMVPN

* IPSec, GRE and NHRP
* Traffic segmentation across VPNs supported
* VRF-aware
* Spoke connection info not configured
* mGRE and profiles instead
* Spokes point to one or more hubs
* Supports m'cast traffic from hub to spoke

Benefits
* Simpler hub config
* No config changes on hub
* IPsec faciliated by NHRP (auto inititated)
* Dynamic address support
* Dynamic spoke-to-spoke tunnels
* VRF integrations

* Routing Protocol between hubs and spokes
* Cisco recommend DV protocol
* Learn about networks at other spokes
* Next hop IP for spoke is tunnel int
* Clients register with NHRP server
* Spoke queries hubs for outside IP of dest spoke
* Hub replies, spoke initiates dynamic tunnel to other spoke
* Torn down after idle

Config sections
1. Basic IP config
2. mGRE tunnel config
3. Configure IPsec
4. DMVPN routing config

**Basic IP config**
```
R1

int Fa0/0
 ip address 192.168.123.1 255.255.255.0

int lo0
 ip address 1.1.1.1 255.255.255.255

R2

int Fa0/0
 ip address 192.168.123.2 255.255.255.0

int lo0
 ip address 2.2.2.2 255.255.255.255

R3

int Fa0/0
 ip address 192.168.123.3 255.255.255.0

int lo0
 ip address 3.3.3.3 255.255.255.255
```

**GRE MP Tunnel config**

* **ip nhrp map multicast dynamic** - Allows multicast, RIP, EIGRP OSPF
* Each DMVPN needs unique ID, **ip nhrp network-id**
* **ip nhrp authentication** - password

```
R1 (HUB)

int tun0
 ip address 172.16.123.1 255.255.255.0
 tunnel mode gre multipoint
 tunnel mode source Fa0/0
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp authentication cisco

R2 (Spoke)

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
```

* NHS command points to outside IP of hub
* Multicast to hub only (see map)

* **show dmvpn**
 * Ent - Entries in NHRP DB for spoke
 * Peer NBMA address - IP of outside int of spokes
 * Peer Tunnel add - IP of tunnel of spokes
 * State - up or down
 * UpDn T - Time up or down, populated after tunnels in serve

**Config IPsec**

```
crypto isakmp policy 1
 encryption aes
 hash md5
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key 0 TEST address 0.0.0.0

crypto ipsec transform-set MYSET esp-aes esp-md5-hmac

crypto ipsec profile MGRE
 set security-association lifetime seconds 86400
 set transform-set MYSET

int Tun0
 tunnel protection ipsec profile MGRE
```

* **show crypto session**
* **show crypto ipsec sa**

**DMVPN Routing Config**

```
R1 (HUB)

router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
 network 172.16.123.0 0.0.0.255 area 0

int tun 0
 ip ospf network broadcast

R2

router ospf 1
 network 2.2.2.2 0.0.0.0 area 0
 network 172.16.123.0 0.0.0.255 area 0

int Tun0
 ip ospf network broadcast
 ip ospf priority 0
```

* Broadcast means spokes use each others IP as next hop

# IPv6 First Hop Security

## First Hop Security for IPv6

* More end nodes on individual link
* Large neighbour cache on end nodes
* Again on default router

* NDP - Integrates all link operations, for address assignment, router
  discovery, redirect etc
* DHCP - smaller role in address assignment
* Noncentralized address assignment

## Link Operations

* Locations to enforce are end nodes, first hop and last hop

### End Node Security Enforcement

* Does not need central admin
* Each node takes care of itself
* Secure ND in RFC 3971
* Good for threads from link, poor from offlink

### First Hop Switch Enforcement

* Based on centralized model
* Useful only when users go through agg device

### Last Router Security Enforcement

* Attached link and downstream protected
* Needs to be with First Hop Switch
* Last hop discovers all end nodes on segment

## ICMPv6 and Neighbour Discovery Protocol

* ICMP used more in v6
* NDP part of this, non auth'd at base
* Heavy m'cast usage

Apps of NDP
* Router discovery
* Address autoconfig
* v6 address resolution (ARP reaplacement)
* Neighbour reachability
* Duplicate address detection
* Redirection

### Secure Neighbour Discovery (SeND)

* Security for NDP

Provides: -
* Proof of address ownership
 * "Impossible" to take address
 * Used in router discovery, DAD and addr resolution
 * Based on Crypo Generated Address (CGA)
 * Alternatively, non-CGAs with certificates
* Message protection
 * Message integrity protection
 * Replay protection
 * Req/response correlation
 * Used in all NDP messages
* Router auth
 * Auths routers to act as def gw
 * Prefixs router allowed to announce on link

* No end to end security
* No packet confidentiality

* Works with priv/pub keys for each v6 node
* Combined with CGA and RSA
* Cannot use own interface ID, crypto gen'd on v6 prefix and pub key
* CGA ID not sufficient to guarantee address used by right node

* SeND messages signed by RSA pub and priv key pair
* In addition to Neighbour Advertisement with MAC to v6 mapping, also
  CGA parameter and public key, plus priv key sig of all NA fields
* Node 1 pub key verifies CGA address priv key sig of node 2

### Securing at first hop

* Inspect ND traffic
* L2/L3 binding
* Monitor use of ND by host
* Can block RAs and rogue DHCP server advertisements

```
ipv6 access-list ACCESS_PORT
 remark Block all traffic DHCP server - client
 deny udp any eq 547 any eq 546
 remark Block Router Advertisements
 deny icmp any any router-advertisements 
 permit any any

int Gi1/0/1
 switchport
 ipv6 traffic-filter ACCESS_PORT in
```

## RA Guard

* RA snooping exists
* Switch uses upper layer info
* Works against rogue RA attacks
* Must be some intermediary device in network that all traffic will pass
  through
* Inspects RAs, and decides whether to drop or forward

```
ipv6 nd raguard policy POLICY-NAME
 device-role {host | router}

int Fa0/0
 ipv6 nd raguard attach-policy POLICY-NAME
```

* If hosts, RAs dropped

## DHCPv6 Guard

* Blocks replies and advertisement messages from unauthed DHCP servers
  and relay agents
* Client messages always switched
* DHCP server messages only if device role server
* Can process for source validation and server prefernece on
  advertisements
* Also server replies, for permitted prefixes

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

* v6 snooping builds db table of v6 neighbours connected to device
* Created from source like DHCPv6 etc
* Binding table used by v6 FHS features
 * validates link layer addresses
 * validates v6 addresses
 * prefix binding of neighbours to prevent spoofing and redirect attacks
* Binding table auto poppd after snooping enabled

* **show ipv6 neighbors binding**
 * Entires with ND learned from NDP snooping
* v6 snooping integrated with DHCPv6 guard and RA guard

**DHCPv6 Address Integrated with v6 binding database**
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

* After this, DHCPv6 working and binding table popped, DH indicates
  learned by this

RA policy to allow RAs from legitimate router

```
ipv6 nd raguard policy ra_pol
 device-role router
 trusted-port

int Gi1/0/1
 ipv6 nd raguard attach-policy ra_pol
```

## IPv6 Device Tracking

* Provides v6 host tracking
* Neighbour table updates immediately if host drops
* Tracks reachability
* Revokes access when inactive

```
ipv6 neighbor binding vlan 100 interface Gi1/0/1 reachable-lifetime 100
ipv6 neighbor binding max-entries 100
ipv6 neighbor binding logging
```

## IPv6 Neighbor Discovery Inspection

* Learns and secures SLAAC address bindings in L2 tables
* Analyzes ND messages
* Builds trusted binding table
* v6 ND messages not conforming dropped
 * Trusted if v6-to-MAC mapping verifiable

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

* **show ipv6 nd inspection policy**
* Apply to interface with **ipv6 nd inspection attach-policy**
  *policy-name*

## IPv6 Source Guard

* Denies traffic originated from address not in binding table
* No ND or DHCP inspection
* Works in conjunction with ND inspection or v6 address glean (both
  detect existing address on link and store in binding table)
* v6 prefixes in binding table for v6 source guard to work

* Can deny from unknown sources or unallocated addr
* When traffic denied, v6 glean notified to try and recover traffic
 * Queries DHCP server or using ND
 * Data-glean stops deadlock (where valid address fails to be stored in
   binding table and blocking user completely)
* If traffic denied with unknown IP/SMAC, address glean
* Recovers with DHCP_LEASEQUERY to server, DAD NS back
* NA fomr host, DHCP LEASEQUERY_REPLY comes back

```
ipv6 source-guard policy example_policy
 deny global-autoconf  # Denies data traffic from auto config'd global
 permit link-local # Allows from link-local address

int Gi1/0/1
 ipv6 source-guard attach-policy example_policy
```

* Verify with **show ipv6 source-guard policy** *example_policy*

# Port Access Control Lists (PACL)

* Very similar to router ACLs
* Applied to switch ports
* Can be ingress or egress
* Processed first by switch IOS, then VACL

```
int Gi1/0/1
 ip access-group PACLIPList in
 mac access-group PACLMACList in
```
 
