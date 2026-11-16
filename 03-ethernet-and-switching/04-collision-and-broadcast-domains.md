# Collision and Broadcast Domains

**Level:** Intermediate | **Prerequisites:** [Network Devices](../01-foundations/04-network-devices.md), [Ethernet Frames and Standards](02-ethernet-frames.md), [How Switches Work](03-how-switches-work.md)

## In One Line

A collision domain is the group of devices whose signals can crash into each other. A broadcast domain is the group of devices that all receive one broadcast. Hubs split neither, switches split collision domains, and routers split both.

## Why It Matters

These two ideas explain why networks are built the way they are.

Early Ethernet put every computer on one shared cable. Only one computer could talk at a time. As more computers joined, they spent more time crashing into each other. Switches solved that problem.

Switches created a second problem. A switch still sends every broadcast to all of its ports. If you join hundreds of switches, one small mistake can fill the whole network with copies of the same message. Routers and VLANs solve that problem.

Interviewers ask about this for three reasons:

- It tests whether you know what each device really does at each layer.
- The "count the domains in this diagram" question is a classic. It has one right number and many traps.
- Broadcast storms are a real outage. A candidate who can explain one, and stop one, sounds experienced.

## Simple Explanation

**Collision domain: one walkie-talkie channel.** Imagine a group of people sharing one walkie-talkie channel. Only one person can speak at a time. If two people press the button together, both voices mix and nobody understands. Both must stop and try again. Everyone on that channel is in the same collision domain.

Where the analogy stops being true: people choose when to wait. Ethernet devices follow a fixed rule with a random timer. Also, modern Ethernet is not one channel. It is more like a private phone call, where both sides can speak at once.

**Broadcast domain: one loudspeaker system.** Imagine a school with a loudspeaker in every room. When the office makes an announcement, every room hears it, even if the message is for one student. Every room reached by that announcement is in one broadcast domain. A second school has its own system, so it hears nothing. The wall between the two schools is the router.

Where the analogy stops being true: a person can ignore an announcement for free. A computer must spend some CPU time to look at every broadcast, even one that does not concern it.

**The three devices:**

| Device | Everyday picture |
|---|---|
| Hub | One room. Everyone hears everything, and one person speaks at a time. |
| Switch | An operator who connects private calls, but who repeats every public announcement to everyone. |
| Router | A guarded door between two buildings. Announcements stay inside. Only addressed mail passes. |

## How It Works

### Two definitions

**Frame:** the unit of data that Ethernet sends. Details are in [Ethernet Frames and Standards](02-ethernet-frames.md).

**Collision:** two devices send on the same shared medium at the same time. Their electrical signals overlap and both frames are destroyed.

**Collision domain:** a part of the network where a collision can happen, so that only one device can send at a time. A device that sends in this part blocks all the others in this part.

**Broadcast:** a frame sent to the special destination MAC address `ff:ff:ff:ff:ff:ff`, which means "everyone here". A MAC address is the hardware address of a network card. See [MAC Addresses](01-mac-addresses.md).

**Broadcast domain:** the set of devices that receive a broadcast sent by any one of them. The edge of a broadcast domain is a router, or a VLAN boundary (explained below).

Simple test:

- If two devices send at the same moment, do their frames crash? If yes, they share a collision domain.
- If one device sends a broadcast, does the other receive it? If yes, they share a broadcast domain.

### How collisions happen and how Ethernet handled them

Half duplex means a device can send or receive, but not both at once. Full duplex means it can do both at once. Collisions can only happen in half duplex.

Half-duplex Ethernet uses a rule called **CSMA/CD** (Carrier Sense Multiple Access with Collision Detection). The name means: listen first, many devices share the medium, and notice crashes.

1. The device listens to the wire. Is anyone sending?
2. If the wire is quiet, it starts to send.
3. While it sends, it keeps watching the wire for a collision.
4. If it detects a collision, it stops the frame and sends a short jam signal, so that every device notices.
5. It waits for a random time. This is called backoff.
6. It tries again. After 16 failed attempts it gives up and drops the frame.

Why can two devices send at once if both listen first? Because a signal takes time to travel. PC1 starts sending. Before the signal reaches PC2, the wire still looks quiet to PC2, so PC2 starts too.

```text
  Time 1:  PC1 sends -->                       <-- PC2 sends
  Time 2:           ---->               <----
  Time 3:                  ---->XXX<----
                           collision here
  Time 4:  Both see the crash, send jam, wait a random time
```

CSMA/CD is not needed on full-duplex links. The fuller truth, with timing numbers, is in Going Deeper.

### Hub: one shared wire in a box

A hub is a Layer 1 device. It understands electrical signals, not addresses. A signal that enters one port is repeated out of all other ports. If two ports send at once, the signals collide.

- All hub ports are in **one** collision domain.
- All hub ports are in **one** broadcast domain.

A 100 Mb/s hub with 8 PCs gives 100 Mb/s shared by all 8, and less in real use because collisions waste time. Hubs are obsolete, but they stay in exam diagrams because they make the point clearly.

```text
  PC1 sends a frame to PC3. The hub copies it everywhere.

  PC1 ====> +-----+ ====> PC2   (not for PC2, ignored)
            | HUB | ====> PC3   (the real target)
            +-----+ ====> PC4   (not for PC4, ignored)

  One collision domain. One broadcast domain.
```

### Switch: one collision domain per port

A switch is a Layer 2 device. It reads the destination MAC address and sends each frame only out of the correct port (see [How Switches Work](03-how-switches-work.md)). It also keeps each port separate, so a signal on one port never reaches another port as a raw signal.

- **Every switch port is its own collision domain.**
- All ports are still in **one** broadcast domain, unless you use VLANs.

Why one broadcast domain? When a switch receives a broadcast, it floods it. To flood means to copy the frame out of every port in the same VLAN except the port it came from.

```text
  PC1 sends a broadcast. The switch floods it.

  PC1 ====> +--------+ ====> PC2
            | SWITCH | ====> PC3
            +--------+ ====> PC4

  Four collision domains. One broadcast domain.
```

A **bridge** is an older device that does the same job as a switch, usually with only two ports. It has the same effect: each port is a separate collision domain, and all ports are in one broadcast domain.

### Router: stops both

A router is a Layer 3 device. It reads IP addresses and moves packets between networks. It does not flood. A broadcast that arrives on one interface stops there.

- **Every router interface is its own collision domain.**
- **Every router interface is its own broadcast domain.**

```text
  PC1 sends a broadcast. It stops at R1.

  PC1 ---+                                +--- PC3
         SW1 ---- G0/0 [R1] G0/1 ---- SW2
  PC2 ---+       X stops here             +--- PC4

  |<--- broadcast domain 1 --->|  |<--- broadcast domain 2 --->|
         192.168.10.0/24                 192.168.20.0/24
```

Because of this, a host that wants to reach another network does not broadcast for it. It sends the frame to its [default gateway](../04-ip-addressing/08-default-gateway.md), which is the router on its own network.

### VLANs: many broadcast domains on one switch

A **VLAN** (Virtual LAN) is a group of switch ports that behaves like a separate switch. A broadcast in VLAN 10 is flooded only to VLAN 10 ports. So each VLAN is its own broadcast domain.

A switch with no VLAN setup puts all ports in one default VLAN (VLAN 1 on Cisco). That is why it has one broadcast domain. Traffic between VLANs needs a router or a Layer 3 switch. Full details are in [VLANs](06-vlans.md) and [Inter-VLAN Routing](08-inter-vlan-routing.md).

### Summary table

| Device | OSI layer | Splits collision domains? | Splits broadcast domains? |
|---|---|---|---|
| Repeater or hub | 1 | No | No |
| Bridge | 2 | Yes, per port | No |
| Switch, no VLANs | 2 | Yes, per port | No |
| Switch with VLANs | 2 | Yes, per port | Yes, per VLAN |
| Router | 3 | Yes, per interface | Yes, per interface |
| Layer 3 switch | 2 and 3 | Yes, per port | Yes, per VLAN or routed port |
| Wi-Fi access point | 2 | No, one radio channel is shared | No |

### Wi-Fi: a collision domain that still matters

An **access point** (AP) connects Wi-Fi devices to the wired network. All clients on one AP share one radio channel, just like a hub. A radio cannot easily listen while it sends, so Wi-Fi cannot detect collisions. It uses **CSMA/CA** (Collision Avoidance) instead: it tries to avoid them and uses acknowledgements to learn that a frame arrived. See [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md).

An AP bridges Wi-Fi to the wired LAN, so Wi-Fi clients and wired hosts in the same VLAN share one broadcast domain.

### How to count domains in a diagram

Use these two recipes. Do not try to count in your head.

**Counting collision domains:**

1. Draw a circle around each hub and everything directly connected to it. That whole circle is 1 domain. Two hubs joined directly share one circle.
2. Every other link is 1 domain by itself. This includes PC to switch, switch to switch, switch to router, and router to router.
3. Add them up.

The trap: count **links, not ports**. A link between two switches uses two ports but it is one collision domain.

**Counting broadcast domains:**

1. Mentally remove every router.
2. Count the separate pieces that remain. A link between two routers is a piece too.
3. If a piece has VLANs, count each VLAN once. Do not count a VLAN again because it spans two switches.

If there is a Layer 3 switch, treat it as a router between its VLANs.

### Broadcast storms

A **broadcast storm** happens when broadcast frames fill the network until normal traffic cannot pass.

The most common cause is a **Layer 2 loop**: two or more paths between switches, with nothing blocking the extra path. For example, someone plugs both ends of a cable into the same small switch.

```text
  Two switches, two links, no loop protection.

        PC1
         |
      +--+--+   link A   +-----+
      |     |============|     |
      | SW1 |            | SW2 |---- PC2
      |     |============|     |
      +-----+   link B   +-----+
```

What happens, step by step:

1. PC1 sends one ARP request. ARP finds a MAC address from an IP address, and its request is a broadcast. See [ARP](05-arp.md).
2. SW1 floods it out of link A and link B. There are now two copies.
3. SW2 receives the copy on link A. It floods it to PC2 and out of link B.
4. SW2 receives the copy on link B. It floods it to PC2 and out of link A.
5. SW1 receives both copies and floods them again, also back to PC1.
6. This repeats forever at full link speed. Every host receives a copy on every loop.

**Why does it never stop?** The Ethernet header has no TTL. TTL (Time To Live) is the counter in an IP packet that drops by one at each router, so a looping packet dies. An Ethernet frame has no such counter. A looping frame lives until someone breaks the loop.

**Why does it grow?** Two things add up:

- Hosts keep sending new broadcasts, and none of the old ones die.
- A switch with three or more links into the loop turns one copy into two or more.

Four switches in a full mesh show the growth. Each switch has 3 links to the others. A copy that arrives on one link leaves on the other 2.

| Hop | Copies moving between switches |
|---|---|
| 0 (first switch floods) | 3 |
| 1 | 6 |
| 2 | 12 |
| 3 | 24 |

Limits: a 1 Gb/s link can carry about 1,488,000 minimum-size frames per second. A storm can reach that rate. Switch CPUs and host CPUs often cannot cope.

**Analogy:** a microphone too close to its loudspeaker. Sound goes in, comes out louder, goes in again, and soon the howl hides every voice. Where it stops being true: audio feedback grows because an amplifier adds power. Ethernet copies are exact, and they grow only because of fan-out and new traffic.

**Symptoms of a storm:**

- Link lights on the switch flash fast and together.
- Many users lose the network at once, on one VLAN.
- Switch CPU is very high, and SSH to the switch is slow or fails.
- Logs show MAC flapping: one MAC address keeps moving between ports.

**Other causes:** a faulty network card that keeps sending, a bad driver, or malware that scans the network. These flood the network without a loop.

**Prevention:**

| Tool | What it does |
|---|---|
| [STP](09-spanning-tree.md) (Spanning Tree Protocol) | Finds loops and blocks extra paths |
| BPDU Guard | Shuts an access port that receives STP messages, which only a switch should send. See [Layer 2 Security](11-layer2-security.md) |
| Storm control | Drops or blocks traffic when broadcasts pass a set rate |
| [Link aggregation](10-link-aggregation.md) | Lets parallel links act as one link, so they form no loop |
| Routed design | Smaller broadcast domains mean a smaller blast radius |

Fuller truth: many small unmanaged switches do not run STP at all. Plugging one into a network is a classic way to create a loop.

### Why large broadcast domains are bad

A large broadcast domain works, but it has costs. **Analogy:** a group chat with 5,000 people. Every phone buzzes for every message, though almost none matter. Where it stops being true: a phone can mute the chat. A network card cannot mute a broadcast, because the message might be for it.

1. **CPU on every host.** A network card ignores unicast frames for other hosts. It must pass every broadcast up to the operating system, which spends CPU to read it and throw it away.
2. **Wasted bandwidth.** On wired gigabit links this is usually small. On Wi-Fi it is a real problem, as Going Deeper shows.
3. **Bigger failure area.** A storm reaches every device in the domain. With 5,000 hosts in one domain, one bad cable can stop 5,000 hosts. With 50 domains of 100 hosts, it stops 100.
4. **Weaker security.** Attacks such as ARP spoofing and rogue DHCP servers reach everyone in the domain. See [Layer 2 Security](11-layer2-security.md).
5. **Harder troubleshooting.** More hosts, more switches and a bigger spanning tree mean more places to search.
6. **More table entries.** The gateway must hold an ARP entry for each host, and each switch must hold more MAC entries.

The fix is to split the network: use more VLANs and smaller subnets, and route between them. Many designs keep a VLAN around a /24 (254 usable hosts). That is a common habit, not a standard. See [Network Design Models](../10-enterprise-and-datacenter/01-network-design-models.md).

## Example

### Worked counting examples

Show your method in an interview. Say the links as you count them.

#### Example 1: one hub, then one switch

```text
  PC1    PC2    PC3    PC4
   |      |      |      |
 +-+------+------+------+-+
 |      HUB or SWITCH     |
 +------------------------+
```

- With a hub: **1** collision domain, because the hub and all its links form one circle, and **1** broadcast domain.
- With a switch: **4** collision domains, because there are 4 links, and **1** broadcast domain, because there is no router and there are no VLANs.

#### Example 2: a router between a switch and a hub

```text
  PC1  PC2  PC3              PC4  PC5  PC6
   |    |    |                |    |    |
 +-+----+----+-+            +-+----+----+-+
 |     SW1     |            |    HUB1     |
 +------+------+            +------+------+
        |                          |
        | G0/0                     | G0/1
      +-+--------------------------+-+
      |              R1              |
      +------------------------------+
```

- Collision domains: **5**.
  - SW1 side: PC1, PC2, PC3 and the SW1-R1 link give 4.
  - Hub side: HUB1 with PC4, PC5, PC6 and its link to R1 gives 1.
- Broadcast domains: **2**. Remove R1 and two pieces remain.

#### Example 3: hubs behind a switch

```text
            PC1   PC2
             |     |
          +--+-----+--+
          |    SW1    |
          +--+-----+--+
             |     |
         +---+     +---+
         |             |
      +--+---+      +--+---+
      | HUB1 |      | HUB2 |
      +-+--+-+      +-+--+-+
        |  |          |  |
      PC3  PC4      PC5  PC6
```

- Collision domains: **4**. PC1 link, PC2 link, the HUB1 circle, the HUB2 circle. PC3 and PC4 can still collide. PC3 and PC5 cannot, because SW1 is between them.
- Broadcast domains: **1**. A broadcast from PC3 reaches all 6 PCs.

#### Example 4: the classic exam diagram

```text
  PC1  PC2                                  PC5  PC6  PC7
   |    |                                    |    |    |
 +-+----+-+                                +-+----+----+-+
 |  SW1   |--------[R1]--------[R2]--------|    HUB2     |
 +---+----+                                +-------------+
     |
  +--+---+
  | HUB1 |
  +-+--+-+
    |  |
  PC3  PC4
```

| # | Collision domain | Why |
|---|---|---|
| 1 | PC1 to SW1 | Switch link |
| 2 | PC2 to SW1 | Switch link |
| 3 | SW1 to HUB1, with PC3 and PC4 | Hub circle |
| 4 | SW1 to R1 | Switch link |
| 5 | R1 to R2 | Router link |
| 6 | R2 to HUB2, with PC5, PC6, PC7 | Hub circle |

| # | Broadcast domain | Members |
|---|---|---|
| 1 | Left LAN | PC1, PC2, PC3, PC4, R1 left interface |
| 2 | Link between routers | R1 right interface, R2 left interface |
| 3 | Right LAN | PC5, PC6, PC7, R2 right interface |

Answer: **6 collision domains, 3 broadcast domains.** The usual wrong answers forget the SW1-R1 link or the R1-R2 link.

Fuller truth: if R1-R2 is a serial WAN link, it is not Ethernet and has no CSMA/CD. Exam answers still count it as one of each.

#### Example 5: VLANs with a router on a stick

```text
                       R1
                       |  G0/0 trunk (VLAN 10, 20)
                       |  G0/0.10 = 192.168.10.1/24
                       |  G0/0.20 = 192.168.20.1/24
                       |
     +-----------------+-----------------+
     |                SW1                |
     +--+----+----+----------+----+----+-+
        |    |    |          |    |    |
       PC1  PC2  PC3        PC4  PC5  PC6
       |-- VLAN 10 --|      |-- VLAN 20 --|
```

A **trunk** is a link that carries many VLANs, each frame marked with its VLAN number. See [Trunking and 802.1Q](07-trunking-8021q.md).

- Collision domains: **7**. 6 PC links plus 1 trunk.
- Broadcast domains: **2**. VLAN 10 and VLAN 20.

The router has only one cable, yet it joins two broadcast domains. The count follows VLANs, not cables.

#### Example 6: one VLAN across two switches

```text
                      R1
                      |
                      | trunk (VLAN 10, 20, 30)
                      |
  PC1 (VLAN 10) ---- SW1 ======= SW2 ---- PC3 (VLAN 10)
  PC2 (VLAN 20) ------+   trunk   +------ PC4 (VLAN 30)
```

- Collision domains: **6**. 4 PC links, R1-SW1, SW1-SW2.
- Broadcast domains: **3**. VLANs 10, 20 and 30. VLAN 10 is on two switches but it counts once. A broadcast from PC1 crosses the trunk to reach PC3.

#### Example 7: a home router

A home "router" is three devices in one box: a router, a small switch and a Wi-Fi access point.

```text
               Internet (ISP)
                    |
                    | WAN port, 203.0.113.50
   +----------------+----------------+
   |  Home router box                |
   |    [ router part ]              |
   |       | LAN 192.168.1.1/24      |
   |    [ 4-port switch ] [ Wi-Fi ]  |
   +-------+------+-----------+------+
           |      |           :
          PC1    PC2        Phone (radio)
```

- Broadcast domains: **2**. The WAN side and the LAN side. Wi-Fi is bridged to the LAN side.
- Collision domains: **4**. WAN link, PC1 link, PC2 link, the shared Wi-Fi channel.

The link inside the box is not counted, because it is not drawn as a port. Guest Wi-Fi is often a separate network, which would add one more broadcast domain.

### Quick answers to check yourself

| Network | Collision | Broadcast |
|---|---|---|
| 8-port hub with 8 PCs | 1 | 1 |
| 24-port switch with 24 PCs, no VLANs | 24 | 1 |
| 24-port switch with 24 PCs, 4 VLANs | 24 | 4 |
| Two linked switches, 5 PCs on each | 11 | 1 |
| Router with 3 LANs, each a switch with 10 PCs | 33 | 3 |
| Bridge between two hubs, 4 PCs on each | 2 | 1 |
| Router between two hubs | 2 | 2 |

If a question shows a switch but no cables, count only connected ports unless it says "all ports in use".

### See your broadcast domain on real systems

Setup: a Linux host at 192.168.10.21 and a Windows host at 192.168.10.22, both in 192.168.10.0/24. The gateway is 192.168.10.1.

**Watch broadcasts arrive (Linux).** `-e` shows MAC addresses and `-n` turns off name lookup.

```bash
$ sudo tcpdump -eni eth0 ether broadcast
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:02:11.482913 52:54:00:aa:10:16 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.10.1 tell 192.168.10.22, length 46
```

The Linux host received an ARP request that was not for it. That proves both hosts share a broadcast domain. A host in 192.168.20.0/24, behind the router, would never see this frame.

More on captures: [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) and [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) (display filter `eth.dst == ff:ff:ff:ff:ff:ff`).

**See your neighbours.** The ARP table lists only hosts in your own broadcast domain, including the gateway.

```bash
# Linux
$ ip neigh show
192.168.10.1 dev eth0 lladdr 52:54:00:aa:10:01 REACHABLE
192.168.10.22 dev eth0 lladdr 52:54:00:aa:10:16 STALE
```

```powershell
# Windows
PS> arp -a

Interface: 192.168.10.22 --- 0x7
  Internet Address      Physical Address      Type
  192.168.10.1          52-54-00-aa-10-01     dynamic
  192.168.10.21         52-54-00-aa-10-15     dynamic
  192.168.10.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  255.255.255.255       ff-ff-ff-ff-ff-ff     static
```

**Check duplex.** A full-duplex link has no collisions.

```bash
# Linux
$ ethtool eth0 | grep -E 'Speed|Duplex'
	Speed: 1000Mb/s
	Duplex: Full
```

```powershell
# Windows
PS> Get-NetAdapter | Select-Object Name, LinkSpeed, FullDuplex

Name     LinkSpeed FullDuplex
----     --------- ----------
Ethernet 1 Gbps          True
```

**Count broadcast domains on a switch (Cisco IOS).** Each VLAN in use is one broadcast domain. Output is trimmed.

```text
SW1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -----------------------
1    default                          active    Gi0/7, Gi0/8
10   SALES                            active    Gi0/1, Gi0/2, Gi0/3
20   HR                               active    Gi0/4, Gi0/5, Gi0/6
```

**Limit a storm (Cisco IOS).** This sets a limit of 5 percent of link bandwidth for broadcasts on one port. Syntax varies by platform.

```text
SW1(config)# interface gigabitethernet0/1
SW1(config-if)# storm-control broadcast level 5.00
SW1(config-if)# storm-control action shutdown
SW1(config-if)# end

SW1# show storm-control gigabitethernet0/1 broadcast
Interface  Filter State   Upper        Lower        Current
---------  -------------  -----------  -----------  ----------
Gi0/1      Forwarding          5.00%        5.00%       0.12%
```

With no action set, the usual Catalyst behaviour is to drop the excess traffic. The `shutdown` action instead puts the port into the err-disabled state, so an engineer must clear it.

**Spot a loop (Cisco IOS).** The log shows MAC flapping:

```text
%SW_MATM-4-MACFLAP_NOTIF: Host 5254.00aa.1015 in vlan 10 is flapping between port Gi0/23 and port Gi0/24
```

The fix: find the extra link, unplug it, and check why STP did not block it. See [Common Network Problems and Fixes](../12-troubleshooting-and-tools/09-common-network-problems.md).

## Going Deeper

### Why collision timing shaped Ethernet

A sender must still be sending when news of a collision comes back. If it has already finished, it will think the frame arrived safely. This sets two linked limits:

- **Slot time:** 512 bit times for 10 and 100 Mb/s Ethernet. That is 51.2 microseconds at 10 Mb/s and 5.12 microseconds at 100 Mb/s.
- **Minimum frame size:** 64 bytes, which is 512 bits. So every frame lasts at least one slot time.

Cable length is limited by the same timing. The old **5-4-3 rule** for 10 Mb/s shared Ethernet said: at most 5 segments, 4 repeaters, and 3 segments with hosts on them. It is history now, but it may appear in exams.

Backoff is exact too. After collision number n, a device picks a random number of slot times from 0 to 2^n - 1. The value of n stops growing at 10. After 16 attempts, the frame is dropped. So the wait range doubles each time, which spreads devices out on a busy network.

### Late collisions and duplex mismatch

A **late collision** is a collision seen after the first 64 bytes of a frame have gone out. The sender may already be past the point where it can retry by itself. Late collisions point to a fault, usually a **duplex mismatch**.

A duplex mismatch means one end uses full duplex and the other uses half duplex. It often happens when one end is hard-coded to 100 Mb/s full and the other uses auto-negotiation. The auto side can sense the speed but not the duplex, so it falls back to half.

- The full-duplex end sends whenever it wants.
- The half-duplex end sees that as collisions, and shows collisions and late collisions.
- The full-duplex end receives broken frames, and shows CRC errors and runts (frames too short).

The link works but is slow, and it gets worse under load.

```text
SW1# show interfaces gigabitethernet0/3
GigabitEthernet0/3 is up, line protocol is up (connected)
  Half-duplex, 100Mb/s, media type is 10/100/1000BaseTX
  ...
     48213 packets input, 6120344 bytes, 0 no buffer
     Received 1204 broadcasts (893 multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     ...
     52117 packets output, 7193021 bytes, 0 underruns
     0 output errors, 1832 collisions, 0 interfaces resets
     0 babbles, 147 late collision, 0 deferred
```

On a healthy full-duplex port, collision counters stay at zero. The fix is to set both ends the same, and auto on both ends is the usual advice.

### Do collision domains still exist?

On a full-duplex switched port, collisions cannot happen. There is still "one collision domain per port" in exam language, but in practice that domain is never used.

- 10 Gb/s and faster Ethernet is full duplex only. CSMA/CD is not used at those speeds.
- 1000BASE-T half duplex exists in the standard but is rarely seen.
- Wi-Fi remains a shared medium, so it is the place where this idea still matters daily.
- 10BASE-T1S, a newer single-pair Ethernet for cars and factories, can share one wire between several nodes. It has an option called PLCA to take turns and avoid collisions.

The old word **microsegmentation** meant giving each host its own switch port. Today it usually means fine-grained security rules between workloads. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### More than broadcasts: BUM traffic

Switches flood three kinds of traffic inside a broadcast domain, together called **BUM**:

| Letter | Traffic | Why it is flooded |
|---|---|---|
| B | Broadcast | Meant for everyone |
| U | Unknown unicast | Destination MAC is not in the switch table yet |
| M | Multicast | Flooded unless the switch tracks group members |

**IGMP snooping** lets a switch watch multicast joins, so it sends multicast only to ports that asked for it. Storm control can usually limit all three types. Multicast is covered in [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

So a "broadcast domain" could better be called a "flooding domain". It is the area one flooded frame can reach.

### What sends broadcasts?

ARP requests, DHCP Discover messages, older Windows name services (NetBIOS) and most Wake-on-LAN tools all use broadcast. Protocols such as mDNS and SSDP use multicast instead, but they are often flooded the same way.

### How routers treat IP broadcasts

- **Limited broadcast, 255.255.255.255:** stays on the local network. A router never forwards it.
- **Directed broadcast, for example 192.168.20.255 sent from another network:** a router could deliver this to the far LAN. Modern routers turn this off by default (`no ip directed-broadcast` on Cisco IOS), because it was abused in amplification attacks.

Some services need to cross this border. A host looking for a DHCP server sends a broadcast that the router will not forward. A **DHCP relay** solves this: the router listens for the broadcast and sends it as a unicast to the server.

```text
R1(config)# interface gigabitethernet0/1
R1(config-if)# ip helper-address 192.168.99.10
```

See [DHCP](../07-application-protocols/04-dhcp.md).

### VLAN, subnet and broadcast domain

Good designs map them one to one: one VLAN, one IP subnet, one broadcast domain. They are still different things. A VLAN is a Layer 2 grouping on switches, a subnet is a Layer 3 address range, and a broadcast domain is the area a flooded frame reaches.

You can break the mapping, for example with two subnets on one VLAN. Both subnets then share one broadcast domain. This usually causes confusion, so avoid it.

### Layer 3 switches

A Layer 3 switch both switches and routes. An **SVI** (Switched Virtual Interface) is a virtual router interface for a VLAN, and each SVI is the gateway for one broadcast domain. A **routed port** acts like a router interface, so it is a broadcast domain by itself. Modern campus designs often route right down to the access layer, which keeps broadcast domains small. Details are in [Inter-VLAN Routing](08-inter-vlan-routing.md).

### IPv6 has no broadcast

IPv6 removed broadcast. It uses multicast instead. Neighbor Discovery replaces ARP and sends to a solicited-node multicast group, so far fewer hosts need to process each message. See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).

The idea of a broadcast domain still applies. The IPv6 "link" is the same area, and an all-nodes multicast to `ff02::1` reaches it all.

### Stretching a broadcast domain across routers

A router normally ends a broadcast domain, but tunnels can carry one across routers.

**VXLAN** wraps Ethernet frames in UDP packets, so one Layer 2 segment can span a routed data centre. The underlay routers see only UDP. The overlay is still one broadcast domain, so a storm or a BUM flood can travel between sites. **EVPN** adds a control plane that lets switches answer many ARP requests locally, which is called ARP suppression. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

So the rule "routers stop broadcasts" is true for frames the router handles. It says nothing about frames hidden inside a tunnel.

### Wi-Fi airtime and broadcasts

Wi-Fi sends broadcast frames at a low basic rate, so that every client can decode them. Clients do not acknowledge them.

Take 500 broadcasts per second, each 64 bytes (512 bits). At a 1 Mb/s basic rate, each takes about 0.5 ms for the data alone. That is about 256 ms of every second, or more than a quarter of the airtime. On wired gigabit, the same traffic is about 256 Kb/s, which is under 0.03 percent of the link.

Numbers here are for illustration. Modern designs raise the basic rate, and many controllers answer ARP on behalf of clients. See [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).

### Cloud and containers

- A **Linux bridge** is a software switch. Each Docker bridge network is one broadcast domain. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).
- **AWS VPCs do not support broadcast.** ARP does not flood there. The platform answers it. So broadcast storms in the usual sense are not a VPC problem. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

## Common Mistakes

- **Counting ports instead of links.** A switch-to-switch link is one collision domain, not two.
- **Forgetting links to and between routers.** The SW1-R1 link and the R1-R2 link both count.
- **Thinking a switch stops broadcasts.** A switch floods them. Only VLANs or Layer 3 devices divide a broadcast domain.
- **Counting one VLAN twice.** A VLAN on three switches is still one broadcast domain.
- **Saying a switch "removes" collisions.** Only full duplex removes them. A half-duplex switch port can still collide with the device on its other end.
- **Saying a hub port is its own domain.** All ports on a hub share one collision domain, including the uplink.
- **Thinking a VLAN gives connectivity by itself.** Hosts in different VLANs cannot reach each other without a router or Layer 3 switch.
- **Believing a storm needs high user traffic.** One ARP request in a loop is enough.
- **Thinking storms are only about bandwidth.** The CPU load on switches and hosts often causes the real outage.
- **Thinking collision counts prove a bad cable.** On a half-duplex link some collisions are normal. Late collisions point to a fault, and a duplex mismatch is the first thing to check.
- **Assuming a router always isolates traffic.** A tunnel such as VXLAN can carry a broadcast domain across routers.

## Interview Questions

**Q: What is a collision domain?**
A: It is a part of the network where two devices that send at the same time would destroy each other's frames. Only one device in it can send at a time. A hub has one collision domain across all ports. Each switch port is a separate one.

**Q: What is a broadcast domain?**
A: It is the set of devices that receive a broadcast sent by any one of them. It usually matches one VLAN and one IP subnet. Routers end it, and switches do not, unless VLANs divide them.

**Q: How do a hub, a switch and a router affect these domains?**
A: A hub splits neither, so it has one of each. A switch gives one collision domain per port, but it keeps one broadcast domain per VLAN. A router gives one collision domain and one broadcast domain per interface.

**Q: A 24-port switch has 24 PCs connected and 3 VLANs. How many domains does it have?**
A: 24 collision domains, since each connected port is one. It has 3 broadcast domains, one per VLAN.

**Q: How many collision domains does a link between two switches create?**
A: One. It has two ports, but it is one link. That is the most common mistake in counting questions.

**Q: Do collision domains matter on full-duplex links?**
A: On a full-duplex link, collisions cannot happen, because both ends can send at the same time without harm. CSMA/CD is not used there. We still say one collision domain per port, but it is mostly a historic idea on wired networks. It still matters for Wi-Fi, which shares a radio channel.

**Q: Does Wi-Fi have collision domains?**
A: Yes, in effect. All devices on one AP share one channel, so they behave like a hub. Radios cannot detect collisions while sending, so Wi-Fi uses CSMA/CA, which tries to avoid them. The AP bridges to the wired LAN, so Wi-Fi clients stay in the same broadcast domain as their VLAN.

**Q: What is a broadcast storm, and why does it not die by itself?**
A: It is when broadcast frames loop or multiply until they use up the links and the device CPUs. Usually it comes from a Layer 2 loop. It does not die because an Ethernet frame has no TTL field. Switches flood it forever until the loop is broken.

**Q: How do you prevent broadcast storms?**
A: I would run Spanning Tree so that extra paths are blocked. I would add BPDU Guard on access ports and storm control to limit broadcast rate. I would also use EtherChannel for parallel links, and keep broadcast domains small so that a failure stays small.

**Q: Why are large broadcast domains bad?**
A: Every host must process each broadcast, which wastes CPU and, on Wi-Fi, a lot of airtime. A storm or loop hits everyone in the domain. Attacks like ARP spoofing reach more hosts. It is also harder to troubleshoot. So we divide networks with VLANs and routing.

**Q: DHCP uses broadcast and routers stop broadcasts. How does a client get an address from a server on another subnet?**
A: Through a DHCP relay. The router, or Layer 3 switch, receives the broadcast on the client's network and forwards it as a unicast to the server. On Cisco this is the `ip helper-address` command on the client-facing interface.

**Q: A whole floor loses the network, and all switch lights flash together. What do you do?**
A: That looks like a loop and a storm. I would check switch logs for MAC flapping and check CPU and interface broadcast rates. Next I would find the port or link that causes the loop and shut it down. After that I would check why STP did not block it, and whether BPDU Guard and storm control were in place.

## Quick Recap

- Collision domain: devices that can crash into each other. Broadcast domain: devices that all hear one broadcast.
- Hub: 1 and 1. Switch: 1 collision domain per port, 1 broadcast domain per VLAN. Router: 1 of each per interface.
- To count collision domains, count links and treat each hub circle as one. To count broadcast domains, remove routers and count pieces and VLANs.
- Full duplex removes collisions. Half duplex uses CSMA/CD, and Wi-Fi uses CSMA/CA.
- A broadcast storm usually comes from a Layer 2 loop, and it lasts because Ethernet frames have no TTL.
- Stop storms with STP, BPDU Guard, storm control and good design.
- Large broadcast domains waste CPU and airtime, spread failures and attacks, and are harder to troubleshoot. Split them with VLANs and routing.
