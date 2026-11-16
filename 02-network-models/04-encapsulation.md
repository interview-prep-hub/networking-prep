# Encapsulation and Decapsulation

**Level:** Beginner | **Prerequisites:** [The OSI Model](02-osi-model.md), [The TCP/IP Model](03-tcp-ip-model.md)

## In One Line

Encapsulation means each layer wraps the data from the layer above with its own control information on the way down the stack. Decapsulation means each layer removes that wrapper on the way up.

## Why It Matters

Your application wants to send a few bytes of text. The network needs much more than that text. It needs to know which computer, which program, and which device is the next stop. It also needs a way to detect damage.

One layer could put all of this in one giant header. Then every change would touch everything. A new Wi-Fi standard would force changes in your web browser. Layers avoid this. Each layer adds only its own information and reads only its own information. So TCP does not care if the cable is copper, fibre, or radio.

Interviewers ask about this topic for three reasons:

- It shows you understand how the layers really work together, not only their names.
- It is the base for troubleshooting. Every line in a packet capture is a header.
- It explains real numbers, such as why 1460 is a common TCP data size and why a 1 Gbps link gives about 940 Mbps in a speed test.

## Simple Explanation

Imagine you send a letter to a person in a large office building in another city.

1. You write the letter. This is the **data**.
2. You put it in an envelope marked "Room 80". This tells the building which room gets it. This is like the **TCP header**.
3. You put that envelope in a bigger one with the street address. This tells the post office which building. This is like the **IP header**.
4. The post office puts it in a van with a label: "Next stop: city depot". This is like the **Ethernet header**.
5. The van drives on the road. This is like the **bits** on the cable.

Wrapping is **encapsulation**. At the far end, the steps run in reverse. The van is unloaded, the street envelope is opened, the mailroom reads "Room 80", and the person reads the letter. Unwrapping is **decapsulation**.

Each depot on the way puts the parcel in a new van with a new "next stop" label. The street address inside does not change. This is the most important idea for later chapters.

**Where the analogy stops being true:**

- There are no real envelopes. A packet is one flat row of bytes. A "header" is only the bytes that come first.
- Paper envelopes add almost no weight. Network headers can be bigger than the data they carry, as you will see below.
- The analogy has only wrappers at the front. Some layers also add a piece at the end.

## How It Works

### Headers, trailers and payload

Three terms describe the parts of any wrapped unit:

- **Header:** control information placed in front of the data. Example: source and destination addresses.
- **Trailer:** control information placed after the data. Example: an error check.
- **Payload:** the content being carried. A layer does not read its payload. It only moves it.

One layer's whole unit is the next lower layer's payload. The IP layer sees the TCP header and the data together as one opaque payload.

Most protocols use only a header. TCP, UDP, IPv4 and IPv6 have no trailer. Ethernet has both, and so does Wi-Fi. A header goes first because a receiver must read addresses and types before it can process anything else. Ethernet puts its error check last because the sending hardware can calculate it while the bits go out, then add the result at the end.

### PDU names

A **PDU** (Protocol Data Unit) is the name for "the unit of data at this layer". Each layer has its own name, and interviewers expect you to know them.

| TCP/IP layer | OSI layer number | PDU name | Added here |
|---|---|---|---|
| Application | 7, 6, 5 | Data (or message) | Application content, such as an HTTP request |
| Transport | 4 | Segment (TCP) or datagram (UDP) | TCP or UDP header |
| Network (Internet) | 3 | Packet (or IP datagram) | IP header |
| Data link | 2 | Frame | Ethernet header and trailer |
| Physical | 1 | Bits | Signal on the medium |

A common memory trick, from Layer 7 down: **D**o **S**ergeants **P**ay **F**or **B**eer (Data, Segment, Packet, Frame, Bits).

**Fuller truth:** the word "datagram" has two uses. RFC 791, the IPv4 standard, calls its unit a datagram. People also say "UDP datagram" at Layer 4. In daily speech, "packet" is often used loosely for any unit. In an interview, use the exact names from the table.

### Going down the stack (encapsulation)

The numbers below are in bytes and come from the worked example later.

```text
Sender: each layer adds its part. Numbers are bytes.

Application                                  +---------+
(data, 74)                                   | HTTP 74 |
                                             +---------+
Transport                           +--------+---------+
(segment, 94)                       | TCP 20 | HTTP 74 |
                                    +--------+---------+
Network                    +--------+--------+---------+
(packet, 114)              | IP  20 | TCP 20 | HTTP 74 |
                           +--------+--------+---------+
Data link         +--------+--------+--------+---------+--------+
(frame, 132)      | Eth 14 | IP  20 | TCP 20 | HTTP 74 | FCS  4 |
                  +--------+--------+--------+---------+--------+
Physical          1 0 1 1 0 0 1 0 ... (bits on the cable)
```

1. **Application.** The program creates data and hands it to the operating system through a socket (the program's connection point to the network).
2. **Transport.** TCP or UDP adds a header with source and destination **ports**. A port is a number that identifies a program on a computer. See [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md).
3. **Network.** IP adds a header with source and destination **IP addresses**. An IP address identifies a device across networks.
4. **Data link.** Ethernet adds a header with source and destination **MAC addresses**, and a trailer. A MAC address identifies a network interface on the local link. The destination MAC is the *next hop*, not always the final server. If the server is on another network, it is the default gateway. The sender finds that MAC using [ARP](../03-ethernet-and-switching/05-arp.md).
5. **Physical.** The **NIC** (network interface card, the hardware that connects a device to the network) sends the frame as electrical, light, or radio signals.

### Going up the stack (decapsulation)

The receiver reverses the process. Each layer checks its own header, removes it, and passes the payload up. If a check fails, the data stops there.

| Layer | What it checks | If the check fails |
|---|---|---|
| Physical and data link | FCS is correct. Destination MAC is ours, or broadcast, or a multicast group we joined | Frame is dropped silently |
| Network | Header is valid. Destination IP is ours | A host drops it. A router forwards it instead (that is a router's job) |
| Transport | Checksum is correct. A program is listening on this port | Bad checksum: dropped, and TCP will resend. Closed TCP port: RST sent. Closed UDP port: ICMP "port unreachable" usually sent |
| Application | The request makes sense | Application error, such as HTTP 400 |

### How each layer knows what comes next

A byte stream has no labels by itself. So how does the receiver know the payload of a frame is IPv4 and not IPv6? Each header has a field that names the protocol inside. This is called **demultiplexing**: one stream arrives, and it is sorted to the right handler.

| Header | Field | Common values |
|---|---|---|
| Ethernet | EtherType | `0x0800` IPv4, `0x86DD` IPv6, `0x0806` ARP |
| IPv4 | Protocol | 1 ICMP, 6 TCP, 17 UDP |
| IPv6 | Next Header | 6 TCP, 17 UDP, 58 ICMPv6 |
| TCP or UDP | Destination port | 80 HTTP, 443 HTTPS, 53 DNS |

The reverse, on the sender, is **multiplexing**: many programs share one network connection. Without these fields, encapsulation could not be undone.

### Which device reads which layer

Not every device opens every wrapper. A device only goes as high as it needs to.

```text
   PC           Switch          Router         Server
+------+                                      +------+
| App  |                                      | App  |
+------+                                      +------+
| TCP  |                                      | TCP  |
+------+                       +------+       +------+
| IP   |                       |  IP  |       | IP   |
+------+      +------+      +--+------+--+    +------+
| Eth  |      | Eth  |      | Eth    Eth |    | Eth  |
+------+      +------+      +------------+    +------+
| Bits |======| Bits |======| Bits  Bits |====| Bits |
+------+      +------+      +------------+    +------+
```

- A **switch** reads the frame header and forwards the frame. It does not rebuild it.
- A **router** removes the frame, reads the IP header, chooses the next hop, and builds a brand new frame.
- A **host** (an end device such as a PC or server) goes all the way up.

## Example

### The setup

A Windows PC requests a web page from a server. DNS has already given the server's address, and the TCP connection is already open. See [TCP handshake](../06-transport-layer/03-tcp-handshake-and-header.md). The PC is about to send its HTTP request.

```text
 +----+      +-----+      +--------+      +----+      +--------+
 | PC |------| SW1 |------|   R1   |------| R2 |------| Server |
 +----+      +-----+      | (NAT)  |      +----+      +--------+
                          +--------+
   192.168.1.50      LAN 192.168.1.1
                     WAN 198.51.100.2
                              R2 west 198.51.100.1
                              R2 east 203.0.113.1
                                         Server 203.0.113.10
```

| Interface | MAC address |
|---|---|
| PC | `02:00:00:00:00:50` |
| R1 LAN | `02:00:00:00:01:01` |
| R1 WAN | `02:00:00:00:01:02` |
| R2 west | `02:00:00:00:02:01` |
| R2 east | `02:00:00:00:02:02` |
| Server | `02:00:00:00:03:10` |

### Down the stack on the PC

**Step 1, application data.** The HTTP request is plain text. Each line ends with two bytes: carriage return and line feed (`\r\n`).

```text
Line                         Characters   With \r\n
GET / HTTP/1.1                       14          16
Host: example.com                    17          19
User-Agent: curl/8.5.0               22          24
Accept: */*                          11          13
(blank line)                          0           2
                                      Total:     74 bytes
```

**Step 2, TCP segment.** TCP adds a 20-byte header. It holds source port 51514, destination port 80, sequence and acknowledgement numbers, flags, window, and a checksum. A **checksum** is a small number calculated from the bytes, so the receiver can detect damage. The result is 74 + 20 = **94 bytes**.

**Step 3, IP packet.** IPv4 adds a 20-byte header. It holds source 192.168.1.50, destination 203.0.113.10, Protocol 6 (TCP), and Total Length 114. It also holds **TTL** (Time To Live), a counter that each router lowers by one, so a lost packet cannot loop forever. Windows commonly starts at 128, and Linux commonly at 64. The result is 94 + 20 = **114 bytes**.

**Step 4, Ethernet frame.** The server is on another network, so the destination MAC is R1's LAN address. Ethernet adds a 14-byte header (6 destination MAC, 6 source MAC, 2 EtherType) and a 4-byte trailer called the **FCS** (Frame Check Sequence, a CRC-32 error check). The result is 14 + 114 + 4 = **132 bytes**.

**Step 5, bits.** Before the frame, the NIC sends 8 bytes of preamble and start delimiter, so the receiver can lock on to the signal. After the frame, it stays quiet for a 12-byte gap. So the frame uses 132 + 20 = **152 bytes of line time**, or 1216 bits. At 1 Gbps this takes 1.216 microseconds.

Here is the same frame as a byte map:

```text
Offset  Size  Field                  Value
------  ----  ---------------------  ---------------------
0       6     Destination MAC        02:00:00:00:01:01
6       6     Source MAC             02:00:00:00:00:50
12      2     EtherType              0x0800 (IPv4)
14      20    IPv4 header            first byte 0x45
                                     Total Length 114
                                     TTL 128, Protocol 6
                                     192.168.1.50 ->
                                     203.0.113.10
34      20    TCP header             51514 -> 80
                                     Flags PSH, ACK
54      74    HTTP data              "GET / HTTP/1.1..."
128     4     FCS                    CRC-32 of offset 0-127
```

In the first IP byte, `0x45`, the 4 means IPv4, and the 5 means the header is 5 groups of 4 bytes, so 20 bytes.

TCP has no field for its own data length. The receiver calculates it: 114 (IP total) - 20 (IP header) - 20 (TCP header) = 74 bytes.

### The overhead in numbers

**Overhead** means the bytes that are not your data. The table compares three real cases on standard Ethernet, using TCP with no options.

| Item | Short request | Full-size segment | Empty ACK |
|---|---|---|---|
| Application data | 74 | 1460 | 0 |
| + TCP header (20) | 94 | 1480 | 20 |
| + IPv4 header (20) | 114 | 1500 | 40 |
| + Ethernet header and FCS (18) | 132 | 1518 | 58 |
| + Padding to reach 64 | 0 | 0 | 6, total 64 |
| + Preamble and gap (20) | 152 | 1538 | 84 |
| Data share of line time | 48.7% | 94.9% | 0% |

Three lessons come from this table:

- **Small messages are expensive.** For the 74-byte request, more than half of the line time is overhead.
- **1460 comes from subtraction.** The usual **MTU** (Maximum Transmission Unit, the largest IP packet a link carries) on Ethernet is 1500. Take off 20 for IP and 20 for TCP, and you get 1460. This value is called the **MSS** (Maximum Segment Size). See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).
- **Ethernet has a minimum size.** A frame must be at least 64 bytes, so a short payload gets **padding** (filler bytes). The receiver uses the IP Total Length to know where the real packet ends, and it ignores the padding. See [Ethernet Frames](../03-ethernet-and-switching/02-ethernet-frames.md).

### What each hop changes

Now follow the frame across the network. The HTTP data stays the same, but the outer wrapper does not.

| Link | Source MAC | Destination MAC | Source IP and port | Destination IP and port | TTL |
|---|---|---|---|---|---|
| PC to R1 | `...00:50` | `...01:01` | 192.168.1.50:51514 | 203.0.113.10:80 | 128 |
| R1 to R2 | `...01:02` | `...02:01` | 198.51.100.2:40001 | 203.0.113.10:80 | 127 |
| R2 to server | `...02:02` | `...03:10` | 198.51.100.2:40001 | 203.0.113.10:80 | 126 |

**SW1** does nothing to the frame. It reads the destination MAC and sends the frame out of the right port. The frame leaves exactly as it came in.

**R1** does this:

1. Checks the FCS, sees that the destination MAC is its own, and removes the Ethernet header and FCS.
2. Reads the destination IP and looks it up in its routing table.
3. Lowers TTL from 128 to 127.
4. Applies **NAT** (Network Address Translation, which swaps a private address for a public one). The source becomes 198.51.100.2:40001. The port does not always change, but it can. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).
5. Recalculates the IPv4 header checksum, because TTL and the address changed. It also updates the TCP checksum, because that checksum covers the IP addresses and the port.
6. Builds a new Ethernet frame with new MACs and a new FCS, then sends it.

**R2** is a normal router, so it does steps 1, 2, 3, 5 (IP checksum only) and 6. It does not touch the IP addresses or TCP header.

Here is the summary:

| Field | Switch | Router | Router with NAT |
|---|---|---|---|
| Source and destination MAC | Same | **New** | **New** |
| FCS | Same (new only if a VLAN tag is added or removed) | **New** | **New** |
| TTL | Same | **Down by 1** | **Down by 1** |
| IPv4 header checksum | Same | **New** | **New** |
| Source IP and port | Same | Same | **Changed** |
| Destination IP and port | Same | Same | Same (outbound) |
| TCP checksum | Same | Same | **Updated** |
| Application data | Same | Same | Same |

The short version for an interview: **MAC addresses change at every router hop. IP addresses stay the same, unless NAT is in the path.**

**Fuller truth:** other things can change too. A router may fragment an IPv4 packet if the next link is too small and the packet allows it. See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md). The next link might not be Ethernet at all, so the Layer 2 wrapper can be a different type. For example, a Wi-Fi access point converts between 802.11 frames and Ethernet frames. A proxy or Layer 7 load balancer goes further: it ends the TCP connection and opens a new one. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

### Up the stack on the server

1. The server NIC receives 132 bytes. The FCS is good, and the destination MAC `...03:10` is its own. Ethernet is removed, and EtherType `0x0800` says IPv4. **114 bytes** remain.
2. IP checks the header checksum. The destination 203.0.113.10 is its own. TTL arrived as 126. IP is removed, and Protocol 6 says TCP. **94 bytes** remain.
3. TCP checks its checksum and finds the connection using four values: 198.51.100.2, 40001, 203.0.113.10, 80. TCP is removed. **74 bytes** remain.
4. The web server reads the 74 bytes and sends its response. That response goes through the same process, in reverse. R1 uses its NAT table to send it back to 192.168.1.50:51514.

### See it yourself

**Wireshark (Windows or Linux).** The packet details pane lists the layers in order, from outside to inside. Each line is one wrapper.

```text
Frame 4: 128 bytes on wire (1024 bits), 128 bytes captured
Ethernet II, Src: 02:00:00:00:00:50, Dst: 02:00:00:00:01:01
Internet Protocol Version 4, Src: 192.168.1.50, Dst: 203.0.113.10
Transmission Control Protocol, Src Port: 51514, Dst Port: 80, Len: 74
Hypertext Transfer Protocol
```

It shows 128, not 132, because most capture tools never see the 4-byte FCS. The NIC adds it on send and removes it on receive. The capture also misses the preamble and gap.

**tcpdump (Linux).** Here is the same request sent from Linux. The line is wrapped to fit the page.

```bash
sudo tcpdump -i eth0 -n -e -c 1 \
  'tcp dst port 80 and tcp[tcpflags] & tcp-push != 0'
```

```text
10:15:02.418233 02:00:00:00:00:50 > 02:00:00:00:01:01,
  ethertype IPv4 (0x0800), length 140:
  192.168.1.50.51514 > 203.0.113.10.80: Flags [P.], seq 1:75,
  ack 1, win 502, options [nop,nop,TS val 3012345678 ecr 1234567890],
  length 74: HTTP: GET / HTTP/1.1
```

The two `length` values mean different things. The first, 140, is the frame without FCS. The second, 74, is the TCP payload. It is 140 and not 128 because Linux normally adds a 12-byte TCP timestamp option, so its TCP header is 32 bytes: 14 + 20 + 32 + 74 = 140. Windows has TCP timestamps off by default.

**Ping can prove the header sizes.** An ICMP echo has an 8-byte header, and IPv4 adds 20. So 1472 bytes of ping data make exactly a 1500-byte packet. These commands also set "do not fragment".

Linux:

```bash
ping -c 1 -s 1472 -M do 192.168.1.1
```

```text
PING 192.168.1.1 (192.168.1.1) 1472(1500) bytes of data.
1480 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.612 ms
```

Windows:

```powershell
ping -n 1 -l 1472 -f 192.168.1.1
```

```text
Pinging 192.168.1.1 with 1472 bytes of data:
Reply from 192.168.1.1: bytes=1472 time=1ms TTL=64
```

Try 1473 and the packet no longer fits. Linux reports `message too long, mtu=1500`, and Windows reports `Packet needs to be fragmented but DF set.`

## Going Deeper

**SDU and PCI.** The OSI model has formal names for the parts. The **SDU** (Service Data Unit) is what a layer receives from above. The **PCI** (Protocol Control Information) is the header or trailer that it adds. So PDU = PCI + SDU. The PDU of one layer becomes the SDU of the layer below.

**Tunnels are encapsulation again.** A **tunnel** puts a full packet inside another packet, so it can cross a network that would not carry it directly. Then the inner IP packet becomes a payload. This adds overhead, and that reduces the room for real data. See [VPNs](../09-security/07-vpn.md) and [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

| Extra wrapper | Bytes added |
|---|---|
| 802.1Q VLAN tag | 4 |
| PPPoE (common on DSL and some fibre) | 8 |
| GRE over IPv4, basic header | 24 |
| VXLAN over IPv4 | 50 |
| TLS 1.3 record with AES-GCM | 22 per record |

The PPPoE row explains the MTU of 1492 that you see on many home connections. The VXLAN row counts the outer Ethernet (14), IPv4 (20), UDP (8), and VXLAN (8) headers.

**IPv6 costs 20 more bytes.** Its fixed header is 40 bytes (RFC 8200). The same 1500-byte MTU gives an MSS of 1440, not 1460. Routers lower the Hop Limit, but there is no header checksum to recalculate, and routers do not fragment.

**A message is not always one segment.** TCP is a byte stream. A 10 MB file becomes thousands of segments, each with its own headers. Also, two small writes may share one segment. **Segmentation** (cutting data to fit) and encapsulation (adding headers) are different ideas, though both happen at the same time.

**Captures can look wrong because of offload.** Modern NICs take over some work from the CPU. With checksum offload, the NIC fills in the checksum, so a capture on the sender may show "bad checksum" for a healthy packet. With TCP segmentation offload, the operating system hands the NIC one big block to cut up, so a sender-side capture may show "packets" much larger than 1500 bytes. Neither is a fault. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

**Line coding is extra.** The physical layer also encodes bits into signals. For example, 1000BASE-X uses 8b/10b coding. The 1 Gbps rate is already measured after this coding, so it does not change the math above.

## Common Mistakes

- **"The router only reads the IP header and passes the frame along."** No. The router throws the old frame away and builds a new one.
- **"The destination MAC is the server's MAC."** Only when the server is on the same network. Otherwise it is the MAC of the next router.
- **"A switch changes the frame."** A normal switch forwards it unchanged. The exception is adding or removing a VLAN tag on a [trunk](../03-ethernet-and-switching/07-trunking-8021q.md), which also needs a new FCS.
- **"IP addresses never change."** NAT changes them. The correct rule is that routing alone does not change them.
- **"Every layer has a trailer."** Most layers use only a header. In this stack, only Ethernet added one.
- **"Payload always means user data."** Payload is relative. IP's payload includes the TCP header.
- **"The frame is 1500 bytes."** 1500 is the IP packet limit. The Ethernet frame is 1518 with header and FCS, or 1522 with a VLAN tag.
- **"The MTU includes the Ethernet header."** In normal IP networking, it does not. Some vendors show a different number on their interfaces, so check how your platform counts it.
- **"Wireshark is missing bytes, so something is broken."** It normally does not show the FCS, preamble, or gap.
- **"A segment and a datagram are the same."** Segment is for TCP. Datagram is for UDP and, in formal language, IP.

## Interview Questions

**Q: What is encapsulation?**

A: It is the process where each layer adds its own header, and sometimes a trailer, to the data from the layer above. The receiver removes them in reverse order, which is decapsulation. This keeps layers independent, because each one reads only its own information.

**Q: What is the PDU at each layer?**

A: Data at the application layer, segment for TCP or datagram for UDP at the transport layer, packet at the network layer, frame at the data link layer, and bits at the physical layer.

**Q: What is the difference between a header and a trailer? Name a protocol with a trailer.**

A: A header goes before the data, and a trailer goes after it. Ethernet has a trailer: the 4-byte FCS, a CRC-32 used to detect damaged frames. TCP, UDP and IP have only headers.

**Q: What changes when a packet passes through a router, and what stays the same?**

A: The router removes the old frame and builds a new one, so the source and destination MAC addresses and the FCS change. It lowers the TTL by one, and in IPv4 it recalculates the header checksum. The IP addresses, ports and data stay the same, unless NAT or fragmentation is involved.

**Q: Does a switch change the frame?**

A: Normally no. It reads the destination MAC and forwards the frame unchanged, without touching the IP header. The exception is adding or removing an 802.1Q VLAN tag, which also means a new FCS.

**Q: How does the receiver know which protocol is inside each header?**

A: Each header has a field that names the next protocol. Ethernet uses EtherType, such as 0x0800 for IPv4. IPv4 uses the Protocol field, such as 6 for TCP. TCP and UDP use port numbers to reach the right program.

**Q: Why is the common TCP MSS 1460?**

A: Standard Ethernet has an MTU of 1500 bytes for the IP packet. Subtract 20 for the IPv4 header and 20 for the TCP header, and 1460 bytes remain for data. With IPv6 it is 1440, because the IPv6 header is 40 bytes.

**Q: Why does a clean 1 Gbps link show about 940 Mbps in a speed test?**

A: Headers and framing use part of the line. A full-size frame takes 1538 bytes of line time, including preamble and gap, but carries only 1460 bytes of data, or 1448 with TCP timestamps. That gives about 949 or 941 Mbps of real data.

**Q: A capture shows a 128-byte frame, but you calculated 132. Why?**

A: Capture tools normally do not see the 4-byte FCS, because the NIC adds it on send and removes it on receive. So the frame is really 132 bytes, and the capture shows 128.

**Q: What happens if a frame arrives with a bad FCS?**

A: The receiver drops it silently, and no error message goes back to the sender. If it was TCP, the sender sees the data was not acknowledged and sends it again. If it was UDP, the data is lost unless the application handles that itself.

**Q: What is a tunnel, in terms of encapsulation?**

A: A tunnel puts a whole packet inside another packet as its payload. VXLAN, GRE and IPsec all work this way. It lets traffic cross a network that could not carry it directly, but the extra headers reduce the room for data, so MTU must be planned.

## Quick Recap

- Encapsulation adds headers (and sometimes trailers) going down. Decapsulation removes them going up.
- PDU names: data, segment or datagram, packet, frame, bits.
- Typical sizes: TCP 20, UDP 8, IPv4 20, IPv6 40, Ethernet 14 + 4. Line overhead adds 20 more.
- Our 74-byte HTTP request became a 94-byte segment, a 114-byte packet, and a 132-byte frame.
- Fields such as EtherType, Protocol, and port tell each layer what comes next.
- A switch forwards the frame unchanged. A router builds a new frame and lowers TTL.
- MAC addresses change at each router hop. IP addresses stay the same unless NAT is in the path.
