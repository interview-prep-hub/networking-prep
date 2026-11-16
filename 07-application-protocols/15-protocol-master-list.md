# Protocol Master List

**Level:** Beginner | **Prerequisites:** [The OSI Model](../02-network-models/02-osi-model.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md)

This page is one big lookup table for every protocol in the course. Use it to answer the four questions interviewers ask most: What does it stand for? What layer is it? What port does it use? What is the secure version?

Do not try to read it in one sitting. Learn the tables you need, then come back when you forget a number.

Think of it as a phone book. It tells you who everyone is and how to reach them. It does not tell you their life story. For the full story, follow the links to each chapter.

## How To Read The Tables

### The words in the table headings

- **Protocol:** an agreed set of rules for how two devices talk. Like the rules of a phone call: say hello, take turns, say goodbye.
- **Layer:** the job level of the protocol. This page uses the numbers of the OSI model, a 7-layer teaching model. Layer 1 is the signal on the wire. Layer 7 is the program you use.
- **Transport and port:** how the protocol is carried, and which door it uses.
- **Secure alternative:** what to use, or what to add, so that outsiders cannot read or change the traffic.

### What can appear in "Transport and port"

- **TCP:** Transmission Control Protocol. It sets up a connection and resends lost data. Reliable, like a tracked parcel.
- **UDP:** User Datagram Protocol. It just sends. No connection and no resend. Fast, like a postcard.
- **Port:** a number from 0 to 65535 that says which program on a device should get the data. If the IP address is the building, the port is the flat number.
- **IP protocol number:** some protocols do not use TCP or UDP. They sit directly on IP. The IP header has a field that names them. OSPF, for example, is IP protocol 89. These protocols have no ports.
- **EtherType:** a number in the Ethernet frame that says what is inside. ARP is EtherType 0x0806. These protocols do not use IP at all.

The ports listed are the default or registered ones. An admin can run most services on another port. A port number is a habit, not a law.

### What can appear in "Secure alternative"

| Text in the table | Meaning |
|---|---|
| A protocol name | Replace the old protocol with this one |
| Built in | The protocol is already encrypted or protected by design |
| Protect: ... | No secure version is in common use, so add this control around it |
| None common | No widely used secure version exists |

Two more words matter here:

- **Encryption:** scrambling data so only the right receiver can read it.
- **Authentication:** proving who you are, or proving that a message is really from the sender.

Note that the analogy stops here: a real phone book lists one number per person. A protocol can use several ports, and two protocols can share one port number on different transports.

### A simplification to keep in mind

The OSI model is a teaching tool. Real protocols do not always fit one layer. This page puts each protocol where most books and interviewers put it. Section [Tricky Layer Questions](#tricky-layer-questions) covers the arguable cases.

## The Whole Stack On One Page

This map shows who carries whom. Read from the bottom up.

```text
 Layer 7   HTTP  DNS  DHCP  SSH  SMTP  FTP  SNMP  SIP  BGP  RIP
 program        |         |          |              |
 ---------------+---------+----------+--------------+-----------
 Layer 6/5      TLS (for TCP)              DTLS (for UDP)
 ---------------+---------------------------+-------------------
 Layer 4        TCP                        UDP    QUIC runs on UDP
                |                           |
 ---------------+-----------+---------------+-------------------
 Layer 3                 IPv4 / IPv6
 addresses      |     |     |     |      |      |
                ICMP  OSPF  EIGRP GRE    ESP    PIM
                (these sit directly on IP, no ports)
 ---------------------------+-----------------------------------
 Layer 2     Ethernet / Wi-Fi / PPP
 local link     |      |      |      |
                ARP    STP    LLDP   IS-IS
                (these sit directly on the link, no IP)
 ---------------------------+-----------------------------------
 Layer 1     copper, fibre, radio
```

The simple rule: a protocol's port type tells you what is under it.

| What it has | What it sits on | Examples |
|---|---|---|
| A TCP or UDP port | TCP or UDP, then IP | HTTP, DNS, BGP, RIP |
| An IP protocol number | IP directly | ICMP, OSPF, EIGRP, GRE, ESP |
| An EtherType or link address only | The link directly | ARP, LACP, LLDP, STP, IS-IS |

## Layer 1 And Layer 2: Physical And Link

Layer 1 moves signals. Layer 2 moves frames between devices on the same link. A **frame** is one Layer 2 message. Nothing here uses IP or ports, so these protocols cannot cross a router on their own.

More detail: [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md), [Cables and Transmission Media](../01-foundations/05-cables-and-media.md).

### Physical and access technologies

Many of these are standards for signals, not protocols in the strict sense.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| 802.3 PHY | IEEE 802.3 physical layer (1000BASE-T and others) | 1 | None | Defines Ethernet signals on copper and fibre | Protect: physical access control |
| PoE | Power over Ethernet (IEEE 802.3af, at, bt) | 1 | None | Sends power and data on one cable | None common |
| DSL | Digital Subscriber Line | 1 | None | Internet over telephone copper | Protect: encrypt above this layer |
| DOCSIS | Data Over Cable Service Interface Specification | 1-2 | None | Internet over cable TV lines | Built in: link encryption, called BPI+ |
| PON | Passive Optical Network (GPON, XGS-PON) | 1-2 | None | Fibre to the home, shared by many homes | Built in: downstream encryption |
| SONET/SDH | Synchronous Optical Networking / Synchronous Digital Hierarchy | 1 | None | Older carrier transport on fibre | Protect: encrypt above this layer |

### Link protocols

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| Ethernet | Ethernet (IEEE 802.3) | 1-2 | None, carries an EtherType | Standard for wired LANs | MACsec |
| LLC | Logical Link Control (IEEE 802.2) | 2 | None | Upper part of Layer 2, names the protocol inside a frame | None common |
| PPP | Point-to-Point Protocol | 2 | None, runs on serial and other links | Joins two devices on a direct link, with login and IP setup | Protect: use CHAP, not PAP |
| PAP | Password Authentication Protocol | 2 | Inside PPP | PPP login that sends the password in clear text | CHAP or EAP |
| CHAP | Challenge Handshake Authentication Protocol | 2 | Inside PPP | PPP login that never sends the password itself | EAP-TLS (stronger) |
| PPPoE | PPP over Ethernet | 2 | EtherType 0x8863 and 0x8864 | Runs PPP on Ethernet, common for home DSL and fibre | Protect: use CHAP |
| HDLC | High-Level Data Link Control | 2 | None | Simple framing on serial links, default on Cisco serial ports | Protect: PPP with CHAP |
| Frame Relay | Frame Relay | 2 | None | Old shared WAN service (historic) | MPLS VPN or IPsec VPN |
| ATM | Asynchronous Transfer Mode | 2 | None | Old WAN using fixed 53-byte cells (historic) | MPLS VPN or IPsec VPN |

### Switching and LAN control

Interview tip: Cisco-only protocols often have an open standard twin. PAgP pairs with LACP. CDP pairs with LLDP. Know both.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| ARP | Address Resolution Protocol | 2 (some say 2.5) | EtherType 0x0806 | Finds the MAC address for an IPv4 address | Protect: Dynamic ARP Inspection |
| RARP | Reverse ARP | 2 | EtherType 0x8035 | Asked for an IP from a MAC (historic) | Replaced by DHCP |
| 802.1Q | IEEE 802.1Q VLAN tagging | 2 | EtherType 0x8100 (the tag) | Marks which VLAN a frame belongs to on a trunk | Protect: change the native VLAN |
| QinQ | IEEE 802.1ad stacked VLANs | 2 | EtherType 0x88A8 | Puts a provider tag over a customer tag | Protect: MACsec |
| ISL | Inter-Switch Link | 2 | None, Cisco wrapper | Old Cisco trunking (historic) | Replaced by 802.1Q |
| DTP | Dynamic Trunking Protocol | 2 | None, Cisco frames | Cisco switches agree to form a trunk | Protect: turn it off, set ports by hand |
| VTP | VLAN Trunking Protocol | 2 | None, Cisco frames | Cisco switches share the VLAN list | Protect: VTPv3 with a password, or transparent mode |
| STP | Spanning Tree Protocol (IEEE 802.1D) | 2 | None, LLC frames to 01:80:C2:00:00:00 | Blocks ports so that switch loops cannot form | Protect: BPDU Guard, Root Guard |
| RSTP | Rapid Spanning Tree Protocol (IEEE 802.1w) | 2 | Same as STP | Faster version of STP | Protect: BPDU Guard, Root Guard |
| MSTP | Multiple Spanning Tree Protocol (IEEE 802.1s) | 2 | Same as STP | Maps many VLANs to a few spanning trees | Protect: BPDU Guard, Root Guard |
| PVST+ | Per-VLAN Spanning Tree Plus | 2 | Cisco frames to 01:00:0C:CC:CC:CD | One spanning tree per VLAN, Cisco | Protect: BPDU Guard, Root Guard |
| LACP | Link Aggregation Control Protocol (IEEE 802.1AX, formerly 802.3ad) | 2 | EtherType 0x8809 | Bundles links into one logical link, open standard | None common |
| PAgP | Port Aggregation Protocol | 2 | None, Cisco frames | Bundles links, Cisco only | None common |
| CDP | Cisco Discovery Protocol | 2 | None, frames to 01:00:0C:CC:CC:CC | Tells neighbours your model, name and port | Protect: turn off on untrusted ports |
| LLDP | Link Layer Discovery Protocol (IEEE 802.1AB) | 2 | EtherType 0x88CC | Open standard version of CDP | Protect: turn off on untrusted ports |
| 802.1X | IEEE 802.1X port access control | 2 | EAPOL, EtherType 0x888E | Device must log in before the port opens | Built in, it is the security control |
| MACsec | Media Access Control Security (IEEE 802.1AE) | 2 | EtherType 0x88E5 | Encrypts frames hop by hop on a link | Built in |

More detail: [ARP](../03-ethernet-and-switching/05-arp.md), [VLANs](../03-ethernet-and-switching/06-vlans.md), [Trunking and 802.1Q](../03-ethernet-and-switching/07-trunking-8021q.md), [Spanning Tree Protocol](../03-ethernet-and-switching/09-spanning-tree.md), [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md), [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).

### Wireless and mobile

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| Wi-Fi | IEEE 802.11 | 1-2 | None | Wireless LAN | WPA3 |
| WEP | Wired Equivalent Privacy | 2 | Inside 802.11 | First Wi-Fi security, broken | WPA3 |
| WPA | Wi-Fi Protected Access | 2 | Inside 802.11 | Stopgap using TKIP, deprecated | WPA3 |
| WPA2 | Wi-Fi Protected Access 2 | 2 | Inside 802.11 | Wi-Fi encryption with AES-CCMP | WPA3 |
| WPA3 | Wi-Fi Protected Access 3 | 2 | Inside 802.11 | Current Wi-Fi security, uses SAE for password login | Built in |
| EAP | Extensible Authentication Protocol | 2 | Inside 802.1X or PPP | Framework that carries many login methods | Use EAP-TLS or PEAP, not EAP-MD5 |
| CAPWAP | Control and Provisioning of Wireless Access Points | Tunnel | UDP 5246 (control), UDP 5247 (data) | Links access points to a wireless controller | Built in: DTLS on control, optional on data |
| GTP | GPRS Tunnelling Protocol | Tunnel | UDP 2123 (GTP-C), UDP 2152 (GTP-U) | Carries user traffic inside mobile networks | Protect: IPsec on backhaul links |
| Bluetooth | Bluetooth | 1-2 and up | None, own stack | Short-range device links | Built in: pairing and encryption |
| Zigbee | Zigbee (on IEEE 802.15.4) | 1-3 and up | None, own stack | Low-power mesh for sensors | Built in: AES-based security |
| LoRaWAN | Long Range Wide Area Network | 2 | None, own radio stack | Long-range, low-rate IoT links | Built in: AES-based security |

More detail: [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md), [Wi-Fi Security](../08-wireless-and-mobile/02-wifi-security.md), [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md), [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).

## Layer 3: Network

Layer 3 moves packets between networks. A **packet** is one Layer 3 message. Routers work at this layer.

### IP and helpers

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| IPv4 | Internet Protocol version 4 | 3 | EtherType 0x0800 | 32-bit addressing and packet delivery | IPsec |
| IPv6 | Internet Protocol version 6 | 3 | EtherType 0x86DD | 128-bit addressing and packet delivery | IPsec |
| ICMP | Internet Control Message Protocol | 3 | IP protocol 1 | Error and test messages for IPv4, used by ping | Protect: filter by type, rate limit |
| ICMPv6 | ICMP for IPv6 | 3 | IP protocol 58 | Errors, tests and neighbour messages for IPv6 | Protect: filter with care, IPv6 needs it |
| NDP | Neighbor Discovery Protocol | 3 | ICMPv6 types 133-137 | Replaces ARP, finds routers and neighbours in IPv6 | Protect: RA Guard. SEND exists but is rare |
| SLAAC | Stateless Address Autoconfiguration | 3 | Uses NDP | IPv6 host builds its own address | Protect: RA Guard |
| NAT/PAT | Network Address Translation / Port Address Translation | 3-4 | Not a protocol, a router action | Rewrites addresses so many hosts share one public IP | Not a security tool on its own |

More detail: [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md), [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

### Routing protocols

A **routing protocol** lets routers share the paths they know. The job is Layer 3 in every case, but the carrier differs. That is why RIP and BGP are a classic trick question.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| RIP | Routing Information Protocol (v1, v2) | Job 3, stack 7 | UDP 520 | Simple distance-vector routing, limit of 15 hops | Protect: RIPv2 authentication |
| RIPng | RIP next generation | Job 3, stack 7 | UDP 521 | RIP for IPv6 | Protect: IPsec |
| OSPF | Open Shortest Path First | 3 | IP protocol 89 | Link-state routing inside one organisation | Protect: OSPF authentication (SHA for OSPFv2, IPsec or auth trailer for OSPFv3) |
| EIGRP | Enhanced Interior Gateway Routing Protocol | 3 | IP protocol 88 | Fast hybrid routing, made by Cisco | Protect: MD5 or SHA-256 authentication |
| IS-IS | Intermediate System to Intermediate System | 2 carrier, job 3 | None, runs directly on Layer 2 | Link-state routing, common with service providers | Protect: HMAC authentication |
| BGP | Border Gateway Protocol | Job 3, stack 7 | TCP 179 | Routing between organisations, runs the internet | Protect: TCP-AO or MD5 on sessions, RPKI for route origin |
| BFD | Bidirectional Forwarding Detection | 3 helper | UDP 3784 (single hop), UDP 4784 (multihop) | Detects a dead link within milliseconds | Protect: BFD authentication |

More detail: [Dynamic Routing Overview](../05-routing/04-dynamic-routing-overview.md), [RIP](../05-routing/05-rip.md), [OSPF](../05-routing/06-ospf.md), [EIGRP](../05-routing/07-eigrp.md), [IS-IS](../05-routing/08-is-is.md), [BGP](../05-routing/09-bgp.md), [Advanced Routing Techniques](../05-routing/12-advanced-routing-techniques.md).

### First hop redundancy

These protocols let two routers share one gateway address, so a host keeps working when a router fails.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| HSRP | Hot Standby Router Protocol | Job 3, carried by UDP | UDP 1985 | One active router, one standby, Cisco | Protect: MD5 authentication |
| VRRP | Virtual Router Redundancy Protocol | 3 | IP protocol 112 | Open standard version of HSRP | None built in for VRRPv3. Protect: filter on the LAN |
| GLBP | Gateway Load Balancing Protocol | Job 3, carried by UDP | UDP 3222 | Several routers share gateway load, Cisco | Protect: MD5 authentication |

More detail: [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

### Multicast

**Multicast** means one sender reaches a group of receivers, without sending to everyone.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| IGMP | Internet Group Management Protocol | 3 | IP protocol 2 | IPv4 hosts join multicast groups | Protect: IGMP filtering on switches |
| MLD | Multicast Listener Discovery | 3 | Inside ICMPv6 | IPv6 version of IGMP | Protect: MLD filtering on switches |
| PIM | Protocol Independent Multicast | 3 | IP protocol 103 | Routers build multicast delivery trees | Protect: PIM neighbour filters |
| MSDP | Multicast Source Discovery Protocol | 3 helper | TCP 639 | Shares multicast source info between domains | Protect: TCP-MD5 on the session |

More detail: [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md), [Multicast Routing](../05-routing/14-multicast-routing.md).

### MPLS

**MPLS** forwards by short labels, not by full IP lookups. Books call it Layer 2.5 because the label sits between the Layer 2 and Layer 3 headers.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| MPLS | Multiprotocol Label Switching | 2.5 | EtherType 0x8847 (unicast), 0x8848 (multicast) | Fast label forwarding, carrier VPNs | Protect: not encrypted, add IPsec if needed |
| LDP | Label Distribution Protocol | Control for 2.5 | UDP 646 (hello), TCP 646 (session) | Routers agree on labels | Protect: MD5 or TCP-AO |
| RSVP-TE | Resource Reservation Protocol, Traffic Engineering | Control for 2.5 | IP protocol 46 | Builds MPLS paths with reserved bandwidth | Protect: RSVP authentication |
| Segment Routing | Segment Routing (SR-MPLS, SRv6) | 2.5 or 3 | Uses MPLS labels or IPv6 headers | The source picks the path, no LDP needed | Protect: strict edge filtering |

More detail: [MPLS](../05-routing/13-mpls.md).

### Tunnels, VPNs and overlays

A **tunnel** puts one packet inside another, like a letter inside a second envelope. A **VPN** (Virtual Private Network) is a tunnel that is also encrypted. Be careful: a tunnel is not always encrypted. GRE and VXLAN are plain envelopes that anyone on the path can open.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| GRE | Generic Routing Encapsulation | 3 | IP protocol 47 | Simple tunnel for many protocols, no encryption | GRE over IPsec |
| IP-in-IP | IP encapsulated in IP | 3 | IP protocol 4 | Puts an IPv4 packet inside another | IPsec |
| 6in4 | IPv6 in IPv4 | 3 | IP protocol 41 | Carries IPv6 across an IPv4 network | IPsec |
| NHRP | Next Hop Resolution Protocol | 3 helper | Carried inside GRE | Finds the public address of a peer in DMVPN | IPsec around the GRE tunnel |
| AH | Authentication Header (IPsec) | 3 | IP protocol 51 | Proves a packet is genuine, does not encrypt. Fails through NAT | ESP |
| ESP | Encapsulating Security Payload (IPsec) | 3 | IP protocol 50, or inside UDP 4500 | Encrypts and authenticates packets | Built in |
| IKE | Internet Key Exchange (v1, v2) | 7 | UDP 500, UDP 4500 with NAT | Sets up keys for IPsec | Built in. Prefer IKEv2 |
| NAT-T | NAT Traversal | 4 | UDP 4500 | Wraps ESP in UDP so that it passes NAT | Built in |
| L2TP | Layer 2 Tunnelling Protocol | Tunnel | UDP 1701 | Tunnels PPP, no encryption alone | L2TP over IPsec |
| PPTP | Point-to-Point Tunnelling Protocol | Tunnel | TCP 1723 plus GRE | Old Microsoft VPN, broken | IPsec, WireGuard or OpenVPN |
| SSTP | Secure Socket Tunnelling Protocol | Tunnel | TCP 443 | Microsoft VPN inside TLS | Built in |
| OpenVPN | OpenVPN | Tunnel | UDP 1194 by default, TCP possible | Open source VPN using TLS | Built in |
| WireGuard | WireGuard | Tunnel | UDP, 51820 by convention | Small modern VPN | Built in |
| VXLAN | Virtual Extensible LAN | Tunnel for Layer 2 | UDP 4789 | Carries Layer 2 frames over IP, 24-bit network ID | Protect: not encrypted, use IPsec or MACsec on the underlay |
| GENEVE | Generic Network Virtualization Encapsulation | Tunnel for Layer 2 | UDP 6081 | Flexible overlay with extra metadata | Protect: not encrypted, use IPsec on the underlay |
| EVPN | Ethernet VPN | Control plane | BGP, TCP 179 | Uses BGP to share MAC and IP locations | Protect: same as BGP |

More detail: [VPNs](../09-security/07-vpn.md), [IPsec](../09-security/08-ipsec.md), [WAN Technologies](../10-enterprise-and-datacenter/08-wan-technologies.md), [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

## Layer 4: Transport

Layer 4 moves data between programs, not just between machines. It is the layer that owns ports.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| TCP | Transmission Control Protocol | 4 | IP protocol 6 | Reliable, ordered byte stream with a handshake | TLS on top |
| UDP | User Datagram Protocol | 4 | IP protocol 17 | Fast, connectionless messages | DTLS on top |
| SCTP | Stream Control Transmission Protocol | 4 | IP protocol 132 | Message streams and multihoming, common in telecom | DTLS or IPsec |
| DCCP | Datagram Congestion Control Protocol | 4 | IP protocol 33 | Unreliable messages with congestion control, rare | DTLS |
| QUIC | QUIC (a name, not an acronym in the standard) | 4 | UDP, usually 443 | Reliable streams with built-in encryption, base of HTTP/3 | Built in, uses TLS 1.3 |

### Security layers that sit on transport

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| TLS | Transport Layer Security | Between 4 and 7 | On TCP, the port belongs to the protocol inside | Encrypts and verifies a connection | Built in. Use TLS 1.2 or 1.3 |
| SSL | Secure Sockets Layer | Between 4 and 7 | On TCP | Old name and old versions of TLS, all broken | TLS 1.3 |
| DTLS | Datagram Transport Layer Security | Between 4 and 7 | On UDP | TLS adapted for UDP | Built in |

More detail: [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md), [UDP in Depth](../06-transport-layer/07-udp.md), [QUIC](../06-transport-layer/08-quic.md), [TLS and HTTPS](07-tls-and-https.md).

## Layer 7: Application

These are the protocols programs use directly. Books list them at Layer 7. The TCP/IP model merges OSI Layers 5, 6 and 7 into one Application layer, so this is the simple home for all of them.

### Naming, addressing and time

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| DNS | Domain Name System | 7 | UDP 53, TCP 53 | Turns names into IP addresses | DoT, DoH or DoQ for privacy. DNSSEC for trust |
| DNSSEC | DNS Security Extensions | 7 | Same as DNS | Signs DNS data to prove it is genuine, does not hide it | Built in for integrity only |
| DoT | DNS over TLS | 7 | TCP 853 | Encrypts DNS queries | Built in |
| DoH | DNS over HTTPS | 7 | TCP 443 (or UDP 443 with HTTP/3) | Encrypts DNS inside web traffic | Built in |
| DoQ | DNS over QUIC | 7 | UDP 853 | Encrypts DNS over QUIC | Built in |
| mDNS | Multicast DNS | 7 | UDP 5353 | Finds local names with no DNS server (`printer.local`) | Protect: keep to trusted LANs |
| LLMNR | Link-Local Multicast Name Resolution | 7 | UDP 5355 | Windows local name lookup | Protect: turn it off, attackers can answer falsely. Use DNS |
| NBNS | NetBIOS Name Service | 5-7 | UDP 137 | Old Windows name lookup | Protect: turn it off. Use DNS |
| DHCP | Dynamic Host Configuration Protocol | 7 | UDP 67 (server), UDP 68 (client) | Gives hosts an IP, mask, gateway and DNS server | Protect: DHCP snooping on switches |
| DHCPv6 | DHCP for IPv6 | 7 | UDP 547 (server), UDP 546 (client) | Gives IPv6 settings | Protect: DHCPv6 Guard on switches |
| BOOTP | Bootstrap Protocol | 7 | UDP 67, UDP 68 | Older parent of DHCP (historic) | Replaced by DHCP |
| NTP | Network Time Protocol | 7 | UDP 123 | Keeps clocks in sync | NTS |
| NTS | Network Time Security | 7 | TCP 4460 for key setup, then UDP 123 | Adds cryptographic protection to NTP | Built in |
| PTP | Precision Time Protocol (IEEE 1588) | 7 or 2 | UDP 319, UDP 320, or EtherType 0x88F7 | Very exact time for trading and telecom | Protect: isolated network |
| WHOIS | WHOIS | 7 | TCP 43 | Looks up who holds a domain or IP block | RDAP over HTTPS |

More detail: [DNS Basics](01-dns-basics.md), [DNS Record Types](02-dns-records.md), [DNS Advanced](03-dns-advanced.md), [DHCP](04-dhcp.md).

### Web and APIs

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| HTTP | Hypertext Transfer Protocol | 7 | TCP 80 (8080 is common for tests) | Fetches web pages and API data | HTTPS |
| HTTPS | HTTP Secure (HTTP over TLS) | 7 | TCP 443 | Encrypted web traffic | Built in |
| HTTP/2 | HTTP version 2 | 7 | TCP 443 | Many requests on one connection. Browsers use it only with TLS | Built in when on TLS |
| HTTP/3 | HTTP version 3 | 7 | UDP 443, on QUIC | HTTP that avoids TCP head-of-line blocking | Built in |
| WebSocket | WebSocket | 7 | TCP 80 (`ws://`), TCP 443 (`wss://`) | Two-way live channel, started from HTTP | `wss://` |
| gRPC | gRPC Remote Procedure Calls | 7 | HTTP/2, no fixed port (50051 is common in samples) | Fast service-to-service calls | TLS or mutual TLS |
| SOCKS5 | SOCKS version 5 | 5 | TCP 1080 by convention | General proxy for any TCP or UDP traffic | Protect: wrap in SSH or TLS |

More detail: [HTTP Basics](05-http-basics.md), [HTTP/1.1, HTTP/2 and HTTP/3](06-http-versions.md), [Real-Time and Modern Protocols](14-realtime-and-modern-protocols.md), [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

### File transfer

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| FTP | File Transfer Protocol | 7 | TCP 21 (control), TCP 20 (active data) or a high port (passive data) | Moves files, password in clear text | SFTP or FTPS |
| FTPS | FTP over TLS | 7 | TCP 21 with explicit TLS, TCP 990 with implicit TLS | Classic FTP wrapped in TLS | Built in |
| SFTP | SSH File Transfer Protocol | 7 | TCP 22 | File transfer inside SSH. Not related to FTP | Built in |
| SCP | Secure Copy Protocol | 7 | TCP 22 | Simple copy over SSH, old protocol | SFTP. Newer OpenSSH `scp` already uses it |
| TFTP | Trivial File Transfer Protocol | 7 | UDP 69 | Tiny, no login. Used for boot images and device configs | Protect: isolated network. SCP or SFTP where the device allows |
| rsync | rsync | 7 | TCP 873 (daemon mode) | Copies only the changed parts of files | rsync over SSH, TCP 22 |

More detail: [File Transfer Protocols](08-file-transfer-protocols.md).

### Remote access

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| SSH | Secure Shell | 7 | TCP 22 | Encrypted command line and tunnels | Built in |
| Telnet | Telecommunication Network | 7 | TCP 23 | Clear text command line | SSH |
| rlogin, rsh, rexec | Remote login, remote shell, remote execution | 7 | TCP 513, TCP 514, TCP 512 | Old Unix remote tools, clear text | SSH |
| RDP | Remote Desktop Protocol | 7 | TCP 3389, UDP 3389 | Windows remote desktop | Built in with TLS. Do not expose it directly: use a VPN or gateway |
| VNC | Virtual Network Computing (uses the RFB protocol) | 7 | TCP 5900 and up | Remote desktop, works on many systems | Protect: tunnel in SSH or use a VPN |

More detail: [Remote Access Protocols](09-remote-access-protocols.md).

### Email

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| SMTP | Simple Mail Transfer Protocol | 7 | TCP 25 | Moves mail between mail servers | STARTTLS, with MTA-STS or DANE to enforce it |
| Submission | Message Submission (SMTP) | 7 | TCP 587 with STARTTLS, TCP 465 with implicit TLS | Your mail client sends mail to your server | Built in. TCP 465 is the current preference |
| POP3 | Post Office Protocol version 3 | 7 | TCP 110 | Downloads mail to one device | POP3S, TCP 995 |
| IMAP | Internet Message Access Protocol | 7 | TCP 143 | Keeps mail on the server, syncs many devices | IMAPS, TCP 993 |
| SPF | Sender Policy Framework | 7 | A DNS TXT record | Lists servers allowed to send for a domain | Built in, anti-spoofing |
| DKIM | DomainKeys Identified Mail | 7 | Key in DNS, signature in the mail header | Signs mail to prove it was not changed | Built in, anti-spoofing |
| DMARC | Domain-based Message Authentication, Reporting and Conformance | 7 | A DNS TXT record | Says what to do when SPF and DKIM fail | Built in, anti-spoofing |

More detail: [Email Protocols](10-email-protocols.md).

### Network management and monitoring

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| SNMP v1, v2c | Simple Network Management Protocol | 7 | UDP 161 (queries), UDP 162 (traps) | Reads device stats. Community string is in clear text | SNMPv3 |
| SNMPv3 | SNMP version 3 | 7 | UDP 161, UDP 162 | SNMP with users, authentication and encryption | Built in with authPriv mode |
| Syslog | System Logging Protocol | 7 | UDP 514 (TCP 514 on some vendors) | Sends log messages to a server | Syslog over TLS, TCP 6514 |
| NETCONF | Network Configuration Protocol | 7 | TCP 830 over SSH | Configures devices with XML | Built in |
| RESTCONF | REST-based Configuration Protocol | 7 | TCP 443 | Configures devices with HTTPS calls | Built in |
| gNMI | gRPC Network Management Interface | 7 | gRPC on TCP, port varies by vendor, 9339 is common | Config and streaming telemetry | Built in with TLS |
| NetFlow | NetFlow | 7 | UDP, no fixed port (2055 and 9995 are common) | Exports records of who talked to whom | Protect: management network |
| IPFIX | IP Flow Information Export | 7 | UDP, TCP or SCTP 4739 | Standard version of NetFlow | TLS or DTLS, port 4740 |
| sFlow | Sampled Flow | 7 | UDP 6343 | Samples packets for traffic stats | Protect: management network |
| OpenFlow | OpenFlow | 7 | TCP 6653 (older gear used 6633) | SDN controller programs switches | TLS |

More detail: [Network Management Protocols](11-network-management-protocols.md), [Network Monitoring and Observability](../12-troubleshooting-and-tools/11-network-monitoring.md), [Network Automation](../10-enterprise-and-datacenter/12-network-automation.md), [Software-Defined Networking](../10-enterprise-and-datacenter/09-software-defined-networking.md).

### Directory and authentication

The usual memory hook is **AAA**: Authentication (who are you), Authorization (what may you do), Accounting (what did you do).

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| LDAP | Lightweight Directory Access Protocol | 7 | TCP 389 (UDP 389 for small lookups) | Searches user and group directories | LDAP with StartTLS on 389, or LDAPS |
| LDAPS | LDAP over TLS | 7 | TCP 636 | Encrypted LDAP | Built in |
| Global Catalog | Active Directory Global Catalog | 7 | TCP 3268, TLS on TCP 3269 | Forest-wide LDAP search in Active Directory | TCP 3269 |
| Kerberos | Kerberos | 7 | TCP 88, UDP 88 (password change on 464) | Ticket-based login, default in Active Directory | Built in |
| NTLM | NT LAN Manager | 7 | Inside SMB, HTTP and others | Old Windows challenge-response login | Kerberos |
| RADIUS | Remote Authentication Dial-In User Service | 7 | UDP 1812 (auth), UDP 1813 (accounting), old 1645, 1646 | AAA for network access, Wi-Fi and VPN | RadSec (RADIUS over TLS), TCP 2083 |
| TACACS+ | Terminal Access Controller Access-Control System Plus | 7 | TCP 49 | AAA for admin login to network devices | Body is obfuscated. Protect: management network |
| Diameter | Diameter (a play on the word radius) | 7 | TCP or SCTP 3868 | Successor to RADIUS, mainly in mobile networks | TLS on TCP 5868 |
| SAML | Security Assertion Markup Language | 7 | HTTPS, TCP 443 | Single sign-on with XML, common in companies | Built in on HTTPS |
| OAuth 2.0 | Open Authorization 2.0 | 7 | HTTPS, TCP 443 | Lets an app act for you without your password | Built in on HTTPS |
| OIDC | OpenID Connect | 7 | HTTPS, TCP 443 | Login layer on top of OAuth 2.0 | Built in on HTTPS |
| MS-RPC | Microsoft Remote Procedure Call | 7 | TCP 135, then a dynamic high port | Windows services call each other | Protect: firewall, packet privacy |

More detail: [Directory and Authentication Protocols](12-directory-and-authentication-protocols.md).

### File sharing and storage

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| SMB | Server Message Block | 7 | TCP 445 | Windows file and printer sharing | SMB 3 with encryption. Turn off SMBv1 |
| SMB over QUIC | SMB over QUIC | 7 | UDP 443 | SMB across the internet with no VPN | Built in |
| NetBIOS over TCP/IP | Network Basic Input/Output System | 5 | UDP 137, UDP 138, TCP 139 | Old transport for SMB | SMB directly on TCP 445 |
| NFS | Network File System | 7 | TCP 2049 (old versions also UDP) | Unix and Linux file sharing | NFSv4 with Kerberos (`krb5p`) |
| rpcbind | Port mapper for ONC RPC | 7 | TCP 111, UDP 111 | Tells clients where NFS helpers listen. NFSv4 does not need it | Protect: firewall |
| AFP | Apple Filing Protocol | 7 | TCP 548 | Old Apple file sharing, deprecated | SMB |
| iSCSI | Internet Small Computer Systems Interface | 7 | TCP 3260 | Disk blocks over IP | Protect: CHAP, IPsec, separate network |
| Fibre Channel | Fibre Channel | Own stack | None, own network | Lossless storage network | Protect: zoning |
| FCoE | Fibre Channel over Ethernet | 2 | EtherType 0x8906 | Fibre Channel on lossless Ethernet | Protect: zoning |
| FCIP | Fibre Channel over IP | Tunnel | TCP 3225 | Links storage networks across IP | IPsec |
| NVMe/TCP | Non-Volatile Memory Express over TCP | 7 | TCP 4420 (data), TCP 8009 (discovery) | Fast SSD access over IP | TLS, optional in the standard |

More detail: [File Sharing and Storage Protocols](13-file-sharing-and-storage-protocols.md).

### Real-time media and messaging

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| RTP | Real-time Transport Protocol | 7 | UDP, dynamic ports (Cisco voice often uses 16384-32767) | Carries live voice and video | SRTP |
| RTCP | RTP Control Protocol | 7 | UDP, usually RTP port + 1, or same port with rtcp-mux | Quality reports for RTP | SRTCP |
| SRTP | Secure RTP | 7 | UDP, dynamic | Encrypted voice and video | Built in |
| SIP | Session Initiation Protocol | 7 | UDP or TCP 5060 | Sets up and ends calls | SIP over TLS, TCP 5061 |
| SDP | Session Description Protocol | 7 | Text inside SIP or WebRTC signalling | Describes codecs and media ports | Protected by its carrier |
| H.323 | ITU-T H.323 | 7 | TCP 1720 for call setup | Older voice and video call system | SIP over TLS and SRTP |
| RTSP | Real Time Streaming Protocol | 7 | TCP 554 | Controls media streams, common on IP cameras | RTSP over TLS |
| RTMP | Real-Time Messaging Protocol | 7 | TCP 1935 | Sends live video to streaming platforms | RTMPS, usually TCP 443 |
| HLS, DASH | HTTP Live Streaming, Dynamic Adaptive Streaming over HTTP | 7 | HTTPS, TCP 443 | Video in small chunks over HTTP | Built in on HTTPS |
| WebRTC | Web Real-Time Communication | 7 | Mostly UDP, dynamic ports | Browser voice, video and data | Built in, always encrypted |
| STUN | Session Traversal Utilities for NAT | 7 | UDP or TCP 3478 | Tells a host its public address | TLS on TCP 5349 |
| TURN | Traversal Using Relays around NAT | 7 | UDP or TCP 3478 | Relays media when direct paths fail | TLS on TCP 5349 |
| ICE | Interactive Connectivity Establishment | 7 | Uses STUN and TURN | Tries paths and picks one that works | Runs in the WebRTC security model |
| XMPP | Extensible Messaging and Presence Protocol | 7 | TCP 5222 (client), TCP 5269 (server) | Open chat and presence | STARTTLS on the same ports |
| IRC | Internet Relay Chat | 7 | TCP 6667 | Old group chat | IRC over TLS, TCP 6697 |
| MQTT | Message Queuing Telemetry Transport | 7 | TCP 1883 | Light publish and subscribe for IoT | MQTT over TLS, TCP 8883 |
| CoAP | Constrained Application Protocol | 7 | UDP 5683 | Web-style requests for tiny devices | CoAP over DTLS, UDP 5684 |
| AMQP | Advanced Message Queuing Protocol | 7 | TCP 5672 | Message queues between services | AMQP over TLS, TCP 5671 |

More detail: [Real-Time and Modern Protocols](14-realtime-and-modern-protocols.md).

### Device discovery and printing

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| SSDP | Simple Service Discovery Protocol | 7 | UDP 1900 | Part of UPnP, finds devices on the LAN | Protect: keep on the LAN, block from the internet |
| IPP | Internet Printing Protocol | 7 | TCP 631 | Modern network printing | IPP over TLS, same port |
| LPD | Line Printer Daemon | 7 | TCP 515 | Old network printing | IPP over TLS |

### Cloud and container service ports

These are services, not formal internet protocols. Interviewers for DevOps roles still ask about them.

| Protocol | Full name | Layer | Transport and port | Purpose | Secure alternative |
|---|---|---|---|---|---|
| Kubernetes API | Kubernetes API server | 7 | TCP 6443 | Main control entry for a cluster | Built in with TLS |
| Kubelet API | Kubelet API | 7 | TCP 10250 | Control plane talks to each node | Built in with TLS |
| etcd | etcd key-value store | 7 | TCP 2379 (client), TCP 2380 (peer) | Stores cluster state | Mutual TLS |
| NodePort | Kubernetes NodePort range | 4 | TCP or UDP 30000-32767 by default | Opens a service on every node | Protect: firewall, use a load balancer |
| Docker API | Docker daemon API | 7 | TCP 2375 (no TLS), TCP 2376 (TLS) | Remote control of Docker | TCP 2376 with client certs |
| Swarm | Docker Swarm | 7 | TCP 2377, TCP and UDP 7946, UDP 4789 | Cluster control, node gossip, VXLAN overlay | Overlay encryption option, IPsec |

More detail: [Docker Networking](../11-cloud-and-containers/07-docker-networking.md), [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md), [Kubernetes Ingress and Network Policies](../11-cloud-and-containers/09-kubernetes-ingress-and-policies.md).

## Numbers Worth Knowing

The port list has its own chapter: [Common Ports Reference](../06-transport-layer/10-common-ports-reference.md). The tables below cover numbers that are not ports.

### IP protocol numbers

| Number | Protocol |
|---|---|
| 1 | ICMP |
| 2 | IGMP |
| 4 | IP-in-IP |
| 6 | TCP |
| 17 | UDP |
| 33 | DCCP |
| 41 | IPv6 in IPv4 |
| 46 | RSVP |
| 47 | GRE |
| 50 | ESP |
| 51 | AH |
| 58 | ICMPv6 |
| 88 | EIGRP |
| 89 | OSPF |
| 103 | PIM |
| 112 | VRRP |
| 132 | SCTP |

### EtherType values

| Value | Protocol |
|---|---|
| 0x0800 | IPv4 |
| 0x0806 | ARP |
| 0x8035 | RARP |
| 0x8100 | 802.1Q VLAN tag |
| 0x86DD | IPv6 |
| 0x8809 | Slow protocols, including LACP |
| 0x8847 | MPLS unicast |
| 0x8848 | MPLS multicast |
| 0x8863 | PPPoE discovery |
| 0x8864 | PPPoE session |
| 0x888E | EAPOL (802.1X) |
| 0x88A8 | QinQ outer tag |
| 0x88CC | LLDP |
| 0x88E5 | MACsec |
| 0x88F7 | PTP |
| 0x8906 | FCoE |

### Multicast addresses used by protocols

The IPv4 range 224.0.0.0/24 stays on the local link. Routers do not forward it. The IPv6 FF02:: scope is the same idea.

| IPv4 | IPv6 | Used by |
|---|---|---|
| 224.0.0.1 | FF02::1 | All hosts on the link (all nodes in IPv6) |
| 224.0.0.2 | FF02::2 | All routers on the link |
| 224.0.0.5 | FF02::5 | All OSPF routers |
| 224.0.0.6 | FF02::6 | OSPF Designated Routers |
| 224.0.0.9 | FF02::9 | RIPv2, RIPng |
| 224.0.0.10 | FF02::A | EIGRP |
| 224.0.0.13 | FF02::D | PIM |
| 224.0.0.18 | FF02::12 | VRRP |
| 224.0.0.22 | FF02::16 | IGMPv3 reports, MLDv2 reports |
| 224.0.0.102 | Not used here | HSRPv2 and GLBP |
| 224.0.0.251 | FF02::FB | mDNS |
| 224.0.0.252 | FF02::1:3 | LLMNR |
| 239.255.255.250 | FF02::C | SSDP |
| Not used | FF02::1:2 | DHCPv6 servers and relays |

HSRPv1 uses 224.0.0.2. RIPv1 used broadcast, not multicast.

## Comparison Lists

### Insecure to secure: the swap table

This is the fastest revision list on this page. Interviewers love it.

| Insecure | Port | Secure replacement | Port |
|---|---|---|---|
| HTTP | TCP 80 | HTTPS | TCP 443 |
| Telnet | TCP 23 | SSH | TCP 22 |
| FTP | TCP 21 | SFTP | TCP 22 |
| FTP | TCP 21 | FTPS (implicit) | TCP 990 |
| POP3 | TCP 110 | POP3S | TCP 995 |
| IMAP | TCP 143 | IMAPS | TCP 993 |
| SMTP submission | TCP 25 | Submission with TLS | TCP 465 or 587 |
| LDAP | TCP 389 | LDAPS | TCP 636 |
| DNS | UDP 53 | DoT | TCP 853 |
| DNS | UDP 53 | DoH | TCP 443 |
| SNMP v1, v2c | UDP 161 | SNMPv3 | UDP 161 (same port) |
| Syslog | UDP 514 | Syslog over TLS | TCP 6514 |
| SIP | TCP 5060 | SIP over TLS | TCP 5061 |
| RTP | UDP dynamic | SRTP | UDP dynamic |
| MQTT | TCP 1883 | MQTT over TLS | TCP 8883 |
| NTP | UDP 123 | NTS | TCP 4460, then UDP 123 |
| RADIUS | UDP 1812 | RadSec | TCP 2083 |
| WEP, WPA | Radio | WPA3 | Radio |
| PPTP | TCP 1723 | WireGuard | UDP 51820 (convention) |

Two patterns help:

- Most secure twins have a **new port**. SNMPv3 does not. It keeps the same port and changes the message format.
- Some protocols upgrade on the **same port** using STARTTLS: SMTP, IMAP, LDAP and FTPS in explicit mode. **STARTTLS** is a command that says "switch to encryption now". So a port number alone does not prove that traffic is safe, or unsafe.

### Retired: do not deploy

| Protocol | Problem | Use instead |
|---|---|---|
| SSL 2.0, SSL 3.0 | Broken | TLS 1.3 |
| TLS 1.0, TLS 1.1 | Deprecated by RFC 8996 | TLS 1.2 or 1.3 |
| WEP | Key can be recovered quickly | WPA3 |
| WPA with TKIP | Deprecated | WPA3, or WPA2 with AES |
| PPTP | Weak authentication | IPsec, WireGuard, OpenVPN |
| Telnet, rlogin, rsh | Clear text | SSH |
| SMBv1 | Weak, abused by the WannaCry worm in 2017 | SMB 3 |
| SNMP v1, v2c | Community string is in clear text | SNMPv3 |
| NTLMv1 | Weak hash | Kerberos |
| RIPv1 | Classful, no authentication | OSPF, or RIPv2 for small labs |
| IKEv1 | Older and more complex | IKEv2 |

"Retired" here means not for new work. You may still meet them in old networks.

### Protocols that use more than one port

| Protocol | Ports | Why |
|---|---|---|
| FTP | TCP 21, TCP 20 | Control and active data are separate |
| DHCP | UDP 67, UDP 68 | Server port and client port |
| DHCPv6 | UDP 547, UDP 546 | Server port and client port |
| SNMP | UDP 161, UDP 162 | Queries and traps |
| RADIUS | UDP 1812, UDP 1813 | Authentication and accounting |
| IKE | UDP 500, UDP 4500 | Normal, then moved when NAT is found |
| CAPWAP | UDP 5246, UDP 5247 | Control and data |
| GTP | UDP 2123, UDP 2152 | Control and user data |
| NetBIOS | UDP 137, UDP 138, TCP 139 | Name, datagram and session |
| etcd | TCP 2379, TCP 2380 | Clients and cluster peers |

### Protocols that use both TCP and UDP

| Protocol | Port | UDP use | TCP use |
|---|---|---|---|
| DNS | 53 | Normal small queries | Zone transfers, large answers, retries |
| Kerberos | 88 | Small tickets | Large tickets (the usual case in Active Directory) |
| LDAP | 389 | Small lookups (CLDAP) | Normal LDAP |
| RDP | 3389 | Faster screen updates | Session and fallback |
| SIP | 5060 | Common signalling | Larger messages |
| HTTPS | 443 | HTTP/3 on QUIC | HTTP/1.1 and HTTP/2 |
| LDP | 646 | Hello messages | Label sessions |

### Same number, different protocol

A port number has meaning only with its transport.

| Port | TCP | UDP |
|---|---|---|
| 514 | rsh (historic) | Syslog |
| 520 | EFS (little used today) | RIP |
| 443 | HTTPS with HTTP/1.1 or HTTP/2 | HTTP/3, SMB over QUIC, DoH with HTTP/3 |

### Which protocol for which job

| I need to... | Use |
|---|---|
| Get an IP address automatically | DHCP, or SLAAC in IPv6 |
| Turn a name into an IP | DNS |
| Turn an IP into a MAC | ARP in IPv4, NDP in IPv6 |
| Test if a host is reachable | ICMP (ping) |
| Log in to a router | SSH |
| Move a file securely | SFTP |
| Back up a config to a simple server | TFTP or SCP |
| Sync clocks | NTP |
| Collect device stats | SNMPv3 or streaming telemetry |
| Collect logs | Syslog |
| See who talks to whom | NetFlow or IPFIX |
| Route inside a company | OSPF, EIGRP or IS-IS |
| Route between companies | BGP |
| Stop Layer 2 loops | STP family |
| Bundle links | LACP |
| Log admins in to network gear | TACACS+ |
| Log users on to Wi-Fi | RADIUS with 802.1X |
| Link two sites securely | IPsec or WireGuard |
| Stretch Layer 2 over Layer 3 | VXLAN |
| Share gateway failover | HSRP or VRRP |
| Make a voice call | SIP and RTP |

## One Worked Example: A Page Load Touches Many Rows

A laptop joins a LAN and opens `https://www.example.com`. Here is the path through the tables. The addresses are from the same small example.

- Laptop: 192.168.1.50/24
- Gateway and local DNS resolver: 192.168.1.1
- Web server: 203.0.113.10

| Step | Protocol | Layer | Carried on | What happens |
|---|---|---|---|---|
| 1 | DHCP | 7 | UDP 68 to 67 | Laptop gets 192.168.1.50/24, gateway and DNS 192.168.1.1 |
| 2 | ARP | 2 | EtherType 0x0806 | Laptop learns the gateway's MAC address |
| 3 | DNS | 7 | UDP, to port 53 | `www.example.com` resolves to 203.0.113.10 |
| 4 | TCP | 4 | IP protocol 6 | Handshake from 192.168.1.50:51514 to 203.0.113.10:443 |
| 5 | TLS 1.3 | Between 4 and 7 | On TCP | Server proves its identity, keys are agreed |
| 6 | HTTP/2 | 7 | Inside TLS | Browser sends `GET /`, server returns the page |

```text
 Laptop 192.168.1.50                         Server 203.0.113.10
    |                                                 |
    |--- DHCP (UDP 67/68), to the LAN                 |
    |--- ARP (no IP), to the LAN                      |
    |--- DNS query (UDP 53), to 192.168.1.1           |
    |                                                 |
    |=========== TCP handshake, port 443 ============>|
    |<========== TLS 1.3 handshake ==================>|
    |=========== HTTP/2 GET / =======================>|
    |<========== 200 OK ==============================|
```

If both sides use HTTP/3, steps 4 to 6 change. QUIC on UDP 443 does the connection and TLS 1.3 work in one go.

Full walk-through: [What Happens When You Open a Website](../02-network-models/05-what-happens-when-you-open-a-website.md).

## Check Any Port Yourself

Do not trust memory in a lab. Check it.

Look up a name in the local services file.

Linux:

```bash
grep -w 53 /etc/services
```

```text
domain    53/tcp    # Domain Name Server
domain    53/udp
```

Windows PowerShell:

```powershell
$file = "C:\Windows\System32\drivers\etc\services"
Select-String -Path $file -Pattern "\b53/"
```

See which ports a machine is listening on.

Linux:

```bash
ss -tuln
```

```text
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:Port
udp   UNCONN 0      0            0.0.0.0:68        0.0.0.0:*
tcp   LISTEN 0      128          0.0.0.0:22        0.0.0.0:*
tcp   LISTEN 0      511          0.0.0.0:443       0.0.0.0:*
```

Windows PowerShell:

```powershell
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

```text
LocalAddress LocalPort OwningProcess
------------ --------- -------------
0.0.0.0            135          1044
0.0.0.0            445             4
0.0.0.0           3389          1180
```

Test whether one remote port answers.

Linux:

```bash
nc -vz 192.168.1.10 443
```

Windows PowerShell:

```powershell
Test-NetConnection -ComputerName 192.168.1.10 -Port 443
```

More detail: [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

## Memory Tricks

| Group | Trick |
|---|---|
| 20, 21, 22, 23, 25 | FTP data, FTP control, SSH, Telnet, SMTP. Learn them as one run of numbers |
| 67, 68 | DHCP. Server is the lower number |
| 80, 443 | HTTP, HTTPS |
| 110, 143 | POP3, IMAP. Secure twins are 995, 993 |
| 161, 162 | SNMP queries, then traps |
| 179 | BGP. It is on TCP, so it needs a working IP path first |
| 389, 636 | LDAP, LDAPS |
| 445 | SMB |
| 3389 | RDP |
| 88 and 89 | EIGRP and OSPF as IP protocol numbers. Do not confuse them with Kerberos port 88 |
| 50 and 51 | ESP then AH. ESP encrypts, AH only signs |

## Tricky Layer Questions

A strict answer and a job-based answer can differ. Say both. That shows you understand the idea.

| Protocol | Job-based answer | Stack-based answer |
|---|---|---|
| ARP | Helps Layer 3 work | Sits on Ethernet with no IP header, so Layer 2 |
| ICMP | Network control | Sits on IP, in the IP protocol field. Treated as Layer 3 |
| OSPF | Routing, Layer 3 | Sits on IP, Layer 3 |
| RIP | Routing, Layer 3 | Uses UDP, so an application by stack |
| BGP | Routing, Layer 3 | Uses TCP, so an application by stack |
| IS-IS | Routing, Layer 3 | Sits directly on Layer 2 |
| TLS | Security for apps | Between Layers 4 and 7. Often called Layer 6 |
| QUIC | Transport, Layer 4 | Runs inside UDP, and programs often build it in |
| DHCP | Gives IP settings, Layer 3 help | Uses UDP, so Layer 7 |
| MPLS | Forwarding | Between Layers 2 and 3, called 2.5 |

Analogy: a nurse who works at a school does a health job, but sits inside a school. Job and workplace can differ. The analogy stops there, because a nurse has one clear employer, while a protocol has no single right label. Books disagree, so give your reason.

## Common Mistakes

- Thinking SFTP is FTP with TLS. SFTP runs inside SSH on TCP 22. FTP with TLS is FTPS.
- Saying DNS is UDP only. TCP 53 is standard and required for zone transfers and large answers.
- Saying ping uses a port. ICMP has no ports. It uses a type and code.
- Saying OSPF uses TCP or UDP. It uses IP protocol 89 directly.
- Saying DHCP uses TCP. It uses UDP, because the client has no IP address yet and must broadcast.
- Thinking HTTPS is always TCP. HTTP/3 uses UDP 443.
- Calling modern encryption "SSL". The name stuck, but real deployments use TLS.
- Thinking DNSSEC encrypts DNS. It signs answers so you can trust them. DoT, DoH and DoQ hide the query.
- Thinking a tunnel is secure. GRE and VXLAN do not encrypt.
- Thinking AH encrypts. Only ESP encrypts.
- Saying RADIUS encrypts the whole message. It hides only the password. TACACS+ obfuscates the whole body, but with an old method, so keep it on a management network.
- Thinking a high port number means a service is safe. Moving SSH to port 2222 hides nothing from a port scanner.
- Thinking a port proves the protocol. Anyone can run anything on port 443. Firewalls that matter inspect the content too.
- Confusing port 88 (Kerberos) and IP protocol 88 (EIGRP). They are different number spaces.
- Thinking NAT is a security feature. It hides inside addresses, but it does not replace a firewall.

## Interview Drill

**Q: What is the difference between a port number and an IP protocol number?**
A: The IP protocol number is in the IP header and says what the packet carries, such as TCP (6) or OSPF (89). A port number is in the TCP or UDP header and says which program gets the data. OSPF has a protocol number but no port.

**Q: Which protocols sit directly on IP, with no TCP or UDP?**
A: The common ones are ICMP (1), IGMP (2), GRE (47), ESP (50), AH (51), EIGRP (88), OSPF (89), PIM (103) and VRRP (112).

**Q: What is the difference between SFTP, FTPS and SCP?**
A: FTPS is classic FTP wrapped in TLS, on TCP 21 or 990. SFTP is a different protocol that runs inside SSH on TCP 22. SCP is a simpler copy method, also over SSH on TCP 22.

**Q: Why does DNS use both UDP and TCP?**
A: UDP is fast for small queries. TCP is used for zone transfers and for answers too large for UDP. DoT also uses TCP, on port 853.

**Q: What layer is BGP, and what does it run on?**
A: BGP does a Layer 3 job, because it shares routes. It runs on TCP port 179, so by the stack it is an application. I would give both answers.

**Q: What layer is ARP?**
A: Most books say Layer 2, because it runs directly on Ethernet with no IP header. It serves Layer 3 by mapping an IPv4 address to a MAC, so some people say Layer 2.5.

**Q: Name the secure replacements for Telnet, HTTP, FTP, SNMPv2c and LDAP.**
A: SSH on TCP 22, HTTPS on TCP 443, SFTP on TCP 22 or FTPS, SNMPv3 on UDP 161, and LDAPS on TCP 636.

**Q: Which uses UDP: RADIUS or TACACS+? Which is better for admin logins?**
A: RADIUS uses UDP 1812 and 1813. TACACS+ uses TCP 49. TACACS+ is the usual choice for admin login, because it separates authentication, authorization and accounting and can authorize each command. RADIUS is the usual choice for user network access.

**Q: What is the open standard version of HSRP, CDP and PAgP?**
A: VRRP for HSRP, LLDP for CDP and LACP for PAgP.

**Q: Does ping use TCP or UDP?**
A: Neither. Ping uses ICMP echo request and echo reply, which is IP protocol 1. Windows `tracert` uses ICMP too, while Linux `traceroute` sends UDP by default.

**Q: Which protocols use port 443?**
A: HTTPS on TCP 443 and HTTP/3 on UDP 443. Many others share it because firewalls usually allow it, such as DoH, SSTP and SMB over QUIC.

**Q: What protocol would you check first if a device has an IP but cannot reach a name?**
A: DNS, on UDP and TCP 53. I would test by IP first, then run `nslookup` or `dig`. If the IP works and the name fails, DNS is the likely problem.

**Q: Why is VXLAN on UDP?**
A: UDP lets the outer packet cross any IP network. Routers can also hash the UDP source port to spread flows over many paths. VXLAN itself does not encrypt.

**Q: Which ports would you open for an email client to send and receive securely?**
A: TCP 465 or 587 to send, TCP 993 for IMAP, or TCP 995 for POP3.

## Quick Recap

- Read a protocol as four facts: full name, layer, carrier and port, secure version.
- Port means TCP or UDP. A protocol number means it sits on IP. An EtherType means it sits on the link.
- Most insecure protocols have a secure twin on a new port. SNMPv3 and STARTTLS are the main exceptions.
- Some protocols use both TCP and UDP, and a port number means nothing without its transport.
- A tunnel is not always encrypted. GRE and VXLAN are not.
- For arguable layers, give the job-based and the stack-based answer.
- Defaults can change. Check real systems with `ss`, `Get-NetTCPConnection`, `nc` or `Test-NetConnection`.
