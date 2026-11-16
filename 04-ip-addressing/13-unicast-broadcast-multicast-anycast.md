# Unicast, Broadcast, Multicast and Anycast

**Level:** Beginner | **Prerequisites:** [IPv4 Addressing Basics](01-ipv4-basics.md), [IPv6 Basics](11-ipv6-basics.md), [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md)

## In One Line

The destination address decides who receives a packet: one device (unicast), every device on the local network (broadcast), a group that asked for it (multicast), or the nearest of many servers that share one address (anycast).

## Why It Matters

Sending data has a cost. If a video goes to 1,000 viewers as 1,000 separate copies, the sender and the links carry 1,000 copies. If a device shouts every question to everyone, every other device must stop and look at it. The four delivery types exist to send each kind of traffic to the right receivers and nobody else.

Interviewers like this topic because it joins many areas: MAC addresses, switches, routers, IPv6, DNS and CDNs. The questions they ask most are:

- Why did IPv6 remove broadcast?
- How does a switch treat a multicast frame?
- How can two servers in different countries use the same IP address?

## Simple Explanation

Think of four ways to pass on a message.

| Type | Real-life analogy | Where the analogy stops being true |
|---|---|---|
| Unicast | A private phone call. One person calls one number. | Very little. The main difference is that a call holds a line open, while packets share links with other traffic. |
| Broadcast | A loudspeaker in one building. Everyone hears it, whether they care or not. | A person can ignore a loudspeaker with no effort. A computer must spend time reading each broadcast before it can drop it. |
| Multicast | A TV channel. It goes out once, and only people who tune in watch it. | Over the air, the signal reaches every home. A good multicast network sends traffic only toward places that have viewers. |
| Anycast | An emergency number. People in every city dial the same number and reach their local call centre. | "Local" here means the shortest path in routing terms, which is not always the closest place on a map. |

The same ideas as pictures:

```text
Unicast: one sender, one receiver

  [S] -----------> [A]     [B]     [C]     [D]

Broadcast: one sender, everyone on the local network

  [S] ---+-------> [A]
         +-------> [B]
         +-------> [C]
         +-------> [D]

Multicast: one sender, only the members of a group (A and C joined)

  [S] ---+-------> [A]  member
         |         [B]  not a member
         +-------> [C]  member
                   [D]  not a member

Anycast: one sender, the nearest server that has the shared address

  [S] -----------> [X 203.0.113.53]  nearest, receives the packet
                   [Y 203.0.113.53]  farther, receives nothing
                   [Z 203.0.113.53]  farther, receives nothing
```

## How It Works

Two addresses matter when a packet travels on a LAN (a local network):

- The **MAC address** is the hardware address used at Layer 2, inside an Ethernet **frame**. A frame is the unit of data on one local link. A **switch** reads this address.
- The **IP address** is the logical address used at Layer 3, inside a **packet**. A **router** reads this address.

Each delivery type has its own look at each layer.

### Unicast

Unicast is one sender to one receiver. Web browsing, SSH and email all use it. Almost all traffic on most networks is unicast.

- **Layer 3:** the destination is one normal address, such as 192.168.1.20 or 2001:db8:1::20.
- **Layer 2:** the destination MAC is one device, such as 00:1a:2b:3c:4d:5e. If the packet must leave the LAN, this is the MAC of the [default gateway](08-default-gateway.md), not the final server.
- **Switch:** it looks up the MAC in its table and sends the frame out one port. If the MAC is not in the table, the switch sends the frame out all ports in that VLAN except the one it came in on. This is called **unknown unicast flooding**, and a [VLAN](../03-ethernet-and-switching/06-vlans.md) is a separate logical LAN on a switch.

### Broadcast

Broadcast is one sender to every device in the same **broadcast domain**. A broadcast domain is the set of devices that receive each other's broadcasts. Normally this is one subnet or one VLAN. [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md) covers it fully.

IPv4 has two kinds of broadcast address:

| Kind | Example | Meaning |
|---|---|---|
| Limited broadcast | 255.255.255.255 | Everyone on my own local network. Routers never forward it. |
| Directed broadcast | 192.168.1.255 for 192.168.1.0/24 | Everyone on that subnet. It is the last address of the subnet. |

- **Layer 2:** the destination MAC is always ff:ff:ff:ff:ff:ff (all bits set to 1).
- **Switch:** it floods the frame out all ports in the VLAN except the one it came in on.
- **Router:** it stops the broadcast. This is what keeps broadcasts local.

Two common users of broadcast:

- [ARP](../03-ethernet-and-switching/05-arp.md) requests: "Who has 192.168.1.1? Tell 192.168.1.20." ARP finds the MAC address that belongs to an IP address.
- [DHCP](../07-application-protocols/04-dhcp.md) Discover: a new device has no IP address yet, so it sends from 0.0.0.0 to 255.255.255.255. DHCP is the protocol that gives devices an IP address automatically.

Fuller truth: a /31 point-to-point link has no broadcast address, because both of its addresses are used for the two ends. A /32 is a single address and also has none.

### Multicast

Multicast is one sender to a group of receivers who asked to join. The sender sends one copy. The network makes copies only where the path splits.

```text
Video source 10.1.1.5 sends ONE stream to group 239.1.1.1
                |
            [ Router ]
                |
            [ Switch ]    IGMP snooping is on
            /    |    \
         [A]    [B]    [C]
       joined  did not  joined
                join
A and C get the stream. B gets nothing.
```

The steps:

1. The group has its own address. In IPv4 this is 224.0.0.0/4 (224.0.0.0 to 239.255.255.255). In IPv6 it is ff00::/8.
2. A receiver joins by telling its local router, using **IGMP** (Internet Group Management Protocol) in IPv4 or **MLD** (Multicast Listener Discovery) in IPv6.
3. The source sends to the group address. Its own source address stays a normal unicast address.
4. Switches with **IGMP snooping** listen to those join messages and send the group's traffic only to ports with members. Without snooping, a switch floods multicast like a broadcast.
5. To cross routers, the network needs multicast routing. This is usually PIM, and it is covered in [Multicast Routing](../05-routing/14-multicast-routing.md).

Common multicast addresses to remember:

| IPv4 | IPv6 | Used by |
|---|---|---|
| 224.0.0.1 | ff02::1 | All hosts or all nodes on this link |
| 224.0.0.2 | ff02::2 | All routers on this link |
| 224.0.0.5 | ff02::5 | All OSPF routers |
| 224.0.0.6 | ff02::6 | OSPF designated routers |
| 224.0.0.9 | ff02::9 | RIPv2 / RIPng |
| 224.0.0.10 | ff02::a | EIGRP |
| 224.0.0.251 | ff02::fb | mDNS (printers and devices finding each other on a LAN) |

Two IPv4 ranges are also worth knowing:

- **224.0.0.0/24** is link-local. Routers do not forward it, so routing protocols use it to talk to direct neighbours.
- **239.0.0.0/8** is administratively scoped. It is for private use inside an organisation, a bit like 10.0.0.0/8 for unicast.

**Layer 2 for multicast:** the destination MAC is built from the group address.

| IP version | MAC starts with | Rest of the MAC | Example |
|---|---|---|---|
| IPv4 | 01:00:5e | Low 23 bits of the group address | 224.0.0.5 becomes 01:00:5e:00:00:05 |
| IPv6 | 33:33 | Low 32 bits of the group address | ff02::1 becomes 33:33:00:00:00:01 |

### Anycast

Anycast is one sender to the nearest of many devices that share the same address. Only one of them receives each packet.

Anycast has no special packet format. In IPv4 there is no anycast address range, and an anycast address looks exactly like unicast. The magic is in routing: the same prefix is announced from many places, and each router picks its own best path.

- **Layer 3:** the destination is a normal-looking address, such as 1.1.1.1. Nothing in the packet says "anycast".
- **Layer 2:** the destination MAC is the normal unicast MAC of the next hop. A switch cannot tell that anything special is happening.
- **IPv6:** anycast is formally defined in [RFC 4291](https://www.rfc-editor.org/rfc/rfc4291). Anycast addresses come from the unicast space and look the same, but a device must be told that an address is anycast. On Cisco IOS the keyword is `ipv6 address 2001:db8:1::1/64 anycast`.

The steps across the internet:

1. The same IP address, for example 203.0.113.53, is configured on servers at many sites.
2. Each site's routers announce the same prefix, 203.0.113.0/24, using **BGP** (Border Gateway Protocol, the routing protocol between networks on the internet).
3. Routers everywhere receive several paths to that prefix and pick one best path.
4. A client's packet follows its router's best path to one site.
5. If a site fails, it stops announcing the prefix. Routers move to the next best path, and traffic shifts to another site.

```text
    Client in Asia                  Client in Europe
          |                                |
     ISP routers                      ISP routers
   (best path: Tokyo)             (best path: Frankfurt)
          |                                |
          v                                v
 +------------------+            +------------------+
 | Site Tokyo       |            | Site Frankfurt   |
 | AS 64512         |            | AS 64512         |
 | 203.0.113.53     |            | 203.0.113.53     |
 | announces        |            | announces        |
 | 203.0.113.0/24   |            | 203.0.113.0/24   |
 +------------------+            +------------------+
```

An **AS** (Autonomous System) is one network under one organisation's control. BGP treats each AS as one step on the path. Fuller detail is in [BGP](../05-routing/09-bgp.md).

### Anycast in DNS

DNS is the system that turns names into IP addresses. It suits anycast very well, because most queries are one small UDP packet and one reply, so any server can answer.

- The root DNS system has 13 named identities, a.root-servers.net to m.root-servers.net. All of them use anycast today, so behind those 13 names sit well over a thousand physical servers around the world.
- Public resolvers such as 1.1.1.1 (Cloudflare) and 8.8.8.8 (Google) are anycast. Users everywhere type the same address and reach a nearby site.

Benefits: lower delay, no single point of failure, and a flood of attack traffic gets split across many sites instead of hitting one. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

### Anycast in CDNs

A **CDN** (Content Delivery Network) keeps copies of websites in many cities, so users download from somewhere close.

- With anycast, one IP address is announced from every CDN site. BGP brings each user to a nearby site. Cloudflare is the well-known example.
- With DNS steering, the CDN's DNS server gives different users different IP addresses. This gives more control over which site a user reaches. Many CDNs use this method, and some mix both.

Anycast and TCP need care. TCP keeps a connection with one server. If routing changes in the middle, packets may land at a different site that knows nothing about the connection, and the connection breaks. In practice, routes are stable enough that CDNs serve TCP over anycast at huge scale. See [Content Delivery Networks](../10-enterprise-and-datacenter/06-cdn.md).

### Why IPv6 Removed Broadcast

IPv6 has no broadcast addresses. Its addressing standard, RFC 4291, says multicast replaces their function. The reasons:

1. **Broadcast wastes work.** Every device must receive and read every broadcast. An IPv4 ARP request for one device disturbs all the others, and on a big VLAN these add up.
2. **Multicast can do the same job, better.** A network card can filter group addresses in hardware. Frames for groups the device has not joined are dropped before the main CPU has to do anything.
3. **Almost nothing truly needs "everyone".** Most broadcasts are looking for one device or one service. Each of those can have its own group.
4. **When everyone really is needed, a group exists.** ff02::1 is the all-nodes group. It is used for things like router advertisements, not for finding one host.

The clearest example is finding a neighbour's MAC address. IPv4 uses ARP, and IPv6 uses **NDP** (Neighbor Discovery Protocol):

```text
IPv4: find the MAC of 192.168.1.1
  ARP request to ff:ff:ff:ff:ff:ff
  Every device on the VLAN must read it

IPv6: find the MAC of 2001:db8:10::abcd:1234
  Neighbor Solicitation to ff02::1:ffcd:1234
  Frame sent to 33:33:ff:cd:12:34
  Only devices with an address ending cd:1234 are listening
```

That group is called the **solicited-node multicast address**. It is ff02::1:ff followed by the last 24 bits of the address. Every IPv6 device joins it for each of its addresses.

Fuller truth: a switch without MLD snooping still floods this frame out all ports. The saving then comes from the receiving network cards, which drop it in hardware. With MLD snooping, the switch saves the bandwidth too.

### Comparison Table

| | Unicast | Broadcast | Multicast | Anycast |
|---|---|---|---|---|
| Pattern | One to one | One to all on the LAN | One to a joined group | One to nearest of many |
| Who receives | One device | Whole broadcast domain | Members only | One device, chosen by routing |
| IPv4 example | 192.168.1.20 | 255.255.255.255, 192.168.1.255 | 239.1.1.1 | 8.8.8.8 |
| IPv6 example | 2001:db8:1::20 | Does not exist | ff02::1 | 2001:4860:4860::8888 |
| Destination MAC | One device | ff:ff:ff:ff:ff:ff | 01:00:5e:... or 33:33:... | Next hop, normal unicast |
| Switch action | One port, or flood if unknown | Flood in VLAN | Flood, or member ports with snooping | Same as unicast |
| Crosses routers | Yes | No (see Going Deeper) | Only with multicast routing | Yes |
| Transport | TCP or UDP | UDP | Almost always UDP | UDP, and TCP with care |
| Typical use | Web, SSH, email | ARP, DHCP Discover | OSPF, IPTV, NDP | DNS, CDN |

## Example

### Example 1: Calculate a multicast MAC address

**Task:** find the destination MAC for IPv4 group 239.1.1.1.

1. Write the group in binary.
2. Keep only the low 23 bits. This means dropping the first octet and the top bit of the second octet.
3. Put those 23 bits after 01:00:5e and a 0 bit.

```text
Group:    239       1          1         1
Binary:   11101111  0 0000001  00000001  00000001
          dropped   ^ kept     kept      kept
                    dropped

Kept 23 bits:         0000001  00000001  00000001
Add a 0 bit in front: 00000001 00000001  00000001
In hex:                  01       01        01

Result:   01:00:5e:01:01:01
```

Why is the top bit of the second octet dropped? Only 23 bits are free in the MAC, but a group has 28 bits that can change. So 5 bits are lost, and **32 different IP groups share each MAC**.

For example, 224.1.1.1, 225.1.1.1 and 239.129.1.1 all map to 01:00:5e:01:01:01. A device that joined one of them may receive frames for the others on the same MAC. Its IP layer then checks the real group address and drops what it did not ask for.

**IPv6 version:** ff02::1:ffcd:1234 keeps its last 32 bits (ffcd:1234) after 33:33, which gives 33:33:ff:cd:12:34. Groups with the same last 32 bits would share a MAC, but this is rare.

### Example 2: See delivery types in a capture

**Setup (Linux host on a LAN):** the host is 192.168.1.20/24 with MAC 00:1a:2b:3c:4d:5e. The router is 192.168.1.1/24 with MAC 00:aa:bb:cc:dd:01.

`tcpdump` is a tool that shows packets on an interface. The `-e` flag prints MAC addresses and `-n` keeps addresses as numbers.

```bash
# Linux: show ARP and OSPF traffic with MAC addresses
sudo tcpdump -eni eth0 -c 3 'arp or ip proto 89'
```

```text
10:15:02.114233 00:1a:2b:3c:4d:5e > ff:ff:ff:ff:ff:ff, ethertype ARP
  (0x0806), length 42: Request who-has 192.168.1.1 tell 192.168.1.20,
  length 28
10:15:02.114871 00:aa:bb:cc:dd:01 > 00:1a:2b:3c:4d:5e, ethertype ARP
  (0x0806), length 60: Reply 192.168.1.1 is-at 00:aa:bb:cc:dd:01,
  length 46
10:15:05.201144 00:aa:bb:cc:dd:01 > 01:00:5e:00:00:05, ethertype IPv4
  (0x0800), length 78: 192.168.1.1 > 224.0.0.5: OSPFv2, Hello,
  length 44
```

(Lines are wrapped here to fit. Real output prints each packet on one line.)

Reading it:

| Packet | Destination MAC | Type |
|---|---|---|
| ARP request | ff:ff:ff:ff:ff:ff | Broadcast |
| ARP reply | 00:1a:2b:3c:4d:5e | Unicast |
| OSPF Hello | 01:00:5e:00:00:05 | Multicast (224.0.0.5 mapped to MAC) |

### Example 3: See your multicast groups

```bash
# Linux: list group memberships on eth0 (output is trimmed)
ip maddr show dev eth0
```

```text
2:  eth0
    link  33:33:00:00:00:01
    link  01:00:5e:00:00:01
    link  33:33:ff:cd:12:34
    inet  224.0.0.1
    inet6 ff02::1:ffcd:1234
    inet6 ff02::1
```

This host has an IPv6 address ending in cd:1234, so it joined the matching solicited-node group. Each group has its matching MAC entry.

```powershell
# Windows: list IPv4 groups (use ipv6 in place of ipv4 for IPv6)
netsh interface ipv4 show joins
```

```text
Interface 12: Ethernet

Scope       References  Last  Address
----------  ----------  ----  ---------------------------------
0                    2  Yes   224.0.0.1
0                    1  Yes   224.0.0.251
0                    1  Yes   239.255.255.250
```

### Example 4: See anycast from your own machine

Cloudflare's resolver answers a special query that names the site you reached. Run it from two distant places. The IP address is the same, but the site name is different.

```bash
# Linux or macOS
dig +short @1.1.1.1 id.server CH TXT
```

```powershell
# Windows
nslookup -class=chaos -type=txt id.server 1.1.1.1
```

```text
"fra01"
```

The name depends on where you are, and the format can change over time. A user in Karachi may see a different site code from a user in Frankfurt, while both use 1.1.1.1.

## Going Deeper

**How a router sees anycast.** This Cisco IOS output shows one router's choice between two paths to the same prefix. The `>` marks the best path. The shorter AS path wins here, so this router sends traffic toward that site. Other routers may choose a different site.

```text
R1# show ip bgp
     Network          Next Hop        Metric LocPrf Weight Path
 *   203.0.113.0/24   198.51.100.9                       0 64520 64512 i
 *>                   198.51.100.1                       0 64512 i
```

**Anycast inside a site.** A site often has many servers behind one address. The router spreads flows across them using ECMP (Equal-Cost Multi-Path), which hashes each flow's addresses and ports so that one flow stays on one server. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

**Other anycast uses.** Google Cloud global load balancers and AWS Global Accelerator give customers anycast IP addresses. Data centre fabrics use a [distributed anycast gateway](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md), where every leaf switch has the same gateway IP and MAC. IPv6 also defines a **Subnet-Router anycast address**, which is the subnet prefix with all zeros after it. For 2001:db8:1:1::/64 that is 2001:db8:1:1::, and routers on that subnet answer it.

**Directed broadcast and routers.** Old routers could forward a directed broadcast from a remote network and release it on the target subnet. The Smurf attack abused this to flood victims, so RFC 2644 changed the default to off. On Cisco IOS the command is `ip directed-broadcast`, and it is off by default in modern releases. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

**Getting DHCP across a router.** Routers stop the DHCP broadcast, so a remote DHCP server would never see it. A **DHCP relay** fixes this by turning the broadcast into a unicast to the server:

```text
! Cisco IOS: on the LAN interface that faces the clients
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 ip helper-address 10.0.0.10
```

**Wi-Fi.** Broadcast and multicast are costly on Wi-Fi. These frames are normally sent at a low data rate and are not acknowledged, so they use a lot of air time and can be lost.

## Common Mistakes

- **Thinking a router forwards broadcasts.** A router is the edge of a broadcast domain. Limited broadcasts never cross it, and directed broadcast forwarding is off by default.
- **Thinking anycast is a special packet type.** The packet looks like unicast. The behaviour comes from routing.
- **Thinking "nearest" in anycast means closest on a map.** It means the best path by BGP policy and routing, which can be far from the shortest distance.
- **Thinking anycast delivers to all the servers.** Each packet goes to one. Sending to many is multicast.
- **Saying IPv6 has no way to reach everyone.** ff02::1 reaches all nodes on a link. What IPv6 removed is the broadcast address type.
- **Thinking multicast always saves bandwidth on a switch.** Without IGMP or MLD snooping, a switch floods it like a broadcast.
- **Using a multicast address as a source.** A source address is always that of one sender. Group addresses are for destinations only.
- **Assuming each IPv4 group has its own MAC.** 32 groups share each MAC address.
- **Using `ping 192.168.1.255` to find all hosts.** Most modern hosts ignore pings to a broadcast address.
- **Mixing up flooding and broadcast.** Broadcast is a type of address. Flooding is what a switch does, and it may flood unknown unicast too.

## Interview Questions

**Q: What are the four delivery types?**
A: Unicast is one to one. Broadcast is one to everyone on the local network. Multicast is one to a group that joined. Anycast is one to the nearest of several devices that share one address.

**Q: Why did IPv6 remove broadcast?**
A: Broadcast makes every device on the link read the packet, even when it has nothing to do with them. IPv6 uses multicast instead, so only interested devices listen. For example, Neighbor Solicitation goes to a solicited-node group rather than to all devices like ARP does. If everyone must be reached, the all-nodes group ff02::1 does that.

**Q: How does a switch handle unicast, broadcast and multicast frames?**
A: For a known unicast MAC, it forwards out one port. For an unknown unicast MAC or a broadcast, it floods all ports in the VLAN except the incoming port. It floods multicast the same way, unless IGMP or MLD snooping is on, in which case it sends only to member ports.

**Q: What is the MAC address for IPv4 group 224.0.0.5?**
A: 01:00:5e:00:00:05. An IPv4 multicast MAC starts with 01:00:5e and the low 23 bits of the group address fill the rest.

**Q: Why can two IPv4 multicast groups end up with the same MAC address?**
A: A group address has 28 variable bits, but the MAC carries only 23 of them. The other 5 bits are lost, so 32 groups share each MAC. The host checks the real group address at the IP layer.

**Q: How can servers in different countries use the same IP address?**
A: That is anycast. Each site announces the same prefix with BGP. Every router on the internet picks its own best path, so each client reaches one site, usually a nearby one. If a site fails, its announcement is withdrawn and traffic moves to the next best site.

**Q: Why is anycast such a good fit for DNS?**
A: Most DNS queries are one UDP packet and one reply, so there is no long connection to keep on one server. Users get a nearby site, one failed site does not stop the service, and attack traffic gets spread across many sites.

**Q: Does anycast work with TCP?**
A: Yes, but with a risk. If routing changes during a connection, packets can reach a different site that has no record of the connection, and it breaks. Routes are usually stable, so CDNs run TCP on anycast at large scale.

**Q: Why does multicast normally use UDP and not TCP?**
A: TCP is a connection between exactly two endpoints, with acknowledgements from one receiver. That model does not fit one sender and many receivers. UDP has no connection, so one packet can go to a whole group.

**Q: How does a DHCP client get an address if routers block broadcasts?**
A: If the server is on the same LAN, the broadcast reaches it directly. If not, a DHCP relay on the router, set with `ip helper-address` on Cisco, receives the broadcast and forwards it as unicast to the server.

## Quick Recap

- Unicast reaches one device, broadcast reaches a whole LAN, multicast reaches joined members, and anycast reaches the nearest of many.
- Broadcast MAC is ff:ff:ff:ff:ff:ff. IPv4 multicast MACs start with 01:00:5e, and IPv6 ones start with 33:33.
- Anycast has no special look at either layer. It is one prefix announced from many places by BGP.
- Routers stop broadcasts. Multicast crosses routers only with multicast routing.
- Without IGMP or MLD snooping, a switch floods multicast.
- IPv6 has no broadcast. It uses groups such as ff02::1 and solicited-node addresses.
- DNS and CDNs use anycast for low delay, failover and absorbing attacks.
