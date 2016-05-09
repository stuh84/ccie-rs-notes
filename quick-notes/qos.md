# Messages

# Packet formats

* IP header
* 802.1p
* FR Header
* ATM Cell Priority

## IP Header and DSCP

* RFC791 defined 1 byte ToS field
 * High order 3 bytes IPP
  * Routine, Priority, Immediate, Flash, Flash Override, Critical, Internetwork Control, Network Control
 * 3 to 6 - flags, 7 not defined
* ToS byte renamed DiffServ
 * 6 bit field instead of IPP (0-5 bits)
  * Replaces precedence and ToS
  * ToS byte and DS field same length

### DSCP

* Decimal 46 - EF
* Class Selector
 * RFC 2475
 * CS0 - Binary 000000
 * CS1 - Binary 001000
 * CS2 - Binary 010000
 * Etc up to CS7
* AF PHB
 * Three levels of drop prob
 * 4 Classes
 * 4xy, X being queue, Y being priority
  * Higher X, better treatment
  * Higher Y worse drop
  * AF11, 12 and 13 - 001010, 001100, 001110
  * AF21, 22 and 23 - 010010, 010100, 010110
  * AF31, 32 and 33 - 011010, 011100, 011110
  * AF41, 42 and 43 - 100010, 100100, 100110
  * Decimal from name - 8x + 2y
  * Still IPP compatible

### LAN CoS

* 3 bit QoS Field
* Only with .1q or ISL
* 3 most sig bits in tag control, user priority in 1q
* 3 least sig from 1 byte user field, called CoS in ISL

### WAN

* FR and ATM have single bit for drop probablity
* DE for FR, CLP for ATM
* Can be set by rotuer or ATM/FR switch
* MPLS EXP bit, remarks DSCP or IPP usually

# Trivia

* Classify on ingress only, mark on egress only

## MQC

* All MQC tools "Class Based"
* Class map matches
 * Can match QoS Fields, ACLs, macs
 * Case sensitive
 * match protocol - nbar
 * Match any - all packets
 * Up to four IPP or 8 DSCP in single match CoS, Prec or DSCP
  * matches any, not all
 * Can define to match any or match all for multiple matches
 * Match class matches another class map (so can match all in one, then match any in another)
* Policy map applies actions
* service policy binds to interface

### NBAR

* Can refer to hostname, URL, mime type etc
* Citrix
* RTP matching on even number only, can classify payload (odd numbers are contro)

# Processes

# Config
