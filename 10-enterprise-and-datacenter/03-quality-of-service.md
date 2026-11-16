# Quality of Service (QoS)

**Level:** Advanced | **Prerequisites:** [Bandwidth, Latency and Performance](../01-foundations/06-network-performance-basics.md), [Trunking and 802.1Q](../03-ethernet-and-switching/07-trunking-8021q.md), [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md), [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md)

## In One Line

QoS is a set of tools that decide which packets go first, which packets wait, and which packets are dropped when a link is full. It does not create bandwidth. It shares the bandwidth you already have, on purpose.

## Why It Matters

A network link has a fixed speed. When more traffic arrives than the link can send, the extra packets wait in a queue. A queue is a waiting line inside the router's memory (the memory is called a buffer). When the buffer is full, new packets are dropped. This state is called congestion.

Without QoS, every packet waits in the same line. Think of a branch office with a 20 Mbps WAN link:

- A backup job starts and sends 30 Mbps.
- 40 phone calls need about 3.5 Mbps.
- The queue fills up with backup packets.
- Voice packets wait behind them, or get dropped.
- People on calls hear gaps and robot voices.

The backup does not care if it finishes 10 seconds later. The phone call cares about every 20 milliseconds. QoS uses this difference.

Different traffic has different needs:

| Traffic | Bandwidth need | Hates delay? | Hates loss? |
|---|---|---|---|
| Voice call | Small, steady | Yes, very much | Yes |
| Video meeting | Medium to large, bursty | Yes | Yes |
| Web browsing | Bursty | A little | No, TCP resends |
| File backup | Large | No | No, TCP resends |

**Why interviewers ask:** QoS shows if you understand what happens inside a device under load. It also joins many topics: IP headers, TCP behaviour, WAN design and voice. Common questions are "policing vs shaping", "what is DSCP 46", "what is LLQ", and "why does QoS not work on my link that is not full".

## Simple Explanation

Think of an airport.

1. **Classification:** staff look at your ticket and decide what kind of passenger you are.
2. **Marking:** they stamp your boarding pass, so later staff do not need to check your ticket again.
3. **Queuing:** there are separate lanes. The first-class lane moves first. Everyone else shares the other lanes.
4. **Policing:** a bag over the weight limit is thrown out or charged extra at the desk.
5. **Shaping:** the airport holds passengers in a waiting room and lets them walk to security at a steady pace.
6. **Early dropping (WRED):** when the hall starts to get too full, staff ask a few people to come back later, before the hall is packed.

**Where the analogy stops:** an airport can open a new lane. A link cannot go faster than its speed. Also, QoS only acts when there is a queue. If the airport is empty, the special lane gives no benefit. In the same way, QoS does nothing on a link that is not congested.

## How It Works

### The four things QoS controls

| Term | Plain meaning | Example |
|---|---|---|
| Bandwidth | How many bits per second a path can carry | 20 Mbps WAN link |
| Delay (latency) | Time for a packet to travel from sender to receiver | 80 ms one way |
| Jitter | Change in delay from one packet to the next | Packets arrive 20, 35, 22 ms apart |
| Loss | Packets that never arrive | 2 lost out of 100 is 2 percent |

Delay has four parts:

| Delay type | Cause | Can QoS change it? |
|---|---|---|
| Serialization | Time to push the bits of one packet onto the wire | No, except by making packets smaller |
| Propagation | Time for the signal to travel the distance | No. About 5 ms per 1,000 km of fibre |
| Processing | Time for the device to look up and forward the packet | Almost no |
| Queuing | Time spent waiting in a buffer | Yes. This is the main one |

Serialization delay is packet bits divided by link speed. A 1,500 byte packet is 12,000 bits:

- On a 1 Mbps link: 12,000 / 1,000,000 = 12 ms.
- On a 1 Gbps link: 12,000 / 1,000,000,000 = 0.012 ms (12 microseconds).

So a big packet on a slow link is a real problem for voice. On a fast link it is not.

### Where congestion happens

Congestion happens where traffic meets a slower or busier exit. There are two main cases.

**Speed mismatch:** a fast link feeds a slow link.

```text
   LAN side                         WAN side
 +--------+   1 Gbps   +--------+  20 Mbps   +-------+
 | Switch |----------->| Router |----------->|  ISP  |
 +--------+            +--------+            +-------+
                           ^
                           |
              Queue builds here, on the way OUT.
```

**Aggregation:** many links feed one link.

```text
 Access port 1 (1 Gbps) --+
 Access port 2 (1 Gbps) --+--> Uplink (10 Gbps)
 ...                      |
 Access port 48 (1 Gbps) -+    48 Gbps possible, 10 Gbps exit
```

Queues build on the **exit (output) interface**. This is why queuing is an outbound tool.

### Three service models

| Model | Idea | Today |
|---|---|---|
| Best effort | No special treatment. All packets are equal | Default on the public Internet |
| IntServ (Integrated Services, RFC 1633) | Each flow reserves bandwidth on every router along the path using RSVP (Resource Reservation Protocol, RFC 2205) | Rare. Keeping per-flow state does not scale well |
| DiffServ (Differentiated Services, RFC 2475) | Packets carry a label. Each router treats each label in a set way, with no reservation | The standard choice |

A flow is one stream of related packets, such as one call. DiffServ does not track flows. It only groups packets into a small number of classes. That is why it scales.

### The QoS path through a device

```text
 PACKET COMES IN (ingress)
   |
   v
 +-----------+    +---------+    +-------------+
 | Classify  |--->|  Mark   |--->|   Police    |
 | what is   |    | stamp   |    | enforce a   |
 | this?     |    | a label |    | speed limit |
 +-----------+    +---------+    +-------------+
                                        |
                                        v
                                 Routing decision
                                        |
 PACKET GOES OUT (egress)               v
 +-----------+    +---------+    +-------------+
 |  Wire     |<---| Schedule|<---| Queue, with |
 |           |    | and     |    | WRED early  |
 |           |    | shape   |    | drop        |
 +-----------+    +---------+    +-------------+
```

Fuller truth: the exact order changes by platform. Marking and policing can also run on the way out. The picture shows the normal design.

### Step 1: Classification

Classification means deciding what a packet is. A device can look at:

- Layer 2: CoS value, VLAN, MAC address.
- Layer 3: source or destination IP, DSCP value.
- Layer 4: protocol and port, such as UDP port 5060 for SIP.
- Layer 7: application, using deep inspection such as Cisco NBAR (Network Based Application Recognition).

Deep inspection is slow and encrypted traffic hides most of the content. So we classify once, near the edge, and then mark. Devices further on only read the mark.

### Step 2: Marking

Marking means writing a value into the packet header. There are two main places to write it.

**Layer 2: CoS (Class of Service).** CoS is a 3-bit field called PCP (Priority Code Point) inside the 802.1Q VLAN tag. Values are 0 to 7. It was first defined as 802.1p, and it is now part of the 802.1Q standard.

```text
 802.1Q tag (4 bytes), added to the Ethernet frame

 +----------------+---------+---------+-----------------+
 | TPID 0x8100    | PCP/CoS |   DEI   |     VLAN ID     |
 | 16 bits        | 3 bits  |  1 bit  |     12 bits     |
 +----------------+---------+---------+-----------------+
```

DEI (Drop Eligible Indicator) says "drop this frame first if needed".

Important limit: CoS lives in the VLAN tag. There is no tag on an access link without a voice VLAN, and a router removes the Layer 2 header at every hop. So CoS does not survive routing.

**Layer 3: DSCP (Differentiated Services Code Point).** IPv4 has one byte that was first called ToS (Type of Service). The first 3 bits were called IP Precedence. RFC 2474 redefined the first 6 bits as DSCP. RFC 3168 uses the last 2 bits for ECN (Explicit Congestion Notification). IPv6 has the same layout in its Traffic Class byte.

```text
 ToS byte in IPv4 / Traffic Class byte in IPv6

 Bit:   7     6     5     4     3     2     1     0
      +-----+-----+-----+-----+-----+-----+-----+-----+
      |            DSCP (6 bits)          | ECN (2)   |
      +-----+-----+-----+-----+-----+-----+-----+-----+
      |<-- IP Precedence -->|
      |      (old use)      |
```

DSCP has 6 bits, so it has 64 values (0 to 63). The value stays in the IP header across the whole path, unless someone rewrites it.

**PHB (Per-Hop Behaviour)** is the treatment a packet gets at one router. DSCP is the label. PHB is what the router does because of that label. Think of a parcel sticker: "Fragile" is the DSCP. Carrying it with care is the PHB.

The standard PHB groups:

| PHB | DSCP | Meaning | RFC |
|---|---|---|---|
| Default (DF) | 0 | Best effort | RFC 2474 |
| Class Selector (CS) | CS0 to CS7: 0, 8, 16, 24, 32, 40, 48, 56 | Matches old IP Precedence values | RFC 2474 |
| Expedited Forwarding (EF) | 46 | Low delay, low jitter, low loss | RFC 3246 |
| Assured Forwarding (AF) | 12 values, see below | Good service, with drop levels | RFC 2597 |
| Lower Effort (LE) | 1 | Goes behind best effort | RFC 8622 |

**EF** is binary 101110. It is for traffic that must not wait, which is mainly voice. The first 3 bits are 101, which is IP Precedence 5. So an old device that only reads precedence still gives it a high value.

**AF** has 4 classes and 3 drop levels. It is written AFxy, where x is the class (1 to 4) and y is the drop precedence (1 to 3). Inside one class, y = 3 is dropped first.

| | Low drop (y=1) | Medium drop (y=2) | High drop (y=3) |
|---|---|---|---|
| Class 1 | AF11 = 10 | AF12 = 12 | AF13 = 14 |
| Class 2 | AF21 = 18 | AF22 = 20 | AF23 = 22 |
| Class 3 | AF31 = 26 | AF32 = 28 | AF33 = 30 |
| Class 4 | AF41 = 34 | AF42 = 36 | AF43 = 38 |

Easy formula: **DSCP = (8 x class) + (2 x drop)**. AF31 is 8x3 + 2x1 = 26.

Common trap: AF4 is not automatically better than AF1. The standard does not rank the classes. A class gets only what you configure for it. Also, AF12 is not "worse quality" than AF11 in order or speed. Packets in the same AF class share a queue, and only the drop chance differs. Keeping one queue stops packets of the same flow from going out of order.

**The ToS byte is DSCP shifted left by 2 bits.** Tools often show the whole byte, so you must convert:

| Name | DSCP decimal | DSCP binary | ToS byte decimal | ToS byte hex |
|---|---|---|---|---|
| EF | 46 | 101110 | 184 | 0xb8 |
| AF41 | 34 | 100010 | 136 | 0x88 |
| AF21 | 18 | 010010 | 72 | 0x48 |
| CS6 | 48 | 110000 | 192 | 0xc0 |
| CS3 | 24 | 011000 | 96 | 0x60 |
| CS1 | 8 | 001000 | 32 | 0x20 |

Typical marking plan. It is based on RFC 4594 (service class guidelines), but plans differ between companies:

| Traffic | DSCP | CoS | Treatment |
|---|---|---|---|
| Network control (OSPF, BGP) | CS6 (48) | 6 | Small protected queue |
| Voice media | EF (46) | 5 | Priority queue |
| Interactive video | AF41 (34) | 4 | Guaranteed bandwidth |
| Call signalling (SIP) | CS3 (24) or CS5 (40) | 3 or 5 | Small guaranteed bandwidth |
| Business apps | AF21 (18) | 2 | Guaranteed bandwidth |
| Bulk transfer | AF11 (10) | 1 | Guaranteed bandwidth, lower share |
| Normal traffic | DF (0) | 0 | Default queue |
| Scavenger (unwanted bulk) | CS1 (8) or LE (1) | 1 | Gets only what is spare |

RFC 4594 suggests CS5 for signalling. Many Cisco designs use CS3. Both exist in real networks, so check the local plan.

### The trust boundary

A PC can set DSCP 46 on all its packets. If the network believes it, every download becomes "voice". So you choose a line. Marks from inside the line are believed. Marks from outside are rewritten. This line is the **trust boundary**.

```text
 untrusted      |        trusted
                |
 +----+  +-------+  +--------+  +-------------+  +--------+
 | PC |--| Phone |--| Access |--| Distribution|--| Router |
 +----+  +-------+  | switch |  +-------------+  +--------+
 CoS 0   CoS 5, EF  +--------+
                        ^
                        |
          Trust boundary is the access port
          (or the phone, if the switch trusts it)
```

Put the boundary as close to the source as you can. Classify and mark there. Everything behind it then trusts the marks.

### Step 3: Queuing and scheduling

Queuing means holding packets in lines. Scheduling means picking which line sends next.

A key detail: an interface has a small hardware FIFO queue, the **transmit ring (tx-ring)**. Packets go there just before the wire. Software queues only fill when the tx-ring is full. That is why QoS queuing does nothing on a link that is not busy.

```text
 Packets --> [ software QoS queues ] --> [ tx-ring ] --> wire
              used only when               small
              tx-ring is full              hardware FIFO
```

**FIFO (First In, First Out).** One line. Packets leave in the order they arrive.

- Simple and fast.
- No idea of priority.
- It is the normal default on fast interfaces.

**PQ (Priority Queuing).** Several queues with strict order. The scheduler always empties the high queue first. It serves the lower queues only when the higher ones are empty.

- Lowest possible delay for the top queue.
- Big risk: **starvation**. If the high queue never empties, lower queues never get a turn.

**Round robin and its family.** The scheduler visits each queue in turn. Plain round robin counts packets, so a queue with big packets gets more bandwidth. WRR (Weighted Round Robin) gives bigger turns to some queues. DRR and DWRR (Deficit Round Robin and Deficit Weighted Round Robin) count bytes, which is fairer. Many modern switch chips use this.

**WFQ (Weighted Fair Queuing).** An older method that gives each flow its own share automatically. You do not control the groups. It was the default on slow serial links in older Cisco IOS. Now it is mostly history, but its math lives on in CBWFQ.

**CBWFQ (Class-Based Weighted Fair Queuing).** You define classes and give each class a minimum bandwidth.

- Each class gets its own queue.
- The bandwidth is a **minimum guarantee during congestion**. It is not a limit.
- If a class does not use its share, others may use it.
- Packets in a class that are over its share may be dropped if the link is full.

Weakness: it has no strict priority. A voice packet still waits for its turn behind other queues. That adds jitter.

**LLQ (Low Latency Queuing).** LLQ is CBWFQ plus one strict priority queue.

- Priority class packets go first, always.
- During congestion, the priority class is **policed** to its configured rate. Traffic above that rate is dropped.
- This policer is what stops starvation. PQ has no such limit.

```text
                     +---------------------------+
 EF voice ---------->| Priority queue (policed)  |-----+
                     +---------------------------+     |
                        always served first            v
                     +---------------------------+  +-------+
 AF41 video -------->| CBWFQ: 30% minimum        |->|       |
                     +---------------------------+  | Sched |--> wire
 AF21 apps --------->| CBWFQ: 25% minimum        |->|       |
                     +---------------------------+  |       |
 Everything else --->| class-default: 20% minimum|->|       |
                     +---------------------------+  +-------+
```

**Analogy:** LLQ is an ambulance lane with a speed camera. Ambulances never wait. But if too many vehicles try to use the lane, the camera stops them, so normal lanes keep moving. **Where it stops:** a real ambulance is never fined. In LLQ, the extra voice packets are simply dropped, and all calls in that class can suffer.

| Method | Strict priority? | Bandwidth guarantee? | Starvation risk? | Common use |
|---|---|---|---|---|
| FIFO | No | No | No | Default on fast links |
| PQ | Yes | No | Yes | Legacy |
| WFQ | No | By flow, automatic | No | Legacy, slow links |
| CBWFQ | No | Yes, by class | No | Data classes |
| LLQ | Yes, for one class | Yes | No, the priority class is policed | Voice with data |

### Step 4: Congestion avoidance

A queue has a limit. When it is full, every new packet is dropped. This is **tail drop**.

Tail drop creates a real problem with TCP. TCP is the transport protocol that resends lost data and slows down when it sees loss. If many TCP flows lose packets at the same moment, they all slow down at the same moment. Then they all speed up together and fill the queue again. This is **TCP global synchronization**.

```text
 Link use
 100% |    /|    /|    /|     all flows grow, queue fills
      |   / |   / |   / |
      |  /  |  /  |  /  |
  50% | /   | /   | /   |     tail drop hits all flows at once
      |/    |/    |/    |/
      +-------------------------> time
```

The link swings between full and half empty. Bandwidth is wasted.

**RED (Random Early Detection)** drops a few random packets before the queue is full. Only some flows slow down, and not all at the same time.

**WRED (Weighted RED)** does the same, but with different thresholds per DSCP or precedence. Higher drop precedence gets dropped earlier.

WRED has three settings:

- **Minimum threshold:** below this average queue depth, nothing is dropped.
- **Maximum threshold:** above this, all new packets in that profile are dropped.
- **Mark probability denominator (MPD):** at the maximum threshold, 1 packet in MPD is dropped.

```text
 Drop chance
 100% |                         +-------------
      |                         |
      |                         |
 10%  |                       / |     1/MPD, here MPD = 10
      |                    /    |
      |                 /       |
   0% +---------------+---------+------------> avg queue depth
                     min       max
```

Worked example: min = 20 packets, max = 40 packets, MPD = 10, average depth = 30 packets.

- 30 is halfway from 20 to 40.
- The chance at 40 is 1/10, so 10 percent.
- The chance now is half of that, so **5 percent**.

WRED uses the **average** queue depth, not the current depth. So a short burst does not cause drops.

Example profile in one AF class, on a queue with a limit of 64 packets:

| DSCP | Min | Max | Effect |
|---|---|---|---|
| AF21 | 28 | 40 | Dropped last |
| AF22 | 24 | 40 | Dropped earlier |
| AF23 | 20 | 40 | Dropped first |

**Analogy:** a bouncer turns a few people away before the club is full, so there is no crowd crush at the door. **Where it stops:** a person sent away knows why. A TCP sender only sees a missing packet and has to guess.

Only use WRED on TCP traffic. A UDP voice call does not slow down when it loses a packet, so a drop only hurts the call.

**ECN (Explicit Congestion Notification, RFC 3168)** is a kinder signal. If both ends support it, the router sets a mark instead of dropping the packet. The receiver tells the sender, and the sender slows down. No data is lost.

| ECN bits | Meaning |
|---|---|
| 00 | Sender does not support ECN |
| 01 | ECN capable, ECT(1) |
| 10 | ECN capable, ECT(0) |
| 11 | CE, Congestion Experienced |

### Step 5: The token bucket

Policers and shapers both need to measure speed. A bit is either on the wire or not, so a "rate" only has meaning over time. The token bucket is the classic way to measure this.

Terms:

- **CIR (Committed Information Rate):** the average speed you are allowed, in bits per second.
- **Bc (Committed Burst):** the size of the bucket. It is how much can be sent in one go.
- **Tc (Committed Time Interval):** the time window. **Tc = Bc / CIR**.
- **Be (Excess Burst):** extra allowance saved from quiet periods.

How it works:

1. Tokens drop into a bucket at the CIR rate. One token allows one byte (or one bit, depending on the tool).
2. The bucket holds at most Bc tokens. Extra tokens spill out and are lost.
3. A packet arrives. The device checks if there are enough tokens for its size.
4. If yes, it removes those tokens and the packet **conforms**.
5. If no, the packet **exceeds**.
6. A policer drops or re-marks it. A shaper makes it wait for tokens.

```text
        tokens fall in at CIR rate
                 |
                 v
          |  o  o  o  o  |
          |  o  o  o  o  |   <-- bucket, size Bc
          |  o  o  o  o  |       overflow is lost
          +------+-------+
                 |
 Packet ---> [ enough tokens? ] --yes--> send (conform)
                 |
                 no
                 v
         police: drop or re-mark
         shape:  wait in a queue
```

**Analogy:** a prepaid phone card that gets a little credit every second, up to a maximum balance. You can make a big call if you saved credit. You cannot spend what you do not have. **Where it stops:** a real card might have no maximum, but a token bucket always has one. This cap is what limits bursts.

Worked example. CIR = 8 Mbps, which is 1,000 bytes per ms. Bc = 3,000 bytes. Every packet is 1,500 bytes. One token is one byte.

| Time (ms) | Tokens before | Result | Tokens after |
|---|---|---|---|
| 0.0 | 3,000 (full) | Conform | 1,500 |
| 0.1 | 1,600 | Conform | 100 |
| 0.2 | 200 | Exceed | 200 |
| 1.2 | 1,200 | Exceed | 1,200 |
| 1.5 | 1,500 | Conform | 0 |
| 6.0 | 3,000 (capped, not 4,500) | Conform | 1,500 |

Rule to remember: over any time T, a bucket lets through at most **Bc + (CIR x T)**. Traffic can pass at line speed for a moment, but the long-term average stays at CIR.

Fuller truth: real devices do not add tokens on a clock. They work out how many tokens to add when a packet arrives, based on the time since the last packet. The result is the same.

### Step 6: Policing vs shaping

Both enforce a rate. They differ in what they do with the excess.

```text
 Offered traffic        After policing         After shaping

 rate                   rate                   rate
  |   _     _            |                      |
  |  | |   | |           |  __    __            |  _______
 C|--|-|---|-|--      C  |-|--|--|--|--      C  |-|-------|-----
  |__| |___| |__         |_|  |__|  |__         |_|       |_____
  +------------> t       +------------> t       +------------> t

                         Excess dropped         Excess delayed
```

| Feature | Policing | Shaping |
|---|---|---|
| Excess traffic | Dropped or re-marked | Held in a queue, sent later |
| Adds delay? | No | Yes |
| Adds loss? | Yes | Only if the shaper queue fills up |
| Needs a buffer? | No | Yes |
| Direction | In or out | Normally out |
| Result on the wire | Saw-tooth edges | Smooth |
| Effect on TCP | Drops cause slowdowns and resends | Fewer drops, flows slow down gently |
| Typical place | ISP edge, enforcing a contract | Customer edge, staying within a contract |

**Analogy:** a policer is a speed camera. Go too fast and you are punished at once. A shaper is a traffic light on a motorway on-ramp. It holds cars and lets them in at a steady pace. **Where it stops:** a car held at a ramp keeps its place forever. A shaper queue is finite, so it can still drop.

Policer action names:

- **Conform:** within the rate. Usually transmit.
- **Exceed:** over the rate. Drop or re-mark to a higher drop level.
- **Violate:** over a second limit. Usually drop. Only three-colour policers have this.

| Policer type | Rates | Buckets | Standard |
|---|---|---|---|
| Single-rate, two-colour | CIR | One | Common basic form |
| Single-rate, three-colour | CIR | Two, Bc and Be. Spare tokens overflow from the first to the second | RFC 2697 |
| Two-rate, three-colour | CIR and PIR | Two, each filled at its own rate | RFC 2698 |

PIR (Peak Information Rate) is the maximum speed allowed.

### Step 7: Voice requirements

Voice is the reason most QoS exists.

| Measure | Target | Source |
|---|---|---|
| One-way delay | 150 ms or less, mouth to ear | ITU-T G.114. Up to 400 ms is the planning limit |
| Jitter | 30 ms or less | Common vendor design guide |
| Packet loss | 1 percent or less | Common vendor design guide |
| Bandwidth | 87.2 kbps per G.711 call on Ethernet | Calculation below |

A codec turns voice into bits. G.711 sends 64 kbps. G.729 compresses this to 8 kbps. Most phones put 20 ms of sound into each packet, so there are 50 packets per second.

Voice packet layout with G.711:

```text
 +----------+----------+---------+----------+-------------+
 | Ethernet |   IPv4   |   UDP   |   RTP    | Voice data  |
 | 18 bytes | 20 bytes | 8 bytes | 12 bytes | 160 bytes   |
 +----------+----------+---------+----------+-------------+
            |<---------- 200 bytes (IP) ---------------->|
 |<----------------- 218 bytes (frame) ----------------->|
```

RTP (Real-time Transport Protocol) carries the sound. Calculation for G.711:

- Payload: 64,000 bits/s x 0.020 s = 1,280 bits = 160 bytes.
- IP packet: 160 + 12 + 8 + 20 = 200 bytes.
- IP rate: 200 x 8 x 50 = 80,000 bits/s = 80 kbps.
- Ethernet frame: 200 + 18 = 218 bytes (14 header + 4 FCS).
- Ethernet rate: 218 x 8 x 50 = 87,200 bits/s = **87.2 kbps**.

| Codec | Codec rate | Payload | IP packet | kbps at IP | kbps on Ethernet |
|---|---|---|---|---|---|
| G.711 | 64 kbps | 160 bytes | 200 bytes | 80 | 87.2 |
| G.729 | 8 kbps | 20 bytes | 60 bytes | 24 | 31.2 |

Both assume 20 ms packets and no VLAN tag. A VLAN tag adds 4 bytes per frame. If you also count preamble and the gap between frames, add 20 more bytes. Modern apps often use Opus (RFC 6716), a codec whose bit rate changes with the sound and the network. For those, measure the real traffic.

**How jitter becomes loss.** A receiver has a **jitter buffer**. It holds packets for a short time, then plays them at a steady pace.

**Analogy:** buses arrive at uneven times, but a school releases children from the hall at exact times. A short wait hides the unevenness. **Where it stops:** a late child still joins the class. A voice packet that arrives after its play time is thrown away. So high jitter acts like packet loss.

A sample one-way delay budget:

| Part | Delay |
|---|---|
| Codec and packing 20 ms of sound | 25 ms |
| Network queuing with LLQ | 5 ms |
| Serialization along the path | 1 ms |
| Propagation, 10,000 km of fibre | 50 ms |
| Jitter buffer at the receiver | 40 ms |
| **Total** | **121 ms** (within 150 ms) |

Interactive video has similar delay and loss needs, but it uses much more bandwidth and its traffic comes in bursts. Streaming video (one way, such as a recorded training) is more relaxed, because the player can buffer several seconds.

## Example

### The scenario

A branch has 30 phones, 60 PCs and a contract for 20 Mbps WAN. The ISP gives a 100 Mbps Ethernet handoff and polices at 20 Mbps.

```text
 Voice VLAN 110: 10.10.110.0/24
 Data  VLAN 10:  10.10.10.0/24

 +--------+    +--------+ Gi0/0/1          +-----+       +----+
 | Switch |----|   R1   |------------------| ISP |-------| HQ |
 +--------+    +--------+ 172.16.0.2/30    +-----+       +----+
        Gi0/0/0     100 Mbps port,        polices        10.20.0.0/16
                    20 Mbps contract      at 20 Mbps
```

HQ servers: ERP app at 10.20.30.10 (TCP 8443), backup at 10.20.30.50 (TCP 873), video bridge at 10.20.40.10 (UDP 50000-50100).

### Step 1: Do the math

Voice: 30 G.711 calls x 87.2 kbps = 2,616 kbps. Round up to 3,000 kbps.

| Class | DSCP | Rate | Share of 20 Mbps | Queue type |
|---|---|---|---|---|
| VOICE | EF | 3,000 kbps | 15% | Priority |
| NET-CONTROL | CS6 | 500 kbps | 2.5% | CBWFQ |
| SIGNALING | CS3 | 500 kbps | 2.5% | CBWFQ |
| VIDEO | AF41 | 6,000 kbps | 30% | CBWFQ |
| CRITICAL-DATA | AF21 | 5,000 kbps | 25% | CBWFQ with WRED |
| SCAVENGER | CS1 | 1,000 kbps | 5% | CBWFQ |
| class-default | DF | 4,000 kbps | 20% | CBWFQ, fair-queue |
| **Total** | | **20,000 kbps** | **100%** | |

The priority share is 15 percent. A common Cisco design guide is to keep strict priority at or under about 33 percent of the link, so other traffic is not squeezed.

### Step 2: Classify and mark on the LAN side

Cisco IOS:

```text
ip access-list extended ACL-RTP
 permit udp 10.10.110.0 0.0.0.255 any range 16384 32767
ip access-list extended ACL-SIP
 permit udp 10.10.110.0 0.0.0.255 any eq 5060
 permit tcp 10.10.110.0 0.0.0.255 any eq 5060
ip access-list extended ACL-VIDEO
 permit udp 10.10.10.0 0.0.0.255 host 10.20.40.10 range 50000 50100
ip access-list extended ACL-ERP
 permit tcp 10.10.10.0 0.0.0.255 host 10.20.30.10 eq 8443
ip access-list extended ACL-BACKUP
 permit tcp 10.10.10.0 0.0.0.255 host 10.20.30.50 eq 873

class-map match-any MARK-VOICE
 match access-group name ACL-RTP
class-map match-any MARK-SIGNALING
 match access-group name ACL-SIP
class-map match-any MARK-VIDEO
 match access-group name ACL-VIDEO
class-map match-any MARK-ERP
 match access-group name ACL-ERP
class-map match-any MARK-BACKUP
 match access-group name ACL-BACKUP

policy-map LAN-IN
 class MARK-VOICE
  set dscp ef
 class MARK-SIGNALING
  set dscp cs3
 class MARK-VIDEO
  set dscp af41
 class MARK-ERP
  set dscp af21
 class MARK-BACKUP
  set dscp cs1
 class class-default
  set dscp default

interface GigabitEthernet0/0/0
 description LAN trunk, VLANs 10 and 110
 service-policy input LAN-IN
```

The last class rewrites everything else to 0. That stops a PC from giving itself high priority. The match can also use NBAR, for example `match protocol rtp`. NBAR depends on the router's software and licence, and on what is visible in the traffic.

### Step 3: Build the queues (the child policy)

```text
class-map match-any VOICE
 match dscp ef
class-map match-any NET-CONTROL
 match dscp cs6
class-map match-any SIGNALING
 match dscp cs3
class-map match-any VIDEO
 match dscp af41 af42 af43
class-map match-any CRITICAL-DATA
 match dscp af21 af22 af23
class-map match-any SCAVENGER
 match dscp cs1

policy-map WAN-CHILD
 class VOICE
  priority 3000
 class NET-CONTROL
  bandwidth 500
 class SIGNALING
  bandwidth 500
 class VIDEO
  bandwidth 6000
 class CRITICAL-DATA
  bandwidth 5000
  random-detect dscp-based
  random-detect dscp af21 28 40 10
  random-detect dscp af22 24 40 10
  random-detect dscp af23 20 40 10
 class SCAVENGER
  bandwidth 1000
 class class-default
  bandwidth 4000
  fair-queue
```

Units matter here. `priority` and `bandwidth` take **kbps**. `shape average` takes **bits per second**. In `random-detect`, the numbers are min, max and MPD, and the thresholds are in packets on most platforms.

Queues on this router's WAN port would never fill, because the port runs at 100 Mbps and we only send 20 Mbps. The congestion happens in the ISP's policer, where our priority queue has no effect. So we need one more step.

### Step 4: Shape to the contract (the parent policy)

```text
policy-map WAN-PARENT
 class class-default
  shape average 20000000
  service-policy WAN-CHILD

interface GigabitEthernet0/0/1
 description WAN to ISP, 100 Mbps port, 20 Mbps contract
 ip address 172.16.0.2 255.255.255.252
 service-policy output WAN-PARENT
```

This is **Hierarchical QoS (HQoS)**. The parent shaper creates a fake "20 Mbps wire". When traffic is over 20 Mbps, the shaper queues it, and this makes the child queues fill. Then LLQ can work.

```text
 Packets -> [ child: LLQ + CBWFQ ] -> [ parent: shaper ] -> port
             picks the ORDER           sets the SPEED
```

Practical note: the shaper and the ISP may not count the same header bytes. Many engineers shape a little below the contract (for example 95 percent), so the ISP policer never fires. Check with your provider.

### Step 5: Verify

Cisco IOS. Output is shortened, and the layout changes by platform and version.

```text
R1# show policy-map interface GigabitEthernet0/0/1 output
 GigabitEthernet0/0/1

  Service-policy output: WAN-PARENT

    Class-map: class-default (match-any)
      30 second offered rate 23887000 bps, drop rate 3890000 bps
      Match: any
      Queueing
      queue limit 64 packets
      (queue depth/total drops/no-buffer drops) 0/61208/0
      shape (average) cir 20000000, bc 80000, be 80000
      target shape rate 20000000

      Service-policy : WAN-CHILD

        Class-map: VOICE (match-any)
          1204100 packets, 257677400 bytes
          30 second offered rate 2397000 bps, drop rate 0000 bps
          Match: dscp ef (46)
          Priority: 3000 kbps, burst bytes 75000, b/w exceed drops: 0

        Class-map: SCAVENGER (match-any)
          30 second offered rate 3200000 bps, drop rate 980000 bps
          Match: dscp cs1 (8)
          Queueing
          queue limit 64 packets
          (queue depth/total drops/no-buffer drops) 21/18400/0
          bandwidth 1000 kbps

        Class-map: class-default (match-any)
          30 second offered rate 8120000 bps, drop rate 2910000 bps
          Match: any
          Queueing
          queue limit 64 packets
          (queue depth/total drops/no-buffer drops) 37/42808/0
          bandwidth 4000 kbps
```

How to read it:

- Total offered is 23.887 Mbps, and the shaper sends about 20 Mbps. So 3.89 Mbps is dropped.
- Drops: 18,400 in SCAVENGER + 42,808 in class-default = 61,208, which matches the parent counter.
- Voice: 2.397 Mbps offered and **0 drops**. Calls sound clean during the overload.
- Voice packets: 257,677,400 / 1,204,100 = 214 bytes each. That is the 200 byte IP packet plus a 14 byte Ethernet header.
- 28 calls x 50 packets x 214 bytes x 8 bits = 2,396,800 bits/s. This matches the 2.397 Mbps rate.
- Burst of 75,000 bytes is 200 ms of traffic at 3 Mbps. This is a common default on Cisco IOS.
- The scavenger class got 2.22 Mbps, more than its 1 Mbps minimum, because spare bandwidth is shared.
- Shaper: bc 80,000 bits at 20 Mbps gives Tc = 80,000 / 20,000,000 = 4 ms. This default varies by platform.

### Step 6: Check the marks on the wire

Linux, using [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) on a softphone host at 10.10.110.21:

```bash
# Show only packets with DSCP 46 (top 6 bits of the ToS byte)
sudo tcpdump -ni eth0 -v 'ip[1] & 0xfc == 0xb8'
```

```text
14:02:11.512344 IP (tos 0xb8, ttl 64, id 5012, offset 0, flags [DF],
proto UDP (17), length 200)
    10.10.110.21.16504 > 10.20.110.35.17210: UDP, length 172
```

`tos 0xb8` is DSCP 46. Length 200 is the IP packet. The UDP payload of 172 is 12 bytes of RTP plus 160 bytes of voice.

In [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md), use the display filter `ip.dsfield.dscp == 46`.

Test a marked ping:

```bash
# Linux: send pings with ToS byte 184 (DSCP 46)
ping -Q 184 -c 4 10.20.110.35
```

```text
! Cisco IOS: extended ping with the same ToS byte
R1# ping 10.20.110.35 tos 184
```

### Access switch for a phone

Cisco Catalyst. The commands differ between switch families.

```text
! Catalyst 9000 series (IOS XE)
interface GigabitEthernet1/0/10
 description IP phone with a PC behind it
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 110
 trust device cisco-phone

! Older Catalyst 2960 or 3560 style
mls qos
interface GigabitEthernet0/10
 switchport access vlan 10
 switchport voice vlan 110
 mls qos trust device cisco-phone
 mls qos trust cos
```

`auto qos voip cisco-phone` is a one-line macro on many Catalyst models. It builds a recommended policy, so check what it created.

Old switch trap: on a 2960 or 3560, `mls qos` turns QoS on, and from then on ports are untrusted by default, so marks get reset to 0. Catalyst 9000 switches trust marks by default.

### Policing a customer: ISP side

A provider sells 10 Mbps committed and allows bursts up to 20 Mbps. It uses a two-rate, three-colour policer.

```text
policy-map CUSTOMER-IN
 class class-default
  police cir 10000000 bc 31250 pir 20000000 be 62500
   conform-action transmit
   exceed-action set-dscp-transmit af12
   violate-action drop

interface GigabitEthernet0/1
 description Customer A
 service-policy input CUSTOMER-IN
```

- Up to 10 Mbps: sent as it is.
- From 10 to 20 Mbps: sent, but re-marked to AF12, so it is dropped first if the core is busy.
- Above 20 Mbps: dropped.
- `bc` and `be` here are in **bytes**. 31,250 bytes is 25 ms at 10 Mbps. 62,500 bytes is 25 ms at 20 Mbps.

### The same ideas on Linux and Windows

Linux, using `tc` (traffic control) with HTB (Hierarchical Token Bucket). This shapes eth0 to 20 Mbit/s and gives EF packets a high-priority class:

```bash
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 20mbit ceil 20mbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 3mbit ceil 3mbit prio 0
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 17mbit ceil 20mbit prio 1
tc qdisc add dev eth0 parent 1:30 fq_codel
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
  match ip dsfield 0xb8 0xfc flowid 1:10

# See counters
tc -s class show dev eth0

# Mark outgoing RTP as EF with iptables
iptables -t mangle -A OUTPUT -p udp --dport 16384:32767 \
  -j DSCP --set-dscp-class EF

# Police incoming traffic to 10 Mbit/s
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ffff: protocol ip prio 1 matchall \
  action police rate 10mbit burst 100k conform-exceed drop
```

Windows, in an elevated PowerShell:

```powershell
# Mark outgoing UDP 16384-32767 as DSCP 46
New-NetQosPolicy -Name "Voice" -IPProtocolMatchCondition UDP `
  -IPDstPortStartMatchCondition 16384 -IPDstPortEndMatchCondition 32767 `
  -DSCPAction 46 -NetworkProfile All

# Limit one program to 10 Mbit/s
New-NetQosPolicy -Name "Backup" -AppPathNameMatchCondition "backup.exe" `
  -ThrottleRateActionBitsPerSecond 10000000 -NetworkProfile All

# List the policies
Get-NetQosPolicy
```

## Going Deeper

### Shaping interval and voice

A shaper does not really run at the CIR. The port always sends at its full speed. The shaper sends Bc bits, then stops until the next Tc.

Example: 20 Mbps shaper on a 100 Mbps port, Bc = 200,000 bits.

- Tc = 200,000 / 20,000,000 = 10 ms.
- Sending 200,000 bits at 100 Mbps takes 2 ms.
- The port sends for 2 ms, then stays quiet for 8 ms.

```text
 100 Mbps  ##........##........##........
           |<- 10ms ->|<- 10ms ->|
           2 ms sending, 8 ms silent. Average is 20 Mbps.
```

A voice packet that arrives just after the burst waits up to 8 ms. So on links that carry voice, keep Tc small. A common guide is 10 ms or less. A smaller Bc gives a smaller Tc, but a Bc that is too small can hurt throughput.

Cisco trap: in `shape`, Bc is in **bits**. In `police`, Bc is in **bytes**.

### How spare bandwidth is shared

`bandwidth` is a minimum. When classes have idle bandwidth, active classes share it. The split rule depends on the software. Older IOS shares it in proportion to `bandwidth` values. Newer schedulers may share it equally by default, unless you set `bandwidth remaining ratio` or `bandwidth remaining percent`.

Older IOS also limited total reserved bandwidth to 75 percent of the interface, unless you changed `max-reserved-bandwidth`. Check the platform. Most IOS versions also expect one unit per policy, so do not mix kbps and percent in the same policy.

### More than one priority class

On classic IOS, you can have `priority` under two classes (for example voice and video). They share one priority queue, but each has its own policer. A burst of video can then add jitter to voice. Some platforms offer `priority level 1` and `priority level 2`, which keep them in separate strict levels.

Putting video in the priority queue is a design choice with a cost. Video is bursty, so many designs keep it in CBWFQ. Others accept the risk.

### Switch queue names

Switch hardware has a fixed set of queues. Names like **1P3Q3T** mean 1 priority queue, 3 normal queues, and 3 drop thresholds per queue. The exact layout depends on the switch model.

### The CoS to DSCP trap

Many Catalyst switches map CoS to DSCP by simple multiplication by 8. So CoS 5 becomes DSCP 40, not 46. That is CS5, not EF. If your router matches only EF, voice falls into the wrong class. On older Catalyst models, fix the map:

```text
mls qos map cos-dscp 0 8 16 24 32 46 48 56
```

### Marks in tunnels

When a router wraps a packet in GRE or IPsec, the original header is hidden. Cisco IOS copies the ToS byte to the outer header by default. So DSCP queuing still works. But rules that match inner ports or IPs will not work after encryption, unless you use `qos pre-classify`, which classifies on the original header.

Another trap: [IPsec](../09-security/08-ipsec.md) has an anti-replay window. If QoS delays low-priority packets a lot, they can arrive outside the window and be dropped by the receiver, even though they are valid.

### QoS across MPLS

An MPLS label has a 3-bit Traffic Class field, formerly called EXP (RFC 5462). A provider maps your DSCP to it, because core routers read the label, not the IP header. RFC 3270 describes three tunnel models: Uniform, Pipe and Short Pipe. The difference is whether the provider's marks replace yours when the packet leaves. See [MPLS](../05-routing/13-mpls.md).

### QoS and the Internet

The public Internet does not honour your DSCP. Providers may ignore it, zero it or remark it. So LLQ only protects you on links you control, up to the first congested point you do not own. Over Internet links, you can still shape on your exit. That moves the queue into your own router, where you can sort it. Tools such as [SD-WAN](10-sd-wan.md) also pick the best path by measured loss and delay. See [WAN Technologies](08-wan-technologies.md) for private circuits that do offer QoS classes.

### QoS on Wi-Fi

Wi-Fi uses WMM (Wi-Fi Multimedia), which has four access categories: voice, video, best effort and background. Each one waits a different random time before it sends. RFC 8325 gives a guide for mapping DSCP to these categories. See [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).

### Call Admission Control

LLQ protects the voice class, but it cannot limit the number of calls. If the priority class fits 34 calls and call 35 starts, the policer drops packets across **all** calls during congestion. All calls sound bad, not just the new one.

CAC (Call Admission Control) solves this. The call system counts calls per site and refuses or reroutes a new call when the limit is reached. The caller gets a busy tone instead of harming others.

### Bufferbloat and modern queue management

Big buffers avoid loss, but they create long delay, because packets sit in a full queue for hundreds of milliseconds. This is **bufferbloat**.

AQM (Active Queue Management) means dropping or marking packets before the queue is full. RFC 7567 recommends it. RED was the first well-known AQM, but it is hard to tune. Newer methods need less tuning:

- **CoDel** (RFC 8289) watches how long packets sit in the queue, not how many packets there are.
- **FQ-CoDel** (RFC 8290) gives flows their own small queues and runs CoDel on each. It is a common Linux default.
- **PIE** (RFC 8033) is another delay-based AQM.
- **L4S** (RFC 9330, 9331, 9332) aims for very low queuing delay by using ECN marks with scalable congestion control.

### Data centres

Links are 25 to 400 Gbps, and average use is often low. But **microbursts** happen. A microburst is a very short surge that lasts a few milliseconds or less. Many servers reply at once to one host, and a shallow buffer overflows. This is called incast. A monitor that shows a 5-minute average of 20 percent can hide these drops. Check interface output drops, not only the average use. See [Network Monitoring](../12-troubleshooting-and-tools/11-network-monitoring.md).

Some storage and GPU traffic, such as RoCEv2 (RDMA over Converged Ethernet), expects a network with almost no loss. It uses:

- **PFC (Priority Flow Control, IEEE 802.1Qbb):** a switch tells its neighbour to pause one CoS value only, instead of pausing the whole link.
- **ETS (Enhanced Transmission Selection, IEEE 802.1Qaz):** shares link bandwidth between traffic classes.
- **ECN:** marks packets early, so senders slow down before PFC is needed.

PFC has risks too, such as pause storms that spread across the fabric. See [Spine-Leaf Architecture](02-spine-leaf.md).

### Old tools for slow links

These help on very slow links, and you may still see them in exams:

- **LFI (Link Fragmentation and Interleaving):** cuts big packets into small pieces and puts voice packets between them. This lowers serialization delay.
- **cRTP (Compressed RTP, RFC 2508):** shrinks the 40 byte IP/UDP/RTP header to a few bytes.

## Common Mistakes

- **"QoS adds bandwidth."** It does not. It only picks who suffers during congestion. If voice alone is more than the link, QoS cannot help.
- **Queuing on a link that never fills.** A 100 Mbps port with a 20 Mbps contract never queues on the router. The drops happen at the ISP's policer. You need a shaper in a parent policy.
- **Marking everything as high priority.** If everything is special, nothing is. Priority for all is the same as FIFO.
- **Trusting end devices.** Without a trust boundary, a PC can mark its downloads as EF.
- **Thinking QoS works end to end by default.** Every hop must be configured. One unconfigured congested device breaks the plan. The public Internet will not honour your marks.
- **Believing CoS survives routing.** It lives in the Layer 2 tag. Routers remove it. DSCP carries the value across Layer 3.
- **Mixing up DSCP and the ToS byte.** DSCP 46 is ToS byte 184 (0xb8). They are the same mark shown in two ways.
- **Assuming CoS 5 becomes DSCP 46.** Default maps on many switches give DSCP 40.
- **Using WRED on voice.** Voice is UDP and does not slow down on loss. WRED just damages calls.
- **Thinking `bandwidth` is a speed cap.** It is a minimum during congestion. Use a policer or a shaper for a cap.
- **Forgetting the LLQ policer.** During congestion, too many calls cause drops inside the priority class, even while data classes are still sending.
- **Reading a 5-minute average and saying "no congestion".** Bursts can overflow buffers inside a few milliseconds. Check the drop counters.
- **Wrong units.** `priority` and `bandwidth` use kbps, `shape` and `police` use bits per second, shaper Bc uses bits, and policer Bc uses bytes.
- **Applying a queuing policy inbound.** Queues are for the exit. Input policies are for marking and policing.
- **Not sizing for real packet overhead.** A G.711 call is 87.2 kbps on Ethernet, not 64 kbps.

## Interview Questions

**Q: What is QoS and when does it help?**

A: QoS is a set of tools that give different treatment to different traffic. It classifies, marks, queues, drops and limits packets. It only helps during congestion, when a queue forms. It does not add bandwidth. It decides who gets the bandwidth and who waits.

**Q: What are the differences between CoS and DSCP?**

A: CoS is a 3-bit Layer 2 field in the 802.1Q tag, with values 0 to 7. It is lost when a router removes the Layer 2 header. DSCP is a 6-bit Layer 3 field in the IP header, with values 0 to 63. It stays with the packet end to end unless a device rewrites it.

**Q: What is DSCP 46, and why is it used for voice?**

A: DSCP 46 is Expedited Forwarding. It asks for low delay, low jitter and low loss. Voice needs this, because a late packet is useless. In the ToS byte it shows as 184, or 0xb8. Devices normally put it into a strict priority queue with a rate limit.

**Q: Explain the AF naming. What does AF31 mean?**

A: AFxy means class x and drop precedence y. AF31 is class 3 with low drop precedence. Its value is 8 x 3 + 2 x 1 = 26. Inside class 3, AF33 is dropped before AF32, and AF32 before AF31. They share a queue, so packets stay in order.

**Q: What is the difference between policing and shaping?**

A: Both enforce a rate using a token bucket. A policer drops or re-marks the excess at once, so it adds no delay but causes loss. A shaper buffers the excess and sends it later, so it adds delay but causes less loss and works better with TCP. ISPs police on their side, and customers shape to stay within the contract.

**Q: How does a token bucket work?**

A: Tokens are added to a bucket at the CIR rate, and the bucket has a maximum size, Bc. A packet can pass if there are enough tokens for its size, and those tokens are removed. If not, it exceeds and is dropped, re-marked or delayed. This allows short bursts but keeps the long-term average at CIR. The time interval is Tc = Bc / CIR.

**Q: What is the difference between PQ, CBWFQ and LLQ?**

A: PQ always serves the top queue first, so low queues can starve. CBWFQ gives each class a minimum bandwidth during congestion, but it has no strict priority, so voice can get jitter. LLQ adds one strict priority queue to CBWFQ. That queue is policed during congestion, so it cannot starve the others.

**Q: What is WRED and why do we use it?**

A: Tail drop can hit many TCP flows at once, and they all slow down together. That is global synchronization. WRED randomly drops a few packets as the average queue grows, so flows slow down at different times. It drops high drop-precedence packets earlier. It suits TCP traffic and not voice.

**Q: What are the network requirements for voice? How much bandwidth does one call use?**

A: The usual targets are one-way delay at most 150 ms, jitter at most 30 ms and loss at most 1 percent. G.711 with 20 ms packets sends 50 packets per second, each 200 bytes at IP level. That is 80 kbps at IP level, or 87.2 kbps on Ethernet. G.729 is 31.2 kbps on Ethernet.

**Q: My link is 100 Mbps, but the contract is 20 Mbps. LLQ is configured on the port, and voice is still bad. Why?**

A: The port never congests, because it can send at 100 Mbps, so the LLQ never activates. The ISP polices at 20 Mbps and drops packets without looking at priority. The fix is Hierarchical QoS: a parent shaper at 20 Mbps with the LLQ policy as its child. Then queuing happens on my router, where I control the order.

**Q: Where should you mark traffic, and what is a trust boundary?**

A: Mark as close to the source as possible, so all later devices can act on it. The trust boundary is the point where the network starts to believe marks. Normally that is the access switch port or a trusted IP phone. Outside it, marks are rewritten, so that users cannot give themselves priority.

**Q: If we have enough bandwidth, do we still need QoS?**

A: Often less, but it depends. Adding capacity is simple and works well inside a campus. But bursts still fill buffers for short times, and failures can cut capacity in half. WAN links are also costly. So I would over-provision first, then still use QoS at the choke points, mainly on WAN edges and where voice or storage traffic is sensitive.

## Quick Recap

- QoS manages bandwidth, delay, jitter and loss during congestion. It creates no capacity, and it acts only when a queue exists.
- The path is: classify, mark, police on the way in. Then queue, drop early, shape and schedule on the way out.
- CoS is 3 bits in the VLAN tag and is lost at a router. DSCP is 6 bits in the IP header. EF is 46, AFxy is 8x + 2y, and the ToS byte is DSCP x 4.
- FIFO has no priority, PQ can starve others, CBWFQ gives minimums, and LLQ adds a policed strict priority queue for voice.
- WRED drops early to stop TCP global synchronization. ECN marks instead of dropping.
- A token bucket measures rate with CIR, Bc and Tc = Bc / CIR. A policer drops the excess, and a shaper delays it.
- Voice needs at most 150 ms one way, 30 ms jitter and 1 percent loss, and a G.711 call uses 87.2 kbps on Ethernet. On a sub-rate handoff, use a parent shaper with a child LLQ.
