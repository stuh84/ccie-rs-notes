# 802.1D STP and improvements

* 802.1D for STP originally
* 802.1w RSTP
* 802.1s MSTP
* 802.1D-2004 now only RSTP
* 802.1s integrated into 802.1Q-2005

![STP BPDU](https://raw.githubusercontent.com/stuh84/ccie-rs-notes/master/images/stp-bpdu.png)

STP  
* Protocol ID of 0x0000
* Protocol Ver - 0x00
* BPDU field shows config and tcn BPDUs
* Flags handles TC events (TC Ack flag and TC flag)
* Fields after show root bridge, distance of BPDU sender to root, sender bridge ID, sender port ID that BPDU traversed
* MessageAge - Set to 0 at root, others increment by 1 before forwarding
* Remaining lifetime of bpdu is MaxAge-MessageAge
* Timers reflect timers of root (MaxAge, HelloTime, ForwardDelay)

IDs of bridges and ports in BPDU. All have configurable priority.

Config BPDUs compared, superior based upon lowest: -

* RBID
* Root Path Cost
* Sender BID
* Sender PID
* Receiver PID (not in BPDU, local)

Above is order. Only config BPDUs compared. One RP on non-root, one DP per segment. TCNs not compared.

STP stores superior BPDU sent/received. Root and blocking ports store upstream BPDU, DP store own

BPDU lasts for MaxAge-MessageAge seconds

BID - 2 byte priority then MAC

## Choosing ports

1. Elect root switch (lowest BID)
2. Choose RP - Superior BPDU to root
3. Choose DP - Swich that forwards superior BPDU from all forwarded BPDUs on segment

### Electing Root

* All claim root until superior BPDU
* Once sup received, advertised on

802.1t amended BID for PVST+ and MST. Made priority 4 bits (multiples of 4096), 12 bits for Sys Ex ID (usually Vlan ID). Called MAC address reduction. VLAN IDs make ID unique (VLAN ID plus switch MAC). Configure with **spanning-tree extend system-id**, newer switches can't remove this command.

### Determining RP

1. Root sends hellos (default 2 seconds). Contains RBID and SBID (Root ID for both), RPC 0, SPID of egress
2. Nonroot adds port cost to RPC in BPDU, superior becomes RP
3. Hellos from RP sent out DP (updated RPC, SBID, SPID and MessageAge)
4. No hellos on blocking ports

Least path cost to root. RPC on received port added to BPDU.

Port costs: -

Standard | 10M | 100M | 1G | 10G
---------|-----|------|----|----
Pre-802.1D-1998 |100|10|1|1|
802.1D-1998|100|19|4|2|
802.1D-2004|2000000|200000|20000|2000

1998 default on most CAT switches. 2004 for MST by default. Change with **spanning-tree pathcost method long**

### Determining DP

* Hellos forwarded onto LAN segment by designated switch
* Port forwarding is DP
* ALl others root or blocked
* Superior hellos

Tiebreakers same as before

### Summary of rules

* Root Switch - Lowest BID
* RP - Least path cost to root
* DP - Sends best BPDUs to segment
* ALl other ports blocking
* Config BPDUs only from DP (would be inferior on other ports, including RP)
* Each port stores best BPDU sent/received. DP store best sent, RP and block store best rx'd
* RXd BPDUs expire - MaxAge-MessageAge

## Converging to a new STP topology

When stable, BPDUs unchanged, same results calculated.

TC event when: -
* TCN BPDU rx'd on DP
* Port moves to forwarding and switch has at least one DP
* Port moves from Learning or forwarding to block
* Switch becomes root

During TC, switch sends BPDUs with updated contents. Neighbours recalculate ports

## TCN and updating CAM

* Switches instructed to age out unused CAM entries
* Forward Delay timer (default 15s) to time out CAM entries

TCNs go to Root, root informs switches. TCNs sent as config BPDUs cannot go upstream.

1. TC occurs
2. Switch sends TCN BPDUs out RP until acked
3. Upstream switch acks with next hello, marks TCA bit
4. Upstream repeats 1-3
5. When TCN at root, BPDU with TCA sent through rx'd port
6. For MaxAge+ForwardDelay seconds, root sends BPDUs with TC bit set, all switches time out CAM entries

## Transitioning from blocking to forwarding

No immediate change as loop potential.

Listening then learning, both ForwardDelay (default 15s)

Listening - No forwarding, no MAC learning
Learning - No forwarding, MAC learning

# PVST and STP over trunks

PVST+ has STP per VLAN. Different roots, different ints for forwarding/blocking.

With 802.1Q and non-Cisco switches, only CST.

PVST+ runs on trunks as VLAN 1 STP instance. In CST regions, binding for all VLANs. In PVST+ region, only for 1 VLAN.

* CST treated as loop-free shared segment
* PVST+ BPDUs encapd with multicast dest mac of 0100.0CCC.CCCD, tagged with correct VLAN
* Using SNAP encap (ordinary BPDUs use LLC). 
* TLV at end of BPDU with VLAN number of BPDU. Used to check for native VLAN mismatches
* VLAN1 on PVST+ uses standard BPDUs and PVST+ BPDU. PVST+ one only for mismatches

When sending BPDUs, access ports tx IEEE BPDUs to their access VLANs.

Trunks do: -
* IEEE BPDU for VLAN 1 (untagged)
* PVST+ BPDUs for all existing and allowed VLANs

If access port gets BPDU with wrong VLAN, Type Inconsistent.

On trunk: 

* IEEE BPDUs processed by VLAN 1 STP instance
* PVST+ BPDUs go through: -

1. Check VLAN tag, if tagged, BPDU in that VLAN. If no tag, native VLAN
2. Check PVID TLV. If no match, PVIDInconsitent state, BPDU dropped
3. If PVID TLV match, processed by VLAN STP. PVST+ VLAN 1 duplicate of IEEE.

## STP Config and Analysis

**show spanning-tree root** - If on root, "This bridge is root"

**spanning-tree vlan 1 priority 28672**

```
int Fa0/1
 spanning-tree vlan 1 cost 100
```

**spanning-tree vlan vlan-id root { primary | secondary } [diameter value] - Diameter lowers timers, sets pririty to 24756 if current root larger, or 4096 below root. Secondary set to 28672 always.

# RSTP

IEEE 802.1w
