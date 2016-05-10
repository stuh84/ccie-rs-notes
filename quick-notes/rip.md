# Messages

* Request for full or partial update rather than waiting
* No partials on cisco

# Timers

* 30s full table updates
* Invalid after - 180s, holddown starts after
* Holddown - 180s -, routing entry stays int table until holddown expires
* Flushed After - 240s - reset after update - router emoves route

# Trivia

* UDP 520
* 224.0.0.9
* On demand full update once, silent until changes
* triggered updats
* Simple or MD5 auth
* Route tags allowed
* Can rewrite next hops
* Updates out each int every update timer
* 4 ECMP routes by default, can be bw 1 and 16 (or 32 if better platform)
* Split horizon by default unless FR or ATM (seen in show ip interface)
* Route poisoning means backup path found quickly
* When route removed, marked as possibly down in database (can be repeatedly advertised as unreachable)
* Flash/triggered updates moment change occurs
* Triggered extensions - On Demand Circuits
* Holddown used when updates could be lost in transit, next hop crashed, next hop router sees router as next hop (split horizon), RIP removed, summraization, next hop doesn't support poisoning
* After holddown, entry unlocked, convergence
* With auth, 24 routes instead of 25 per update (20 bytes for auth)
* Offset lists add to metric, before or after update

## RIPng

* UDP 521
* FF02::9
* No AFI support
* No auth support on IOS
* Split horizon per process only
* No passive ints
* No static neighbours
* Route entry of :: reverts send of message to next hop
* Multiple processes can be run (using name)
* Route poisoning activated per process
* Ints can have metric-offset value (applied to all updates, i.e. RIP link cost)
* Def route originated per interface

## Loop prevention

* Count to infinity - sudden increase in metric, accept and update, if reaches inifinity, stop using next hop

# Processes

# Config

```
router rip 
 version 2
 network 172.31.0.0
 passive-interface Lo0 - Still processes updates, just doesnt send any
 distribute-list BLA in/out (acl or prefix list), add int for outgoing int of route received

int Fa0/0
 ip rip authentication mode text/md5
 ip rip authentication key-chain NAME
```

```
ipv6 unicast-routing
ipv6 cef

int Fa0/0
 ipv6 address 2001::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 default-information only

int Se0/0
 ipv6 address 2001:1::1/64
 ipv6 rip 1 enable
 ipv6 rip 1 metric-offset 3

ipv6 router rip 1
 poison-reverse
```
