# ARP, Proxy ARP, Reverse ARP, BOOTP, DHCP

* Learn another hosts MAC - ARP, Proxy ARP
* Host discovers its own IP - RARP, BOOTP, DHCP

## ARP and Proxy ARP

* RFC826 - ARP
* RFC 1027 - Proxy ARP

**ARP**
* ARP destination of 255.255.255.255
* No IP header, source and dest IP in same relative position
* Protocol type 0x0806

**Proxy ARP**
* Same as ARP messages but for MAC not on local subnet
* Router can issue Proxy ARP request on behalf of target
* LAN broadcast, router replies with own MAC
* Before DHCP, proxy ARP relied on, hosts used default masks in networks

## RARP, BOOTP, DHCP

* For host to dynamically learn IP
* Broadcast to begin discovery
* All rely on server hearing request

### RARP

* ARP messages used
* ARP request lists its own MAC as target
* Target IP of 0.0.0.0
* Preconf'd RARP server (same subnet as client) receives request
* Server looks in config
* ARP reply with configured IP in source IP field

### BOOTP

* RFC 951 defined messages
* Commands encap'd in IP and UDP header
* Can go to other subnets
* Can assign subnet mask, def gw, DNS and IP of boot server
* Preconfig required

### DHCP



