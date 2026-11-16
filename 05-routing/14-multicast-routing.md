# Multicast Routing

**Level:** Advanced | **Prerequisites:** [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md), [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md), [How Routers Work](01-how-routers-work.md), [The Routing Table](02-routing-table.md)

## In One Line

Multicast sends one copy of a packet into the network, and routers and switches copy it only where a branch leads to a receiver that asked for it. Hosts ask with IGMP, routers build the delivery tree with PIM, and the RPF check stops loops.

## Why It Matters

Imagine a 5 Mbps live video stream with 1,000 viewers.

- **Unicast** (one sender, one receiver): the server sends 1,000 copies. That is 5 Gbps leaving the server.
- **Broadcast** (one sender, everyone on the LAN): it reaches every device in the VLAN, wanted or not. It also stops at the router.
- **Multicast** (one sender, only those who asked): the server sends 1 copy, 5 Mbps. Each link carries one copy at most, however many viewers sit behind it.

The saving is large when many receivers want the same data at the same time. Real networks use multicast for IPTV from an ISP, stock market data, video surveillance, live video production, and the underlay of some VXLAN fabrics. Routing protocols also use it on a local link: OSPF hellos go to 224.0.0.5.

Interviewers ask about multicast because it turns normal routing upside down. Unicast routing asks "where is the destination?". Multicast routing asks "where did this packet come from?". A candidate who can explain that one sentence, the RPF check, and the steps of PIM sparse mode stands out, because many engineers only know the names.

## Simple Explanation

Think of a **radio station**. The station transmits once. A listener tunes to a frequency. The station does not know who is listening, and 10 or 10 million listeners cost it the same.

- The **frequency** is the multicast group address, such as 239.1.1.1.
- **Tuning in** is a host joining the group with IGMP (Internet Group Management Protocol).
- The **station** is the source. It only needs to send to the group address.

Where the analogy stops being true: radio waves reach everyone, and the radio chooses to listen. In a network, nothing arrives until someone asks. Each router must build a path toward each listener, and it tears down that path when the last listener leaves.

Think of the network as water pipes and valves. All valves start closed. Each time a person in a house asks for water, the street valve for that house opens, then the district valve, back toward the source. The pipe tree grows only where people asked. PIM (Protocol Independent Multicast) is the plumber that opens and closes the valves.

Multicast uses UDP in almost all cases. TCP needs a two-way conversation with one partner, so it cannot work with one sender and an unknown crowd. There is no built-in retransmission. See [UDP in Depth](../06-transport-layer/07-udp.md).

## How It Works

### The three jobs

Multicast needs three separate jobs done. Most confusion comes from mixing them up.

```text
 Source                                            Receiver
   |                                                  |
   |  Job 3: router to router (PIM builds the tree)   |
  [R1] ------------ [R2] ------------ [R3]            |
                                       |              |
                                  Job 2: host to router (IGMP)
                                       |              |
                                    [Switch] ---------+
                                  Job 2b: IGMP snooping keeps
                                  frames off unused ports

 Job 1: everyone agrees on addresses (group IP and group MAC)
```

Three terms are used everywhere:

- **Group (G):** a multicast address that stands for "everyone who wants this data".
- **Source (S):** the sender. Its source IP is a normal unicast IP. A multicast address is never a source address.
- **Receiver:** a host that joined the group. A sender does not need to join.

Two router roles:

- **FHR (First Hop Router):** the router next to the source.
- **LHR (Last Hop Router):** the router next to the receivers.

### Multicast IPv4 addresses

All IPv4 multicast addresses are in **224.0.0.0/4**, which is 224.0.0.0 to 239.255.255.255. The first 4 bits are always 1110. The old name is Class D. See [IPv4 Classes](../04-ip-addressing/02-ipv4-classes.md).

| Range | Name | Meaning |
|---|---|---|
| 224.0.0.0/24 | Local Network Control | Stays on the local link. Routers never forward it, whatever the TTL. |
| 224.0.1.0/24 | Internetwork Control | Control traffic that can be routed, such as 224.0.1.1 (NTP). |
| 232.0.0.0/8 | SSM | Source-Specific Multicast (explained below). |
| 233.0.0.0/8 | GLOP | Addresses derived from a 16-bit AS number (RFC 3180). Now rare. |
| 233.252.0.0/24 | MCAST-TEST-NET | For documentation. |
| 239.0.0.0/8 | Administratively scoped | Private to an organization, like RFC 1918 for unicast (RFC 2365). |

Other parts of the range have other assignments. RFC 5771 holds the full guidelines, and IANA keeps the registry.

Addresses you should recognise:

| Address | Used by |
|---|---|
| 224.0.0.1 | All systems on this link |
| 224.0.0.2 | All routers on this link |
| 224.0.0.5 and 224.0.0.6 | OSPF all routers and OSPF DRs |
| 224.0.0.9 | RIPv2 |
| 224.0.0.10 | EIGRP |
| 224.0.0.13 | All PIM routers |
| 224.0.0.18 | VRRP |
| 224.0.0.22 | IGMPv3 reports |
| 224.0.0.251 | mDNS |
| 224.0.0.252 | LLMNR |
| 239.255.255.250 | SSDP (device discovery in UPnP) |

Use 239.0.0.0/8 for your own private applications, or 232.0.0.0/8 for SSM.

### Mapping a group IP to a MAC address

A switch works with MAC addresses, so each group IP needs a group MAC. No ARP is needed, because the MAC is calculated from the IP.

The rule for IPv4: **start with 01:00:5E, then a 0 bit, then the low 23 bits of the IP.**

```text
Bits of 239.1.1.1:

   1110    11110    0000001 00000001 00000001
   ----    -----    ------------------------
   fixed   5 bits   low 23 bits
           LOST     copied into the MAC

MAC bits:

   00000001 00000000 01011110 0 0000001 00000001 00000001
   -------- -------- -------- - ------------------------
      01       00       5E    0    01       01       01

Result: 01:00:5E:01:01:01
```

A quick method by hand:

1. Write 01:00:5E.
2. Take the second octet of the IP. If it is 128 or more, subtract 128.
3. Write that value in hex, then the third and fourth octets in hex.

| Group IP | Working | MAC |
|---|---|---|
| 224.0.0.1 | 0, 0, 1 | 01:00:5E:00:00:01 |
| 224.0.0.251 | 0, 0, 251 | 01:00:5E:00:00:FB |
| 239.1.1.1 | 1, 1, 1 | 01:00:5E:01:01:01 |
| 239.129.1.1 | 129 - 128 = 1, 1, 1 | 01:00:5E:01:01:01 |
| 232.10.20.30 | 10, 20, 30 | 01:00:5E:0A:14:1E |
| 239.200.5.9 | 200 - 128 = 72, 5, 9 | 01:00:5E:48:05:09 |
| 239.255.255.250 | 255 - 128 = 127, 255, 250 | 01:00:5E:7F:FF:FA |

**The 32-to-1 overlap.** A group IP has 28 free bits, but only 23 go into the MAC. The 5 lost bits give 2^5 = 32 group IPs for every MAC. So 224.1.1.1, 224.129.1.1, 225.1.1.1, and 239.129.1.1 all use 01:00:5E:01:01:01.

Analogy: 32 families share one mailbox. The mailbox gets all their letters, and each family reads the name on the envelope and throws away the rest. Where the analogy stops: the host does this filtering in software, so it spends CPU on packets it did not want.

Real effects of the overlap:

- A switch cannot tell 239.1.1.1 from 239.129.1.1 if it filters by MAC. Both go to the same ports.
- The host's network card accepts the frame. The IP layer then checks the real group and drops it if the host did not join.
- **Dangerous case:** any group that maps into 01:00:5E:00:00:xx shares a MAC with 224.0.0.x. For example 239.0.0.1 and 239.128.0.1 both map to 01:00:5E:00:00:01. Many switches flood these MACs to all ports, because link-local control traffic must never be blocked. So avoid x.0.0.y and x.128.0.y groups in your applications.

**Why the first byte is 01:** the lowest bit of the first MAC byte is the group bit. When it is 1, the frame is for a group. See [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md).

**IPv6 mapping:** start with 33:33, then copy the low 32 bits of the group. So ff02::1 becomes 33:33:00:00:00:01. The solicited-node group ff02::1:ff12:3456 becomes 33:33:FF:12:34:56.

### IGMP: how a host asks for a group

IGMP runs between hosts and their local router. It is IP protocol 2. Its packets have TTL 1, so it never leaves the link. It is not a routing protocol, and it says nothing about paths.

**Normal cycle:**

1. A host wants group G. It sends a **Membership Report** on the LAN, without waiting to be asked.
2. The router notes "this interface has a member of G". It does not track each host, only whether at least one member exists. (Some vendors can track hosts as an option.)
3. One router, the **querier**, sends a **General Query** to 224.0.0.1 from time to time. It asks "who still wants anything?".
4. Hosts answer with Reports after a random short delay.
5. If nobody answers for long enough, the router removes G from that interface.

Timers from the standard: Query Interval 125 seconds, Query Response Interval 10 seconds, Robustness Variable 2. Membership times out after (2 x 125) + 10 = 260 seconds. Vendors differ: Cisco IOS sends queries every 60 seconds by default.

**IGMP message types:**

| Type (hex) | Message | Sent to |
|---|---|---|
| 0x11 | Membership Query (General) | 224.0.0.1 |
| 0x11 | Membership Query (Group-Specific) | The group |
| 0x12 | IGMPv1 Report | The group |
| 0x16 | IGMPv2 Report | The group |
| 0x17 | IGMPv2 Leave Group | 224.0.0.2 |
| 0x22 | IGMPv3 Report | 224.0.0.22 |

The IGMPv2 packet is 8 bytes:

```text
 0               8               16                             31
+---------------+---------------+-------------------------------+
| Type (8 bits) | Max Resp Time |        Checksum (16 bits)     |
+---------------+---------------+-------------------------------+
|                    Group Address (32 bits)                    |
+---------------------------------------------------------------+

Max Resp Time is in tenths of a second. 100 means 10 seconds.
```

**IGMP versions:**

| Feature | IGMPv1 | IGMPv2 | IGMPv3 |
|---|---|---|---|
| RFC | 1112 | 2236 | 9776 (replaced 3376) |
| Leave message | No, membership times out | Yes, to 224.0.0.2 | Yes, as a state-change report |
| Query for one group | No | Yes | Yes, also group-and-source |
| Querier election | No, uses the routing protocol | Yes, lowest IP wins | Yes, lowest IP wins |
| Max response time in query | No, fixed at 10 seconds | Yes | Yes |
| Report sent to | The group | The group | 224.0.0.22 |
| Report suppression | Yes | Yes | No |
| Pick sources | No | No | Yes |

**Report suppression (v1, v2):** a host hears another host's report for the same group and cancels its own. The router only needs one answer. This saves traffic, but it also means the router cannot count hosts. IGMPv3 drops this, so a switch or router can see each host's report.

**Leave in IGMPv2:**

1. The host sends Leave to 224.0.0.2.
2. The querier sends a Group-Specific Query to that group. It repeats this, by default 2 times, 1 second apart.
3. If another host still wants it, that host sends a Report and the stream continues.
4. If nobody answers, the router stops forwarding G on that interface.

So a leave takes about 2 seconds. With IGMPv1 it can take minutes. That matters for IPTV, where channel changes are a leave plus a join.

**Source filtering in IGMPv3:** a host states a filter mode and a source list for each group.

- **INCLUDE {S1}:** "I only want this group from S1." SSM needs this.
- **EXCLUDE {S2}:** "I want this group from anyone but S2."
- **EXCLUDE { }:** "I want this group from anyone." This equals a plain v2 join.

**Mixed versions:** versions work together by falling back. If a v2 router sends queries, a v3 host answers in v2. If a v3 router hears a v2 report for a group, it uses v2 rules for that group.

**Host defaults:** Linux and Windows both use IGMPv3 by default, and fall back as above.

**For IPv6**, the same job is done by **MLD** (Multicast Listener Discovery), carried inside ICMPv6. MLDv1 (RFC 2710) matches IGMPv2. MLDv2 (RFC 9777, which replaced RFC 3810) matches IGMPv3.

### IGMP snooping: multicast on the switch

Without help, a switch treats multicast frames like broadcasts and floods them out of all ports in the VLAN. A switch never learns a group MAC as a source, because a source address is never a group address. A 10 Mbps stream then hits every port, including a printer on a slow link.

**IGMP snooping** fixes this. The switch reads IGMP packets as they pass, even though IGMP is a Layer 3 protocol, and builds a table of which ports want which group.

Analogy: a hotel floor manager listens to guests ordering newspapers, and then delivers each paper only to the rooms that ordered it. Where it stops: the manager also has to send every order down to the front desk, because the switch must pass reports up to the router.

```text
                  [Router]  mrouter port (Gi1/0/24)
                     |
                  [Switch]  snooping table:
                 /   |    \   VLAN 20, 239.1.1.1 -> Gi1/0/5, Gi1/0/24
                /    |     \
         Gi1/0/5  Gi1/0/6  Gi1/0/7
         [PC A]   [PC B]   [Printer]
         joined   nothing  nothing
           ^
           |  Stream reaches only PC A
```

How it works:

1. The switch finds **mrouter ports** (ports that lead to a multicast router). It learns them from IGMP Queries and PIM Hellos, or from manual config.
2. A host Report arrives. The switch adds that port to the group and sends the Report only to mrouter ports. Other hosts do not hear it, so they all keep reporting. This also works around report suppression.
3. Traffic for the group goes only to member ports and mrouter ports.
4. On a Leave, the switch checks whether others remain, usually with a group query, then removes the port.

Key facts:

- **224.0.0.x groups are always flooded.** OSPF hellos would break otherwise.
- **Snooping needs a querier.** Without Queries, entries age out. If a VLAN has no multicast router, turn on the **snooping querier** feature on one switch. A classic symptom of a missing querier: the stream works for about 3 to 5 minutes, then stops.
- **Spanning Tree changes:** after a topology change, many switches flood multicast for a short time, so that a query can rebuild the table.
- **Unknown multicast** (a group with no entries) behaves differently by platform. Some flood it, and some drop it. Check your switch's docs.
- **MLD snooping** does the same job for IPv6.
- Snooping is defined as guidance in RFC 4541, which is Informational. Details vary by vendor. Cisco Catalyst switches have it on by default. Cheap unmanaged switches often have no snooping at all.

### Trees: the shape of multicast delivery

A **distribution tree** is the path a group's packets follow. Routers store trees as multicast routes, written in one of two forms.

| Notation | Meaning | Name |
|---|---|---|
| (S, G) | Traffic from this source to this group | Source tree, also SPT (Shortest Path Tree) |
| (*, G) | Traffic from any source to this group | Shared tree, also RPT (Rendezvous Point Tree) |

Each entry has an **incoming interface** (IIF) and an **outgoing interface list** (OIL). A packet comes in one interface and is copied to every interface in the OIL.

```text
Source tree: rooted at the source. Each source has its own tree.

   S ---[R1]---[R3]--- Receiver      shortest path

Shared tree: rooted at the RP. All sources use the same tree.

   S ---[R1]---[RP]---[R3]--- Receiver    maybe longer
```

| | Source tree (S,G) | Shared tree (*,G) |
|---|---|---|
| Root | The source | The RP |
| Path | Shortest from source to receiver | Source to RP, then RP to receiver |
| State on routers | One entry per source per group | One entry per group |
| Delay | Lower | Can be higher |
| RP needed | No | Yes |

Analogy: a source tree is a direct flight. A shared tree is a flight through a hub. Hub travel needs fewer routes, but the trip can be longer. Where it stops: with planes you pay for each route, but in a network the extra cost is memory on the routers.

### The RPF check: how loops are stopped

A unicast router looks at the destination. A multicast router cannot, since a group is not a place. It looks at the **source** first and asks:

> "Did this packet arrive on the interface I would use to send traffic back to the source?"

This is the **Reverse Path Forwarding (RPF) check**.

1. A packet from S arrives on an interface.
2. The router looks up S in the unicast routing table.
3. If the route points out that same interface, the check passes. The packet is copied to the OIL.
4. If not, the packet is dropped silently.

For a (*,G) shared tree, the check uses the RP's address in place of the source.

```text
Source 192.168.10.10

      [R1]----------[R2]
        \           /
    Gi0/1\         /Gi0/0
          \       /
            [R3]

R3 route: 192.168.10.0/24 via Gi0/1

 Packet arrives on Gi0/1 -> RPF interface  -> PASS, forward
 Packet arrives on Gi0/0 -> wrong interface -> FAIL, drop
```

Why this stops loops: a looped copy arrives from a wrong direction, so it fails the check and dies. Multicast has no single destination to stop at, so without this check a loop could multiply packets very fast.

Analogy: a courier gets a parcel from Karachi and checks the road it came in on. If that is not the road to Karachi, the parcel took a strange route, so the courier throws it away. Where it stops: a real courier would investigate. A router drops the packet with no error message, so an RPF failure looks like silence.

The RPF check needs a working unicast table. That is why **PIM is called "protocol independent"**: it does not exchange its own routes. It uses whatever fills the routing table, whether OSPF, BGP, or static routes.

Do not confuse this with **unicast RPF (uRPF)**, an anti-spoofing feature. They share a name and an idea, but uRPF is an optional security feature, while the multicast RPF check is always on and is required for forwarding.

### PIM basics

PIM runs between routers. It is IP protocol 103, and PIM Hellos go to 224.0.0.13 with TTL 1.

- **Hello:** every 30 seconds by default. Hold time 105 seconds. Finds neighbors.
- **Join/Prune:** "send me this traffic" or "stop". Sent toward the upstream neighbor. State is soft, so joins repeat every 60 seconds by default, and the state expires after 210 seconds without a refresh.
- **Register and Register-Stop:** unicast messages between the FHR and the RP, in sparse mode.
- **Assert:** picks one forwarder on a shared LAN.

PIM has four main modes:

| Mode | Idea | Standard | Today |
|---|---|---|---|
| PIM-DM (Dense Mode) | Flood first, prune later | RFC 3973 (Experimental) | Legacy |
| PIM-SM (Sparse Mode) | Nothing flows until joined, uses an RP | RFC 7761 | The usual choice |
| PIM-SSM | Sparse mode with source-only trees, no RP | RFC 4607 | Best for one-to-many |
| Bidir-PIM | One shared tree, traffic both ways | RFC 5015 | Many-to-many |

### PIM Dense Mode: flood and prune

Dense mode assumes "almost everyone wants this". It is a push model.

Analogy: a shop puts a flyer in every mailbox in the city. Anyone who does not want it calls to say stop. After 3 minutes the shop forgets and sends flyers again.

1. The source sends. The FHR floods the traffic out of all PIM interfaces.
2. Every router does the RPF check, then floods onward.
3. A router with no receivers and no downstream neighbors sends a **Prune** upstream.
4. The upstream router stops sending on that branch.
5. The prune state expires, after about 3 minutes (RFC 3973 uses a default of 210 seconds). Flooding starts again and the prunes repeat.
6. If a new receiver appears on a pruned branch, its router sends a **Graft**, which is acknowledged with a Graft-Ack. The branch reopens without waiting.

```text
Step 1: flood                      Step 2: after prunes

    S                                  S
    |                                  |
  [R1]---->[R2]---->[R3]             [R1]---->[R2]      [R3]
    |        |                         |        ^ prune   ^
    v        v                         v        | from    | prune
  [R4]     [R5]--Rx                  [R4]     [R5]--Rx
  no rx                              R4 pruned, R3 pruned
```

PIM-DM needs no RP. But it wastes bandwidth and puts (S,G) state on every router, including routers with no receivers. An optional **State Refresh** message (every 60 seconds by default) keeps prunes alive and stops the repeat flood. It is little used, since dense mode itself is rare now.

### PIM Sparse Mode: join first

Sparse mode assumes "few want this, so send nothing until asked". It is a pull model. It needs a meeting point, because a source and its receivers have no way to find each other.

That meeting point is the **Rendezvous Point (RP)**, a router chosen by the network admin.

Analogy: a notice board at a train station. A person selling a bicycle posts a notice. A buyer checks the board. Once they meet, they can deal directly and stop using the board. Where it stops: a board only holds notices, but an RP may carry the actual data for a short time.

Rules to remember:

- Receivers' routers send (*,G) Joins **toward the RP**.
- Sources' routers send **Register** messages to the RP.
- Each router sends a Join out of its RPF interface, so the tree is built along the reverse of the unicast path.
- Each LAN elects a **DR (Designated Router)**, the one router on that LAN that sends Joins and Registers.

The full packet-by-packet story is in the Example section below. In short:

1. Receiver sends an IGMP Report. The LHR sends a (*,G) Join to the RP.
2. Source sends. The FHR wraps the packets in Register messages and unicasts them to the RP.
3. The RP unwraps them and sends them down the shared tree. It joins toward the source with an (S,G) Join.
4. Native multicast arrives at the RP. The RP sends Register-Stop to the FHR.
5. The LHR learns the source's address from the first packet. It joins directly toward the source. This is the **SPT switchover**.
6. The LHR prunes that source from the shared tree. Traffic now takes the shortest path.

**If no receiver exists** when a source starts, the RP sends Register-Stop at once. The FHR drops the packets, but it sends a small Null-Register from time to time to check for receivers. By default it suppresses registers for about 60 seconds at a time.

### Choosing the RP

Every router must agree on which RP serves which group.

| Method | How it works | Notes |
|---|---|---|
| Static | Type the RP address on all routers | Simple. Backup needs extra design. |
| Auto-RP | Candidate RPs announce on 224.0.1.39. A mapping agent chooses and publishes on 224.0.1.40 | Created by Cisco. Uses UDP port 496. |
| BSR | Candidates report to a Bootstrap Router, which floods the set hop by hop | Standard (RFC 5059). Each router runs the same hash to pick an RP. |
| Anycast RP | Several RPs share one IP. Nearest wins by routing | Gives fast failover. Works with any method above. |
| Embedded RP | IPv6 only: the RP address is inside the group address | RFC 3956. |

**Auto-RP's chicken-and-egg problem:** the RP info travels in multicast groups, but a sparse router needs an RP before it can receive multicast. Older designs used `ip pim sparse-dense-mode` to flood those two groups. Today, `ip pim autorp listener` does this for only those two groups.

Place the RP on a loopback, at a central point, on a strong router. It should be close to the sources, if most sources sit in one place. After SPT switchover, it mostly carries little data.

### Source-Specific Multicast (SSM)

The model above is **ASM (Any-Source Multicast)**: receivers join a group, and any source can send. It needs an RP so that receivers can find sources.

With SSM, the receiver says exactly which source it wants, so there is nothing to discover.

- A receiver joins an (S,G) **channel** with IGMPv3 INCLUDE. Its LHR sends an (S,G) Join straight toward S.
- There is no RP, no shared tree, no Register, and no SPT switchover.
- The default range is 232.0.0.0/8 for IPv4 and ff3x::/32 for IPv6 (RFC 4607).

Why it is better for one-to-many:

- **Simpler.** The RP and MSDP go away.
- **Safer.** A stranger cannot inject traffic to your receivers, since a different source is a different channel.
- **More addresses.** (S1,G) and (S2,G) are separate channels, so two sources can reuse one group.

The catch: the receiver must learn the source's IP some other way. In practice this comes from a channel guide, a web page, or SDP (Session Description Protocol) data. See [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

If some hosts only speak IGMPv2, Cisco's **SSM mapping** can convert their (*,G) joins to (S,G) joins using a static or DNS-based table on the LHR.

### Comparing the modes

| | PIM-DM | PIM-SM (ASM) | PIM-SSM | Bidir-PIM |
|---|---|---|---|---|
| Model | Push | Pull | Pull | Pull |
| RP | No | Yes | No | Yes |
| Tree | (S,G) | (*,G), then (S,G) | (S,G) | (*,G) only |
| IGMP needed | v2 or more | v2 or more | v3 | v2 or more |
| Best for | Old small LANs | Unknown sources | One-to-many | Many-to-many |

## Example

### Example 1: PIM sparse mode, step by step

Topology, with OSPF running on all routers and all links at equal cost:

```text
 Source S 192.168.10.10               Receiver 192.168.20.25
      |                                     |
      | Gi0/2 192.168.10.1/24               | Gi0/2 192.168.20.1/24
    [R1] Gi0/1 10.0.13.1 ------- 10.0.13.2 Gi0/1 [R3]
      | Gi0/0 10.0.12.1                     | Gi0/0 10.0.23.2
      |                                     |
      |  10.0.12.2 Gi0/0     Gi0/1 10.0.23.1|
      +------------------[R2]---------------+
                     RP: Loopback0 10.0.0.2

 Group G = 239.1.1.1      MAC = 01:00:5E:01:01:01
 Links are /30. R1 = FHR, R2 = RP, R3 = LHR.
```

**Config (Cisco IOS).** Only the multicast lines are shown. Interface addresses and OSPF are assumed.

```text
! R1, R2 and R3 (all routers)
ip multicast-routing
ip pim rp-address 10.0.0.2
!
! On every interface in use, on every router
! (R2 has only Gi0/0 and Gi0/1)
interface GigabitEthernet0/0
 ip pim sparse-mode
interface GigabitEthernet0/1
 ip pim sparse-mode
interface GigabitEthernet0/2
 ip pim sparse-mode
!
! R2 also needs it on the RP loopback
interface Loopback0
 ip address 10.0.0.2 255.255.255.255
 ip pim sparse-mode
!
! R3 receiver LAN
interface GigabitEthernet0/2
 ip igmp version 3
```

Some platforms need `ip multicast-routing distributed` instead.

**Generate test traffic with iperf2.** iperf3 does not support multicast.

```bash
# Linux receiver (192.168.20.25): join the group
iperf -s -u -B 239.1.1.1 -i 5

# Linux source (192.168.10.10): 1 Mbps, TTL 16, 60 seconds
iperf -c 239.1.1.1 -u -b 1M -T 16 -t 60
```

The `-T 16` matters. The default multicast TTL on a socket is 1, so without it the first router would drop the traffic.

**Phase 1: the receiver joins.**

```text
Receiver --IGMP Report--> R3 --PIM (*,G) Join--> R2 (RP)

R3 creates: (*, 239.1.1.1)  IIF Gi0/0   OIL Gi0/2
R2 creates: (*, 239.1.1.1)  IIF Null    OIL Gi0/1
```

Check the host side:

```bash
# Linux: see the joined groups
ip maddr show dev eth0
```

```text
2:	eth0
	link  01:00:5e:00:00:01
	link  01:00:5e:01:01:01
	inet  239.1.1.1
	inet  224.0.0.1
```

```bash
# Linux: watch IGMP on the wire
sudo tcpdump -ni eth0 -v igmp
```

```text
IP (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2),
length 40, options (RA))
 192.168.20.25 > 224.0.0.22: igmp v3 report, 1 group record(s)
 [gaddr 239.1.1.1 to_ex, 0 source(s)]
```

`to_ex, 0 source(s)` means "change to EXCLUDE nothing". That is "I want this group from any source". `options (RA)` is the Router Alert option, which tells routers to look inside the packet.

```powershell
# Windows: see the joined groups
netsh interface ipv4 show joins
```

```text
Interface 12: Ethernet

Scope       References  Last  Address
----------  ----------  ----  ------------------
0                    0  Yes   224.0.0.1
0                    1  Yes   239.1.1.1
```

**Phase 2: the source starts, and R1 registers.**

```text
S --multicast--> R1 ==Register (unicast)==> R2 --multicast--> R3 --> Rx

 R1 wraps each packet:

 +---------------------------+-----------------+--------------------+
 | Outer IP                  | PIM Register    | Original packet    |
 | 10.0.12.1 -> 10.0.0.2     | header          | S -> 239.1.1.1     |
 | protocol 103              |                 |                    |
 +---------------------------+-----------------+--------------------+
```

Register is a tunnel, so the first packets reach the receiver before any native tree exists to the source. It is slow, since the RP has to unwrap each packet.

**Phase 3: the RP joins toward the source.**

```text
R2 --PIM (S,G) Join--> R1           (R1 adds Gi0/0 to OIL)
R1 --native multicast--> R2
R2 --Register-Stop (unicast)--> R1  (R1 stops wrapping)
```

**Phase 4: SPT switchover.** R3 now sees the source's IP in the first packet. Cisco IOS moves to the source tree after the first packet by default.

```text
R3 --PIM (S,G) Join out of Gi0/1--> R1   (R3 route to S is via R1)
R1 --native multicast on Gi0/1--> R3
R3 --(S,G,rpt) Prune--> R2               ("not this S on shared tree")
R2 --(S,G) Prune--> R1                   (R2 has no one left for S)
```

**Final state:**

```text
S --> R1 --> R3 --> Receiver      (2 hops, R2 carries no data)

R2 stays the RP, with the (*,G) tree ready for new sources.
```

**Verify on R3 (LHR).** Legends are removed from all outputs below.

```text
R3# show ip igmp groups
Group Address  Interface  Uptime    Expires   Last Reporter
239.1.1.1      Gi0/2      00:05:12  00:02:41  192.168.20.25

R3# show ip pim neighbor
Neighbor    Interface  Uptime/Expires     Ver  DR Prio/Mode
10.0.23.1   Gi0/0      01:12:05/00:01:34  v2   1 / S P G
10.0.13.1   Gi0/1      01:12:03/00:01:22  v2   1 / S P G

R3# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s): 224.0.0.0/4, Static
    RP: 10.0.0.2 (?)

R3# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:05:10/stopped, RP 10.0.0.2, flags: SJC
  Incoming interface: GigabitEthernet0/0, RPF nbr 10.0.23.1
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:05:10/00:02:43

(192.168.10.10, 239.1.1.1), 00:03:02/00:02:57, flags: JT
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.13.1
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:03:02/00:02:43
```

How to read it:

- `(*, 239.1.1.1)` points at R2. This is the shared tree.
- `(192.168.10.10, 239.1.1.1)` points at R1. This is the source tree.
- `T` means the SPT is in use. `J` means join the SPT. `C` means a member is directly connected. `S` means sparse.

**On R1 (FHR):**

```text
R1# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:03:05/stopped, RP 10.0.0.2, flags: SPF
  Incoming interface: GigabitEthernet0/0, RPF nbr 10.0.12.2
  Outgoing interface list: Null

(192.168.10.10, 239.1.1.1), 00:03:05/00:03:24, flags: FT
  Incoming interface: GigabitEthernet0/2, RPF nbr 0.0.0.0
  Outgoing interface list:
    GigabitEthernet0/1, Forward/Sparse, 00:03:02/00:02:57
```

`F` is the Register flag: this router is the first hop for this source. RPF neighbor 0.0.0.0 means the source is directly connected.

**On R2 (RP):**

```text
R2# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:05:10/00:03:19, RP 10.0.0.2, flags: S
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    GigabitEthernet0/1, Forward/Sparse, 00:05:10/00:03:19

(192.168.10.10, 239.1.1.1), 00:03:05/00:02:20, flags: PT
  Incoming interface: GigabitEthernet0/0, RPF nbr 10.0.12.1
  Outgoing interface list: Null
```

The (S,G) has an empty OIL and the `P` (pruned) flag. R2 still knows the source, but it does not carry the data. The incoming interface Null on (*,G) is normal on the RP, because the RP is the root of the shared tree.

**Check the switch on the receiver LAN** (Catalyst). R3 is on Gi1/0/24 and the receiver is on Gi1/0/5.

```text
SW1# show ip igmp snooping groups
Vlan  Group        Type  Version  Port List
---------------------------------------------------
20    239.1.1.1    igmp  v3       Gi1/0/5, Gi1/0/24

SW1# show ip igmp snooping mrouter
Vlan    ports
-------+---------
  20    Gi1/0/24(dynamic)
```

### Example 2: the same network with SSM

Add this on all routers, and keep IGMPv3 on R3's receiver LAN:

```text
! Cisco IOS: treat 232.0.0.0/8 as SSM
ip pim ssm default
```

The receiver asks for one source. No RP is used.

```bash
# Linux receiver, iperf2: join (192.168.10.10, 232.1.1.1)
iperf -s -u -B 232.1.1.1 -H 192.168.10.10 -i 5

# Linux source
iperf -c 232.1.1.1 -u -b 1M -T 16 -t 60
```

```text
R3# show ip mroute 232.1.1.1
(192.168.10.10, 232.1.1.1), 00:01:22/00:02:37, flags: sTI
  Incoming interface: GigabitEthernet0/1, RPF nbr 10.0.13.1
  Outgoing interface list:
    GigabitEthernet0/2, Forward/Sparse, 00:01:22/00:02:37
```

There is no (*,G) entry. `s` means SSM group, and `I` means a source-specific host report arrived. R3 sent an (S,G) Join straight to R1, and R2 never took part.

### Example 3: finding an RPF failure

**Symptom:** the first second of video appears, then stops. This happens when the shared tree works but the SPT switch fails.

R3's route to S goes out Gi0/1, but someone removed `ip pim sparse-mode` from that interface. Data keeps coming from R2 on Gi0/0, which is not the RPF interface for S.

```text
R3# show ip rpf 192.168.10.10
RPF information for ? (192.168.10.10)
  RPF interface: GigabitEthernet0/1
  RPF neighbor: ? (10.0.13.1)
  RPF route/mask: 192.168.10.0/24
  RPF type: unicast (ospf 1)

R3# show ip mroute count | include Source:
  Source: 192.168.10.10/32, Forwarding: 0/0/0/0, Other: 1500/1500/0
```

The `Other` numbers are total, RPF failed, and other drops. 1,500 of 1,500 packets failed RPF.

Fix A, the right fix: enable PIM on the link.

```text
interface GigabitEthernet0/1
 ip pim sparse-mode
```

Fix B, when that link must not carry multicast: override RPF with a static multicast route. It only affects the RPF check, and it does not change unicast forwarding.

```text
ip mroute 192.168.10.0 255.255.255.0 10.0.23.1
```

R3 now uses R2 as the upstream for this source.

### Example 4: a quick test without a host

```text
! Cisco IOS on R3: make the router itself join, then ping from R1
R3(config)# interface GigabitEthernet0/2
R3(config-if)# ip igmp join-group 239.1.1.1

R1# ping 239.1.1.1 repeat 3
Reply to request 0 from 10.0.13.2, 8 ms
Reply to request 1 from 10.0.13.2, 4 ms
Reply to request 2 from 10.0.13.2, 4 ms
```

`ip igmp join-group` handles packets in the router's CPU. To force forwarding onto a port without a real receiver, and without loading the CPU, use `ip igmp static-group`. Remove test joins afterwards.

## Going Deeper

### Designated Router and querier are different

On a LAN with two routers, two separate elections run.

| Role | Protocol | Winner | Job |
|---|---|---|---|
| IGMP querier | IGMPv2, v3 | **Lowest** IP | Sends Queries |
| PIM DR | PIM Hello | **Highest** priority (default 1), then **highest** IP | Sends Registers and Joins |

The same router may not win both. That is normal.

```text
! Cisco IOS: make this router the DR on this LAN
interface GigabitEthernet0/2
 ip pim dr-priority 100
```

### Assert: stopping duplicates on a LAN

Two routers on one LAN can both have a route to a source and both forward, which gives duplicate packets. Each hears the other's packet on an outgoing interface, and that triggers an **Assert** message.

The winner is chosen by, in order: lowest administrative distance to the source, then lowest metric, then highest IP. The loser prunes its interface. Downstream routers then send Joins to the Assert winner.

### Join suppression and prune override

On a LAN with several downstream routers, one router's Prune could cut off another that still wants the data. So the upstream router waits about 3 seconds before pruning. A second downstream router hears the Prune and sends a Join to override it.

In the same way, a router that hears another's Join for the same tree can skip its own.

### Controlling SPT switchover

SPT switchover is an implementation choice, not a must. Cisco IOS switches after the first packet by default.

```text
! Cisco IOS: stay on the shared tree for all groups
ip pim spt-threshold infinity
```

Reasons to stay: fewer (S,G) entries when there are many low-rate sources. Reasons to switch: shorter paths and less load on the RP.

In the (S,G,rpt) Prune from Example 1, the "rpt" means "prune this source on the shared tree only". It is a different message from a normal (S,G) Prune. Where the two trees share links, one router, the **turnaround router**, handles the change. You rarely need more detail.

### Bidirectional PIM

Many-to-many apps, such as trading floors or video conferencing with many senders, create huge (S,G) state in sparse mode. Bidir-PIM (RFC 5015) keeps only (*,G) entries.

- Traffic flows up to the RP and down to receivers on the same tree.
- There are no Registers and no (S,G) state.
- Each link elects a **DF (Designated Forwarder)**, the only router allowed to forward toward the RP. This replaces the RPF check for loop safety.
- The RP does not need to be a real router: an unused address on a known subnet works, a design called a **phantom RP**.

### Anycast RP and MSDP

A single RP is a single point of failure. **Anycast RP** gives two or more routers the same /32 address, and routing picks the nearest.

A problem follows: S registers with RP-A while the receiver joined at RP-B. They need to share source info. **MSDP** (Multicast Source Discovery Protocol, RFC 3618) does this. RPs peer over TCP port 639 and send **SA (Source Active)** messages to say "I have source S for group G".

```text
              MSDP peering (TCP 639)
     [RP-A] <========================> [RP-B]
    Lo1 10.0.0.100                  Lo1 10.0.0.100  (shared)
    Lo0 10.0.0.11                   Lo0 10.0.0.12   (unique)
       ^                                ^
       | Register from S                | (*,G) Join
      [FHR]                            [LHR]
```

```text
! Cisco IOS on RP-A (RP-B mirrors this with 10.0.0.11)
interface Loopback0
 ip address 10.0.0.11 255.255.255.255
interface Loopback1
 ip address 10.0.0.100 255.255.255.255
 ip pim sparse-mode
!
ip pim rp-address 10.0.0.100
ip msdp peer 10.0.0.12 connect-source Loopback0
ip msdp originator-id Loopback0
```

RFC 4610 defines a version that uses PIM itself to share Registers between RPs, without MSDP.

When an RP fails, the shared address moves to the other RP as soon as unicast routing converges. Existing SPT flows keep going, since they do not cross the RP. New sources and new receivers are what break on a failed RP that has no backup.

### Interdomain multicast

Across AS boundaries, the classic stack was PIM-SM, MSDP between RPs, and **MBGP** (Multiprotocol BGP) with the multicast address family. MBGP lets the multicast RPF path differ from the unicast path. See [BGP](09-bgp.md).

RFC 8815 (BCP 229) says to phase out ASM between domains and use SSM instead. MSDP itself is Experimental. Multicast is rare on the public Internet, and large streaming services use unicast through [CDNs](../10-enterprise-and-datacenter/06-cdn.md).

**AMT** (Automatic Multicast Tunneling, RFC 7450) lets receivers on a network without multicast reach one with it through tunnels.

### Multicast over tunnels and VPNs

Classic IPsec tunnels do not carry multicast, so a usual design is GRE inside IPsec, with PIM running on the tunnel interface. See [IPsec](../09-security/08-ipsec.md).

Watch the RPF check. If the unicast route to the source uses the physical path, packets arriving on the tunnel fail RPF. Fix this with a static mroute through the tunnel:

```text
ip mroute 192.168.10.0 255.255.255.0 Tunnel0
```

In service provider networks, **MVPN** carries customer multicast across MPLS. Older designs use GRE-based default and data MDTs (Multicast Distribution Trees, RFC 6037). Newer designs use BGP signaling (RFC 6513, RFC 6514) with mLDP (RFC 6388) or RSVP-TE trees. See [MPLS](13-mpls.md).

**BIER** (Bit Index Explicit Replication, RFC 8279) puts a bitmask of exit routers in each packet, so core routers keep no per-flow tree state. It is newer, and support is limited.

### IPv6 multicast

- Addresses are in **ff00::/8**. The 4th hex digit holds the **scope**: ff02:: is link-local, ff05:: is site-local, ff0e:: is global.
- Well-known groups: ff02::1 (all nodes), ff02::2 (all routers), ff02::5 and ff02::6 (OSPFv3), ff02::d (PIM routers), ff02::16 (MLDv2 reports).
- IPv6 has no broadcast. Solicited-node groups (ff02::1:ffxx:xxxx) replace ARP broadcast. See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).
- MLD runs on ICMPv6: type 130 Query, 131 MLDv1 Report, 132 Done, 143 MLDv2 Report.
- PIM works the same way. On Cisco IOS, `ipv6 multicast-routing` enables PIMv6 on IPv6 interfaces automatically.
- SSM uses ff3x::/96 groups in practice, inside the ff3x::/32 block.

### Scoping: keeping multicast inside

- **TTL scoping (older):** the sender sets a low TTL. It is a blunt tool.
- **Admin scoping:** block a range at a border.

```text
! Cisco IOS: block 239.0.0.0/8 at the site border
ip access-list standard NO-PRIVATE-MCAST
 deny   239.0.0.0 0.255.255.255
 permit 224.0.0.0 15.255.255.255
!
interface GigabitEthernet0/3
 ip multicast boundary NO-PRIVATE-MCAST
```

### Security and scale

Risks and controls:

- **Fake sources:** with ASM, anybody can send to a group. Limit senders with `ip pim accept-register` on the RP, or move to SSM.
- **Fake joins:** a host joining hundreds of groups can fill the router's state table. Limit with `ip igmp access-group` and `ip igmp limit`.
- **Rogue PIM routers:** filter neighbors with `ip pim neighbor-filter`. A rogue RP can capture streams, so filter Auto-RP and BSR messages at the edge.
- **Rogue queriers:** on some networks, a host that sends Queries can disrupt snooping.
- **Hardware limits:** switches hold multicast entries in limited hardware tables. Too many (S,G) entries can push forwarding into the CPU. Plan for the group count.

Multicast has no flow control. A slow receiver just loses packets. Protect streams with [QoS](../10-enterprise-and-datacenter/03-quality-of-service.md).

### Real use cases

| Use case | Why multicast fits | Typical design |
|---|---|---|
| ISP IPTV | Hundreds of channels to many homes | SSM, IGMP from the set-top box, snooping in access switches, fast leave for channel changes |
| Market data | Same prices to many traders, at the same moment | Dual feeds (A and B) over separate networks, SSM or PIM-SM, gap recovery at app level |
| Video surveillance | Many viewers, recorders, and displays per camera | PIM-SM or SSM inside the enterprise, 239.x groups |
| Live video production | Uncompressed flows at many Gbps, such as SMPTE ST 2110 | Separate networks with PIM or controller-set paths, plus timing with PTP |
| OS imaging | One image to a whole lab at one time | Single VLAN, with snooping and a querier |
| VXLAN underlay | BUM traffic (broadcast, unknown unicast, multicast) in a fabric | PIM-SM or Bidir, though many fabrics now use ingress replication. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) |
| Local service discovery | Finding printers and speakers | mDNS, SSDP, link-local or a site scope |
| Routing protocols | Reaching all neighbors at once | OSPF, EIGRP, PIM itself, on 224.0.0.x |

**Public cloud:** most cloud networks do not offer native multicast at the VPC level. AWS Transit Gateway has a multicast feature. Many apps use overlays or unicast instead. Always check the provider's current docs.

### Troubleshooting path

Work from the receiver back to the source.

1. **Host:** did it join? `ip maddr` or `netsh interface ipv4 show joins`. Does the firewall allow it?
2. **Switch:** `show ip igmp snooping groups`. Is the port listed? Is there a querier and an mrouter port?
3. **LHR:** `show ip igmp groups`. Is the group there?
4. **RP:** `show ip pim rp mapping` on each router. Do they all agree? Can each reach the RP?
5. **PIM:** `show ip pim neighbor`. Is PIM on every interface along the path?
6. **Tree:** `show ip mroute G`. Are the IIF and OIL right?
7. **RPF:** `show ip rpf S` and `show ip mroute count`. Is the RPF failed count rising?
8. **Source:** Is it sending, and with a big enough TTL? Run `tcpdump -ni eth0 host 239.1.1.1`.

`mtrace` traces the reverse path from receiver to source, where the platform supports it. See [Common Network Problems](../12-troubleshooting-and-tools/09-common-network-problems.md) for general methods.

## Common Mistakes

- **"Multicast needs a Layer 3 network to work."** Two hosts on one VLAN need no router. They need only group addresses and working switch behavior.
- **"IGMP is used between routers."** IGMP is host to router. PIM is router to router.
- **"PIM exchanges routes."** PIM has no routing table of its own. It uses the unicast table for RPF.
- **"RPF means the packet's destination is looked up."** It checks the source (or the RP, for a (*,G) tree).
- **"Sources must join the group."** A source just sends. Only receivers join.
- **"All multicast traffic flows through the RP."** Only until the SPT switchover in the usual setup. With SSM there is no RP at all.
- **"If the RP dies, all traffic dies."** Existing source-tree flows keep going. New joins and new sources fail.
- **"Each group IP has a unique MAC."** 32 IPs share one MAC.
- **"The switch handles multicast by itself."** Without IGMP snooping it floods. With snooping and no querier, it can stop delivering after a few minutes.
- **Using x.0.0.y groups in an app.** Such groups may map to the 224.0.0.x MACs and flood everywhere.
- **Keeping the sender TTL at 1.** Socket defaults are often 1, so the first router drops the packet. The stream works on the LAN but not across routers.
- **Forgetting that PIM must be on the RPF interface.** PIM on most of the path is not enough. A single missed link can cause RPF failures.
- **Thinking the querier and the DR must be one router.** They use opposite tie-breaks, lowest and highest IP.
- **Using SSM with IGMPv2 hosts.** IGMPv2 cannot name a source, so plain v2 joins will not work without SSM mapping.

## Interview Questions

**Q: When would you use multicast instead of unicast or broadcast?**
A: When one source has the same data for many receivers at the same time. Unicast sends one copy per receiver, so load grows with viewers. Broadcast reaches everyone on the VLAN and stops at the router. Multicast sends one copy per link and goes only where someone asked. It is usually UDP, so there is no built-in reliability.

**Q: How do you map 239.129.1.1 to a MAC address, and what is the catch?**
A: Take 01:00:5E, a 0 bit, then the low 23 bits of the IP. The second octet is 129, and I drop the top bit to get 1. So the MAC is 01:00:5E:01:01:01. The catch is that 5 bits are lost, so 32 group IPs share each MAC, and 239.1.1.1 gets this same MAC. Hosts filter the extra traffic at the IP layer.

**Q: What are the differences between IGMPv1, v2 and v3?**
A: v1 has queries and reports only, so a leave relies on timeout. v2 adds a Leave message, group-specific queries, querier election by lowest IP, and a configurable response time. v3 adds source filtering with INCLUDE and EXCLUDE, sends reports to 224.0.0.22, and removes report suppression. SSM needs v3.

**Q: What is IGMP snooping, and why might a stream stop after a few minutes?**
A: A switch reads IGMP Reports and Leaves, and forwards a group only to ports that want it and to router ports. Entries depend on Queries. If the VLAN has no querier, the entries age out, the switch stops sending to those ports, and the stream dies. I would fix this with a multicast router on the VLAN or a snooping querier.

**Q: Explain the RPF check.**
A: When a multicast packet arrives, the router looks up its source in the unicast table. If the packet came in on the interface the router would use to reach that source, it passes and is forwarded. If not, it is dropped. This prevents loops, since there is no single destination to stop at. For a shared tree the check uses the RP address.

**Q: Compare PIM dense mode and sparse mode.**
A: Dense mode is push. It floods everywhere and prunes branches with no receivers, and repeats about every 3 minutes. It needs no RP, but it wastes bandwidth and state. Sparse mode is pull. Nothing flows until a router joins, and it uses an RP so sources and receivers can find each other. Sparse mode is the normal choice today.

**Q: What does the RP do, and what happens if it fails?**
A: The RP is the meeting point for ASM. It is the root of the shared tree, it receives Registers from first hop routers, and it joins toward sources. If it fails, existing source-tree traffic keeps flowing, but new sources and new receivers cannot meet. I would avoid that with Anycast RP, using MSDP or PIM to sync the RPs.

**Q: Walk through how PIM sparse mode delivers the first packets to a receiver.**
A: The receiver sends an IGMP Report, and the last hop router sends a (*,G) Join toward the RP. The source sends, and the first hop router wraps packets in unicast Registers to the RP. The RP unwraps them, sends them down the shared tree, and joins toward the source with an (S,G) Join. Native traffic arrives, and the RP sends Register-Stop. The last hop router then joins the source directly and prunes that source from the shared tree.

**Q: Shared tree or source tree: what is the difference and when does a router switch?**
A: A shared tree is (*,G), rooted at the RP. It uses less state, but paths can be longer. A source tree is (S,G), rooted at the source. It gives the shortest path but needs one entry per source. Cisco IOS moves to the source tree after the first packet by default. `ip pim spt-threshold infinity` keeps it on the shared tree.

**Q: What is SSM and why is it preferred for one-to-many?**
A: SSM means the receiver joins a specific (S,G) channel with IGMPv3, in 232.0.0.0/8 by default. There is no RP and no shared tree. It is simpler, and only the chosen source can reach the receivers. The trade-off is that the app must learn the source address some other way, such as from a channel guide.

**Q: Receivers on one VLAN get no stream, but the source is sending. How do you troubleshoot?**
A: I start at the receiver. I check that the host joined and that the switch has its port in the snooping table. I confirm a querier exists and that the last hop router shows the group in `show ip igmp groups`. Then I check PIM neighbors, RP agreement with `show ip pim rp mapping`, and `show ip mroute`. Next is `show ip rpf` and RPF failed counters. Finally, I check that the source's TTL is high enough.

**Q: How do the IGMP querier and the PIM DR elections differ?**
A: The IGMP querier is the lowest IP on the LAN and sends Queries. The PIM DR has the highest DR priority, then the highest IP, and it sends Joins and Registers. The two roles can sit on different routers, and that is fine.

## Quick Recap

- Multicast is one copy per link, delivered only where receivers asked. It uses 224.0.0.0/4, and the source is always a unicast IP.
- MAC mapping is 01:00:5E plus the low 23 bits, so 32 group IPs share a MAC.
- IGMP is host to router. v2 adds Leave, and v3 adds source filtering. Snooping keeps frames off unused switch ports, and it needs a querier.
- PIM is router to router and uses the unicast table. The RPF check drops packets that arrive on the wrong interface for their source.
- Dense mode floods and prunes. Sparse mode joins first and uses an RP, with Registers from the source side and SPT switchover after that.
- (*,G) is the shared tree rooted at the RP. (S,G) is the source tree rooted at the source.
- SSM removes the RP: IGMPv3 plus (S,G) joins in 232.0.0.0/8. It is the preferred one-to-many design.
