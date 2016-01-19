# Virtual Lans

Ports grouped into one broadcast domain. Should to 1-to-1 VLAN per subnet

# VLAN Config

VLANs can be active or suspended. Access is suspended VLAN drop all frames

## VLAN Database Mode

Obsolete mode, still exists

Vlans 1 to 1005, in vlan.dat in flash

```
vlan database
 vlan 21
```

show current - All VLANs when switch in VTP server mode

show proposed

apply  
abort - aborts changes  
reset - as above, but stays in VLAN DB config mode  

## INts in VLANs

show vlan brief - show ports in VLANs (access)

## Config mode for creating VLANs

Extended range and PVLANs

switchport access vlan 31 - creates VLAN 31  
vlan 32

## VLAN operational status

state suspend - Both config modes, suspends through VTP domain  
shutdown - local  

## PVLANs

* Primary VLANs have associated Secondary VLANs
* Community or isolated.
* 1 Isolated per primary
* Promiscious associated with primary VLAN, all secondary of primary can talk to promisc
* Frame received on promisc, comm or isol always forwarded through trunk

Over trunks, frame on isol/comm goes via secondary VLAN, forwarded based upon type. For Promisc, sent as primary VLAN

Trunk types of Promisc and Isolated

* Promisc - When frame from secondary VLAN going out trunk, VLAN tag rewritten to primary VLAN 
* Isolated - When primary VLAN received, sent as secondary. Relies on switch isolating ports itself

### Config

```
vlan 199
 private-vlan isolated

vlan 101
 private-vlan community

vlan 100
 private-vlan primary
 private-vlan association 101,199

int Fa0/1
 switchport mode private-vlan host
 switchport private-vlan host association 100,101

int Fa0/13
 switchport mode private-vlan promisc
 switchport private-vlan mapping 100,101,199

int vlan 100
 private-vlan mapping 101,199
 ip address 10.1.1.1 255.255.255.0
```

# VLAN Trunking

**ISL**

* Cisco proprietary
* Encapsulates
* Normal and extended range
* No native
* 26-byte header and trailer with FCS
* Source MAC of trunk used
* Multicast dest of either 0100.0C00.0000 or 0300.0C00.0000
* Technically SNAP-encap frame

**802.1q**

* IEEE defined
* Tags frame
* Extended range
* Native VLAN
* Inserts 4 byte header (after source address field)
* Original frames address intact
* First 2 bytes are Ethernet Type of 0x8100

Native VLAN mismatches detected/blocked by PVST+ and RPVST+. CDP detects and reports

## Config

Cisco use DTP. Modes are auto (negotiates but prefers access) and desirable (prefers trunk). Desirable higher priority (i.e. trunk will form if config'd on one end)

2950s and 3550s default desirable, 2960s, 3560s and 2750s default to auto

DTP carries VTP domain in messages. Must match. Means different domains don't negotiate.

```
switchport
switchport mode - DTP parameters
switchport trunk - trunk parameters
switchport access - Nontrunk parameters
```

show int trunk  
show int *int* trunk  
show int *int* switchport  
show dtp  

## Allowed, Active and Pruned VLANs

* switchport trunk allowed
* VTP can prune
* show int trunk shows allowed, allowed and active (PVST+ STP instance will be running on this trunk for VLANs in list), active and not pruned (no VTP/PVST+ blocked/pruned VLANs)

## Trunking Config

* switchport mode
* switchport nonegotiate
* switchport mode trunk - Always trunks this side, DTP helps other side
* switchport mode dynamic desirable - hopes to trunk
* switchport mode dynamic auto - hopes for access
* switchport mode access - never trunks, DTP helps other side
* switchport trunk encapsulation - sets trunking type

## Trunking on routers

Sub-IFs, do not need to match VLAN ID. Set it with encapsulation command. Recommend to use **encapsulation dot1q vlan-id native**, allows both untagged and CoS marked frames tagged with VLAN ID

# Q-in-Q Tunneling

802.1ad (provider bridges)  
802.1ah (provider backbone bridges)

*vlan dot1g tag native*

```
int Fa0/1
 switchport mode dot1q-tunnel
 switchport access vlan 5
 l2protocol-tunnel cdp
 l2protocol-tunnel lldp
 l2protocol-tunnel stp
 l2protocol-tunnel vtp
```

show int fa0/1 then shows admin and operational mode of tunnel

# VLAN Trunking Protocol

* Advertises VLAN ID, name, VLAN type and state
* Three versions, v1 and v2 on IOS and CatOS, v3 on IOS from 12.2(52)SE
* v1 no extended range VLANs

V2 additions

* TrCRF and TrBRF type VLANs (token ring)
* Unknown TLV propagation
* DB consistency check done at CLI input, rather than on SNMP or VTP receive


VTP transparent allows forwarding messages if domain is null, otherwise for its domain

v3 additions

* Primary and secondary servers - Only primary can update, secondary can be promoted to primary
* Passwords stored encrypted, can be transmitted to another switch, promotion requires using it
* Extended range and PVLANs, pruning only on normal range
* Off mode - drops all VTP messages (per trunk or global)
* Cab distribute MST region config

v1 and v2 use four message types

* Summary advertisement - From all switches, every 5 mins and after db mod, VTP domain name, revision number, identity of updater, time stamp of update, md5 sum over VLAN DB contents and VTP password, and number of subsets to follow
* Subset advertisement - Originated by db modifier, carries full VLAN db contents. One advertisement holds multiple VLAN db entries, may need multiple subset messages
* Advertisement request - To request complete vlan db or part, sent after advertisement, or whe receiving summary with higher rev number
* Join - Sent every 6 seconds if VTP pruning active, contains bitfield for each VLAN in normal range, and used/unused

VTP messages only on trunks

## Process and Rev Numbers

* v1 and v2, update when VLAN added/deleted/updated. Rev number up by 1, entire VLAN db and rev number sent out
* If VLAN db with higher rev number received, automatically assumed to be new VLAN db
* Default of VTP server
* No updates sent until domain config'd
* No domain config on client, uses VTP domain of first rx'd message, mode config required
* VLAN config stored in vlan.dat (not NVRAM)
* Config db can be flash by trunks failing and changes

Newly connected client can change another switches VTP db if: -

* New link is trunk
* Same domain name
* Higher rev number
* Same password

VLAN db hash with current db and pw, compared to MD5 in Summary and at least one subset

v3
* Primary server - only vlan DB that can be sent through domain
* Clients and servers must agree on domain and primary server (base MAC defined)
* Secondary servers - no changes, can be promoted (**vtp primary**)
* No sync if don't match
* One primary server per domain. 
* Reload of primary makes it secondary on reboot
* Can't reset revision with rev 0 and VTP transparent
* Can reset with different VTP domain or pw
* v3 reverts to v2 operation if v2 present
* not compatible with v1

## Config

```
vtp domain DOMAIN

show vtp status

vtp password

vtp mode server/transparent/client/off

vtp version <---- 1 and 2 applies to all switches in domain, v3 done per switch (must have domain set too)
```

## Normal and Extended Range VLANs

1-1005 supported in v1 and v2, in config or db mode

Extended dont go in vlan.dat, only work in transparent mode. In v3, all info stored in vlan.dat

ISL used 10 bits for VLAN ID, so couldn't do extended range. Later update to match 802.1q (12 bits)

1002-1005 - FDDI and TR

## Storing VLAN config

* vlan.dat or running config (based upon vtp ver and extended vlans)
* In v3, all in vlan.dat. If transparent or off, also in running config

# Configuring PPPoE

* Virtualizes ethernet, turns into multiple p2p between client and AC
* Per-user auth
* Negotiation of upper protocols (eg IPCP)
* Neg of compression and link bundling
* IOS Router can be client rather than setting on a host
* 8 byte overhead for PPPoE (2 byte PPP, 6 byte PPPoE)
* MTU 1492
* MSS clamped to 1452 to fit IP an TCP headers plus 8 byte PPPoE header

```
int Fa0/0
 ip address 192.168.100.1 255.255.255.0
 ip nat inside

int Fa0/1
 pppoe-client dial-pool-number 1

int dialer1
 mtu 1492
 ip tcp adjust-mss 1452
 encap ppp
 ip address negotiated
 ppp chap hostname Username@ISP 
 ppp chap password Password4ISP
 ip nat outside
 dialer pool 1

ip nat inside source list 1 interface dialer 1 overload

access-list 1 permit 192.168.100.0 0.0.0.255
ip route 0.0.0.0 0.0.0.0 dialer1
```

**show pppoe session**
**debug pppoe data/errors/events/packets**


