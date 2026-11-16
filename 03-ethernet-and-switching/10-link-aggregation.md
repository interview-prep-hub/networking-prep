# Link Aggregation (EtherChannel, LACP)

**Level:** Intermediate | **Prerequisites:** [How Switches Work](03-how-switches-work.md), [Trunking and 802.1Q](07-trunking-8021q.md), [Spanning Tree Protocol (STP)](09-spanning-tree.md)

## In One Line

Link aggregation joins several physical Ethernet links into one logical link, which gives more total bandwidth and keeps traffic moving when a cable fails. Each single flow still uses only one physical link.

## Why It Matters

Say two switches are joined by one 1 Gbit/s cable. The link is full, and that one cable is a single point of failure. A natural fix is to add a second cable.

This does not work on its own. Two cables between two switches form a loop. Spanning Tree Protocol (STP, the protocol that stops Layer 2 loops) blocks one cable. You paid for two links and you can use one.

```text
Two normal links: STP blocks one

  +-----+ Gi1/0/1 forwarding +-----+
  | SW1 |====================| SW2 |    Usable: 1 Gbit/s
  |     |---------XX---------|     |
  +-----+ Gi1/0/2 blocked    +-----+

Two links in one bundle: STP sees one link

  +-----+ Gi1/0/1            +-----+
  | SW1 |====================| SW2 |    Usable: 2 Gbit/s total
  |     |====================|     |    Lose one cable: 1 Gbit/s
  +-----+ Gi1/0/2            +-----+    and no STP change
          \____Port-channel1____/
```

Link aggregation solves three problems at once:

- **More capacity.** Two 10 Gbit/s links give 20 Gbit/s in total.
- **Redundancy.** If one link fails, traffic moves to the others, usually in under a second.
- **No blocked links.** STP sees one logical link, so nothing is blocked.

Interviewers like this topic because it tests real understanding. Many candidates say "two 10G links make a 20G link". The strong answer is: "20G in total, but one TCP connection still gets at most 10G, and here is why."

**Names you will hear.** They mostly mean the same thing:

| Name | Who uses it |
|------|-------------|
| Link aggregation, LAG (Link Aggregation Group) | IEEE and most vendors |
| EtherChannel, port-channel | Cisco |
| Bond | Linux |
| Team | Windows Server |
| Trunk | Some vendors (HPE, for example). Do not mix this up with an 802.1Q VLAN trunk |

## Simple Explanation

Think of a motorway.

One lane carries a certain number of cars per hour. Each car has a speed limit of 100 km/h. Add three more lanes and the road carries four times as many cars. But one car still drives at 100 km/h. Extra lanes give more total traffic, not a faster car.

A **flow** is one car. A flow is a group of frames that belong to one conversation, such as one TCP connection between a browser and a server. A **physical link** is one lane.

Now picture a sign at the start of the motorway. It reads each car's number plate and does a quick sum, and the answer picks the lane. The same plate always gives the same answer, so the car stays in one lane for the whole trip. That sum is the **hash**: a fast calculation that turns some input (here, addresses) into a small number.

**Where the analogy stops being true:** A real driver sees an empty lane and moves to it. A hash does not look at how busy a lane is. It only does the sum. Two big flows can land on the same link while another link sits idle. On a real road, cars can also change lanes. A flow normally cannot.

## How It Works

### The logical interface

You put physical ports into a group. The switch creates one logical interface for that group. On Cisco it is called `Port-channel1`. On Linux it is called `bond0`.

- The physical ports are called **member ports** or **member links**.
- VLANs, trunking and IP addresses are configured on the logical interface.
- The switch learns MAC addresses on the logical interface, not on one member. So its MAC table shows `Po1`, not `Gi1/0/1`.
- STP treats the bundle as one port.

### Why a bundle does not make a loop

Two rules prevent loops inside the bundle:

1. A flooded frame (broadcast, unknown unicast or multicast) is sent out only one member of the bundle, not all of them.
2. A frame that arrives on a bundle is never sent back out of the same bundle.

### How a frame picks a link

Each sending device runs this process for each frame it sends:

1. Read chosen fields from the frame. These can be MAC addresses, IP addresses or TCP/UDP ports.
2. Run a hash on those fields. The result is a small number.
3. Map that number to one member link.
4. Send the frame on that link.

Same input gives same output. So all frames of one flow take the same link and stay in order. This is the whole reason hashing is used. If frames of one TCP connection were spread across links, they could arrive out of order, and TCP could treat that as packet loss and slow down. See [TCP Reliability and Flow Control](../06-transport-layer/05-tcp-reliability-and-flow-control.md).

### A hash example with real numbers

Here is a simple model: XOR the last 2 bits of the source and destination IP. XOR means "compare two bits: the result is 1 if they differ, 0 if they match." Two bits give four answers (0-3), which is perfect for four links.

```text
Bundle: 4 links. Link 0=Gi1/0/1  1=Gi1/0/2  2=Gi1/0/3  3=Gi1/0/4
Hash input: last 2 bits of source IP XOR last 2 bits of dest IP

Source      Last 2   Dest        Last 2   XOR   Link used
10.1.10.11  11       10.1.20.5   01       10    2 (Gi1/0/3)
10.1.10.12  00       10.1.20.5   01       01    1 (Gi1/0/2)
10.1.10.13  01       10.1.20.5   01       00    0 (Gi1/0/1)
10.1.10.14  10       10.1.20.5   01       11    3 (Gi1/0/4)
```

Four clients talk to one server and each gets a different link. That spread is good, but it comes from the addresses. Clients 10.1.10.11 and 10.1.10.15 both end in `11` and would share a link.

**Fuller truth:** Real switches use stronger, vendor-specific hash functions, often over more bits. Few vendors publish the details. The idea is the same: fixed fields go in, one link comes out.

### Which fields go into the hash

| Setting (Cisco name) | Input | Good for | Bad when |
|----------------------|-------|----------|----------|
| `src-mac` | Source MAC | Many clients sending through a switch | All traffic comes from one router or gateway |
| `dst-mac` | Destination MAC | Many clients receiving | All traffic goes to one gateway |
| `src-dst-mac` | Both MACs | Mixed Layer 2 traffic | Two routers talk over the bundle: there is only one MAC pair |
| `src-dst-ip` | Both IPs | Most networks | One client and one server use many connections |
| `src-dst-mixed-ip-port` | IPs and TCP/UDP ports | Best spread on many platforms | Hardly ever bad, but not every platform has it |

The available options and the default vary by platform and software version. Many older access switches default to `src-mac`. Check yours with `show etherchannel load-balance`. On Catalyst switches this setting is usually global, so it applies to all port-channels.

A classic trap: a router sends through a switch to a firewall. Every frame has the same source MAC and the same destination MAC. With `src-dst-mac`, the hash input never changes, so every frame takes the same link. The other links carry nothing. The fix is to hash on IP addresses and ports.

Each side chooses its own hash, and only for the traffic it sends. The two ends do not need to match. SW1 can use `src-dst-ip` and SW2 can use `src-mac`. This also means that a flow's two directions can use different links, which is normal.

### Why one flow does not get double speed

Follow one TCP connection across a 2 x 10 Gbit/s bundle:

1. The first frame is hashed. The result is link 1.
2. Every later frame in this connection has the same addresses and ports.
3. So every later frame also hashes to link 1.
4. Link 1 is 10 Gbit/s, so the flow tops out at about 10 Gbit/s.
5. Link 2 could be empty, and this flow would still not use it.

```text
         2 x 10 Gbit/s bundle
       +----------------------+
Flow A | ===== link 1 ======> |  Flow A alone: 10 Gbit/s maximum
       |                      |
       |       link 2 (idle)  |  Spare capacity flow A cannot reach
       +----------------------+

       +----------------------+
Flow A | ===== link 1 ======> |  Many flows: close to 20 Gbit/s
Flow B | ===== link 2 ======> |  in total
Flow C | ===== link 1 ======> |  A and C share link 1
Flow D | ===== link 2 ======> |  B and D share link 2
       +----------------------+
```

So 2 x 10 Gbit/s is not the same as 1 x 20 Gbit/s. A bundle helps many flows. It does not help one big flow, such as a single backup job over one TCP connection. To speed that up, use a faster link or split the job into many connections.

**Fuller truth:** Linux `balance-rr` mode sends frames round-robin, so one flow can pass the speed of one link. The cost is reordering, which is why it is rarely used. Separately, a single 40 or 100 Gbit/s port is not a LAG, even though it often has 4 lanes inside. It splits data across lanes below the Ethernet frame level, so one flow can use the full port speed.

### Three ways to form a bundle

| | Static (`on`) | PAgP | LACP |
|---|---|---|---|
| Full name | None, manual only | Port Aggregation Protocol | Link Aggregation Control Protocol |
| Owner | No protocol | Cisco proprietary | IEEE 802.1AX (first published as 802.3ad) |
| Sends control frames | No | Yes | Yes |
| Modes | `on` | `desirable`, `auto` | `active`, `passive` |
| Works between vendors | Yes, if both sides are set by hand | No | Yes |
| Detects a miswired port | No | Yes | Yes |
| Detects a silent failure | No | Yes | Yes |
| Hot-standby links | No | No | Yes |
| Use today | Only if a device has no LACP | Legacy networks only | The default choice |

Use LACP unless you have a clear reason not to.

Why is static risky? With static mode, a port joins the bundle as soon as its link is up. Nothing checks the other end. Suppose a media converter sits in the middle of one link. The far side dies, but the near side still shows link up. The switch keeps hashing traffic onto a dead path, and part of your traffic vanishes with no error. LACP notices this, because control frames stop arriving.

### Active and passive modes

- **Active** means "I start the talk". The port sends LACP frames by itself.
- **Passive** means "I only answer". The port stays quiet until it hears LACP frames.

At least one side must start.

| Side A | Side B | Bundle forms? |
|--------|--------|---------------|
| LACP active | LACP active | Yes (recommended) |
| LACP active | LACP passive | Yes |
| LACP passive | LACP passive | No, both wait forever |
| PAgP desirable | PAgP desirable | Yes |
| PAgP desirable | PAgP auto | Yes |
| PAgP auto | PAgP auto | No, both wait forever |
| on | on | Yes |
| on | LACP active | No, and the ports may misbehave |
| LACP | PAgP | No, they are different protocols |

Think of two shy people at a party. Passive-passive is two people each waiting for the other to say hello. **Where the analogy stops being true:** People eventually give up and try something new. Passive ports wait forever.

### How LACP forms a bundle

Each LACP frame is called an **LACPDU** (LACP Data Unit, a small control message). In each one, a port says two things: facts about itself (it is the **Actor**) and what it has heard about the other side (the **Partner**).

Three values identify everything:

- **System ID**: names the device. It is a priority (default 32768 on Cisco) plus a MAC address.
- **Key**: names a group of ports that can bundle together. On Cisco it follows the channel-group number.
- **Port ID**: names one port. It is a priority (default 32768 on Cisco) plus a port number.

The steps are:

1. A link comes up. An active port sends an LACPDU with its system ID, key and port ID.
2. The other port stores these as its Partner data and replies, repeating what it heard.
3. Each side now knows that the other can hear it. This is a two-way check.
4. Each device groups the ports that have the same local key and the same partner system ID and key.
5. If there are too many ports, one device picks the active ones (see below).
6. Chosen ports set their state bits to Synchronized, Collecting and Distributing. Data now flows.
7. LACPDUs keep flowing as a keepalive.

```text
SW1 Gi1/0/1 (active)                     SW2 Gi1/0/1 (active)
      |                                          |
      |--- LACPDU: Actor=SW1, Partner=unknown -->|
      |<-- LACPDU: Actor=SW2, Partner=SW1 -------|
      |--- LACPDU: Actor=SW1, Partner=SW2 ------>|
      |                                          |
      |    Both agree. Port joins the bundle.    |
      |                                          |
      |<========= data frames both ways ========>|
      |<-------- LACPDU every 1 or 30 s -------->|
```

**Timers.** LACP has two rates:

| Rate | LACPDU every | Link declared dead after |
|------|--------------|--------------------------|
| Slow (long timeout) | 30 seconds | 90 seconds (3 missed) |
| Fast (short timeout) | 1 second | 3 seconds (3 missed) |

Slow is the usual default on Cisco and Linux. These timers matter for silent failures only. If a port loses its physical link, the device removes it from the bundle right away and does not wait for a timer.

**Miswire protection.** Suppose Gi1/0/1 goes to SW2 but Gi1/0/2 goes to SW3 by mistake. The two ports report different partner system IDs, so LACP will not bundle them together.

### Hot standby and the 8-link limit

Catalyst switches allow up to 8 active links in one EtherChannel. With LACP you can configure up to 16: 8 active and 8 in hot-standby. A standby link carries no data and takes over if an active link fails. Limits vary. For example, Nexus 9000 switches allow up to 32 active links.

Who picks the active links? The device with the lower system ID decides. It prefers ports with the lower port ID. Use `lacp system-priority` and `lacp port-priority` to control this. Lower numbers win.

### Requirements for matching ports

All members of a bundle must look the same. Otherwise a switch cannot treat them as one port. On Cisco switches, check these:

| Must match | Example of a mismatch |
|------------|-----------------------|
| Speed | One 1 Gbit/s port and one 100 Mbit/s port |
| Duplex | LACP needs full duplex |
| Port mode | One access port and one trunk port |
| Access VLAN | VLAN 10 and VLAN 20 |
| Native VLAN and allowed VLANs on trunks | 10,20 and 10,20,30 |
| Layer 2 or Layer 3 mode | One `switchport`, one `no switchport` |
| STP settings on the port | Different port cost |
| MTU | 1500 on one, 9216 on the other |
| Protocol and channel-group number | Part LACP and part PAgP in one group |

Also note:

- A SPAN destination port (a port that receives a copy of traffic for monitoring) cannot join a bundle.
- Ports do not need to be next to each other. They can be on different line cards or stack members, which gives better protection.
- All links must end on one logical device. Bundling to two independent switches needs MLAG (see below).
- Some security and other per-port features limit bundling, and this varies by platform.

**Fuller truth:** The IEEE standard groups ports by key and leaves the rule about mixed speeds open. In practice, Cisco switches and the Linux driver only bundle ports of the same speed.

### Linux bonding modes

Linux offers seven modes. The important question is whether the switch must be configured too.

| Mode | Name | How it works | Switch config needed |
|------|------|--------------|----------------------|
| 0 | `balance-rr` | Round-robin, one frame per link | Static bundle |
| 1 | `active-backup` | One link works, the other waits | None |
| 2 | `balance-xor` | Hash picks the link | Static bundle |
| 3 | `broadcast` | Sends every frame on all links | Static bundle |
| 4 | `802.3ad` | LACP and hash | LACP bundle |
| 5 | `balance-tlb` | Sending shared by load, receiving on one link | None |
| 6 | `balance-alb` | Like mode 5, plus receive sharing using ARP replies for IPv4 | None |

Use mode 4 when the switch supports LACP. Use mode 1 when you want simple failover with no switch work.

The Linux hash is set by `xmit_hash_policy`:

| Policy | Fields used |
|--------|-------------|
| `layer2` (default) | MAC addresses |
| `layer2+3` | MAC and IP addresses |
| `layer3+4` | IP addresses and TCP/UDP ports |

The kernel docs warn that `layer3+4` is not fully 802.3ad compliant, because fragmented packets can take a different path from unfragmented ones. In practice it is widely used and works well for normal TCP traffic.

### Windows Server teaming

The built-in teaming, called LBFO (Load Balancing and Failover), is a Windows Server feature.

```powershell
# Windows Server
New-NetLbfoTeam -Name "Team1" -TeamMembers "Ethernet1","Ethernet2" -TeamingMode Lacp -LoadBalancingAlgorithm Dynamic
Get-NetLbfoTeam
Get-NetLbfoTeamMember
```

`-TeamingMode` can be `Lacp`, `Static` or `SwitchIndependent`. For Hyper-V hosts, Microsoft recommends Switch Embedded Teaming (SET), which is switch-independent and does not use LACP.

### MLAG: one bundle to two switches

A normal bundle has a weakness: all its links end on one switch. If that switch dies, the bundle dies.

**MLAG** (Multi-chassis Link Aggregation) fixes this. Two separate switches cooperate and act like one switch towards the device below. That device runs normal LACP and has no idea there are two switches.

```text
          +---------+  peer-link   +---------+
          | Switch1 |==============| Switch2 |
          |         |--keepalive---|         |
          +----+----+              +----+----+
               |                        |
               |   LACP sees one        |
               |   system ID            |
               +-----+            +-----+
                     |            |
                 +---+------------+---+
                 |   Server: bond0    |
                 +--------------------+
```

Each switch keeps its own control plane (the software brain that runs protocols) and its own configuration. Both forward traffic at the same time. Both send LACP frames with the same shared system ID.

The analogy is two receptionists sharing one phone number. Callers think there is one office. The receptionists keep each other updated on an internal line. **Where the analogy stops being true:** If the internal line breaks, the two receptionists can give conflicting answers. In a network this is called **split-brain**, and it needs a planned response.

MLAG has no common industry form, and each vendor has its own design. Peers must normally be the same vendor. Common names are Cisco vPC, Arista MLAG, Juniper MC-LAG, NVIDIA Cumulus MLAG, HPE Aruba VSX and Dell VLT. IEEE 802.1AX does define a related method (Distributed Resilient Network Interconnect, DRNI), but vendor designs are what you will mostly meet.

### vPC: Cisco's MLAG on Nexus

vPC (Virtual Port Channel) is the Nexus version of MLAG. Learn these terms:

| Term | Meaning |
|------|---------|
| vPC domain | The pair of peer switches, with one domain number |
| vPC peer | One of the two switches |
| Peer-link | Port-channel between the peers. It syncs state and carries traffic when needed |
| Peer-keepalive | A separate Layer 3 path used only to ask "are you alive?" |
| Member port | A port in a vPC, towards the downstream device |
| Orphan port | A port connected to one peer only |
| Primary, secondary | Roles. Lower role priority becomes primary |

Key behaviour:

- **Traffic stays local.** A switch sends traffic out its own links when it can. The peer-link is used mainly during failures and for orphan ports.
- **Loop rule.** A frame that crossed the peer-link is not sent out a vPC member port, unless the matching port on the other peer is down.
- **Peer-link fails, keepalive still up.** The secondary suspends its vPC member ports and SVIs (the switch's Layer 3 VLAN interfaces). This avoids split-brain, and traffic continues through the primary.
- **Both peer-link and keepalive fail.** Each peer thinks the other is dead, and both act as primary. This is the dangerous dual-active case, which is why those two paths should not share a failure point.
- **Consistency checks.** Peers compare config. A Type 1 mismatch can suspend a vPC. A Type 2 mismatch gives a warning.

Basic configuration on the first peer. The second peer mirrors it, with the keepalive addresses swapped.

```text
! Cisco NX-OS, on N9K-1
feature lacp
feature vpc

vpc domain 10
  role priority 100
  peer-keepalive destination 192.168.100.2 source 192.168.100.1 vrf management
  peer-gateway

interface port-channel1
  switchport mode trunk
  vpc peer-link

interface Ethernet1/49-50
  switchport mode trunk
  channel-group 1 mode active

interface port-channel20
  switchport mode access
  switchport access vlan 20
  vpc 20

interface Ethernet1/3
  switchport mode access
  switchport access vlan 20
  channel-group 20 mode active
```

Verify with `show vpc brief` and `show vpc consistency-parameters global`.

```text
N9K-1# show vpc brief
vPC domain id                     : 10
Peer status                       : peer adjacency formed ok
vPC keep-alive status             : peer is alive
Configuration consistency status  : success
vPC role                          : primary
Number of vPCs configured         : 1

vPC Peer-link status
---------------------------------------------------------
id    Port   Status Active vlans
--    ----   ------ -------------------------------------
1     Po1    up     1,20

vPC status
---------------------------------------------------------
Id    Port     Status Consistency Reason     Active vlans
--    ------   ------ ----------- ------     ------------
20    Po20     up     success     success    20
```

### Stacking

**Stacking** joins several physical switches with special cables so that they behave as one switch. They share one control plane, one configuration file and one management IP address. Cisco calls this StackWise on Catalyst 9300 and similar models. Catalyst 9300 stacks can have up to 8 members.

One member is the **active** and runs everything. Another is the **standby** and takes over if the active fails. The others just forward traffic.

A bundle with members on different stack switches is called a **cross-stack EtherChannel**. To the device below, this looks like a normal bundle, and it survives a switch failure. Use LACP for this. Do not assume PAgP works across members, as support varies.

```text
Stack (one brain)              MLAG (two brains)

+---------------+              +-----------+    +-----------+
| Member 1      |              | Switch 1  |====| Switch 2  |
+---stack-cable-+              | own brain |    | own brain |
| Member 2      |              +-----+-----+    +-----+-----+
+---------------+                    |                |
   |         |                       +-------+  +-----+
   +---+  +--+                               |  |
       |  |                               +--+--+--+
    +--+--+--+                            | Server |
    | Server |                            +--------+
    +--------+
```

The analogy: a stack is one manager with several staff. MLAG is two equal managers who coordinate by phone. **Where the analogy stops being true:** A failed stack active does not leave the team leaderless for long, because the standby takes over quickly.

There is also a middle case. **VSS** (Virtual Switching System, on older Catalyst 6500 and 4500 models) and **StackWise Virtual** (on models such as Catalyst 9400, 9500 and 9600) join two chassis over normal Ethernet links into one logical switch. They are one brain, like a stack, but they use Ethernet instead of stack cables.

| | Stack | VSS, StackWise Virtual | MLAG, vPC |
|---|---|---|---|
| Control planes | One | One | Two |
| Management | One IP, one config | One IP, one config | Two of each |
| Link between switches | Special stack cables | Ethernet | Ethernet peer-link plus keepalive |
| Software upgrade | Often all members reload | In-service upgrade possible | One peer at a time |
| Main risk | A software fault hits all members | A software fault hits both chassis | Config drift, split-brain |
| Typical place | Access closet | Campus core or distribution | Data centre server racks |
| Device below sees | One switch | One switch | One switch |

## Example

### Example 1: Two Catalyst switches, 2 x 1 Gbit/s LACP trunk

```text
       VLANs 10,20 over Port-channel1

  +-----+ Gi1/0/1 ---------- Gi1/0/1 +-----+
  | SW1 |                            | SW2 |
  +-----+ Gi1/0/2 ---------- Gi1/0/2 +-----+
```

Configure SW1. SW2 is identical.

```text
! Cisco IOS XE
SW1(config)# interface range GigabitEthernet1/0/1 - 2
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# channel-group 1 mode active
Creating a port-channel interface Port-channel 1
SW1(config-if-range)# exit
SW1(config)# interface Port-channel1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# exit
SW1(config)# port-channel load-balance src-dst-ip
```

Check the bundle. `SU` on the port-channel means Layer 2 and in use. `P` on each member means bundled.

```text
SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-------------------------
1      Po1(SU)         LACP      Gi1/0/1(P)  Gi1/0/2(P)
```

Check the neighbour. The Dev ID is SW2's system MAC, and both ports see the same one.

```text
SW1# show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode
        P - Device is in Passive mode

Channel group 1 neighbors

Port     Flags  Priority  Dev ID          Age  Key  Port   State
Gi1/0/1  SA     32768     00d7.8f12.3400  12s  0x1  0x102  0x3D
Gi1/0/2  SA     32768     00d7.8f12.3400  12s  0x1  0x103  0x3D
```

(The columns are shortened here to fit. Real output also shows the admin key.)

The state `0x3D` is `0011 1101` in binary. Read from the lowest bit, it means Activity on, Timeout slow, Aggregation yes, Synchronized, Collecting and Distributing. That is a healthy slow-rate port. A healthy fast-rate port shows `0x3F`.

Check the hash and total bandwidth:

```text
SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-dst-ip

EtherChannel Load-Balancing Addresses Used Per-Protocol:
Non-IP: Source XOR Destination MAC address
  IPv4: Source XOR Destination IP address
  IPv6: Source XOR Destination IP address

SW1# show interfaces Port-channel1 | include BW|Members
  MTU 1500 bytes, BW 2000000 Kbit/sec, DLY 10 usec,
  Members in this channel: Gi1/0/1 Gi1/0/2
```

Ask the switch which link a flow would use:

```text
SW1# test etherchannel load-balance interface port-channel 1 ip 10.1.10.11 10.1.20.5
Would select Gi1/0/2 of Po1
```

Now simulate a failure on Gi1/0/2:

```text
SW1(config)# interface GigabitEthernet1/0/2
SW1(config-if)# shutdown
SW1(config-if)# end

SW1# show etherchannel summary | begin Group
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-------------------------
1      Po1(SU)         LACP      Gi1/0/1(P)  Gi1/0/2(D)

SW1# show interfaces Port-channel1 | include BW
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
```

`Po1` stays up, so STP sees no link failure. Flows that used Gi1/0/2 now hash to Gi1/0/1. Capacity falls to 1 Gbit/s.

### Example 2: Linux server with 2 x 10 Gbit/s LACP

```text
  +----------------------+
  | Server 10.20.0.50/24 |          +--------------------+
  |  bond0               |          | SW3  VLAN 20       |
  |    eno1 ------------------------- Te1/0/1            |
  |    eno2 ------------------------- Te1/0/2            |
  +----------------------+          | Gateway 10.20.0.1  |
                                    +--------------------+
```

Switch side:

```text
! Cisco IOS XE
interface range TenGigabitEthernet1/0/1 - 2
 switchport mode access
 switchport access vlan 20
 channel-group 10 mode active
 lacp rate fast
!
interface Port-channel10
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
```

Linux side, with `ip` commands. This is lost at reboot, so it suits a quick test.

```bash
# Linux, iproute2
sudo ip link add bond0 type bond mode 802.3ad miimon 100 \
  lacp_rate fast xmit_hash_policy layer3+4
sudo ip link set eno1 down
sudo ip link set eno2 down
sudo ip link set eno1 master bond0
sudo ip link set eno2 master bond0
sudo ip link set bond0 up
sudo ip addr add 10.20.0.50/24 dev bond0
sudo ip route add default via 10.20.0.1
```

`miimon 100` checks link state every 100 ms.

For a permanent setup on Ubuntu, use netplan (`/etc/netplan/10-bond.yaml`):

```text
network:
  version: 2
  ethernets:
    eno1: {}
    eno2: {}
  bonds:
    bond0:
      interfaces: [eno1, eno2]
      addresses: [10.20.0.50/24]
      routes:
        - to: default
          via: 10.20.0.1
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
        transmit-hash-policy: layer3+4
```

On RHEL and similar systems, use NetworkManager:

```bash
# Linux, NetworkManager
sudo nmcli con add type bond con-name bond0 ifname bond0 \
  bond.options "mode=802.3ad,miimon=100,lacp_rate=fast,xmit_hash_policy=layer3+4"
sudo nmcli con add type ethernet con-name bond0-p1 ifname eno1 master bond0
sudo nmcli con add type ethernet con-name bond0-p2 ifname eno2 master bond0
sudo nmcli con mod bond0 ipv4.method manual \
  ipv4.addresses 10.20.0.50/24 ipv4.gateway 10.20.0.1
sudo nmcli con up bond0
```

Check the bond:

```bash
cat /proc/net/bonding/bond0
```

```text
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100

802.3ad info
LACP rate: fast
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 2
        Actor Key: 15
        Partner Key: 10
        Partner Mac Address: 00:d7:8f:12:56:00

Slave Interface: eno1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Aggregator ID: 1

Slave Interface: eno2
MII Status: up
Speed: 10000 Mbps
Duplex: full
Aggregator ID: 1
```

(This output is trimmed.) The partner key `10` matches channel-group 10. If `Partner Mac Address` shows all zeros, the switch is not talking LACP.

The bond reports the total speed:

```bash
$ ethtool bond0 | grep Speed
        Speed: 20000Mb/s
```

Now prove that one flow does not get 20 Gbit/s. Host 10.20.0.60 runs `iperf3 -s` and has a 25 Gbit/s link. See [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

```bash
# One TCP stream
$ iperf3 -c 10.20.0.60 -t 10
[ ID] Interval           Transfer     Bitrate
[  5]  0.00-10.00 sec    11.0 GBytes  9.41 Gbits/sec   sender

# Eight parallel TCP streams
$ iperf3 -c 10.20.0.60 -t 10 -P 8
[ ID] Interval           Transfer     Bitrate
[SUM]  0.00-10.00 sec    21.4 GBytes  18.4 Gbits/sec   sender
```

One stream stops near the speed of one link. Eight streams have different source ports, so `layer3+4` spreads them across both links. The split may not be even. For example, 5 streams can land on one link and 3 on the other.

### Example 3: A port that will not join

You change the allowed VLANs on one member of SW2, not on the port-channel. This appears:

```text
%EC-5-CANNOT_BUNDLE2: Gi1/0/2 is not compatible with Gi1/0/1
and will be suspended (vlan mask is different)
```

```text
SW2# show etherchannel summary | begin Group
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-------------------------
1      Po1(SU)         LACP      Gi1/0/1(P)  Gi1/0/2(s)
```

The `s` means suspended. The fix is to make the member match, then bounce the port.

```text
SW2(config)# interface GigabitEthernet1/0/2
SW2(config-if)# switchport trunk allowed vlan 10,20
SW2(config-if)# shutdown
SW2(config-if)# no shutdown
```

The habit that avoids this: after a bundle exists, change settings on `Port-channel1`. Members inherit them.

## Going Deeper

### Inside an LACPDU

An LACPDU is a slow-protocol frame. Its destination MAC is `01:80:C2:00:00:02`, and its EtherType is `0x8809`. Switches do not forward this address, so LACP only reaches the direct neighbour.

```text
+-------------------+-------+----------------------------------+
| Field             | Bytes | Meaning                          |
+-------------------+-------+----------------------------------+
| Subtype           | 1     | 1 means LACP                     |
| Version           | 1     | Protocol version                 |
| Actor info        | 20    | Facts about the sender           |
| Partner info      | 20    | What the sender heard about you  |
| Collector info    | 16    | Collector timing value           |
| Terminator, pad   | 52    | End marker and padding           |
+-------------------+-------+----------------------------------+

Actor info and Partner info carry the same fields:

+-----------------------+-------+
| Field                 | Bytes |
+-----------------------+-------+
| Type and length       | 2     |
| System priority       | 2     |
| System MAC            | 6     |
| Key                   | 2     |
| Port priority         | 2     |
| Port number           | 2     |
| State                 | 1     |
| Reserved              | 3     |
+-----------------------+-------+
```

The State byte is the first thing to read in a capture:

```text
Bit  Name             When set to 1
0    Activity         Port is in active mode
1    Timeout          Short timeout (fast rate)
2    Aggregation      Port can be bundled
3    Synchronization  Port is in the right group
4    Collecting       Port accepts incoming data
5    Distributing     Port sends data
6    Defaulted        Using default data: no partner heard
7    Expired          Partner data has timed out
```

Note the Timeout bit carefully. It says how fast I want the *partner* to send to me. So `lacp rate fast` on one side controls how often the *other* side sends LACPDUs. Linux docs describe `lacp_rate` the same way.

### Why 3 links can be uneven

Many Catalyst platforms hash into 8 buckets, then assign buckets to links. With 2, 4 or 8 links, buckets divide evenly. With other counts they do not.

```text
Links   Buckets per link     Share of flows per link
2       4 4                  50%   50%
3       3 3 2                37.5% 37.5% 25%
4       2 2 2 2              25% each
5       2 2 2 1 1            25% 25% 25% 12.5% 12.5%
6       2 2 1 1 1 1          25% 25% 12.5% x 4
7       2 1 1 1 1 1 1        25% then 12.5% x 6
8       1 1 1 1 1 1 1 1      12.5% each
```

That is why engineers like 2, 4 or 8 links. Shares here count flows, not bytes. One large flow can still make a link much busier. Some newer platforms use more buckets, so this varies.

### Hash polarisation

Picture two layers of switches, each with a 2-link bundle. Layer 1 sends all flows with hash result 0 to link A. Those flows reach layer 2. If layer 2 uses the same hash, the same fields and the same seed, those flows all give result 0 again. So one link of layer 2 gets everything, and the other gets nothing.

This is called **hash polarisation**. To avoid it, use different hash inputs at each layer, or use a platform feature that changes the hash seed per device. The same issue affects equal-cost routing. See [Advanced Routing Techniques](../05-routing/12-advanced-routing-techniques.md).

### Tunnels hide flows

Hashing only sees the outer headers on many platforms. Thousands of flows inside one GRE or IPsec tunnel have one outer source and destination. They look like one flow, so they use one link.

VXLAN handles this well. It changes the outer UDP source port based on the inner frame, which gives the hash something to work with. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

### min-links

Say a bundle of 4 x 10 Gbit/s normally carries 25 Gbit/s. If 3 links die, one link is badly overloaded but the port-channel still looks up. Routers keep using it.

`port-channel min-links 2` brings the whole port-channel down if fewer than 2 links remain. Then routing moves traffic to a healthier path. On Linux the matching option is `min_links`.

### What happens when a member changes

When a link is added or removed, buckets are reassigned. Flows on the failed link move to survivors. Some healthy flows may also move, depending on the platform. A moved flow can see brief reordering or loss. TCP recovers, but a very sensitive app might notice.

### When LACP gets no reply

What a port does when it hears no LACPDUs varies:

- Catalyst IOS XE usually leaves it as stand-alone (flag `I`), working as a normal port.
- NX-OS suspends it by default. `no lacp suspend-individual` allows it to act alone.

This matters for servers doing PXE boot (starting from the network). The boot firmware usually has no LACP, so the suspended port stays dead and the install fails. Vendors offer a fix, often named **LACP fallback**, that lets one port work until LACP starts.

### Guard against static mistakes

If one side is `on` and the other is not bundled, a loop can form. Cisco **EtherChannel guard** detects this and error-disables the ports with reason `channel-misconfig`. The command is `spanning-tree etherchannel guard misconfig`, and it is commonly on by default on Catalyst.

### LAG, ECMP and modern data centres

A LAG balances at Layer 2 across parallel links between two devices. **ECMP** (Equal-Cost Multi-Path) balances at Layer 3 across routed paths, and these can go through different next hops. Both hash flows, and both have the "one flow, one path" rule.

[Spine-leaf](../10-enterprise-and-datacenter/02-spine-leaf.md) networks use ECMP between switches and keep LAGs mainly for server connections. For those, **EVPN multihoming** (often called ESI-LAG) is a modern alternative to MLAG. It uses EVPN to coordinate the switches, and it needs no peer-link.

### Routed bundles

A port-channel can be a Layer 3 interface with its own IP address.

```text
! Cisco IOS XE
interface range GigabitEthernet1/0/1 - 2
 no switchport
 channel-group 2 mode active
!
interface Port-channel2
 no switchport
 ip address 10.0.0.1 255.255.255.252
```

Here, STP is not involved, and routing protocols see one interface.

### vPC details worth knowing

- Keep STP on even with vPC, as a safety net against cabling mistakes.
- Both vPC peers can forward traffic sent to the HSRP virtual MAC, so first-hop redundancy works active-active. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).
- Forming routing protocol neighbours across a vPC needs care. Older NX-OS releases did not support it for all cases. Newer ones have `layer3 peer-router`. Check the docs for your release.

## Common Mistakes

- **"Two 10G links give me a 20G link."** They give 20G in total. One flow gets at most 10G.
- **"LACP balances the traffic."** LACP only forms and watches the bundle. The hash does the balancing, and static bundles use a hash too.
- **"Traffic balances by how busy each link is."** Normal hashing ignores load. Two large flows can share a link while another is idle.
- **Using MAC-based hashing between two routers.** There is only one MAC pair, so one link carries everything.
- **Setting both sides to passive, or both to auto.** Neither side starts, so no bundle forms. Use active on both sides.
- **Mixing `on` with LACP or PAgP.** `on` sends no control frames. The result can be a failed bundle, a loop or dropped traffic.
- **Changing config on a member port.** The change makes it inconsistent, and it gets suspended. Configure the port-channel interface instead.
- **Trusting "link up" with static bundles.** Static mode cannot see a failure beyond the local link.
- **Expecting the same hash on both ends to be required.** Each side picks independently, for its own sent traffic.
- **Using 3, 5, 6 or 7 links without checking.** On many platforms, some links get twice the share of others.
- **Mixing up "trunk" meanings.** On Cisco, a trunk carries many VLANs. On some vendors, a trunk means a LAG.
- **Thinking a port-channel means full redundancy.** With all links on one switch, that switch is still a single point of failure. Use a stack or MLAG for device redundancy.
- **Making the peer-link too small.** After a failure, much of the traffic may cross it.
- **Running peer-link and keepalive over the same path.** One failure then takes out both, which can cause dual-active.
- **Bonding at the server, with no matching switch config.** Mode 4 needs LACP on the switch, and modes 0 and 2 need a static bundle. Without this, you will see flapping MACs or unstable traffic.

## Interview Questions

**Q: What is link aggregation and why use it?**
A: It combines several physical links into one logical link. It gives more total bandwidth and redundancy, and because STP sees one link, nothing is blocked. Cisco calls it EtherChannel, and Linux calls it bonding.

**Q: I have two 1 Gbit/s links in a port-channel. Can one file transfer reach 2 Gbit/s?**
A: Normally no. The switch hashes fields such as IPs and ports to pick a link. All frames in one TCP connection have the same fields, so they use one link, which keeps them in order. The limit is 1 Gbit/s per flow. Many flows together can use nearly 2 Gbit/s.

**Q: What is the difference between LACP, PAgP and static mode?**
A: LACP is the IEEE open standard, 802.1AX, and it works between vendors. PAgP does a similar job but is Cisco proprietary and is now legacy. Static mode, `on` in Cisco, uses no protocol, so it cannot detect miswiring or silent failures. I would use LACP by default.

**Q: Explain LACP active and passive.**
A: Active ports send LACPDUs without waiting. Passive ports only reply. Active-active and active-passive form a bundle. Passive-passive does not, because nobody starts. I set both sides to active.

**Q: What must match for ports to join a bundle?**
A: Speed, duplex, access or trunk mode, the access VLAN or the native and allowed VLANs, Layer 2 or Layer 3 mode, STP port settings and MTU. If a member does not match, the switch suspends it. I fix this by configuring the port-channel interface, so members inherit the settings.

**Q: Traffic between a router and a firewall crosses a 4-link bundle, but only one link is busy. Why?**
A: The hash is probably using MAC addresses. Between two devices, every frame has the same source and destination MAC, so the result never changes. I would change to a hash with IPs and ports, such as `src-dst-mixed-ip-port`, and check with `show etherchannel load-balance` and `test etherchannel load-balance`.

**Q: Why does a port-channel not cause a loop?**
A: The switch treats the bundle as one port. It sends a flooded frame out one member only, and it never sends a frame back into the bundle it came from. STP also sees just one logical interface.

**Q: What happens when one member link fails?**
A: The port-channel stays up with less capacity. Flows on the failed link are rehashed onto the other members. STP sees no change. If the failure is silent, LACP finds it when LACPDUs stop: after 3 seconds at fast rate or 90 seconds at slow rate.

**Q: What is the difference between stacking and MLAG?**
A: A stack has one control plane, one config and one management IP, so it is simple to run, but a software fault affects all members. MLAG keeps two independent switches, so you can upgrade one at a time, but you manage two configs and must plan for split-brain. Both let a device use LACP across two physical switches.

**Q: In vPC, what is the difference between the peer-link and the peer-keepalive?**
A: The peer-link is a port-channel that syncs state and can carry data traffic. The keepalive is a separate Layer 3 path that only checks if the peer is alive. If the peer-link fails but keepalive works, the secondary suspends its vPC ports to avoid split-brain.

**Q: Why are 2, 4 or 8 links preferred in a bundle?**
A: Many platforms hash into 8 buckets. These numbers divide the buckets evenly. With 3 links, the split is 3, 3 and 2 buckets, so one link gets less. Newer platforms may have more buckets, so I would check the platform.

**Q: A Linux server in bond mode 4 shows all zeros for the partner MAC. What does that mean?**
A: The server sends LACPDUs but hears nothing back. The switch ports are probably not configured for LACP, or they are in a different mode. I would check the switch with `show etherchannel summary` and `show lacp neighbor`, and the server with `/proc/net/bonding/bond0`.

## Quick Recap

- Link aggregation makes several physical links act as one logical link, for more total capacity and redundancy, with no STP-blocked links.
- A hash on fixed frame fields picks one link for each flow. This keeps frames in order, but one flow is limited to one link's speed.
- LACP (IEEE 802.1AX) is the default. PAgP is Cisco legacy. Static `on` has no checks. At least one side must be active or desirable.
- Member ports must match in speed, duplex, VLANs, mode, STP settings and MTU. Make changes on the port-channel interface.
- Choose a hash that fits your traffic. MAC hashing fails between two routers. IP and port hashing spreads best.
- Linux mode 4 (`802.3ad`) needs LACP on the switch. Mode 1 (`active-backup`) needs no switch config.
- Stacks and VSS use one brain. MLAG and vPC use two cooperating switches. All let one LACP bundle span two physical switches.
