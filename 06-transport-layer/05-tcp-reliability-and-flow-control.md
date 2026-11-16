# TCP Reliability and Flow Control

**Level:** Advanced | **Prerequisites:** [TCP vs UDP](02-tcp-vs-udp.md), [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md), [TCP Connection States](04-tcp-states.md)

## In One Line

TCP makes a lossy network look like a perfect pipe: it numbers every byte, confirms what arrived, resends what did not, and uses a window so the sender never overruns the receiver.

## Why It Matters

IP makes no promises. A packet can be dropped, delayed, duplicated or delivered out of order. Your database driver, your web browser and your SSH session still need a clean stream of bytes. TCP builds that stream on top of IP.

The same machinery causes many real production problems:

- A file transfer is stuck at 10 Mbps on a 1 Gbps link. The cause is often the window size, not the link.
- A web API has exactly 40 ms of extra delay on some requests. The cause is often Nagle and delayed ACK working against each other.
- A connection hangs for 15 minutes before an error appears. The cause is the retransmission timer backing off.
- A connection dies after sitting idle for an hour. The cause is often a firewall that forgot it, with no keepalive to keep it active.
- A capture shows `TCP ZeroWindow`. The cause is usually a slow application, not the network.

Interviewers ask about this topic because it separates people who memorised "TCP is reliable" from people who can debug a slow system. They want to hear how, with numbers.

## Simple Explanation

Imagine you mail a long book to a friend, one page in each envelope, and the postal service sometimes loses or mixes up envelopes.

You write a number on each page. Your friend sorts the pages and sends you postcards that say, "I have everything up to page 7. Send page 8 next."

- If no postcard comes for a long time, you mail the page again. That is the **retransmission timer**.
- If your friend keeps saying "still need page 8" while later pages arrive, you know page 8 was lost. You resend it at once. That is **fast retransmit**.
- If your friend writes, "I also have pages 10 to 12," you do not resend those. That is **SACK**.
- Your friend has one small desk. The postcard also says, "I have room for 5 more pages." You never send more than that. That is the **receive window**, and using it is **flow control**.
- If the postcard says "room for 0 pages," you stop, and you call once in a while to ask if there is room yet. That is the **zero window** and the **persist timer**.

**Where the analogy stops being true:** TCP numbers bytes, not pages. A retransmission can also carry a different amount of data from the original. And the postcards can be lost too, which TCP handles without any special step, as you will see.

## How It Works

### The building blocks

A few words first.

- **Byte stream:** TCP treats your data as one long line of bytes. It does not keep your message boundaries.
- **Segment:** one TCP unit (header plus some data) carried inside one IP packet.
- **MSS (Maximum Segment Size):** the largest amount of data in one segment. A common value is 1460 bytes, or 1448 when the timestamp option is used. See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).
- **Sequence number (SEQ):** the number of the first data byte in a segment. Each byte has its own number.
- **Acknowledgment number (ACK):** the number of the next byte the receiver expects.
- **RTT (Round-Trip Time):** the time from sending a segment to getting the ACK for it.
- **In flight:** bytes that were sent but are not yet acknowledged.
- **Socket buffer:** memory in the operating system kernel that holds data for one connection.

TCP is full duplex. Data can go both ways at once, so each direction has its own sequence numbers, ACKs and window.

The header fields used here are the 32-bit Sequence and Acknowledgment numbers, the 16-bit Window and Checksum, and up to 40 bytes of options. The full layout is in [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md).

TCP gets reliability from four tools working together:

1. **Checksum:** finds damage. A damaged segment is dropped without any message.
2. **Sequence numbers:** put data in order and remove duplicates.
3. **ACKs:** tell the sender what arrived.
4. **Retransmission:** fills the gaps.

### ACKs and the cumulative ACK

**The rule:** if a segment has SEQ = S and carries L bytes, the receiver answers with ACK = S + L, as long as everything before S has already arrived.

Example with SEQ = 1001 and 500 bytes of data:

```text
Data bytes are numbered 1001 to 1500.
Next expected byte = 1001 + 500 = 1501.
Receiver sends ACK = 1501.
```

**Cumulative ACK** means one ACK number covers everything before it. ACK = 1501 says, "I have every byte up to and including 1500." It says nothing about bytes after any gap.

This design has a helpful side effect: losing an ACK usually does no harm. Later ACKs repeat the same information and more.

```text
Sender                                  Receiver
  |--- SEQ=1     LEN=1000 ---------------->|
  |--- SEQ=1001  LEN=1000 ---------------->|
  |            X<------------- ACK=1001 ---|  this ACK is lost
  |<------------------------- ACK=2001 ----|  covers both segments
```

Other facts you should know:

- SYN and FIN each use up one sequence number, even with no data. See [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md).
- A pure ACK (no data) uses no sequence number and is never retransmitted by itself.
- **Piggybacking:** an ACK can ride on a data segment going the other way, which saves a packet.
- TCP has no negative ACK. The closest things are a repeated ACK and SACK.
- Sequence numbers start at a random value. Wireshark and tcpdump show **relative** numbers (starting near 1) to make reading easy.

**Important:** an ACK means the receiver's TCP has the data in its kernel buffer. It does not mean the application has read it.

### RTT estimate and the retransmission timeout (RTO)

**RTO (Retransmission Timeout)** is how long the sender waits for an ACK before it resends the oldest unacknowledged data.

**Analogy:** you text a friend who usually replies in about a minute. You start to worry after three or four minutes, not after 10 seconds. If that friend is sometimes very fast and sometimes slow, you wait longer before you worry. You learn from past replies.

**Where the analogy stops:** you cannot tell if your friend is ignoring you or if your text was lost. TCP cannot tell either, so it resends either way, and the receiver drops any duplicate.

The RTO must adapt. Inside a data centre, RTT may be 0.2 ms. To another continent, it may be 200 ms. It also changes over time.

The standard algorithm (RFC 6298) keeps two values for each connection:

- **SRTT (Smoothed RTT):** a running average of RTT.
- **RTTVAR (RTT variance):** how much RTT jumps around. It is a smoothed mean deviation.

**First measurement R:**

```text
SRTT   = R
RTTVAR = R / 2
RTO    = SRTT + max(G, 4 * RTTVAR)     G = clock granularity
```

**Each later measurement R':** (update RTTVAR first, using the old SRTT)

```text
RTTVAR = (3/4) * RTTVAR + (1/4) * |SRTT - R'|
SRTT   = (7/8) * SRTT   + (1/8) * R'
RTO    = SRTT + max(G, 4 * RTTVAR)
```

Rules around the timer:

1. The sender uses one timer per connection. It starts when data is sent and no timer is running.
2. When an ACK covers new data, the timer restarts. When nothing is left in flight, the timer stops.
3. When the timer expires, the sender resends the earliest unacknowledged segment.
4. It then **doubles the RTO**. This is **exponential backoff**. If the network is broken, sending faster makes things worse.
5. RFC 6298 says the RTO should be at least 1 second. The initial RTO, before any measurement, is 1 second.

**Karn's algorithm:** do not take an RTT sample from a segment that was retransmitted. If an ACK arrives after a resend, TCP cannot tell which copy it answers. The sample could be far too small or too large. TCP keeps the backed-off RTO until a fresh sample arrives. The timestamp option (see Going Deeper) removes this doubt.

**Fuller truth:** real systems differ from the RFC numbers.

| Item | RFC 6298 | Linux | Windows 11 |
|---|---|---|---|
| Initial RTO | 1 s | 1 s | 1 s (`netsh` shows `Initial RTO: 1000`) |
| Minimum RTO | 1 s (SHOULD) | 200 ms | Version and profile dependent |
| Maximum RTO | At least 60 s if capped | 120 s | Version dependent |

Linux also uses a modified formula. Its variance term has a floor of about 200 ms, so a typical RTO is close to RTT + 200 ms. That is why `ss` often shows `rto:204` on a LAN.

An RTO event is expensive for a second reason. After a timeout, the congestion window usually falls to one segment. See [TCP Congestion Control](06-tcp-congestion-control.md).

### Duplicate ACK and fast retransmit

Waiting for an RTO wastes time. On a 20 ms path with a 200 ms RTO, the sender would sit idle for ten times the RTT. TCP has a faster signal.

**Duplicate ACK (dup ACK):** an ACK that repeats the same ACK number, carries no data, and does not change the window, while data is still in flight.

**Why it happens:** when a segment arrives after a gap, the receiver cannot move its ACK number forward. RFC 5681 says it should immediately send an ACK that repeats the old number. Each dup ACK says, "something arrived, but I am still missing this byte."

**Analogy:** you wait at a deli counter and the worker keeps calling, "Ticket 8? Ticket 8? Ticket 8?" as other tickets pile up. After a few calls, everyone knows ticket 8 is missing.

**Where the analogy stops:** the receiver is not repeating itself to be noisy. Each dup ACK is triggered by a real new segment arriving, so it also proves that data is still getting through. That is why TCP treats dup ACK loss as milder than a timeout.

**Fast retransmit (RFC 5681):**

1. The sender counts dup ACKs for the same ACK number.
2. On the **third duplicate** (four ACKs in total with that number), it resends the missing segment at once, without waiting for the RTO.
3. It then enters **fast recovery**, a mode owned by [TCP Congestion Control](06-tcp-congestion-control.md).

**Why three, not one?** IP can reorder packets. One or two dup ACKs may mean a segment took a slightly different path and will arrive soon. Three is a practical balance between speed and false alarms.

**Limit:** fast retransmit needs later segments to arrive. If the **last** segment of a burst is lost, nothing triggers dup ACKs. This is **tail loss**, and classic TCP has to wait for the RTO. Going Deeper covers the modern fix.

**Fuller truth:** Linux starts with a threshold of 3 (`net.ipv4.tcp_reordering`) but raises it when it sees reordering on a connection. Modern Linux also uses RACK, which works with time rather than counting.

### SACK (Selective Acknowledgment)

Cumulative ACKs have a blind spot. They only name the first hole. If several segments are lost in one window, the sender learns about one hole per RTT, or it may resend data the receiver already has.

**SACK (RFC 2018)** lets the receiver list the blocks it holds beyond the hole.

**Analogy:** a shopping order arrives with items missing. A cumulative ACK says, "I received items up to number 3." SACK adds, "I also have items 6 to 8." The shop now knows to resend only items 4 and 5.

**Where the analogy stops:** with SACK, the receiver is allowed to throw away the extra items later. So the sender must keep its copy until a cumulative ACK covers that data.

**How it is set up:**

1. Each side sends the **SACK-Permitted** option in its SYN.
2. If both sent it, either side may send **SACK** options on ACKs.
3. Each block has a left edge and a right edge. The left edge is the first byte in the block. The right edge is the first byte **after** the block.

```text
SACK-Permitted option (in SYN segments only)
+--------+--------+
| Kind=4 | Len=2  |
+--------+--------+

SACK option (N blocks, length = 2 + 8 * N bytes)
+--------+--------+
| Kind=5 | Len    |
+--------+--------+--------+--------+
|     Left edge of block 1 (32)     |
+-----------------------------------+
|     Right edge of block 1 (32)    |
+-----------------------------------+
|     ... up to block N ...         |
+-----------------------------------+
```

The options area is 40 bytes at most. So a segment can carry up to 4 SACK blocks (2 + 8 * 4 = 34 bytes). With the 10-byte timestamp option and 2 bytes of padding, only 3 blocks fit (12 + 26 = 38 bytes). The first block must describe the most recent data received.

SACK does not replace the cumulative ACK. The ACK number still moves only when the data is continuous. With SACK, the sender keeps a **scoreboard**: a map of which bytes are acknowledged, SACKed or missing. RFC 6675 defines loss recovery using this scoreboard.

SACK is on by default in Linux (`net.ipv4.tcp_sack = 1`), Windows and macOS, and most connections on the internet negotiate it.

### Sliding window

If TCP had to wait for an ACK after each segment, a 100 ms path would give only 10 segments per second. So TCP lets the sender keep many bytes in flight. The limit is the **window**.

**Sliding window:** a range of sequence numbers that the sender is allowed to have outstanding. When ACKs arrive, the left edge moves right, and the window "slides" along the byte stream.

**Analogy:** a teacher lets a student borrow up to 5 books at a time. When a book comes back, the student can take a new one. The student does not have to return all 5 first.

**Where the analogy stops:** with TCP, the limit can change on every ACK, and the sender does not count segments. It counts bytes.

The sender tracks these values (names from RFC 9293):

- **SND.UNA:** oldest byte sent but not acknowledged.
- **SND.NXT:** next byte to send.
- **SND.WND:** the window the receiver last advertised.

```text
           SND.UNA         SND.NXT         SND.UNA + SND.WND
              |               |                   |
              v               v                   v
  ------------+---------------+-------------------+-----------
   sent and   |  sent, not    |  allowed to send  |  not allowed
   acked      |  acked yet    |  right now        |  yet
              |  (in flight)  |  (usable window)  |
  ------------+---------------+-------------------+-----------
```

**The formula:**

```text
In flight      = SND.NXT - SND.UNA
Usable window  = (SND.UNA + SND.WND) - SND.NXT
```

**Worked numbers:** SND.UNA = 5001, SND.NXT = 9001, SND.WND = 10000.

```text
In flight     = 9001 - 5001           = 4000 bytes
Right edge    = 5001 + 10000          = 15001
Usable window = 15001 - 9001          = 6000 bytes
```

Now an ACK arrives. Three things can happen:

| Event | New values | Right edge | Meaning |
|---|---|---|---|
| ACK=7001, WND=10000 | UNA=7001 | 17001 | Window **slides**: app is keeping up |
| ACK=7001, WND=8000 | UNA=7001 | 15001 | Left edge moved, right edge did not: app did not read |
| ACK=5001, WND=12000 | UNA=5001 | 17001 | Window **opens**: app read data, no new data acked |

**The real sender limit** includes the network too. The **congestion window (cwnd)** is the sender's own estimate of what the network can carry.

```text
Max in flight = min(cwnd, rwnd)
```

- **rwnd** protects the receiver. This is flow control, and this chapter owns it.
- **cwnd** protects the network. This is congestion control, owned by [TCP Congestion Control](06-tcp-congestion-control.md).

**Maximum throughput from a window:**

```text
Throughput <= window / RTT
```

Window limits matter more as RTT grows:

| Window | RTT | Max throughput |
|---|---|---|
| 65,535 bytes | 1 ms | 524 Mbps |
| 65,535 bytes | 10 ms | 52.4 Mbps |
| 65,535 bytes | 50 ms | 10.5 Mbps |
| 65,535 bytes | 100 ms | 5.2 Mbps |
| 6,250,000 bytes | 50 ms | 1 Gbps |

**BDP (Bandwidth-Delay Product):** link speed times RTT. It is the amount of data the path itself can hold. To fill the link, the window must be at least the BDP.

```text
BDP = 1 Gbps * 50 ms = 1,000,000,000 bit/s * 0.05 s = 50,000,000 bits
                     = 6,250,000 bytes (6.25 MB)
```

### Receive window and flow control

**Flow control** stops a fast sender from overrunning a slow receiver. Without it, the receiver would have to throw away data it has no room for, and the sender would waste time resending.

**Receive window (rwnd):** the free space the receiver has for this connection. It goes in the Window field of **every** segment.

```text
+-----------------------------------------------------+
|              Receive buffer (65,536 bytes)          |
+------------------------------+----------------------+
|  received, waiting for app   |    free space        |
|  20,000 bytes                |    45,536 bytes      |
+------------------------------+----------------------+
                                  ^
                                  rwnd = 45,536
```

```text
rwnd = buffer size - data waiting for the application
```

**Analogy:** a sink with the tap on and the drain open. The tap is the sender. The drain is the application reading data. The sink is the buffer. If the drain is slow, the sink fills, and you must turn down the tap.

**Where the analogy stops:** the tap cannot see the sink. TCP sends the water level back in ACKs, so the sender's view is always one trip old. Data already in flight can still arrive.

**Backpressure:** when a slow consumer makes a fast producer slow down. TCP gives you this for free:

1. The application reads slowly.
2. The receive buffer fills, and rwnd falls.
3. The sender stops at the window limit.
4. The sender's own send buffer fills.
5. The sender application's `write()` call blocks, or returns "would block" on a non-blocking socket.

**Fuller truth:** rwnd is not exactly the free buffer bytes. The kernel keeps part of the buffer for overhead, and it rounds the value it advertises. A modern kernel also grows the buffer as needed (see Going Deeper).

### Window scaling

The Window field has 16 bits, so its maximum value is 65,535 bytes. That was fine in the early 1980s. It is too small for a modern high-speed link: from the table above, 64 KB over 50 ms gives only about 10 Mbps.

**Window scaling (RFC 7323)** adds a multiplier. Each side announces a **shift count** in its SYN. The true window is the field value shifted left by that count.

```text
Real window = Window field * 2^shift
```

**Analogy:** a price list says "All prices in thousands." The printed number stays small, but its meaning is larger.

**Where the analogy stops:** the scaling is a power of two, each direction has its own scale, and it cannot change after the handshake.

```text
Window Scale option (SYN and SYN-ACK only)
+--------+--------+---------+
| Kind=3 | Len=3  | Shift   |
+--------+--------+---------+
```

Rules:

1. Each side sends the option in its SYN only. It is never sent later.
2. Scaling is active only if **both** sides sent it. If one side did not, neither scales.
3. The Window field in the SYN and SYN-ACK is **never** scaled.
4. The maximum shift is 14. So the maximum window is 65,535 * 16,384 = 1,073,725,440 bytes, about 1 GiB.
5. Each direction can use a different shift.

**Why stop at 14?** The sequence space is 2^32. TCP must be able to tell old bytes from new bytes, so the window has to stay well below half of that space.

**Example:** the field says 502 and the shift is 7.

```text
Real window = 502 * 2^7 = 502 * 128 = 64,256 bytes
```

**Trap:** captures show this field raw. tcpdump prints `win 502`. If Wireshark did not see the handshake, it does not know the shift, so it cannot show the real value. It then shows the scale factor as unknown. Always capture the SYN when you study window problems.

**Precision cost:** with shift 7, the receiver can only signal multiples of 128 bytes. So a free space under 128 bytes is advertised as 0. This does no harm in practice.

### Zero window and the persist timer

If the application stops reading, the buffer fills and the receiver advertises **Window = 0**. This is a **zero window**.

The sender must stop sending new data. But now there is a trap:

1. The receiver sends Window = 0. The sender stops.
2. The application reads, so buffer space opens up.
3. The receiver sends a **window update**, an ACK with a new window.
4. That ACK is lost. Pure ACKs are not retransmitted.
5. The sender waits for a window update. The receiver waits for data. Both wait forever.

**The fix: the persist timer.** While the peer's window is zero, the sender runs a timer and sends a **zero window probe** when it fires. The receiver must answer with an ACK that shows its current window.

- The first probe comes after about one RTO.
- The gap between probes grows exponentially.
- RFC 9293 says the connection must stay open while the receiver keeps answering. A zero window is not a failure by itself.

**Analogy:** a restaurant says it is full and will call you. You worry they may lose your number, so you phone them from time to time and ask, "Any table yet?"

**Where the analogy stops:** a probe contains at most a tiny amount of data, and the answer always comes from the kernel. The application does not have to do anything.

**Fuller truth:** RFC 9293 describes a probe with at least one byte of new data. Linux normally sends a zero-length segment with an old sequence number (SND.UNA - 1), which forces the peer to send an ACK. Both work.

**Silly window syndrome** is the related problem of tiny windows. Going Deeper covers it.

### Keepalive

An idle TCP connection sends **nothing**. If the peer vanishes (power loss, cable pulled), the other side never finds out. It stays in ESTABLISHED forever. This is a **half-open connection**.

**TCP keepalive** sends a small probe after a long idle period.

**Analogy:** during a long silence on a phone call, you ask, "Are you still there?"

**Where the analogy stops:** the person answering is the remote kernel, not the application. A frozen program whose host is healthy still answers. So keepalive proves the machine and path work, not the application.

**How it works:**

1. The connection is idle for `idle` seconds.
2. The sender sends a probe. It is an ACK with SEQ = SND.NXT - 1, usually with no data. The old sequence number forces the peer to answer with an ACK.
3. **If an ACK returns:** the peer is alive. The idle timer resets.
4. **If an RST returns:** the peer rebooted or lost the connection. TCP closes it with an error.
5. **If nothing returns:** TCP sends more probes, `interval` seconds apart. After `count` failures, it closes the connection with a timeout error.

**Time to detect a dead peer:**

```text
Detect time = idle + (interval * count)
```

**Rules and defaults:**

- RFC 9293 makes keepalive optional. It must be off by default, and if used, the default idle time must be at least 2 hours.
- An application turns it on per socket with `SO_KEEPALIVE`.

| Setting | Linux default | Windows default |
|---|---|---|
| Idle time | 7200 s (`tcp_keepalive_time`) | 2 hours (`KeepAliveTime` registry value) |
| Interval | 75 s (`tcp_keepalive_intvl`) | 1 s (`KeepAliveInterval` registry value) |
| Probe count | 9 (`tcp_keepalive_probes`) | 10 |

So Linux needs 7200 + 75 * 9 = 7875 seconds (about 2 hours 11 minutes) to detect a dead peer with the defaults.

**Why keepalive is often really about firewalls:** stateful firewalls and [NAT](../04-ip-addressing/09-nat-and-pat.md) devices remove idle entries from their tables. After that, the next packet is dropped, and the connection dies without warning. RFC 5382 requires NAT devices to keep established TCP entries for at least 2 hours 4 minutes, but real timers are often much shorter and vary by product. For example, an AWS Network Load Balancer defaults to 350 seconds, and Cisco ASA defaults to 1 hour for TCP connections. A 2-hour keepalive will not help in those cases. Set the idle time below the shortest timer on the path.

**Keepalive vs other things with similar names:**

| Thing | Layer | Purpose |
|---|---|---|
| TCP keepalive | Transport | Check that the peer and path still work, and keep state alive |
| HTTP keep-alive | Application | Reuse one connection for many requests. See [HTTP Basics](../07-application-protocols/05-http-basics.md) |
| Application heartbeat | Application | Prove that the program itself still responds |

### Nagle's algorithm

**The problem:** an application that writes one byte at a time produces one packet per byte. Each has 40 bytes of IPv4 and TCP headers. That is 41 bytes on the wire to carry 1 byte. These are called **tinygrams**.

**Nagle's algorithm (RFC 896)** reduces them. It runs on the **sender**.

```text
When the application gives TCP new data:

  if a full MSS is ready and the window allows it:
      send it now
  else if there is unacknowledged data in flight:
      hold the small data in the buffer
  else:
      send it now
```

In plain words: **only one small segment may be in flight at a time.** While waiting for its ACK, small writes collect together. When the ACK arrives, TCP sends all of them as one segment.

**Analogy:** a shuttle bus has one small van. It goes out with whoever is waiting. While it is away, new passengers gather at the stop. When it returns, they all ride together. A full coach always leaves at once.

**Where the analogy stops:** the "return" is an ACK, so the pace depends on the path. On a very fast LAN, Nagle barely holds anything. There is no timer in Nagle: it waits for an ACK and nothing else.

RFC 9293 says hosts should implement Nagle, and must give applications a way to turn it off for a connection.

### Delayed ACK

**The problem:** a pure ACK costs a packet. If the receiver waits a moment, it may be able to piggyback the ACK on response data, or cover two segments with a single ACK.

**Delayed ACK** does this. It runs on the **receiver**.

Rules (RFC 1122, RFC 5681 and RFC 9293):

1. The delay must be under 0.5 seconds.
2. An ACK should be sent for at least every second full-sized segment.
3. Out-of-order segments, and segments that fill a hole, should be ACKed at once.

**Real values vary:** Linux adapts between about 40 ms and 200 ms, and it uses a "quick ACK" mode at the start of a connection. Windows uses 200 ms by default.

**Analogy:** a waiter takes your drink order but does not walk to the kitchen right away, in case you are about to order food too. One trip then covers both.

**Where the analogy stops:** the waiter can see you thinking. TCP cannot see whether the application will reply. It only sets a timer and hopes.

### When Nagle and delayed ACK meet

Each one is sensible alone. Together they can deadlock until a timer fires. The classic pattern is **write-write-read**: the application sends a small header, then a small body, then waits for a reply.

```text
Client (Nagle on)                      Server (delayed ACK on)
   |                                         |
   | write(200 bytes): nothing in flight     |
   |--- SEQ=1 LEN=200 --------------------->|  no reply yet: the
   |                                         |  app needs the body,
   | write(300 bytes): 200 bytes unacked     |  so TCP delays the ACK
   | Nagle HOLDS this data                   |
   |                                         |
   |      client waits for an ACK            |
   |      server waits for more data         |
   |                                         |
   |              (40 ms on Linux)           |
   |                                         |
   |<------------------- ACK=201 ------------|  delayed ACK timer
   |--- SEQ=201 LEN=300 ------------------->|
   |<------------ SEQ=1 ACK=501 (response) --|
```

Neither side is wrong. Each one waits politely for the other, until the delayed ACK timer ends the standoff.

**Why it needs two small writes:** one write of 500 bytes goes out in one piece. A write of at least one full MSS also goes out at once. Only a small second write is held back.

### TCP_NODELAY and other fixes

**TCP_NODELAY** is a socket option that turns off Nagle for one connection. Small writes then go out at once, as long as the windows allow.

```text
// C on Linux or Windows
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, (char *)&one, sizeof(one));

# Python on Linux or Windows
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

**What it does not do:**

- It does not turn off delayed ACK. That is on the other host.
- It does not bypass rwnd or cwnd.
- It does not make the network faster. It only removes one reason to wait.

**Which fix to choose:**

| Fix | What it does | When to use it |
|---|---|---|
| Combine writes | Build the whole message, then make one `write()` or `writev()` | **Best fix.** Also saves system calls |
| `TCP_NODELAY` | Turns off Nagle | Request-response, RPC, games, live data |
| `TCP_CORK` (Linux) | Holds partial segments until uncorked, 200 ms at most | Sending headers and a file together |
| `MSG_MORE` (Linux) | Tells `send()` that more data is coming | One-call version of corking |
| `TCP_QUICKACK` (Linux) | Asks for immediate ACKs on the receiver. Not permanent, so it must be set again | Rarely. When you cannot change the sender |

Many systems already set it for you. Go sets `TCP_NODELAY` on TCP connections by default. nginx has `tcp_nodelay on` by default for keep-alive connections. OpenSSH sets it for interactive sessions.

**Cost:** with Nagle off and many tiny writes, you can send many tiny packets. Fix that by buffering in the application.

## Example

### Example 1: a lost segment, recovered by fast retransmit and SACK

**Setup:** sender 10.0.0.20 port 51514, receiver 10.0.0.10 port 5001, RTT 20 ms. We use an MSS of 1000 bytes to keep the numbers easy. SACK is on. The second segment is lost.

```text
Sender 10.0.0.20:51514                Receiver 10.0.0.10:5001
      |                                        |
      |--- SEQ=1     LEN=1000 ---------------->| got 1-1000
      |<-------------------------- ACK=1001 ---|
      |                                        |
      |--- SEQ=1001  LEN=1000 ------X          | lost on the way
      |--- SEQ=2001  LEN=1000 ---------------->| hole at 1001
      |<--- ACK=1001 SACK 2001-3001 -----------| dup ACK 1
      |--- SEQ=3001  LEN=1000 ---------------->|
      |<--- ACK=1001 SACK 2001-4001 -----------| dup ACK 2
      |--- SEQ=4001  LEN=1000 ---------------->|
      |<--- ACK=1001 SACK 2001-5001 -----------| dup ACK 3
      |                                        |
      |   third dup ACK: resend now, no timer  |
      |--- SEQ=1001  LEN=1000 ---------------->| hole filled
      |<-------------------------- ACK=5001 ---|
```

Step by step:

1. Segment 1 arrives. ACK = 1 + 1000 = 1001.
2. The segment at SEQ 1001 is lost.
3. SEQ 2001 arrives, but the next expected byte is 1001. The receiver keeps it in its out-of-order queue and at once sends ACK=1001 with SACK 2001-3001.
4. Two more segments arrive, and two more dup ACKs go back. The SACK block grows to 2001-5001.
5. The sender has now seen four ACKs that say 1001: the original and three duplicates. It resends only the segment at 1001.
6. The hole is filled. The receiver holds bytes 1 to 5000, so it jumps to ACK=5001 in a single step.

Recovery took about one RTT (20 ms). Waiting for an RTO on Linux would take at least 200 ms. Without SACK, the sender would not know what was safe, and might resend bytes 2001-5000 as well.

The same event in a real capture, taken on the receiver. This one has a 1448-byte MSS. Options other than SACK are trimmed.

```bash
# Linux: capture on the receiver
sudo tcpdump -ni eth0 'tcp port 5001'
```

```text
11:02:10.200100 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 1:1449, ack 1, win 502, length 1448
11:02:10.200120 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 1449, win 1024, length 0
11:02:10.200400 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 2897:4345, ack 1, win 502, length 1448
11:02:10.200415 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 1449, win 1024, options [sack 1 {2897:4345}], length 0
11:02:10.200700 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 4345:5793, ack 1, win 502, length 1448
11:02:10.200715 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 1449, win 1024, options [sack 1 {2897:5793}], length 0
11:02:10.201000 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 5793:7241, ack 1, win 502, length 1448
11:02:10.201015 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 1449, win 1024, options [sack 1 {2897:7241}], length 0
11:02:10.221100 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 1449:2897, ack 1, win 502, length 1448
11:02:10.221120 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 7241, win 1024, length 0
```

How to read it:

- Bytes 1449:2897 never appear in order. This is the hole.
- Three ACKs after the first repeat `ack 1449`: those are the three dup ACKs.
- The retransmission arrives 20.1 ms after the third dup ACK left. That is about one RTT, so this was fast retransmit, not an RTO.
- `ack 7241` jumps past everything at once.

In Wireshark, these filters find the same events. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

```text
tcp.analysis.duplicate_ack
tcp.analysis.fast_retransmission
tcp.analysis.retransmission
tcp.analysis.out_of_order
tcp.options.sack_le
```

### Example 2: calculating the RTO

Follow RFC 6298 with four RTT samples. All values are in milliseconds, and we ignore clock granularity G.

**Sample 1, R = 100:**

```text
SRTT   = 100
RTTVAR = 100 / 2 = 50
RTO    = 100 + 4 * 50 = 300
```

**Sample 2, R' = 120:**

```text
RTTVAR = 0.75 * 50  + 0.25 * |100 - 120|  = 37.5 + 5     = 42.5
SRTT   = 0.875 * 100 + 0.125 * 120        = 87.5 + 15    = 102.5
RTO    = 102.5 + 4 * 42.5                                = 272.5
```

**Sample 3, R' = 300 (a sudden delay spike):**

```text
RTTVAR = 0.75 * 42.5 + 0.25 * |102.5 - 300| = 31.88 + 49.38 = 81.25
SRTT   = 0.875 * 102.5 + 0.125 * 300        = 89.69 + 37.5  = 127.19
RTO    = 127.19 + 4 * 81.25                                 = 452.19
```

Sample 4, R' = 100, works the same way.

| Sample | R (ms) | SRTT (ms) | RTTVAR (ms) | Computed RTO (ms) |
|---|---|---|---|---|
| 1 | 100 | 100.00 | 50.00 | 300.00 |
| 2 | 120 | 102.50 | 42.50 | 272.50 |
| 3 | 300 | 127.19 | 81.25 | 452.19 |
| 4 | 100 | 123.79 | 67.73 | 394.73 |

What this shows:

- One spike moved SRTT by about 25 ms, but it pushed the RTO up by about 180 ms. The variance term reacts strongly, on purpose.
- The RTO falls slowly afterwards, so TCP stays cautious for a while.
- A real RFC-following stack would apply the 1 second floor, so its RTO would be 1000 ms for all four. Linux would use its own formula with a much lower floor.

**Backoff on Linux**, starting from a 200 ms RTO with no ACKs at all:

```text
Retry:    1    2    3    4    5    6     7     8     9     10   11...
Wait(s): 0.2  0.4  0.8  1.6  3.2  6.4  12.8  25.6  51.2  102.4  120...
```

### Example 3: a slow transfer caused by a small window

**Setup:** 1 Gbps link, 50 ms RTT, sender 10.0.0.20, receiver 10.0.0.10. A receiver application has fixed its socket buffer at 64 KB.

**Step 1: find the limit.**

```text
Max throughput = 65,535 bytes * 8 / 0.05 s
               = 10,485,600 bit/s = 10.5 Mbps
```

**Step 2: find the needed window.**

```text
BDP = 1 Gbps * 0.05 s / 8 = 6,250,000 bytes
```

**Step 3: find the needed shift.**

```text
6,250,000 / 65,535 = 95.4
2^6 = 64 is too small. 2^7 = 128 is enough.
Shift 7 gives a max of 65,535 * 128 = 8,388,480 bytes.
```

**Step 4: confirm with a test.** `-w` sets the socket buffers, so the real window can differ a little.

```bash
# Linux: receiver on 10.0.0.10
iperf3 -s

# Linux: sender with a 64 KB limit, then with autotuning
iperf3 -c 10.0.0.10 -t 10 -w 64K
iperf3 -c 10.0.0.10 -t 10
```

```text
# With -w 64K
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  12.4 MBytes  10.4 Mbits/sec    0    sender

# With autotuning
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.05 GBytes   900 Mbits/sec   14    sender
```

**Step 5: check the handshake.** Both sides offer `sackOK` and `wscale 7` here, so scaling is working. Notice the raw `win 502` on the last line. With shift 7, it means 64,256 bytes.

```text
IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [S], seq 3405691582, win 64240, options [mss 1460,sackOK,TS val 811000 ecr 0,nop,wscale 7], length 0
IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [S.], seq 1122334455, ack 3405691583, win 65160, options [mss 1460,sackOK,TS val 422000 ecr 811000,nop,wscale 7], length 0
IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], ack 1, win 502, options [nop,nop,TS val 811001 ecr 422000], length 0
```

**Check the settings:**

```bash
# Linux
sysctl net.ipv4.tcp_window_scaling net.ipv4.tcp_sack net.ipv4.tcp_timestamps
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

```text
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_sack = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_rmem = 4096        131072  6291456
net.ipv4.tcp_wmem = 4096        16384   4194304
```

The three `tcp_rmem` and `tcp_wmem` numbers are minimum, default and maximum in bytes. They vary with kernel version and system memory.

```powershell
# Windows
netsh interface tcp show global
```

```text
TCP Global Parameters
----------------------------------------------
Receive-Side Scaling State          : enabled
Receive Window Auto-Tuning Level    : normal
RFC 1323 Timestamps                 : allowed
Initial RTO                         : 1000
```

Output is trimmed, and the fields vary by Windows version. If the auto-tuning level says `disabled`, the receive window is stuck near 64 KB. RFC 7323 replaced RFC 1323, but Windows keeps the old label.

### Example 4: reading a zero window

**Setup:** the receiver application on 10.0.0.10 stops reading. Shift is 7. Data comes from 10.0.0.20. Options are trimmed. The capture was taken on the sender.

```text
10:15:02.098000 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 127425, win 23, length 0
10:15:02.098100 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 127425:128873, ack 1, win 502, length 1448
10:15:02.098150 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 128873:130321, ack 1, win 502, length 1448
10:15:02.101300 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 130321, win 0, length 0
10:15:02.305000 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], ack 1, win 502, length 0
10:15:02.308100 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 130321, win 0, length 0
10:15:02.713000 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], ack 1, win 502, length 0
10:15:02.716100 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 130321, win 0, length 0
10:15:03.529000 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], ack 1, win 502, length 0
10:15:03.532100 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 130321, win 0, length 0
10:15:03.702000 IP 10.0.0.10.5001 > 10.0.0.20.51514: Flags [.], ack 130321, win 1024, length 0
10:15:03.702100 IP 10.0.0.20.51514 > 10.0.0.10.5001: Flags [.], seq 130321:131769, ack 1, win 502, length 1448
```

How to read it:

1. `win 23` means 23 * 128 = 2944 bytes are free. The sender sends two segments (2896 bytes).
2. Only 48 bytes remain. With shift 7, that rounds down to `win 0`.
3. The sender sends probes at about +204 ms, +408 ms and +816 ms from the previous one. That is the exponential backoff of the persist timer. These are Linux-style probes with no data.
4. Each probe gets `win 0` back.
5. The application wakes up and reads. The receiver sends `win 1024` (131,072 bytes) by itself. This is the **window update**.
6. Data flows again, from exactly the byte where it stopped: `seq 130321`.

**Prove it is the application, not the network.** On Linux, `ss` shows the queues.

```bash
# Linux, on the receiver
ss -tn sport = :5001
```

```text
State   Recv-Q   Send-Q   Local Address:Port   Peer Address:Port
ESTAB   130320   0        10.0.0.10:5001       10.0.0.20:51514
```

```bash
# Linux, on the sender
ss -tno dport = :5001
```

```text
State   Recv-Q   Send-Q    Local Address:Port    Peer Address:Port
ESTAB   0        2097152   10.0.0.20:51514       10.0.0.10:5001    timer:(persist,812ms,0)
```

- A high **Recv-Q** on the receiver means data is sitting in the kernel, and the application is not reading it.
- A high **Send-Q** on the sender means data is waiting to be acknowledged or sent.
- `timer:(persist,...)` confirms that the sender is probing.

### Example 5: the 40 ms delay

**Setup:** an RPC client on Linux, 10.0.0.20, talks to a Linux server, 10.0.0.10 port 8080. RTT is 1 ms. The client writes a 200-byte header and then a 300-byte body, with two `send()` calls. Capture on the client.

```text
14:20:01.000000 IP 10.0.0.20.40110 > 10.0.0.10.8080: Flags [P.], seq 1:201, ack 1, win 502, length 200
14:20:01.041000 IP 10.0.0.10.8080 > 10.0.0.20.40110: Flags [.], ack 201, win 510, length 0
14:20:01.041050 IP 10.0.0.20.40110 > 10.0.0.10.8080: Flags [P.], seq 201:501, ack 1, win 502, length 300
14:20:01.042100 IP 10.0.0.10.8080 > 10.0.0.20.40110: Flags [P.], seq 1:101, ack 501, win 510, length 100
```

**The clue:** a gap of 41 ms between the first data and its ACK, on a path with a 1 ms RTT. An ACK that takes about 40 ms on Linux, or about 200 ms on Windows, is a delayed ACK timer.

| Step | Without fix | With one write or TCP_NODELAY |
|---|---|---|
| Header sent | 0 ms | 0 ms |
| Body sent | 41 ms | 0 ms |
| Response back | 42 ms | About 1 ms |
| Requests per second on one connection | About 24 | About 950 |

**Fix 1 (best): send it all in one call.** Fix 2 is to set `TCP_NODELAY` on the client, as shown earlier.

```text
# Python on Linux or Windows
s.sendall(header + body)
```

**Note:** you may not see this on the first request. Linux quick ACK mode often sends immediate ACKs early in a connection. The delay can appear later on the same connection, which makes it feel random.

### Example 6: tuning keepalive

**Goal:** detect a dead peer within 2 minutes, and stay inside a firewall idle timer of 5 minutes.

```text
Choose idle = 60 s, interval = 10 s, count = 5
Detect time = 60 + (10 * 5) = 110 s
```

Per socket is better than system-wide, because it only changes the connections you own.

```text
# Python on Linux
s.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 5)

# Python on Windows (on/off, idle in ms, interval in ms)
s.ioctl(socket.SIO_KEEPALIVE_VALS, (1, 60000, 10000))
```

The system-wide Linux defaults (`sysctl net.ipv4.tcp_keepalive_time` and friends) apply only to sockets that have turned on `SO_KEEPALIVE`. Check a live socket with `ss -o`:

```bash
# Linux
ss -tno state established dport = :5001
```

```text
Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
0       0       10.0.0.20:51514     10.0.0.10:5001     timer:(keepalive,52sec,0)
```

### Live checks on a real host

```bash
# Linux: per-connection detail
ss -tin dst 10.0.0.10
```

```text
State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
ESTAB  0       0       10.0.0.20:51514     10.0.0.10:5001
         ts sack cubic wscale:7,7 rto:204 rtt:3.2/0.8 ato:40 mss:1448 pmtu:1500 rcvmss:536 advmss:1448 cwnd:10 send 36.2Mbps
```

| Field | Meaning |
|---|---|
| `ts sack` | Timestamps and SACK were negotiated |
| `cubic` | Congestion control algorithm |
| `wscale:7,7` | Shift for sending, shift for receiving |
| `rto:204` | Current RTO in ms |
| `rtt:3.2/0.8` | SRTT / RTTVAR in ms |
| `ato:40` | Delayed ACK timeout estimate in ms |
| `mss:1448` | 1500 - 20 (IP) - 20 (TCP) - 12 (timestamps) |
| `send 36.2Mbps` | cwnd * MSS * 8 / SRTT |

```bash
# Linux: system-wide counters since boot
nstat -az TcpOutSegs TcpRetransSegs TcpExtTCPTimeouts TcpExtTCPFastRetrans TcpExtTCPLossProbes TcpExtTCPToZeroWindowAdv
```

```text
#kernel
TcpOutSegs                      3901220            0.0
TcpRetransSegs                  8214               0.0
TcpExtTCPTimeouts               96                 0.0
TcpExtTCPLossProbes             1204               0.0
TcpExtTCPFastRetrans            5610               0.0
TcpExtTCPToZeroWindowAdv        37                 0.0
```

```powershell
# Windows: system-wide counters
netstat -s -p tcp
```

```text
TCP Statistics for IPv4

  Segments Sent                       = 3901220
  Segments Retransmitted              = 8214
```

Here, the retransmission rate is 8214 / 3901220 = 0.21%. A rough rule says that sustained rates near 1% or higher deserve a look, but that is a rule of thumb, not a standard. Many fast retransmits with few timeouts is healthier than the reverse. For capture skills, see [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

## Going Deeper

### Timestamps, better RTT and PAWS

The **TCP Timestamps option (RFC 7323)** adds two values to each segment.

```text
Timestamps option
+--------+--------+-------------------+-------------------+
| Kind=8 | Len=10 | TSval (32 bits)   | TSecr (32 bits)   |
+--------+--------+-------------------+-------------------+
```

- **TSval:** the sender's clock value.
- **TSecr:** the echo of the last TSval it received.

**Benefit 1, better RTT.** Each ACK echoes a time, so the sender measures RTT without ambiguity, even after a retransmission. This solves the problem behind Karn's algorithm.

**Benefit 2, PAWS (Protection Against Wrapped Sequences).** The sequence number has 32 bits, so it repeats after 4 GiB. At 10 Gbps, that takes only about 3.4 seconds. A very old delayed segment could then look valid. PAWS rejects a segment whose timestamp is older than the latest one seen.

Timestamps cost 12 bytes of header in each segment, including padding. That is why the MSS drops from 1460 to 1448. Linux enables them by default. Recent Windows builds show `allowed` in `netsh`, which means the host can use them if the peer asks. Older Windows versions may show `disabled`.

### D-SACK and spurious retransmissions

A **spurious retransmission** is one that was not needed. The original was only delayed, or its ACK was. It wastes bandwidth, and worse, TCP may cut its sending rate for no reason.

- **D-SACK (RFC 2883):** the receiver reports a duplicate segment by using the first SACK block to name data it had already received. The sender learns that its retransmit was unneeded.
- **F-RTO (RFC 5682):** after a timeout, the sender checks whether the next ACKs prove the RTO was spurious. If so, it can undo its reaction.

### Tail loss and RACK-TLP

**TLP (Tail Loss Probe)** deals with tail loss. If ACKs stop and data is still in flight, the sender sends a probe after about 2 RTTs instead of waiting for the full RTO. The probe is either new data or the last segment again. The ACK for it includes SACK information that reveals the loss, and normal fast recovery can then run.

**RACK (Recent ACKnowledgment)** changes loss detection from counting to time. Instead of counting three dup ACKs, it says, "A segment sent well before one that has already been delivered is probably lost." This handles reordering, lost retransmissions and small flights better than a fixed count.

RFC 8985 defines the two together as RACK-TLP. Current Linux kernels use it by default, so "three dup ACKs" is the classic answer, not the full story. Mention this in an interview if you want to stand out.

### Silly window syndrome

**Silly window syndrome (SWS)** is what happens when the window opens by tiny amounts and data moves in tiny segments. Example: the application reads 1 byte. The receiver advertises 1 byte. The sender sends a 41-byte packet to carry 1 byte. Throughput collapses.

**Receiver side fix.** Do not advertise a small increase. Keep the window closed until it can grow by the smaller of one MSS or half the buffer.

**Sender side fix.** Do not send a small segment just because the window allows it. Wait until a full MSS fits, or until the data is a good share of the largest window ever seen, or until everything else is acknowledged. Nagle is part of this.

RFC 9293 includes both of these rules. That is why modern stacks rarely show this problem.

**A related rule:** a receiver should not move the window's right edge back to the left. That would take back permission for bytes the sender may already have sent.

### Buffer autotuning

Modern stacks size buffers per connection, so you rarely need to set them.

- **Linux:** the receive buffer grows between the `tcp_rmem` min and max while `net.ipv4.tcp_moderate_rcvbuf = 1`.
- **Linux trap:** setting `SO_RCVBUF` by hand **turns off** autotuning for that socket, and `net.core.rmem_max` caps the value. Linux also doubles the value you pass, to leave room for overhead. A hard-coded small value is a common cause of slow transfers.
- **Windows:** controlled by the auto-tuning level (`Get-NetTCPSetting`, `netsh interface tcp show global`).

**For high-BDP paths:** raise the maximum values, not the defaults. For a 10 Gbps, 100 ms path, BDP is 125 MB, and shift 11 is needed (125,000,000 / 65,535 = 1907, and 2^11 = 2048). Linux's default maximum of about 6 MB falls well short.

**Bigger is not always better.** Huge send buffers can hold a lot of queued data. The application then cannot change its mind about what to send. `TCP_NOTSENT_LOWAT` on Linux and macOS limits the queue of unsent data. Too much queueing in the network itself is **bufferbloat**: see [TCP Congestion Control](06-tcp-congestion-control.md).

### How long TCP keeps retrying

If data is in flight and no ACK ever returns, keepalive does not help, because the connection is not idle. The retransmission timer owns this case.

On Linux, `net.ipv4.tcp_retries2` defaults to 15. With a 200 ms starting RTO, that gives a time-out of about 924.6 seconds (15.4 minutes) as a lower bound. The kernel documentation says that the real effect depends on the RTO, and the man page gives a range of about 13 to 30 minutes.

```text
Sum of first 10 waits: 0.2 + 0.4 + ... + 102.4 = 204.6 s
Then 6 waits at the 120 s cap:           6 * 120 = 720.0 s
Total:                                             924.6 s
```

This is the usual reason for "my app hung for 15 minutes when the server died." The fixes:

- Set `TCP_USER_TIMEOUT` on Linux. It limits how long sent data may stay unacknowledged.
- Set application-level timeouts. Do this anyway, because no TCP timer can see a stuck application.
- Lower `tcp_retries2`, but only with care, since it affects the entire host.

### Head-of-line blocking

TCP delivers bytes strictly in order. If one segment is lost, all later bytes wait in the kernel, even if they arrived and even if they belong to a different HTTP/2 stream. This is **head-of-line blocking**. It is a key reason [QUIC](08-quic.md) gives each stream its own ordering.

### Captures can mislead

- **TSO (TCP Segmentation Offload):** the network card cuts up large blocks. A capture on the sender can show `length 65160` on a 1500-byte MTU network. The wire still carries normal segments.
- **GRO (Generic Receive Offload):** the receiver merges segments before tcpdump sees them. A capture may show one large segment.
- **Where you capture matters.** A sender capture shows a packet that a switch later dropped. Only a receiver capture proves it arrived.

### SACK and security

SACK adds complex code in the kernel. In 2019, "SACK Panic" (CVE-2019-11477) showed that crafted SACK sequences could crash Linux kernels. The fix was to patch the kernel. Disabling SACK was only a temporary workaround, since it hurts performance.

### TCP on routers and switches

These commands affect only TCP sessions that **end on the device**, such as SSH and BGP. They do not touch traffic passing through it.

```text
! Cisco IOS: global configuration
Router(config)# ip tcp selective-ack
Router(config)# ip tcp timestamp
Router(config)# ip tcp window-size 65535
Router(config)# service tcp-keepalives-in
Router(config)# service tcp-keepalives-out

! Cisco IOS: show TCP sessions on the device
Router# show tcp brief
```

Keepalives here clear dead VTY sessions, which matters when login lines are limited.

## Common Mistakes

- **"An ACK means the application got the data."** No. It means the remote kernel has it. A `write()` that returns only means your local kernel has it. If you need proof of processing, the application has to reply.
- **"The ACK number is the last byte received."** It is the **next byte expected**. ACK=1001 means bytes up to 1000 arrived.
- **"Three duplicate ACKs means three ACKs."** It is the original ACK plus three duplicates, so four ACKs with the same number.
- **"A duplicate ACK proves loss."** It may be reordering. That is why TCP waits for several, and why RACK uses time.
- **"SACK replaces the cumulative ACK."** It adds to it. The sender must keep SACKed data until the cumulative ACK passes it, because the receiver is allowed to discard it.
- **"Receive window and congestion window are the same thing."** The receiver sets rwnd and sends it in the header. The sender computes cwnd and never sends it. The limit is the smaller of the two.
- **"Zero window means the network is congested."** No. It means the receiver's application is not reading fast enough. It is a host problem.
- **"`win 502` means 502 bytes."** tcpdump shows the raw field. Multiply by 2 to the shift, which comes from the SYN.
- **"The window is fixed at the handshake."** The window changes on every segment. Only the **scale** is fixed at the handshake.
- **"The RTO is always at least 1 second."** That is the RFC guidance. Linux uses a 200 ms minimum, and data centre stacks may go lower.
- **"Keepalive is on by default."** TCP keepalive is off unless the application asks for it. And 2 hours is too long for most firewalls.
- **"Keepalive proves the service is healthy."** The remote kernel answers it. A stuck application still looks alive.
- **"HTTP keep-alive and TCP keepalive are the same."** The first reuses a connection. The second probes an idle one.
- **"TCP_NODELAY turns off delayed ACK."** It turns off Nagle on the local sender only.
- **"TCP_NODELAY means data goes out no matter what."** Windows still apply.
- **"Nagle waits for a timer."** Nagle has no timer. It waits for an ACK. The 40 ms comes from the other side's delayed ACK timer.
- **"Set a big SO_RCVBUF to go faster."** On Linux, this disables autotuning, and the value is capped. It often makes things worse.
- **"A retransmission in a capture always means packet loss."** It can be spurious, or the capture point may hide the truth. See "Captures can mislead."

## Interview Questions

**Q: How does TCP provide reliable delivery over an unreliable network?**

A: It numbers every byte with sequence numbers, and the receiver sends cumulative ACKs for what arrived. A checksum finds damage. If an ACK does not arrive in time, or duplicate ACKs show a gap, the sender retransmits. The receiver reorders data and removes duplicates, so the application sees a clean stream.

**Q: What is a cumulative ACK, and what is its weakness?**

A: ACK number N means all bytes before N arrived. A lost ACK does little harm, because the next one covers it. The weakness is that it reports only the first gap. The sender cannot see what arrived after it, so with many losses it learns about only one hole per RTT. SACK fixes this.

**Q: How does TCP compute the RTO?**

A: It keeps SRTT, a smoothed average, and RTTVAR, a smoothed variance. RFC 6298 uses RTO = SRTT + 4 * RTTVAR, with SRTT weighted by 1/8 and RTTVAR by 1/4. On expiry, it retransmits and doubles the RTO. Karn's algorithm says not to sample RTT from retransmitted segments. The RFC floor is 1 second, but Linux uses 200 ms.

**Q: Explain fast retransmit. Why three duplicate ACKs?**

A: When data arrives after a gap, the receiver immediately repeats its last ACK. After the third duplicate, the sender resends the missing segment without waiting for the RTO, so recovery takes about one RTT. It waits for three because IP can reorder packets, and one or two duplicates may be only reordering. Modern Linux also uses RACK, which uses time instead of a fixed count.

**Q: What is SACK, and how many blocks fit in a segment?**

A: SACK lets the receiver list ranges it received beyond the cumulative ACK, so the sender resends only the gaps. It is negotiated with SACK-Permitted in the SYN. The option space is 40 bytes, so 4 blocks fit, or 3 when timestamps are in use. The receiver may discard SACKed data, so the sender must keep it until the cumulative ACK covers it.

**Q: What is the difference between flow control and congestion control?**

A: Flow control protects the receiver. The receiver advertises rwnd in each segment. Congestion control protects the network. The sender estimates cwnd from signals like loss. In-flight data is limited to the smaller of the two.

**Q: Why was window scaling needed? Compute the throughput limit without it.**

A: The Window field is 16 bits, so the maximum is 65,535 bytes. Throughput is limited to window divided by RTT. At 100 ms, that is about 5.2 Mbps, even on a 10 Gbps link. Window scaling is negotiated in the SYN, with a shift of up to 14, for a window of about 1 GiB. To fill the link, the window must reach the bandwidth-delay product.

**Q: A server advertises a zero window. What happens next, and what would you check?**

A: The sender stops new data and starts the persist timer, sending window probes with exponential backoff. Probes are needed because the window update ACK could be lost, and that would deadlock both sides. The connection stays up as long as probes get answers. I would check the receiver: a high Recv-Q in `ss` means the application is not reading, so I would look at its CPU, its locks, and its downstream dependencies.

**Q: How does TCP keepalive work, and what does it not tell you?**

A: After an idle period, the sender sends a probe with an old sequence number, which forces an ACK. An ACK means alive, an RST means the peer lost the connection, and no reply leads to more probes and then an error. Linux defaults are 7200 s idle, 75 s interval and 9 probes. It is off unless the application sets `SO_KEEPALIVE`. It tells you nothing about application health, because the kernel answers.

**Q: You see a 40 ms delay on small requests. What is your guess, and how do you prove and fix it?**

A: Nagle on the sender and delayed ACK on the receiver. With a write-write-read pattern, the second small write waits for an ACK, and the receiver delays that ACK until its timer fires. I would prove it with tcpdump: a gap of about 40 ms before a pure ACK, on a low-RTT path. The best fix is to send everything in one write. Otherwise, set `TCP_NODELAY` on the sender.

**Q: What does TCP_NODELAY do, and should it always be on?**

A: It turns off Nagle, so small writes go out at once, within the windows. It helps request-response and real-time traffic, and most modern servers and frameworks use it. It does not change the peer's delayed ACK. If an application makes many tiny writes, it can create many tiny packets, so the application should buffer its own writes.

**Q: A server died suddenly, and a client with data in flight hung for 15 minutes. Why?**

A: Keepalive does not apply, because the connection is not idle. The retransmission timer is running with exponential backoff. On Linux, `tcp_retries2` defaults to 15, which gives about 924 seconds or more before TCP gives up. The fix is `TCP_USER_TIMEOUT`, plus application-level timeouts.

## Quick Recap

- ACK = next byte expected. It is cumulative, and it means the kernel has the data, not the application.
- RTO = SRTT + 4 * RTTVAR, with exponential backoff on expiry. Karn's algorithm skips samples from retransmitted data. The RFC floor is 1 s, and Linux uses 200 ms.
- Three duplicate ACKs trigger fast retransmit. SACK says exactly what arrived. RACK-TLP is the modern approach, and it also handles tail loss.
- The sender is limited to min(cwnd, rwnd), and throughput is at most window / RTT. To fill a link, the window must reach the BDP, which needs window scaling (shift set in the SYN, maximum 14).
- A zero window means the receiver's application is slow. The persist timer's probes prevent a deadlock.
- Keepalive is off by default, is answered by the kernel, and must be tuned below firewall idle timers.
- Nagle plus delayed ACK on a write-write-read pattern gives the classic 40 ms delay. Write once, or set `TCP_NODELAY` on the sender.
