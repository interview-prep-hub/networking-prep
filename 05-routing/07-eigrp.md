# EIGRP

**Level:** Advanced | **Prerequisites:** [The Routing Table](02-routing-table.md), [Dynamic Routing Overview](04-dynamic-routing-overview.md), [RIP](05-rip.md), [OSPF](06-ospf.md)

## In One Line

EIGRP (Enhanced Interior Gateway Routing Protocol) is a fast routing protocol from Cisco that keeps a backup path ready, and it uses an algorithm called DUAL to prove that the backup cannot create a loop.

## Why It Matters

Routers need to find a new path quickly when a link fails. Old distance vector protocols like RIP are slow here. RIP sends its full table every 30 seconds, and a bad route can bounce between routers for minutes.

EIGRP solves this in three ways:

- It sends updates only when something changes, and only to routers that need them.
- It stores a checked backup path before any failure happens. When the main path dies, the switch to the backup takes very little time.
- It never uses a path that could form a loop, even for a moment.

You will still find EIGRP in many Cisco enterprise networks, especially in campus networks and in WANs (wide area networks, the links that join distant sites). Many new designs choose OSPF or BGP because they run on every vendor's equipment.

Interviewers like EIGRP for a simple reason. The questions test real understanding. Anyone can learn "feasible successor" as a phrase. A strong candidate can look at numbers and say which path is a backup, and why. They will also ask about metric calculation, K values, stuck in active, and unequal cost load balancing, because no other common interior protocol has that last feature.

## Simple Explanation

Imagine you want to drive to the airport. You are 30 km away by your best road, so 30 is your personal best distance. You have three friends who live nearby. You call each one and ask, "How far are you from the airport?"

- Asha says, "20 km." Your best road goes through her town.
- Bilal says, "25 km." He is closer to the airport than you are.
- Chen says, "35 km." He is farther from the airport than you are.

Now Asha's road closes. Can you trust Bilal? Yes. He is closer to the airport than you are, so his route cannot possibly run back through your house. If it did, he would be farther away than you. You can use Bilal at once, with no more phone calls.

Can you trust Chen? Maybe, but you cannot be sure. His route might go through you. He might have said 35 because he was planning to drive to your house first (30 km to the airport, plus 5 km to you). So before you use him, you must phone around and check.

That is the heart of EIGRP:

| Story | EIGRP term | Meaning |
|---|---|---|
| Your best distance, 30 km | Feasible distance (FD) | The best metric this router has to the destination |
| What a friend tells you, such as 25 km | Reported distance (RD) | The metric the neighbour says it has |
| Asha, who is on your best road | Successor | The neighbour on the best path |
| Bilal, who is closer than you | Feasible successor (FS) | A safe backup neighbour |
| "Is this friend closer than I am?" | Feasibility condition (FC) | The rule that decides if a backup is safe |
| Phoning around | Query | A message asking neighbours for a new path |

**Where the analogy stops being true:** a metric is not a distance in km. It is a number made from link speed and delay. Also, real friends can be wrong, but EIGRP neighbours work from exact numbers that each router calculates. And Chen may be perfectly safe. The rule only rejects paths it cannot prove safe.

## How It Works

### 1. What "advanced distance vector" means

A **distance vector** protocol tells neighbours "I can reach this network, at this distance, in this direction." A router only knows what its neighbours tell it. It has no full map of the network.

A **link-state** protocol such as OSPF is different. Each router gets a full map and works out paths for itself.

EIGRP is a distance vector protocol because it has no map. It trusts what neighbours report. It is called **advanced** because it fixes the classic weaknesses:

| Classic distance vector (RIP) | EIGRP |
|---|---|
| Sends the full table on a timer | Sends only changes, only when they happen |
| Loops are possible during a change | DUAL keeps paths loop-free at all times |
| Metric is a simple hop count | Metric uses bandwidth and delay |
| No stored backup path | Keeps feasible successors ready |
| Neighbours are not tracked | Keeps a neighbour table, using Hello packets |

Older Cisco material calls EIGRP a "hybrid" protocol. That word is marketing. It does not use link-state flooding. "Advanced distance vector" is the correct phrase.

### 2. The three tables

EIGRP keeps three tables. Learn these first, because every other idea uses them.

| Table | What it holds | Show command (Cisco IOS) |
|---|---|---|
| Neighbour table | Routers directly connected that speak EIGRP | `show ip eigrp neighbors` |
| Topology table | Every path learned from neighbours, with FD and RD | `show ip eigrp topology` |
| Routing table | Only the best paths, used to forward packets | `show ip route eigrp` |

```text
  Hello packets        Update packets         DUAL chooses
       |                     |                     |
       v                     v                     v
 +-----------+        +--------------+      +---------------+
 | Neighbour | -----> |   Topology   | ---> | Routing table |
 |   table   |        |    table     |      |  (RIB)        |
 +-----------+        +--------------+      +---------------+
```

RIB means routing information base. It is another name for the routing table.

Despite its name, the topology table is not a map of the network. It is a list of what each neighbour reported.

### 3. Forming neighbours

EIGRP sends **Hello** packets out of each enabled interface. A Hello says "I am here." The packets go to the multicast address 224.0.0.10 (FF02::A for IPv6). Multicast means one packet reaches a group of listeners. EIGRP runs directly on IP, as **IP protocol 88**. It does not use TCP or UDP.

Default timers:

| Link type | Hello interval | Hold time |
|---|---|---|
| LAN, point-to-point links, multipoint faster than T1 | 5 seconds | 15 seconds |
| Multipoint links at T1 speed (1.544 Mbps) or slower | 60 seconds | 180 seconds |

The **hold time** is how long a router waits without hearing a neighbour before it declares that neighbour dead. It is three times the Hello interval by default.

Two routers become neighbours only if these match:

1. Same autonomous system (AS) number. In EIGRP, the AS number is just a label for one EIGRP domain.
2. Same K values (explained below).
3. Same subnet, based on the primary IP address of each interface.
4. Same authentication settings, if authentication is used.
5. The interface is not passive, and no filter blocks IP protocol 88.

Hello and hold timers do **not** need to match. Interface MTU also does not need to match, which is different from OSPF.

Once two routers accept each other, they exchange routes:

```text
 R1 (10.0.12.1)                          R2 (10.0.12.2)
   |                                        |
   |------- Hello to 224.0.0.10 ----------->|
   |<------ Hello to 224.0.0.10 ------------|
   |                                        |
   |<------ Update, INIT flag, unicast -----|
   |------- Ack, and own INIT Update ------>|
   |<------ Ack ----------------------------|
   |                                        |
   |<------ Updates: all best routes -------|
   |------- Updates: all best routes ------>|
   |<------ Acks in both directions ------->|
   |                                        |
   DUAL runs on both routers, routes install
```

After that first exchange, they send only Hellos, plus Updates when something changes.

### 4. Reliable delivery with RTP

IP does not promise delivery, and EIGRP does not use TCP. So it has its own small reliability layer called **RTP** (Reliable Transport Protocol). This has nothing to do with the RTP that carries voice and video.

Each reliable packet has a sequence number. The receiver must send an acknowledgement. If the ACK does not arrive, the sender sends the packet again, this time as unicast to that one router.

| Packet | Purpose | Reliable? | Normally sent as |
|---|---|---|---|
| Hello | Find and keep neighbours | No | Multicast |
| ACK | Confirm a reliable packet | No | Unicast |
| Update | Carry route information | Yes | Multicast, or unicast to a new neighbour |
| Query | Ask for a new path | Yes | Multicast |
| Reply | Answer a Query | Yes | Unicast |
| SIA-Query | "Are you still working on my Query?" | Yes | Unicast |
| SIA-Reply | "Yes, still working" | Yes | Unicast |

An ACK is really a Hello packet with no data and with an acknowledgement number set.

If a neighbour does not acknowledge a packet after 16 retransmissions, the router resets that neighbour. The log shows "retry limit exceeded".

### 5. The composite metric

A **metric** is the number a routing protocol uses to compare paths. Lower is better. EIGRP's metric is called **composite** because it can combine several values:

| Value | Meaning | Used by default? |
|---|---|---|
| Bandwidth | The slowest link speed along the path | Yes |
| Delay | The sum of interface delays along the path | Yes |
| Load | How busy the link is, 1 to 255 | No |
| Reliability | How error-free the link is, 255 is perfect | No |

Updates also carry MTU (largest packet size) and hop count. Neither is part of the metric. MTU is only informational, and hop count is used to limit how far a route can travel (100 hops by default, 255 at most).

Updates carry these raw values, not a finished metric. Each router works out the metric for itself. This is one reason why all routers must use the same formula.

**K values** are five weights (K1 to K5) that decide which values count:

```text
metric = 256 * ( K1*BW + (K2*BW)/(256 - load) + K3*delay )

If K5 is not 0, also multiply by:  K5 / (reliability + K4)

BW    = 10,000,000 / (lowest bandwidth on the path, in kbps)
delay = (sum of delays on the path, in microseconds) / 10
```

Both divisions drop any fraction. The defaults are:

```text
K1 = 1 (bandwidth on)     K2 = 0 (load off)
K3 = 1 (delay on)         K4 = 0, K5 = 0 (reliability off)
```

With the defaults, the formula becomes very simple:

```text
metric = 256 * (BW + delay)
```

**A trap:** when K5 is 0, the reliability part is skipped. It does not multiply the metric by zero.

The number 256 comes from history. IGRP, the older Cisco protocol that EIGRP replaced, had a 24-bit metric. EIGRP uses 32 bits, so the old value was shifted up by 8 bits, which is the same as multiplying by 256.

Default values on Cisco IOS interfaces:

| Interface | Bandwidth (kbps) | Delay (microseconds) | Connected metric |
|---|---|---|---|
| Serial T1 | 1,544 | 20,000 | 2,169,856 |
| FastEthernet | 100,000 | 100 | 28,160 |
| GigabitEthernet | 1,000,000 | 10 | 2,816 |
| Loopback | 8,000,000 | 5,000 | not used here |

These are configured values, not measured values. Check yours with `show interfaces`, which shows lines like `BW 1000000 Kbit/sec, DLY 10 usec`. Tunnel interfaces deserve special care, because they often start with a very low default bandwidth that varies by IOS version.

You can change both values:

```text
! Cisco IOS
interface GigabitEthernet0/0
 bandwidth 100000     ! in kbps, does not change the real speed
 delay 100            ! in tens of microseconds, so 1000 usec
```

Neither command changes how fast the link really is. They only change what EIGRP calculates.

### 6. DUAL: the key terms

**DUAL** (Diffusing Update Algorithm) is the logic that chooses paths and keeps them loop-free. It was developed by J.J. Garcia-Luna-Aceves. The `D` code for EIGRP routes in the routing table comes from DUAL.

Four terms carry almost all the meaning.

**Reported distance (RD).** The metric a neighbour tells you it has to the destination. It is the neighbour's own best metric. Older Cisco documents call it **advertised distance (AD)**. Do not confuse that with administrative distance, which is also shortened to AD.

**Feasible distance (FD).** Your own best metric to the destination. It equals the neighbour's RD plus your cost to reach that neighbour.

**Successor.** The neighbour on the best path. This is the next hop that goes into the routing table. You can have more than one if paths tie.

**Feasible successor (FS).** A neighbour that is not the successor, but passes the safety rule. It waits in the topology table as a ready backup.

The safety rule is the **feasibility condition (FC)**:

```text
A path is a feasible successor only if:

    its RD  <  the current FD
```

The sign is strictly "less than". If RD equals FD, the path fails.

**Why it stops loops.** If a neighbour's distance to the destination is less than mine, its path cannot pass through me. If it did, its distance would be my distance plus something, so it would be larger than mine. A smaller number is proof.

**Worked example with simple numbers.** Router X has four neighbours. To keep the sums easy, these numbers just add up. Real metrics do not add this simply, because the slowest bandwidth is a minimum and not a sum.

| Neighbour | RD | Link cost | Total | Role |
|---|---|---|---|---|
| A | 20 | 10 | 30 | Successor (lowest total), so FD = 30 |
| B | 25 | 15 | 40 | RD 25 < FD 30, so feasible successor |
| C | 30 | 5 | 35 | RD 30 is not less than 30, so not FS |
| D | 45 | 2 | 47 | RD 45 is not less than 30, so not FS |

Look closely at C. Its total, 35, is better than B's total, 40. It still fails, because the test looks at **RD, not the total**. So the backup used by EIGRP is B, even though C has the lower total.

### 7. What happens when a path fails

A route in the topology table has one of two states:

- **Passive:** stable. DUAL is not working on this route. This is the good state.
- **Active:** the router has lost its path and is actively searching for another.

These names feel backwards. Passive is healthy. Active means there is a problem.

```text
   Successor lost, or its metric got worse
                     |
                     v
        Is there a feasible successor?
            |                     |
           yes                    no
            |                     |
            v                     v
   Use it straight away     Route goes ACTIVE
   Route stays PASSIVE      Send Query to all neighbours
   Send Update              Wait for ALL Replies
                                  |
                                  v
                            Choose the best path
                            Reset FD, go PASSIVE
                            Send Update
```

The first branch is called a **local computation**. No other router needs to help, so it is very fast. Typically it is well under a second, once the failure is detected.

The second branch is called a **diffusing computation**, because the question spreads outward through the network. When a router receives a Query, it does one of the following:

| Situation at the router that got the Query | What it does |
|---|---|
| It has no route to that destination | Replies "unreachable" at once |
| The Query came from a neighbour that is not its successor | Keeps its route and replies with its metric |
| The Query came from its successor, and it has an FS | Switches to the FS and replies with the new metric |
| The Query came from its successor, and it has no FS | Goes Active itself, and sends its own Queries |

In the last case the router cannot answer until its own neighbours answer. That is how a Query travels deep into the network, and why large networks can have trouble (see stuck in active below).

### 8. Load balancing

**Equal cost.** If two paths have the same metric, and both pass the FC, then both are successors and both go into the routing table. IOS installs up to 4 paths by default, set with `maximum-paths`. The upper limit depends on the platform.

**Unequal cost.** EIGRP can also use paths with different metrics, at the same time. OSPF and IS-IS do not do this natively. The `variance` command sets a multiplier. The default is 1, which means equal cost only.

A path is used for unequal cost load balancing only when **both** of these are true:

1. It is a feasible successor (RD is less than FD).
2. Its metric is within variance times the best metric.

```text
Best metric = 100, variance 2, so the limit is 200

Path   Metric   RD     Passes FC?   Within limit?   Used?
P1     100      80     successor    yes             yes
P2     150      90     yes          yes             yes
P3     180      120    no           yes             no
P4     250      90     yes          no              no
```

The sources differ on whether a metric exactly on the limit counts, so never design a network to sit on the edge.

Traffic is shared in proportion to the metric. A path with a metric 2 times worse gets about half as much traffic. IOS does this with the `traffic-share balanced` setting, which is the default. With normal CEF forwarding (fast hardware or software switching) the split is per flow, not per packet.

### 9. Basic configuration

This is the classic style on Cisco IOS.

```text
! Cisco IOS, classic mode
router eigrp 64512
 eigrp router-id 10.255.0.1
 passive-interface default
 no passive-interface GigabitEthernet0/0
 network 10.0.12.0 0.0.0.255
 network 10.1.0.0 0.0.0.255
```

What each line does:

- `router eigrp 64512` starts EIGRP. The number is the AS number, from 1 to 65535. It must match on neighbours. It has no link to BGP AS numbers or public registries.
- `eigrp router-id` sets a 32-bit name for this router. If you leave it out, IOS picks the highest loopback IP, or else the highest IP on an active interface.
- `passive-interface default` stops Hellos on all interfaces. This is a safe habit, so no stranger can become a neighbour on a user LAN.
- `no passive-interface` allows Hellos on the links that face other routers.
- `network` does **not** say "advertise this network" directly. It says "enable EIGRP on any interface with an IP inside this range". The subnet of that interface is then advertised. The second number is a wildcard mask, which is the reverse of a subnet mask.

Auto-summary, which shrinks routes down to classful borders, is off by default on modern IOS (15.0 and later). On older software you must enter `no auto-summary`.

Useful interface tuning:

```text
! Cisco IOS, classic mode
interface GigabitEthernet0/0
 ip hello-interval eigrp 64512 1
 ip hold-time eigrp 64512 3
```

Changing the Hello interval does not change the hold time for you. Change both.

## Example

### The lab

```text
                   1G             1G
           +------------- R2 -------------+
           |                              |
           |      100M            1G      |
   R1 -----+------------- R3 -------------+----- R4 ----- LAN
           |                              |    10.4.0.0/24
           |      100M           100M     |    (1G, Gi0/2)
           +------------- R5 -------------+
```

| Link | Subnet | Speed | Addresses |
|---|---|---|---|
| R1 Gi0/0 to R2 Gi0/0 | 10.0.12.0/24 | 1 Gbps | R1 .1, R2 .2 |
| R1 Fa0/0 to R3 Fa0/0 | 10.0.13.0/24 | 100 Mbps | R1 .1, R3 .3 |
| R1 Fa0/1 to R5 Fa0/0 | 10.0.15.0/24 | 100 Mbps | R1 .1, R5 .5 |
| R2 Gi0/1 to R4 Gi0/0 | 10.0.24.0/24 | 1 Gbps | R2 .2, R4 .4 |
| R3 Gi0/1 to R4 Gi0/1 | 10.0.34.0/24 | 1 Gbps | R3 .3, R4 .4 |
| R5 Fa0/1 to R4 Fa0/0 | 10.0.45.0/24 | 100 Mbps | R5 .5, R4 .4 |
| R4 Gi0/2 LAN | 10.4.0.0/24 | 1 Gbps | R4 .1 |

All interfaces use default bandwidth and delay. So a gigabit interface has a delay of 10 microseconds, and a FastEthernet interface has 100 microseconds. The goal is to find how R1 reaches 10.4.0.0/24.

### Configuration

```text
! Cisco IOS, R1
router eigrp 64512
 eigrp router-id 10.255.0.1
 network 10.0.12.0 0.0.0.255
 network 10.0.13.0 0.0.0.255
 network 10.0.15.0 0.0.0.255

! Cisco IOS, R4
router eigrp 64512
 eigrp router-id 10.255.0.4
 passive-interface GigabitEthernet0/2
 network 10.0.24.0 0.0.0.255
 network 10.0.34.0 0.0.0.255
 network 10.0.45.0 0.0.0.255
 network 10.4.0.0 0.0.0.255
```

R2, R3 and R5 follow the same pattern, with their own subnets and router IDs.

### Step 1: calculate each neighbour's RD

Each neighbour's RD is its own best metric to the LAN.

**R2:** the path is its link to R4 (1 Gbps, 10 us) plus the LAN (1 Gbps, 10 us).

```text
BW    = 10,000,000 / 1,000,000 = 10
delay = (10 + 10) / 10         = 2
RD    = 256 * (10 + 2)         = 3072
```

**R3:** the same shape as R2, because its link to R4 is also gigabit. RD = 3072.

**R5:** the link to R4 is 100 Mbps with delay 100 us, then the LAN.

```text
BW    = 10,000,000 / 100,000  = 100
delay = (100 + 10) / 10       = 11
RD    = 256 * (100 + 11)      = 28416
```

Note how the slowest link sets BW for the whole path, but delays add up.

### Step 2: calculate R1's total metric through each neighbour

```text
Via R2: slowest link 1 Gbps, delays 10 + 10 + 10 = 30 us
        256 * (10 + 3)    = 3328

Via R3: slowest link 100 Mbps, delays 100 + 10 + 10 = 120 us
        256 * (100 + 12)  = 28672

Via R5: slowest link 100 Mbps, delays 100 + 100 + 10 = 210 us
        256 * (100 + 21)  = 30976
```

### Step 3: apply the feasibility condition

| Neighbour | Total metric | RD | Test against FD 3328 | Role |
|---|---|---|---|---|
| R2 | 3328 | 3072 | Lowest total | Successor |
| R3 | 28672 | 3072 | 3072 < 3328, passes | Feasible successor |
| R5 | 30976 | 28416 | 28416 > 3328, fails | Not a backup |

The R5 path has no loop at all. EIGRP still cannot prove that from the numbers, so it does not treat R5 as a ready backup. This is normal.

### Step 4: confirm on the router

```text
R1# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(64512)
H  Address      Interface  Hold Uptime   SRTT  RTO  Q   Seq
                           (sec)         (ms)      Cnt  Num
2  10.0.15.5    Fa0/1        12 00:21:07    2  100  0   31
1  10.0.13.3    Fa0/0        14 00:21:44    1  100  0   27
0  10.0.12.2    Gi0/0        11 00:22:03    1  100  0   42
```

How to read the neighbour table:

| Column | Meaning |
|---|---|
| H | Handle, the order in which neighbours were learned |
| Hold | Seconds left before the neighbour is declared dead. It counts down and resets on each Hello |
| Uptime | How long the neighbour has been up. A low value on an old link means it keeps flapping |
| SRTT | Smooth round trip time, the average time for a packet and its ACK, in ms |
| RTO | Retransmission timeout, how long to wait before sending again, in ms |
| Q Cnt | Packets waiting to be sent. It should be 0. A stuck number shows a problem |
| Seq Num | The last sequence number received from this neighbour |

```text
R1# show ip eigrp topology
EIGRP-IPv4 Topology Table for AS(64512)/ID(10.255.0.1)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.4.0.0/24, 1 successors, FD is 3328
        via 10.0.12.2 (3328/3072), GigabitEthernet0/0
        via 10.0.13.3 (28672/3072), FastEthernet0/0
```

The first number in brackets is the total metric through that neighbour. The second is the RD. R5 is missing here, because this command shows only successors and feasible successors. To see all paths:

```text
R1# show ip eigrp topology all-links
P 10.4.0.0/24, 1 successors, FD is 3328
        via 10.0.12.2 (3328/3072), GigabitEthernet0/0
        via 10.0.13.3 (28672/3072), FastEthernet0/0
        via 10.0.15.5 (30976/28416), FastEthernet0/1
```

Output is trimmed here to the route we care about. The routing table holds only the winner:

```text
D     10.4.0.0/24 [90/3328] via 10.0.12.2, 00:21:53, GigabitEthernet0/0
```

Here 90 is the administrative distance and 3328 is the metric.

### Failure 1: R1 loses the link to R2 (a backup exists)

1. R1 Gi0/0 goes down. R1 drops R2 as a neighbour at once.
2. DUAL looks at the remaining paths. R3 has RD 3072, which is below FD 3328. R3 is a feasible successor.
3. R1 installs 10.0.13.3 as the next hop, with metric 28672. No Query is sent.
4. R1 sends an Update to tell its neighbours about the new metric.

```text
D     10.4.0.0/24 [90/28672] via 10.0.13.3, 00:00:03, FastEthernet0/0
```

The route never went Active. This is the fast case.

If the link stays up physically and R2 just dies, for example because a switch sits between them, R1 only notices when the 15 second hold time ends. So the real outage is detection time plus a very short switch. See [BFD](#faster-failure-detection) below.

### Failure 2: no feasible successor

Start again, but imagine that R1's link to R3 is shut down. R1 now has only R2 (3328/3072) and R5 (30976/28416). The link to R2 fails.

1. The only path left is R5. Its RD 28416 is not less than FD 3328. There is no FS.
2. The route goes Active. R1 sends a Query to R5.
3. R5's own successor is R4, not R1. R5 keeps its route and replies with 28416.
4. R1 has all its replies. It installs R5, metric 30976. FD is reset to 30976. The route returns to Passive.

```text
 R1                                  R5
  |   Query: path to 10.4.0.0/24?    |
  |--------------------------------->|
  |                                  |  Own path via R4 is fine
  |   Reply: metric 28416            |
  |<---------------------------------|
  |   Ack                            |
  |--------------------------------->|
  R1: install via R5, FD = 30976, Passive
```

The result is the same as with a feasible successor. It just took a round of messages, and the destination had no path for that short time.

### Adding unequal cost load balancing

Can R1 use both R2 and R3 at once? R3 is an FS, so it passes the first test. For the second test:

```text
Metric ratio = 28672 / 3328 = 8.6, so variance must be 9

variance 8:  8 * 3328 = 26624, less than 28672, R3 not used
variance 9:  9 * 3328 = 29952, more than 28672, R3 used
```

```text
! Cisco IOS, R1
router eigrp 64512
 variance 9
```

```text
R1# show ip route eigrp | begin 10.4.0.0
D     10.4.0.0/24 [90/3328] via 10.0.12.2, 00:00:08, GigabitEthernet0/0
                  [90/28672] via 10.0.13.3, 00:00:08, FastEthernet0/0
```

```text
R1# show ip route 10.4.0.0 255.255.255.0
Routing entry for 10.4.0.0/24
  Known via "eigrp 64512", distance 90, metric 3328, type internal
  Routing Descriptor Blocks:
  * 10.0.12.2, from 10.0.12.2, 00:00:08 ago, via GigabitEthernet0/0
      Route metric is 3328, traffic share count is 240
      Total delay is 30 microseconds, minimum bandwidth is 1000000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 2
    10.0.13.3, from 10.0.13.3, 00:00:08 ago, via FastEthernet0/0
      Route metric is 28672, traffic share count is 28
      Total delay is 120 microseconds, minimum bandwidth is 100000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 2
```

The share counts show a split of about 8.6 to 1, which matches the metric ratio. R5 will never join this group, even with a very high variance, because it is not a feasible successor.

### Seeing EIGRP on the wire

EIGRP runs on routers, not on normal hosts, so there is no Linux or Windows client to test with. But you can watch it from a capture point in a lab.

```bash
# Linux: capture EIGRP on a mirrored port or lab bridge
sudo tcpdump -ni eth0 'ip proto 88'
```

```text
10:15:02.118204 IP 10.0.12.1 > 224.0.0.10: EIGRP Hello, length: 40
10:15:03.402771 IP 10.0.12.2 > 224.0.0.10: EIGRP Hello, length: 40
```

On Windows, use Wireshark with the display filter `eigrp`. More detail is in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) and [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

## Going Deeper

### The EIGRP packet header

Every EIGRP packet sits inside IP, with protocol 88, and starts with a 20-byte header. After it come TLVs (Type, Length, Value), which are small labelled blocks of data.

| Offset (bytes) | Field | Size | Notes |
|---|---|---|---|
| 0 | Version | 8 bits | Header version, value 2 |
| 1 | Opcode | 8 bits | 1 Update, 3 Query, 4 Reply, 5 Hello, 10 SIA-Query, 11 SIA-Reply |
| 2 | Checksum | 16 bits | Covers the EIGRP packet |
| 4 | Flags | 32 bits | Includes the INIT flag |
| 8 | Sequence | 32 bits | Used by RTP |
| 12 | Acknowledgement | 32 bits | Non-zero when acknowledging |
| 16 | Virtual Router ID | 16 bits | Separates EIGRP instances |
| 18 | AS number | 16 bits | Must match the receiver |
| 20 | TLVs | Variable | Parameters, routes, authentication and more |

The Hello's Parameter TLV carries the K values and the hold time.

### What FD really tracks

The simple definition, "FD is the current best metric", is true when the network is calm. The fuller truth is that FD is the **lowest metric known since the route last went from Active to Passive**. During a local computation, FD can stay the same or go down, but it does not go up.

In Failure 1, R1 moved to metric 28672 without going Active. By the DUAL rule, the stored FD stays at 3328, and the output can look like this:

```text
P 10.4.0.0/24, 1 successors, FD is 3328
        via 10.0.13.3 (28672/3072), FastEthernet0/0
```

So R5 (RD 28416) is still not a feasible successor. If the R3 path also fails, R1 must go Active. FD is then reset to the new best metric when the route returns to Passive. Many textbooks skip this and show FD jumping to the new metric at once, so do not be alarmed if a study guide and a router disagree.

### The feasibility condition is careful, not perfect

The FC is **sufficient** but not **necessary**. That means every path that passes is safe, but some safe paths fail. R5 in our lab is one. EIGRP accepts this. The cost is a Query in rare cases. The gain is a simple rule that can never create a loop.

### Stuck in active (SIA)

When a router goes Active it starts a timer, 3 minutes by default. It must receive a Reply from every neighbour it sent a Query to. If one neighbour stays silent, the route is **stuck in active**.

Why it happens:

- The Query travelled very far, and a distant link is slow.
- A router along the way has high CPU or little memory.
- A link loses packets in one direction, so Replies are lost.
- The network is too big, with no boundaries on Queries.

```text
 R1 --Query--> R2 --Query--> R3 --Query--> R4 --Query--> R5 (slow)
 R1 waits      R2 waits      R3 waits      R4 waits      ...

 No router can answer until the router after it answers.
```

Modern IOS reacts in a calmer way than older versions. At the halfway point of the timer (90 seconds), it sends a **SIA-Query** to the silent neighbour. If a **SIA-Reply** comes back, the neighbour is alive and working, so the neighbour stays up and the timer gets more time. After up to three of these rounds, or if no SIA-Reply arrives, the router resets that neighbour.

Older IOS simply reset the neighbour when the timer ended, which removed every route through that neighbour. The log line looks like this:

```text
%DUAL-3-SIA: Route 10.4.0.0/24 stuck-in-active state
in IP-EIGRP(0) 64512.
```

To see a route that is currently Active:

```text
R1# show ip eigrp topology active
A 10.4.0.0/24, 0 successors, FD is Inaccessible, Q
    1 replies, active 00:00:42, query-origin: Local origin
    Remaining replies:
         via 10.0.15.5, r, FastEthernet0/1
```

The small `r` means R1 is still waiting for a Reply from that neighbour. That is the router to investigate next.

The fix is to shorten the path of Queries, using the two tools below. Do not treat a longer active timer as a real fix.

### Stub routers

A **stub** router is a router that should never be used as a path to somewhere else, such as a branch office with one or two uplinks.

```text
! Cisco IOS, on the branch router
router eigrp 64512
 eigrp stub connected summary
```

The stub tells its neighbours this fact in its Hello. Neighbours then do **not** send it Queries. This has two benefits:

- Branch routers do not get caught in Queries, so a slow WAN link cannot cause SIA.
- The hub routers do not wait on hundreds of branch replies.

`eigrp stub` by itself means `connected summary`. Other keywords include `static`, `redistributed` and `receive-only`.

### Summarisation

A summary replaces many small routes with one bigger route. See [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md). EIGRP lets you summarise on **any** interface, on any router. OSPF allows this only at area borders.

```text
! Cisco IOS, classic mode
interface GigabitEthernet0/0
 ip summary-address eigrp 64512 10.4.0.0 255.255.0.0
```

The router also creates a discard route, so it drops packets for parts of the summary that it does not have:

```text
D     10.4.0.0/16 is a summary, 00:01:20, Null0
```

This discard route has administrative distance 5.

Summaries also limit Queries. A router that receives a Query for 10.4.0.0/24, when it only knows the summary 10.4.0.0/16, has no exact match. It replies "unreachable" at once and does not pass the Query on. So the failure stays inside the summarised region.

### Named mode and wide metrics

Classic metrics have a problem with fast links. BW is 10,000,000 divided by the speed in kbps, with the fraction dropped.

```text
 10 Gbps: 10,000,000 / 10,000,000  = 1
 40 Gbps: 10,000,000 / 40,000,000  = 0 (fraction dropped)
100 Gbps: 10,000,000 / 100,000,000 = 0 (fraction dropped)
```

So classic metrics cannot tell 40 Gbps from 100 Gbps. **Named mode** fixes this with **wide metrics**, which are 64 bits and scale values by 65536. All the configuration for an address family sits in one place.

```text
! Cisco IOS, named mode
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 64512
  network 10.0.12.0 0.0.0.255
  eigrp router-id 10.255.0.1
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/0
   no passive-interface
  exit-af-interface
  topology base
   variance 9
  exit-af-topology
 exit-address-family
```

The name CAMPUS is only local. The number in `autonomous-system` is what neighbours must agree on.

Example, one gigabit interface with a delay of 10 microseconds:

```text
Throughput = 65536 * 10,000,000 / 1,000,000     = 655360
Latency    = 65536 * 10,000,000 ps / 1,000,000  = 655360
Composite  = 655360 + 655360                    = 1310720
```

The routing table in IOS only holds a 32-bit metric. So the wide metric is divided by `rib-scale`, which is 128 by default. Here, 1310720 / 128 = 10240.

Wide metrics add **K6**, for extended attributes such as jitter and energy. It is 0 by default. Interface delay defaults can differ in named mode, depending on speed and platform, so read the values from your own router. Classic and named routers can peer with each other, and IOS scales the values, but a consistent design is easier to reason about.

The command `eigrp upgrade-cli` converts a classic configuration to named mode. Named mode has its own show commands, such as `show eigrp address-family ipv4 64512 neighbors`. The `show ip eigrp` commands also still work on most software.

### K value changes

```text
! Cisco IOS: metric weights TOS K1 K2 K3 K4 K5 (TOS must be 0)
router eigrp 64512
 metric weights 0 1 0 1 0 0
```

That line sets the defaults. Almost nobody should change it. Load and reliability change over time, so a metric built from them can shift and cause extra work and routing changes. Also, one router with different K values drops all its neighbours:

```text
%DUAL-5-NBRCHANGE: EIGRP-IPv4 64512: Neighbor 10.0.12.2
(GigabitEthernet0/0) is down: K-value mismatch
```

### Graceful shutdown

When EIGRP is removed from a router or an interface, the router sends a Hello with all K values set to 255. This is known as a **Goodbye** message. Neighbours drop the adjacency at once, without waiting for the hold time.

### Faster failure detection

Hold time is slow compared to a fast network. **BFD** (Bidirectional Forwarding Detection) is a small protocol that checks a link many times per second, and can tell EIGRP about a failure in under a second. On IOS, classic mode uses `bfd all-interfaces` under `router eigrp`, plus BFD timers on the interface.

**Graceful restart** (also called NSF, non-stop forwarding) lets a router restart its control software while forwarding continues, and its neighbours keep the routes for a short time.

### Router ID and external routes

EIGRP does not need a unique router ID for neighbours to form. Two routers with the same ID can still peer. But the ID is carried in external routes and used to stop loops. If a router receives an external route that shows its own ID as the source, it ignores it. So duplicate IDs can make external routes silently vanish.

### Authentication

Authentication proves that a packet came from a trusted neighbour. It does not encrypt anything. A wrong key stops the neighbour from forming.

```text
! Cisco IOS, classic mode: MD5
key chain EIGRP-KEYS
 key 1
  key-string Lab-Only-Key-1
interface GigabitEthernet0/0
 ip authentication mode eigrp 64512 md5
 ip authentication key-chain eigrp 64512 EIGRP-KEYS

! Cisco IOS, named mode: HMAC-SHA-256
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 64512
  af-interface GigabitEthernet0/0
   authentication mode hmac-sha-256 Lab-Only-Key-1
```

Prefer HMAC-SHA-256 where it is available, because MD5 is weak.

### EIGRP for IPv6

EIGRP also carries IPv6. Neighbours use link-local addresses, and Hellos go to FF02::A. There is no `network` command in the IPv6 classic style. You enable it per interface.

```text
! Cisco IOS, classic style
ipv6 unicast-routing
ipv6 router eigrp 64512
 eigrp router-id 10.255.0.1
 no shutdown
interface GigabitEthernet0/0
 ipv6 enable
 ipv6 eigrp 64512
```

The router ID is still a 32-bit number, so a router with no IPv4 address needs it set by hand. On some software the process starts shut down, which is why `no shutdown` appears above. In named mode, `address-family ipv6 unicast autonomous-system 64512` turns it on for all IPv6 interfaces.

### Redistribution and external routes

Routes brought in from another source are called **external**. They show as `D EX` in the routing table, with administrative distance 170. Internal routes have 90.

```text
! Cisco IOS, classic mode
router eigrp 64512
 redistribute static metric 100000 100 255 1 1500
```

The five numbers are bandwidth (kbps), delay (tens of microseconds), reliability, load and MTU. Full coverage is in [Advanced Routing Techniques](12-advanced-routing-techniques.md).

### Split horizon and hub-and-spoke WANs

**Split horizon** means a router does not advertise a route out of the interface it uses to reach that route. It is on by default, and it helps stop loops.

On a hub-and-spoke network, such as DMVPN, all spokes connect to one hub interface. The hub cannot pass one spoke's routes to another spoke, because they would go back out of the same interface. The fix on the hub is `no ip split-horizon eigrp 64512`. For more about WAN designs see [WAN Technologies](../10-enterprise-and-datacenter/08-wan-technologies.md).

### EIGRP compared with OSPF and RIP

| Feature | EIGRP | OSPF | RIPv2 |
|---|---|---|---|
| Type | Advanced distance vector | Link state | Distance vector |
| Algorithm | DUAL | Dijkstra SPF | Bellman-Ford |
| Metric | Bandwidth + delay | Cost | Hop count |
| Admin distance | 90 internal, 170 external | 110 | 120 |
| Carried in | IP protocol 88 | IP protocol 89 | UDP port 520 |
| Multicast | 224.0.0.10 | 224.0.0.5, 224.0.0.6 | 224.0.0.9 |
| Updates | Partial, on change | On change, with a slow refresh | Full table every 30 s |
| Unequal cost paths | Yes, `variance` | No | No |
| Hierarchy | None needed | Areas | None |
| Backup precomputed | Yes, FS | Not by default | No |
| Vendors | Mostly Cisco | All major vendors | All major vendors |

### Standards status

EIGRP was Cisco proprietary for many years. Cisco built it in the early 1990s. In 2016, Cisco published the basics as [RFC 7868](https://www.rfc-editor.org/rfc/rfc7868). It is an **Informational** RFC, not an Internet Standard. Open source support exists in FRRouting, but in practice EIGRP is a Cisco protocol, and multi-vendor networks choose OSPF, IS-IS or BGP.

## Common Mistakes

- **Testing the total metric instead of the RD.** The FC compares the neighbour's RD with your FD. A path's total metric does not matter for this test.
- **Accepting RD equal to FD.** The condition is strictly "less than". Equal fails.
- **Thinking the second-best path is always the backup.** A path can have the second-lowest total and still fail the FC.
- **Thinking `variance` alone adds paths.** Only feasible successors can be used, no matter how high variance goes.
- **Reading "Active" as good.** Active means the router has lost its path and is searching. Passive is the healthy state.
- **Adding bandwidth along the path.** Bandwidth is the slowest link. Only delay is summed.
- **Forgetting that K5 = 0 skips reliability.** It does not make the metric zero.
- **Using the `delay` value with the wrong unit.** `show interfaces` shows microseconds. The `delay` command takes tens of microseconds. The formula divides by 10.
- **Thinking `bandwidth` changes link speed.** It changes only the numbers that protocols calculate. It also limits how much bandwidth EIGRP traffic can use, which is 50 percent of the configured value by default.
- **Treating `network` as an advertisement command.** It selects interfaces. A wide statement can start EIGRP on interfaces you did not intend.
- **Blaming timers when neighbours do not form.** Hello and hold timers do not need to match. Check the AS number, K values, subnet, authentication, passive state and ACLs. The `show ip eigrp interfaces` command confirms which interfaces are really running EIGRP.
- **Mixing up AD meanings.** Advertised distance is another name for RD. Administrative distance is the 90 and 170 value used to compare routing sources.
- **Setting only the Hello interval.** The hold time stays as it was, so a low Hello with an old hold time gives you no faster detection.

## Interview Questions

**Q: Why is EIGRP called an advanced distance vector protocol?**

A: Routers do not have a map of the network. They rely on what neighbours report, so it is distance vector. It is advanced because it sends only partial updates on change, keeps neighbour relationships with Hellos, stores backup paths, and uses DUAL to stay loop-free at all times.

**Q: Define feasible distance, reported distance, successor and feasible successor.**

A: Reported distance is the metric a neighbour says it has to the destination. Feasible distance is my best metric to that destination. The successor is the neighbour on my best path. A feasible successor is another neighbour whose reported distance is lower than my feasible distance, so it is a safe backup.

**Q: What is the feasibility condition, and why does it prevent loops?**

A: The reported distance of a neighbour must be strictly less than my feasible distance. If the neighbour is closer to the destination than I am, its path cannot go through me, because then its distance would be greater than mine. So using that neighbour cannot create a loop.

**Q: My FD is 3328. A neighbour reports RD 3328 and a total of 3500. Is it a feasible successor?**

A: No. The RD must be strictly less than the FD, and here they are equal. The total of 3500 does not matter for this test. The path may be loop-free, but EIGRP cannot prove it, so it would need a Query before using this path.

**Q: How do you calculate the default EIGRP metric? Give an example.**

A: With default K values, the metric is 256 times the sum of BW and delay. BW is 10 million divided by the lowest bandwidth in kbps. Delay is the sum of delays in microseconds, divided by 10. For a path with a lowest bandwidth of 1 Gbps and 30 microseconds of delay, it is 256 times 10 plus 3, which is 3328.

**Q: What are K values, and what happens if they do not match?**

A: They are weights that decide what goes into the metric. K1 is bandwidth, K3 is delay, and K2, K4 and K5 relate to load and reliability. The default is K1 and K3 set to 1 and the others 0. If they do not match, routers refuse to be neighbours, because they would calculate different metrics from the same data.

**Q: How does unequal cost load balancing work?**

A: You set `variance`, which is a multiplier. A path is used if it is a feasible successor and its metric is within variance times the best metric. Traffic is shared in proportion to the metrics. For example, with best metric 3328 and a feasible successor at 28672, variance 9 is needed, because 8 times 3328 is too low and 9 times 3328 is 29952.

**Q: What does stuck in active mean, and how do you prevent it?**

A: A route went Active, and a neighbour has not replied to a Query within the active timer, which is 3 minutes by default. Modern IOS sends a SIA-Query at the halfway point and resets the neighbour only if it stops answering. The usual prevention is to limit how far Queries travel, with summarisation and stub routers, and to fix slow or lossy links.

**Q: What is the difference between Passive and Active states?**

A: Passive means the route is stable and no calculation is running. Active means the router lost its successor, has no feasible successor, and is sending Queries while it waits for all Replies. A healthy network has every route in Passive.

**Q: Two EIGRP routers will not become neighbours. What do you check?**

A: I check that the link works and that both IPs are in the same subnet. Then I confirm the AS number, K values and authentication keys match. I check for a passive interface, and that no ACL blocks protocol 88 or 224.0.0.10. Timers and MTU do not stop the adjacency. I use `show ip eigrp interfaces`, `show ip protocols` and the log.

**Q: What is an EIGRP stub router, and why use one?**

A: A stub is a router that should not be used as a transit path, such as a branch site. It tells neighbours it is a stub in its Hello, so they do not send it Queries. That keeps Queries off slow WAN links, reduces the risk of stuck in active, and helps the network scale.

**Q: When would you choose EIGRP over OSPF?**

A: EIGRP suits an all-Cisco network where you want simple design with no areas, summarisation anywhere, and unequal cost load balancing. OSPF is the better choice for multi-vendor networks, because it is an open standard. In a new design, vendor freedom usually decides in favour of OSPF.

## Quick Recap

- EIGRP is an advanced distance vector protocol. It uses IP protocol 88, multicast 224.0.0.10, and administrative distance 90 internal and 170 external.
- It keeps three tables: neighbour, topology and routing. RTP makes Updates, Queries and Replies reliable.
- Default metric = 256 * (BW + delay), with BW from the slowest link and delay summed. Defaults are K1 = 1, K3 = 1, the rest 0, and K values must match.
- A feasible successor needs RD strictly less than FD. That rule proves no loop is possible.
- With an FS, failover is local and fast. Without one, the route goes Active and Queries spread until all Replies arrive.
- `variance` allows unequal cost load balancing, but only over feasible successors.
- Control Query size with stub routers and summarisation to avoid stuck in active.
