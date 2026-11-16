# High Availability and Redundancy

**Level:** Intermediate | **Prerequisites:** [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md), [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md), [BGP](../05-routing/09-bgp.md)

## In One Line

High availability (HA) means a service keeps working when parts of it fail. You get it by removing single points of failure, detecting failure fast, and moving traffic to a healthy path automatically.

## Why It Matters

Everything breaks. Fibre gets cut. Power supplies die. Software crashes. People type the wrong command.

If a company has one internet link and that link fails, the online shop stops selling. Every minute of downtime costs money and trust. For a hospital or a bank, the cost can be much higher than money.

Interviewers ask about HA because it tests real engineering thinking:

- Can you do the maths? ("How much downtime is 99.99%?")
- Can you see hidden weak points in a design?
- Do you know the difference between having a backup and having a backup that works?
- Can you explain trade-offs in cost, complexity and speed?

## Simple Explanation

Think about a car.

A car with one spare tyre has **redundancy**: an extra part that sits ready. When a tyre fails, you stop, change it, and drive on. The trip stops for 20 minutes. This is like **active-passive**: the spare does nothing until it is needed, and there is a short break.

Now think about a big aeroplane with two engines. Both engines run all the time. If one fails, the plane keeps flying on the other. There is no stop. This is like **active-active**: all parts work at once, and the others carry the load when one fails.

The plane must be able to fly on one engine. If it needs both engines to stay in the air, the second engine gives no safety at all. In the same way, two routers that each run at 80% load are not redundant. One router cannot carry 160%.

**Where the analogy stops being true:** a tyre fails in a way you can see and feel. Network failures are often silent. A link can look "up" while it drops all traffic. So the hard part of HA is often not the spare part. It is noticing the failure and deciding what to do.

## How It Works

### Key words

| Term | Plain meaning |
|---|---|
| Availability | The share of time a service works. Written as a percentage. |
| Reliability | How long a thing runs before it fails. |
| Redundancy | Extra parts, so one failure does not stop the service. |
| High availability (HA) | A design goal: very little downtime. Redundancy is one tool for it. |
| Fault tolerance | The service sees no interruption at all when a part fails. Stronger than HA. |
| Disaster recovery (DR) | The plan to recover after a large event, such as a whole site being lost. |
| Failover | Moving work from a failed part to a healthy part. |
| Failback | Moving work back after the failed part is repaired. |
| Single point of failure (SPOF) | One part whose failure stops the whole service. |
| Downtime | Time when the service does not work. |

Reliability and availability are different. A server that crashes every day but restarts in one second is highly available but not reliable. A server that runs for ten years and then takes a month to repair is reliable, but that month hurts its availability.

### Availability and the nines

Availability is measured like this:

```text
Availability = uptime / (uptime + downtime)
```

People describe it in "nines". 99.9% is "three nines". 99.999% is "five nines".

To turn a percentage into downtime:

```text
Downtime = (1 - availability) x total time

Example for 99.99% over one year (365 days = 525,600 minutes):
(1 - 0.9999) x 525,600 = 0.0001 x 525,600 = 52.56 minutes
```

| Availability | Nines | Downtime per year | Per 30-day month | Per week |
|---|---|---|---|---|
| 99% | 2 | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% | 3 | 8.76 hours | 43.2 minutes | 10.08 minutes |
| 99.95% | 3.5 | 4.38 hours | 21.6 minutes | 5.04 minutes |
| 99.99% | 4 | 52.56 minutes | 4.32 minutes | 60.5 seconds |
| 99.999% | 5 | 5.26 minutes | 25.9 seconds | 6.05 seconds |
| 99.9999% | 6 | 31.5 seconds | 2.59 seconds | 0.6 seconds |

Memory trick: 99.9% is about 9 hours a year. Each extra nine divides the downtime by 10.

Five nines is about 5 minutes a year. A human cannot even log in within that time. So above about four nines, recovery must be fully automatic.

**Fuller truth:** the number hides the shape of the failures. One 52-minute outage and 52 one-minute outages give the same percentage, but they feel very different to users. Some contracts also leave out planned maintenance, so always ask what is counted.

### SLA, SLO and SLI

- **SLI** (service level indicator): what you measure. Example: the share of requests that succeed.
- **SLO** (service level objective): your internal target. Example: 99.95% each month.
- **SLA** (service level agreement): a contract with a customer, usually with money back if it is missed.

An SLA does not make a service available. If your ISP misses a 99.9% SLA, you get a small credit on your bill. You do not get your lost sales back.

### Series and parallel maths

**Series** means every part must work. A chain: user, router, firewall, server. If any link fails, the service fails. You multiply:

```text
A(series) = A1 x A2 x A3

Three parts at 99.9%:
0.999 x 0.999 x 0.999 = 0.997 = 99.7%
That is about 26.3 hours of downtime a year.
```

So a chain is always weaker than its weakest part. Adding more parts in series lowers availability.

**Parallel** means any one part is enough. It fails only when all parts fail together. You multiply the chances of failure:

```text
A(parallel) = 1 - (1 - A1) x (1 - A2)

Two parts at 99%:
1 - (0.01 x 0.01) = 1 - 0.0001 = 0.9999 = 99.99%
```

Two weak parts in parallel beat one strong part. Two 99% links give 99.99%.

```text
Series (all must work):

   --[ A ]----[ B ]----[ C ]-->

Parallel (one is enough):

       +--[ A ]--+
   ----+         +---->
       +--[ B ]--+
```

**Fuller truth:** the parallel formula assumes the two parts fail independently. They often do not. Two links in the same trench are cut by the same digger. The formula also assumes failover always works and takes no time. Real results are lower than the maths.

### MTBF and MTTR

- **MTBF** (mean time between failures): the average time a thing runs between failures.
- **MTTR** (mean time to repair): the average time to fix it and return to service.

```text
MTBF = total running time / number of failures

Availability = MTBF / (MTBF + MTTR)
```

Worked example. A site has 10 switches. Each runs for a full year (8,760 hours). There are 4 failures. Each repair takes 4 hours.

```text
Total running time = 10 x 8,760 = 87,600 hours
MTBF = 87,600 / 4 = 21,900 hours
Availability = 21,900 / (21,900 + 4) = 0.99982 = 99.982%
Downtime per switch per year = 0.000183 x 8,760 = about 1.6 hours
```

**Why MTTR is often the better lever.** Downtime is roughly MTTR divided by MTBF. So cutting repair time in half helps as much as doubling the time between failures. Keeping a spare switch on the shelf is often far cheaper than buying a more reliable switch.

MTTR has parts, and each one can be improved:

```text
Failure --> Detect --> Diagnose --> Repair --> Verify --> Service back
            (alert)    (find it)    (fix)      (test)
|<----------------------- MTTR ----------------------------------->|
```

**The MTBF myth.** A data sheet may say MTBF is 1,000,000 hours, which is about 114 years. It does not mean one switch lasts 114 years. It is a failure rate for a large group of units during their normal working life. If you own 1,000 of these, expect about one failure every 1,000 hours, which is about every 42 days.

Related terms:

- **MTTF** (mean time to failure): used for parts that are replaced and not repaired, such as a single disk.
- **MTTD** (mean time to detect): how long before anyone or anything notices.
- MTTR is also used to mean mean time to restore, recover or respond. Ask which one a person means.

### RTO and RPO

These two numbers set the goals for recovery. They come from the business, not from engineers.

- **RTO** (recovery time objective): the longest time the service may be down. "We must be back within 15 minutes."
- **RPO** (recovery point objective): the most data you may lose, measured in time. "We may lose at most 5 minutes of data."

Analogy: you are writing a long document and your laptop dies.

- RPO: your last save was 10 minutes ago, so you lose 10 minutes of work.
- RTO: it takes 30 minutes to get a new laptop and start working again.

**Where the analogy stops being true:** a laptop has one user. A shared service has many writers at once, and the copies on different sites may each hold different recent changes.

```text
                 Last good copy        Failure      Service back
                       |                  |              |
   --------------------+------------------+--------------+------>
                       |<----- RPO ------>|<---- RTO --->|
                       (data lost)         (time down)
```

| Method | Typical RPO | Note |
|---|---|---|
| Nightly backup | Up to 24 hours | Cheap. A crash at 23:00 loses a full day. |
| Asynchronous replication | Seconds | The primary does not wait for the copy. |
| Synchronous replication | Zero | The primary waits for the copy, so every write is slower. |

Distance matters for synchronous replication. Light in fibre travels about 200 km per millisecond. So 100 km adds about 1 ms of round-trip time to every write, before any device delay.

| Service | RTO | RPO |
|---|---|---|
| Bank payments | Under 1 minute | Zero |
| Online shop | 15 minutes | 1 minute |
| Company wiki | 8 hours | 24 hours |

These example numbers show the idea. Each business sets its own.

**RTO and MTTR are different.** RTO is the target. MTTR is what you really measured. If your MTTR is greater than your RTO, you have a problem.

**HA is not backup.** Replication copies mistakes as well. If someone deletes a table, the delete reaches the replica in seconds. Only a backup can bring it back.

### Finding single points of failure

To find a SPOF, walk the full path of a request. At each step, ask: "If this one thing dies, does the service stop?"

```text
Bad design: every box is a SPOF

  Users --- ISP --- Router --- Firewall --- Switch --- Server

Better design: two of each, with cross-connections

         ISP-A           ISP-B
           |               |
        Router1 ------- Router2
           |      \ /      |
           |       X       |
           |      / \      |
          FW1 ----------- FW2
           |      \ /      |
           |       X       |
           |      / \      |
          SW1 ----------- SW2
             \           /
              \         /
               Server(s)
```

The obvious SPOFs are devices and links. The dangerous ones are hidden:

| Hidden SPOF | What goes wrong |
|---|---|
| Same trench or building entry | One digger cuts both "diverse" fibres. |
| Same power feed | Two power supplies on one circuit both die together. |
| Same upstream provider | Two ISPs buy transit from the same carrier. |
| Same software version | One bug crashes both routers. |
| Same automation | One bad config push reaches both devices at once. |
| Same DNS provider | The servers are fine, but nobody can find them. |
| One expired certificate or licence | Both nodes stop on the same day. |
| Out-of-band access | The tool you need to fix the network depends on that network. |

Engineers call parts that fail together **shared fate**. A group of links that share one physical risk is a **shared risk link group** (SRLG).

Physical redundancy matters too: dual power supplies fed from separate A and B circuits, a UPS (battery that bridges a power cut), a generator, and two fibre entries on opposite sides of the building.

A real case: on 4 October 2021, Facebook was offline for about six hours. A maintenance command cut its data centres off from its backbone. Its DNS servers then withdrew their BGP routes. Its redundant hardware did not help, because the failure was in shared configuration and shared tools.

### Active-passive and active-active

**Active-passive:** one unit handles all traffic. The other waits. It is also called active-standby.

**Active-active:** all units handle traffic at the same time.

```text
Active-passive                  Active-active

   Traffic 100%                    Traffic 100%
        |                          /          \
    +--------+   +--------+    50%|            |50%
    | Active |   |Standby |   +--------+   +--------+
    +--------+   +--------+   | Active |   | Active |
                  (idle)      +--------+   +--------+
```

| Point | Active-passive | Active-active |
|---|---|---|
| Use of hardware | Standby sits idle | All capacity is in use |
| Failover time | A short break | Often faster, sometimes none |
| Complexity | Simpler | Harder: state must be shared, traffic must be split |
| Risk | Standby may be broken and nobody knows | Hidden overload when one unit fails |
| Common uses | Firewalls, many databases | Load-balanced web servers, ECMP routing |

**The capacity trap.** Two active units at 70% each look healthy. When one fails, the other needs 140%, and it falls over too. This is a **cascading failure**.

The rule for two units: keep each below 50%. For N units that must survive one failure, keep each below (N-1)/N. Four units may run at up to 75%.

Standby units come in three temperatures:

| Type | State | Failover time |
|---|---|---|
| Hot standby | Powered on, in sync with live state | Seconds or less |
| Warm standby | Powered on, data slightly old | Minutes |
| Cold standby | Off, or in a box on a shelf | Hours or more |

Capacity is often written with N, the number of units you need to carry the load:

- **N:** no spare at all.
- **N+1:** one spare for the whole group.
- **2N:** a full second copy of everything.
- **2N+1:** a full second copy plus one more spare.

### How failover works, step by step

1. **Detect.** Something notices the failure: lost link signal, missed hellos, a failed health check.
2. **Decide.** Which unit takes over? Is this a real failure or a false alarm?
3. **Move traffic.** The new unit takes the IP address, or routes change, or a load balancer stops using the dead server.
4. **Tell the neighbours.** For example, a gratuitous ARP tells switches and hosts the new location of a MAC address. A gratuitous ARP is an ARP message that nobody asked for. See [ARP](../03-ethernet-and-switching/05-arp.md).
5. **Recover sessions.** Connections either carry on, or clients must reconnect.
6. **Fail back.** After repair, return to normal. This can be automatic (called **preemption**) or manual.

Total outage time is detection plus reaction. Detection is usually the largest part, and timers decide it:

| Method | Default or typical timer | Worst-case detection |
|---|---|---|
| Link loss on a directly connected port | Hardware | Milliseconds |
| BFD | 300 ms x 3 (a common setting) | 0.9 seconds |
| VRRP | Advertise every 1 second | About 3.6 seconds |
| HSRP (Cisco default) | Hello 3 s, hold 10 s | 10 seconds |
| EIGRP on fast links (Cisco default) | Hello 5 s, hold 15 s | 15 seconds |
| OSPF on broadcast links | Hello 10 s, dead 40 s | 40 seconds |
| Classic STP | Max age 20 s, forward delay 15 s | Up to 50 seconds |
| BGP on Cisco IOS | Keepalive 60 s, hold 180 s | 3 minutes |
| DNS-based failover | The record's TTL | Minutes, sometimes longer |

Other notes on these timers:

- RFC 4271 suggests a BGP hold time of 90 seconds, and Junos uses that value. Other vendors differ.
- [Rapid Spanning Tree](../03-ethernet-and-switching/09-spanning-tree.md) converges far faster than classic STP.
- HSRP and VRRP are covered in [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

**Stateless and stateful failover.** A router only forwards packets, so a new router can carry on at once. A firewall or NAT device keeps a table of live connections. If the standby does not have that table, it drops all existing sessions, and users must reconnect. **Stateful failover** copies the table to the standby all the time, so sessions survive. See [Firewalls](../09-security/04-firewalls.md).

### BFD: fast failure detection

**The problem.** Many failures do not bring the link down. Suppose two routers connect through a carrier switch. The far side dies, but the near port still shows "up". The router only finds out when its routing protocol timers expire. With BGP on default Cisco timers, that takes up to 3 minutes.

**BFD** (Bidirectional Forwarding Detection, RFC 5880) is a tiny protocol with one job: check that a path to a neighbour is working. It sends small packets very often. If they stop, it tells the routing protocol at once.

Analogy: two people on a phone call say "still there?" every second. If you hear nothing three times, you hang up. **Where the analogy stops being true:** BFD does not choose a new path. It only raises the alarm. OSPF or BGP then does the rerouting.

```text
R1 and R2 exchange BFD packets every 300 ms

R1                                  R2
 |------- BFD (I am Up) ----------->|
 |<------ BFD (I am Up) ------------|
 |                                  X  R2 dies
 |   300 ms: nothing                |
 |   600 ms: nothing                |
 |   900 ms: nothing                |
 |                                  |
 BFD session Down
 BFD tells BGP: neighbor is gone
 BGP removes routes, picks the backup path
```

The formula:

```text
Detection time = interval x multiplier
300 ms x 3 = 900 ms
```

Key facts:

- Runs over UDP. Single-hop control packets use destination port 3784. Multihop uses 4784.
- Has states: AdminDown, Down, Init, Up.
- Does not find neighbours by itself. A client such as OSPF, BGP or a static route asks for a session.
- Can run in hardware on many platforms, which allows short timers such as 50 ms x 3.

Cisco IOS example:

```text
! Cisco IOS: BFD with BGP and OSPF
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3
!
router bgp 64512
 neighbor 198.51.100.1 fall-over bfd
!
router ospf 1
 bfd all-interfaces
```

```text
R1# show bfd neighbors

IPv4 Sessions
NeighAddr                    LD/RD         RH/RS     State     Int
198.51.100.1              4097/4098        Up        Up        Gi0/0
```

BFD must be set up on both ends, so your ISP must agree to run it. Not all do.

On Linux routers, the FRR routing suite provides BFD through its `bfdd` daemon.

**Why not just lower the OSPF or BGP timers?** You can, but those protocols run on the main CPU and each must be tuned on its own. BFD is one light mechanism that many protocols share.

**Do not set timers too low.** Very short timers on a busy device or a slightly lossy link cause false alarms. Each false alarm tears down routing and makes things worse.

### NIC teaming: redundancy for servers

A server with one network card (NIC) has a SPOF: the card, the cable and the switch port. **NIC teaming**, called **bonding** on Linux, joins two or more NICs into one logical interface with one IP address.

```text
Same-switch teaming              Two-switch teaming (better)

    +--------+                       +--------+
    | Server |                       | Server |
    +--------+                       +--------+
     eth0 eth1                        eth0  eth1
       |   |                           |      |
    +--------+                      +-----+ +-----+
    |  SW1   |                      | SW1 |-| SW2 |
    +--------+                      +-----+ +-----+
  (SW1 is still a SPOF)             (no switch SPOF)
```

Linux bonding modes:

| Number | Name | What it does | Switch setup needed |
|---|---|---|---|
| 0 | balance-rr | Sends packets in turn on each NIC. Can reorder packets. | Yes, static EtherChannel |
| 1 | active-backup | One NIC works, the other waits | No |
| 2 | balance-xor | Picks a NIC by a hash | Yes, static EtherChannel |
| 3 | broadcast | Sends every packet on all NICs | Yes, static EtherChannel |
| 4 | 802.3ad | LACP. The normal choice for load sharing. | Yes, LACP |
| 5 | balance-tlb | Shares transmit load | No |
| 6 | balance-alb | Shares transmit and receive load | No |

**Teaming does not double the speed of one connection.** Most modes pick a NIC per flow using a hash, so one TCP session stays on one 10G link. Many flows together can use both. See [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md).

**LACP to two switches.** LACP normally expects one switch at the far end. To go to two switches, they must act as one, using a stack or MLAG (Cisco calls its version vPC). Active-backup has no such need.

Linux setup with active-backup:

```bash
# Linux: create an active-backup bond, check link every 100 ms
sudo ip link add bond0 type bond mode active-backup miimon 100
sudo ip link set eth0 down
sudo ip link set eth1 down
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0
sudo ip link set bond0 up
sudo ip addr add 10.10.10.20/24 dev bond0
sudo ip route add default via 10.10.10.1
```

These commands do not survive a reboot. For a lasting setup, use your distribution's network tool, such as NetworkManager or netplan.

```bash
$ cat /proc/net/bonding/bond0
Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100

Slave Interface: eth0
MII Status: up
Link Failure Count: 0
Permanent HW addr: 02:00:00:00:10:01

Slave Interface: eth1
MII Status: up
Link Failure Count: 0
Permanent HW addr: 02:00:00:00:10:02
```

Windows Server setup:

```powershell
# Windows Server LBFO teaming (not on Windows 10 or 11 client)
New-NetLbfoTeam -Name "Team1" -TeamMembers "Ethernet1","Ethernet2" `
  -TeamingMode SwitchIndependent -LoadBalancingAlgorithm Dynamic

# Make the second NIC a standby, like Linux active-backup
Set-NetLbfoTeamMember -Name "Ethernet2" -AdministrativeMode Standby

New-NetIPAddress -InterfaceAlias "Team1" -IPAddress 10.10.10.20 `
  -PrefixLength 24 -DefaultGateway 10.10.10.1

Get-NetLbfoTeam
```

For Hyper-V hosts, Microsoft recommends Switch Embedded Teaming (SET) in place of LBFO.

**A trap with link monitoring.** Linux `miimon` only checks the local link. If the cable is fine but the switch's uplink is dead, the bond stays on the bad path. ARP monitoring (`arp_interval` and `arp_ip_target`) sends ARP requests to a real target, so it catches more failures.

### Dual ISP

One ISP is a SPOF. There are two main ways to use two ISPs.

| Point | Without BGP | With BGP |
|---|---|---|
| Your public IPs | One set from each ISP | One block of your own, reachable via both |
| What you need | Two links and NAT | Your own AS number and address block, plus ISP support |
| Outbound failover | Yes, by tracked static routes | Yes, automatic |
| Inbound failover | Weak. Depends on DNS TTL. | Yes, the Internet reroutes to you |
| Live sessions after failover | Break, because the source IP changes | Can survive, because the IP stays the same |
| Cost and skill | Low | Higher |

**Without BGP** you use two default routes and a health check. A plain static route only fails if the local link goes down. It will not notice that the ISP is broken further along. So use IP SLA, which sends test pings, and tie the route to the result.

```text
! Cisco IOS: primary default via ISP-A, backup via ISP-B
ip sla 1
 icmp-echo 198.51.100.77 source-interface GigabitEthernet0/0
 frequency 5
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
 delay down 10 up 30
!
! Keep the probe on the ISP-A link at all times
ip route 198.51.100.77 255.255.255.255 198.51.100.1
!
! Primary route, removed if track 1 goes down
ip route 0.0.0.0 0.0.0.0 198.51.100.1 track 1
! Backup route with a higher administrative distance
ip route 0.0.0.0 0.0.0.0 192.0.2.1 10
```

The address 198.51.100.77 stands for a stable host beyond ISP-A. Use a target you trust. A public server may rate-limit or drop your pings, and that would cause a false failover.

The `delay down 10 up 30` line stops the route from flapping (going up and down again and again). The tracked object must stay failed for 10 seconds before the route is removed. It must stay healthy for 30 seconds before it returns.

You also need NAT rules for each ISP link, because each ISP only accepts its own addresses. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md). For inbound services, you need DNS with a low TTL, or both addresses in DNS. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

Choosing the two ISPs well matters as much as the config. Ask for proof of different physical paths and different building entries. A cheap 5G or LTE backup link is a good choice for a branch, since it shares no cable with the main link. [SD-WAN](10-sd-wan.md) automates much of this.

### BGP multihoming

**Multihoming** means connecting one network to the Internet through more than one link. Words vary, but this is the usual use:

| Name | Meaning |
|---|---|
| Single-homed | One link to one ISP |
| Dual-homed | Two links to the same ISP |
| Multihomed | Links to two or more ISPs |
| Dual multihomed | Two links to each of two ISPs |

Dual-homed protects against a link or router failure, but not an ISP failure.

Analogy: your village has two roads, and every map in the country shows both. If one road closes, the map makers publish a change and drivers take the other. **Where the analogy stops being true:** on the Internet, you do not choose the road people use to reach you. Each network picks its own best path. You can only give hints.

**What you need:**

- A public AS number (ASN). Private ASNs (64512-65534, RFC 6996) are only for internal use and are used here as placeholders.
- An address block that can be announced by both ISPs. For IPv4 this is at least a /24. For IPv6 it is at least a /48. Networks often filter longer prefixes.
- Agreement from both ISPs, and enough memory if you take full routes.

**Which routes to take from the ISPs:** a default route only (simple, little control), partial routes (default plus the ISP's customer routes), or the full table (around one million IPv4 routes, so it needs a strong router).

**Controlling traffic.** Outbound and inbound use different tools. Remember this rule: **you control what you send, and you can only influence what you receive.**

| Direction | Tool | How it works |
|---|---|---|
| Outbound | Local preference | Higher wins. Set it on routes from the ISP you prefer. |
| Inbound | AS path prepending | Add your AS number several times so that path looks longer. |
| Inbound | More specific prefixes | Longest match wins, so a /24 beats a /23. |
| Inbound | ISP communities | Tags that ask the ISP to change how it treats your route. |
| Inbound | MED | A hint between you and one neighbouring AS. Usually not compared across two different ISPs. |

To split inbound traffic with more specific prefixes, you need a block larger than /24. With a /23 you can announce one /24 to each ISP, plus the /23 to both as a safety net. A single /24 cannot be split, because the /25s would be filtered.

**Never become a transit network.** If you pass routes from ISP-A on to ISP-B, the Internet may send traffic between them through your small link. This is a **route leak**. Always filter outbound, so you only announce your own prefixes. Every multihomed network needs this rule.

BGP is covered in full in the [BGP chapter](../05-routing/09-bgp.md).

### Split brain

**Split brain** happens when two HA nodes lose contact with each other and both decide they are the active one.

Analogy: a shop has two managers who share work by phone. The phone line dies. Each thinks the other has gone home, so each orders stock and pays the same bills. **Where the analogy stops being true:** the managers can meet later and sort it out. Two databases that took different writes may have no clean way to merge them.

```text
Normal                          Split brain

 +------+ heartbeat +------+     +------+    X    +------+
 |Active|<--------->|Stdby |     |Active|  link   |Active|
 +------+           +------+     +------+  lost   +------+
    |                   |           |                 |
 owns 10.10.10.1     waits       claims 10.10.10.1  claims too!
```

A **heartbeat** is a regular "I am alive" message between nodes. The common cause of split brain is that the heartbeat path fails while both nodes are still healthy.

**What goes wrong:**

- Two devices answer for one IP and MAC address, so traffic flaps between them.
- Two databases accept different writes, and data no longer matches. This is the worst case.
- Two firewalls each hold only part of the session table.

**How to prevent it:**

| Method | How it works |
|---|---|
| More than one heartbeat path | Use two separate links, so one failure does not hide a healthy peer. |
| Quorum | A node may stay active only if it can reach a majority of voters. |
| Witness (arbiter) | A small third node in another place casts the deciding vote. |
| Fencing | Force the other node off, or cut its storage, before taking over. |
| Manual failover | A human decides. Slower, but safe for critical data. |

**Quorum** is the number of votes needed to act. It is a majority: floor(N/2) + 1, where floor means round down.

| Nodes | Quorum | Failures tolerated |
|---|---|---|
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |

So clusters use odd numbers. Four nodes survive no more failures than three. Two nodes cannot form a safe quorum alone, which is why two-node clusters need a witness or fencing.

**Fencing** in Linux clusters is often called **STONITH** ("shoot the other node in the head"). Pacemaker with Corosync is a common Linux cluster stack, and it can power off a peer through its management card.

In networking gear, the same problem appears as **dual-active**:

- **Cisco vPC:** the peer link carries data and sync traffic. A separate **peer-keepalive** link tells each switch that its peer is still alive, so a lost peer link does not cause dual-active.
- **Switch stacks and VSS:** they have their own dual-active detection methods. Details vary by platform.
- **HSRP or VRRP:** if hellos are blocked by a missing VLAN or an ACL, both routers become active.

```text
R1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active      Standby     Virtual IP
Gi0/1       10   110 P Active  local       unknown     10.10.10.1
```

R2 shows the same line with priority 100: also Active, also `local`, also `unknown`.

Two Active routers, with Standby `unknown` on both, is the classic sign. Check that VLAN on the trunk between them first.

**The core trade-off:** when the network splits, a node cannot tell whether its peer is dead or only unreachable. It must choose. It can keep serving and risk bad data, or stop and lose availability. For data, safe systems usually stop. This is the CAP theorem in practice.

## Example

**The job.** A company runs a web shop from one site. The business asks for 99.99% availability on the network path. They own 203.0.113.0/24 and use AS 64512.

**Step 1: turn the target into a number.** 99.99% is 52.56 minutes of downtime per year.

**Step 2: measure the present design.** Everything is single and in series.

| Part | Availability |
|---|---|
| ISP link | 99.5% |
| Router | 99.9% |
| Switch | 99.95% |

```text
0.995 x 0.999 x 0.9995 = 0.99351 = 99.35%
Downtime = 0.00649 x 8,760 hours = about 56.9 hours a year
```

This is far from the goal.

**Step 3: double each tier and calculate again.**

```text
Links:    1 - (0.005  x 0.005)  = 0.999975
Routers:  1 - (0.001  x 0.001)  = 0.999999
Switches: 1 - (0.0005 x 0.0005) = 0.99999975

Total = 0.999975 x 0.999999 x 0.99999975 = 0.99997375 = 99.9974%
Downtime = 0.00002625 x 525,600 minutes = about 13.8 minutes a year
```

Now add failover time. Say there are 10 failures a year and each causes a 30-second break. That adds 5 minutes, for 18.8 minutes in total, or about 99.996%. It meets the target, on paper.

**Step 4: build it.**

```text
                     Internet
                    /        \
      ISP-A AS 64501          ISP-B AS 64502
      198.51.100.1            192.0.2.1
           |                       |
      198.51.100.2            192.0.2.2
        Gi0/0                   Gi0/0
       +------+  10.0.12.0/30  +------+
       |  R1  |----------------|  R2  |
       +------+ Gi0/2    Gi0/2 +------+
        Gi0/1                   Gi0/1
      10.10.10.2              10.10.10.3
           |  HSRP VIP 10.10.10.1  |
       +------+                +------+
       | SW1  |----------------| SW2  |
       +------+                +------+
            \                  /
           eth0              eth1
              \              /
              +--------------+
              | Server bond0 |
              | 10.10.10.20  |
              +--------------+
```

The server uses the active-backup bond shown earlier, with gateway 10.10.10.1. That gateway is a virtual IP shared by R1 and R2 through HSRP. A real site also has a firewall and NAT pair between the routers and the servers. It is left out to keep the drawing small.

Each ISP sends only a default route. R1 is the preferred exit. The design is active-passive in both directions, so that either link can carry all the traffic.

R1 configuration:

```text
! Cisco IOS: R1
interface GigabitEthernet0/0
 description Link to ISP-A
 ip address 198.51.100.2 255.255.255.252
 bfd interval 300 min_rx 300 multiplier 3
!
interface GigabitEthernet0/1
 description LAN
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.1
 standby 10 timers 1 3
 standby 10 priority 110
 standby 10 preempt delay minimum 60
!
interface GigabitEthernet0/2
 description Link to R2
 ip address 10.0.12.1 255.255.255.252
!
ip prefix-list OUR-NET seq 10 permit 203.0.113.0/24
!
route-map FROM-ISP-A permit 10
 set local-preference 200
!
route-map TO-ISP-A permit 10
 match ip address prefix-list OUR-NET
!
router bgp 64512
 bgp router-id 10.0.0.1
 neighbor 198.51.100.1 remote-as 64501
 neighbor 198.51.100.1 fall-over bfd
 neighbor 10.0.12.2 remote-as 64512
 !
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.0
  neighbor 198.51.100.1 activate
  neighbor 198.51.100.1 route-map FROM-ISP-A in
  neighbor 198.51.100.1 route-map TO-ISP-A out
  neighbor 10.0.12.2 activate
  neighbor 10.0.12.2 next-hop-self
 exit-address-family
!
! The network statement needs this exact route to exist
ip route 203.0.113.0 255.255.255.0 Null0
```

What each part does:

- `local-preference 200` makes all traffic leave through ISP-A while it is up.
- `TO-ISP-A` allows only our own prefix out, so we never become a transit network.
- `fall-over bfd` drops the BGP session as soon as BFD fails.
- `preempt delay minimum 60` gives R1 time to bring up BGP before it takes the gateway role back.

R2 is the same, with its own addresses. It has no local-preference line, so its ISP-B routes keep the default of 100. Its HSRP priority stays at the default of 100. It also makes the ISP-B path look longer to the Internet, using a route map:

```text
! Cisco IOS: R2, outbound policy toward ISP-B
route-map TO-ISP-B permit 10
 match ip address prefix-list OUR-NET
 set as-path prepend 64512 64512
```

**Step 5: verify.**

```text
R2# show ip bgp
     Network          Next Hop         Metric LocPrf Weight Path
 *>i 0.0.0.0          10.0.12.1             0    200      0 64501 i
 *                    192.0.2.1             0             0 64502 i
 *>  203.0.113.0/24   0.0.0.0               0         32768 i
 * i                  10.0.12.1             0    100      0 i
```

R2 holds the ISP-B default as a backup, but it uses the path through R1, because local preference 200 beats 100.

**Step 6: test each failure, on purpose.**

| Failure | Detected by | Time | What happens |
|---|---|---|---|
| Server NIC or cable | Bond link check | About 0.1 s | eth1 takes over and sends a gratuitous ARP. |
| ISP-A fails, port stays up | BFD | 0.9 s | R1 drops the session. R2 becomes the exit. R1 sends traffic through R2. |
| R1 dies | HSRP, BFD | Up to 3 s | R2 becomes the gateway and uses ISP-B. |
| SW1 dies | Bond, HSRP | Up to 3 s | Server moves to eth1. R2 becomes gateway and still exits through R1 over the R1-R2 link. |
| R1-R2 link fails | BGP on the iBGP session | Up to 180 s, but no user impact | Both routers keep their own ISP path. R1 stays the gateway. |

When ISP-A fails, the router sees this:

```text
%BGP-5-ADJCHANGE: neighbor 198.51.100.1 Down BFD adjacency down
```

Outbound recovers in about a second. Inbound takes longer. ISP-A must withdraw 203.0.113.0/24, and that news must spread across the Internet. This takes seconds to minutes. During this time, some traffic toward you is lost.

Our public addresses do not change, so TCP sessions can survive, as long as the firewall or NAT device holding their state did not fail. That is the big gain over the NAT-only dual ISP design.

## Going Deeper

### Redundancy per tier beats two separate paths

There are two ways to spend two sets of hardware. Using the example numbers:

```text
Two full chains, no cross-links:
One chain = 0.99351
1 - (0.00649 x 0.00649) = 0.999958 = 99.9958%, about 22.2 min a year

Two of each part, with cross-links:
= 99.9974%, about 13.8 min a year
```

With separate chains, one link failure on chain A and one switch failure on chain B stop everything. With cross-links, any working part in each tier can be combined with any working part in the next. The price is more links and more failover paths to test.

### More parts can mean less availability

Redundant systems add software, sync links and failover logic. These can fail too. Many large outages are caused by the HA system itself: a false failover, a bad sync, or a failover that overloads the survivor.

A standby can also break quietly. This is a **latent failure**. Monitor the standby as closely as the active unit, and test failover on a schedule. Netflix made this idea well known with Chaos Monkey, a tool that shuts down production servers at random so that teams must build for it.

### Gray failures

A **gray failure** is a partial failure. A link drops 3% of packets. A line card corrupts some frames. BFD packets may still get through, so nothing fails over, while users suffer. Catch these with interface error counters and path probes. See [Network Monitoring](../12-troubleshooting-and-tools/11-network-monitoring.md).

A failing link can also **flap**. Each flap causes a routing change, which can be worse than a clean failure. Hold-down timers and dampening keep a flapping link out of service until it has been stable for some time.

### BFD details

**Control packet.** The mandatory part is 24 bytes:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-----+---------+---+-+-+-+-+-+-+---------------+---------------+
|Vers |  Diag   |Sta|P|F|C|A|D|M|  Detect Mult  |    Length     |
+-----+---------+---+-+-+-+-+-+-+---------------+---------------+
|                       My Discriminator                        |
+---------------------------------------------------------------+
|                      Your Discriminator                       |
+---------------------------------------------------------------+
|                    Desired Min TX Interval                    |
+---------------------------------------------------------------+
|                   Required Min RX Interval                    |
+---------------------------------------------------------------+
|                 Required Min Echo RX Interval                 |
+---------------------------------------------------------------+
```

- **Discriminators** are session IDs. Each side picks its own.
- **Sta** is the state. **Diag** says why the session last went down.
- **Intervals** are in microseconds.
- The **C** bit means BFD does not depend on the control plane, so it can keep running through a control plane restart.

**Timer negotiation.** Each side says how fast it wants to send and how fast it can receive. The sender uses the slower of the two. Each side can have a different result:

```text
R1: tx 300 ms, rx 300 ms, multiplier 3
R2: tx 100 ms, rx 500 ms, multiplier 4

R2 sends to R1 every max(100, 300) = 300 ms
R1 sends to R2 every max(300, 500) = 500 ms

R1 detects R2 failure in 4 x 300 ms = 1,200 ms
R2 detects R1 failure in 3 x 500 ms = 1,500 ms
```

The multiplier used is the one sent by the remote side. While a session is not Up, RFC 5880 requires a transmit interval of at least one second, so startup is slow on purpose.

**Modes.** In the normal **asynchronous mode**, both sides send packets on a timer. In **echo mode**, a router sends packets that the neighbour loops back through its forwarding path. This tests the data path without using the neighbour's CPU.

**Related RFCs:** RFC 5881 (single hop over IPv4 and IPv6), RFC 5883 (multihop), RFC 7130 (a BFD session on each member link of a LAG).

### Keeping forwarding alive during a restart

Modern routers separate the **control plane** (the CPU that runs protocols) from the **data plane** (the hardware that forwards packets). This allows failover inside one box.

- **SSO** (stateful switchover, a Cisco term): a chassis has two supervisor cards. If one fails, the other takes over with state already copied.
- **NSF** (non-stop forwarding, a Cisco term) and **graceful restart**: while the control plane restarts, the data plane keeps forwarding, and neighbours agree to keep the routes for a short time. See RFC 4724 for BGP and RFC 3623 for OSPF.
- **ISSU** (in-service software upgrade): upgrading software with little or no traffic loss. Support varies by platform.

There is a conflict here. BFD wants to report failure in under a second. Graceful restart wants neighbours to be patient. If BFD runs on the CPU, a restart looks like a failure, and neighbours drop the routes that graceful restart meant to save. Hardware-based BFD with the C bit set helps. Check your platform.

### Stack, chassis or MLAG

| Option | Control planes | Main risk |
|---|---|---|
| Switch stack | One shared | A software bug or an upgrade can hit all members |
| Dual-supervisor chassis | One active, one standby | The chassis itself is shared |
| MLAG or vPC | Two independent | More complex. Split-brain handling is needed. |
| Two routed devices (ECMP) | Two independent | Must be Layer 3 to the edge. See [Spine-Leaf](02-spine-leaf.md). |

ECMP (equal-cost multi-path) sends traffic over several equal routes at the same time, which makes a routed design active-active by nature. In large data centres, this is now the usual design, with servers behind pairs of switches. [VXLAN and EVPN](11-vxlan-and-evpn.md) can replace MLAG with EVPN multihoming.

### HA above the network

Network redundancy cannot fix a broken application. Other layers matter too:

- [Load balancers](04-load-balancing.md) run health checks and remove dead servers.
- [Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md) announces one address from many sites, and traffic goes to the nearest healthy one.
- Clouds offer Availability Zones, which are separate failure areas inside one region. See [AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).
- Before planned maintenance, drain traffic first. RFC 8326 defines a BGP community for this, called GRACEFUL_SHUTDOWN.

## Common Mistakes

- **Thinking redundancy equals availability.** Two devices with one shared power circuit, trench or software version can fail together.
- **Never testing failover.** A standby that has never been tested is only a hope. Test it on a schedule, in production where you can.
- **Running active-active above 50% each.** The survivor gets overloaded, and one failure becomes two.
- **Mixing up RTO and RPO.** RTO is time down. RPO is data lost.
- **Treating replication as a backup.** Replication copies deletes and corruption too.
- **Thinking teaming doubles the speed of one download.** One flow normally stays on one link.
- **Using a static backup route with no tracking.** It does not notice an ISP failure beyond the first hop.
- **Using MED to choose between two different ISPs.** It is normally compared only between routes from the same neighbouring AS.
- **Skipping outbound BGP filters.** This can turn your network into transit for your two ISPs.
- **Setting BFD timers too low.** False alarms cause more outages than they prevent.
- **Building a two-node cluster with no witness or fencing.** That is a split-brain waiting to happen.
- **Leaving preemption with no delay.** A router may take the gateway role back before its routes are ready, and traffic is lost.

## Interview Questions

**Q: How much downtime per year does 99.99% allow, and how do you work it out?**

A: About 52.6 minutes. I take 1 minus the availability, which is 0.0001, and multiply by the minutes in a year, 525,600. A quick rule is that 99.9% is about 9 hours a year, and each extra nine divides that by 10.

**Q: You have three devices in series, each at 99.9%. What is the total? What if you put two 99% links in parallel?**

A: In series I multiply, so 0.999 cubed is about 99.7%, which is worse than any single part. In parallel I multiply the failure chances, so 0.01 times 0.01 is 0.0001, which gives 99.99%. The parallel result assumes the links fail independently and failover works.

**Q: What is a single point of failure, and how do you find one?**

A: It is one part whose failure stops the whole service. I walk the full request path and ask what happens if each thing dies. I also look for hidden ones: shared fibre routes, shared power, the same software version, DNS, certificates and management access.

**Q: Compare active-active and active-passive.**

A: In active-passive, one unit does the work and the other waits, so it is simple but wastes capacity and has a short failover break. In active-active, all units carry traffic, so capacity is used and failover is often faster, but it is more complex. Each unit must stay under about 50% in a pair, so the survivor can carry everything.

**Q: What are MTBF and MTTR, and which would you improve first?**

A: MTBF is the average time between failures, and MTTR is the average time to restore service. Availability is MTBF divided by MTBF plus MTTR. I usually start with MTTR, through better monitoring, spare parts and automation, because halving it helps as much as doubling MTBF, and it is often cheaper.

**Q: What is the difference between RTO and RPO?**

A: RTO is how long the service may be down. RPO is how much data you may lose, measured in time. A nightly backup gives an RPO of up to 24 hours. Synchronous replication gives an RPO of zero, but it adds latency to every write.

**Q: What is BFD, and why use it instead of lowering routing timers?**

A: BFD is a light protocol that checks a forwarding path, using small UDP packets at short intervals. With 300 ms and a multiplier of 3, it detects failure in 0.9 seconds and tells OSPF or BGP. One mechanism serves many protocols, and it can run in hardware, while routing protocol timers run on the CPU and must be tuned one by one. It is useful when a link fails without the port going down.

**Q: What are the main NIC teaming modes? Does teaming double bandwidth?**

A: On Linux, the common ones are active-backup, which needs no switch config, and 802.3ad, which uses LACP. Teaming adds total capacity across many flows, but one flow usually stays on one link, so a single download does not get faster. For the best protection, I connect the NICs to two switches, and for LACP those switches need to be a stack or MLAG pair.

**Q: How do you connect a site to two ISPs, with and without BGP?**

A: Without BGP, I use a primary default route tracked by IP SLA, a floating static backup, and NAT for each ISP. It is cheap, but inbound is weak and sessions break because the source address changes. With BGP, I announce my own prefix to both ISPs, so inbound fails over too and the addresses stay the same. That needs a public ASN, a /24 or larger, and ISP support.

**Q: In BGP multihoming, how do you control outbound and inbound traffic?**

A: For outbound, I set local preference, since I decide where my own traffic goes. For inbound, I can only influence others, using AS path prepending, more specific prefixes, or ISP communities. MED usually does not help across two different ISPs.

**Q: How do you stop your network becoming a transit path between your two ISPs?**

A: I apply an outbound filter on each eBGP session, so I only announce my own prefixes. That can be a prefix list, or an AS path filter that matches only routes with an empty AS path. Without it, I could leak routes and pull Internet traffic through my links.

**Q: What is split brain, and how do you prevent it?**

A: Two nodes lose contact and both act as active, so you get duplicate IPs or data that no longer matches. I prevent it with multiple heartbeat paths, quorum with an odd number of voters or a witness, and fencing to force the other node off. On Cisco vPC, the peer-keepalive link serves this purpose.

## Quick Recap

- Availability is uptime divided by total time. 99.9% is about 8.76 hours a year. 99.99% is about 52.6 minutes. 99.999% is about 5.3 minutes.
- Series parts multiply availability. Parallel parts multiply failure chances. Real designs fall short of the maths because of shared risks and failover time.
- Availability = MTBF / (MTBF + MTTR). RTO is allowed time down. RPO is allowed data loss.
- Find SPOFs by walking the whole path, including power, fibre routes, software, DNS and people.
- Active-passive is simple. Active-active uses all capacity, but keep enough spare for one failure.
- BFD gives sub-second detection. NIC teaming protects servers. Dual ISP with BGP protects both directions, but inbound can only be influenced.
- Split brain is prevented by multiple heartbeat paths, quorum, witnesses and fencing. A failover you have not tested is not a real failover.
