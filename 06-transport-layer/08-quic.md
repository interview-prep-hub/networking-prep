# QUIC

**Level:** Advanced | **Prerequisites:** [TCP vs UDP](02-tcp-vs-udp.md), [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md), [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md), [TCP Congestion Control](06-tcp-congestion-control.md), [UDP in Depth](07-udp.md), [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md)

## In One Line

QUIC is a reliable, encrypted transport protocol that runs on top of UDP. It gives one connection many independent streams, a faster setup, and the ability to survive an IP address change.

## Why It Matters

TCP plus TLS works well, but it has four problems that are very hard to fix inside TCP.

1. **One lost packet stops everything.** HTTP/2 sends many files over one TCP connection. TCP only knows one ordered line of bytes. If one packet is lost, every file waits, even files whose data arrived safely. This is called head-of-line blocking (HOLB): the item at the front of the line blocks all items behind it.
2. **Setup is slow.** TCP needs one round trip. TLS 1.3 needs another one. A round trip time (RTT) is the time for a packet to go to the other side and for the answer to come back. On a 100 ms mobile link, that is 200 ms before the first request can leave.
3. **The connection is tied to an IP address.** A TCP connection is identified by four values: source IP, source port, destination IP, destination port. Walk out of Wi-Fi coverage onto mobile data and your IP changes. The TCP connection dies.
4. **TCP is frozen.** Firewalls, NAT devices and load balancers read TCP headers and sometimes drop anything they do not recognise. TCP also lives in the operating system kernel, so a new feature takes years to reach users. People call this ossification: the protocol has turned hard like bone and can no longer change.

QUIC solves all four. It carries HTTP/3, which a large share of web traffic now uses. Browsers, big websites and CDNs all use it, so a network engineer will meet it in real packet captures and real firewall rules.

Interviewers ask about QUIC because it tests whether you understand *why* TCP behaves as it does. They often ask: why build on UDP, what head-of-line blocking QUIC removes and what it does not remove, why 0-RTT is risky, and what happens when a firewall blocks UDP port 443.

## Simple Explanation

**TCP is a single-lane road.** All cars from all families share one lane, and they must arrive in order. If one car breaks down, every car behind it stops, even cars going to a different house.

**QUIC is a road with many lanes.** Each family has its own lane. A broken car stops its own lane only. The other lanes keep moving.

**Where the analogy stops:** the lanes are not physical. All QUIC streams share the same UDP path, the same bandwidth and the same congestion control. If the whole road is full, all lanes slow down together.

**The connection ID is like your phone number.** A TCP connection is like post sent to your home address. Move house and the post is lost. QUIC is like a phone number: you move to a new city and people still reach you, because they call the number and not the building.

**Where the analogy stops:** a QUIC connection uses many connection IDs and changes them on purpose, for privacy. A real phone number stays the same.

**UDP is the plain delivery van.** The van carries a sealed box and promises nothing. QUIC is the logistics company inside the box: it numbers parcels, checks arrival, resends lost ones, and locks everything. The road network only sees vans.

**Where the analogy stops:** real logistics firms own their process end to end. In QUIC, both endpoints run the logic, and nothing in the middle helps.

## How It Works

### What QUIC is, in layers

QUIC replaces TCP, and it also takes over the record protection job of TLS. It sits on top of UDP. HTTP/3 sits on top of it. For the HTTP side, see [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

```text
     HTTP/2 stack                HTTP/3 stack
  +----------------+          +----------------+
  |    HTTP/2      |          |    HTTP/3      |
  +----------------+          +----------------+
  |    TLS 1.3     |          |  QUIC          |
  +----------------+          |  - streams     |
  |    TCP         |          |  - reliability |
  +----------------+          |  - congestion  |
  |    IP          |          |  - TLS 1.3     |
  +----------------+          +----------------+
                              |    UDP         |
                              +----------------+
                              |    IP          |
                              +----------------+
```

A short history helps, because interviewers sometimes ask:

- Jim Roskind designed the first QUIC at Google around 2012. Chrome started experiments in 2013. This early version is often called gQUIC.
- The IETF, the group that writes Internet standards, formed a working group in 2016.
- The core standard, RFC 9000, was published in May 2021. HTTP/3 (RFC 9114) followed in June 2022.
- In the standard, QUIC is a name. It is not an acronym, although the early Google version was called "Quick UDP Internet Connections".

### Why build on UDP

UDP adds only ports, a length and a checksum to IP. It promises nothing. So why build a reliable protocol on it?

There were three other options, and each one failed a practical test.

| Option | Why it does not work well |
|---|---|
| Change TCP | Middleboxes drop or strip unknown TCP options. Kernels update slowly. New features such as TCP Fast Open (RFC 7413) deployed poorly for this reason. |
| Create a new IP protocol | NAT devices and firewalls only understand TCP and UDP well. SCTP is a real example: a good protocol that rarely crosses the public Internet. |
| Use UDP | Almost every network already passes UDP, because DNS and other services need it. |

A middlebox is any device in the path that does more than forward packets: a NAT router, firewall, proxy or load balancer.

UDP gives QUIC two gifts:

- **It passes through networks.** To a router, QUIC looks like normal UDP traffic.
- **It can live in user space.** User space is where normal programs run, outside the operating system kernel. A browser can ship a new QUIC version in its next update, with no OS change.

**Analogy:** you want to send a new kind of parcel, but the post office only accepts standard envelopes. So you fit your parcel inside a standard envelope. **Where it stops:** some networks limit or block UDP, so the "standard envelope" is not accepted everywhere. See [Middleboxes and firewalls](#middleboxes-and-firewalls).

**Important:** UDP's lack of reliability does not make QUIC unreliable. QUIC builds its own acknowledgements, retransmission, flow control and congestion control on top. For the basics of UDP itself, see [UDP in Depth](07-udp.md).

### Datagram, packet, frame, stream

Four words are easy to mix up. Learn them now.

- **UDP datagram:** the unit IP and UDP carry.
- **QUIC packet:** an encrypted unit with a QUIC header. One UDP datagram can hold several QUIC packets. This is called coalescing, and it is common during the handshake.
- **Frame:** a small typed message inside a QUIC packet. One packet can hold many frames, and they can belong to different streams.
- **Stream:** an ordered flow of bytes, like one file transfer. Its data travels in STREAM frames.

```text
+----------+-----------+-------------------------------------------+
| IP hdr   | UDP hdr   | QUIC packet                               |
| 20 bytes | 8 bytes   |                                           |
+----------+-----------+-------------------------------------------+
                         |
                         v
          +-------------+---------------------------------+
          | QUIC header | encrypted payload               |
          +-------------+---------------------------------+
                         |
                         v
          +---------+------------------+------------------+
          | ACK     | STREAM id=0      | STREAM id=4      |
          | frame   | offset=0 len=500 | offset=0 len=600 |
          +---------+------------------+------------------+
```

### Packet headers

QUIC has two header forms. The first bit tells them apart.

**Long header:** used during the handshake. It carries version and connection ID details, because the two sides are still agreeing on them.

```text
Long header (RFC 9000)

Field               Size          Meaning
------------------  ------------  -------------------------------
Header Form         1 bit         1 = long header
Fixed Bit           1 bit         1 in QUIC v1
Long Packet Type    2 bits        Initial, 0-RTT, Handshake, Retry
Type-Specific Bits  4 bits        Depends on packet type
Version             32 bits       0x00000001 = QUIC v1
DCID Length         8 bits        Destination Connection ID length
DCID                0-160 bits    Destination Connection ID
SCID Length         8 bits        Source Connection ID length
SCID                0-160 bits    Source Connection ID
Payload             variable      Depends on packet type
```

**Short header:** used after the handshake. It is small, so normal data has low overhead.

```text
Short header, also called a 1-RTT packet (RFC 9000)

Field               Size          Meaning
------------------  ------------  -------------------------------
Header Form         1 bit         0 = short header
Fixed Bit           1 bit         1 in QUIC v1
Spin Bit            1 bit         Optional RTT signal for observers
Reserved Bits       2 bits        Must be 0
Key Phase           1 bit         Which traffic key is in use
Packet Number Len   2 bits        Packet number takes 1-4 bytes
DCID                0-160 bits    Destination Connection ID
Packet Number       8-32 bits     Encrypted, unique in its space
Payload             variable      Encrypted frames, then a tag
```

Compare this with TCP. TCP shows sequence numbers, ACK numbers, window size and flags to everyone on the path. QUIC encrypts almost all of this. An observer sees the IP addresses, the UDP ports, a few header bits and the connection ID. The short header has no source connection ID and no length byte. The receiver knows the length because it chose that ID itself.

### Streams without head-of-line blocking

A stream is a lightweight, ordered byte flow inside a connection. Opening one needs no handshake: you just send data with a new stream ID.

The two lowest bits of the stream ID show who opened the stream and which direction it runs.

| Lowest 2 bits | Type | Example IDs |
|---|---|---|
| 0b00 | Client-opened, two-way | 0, 4, 8, 12 |
| 0b01 | Server-opened, two-way | 1, 5, 9, 13 |
| 0b10 | Client-opened, one-way | 2, 6, 10, 14 |
| 0b11 | Server-opened, one-way | 3, 7, 11, 15 |

In HTTP/3, each request uses a new client-opened two-way stream. That is why Wireshark shows requests on stream 0, 4, 8 and so on.

Each STREAM frame says which stream it belongs to and the byte offset of its data inside that stream. So the receiver can rebuild each stream on its own.

```text
Sender has 3 files on one connection. Packet 11 is lost.

Packet 10: STREAM 0 offset 0     arrives
Packet 11: STREAM 4 offset 0     LOST
Packet 12: STREAM 8 offset 0     arrives
Packet 13: STREAM 0 offset 1100  arrives

TCP (HTTP/2): all data is one byte line. Bytes after the gap wait.

  received in TCP buffer: [10] [gap] [12] [13]
  given to application:   [10]  ... nothing more until gap is filled

QUIC (HTTP/3): each stream has its own order.

  stream 0: [10] [13]  delivered now
  stream 4: [gap]      waits for retransmission
  stream 8: [12]       delivered now
```

QUIC also lets one side cancel a stream without killing the connection. RESET_STREAM says "I stop sending on this stream." STOP_SENDING says "please stop sending on this stream." For example, a browser can cancel an image when you click away, and the other streams continue.

**What QUIC does not remove.** This is a favourite interview trap.

- Order is still kept *inside* a stream. A lost packet still stalls its own stream.
- A lost packet with frames from 3 streams delays all 3.
- All streams share congestion control. Heavy loss slows all of them.
- All streams share connection-level flow control. One greedy stream can use up the shared limit.
- Header compression can create blocking of its own. HTTP/2's HPACK depends on strict order, so HTTP/3 uses QPACK (RFC 9204), which is built to reduce this risk. A stream can still choose to wait for QPACK table updates.

### TLS 1.3 integrated

With TCP, TLS is a separate layer. TCP finishes its own handshake, then TLS starts. QUIC combines both into one handshake (RFC 9001).

Four key points:

1. **TLS messages travel in CRYPTO frames.** TLS gives QUIC the handshake messages and keys. There is no TLS record layer. The record layer is the part of TLS that wraps and encrypts data, and QUIC does that job itself.
2. **QUIC protects each packet.** It uses an AEAD cipher, such as AES-GCM or ChaCha20-Poly1305. AEAD means "authenticated encryption with associated data": it hides the data and also proves nobody changed it. The header is authenticated too, so a device in the path cannot change it without being detected.
3. **QUIC also hides the packet number.** This is called header protection. A mask made from part of the encrypted data covers the packet number and some first-byte bits.
4. **Encryption is mandatory.** Standard QUIC has no plain-text mode. It requires TLS 1.3 or later, and it requires ALPN. ALPN (Application-Layer Protocol Negotiation) is a TLS extension that names the application protocol. HTTP/3 uses the value `h3`.

The two sides exchange transport parameters inside the TLS handshake, in the `quic_transport_parameters` extension. These are settings such as the flow control limits and the idle timeout. Because they sit inside the TLS handshake, an attacker cannot change them unnoticed.

QUIC uses separate keys for separate stages. Each stage has its own packet number space, a separate numbering sequence.

| Packet type | Keys come from | Packet number space | Who can read it |
|---|---|---|---|
| Initial | Public salt plus client's first DCID | Initial | Anyone who sees the packet |
| Handshake | TLS key exchange | Handshake | Only the two endpoints |
| 0-RTT | Earlier session (pre-shared key) | Application | Only the two endpoints |
| 1-RTT | TLS key exchange | Application | Only the two endpoints |

Look at the first row. **Initial packets are obfuscated, not secret.** The salt is published in the RFC and the client's DCID is in the packet, so anyone can derive the keys. Wireshark does this automatically. It means the ClientHello, and usually the server name (SNI), can be read by a passive observer, unless Encrypted Client Hello (ECH) is used. For TLS details, see [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

### The 1-RTT handshake

"1-RTT" means the client can send request data after one round trip.

```text
Client                                                 Server
  |                                                      |
  | Initial: CRYPTO(ClientHello), PADDING                |
  |----------------------------------------------------->|
  |                                                      |
  | Initial: ACK, CRYPTO(ServerHello)                    |
  | Handshake: CRYPTO(EncryptedExtensions, Certificate,  |
  |            CertificateVerify, Finished)              |
  |<-----------------------------------------------------|
  |                                                      |
  | Initial: ACK                                         |
  | Handshake: ACK, CRYPTO(Finished)                     |
  | 1-RTT: STREAM 0 (HTTP request)                       |
  |----------------------------------------------------->|
  |                                                      |
  | 1-RTT: HANDSHAKE_DONE, ACK, STREAM 0 (response)      |
  |<-----------------------------------------------------|
```

Step by step:

1. The client picks a random Destination Connection ID of at least 8 bytes.
2. The client sends a ClientHello in an Initial packet. It pads the UDP datagram to at least 1200 bytes.
3. The server replies with its ServerHello, then its certificate and Finished message in Handshake packets. These often share one datagram.
4. The client checks the certificate, sends its own Finished message, and sends its first request right away.
5. The server sends HANDSHAKE_DONE. The handshake is now confirmed, and the client can drop handshake keys.

**Why at least 1200 bytes?** There are two reasons. It proves the path can carry a reasonable packet size. It also limits abuse, as the next section shows.

### Anti-amplification

UDP has no handshake, so an attacker can fake the source IP address. Imagine a 100-byte request that makes a server send 10,000 bytes to a victim. That is an amplification attack.

QUIC's rule: until the server has validated the client's address, it may send at most **3 times** the bytes it has received from that address. A server that received 1200 bytes may send 3600 bytes.

**Analogy:** a shop that will not ship goods to a new address until the customer proves they live there. **Where it stops:** the shop refuses all shipping, but QUIC allows a small amount, because it needs to reply at all.

The server treats the address as validated when it can process a Handshake packet from the client, or when the client presents a valid token. A large certificate chain may not fit inside 3600 bytes. Then the handshake needs one more round trip. This is one reason certificate size matters for QUIC, and one reason certificate compression (RFC 8879) is useful.

**Retry** gives a server under load a stronger option. The server sends a Retry packet with a token, and keeps no state. The client repeats its Initial packet with that token, which proves it can receive at that address. The cost is one extra round trip.

```text
Client                                      Server
  | Initial (no token)                        |
  |------------------------------------------>|
  |                 Retry (token), no state   |
  |<------------------------------------------|
  | Initial (token)                           |
  |------------------------------------------>|
  |          normal handshake continues       |
```

### 0-RTT resumption

"0-RTT" means the client sends application data in its very first flight, with no waiting.

It only works if the client has talked to this server before. In the earlier connection, the server gave the client a TLS session ticket. Later, the client derives keys from that ticket and sends encrypted data at once.

```text
Client                                                 Server
  |                                                      |
  | Initial: CRYPTO(ClientHello with pre-shared key)     |
  | 0-RTT: STREAM 0 (GET /index.html)                    |
  |----------------------------------------------------->|
  |                                                      |
  | Initial and Handshake: rest of TLS handshake         |
  | 1-RTT: STREAM 0 (response)                           |
  |<-----------------------------------------------------|
```

**Analogy:** a regular customer calls the cafe and says "my usual" while walking in. No menu needed. **Where it stops:** the cafe knows the caller's voice. A replay attacker can repeat the same encrypted order without knowing what it says, so the server may see the order twice.

0-RTT has real risks:

- **Replay.** An attacker can capture the 0-RTT data and send it again. The server may process it twice. TLS and QUIC do not fully prevent this. It is safe for `GET /logo.png`. It is dangerous for `POST /transfer-money`.
- **Weaker forward secrecy.** Forward secrecy means old traffic stays safe even if a long-term key leaks later. 0-RTT data depends only on the session ticket secret. If that secret leaks, recorded 0-RTT data can be decrypted.

How systems reduce the risk:

- Servers can refuse 0-RTT. The client then sends the same data again as 1-RTT data.
- Clients should use 0-RTT only for safe, repeatable requests.
- An HTTP server can answer `425 Too Early` (RFC 8470), which tells the client to try again after the handshake.
- Servers can keep a list of tickets already used, or allow each ticket only once.

The server must remember the flow control limits it promised. The client reuses the remembered transport parameters for its 0-RTT data, and the server must not lower them below those values if it accepts 0-RTT.

### Connection IDs and migration

Each side picks the connection IDs that the *other* side must put in the DCID field. A UDP receiver uses the DCID to find the right connection. The IP address and port are no longer the identity of the connection.

This is why QUIC survives a network change.

There are two kinds of address change:

- **NAT rebinding (passive).** A NAT device silently changes your public port. The client may not even know. The server sees a new address with the same CID, then checks the new path.
- **Active migration.** The client knows it moved, for example from Wi-Fi to mobile data. It uses a new connection ID on the new path.

Why a new CID? If the same ID appeared on both networks, an observer could link your two locations. RFC 9000 says an endpoint must not reuse a connection ID when it sends from more than one local address. The server gives spare IDs ahead of time in NEW_CONNECTION_ID frames, and old ones are given up with RETIRE_CONNECTION_ID.

The server must also confirm that the new address is real. Otherwise, an attacker could trick it into flooding a victim.

```text
Client (new network)                               Server
  | 1-RTT: new DCID, data                            |
  |------------------------------------------------->|
  |                                                  |
  | PATH_CHALLENGE (8 random bytes)                  |
  |<-------------------------------------------------|
  |                                                  |
  | PATH_RESPONSE (same 8 bytes)                     |
  |------------------------------------------------->|
  |                                                  |
  | path validated, normal data continues            |
  |<------------------------------------------------>|
```

Details worth knowing:

- Only the client starts migration in QUIC v1. A server can offer a `preferred_address` in the handshake, and the client may move to it.
- Migration must wait until the handshake is confirmed.
- A server can forbid active migration with the `disable_active_migration` transport parameter.
- After validation, the sender resets its RTT estimate and congestion state for the new path, because the new path may be very different. If only the port changed, it may keep them.
- A zero-length connection ID is allowed. Then the receiver falls back to using the address and port, and migration does not work well.

**Load balancer problem.** A simple load balancer hashes the four values (IP and port for each side) to pick a backend server. After migration the hash changes, and packets can reach the wrong server. A QUIC-aware load balancer routes on the connection ID instead. A common design puts a server identifier inside the CID. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

### Loss recovery

QUIC's loss recovery (RFC 9002) learned from decades of TCP. The core idea is the same as in [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md): acknowledge data and resend what is missing. The details are cleaner.

**1. Packet numbers never repeat.** Every packet gets a new number, even a retransmission. QUIC resends *frames*, not packets.

TCP has a problem here. If TCP resends a segment and then gets an ACK, it cannot tell if the ACK is for the original or the retry. This is retransmission ambiguity. It makes RTT measurement hard, and Karn's algorithm and timestamps exist to work around it. QUIC does not have this problem. Each ACK names one exact packet.

**2. ACK frames carry many ranges.** An ACK frame lists exact ranges of received packets. TCP SACK has only a few blocks, because the TCP options area is 40 bytes at most. QUIC's ACK frame can describe many gaps.

**3. ACK frames report the delay.** The ACK Delay field says how long the receiver held the packet before it acknowledged. The sender subtracts this and gets a better RTT.

**4. Two loss tests.** A packet is declared lost if an ACK exists for a later packet and one of these is true:

- **Packet threshold:** the packet is at least 3 below the largest acknowledged packet number (`kPacketThreshold` = 3).
- **Time threshold:** it was sent more than 9/8 of the RTT ago.

**5. Probe Timeout (PTO).** If ACKs stop coming, the sender uses a timer:

```text
PTO = smoothed_rtt + max(4 * rttvar, 1 ms) + max_ack_delay
```

- `smoothed_rtt` is the average RTT.
- `rttvar` is how much the RTT varies.
- `max_ack_delay` is the longest the peer says it will delay an ACK (default 25 ms).

Example: smoothed_rtt = 50 ms, rttvar = 5 ms, max_ack_delay = 25 ms.

```text
PTO = 50 + max(20, 1) + 25 = 95 ms
```

If nothing arrives after 95 ms, the sender sends one or two probe packets. Each repeat doubles the timer: 190 ms, then 380 ms. A probe timeout does not by itself declare loss, and it does not cut the send rate. Compare that with TCP, where an RTO collapses the congestion window.

Before any RTT sample exists, RFC 9002 uses an initial RTT of 333 ms. That gives a first PTO of about 1 second.

**6. Congestion control.** RFC 9002 describes a sender based on NewReno, in bytes, but the standard allows others. Real stacks often use CUBIC or BBR. For these algorithms, see [TCP Congestion Control](06-tcp-congestion-control.md).

The RFC's recommended starting window is:

```text
min(10 * max_datagram_size, max(14720, 2 * max_datagram_size))
```

With a 1200-byte datagram size, that is min(12000, 14720) = 12000 bytes.

If loss continues for a long time, the sender declares persistent congestion and drops to the minimum window of 2 datagrams. QUIC also supports ECN (Explicit Congestion Notification), where routers mark packets instead of dropping them. The ACK frame can report these marks.

**Analogy:** in TCP, a resent parcel keeps its old tracking number, so you cannot tell which delivery the receipt is for. In QUIC, each attempt gets a new tracking number. **Where it stops:** the stream offset still identifies the data itself. Only the packet number is new.

### Flow control

Flow control stops a fast sender from filling a slow receiver's memory. QUIC has three limits.

| Limit | Frame that raises it | What it protects |
|---|---|---|
| Bytes per stream | MAX_STREAM_DATA | Buffer for one stream |
| Bytes per connection | MAX_DATA | Total buffer for all streams |
| Number of streams | MAX_STREAMS | Memory for stream state |

The limits are absolute byte offsets, unlike the sliding window in TCP. `MAX_STREAM_DATA = 500000` means "you may send up to byte 500,000 on this stream".

### Closing a connection

QUIC ends a connection in three ways:

- **Immediate close:** a CONNECTION_CLOSE frame, which carries an error code.
- **Idle timeout:** both sides agree a `max_idle_timeout`. If nothing is heard for that long, the connection is dropped silently.
- **Stateless reset:** if a server crashed and lost its state, it cannot read your packets. It can still send a special packet containing a 16-byte token that it gave out earlier. This tells the client the connection is dead.

Stateless reset looks like TCP RST, with one big difference. A device in the path can forge a TCP RST, because TCP has no authentication. It cannot forge a stateless reset, because it does not know the token.

### Middleboxes and firewalls

QUIC changes life for network devices. Here is what an engineer sees in practice.

**What a device can see:**

| Item | TCP plus TLS | QUIC |
|---|---|---|
| IP addresses and ports | Yes | Yes |
| SYN, FIN, RST flags | Yes | No |
| Sequence and ACK numbers | Yes | No |
| Window size | Yes | No |
| Version | n/a | Long header only |
| Connection ID | n/a | Yes |
| Server name (SNI) | Yes, unless ECH | Yes in Initial, unless ECH |
| Server certificate | Hidden in TLS 1.3 | Hidden |

**1. Firewalls often block it.** Many enterprise firewalls block UDP 443. The reason is simple: their TLS inspection works on TCP, and they want traffic to go through it. Several security vendors recommend this practice. When QUIC fails, the browser falls back to TCP, so users usually see no error. See [Firewalls](../09-security/04-firewalls.md).

**2. Fallback is built in.** Browsers try QUIC and TCP in a race, or use QUIC only when they have reason to believe the server supports it. A client learns about HTTP/3 in two common ways:

- An `Alt-Svc` response header (RFC 7838): `Alt-Svc: h3=":443"; ma=86400`. The first visit uses TCP, then later visits can use QUIC. `ma` is the time in seconds the client can remember this.
- A DNS HTTPS record (RFC 9460), which announces `h3` before the first connection.

**3. No connection flags to track.** A stateful firewall sees SYN and FIN for TCP. For QUIC, it only sees a UDP flow. It uses timers to decide when the flow has ended.

**4. NAT timers are shorter for UDP.** NAT routers keep mapping entries for UDP for much less time than for TCP. RFC 4787 asks for at least 2 minutes, but real devices can be much shorter. Linux conntrack, for example, uses 30 seconds by default for a UDP flow with packets in one direction only, and 120 seconds for one with traffic both ways. QUIC clients send a PING frame as a keepalive when they must hold an idle connection open. RFC 9308 suggests that sending about every 30 seconds keeps most middlebox state alive. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

**5. UDP rate limiting.** Some networks and DDoS filters limit UDP because attacks often use it. QUIC can suffer here, and the symptom looks like a slow site, not a blocked one.

**6. Visibility is lower.** Engineers used to measure RTT and loss from TCP headers. QUIC hides those fields. The optional spin bit in the short header gives a passive observer a rough RTT, and an endpoint may turn it off.

**7. Size limits.** QUIC requires a path that can carry at least a 1200-byte UDP payload. A tunnel with too small an MTU breaks it, and the client falls back to TCP. QUIC sets the DF (Don't Fragment) bit on IPv4, because QUIC packets must not be fragmented. It finds a larger size with its own probes. See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).

**Analogy:** a building guard used to read postcards, so he knew who was talking and when they said goodbye. Now everyone uses sealed envelopes. The guard has two choices: let them pass, or ban them. Many companies ban them and make people use postcards again. **Where it stops:** a guard can still see who sent the envelope and when, just as a firewall still sees IPs, ports and timing.

### Comparison with TCP plus TLS

| Feature | TCP + TLS 1.3 | QUIC |
|---|---|---|
| Runs on | IP | UDP over IP |
| Usual location | Kernel (TCP), library (TLS) | User space library |
| Setup, new connection | 2 RTT | 1 RTT |
| Setup, resumed | 1 RTT, or 0 with TCP Fast Open | 0 RTT possible |
| Encryption | Separate layer, TCP header visible | Built in, most of header hidden |
| Multiplexing | Added by HTTP/2 | Native streams |
| HOLB at transport | Yes, across all data | Only inside one stream |
| Connection identity | 4-tuple | Connection ID |
| IP change | Connection breaks | Connection can migrate |
| Packet numbering | Byte sequence, retries reuse it | New number each send |
| ACK information | Cumulative plus limited SACK | Many ACK ranges |
| Firewall support | Universal | Often blocked or limited |
| Typical CPU cost | Lower, strong kernel and NIC offload | Higher |
| Spoofed reset | Possible (RST injection) | Not possible after handshake |
| Ease of change | Slow, kernel and middleboxes | Fast, ships with the app |

TCP plus TLS has not lost. It is simpler to operate, works everywhere, and often uses less CPU. Most sites offer both.

## Example

### Worked example 1: time to first byte

Assume a 100 ms RTT and a server that answers instantly. Each entry shows when the first response byte reaches the client.

| Stack | Client sends request at | First response byte at |
|---|---|---|
| TCP + TLS 1.2 (full) | 300 ms | 400 ms |
| TCP + TLS 1.3 (full) | 200 ms | 300 ms |
| QUIC 1-RTT | 100 ms | 200 ms |
| QUIC 0-RTT | 0 ms | 100 ms |

QUIC with a new connection saves one full round trip over TCP plus TLS 1.3. On a 20 ms fibre link, this saves only 20 ms. On a 200 ms satellite or weak mobile link, it saves 200 ms.

### Worked example 2: a real handshake, packet by packet

Client: 10.0.0.25 port 51000. Server: 198.51.100.10 port 443. RTT: 50 ms.

The client picks a random first DCID `c1a2b3c4d5e6f708` and its own SCID `a1a1a1a1`.

| Datagram | Direction | Bytes | Contents |
|---|---|---|---|
| 1 | C to S | 1200 | Initial pn=0: CRYPTO(ClientHello), PADDING. DCID=c1a2b3c4d5e6f708, SCID=a1a1a1a1 |
| 2 | S to C | 1200 | Initial pn=0: ACK 0, CRYPTO(ServerHello). Handshake pn=0: CRYPTO(EncryptedExtensions, part of Certificate). DCID=a1a1a1a1, SCID=5f0e7d3c2b1a0908 |
| 3 | S to C | 1200 | Handshake pn=1: CRYPTO(rest of Certificate, CertificateVerify, Finished) |
| 4 | C to S | 1200 | Initial pn=1: ACK 0. Handshake pn=0: ACK 0-1, CRYPTO(Finished). 1-RTT pn=0: STREAM 0 (GET /). DCID=5f0e7d3c2b1a0908 |
| 5 | S to C | 1025 | 1-RTT pn=0: ACK 0, HANDSHAKE_DONE, STREAM 0 (response) |

Check the details:

- Datagram 1 and datagram 4 are exactly 1200 bytes, because any client datagram holding an Initial packet must be at least that large.
- The server received 1200 bytes and sent 2400 bytes. That is inside the 3 times limit of 3600 bytes.
- From datagram 2 onward, the client uses the server's chosen ID as DCID. The first random DCID has done its job.
- Packet numbers restart at 0 in each space. That is why datagram 4 has Handshake pn=0 and 1-RTT pn=0, side by side.

### Worked example 3: loss on one stream

The server sends 5 packets. Stream 0 and stream 8 go well, but packet 11 is lost.

```text
pn 10: STREAM 0, offset 0,    length 1100   received
pn 11: STREAM 4, offset 0,    length 1100   LOST
pn 12: STREAM 8, offset 0,    length 1100   received
pn 13: STREAM 0, offset 1100, length 1100   received
pn 14: STREAM 8, offset 1100, length 1100   received
```

The client sends this ACK frame:

```text
Largest Acknowledged = 14
First ACK Range      = 2    covers 12, 13, 14
Gap                  = 0    one packet missing: 11
ACK Range Length     = 0    covers 10
```

The sender then works it out:

1. Largest acknowledged is 14, and packet 11 is not listed.
2. 14 - 11 = 3, which meets the packet threshold of 3.
3. Packet 11 is lost. The sender looks at what it held: STREAM 4, offset 0, length 1100.
4. It puts that frame in a **new** packet, pn 15. The number 11 is never used again.

Meanwhile, the client has already given all of stream 0 and stream 8 to the application. Only stream 4 waited. Over TCP, the same loss would hold up all the bytes after the gap.

### Worked example 4: phone leaves Wi-Fi

A phone is downloading a file. Server: 198.51.100.10:443.

```text
On Wi-Fi: phone 10.0.0.25, NAT public 203.0.113.20:40001
  DCID in use: 5f0e7d3c2b1a0908
  Server has already issued a spare ID: 77aa88bb99cc0011

Phone moves to mobile data: public 203.0.113.77:52344

1. Client sends 1-RTT packet from 203.0.113.77:52344
   DCID = 77aa88bb99cc0011 (new ID, no link to old path)
2. Server finds the connection from that DCID.
   The 4-tuple is new, so the server must validate it.
3. Server sends PATH_CHALLENGE data=9f3c12ab44de0177
4. Client sends PATH_RESPONSE  data=9f3c12ab44de0177
5. Server resets RTT and congestion state for the new path.
6. Download continues on the same connection and streams.
```

Until step 4, the server limits data to the new address to 3 times what it received from it. With TCP, the same move would end the connection, and the application would need to reconnect and resume.

### Commands: check HTTP/3 from a client

`example.com` is a placeholder here. Use a server you know supports HTTP/3. Your curl must also be built with HTTP/3 support. Check for `HTTP3` in the Features line of `curl -V` (Linux) or `curl.exe -V` (Windows).

Linux:

```bash
# Ask for HTTP/3 only, with no fallback
curl --http3-only -sI https://example.com/
```

Output:

```text
HTTP/3 200
server: nginx
content-type: text/html
alt-svc: h3=":443"; ma=86400
```

Compare timing on a 50 ms RTT path:

```bash
fmt='http=%{http_version} ttfb=%{time_starttransfer}\n'
curl --http2 -so /dev/null -w "$fmt" https://example.com/
curl --http3-only -so /dev/null -w "$fmt" https://example.com/
```

Output:

```text
http=2 ttfb=0.157
http=3 ttfb=0.107
```

The gap is about one RTT. Real numbers vary with server time and DNS.

Find out if a site announces HTTP/3 in DNS:

```bash
dig +short HTTPS example.com
```

Output:

```text
1 . alpn="h3,h2" ipv4hint=198.51.100.10
```

Windows PowerShell: `curl.exe` works the same way when built with HTTP/3 support. Be careful with this one:

```powershell
# This tests TCP only. It says NOTHING about QUIC.
Test-NetConnection example.com -Port 443
```

In a browser, open Developer Tools, Network tab, and turn on the Protocol column. HTTP/3 shows as `h3`.

### Commands: see QUIC on the wire

Linux, capturing on the client:

```bash
sudo tcpdump -ni eth0 -c 5 udp port 443
```

Output (these lines match worked example 2):

```text
10:15:02.100123 IP 10.0.0.25.51000 > 198.51.100.10.443: UDP, length 1200
10:15:02.150410 IP 198.51.100.10.443 > 10.0.0.25.51000: UDP, length 1200
10:15:02.150662 IP 198.51.100.10.443 > 10.0.0.25.51000: UDP, length 1200
10:15:02.151024 IP 10.0.0.25.51000 > 198.51.100.10.443: UDP, length 1200
10:15:02.201538 IP 198.51.100.10.443 > 10.0.0.25.51000: UDP, length 1025
```

Newer tcpdump versions may decode these lines as QUIC instead of plain UDP. Either way, there is no SYN and no FIN. See [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

Wireshark display filters ([Wireshark](../12-troubleshooting-and-tools/07-wireshark.md)):

```text
quic                                     all QUIC packets
quic.long.packet_type == 0               Initial packets
tls.handshake.extensions_server_name     ClientHello with a server name
```

To see the encrypted parts, set the `SSLKEYLOGFILE` environment variable before you start the browser or curl. Then give that key log file to Wireshark in its TLS settings. Keep this only on test machines, because that file can decrypt your traffic.

### Commands: enable HTTP/3 on a server

nginx (version 1.25 or later, built with QUIC support):

```text
server {
    listen 443 quic reuseport;
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/ssl/example.com.crt;
    ssl_certificate_key /etc/ssl/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

Open UDP 443 on the host. Opening TCP 443 is not enough.

```bash
# Linux with ufw
sudo ufw allow 443/udp
# Check that something listens
sudo ss -uln 'sport = :443'
```

Output:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
UNCONN 0      0            0.0.0.0:443       0.0.0.0:*
```

```powershell
# Windows
New-NetFirewallRule -DisplayName "Allow QUIC" -Direction Inbound `
    -Protocol UDP -LocalPort 443 -Action Allow
Get-NetUDPEndpoint -LocalPort 443
```

Output:

```text
LocalAddress  LocalPort
------------  ---------
0.0.0.0             443
```

Cloud firewalls and security groups also need a UDP 443 rule.

### Commands: block QUIC and force fallback

Cisco IOS, on a router interface facing the LAN:

```text
ip access-list extended BLOCK-QUIC
 remark Force browsers back to TCP 443
 deny   udp any any eq 443
 permit ip any any
!
interface GigabitEthernet0/1
 ip access-group BLOCK-QUIC in
```

Linux gateway:

```bash
sudo iptables -I FORWARD -p udp --dport 443 -j REJECT
```

Test the fallback from a client:

```bash
curl --http3 -sI https://example.com/ | head -1
```

Output, because `--http3` is allowed to fall back:

```text
HTTP/2 200
```

`--http3-only` would fail here. REJECT sends a clear error, while DROP stays silent. A clear error can help a client give up on QUIC sooner, but the exact behaviour depends on the browser.

## Going Deeper

### The RFC family

| RFC | Topic |
|---|---|
| RFC 8999 | Version-independent properties of QUIC (the invariants) |
| RFC 9000 | The QUIC transport (core) |
| RFC 9001 | Using TLS to secure QUIC |
| RFC 9002 | Loss detection and congestion control |
| RFC 9114 | HTTP/3 |
| RFC 9204 | QPACK, HTTP/3 header compression |
| RFC 9218 | Extensible priorities for HTTP |
| RFC 9221 | Unreliable datagram extension |
| RFC 9250 | DNS over QUIC |
| RFC 9287 | Greasing the QUIC bit |
| RFC 9297 | HTTP datagrams and the capsule protocol |
| RFC 9298 | Proxying UDP in HTTP (part of MASQUE) |
| RFC 9308 | Applicability guide for QUIC |
| RFC 9312 | Manageability guide for network operators |
| RFC 9368 | Compatible version negotiation |
| RFC 9369 | QUIC version 2 |

For an interview, remember the core four: 8999, 9000, 9001, 9002.

### Invariants, versions and greasing

RFC 8999 lists the few things that will stay the same in *every* QUIC version: the first bit, the version field position, and connection ID layout rules. Everything else may change. Network devices should not depend on anything else.

Version negotiation works like this. If a server does not support the client's version, it sends a Version Negotiation packet. This packet has Version = 0 and lists versions the server does support.

QUIC v2 (RFC 9369, version number 0x6b3343cf) is nearly identical to v1. It changes the packet type values and some key constants. Its main purpose is to keep the version field in active use, so that middleboxes cannot hard-code v1.

This is an anti-ossification method called greasing. Greasing means sending deliberately odd but allowed values, so that devices must learn to ignore them. QUIC reserves versions of the form `0x?a?a?a?a` for this. RFC 9287 does the same to the Fixed Bit, so that devices do not rely on it being 1.

**Analogy:** a company regularly tests its fire doors so they do not rust shut. **Where it stops:** a door has no intent. A middlebox vendor can still write bad rules, so greasing lowers the risk and does not remove it.

### Variable-length integers

Most QUIC fields use a compact integer format. The first 2 bits of the first byte give the total length.

| First 2 bits | Total bytes | Usable bits | Largest value |
|---|---|---|---|
| 00 | 1 | 6 | 63 |
| 01 | 2 | 14 | 16,383 |
| 10 | 4 | 30 | 1,073,741,823 |
| 11 | 8 | 62 | 4,611,686,018,427,387,903 |

So packet numbers, stream IDs and offsets can reach 2^62 - 1. Packet numbers cannot wrap in any realistic connection, unlike TCP's 32-bit sequence numbers. On the wire, packet numbers are cut down to 1-4 bytes, and the receiver rebuilds the full value.

### Flow control math

Suppose a receiver sets these limits:

- Per stream limit: 100,000 bytes
- Connection limit: 250,000 bytes

A sender with 3 big streams sends 100,000 bytes on stream 0, and 100,000 bytes on stream 4. That is 200,000 bytes. Stream 8 can only send 50,000 bytes, because the connection limit is reached. All three streams stall until the receiver reads data and sends MAX_DATA or MAX_STREAM_DATA.

This is why a poor limit can hurt speed. The window must be at least the bandwidth-delay product, as in TCP. At 100 Mbps and 50 ms RTT, that is 100,000,000 / 8 * 0.05 = 625,000 bytes.

### Key updates and error codes

TLS 1.3 over TCP changes keys with a KeyUpdate message. QUIC does not use that message. It flips the Key Phase bit in the short header. The receiver sees the change and moves to the next key.

TLS alerts become QUIC errors in the range 0x0100-0x01ff. For example, a TLS `certificate_unknown` alert (46, or 0x2e) appears as CONNECTION_CLOSE with error 0x012e.

### Frame types worth recognising

| Type | Frame | Purpose |
|---|---|---|
| 0x00 | PADDING | Fill space |
| 0x01 | PING | Ask for an ACK, keep the path alive |
| 0x02-0x03 | ACK | Report received packets |
| 0x04 | RESET_STREAM | Abandon sending on a stream |
| 0x05 | STOP_SENDING | Ask the peer to stop a stream |
| 0x06 | CRYPTO | Carry TLS handshake data |
| 0x07 | NEW_TOKEN | Give a token for future address validation |
| 0x08-0x0f | STREAM | Carry application data |
| 0x10 | MAX_DATA | Raise the connection limit |
| 0x11 | MAX_STREAM_DATA | Raise a stream limit |
| 0x12-0x13 | MAX_STREAMS | Raise the stream count limit |
| 0x18 | NEW_CONNECTION_ID | Offer a spare CID |
| 0x19 | RETIRE_CONNECTION_ID | Stop using a CID |
| 0x1a | PATH_CHALLENGE | Test a path |
| 0x1b | PATH_RESPONSE | Answer a path test |
| 0x1c-0x1d | CONNECTION_CLOSE | End the connection |
| 0x1e | HANDSHAKE_DONE | Confirm the handshake to the client |

### Other users of QUIC

QUIC is a general transport. HTTP/3 is only the best-known user.

- **DNS over QUIC (DoQ):** RFC 9250, UDP port 853. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).
- **SMB over QUIC:** Windows file sharing on UDP 443, without a VPN.
- **MASQUE:** tunnels UDP or IP inside HTTP/3. RFC 9298 is part of this work. Apple's iCloud Private Relay is a well-known real system that uses QUIC-based proxying.
- **QUIC datagrams:** RFC 9221 adds a DATAGRAM frame for unreliable, encrypted data. It suits games and live media that prefer new data over resent old data.
- **WebTransport and Media over QUIC:** these are newer work in the IETF. Check the current status before you quote them.
- **Multipath QUIC:** this work lets one connection use two paths at once, such as Wi-Fi and mobile data. It extends the single-active-path model of RFC 9000. Check its current status before you quote it.

### CPU cost and tuning

QUIC often uses more CPU than TCP for the same data. Three things cause this:

- Packets cross between kernel and user space, often one by one.
- NICs and kernels have years of TCP offload, such as TSO. Much of this is not available for encrypted QUIC.
- Every packet is encrypted separately.

Stacks reduce this cost. On Linux, they use UDP GSO and GRO (batching many packets into one kernel call), and `sendmmsg` and `recvmmsg`. A server under load can also lose packets simply because the UDP socket buffer is full. The fix is a bigger limit:

```bash
# Linux: check drops, then raise the buffer limit (example value)
netstat -su | grep -i 'receive buffer'
sudo sysctl -w net.core.rmem_max=7500000
```

There is one more trade-off. Every application brings its own QUIC stack, so bugs must be fixed in every app, not in one kernel.

Well-known stacks include quiche (Cloudflare, Rust), quiche (Google, C++), MsQuic (Microsoft), mvfst (Meta), quic-go (Go), quinn (Rust), ngtcp2 (C) and s2n-quic (AWS). nginx and HAProxy include their own support.

### Debugging with qlog

Packet captures are hard to use when almost everything is encrypted. Many stacks can write qlog, a structured event log from inside the endpoint, showing RTT estimates, loss events and flow control. The qvis tool draws these logs as charts. A qlog is the first thing to ask for when QUIC performance looks wrong.

### Security summary

| Threat | QUIC defence |
|---|---|
| Reflection and amplification | 3 times limit, 1200-byte minimum, Retry |
| Address spoofing | Handshake check, Retry token, path validation |
| Injected resets | Authenticated packets, stateless reset token |
| Header tampering | AEAD protects the header too |
| Tracking across networks | New connection ID on each path |
| 0-RTT replay | Not fully solved. Limit to safe requests. |
| Handshake interference | Not fully solved. See below. |

Initial packets use public keys, so an on-path attacker can forge one and disturb the handshake. TLS still protects the real result: a forger cannot complete a false handshake, but can make it fail. Full-strength protection starts with Handshake packets. For attack types in general, see [Common Network Attacks](../09-security/03-common-network-attacks.md).

## Common Mistakes

- "QUIC is unreliable because it uses UDP." QUIC adds its own reliability. UDP is only the carrier.
- "QUIC and HTTP/3 are the same thing." QUIC is the transport. HTTP/3 is one application on top of it. DNS and SMB also use QUIC.
- "QUIC removes all head-of-line blocking." It removes blocking *between* streams at the transport. Order inside each stream still holds, and streams still share congestion and flow control limits.
- "Everything in QUIC is encrypted." Initial packets use keys anyone can derive, so SNI is often readable. IPs, ports and connection IDs are visible.
- "Every QUIC connection is 0-RTT." The first connection takes 1 RTT. 0-RTT needs an earlier session, and the server can refuse it.
- "0-RTT is safe for any request." 0-RTT data can be replayed. Use it only for safe, repeatable requests.
- "A retransmitted packet keeps its number." QUIC resends the frame in a new packet with a new number.
- "Blocking UDP 443 breaks the web." Normally, browsers fall back to TCP 443. The user sees a working site, sometimes with a small delay.
- "Migration just works everywhere." It needs server support, and a load balancer that can route on connection ID. Only the client starts it in v1.
- "Opening port 443 is enough." QUIC needs UDP 443, and TCP 443 alone does not open it. The server must also announce HTTP/3 through Alt-Svc or a DNS HTTPS record.
- "Test-NetConnection passed, so QUIC works." That command tests TCP only.
- "QUIC is always faster." The gain is largest on high-latency or lossy paths. On a clean, fast, low-latency link the difference can be small, and CPU use is often higher.
- "NAT does not matter for QUIC." UDP mappings expire quickly, so long idle connections need keepalives, and a NAT port change can cause rebinding.

## Interview Questions

**Q: Why was QUIC built on UDP instead of changing TCP?**

A: TCP is hard to change. Middleboxes block or strip unknown TCP options, and TCP sits in the kernel, so updates take years. A new IP protocol would be dropped by NATs and firewalls. UDP already crosses almost every network, and it lets QUIC run in user space, so applications can ship updates quickly. QUIC then adds reliability and congestion control on top.

**Q: How does QUIC solve head-of-line blocking, and what blocking remains?**

A: TCP has one ordered byte stream, so one lost packet holds back everything behind it. QUIC has many streams, each with its own offsets. A lost packet delays only the streams whose frames were inside that packet. Blocking remains inside a stream, and streams still share congestion control and connection flow control. QPACK can also add a small blocking risk.

**Q: Walk me through a QUIC handshake.**

A: The client sends an Initial packet with a TLS ClientHello in a CRYPTO frame, padded to at least 1200 bytes. The server replies with Initial carrying ServerHello, and Handshake packets carrying the certificate and Finished. The client sends Finished and can send its first request in a 1-RTT packet in the same flight. The server sends HANDSHAKE_DONE. So one round trip gives both transport setup and encryption.

**Q: What is 0-RTT and what is the risk?**

A: A returning client uses a session ticket from an earlier connection and sends encrypted data in its first flight. The risk is replay: an attacker can capture and resend that data, so a server may process it twice. 0-RTT data also has weaker forward secrecy. So I would use it only for safe, repeatable requests, and I would let the server answer 425 Too Early.

**Q: Why must the first client datagram be at least 1200 bytes?**

A: It helps in two ways. It proves the path can carry a 1200-byte payload. It also supports the anti-amplification limit: the server may send only 3 times what it received until the client's address is validated. A larger first packet gives the server room for its reply, and it raises the cost of attacks.

**Q: How does connection migration work?**

A: A QUIC connection is identified by a connection ID, not by IP and port. When the client moves, it sends from the new address using a spare connection ID. Using a new ID stops observers from linking the two paths. The server checks the new path with PATH_CHALLENGE, and the client echoes it in PATH_RESPONSE. The sender then resets congestion state for that path. The streams keep running.

**Q: How does QUIC loss detection differ from TCP?**

A: QUIC packet numbers never repeat, so a retransmission gets a new number and there is no retransmission ambiguity. It resends frames, not packets. ACK frames carry many ranges and report the delay. A packet is lost if it is 3 or more below the largest ACKed packet, or if it is too old by a time threshold. When ACKs stop, a Probe Timeout sends probes, and it does not collapse the congestion window on its own.

**Q: What does a firewall see in QUIC traffic, and why do companies block it?**

A: It sees IPs, UDP ports, the version on long headers, connection IDs, and the Initial packet contents, including SNI unless ECH is used. It does not see flags, sequence numbers or application data. Companies block UDP 443 because their TLS inspection tools usually work on TCP. Browsers then fall back to HTTP/2 over TCP, so users normally do not notice.

**Q: A user says a site is slow only on HTTP/3. How would you debug it?**

A: First I would compare with `curl --http2` and `curl --http3-only`, and check for loss with tcpdump. Then I would look for UDP rate limiting on the path, a low MTU, and short NAT timers. On the server I would check UDP receive buffer drops and CPU. If I own the endpoints, qlog shows RTT, loss and flow control limits. As a quick workaround, removing the Alt-Svc header sends clients back to TCP.

**Q: What breaks when you put QUIC behind a load balancer?**

A: A normal layer 4 load balancer hashes IP and port. After migration or a NAT rebind, the hash changes and packets can reach the wrong backend, which has no connection state. The fix is to route on the connection ID, usually by encoding a server identifier in the ID. Stateless reset and Retry tokens also need shared keys across servers.

**Q: What are the main QUIC RFCs?**

A: RFC 9000 is the transport. RFC 9001 covers TLS integration. RFC 9002 covers loss detection and congestion control. RFC 8999 lists the invariants that stay the same across versions. HTTP/3 is RFC 9114 and QPACK is RFC 9204. QUIC version 2 is RFC 9369.

**Q: When would you still choose TCP over QUIC?**

A: I would keep TCP for internal, low-latency, low-loss networks where a mature kernel stack with hardware offload uses less CPU. I would also keep it where UDP is blocked, or where tools depend on TLS inspection. In practice, public services should offer both, and let QUIC be the fast path with TCP as the fallback.

## Quick Recap

- QUIC is a reliable, encrypted transport on top of UDP, defined mainly by RFC 9000, 9001, 9002 and 8999. HTTP/3 runs on it.
- Streams are independent, so a lost packet delays only the streams it carried. Order inside each stream still holds, and shared limits still apply.
- TLS 1.3 is built in. A new connection takes 1 RTT. A resumed one can use 0-RTT, which can be replayed, so use it only for safe requests.
- Connection IDs, not IP and port, identify a connection. Migration uses a fresh ID and PATH_CHALLENGE and PATH_RESPONSE.
- The 1200-byte minimum and the 3 times limit protect against amplification. Retry adds stronger address validation.
- Packet numbers never repeat, ACKs carry many ranges, and PTO replaces the classic RTO, which removes retransmission ambiguity.
- Many firewalls block UDP 443 and traffic falls back to TCP. Initial packets are readable by anyone, NAT timers are short, and operators have less visibility.
