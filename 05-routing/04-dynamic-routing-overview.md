# Dynamic Routing Overview

**Level:** Intermediate | **Prerequisites:** [How Routers Work](01-how-routers-work.md), [The Routing Table](02-routing-table.md), [Static and Default Routes](03-static-and-default-routes.md)

## In One Line

Dynamic routing lets routers talk to each other, share what they know about the network, and rebuild their routing tables by themselves when something changes.

## Why It Matters

A router forwards a packet only if it has a route to the destination. A **route** is one entry that says "to reach this network, send the packet to this next router". You can type routes by hand. These are static routes. They work well in a small network and fail badly in a large one.

Look at real numbers. Take 10 routers and 25 networks: 10 LANs and 15 links between routers. Each router is directly connected to about 4 of those networks, so it must learn the other 21.

- By hand, that is 10 x 21 = 210 static routes.
- Adding one new LAN means logging in to 9 routers.
- If a link fails, traffic can keep going to the dead path until a person fixes it.

With a routing protocol, each router needs only a few lines of config. A new LAN is announced automatically. A failed link is routed around in seconds, with nobody awake at 3 a.m.

The Internet has roughly 1 million IPv4 routes and more than 200,000 IPv6 routes, and they change all day. No human team could type that.

Interviewers ask about this topic for three reasons:

- It tests whether you understand *why* OSPF and BGP are designed the way they are.
- Questions such as "distance vector vs link state" and "what is count to infinity?" are classics.
- Loops and slow convergence cause real outages, so the ideas are used in daily troubleshooting.

## Simple Explanation

**Static routing** is a printed list of directions. It is simple and predictable. If a road closes, the paper does not change.

**Dynamic routing** is a group of drivers who share road news with each other. When a bridge closes, the news spreads, and every driver picks a new way.

There are three styles of sharing, and each has its own analogy.

**Distance vector: road signs.** At a junction, a sign says "Lahore 50 km, this way". You learn a direction and a distance. You do not see the roads ahead. You trust the sign. A distance vector router knows only what its neighbours tell it. People call this "routing by rumour".

*Where the analogy stops:* road signs do not change by themselves. Distance vector routers update each other all the time.

**Link state: everyone has the same map.** Each driver reports the roads next to their own house. These reports are copied to all drivers. Soon every driver holds the same full map, and each one uses GPS to find the best way.

*Where the analogy stops:* a router does not get a picture. It gets a list of facts, such as "R1 connects to R2 at cost 1", and builds a graph from them.

**Path vector: a list of countries.** A travel agent says "to reach Warsaw, go through France, then Germany, then Poland". You see the list of countries, not each road. You can refuse a trip through a country you dislike. If your own country is on the list, you know the trip goes in a circle.

*Where the analogy stops:* the shortest list is not always the chosen one. Business policy often wins, as you will see in [BGP](09-bgp.md).

## How It Works

### What a routing protocol does

A **routing protocol** is a set of rules that routers use to exchange route information. Every routing protocol does the same five jobs:

1. **Find neighbours.** A **neighbour** is another router that speaks the same protocol on a shared link. Most protocols send small "hello" messages to find them.
2. **Share information.** Routers exchange what they know.
3. **Choose the best path.** A **metric** is the number a protocol uses to compare paths. Lower is better in RIP, OSPF, EIGRP and IS-IS.
4. **Install routes.** The best routes are offered to the routing table.
5. **React to change.** When a link fails or a network appears, the protocol repeats steps 2 to 4.

All this work happens in the **control plane**, the part of the router that makes decisions. Moving the actual packets happens in the **data plane**.

```text
 +-------------------------------------------------+
 | CONTROL PLANE (makes decisions)                 |
 |                                                 |
 |  neighbours <--> routing protocol (OSPF, BGP)   |
 |                        |                        |
 |                        | best routes            |
 |                        v                        |
 |                  routing table (RIB)            |
 +------------------------|------------------------+
                          v
 +-------------------------------------------------+
 | DATA PLANE (moves packets)                      |
 |                                                 |
 |  forwarding table (FIB) --> out the interface   |
 +-------------------------------------------------+
```

RIB means Routing Information Base. FIB means Forwarding Information Base. [The Routing Table](02-routing-table.md) explains both.

A note on two words that sound alike. A **routed protocol** carries user data, and IP is the main example. A **routing protocol** carries route information, and OSPF is an example. OSPF tells routers where to send IP packets.

### Static vs dynamic routing

| Point | Static routing | Dynamic routing |
|---|---|---|
| Who adds routes | A person | Routers, automatically |
| Reacts to failure | No, unless a backup route is prepared | Yes |
| Scale | Small networks | Small to Internet size |
| CPU, memory, link use | Almost none | Some, depends on protocol |
| Predictability | Very high | Lower, paths can move |
| Security | Nothing is advertised | Needs authentication and filtering |
| Good for | Stub sites, default routes | Any network with many paths or frequent change |

Real networks use both. A branch office often has one [static default route](03-static-and-default-routes.md), while the core runs OSPF.

### Autonomous systems

An **autonomous system (AS)** is a group of networks under one administration with one routing policy. "Policy" here means the rules for which routes to accept and which paths to prefer. RFC 1930 gives the formal definition.

Analogy: an AS is like a country. Inside, it makes its own road rules. At the border, it deals with other countries under agreements.

*Where the analogy stops:* one company can run several ASes, and an AS has no fixed geography. It is a routing idea, not a place.

Each AS on the public Internet has a globally unique **AS number (ASN)**. Regional Internet Registries such as RIPE NCC and ARIN assign them. Real examples are AS 13335 (Cloudflare) and AS 15169 (Google).

| ASN range | Use |
|---|---|
| 0 | Reserved |
| 1-64495 | Public 16-bit ASNs (23456 is reserved for 4-byte ASN transition) |
| 64496-64511 | Documentation |
| 64512-65534 | Private use, 16-bit |
| 65535 | Reserved |
| 65536-65551 | Documentation |
| 65552-4199999999 | Public 32-bit ASNs (with some reserved blocks) |
| 4200000000-4294967294 | Private use, 32-bit |
| 4294967295 | Reserved |

ASNs began as 16-bit numbers. They grew to 32 bits because the 16-bit pool was running out (RFC 6793). Private ASNs are defined in RFC 6996. They are like private IP addresses: usable inside, but they should not appear on the public Internet.

A small company with one ISP usually does not need a public ASN. It needs one when it connects to two or more ISPs and wants to control its own routing.

### IGP vs EGP

Routing protocols are split by where they run.

An **IGP (Interior Gateway Protocol)** runs inside one AS. Its goal is to find the best, fastest path. The AS trusts all its own routers. RIP, OSPF, EIGRP and IS-IS are IGPs.

An **EGP (Exterior Gateway Protocol)** runs between ASes. Its goal is to apply policy across organisations that do not fully trust each other. BGP is the only EGP in use today.

"Gateway" is the old word for router.

```text
 AS 64512               AS 64513               AS 64514
 +---------------+      +---------------+      +---------------+
 |  R1 --- R2    | eBGP |  R3 --- R4    | eBGP |  R5 --- R6    |
 |  OSPF inside  |<---->|  IS-IS inside |<---->|  OSPF inside  |
 +---------------+      +---------------+      +---------------+
```

| Point | IGP | EGP (BGP) |
|---|---|---|
| Runs | Inside one AS | Between ASes |
| Main goal | Speed and best path | Policy and control |
| Trust | High, same owner | Low, different owners |
| Routes carried | Hundreds to a few thousand, sometimes more | About 1 million IPv4 for a full table |
| Metric | Hops, cost, bandwidth and delay | Attributes such as AS path and local preference |
| Failover speed | Seconds or better | Often slower, seconds to minutes |

Two fuller truths:

- "EGP" is also the name of an old, retired protocol (RFC 904) that BGP replaced. Today the term almost always means the category.
- BGP can also run inside an AS. This is **iBGP** (internal BGP), and BGP between ASes is **eBGP** (external BGP). Large data centres even use eBGP as their only routing protocol (RFC 7938). So "BGP is only for the Internet edge" is false.

### The three families of protocols

The families differ in one main thing: what a router knows and what it tells others.

#### Distance vector

"Distance" is the metric. "Vector" means direction, which here is the next-hop neighbour. So a router knows: "network X is this far away, through this neighbour." It does not know the path beyond that neighbour.

The steps:

1. Each router starts with its directly connected networks.
2. It sends its route list to its direct neighbours.
3. It adds the cost of the link to every number it hears.
4. For each network, it keeps the lowest total.
5. It repeats. Classic RIP sends its whole table every 30 seconds.

This is the **Bellman-Ford** algorithm:

```text
best to X = lowest of (cost to neighbour + neighbour's distance to X)
```

Example with link cost 1:

| Neighbour | Neighbour says | Link cost | Total |
|---|---|---|---|
| R2 | 192.168.3.0/24 is 3 away | 1 | 4 |
| R4 | 192.168.3.0/24 is 1 away | 1 | **2 (chosen)** |

```text
 Distance vector: tell only your neighbours what you believe

 R1 <---------- R2 <---------- R3 ---- 192.168.3.0/24
     "2 away"       "1 away"
```

R1 never learns that R3 exists. It only knows "send it to R2".

Examples: RIP, and EIGRP (an advanced form).

#### Link state

"Link state" means the status of a router's links: which neighbours it has and the cost to each.

The steps:

1. Each router finds its neighbours with hello messages and forms an **adjacency**, a formal neighbour relationship.
2. Each router describes its own links in a small message. OSPF calls it an **LSA (Link State Advertisement)**. IS-IS calls it an **LSP (Link State PDU)**.
3. The message is **flooded**, which means copied to every router in the area, unchanged.
4. Every router stores all messages in a **link-state database (LSDB)**. All routers in the area hold the same LSDB.
5. Each router runs the **Dijkstra SPF (Shortest Path First)** algorithm, with itself as the root, to build a tree of best paths.
6. When something changes, only the changed LSA is flooded, and routers run SPF again.

```text
 Link state: flood your own facts to everybody

 R1's fact    "R1 links to R2, cost 1"     --> copied to R2, R3
 R2's fact    "R2 links to R1, R3, cost 1" --> copied to R1, R3
 R3's fact    "R3 links to R2, cost 1, and
               has 192.168.3.0/24"         --> copied to R1, R2

 Result: R1, R2 and R3 all hold the same map.
```

A link-state router reports only facts about itself, and it tells everyone. Distance vector is the opposite: tell only neighbours, but share everything you believe.

Examples: OSPF, IS-IS.

#### Path vector

A path vector router advertises a network plus the full list of ASes the route passed through. BGP calls this list **AS_PATH**.

```text
 AS 64514 announces 203.0.113.0/24

 AS 64514 --> AS 64513 --> AS 64512 --> AS 64515
 path:        64514        64513 64514  64512 64513 64514
```

The second row shows the path each AS receives. Loop prevention is simple. If a router in AS 64512 receives this route with path `64515 64512 64513 64514`, it sees its own AS in the list and rejects it.

BGP picks a path by policy first. Length of the AS path comes later in the decision. See [BGP](09-bgp.md) for the full process.

Example: BGP.

#### The three families compared

| Point | Distance vector | Link state | Path vector |
|---|---|---|---|
| What is shared | Route table: network and distance | Facts about own links | Network and list of ASes |
| Shared with | Direct neighbours | All routers in the area, by flooding | Configured peers |
| When | Periodic in RIP, on change in EIGRP | On change, plus slow refresh | On change |
| View of network | Next hop only | Full map of the area | AS-level path |
| Algorithm | Bellman-Ford (DUAL in EIGRP) | Dijkstra SPF | Policy-based best-path process |
| Main loop defence | Split horizon, poisoning, hold-down | Same map on all routers | AS_PATH check |
| CPU and memory | Low | Higher | Depends on table size, high for a full table |
| Protocols | RIP, EIGRP | OSPF, IS-IS | BGP |

### Convergence

A network has **converged** when all routers agree on the current network and their tables are stable. **Convergence time** is how long that takes after a change.

Analogy: after a road closes, convergence is the time until every driver has heard the news and picked a new way.

*Where the analogy stops:* a driver who has not heard the news just gets delayed. A router with old information can drop packets or send them in a circle.

Convergence has four stages:

```text
 |--- detect ---|--- tell others ---|--- compute ---|--- install ---|
 link fails     router knows        all have heard  new path found
                                                    FIB is updated
```

1. **Detect.** The router notices the failure. If the cable is cut, the interface goes down almost at once. If a neighbour dies behind a switch, the port stays up, and the router must wait for hello messages to stop arriving.
2. **Tell others.** The protocol sends updates.
3. **Compute.** Routers work out new best paths.
4. **Install.** The new route goes into the routing table, then into the FIB.

Stage 1 is often the slowest part. Common failure detection timers are:

| Protocol | Hello or update | Declared dead after | Rough convergence after detection |
|---|---|---|---|
| RIP | 30 s update | 180 s (invalid timer) | Tens of seconds to minutes |
| OSPF | 10 s hello | 40 s dead timer | Seconds, less when tuned |
| EIGRP | 5 s hello | 15 s hold timer | Under a second to seconds |
| IS-IS | 10 s hello | 30 s hold timer | Seconds, less when tuned |
| BGP | 60 s keepalive | 180 s hold timer | Tens of seconds to minutes |

These are common defaults on Cisco IOS, on fast LAN links. They differ by link type, vendor and version. BGP is a good example: RFC 4271 suggests a 90 second hold time, and Juniper uses 90 by default, while Cisco IOS uses 180. The last column is a rough guide, not a promise.

Bad things can happen before convergence finishes:

- **Black hole:** a router drops packets because it has no route, or it sends them towards a dead path.
- **Routing loop:** packets go in a circle.
- **Packet reordering or loss** while paths move.

### Routing loops

A **routing loop** happens when routers point at each other, so a packet goes round and round.

```text
 [A]--------[B]--------[C]---x--- 192.168.3.0/24 (link failed)

 B thinks: "go through C"
 C thinks: "go through B"

 Packet to 192.168.3.5:  B --> C --> B --> C --> B ...
```

The IP **TTL** (Time To Live) field stops a packet from looping forever. Each router lowers it by 1, and the packet is dropped at 0. In IPv6 this field is the Hop Limit. So TTL limits the damage but does not fix the loop. The looping packets still use bandwidth and CPU, and users get no service. See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

This is different from a Layer 2 loop. Ethernet frames have no TTL, so a switching loop can go on forever. See [Spanning Tree](../03-ethernet-and-switching/09-spanning-tree.md).

#### Count to infinity

**Count to infinity** is the classic distance vector failure. After a network dies, routers keep raising its metric in steps, believing old information from each other, until the number reaches a limit.

Use the diagram above, with no loop defences. Each router adds 1 to the number it hears. C counts its connected network as 0. These are the numbers Cisco shows for RIP.

One rule drives the problem: **a router must believe its current next hop, even when the news is worse.** It has no way to check the news.

The hop counts for 192.168.3.0/24 change like this:

| Step | What happens | A | B | C |
|---|---|---|---|---|
| 0 | Normal state | 2 via B | 1 via C | 0, connected |
| 1 | C's LAN link fails | 2 via B | 1 via C | 16, unreachable |
| 2 | B's update reaches C first: "1 away" | 2 via B | 1 via C | **2 via B** |
| 3 | C tells B "2 away" | 2 via B | **3 via C** | 2 via B |
| 4 | B tells A and C "3 away" | **4 via B** | 3 via C | **4 via B** |
| 5 | C tells B "4 away" | 4 via B | **5 via C** | 4 via B |
| 6 | B tells A and C "5 away" | **6 via B** | 5 via C | **6 via B** |
| ... | The numbers keep climbing | ... | ... | ... |
| 16 | B tells A and C "15 away" | **16** | 15 via C | **16** |
| 17 | C tells B "16 away" | 16 | **16** | 16 |

Step 2 is the mistake. C hears "1 away" from B and thinks B has another path. But B's path goes through C.

RIP solves the endless count with a small ceiling: **16 means unreachable**, so the largest usable path is 15 hops. This is why RIP cannot serve big networks. The ceiling ends the problem but does not make it fast. Each step can wait up to 30 seconds for the next update, so the loop can last for minutes.

Analogy: you ask a friend the way to a shop that has closed. The friend says "2 km, I asked you yesterday". You now say "3 km, my friend knows". Each of you trusts the other, and the shop is still closed.

*Where the analogy stops:* a person might get suspicious. A basic router has no memory of where the information began.

### Loop defences

Each defence below is a small rule. Real distance vector protocols use several together.

#### Split horizon

**Rule:** do not advertise a route back out of the interface you learned it from.

```text
 [A]--------[B]--------[C]---- 192.168.3.0/24

 C --> B : "192.168.3.0/24, 1 away"   allowed
 B --> A : "192.168.3.0/24, 1 away"   allowed
 B --> C : nothing about this route   blocked by split horizon
```

This removes step 2 of the table above. C never hears "1 away" from B.

Analogy: do not give directions back to the person who just gave them to you.

*Where the analogy stops:* split horizon is per interface and per route. It blocks only the direct link, so it cannot stop a loop of three or more routers. [Going Deeper](#why-split-horizon-is-not-enough) shows this case.

#### Route poisoning

**Rule:** when a route dies, advertise it with an unreachable metric instead of staying silent. In RIP this metric is 16.

C sets 192.168.3.0/24 to 16 and tells B. B tells A. Everyone removes the route quickly, without waiting for it to time out.

Analogy: put up a "road closed" sign rather than removing the old sign and hoping drivers work it out.

#### Poison reverse

**Rule:** a router that learned a route from a neighbour sends that route back to the neighbour with metric unreachable.

```text
 B --> C : "192.168.3.0/24, 16"  meaning "do not use me for this"
```

It is stronger than split horizon because it is an explicit message, not silence. The cost is larger updates.

Do not mix these up:

| Technique | Who sends what | When |
|---|---|---|
| Split horizon | Sends nothing back towards the source | Always |
| Route poisoning | The router that sees the failure sends metric 16 | When a route fails |
| Poison reverse | The downstream router sends metric 16 back towards the source | Always (or after a poison, on some implementations) |

The fuller truth: many courses use the words loosely. In Cisco's RIP, for example, a router that hears a poison may send it back, which is called "poison reverse" there too. RFC 2453 describes the always-on form as "split horizon with poisoned reverse". Explain what you mean in an interview, and you will be safe.

#### Triggered updates

**Rule:** when a route changes, send an update now. Do not wait for the next 30 second timer. This reduces the window in which bad news and old news can cross each other. It does not close the window: an old update may already be on its way.

#### Hold-down timer

**Rule:** after a route goes bad, ignore news about it that is not better than the old route, for a fixed time.

The router uses this time to let the bad news spread everywhere. Without it, old good news from a slow router could bring a dead route back.

Analogy: after you hear a shop has closed, you ignore rumours that it is open for a few minutes, unless you hear strong proof.

*Where the analogy stops:* the wait costs something real. If the link comes back through another path with a worse metric, that path is ignored until the timer ends. Hold-down can slow recovery.

The RFC for RIP has no hold-down timer. It has timeout (180 seconds) and garbage collection (120 seconds) timers. Hold-down is a vendor feature. Cisco's RIP defaults are:

| Timer | Default | Meaning |
|---|---|---|
| Update | 30 s | Send full table |
| Invalid | 180 s | No update heard, mark route invalid |
| Hold-down | 180 s | Ignore news that is not better |
| Flush | 240 s | Remove route from the table |

#### How the other families avoid these problems

TTL and a maximum metric, such as RIP's 16, are only last-line defences. They limit a loop's damage but do not prevent it. The other families avoid the problem by design.

- **Link state:** a router does not believe neighbours' conclusions. Every router computes from the same facts, so count to infinity cannot happen inside an area. Small temporary loops can still occur during flooding (see [micro-loops](#micro-loops)).
- **Path vector:** the AS_PATH check rejects a route that contains your own AS.
- **EIGRP:** the DUAL algorithm accepts a backup path only if it is proven loop-free.

### Protocol comparison table

The first table shows design. The second shows practical details.

| Protocol | Type | Scope | Algorithm | Metric | Max size | Status |
|---|---|---|---|---|---|---|
| RIPv2 | Distance vector | IGP | Bellman-Ford | Hop count | 15 hops | Legacy |
| EIGRP | Advanced distance vector | IGP | DUAL | Bandwidth and delay by default | Large | Mostly Cisco networks |
| OSPF | Link state | IGP | Dijkstra SPF | Cost, from bandwidth on Cisco | Large, with areas | Most common enterprise IGP |
| IS-IS | Link state | IGP | Dijkstra SPF | Cost, default 10 per link on Cisco | Very large | Common with ISPs and large providers |
| BGP | Path vector | EGP, also used inside | Best-path process | Attributes, not a single metric | Internet scale | Runs the Internet |

| Protocol | Carried over | Multicast group (IPv4) | Cisco AD | Standard |
|---|---|---|---|---|
| RIPv2 | UDP port 520 | 224.0.0.9 | 120 | RFC 2453 |
| EIGRP | IP protocol 88 | 224.0.0.10 | 90 internal, 170 external | RFC 7868 (informational, Cisco origin) |
| OSPFv2 | IP protocol 89 | 224.0.0.5, 224.0.0.6 | 110 | RFC 2328 |
| IS-IS | Directly on Layer 2, not in IP | Layer 2 multicast | 115 | ISO/IEC 10589, RFC 1195 for IP |
| BGP-4 | TCP port 179 | None, unicast sessions | 20 eBGP, 200 iBGP | RFC 4271 |

**AD (Administrative Distance)** is Cisco's trust score for a route source. Lower is more trusted. AD compares different protocols, and metric compares paths within one protocol. Other vendors use other numbers. On Juniper, for example, OSPF internal routes have preference 10 and BGP has 170.

The order of choice for a packet is covered in [The Routing Table](02-routing-table.md): longest prefix first, then AD, then metric.

### Which one to use

| Situation | Common choice | Why |
|---|---|---|
| Multi-vendor enterprise | OSPF | Open standard, widely known |
| Cisco-only enterprise | OSPF or EIGRP | EIGRP is simple and fast, but ties you to Cisco in practice |
| ISP core | IS-IS, with BGP on top | Scales well, easy to extend |
| Large data centre | eBGP | Simple failure behaviour, strong policy (RFC 7938) |
| Connecting to ISPs or cloud | BGP | The only choice between ASes |
| Tiny site with one exit | Static default route | No protocol needed |
| New network | Not RIP | Slow and limited to 15 hops |

## Example

### Example 1: OSPF heals a broken link

Three Cisco routers form a triangle. All links are Gigabit Ethernet. The goal: R1's LAN reaches R3's LAN, and a failed link is routed around automatically.

```text
  192.168.1.0/24                            192.168.3.0/24
        |                                          |
     G0/0 .1                                    G0/0 .1
    +--------+ G0/1 .1  10.0.13.0/30  G0/1 .2 +--------+
    |   R1   |--------------------------------|   R3   |
    +--------+                                +--------+
     G0/2 .1                                   G0/2 .2
        \                                        /
    10.0.12.0/30                         10.0.23.0/30
          \                                    /
        G0/0 .2                            G0/1 .1
            \           +--------+           /
             +----------|   R2   |----------+
                        +--------+
```

**Step 1: configure OSPF (Cisco IOS).** The interface addresses are already set. Each `network` line says which interfaces join OSPF. `0.0.0.3` is a wildcard mask, the inverse of a /30 mask. `passive-interface` means: include this LAN in OSPF but send no hellos there, because no routers live on it.

```text
! R1
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.13.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0

! R2
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0

! R3
router ospf 1
 router-id 3.3.3.3
 network 192.168.3.0 0.0.0.255 area 0
 network 10.0.13.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 passive-interface GigabitEthernet0/0
```

Compare this to static routing: no router line names a remote network. R1 is never told about 192.168.3.0/24. It learns this.

**Step 2: check the neighbours.**

```text
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3           1   FULL/DR         00:00:34    10.0.13.2       GigabitEthernet0/1
2.2.2.2           1   FULL/DR         00:00:37    10.0.12.2       GigabitEthernet0/2
```

`FULL` means the two routers have the same LSDB. If you see no neighbours, check that the link is up and the area, timers and subnet match. DR is the Designated Router, covered in [OSPF](06-ospf.md).

**Step 3: read the routes.**

```text
R1# show ip route ospf

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
O        10.0.23.0/30 [110/2] via 10.0.13.2, 00:03:11, GigabitEthernet0/1
                      [110/2] via 10.0.12.2, 00:03:11, GigabitEthernet0/2
O     192.168.3.0/24 [110/2] via 10.0.13.2, 00:03:11, GigabitEthernet0/1
```

How to read the last line:

- `O` means learned from OSPF.
- `[110/2]` means AD 110 and metric 2.
- `via 10.0.13.2` is the next hop, R3.

Why metric 2? Each Gigabit link has cost 1 by default here. To reach R3's LAN: link to R3 (1) plus R3's LAN interface (1) equals 2. Through R2 it would be 3, so that path loses.

10.0.23.0/30 has two equal-cost paths, and both are installed. This is **ECMP (Equal-Cost Multi-Path)**: traffic is shared across both.

**Step 4: test.**

```text
R1# ping 192.168.3.1 source 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

**Step 5: cut the R1-R3 cable.** R1 sees the interface go down and drops the neighbour straight away.

```text
*Oct 30 10:15:02.114: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
*Oct 30 10:15:02.118: %OSPF-5-ADJCHG: Process 1, Nbr 3.3.3.3 on GigabitEthernet0/1 from FULL to DOWN, Neighbor Down: Interface down or detached
```

R1 and R3 flood new LSAs. All routers run SPF again. Within a few seconds at most, the table changes. The exact time depends on platform and timers.

```text
R1# show ip route ospf

      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O        10.0.23.0/30 [110/2] via 10.0.12.2, 00:00:04, GigabitEthernet0/2
O     192.168.3.0/24 [110/3] via 10.0.12.2, 00:00:04, GigabitEthernet0/2
```

The metric went from 2 to 3, and the next hop is now R2. No one touched a router. With static routes and no backup, R3's LAN would now be unreachable from R1.

If R1 and R3 were joined through a switch, and R3 crashed, R1's port would stay up. R1 would wait for the 40 second dead timer. That is a 40 second outage, and it is why BFD exists. See [fast failure detection](#fast-failure-detection).

**Seeing the route source on a host.** End hosts usually do not run routing protocols. They use a default gateway, and the routing table shows that.

```bash
# Linux: the "proto" field shows where a route came from
ip route show default
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.50 metric 100
```

```powershell
# Windows: the Protocol column shows where a route came from
Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object DestinationPrefix, NextHop, Protocol

DestinationPrefix NextHop     Protocol
----------------- -------     --------
0.0.0.0/0         192.168.1.1 Dhcp
```

A Linux server can act as a router with software such as FRRouting. Its OSPF routes then show `proto ospf` in `ip route`.

### Example 2: hop count vs cost

Two protocols can choose different paths through the same network. R1 wants to reach R2.

```text
                R1 --- 10 Mbps, cost 10 --- R2
                |                           |
            100 Mbps, cost 1            100 Mbps, cost 1
                |                           |
                R3 --- 100 Mbps, cost 1 --- R4
```

Costs here use OSPF's classic formula on Cisco: reference bandwidth 100 Mbps divided by link speed.

**RIP** counts hops. Direct is 1 hop, and the way through R3 and R4 is 3 hops. RIP uses the slow 10 Mbps link.

**OSPF** runs SPF from R1. It keeps a "tree" of settled routers and a list of candidates:

| Step | Added to tree | Candidates after this step |
|---|---|---|
| 1 | R1 (0) | R3 = 1, R2 = 10 |
| 2 | R3 (1) | R4 = 1 + 1 = 2, R2 = 10 |
| 3 | R4 (2) | R2 = 2 + 1 = 3, better than 10 |
| 4 | R2 (3) | None, finished |

Result: R1 reaches R2 through R3 and R4, at cost 3. All three destinations use next hop R3. The longer path is ten times faster.

A warning. The old 100 Mbps reference gives 100 Mbps, 1 Gbps and 10 Gbps links all the same cost of 1. Change the reference bandwidth on every router in modern networks. [OSPF](06-ospf.md) shows how.

## Going Deeper

### Why split horizon is not enough

Split horizon blocks only a direct link. It fails in a triangle.

```text
        [A]-----------[B]
          \           /
           \         /
            \       /
              [C]----x---- 192.168.3.0/24
```

Before the failure, A and B both reach the LAN through C at metric 1. A also tells B "1 away" (A's next hop is C, not B, so split horizon allows it). B tells A the same.

1. The LAN link fails. C sends a poison to A and B.
2. B gets it. The copy to A is lost.
3. A's regular update reaches B: "1 away". B accepts: 2 via A.
4. B tells C "2 away". Split horizon blocks only the link to A. C accepts: 3 via B.
5. C tells A "3 away". A's next hop is C, so A must accept: 4 via C.
6. A tells B "4 away". B's next hop is A, so B accepts: 5 via A.
7. The loop A to C to B to A keeps counting until 16.

A hold-down timer stops this at step 3: B ignores A's news because it is not better than the old route. This is why distance vector protocols stack several defences.

### DUAL, and is EIGRP a hybrid?

EIGRP is a distance vector protocol that keeps extra information. It remembers what every neighbour reported, in a **topology table**.

Terms:

- **Feasible distance:** this router's best total metric to the destination.
- **Reported distance:** the metric a neighbour says it has.
- **Successor:** the best next hop.
- **Feasible successor:** a backup neighbour that passes the **feasibility condition**: its reported distance is lower than this router's feasible distance.

Why does that test work? If a neighbour is strictly closer to the destination than you are, its path cannot go through you. So it is loop-free. When the successor fails, EIGRP can switch to the feasible successor at once, with no recomputation. If there is no such backup, the route goes **active**, and the router sends queries to its neighbours.

This is the **DUAL** (Diffusing Update Algorithm). See [EIGRP](07-eigrp.md).

Older Cisco material called EIGRP "hybrid". It sends updates only on change and uses hellos, like link state. But it does not flood a map or run SPF, and routers still share distances with neighbours. The correct modern description is **advanced distance vector**.

### Micro-loops

Link-state routers do not update at the same instant. For a short time, R1 may use the new map while R2 uses the old map. They can then point at each other for milliseconds to seconds. This is a **micro-loop**. It ends by itself when convergence finishes. This is why "link state has no loops" is only true for stable state.

### Fast failure detection

Hello timers are too slow for many networks, and tuning them too low adds CPU load.

**BFD (Bidirectional Forwarding Detection, RFC 5880)** is a very light protocol that checks whether the path to a neighbour works. It is often run in hardware, with timers such as 50 ms x 3, which detects failure in about 150 ms. OSPF, IS-IS, EIGRP and BGP can all register with BFD and react to its signal.

Other techniques:

- **Fast reroute:** a pre-computed backup path is already in the FIB, so the router switches in tens of milliseconds. **LFA (Loop-Free Alternate, RFC 5286)** is one common form. [Advanced Routing Techniques](12-advanced-routing-techniques.md) has more.
- **Graceful restart:** when a router's control plane restarts, neighbours keep forwarding through it for a short time, so a software restart does not cause an outage.
- **SPF and LSA throttling:** during a flapping link, a router waits longer between SPF runs. This is a modern cousin of hold-down: slow down to stay stable.

A **flap** is a link or route going up and down repeatedly. Flaps create constant recalculation, and every protocol has ways to limit the effect.

### Scaling link state with areas

An LSDB grows with the network, and SPF runs over all of it. So OSPF and IS-IS split a network into **areas**. Routers know the full map of their own area. Between areas they get summaries.

The fuller truth: between areas, OSPF acts somewhat like distance vector, because a router does not see inside other areas. So OSPF requires all areas to connect through backbone area 0, which avoids loops between areas. [OSPF](06-ospf.md) explains this.

### Loops inside BGP

The AS_PATH check works only between ASes. Inside an AS the path does not change, so it cannot detect a loop. iBGP uses another rule: **a route learned from an iBGP peer is not passed to another iBGP peer.** This is like split horizon. As a result, iBGP needs a full mesh or route reflectors. See [BGP](09-bgp.md).

### Classful and classless protocols

A **classful** protocol does not send the subnet mask with a route. RIPv1 and IGRP are classful, so they cannot support [VLSM](../04-ip-addressing/07-vlsm-and-supernetting.md). A **classless** protocol sends the mask. RIPv2, OSPF, EIGRP, IS-IS and BGP are all classless.

### IPv6 versions

| IPv4 protocol | IPv6 version |
|---|---|
| RIPv2 | RIPng (RFC 2080), UDP port 521 |
| OSPFv2 | OSPFv3 (RFC 5340) |
| EIGRP | EIGRP for IPv6, same protocol |
| IS-IS | Same protocol, extended (RFC 5308) |
| BGP-4 | Multiprotocol BGP (RFC 4760) |

### Redistribution, security and central control

- **Redistribution** copies routes from one protocol into another, for example after a merger. A route can leave one protocol and return through another, so it can create loops. See [Advanced Routing Techniques](12-advanced-routing-techniques.md).
- **Security:** a rogue router on a LAN could advertise false routes. Use `passive-interface` on user-facing ports, turn on protocol authentication, and filter what you accept. RPKI for the Internet is in [BGP](09-bgp.md).
- **Central control:** [SDN](../10-enterprise-and-datacenter/09-software-defined-networking.md) moves decisions to a controller. Distributed protocols remain the base of the Internet.

## Common Mistakes

- **"OSPF is a routed protocol."** OSPF is a routing protocol. IP is the routed protocol.
- **"Fewest hops is always the best path."** A one-hop 10 Mbps link can be far slower than three 100 Mbps hops.
- **"AD and metric mean the same thing."** AD compares sources, is local, and is never advertised. Metric compares paths within one source.
- **"BGP runs only between ASes."** iBGP runs inside an AS, and many data centres use BGP as their main protocol.
- **"Split horizon stops all loops."** It only blocks the direct link. Loops of three or more routers can still form.
- **"Route poisoning and poison reverse are the same."** Poisoning announces a dead route. Poison reverse sends a route back with an unreachable metric.
- **"Link-state protocols never loop."** Micro-loops can appear during convergence.
- **"Hold-down is in the RIP standard."** It is a vendor feature. The RFC defines timeout and garbage collection timers.
- **"RIP's limit is 16 hops."** The limit is 15. 16 means unreachable.
- **"Convergence time is just SPF time."** Detection is often the largest part.
- **"TTL prevents routing loops."** TTL only stops a single packet from looping forever.
- **"Dynamic routing replaces static routes."** Most networks use both.

## Interview Questions

**Q: Why use dynamic routing instead of static routes?**

A: Static routes need manual work on every router and do not react to failures without extra setup. Dynamic routing learns routes automatically, adapts when links fail, and scales to large networks. The cost is some CPU, memory and bandwidth, plus more complexity. In practice, I use both: static or default routes at simple edges, and a dynamic protocol in the core.

**Q: What is an autonomous system?**

A: An AS is a group of networks under one administration with one routing policy. Each public AS has a unique number, assigned by a Regional Internet Registry. ASNs were 16-bit and are now 32-bit. The private ranges are 64512-65534 and 4200000000-4294967294.

**Q: What is the difference between an IGP and an EGP?**

A: An IGP runs inside one AS and focuses on fast, best-path routing. RIP, OSPF, EIGRP and IS-IS are IGPs. An EGP runs between ASes and focuses on policy. BGP is the EGP used today.

**Q: Compare distance vector and link state.**

A: A distance vector router tells its neighbours its distances to networks and knows only the next hop. It uses Bellman-Ford. A link-state router floods facts about its own links to all routers in the area. Each one builds the same map and runs Dijkstra. Link state needs more CPU and memory, but it converges faster and does not suffer from count to infinity.

**Q: Why is BGP called a path vector protocol?**

A: Each BGP route carries AS_PATH, the list of ASes it passed through. So a router sees the path at AS level, not just a distance. It uses this list to detect loops and to apply policy.

**Q: What is convergence, and what affects it?**

A: Convergence is when all routers agree on the network and their tables are stable after a change. It has four stages: detect the failure, spread the news, compute new paths, and install them in the FIB. Timers, network size and protocol type all affect it. Detection is often the slowest stage, which is why BFD is used.

**Q: What is count to infinity, and why is RIP limited to 15 hops?**

A: After a network fails, two distance vector routers can keep learning it from each other, and each one raises the metric step by step. RIP treats 16 as unreachable, so the counting stops there. The low limit keeps the counting short, but it means RIP paths can have at most 15 hops.

**Q: Explain split horizon, route poisoning and poison reverse.**

A: Split horizon means a router does not advertise a route back out of the interface it learned it from. Route poisoning means a router advertises a failed route with an unreachable metric, 16 in RIP, so others remove it quickly. Poison reverse means a router sends a learned route back to its source with an unreachable metric, as a clear signal that it is not a valid path.

**Q: What does a hold-down timer do, and what is its downside?**

A: After a route goes bad, the router ignores news about it that is not better, for a fixed time. Cisco RIP uses 180 seconds. This gives the bad news time to spread and stops old information from bringing back a dead route. The downside is slower recovery, because a valid alternative path may be ignored until the timer ends.

**Q: How do link-state protocols and BGP avoid routing loops?**

A: In link state, all routers compute from the same database, so stable tables agree with each other. Micro-loops can still appear briefly during convergence. BGP rejects any route whose AS_PATH contains its own AS. Inside an AS, iBGP does not pass a route from one iBGP peer to another.

**Q: Is EIGRP a hybrid protocol?**

A: Older material says so, but it is more accurate to say advanced distance vector. It shares distances with neighbours, not a map. It adds DUAL, which keeps loop-free backups called feasible successors, so it can fail over very quickly.

**Q: A router learns the same prefix from OSPF and from RIP. Which route is used?**

A: With the same prefix length, it picks the lower administrative distance. On Cisco, OSPF is 110 and RIP is 120, so OSPF wins. Metrics are not compared across protocols. A longer prefix from RIP would still win for matching packets, because longest match comes first.

## Quick Recap

- Dynamic routing protocols find neighbours, share routes, choose best paths and react to change automatically.
- An AS is a set of networks under one policy. IGPs run inside an AS, and BGP runs between ASes.
- Distance vector shares distances with neighbours, link state floods link facts and builds a map, and path vector shares a list of ASes.
- Convergence means detect, tell, compute, install. Detection is often the slowest step.
- Count to infinity is the classic distance vector failure, and RIP stops it at 16.
- Defences include split horizon, route poisoning, poison reverse, triggered updates and hold-down. TTL only limits the damage.
- OSPF and IS-IS are common IGPs, EIGRP is mostly used on Cisco networks, RIP is legacy, and BGP runs the Internet.
