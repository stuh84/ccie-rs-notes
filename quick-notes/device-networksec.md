# Messages

# Protocols

## AAA

* Radius - Encrypts passw, UDP, 1812/1645 (Well known/IOS default), RFC2865
* TACACS - Encrypts payload, TCP, 49/49, Proprietary

# Timers

# Trivia

* enable password and enable secret - secret stored by MD5, not affected by service pw encryp
 * enable secret precedence

## AAA

### Default methods

* Single method way to auth user
 * Could be radius, could be local
* List tried in order until accept or reject from method
* Default login applies to all logins (Console/Telnet/Aux)

### Multiple auth methods

* No limit to servers
* Try first method, if does not respond, move to next
* If no response for any, reject

## Layer 2 Security

**Unused**
* Disable CDP and DTP
* Access only
* BPDU and Root guard
* DAI or PVLANs
* Port security
* 802.1x
* DHCP Snooping and IP source guard

```
int Fa0/0
 no cdp enable
 switchport mode access
 switchport nonegotiate
 spanning-tree guard root
 spanning-tree bpduguard enable
```

**Other**
* PVLANs where possible
* VTP auth globally
* Disabled unused ports, put in dead vlan
* Avoid VLAN 1
* Avoid native VLAN on trunks

## Port security
* Dynamic learning, lost on reboot - switch port-security [max-value] - default 1
* Static - switchport port-security mac-address ADDRESS [vlan {id | {access | voice}}]
* Dynamic learning, saved (sticky) - switchport port-security mac-address sticky

## DAI

* GAs can happen, b'cast dest
* Can claim IP of host
* Examines arps, filters inappropriate
* DAI on untrusted
* Inapprorpiate if
 * ARP reply lists source IP not DHCP assigned to device and port
 * As above, static
 * Fo reply, source MAC in header should be source MAC in ARP
 * As above, dest mac and target
 * Unexpected IPs (0.0.0.0, 255.255.255.255, multicast)
* Requires DHCP snooping, for binding DB

## DHCP snooping

* Table of IP and port mappings
* DAI and Source guard use
* Clients on untrusted, servers on trusted
* Examines client messages on untrusted
* Untrusted port logic
 * Filter all DHCP server messages
 * Check DHCP release, declines against binding - if IP not in table, filtered
 * Optionally, DHCP client hw add with source MAC

## IP Soure Guard

* Adds to DHCP snooping
* Checks source of IP packets against db
* Can check both source IP and MAC

## 802.1x Auth using EAP

* Verified by Radius
* Requires UN and PW
* First EAP in eth frame
* Supplicant to authenticator (switch)
* Frames EAPoL (EAP over LAN)
* Switch translates to RADIUS message
* Supplicant - U/N and PW prompt to user, tx/rx EAPoL
* Authenticator - EAPoL/RADIUS trans, enables/disables ports
* Auth server - Stores creds, verifies rad essages

## General L3 Considerations

* Smurfs - Lots of ICMP echos, dest IP is subnet broadcast, avoid with no ip direct-broadcast
* Enable uRPF with `ip verify unicast source reachable-via {rx | any } [allow-default] [allow-self-ping] [list]`
 * rx - strict
* Fraggles similar but UDP echo

* TCP Intercept 
 * Watch mode - keep s state of tcp connections, resets if three way not complete in time period
 * If large number in a second (defaulf 1100 per second), new TCP filtered
 * Intercept - router replies instead of server, merges if handshake completes

## Classic IOS Firewall

* CBAC
* Inspects traffic
* Based on protocol commands
* Temporarily opens ports
* Works on TCp and UDP
* Config inspect protocols
* TCP easy to handle (can recognize control channel)
* UDP connectionless, so based on relative timer
* Comes after ACLs
* Can't protect internal attacks
* Doesn't inspect local destined or source traffic
* Restrictions on ecnrypted traffic
* Use ip inspect (global for times and thresholds, eg ip inspection actionjackson ftp timeout 3000)
* On int for inspection, apply opposite to ACL

## ZBF

* MQC Style
* Ints in securiy zones
* Blocked bw zones
* Ints talk in same zone
* Ints not in zone blocked
* Uses inspect class and policy maps

## CoPP

* MQC, rate limits or drops
* control-plane, service-policy

## IPv6 First Hop Security

### SeND

* Security for NDP
* Used in router discovery, DAD and addr resolution
* Based on CGA (Crypto Generated Address), or non-CGAs with Certs
* Auths router to act as def gw
* Says what prefixes router allowed to announce

### Secure at First hop

* Inspect ND traffic
* L2/L3 binding
* Monitor use of ND by host
* Can blow RAs and rogue DHCP advertisements

### RA Guard

* RA snooping exists
* Switch uses upper layer info
* Must be an intermediary device all traffic passes through
* `ipv6 nd raguard policy NAME... device-role {host | router}`
* ` int Fa0/0... ipv6 nd raguard attac-policy NAME`
* If hosts, RA dropped

### DHCPv6 Guard

* Blocks replies and adv from unauth servers and relays
* Client messages always switched
* DHCP server messagesonly if device role server
* Can do source vlidation and service pref
* Also server replies for permitted prefixes (match reply)
* match server - matches servers allowed

### DHCPv6 Guard and Binding DB

* v6 snooping builds db table of v6 neighborus
* Create from sources lik DHCPv6
* Validates link layer, v6 addr, prefix binding to prevent spoofing
* Auto populated after enabled
* Integrated with DHCPv6 guard and RA guard

### IPv6 Device Tracking

* Host tracking
* Neighbour table updates if host drops
* Revokes access when inactive

### IPv6 Neighbour Didsocvery Inspection

* Learns and secures SLAAC addresses
* analyzes ND
* Builds trusted binding table
* Trusted if v6-to_MAC mapping verifiable

### IPv6 Source Guard

* No ND or DHCP inspection
* Denies traffic if not in binding table
* Works with ND inspection or v6 address glean
* When traffic denied, v6 glean tries to recover (Queries DHCP server or uses ND)
* Recovers with DHCP_LEASEQUERY to server, DAD NS back
* NA From host, DHCP LEASEQUERY_REPLY comes back

# Processes

## Enable SSH

1. hostname
2. ip domain-name
3. Client auth method (local user, AAA etc)
4. Gen RSA keys
5. Specify SSH version
6. Disable telnet on VTY
7. Enable SSH on VTY (transport input ssh)

## ZBF

1. Decide zones
2. Zone pairs - traffic between zones (zone-pair security Internal source LAN destination WAN)
3. Class maps, identify traffic
4. Policies
5. Apply policy to zone pair (zone-pair .... service-policy type inspect LAN2WAN - match a policy map with type inspect)
6. Assign ints to zone

# Config

* `service password-encryption` - Weak, not changed in startup until copy run start or wr mem
 * not auto decrypted

## AAA

**Default methods**

```
enable secret 5 HASH
username cisco password 0 cisco
aaa new-model
aaa authentication enable default group radius local
aaa authentication login default group radius none
radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
```

**Multiple auth methods**

* group radius - config'd radius
* group tacacs+ - as above but tacacs
* aaa group server ldap - as above, LDAP
* group name - User defined
* enable - Use enable pw
* line - Config in line
* local - Usernames in local config, U/N any case
* local-case - Username in local, case sensitive user
* none - auth always

```
aaa group server radius fred
 server 10.1.1.3 auth-port 1645 acc-port 1646
 server 10.1.1.4 auth-port 1645 acc-port 1646

aaa new-model
aaa authentication enable default group fred local
```

**Override defaults**
* Can use for console, VTY and Aux

```
aaa authentication login for-console group radius line
aaa authentication login for-vty group radius local

line con 0 
 password 7 84578293472
 login auth for-console

line vty 0 4
 login authentication for-vty
 password 7 787978
```

**PPP**
* PAP and CHAP can be used
* Default auth use local username and pw
* Use AAA with

```
aaa new-model

aaa group server ....
 ... 
 ...

aaa authentication ppp default
aaa authentication ppp NAME method1 method2
int dialer 0
 ppp authentication chap fred
```

## DAI

```
ip arp inspection vlan RANGE <--- global
[no] ip arp inspection trust
ip arp inspection filter ARP-ACL vlan RANGE [static] - defines static entries
ip arp inspection validate {[src-mac] [dst-mac] [ip]}
ip arp inspection limit { rate PPS [burst interval SEC] | none} - Default 15 arp messages per second
```

## DHCP Snooping

```
ip dhcp snooping vlan RANGE
[no] ip dhcp snoopint trust
ip snooping binding MAC vlan VLAN IP interface INT <--- static
ip dhcp snooping verify mac-address
ip dhcp snooping limit rate RATE
```

## IP Source Guard

```
ip verify source
ip verify source port-security - IP and mac
ip source binding MAC vlan VLAN IP interface INT <---static
```

## 802.1x

```
aaa new-model
radius-server host ....
radius-server key ...
aaa authentication dot1x default - radius
aaa authentication dot1x group NAME
dot1x system-auth-control

int Fa0/0
 authentication port control {auto | force-authorized | force-unauthorized }
```

## Storm control

```
int Fa0/0
 storm-control broadcast level pps 100 50 <--- Rising and falling thresh
 storm-control multicast level 0.50 0.40 (percent)
 storm-control unicast level 80 - same rising and falling thresh
 storm control trap <-- can be rate limit, trap or shutdown
```

* Doesn't work on etherchannel

# Verification

```
show port-security interfaces <-- SecureUP means port up and secure

show ip dhcp snooping binding

show zone-pair security

show policy-map control-plane

show dmvpn
 Ent - Entires in NHRP DB for spoke
 Peer NBMA Address - outside IP
 Peer tunnel add - IP of tunnel
 State - up or down
 UpDn T - time up or down

show ipv6 dhcp guard policy NAME

show ipv6 neighbors binding

show ipv6 nd inspection policy
```
