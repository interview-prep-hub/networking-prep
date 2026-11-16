# TCP Congestion Control

**Level:** Advanced | **Prerequisites:** [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md), [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md), [Bandwidth, Latency and Performance](../01-foundations/06-network-performance-basics.md)

## In One Line

Congestion control is how a TCP sender guesses how much data the network can carry right now, by speeding up until it sees a sign of trouble and then slowing down.

## Why It Matters

The internet has no central traffic controller. Billions of TCP senders share the same links. No router tells a sender, "you may use 12 Mbps." Each sender must work out a fair speed on its own.

This is not a theory problem. In October 1986 the internet had its first "congestion collapse". Throughput on one path dropped from 32 Kbps to 40 bps. The links were busy, but they were carrying mostly retransmissions (copies of data sent again) of packets that were dropped anyway. Everyone sent more, so more was dropped.

Van Jacobson published the fix in 1988. His ideas (slow start, congestion avoidance, fast retransmit) are still the base of TCP today.

Interviewers ask about this because it connects many things:

- It explains real problems, such as "we have a 1 Gbps link but the copy runs at 5 Mbps".
- It separates people who memorised "TCP is reliable" from people who know how TCP behaves under load.
- It leads into system design topics: CDNs, QUIC, connection reuse, and why latency matters more than bandwidth for web pages.

## Simple Explanation

Imagine you run a delivery company. You send trucks to a customer over a road you cannot see.

- **The customer's warehouse** has limited space. The customer phones you and says, "I have room for 50 more boxes." This is **flow control**. It protects the receiver.
- **The road** has limited space too. Nobody phones you about the road. You only learn about traffic jams when a truck goes missing or arrives late. This is **congestion control**. It protects the network.

So you use a simple plan:

1. Send a few trucks. If they all arrive, send twice as many next time.
2. When you get close to the level that caused trouble before, add just one more truck each trip.
3. If a truck goes missing, assume a jam. Cut the number of trucks. Then slowly build up again.

Every other delivery company on the road does the same thing. Nobody talks to anybody, yet the road stays usable and each company gets a share.

**Where the analogy stops being true:** real trucks do not vanish in a traffic jam. In a network, a full router queue really does throw packets away. Also, a modern network can sometimes warn the sender before any loss happens (ECN, covered below), like a radio traffic report.

## How It Works

### Flow control vs congestion control

Both limit how fast the sender goes, but they protect different things.

| | Flow control | Congestion control |
|---|---|---|
| Protects | The receiver | The network |
| Problem it solves | Receiver cannot keep up | Links and queues are overloaded |
| Limit is called | rwnd (receive window) | cwnd (congestion window) |
| Who sets it | The receiver | The sender |
| Where it lives | Sent in every TCP header | Only in sender memory, never sent |
| How the sender learns it | Told directly by the receiver | Guessed from ACKs, loss, delay |

**ACK** means acknowledgement: the receiver's message saying "I got the data up to here".

The sender obeys both limits at once. It takes the smaller one:

```text
data the sender may have unacknowledged = min(cwnd, rwnd)
```

```text
 Sender                 Network                 Receiver
 +----------+      +----------------+      +--------------+
 | cwnd:    | ---> | routers, links | ---> | rwnd:        |
 | my guess |      | and queues     |      | free buffer  |
 | about    |      |                |      | space, told  |
 | the path |      | (say nothing)  |      | in each ACK  |
 +----------+      +----------------+      +--------------+
       ^                                          |
       +---------------- ACKs -------------------+
```

The full detail of rwnd, sliding windows and window scaling is in [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md).

### Terms you need

- **MSS (Maximum Segment Size):** the most TCP data in one packet. Commonly 1460 bytes on Ethernet (1448 when TCP timestamps are on). See [MTU, MSS and Path MTU Discovery](09-mss-mtu-and-pmtud.md).
- **RTT (Round Trip Time):** time for data to reach the other side and for the ACK to come back.
- **cwnd (congestion window):** the most unacknowledged data the sender allows itself to have in the network. Measured in bytes in the RFCs. Linux counts it in packets. This chapter uses segments for easy numbers.
- **In flight:** data that was sent but not yet acknowledged.
- **ssthresh (slow start threshold):** the cwnd value where growth changes from fast to careful.
- **BDP (Bandwidth-Delay Product):** bandwidth x RTT. This is how much data fits "in the pipe" with the link fully busy and no queue.

Roughly, the sending rate is:

```text
rate = cwnd / RTT
```

A cwnd of 100 segments, with 1460-byte segments and a 50 ms RTT, gives 146,000 bytes every 0.05 s. That is 2.92 MB/s, or 23.4 Mbps.

### How TCP detects congestion

TCP has no direct view of the network. Classic TCP reads two signals:

1. **Three duplicate ACKs.** The receiver keeps asking for the same missing data while later data arrives. One packet was probably lost, but the path still works. This is a mild signal.
2. **Retransmission timeout (RTO).** A timer ran out with no ACK for the data. This could mean many packets were lost, so it is a strong signal.

Newer signals are ECN marks (a router says "I am getting full") and delay or measured rate (used by BBR).

### Slow start

A new connection knows nothing about the path. It could be 1 Mbps or 100 Gbps. So TCP starts small and grows quickly.

1. Set cwnd to the **initial window** (IW). Current practice is 10 segments, about 14,600 bytes (RFC 6928, an experimental RFC that became normal practice). Linux has used it since kernel 2.6.39. Very old systems used 2 to 4.
2. For each ACK that covers new data, add up to 1 MSS to cwnd.
3. Every segment in a window produces an ACK, so cwnd **doubles each RTT**.
4. Stop when cwnd reaches ssthresh, or when loss happens.

```text
Sender                                   Receiver
  |--- 10 segments ------------------------>|   round 1, cwnd 10
  |<----------------- ACKs -----------------|
  |--- 20 segments ------------------------>|   round 2, cwnd 20
  |<----------------- ACKs -----------------|
  |--- 40 segments ------------------------>|   round 3, cwnd 40
  |<----------------- ACKs -----------------|
  |--- 80 segments ------------------------>|   round 4, cwnd 80
```

"Slow" start is not slow. It is exponential growth. The name is a comparison with TCP before 1988, which sent the entire receive window at once.

**Fuller truth:** receivers usually use delayed ACKs (one ACK per two segments), so the real growth can be less than a clean doubling. RFC 5681 lets the sender count ACKed bytes with a cap per ACK. Real stacks differ a little.

### Congestion avoidance

Once cwnd reaches ssthresh, the sender is near the level that caused trouble. It grows slowly now.

1. For each ACK, add about `MSS x MSS / cwnd` bytes to cwnd.
2. Over one full window of ACKs, this adds up to about **1 MSS per RTT**.
3. This is a straight line, not a curve.

Example with ssthresh 320: cwnd goes 320, 321, 322, 323 over four RTTs.

### AIMD: the core rule

**AIMD** means Additive Increase, Multiplicative Decrease.

- **Additive increase:** when things are fine, add a fixed amount (1 MSS per RTT).
- **Multiplicative decrease:** when there is congestion, multiply by a fraction (Reno uses 0.5).

Why this exact mix? Because it makes competing flows fair without any talk between them. Chiu and Jain showed this in 1989.

Take two flows on a 100-unit link. They start unfair: 80 and 20.

```text
Step                     Flow A    Flow B    Gap
Link full, cut in half     40        10       30
Each adds 25 (full)        65        35       30
Link full, cut in half     32.5      17.5     15
Each adds 25 (full)        57.5      42.5     15
Link full, cut in half     28.75     21.25    7.5
```

Adding keeps the gap the same. Halving cuts the gap in half. Repeat this, and the two flows become equal.

**Fuller truth:** this fairness assumes the same RTT. A flow with a short RTT adds faster, because its "per RTT" clock ticks more often. It takes a bigger share. Going Deeper returns to this.

### Fast retransmit

The receiver sends an ACK for the next byte it expects. If a segment is lost, each later segment makes the receiver repeat the same ACK. A repeat is a **duplicate ACK**.

```text
Sender                                   Receiver
  |--- seg 1 ------------------------------>|
  |<------------------------------ ACK 2 ---|
  |--- seg 2 ---------X  lost               |
  |--- seg 3 ------------------------------>|
  |<------------------------------ ACK 2 ---|  dup 1
  |--- seg 4 ------------------------------>|
  |<------------------------------ ACK 2 ---|  dup 2
  |--- seg 5 ------------------------------>|
  |<------------------------------ ACK 2 ---|  dup 3
  |--- seg 2 (retransmit) ----------------->|  no waiting for RTO
  |<------------------------------ ACK 6 ---|
```

(Segment numbers are used here for reading ease. Real ACK numbers count bytes.)

The sender waits for three duplicates, not one, because packets can arrive out of order. One or two duplicates may mean only reordering.

Fast retransmit is much quicker than the RTO timer. RFC 6298 recommends a minimum RTO of 1 second, and Linux uses a 200 ms minimum. A lost packet can be repaired in about one RTT.

### Fast recovery

Duplicate ACKs prove that packets are still arriving. The path is alive. So there is no need to restart from nothing. This is what Reno added.

Steps for classic Reno (RFC 5681):

1. On the third duplicate ACK, set `ssthresh = max(FlightSize / 2, 2 x MSS)`. **FlightSize** is the data currently in flight.
2. Retransmit the lost segment.
3. Set `cwnd = ssthresh + 3 MSS`. The 3 stands for the three segments that have left the network, because the receiver got them.
4. For each further duplicate ACK, add 1 MSS to cwnd. Each one means another packet left the network, so one new packet can enter. This keeps data moving during the repair.
5. When a new ACK arrives (covering the lost data), set `cwnd = ssthresh`. This step is called "deflating" the window.
6. Continue in congestion avoidance.

### The full state machine

```text
             +-------------------+
  start ---->|    Slow Start     |
             | cwnd +1 per ACK   |
             +-------------------+
                |             ^
  cwnd reaches  |             |  timeout:
  ssthresh      |             |  ssthresh = cwnd / 2
                v             |  cwnd = 1
             +-------------------+
             | Congestion Avoid. |
             | cwnd +1 per RTT   |
             +-------------------+
                |             ^
  3 dup ACKs:   |             |  new ACK:
  ssthresh =    |             |  cwnd = ssthresh
  cwnd / 2      v             |
             +-------------------+
             |  Fast Recovery    |
             | resend lost data  |
             +-------------------+

  A timeout in any state leads back to Slow Start.
```

### Tahoe vs Reno

These two are named after BSD Unix releases (4.3BSD Tahoe in 1988, 4.3BSD Reno in 1990).

| Event | Tahoe | Reno |
|---|---|---|
| 3 duplicate ACKs | ssthresh = half, cwnd = 1, slow start | ssthresh = half, cwnd = about half, fast recovery |
| Timeout | ssthresh = half, cwnd = 1, slow start | Same as Tahoe |
| Fast retransmit | Yes | Yes |
| Fast recovery | No | Yes |

Here is the same loss for both, with IW 1, ssthresh 8, and a loss seen at cwnd 12.

```text
Round:  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
Tahoe:  1  2  4  8  9 10 11 12  1  2  4  6  7  8  9
Reno :  1  2  4  8  9 10 11 12  6  7  8  9 10 11 12
```

```text
cwnd
 12 |                      *                    R
 11 |                   *                    R
 10 |                *                    R
  9 |             *                    R        T
  8 |          *                    R        T
  7 |                            R        T
  6 |                         R        T
  5 |
  4 |       *                       T
  3 |
  2 |    *                       T
  1 | *                       T
    +---------------------------------------------> round
      1  2  3  4  5  6  7  8  9 10 11 12 13 14 15

    * = both   T = Tahoe only   R = Reno only
```

Rounds 1-4 are slow start. Rounds 5-8 are congestion avoidance. Tahoe then falls to 1. Reno keeps going from half. This repeated climb and halving is the famous **sawtooth** pattern.

Note: The Reno row skips the short fast recovery phase to keep the numbers simple.

**NewReno** (RFC 6582) fixes a weakness in Reno. If several packets are lost in one window, classic Reno can leave fast recovery too early and halve repeatedly, or stall into a timeout. NewReno stays in recovery until everything that was sent before the loss is acknowledged. A "partial ACK" tells it the next hole to repair.

**SACK (Selective Acknowledgement, RFC 2018)** is better still. The receiver lists exactly which blocks arrived, so the sender can repair several holes in one RTT.

### CUBIC

Reno has a problem on fast, long paths. It adds only 1 segment per RTT.

Take a 10 Gbps path with a 100 ms RTT. It needs a window of about 83,333 segments (1500-byte packets). After one loss, cwnd halves. Regaining 41,667 segments at 1 per RTT takes 41,667 RTTs. That is about 69 minutes. To stay at full speed, the path could lose only about one packet in five billion (RFC 3649 gives these figures).

CUBIC fixes this by making growth depend on **time since the last loss**, not on the number of RTTs. It is the default in Linux since kernel 2.6.19 (2006), and is defined in RFC 9438. Modern Windows and macOS also use CUBIC by default. Older Windows versions used Compound TCP or NewReno, depending on version.

The window follows a cubic curve:

```text
W(t) = C x (t - K)^3 + Wmax

K = cube root of ( Wmax x (1 - beta) / C )

W(t) = target window, in segments
Wmax = window just before the last loss
t    = seconds since the last loss
C    = 0.4 (scaling constant)
beta = 0.7 (cwnd is cut to 70 percent on loss)
K    = seconds until W gets back to Wmax
```

Example with Wmax = 1000 segments:

- On loss, cwnd falls to 700 (0.7 x 1000).
- K = cube root of (1000 x 0.3 / 0.4) = cube root of 750 = 9.09 seconds.

| Seconds after loss | W(t) in segments |
|---|---|
| 0 | 700 |
| 3 | 910 |
| 6 | 988 |
| 9 | 1000 |
| 12 | 1010 |
| 15 | 1083 |
| 18 | 1283 |

```text
cwnd
 1300 |                                                        *
 1250 |
 1200 |                                                     *
 1150 |                                                  *
 1100 |                                               *
 1050 |                                            *
 1000 |....................*..*..*..*..*..*..*..*............... Wmax
  950 |              *  *
  900 |           *
  850 |        *
  800 |     *
  750 |
  700 |  *
      +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+-> sec
         0        3        6        9       12       15       18
```

The shape has a clear meaning:

1. **Fast at first:** far below the old maximum, it is safe to rush back.
2. **Flat near Wmax:** this level caused loss last time, so it stays there a while and collects data at a high rate.
3. **Fast again after Wmax:** if there is no loss above Wmax, the network probably has new room, so it probes faster and faster.

CUBIC has a safety rule, the **Reno-friendly region**. It estimates what standard TCP would have at this moment, and uses that value if it is larger than the cubic curve. This keeps CUBIC from being worse than Reno on short, low-speed paths.

**Why time-based growth helps:** the window curve does not depend on RTT. A 10 ms flow and a 100 ms flow grow their windows at the same speed. This reduces (but does not remove) RTT unfairness.

CUBIC is still **loss-based**. It keeps pushing until a queue overflows. This is the root of bufferbloat.

### BBR

BBR means Bottleneck Bandwidth and Round-trip propagation time. Google published it in 2016. Its idea is different: do not wait for loss. Measure the path instead.

The **bottleneck** is the slowest link on the path. A path has two key numbers:

- **BtlBw:** the bottleneck bandwidth.
- **RTprop:** the RTT with empty queues.

The best place to run is exactly at the BDP (BtlBw x RTprop). Below it, the link is not full. Above it, a queue builds and adds delay. Far above it, the queue overflows and packets drop.

```text
Delivery rate
     ^
     |            +--------------------------  BtlBw
     |          /
     |        /
     |      /
     +----+-----------+-----------+-----> data in flight
         BDP      BDP + buffer

RTT
     ^                            |  loss starts here
     |                          / |
     |                        /   |
     |                      /     |
     |--------------------/       |
     |  RTprop                    |
     +----+-----------+-----------+-----> data in flight
         BDP      BDP + buffer

     BBR aims to run here: at BDP.
     Loss-based TCP runs near: BDP + buffer.
```

**Analogy:** a loss-based sender is a driver who speeds up until they hit the car in front, then brakes. BBR is a driver who watches the road and holds the speed that the road can carry. The analogy stops being true because BBR cannot see the road directly. It still has to test by briefly going faster.

How BBR works:

1. **Startup:** double the sending rate each RTT, as in slow start. Leave this state when measured bandwidth stops growing by 25 percent over three rounds.
2. **Drain:** startup overfilled the queue, so send slower for a short time to empty it.
3. **ProbeBW:** the main state. Cycle the pacing rate through 1.25, 0.75, then 1.0 for six phases. The 1.25 tests for more bandwidth. The 0.75 drains any queue the test created.
4. **ProbeRTT:** if no lower RTT has been seen for 10 seconds, drop cwnd to 4 packets for at least 200 ms, to measure the empty-queue RTT again.

**Pacing** means spreading packets evenly over time instead of sending bursts. BBR depends on it. In Linux, BBR originally needed the `fq` queue discipline for pacing. Since kernel 4.13, TCP can do its own pacing, though `fq` is still commonly recommended.

**Versions and status:**

- BBRv1 has been in mainline Linux since kernel 4.9.
- BBRv2 and BBRv3 added loss and ECN awareness to improve fairness. Google has developed these in its own kernel tree. Check what your kernel really ships, as this has changed over time.
- BBR is not a published standards-track RFC. The IETF work on it has been in Internet-Draft form. Treat it as widely deployed, not as a finished standard.

**Known weakness:** BBRv1 largely ignores loss. On shallow buffers it can take too much bandwidth from CUBIC flows and cause many retransmissions. On very deep buffers, CUBIC can win instead. Many BBR flows together can overestimate the bandwidth and keep a standing queue.

### ECN (Explicit Congestion Notification)

Dropping a packet to signal congestion is wasteful. The packet must be sent again. ECN (RFC 3168) lets a router mark a packet instead of dropping it.

**Analogy:** instead of removing a truck from the road, a police officer puts a sticker on it saying "slow down". The analogy stops being true because the sticker must first reach the customer, who then tells the sender.

ECN uses 2 bits in the IP header, the last 2 bits of the old ToS byte. This byte is called Traffic Class in IPv6.

```text
 0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
|      DSCP (6 bits)    |  ECN  |
+---+---+---+---+---+---+---+---+
```

| ECN bits | Name | Meaning |
|---|---|---|
| 00 | Not-ECT | Sender does not support ECN |
| 01 | ECT(1) | ECN-capable transport |
| 10 | ECT(0) | ECN-capable transport |
| 11 | CE | Congestion Experienced, set by a router |

TCP uses 2 flags in the header:

- **ECE (ECN-Echo):** set by the receiver, meaning "a packet reached me marked CE".
- **CWR (Congestion Window Reduced):** set by the sender, meaning "I got the message and slowed down".

```text
Sender          Router (queue filling)         Receiver
  |                    |                          |
  |-- SYN, ECE+CWR ---------------------------->  |  "I can do ECN"
  |<------------------------- SYN-ACK, ECE ------ |  "So can I"
  |                    |                          |
  |-- data, ECT(0) --> | marks CE                 |
  |                    |-- data, CE ------------> |
  |<---------------------------- ACK, ECE ------- |  repeats until CWR
  | cuts cwnd, as for one loss                    |
  |-- data, CWR ------------------------------->  |  stops the echo
```

Three facts people miss:

1. Both endpoints and the router must take part. A router marks only if it has an active queue management policy with ECN turned on.
2. ECN does not remove loss. If a queue overflows, packets still drop.
3. Classic ECN gives only one signal per RTT, and the sender reacts as it would to a loss.

### Bufferbloat

**Bufferbloat** means too-large buffers in network devices cause very high delay. Jim Gettys made the term popular around 2010.

Router and modem buffers absorb short bursts. That is useful. But loss-based TCP only slows down when a buffer is full. So it **fills** every buffer on its path, on purpose, again and again.

**Analogy:** a bigger waiting room does not make the doctor work faster. It only makes patients wait longer. The analogy stops being true because a short queue is healthy on a network. It keeps the link busy during bursts.

Delay added by a full buffer is:

```text
queue delay = buffer size / link speed
```

A 2 MB buffer on a 20 Mbps uplink adds 16 Mbit / 20 Mbps = **800 ms** of delay. One large upload makes video calls and games unusable for everyone on that link.

Note the bitter part: the link speed did not change, and almost no packets are lost. Tools can report "no problem" while every user suffers.

**The fixes:**

- **AQM (Active Queue Management):** the router drops or marks early, before the buffer is full. Examples are RED (old, hard to tune), CoDel (RFC 8289), and PIE (RFC 8033).
- **Fair queuing:** give each flow its own small queue, so one upload does not delay a video call. FQ-CoDel (RFC 8290) and CAKE combine this with AQM.
- **Shaping:** limit your traffic just below the true link speed, so the queue forms in your own smart device, not in the modem.
- **Delay-aware senders:** BBR tries not to build a queue.

CoDel's idea is smart: it measures how long a packet waited in the queue, not how many bytes are queued. By default, if the waiting time stays above 5 ms for 100 ms, it starts dropping or marking.

### The throughput formula

The **Mathis equation** (Mathis et al., 1997) gives the rough long-run limit of Reno-style TCP on a path with random loss.

```text
                 MSS        C
Throughput  <=  ----- x  --------
                 RTT     sqrt(p)

MSS = segment size
RTT = round trip time
p   = packet loss rate (0.0001 means 0.01 percent)
C   = about 1.22 (sqrt(3/2))
```

Three lessons:

- Double the RTT, and throughput halves.
- Four times the loss, and throughput halves.
- Link speed is not in the formula at all.

The true limit of a connection is the smallest of these:

```text
Throughput = min( link capacity,
                  rwnd / RTT,
                  send buffer / RTT,
                  cwnd / RTT )
```

**Fuller truth:** the formula is a model of Reno. It does not apply to BBR, which does not treat random loss as congestion. For CUBIC, the published analysis gives a different form, where throughput falls with roughly `1 / (RTT^0.25 x p^0.75)`. So CUBIC suffers less from high RTT but still suffers from loss. The value of C also depends on ACK behaviour.

### Why high latency hurts

High RTT hurts TCP in six separate ways.

1. **Every feedback loop is slower.** cwnd grows per RTT. A sender on a 200 ms path gets 10 times fewer growth steps per second than one on a 20 ms path.
2. **Slow start takes longer.** Reaching a given window needs the same number of rounds, but each round is longer.
3. **Short transfers never reach full speed.** Most web objects finish during slow start, so their time is set by RTT, not bandwidth.
4. **You need a bigger window.** Rate = window / RTT. Without window scaling, the 65,535-byte limit gives only 5.2 Mbps at a 100 ms RTT, no matter how fast the link is.
5. **Recovery is slow.** A lost packet needs at least one RTT to repair. Reno regains its window at 1 MSS per RTT.
6. **Unfair sharing.** On a shared link, the low-RTT flow grows faster and gets more of the bandwidth.

## Example

### Worked example: one connection from start to first loss

**Setup:**

```text
Client                  Router                           Server
10.0.0.5  -- 1 Gbps --  buffer 200  -- 100 Mbps path --  203.0.113.10
                        packets        (bottleneck)

RTT with empty queues:  50 ms
MSS:                    1460 bytes
Initial window:         10 segments
rwnd:                   large, not the limit
```

**Step 1: find the BDP.**

```text
BDP = 100,000,000 bits/s x 0.05 s = 5,000,000 bits = 625,000 bytes
BDP in segments = 625,000 / 1460 = about 428 segments
```

So 428 segments fill the link exactly. There is no queue yet.

**Step 2: find the point where loss starts.**

```text
Pipe capacity = BDP + buffer = 428 + 200 = 628 segments
```

A cwnd above 628 overflows the router buffer.

**Step 3: run slow start.**

| Round | Time (ms) | cwnd | Sent this round (KB) | Total sent (KB) |
|---|---|---|---|---|
| 1 | 0 | 10 | 14.6 | 14.6 |
| 2 | 50 | 20 | 29.2 | 43.8 |
| 3 | 100 | 40 | 58.4 | 102.2 |
| 4 | 150 | 80 | 116.8 | 219.0 |
| 5 | 200 | 160 | 233.6 | 452.6 |
| 6 | 250 | 320 | 467.2 | 919.8 |
| 7 | 300 | 640 | 934.4 | 1854.2 |

In round 7, cwnd is 640, which is above 628. The buffer overflows and packets drop.

(To keep this simple, the time column ignores the extra queue delay in the last rounds.)

**Step 4: compare the algorithms on this loss.**

| | Tahoe | Reno | CUBIC |
|---|---|---|---|
| New ssthresh | 320 | 320 | 448 (0.7 x 640) |
| cwnd after loss | 1 | 320 | 448 |
| Is cwnd above BDP (428)? | No | No | Yes |
| Rate right after loss | Very low for about 0.45 s (9 RTTs of slow start) | About 75 Mbps | 100 Mbps |
| Time to the next loss | About 18.5 s | About 18 s | About 5 s |

The Reno rate is `320 x 1460 x 8 / 0.05 = 74.8 Mbps`.

The CUBIC case is interesting. Since 448 is still above the BDP of 428, the link never goes idle. CUBIC keeps the link full through the loss.

**Step 5: Reno's time to recover.**

1. From 320 to 428 is 108 RTTs at 50 ms each: 5.4 seconds. The link is not full in this phase.
2. From 428 to 628 is 200 RTTs. The queue now grows, so the RTT grows too. Each queued packet adds 0.117 ms (1460 x 8 bits at 100 Mbps). A full queue of 200 adds 23 ms, so RTT climbs from 50 ms to 73 ms. This phase takes about 12.3 seconds.
3. Total time to the next loss: about 18 seconds.

Average rate in the cycle: about 87 Mbps for 5.4 s, then 100 Mbps for 12.3 s. That gives about **96 Mbps** on average. The buffer makes this high use possible.

**Step 6: CUBIC's time to the old maximum.**

```text
K = cube root of (640 x 0.3 / 0.4) = cube root of 480 = 7.8 seconds
```

This does not depend on RTT. Loss comes a bit sooner, at 628, about 5 seconds after the last loss.

**Step 7: what the user feels.** Loss-based senders sit near a full queue, so the path RTT is up to 73 ms, not 50 ms. BBR would aim to hold about 428 segments in flight and keep RTT near 50 ms, with the same throughput.

### Worked example: "1 Gbps link, but my copy runs at 5 Mbps"

A team copies files between Frankfurt and Sydney on a 1 Gbps path. RTT is 280 ms. Loss is 0.01 percent (p = 0.0001). MSS is 1460.

```text
MSS / RTT = (1460 x 8) bits / 0.28 s = 41,714 bits/s
C / sqrt(p) = 1.22 / 0.01 = 122
Throughput = 41,714 x 122 = about 5.09 Mbps
```

That is 0.5 percent of the link. Here is the same loss rate at other RTTs (Reno model):

| RTT | Limit from Mathis |
|---|---|
| 1 ms | 1.42 Gbps (link limit of 1 Gbps applies) |
| 10 ms | 142 Mbps |
| 50 ms | 28.5 Mbps |
| 100 ms | 14.2 Mbps |
| 280 ms | 5.09 Mbps |
| 600 ms (GEO satellite) | 2.37 Mbps |

Using the CUBIC formula, the 280 ms path does better, about 17 Mbps. It is still far below 1 Gbps.

Then check the window needed:

```text
Window for 1 Gbps at 280 ms = 1,000,000,000 x 0.28 / 8 = 35 MB
```

So socket buffers must also allow about 35 MB, or they become the limit first.

**What can fix it?**

| Fix | Why it helps |
|---|---|
| Find and fix the loss source | Lower p has a square-root effect |
| Use BBR on the sender | Random loss at this level does not cut its rate |
| Larger socket buffers | Removes the rwnd and send buffer limits |
| Parallel streams (for example 8) | Each stream has its own cwnd |
| Move data closer (CDN, relay) | Lower RTT, see [Content Delivery Networks](../10-enterprise-and-datacenter/06-cdn.md) |

### Worked example: a 1 MB page object on a fast link

Bandwidth is not the limit here. The link is 1 Gbps. How long does 1 MB take?

1 MB is about 685 segments. Rounds send 10, 20, 40, 80, 160, 320, which totals 630. So a seventh round is needed.

| RTT | 7 rounds of data | Total with TCP handshake (8 RTT) |
|---|---|---|
| 20 ms | 140 ms | 160 ms |
| 200 ms | 1400 ms | 1600 ms |

At 1 Gbps the raw send time is just 8 ms. The time is almost all RTT. A 10 times better RTT gives a 10 times better result. A 10 times better link gives almost nothing.

### Check the algorithm on real hosts

**Linux:**

```bash
# Which algorithm is active, and which are loaded
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
```

```text
net.ipv4.tcp_congestion_control = cubic
net.ipv4.tcp_available_congestion_control = reno cubic
```

**Windows (PowerShell):**

```powershell
Get-NetTCPSetting -SettingName Internet |
    Select-Object SettingName, CongestionProvider,
        InitialCongestionWindowMss
```

```text
SettingName CongestionProvider InitialCongestionWindowMss
----------- ------------------ --------------------------
Internet    CUBIC                                      10
```

Values vary by Windows version. Windows has separate templates, such as `Internet` and `Datacenter`.

**See live cwnd for a connection (Linux).** This is the 100 Mbps path from the worked example, with timestamps on, so the MSS is 1448:

```bash
ss -tin dst 203.0.113.10
```

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port
ESTAB  0      812544 10.0.0.5:48712      203.0.113.10:5201
  cubic wscale:7,7 rto:264 rtt:62.5/2.1 mss:1448 pmtu:1500
  cwnd:540 ssthresh:448 send 100Mbps pacing_rate 120Mbps
  delivery_rate 95.2Mbps retrans:0/214 minrtt:50.1
```

(Output is cut and wrapped for width. Real `ss` prints more fields on one line.)

How to read this:

- `cwnd:540` is above `ssthresh:448`, so it is in congestion avoidance.
- `send 100Mbps` is `cwnd x mss x 8 / rtt` = 540 x 1448 x 8 / 0.0625.
- `rtt:62.5` is above `minrtt:50.1`. The extra 12 ms is queue delay.
- `retrans:0/214` means 0 unacknowledged retransmits now, and 214 in total.

**Watch cwnd during a test with iperf3 (Linux or Windows).** More on this tool in [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

```bash
iperf3 -c 203.0.113.10 -t 6
```

```text
Connecting to host 203.0.113.10, port 5201
[  5] local 10.0.0.5 port 48712 connected to 203.0.113.10 port 5201
[ ID] Interval         Transfer     Bitrate        Retr  Cwnd
[  5] 0.00-1.00 sec  8.12 MBytes  68.1 Mbits/sec  214   698 KBytes
[  5] 1.00-2.00 sec  11.4 MBytes  95.6 Mbits/sec    0   772 KBytes
[  5] 2.00-3.00 sec  11.4 MBytes  95.6 Mbits/sec    0   829 KBytes
[  5] 3.00-4.00 sec  11.4 MBytes  95.6 Mbits/sec    0   865 KBytes
[  5] 4.00-5.00 sec  11.4 MBytes  95.6 Mbits/sec    0   888 KBytes
[  5] 5.00-6.00 sec  11.2 MBytes  94.0 Mbits/sec   37   661 KBytes
```

Retransmits in a row with a falling Cwnd show a congestion event. The climb that slows down near 888 KBytes is the flat part of the CUBIC curve.

**Enable BBR (Linux).** Test first, because it can affect other flows.

```bash
sudo modprobe tcp_bbr
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# Keep after reboot
sudo tee /etc/sysctl.d/90-bbr.conf <<EOF
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
```

This changes new connections only. The algorithm runs on the **sender**. To speed up downloads from a server, change the server, not the client.

**Set the algorithm for one route (Linux):**

```bash
sudo ip route change 198.51.100.0/24 via 10.0.0.1 dev eth0 congctl bbr
```

**ECN checks (Linux):**

```bash
sysctl net.ipv4.tcp_ecn
# 0 = off
# 1 = ask for ECN on outgoing connections, accept on incoming
# 2 = accept on incoming only (the kernel default)

sudo sysctl -w net.ipv4.tcp_ecn=1

# Capture packets with ECE or CWR set (byte 13 of the TCP header)
sudo tcpdump -ni eth0 'tcp[13] & 0xc0 != 0'

# Capture IPv4 packets marked CE
sudo tcpdump -ni eth0 'ip[1] & 0x03 = 3'
```

**ECN on a Cisco IOS WAN edge.** Routers do not run congestion control for traffic that passes through them, but they do manage queues. This policy marks instead of dropping when the queue grows. Feature support varies by platform.

```text
class-map match-all BULK
 match dscp af11
!
policy-map WAN-EDGE
 class BULK
  bandwidth percent 40
  random-detect dscp-based
  random-detect ecn
!
interface GigabitEthernet0/1
 service-policy output WAN-EDGE
```

More on queues is in [Quality of Service (QoS)](../10-enterprise-and-datacenter/03-quality-of-service.md).

### Worked example: detect and fix bufferbloat

A home user at 192.168.1.20 has a 20 Mbps uplink. Video calls freeze during cloud backups.

**Step 1: run ping and load the link together.**

```bash
# Terminal 1
ping 203.0.113.10

# Terminal 2, a 30 second upload
iperf3 -c 203.0.113.10 -t 30
```

```text
64 bytes from 203.0.113.10: icmp_seq=1 ttl=54 time=12.4 ms
64 bytes from 203.0.113.10: icmp_seq=2 ttl=54 time=12.1 ms
                 (iperf3 starts here)
64 bytes from 203.0.113.10: icmp_seq=3 ttl=54 time=96.7 ms
64 bytes from 203.0.113.10: icmp_seq=4 ttl=54 time=358 ms
64 bytes from 203.0.113.10: icmp_seq=5 ttl=54 time=641 ms
64 bytes from 203.0.113.10: icmp_seq=6 ttl=54 time=806 ms
```

RTT rose from 12 ms to over 800 ms with no ping loss. That is about 800 ms of buffer on a 20 Mbps link, so about 2 MB of queue. This is bufferbloat.

**Step 2: fix it on a Linux router, on the WAN port eth0.** Shape to 90 percent of the uplink, so the queue forms here, under smart control.

```bash
sudo tc qdisc replace dev eth0 root cake bandwidth 18mbit
```

**Step 3: measure again.**

```text
64 bytes from 203.0.113.10: icmp_seq=1 ttl=54 time=12.6 ms
64 bytes from 203.0.113.10: icmp_seq=2 ttl=54 time=16.2 ms
64 bytes from 203.0.113.10: icmp_seq=3 ttl=54 time=19.1 ms
```

The trade is 2 Mbps of upload speed for about 790 ms less delay. That is a very good trade for almost every user.

## Going Deeper

### Why the Mathis constant is 1.22

Assume a Reno sawtooth with one loss per cycle.

1. cwnd moves between W/2 and W. The average is 3W/4.
2. The cycle lasts W/2 RTTs, so packets per cycle = 3W/4 x W/2 = 3W^2/8.
3. One loss per cycle means p = 8 / (3W^2), so W = sqrt(8 / (3p)).
4. Average window = 3/4 x sqrt(8 / (3p)) = sqrt(3 / (2p)) = 1.22 / sqrt(p).
5. Throughput = average window x MSS / RTT.

### The CUBIC formula for throughput

With C = 0.4 and beta = 0.7, the average CUBIC window in segments is about:

```text
W_avg = 1.054 x RTT^0.75 / p^0.75     (RTT in seconds)
```

At RTT 0.28 s and p = 0.0001, W_avg is about 406 segments, against 122 for Reno. At short RTTs this formula gives a smaller number than Reno, and the Reno-friendly region takes over. In other words, CUBIC mostly helps on long, fast paths.

### More CUBIC detail

- **Fast convergence:** if a loss happens below the previous Wmax, the flow is probably losing share to a new flow. CUBIC then sets Wmax lower (`cwnd x (1 + beta) / 2`), to release bandwidth sooner.
- **The Reno-friendly slope:** the estimate uses an additive increase of about 0.53 MSS per RTT (3 x 0.3 / 1.7). This matches Reno's average rate even with CUBIC's smaller cut.
- **HyStart:** classic slow start doubles until loss, then overshoots badly. HyStart watches for rising RTT and leaves slow start early. Linux CUBIC has its own version. RFC 9406 defines HyStart++.

### Modern loss recovery

- **SACK-based recovery:** RFC 6675 uses SACK data to estimate what is really in flight ("pipe").
- **PRR (Proportional Rate Reduction):** RFC 6937. It lowers cwnd smoothly during recovery, instead of Reno's stop-and-burst pattern.
- **RACK-TLP:** RFC 8985. It decides loss by **time** ("this was sent long before something that already arrived"), not by counting three duplicates. A Tail Loss Probe resends the last segment early, so a lost tail need not wait for the RTO. Modern Linux uses RACK.
- **Spurious timeouts:** if a delay spike fires the RTO but nothing was lost, TCP cut cwnd for no reason. F-RTO (RFC 5682) detects this.

### The idle problem

cwnd is a measure of the path **right now**. After a long idle, it may be out of date. If a stack sends a full old window at once, it can cause a burst of loss.

Linux resets cwnd after an idle period by default (`net.ipv4.tcp_slow_start_after_idle = 1`). This surprises people who reuse HTTP keep-alive connections with bursty traffic: their "warm" connection is cold again. Servers that do this may set the value to 0. The risk then is a burst on a path that has changed.

```bash
sudo sysctl -w net.ipv4.tcp_slow_start_after_idle=0
```

### Algorithm overview

| Algorithm | Main signal | Growth | Loss response | Notes |
|---|---|---|---|---|
| Tahoe | Loss | SS + AIMD | cwnd = 1 | History |
| Reno | Loss | SS + AIMD | cwnd x 0.5 | Reference model |
| NewReno | Loss | SS + AIMD | cwnd x 0.5 | Handles multi-loss |
| Vegas | Delay | Adjusts to hold small queue | cwnd x 0.75 | Loses to loss-based flows |
| Compound TCP | Loss + delay | Mixed | About 0.5 | Older Windows |
| BIC | Loss | Binary search | cwnd x 0.8 | Linux default before CUBIC |
| CUBIC | Loss | Cubic in time | cwnd x 0.7 | Common default today |
| DCTCP | ECN marks | AIMD | Proportional to marks | Data centres |
| BBR | Rate + RTT model | Model-based | Not loss-driven (v1) | Google, wide use |

(SS = slow start.)

### Data centre congestion

A data centre has RTTs of tens of microseconds and shallow switch buffers.

- **Incast:** one client asks 100 servers for data at once. All replies arrive together and overflow one switch port queue. Some flows lose packets and then wait for a minimum RTO that is thousands of times longer than the RTT.
- **DCTCP** (RFC 8257): switches mark ECN at a shallow threshold. The sender measures the **fraction** of marked packets, called alpha, and sets `cwnd = cwnd x (1 - alpha / 2)`. A small problem gives a small cut. This keeps queues very short. DCTCP is meant for controlled networks, not the open internet, as it does not mix fairly with classic TCP.

### L4S

**L4S** (Low Latency, Low Loss, Scalable throughput, RFC 9330) brings the DCTCP idea to the internet. Traffic uses ECT(1) to say "I am a scalable flow." Routers use a dual queue (RFC 9332) to keep this traffic apart from classic traffic. The goal is very low queue delay. Deployment is at an early stage in 2026.

### How big should a buffer be?

The classic rule is buffer = BDP. It comes from Reno's sawtooth. After cwnd halves, a buffer of one BDP keeps the link full. The worked example shows the case with a buffer of about half the BDP, where the link was underused for 5.4 s of each cycle.

With many flows on one link, their sawtooths do not line up. Research by Appenzeller and others in 2004 showed that buffer = BDP / sqrt(N) can be enough for N long flows. This is why backbone routers do not need huge buffers. Too-small buffers still hurt bursty traffic.

### Random loss and wireless

Classic TCP reads every loss as congestion. A Wi-Fi or radio bit error then cuts cwnd for no good reason. Wi-Fi and mobile networks hide most of this with retransmissions at Layer 2, but that adds delay jitter (changing delay). Delay jitter can confuse delay-based and rate-based algorithms too. See [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).

GEO satellite links have about 600 ms RTT, so they often use a **Performance Enhancing Proxy** (PEP, RFC 3135). The PEP splits one TCP connection into shorter ones. End-to-end encryption at the transport level, as in QUIC, stops this trick.

### Congestion control and QUIC

QUIC uses the same ideas: its RFC 9002 describes a NewReno-style controller, and real stacks commonly use CUBIC or BBR. The big difference is that QUIC runs in user space. A company can change its algorithm with an app update, with no kernel change on any machine. See [QUIC](08-quic.md).

### Bandwidth share is per flow, not per user

TCP fairness is per connection. An app with 8 connections gets about 8 shares. This is one reason download tools open many streams, and one reason HTTP/2 moved to a single connection per host. Fairness between users, if required, must come from the network, by [QoS](../10-enterprise-and-datacenter/03-quality-of-service.md) or fair queuing.

## Common Mistakes

- **Mixing up flow control and congestion control.** rwnd protects the receiver. cwnd protects the network. The sender uses the smaller one.
- **Thinking cwnd is in the TCP header.** It is not. Only rwnd is sent. cwnd lives in the sender's memory, and a packet capture shows it only as an estimate of bytes in flight.
- **Thinking "slow start" is slow.** It is exponential. It doubles each RTT.
- **Thinking every loss resets cwnd to 1.** Only a timeout does this in Reno. Three duplicate ACKs cut it to about half.
- **Thinking the receiver or a router chooses the algorithm.** Each sender picks its own. Two ends of one connection can use different algorithms.
- **Changing the client to speed up downloads.** The server is the sender of the download, so the server's algorithm controls it.
- **Assuming more bandwidth fixes slow transfers.** On a long path, RTT and loss usually set the limit.
- **Assuming a bigger buffer is always better.** It reduces loss but adds delay. That is bufferbloat.
- **Saying BBR is always better.** It can be unfair to CUBIC and can retransmit more. Test it on your own traffic.
- **Using ping on an idle link to check latency.** Bufferbloat shows only under load. Measure RTT while the link is busy.
- **Thinking ECN removes the need for drops.** Buffers can still overflow. And ECN needs support at both ends and in the routers.
- **Thinking UDP traffic joins in.** UDP has no congestion control. The application must add its own, as QUIC does.

## Interview Questions

**Q: What is the difference between flow control and congestion control?**

A: Flow control protects the receiver. The receiver advertises rwnd in each TCP header. Congestion control protects the network. The sender keeps a private cwnd, guessed from ACKs, loss and delay. The sender may have at most the smaller of the two in flight.

**Q: Walk me through slow start and congestion avoidance.**

A: The sender starts with an initial window, usually 10 segments. In slow start, each ACK adds up to one MSS, so cwnd doubles each RTT. When cwnd reaches ssthresh, it moves to congestion avoidance and adds about one MSS per RTT. Loss cuts cwnd and sets a new ssthresh.

**Q: What is the difference between three duplicate ACKs and a timeout?**

A: Three duplicate ACKs mean a packet was lost but later packets arrived, so the path still works. Reno halves cwnd and uses fast recovery. A timeout means nothing useful came back, so it could be serious. TCP sets cwnd to one and returns to slow start. A timeout also waits much longer.

**Q: Why AIMD and not, say, multiplicative increase and decrease?**

A: Additive increase with multiplicative decrease is what moves flows to a fair share. Adding the same amount keeps the gap between two flows the same. Cutting by half halves the gap. Over many cycles the gap goes to zero. With multiplicative increase and decrease the ratio between flows never changes.

**Q: What is the difference between Tahoe and Reno?**

A: Both have slow start, congestion avoidance and fast retransmit. On three duplicate ACKs, Tahoe sets cwnd to one and does slow start. Reno adds fast recovery: it cuts cwnd to about half and continues. On a timeout, both go to one.

**Q: Why did Linux move to CUBIC?**

A: Reno adds one segment per RTT, so on a fast long path it takes far too long to recover. A 10 Gbps path with 100 ms RTT needs over an hour. CUBIC grows by a cubic function of time since the last loss. It cuts to 70 percent, returns quickly to the old maximum, stays flat there, then probes faster. Growth does not depend on RTT.

**Q: How is BBR different from CUBIC?**

A: CUBIC is loss-based: it grows until a queue overflows. BBR builds a model. It measures the bottleneck bandwidth and the minimum RTT, then paces traffic at about the BDP. It avoids filling buffers, so it gives low delay, and it holds up on paths with random loss. BBRv1 can be unfair to CUBIC, and later versions work on this.

**Q: What is bufferbloat, and how do you detect and fix it?**

A: It is high delay caused by oversized buffers that loss-based TCP fills. To detect it, I run ping while the link is under a full load. If the RTT jumps from 12 ms to 800 ms, that is bufferbloat. To fix it, I use AQM and fair queuing, such as FQ-CoDel or CAKE, and shape slightly below the link speed so the queue forms where I control it.

**Q: What does ECN do?**

A: It lets a router signal congestion without dropping. Endpoints negotiate ECN in the handshake and send packets marked ECT. A router with AQM sets CE instead of dropping. The receiver echoes this with ECE in the ACKs. The sender cuts cwnd and sets CWR. It avoids a retransmission, but it needs support along the path.

**Q: How would you estimate TCP throughput on a lossy path?**

A: I use the Mathis equation: throughput is at most MSS / RTT x 1.22 / sqrt(p). With 1460 bytes, 100 ms and 0.01 percent loss, that gives about 14 Mbps. It models Reno, so it is a rough guide. I also check the window limit, which is window / RTT.

**Q: Why does high latency hurt TCP even when bandwidth is huge?**

A: All feedback runs per RTT. Slow start and window growth take longer, and repair takes at least one RTT. Throughput in loss-based TCP falls with RTT. The window must also grow to the BDP, so buffers can become the limit. Short transfers finish during slow start, so their time depends on RTT, not bandwidth.

**Q: A team says their new 1 Gbps intercontinental link gives only 5 Mbps per transfer. How do you debug?**

A: First I measure RTT and loss, and read `ss -ti` on the sender. I check whether it is limited by cwnd, rwnd or send buffer. I compare the Mathis number with what they see. Then I look for the loss source, raise the socket buffers to the BDP, try BBR on the sender, and test parallel streams with iperf3.

## Quick Recap

- Flow control protects the receiver with rwnd. Congestion control protects the network with cwnd. The sender obeys the smaller.
- Slow start doubles cwnd each RTT. Congestion avoidance adds 1 MSS per RTT. AIMD makes flows converge to a fair share.
- Three duplicate ACKs trigger fast retransmit. Reno adds fast recovery. A timeout sends cwnd back to 1.
- CUBIC is the common default: it cuts to 70 percent and grows by a cubic curve of time. BBR models bandwidth and RTT, and paces at the BDP.
- ECN marks packets instead of dropping them. AQM such as CoDel keeps queues short. Bufferbloat is delay from buffers that loss-based TCP keeps full.
- Reno-style throughput is about (MSS / RTT) x (1.22 / sqrt(p)). RTT and loss set the limit, not link speed.
- High latency slows every feedback loop: growth, recovery and short transfers all pay per RTT.
