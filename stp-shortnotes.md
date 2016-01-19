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
