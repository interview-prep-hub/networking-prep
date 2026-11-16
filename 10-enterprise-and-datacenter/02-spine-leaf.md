# Spine-Leaf Architecture

**Level:** Advanced | **Prerequisites:** [Network Design Models](01-network-design-models.md), [Spanning Tree Protocol (STP)](../03-ethernet-and-switching/09-spanning-tree.md), [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md), [BGP](../05-routing/09-bgp.md)

## In One Line

Spine-leaf is a two-layer data centre network where every leaf switch connects to every spine switch, so any server reaches any other server in the same small number of hops, and all links carry traffic at once.

## Why It Matters

A modern data centre has a traffic problem. One web request can trigger hundreds of calls between services, databases, caches and storage systems. Most of that traffic moves between servers inside the building, not out to the internet.

The older three-tier design was built for a different pattern. It expected traffic to go from a user, down to a server, and back. It wastes links, adds uneven delay, and is hard to grow when servers talk mostly to each other.

Spine-leaf fixes this. It gives:

- The same path length between any two racks.
- Use of all links at the same time, with no blocked links.
- Growth by adding more switches, not by buying a bigger one.
- Small damage when one switch fails.

Interviewers ask about it because it tests many skills at once: Layer 2 versus Layer 3, routing protocols, hashing, capacity maths, and failure thinking. It is standard in data centres and also in the physical networks that run public clouds.

## Simple Explanation

Think of air travel.

Each small city has one airport. That is a **leaf**. A few big hub airports serve the whole country. Those are the **spines**.

The rule is simple: every small city has a direct flight to every hub. No small city flies directly to another small city. No hub flies to another hub.

Now any trip between two small cities takes exactly two flights: city to hub, then hub to city. If the airline adds one more hub, every city gets one more route, and total capacity grows. If one hub closes because of snow, passengers use the other hubs. Nobody is stranded.

In the network:

- A **leaf** is a switch that connects to servers, usually at the top of a rack. People also call it a ToR switch (top of rack).
- A **spine** is a switch that connects only to leaves.
- A **fabric** is the whole set of switches and links, working as one large network.

**Where the analogy stops being true:** an airline looks at how full each flight is before it sells a seat. Basic spine-leaf does not look at load. It picks a path with a math function, so two very large flows can land on the same link even when other links are empty. We will see why under ECMP.

## How It Works

### Why three-tier fails in data centres

The classic design has three layers. The [Network Design Models](01-network-design-models.md) chapter covers it in full. In short:

- **Access layer:** switches that servers plug into.
- **Aggregation layer (also called distribution):** a pair of larger switches that connect many access switches. It often holds the default gateways.
- **Core layer:** the fastest switches, which connect the aggregation blocks and the outside world.

```text
                +--------+      +--------+
  Core          | Core 1 |======| Core 2 |
                +--------+      +--------+
                  |     \        /     |
                  |      \      /      |
                +--------+      +--------+
  Aggregation   | Agg 1  |======| Agg 2  |  gateways, STP root
                +--------+      +--------+
                  |    :          |    :
                  |    :          |    :    : = blocked by STP
                +--------+      +--------+
  Access        | Acc 1  |      | Acc 2  |
                +--------+      +--------+
                  |  |  |         |  |  |
                 servers         servers
```

This design served campus and early data centre networks well. It has five problems at modern scale.

**Problem 1: it was built for north-south traffic.** North-south traffic moves in and out of the data centre, such as a user to a web server. East-west traffic moves sideways, from server to server. Microservices, storage replication, big data jobs and virtual machine moves are all east-west. In three-tier, this traffic must climb up the tree and come back down.

**Problem 2: uneven path length.** The number of switches between two servers depends on where they sit.

| Where the two servers are | Switches crossed in three-tier | Switches crossed in spine-leaf |
|---|---|---|
| Same access switch or leaf | 1 | 1 |
| Same aggregation block, different access switch | 3 | 3 |
| Different aggregation blocks | 5 | 3 |

Uneven paths mean uneven delay. An application may run well in one rack layout and badly in another.

**Problem 3: Spanning Tree wastes links.** Layer 2 frames have no TTL (a counter that kills looping packets), so a loop can run forever. [Spanning Tree Protocol (STP)](../03-ethernet-and-switching/09-spanning-tree.md) prevents loops by blocking redundant links. A switch with two 40G uplinks then uses only one. You paid for 80G and get 40G.

Multi-chassis link aggregation helps. It makes two aggregation switches look like one to the access switch, so both links stay active. Cisco calls it vPC, Arista calls it MLAG, and Juniper calls it MC-LAG. But it stops at two boxes. You cannot use this to spread load across eight aggregation switches.

**Problem 4: oversubscription stacks up.** Oversubscription means the capacity coming into a layer is bigger than the capacity going out. Each layer adds its own ratio, and they multiply.

Example of a worst case:

```text
Access switch:
  Down: 48 x 10G = 480G
  Up:   2 x 40G, STP blocks one, so 40G is usable
  Ratio: 480 / 40 = 12:1

Aggregation block:
  Down: 20 access switches x 40G = 800G
  Up:   2 x 100G = 200G
  Ratio: 800 / 200 = 4:1

End to end: 12 x 4 = 48:1
Each server, if all are busy across blocks: 10G / 48 = about 208 Mbps
```

A 10G server gets about 2% of its speed in this worst case.

**Problem 5: scale up, and large failure domains.** To grow three-tier, you replace the aggregation or core pair with bigger, more costly chassis. This is **scale up**. Spine-leaf uses **scale out**: add more ordinary boxes.

A pair also has poor failure maths. Lose one of two switches and half the capacity is gone. Lose one of eight spines and only 12.5% is gone.

A large Layer 2 domain also spreads trouble. One broadcast storm or one bad loop can hurt every server in the block. Gateways and ARP tables also concentrate at the aggregation pair, which limits growth.

### The rules of a spine-leaf fabric

The structure is strict.

1. Every leaf connects to every spine.
2. Leaves do not connect to each other for normal traffic.
3. Spines do not connect to each other.
4. Servers and outside devices connect only to leaves. Nothing but leaves connects to spines.

```text
            Leaf 1      Leaf 2      Leaf 3      Leaf 4
              |           |           |           |
  Spine 1 ----*-----------*-----------*-----------*
              |           |           |           |
  Spine 2 ----*-----------*-----------*-----------*
              |           |           |           |
  Spine 3 ----*-----------*-----------*-----------*
              |           |           |           |
  Spine 4 ----*-----------*-----------*-----------*

  * = one physical link between that leaf and that spine
  4 leaves x 4 spines = 16 links
```

From these rules we get a key result. Any leaf reaches any other leaf through **any** spine. With 4 spines, there are 4 equal paths, each crossing exactly one spine.

One exception exists to rule 2. If two leaves form an MLAG or vPC pair for dual-attached servers, they often have a peer link between them. It is for pair control and failure cases, not for normal fabric paths. Newer designs avoid it using EVPN multihoming, covered in [VXLAN and EVPN](11-vxlan-and-evpn.md).

### The Clos fabric

Spine-leaf is not a new idea. It is a form of Clos network, named after Charles Clos of Bell Labs, who published the design in 1953. He wanted to build large telephone exchanges from small switches.

A Clos network has three stages of switches: an ingress stage (where calls enter), a middle stage, and an egress stage (where calls leave). Each ingress switch connects to every middle switch. Each middle switch connects to every egress switch.

```text
   Unfolded 3-stage Clos              Folded (spine-leaf)

  Ingress   Middle   Egress               Spine (middle)
  [Leaf 1]--[Sp 1]--[Leaf 1]             [Sp 1]    [Sp 2]
          \/      \/                       | \      / |
          /\      /\                       |   \  /   |
  [Leaf 2]--[Sp 2]--[Leaf 2]               |   /  \   |
                                           | /      \ |
  Traffic flows left to right            [Leaf 1]  [Leaf 2]
                                          Traffic goes up,
                                          then back down
```

Fold the drawing in half along the middle stage. The ingress and egress stages now sit on top of each other. A real leaf acts as both: ingress for traffic it sends up, egress for traffic it receives. So a two-layer spine-leaf network is a **folded three-stage Clos**.

That is why engineers say "three-stage" for a network with two layers of boxes. The count follows the path of a packet: leaf, spine, leaf.

**The classic maths.** Let `n` be the inputs on each ingress switch and `m` the number of middle switches.

| Condition | Name | Meaning |
|---|---|---|
| m >= 2n - 1 | Strictly non-blocking | A new call always finds a free path, with no change to other calls |
| m >= n | Rearrangeably non-blocking | A new call can always fit, but existing calls may have to move |

For `n = 4`, strict needs 7 middle switches and rearrangeable needs 4.

**Fuller truth:** those proofs are for circuits, where a connection owns a path. A packet fabric does not work that way. Flows share links, and a hash picks the path, so a 1:1 fabric is not truly non-blocking. People use "non-blocking" loosely to mean "enough total capacity exists". Two flows can still collide on one link, as the next section shows.

A related term is **fat tree**, a Clos built so that links get "fatter" (more total capacity) toward the top. A well-known 2008 SIGCOMM paper by Al-Fares and others showed a k-ary fat tree can link k cubed divided by 4 hosts using only k-port commodity switches. For `k = 48`, that is 27,648 hosts.

### ECMP: using all the paths

**ECMP (equal-cost multipath)** means a router keeps several best next hops to one destination and spreads traffic across them. Without it, a routing protocol picks just one path, and the other spines sit idle.

A **flow** is one conversation between two programs. It is normally identified by the **5-tuple**: source IP, destination IP, protocol, source port and destination port.

The switch does not send each packet on a random path. If it did, packets of one TCP connection could take paths with different delay and arrive out of order. TCP may treat that as loss and slow down. So the switch keeps each flow on one path:

1. The packet arrives at the leaf.
2. The leaf reads the fields of the 5-tuple.
3. It runs them through a **hash**, a function that turns input into a number. The same input always gives the same number.
4. It divides by the number of next hops and takes the remainder.
5. The remainder selects the next hop.

```text
  Flow: 10.20.1.11:49152 -> 10.20.2.21:443, TCP

  hash(5-tuple) = 2,718,281    (made-up value)
  2,718,281 mod 4 = 1

  Index:   0         1         2         3
  Path:  Spine 1  [Spine 2]  Spine 3   Spine 4

  New connection, source port 49153:
  hash(5-tuple) = 2,718,282    (made-up value)
  2,718,282 mod 4 = 2  -> Spine 3
```

Real hardware uses functions such as CRC-based hashes and lookup tables, not a plain `mod`. The idea is the same.

**Analogy.** A supermarket has four checkout lanes. The manager says: "Use the lane that matches the last digit of your loyalty card." One family always stays in one lane, in order. That is per-flow hashing. **Where it stops being true:** real shoppers switch to a shorter lane. A flow does not. It stays on its lane even if that lane is full.

**The big weakness: elephants and mice.** Most flows are small, short "mice". A few are huge, long "elephants", such as a backup or a data shuffle. ECMP counts flows, not bytes. It can place two elephants on one link.

The chance is not small. Suppose four 100G elephants share four 100G uplinks. The chance that all four take different paths is:

```text
(4 x 3 x 2 x 1) / (4 x 4 x 4 x 4) = 24 / 256 = about 9.4%
```

So about 90.6% of the time, at least two elephants share a link, and they each get less than their fair speed. Meanwhile another link sits partly empty.

The common defence is simple: make uplinks faster than server links. With 25G servers on 100G uplinks, one flow can never fill an uplink, and many flows average out well.

**What happens when a path dies?** Hash results change. With plain modulo, moving from 4 paths to 3 changes the answer for most flows, even those not on the failed path.

```text
Hash h:     0  1  2  3  4  5  6  7  8  9 10 11
h mod 4:    0  1  2  3  0  1  2  3  0  1  2  3
h mod 3:    0  1  2  0  1  2  0  1  2  0  1  2
Same?       Y  Y  Y  n  n  n  n  n  n  n  n  n

Only 3 of 12 stay in place. 75% of flows move.
Only 25% needed to move, because only 1 of 4 paths failed.
```

**Resilient hashing** fixes this. The switch keeps a table of buckets. When a next hop fails, only that next hop's buckets get a new owner. Other flows stay put. For plain routing, moving flows causes brief reordering. It matters much more when ECMP is spreading traffic to stateful servers, such as anycast load balancers, because a moved flow may hit a server that does not know the connection. See [Load Balancing](04-load-balancing.md).

Group size also has limits. A switch chip supports only so many next hops per ECMP group and so many groups. The numbers vary by chip, so check the vendor data sheet.

### Oversubscription maths

**Oversubscription** is the ratio of the bandwidth that could enter a device to the bandwidth that can leave it.

```text
Oversubscription ratio = total downlink capacity / total uplink capacity
```

**Analogy.** A gym sells 1,000 memberships but has 100 machines. That works because not everyone comes at once. **Where it stops being true:** gym members wait in a line. A switch has small buffers. When a queue is full, packets are dropped, and TCP slows down.

Worked examples for one leaf:

| Server ports | Downlink | Uplinks | Uplink total | Ratio | Per server if all busy |
|---|---|---|---|---|---|
| 48 x 25G | 1,200G | 8 x 100G | 800G | 1.5:1 | 16.7G |
| 48 x 25G | 1,200G | 6 x 100G | 600G | 2:1 | 12.5G |
| 48 x 25G | 1,200G | 4 x 100G | 400G | 3:1 | 8.3G |
| 32 x 100G | 3,200G | 8 x 400G | 3,200G | 1:1 | 100G |

The last column is simply server speed divided by the ratio.

Three more facts complete the picture.

**Only traffic leaving the leaf uses uplinks.** Suppose 30% of traffic stays in the rack. Then at full load, the uplinks must carry 70% of 1,200G, which is 840G. On 800G uplinks, that is 1.05:1, much better than the 1.5:1 headline number.

**Ratios multiply across tiers.** In a three-stage fabric the spines have no uplinks, so only the leaf ratio counts. In a five-stage fabric, a 1.5:1 leaf and a 2:1 spine give 3:1 between pods.

**Plan for failure.** With `N` spines, losing one multiplies the ratio by `N / (N - 1)`.

```text
Leaf: 1,200G down, 4 x 100G up, 4 spines
Normal:          1,200 / 400 = 3:1
One spine down:  1,200 / 300 = 4:1
Two spines down: 1,200 / 200 = 6:1
```

**Bisection bandwidth** is another common measure. Cut the fabric into two equal halves, and add up the capacity across the cut. With 64 leaves, each with 800G of uplinks, the cut carries 32 x 800G = 25.6 Tbps in one direction. The server demand in that half is 32 x 1,200G = 38.4 Tbps. That is the same 1.5:1. A fabric with **full bisection** can carry all of it (1:1). Some vendors count both directions, so check before you compare numbers.

**What is a good ratio?** There is no standard. General compute often runs at 2:1 to 4:1. Storage and big data are often nearer 1:1 to 2:1. GPU clusters for machine learning usually aim for 1:1. Measure your real traffic before you decide.

Also remember that a ratio describes long-term capacity. A **microburst** is a very short spike, often under a millisecond, that overflows a buffer even when one-minute graphs look calm. See [Going Deeper](#going-deeper).

### Routing to the leaf: Layer 3 fabric

The next question is: do the links between leaf and spine act as Layer 2 or Layer 3?

Modern fabrics use **Layer 3 links**. Every leaf-to-spine link is a small routed subnet, and a routing protocol runs between them.

- No STP is needed in the fabric, because there is no Layer 2 between switches.
- Routed packets have a TTL, so a loop dies out.
- ECMP works natively.
- A broadcast stays within one rack or VLAN.

The subnet on each link is often a /31, which has two addresses and no wasted network or broadcast address. RFC 3021 defines this. See [Subnetting Step by Step](../04-ip-addressing/05-subnetting-step-by-step.md).

The most common design in large fabrics is eBGP on every link, described in RFC 7938. **BGP** is the routing protocol of the internet, and eBGP means BGP between different autonomous systems. An **autonomous system (AS)** is a group of routers under one routing policy, named by an ASN (AS number). In the usual pattern:

- All spines share one ASN.
- Each leaf gets its own ASN.

OSPF and IS-IS also work well, and many enterprises use them. [OSPF](../05-routing/06-ospf.md) and [IS-IS](../05-routing/08-is-is.md) have their own chapters. The comparison is in Going Deeper.

**But applications may need Layer 2.** Some older clusters and virtual machine moves expect the same subnet across racks. The answer is an **overlay**: a virtual network built on top of the physical one. The physical routed network is the **underlay**. VXLAN carries Layer 2 frames inside UDP packets across the routed fabric, and EVPN uses BGP to share who is where. That is the job of the [VXLAN and EVPN](11-vxlan-and-evpn.md) chapter. Here, you just need to know the two ideas work together: spine-leaf is the physical shape, VXLAN and EVPN provide flexible segments on top.

### Routing to the host

"Routing to the host" takes one more step. Each server joins the routing protocol. The Layer 3 boundary moves from the leaf down into the server itself.

Without it, a dual-attached server needs a bond (two links acting as one) to two leaves. Those leaves then need MLAG, which adds shared state between them and some rare, painful failure modes.

With routing to the host:

1. The server has a loopback address, a virtual interface that never goes down. Example: 10.20.1.11/32.
2. The server runs a small routing daemon, such as FRR or BIRD.
3. It has one link to each of two leaves. Each link is an independent routed link.
4. It announces its /32 to both leaves with BGP.
5. It learns a default route from both leaves.
6. Now the server does ECMP to the two leaves. If one link fails, BGP removes that path.

```text
             Leaf 1 (AS 65001)      Leaf 2 (AS 65002)
             10.10.1.1/24           10.10.2.1/24
                    \                  /
              eth0   \                / eth1
         10.10.1.11   \              /  10.10.2.11
                     +----------------+
                     |  Server: FRR   |  AS 65100
                     |  lo 10.20.1.11 |  announces this /32
                     +----------------+
```

**Analogy.** A house has one main street, and if it floods you are stuck. Now give the house two driveways to two streets, and let delivery drivers use either one. No gate guard needs to coordinate them. **Where it stops being true:** a road closure can take a while to learn about. Routing needs a fast detector, so we pair BGP with BFD.

Benefits:

- No MLAG and no Layer 2 between leaves.
- A service can have one stable address, even if a link or leaf fails.
- A server can announce extra addresses, such as container subnets or anycast service addresses.

Costs:

- Every server needs a routing daemon, plus operating and security discipline.
- Leaves must filter what servers announce. One wrong server must not be able to announce a default route.
- Storage systems and appliances may not support it.
- A /32 per server means larger routing tables. Choose hardware to fit.

**Common real uses.** Kubernetes network plugins such as Calico can use BGP to announce pod routes to the fabric. Cilium also has a BGP feature. MetalLB uses BGP to announce service addresses in bare-metal clusters. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

**Windows note:** Windows Server has a BGP router feature in its Routing and Remote Access role, but routing to the host is far more common on Linux.

### Scaling: how far it can grow

Three numbers define a three-stage fabric with one link from each leaf to each spine.

| Name | Meaning |
|---|---|
| S | Ports on each spine |
| U | Uplink ports on each leaf |
| D | Server ports on each leaf |

```text
Maximum leaves      = S
Maximum spines      = U
Maximum server ports = S x D
Total fabric links   = S x U     (when fully built)
```

So spine port count limits how many leaves you can have, and leaf uplink count limits how many spines you can have.

| Spine ports (S) | Spines (U) | Leaves | Server ports (D = 48) | Fabric links |
|---|---|---|---|---|
| 32 | 4 | 32 | 1,536 | 128 |
| 64 | 8 | 64 | 3,072 | 512 |
| 128 (32 x 400G split, see below) | 8 | 128 | 6,144 | 1,024 |

**Breakout** means splitting one fast port into several slower ones with a special cable. A 400G port can split into four 100G ports, so 32 x 400G can act as 128 x 100G.

Ways to grow, in order of ease:

1. **Add leaves** until spine ports run out.
2. **Add spines** until leaf uplinks run out. This lowers oversubscription.
3. **Raise link speed**, such as 100G to 400G.
4. **Move to five stages** when one set of spines can no longer hold all leaves.

Two points of discipline matter. Keep all spines the same size, and all leaf-to-spine links the same speed, because ECMP assumes equal paths. Also, keep leaf-to-spine link counts even across spines.

### Five-stage fabric

A **pod** is a small spine-leaf block. A **super-spine** layer connects many pods. A packet between pods crosses five stages: leaf, spine, super-spine, spine, leaf.

```text
   +-------+ +-------+ +-------+ +-------+
   |  SS1  | |  SS2  | |  SS3  | |  SS4  |   Super-spine
   +-------+ +-------+ +-------+ +-------+
       \ \      | |       | |      / /
    (each spine links to the super-spines in its plane)
       / /      | |       | |      \ \
   +-----+ +-----+         +-----+ +-----+
   | Sp1 | | Sp2 |         | Sp3 | | Sp4 |   Spine
   +-----+ +-----+         +-----+ +-----+
     | \   / |               | \   / |
     |  \ /  |               |  \ /  |
     |  / \  |               |  / \  |
     | /   \ |               | /   \ |
   +-----+ +-----+         +-----+ +-----+
   | L1  | | L2  |         | L3  | | L4  |   Leaf
   +-----+ +-----+         +-----+ +-----+
        Pod 1                   Pod 2
```

Worked sizing with 64-port 100G switches at the spine and super-spine layers, and leaves with 48 x 25G plus 8 x 100G:

```text
Each spine: 32 ports down, 32 ports up (1:1 at the spine)
Each pod:   32 leaves, 8 spines
Server ports per pod: 32 x 48 = 1,536

8 spine positions means 8 super-spine planes.
Each plane has 32 super-spines (one per spine uplink).
Total super-spines: 8 x 32 = 256

Each super-spine has 64 ports, so up to 64 pods.

Total server ports: 64 x 1,536 = 98,304
Total switches:
  Leaves       64 x 32 = 2,048
  Spines       64 x 8  =   512
  Super-spines         =   256
  All                  = 2,816
```

Large operators publish designs of this kind. Facebook (now Meta) described a pod-based fabric with spine planes in 2014, and Google has published papers on its Clos-based Jupiter network.

### Failure behaviour

Here is what happens if a spine dies in a healthy design. Assume BFD at 300 ms and a multiplier of 3.

**BFD (Bidirectional Forwarding Detection)**, defined in RFC 5880, is a light hello protocol that detects a dead path far faster than normal BGP timers.

1. The leaf misses three BFD packets, in about 900 ms. If the light on the link goes off, the interface goes down sooner.
2. BFD tells BGP. The session drops and BGP removes those routes.
3. The switch removes the failed next hop from its ECMP groups. Hardware shares one ECMP group across many routes, so one small update can repair thousands of prefixes.
4. New packets hash across the remaining spines.
5. Capacity falls by 1 of N. The fabric keeps running.

## Example

### Design a fabric for 800 servers

**Requirements**

- 800 servers, 40 per rack, so 20 racks.
- Each server has two 25G links, one to each of two leaves in its rack.
- No more than 3:1 in normal running, and under 4:1 with one spine failed.
- Room to grow to 1,200 servers without more spine switches.
- Routing to the host, with no MLAG.

**Hardware choice**

- Leaf: 48 x 25G plus 8 x 100G.
- Spine: 64 x 100G.

**Step 1: count the leaves**

```text
Server leaves: 20 racks x 2 = 40
Border leaves: 2 (connect to firewalls, routers, the WAN)
Total leaves:  42
```

Border leaves are ordinary leaves that link the fabric to the outside. Never connect outside gear directly to spines. It would break the uniform pattern.

**Step 2: uplinks and oversubscription**

```text
Leaf downlink in use: 40 x 25G = 1,000G
With 4 x 100G uplinks: 400G
Ratio:                 1,000 / 400 = 2.5:1   (meets 3:1)
One spine failed:      1,000 / 300 = 3.33:1  (meets under 4:1)
Worst-case per server: 50G / 2.5 = 20G
```

**Step 3: spines and growth**

```text
One uplink per spine, 4 uplinks: 4 spines
Ports used on each spine:        42 of 64
Spare ports on each spine:       22

Growth to 1,200 servers: 30 racks x 2 = 60 leaves + 2 border = 62
62 is at or below 64, so the spines have room.

Extra capacity later: add 4 more spines, and use all 8 uplinks.
Ratio then: 1,000 / 800 = 1.25:1
```

**Step 4: count links and optics**

```text
Fabric links: 42 leaves x 4 spines = 168
Optics:       168 x 2 ends = 336
```

**Step 5: addressing and AS numbers**

| Item | Pattern | Example |
|---|---|---|
| Spine loopback | 10.255.0.s/32 | spine1 = 10.255.0.1 |
| Leaf loopback | 10.255.1.n/32 | leaf3 = 10.255.1.3 |
| Link leaf n to spine s | 10.254.s.(2n-2) on spine, 10.254.s.(2n-1) on leaf, /31 | leaf3 to spine1: 10.254.1.4 and 10.254.1.5 |
| Leaf n server subnet | 10.10.n.0/24, gateway 10.10.n.1 | leaf1 = 10.10.1.0/24 |
| Server loopback, rack r, server x | 10.20.r.x/32 | 10.20.2.21 |
| Spine ASN | 65000 | all spines |
| Leaf ASN | 65000 + n | leaf1 = 65001, border leaves = 65041 and 65042 |
| Server ASN | 65100 | shared, safe here because servers only learn a default route |

```text
                 AS 65000 (all spines)
   +--------+  +--------+  +--------+  +--------+
   | spine1 |  | spine2 |  | spine3 |  | spine4 |
   +--------+  +--------+  +--------+  +--------+
        \ \        | |        | |        / /
         (every leaf has one 100G link to each spine)
        / /        | |        | |        \ \
   +--------+  +--------+  +--------+  +--------+
   | leaf1  |  | leaf2  |  | leaf3  |  | leaf4  |
   | 65001  |  | 65002  |  | 65003  |  | 65004  |
   +--------+  +--------+  +--------+  +--------+
        \          /            \          /
         \        /              \        /
     Rack 1, srv 10.20.1.11   Rack 2, srv 10.20.2.21
     eth0 10.10.1.11          eth0 10.10.3.21
     eth1 10.10.2.11          eth1 10.10.4.21
```

**Step 6: routing table size**

```text
40 server subnets + 46 device loopbacks + 800 server /32s + 1 default
= 887 routes
```

Any data centre switch holds this easily. Containers or many announced service addresses would add more routes.

### Leaf configuration

These examples use Cisco NX-OS, the data centre operating system on Cisco Nexus switches. It looks like IOS, but some commands differ. Only the neighbours for spine1 are shown. Spines 2-4 follow the same pattern.

```text
! Cisco NX-OS, leaf1
feature bgp
feature bfd
feature interface-vlan

ip prefix-list SERVER-LOOPBACKS seq 10 permit 10.20.0.0/16 eq 32
ip prefix-list DEFAULT-ONLY seq 10 permit 0.0.0.0/0

route-map FROM-SERVERS permit 10
  match ip address prefix-list SERVER-LOOPBACKS
route-map TO-SERVERS permit 10
  match ip address prefix-list DEFAULT-ONLY

vlan 10

interface Vlan10
  description Rack 1 servers, leaf1 side
  ip address 10.10.1.1/24
  no shutdown

interface Ethernet1/1
  description srv11 eth0
  switchport
  switchport access vlan 10
  no shutdown

interface loopback0
  ip address 10.255.1.1/32

interface Ethernet1/49
  description to spine1 Eth1/1
  no switchport
  mtu 9216
  no ip redirects
  ip address 10.254.1.1/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown

router bgp 65001
  router-id 10.255.1.1
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    network 10.255.1.1/32
    network 10.10.1.0/24
    maximum-paths 8
  neighbor 10.254.1.0
    description spine1
    remote-as 65000
    bfd
    address-family ipv4 unicast
  neighbor 10.10.1.0/24
    description rack 1 servers on eth0
    remote-as 65100
    address-family ipv4 unicast
      route-map FROM-SERVERS in
      route-map TO-SERVERS out
```

Why these lines matter:

- `maximum-paths 8` allows ECMP. The default on NX-OS is a single path. Eight leaves room to grow to 8 spines.
- `bestpath as-path multipath-relax` is needed here. A server is attached to two leaves, so the same /32 arrives with different AS paths, such as `65000 65003 65100` and `65000 65004 65100`. BGP normally requires AS paths to match exactly for multipath. This setting relaxes that to "same length".
- `route-map FROM-SERVERS in` stops a server from announcing anything except a /32 in 10.20.0.0/16.
- `route-map TO-SERVERS out` sends only the default route. Border leaves are the origin of that default. If leaf1 loses all spines, the default disappears, and it stops being sent to servers. So servers will use leaf2 alone. A forced default from each leaf would hide this failure.

### Spine configuration

```text
! Cisco NX-OS, spine1
feature bgp
feature bfd

interface loopback0
  ip address 10.255.0.1/32

interface Ethernet1/1
  description to leaf1 Eth1/49
  no switchport
  mtu 9216
  no ip redirects
  ip address 10.254.1.0/31
  bfd interval 300 min_rx 300 multiplier 3
  no shutdown

router bgp 65000
  router-id 10.255.0.1
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    network 10.255.0.1/32
    maximum-paths 2
  neighbor 10.254.1.1
    description leaf1
    remote-as 65001
    bfd
    address-family ipv4 unicast
```

The spine holds no servers, VLANs or gateways. It only moves packets between leaves. `maximum-paths 2` lets it balance a server /32 across that server's two leaves.

### Server configuration

```text
! Linux, FRR, /etc/frr/frr.conf on the server in rack 1
hostname srv11
interface lo
 ip address 10.20.1.11/32
router bgp 65100
 bgp router-id 10.20.1.11
 no bgp ebgp-requires-policy
 bgp bestpath as-path multipath-relax
 neighbor 10.10.1.1 remote-as 65001
 neighbor 10.10.2.1 remote-as 65002
 address-family ipv4 unicast
  network 10.20.1.11/32
  maximum-paths 2
 exit-address-family
route-map SET-SRC permit 10
 set src 10.20.1.11
ip protocol bgp route-map SET-SRC
```

The last three lines are easy to forget. Without `set src`, the kernel may pick the eth0 address as the source for new connections. Then a failure of eth0 kills those connections, even though the server still has a path. With it, new connections use the stable loopback address.

Check the routes on the server:

```bash
# Linux
$ ip route show
default nhid 31 proto bgp src 10.20.1.11 metric 20
	nexthop via 10.10.1.1 dev eth0 weight 1
	nexthop via 10.10.2.1 dev eth1 weight 1
10.10.1.0/24 dev eth0 proto kernel scope link src 10.10.1.11
10.10.2.0/24 dev eth1 proto kernel scope link src 10.10.2.11
```

By default, Linux hashes IPv4 multipath routes using addresses only. To include ports, so that different connections to the same server can use different links:

```bash
# Linux: 0 = Layer 3 hash (default), 1 = Layer 4 (5-tuple) hash
sudo sysctl -w net.ipv4.fib_multipath_hash_policy=1
```

### Follow one packet

A packet goes from 10.20.1.11 in rack 1 to 10.20.2.21 in rack 2.

1. srv11 looks up the destination and matches the default route. Its hash picks eth0, so the packet goes to leaf1 (10.10.1.1). The source address is 10.20.1.11. TTL starts at 64.
2. leaf1 has four equal next hops. Its hash picks spine3 (10.254.3.0). TTL becomes 63.
3. spine3 has two next hops for this /32: leaf3 (10.254.3.5) and leaf4 (10.254.3.7). Its hash picks leaf3. TTL becomes 62.
4. leaf3 knows the server through BGP, next hop 10.10.3.21. It uses ARP to find the server's MAC address, and sends the frame out of the server port. TTL becomes 61.
5. srv21 receives the packet on eth0 with TTL 61. Three routers, three hops.

The reply is a separate decision. Each direction is hashed by itself, so the return path may use a different spine. That is normal, and it is called **asymmetric routing**.

### Check the fabric

```text
! Cisco NX-OS, on leaf1
leaf1# show ip route 10.20.2.21/32
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.20.2.21/32, ubest/mbest: 4/0
    *via 10.254.1.0, [20/0], 02:14:07, bgp-65001, external, tag 65000
    *via 10.254.2.0, [20/0], 02:14:07, bgp-65001, external, tag 65000
    *via 10.254.3.0, [20/0], 02:14:07, bgp-65001, external, tag 65000
    *via 10.254.4.0, [20/0], 02:14:07, bgp-65001, external, tag 65000
```

Four `*via` lines mean four active paths. `ubest 4` confirms it. If you see one, check `maximum-paths` and `multipath-relax` first.

```bash
# Linux, on srv11. Each probe has a new UDP port, so it can hash differently.
$ traceroute -n 10.20.2.21
traceroute to 10.20.2.21 (10.20.2.21), 30 hops max, 60 byte packets
 1  10.10.1.1  0.25 ms  0.21 ms  0.20 ms
 2  10.254.3.0  0.41 ms  10.254.1.0  0.39 ms  10.254.4.0  0.44 ms
 3  10.254.2.5  0.62 ms  10.254.3.7  0.60 ms  10.254.4.5  0.58 ms
 4  10.20.2.21  0.71 ms  0.69 ms  0.70 ms
```

Hop 2 shows three different spines. Hop 3 shows leaf3 (`.5`) and leaf4 (`.7`). That is ECMP working. Hop 1 stays on leaf1 here because the server is using the default Layer 3 hash.

Windows `tracert` sends ICMP echo requests, which have no port numbers. So all probes usually follow one path. This example runs from a Windows host at 10.10.1.50 in rack 1, with gateway 10.10.1.1.

```powershell
# Windows
C:\> tracert -d 10.20.2.21

Tracing route to 10.20.2.21 over a maximum of 30 hops

  1    <1 ms    <1 ms    <1 ms  10.10.1.1
  2    <1 ms    <1 ms    <1 ms  10.254.3.0
  3    <1 ms    <1 ms    <1 ms  10.254.3.5
  4    <1 ms    <1 ms    <1 ms  10.20.2.21

Trace complete.
```

### Test a link failure

Shut down the link from leaf1 to spine2, then look again.

```text
! Cisco NX-OS, on leaf1
leaf1# configure terminal
leaf1(config)# interface Ethernet1/50
leaf1(config-if)# shutdown
leaf1(config-if)# end

leaf1# show ip route 10.20.2.21/32 | include via|ubest
10.20.2.21/32, ubest/mbest: 3/0
    *via 10.254.1.0, [20/0], 02:16:31, bgp-65001, external, tag 65000
    *via 10.254.3.0, [20/0], 02:16:31, bgp-65001, external, tag 65000
    *via 10.254.4.0, [20/0], 02:16:31, bgp-65001, external, tag 65000
```

Now leaf1 has 300G of uplink, so its ratio is 3.33:1, the failure case we planned for. The other leaves still use spine2, but spine2 no longer has a route to leaf1, so they use the other spines for leaf1's subnet.

## Going Deeper

### Choosing the underlay protocol

| Option | Strengths | Weaknesses | Typical home |
|---|---|---|---|
| eBGP on every link | Simple failure model, rich policy, scales very far, RFC 7938 | Many neighbours and ASNs without automation | Large operators, many EVPN fabrics |
| OSPF or IS-IS, with iBGP for EVPN | Familiar, little config per link | Flooding grows with fabric size, less policy | Enterprises, medium fabrics |
| iBGP with route reflectors only | One ASN | Needs an IGP or next-hop tricks, more moving parts | Less common |

An **IGP** is an interior routing protocol such as OSPF or IS-IS. A **route reflector** is a BGP router that passes routes between iBGP peers.

### AS number design and path hunting

Why do all spines share one ASN in the usual eBGP design? To prevent **path hunting**.

Say a leaf withdraws a prefix. If each spine had its own ASN, spines could still hold stale paths through other leaves and spines, and try each one in turn before giving up. This can slow convergence.

With a shared spine ASN, BGP loop prevention helps. A path that has passed through one spine has 65000 in its AS path. Another spine refuses it, because it sees its own ASN. So a path can never go spine to leaf to spine.

The 2-byte private ASN range is 64512-65534, which is only 1,023 numbers. RFC 6996 also reserves 4-byte private ASNs (4200000000-4294967294). Those suit large fabrics. RFC 7938 also describes reusing ASNs on leaves in different clusters, together with features that allow the same ASN to appear in a path.

### BGP unnumbered

Assigning two addresses to each of hundreds of links is tedious. **BGP unnumbered** skips it. Each side uses its automatic IPv6 link-local address on the interface, and finds its neighbour with IPv6 Router Advertisements. IPv4 routes can then carry an IPv6 next hop, as RFC 8950 defines (it replaced RFC 5549).

```text
! Linux, FRR on a leaf: neighbours are named by interface
router bgp 65001
 neighbor swp51 interface remote-as external
 neighbor swp52 interface remote-as external
 address-family ipv4 unicast
  maximum-paths 8
 exit-address-family
```

`remote-as external` means "any different AS". This removes the need for per-link addressing. In our 42-leaf design, that is 336 addresses you do not have to plan.

### More on hashing

- **Hash polarisation.** If every layer runs the same hash on the same fields, a flow's choices line up. In a five-stage fabric, flows that went left at the leaf could all go left again at the spine, and some links stay idle. The fix is a per-switch seed or a different hash at each tier.
- **Hash inputs.** Fields can include MAC addresses, IP addresses, ports, and inner headers. Check the settings. A storage system using a single address pair and port may give you one flow, however many paths you have.
- **VXLAN entropy.** Outer headers of tunnelled traffic would all look the same. RFC 7348 recommends deriving the outer UDP source port from a hash of the inner headers, so ECMP can still spread flows.
- **Unequal links.** If one spine has less capacity toward a leaf, equal ECMP sends it too much. Weighted ECMP solves that, for example with the BGP link bandwidth extended community, where supported.
- **Flowlets.** A burst of a flow separated by a gap can safely move to another path without reordering, if the gap is longer than the delay difference between paths. Some switches use this.
- **Packet spraying.** Some fabrics send packets of one flow over all paths, and let a smart NIC (network card) reorder them. This fits lossless GPU fabrics more than general traffic.

### Buffers and incast

**Incast** is many senders replying to one receiver at the same moment, such as 100 storage nodes replying to one client. All replies meet at one leaf port and overflow its buffer, even if the fabric has plenty of capacity. Spine-leaf cannot solve this, because the bottleneck is the last link.

Tools that help:

- **ECN (Explicit Congestion Notification)**, RFC 3168: a switch marks packets when a queue grows, so senders slow down before drops begin.
- **DCTCP**, RFC 8257: a TCP variant for data centres that reacts to ECN marks in proportion to how much marking it sees.
- **Deep-buffer switches**: useful at some points, such as storage or WAN edges, but they can add delay.
- **PFC (Priority Flow Control)**, IEEE 802.1Qbb: tells the upstream device to pause one traffic class. This supports lossless traffic but can cause pause spreading and deadlocks if done badly.

See [Quality of Service (QoS)](03-quality-of-service.md) and [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).

### GPU training fabrics

Large machine learning clusters push spine-leaf hard. Thousands of GPUs exchange data in the same step, so a few huge flows run at the same time. A single slow link delays the entire job.

Such fabrics often use:

- 1:1 subscription, with 400G or 800G links.
- **RoCEv2 (RDMA over Converged Ethernet)**, which moves memory directly between machines over UDP port 4791, and is sensitive to loss.
- ECN and PFC tuning.
- Adaptive routing or packet spraying rather than plain flow hashing.

The Ultra Ethernet Consortium published its 1.0 specification in 2025, aimed at this kind of workload. InfiniBand remains the main non-Ethernet alternative.

### Server attachment choices

| Method | How it works | Trade-off |
|---|---|---|
| MLAG or vPC | Two leaves act as one LAG partner | Mature, works with any server, but adds state between the pair |
| EVPN multihoming | Leaves share a segment using BGP (RFC 7432) | No peer link, needs EVPN |
| Routing to the host | Server is a BGP router | Very clean network, more server work |
| Single attached | One link to one leaf | Simple and cheap, the app must survive rack loss |

For LAG basics, see [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md).

### Other design items

- **Default gateway.** With routing at the leaf, the gateway is on the leaf. Protocols such as HSRP and VRRP are not needed with routing to the host. Overlays use an **anycast gateway**, the same gateway IP and MAC on every leaf. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).
- **MTU.** VXLAN adds about 50 bytes. Fabric links often run jumbo frames of around 9,000 bytes or more. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).
- **Security.** East-west traffic does not cross a central firewall by default. Use segmentation or host policy. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).
- **Draining.** Before spine work, steer traffic away from it. AS path prepending (repeating an ASN to make a path look longer) on the spine, or a vendor maintenance mode, moves flows off gently before the shutdown.
- **Spines stay simple.** No servers, no gateways, no firewalls on spines. That keeps them replaceable.
- **Monitoring.** Watch per-link utilisation for imbalance, drops, and optics errors. Compare uplinks: a large gap between them suggests elephants or a hashing issue. See [Network Monitoring](../12-troubleshooting-and-tools/11-network-monitoring.md).
- **Small sites.** Two switches may be enough for ten servers. Spine-leaf pays off once you outgrow a pair. See [High Availability and Redundancy](07-high-availability.md).

## Common Mistakes

- **"Spine-leaf means two hops, so all latency is equal."** Path length is equal. Queues are not. A congested link can add far more delay than the switch hop.
- **"Spine-leaf has three layers because of three stages."** It has two layers of switches. "Three-stage" counts the leaf, spine, leaf path.
- **"ECMP balances traffic evenly."** ECMP balances flows by hash, not bytes. One elephant can fill a link while others are quiet.
- **"ECMP is per-packet."** Normal ECMP is per-flow, to prevent reordering.
- **"1:1 means nothing can ever be congested."** Collisions and incast still happen. A 1:1 ratio is a total capacity statement.
- **"Connect leaves to each other for extra capacity."** This breaks the equal-path pattern and creates routes that ECMP does not treat as equal.
- **"Connect firewalls or the WAN to the spines."** Use border leaves. Spines should be plain transit.
- **"Spine-leaf must be Layer 3 everywhere, so VLANs cannot span racks."** An overlay such as VXLAN can extend segments across a routed fabric.
- **"Spine-leaf and VXLAN are the same thing."** Spine-leaf is the physical shape. VXLAN is a tunnelling method. Either can exist without the other.
- **Sizing only for the normal state.** Calculate the ratio with one spine down, and check that it is still acceptable.
- **Forgetting `multipath-relax` with dual-homed servers.** Different leaf ASNs in the AS path stop BGP from installing multipath.
- **Letting servers announce anything.** With routing to the host, always filter inbound. A server should never be able to announce a default route.
- **Using the same speed for servers and uplinks.** With 100G servers on 100G uplinks, one hash collision can halve throughput.

## Interview Questions

**Q: Why did data centres move away from three-tier?**

A: Traffic shifted to east-west, between servers. Three-tier has uneven hop counts, it depends on STP which blocks links, its oversubscription multiplies at each layer, and it grows by buying bigger pairs of switches. Spine-leaf gives equal paths, uses all links with ECMP, and grows by adding more switches.

**Q: What is the relationship between Clos and spine-leaf?**

A: Spine-leaf is a folded three-stage Clos network. The ingress and egress stages fold together into the leaf layer, and the middle stage becomes the spine layer. So a packet goes leaf, spine, leaf, which is three stages using two physical layers.

**Q: How does ECMP choose a path, and why not per packet?**

A: The switch hashes header fields, usually the 5-tuple, and uses the result to select a next hop. All packets of one flow go the same way. Per-packet spreading could reorder packets on paths with different delays, and TCP could treat that as loss and slow down.

**Q: A leaf has 48 x 25G server ports and 6 x 100G uplinks. What is the oversubscription?**

A: Downlink is 1,200G and uplink is 600G, so 2:1. With all servers busy across racks, each gets 12.5G. If one of six spines fails, uplink falls to 500G, and the ratio becomes 2.4:1.

**Q: What limits the size of a three-stage fabric?**

A: Spine port count sets the maximum number of leaves. Leaf uplink count sets the maximum number of spines. For 64-port spines and 48-port server leaves, the limit is 64 x 48 = 3,072 server ports. To go beyond that, use larger spines, or a five-stage design with pods and super-spines.

**Q: Why do many fabrics use eBGP instead of OSPF?**

A: BGP scales well, has fine policy control, and fails in a simple way, link by link. RFC 7938 describes this design. Each leaf has its own ASN, and spines share one. OSPF is simpler in smaller fabrics, but link-state flooding touches every router on each change.

**Q: Why do spines share one ASN?**

A: It prevents path hunting. A route cannot go from spine to leaf and then to another spine, because that spine sees its own ASN in the AS path and rejects it. So a withdrawn prefix does not trigger a search through bad paths.

**Q: What is routing to the host, and why use it?**

A: The server runs BGP, often with FRR, and announces its own /32 loopback to two leaves. It uses ECMP over both links. This removes MLAG and Layer 2 between leaves, and gives the service a stable address. The trade-off is that servers now need a routing daemon, and leaves must filter what they accept.

**Q: A spine fails. Walk me through what happens.**

A: BFD or link-down detects it, in under a second with a 300 ms times 3 timer. BGP removes the routes through that spine. Leaves drop that next hop from ECMP groups, and flows rehash to the others. Capacity falls by one part in N. With resilient hashing, only flows that used the failed spine move.

**Q: Why not connect spines together, or leaves together?**

A: It adds paths of unequal length and breaks the design's predictability. All leaf pairs are already joined through every spine, so these links add little for fabric traffic. The only common exception is a peer link for an MLAG pair of leaves.

**Q: All links show low average use, but applications report drops. What do you suspect?**

A: Microbursts or incast. One-minute averages hide bursts that fill a buffer. I would check egress queue drops on server-facing ports, enable finer telemetry, and consider ECN and buffer tuning. I would also compare uplink utilisation to rule out ECMP imbalance.

**Q: How do you stretch a Layer 2 segment across a routed spine-leaf fabric?**

A: Use an overlay. VXLAN carries Layer 2 frames in UDP across the routed underlay, and EVPN uses BGP to share MAC and IP reachability. The physical fabric stays pure Layer 3, with no STP, and the segment appears wherever it is needed.

## Quick Recap

- Spine-leaf is a folded Clos: every leaf links to every spine, and leaves or spines never link to each other for fabric traffic.
- It replaces three-tier because east-west traffic needs equal paths, active links and scale out, not STP and bigger pairs.
- ECMP hashes each flow onto one of many equal paths. It balances flows, not bytes, so keep uplinks faster than server links.
- Oversubscription is downlink capacity divided by uplink capacity. Calculate it with a spine failed too.
- A three-stage fabric maxes out at spine ports x leaf server ports. Past that, use five stages with pods and super-spines.
- Modern fabrics are Layer 3, often eBGP on every link, with VXLAN and EVPN on top when Layer 2 is needed.
- Routing to the host makes the server a BGP speaker with a stable /32, and removes MLAG at the cost of more server responsibility.
