# Messages

# Timers

# Trivia

* IPSec VTI - routable int, terminates IPSec, supports multicast

## DMVPN

* Each spoke persisent IPSec tun to hub
* Address of hub must be known by all spokes
* Spokes register address as client to NHS
* NHS maintains DB of public addr used by spokes
* Spoke to spoke - requests to NHS for pub ip of other spoke
 * On Demand

**Phase 1**
* Simple hub and spoke
* Dynamic IPs on spokes
* Hubs mGRE
* Spokes GRE

**Phase 2**
* Adds spoke-to-spoke comms
* Allow with `no ip next-hop-self eigrp AS` on tunnel
* No summarization
* OSPF single area, limited hubs due to OSPF DR/BDR
* First packet through hub process switched
* All mGRE

**Phase 3**
* Two NHRP enhancements
 * NHRP Redirect - Hub to spoke, lets spoke know of better path to other spoke through hub (Conf on hub only)
* NHRP Shortcut - Overwrites CEF info on spoke (configure on spoke and hub)
* Make sure `no ip split-horizon eigrp ASN` on for spoke to spoke
* No need for no ip next-hop-self, as NHRP will redirect regardless
* Packet still send to hup unless IPSec session comes up

* Routes go in as NHRP routes

## IPv6 Tunnelling

* Auto 6to4 - Point to MP, 2002::/16 - Connects isolated v6 islands
 * V6 prefix is 2002:border-router-ipv4-address::/48
 * one tunnel per router
 * Tunnel dest not config'd
 * Need to route 2002::/16 over tunnel
* Manual config - any addr space, P2P, dual stack required both ends, carries v6 across v4
* v6 over v4 gre - UNicast addr, p2p, carries any traffic across
* ISATAP - p2MP, any mcast, v6 hosts in single site
 * [64bit link local or global unicast prefix]:0000:5EFE:[IPv4 address of ISATAP Link]
 * Disables RAs by default
 * no ipv6 nd suppress-ra
* Auto v4-compat - P2MP, ::96 address space, dual stack both ends, deprecated, use ISATAP
 * First 96 bits tunnel int all 0s, rremaining 32 from v4 addr

## L2 VPNs

* Tagged mode - ID mathes on AC either end
 * VLAN match on each end
 * pw type of 0x0004
 * Every frame on PW different VLAN for each customer (VLAN = service delimiting tag)
 * If frame RX'd missing VLAN tag, PE prepends one
* Raw Mode - Tag not always present
 * PW Type 0x0005
 * Service tags never through AC
 * Stripped from frame before transmitting
* L2TPv3 
 * Any l2 payload
 * IP Proto 115
 * CEF required
 * Has control channel
* OTV - Looks like VPLS
 * Fault domain isolation
 * STP root doesn't change, each CE has own
 * Deployed at CE
 * Multicast similar to whats in L3 VPNs
 * like VPLS but wihtout MPLS transport
 * L2 LAN-E over L3, L2 or MPLS networks

## GET VPN

* Encrypts through insecure networks
* KS and GM
* KS creates, maintains and sends policy to GM
 * Policy is what traffic and algs
* KEK and TEK
 * KEK is between KS and GM, used for rekey phase
 * Tek for traffic
* Uses ESP
* RSA Key used by KS for Rekey
* New TEK and KEK before TEK expires (default 3600s)
* Phase authed and secure by ISAKMP SA between KS and GM
* GDOI messages build SA and Encrypt GM registration, UDP port 848

# Processes

# Config

```
int Fa0/0
 xconnect 4.4.4.4 204 encapsulation { mpls | l2tpv2 | l2tpv3 }
```

 ## GETVPN

**KS**
```
ip domain-name cisco.com

crypto key gen rsa mod 1024

crypto isakmp policy 10
 auth pre-share

crypto isakmp key GET-VPN-R5 address 10.1.25.5
crypto isakmp key GET-VPN-R4 address 10.1.24.4

crypto ipsec transform-set TSET esp-aes esp-sha-hmac

crypto ipsec profile GETVPN-PROF
 set transform-set TSET

crypto gdoi group GETVPN
 identity number 1
 server local

rekey auth mypubkey rsa R1.cisco.com
rekey transmit 10 number 2
rekey transport unicast/multicast

authorization address ipv4 GM-LIST

sa ipsec 1
 profile GETVPN-PROF
 match address ipv4 LAN-LIST <--- what to encrypt
 replay counter window-size 64
 address ipv4 10.1.12.1 <--- KS IPs

ip access-list standard GM-LIST
 permit 10.1.25.5
 permit 10.1.24.4

ip access-list extended LAN-LIST
 deny udp any eq 848 any eq 848
 permit ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
```

**GMs**

```
crypto isakmp policy 10
 auth preshare

crypto isakmp key GET-VPN-R5 10.1.12.1

crypto gdoi group GETVPN
 identity number 1
 server address ipv4 10.1.12.1

ip access-list extended DO-NOT-ENCRYPT
 deny tcp 192.168.4.0 0.0.0.255 eq 22 192.168.5.0 0.0.0.255

crypto map CMAP-GETVPN 10 gdoi
 set group GETVPN
 match address DO-NOT-ENCRYPT

int Se0/1/0.52
 crypto map CMAP-GETVPN
```
## GETVPN

**KS**
```
ip domain-name cisco.com

crypto key gen rsa mod 1024

crypto isakmp policy 10
 auth pre-share

crypto isakmp key GET-VPN-R5 address 10.1.25.5
crypto isakmp key GET-VPN-R4 address 10.1.24.4

crypto ipsec transform-set TSET esp-aes esp-sha-hmac

crypto ipsec profile GETVPN-PROF
 set transform-set TSET

crypto gdoi group GETVPN
 identity number 1
 server local

rekey auth mypubkey rsa R1.cisco.com
rekey transmit 10 number 2
rekey transport unicast/multicast

authorization address ipv4 GM-LIST

sa ipsec 1
 profile GETVPN-PROF
 match address ipv4 LAN-LIST <--- what to encrypt
 replay counter window-size 64
 address ipv4 10.1.12.1 <--- KS IPs

ip access-list standard GM-LIST
 permit 10.1.25.5
 permit 10.1.24.4

ip access-list extended LAN-LIST
 deny udp any eq 848 any eq 848
 permit ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
```

**GMs**

```
crypto isakmp policy 10
 auth preshare

crypto isakmp key GET-VPN-R5 10.1.12.1

crypto gdoi group GETVPN
 identity number 1
 server address ipv4 10.1.12.1

ip access-list extended DO-NOT-ENCRYPT
 deny tcp 192.168.4.0 0.0.0.255 eq 22 192.168.5.0 0.0.0.255

crypto map CMAP-GETVPN 10 gdoi
 set group GETVPN
 match address DO-NOT-ENCRYPT

int Se0/1/0.52
 crypto map CMAP-GETVPN
```
## GETVPN

**KS**
```
ip domain-name cisco.com

crypto key gen rsa mod 1024

crypto isakmp policy 10
 auth pre-share

crypto isakmp key GET-VPN-R5 address 10.1.25.5
crypto isakmp key GET-VPN-R4 address 10.1.24.4

crypto ipsec transform-set TSET esp-aes esp-sha-hmac

crypto ipsec profile GETVPN-PROF
 set transform-set TSET

crypto gdoi group GETVPN
 identity number 1
 server local

rekey auth mypubkey rsa R1.cisco.com
rekey transmit 10 number 2
rekey transport unicast/multicast

authorization address ipv4 GM-LIST

sa ipsec 1
 profile GETVPN-PROF
 match address ipv4 LAN-LIST <--- what to encrypt
 replay counter window-size 64
 address ipv4 10.1.12.1 <--- KS IPs

ip access-list standard GM-LIST
 permit 10.1.25.5
 permit 10.1.24.4

ip access-list extended LAN-LIST
 deny udp any eq 848 any eq 848
 permit ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
```

**GMs**

```
crypto isakmp policy 10
 auth preshare

crypto isakmp key GET-VPN-R5 10.1.12.1

crypto gdoi group GETVPN
 identity number 1
 server address ipv4 10.1.12.1

ip access-list extended DO-NOT-ENCRYPT
 deny tcp 192.168.4.0 0.0.0.255 eq 22 192.168.5.0 0.0.0.255

crypto map CMAP-GETVPN 10 gdoi
 set group GETVPN
 match address DO-NOT-ENCRYPT

int Se0/1/0.52
 crypto map CMAP-GETVPN
```

# Verification

## DMVPN

```
show dmvpn

show crypto isakmp sa
show crypto ipsec sa
show crypto ipsec session

show nhrp

show crypto gdoi group NAME
show crypto gdoi ks policy
show crypto gdoi ks all
show crypto gdoi ks members
```


