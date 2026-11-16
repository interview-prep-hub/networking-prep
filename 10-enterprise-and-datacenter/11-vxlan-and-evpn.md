# VXLAN and EVPN

**Level:** Advanced | **Prerequisites:** [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [VLANs](../03-ethernet-and-switching/06-vlans.md), [ARP](../03-ethernet-and-switching/05-arp.md), [BGP](../05-routing/09-bgp.md), [Spine-Leaf Architecture](02-spine-leaf.md), [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md)

## In One Line

VXLAN wraps a Layer 2 Ethernet frame inside a UDP/IP packet, so a virtual LAN can stretch across a routed network. EVPN uses BGP to tell every switch where each MAC and IP address lives, so the network does not have to flood to find them.

## Why It Matters

A modern data centre is built as a routed spine-leaf fabric. Routing gives many equal paths, fast failover, and no Spanning Tree. But applications still want Layer 2. A virtual machine that moves to another rack wants to keep its IP address, and that means the same subnet must exist on both racks.

So two needs fight each other:

- The physical network wants to be Layer 3 (routed).
- The workloads want Layer 2 (same subnet everywhere).

VXLAN solves this fight. It carries Layer 2 traffic on top of a Layer 3 network.

It also fixes these older problems:

- **Too few VLANs.** A VLAN ID is 12 bits, so you get only 4094 usable VLANs. A cloud provider with many customers runs out.
- **Spanning Tree.** [STP](../03-ethernet-and-switching/09-spanning-tree.md) blocks links to stop loops, so you pay for links you cannot use.
- **Large MAC tables.** In a stretched Layer 2 network, every switch may learn every host. In VXLAN, the spine switches learn no host addresses at all.

EVPN solves the next problem: how the switches find each other's hosts without flooding.

Interviewers ask about this for data centre, cloud, and network engineer roles. They want to know if you can explain the following:

- Why the overhead is 50 bytes, and what breaks when you forget it.
- Why the UDP source port changes per flow.
- What each EVPN route type does.
- How a packet moves from one host to another, step by step.

## Simple Explanation

Think of a company with offices in many cities. Each office has a mailroom.

An employee in Lahore writes a note to a colleague in Dubai, using only an internal name: "To Sara, Finance." The public courier does not understand internal names.

So the Lahore mailroom puts the note inside a courier envelope. On the outside it writes "To: Dubai office mailroom." It also writes a code, "Dept 10010", so Dubai knows which department it is for.

The courier reads only the outside. It knows nothing about Sara. In Dubai, the mailroom opens the envelope, sees the department code, and hands the note to Sara. The note was never changed.

Map this to the network:

| Mail story | Network term | Meaning |
|---|---|---|
| Internal note | Inner frame | The original Ethernet frame from the host |
| Courier envelope | Outer IP and UDP headers | The wrapper the fabric reads |
| Courier company | Underlay | The physical routed network |
| Private internal mail system | Overlay | The virtual network the hosts see |
| Mailroom | VTEP | The device that wraps and unwraps |
| Department code | VNI | The number that says which virtual network |

How does the Lahore mailroom know that Sara sits in Dubai? There are two ways.

- **Flood-and-learn.** The mailroom sends a copy of the note to every office and asks "Is Sara there?" When Sara replies, it reads the return address and remembers it. This works, but it is noisy.
- **BGP EVPN.** Every mailroom keeps a shared company directory. When Sara joins Dubai, the Dubai mailroom publishes the update, and everyone knows at once. This is the control plane.

One more idea: the **anycast gateway**. Every office has a reception desk with the same internal phone number, 100. Wherever you sit, you dial 100 and reach the desk in your own building. In the network, every leaf switch is the default gateway, with the same IP and the same MAC.

**Where the analogy stops being true.** A courier tracks parcels and can retry a lost one. VXLAN does not. It runs over UDP, so a lost packet stays lost, and only the host's own protocol (such as TCP) can resend. Also, a mailroom can read the note. A VTEP can read the frame too, because VXLAN has no encryption.

## How It Works

### Overlay and underlay

The **underlay** is the real physical network: switches, cables, and a routing protocol. Its only job is to move IP packets between switches.

The **overlay** is a virtual network built on top of the underlay. It uses **tunnels**. A tunnel is a path where one packet travels inside another packet, so the devices in the middle see only the outer packet.

```text
OVERLAY (what the hosts see)

  Host A ======= one virtual Layer 2 segment ======= Host B
  10.10.10.10          VNI 10010                10.10.10.20
      |                                               |
------+-----------------------------------------------+------
      |                                               |
UNDERLAY (what the fabric sees)

    Leaf1 ------------- Spine1 ------------------- Leaf2
    10.0.1.11     routes the outer IP only      10.0.1.12
```

The underlay needs only three things:

1. **IP reachability** between all switch loopback addresses. A **loopback** is a virtual interface that stays up while the switch is up.
2. **Equal-cost paths**, called **ECMP** (Equal-Cost Multi-Path). This means traffic can spread over several links of the same cost.
3. **A large enough MTU.** The **MTU** (Maximum Transmission Unit) is the largest packet a link will carry.

Common underlay routing protocols are [OSPF](../05-routing/06-ospf.md), [IS-IS](../05-routing/08-is-is.md), and eBGP. The underlay knows nothing about hosts, tenants, or VLANs. A **tenant** is a customer or group whose traffic must stay separate from other groups.

### The 4094 VLAN limit

The [802.1Q tag](../03-ethernet-and-switching/07-trunking-8021q.md) has a 12-bit VLAN ID.

- 12 bits give 2^12 = 4096 values.
- Values 0 and 4095 are reserved.
- That leaves **4094 usable VLANs**.

For one company this is often enough. For a cloud provider with thousands of tenants, each wanting several networks, it is not.

**Fuller truth:** vendors reserve some more. For example, Cisco has historically reserved VLANs 1002-1005, and some platforms reserve others for internal use. QinQ (stacking two tags) also gives more IDs, but it keeps all the Layer 2 problems such as Spanning Tree.

### VNI: 24 bits

The **VNI** (VXLAN Network Identifier) names one virtual network. It does the job the VLAN ID did, but it is 24 bits long.

- 2^24 = 16,777,216 values, so about **16 million** networks.
- That is 4096 times more than the VLAN ID space.

A VNI is global across the fabric. A VLAN now has only local meaning on one switch. For example:

| Switch | Local VLAN | VNI |
|---|---|---|
| Leaf1 | 10 | 10010 |
| Leaf2 | 10 | 10010 |
| Leaf3 | 300 | 10010 |

All three ports are in the same virtual network, even though Leaf3 calls it VLAN 300.

**Fuller truth:** the header allows 16 million VNIs, but one switch cannot hold that many. Hardware has limits, often a few thousand VNIs per switch. Check the vendor data sheet.

### VTEP

A **VTEP** (VXLAN Tunnel End Point) is the device that adds and removes the VXLAN wrapper.

A VTEP has two sides:

- A **local side**, facing hosts. It sees normal Ethernet frames.
- A **fabric side**, facing the underlay. It sees IP packets.

```text
         local side                 fabric side
Host ---> [ VLAN 10 ] ---> [ VTEP encap ] ---> IP network
Host <--- [ VLAN 10 ] <--- [ VTEP decap ] <--- IP network
```

**Encapsulation** means adding the wrapper. **Decapsulation** means removing it. Short forms are encap and decap.

A VTEP can be:

- A **hardware VTEP**: a leaf switch. This is common in physical fabrics.
- A **software VTEP**: a hypervisor, a Linux host, or a virtual switch such as Open vSwitch.

The VTEP has an IP address, usually on a loopback interface. A loopback does not go down when one physical link fails, so the address stays reachable by another path.

Spine switches are usually **not** VTEPs. They just route the outer IP packet.

### The VXLAN packet

[RFC 7348](https://www.rfc-editor.org/rfc/rfc7348) defines VXLAN. It is an Informational RFC. It wraps the whole original frame like this:

```text
+----------+----------+---------+---------+---------------------+
| Outer    | Outer    | Outer   | VXLAN   | Original frame      |
| Ethernet | IPv4     | UDP     | header  | Ethernet + IP +     |
| 14 bytes | 20 bytes | 8 bytes | 8 bytes | payload (unchanged) |
+----------+----------+---------+---------+---------------------+
|<---------- 50 bytes added ------------->|
```

The VXLAN header is 8 bytes:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|R|R|R|R|I|R|R|R|                Reserved (24 bits)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            VNI (24 bits)                      | Reserved (8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **I flag:** must be 1. It means "the VNI field is valid." This makes the first byte 0x08.
- **VNI:** 24 bits.
- **Reserved bits:** sent as zero and ignored on receipt.

Example: VNI 10010 is 0x271A in hexadecimal. The 8 header bytes are:

```text
08 00 00 00 00 27 1A 00
^^          ^^^^^^^^
flags       VNI = 0x00271A = 10010
```

The outer UDP header has these values:

| Field | Value | Why |
|---|---|---|
| Destination port | **4789** | The IANA-assigned VXLAN port |
| Source port | A hash of the inner headers | Gives ECMP something to spread on |
| Checksum | Usually 0 for IPv4 | RFC 7348 says it should be sent as zero |

**Why the source port changes.** A **hash** is a math function that turns input data into a number. The same input always gives the same number.

Two VTEPs talk with the same outer IP addresses all day. If the outer headers never changed, every flow would hash to the same ECMP link. One link would be overloaded while others sit idle.

So the VTEP hashes the inner headers and puts the result in the outer UDP source port. RFC 7348 recommends the range 49152-65535.

- Different inner flows get different source ports, so they spread across links.
- One flow keeps the same source port, so its packets stay in order on one path.

**Why UDP and not TCP?** Running TCP inside TCP causes bad behaviour, because two layers would both try to retransmit and control congestion. UDP also needs no connection setup. See [UDP in Depth](../06-transport-layer/07-udp.md).

**History note:** early Linux VXLAN used UDP port 8472, before IANA assigned 4789. The Linux kernel default is still 8472 if you do not set a port, and tools such as Flannel keep that default. If two VTEPs use different ports, they cannot talk.

### Encapsulation overhead: the 50 bytes

The wrapper adds:

| Piece | Bytes |
|---|---|
| Outer Ethernet header (no VLAN tag) | 14 |
| Outer IPv4 header (no options) | 20 |
| Outer UDP header | 8 |
| VXLAN header | 8 |
| **Total** | **50** |

Now follow a full-size packet from a host with a normal 1500-byte MTU:

| Step | Size in bytes |
|---|---|
| Inner IP packet | 1500 |
| Add inner Ethernet header | 1514 |
| Add VXLAN header | 1522 |
| Add outer UDP | 1530 |
| Add outer IPv4: this is the **outer IP packet** | **1550** |
| Add outer Ethernet: this is the **outer frame** | **1564** |

The frame also has a 4-byte **FCS** (Frame Check Sequence), a checksum at the end of the frame. MTU settings usually do not count it.

So every underlay link needs an **IP MTU of at least 1550**. Notice why the number is still 50: the outer IP packet is 50 bytes larger than the inner IP packet, because the inner Ethernet header (14) is now payload and replaces the outer one in the count.

The overhead grows in some cases:

| Case | Overhead |
|---|---|
| IPv4 underlay, no outer VLAN tag | 50 |
| IPv4 underlay, outer 802.1Q tag | 54 |
| IPv6 underlay, no outer VLAN tag | 70 |
| Inner 802.1Q tag kept in the frame | Add 4 more |

RFC 7348 says a VTEP must not fragment VXLAN packets. A router in the middle might fragment them, but the far VTEP may silently drop the fragments. So the safe rule is simple: make the underlay MTU large enough.

Common practice is to set fabric links to a jumbo MTU such as 9216 on Cisco and Arista switches. Then a host with a 9000-byte MTU also fits: 9000 + 50 = 9050.

**Fuller truth:** vendors count MTU in different ways. Cisco NX-OS `mtu 9216` includes the Ethernet header. Linux MTU excludes it. Read the docs before you compare numbers.

### Finding the far VTEP: two methods

A VTEP needs one table: "This MAC in this VNI lives behind that VTEP IP."

The **data plane** is the part that forwards packets. The **control plane** is the part that builds the tables. VXLAN itself is only a data plane. It needs a way to fill the table.

Also, you need a way to send **BUM** traffic. BUM stands for **B**roadcast, **U**nknown unicast, and **M**ulticast. All three must reach every member of the VNI.

### Method 1: Flood-and-learn

This is the original method in RFC 7348. There is no control plane. The VTEP learns from packets, as a normal switch does (see [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md)).

Each VNI maps to an underlay multicast group. For example, VNI 10010 uses group 239.1.1.10. All VTEPs with that VNI join the group. See [Multicast Routing](../05-routing/14-multicast-routing.md).

1. Host A sends an ARP broadcast for Host B.
2. Leaf1 wraps it and sends it to group 239.1.1.10. The outer source IP is 10.0.1.11.
3. Leaf2 and Leaf3 receive it. They unwrap it and record: "Host A's MAC is behind 10.0.1.11, VNI 10010."
4. They flood the frame out their local ports.
5. Host B replies with a unicast frame to Host A.
6. Leaf2 already knows Host A, so it sends the reply straight to 10.0.1.11.
7. Leaf1 unwraps it and records: "Host B's MAC is behind 10.0.1.12."

```text
Host A           Leaf1          Multicast group        Leaf2 Host B
  |  ARP broadcast  |                  |                 |     |
  |---------------->|  wrap, send      |                 |     |
  |                 |----------------->|  copy to Leaf2  |     |
  |                 |                  |---------------->|     |
  |                 |                  |  learn A=Leaf1  |     |
  |                 |                  |                 |---->|
  |                 |      unicast ARP reply             |<----|
  |                 |<-----------------------------------|     |
  |<----------------| learn B=Leaf2                      |     |
```

Problems with this method:

- It needs multicast in the underlay, and many teams do not want to run it.
- Every host's first conversation is flooded, so wide networks stay noisy.
- Any device that sends packets gets learned. There is no check.
- It says nothing about routing between subnets.

Some networks avoid multicast with **ingress replication** (also called head-end replication). The first VTEP makes one unicast copy for each other VTEP, using a configured list. This needs no multicast, but the list is manual, and the learning still comes from flooding.

### Method 2: BGP EVPN

**EVPN** (Ethernet VPN) is a BGP-based control plane. It was first defined for MPLS in [RFC 7432](https://www.rfc-editor.org/rfc/rfc7432). [RFC 8365](https://www.rfc-editor.org/rfc/rfc8365) applies it to overlays such as VXLAN.

The key idea: learn hosts with BGP, not with flooded packets.

1. Host B sends a frame. Leaf2 learns its MAC on the local port, as any switch would.
2. Leaf2 puts the MAC into a BGP update. If it also sees the IP (from ARP), it includes that.
3. A **route reflector** copies the update to all other leaves. A route reflector is a BGP router that passes routes between iBGP peers, so they do not all need to peer with each other. In a fabric, spines usually do this.
4. Each leaf checks whether it wants the route. If yes, it writes the MAC into its table: "Host B, VNI 10010, behind 10.0.1.12."

```text
                  +----------------+
                  | Spine1         |
                  | route reflector|
                  +----------------+
                    ^            |
   BGP update: MAC  |            |  BGP update reflected
   of Host B is     |            |  to all other leaves
   behind 10.0.1.12 |            v
              +--------+    +--------+
              | Leaf2  |    | Leaf1  |  installs MAC in table
              +--------+    +--------+
                  |
               Host B
```

Now all VTEPs know all hosts **before** the first data packet is sent.

**Fuller truth:** learning on the host-facing port is still done from packets. EVPN replaces the learning **between** VTEPs, not on the local access port.

| | Flood-and-learn | BGP EVPN |
|---|---|---|
| Where is learning done? | Data plane, from packets | Control plane, by BGP |
| How are VTEPs found? | Multicast group or manual list | Automatic (Type 3 routes) |
| Underlay multicast | Usually needed | Optional |
| ARP flooding | Full | Reduced by ARP suppression |
| Routing between subnets | Not defined | Built in |
| Anycast gateway | Not part of it | Standard design |
| Current status (2026) | Legacy, simple cases | Normal choice for new fabrics |

### What BGP carries for EVPN

EVPN is a new "address family" in **MP-BGP** (Multiprotocol BGP). MP-BGP lets BGP carry more than IPv4 routes. EVPN uses AFI 25 (L2VPN) and SAFI 70 (EVPN). AFI and SAFI are two numbers that say what kind of route is in the update.

Two terms come from [MPLS](../05-routing/13-mpls.md) VPNs:

- **RD (Route Distinguisher):** an 8-byte value added to a route to make it unique. Two tenants can use the same address, and the RD keeps their routes apart in BGP. It says nothing about who should import the route.
- **RT (Route Target):** a tag, carried as a BGP extended community, that controls who imports a route. A VTEP exports routes with an RT. Other VTEPs import only the RTs they are configured for.

Simple rule: the **RD makes routes unique, the RT decides who takes them.**

Example with NX-OS auto values, in AS 65001:

- RD on Leaf2 for VLAN 10: `10.0.0.12:32777` (BGP router ID, then 32767 + VLAN ID).
- RT for VNI 10010: `65001:10010` (AS number, then VNI).

EVPN routes also carry these extended communities:

| Community | Meaning |
|---|---|
| Route Target | Who imports this route |
| Encapsulation | Tunnel type. `ENCAP:8` means VXLAN |
| Router MAC | The MAC of the sending VTEP, used for routed traffic |
| MAC Mobility | Sequence number, used when a host moves |

### EVPN route types

An EVPN update has a **route type** number. These are the ones to know:

| Type | Name | Who sends it and why | Defined in |
|---|---|---|---|
| 1 | Ethernet Auto-Discovery | Multi-homing: fast failover and load sharing | RFC 7432 |
| 2 | MAC/IP Advertisement | "This MAC, and maybe this IP, is behind me" | RFC 7432 |
| 3 | Inclusive Multicast Ethernet Tag | "I am a member of this VNI. Send BUM to me" | RFC 7432 |
| 4 | Ethernet Segment | Multi-homing: find peers, elect a forwarder | RFC 7432 |
| 5 | IP Prefix | "This whole subnet is reachable through me" | RFC 9136 |

Types 2, 3, and 5 are what a normal single-homed fabric uses. Types 1 and 4 appear when a host connects to two leaves using EVPN multi-homing.

Later RFCs add more. Types 6, 7, and 8 handle IGMP and MLD (RFC 9251). Types 9, 10, and 11 handle multicast tree setup (RFC 9572). Interviews usually focus on 1 to 5.

**Type 2 in detail.** It can carry:

- RD
- ESI (Ethernet Segment Identifier), zero for a single-homed host
- Ethernet Tag ID, usually 0
- MAC address
- IP address (optional)
- Label 1: the L2 VNI
- Label 2: the L3 VNI (optional)

The field is called a "label" because EVPN was built for MPLS. RFC 8365 reuses this 24-bit field to carry the VNI.

A leaf often sends two Type 2 routes for one host: a MAC-only route and a MAC+IP route.

**Type 3 in detail.** Each VTEP sends one for each L2 VNI it has. It says: "I am 10.0.1.12, I have VNI 10010." From these, every VTEP builds its flood list automatically. It also says which BUM method to use, in a PMSI Tunnel attribute. This is usually ingress replication or a multicast group.

**Type 5 in detail.** It carries a prefix such as 10.10.30.0/24, not a host. It is used for routes learned from outside the fabric, and for subnets where a host has not spoken yet.

### ARP suppression

[ARP](../03-ethernet-and-switching/05-arp.md) requests are broadcast, so in a big network they are a large share of BUM traffic.

With EVPN, Type 2 routes give each leaf a table of IP to MAC. When a local host asks "Who has 10.10.10.20?", the leaf replies itself. The broadcast never enters the fabric.

If the leaf has no entry, it floods the request as normal. So ARP suppression reduces flooding but does not remove it.

### L2 VNI, L3 VNI, and routing between subnets

A **VRF** (Virtual Routing and Forwarding instance) is a separate routing table on one device. Each tenant usually gets its own VRF.

- **L2 VNI:** one per Layer 2 segment. It is for bridging inside a subnet. Example: VNI 10010 for VLAN 10.
- **L3 VNI:** one per VRF. It is for routing between subnets. Example: VNI 50000 for VRF TENANT-A. No host sits in it. It is a transit network between VTEPs.

**IRB** (Integrated Routing and Bridging) means a switch does both bridging and routing on one device. An **SVI** (Switched Virtual Interface) is a virtual Layer 3 interface for a VLAN, such as `interface Vlan10`.

There are two IRB models, symmetric and asymmetric. Most fabrics use symmetric. See [Going Deeper](#going-deeper).

### Anycast gateway

The old way to give a subnet a gateway was a pair of routers using [HSRP or VRRP](../05-routing/11-first-hop-redundancy.md). In a fabric that design has two problems:

- Traffic between subnets must travel to the gateway pair, then back. This is called **hairpinning**. Two hosts on the same leaf may send traffic across the whole fabric just to be routed.
- If a VM moves, it may be far from its gateway.

The **distributed anycast gateway** fixes this. Every leaf with VLAN 10 has an SVI with the **same IP** and the **same MAC**.

```text
  Leaf1                Leaf2                Leaf3
  Vlan10               Vlan10               Vlan10
  10.10.10.1           10.10.10.1           10.10.10.1
  0000.2222.3333       0000.2222.3333       0000.2222.3333
     |                    |                    |
  Host A               Host B               VM after move
  gateway:             gateway:             gateway:
  10.10.10.1           10.10.10.1           10.10.10.1
```

Results:

- A host always reaches a gateway one hop away, on its own leaf.
- A moved VM keeps working. Its ARP cache entry for the gateway is still correct, because the MAC is the same everywhere.
- All leaves are active. There is no active and standby.

Why is the duplicate address safe? Each leaf answers only local hosts. The gateway address is not reachable across the overlay as a normal host. So no host ever sees two answers.

It differs from HSRP in this way: HSRP has one active router out of a group. An anycast gateway has every leaf active at the same time, with no hello protocol between them.

## Example

### Topology

Three leaves and two spines, one tenant, two subnets. The overlay uses iBGP in AS 65001, with both spines as route reflectors. The underlay uses OSPF. BUM uses ingress replication.

```text
        +----------+                  +----------+
        |  Spine1  |                  |  Spine2  |
        | 10.0.0.1 |                  | 10.0.0.2 |
        +----------+                  +----------+
              |  every leaf links to both spines |
    +---------+----------+-----------------------+
    |                    |                       |
+---+------+       +-----+----+            +-----+----+
| Leaf1    |       | Leaf2    |            | Leaf3    |
| Lo0      |       | Lo0      |            | Lo0      |
| 10.0.0.11|       | 10.0.0.12|            | 10.0.0.13|
| VTEP Lo1 |       | VTEP Lo1 |            | VTEP Lo1 |
| 10.0.1.11|       | 10.0.1.12|            | 10.0.1.13|
+---+------+       +-----+----+            +-----+----+
    | Eth1/10            | Eth1/10               | Eth1/10
 Host A               Host B                  Host C
 VLAN 10              VLAN 10                 VLAN 20
 10.10.10.10          10.10.10.20             10.10.20.30
```

| Item | Value |
|---|---|
| VLAN 10 | VNI 10010, subnet 10.10.10.0/24, gateway 10.10.10.1 |
| VLAN 20 | VNI 10020, subnet 10.10.20.0/24, gateway 10.10.20.1 |
| VRF TENANT-A | L3 VNI 50000, using VLAN 999 |
| Anycast gateway MAC | 0000.2222.3333 |
| Host A MAC | 0200.0000.000a |
| Host B MAC | 0200.0000.000b |
| Host C MAC | 0200.0000.000c |
| Leaf1 router MAC | 0211.1111.1111 |
| Leaf2 router MAC | 0222.2222.2222 |
| Leaf3 router MAC | 0233.3333.3333 |

### Leaf1 configuration (Cisco NX-OS)

VXLAN EVPN syntax is different on each platform. This is NX-OS on a Nexus switch, not classic IOS.

```text
! Turn on the features
nv overlay evpn
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
feature fabric forwarding

! Same anycast gateway MAC on every leaf
fabric forwarding anycast-gateway-mac 0000.2222.3333

! Map VLANs to VNIs
vlan 10
  vn-segment 10010
vlan 20
  vn-segment 10020
vlan 999
  vn-segment 50000

! Tenant VRF and its L3 VNI
vrf context TENANT-A
  vni 50000
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

! Anycast gateway for VLAN 10
interface Vlan10
  no shutdown
  vrf member TENANT-A
  ip address 10.10.10.1/24
  fabric forwarding mode anycast-gateway

! Interface for the L3 VNI: no IP address needed
interface Vlan999
  no shutdown
  vrf member TENANT-A
  ip forward

! Underlay link: note the jumbo MTU
interface Ethernet1/1
  description to Spine1
  no switchport
  mtu 9216
  ip address 10.1.1.1/31
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  no shutdown

! Host port
interface Ethernet1/10
  switchport access vlan 10

! Loopbacks: BGP peering and VTEP source
interface loopback0
  ip address 10.0.0.11/32
  ip router ospf UNDERLAY area 0.0.0.0
interface loopback1
  ip address 10.0.1.11/32
  ip router ospf UNDERLAY area 0.0.0.0

! The VTEP itself
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    suppress-arp
    ingress-replication protocol bgp
  member vni 10020
    suppress-arp
    ingress-replication protocol bgp
  member vni 50000 associate-vrf

! EVPN instance for each L2 VNI
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 10020 l2
    rd auto
    route-target import auto
    route-target export auto

! BGP overlay to both spines
router ospf UNDERLAY
  router-id 10.0.0.11
router bgp 65001
  router-id 10.0.0.11
  neighbor 10.0.0.1
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.0.2
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf TENANT-A
    address-family ipv4 unicast
      advertise l2vpn evpn
```

The `send-community extended` lines are essential. Route Targets are extended communities. Without them, routes arrive with no RT and nobody imports them.

On some platforms, `suppress-arp` also needs hardware table space to be set aside first. Check your model.

### Verifying the control plane (Cisco NX-OS)

Check that the VTEPs see each other:

```text
Leaf1# show nve peers
Interface Peer-IP          State LearnType Uptime   Router-Mac
--------- ---------------  ----- --------- -------- -----------------
nve1      10.0.1.12        Up    CP        02:14:33 0222.2222.2222
nve1      10.0.1.13        Up    CP        02:14:30 0233.3333.3333
```

`CP` means learned by the control plane. `DP` would mean flood-and-learn.

Check the VNIs:

```text
Leaf1# show nve vni
Codes: CP - Control Plane        DP - Data Plane
       UC - Unconfigured         SA - Suppress ARP

Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10010    UnicastBGP        Up    CP   L2 [10]            SA
nve1      10020    UnicastBGP        Up    CP   L2 [20]            SA
nve1      50000    n/a               Up    CP   L3 [TENANT-A]
```

Look at the EVPN routes for Host B, received from Leaf2 (output trimmed):

```text
Leaf1# show bgp l2vpn evpn
   Network            Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.0.0.12:32777
*>i[2]:[0]:[0]:[48]:[0200.0000.000b]:[0]:[0.0.0.0]/216
                      10.0.1.12                     100      0 i
*>i[2]:[0]:[0]:[48]:[0200.0000.000b]:[32]:[10.10.10.20]/272
                      10.0.1.12                     100      0 i
*>i[3]:[0]:[32]:[10.0.1.12]/88
                      10.0.1.12                     100      0 i
```

Read the second line piece by piece:

| Piece | Meaning |
|---|---|
| `[2]` | Route type 2 |
| `[0]` | ESI 0: single-homed host |
| `[0]` | Ethernet Tag 0 |
| `[48]` | MAC length, in bits |
| `[0200.0000.000b]` | Host B's MAC |
| `[32]` | IP length, in bits |
| `[10.10.10.20]` | Host B's IP |

The `[3]` line says: "Leaf2 is a member of this VNI." This is how Leaf1 added 10.0.1.12 to its flood list.

Now see the attributes of the MAC+IP route (header lines trimmed):

```text
Leaf1# show bgp l2vpn evpn 10.10.10.20
  AS-Path: NONE, path sourced internal to AS
    10.0.1.12 (metric 41) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010 50000
      Extcommunity: RT:65001:10010 RT:65001:50000 ENCAP:8
          Router MAC:0222.2222.2222
```

This one output shows nearly everything:

- Next hop 10.0.1.12: the far VTEP.
- Label 10010: the L2 VNI. Label 50000: the L3 VNI.
- `ENCAP:8`: use VXLAN.
- Router MAC: use this inner MAC when routing to Leaf2.

Check the MAC table:

```text
Leaf1# show mac address-table vlan 10
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+----------
*   10     0200.0000.000a   dynamic  0         F      F    Eth1/10
C   10     0200.0000.000b   dynamic  0         F      F    nve1(10.0.1.12)
```

The first entry is a local port. The second entry points to a tunnel. `C` means the entry came from the control plane.

### Packet walk 1: same subnet (Host A to Host B)

Host A pings 10.10.10.20. Both hosts are in VLAN 10, on different leaves.

```text
Host A      Leaf1 (VTEP)      Spine1      Leaf2 (VTEP)     Host B
  |  frame       |               |              |             |
  |------------->|  encap        |              |             |
  |              |-------------->|  route       |             |
  |              |               |------------->|  decap      |
  |              |               |              |------------>|
```

1. **ARP.** Host A asks "Who has 10.10.10.20?" Leaf1 already has the answer from a Type 2 route. With ARP suppression, it replies itself: `0200.0000.000b`. Nothing enters the fabric.
2. **Frame leaves the host.** Source MAC `0200.0000.000a`, destination MAC `0200.0000.000b`. It reaches Leaf1 on Eth1/10, in VLAN 10.
3. **Lookup on Leaf1.** The MAC table says: VLAN 10, this MAC, go to `nve1(10.0.1.12)`.
4. **Encapsulation.** Leaf1 keeps the original frame unchanged and adds:
   - VXLAN header: VNI 10010.
   - UDP: source port 52100 (from a hash), destination port 4789.
   - IP: source 10.0.1.11, destination 10.0.1.12.
   - Ethernet: its uplink MAC to Spine1's MAC.
5. **Fabric.** Leaf1 picks a spine by ECMP. Spine1 does a normal IP route lookup on 10.0.1.12. It rewrites the outer MACs and lowers the outer TTL. It never looks at the VNI or the inner frame.
6. **Decapsulation on Leaf2.** The outer destination IP is its own VTEP address, and the UDP port is 4789, so it removes the wrapper. VNI 10010 maps to local VLAN 10.
7. **Delivery.** Leaf2 looks up the inner destination MAC, finds Eth1/10, and sends the original frame to Host B.

The reply takes the same steps in reverse, possibly with a different source port and through a different spine.

Which fields change at each hop:

| Link | Outer src MAC | Outer dst MAC | Outer IPs | Inner frame |
|---|---|---|---|---|
| Host A to Leaf1 | (none) | (none) | (none) | Original |
| Leaf1 to Spine1 | Leaf1 uplink | Spine1 port | 10.0.1.11 to 10.0.1.12 | Same |
| Spine1 to Leaf2 | Spine1 port | Leaf2 uplink | 10.0.1.11 to 10.0.1.12 | Same |
| Leaf2 to Host B | (none) | (none) | (none) | Same |

A capture on a Linux box receiving a mirror of the Leaf1 uplink shows both layers:

```bash
# Linux
sudo tcpdump -nt -i eth1 -c 1 'udp port 4789'
```

```text
IP 10.0.1.11.52100 > 10.0.1.12.4789: VXLAN, flags [I] (0x08), vni 10010
IP 10.10.10.10 > 10.10.10.20: ICMP echo request, id 1, seq 1, length 64
```

Check the sizes. ICMP is 64 bytes.

| Layer | Size in bytes |
|---|---|
| ICMP | 64 |
| Inner IP packet (+20) | 84 |
| Inner frame (+14) | 98 |
| VXLAN (+8) | 106 |
| UDP (+8) | 114 |
| Outer IP packet (+20) | 134 |
| Outer frame (+14) | 148 |

148 - 98 = 50. The overhead matches.

In Wireshark, the filter `vxlan.vni == 10010` shows only this VNI. Wireshark decodes UDP 4789 as VXLAN by itself.

### Packet walk 2: different subnets (Host A to Host C)

Host A (10.10.10.10, VLAN 10, Leaf1) pings Host C (10.10.20.30, VLAN 20, Leaf3). This uses symmetric IRB. Leaf1 does not need VLAN 20 at all for this to work.

Leaf1's VRF table already has a host route from Leaf3's Type 2 route:

```text
Leaf1# show ip route vrf TENANT-A 10.10.20.30
10.10.20.30/32, ubest/mbest: 1/0
    *via 10.0.1.13%default, [200/0], 01:12:08, bgp-65001, internal,
         tag 65001 (evpn) segid: 50000 tunnelid: 0xa00010d encap: VXLAN
```

1. **Host A sees a different subnet.** It sends to its gateway. Destination MAC is `0000.2222.3333`, the anycast gateway MAC. Destination IP is 10.10.20.30. TTL is 64.
2. **Leaf1 routes.** The frame is to its own gateway MAC, so it does a lookup in VRF TENANT-A. The route says: VTEP 10.0.1.13, VNI 50000. TTL becomes 63.
3. **Leaf1 builds a new inner frame.** Source MAC is its own router MAC `0211.1111.1111`. Destination MAC is Leaf3's router MAC `0233.3333.3333`, learned from the Router MAC community.
4. **Encapsulation.** VNI **50000** (the L3 VNI, not 10010 or 10020). Outer IP 10.0.1.11 to 10.0.1.13.
5. **Fabric.** The spines route on the outer IP.
6. **Leaf3 decapsulates.** VNI 50000 maps to VRF TENANT-A. The inner destination MAC is its own router MAC, so it routes.
7. **Leaf3 routes.** 10.10.20.30 is on local VLAN 20. TTL becomes 62.
8. **Delivery.** Leaf3 builds a new frame: source MAC `0000.2222.3333`, destination MAC `0200.0000.000c`, and sends it out Eth1/10.

Traffic goes in and out on the same VNI, 50000. That is why it is called symmetric.

### Small Linux lab

Two Linux machines can build a VXLAN tunnel with static settings. This is flood-and-learn at its smallest. The machines have underlay addresses 192.168.56.11 and 192.168.56.12 on eth0, with MTU 1500. The firewall must allow UDP 4789.

```bash
# Linux, machine 1
sudo ip link add vxlan100 type vxlan id 100 dstport 4789 \
  local 192.168.56.11 remote 192.168.56.12 dev eth0
sudo ip addr add 10.200.0.1/24 dev vxlan100
sudo ip link set vxlan100 up

# Linux, machine 2
sudo ip link add vxlan100 type vxlan id 100 dstport 4789 \
  local 192.168.56.12 remote 192.168.56.11 dev eth0
sudo ip addr add 10.200.0.2/24 dev vxlan100
sudo ip link set vxlan100 up
```

Test from machine 1. Linux lowers the tunnel MTU by 50 for you:

```bash
ping -c 1 10.200.0.2
cat /sys/class/net/vxlan100/mtu
bridge fdb show dev vxlan100
```

```text
1 packets transmitted, 1 received, 0% packet loss, time 0ms
1450
6a:1f:3c:88:02:22 dst 192.168.56.12 self
00:00:00:00:00:00 dst 192.168.56.12 self permanent
```

The all-zero entry means "send unknown traffic here." The other entry was learned from a real packet. This is flood-and-learn in two lines.

Windows has no simple built-in command to create a tunnel like this. Windows Server uses VXLAN through its Software Defined Networking stack. But you can capture it, if Wireshark is installed:

```powershell
# Windows, with Wireshark installed
& "C:\Program Files\Wireshark\tshark.exe" -i Ethernet -f "udp port 4789" -Y "vxlan.vni == 100"
```

### Testing the underlay MTU

To test that the underlay carries a 1550-byte IP packet, send 1522 bytes of ICMP data with fragmentation blocked. 1522 + 8 (ICMP) + 20 (IP) = 1550.

```bash
# Linux
ping -M do -s 1522 -c 3 192.168.56.12
```

```powershell
# Windows
ping -f -l 1522 -n 3 192.168.56.12
```

If the path MTU is only 1500, Linux reports `message too long`, and Windows reports `Packet needs to be fragmented but DF set.`

On a Nexus fabric with jumbo links, test from loopback to loopback:

```text
Leaf1# ping 10.0.1.12 source-interface loopback1 df-bit packet-size 8972 count 3
3 packets transmitted, 3 packets received, 0.00% packet loss
```

## Going Deeper

### Symmetric vs asymmetric IRB

In **asymmetric IRB**, the ingress leaf (the first leaf) does routing and bridging, and the egress leaf (the last leaf) does bridging only.

Using the example above, Leaf1 routes from VLAN 10 into VLAN 20, then sends the frame on VNI 10020 with Host C's real MAC as the inner destination. The reply comes back on VNI 10010. So the two directions use different VNIs.

| | Asymmetric IRB | Symmetric IRB |
|---|---|---|
| Ingress leaf | Routes and bridges | Routes |
| Egress leaf | Bridges only | Routes |
| VNI used | Destination subnet's L2 VNI | One L3 VNI per VRF |
| VNIs per leaf | Every leaf needs every VNI | Only VNIs with local hosts |
| Host state | Each leaf needs ARP and MAC for all | Host routes, far less L2 state |
| Scale | Poorer | Better |
| Inner destination MAC | Real host MAC | Egress leaf's router MAC |

Both are described in [RFC 9135](https://www.rfc-editor.org/rfc/rfc9135). Some platforms support only one model, and the two do not mix inside one VRF without care.

### Silent hosts and Type 5 routes

A **silent host** sends nothing until it is contacted, such as a printer or a passive server. No traffic means no MAC learning and so no Type 2 route.

Leaf1 then has no host route for it. The fix is for each leaf to advertise its subnet as a Type 5 route. Traffic for the silent host follows the subnet route to a leaf that owns that subnet. That leaf sends an ARP request, the host answers, and a Type 2 route follows.

Type 5 is also how external routes enter the fabric. A **border leaf** connects the fabric to the outside world, such as a firewall or WAN router. It learns external prefixes, usually over a VRF-aware BGP or OSPF session, and re-advertises them as Type 5 routes.

### Host mobility

When a VM moves from Leaf1 to Leaf3, what stops old routes from winning?

1. The VM (or its hypervisor) sends a gratuitous ARP or RARP after the move.
2. Leaf3 learns the MAC locally, but sees a BGP route for the same MAC from Leaf1.
3. Leaf3 advertises the MAC with a **MAC Mobility** sequence number that is one higher.
4. All VTEPs prefer the higher sequence number.
5. Leaf1 sees it lost and withdraws its old route.

RFC 7432 also describes MAC duplication detection. If a MAC moves too often, the PE (Provider Edge device, here the VTEP) treats this as a duplicate, not as a real move. The RFC default is 5 moves in 180 seconds. Vendors let you change this. Two hosts with the same MAC, or a loop, can trigger it.

### Multi-homing

A server often connects to two leaves for redundancy. There are two main approaches.

**vPC or MLAG style.** Cisco vPC and vendor MLAG make two switches look like one to the server (see [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md)). Both switches share one VTEP IP, often called an anycast VTEP. On Nexus, this is a secondary IP on the VTEP loopback. Remote leaves see one VTEP, and ECMP handles the rest. It needs a peer link and vendor-specific protocols.

**EVPN multi-homing (ESI).** No peer link is needed. The link bundle gets a 10-byte **ESI** (Ethernet Segment Identifier), the same on both leaves. Types 1 and 4 then do the work:

| Problem | Solution | Route type |
|---|---|---|
| Which leaves share this segment? | Discovery | Type 4 |
| Who floods BUM to the host, so it gets one copy? | Designated Forwarder (DF) election | Type 4 |
| How to share load when only one leaf learned the MAC? | Aliasing | Type 1 per EVI |
| How to fail over fast when a link breaks? | Mass withdrawal | Type 1 per segment |
| How to stop BUM looping back to the same segment? | Split horizon | RFC 8365 local bias |

The **DF** is the one leaf chosen to send BUM to the shared host. The default method in RFC 7432 is a simple modulo on the VLAN number, and RFC 8584 adds better options. **Mass withdrawal** is one route withdrawal that removes the path for thousands of MACs at once, rather than thousands of Type 2 withdrawals. An **EVI** is an EVPN Instance, usually one per L2 VNI.

### BUM: multicast vs ingress replication

| | Underlay multicast | Ingress replication |
|---|---|---|
| Who makes copies | The fabric | The first VTEP |
| Copies sent by first VTEP | 1 | N - 1 |
| Underlay needs | PIM, rendezvous points | Nothing extra |
| Operations | More work | Simple |

If a VNI spans 40 leaves, ingress replication sends 39 copies of each BUM frame. With ARP suppression, this is often acceptable, and many fabrics now choose it for simplicity. Very heavy BUM use still favours multicast.

### iBGP or eBGP for the overlay

- **iBGP overlay.** One AS across the fabric, with spines as route reflectors. RTs auto-generated from `AS:VNI` match everywhere. The underlay is commonly OSPF or IS-IS.
- **eBGP overlay.** Each leaf (or leaf pair) has its own AS, and BGP may also be the underlay. This needs care:
  - Spines must keep the next hop unchanged, so it stays the far VTEP.
  - Spines hold no VNIs, so by default they may drop routes with no matching RT. NX-OS uses `retain route-target all` for this.
  - Auto-generated RTs differ per AS. Use manual RTs, or a vendor feature that rewrites them.

### Scaling beyond one fabric

Several fabrics can link through **border gateways**, as in Cisco's EVPN Multi-Site. Each site keeps its own flood domain, so a problem in one site stays there. Inter-site traffic is re-encapsulated at the border.

### Security

VXLAN has **no encryption and no authentication**. Anyone who can send UDP 4789 to a VTEP can try to inject frames into a tenant network.

- Filter UDP 4789 at the fabric edge, so outside devices cannot reach VTEP addresses.
- BGP sessions can use MD5 or TCP-AO authentication.
- To encrypt, use MACsec on the links or IPsec on the path. See [IPsec](../09-security/08-ipsec.md).
- A VNI gives traffic separation, like a VLAN. Firewalls are still needed between tenants. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### Related encapsulations

| Name | Transport | Note |
|---|---|---|
| VXLAN | UDP 4789 | Most common. Fixed 8-byte header |
| Geneve | UDP 6081 | RFC 8926. Variable options, so it is extensible |
| NVGRE | GRE | RFC 7637. Now rare |
| MPLS | Labels | EVPN's first transport |

Geneve is used by VMware NSX and by OVN. VXLAN also appears in container networks, such as [Docker overlay networks](../11-cloud-and-containers/07-docker-networking.md) and several [Kubernetes CNI plugins](../11-cloud-and-containers/08-kubernetes-networking.md). There, the VTEP is the Linux kernel on each node.

The reserved bits in the VXLAN header have also been used by vendor extensions, such as a group policy tag. These extensions are not part of RFC 7348.

### Arista EOS for comparison

The ideas are the same, and the words change:

```text
! Arista EOS
ip virtual-router mac-address 00:00:22:22:33:33

interface Vlan10
   vrf TENANT-A
   ip address virtual 10.10.10.1/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf TENANT-A vni 50000
```

### Troubleshooting order

Start at the bottom. Overlay problems are often underlay problems.

1. **Underlay reachability.** Ping VTEP loopback to VTEP loopback, using the loopback as source.
2. **MTU.** Ping with the DF bit and a large size. If small pings work and large transfers hang, suspect MTU first.
3. **BGP overlay.** Is the `l2vpn evpn` session up? Are prefixes received? (`show bgp l2vpn evpn summary`)
4. **VTEP state.** Is the NVE interface up? (`show nve peers`, `show nve vni`)
5. **Routes.** Is there a Type 2 for the host? Is it imported? If it is received but not imported, check the RT.
6. **Tables.** Is the MAC in the MAC table with an `nve1` next hop?
7. **Capture.** Filter on `udp port 4789`, and check the VNI matches on both ends. A VNI mismatch makes a VTEP drop traffic silently.

### Standards summary

| Document | Subject |
|---|---|
| RFC 7348 | VXLAN |
| RFC 7432 | BGP MPLS-based EVPN (with later updates) |
| RFC 8365 | EVPN for network virtualization overlays |
| RFC 8584 | EVPN Designated Forwarder election framework |
| RFC 9135 | Integrated Routing and Bridging in EVPN |
| RFC 9136 | IP Prefix Advertisement in EVPN (Type 5) |
| RFC 9161 | Proxy ARP and ND in EVPN |
| RFC 9251 | IGMP and MLD proxies for EVPN |

## Common Mistakes

- **"VXLAN and EVPN are the same thing."** VXLAN is the data plane: how packets are wrapped. EVPN is the control plane: how addresses are learned. VXLAN can run without EVPN, and EVPN can run over MPLS.
- **Forgetting the underlay MTU.** Small pings work and ARP works, but big transfers hang. The fix is on the underlay links, not on the hosts.
- **"The UDP source port is also 4789."** Only the destination is 4789. The source port is a hash value, and it is what makes ECMP work.
- **"VXLAN is encrypted, because it is a tunnel."** It is not. The inner frame is plain text in a capture.
- **"VXLAN is reliable."** It is UDP. Lost packets are not resent by VXLAN.
- **"Spines need the VNIs."** Spines route the outer IP. They need no VNIs and no host tables, even when they are route reflectors.
- **"16 million VNIs means 16 million on one switch."** The header allows it, but hardware does not.
- **"EVPN removes all flooding."** It reduces it. Unknown unicast and real broadcasts can still be flooded, and BUM handling is still needed.
- **"VLANs are gone."** VLANs are still used on host ports. They just have local meaning now.
- **Mixing up RD and RT.** The RD makes a route unique. The RT controls import. A wrong RT is a very common cause of "the route is there, but it is not used."
- **Omitting `send-community extended`.** With no extended communities, there are no RTs and no working overlay.
- **"An anycast gateway is just HSRP."** HSRP has one active gateway. An anycast gateway is active on every leaf.
- **"The L3 VNI holds hosts."** It holds no hosts. It is a transit VNI for routed traffic between VTEPs in one VRF.
- **Removing all loop protection.** The fabric core has no STP, but host-facing ports can still see a loop from a cable mistake. Keep edge protection such as BPDU Guard.
- **Mismatched UDP ports.** Linux may use 8472 by default, while switches use 4789. The two will not talk until you set the same port.

## Interview Questions

**Q: What problem does VXLAN solve?**

A: It lets a Layer 2 segment stretch over a routed Layer 3 network. So I can have a routed spine-leaf fabric with ECMP and no Spanning Tree, while hosts still share a subnet across racks. It also raises the segment limit from 4094 VLANs to about 16 million VNIs.

**Q: What is the difference between overlay and underlay?**

A: The underlay is the physical routed network that moves IP packets between VTEPs. The overlay is the virtual network built with tunnels on top of it. Spines in the underlay route only on the outer IP and know nothing about tenants.

**Q: Why is the VXLAN overhead 50 bytes? What MTU do you need?**

A: Outer Ethernet 14, outer IP 20, UDP 8, VXLAN 8, which is 50. A 1500-byte inner packet becomes a 1550-byte outer IP packet and a 1564-byte frame. So the underlay IP MTU must be at least 1550. In practice we use jumbo frames such as 9216 on fabric links. With an outer VLAN tag it is 54, and with an IPv6 underlay it is 70.

**Q: VXLAN uses destination port 4789. Why is the source port different for each flow?**

A: Two VTEPs use the same outer IPs for all traffic, so ECMP would see one flow and use one link. The VTEP hashes the inner headers into the UDP source port. Then different inner flows take different paths, while one flow stays on one path and keeps its order.

**Q: What is a VTEP, and where does it run?**

A: It is the tunnel endpoint that wraps and unwraps VXLAN packets. It usually uses a loopback address as its source. It can be a hardware leaf switch or software on a hypervisor or Linux host. Spines are normally not VTEPs.

**Q: Compare flood-and-learn with BGP EVPN.**

A: Flood-and-learn has no control plane. VTEPs flood BUM through multicast or ingress replication, and learn remote MACs from packets. BGP EVPN sends MAC and IP information in BGP updates, so VTEPs know hosts before traffic flows. EVPN also gives automatic VTEP discovery, ARP suppression, and routing between subnets, so it is the normal choice for new fabrics.

**Q: Explain EVPN route types 1 to 5.**

A: Type 1 is Ethernet Auto-Discovery, used in multi-homing for fast failover and load sharing. Type 2 advertises a MAC and optionally an IP. Type 3 says which VTEPs are in a VNI, which builds the BUM flood list. Type 4 is the Ethernet Segment route, used to find peers and elect the designated forwarder. Type 5 advertises an IP prefix, such as an external route.

**Q: What is the difference between a Route Distinguisher and a Route Target?**

A: The RD makes a route unique inside BGP, so overlapping addresses do not collide. The RT is an extended community that controls which VTEPs import the route. If a route is present but not used, I check the RT first.

**Q: What is a distributed anycast gateway, and why is it better than HSRP?**

A: Every leaf has the same gateway IP and MAC for a subnet, so each host's gateway is its own leaf. This avoids hairpinning traffic to a central pair, and all leaves are active. A moved VM keeps working, because its gateway ARP entry is still valid. HSRP has one active router and needs hellos between routers.

**Q: What is the difference between an L2 VNI and an L3 VNI?**

A: An L2 VNI maps to one Layer 2 segment and is used for bridging inside a subnet. An L3 VNI maps to a VRF and carries routed traffic between VTEPs in symmetric IRB. No host sits in an L3 VNI.

**Q: Walk me through a packet between two hosts in different subnets on different leaves, using symmetric IRB.**

A: The host sends to its anycast gateway MAC. The ingress leaf routes in the VRF and finds a host route from a Type 2 update, with the remote VTEP as next hop. It sets the inner destination MAC to the remote leaf's router MAC and encapsulates with the L3 VNI, on UDP 4789. Spines route on the outer IP. The egress leaf decapsulates, maps the L3 VNI to the VRF, routes again, and delivers to the host in its local VLAN.

**Q: Small pings work over a new VXLAN fabric, but file transfers hang. What do you check?**

A: I suspect MTU first. I test with a ping that has the DF bit set and a large size, loopback to loopback across the underlay. For a 1550-byte test that is 1522 bytes of data on Linux or Windows. I check each underlay link. Then I fix the MTU on the fabric, since VXLAN packets should not be fragmented.

## Quick Recap

- VXLAN carries Ethernet frames inside UDP/IP on destination port 4789. EVPN is the BGP control plane that fills the tables.
- A VLAN ID is 12 bits, giving 4094 usable VLANs. A VNI is 24 bits, giving about 16 million segments.
- The overhead is 50 bytes (14 + 20 + 8 + 8), so the underlay needs an IP MTU of at least 1550. Jumbo frames are the common choice.
- The VTEP wraps and unwraps. The UDP source port is a hash of the inner headers, for ECMP. Spines only route the outer IP.
- Flood-and-learn learns from data packets. BGP EVPN learns through BGP and reduces flooding with ARP suppression.
- Type 2 is MAC/IP, Type 3 is VNI membership and flood list, Type 5 is IP prefix, Types 1 and 4 are for multi-homing.
- The anycast gateway puts the same IP and MAC on every leaf. Symmetric IRB routes through an L3 VNI, on both the first and last leaf.
