# Messages

# Timers

# Trivia

## NAT

* Inside Local - Private IP
* Inside Global - Public IP
* Outside Local - Private IP
* Outside Global - Public IP

## IPv6

### General Prefix

* Defined, like a summary, changes to general affect more specific

### Extension headers

* Next Header field indicates if more
* Each EH can refer to upper layer or next
* Includes Basic v6 header, hop by hop, destination options, routing header, AH, ESP, frag header, mobnility header etc etc
* Can be matched by ACLs

### Stateful DHCP

* Multicasts to FF02::1:2
* Clients detect routers using ND/RS messages
* Look at RA, see if Managed Config Flag
* Can delegate v6 prefixes to leaf CPEs

### Stateless DHCP

* Look at RA for Other Config Flag

### DHCPv6-PD

* Prefix delegation - set a router to hand out subsets from a larger prefix
 * eg 2001:db8::/32, hand out 48s from this
 * v6 client picks up 48, and then uses on interface

### Transition technologies

**NAT-PT**
* Not recommended when host dual stacked
* Static - v6 mapping to v4 address (v4 must be stable)
* Dynamic - pooling, temp address, at least one static mapping for v4 dns server, ACLs determine what packets translated
* PAT - Single v4
* IPv4 Mapped - ACL check to see if source in ACL/list, if rule for source address translation, last 32 bits of destv6 should be v4
 * DNS ALG v4 converted to v6 address, dns packets translated

**NAT64**
* Stateless - v6 to v4 and vice versa, no state, supports v4 or v6 initiated comms
* Stateful - Creates/moidfies bindings
* ALG required when IP info in comms (FTP, SIP)
* AFT (Address Family Translation)
* Fragging done by fragging v6 datagram, and setting DF bits in v4 header, done by Stateless NAT64 translator

**Stateful caveats**
* No mcast
* Cold redundancy
* No options, routing headers etc
* No VRF support
* TCP/UDP only
* v6 to v4, dest IP must match stateful prefix to NAT hairpin, source must not match one (dropped if so)
* No route-maps
* No ICMP + FTP ALGs
* Only supports v6 initiation
* Can't have NAT44 and 64 on same int
* Source v6 add associated with v4 config'd pool
* Dest v6 based on NAT64 stateful prefix or well known prefix
* Translated and fw

## IPv4 Options

* Options 0  and 1 exactly one octet (type field)
* All others 1 octet type, length, 2 octets for data
* Type is one bit copied field, two bit class, five bit option
 * Copied means whether to go in each fragment
* Option - 0 for control, 2 for debugging and measurement
* Option numbers
 * 0 - end of list
 * 1 - no operation
 * 2 - security - security codes
 * 3 - Loose Source routing - can forward to any intermediate routes to get to dest
 * 4 - Internet timestamp
 * 7 - Record route - records route datagrams take
 * 8 Stream Id - 4 octets
 * 9 - Strict Source routing - can only forward based on what source route indicated
 
## Static NAT and IP Aliasing
* No-alias in NAT command prevents router from installing local IP alias
* Router still responds to ARP for global translated IPs
* Does not terminate connection itself
* Essentialy means it will not respond to arp for global addresses

## Policy NAT

* NAT with same dest can map to different IPs (route map)

## Stateful NAT with HSRP
* State info must be transferred before state change
* Has HSRP mode or active/backup mode
* UDP Comms for stateful info
* Can use TCP

## Stateful NAT

* Allows for Inside/Outside assymetry
* Works with some ALGS

# Processes

# Config

## NAT

**Static**

```
int E0/0
 ip nat inside

int E0/1
 ip nat outside

ip nat inside source static 10.1.1.1 8.8.8.1
```

**NAT pool**

```
int E0/0
 ip nat inside

int E0/1
 ip nat outside

ip nat pool fred 8.8.8.3 8.8.8.4 netmask 255.255.255.252
ip nat inside source list 1 pool fred
access-list 1 permit 10.1.1.0 255.255.255.0
```

**Overload**

```
int E0/0
 ip nat inside

int E0/1
 ip nat outside

access-list 1 permit 10.1.1.0 255.255.255.0
ip nat inside source list 1 pool fred overload
OR
ip nat inside source list 1 interface E0/1 overload
```

## Stateless NAT64 config

```
nat64 enable
nat64 prefix stateless 2001:0db8:0:1::/96 <--- add 32 bit v4 to this
nat64 route 203.0.113.0/24 Gi0/0/0 <--- routes v4 traffic to correct v6 int
ipv6 route 2001:db8:0:1::CB00:7100/120 Gi0/0/0 <--- Routes translated packets to v4 address
```

**Multiple prefixes**
```
nat64 prerix stateless v6v4 2001:0db8:0:1::/96 <--- On interface
nat64 prefix stateless v4v6 2001:db8:2::/96 <--- On interface
```

## Stateful

**Static** - As per stateless except

```
nat64 prefix stateful PREFIX/LENGTH
nat64 v6v4 static V6ADDR V4ADDR
```

**Dynamic** - no static part
```
nat64 v4 pool NAME STARTIP ENDIP
nat64 v6v4 list ACL pool NAME
```

## DHCPv6-PD

**Server**
```
ipv6 dhcp pool dhcpv6

prefix-delegation pool dhcpv6-pool1 lifetime 1800 600

ipv6 local pool dhcpv6-pool1 2001:db8:1200::/40 48

int Se0/0
 ipv6 dhcp server dhcpv6
```

**Client**

```
int Se0/0
 ipv6 address autoconfig default

ipv6 dhcp client pd prefix-from-provider <--- NAME

int Fa0/0
 ipv6 address prefix-from-provider ::1:0:0:0:1/64
```

**PAT** - Add overload to above command

## General prefix

```
ipv6 general-prefix NAME prefix/length|6to4 INT

int e0/0
 ipv6 address PREFIX-NAME sub-bits/length
```

# Policy NAT

```
access-list 102 permit tcp host 1.1.1.1 host 2.2.2.2 eq 80
access-list 103 permit tcp host 1.1.1.1 host 2.2.2.2 eq 443

route-map One 
 match ip address 102 
 set int Fa0/0

route-map Two
 match ip address 103
 set int Fa0/1

ip nat inside-source route-map NAME int Fa0/0
ip nat inside source static 1.1.1.1 1.1.1.3 route-map NAMES
```

## Stateful NAT with HSRP

```
int E0/0
 standby GROUP ip
 standby GROUP preempt delay .....

ip nat stateful-id NUMBER redundancy HSRPGROUP mapping-id NUM [protocol udp] [as-queuing disable]
ip nat inside source route-map NAME pool NAME mapping-id NUMBER [overload]
```

## Stateful NAT with Active/Backup

```
ip nat stateful id NUM primary IP peer IP mapping-id NUM
ip nat inside source route-map NAME pool NAME mapping-id NUMBER [overload]
```

* Backup instead of primary for standby

```
ip nat inside destination LIST NUM pool NAME mapping-id ID
ip nat outside source static GLOBIP LOCIP extendable mapping-id ID
```
* Above allows ALG

## NAT Default Interface

```
ip nat inside source-list ALL interface S0/0 overload --- Inbound out
ip nat inside source static X.X.X.X int Se0/0 --- Outbount in to X.X.X.X
```

# Verification

```
show nat64 aliases
show nat64 logging
show nat64 prefix stateful
show nat64 timeouts
```
## Stateful NAT

```
show ip snat distributed verbose
```
