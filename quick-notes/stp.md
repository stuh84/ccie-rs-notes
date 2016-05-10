# Messages

* Protocl Ver in BPDU for STP of 0x00
* BID - 2 byte priority then MAC

# Timers

* BPDU lasts for MaxAge-MessageAge seconds
* MaxAge in STP default 20s

# Trivia

* IDs of bridge and ports, all configurable priority
* Config BPDUs compared, superior based on
 * RBID
 * RPC
 * Sender BID
 * Sender PID
 * Receiver PID (not in BPDU, local)
* Only config BPDUs compared
* One RP on non-root, one DP per seg
* Superior BPDUs stored that are sent/rx'd
* DP Stores own, root and blocking store upstream
* All claim root until superior BPDU
* For BID in PVST+ and MST, priority 4 bits (4096), 12 bits for SysExId (usually vlan id), MAC Address Reduction
 * Seen with `spanning-tree extend system-id`

## Convergence on new topology

* TC when TCN BPDU on DP, port moves to forwarding and switch has at least one DP, port goes blocking, or switch becomes root
* Switches age out unused CAM, forward delay (15s) to time out CAM

## PVST and STP over trunks

* STP per VLAN, differenr roots
* With 802.1Q and non cisco, only CST
* PVST+ runs on trunks as VLAN 1 STP instannce
 * CST regions bind for all vlans
* CST as loop free shared seg
* PVST+ BPDUs encaped in m'cast dest of 0100.0CCC.CCCD (tagged with correct VLAN)
* SNAP ENCAP (ordinary BPDUs LLC)
* TLV at end with VLAN number, checks for VLAN mismatches
* VLAN1 on PVST+ uses standard BPDUs and PVST+ BPDU, latter only for mismatches
* If access port gets BPDU with rong VLAN, type inconsistent
* On trunks, If tagged, BPDU in that VLAN, if no tag, native
* If PVID TLV match, processed, otherwise PVIDInconsistent, PVID = port vlan (not necessarily native)

## RSTP

* Discarding
* Learning
* Forwarding
* Alternate - Root backup
 * If RP lost, AP with best BPDU promoted
* Backup - DP backup
 * Takes over if DP fails, not rapid
 * Three BPDUs lost on DP, one remains best, rest back to Backup Discarding
* Edge or Non-Edge

### BPDU Format And Handling
* Proposal/Agreement bit, and also port states
* RSTP ages out BPDUs after 3 hellos, message age only a hop count
* Inferior BPDUs immediately accepted, as implies a change

### Proposal/Agreement
* On new link installation
 * Both ends designated discarding
 * DPs in dscarding/learning send BPDU with proposal
 * If one side sees BPDU now best, goes from DP to RP (stays discarding)
 * Proposal on RP makes all non-edge DPs int discarding (sync state)
 * Once done, RP to forwarding, upstream change to forwarding
 * Cascades

### TCN handling
* BPDUs flooded with TC flag
* Switch seeing TCN sets tcWhile to hello plus 1 sec on all non-edge DP and RP (except where TC learned)
* Flushes macs
* Sends TC flagged BPDUs on these ports until TC while expires

## RPVST+

* Non p2p switches revert to 802.1D, or PVST to legacy switches

## MST

* Sys ID used
* 0-4095 instances (0-15 only on 2950)
* 65 active (0 plus 64 user)
* Single BPDU for all info
* IST is Instance 0
* IST for outside region
* All VLANs have same port state as IST on boundary
* MST region is single switch outside it
* CST has no per vlan ability
* CST cost only cost of links between regions (external)
* CST on region boundary merges with IST inside (CIST)
* Multiple roots, one for entire region, rest per region (CIST Region Root)
* CIST Root - lowest BID from all CIST switches
 * IST BID from IST priority, instance 0, base mac
 * All STP and RSTP switches in this (using only BIDs)
* Non-CIST root regions, have only switches at region boundary in IST root switch election
* IST root elected by lowest external RPC to CIST root, sum of all inter-region links to reach region from root
 * Lowest IST BID if tie
* CIST Regional RP sitting on Region Root Switch is Master Port, provides connectivity to CIST root for all instances inside region

### Interop 

* STP and RSTP - Speak exclusively on instance 0
* MSTP and PVST+ - Single represenative on behalf of region, interaction determines port roles and states for all VLANS
 * MST side works (as port role for all instances matches IST)
 * MST must deliver info to PVST+ switches so every PVST+ instance makes same choice
* MST --> PVST+ - IST info replicated to PVST BPDUs on all active VLANs
* PVST+ --> MST - VLAN 1 instance for entire region
* MST Boundary becomes RP if BPDUs superior to bounary ports own BPDUs, but best VLAN 1 VPST+ BPDU on bounary (i.e. CIST root in PVST+ region and vlan 1)
 * All BPDUs checked to see if identical or superior to those in VLAN 1
 * If sysid ext in PVST+, cannot be identical, so would need to be at leas 4096 lower than PVST+ vlan
 * If not met, PVST simulation failed, port blocking
* MST boundary is non-DP if incoming VLAN 1 PVST+ bpdu superior, but not enough to be root, must monitor all PVST+ BPDUs
 * If true, port blocking, if not met, PVST simulation declared
* Best make MST region as root to all PVST+ instances

## Protecting and optimizing STP

* Portfast - Not part of Sync/PA
 * Expects no BPDUs, otherwise portfast disabled

**BPDU Guard**
* Per port, or globally
 * err disables port on BPDU Rx
 * `spanning-tree bpduguard enable`
 * `spanning-tree portfast bpduguard default`
 * `spanning-tree bpduguard disable`

**Root Guard**
* Ignores superior BPDUs
* Root Inconsistent
* Blocks port until BPDU cease
* Receovers when BPDUs expire (MaxAge-MessageAge in STP, 3x hello in RSTP)

**BPDU Filter**
* Stops Tx
* Optionally stops Rx
* If global configged, applies only to edge ports
 * 10 hellos send to start, then stops BPDUs
 * If BPDU Rx'd disables BPDU filter (back to 10 hellos)
* If per port, just stops completely
* Global along with per port bpdu guard works, received BPDU auto errdisables
* Per port doesn't make sense as BPDUs auto stopped anyway

**Unidirectional Link Issues**

* UDLD
 * Echo mechanism
 * Switch ID and port ID in message
 * If Switch/Port pair doesnt appear in other switches list, broken
 * If own seen, looped
 * If UDLD message contains more than one neighbour, shared media issue
 * all err disabled
 * Normal - Attempts to reconnect 8 times if messages lost, no action
 * Aggressive - as above, but err disables
* STP Loop Guard
 * Assumes if BPDUs rx'd on a port previously (i.e. on a Root or Alternate), and not working now, shouldn't be possible in working nework. Port can't be DP
 * Loop inconsistent, starts on loss of BPDU, stops on rx BPDU
* Bridge Assurance - RPVST+ and MST only, on p2p links
 * BPDUs sent on all ports as hellos
 * Loss of BPDUs make BA-inconsistent
 * `spanning-tree bridge assurance`
 * `spanning-tree portfast network`
* Dispute Mechanism
 * Role and state of port in RSTP and MST BPDUs
 * If inferior BPDU from a port claiming Designated Learning or Forwarding, moves to discarding
 * already exists

## Port Channels

* Must have same
 * Speed and duplex
 * Same mode (trunk, access, dynamic)
 * Same access vlan
 * Same allowed/native
 * No span ports
* Port suspended if no config match
* Config changes on non-suspended members only
* Ethernchannel Misconfig Gguard - BPDUs rx'd should have same source MAC. If not, ports are indiv links (doesn't help if only one BPDU rx'd over one link)
 

# Processes

## STP Choosing Ports

1. elect root switch (lowest BID)
2. Choose RP (superior cost to root)
3. Choose DP (switch forwarding superior BPDU to segment)

## Determining RP

1. Root sends hellos (every 2s), with RBID and SBID (Root ID for both), RPC 0, SPID of egress
2. Nonroot adds port cost to RPC, superior becomes RP
3. Hellos from RP sent out DP (updated RPC, SBID, SPID and MessageAge)
4. No hellos on blocking

## Determining DP

1. Hellos onto LAN
2. Port forwarding is DP
3. Superior hellos
4. All others root or blocked

## TC

1. TC Occurs
2. TCN BPDU out RP until acked
3. Upstream switches ack with next hello, marks TCA bit
4. 1-3 until root
5. BPDU with TCA sent through rx'd port by root
6. MaxAge+ForwardDelay seconds, root sends BPDUs with TC bit set, CAM entries reset

# Config

```
vtp mode server mst
vtp primary mst
```
* Uses VTP to do MST region config dispersion
