# TCP Header and the Three-Way Handshake

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](01-ports-and-sockets.md), [TCP vs UDP](02-tcp-vs-udp.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [Binary and Hexadecimal](../01-foundations/07-binary-and-hexadecimal.md)

## In One Line

TCP puts a header of at least 20 bytes on every piece of data, and uses three short messages (SYN, SYN-ACK, ACK) to agree on starting numbers before any data flows.

## Why It Matters

IP delivers packets on a "best effort" basis. Packets can be lost, repeated, or arrive in the wrong order. Applications such as web servers and databases need a clean stream of bytes. TCP (Transmission Control Protocol) builds that stream on top of IP.

To do this, both computers must keep track of the same numbers. The header carries those numbers. The handshake sets them up.

Interviewers ask about this for three reasons:

- It tests whether you understand the numbers, not only the names. "What is the ack number after a 100-byte segment?" separates memorising from understanding.
- Real problems show up here. "Connection refused", "connection reset by peer", and connect timeouts each map to a specific packet pattern.
- It leads to security and scale topics: SYN floods, SYN cookies, spoofing, and connection queues.

The current TCP standard is RFC 9293 (published 2022). It replaced the original RFC 793 from 1981.

## Simple Explanation

Think of a phone call on a bad line.

1. You say: "Hello, can you hear me?"
2. Your friend says: "Yes, I hear you. Can you hear me?"
3. You say: "Yes, I hear you."

Now both people know the line works in both directions. Only then does the real talk start. That is the three-way handshake.

Now add one more idea. Before the call, each person picks a random page number to start from. Every word they say is numbered, starting from that page. If the listener misses a word, they can say "I got everything up to word 1050, please go on from there." That is what sequence and acknowledgement numbers do.

**Where the analogy stops being true:** a phone call has one live line. TCP has no line at all. Each message is a separate packet, and it may take a different path. Also, the handshake does not test line quality. It only proves that one packet got through each way, and it fixes the starting numbers.

Key words used in the rest of this chapter:

| Term | Plain meaning |
|---|---|
| Segment | One TCP unit: a TCP header plus (maybe) some data |
| Byte stream | Data seen as one long row of bytes, with no message edges |
| Flag | One bit in the header that switches a meaning on or off |
| Handshake | The short exchange that sets up a connection |
| ISN | Initial Sequence Number, the first number a side picks |
| RTT | Round-trip time, the time to go there and come back |
| 4-tuple | Source IP, source port, destination IP, destination port. These four values name one connection |

## How It Works

### The TCP header

TCP sits inside an IP packet. IP uses protocol number 6 to say "the payload is TCP". The same TCP header is used over IPv4 and IPv6.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Acknowledgement Number                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Data  | Rsrvd |C|E|U|A|P|R|S|F|                               |
| Offset|       |W|C|R|C|S|S|Y|I|            Window             |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |        Urgent Pointer         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Options and padding (0 to 40 bytes)              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Data (may be empty)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fixed part is 20 bytes. With options the header can grow to 60 bytes.

| Field | Size | What it does |
|---|---|---|
| Source Port | 16 bits | Port of the sender, for example 51514 |
| Destination Port | 16 bits | Port of the receiver, for example 80 |
| Sequence Number | 32 bits | Number of the first data byte in this segment. In a SYN, it is the ISN |
| Acknowledgement Number | 32 bits | The next byte the sender expects to receive. Only valid when the ACK flag is set |
| Data Offset | 4 bits | Header length, counted in 4-byte words. 5 means 20 bytes, 15 means 60 bytes |
| Reserved | 4 bits | Not in use. Senders set them to 0 |
| Flags | 8 bits | CWR, ECE, URG, ACK, PSH, RST, SYN, FIN (next section) |
| Window | 16 bits | How many more bytes the sender of this segment can accept. Used for flow control |
| Checksum | 16 bits | Error check over a pseudo-header, the TCP header and the data |
| Urgent Pointer | 16 bits | Only used with URG. An offset that marks the end of urgent data |
| Options | 0-40 bytes | Extra features such as MSS and window scaling |

Details that interviewers like:

- **There is no length field for the data.** The receiver works it out: IP total length, minus IP header length, minus TCP header length. Example: 127 - 20 - 32 = 75 bytes of data.
- **There are no IP addresses in the TCP header.** IP carries them. The checksum still protects them through the pseudo-header.
- **The pseudo-header** is a small block built only for the checksum math. It holds the source IP, destination IP, protocol number and TCP length. It is never sent. Because of it, a segment delivered to the wrong IP address fails the checksum.
- **The checksum is required in TCP**, on both IPv4 and IPv6. It is weak (16 bits), so strong integrity comes from TLS (the encryption layer used by HTTPS) at a higher layer.
- **The Window field is too small for fast links.** At 16 bits the most it can say is 65,535 bytes. The Window Scale option fixes this. See [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md).

### The flags

Flags are single bits in byte 13 of the header (counting from 0). Each bit has a value, so a set of flags is often written in hex.

| Bit value | Flag | Name | Meaning |
|---|---|---|---|
| 0x80 | CWR | Congestion Window Reduced | "I saw your ECN signal and slowed down" |
| 0x40 | ECE | ECN-Echo | "The network marked a packet as congested" |
| 0x20 | URG | Urgent | The Urgent Pointer field is valid |
| 0x10 | ACK | Acknowledgement | The ack number is valid |
| 0x08 | PSH | Push | "Hand this to the application now" |
| 0x04 | RST | Reset | "Abort this connection now" |
| 0x02 | SYN | Synchronise | "Here is my starting sequence number" |
| 0x01 | FIN | Finish | "I have no more data to send" |

ECN (Explicit Congestion Notification, RFC 3168) lets routers mark packets instead of dropping them. It is covered in [TCP Congestion Control](06-tcp-congestion-control.md).

Common combinations, with how tcpdump prints them:

| Hex | Flags | tcpdump | Where you see it |
|---|---|---|---|
| 0x02 | SYN | `[S]` | First packet of a handshake |
| 0x12 | SYN, ACK | `[S.]` | Second packet of a handshake |
| 0x10 | ACK | `[.]` | Pure acknowledgement |
| 0x18 | PSH, ACK | `[P.]` | Normal data segment |
| 0x11 | FIN, ACK | `[F.]` | Normal close |
| 0x04 | RST | `[R]` | Abort |
| 0x14 | RST, ACK | `[R.]` | Reply to a SYN on a closed port |

In tcpdump, the dot means ACK.

Three flags are often misunderstood:

- **ACK** is set on almost every segment. The only normal segments without it are the first SYN and some RST segments.
- **PSH** does not make the packet travel faster. It tells the receiving TCP not to hold data back from the application. Applications normally do not set it. The sending TCP sets it when it reaches the end of the data the application gave it.
- **URG** is almost unused today. Old Telnet and FTP used it to send an interrupt. RFC 6093 says new applications should not use it. The name "urgent" does not mean the segment skips any queue.

The four reserved bits are sometimes reused by newer work. For example, an older experiment used one as the NS flag, and that experiment has ended. Accurate ECN (AccECN) now gives that same bit a new use. You may see this in very new captures.

### Sequence and acknowledgement numbers

TCP numbers **bytes, not segments**. This is the most important fact in the chapter.

The rules are simple:

1. **Seq** is the number of the first byte of data in this segment.
2. **Ack** is the number of the next byte I expect from you. So "ack 1101" means "I have everything up to byte 1100."
3. Next seq = this seq + data length. Add 1 more if the SYN flag is set. Add 1 more if the FIN flag is set.
4. A pure ACK has no data, so it uses up no sequence number.

SYN and FIN each take up one sequence number even though they carry no data. They do this so that they can be acknowledged and resent like data. If a SYN used no number, an ack could not say "I got your SYN."

Each direction has its own separate counter. The client's seq counts client bytes. The server's seq counts server bytes.

**Analogy:** two people writing a shared story, each numbering their own letters. When you reply, you write "my letter number" and "the next letter number I expect from you." **Where it stops:** TCP numbers each byte, and not each letter or message. A sender can resplit bytes into different segments when it resends.

### The three-way handshake, with real numbers

Here the client picks ISN 1000 and the server picks ISN 5000. Small numbers are easier to follow. Real ISNs are large random values.

```text
Client 192.168.1.10:51514            Server 192.168.1.20:80
CLOSED                                                 LISTEN
  |                                                       |
  | 1. SYN  seq=1000  len=0                               |
  |------------------------------------------------------>|
SYN_SENT                                         SYN_RECEIVED
  |                                                       |
  | 2. SYN,ACK  seq=5000  ack=1001  len=0                 |
  |<------------------------------------------------------|
ESTABLISHED                                               |
  |                                                       |
  | 3. ACK  seq=1001  ack=5001  len=0                     |
  |------------------------------------------------------>|
  |                                             ESTABLISHED
```

1. **SYN.** The client says: "My numbering starts at 1000." The ack field is not used yet, so tools show it as 0.
2. **SYN-ACK.** The server does two jobs in one packet. It says "my numbering starts at 5000" and "I got your SYN, so I expect byte 1001 next."
3. **ACK.** The client says: "I got your SYN, so I expect byte 5001 next." Its seq is 1001, because the SYN used up number 1000.

The options in steps 1 and 2 also agree on features for this connection. See the options section below.

Data can travel with step 3. The client does not have to wait. So data can leave the client after one RTT.

**Timing example.** The RTT is 40 ms. The server is reached in 20 ms.

| Time | Event |
|---|---|
| 0 ms | Client sends SYN |
| 20 ms | Server receives SYN, sends SYN-ACK |
| 40 ms | Client receives SYN-ACK, sends ACK and the request |
| 60 ms | Server receives the request, sends the response |
| 80 ms | Client receives the first response bytes |

If TLS is used, its own handshake comes on top of this. See [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

### Data transfer, with real numbers

The connection continues. The client sends a 100-byte request. The server sends a 500-byte response.

| # | From | Flags | Seq | Ack | Len | Why these numbers |
|---|---|---|---|---|---|---|
| 1 | C | SYN | 1000 | - | 0 | Client ISN |
| 2 | S | SYN, ACK | 5000 | 1001 | 0 | Server ISN. Ack is 1000 + 1 for the SYN |
| 3 | C | ACK | 1001 | 5001 | 0 | Ack is 5000 + 1 for the SYN |
| 4 | C | PSH, ACK | 1001 | 5001 | 100 | Seq has not moved, as step 3 had no data |
| 5 | S | ACK | 5001 | 1101 | 0 | Ack is 1001 + 100 |
| 6 | S | PSH, ACK | 5001 | 1101 | 500 | Seq has not moved, as step 5 had no data |
| 7 | C | ACK | 1101 | 5501 | 0 | Ack is 5001 + 500 |
| 8 | C | FIN, ACK | 1101 | 5501 | 0 | Client is done |
| 9 | S | ACK | 5501 | 1102 | 0 | Ack is 1101 + 1 for the FIN |
| 10 | S | FIN, ACK | 5501 | 1102 | 0 | Server is done |
| 11 | C | ACK | 1102 | 5502 | 0 | Ack is 5501 + 1 for the FIN |

The check to do in an interview: the ack you send is always the other side's seq, plus its data length, plus 1 if it had SYN or FIN.

In real traffic, steps 5 and 6 often merge into one segment. That is called **piggybacking**: an ACK rides along with data. An ACK is also cumulative, so it covers all bytes before that number. See [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md) for loss and retransmission.

### Why three steps and not two

A connection needs two things: each side must pick an ISN, and each side must know the other received it. That takes four actions: SYN, ACK, SYN, ACK. The middle two fit in one packet, so the total is three.

With only two steps, three things break.

**1. The server's ISN is never confirmed.** The server sends its number, but never learns if it arrived. Then it cannot safely number its own data.

**2. Old duplicate SYNs would open false connections.** This is the main reason given in the original TCP design. Follow the case:

- A client sends a SYN. It gets stuck in the network for a long time.
- The client gives up. That connection attempt ends.
- The old SYN finally reaches the server.

With two steps, the server treats this as a real connection. It sets up memory and may start sending data to a client that does not want it.

With three steps, the server only sends a SYN-ACK and waits. The client looks at the ack number, sees it does not match anything, and sends RST. The server drops the half-made connection.

```text
Client                                      Server
  |     (old SYN seq=900, delayed in network)  |
  |------------------------------------------->|
  |     SYN,ACK seq=7000 ack=901               |
  |<-------------------------------------------|
  | "I have no attempt with seq 900"           |
  |     RST seq=901                            |
  |------------------------------------------->|
  |                  server drops the attempt  |
```

**3. Forged source addresses would be easy.** If two steps were enough, an attacker could fake any source IP and open a connection at once. With three steps, the sender must be able to see the SYN-ACK, or at least guess its number. Random ISNs stop that guess.

**Analogy:** a restaurant gets a booking letter that was lost in the post for a month. It phones the guest to confirm. The guest says "I did not book." The restaurant cancels and keeps the table free. **Where it stops:** TCP does not ask a person. The check is automatic and based only on numbers.

### Why the ISN is random

The ISN is the first sequence number. It could be 0, but it never is, for two reasons.

**Reason 1: old segments.** Suppose a connection closes, and a new one opens with the same 4-tuple. A delayed segment from the old connection could arrive. If both connections started at 0, its numbers might look valid. Different starting points make this much less likely.

**Reason 2: attacks.** To send a fake segment that a receiver accepts, the attacker must have a seq number that falls inside the receive window. If ISNs are guessable, an attacker who cannot see the traffic can still:

- Fake a whole connection from a trusted IP address.
- Insert false data into a live connection.
- Kill a connection with a fake RST.

**History.** RFC 793 used a clock that added 1 about every 4 microseconds. This was easy to predict. Robert T. Morris described the weakness in 1985. Kevin Mitnick's well-known 1994 attack used ISN prediction with spoofed addresses.

**Current practice.** RFC 1948 proposed the fix, later RFC 6528, and now it is part of RFC 9293:

```text
ISN = M + F(local IP, local port, remote IP, remote port, secret key)
```

- M is a clock that ticks every 4 microseconds or so.
- F is a keyed hash, known as a pseudorandom function. Linux uses SipHash.
- The secret key is random and known only to this host.

Each 4-tuple gets its own hidden offset. An attacker who opens a connection and sees an ISN learns nothing useful about the ISN of another 4-tuple. The clock part still moves the numbers forward, which keeps the protection from Reason 1.

**Analogy:** a shipping company that starts each customer's parcel numbers from a secret offset. A fraudster cannot guess a valid number for someone else's parcel. **Where it stops:** an attacker who can see the traffic reads the numbers directly. Such an attacker is called "on-path", and one who cannot see the traffic is "off-path". ISN randomness only stops off-path attackers. On-path protection needs encryption such as TLS or IPsec.

### TCP options

Options live after the fixed header, in the extra space set by Data Offset. They make TCP extensible. Most are only offered in SYN segments, so a connection's features are fixed at the handshake.

Most options use the form Kind, Length, Value. The two exceptions are one byte long.

| Kind | Length | Name | Purpose |
|---|---|---|---|
| 0 | 1 | End of Option List | Marks the end of options |
| 1 | 1 | NOP (No-Operation) | Padding, to align the next option on a 4-byte edge |
| 2 | 4 | MSS | Largest data size I can receive in one segment |
| 3 | 3 | Window Scale | Multiplies the Window field by 2 to the power of this value |
| 4 | 2 | SACK Permitted | "I understand selective acks" |
| 5 | Variable | SACK | Lists blocks of bytes received out of order |
| 8 | 10 | Timestamps | TSval and TSecr, used for RTT and PAWS |
| 19 | 18 | TCP MD5 Signature | Old segment signing, still seen on BGP sessions |
| 28 | 4 | User Timeout | Tells the peer how long to wait before giving up |
| 29 | Variable | TCP-AO | Authentication Option, the successor to TCP MD5 |
| 30 | Variable | MPTCP | Multipath TCP, using more than one path |
| 34 | Variable | TCP Fast Open | Carries the Fast Open cookie |

The main ones in more detail:

- **MSS (Maximum Segment Size), RFC 9293.** The largest amount of data, not counting headers, that this side wants in one segment. MTU (Maximum Transmission Unit) is the largest IP packet a link can carry. On Ethernet with MTU 1500 and IPv4, MSS is 1500 - 20 - 20 = 1460. It is not a negotiation: each side announces its own limit. If it is missing, TCP assumes 536 bytes for IPv4 and 1220 for IPv6. See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).
- **Window Scale, RFC 7323.** Only valid in SYN segments, with a maximum shift of 14. Both sides must offer it, or neither side uses it. With a shift of 7, a Window field of 502 means 502 x 128 = 64,256 bytes. The largest possible window is 65,535 x 2^14 = 1,073,725,440 bytes, about 1 GiB. The Window field in a SYN or SYN-ACK is never scaled.
- **SACK (Selective Acknowledgement), RFC 2018.** SACK Permitted appears in the SYN. Later segments may hold up to 4 blocks, or 3 if timestamps are also on, since options have only 40 bytes.
- **Timestamps, RFC 7323.** Each side places its clock value in TSval and repeats the other side's last value in TSecr. This gives RTT measurements, and PAWS (see Going Deeper).

Option space is small: 40 bytes at most. A typical Linux SYN uses 20 of them.

### Four-way teardown

TCP is **full duplex**. That means each direction works on its own. So each direction closes on its own, and that takes two steps per direction.

```text
Client (active close)                   Server (passive close)
ESTABLISHED                                        ESTABLISHED
  |                                                       |
  | 1. FIN,ACK  seq=1101  ack=5501                        |
  |------------------------------------------------------>|
FIN_WAIT_1                                         CLOSE_WAIT
  |                                                       |
  | 2. ACK  seq=5501  ack=1102                            |
  |<------------------------------------------------------|
FIN_WAIT_2                                                |
  |              (server may still send data here)        |
  |                                                       |
  | 3. FIN,ACK  seq=5501  ack=1102                        |
  |<------------------------------------------------------|
TIME_WAIT                                            LAST_ACK
  |                                                       |
  | 4. ACK  seq=1102  ack=5502                            |
  |------------------------------------------------------>|
  | waits 2 x MSL                                    CLOSED
CLOSED
```

1. The client's application finishes and calls close. TCP sends FIN: "I will send no more data."
2. The server's TCP confirms at once. The server's application may not even know yet.
3. Later, the server's application finishes and calls close. TCP sends its own FIN.
4. The client confirms. Then it stays in TIME_WAIT for a while.

**Why four and not three, like the handshake?** In the handshake, the server's SYN and ACK are ready together. In teardown, the ACK comes from the kernel right away, but the FIN must wait for the application to finish. The two events can be far apart, so they cannot always share a packet.

If the server's application closes quickly, the ACK and FIN can merge. You then see three segments: FIN, FIN-ACK, ACK. This is common in captures. It is still logically the same four actions.

The side that closes first is the **active closer**, and it holds TIME_WAIT. TIME_WAIT lasts 2 x MSL (Maximum Segment Lifetime). It has two jobs: it lets the side resend the last ACK if that ACK gets lost, and it lets old segments die out before the 4-tuple is reused. RFC 9293 gives MSL as 2 minutes. Linux uses a fixed 60 seconds for TIME_WAIT. The state machine and TIME_WAIT in detail are in [TCP Connection States](04-tcp-states.md).

**Analogy:** a two-lane road being closed. Each lane has its own crew, and each crew reports "closed" and gets a reply. **Where it stops:** a road crew does not hold the road for a minute afterwards. TIME_WAIT has no real-world match in this picture.

### Half close

After step 2 above, the client can no longer send, but it can still receive. This state is a **half close**. It is a normal, standard feature.

The socket call that causes it is `shutdown(sock, SHUT_WR)`. It sends a FIN but keeps the receive side open. The other call, `close()`, gives up the socket in both directions.

| Call | Sends FIN? | Can still read? | Later data from peer |
|---|---|---|---|
| `shutdown(SHUT_WR)` | Yes | Yes | Read as normal |
| `close()` | Yes | No | Kernel answers with RST |

Half close is how a client says "that is all my input" without a separate message. Example: a client sends lines to a sorting service.

```text
Client                                    Server
  | data: "pear\napple\n"                   |
  |---------------------------------------->| read() gets data
  | FIN  (shutdown SHUT_WR)                 |
  |---------------------------------------->| read() returns 0, EOF
  | ACK                                     |
  |<----------------------------------------| server now sorts
  | data: "apple\npear\n"                   |
  |<----------------------------------------| reply still flows
  | ACK                                     |
  |---------------------------------------->|
  | FIN                                     |
  |<----------------------------------------| server closes
  | ACK                                     |
  |---------------------------------------->|
```

EOF (End Of File) is what the application sees when a FIN arrives: `read()` returns 0.

**Analogy:** a reporter on a radio says "those are all my questions, I am listening now." **Where it stops:** a radio is one-way at a time. TCP can carry both directions at once before the half close.

Do not confuse these two terms:

| Term | Meaning | Normal? |
|---|---|---|
| Half close | One side sent FIN on purpose, the other direction still works | Yes |
| Half open | One side crashed or lost state, the other still thinks the connection is alive | No, it is a failure |

"Half open" is also used for a connection stuck in SYN_RECEIVED, during a SYN flood. The meaning depends on context. A TCP keepalive (a small test segment sent on an idle connection) or an application heartbeat can find a dead peer.

### RST: abort the connection

RST (reset) ends a connection at once. Unlike FIN, it has no goodbye.

- Data still in buffers may be thrown away.
- The receiver does not answer it. No ACK, and never an RST in reply to an RST.
- The application sees an error, not a normal EOF.

When RST is sent:

| Case | Who sends | Application sees |
|---|---|---|
| SYN to a port with no listener | Target host | `Connection refused` (ECONNREFUSED) |
| Segment for a connection that does not exist, for example after a crash and reboot | The host that lost state | `Connection reset by peer` (ECONNRESET) |
| Application calls `close()` with unread data in its receive buffer | Local TCP (Linux and most others) | Peer sees a reset |
| Application uses SO_LINGER with timeout 0, then close | Local TCP | Peer sees a reset |
| Firewall or load balancer ends an idle or banned flow | The middle device, faking one end | `Connection reset by peer` |
| Old SYN-ACK arrives at a client with no such attempt | Client | Nothing, it is silent cleanup |

The numbers in an RST follow a rule in RFC 9293:

- If the segment that caused it had an ACK: RST seq = that segment's ack number, and no ACK flag.
- If it had no ACK: RST seq = 0, ACK flag set, ack = its seq + its length. A SYN counts as length 1.

So a SYN with seq 1000 to a closed port gets `RST,ACK seq=0 ack=1001`.

A diagnostic tip: if an RST has a different IP TTL, or arrives faster than the server's normal replies, a device in the middle may have sent it.

**Analogy:** FIN is saying "goodbye" and hanging up. RST is pulling the phone cable from the wall. **Where it stops:** the phone company cannot usually fake a hang-up. A firewall in the path can and does send an RST in the name of the far end.

| | FIN | RST |
|---|---|---|
| Type of close | Graceful | Abrupt |
| Pending data | Delivered first | May be lost |
| Is it acknowledged? | Yes | No |
| Uses a sequence number? | Yes, 1 | No |
| TIME_WAIT? | Yes, on the active closer | No |

### SYN floods and SYN cookies

When a SYN arrives at a listening port, the server normally creates a small entry for the half-made connection, sends a SYN-ACK, and waits. This waiting list is limited. A **SYN flood** abuses that: an attacker sends many SYNs, often with fake source addresses. The SYN-ACKs go nowhere, no third ACK comes, and the list fills. Real clients are then refused or ignored. More on attacks: [Common Network Attacks](../09-security/03-common-network-attacks.md).

**SYN cookies** let the server keep no state until the third packet proves the client is real. Daniel J. Bernstein and Eric Schenk created this idea in 1996.

The trick: the server encodes what it needs into its own ISN, sends it in the SYN-ACK, and forgets. The client returns that number plus 1 in its ACK.

```text
Classic SYN cookie layout (32-bit server ISN)

 bits 31-27   bits 26-24    bits 23-0
+-----------+------------+---------------------------------+
| t (time)  | m (MSS     | s = keyed hash of 4-tuple,      |
| 5 bits    | index)     | t, and a secret  (24 bits)      |
+-----------+------------+---------------------------------+
```

Worked example:

1. A SYN arrives with MSS 1460. In a small server table, that MSS is index 3, so m = 3.
2. The time counter (it ticks slowly, for example every 64 seconds) gives t = 17.
3. The hash of the 4-tuple, t and the secret gives s = 0xA1B2C3.
4. Cookie = (17 << 27) | (3 << 24) | 0xA1B2C3 = 0x8BA1B2C3. In decimal it is 2342630083.
5. The server sends a SYN-ACK with seq 2342630083, and stores nothing.
6. A real client replies with ack 2342630084.
7. The server subtracts 1, gets the cookie, reads t and m, and recomputes the hash.
8. If the hash matches and t is recent, the server now creates the connection, with MSS 1460.

A spoofed sender never sees the SYN-ACK, so it cannot return the right ack.

```text
Client                                  Server
  | SYN seq=1000 MSS=1460                 |
  |-------------------------------------->| compute cookie
  | SYN,ACK seq=cookie ack=1001           | store NOTHING
  |<--------------------------------------|
  | ACK seq=1001 ack=cookie+1             |
  |-------------------------------------->| check hash
  |                                       | now create state
```

**Analogy:** a cloakroom during a huge crowd. It has no space for a list, so it gives each guest a signed ticket with all the details written on it. When a guest comes back with a valid signature, the staff read the ticket and trust it. **Where it stops:** a TCP cookie has only 32 bits, so it can hold very little.

**Limits of SYN cookies:**

- Only a few MSS values fit, so the true MSS may be rounded down.
- There is no room for options in the classic design. Window scale and SACK would be lost. Linux can store them in the low bits of the TCP timestamp when the client uses timestamps.
- The server cannot resend the SYN-ACK, as it has no record. If the third ACK is lost and the server speaks first in the protocol (SMTP, for example), the connection can stall until the client sends something.
- Cookies cost some CPU for each SYN, and they do not help against a flood big enough to fill the link.

Because of this, Linux uses cookies only when the queue overflows, when `tcp_syncookies` is 1, which is the kernel default. Real implementations differ from the classic layout: Linux has its own bit layout and a smaller MSS table. RFC 4987 describes SYN floods and common defences. Other defences include SYN proxies (a firewall or load balancer finishes the handshake first, then connects to the server) and upstream scrubbing (a provider filters attack traffic before it reaches you).

### TCP Fast Open

A normal handshake costs one RTT before any data can flow. TCP Fast Open (TFO, RFC 7413) sends data inside the SYN on repeat visits. It is an Experimental RFC.

Plain TCP allows data in a SYN, but the receiver must hold it until the handshake ends. TFO lets the server use it right away. To do that safely, it needs proof that the client really owns its source IP. That proof is a **TFO cookie**.

**First visit: get a cookie (a normal one-RTT handshake).**

```text
Client                                     Server
  | SYN  TFO option: cookie request          |
  |----------------------------------------->|
  | SYN,ACK  TFO option: cookie=C            | C = keyed MAC
  |<-----------------------------------------| of client IP
  | ACK                                      |
  |----------------------------------------->|
  | client stores C for this server          |
```

MAC here means Message Authentication Code, a keyed hash that only the server can produce.

**Later visit: send data in the SYN.** The client ISN is 7000, and the request is 75 bytes.

```text
Client                                     Server
  | SYN seq=7000 cookie=C + 75 bytes data    |
  |----------------------------------------->| cookie valid:
  |                                          | pass data to app
  | SYN,ACK seq=9000 ack=7076                |
  |<-----------------------------------------|
  | data seq=9001 len=850                    | reply sent before
  |<-----------------------------------------| the third ACK
  | ACK seq=7076 ack=9851                    |
  |----------------------------------------->|
```

Ack 7076 is 7000 + 1 for the SYN + 75 bytes. Ack 9851 is 9001 + 850.

If the cookie is bad, the server ignores the data and acks only the SYN (ack=7001). The client then resends the 75 bytes as normal. No harm is done, only the time saving is lost.

With the same 40 ms RTT as before, the first response bytes now arrive at 40 ms instead of 80 ms.

**Analogy:** a gym member with a card walks in and asks for a towel while still showing the card. A first-time visitor has to sign up at the desk first. **Where it stops:** a duplicate towel request is harmless, but a repeated network request may not be. See below.

**Limits of TFO:**

- **Replay.** A SYN with data can be duplicated by the network, or by an attacker. The server application could run the same request twice. So TFO is only safe for idempotent requests, meaning requests that are safe to repeat. A GET is usually fine, and a money transfer is not.
- **Middleboxes.** A middlebox is any device in the path that reads or changes traffic. Some firewalls and NAT devices drop SYNs with data or unknown options. Clients need a fallback.
- **Low use.** Chrome and Firefox do not use it by default today. Encrypted setups that want early data often use TLS 1.3 early data or [QUIC](08-quic.md) 0-RTT instead.

Linux controls it with `net.ipv4.tcp_fastopen`, a bit mask: 1 enables the client side, 2 the server side, 3 both.

## Example

The setup: a Linux client at 192.168.1.10 runs curl against a web server at 192.168.1.20 on port 80. Both are on the same LAN, so there is no NAT.

### Capture a full connection

```bash
# Linux, on the client (-nn: no name lookups, -S: absolute seq numbers)
sudo tcpdump -i eth0 -nn -S 'host 192.168.1.20 and tcp port 80'

# In a second terminal
curl -s -o /dev/null http://192.168.1.20/
```

Output below is wrapped to fit. Each packet is one line in the real output.

```text
10:15:02.100123 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [S],
    seq 2839104512, win 64240,
    options [mss 1460,sackOK,TS val 1583112 ecr 0,nop,wscale 7],
    length 0
10:15:02.100654 IP 192.168.1.20.80 > 192.168.1.10.51514: Flags [S.],
    seq 1178235904, ack 2839104513, win 65160,
    options [mss 1460,sackOK,TS val 9912405 ecr 1583112,nop,wscale 7],
    length 0
10:15:02.100701 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [.],
    ack 1178235905, win 502,
    options [nop,nop,TS val 1583113 ecr 9912405], length 0
10:15:02.100812 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [P.],
    seq 2839104513:2839104588, ack 1178235905, win 502,
    options [nop,nop,TS val 1583113 ecr 9912405],
    length 75: HTTP: GET / HTTP/1.1
10:15:02.101230 IP 192.168.1.20.80 > 192.168.1.10.51514: Flags [.],
    ack 2839104588, win 509,
    options [nop,nop,TS val 9912406 ecr 1583113], length 0
10:15:02.101498 IP 192.168.1.20.80 > 192.168.1.10.51514: Flags [P.],
    seq 1178235905:1178236755, ack 2839104588, win 509,
    options [nop,nop,TS val 9912406 ecr 1583113],
    length 850: HTTP: HTTP/1.1 200 OK
10:15:02.101533 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [.],
    ack 1178236755, win 495,
    options [nop,nop,TS val 1583114 ecr 9912406], length 0
10:15:02.101790 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [F.],
    seq 2839104588, ack 1178236755, win 495,
    options [nop,nop,TS val 1583114 ecr 9912406], length 0
10:15:02.102105 IP 192.168.1.20.80 > 192.168.1.10.51514: Flags [F.],
    seq 1178236755, ack 2839104589, win 509,
    options [nop,nop,TS val 9912407 ecr 1583114], length 0
10:15:02.102140 IP 192.168.1.10.51514 > 192.168.1.20.80: Flags [.],
    ack 1178236756, win 495,
    options [nop,nop,TS val 1583115 ecr 9912407], length 0
```

Read it with the rules:

| Check | Math |
|---|---|
| Server's first ack | 2839104512 + 1 = 2839104513 |
| Client's first ack | 1178235904 + 1 = 1178235905 |
| Request, 75 bytes | 2839104513 + 75 = 2839104588, shown as the end of the range |
| Server's ack of request | 2839104588 |
| Response, 850 bytes | 1178235905 + 850 = 1178236755 |
| Server's ack of client FIN | 2839104588 + 1 = 2839104589 |
| Client's ack of server FIN | 1178236755 + 1 = 1178236756 |
| Client's real window at first ACK | 502 x 128 = 64,256 bytes (wscale 7) |

Things to note:

- The teardown has three segments, not four. The server's ACK and FIN merged. The client closed first, so the client holds TIME_WAIT.
- tcpdump leaves out the seq on a pure ACK, because no data follows it.
- Without `-S`, tcpdump shows small relative numbers after the handshake. Wireshark also shows relative numbers by default, starting at 0. These are display tricks. The packet holds the full 32-bit value.
- In a capture on the sender, you may see "cksum incorrect". This is usually checksum offload: the network card fills the checksum in later.

### Decode the SYN byte by byte

These are the 40 bytes of TCP header from the first packet, as `tcpdump -x` style hex.

```text
c93a 0050 a939 4c00 0000 0000 a002 faf0
e2bb 0000 0204 05b4 0402 080a 0018 2808
0000 0000 0103 0307
```

| Bytes | Hex | Field | Value |
|---|---|---|---|
| 0-1 | c93a | Source Port | 51514 |
| 2-3 | 0050 | Destination Port | 80 |
| 4-7 | a9394c00 | Sequence Number | 2839104512 |
| 8-11 | 00000000 | Ack Number | 0 (no ACK flag, so not used) |
| 12 (high 4 bits) | a | Data Offset | 10 words = 40 bytes |
| 12 (low 4 bits) | 0 | Reserved | 0 |
| 13 | 02 | Flags | SYN only |
| 14-15 | faf0 | Window | 64240 |
| 16-17 | e2bb | Checksum | Checks out against the pseudo-header |
| 18-19 | 0000 | Urgent Pointer | 0 |
| 20-23 | 0204 05b4 | Option: MSS | Kind 2, length 4, value 1460 |
| 24-25 | 0402 | Option: SACK Permitted | Kind 4, length 2 |
| 26-35 | 080a 00182808 00000000 | Option: Timestamps | TSval 1583112, TSecr 0 |
| 36 | 01 | Option: NOP | Padding |
| 37-39 | 030307 | Option: Window Scale | Kind 3, length 3, shift 7 |

Options total 4 + 2 + 10 + 1 + 3 = 20 bytes. Header total is 20 + 20 = 40 bytes. The IP packet is 20 + 40 = 60 bytes.

### A closed port

```bash
# Linux (OpenBSD netcat), port 81 has no listener
nc -vz 192.168.1.20 81
```

```text
nc: connect to 192.168.1.20 port 81 (tcp) failed: Connection refused
```

The capture shows two packets. The second answers the first in the same flow.

```text
10:20:11.402117 IP 192.168.1.10.51520 > 192.168.1.20.81: Flags [S],
    seq 1203948576, win 64240,
    options [mss 1460,sackOK,TS val 1892411 ecr 0,nop,wscale 7],
    length 0
10:20:11.402533 IP 192.168.1.20.81 > 192.168.1.10.51520: Flags [R.],
    seq 0, ack 1203948577, win 0, length 0
```

If a firewall silently drops the SYN, there is no reply. The client retries and then times out. On Linux, `tcp_syn_retries` defaults to 6, and the retries double in spacing from 1 second, so a blocking `connect()` takes around 127 seconds to fail.

| What you see | Likely cause |
|---|---|
| SYN, then RST-ACK | Host is up, port is closed, or a firewall sent a reject |
| SYN, SYN, SYN, no reply | Packet is dropped, or the host is down or there is no route |
| SYN, SYN-ACK, ACK, then RST | Application or middlebox rejected the connection after setup |

### Filter on flags

```bash
# Linux: only SYN and SYN-ACK packets
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0'

# Linux: only packets where SYN is the only flag set (byte 13 equals 2)
sudo tcpdump -i eth0 -nn 'tcp[13] = 2'

# Linux: only RST packets
sudo tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'
```

In Wireshark, the display filter for a first SYN is `tcp.flags.syn == 1 && tcp.flags.ack == 0`. More in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) and [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

### Test and inspect from each OS

**Test if a TCP connection can be set up:**

```bash
# Linux
nc -vz 192.168.1.20 80
```

```text
Connection to 192.168.1.20 80 port [tcp/http] succeeded!
```

```powershell
# Windows, from a host at 192.168.1.15 on the same LAN
Test-NetConnection 192.168.1.20 -Port 80
```

```text
ComputerName     : 192.168.1.20
RemoteAddress    : 192.168.1.20
RemotePort       : 80
InterfaceAlias   : Ethernet
SourceAddress    : 192.168.1.15
TcpTestSucceeded : True
```

**See handshake-related settings:**

```bash
# Linux. Values are typical and vary by kernel and distribution.
sysctl net.ipv4.tcp_syncookies net.ipv4.tcp_syn_retries \
       net.ipv4.tcp_synack_retries net.core.somaxconn \
       net.ipv4.tcp_fastopen
```

```text
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.core.somaxconn = 4096
net.ipv4.tcp_fastopen = 1
```

```powershell
# Windows: shows Max SYN Retransmissions, Initial RTO and more
netsh interface tcp show global

# Windows: live TCP connections for a remote port
Get-NetTCPConnection -RemotePort 80
```

**Check a server for SYN pressure:**

```bash
# Linux: count half-made connections on port 80
ss -tan state syn-recv '( sport = :80 )' | wc -l

# Linux: SYN cookie and listen queue counters
nstat -az TcpExtSyncookiesSent TcpExtSyncookiesRecv \
          TcpExtListenOverflows TcpExtListenDrops
```

The kernel log also warns during a flood, with a line like `Possible SYN flooding on port 80. Sending cookies.`

## Going Deeper

### The two queues behind a listening socket

On Linux and similar systems, a listening socket has two queues:

```text
 SYN arrives                third ACK arrives          accept()
      |                            |                      |
      v                            v                      v
+-----------------+         +-----------------+      +---------+
| SYN queue       |  ---->  | Accept queue    | ---> | Program |
| (SYN_RECEIVED)  |         | (ESTABLISHED,   |      |         |
|                 |         |  waiting)       |      |         |
+-----------------+         +-----------------+      +---------+
```

- The **SYN queue** holds half-made connections.
- The **accept queue** holds finished connections that the program has not yet taken with `accept()`.

The accept queue size is the smaller of the `backlog` value given to `listen()` and `net.core.somaxconn`.

If the accept queue is full, Linux by default drops the third ACK. The client thinks it is connected and sends data. The server then resends its SYN-ACK and ignores data for a while. The symptom is slow first requests with no clear error. Set `tcp_abort_on_overflow` to 1 and the server sends RST instead. Check with `ss -lnt`: for a listener, Recv-Q shows the current accept queue, and Send-Q shows its limit.

### Handshake loss cases

| Lost packet | Result |
|---|---|
| SYN | Client resends after its timer, starting around 1 second (RFC 6298) and doubling |
| SYN-ACK | Client resends SYN, and server also resends SYN-ACK |
| Third ACK | Client is ESTABLISHED, server is not. The next client data carries the ACK and fixes this. If the client stays silent, the server resends the SYN-ACK |

### Simultaneous open and close

Two sides can send SYN to each other at the same moment, with ports known in advance. Both go from SYN_SENT to SYN_RECEIVED, then each sends a SYN-ACK. The result is one connection built with four segments and no listener. It is rare, but the standard defines it. NAT (Network Address Translation) traversal by "TCP hole punching" can rely on it.

Both sides can also send FIN at once. Both move to the CLOSING state and then TIME_WAIT. See [TCP Connection States](04-tcp-states.md).

### Sequence wrap and PAWS

32 bits gives 4,294,967,296 numbers, about 4 GiB of data. After that the counter wraps back to 0. TCP compares numbers with modular math, so a wrap by itself is fine.

The risk is on fast links. At 10 Gbit/s, the counter wraps in about 3.4 seconds. An old delayed segment could then have a number that looks valid again. PAWS (Protection Against Wrapped Sequences, RFC 7323) uses timestamps to reject it: a segment with a TSval older than the last one seen is dropped, even if its seq looks fine.

### Blind attacks, RFC 5961

Old rules let any RST with a seq inside the receive window kill a connection. A large window made guessing easier. RFC 5961 tightened this:

- An RST with seq exactly equal to the next expected number resets the connection.
- An RST with seq elsewhere inside the window gets a **challenge ACK** instead. If the RST was real, the true peer replies with a correct RST. If it was forged, nothing more happens.
- A SYN on an open connection also gets a challenge ACK.

Rate limits on challenge ACKs matter too. In 2016, a side channel in Linux's global challenge ACK limit (CVE-2016-5696) let an off-path attacker find connection details. The fix included randomising the limit.

### Signing TCP segments

BGP (the routing protocol between networks on the internet) runs over TCP, so a forged RST could drop a routing session. TCP MD5 (RFC 2385) added a keyed hash on each segment. It is obsoleted by TCP-AO (RFC 5925), which has better key handling, but MD5 stays widely deployed. See [BGP](../05-routing/09-bgp.md).

### ECN in the handshake

A client that wants ECN sets ECE and CWR in its SYN (flags 0xC2). A server that agrees sets ECE in the SYN-ACK (flags 0x52). On Linux, `tcp_ecn` defaults to 2: it accepts ECN when asked, but does not ask for it on outgoing connections.

### Why TCP is hard to change

Many firewalls and NAT devices read and rewrite TCP headers, and may drop what they do not know. This is called **ossification**. It is one reason TCP Fast Open was slow to spread, why option space is still only 40 bytes, and why MPTCP needs fallback. QUIC runs over UDP and encrypts most of its transport header partly to avoid this problem. See [QUIC](08-quic.md).

## Common Mistakes

- **"Seq numbers count packets."** They count bytes. Two 100-byte segments move seq by 200, not 2.
- **"Ack means the last byte received."** Ack is the next byte expected. Ack 1101 means bytes through 1100 arrived.
- **"A pure ACK uses a sequence number."** It does not. Only data, SYN and FIN move seq forward.
- **"After a SYN with seq 1000, the first data byte is at 1000."** The SYN uses 1000, so data starts at 1001.
- **"The ISN starts at 0."** Wireshark shows 0, but that is a relative number. The real ISN is a large unpredictable value.
- **"Random ISNs make TCP secure."** They only stop off-path guessing. Anyone who can see the traffic can read the numbers, and data is still in clear text.
- **"MSS is negotiated to one value."** Each side announces its own limit. The two values can differ.
- **"MSS counts the headers."** It counts only TCP data. TCP timestamps also use part of each segment, so real data per segment is often 1448 bytes, not 1460.
- **"PSH means high priority."** It only asks the receiver to pass data to the application without waiting.
- **"RST means a network failure."** RST is a clear message from a working TCP stack or a middle device. Silence, not RST, is what a network failure looks like.
- **"Connection timed out and connection refused are the same."** Refused means an RST came back. Timed out means no answer came back.
- **"Teardown is always four packets."** It is four actions. ACK and FIN often merge, giving three packets.
- **"The server always holds TIME_WAIT."** Whoever closes first holds it. That can be a client or a server.
- **"close() and shutdown() are the same."** `close()` gives up the socket. `shutdown(SHUT_WR)` only ends sending and keeps receiving.
- **"SYN cookies stop every SYN flood."** They remove queue exhaustion, but not link saturation.
- **"TFO removes the handshake."** It still sends SYN, SYN-ACK and ACK. It only lets data ride on the SYN once a cookie exists.

## Interview Questions

**Q: Walk me through the three-way handshake.**

A: The client sends a SYN with its random ISN. The server replies with SYN-ACK: it gives its own ISN, and its ack is the client ISN plus 1. The client sends an ACK, with ack set to the server ISN plus 1. Both sides also exchange options such as MSS, window scale, SACK and timestamps in the first two steps. After this, both are ESTABLISHED.

**Q: Client seq is 1000 and it sends 100 bytes. What does the server ack, and what is the client's next seq?**

A: The server acks 1100, because ack means the next byte expected. The client's next seq is also 1100. If the segment had a SYN or FIN flag, it would count as one more number.

**Q: Why three steps and not two?**

A: Each side must pick an ISN and know the other received it, which takes three messages because the middle one combines an ACK and a SYN. It also protects against old duplicate SYNs: if a delayed SYN reaches the server, the client rejects the SYN-ACK with an RST, and no false connection is made. Two steps would leave the server's ISN unconfirmed.

**Q: Why is the ISN random?**

A: First, so old segments from an earlier connection with the same 4-tuple do not look valid in a new one. Second, so off-path attackers cannot guess numbers and spoof a connection, inject data, or send fake resets. RFC 9293 uses a clock plus a keyed hash of the 4-tuple and a secret.

**Q: Why does closing take four steps?**

A: TCP is full duplex, so each direction closes on its own. The ACK of a FIN is sent by the kernel straight away, but the other side's FIN waits for its application to finish. Those events can be far apart. When they happen together, the ACK and FIN merge and you see three packets.

**Q: What is a half close? Give an example.**

A: One side sends FIN with `shutdown(SHUT_WR)` and keeps reading. It says "I have no more input, but I still want your reply." A client that streams data to a server and then waits for the result is the usual case. The server sees EOF, finishes, and replies.

**Q: What is the difference between FIN and RST?**

A: FIN is a graceful close: pending data is delivered, the FIN uses a sequence number, it is acknowledged, and the active closer enters TIME_WAIT. RST aborts at once: data can be lost, it is never acknowledged, and the application gets an error.

**Q: When would you see an RST? How do "connection refused" and a timeout differ?**

A: RST comes from a SYN to a closed port, a segment for an unknown connection, an application abort, or a firewall ending a flow. Connection refused means a RST came back, so the host is reachable. A timeout means nothing came back, so something dropped the packets or the host is unreachable.

**Q: What is a SYN flood, and how do SYN cookies help?**

A: An attacker sends many SYNs, often spoofed, so the server fills its queue of half-made connections. With SYN cookies, the server stores nothing. It encodes the time, the MSS index and a keyed hash in its ISN. A real client returns that value plus 1, and the server rebuilds the state after checking the hash. The costs are a limited MSS and possible loss of options, and the SYN-ACK cannot be resent.

**Q: What happens if the third ACK of the handshake is lost?**

A: The client is ESTABLISHED and may send data. That data also has the ACK, so the server completes the handshake when it arrives. If the client stays silent, the server's timer resends the SYN-ACK, and the client answers again.

**Q: What is TCP Fast Open, and what is its main risk?**

A: It sends data in the SYN on repeat connections, using a cookie the server gave earlier, which saves one RTT. The main risk is replay: a duplicated SYN could run the request twice, so it fits only idempotent requests. Middleboxes also sometimes block it.

**Q: A server shows many half-open connections and clients hang after connecting. How do you look at this?**

A: I check `ss -tan state syn-recv` for the count and `ss -lnt` for accept queue use. Then I check `nstat` for ListenOverflows, ListenDrops and SyncookiesSent, and read the kernel log. Many SYN_RECEIVED from random IPs suggests a SYN flood. A full accept queue suggests a slow application or a small backlog. A tcpdump confirms whether third ACKs arrive.

## Quick Recap

- The TCP header is 20 bytes fixed plus up to 40 bytes of options. It holds ports, seq, ack, offset, flags, window, checksum and urgent pointer. It has no IP addresses and no data length.
- Seq and ack count bytes. Ack is the next byte expected. SYN and FIN each use one number, a pure ACK uses none.
- Three steps confirm both ISNs and reject old duplicate SYNs. MSS, window scale, SACK and timestamps are set up in the SYN and SYN-ACK.
- The ISN is a clock plus a keyed hash of the 4-tuple. It guards against old segments and off-path attacks, not on-path readers.
- Closing takes four actions as each direction closes alone. Half close uses `shutdown(SHUT_WR)`, and the active closer holds TIME_WAIT.
- RST aborts at once. "Refused" means an RST came back. "Timed out" means nothing came back.
- SYN cookies keep no state during a SYN flood by hiding it in the ISN. TCP Fast Open sends data in the SYN using a cookie, at the risk of replay.
