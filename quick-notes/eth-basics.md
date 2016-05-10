# Messages

* MAC addressing
 * I/G least sign bit - 0 for unicast, 1 for multicast
 * U/L next least sig - 0 vendor assigned, 1 admin assigned

# Trivia

## Type

* DSAP and SNAP

## SPAN, RSPAN, ERSPAN

* SPAN - local
* RSPAN - Same source, dest vlan, ar RSPAN Dest RSPAN VLAN is source
* ERSPAN - Encap'd RSPAN, avail in IOS-XE, Cat 6500, Nexus, can monitor Po, FE and GigE

### Restrictions

* Orig config overwritten on dest port
* Dest ports removed from EC
* Dest dont support 802.1x or PVLANs
* Dest dont support CDP, STP, VTP
* Source can be one or more ports or a vlan, not mix
* 64 span dests max on switch
* Single SPAN cannot deliver traffic to dest when source is mix of SPAN, RSPAN and ERSPAN
* SPAN dest cant be source
* Only one dest session per source
* Span dest passes only span traffic
* Trunks can be source
* Traffic routed from VLAN to source VLAN cannot be monitored (internal traffic)
* RX Traffic transported with mod
* TX after ACL, QoS etc, can exempt some

## VSS

* One management plane, multiple devices
* SSO - Stateful Switchover
* NSF
* One chassis active, one standby virtual switch
* Active sup manages management, (SNMP, SSH), L2 protos, l3 protocols, software data path
* Active programs PFC on standby
* show switch virtual - shoes mode, domain, switch number, and peer switch number and roles
* show switch virtual redundancy - Switch IDs, last switchvoer, mode, sw state, uptime, IOS version, conf reg
* MAC for L3 from active
* Stays same after switchover, only changes on full stack reboot
* Can assign virtual macs
 * mac-address use-virtual

### Virtual Switch Link

* EC
* LB'd using EC alg
* Frames encap'd with Virtual Switch Header, added by egress ASIC, stripped by ingress ASIC
* Carries ingress port index, dest port index, VLAN, CoS
* 32 bytes long
* Placed after Ethernet preamable, before L2 header
* VSL must be up before bringing up VSS

### Initilization

* Sup says which ports part of VSL
* Config preparsed for VSL commands and INTs
* Link management Protocol  on each VSL link - part of VSLP, rejects unid links, exchange switch IDs

### VSS Role Resolution

* Role Resolution Protocol
* Determines if hw/sw versions allow vss to form
* Sees whats active/standby
* If hw/sw/config check fails, revert to RPR (route-processor redundancy), all modules powered down
 * not NSF/SSO mode
* If ports shut down on one side and not other, enough to fail check

### System PFC Operating mode

* v3 and 4 - support for forwarding performance level and feature set
* XL/non-xl - size and capacity of different hw resources (L2/L3 tables, acl entries etc)
* Can preneg XL/non-xl

### VSL Redundancy

* Must use specific types of 10g/40g ports
* Port must be capable of adding VSH

### Initialiation process

|Number|Chassis 1|Direction|Chassis 2|
|------|---------|---------|---------|
|1|Initilization|N/A|Initilization|
|2|Pre-Parse config|N/A|Pre-Parse Config|
|3|Bring up VSL linecards and ports|N/A|Bring up VSL linecards and ports|
|4|Run VSLP|1 to 2, then 2 to 1|Run VSLP|
|5|Run RRP|1 to 2, then 2 to 1|Run RRP|
|6|Interchassis SSO|1 to 2, then 2 to 1|Interchassis SSO|
|7|Continue bootup|1 to 2, then 2 to 1|Continue bootup|

* Quad-Sup uplink forwarding - Redundant sup and reundant link per chassis
 * Cross connects

* Adaptive load balancing - combats resetting hash value when adding nw ports
 * port-channel has-distribution adaptive

* Port channels same source index, regardless of chassis
 * B'casts and unknown unicasts not sent back on VSL if rx'd on one
 * Control traffic to AVS, redirect through VSL from SVS

### Virtual Switch Mode

* Requires conversion
* Chassis reloads, operates in virtual mode
* Switch ID
 * Unique per chassis
 * Part of int naming
 * switch set switch_num 2
* If misconfig, VSL formation on initlization fails
* Console disabled on standby
* Complete once reached SSO Stanbdy Hot Redundancy Mode

* Reload members with redundancy force-switchover
 * redundancy reload shelf 1
 * redundancy reload peer

### SSO

* show virtual reundancy

### RPR-WARM

* Only in VSS
* Sync allows in chassis standby sup to reload and take over from active
* Not stateful - linecards reload during sup reload

### In chassis standby boot process

* Once it knows to be standby, determines if part of VSS
* If so, boots to RPR-WARM
 * Loads new IOS image
 * Sup LC
 * Once loaded, runs as DFC-enabled line card

* First switch up is active
* If both at same time, lowest switch ID
* Set priority with switch NUMBER priority NUMBER, highest preferred
* With preemption, SSO performed if necessary
* switch 2 preempt TIMER (time after VSL comes up)
* Heartbeats across VSL detect sup failure

## Stackwise

* up to 9 in stack
* Stackwise plus or sstackwise ports
 * plus on 3750x or e, 3750 non plus
* Homogenous - all one type
* Mixed stack - any model, or same model diff features, or both
* Switch stack ID'd by Bridge ID, and if L3, router MAC (determined by master)
* Stack member number
* Stack number priority - highest becomes new master on failure
* Master has start and running configs
 * Each member has copy for backup
* If switch replaced with identical model, gets same config
* Membership change causes no interruptions, unless stack master gone, or adding powered on switch
 * Powered on switches causes masters to elect n/w each aother, and other memebrs to reboot

### Election

1. Current stack master
2. Highest priority
3. Not using default int-level config
4. Highest priority feature and image combination
 * IP services and crypto
 * IP services no crypto
 * IP Base and crypto
 * IP Base no crypto
5. Lowest MAC

* No preemption
* If stack master changes, BID and router MAC could change
 * Enable persistent mac

* show switch - shows switch members
 * default is 1
 * Takes lowest number when joining stack
* `switch CURRENTNUM renumber NEW`
 * Resets config if none associated
 * Cannot use on provisioned switch
* `switch NUM priority VALUE`
* `switch NUM provision TYPE`
 * Enables ability to config for switch not up yet
 * if type doesn't match, default config applied
 * If numbe conflicts, renummered and applies provisioned config
* SDM Mismatch
 * Resolved after version mismatch
* All members must run IOS image and feature set to ensure compat
* show platfcorm stack-manager all
* Same IOS means same version
* Same major versions but different minors particually comptaible
 * Will try to upgrade
 * Can advise if no image found which to use
* Default stack mac addr timer disabled, member number 1, priority 1, offline config (not provisioned)


## IOS-XE

* Kernel based open system
* Cross platform
* IOS as daemon
* Each process balances among CPUs and cores
* IOSd supports multiple threads
* Can control apps
* Common Management Enabling Technology, CLI, XML, SNMP, HTTP
* Uses off the shelf drivers and Cisco-specific drivers
* Wirehsark works on Cat4500 for example
* All routing protocols in IOSd
* 64 bit, memory above 4gb
* Kernal Allocates virtual memory to IOSd, IOSd allocates memory to functions
* Memory protection exists, prevents corruption between processes
* FFM - Forwarding And Feature Manager
* FED - Forwarding Engine Driver
* FFM - APIs for Control Plane processes, programmes FED, maintains forwarding states
* FED - Drivers affect data planes

# Timers

# Processes

# Config

**SPAN**
```
monitor session 1 source int <SOURCE-IF>
monitor session 1 dest int <DESTINATION-IF>
```

**Complex SPAN Config**
```
monitor session 1 source int Fa0/18 rx
monitor session1 filter vlan 1-3, 229 <--- do not monitor
monitor session 1 dest interface Fa0/24 encap replicate
```

**RSPAN**
Source

```
vlan 199
 remote span

monitor session 3 source vlan 66-68 rx
monitor session 3 destination remote vlan 199
```
Destination
```
vlan 199
 remote span

monitor session 63 source remote vlan 199
monitor sesison 63 destination interface Fa0/24
```

* Range of VLANs for session IDs must be 1-66

**ERSPAN**

Source
```
monitor session 1 type erspan-source
 source interface Gi0/1/0 rx
 no shutdown
 destination
  erspan-id 101
  ip address 10.1.1.1
  origin ip address 172.16.1.1
```

Destination
```
monitor session 2 type erspan-destination
 destination interface Gi2/2/1
 no shutdown
 source
  erspan-id 101
  ip address 10.1.1.1
```

## Conversion to Virtual Switch Mode

```
switch virtual domain 100 
 switch 1

int port-channel 1
 switch virtual link 1

int po2
 switch virtual link 2 (link number same as switch ID)

int Ten5/4-5 
 channel-group 1 mode on 
 no shut

switch conver mode virtual
```
