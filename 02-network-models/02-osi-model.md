# The OSI Model

**Level:** Beginner | **Prerequisites:** [Protocols, Standards and Why We Use Layers](01-why-layers-and-protocols.md), [Network Devices](../01-foundations/04-network-devices.md)

## In One Line

The OSI model splits network communication into 7 layers, each with one job, so people can design, explain and fix networks one piece at a time.

## Why It Matters

Opening one web page involves cables, radio waves, addresses, routes, lost data, encryption and file formats. Nobody can think about all of that at once.

OSI solves this by cutting the problem into 7 parts. A cable engineer can work on Layer 1 without knowing HTTP. A web developer can work on Layer 7 without knowing how light moves through fibre.

OSI stands for Open Systems Interconnection. It was published by ISO (the International Organization for Standardization). The current text is ISO/IEC 7498-1, which matches ITU-T X.200.

OSI is a **reference model**: a shared map of how networking is organised. It is not a program you install. The internet actually runs on the TCP/IP protocols, but engineers still use OSI numbers every day.

You will hear phrases like these at work:

- "This is a Layer 2 problem."
- "We need a Layer 7 load balancer."
- "The firewall blocks it at Layer 4."

Interviewers ask about OSI for three reasons. It shows you know the vocabulary. It shows you know where each device and protocol fits. Most of all, it shows you can find a fault by working through the layers, not by guessing.

## Simple Explanation

Imagine a company in Karachi sending a large order to a company in Berlin.

| Layer | Who does it in the company | What they do |
|---|---|---|
| 7 Application | The manager | Decides what to say: "Please send 500 units." |
| 6 Presentation | The translator | Translates to German, shrinks the file, locks it in a secure envelope |
| 5 Session | The personal assistant | Opens the conversation, keeps track of it, closes it |
| 4 Transport | The shipping department | Splits the order into numbered boxes, tracks them, resends lost ones |
| 3 Network | The national post system | Reads the city address and picks a route across countries |
| 2 Data Link | The van driver | Moves the boxes from one depot to the next depot only |
| 1 Physical | The road | Carries the van |

Each person only talks to the person above and below. The manager never thinks about roads. The driver never reads the letter. In Berlin, the same steps happen in reverse order.

**Where the analogy stops being true:** a real computer does all of this in microseconds. Also, a real post office tries hard to deliver every box, but Layer 3 may drop data with no warning. Real layers are software and hardware, not people.

## How It Works

### The Seven Layers at a Glance

A **protocol** is an agreed set of rules for talking. A **PDU** (protocol data unit) is the name for a piece of data at a given layer.

| # | Layer | Main job | PDU | Address used | Example protocols | Example devices |
|---|---|---|---|---|---|---|
| 7 | Application | Give apps network services | Data | Hostname, URL | HTTP, DNS, SMTP, SSH | Proxy, L7 load balancer |
| 6 | Presentation | Format, compress, encrypt | Data | None | UTF-8, JPEG, gzip, TLS (commonly) | Host software |
| 5 | Session | Open, manage, close sessions | Data | Session ID | RPC, NetBIOS | Host software |
| 4 | Transport | App-to-app delivery | Segment (TCP), datagram (UDP) | Port | TCP, UDP | L4 firewall, L4 load balancer |
| 3 | Network | Move data between networks | Packet | IP address | IPv4, IPv6, ICMP | Router, L3 switch |
| 2 | Data Link | Move data on one local link | Frame | MAC address | Ethernet, Wi-Fi, PPP | Switch, bridge, access point |
| 1 | Physical | Move raw bits as signals | Bit | None | 1000BASE-T, 10GBASE-SR | Cable, hub, repeater |

Layers 1-3 are often called the **media layers**, because they move data. Layers 4-7 are often called the **host layers**, because they mostly run on the end computers.

### How Data Moves Through the Layers

When you send data, it goes down the layers. Each layer adds its own **header**, a block of control information placed in front of the data. This wrapping is called **encapsulation**.

```text
 L5-7                               +-----------+
 Data                               |   Data    |
                                    +-----------+
 L4                       +---------+-----------+
 Segment                  | TCP hdr |   Data    |
                          +---------+-----------+
 L3             +---------+---------+-----------+
 Packet         | IP hdr  | TCP hdr |   Data    |
                +---------+---------+-----------+
 L2   +---------+---------+---------+-----------+-----+
 Frame| Eth hdr | IP hdr  | TCP hdr |   Data    | FCS |
      +---------+---------+---------+-----------+-----+
 L1
 Bits  1 0 1 1 0 0 1 0 1 1 1 0 0 1 0 1 0 0 1 1 0 ...
```

The receiver goes up the layers and removes each header. That is **decapsulation**. Each layer reads only the header written by the same layer on the other side. Full detail is in [Encapsulation and Decapsulation](04-encapsulation.md).

So every layer has two relationships:

1. It **uses the service** of the layer directly below it.
2. It **talks logically** to the same layer on the other machine.

### Layer 1: Physical

**Purpose:** send and receive raw bits. A **bit** is a single 0 or 1. Layer 1 turns bits into signals: electricity in copper, light in fibre, radio waves in Wi-Fi.

Layer 1 defines the things you can touch and measure: connectors, pin layout, voltages, light wavelengths, radio frequencies and bit speed.

- **PDU:** bit.
- **Addressing:** none. Layer 1 has no idea who the data is for.
- **Standards:** 1000BASE-T (1 Gbps on copper), 10GBASE-SR (10 Gbps on fibre), the radio part of Wi-Fi.
- **Devices:** cables, connectors such as RJ45, fibre transceivers, repeaters, hubs. Modems are mostly here too (fuller truth: they also do some Layer 2 work).

**Real example:** a 1000BASE-T Ethernet link on copper can run up to 100 m. Go much past that, or crush the cable, and signals weaken. The bits arrive damaged. See [Cables and Transmission Media](../01-foundations/05-cables-and-media.md).

### Layer 2: Data Link

**Purpose:** move data between two devices on the same local link, and detect damaged data.

A group of bits with a header and a trailer is a **frame**. Layer 2 uses a **MAC address**, a 48-bit hardware address such as `00:00:5e:00:53:10`.

```text
 Ethernet frame (without preamble, sizes in bytes)
 +---------+---------+------+------------------+-----+
 | Dst MAC | Src MAC | Type | Payload          | FCS |
 |    6    |    6    |  2   | 46-1500          |  4  |
 +---------+---------+------+------------------+-----+
```

**FCS** (frame check sequence) is a number calculated from the frame. The receiver calculates it again. If the numbers differ, the frame is damaged and is dropped. Ethernet detects errors but does not fix them. (Fuller truth: Wi-Fi does resend lost frames at Layer 2, because radio loses much more data.)

Layer 2 has two sublayers:

- **LLC** (logical link control): the upper part, which identifies what is carried inside.
- **MAC** (media access control): the lower part, which handles addresses and access to the medium.

The details:

- **PDU:** frame.
- **Addressing:** MAC address. It works only on the local link.
- **Protocols:** Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11), PPP (for a direct link between two devices), 802.1Q VLAN tags (they split one switch into separate networks), STP (it stops loops), LACP (it bundles links together).
- **Devices:** switches, bridges, wireless access points. A **NIC** (network interface card) works at Layers 1 and 2.

**Real example:** a switch learns which MAC address lives on which port, then sends each frame only to the right port.

```text
Cisco IOS

Switch# show mac address-table dynamic
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    0000.5e00.5310    DYNAMIC     Gi0/1
  10    0000.5e00.5301    DYNAMIC     Gi0/24
```

Learn more in [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md).

### Layer 3: Network

**Purpose:** move data from one network to another, across many links. Layer 2 handles one link. Layer 3 handles the whole journey.

Data at this layer is a **packet**. Each packet carries a source and destination **IP address**, a logical address given by the network, not burned into hardware. **Routing** means choosing the path. A **router** is a device that forwards packets between networks. Each router on the way is called a **hop**.

IP is **best effort**: it tries to deliver but gives no promise. Lost packets are not recovered at this layer. Each IPv4 packet also has a **TTL** (time to live), a counter lowered by 1 at every router. When it reaches 0, the packet is dropped. This stops packets looping forever.

- **PDU:** packet.
- **Addressing:** IPv4 (32 bits, like `192.168.1.50`), IPv6 (128 bits, like `2001:db8::50`).
- **Protocols:** IPv4, IPv6, ICMP (error and test messages, used by `ping`), IPsec (it encrypts IP packets). Routing protocols such as OSPF, which let routers share routes, are usually grouped here.
- **Devices:** routers, Layer 3 switches, many firewalls.

**Real example:** a laptop with `192.168.1.50/24` wants to reach `203.0.113.10`. That address is outside its own network, so the laptop sends the packet to its default gateway, `192.168.1.1`.

```bash
# Linux
$ ip route
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50
```

```powershell
# Windows
Get-NetRoute -DestinationPrefix 0.0.0.0/0
```

**The key idea:** MAC addresses change at every hop. IP addresses stay the same.

| Link | Source MAC | Destination MAC | Source IP | Destination IP |
|---|---|---|---|---|
| Laptop to R1 | 00:00:5e:00:53:10 | 00:00:5e:00:53:01 | 192.168.1.50 | 203.0.113.10 |
| R1 to R2 | 00:00:5e:00:53:02 | 00:00:5e:00:53:03 | 192.168.1.50 | 203.0.113.10 |
| R2 to server | 00:00:5e:00:53:04 | 00:00:5e:00:53:20 | 192.168.1.50 | 203.0.113.10 |

Think of a passport and bus tickets. Your passport name (IP) stays the same for the whole trip. You get a new ticket (MAC) for each bus. The analogy stops here: IP does have an exception, which is NAT.

**Fuller truth:** a private address like `192.168.1.50` cannot travel on the public internet. In real life, R1 would use [NAT](../04-ip-addressing/09-nat-and-pat.md) to replace the source IP with a public one, such as `198.51.100.7`. The destination IP stays the same.

### Layer 4: Transport

**Purpose:** deliver data from one application on a machine to one application on another machine. Layer 3 gets data to the right computer. Layer 4 gets it to the right program.

Layer 4 uses a **port**, a 16-bit number (0-65535) that identifies a program or service. HTTPS servers usually listen on port 443. Your browser picks a temporary port, such as 51514.

Two main protocols live here:

| | TCP | UDP |
|---|---|---|
| Connection | Yes, set up first | No |
| Lost data | Resent | Not resent |
| Order | Kept | Not guaranteed |
| Speed control | Flow and congestion control | None |
| PDU name | Segment | Datagram |
| Used by | Web, email, SSH | DNS queries, voice, video |

TCP also splits large data into pieces. On a normal Ethernet path, one segment often holds up to 1460 bytes of data (1500 minus 20 for IP, minus 20 for TCP). So 5000 bytes become 4 segments: 1460, 1460, 1460 and 620. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

- **PDU:** segment (TCP), datagram (UDP).
- **Addressing:** ports.
- **Protocols:** TCP, UDP, SCTP. QUIC does transport work but runs on top of UDP.
- **Devices:** mainly end hosts. Stateful firewalls, NAT devices and Layer 4 load balancers also read this layer.

**Real example:** two browser tabs open to the same server. The IPs match. The source ports tell the two conversations apart.

```bash
# Linux
$ ss -tn
State  Recv-Q Send-Q Local Address:Port    Peer Address:Port
ESTAB  0      0       192.168.1.50:51514   203.0.113.10:443
ESTAB  0      0       192.168.1.50:51516   203.0.113.10:443
```

```powershell
# Windows
Get-NetTCPConnection -State Established
```

A connection is identified by five values, called the **5-tuple**: protocol, source IP, source port, destination IP, destination port.

### Layer 5: Session

**Purpose:** open, manage and close a long conversation (a **session**) between two applications.

Layer 5 has three ideas:

- **Set up and tear down:** start a conversation and end it cleanly.
- **Dialog control:** decide who may send, one side at a time (half duplex) or both together (full duplex).
- **Checkpoints:** mark safe points, so work can continue from there after a failure.

The details:

- **PDU:** data.
- **Addressing:** no standard address. Programs use a session ID.
- **Protocols usually listed here:** RPC (remote procedure call), NetBIOS session service, SOCKS. SIP (which sets up voice calls) is often listed at Layers 5-7.
- **Devices:** none. This is host software.

**Real example:** a video call keeps audio and video in step, and rejoins if your network drops for a moment. A large file download can restart from the last good point, not from zero.

**Fuller truth:** TCP/IP has no separate session layer. Sessions are built by TCP plus the application. A login cookie is an application feature, but it shows the same idea: the server remembers an ongoing conversation.

### Layer 6: Presentation

**Purpose:** make data readable by the other side. This layer deals with the *form* of the data, not its meaning.

Layer 6 does three jobs:

1. **Translation:** agree on how data is written as bytes. The letter `A` is `0x41` in ASCII and UTF-8, but `0xC1` in the older EBCDIC.
2. **Compression:** make data smaller, as with gzip or JPEG.
3. **Encryption:** scramble data so only the right person can read it.

The details:

- **PDU:** data.
- **Addressing:** none.
- **Formats and protocols:** ASCII, UTF-8, JPEG, PNG, MP3, MPEG, gzip, JSON, XML, ASN.1, MIME, and TLS in most training material.
- **Devices:** none. This is host software.

**Real example:** these HTTP headers describe Layer 6 work, even though HTTP itself is Layer 7.

```text
Content-Type: application/json; charset=utf-8
Content-Encoding: gzip
```

**Fuller truth:** real applications do this work themselves, in libraries. TLS does not fit neatly at Layer 6 (see [Limits of the Model](#limits-of-the-model)).

### Layer 7: Application

**Purpose:** give programs network services, such as fetching a page, sending mail or finding a name. This is the layer closest to the user.

**Layer 7 is not the application itself.** Chrome is a program. HTTP, which Chrome uses, is the Layer 7 protocol.

- **PDU:** data (often just called a message).
- **Addressing:** hostnames, URLs, email addresses.
- **Devices:** end hosts, web proxies, Layer 7 load balancers, web application firewalls.

| Protocol | Common port | Job |
|---|---|---|
| HTTP / HTTPS | TCP 80 / 443 | Web |
| DNS | UDP and TCP 53 | Turns names into addresses |
| DHCP | UDP 67, 68 | Gives hosts IP settings |
| SMTP | TCP 25, 587 | Sends email |
| SSH | TCP 22 | Secure remote login |
| FTP | TCP 21 | File transfer |
| SNMP | UDP 161 | Device monitoring |
| NTP | UDP 123 | Time sync |

**Real example:** this short conversation is Layer 7. Everything below it is hidden from the browser.

```text
GET / HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
Content-Type: text/html
```

### Mnemonics

Pick one sentence and practise it until it is automatic.

| Direction | Sentence | Letters |
|---|---|---|
| Layer 7 down to 1 | **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing | Application, Presentation, Session, Transport, Network, Data Link, Physical |
| Layer 1 up to 7 | **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way | Physical, Data Link, Network, Transport, Session, Presentation, Application |
| PDUs, top down | **D**o **S**ome **P**eople **F**ear **B**irthdays | Data, Segment, Packet, Frame, Bit |

For devices, remember the count: **hub 1, switch 2, router 3**.

### Where Common Things Sit

| Thing | Layer | Note |
|---|---|---|
| Cable, fibre, hub, repeater | 1 | Only signals |
| NIC | 1, 2 | Also has the MAC address |
| Switch, bridge | 2 | Forwards by MAC |
| Wireless access point | 1, 2 | Bridges Wi-Fi to Ethernet |
| VLAN, 802.1Q, STP, LACP | 2 | Local network features |
| ARP | Between 2 and 3 | Maps IP to MAC; has no IP header |
| MPLS | "2.5" | Label sits between Layer 2 and 3 headers |
| Router | 3 | Also has Layers 1 and 2 on each port |
| Layer 3 switch | 2, 3 | A switch that can also route |
| IP, ICMP | 3 | ICMP is carried inside IP but supports Layer 3 |
| OSPF, EIGRP | 3 | Run directly over IP (protocols 89 and 88) |
| IPsec | 3 | Protects IP packets |
| NAT | 3 | PAT also changes Layer 4 ports |
| TCP, UDP | 4 | Ports live here |
| QUIC | 4 and above | Runs over UDP, includes TLS 1.3 |
| BGP | 7 by carrier, 3 by job | Runs over TCP port 179, but its job is routing |
| TLS | Between 4 and 7 | Usually answered as Layer 6 |
| DNS, DHCP, NTP, SNMP | 7 | They run over UDP or TCP |
| HTTP, SMTP, SSH, FTP | 7 | Application protocols |
| Packet-filter or stateful firewall | 3, 4 | Reads IPs and ports |
| Next-generation firewall | 3-7 | Can identify applications |
| Load balancer | 4 or 7 | Depends on the type |
| Reverse proxy, WAF | 7 | Reads HTTP |
| VPN | Depends | IPsec is 3, TLS VPNs run higher |
| Home "router" | 1-7 | One box: router, switch, access point, NAT, DHCP |

### How Far Each Device Reads

Devices in the middle read only as high as they need. Only the two ends use all 7 layers.

```text
 Laptop                                                  Server
 +---+                                                   +---+
 | 7 | <----------- apps talk end to end ------------->  | 7 |
 | 6 |                                                   | 6 |
 | 5 |                                                   | 5 |
 | 4 |                                                   | 4 |
 | 3 |                      Router                       | 3 |
 | 2 |      Switch          +---+        Switch          | 2 |
 | 1 |      +---+           | 3 |        +---+           | 1 |
 +---+      | 2 |           | 2 |        | 2 |           +---+
   |        | 1 |           | 1 |        | 1 |             |
   |        +---+           +---+        +---+             |
   |          |               |            |               |
   +----------+---------------+------------+---------------+
```

**Fuller truth:** middle boxes such as firewalls, NAT devices and load balancers do read higher layers. The diagram shows a simple path.

### Limits of the Model

OSI is a teaching map, not a description of the real internet. Interviewers respect a candidate who says so.

- **The internet uses TCP/IP, not OSI.** The real OSI protocol suite lost out. See the mapping below and [The TCP/IP Model](03-tcp-ip-model.md).
- **Layers 5 and 6 are rarely separate.** Applications do this work themselves, mostly inside libraries.
- **Some protocols do not fit.** ARP, MPLS, TLS and BGP all sit awkwardly.
- **QUIC merges layers.** It does transport, session-like and encryption work together. See [QUIC](../06-transport-layer/08-quic.md).
- **Tunnels repeat layers.** A tunnel carries one packet or frame inside another. VXLAN puts a whole Layer 2 frame inside UDP (port 4789). The stack then has two Layer 2s, two Layer 3s and more.
- **Devices cross layers.** Layer 3 switches, next-generation firewalls and NAT devices work on several layers at once.
- **Some layers break the rules on purpose.** NAT changes Layer 3 and Layer 4 fields together. TCP checksums even include some IP header fields.

```text
 OSI                     TCP/IP (common 4-layer form)
 7 Application    \
 6 Presentation    >---  Application
 5 Session        /
 4 Transport      -----  Transport
 3 Network        -----  Internet
 2 Data Link      \
 1 Physical       /---   Link
```

### Troubleshooting by Layer

Layers give you a checklist. If Layer 3 works, then Layers 1 and 2 must work on that path too, so you can stop looking there.

There are three common approaches:

1. **Bottom up:** start at Layer 1 and climb. Best when many things are broken.
2. **Top down:** start at the application and go lower. Best when only one app has a problem.
3. **Divide and conquer:** start in the middle, usually with `ping`. Pass means look higher, fail means look lower. This is often the fastest.

| Layer | Typical symptoms | Linux | Windows | Cisco IOS |
|---|---|---|---|---|
| 1 Physical | No link light, port down, errors rising | `ip link`, `ethtool eth0` | `Get-NetAdapter` | `show interfaces` |
| 2 Data Link | Link up but no ARP reply, wrong VLAN, blocked port | `ip neigh` | `arp -a` | `show mac address-table`, `show vlan brief` |
| 3 Network | Wrong IP, mask or gateway, no route, `169.254.x.x` address | `ip addr`, `ip route`, `ping`, `traceroute` | `ipconfig /all`, `ping`, `tracert` | `show ip route`, `show ip interface brief` |
| 4 Transport | Connection refused or timed out | `ss -tn`, `nc -vz host port` | `Test-NetConnection -Port` | `show access-lists` |
| 5 Session | Drops after idle time, login sessions expire | Application logs | Application logs | `show ip nat translations` |
| 6 Presentation | Certificate errors, TLS version mismatch, garbled text | `openssl s_client` | Browser certificate view | Not usual |
| 7 Application | HTTP 404, 500, 503, DNS name not found | `curl -v`, `dig` | `curl.exe -v`, `nslookup` | Not usual |

Cisco's interface status line gives you two layers at once:

| Status line | Meaning |
|---|---|
| `up, line protocol is up` | Layers 1 and 2 are fine |
| `down, line protocol is down` | Layer 1 problem: cable, far end, optic |
| `up, line protocol is down` | Often Layer 2: for example, mismatched encapsulation |
| `administratively down` | Someone typed `shutdown` |

Two more clues help at Layer 4. **Connection refused** usually means you reached the host but nothing is listening on that port. **Timed out** usually means something dropped your traffic silently, often a firewall.

The full method is in [Troubleshooting Methodology](../12-troubleshooting-and-tools/01-troubleshooting-method.md).

## Example

### Example 1: Finding All Layers in One Real Packet

The laptop `192.168.1.50` opens `https://www.example.com`, which resolves to `203.0.113.10`. Here is one packet as a capture tool such as [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) shows it:

```text
Frame 7: 583 bytes on wire
Ethernet II, Src: 00:00:5e:00:53:10, Dst: 00:00:5e:00:53:01
Internet Protocol Version 4, Src: 192.168.1.50, Dst: 203.0.113.10
Transmission Control Protocol, Src Port: 51514, Dst Port: 443
Transport Layer Security
    TLS Record Layer: Handshake Protocol: Client Hello
        Server Name: www.example.com
```

Read it line by line:

| Line | Layer | What it tells you |
|---|---|---|
| Frame 7: 583 bytes | 1 | Bits on the wire: 14 Ethernet, 20 IP, 32 TCP, 517 TLS |
| Ethernet II | 2 | Destination MAC is the gateway, not the server |
| Internet Protocol | 3 | Final destination is the server |
| TCP, ports | 4 | Port 443 picks the HTTPS service |
| TLS | 5-6 | Security being set up for this session |
| Server Name | 7 | Which website the browser wants |

Note the Layer 2 destination. The laptop knows the server is not local, so it addresses the frame to its gateway. The IP destination still names the server. The 32 bytes of TCP are the 20-byte basic header plus 12 bytes of options.

### Example 2: Troubleshooting a Site That Will Not Open

A user says "`https://www.example.com` is down." The laptop is `192.168.1.50/24`, the gateway is `192.168.1.1`, and the site is at `203.0.113.10`. We go bottom up.

**Layer 1: is the link up?**

```bash
# Linux
$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:00:5e:00:53:10 brd ff:ff:ff:ff:ff:ff
```

`LOWER_UP` means a physical link exists. Layer 1 passes.

**Layer 2: can we reach the gateway's MAC?**

```bash
# Linux
$ ip neigh show 192.168.1.1
192.168.1.1 dev eth0 lladdr 00:00:5e:00:53:01 REACHABLE
```

```powershell
# Windows
PS> arp -a

Interface: 192.168.1.50 --- 0xc
  Internet Address      Physical Address      Type
  192.168.1.1           00-00-5e-00-53-01     dynamic
```

The gateway's MAC is known. Layer 2 passes.

**Layer 3: can we reach the server's IP?**

```bash
# Linux
$ ping -c 3 203.0.113.10
PING 203.0.113.10 (203.0.113.10) 56(84) bytes of data.
64 bytes from 203.0.113.10: icmp_seq=1 ttl=54 time=18.4 ms
64 bytes from 203.0.113.10: icmp_seq=2 ttl=54 time=18.1 ms
64 bytes from 203.0.113.10: icmp_seq=3 ttl=54 time=18.7 ms

--- 203.0.113.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 18.100/18.400/18.700/0.245 ms
```

Layer 3 passes. Many people stop here and say "the network is fine." That is too early.

**Layer 4: is port 443 reachable?**

```bash
# Linux
$ nc -vz -w 5 203.0.113.10 443
nc: connect to 203.0.113.10 port 443 (tcp) timed out: Operation now in progress
```

```powershell
# Windows
PS> Test-NetConnection 203.0.113.10 -Port 443
WARNING: TCP connect to (203.0.113.10 : 443) failed

ComputerName           : 203.0.113.10
RemoteAddress          : 203.0.113.10
RemotePort             : 443
InterfaceAlias         : Ethernet
SourceAddress          : 192.168.1.50
PingSucceeded          : True
PingReplyDetails (RTT) : 18 ms
TcpTestSucceeded       : False
```

Layer 4 fails, and it is a timeout, not a refusal. So we suspect a silent drop. We ask the server owner to check the web service:

```bash
# Linux, on the server
$ ss -ltn 'sport = :443'
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       511           0.0.0.0:443        0.0.0.0:*
```

The service is listening, so the server application is fine. The firewall in front of the server has no rule that allows TCP 443. The team adds the rule, and we test again:

```bash
# Linux
$ nc -vz -w 5 203.0.113.10 443
Connection to 203.0.113.10 443 port [tcp/https] succeeded!

$ curl -sI https://www.example.com | head -n 1
HTTP/2 200
```

Layer 7 now answers. The fault was found in five commands, and no one had to guess.

## Going Deeper

### Attacks by Layer

Interviewers often ask for one attack per layer. Each one has its own defence. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

| Layer | Example attack | Example defence |
|---|---|---|
| 1 | Cable tapping, radio jamming | Locked rooms, physical security |
| 2 | ARP spoofing, MAC flooding | Dynamic ARP inspection, port security |
| 3 | IP spoofing, ICMP flood | Filtering, rate limits |
| 4 | SYN flood | SYN cookies |
| 5 | Session hijacking | Secure session tokens, TLS |
| 6 | Forced downgrade to weak encryption | Modern TLS, HSTS |
| 7 | SQL injection, HTTP flood | WAF, input checks |

A **WAF** (web application firewall) reads HTTP and blocks bad requests. **HSTS** is a header that tells browsers to use only HTTPS for a site. **SYN cookies** let a server survive a flood of half-open TCP connections without storing state for each one.

### Layers on a Real Computer

On a typical host, the work is divided like this:

| Layers | Usually handled by |
|---|---|
| 1 and 2 | NIC hardware and its driver |
| 3 and 4 | Operating system kernel |
| 5, 6 and 7 | The application and its libraries |

**Fuller truth:** this line moves. NICs can take on some TCP work (offload), and QUIC usually runs in the application, not the kernel.

### What Happened to the Real OSI Protocols

OSI was more than a model. It was also a full protocol family, built to compete with TCP/IP. TCP/IP won in the market, but parts of OSI survive:

- **IS-IS**, a routing protocol still used by large networks. It runs directly over Layer 2, not over IP. See [IS-IS](../05-routing/08-is-is.md).
- **X.509**, the certificate format used in TLS, comes from the OSI X.500 directory work.
- **LDAP** grew out of X.500 as a lighter way to reach a directory.
- **ASN.1**, a data description language, is used by SNMP and by certificates.

### Layer 4 vs Layer 7 Decisions

The layer a device reads decides what it can do. A Layer 4 load balancer sees only IPs and ports, so it is fast but simple. A Layer 7 load balancer reads HTTP, so it can route `/api` and `/images` to different servers, but it does more work and usually must decrypt TLS. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

## Common Mistakes

- **"OSI is a protocol."** It is a model. No packet on the internet is an "OSI packet".
- **"Chrome is a Layer 7 application."** Chrome is a program. HTTP is the Layer 7 protocol it uses.
- **"A router only has Layer 3."** It has Layers 1 and 2 on each port as well. It must, to send and receive frames.
- **"All switches are Layer 2."** Layer 3 switches also route.
- **"DNS is Layer 3 because it gives you IP addresses."** DNS is a Layer 7 protocol. What it talks about does not set its layer.
- **"DHCP is Layer 3 because it assigns IPs."** Same mistake. DHCP is Layer 7, over UDP.
- **"ICMP is Layer 4 because it sits inside IP."** It is treated as Layer 3, because it supports IP. It has no ports.
- **"The MAC address stays the same end to end."** The MAC changes at every hop. The IP stays the same, unless NAT changes it.
- **"Ping works, so the application works."** Ping tests Layer 3 only. A firewall can still block the port, and the service can still be down.
- **"Ping fails, so the host is down."** Many firewalls block ICMP. The host may be fine.
- **"Segment, packet and frame mean the same."** They are different layers. People use "packet" loosely in speech, but use the exact word in an interview.
- **"TLS is exactly Layer 6."** That is the common training answer, but it runs on TCP and below HTTP, and it does not fit one layer cleanly.

## Interview Questions

**Q: What is the OSI model and why do we use it?**

A: It is a 7-layer reference model from ISO that describes how network communication is organised. Each layer has one job and serves the layer above. We use it as a shared language and as a checklist for troubleshooting, even though the internet itself runs on TCP/IP.

**Q: Name the 7 layers and their PDUs.**

A: From the top: Application, Presentation and Session use data. Transport uses segments for TCP and datagrams for UDP. Network uses packets, Data Link uses frames, and Physical uses bits.

**Q: What is the difference between a frame, a packet and a segment?**

A: A segment is Layer 4 and carries ports. A packet is Layer 3 and carries IP addresses. A frame is Layer 2 and carries MAC addresses. Each one wraps the one above it.

**Q: At which layer do a hub, a switch and a router work?**

A: A hub works at Layer 1 and repeats signals out of every port. A switch works at Layer 2 and forwards by MAC address. A router works at Layer 3 and forwards by IP address. A Layer 3 switch does both of the last two.

**Q: Which layer is ARP at?**

A: It sits between Layers 2 and 3. It maps an IP address to a MAC address, so it serves Layer 3. But it travels directly in an Ethernet frame with no IP header, so many people call it Layer 2.

**Q: Which layer is TLS at?**

A: The usual exam answer is Layer 6, because it encrypts data. In practice it sits between TCP and the application, and it also handles session-like setup. So I would say Layer 6 by convention, but it does not map cleanly.

**Q: Does the IP address or the MAC address change as a packet crosses the internet?**

A: The MAC addresses change at every hop, because each frame only covers one link. The IP addresses stay the same end to end, unless a NAT device changes them.

**Q: Why is OSI still taught when the internet uses TCP/IP?**

A: Because it is a clear way to explain and separate functions. Engineers still say Layer 2, Layer 4 and Layer 7 every day. Those numbers come from OSI, and it remains a good tool for finding faults.

**Q: What are the limits of the OSI model?**

A: Real protocols do not fit it perfectly. Layers 5 and 6 are normally built into applications. ARP, MPLS, TLS and QUIC cross layers. Tunnels such as VXLAN repeat layers, and devices like next-generation firewalls work across many layers.

**Q: Ping works but the website does not open. How do you troubleshoot?**

A: Ping proves Layer 3 works, so I move up. I test the port with `nc` or `Test-NetConnection`. A timeout suggests a firewall drop, and a refusal suggests no service is listening. Then I check TLS and use `curl -v` to see the HTTP response. I also confirm DNS returns the right IP.

**Q: What is the difference between a Layer 4 and a Layer 7 load balancer?**

A: A Layer 4 load balancer decides using IPs and ports, so it is fast but cannot see the content. A Layer 7 load balancer reads the HTTP request, so it can route by URL or header, at the cost of more processing.

**Q: How do you check Layers 1 and 2 on a Cisco interface?**

A: I run `show interfaces`. The first status is Layer 1 and the line protocol is Layer 2. Up and up is healthy. Down and down points to a cable or the far end. Up and down often points to a Layer 2 problem.

## Quick Recap

- OSI is a 7-layer reference model, not a protocol: Physical, Data Link, Network, Transport, Session, Presentation, Application.
- PDUs from the top: data, segment or datagram, packet, frame, bit.
- Addresses by layer: ports at 4, IP at 3, MAC at 2. MAC changes each hop, IP stays the same unless NAT changes it.
- Devices: hub 1, switch 2, router 3, with firewalls and load balancers working higher.
- ARP, TLS, BGP, MPLS and QUIC do not fit neatly, and the real internet runs on TCP/IP.
- To troubleshoot, test layer by layer. Ping only proves Layer 3.
