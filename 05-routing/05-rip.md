# RIP

**Level:** Intermediate | **Prerequisites:** [The Routing Table](02-routing-table.md), [Dynamic Routing Overview](04-dynamic-routing-overview.md), [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md)

## In One Line

RIP (Routing Information Protocol) is a simple routing protocol where each router tells its direct neighbors, every 30 seconds, which networks it knows and how many routers away they are.

## Why It Matters

Picture a company with 10 routers. You could type static routes on each one, but every new network means editing 10 boxes. When a link fails, nothing moves traffic to another path until a person acts.

RIP solves this. Routers share routes by themselves and repair the table when something breaks.

RIP is old and almost nobody picks it for a new network today. Interviewers still ask about it for three reasons:

- It is the cleanest way to test whether you understand distance vector routing.
- Its problems (routing loops, count to infinity) explain why OSPF and BGP were designed the way they were.
- A few exact facts (15 hops, 30 seconds, UDP 520, 224.0.0.9) are easy to check quickly.

Key words:

- **Routing protocol:** software that lets routers share routes with each other automatically.
- **Metric:** the number a protocol uses to compare paths. Lower is better. In RIP, the metric is hop count.
- **Hop count:** the number of routers a packet must cross to reach a network.
- **Distance vector:** a family of protocols where a router learns only "this network is this far away, in that direction" from its neighbors. It does not see the whole map. See [Dynamic Routing Overview](04-dynamic-routing-overview.md).
- **Convergence:** the moment when all routers agree on the current paths after a change.
- **Interior gateway protocol (IGP):** a routing protocol used inside one organization. RIP is an IGP.

## Simple Explanation

Imagine a line of villages with no maps. Every 30 seconds, each villager stands at the edge of the village and shouts to the next village: "I can reach Town A in 1 step and Town B in 3 steps."

The neighbor hears this and adds 1 step, because reaching the shouter takes one more step. Now the neighbor knows the distance and the direction, and repeats the news to the next village.

Nobody has a map. Each person trusts what the person next door says. After a few rounds, everyone knows the distance to every town.

Where the analogy stops being true:

- Villagers could doubt a rumor. RIP believes its neighbors with no way to check, unless you add authentication.
- Steps in the analogy feel like distance. RIP counts only routers, not speed or cable length. A slow link and a fast link both count as 1.

## How It Works

### RIP in plain facts

| Fact | Value |
|---|---|
| Protocol type | Distance vector, interior gateway protocol |
| Metric | Hop count |
| Maximum usable metric | 15 |
| Unreachable metric | 16 (called infinity) |
| Update interval | 30 seconds |
| Transport | UDP port 520 (RIPv1, RIPv2), UDP port 521 (RIPng) |
| Cisco administrative distance | 120 |

**Administrative distance** is a trust score a router uses when two protocols offer the same prefix. Lower wins. The value 120 is a Cisco default, not part of RIP itself. Juniper uses a preference of 100 for RIP.

RIP runs over UDP, which has no delivery guarantee. That is acceptable here, because a lost update is replaced by the next one 30 seconds later. See [UDP in Depth](../06-transport-layer/07-udp.md).

### What a router does, step by step

1. **Startup.** RIP puts its directly connected networks in its table. It then sends a Request message to ask neighbors for their full tables, so it does not wait 30 seconds.
2. **Sending.** Every 30 seconds, it sends a Response message on each RIP interface. The message carries its routes and metrics.
3. **Receiving.** For each route in a received update, the router adds 1 to the metric (capped at 16), and then applies these rules:
   - Network not in the table: add it, unless the metric is 16.
   - Update comes from the current next hop: accept it, even if the metric got worse. Reset the timer.
   - Update comes from a different router: accept it only if the metric is better.
4. **Aging.** If updates for a route stop coming, the route is first marked bad, and later removed.

Rule 3 has a detail that many people miss. The current next hop is always believed. If the router you are using says the path got worse, you must accept that, because your path runs through it.

### How routes spread

Use three routers in a line. Follow one network, 192.168.3.0/24, from R3 to R1.

```text
 192.168.1.0/24                                     192.168.3.0/24
       |                                                     |
       | G0/0 .1                                     G0/0 .1 |
    +----+      10.0.12.0/30     +----+    10.0.23.0/30    +----+
    | R1 |-----------------------| R2 |--------------------| R3 |
    +----+ G0/1 .1     .2 G0/0   +----+ G0/1 .1    .2 G0/1 +----+
```

| Step | Who sends | What it says | Result |
|---|---|---|---|
| 1 | R3 to R2 | 192.168.3.0/24, metric 1 | R2 installs it, metric 1, via 10.0.23.2 |
| 2 | R2 to R1 | 192.168.3.0/24, metric 2 | R1 installs it, metric 2, via 10.0.12.2 |

R1 never hears from R3. It only knows "go to R2, and it is 2 hops." This is why people call distance vector "routing by rumor."

### Hop count and the 15-hop limit

Each router crossed adds 1. Metric 16 means unreachable, so 15 is the largest usable value.

On Cisco IPv4 RIP, this gives a practical size. A network on R1 is metric 1 at R2, 2 at R3, and 15 at R16. At R17, it would be 16, which means unreachable. So a straight chain of 16 routers works and a chain of 17 does not.

Why such a small number? Infinity must be small, because of a loop problem described below: routers may count upward until they reach it. A big infinity would mean a very long count. The 15 limit was a deliberate trade-off, not a technical accident.

**The hop count trap:** RIP ignores speed.

```text
              10 Mbps link (1 hop)
     R1 ------------------------------------ R3
      \                                     /
       \ 10 Gbps                   10 Gbps /
        \                                 /
         -------------- R2 ---------------
                  (2 hops)
```

RIP sends traffic over the 10 Mbps link, because 1 hop beats 2 hops. OSPF would choose the fast path, because its metric is based on bandwidth by default.

### The three RIP versions

| Feature | RIPv1 | RIPv2 | RIPng |
|---|---|---|---|
| Standard | RFC 1058 (1988) | RFC 2453 (1998) | RFC 2080 (1997) |
| IP version | IPv4 | IPv4 | IPv6 |
| Classful or classless | Classful | Classless | Classless |
| Sends mask or prefix length | No | Yes | Yes |
| VLSM and CIDR | No | Yes | Yes |
| Updates sent to | Broadcast 255.255.255.255 | Multicast 224.0.0.9 | Multicast ff02::9 |
| Transport | UDP 520 | UDP 520 | UDP 521 |
| Authentication | None | Plain text, MD5 (later stronger hashes) | None of its own, relies on IPsec |
| Route tag | No | Yes | Yes |
| Next hop field | No | Yes | Yes (link-local address) |
| Metric limit | 15 | 15 | 15 |

New words in this table:

- **Classful:** the protocol assumes the mask from the first bits of the address (10.x.x.x is /8, 172.16.x.x is /16, 192.168.1.x is /24). See [IPv4 Classes](../04-ip-addressing/02-ipv4-classes.md).
- **Classless:** the mask travels with the route, so any prefix length works.
- **VLSM (Variable Length Subnet Masks):** using different mask sizes inside one network. See [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md).
- **Multicast:** one packet sent to a group. Only devices that joined the group process it. Broadcast, by contrast, makes every device on the link read the packet. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).
- **Route tag:** a 16-bit label attached to a route, often used to mark routes that came from another protocol.
- **Link-local address:** an IPv6 address valid on one link only, starting with fe80. See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).

RIPv2 is not a new protocol. It uses the same message structure as RIPv1 and fills fields that RIPv1 left as zero. Hop count, timers and limits stay the same.

RIPng means "RIP next generation." It is not a small edit of RIPv2. It has its own message format and cannot carry IPv4 routes.

### The timers

**Fuller truth:** the standard and Cisco use different names and values for the last stages. Interviewers usually want the Cisco set, but a good candidate knows the difference.

| Timer | Standard (RFC 2453) | Cisco IOS default | Meaning |
|---|---|---|---|
| Update | 30 s | 30 s | How often the full table is sent |
| Timeout / Invalid | 180 s | 180 s | No update for this long: the route is bad |
| Garbage collection | 120 s after timeout | Not used by this name | Keep advertising the route as 16, then delete |
| Holddown | Not one of the core timers | 180 s | Do not trust news about this route |
| Flush | Not used by this name | 240 s from the last update | Remove the route from the table |

FRRouting (open source routing software for Linux) follows the standard, with 30, 180 and 120 seconds.

Two details worth knowing:

- **Random offset.** If all routers sent at the same moment, their updates could collide and stay locked together. The standard adds a random offset of up to 5 seconds either way to each 30-second interval.
- **Cisco flush comes before holddown ends.** Invalid plus holddown would be 360 seconds, but flush happens at 240. So with defaults, the route is deleted 60 seconds after going invalid, and holddown effectively ends there.

Here is a silent failure on Cisco. R2 dies without any signal to R1, so R1's link stays up.

```text
 Time (s)    0       30      60    ...   180            240
             |       |       |            |              |
 Update      got     lost    lost        lost            |
 R1 route    good    good    good     INVALID         FLUSHED
                                   "possibly down"    (removed)
                                   starts holddown
```

During the 180 seconds, R1 keeps sending traffic toward a dead router. That delay is the core weakness of RIP.

### Routing loops and count to infinity

Take R2 and R3 from the earlier diagram. Suppose R3's LAN fails, and RIP has none of its usual protections. Think of two friends. One says "the shop is closed," but the other replies "no, I can get there in 1 step." The first friend believes it, not knowing that the 1-step path ran through himself.

| Round | R3's view of 192.168.3.0/24 | R2's view |
|---|---|---|
| Start | Lost, metric 16 | Metric 1 via R3 |
| 1 | Hears R2, metric 2 via R2 | Metric 1 via R3 |
| 2 | Metric 2 via R2 | Hears R3, metric 3 via R3 |
| 3 | Hears R2, metric 4 via R2 | Metric 3 via R3 |
| ... | Keeps climbing | Keeps climbing |
| End | Metric 16, removed | Metric 16, removed |

While this runs, packets bounce between R2 and R3 until their TTL (the IP hop limit) runs out. This is **count to infinity**. The value 16 is what stops it.

Where the analogy stops being true: friends would soon notice they are talking in a circle. Routers cannot, because a RIP update holds no information about the path behind a metric.

### The loop prevention tools

| Tool | What it does | Real-life version |
|---|---|---|
| Maximum metric | 16 means unreachable, so counting stops | "After 15 steps, give up" |
| Split horizon | Never advertise a route out of the interface it was learned on | Do not tell someone directions they gave you |
| Route poisoning | When a route fails, advertise it at metric 16 | Tell everyone "that shop is closed" |
| Poison reverse | Send a learned route back to its source at metric 16 | "I go through you, so do not use me for that" |
| Holddown | Ignore news of alternate paths for a while after a failure | Ignore rumors for a while after bad news |
| Triggered update | Send a change immediately instead of waiting up to 30 s | Do not wait for the next shouting time |

Fuller truth:

- Split horizon stops loops between two routers. In a ring of three or more routers, a loop can still form. The metric limit is the final safety net.
- The standard asks for a short random delay of 1 to 5 seconds between triggered updates, to avoid a storm of messages.
- On Cisco IOS, split horizon is normally on. Poison reverse is off for normal periodic updates, but failed routes are still poisoned.

### Why RIP is legacy

- **15-hop limit.** A large network simply does not fit.
- **Slow convergence.** A silent failure takes 180 seconds to detect, which is far too long for voice or video. OSPF can react in seconds, or less with BFD (Bidirectional Forwarding Detection, a fast link check).
- **Blind to bandwidth.** Hop count ignores speed.
- **Full table every 30 seconds.** It sends everything even when nothing changed.
- **No structure.** There are no areas or hierarchy, so all routers carry all routes.
- **Weak security by default.** RIPv1 has no authentication. RIPv2 has it, but you must turn it on.

The cost of updates matters less than people think. A full 25-route update is 532 bytes at the IP level (20 IP, 8 UDP, 504 RIP). With 1,000 routes, that is 40 packets, or 21,280 bytes every 30 seconds, about 5.7 kbps on each link. On a 64 kbps WAN link of the 1990s, that was about 9 percent. On a modern link it is nothing. Today's real reasons to avoid RIP are convergence and hop limit.

### Where RIP still lives

- Old branch sites and legacy equipment not yet replaced.
- Some small routers and customer-premises devices.
- Open source stacks such as FRRouting and BIRD, which still include it.
- Teaching labs, because it takes four lines to configure.

Cisco's current CCNA focuses on static routes and OSPF. RIP remains in many older books, labs and interview question lists.

### Why it still gets asked

An interviewer who asks about RIP is rarely checking if you can deploy it. The question usually tests these ideas:

- Do you understand how a router learns from neighbors?
- Can you explain what a routing loop is and how protocols prevent one?
- Can you explain why hop count is a poor metric, and so why OSPF uses cost?
- Do you know classful versus classless, and why masks in updates matter?

## Example

### Configuring RIPv2 on three routers (Cisco IOS)

The topology is the earlier diagram. R2 has no LAN.

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# router rip
R1(config-router)# version 2
R1(config-router)# no auto-summary
R1(config-router)# network 10.0.0.0
R1(config-router)# network 192.168.1.0
R1(config-router)# passive-interface GigabitEthernet0/0
```

```text
R2(config)# router rip
R2(config-router)# version 2
R2(config-router)# no auto-summary
R2(config-router)# network 10.0.0.0
```

```text
R3(config)# router rip
R3(config-router)# version 2
R3(config-router)# no auto-summary
R3(config-router)# network 10.0.0.0
R3(config-router)# network 192.168.3.0
R3(config-router)# passive-interface GigabitEthernet0/0
```

What each line does:

| Command | Purpose |
|---|---|
| `router rip` | Start the RIP process |
| `version 2` | Send and receive RIPv2 only. Without it, classic IOS sends v1 and receives v1 and v2 |
| `no auto-summary` | Stop shrinking subnets to classful networks at boundaries |
| `network 10.0.0.0` | Run RIP on all interfaces in that classful network, and advertise their subnets |
| `passive-interface` | Do not send RIP updates out this interface |

The `network` command takes no mask. If you type `network 10.0.12.0`, IOS stores `network 10.0.0.0`, because it works on classful boundaries.

Auto-summary defaults have differed across IOS versions, so always type `no auto-summary` and confirm with `show ip protocols`.

### Checking the result

```text
R1# show ip route rip
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
R        10.0.23.0/30 [120/1] via 10.0.12.2, 00:00:12, GigabitEthernet0/1
R     192.168.3.0/24 [120/2] via 10.0.12.2, 00:00:12, GigabitEthernet0/1
```

Reading `[120/2]`: 120 is the administrative distance and 2 is the hop count. `00:00:12` is the time since the last update. If this number passes 30, updates are being lost.

```text
R1# show ip protocols
Routing Protocol is "rip"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Sending updates every 30 seconds, next due in 18 seconds
  Invalid after 180 seconds, hold down 180, flushed after 240
  Redistributing: rip
  Default version control: send version 2, receive version 2
    Interface             Send  Recv  Triggered RIP  Key-chain
    GigabitEthernet0/1    2     2
  Automatic network summarization is not in effect
  Maximum path: 4
  Routing for Networks:
    10.0.0.0
    192.168.1.0
  Passive Interface(s):
    GigabitEthernet0/0
  Routing Information Sources:
    Gateway         Distance      Last Update
    10.0.12.2            120      00:00:12
  Distance: (default is 120)
```

```text
R1# show ip rip database
10.0.12.0/30    directly connected, GigabitEthernet0/1
10.0.23.0/30
    [1] via 10.0.12.2, 00:00:12, GigabitEthernet0/1
192.168.1.0/24    directly connected, GigabitEthernet0/0
192.168.3.0/24
    [2] via 10.0.12.2, 00:00:12, GigabitEthernet0/1
```

Watch the updates live. `debug` output can load a busy router, so use it in a lab or with care.

```text
R1# debug ip rip
RIP protocol debugging is on
RIP: received v2 update from 10.0.12.2 on GigabitEthernet0/1
     10.0.23.0/30 via 0.0.0.0 in 1 hops
     192.168.3.0/24 via 0.0.0.0 in 2 hops
RIP: sending v2 update to 224.0.0.9 via GigabitEthernet0/1 (10.0.12.1)
RIP: build update entries
     192.168.1.0/24 via 0.0.0.0, metric 1, tag 0
```

Split horizon is visible here. R1 learned 10.0.23.0/30 and 192.168.3.0/24 on G0/1, so it sends neither back out of G0/1. The `via 0.0.0.0` means "use the sender as the next hop."

### A link failure

The link between R2 and R3 goes down. R2 sees its interface go down, so there is no 180-second wait.

1. R2 removes 10.0.23.0/30 and 192.168.3.0/24.
2. R2 sends a triggered update to R1, with both routes at metric 16.
3. R1 marks them unreachable, starts holddown and sends nothing about them back as a good route.

```text
RIP: received v2 update from 10.0.12.2 on GigabitEthernet0/1
     10.0.23.0/30 via 0.0.0.0 in 16 hops (inaccessible)
     192.168.3.0/24 via 0.0.0.0 in 16 hops (inaccessible)
```

```text
R1# show ip route 192.168.3.0
Routing entry for 192.168.3.0/24
  Known via "rip", distance 120, metric 4294967295 (inaccessible)
  Redistributing via rip
  Hold down timer expires in 171 secs

R1# show ip route rip
R     192.168.3.0/24 is possibly down, routing via 10.0.12.2, GigabitEthernet0/1
```

So a failure the router can see spreads in seconds. A failure it cannot see, such as a dead neighbor, takes minutes.

### Other common configuration

Send a default route to all RIP routers. On R1, which here has an internet link:

```text
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
R1(config)# router rip
R1(config-router)# default-information originate
```

```text
R2# show ip route rip | include 0.0.0.0
Gateway of last resort is 10.0.12.1 to network 0.0.0.0
R*    0.0.0.0/0 [120/1] via 10.0.12.1, 00:00:05, GigabitEthernet0/0
```

Change the timers (all routers must match, or routes will flap):

```text
R1(config-router)# timers basic 30 180 180 240
```

Add MD5 authentication on a link, on both ends:

```text
R1(config)# key chain RIPKEYS
R1(config-keychain)# key 1
R1(config-keychain-key)# key-string ExamplePass1
R1(config-keychain-key)# exit
R1(config-keychain)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip rip authentication mode md5
R1(config-if)# ip rip authentication key-chain RIPKEYS
```

Summarise routes out of an interface:

```text
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip summary-address rip 10.2.0.0 255.255.0.0
```

### RIPng on Cisco IOS

The same routers, with IPv6. R1 uses 2001:db8:1::1/64 on its LAN and 2001:db8:12::1/64 toward R2. R2 uses 2001:db8:12::2/64 and 2001:db8:23::1/64. R3 uses 2001:db8:23::2/64 and 2001:db8:3::1/64 on its LAN. Link-local addresses are set by hand: fe80::1 on R1 and fe80::2 on R2.

```text
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 router rip CAMPUS
R1(config-rtr)# exit
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ipv6 address 2001:db8:1::1/64
R1(config-if)# ipv6 rip CAMPUS enable
R1(config-if)# interface GigabitEthernet0/1
R1(config-if)# ipv6 address fe80::1 link-local
R1(config-if)# ipv6 address 2001:db8:12::1/64
R1(config-if)# ipv6 rip CAMPUS enable
```

There is no `network` command. You enable RIPng on each interface. `CAMPUS` is a local process name, so it does not need to match on neighbors.

```text
R1# show ipv6 route rip
R   2001:DB8:3::/64 [120/3]
     via FE80::2, GigabitEthernet0/1
R   2001:DB8:23::/64 [120/2]
     via FE80::2, GigabitEthernet0/1

R1# show ipv6 rip
RIP process "CAMPUS", port 521, multicast-group FF02::9, pid 62
     Administrative distance is 120. Maximum paths is 16
     Updates every 30 seconds, expire after 180
     Holddown lasts 0 seconds, garbage collect after 120
     Split horizon is on; poison reverse is off
     Default routes are not generated
     Periodic updates 5, trigger updates 2
```

Note that the next hop is a link-local address, and that R3's LAN shows metric 3 here, compared with 2 in the IPv4 example. The Going Deeper section explains why.

### RIP on Linux and viewing RIP routes

Linux has no RIP in the kernel. You run a routing daemon such as FRRouting. Set `ripd=yes` in `/etc/frr/daemons`, then use its Cisco-like shell.

```text
! Linux, FRRouting (run: sudo vtysh, then: configure terminal)
router rip
 version 2
 network 10.0.0.0/8
 network eth1
 passive-interface eth1
 redistribute connected
```

Unlike Cisco, FRR accepts a real prefix or an interface name in `network`.

```bash
# Linux: view and capture
sudo vtysh -c "show ip rip"
ip route show proto rip
sudo tcpdump -ni eth0 udp port 520
```

```text
10:15:17.455012 IP 10.0.12.1.520 > 224.0.0.9.520: RIPv2, Response, length: 24, routes: 1 or less
```

The length 24 is 4 bytes of RIP header plus one 20-byte route.

```powershell
# Windows: list routes installed by RIP, if any
Get-NetRoute -Protocol RIP
```

Windows Server can run RIP through the Routing and Remote Access role. Most Windows machines never use it.

## Going Deeper

### RIPv1 and RIPv2 message format

The UDP payload is a 4-byte header followed by 1 to 25 route entries of 20 bytes each.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+-------------------------------+
|  Command (1)  |  Version (1)  |        Must be zero (2)       |
+---------------+---------------+-------------------------------+
|     Address Family (2)        |          Route Tag (2)        |
+-------------------------------+-------------------------------+
|                         IP Address (4)                        |
+---------------------------------------------------------------+
|                         Subnet Mask (4)                       |
+---------------------------------------------------------------+
|                          Next Hop (4)                         |
+---------------------------------------------------------------+
|                           Metric (4)                          |
+---------------------------------------------------------------+
                  Up to 25 route entries in total
```

| Field | Notes |
|---|---|
| Command | 1 = Request, 2 = Response |
| Version | 1 or 2 |
| Address Family | 2 for IPv4. 0xFFFF marks an authentication entry |
| Route Tag | RIPv2 only. Zero in RIPv1 |
| Subnet Mask | RIPv2 only. Zero in RIPv1 |
| Next Hop | RIPv2 only. 0.0.0.0 means use the sender |
| Metric | 1 to 16 |

The maximum size is 4 + (25 x 20) = 504 bytes of RIP data, or 512 bytes with the 8-byte UDP header. A router with more routes sends more messages.

With authentication, the first entry holds the auth data, so only 24 routes fit in that message.

A Request with a single entry of address family 0 and metric 16 means "send me your whole table."

### RIPng message format

The header is the same size, and the version value is 1 (a good trick question). Each route entry is 20 bytes:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+-------------------------------+
|  Command (1)  |  Version (1)  |        Must be zero (2)       |
+---------------+---------------+-------------------------------+
|                                                               |
|                       IPv6 Prefix (16)                        |
|                                                               |
|                                                               |
+-------------------------------+---------------+---------------+
|         Route Tag (2)         | Prefix Len (1)|  Metric (1)   |
+-------------------------------+---------------+---------------+
```

- There is no fixed limit of 25. The count depends on the link MTU. With an MTU of 1500: (1500 - 40 IPv6 - 8 UDP - 4 RIPng) / 20 = 72 entries.
- A next hop is sent as a special entry with metric 0xFF, and it applies to the entries after it.
- Updates use a link-local source address and a hop limit of 255, so a receiver can reject packets from off the link.

### Why RIPv1 cannot handle subnets well

RIPv1 sends no mask, so the receiver must guess:

- If the route is in the same major network as the receiving interface, it uses the interface's mask.
- Otherwise it assumes the classful mask.

This forces all subnets of a major network to use one mask, so VLSM is not possible.

It also breaks **discontiguous networks**, where parts of one major network are separated by another network. Take the same three routers, but use LANs 172.16.1.0/24 on R1 and 172.16.3.0/24 on R3. Both routers cross into 10.0.0.0 and summarise to 172.16.0.0/16.

```text
R2# show ip route rip
R     172.16.0.0/16 [120/1] via 10.0.12.1, 00:00:09, GigabitEthernet0/0
                    [120/1] via 10.0.23.2, 00:00:21, GigabitEthernet0/1
```

R2 sees one route with two paths. Traffic for a LAN can go to the wrong router and fail. RIPv2 with auto-summary left on fails the same way. The fix is `version 2` with `no auto-summary`, so that the real /24 routes are advertised.

### Hop count on Cisco versus the standard

**Fuller truth:** the standard says a receiver adds the cost of the incoming link (normally 1) to the metric it hears, and a directly connected network has cost 1.

Cisco IPv4 RIP reaches the same behavior between routers, but displays it differently. It advertises a connected network as metric 1 and installs that value as received. So one router away shows `[120/1]`.

Cisco RIPng adds 1 on receipt, so one router away shows `[120/2]`. This is why the same network appears as 2 in IPv4 and 3 in RIPng in the examples above. Other vendors may vary, so read the display, not your assumption.

### passive-interface works differently in RIP

In RIP, `passive-interface` stops sending updates, but the router still listens and accepts them. In OSPF and EIGRP, it stops Hello messages, so no neighbor forms and no routes arrive at all.

To truly block RIP in from a segment, use a filter such as `distribute-list` or an ACL.

### Fixing the hop count trap

An `offset-list` adds to the metric to make a path look worse. To push R1 away from the slow link in the earlier 10 Mbps example:

```text
R1(config)# access-list 10 permit 192.168.3.0 0.0.0.255
R1(config)# router rip
R1(config-router)# offset-list 10 in 5 GigabitEthernet0/2
```

Routes matching ACL 10 learned on G0/2 now get 5 extra hops.

### Mixed versions and migration

Cisco interfaces can send and receive per version:

```text
R1(config-if)# ip rip send version 1 2
R1(config-if)# ip rip receive version 1 2
```

This helps during a move from v1 to v2. Remember that v1 cannot carry VLSM routes correctly.

### Security

Without authentication, any device on the link can send a fake RIP update. This can pull traffic through an attacker or black-hole a network. RIPv2 supports:

- Plain text passwords, which anyone with a packet capture can read.
- Keyed MD5 (first in RFC 2082, now covered by RFC 4822), which is better but old.
- RFC 4822 also defines SHA-family options. Availability depends on the vendor.

Use `passive-interface` on user-facing ports, and ACLs to limit UDP 520 and 521. See [Access Control Lists](../09-security/05-access-control-lists.md).

Cisco also runs a check named `validate-update-source` by default. It ignores updates whose source address is not on the receiving interface's subnet.

### Triggered RIP for WAN links

RFC 2091 extends RIP for demand circuits, which are links that cost money or wake up when used. Periodic updates stop after the first full exchange, and only changes are sent. On Cisco, this is `ip rip triggered`. It is rare today.

### Redistribution

RIP can import routes from other sources. On Cisco, routes taken from another routing protocol need an explicit metric, or they are treated as unreachable. Keep it below 16, and low enough to allow for the hops they will still cross.

```text
R1(config-router)# redistribute ospf 1 metric 3
R1(config-router)# redistribute static
```

See [Advanced Routing Techniques](12-advanced-routing-techniques.md).

### Troubleshooting checklist

| Symptom | Likely cause | Check |
|---|---|---|
| No routes from neighbor | `network` missing, or interface passive | `show ip protocols` |
| `ignored v1 packet` in debug | Version mismatch | `show ip protocols`, `debug ip rip` |
| `invalid authentication` in debug | Key or mode mismatch | `show ip protocols`, key chain |
| Subnets appear as /8 or /16 | Auto-summary on, or RIPv1 | `show ip route` |
| Route age climbs above 30 s | Lost updates, ACL blocking UDP 520 | `show ip route`, ACLs |
| Far networks missing | Metric reached 16 | `show ip rip database` |

### Where distance vector went next

The ideas in RIP did not die. EIGRP keeps distance vector but adds a loop-free algorithm. See [EIGRP](07-eigrp.md). BGP is a path vector protocol: it sends the full list of networks crossed, so a router can spot a loop in the path itself. See [BGP](09-bgp.md). Babel (RFC 8966) is a modern distance vector protocol, used in some small and wireless mesh networks.

## Common Mistakes

- **"RIP allows 16 hops."** No. 15 is the maximum usable metric. 16 means unreachable.
- **"RIPv2 fixed the hop limit."** No. v2 added masks, multicast and authentication. The 15-hop limit and 30-second timer stayed the same.
- **"RIPv2 updates only send changes."** No. It sends the full table every 30 seconds, with triggered updates in addition.
- **"RIP finds the fastest path."** No. It finds the fewest routers. A 10 Mbps link at 1 hop beats two 10 Gbps links.
- **"RIPng is RIPv3."** No. It is a separate protocol for IPv6, using UDP 521 and ff02::9, and its version field is 1.
- **"RIPv1 is classless if you use subnets."** No. It sends no mask, so it cannot carry VLSM.
- **"RIP runs on TCP."** No. It uses UDP.
- **"The flush timer is 240 seconds after invalid."** No. On Cisco, it is 240 seconds from the last update, so 60 seconds after invalid.
- **"Standard RIP and Cisco use the same timers."** Not quite. The standard uses a 120-second garbage collection timer. Flush and holddown are Cisco terms.
- **"`network 192.168.1.0` advertises only that network."** It does two things: it enables RIP on matching interfaces, and advertises those interfaces' subnets.
- **"`passive-interface` makes RIP ignore the interface."** No. It stops sending. The router still accepts updates there.
- **"Split horizon removes all loops."** No. It covers two-router loops. Larger rings still rely on the metric limit and holddown.

## Interview Questions

**Q: What is RIP, and what metric does it use?**

A: RIP is a distance vector interior routing protocol. It uses hop count as its metric, which is the number of routers to reach a network. The lowest hop count wins.

**Q: Why is the maximum hop count 15?**

A: 16 is defined as infinity, meaning unreachable. It is kept small because routers can count upward during a loop, and a low limit stops that quickly. The cost is that RIP does not fit large networks.

**Q: What is the difference between RIPv1, RIPv2 and RIPng?**

A: RIPv1 is classful, sends broadcasts and has no authentication. RIPv2 is classless, carries masks, uses multicast 224.0.0.9 and supports authentication. Both use UDP 520. RIPng is for IPv6, uses UDP 521 and ff02::9, and relies on IPsec for security.

**Q: What are the RIP timers?**

A: On Cisco, update is 30 seconds, invalid is 180, holddown is 180 and flush is 240. The standard uses 30, a 180-second timeout, and a 120-second garbage collection period. So the standard deletes a route after 300 seconds, while Cisco flushes at 240.

**Q: How does RIP prevent routing loops?**

A: It uses a maximum metric of 16, split horizon, route poisoning, poison reverse, holddown timers and triggered updates. Split horizon stops a router from advertising a route back where it learned it. Poisoning tells others that a route is dead.

**Q: What is count to infinity?**

A: After a route fails, two routers may keep believing the other still has a path. Each advertises a slightly higher metric, and they count up until they reach 16. During that time, packets loop between them.

**Q: Why does RIP have slow convergence?**

A: Updates run on a 30-second cycle, and a silent failure is only noticed after 180 seconds of missed updates. Holddown adds more delay. Triggered updates help when a router sees its own link fail, but not when a neighbor dies quietly.

**Q: RIP is legacy, so why learn it?**

A: It shows distance vector routing in its simplest form. Loops, split horizon and hop count limits explain why OSPF uses link state and cost, and why BGP carries a path. It also still appears on old networks.

**Q: Why is hop count a poor metric? Give an example.**

A: It ignores bandwidth, delay and load. If a direct 10 Mbps link is 1 hop and a path through a third router over 10 Gbps links is 2 hops, RIP picks the slow link. OSPF would pick the fast one.

**Q: Why does auto-summary break discontiguous networks?**

A: Auto-summary shrinks subnets to the classful network at a boundary. If two sites use 172.16.1.0/24 and 172.16.3.0/24, separated by 10.0.0.0, both advertise 172.16.0.0/16. The router in the middle cannot tell which side holds which subnet. Using `no auto-summary` with RIPv2 fixes it.

**Q: How does passive-interface differ between RIP and OSPF?**

A: In RIP, it stops the router sending updates, but it still accepts them. In OSPF, it stops Hellos, so no neighbor forms and no routes are exchanged on that interface.

**Q: What transport and addresses does RIP use, and why is UDP acceptable?**

A: RIPv1 and RIPv2 use UDP 520, and RIPng uses UDP 521. RIPv2 sends to 224.0.0.9 and RIPng to ff02::9. UDP is fine because the full table is sent again in 30 seconds, so one lost update is repaired by the next.

## Quick Recap

- RIP is a distance vector IGP that uses hop count. 15 is the maximum and 16 means unreachable.
- It sends the full table every 30 seconds. Cisco timers are 30, 180, 180, 240. The standard is 30, 180, plus 120 garbage collection.
- RIPv1 is classful and broadcast. RIPv2 is classless, multicast to 224.0.0.9, and has authentication. Both use UDP 520.
- RIPng is the IPv6 version, on UDP 521 and ff02::9, with no authentication of its own.
- Loops are controlled with the metric limit, split horizon, poisoning, holddown and triggered updates.
- RIP is legacy because of its hop limit, slow convergence and blindness to bandwidth.
- It is still asked because it tests routing basics, and its flaws explain the design of OSPF and BGP.
