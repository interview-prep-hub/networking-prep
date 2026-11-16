# MTU, MSS and Path MTU Discovery

**Level:** Advanced | **Prerequisites:** [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md), [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md), [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md)

## In One Line

MTU is the largest IP packet a link can carry. MSS is the largest block of TCP data that fits inside that packet. Path MTU Discovery is how a sender learns the smallest MTU on the whole path.

## Why It Matters

Every link has a size limit. A packet bigger than the limit must be cut into pieces or dropped. Most modern traffic asks routers not to cut it, so it gets dropped.

This creates one of the most confusing faults in networking:

- Ping works.
- The TCP connection opens.
- Small web pages load.
- Large pages, file copies and some logins hang forever.

The small packets fit. The large ones vanish. Nothing shows an error, and no log line says "packet too big".

This often appears right after someone adds a VPN or a tunnel, because a tunnel makes the usable packet size smaller. Interviewers love this topic because it tests four things together: maths, TCP, ICMP and real troubleshooting. The classic question is: "The VPN is up and ping works, but file transfers hang. What do you check?"

## Simple Explanation

Think of a delivery truck on a long road.

- The road has bridges. Each bridge has a height limit. That limit is the **MTU**.
- The truck has a cab and a frame that take up space. The space left for your boxes is the **MSS**.
- The lowest bridge on the whole road decides the tallest truck that can finish the trip. That lowest limit is the **path MTU**.
- The truck has a sign: "Do not split this load." That sign is the **Don't Fragment** flag.
- A guard at a low bridge stops a tall truck, sends it back, and phones the sender: "The limit here is 3.5 metres." That phone call is an **ICMP** message.
- If the guard's phone line is cut, the sender never hears anything. Trucks leave and never arrive. That is an **ICMP black hole**.

Where the analogy stops being true: a real router does not send the packet back. It destroys the packet and sends only a short message. The sender can still resend the data because TCP keeps a copy until the data is acknowledged.

## How It Works

### Basic terms

- **Byte:** 8 bits. All sizes in this chapter are in bytes.
- **Header:** control information added in front of data, like the address on an envelope.
- **Payload:** the data carried behind a header.
- **Frame:** the Layer 2 unit, such as an Ethernet frame.
- **Packet:** the Layer 3 unit, such as an IP packet.
- **Segment:** the TCP unit, which is a TCP header plus some data.

Each layer wraps the layer above. This wrapping is called encapsulation. See [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md).

### MTU: the packet limit of one link

**MTU** stands for Maximum Transmission Unit. It is the largest IP packet, including the IP header, that one link can carry without fragmenting it.

Standard Ethernet has an MTU of 1500 bytes. The Ethernet header and trailer are not counted.

```text
Size on an Ethernet link with a full-size packet

+---------+----------------------------------------------+-----+
| Eth hdr | IP packet (the part the MTU limits)          | FCS |
| 14      | 1500                                         | 4   |
+---------+----------------------------------------------+-----+
|<------------------ frame = 1518 bytes ---------------------->|
```

- **FCS** is the Frame Check Sequence, a 4-byte error check at the end of the frame.
- With an 802.1Q VLAN tag, the frame is 1522 bytes. The IP MTU is still 1500.
- See [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md) for the frame details.

Minimum limits exist too:

| Rule | Value | Meaning |
|---|---|---|
| IPv4 smallest link MTU | 68 | Every IPv4 link must carry this |
| IPv4 hosts must accept | 576 | A host must accept a datagram this large |
| IPv6 smallest link MTU | 1280 | Every IPv6 link must carry this (RFC 8200) |

### MSS: the data limit of one TCP segment

**MSS** stands for Maximum Segment Size. It is the largest amount of TCP data in one segment. The name misleads: MSS does **not** include the TCP header or the IP header.

The basic maths:

```text
MSS = MTU - IP header - TCP header

IPv4: 1500 - 20 - 20 = 1460
IPv6: 1500 - 40 - 20 = 1440
```

```text
|<------------------------ MTU = 1500 ------------------------>|
+------------+-------------+-----------------------------------+
| IPv4 hdr   | TCP hdr     | TCP data                          |
| 20         | 20          | 1460                              |
+------------+-------------+-----------------------------------+
                           |<--------- MSS = 1460 ------------>|
```

Fuller truth: 20 bytes is the smallest header size. Headers can carry options and grow. Most TCP connections on Linux use the Timestamps option, which takes 12 bytes in every segment. MSS stays at 1460 in the handshake, but each segment then carries only 1448 bytes of data.

```text
1500 - 20 (IP) - 20 (TCP) - 12 (timestamps) = 1448 bytes of data
```

That is why you often see `length 1448` in captures, not 1460.

### How MSS is "negotiated"

The word "negotiated" is common, but it is not quite right. There is no bargaining. Each side simply announces the most it is willing to receive.

1. The client puts an MSS option in its SYN. SYN is the first packet of the [three-way handshake](03-tcp-handshake-and-header.md).
2. The server puts its own MSS option in its SYN-ACK.
3. Each sender limits data to the value the **other** side announced.
4. Each sender also limits itself by its own outgoing MTU.

The MSS option is 4 bytes: Kind 2, Length 4, and a 2-byte value. It appears only in SYN and SYN-ACK.

```text
+----------+----------+----------------------+
| Kind = 2 | Len = 4  | MSS value (16 bits)  |
+----------+----------+----------------------+
```

So the rule for each direction is:

```text
send size = min(MSS the peer announced, my own path MTU - 40)
```

A real example. The client is on Ethernet (MTU 1500). The server sits behind PPPoE (MTU 1492, explained below).

```text
Client announces: MSS 1460   (1500 - 40)
Server announces: MSS 1452   (1492 - 40)

Client to server: min(1452, 1460) = 1452
Server to client: min(1460, 1452) = 1452
```

The two announcements can differ, and the two directions are separate. The result looks like "use the smaller one" here because each sender is also held back by its own link. If you see only one side's SYN, you do not know the full story.

If a SYN has no MSS option, the rules in RFC 9293 say to assume:

- 536 for IPv4 (576 - 40)
- 1220 for IPv6 (1280 - 60)

### What MSS cannot see

MSS only reflects the links at the two ends. It knows nothing about the middle.

```text
Host A           R1                R2              Host B
MTU 1500 ------ MTU 1500 ===1400=== MTU 1500 ------ MTU 1500
MSS 1460                                           MSS 1460
```

Both hosts announce 1460. Both think 1500-byte packets are fine. The link in the middle only carries 1400. The handshake uses tiny packets, so it succeeds and hides the problem. This gap is the reason Path MTU Discovery exists.

### Fragmentation and the DF flag

A packet that is too big has two possible fates.

1. **Fragmentation:** a device cuts it into smaller IP packets. The destination joins them again.
2. **Drop:** the device destroys it and reports the problem.

The IPv4 header has a flag named **DF** (Don't Fragment). If DF is 1, routers must not cut the packet. Linux and Windows set DF on TCP packets by default, because they use Path MTU Discovery.

Why avoid fragmentation?

- Losing one fragment loses the whole packet, so retransmits get larger.
- Only the first fragment holds the TCP or UDP ports. Firewalls, NAT devices and load balancers struggle with the rest.
- Rejoining fragments uses memory and CPU, and attackers can abuse this.

RFC 8900 is titled "IP Fragmentation Considered Fragile" and it says exactly this. The offset fields and fragment maths are covered in [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

IPv6 goes further: routers never fragment. Only the source host may fragment, using an extension header.

### Overhead of VPNs and tunnels

A **tunnel** puts a whole IP packet inside another IP packet. The outer packet travels across a network that may know nothing about the inner addresses.

Analogy: you place a letter inside a second, larger envelope. The second envelope adds thickness, and the post office limit stays the same, so the letter inside must get thinner. Where it stops being true: real tunnels may also encrypt the letter and add padding, so the extra size can change from packet to packet.

```text
Before the tunnel:
+--------+---------+-------------------------------+
| IP 20  | TCP 20  | Data 1460                     |  = 1500
+--------+---------+-------------------------------+

Inside GRE, without reducing the size:
+--------+--------+--------+---------+-------------+
| IP 20  | GRE 4  | IP 20  | TCP 20  | Data 1460   |  = 1524
+--------+--------+--------+---------+-------------+
                                     Too big for 1500

Inside GRE, with the inner packet reduced to 1476:
+--------+--------+--------+---------+-------------+
| IP 20  | GRE 4  | IP 20  | TCP 20  | Data 1436   |  = 1500
+--------+--------+--------+---------+-------------+
```

Common overheads on a 1500-byte underlying path. The **inner MTU** is the size left for the packet inside. MSS assumes an IPv4 inner packet with no options.

| Technology | Overhead | Inner MTU | TCP MSS |
|---|---|---|---|
| 802.1Q VLAN tag | 4 at Layer 2 | 1500 | 1460 |
| PPPoE | 8 | 1492 | 1452 |
| IP-in-IP | 20 | 1480 | 1440 |
| GRE, basic header | 24 | 1476 | 1436 |
| GRE with key | 28 | 1472 | 1432 |
| DS-Lite (IPv4 in IPv6) | 40 | 1460 | 1420 |
| VXLAN over IPv4 | 50 | 1450 | 1410 |
| Geneve over IPv4 | 50 plus options | up to 1450 | up to 1410 |
| WireGuard over IPv4 | 60 | 1440 | 1400 |
| VXLAN over IPv6 | 70 | 1430 | 1390 |
| WireGuard over IPv6 | 80 | 1420 | 1380 |
| IPsec ESP tunnel, AES-GCM | 54 to 57 | 1446 | 1406 |
| Same with NAT-T | 62 to 65 | 1438 | 1398 |
| IPsec ESP tunnel, AES-CBC, SHA-256 | 62 to 77 | 1438 | 1398 |
| Same with NAT-T | 70 to 85 | 1422 | 1382 |

Notes on the table:

- **PPPoE** is PPP over Ethernet, used by many home broadband lines. It adds 6 bytes of PPPoE header and 2 bytes of PPP. RFC 4638 lets a PPPoE link keep MTU 1500 if the Ethernet below carries 1508.
- **WireGuard** overhead is 32 bytes of its own, plus 8 of UDP and 20 or 40 of outer IP. wg-quick picks an MTU from the outgoing interface minus 80, so 1420 is the usual default. It subtracts 80 to stay safe even over IPv6.
- **VXLAN** counts the inner Ethernet header: 20 outer IP + 8 UDP + 8 VXLAN + 14 inner Ethernet = 50. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).
- **IPsec** overhead changes because of padding. NAT-T (NAT Traversal) adds 8 bytes of UDP. The table shows the best case for inner MTU. See [IPsec](../09-security/08-ipsec.md).
- **OpenVPN** has no single number. It depends on UDP or TCP transport, cipher and settings.
- **MPLS** adds 4 bytes per label. Providers usually raise their core MTU, so customers do not notice. See [MPLS](../05-routing/13-mpls.md).

For IPsec, engineers rarely use the exact limit. Cisco's common guidance for tunnels is `ip mtu 1400` and `ip tcp adjust-mss 1360`. This leaves room for different ciphers and for a PPPoE link underneath.

Overheads stack. Here is a Kubernetes VXLAN overlay on a WireGuard link on PPPoE:

```text
Start, Ethernet                1500
PPPoE                 -8   =   1492
WireGuard over IPv4  -60   =   1432
VXLAN over IPv4      -50   =   1382  <- inner MTU for the pod
IPv4 + TCP headers   -40   =   1342  <- TCP MSS for the pod
```

### Path MTU Discovery (PMTUD)

**Path MTU** is the smallest MTU of all links on a path. Classic **PMTUD** lets a sender find it without help from any central system. For IPv4 it is defined in RFC 1191. For IPv6 it is defined in RFC 8201.

The process:

1. The sender sets its path MTU estimate to the MTU of its first link.
2. It sends packets with DF set.
3. A router that cannot forward a packet because of its next link drops it.
4. That router sends an ICMP error to the sender, with the next link's MTU inside.
5. The sender lowers its estimate, lowers the TCP segment size, and resends the lost data.
6. If a later link is smaller still, steps 3 to 5 repeat.
7. After some time, the sender tries a larger size, in case the path has changed.

**ICMP** is the Internet Control Message Protocol, the error and test protocol of IP. See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

```text
Server                  R2                            PC
10.20.0.50          tunnel MTU 1400            192.168.10.30
   |                     |                            |
   |-- 1500 bytes, DF -->|                            |
   |                     | too big: DROP              |
   |<- ICMP type 3 code 4|                            |
   |   next-hop MTU 1400 |                            |
   |                     |                            |
   | PMTU to PC = 1400   |                            |
   |-- 1400 bytes, DF -->|======= tunnel ============>|
   |<----------------------- TCP ACK -----------------|
```

The two message formats:

```text
ICMPv4 "Fragmentation Needed and DF set"
+----------+----------+---------------------+
| Type = 3 | Code = 4 | Checksum            |
+----------+----------+---------------------+
| Unused (0)          | Next-Hop MTU        |
+---------------------+---------------------+
| Original IP header + first 8 bytes of     |
| original data (routers often send more)   |
+-------------------------------------------+

ICMPv6 "Packet Too Big"
+----------+----------+---------------------+
| Type = 2 | Code = 0 | Checksum            |
+----------+----------+---------------------+
| MTU (32 bits)                             |
+-------------------------------------------+
| As much of the original packet as fits    |
| without the message going over 1280       |
+-------------------------------------------+
```

The copy of the original header matters. It tells the sender which connection the error belongs to.

Details worth knowing:

- Both message types are often called **PTB** (Packet Too Big).
- Very old routers put 0 in the Next-Hop MTU field. RFC 1191 tells hosts to guess the next lower common value in that case, from a table of plateau values such as 1492, 1006 and 508.
- The result is stored per destination, not per connection.
- RFC 1191 says a host should wait at least 5 minutes after a PTB before trying larger packets, and recommends 10 minutes. On Linux, `net.ipv4.route.mtu_expires` defaults to 600 seconds.
- Path MTU is one-way. The path from A to B and from B to A may be different, so each side discovers its own.

IPv4 and IPv6 compared:

| Topic | IPv4 | IPv6 |
|---|---|---|
| Who may fragment | Source and routers | Source only |
| DF flag | Exists, set by the sender | No flag, behaves as always on |
| Error message | ICMP type 3, code 4 | ICMPv6 type 2, code 0 |
| Minimum MTU | 68 | 1280 |
| Effect of a dropped PTB | Black hole | Black hole |
| Standard | RFC 1191 | RFC 8201 |

### ICMP black holes

An **ICMP black hole** exists when packets are dropped because they are too big, but the PTB message never reaches the sender. The sender keeps resending the same large packet. Each copy is dropped. The connection hangs.

```text
Server                  R2                       Server firewall
   |                     |                            |
   |-- 1500 bytes, DF -->| DROP                       |
   |                     |-- ICMP type 3 code 4 ----->X blocked
   |                     |                            |
   | no ACK, wait, resend the same 1500 bytes ...     |
   |-- 1500 bytes, DF -->| DROP                       |
   | ... repeats until the connection times out       |
```

Common causes:

- A firewall rule such as "block all ICMP", often added in the name of security.
- A cloud security group that does not allow the message in.
- A router configured not to send unreachables. On Cisco IOS, `no ip unreachables` on an interface can have this effect.
- ICMP rate limits on a busy router.
- A router that sends the PTB from a private address, and an edge filter that drops private source addresses.
- A NAT device (address translation) or firewall that does not match the ICMP message to the existing connection.

Typical symptoms:

- The TCP handshake succeeds, because SYN packets are small.
- TLS (the encryption behind HTTPS) stalls when the server sends its certificates, because that is the first large data.
- SSH stops at `expecting SSH2_MSG_KEX_ECDH_REPLY`, or connects and then freezes on large output.
- Small HTTP responses work and large ones hang.
- Captures show the same segment retransmitted again and again.

Important: the block can be on the sender's side, far from the low-MTU link. The missing message must travel back to the sender of the **large** packet, and for a download that sender is the server.

The fix is simple to state. Do not drop these messages.

```text
! Cisco IOS ACL
ip access-list extended OUTSIDE-IN
 permit icmp any any packet-too-big
```

```bash
# Linux, IPv4 and IPv6
iptables  -A INPUT -p icmp   --icmp-type fragmentation-needed -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big     -j ACCEPT
```

Treat these as required for IPv6. RFC 4890 gives advice on filtering ICMPv6, and Packet Too Big is on the "must not be dropped" list.

### MSS clamping

**MSS clamping** is a repair done by a router or firewall in the path. When a TCP SYN or SYN-ACK passes through, the device rewrites the MSS option to a smaller value, then fixes the TCP checksum.

Hosts then build small segments from the start, so PTB is never needed.

Analogy: a clerk at the depot changes the number on your order form to "maximum 3 metres" before it reaches the factory. The factory never builds a truck that is too tall. Where it stops being true: the clerk needs to read the form. If it is sealed (encrypted) or if the goods do not use that form (UDP), this trick does nothing.

```text
PC                   R1 (clamp 1360)               Server
 |--- SYN MSS=1460 ---->|                             |
 |                      |----- SYN MSS=1360 --------->|
 |                      |<---- SYN-ACK MSS=1460 ------|
 |<-- SYN-ACK MSS=1360 -|                             |
 |                      |                             |
 |    Both hosts now send at most 1360 bytes of data  |
```

One clamp on one router fixes both directions, because it rewrites both handshake packets.

Choose the value from the tunnel MTU:

```text
IPv4 MSS = tunnel MTU - 40       1400 - 40 = 1360
IPv6 MSS = tunnel MTU - 60       1400 - 60 = 1340
```

Configuration:

```text
! Cisco IOS, on the tunnel interface
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ipv6 tcp adjust-mss 1340
```

```bash
# Linux iptables: use the MTU of the outgoing route
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Linux iptables: fixed value, only for traffic leaving wg0
iptables -t mangle -A FORWARD -o wg0 -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1380
```

```bash
# Linux nftables: add to a forward chain you already have
nft add rule inet filter forward tcp flags syn tcp option maxseg size set rt mtu
```

Limits of clamping:

- It works only for TCP. UDP traffic, including QUIC, gets no help.
- The device must see the SYN in clear. It must clamp before encryption. A router in the middle of an IPsec tunnel only sees encrypted packets.
- It does not change connections that are already open.
- The value must be low enough. If a smaller link exists elsewhere on the path, TCP still needs PMTUD.
- Traffic on another path, with no clamp, is not covered.

More on the Linux rule syntax is in [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

### PLPMTUD: discovery without ICMP

**PLPMTUD** is Packetization Layer Path MTU Discovery, defined in RFC 4821. The packetization layer is the layer that decides packet sizes, such as TCP.

The idea: stop waiting for an ICMP message. Test the path directly. Send a larger probe. If it is acknowledged, that size works. If it is lost, that size probably does not.

Analogy: instead of waiting for a phone call from the guard, you send one taller truck and see if the customer signs for it. Where it stops being true: a truck can vanish for other reasons. A lost probe may be plain packet loss, so stacks must retry before they decide.

The process:

1. Start from a size known to be safe. This is `search_low`.
2. Treat the local MTU as the top of the range. This is `search_high`.
3. Send a probe between the two, with DF set.
4. Acknowledged: raise `search_low`.
5. Lost: lower `search_high`.
6. Stop when the range is small.
7. Probe again later, since paths change.

A loss of a probe is not treated as congestion, so it does not cut the sending rate. See [TCP Congestion Control](06-tcp-congestion-control.md) for why that matters.

A simplified search with a true path MTU of 1400:

| Step | Range | Probe size | Result |
|---|---|---|---|
| 1 | 1280 to 1500 | 1390 | Acknowledged |
| 2 | 1390 to 1500 | 1445 | Lost |
| 3 | 1390 to 1445 | 1417 | Lost |
| 4 | 1390 to 1417 | 1403 | Lost |
| 5 | 1390 to 1403 | 1396 | Acknowledged |
| 6 | 1396 to 1403 | range small enough, use 1396 | Done |

The result can be slightly below the true value. That is an accepted cost. On Linux, the stop threshold is `net.ipv4.tcp_probe_threshold`, default 8 bytes.

Linux controls for TCP:

| Setting | Value | Meaning |
|---|---|---|
| `net.ipv4.tcp_mtu_probing` | 0 | Off (upstream default) |
| | 1 | Off until a black hole is suspected |
| | 2 | Always on |
| `net.ipv4.tcp_base_mss` | 1024 | Starting MSS on modern kernels |

```bash
# Linux: turn on black hole recovery now
sudo sysctl -w net.ipv4.tcp_mtu_probing=1

# Linux: keep it after reboot
echo 'net.ipv4.tcp_mtu_probing = 1' | sudo tee /etc/sysctl.d/90-mtu.conf
```

Windows has a related feature, black hole detection. It is off by default, and it is not full PLPMTUD. After repeated retransmits, it tries smaller segments.

```powershell
# Windows: enable black hole detection, then restart
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' `
  -Name EnablePMTUBHDetect -Value 1 -Type DWord
```

**DPLPMTUD** (Datagram PLPMTUD, RFC 8899) applies the same idea to datagram transports, such as UDP-based protocols and SCTP. QUIC can use this method, as RFC 9000 allows. A QUIC probe is a datagram filled with padding. See [QUIC](08-quic.md).

Comparison of the three methods:

| | Classic PMTUD | MSS clamping | PLPMTUD |
|---|---|---|---|
| Who acts | Hosts and routers | A device in the path | Hosts only |
| Needs ICMP | Yes | No | No |
| Works for | Any IP traffic that honours it | TCP only | Protocols that implement it |
| Reacts to change | Yes, on the next PTB | No, set at SYN | Yes, slowly |
| Speed of finding | Fast, one round trip | Instant | Slower, needs probes |

### Diagnosing with ping and the DF flag

The trick: send a ping of a chosen size with DF set, then change the size until you find the limit.

Ping size is the ICMP data size, so you must add headers:

```text
IPv4: packet size = ping data + 8 (ICMP) + 20 (IP)
IPv6: packet size = ping data + 8 (ICMP) + 40 (IPv6)

Test for 1500 on IPv4: 1500 - 28 = 1472
Test for 1500 on IPv6: 1500 - 48 = 1452
```

| Path MTU to test | IPv4 ping data | IPv6 ping data |
|---|---|---|
| 9000 | 8972 | 8952 |
| 1500 | 1472 | 1452 |
| 1492 | 1464 | 1444 |
| 1450 | 1422 | 1402 |
| 1420 | 1392 | 1372 |
| 1400 | 1372 | 1352 |
| 1280 | 1252 | 1232 |

The commands:

```powershell
# Windows, IPv4: -f sets DF, -l sets the data size
ping -f -l 1472 10.20.0.50

# Windows, IPv6: -f is IPv4 only, so the host may fragment large pings
ping -6 -l 1452 2001:db8:20::50
```

```bash
# Linux, IPv4: -M do forbids fragmentation, -s sets the data size
ping -M do -s 1472 -c 3 10.20.0.50

# Linux, IPv6
ping -6 -M do -s 1452 -c 3 2001:db8:20::50

# Linux: find the path MTU hop by hop
tracepath -n 10.20.0.50

# macOS: -D sets DF
ping -D -s 1472 -c 3 10.20.0.50
```

```text
! Cisco IOS: size is the WHOLE IP packet, not the data
R1# ping 10.20.0.50 size 1400 df-bit
```

What the results mean:

| Result | Meaning |
|---|---|
| Normal reply | That size fits the whole path, there and back |
| `Packet needs to be fragmented but DF set` (Windows) | Too big, and PTB works or the local MTU is too small |
| `Frag needed and DF set (mtu = 1400)` (Linux) | Too big, and a router told you the limit |
| `local error: message too long, mtu=1400` (Linux) | Too big for your local or cached MTU |
| `M` (Cisco IOS) | Could not fragment |
| Timeout at large sizes, reply at small sizes | Possible black hole: too big, and no PTB reached you |

Other useful commands:

```powershell
# Windows
netsh interface ipv4 show subinterfaces        # interface MTU
netsh interface ipv4 show destinationcache     # learned path MTU
Get-NetIPInterface -AddressFamily IPv4 | Select-Object InterfaceAlias, NlMtu
```

```bash
# Linux
ip link show eth0           # interface MTU
ip route get 10.20.0.50     # shows a cached mtu, if one was learned
ss -tin                     # per-connection mss and pmtu
ip route flush cache        # forget learned values
```

For the tools themselves, see [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

## Example

### The network

Two offices join over an IPsec tunnel with an IP MTU of 1400. The server's firewall allows only TCP 443 and ICMP echo. It drops ICMP error messages.

```text
 Site A: 192.168.10.0/24                 Site B: 10.20.0.0/24

 win-pc   192.168.10.20
 linux-pc 192.168.10.30                  server 10.20.0.50
      |                                        |
      | 192.168.10.1              10.20.0.1    |
    +----+  198.51.100.2  203.0.113.2  +----+
    | R1 |=============================| R2 |
    +----+  Tunnel0, ip mtu 1400       +----+
          172.16.0.1      172.16.0.2

 All LAN links MTU 1500. Internet path MTU 1500.
```

The complaint: "The VPN is up. Ping works. The internal site never finishes loading."

### Step 1: Prove the small path works

```bash
# linux-pc
$ ping -c 2 10.20.0.50
PING 10.20.0.50 (10.20.0.50) 56(84) bytes of data.
64 bytes from 10.20.0.50: icmp_seq=1 ttl=62 time=24.1 ms
64 bytes from 10.20.0.50: icmp_seq=2 ttl=62 time=23.8 ms
```

TTL (Time To Live) is a hop counter. TTL 62 fits: the server starts at 64, and R1 and R2 subtract one each. A small ping sets the baseline. Without this step, a later timeout would prove nothing.

### Step 2: Look at the failure

Capture on the server while `linux-pc` opens the site.

```bash
$ sudo tcpdump -ni eth0 host 192.168.10.30 and tcp port 443
12:04:09.981204 IP 192.168.10.30.51514 > 10.20.0.50.443: Flags [S], seq 3021456110, win 64240, options [mss 1460,sackOK,TS val 811 ecr 0,nop,wscale 7], length 0
12:04:09.981260 IP 10.20.0.50.443 > 192.168.10.30.51514: Flags [S.], seq 918273645, ack 3021456111, win 65160, options [mss 1460,sackOK,TS val 402 ecr 811,nop,wscale 7], length 0
12:04:10.005871 IP 192.168.10.30.51514 > 10.20.0.50.443: Flags [.], ack 1, win 502, length 0
12:04:10.006533 IP 192.168.10.30.51514 > 10.20.0.50.443: Flags [P.], seq 1:518, ack 1, win 502, length 517
12:04:10.200113 IP 10.20.0.50.443 > 192.168.10.30.51514: Flags [.], seq 1:1449, ack 518, win 501, length 1448
12:04:10.412207 IP 10.20.0.50.443 > 192.168.10.30.51514: Flags [.], seq 1:1449, ack 518, win 501, length 1448
12:04:10.836311 IP 10.20.0.50.443 > 192.168.10.30.51514: Flags [.], seq 1:1449, ack 518, win 501, length 1448
12:04:11.684402 IP 10.20.0.50.443 > 192.168.10.30.51514: Flags [.], seq 1:1449, ack 518, win 501, length 1448
```

(Most options are trimmed from the data lines.)

Reading this:

- Both sides announce MSS 1460. Neither side knows about the tunnel.
- The handshake works. The 517-byte TLS ClientHello works.
- The first full-size server packet is retransmitted, with no ACK and a doubling wait. The packet is 1448 + 32 + 20 = 1500 bytes.

This pattern means: suspect MTU.

### Step 3: Measure the path MTU

On Windows, start at 1472.

```powershell
PS C:\> ping -f -l 1472 10.20.0.50

Pinging 10.20.0.50 with 1472 bytes of data:
Reply from 192.168.10.1: Packet needs to be fragmented but DF set.
Packet needs to be fragmented but DF set.
Packet needs to be fragmented but DF set.
Packet needs to be fragmented but DF set.

Ping statistics for 10.20.0.50:
    Packets: Sent = 4, Received = 1, Lost = 3 (75% loss),
```

The first line came from R1. After that, Windows knows the limit and refuses locally.

Now try smaller sizes:

| `ping -f -l` | Packet size | Result |
|---|---|---|
| 1472 | 1500 | Too big |
| 1272 | 1300 | Reply |
| 1372 | 1400 | Reply |
| 1373 | 1401 | Too big |

```powershell
PS C:\> ping -f -l 1372 10.20.0.50

Pinging 10.20.0.50 with 1372 bytes of data:
Reply from 10.20.0.50: bytes=1372 time=24ms TTL=62
Reply from 10.20.0.50: bytes=1372 time=24ms TTL=62
Reply from 10.20.0.50: bytes=1372 time=25ms TTL=62
Reply from 10.20.0.50: bytes=1372 time=24ms TTL=62

Ping statistics for 10.20.0.50:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 24ms, Maximum = 25ms, Average = 24ms
```

The path MTU is 1372 + 28 = **1400**. Windows has stored it:

```powershell
PS C:\> netsh interface ipv4 show destinationcache

Interface 12: Ethernet

PMTU Destination Address                  Next Hop Address
---- ------------------------------------ ----------------
1400 10.20.0.50                           192.168.10.1
```

The same test on Linux:

```bash
$ ping -M do -s 1472 -c 3 10.20.0.50
PING 10.20.0.50 (10.20.0.50) 1472(1500) bytes of data.
From 192.168.10.1 icmp_seq=1 Frag needed and DF set (mtu = 1400)
ping: local error: message too long, mtu=1400
ping: local error: message too long, mtu=1400

--- 10.20.0.50 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2044ms

$ ping -M do -s 1372 -c 2 10.20.0.50
PING 10.20.0.50 (10.20.0.50) 1372(1400) bytes of data.
1380 bytes from 10.20.0.50: icmp_seq=1 ttl=62 time=24.3 ms
1380 bytes from 10.20.0.50: icmp_seq=2 ttl=62 time=24.0 ms

$ ip route get 10.20.0.50
10.20.0.50 via 192.168.10.1 dev eth0 src 192.168.10.30 uid 1000
    cache expires 586sec mtu 1400

$ tracepath -n 10.20.0.50
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.10.1                                          0.611ms
 1:  192.168.10.1                                          0.508ms
 2:  192.168.10.1                                          0.520ms pmtu 1400
 2:  172.16.0.2                                           23.871ms
 3:  10.20.0.50                                           24.310ms reached
     Resume: pmtu 1400 hops 3 back 3
```

And from the router. Remember that Cisco `size` is the full packet.

```text
R1# ping 10.20.0.50 source 192.168.10.1 size 1400 df-bit
Type escape sequence to abort.
Sending 5, 1400-byte ICMP Echos to 10.20.0.50, timeout is 2 seconds:
Packet sent with the DF bit set
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/24/28 ms
```

### Step 4: Find why PMTUD failed

The client received PTB messages, so PMTUD works from Site A to Site B. But the large packets flow **from** the server. R2 is the tunnel entrance in that direction, so R2 must tell the server. Capture again on the server with an ICMP filter:

```bash
$ sudo tcpdump -ni eth0 'icmp[0] == 3 and icmp[1] == 4'
12:06:41.200113 IP 10.20.0.1 > 10.20.0.50: ICMP 192.168.10.30 unreachable - need to frag (mtu 1400), length 556
```

If you see this line on the network card but the server does not react, a host firewall is dropping it. In this network, tcpdump sees the packet before the host firewall does. The server's rule set allows only echo. This is a black hole.

Without shell access on the server, you can get the same conclusion another way: if R2 counts ICMP unreachables sent, and the server keeps resending, the message is not being used.

### Step 5: Fix it, best first

**Fix 1:** allow the PTB message on the server firewall.

```bash
# server (Linux)
sudo iptables -I INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
```

On Windows Firewall, the matching built-in rule is named "Core Networking - Destination Unreachable Fragmentation Needed (ICMPv4-In)".

**Fix 2:** clamp MSS at the tunnel. This protects all TCP hosts behind the tunnel, including ones you do not control.

```text
! R1, Cisco IOS
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/1
 tunnel destination 203.0.113.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VPN-PROFILE
```

**Fix 3 (host side, optional):** enable probing on the server, so it recovers even if another black hole appears.

```bash
sudo sysctl -w net.ipv4.tcp_mtu_probing=1
```

### Step 6: Verify

Open the site again and capture on the server.

```bash
12:15:02.401120 IP 192.168.10.30.51620 > 10.20.0.50.443: Flags [S], seq 77120344, win 64240, options [mss 1360,sackOK,TS val 912 ecr 0,nop,wscale 7], length 0
12:15:02.401178 IP 10.20.0.50.443 > 192.168.10.30.51620: Flags [S.], seq 550132211, ack 77120345, win 65160, options [mss 1460,sackOK,TS val 433 ecr 912,nop,wscale 7], length 0
...
12:15:02.621901 IP 10.20.0.50.443 > 192.168.10.30.51620: Flags [.], seq 1:1349, ack 518, win 501, length 1348
```

- The client sent MSS 1460, but the server received 1360. R1 rewrote it.
- Each data segment now has 1348 bytes: 1360 minus 12 for timestamps.
- Packet size: 1348 + 32 + 20 = 1400. It fits the tunnel.

On the client:

```bash
$ ss -tin dst 10.20.0.50
State  Recv-Q Send-Q  Local Address:Port    Peer Address:Port
ESTAB  0      0       192.168.10.30:51620     10.20.0.50:443
     cubic wscale:7,7 rto:228 rtt:24.6/3.1 ato:40 mss:1348 pmtu:1500 rcvmss:1348 advmss:1448 cwnd:10
```

`pmtu:1500` with `mss:1348` tells the story: the host never needed PMTUD. The clamp did the work. The site now loads.

## Going Deeper

### The exact send size

RFC 9293 defines the effective send MSS, in simple form:

```text
Eff.snd.MSS = min(SendMSS + 20, MMS_S)
              - TCP header size - IP options size
```

Here `SendMSS` is what the peer announced, and `MMS_S` is the largest message the local stack can send. The key point is that TCP header size here includes options. MSS itself is calculated as if there were no options. A sender must shrink its data when it adds options. That is why timestamps reduce 1460 to 1448.

### Efficiency and throughput

Every frame on the wire also needs 8 bytes of preamble and a 12-byte gap. So a 1500-byte MTU costs 1538 bytes of line time.

```text
Without timestamps: 1460 / 1538 = 94.9 percent
With timestamps:    1448 / 1538 = 94.1 percent
```

This explains the famous number: `iperf` on 1 Gbit/s Ethernet shows about 941 Mbit/s. Nothing is broken. The rest is headers and gaps.

With jumbo frames (MTU 9000, timestamps on): 8948 / 9038 = 99.0 percent.

A smaller MSS has other costs:

- More packets per second for the same data, so more CPU work on routers and hosts.
- Slower start. TCP's window grows in units of MSS, and a common initial window is 10 MSS (RFC 6928).
- Lower throughput when there is packet loss, since common congestion control models scale with MSS. See [TCP Congestion Control](06-tcp-congestion-control.md).

### Offloads make captures look wrong

Modern network cards split large data blocks themselves.

- **TSO** (TCP Segmentation Offload): the stack hands the NIC a large block, and the NIC cuts it into MSS-size segments.
- **GSO** (Generic Segmentation Offload): the same idea in software.
- **GRO** (Generic Receive Offload): the receiver merges segments before the stack sees them.

Because tcpdump runs above the driver, you may see `length 14480` on a 1500 MTU link. That is not a bug, and it does not mean jumbo frames are working. To see the real wire view, capture on a switch mirror port, or turn the offloads off for a short test:

```bash
sudo ethtool -K eth0 tso off gso off gro off
```

### Layer 2 MTU and vendor differences

Vendors do not agree on what "MTU" counts.

- Linux and Windows count only the IP packet. MTU 1500 means 1500 bytes of IP.
- Cisco IOS has two values. `mtu` is the interface limit, and `ip mtu` is the IP limit. `ip mtu` cannot be larger than `mtu`.
- Junos counts the Layer 2 header in its interface `mtu`. The media MTU on Ethernet defaults to 1514, which equals IP MTU 1500.

A safe habit: read the vendor's definition, then test with DF ping.

### Jumbo frames

**Jumbo frames** are Ethernet frames above the standard size. 9000 bytes is the common IP MTU, but there is no IEEE standard for that number, and hardware limits differ. Some switches accept 9216 at Layer 2. Cisco Catalyst 9000 switches, for example, use `system mtu 9198`.

Rules for jumbo frames:

- Every device in the Layer 2 domain must agree.
- A mismatch can be silent. Small packets pass and large ones are dropped, often as "giants" on the switch.
- Jumbo frames suit storage, backup and data centre traffic. They are of no use on the public Internet, where the path MTU is at most about 1500 in practice.

### What a tunnel does with a packet that does not fit

A tunnel has two sides to think about: the inner packet and the outer packet.

- **Inner DF is 1 and the packet exceeds the tunnel MTU:** the router drops it and sends a PTB.
- **Inner DF is 0:** the router may fragment the inner packet first, then wrap each piece. This is pre-fragmentation. The destination host does the rejoining.
- **Wrapped first, then too big for the outer link:** the outer packet is fragmented. This is post-fragmentation. The far tunnel router must rejoin it, often in CPU, which is slow.

Pre-fragmentation is the better of the two. Better than both is to set the tunnel MTU and clamp MSS so no fragmentation happens.

The tunnel router also needs to know the MTU of the path to its peer. On a Cisco GRE tunnel, `tunnel path-mtu-discovery` enables this. On IPsec, `crypto ipsec df-bit` controls the DF value on the outer header, and the default is to copy it from the inner packet.

### Load balancers and anycast

ICMP error messages break a common assumption of load balancers. Traffic is sent to a backend by a hash of the addresses and ports. An ICMP message comes from a router, not the client, so the hash is different, and it may reach the wrong backend. The backend with the real connection never hears the news.

Large operators solve this by reading the embedded original header, or by copying PTB messages to all backends in a group. Many also use a lower MSS on their servers. Keep this in mind when PMTUD fails only sometimes, or only for some servers in a pool. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

### PTB messages can be attacked

A forged PTB could make a host use tiny packets, which wastes resources. Defences:

- The host checks the TCP sequence number inside the embedded header, and ignores messages that do not match an active connection. RFC 5927 discusses ICMP attacks on TCP.
- Linux refuses to go below `net.ipv4.route.min_pmtu`, default 552 bytes.
- IPv6 hosts must not lower their estimate below 1280.

This is the correct answer to "should we block ICMP for security?" Block what you do not need, but keep PTB, and rely on these checks.

### UDP and DNS

UDP has no MSS and no handshake to carry one. The application must choose its size. On a 1500 path, the largest IPv4 UDP payload without fragmentation is 1472 bytes (1500 - 20 - 8). See [UDP in Depth](07-udp.md).

DNS shows this well. EDNS lets UDP DNS replies exceed 512 bytes. Large replies were fragmented, and fragments were often lost. The DNS Flag Day 2020 effort recommended a default EDNS buffer of 1232 bytes:

```text
1280 (IPv6 minimum MTU) - 40 (IPv6) - 8 (UDP) = 1232
```

See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

### QUIC

QUIC runs over UDP, so MSS clamping does nothing for it. RFC 9000 requires:

- The path must support UDP payloads of at least 1200 bytes.
- A client's first datagrams must be at least 1200 bytes. This also tests the path.
- Datagrams must not be fragmented at the IP layer, so DF is set where possible.

From 1200, QUIC may probe upward. Over a tunnel where MTU is below 1200 plus headers, QUIC fails, and browsers fall back to TCP.

### Cloud and containers

- **AWS:** many EC2 instance types support MTU 9001 inside a VPC. Traffic through an internet gateway is limited to 1500. So PMTUD matters there, and AWS tells you to allow "Destination Unreachable: fragmentation needed" in security groups for it to work.
- **Google Cloud:** new VPC networks default to MTU 1460, not 1500.
- **Docker:** the bridge network defaults to 1500. If the host link is smaller, for example over a VPN, containers can hit black holes. Set `"mtu"` in `daemon.json`. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).
- **Kubernetes:** each overlay needs a pod MTU below the node MTU. VXLAN on a 1500 network gives 1450. Many CNI plugins detect this for you. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

Check current vendor documentation before you rely on these values, since cloud limits change.

### Symptoms in routing protocols

MTU faults are not only for user traffic.

- **OSPF:** neighbours with mismatched MTU can stick in ExStart or Exchange, because large Database Description packets are refused. See [OSPF](../05-routing/06-ospf.md).
- **BGP:** the session comes up, then drops with a hold timer expiry during the first full table send. Small keepalives pass, while large updates are lost. See [BGP](../05-routing/09-bgp.md).

### Post-quantum TLS

Hybrid key exchange such as X25519MLKEM768 adds over a kilobyte to the TLS ClientHello. It no longer fits in one normal packet, so the **client's** first flight can now hit MTU and middlebox faults that once appeared only in server replies. See [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

### Order of fixes in real networks

1. Allow PTB in all firewalls, ACLs and security groups.
2. Set the right MTU on every tunnel interface.
3. Clamp MSS at tunnel entrances.
4. Raise the underlay MTU where you own the links. Then an overlay can carry 1500 inside.
5. Enable black hole recovery on hosts.

Lowering the MTU on every host is the last resort. It is hard to maintain, and it hurts traffic that never uses the tunnel.

```powershell
# Windows: last resort, set a lower MTU on one interface
netsh interface ipv4 set subinterface "Ethernet" mtu=1400 store=persistent
```

```bash
# Linux: last resort, not persistent across reboot
sudo ip link set dev eth0 mtu 1400

# Linux: better, limit only the route that needs it
sudo ip route replace 10.20.0.0/24 via 192.168.10.1 mtu lock 1400 advmss 1360
```

## Common Mistakes

- **"MSS includes the headers."** No. MSS counts only TCP data. MTU counts IP header, TCP header and data.
- **"MTU includes the Ethernet header."** Not on Linux, Windows or Cisco `ip mtu`. Some vendors, such as Junos, do count it on the interface setting.
- **"MSS and MTU are negotiated between all devices."** MSS is only announced by the two end hosts. MTU is set per link. Routers do not agree on anything unless a clamp rewrites the SYN.
- **"Both sides must use the same MSS."** Each direction is separate. The values may differ.
- **Testing with `ping -l 1500`.** That builds a 1528-byte packet, so it fails on a healthy link. Use 1472 for IPv4.
- **Pinging without DF.** The packet gets fragmented and passes, so you prove nothing.
- **Mixing up ping sizes between systems.** Cisco `size` is the whole IP packet. Windows `-l` and Linux `-s` are data only.
- **Using Windows `-f` with IPv6.** It is IPv4 only. Without DF, the source can fragment a large ping.
- **"Block all ICMP for security."** This creates black holes. Allow at least type 3 code 4 and ICMPv6 type 2.
- **Reading a timeout as proof of a black hole.** Echo may be blocked, or the reply may be lost. Always start with a small ping.
- **Testing only one direction.** Path MTU is one-way, and the block can sit on the far side. Test from both ends if possible.
- **Expecting MSS clamping to fix UDP or QUIC.** Clamping only rewrites TCP SYN packets.
- **Clamping after encryption.** The SYN is hidden by then. Clamp at the entrance, on the tunnel interface.
- **Setting the IPv6 clamp to the IPv4 number.** IPv6 headers are 20 bytes larger, so the MSS must be 20 smaller.
- **Calling a tcpdump `length 14480` a jumbo frame.** It is usually TSO or GRO.
- **"Fragmentation means the network is broken."** Some is normal for non-DF UDP. The problem is depending on it.
- **Setting a huge MTU on one switch only.** Every device in the Layer 2 domain must match, or large frames die silently.

## Interview Questions

**Q: What is the difference between MTU and MSS, and why is the MSS usually 1460?**

A: MTU is the largest IP packet a link can carry, with IP and TCP headers included. MSS is the largest amount of TCP data in a segment, with no headers. Ethernet MTU is 1500. Subtract 20 for IPv4 and 20 for TCP and you get 1460. On IPv6, the IP header is 40, so MSS is 1440.

**Q: How is MSS negotiated?**

A: It is more of an announcement than a negotiation. Each side puts an MSS option in its SYN or SYN-ACK to say the largest segment it can receive. Each sender then stays under the other side's value and under its own path MTU. The values can differ. If the option is missing, the default is 536 for IPv4 and 1220 for IPv6.

**Q: Why do we see 1448 in captures instead of 1460?**

A: The TCP Timestamps option takes 12 bytes in every segment. MSS is calculated without options, so the sender removes 12 from its data. 1460 minus 12 is 1448.

**Q: Explain Path MTU Discovery.**

A: The sender sets DF on its packets and starts with its local MTU. If a router has a smaller next link, it drops the packet and sends ICMP type 3 code 4, or ICMPv6 Packet Too Big, with the next-hop MTU. The sender lowers its size and resends. This repeats until the data gets through. The value is cached per destination and retried later, about 10 minutes on Linux.

**Q: What is an ICMP black hole and how would you recognise one?**

A: It is when a packet is dropped for being too big, but the ICMP error never gets back to the sender, usually because a firewall blocks it. The sender keeps retrying the same large packet. The symptoms are that ping and the TCP handshake work, but TLS or large transfers hang. In a capture you see the same full-size segment retransmitted with no ACK.

**Q: The VPN is up and ping works, but file transfers hang. How do you troubleshoot?**

A: I suspect MTU first. I confirm a small ping works, then run a DF ping: `ping -f -l 1472` on Windows or `ping -M do -s 1472` on Linux. I lower the size until it passes and add 28 to get the path MTU. Then I capture on the sender to see retransmits and check whether PTB messages arrive. To fix it, I allow PTB in firewalls, set the tunnel MTU correctly, and clamp MSS on the tunnel interface.

**Q: What is MSS clamping, and what are its limits?**

A: A router or firewall rewrites the MSS option in TCP SYN and SYN-ACK packets to a lower value, so hosts send smaller segments from the start. On Cisco, it is `ip tcp adjust-mss 1360`. On Linux, it is the iptables TCPMSS target. It only works for TCP, needs to see the SYN in clear, and does nothing for connections already open. UDP and QUIC are not helped.

**Q: Why not let routers fragment everything?**

A: Losing one fragment loses the whole packet. Later fragments have no port numbers, so firewalls, NAT and load balancers handle them badly. Rejoining costs memory and CPU and can be attacked. IPv6 removed router fragmentation altogether, and RFC 8900 explains why fragmentation is fragile.

**Q: How does PLPMTUD differ from classic PMTUD?**

A: Classic PMTUD depends on ICMP coming back from routers. PLPMTUD, from RFC 4821, does not need ICMP. The transport sends probes of different sizes and uses acknowledgements to learn what fits. It works through black holes but is slower. Linux enables it with `tcp_mtu_probing`, and QUIC uses the datagram version from RFC 8899.

**Q: How does a VXLAN overlay end up with an MTU of 1450?**

A: VXLAN adds 50 bytes on IPv4: 20 outer IP, 8 UDP, 8 VXLAN and 14 inner Ethernet. On a 1500 underlay, 1450 is left for the inner IP packet. To carry a full 1500 inside, the underlay needs at least 1550, and many designs simply use jumbo frames.

**Q: How does IPv6 change all of this?**

A: Routers never fragment, so PTB is not optional: there is no fallback. The minimum link MTU is 1280, the message is ICMPv6 type 2, and PMTUD is defined in RFC 8201. The MSS on a 1500 link is 1440. Blocking Packet Too Big on IPv6 breaks traffic just as it does on IPv4, with no router fragmentation as an escape.

**Q: A 9000-byte MTU is set on the servers, but large transfers fail and small ones pass. What is likely wrong?**

A: A switch or other device on the path is probably still at 1500. Because it is the same Layer 2 network, no router sends a PTB, and the switch simply drops the frames. I would test with `ping -M do -s 8972` between hosts, and check switch ports for giant counters. Then I would make the MTU match everywhere.

## Quick Recap

- MTU is the largest IP packet on a link. MSS is TCP data only. 1500 - 20 - 20 = 1460 on IPv4, 1440 on IPv6, and 1448 with timestamps.
- MSS is announced in SYN and SYN-ACK. It knows nothing about the links in the middle.
- Tunnels add headers, which lower the inner MTU: PPPoE 1492, GRE 1476, VXLAN 1450, WireGuard 1420, and IPsec about 1400 in practice.
- PMTUD uses DF and ICMP type 3 code 4 (ICMPv6 type 2). Block those messages and you get a black hole: small traffic works, large traffic hangs.
- MSS clamping rewrites the SYN and fixes TCP only. PLPMTUD probes the path with no ICMP.
- Test with `ping -f -l 1472` on Windows and `ping -M do -s 1472` on Linux, then add 28 to the largest working size. On Cisco, `size` already means the whole packet.
- Best order of fixes: allow PTB, set the tunnel MTU, clamp MSS, then raise the underlay MTU where you can.
