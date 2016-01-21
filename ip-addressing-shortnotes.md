# TCP Operation

Guarantees reliable delivery

# UDP Operation

Connectionless, acks could slow down process

# IP Addressing and Subnetting

## Subnetting classful network

Sie of network defined by 8, 16 or 24 its, host field shortened to creae subnets

* Network - 8/16/24 bits
* Subnet - 32 - network and host bits
* Host - Binary zeros in mask

## Subnetting math

Cisco allow zero subnet by default and broadcast subnets. Disable with **no ip subnet-zero**

# CIDR, Private addresses and NAT

## CIDR

RFC1517 to 1520
* Improving scalability of internet routers
* Aggregating routes for multiple classful nets into single entry
* Contiguous blocks assigned by ISPs
* Region auths assigned lare address blocks

## NAT

* RFC1631
* Private host to public network

Inside Local - Inside enterprise (private IP usually)
Inside GLobal - Inside ent, public IP
Outside Local - In internet, usually private
Outside Global - Internet, public

### Static NAT

Maps one address to another, no IP conservation

```
int E0/0
 ip address 10.1.1.3 255.255.255.0
 ip nat inside

int Se0/0
 ip address 8.8.8.1 255.255.255.248
 ip nat outside

ip nat inside source static 10.1.1.1 8.8.8.2 
ip nat inside source static 10.1.1.2 8.8.8.3
```

NAT only for inside addresses. Static outisde config'd looks at dest of inside to outside, source of outside to inside

### DYnamic NAT without PAT

One to one, from pool

### Overloading NAT with PAT

Large number of TCP and UDP flows appear behind fewer IPs. 

Each inside global can support 65k TCP and UDP flows

### DYnamic NAT and PAT config

```
int E0/0
 ip address 10.1.1.1 255.255.255.0
 ip nat inside

int Se0/0
 ip address 8.8.8.1 255.255.255.248
 ip nat outside

ip nat pool fred 8.8.8.2 8.8.8.3 netmask 255.255.255.252
ip nat inside source list 1 pool fred
access-list 1 permit 10.1.1.0 255.255.255.0
```

PAT would be

```
ip nat inside source list 1 pool fred overload
```

# IPv6

128 bits long

## Address Format

* Leading 0s replaced with ::
* Pair of colons represents successive 0 fields, only once per address

## Address types

* Unicast
* Multicast
* Anycast - Closest interface

## Address management and assignment

* Static
* SLAAC - Host autonomously configs, RS messages sent by host to request RAs, RFC2462
* Stateful DHCPv6 - Gets v6 address from server, similar to v4, RFC 3315
* Stateless DHCPv6 - SLAAC plus DHCP for TFTP, WINS etc

Config choice relies on RA flags sent by routers

### SLAAC

* Combines network prefix with interface ID
* Router on local link sends network info in RAs with prefix and default route
* Host vuilds address by adding EUI-64 to /64 prefix in RA.
* Easy to renumber hosts

### Stateful DHCPv6

* Similar to v4, but multicasts messages
* Client detects routers using ND messages
* If router found, look at RA to see if using DHCP
* Managed flag in RA for DHCP
* Autoconfig for none DHCP
* More control
* Can be used alongside SLAAC
* Renumbering
* Auto DNS registration of hosts
* Delegated v6 prefixes to leaf CPEs

### Stateless DHCP

Builds based upon SLAAC, then DHCP solicit for further info

## Transition technologies

### Dual stacks

Run both v4 and v6

### Tunnelling

Encaps v6 within v4 packets. Many types

**Dynamic tunneling config**

```
R2

int tun 23
 ipv6 address 23::2/64
 tunnel source lo0
 tunnel destination 3.3.3.3
 tunnelmode ipv6ip

R3

int tun 32
 ipv6 address 23::3/64
 tunnel source lo2
 tunnel destination 2.2.2.2
 tunnel mode ipv6ip
```

### Translation

AFT translates from one address family to another. V6 hosts with v4 contents. Can be stateless (reserved v6 maps to v4 automatically), can be stateful from configured range to map packets
