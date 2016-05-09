# Messages

# Timers

# Trivia

* FCS checked, if errors dropped
* No header checksum in v6

## CEF

* L2 headers preconstructed
* Constructed as routing table constructed
* Each enry in FIB has pointer to adj entry
* Recursion resolved on FIB creation
* FIB entries for same next hop point to same entry
* v4 and v6 had different adj entries, diff preconscructed headers
* ip cef
* ipv6 cef

### Load sharing
* Per packet
* Per dest - default
 * Pseudo load share table b/w fin and adj - 16 pointers to entries in adj
 * Entries populate so ratio of adj to cost of parallel routes - 8 per path for 2 ECMP, 5 per path for 3 ECMP routes
 * `ip load-share { per-destination | per-packet }`

## Polarization

* All loadshare to same link a lot of the time (all meeting same criteria)
* 4B long number called universal ID used as seeding function, different per router (hence different results)
* Algoritihms - Original (unseeded), Universal (seeded), Tunnel, L4 part (based on universal)
 * `ip cef load-sharing algorithm` and `ipv6 cef load-sharing algorithm`
* On cat6500 and others, `mls ip cef load-sharing`
 * Default - source, dest IP, uni id
 * Full - Source IP/Port, Dest IP/port - polarization
 * Simple - Source and dest, no uni ID
 * Full simple - Fewer adjs in hardware, similar to Full

## Internal Vlans

* 1006 and up, or 4094 down
 * `vlan internal allocation policy { ascending | descending }`
 * show vlan internal usage

## Policy routing

* ip policy on interface
* Route map specifies action
* SDM prefer template on certain platforms (3650 and 3860 need advanced, others need routing/access/dual-ipv4-and-ipv6-routing)
 * show sdn prefer
* set ip next hop, set int, set default ip, set default interface - Processed in this order of preference

# Processes

# Config
