# Messages

## VTP v1 and v2

* Summary Adv - Every 5 mins and after db mode, VTP domain, rev number, identity, time stamp, md5 sum over VLAN DB, VTP password, number of subsets to follo* Subset adv - Originated by db modifier, carries full VLAN db contents - multiple held, may need more
* Adv request - Requests complete db or part, sent after advertisment or summary rx'd with higher rev number
* Join - every 6 seconds, if VTP pruning active, bitfield for each VLAN in nromal range, used and unused

# Timers

# Trivia

* Suspended VLAN drops all frames
* Show current - all vlans when switch in VTP server mode, from db mode
* Promisc VLAN trunk - Anything secondary VLAN changed to primary VLAN and forwarded over trunk
* Isolated VLAN trunk - anything primary vlan changed to secondary and forwarded

## DTP

* Modes are auto, and desirable
 * Auto prefers access
 * Desirable prefers trunk
 * Desirable higher priority
* Carries VTP domain in messages, otherwise can't negotiate
* switchport mode trunk - always trunk, helps other side
* mode access- never trunks, DTP helps other side

## QinQ
* switchport mode dot1q-tunnel
* vlan dot1q tag native
* encapsulation dot1q 10 native
* show int will show admin status and operation of tunnel

## VTP

* Vlan ID, name, type and state
* V1, 2 and v3 (v3 in IOS only, from 12.2(52)SE and up
* V1 no extended range VLANs
* V2 added unknown TLV propagation
 * DB consistency also done at CLI input
* VTP Transparent forwards messages if domain null, otherwise for its domain

### V3

* Primary and secondary servers - secondary can be promoted, but only with password
* Passwords stored encrypted
* Extended range and PVLANs transmitted, only prune normal
* Can be in off mode
* Can set MST region

### Rev numbers etc

* No updates sent until domain config'd
* Default as server
* No domain config on client, uses VTP domian of first rx'd message
* VLAN config in vlan.dat
* New client can change other switches VTP db if
 * new link is trunk
 * same domain
 * higher ev
 * same pw

### V3

* Primary server - only vlan db sent
 * Clients and servers must agree on domain and primary server
 * No sync if dont match
 * one primary server per domain
 * Reload of pri becomes secondary on reboot
 * Reset rev with different VTP domain or PW
 * v3 reverts to v2 if v2 present
 * v1 not compat

# Processes

##VTP

* V1 and v2 update when VLAN added/deleted/update, up rev num
* If v lan DB with higher rev, auto assumed new vlan DB

# Config

```
switchport trunk pruning vlan [add/except/none/remove] <--- Pruning eligible list
````
