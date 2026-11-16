# IS-IS

**Level:** Advanced | **Prerequisites:** [Dynamic Routing Overview](04-dynamic-routing-overview.md), [OSPF](06-ospf.md), [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md)

## In One Line

IS-IS (Intermediate System to Intermediate System) is a link-state routing protocol that runs directly on Layer 2 instead of on IP. It is the main internal routing protocol in many large ISP backbones.

## Why It Matters

A large network needs every router to know the best path to every other router. This job belongs to an IGP (Interior Gateway Protocol), a routing protocol used inside one organisation's network. [BGP](09-bgp.md) carries internet routes, but BGP depends on an IGP to reach its next hops. If the IGP fails, BGP traffic fails too.

Most enterprises pick [OSPF](06-ospf.md) for this job. Many ISPs (Internet Service Providers) pick IS-IS. Both protocols do the same basic work, so the interesting question is why anyone would choose the less famous one.

Where you find IS-IS today:

- **ISP and carrier backbones.** This is its home. It often runs with [MPLS](13-mpls.md) or Segment Routing and carries the loopback addresses that BGP uses.
- **Some data centres.** It can be the underlay (the basic IP routing layer) in a [spine-leaf](../10-enterprise-and-datacenter/02-spine-leaf.md) fabric. Cisco ACI fabrics use it inside. Many very large data centres use BGP instead, so IS-IS is a real choice here but not the only one.
- **Layer 2 fabric protocols.** TRILL (RFC 6325), SPB (Shortest Path Bridging, IEEE 802.1aq) and Cisco FabricPath all reuse IS-IS to carry their own information.

Interviewers ask about IS-IS for three reasons:

1. It shows whether you understand link-state routing beyond one protocol.
2. "Why do ISPs prefer IS-IS over OSPF?" is a classic senior question.
3. The odd parts (no IP, NET addresses, levels) quickly show who has only memorised OSPF.

## Simple Explanation

**Talking over the fence.** Most routing protocols put their messages inside IP packets. That is like writing a letter and using the postal system, even to reach your next-door neighbour. IS-IS does not use the post. It talks to the neighbour directly over the fence.

Because of this, nobody far away can post a fake letter to it. The message can only come from someone standing next to the fence.

*Where the analogy stops:* neighbours pass news along, so one router's news still reaches the whole network, hop by hop. Also, "over the fence" is not automatically safe. Someone on the same link can still lie, so authentication is still needed.

**City maps and the motorway map.** Imagine a country with many cities.

- A driver who stays in town needs only the city map. To leave town, the driver just heads to the nearest motorway junction.
- A long-distance driver needs the motorway map, which shows how all cities connect.
- A junction touches both worlds, so it holds both maps.

In IS-IS, the city map is **Level 1**, the motorway map is **Level 2**, and the junction is a **Level 1/Level 2 router**.

*Where the analogy stops:* a human driver can read signs and pick a smarter junction. A Level 1 router only picks the closest junction, even when another one gives a shorter total trip.

**A form with labelled boxes.** Each router publishes one sheet about itself: who its neighbours are and which networks it owns. The sheet is a form with labelled boxes. If a router gets a form with a box label it does not know, it skips that box but still passes the form on. This is why adding new features to IS-IS is easy.

## How It Works

### The language: OSI words for normal things

IS-IS was built for OSI (Open Systems Interconnection) networks, the ISO family of protocols that competed with TCP/IP. So it uses different words. Learn this table first and the rest gets much easier.

| IS-IS word | Plain meaning | Common IP word |
| --- | --- | --- |
| IS (Intermediate System) | A device that forwards traffic | Router |
| ES (End System) | A device at the edge | Host |
| Domain | All routers under one IS-IS design | Routing domain, like one AS |
| Circuit | A link on a router | Interface |
| SNPA (Subnetwork Point of Attachment) | The Layer 2 address on a link | MAC address |
| PDU (Protocol Data Unit) | One protocol message | Packet |
| LSP (Link State PDU) | One router's published sheet | Similar to OSPF LSAs |
| NET (Network Entity Title) | The router's OSI address | Router ID plus area |
| DIS (Designated Intermediate System) | The chosen leader on a LAN | Similar to OSPF DR |

Warning: **LSP has two meanings.** In IS-IS it means Link State PDU. In MPLS it means Label Switched Path. They are unrelated.

### A short history

DEC (Digital Equipment Corporation) designed IS-IS for its DECnet Phase V network. ISO then published it as ISO/IEC 10589, to route CLNP (Connectionless Network Protocol, the OSI version of IP).

At that time, people expected OSI to replace TCP/IP. It did not. RFC 1195 (1990) added IP support, and that version is called **Integrated IS-IS**. It can route CLNP, IP, or both. Today almost every network uses it for IP only.

So a modern IS-IS network uses OSI addresses to name its routers, but it forwards only IP traffic. No CLNP user data flows anywhere.

### It runs directly on Layer 2

Compare how each routing protocol travels on Ethernet:

```text
RIP:    Ethernet -> IP -> UDP port 520 -> RIP
BGP:    Ethernet -> IP -> TCP port 179 -> BGP
OSPF:   Ethernet -> IP protocol 89     -> OSPF
EIGRP:  Ethernet -> IP protocol 88     -> EIGRP
IS-IS:  Ethernet -> LLC                -> IS-IS

IS-IS has no IP header, no IP protocol number and no port.
```

LLC (Logical Link Control) is a small 3-byte header defined by IEEE 802.2. It sits after the Ethernet length field and says what is inside. An IS-IS frame on Ethernet looks like this:

```text
+----------+----------+--------+------+------+------+----------+-----+
| Dest MAC | Src MAC  | Length | DSAP | SSAP | Ctrl | IS-IS    | FCS |
| 6 bytes  | 6 bytes  | 2      | 0xFE | 0xFE | 0x03 | PDU      | 4   |
+----------+----------+--------+------+------+------+----------+-----+
                                <--- LLC, 3 bytes -->
```

- **DSAP and SSAP** (Destination and Source Service Access Point) are small labels for the upper protocol. The value 0xFE means OSI.
- **Length, not EtherType.** An IP frame has EtherType 0x0800 here. An IS-IS frame has a length value, then LLC.
- **Space left for IS-IS** on a 1500-byte link is 1497 bytes, because LLC uses 3 bytes.

IS-IS uses these destination MAC addresses:

| Destination MAC | Name | Used for |
| --- | --- | --- |
| 01:80:C2:00:00:14 | AllL1ISs | Level 1 messages on a LAN |
| 01:80:C2:00:00:15 | AllL2ISs | Level 2 messages on a LAN |
| 09:00:2B:00:00:05 | AllISs | Hellos on point-to-point links |

Running on Layer 2 has real effects:

1. **Harder to attack from far away.** A remote attacker cannot send IS-IS by routing a packet to the router. No IP packet can carry it to the control plane. The attacker needs access to a directly connected link.
2. **IP problems do not break the protocol transport.** IS-IS can carry IPv4 and IPv6 information without depending on either one. Adding IPv6 needs no second protocol.
3. **IP access lists do not see it.** A normal IP ACL (Access Control List) cannot permit or block IS-IS, because it is not IP.
4. **It cannot cross an IP-only path.** IS-IS does not cross a router hop by itself. Over an IP network, you need a tunnel that can carry non-IP frames, such as GRE (Generic Routing Encapsulation). Many cloud networks pass only IP, so plain IS-IS does not work between cloud VMs.

*Fuller truth:* "Layer 2" here means the message is carried directly in the data link frame. IS-IS is still a Layer 3 routing protocol in purpose, because it builds IP routes. Do not call it a "Layer 2 protocol" in an interview without saying this.

### NET addresses

Every IS-IS router needs a **NET**. A NET is a special form of NSAP (Network Service Access Point) address, the OSI equivalent of an IP address. It names the router, not an interface. A router has one NET no matter how many interfaces it has.

Here is a NET, split into parts:

```text
49.0001.0100.0000.0001.00
|_____| |____________| ||
   |          |         |
  Area    System ID    NSEL

Area       49.0001         1 to 13 bytes, here 3 bytes
System ID  0100.0000.0001  always 6 bytes in practice
NSEL       00              1 byte, always 00 in a NET
```

- **Area ID.** The first byte is the AFI (Authority and Format Identifier). AFI 49 means a private address, like RFC 1918 space in IPv4. The rest (0001) is your own area number. Everything to the left of the last 7 bytes is the area.
- **System ID.** It identifies the router, like an OSPF Router ID. It is 6 bytes (48 bits). It must be unique in the area, and good design keeps it unique across the whole domain.
- **NSEL** (NSAP Selector). It picks a service inside the node, a bit like a port number. In a NET it is always 00, which means "the router itself".

The dots are only for reading. Digits are hexadecimal, and each pair is one byte. The whole NET can be 8 to 20 bytes long.

**How people create a System ID.** The most common trick is to build it from the loopback IP, so it is easy to remember. Start from 192.168.10.1:

1. Pad each number to 3 digits: 192.168.010.001
2. Remove the dots: 192168010001
3. Regroup in 4s: 1921.6801.0001
4. Add the area and the NSEL: 49.0001.1921.6801.0001.00

This is a text trick, not a decimal to hex conversion. Others use the MAC address of the router. Any unique 6-byte value is fine.

*Analogy:* a NET is a postal address. The area is the town, the System ID is the house number, and 00 means "the house itself". *Where it stops:* a house has one postal address, and a router also has many IP addresses on its interfaces. The NET sits beside those IP addresses. It does not replace them.

### Level 1, Level 2 and routers

IS-IS has a two-level design:

- **Level 1 (L1) routing** is inside one area.
- **Level 2 (L2) routing** is between areas. All L2 routers together form the backbone, called the **L2 subdomain**.

A router can be one of three types:

| Type | Knows | Keeps | Like in OSPF |
| --- | --- | --- | --- |
| L1 | Its own area only | One L1 LSDB | Internal router in a totally stubby area |
| L2 | The backbone | One L2 LSDB | Backbone router |
| L1/L2 | Both | Two separate LSDBs | ABR (Area Border Router) |

LSDB means Link State Database, the full map each router keeps.

The default on Cisco IOS is L1/L2. It works, but it means two databases and double the hellos on each link. Good designs set the type on purpose.

```text
          Level 2 backbone (joins all areas)
   +--------------------------------------------------+
   |  [R2 L1/L2] ======= [R5 L2] ======= [R3 L1/L2]   |
   +------|-----------------------------------|-------+
          |                                   |
   +------|----------+              +---------|-------+
   |   [R1 L1]       |              |      [R4 L1]    |
   |   Area 49.0001  |              |   Area 49.0002  |
   +-----------------+              +-----------------+
```

The biggest difference from OSPF is **where the area border sits**.

```text
OSPF: the border runs through the router.

   Area 10          |          Area 0
   [R1] ------- [ ABR router ] ------- [R3]
                    |

IS-IS: the border runs on the link.

   Area 49.0001         |      Area 49.0002
   [R1] ------- [R2] ---+--- [R3] ------- [R4]
                        |
```

An IS-IS router sits fully inside one area. It does not have one interface in area A and another in area B. The link between two L2 neighbours in different areas is the border.

This also means the backbone is not one special area number. OSPF has Area 0. IS-IS has no Area 0. The backbone is any connected chain of L2-capable routers, and that chain can pass through many areas. The chain must stay connected, because there is no virtual link in IS-IS to fix a broken backbone.

### Who becomes neighbours

| Router A | Router B | Same area? | Adjacency formed |
| --- | --- | --- | --- |
| L1 | L1 | Yes | L1 |
| L1 | L1 | No | None |
| L1 | L2 | Any | None |
| L1 | L1/L2 | Yes | L1 |
| L1 | L1/L2 | No | None |
| L2 | L2 | Any | L2 |
| L2 | L1/L2 | Any | L2 |
| L1/L2 | L1/L2 | Yes | L1 and L2 (two adjacencies) |
| L1/L2 | L1/L2 | No | L2 |

An adjacency is a working neighbour relationship between two routers. The rule is simple: **L1 needs the same area. L2 does not care about the area.**

### The message types

| PDU | Full name | Job | PDU type numbers |
| --- | --- | --- | --- |
| IIH | IS-IS Hello | Find and keep neighbours | 15 L1 LAN, 16 L2 LAN, 17 point-to-point |
| LSP | Link State PDU | Carry a router's links and prefixes | 18 L1, 20 L2 |
| CSNP | Complete Sequence Number PDU | List all LSPs in the database | 24 L1, 25 L2 |
| PSNP | Partial Sequence Number PDU | Ask for LSPs or acknowledge them | 26 L1, 27 L2 |

A sequence number is a counter on each LSP. A higher number means newer information. CSNPs and PSNPs are short lists of "LSP ID, sequence number, lifetime, checksum". They carry no routes, only a list of what exists.

Every PDU starts with the same 8-byte common header:

| Byte | Field | Typical value |
| --- | --- | --- |
| 1 | Intradomain Routing Protocol Discriminator | 0x83 for IS-IS |
| 2 | Length Indicator | Header length in bytes |
| 3 | Version / Protocol ID Extension | 1 |
| 4 | ID Length | 0, which means 6 bytes |
| 5 | PDU Type (low 5 bits) | 15 to 27 |
| 6 | Version | 1 |
| 7 | Reserved | 0 |
| 8 | Maximum Area Addresses | 0, which means 3 |

After the header come some fields for that PDU type, and then the TLVs.

### TLVs: why IS-IS is easy to extend

A TLV (Type, Length, Value) is one labelled box of data:

```text
+--------+--------+---------------------+
| Type   | Length | Value               |
| 1 byte | 1 byte | 0 to 255 bytes      |
+--------+--------+---------------------+
```

A router that does not understand a TLV skips it using the Length field. It still floods the whole LSP to its neighbours. So new features can roll out one router at a time.

Common TLVs:

| TLV | Meaning | Defined for |
| --- | --- | --- |
| 1 | Area addresses | Base protocol |
| 6 | IS neighbours on LAN (their MACs) | Base protocol |
| 9 | LSP entries, used in CSNP and PSNP | Base protocol |
| 10 | Authentication | Base protocol |
| 22 | Extended IS reachability, wide metric | RFC 5305 |
| 128 | IPv4 reachability, narrow metric | RFC 1195 |
| 132 | IPv4 interface address | RFC 1195 |
| 135 | Extended IPv4 reachability, wide | RFC 5305 |
| 137 | Dynamic hostname | RFC 5301 |
| 236 | IPv6 reachability | RFC 5308 |
| 240 | Point-to-point three-way adjacency | RFC 5303 |
| 242 | Router capability | RFC 7981 |

### Step by step: from power on to routes

**Step 1: Form adjacencies.** On a LAN (broadcast link), this takes three steps:

1. R1 sends a hello to the multicast MAC. The hello lists the neighbours R1 has heard.
2. R2 hears it and moves R1 to the **Init** state. R2 adds R1's MAC to TLV 6 in its own hello.
3. R1 sees its own MAC in R2's hello, so R1 knows the link works both ways. R1 moves R2 to **Up**. R1's next hello lets R2 do the same.

On a point-to-point link, RFC 5303 gives the same three-way check using TLV 240.

Hellos must agree on a few things: the System ID length, the maximum area addresses, the level, the area (for L1), and the authentication. On Cisco, the IP addresses must also be in the same subnet. **Hello and hold timers do not have to match**, which is different from OSPF.

**Step 2: Exchange databases.** The routers compare what they have. On a point-to-point link, each side sends a CSNP once when the link comes up. Each side then sends the missing LSPs.

**Step 3: Flood LSPs.** Each router creates its own LSP. Then every router copies it, unchanged, to all other neighbours in that level. L1 LSPs stay inside the area. L2 LSPs stay inside the backbone.

**Step 4: Run SPF.** Each router runs the SPF (Shortest Path First) algorithm, also called Dijkstra's algorithm, on each LSDB. The result is a tree of best paths, with the router at the root.

**Step 5: Install routes.** The best paths go into the routing table. On Cisco, IS-IS routes have administrative distance 115. See [The Routing Table](02-routing-table.md) for how that number is used.

**Step 6: Keep it fresh.** Each LSP has a remaining lifetime. Each router refreshes its own LSPs before they reach zero.

### LSP identity and lifetime

An LSP ID has three parts:

```text
 0100.0000.0001 . 00 - 00
 |____________|   |    |
   System ID      |    Fragment number (0-255)
                  Pseudonode ID (00 = a real router)
```

With hostnames, this reads as `R1.00-00`.

A router advertises everything in one LSP. If the data does not fit in one frame, the router splits it into **fragments** (`R1.00-00`, `R1.00-01`, and so on). One router can have up to 256 fragments.

LSP timers in the standard:

| Item | Value | Note |
| --- | --- | --- |
| Maximum lifetime | 1200 seconds | Counts down to 0 |
| Refresh | 900 seconds | Router resends before expiry |
| Maximum configurable lifetime | 65535 seconds | The field is 16 bits, about 18 hours |

When an LSP must be removed, a router sends a **purge**: the LSP header with lifetime 0, so all routers delete it.

### LANs: DIS and the pseudonode

On an Ethernet segment with many routers, every router would otherwise show a link to every other router. Five routers would need 10 links in every database. To avoid this, IS-IS treats the LAN as one fake router called a **pseudonode**.

```text
 Real wires:                    What SPF sees:

  R1    R2    R3                 R1    R2    R3
   |     |     |                   \    |    /
 --+-----+-----+-- LAN              [pseudonode]
                                      R3.01
```

The DIS is the router that speaks for the pseudonode. The DIS does two jobs:

1. It creates the pseudonode LSP, such as `R3.01-00`. Here 01 is a number the DIS picks for that LAN.
2. It sends a CSNP every 10 seconds (Cisco default) so all routers can check their database.

**Election.** The highest priority wins. The range is 0-127 and the default is 64. If there is a tie, the highest MAC address wins. Each level has its own election, so L1 and L2 can have different DIS routers on one LAN.

**How flooding works on a LAN.** LSPs are multicast and nobody acknowledges them. Reliability comes from the CSNP:

1. The DIS lists all LSPs it has.
2. Each router compares the list with its own database.
3. If a router is missing an LSP or has an older one, it sends a PSNP to request it.
4. If a router has a newer one, it floods that LSP.

*Analogy:* a teacher reads the class register every 10 seconds. A student who has no copy of page 12 raises a hand. *Where it stops:* any student who has a newer page shares it right away, without waiting to be asked.

**On a point-to-point link** there is no DIS and no pseudonode. Each LSP is sent directly and acknowledged with a PSNP. It is resent if no acknowledgement arrives.

DIS compared with the OSPF DR (Designated Router):

| Point | IS-IS DIS | OSPF DR |
| --- | --- | --- |
| Backup | None, a new election runs if it fails | BDR (Backup DR) waits ready |
| Preemption | Yes, a better router takes over | No, the DR stays until it fails |
| Adjacencies on the LAN | All routers with all routers | DROthers only reach Full with DR and BDR |
| Priority 0 | Still can be elected | Never elected |
| Hello speed | DIS sends 3 times faster | Same as others |

### Metrics: narrow and wide

The metric is the cost of a link. SPF picks the path with the lowest total.

| Style | TLVs | Interface metric | Total path |
| --- | --- | --- | --- |
| Narrow (original) | 2, 128, 130 | 6 bits, 1-63 | 10 bits, up to 1023 |
| Wide (RFC 5305) | 22, 135 | 24 bits, up to 16777215 | 32 bits for prefixes |

Narrow metrics are too small for modern links, and wide metrics also carry traffic engineering data. Every modern network should use wide metrics.

Defaults vary, so always check them:

- Classic Cisco IOS defaults to **narrow** metrics and gives **every interface metric 10**, whatever its speed. A 1 Gbps link and a 100 Gbps link look the same.
- Junos also uses 10, but 0 on the loopback. It can calculate from bandwidth if you set a `reference-bandwidth`.
- FRRouting (open-source routing software for Linux) defaults to wide metrics.

ISPs usually set metrics by hand, often based on bandwidth or distance.

### Level 1 traffic leaving the area

An L1 router has no routes to other areas. It finds the exit like this:

1. An L1/L2 router sees another area through its L2 database.
2. It sets the **ATT (Attached) bit** in its L1 LSP.
3. L1 routers see the ATT bit and install a default route to the nearest L1/L2 router.

For the other direction:

- **L1 to L2:** an L1/L2 router copies L1 prefixes into its L2 LSP by default.
- **L2 to L1:** nothing is copied by default. L1 gets only the ATT bit.

So a default L1 area behaves much like an OSPF totally stubby area. One difference: an L1 router can still redistribute external routes into the area.

### Comparison with OSPF

| Topic | IS-IS | OSPF |
| --- | --- | --- |
| Family | Link state, Dijkstra SPF | Link state, Dijkstra SPF |
| Standard | ISO/IEC 10589, RFC 1195 for IP | RFC 2328 (OSPFv2), RFC 5340 (OSPFv3) |
| Transport | Directly on Layer 2 | IP protocol 89 |
| Router identity | 6-byte System ID in a NET | 32-bit Router ID |
| Area border | On the link | Inside the router (ABR) |
| Backbone | Connected chain of L2 routers | Area 0 |
| Virtual links | No | Yes |
| Area types | L1 and L2 only | Standard, stub, totally stubby, NSSA |
| Database items | One LSP per router (plus fragments) | Many LSA (Link State Advertisement) types |
| Messages | IIH, LSP, CSNP, PSNP | Hello, DBD, LSR, LSU, LSAck |
| Link types | Broadcast, point-to-point | Broadcast, point-to-point, NBMA, others |
| LAN leader | DIS, no backup, preemptive | DR and BDR, not preemptive |
| Cisco default metric | 10 on every link | Based on bandwidth |
| LSA or LSP age | Counts down from 1200 s | Counts up to 3600 s |
| Refresh | 900 s, can be much longer | 1800 s, fixed by the standard |
| Hello timers must match | No | Yes |
| IPv4 and IPv6 | One process carries both | OSPFv2 for IPv4, OSPFv3 for IPv6 |
| New features | New TLVs | New LSA types or opaque LSAs |
| Cisco admin distance | 115 | 110 |
| Common home | ISPs, some data centres | Enterprises |

*Fuller truth:* OSPFv3 can carry IPv4 too (RFC 5838), and it also moved prefixes out of Router LSAs. OSPF is not "worse". Many gaps closed over the years.

### Why ISPs like it

1. **One protocol for IPv4 and IPv6.** New address families are only new TLVs. An ISP does not run two parallel protocols.
2. **Easy to extend.** Traffic engineering, Segment Routing and Flexible Algorithm all fit in as TLVs. Routers that do not know a TLV still flood it.
3. **Handles big flat networks.** Many ISPs run one single Level 2 domain, with no areas at all. This is simple and it keeps all loopbacks visible everywhere, which MPLS needs.
4. **Stable.** LSP lifetimes can be raised to about 18 hours, so there is little refresh traffic.
5. **Cheaper updates.** Prefixes are leaves on the SPF tree. If only a prefix changes, the router runs PRC (Partial Route Calculation), not a full SPF.
6. **Harder to attack.** No IP means no remote path to the routing process.
7. **Works with MPLS and BGP-free cores.** Routers in the core only need loopbacks and link routes from IS-IS. See [MPLS](13-mpls.md).

History also matters. In the 1990s, several big ISPs chose IS-IS, and the skills and tools stayed. Modern OSPF implementations also scale well. An honest answer in an interview is: "IS-IS has real technical advantages, and the ISP habit also keeps it popular."

## Example

### Topology

Two areas, each with an L1 router and an L1/L2 router. The hosts H1 and H4 are ordinary Linux or Windows machines.

```text
 H1 192.168.1.10                          H4 192.168.4.10
  |                                            |
  | 192.168.1.1                    192.168.4.1 |
 [R1] ---------- [R2] =========== [R3] ---------- [R4]
  L1    L1 link  L1/L2  L2 link   L1/L2  L1 link   L1

 |---- Area 49.0001 ----|         |---- Area 49.0002 ----|
```

| Router | Loopback0 | NET | Type |
| --- | --- | --- | --- |
| R1 | 10.0.0.1/32 | 49.0001.0100.0000.0001.00 | L1 |
| R2 | 10.0.0.2/32 | 49.0001.0100.0000.0002.00 | L1/L2 |
| R3 | 10.0.0.3/32 | 49.0002.0100.0000.0003.00 | L1/L2 |
| R4 | 10.0.0.4/32 | 49.0002.0100.0000.0004.00 | L1 |

| Link | Left side | Right side |
| --- | --- | --- |
| R1-R2 | R1 Gi0/0 10.1.12.1/30 | R2 Gi0/0 10.1.12.2/30 |
| R2-R3 | R2 Gi0/1 10.1.23.1/30 | R3 Gi0/1 10.1.23.2/30 |
| R3-R4 | R3 Gi0/0 10.1.34.1/30 | R4 Gi0/0 10.1.34.2/30 |

### Configuration (Cisco IOS)

R1, an L1 router:

```text
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.12.1 255.255.255.252
 ip router isis CORE
 isis network point-to-point
!
interface GigabitEthernet0/1
 ip address 192.168.1.1 255.255.255.0
!
router isis CORE
 net 49.0001.0100.0000.0001.00
 is-type level-1
 metric-style wide
 passive-interface Loopback0
 passive-interface GigabitEthernet0/1
```

R2, an L1/L2 router:

```text
interface Loopback0
 ip address 10.0.0.2 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.1.12.2 255.255.255.252
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-1
!
interface GigabitEthernet0/1
 ip address 10.1.23.1 255.255.255.252
 ip router isis CORE
 isis network point-to-point
 isis circuit-type level-2-only
!
router isis CORE
 net 49.0001.0100.0000.0002.00
 is-type level-1-2
 metric-style wide
 passive-interface Loopback0
```

What each line does:

- `router isis CORE` starts IS-IS. `CORE` is a local name only. It does not need to match on neighbours.
- `net` sets the area and the System ID.
- `ip router isis CORE` turns IS-IS on for that interface.
- `isis network point-to-point` skips DIS election on this two-router link. It must be set on both ends.
- `isis circuit-type` limits the hellos on that link to one level.
- `passive-interface` advertises the network but sends no hellos, which suits loopbacks and host LANs.

R3 mirrors R2, and R4 mirrors R1, using the addresses in the tables.

### Check the neighbours

```text
R2#show isis neighbors

Tag CORE:
System Id  Type Interface  IP Address      State Holdtime Circuit Id
R1         L1   Gi0/0      10.1.12.1       UP    28       00
R3         L2   Gi0/1      10.1.23.2       UP    25       00
```

R2 has an L1 neighbour in its own area and an L2 neighbour in the other area. This matches the adjacency table.

### Check the database

```text
R1#show isis database

Tag CORE:
IS-IS Level-1 Link State Database:
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime   ATT/P/OL
R1.00-00            * 0x00000004   0x7A1C        1043           0/0/0
R2.00-00              0x00000005   0x3D92        987            1/0/0
```

- The `*` marks the router's own LSP.
- **LSP Holdtime** is the remaining lifetime, counting down from 1200.
- **ATT is 1 for R2.** R2 tells R1, "I reach other areas."
- R1 sees only its own area. R3 and R4 are not in this database.

```text
R1#show isis database R2.00-00 detail

IS-IS Level-1 LSP R2.00-00
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime   ATT/P/OL
R2.00-00              0x00000005   0x3D92        987            1/0/0
  Area Address: 49.0001
  NLPID:        0xCC
  Hostname: R2
  IP Address:   10.0.0.2
  Metric: 10         IS-Extended R1.00
  Metric: 10         IP 10.0.0.2/32
  Metric: 10         IP 10.1.12.0/30
```

NLPID (Network Layer Protocol ID) 0xCC means IPv4. The address 10.1.23.0/30 is not here, because R2 runs only L2 on Gi0/1. That prefix is in R2's L2 LSP.

### Check the routes

R1 has only one way out:

```text
R1#show ip route isis
Codes: i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1
       L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default

Gateway of last resort is 10.1.12.2 to network 0.0.0.0

i*L1  0.0.0.0/0 [115/10] via 10.1.12.2, 00:05:12, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
i L1  10.0.0.2/32 [115/20] via 10.1.12.2, 00:05:12, GigabitEthernet0/0
```

The default route comes from the ATT bit. Its metric of 10 is the cost to reach R2. The metric to 10.0.0.2/32 is 20: link metric 10 plus loopback metric 10.

R2 sees everything:

```text
R2#show ip route isis
      10.0.0.0/8 is variably subnetted, 9 subnets, 2 masks
i L1  10.0.0.1/32 [115/20] via 10.1.12.1, 00:06:40, GigabitEthernet0/0
i L2  10.0.0.3/32 [115/20] via 10.1.23.2, 00:06:02, GigabitEthernet0/1
i L2  10.0.0.4/32 [115/30] via 10.1.23.2, 00:06:02, GigabitEthernet0/1
i L2  10.1.34.0/30 [115/20] via 10.1.23.2, 00:06:02, GigabitEthernet0/1
i L1  192.168.1.0/24
        [115/20] via 10.1.12.1, 00:06:40, GigabitEthernet0/0
i L2  192.168.4.0/24
        [115/30] via 10.1.23.2, 00:06:02, GigabitEthernet0/1
```

Check one number by hand. To reach 192.168.4.0/24:

```text
R4 advertises the LAN with metric          10
R3 adds its link to R4                    +10  = 20 in R3's L2 LSP
R2 adds its link to R3                    +10  = 30
```

### Test end to end

Linux on H1:

```bash
traceroute -n 192.168.4.10
```

```text
traceroute to 192.168.4.10 (192.168.4.10), 30 hops max, 60 byte packets
 1  192.168.1.1   0.412 ms  0.388 ms  0.371 ms
 2  10.1.12.2     1.204 ms  1.188 ms  1.150 ms
 3  10.1.23.2     2.031 ms  2.016 ms  1.994 ms
 4  10.1.34.2     2.860 ms  2.841 ms  2.822 ms
 5  192.168.4.10  3.455 ms  3.437 ms  3.419 ms
```

Windows on H1:

```powershell
tracert -d 192.168.4.10
```

The hops are the same. Hosts do not know IS-IS exists. They only send to their default gateway.

### See the frames

The links above are point-to-point, so this capture comes from a separate lab LAN with two L2 routers. Linux, using tcpdump (the `isis` filter word is built in):

```bash
sudo tcpdump -i eth0 -nn -e isis
```

```text
12:00:01.123456 52:54:00:12:00:05 > 01:80:c2:00:00:15, 802.3,
length 1514: LLC, dsap OSI (0xfe) Individual, ssap OSI (0xfe)
Command, ctrl 0x03: OSI NLPID IS-IS (0x83): L2 Lan IIH,
src-id 0100.0000.0005, lan-id 0100.0000.0006.01, prio 64,
length 1497
```

tcpdump prints this as one long line. It says `802.3` and `LLC`, and no IP addresses appear. It is a padded L2 LAN hello. The frame is 1514 bytes: 14 bytes of Ethernet header, 3 bytes of LLC and 1497 bytes of IS-IS.

On Windows, use Wireshark with the display filter `isis`. Windows does not run IS-IS itself, but it can capture it from a mirror port.

### Running IS-IS on Linux and Junos

FRRouting includes a daemon called `isisd`. Turn it on with `isisd=yes` in `/etc/frr/daemons`, then use this config:

```text
! FRRouting, /etc/frr/frr.conf
interface lo
 ip router isis CORE
 isis passive
!
interface eth0
 ip router isis CORE
 isis network point-to-point
!
router isis CORE
 net 49.0001.0100.0000.0005.00
 is-type level-2-only
 metric-style wide
```

Check it with `sudo vtysh -c "show isis neighbor"`. On Junos, the classic gotcha is that each IS-IS interface also needs `family iso`.


## Going Deeper

### Suboptimal routing and route leaking

The ATT bit is simple, but it hides information. Look at this network:

```text
         Area 49.0010                  Area 49.0020
                   metric 50
   +----- [R6] ======================= +
   |  10                               |
 [R5]                                 [R9] --- 172.16.9.0/24
   |  20                               |       (metric 10)
   +----- [R7] ======================= +
                   metric 10
```

R5 is an L1 router. It only compares the cost to its exits:

```text
Exit    Cost R5 sees    Real total to 172.16.9.0/24
R6      10              10 + 50 + 10 = 70
R7      20              20 + 10 + 10 = 40
```

R5 picks R6 and pays 70. The better path costs 40. This is **suboptimal routing**. Return traffic can also take a different path, which makes troubleshooting harder.

The fix is **route leaking**: copying chosen L2 prefixes down into L1 (RFC 5302). On both R6 and R7 (Cisco IOS):

```text
ip prefix-list LEAK permit 172.16.9.0/24
!
route-map L2-INTO-L1 permit 10
 match ip address prefix-list LEAK
!
router isis CORE
 redistribute isis ip level-2 into level-1 route-map L2-INTO-L1
```

R5 now sees the specific prefix as inter-area (`ia`), with next hop R7 at 10.5.7.2:

```text
i ia  172.16.9.0/24 [115/40] via 10.5.7.2, 00:01:03, GigabitEthernet0/2
```

A longer prefix always beats the default route, so traffic uses R7.

**Loop prevention.** A leaked route gets the **up/down bit** set. This bit says, "this prefix came down from L2, do not send it back up." Without it, a leaked route could go back into L2 and create a loop.

### The ATT bit trap

An L1/L2 router sets ATT only if it can reach **another area**. If all its L2 neighbours are in its own area, ATT stays 0, and L1 routers get no default route.

### Single-level design

Most big ISPs do not use areas. They set every router to L2 only:

```text
router isis CORE
 net 49.0001.0100.0000.0002.00
 is-type level-2-only
 metric-style wide
```

Reasons:

- No suboptimal routing from the ATT default.
- Every router sees every /32 loopback, which MPLS LDP and BGP next hops depend on. Summarising loopbacks can break label paths.
- Modern routers have plenty of CPU and memory for one large database.

Why L2 and not L1? L2 leaves room to grow. You can add L1 areas later without changing the core.

### Multiple NETs and area migration

A router can hold up to 3 area addresses by default. With the same System ID, it treats them as one area. This lets you rename or merge areas with no outage:

```text
router isis CORE
 net 49.0001.0100.0000.0002.00
 net 49.0099.0100.0000.0002.00
```

Add the new NET on all routers, check the adjacencies, then remove the old NET. OSPF has no direct equivalent to this trick.

### The overload bit

A router can set the **OL (Overload) bit** in its LSP. It means, "do not send transit traffic through me." Other routers still reach prefixes directly attached to it, but they stop using it as a path to somewhere else.

It was first meant for routers out of memory. Today its main use is at boot. An IS-IS router comes up faster than BGP, so it may attract traffic before it has BGP routes, and that traffic gets dropped. The fix on Cisco:

```text
router isis CORE
 set-overload-bit on-startup wait-for-bgp
```

You can also give a time, such as `set-overload-bit on-startup 300`. Engineers also set the bit by hand before maintenance. The OSPF version of this idea is stub router (RFC 6987).

### Hello padding and MTU

MTU (Maximum Transmission Unit) is the largest payload a link will carry. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

IS-IS frames are not fragmented on the way, so every LSP must fit on every link. Because of this, many implementations pad hellos up to the full link MTU with TLV 8. If padded hellos cannot pass, the adjacency does not form. This finds an MTU problem early, instead of letting LSPs vanish later.

Padding also wastes some bandwidth on slow links. Cisco offers `no hello padding` under the router process.

Cisco IOS defaults to an LSP size of 1497 bytes, and some other platforms use 1492. In a mixed network, set the LSP size to fit the smallest link in the domain, using `lsp-mtu` on Cisco.

### Timers

| Item | Cisco IOS default | Note |
| --- | --- | --- |
| Hello, normal router | 10 s | Junos uses 9 s |
| Hello multiplier | 3 | Hold time is 30 s |
| Hello, DIS | About 3.3 s | Hold time is 10 s |
| CSNP on LAN | 10 s | From the DIS |
| LSP lifetime | 1200 s | `max-lsp-lifetime` |
| LSP refresh | 900 s | `lsp-refresh-interval` |

ISPs often raise the lifetime near 65535 s, with refresh a bit lower, such as 65000 s. Refresh must stay below the lifetime.

For fast failure detection, use BFD (Bidirectional Forwarding Detection), not tiny hello timers. BFD is a light protocol built only to check that a path is alive.

```text
router isis CORE
 bfd all-interfaces
```

Three tuning knobs slow down work during a storm of changes, using exponential backoff (wait a little at first, then longer if changes continue): `spf-interval`, `prc-interval` and `lsp-gen-interval`. For example, `spf-interval 5 50 200` means a maximum wait of 5 seconds, a first wait of 50 ms and a second wait of 200 ms. That is an example, not a default.

### What happens when a link fails

1. The link goes down, or BFD reports a failure. Without either signal, the hold timer must expire, up to 30 s by default.
2. The router builds a new LSP with a higher sequence number and floods it. Every router floods it onward.
3. Every router runs SPF, after a short wait.
4. Routers update their RIB (Routing Information Base, the routing table) and FIB (Forwarding Information Base, the table used by the hardware).

For a much faster repair, routers can pre-calculate backups with LFA (Loop-Free Alternate) or TI-LFA (Topology Independent LFA), see [Advanced Routing Techniques](12-advanced-routing-techniques.md).

### IPv6 and multi-topology

RFC 5308 adds IPv6 with TLV 236 for prefixes and TLV 232 for interface addresses. In **single-topology** mode, IPv4 and IPv6 share one SPF tree. This means every IS-IS link must run both. If one link has only IPv4, IPv6 traffic can be sent into a black hole.

**Multi-topology** (RFC 5120) runs a separate SPF tree for IPv6, using TLVs 222, 229, 235 and 237. Links can then differ.

```text
ipv6 unicast-routing
!
interface GigabitEthernet0/0
 ipv6 address 2001:db8:12::1/64
 ipv6 router isis CORE
!
router isis CORE
 metric-style wide
 address-family ipv6
  multi-topology
 exit-address-family
```

### Narrow to wide migration

Changing all routers at once causes an outage, because each style ignores the other. On Cisco, first set `metric-style transition` on all routers, so they send and accept both. Then set `metric-style wide`, one router at a time.

### Authentication

IS-IS can authenticate three scopes: hellos on a link (set on the interface), L1 LSPs and SNPs (area scope, set under the router) and L2 LSPs and SNPs (domain scope, set under the router). SNP means Sequence Number PDU, so a CSNP or PSNP.

Options are clear text (avoid it), HMAC-MD5 (RFC 5304) and generic cryptographic authentication with SHA family hashes (RFC 5310), where the platform supports it.

```text
key chain ISIS-KEYS
 key 1
  key-string Example-Only-Key
!
interface GigabitEthernet0/1
 isis authentication mode md5
 isis authentication key-chain ISIS-KEYS
!
router isis CORE
 authentication mode md5
 authentication key-chain ISIS-KEYS
```

### Smaller details worth knowing

- **Flooding flags.** Each router keeps two flags per LSP, per interface. SRM (Send Routing Message) means "still send this LSP on this link". SSN (Send Sequence Number) means "still acknowledge it on this link".
- **Mesh groups** (RFC 2973) limit reflooding on full-mesh links. In dense fabrics, the same problem led to **dynamic flooding**, where only a chosen subset of links carries the flood.
- **Purges.** In OSPF, normally only the originator flushes its LSA. In IS-IS, other routers can purge in some cases, and a faulty router can cause a purge storm. RFC 6232 adds a TLV that names who started the purge.
- **Hiding link prefixes.** Cisco IOS XE offers `advertise passive-only` under `router isis`. Only passive interface prefixes (usually loopbacks) are advertised, so the database is smaller and core link addresses are less exposed.
- **Network types.** IS-IS supports only broadcast and point-to-point. There is no NBMA (Non-Broadcast Multi-Access) mode. On Ethernet links between two routers, operators nearly always use point-to-point mode.
- **Broken areas.** The ISO standard defines optional L1 partition repair, but vendors rarely implement it. Treat a split area as a real outage.

### IS-IS, MPLS and Segment Routing

IS-IS gives MPLS the topology. RSVP-TE (Resource Reservation Protocol, Traffic Engineering) uses wide-metric TLVs that carry bandwidth and link attributes.

Segment Routing goes further. It carries labels (called SIDs, Segment Identifiers) directly in IS-IS, so LDP (Label Distribution Protocol) is no longer needed. RFC 8667 covers the MPLS version, and RFC 9352 covers SRv6 (Segment Routing over IPv6). Flexible Algorithm (RFC 9350) lets one network calculate more than one kind of best path, such as lowest latency. See [MPLS](13-mpls.md).

### IS-IS in the data centre

In a leaf-spine fabric, IS-IS can provide the underlay, the plain IP reachability between switches. [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) then run on top. The common alternatives are eBGP (described in RFC 7938) and OSPF.

IS-IS is simple to run on a leaf-spine fabric, but flooding in a dense fabric is a real cost. Large operators often pick eBGP for its policy control and its limited blast radius. Vendor fabrics use IS-IS where its TLV design and IP-independence are handy.

## Common Mistakes

- **Saying IS-IS runs over IP.** It has no IP protocol number and no port. It runs directly in the Layer 2 frame.
- **Thinking CLNP traffic must flow.** Only the NET and the frame format are OSI. User traffic is plain IP.
- **Giving each interface a NET.** The NET belongs to the router. Interfaces only have IP addresses.
- **Putting a router in two areas by interface.** That is OSPF thinking. The border is on the link.
- **Setting NSEL to a non-zero value in a NET.** A NET always ends in 00.
- **Duplicate System IDs.** Two routers keep overwriting each other's LSPs, which causes flapping. Cisco logs a "possible duplicate system ID" message.
- **Expecting L1 neighbours across areas.** L1 needs the same area. Across areas, only L2 forms.
- **Expecting L2 routes in L1 by default.** L1 gets only the ATT default route, unless you leak.
- **Mixing narrow and wide routers.** They silently ignore each other's TLVs. Adjacencies stay up, but routes go missing.
- **Believing Cisco metrics follow bandwidth.** Every link is 10 by default, so a slow link can carry the traffic.
- **Setting `isis network point-to-point` on one end only.** LAN hellos and point-to-point hellos do not match, so no adjacency forms.
- **Looking for a backup DIS.** There is none. A new election runs when the DIS leaves.
- **Thinking priority 0 stops a router from being DIS.** That is OSPF. In IS-IS, priority 0 is only the lowest value.
- **Ignoring MTU.** If padded hellos do not pass, the adjacency stays down, and the interface still looks fine.
- **Trying to filter IS-IS with an IP ACL.** IP ACLs cannot see it. Control it with interface configuration and authentication.
- **Calling IS-IS a "Layer 2 protocol".** It uses Layer 2 as transport, but it builds Layer 3 routes.

## Interview Questions

**Q: What is IS-IS and where is it used?**

A: IS-IS is a link-state IGP that uses Dijkstra SPF, the same family as OSPF. It was built for OSI and extended for IP by RFC 1195. It is mainly used in ISP backbones, often with MPLS or Segment Routing. It also appears in some data centre underlays and in Layer 2 fabric protocols such as TRILL and SPB.

**Q: How does IS-IS travel on Ethernet, and why does it matter?**

A: It runs directly on Layer 2 in an 802.3 frame with an LLC header, using DSAP and SSAP 0xFE. There is no IP header. So a remote attacker cannot reach the routing process with an IP packet, IP ACLs do not see it, and it needs no IP to carry IPv4 and IPv6 data. The downside is that it cannot cross an IP-only path without a tunnel such as GRE.

**Q: Explain the NET 49.0001.0100.0000.0001.00.**

A: 49.0001 is the area, where 49 is the private AFI. 0100.0000.0001 is the 6-byte System ID, which is unique to the router. The final 00 is the NSEL, and in a NET it is always 00, meaning the router itself. This System ID looks built from loopback 10.0.0.1, which is a common habit.

**Q: What are Level 1, Level 2 and L1/L2 routers?**

A: L1 routers know only their own area and send other traffic to the nearest L1/L2 router by a default route. L2 routers form the backbone between areas. L1/L2 routers do both and keep two separate databases. They are like OSPF ABRs, but the area border in IS-IS sits on the link, not inside the router.

**Q: How does an L1 router reach another area?**

A: An L1/L2 router that sees another area through L2 sets the ATT bit in its L1 LSP. L1 routers then install a default route to the nearest router with ATT set. Because they only look at the nearest exit, this can be suboptimal. Route leaking from L2 into L1 fixes that, and the up/down bit stops leaked routes from looping back into L2.

**Q: What are the main differences between IS-IS and OSPF?**

A: Both are link state and use SPF. IS-IS runs on Layer 2, while OSPF runs on IP protocol 89. The IS-IS border is on the link, while the OSPF border is in the ABR. IS-IS has no Area 0, just a connected L2 backbone. IS-IS carries IPv4 and IPv6 in one process using TLVs, while OSPFv2 needs OSPFv3 for IPv6. IS-IS has a DIS with no backup and preemption, while OSPF has a DR and BDR without preemption.

**Q: Why do ISPs like IS-IS?**

A: First, TLVs make it easy to add features, and unknown TLVs are still flooded. Second, one process carries IPv4 and IPv6. Third, it runs a big flat Level 2 network well, and long LSP lifetimes and partial route calculation keep it calm. Fourth, it is harder to attack since it is not on IP. Also, it has a long history in ISPs, so the skills and tools already exist there.

**Q: What does the DIS do, and how is it different from an OSPF DR?**

A: The DIS creates the pseudonode LSP for the LAN and sends CSNPs every 10 seconds so routers can sync. It is chosen by highest priority, then highest MAC. Unlike a DR, it has no backup, it can be preempted, and all routers on the LAN stay fully adjacent to each other. Hellos from the DIS are also three times faster.

**Q: What are CSNP and PSNP?**

A: A CSNP is a full list of LSPs in the database. A PSNP is a short list, used to request LSPs or to acknowledge them. On a LAN, the DIS sends CSNPs and routers use PSNPs to ask for what they miss. On a point-to-point link, a CSNP is sent at start-up and each LSP is acknowledged with a PSNP.

**Q: What are narrow and wide metrics?**

A: Narrow metrics use 6 bits per interface, so 63 at most, and 1023 for a whole path. Wide metrics from RFC 5305 use 24 bits per link and carry traffic engineering information. All routers must agree, or they ignore each other's routes. To migrate, use `metric-style transition` everywhere, then move to wide.

**Q: What is the overload bit used for?**

A: It tells other routers not to use this router for transit traffic, although its directly attached prefixes stay reachable. The most common use is at boot, with `set-overload-bit on-startup wait-for-bgp`, so the router does not attract traffic before BGP is ready. It is also useful before maintenance.

**Q: Two IS-IS routers are connected but do not form an adjacency. What do you check?**

A: I check the link is up and the interface is enabled for IS-IS. Then the level and area: L1 needs the same area, and an L1 and an L2 router will not pair. Then the link type on both ends, the same subnet, the MTU because of hello padding, the authentication, and the System ID length. `show clns neighbors`, `show isis neighbors` and `debug isis adj-packets` show where it stops. Hello timers do not need to match.

## Quick Recap

- IS-IS is a link-state IGP, run directly on Layer 2 with no IP header, and it is common in ISP backbones.
- A NET names the router: area, 6-byte System ID, and NSEL 00.
- L1 is inside an area, L2 joins areas, and L1/L2 routers do both. The border sits on the link, and there is no Area 0.
- L1 routers leave the area through a default route from the ATT bit. Route leaking fixes bad exit choices.
- LSPs carry TLVs. On a LAN, the DIS and CSNPs keep routers in sync, and a pseudonode keeps the map small.
- Use wide metrics, set metrics on purpose, and use point-to-point mode on two-router links.
- ISPs like it for one process with IPv4 and IPv6, easy extension, stability at scale and low exposure to remote attack.
