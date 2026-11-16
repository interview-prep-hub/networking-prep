# Bandwidth, Latency and Performance

**Level:** Beginner | **Prerequisites:** [What Is a Network?](01-what-is-a-network.md), [Cables and Transmission Media](05-cables-and-media.md)

## In One Line

Bandwidth is how much data a link can carry each second. Latency is how long data takes to arrive. Network performance depends on both, and they are different things.

## Why It Matters

Users never say "the network has a problem". They say "it is slow". Slow can mean many things:

- A large file takes too long: usually a bandwidth or throughput problem.
- A website feels slow even on a fast plan: often a latency problem.
- A video call breaks up: usually jitter or packet loss.

Each cause has a different fix. Buying a faster internet plan does nothing for a server that is 10,000 km away, because no plan can make light go faster.

Interviewers ask about this topic because it shows how you think. They want to see that you can separate speed from delay, that you can do the math in your head, and that you know why a "1 Gbps" link never gives you 1 Gbps of file transfer.

## Simple Explanation

Think of a highway.

- **Bandwidth** is the number of lanes. More lanes means more cars can pass each minute.
- **Latency** is the time one car takes to drive from city A to city B.
- **Throughput** is the number of cars that really arrive each minute, after traffic jams and accidents.
- **Goodput** is the number of passengers delivered. The cars themselves are not what you wanted to move. They are just the packaging.

A 10-lane highway does not make the drive shorter. If the speed limit is the same and the distance is the same, each car still needs the same time. This is the key idea: **more bandwidth does not mean less delay**.

Where the analogy stops: real links do not have side-by-side lanes for each flow. A faster link sends bits one after another at a higher rate, so the bits sit closer together on the wire. The signal itself does not move faster.

## How It Works

### Bits and bytes: Mbps vs MBps

A **bit** is the smallest unit of data: one 0 or one 1. A **byte** is 8 bits.

Networks measure speed in bits per second. Files are measured in bytes. This is the source of much confusion.

| Written | Meaning | Value |
|---|---|---|
| Mbps (small b) | megabits per second | 1,000,000 bits each second |
| MBps or MB/s (big B) | megabytes per second | 1,000,000 bytes each second |

To convert, divide bits by 8.

| Link speed | Same speed in bytes |
|---|---|
| 8 Mbps | 1 MB/s |
| 100 Mbps | 12.5 MB/s |
| 1 Gbps | 125 MB/s |
| 10 Gbps | 1.25 GB/s |

So a browser that shows 12 MB/s on a 100 Mbps plan is working well. Nothing is wrong.

Network speeds always use decimal prefixes: 1 kilo is 1,000 and 1 mega is 1,000,000. Storage is messier. **Binary prefixes** are based on 1,024: 1 MiB (mebibyte) is 1,048,576 bytes and 1 GiB (gibibyte) is 1,073,741,824 bytes. Windows File Explorer shows binary values but labels them "MB" and "GB". That is why a 1 TB drive appears as about 931 GB.

### Bandwidth, throughput and goodput

These three words describe three different rates.

- **Bandwidth:** the maximum rate of a link, set by the hardware or the contract. Example: 1 Gbps Ethernet.
- **Throughput:** the rate of data that is really delivered, measured over time. It is always at or below bandwidth.
- **Goodput:** the rate of useful application data only. It leaves out headers and any data that had to be sent again.

A **header** is extra information added to each packet so the network can deliver it, like an address on an envelope. A **packet** is one small chunk of data sent across the network. See [Encapsulation](../02-network-models/04-encapsulation.md) for how headers are added.

Here is a full-size TCP packet on Ethernet, with sizes in bytes:

```text
| Preamble | Eth    | IP | TCP | TCP     | App  | FCS | Gap |
| + SFD    | header |    |     | options | data |     |     |
|    8     |   14   | 20 | 20  |   12    | 1448 |  4  | 12  |

Total on the wire: 1538 bytes.  Useful data: 1448 bytes.
```

Goodput is 1448 / 1538 = 94.1% of the link. On a 1 Gbps link, that is about 941 Mbps. This is why a perfect gigabit LAN test shows about 940 Mbps, not 1000. Details of these fields are in [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md).

Note: the word "throughput" is used loosely in practice. Many tools and people say throughput when they mean goodput. In an interview, state which one you mean.

Fuller truth: "bandwidth" also has a second meaning in radio. There it means a range of frequencies in hertz, such as a 20 MHz Wi-Fi channel. More hertz allows more bits per second, but the two are not the same unit.

### The bottleneck

Data travels across many links. The whole path can only go as fast as its slowest link. That link is called the **bottleneck**.

```text
Server ==1 Gbps== ISP ==200 Mbps== Home router ~~80 Mbps Wi-Fi~~ Laptop
                                                       ^
                                     bottleneck: at most 80 Mbps
```

Your 200 Mbps plan does not help here. To go faster, upgrade the Wi-Fi link, not the plan.

### Latency and RTT

**Latency** is the time data takes to go from one point to another. It is measured in milliseconds (ms). One millisecond is 1/1000 of a second.

**RTT (round-trip time)** is the time to go there and come back. If latency is 15 ms each way, RTT is about 30 ms. The `ping` command measures RTT, not one-way latency. The two directions can differ, because traffic may take a different path back.

Rough RTT values you can expect (real values vary):

| Path | Typical RTT |
|---|---|
| Same wired LAN | under 1 ms |
| Home Wi-Fi to router | 1-10 ms |
| To a server in your own city | 5-20 ms |
| New York to London | 70-80 ms |
| Geostationary satellite | about 600 ms |

### Jitter

**Jitter** is the variation in latency from one packet to the next.

Analogy: a bus is due every 10 minutes. If buses come at 10, 9, 11, 10 minutes, you can plan your day. If they come at 2, 25, 3, 18 minutes, you cannot, even if the average is the same. Where the analogy stops: late buses stay in order, but late packets can arrive out of order.

Say five packets have RTTs of 20, 22, 19, 45 and 21 ms. The average is 25.4 ms, which looks fine. But the changes between neighbours are 2, 3, 26 and 24 ms, so the average change is 13.75 ms. That is high jitter.

Jitter hurts real-time traffic such as voice and video. Those apps use a **jitter buffer**: they hold packets for a short time, then play them at a steady rate. A bigger buffer hides more jitter but adds more delay.

Fuller truth: there are several ways to measure jitter. RTP (the protocol that carries voice and video) defines a smoothed formula in RFC 3550. The simple average above is only for learning.

### Packet loss

**Packet loss** means a packet was sent but never arrived. It is shown as a percentage: 2 lost out of 100 is 2% loss.

Common causes:

- **Congestion:** a queue in a router fills up and new packets are dropped. This is the most common cause.
- **Bad physical links:** a damaged cable or dirty fibre corrupts a packet, its checksum fails, and the receiver throws it away.
- **Wireless interference:** other radios and walls cause errors.
- **Rate limits:** a provider or firewall drops traffic above an allowed rate.

The effect depends on the protocol. **TCP** (the reliable transport protocol) detects the loss, sends the data again, and slows down. **UDP** (the simple transport protocol) does not resend anything, so a voice call just loses a tiny piece of sound. See [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md).

Common rule of thumb for voice calls: one-way delay under 150 ms (from ITU-T G.114), jitter under 30 ms, and loss under 1%. These are guidelines, not hard limits. Modern codecs can handle more.

### The four types of delay

Every packet meets four delays at each hop. A **hop** is one step from one device to the next. A **router** is a device that forwards packets between networks.

```text
 Packet arrives at Router A
        |
        v
 +-------------+    +--------+    +--------------+
 | Processing  | -> | Queue  | -> | Transmission |
 | read header |    | wait   |    | push bits    |
 +-------------+    +--------+    +--------------+
                                         |
                                         | Propagation
                                         | (signal crosses the cable)
                                         v
                                  Packet reaches Router B
```

Analogy: a line of cars passes through a toll booth, then drives to the next one.

1. **Processing delay:** the booth officer reads your ticket. The router reads the header, checks for errors and looks up where to send the packet. In modern hardware this is usually microseconds (1 microsecond is 1/1000 of a millisecond). Firewalls that inspect content can add more.
2. **Queuing delay:** you wait in line behind other cars. The packet waits in a memory buffer until the outgoing link is free. This is the only delay that changes a lot from moment to moment. It can be zero on a quiet link and hundreds of ms on a busy one.
3. **Transmission delay:** the whole line of cars passes through the booth, one by one. This is the time to push all the bits of the packet onto the wire. It is also called **serialization delay**.
4. **Propagation delay:** the drive to the next booth. This is the time for the signal to travel the length of the cable.

Where the analogy stops: a car leaves the booth and drives on its own, but a packet is a stream of bits, and its front edge can reach the next router before its tail has left the first one.

The two formulas you must know:

```text
Transmission delay = packet size in bits / link rate in bits per second
Propagation delay  = distance / signal speed
```

**Transmission delay** for a 1500-byte packet (12,000 bits):

| Link rate | Time |
|---|---|
| 10 Mbps | 1.2 ms |
| 100 Mbps | 120 microseconds |
| 1 Gbps | 12 microseconds |
| 10 Gbps | 1.2 microseconds |

**Propagation delay** depends on the medium. Signals in fibre travel at about 200,000 km/s, roughly two thirds of the speed of light in a vacuum. Copper is in the same range. Radio through air and space travels at nearly 300,000 km/s.

Handy rule: in fibre, each 1000 km adds about **5 ms one way, or 10 ms RTT**.

New York to London is about 5,600 km in a straight line, so the best case is about 56 ms RTT. Real paths are longer and pass through routers, which is why you see 70-80 ms.

Bandwidth affects transmission delay. Distance affects propagation delay. Upgrading a link cannot change distance.

Geostationary satellites orbit about 35,786 km above the equator. Up and down is about 71,600 km, so one way takes about 240 ms and the RTT is at least 480 ms. That is pure physics. See [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).

### Simplex, half duplex and full duplex

**Duplex** describes whether data can flow in both directions, and whether it can do so at the same time.

```text
Simplex       A ------------------> B     One direction only

Half duplex   A ------------------> B     Both directions,
              A <------------------ B     but one at a time

Full duplex   A ==================> B     Both directions
              A <================== B     at the same time
```

| Mode | Analogy | Network example |
|---|---|---|
| Simplex | One-way street, or FM radio | Broadcast TV, a single fibre used one way |
| Half duplex | Walkie-talkie: say "over", then listen | Ethernet with a hub, Wi-Fi on a channel |
| Full duplex | Phone call: both talk at once | Ethernet between a switch and a PC |

Half-duplex Ethernet devices share the medium. If two talk at once, a **collision** happens, so they use a method called CSMA/CD to detect it and try again. See [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).

A modern switch port is full duplex. Each direction has its own path, so collisions cannot happen. A 1 Gbps full-duplex port carries 1 Gbps each way at the same time. Marketing sometimes calls that "2 Gbps", but no single flow can go faster than 1 Gbps. Ethernet at 10 Gbps and above is full duplex only.

Fuller truth: Wi-Fi is half duplex because one radio cannot normally send and listen on the same channel at once. Features like MU-MIMO and OFDMA let an access point serve several clients at the same time, but they do not make a channel full duplex. See [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md).

Both ends of a link must agree on speed and duplex. They normally do this with **auto-negotiation**, a short exchange when the link comes up. A **duplex mismatch** happens when one side uses full and the other half. The link works, but slowly and with errors. It often comes from setting one end by hand. Cisco IOS, Linux and Windows can all show the current state:

```text
! Cisco IOS
Switch# show interfaces GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up
  ...
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
  ...
  Full-duplex, 1000Mb/s, media type is 10/100/1000BaseTX
```

```bash
# Linux (output shortened)
$ ethtool eth0 | grep -E 'Speed|Duplex'
        Speed: 1000Mb/s
        Duplex: Full
```

```powershell
# Windows PowerShell
PS> Get-NetAdapter | Format-Table Name, LinkSpeed, FullDuplex

Name     LinkSpeed FullDuplex
----     --------- ----------
Ethernet 1 Gbps          True
```

### Bandwidth-delay product

The **bandwidth-delay product (BDP)** tells you how much data can be "on the way" at one moment.

```text
BDP = bandwidth x RTT
```

Analogy: picture a long pipe filled with water. Its width is bandwidth and its length is delay. BDP is the volume of the pipe. Where the analogy stops: the water does not need a reply, but TCP does, which is why we use RTT and not one-way delay.

TCP sends data, then waits for an **ACK (acknowledgement)**, a short "I got it" message from the receiver. It may only have a limited amount of unacknowledged data out at once. That limit is called the **window**.

```text
Sender                                              Receiver
  | ==> data ==> data ==> data ==> data ==> data ==>  |
  | <-- ACK <-- ACK <-- ACK <-- ACK <-- ACK <-- ACK   |

Data sent but not yet acknowledged can be up to: bandwidth x RTT
```

Example: 100 Mbps with 50 ms RTT.

```text
100,000,000 bits/s x 0.050 s = 5,000,000 bits
5,000,000 / 8                = 625,000 bytes
```

To use the link fully, the sender must be able to have 625 KB out at once. If the window is smaller, the sender stops and waits, and some of the link goes unused.

| Bandwidth | RTT | BDP |
|---|---|---|
| 100 Mbps | 50 ms | 625 KB |
| 1 Gbps | 10 ms | 1.25 MB |
| 1 Gbps | 100 ms | 12.5 MB |
| 10 Gbps | 100 ms | 125 MB |

This is why a fast long-distance link can feel slow. The original TCP window field allows at most 65,535 bytes. On a 100 ms path, that caps TCP at about 5.2 Mbps, however big the link is. The **window scaling** option (RFC 7323) removes this limit, and all modern systems use it. Details are in [TCP Reliability and Flow Control](../06-transport-layer/05-tcp-reliability-and-flow-control.md).

### Download time calculation

```text
Time in seconds = file size in bits / speed in bits per second
```

Steps:

1. Convert the file size to bits (multiply bytes by 8).
2. Make sure the speed uses the same prefix.
3. Divide.
4. Add about 5-10% for overhead if you want a real-world figure.

## Example

### Example 1: Download time, ideal and real

You have a 200 Mbps plan and you download a 5 GB file.

Ideal case:

```text
5 GB = 5,000,000,000 bytes x 8 = 40,000,000,000 bits
40,000,000,000 / 200,000,000 = 200 seconds = 3 min 20 s
```

Real case, using about 94% goodput:

```text
200 Mbps x 0.94 = 188 Mbps
40,000,000,000 / 188,000,000 = about 213 seconds = 3 min 33 s
```

The browser would show about 23.5 MB/s, because 188 / 8 = 23.5. Windows File Explorer would list this file as about 4.66 GB, because it divides by 1,024 three times. Same file, different label.

Fuller truth: TCP starts slowly and speeds up, a server may limit you, and other people share the path. See [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).

### Example 2: Total delay of one packet

A laptop sends a 1500-byte packet (12,000 bits) through one router to a server. The processing and queue times below are assumed values for this example.

```text
Laptop --100 Mbps, 100 m--> Router --1 Gbps, 2000 km--> Server
         (copper)                    (fibre)
```

| Step | Calculation | Time (microseconds) |
|---|---|---|
| Transmission, link 1 | 12,000 / 100,000,000 s | 120 |
| Propagation, link 1 | 100 m / 200,000,000 m/s | 0.5 |
| Router processing | assumed | 20 |
| Router queue | assumed | 500 |
| Transmission, link 2 | 12,000 / 1,000,000,000 s | 12 |
| Propagation, link 2 | 2,000 km / 200,000 km/s | 10,000 |
| **Total** | | **10,652.5** (about 10.65 ms) |

Propagation is about 94% of the total. Doubling both link speeds would save only 66 microseconds. Moving the server closer would save far more.

Transmission counts on each link because a router normally uses **store and forward**: it receives the whole packet before it sends it on.

### Example 3: Which link is better?

Link A is 100 Mbps with 10 ms one-way latency. Link B is 10 Mbps with 1 ms one-way latency. Total time is transmission plus latency.

| Message | Link A | Link B | Winner |
|---|---|---|---|
| 1 KB (8,000 bits) | 0.08 + 10 = 10.08 ms | 0.8 + 1 = 1.8 ms | B |
| 1 GB (8,000,000,000 bits) | 80 s + 10 ms | 800 s + 1 ms | A |

Small messages care about latency. Large files care about bandwidth. Web pages make many small requests, which is why low latency makes browsing feel fast.

### Example 4: Measuring with real tools

Ping shows RTT, loss and a jitter hint.

```bash
# Linux
$ ping -c 5 203.0.113.10
PING 203.0.113.10 (203.0.113.10) 56(84) bytes of data.
64 bytes from 203.0.113.10: icmp_seq=1 ttl=54 time=21.3 ms
64 bytes from 203.0.113.10: icmp_seq=2 ttl=54 time=20.8 ms
64 bytes from 203.0.113.10: icmp_seq=3 ttl=54 time=24.1 ms
64 bytes from 203.0.113.10: icmp_seq=4 ttl=54 time=21.0 ms
64 bytes from 203.0.113.10: icmp_seq=5 ttl=54 time=23.8 ms

--- 203.0.113.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 20.800/22.200/24.100/1.441 ms
```

```powershell
# Windows
PS> ping -n 5 203.0.113.10

Pinging 203.0.113.10 with 32 bytes of data:
Reply from 203.0.113.10: bytes=32 time=21ms TTL=54
Reply from 203.0.113.10: bytes=32 time=20ms TTL=54
Reply from 203.0.113.10: bytes=32 time=24ms TTL=54
Reply from 203.0.113.10: bytes=32 time=21ms TTL=54
Reply from 203.0.113.10: bytes=32 time=24ms TTL=54

Ping statistics for 203.0.113.10:
    Packets: Sent = 5, Received = 5, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 20ms, Maximum = 24ms, Average = 22ms
```

Read it like this: RTT is about 22 ms, loss is 0%, and the `mdev` value of 1.4 ms (a measure of spread) means jitter is low. More on reading this output is in [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

`iperf3` measures goodput between two machines. Start `iperf3 -s` on the server (192.168.1.10), then run this on the client. The same command works on Windows if you install iperf3.

```bash
# Linux client on a 1 Gbps LAN (output shortened)
$ iperf3 -c 192.168.1.10 -t 10
Connecting to host 192.168.1.10, port 5201
[  5] local 192.168.1.20 port 49822 connected to 192.168.1.10 port 5201
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec    0       sender
[  5]   0.00-10.00  sec  1.09 GBytes   938 Mbits/sec            receiver
```

The 941 Mbps matches our calculation of 94.1%. `Retr` is the count of TCP retransmissions: 0 means no loss. Watch the units: iperf3 shows transfer in binary GBytes but bitrate in decimal Mbits.

## Going Deeper

**Queuing grows fast near full load.** A simple queuing model says average waiting time is `U / (1 - U)` times the time to send one packet, where U is how busy the link is. At 50% busy the factor is 1, at 80% it is 4, at 90% it is 9, and at 99% it is 99. Real traffic is burstier than this model, but the shape is right: delay jumps as a link nears full.

**Bufferbloat** is the name for very long delay caused by oversized buffers. A home router may hold seconds of data in its queue during a big upload, so ping goes from 20 ms to 500 ms or more, with no packet loss at all. Smart queue methods such as fq_codel and CAKE keep queues short. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

**Loss limits TCP speed.** The Mathis formula gives a rough limit for classic loss-based TCP: rate is at most `(MSS / RTT) x (1.22 / sqrt(loss))`. MSS is the largest TCP payload per packet. With MSS 1460 bytes, RTT 100 ms and 0.1% loss, the limit is about 4.5 Mbps, even on a 10 Gbps link. Newer algorithms like CUBIC and BBR behave differently, but the lesson stands: on long paths, small loss causes a big slowdown.

**Bits per second is not packets per second.** A minimum Ethernet frame is 64 bytes, and with preamble and gap it takes 84 bytes (672 bits) on the wire. So 1 Gbps carries at most 1,000,000,000 / 672 = about 1.488 million packets per second. Routers and firewalls are often limited by packet rate, not bit rate, so small packets can overload a device long before the link looks full.

## Common Mistakes

- Treating bandwidth and latency as one thing called "speed". A 1 Gbps link can have worse latency than a 100 Mbps link.
- Reading Mbps as MBps. A 100 Mbps plan gives about 12 MB/s, not 100 MB/s.
- Forgetting to multiply file size by 8 before you divide.
- Expecting a 1 Gbps link to give 1000 Mbps of file transfer. Headers take about 5-6%.
- Treating an ISP plan as the path speed. The bottleneck may be your Wi-Fi, your disk, or the server.
- Thinking ping shows one-way latency. It shows RTT.
- Mixing up transmission delay and propagation delay. Transmission depends on packet size and link rate. Propagation depends on distance only.
- Believing full duplex 1 Gbps means one flow can reach 2 Gbps. It means 1 Gbps each way.
- Thinking the Cisco `bandwidth` interface command changes the real link speed. It only sets a value used for things like routing metrics.
- Forgetting that home plans are often asymmetric, such as 500 Mbps down and 50 Mbps up. Uploads and video calls use the smaller number.
- Trusting ping alone. Some routers rate-limit ICMP (the protocol ping uses), so a device can show loss in ping while forwarding real traffic perfectly.

## Interview Questions

**Q: What is the difference between bandwidth, throughput and goodput?**
A: Bandwidth is the maximum rate a link can carry. Throughput is the rate actually delivered, measured over time. Goodput is the useful application data only, without headers and retransmissions. So goodput is at or below throughput, and throughput is at or below bandwidth.

**Q: What is the difference between latency and RTT?**
A: Latency usually means one-way delay from sender to receiver. RTT is the time to go there and come back. Ping measures RTT, and the two directions are not always equal.

**Q: What is jitter, and which applications care about it?**
A: Jitter is the variation in delay between packets. Voice, video calls and online games care most, because they need a steady flow. They use a jitter buffer to smooth it out, which adds a small delay.

**Q: How long does a 500 MB file take on a 20 Mbps link?**
A: 500 MB times 8 is 4,000 megabits. Divide by 20 and you get 200 seconds. In practice I would add 5 to 10 percent for overhead, so about 3 and a half minutes.

**Q: What are the four types of network delay?**
A: Processing is the router reading the header and deciding where to send the packet. Queuing is waiting in a buffer. Transmission is pushing the bits onto the link, which is packet size divided by link rate. Propagation is the signal traveling the cable, which is distance divided by signal speed.

**Q: What is the difference between transmission and propagation delay?**
A: Transmission delay is how long it takes to put the packet on the wire, so it depends on size and link rate. Propagation delay is how long one bit takes to reach the far end, so it depends on distance and the medium.

**Q: Will upgrading from 100 Mbps to 1 Gbps reduce latency?**
A: Only a little. It reduces transmission delay and can reduce queuing, but propagation stays the same. If the server is far away, most of the latency is distance, so a CDN or a closer server helps more than bandwidth.

**Q: Why does a 1 Gbps link show about 940 Mbps in a speed test?**
A: Each packet carries Ethernet, IP and TCP headers, plus the preamble and the inter-frame gap. A packet with 1448 bytes of data takes 1538 bytes on the wire, which is about 94 percent. So the link is working correctly.

**Q: What is the difference between half and full duplex, and what is a duplex mismatch?**
A: Half duplex can send both ways but only one way at a time. Full duplex does both at once. In a mismatch, one end uses half and the other full. The link stays up but is slow. The half-duplex side typically logs late collisions, and the full-duplex side logs CRC errors and runts.

**Q: What is the bandwidth-delay product, and why does it matter?**
A: It is bandwidth times RTT, and it tells you how much data can be in flight. For 1 Gbps and 40 ms, that is 40 million bits, or 5 MB. If the TCP window is smaller than this, the sender has to wait, and it cannot fill the link.

**Q: A server has a 1 Gbps link, but a user downloads at 50 Mbps. What would you check?**
A: I would find the bottleneck first. I would check the path with iperf3, then look at RTT and loss, then the TCP window against the BDP, then interface errors, speed and duplex. A bad cable can drop a link to 100 Mbps. It could also be Wi-Fi, disk speed, or a rate limit on the server.

**Q: Ping is 20 ms when idle but 400 ms during a large upload. Why?**
A: That is queuing delay, usually called bufferbloat. The upload fills a buffer at the slowest link, so the ping packets wait in line behind it. Propagation has not changed. Smart queue management or shaping just below the link rate fixes it.

## Quick Recap

- Bandwidth is the maximum rate. Throughput is what is delivered. Goodput is the useful part.
- Latency is delay, RTT is there and back, jitter is variation in delay, and loss is packets that never arrive.
- 1 byte is 8 bits. Divide Mbps by 8 to get MB/s, and multiply file size by 8 before you divide by link speed.
- Four delays: processing, queuing, transmission (size / rate) and propagation (distance / speed). In fibre, each 1000 km adds about 10 ms RTT.
- Simplex is one way, half duplex is both ways in turns, and full duplex is both ways at once.
- BDP = bandwidth x RTT. It is the amount of data in flight, and TCP's window must be at least this big to fill the link.
- More bandwidth helps large transfers. Lower latency helps small, chatty traffic. Find the bottleneck before you upgrade.
