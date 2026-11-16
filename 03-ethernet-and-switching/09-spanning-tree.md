# Spanning Tree Protocol (STP)

**Level:** Advanced | **Prerequisites:** [How Switches Work](03-how-switches-work.md), [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md), [VLANs](06-vlans.md), [Trunking and 802.1Q](07-trunking-8021q.md)

## In One Line

Spanning Tree Protocol lets switches find and block redundant paths, so the Layer 2 network has no loops but still has spare links ready if a working link fails.

## Why It Matters

Good networks have spare links. If one cable fails, traffic should use another. But two paths between switches create a **loop**: a circle that a frame can travel around again and again. A frame is one unit of data at Layer 2.

A loop at Layer 2 can take down a whole building in seconds. Ethernet has no way to stop a frame that travels in a circle. So we face a conflict:

- We want extra links for safety.
- Extra links create loops.

STP solves this conflict. It leaves all cables connected, but it logically blocks a few ports. The result is one loop-free path between any two switches. If a link fails, a blocked port opens.

Interviewers love STP for three reasons:

- It has clear rules you can apply on a whiteboard, such as "which port blocks here?"
- It is a classic cause of real outages.
- It tests whether you know old behaviour (30 to 50 seconds) from modern behaviour (often under a second).

## Simple Explanation

Imagine a town with many roads. Some roads form circles. A delivery driver with bad directions could drive around a circle forever.

The town picks one central building, the town hall. Then it marks one best road from every neighbourhood to the town hall. Every road that would create a circle gets a "closed" sign. The road is still there. If an open road breaks, a closed road opens.

The result looks like a tree: one trunk, many branches, no circles. That is why it is called a **spanning tree**. "Spanning" means it reaches every switch. "Tree" means it has no circles.

Map the town to the network:

| Town | Network |
|---|---|
| Town hall | Root bridge |
| Road | Link between switches |
| Closed sign | Blocked port |
| Letters between neighbours | BPDUs |

**Where the analogy stops:** a real town has a mayor who decides. Switches have no central control. Each switch makes its own decision from messages it receives, and all switches reach the same answer because they follow the same rules.

The word **bridge** appears everywhere in STP. A bridge is the older name for a switch. In this chapter, "bridge" and "switch" mean the same thing.

## How It Works

### Why Loops Are Deadly

A switch **floods** a frame when it does not know where to send it. Flooding means sending a copy out every port in the VLAN (a VLAN is a virtual LAN: a separate logical network on the same switches), except the port the frame came in on. Broadcast frames, such as [ARP](05-arp.md) requests, are always flooded.

Now look at two switches joined by two cables, with no STP:

```text
                 Link A
   +-------+ ------------------ +-------+
   |  SW1  |                    |  SW2  |
   +-------+ ------------------ +-------+
       |         Link B             |
    PC-A                         PC-B
```

PC-A sends one broadcast:

1. SW1 sends a copy on Link A and a copy on Link B.
2. SW2 gets the Link A copy. It floods it out Link B and to PC-B.
3. SW2 gets the Link B copy. It floods it out Link A and to PC-B.
4. SW1 gets both copies and floods each one back again.
5. This never stops.

Three separate problems happen at once.

**1. Broadcast storm.** Frames circulate forever. Each new broadcast adds more traffic that never leaves. Links reach 100 percent use. Switch CPUs and every host in the VLAN must process the flood. The network stops working.

**2. MAC table instability.** A switch learns where a device is from the source MAC address of each frame. In a loop, PC-A's frame arrives on Link A, then on Link B, then on Link A. The switch keeps moving PC-A's MAC address between ports. This is called **MAC flapping**. Cisco switches log it like this:

```text
%SW_MATM-4-MACFLAP_NOTIF: Host 0050.7966.6800 in vlan 10 is flapping
between port Gi0/1 and port Gi0/2
```

**3. Duplicate frames.** A device can receive the same unicast frame many times. A unicast frame is a frame sent to one device.

**Why does nothing stop the frame?** An IP packet has a TTL (Time To Live) field. Each router lowers it by 1, and the packet dies at 0. An Ethernet frame has no such field. See [Ethernet Frames](02-ethernet-frames.md). Nothing in the frame says "you have been here before".

> **Fuller truth:** in a simple ring, each copy of a broadcast keeps circling, and the count of copies does not grow on its own. The traffic grows because each new broadcast joins the old ones and none ever leave. In a mesh, where a switch has three or more looped links, one frame also multiplies into more copies at each switch. Either way, the links fill up, often within seconds.

### What STP Does, in Four Steps

1. Elect one **root bridge**.
2. On each other switch, pick one **root port**: the best path to the root.
3. On each link, pick one **designated port**: the port that forwards on that link.
4. Block every port that is neither root nor designated.

The sections below explain each step. First we need the message that makes this possible, and the ID that names each switch.

### BPDUs: How Switches Talk

A **BPDU** (Bridge Protocol Data Unit) is a small STP control message that switches send to each other. Hosts do not use BPDUs, although a host can capture them.

Key facts:

- Destination MAC is the multicast address `01:80:C2:00:00:00`. Multicast means "for a group", here all STP bridges.
- A bridge must not forward frames sent to this address as normal traffic. It reads them and uses them.
- The source MAC is the MAC of the sending port.
- BPDUs are sent every **hello time**, 2 seconds by default.
- BPDUs use an IEEE 802.3 frame with an LLC header (DSAP 0x42, SSAP 0x42, Control 0x03). LLC (Logical Link Control) is a small header that says which protocol is inside.

There are two classic BPDU types:

- **Configuration BPDU:** carries the tree information (who is root, how far away, who is sending).
- **TCN BPDU** (Topology Change Notification): tells the root that something changed. It is only 4 bytes.

The classic configuration BPDU is 35 bytes:

| Field | Bytes | Meaning |
|---|---|---|
| Protocol ID | 2 | Always 0x0000 |
| Version | 1 | 0 = STP, 2 = RSTP, 3 = MST |
| BPDU Type | 1 | 0x00 = Config, 0x80 = TCN, 0x02 = RST or MST |
| Flags | 1 | Topology Change, TC Ack, and in RSTP the role and proposal bits |
| Root ID | 8 | Who the sender thinks is root |
| Root Path Cost | 4 | Sender's total cost to reach the root |
| Bridge ID | 8 | The sender's own ID |
| Port ID | 2 | The sending port (priority + number) |
| Message Age | 2 | Hops from the root, roughly |
| Max Age | 2 | When this information expires |
| Hello Time | 2 | How often the root sends BPDUs |
| Forward Delay | 2 | Time spent in each transition state |

The four timer fields are stored in units of 1/256 of a second. An RSTP BPDU adds one more byte (Version 1 Length, value 0), so it is 36 bytes.

Think of a BPDU as a short letter: "I believe the root is X. I am Y. My cost to X is Z." Each switch compares letters and keeps the best one.

**Where the analogy stops:** letters arrive once. BPDUs must keep arriving. If they stop, the switch treats the information as old and throws it away.

### Bridge ID

The **Bridge ID** (BID) is an 8-byte number that names each switch in STP. Lower is better.

The original format was 2 bytes of priority plus a 6-byte MAC address. To run a separate tree per VLAN, the 802.1t update split the priority field. This split is called the **extended system ID**:

```text
+----------+--------------------+--------------------------+
| Priority | Extended System ID |       MAC address        |
|  4 bits  | 12 bits (VLAN ID)  |         48 bits          |
+----------+--------------------+--------------------------+
|<-- bridge priority, 16 bits ->|<-- 48 bits, unique ID -->|
```

Results of this design:

- The default priority is **32768**.
- With only 4 bits left, priority must be a multiple of **4096**.
- There are 16 valid values: 0, 4096, 8192, 12288, 16384, 20480, 24576, 28672, 32768, 36864, 40960, 45056, 49152, 53248, 57344, 61440.
- The shown priority is the configured priority plus the VLAN ID.

Examples:

| Configured priority | VLAN | Priority shown |
|---|---|---|
| 32768 | 1 | 32769 |
| 32768 | 10 | 32778 |
| 24576 | 10 | 24586 |
| 4096 | 10 | 4106 |

This is why a default Cisco switch shows 32769 in VLAN 1.

### Root Bridge Election

The **root bridge** is the one switch that is the centre of the tree. The switch with the **lowest Bridge ID** wins.

1. When a switch starts, it assumes it is the root. It sends BPDUs with its own ID as Root ID and cost 0.
2. When a switch hears a better Root ID, it stops claiming to be root.
3. It stores that better ID and sends it on to its neighbours, with its own cost added.
4. Soon all switches agree on one root.

Comparison order: lowest priority first. If priorities are equal, lowest MAC address wins. MAC addresses are unique, so there is always one winner.

Think of a class that picks a leader. Everyone starts by saying "I have the lowest student number". When you hear a lower number, you stop and repeat that number to others.

**Where the analogy stops:** this election never finishes. The root keeps proving itself with BPDUs. If a better switch appears later, it takes over right away. This is called preemption.

**Warning:** on default settings, the lowest MAC wins. Old switches often have the lowest MAC. So your oldest, slowest switch in a closet can become the centre of your network. Always set the root yourself.

### Path Cost

Each link has a **cost**, a number based on its speed. Faster links have lower cost. A switch's **root path cost** is the sum of the costs of the ports that receive BPDUs on the way from the root.

| Speed | Short method (802.1D-1998) | Long method (802.1t) |
|---|---|---|
| 10 Mbps | 100 | 2,000,000 |
| 100 Mbps | 19 | 200,000 |
| 1 Gbps | 4 | 20,000 |
| 10 Gbps | 2 | 2,000 |
| 25 Gbps | 1 (typical Cisco) | 800 |
| 40 Gbps | 1 (typical Cisco) | 500 |
| 100 Gbps | 1 (typical Cisco) | 200 |

The short method has a problem: costs are small, so all links from 25 Gbps up can look the same. The long method (32-bit values) solves this. The rule is: 20,000,000 divided by speed in Mbps.

- Set it on Cisco with `spanning-tree pathcost method long`. Use the same method on all switches.
- MST always uses long costs on Cisco.
- The default on PVST+ and Rapid PVST+ varies by platform. Check with `show spanning-tree summary`.

Link aggregation also changes cost. A port channel made of 2 x 1 Gbps links usually shows cost 3 on the Cisco short method. STP treats the whole bundle as one link. See [Link Aggregation](10-link-aggregation.md).

### Choosing Port Roles

Every STP decision uses the same four tie-breakers, in order. Lower wins each time.

```text
  1. Lowest Root ID         (who is the root?)
  2. Lowest root path cost  (how far is the root?)
  3. Lowest sender BID      (who told me?)
  4. Lowest sender port ID  (which of their ports told me?)
```

**Port ID** is 2 bytes: a port priority (default 128 on Cisco) and a port number. Cisco shows it as `128.1`.

**Step 1: Root port (RP).** Each non-root switch picks one root port: the port with the best path to the root. The root bridge has no root port.

**Step 2: Designated port (DP).** Each link gets one designated port. This port sends the best BPDU on that link, and it forwards frames for that link. All active ports on the root bridge are designated, because their cost is 0.

**Step 3: Blocked port.** Any port that is neither RP nor DP is blocked. It drops user frames but keeps listening to BPDUs.

Every switch needs exactly one way toward the root, and every link needs exactly one port serving it. Note that only one end of a link blocks. The other end stays designated and keeps sending BPDUs.

```text
          +-------------+
          |  ROOT       |
          |  DP     DP  |     Root bridge: all ports designated
          +--+-------+--+
             |       |
             RP      RP       One root port per non-root switch
          +--+--+ +--+--+
          | SW2 | | SW3 |
          +--+--+ +--+--+
             DP------X        One DP per link, the other end blocks
```

### Port States and Timers (Classic 802.1D)

A port does not jump from blocked to forwarding. If it did, a new loop might form before all switches learned the news. So it waits in steps.

| State | Receives BPDUs | Sends BPDUs | Learns MACs | Forwards user frames |
|---|---|---|---|---|
| Disabled | No | No | No | No |
| Blocking | Yes | No | No | No |
| Listening | Yes | Yes | No | No |
| Learning | Yes | Yes | Yes | No |
| Forwarding | Yes | Yes | Yes | Yes |

- **Disabled:** the port is shut down or has no link.
- **Blocking:** the stable state for a blocked port.
- **Listening:** the port takes part in deciding roles. This lets old forwarding paths die first.
- **Learning:** the port fills the MAC table, so it will not flood too much when it starts forwarding.
- **Forwarding:** normal work.

Three timers control this. All three come from the root bridge, inside its BPDUs:

| Timer | Default | Meaning |
|---|---|---|
| Hello | 2 s | How often the root sends a BPDU |
| Forward Delay | 15 s | Time in Listening, and again in Learning |
| Max Age | 20 s | How long a port keeps stored BPDU data with no refresh |

Think of a train crossing. The lights flash first (listening). The gate comes down (learning). Only then can the train pass (forwarding). The wait avoids a crash, even when the track looks clear.

**Where the analogy stops:** the wait in STP is a fixed timer. It does not check if the way is clear. RSTP fixes this by asking the neighbour directly.

The numbers interviewers want:

```text
  Port comes up:
    Listening 15 s + Learning 15 s = 30 s

  Indirect failure (blocked port must notice by itself):
    Max Age 20 s + Listening 15 s + Learning 15 s = 50 s
```

This explains a classic help desk call: "My PC gets no IP address for 30 seconds after boot." The DHCP request (the message that asks for an IP address) is sent while the port is not forwarding yet. PortFast fixes this.

Change timers only on the root, as all other switches copy them. Most networks should leave them alone. On Cisco, the allowed ranges are hello 1-10, forward delay 4-30, and max age 6-40 seconds.

### RSTP: Roles and Speed

**RSTP** (Rapid Spanning Tree Protocol, IEEE 802.1w) keeps the same root election and the same four tie-breakers. It replaces the slow timers with a fast handshake.

RSTP was later folded into 802.1D-2004, and today the spanning tree text lives in IEEE 802.1Q.

**Fewer states.** Blocking, Listening and Disabled all do the same thing to user traffic: they drop it. RSTP merges them into one state.

| 802.1D state | RSTP state |
|---|---|
| Disabled | Discarding |
| Blocking | Discarding |
| Listening | Discarding |
| Learning | Learning |
| Forwarding | Forwarding |

**More roles.** The old "blocked" port is split by purpose:

| RSTP role | Meaning | Normal state |
|---|---|---|
| Root | Best path to the root | Forwarding |
| Designated | Serves a link | Forwarding |
| Alternate | Backup path to the root, through another switch | Discarding |
| Backup | Spare for a DP on the same shared link | Discarding |
| Disabled | Not in use | Discarding |

A Backup port needs two ports of one switch on the same shared segment, usually through a hub (an old device that repeats every signal to all its ports). This is rare today.

```text
 Alternate: spare for a ROOT port    Backup: spare for a DESIGNATED port

     to root      to root                 +-----------+
        |            |                    |   HUB     |
     +--+--+ link +--+--+                 +--+-----+--+
     | SW1 |------| SW2 |                    |     |
     +-----+    X +-----+                    DP   Backup
                ^                         +--+-----+--+
                SW2 port: Alternate       |    SW1    |
                                          +-----------+
```

**Link types.** RSTP's fast moves work only on some ports:

- **Edge port:** connects to a host, not a switch. It goes to forwarding at once. On Cisco, this is what PortFast sets.
- **Point-to-point link:** a direct link to one switch. Full-duplex ports (they send and receive at the same time) are treated as point-to-point.
- **Shared link:** half-duplex (one direction at a time), and may have a hub. RSTP falls back to the old timers here.

#### Why RSTP Is Fast

Four changes make the speed:

**1. An Alternate port takes over at once.** If a root port fails, the alternate port becomes the new root port and starts forwarding right away. There is no 30-second wait.

**2. Proposal and Agreement replace timers.** A designated port that wants to forward asks its neighbour. The neighbour answers. This takes moments.

```text
    SW1 (upstream)                    SW2 (downstream)
    DP, discarding                    port hears better info
        |                                 |
        | ---- BPDU: Proposal ----------> |
        |                                 | Sync: block own
        |                                 | non-edge DPs
        | <--- BPDU: Agreement ---------- |
    DP, forwarding                    RP, forwarding
                                          |
                                      SW2 repeats this with each
                                      switch below it
```

The trick is **sync**. Before SW2 agrees, it blocks its other non-edge designated ports. So no loop can form through SW2. Then SW2 does the same with the switches below it. The change moves down the tree like a wave.

It is like two people about to carry a heavy table through a door. One says "ready?" The other clears the way and says "ready". They need no fixed countdown.

**Where the analogy stops:** the agreement passes through the whole tree, switch by switch, not only between two.

**3. Every switch sends its own BPDUs.** In classic STP, non-root switches mainly relay the root's BPDUs. In RSTP, each switch sends BPDUs every hello. So BPDUs also work as a keepalive between neighbours: a regular "I am still here" signal.

**4. Faster failure detection.** If a port misses 3 hellos (6 seconds by default), it treats the neighbour's data as lost. If the link goes down physically, the switch reacts at once.

**Result:** on point-to-point links, RSTP often converges in well under a second after a link-down event. In the worst case, when only missed hellos reveal the failure, allow about 6 seconds. With 802.1D neighbours, a port falls back to old timers.

**Backward compatibility:** when an RSTP port hears a version 0 BPDU, it acts as 802.1D on that port. It may stay that way after the old switch is gone. On Cisco, `clear spanning-tree detected-protocols` restores normal mode.

### PVST+

Plain 802.1D uses one tree for all VLANs. This is called the **CST** (Common Spanning Tree). It works, but one blocked port blocks every VLAN on that link. The spare link sits idle.

**PVST+** (Per-VLAN Spanning Tree Plus) is Cisco's version. It runs one separate tree per VLAN, using 802.1D rules and timers. "Plus" means it works over standard 802.1Q trunks. A trunk is a link that carries many VLANs.

Benefit: you can use both uplinks.

```text
         DSW1                      DSW2
   root for VLAN 10          root for VLAN 20
         |   \                  /   |
         |     \              /     |
         |       \          /       |
         |         \      /         |
         |          +---+           |
         +----------|ASW|-----------+
                    +---+

  VLAN 10: left uplink forwards, right uplink blocks
  VLAN 20: right uplink forwards, left uplink blocks
```

Cost: each VLAN uses CPU and memory, with BPDUs every 2 seconds per VLAN per trunk. With hundreds of VLANs, this adds up. Catalyst platforms limit the number of instances, and the limit varies by model.

On an 802.1Q trunk, Cisco sends BPDUs for VLANs other than the common tree **tagged**, to `01:00:0C:CC:CC:CD`. Standard switches flood this address as ordinary multicast. This lets PVST+ cross a non-Cisco cloud. The common tree (VLAN 1) still uses the IEEE address, untagged, so it can join a standard STP domain.

### Rapid PVST+

**Rapid PVST+** is the same idea with RSTP inside: one RSTP tree per VLAN. It gives per-VLAN load sharing and fast convergence.

It is the common choice on Cisco campus networks. The default mode varies by platform and software. Older Catalyst switches often default to PVST+, while many current Catalyst and Nexus platforms default to Rapid PVST+. Other vendors differ: for example, Arista EOS defaults to MST. Always check.

```text
! Cisco IOS
SW1(config)# spanning-tree mode rapid-pvst
```

### MST

**MST** (Multiple Spanning Tree, IEEE 802.1s, now part of 802.1Q) maps many VLANs onto a few trees. If you have 500 VLANs but only two useful paths, you need two trees, not 500.

Think of commuters. PVST+ gives each person a private car. MST puts them on two buses, one per route. Same routes, far fewer vehicles.

**Where the analogy stops:** all VLANs on an MST instance share one fate. If that tree blocks a link, it blocks it for every VLAN in the group.

Key terms:

- **MST region:** a group of switches with matching MST configuration. To outside switches, the whole region looks like one big bridge.
- **MST configuration:** three things must match exactly: region **name**, **revision number**, and the **VLAN-to-instance map**. A switch hashes the map into a digest, and sends the digest in its BPDUs. Any difference means a different region.
- **Instance (MSTI):** one tree, carrying a group of VLANs.
- **IST** (Internal Spanning Tree): instance 0. It carries the BPDUs for the region. All VLANs start here until you map them.
- **CIST** (Common and Internal Spanning Tree): the single tree that joins regions and non-MST switches.

Only one BPDU is sent per hello, with the data for all instances in it. So 500 VLANs add no extra BPDUs. Inside a region, loop protection uses a hop count, not message age. The Cisco default is 20 hops.

Cisco IOS Catalyst switches support 16 instances (0-15). The standard allows more, and NX-OS supports more.

### PortFast

**PortFast** tells a switch: "this port goes to a host, not a switch." The port skips Listening and Learning and forwards at once. It also does not cause topology change events when it goes up or down.

Under RSTP, PortFast makes the port an **edge port**.

```text
! Cisco IOS: one access port
SW1(config)# interface GigabitEthernet0/5
SW1(config-if)# spanning-tree portfast

! All access ports (not trunks)
SW1(config)# spanning-tree portfast default
```

Newer IOS XE uses `spanning-tree portfast edge` and `spanning-tree portfast edge default`. Use `spanning-tree portfast trunk` (or `portfast edge trunk`) on a trunk to a server or hypervisor that does not bridge between its ports.

If a PortFast port receives a BPDU, it loses its PortFast status and acts like a normal STP port. But the switch has already forwarded for a moment, so PortFast alone is not safe. Pair it with BPDU Guard.

### BPDU Guard

**BPDU Guard** shuts down a port the moment it receives any BPDU. The port goes to the **err-disabled** state: a software shutdown caused by an error. A host never sends BPDUs. So a BPDU on an access port means someone connected a switch, or is attacking.

Think of a fuse. When something wrong arrives, it cuts the circuit at once to protect the rest.

**Where the analogy stops:** an unwanted switch that does not run STP sends no BPDUs. BPDU Guard will not catch it.

```text
! Cisco IOS: on one port
SW1(config-if)# spanning-tree bpduguard enable

! On all PortFast ports
SW1(config)# spanning-tree portfast bpduguard default
```

The interface command works with or without PortFast. The global command applies only to ports that are in PortFast state.

Log and recovery:

```text
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/5 with BPDU
Guard enabled. Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Gi0/5, putting Gi0/5
in err-disable state

SW1# show interfaces status err-disabled

Port      Name       Status       Reason          Err-disabled Vlans
Gi0/5                err-disabled bpduguard

! Remove the cause first, then recover by hand
SW1(config)# interface GigabitEthernet0/5
SW1(config-if)# shutdown
SW1(config-if)# no shutdown

! Or recover automatically (Cisco default interval is 300 seconds)
SW1(config)# errdisable recovery cause bpduguard
SW1(config)# errdisable recovery interval 300
```

### Root Guard

**Root Guard** protects the position of the root. It stops a port from becoming a root port. If a guarded port receives a **superior BPDU** (one better than the current root's), the switch puts the port in **root-inconsistent** state, for that VLAN. The port forwards no user traffic.

Use it on designated ports that face switches that must never be root: downlinks from distribution to access, or ports to a customer's switch.

It is like a guest list at a meeting: the guest can attend, but cannot take the chair's seat.

**Where the analogy stops:** the port recovers by itself. Once the superior BPDUs stop for long enough, it returns to normal.

```text
! Cisco IOS
DSW1(config)# interface GigabitEthernet0/24
DSW1(config-if)# spanning-tree guard root

%SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port Gi0/24 on
VLAN0010.

DSW1# show spanning-tree inconsistentports

Name                 Interface              Inconsistency
-------------------- ---------------------- ------------------
VLAN0010             GigabitEthernet0/24    Root Inconsistent

Number of inconsistent ports (segments) in the system : 1
```

### Loop Guard

**Loop Guard** protects against a silent failure: a port stops receiving BPDUs, but the link still looks up.

Normally a blocked port stays blocked because it keeps hearing BPDUs. If BPDUs stop, STP thinks the path is gone. The port expires the old data, becomes designated, and starts forwarding. If the real cause was a one-way fibre fault or a software bug, the other side still forwards too. A loop forms.

```text
       DP (sends BPDUs)            Alternate (blocked)
     SW2 ---------- TX fibre broken ---X---> SW3
         <----------- RX fibre fine ------

  Without Loop Guard: SW3 hears nothing, unblocks, LOOP.
  With Loop Guard:    SW3 hears nothing, goes loop-inconsistent, safe.
```

With Loop Guard, a root or alternate port that stops hearing BPDUs goes to **loop-inconsistent** state, not forwarding. When BPDUs return, it recovers by itself.

Think of a guard who receives a "keep the gate shut" radio call every 2 seconds. If the radio goes quiet, a careless guard opens the gate. A careful guard keeps it shut until the radio works.

**Where the analogy stops:** a real guard might call for help. Loop Guard just stays blocked and logs a message.

```text
! Cisco IOS: on one port
SW3(config-if)# spanning-tree guard loop

! On all point-to-point ports
SW3(config)# spanning-tree loopguard default

%SPANTREE-2-LOOPGUARD_BLOCK: Loop guard blocking port Gi0/2 on
VLAN0010.
```

Do not combine Root Guard and Loop Guard on the same port. Root Guard belongs on designated ports, and Loop Guard belongs on non-designated ports.

### Guard Features Compared

| Feature | Put it on | Trigger | Action | Recovery |
|---|---|---|---|---|
| PortFast | Host ports | Link up | Skip to forwarding | Not applicable |
| BPDU Guard | Host ports | Any BPDU | Err-disable the port | Manual or errdisable recovery |
| Root Guard | Designated ports facing switches that must not be root | Superior BPDU | Root-inconsistent | Automatic |
| Loop Guard | Root and alternate ports | BPDUs stop arriving | Loop-inconsistent | Automatic |

## Example

### Example 1: Triangle With Equal Costs

Three switches in VLAN 1, all on default priority. All links are 1 Gbps, short method, so each link costs 4.

| Switch | Priority | MAC |
|---|---|---|
| SW1 | 32769 | 0011.2233.0001 |
| SW2 | 32769 | 0011.2233.0002 |
| SW3 | 32769 | 0011.2233.0003 |

```text
                +-----------+
                |    SW1    |
                +-----------+
            Gi0/1 /       \ Gi0/2
                 /         \
          Gi0/1 /           \ Gi0/1
          +-------+       +-------+
          |  SW2  |-------|  SW3  |
          +-------+       +-------+
              Gi0/2       Gi0/2
```

**Step 1: Elect the root.** Priorities tie at 32769. Lowest MAC wins. SW1 is root.

**Step 2: Pick root ports.**

| Switch | Path via | Cost | Result |
|---|---|---|---|
| SW2 | Gi0/1 direct to SW1 | 4 | Root port |
| SW2 | Gi0/2 via SW3 | 4 + 4 = 8 | Not root port |
| SW3 | Gi0/1 direct to SW1 | 4 | Root port |
| SW3 | Gi0/2 via SW2 | 4 + 4 = 8 | Not root port |

**Step 3: Pick designated ports.**

| Link | Candidates | Winner | Why |
|---|---|---|---|
| SW1-SW2 | SW1 Gi0/1 (cost 0), SW2 Gi0/1 (cost 4) | SW1 Gi0/1 | Lower cost |
| SW1-SW3 | SW1 Gi0/2 (cost 0), SW3 Gi0/1 (cost 4) | SW1 Gi0/2 | Lower cost |
| SW2-SW3 | SW2 Gi0/2 (cost 4), SW3 Gi0/2 (cost 4) | SW2 Gi0/2 | Cost ties, lower BID |

**Step 4: Block the rest.** SW3 Gi0/2 is neither RP nor DP. It blocks.

```text
                  SW1 (ROOT)
            Gi0/1 DP     DP Gi0/2
                 /         \
          Gi0/1 RP         RP Gi0/1
               SW2         SW3
          Gi0/2 DP ------X (blocked) Gi0/2
```

Frames from SW2 to SW3 now travel SW2, SW1, SW3.

### Example 2: The Direct Link Is Not the Best Link

Four switches in a square. SW1 is root. One link is only 100 Mbps.

```text
    +-----+    1G (cost 4)    +-----+
    | SW1 |-------------------| SW2 |
    +-----+                   +-----+
       |                         |
       | 100M (cost 19)          | 1G (cost 4)
       |                         |
    +-----+    1G (cost 4)    +-----+
    | SW3 |-------------------| SW4 |
    +-----+                   +-----+
```

Assume BIDs rise in number order: SW1 is lowest, SW4 is highest.

**Root port for each switch:**

| Switch | Option | Cost | Result |
|---|---|---|---|
| SW2 | Direct to SW1 | 4 | Root port |
| SW2 | Via SW4, SW3, SW1 | 4 + 4 + 19 = 27 | No |
| SW4 | Via SW2, SW1 | 4 + 4 = 8 | Root port |
| SW4 | Via SW3, SW1 | 4 + 19 = 23 | No |
| SW3 | Direct to SW1 | 19 | No |
| SW3 | Via SW4, SW2, SW1 | 4 + 4 + 4 = 12 | Root port |

SW3's direct link to the root loses. Three fast hops cost 12, and one slow hop costs 19.

**Designated ports:**

| Link | Port costs | Designated port |
|---|---|---|
| SW1-SW2 | SW1 0, SW2 4 | SW1 |
| SW2-SW4 | SW2 4, SW4 8 | SW2 |
| SW4-SW3 | SW4 8, SW3 12 | SW4 |
| SW1-SW3 | SW1 0, SW3 12 | SW1 |

**Blocked:** SW3's port on the 100 Mbps link. It is neither RP nor DP.

```text
    SW1 (ROOT) [DP]----------------[RP] SW2
     [DP]                              [DP]
       |                                 |
       |                                 |
       X                                 |
     [blocked]                         [RP]
    SW3 [RP]----------------------[DP] SW4

    Traffic from SW3 to SW1 goes SW3, SW4, SW2, SW1.
```

**Lesson:** STP looks at cost, not hop count. Note also that the SW1 end of the blocked link stays designated. Only the SW3 end blocks.

### Example 3: Two Parallel Links

SW1 is root. Two 1 Gbps cables go from SW1 to SW2.

```text
        Gi0/1 (128.1) -------------- Gi0/1
  SW1                                       SW2
        Gi0/2 (128.2) -------------- Gi0/2
```

SW2 compares the two BPDUs:

1. Root ID: same.
2. Cost: same, 4.
3. Sender BID: same, both from SW1.
4. Sender port ID: 128.1 beats 128.2.

So SW2's Gi0/1 is the root port and SW2's Gi0/2 blocks. To change this, lower the port priority **on SW1**, the sender:

```text
! Cisco IOS on SW1: make Gi0/2 win
SW1(config)# interface GigabitEthernet0/2
SW1(config-if)# spanning-tree vlan 10 port-priority 64
```

Setting port priority on SW2 has no effect here, as SW2 reads the sender's port ID. In a real network, bundle these links with [EtherChannel](10-link-aggregation.md), so both can carry traffic.

### Example 4: Set the Root and Verify

Using the triangle from Example 1, now in VLAN 10. Make SW1 the clear root.

```text
! Cisco IOS on SW1
SW1(config)# spanning-tree mode rapid-pvst
SW1(config)# spanning-tree vlan 10 priority 4096
```

`spanning-tree vlan 10 root primary` also works. It is a one-time macro: it sets priority 24576, or 4096 below the current root if that is not low enough. If a lower-priority switch joins later, it does not adjust. Setting the value directly is clearer.

```text
SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0011.2233.0001
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106   (priority 4096 sys-id-ext 10)
             Address     0011.2233.0001
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ---------
Gi0/1               Desg FWD 4         128.1    P2p
Gi0/2               Desg FWD 4         128.2    P2p
```

```text
SW3# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0011.2233.0001
             Cost        4
             Port        1 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0011.2233.0003
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ---------
Gi0/1               Root FWD 4         128.1    P2p
Gi0/2               Altn BLK 4         128.2    P2p
```

How to read this:

- The Root ID and Bridge ID differ, so SW3 is not the root.
- `Cost 4` is SW3's root path cost.
- `Altn BLK` is an alternate port. IOS prints BLK even though the RSTP state is really Discarding.
- `P2p` is a point-to-point link type. An edge port shows `P2p Edge`.

To check the mode and defaults:

```text
SW3# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: none
Extended system ID           is enabled
Portfast Default             is disabled
PortFast BPDU Guard Default  is disabled
Portfast BPDU Filter Default is disabled
Loopguard Default            is disabled
EtherChannel misconfig guard is enabled
UplinkFast                   is disabled
BackboneFast                 is disabled
Configured Pathcost method used is short
```

### Example 5: Failure on Classic STP vs RSTP

Same triangle. The SW1-SW2 link fails. SW3 Gi0/2 is the blocked port. SW3 cannot see the failure directly, as its own links are still up. This is an **indirect failure**.

**Classic 802.1D:**

| Time | Event |
|---|---|
| 0 s | SW2 loses its root port. It has no other path it knows about, so it claims to be root |
| 0 s | SW2 sends BPDUs out Gi0/2 saying "I am root" |
| 0 to 20 s | SW3 Gi0/2 ignores these worse BPDUs, as it still stores the old good data |
| 20 s | Max Age expires. SW3 Gi0/2 enters Listening and sends BPDUs naming SW1 as root, cost 4 |
| 20 s | SW2 hears a better root. Gi0/2 becomes SW2's root port |
| 35 s | SW3 Gi0/2 enters Learning |
| 50 s | SW3 Gi0/2 enters Forwarding. SW2 reaches SW1 through SW3 |

**RSTP:**

| Time | Event |
|---|---|
| About 0 s | SW2 loses its root port and claims to be root on Gi0/2 |
| Moments later | SW3 receives this worse BPDU from the designated port on that link. RSTP accepts it at once |
| Moments later | SW3 knows a better root, so Gi0/2 becomes designated and sends a proposal |
| Well under 1 s | SW2 sees a better root, makes Gi0/2 its root port, and agrees. SW3 Gi0/2 forwards |

Same topology. 50 seconds became less than a second.

### Example 6: See BPDUs From a Host

Hosts can see BPDUs on a switch access port, which helps in troubleshooting. This also shows why a rogue device can read your topology.

Linux, capturing on an access port in VLAN 10 (SW2 Gi0/5):

```bash
# Linux: capture 1 BPDU, show MAC addresses, decode fully
sudo tcpdump -i eth0 -c 1 -e -nn -vv 'ether dst 01:80:c2:00:00:00'
```

```text
10:42:07.120311 00:11:22:33:00:02 > 01:80:c2:00:00:00, 802.3,
length 60: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command,
ctrl 0x03: STP 802.1w, Rapid STP, Flags [Learn, Forward],
bridge-id 800a.00:11:22:33:00:02.8005, length 36
    message-age 1.00s, max-age 20.00s, hello-time 2.00s,
    forwarding-delay 15.00s
    root-id 100a.00:11:22:33:00:01, root-pathcost 4,
    port-role Designated
```

Decoding the values:

- `800a` is 0x800A = 32778: priority 32768 + VLAN 10.
- `100a` is 0x100A = 4106: root priority 4096 + VLAN 10.
- `8005` is 128.5: port priority 128 (0x80), port number 5.
- `root-pathcost 4` is SW2's cost to SW1.
- `message-age 1.00s` shows one hop from the root.

Windows does not ship a tool that decodes STP in this way. Use Wireshark's `tshark` with the same filter:

```powershell
# Windows 11, with Wireshark installed
& "C:\Program Files\Wireshark\tshark.exe" -i Ethernet -c 1 -V `
  -f "ether dst 01:80:c2:00:00:00"
```

Normal hosts do not run STP. A Linux bridge can:

```bash
# Linux: turn on the kernel STP for a software bridge
sudo ip link add br0 type bridge
sudo ip link set br0 type bridge stp_state 1
bridge link show
```

The Linux kernel bridge runs classic 802.1D. RSTP and MST on Linux need a user-space daemon such as `mstpd`. See [Linux Networking](../11-cloud-and-containers/01-linux-networking.md) for bridges.

### Example 7: Typical Config for a Campus

```text
! Distribution switch DSW1 (Cisco IOS)
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 24576
spanning-tree vlan 20 priority 28672
!
interface range GigabitEthernet1/0/1 - 24
 description Downlinks to access switches
 spanning-tree guard root
```

DSW2 uses the same config, with the two priorities swapped: 28672 for VLAN 10 and 24576 for VLAN 20.

```text
! Access switch ASW1 (Cisco IOS)
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
spanning-tree loopguard default
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

Keep the STP root on the same switch as the active default gateway for that VLAN. Traffic then takes a direct path to the gateway. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

### Example 8: MST Region

Two instances carry four VLANs. Every switch in the region needs the same three settings.

```text
! Cisco IOS on all switches in the region
SW1(config)# spanning-tree mode mst
SW1(config)# spanning-tree mst configuration
SW1(config-mst)# name CAMPUS
SW1(config-mst)# revision 1
SW1(config-mst)# instance 1 vlan 10,20
SW1(config-mst)# instance 2 vlan 30,40
SW1(config-mst)# exit

! Root placement differs per instance
SW1(config)# spanning-tree mst 1 priority 24576
SW1(config)# spanning-tree mst 2 priority 28672
```

```text
SW1# show spanning-tree mst configuration
Name      [CAMPUS]
Revision  1     Instances configured 3

Instance  Vlans mapped
--------  ---------------------------------------------
0         1-9,11-19,21-29,31-39,41-4094
1         10,20
2         30,40
```

## Going Deeper

### The Priority Vector

RSTP describes the four tie-breakers as a **priority vector**, a list compared from left to right:

```text
{ Root ID, Root Path Cost, Sender BID, Sender Port ID,
  Receiving Port ID }
```

The fifth item, the receiving port ID, matters only when a hub puts two ports of one switch on the same link. Then the bridge cannot tell which is better by the sender's data alone.

Any BPDU better than the one stored is called **superior**. A worse one is **inferior**. This simple comparison drives all of STP.

### Cost Is Added on Receipt

A BPDU carries the sender's cost, without the link between sender and receiver. The receiver adds the cost of its own receiving port.

This means cost is set per direction. If you run `spanning-tree cost 100` on SW2's port only, it changes SW2's view of the path, but not SW1's. This causes confusing, asymmetric trees. Change both ends, unless you have a clear reason not to.

The same logic explains port priority in Example 3. Use the right tool for the right side:

| Goal | Knob | Where |
|---|---|---|
| Choose the root | Bridge priority | The switch that should be root |
| Choose a switch's root port | Port cost | The local switch |
| Break a parallel link tie | Port priority | The upstream switch |

### Timer Rules

The standard ties the three timers together:

```text
  2 x (Forward Delay - 1) >= Max Age >= 2 x (Hello + 1)
```

Default check: 2 x 14 = 28 >= 20 >= 6. These defaults assume a **diameter** of 7, where diameter is the largest number of switches between any two end stations.

Message Age grows by about 1 per hop. A switch drops the BPDU when Message Age reaches Max Age. So a very deep tree, or a much lower Max Age, can break a network far from the root. Do not lower timers on a whim. If you need speed, move to RSTP.

### Topology Changes and the MAC Table

After a topology change, the MAC table may point the wrong way. Traffic can go into a black hole until entries age out, 300 seconds by default on Cisco. So STP must tell switches to clear them.

**Classic 802.1D:**

1. A switch detects a change.
2. It sends a TCN BPDU out its root port, repeating until the upstream switch acks with the TCA (Topology Change Acknowledge) flag.
3. Each upstream switch does the same, until the TCN reaches the root.
4. The root sets the TC (Topology Change) flag in its BPDUs, for Max Age + Forward Delay (35 seconds).
5. All switches lower MAC aging to Forward Delay (15 seconds) for that time.

**RSTP** is more direct:

- Only a non-edge port moving to forwarding counts as a topology change. A port going down does not, as that alone cannot create a loop.
- The detecting switch sets the TC bit in its BPDUs for 2 x hello, and sends them out its designated and root ports.
- Each switch that receives it flushes MAC entries at once on all ports, except the one it came in on and edge ports.
- It then passes the news on. No trip to the root is needed.

This flush causes a brief flood. If hosts connect to non-edge ports, every host reboot causes a flush. This is why PortFast matters on access ports, even under RSTP.

**Finding the source of repeated changes:**

```text
SW2# show spanning-tree vlan 10 detail | include executing|occurr|from
 VLAN0010 is executing the rstp compatible Spanning Tree protocol
  Number of topology changes 47 last change occurred 00:00:32 ago
          from GigabitEthernet0/7
```

Follow the `from` port switch by switch, until you find the flapping device.

### MST Design Details

- **All three settings must match.** A switch with a different VLAN map is in a different region, even if the name is the same. So map VLANs ahead of time, including VLANs not yet in use.
- **The map is one change for all.** Changing it means touching every switch, and the region splits until all are done.
- **Trunks must carry all VLANs of an instance.** MST blocks per instance, not per VLAN. If a link forwards for instance 1, but VLAN 20 is pruned from that trunk, VLAN 20 is cut off, even though a blocked path exists elsewhere.
- **Instance 0 is special.** It carries the BPDUs and talks to the outside. Keep user VLANs out of it where possible.
- **Inside the region** the root of each instance is called a regional root. The root of the CIST across regions is the CIST root.
- **The region looks like one bridge** to outside switches. A Cisco MST boundary can interoperate with PVST+ neighbours, but with strict rules on where the root may sit. Keep the CIST root inside the MST region.

### Comparison of the Variants

| Protocol | Origin | Trees | Typical convergence | Notes |
|---|---|---|---|---|
| STP | IEEE 802.1D | 1 for all VLANs | 30-50 s | Original |
| PVST+ | Cisco | 1 per VLAN | 30-50 s | Load sharing by VLAN |
| RSTP | IEEE 802.1w, now in 802.1Q | 1 for all VLANs | Under a second to a few seconds | Standard, rapid |
| Rapid PVST+ | Cisco | 1 per VLAN | Under a second to a few seconds | Common Cisco choice |
| MST | IEEE 802.1s, now in 802.1Q | Few, shared by groups of VLANs | Under a second to a few seconds | Best at scale, more planning |

### Other Protection Features

**BPDU Filter.** Stops BPDUs on a port. Two very different modes on Cisco:

- Global (`spanning-tree portfast bpdufilter default`): applies to PortFast ports. They send a few BPDUs at link up, then stop. If a BPDU arrives, the port loses PortFast and filtering.
- Interface (`spanning-tree bpdufilter enable`): the port sends and ignores all BPDUs. This turns STP off on that port. A mistake here can create a real loop.

**UDLD** (Unidirectional Link Detection). A Cisco protocol that checks both directions of a link at a lower level, mostly on fibre. It finds a one-way link, even when no STP port is at risk. Use it with Loop Guard, as they cover different cases:

| | Loop Guard | UDLD |
|---|---|---|
| Finds one-way fibre | Yes | Yes |
| Finds a software fault that stops BPDUs | Yes | No |
| Finds miswired fibre pairs | No | Yes |
| Scope | Per VLAN | Per port |

**Bridge Assurance.** Found on Nexus and some Catalyst platforms. On ports set as network type, BPDUs must flow in both directions. If they stop, the port goes to an inconsistent state. It generalises the idea behind Loop Guard. Both ends must support it.

**UplinkFast and BackboneFast.** Older Cisco add-ons for 802.1D. UplinkFast opened a spare root port quickly. BackboneFast skipped the Max Age wait on indirect failure. RSTP includes both ideas, so they are not needed with RSTP or MST.

**Dispute check.** RSTP on Cisco can spot a designated port that hears an inferior BPDU which also claims to be learning or forwarding. This suggests a one-way link, and the port blocks.

**Native VLAN mismatch.** A PVST+ switch detects a VLAN mismatch across a trunk and blocks the affected VLANs with a port-type or PVID inconsistency. See [Trunking and 802.1Q](07-trunking-8021q.md).

### STP as an Attack Surface

BPDUs have no authentication. A device that sends superior BPDUs can become root and draw traffic through itself, or force constant reconvergence. BPDU Guard and Root Guard are the main defences. See [Layer 2 Security](11-layer2-security.md).

### Modern Designs That Avoid Blocking

A blocked port is paid-for bandwidth that carries nothing. Modern designs reduce STP's job:

- **EtherChannel:** bundles parallel links into one logical link, so nothing blocks. See [Link Aggregation](10-link-aggregation.md).
- **Multi-chassis link aggregation** (Cisco vPC, StackWise, and vendor MLAG): two switches appear as one to a downstream switch.
- **Routed access:** Layer 3 reaches the access switch, so the routing protocol handles paths. See [Network Design Models](../10-enterprise-and-datacenter/01-network-design-models.md).
- **Spine-leaf with ECMP** (Equal-Cost Multi-Path): routing uses all equal links at once, with [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) carrying Layer 2 where needed. See [Spine-Leaf](../10-enterprise-and-datacenter/02-spine-leaf.md).

**STP is still kept on as a safety net.** Someone will eventually plug a cable into the wrong port. Leaving STP running at the edge protects against that human error. Do not disable it.

### A Short History

Radia Perlman designed the spanning tree algorithm at Digital Equipment Corporation in the 1980s. IEEE made it 802.1D. RSTP (802.1w) arrived about 2001, and MST (802.1s) about 2002. Today both are maintained in IEEE 802.1Q.

### Troubleshooting a Suspected Loop

1. Look for symptoms: links at 100 percent, high switch CPU, MAC flap logs, all VLAN hosts slow or down.
2. Find the root with `show spanning-tree root`, and confirm it is the switch you expect.
3. Check that blocked ports you expect to block are actually blocking.
4. Check for inconsistent ports and err-disabled ports.
5. In an active storm, shut one redundant link to break the loop first, then find the cause.
6. Look for a small switch or hub added by a user, or a port with `bpdufilter`.

See [Common Network Problems](../12-troubleshooting-and-tools/09-common-network-problems.md) for the wider method.

## Common Mistakes

- **Saying "the highest priority wins."** The lowest number wins. In STP, a lower Bridge ID is better.
- **Leaving the root to chance.** With defaults, the lowest MAC wins, often the oldest switch. Set priority by design.
- **Setting a priority like 10000.** Priority must be a multiple of 4096 when the extended system ID is in use.
- **Thinking a link is blocked at both ends.** Only one port blocks. The other end stays designated and keeps sending BPDUs.
- **Thinking blocked ports do nothing.** A blocked port drops user frames, but keeps receiving BPDUs. That is how it knows when to take over.
- **Thinking a blocked port blocks everything on all VLANs.** On PVST+ and Rapid PVST+, a port can block for VLAN 10 and forward for VLAN 20.
- **Choosing the direct link as the root port.** The lowest total cost wins, not the fewest hops.
- **Changing port priority on the wrong switch.** A switch reads the sender's port ID, so set it on the upstream switch.
- **Using PortFast on a link to a switch.** This can create a temporary loop. Use it on host ports only, and pair it with BPDU Guard.
- **Mixing up guards.** Root Guard reacts to a better BPDU. Loop Guard reacts to missing BPDUs. BPDU Guard reacts to any BPDU.
- **Thinking `root primary` is a lasting setting.** It runs once and writes a fixed priority. It does not react to later changes.
- **Thinking RSTP is fast everywhere.** Shared links and ports facing 802.1D neighbours fall back to timer-based behaviour.
- **Using BPDU Filter to stop an error message.** On an interface, this removes loop protection for that port.
- **Mismatching MST region settings.** A name, revision or VLAN map difference creates a separate region, and the design no longer does what you planned.
- **Saying STP is obsolete.** It is reduced by modern designs, but it remains a safety net on most Layer 2 networks.

## Interview Questions

**Q: Why does a Layer 2 network need Spanning Tree?**

A: Redundant links create loops, and an Ethernet frame has no TTL, so a looped frame never dies. That gives broadcast storms, MAC table flapping and duplicate frames. STP blocks some ports to leave one loop-free path, and opens a blocked port if a link fails.

**Q: How is the root bridge elected?**

A: Every switch starts by claiming to be root and sends BPDUs. The lowest Bridge ID wins. The Bridge ID is priority first, then MAC address. Default priority is 32768 plus the VLAN ID. I set it by design, so the root sits at the core or distribution layer.

**Q: Why must bridge priority be a multiple of 4096?**

A: The original 16-bit priority was split into 4 bits of priority and a 12-bit extended system ID that holds the VLAN ID. Only the top 4 bits remain configurable, so the step is 4096, giving 16 possible values.

**Q: Walk me through how STP picks port roles.**

A: First elect the root. Then each non-root switch picks one root port, with the lowest total cost to the root. Then each link gets one designated port, the end with the lowest cost, using BID and port ID as tie-breakers. Every other port blocks. All ports on the root are designated.

**Q: What are the STP tie-breakers, in order?**

A: Lowest root ID, then lowest root path cost, then lowest sender Bridge ID, then lowest sender port ID. On a hub, the receiving port ID can break a final tie.

**Q: What are the classic STP states and timers?**

A: Blocking, Listening, Learning, Forwarding, plus Disabled. Hello is 2 seconds, Forward Delay 15 and Max Age 20. A port coming up takes 30 seconds, 15 in Listening and 15 in Learning. An indirect failure takes up to 50 seconds, because it adds Max Age.

**Q: Why is RSTP so much faster than 802.1D?**

A: RSTP does not rely on timers on point-to-point links. Alternate ports take over immediately, Proposal and Agreement moves a designated port to forwarding after a handshake, edge ports forward at once, and each switch sends its own hellos, so three missed hellos reveal a failure. Convergence is often under a second.

**Q: What is the difference between PVST+, Rapid PVST+ and MST?**

A: PVST+ is Cisco's one 802.1D tree per VLAN. Rapid PVST+ is one RSTP tree per VLAN. MST is an IEEE standard that maps many VLANs onto a few RSTP-based instances. It scales better, but all switches in a region must match on name, revision and the VLAN map.

**Q: What is the difference between PortFast, BPDU Guard, Root Guard and Loop Guard?**

A: PortFast makes a host port forward at once. BPDU Guard err-disables a port that receives any BPDU, so it goes on access ports. Root Guard blocks a port that receives a superior BPDU, to protect root placement. Loop Guard stops a non-designated port from forwarding when BPDUs stop, which protects against one-way links.

**Q: A user plugs a small switch into two wall ports and the VLAN goes down. What happened, and how do you prevent it?**

A: Those two ports probably joined a loop. If they had PortFast without protection, or the small switch dropped BPDUs, STP may not have blocked it. I would shut one port to stop the storm. To prevent it, I would use BPDU Guard on access ports. If the small switch drops BPDUs, BPDU Guard cannot see it, so I would add storm control, which limits broadcast traffic on a port.

**Q: One port on SW3 is blocked. SW1 is root. A link elsewhere fails, and SW3 sees no link change. How long until recovery?**

A: With 802.1D, up to 50 seconds: 20 for Max Age, then 15 Listening and 15 Learning. With RSTP, usually under a second, since SW3 accepts the neighbour's inferior BPDU, then the two use Proposal and Agreement.

**Q: How would you make traffic use both uplinks from an access switch?**

A: With Rapid PVST+, make one distribution switch root for some VLANs, and the other root for the rest. Each uplink then forwards for its own set. Or use MST with two instances. Better still, use EtherChannel, vPC-style link aggregation or a routed access layer, so no link blocks.

## Quick Recap

- Ethernet frames have no TTL, so loops cause broadcast storms, MAC flapping and duplicate frames.
- The lowest Bridge ID (priority, then MAC) becomes root. Priority is a multiple of 4096, plus the VLAN ID.
- Every non-root switch has one root port, every link has one designated port, and everything else blocks. Tie-breakers: root ID, cost, sender BID, sender port ID.
- Classic timers are hello 2, forward delay 15, max age 20, giving 30 to 50 seconds of convergence.
- RSTP adds Alternate and Backup roles, plus Proposal and Agreement, and often converges in under a second on point-to-point links.
- PVST+ and Rapid PVST+ run one tree per VLAN. MST groups VLANs into a few instances inside a region.
- Use PortFast and BPDU Guard on host ports, Root Guard on ports facing switches that must not be root, and Loop Guard on non-designated ports.
