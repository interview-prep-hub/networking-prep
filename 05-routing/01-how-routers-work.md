# How Routers Work

**Level:** Beginner | **Prerequisites:** [Network Devices](../01-foundations/04-network-devices.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md), [ARP](../03-ethernet-and-switching/05-arp.md), [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md)

## In One Line

A router reads the destination IP address of a packet, chooses the next device on the path, and sends the packet there inside a brand new frame.

## Why It Matters

Devices on the same network can talk directly. Devices on different networks cannot. A router is the device that joins networks together, and the internet is simply a very large number of networks joined by routers.

Three real problems are solved here:

- **Reach:** a packet can cross many networks, even when no device knows the whole path.
- **Mixed links:** a packet can leave on Ethernet and cross fibre or a serial line, because each router builds a new frame for the next link.
- **Speed:** a good router makes this decision for millions of packets each second.

Interviewers love this topic because it shows if you really understand layers. "Why does the MAC address change but the IP address stay the same?" is one of the most common networking questions. A clear answer proves you understand Layer 2, Layer 3, ARP and routing all at once.

## Simple Explanation

Imagine you post a letter from Karachi to a friend in London.

The address on your envelope is the final address. It never changes. This is the **IP address**.

Post offices do not carry one letter at a time. Each office puts your letter into a bag with a label: "from Karachi office to Dubai office". In Dubai, staff throw that bag away. They read your envelope, look at a sorting chart, and put the letter into a new bag: "from Dubai office to London office".

The bag label is the **MAC address**. It is only good for one trip between two neighbouring offices. The sorting chart is the **routing table**. Each office knows only the next office, not the whole path.

Now imagine one more rule. A small number is written on the envelope, and each office subtracts 1. If it reaches 0, the letter is destroyed and a note goes back to the sender. This stops a lost letter from travelling forever. That number is the **TTL**.

**Where the analogy stops being true:** a post office may track your letter. A router normally remembers nothing about a packet after it leaves. Each packet is a new decision. Also, a real letter usually takes one planned path, while two packets to the same place can take different paths.

## How It Works

### The words you need

- **Packet:** the Layer 3 unit. It has an IP header with a source and destination IP address.
- **Frame:** the Layer 2 wrapper around a packet. On Ethernet it has a source and destination MAC address.
- **Interface:** a port on the router. Each interface usually connects to a different network and has its own IP address and MAC address.
- **Hop:** one trip from a device to the next router or to the final host. A switch in the middle does not count as a hop.
- **Routing table:** the router's list of known networks and how to reach each one. See [The Routing Table](02-routing-table.md).
- **Next hop:** the next router on the path.
- **ARP (Address Resolution Protocol):** the protocol that finds the MAC address that belongs to an IP address on the local link. See [ARP](../03-ethernet-and-switching/05-arp.md).
- **ICMP (Internet Control Message Protocol):** the protocol that carries error messages about IP delivery. See [ICMP, Ping and Traceroute](10-icmp-ping-and-traceroute.md).

### What a router does, step by step

```text
 Frame arrives on an interface
            |
            v
 1. FCS correct? ------------------- no --> drop
            | yes
            v
 2. Dest MAC is mine or broadcast? - no --> drop
            | yes
            v
 3. Remove frame header, read EtherType
            |
            v
 4. IP header valid? --------------- no --> drop
            | yes
            v
 5. Dest IP is mine? --------------- yes -> give to router CPU
            | no
            v
 6. TTL - 1 = 0? ------------------- yes -> drop, ICMP Time Exceeded
            | no
            v
 7. Route found? ------------------- no --> drop, ICMP Unreachable
            | yes
            v
 8. Next-hop MAC known? ------------ no --> send ARP request
            | yes
            v
 9. Packet fits exit link MTU? ----- no --> fragment or drop
            | yes
            v
10. Build new frame, queue it, send
```

1. **Receive the frame and check it.** The frame ends with an FCS (Frame Check Sequence), a number used to detect damage. If it does not match, the frame is dropped.
2. **Check the destination MAC.** Is this frame for me? It must carry the MAC address of this interface, or a broadcast or multicast address the router accepts.
3. **Open the frame.** The router removes the Ethernet header. A field called EtherType says what is inside: 0x0800 means IPv4, 0x86DD means IPv6, and 0x0806 means ARP.
4. **Check the IP header.** The router checks the version, the length and, for IPv4, the header checksum (a number used to detect damage in the header). A bad header means the packet is dropped.
5. **Ask "is this packet for me?"** If the destination IP is one of the router's own addresses, the packet goes up to the router's own software. A routing update or an SSH login arrives this way.
6. **Reduce the TTL.** The router subtracts 1. If the result is 0, it drops the packet and tells the sender.
7. **Look up the destination in the routing table.** The router picks the most specific matching route. This rule is called longest prefix match. If nothing matches and there is no default route (a catch-all route for unknown destinations), the router drops the packet and sends ICMP Destination Unreachable.
8. **Find the next-hop MAC address.** The router checks its ARP table. If the entry is missing, it sends an ARP request.
9. **Check the size.** MTU (Maximum Transmission Unit) is the largest packet a link can carry. If the packet is too big, an IPv4 router may fragment it (split it into pieces). If the Don't Fragment flag is set, or the packet is IPv6, the router drops it and sends an ICMP error. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).
10. **Build a new frame and send it.** The source MAC is the exit interface. The destination MAC is the next hop. A new FCS is calculated. If the link is busy, the frame waits in a queue. If the queue is full, it may be dropped.

**Fuller truth:** this order is a teaching order. RFC 1812 (the requirements for IPv4 routers) describes these jobs, but real hardware does many of them at the same time. Filters such as [ACLs](../09-security/05-access-control-lists.md) (Access Control Lists) and address changes such as [NAT](../04-ip-addressing/09-nat-and-pat.md) (Network Address Translation) also run along this path.

### Why the MAC changes and the IP stays the same

Each address type has a different job.

| | MAC address | IP address |
|---|---|---|
| Layer | Layer 2 | Layer 3 |
| Scope | One link or one local network | The whole path, end to end |
| Question it answers | Who gets this frame next? | Who gets this packet in the end? |
| Changes at each router? | Yes | No (unless NAT is used) |

The MAC changes for three reasons:

1. **A MAC address has no location.** It does not say where a device is, so it cannot guide a packet across the world.
2. **PC-A cannot know the far MAC.** ARP works only on the local network, because ARP requests are broadcasts and routers do not forward them.
3. **The next link may not be Ethernet.** Some links, such as PPP (Point-to-Point Protocol) serial links, do not use MAC addresses at all.

Here is a real path. We will use it for the rest of the chapter.

```text
+------+        +----+         +----+         +----+        +--------+
| PC-A |--------| R1 |---------| R2 |---------| R3 |--------| Server |
+------+ Link 1 +----+ Link 2  +----+ Link 3  +----+ Link 4 +--------+

Link 1 = 192.168.1.0/24     Link 2 = 10.0.12.0/30
Link 3 = 10.0.23.0/30       Link 4 = 192.168.2.0/24
```

| Device and interface | IP address | MAC address |
|---|---|---|
| PC-A eth0 | 192.168.1.10/24 | 02:00:00:aa:00:01 |
| R1 G0/0 | 192.168.1.1/24 | 02:00:00:11:00:00 |
| R1 G0/1 | 10.0.12.1/30 | 02:00:00:11:00:01 |
| R2 G0/0 | 10.0.12.2/30 | 02:00:00:22:00:00 |
| R2 G0/1 | 10.0.23.1/30 | 02:00:00:22:00:01 |
| R3 G0/0 | 10.0.23.2/30 | 02:00:00:33:00:00 |
| R3 G0/1 | 192.168.2.1/24 | 02:00:00:33:00:01 |
| Server eth0 | 192.168.2.20/24 | 02:00:00:bb:00:01 |

PC-A sends one packet to the Server. This is what is on each link:

| Link | Source MAC | Destination MAC | Source IP | Destination IP | TTL |
|---|---|---|---|---|---|
| 1: PC-A to R1 | 02:00:00:aa:00:01 | 02:00:00:11:00:00 | 192.168.1.10 | 192.168.2.20 | 64 |
| 2: R1 to R2 | 02:00:00:11:00:01 | 02:00:00:22:00:00 | 192.168.1.10 | 192.168.2.20 | 63 |
| 3: R2 to R3 | 02:00:00:22:00:01 | 02:00:00:33:00:00 | 192.168.1.10 | 192.168.2.20 | 62 |
| 4: R3 to Server | 02:00:00:33:00:01 | 02:00:00:bb:00:01 | 192.168.1.10 | 192.168.2.20 | 61 |

Read it column by column. Both MAC columns change on every link. Both IP columns never change. TTL falls by 1 at each router.

This is what R1 does to the frame:

```text
Arrives at R1 on G0/0
+------------------+------------------+----------------------+-----+
| Dst MAC          | Src MAC          | IP packet            | FCS |
| 02:00:00:11:00:00| 02:00:00:aa:00:01| to 192.168.2.20      |     |
|                  |                  | TTL 64               |     |
+------------------+------------------+----------------------+-----+

Leaves R1 on G0/1 (new header, same IP addresses, new TTL, new FCS)
+------------------+------------------+----------------------+-----+
| Dst MAC          | Src MAC          | IP packet            | FCS |
| 02:00:00:22:00:00| 02:00:00:11:00:01| to 192.168.2.20      |     |
|                  |                  | TTL 63               |     |
+------------------+------------------+----------------------+-----+
```

**Fuller truth:** "IP never changes" is true for plain routing. [NAT](../04-ip-addressing/09-nat-and-pat.md) rewrites IP addresses on purpose. A [VPN](../09-security/07-vpn.md) (Virtual Private Network) tunnel wraps the packet in a second, outer IP header.

### TTL: the safety counter

TTL (Time To Live) is an 8-bit field in the IPv4 header, so it holds 0 to 255.

The sender picks the starting value. Common defaults are 64 on Linux, 128 on Windows and 255 on Cisco IOS. These are operating system defaults, not a standard, and an administrator can change them.

Each router subtracts 1. If the result is 0, the router:

1. Drops the packet.
2. Sends ICMP Time Exceeded (Type 11, Code 0) back to the source IP address.

**Why it exists:** routing mistakes can create loops. Ethernet frames have no TTL, but IP packets do. Without TTL, a packet in a loop would circle forever, and more and more packets would pile up on the link.

**Analogy:** think of a train ticket with a fixed number of stations. Each station punches it once. When it runs out, you must leave the train. The analogy stops here: a real ticket is used to pay for travel, but TTL exists only to end a journey that went wrong.

**Fuller truth:** RFC 791 defined TTL in seconds, which explains the name. In practice routers subtract 1, so it counts hops. IPv6 (RFC 8200) uses the honest name **Hop Limit**. When it runs out, the router sends ICMPv6 Time Exceeded (Type 3, Code 0). See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).

Because the TTL changes, the IPv4 header checksum must be recalculated at each hop. IPv6 has no header checksum, so this work is gone.

### The three planes of a router

A router does three kinds of work. We call them **planes**. A plane is simply a group of related jobs.

```text
+------------------------------------------------------------+
| MANAGEMENT PLANE: how people and tools run the router      |
| SSH, SNMP, syslog, NETCONF, the command line               |
+------------------------------------------------------------+
                 | configures, monitors
                 v
+------------------------------------------------------------+
| CONTROL PLANE: decides where traffic should go             |
| OSPF, BGP, static routes, ARP --> routing table --> FIB    |
+------------------------------------------------------------+
                 | installs forwarding tables
                 v
+------------------------------------------------------------+
| DATA PLANE: moves the traffic                              |
| in port ===== lookup, rewrite, queue =====> out port       |
+------------------------------------------------------------+
```

| | Data plane | Control plane | Management plane |
|---|---|---|---|
| Job | Forward packets | Build the map | Let humans and tools control the box |
| Other name | Forwarding plane | Routing logic | Admin access |
| Examples | User traffic passing through | OSPF, BGP, ARP | SSH, SNMP, syslog |
| Runs on | ASICs (custom chips) or fast software path | Main CPU | Main CPU |
| Speed need | Extremely high | Lower | Lowest |

New names in that picture: OSPF and BGP are routing protocols, which routers use to share routes. SSH is secure remote login. SNMP is a protocol for reading device health and counters. Syslog sends log messages to a server. NETCONF lets software tools change the configuration.

**Analogy:** think of a post office. Sorting workers move letters very fast (data plane). Planners talk to other post offices and write the sorting chart (control plane). The manager's office sets the rules and reads reports (management plane).

**Where the analogy stops:** on a small router, one CPU may do all three jobs. The planes are a way of thinking about the work, not always separate hardware. Also, some books place management inside the control plane.

Key point: an SSH login to the router enters like any other packet, but its destination is the router itself. It is management traffic. A packet that only passes through the router is data plane traffic.

### Process switching and CEF

Here "switching" means moving a packet from the input interface to the output interface. It is not about Layer 2 switches.

**Process switching** is the slow way. Every packet goes to the main CPU. The CPU does a full routing table lookup, finds the ARP entry, and builds the frame. On Cisco IOS, a software process named IP Input does this.

**Fast switching** is the older middle step. The first packet to a destination is process switched, and the answer is saved in a cache. Later packets use the cache. The problem is that the cache is built only after traffic arrives, so many new destinations cause many slow lookups. Treat it as history.

**CEF (Cisco Express Forwarding)** is the modern way. It prepares the answers before any packet arrives, using two tables:

- **FIB (Forwarding Information Base):** a copy of the best routes, built for very fast lookup. It gives the next hop and exit interface.
- **Adjacency table:** the ready-made Layer 2 header for each directly connected neighbour, built from ARP data.

```text
Process switching: the CPU works on every packet
  pkt 1 --> CPU --> out
  pkt 2 --> CPU --> out
  pkt 3 --> CPU --> out

CEF: the control plane prepares tables in advance
  routing table --> FIB          ARP table --> adjacency table
                     \                         /
  pkt 1, 2, 3 ------> lookup + ready header -----> out
```

**Analogy:** process switching is a shop worker who looks up the price in a big book for every customer. CEF is a price list printed each morning. The analogy stops here: the FIB is not printed once a day. It updates as soon as routes change.

The main CPU is now free for the control plane. On hardware routers, the FIB is copied to chips on the line cards (the boards that hold the ports), so forwarding does not touch the main CPU at all.

CEF is a Cisco name. The idea is universal. Juniper uses a Routing Engine for the control plane and a Packet Forwarding Engine for the data plane. The Linux kernel keeps its own FIB too.

## Example

### Setup on Cisco IOS

Use the topology from How It Works. All three routers need routes in both directions. The most common beginner error is to add only the forward route.

```text
! Cisco IOS - R1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface GigabitEthernet0/1
 ip address 10.0.12.1 255.255.255.252
 no shutdown
ip route 192.168.2.0 255.255.255.0 10.0.12.2

! Cisco IOS - R2 (after its interface addresses)
ip route 192.168.1.0 255.255.255.0 10.0.12.1
ip route 192.168.2.0 255.255.255.0 10.0.23.2

! Cisco IOS - R3 (after its interface addresses)
ip route 192.168.1.0 255.255.255.0 10.0.23.1
```

### Step 1: PC-A decides where to send

The Server (192.168.2.20) is not in 192.168.1.0/24, so PC-A sends the packet to its default gateway (the router a host uses to reach other networks), R1.

```bash
# Linux on PC-A
$ ip route get 192.168.2.20
192.168.2.20 via 192.168.1.1 dev eth0 src 192.168.1.10 uid 1000
    cache

$ ip neigh show 192.168.1.1
192.168.1.1 dev eth0 lladdr 02:00:00:11:00:00 REACHABLE
```

```powershell
# Windows, if PC-A runs Windows
PS> arp -a 192.168.1.1

Interface: 192.168.1.10 --- 0x6
  Internet Address      Physical Address      Type
  192.168.1.1           02-00-00-11-00-00     dynamic
```

The packet is for the Server, but the frame is for R1. That one line is the heart of this chapter.

### Step 2: R1 decides

```text
! Cisco IOS - R1
R1# show ip route
Codes: L - local, C - connected, S - static (list trimmed)

Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.12.0/30 is directly connected, GigabitEthernet0/1
L        10.0.12.1/32 is directly connected, GigabitEthernet0/1
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
S     192.168.2.0/24 [1/0] via 10.0.12.2
```

The packet matches the `S` line, so the next hop is 10.0.12.2. CEF already holds this answer:

```text
R1# show ip cef 192.168.2.20
192.168.2.0/24
  nexthop 10.0.12.2 GigabitEthernet0/1

R1# show arp 10.0.12.2
Protocol  Address    Age (min)  Hardware Addr   Type  Interface
Internet  10.0.12.2         12  0200.0022.0000  ARPA  GigabitEthernet0/1

R1# show adjacency GigabitEthernet0/1 detail
Protocol Interface                 Address
IP       GigabitEthernet0/1        10.0.12.2(7)
                                   Encap length 14
                                   0200002200000200001100010800
                                   ARP
```

(Output is trimmed, and the layout varies by IOS version.)

Look closely at that long hex string. It is the complete 14-byte Ethernet header, ready to use:

```text
0200.0022.0000  0200.0011.0001  0800
Dst MAC (R2)    Src MAC (R1)    EtherType (IPv4)
```

R1 only adds this header, and the frame is ready. No ARP lookup is needed for each packet.

### Step 3: See the hops and the TTL

```bash
# Linux on PC-A
$ traceroute -n 192.168.2.20
traceroute to 192.168.2.20 (192.168.2.20), 30 hops max, 60 byte packets
 1  192.168.1.1   1.204 ms  1.101 ms  1.080 ms
 2  10.0.12.2     2.310 ms  2.254 ms  2.201 ms
 3  10.0.23.2     3.402 ms  3.377 ms  3.351 ms
 4  192.168.2.20  3.921 ms  3.890 ms  3.862 ms

$ ping -c 2 192.168.2.20
PING 192.168.2.20 (192.168.2.20) 56(84) bytes of data.
64 bytes from 192.168.2.20: icmp_seq=1 ttl=61 time=3.92 ms
64 bytes from 192.168.2.20: icmp_seq=2 ttl=61 time=3.87 ms
```

```powershell
# Windows, if PC-A runs Windows
PS> tracert -d 192.168.2.20

Tracing route to 192.168.2.20 over a maximum of 30 hops

  1     1 ms     1 ms     1 ms  192.168.1.1
  2     2 ms     2 ms     2 ms  10.0.12.2
  3     3 ms     3 ms     3 ms  10.0.23.2
  4     4 ms     3 ms     4 ms  192.168.2.20

Trace complete.

PS> ping -n 2 192.168.2.20

Pinging 192.168.2.20 with 32 bytes of data:
Reply from 192.168.2.20: bytes=32 time=3ms TTL=61
Reply from 192.168.2.20: bytes=32 time=3ms TTL=61
```

Why is it TTL 61? The TTL you see in ping belongs to the **reply**. The Server runs Linux, so it starts at 64. Three routers subtract 1 each: 64 - 3 = 61.

Traceroute uses the TTL on purpose. It sends packets with TTL 1, then 2, then 3, and each router that drops one reports its address. Full detail is in [ICMP, Ping and Traceroute](10-icmp-ping-and-traceroute.md).

### What a routing loop looks like

Suppose R1 sends unknown traffic to R2. By mistake, R2's default route points to R3, and R3's default route points back to R2. PC-A sends a packet to 203.0.113.50 with TTL 64.

```text
R1: 64 -> 63
R2: 63 -> 62      R3: 62 -> 61
R2: 61 -> 60      R3: 60 -> 59
...               ...
R2:  3 -> 2       R3:  2 -> 1
R2:  1 -> 0       drop, ICMP Time Exceeded to 192.168.1.10
```

After 64 router hops the packet dies. In traceroute the loop is easy to spot, because two addresses repeat:

```text
 1  192.168.1.1
 2  10.0.12.2
 3  10.0.23.2
 4  10.0.23.1
 5  10.0.23.2
 6  10.0.23.1
```

## Going Deeper

This section is optional for a first read.

**Why the data plane needs hardware.** The smallest Ethernet frame takes 84 bytes on the wire (64 bytes of frame plus 20 bytes of preamble and gap). On a 100 Gb/s link that is about 148.8 million packets per second, or one packet every 6.7 nanoseconds. A general CPU cannot do a full lookup in that time, so high-end routers use ASICs (Application-Specific Integrated Circuits). Many use TCAM (Ternary Content-Addressable Memory), which searches all its entries at the same moment.

**CEF adjacency types.** Not every FIB entry points to a normal next hop. You will see these names in `show ip cef`:

- **attached:** the network is directly connected.
- **receive:** the address belongs to this router, so the packet goes to the CPU.
- **glean:** the network is connected, but the MAC of the host is not known yet, so ARP is needed.
- **drop or null:** the packet is thrown away, as with a route to Null0 (a virtual interface that discards traffic).

**What still goes to the CPU with CEF.** This is called a **punt**. Common cases are packets for the router itself, packets whose TTL expired (someone must build the ICMP error), packets that need ARP first, and packets with IP options.

**Protecting the CPU.** Punted traffic can overload the slow CPU. Control Plane Policing (CoPP) limits the rate of traffic sent to it.
**Control plane restart.** Since the FIB lives apart from the routing process, a router can keep forwarding for a short time while its control plane restarts. Non-Stop Forwarding and Graceful Restart use this idea. Neighbours must cooperate, or they will remove the routes.

**Control plane packets and TTL.** OSPF packets are sent with TTL 1, and eBGP (BGP between different organisations) uses TTL 1 by default, so they do not travel past the neighbour. GTSM (Generalized TTL Security Mechanism, RFC 5082) flips this idea: neighbours send with TTL 255, and the receiver rejects anything lower. A distant attacker cannot make a packet arrive with TTL 255, since every router on the way subtracts 1.

**Routers and Layer 3 switches.** A Layer 3 switch follows the same logic in hardware. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md). On many Cisco Catalyst switches, routing is off until you enter `ip routing`. On Cisco routers, IPv6 forwarding needs `ipv6 unicast-routing`.

**Any computer can route.** Forwarding is off by default on normal hosts.

```bash
# Linux: turn on IPv4 forwarding until the next reboot
sudo sysctl -w net.ipv4.ip_forward=1
```

```powershell
# Windows: turn on forwarding on one interface
Set-NetIPInterface -InterfaceAlias "Ethernet" -Forwarding Enabled
```

## Common Mistakes

- **"The router changes the destination IP to the next router."** No. The destination MAC points to the next hop. The destination IP stays the final destination.
- **"The source sends a frame to the final MAC address."** Only on the same network. For a remote host, the frame goes to the gateway's MAC.
- **"A switch changes the MAC addresses."** A normal Layer 2 switch forwards the frame unchanged. Only routers build new frames.
- **"Switches reduce the TTL."** Only devices that route do this. A Layer 2 switch never touches the IP header.
- **"The routing table contains the whole path."** It holds only the next step. Each router decides alone.
- **"The return path is the same as the forward path."** Each direction is routed separately. Traffic often takes a different path back, and the return route must exist.
- **"TTL is measured in seconds."** In practice it counts hops.
- **"A dropped packet always sends an error back."** Not always. Errors can be rate limited or filtered, and bad frames are dropped silently.
- **"The routing table and the FIB are the same thing."** The routing table (RIB) is the control plane view. The FIB is the copy optimised for forwarding.
- **"A first-ping failure means a fault."** On Cisco IOS, the first ping often shows `.!!!!`. The first packet can be lost while ARP completes. This is normal.

## Interview Questions

**Q: What does a router do when it receives a packet?**

A: It checks the frame, removes the Layer 2 header, and validates the IP header. If the packet is not for the router, it reduces the TTL and finds the best route with longest prefix match. Then it finds the next-hop MAC address, builds a new frame, and sends it out the exit interface.

**Q: Why does the MAC address change at each hop while the IP address stays the same?**

A: MAC addresses are for delivery on one local link, and IP addresses identify the final source and destination. Each router removes the old frame and builds a new one for the next link. The IP packet inside stays the same end to end, except for fields like TTL. NAT is the main exception.

**Q: Which fields in the IPv4 header change at each hop?**

A: The TTL goes down by 1, so the header checksum must be recalculated. Source and destination IP stay the same unless NAT is used. In IPv6, only the Hop Limit changes, because there is no header checksum.

**Q: What is TTL and what happens when it reaches 0?**

A: TTL is an 8-bit counter in the IPv4 header that stops packets from looping forever. Each router reduces it by 1. When it reaches 0, the router drops the packet and sends ICMP Time Exceeded to the source. Traceroute uses this behaviour to discover each hop.

**Q: Explain the control plane, data plane and management plane.**

A: The data plane forwards user traffic and must be very fast. The control plane builds the forwarding information, using protocols such as OSPF, BGP and ARP. The management plane is how we access and monitor the device, using SSH, SNMP and syslog.

**Q: Is an SSH session to a router data plane or management plane traffic?**

A: It is management plane traffic, because the router itself is the destination and the packet is handed to its CPU. If the same SSH session only passes through the router to another host, it is data plane traffic for that router.

**Q: What happens if the control plane crashes?**

A: Forwarding can continue for a short time, because the FIB is already in the data plane. But the router stops learning changes, and neighbours soon notice it has gone and remove their routes. Features like Non-Stop Forwarding and Graceful Restart reduce this impact.

**Q: What is the difference between process switching and CEF?**

A: In process switching, the CPU handles every packet with a full lookup, so it is slow. CEF builds the FIB and the adjacency table in advance, so each packet gets a fast lookup and a ready-made Layer 2 header. CEF is the normal mode on modern Cisco routers, and process switching stays for special cases.

**Q: What is the difference between the RIB and the FIB?**

A: The RIB, or routing table, is built by the control plane and holds the best routes from all sources. The FIB is built from the RIB and is structured for fast forwarding in the data plane. The adjacency table supplies the Layer 2 headers.

**Q: What does a router do with a packet that matches no route?**

A: If there is no matching route and no default route, it drops the packet. It normally sends ICMP Destination Unreachable back to the source.

**Q: Do routers forward broadcasts?**

A: No. Each router interface ends a broadcast domain. DHCP relay is a common special case: the router receives a host's broadcast request for an address and sends a new unicast packet to the DHCP server.

**Q: What is the difference between how a switch and a router handle a frame?**

A: A switch looks at the destination MAC and forwards the frame unchanged within the same network. A router removes the frame, reads the destination IP, reduces the TTL, and builds a new frame for the next network.

## Quick Recap

- A router joins networks and chooses the next hop for each packet using the destination IP.
- A router removes the old frame and builds a new one, so MAC addresses change on every link.
- IP addresses stay the same end to end, unless NAT or a tunnel is involved.
- TTL falls by 1 at each router. At 0 the packet is dropped, and ICMP Time Exceeded goes to the sender.
- The data plane forwards, the control plane decides, and the management plane lets people run the device.
- Process switching uses the CPU for every packet. CEF uses a prebuilt FIB and adjacency table.
