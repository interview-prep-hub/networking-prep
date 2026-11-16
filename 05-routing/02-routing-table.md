# The Routing Table

**Level:** Intermediate | **Prerequisites:** [How Routers Work](01-how-routers-work.md), [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md)

## In One Line

A routing table is the list of destinations a device knows, with the next step for each one. For every packet, the device picks the most specific matching entry and sends the packet that way.

## Why It Matters

A router receives a packet for 192.168.2.5. It has several ports and must choose one, millions of times per second. The routing table gives the answer.

Routers are not the only devices with this table. Your Linux server, Windows laptop, phone and containers all have one. A laptop's table is small, but it follows the same logic.

Many network tickets come down to the routing table:

- There is no route to the destination.
- A route exists, but the wrong one is being used.
- The route out is fine, but the return route is missing on another device.

Interviewers ask about this topic because it shows if you understand how forwarding really works. A classic question: "A router has an OSPF /24 and a RIP /25 for the same address. Which one is used?" People who mix up longest prefix match and administrative distance get this wrong. This chapter makes sure you do not.

## Simple Explanation

Think of a sorting room in a post office. On the wall is a list:

```text
City A, Green Street    -> put in bag 7
City A                  -> put in bag 3
Anything else           -> put in bag 1 (to the main hub)
```

A letter for "City A, Green Street, house 9" matches all three lines. The sorter uses the most detailed line, so it goes in bag 7. This is **longest prefix match**. The "anything else" line is the **default route**.

The sorter does not carry the letter to the door. The sorter only chooses the next bag. The next post office has its own list and repeats the job. Routers work the same way, one hop at a time.

Who wrote the lines on the wall? Three kinds of source:

- The sorter's own building. It knows its own streets for sure. These are **connected routes**.
- A manager wrote a line by hand. These are **static routes**.
- Other post offices sent updates. These are **dynamic routes**.

If two sources disagree about the exact same place, the sorter believes the more trusted source. That trust ranking is **administrative distance**. If one source offers two ways, the sorter takes the shorter one. That distance is the **metric**.

**Where the analogy stops.** Postal addresses split at names like city and street. IP prefixes split at any bit, and any length from /0 to /32 is possible. Also, a real table changes by itself within seconds when a link fails.

Key words, in plain language:

| Term | Meaning |
|---|---|
| Route | One line in the table |
| Prefix | A network address plus its length, like 192.168.2.0/24 |
| Next hop | The IP address of the next router on the path |
| Exit interface | The local port the packet leaves from |
| Administrative distance (AD) | How much the router trusts the source of a route. Lower is better |
| Metric | The cost of a route, inside one routing protocol. Lower is better |
| Default route | 0.0.0.0/0, the route that matches everything |

## How It Works

### What One Route Contains

Every route answers two questions: "Which destinations does this cover?" and "Where do I send them?" Most systems also record who supplied the route and how good it is.

We will use this small network in most examples:

```text
                     ISP router 203.0.113.1
                            |
                            | G0/2 203.0.113.2/30
                            |
 LAN 192.168.1.0/24  G0/0 +-+--+ G0/1          G0/1 +----+
 PC1 .50 -----------------| R1 |--------------------| R2 |---- LAN
 Windows PC .60       .1  +----+ 10.0.12.1/30       +----+
                                      10.0.12.2/30
                                                  192.168.2.0/24
                                                  172.16.0.0/16
```

G0/0 is short for GigabitEthernet0/0, one port on the router. R1 learns 192.168.2.0/24 from R2 through OSPF, a routing protocol. An administrator typed a static route for 172.16.0.0/16 and a static default route to the ISP.

### Reading the Table on Cisco IOS

```text
R1# show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile
       B - BGP, D - EIGRP, EX - EIGRP external, O - OSPF
       IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1
       L2 - IS-IS level-2, ia - IS-IS inter area, * - candidate default
       U - per-user static, o - ODR
       P - periodic downloaded static route, + - replicated route

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.12.0/30 is directly connected, GigabitEthernet0/1
L        10.0.12.1/32 is directly connected, GigabitEthernet0/1
S     172.16.0.0/16 [1/0] via 10.0.12.2
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
O     192.168.2.0/24 [110/2] via 10.0.12.2, 00:14:32, GigabitEthernet0/1
      203.0.113.0/24 is variably subnetted, 2 subnets, 2 masks
C        203.0.113.0/30 is directly connected, GigabitEthernet0/2
L        203.0.113.2/32 is directly connected, GigabitEthernet0/2
```

The exact legend changes a little between IOS versions. The route lines keep the same form.

Here is one route line, taken apart:

```text
O     192.168.2.0/24 [110/2] via 10.0.12.2, 00:14:32, GigabitEthernet0/1
|     |               |   |      |          |         |
|     |               |   |      |          |         exit interface
|     |               |   |      |          age of the route
|     |               |   |      next hop IP address
|     |               |   metric (OSPF cost)
|     |               administrative distance
|     destination prefix
route source (O = OSPF)
```

Three more parts of the output need explaining:

- **"Gateway of last resort"** is the router's name for its chosen default route. "Gateway of last resort is not set" means there is no default route.
- **Lines with no code on the left**, like "10.0.0.0/8 is variably subnetted", are headers. They group routes for reading. Nothing is forwarded using them. "Variably subnetted" only means the group holds different mask lengths.
- **"S\*"** means a static route that is also a candidate default. A candidate default is a route the router may use as its default route.

Useful filters on Cisco:

```text
R1# show ip route 192.168.2.5             ! details for one address
R1# show ip route ospf                    ! only OSPF routes
R1# show ip route connected               ! only connected routes
R1# show ip route summary                 ! counts per source
R1# show ip route 10.0.0.0 255.0.0.0 longer-prefixes
```

The last command lists every route inside 10.0.0.0/8. Here is the detail view for one address:

```text
R1# show ip route 192.168.2.5
Routing entry for 192.168.2.0/24
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 10.0.12.2 on GigabitEthernet0/1, 00:14:32 ago
  Routing Descriptor Blocks:
  * 10.0.12.2, from 10.255.0.2, 00:14:32 ago, via GigabitEthernet0/1
      Route metric is 2, traffic share count is 1
```

Here 10.255.0.2 is R2's OSPF router ID, a number that names a router inside OSPF. It is not the next hop.

### Route Sources and Codes

These are the codes you will meet most often on Cisco. AD is explained below.

| Code | Source | Default AD |
|---|---|---|
| L | Local: the router's own interface address | 0 |
| C | Connected: a subnet on an active interface | 0 |
| S | Static: typed by an administrator | 1 |
| D | EIGRP internal | 90 |
| D EX | EIGRP external (brought in from outside EIGRP) | 170 |
| O | OSPF, same area | 110 |
| O IA | OSPF, from another area | 110 |
| O E1, O E2 | OSPF external (brought in from outside OSPF) | 110 |
| O N1, O N2 | OSPF external, through a special area type called NSSA | 110 |
| i L1, i L2, i ia | IS-IS level 1, level 2, inter-area | 115 |
| R | RIP | 120 |
| B | BGP | 20 external, 200 internal |
| \* | Marker: candidate default route | Not a source |
| + | Marker: route copied from another routing table | Not a source |

Each protocol has its own chapter: [RIP](05-rip.md), [OSPF](06-ospf.md), [EIGRP](07-eigrp.md), [IS-IS](08-is-is.md) and [BGP](09-bgp.md).

### Connected and Local Routes

A **connected route** is the subnet an interface sits on. A **local route** is the single address of the interface itself.

Type this:

```text
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
```

When the interface comes up, two routes appear with no routing protocol at all:

```text
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
```

The two routes do different jobs:

| Route | Plain meaning | What the router does |
|---|---|---|
| C 192.168.1.0/24 | "This whole subnet is on that port" | Finds the host's MAC address with ARP and sends the frame straight to it |
| L 192.168.1.1/32 | "This one address is me" | Keeps the packet and hands it to the router's own CPU, as with an SSH session or a ping to the router |

ARP is the protocol that finds a MAC address from an IP address. See [ARP](../03-ethernet-and-switching/05-arp.md).

Facts worth remembering:

- Connected routes appear only when the interface is up and has an IP address. On Cisco, that is status up and protocol up. If the interface goes down, both routes vanish.
- Each secondary IP address on an interface adds its own connected and local route.
- IPv4 local routes appear on IOS 15 and later. Older IOS did not list them in `show ip route`. IPv6 tables have shown them for longer.
- A loopback with a /32 mask shows only a `C` line. There is no separate `L` line, because the two would be the same route.

### Reading the Table on Linux

Linux uses the `ip` command from the iproute2 package:

```bash
# Linux
$ ip route show
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.50 metric 100
10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.2
172.16.50.0/24 via 192.168.1.254 dev eth0 proto static metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50 metric 100
```

Here 192.168.1.254 is a second router on the LAN, and tun0 is a VPN interface.

| Field | Meaning |
|---|---|
| `default` | The same as 0.0.0.0/0 |
| `via 192.168.1.1` | Next hop. No `via` means the network is directly connected |
| `dev eth0` | Exit interface |
| `proto` | Who added the route: `kernel`, `dhcp`, `static`, `ospf`, `bgp` and others |
| `scope link` | The destination is on a directly attached link |
| `src` | The source IP the host prefers for packets it creates |
| `metric` | Cost. Lower wins |

To ask "which route would be used for this address?", use `ip route get`. It runs a real lookup:

```bash
# Linux
$ ip route get 198.51.100.7
198.51.100.7 via 192.168.1.1 dev eth0 src 192.168.1.50 uid 1000
    cache
```

Linux has more than one table. What you saw above is the `main` table. Local addresses live in a table called `local`:

```bash
# Linux
$ ip route show table local
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
local 192.168.1.50 dev eth0 proto kernel scope host src 192.168.1.50
broadcast 192.168.1.255 dev eth0 proto kernel scope link src 192.168.1.50

$ ip rule show
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

The rules are checked in order, so `local` comes first. This is the Linux version of Cisco's `L` routes.

Older tools still work on many systems. `route -n` and `netstat -rn` come from the old net-tools package:

```bash
# Linux (legacy tools)
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

Flag `U` means the route is up. `G` means it goes through a gateway. A gateway of 0.0.0.0 means directly connected.

Linux has no administrative distance. The kernel chooses by prefix length, then by metric. If you run routing software such as FRR on Linux, that software applies AD first and then installs the winner into the kernel. See [Linux Networking](../11-cloud-and-containers/01-linux-networking.md).

### Reading the Table on Windows

```powershell
# Windows
C:\> route print -4
===========================================================================
Interface List
 12...00 15 5d 01 02 03 ......Intel(R) Ethernet Connection
  1...........................Software Loopback Interface 1
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0      192.168.1.1     192.168.1.60     25
        127.0.0.0        255.0.0.0         On-link         127.0.0.1    331
        127.0.0.1  255.255.255.255         On-link         127.0.0.1    331
  127.255.255.255  255.255.255.255         On-link         127.0.0.1    331
      192.168.1.0    255.255.255.0         On-link      192.168.1.60    281
     192.168.1.60  255.255.255.255         On-link      192.168.1.60    281
    192.168.1.255  255.255.255.255         On-link      192.168.1.60    281
        224.0.0.0        240.0.0.0         On-link         127.0.0.1    331
        224.0.0.0        240.0.0.0         On-link      192.168.1.60    281
  255.255.255.255  255.255.255.255         On-link         127.0.0.1    331
  255.255.255.255  255.255.255.255         On-link      192.168.1.60    281
===========================================================================
Persistent Routes:
  None
```

How to read it:

- **Gateway "On-link"** means directly connected. No next hop is needed.
- **Interface** shows the local IP address of the interface, not its name.
- **Persistent Routes** are static routes that survive a reboot.
- The 192.168.1.60 /32 line is the Windows version of a local route.
- 224.0.0.0/4 covers multicast, and 255.255.255.255 is the local broadcast.

PowerShell gives the same data as objects. This is easier to filter:

```powershell
# Windows
PS C:\> Get-NetRoute -AddressFamily IPv4 -InterfaceAlias Ethernet

ifIndex DestinationPrefix   NextHop      RouteMetric ifMetric PolicyStore
------- -----------------   -------      ----------- -------- -----------
12      255.255.255.255/32  0.0.0.0      256         25       ActiveStore
12      224.0.0.0/4         0.0.0.0      256         25       ActiveStore
12      192.168.1.255/32    0.0.0.0      256         25       ActiveStore
12      192.168.1.60/32     0.0.0.0      256         25       ActiveStore
12      192.168.1.0/24      0.0.0.0      256         25       ActiveStore
12      0.0.0.0/0           192.168.1.1  0           25       ActiveStore

PS C:\> Find-NetRoute -RemoteIPAddress 198.51.100.7
```

`Find-NetRoute` does a lookup, like `ip route get` on Linux. It shows the source address and the route Windows would choose.

Windows adds two numbers to get the real cost:

```text
Effective metric = route metric + interface metric

Default route:    0 + 25 =  25
Connected subnet: 256 + 25 = 281
```

With automatic metric on, Windows sets the interface metric from the link speed. Faster links get lower numbers. A 1 Gbps link commonly gets 25. Windows also has no administrative distance.

### Same Idea, Three Systems

| Idea | Cisco IOS | Linux | Windows |
|---|---|---|---|
| Show the table | `show ip route` | `ip route show` | `route print -4` or `Get-NetRoute` |
| Look up one address | `show ip cef 192.168.2.5` | `ip route get 192.168.2.5` | `Find-NetRoute -RemoteIPAddress 192.168.2.5` |
| Default route | `S* 0.0.0.0/0` | `default via ...` | `0.0.0.0  0.0.0.0` |
| Connected subnet | `C` | `proto kernel scope link` | Gateway `On-link` |
| Own address | `L` | `local` in table `local` | /32 line, `On-link` |
| Who added it | Code letter | `proto` field | `Protocol` property of `Get-NetRoute` |
| Administrative distance | Yes | No (kernel level) | No |

### How the Table Is Built

The table is the result of a contest. Many sources offer routes, and the best offer for each prefix is installed.

```text
 Interfaces    Static      OSPF         EIGRP        BGP
     |         config        |            |           |
     |           |      own database  own topology  own table
     |           |      best path     best path     best path
     |           |           |            |           |
     v           v           v            v           v
 +-----------------------------------------------------------+
 | RIB manager: for each exact prefix, lowest AD wins        |
 +-----------------------------------------------------------+
                             |
                             v
               Routing table (RIB): the winners
                             |
                             v
            Forwarding table (FIB): used for packets
```

RIB means Routing Information Base, the routing table. FIB means Forwarding Information Base, a faster copy built for forwarding. More on this in Going Deeper.

Step by step:

1. **Interfaces come up.** Each active interface with an IP address adds its connected and local routes.
2. **Static routes are checked.** A static route is added only if it can be used. Its next hop must be reachable, or its exit interface must be up.
3. **Protocols learn routes.** Each routing protocol talks to neighbours and keeps its own database. It picks its own best path to each prefix, using its metric.
4. **Protocols offer routes.** Each protocol offers its best path to the routing table.
5. **The router compares offers.** If two sources offer the exact same prefix and length, the lower AD wins. The loser stays in its own protocol's database, as a backup.
6. **Next hops are resolved.** The router works out which interface reaches each next hop.
7. **The FIB is updated.** The winners are copied to the forwarding table.
8. **The table keeps changing.** If a link fails, its routes are removed. A backup from a protocol's database can then take the empty place.

**Key point.** `show ip route` shows only winners. A route that lost the contest is not there, but the protocol may still know it.

### Administrative Distance

**Administrative distance (AD)** is a trust score for the source of a route. Lower is more trusted. It decides between two sources offering the same prefix.

Why is it needed? OSPF says cost 20. RIP says 2 hops. Those are different units, so they cannot be compared. AD gives a common scale.

Analogy: you ask two friends how far the airport is. One says "20 kilometres". The other says "2 traffic lights". You cannot compare the numbers, so you believe the friend you trust more. The analogy stops here: a person might mix both answers. A router never does. It uses only the winning route.

Cisco IOS defaults:

| Route source | Default AD |
|---|---|
| Connected interface | 0 |
| Static route | 1 |
| EIGRP summary route | 5 |
| External BGP (eBGP) | 20 |
| Internal EIGRP | 90 |
| IGRP (old, removed from modern IOS) | 100 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| External EIGRP | 170 |
| Internal BGP (iBGP) | 200 |
| Default route learned by DHCP on an IOS interface | 254 |
| Unknown, never trusted | 255 |

A route with AD 255 is never installed.

AD is not part of any protocol standard. It is a local number on each router and is never sent to neighbours. Vendors choose their own values. Juniper Junos calls it "route preference": its defaults are static 5, OSPF internal 10, RIP 100 and BGP 170, for both internal and external BGP.

You can change AD. A common use is the **floating static route**: a backup static route with a high AD, so it enters the table only when the dynamic route disappears.

```text
! Cisco IOS: backup default route, used only if OSPF loses its default
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1 130
```

Details are in [Static and Default Routes](03-static-and-default-routes.md).

### Metric

The **metric** is a protocol's cost for a path. Lower is better. A protocol uses it to choose among its own paths.

| Source | Metric is based on | Notes |
|---|---|---|
| RIP | Hop count | Maximum usable is 15. 16 means unreachable |
| OSPF | Cost, from interface bandwidth | Cisco default: 100 Mbps divided by interface speed, minimum 1 |
| EIGRP | Composite number | By default uses bandwidth and delay |
| IS-IS | Cost per interface | Cisco default is 10 per interface |
| BGP | No single metric | Uses a list of path attributes. The MED attribute appears in the metric slot |
| Static, connected | Not used | Static shows `[1/0]` |
| Linux, Windows host | A plain number | Used to prefer one local route over another |

Two warnings:

- Never compare metrics from different protocols. AD does that job.
- With OSPF on Cisco, the default reference of 100 Mbps gives 100 Mbps, 1 Gbps and 10 Gbps links the same cost of 1. See [OSPF](06-ospf.md).

If two paths from one protocol have the same metric, the router can install both and share traffic. This is **equal-cost multipath**, covered in Going Deeper.

On a host, the metric picks between interfaces. A Linux laptop on both cable and Wi-Fi often has:

```bash
# Linux, with NetworkManager defaults
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.50 metric 100
default via 192.168.1.1 dev wlan0 proto dhcp src 192.168.1.51 metric 600
```

The cable wins because 100 is lower than 600. NetworkManager sets those numbers. Other network tools may use other values.

### Longest Prefix Match

**Longest prefix match** is the rule for choosing a route for a packet. Of all routes that contain the destination address, use the one with the longest prefix. The longest prefix is the highest number after the slash.

Why? A longer prefix covers fewer addresses, so it is more specific. A /32 is one host. A /0 is the whole Internet.

The steps for each packet:

1. Read the destination IP address from the packet. The source address plays no part.
2. Find every route that contains that address.
3. Choose the route with the longest prefix.
4. If several routes share that prefix, use them all with equal-cost multipath.
5. If no route matches, drop the packet.

To test if an address fits a route, apply the mask to the address, then compare the result with the route's network. Worked examples are in the Example section.

**AD and metric do not compete with longest prefix match.** They work at a different time. AD and metric choose between routes for the exact same prefix when the table is built. Longest prefix match then picks among what is in the table.

```text
Building the table (when routes change)
   Same prefix and length, from different sources: lowest AD wins
   Same prefix and length, from same source: lowest metric wins

Using the table (for every packet)
   Longest matching prefix wins
```

**If nothing matches.** With no matching route and no default route, the router drops the packet. It normally sends an ICMP "destination network unreachable" message (type 3, code 0) to the sender. See [ICMP, Ping and Traceroute](10-icmp-ping-and-traceroute.md).

*Fuller truth:* some features act before the normal lookup. Policy-based routing (rules that choose a path by source or other fields) can override it. A VRF (Virtual Routing and Forwarding, several separate tables on one router) selects a different table. Both are described in [Advanced Routing Techniques](12-advanced-routing-techniques.md).

### Next Hop vs Exit Interface

These two are often confused:

- **Next hop:** the IP address of the next router. "Give it to 10.0.12.2."
- **Exit interface:** the local port. "Send it out of G0/1."

Analogy: the exit interface is the door of your house. The next hop is the neighbour you hand the parcel to. The analogy stops here: on an Ethernet port, many neighbours can stand behind one door, so the door alone does not say who gets the parcel.

**The next hop IP address never goes in the packet.** The packet keeps its final destination IP. The router uses the next hop only to find a MAC address:

```text
Packet to 192.168.2.5 leaving R1 on G0/1

 Ethernet header (changes per hop)    IP header (same end to end)
 +------------------------------+     +-----------------------------+
 | Dest MAC = R2 G0/1 MAC       |     | Source IP = 192.168.1.50    |
 | Src MAC  = R1 G0/1 MAC       |     | Dest IP   = 192.168.2.5     |
 +------------------------------+     +-----------------------------+
   next hop 10.0.12.2 was used           10.0.12.2 does not appear
   to find this MAC with ARP             anywhere in the packet
```

Without NAT, the IP addresses stay the same across hops. The TTL drops by one. TTL is a hop counter in the IP header.

On Cisco, a static route can be written in three ways:

| Form | Command | Shown as |
|---|---|---|
| Next hop only | `ip route 172.16.0.0 255.255.0.0 10.0.12.2` | `S 172.16.0.0/16 [1/0] via 10.0.12.2` |
| Exit interface only | `ip route 172.16.0.0 255.255.0.0 G0/1` | `S 172.16.0.0/16 is directly connected, GigabitEthernet0/1` |
| Both | `ip route 172.16.0.0 255.255.0.0 G0/1 10.0.12.2` | `S 172.16.0.0/16 [1/0] via 10.0.12.2, GigabitEthernet0/1` |

The middle form looks like a connected route, but the code is still `S` and the AD is still 1.

When to use which:

- **Point-to-point link** (only one device at the far end, such as a tunnel or serial link): exit interface alone is fine.
- **Ethernet:** always give a next hop. With an exit interface alone, the router treats every destination as if it were on the wire. It sends an ARP request for each one. This works only if the next router replies on their behalf, which is called proxy ARP. It also fills the ARP table. IOS warns about this for a default route.
- **IPv6 with a link-local next hop:** give both. The same fe80:: address can exist on every link, so the router must be told which one.

**Recursive lookup.** A route may give a next hop that is not directly connected. The router then looks up the next hop itself. Suppose R1 also runs BGP:

```text
B     198.51.100.0/24 [200/0] via 10.0.0.4, 00:05:11
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O        10.0.0.4/32 [110/3] via 10.0.12.2, 00:20:44, GigabitEthernet0/1
C        10.0.12.0/30 is directly connected, GigabitEthernet0/1
L        10.0.12.1/32 is directly connected, GigabitEthernet0/1
```

A packet for 198.51.100.7 is resolved like this:

```text
Lookup 1: 198.51.100.7  -> B route -> next hop 10.0.0.4 (no interface)
Lookup 2: 10.0.0.4      -> O route -> next hop 10.0.12.2, G0/1
Lookup 3: 10.0.12.2     -> C route -> directly connected on G0/1
Result:   send out G0/1 to the MAC address of 10.0.12.2
```

This is normal for BGP, where the next hop is often many routers away. Modern routers do this work ahead of time and store the result in the FIB, so packets do not trigger three lookups.

**Failure behaviour.** A static route with only a next hop stays in the table as long as the next hop's subnet is reachable. If R2 dies but the switch between R1 and R2 stays up, R1's port stays up and the route stays. Packets vanish. Tracking tools or a routing protocol solve this. See [Static and Default Routes](03-static-and-default-routes.md).

## Example

### Example 1: Longest Prefix Match

A router has these routes:

| Entry | Prefix | Next hop | Range covered |
|---|---|---|---|
| A | 0.0.0.0/0 | 203.0.113.1 | Everything |
| B | 10.0.0.0/8 | 10.0.12.2 | 10.0.0.0 to 10.255.255.255 |
| C | 10.1.0.0/16 | 10.0.13.2 | 10.1.0.0 to 10.1.255.255 |
| D | 10.1.2.0/24 | 10.0.14.2 | 10.1.2.0 to 10.1.2.255 |
| E | 10.1.2.128/25 | 10.0.15.2 | 10.1.2.128 to 10.1.2.255 |
| F | 10.1.2.200/32 | 10.0.16.2 | Only 10.1.2.200 |

Now send some packets:

| Destination | Matching entries | Winner | Next hop |
|---|---|---|---|
| 10.1.2.200 | A, B, C, D, E, F | F (/32) | 10.0.16.2 |
| 10.1.2.129 | A, B, C, D, E | E (/25) | 10.0.15.2 |
| 10.1.2.127 | A, B, C, D | D (/24) | 10.0.14.2 |
| 10.1.2.5 | A, B, C, D | D (/24) | 10.0.14.2 |
| 10.1.9.9 | A, B, C | C (/16) | 10.0.13.2 |
| 10.200.1.1 | A, B | B (/8) | 10.0.12.2 |
| 198.51.100.7 | A | A (/0) | 203.0.113.1 |

Check entry E by hand. A /25 mask is 255.255.255.128. Only the last byte needs work.

```text
Is 10.1.2.129 inside 10.1.2.128/25?
  129 = 1000 0001
  128 = 1000 0000  (mask, last byte)
  AND = 1000 0000  = 128
  Result 10.1.2.128 equals the route's network. MATCH.

Is 10.1.2.5 inside 10.1.2.128/25?
    5 = 0000 0101
  128 = 1000 0000  (mask, last byte)
  AND = 0000 0000  = 0
  Result 10.1.2.0 does not equal 10.1.2.128. NO MATCH.
```

The same check in full binary shows what "25 bits must match" means:

```text
               <------- 25 bits ---------->
10.1.2.128/25  00001010 00000001 00000010 1 | 0000000
10.1.2.129     00001010 00000001 00000010 1 | 0000001   match
10.1.2.5       00001010 00000001 00000010 0 | 0000101   no match
                                          ^
                                          bit 25 differs
```

### Example 2: AD and Longest Prefix Match Together

A router called R5 is offered these routes:

| Source | Prefix | AD | Metric | Next hop |
|---|---|---|---|---|
| OSPF | 172.16.5.0/24 | 110 | 3 | 10.0.1.2 |
| RIP | 172.16.5.0/24 | 120 | 2 | 10.0.2.2 |
| RIP | 172.16.5.0/25 | 120 | 2 | 10.0.2.2 |
| Static | 172.16.0.0/16 | 1 | 0 | 10.0.3.2 |
| EIGRP | 172.16.0.0/16 | 90 | 28416 | 10.0.4.2 |

**Step 1: build the table.** Group the offers by exact prefix.

- 172.16.5.0/24: OSPF (110) beats RIP (120). The lower RIP hop count does not matter.
- 172.16.5.0/25: only RIP offers this. It wins with no contest, even with AD 120.
- 172.16.0.0/16: static (1) beats EIGRP (90).

Result:

```text
      172.16.0.0/16 is variably subnetted, 3 subnets, 3 masks
S        172.16.0.0/16 [1/0] via 10.0.3.2
O        172.16.5.0/24 [110/3] via 10.0.1.2, 00:03:12, GigabitEthernet1
R        172.16.5.0/25 [120/2] via 10.0.2.2, 00:00:17, GigabitEthernet2
```

**Step 2: forward packets.**

| Destination | Matches | Used route | Why |
|---|---|---|---|
| 172.16.5.10 | /16, /24, /25 | RIP /25 | Longest prefix. AD is not considered |
| 172.16.5.200 | /16, /24 | OSPF /24 | Outside the /25, which ends at .127 |
| 172.16.9.1 | /16 | Static /16 | Only match |
| 10.1.1.1 | None | Dropped | No default route. ICMP unreachable sent |

The first row is the classic interview trap. RIP has the worse AD, but it carries the packet.

### Example 3: One Packet Through R1

PC1 (192.168.1.50) sends a packet to PC2 (192.168.2.5), using the network from How It Works.

1. **PC1 checks its table.** 192.168.2.5 is not in its own subnet, so the default route applies.

   ```bash
   # Linux, on PC1
   $ ip route get 192.168.2.5
   192.168.2.5 via 192.168.1.1 dev eth0 src 192.168.1.50 uid 1000
       cache
   ```

2. **PC1 sends the frame.** The destination MAC is R1's G0/0 MAC. The destination IP stays 192.168.2.5.
3. **R1 receives it.** The destination MAC is R1's own, so R1 removes the Ethernet header and reads the IP header.
4. **R1 finds matches.** Two routes contain 192.168.2.5: 0.0.0.0/0 and 192.168.2.0/24.
5. **Longest prefix wins.** /24 beats /0. The route says next hop 10.0.12.2, out of G0/1.
6. **R1 confirms it with the forwarding table:**

   ```text
   R1# show ip cef 192.168.2.5
   192.168.2.0/24
     nexthop 10.0.12.2 GigabitEthernet0/1
   ```

7. **R1 rewrites and sends.** It lowers the TTL by one, updates the IP header checksum, and builds a new Ethernet header. The destination MAC is now R2's G0/1 MAC.
8. **R2 repeats the job.** Its table shows 192.168.2.0/24 as connected, so it uses ARP to find PC2's MAC and delivers the frame.

The reply is a separate journey. R2 must have a route to 192.168.1.0/24, or PC2's reply will never arrive.

## Going Deeper

### RIB and FIB

The routing table is not what fast routers search for each packet. Most routers split the job in two:

| | RIB (routing table) | FIB (forwarding table) |
|---|---|---|
| Purpose | Decide the best routes | Forward packets fast |
| Lives in | Control plane, software, CPU | Data plane, often in hardware |
| Holds | Source, AD, metric, age, next hop | Prefix, resolved next hop, link to Layer 2 data |
| Changes | When routes change | Copied from the RIB |

On Cisco, the system is called CEF (Cisco Express Forwarding). CEF has two parts: the FIB and the **adjacency table**, which holds the ready-made Layer 2 header for each neighbour.

```text
R1# show ip cef 192.168.2.5
R1# show adjacency GigabitEthernet0/1
R1# show ip cef exact-route 192.168.1.50 192.168.2.5
```

The last command shows which path a specific source and destination pair will take. This helps when there are equal-cost paths.

`show ip route` uses the RIB. Packets use the FIB. So `show ip cef` is the better answer to "where will this packet really go?" Here is a real gap: `show ip route 198.51.100.7` can print "% Network not in table" even though the default route would forward that packet. `show ip cef 198.51.100.7` shows the 0.0.0.0/0 entry that is really used.

High-end routers keep the FIB in TCAM, a special memory that can check many entries at once. Software routers use tree structures. Linux uses a compressed tree called an LC-trie. This matters at scale. The global BGP table now holds about one million IPv4 prefixes, and over 200,000 IPv6 prefixes, and the numbers keep growing.

### Equal-Cost Multipath (ECMP)

If one protocol finds two paths with the same metric, both go in the table:

```text
O     192.168.3.0/24 [110/3] via 10.0.12.2, 00:02:10, GigabitEthernet0/1
                     [110/3] via 10.0.13.2, 00:02:10, GigabitEthernet0/3
```

The router normally hashes packet fields, so all packets of one flow take the same path. This protects packet order. It is not a perfect 50/50 split.

The limit is set with `maximum-paths` under the routing process. On IOS, common defaults are 4 for IGPs and 1 for BGP, but these vary by platform. An IGP is a routing protocol used inside one organisation, such as OSPF.

```bash
# Linux: one prefix, two next hops
$ sudo ip route add 10.50.0.0/16 \
    nexthop via 192.168.1.1 dev eth0 weight 1 \
    nexthop via 192.168.1.254 dev eth0 weight 1
```

On Linux, `net.ipv4.fib_multipath_hash_policy` selects the hash. 0 uses Layer 3 fields, and 1 adds ports.

### Null Routes

A route can point to nowhere on purpose. Traffic matching it is dropped.

```text
! Cisco IOS: drop this prefix
R1(config)# ip route 10.99.0.0 255.255.0.0 Null0
```

```bash
# Linux
$ sudo ip route add blackhole 10.99.0.0/16
```

Null routes stop routing loops when a router advertises a summary. If one subnet of the summary disappears, packets for it hit the null route rather than following a default route back and forth. EIGRP adds this automatically:

```text
D        10.0.0.0/8 is a summary, 00:01:00, Null0
```

That is the AD 5 entry in the table above. See [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md).

### The IPv6 Table

The logic is the same: longest prefix match, then AD and metric at build time. The prefix can be up to /128.

```text
R1# show ipv6 route
IPv6 Routing Table - default - 7 entries
Codes: C - Connected, L - Local, S - Static, O - OSPF Intra
S   ::/0 [1/0]
     via 2001:DB8:FFFF::1
C   2001:DB8:10::/64 [0/0]
     via GigabitEthernet0/0, directly connected
L   2001:DB8:10::1/128 [0/0]
     via GigabitEthernet0/0, receive
O   2001:DB8:20::/64 [110/2]
     via FE80::2, GigabitEthernet0/1
C   2001:DB8:FFFF::/64 [0/0]
     via GigabitEthernet0/2, directly connected
L   2001:DB8:FFFF::2/128 [0/0]
     via GigabitEthernet0/2, receive
L   FF00::/8 [0/0]
     via Null0, receive
```

Note the differences. The route's details sit on a second line. The OSPF next hop is a link-local fe80:: address, so an interface is shown with it. The legend above is shortened.

```bash
# Linux
$ ip -6 route show
```

```powershell
# Windows
route print -6
Get-NetRoute -AddressFamily IPv6
```

### Routing Is One-Way

Each router decides alone, using only its own table and the destination address. Two results follow:

- The forward path and the return path may differ. This is called **asymmetric routing**. It is common on the Internet, and it can break stateful firewalls.
- A routing loop can form if two routers point at each other. The TTL stops the packet from circling forever.

### Hosts and Layer 3 Switches

Hosts have tables, but they do not forward packets for others by default. On Linux, `net.ipv4.ip_forward` set to 0 means off. Many Catalyst Layer 3 switches need `ip routing` turned on, or they act like a host.

## Common Mistakes

- **"Lowest AD is always used."** AD only compares routes for the exact same prefix and length. A longer prefix with a worse AD still wins the lookup.
- **Comparing 10.0.0.0/8 and 10.0.0.0/16 by AD.** They are two different routes. Both can be in the table.
- **Comparing metrics across protocols.** OSPF cost 20 and RIP 2 hops have different units.
- **Thinking the table is read top to bottom.** That is how ACLs work. Routing uses the longest match, and line order has no meaning.
- **Thinking the next hop IP is written in the packet.** It is used only to find the next hop's MAC address.
- **Assuming a route is unknown because `show ip route` does not list it.** The protocol may have learned it and lost on AD. Check the protocol's own database, such as `show ip ospf database` or `show ip bgp`.
- **Thinking "C" and "L" are the same.** C means "forward onto this subnet". L means "this packet is for me".
- **Looking only at the forward path.** Ping needs a route in both directions. Check the far router too.
- **Reading the metric as AD in `[110/2]`.** The first number is AD. The second is metric.
- **Using only an exit interface for a static route on Ethernet.** It depends on proxy ARP and fills the ARP table.
- **Expecting AD on Linux or Windows.** Their kernels use prefix length, then metric.
- **Trusting `show ip route <address>` for default traffic.** It may say the network is not in the table. Use `show ip cef <address>`.

## Interview Questions

**Q: What is a routing table, and what does one entry hold?**
A: It is the list of networks a device knows and how to reach them. An entry holds the destination prefix, the next hop or exit interface, and on routers the source, administrative distance and metric. Routers and normal hosts both have one.

**Q: What does `[110/2]` mean in `show ip route`?**
A: The first number is administrative distance, so 110 is the OSPF default. The second is the metric, here an OSPF cost of 2.

**Q: What is the difference between a C route and an L route?**
A: C is the connected subnet on an interface, like 192.168.1.0/24. Traffic to it goes out that interface to the host. L is the router's own address as a /32, like 192.168.1.1/32. Traffic to it stays on the router and goes to its CPU.

**Q: A router has an OSPF route for 10.1.1.0/24 and a RIP route for 10.1.1.0/25. A packet arrives for 10.1.1.10. Which route is used?**
A: The RIP /25. Different prefix lengths are different routes, so both are in the table and AD does not compare them. Longest prefix match chooses the /25.

**Q: What is the difference between administrative distance and metric, and in what order are they used?**
A: AD compares different sources, and metric compares paths within one source. When building the table, the router picks by lowest AD for each exact prefix. The protocol has already used its metric to pick its own best path. When forwarding, the router uses longest prefix match among the installed routes.

**Q: Give the common Cisco administrative distances.**
A: Connected 0, static 1, eBGP 20, internal EIGRP 90, OSPF 110, IS-IS 115, RIP 120, external EIGRP 170, iBGP 200, and 255 means never use. These are vendor defaults, not a standard. Junos, for example, uses different values.

**Q: What is the difference between next hop and exit interface?**
A: The next hop is the IP address of the next router. The exit interface is the local port. On Ethernet you should give a next hop, because an exit interface alone makes the router ARP for every destination. On a point-to-point link, the exit interface alone is fine.

**Q: What is a recursive lookup?**
A: It happens when a route's next hop is not directly connected, so the router looks up the next hop in the table too. This repeats until it reaches an exit interface. BGP routes commonly need it, for example a next hop on a loopback learned through OSPF. Modern routers pre-resolve this in the FIB.

**Q: What is the difference between the RIB and the FIB?**
A: The RIB is the routing table in the control plane, and it holds the best routes with their details. The FIB is built from the RIB for fast forwarding, often in hardware, with next hops already resolved. On Cisco, CEF is the system that holds the FIB and the adjacency table.

**Q: What happens if no route matches a packet?**
A: If there is no match and no default route, the router drops the packet and normally sends ICMP destination network unreachable to the source. A default route, 0.0.0.0/0, matches everything, so it catches anything with no better match.

**Q: A protocol learned a route, but it is not in the routing table. Why?**
A: The most likely reason is that another source offered the same prefix with a lower AD. It can also fail if its next hop cannot be resolved. The route stays in the protocol's own database and can be installed if the winner disappears.

**Q: How do you see which route is used for an address on Linux and Windows?**
A: On Linux I run `ip route get 198.51.100.7`, which does a real lookup and shows the next hop, device and source IP. On Windows I use `Find-NetRoute -RemoteIPAddress 198.51.100.7`, or I read `route print` and apply longest match myself. On Windows, the cost is route metric plus interface metric.

## Quick Recap

- The routing table lists prefixes and how to reach them. Routers and hosts both use one.
- Cisco uses `show ip route`, Linux uses `ip route`, and Windows uses `route print` or `Get-NetRoute`.
- C is the connected subnet. L is the device's own /32 address.
- For each exact prefix, the lowest AD enters the table. The protocol's metric picks its own best path.
- For each packet, longest prefix match decides. No match and no default route means drop.
- The next hop is used to find a MAC address. It is never written in the packet.
- The RIB is the decision table. The FIB is the fast copy that forwards packets.
