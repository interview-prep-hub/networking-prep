# UDP in Depth

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](01-ports-and-sockets.md), [TCP vs UDP](02-tcp-vs-udp.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md)

## In One Line

UDP (User Datagram Protocol) adds ports and an optional error check to IP, then sends each message on its own, with no connection, no delivery promise, and no speed control.

## Why It Matters

Some traffic cannot wait for TCP. A voice packet that arrives 500 ms late is useless, so retransmitting it wastes time. A DNS lookup is one question and one answer, so a handshake would double the delay. A server with one million clients cannot always keep one million connections.

UDP solves this by doing almost nothing. The application gets direct access to the network and decides for itself what to do about loss, order, and speed.

This freedom has a price, and that is why interviewers ask about UDP:

- DNS, DHCP, NTP, VoIP, video calls, online games, VPNs, and HTTP/3 all run on it.
- UDP carries the largest reflection attacks, because its source address is easy to fake.
- UDP often breaks in ways that are hard to see: silent drops, NAT timeouts, and one-way audio.
- A strong candidate can explain why QUIC, a reliable protocol, is built on an "unreliable" one.

## Simple Explanation

Think of UDP as sending **postcards**.

You write the address, drop the card in the box, and walk away. You do not call first. You get no receipt. If you send three cards, they may arrive in a different order, or one may never arrive. Each card stands alone.

TCP is more like a **phone call**. You dial, the other side answers, and both of you stay on the line until someone hangs up. If a word is lost, you say "pardon?" and hear it again.

Where the analogy stops being true:

- A postcard is rarely copied by the post office. A UDP datagram can arrive twice.
- The post office sorts cards at its own pace. UDP has no such limit, so a sender can send far too fast and nothing stops it until packets are dropped.
- A postcard has one small size. A UDP datagram can be large, but large ones are split by IP and become much more fragile.

A **datagram** is one self-contained message sent over a network. In UDP, one send by the application normally produces one datagram, and one receive gives back one whole datagram. The message edges are kept. TCP does not keep them, because TCP is a stream of bytes.

## How It Works

### Where UDP sits

UDP is a transport layer protocol. It sits on top of IP and below the application. In the IP header, the protocol number 17 means "the data here is UDP". TCP is 6.

The standard is RFC 768, written in 1980. It is only about three pages long, and it is still the current definition.

```text
+-------------------------------------------------+
| Ethernet | IPv4     | UDP     | Application     |
| header   | header   | header  | data (payload)  |
| 14 bytes | 20 bytes | 8 bytes | e.g. DNS query  |
+-------------------------------------------------+
           | proto=17 |
```

The extra job UDP does over raw IP is small but vital:

1. **Multiplexing**: many applications on one host share one IP address, and ports tell them apart. See [Ports and Sockets](01-ports-and-sockets.md).
2. **Error detection**: the checksum finds damaged data.

That is all. There is no third job.

### The 8 byte header

A header is the control information placed in front of the data. UDP's header is always 8 bytes, with four fields of 16 bits each. A TCP header is at least 20 bytes.

```text
 Bit 0            15 16             31
+-------------------+-------------------+
|    Source Port    |  Destination Port |
|     (16 bits)     |     (16 bits)     |
+-------------------+-------------------+
|      Length       |     Checksum      |
|     (16 bits)     |     (16 bits)     |
+-------------------+-------------------+
|                                       |
|        Data (the payload)             |
|                                       |
+-------------------+-------------------+
```

| Field | Size | Meaning |
|---|---|---|
| Source Port | 2 bytes | Port of the sender. Replies go here. Optional: 0 means "no reply expected". |
| Destination Port | 2 bytes | Port of the receiving application, such as 53 for DNS. |
| Length | 2 bytes | Header plus data, in bytes. Minimum value is 8 (no data at all). |
| Checksum | 2 bytes | Error check over a pseudo-header, the UDP header, and the data. |

What is **missing** tells you as much as what is there:

| Not in the UDP header | Result |
|---|---|
| Sequence number | No ordering, no way to spot loss or duplicates |
| Acknowledgement number | The sender never learns what arrived |
| Window | No flow control, so a fast sender can flood a slow receiver |
| Flags (SYN, FIN, RST) | No connection setup or teardown |

### Size limits

The Length field is 16 bits, so its largest value is 65,535. That number includes the 8 byte header.

| Limit | Calculation | Bytes of data |
|---|---|---|
| UDP theory | 65,535 - 8 | 65,527 |
| Over IPv4 | 65,535 - 20 (IP) - 8 (UDP) | 65,507 |
| Over IPv4 on Ethernet, no fragmentation | 1,500 - 20 - 8 | 1,472 |
| Over IPv6 on Ethernet, no fragmentation | 1,500 - 40 - 8 | 1,452 |
| Safe on any IPv6 path | 1,280 - 40 - 8 | 1,232 |

The 65,507 number is the common interview answer. The 1,472 number is the useful one. Anything larger gets split into IP fragments, and that causes trouble. See [Fragmentation](#fragmentation-the-hidden-weakness).

Fuller truth: over IPv6, the IP payload length does not count the 40 byte IPv6 header, so a normal IPv6 packet can carry 65,527 bytes of UDP data. Larger "jumbograms" (RFC 2675) exist too, but they are rare and need links that support them.

### The checksum

A **checksum** is a small number calculated from the data. The sender writes it in the packet. The receiver calculates it again. If the two do not match, the data was damaged.

**Analogy:** a shop bill has a total at the bottom. If you add the items and get a different total, something was changed. This stops being true in one way: a checksum catches accidents, not attacks. Anyone who changes the data on purpose can also fix the total. A checksum is not security.

The steps:

1. Build a **pseudo-header**: a fake header used only for the calculation and never sent. For IPv4 it holds the source IP, destination IP, a zero byte, the protocol number (17), and the UDP length.
2. Set the checksum field to 0.
3. Put the pseudo-header, UDP header, and data together. If the total length is odd, add one zero byte at the end, only for the calculation.
4. Add everything as 16-bit numbers. If the sum overflows past 16 bits, add the overflow back into the low 16 bits. This is called **one's complement addition**.
5. Flip every bit of the result. This is the checksum.
6. If the result is exactly 0x0000, send 0xFFFF instead, because 0x0000 has a special meaning.

```text
IPv4 pseudo-header (12 bytes, never sent)

 Bit 0                                31
+--------------------------------------+
|          Source IP address           |
+--------------------------------------+
|        Destination IP address        |
+------------+-----------+-------------+
|  Zero (8)  | Proto (8) | UDP Length  |
|            |  = 17     |    (16)     |
+------------+-----------+-------------+
```

**Why include IP addresses in a transport checksum?** So a datagram delivered to the wrong host is caught. If a bit in the destination IP flips and the packet reaches the wrong machine, the checksum fails there and the packet is dropped.

**Rules on IPv4 and IPv6:**

| | IPv4 | IPv6 |
|---|---|---|
| Is the checksum required? | Optional. 0x0000 means "not calculated". | Required, because IPv6 has no header checksum of its own (RFC 8200). |
| Exception | | Tunnel protocols may use a zero checksum under strict rules (RFC 6935, RFC 6936). |

**What happens on failure:** the receiver drops the datagram silently. No error goes back to the sender. Linux counts it in `InCsumErrors`.

**NAT and the checksum:** a NAT device changes the IP and often the port. Both are inside the checksum, so NAT must also update the UDP checksum. If the checksum was 0, it stays 0.

Fuller truth: the checksum is weak. It cannot notice two 16-bit words that swap places, and some multi-bit errors cancel out. Ethernet's CRC is much stronger, but CRC only protects one hop. The UDP checksum runs end to end, so it can catch damage inside a router's memory.

### Connectionless behaviour

**Connectionless** means no setup happens before data flows, and neither end keeps shared state about a conversation.

```text
TCP: 3 packets before the first byte of data

Client                       Server
  | ------- SYN ------------->  |
  | <------ SYN-ACK ----------  |
  | ------- ACK ------------->  |
  | ------- Data ------------>  |

UDP: data is in the first packet

Client                       Server
  | ------- Data ------------>  |
  | <------ Data (if any) ----  |
```

What this means in practice:

- **No delay to start.** Data leaves in the first packet.
- **No state.** A server does not need memory per client for transport, so one socket can serve many clients.
- **Replies are the application's job.** The server reads the source IP and port from each datagram and sends the answer there.
- **No delivery report.** `sendto()` returning success means the datagram was handed to the local network stack. It does not mean the data arrived.
- **No ordering.** Datagrams can arrive in a different order or twice.
- **No speed control.** UDP sends as fast as the application asks. If the network is full, packets are dropped.
- **Message edges are kept.** Three sends of 100 bytes give three reads of 100 bytes.

A TCP socket is found by four values: source IP, source port, destination IP, destination port. An ordinary UDP server socket is found by just two: destination IP and destination port. So all clients land in one socket.

```text
Client A 10.0.0.5:50001  ---+
                            |
Client B 10.0.0.6:50002  ---+--> one socket 10.0.0.1:53
                            |    reads each sender's address
Client C 10.0.0.7:50003  ---+    from each datagram
```

**The `connect()` trap:** you can call `connect()` on a UDP socket. It sends **no packets**. It only records a default peer in the kernel. After that, the socket accepts data only from that peer and can receive reports of ICMP errors. On Linux this shows up as `ECONNREFUSED` on a later send or receive.

**What does the network say when nobody is listening?** If the destination host has no socket on that port, it normally sends back an ICMP (Internet Control Message Protocol) "Port Unreachable" message, type 3 code 3. See [ICMP](../05-routing/10-icmp-ping-and-traceroute.md). A firewall may drop the datagram or the ICMP, and then the sender hears nothing at all.

### Where UDP is used

| Use | Common port | Why UDP fits |
|---|---|---|
| DNS | 53 | One short question and one short answer |
| DHCP | 67 (server), 68 (client) | The client has no IP yet, and broadcast is needed |
| NTP (time sync) | 123 | Small messages, and retransmit delay would spoil the timing |
| TFTP | 69 | Very small code, used in network boot |
| SNMP | 161, 162 (traps) | Light polling of many devices |
| Syslog | 514 | Simple logs, and a lost line is often acceptable |
| RADIUS | 1812, 1813 | Short request and response, with its own retries |
| VoIP and video (RTP) | Dynamic | Late data is worthless, so do not retransmit |
| HTTP/3 (QUIC) | 443 | Reliable protocol built in user space, see [QUIC](08-quic.md) |
| WireGuard VPN | 51820 (usual choice, configurable) | Tunnels work best on an unreliable carrier |
| IPsec through NAT | 4500 | NAT can translate UDP ports but not bare ESP |
| VXLAN | 4789 | Passes through routers, and the source port helps load sharing |
| mDNS | 5353 | Multicast name lookup on a LAN |
| Online games | Vendor choice | Newest position matters, old position does not |

The full list lives in [Common Ports Reference](10-common-ports-reference.md).

A few reasons repeat. UDP fits when:

1. Exchanges are one small request and one small reply.
2. Late data is as bad as lost data.
3. You need broadcast or multicast.
4. You want to build your own transport.
5. You are carrying tunnels.

**Why VPNs prefer UDP:** if you run TCP inside a TCP tunnel, both layers retransmit on loss. Their timers fight, and speed can collapse. This is called TCP meltdown. Over a UDP tunnel, only the inner TCP handles recovery.

**DNS is not UDP only.** A DNS server can set the TC (truncated) flag to tell the client to retry over TCP. Zone transfers also use TCP. See [DNS Basics](../07-application-protocols/01-dns-basics.md).

### Building reliability above UDP

UDP gives you no guarantees, but the application can add exactly those it needs. Think of it as buying a plain bicycle and adding only the parts you want. TCP is the fixed car: full features, no choice. The analogy stops being true at one point: if you build these parts badly, you do not only hurt yourself. You can also hurt everyone sharing the network.

| TCP gives you | How to build it above UDP |
|---|---|
| Loss detection | Add a sequence number to each message |
| Delivery proof | Receiver sends acknowledgements (ACKs) |
| Recovery | Sender keeps a copy, sets a timer, and resends on timeout |
| Ordering | Receiver buffers and sorts by sequence number |
| Duplicate removal | Receiver remembers recent sequence numbers |
| Flow control | Receiver tells the sender how much it can take |
| Congestion control | Sender slows down on loss or rising delay |

A minimal retransmit flow:

```text
Sender                                 Receiver
  | --- seq=1 "hello" ------------------> |
  | <-------------------------- ack=1 --- |
  | --- seq=2 "world" -------X (lost)     |
  |                                       |
  |    timer expires, no ack=2            |
  |                                       |
  | --- seq=2 "world" ------------------> |
  | <-------------------------- ack=2 --- |
```

Details that decide if the design is good:

- **Timeout:** too short creates false resends, and too long adds delay. Measure the round-trip time (RTT) and adapt, as TCP does.
- **Backoff:** double the wait after each failed try, and add a little randomness, called **jitter** here. Without this, thousands of clients retry at the same moment.
- **Idempotency:** an operation is idempotent if doing it twice has the same effect as doing it once. A lost ACK causes a resend, so the receiver sees the request twice. "Add 10 to the balance" is dangerous. Attach a request ID, or use "set the balance to 110".
- **Security:** each datagram can be forged, so a serious protocol adds encryption and sender checks. DTLS (RFC 9147) is TLS adapted for datagrams.
- **Congestion control:** RFC 8085 (UDP Usage Guidelines) says applications are responsible for this. See [TCP Congestion Control](06-tcp-congestion-control.md) for the ideas.

**Real designs choose different levels:**

| Protocol | Reliability choice |
|---|---|
| DNS | Client retries with a timer. The 16-bit ID field matches replies to questions. |
| TFTP | Stop-and-wait: send one block, wait for its ACK, then send the next. |
| RTP (RFC 3550) | Sequence numbers and timestamps show loss and order. No retransmit by default. A jitter buffer smooths arrival. |
| Games | Resend important events (player died). Never resend old positions, since a newer one replaces them. |
| QUIC (RFC 9000) | Full reliability, ordering, congestion control, and encryption, per stream. |

**A number to remember, TFTP:** the original TFTP sends 512 byte blocks and waits for each ACK (RFC 1350). With an RTT of 50 ms, you get 20 blocks per second. That is 10,240 bytes per second, about 10 KB/s, no matter how fast the link is. This is why the blocksize option (RFC 2348) and windowsize option (RFC 7440) were added.

**Forward Error Correction (FEC)** is another option. The sender adds extra repair data, so the receiver can rebuild a lost packet without asking. It costs bandwidth but adds no RTT, so it suits live media.

**A jitter buffer** handles uneven arrival. **Jitter** in media means the variation in packet arrival time. The receiver holds packets for a short time, say 40 ms, then plays them at a steady pace. It trades a little delay for smooth sound.

### Broadcast and multicast over UDP

**Unicast** is one sender to one receiver. **Broadcast** is one sender to all hosts on a local network. **Multicast** is one sender to a group of hosts who asked to receive it. The concepts are covered in [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

**Why only UDP?** TCP needs a handshake and ACKs with exactly one peer. You cannot build one connection with a whole group. UDP has no peer state, so one packet to many receivers is natural.

**Analogy:** broadcast is a loudspeaker in a room: everyone hears, like it or not. Multicast is a radio channel: only people tuned in hear it. The analogy stops being true because the radio signal reaches every home anyway. In a good multicast network, switches and routers forward the stream only toward links with listeners.

#### Broadcast

| Type | Example | Reaches |
|---|---|---|
| Limited broadcast | 255.255.255.255 | The local network only. Routers never forward it. |
| Directed broadcast | 192.168.1.255 on 192.168.1.0/24 | That one subnet. Routers normally refuse to forward it from outside. |

Real example: a DHCP client has no IP yet, so it sends:

```text
SRC 0.0.0.0:68  -->  DST 255.255.255.255:67
Dest MAC ff:ff:ff:ff:ff:ff
```

Every host on the VLAN must accept the frame and inspect it. A router would stop it, so a relay is used to pass it to a central server. See [DHCP](../07-application-protocols/04-dhcp.md).

```text
! Cisco IOS: relay DHCP from this VLAN to a central server
interface Vlan10
 ip helper-address 10.0.50.5
```

Also note:

- An application must set the socket option `SO_BROADCAST` before it can send broadcast.
- Cisco IOS disables forwarding of directed broadcasts by default since 12.0. This blocks the old Smurf attack (ICMP sent to a broadcast address).
- IPv6 has no broadcast. It uses multicast instead.

#### Multicast

The IPv4 multicast range is 224.0.0.0/4. Addresses in 224.0.0.0/24 stay on the local link. Addresses in 239.0.0.0/8 are for private use inside an organisation. IPv6 multicast is ff00::/8.

How a group receive works:

1. The receiver binds a UDP socket to a port, say 5000.
2. It joins a group, say 239.1.1.1, through the socket option `IP_ADD_MEMBERSHIP`.
3. The host sends an IGMP membership report. IGMP (Internet Group Management Protocol) is how IPv4 hosts tell routers what they want. IPv6 uses MLD.
4. A switch with **IGMP snooping** listens to this and forwards the group only to ports that asked.
5. A multicast router uses a protocol such as PIM to bring the stream across subnets. See [Multicast Routing](../05-routing/14-multicast-routing.md).
6. The sender sends one UDP datagram to 239.1.1.1:5000. The network copies it only where paths split.

```text
                 Sender 10.0.0.9
                       |
                 one stream
                       |
                  [ Router ]
                  /        \
          one copy          one copy
              /                \
        [ Switch ]          [ Switch ]
         /      \                |
    Receiver  Receiver       Receiver
```

Without multicast, a 5 Mbit/s video to 100 receivers needs 500 Mbit/s from the sender. With multicast, the sender uses 5 Mbit/s.

At Layer 2, an IPv4 group maps to a MAC address that starts with `01:00:5e`, followed by the low 23 bits of the group. So 239.1.1.1 maps to `01:00:5e:01:01:01`. Only 23 of the 28 group bits fit, so 32 IP groups share each MAC address. For IPv6 the MAC starts with `33:33`.

Common groups:

| Group | Port | Use |
|---|---|---|
| 224.0.0.251 | 5353 | mDNS, name lookup on a LAN |
| 239.255.255.250 | 1900 | SSDP, device discovery |
| 224.0.0.9 | 520 | RIPv2 updates |

Limits you should know:

- The default TTL of a multicast socket is 1, so packets do not leave the local subnet unless the application raises it.
- Multicast rarely works across the public Internet. It is used inside enterprises, ISPs (IPTV), and financial market data feeds.
- Reliability is hard. If 10,000 receivers all send ACKs to one sender, they flood it. This is called **ACK implosion**, so reliable multicast systems use negative ACKs, FEC, or both.

### NAT timeouts

**NAT** (Network Address Translation) lets many private hosts share one public IP. The NAT device keeps a table of mappings: inside address and port to outside address and port. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

TCP tells NAT when a conversation ends, through FIN or RST. UDP never does. The NAT device can only guess, so it uses an **idle timer**. If no packet uses a mapping for that long, the mapping is deleted.

**Analogy:** a security guard lets your friend's reply in, because you told him "someone may call back". After a few minutes without news, he forgets. If the friend then calls, he is turned away, and nobody tells you. The analogy stops being true because the guard could ask you. NAT has no way to ask.

```text
Inside host       NAT table                   Server
    |                  |                         |
    | ---- packet ---> | create mapping          |
    |                  | ---- packet ----------> |
    | <--- reply ----- | <---- reply ----------- |
    |                  |                         |
    |  ...silence longer than the idle timer...  |
    |                  | mapping deleted         |
    |                  |                         |
    |                  | <---- packet ---------- |
    |                  | no mapping: drop        |
    |   (host never    |                         |
    |    learns this)  |                         |
```

**Standard:** RFC 4787 says a NAT's UDP mapping timer must not expire in less than 2 minutes, and recommends 5 minutes or more. It may be shorter for well-known destination ports such as 53. It also says traffic from inside must refresh the timer.

**Real life differs.** Many devices are shorter than the RFC, and home routers vary widely. Do not assume a value. Some known defaults:

| Device | Setting | Default |
|---|---|---|
| Linux conntrack, one-way traffic | `nf_conntrack_udp_timeout` | 30 s |
| Linux conntrack, two-way active flow | `nf_conntrack_udp_timeout_stream` | 120 s (180 s on older kernels) |
| Cisco IOS NAT | `ip nat translation udp-timeout` | 300 s |
| Cisco IOS NAT, DNS | `ip nat translation dns-timeout` | 60 s |
| Cisco ASA | `timeout udp` | 2 minutes |

Check your version, since defaults can change between releases.

**Typical symptoms:**

- A VoIP call goes silent one way after being on hold.
- A VPN stops passing traffic after a quiet period, then recovers when the client sends.
- A phone stops receiving incoming calls, because its registration path has expired.
- A game loses a player who stood still and sent nothing.

**Fixes:**

1. **Keep-alives:** send a small packet from the inside, more often than the shortest timer. WireGuard's `PersistentKeepalive = 25` is the common example. IPsec NAT traversal keep-alives are commonly about 20 seconds.
2. **Raise the timer** on devices you control, but each mapping uses memory and a port.
3. **Design for recovery:** let the protocol handle a new address. QUIC does this with connection IDs, see [QUIC](08-quic.md).

**The trade-off:** long timers waste ports on a busy NAT. A resolver behind NAT may create thousands of short DNS flows per second, which is why DNS often has a shorter timer. Short timers break quiet flows. No single value suits everything.

Stateful firewalls treat UDP the same way. They create a "pseudo connection" from the first packet, allow replies, and delete it on an idle timer. See [Firewalls](../09-security/04-firewalls.md).

### Amplification and reflection

UDP has no handshake, so nothing proves that a source address is real. Anyone who can send **spoofed** packets (packets with a fake source IP) can make a server reply to a third party.

- **Reflection:** the attacker bounces traffic off an innocent server, so the attack seems to come from that server.
- **Amplification:** the reply is much bigger than the request, so the attacker's traffic is multiplied.

**Analogy:** you phone many restaurants and order a large catalogue, giving your enemy's address. Each call costs you little, and the enemy gets a pile of mail. The analogy stops being true because a catalogue arrives in days. Reflected packets arrive in milliseconds, and the victim has no way to refuse them before they fill the link.

```text
Attacker (true IP is hidden)
   |  small request, 64 bytes
   |  SRC = 203.0.113.50 (forged: the victim)
   |  DST = 198.51.100.53, UDP port 53
   v
+-------------------+
| Open DNS resolver |   reflector and amplifier
| 198.51.100.53     |
+-------------------+
   |  large reply, 3000 bytes
   |  SRC = 198.51.100.53
   |  DST = 203.0.113.50
   v
Victim 203.0.113.50
```

**Worked numbers:**

- Bandwidth amplification factor = 3,000 / 64, or about 47.
- An attacker with 100 Mbit/s of upload sends 100 x 47 = 4.7 Gbit/s at the victim.
- With many reflectors, the victim sees traffic from thousands of real servers, so blocking by source address is hard.

Published factors, from US-CERT alert TA14-017A and later reports:

| Protocol | UDP port | Amplification factor (approx.) |
|---|---|---|
| DNS | 53 | 28-54 |
| NTP (`monlist`) | 123 | 556 |
| SSDP | 1900 | 30 |
| CharGEN | 19 | 358 |
| SNMPv2 | 161 | 6 |
| Memcached | 11211 | 10,000-51,000 |

In February 2018, GitHub was hit by a memcached reflection attack that peaked at 1.35 Tbit/s.

**Why the attack needs UDP:** a good reflector needs a stateless protocol that replies to one packet with a larger answer. In TCP, the handshake must finish before real data flows, and the handshake cannot finish with a forged source.

Fuller truth: TCP is not fully immune. SYN-ACK packets can be reflected and retransmitted, and some middleboxes have been abused, but the factors are usually much lower.

**Defences:**

| Who | Action |
|---|---|
| Network operators | Filter spoofed sources at the edge (BCP 38, RFC 2827) |
| Router admins | Enable unicast RPF (uRPF), which checks if the source is reachable through that interface |
| Service owners | Do not expose DNS, NTP, or memcached to the Internet without need |
| DNS operators | Close open resolvers, and use response rate limiting |
| NTP operators | Disable `monlist` (modern ntpd has done so) |
| Memcached operators | Keep UDP off (default off since version 1.5.6) and bind to private addresses |
| Protocol designers | Do not reply bigger than the request until the source is proved |
| Victims | Use a DDoS scrubbing service or upstream filtering |

```text
! Cisco IOS: strict uRPF on an edge interface
ip cef
interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx
```

Good modern designs learned from this. QUIC requires a client's first datagrams to be at least 1,200 bytes, and a server may send at most 3 times the bytes received until it has validated the address (RFC 9000). DTLS uses a cookie exchange for the same reason.

The wider attack list is in [Common Network Attacks](../09-security/03-common-network-attacks.md).

## Example

### Example 1: Build a datagram and calculate its checksum

Host 192.168.1.10 sends the text `hi` from port 50000 to 192.168.1.20 port 9000.

**Step 1: find the header values.**

| Field | Decimal | Hex |
|---|---|---|
| Source Port | 50000 | c350 |
| Destination Port | 9000 | 2328 |
| Length | 8 + 2 = 10 | 000a |
| Checksum | 0 for now | 0000 |
| Data `hi` | | 6869 |

**Step 2: add all 16-bit words, pseudo-header first.**

```text
Pseudo-header
  192.168 (src IP, part 1)     c0a8
    1.10  (src IP, part 2)     010a
  192.168 (dst IP, part 1)     c0a8
    1.20  (dst IP, part 2)     0114
  zero + protocol 17           0011
  UDP length                   000a
UDP header and data
  source port                  c350
  destination port             2328
  length                       000a
  checksum field (zero)        0000
  data "hi"                    6869
                             ------
  Sum                         2d274
```

**Step 3: fold the overflow.** The sum has an extra digit `2` above 16 bits: 0xd274 + 0x2 = 0xd276.

**Step 4: flip all bits.** 0xffff - 0xd276 = 0x2d89. The checksum is **0x2d89**.

**Step 5: how the receiver checks.** It adds everything again, this time including 0x2d89. The result must be 0xffff. Indeed 0xd276 + 0x2d89 = 0xffff.

The full UDP datagram is 10 bytes:

```text
c350 2328 000a 2d89 6869
```

Now try it on real machines. The port number on the sender is chosen by the OS, so it will differ from this example unless you set it.

```bash
# Linux receiver on 192.168.1.20 (OpenBSD netcat syntax)
nc -u -l 9000

# Linux sender on 192.168.1.10, fixed source port, no newline
printf 'hi' | nc -u -w1 -p 50000 192.168.1.20 9000

# Capture on the receiver
sudo tcpdump -i eth0 -nn -vv -X udp port 9000
```

```powershell
# Windows sender (no netcat by default)
$udp = New-Object System.Net.Sockets.UdpClient
$bytes = [Text.Encoding]::ASCII.GetBytes("hi")
$udp.Send($bytes, $bytes.Length, "192.168.1.20", 9000)
$udp.Close()
```

The `Send` call prints `2`, the bytes handed to the stack. This does not prove that they arrived.

Capture output:

```text
12:00:01.000123 IP (tos 0x0, ttl 64, id 4242, offset 0, flags [DF],
    proto UDP (17), length 30)
    192.168.1.10.50000 > 192.168.1.20.9000: [udp sum ok] UDP, length 2
    0x0000:  4500 001e 1092 4000 4011 a6ce c0a8 010a  E.....@.@.......
    0x0010:  c0a8 0114 c350 2328 000a 2d89 6869       .....P#(..-.hi
```

Reading it:

- IP total length is 30: 20 (IP) + 8 (UDP) + 2 (data).
- `4011`: TTL 0x40 (64) and protocol 0x11 (17, UDP).
- `c350 2328 000a 2d89 6869` is the exact datagram we built.

More on reading captures is in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

### Example 2: Datagram to a closed port

Host 192.168.1.20 has nothing on UDP port 9001.

```bash
# Linux, on 192.168.1.10
printf 'hi\n' | nc -u -w1 192.168.1.20 9001
```

```text
12:05:10.201455 IP 192.168.1.10.50001 > 192.168.1.20.9001: UDP, length 3
12:05:10.201702 IP 192.168.1.20 > 192.168.1.10: ICMP 192.168.1.20
    udp port 9001 unreachable, length 39
```

The ICMP length of 39 is 8 (ICMP header) + 20 (original IP header) + 8 (original UDP header) + 3 (original data). The original datagram is quoted back so the sender can match the error to its socket.

To see which UDP ports are open:

```bash
# Linux: UDP sockets, numeric, with process
$ sudo ss -ulnp
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
UNCONN  0       0       0.0.0.0:123         0.0.0.0:*          users:(("chronyd",pid=702,fd=6))
UNCONN  0       0       192.168.1.20:9000   0.0.0.0:*          users:(("nc",pid=4120,fd=3))
```

The state is `UNCONN` (unconnected). You will never see `LISTEN` or `ESTABLISHED` for ordinary UDP, because those are TCP states.

```powershell
# Windows: UDP sockets with process ID
netstat -ano -p udp
```

```text
Active Connections

  Proto  Local Address          Foreign Address        State           PID
  UDP    0.0.0.0:123            *:*                                    1456
  UDP    0.0.0.0:5353           *:*                                    2210
```

### Example 3: Finding a NAT timeout problem

A game client at 192.168.1.10 talks to 198.51.100.20 port 4000 through a NAT with public address 203.0.113.7. Players complain that after standing still for a minute or two, they stop getting updates.

**On a Linux NAT router:**

```bash
$ sudo sysctl net.netfilter.nf_conntrack_udp_timeout \
    net.netfilter.nf_conntrack_udp_timeout_stream
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 120

$ sudo conntrack -L -p udp
udp 17 112 src=192.168.1.10 dst=198.51.100.20 sport=50514 dport=4000
    src=198.51.100.20 dst=203.0.113.7 sport=4000 dport=50514
    [ASSURED] mark=0 use=1
```

The output is one line, wrapped here to fit. `112` is seconds left before the entry dies. Each matching packet resets it.

**On a Cisco IOS NAT router:**

```text
Router# show ip nat translations udp verbose
Pro Inside global     Inside local       Outside local      Outside global
udp 203.0.113.7:50514 192.168.1.10:50514 198.51.100.20:4000 198.51.100.20:4000
    create 00:01:12, use 00:00:08, timeout:300000, left 00:04:52

Router(config)# ip nat translation udp-timeout 600
```

The mapping is 72 seconds old, it was last used 8 seconds ago, and the timeout is 300,000 ms. So 292 seconds remain (4 minutes 52 seconds).

**The fix** for the Linux router's 120 second timer: send a keep-alive from the client every 25 seconds.

**The cost:** 86,400 seconds in a day / 25 = 3,456 packets. At 60 bytes each (IP, UDP, and small data), that is about 207 KB per day. This is nothing for a desktop. On a phone, each packet wakes the radio, so keep-alives also cost battery.

### Example 4: Seeing loss that UDP hides

```bash
# Linux server 192.168.1.20
iperf3 -s

# Linux client 192.168.1.10: 100 Mbit/s of UDP for 10 seconds
iperf3 -c 192.168.1.20 -u -b 100M -l 1460
```

```text
[ ID] Interval          Transfer    Bitrate        Jitter    Lost/Total Datagrams
[  5] 0.00-10.00 sec    119 MBytes  100 Mbits/sec  0.000 ms  0/85616 (0%)  sender
[  5] 0.00-10.04 sec    116 MBytes  96.7 Mbits/sec 0.312 ms  2501/85616 (2.9%)  receiver
```

The sender reports zero loss, because it has no idea. Only the receiver's count shows 2.9% lost. A TCP test on the same link would simply slow down. UDP keeps going and drops. See [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

## Going Deeper

### Fragmentation: the hidden weakness

If a datagram is larger than the path MTU (Maximum Transmission Unit, the largest packet a link can carry), IP splits it. Details are in [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

This is bad for UDP in three ways:

1. **Loss multiplies.** One lost fragment loses the whole datagram. With 1% packet loss and 6 fragments, delivery is 0.99^6, about 94%. So about 6% of datagrams are lost.
2. **No port in later fragments.** Only the first fragment holds the UDP header. Firewalls and NATs may drop the others, and load balancers cannot hash them the same way.
3. **Resources.** Reassembly uses memory, and fragment floods abuse this.

TCP avoids this by cutting data to fit. UDP cannot, so the application must. It should keep each datagram under the path MTU, or probe for it (RFC 8899 defines this for datagram transports). See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).

This is why DNS moved away from large UDP answers. The DNS Flag Day 2020 recommended an EDNS buffer size of 1,232 bytes, which comes from 1,280 - 40 - 8.

### Receive buffers and silent drops

The kernel keeps a queue per UDP socket. If the application reads too slowly, the queue fills and new datagrams are dropped. Nothing tells the sender. This is the most common real cause of "UDP loss" on servers, and the network is innocent.

```bash
# Linux counters
$ netstat -su
Udp:
    1284567 packets received
    312 packets to unknown port received
    48211 packet receive errors
    1290040 packets sent
    48211 receive buffer errors
    0 send buffer errors

# Modern equivalent
$ nstat -az UdpRcvbufErrors

# See the queue for each socket (Recv-Q column)
$ ss -ulnm

# Raise the limit an application may request (26 MB here)
$ sudo sysctl -w net.core.rmem_max=26214400
```

Raising the limit alone does nothing unless the application also requests a bigger buffer through `SO_RCVBUF`. If `receive buffer errors` keeps climbing, look at the application first.

```powershell
# Windows counters
netstat -s -p udp
```

```text
UDP Statistics for IPv4

  Datagrams Received    = 184223
  No Ports              = 1042
  Receive Errors        = 17
  Datagrams Sent        = 90311
```

### High-speed UDP

TCP gets hardware help that UDP lacked for years, so early QUIC was often heavier on CPU. Modern Linux gives UDP similar tools:

- `SO_REUSEPORT` lets many sockets share one port, so each CPU core can have its own.
- `sendmmsg()` and `recvmmsg()` handle many datagrams in one system call.
- UDP GSO and GRO let the kernel and network card split and join data in large batches.

**Checksum offload** means the network card calculates the checksum, not the CPU. Because of this, tcpdump on the sender can show `bad udp cksum`, since the capture happens before the card fills it in. This is usually harmless.

### Load balancing UDP

Routers using ECMP (Equal-Cost Multi-Path) hash the source and destination IP and port to pick a path. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

- Same 4 values, same path, so order is usually kept in a flow.
- A UDP load balancer has no FIN to watch, so it uses an idle timer, just as NAT does.
- If a client's IP or port changes, the flow may land on a new server. QUIC's connection ID lets a load balancer route without depending on the address.
- Tunnels use this on purpose. VXLAN hashes the inner flow into the outer UDP source port, so the underlay spreads tunnels across paths. VXLAN over IPv4 usually leaves the checksum at zero, since the inner packet has its own checks.

### UDP and spoofed replies

Since UDP has no handshake, an attacker can try to forge a reply before the real one arrives. In 2008, Dan Kaminsky showed how this poisons DNS caches at scale. A 16-bit DNS ID alone is too weak.

The fix was to randomise the **source port** too, adding many more bits of guesswork (RFC 5452). DNSSEC gives real proof of authenticity, and it is covered in [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

### Fairness and congestion

If TCP and an uncontrolled UDP flow share a full link, TCP sees loss and slows down. The UDP flow does not, so it takes the capacity. The cost falls on others. Serious UDP applications add congestion control for this reason, and QoS (Quality of Service) policies can limit UDP. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

### NAT types and hole punching

Two hosts behind different NATs can talk directly over UDP, with a helper server.

1. Host A asks a STUN server (RFC 8489): "What is my public address and port?"
2. Host B does the same.
3. They swap the answers through a signalling server.
4. Both send UDP to each other's public address at about the same time.
5. Each outbound packet creates a mapping in its own NAT, so the other side's packet now finds a hole.

```text
Host A           NAT A          NAT B           Host B
  | ---- send ----> |             | <---- send ---- |
  |                 | ---X   X--- |                 |
  |   first packets may be dropped, but each NAT    |
  |   now has a mapping toward the other side       |
  | <============== direct flow ==================> |
```

This works when the NAT keeps one public port for an inside socket whatever the destination is (endpoint-independent mapping, RFC 4787). If the NAT picks a new public port per destination (the older name is symmetric NAT), the port learned from STUN is wrong for the peer. The fallback is a TURN relay (RFC 8656), which forwards the traffic at a cost. ICE (RFC 8445) tries the options in order. WebRTC uses all three, see [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

### Relatives of UDP

- **UDP-Lite** (RFC 3828, IP protocol 136): the checksum may cover only part of the data, so a damaged media payload still gets delivered. It is rarely used, because middleboxes often block it.
- **DCCP** (RFC 4340): unreliable delivery with congestion control. It is also rarely deployed.
- **SCTP** (RFC 9260): message based and reliable, used in telecom signalling and inside WebRTC data channels, where it runs over DTLS and UDP.

This pattern repeats. New IP protocols get blocked by firewalls and NATs, so new transports are built on UDP, which already passes through nearly everything. QUIC is the best example.

## Common Mistakes

- **"UDP is always faster than TCP."** UDP starts faster and has less overhead. On a lossy link, a badly built UDP application can deliver less useful data than TCP.
- **"Unreliable means it usually loses packets."** It means no guarantee. On a healthy LAN, loss is close to zero. UDP adds no loss of its own.
- **"UDP has no error checking."** It has a checksum. It has no error **recovery**. Bad datagrams are dropped, not fixed.
- **"`sendto()` succeeded, so the data arrived."** It only reached the local stack.
- **"`connect()` on a UDP socket does a handshake."** It sends nothing.
- **"No reply means the port is closed."** No reply could mean open and silent, or filtered by a firewall. That is why nmap reports `open|filtered` on UDP.
- **"`nc -zu` showed success, so the service works."** With UDP, netcat often reports success simply because no ICMP error came back.
- **"`Test-NetConnection` can test my UDP port."** Its port test uses TCP only. Use an application test, such as `nslookup` for DNS.
- **"DNS uses only UDP."** TCP is also a normal part of DNS.
- **"I can send 65,507 bytes safely."** You can, but it will fragment, and it is often lost. Stay near 1,200 bytes on the Internet.
- **"`bad udp cksum` means the network corrupted my packet."** On the sender, it usually means checksum offload.
- **"Packet loss means the network is bad."** Check `receive buffer errors` first. The application may be too slow.
- **"Multicast works across the Internet."** Usually it does not. It needs routers along the path to support it.
- **"A longer NAT timer fixes everything."** It wastes ports and memory, and you do not control every NAT on the path. Keep-alives are the portable fix.
- **"Firewalls cannot track UDP."** Stateful firewalls track it with an idle timer.
- **"UDP has no congestion control, so I can send as fast as I want."** You can, but RFC 8085 says the application must behave, and you may harm others.

## Interview Questions

**Q: What is in the UDP header, and how big is it?**

A: It is always 8 bytes, with four 16-bit fields: source port, destination port, length, and checksum. There are no sequence numbers, ACKs, window, or flags, so ordering and reliability are left to the application.

**Q: Why does DNS use UDP, and when does it use TCP?**

A: A normal lookup is one small question and one small answer, so a TCP handshake only adds delay and server state. The client handles loss by retrying. DNS uses TCP when an answer does not fit and the server sets the truncated flag, and for zone transfers.

**Q: What is the largest UDP payload?**

A: The length field allows 65,535 bytes including the 8 byte header. After 20 bytes of IPv4 header, that gives 65,507 bytes of data. In practice you stay under the path MTU, which is 1,472 bytes on standard Ethernet with IPv4, because fragmentation makes loss much worse.

**Q: How does the UDP checksum work, and is it mandatory?**

A: It is a 16-bit one's complement sum over a pseudo-header, the UDP header, and the data. The pseudo-header has the IP addresses, protocol, and length, so wrongly delivered packets are also caught. It is optional on IPv4, where zero means not used, and required on IPv6, except for some tunnel cases. A failed datagram is dropped silently.

**Q: What happens when you send UDP to a port with no listener?**

A: The host normally returns ICMP Port Unreachable, type 3 code 3. On Linux, the application normally sees this only on a connected socket. If a firewall drops it, the sender hears nothing, so silence does not prove the port is open or closed.

**Q: If UDP is unreliable, why is QUIC built on it?**

A: QUIC builds its own reliability, congestion control, and encryption, so it only needs something that passes through the network. A new IP protocol would be blocked by firewalls and NATs. UDP passes nearly everywhere, and it lets QUIC live in user space, so it can be updated without changing the OS kernel.

**Q: How would you add reliability on top of UDP?**

A: Add a sequence number to each message, have the receiver send ACKs, and keep a copy with a retransmit timer based on measured RTT and exponential backoff with jitter. The receiver sorts and removes duplicates, so operations need to be idempotent. I would also add flow and congestion control, and encryption if needed. I would only add the parts the application actually needs.

**Q: Explain UDP reflection and amplification, and how to stop it.**

A: The attacker sends small requests with the victim's IP as the source. Servers send large replies to the victim. It works because UDP has no handshake to prove the source. For example, a 64 byte query with a 3,000 byte reply gives about 47 times the traffic. Defences are source filtering with BCP 38 and uRPF, closing open resolvers and exposed services, rate limiting, and protocols that do not reply bigger than the request until the address is validated, as QUIC does.

**Q: Why can UDP do broadcast and multicast when TCP cannot?**

A: TCP needs a handshake and ACK state with exactly one peer, which cannot work with a group. UDP keeps no peer state, so one datagram can go to many hosts. Broadcast reaches everyone on the subnet. Multicast reaches hosts that joined the group through IGMP, or MLD on IPv6.

**Q: A VoIP call loses audio in one direction after being on hold. What do you suspect?**

A: A NAT or firewall UDP idle timeout. UDP has no end-of-connection signal, so devices delete a mapping after silence. Once it is gone, inbound packets are dropped. I would check the translation table and the timer, then enable keep-alives shorter than the timer. RFC 4787 asks for at least 2 minutes, but real devices vary.

**Q: Users report UDP loss on a server. How do you debug it?**

A: First I find where the loss is. I check `netstat -su` for receive buffer errors, which point to a slow application or small socket buffer. Then I capture on both ends with tcpdump to see if packets reach the host. I test the path with `iperf3 -u` and check MTU and fragmentation. I also check firewall and conntrack limits.

**Q: Can one host use TCP port 53 and UDP port 53 at the same time?**

A: Yes. They are separate port spaces, because the IP protocol number separates them. DNS servers do exactly this.

## Quick Recap

- UDP is IP plus ports and a checksum, with an 8 byte header: source port, destination port, length, checksum.
- It is connectionless: no handshake, no ACKs, no ordering, no flow or congestion control, and it keeps message edges.
- The checksum uses a pseudo-header, is optional on IPv4, and is required on IPv6. Bad datagrams are dropped silently.
- Reliability is the application's choice: sequence numbers, ACKs, timers, and backoff, as in TFTP, RTP, and QUIC.
- Only UDP suits broadcast and multicast, since TCP needs one peer.
- NAT and firewalls expire UDP flows on an idle timer, so quiet flows need keep-alives.
- No handshake means spoofing is easy, which enables reflection and amplification. Filter sources, and never reply bigger than the request to an unproved address.
