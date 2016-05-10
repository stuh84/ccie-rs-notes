# Messages

# Timers

# Trivia

## PPP

### LCP

* Controls features independent of L3 protocol
* Each L3 proto has NCP
 * IPCP does dynamic IP assignment
* PPP comes up, CTS, then Data Send Read, and Data Carrier Detect to bring up physical
* LCP parameter negotiation
* Auth methods done with LCP
* After LCP, considered up
* L3 CP then begins
* Features
 * Link Quality Monitoring
 * Looped link detection
 * L2 Load Balancing
 * Auth
* ppp lcp predictive - Reduces neg time, predicts reponses from peers
* As above but ppp ipcp predictive

**LCP Process**

Config-Request ->
Config Reject 
Config Request ->
Config-Nack
Config Request
Config-Ac

* async mode interactive/dedicated
 * Dedicated - reserved for PPP and SLIP connections
 * Interactive, users prompted to connect to this interface
* ppp async - sending data cross async ints
 * byte stuffing - ecape character placed bfore control character in data stream, then mapped to other characters that data link wont confused
 * XON and XOFF non-printable

### IPCP

* Ignores mask requests and offers, problem with running RIP (use no ip validate source)
 * because RIP checks if belongs to same network as interface
 * Different subnet masks on interface as to source of update
 * Stop validating


### MLPPP

* Frags each data link frame - based on link number or config'd frag delay
* Header added including seq number, and flags with beginning and end frag
* LFI - Puts smaller packets inside bigger packets (frags them)
 * ppp multilink interleave
 * ppp multlink fragment-delay x - Frag size indrectly, delay in ms so size = x * bw

### PPP Compression

* Header or payload
* Header great with VoIP (TCP/RTP)
* LZ more cpu use, better ratio, predictor less cpu intensive
* Just add `compression` command on interfaces

### PPPoE
* hosts to aggregator
 
# Processes

# Config

## Basic LCP and PPP config

```
username R4 password 0

int Se0/1/0
 encapsulation ppp
 ppp authentication chap
```

## Multilink

```
int Multilink 1
 ip address 10.1.1.1 255.255.255.0
 encapsulation ppp
 ppp multilink
 ppp multilink group 1

int Se0/1/0
 encapsulation ppp
 ppp multilink group 1

int Se0/1/1
 encapsulation ppp
 ppp multilink group 1
```

## PPP compression

```
policy-map 1
 class voice
  bandwidth 82
  compress header ip rtp
 class other
  bandwith 100
  compression header ip tcp
```

## PPPoE

**Server**
```
bba-group pppoe BBA-GROUP
 virtual-tempalte 1
 session per-mac limit 2

int virtual-template 1
 ip address 10.0.0.1 255.255.255.0
 peer default ip address pool PPPOE_POOL
 ppp authentication chap callin

username PPP password PPPpassword

ip local pool PPPoE_POOL 10.0.0.2 10.0.0.254

int Fa0/0
 no ip address
 pppoe enable group BBA-GROUP
 no shutdown
```

**Client**
```
int dialer 1
 dialer pool 1
 encapsulation ppp
 ip address negotiated
 mtu 1492
 ppp chap password MyPassword

int Fa0/0
 no ip address
 pppoe-client dial-pool-number 1
 no shutdown
```

# Verification

```
show pppoe session
```
