# The TCP/IP Model

**Level:** Beginner | **Prerequisites:** [Protocols, Standards and Why We Use Layers](01-why-layers-and-protocols.md), [The OSI Model](02-osi-model.md)

## In One Line

The TCP/IP model describes how the real Internet works, using four layers (Application, Transport, Internet, Link). Many textbooks split the bottom layer in two and call it the 5-layer model.

## Why It Matters

Two computers on different continents need to exchange data. They may use different hardware, different operating systems and different kinds of cable. Something must make them all work together. The TCP/IP protocols do this job, and the TCP/IP model explains how those protocols fit together.

A **protocol** is an agreed set of rules for talking, such as how a message starts, what each part means, and what to do when something goes wrong. A **model** is a way to organise many protocols into groups. Each group is called a **layer**.

The [OSI model](02-osi-model.md) is the theory most people learn first. TCP/IP is what actually runs on your phone, your laptop and every cloud server.

Interviewers ask about it because:

- It shows if you know the difference between a teaching model and a working system.
- The questions have easy-sounding answers with traps, such as "Is it 4 or 5 layers?" and "Which layer is ARP in?"
- Real troubleshooting follows these layers: link up, IP reachable, port open, application healthy.

## Simple Explanation

Imagine you send a gift to a friend in another country.

1. **Application:** you write the letter. Only you and your friend care what it says.
2. **Transport:** your office mailroom puts it in an envelope, writes the department name on it, and keeps a tracking record. If it gets lost, the mailroom sends it again.
3. **Internet:** the national post office reads the street address and picks a route across the world. It only cares about the next sorting centre.
4. **Link:** a truck carries the box from one sorting centre to the next. A different truck, plane or ship does the next part of the trip.

Each worker does one job and trusts the others to do theirs. The truck driver does not read your letter. The letter writer does not plan the truck route.

**Where the analogy stops being true:** a post office moves one parcel at a time. A network cuts your data into many small pieces, and each piece may take a different route. Also, a real truck driver knows nothing about the other trucks, but link technologies such as Wi-Fi share one medium and must take turns.

## How It Works

### The Two Versions: 4 Layers and 5 Layers

The name "TCP/IP" comes from its two most famous protocols: TCP (Transmission Control Protocol) and IP (Internet Protocol). The full family has hundreds of protocols, so it is also called the **Internet protocol suite**. A **suite** is a set of protocols designed to work together.

The 4-layer version comes from RFC 1122 (1989). An **RFC** (Request for Comments) is a numbered document published by the IETF (Internet Engineering Task Force), the group that writes Internet standards. RFC 1122 names the layers Application, Transport, Internet and Link.

The 5-layer version comes from textbooks. It keeps the top three layers and splits Link into Data Link and Physical. It also often says "Network" instead of "Internet". No RFC defines it, but it is very common because cables and signals deserve their own discussion.

```text
  OSI (7 layers)     TCP/IP 5-layer     TCP/IP 4-layer
+----------------+ +----------------+ +----------------+
| 7 Application  | |                | |                |
+----------------+ |                | |                |
| 6 Presentation | | Application    | | Application    |
+----------------+ |                | |                |
| 5 Session      | |                | |                |
+----------------+ +----------------+ +----------------+
| 4 Transport    | | Transport      | | Transport      |
+----------------+ +----------------+ +----------------+
| 3 Network      | | Network        | | Internet       |
+----------------+ +----------------+ +----------------+
| 2 Data Link    | | Data Link      | |                |
+----------------+ +----------------+ | Link           |
| 1 Physical     | | Physical       | |                |
+----------------+ +----------------+ +----------------+
```

**Which one is correct?** Both. The 4-layer version is the one written in the standard. The 5-layer version is a teaching choice. You may also see older names: "Network Access" or "Network Interface" for Link, and "DoD model" for the whole thing, because the work was funded by the US Department of Defense (DoD).

**Fuller truth:** the IETF does not treat these layers as strict law. Real protocols sometimes sit between layers. The layers are a helpful guide, not a set of walls.

### What Each Layer Does

**Application layer.** This is where programs talk to each other, such as a browser and a web server. It covers the meaning of the data: "give me this page", "here is an email". It also does the jobs of the OSI Session and Presentation layers. TCP/IP has no separate layers for them, so each application handles them itself. For example, cookies keep an HTTP session, and gzip compresses data.

**Transport layer.** This layer moves data between two programs on two machines. A **port** is a number from 0 to 65535 that tells the machine which program should get the data. Web servers normally listen on port 443 for HTTPS. TCP adds reliable delivery, and UDP (User Datagram Protocol) adds almost nothing. See [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md).

**Internet layer.** This layer moves packets from one machine to another across many networks. An **IP address** identifies a machine's connection to a network, such as 192.168.1.20. A **router** is a device that connects networks and forwards packets toward their destination. IP is **best effort**: it tries to deliver but gives no promise. Packets may be lost, arrive in the wrong order, or arrive twice. TCP fixes those problems at the ends.

**Link layer.** This layer moves data across one local link, such as one Ethernet cable or one Wi-Fi network. A **MAC address** is the hardware address used on that local link, such as 00:00:5e:00:53:01. A switch works at this layer. In the 5-layer version, the Data Link part handles frames and MAC addresses, and the Physical part handles the actual electrical, light or radio signals.

### Data Names at Each Layer

Each layer wraps the data from the layer above with its own **header**, a small block of control information at the front. This wrapping is called **encapsulation**. See [Encapsulation and Decapsulation](04-encapsulation.md) for the full process.

| Layer (4-layer) | Name of the data unit | Main address used |
|---|---|---|
| Application | Data or message | Names, such as example.com |
| Transport | Segment (TCP), datagram (UDP) | Port number |
| Internet | Packet | IP address |
| Link | Frame (bits on the wire) | MAC address |

### Mapping to OSI

| OSI layer | 4-layer TCP/IP | 5-layer TCP/IP | Fit |
|---|---|---|---|
| 7 Application | Application | Application | Merged |
| 6 Presentation | Application | Application | Merged |
| 5 Session | Application | Application | Merged |
| 4 Transport | Transport | Transport | Close match |
| 3 Network | Internet | Network | Close match |
| 2 Data Link | Link | Data Link | Merged in 4-layer |
| 1 Physical | Link | Physical | Merged in 4-layer |

The mapping is a good guide, not an exact match. Ethernet shows why: the IEEE 802.3 Ethernet standard covers both frames and signals, so it fits OSI layers 1 and 2 at the same time. Treating them as one Link layer is natural.

**Layer numbers:** when an engineer says "Layer 3" or "Layer 7", they mean OSI numbers, even when the protocols are TCP/IP. A "Layer 4 load balancer" works with TCP and UDP ports. A "Layer 7 load balancer" reads HTTP. This mix is normal in daily work.

### The Protocol Suite by Layer

| Layer | Protocols | Notes |
|---|---|---|
| Application | HTTP, DNS, DHCP, SMTP, IMAP, FTP, SSH, NTP, SNMP, TLS (usually placed here) | Runs on top of TCP, UDP or QUIC |
| Transport | TCP, UDP, QUIC, SCTP | QUIC runs on top of UDP but does transport work |
| Internet | IPv4, IPv6, ICMP, ICMPv6, IGMP, IPsec, OSPF (by function) | Moves packets and helps manage that movement |
| Link | Ethernet, Wi-Fi (IEEE 802.11), PPP, ARP, NDP (uses ICMPv6) | One local link at a time |
| Physical (5-layer only) | Copper, fibre, radio; for example 1000BASE-T | Signals, not software messages |

Some protocols do not sit neatly in one layer. These are favourite interview topics:

| Protocol | Common placement | Why it is tricky |
|---|---|---|
| ARP (Address Resolution Protocol) | Link | It finds a MAC address for an IP address, so it helps the Internet layer. It is carried directly in Ethernet, not in IP. RFC 1122 describes it in its Link Layer section. |
| ICMP (Internet Control Message Protocol, used for errors and ping) | Internet | It is carried inside IP (protocol number 1) but is treated as part of IP. It has no ports. |
| TLS (Transport Layer Security, which encrypts traffic) | Application | It sits between TCP and the application. It does not fit one layer in either model. |
| QUIC (a newer transport protocol) | Transport | It runs inside UDP, but provides connections and reliability. |
| OSPF (a routing protocol used inside one organisation) | Internet | It runs directly on IP (protocol number 89). It is a routing protocol, so by function it serves the Internet layer. |
| BGP (the routing protocol between networks on the Internet) | Application by carrier | It runs on TCP port 179, but its job is routing. |
| DNS (Domain Name System, which turns names into IP addresses) | Application | The whole Internet depends on it, but it is still an application. |

For a full list with port numbers, see the [Protocol Master List](../07-application-protocols/15-protocol-master-list.md).

**IPv6 note:** the model does not change with IPv6. IPv6 replaces IPv4 at the Internet layer, ICMPv6 replaces ICMP, and NDP (Neighbor Discovery Protocol) does the job of ARP. See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).

### Which Devices Use Which Layers

The top two layers run only on the two end machines. Routers in the middle stop at the Internet layer. A device at the edge of a network, such as your laptop or a server, is called a **host**.

```text
Host A                                          Host B
+-----------+                                   +-----------+
| App       |<---------- end to end ----------->| App       |
+-----------+                                   +-----------+
| Transport |<---------- end to end ----------->| Transport |
+-----------+     +----------+   +----------+   +-----------+
| Internet  |<--->| Internet |<->| Internet |<->| Internet  |
+-----------+     +----------+   +----------+   +-----------+
| Link      |<--->| Link     |<->| Link     |<->| Link      |
+-----------+     +----------+   +----------+   +-----------+
                    Router 1       Router 2
```

This shows the main design idea:

1. **Transport and Application are end to end.** Only the two hosts use them.
2. **Internet is host to host, across routers.** The IP addresses normally stay the same for the whole trip.
3. **Link is one hop at a time.** Each router removes the old frame and builds a new one, so the MAC addresses change at every hop.

**Fuller truth:** NAT (Network Address Translation) devices, firewalls and load balancers do read or change transport headers. A home router with NAT changes the source IP and port. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

### Why TCP/IP Won

Think of two teams building a road. One team opens a simple road that people can drive on today. The other team draws a perfect plan for a better road, and building it takes years. Drivers use the open road, and soon everything is built along it.

The same thing happened with networking:

1. **It came first.** TCP and IP were published in 1981 (RFC 791 and RFC 793). The ARPANET, an early US research network, switched to TCP/IP on 1 January 1983. The OSI model was published as a standard in 1984, and working OSI products came later.
2. **It was free and open.** Anyone could read RFCs at no cost. OSI documents were usually sold for a fee at that time.
3. **It shipped with an operating system.** 4.2BSD Unix (1983) included TCP/IP and the sockets programming interface, so universities got it for free. A **socket** is how a program opens a network connection. See [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md).
4. **It was built from running code.** The IETF culture is "rough consensus and running code": prove it works, then standardise it. OSI was designed by large committees before products existed, and it became big and full of options.
5. **Its core is simple.** IP only tries to deliver packets. The smart work happens in the hosts. Routers stay simple, and IP can run over almost any link.
6. **The Internet grew.** Once the Web arrived in the 1990s, everyone needed TCP/IP. Each new user made it more valuable.

**Where the road analogy stops being true:** the OSI work was not wasted. OSI's layer vocabulary is still used every day. IS-IS, a major routing protocol, came from the OSI world. X.509, the certificate format used by TLS, came from the related X.500 directory work.

**Fuller truth:** some governments did require OSI for a time in the late 1980s and early 1990s. The market still chose TCP/IP. Also, Ethernet won at the lower layers on its own, because neither model decides which hardware people buy.

### Comparison Table: OSI vs TCP/IP

| Point | OSI | TCP/IP |
|---|---|---|
| Number of layers | 7 | 4 in RFC 1122, 5 in many textbooks |
| Created by | ISO, an international standards body | ARPANET researchers, later the IETF |
| Key dates | Published 1984 (ISO 7498) | Protocols 1981, layer model written down 1989 |
| Order of work | Model first, protocols after | Protocols first, model after |
| Main use today | Teaching and shared vocabulary | Runs the real Internet |
| Session and Presentation | Separate layers | Handled inside the application |
| Physical and Data Link | Separate layers | One Link layer in the 4-layer version |
| Network layer service | Connectionless and connection-oriented | Connectionless only (IP) |
| Transport layer service | Mainly connection-oriented | Both: TCP (connection) and UDP (no connection) |
| Reliability | Can be built into lower layers | Mostly left to the end hosts |
| Style | Formal, strict, detailed | Practical, flexible, simple core |

"Connection-oriented" means the two sides set up a conversation before sending data. "Connectionless" means each piece of data is sent alone, with no setup.

## Example

### One Web Request Through the Four Layers

A laptop opens https://example.com. For this example, the name example.com resolves to 203.0.113.10.

| Item | Value |
|---|---|
| Laptop IP and mask | 192.168.1.20/24 |
| Laptop MAC | 00:00:5e:00:53:20 |
| Default gateway (router) | 192.168.1.1 |
| Gateway MAC | 00:00:5e:00:53:01 |
| Server IP | 203.0.113.10 |
| Source port (chosen by the laptop) | 51514 |
| Destination port | 443 |

What each layer adds on the way out:

| Layer | Adds | Values here |
|---|---|---|
| Application | HTTP request, protected by TLS | Host: example.com |
| Transport | TCP header | Source port 51514, destination port 443 |
| Internet | IPv4 header | Source 192.168.1.20, destination 203.0.113.10, protocol 6 (TCP) |
| Link | Ethernet header and trailer | Source MAC ...53:20, destination MAC ...53:01, type 0x0800 (IPv4) |

```text
+----------+-------------+------------+-----------------+------+
| Ethernet | IPv4 header | TCP header | TLS and HTTP    | FCS  |
| header   | 20 bytes    | 20 bytes   | data            |  4   |
| 14 bytes | or more     | or more    |                 | bytes|
+----------+-------------+------------+-----------------+------+
|  Link    |  Internet   | Transport  |  Application    | Link |
+----------+-------------+------------+-----------------+------+
```

FCS (Frame Check Sequence) is a checksum that lets the receiver detect a damaged frame.

Notice one detail. The destination IP is the server, but the destination MAC is the **router**. MAC addresses work only on the local link, so the laptop hands the frame to its [default gateway](../04-ip-addressing/08-default-gateway.md). It finds the gateway's MAC with [ARP](../03-ethernet-and-switching/05-arp.md).

### See the Layers on a Real Machine

Linux:

```bash
# Internet layer: my address
$ ip -br -4 addr show eth0
eth0             UP             192.168.1.20/24

# Internet layer: how I reach the server
$ ip route get 203.0.113.10
203.0.113.10 via 192.168.1.1 dev eth0 src 192.168.1.20 uid 1000
    cache

# Link layer: the gateway's MAC
$ ip neigh show 192.168.1.1
192.168.1.1 dev eth0 lladdr 00:00:5e:00:53:01 REACHABLE

# Transport layer: the open connection
$ ss -tn dst 203.0.113.10
State  Recv-Q Send-Q  Local Address:Port    Peer Address:Port
ESTAB  0      0        192.168.1.20:51514   203.0.113.10:443
```

Windows (PowerShell):

```powershell
# Internet layer: my address and gateway
PS> ipconfig
Ethernet adapter Ethernet:
   IPv4 Address. . . . . . . . . . . : 192.168.1.20
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

# Link layer: the gateway's MAC
PS> arp -a 192.168.1.1
Interface: 192.168.1.20 --- 0x7
  Internet Address      Physical Address      Type
  192.168.1.1           00-00-5e-00-53-01     dynamic

# Transport layer: the open connection
PS> Get-NetTCPConnection -RemoteAddress 203.0.113.10 |
>>   Format-Table LocalAddress,LocalPort,RemoteAddress,RemotePort,State

LocalAddress LocalPort RemoteAddress RemotePort       State
------------ --------- ------------- ----------       -----
192.168.1.20     51514 203.0.113.10         443 Established
```

### One Packet Showing Three Layers

On Linux, `tcpdump` can print the headers of each packet. See the [tcpdump chapter](../12-troubleshooting-and-tools/06-tcpdump.md).

```bash
sudo tcpdump -n -e -c 1 -i eth0 'host 203.0.113.10 and port 443'
```

This is the first packet of the connection. The output is one long line, wrapped here to fit:

```text
10:15:02.123456 00:00:5e:00:53:20 > 00:00:5e:00:53:01,
  ethertype IPv4 (0x0800), length 74:
  192.168.1.20.51514 > 203.0.113.10.443: Flags [S],
  seq 1234567890, win 64240,
  options [mss 1460,sackOK,TS val 1111 ecr 0,nop,wscale 7],
  length 0
```

| Part of the output | Layer |
|---|---|
| `00:00:5e:00:53:20 > 00:00:5e:00:53:01` | Link: source and destination MAC |
| `ethertype IPv4 (0x0800)` | Link: says what is inside the frame |
| `192.168.1.20` and `203.0.113.10` | Internet: source and destination IP |
| `.51514`, `.443`, `Flags [S]` | Transport: ports and the TCP SYN flag, which asks to start a connection |
| final `length 0` | Application: no data yet, because this is only the connection setup |

The size adds up: 14 bytes Ethernet + 20 bytes IP + 40 bytes TCP (20 fixed + 20 options) = 74 bytes.

### Four Everyday Services, Layer by Layer

| Layer | Web, HTTP/2 | Web, HTTP/3 | DNS lookup | ping |
|---|---|---|---|---|
| Application | HTTP/2, TLS | HTTP/3 | DNS | ping program |
| Transport | TCP 443 | QUIC (with TLS 1.3 built in), UDP 443 | UDP 53 (TCP 53 is also used) | None |
| Internet | IP | IP | IP | ICMP in IP |
| Link | Ethernet or Wi-Fi | Ethernet or Wi-Fi | Ethernet or Wi-Fi | Ethernet or Wi-Fi |

`ping` skips the Transport layer completely, which is why it has no port number. If `ping` works but a website does not, the Internet layer is fine, and the problem is higher up (or a firewall is treating the traffic differently). See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

## Going Deeper

### The Narrow Waist

Draw the protocols as a picture and it looks like an hourglass. Many applications sit at the top, many link types sit at the bottom, and there is one main protocol in the middle: IP.

```text
  HTTP  DNS  SMTP  SSH  NTP  ...      many applications
     \    \    |    /    /
        TCP  UDP  QUIC                a few transports
             \  |  /
                IP                    one narrow waist
             /  |  \
    Ethernet  Wi-Fi  5G  fibre  ...   many link types
```

Shipping containers are a good analogy. Any goods fit in a standard container, and any ship, train or truck can carry one. A new phone network needs to carry IP, and every application then works on it without a change.

The analogy stops here: a container never changes, but the waist is now slowly widening. Both IPv4 and IPv6 are in use, and they cannot talk directly to each other. The cost of a narrow waist is that changing it is very hard, which is why the move to IPv6 has taken decades.

### Where the Code Runs

On a normal Linux or Windows host, the layers map to parts of the system:

| Layer | Where it usually runs |
|---|---|
| Application | User programs and libraries (TLS is often a library such as OpenSSL) |
| Transport | Operating system kernel (QUIC is usually in the application) |
| Internet | Operating system kernel |
| Link | Device driver and network card |

The socket interface is the door between the application and the kernel. QUIC runs in user programs on top of UDP partly so that new versions can ship with an app update, with no need to change the kernel. See [QUIC](../06-transport-layer/08-quic.md).

### Layers Can Repeat

Tunnels put one full stack inside another. A VPN carries an inner IP packet inside an outer IP packet. [VXLAN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) carries an entire Ethernet frame inside UDP. So "Link" data can sit inside "Transport" data. The model still helps, but you apply it twice: once for the outer stack, once for the inner one.

## Common Mistakes

- **"TCP/IP means only TCP and IP."** It is a whole suite. UDP, ICMP, DNS and many others belong to it.
- **"The official model has 5 layers."** RFC 1122 defines 4. The 5-layer version is a common textbook form. Say both in an interview.
- **"The Internet layer means the whole Internet."** It is only the layer that moves packets across networks using IP.
- **"IP guarantees delivery."** IP is best effort. TCP adds reliability at the ends. UDP does not.
- **"All traffic uses TCP."** DNS queries normally use UDP, and HTTP/3 runs on QUIC over UDP.
- **"Layers map one to one with OSI."** Application merges three OSI layers, and real protocols such as TLS do not fit neatly.
- **"ping uses a port."** ICMP has no ports. You cannot "ping port 443" with normal ping. Use a tool such as `nc` or `Test-NetConnection`.
- **"ARP is an Internet layer protocol because it deals with IP addresses."** It is carried directly in Ethernet with no IP header, and it is normally placed in Link. If you say "between 2 and 3" and explain why, that is also accepted.
- **"Routers read TCP ports."** Plain routing only looks at the IP header. NAT, firewalls and load balancers look deeper.
- **"OSI is useless now."** It lost as a protocol suite but won as a vocabulary. "Layer 2" and "Layer 7" are OSI terms used every day.

## Interview Questions

**Q: What are the layers of the TCP/IP model?**

A: RFC 1122 defines four: Application, Transport, Internet and Link. Many textbooks use a five-layer version that splits Link into Data Link and Physical. The idea is the same in both.

**Q: How does TCP/IP map to the OSI model?**

A: The Application layer covers OSI layers 7, 6 and 5. Transport matches layer 4. Internet matches layer 3. Link covers layers 2 and 1. The mapping is a guide, not an exact match.

**Q: What is the main difference between OSI and TCP/IP?**

A: OSI is a seven-layer reference model that was designed first, and its protocols mostly failed in the market. TCP/IP started with working protocols, and the model was written down afterwards. TCP/IP is what runs the Internet, and OSI is mostly the vocabulary we use to talk about it.

**Q: Why did TCP/IP win over OSI?**

A: It was already working when OSI was still being finished. Its specifications were free, and it came with BSD Unix at no cost. Its core was simple, because IP just tries to deliver packets and the hosts do the smart work. Then the Internet grew on it, and that growth made the choice final.

**Q: Why does TCP/IP have no Session or Presentation layer?**

A: Most applications need these jobs in different ways, so TCP/IP leaves them to the application. For example, HTTP uses cookies for sessions, and formats such as JSON and compression such as gzip handle data presentation. TLS, which handles encryption, is usually used as a library by the application.

**Q: In which layer is ARP?**

A: Normally the Link layer. RFC 1122 describes it there, and it is carried directly in Ethernet with no IP header. Its job is to help IP by finding a MAC address, so many people say it sits between layers 2 and 3.

**Q: In which layer is ICMP, and does it use ports?**

A: ICMP is in the Internet layer. It is carried inside IP with protocol number 1, but it is treated as a part of IP. It has no ports. It uses type and code numbers instead.

**Q: Where does TLS fit?**

A: In the TCP/IP model it is usually placed in the Application layer, because applications use it directly. It really sits between the application and TCP. It does not fit exactly into one OSI layer either, although it does presentation-like work such as encryption.

**Q: What does "best effort" mean for IP?**

A: IP tries to deliver each packet but promises nothing. Packets can be lost, repeated or arrive out of order. This keeps routers simple, and hosts can use TCP to add reliability if they need it.

**Q: Which layers do a switch, a router and a host use?**

A: A basic switch uses the Link layer and forwards by MAC address. A router works up to the Internet layer and forwards by IP address. A host uses all the layers, because the transport and application work happens at the two ends.

**Q: What changes and what stays the same as a packet crosses routers?**

A: The source and destination IP addresses normally stay the same. The MAC addresses change at each hop, because each router builds a new frame. The TTL (Time To Live, a hop limit in the IP header) also drops by one at each hop. NAT is the common exception, because it changes the addresses and ports.

**Q: What is the "narrow waist" of the Internet?**

A: It is the idea that IP sits in the middle of an hourglass. Many applications sit above it and many link types sit below it. New applications and new link technologies can appear without changing each other, as long as they all work with IP.

## Quick Recap

- The TCP/IP model has 4 layers in RFC 1122 (Application, Transport, Internet, Link). The 5-layer textbook version splits Link into Data Link and Physical.
- Application covers OSI 7, 6 and 5. Transport is OSI 4. Internet is OSI 3. Link is OSI 2 and 1.
- Data units are message, segment or datagram, packet, and frame. Addresses are ports, IP addresses and MAC addresses.
- Transport and Application run end to end. Routers stop at the Internet layer. MAC addresses change at each hop.
- TCP/IP won because it came first, was free, shipped with Unix, and had a simple core.
- OSI is the vocabulary. TCP/IP is the working system.
- ARP, ICMP, TLS, QUIC and BGP do not fit neatly in one layer, so explain your reasoning.
