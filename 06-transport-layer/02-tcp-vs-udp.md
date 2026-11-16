# TCP vs UDP

**Level:** Beginner | **Prerequisites:** [The TCP/IP Model](../02-network-models/03-tcp-ip-model.md), [Ports and Sockets](01-ports-and-sockets.md)

## In One Line

TCP gives applications a reliable, ordered stream of bytes, and it pays for that with setup time and extra work. UDP sends single messages with no setup and no promises, and leaves the rest to the application.

## Why It Matters

IP moves packets between computers, but it makes no promises. A packet (one small unit of data on the network) can be lost, copied, or arrive late. Applications need something on top of IP, and they want different things.

A bank transfer must arrive complete, even if it takes a little longer. A voice call must arrive now, even if a tiny piece is missing. No single design is best for both, so the internet has two main transport protocols.

Interviewers ask about TCP vs UDP because the answer shows how you think about trade-offs. A weak answer is "TCP is reliable, UDP is fast". A strong answer explains what each one does, what it costs, and why a real application chose it. Good follow-up questions include "Why does DNS use both?" and "Why does HTTP/3 run on UDP?"

## Simple Explanation

Imagine you want to send a book to a friend, one page at a time. The postal system is IP: it tries to deliver, but it can lose a page.

**TCP is a careful helper.** First, you call your friend: "Are you ready?" Your friend says "Yes, are you ready?" and you say "Yes." This is the setup. You number every page. Your friend reports which pages arrived. If page 7 is missing, you send it again. Your friend puts the pages in order before reading. If your friend is reading slowly, you slow down.

**UDP is a stack of postcards.** You write each postcard and drop it in the box. You do not call first. You do not number anything unless you choose to. Most postcards arrive. Some may not, and nobody tells you.

**Where the analogy stops being true:**

- Postcards are slow in real life, but UDP packets are not slower than TCP packets. Both travel at the same speed on the same wires.
- A TCP "connection" is not a private line. It is only shared memory of the conversation, kept by the two end computers. Routers in the middle normally do not know it exists.
- Each postcard is a separate message, and that part is true: one UDP send makes one message. TCP has no pages at all. It carries one long stream of bytes.

## How It Works

### Where TCP and UDP Sit

Both protocols live at the transport layer (Layer 4). They run on the two end hosts, on top of IP.

```text
+--------------------------------------------------+
| Applications: HTTP, SSH, DNS, NTP, games         |
+------------------------+-------------------------+
|          TCP           |           UDP           |  Layer 4
+------------------------+-------------------------+
|              IP (no delivery promise)            |  Layer 3
+--------------------------------------------------+
|              Ethernet, Wi-Fi, fibre              |  Layers 1-2
+--------------------------------------------------+
```

Both add ports, so one computer can run many network applications at once. See [Ports and Sockets](01-ports-and-sockets.md). The IP header says which one is inside the packet: protocol number 6 means TCP and 17 means UDP.

TCP calls its unit of data a **segment**. UDP calls its unit a **datagram**.

### What TCP Does

1. **Sets up a connection.** The two sides exchange three messages: SYN, SYN-ACK, ACK. This is the three-way handshake. It costs one round trip time (RTT, the time for a packet to go there and for a reply to come back) before the client can send data.
2. **Numbers the data.** Every byte has a sequence number, so the receiver knows the correct order.
3. **Confirms delivery.** The receiver sends an acknowledgment (ACK), a small message saying "I have everything up to this byte".
4. **Resends lost data.** If an ACK does not come, the sender sends the data again. This is retransmission.
5. **Restores order.** The receiver holds early data and gives bytes to the application in the correct order.
6. **Protects the receiver.** The receiver tells the sender how much free buffer space it has. This is flow control.
7. **Protects the network.** The sender slows down when it sees signs of overload. This is congestion control.
8. **Closes cleanly.** Each side sends a FIN message to say "I have finished sending".

The details have their own chapters: [handshake and header](03-tcp-handshake-and-header.md), [reliability and flow control](05-tcp-reliability-and-flow-control.md), and [congestion control](06-tcp-congestion-control.md).

**Fuller truth:** TCP cannot truly guarantee delivery. If the cable is cut, no protocol can deliver. TCP guarantees that the application gets correct, in-order bytes or learns that the connection failed.

### What UDP Does

1. The application gives UDP one message.
2. UDP adds an 8-byte header with ports, length and a checksum (a small number computed from the data, used to detect damage).
3. IP sends the datagram. UDP keeps no record of it.
4. The receiver checks the checksum and the port. If all is well, the application gets the whole message. If not, it is dropped silently.

That is all. UDP adds ports and damage detection to IP. Everything else is the application's job. [UDP in Depth](07-udp.md) covers it fully.

### One Lost Packet, Side by Side

TCP notices the loss and repairs it:

```text
Sender                     Receiver
  |------- SYN --------------->|
  |<------ SYN-ACK ------------|
  |------- ACK --------------->|   connection is ready
  |------- data 1 ------------>|
  |------- data 2 -----X       |   lost in the network
  |------- data 3 ------------>|   held, it cannot be used yet
  |<------ ACK "have 1 only" --|
  |------- data 2 (again) ---->|   retransmission
  |<------ ACK "have 1-3" -----|   app now reads 1, 2, 3
```

UDP does not notice:

```text
Sender                     Receiver
  |------- message 1 --------->|   app gets 1
  |------- message 2 --X       |   lost, nobody is told
  |------- message 3 --------->|   app gets 3
```

### Header Size

The UDP header is 8 bytes. The TCP header is 20 bytes at minimum and up to 60 bytes with options.

UDP header, 32 bits per row:

```text
+------------------+------------------+
| Source Port (16) | Dest Port (16)   |
+------------------+------------------+
| Length (16)      | Checksum (16)    |
+------------------+------------------+
```

TCP header, simplified, 32 bits per row:

```text
+------------------+------------------+
| Source Port (16) | Dest Port (16)   |
+------------------+------------------+
| Sequence Number (32)                |
+-------------------------------------+
| Acknowledgment Number (32)          |
+------------------+------------------+
| Offset and flags | Window (16)      |
+------------------+------------------+
| Checksum (16)    | Urgent Ptr (16)  |
+------------------+------------------+
| Options (0-40 bytes)                |
+-------------------------------------+
```

The TCP fields map directly to its features: sequence number for order, acknowledgment number for delivery, flags for setup and close, window for flow control.

### Table of Differences

| Topic | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented, handshake first | Connectionless, send at once |
| Setup cost | 1 RTT before data | None |
| Delivery | Lost data is resent | Lost data stays lost |
| Order | Bytes arrive in order | No order promise |
| Data model | Byte stream, no message edges | Separate messages (datagrams) |
| Flow control | Yes, with the receive window | None |
| Congestion control | Yes, built in | None, the app must add it |
| Header size | 20-60 bytes | 8 bytes |
| State on hosts | Per-connection state on both ends | Almost none |
| Damage detection | Checksum, always required | Checksum, optional on IPv4, required on IPv6 |
| One to many | No, exactly two endpoints | Yes, broadcast and multicast |
| Message size | No limit, it is a stream | At most 65,507 data bytes on IPv4 |
| Failure signal | Timeout or reset reaches the app | Often none |
| IP protocol number | 6 | 17 |
| Standard | RFC 9293 | RFC 768 |
| Analogy | Careful helper with numbered pages | Postcards |

Two of those numbers need a note. The 65,507 limit is 65,535 (largest IPv4 packet) minus 20 (IP header) minus 8 (UDP header). In practice, applications keep UDP much smaller. On normal Ethernet with a 1500-byte MTU (the largest packet a link carries), 1472 bytes of UDP data fit in one packet. Larger datagrams are split by IP, and losing one piece loses the whole datagram. See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).

### Stream vs Messages

This difference is small on paper and causes real bugs.

```text
Application sends "HELLO", then "WORLD"

TCP reader might get:  "HELLOWORLD"            (one read)
                  or:  "HEL", then "LOWORLD"   (two reads)

UDP reader gets:       "HELLO" and "WORLD"     (two whole messages)
                       if both arrive, in either order
```

TCP keeps the bytes in order but does not keep your message edges. So protocols on TCP must mark them. HTTP uses a Content-Length header. DNS over TCP puts a 2-byte length in front of each message.

### What Happens at a Closed Port

- **TCP:** The server replies with RST (a reset message). The client program reports "Connection refused" at once.
- **UDP:** The host may reply with ICMP Port Unreachable (Type 3, Code 3). A firewall may drop the packet and send nothing. The sender often learns nothing at all.

This is why UDP checks and UDP port scans are slow and uncertain. Silence can mean "open", "closed behind a firewall", or "packet lost". See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

### Which Application Uses Which, and Why

| Application | Transport and port | Why |
|---|---|---|
| Web, HTTP/1.1 and HTTP/2 | TCP 80, 443 | Pages must arrive complete |
| Web, HTTP/3 | UDP 443 (QUIC) | QUIC builds its own reliability on UDP |
| SSH | TCP 22 | Every keystroke must arrive in order |
| Email: SMTP, IMAP, POP3 | TCP 25/587, 143/993, 110/995 | Messages must be complete |
| File transfer: FTP, SFTP | TCP 21 (plus data ports), 22 | One wrong byte ruins a file |
| Databases: MySQL, PostgreSQL | TCP 3306, 5432 | Queries and results must be exact |
| BGP | TCP 179 | Routers need a reliable session |
| DNS | UDP 53, and TCP 53 | Small queries on UDP, large replies on TCP |
| DHCP | UDP 67, 68 | Client has no IP yet and must broadcast |
| NTP | UDP 123 | Tiny exchange, a late reply is useless |
| SNMP | UDP 161, 162 | Light polling of many devices |
| Syslog | UDP 514 (TLS version: TCP 6514) | Light logging, loss is often accepted |
| TFTP | UDP 69 | Very small code, used for boot files |
| Voice and video calls (RTP) | UDP, dynamic ports | Late audio is useless |
| Fast online game updates | Usually UDP | The newest position beats an old one |
| WireGuard, IKE/IPsec | UDP (often 51820), UDP 500/4500 | Tunnels avoid TCP inside TCP |
| VXLAN | UDP 4789 | Tunnel, the inner traffic handles reliability |
| OSPF | Neither, IP protocol 89 | Runs directly on IP |
| ICMP (ping) | Neither, IP protocol 1 | Runs directly on IP |

The port numbers are in the [Common Ports Reference](10-common-ports-reference.md). Three cases deserve more explanation.

**DHCP cannot use TCP.** A new laptop has no IP address and does not know the server. It must broadcast (send to everyone on the LAN). TCP needs two known endpoints, so it cannot do this. See [DHCP](../07-application-protocols/04-dhcp.md).

**Voice cannot wait.** A common voice setup sends one packet every 20 ms, 50 packets per second. If one packet is lost, the listener misses 20 ms of sound and barely notices. If a resent packet arrives 100 ms later, the moment to play it has passed. Worse, TCP would hold all later audio until the old packet arrives. This is called head-of-line blocking: the first missing item in the line blocks everything behind it.

```text
           Packet:    1   2   3   4   5

TCP       Arrived:    1   X   3   4   5    (2 was lost)
       App gets now:  1   .   .   .   .    (3, 4, 5 must wait)
     After resend:    1   2   3   4   5    (complete, but late)

UDP       Arrived:    1   X   3   4   5    (2 was lost)
       App gets now:  1   .   3   4   5    (app hides the gap)
```

**Movie streaming is not on this "needs UDP" list.** Services such as Netflix and YouTube send video as small files over HTTP, using TCP or QUIC. The player keeps several seconds of video in a buffer, so a short delay for retransmission is invisible. Live calls have no such buffer.

### How to Choose

Pick **TCP** when:

- Every byte must arrive (files, web pages, payments).
- You want reliability without building it yourself.
- The traffic must pass strict firewalls. Some networks block most UDP.

Pick **UDP** when:

- Late data is worthless (voice, video calls, live game state).
- The exchange is one small question and one small answer (DNS, NTP).
- You need broadcast or multicast.
- You are building your own transport with your own rules (QUIC, WireGuard).

If you pick UDP for anything serious, you must also handle loss, order, size limits and congestion yourself. Choosing UDP moves that work into your application. It does not remove it.

### DNS Over Both

DNS is the classic interview case because it uses both protocols on port 53. TCP port 53 and UDP port 53 are separate, because each transport has its own set of port numbers.

UDP is the first choice for normal lookups. A query and answer are usually small, so two packets finish the job. A busy server can answer many clients without keeping a connection for each one.

TCP is needed in these cases:

1. **Large answers.** The server cannot fit the answer, so it sets the TC (truncated) flag in its reply. The client then asks again over TCP.
2. **Zone transfers.** AXFR and IXFR copy DNS data between servers. These are large and must be complete, so they run on TCP.
3. **Reuse.** A client can send many queries on one open connection, which is cheap after the first handshake.

```text
Client                           DNS server
  |--- UDP: A? example.com -------->|
  |<-- UDP: reply with TC=1 --------|   too big to fit
  |=== TCP handshake ===============|
  |--- TCP: same query ------------>|
  |<-- TCP: full answer ------------|
```

Classic DNS limited UDP messages to 512 bytes. EDNS(0) (RFC 6891) is an extension that lets a client advertise a larger size. Since the 2020 DNS Flag Day, the common recommendation is 1232 bytes, chosen to avoid IP fragmentation. Answers signed with DNSSEC (digital signatures for DNS data) are often large, so they are more likely to need TCP.

TCP is not optional for DNS. RFC 7766 requires DNS implementations to support it, and RFC 9210 says internet DNS services need it to work. A firewall that blocks TCP 53 will cause failures that look random, because only large answers break.

Newer encrypted options also pick a side:

| Method | Transport | Port | RFC |
|---|---|---|---|
| DNS over TLS (DoT) | TCP | 853 | 7858 |
| DNS over HTTPS (DoH) | TCP or QUIC | 443 | 8484 |
| DNS over QUIC (DoQ) | UDP (QUIC) | 853 | 9250 |

The full DNS story is in [DNS Basics](../07-application-protocols/01-dns-basics.md) and [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

### Myth: UDP Is Always Faster

This is the most common myth. It is partly true and mostly incomplete.

**UDP does win here:**

- **First data.** There is no handshake, so a new exchange saves one RTT.
- **After a loss.** Data behind a lost packet is not held back.
- **Header.** It saves at least 12 bytes per packet. On a 1500-byte packet, that is only 0.8%.

**UDP does not win here:**

- **Speed on the wire.** A UDP packet does not move faster than a TCP packet. Same cables, same routers, same distance.
- **An open TCP connection.** The handshake is already paid for. A new request costs one RTT, the same as UDP.
- **Bulk transfer.** Network cards and operating systems have strong support for TCP, such as TCP Segmentation Offload (the card splits large data into packets, which saves CPU). So on many hosts, TCP moves bulk data at high speed for less CPU than a simple UDP program.
- **Lossy or overloaded paths.** Plain UDP does not slow down. Send 1 Gb/s of UDP into a 100 Mb/s link and at least 90% of the packets must be dropped. A TCP sender adapts to the link and delivers close to 100 Mb/s of useful data.
- **Complete delivery.** If you need every byte, your program must detect loss and resend on UDP too. The time cost comes back, inside your application.

**Fuller truth:** TCP can add small delays by itself. Nagle's algorithm may hold tiny writes to combine them, and programs such as SSH turn it off with the `TCP_NODELAY` socket option. So a slow TCP app is not always TCP's fault, and it is often fixable.

A good interview sentence: "UDP has lower overhead, so it can have lower latency, but it is not faster in general. The right question is what the application needs when a packet is lost."

## Example

### Example 1: One DNS Lookup Over UDP and Over TCP

**Setup:** The client is 192.168.1.20. The DNS resolver is 10.20.0.53, in a head office with an RTT of 20 ms. The answer is already cached.

Run the lookup over UDP, which is the default for `dig`:

```bash
# Linux
dig @10.20.0.53 example.com A
```

```text
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
example.com.        300     IN      A       203.0.113.10

;; Query time: 21 msec
;; SERVER: 10.20.0.53#53(10.20.0.53) (UDP)
;; MSG SIZE  rcvd: 56
```

Now force TCP:

```bash
# Linux
dig +tcp @10.20.0.53 example.com A
```

```text
;; ANSWER SECTION:
example.com.        300     IN      A       203.0.113.10

;; Query time: 41 msec
;; SERVER: 10.20.0.53#53(10.20.0.53) (TCP)
;; MSG SIZE  rcvd: 56
```

The answer is the same, but it took about twice as long. The extra 20 ms is the handshake.

The same test on Windows:

```powershell
# Windows PowerShell
Resolve-DnsName example.com -Type A -Server 10.20.0.53
Resolve-DnsName example.com -Type A -Server 10.20.0.53 -TcpOnly
```

```text
Name           Type   TTL   Section    IPAddress
----           ----   ---   -------    ---------
example.com    A      300   Answer     203.0.113.10
```

Capture the packets to see why. This is `tcpdump` on Linux, with timestamps and some fields removed so it fits. The tool is covered in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

```bash
# Linux
sudo tcpdump -n -i eth0 host 10.20.0.53 and port 53
```

UDP, 2 packets:

```text
IP 192.168.1.20.51514 > 10.20.0.53.53: 41221+ [1au] A? example.com. (52)
IP 10.20.0.53.53 > 192.168.1.20.51514: 41221 1/0/1 A 203.0.113.10 (56)
```

TCP, 9 packets:

```text
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [S], seq 3301200100, length 0
IP 10.20.0.53.53 > 192.168.1.20.43210: Flags [S.], seq 1875500200, ack 3301200101, length 0
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [.], ack 1, length 0
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [P.], seq 1:55, ack 1, length 54 41230+ [1au] A? example.com. (52)
IP 10.20.0.53.53 > 192.168.1.20.43210: Flags [P.], seq 1:59, ack 55, length 58 41230 1/0/1 A 203.0.113.10 (56)
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [.], ack 59, length 0
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [F.], seq 55, ack 59, length 0
IP 10.20.0.53.53 > 192.168.1.20.43210: Flags [F.], seq 59, ack 56, length 0
IP 192.168.1.20.43210 > 10.20.0.53.53: Flags [.], ack 60, length 0
```

How to read it:

- `[S]` is SYN, `[S.]` is SYN-ACK, `[.]` is a plain ACK, `[P.]` carries data, and `[F.]` is FIN.
- TCP data is 2 bytes longer than the DNS message (54 vs 52, 58 vs 56). That is the length prefix TCP needs, because a stream has no message edges.

Now count the total bytes at the IP level, on IPv4:

| Exchange | Calculation | Total |
|---|---|---|
| UDP query | 52 + 8 UDP + 20 IP | 80 bytes |
| UDP reply | 56 + 8 UDP + 20 IP | 84 bytes |
| **UDP total** | 2 packets | **164 bytes** |
| TCP SYN, SYN-ACK | 60 each (typical Linux, with options) | 120 bytes |
| TCP query | 54 + 32 TCP + 20 IP | 106 bytes |
| TCP reply | 58 + 32 TCP + 20 IP | 110 bytes |
| Other 5 packets | 52 each (no data) | 260 bytes |
| **TCP total** | 9 packets | **596 bytes** |

For one tiny lookup, TCP used more than 3 times the bytes and more than 4 times the packets. Multiply this by millions of lookups per second and you see why DNS starts with UDP.

**Fuller truth:** exact packet counts vary. Packets can be merged, and options differ by operating system.

### Example 2: Round Trips Until the Answer

Keep the same 20 ms RTT and ignore processing time.

| Method | Round trips | Time |
|---|---|---|
| DNS over UDP | 1 | 20 ms |
| DNS over TCP, new connection | 2 | 40 ms |
| DNS over TCP, connection already open | 1 | 20 ms |
| DNS over TLS, new connection (TCP + TLS 1.3) | 3 | 60 ms |
| DNS over QUIC, new connection | 2 | 40 ms |

Look at row 3. On an open connection, TCP matches UDP. The cost is in setup, not in every packet. This is why keeping connections open is a major speed trick.

## Going Deeper

### Reliability on Top of UDP

UDP is a blank base, and many protocols build on it:

- **TFTP** sends a block (512 bytes by default) and waits for an ACK before the next one. Simple, and slow on long paths.
- **QUIC** adds connections, encryption, retransmission, and congestion control.
- **WebRTC** (video calls in browsers) uses RTP, with loss repair and bandwidth adaptation added.

So "uses UDP" does not mean "accepts loss". It often means "handles loss in its own way".

### Why HTTP/3 Runs on UDP

Changing TCP is hard for two reasons. TCP lives inside operating systems, which update slowly. Also, firewalls and other middle boxes often block unknown TCP behaviour or new protocol numbers. UDP passes through most networks, and it leaves all behaviour to the application.

QUIC (RFC 9000) uses that freedom. It joins transport and TLS 1.3 setup into one handshake. It also carries many streams, so a lost packet blocks only its own stream. If UDP is blocked, browsers fall back to HTTP/2 over TCP. See [QUIC](08-quic.md) and [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

**TCP inside TCP.** VPNs avoid this when they can. When a packet is lost, the inner and outer TCP may both retransmit and both slow down, and performance can collapse. This is why WireGuard uses UDP and why OpenVPN uses UDP by default.

### SCTP and DCCP in Brief

TCP and UDP are not the only transport protocols.

**SCTP** (Stream Control Transmission Protocol, RFC 9260, IP protocol 132) is reliable like TCP but keeps message edges like UDP. Two features stand out:

- **Multi-streaming:** one association (SCTP's word for a connection) carries many streams. Loss in one stream does not block the others.
- **Multi-homing:** one association can use several IP addresses on each end, so it survives the failure of one network path.

SCTP also uses a four-way handshake with a cookie, which resists SYN floods. Its main home is telecom signalling, such as links between 4G and 5G network functions. WebRTC data channels also use it, wrapped inside DTLS and UDP. It is rare on the public internet because many NAT devices and firewalls do not handle it.

**DCCP** (Datagram Congestion Control Protocol, RFC 4340, IP protocol 33) is like UDP with congestion control: unreliable datagrams, but with a connection setup and a rate that adapts to the network. It was designed for media apps that want to be fair to other traffic. It saw very little real use, and recent Linux kernels have removed it. QUIC and application-level congestion control filled the need instead.

| Protocol | Reliable | Ordered | Congestion control | Data model | Runs on |
|---|---|---|---|---|---|
| TCP | Yes | Yes | Yes | Byte stream | IP (6) |
| UDP | No | No | No | Datagrams | IP (17) |
| QUIC | Yes | Within a stream | Yes | Many streams | UDP |
| SCTP | Yes | Within a stream (optional) | Yes | Messages, many streams | IP (132) |
| DCCP | No | No | Yes | Datagrams | IP (33) |

### State in the Middle of the Network

UDP has no connection, but stateful firewalls and NAT devices still track UDP flows. A stateful device remembers each conversation so it can allow the replies. With no FIN to mark the end, it uses a timer. UDP timers are usually much shorter than timers for established TCP, and values vary by vendor. Long-lived UDP apps therefore send small keepalive packets. See [Firewalls](../09-security/04-firewalls.md) and [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

### Security Differences

Neither TCP nor UDP encrypts data. TLS, QUIC, and IPsec add that.

The real difference is proof of address. The TCP handshake shows that the client can receive packets at its claimed IP. UDP has no such step, so an attacker can fake the source address. The attacker sends a small request with the victim's address, and a server sends a much larger reply to the victim. This is a reflection and amplification attack, often using DNS or NTP. TCP has its own attack, the SYN flood. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

## Common Mistakes

- **"TCP is reliable and UDP is unreliable, so TCP is better."** Reliable here is a technical word. It means delivery is checked and repaired. For a voice call, that repair makes the service worse.
- **"UDP has no checksum."** It has one. It is optional on IPv4 (zero means not used) and required on IPv6, with a narrow tunnel exception.
- **"DNS uses UDP."** DNS uses both. Blocking TCP 53 breaks large answers and zone transfers.
- **"Each TCP send is one message at the other end."** TCP is a byte stream. A program that assumes one read equals one send has a bug that may appear only under load.
- **"UDP is less secure than TCP" or "TCP is secure."** Neither encrypts. The difference is that UDP sources are easier to fake.
- **"An ACK means the application processed my data."** It means the receiving TCP has the data. The application may crash before it reads it. Important systems add their own confirmation.
- **"The TCP checksum makes data perfect."** It is a weak 16-bit check, and some damage can pass it. TLS gives much stronger protection.
- **"Video streaming uses UDP."** Video calls usually do. Movie streaming mostly uses HTTP on TCP or QUIC.
- **"UDP cannot have congestion control."** Plain UDP does not, but an application on UDP can. QUIC does.
- **"Ping uses TCP or UDP."** Ping uses ICMP, which runs directly on IP and has no ports.
- **"A port is open on my server, so UDP traffic works."** You may have tested only TCP. TCP 443 and UDP 443 are different ports, and firewall rules treat them separately.

## Interview Questions

**Q: What is the main difference between TCP and UDP?**
A: TCP is connection-oriented. It gives a reliable, ordered byte stream with flow control and congestion control. UDP is connectionless. It sends single datagrams with no delivery or order promise. TCP costs more setup and state, and UDP leaves those jobs to the application.

**Q: Is UDP always faster than TCP?**
A: No. UDP skips the handshake and does not hold data after a loss, so it can have lower latency. But packets do not travel faster, and on an open connection TCP also takes one RTT per request. For bulk transfer, TCP is often better, because it adapts to the network and has strong hardware offload.

**Q: Why does DNS use both UDP and TCP?**
A: Normal queries are small, so UDP finishes in two packets and the server keeps no connection state. If an answer does not fit, the server sets the TC flag and the client retries on TCP. Zone transfers also use TCP. RFC 7766 requires TCP support, so a firewall must allow both on port 53.

**Q: Why do voice calls use UDP?**
A: Audio is useful only at the moment it plays. A resent packet would arrive too late, and TCP would hold later audio behind it. It is better to lose 20 ms of sound and keep going. The application hides small gaps.

**Q: Why does DHCP use UDP?**
A: The client has no IP address yet and does not know the server, so it must broadcast. TCP needs two known endpoints and a handshake, so it cannot work here.

**Q: Why can TCP not do multicast?**
A: A TCP connection is between exactly two endpoints. It tracks sequence numbers and ACKs for one peer, and that does not scale to a group. UDP keeps no per-peer state, so one datagram can go to many receivers.

**Q: Can you make UDP reliable?**
A: Yes, at the application level. You add sequence numbers, ACKs, retransmission, and congestion control. TFTP does a simple version and QUIC does a full one. The work does not disappear, it moves from the kernel to the application.

**Q: Why does HTTP/3 run on UDP and not on a new protocol?**
A: A new IP protocol would be blocked by many firewalls and NAT devices. TCP is hard to change because it lives in operating systems and middle boxes depend on how it looks. UDP passes through most networks, so QUIC builds on it and adds reliability, TLS 1.3, and independent streams.

**Q: What happens if you send to a closed port?**
A: With TCP, the host replies with RST and the client sees "Connection refused". With UDP, the host may send ICMP Port Unreachable, but a firewall may drop the packet silently. So the UDP sender often learns nothing.

**Q: What is the difference between a byte stream and a datagram?**
A: A datagram keeps its edges: one send arrives as one message. A stream does not: two sends may come as one read, or one send as two reads. So protocols on TCP add their own framing, such as Content-Length in HTTP or the 2-byte length in DNS over TCP.

**Q: Which protocols run on neither TCP nor UDP?**
A: ICMP is IP protocol 1, OSPF is 89, ESP from IPsec is 50, and GRE is 47. They sit directly on IP. They have no TCP or UDP ports, which matters when you write firewall rules.

**Q: When would you choose SCTP over TCP?**
A: When you need reliable messages with several independent streams, or need one association to survive a path failure through multi-homing. It is common in telecom signalling. On the public internet, NAT and firewall support is poor, so QUIC is usually the practical choice.

## Quick Recap

- TCP gives a connection, reliable and ordered bytes, flow control, and congestion control. UDP gives ports and a checksum, then sends single datagrams.
- TCP headers are 20-60 bytes, and new connections cost 1 RTT. UDP headers are 8 bytes, with no setup.
- Use TCP when every byte matters. Use UDP when late data is useless, for tiny exchanges, for broadcast or multicast, or to build a custom transport.
- DNS uses UDP first, then TCP for truncated answers and zone transfers. Both must be allowed on port 53.
- UDP is not always faster. It saves setup and avoids waiting after a loss, but packets are not quicker, and full reliability must be built by the application.
- QUIC brings reliable, encrypted streams on top of UDP. SCTP is a niche reliable message protocol used in telecom. DCCP is rarely used.
