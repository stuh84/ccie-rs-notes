# GRE Tunnels

* Passenger packets encap'd inside another protocol
* IPSec VTI - routable interface, terminates IPsec tunnel
 * Support multicast

**GRE Tunnel Config**

```
int lo0
 ip address 150.1.2.2 255.255.255.0

int tun0
 ip address 192.168.201.2 255.255.255.0
 tunnel source lo0
 tunnel destination 150.1.3.3
```

# DMVPN Tunnels

## Operation

Benefits: -
* Dynamic tunnels
* Reduced config
* Zero touch hub config
* Dynamic IPsec
* Dynamic spoke-to-spoke tunnels
* Supports multicast
* VRF aware
* Routing protocols over individual or all tunnels
* Supports MPLS
* Load balancing
* Reroutes during outages

### Components

* GRE
* NHRP
* Dynamic routing
* IPSec

### Operation

* Each spoke persisent IPsec tun to hub
 * No spoke to spoke with this
* Address of hub must be known by all spokes
* Spoke registers address as client to NHS
* NHS maintains DB of public addr used by spokes
* Spoke to spoke - request to NHS for pub ip of other spoke
 * On demand

**Phase 1**
* Simple hub and spoke topology
* Dynamic IPs on spokes

```
crypto isakmp policy 1
 encryption aes
 authentication pre-share
 group 2
 crypto isakmp key cisco 123 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TSET esp-3des esp-sha-hmac
 mode transport

crypto ipsec profile DMVPN
 set transform-set ISET

int Tun0
 ip address 172.16.145.1 255.255.255.0
 ip mtu 1400
 ip nhrp authentication cisco123
 ip nhrp map multicast dynamic
 ip nhrp network-id 12344
 no ip split-horizon eigrp 145 # Routes between spokes
 tunnel source Fa0/0
 tunnel mode gre multipoint
 tunnel key 12345
 tunnel protection ipsec profile DMVPN
```

**Phase 2**

* Adds spoke-to-spoke comms
* Allow with **no ip next-hop-self eigrp** *as*
* Verify with **show crypto ipsec sa** and **show dmvpn**

**Phase 3**

* Fixes some Ph2 issues
* Ph2 allows daisy-chaining, OSPF single area and limited hubs due to OSPF DR/BDR election
* Ph2 does not allow route summarization at hub, all prefixes to all spokes to set up spoke-to-spoke
* Ph2 has first packet through hub process switched
* Ph3 added two NHRP enhancements
 * NHRP Redirect - New message, hub to spoke, lets spoke know a better path to other spoke than throug hub
 * NHRP Shortcut - Overwrites CEF info on spoke
* Make sure using **no ip split-horizon eigrp** *asn* on for spoke to spoke
* No other differences in config
* NO need for **no ip next-hop-self eigrp** command

# IPv6 Tunneling and related technologies

## Tunneling overview

|Tunnel Mode|Topology and Address Space|Applications|
|-----------|--------------------------|------------|
|Auto 6-to-4|Point-to-Multipoint; 2002::/16|Connecting isolated v6 island networks|
|Manually config'd|P2P, any addr space, dual stack both ends required|Carries v6 packets across v4|
|v6 over v4 GRE|P2P, unicast addr, dual stack both ends|Carries v6, CLNS and other traffic|
|ISATAP|P2MP, any mcast address|v6 hosts in a single site|
|Automatic v4-compatible|P2MP, ::96 address space, dual stack both ends|Deprecated, use ISATAP instead|

Config wise

|Type|Tunnel Mode|Destination|
|----|-----------|-----------|
|Manual|ipv6ip|v4 address|
|GRE over IPv4|gre ip|v4 address|
|Auto 6to4|ipv6ip 6to4|Auto determined|
|ISATAP|ipv6ip isatap|Auto determined|
|Automatic v4 compatible|ipv6ip auto-tunnel|Auto determined|

### Manual config'd tunnels

* P2P
* Static destination
* Almost v4 GRE config

```
int tun0
 no ip address
 ipv6 address 2001:DB8::1:1/64
 tunnel source lo0
 tunnel destination 172.30.20.1
 tunnel mode ipv6ip
```

### Auto IPv4-Compatible Tunnels

* v4-compat v6 addr for tunnel int
* Taken from ::/96 space
* First 96 bits tunnel int all 0s
* Remaining 32 from v4 addr
* Tunnel dest determined from low-order 32 bit of tunnel int 
* **tunnel mode ipv6ip auto-tunnel**
* Not widely deployed
* Doesn't conformed to global usage of v6 space
* Also doesn't scale well

### IPv6-over-IPv4 GRE

* Encap nonv6 traffic
* Supports IPsec
* P2P operation
* Over v4
* Only diff from manual config is **tunnel mode gre ipv6**

### Auto 6to4 Tunnels

* Treat v4 as nbma cloud
* Per-packet basis, encaps traffic to correct dest
* Combines v6 prefix with global unique dest 6to4  border routers v4 address
 * Begins with 2002:border-router-IPV4-address::/48
* Prefix leaves another 16 bits in 64 bit for numbering networks within site
* Only one tunnel allowed per IOS router
* Mode of **tunnel mode ipv6ip 6to4**
* Tunnel dest not explcitly config'd
* need to route 2002::/16 over tunnel (or at least other sides prefix)

```
int Fa0/0
 ipv6 address 2002:0a01:6401:1::1/64

int Fa0/1
 ipv6 address 2002:0a01:6401:2::2/64

int E2/0
 ip address 10.1.100.1 255.255.255.0

int tun0
 no ip address
 ipv6 address 2001:0a01:6401::1/64
 tunnel source Eth2/0
 tunnel mode ipv6ip 6to4

ipv6 route 2002::/16 tunnel 0
```

### ISATAP Tunnels

* Intra-Site Automatic Tunnel Addressing Protocol
* v4 as NBMA
* Addressing style differs from 6to4

```
[64-bit link-local or gloabl unicast prefix]:0000:5EFE:[IPv4 address of ISATAP link]
```

* ISATAP int id middle part

Example
* v6 prefix 2001:0DB8:0ABC:0DEF::/64
* v4 tunnel dest 172.20.20.1 (hex AC14:1401)
* 2001:0DB8:0ABC:0DEF:0000:5EFE:AC14:1401
* **tunnel mode ipv6ip isatap**
* v6 address derived from EUI-64
* Last 32 bits of int ID from tunnel source v4
* Tunnels disable RAs by default
 * need enabling to support client autoconf
 * **no ipv6 nd suppress-ra**

### SLAAC and DHCPv6

* SLAAC - router and RAs in some scenarios, DHCPv6 and RAs in others

### NAT-PT

* Not technically tunnelling
* Gateway function, v4 to v6 and vice versa
* Supports static and dynamic translations, and port translations

### NAT ALG

* Allows comms at app level between two disaprate networks (one v4, one v6)

### NAT64

* Transparent services to v6 users by providers
* Seamless v6 migrations

* NAT64 and DNS64 allows v6 client to iitate comms to v4 only server
* Stateful NAT64, translates using IP header trans, algoritihms in RFC 6145 and RFC 6052
* Need nat64 prefix, nat64 router and dns64 server

# L2 VPNs

* PWs for frames over MPLS cloud
* Service has field name, emulated service
* Tagged mode or raw mode

## Tagged mode

* ID matches on ACs at either end
* ID/VLAN match on each end
* pw type of 0x0004
* Every frame on PW different VLAN for each customer (Service-delimiting tag)
* If frame rx'd missing VLAN TAG< PE prepends a dummy one before forwarding on PW

## Raw Mode

* Tag not always present
* PW type 0x0005
* Service delimiting tags never through AC
* Stripepd from frame before transmitting

## L2TPv3

* Enhancements to L2TP
* Tunnels any L2 payload
* IP Proto 115
* Cef must be enabled
* Loopback int must have valid IP reachable from remote PE
* L2TPv3 has control channel

## AToM

Config
```
R2

int Fa0/0
 xconnect 4.4.4.4 204 encapsulation mpls

R4

int Fa0/0 
 xconnect 2.2.2.2 204 encapsulation mpls
```

* Encap can be L2TPv2, v3 or MPLS
* **show xconnect all**
* Segments, S1 customer facing port, S2 is core config

## VPLS

* VPLS over GRE enables VPLS across IP network

## OTV

* Looks like VPLS with MPLS transport
* Multicast support similar to whats in L3 VPNs
* Deployed at CE
* L2 LAN-E over L3, L2 or MPLS networks
* Supported in IOS XE 3.5 or later, NX-OS 6.2(2) or later
* Fault domain isolation advantage
 * STP root doesnt change
 * Each CE own root
 * Auto detection of multihoming and ARP optimization

# GET VPN

* Group Encrypted Transport VPN
* Encrypts through unsecure networks
* IPsec for integrity/confidentiality
* Key Server router, and Group Members (KS, GM)
* KS creates, maintains and sends policy to GM
 * Policy says what traffic, and what algorithms

* Two kinds of keys gen'd
 * Transport Encryption Key - By GM to encryp data
 * Key Encryption Key - Encrypts info between KS and GM

* No IPsec tunnels between GMs
* GM just encrypts packets that conforms to policy
* Uses Encapulsating Security Payload
* Original IPs to route

* RSA key used by KS for Rekey
* KS sends out new TEK and KEK before TEK expires (default 3600s)
* Does in Rekey phase
* Phase auth'd and secure by an ISAKMP SA between KS and GM
* GDOI (Group Domain of Interpretation) messages (mutation of IKE) build SA and encrypt GM registration
 * UDP port 848

```
ip domain-name cisco.com
crypto key gen rsa mod 1024

crypto isakmp policy 10
 auth pre-share

crypto isakmp key GET-VPN-R5 address 10.1.25.5
crypto isakmp key GET-VPN-R4 address 10.1.24.4

crypto ipsec transofmr-set TSET esp-aes esp-sha-hmac

crypto ipsec profile GETVPN-PROF
 set transform-set TSET

crypto gdoi group GETVPN
 identity number 1
 server local
```

* Rekey in two ways
 * Unicast - Rekey to every GM known (where m'cast not on network)
 * Multicast - One packet, to all GMs at once

```
rekey auth mypubkey rsa R1.cisco.com
rekey transmit 10 number 2
rekey transport unicast

authorization address ipv4 GM-LIST
```

* GM List is ACL to limit GMs
* LAN-LIST ACL says what to encrypt
* Can specify window size (for time based antireplay detection)
* Need KSs IP, sent down to GMs, as KS can be on different IPs (eg loopback)

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

**Configure Members**
```
crypto isakmp policy 10
 auth pre-share

crypto isakmp key GET-VPN-R5 10.1.12.1

crypto gdoi group GETVPN
 identity number 1
 server address ipv4 10.1.12.1

ip access-list extended DO-NOT-ENCRYPT
 deny tcp 192.168.4.0 0.0.0.255 eq 22 192.168.5.0 0.0.0.255

crypto map CMAP-GETVPN 10 gdoi
 set group GETVPN
 match address DO-NOT-ENCRYPT

int Se0/1/0.52
 crypto map CMAP-GETVPN
```

* **show crypto gdoi group** *group-name*
* **show crypto gdoi ks policy**
* **show crypto gdoi ks acl**
* **show crypto gdoi ks members**
