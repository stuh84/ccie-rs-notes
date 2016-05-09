# Messages

# Timers

# Trivia

## NAT

* Inside Local - Private IP
* Inside Global - Public IP
* Outside Local - Private IP
* Outside Global - Public IP

## IPv6

### Stateful DHCP

* Multicasts to FF02::1:2
* Clients detect routers using ND/RS messages
* Look at RA, see if Managed Config Flag
* Can delegate v6 prefixes to leaf CPEs

### Stateless DHCP

* Look at RA for Other Config Flag

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

**PAT** - Add overload to above command

# Verification

```
show nat64 aliases
show nat64 logging
show nat64 prefix stateful
show nat64 timeouts
```
