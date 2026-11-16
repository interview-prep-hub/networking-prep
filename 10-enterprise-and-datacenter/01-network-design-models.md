# Network Design Models

**Level:** Intermediate | **Prerequisites:** [VLANs](../03-ethernet-and-switching/06-vlans.md), [Spanning Tree Protocol](../03-ethernet-and-switching/09-spanning-tree.md), [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md), [VLSM and Route Summarisation](../04-ip-addressing/07-vlsm-and-supernetting.md)

## In One Line

A network design model is a proven way to arrange switches, routers and links in layers, so the network is fast, easy to fix and easy to grow, and one failure stays small.

## Why It Matters

Imagine 3,000 people on one big flat network. "Flat" means every device sits in the same Layer 2 network, with no routers between them. This fails in practice:

- **One broadcast reaches everyone.** Every ARP request and every broadcast hits all 3,000 devices.
- **One loop stops everything.** A cable plugged in wrong can create a broadcast storm across the whole company.
- **Nowhere to put rules.** There is no natural point to say "guests cannot reach the finance servers".
- **Growth is painful.** Each new switch adds more links, and nobody can say where traffic really flows.

Design models solve this with structure. Each layer has one job. Each job has a clear place. When something breaks, you know where to look.

Interviewers ask about this because it shows how you think. "Design a network for a 200-person office" is a common question. They want to hear you ask about requirements first, then explain layers, redundancy, trade-offs and an IP plan. Listing products is not enough.

## Simple Explanation

Think of the road system in a country.

- **Local streets** reach every house. They are slow, and there are many of them. This is the **access layer**.
- **City main roads** collect traffic from many local streets. They have traffic lights and rules. This is the **distribution layer**.
- **Motorways** connect cities. They have no traffic lights and no shops. Their only job is speed. This is the **core layer**.

A driver going to the next house stays on the local street. A driver going to another city climbs from street to main road to motorway, then back down. Most trips never touch the motorway, so it stays clear for the long trips.

**Where the analogy stops being true:** a car driver picks a route once and stays on it. In a network, every router makes its own decision for every packet, and traffic can move to another path in under a second when a link fails. Also, a network link can be upgraded by changing a small module, which is much easier than widening a road.

## How It Works

### Terms you need first

| Term | Plain meaning |
|---|---|
| Hierarchy | Devices arranged in layers, each with a different job |
| Uplink | A link from a lower layer up to a higher layer |
| Redundancy | Having a second device or path ready if the first one fails |
| Failure domain | The part of the network affected when one thing breaks |
| Single point of failure | One thing that, if it breaks, takes a service down |
| Oversubscription | Having more port capacity below than uplink capacity above, because not everyone sends at once |
| Layer 2 domain | Devices that reach each other by MAC address, without a router |
| MDF | Main distribution frame, the main wiring room of a site |
| IDF | Intermediate distribution frame, a wiring closet on a floor |

### The three-tier model

The three-tier model is also called the hierarchical model. Cisco made it popular, and it is now general practice. It is a design pattern, not a formal standard.

```text
             To internet, WAN and data centre
                  |               |
CORE         +----+----+     +----+----+
(speed)      | Core 1  |=====| Core 2  |
             +--+---+--+     +--+---+--+
                |    \       /   |
                |     \     /    |
                |      \   /     |
                |       \ /      |
                |        X       |
                |       / \      |
                |      /   \     |
DISTRIBUTION +--+---+-+     +-+--+---+
(policy)     | Dist 1 |=====| Dist 2 |
             +--+---+-+     +-+--+---+
                |    \       /   |
                |     \     /    |
                |      \   /     |
                |       \ /      |
                |        X       |
                |       / \      |
                |      /   \     |
ACCESS       +--+---+-+     +-+--+---+
(connect)    | Acc 1  |     | Acc 2  |
             +-+-+-+-++     ++-+-+-+-+
               | | | |       | | | |
             PCs, phones   APs, printers

  ==== link between the two switches of a pair
```

Every lower device connects to two upper devices. That is why the diagram has crossing lines. Lose any one box or any one line, and a path still exists.

### Access layer: where devices connect

The access layer is the switch in the floor closet. Laptops, desk phones, printers, cameras and Wi-Fi access points (APs) plug in here.

Its jobs:

- Give many ports at low cost, usually 1 Gbps or 2.5 Gbps per port.
- Put each port in the right [VLAN](../03-ethernet-and-switching/06-vlans.md), meaning the right virtual network.
- Supply Power over Ethernet (PoE), which sends power down the data cable to phones and APs.
- Stop bad devices at the door, using [Layer 2 security](../03-ethernet-and-switching/11-layer2-security.md) such as port security, DHCP snooping and 802.1X login.
- Mark traffic priority, so voice can be treated better. See [Quality of Service](03-quality-of-service.md).

Copper Ethernet is limited to 100 m per run. That one fact often decides how many closets a building needs.

### Distribution layer: where rules live

The distribution layer collects many access switches. It usually sits in a building's main wiring room.

Its jobs:

- Combine the uplinks from all access switches in a building or zone.
- Be the default gateway for VLANs, doing [inter-VLAN routing](../03-ethernet-and-switching/08-inter-vlan-routing.md).
- Apply policy, such as [ACLs](../09-security/05-access-control-lists.md) (lists of allow and deny rules).
- Summarise routes, so the core sees one short route for a whole building.
- Mark the boundary between Layer 2 below and Layer 3 above.
- Contain failures. A loop on one floor stays below the distribution layer.

### Core layer: where speed lives

The core is the backbone. It connects distribution blocks to each other and to the data centre, WAN and internet edge.

Its jobs:

- Move packets as fast as possible.
- Stay up, always. If the core fails, every block loses contact with every other block.

The core does **not** do these things: connect end users, run complex ACLs, inspect traffic or carry VLANs across the campus. Each extra feature adds risk and work. In a modern campus, core links are routed Layer 3 links, with no spanning tree in the path.

| | Access | Distribution | Core |
|---|---|---|---|
| Main job | Connect devices | Apply policy, aggregate | Fast transport |
| Typical port speed | 1 to 2.5 Gbps, some 5 or 10 | 10 to 40 Gbps | 40 to 100 Gbps or more |
| Features | PoE, port security, 802.1X | Routing, ACLs, gateway | Routing only |
| Failure hurts | One floor or closet | One building | Everything |

Port speeds in this table are typical in 2026, not rules.

### Two-tier design: the collapsed core

A collapsed core merges the core and distribution jobs into one pair of switches. The name means the two layers are "collapsed" into one.

```text
                 Internet edge
                       |
        +--------------+--------------+
        |     Collapsed core pair     |
        | (core + distribution jobs)  |
        |  CC1 ===================CC2 |
        +--+--------+--------+-----+--+
           |        |        |     |
        +--+--+  +--+--+  +--+--+  +----------+
        |Acc 1|  |Acc 2|  |Acc 3|  | Servers  |
        +-----+  +-----+  +-----+  +----------+
        Floor 1  Floor 2  Floor 3
```

Why use it:

- Fewer boxes, so lower cost, less power and less to manage.
- Enough for one building or a few close buildings.

Why not use it everywhere: the same pair does everything. As you add blocks, port counts grow, and policy work competes with fast forwarding. A software upgrade on that pair touches the whole site.

Why a separate core appears as you grow: connecting every block directly to every other block needs n(n-1)/2 connections.

| Blocks | Connections for a full mesh | Connections through a core pair (one per block) |
|---|---|---|
| 4 | 6 | 4 |
| 8 | 28 | 8 |
| 12 | 66 | 12 |

Each "connection" here is really two or more cables for redundancy, so the real cable count is higher. A core turns a mess of links into a simple star.

### Which model should you pick?

There is no standard number that forces a choice. These are rules of thumb.

| Situation | Sensible model |
|---|---|
| Up to about 50 users, one floor | Firewall plus one switch or stack, no separate tiers |
| One building, a few hundred users | Two-tier collapsed core |
| Several buildings, thousands of users | Three-tier |
| Server farm with heavy server-to-server traffic | [Spine-leaf](02-spine-leaf.md) |

Fuller truth: real networks mix models. A company may run three-tier at head office, collapsed core in a regional office, and a single stack in a shop.

### Redundancy

Redundancy means no single failure stops the service. Think of two bridges across a river. If one closes, traffic uses the other.

**Where the analogy stops being true:** a network's second "bridge" is not idle. It usually carries traffic at the same time. A network also needs a protocol to notice a failure and move traffic, and the time this takes matters.

Add redundancy at each level:

| Level | Technique | Protects against |
|---|---|---|
| Inside a device | Two power supplies, two supervisor cards | Part failure |
| Between devices | Link aggregation (LAG) | Cable or port failure |
| Device pairs | Stacking, virtual switch, MLAG | Whole switch failure |
| Gateway | [HSRP, VRRP, GLBP](../05-routing/11-first-hop-redundancy.md) | Gateway failure |
| Paths | Equal-cost multipath routing (ECMP) | Link or router failure |
| Providers | Two ISPs, different cable routes | ISP outage, cable cut |
| Power and room | UPS, two power feeds, second site | Power or room failure |

Recovery speed depends on the mechanism. These are typical values, and they change with vendor, version and tuning.

| Event | Mechanism | Typical recovery |
|---|---|---|
| One LAG member fails | LACP or EtherChannel | Usually under a second |
| Layer 2 uplink fails | Rapid Spanning Tree | Often a second or two |
| Layer 2 uplink fails | Classic 802.1D STP | 30 to 50 seconds |
| Gateway dies | HSRP, Cisco default timers | Up to 10 seconds (hold time) |
| Routed link goes down | OSPF sees link loss | Usually under a second or two |
| Link stays up but path is dead | OSPF, default broadcast timers | Up to 40 seconds (dead time) |
| Same case with BFD | Fast failure detection protocol | Well under a second |

**The math of redundancy.** A device with 99.9% availability is down about 8.8 hours a year. Two independent devices in parallel give 1 - (0.001 x 0.001) = 99.9999%, about 32 seconds a year. This assumes they fail independently. In real life they share software, power and rooms, so the real gain is smaller.

Three rules keep redundancy honest:

1. **Check surviving capacity.** Two links at 70% load each will overload one link at 140% after a failure. For active-active pairs, keep normal peak load under about 50% per link.
2. **Watch shared risk.** Two fibres in the same duct are one fibre to a digger. Two ISPs may share the same street cable.
3. **Test it.** A backup path that nobody has tested is only a hope.

More depth lives in [High Availability and Redundancy](07-high-availability.md).

### North-south and east-west traffic

These names describe the direction of traffic. They come from diagrams, which put the internet at the top.

- **North-south traffic** crosses the edge of a network: users to the internet, or clients into a data centre.
- **East-west traffic** stays inside: server to server, or service to service.

Think of an office building. People coming in and out of the front door are north-south. People walking between offices inside are east-west.

**Where the analogy stops being true:** there is no real compass here. The words depend on which boundary you mean. A laptop reaching the company data centre is north-south for that data centre, yet it never leaves the company.

```text
                 Internet
                    ^
                    |  north-south (client request)
                    v
              +-----------+
              |  Firewall |
              +-----+-----+
                    |
              +-----+-----+
              | Web server|
              +--+-----+--+
                 |     |     east-west (inside the data centre)
        +--------+     +--------+
        v                       v
  +-----------+           +-----------+
  | Auth      |<--------->| Database  |
  | service   |           |           |
  +-----------+           +-----------+
```

One click on a shopping site is one north-south request. Behind it, the web server may call an auth service, a cart service, a cache and a database. That is many east-west flows for one north-south flow.

Why this changes the design:

| Question | North-south heavy (campus) | East-west heavy (modern data centre) |
|---|---|---|
| Where is the bottleneck? | Internet edge, uplinks | Links between server racks |
| Best fit | Three-tier or collapsed core | Spine-leaf |
| Where are the firewalls? | At the edge | Edge, plus [segmentation](../09-security/09-zero-trust-and-segmentation.md) close to workloads |

The edge firewall never sees traffic between two servers in one VLAN. If an attacker gets into one server, they can move sideways unless east-west traffic is also controlled.

In a three-tier data centre, two servers in different pods may send traffic up through access, aggregation (the data centre word for distribution) and core, then back down. In spine-leaf, any two racks are always leaf, spine, leaf.

```text
Three-tier path, different pods    Spine-leaf path, any two racks

Access > Aggr > Core > Aggr > Access       Leaf > Spine > Leaf
        5 switches                             3 switches
```

Classic three-tier data centres also used Spanning Tree, which blocks redundant links. So part of the bandwidth you paid for sat unused. See [Spine-Leaf Architecture](02-spine-leaf.md).

### Steps to design any network

Follow this order. It also makes a strong interview answer.

1. **Ask for requirements.** How many users and devices? Which apps? Cloud or on-site? How many buildings and floors? What growth over 3 to 5 years? How much does one hour of downtime cost? What is the budget?
2. **Map the physical site.** Where are the rooms? Are cable runs under 100 m? Is there fibre between buildings?
3. **Choose the model.** Use the size and traffic pattern.
4. **Segment.** Split people and devices into VLANs by role: staff, voice, guest, IoT, management.
5. **Plan the IP addresses.** Use one block per site, and leave room to grow.
6. **Size the links.** Check oversubscription and the failed state.
7. **Add redundancy.** Match it to the cost of downtime.
8. **Add security.** Decide who can reach what, and where that is enforced.
9. **Plan operations.** Include monitoring, backups of configs, spares and documentation.

## Example

### Example 1: a small office

**Requirements.** 40 staff on one floor. Desk phones, 5 printers, a small file server (NAS), guest Wi-Fi. Email and most apps are in the cloud. No IT person on site. A two-hour outage is annoying, but not a disaster.

**Choice.** No tiers are needed. One firewall does routing and security. A two-switch stack does access. A stack means several physical switches act as one switch.

**Port count.** 40 desks + 4 APs + 5 printers + 1 NAS + 2 firewall ports = 52 ports. Add about 20% spare and you need about 62. Two 48-port PoE switches give 96.

**PoE budget.** Assume 7 W per phone and 25 W per AP: (40 x 7) + (4 x 25) = 380 W. These are planning guesses, so check the real datasheets and pick switches whose total budget covers this.

```text
      ISP fibre 500 Mbps        5G backup router
      203.0.113.2/30            (second WAN port)
            |                        |
            +-----------+------------+
                        |
                 +------+------+
                 |   Firewall  |  Default gateway for all VLANs
                 |     FW1     |  DHCP, NAT, security rules
                 +---+-----+---+
                     |     |      LAG (Port-channel 1)
              +------+-----+------+
              |  SW1 stack        |
              |  member 1 + 2     |
              +--+------+------+--+
                 |      |      |
              Desks    APs    Printers, NAS
```

**IP plan.** The office block is 10.20.0.0/21, which covers 10.20.0.0 to 10.20.7.255. It holds eight /24 subnets. The third octet is the VLAN ID divided by 10, so people can read the plan from memory.

| VLAN | Name | Subnet | Gateway | Usable hosts | DHCP range |
|---|---|---|---|---|---|
| 10 | STAFF | 10.20.1.0/24 | 10.20.1.1 | 254 | .20 to .250 |
| 20 | VOICE | 10.20.2.0/24 | 10.20.2.1 | 254 | .20 to .250 |
| 30 | PRINT-IOT | 10.20.3.0/24 | 10.20.3.1 | 254 | Static or reserved |
| 40 | GUEST | 10.20.4.0/24 | 10.20.4.1 | 254 | .20 to .250 |
| 50 | MGMT | 10.20.5.0/24 | 10.20.5.1 | 254 | None, static only |
| - | Spare | 10.20.0.0/24, 10.20.6.0/24, 10.20.7.0/24 | - | - | - |

Address rule: .1 is the gateway, .2 to .19 are static (switches, NAS), .20 to .250 are DHCP. The NAS is 10.20.1.10 in the STAFF VLAN, so big file copies do not pass through the firewall.

The WAN link is 203.0.113.0/30. The ISP router is 203.0.113.1 and FW1 is 203.0.113.2.

**Switch config.**

```text
! Cisco IOS: office switch stack SW1
vlan 10
 name STAFF
vlan 20
 name VOICE
vlan 30
 name PRINT-IOT
vlan 40
 name GUEST
vlan 50
 name MGMT
!
interface GigabitEthernet1/0/1
 description Desk 01, PC behind phone
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 spanning-tree portfast
!
interface GigabitEthernet1/0/45
 description AP-1 (untagged VLAN 50 for AP management)
 switchport mode trunk
 switchport trunk native vlan 50
 switchport trunk allowed vlan 10,40,50
 spanning-tree portfast trunk
!
interface range GigabitEthernet1/0/48, GigabitEthernet2/0/48
 description Uplink to FW1
 channel-group 1 mode active
!
interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50
!
interface Vlan50
 ip address 10.20.5.2 255.255.255.0
!
ip default-gateway 10.20.5.1
```

**Firewall rules.**

| From | To | Action |
|---|---|---|
| STAFF | Internet | Allow |
| VOICE | Phone provider only | Allow |
| GUEST | Any private address | Deny |
| GUEST | Internet | Allow, with a rate limit |
| PRINT-IOT | Internet | Deny |
| STAFF | PRINT-IOT, printing ports only | Allow |
| Admin laptop | MGMT | Allow |

**What fails, and what happens.**

| Failure | Effect | Answer |
|---|---|---|
| ISP fibre cut | Firewall health check moves traffic to 5G | Slower, but working |
| One stack member dies | Half the desks go down. The LAG stays up | Split APs across members. Keep a spare switch |
| Firewall dies | Office offline | Accepted risk. Vendor replaces it next day |
| Building power cut | All down | Small UPS on firewall and switches |

A second firewall would remove one single point of failure, but for this office the cost beats the risk. Saying this out loud in an interview shows good judgement. Not every office needs maximum redundancy.

### Example 2: a three-building campus

**Requirements.** 3,000 users in 15 floors. Building A has 6 floors, B has 5, C has 4. Each floor has 200 users. There is an on-site data centre room and two ISP links of 2 Gbps. Downtime is expensive.

**Choice.** Three-tier. There are five blocks to connect (three buildings, data centre, internet edge), and the buildings are separate. A full mesh would need 10 block-to-block connections.

```text
           ISP-1 (2 Gbps)         ISP-2 (2 Gbps)
                 \                    /
            +-----+------------------+-----+
            |  Internet edge: 2 routers,   |
            |  2 firewalls, DMZ            |
            +--------------+---------------+
                           |
            +--------------+---------------+
            |  Core1 =============== Core2 |
            |  Layer 3 only, area 0        |
            +--+--------+--------+------+--+
               |        |        |      |
          +----+---+ +--+----+ +-+-----+ +-+------+
          | DistA  | | DistB | | DistC | | DC sw  |
          | area 1 | | area 2| | area 3| | area 4 |
          +-+-+-+--+ +--+-+--+ +--+-+--+ +---+----+
            | | |       | |       | |        |
         6 floors   5 floors  4 floors    Servers

Each box is a pair of switches. Each line is at least two links.
```

**Layers in this design.**

- **Access:** each floor has a stack of 6 switches with 48 ports of 1 Gbps. Each stack has two 10 Gbps uplinks, from different members, in one LAG.
- **Distribution:** each building has two switches running as one logical switch. Cisco StackWise Virtual is one example of this. The floor LAG lands on both physical switches, so both uplinks forward and Spanning Tree blocks nothing.
- **Core:** two independent Layer 3 switches. They are not merged into one, so a software fault on one does not hit the other.
- **Routing:** OSPF, with the core in area 0 and each building in its own area. See [OSPF](../05-routing/06-ospf.md).

**Link sizing.** The oversubscription ratio is total port capacity below, divided by uplink capacity above.

| Link | Below | Above | Ratio |
|---|---|---|---|
| Floor to DistA | 6 x 48 x 1 Gbps = 288 Gbps | 2 x 10 Gbps = 20 Gbps | 14.4:1 |
| DistA to core | 6 floors x 20 Gbps = 120 Gbps | 4 x 25 Gbps = 100 Gbps | 1.2:1 |

A widely quoted Cisco rule of thumb is up to 20:1 from access to distribution, and up to 4:1 from distribution to core. Both links pass.

Now check the failed state. If one floor uplink dies, that floor has 10 Gbps, so the ratio is 28.8:1 until repair. This is above the guideline, but real users rarely send at full speed at once. Monitoring must confirm that peak use fits in 10 Gbps.

**IP plan: top level.** The campus block is 10.40.0.0/13, which covers 10.40.0.0 to 10.47.255.255. It holds eight /16 blocks. It does not overlap the office in Example 1, so the two can join by VPN later.

| Block | Use |
|---|---|
| 10.40.0.0/16 | Infrastructure: loopbacks, point-to-point links |
| 10.41.0.0/16 | Building A (OSPF area 1) |
| 10.42.0.0/16 | Building B (OSPF area 2) |
| 10.43.0.0/16 | Building C (OSPF area 3) |
| 10.44.0.0/16 | Data centre (OSPF area 4) |
| 10.45.0.0/16 | Internet edge and DMZ |
| 10.46.0.0/16 | Guest Wi-Fi |
| 10.47.0.0/16 | Reserved for growth |

**IP plan: inside a building.** The second octet is the building. The third octet equals the VLAN ID, and its tens digit is the floor. Here is Building A.

| VLAN | Purpose | Subnet | Gateway |
|---|---|---|---|
| 10, 11, 12 | Floor 1 data, voice, IoT | 10.41.10.0/24, 10.41.11.0/24, 10.41.12.0/24 | .1 of each |
| 20, 21, 22 | Floor 2 data, voice, IoT | 10.41.20.0/24, 10.41.21.0/24, 10.41.22.0/24 | .1 of each |
| 30, 31, 32 | Floor 3 data, voice, IoT | 10.41.30.0/24, 10.41.31.0/24, 10.41.32.0/24 | .1 of each |
| 40 to 62 | Floors 4 to 6, same pattern | 10.41.40.0/24 to 10.41.62.0/24 | .1 of each |
| 99 | Switch management | 10.41.99.0/24 | 10.41.99.1 |
| 200 | Staff Wi-Fi, whole building | 10.41.200.0/21 | 10.41.200.1 |

Buildings B and C copy this pattern, so floor 4 data in Building B is 10.42.40.0/24.

Why these sizes:

- A floor data /24 has 254 usable addresses. With 200 users there is room for growth.
- In each /24, .1 is the gateway, and .2 and .3 stay reserved in case the pair later becomes two separate gateways. Then .4 to .20 are static and .21 to .250 are DHCP.
- Staff Wi-Fi gets a /21, with 2,046 usable addresses, because Building A has 1,200 people and many carry two devices. The subnet is wide so that a laptop keeps its IP when moving between floors. Real Wi-Fi designs often tunnel this traffic to a controller instead. See [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).

**Infrastructure and data centre addresses.**

| Item | Address | Notes |
|---|---|---|
| Core1 loopback | 10.40.0.1/32 | Loopback: a virtual interface that stays up |
| Core2 loopback | 10.40.0.2/32 | Also used as router ID |
| DistA, DistB, DistC loopbacks | 10.40.0.11, .21, .31 | /32 each |
| Core1 to DistA link | 10.40.1.0/31 | Core1 .0, DistA .1 |
| Core2 to DistA link | 10.40.1.2/31 | Core2 .2, DistA .3 |
| Core1 to DC link | 10.40.1.12/31 | Core1 .12, DC .13 |
| Core2 to DC link | 10.40.1.14/31 | Core2 .14, DC .15 |
| DC shared services, VLAN 10 | 10.44.10.0/24 | DHCP server at 10.44.10.20 |
| DC applications, VLAN 20 | 10.44.20.0/24 | App server at 10.44.20.30 |
| DC databases, VLAN 30 | 10.44.30.0/24 | |
| Guest Wi-Fi | 10.46.0.0/20 | 4,094 usable, tunnelled to the firewall |

A /31 has only two addresses, and on a point-to-point link both are usable (RFC 3021). A /30 would waste two addresses per link.

**IPv6 plan.** Say the campus has 2001:db8:100::/48. A /48 leaves 16 bits for subnets, which gives 65,536 /64 subnets. Each VLAN gets one /64. To make the plan easy to read, reuse the IPv4 numbers as digits.

| IPv4 subnet | IPv6 subnet | Gateway |
|---|---|---|
| 10.41.30.0/24 | 2001:db8:100:4130::/64 | 2001:db8:100:4130::1 |
| 10.44.20.0/24 | 2001:db8:100:4420::/64 | 2001:db8:100:4420::1 |

Here 4130 is a label, not a math conversion. Read it as "Building 41, VLAN 30". See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

**Gateway and routing config.**

```text
! Cisco IOS: DistA (two switches acting as one logical switch)
interface Vlan30
 description A-F3-DATA
 ip address 10.41.30.1 255.255.255.0
 ip helper-address 10.44.10.20
!
interface Port-channel13
 description LAG to Floor 3 access stack
 switchport mode trunk
 switchport trunk allowed vlan 30,31,32,99,200
!
interface Port-channel1
 description Routed LAG to Core1
 no switchport
 ip address 10.40.1.1 255.255.255.254
!
interface Port-channel2
 description Routed LAG to Core2
 no switchport
 ip address 10.40.1.3 255.255.255.254
!
router ospf 1
 router-id 10.40.0.11
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface Port-channel2
 network 10.40.0.11 0.0.0.0 area 0
 network 10.40.1.0 0.0.0.255 area 0
 network 10.41.0.0 0.0.255.255 area 1
 area 1 range 10.41.0.0 255.255.0.0
```

The last line is route summarisation. Building A has dozens of subnets, but the core sees only 10.41.0.0/16. If a floor subnet goes up or down, the core does not need to recalculate.

```text
DistA# show ip route ospf
(codes and other lines trimmed)
O IA     10.42.0.0/16 [110/3] via 10.40.1.2, 00:42:10, Port-channel2
                      [110/3] via 10.40.1.0, 00:42:10, Port-channel1
O IA     10.43.0.0/16 [110/3] via 10.40.1.2, 00:42:10, Port-channel2
                      [110/3] via 10.40.1.0, 00:42:10, Port-channel1
O IA     10.44.0.0/16 [110/3] via 10.40.1.2, 00:42:10, Port-channel2
                      [110/3] via 10.40.1.0, 00:42:10, Port-channel1
O*E2  0.0.0.0/0 [110/1] via 10.40.1.2, 00:42:10, Port-channel2
                [110/1] via 10.40.1.0, 00:42:10, Port-channel1
```

Three things to read here. Each destination is one short summary. Each has two equal paths, one through each core switch. The default route (0.0.0.0/0) leads to the internet edge.

**Follow one packet.** A laptop on Building A, floor 3, opens an app on 10.44.20.30.

1. The laptop has 10.41.30.55/24. The destination is outside its subnet, so it sends the frame to its gateway, 10.41.30.1.
2. The floor stack hashes the flow onto one member of its LAG and sends the frame to DistA.
3. DistA routes the packet. It finds 10.44.0.0/16 with two equal paths and picks one by hashing the flow.
4. Core1 forwards it to the data centre switch.
5. The data centre switch delivers it into VLAN 20, to the server.

```powershell
# Windows, on the laptop
PS C:\> tracert -d 10.44.20.30

Tracing route to 10.44.20.30 over a maximum of 30 hops

  1     1 ms     1 ms     1 ms  10.41.30.1
  2     1 ms     1 ms     1 ms  10.40.1.0
  3     1 ms     1 ms     1 ms  10.40.1.13
  4     1 ms     1 ms     1 ms  10.44.20.30

Trace complete.
```

```bash
# Linux, on a machine in the same VLAN
traceroute -n 10.44.20.30
# traceroute to 10.44.20.30 (10.44.20.30), 30 hops max, 60 byte packets
#  1  10.41.30.1  0.812 ms  0.774 ms  0.731 ms
#  2  10.40.1.0   0.944 ms  0.902 ms  0.871 ms
#  3  10.40.1.13  1.103 ms  1.067 ms  1.031 ms
#  4  10.44.20.30 1.228 ms  1.190 ms  1.162 ms
```

Each hop matches the plan: DistA, then Core1, then the data centre switch, then the server. A run could show 10.40.1.2 and 10.40.1.15 instead, the path through Core2. With ECMP, different flows can take different equal paths.

**What fails, and what happens.**

| Failure | Effect |
|---|---|
| One floor uplink | LAG keeps running on the other link, with half the bandwidth |
| One physical DistA switch | Floors stay up on the other. Capacity drops by half |
| Core1 | OSPF sends everything through Core2 |
| One ISP | Traffic and routes move to the other ISP. See [BGP](../05-routing/09-bgp.md) |
| One floor stack | That floor is down, but other floors do not notice |

The last row shows the failure domain doing its job. The problem stays on one floor.

### IP planning rules that apply to both examples

- **One block per site.** Then a whole site is one route.
- **Stay on bit boundaries.** 10.40.0.0/13 is valid because 40 is a multiple of 8. 10.42.0.0/13 is not a real boundary.
- **Leave gaps.** Keep whole blocks unused for growth.
- **Make numbers mean something.** If VLAN 30 is 10.41.30.0/24, people need less documentation.
- **Avoid 192.168.0.0/24 and 192.168.1.0/24.** Home routers often use them, so VPN users get address clashes.
- **Check other networks.** Your cloud networks, partners and containers must not overlap. Docker's default bridge uses 172.17.0.0/16, which is a common surprise. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).
- **Write it down.** Keep one source of truth, such as an IPAM tool. IPAM means IP address management.

## Going Deeper

### Where does Layer 2 end?

The biggest design decision is where Layer 2 stops and routing starts.

```text
Layer 2 access                       Routed access

   [ Distribution ]  gateway here       [ Distribution ]
        |    |                               |    |
     L2 trunks                            L3 links (ECMP)
        |    |                               |    |
     [ Access ]                           [ Access ]  gateway here
```

| | Layer 2 access | Routed access |
|---|---|---|
| Gateway location | Distribution | Access switch |
| Failure domain | Larger | Smaller, often one closet |
| Spanning Tree on uplinks | Needed, or use a LAG to a virtual pair | Not needed, uplinks are routed |
| Recovery | Depends on STP or LAG | Routing protocol, usually fast |
| VLAN across closets | Possible | Not possible without an overlay |
| Cost | Lower | Access switches must route |

Overlay networks remove the last limit. An overlay is a virtual network built on top of a routed one. Cisco SD-Access is one campus example, using LISP for control and VXLAN for data. EVPN with VXLAN is the common multi-vendor choice. See [VXLAN and EVPN](11-vxlan-and-evpn.md).

### The classic design: two gateways and Spanning Tree

Example 2 used a logical switch pair. Many networks still run two independent distribution switches with HSRP. HSRP gives one shared virtual gateway IP. Here is how Building A would look.

```text
! Cisco IOS: DistA-1, independent switch running HSRP
interface Vlan30
 ip address 10.41.30.2 255.255.255.0
 standby version 2
 standby 30 ip 10.41.30.1
 standby 30 priority 110
 standby 30 preempt
 ip helper-address 10.44.10.20
```

```text
DistA-1# show standby brief
                     P indicates configured to preempt.
                     |
Interface  Grp  Pri P State   Active          Standby         Virtual IP
Vl30       30   110 P Active  local           10.41.30.3      10.41.30.1
```

Two lessons appear here:

- **Your 20 Gbps may really be 10 Gbps.** One floor VLAN sends upward only toward the active gateway. Spanning Tree may also block one uplink. So the 14.4:1 ratio holds only if both links carry traffic. People often split VLANs, with data active on one switch and voice active on the other, to use both links.
- **Keep STP and HSRP aligned.** Make the switch that is the STP root for a VLAN also the active gateway for that VLAN. If not, traffic takes a needless detour across the link between the switches.

Trade-offs of a logical pair: it is simpler and uses both links, but the two switches share one control plane, so a software bug can hit both. MLAG, such as Cisco vPC on Nexus, keeps two control planes while still offering one LAG. See [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md).

The summary black hole is another classic risk. Suppose two separate distribution switches both advertise 10.41.0.0/16 and one loses its link to a floor. The core still sends part of the traffic there, because the summary is still advertised. A routed link between the two switches lets that traffic cross to the switch that still has the floor.

### Modular design: the enterprise as building blocks

A big network is split into blocks, and each block can change without touching the others.

```text
+------------+   +-------------+   +--------------+
| Building A |   | Building B  |   | Data centre  |
| block      |   | block       |   | block        |
+-----+------+   +------+------+   +-------+------+
      |                 |                  |
+-----+-----------------+------------------+------+
|                     CORE                        |
+-----+-----------------+------------------+------+
      |                 |                  |
+-----+------+   +------+------+   +-------+------+
| WAN edge   |   | Internet    |   | Shared       |
| branches   |   | edge, DMZ   |   | services     |
+------------+   +-------------+   +--------------+
```

The DMZ is a small, separated network for servers that the internet can reach. See [Firewalls](../09-security/04-firewalls.md). The WAN edge connects branch offices. See [WAN Technologies](08-wan-technologies.md) and [SD-WAN](10-sd-wan.md).

### Oversubscription in practice

Three points go past the basic formula.

- **Data centres use lower ratios.** Servers send much more steadily than people, so a figure around 3:1 is common, and some designs aim for 1:1. These are design choices, not standards.
- **The ratio uses port speed, not traffic.** Measure real peak use. A 14.4:1 link at 30% peak is fine. A 2:1 link at 95% is not.
- **Wi-Fi is changing uplinks.** Modern APs can pass more than 1 Gbps, so their switch ports are often 2.5 Gbps or 5 Gbps. Uplinks must grow to match.

### How cloud changes campus design

When apps move to cloud services, the company data centre carries less, and the internet edge carries more. Many branches now send traffic straight to the internet, instead of through head office. That design shift is covered in [SD-WAN](10-sd-wan.md). Security also moves from "inside is safe" toward [Zero Trust](../09-security/09-zero-trust-and-segmentation.md).

## Common Mistakes

- **Thinking every network needs three tiers.** A 40-person office with a core layer wastes money and adds risk.
- **Thinking collapsed core means no redundancy.** A collapsed core is normally a pair of switches. Only the layers are merged.
- **Putting policy in the core.** Heavy ACLs and inspection belong at the edge or distribution.
- **Stretching VLANs across the campus.** It feels convenient, but it makes one big failure domain.
- **Counting redundant links as extra capacity in a failure.** After a failure, the spare is gone. Size for the failed state.
- **Forgetting the blocked link.** With Spanning Tree, two uplinks do not mean twice the bandwidth.
- **Having two of everything in one room.** Two firewalls on one power strip are still one failure.
- **Using 192.168.1.0/24 for an office.** It will clash with home networks over VPN.
- **Designing the IP plan last.** Readdressing a live network is much harder than planning it first.
- **Saying "east-west means sideways in the rack".** It means traffic staying inside a boundary, and that can be between rooms or sites.
- **Starting with products.** Requirements come first, then the model, then the hardware.

## Interview Questions

**Q: Explain the three-tier network model.**

A: It has three layers. Access connects end devices and gives VLANs, PoE and port security. Distribution aggregates access switches, acts as gateway and applies policy. Core moves traffic fast between blocks, with no policy. The layers make the network easy to scale and fix, and they limit how far a failure spreads.

**Q: Why should the core stay simple?**

A: A core failure hurts everyone, so it must be fast and stable. Every extra feature adds processing, bugs and change risk. So I put policy at distribution or the edge, and the core only routes.

**Q: What is a collapsed core, and when would you use it?**

A: It merges core and distribution into one pair of switches, giving a two-tier design. I use it for one building or a small site, where a separate core is not worth the cost. When the number of blocks grows, a separate core stops the links growing as a full mesh.

**Q: What is the difference between north-south and east-west traffic?**

A: North-south crosses the network edge, such as client to internet or client into a data centre. East-west stays inside, such as server to server. It matters because modern applications create a lot of east-west traffic, so the design needs even paths between servers, and security cannot sit only at the edge.

**Q: Why did data centres move from three-tier to spine-leaf?**

A: Three-tier suits north-south traffic. East-west traffic between pods goes through more switches, and path length varies. Spanning Tree also blocks links. Spine-leaf gives every pair of racks the same leaf, spine, leaf path, and uses all links with ECMP.

**Q: A switch has 48 ports of 1 Gbps and two 10 Gbps uplinks. What is the oversubscription ratio?**

A: Downstream is 48 Gbps and upstream is 20 Gbps, so the ratio is 2.4:1. If one uplink fails, it becomes 4.8:1. I would also check real peak traffic, because the ratio only uses port speeds.

**Q: How do you remove single points of failure from a network?**

A: I go level by level: two power supplies, LAGs for links, pairs of switches, a redundant gateway with HSRP or a logical pair, ECMP routing, and two ISPs with different cable routes. Then I check that the surviving side can carry the load, that the pair does not share power or ducts, and that failover has been tested.

**Q: Does more redundancy always mean more availability?**

A: No. Each extra part adds complexity, and complexity causes outages too. Bad configs, software bugs or a failed failover can take down both sides. I match redundancy to the cost of downtime, so a small office may accept one firewall.

**Q: Layer 2 access or routed access: how do you choose?**

A: Layer 2 access is simpler and cheaper, and VLANs can span closets, but the failure domain is bigger. Routed access moves the gateway to the access switch, so it gives a small failure domain, no STP on uplinks and fast recovery with ECMP. The trade-off is that VLANs cannot span closets without an overlay such as VXLAN.

**Q: Design a network for a 50-person office.**

A: First I ask about apps, growth, budget and the cost of downtime. Then I would use a firewall as gateway, with a PoE switch stack and a few APs. I would use VLANs for staff, voice, guest, IoT and management, from one block like 10.20.0.0/21. I would add a backup internet link, and accept a single firewall if the budget is small, with a next-day replacement contract.

**Q: How would you build an IP plan for a three-building campus?**

A: I take one large block, like 10.40.0.0/13, and split it into a /16 per building, plus blocks for infrastructure, data centre, guest and growth. Inside each building, the VLAN number matches the third octet. Each building summarises to one route at distribution. I use /32 loopbacks and /31 links, and I keep the plan in an IPAM tool.

## Quick Recap

- Three-tier means access (connect), distribution (policy, gateway), core (speed).
- A collapsed core merges core and distribution, and it suits one building or a small site.
- Redundancy exists at every level, but it only counts if the survivor can carry the load and shares no hidden risk.
- North-south crosses a boundary and east-west stays inside it. Heavy east-west traffic is why data centres use spine-leaf.
- Oversubscription is capacity below divided by capacity above. Check it in the failed state too.
- A good IP plan gives one summarisable block per site, leaves room to grow and makes numbers readable.
- Start every design with requirements, and match the design to the cost of downtime.
