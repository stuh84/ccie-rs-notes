# Router and Switch Device Security

## Simple Password Proection for CLI

```
line con 0
 login 
 password dave

line vty 0 15
 login
 password barney
```

* **service password-encryption** 
 * Weak encryption
 * Not changed in startup until copy run start or wr mem
* Not auto decrypted on **no service password-encryption**

## Better Protection of enable and username passwords

* **enable password** and **enable secret**
* Secret stored as MD5 hash, not affected by service password-encryption
* If both defined, precedence on secret

## Using SSH

* Auth with local un and pw, or AAA server
* v1 vulnerable to MITM attacks
* Telnet enabled by default

Steps for SSH: -

1. IOS SSH support, K9 image required
2. **hostname**
3. **ip domain-name**
4. Client auth method
5. Generate RSA keys
6. Specify SSH version if v2 required
7. Disable telnet on VTY
8. Enable SSH on vty

```
hostname R3
ip domain-name CCIE2b
username cisco password DAVE-LIKES-SSH
crypto key gen rsa

ip ssh version 2

line vty 0 4
 transport input none
 transport input ssh
```

## User Mode and Privileged Mode AAA Auth

* Strongest method to protect CLI is TACACS+ or RADIUS (eg ACS)
* Router sends un and pw encrypted to server, awaits reply before
  rejecting

|Feature|Radius|TACACS+|
|-------|------|-------|
|Scope of encryption|Password Only|Entire Payload|
|L4 Protocol|UDP|TCP|
|Well-Known Port/IOS default for Auth|1812/1645|49/49|
|Standard or Prop|RFC2865|Prop|

### Using default set of auth methods

* Single auth method a way to auth user
* one could be a radius to auth
* Another to let router look at local
* List tried in order until an auth response (accepting or rejecting
  user)
* Simple AAA config defines default auth methods for all logins, and
  second set for enable
* Defined default login apply to all login session (Console/Telnet/Aux)

1. Enable AAA auth with **aaa new-model**
2. Define servers and keys, **radius-server host, radius-server key,
tacacs-server host, tacacs-server key**
3. Define default auth methods for all CLI access **aaa authentication
login default**
4. Define default set of auth methods for enable **aaa authentication
enable default**

```
enable secret 5 <MD5-HASH>
username cisco password 0 cisco

aaa new-model

aaa authentication enable default group radius local
aaa authentication login default group radius none
radius-server host 10.1.1.1 auth-port 1812 acct-port 1646
radius-server host 10.1.1.2 auth-port 1645 acct-port 1646
```

### Using Multiple Auth Methods

* Four methods in command
* No practical limit to servers
* Logic is, try first method, if does not responde, move to next
 * First to respond with auth or reject stops process
 * If no response for any method, reject request

|Method|Meaning|
|------|-------|
|**group radius**|Use config'd RADIUS servers|
|**group tacacs+**|User config's TACACS sevrers|
|**aaa group server ldap**|AAA Server group with name and enters LDAP config mode|
|**group** *name*|Use defined group of either servers|
|**enable**|Use enable password|
|**line**|Use config in line (cant use with enable auth|
|**local**|Username commands in local config, username case insensitive, password sensitive|
|**local-case**|Above, but case sensitive username|
|**none**|No auth, user automatically auth'd|

### Groups of AAA Servers

* By default, all radius servers in RADIUS group, same for TACACSs
* Can create groups

```
aaa group server radius fred
 server 10.1.1.3 auth-port 1645 acct-port 1646
 server 10.1.1.4 auth-port 1645 acct-port 1646

aaa new-model
aaa authentication enable default group fred local
aaa authentication login default group fred none
```

### Overriding Defaults for Login Security

* Console VTY and Aux can override
* In line config, **login authentication** *name*, point to named config
  methods

```
aaa authentication login for-console group radius line
aaa authentication login for-vty group radius local
aaa authentication login for-aix group radius

line con 0
 password 7 48728934792
 login auth for-console

line aux 0 
 login auth for-aux

line vty 0 4
 password 7 78970952
 login authentication for-vty
```

## PPP Security

* PAP and CHAP can be for auth, useful for dial
* Default auth methods use local un and pw

AAA for PPP

1. **aaa new-model**
2. Config RAD/TAC servers
3. **aaa authentication ppp default**
4. **aaa authentication ppp** *list-name method1 method2*
5. **ppp authentication** *protocol list-name*, eg ppp authentication chap fred

# Layer 2 Security

* Unused - Not connected to devices
* User - Cabled to end evices, or cable drop to physically unprotected area
* Trusted or trunk - Fully trusted devices in areas of good phy security

Best practices on unused and user
* Disable unneeded dynamic proto (CDP, DTP)
* Access ports only
* BPDU and Root guard
* DAI or PVLANs to prevent frame sniffing
* Port security to limit macs, try to restrict to specific macs
* 802.1x
* DHCP snooping and IP source guard

Other recommendations
* For any port, use PVLANs
* Configure VTP auth globally
* Disabled unused ports and placed into dead VLAN
* Avoid VLAN 1
* Avoid native VLAN on trunks

Best PRactices on Unused/User ports
```
int Fa0/0
 no cdp enable
 switchport mode access
 switchport nonegotiate
 spanning-tree guard root
 spanning-tree bpduguard enable
```

### Port Security

* Can restrict number of macs on ports
* Can enforce certain macs on ports
* switch considers whether allowing, before adding source MAC to port

Supports following: -
* Limiting number of MACs
* Limiting actual macs
 * Static config
 * Dynamic learning up to defined maximum, dynamic entries lost on
   reload
 * Dynamic learning, saved in config (sticky)

* Attackers could claim same MAC as user
* Attacker could fill up mac tables

* Ports be be static trunk or access, not dynamic

* **switchport port-security [maximum-value]** - default 1
* **switchport port-security mac-address** *mac-address [vlan {vlan-id |
  {access | voice }}]* - Defines allowed MACs, for a VLAN (trunk) or for
access/voice
* **switchport port-security mac-address sticky** - Remembers dynamic
  learned MACs
* **switchport port-security [aging] [violation {protect | restrict |
  shutdown } ]** - Defines aging timer, and action taken

* First two required
* If config saved with sticky, they become static
* Protect tells switch to perform port security, restrict sends SNMP
  traps too, shutdown err-disables port, shut/no shut to recover

* **show port-security interface** - SecureUp means port up and secured

## Dynamic ARP Inspection

* MITM ca happen by gratuitous ARP
* GAs occur by sending ARP reply without ARP being sent, b'cast dest
* Can be used to claim IP of host
* DAI defeats this
* Examines ARP message, filters inapproriate messages
* Each port untrusted (default) or trusted
* DAI on untrusted

Message inappropriate if: -

1. ARP reply lists source IP not DHCP assigned to that device and port
2. As above, but with static defined IP/MAC combos
3. For received reply, source MAC in header checked against source MAC
in ARP. Should be equal in normal replies
4. Like Step 3, but dest MAC and target MAC
5. Check for unexpected IPs (0.0.0.0, 255.255.255.255, multicast)

* Requires DHCP snooping, for DHCP snooping binding db
* **ip arp inspection vlan** *vlan-range* - Global
* **[no] ip arp inspection trust**
* **ip arp inspection filter** *arp-acl-name* **vlan** *vlan-range
  [static]* - Refers to ARP ACL, defines static entries
* **ip arp inspection validate {[src-mac] [dst-mac] [ip]}** - Additional
  options
* **ip arp inspection limit {rate PPS [burst interval SECONDS] | none }** - Limits ARP rate to prevent DOS attacks
* Auto limit to 15 ARP messages per second

## DHCP Snooping

* Builds table of IP and port mappings, based on legitimate DHCP
  messages
* Table used by DAI and IP Source Guard
* DHCP clients should be on untrusted ports
* DHCP servers on trusted

