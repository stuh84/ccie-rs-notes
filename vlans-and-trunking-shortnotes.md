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

*ISL*
* Cisco proprietary
* Encapsulates
* Normal and extended range
* No native
* 26-byte header and trailer with FCS
* Source MAC of trunk used
* Multicast dest of either 0100.0C00.0000 or 0300.0C00.0000
* Technically SNAP-encap frame

*802.1q*
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
show int <int> trunk
show int <int> switchport
show dtp

