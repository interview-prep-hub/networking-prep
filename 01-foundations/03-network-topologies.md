# Network Topologies

**Level:** Beginner | **Prerequisites:** [What Is a Network?](01-what-is-a-network.md), [Types of Networks](02-network-types.md)

## In One Line

A network topology is the shape of a network: how devices are connected, and which way data travels between them.

## Why It Matters

The shape of a network decides three things: what it costs, how fast it is, and what breaks when something fails.

A question like "what happens if this cable is cut?" is really a topology question. In one shape, one person loses access. In another shape, the whole building goes down.

Interviewers ask about topologies for two reasons:

- It is a quick test of basics: can you name each shape and its weak point?
- It leads to design thinking: why do offices use stars, data centres use spine-leaf, and telecom networks use rings?

## Simple Explanation

Think of a group of people who need to pass messages to each other.

| Topology | Real-life picture | Where the picture stops being true |
|---|---|---|
| Bus | Everyone stands in one corridor. One person speaks and all hear it. | In a real bus, the signal dies at the ends of the cable and must be stopped there. |
| Star | Everyone talks through one receptionist. | A switch needs no thinking time like a person. It forwards frames in microseconds. |
| Ring | People sit around a table and pass a note to the left. | In a real ring, only the token holder may send. It is a strict rule, not a habit. |
| Mesh | Direct roads join every pair of cities. | Roads do not need a routing protocol. Network devices must learn which path to use. |
| Tree | A company chart: boss, managers, staff. | Data does not need to "ask the boss". It goes up only as far as needed, then down. |
| Hybrid | A country's roads: highways between cities, small streets inside them. | This one holds up well. Real networks also change shape by software, which roads cannot do. |

Words used in the rest of this page:

- **Node:** any device on the network, such as a PC, phone, printer, switch or router.
- **Link:** a connection between two nodes. It can be a cable, a fibre or a radio channel.
- **Single point of failure (SPOF):** one part that, if it breaks, stops the service.
- **Redundancy:** having a spare path or a spare device, so that one failure does not stop the service.
- **Physical topology:** where the cables really go.
- **Logical topology:** how data really moves.

The last two are not always the same. That difference is a favourite interview question.

## How It Works

### Bus

All nodes connect to one shared cable, called the **bus** or backbone. Old Ethernet used coaxial cable (a thick copper cable with one centre wire) for this.

```text
  PC-A      PC-B      PC-C      PC-D
   |         |         |         |
T==+=========+=========+=========+==T

T = terminator, a small resistor at each end of the cable
= = the one shared cable
```

How data moves:

1. PC-A sends a signal onto the cable.
2. The signal travels in both directions, so every node receives it.
3. Each node reads the destination address. Only PC-C keeps the data, if it is for PC-C.
4. The terminators absorb the signal at each end, so it does not bounce back.

Only one node can send cleanly at a time. If two send together, the signals mix and are lost. This is a **collision**. Ethernet on a bus used a rule called CSMA/CD: listen first, send when the cable is quiet, and if a collision happens, wait a random time and try again.

- **Pros:** very little cable, cheap, simple.
- **Cons:** all nodes share the bandwidth, collisions grow as nodes are added, and faults are hard to find.
- **Failure:** a cable cut stops the whole segment, not just one part. Each broken end has no terminator, so signals bounce back and corrupt the data. A missing terminator causes the same problem.
- **Today:** gone from office LANs. The old standards were 10BASE5 (segments up to 500 m) and 10BASE2 (up to 185 m). The idea lives on in CAN bus in cars, and in 10BASE-T1S, a single-pair Ethernet for cars and factories that can put many nodes on one cable.

### Star

Every node has its own cable to one central device. Today that device is a **switch** (a box that forwards data only to the right port) or a Wi-Fi access point.

```text
        PC-A     PC-B
           \     /
            \   /
  PC-F ---- SWITCH ---- PC-C
            /   \
           /     \
        PC-E     PC-D
```

How data moves:

1. PC-A sends a frame (a unit of data at this level) to the switch.
2. The switch looks at the destination address.
3. The switch sends the frame out of one port only, to PC-C.

Details are in [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md).

- **Pros:** easy to add a node, easy to find a fault, each port has its own bandwidth with a switch, and one bad cable hurts one node only.
- **Cons:** uses more cable than a bus, and the centre is a single point of failure. Copper twisted pair Ethernet is limited to 100 m per link, so nodes must be near the switch.
- **Failure:** a cable or PC fails: only that node is lost. The switch fails: every node on it is cut off.
- **Today:** the standard for homes and offices. Your home router is the centre of a small star.

### Ring

Each node connects to exactly two neighbours, and the links form a closed circle. Data travels around the circle, passing from node to node.

```text
     +-----> PC-A -----+
     |                 |
     |                 v
   PC-D              PC-B
     ^                 |
     |                 |
     +------ PC-C <----+
```

Classic Token Ring worked like this:

1. A small special frame, the **token**, circles the ring.
2. A node may send only while it holds the token. So there are no collisions.
3. The frame goes around. Each node passes it on.
4. The destination copies the data and marks the frame as received.
5. The frame returns to the sender, who removes it and releases the token.

- **Pros:** no collisions, and every node gets a fair, predictable turn, even under heavy load.
- **Cons:** each node adds delay, and adding or removing a node can disturb the ring. It also needs more complex hardware and rules than switched Ethernet.
- **Failure:** in a simple one-way ring, one cut or one dead node stops the whole loop. A **dual ring** fixes this: a second ring runs the opposite way. FDDI (Fiber Distributed Data Interface, 100 Mbps) worked this way. When a link breaks, traffic turns back at the break and uses the second ring. A second cut splits the ring into two islands.
- **Today:** Token Ring (IEEE 802.5, mostly 4 or 16 Mbps) and FDDI are history. Rings are alive in metro and telecom fibre networks, using SONET/SDH or Ethernet ring protection (ITU-T G.8032). Both aim to recover in about 50 ms. Ring also suits fibre: a loop around a city gives two paths with far less fibre than a mesh.

### Mesh

Nodes have many links to other nodes, so there is more than one path.

In a **full mesh**, every node has a direct link to every other node. In a **partial mesh**, only some nodes have direct links, usually the most important ones.

```text
  Full mesh: 4 nodes, 6 links      Partial mesh: 4 nodes, 4 links

  R1 ---------- R2                 R1 ---------- R2
  | \         / |                  |           / |
  |   \     /   |                  |         /   |
  |     \ /     |                  |       /     |
  |     / \     |                  |     /       |
  |   /     \   |                  |   /         |
  | /         \ |                  | /           |
  R3 ---------- R4                 R3            R4
```

The full mesh formula, for `n` nodes:

```text
links = n(n-1)/2        ports per node = n-1
```

Why divide by 2? Each of the `n` nodes has `n-1` links, but each link has two ends, so it was counted twice.

So 5 nodes need 10 links, 10 need 45, 20 need 190, 50 need 1225, and 100 need 4950. Links grow much faster than nodes. This is why large full meshes are rare.

- **Pros:** strong fault tolerance, no single point of failure, and traffic can share several paths.
- **Cons:** expensive in cables and ports, and hard to configure. Routers need a routing protocol, such as [OSPF](../05-routing/06-ospf.md), to choose paths.
- **Failure:** a link fails and traffic moves to another path, usually within seconds, and faster when tuned. In the partial mesh above, R4 has one link, so that link is still a weak point.
- **Today:** partial mesh is everywhere at the top of networks: ISP backbones, links between data centres, and the Internet itself. Full mesh is kept to small groups, such as 3 or 4 core routers. Wi-Fi mesh systems and IoT protocols such as Zigbee and Thread use radio mesh.

### Tree

A tree is a set of stars joined in levels, also called a **hierarchical star**. One device sits at the top (the **root**), and others branch down from it.

```text
                 [ROOT SWITCH]            level 1
                 /           \
           [SW-A]             [SW-B]      level 2
           /    \             /    \
        [SW-C] [SW-D]      PC-5    PC-6   level 3
        /  \      |
     PC-1  PC-2  PC-3                     level 4
```

A pure tree has exactly one path between any two nodes, and so no loops. A tree of `n` nodes has `n-1` links.

- **Pros:** grows easily, matches floors and departments, and keeps faults inside a branch.
- **Cons:** the upper links carry traffic from many users, so they can become a bottleneck. The root is a critical point.
- **Failure:** a device fails and everything below it is cut off. If SW-A fails, PC-1, PC-2 and PC-3 lose the network, but PC-5 and PC-6 can still talk through SW-B. If the root fails, the two branches cannot reach each other.
- **Today:** the normal shape of an office network, and of cable TV and PON (passive optical network) fibre access, where one fibre is split to many homes with no powered device in between. Offices add spare uplinks, which turns the tree into a hybrid. See [Network Design Models](../10-enterprise-and-datacenter/01-network-design-models.md).

### Hybrid

A hybrid mixes two or more shapes. Almost every real network is a hybrid.

```text
   [CORE-1]=============[CORE-2]     mesh at the top
      |   \             /   |
      |     \         /     |
      |       \     /       |
      |         \ /         |
      |         / \         |
      |       /     \       |
      |     /         \     |
      |   /             \   |
   [ACC-1]              [ACC-2]      star at the bottom
    / | \                / | \
  PC  PC  PC           PC  PC  PC
```

Each access switch is the centre of a star. Each one has two uplinks (links towards the core), so losing one uplink or one core switch does not cut it off.

- **Pros:** you spend money only where failure hurts most.
- **Cons:** harder to design and document.
- **Failure:** it depends on where the failure is. In the picture, a core failure is survived, but ACC-1 is still a single point of failure for its own PCs.
- **Today:** the shape of campuses, enterprise WANs and ISPs.

### Other Names You Should Know

- **Point-to-point:** one link between exactly two nodes. The simplest shape, such as two routers joined by a fibre.
- **Point-to-multipoint:** one node reaches many, as in PON fibre or a wireless bridge.
- **Daisy chain (line):** each node links to the next, in a row. It is a ring with one link missing. A break splits it in two.
- **Hub-and-spoke:** a star, but the word is used for WANs. Branch offices connect to one central site. Branch-to-branch traffic passes through the centre.

### Physical vs Logical Topology

**Physical topology** is the real layout of cables and devices. **Logical topology** is the path that signals or data really follow.

Think of a metro map. The map shows neat straight lines, but the real tunnels twist under the city. Where this stops being true: a metro map is only a simpler drawing. A logical topology is real behaviour. A hub truly acts as a bus, whatever the cables look like.

A **hub** (an old device that copies every signal to every port) is the classic example:

```text
  PHYSICAL (cables)            LOGICAL (signal)

    PC-A    PC-B               PC-A    PC-B    PC-C
       \    /                   |       |       |
        HUB                  ===+=======+=======+===
         |
        PC-C                 one shared wire inside the hub
```

| Technology | Physical | Logical |
|---|---|---|
| 10BASE2 coax Ethernet | Bus | Bus |
| Ethernet with a hub | Star | Bus |
| Token Ring with a MAU (a central ring box) | Star | Ring |
| Ethernet with a switch | Star | Star (each port is a private link) |
| Ring of switches running spanning tree | Ring | Tree (one link is blocked) |
| Wi-Fi with an access point | No cables | Star for the path, bus-like for the shared air |
| VPN tunnels over the Internet | Whatever the ISPs built | Any shape you choose, such as hub-and-spoke |

Fuller truth: "logical topology" has two common meanings. Older books mean how a signal moves on the medium (bus or ring). Modern engineers often mean the shape seen by a higher layer, such as a VPN or a routing protocol. Say which one you mean.

### Comparison

| Topology | Links for n nodes | Cost | One link fails | Weak point | Used today |
|---|---|---|---|---|---|
| Bus | 1 shared cable | Low | All lost | The cable | Cars, factories |
| Star | n cables | Medium | 1 node lost | Centre device | Homes, offices |
| Ring | n (2n if dual) | Medium | All lost (dual ring survives) | Each link and node | Metro, telecom |
| Full mesh | n(n-1)/2 | Very high | Traffic reroutes | Cost, complexity | Small core groups |
| Partial mesh | Between n and full | High | Usually reroutes | Nodes with one link | WAN, ISP, Internet |
| Tree | n-1 | Medium | Branch lost | Root, upper links | Office LANs, PON |
| Hybrid | Depends | Depends | Depends | Depends on the design | Nearly everywhere |

### What Is Used Today, and Why

| Place | Usual shape | Why |
|---|---|---|
| Home | Star around the router | Cheap and simple |
| Office or campus | Tree of stars with spare uplinks | Easy to grow, faults stay small |
| Data centre | Spine-leaf (a planned partial mesh) | Equal paths and high server-to-server bandwidth |
| Enterprise WAN | Hub-and-spoke or partial mesh | Links cost money, so only key sites get extras |
| ISP backbone and Internet | Partial mesh | Must survive failures at large scale |
| Metro fibre | Ring | Two paths with little fibre, fast recovery |
| Fibre to the home | Tree | One fibre serves many homes |

Star won in the LAN for three reasons. Twisted pair cable is cheap and easy to install. One bad cable hurts only one user. And switches ended collisions by giving each port a private, full-duplex link (both directions at once).

## Example

### Worked Example 1: Linking Six Sites

A company has 6 sites and wants VPN tunnels (private links carried over the Internet) between them. Each tunnel is one link to configure and monitor. See [VPNs](../09-security/07-vpn.md).

| Design | Calculation | Tunnels | Cut off by one site failing? |
|---|---|---|---|
| Full mesh | 6 x 5 / 2 | 15 | No |
| Hub-and-spoke | 6 - 1 | 5 | Yes, if the hub fails |
| Two hubs, 4 branches | (4 x 2) + 1 | 9 | No, the other hub remains |

The two-hub partial mesh gives most of the safety for 60 percent of the tunnels (9 out of 15). This is the usual answer.

### Worked Example 2: A Three-Floor Office

The office uses one subnet (one group of addresses), 10.10.0.0/24. One core switch feeds one access switch on each floor.

```text
     [Server 10.10.0.5]      [Router 10.10.0.1]
                   \            /
                    \          /
                     [ CORE-1 ]
                     /    |    \
                    /     |     \
                Gi1/0/1 Gi1/0/2 Gi1/0/3
                  /       |       \
                 /        |        \
           [ACC-1]     [ACC-2]     [ACC-3]
           floor 1     floor 2     floor 3
                       /     \
                      /       \
             10.10.0.21       10.10.0.22
              (Windows)        (Linux)
```

This is a tree. It has 8 nodes shown and 7 links, which matches `n-1`.

**Step 1: confirm the shape.** Do not trust an old diagram. Ask the switches. CDP is a Cisco protocol that tells a device about its direct neighbours. LLDP (IEEE 802.1AB) is the vendor-neutral version.

Cisco IOS (the capability legend at the top is cut here):

```text
CORE-1# show cdp neighbors
Device ID        Local Intrfce     Holdtme    Capability  Platform  Port ID
ACC-1            Gig 1/0/1         151              S I   C9200L-24 Gig 1/0/24
ACC-2            Gig 1/0/2         164              S I   C9200L-24 Gig 1/0/24
ACC-3            Gig 1/0/3         128              S I   C9200L-24 Gig 1/0/24
RTR-1            Gig 1/0/24        173              R S I ISR4331/K Gig 0/0/0

Total cdp entries displayed : 4
```

On Linux, `lldpctl` from the lldpd package gives the same kind of view, if the switch sends LLDP.

**Step 2: break something.** The uplink from ACC-2 to CORE-1 is cut.

Cisco IOS:

```text
CORE-1# show interfaces status

Port      Name       Status       Vlan       Duplex  Speed Type
Gi1/0/1   ACC-1      connected    1          a-full a-1000 10/100/1000BaseTX
Gi1/0/2   ACC-2      notconnect   1            auto   auto 10/100/1000BaseTX
Gi1/0/3   ACC-3      connected    1          a-full a-1000 10/100/1000BaseTX
```

**Step 3: see the effect from floor 2.** The server is unreachable. Note that these replies come from the PC itself. It is on the same subnet as the server, and its ARP request ("who has this address?") gets no answer. See [ARP](../03-ethernet-and-switching/05-arp.md).

Windows:

```powershell
C:\> ping 10.10.0.5

Pinging 10.10.0.5 with 32 bytes of data:
Reply from 10.10.0.21: Destination host unreachable.
Reply from 10.10.0.21: Destination host unreachable.
Reply from 10.10.0.21: Destination host unreachable.
Reply from 10.10.0.21: Destination host unreachable.

Ping statistics for 10.10.0.5:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
```

Linux:

```bash
$ ping -c 3 10.10.0.5
PING 10.10.0.5 (10.10.0.5) 56(84) bytes of data.
From 10.10.0.22 icmp_seq=1 Destination Host Unreachable
From 10.10.0.22 icmp_seq=2 Destination Host Unreachable
From 10.10.0.22 icmp_seq=3 Destination Host Unreachable

--- 10.10.0.5 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2041ms
```

Windows counts those error replies as "Received", so read the lines, not the summary. If the server's address is still in the ARP cache, you see "Request timed out" for a short time instead.

The two PCs on floor 2 can still ping each other, because ACC-2 is alive.

**Step 4: list the failure effects.**

| What fails | Who is affected |
|---|---|
| One PC cable | That PC |
| ACC-2 switch | All of floor 2 |
| ACC-2 uplink | Floor 2 loses the server and Internet, but works inside the floor |
| CORE-1 | All floors lose the server, the Internet and each other |

**Step 5: fix the weak points.** Add CORE-2 and give each access switch a second uplink. Uplinks go from 3 to 6, plus 1 link between the cores. The tree becomes the hybrid shown earlier.

These extra links form loops. Layer 2 networks need [spanning tree](../03-ethernet-and-switching/09-spanning-tree.md) or [link aggregation](../03-ethernet-and-switching/10-link-aggregation.md) to handle that safely.

## Going Deeper

**Spine-leaf is a partial mesh with strict rules.** Every leaf switch connects to every spine switch. Leaves do not link to leaves, and spines do not link to spines. With 4 spines and 20 leaves, you need 4 x 20 = 80 links. A full mesh of those 24 switches would need 276. Any two servers on different leaves are always the same distance apart: leaf, spine, leaf. See [Spine-Leaf Architecture](../10-enterprise-and-datacenter/02-spine-leaf.md).

**Overlays make logical shapes free of physical ones.** An **overlay** is a network built from tunnels on top of another network, called the **underlay**. SD-WAN, VPNs and [VXLAN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) all do this. You can build a logical full mesh over a partial-mesh underlay with no new cables. But the links still fail together: if one fibre carries many tunnels, one cut drops them all.

**The full mesh problem also applies to software.** [BGP](../05-routing/09-bgp.md) inside one network needs a full mesh of sessions between routers by default. With 100 routers, that is 4950 sessions. Route reflectors exist to avoid this.

**Physical diversity is the hidden trap.** Two fibres in the same trench are one failure on a map. A backhoe cuts both. Real redundancy needs separate paths, which is part of [High Availability](../10-enterprise-and-datacenter/07-high-availability.md).

## Common Mistakes

- Thinking a hub and a switch give the same topology. Both look like a star, but a hub behaves as a bus.
- Saying "star has no failure risk". The centre is a single point of failure. It only protects against failures of end cables.
- Thinking a bus cut leaves two working halves. Without terminators, reflections usually kill both halves.
- Believing the Internet is a full mesh. It is a partial mesh of networks joined where they choose to connect.
- Thinking "mesh Wi-Fi" means a full mesh. It often behaves like a tree towards the one Internet gateway.
- Adding physical redundancy at Layer 2 with no plan for loops. Without spanning tree or link aggregation, a loop can cause a broadcast storm: frames circle forever and the network goes down.
- Thinking that `traceroute` shows the full topology. It shows only routers on one path, and it does not show switches.
- Calling every drawing "the topology". Always ask: is this physical or logical?

## Interview Questions

**Q: What is a network topology?**
A: It is the arrangement of nodes and links in a network. Physical topology is how the cables are laid out, and logical topology is how the data really flows.

**Q: Explain physical vs logical topology with an example.**
A: Ethernet with a hub is physically a star, because each PC has a cable to the hub. Logically it is a bus, because the hub repeats every signal to every port, so all PCs share one medium and their signals can collide.

**Q: Which topology is most common in LANs today, and why?**
A: Star, or a tree of stars, using switches. It is cheap, easy to extend, and easy to troubleshoot. One bad cable affects one user, and each port gets its own full-duplex bandwidth.

**Q: What happens when the central device in a star fails?**
A: All nodes on that device lose connectivity. Failure of one cable affects only one node. So important devices get a second switch and spare uplinks.

**Q: How many links does a full mesh of 8 routers need?**
A: The formula is n(n-1)/2, so 8 x 7 / 2 = 28 links. Each router needs 7 ports.

**Q: Why did Ethernet move from bus to star?**
A: On a bus, one cable fault could stop the whole segment, and all nodes shared the bandwidth and collisions. Twisted pair star wiring limited faults to one node. Then switches removed collisions completely.

**Q: Why is a ring still used in telecom networks?**
A: A fibre ring gives every site two paths using far less fibre than a mesh. Ring protection such as SONET/SDH or G.8032 can switch to the other direction in about 50 ms.

**Q: A switch loop is physically a ring. Is it logically a ring?**
A: No. Spanning tree blocks one link, so the logical topology is a tree. If the active path fails, the blocked link opens and traffic flows again.

**Q: What is the topology of Wi-Fi?**
A: In normal infrastructure mode, it is a logical star, because clients talk through the access point. But the air is a shared medium, so stations take turns as on a bus. Wi-Fi uses CSMA/CA for this: listen first, and try to avoid collisions.

**Q: What topology does a data centre use, and why not a tree?**
A: It uses spine-leaf, a kind of partial mesh. A tree forces traffic through upper links and blocks spare paths. Spine-leaf gives many equal paths between servers, which suits heavy server-to-server traffic.

## Quick Recap

- Topology is the shape of a network, and it decides cost, speed and failure behaviour.
- Bus shares one cable. Star uses a centre device. Ring passes data around a loop. Mesh gives many paths. Tree is stars in levels. Hybrid mixes these.
- Physical is where the cables go. Logical is how data moves. A hub is a physical star and a logical bus.
- Full mesh needs n(n-1)/2 links, so it is used only in small groups.
- The failure question is always: what is the single point of failure, and how far does the damage spread?
- Today: stars and trees in LANs, spine-leaf in data centres, partial mesh in WANs and the Internet, rings in metro fibre.
- Almost every real network is a hybrid.
