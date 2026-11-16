# OSPF

**Level:** Advanced | **Prerequisites:** [The Routing Table](02-routing-table.md), [Dynamic Routing Overview](04-dynamic-routing-overview.md), [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md), [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md)

## In One Line

OSPF (Open Shortest Path First) is a link-state routing protocol. Every router in an area keeps the same map of the network and uses that map to calculate the shortest path to every destination.

## Why It Matters

Imagine a company with 200 routers. With static routes, an engineer must type every path by hand. When a link fails, traffic stops until a person fixes it.

RIP was an early answer, but it has hard limits. It counts hops and ignores speed, so it prefers a 2-hop slow link over a 3-hop fast path. It also stops at 15 hops and reacts slowly. See [RIP](05-rip.md).

OSPF solves these problems:

- **Fast reaction.** When a link fails, routers tell each other at once and recalculate. This process of all routers agreeing on the new state is called **convergence**.
- **Speed-aware paths.** OSPF uses a **cost**, a number based on link bandwidth, so it prefers fast links.
- **Growth.** Areas split a big network into smaller parts, so no single router needs every detail.
- **Open standard.** "Open" means the design is public. Cisco, Juniper, Arista, Nokia and Linux routers can all run it together. OSPFv2 for IPv4 is defined in RFC 2328.

OSPF is an **IGP** (Interior Gateway Protocol): it routes inside one organisation. Between organisations, the internet uses [BGP](09-bgp.md).

Interviewers love OSPF because it tests many skills at once: protocol states, design, troubleshooting and Cisco commands. Typical questions are "why is my neighbour stuck in ExStart?" and "what is the difference between a Type 3 and a Type 5 LSA?".

## Simple Explanation

Think about how you find your way in a new city.

**Way 1: signposts.** At each junction, a sign says "City centre, 5 km, go left". You trust the sign. You do not know the roads beyond it. This is how distance-vector protocols such as RIP work. Routers share only "distance and direction".

**Way 2: a full map.** Everyone holds the same printed map. You mark where you are and work out your own best route. If a road closes, a message goes out to everyone: "Road 7 is closed". Each person corrects their map and works out a new route. This is how OSPF works.

A country map with every small street would be huge. So the map is split into **areas**, like districts. You hold a detailed map of your own district. For other districts you only know "these places are that way, through this border".

**Where the analogy stops being true:** a person chooses a route once for a trip. An OSPF router runs the calculation for all destinations at the same time. It then decides hop by hop, because each router only picks the next router, not the whole path. Also, nobody prints or sends the map from a central office. Each router describes only its own links, and the routers together build the map.

## How It Works

### 1. The Link-State Idea

A **link** is a router interface and what it connects to. Its **state** is the facts about it: address, mask, cost and neighbour. OSPF works in five steps:

1. Each router finds its neighbours by sending Hello packets.
2. Each router describes its own links in a record called an **LSA** (Link-State Advertisement).
3. LSAs are copied to all routers in the area. This copying is called **flooding**.
4. Every router stores all LSAs in its **LSDB** (Link-State Database). Inside one area, all routers have an identical LSDB.
5. Each router runs the **SPF** (Shortest Path First) algorithm on the LSDB, with itself as the starting point. The best results go into the routing table.

Same map, but different view. R1 and R2 hold the same LSDB, yet their routing tables differ because each one calculates from its own position.

```text
   Hello --> Neighbours --> LSAs flooded --> LSDB --> SPF --> Routing
                                           (same in    (own    table
                                            the area)  view)
```

### 2. OSPF Packets and How They Travel

OSPF does not use TCP or UDP. It sits directly on IP as **IP protocol 89**. It builds its own reliability with acknowledgements and retransmission.

Most OSPF packets go to one of two multicast addresses. **Multicast** means one sender and a chosen group of receivers. These packets have a TTL of 1, so they stay on the local link.

| Address | Name | Who listens |
|---|---|---|
| 224.0.0.5 | AllSPFRouters | All OSPF routers |
| 224.0.0.6 | AllDRouters | Only the DR and BDR (explained in section 6) |

There are five packet types:

| Type | Name | Job | Everyday meaning |
|---|---|---|---|
| 1 | Hello | Find and keep neighbours | "Are you there?" |
| 2 | DBD (Database Description) | List the LSAs I have | "Here is my table of contents" |
| 3 | LSR (Link-State Request) | Ask for missing LSAs | "Please send me page 7" |
| 4 | LSU (Link-State Update) | Carry the LSAs | "Here is page 7" |
| 5 | LSAck | Confirm delivery | "Got it" |

Every OSPFv2 packet starts with a 24-byte header. A Hello adds these fields after the header:

```text
 0               8               16              24            31
+---------------+---------------+-------------------------------+
| Version = 2   | Type = 1      | Packet length                 |
+---------------+---------------+-------------------------------+
| Router ID                                                     |
+---------------------------------------------------------------+
| Area ID                                                       |
+-------------------------------+-------------------------------+
| Checksum                      | Authentication type           |
+-------------------------------+-------------------------------+
| Authentication data (8 bytes)                                 |
+---------------------------------------------------------------+
| Network mask                                                  |
+-------------------------------+---------------+---------------+
| Hello interval                | Options       | Priority      |
+-------------------------------+---------------+---------------+
| Dead interval                                                 |
+---------------------------------------------------------------+
| Designated Router (DR)                                        |
+---------------------------------------------------------------+
| Backup Designated Router (BDR)                                |
+---------------------------------------------------------------+
| Neighbour router ID (repeats, 4 bytes for each neighbour)     |
+---------------------------------------------------------------+
```

### 3. Router ID

The **router ID** (RID) is the name of a router inside OSPF. It is a 32-bit number written like an IPv4 address, for example 10.255.0.1.

**Analogy:** it is a staff badge number. The badge identifies you, wherever you sit in the building. **Where it stops being true:** a badge is issued by a central office, while a router chooses its own RID, so two routers can pick the same one by mistake.

Cisco IOS picks the RID in this order when the OSPF process starts:

1. The value set with the `router-id` command.
2. The highest IP address on an active loopback interface.
3. The highest IP address on an active physical interface.

If none of these exists, the process cannot start. A **loopback** is a virtual interface that stays up as long as the router is up.

```text
! Cisco IOS
router ospf 1
 router-id 10.255.0.1
```

Key facts:

- The RID only needs to be unique. It does not have to be a real or reachable address.
- A new RID does not apply until the OSPF process restarts with `clear ip ospf process`. This drops all neighbours, so do it in a maintenance window.
- Two routers with the same RID cause broken adjacencies or unstable routes.
- Always set the RID by hand. If it depends on an interface, it can change after a reboot.

### 4. Hello and Dead Timers

**Hello interval:** how often a router sends a Hello.

**Dead interval:** how long a router waits without a Hello before it declares the neighbour dead.

**Analogy:** two neighbours wave at each other every morning. If one has not waved for four days, the other assumes something is wrong. **Where it stops being true:** when a router interface goes physically down, OSPF reacts at once. It does not wait for the dead timer. The dead timer matters when the path fails silently, such as a failure behind a switch.

| Network type | Hello | Dead |
|---|---|---|
| Broadcast, point-to-point | 10 s | 40 s |
| NBMA, point-to-multipoint | 30 s | 120 s |

These are the Cisco defaults. RFC 2328 shows them only as sample values, so other vendors can differ, although most use the same numbers.

On Cisco, if you change the hello interval and do not set dead, dead becomes four times hello. Both ends must match exactly:

```text
! Cisco IOS, interface mode
interface GigabitEthernet0/0
 ip ospf hello-interval 5
 ip ospf dead-interval 20
```

### 5. Neighbour States from Down to Full

A **neighbour** is a router you have seen in Hello packets on a shared link. An **adjacency** is stronger: a neighbour you have fully synchronised your LSDB with. All adjacent routers are neighbours, but not all neighbours are adjacent.

**Analogy:** two librarians meet. First they check that they can hear each other. Then they decide who leads. They swap catalogues, order the books they lack, and finish with identical libraries. **Where it stops being true:** librarians might swap once, but routers keep their databases in sync for as long as the link is up.

| State | Meaning |
|---|---|
| Down | No Hello received recently. |
| Attempt | NBMA networks only. Sending unicast Hellos to a configured neighbour, no reply yet. |
| Init | I received your Hello, but my RID is not in your neighbour list. |
| 2-Way | My RID is in your Hello, so traffic works in both directions. DR election happens here. |
| ExStart | We choose master and slave. The higher RID becomes master. The master controls the sequence numbers. |
| Exchange | We send DBD packets, which list our LSA headers. |
| Loading | We use LSR to ask for missing LSAs, LSU to deliver them, and LSAck to confirm them. |
| Full | Our LSDBs match. We are adjacent. |

Here is the whole process on a point-to-point link:

```text
 R1 (RID 10.255.0.1)                    R2 (RID 10.255.0.2)
 Down                                   Down
  |--- Hello (seen: nobody) -------------->|  R2: Init
  |<-- Hello (seen: 10.255.0.1) -----------|  R1: 2-Way
  |--- Hello (seen: 10.255.0.2) ---------->|  R2: 2-Way
  |--- DBD (I claim master) -------------->|  ExStart
  |<-- DBD (I claim master) ---------------|  R2 wins: higher RID
  |--- DBD (my LSA headers) -------------->|  Exchange
  |<-- DBD (my LSA headers) ---------------|
  |--- LSR (send me LSA X) --------------->|  R1: Loading
  |<-- LSU (here is LSA X) ----------------|
  |--- LSAck (got it) -------------------->|
 Full                                   Full
```

**Fuller truth:** the DBD also carries the interface MTU. **MTU** (Maximum Transmission Unit) is the largest IP packet the interface can send. This is why an MTU mismatch shows up here and not at Hello time.

### 6. DR and BDR Election

On an Ethernet LAN, many routers share one segment. If all pairs formed adjacencies, the count would be `n(n-1)/2`. Ten routers would need 45 adjacencies, and one LSA would be flooded many times.

OSPF fixes this with two special roles:

- **DR** (Designated Router): the central point for flooding on the segment.
- **BDR** (Backup Designated Router): a ready replacement. It stays in sync and takes over if the DR dies.

All other routers are called **DROTHER** (a Cisco term). They go Full only with the DR and BDR. Two DROTHERs stop at 2-Way, and this is normal.

**Analogy:** in a meeting of ten people, nobody tells each person every update one by one. Everyone tells the secretary, and the secretary announces it to the room. **Where it stops being true:** the DR forwards no user traffic. Packets still go straight from router to router. The DR is only a hub for OSPF database updates.

The flood works like this:

1. A DROTHER has a new LSA. It sends the LSU to 224.0.0.6, reaching the DR and BDR.
2. The DR sends the LSU to 224.0.0.5, reaching everyone.
3. The other routers acknowledge it.

The DR also creates the Type 2 LSA for the segment (see section 8).

**Election rules:**

1. Highest interface priority wins. The range is 0-255, and the default is 1 on Cisco.
2. If priorities tie, the highest RID wins.
3. Priority 0 means "never become DR or BDR".
4. The election does **not** preempt. **Preempt** means taking over from a current holder. A new router with a better priority waits until the DR fails.
5. The election is per segment, not per area or per router. One router can be DR on one LAN and DROTHER on another.

**Worked example:** four routers share LAN 10.1.1.0/24.

```text
             Switch (10.1.1.0/24)
   +-----------+---------+-----------+
   |           |         |           |
  R1          R2        R3          R4
  .1          .2        .3          .4
  Pri 1       Pri 1     Pri 100     Pri 0
  RID .1.1    RID .1.2  RID .1.3    RID .1.4
  DROTHER     BDR       DR          DROTHER

  (RIDs are 10.255.1.1 to 10.255.1.4)
```

R3 wins DR on priority. R2 wins BDR because it has the highest RID among the remaining candidates. R4 has the highest RID of all, but priority 0 removes it from the election.

```text
R1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address      Interface
10.255.1.2        1   FULL/BDR        00:00:38    10.1.1.2     Gi0/0
10.255.1.3      100   FULL/DR         00:00:36    10.1.1.3     Gi0/0
10.255.1.4        0   2WAY/DROTHER    00:00:33    10.1.1.4     Gi0/0
```

R1 is Full with the DR and BDR, and 2-Way with the other DROTHER. This is healthy. To set priority on Cisco:

```text
interface GigabitEthernet0/0
 ip ospf priority 100
```

**Fuller truth:** a router on a new segment first waits for the Wait timer, which equals the dead interval (40 s by default), in case a DR already exists. So on a broadcast link, a new router can wait about 40 seconds before Full. RFC 2328 elects the BDR first and then the DR. If the DR dies, the BDR takes over and a new BDR is elected.

### 7. Areas and Router Roles

An **area** is a group of routers and links that share one LSDB. Every interface belongs to exactly one area. An area ID is a 32-bit number, so `area 10` and `area 0.0.0.10` mean the same thing.

Why split a network?

- Each area has a smaller LSDB, which uses less memory.
- A link flap in one area does not make routers in other areas run a full SPF.
- Flooding stays inside the area.
- Summarisation is possible at area borders.

**Analogy:** a country has regions. Every long trip goes through the central motorway. **Where it stops being true:** a real motorway is a physical thing, but area 0 is only a logical label on routers and links.

**Area 0** is the **backbone**. All other areas must connect to it. Inter-area traffic passes through area 0. This rule gives a simple hub-and-spoke shape, which prevents routing loops between areas.

```text
        +-----------------+
        |     Area 0      |
        |   (backbone)    |
        +--+-----------+--+
           |           |
         [ABR]       [ABR]
           |           |
     +-----+----+ +----+-----+
     | Area 10  | | Area 20  |--[ASBR]-- BGP, static,
     +----------+ +----------+            other routing
```

| Router role | Meaning |
|---|---|
| Internal router | All interfaces are in one area. |
| Backbone router | At least one interface is in area 0. |
| ABR (Area Border Router) | Connects area 0 to another area, and keeps one LSDB per area. |
| ASBR (Autonomous System Boundary Router) | Brings external routes into OSPF. |

An **autonomous system** (AS) here means the whole network under one routing control. An external route is any route learned from outside OSPF, such as static routes or BGP. Bringing such routes in is called **redistribution**. See [Advanced Routing Techniques](12-advanced-routing-techniques.md).

A router can hold more than one role. An ABR is also a backbone router, and it can be an ASBR too.

**Fuller truth:** on Cisco, a router with interfaces in areas 10 and 20 but none in area 0 does not behave as a full ABR. It will not pass routes between those areas. RFC 3509 describes this alternative ABR behaviour.

### 8. LSA Types

An LSA is one record in the database. OSPF has several types because different parts of the network need different levels of detail.

**Analogy:** a Type 1 is one house describing its own front gate. A Type 2 is a notice for one shared street, listing the houses on it. A Type 3 is a border sign saying "town X is that way". A Type 4 says how to reach a port, and a Type 5 says "overseas city Y is reached through that port". **Where it stops being true:** real signs stay still, while LSAs age out and are refreshed.

| Type | Name | Created by | Flooded to | What it says |
|---|---|---|---|---|
| 1 | Router | Every router | Its own area | My links, costs and neighbours |
| 2 | Network | DR | Its own area | Routers attached to this LAN |
| 3 | Network Summary | ABR | Other areas | How to reach a network in another area |
| 4 | ASBR Summary | ABR | Other areas | How to reach an ASBR |
| 5 | AS External | ASBR | All normal areas | How to reach an external network |
| 7 | NSSA External | ASBR in NSSA | That NSSA only | External network, inside an NSSA |

The less common types are:

- **Type 6:** Multicast OSPF. Obsolete, and Cisco never supported it.
- **Type 8:** External attributes in OSPFv2. Not used in practice. In OSPFv3, the number 8 means something else (the Link LSA).
- **Types 9, 10, 11:** Opaque LSAs (RFC 5250). These carry extra data with link, area or AS flooding scope. Graceful restart uses Type 9, and MPLS traffic engineering uses Type 10.

Key points that interviewers check:

- A **Type 3 is not a summarised route**, despite the name. It may be one network or a summary. "Summary" means a short description from outside the area. It does not list routers and links.
- **Type 2 exists only where a DR exists.** A point-to-point link creates no Type 2.
- **Type 4 exists because of Type 5.** A Type 5 only says "ASBR 10.255.0.4 knows this network". Routers in other areas cannot see that ASBR in their area's Type 1 LSAs. The ABR adds a Type 4 to explain how to reach it. Inside the ASBR's own area, Type 1 LSAs already show this.
- **The ABR recreates Type 3.** When a Type 3 enters a new area, the ABR makes a new LSA with itself as the advertising router. Type 5 LSAs keep their original advertising router across areas.

This diagram shows how Types 4 and 5 travel when all three areas are standard:

```text
 Area 20                   Area 0                    Area 10
 [R4 ASBR] -----> [R3 ABR] -----------> [R2 ABR] -----------> [R1]

 Type 5 from R4 ---- crosses all three areas unchanged ---->
              R3 adds Type 4 "R4 is via me" ->
                                      R2 adds Type 4 "via me" ->
```

### 9. Cost and Reference Bandwidth

**Cost** is the OSPF metric. A **metric** is the number a routing protocol uses to compare paths, and lower is better. The cost of a path is the sum of the costs of the **outgoing** interfaces along it.

On Cisco IOS the interface cost is:

```text
cost = reference bandwidth / interface bandwidth
```

The default reference bandwidth is 100 Mbps. The result must be a whole number, decimals are dropped, and the minimum is 1.

- 10 Mbps: 100 / 10 = 10
- 100 Mbps: 100 / 100 = 1
- 1 Gbps: 100 / 1000 = 0.1, which becomes 1

The problem is plain: at the default, 100 Mbps, 1 Gbps and 100 Gbps all cost 1. OSPF cannot prefer the faster link. The fix is to raise the reference bandwidth on **all** routers:

```text
! Cisco IOS: the value is in Mbps, so 100000 means 100 Gbps
router ospf 1
 auto-cost reference-bandwidth 100000
```

| Link speed | Cost at 100 Mbps reference | Cost at 100 Gbps reference |
|---|---|---|
| 1.544 Mbps (T1) | 64 | 64766 |
| 10 Mbps | 10 | 10000 |
| 100 Mbps | 1 | 1000 |
| 1 Gbps | 1 | 100 |
| 10 Gbps | 1 | 10 |
| 40 Gbps | 1 | 2 |
| 100 Gbps | 1 | 1 |

You can also set the cost directly. This wins over the formula. The range is 1-65535.

```text
interface GigabitEthernet0/0
 ip ospf cost 50
```

Details that catch people:

- RFC 2328 does not define this formula. It only says a cost exists and an administrator can set it. The formula is a vendor default.
- Defaults vary by platform. Cisco IOS and FRRouting use 100 Mbps. Cisco NX-OS uses 40 Gbps. Check your platform.
- Cisco uses the configured `bandwidth` value on the interface, not a speed test.
- A loopback on Cisco IOS has a bandwidth of 8 Gbps. With a 100 Gbps reference, its cost becomes 12, not 1.
- Costs can differ per direction. If they do, traffic can go one way and return another.
- If two paths have equal cost, OSPF installs both and shares traffic. This is **ECMP** (Equal-Cost Multi-Path). Cisco IOS installs up to 4 paths by default. Change this with `maximum-paths`.

### 10. SPF: Calculating the Best Paths

SPF is Dijkstra's algorithm, named after computer scientist Edsger Dijkstra.

**Analogy:** pour water at your router. It spreads along every link, and slower links take longer. The moment water reaches a router, you have found the shortest path to it. **Where it stops being true:** there is no timing or real water. The router does the sums on the LSDB in memory, without sending anything.

The steps are:

1. Put yourself in the **tree** with cost 0.
2. Look at the links of the router you just added. Put each neighbour in a **candidate list** with its total cost. If a neighbour is already listed and the new cost is lower, replace it.
3. Move the candidate with the lowest cost into the tree.
4. Repeat steps 2 and 3 until the list is empty.

**Worked example.** Router A calculates its tree. The numbers are link costs, equal in both directions.

```text
  A -----10---- B ----2---- D ----5---- E
   \           /           /
    3         4           /
     \       /           8
      \     /           /
       `-- C ----------'
```

| Step | Added to tree | Candidates after this step |
|---|---|---|
| 1 | A (0) | B = 10 via B, C = 3 via C |
| 2 | C (3) | B = 3+4 = 7 via C (better than 10), D = 3+8 = 11 via C |
| 3 | B (7) | D = 7+2 = 9 via C (better than 11) |
| 4 | D (9) | E = 9+5 = 14 via C |
| 5 | E (14) | Empty, so finished |

Router A's result has the direct link to B unused, because A-C-B costs 7 and A-B costs 10.

| Destination | Cost | Path | Next hop |
|---|---|---|---|
| C | 3 | A-C | C |
| B | 7 | A-C-B | C |
| D | 9 | A-C-B-D | C |
| E | 14 | A-C-B-D-E | C |

**When does SPF run?**

- A Type 1 or Type 2 change means the topology inside the area changed, so a full SPF runs in that area.
- A Type 3, 4 or 5 change does not change the tree. The router only recalculates the affected routes. This cheaper work is called a **partial calculation**.

To protect the CPU when links flap, routers delay and space out SPF runs. This is called **SPF throttling**. On Cisco it is set with `timers throttle spf`, for example `timers throttle spf 50 200 5000` (start, hold and maximum wait in milliseconds). Default values vary by release.

### 11. Area Types

Why hide routes? An area with one or two exits does not need every external route. If all paths to the internet go through the same ABR, then one default route does the same job with far less memory.

**Analogy:** a small village with one road out does not need a world map. One sign is enough: "everywhere else, take this road". **Where it stops being true:** a village might grow a second road and still use the same sign. With OSPF, hiding routes can give a poor exit choice when two exits exist, because routers no longer see which exit is closer to each destination.

| Area type | Type 1, 2 | Type 3 | Type 4, 5 | Type 7 | Default from ABR | ASBR allowed |
|---|---|---|---|---|---|---|
| Standard | Yes | Yes | Yes | No | Not automatic | Yes |
| Stub | Yes | Yes | No | No | Automatic | No |
| Totally stubby | Yes | Default only | No | No | Automatic | No |
| NSSA | Yes | Yes | No | Yes | Not automatic | Yes |
| Totally NSSA | Yes | Default only | No | Yes | Automatic | Yes |

**Stub area.** Blocks Type 4 and Type 5 LSAs. The ABR injects a default route as a Type 3 LSA. All routers in the area must be configured as stub.

```text
! Cisco IOS, on every router in area 10
router ospf 1
 area 10 stub
```

**Totally stubby area.** Also blocks normal Type 3 LSAs. The only route from outside the area is the default. This is a Cisco term that other vendors have copied. It is not a separate type in RFC 2328. Only the ABR needs `no-summary`.

```text
! Cisco IOS, on the ABR
router ospf 1
 area 10 stub no-summary
```

**NSSA (Not-So-Stubby Area, RFC 3101).** This solves one specific problem. A branch wants to be a stub, but it also has a small link to a partner network, so it needs an ASBR. A stub area forbids that. An NSSA allows it by using a Type 7 LSA in place of a Type 5.

```text
 NSSA (area 30)                       Area 0
 Partner --> [ASBR] --Type 7--> [ABR] --Type 5--> rest of network
                                  ^
                     translates Type 7 into Type 5
```

- The ASBR creates a Type 7 LSA, which stays inside the NSSA.
- The ABR translates it into a Type 5 for the rest of the network. If an NSSA has several ABRs, one is chosen as translator: the one with the highest RID, unless you configure otherwise.
- In the routing table, these routes appear as `O N1` or `O N2`.
- An NSSA gets no automatic default route. Add one on the ABR.

```text
! Cisco IOS, on every router in area 30
router ospf 1
 area 30 nssa

! Extra on the ABR, to send a default route into the NSSA
router ospf 1
 area 30 nssa default-information-originate

! Or, for a totally NSSA, on the ABR only
router ospf 1
 area 30 nssa no-summary
```

Limits to remember: area 0 cannot be any of these types. Virtual links (see Going Deeper) cannot cross stub areas or NSSAs. All routers in the area must agree on the type, because a flag in the Hello must match.

### 12. Network Types

The **network type** of an interface tells OSPF how the link behaves. It controls DR election, timers and how neighbours are found.

| Network type | DR/BDR | Hello/Dead (Cisco) | Neighbours found by | Typical use |
|---|---|---|---|---|
| Broadcast | Yes | 10/40 | Multicast | Ethernet (default) |
| Point-to-point | No | 10/40 | Multicast | Serial, tunnels, routed links |
| NBMA | Yes | 30/120 | Manual `neighbor` | Old Frame Relay, ATM |
| Point-to-multipoint | No | 30/120 | Multicast | Hub-and-spoke, some DMVPN |
| Point-to-multipoint non-broadcast | No | 30/120 | Manual `neighbor` | Hub-and-spoke without multicast |
| Loopback (Cisco) | No | None | No neighbours | Loopback interfaces |

**NBMA** means Non-Broadcast Multi-Access: many routers on one network, but no way to send to all at once. It is mostly history, but it is still asked about in interviews.

**A strong practical tip:** an Ethernet link between exactly two routers does not need a DR. Set it to point-to-point. There is no election, no 40-second wait, and no Type 2 LSA.

```text
interface GigabitEthernet0/0
 ip ospf network point-to-point
```

**Fuller truth:** Cisco advertises a loopback as a /32 host route, whatever its mask. To advertise the real mask, set `ip ospf network point-to-point` on the loopback.

### 13. Route Types and Selection Order

Cisco marks OSPF routes with these codes:

| Code | Meaning | Source LSA |
|---|---|---|
| `O` | Intra-area, inside my area | Type 1, 2 |
| `O IA` | Inter-area, from another area | Type 3 |
| `O E1` | External Type 1 | Type 5 |
| `O E2` | External Type 2 (Cisco default for redistribution) | Type 5 |
| `O N1`, `O N2` | NSSA external | Type 7 |

OSPF chooses in a fixed order. It looks at the route type first and compares cost only inside the same type:

1. Intra-area (`O`)
2. Inter-area (`O IA`)
3. External Type 1 (`O E1`, then `O N1`)
4. External Type 2 (`O E2`, then `O N2`)

So an `O` route with cost 1000 beats an `O IA` route with cost 10 to the same prefix.

**E1 and E2 differ in cost:**

- **E2:** the cost stays at the external metric, however far away the ASBR is. The internal path cost is ignored. On Cisco the default seed metric is 20 (1 for routes redistributed from BGP).
- **E1:** the cost is the external metric plus the internal cost to the ASBR.

Use E1 when two ASBRs advertise the same external network and you want each router to use the closer one.

On Cisco, OSPF has an **administrative distance** of 110, which is how Cisco compares routes from different protocols. See [The Routing Table](02-routing-table.md).

### 14. What Breaks an Adjacency

Two routers compare fields in each Hello. If a required field differs, they ignore each other.

| Item | Must match? | Stage where it fails |
|---|---|---|
| Area ID | Yes | Hello |
| Hello and dead timers | Yes | Hello |
| Subnet mask | Yes on broadcast and NBMA; ignored on point-to-point | Hello |
| Same IP subnet | Yes on shared networks | Hello |
| Authentication type and key | Yes | Hello |
| Area type (stub or NSSA flag) | Yes | Hello |
| Interface MTU | Yes, unless ignored | DBD (ExStart/Exchange) |
| Router ID | Must be **different** | Hello or DBD |
| Network type | Should match | Often no failure at Hello, see below |
| Process ID | No, it is only local | Never |
| Priority | No | Never |
| Cost | No | Never |

**Use the stuck state as your clue:**

| Symptom | Common causes |
|---|---|
| No neighbour at all, or Down | Link down, OSPF not enabled, passive interface, ACL blocks protocol 89, Hello mismatch |
| Stuck in Init | One-way traffic: an ACL, a bad link, or multicast dropped in one direction |
| Stuck in 2-Way | Normal for two DROTHERs. Otherwise both ends have priority 0, so no DR exists |
| Stuck in ExStart or Exchange | MTU mismatch, duplicate RID, or unicast reachability problem between the routers |
| Stuck in Loading | Corrupt or rejected LSA, MTU problems on large LSUs |
| Flapping | Overloaded CPU, dropped Hellos, dead timer too short, unstable link |
| Full, but routes are missing | Network type mismatch, inbound filter, area not connected to area 0, duplicate RID |

An **ACL** (Access Control List) is a set of permit and deny rules. See [Access Control Lists](../09-security/05-access-control-lists.md). A **passive interface** advertises its network but sends no Hellos, so it never forms a neighbour.

**The network type trap:** if one end is broadcast and the other is point-to-point, the timers match. So the routers can reach Full. But one router describes a LAN with a DR, and the other describes a direct link. The two descriptions do not agree, so SPF rejects the connection and routes do not install. If one end is broadcast and the other is point-to-multipoint, the timers differ, so no neighbour forms at all.

## Example

The example network has three areas. Output was shaped for this page: interface names are shortened to `Gi0/0`, and a few long lines are wrapped.

### Topology

```text
 Users LAN 10.10.1.0/24
        |
      Gi0/1 10.10.1.1
     +----+
     | R1 |  Internal, Lo0 10.255.0.1/32           AREA 10
     +----+                                   (totally stubby)
      Gi0/0 10.10.12.1/30
        |
      Gi0/0 10.10.12.2/30
     +----+
     | R2 |  ABR area 10 and 0, Lo0 10.255.0.2/32
     +----+
      Gi0/1 10.0.23.1/30
        |                                          AREA 0
      Gi0/0 10.0.23.2/30
     +----+
     | R3 |  ABR area 0 and 20, Lo0 10.255.0.3/32
     +----+
      Gi0/1 10.20.34.1/30
        |                                          AREA 20
      Gi0/0 10.20.34.2/30
     +----+
     | R4 |  ASBR, Lo0 10.255.0.4/32
     +----+
      | | |
      | | +- Gi0/3 192.168.99.1/30 -- Partner 192.168.99.2
      | |                             (behind it: 172.16.50.0/24)
      | +--- Gi0/2 192.0.2.2/30 ------ ISP 192.0.2.1
      +----- Gi0/1 10.20.1.1/24 ------ Server LAN 10.20.1.0/24
```

Design choices:

- All transit links are point-to-point, so no DR and no Type 2 LSAs.
- The reference bandwidth is 10 Gbps (10000 Mbps), since 1 Gbps is the fastest link here. So each Gigabit interface costs 10, and each loopback costs 1 (10000 / 8000, rounded down).
- Area 10 has one exit, so it is totally stubby.
- R4 redistributes a static route to the partner and also advertises a default route toward the ISP.

### Configuration (Cisco IOS)

Interfaces are enabled directly with `ip ospf 1 area X`. The `1` is the process ID, which only matters on this router.

```text
! ===== R1 =====
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
 ip ospf 1 area 10
interface GigabitEthernet0/0
 ip address 10.10.12.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 10
interface GigabitEthernet0/1
 ip address 10.10.1.1 255.255.255.0
 ip ospf 1 area 10
router ospf 1
 router-id 10.255.0.1
 auto-cost reference-bandwidth 10000
 passive-interface GigabitEthernet0/1
 area 10 stub
```

```text
! ===== R2 =====
interface Loopback0
 ip address 10.255.0.2 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/0
 ip address 10.10.12.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 10
interface GigabitEthernet0/1
 ip address 10.0.23.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
router ospf 1
 router-id 10.255.0.2
 auto-cost reference-bandwidth 10000
 area 10 stub no-summary
 area 10 range 10.10.0.0 255.255.0.0 cost 20
```

```text
! ===== R3 =====
interface Loopback0
 ip address 10.255.0.3 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/0
 ip address 10.0.23.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
interface GigabitEthernet0/1
 ip address 10.20.34.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 20
router ospf 1
 router-id 10.255.0.3
 auto-cost reference-bandwidth 10000
```

```text
! ===== R4 =====
interface Loopback0
 ip address 10.255.0.4 255.255.255.255
 ip ospf 1 area 20
interface GigabitEthernet0/0
 ip address 10.20.34.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 20
interface GigabitEthernet0/1
 ip address 10.20.1.1 255.255.255.0
 ip ospf 1 area 20
interface GigabitEthernet0/2
 ip address 192.0.2.2 255.255.255.252
interface GigabitEthernet0/3
 ip address 192.168.99.1 255.255.255.252
ip route 0.0.0.0 0.0.0.0 192.0.2.1
ip route 172.16.50.0 255.255.255.0 192.168.99.2
router ospf 1
 router-id 10.255.0.4
 auto-cost reference-bandwidth 10000
 passive-interface GigabitEthernet0/1
 redistribute static subnets
 default-information originate
```

The older way to enable interfaces is with `network` statements and a wildcard mask. A **wildcard mask** is the inverse of a subnet mask. Both ways work. If both match, the interface command wins.

```text
! R1, the classic style
router ospf 1
 network 10.10.12.0 0.0.0.3 area 10
 network 10.10.1.0 0.0.0.255 area 10
 network 10.255.0.1 0.0.0.0 area 10
```

The `redistribute` line above does not carry the default route. `default-information originate` does that, and only when a default route is already in the routing table. Adding `always` advertises it even without one. On older IOS versions the `subnets` keyword is needed to carry non-classful networks. Newer versions add it for you.

### Verify the Neighbours

```text
R2# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address      Interface
10.255.0.1        0   FULL/  -        00:00:35    10.10.12.1   Gi0/0
10.255.0.3        0   FULL/  -        00:00:33    10.0.23.2    Gi0/1
```

`FULL/  -` means Full with no DR, as expected on point-to-point links.

```text
R2# show ip ospf interface brief

Interface  PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Lo0        1     0               10.255.0.2/32      1     LOOP  0/0
Gi0/1      1     0               10.0.23.1/30       10    P2P   1/1
Gi0/0      1     10              10.10.12.2/30      10    P2P   1/1
```

This one command shows each interface's area, cost, state and neighbours. Now look at one interface in detail:

```text
R2# show ip ospf interface GigabitEthernet0/0

Gi0/0 is up, line protocol is up
  Internet Address 10.10.12.2/30, Area 10, Attached via Interface Enable
  Process ID 1, Router ID 10.255.0.2, Network Type POINT_TO_POINT,
  Cost: 10
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:04
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.255.0.1
```

And the process view confirms the router's role and the area type:

```text
R2# show ip ospf

 Routing Process "ospf 1" with ID 10.255.0.2
 It is an area border router
 Reference bandwidth unit is 10000 mbps
    Area BACKBONE(0)
        Number of interfaces in this area is 2 (1 loopback)
    Area 10
        Number of interfaces in this area is 1
        It is a stub area, no summary LSA in this area
        Generates stub default route with cost 1
```

### Verify the Routes

```text
R1# show ip route ospf

Gateway of last resort is 10.10.12.2 to network 0.0.0.0

O*IA  0.0.0.0/0 [110/11] via 10.10.12.2, 00:14:02, Gi0/0
```

R1 needs nothing else. Its entire view of the outside world is one default route. The cost is 1 (R2's stub default cost) plus 10 (R1's outgoing Gi0/0), which is 11.

```text
R3# show ip route ospf

Gateway of last resort is 10.20.34.2 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.20.34.2, 00:09:41, Gi0/1
      10.0.0.0/8 is variably subnetted, 10 subnets, 4 masks
O IA     10.10.0.0/16 [110/30] via 10.0.23.1, 00:12:08, Gi0/0
O        10.20.1.0/24 [110/20] via 10.20.34.2, 00:10:17, Gi0/1
O IA     10.255.0.1/32 [110/21] via 10.0.23.1, 00:12:08, Gi0/0
O        10.255.0.2/32 [110/11] via 10.0.23.1, 00:12:40, Gi0/0
O        10.255.0.4/32 [110/11] via 10.20.34.2, 00:10:17, Gi0/1
      172.16.0.0/24 is subnetted, 1 subnets
O E2     172.16.50.0 [110/20] via 10.20.34.2, 00:09:41, Gi0/1
```

Check these costs by hand:

| Route on R3 | Calculation | Cost |
|---|---|---|
| 10.10.0.0/16 | 10 (R3 Gi0/0) + 20 (summary cost set on R2) | 30 |
| 10.20.1.0/24 | 10 (R3 Gi0/1) + 10 (R4 Gi0/1) | 20 |
| 10.255.0.1/32 | 10 (R3 Gi0/0) + 10 (R2 Gi0/0) + 1 (R1 Lo0) | 21 |
| 10.255.0.2/32 | 10 (R3 Gi0/0) + 1 (R2 Lo0) | 11 |
| 172.16.50.0/24 | E2 external metric only | 20 |
| 0.0.0.0/0 | E2 metric, default for `default-information originate` | 1 |

The R1 loopback appears as a separate route because 10.255.0.1 is outside the 10.10.0.0/16 summary. A real design would pick addresses so the summary covers them.

### Verify the Database

R3 is an ABR, so it holds one LSDB per area, plus the external LSAs that are shared by all normal areas.

```text
R3# show ip ospf database

            OSPF Router with ID (10.255.0.3) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age     Seq#       Checksum Link count
10.255.0.2      10.255.0.2      741     0x80000003 0x0061C4 3
10.255.0.3      10.255.0.3      738     0x80000003 0x00A7E1 3

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age     Seq#       Checksum
10.10.0.0       10.255.0.2      729     0x80000001 0x004E12
10.20.1.0       10.255.0.3      612     0x80000001 0x00C93A
10.20.34.0      10.255.0.3      733     0x80000001 0x00B20D
10.255.0.1      10.255.0.2      729     0x80000001 0x002F7B
10.255.0.4      10.255.0.3      612     0x80000001 0x0091D6

                Summary ASB Link States (Area 0)

Link ID         ADV Router      Age     Seq#       Checksum
10.255.0.4      10.255.0.3      604     0x80000001 0x0079EE

                Router Link States (Area 20)

Link ID         ADV Router      Age     Seq#       Checksum Link count
10.255.0.3      10.255.0.3      735     0x80000002 0x00D4A8 2
10.255.0.4      10.255.0.4      606     0x80000003 0x0033BF 4

                Summary Net Link States (Area 20)

Link ID         ADV Router      Age     Seq#       Checksum
10.0.23.0       10.255.0.3      733     0x80000001 0x0018CD
10.10.0.0       10.255.0.3      725     0x80000001 0x00F03E
10.255.0.1      10.255.0.3      725     0x80000001 0x00D15A
10.255.0.2      10.255.0.3      733     0x80000001 0x00B66F
10.255.0.3      10.255.0.3      733     0x80000001 0x009C84

                Type-5 AS External Link States

Link ID         ADV Router      Age     Seq#       Checksum Tag
0.0.0.0         10.255.0.4      604     0x80000001 0x00B7C2 1
172.16.50.0     10.255.0.4      604     0x80000001 0x005C7E 0
```

How to read it:

- There are no "Net Link States" sections, because point-to-point links create no Type 2 LSAs.
- `Link count` is the number of link entries, not neighbours. On Cisco a numbered point-to-point link adds 2 entries (one for the neighbour, one for the subnet). R4 has 4: 2 for its link to R3, 1 for its LAN and 1 for its loopback.
- In the Area 20 Summary section, R3 is the advertising router for 10.10.0.0, although R2 made the original in area 0. ABRs recreate Type 3 LSAs.
- The Type 5 LSAs still show R4, and R3 added a Type 4 for R4 into area 0.

Now compare R1's database in the totally stubby area:

```text
R1# show ip ospf database

            OSPF Router with ID (10.255.0.1) (Process ID 1)

                Router Link States (Area 10)

Link ID         ADV Router      Age     Seq#       Checksum Link count
10.255.0.1      10.255.0.1      702     0x80000004 0x0048D1 4
10.255.0.2      10.255.0.2      698     0x80000003 0x00E72C 2

                Summary Net Link States (Area 10)

Link ID         ADV Router      Age     Seq#       Checksum
0.0.0.0         10.255.0.2      741     0x80000001 0x00A5B3
```

Only three LSAs. This is the benefit of a totally stubby area. To inspect one external LSA in detail:

```text
R3# show ip ospf database external 172.16.50.0

            OSPF Router with ID (10.255.0.3) (Process ID 1)

                Type-5 AS External Link States

  LS age: 604
  Options: (No TOS-capability, DC, Upward)
  LS Type: AS External Link
  Link State ID: 172.16.50.0 (External Network Number )
  Advertising Router: 10.255.0.4
  LS Seq Number: 80000001
  Checksum: 0x5C7E
  Length: 36
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        MTID: 0
        Metric: 20
        Forward Address: 0.0.0.0
        External Route Tag: 0
```

### Useful Verification Commands

| Cisco IOS command | Shows |
|---|---|
| `show ip ospf neighbor` | Neighbour state and DR role |
| `show ip ospf interface brief` | Area, cost and state per interface |
| `show ip ospf interface Gi0/0` | Timers, network type, DR, passive state |
| `show ip ospf` | RID, role, areas, SPF count |
| `show ip ospf database` | The LSDB summary |
| `show ip ospf database router` | Full Type 1 LSAs (also `network`, `summary`, `asbr-summary`, `external`, `nssa-external`) |
| `show ip ospf border-routers` | Routes to ABRs and ASBRs |
| `show ip route ospf` | Installed OSPF routes |
| `show ip protocols` | Passive interfaces, networks, sources |

### Troubleshooting Example: Stuck in ExStart

Someone sets the MTU on R3 Gi0/0 to 1400. R2 stays at 1500.

```text
R3# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address      Interface
10.255.0.2        0   EXSTART/  -     00:00:36    10.0.23.1    Gi0/0

R2# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address      Interface
10.255.0.3        0   EXCHANGE/  -    00:00:37    10.0.23.2    Gi0/1
```

R3 receives a DBD saying the neighbour's MTU is 1500, which is larger than its own 1400, so it rejects it. R2 accepts R3's smaller value and moves on, so the two ends show different states. A debug confirms it:

```text
R3# debug ip ospf adj
OSPF: Nbr 10.255.0.2 has larger interface MTU
R3# undebug all
```

The fix is to set the same MTU on both ends:

```text
R3(config)# interface GigabitEthernet0/0
R3(config-if)# mtu 1500
```

`ip ospf mtu-ignore` skips the check, but it only hides the mismatch. Large OSPF packets might still be dropped later.

A timer mismatch looks different. The debug names both values, with `R` for received and `C` for configured here:

```text
R1# debug ip ospf hello
OSPF: Rcv hello from 10.255.0.2 area 10 from Gi0/0 10.10.12.2
OSPF: Mismatched hello parameters from 10.10.12.2
OSPF: Dead R 40 C 120, Hello R 10 C 30
      Mask R 255.255.255.252 C 255.255.255.252
```

The exact debug text varies by IOS release. Use `debug` with care on busy production routers.

### OSPF on Linux

Linux has no OSPF in the kernel. Use a routing daemon, such as FRRouting (FRR) or BIRD. Say a Linux server at 10.20.1.9 joins area 20 on the server LAN. For that, R4 must stop treating Gi0/1 as passive.

```bash
# Linux with FRRouting: turn on the OSPF daemon, then restart FRR
sudo sed -i 's/^ospfd=no/ospfd=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

```text
! FRRouting on Linux: /etc/frr/frr.conf
router ospf
 ospf router-id 10.255.0.9
 auto-cost reference-bandwidth 10000
 network 10.20.1.0/24 area 20
```

```bash
# Linux: check neighbours, then watch OSPF packets (IP protocol 89)
sudo vtysh -c "show ip ospf neighbor"
sudo tcpdump -ni eth0 ip proto 89
```

```text
10:15:02.418233 IP 10.20.1.9 > 224.0.0.5: OSPFv2, Hello, length 48
```

The length fits: 24 for the header, 20 for the fixed Hello fields and 4 for one neighbour. For more on capture, see [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

On Windows, there is no built-in OSPF router in current versions. Microsoft removed OSPF from Routing and Remote Access in Windows Server 2008. You can still capture it with [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md), using the display filter `ospf`.

## Going Deeper

### LSA Ageing, Sequence Numbers and Flooding

Each LSA has three fields that decide which copy is newest:

- **Sequence number:** starts at 0x80000001 and rises with each new version. Higher is newer.
- **Checksum:** detects damage. If sequence numbers tie, the higher checksum wins.
- **Age:** starts at 0 and rises each second, up to **MaxAge** (3600 s). An LSA at MaxAge is removed from the network.

The owner of an LSA refreshes it every 30 minutes (1800 s), even with no change. To withdraw an LSA early, the owner sets its age to 3600 and floods it. This is called **premature ageing**.

An LSU is acknowledged with an LSAck. If none arrives, the sender resends it after the retransmit interval, 5 s by default on Cisco. So a reliable flood works without TCP.

### Inside a Router LSA

A Type 1 LSA describes links in four kinds:

| Link type | Meaning | Link ID |
|---|---|---|
| 1 Point-to-point | Direct link to a router | Neighbour's RID |
| 2 Transit | Link to a LAN with a DR | DR's interface address |
| 3 Stub | A network with no neighbour | Network address |
| 4 Virtual | Virtual link | Neighbour's RID |

Note that a "stub network" here means a subnet with no OSPF neighbour. This has nothing to do with a stub area.

A Router LSA also has flag bits: **B** means this router is an ABR, **E** means an ASBR, and **V** means it is the end of a virtual link. That is how routers in the same area find their ASBR without a Type 4.

### How OSPF Avoids Loops Between Areas

Inside an area, OSPF is true link-state. Between areas, it acts more like distance-vector: an ABR says "I can reach X at cost Y", and others trust it.

The hub design keeps this safe. An ABR only uses Type 3 LSAs learned from area 0 when it calculates inter-area routes. It ignores Type 3s heard in a non-backbone area, so routes cannot circle between non-backbone areas.

### Summarisation

Summarisation replaces many routes with one short one. OSPF can do this only at the borders, because LSDBs inside an area must be identical. See [VLSM and Route Summarisation](../04-ip-addressing/07-vlsm-and-supernetting.md).

```text
! On an ABR: summarise Type 3 for routes from area 10
router ospf 1
 area 10 range 10.10.0.0 255.255.0.0

! On an ASBR: summarise Type 5 (or Type 7) external routes
router ospf 1
 summary-address 172.16.0.0 255.255.0.0
```

- A summary hides flaps. If 10.10.1.0/24 fails, routers in other areas see no change.
- The router adds a discard route to `Null0` for the summary. This stops traffic to unused addresses from looping.
- RFC 2328 says the summary takes the highest cost of its parts. The older RFC 1583 used the lowest. Implementations differ, so add `cost` when the number matters, as in the example.

### Filtering

Inside an area you cannot remove an LSA, because that would give routers different maps. What you can filter:

- On an ABR, `area 10 filter-list prefix NAME in` or `out` controls Type 3 LSAs.
- On an ABR, `area 10 range ... not-advertise` hides a range.
- On an ASBR, a route map on `redistribute` controls what enters OSPF.
- On any router, `distribute-list ... in` stops a route entering the local routing table only. The LSA stays in the LSDB and continues to flood. This can cause a traffic black hole, where packets are silently dropped.

### Virtual Links

A **virtual link** is a logical connection through a normal area, used when an area cannot touch area 0 directly, or when area 0 has been split in two.

```text
  Area 0 ---[R2]--- Area 10 ---[R5]--- Area 50
                (transit area)
           <--- virtual link --->
```

```text
! Cisco IOS: on R2 and R5, using each other's RID
! On R2 (R5's RID is 10.255.0.5)
router ospf 1
 area 10 virtual-link 10.255.0.5
! On R5 (R2's RID is 10.255.0.2)
router ospf 1
 area 10 virtual-link 10.255.0.2
```

**Analogy:** a tunnel that lets a village connect to the motorway through another village's roads. **Where it stops being true:** traffic in a virtual link is plain routing through the transit area, with no separate pipe of its own.

A virtual link cannot cross a stub area or NSSA. Treat it as a temporary repair, since a real link or better design is cleaner.

### Authentication

Without authentication, a bad device on a link can become a neighbour and inject routes. OSPFv2 supports:

| Type | Name | Comment |
|---|---|---|
| 0 | Null | No protection (default) |
| 1 | Simple password | Sent in clear text. Avoid. |
| 2 | Cryptographic | MD5 in RFC 2328, HMAC-SHA in RFC 5709 |

```text
! Cisco IOS: MD5 on an interface (legacy, but still common)
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Ex4mple-Only

! Cisco IOS: HMAC-SHA-256 with a key chain (on releases that support it)
key chain OSPF-KEYS
 key 1
  key-string Ex4mple-Only
  cryptographic-algorithm hmac-sha-256
interface GigabitEthernet0/0
 ip ospf authentication key-chain OSPF-KEYS
```

Also mark user-facing interfaces as passive. It is the simplest defence, because no Hellos are sent there. Use `passive-interface default`, then `no passive-interface` on the real links.

### External Routes and the Forwarding Address

A Type 5 LSA has a **forwarding address** field. When it is 0.0.0.0, traffic goes to the ASBR. When it has a value, traffic goes straight to that address, which can avoid an extra hop on a shared LAN.

If two E2 routes have the same metric, the router chooses the one with the lower internal cost to the ASBR. This is called the forward metric.

Two-way redistribution between protocols can cause loops. Use route tags and filters to stop them. See [Advanced Routing Techniques](12-advanced-routing-techniques.md).

### Faster Convergence and Graceful Maintenance

- **BFD** (Bidirectional Forwarding Detection, RFC 5880) is a light protocol for fast failure detection. It can find a failure in well under a second. OSPF then drops the neighbour at once.

```text
! Cisco IOS: BFD with 50 ms timers, 3 missed packets means down
interface GigabitEthernet0/0
 bfd interval 50 min_rx 50 multiplier 3
 ip ospf bfd
```

- **Fast hellos:** `ip ospf dead-interval minimal hello-multiplier 4` gives a 1 s dead time. BFD is the lighter choice.
- **SPF and LSA throttling** react quickly to one event, then slow down during a storm.
- **Stub router advertisement** (RFC 6987): `max-metric router-lsa` tells others to avoid this router by advertising a very high cost. Use it before maintenance. `max-metric router-lsa on-startup 300` protects traffic for 300 s while BGP or other services settle after a reboot.
- **Graceful restart** (RFC 3623): a router restarts its OSPF software while it keeps forwarding, and its neighbours hold their routes. Cisco calls a related feature NSF (Nonstop Forwarding).
- **LFA** (Loop-Free Alternate, RFC 5286): a pre-computed backup next hop, so a router can switch very fast without waiting for SPF.
- **Segment routing** (RFC 8665) extends OSPF to carry segment labels. See [MPLS](13-mpls.md) for the label idea.

### Design Notes

- No RFC sets a maximum for routers per area. The safe size depends on hardware, link stability and the LSA count.
- Keep area 0 stable and contiguous.
- Pick addresses per area so that ABRs can summarise cleanly.
- Use two ABRs per area for redundancy, and remember the exit choice problem when you hide routes.
- OSPF also appears as the underlay in data centres. See [Spine-Leaf Architecture](../10-enterprise-and-datacenter/02-spine-leaf.md).

### OSPFv3

OSPFv3 was built for IPv6, and it is defined in RFC 5340. It keeps the same core: Hello, DR election, areas, SPF, cost, neighbour states and area types. The main changes:

| Topic | OSPFv2 | OSPFv3 |
|---|---|---|
| Carries | IPv4 | IPv6, plus IPv4 with address families (RFC 5838) |
| Runs on | A subnet | A link, and several prefixes can share it |
| Packet source | Interface IPv4 address | IPv6 link-local address |
| Multicast | 224.0.0.5, 224.0.0.6 | FF02::5, FF02::6 |
| Header | 24 bytes, with authentication | 16 bytes, no authentication, has Instance ID |
| Authentication | Built in | IPsec (RFC 4552) or authentication trailer (RFC 7166) |
| Router and Network LSAs | Hold addresses | Hold topology only, no prefixes |
| Neighbour ID | By IP on broadcast links | Always by RID |

The Router ID is still a 32-bit number. On a router with no IPv4 address you must set it by hand, or OSPFv3 will not start.

A **link-local address** works only on one link. It starts with FE80, and routers use it as the next hop. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

OSPFv3 LSAs have new names and numbers:

| LS type | Name | Scope | Like OSPFv2 |
|---|---|---|---|
| 0x2001 | Router | Area | Type 1, without prefixes |
| 0x2002 | Network | Area | Type 2, without prefixes |
| 0x2003 | Inter-Area-Prefix | Area | Type 3 |
| 0x2004 | Inter-Area-Router | Area | Type 4 |
| 0x4005 | AS-External | AS | Type 5 |
| 0x2007 | NSSA | Area | Type 7 |
| 0x0008 | Link | Link | New: link-local address and prefixes on the link |
| 0x2009 | Intra-Area-Prefix | Area | New: the prefixes that Types 1 and 2 no longer carry |

Moving prefixes out of the Router and Network LSAs has a real benefit. Adding a prefix no longer changes the topology, so it does not force a full SPF.

```text
! Cisco IOS: OSPFv3 for IPv6, classic style
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 10.255.0.1
interface GigabitEthernet0/0
 ipv6 address 2001:db8:12::1/64
 ipv6 ospf 1 area 0

! Cisco IOS: OSPFv3 with address families, newer style
router ospfv3 1
 router-id 10.255.0.1
 address-family ipv6 unicast
 exit-address-family
interface GigabitEthernet0/0
 ipv6 address 2001:db8:12::1/64
 ospfv3 1 ipv6 area 0
```

```text
R1# show ipv6 ospf neighbor

Neighbor ID     Pri   State           Dead Time   Interface ID Interface
10.255.0.2        1   FULL/DR         00:00:36    3            Gi0/0
```

Note that the neighbour shows an interface ID, and it has no IPv6 address column. Use `show ipv6 route ospf` to see routes, and expect next hops such as `FE80::2`.

### OSPF Compared to Other IGPs

| Feature | OSPF | IS-IS | EIGRP |
|---|---|---|---|
| Family | Link-state | Link-state | Advanced distance-vector |
| Runs over | IP protocol 89 | Directly on Layer 2 | IP protocol 88 |
| Area design | Area 0 hub | Levels 1 and 2 | No areas |
| Open standard | Yes | Yes | Published by Cisco, mostly Cisco |

See [EIGRP](07-eigrp.md) and [IS-IS](08-is-is.md). IS-IS uses the same Dijkstra algorithm and is common in large provider networks.

## Common Mistakes

- **Thinking 2-Way is a fault.** Two DROTHERs on a LAN stay in 2-Way by design.
- **Thinking the DR forwards user traffic.** It only coordinates OSPF flooding on one segment.
- **Saying a router is "the DR for the area".** The election is per segment.
- **Expecting a better router to take over the DR role.** There is no preemption. Reset the OSPF process, or wait for a failure.
- **Leaving the reference bandwidth at the default.** At 100 Mbps, a 1 Gbps link and a 100 Gbps link have the same cost.
- **Changing the reference bandwidth on only some routers.** Costs stop being comparable, and paths look strange. Set it on all routers.
- **Thinking the process ID must match.** `router ospf 1` and `router ospf 99` can be neighbours.
- **Thinking `Type 3` means a summarised route.** It is any inter-area route.
- **Believing a low-cost `O IA` beats a high-cost `O`.** Route type comes before cost.
- **Assuming E2 cost shows the real distance.** E2 ignores the internal path cost in the displayed metric.
- **Using `distribute-list in` to filter an LSA.** It only affects the local routing table.
- **Forgetting that stub flags must match.** Configure stub on the ABR but not the internal router, and the neighbour drops.
- **Trusting a mask mismatch check on point-to-point links.** The standard ignores the mask on these links, so this mismatch might not stop the adjacency.
- **Changing `router-id` and expecting it to apply at once.** It needs a process restart.
- **Forgetting the default route in an NSSA.** Unlike a stub, an NSSA gets no automatic default, unless it is totally NSSA.

## Interview Questions

**Q: Walk me through how two OSPF routers become adjacent.**
A: They start in Down. One hears a Hello and goes to Init. When a router sees its own RID in the neighbour's Hello, it moves to 2-Way. A DR election happens here on a LAN. In ExStart they pick a master by higher RID. In Exchange they swap DBD packets, which list LSA headers. In Loading they request missing LSAs with LSR, receive them in LSU and confirm with LSAck. Then they are Full.

**Q: A neighbour is stuck in ExStart. What do you check?**
A: First the interface MTU on both ends, since ExStart is where the DBD carries it, and an MTU mismatch is the most common cause. I would compare `show interfaces` on both sides and use `debug ip ospf adj`. Then I would check for a duplicate router ID. The right fix is to match the MTU. `ip ospf mtu-ignore` only hides it.

**Q: Why do we need a DR and BDR?**
A: On a shared LAN, a full mesh of adjacencies needs n(n-1)/2 relationships, and LSAs get flooded many times. With a DR, each router forms adjacencies only with the DR and BDR. They send updates to 224.0.0.6, and the DR floods to 224.0.0.5. It is elected per segment by highest priority, then highest RID, and it is not preemptive. It is not used on point-to-point links.

**Q: What is the difference between Type 1, 2, 3, 4, 5 and 7 LSAs?**
A: Type 1 comes from every router and lists its links. Type 2 comes from the DR and lists routers on a LAN. Both stay in the area. Type 3 comes from an ABR and carries inter-area networks. Type 4 comes from an ABR and tells other areas how to reach an ASBR. Type 5 comes from an ASBR and carries external routes across normal areas. Type 7 is an external route inside an NSSA, and the ABR translates it to Type 5.

**Q: Why must every area connect to area 0?**
A: Between areas OSPF behaves like distance-vector, so loops are possible. With a star shape around area 0, an ABR accepts inter-area routes only from the backbone, so routes cannot loop through non-backbone areas. If an area cannot connect directly, a virtual link can repair it, but that is a temporary fix.

**Q: Compare stub, totally stubby and NSSA areas.**
A: A stub blocks Type 4 and 5, and the ABR sends a default route. A totally stubby area also blocks normal Type 3, so it only has the default route from outside. Neither one can contain an ASBR. An NSSA also blocks Type 5, but it allows an ASBR that creates Type 7 LSAs, and the ABR translates those to Type 5. An NSSA gets no automatic default route unless you configure it or use totally NSSA.

**Q: How is OSPF cost calculated, and what is wrong with the default?**
A: On Cisco, cost is reference bandwidth divided by interface bandwidth, with a minimum of 1 and decimals dropped. The default reference is 100 Mbps, so Fast Ethernet, 1 Gig and 100 Gig all cost 1, and OSPF cannot tell them apart. I would set `auto-cost reference-bandwidth` to a value above the fastest link, the same on every router.

**Q: What is the difference between E1 and E2 routes?**
A: Both are external routes from Type 5 LSAs. E2 keeps the external metric only and is the Cisco default. E1 adds the internal cost to the ASBR. If you have two exits, E1 lets each router choose the nearer one. If the same prefix exists as both, OSPF prefers E1.

**Q: Why do you see 2WAY/DROTHER in `show ip ospf neighbor`, and is it a problem?**
A: It means this router and that neighbour are both non-DR routers on a shared segment. They see each other but do not sync directly, since both sync with the DR and BDR. It is normal. It is only a problem if no DR exists, for example if both routers on a link have priority 0.

**Q: What can stop two routers forming an adjacency?**
A: Different area IDs, hello or dead timers, authentication settings, area types, or subnet masks on a broadcast link. Also an MTU mismatch, a duplicate router ID, a passive interface, an ACL that blocks IP protocol 89, or a Layer 2 fault that drops multicast. I start from the stuck state, then use `show ip ospf interface` and a debug.

**Q: What is new in OSPFv3?**
A: It runs per link, not per subnet, and it uses link-local addresses as source and next hop. It has no authentication of its own, so it uses IPsec or an authentication trailer. Router and Network LSAs hold only topology, and two new LSAs carry prefixes: Link and Intra-Area-Prefix. The rest is familiar: areas, DR, SPF and states. The RID is still 32 bits, so an IPv6-only router needs one set by hand.

**Q: A route is in the LSDB but not in the routing table. Why?**
A: First, it might not be the best route: another protocol with a lower administrative distance could win. Second, a `distribute-list in` may block it. Third, SPF uses a link only if both routers describe it the same way, and a network type mismatch breaks that check. Fourth, the next hop or the ASBR may be unreachable, for example a missing Type 4 or a Type 5 with an unreachable forwarding address. I check `show ip ospf database` for the LSA, then `show ip ospf border-routers` and `show ip route`.

## Quick Recap

- OSPF is a link-state IGP on IP protocol 89. Routers flood LSAs, build an identical LSDB per area, and run SPF to choose routes.
- Neighbours go through Down, Init, 2-Way, ExStart, Exchange, Loading and Full. Attempt appears only on NBMA.
- On a LAN, a DR and BDR reduce flooding. The election goes by priority, then RID, with no preemption.
- Area 0 is the hub. ABRs join areas, ASBRs bring in external routes, and LSA Types 1 and 2 stay inside an area, while Types 3, 4, 5 and 7 cross or describe borders.
- Cost is reference bandwidth over interface bandwidth. Raise the reference on all routers. Route type beats cost.
- Stub, totally stubby and NSSA areas shrink the LSDB. NSSA allows an ASBR through Type 7.
- Adjacency needs matching area, timers, mask (on shared links), authentication, area type and MTU, and it needs unique RIDs. OSPFv3 keeps the same ideas for IPv6, with link-local addressing and new LSAs.
