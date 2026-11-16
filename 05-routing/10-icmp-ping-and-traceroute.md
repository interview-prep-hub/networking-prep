# ICMP, Ping and Traceroute

**Level:** Intermediate | **Prerequisites:** [How Routers Work](01-how-routers-work.md), [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md), [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md)

## In One Line

ICMP is the message system that IP uses to report errors and answer simple tests. Ping and traceroute are small tools built on top of those messages.

## Why It Matters

IP has no way to say "that failed". A router that drops a packet needs some way to tell the sender why. ICMP is that way.

Without ICMP:

- A sender would not learn that a destination has no route.
- A packet stuck in a routing loop would die in silence.
- A sender would not learn that its packets are too big for the path.
- IPv6 hosts could not even find each other on the same LAN.

Interviewers ask about this topic for three reasons:

1. Ping and traceroute are the first tools used in almost every network problem.
2. The tools are often misread. A candidate who knows that `* * *` is not always a failure stands out.
3. "Should we block ICMP on the firewall?" is a real design question, and the common answer ("yes, for security") is wrong.

## Simple Explanation

Think of the postal service. You post a letter (an IP packet). Sometimes the post office sends you a small note:

- "No such street." (Destination Unreachable)
- "This letter passed through too many post offices, so we threw it away." (Time Exceeded)
- "This parcel is too big for our van." (Fragmentation Needed)

ICMP is that small note. It is not your letter. It is a note about your letter.

Ping is like sending a postcard that says "please write back". If a reply comes, you know the address exists and the road works both ways.

Traceroute is a clever trick. Imagine you can write on a letter: "Throw this away after 1 post office." That post office discards it and sends you a note, and the note shows its address. Next you write "after 2 post offices". Now the second one writes to you. Repeat this, and you learn every post office on the way.

**Where the analogy stops being true:** the postal service tries hard to deliver its notes. ICMP does not. ICMP notes can be lost, blocked by a firewall, or never sent because a router is busy. No note does not prove that nothing went wrong.

## How It Works

### What ICMP is

ICMP means **Internet Control Message Protocol**. A protocol is an agreed set of rules for how devices talk.

- ICMP for IPv4 is defined in RFC 792.
- ICMP for IPv6 (ICMPv6) is defined in RFC 4443.

ICMP messages travel inside IP packets. In the IPv4 header, the Protocol field is **1** for ICMP. In IPv6, the Next Header value is **58** for ICMPv6.

```text
+-----------------+-----------------+----------------------------+
| Ethernet header | IPv4 header     | ICMP message               |
|                 | Protocol = 1    | Type, Code, Checksum, data |
+-----------------+-----------------+----------------------------+
```

Three facts about ICMP surprise beginners:

- **It has no ports.** TCP and UDP have port numbers. ICMP has Type and Code numbers instead. So "ping port 80" has no meaning.
- **It is a Layer 3 protocol.** It rides inside IP, which looks like Layer 4. But it exists to serve IP itself, so we treat it as part of the network layer.
- **It is not reliable.** ICMP has no acknowledgment and no retransmission. A lost ICMP message stays lost.

ICMP messages come in two groups:

- **Error messages** report a problem with a packet. Example: Time Exceeded.
- **Query messages** (also called informational) ask and answer. Example: Echo Request and Echo Reply.

### The ICMP header

Every ICMP message starts with the same 4 bytes. The next 4 bytes depend on the type.

```text
 0               8               16                            31
+---------------+---------------+-------------------------------+
|   Type (8)    |   Code (8)    |         Checksum (16)         |
+---------------+---------------+-------------------------------+
|              Rest of header (32 bits, depends on type)        |
+---------------------------------------------------------------+
|              Data (size depends on type)                      |
+---------------------------------------------------------------+
```

- **Type** gives the main category, such as "Destination Unreachable".
- **Code** gives the detail, such as "port unreachable".
- **Checksum** detects damage to the message.

An Echo message uses the "rest of header" like this:

```text
+---------------+---------------+-------------------------------+
|  Type 8 or 0  |   Code 0      |           Checksum            |
+---------------+---------------+-------------------------------+
|           Identifier          |        Sequence number        |
+-------------------------------+-------------------------------+
|              Payload (test data, often a timestamp)           |
+---------------------------------------------------------------+
```

An error message carries a copy of the packet that caused the problem:

```text
+---------------+---------------+-------------------------------+
|     Type      |     Code      |           Checksum            |
+---------------+---------------+-------------------------------+
|     Unused (for type 3 code 4: unused + next-hop MTU)         |
+---------------------------------------------------------------+
|   Original IP header + at least first 8 bytes of its data     |
+---------------------------------------------------------------+
```

That copy is very useful. The first 8 bytes of the original data hold the UDP or TCP ports, or the ICMP identifier and sequence. So the sender can match the error to the exact program and packet that caused it.

**Fuller truth:** RFC 792 asks for the IP header plus 8 bytes. RFC 1812 says routers should quote as much as fits within 576 bytes. Many routers still quote only 28 bytes, and many hosts quote more.

### ICMPv4 types and codes

These are the ones an interviewer may ask about.

| Type | Name | Code | Meaning |
|---|---|---|---|
| 0 | Echo Reply | 0 | Answer to a ping |
| 3 | Destination Unreachable | 0 | Network unreachable |
| 3 | | 1 | Host unreachable |
| 3 | | 2 | Protocol unreachable |
| 3 | | 3 | Port unreachable |
| 3 | | 4 | Fragmentation needed, but DF is set |
| 3 | | 5 | Source route failed |
| 3 | | 6 | Destination network unknown |
| 3 | | 7 | Destination host unknown |
| 3 | | 9 | Network administratively prohibited |
| 3 | | 10 | Host administratively prohibited |
| 3 | | 13 | Communication administratively prohibited (filtered) |
| 4 | Source Quench | 0 | Deprecated by RFC 6633, do not use |
| 5 | Redirect | 0 | Better gateway for this network |
| 5 | | 1 | Better gateway for this host |
| 8 | Echo Request | 0 | A ping |
| 9 | Router Advertisement | 0 | Router announces itself (rarely used in IPv4) |
| 10 | Router Solicitation | 0 | Host asks for routers (rarely used in IPv4) |
| 11 | Time Exceeded | 0 | TTL reached 0 in transit |
| 11 | | 1 | Fragment reassembly time exceeded |
| 12 | Parameter Problem | 0 | Bad value in the IP header |
| 12 | | 1 | Required option missing |
| 12 | | 2 | Bad length |
| 13 / 14 | Timestamp Request / Reply | 0 | Clock query, usually blocked |

Types 17 and 18 (Address Mask) and types 15 and 16 (Information) are old. RFC 6918 formally deprecates them.

**Numbers worth remembering:** 8 and 0 (ping), 3 (unreachable), 11 (TTL expired), 3/3 (port unreachable), 3/4 (too big).

Who sends what:

- A **router** sends 3/0 when it has no route to the destination.
- The **last router** often sends 3/1 when it cannot find the host with [ARP](../03-ethernet-and-switching/05-arp.md) on the final subnet.
- The **destination host** sends 3/3 when no program listens on that UDP port.
- A **router or firewall** with a filter may send 3/13, or may stay silent.

### When ICMP errors are not sent

ICMP errors could create storms of more errors. So there are rules (RFC 1122). A device must not send an ICMP error about:

- Another ICMP error message.
- A packet sent to a broadcast or multicast address.
- A fragment that is not the first fragment.
- A packet whose source is not a single host (for example, a source of 0.0.0.0).

Query messages such as Echo Request are normal traffic. A router can send a Time Exceeded about an Echo Request. That is how Windows `tracert` works.

### How ping works

Ping checks two things: can I reach this address, and how long does the round trip take? That time is the **RTT** (round-trip time).

The name comes from sonar, the sound pulse that ships use to find objects. "Packet Internet Groper" was added later as a joke expansion.

```text
Host A 192.168.1.10                       Host B 203.0.113.10
      |                                           |
      |--- Echo Request (type 8, id 7, seq 1) --->|
      |                                           |
      |<-- Echo Reply   (type 0, id 7, seq 1) ----|
      |                                           |
      |   RTT = time reply arrived - time sent    |
```

Step by step:

1. Ping turns the name into an IP address, if you gave a name. This uses [DNS](../07-application-protocols/01-dns-basics.md).
2. The host looks in its routing table. Is the target on the local subnet, or must it go through the [default gateway](../04-ip-addressing/08-default-gateway.md)?
3. The host uses ARP to find the next-hop MAC address, if it does not have it yet.
4. Ping builds an Echo Request (type 8, code 0) with an identifier, a sequence number and a payload.
5. Each router on the path lowers the TTL by 1 and forwards the packet. **TTL** (Time To Live) is a counter in the IP header. Its job is to stop packets from looping forever.
6. The target receives the request. Its operating system builds an Echo Reply (type 0, code 0). It copies the identifier, sequence and payload.
7. The sender matches the reply by identifier and sequence, then computes the RTT.

Important detail: the **operating system** answers ping, not an application. A server can reply to ping while its web server is dead.

Default behaviour differs by system. These are common defaults on current versions.

| Item | Windows `ping` | Linux `ping` | Cisco IOS `ping` |
|---|---|---|---|
| Number of packets | 4, then stops | Runs until Ctrl+C | 5, then stops |
| Payload size | 32 bytes | 56 bytes | 100 bytes total packet |
| IP packet size | 60 bytes | 84 bytes | 100 bytes |
| Wait for reply | 4 seconds | Depends on RTT and options | 2 seconds |
| Interval | About 1 second | 1 second | Next packet right after reply |
| Run forever | `-t` | Default | Set a large repeat count |
| Set count | `-n 10` | `-c 10` | `repeat 10` |
| Set size | `-l 1400` | `-s 1400` | `size 1400` |
| Set TTL | `-i 5` | `-t 5` | Extended ping |
| Set DF bit | `-f` | `-M do` | `df-bit` |

Watch out: `-t` means "run forever" on Windows but "set TTL" on Linux.

Size math on Linux: 56 bytes payload + 8 bytes ICMP header + 20 bytes IPv4 header = 84 bytes. That is why Linux prints `56(84)`.

### How traceroute abuses TTL

TTL was built for safety, not for mapping paths. Traceroute uses it for something its designers never planned. That is why people say it "abuses" TTL. Van Jacobson wrote the original tool in the late 1980s.

The rule that makes it work:

- A router lowers the TTL by 1.
- If the TTL reaches 0, the router drops the packet.
- The router then sends **Time Exceeded (11/0)** back to the source.
- The source address of that message is an address of the router.

So traceroute sends packets with TTL 1, then TTL 2, then TTL 3. Each one dies one router further away. Each router reveals itself.

This chapter uses one path for all the examples:

```text
Laptop ---- R1 ---- R2 ---- R3 ---- R4 ---- Server

Laptop  192.168.1.10
R1      192.168.1.1
R2      10.20.0.1
R3      198.51.100.1
R4      198.51.100.9
Server  203.0.113.10
```

```text
Laptop      R1          R2          R3          R4        Server
  |  TTL=1  |           |           |           |           |
  |-------->x           |           |           |           |
  |<--------|  Time Exceeded from R1            |           |
  |  TTL=2  |           |           |           |           |
  |-------------------->x           |           |           |
  |<--------------------|  Time Exceeded from R2            |
  |  TTL=3  |           |           |           |           |
  |-------------------------------->x           |           |
  |<--------------------------------|  Time Exceeded from R3
  |  TTL=4  |           |           |           |           |
  |-------------------------------------------->x           |
  |<--------------------------------------------|  From R4
  |  TTL=5  |           |           |           |           |
  |-------------------------------------------------------->|
  |<--------------------------------------------------------|
  |        Port Unreachable (UDP) or Echo Reply (ICMP)      |
```

Step by step:

1. Traceroute sends 3 probes with TTL 1. A **probe** is a test packet.
2. R1 lowers TTL to 0, drops each probe, and sends Time Exceeded.
3. Traceroute records R1's address and three RTT values. That is line 1 of the output.
4. It repeats with TTL 2, then TTL 3, and so on.
5. A probe finally reaches the server. The server's reply is a different kind of message.
6. That different message tells traceroute it has arrived, so it stops. If it never arrives, it stops at the hop limit (30 by default).

A **hop** is one router on the path. Each line of output is one hop.

**How does traceroute know it arrived?** The answer depends on the probe type. Next section.

### Windows ICMP vs Linux UDP

The two most common tools send different probes.

| Item | Windows `tracert` | Linux `traceroute` | Cisco IOS `traceroute` |
|---|---|---|---|
| Probe type | ICMP Echo Request | UDP to high ports | UDP to high ports |
| First port | None, ICMP has no ports | 33434 | 33434 |
| Port change | Sequence number changes | +1 for each probe | +1 for each probe |
| Reply from routers | Time Exceeded (11/0) | Time Exceeded (11/0) | Time Exceeded (11/0) |
| Reply from target | Echo Reply (0/0) | Port Unreachable (3/3) | Port Unreachable (3/3) |
| Probes per hop | 3 | 3 | 3 |
| Max hops | 30 | 30 | 30 |
| Wait per probe | 4 seconds | Up to 5 seconds | 3 seconds |
| Change probe type | Not possible | `-I` ICMP, `-T` TCP | Not in basic command |

Why does UDP use high ports? Traceroute picks ports that no program should be using. The target has nothing listening, so it answers Port Unreachable. That error is the "you have arrived" signal.

With 30 hops and 3 probes per hop, the UDP probes use up to 90 ports: **33434-33523**. You will see this range in firewall rules.

**Why this matters in real life:** the two tools can give different results on the same path. A firewall may allow ICMP Echo but drop UDP to high ports. Then `tracert` reaches the server and `traceroute` shows stars at the end. Other firewalls do the reverse.

macOS `traceroute` also uses UDP by default. Modern Linux `traceroute` can also send several probes at once, up to 16 by default, which makes it faster.

Useful variants:

```bash
# Linux
traceroute -n 203.0.113.10            # UDP, no DNS lookups
sudo traceroute -I -n 203.0.113.10    # ICMP Echo probes, like Windows
sudo traceroute -T -p 443 -n 203.0.113.10   # TCP SYN to port 443
tracepath -n 203.0.113.10             # No root needed, also shows path MTU
traceroute -6 2001:db8:50::10         # IPv6
```

```powershell
# Windows
tracert -d 203.0.113.10               # -d means no DNS lookups
tracert -h 15 203.0.113.10            # Max 15 hops
tracert -6 2001:db8:50::10            # IPv6
Test-NetConnection 203.0.113.10 -TraceRoute   # PowerShell, uses ICMP
```

```text
! Cisco IOS
R1# traceroute 203.0.113.10
R1# traceroute 203.0.113.10 source Loopback0 numeric
```

### Reading ping results

| What you see | What it usually means |
|---|---|
| Reply with time and TTL | Path works both ways, ICMP allowed |
| `Request timed out` (Windows), or silence then 100% loss (Linux) | No reply came. Host down, firewall drop, or no return route |
| `Destination host unreachable` | A device could not reach the host, often ARP failed on the final subnet |
| `Destination net unreachable` | A router has no route |
| `TTL expired in transit` | Routing loop, or TTL set too low |
| `Packet needs to be fragmented but DF set` (Windows), `Frag needed and DF set` (Linux) | Packet is bigger than a link on the path |
| `Packet filtered` (Linux) | Filter sent 3/13 |
| `General failure` or `transmit failed` (Windows) | Problem on the local host, such as a VPN, driver or firewall |
| `Network is unreachable` (Linux, no `From` address) | Local host has no route, packet never left |
| `Name or service not known` (Linux), `could not find host` (Windows) | DNS failed. This is not a reachability result |

Check who sent the error. `Reply from 10.20.0.1: Destination net unreachable` means R2 sent it. So R2 is where to look.

Windows has a trap here. Below, 192.168.1.77 is on the local subnet but does not answer ARP. The laptop reports the failure to itself, and Windows counts each error as a received reply.

```powershell
C:\> ping 192.168.1.77

Pinging 192.168.1.77 with 32 bytes of data:
Reply from 192.168.1.10: Destination host unreachable.
Reply from 192.168.1.10: Destination host unreachable.
Reply from 192.168.1.10: Destination host unreachable.
Reply from 192.168.1.10: Destination host unreachable.

Ping statistics for 192.168.1.77:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
```

**TTL in the reply tells you the return distance.** Common starting values are 64 (Linux, macOS), 128 (Windows) and 255 (many network devices). If you see TTL=60, the sender probably started at 64 and the reply crossed 4 routers.

**Fuller truth:** the starting TTL is only a default and can be changed. This is a clue, not proof of the operating system. It also measures the return path, which can differ from the forward path.

**Jitter:** the difference in RTT between packets is called **jitter**. Steady 24 ms is healthy. Values jumping between 20 ms and 300 ms suggest a busy link.

**Packet loss:** send more than 4 packets. A single lost ping in 4 proves very little. Use 100 or more when you check for real loss.

Cisco IOS prints a symbol for each packet:

| Symbol | Meaning |
|---|---|
| `!` | Reply received |
| `.` | Timed out |
| `U` | Destination unreachable received |
| `N` | Network unreachable received |
| `P` | Protocol unreachable received |
| `Q` | Source quench received |
| `M` | Could not fragment |
| `?` | Unknown packet type |

A first ping of `.!!!!` is normal on a Cisco router. The first packet was lost or delayed while ARP was running. Later packets work.

### Reading traceroute results

Each line shows a hop number, the responding address, and one time per probe.

Five rules for reading it:

1. **Each RTT is a full round trip** from you to that hop and back. It is not the time between two hops.
2. **Look at the trend, not at one hop.** Real delay continues to all later hops.
3. **A high value at one hop only, with lower values after it, is not a problem.** That router is simply slow at creating ICMP messages.
4. **Loss at one hop that does not continue to later hops is not real loss.** Traffic passing through was fine.
5. **Traceroute shows only the forward path.** The replies can return by a different way, so RTT includes a path you cannot see.

Letters after a time show an ICMP error other than Time Exceeded (Linux and Cisco):

| Mark | Meaning |
|---|---|
| `!N` | Network unreachable |
| `!H` | Host unreachable |
| `!P` | Protocol unreachable |
| `!F` | Fragmentation needed (Linux) |
| `!X` | Communication administratively prohibited (Linux) |
| `!A` | Administratively prohibited (Cisco) |
| `*` | No reply before the timeout |

Things traceroute does not show:

- **Layer 2 switches.** They do not touch the TTL, so they never appear.
- **Some firewalls.** Cisco ASA, for example, does not lower the TTL by default, so it stays hidden.
- **Tunnels and many MPLS paths.** The inside of the tunnel looks like one hop.

The address printed for a hop is usually the interface that faces you. This is common practice, not a promise. It can be another interface, a loopback, or even a private address. Treat it as "some address on that router".

### Why asterisks are not always failures

An asterisk `*` means one thing only: **no reply reached the tool before the timeout**. It does not say the probe was dropped. It does not say the path is broken.

Common harmless causes:

- **The router is set not to send Time Exceeded.** The router still forwards traffic normally. It just stays quiet.
- **ICMP rate limiting.** A router has fast hardware to forward packets, but its CPU must build each ICMP message. Routers limit how many they create per second. RFC 1812 recommends this and RFC 4443 requires it for ICMPv6. With many probes at once, some get no answer.
- **The reply was filtered on the way back.** A firewall between you and the router dropped the Time Exceeded message.
- **A firewall on the route drops your probe type.** UDP to port 33434 may be blocked while real traffic on port 443 passes.
- **The final host ignores probes.** It drops the UDP or the Echo Request, so the trace ends in stars even though the service works.
- **The probe or reply was lost by chance.** One `*` among three replies at a hop is common and means little.

How to decide:

```text
Are there replies from LATER hops?
   |
   +-- yes --> Stars are harmless. Traffic passed this hop.
   |
   +-- no, stars run to the end
          |
          +-- Does the real application work?
                 |
                 +-- yes --> The destination or a firewall
                 |           just hides from probes.
                 |
                 +-- no  --> Possible real failure near the
                             last hop that answered.
                             Try another probe type.
                             Check return routes on both sides.
```

**Analogy:** think of a train station that refuses to answer questions. The trains still pass through. If you see your train arrive at the next station, the quiet station is not the problem.

**Where the analogy stops being true:** a quiet hop at the very end is different. There is no "next station" to prove that traffic passed.

### ICMPv6

ICMPv6 does more than ICMPv4 did. In IPv4, separate protocols handle some jobs (ARP, IGMP). In IPv6, ICMPv6 does these jobs too.

Numbering has a simple rule: **types 0-127 are errors, types 128-255 are informational.**

| Type | Name | Code | Notes |
|---|---|---|---|
| 1 | Destination Unreachable | 0 | No route to destination |
| 1 | | 1 | Administratively prohibited |
| 1 | | 2 | Beyond scope of source address |
| 1 | | 3 | Address unreachable |
| 1 | | 4 | Port unreachable |
| 1 | | 5 | Source address failed policy |
| 1 | | 6 | Reject route to destination |
| 2 | Packet Too Big | 0 | Holds the MTU of the next link |
| 3 | Time Exceeded | 0 | Hop limit reached 0 |
| 3 | | 1 | Fragment reassembly time exceeded |
| 4 | Parameter Problem | 0-2 | Bad header field, unknown next header, unknown option |
| 128 | Echo Request | 0 | Ping |
| 129 | Echo Reply | 0 | Ping reply |
| 130-132 | MLD | 0 | Multicast group membership (replaces IGMP) |
| 133 | Router Solicitation | 0 | Host asks for routers |
| 134 | Router Advertisement | 0 | Router gives prefix and gateway |
| 135 | Neighbor Solicitation | 0 | Like an ARP request |
| 136 | Neighbor Advertisement | 0 | Like an ARP reply |
| 137 | Redirect | 0 | Better first hop |
| 143 | MLDv2 Report | 0 | Multicast membership report |

Types 133-137 form **NDP** (Neighbor Discovery Protocol, RFC 4861). See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

Key differences from ICMPv4:

| Item | ICMPv4 | ICMPv6 |
|---|---|---|
| Carried in | IPv4, Protocol 1 | IPv6, Next Header 58 |
| Echo types | 8 and 0 | 128 and 129 |
| Time exceeded | Type 11 | Type 3 |
| Too big | Type 3 code 4 | Type 2, its own message |
| Address resolution | ARP, a separate protocol | NDP, inside ICMPv6 |
| Checksum | ICMP message only | Also covers a pseudo-header with the IPv6 addresses |
| Rate limiting of errors | Recommended | Required |
| Blocking all of it | Breaks PMTUD and diagnostics | Breaks basic IPv6 on the LAN too |

Ping and traceroute work the same way in IPv6. The TTL field is called **Hop Limit**, but the behaviour is the same.

```bash
# Linux
ping -c 4 2001:db8:50::10
ping -c 2 ff02::1%eth0              # All nodes on the link, via eth0
ping -c 2 fe80::1%eth0              # Link-local needs the interface
```

```powershell
# Windows
ping -6 2001:db8:50::10
ping fe80::1%12                     # 12 is the interface index
```

A link-local address (starting with fe80) exists on every link. So the host must be told which interface to use. That is the part after `%`.

### Why blocking all ICMP breaks PMTUD

**MTU** (Maximum Transmission Unit) is the biggest IP packet a link can carry. On normal Ethernet it is 1500 bytes. Tunnels and VPNs have smaller MTUs because they add their own headers.

**PMTUD** (Path MTU Discovery) is how a sender finds the smallest MTU on a whole path. It is defined by RFC 1191 for IPv4 and RFC 8201 for IPv6. Full details are in [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md). Here we focus on why ICMP matters.

How it works:

1. The sender sends packets with the **DF** (Don't Fragment) bit set. TCP does this on most modern systems.
2. A router meets a link that is too small. It cannot split the packet because DF is set, so it drops it.
3. The router sends ICMP **3/4** (IPv6: **Packet Too Big**), which includes the MTU of the small link.
4. The sender makes its packets smaller and sends again.

Now add a firewall that drops all ICMP. Step 3 never reaches the sender. The sender does not know why its packets vanish. It keeps sending large packets, and they keep dying.

This is called a **PMTUD black hole**: packets go in, and nothing comes out, and no one is told.

The symptoms are strange because small packets still work:

- Ping works (small packets).
- The TCP handshake works (small packets).
- SSH login works, and small web pages load.
- File uploads, large pages, or TLS handshakes with big certificates hang.

"Connects but hangs on big data" is the classic sign.

**Analogy:** a low bridge on a road. A driver in a tall truck is stopped there, and a guard writes a note: "Max height 4 metres." If someone tears up every note, the driver keeps sending the same tall truck forever.

**Where the analogy stops being true:** a truck driver can see the bridge and turn around. An IP sender learns nothing unless the note arrives.

IPv6 makes this worse. Routers **never** fragment IPv6 packets. Only the source can fragment. So Packet Too Big is the only normal way to learn about a smaller path. RFC 4890 lists Packet Too Big as a message that firewalls must not drop.

Fixes, best first:

1. **Allow the needed ICMP**: 3/4 for IPv4, type 2 for IPv6.
2. **Clamp the TCP MSS** on the router at the tunnel. **MSS** (Maximum Segment Size) is the biggest TCP payload a side will accept. Lowering it in the SYN packet keeps TCP packets small. This helps TCP only.
3. **Use PLPMTUD** (Packetization Layer Path MTU Discovery, RFC 4821, and RFC 8899 for datagrams). The sender tests sizes on its own, using lost probes as the signal. QUIC can use this method. On Linux, TCP can do it with the `net.ipv4.tcp_mtu_probing` setting.

## Example

### Example 1: Ping and traceroute on a healthy path

We use the path from earlier: laptop 192.168.1.10, gateway 192.168.1.1, server 203.0.113.10.

Linux:

```bash
$ ping -c 4 203.0.113.10
PING 203.0.113.10 (203.0.113.10) 56(84) bytes of data.
64 bytes from 203.0.113.10: icmp_seq=1 ttl=60 time=24.6 ms
64 bytes from 203.0.113.10: icmp_seq=2 ttl=60 time=24.1 ms
64 bytes from 203.0.113.10: icmp_seq=3 ttl=60 time=24.3 ms
64 bytes from 203.0.113.10: icmp_seq=4 ttl=60 time=24.2 ms

--- 203.0.113.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 24.100/24.300/24.600/0.187 ms
```

Windows:

```powershell
C:\> ping 203.0.113.10

Pinging 203.0.113.10 with 32 bytes of data:
Reply from 203.0.113.10: bytes=32 time=25ms TTL=60
Reply from 203.0.113.10: bytes=32 time=24ms TTL=60
Reply from 203.0.113.10: bytes=32 time=24ms TTL=60
Reply from 203.0.113.10: bytes=32 time=24ms TTL=60

Ping statistics for 203.0.113.10:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 24ms, Maximum = 25ms, Average = 24ms
```

Reading it: the server's TTL started at 64. R4, R3, R2 and R1 each lowered it by 1, so it arrived as 60.

Now the traceroute.

Linux:

```bash
$ traceroute -n 203.0.113.10
traceroute to 203.0.113.10 (203.0.113.10), 30 hops max, 60 byte packets
 1  192.168.1.1   1.112 ms  0.982 ms  0.951 ms
 2  10.20.0.1     8.431 ms  8.206 ms  8.377 ms
 3  198.51.100.1  12.604 ms  12.511 ms  12.733 ms
 4  198.51.100.9  23.118 ms  22.947 ms  23.201 ms
 5  203.0.113.10  24.310 ms  24.122 ms  24.208 ms
```

Windows:

```powershell
C:\> tracert -d 203.0.113.10

Tracing route to 203.0.113.10 over a maximum of 30 hops

  1    <1 ms    <1 ms    <1 ms  192.168.1.1
  2     8 ms     8 ms     8 ms  10.20.0.1
  3    13 ms    12 ms    13 ms  198.51.100.1
  4    23 ms    23 ms    23 ms  198.51.100.9
  5    24 ms    24 ms    24 ms  203.0.113.10

Trace complete.
```

Cisco IOS, run from R1:

```text
R1# ping 203.0.113.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 203.0.113.10, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 20/23/28 ms

R1# traceroute 203.0.113.10
Type escape sequence to abort.
Tracing the route to 203.0.113.10
VRF info: (vrf in name/id, vrf out name/id)
  1 10.20.0.1 8 msec 8 msec 8 msec
  2 198.51.100.1 12 msec 12 msec 12 msec
  3 198.51.100.9 24 msec 20 msec 24 msec
  4 203.0.113.10 24 msec 24 msec 24 msec
```

Reading the Linux result:

- There are 5 lines but 4 routers. The last line is the server itself.
- The jump from 12 ms to 23 ms at hop 4 is real, because it continues at hop 5. The link between R3 and R4 is probably long.

Here is what the probes look like on the wire, captured with [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md):

```bash
$ sudo tcpdump -ni eth0 'icmp or udp portrange 33434-33523'
IP 192.168.1.10.48122 > 203.0.113.10.33434: UDP, length 32
IP 192.168.1.1 > 192.168.1.10: ICMP time exceeded in-transit, length 36
IP 192.168.1.10.48125 > 203.0.113.10.33437: UDP, length 32
IP 10.20.0.1 > 192.168.1.10: ICMP time exceeded in-transit, length 36
IP 192.168.1.10.48134 > 203.0.113.10.33446: UDP, length 32
IP 203.0.113.10 > 192.168.1.10: ICMP 203.0.113.10 udp port 33446 unreachable, length 68
```

The capture is shortened to show one probe per hop for hops 1, 2 and 5. Probe sizes check out: 20 bytes IP + 8 bytes UDP + 32 bytes data = 60 bytes.

Hop 5 starts at port 33446 because each of the 4 earlier hops used 3 ports: 33434 + (4 x 3) = 33446.

The router's error is 36 bytes: 8 bytes ICMP + 28 bytes quoted. The server quoted the full 60-byte probe, so its error is 68 bytes.

### Example 2: Reading four unusual traces

**Case A: stars in the middle, path is fine.**

```text
 1  192.168.1.1   1.1 ms  1.0 ms  0.9 ms
 2  10.20.0.1     8.4 ms  8.2 ms  8.3 ms
 3  * * *
 4  198.51.100.9  23.1 ms  22.9 ms  23.2 ms
 5  203.0.113.10  24.3 ms  24.1 ms  24.2 ms
```

R3 does not answer. But hops 4 and 5 answer, so R3 is forwarding. There is no problem.

**Case B: stars at the end, website works.**

```text
 1  192.168.1.1   1.1 ms  1.0 ms  0.9 ms
 2  10.20.0.1     8.4 ms  8.2 ms  8.3 ms
 3  198.51.100.1  12.6 ms  12.5 ms  12.7 ms
 4  198.51.100.9  23.1 ms  22.9 ms  23.2 ms
 5  * * *
 6  * * *
 ...
30  * * *
```

The server's firewall drops UDP to high ports. Test with a port the server really uses:

```bash
$ sudo traceroute -n -T -p 443 203.0.113.10
traceroute to 203.0.113.10 (203.0.113.10), 30 hops max, 60 byte packets
 1  192.168.1.1   1.2 ms  1.0 ms  1.0 ms
 2  10.20.0.1     8.5 ms  8.3 ms  8.4 ms
 3  198.51.100.1  12.7 ms  12.6 ms  12.5 ms
 4  198.51.100.9  23.0 ms  23.1 ms  23.0 ms
 5  203.0.113.10  24.4 ms  24.3 ms  24.2 ms
```

The server answers because port 443 is open, so the path is fine.

**Case C: a scary number that means nothing.**

```text
 1  192.168.1.1   1.1 ms  1.0 ms  0.9 ms
 2  10.20.0.1     8.4 ms  8.2 ms  8.3 ms
 3  198.51.100.1  182.6 ms  176.1 ms  191.4 ms
 4  198.51.100.9  23.1 ms  22.9 ms  23.2 ms
 5  203.0.113.10  24.3 ms  24.1 ms  24.2 ms
```

Hop 3 looks slow, but hops 4 and 5 are normal. Traffic passing through R3 is not delayed. Only the CPU work of creating ICMP is slow. This is a false alarm.

**Case D: a routing loop.**

```text
 1  192.168.1.1   1.1 ms  1.0 ms  0.9 ms
 2  10.20.0.1     8.4 ms  8.2 ms  8.3 ms
 3  198.51.100.1  12.6 ms  12.5 ms  12.7 ms
 4  198.51.100.9  23.1 ms  22.9 ms  23.2 ms
 5  198.51.100.1  34.2 ms  34.0 ms  34.3 ms
 6  198.51.100.9  44.9 ms  45.1 ms  44.8 ms
 7  198.51.100.1  55.7 ms  55.5 ms  55.6 ms
```

The same addresses repeat and the RTT keeps growing. R3 and R4 are sending the packet back and forth. This is a real problem. A ping from Windows would show `TTL expired in transit`.

### Example 3: A PMTUD black hole

A client uploads a file over a VPN tunnel. SSH login works. The upload freezes at 0%.

```text
Client ------- FW ------- R1 ===tunnel=== R2 ------- Server
10.1.1.10  10.1.1.1   10.1.2.2  MTU 1400  10.2.2.1  10.2.2.20
           10.1.2.1
```

The firewall FW has a rule that many admins write: allow Echo Request and Echo Reply, drop all other ICMP. Ping works, so everyone thinks ICMP is fine. But 3/4 is dropped. For PMTUD, this is the same as blocking everything.

```text
Client                FW             R1                   Server
  |-- 1500 bytes, DF=1 ------------->|                       |
  |                   |              | 1500 > 1400: drop     |
  |                   |<-- ICMP 3/4 -|  next-hop MTU = 1400  |
  |                   x  FW drops it                         |
  |   (client hears nothing)                                 |
  |-- retransmit 1500, DF=1 -------->|  drop again           |
```

**Step 1: find the size limit.** 1372 bytes of data + 8 ICMP + 20 IP = 1400. One byte more should fail.

```bash
# Linux on the client. -M do sets DF
$ ping -c 3 -M do -s 1372 10.2.2.20
PING 10.2.2.20 (10.2.2.20) 1372(1400) bytes of data.
1380 bytes from 10.2.2.20: icmp_seq=1 ttl=61 time=31.2 ms
1380 bytes from 10.2.2.20: icmp_seq=2 ttl=61 time=30.9 ms
1380 bytes from 10.2.2.20: icmp_seq=3 ttl=61 time=31.1 ms

--- 10.2.2.20 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 30.900/31.067/31.200/0.125 ms

$ ping -c 3 -M do -s 1373 10.2.2.20
PING 10.2.2.20 (10.2.2.20) 1373(1401) bytes of data.

--- 10.2.2.20 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2046ms
```

```powershell
# Windows on the client. -f sets DF, -l sets payload size
C:\> ping -f -l 1372 10.2.2.20       # Works
C:\> ping -f -l 1373 10.2.2.20       # Request timed out.
```

Small works, one byte more gets total silence, and no error message appears. That silence is the black hole. A healthy network would answer with an error.

**Step 2: fix the firewall.** Allow the useful error messages.

```text
! Cisco IOS ACL on FW, traffic from the outside
ip access-list extended OUTSIDE-IN
 permit icmp any any echo-reply
 permit icmp any any packet-too-big
 permit icmp any any time-exceeded
 permit icmp any any unreachable
 deny   icmp any any
 permit ip any any
```

```bash
# Linux firewall doing the same job
iptables  -A FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT
ip6tables -A FORWARD -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT
```

**Step 3: check again.**

```bash
$ ping -c 3 -M do -s 1373 10.2.2.20
PING 10.2.2.20 (10.2.2.20) 1373(1401) bytes of data.
From 10.1.2.2 icmp_seq=1 Frag needed and DF set (mtu = 1400)
ping: local error: message too long, mtu=1400
ping: local error: message too long, mtu=1400

--- 10.2.2.20 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2030ms
```

Now the client gets a clear answer from R1 at 10.1.2.2. The host remembers the new MTU, so the later packets fail locally without leaving the host. On Linux you can see that memory:

```bash
$ ip route get 10.2.2.20
10.2.2.20 via 10.1.1.1 dev eth0 src 10.1.1.10 uid 1000
    cache expires 582sec mtu 1400
```

**Optional extra: clamp the MSS on R1.** TCP packets can then fit even if some other firewall drops ICMP later. 1400 - 20 (IP) - 20 (TCP) = 1360.

```text
! Cisco IOS on R1
interface Tunnel0
 ip mtu 1400
 ip tcp adjust-mss 1360
```

```bash
# Linux router
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
```

## Going Deeper

### What is safe to block

"Block all ICMP" is bad advice. "Allow all ICMP" from the internet is lazy. A good policy sits between them.

| Message | Advice at a border firewall |
|---|---|
| IPv4 3/4, IPv6 Packet Too Big | Always allow |
| Other Destination Unreachable | Allow |
| Time Exceeded (IPv4 11, IPv6 3) | Allow, needed for traceroute and loop detection |
| Parameter Problem | Allow |
| Echo Request/Reply | Usually allow, with a rate limit, by policy |
| IPv4 Redirect (5) | Block from untrusted networks |
| IPv4 Timestamp (13, 14) | Block, it leaks clock information |
| Source Quench, Address Mask | Block, deprecated |
| IPv6 NDP (133-137) | Allow on the local link only, not across the border |

RFC 4890 gives detailed ICMPv6 filtering advice. See [Firewalls](../09-security/04-firewalls.md) and [ACLs](../09-security/05-access-control-lists.md).

NDP messages must arrive with Hop Limit 255 (RFC 4861). A router lowers that value, so 255 proves the sender is on the same link. A remote attacker cannot fake it.

Stateful firewalls help. They can link an ICMP error to an existing connection using the quoted packet. In Linux conntrack, this state is called RELATED. But a rule like `iptables -A INPUT -p icmp -j DROP` placed before the state rule still kills these errors.

Cisco's `no ip unreachables` is another trap. People use it to hide a router from scanners. It also stops the router from sending 3/4, which can create a black hole.

Cloud firewalls can also bite. AWS security groups do not allow inbound ICMP unless you add a rule, and AWS documents a specific rule for PMTUD. See [Security Groups, NACLs and VPC Connectivity](../11-cloud-and-containers/04-cloud-security-and-connectivity.md).

### Control plane vs data plane

A router has two sides. The **data plane** forwards packets, often in fast hardware. The **control plane** is the CPU, which runs routing protocols and builds ICMP messages.

An expired packet must go to the CPU. So does a ping sent to the router itself. Routers protect the CPU with limits. Cisco calls one such feature **CoPP** (Control Plane Policing).

This explains why ping to a router can show loss or delay while traffic through it is perfect. Ping **through** a router tests the data plane. Ping **to** a router tests its busy CPU.

### ECMP and false paths

**ECMP** (Equal-Cost Multi-Path) means a router uses several equal paths at once. It usually picks a path by a hash of the packet's addresses and ports, so that one flow stays on one path.

Classic UDP traceroute changes the destination port on every probe. So each probe can take a different path. You may see two or three different addresses on one line:

```text
 3  198.51.100.1  12.6 ms  198.51.100.5  12.9 ms  198.51.100.1  12.5 ms
```

This is not a fault. It can even produce fake links between routers that are not directly connected.

**Paris traceroute** solves this. It keeps the flow fields the same for all probes and varies something the hash ignores. TCP traceroute to one port also keeps the flow steadier. `mtr` is covered in [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

### MPLS and traceroute

In an [MPLS](13-mpls.md) network, routers forward by label and may not have a route back to you. A common behaviour: the router sends the Time Exceeded forward along the label path to its end, and only then does it return to you. So several hops in a row can show the same high RTT, close to the total time for the far end. This is a side effect, not delay at each hop.

Some operators turn off TTL copying into MPLS, so the core vanishes and appears as one hop. RFC 4950 lets routers add MPLS labels to ICMP errors. Linux `traceroute -e` can display them.

### ICMP and NAT

ICMP has no ports, so how does [NAT](../04-ip-addressing/09-nat-and-pat.md) track a ping? It uses the Echo **identifier** in place of a port. RFC 5508 describes this behaviour.

For ICMP errors, NAT has more work to do. It must change the outer IP header and also the quoted packet inside the message. Otherwise the inside host could not match the error to its connection.

### Who hears UDP errors

TCP and UDP report a closed port differently.

- **TCP** answers a SYN to a closed port with a **RST** packet. This is not ICMP.
- **UDP** has no such signal, so the host sends ICMP 3/3.

A connected UDP socket then gets a "connection refused" error. See [UDP in Depth](../06-transport-layer/07-udp.md).

A firewall set to REJECT can fake this. Linux `iptables -j REJECT` sends 3/3 by default, so a blocked UDP traceroute may seem to arrive at the firewall.

### Attacks that use ICMP

Short notes here, with details in [Common Network Attacks](../09-security/03-common-network-attacks.md).

- **Ping flood:** send so many Echo Requests that a link or CPU fills up.
- **Smurf attack:** send Echo Requests to a broadcast address with a faked source. All hosts reply to the victim. Cisco IOS has disabled directed broadcast by default since 12.0.
- **Ping of death:** an old attack using fragments that reassemble to more than 65,535 bytes, the maximum IP size. Systems have been patched for decades.
- **ICMP tunnelling:** hiding data inside Echo payloads to get past firewalls.
- **Redirect attacks:** faked Redirects that send a victim's traffic through the attacker.
- **Forged errors:** faked ICMP errors can disturb TCP connections. RFC 5927 explains this. Modern stacks check the quoted packet to reject many fakes.

### Newer ICMP features

- **Multipart messages (RFC 4884):** ICMP errors can carry structured extra data, such as MPLS labels.
- **PROBE (RFC 8335):** Extended Echo (ICMP types 42 and 43, ICMPv6 types 160 and 161) can ask about the state of an interface on a remote node without reaching it directly.

### Testing tips

Ping from a specific source. This tests the return route for that address.

```text
! Cisco IOS
R1# ping 203.0.113.10 source Loopback0
R1# ping 203.0.113.10 size 1500 df-bit
```

```bash
# Linux
ping -I 192.168.1.10 203.0.113.10
```

```powershell
# Windows
ping -S 192.168.1.10 203.0.113.10
```

Ping and `tracert` tell you nothing about TCP ports. See [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

On Linux, sending raw ICMP needs special permission. Modern systems often solve this with file capabilities or the `net.ipv4.ping_group_range` setting. That is why ping can fail inside a locked-down container.

## Common Mistakes

- **"Ping failed, so the host is down."** Many hosts and firewalls drop Echo Requests. Windows Firewall commonly blocks inbound Echo by default. Test the real service port.
- **"Ping works, so the application works."** Ping only proves that IP and ICMP work. The OS answers, not the app. Port 443 could still be closed.
- **"`* * *` means the router is down."** It means no reply came back in time. If later hops respond, traffic is passing.
- **"Blocking ICMP makes us safe."** It hides little and breaks PMTUD, traceroute and error reporting. In IPv6 it can break the whole LAN.
- **"Allowing Echo means ICMP is allowed."** Echo is only two types. The 3/4 message is a different type and can still be dropped.
- **"Each traceroute line shows the delay of that link."** Each line is a full round trip from the source. To estimate a link, compare trends at several hops.
- **"High delay at a middle hop is the problem."** It is a problem only if it continues to all later hops.
- **"Traceroute shows the return path."** It shows only the forward path. Run a trace from the other side to see the return.
- **"ICMP uses a port."** It has no ports. Type and Code do that job.
- **"ICMP is at Layer 4 because it sits inside IP."** It is treated as Layer 3, a part of IP's own control system.
- **"Traceroute uses ICMP probes everywhere."** Windows does. Linux, macOS and Cisco IOS use UDP by default. All of them depend on ICMP replies.
- **"Request timed out and Destination host unreachable are the same."** A timeout means silence. Unreachable means some device answered and told you it failed.
- **"Windows says 0% loss, so it worked."** Windows counts `Destination host unreachable` replies as received. Read the lines, not just the summary.
- **"Ping 127.0.0.1 tests the network card."** Loopback traffic never reaches the network card. It tests only the local IP stack.

## Interview Questions

**Q: What is ICMP and at which layer does it work?**

A: ICMP is the Internet Control Message Protocol. IP uses it to report errors and to run simple tests. It works at Layer 3. It is carried inside IP with protocol number 1, or Next Header 58 for ICMPv6, and it has no ports.

**Q: How does ping work?**

A: The host sends an ICMP Echo Request, type 8, with an identifier and sequence number. The target's operating system answers with an Echo Reply, type 0, and copies those values. The sender matches them and computes the round-trip time. In IPv6 the types are 128 and 129.

**Q: How does traceroute work?**

A: It sends probes with TTL 1, then 2, then 3, and so on. Each router that lowers the TTL to 0 drops the probe and sends ICMP Time Exceeded, which reveals its address. The tool repeats this until the destination answers with a different message, or it reaches 30 hops.

**Q: What is the difference between Windows tracert and Linux traceroute?**

A: Windows sends ICMP Echo Requests, and the destination ends the trace with an Echo Reply. Linux sends UDP to high ports, starting at 33434, and the destination ends the trace with ICMP Port Unreachable. Firewalls can treat these differently, so the results can differ. On Linux I can also use `-I` for ICMP or `-T` for TCP.

**Q: You see `* * *` at hop 5, but hops 6 to 9 respond. Is there a problem?**

A: No. Hop 6 can only answer if the traffic passed hop 5. Hop 5 is either not sending Time Exceeded, rate limiting it, or its replies are filtered. I would not investigate it unless other symptoms point there.

**Q: The trace ends in stars, but the website loads. Why?**

A: The destination or its firewall is ignoring the probe type. It drops UDP to high ports or ICMP, but allows port 443. I would run `traceroute -T -p 443` so the probe looks like the real traffic.

**Q: Hop 3 shows 200 ms, and hops 4 to 8 show 25 ms. What does that mean?**

A: Hop 3 is not slow for traffic passing through. Building ICMP messages is low priority on a router's CPU, so its replies are slow. Real delay would continue on all later hops.

**Q: Why should you not block all ICMP?**

A: Because Path MTU Discovery depends on it. When a packet with DF set is too big, a router drops it and sends ICMP type 3 code 4. If that is blocked, the sender never learns and keeps sending large packets. You get a black hole: small traffic works, but big transfers hang. It also breaks traceroute and useful errors. In IPv6 it breaks neighbor discovery too.

**Q: How would you detect an MTU black hole?**

A: The symptom is that connections open but stall on large data. I would ping with DF set and change the size. On Linux that is `ping -M do -s 1472`, on Windows `ping -f -l 1472`. If a size fails silently, with no "fragmentation needed" message, then the ICMP error is being dropped. The fix is to allow that message, and if needed clamp the TCP MSS.

**Q: What are the main differences between ICMPv4 and ICMPv6?**

A: ICMPv6 does much more. It holds Neighbor Discovery, which replaces ARP, and router discovery through RS and RA. It also holds MLD for multicast. Echo becomes types 128 and 129. Packet Too Big is its own type, type 2, and it is essential because IPv6 routers never fragment. You cannot block all ICMPv6 and still have a working network.

**Q: What does the difference between "Request timed out" and "Destination host unreachable" tell you?**

A: Timed out means no response came at all, so a host or firewall may be silently dropping packets, or the return route may be missing. Unreachable means a device gave an error. I look at who sent it. If it comes from my own host or the last router, ARP probably failed for the target.

**Q: How does ping pass through NAT if ICMP has no ports?**

A: NAT uses the ICMP Echo identifier as if it were a port. It maps the identifier to the inside host, so the reply goes back to the correct device. For ICMP errors, NAT also rewrites the original packet quoted inside the message.

## Quick Recap

- ICMP carries IP's error and test messages. It runs at Layer 3, has no ports, and is not reliable. Protocol 1 for IPv4, Next Header 58 for IPv6.
- Ping uses Echo Request and Reply: types 8 and 0 in IPv4, 128 and 129 in IPv6. It proves little about applications.
- Traceroute raises the TTL one step at a time and collects Time Exceeded (11/0) messages.
- Windows `tracert` uses ICMP Echo. Linux and Cisco use UDP from port 33434 and stop at Port Unreachable (3/3).
- A `*` means only "no reply in time". If later hops answer, the star is harmless. Delay or loss matters only if it continues to later hops.
- ICMPv6 also handles NDP and MLD, so blocking it breaks basic IPv6.
- Blocking ICMP 3/4 or Packet Too Big creates PMTUD black holes: small traffic works, big transfers hang.
