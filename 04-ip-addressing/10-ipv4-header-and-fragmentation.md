# IPv4 Header and Fragmentation

**Level:** Advanced | **Prerequisites:** [IPv4 Addressing Basics](01-ipv4-basics.md), [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md)

## In One Line

The IPv4 header is the 20 to 60 byte label on every packet that tells routers where it goes and how to treat it. Fragmentation is how IPv4 cuts a packet into smaller pieces when it is too big for a link, and only the final destination joins the pieces again.

## Why It Matters

Every link has a size limit. Ethernet normally carries 1500 bytes of IP packet. A VPN tunnel may carry less. A packet that is too big for the next link must be cut, or it must be dropped. The header fields decide which one happens.

This causes real outages. A classic case: after a VPN goes live, ping works and small web pages load, but file uploads hang. The usual cause is a packet size problem, and you can only find it if you understand the DF flag, the MTU, and ICMP.

Interviewers ask about this topic because it shows whether you know how packets really work. Expect these tasks:

- Name the header fields and say what each one does.
- Say which fields change at every router.
- Fragment a 4000 byte packet on a whiteboard, with offsets.
- Explain why fragmentation is bad and how modern systems avoid it.

## Simple Explanation

### Words you need first

- **Packet:** one unit of data at the IP layer. RFCs also call it a **datagram**.
- **Header:** control information at the front of a packet.
- **Payload** (or **data**): what the packet carries behind the header. Often this is a TCP or UDP segment.
- **Byte:** 8 bits. RFCs say **octet**, which means exactly the same thing.
- **Word:** in the IPv4 header, a block of 32 bits (4 bytes).
- **MTU (Maximum Transmission Unit):** the largest IP packet a link can carry, header included. Ethernet's usual MTU is 1500.
- **Fragment:** one piece of a packet that was cut up. Each fragment is itself a valid IP packet with its own header.
- **RFC:** a numbered technical document from the IETF, the group that writes internet standards.

### The header is a shipping label

Think of a parcel. The label says who it is from, who it is for, how heavy it is, and if it is urgent. Postal workers read only the label. They do not open the parcel.

The IPv4 header works the same way. Routers read the header. They do not need the data inside.

Where the analogy stops: a postal label stays the same for the whole trip. Routers change parts of the IPv4 header at every hop.

### Fragmentation is a book sent in envelopes

You want to mail a 400 page book, but the post office only accepts thin envelopes. So you cut the book into three bundles. On each envelope you write:

- Book number 7238 (so the bundles stay together).
- "This bundle starts at page 150."
- "More bundles are coming" (on all except the last).

The bundles may arrive in any order. Your friend uses the numbers to rebuild the book.

Where the analogy stops:

- In IPv4, a router in the middle may also cut a bundle again. You did not plan this and you are not told.
- Only the final reader joins bundles. Post offices on the way never do.
- If one bundle is lost, IP throws away the whole book. It does not ask for the missing bundle again.

## How It Works

### The header at a glance

RFC 791 defines IPv4. Later RFCs changed some fields, and those changes are noted below. The header is at least 20 bytes. Each row in the diagram is one 32 bit word.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |   DSCP    |ECN|         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Identification         |Flags|    Fragment Offset      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |   Protocol    |        Header Checksum        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Source Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Destination Address                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Options (0-40 bytes, if any)  |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | Purpose |
|---|---|---|
| Version | 4 bits | IP version. Always 4 here. |
| IHL (Internet Header Length) | 4 bits | Header length in 32 bit words. Range 5-15, so 20-60 bytes. |
| DSCP | 6 bits | Asks for a class of service, such as voice priority. |
| ECN | 2 bits | Signals congestion without dropping the packet. |
| Total Length | 16 bits | Size of the whole packet (header plus data) in bytes. Maximum 65535. |
| Identification | 16 bits | Number that ties fragments of one packet together. |
| Flags | 3 bits | Reserved bit, DF (Don't Fragment), MF (More Fragments). |
| Fragment Offset | 13 bits | Where this piece belongs in the original data, in 8 byte units. |
| TTL (Time to Live) | 8 bits | Hop limit. Stops packets from looping forever. |
| Protocol | 8 bits | What is inside the payload, such as TCP (6) or UDP (17). |
| Header Checksum | 16 bits | Error check for the header only. |
| Source Address | 32 bits | Sender's IPv4 address. |
| Destination Address | 32 bits | Receiver's IPv4 address. |
| Options | 0-40 bytes | Rare extras. Most packets have none. |
| Padding | 0-3 bytes | Zero bytes that fill the header to a 32 bit boundary. |

The fixed fields add up to 160 bits, which is 20 bytes.

### Version and IHL

Version and IHL share the first byte. Almost every IPv4 packet starts with hex `45`:

- `4` is the version.
- `5` is the IHL: 5 words times 4 bytes is 20 bytes.

A first byte of `46` means a 24 byte header with 4 bytes of options. The largest value, `4F`, means 15 words, or 60 bytes. That is why options can use at most 40 bytes: 60 minus 20.

The receiver needs IHL because the header length can change. IHL tells it where the data starts.

### Total Length

Total Length counts the header and the data, in bytes. It is 16 bits, so the maximum packet size is 65535 bytes. The minimum is 20 bytes: a header with no data.

Data size is simple to find:

```text
data bytes = Total Length - (IHL x 4)
```

Total Length also solves a small Ethernet problem. An Ethernet frame must carry at least 46 bytes, so a tiny packet gets extra padding bytes. The receiver uses Total Length to find where the real packet ends. See [Ethernet Frames](../03-ethernet-and-switching/02-ethernet-frames.md).

### Identification

Identification (often "the IP ID") is a number chosen by the sender. All fragments of one packet carry the same ID. The ID has no other job in the standard. It is not a sequence number, and it does not put packets in order.

### TTL (Time to Live)

TTL is 8 bits, so it ranges from 0 to 255. The sender sets it. Every router lowers it by 1. A router that lowers TTL to 0 drops the packet and sends an ICMP Time Exceeded message (type 11, code 0) back to the source.

```text
Host A          R1            R2            R3         Server
TTL 3 ------> TTL 2 ------> TTL 1 ------> TTL 0, dropped
                                            |
<------ ICMP Type 11, Code 0 from R3 -------+
```

Analogy: a transfer ticket that allows a fixed number of bus changes. Each change punches one hole. When the ticket is full, you must stop.

Where the analogy stops: nobody "pays" for the stop. The router simply drops the packet and reports it, and the sender decides what to do.

TTL exists because of routing loops. If two routers point at each other by mistake, a packet could circle forever and use bandwidth. TTL ends that loop.

**Fuller truth:** RFC 791 described TTL as time in seconds. In practice routers forward in far less than a second, so it became a hop count. IPv6 renamed it Hop Limit.

Common starting values are operating system defaults, not standards. They can differ by version and they can be changed:

| System | Common default TTL |
|---|---|
| Linux, macOS | 64 |
| Windows | 128 |
| Cisco IOS | 255 |

This gives a handy trick. A reply with TTL 54 probably started at 64 and crossed 10 routers.

TTL has other uses too:

- **Traceroute** sends packets with TTL 1, 2, 3 and so on, and collects the ICMP errors. See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).
- **Scope limits:** link-local multicast such as OSPF hellos uses TTL 1, so routers do not forward it.
- **GTSM (Generalized TTL Security Mechanism, RFC 5082):** peers send with TTL 255 and accept only values close to 255. A remote attacker cannot make a packet arrive with TTL 255, because every hop lowers it.

### Protocol numbers

The Protocol field names the protocol carried in the payload. IANA, the group that keeps internet number lists, maintains the registry.

| Number | Protocol | Note |
|---|---|---|
| 1 | ICMP | Ping and error messages |
| 2 | IGMP | Multicast group membership |
| 4 | IPv4 in IPv4 | IP tunnelling |
| 6 | TCP | Web, SSH, BGP and more |
| 17 | UDP | DNS, DHCP, QUIC and more |
| 41 | IPv6 in IPv4 | 6in4 tunnels |
| 47 | GRE | Generic tunnels |
| 50 | ESP | IPsec encryption |
| 51 | AH | IPsec authentication |
| 88 | EIGRP | Cisco-origin routing protocol |
| 89 | OSPF | Link-state routing |
| 103 | PIM | Multicast routing |
| 112 | VRRP | First hop redundancy |
| 132 | SCTP | Message-based transport |

Analogy: the Protocol field is like the department name on an office letter. It says which desk gets the content. A port number is then the person at that desk.

Where the analogy stops: not everything has a protocol number. BGP runs on TCP port 179, and RIP on UDP port 520, so they have port numbers instead. IS-IS does not run over IP at all.

### Header checksum

The checksum is a simple error test for the header only. It does not cover the data. TCP and UDP have their own checksums for that.

To make it, the sender does this:

1. Set the checksum field to 0.
2. Split the header into 16 bit words.
3. Add the words using one's complement addition. Any carry past 16 bits is added back into the low bits.
4. Flip every bit of the result. That value is the checksum.

To check it, the receiver adds all header words, including the checksum. The correct result is `0xFFFF`. Anything else means the header is damaged, and the packet is dropped. No error message is sent, since a damaged header cannot be trusted.

Analogy: a shop receipt has a total at the bottom. If you add the lines and get a different total, something is wrong.

Where the analogy stops: a weak total can miss some errors. If two 16 bit words swap places, the sum stays the same, so the checksum sees nothing wrong.

Because TTL changes at each hop, **every router must update the checksum**. Routers do not need to add the whole header again. They can apply a small change to the old value (RFC 1624 describes this). [NAT](09-nat-and-pat.md) devices also update it, because they change an address.

IPv6 removed this field. Link layers and transport layers already check for errors, and the per-hop update was extra work.

### DSCP and ECN

The second header byte was first called Type of Service (ToS). Its meaning changed over time:

```text
Old (RFC 791 era):
+---+---+---+---+---+---+---+---+
|Precedence |  ToS bits     | 0 |   Precedence: 3 bits
+---+---+---+---+---+---+---+---+

Current (RFC 2474 and RFC 3168):
+---+---+---+---+---+---+---+---+
|          DSCP         |  ECN  |
+---+---+---+---+---+---+---+---+
```

**DSCP (Differentiated Services Code Point)** is a 6 bit class label. Routers read it and pick a treatment, such as a faster queue. The label alone does nothing. Each network must configure what it means.

| Name | DSCP decimal | DSCP binary | ToS byte (ECN 00) | Common use |
|---|---|---|---|---|
| DF (CS0) | 0 | 000000 | 0x00 | Normal best-effort traffic |
| CS1 | 8 | 001000 | 0x20 | Low priority background |
| AF21 | 18 | 010010 | 0x48 | Important data |
| AF31 | 26 | 011010 | 0x68 | Call signalling, business apps |
| AF41 | 34 | 100010 | 0x88 | Video calls |
| EF | 46 | 101110 | 0xB8 | Voice |
| CS6 | 48 | 110000 | 0xC0 | Routing protocols |

The "ToS byte" column is the value tools often show. It is DSCP shifted left by 2 bits. For EF: 46 x 4 = 184 = 0xB8. So `tos 0xb8` in tcpdump means EF.

Careful with names: DSCP "DF" means Default Forwarding. It has nothing to do with the Don't Fragment flag.

**ECN (Explicit Congestion Notification)** uses the last 2 bits:

| Bits | Name | Meaning |
|---|---|---|
| 00 | Not-ECT | Sender does not support ECN |
| 01 | ECT(1) | Sender supports ECN |
| 10 | ECT(0) | Sender supports ECN |
| 11 | CE | Congestion Experienced: a router marked this packet |

Normally a full router queue drops packets, and TCP slows down when it sees the loss. With ECN, a router near overload changes ECT to CE and still forwards the packet. The receiver tells the sender, and the sender slows down. No data was lost.

Analogy for DSCP: a "priority" sticker on a parcel. Analogy for ECN: a road sign saying "traffic ahead, slow down" instead of a crash that stops a car.

Where the analogy stops: a sticker is only a request. Any network may ignore or rewrite DSCP, and this often happens at network borders. ECN works only when both ends agree to use it and the routers on the way support it.

Queues, marking and policing belong to [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

### Flags and DF

The 3 flag bits sit just before Fragment Offset:

```text
Bit 0      Bit 1      Bit 2
+----------+----------+----------+
| Reserved |    DF    |    MF    |
|  must 0  | Don't    | More     |
|          | Fragment | Fragments|
+----------+----------+----------+
```

- **Reserved:** always 0.
- **DF (Don't Fragment):** 1 means no device may cut this packet.
- **MF (More Fragments):** 1 means more data follows this piece. MF is 0 on the last piece, and on any packet that was never cut.

What a router does when a packet is larger than the next link's MTU:

| DF value | Router action |
|---|---|
| DF = 0 | Fragment the packet and forward the pieces. |
| DF = 1 | Drop the packet. Send ICMP Destination Unreachable, type 3, code 4 ("Fragmentation Needed"), which includes the next link's MTU. |

That ICMP message is the base of **Path MTU Discovery (PMTUD)**, where the sender learns the smallest MTU on the path and sends smaller packets. Most TCP stacks set DF = 1 for this reason. The full story, including black holes and MSS clamping, is in [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Fragment Offset and the 8 byte unit

Fragment Offset says where this fragment's data belongs inside the original data. The unit is 8 bytes, not 1 byte.

Why 8? Total Length can reach 65535 bytes. To point at any byte would need 16 bits. The designers kept only 13 bits, to leave room for the 3 flag bits in one 16 bit block. With 13 bits, values go from 0 to 8191. Multiply by 8 and the reach is 65528 bytes, which is enough.

Analogy: a ruler with marks only every 8 cm. You can only start a piece of wood at a mark.

Where the analogy stops: on this ruler, the rule applies only to where pieces **start**. The last piece may end anywhere.

This has one direct result: **every fragment except the last must carry a data size that is a multiple of 8**. If not, the next fragment could not start at a valid mark.

### Fragmentation step by step

A device (the source host, or any router) must fragment a packet. It does this:

1. Find the data size: `data = Total Length - header length`.
2. Find the room per fragment: `room = MTU - header length`.
3. Round the room **down** to a multiple of 8. This is the max data per fragment.
4. Cut the data into pieces of that size. The last piece takes what is left.
5. Copy the header onto each piece.
6. In each copy, set Total Length, MF, Fragment Offset, and a new checksum.

For each piece, the offset is:

```text
offset = (position of the first data byte in the original) / 8
```

Fields that stay the same in all fragments: Version, DSCP and ECN, Identification, Protocol, Source, and Destination.

Fields that differ: Total Length, MF, Fragment Offset, Header Checksum. IHL may differ as well, because some options are copied only into the first fragment.

**Only the first fragment holds the TCP or UDP header.** Later fragments contain raw data and no port numbers. This fact explains most of the trouble below.

```text
Original packet: Total Length 4000

+----+----------------------------------------------+
| IP | data: 3980 bytes (starts with UDP/TCP header)|
+----+----------------------------------------------+

After a link with MTU 1500

+----+----------------+
| IP | bytes 0-1479   |  Length 1500, offset 0,   MF=1
+----+----------------+
+----+----------------+
| IP | bytes 1480-2959|  Length 1500, offset 185, MF=1
+----+----------------+
+----+-----------+
| IP | 2960-3979 |       Length 1040, offset 370, MF=0
+----+-----------+
```

### Reassembly at the destination

**Reassembly** means joining the fragments back into the original packet. In IPv4 the destination host does this, not routers.

Why not routers?

- Fragments may take different paths, so no single router is sure to see them all.
- Holding pieces and rebuilding packets costs memory and time on every router.
- The next link might need the packet cut again anyway.

The destination does this:

1. **Spot a fragment.** A packet is a fragment if MF = 1 or Fragment Offset is not 0.
2. **Find its group.** The key is four values: source address, destination address, protocol, and Identification.
3. **Make a buffer if needed.** The first fragment to arrive creates a buffer and starts a timer. "First to arrive" may be any piece.
4. **Place the data.** Data goes to position `offset x 8`. Order of arrival does not matter.
5. **Learn the total size.** When the piece with MF = 0 arrives, the total data is `offset x 8 + data size of that piece`.
6. **Finish.** When there are no gaps, the packet is complete. It goes up to the protocol in the Protocol field.
7. **Give up on timeout.** If the timer ends first, all pieces are dropped. If the piece with offset 0 did arrive, the host sends ICMP Time Exceeded, type 11, **code 1**.

```text
Key: 192.168.1.10 > 203.0.113.20, proto 1, ID 40321
Pieces arrive in the order F3, F1, F2

F3 (offset 370, MF=0): total = 370 x 8 + 1020 = 3980 bytes
[       gap       ][       gap       ][  2960-3979  ]

F1 (offset 0, MF=1):
[     0-1479      ][       gap       ][  2960-3979  ]

F2 (offset 185, MF=1): no gaps left, deliver packet
[     0-1479      ][    1480-2959    ][  2960-3979  ]
```

Analogy: a jigsaw puzzle on a table, with a kitchen timer. The numbers on the back of each piece show its place. If the timer rings before you finish, you throw away everything.

Where the analogy stops: nobody knows the size of the puzzle until the last piece appears, and one missing piece means nothing is delivered to the application at all.

**Timers vary.** RFC 1122 suggests a fixed timeout of 60 to 120 seconds. The Linux default is 30 seconds (`net.ipv4.ipfrag_time`). Other systems differ.

**Size limits.** Every host must be able to reassemble packets of at least 576 bytes. Most modern hosts handle much more. Every link must have an MTU of at least 68 bytes: a 60 byte maximum header plus an 8 byte minimum fragment.

IP does not retransmit. If a fragment is lost, the whole packet is lost, and it is up to TCP or the application to send it again.

### Why fragmentation is bad

RFC 8900 (BCP 230), "IP Fragmentation Considered Fragile", tells designers to avoid relying on fragmentation. Here are the reasons.

**1. One lost piece kills the whole packet.** Loss grows with the piece count. Say the link loses 1 percent of packets. A maximum size UDP datagram needs 45 fragments, and all 45 must arrive:

```text
Chance all arrive = 0.99 ^ 45 = about 0.64
Chance packet lost = about 36 percent
```

A 1 percent link has become a 36 percent link. Worse, a resend must send all pieces again.

**2. Later fragments have no ports.** Only the first piece has the TCP or UDP header. This hurts every device that looks at ports:

- ACLs and firewalls cannot match the rule on port.
- PAT cannot find the right translation, since it maps by port.
- Load balancers cannot pick the right backend.
- ECMP (equal-cost multi-path, spreading traffic over several links) may hash the first piece using ports and the rest without them. Pieces then take different paths and arrive out of order.

**3. Many firewalls drop fragments.** Fragments are a known attack tool, so some networks block them. The packet then simply disappears.

**4. It costs routers work.** On many routers, normal forwarding runs in fast hardware, while fragmenting needs the slow path in the CPU. High-end hardware may do better, but the cost is rarely zero.

**5. It costs hosts memory.** The receiver must hold pieces until the timer ends. Attackers abuse this by sending first pieces that never complete.

**6. More overhead.** Each piece needs its own header. In the 4000 byte example, 40 extra bytes of IP header cross the wire. On Ethernet, each piece also adds its own frame overhead.

**7. Wrong joins at high speed.** The ID field has only 65536 values. A fast sender can reuse an ID while old pieces still wait in a buffer, and pieces of two packets may be joined by mistake. See [Going Deeper](#the-identification-field-today).

**How modern systems avoid it:**

- TCP sets DF and uses PMTUD and MSS to keep segments small.
- QUIC must not rely on IP fragmentation. RFC 9000 requires DF to be set on IPv4 where possible. See [QUIC](../06-transport-layer/08-quic.md).
- DNS uses smaller UDP sizes and falls back to TCP. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).
- Network teams set tunnel MTUs and clamp TCP MSS on routers.

## Example

### Example 1: The classic 4000 byte packet

A host sends a 4000 byte IP packet with a 20 byte header and DF = 0. The link MTU is 1500.

**Step 1.** Data size: 4000 - 20 = 3980 bytes.

**Step 2.** Room per fragment: 1500 - 20 = 1480 bytes.

**Step 3.** Round down to a multiple of 8: 1480 / 8 = 185 exactly, so 1480 stays.

**Step 4.** Count the pieces: 3980 / 1480 = 2.69, so 3 fragments.

**Step 5.** Fill in the table:

| Fragment | Data bytes | Byte range | Offset | MF | Total Length |
|---|---|---|---|---|---|
| 1 | 1480 | 0-1479 | 0 | 1 | 1500 |
| 2 | 1480 | 1480-2959 | 185 | 1 | 1500 |
| 3 | 1020 | 2960-3979 | 370 | 0 | 1040 |

**Check it:**

- Data adds up: 1480 + 1480 + 1020 = 3980.
- Offsets: 1480 / 8 = 185, and 2960 / 8 = 370.
- Bytes on the wire at IP level: 1500 + 1500 + 1040 = 4040. That is 40 more than the original, for two extra headers.

Here is how the Flags and Offset block looks in hex. MF is the value `0x2000`.

| Fragment | MF | Offset | 16 bit value |
|---|---|---|---|
| 1 | 1 | 0 | `0x2000` |
| 2 | 1 | 185 (`0x00B9`) | `0x20B9` |
| 3 | 0 | 370 (`0x0172`) | `0x0172` |

```text
Fragment 2 in bits:

 R DF MF  Fragment Offset (13 bits)
+-+--+--+---------------------------+
|0| 0| 1| 0 0000 1011 1001           |  offset = 185
+-+--+--+---------------------------+
= 0010 0000 1011 1001 = 0x20B9
```

### Example 2: When the room is not a multiple of 8

A 3000 byte packet (20 byte header) meets a link with MTU 1000.

- Data: 3000 - 20 = 2980 bytes.
- Room: 1000 - 20 = 980 bytes.
- 980 / 8 = 122.5. This is not a whole number.
- Round down: 122 x 8 = **976 bytes** per fragment.

This is the trap. If you use 980, the next offset would be 122.5, and the field cannot hold that.

| Fragment | Data bytes | Byte range | Offset | MF | Total Length |
|---|---|---|---|---|---|
| 1 | 976 | 0-975 | 0 | 1 | 996 |
| 2 | 976 | 976-1951 | 122 | 1 | 996 |
| 3 | 976 | 1952-2927 | 244 | 1 | 996 |
| 4 | 52 | 2928-2979 | 366 | 0 | 72 |

Data check: 976 x 3 + 52 = 2980. Notice that the fragments are 996 bytes, even though the MTU allows 1000. The other 4 bytes cannot be used.

### Example 3: A fragment gets fragmented again

Take the three fragments from Example 1. They reach a link with MTU 576. Room is 556 bytes, and 556 / 8 = 69.5, so the max data is 69 x 8 = 552 bytes.

| From | New piece | Data | Offset | MF | Total Length |
|---|---|---|---|---|---|
| F1 | 1 | 552 | 0 | 1 | 572 |
| F1 | 2 | 552 | 69 | 1 | 572 |
| F1 | 3 | 376 | 138 | 1 | 396 |
| F2 | 4 | 552 | 185 | 1 | 572 |
| F2 | 5 | 552 | 254 | 1 | 572 |
| F2 | 6 | 376 | 323 | 1 | 396 |
| F3 | 7 | 552 | 370 | 1 | 572 |
| F3 | 8 | 468 | 439 | 0 | 488 |

Two rules show here:

- **Offsets stay absolute.** They always count from the start of the original data. The router adds to the fragment's own offset, for example 185 + 69 = 254.
- **The final piece keeps the old MF.** Piece 3 has MF = 1, since F1 had MF = 1. Only piece 8 has MF = 0, since it comes from the true last fragment.

The destination does not care how many times a packet was cut. The offsets still fit together, and all 8 pieces share one ID.

### Example 4: See fragments live

**Setup (Linux):** host 192.168.1.10, gateway 192.168.1.1, MTU 1500, target 203.0.113.20.

Ping needs an IP packet of 4000 bytes. Ping's `-s` value is the ICMP payload size, without the 8 byte ICMP header and the 20 byte IP header. So `-s` must be 4000 - 28 = 3972.

```bash
# Linux, terminal 1: capture outgoing ICMP
sudo tcpdump -n -v -i eth0 'dst host 203.0.113.20 and icmp'

# Linux, terminal 2: send one large ping
ping -c 1 -s 3972 203.0.113.20
```

Sample output (timestamps removed):

```text
IP (tos 0x0, ttl 64, id 40321, offset 0, flags [+], proto ICMP (1), length 1500)
    192.168.1.10 > 203.0.113.20: ICMP echo request, id 3105, seq 1, length 1480
IP (tos 0x0, ttl 64, id 40321, offset 1480, flags [+], proto ICMP (1), length 1500)
    192.168.1.10 > 203.0.113.20: ip-proto-1
IP (tos 0x0, ttl 64, id 40321, offset 2960, flags [none], proto ICMP (1), length 1040)
    192.168.1.10 > 203.0.113.20: ip-proto-1
```

How to read it:

- All three lines share `id 40321`. That is the group.
- `flags [+]` means MF = 1. `flags [none]` means MF = 0.
- `offset 1480` is in **bytes**. tcpdump multiplies by 8 for you. The field itself holds 185. Always check what unit your tool shows.
- Later pieces say `ip-proto-1`, because they have no ICMP header for tcpdump to read.

Check the counters after the test:

```bash
# Linux
nstat -az | grep -E 'Frag|Reasm'
```

```text
IpReasmTimeout                  0                  0.0
IpReasmReqds                    3                  0.0
IpReasmOKs                      1                  0.0
IpReasmFails                    0                  0.0
IpFragOKs                       1                  0.0
IpFragFails                     0                  0.0
IpFragCreates                   3                  0.0
```

The sender cut 1 packet into 3 pieces. Three pieces of the fragmented reply came back and made 1 packet.

```powershell
# Windows 11
netstat -s -p ip | Select-String 'Reassembly|Fragment'
```

```text
  Reassembly Required                = 3
  Reassembly Successful              = 1
  Reassembly Failures                = 0
  Datagrams Successfully Fragmented  = 1
  Datagrams Failing Fragmentation    = 0
  Fragments Created                  = 3
```

```text
! Cisco IOS (exact wording varies by version)
R1# show ip traffic | include Frags|fragment
  Frags: 12 reassembled, 0 timeouts, 0 couldn't reassemble
         48 fragmented, 144 fragments, 3 couldn't fragment
```

On a router, "couldn't fragment" usually means packets that were too big with DF set.

### Example 5: Test a path with DF set

Now a link past the gateway has MTU 1400. Set DF and test.

```text
Host                     R1                       Server
192.168.1.10 --MTU 1500-- 192.168.1.1 --MTU 1400-- ... 203.0.113.20
```

A 1500 byte packet needs 1472 bytes of ping data, because 1500 - 28 = 1472.

```bash
# Linux: -M do sets DF
$ ping -M do -s 1472 -c 1 203.0.113.20
PING 203.0.113.20 (203.0.113.20) 1472(1500) bytes of data.
From 192.168.1.1 icmp_seq=1 Frag needed and DF set (mtu = 1400)

--- 203.0.113.20 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

R1 refused the packet and told us the MTU. Try 1400 - 28 = 1372:

```bash
# Linux
$ ping -M do -s 1372 -c 1 203.0.113.20
PING 203.0.113.20 (203.0.113.20) 1372(1400) bytes of data.
1380 bytes from 203.0.113.20: icmp_seq=1 ttl=54 time=18.4 ms
```

It works. The reply shows 1380 bytes: 1372 of data plus 8 of ICMP header. TTL 54 suggests 10 hops from a start of 64.

If the packet is too big for your own interface, the error is local, and no packet leaves the host:

```bash
# Linux
$ ping -M do -s 1473 -c 1 203.0.113.20
PING 203.0.113.20 (203.0.113.20) 1473(1501) bytes of data.
ping: local error: message too long, mtu=1500
```

The same test on Windows and Cisco:

```powershell
# Windows: -f sets DF, -l sets data size
PS> ping -f -l 1472 -n 1 203.0.113.20

Pinging 203.0.113.20 with 1472 bytes of data:
Reply from 192.168.1.1: Packet needs to be fragmented but DF set.
```

```text
! Cisco IOS: size is the WHOLE IP packet, not the ping data
R1# ping 203.0.113.20 size 1500 df-bit repeat 2
Type escape sequence to abort.
Sending 2, 1500-byte ICMP Echos to 203.0.113.20, timeout is 2 seconds:
Packet sent with the DF bit set
MM
Success rate is 0 percent (0/2)
```

Each `M` means "could not fragment". Remember the size rule:

| Tool | Size value means | For a 1500 byte IP packet |
|---|---|---|
| Linux `ping -s` | ICMP data only | 1472 |
| Windows `ping -l` | ICMP data only | 1472 |
| Cisco IOS `ping size` | Whole IP packet | 1500 |

### Example 6: Decode a header and check its checksum

These 20 bytes are the IPv4 header of a TCP SYN (the first packet of a TCP connection):

```text
45 00 00 34 1c 46 40 00 40 06 20 b7 c0 a8 01 0a cb 00 71 14
```

| Bytes | Hex | Field | Meaning |
|---|---|---|---|
| 0 | `45` | Version, IHL | Version 4, 5 words = 20 bytes |
| 1 | `00` | DSCP, ECN | Best effort, no ECN |
| 2-3 | `00 34` | Total Length | 52 bytes (32 bytes of TCP) |
| 4-5 | `1c 46` | Identification | 7238 |
| 6-7 | `40 00` | Flags, Offset | DF = 1, MF = 0, offset 0 |
| 8 | `40` | TTL | 64 |
| 9 | `06` | Protocol | TCP |
| 10-11 | `20 b7` | Checksum | 0x20B7 |
| 12-15 | `c0 a8 01 0a` | Source | 192.168.1.10 |
| 16-19 | `cb 00 71 14` | Destination | 203.0.113.20 |

**Compute the checksum.** Replace the checksum field with zero, then add the 16 bit words:

```text
  4500
+ 0034
+ 1c46
+ 4000
+ 4006
+ 0000   (checksum field set to zero)
+ c0a8
+ 010a
+ cb00
+ 7114
------
 2df46   (sum, with a carry of 2 past 16 bits)

Fold the carry:  df46 + 2 = df48
Flip all bits:   ffff - df48 = 20b7   (matches the header)
```

**Verify as a receiver.** Add the same words with the real checksum in place: `2df46 + 20b7 = 2fffd`. Fold it: `fffd + 2 = ffff`. The result is `0xFFFF`, so the header is good.

**After one router,** TTL becomes 63. The word `4006` becomes `3f06`, which is 0x100 smaller. So the checksum grows by 0x100, from `20b7` to `21b7`. This is the small update routers use instead of adding everything again.

## Going Deeper

### IP options

Options sit after the Destination Address and take 0-40 bytes. Each router must process them, so they are slow, and many networks drop or ignore them.

| Option | Type value | Copied to all fragments? | Purpose |
|---|---|---|---|
| Record Route | 7 | No | Routers add their address to the packet |
| Timestamp | 68 | No | Routers add a time value |
| Loose Source Route | 131 | Yes | Sender lists routers the packet must visit |
| Strict Source Route | 137 | Yes | Sender lists the exact path |
| Router Alert | 148 | Yes | Asks routers to look inside the packet (used by IGMP and RSVP) |

The top bit of the type value is the "copied" flag. If it is 1, the option goes into every fragment. If it is 0, it stays only in the first fragment. So a later fragment can have a shorter header than the first.

Try it on Linux with `ping -R 203.0.113.20`. Record Route can hold at most 9 addresses, because 40 bytes minus 3 bytes of option header leaves room for 9 four byte addresses.

Source routing lets the sender choose the path, and that can bypass security controls. It is commonly disabled:

```text
! Cisco IOS, global config
no ip source-route
```

### The Identification field today

RFC 6864 updated the rules for this field. An **atomic datagram** is a packet that is not fragmented and cannot be: DF = 1, MF = 0, offset 0. For these packets, the ID has no use, and the source need not make it unique. So you may see zero or per-flow values in captures. That is normal.

For packets that can be fragmented, the ID must be unique per source, destination, and protocol for as long as pieces might still exist. At high speed that is hard. A sender at 1 Gbps with 1500 byte packets sends about 83000 packets per second, so a single flow uses all 65536 IDs in under a second. Reassembly timers last tens of seconds. RFC 4963 covers this failure, where pieces of different packets get mixed. The IP header checksum does not help, and a 16 bit transport checksum may miss the damage.

### Fragment-based attacks

Fragment handling has a long security history. For the general view, see [Common Network Attacks](../09-security/03-common-network-attacks.md).

| Attack | How it works | Fix |
|---|---|---|
| Ping of Death | Pieces rebuild to more than 65535 bytes and overflow a buffer. For example, offset 8191 (65528 bytes) with too much data. | Check size during reassembly. |
| Teardrop | Overlapping offsets confuse old reassembly code. | Handle or drop overlaps safely. |
| Tiny fragments | First piece is so small that the TCP flags sit in the second piece, and a filter cannot see them. | RFC 1858: drop TCP packets with offset 1. |
| Fragment overlap | A later piece overwrites bytes that a filter already checked. | Reassemble before checking, drop overlaps. |
| Fragment flood | Many incomplete packets fill the reassembly memory. | Memory limits, timers, rate limits. |
| Evasion | An IDS sees pieces differently from the target. For example, `nmap -f` splits probes. | IDS reassembles traffic as the target would. |

Why does the RFC 1858 rule use offset 1? Offset 1 means the piece starts at byte 8 of the TCP header. The TCP flags are at byte 13, so such a piece could rewrite them. No honest sender needs this.

### Fragments and middleboxes

A **middlebox** is any device on the path that does more than plain routing, such as a firewall or a NAT device.

- **Virtual reassembly:** many firewalls and NAT devices must see ports, so they collect the fragments, check them as one packet, and then forward the original pieces. Cisco IOS calls this Virtual Fragmentation Reassembly (`ip virtual-reassembly`). This does not break the "destination reassembles" rule, because the pieces are forwarded still separate.
- **Cisco ACLs:** the `fragments` keyword matches non-initial fragments only. See [Access Control Lists](../09-security/05-access-control-lists.md).

```text
! Cisco IOS: drop all non-initial fragments, allow the rest
ip access-list extended BLOCK-FRAGS
 deny ip any any fragments
 permit ip any any
```

Be careful with a rule like this. It will break anything that truly needs fragments.

- **ECMP and load balancers:** they usually hash only on addresses for fragments, because ports are missing. Traffic can then land on a different path or backend than the non-fragmented packets of the same flow.

### Tunnels make it worse

A tunnel wraps each packet inside a new IP header, so a 1500 byte packet no longer fits a 1500 byte link. GRE adds 24 bytes (20 of outer IP and 4 of GRE), so the usual GRE tunnel MTU is 1476. IPsec overhead depends on the cipher and mode. See [IPsec](../09-security/08-ipsec.md).

Two things can happen:

- **Inner fragmentation:** the packet is cut before it is wrapped. Each piece becomes a full tunnel packet, and the final host reassembles.
- **Outer fragmentation:** the tunnel packet is cut after wrapping. The tunnel endpoint is now the destination, so it must reassemble. This uses the router's CPU and memory, and it is usually the worse choice.

Common fixes:

```text
! Cisco IOS, tunnel interface
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

`ip mtu 1400` sets the largest IP packet on the tunnel. `ip tcp adjust-mss 1360` rewrites the MSS value in TCP SYN packets so TCP stays small (1400 - 20 IP - 20 TCP = 1360). The MSS logic belongs to [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### IPv4 versus IPv6 on these topics

| Topic | IPv4 | [IPv6](11-ipv6-basics.md) (RFC 8200) |
|---|---|---|
| Header size | 20-60 bytes, variable | 40 bytes, fixed |
| Who may fragment | Source and routers | Source only |
| Fragment info | In the main header | In an optional Fragment extension header |
| Too big at a router | Fragment, or drop if DF = 1 | Always drop, send ICMPv6 Packet Too Big |
| Header checksum | Yes | Removed |
| Loop limit | TTL | Hop Limit |
| Minimum link MTU | 68 bytes | 1280 bytes |

IPv6 keeps the idea of fragments but removes the router's job. This is a direct lesson from IPv4's problems.

### Useful capture filters

The Flags and Offset block is at byte 6 of the header. These tcpdump filters read it directly:

```bash
# Linux: all fragments (MF set, or offset not zero)
sudo tcpdump -n -i eth0 'ip[6:2] & 0x3fff != 0'

# Non-first fragments only (offset not zero)
sudo tcpdump -n -i eth0 'ip[6:2] & 0x1fff != 0'

# Packets with DF set
sudo tcpdump -n -i eth0 'ip[6] & 0x40 != 0'

# Packets marked EF (ignore the ECN bits)
sudo tcpdump -n -i eth0 '(ip[1] & 0xfc) == 0xb8'

# ICMP Fragmentation Needed
sudo tcpdump -n -i eth0 'icmp[0] == 3 and icmp[1] == 4'
```

`0x3fff` covers MF (`0x2000`) and all 13 offset bits (`0x1fff`). It leaves out DF, so normal TCP traffic with DF set does not match. More on the tool in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

Linux settings worth knowing:

```bash
# Linux: default TTL for packets this host sends
sysctl net.ipv4.ip_default_ttl

# Seconds to wait for all fragments
sysctl net.ipv4.ipfrag_time

# Memory limits for reassembly (defaults depend on kernel version)
sysctl net.ipv4.ipfrag_high_thresh net.ipv4.ipfrag_low_thresh
```

## Common Mistakes

- **"Fragment Offset counts bytes."** The field counts 8 byte units. A field value of 185 means byte 1480. Tools such as tcpdump print the byte value, which adds to the confusion.
- **"Each fragment holds 1500 bytes of data."** MTU includes the IP header. On a 1500 MTU link, each fragment holds at most 1480 bytes of data.
- **Forgetting to round down.** With MTU 1000, the data per fragment is 976, not 980.
- **Computing offsets from the total length.** Offsets count data bytes only. Never include the 20 byte headers.
- **"The last fragment must be a multiple of 8."** Only the non-last fragments must be. The last can be any size.
- **"Routers reassemble fragments."** Normal forwarding routers do not. Only the destination does. Firewalls may check pieces with virtual reassembly, but that is a separate feature.
- **"IP will resend a lost fragment."** IP never resends. The whole packet is lost, and a higher layer must recover it.
- **"TTL is measured in seconds."** That was the original idea. In practice it is a hop count.
- **"The checksum protects the data."** It covers the header only.
- **"The header never changes on the way."** TTL and checksum change at every router. DSCP may be rewritten, NAT changes addresses, and fragmenting changes several fields.
- **"Protocol 6 and port 6 are related."** The Protocol field and ports are different things. TCP is protocol 6. Port numbers live in the TCP or UDP header.
- **"DF means the packet is important."** DF only says "do not cut this". If the packet does not fit, it is dropped.
- **"A capture shows a 9000 byte packet, so my MTU must be 9000."** Modern NICs use offload features, so a capture on the host can show large packets before the NIC cuts them into segments. For the same reason, a capture on the sender may show checksum errors that do not exist on the wire.
- **"ping -s 1500 tests a 1500 byte packet."** On Linux and Windows, add 28 bytes for headers. That command makes a 1528 byte packet.

## Interview Questions

**Q: What is the size of the IPv4 header, and how does a receiver know it?**

A: It is 20 bytes without options and up to 60 bytes with them. The IHL field gives the length in 32 bit words, so a value of 5 means 20 bytes and 15 means 60 bytes.

**Q: Which fields change as a packet crosses a router?**

A: TTL goes down by 1, so the header checksum must also change. If the router fragments the packet, Total Length, MF, and Fragment Offset change as well. DSCP may be rewritten by policy, and NAT changes addresses and ports.

**Q: Fragment a 4000 byte packet with a 20 byte header over a 1500 MTU link.**

A: The data is 3980 bytes, and each fragment can hold 1480. That gives three fragments with 1480, 1480, and 1020 bytes of data. The offsets are 0, 185, and 370. Total Lengths are 1500, 1500, and 1040. MF is 1, 1, 0.

**Q: Why is Fragment Offset measured in 8 byte units?**

A: The offset field has only 13 bits, because 3 bits of that 16 bit block are used for flags. Thirteen bits give 8192 values. Multiplying by 8 lets it reach the whole 64 KB packet range. As a result, all fragments except the last must carry a multiple of 8 data bytes.

**Q: What does the DF bit do? What happens if a packet with DF set is too big?**

A: DF tells devices not to fragment the packet. If it is too big for the next link, the router drops it and sends ICMP type 3, code 4, with the next link's MTU. Path MTU Discovery depends on this message. If a firewall blocks it, you get a black hole, where small packets work and large ones vanish.

**Q: How does a receiver know which fragments belong together, and when it has them all?**

A: It groups them by source, destination, protocol, and Identification. The fragment with MF = 0 reveals the total size: offset times 8 plus its data length. When no gaps remain, the packet is complete. If the timer ends first, it drops everything and may send ICMP Time Exceeded, code 1.

**Q: Why is reassembly done at the destination and not at routers?**

A: Fragments can take different paths, so a router may not see them all. It would also need memory and CPU for every flow, and a later link might need the packet cut again anyway. Keeping that work at the end host keeps routers simple.

**Q: Why is IP fragmentation considered bad?**

A: Losing one fragment loses the whole packet, so packet loss gets worse. Later fragments have no ports, which breaks firewalls, PAT, and load balancing. It also adds router CPU work, uses memory at the receiver, enables attacks, and can cause wrong joins when IDs wrap. RFC 8900 advises against relying on it.

**Q: What is TTL for, and what happens when it reaches zero?**

A: TTL stops packets from looping forever. Each router lowers it by 1. When it hits 0, the router drops the packet and sends ICMP Time Exceeded, type 11, code 0. Traceroute uses this behaviour to discover each hop.

**Q: Why does the IPv4 checksum change at each hop, and why did IPv6 remove it?**

A: It covers the header, and TTL changes at each hop, so the value must be updated each time. IPv6 removed it because link and transport layers already detect errors, and removing it saves per-hop work.

**Q: What is the difference between DSCP and ECN?**

A: Both live in the old ToS byte. DSCP is 6 bits and labels the packet's class, so routers can give it a treatment such as a priority queue. ECN is 2 bits and lets a router signal congestion by marking a packet instead of dropping it.

**Q: How does fragmentation differ in IPv6?**

A: Routers never fragment in IPv6. A packet that is too big is dropped, and the router sends ICMPv6 Packet Too Big. Only the source can fragment, using a Fragment extension header. The minimum link MTU is 1280 bytes, and there is no DF bit since the behaviour is always like DF set.

## Quick Recap

- The IPv4 header is 20-60 bytes. IHL gives its length in 4 byte words, and Total Length gives the whole packet size, up to 65535 bytes.
- TTL is a hop limit, and Protocol names the payload (TCP 6, UDP 17, ICMP 1). The checksum covers only the header and changes at every hop.
- The old ToS byte is now 6 bits of DSCP plus 2 bits of ECN.
- DF = 1 means drop and send ICMP type 3, code 4 if too big. MF = 1 means more pieces follow.
- Max data per fragment is `MTU - header`, rounded down to a multiple of 8. Offset equals the byte position divided by 8.
- Only the destination reassembles. It groups by source, destination, protocol, and ID, and it uses a timer.
- Avoid fragmentation: one lost piece loses the packet, and later pieces have no ports. TCP uses DF with PMTUD and MSS, and IPv6 removed router fragmentation.
