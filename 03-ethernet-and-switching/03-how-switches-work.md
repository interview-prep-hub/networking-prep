# How Switches Work

**Level:** Beginner | **Prerequisites:** [MAC Addresses](01-mac-addresses.md), [Ethernet Frames and Standards](02-ethernet-frames.md)

## In One Line

A switch learns which device sits behind which port by reading the source MAC address of every frame, then uses that table to send each frame only where it needs to go.

## Why It Matters

Inside a network, a host is any device that sends or receives data, such as a laptop, server or printer. All hosts in an office need to talk to each other. The old device for this was the hub. A hub copies every bit out of every port, so every host hears all traffic and only one host can talk at a time.

A switch fixes this. If host A talks to host B, the other hosts see nothing. At the same moment, host C can talk to host D at full speed. A 24-port gigabit switch is not one shared 1 Gbps pipe. Each port gets its own 1 Gbps in each direction.

Interviewers ask about this because many other topics stand on it. ARP, VLANs, spanning tree and Layer 2 attacks all make sense only when you know the four actions a switch can take. "What does a switch do with a frame it has never seen before?" is one of the most common networking questions.

## Simple Explanation

Think of a receptionist in a large hotel. Mail arrives, but on day one she does not know who is in which room.

- A guest in room 12 hands her a letter. She reads the sender's name on the back and writes in her notebook: "Maria, room 12". This is **learning**.
- The letter is for Omar, who is not in her notebook. She makes a copy and slides it under every door. Omar keeps it and everyone else throws it away. This is **flooding**.
- Later, a letter arrives for Maria. The notebook says room 12, so she delivers it to room 12 only. This is **forwarding**.
- A guest in room 12 hands her a letter for his room-mate, also in room 12. She does nothing. They can pass it across the room. This is **filtering**.
- If she does not hear from a guest for five minutes, she erases that line. Guests move rooms, and an old note is worse than no note. This is **ageing**.

**Where the analogy stops:** a real receptionist reads names and thinks. A switch does this in hardware, in a tiny fraction of a second, for millions of frames. It also learns only from frames that pass through it, and it never checks whether the frame was delivered.

## How It Works

### The words you need

- **Frame:** one unit of data on an Ethernet network. It carries a header with addresses, then the data.
- **MAC address:** the 48-bit hardware address of a network card, for example `0011.2233.4401`.
- **Port:** a physical socket on the switch. `Gi0/1` means GigabitEthernet port 0/1 on a Cisco switch.
- **Layer 2:** the data link layer of the [OSI model](../02-network-models/02-osi-model.md). A plain switch makes decisions from MAC addresses. It does not need IP addresses.
- **Unicast:** a frame for one host. **Broadcast:** a frame for all hosts, sent to `ffff.ffff.ffff`. **Multicast:** a frame for a group of hosts.

The official name for this behaviour is **transparent bridging**. "Transparent" means hosts do not know the switch exists. A switch is really a bridge with many ports. The method was first defined in IEEE 802.1D and is now part of IEEE 802.1Q.

### The MAC address table

The **MAC address table** is the switch's notebook. Each row links a MAC address to a port, inside one VLAN. A **VLAN** is a separate virtual network on the same switch. It is covered in [VLANs](06-vlans.md).

| VLAN | MAC address | Type | Port | Timer |
|---|---|---|---|---|
| 1 | 0011.2233.4401 | DYNAMIC | Gi0/1 | 12 s since last seen |
| 1 | 0011.2233.4402 | DYNAMIC | Gi0/2 | 40 s since last seen |

Four facts to remember:

- The table starts empty when the switch boots.
- Most entries are **dynamic**, which means the switch learned them by itself. An admin can add **static** entries, and these do not age out.
- One port can hold many MAC addresses. This is normal on a link to another switch.
- One MAC address in one VLAN points to only one port at a time.

### What the switch reads from a frame

The frame layout is covered in [Ethernet Frames](02-ethernet-frames.md). For switching, three fields matter.

```text
+-------------+-------------+-----------+-----------+---------+
| Destination | Source      | Type      | Payload   | FCS     |
| MAC         | MAC         |           | (data)    |         |
| 6 bytes     | 6 bytes     | 2 bytes   | 46-1500   | 4 bytes |
+-------------+-------------+-----------+-----------+---------+
```

The **destination MAC** decides where the frame goes. The **source MAC** teaches the switch where the sender is. The **FCS** (Frame Check Sequence) is a checksum at the very end that shows whether the frame was damaged.

### The process for every frame

1. A frame arrives on a port. The switch notes the port and the VLAN.
2. **Learn.** The switch reads the source MAC. If it is new, the switch adds a row. If it is already there, the switch resets the timer and updates the port if needed.
3. **Look up.** The switch searches the table for the destination MAC in the same VLAN.
4. **Decide.** It takes one of the actions below.

```text
          Frame arrives on port P in VLAN V
                       |
                       v
       LEARN: source MAC -> port P, VLAN V
              (add entry or reset its timer)
                       |
                       v
           Check the destination MAC
                       |
      +----------------+----------------+
      |                |                |
 Broadcast or     Unicast, NOT     Unicast, IN table
 multicast        in table              |
      |                |          +-----+------+
      v                v          |            |
    FLOOD            FLOOD      Same as P?   Other port?
 (all ports in     (unknown       |            |
  VLAN V           unicast)       v            v
  except P)                     FILTER      FORWARD
                                (drop)      (that port only)
```

### The four actions

| Action | When it happens | What the switch does |
|---|---|---|
| Learn | Every valid frame | Saves source MAC, port and VLAN |
| Flood | Broadcast, multicast, or unknown unicast | Sends out all ports in the VLAN, except the one it came in on |
| Forward | Destination known, on a different port | Sends out that one port |
| Filter | Destination known, on the same port it came in on | Drops the frame |

**Fuller truth on the word "filter":** forwarding out one port also means holding the frame back from all other ports. Books use "filter" for both ideas. In an interview, the expected meaning is: the destination is on the arrival port, so the switch drops the frame.

**Fuller truth on multicast:** a basic switch floods multicast like broadcast. Many switches use IGMP snooping, a feature that watches group membership and sends multicast only to ports that asked for it. See [Multicast Routing](../05-routing/14-multicast-routing.md).

### Unknown unicast

An **unknown unicast** frame is a normal frame for one host, but the switch has no row for that destination MAC. The switch does not drop it, because the host may well exist and be quiet. So the switch floods it and hopes the reply teaches it the location.

Do not mix this up with broadcast. The destination of a broadcast is `ffff.ffff.ffff`, and every host accepts it. The destination of an unknown unicast is one real MAC address. Every other host that gets a copy sees it is not addressed to them and throws it away.

### Ageing

**Ageing** removes rows that have not been used. Each dynamic row has a timer. When a frame arrives with that **source** MAC, the timer goes back to zero. If the timer reaches the limit, the switch deletes the row.

- The Cisco Catalyst default is **300 seconds** (5 minutes). IEEE also recommends 300 seconds, and many other vendors and Linux bridges use the same value. Check your platform, because a default can vary.
- The timer is reset by frames **from** the host, not frames **to** it.
- Without ageing, the table would fill with old entries, and a moved device could stay unreachable for a long time.
- If a host moves and sends a frame, the row moves to the new port at once. The switch does not wait for the timer.
- On Cisco Catalyst switches, when a port goes down, its dynamic entries are removed straight away.

```text
SW1# show mac address-table aging-time
Global Aging Time:  300
Vlan    Aging Time
----    ----------

SW1(config)# mac address-table aging-time 600
```

The range on Catalyst is typically 10-1000000 seconds, and 0 turns ageing off. Older IOS versions spell the command `mac-address-table`, with a second hyphen.

### Store-and-forward vs cut-through

These are two ways of moving a frame from the input port to the output port. Think of a train at a junction. In store-and-forward, the whole train pulls into a station, gets inspected, then leaves. In cut-through, the signal reads only the front of the train, so the engine is already leaving while the last carriage is still coming in.

**Where the analogy stops:** a train is one object. A switch is copying bits, and it can hold several frames in memory at once.

**Store-and-forward:**

1. Receive the whole frame into memory (called the **buffer**).
2. Calculate the checksum and compare it with the FCS.
3. If the frame is damaged, drop it.
4. If it is good, look up the destination and send it.

**Cut-through:**

1. Read until the destination MAC is received. This is the first 6 bytes of the frame, and it is why the destination comes first in the header.
2. Look up the destination.
3. Start sending at once, while the rest of the frame is still arriving.

The delay from storing a frame is its size divided by the link speed:

| What must arrive before sending | At 1 Gbps | At 10 Gbps |
|---|---|---|
| 6 bytes (cut-through) | 0.048 microseconds | 0.0048 microseconds |
| 64-byte frame (store-and-forward) | 0.512 microseconds | 0.0512 microseconds |
| 1518-byte frame (store-and-forward) | 12.144 microseconds | 1.2144 microseconds |

Example: 1518 bytes x 8 = 12,144 bits. At 1,000,000,000 bits per second, that takes 12.144 microseconds. These numbers leave out the time for the lookup and other work inside the switch.

| Topic | Store-and-forward | Cut-through |
|---|---|---|
| When sending starts | After the whole frame arrives | After the destination MAC arrives |
| Delay | Grows with frame size | Low and mostly the same for all frame sizes |
| Damaged frames | Dropped here | Passed on, dropped by a later device |
| Different port speeds | Handled easily | Needs buffering anyway |
| Typical use | Campus and office switches | Many data centre and low-latency switches |

Cut-through only saves time when the output port is free and the port speeds suit it. If the output port is busy, the frame must wait in a buffer. If the input is slower than the output, the switch must buffer too, or it would run out of bits to send in the middle of a frame.

**Fuller truth:** a cut-through switch can still check the FCS when the end of the frame passes. It cannot call the frame back, but it can count the error. Some platforms also mark the outgoing FCS as bad on purpose so the next device drops it.

There is also an older third mode, **fragment-free**. The switch waits for the first 64 bytes, then sends. A **runt** is a frame under 64 bytes, often the broken remains of a collision. Fragment-free drops these. On full-duplex links there are no collisions, so this mode matters only as history and as an interview fact.

### CAM and TCAM

A switch at 10 Gbps can receive over 14 million small frames per second, so a slow search through a list would never keep up. It uses special memory.

Normal memory (**RAM**) works like a street: you give it an address and it tells you what is stored there. To find a value, you would need to check rows one by one.

**CAM (Content Addressable Memory)** works the other way. You give it the value and it tells you the location. Think of a teacher calling a name in a hall: only that student stands up, and the teacher does not walk down each row. In the chip, every row compares itself at the same time, so the search takes about the same time with 10 rows or 10,000.

**Where the analogy stops:** nobody is listening or taking turns. Each row has its own comparison circuit, which is why this memory costs more and uses more power.

A CAM match is exact. Each stored bit is 0 or 1, so it suits a MAC lookup, where a full address must match.

**TCAM (Ternary Content Addressable Memory)** adds a third state. "Ternary" means three values: 0, 1 and X. X means "don't care", so that bit matches anything. This allows matching a range, which is what IP routes and access lists need.

Look up the packet destination 10.1.1.5 (`00001010 00000001 00000001 00000101`):

| Position | Rule | Stored pattern | Match? |
|---|---|---|---|
| 1 | 10.1.1.0/24 | `00001010 00000001 00000001 XXXXXXXX` | Yes |
| 2 | 10.1.0.0/16 | `00001010 00000001 XXXXXXXX XXXXXXXX` | Yes |
| 3 | 0.0.0.0/0 | `XXXXXXXX XXXXXXXX XXXXXXXX XXXXXXXX` | Yes |

All three rows match in the same step, and the chip returns the first one by position. Because longer prefixes are stored first, the result is the longest match. See [Binary and Hexadecimal](../01-foundations/07-binary-and-hexadecimal.md) if these patterns look strange.

| Memory | Search by | Bit values | Common use in a switch |
|---|---|---|---|
| RAM | Address | 0, 1 | Buffers, software, configuration |
| CAM | Content, exact | 0, 1 | MAC address table |
| TCAM | Content, with wildcards | 0, 1, X | ACLs, QoS rules, IP routes on Layer 3 switches |

An **ACL** (access control list) is a list of permit and deny rules. See [ACLs](../09-security/05-access-control-lists.md). **QoS** (quality of service) gives some traffic priority. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

**Fuller truth:** engineers say "CAM table" for the MAC table, but many modern switches build it as a hash table in fast RAM, not as true CAM. A hash turns the address into a row number. The lookup is still fast and the name has stuck. One side effect: if several addresses hash to the same full group of rows, a new address may fail to be learned even when the table is not full overall.

## Example

### Walk-through with four hosts

Switch SW1 has just restarted, so its MAC table is empty. Hosts A and B each have their own port. Hosts C and D share port Gi0/3 through an old hub, which repeats every frame to all its ports. The hub lets us see every action, including filter.

```text
              +---------------------------+
              |         SW1 (VLAN 1)      |
              |  Gi0/1    Gi0/2    Gi0/3  |
              +----+--------+--------+----+
                   |        |        |
                 Host A   Host B   [ Hub ]
                                    /   \
                               Host C   Host D
```

| Host | IP address | MAC address | Reached through |
|---|---|---|---|
| A | 192.168.1.11 | 0011.2233.4401 | Gi0/1 |
| B | 192.168.1.12 | 0011.2233.4402 | Gi0/2 |
| C | 192.168.1.13 | 0011.2233.4403 | Gi0/3 (hub) |
| D | 192.168.1.14 | 0011.2233.4404 | Gi0/3 (hub) |

**Frame 1: A pings B, so it first asks for B's MAC.** A does not know B's MAC, so it sends an [ARP](05-arp.md) request to the broadcast address.

- Source A arrives on Gi0/1, so the switch learns A.
- The destination is broadcast, so it floods out Gi0/2 and Gi0/3.
- B, C and D all receive it. Only B answers.

**Frame 2: B sends the ARP reply to A.**

- Source B arrives on Gi0/2, so the switch learns B.
- Destination A is in the table on Gi0/1, so it forwards out Gi0/1 only.

**Frame 3: A sends the ping to B.**

- Source A is known, so its timer resets.
- Destination B is on Gi0/2, so it forwards out Gi0/2 only. C and D see nothing.

**Frame 4: A sends a frame to C.** Assume A kept C's MAC in its ARP cache from before the switch restarted. So A sends a normal unicast with no ARP first.

- Source A is known, so its timer resets.
- Destination C is not in the table. This is **unknown unicast**, so the switch floods out Gi0/2 and Gi0/3.
- B sees that the frame is not addressed to it and discards it. The hub gives a copy to C and D. C accepts it and D discards it.

**Frame 5: C replies to A.**

- Source C arrives on Gi0/3, so the switch learns C.
- Destination A is on Gi0/1, so it forwards out Gi0/1 only.

**Frame 6: D asks for C's MAC.** D sends an ARP broadcast. The hub repeats it to C and to Gi0/3.

- Source D arrives on Gi0/3, so the switch learns D.
- The switch floods out Gi0/1 and Gi0/2, but not back out Gi0/3.

**Frame 7: C sends the ARP reply to D.** The hub repeats it to D and to Gi0/3.

- Source C is known, so its timer resets.
- Destination D is on Gi0/3, the same port the frame came in on. D already has the frame from the hub, so the switch **filters** it.

| # | Frame (source to destination) | In port | Learned | Action | Out ports |
|---|---|---|---|---|---|
| 1 | A to broadcast | Gi0/1 | A | Flood | Gi0/2, Gi0/3 |
| 2 | B to A | Gi0/2 | B | Forward | Gi0/1 |
| 3 | A to B | Gi0/1 | none, timer reset | Forward | Gi0/2 |
| 4 | A to C | Gi0/1 | none, timer reset | Flood (unknown unicast) | Gi0/2, Gi0/3 |
| 5 | C to A | Gi0/3 | C | Forward | Gi0/1 |
| 6 | D to broadcast | Gi0/3 | D | Flood | Gi0/1, Gi0/2 |
| 7 | C to D | Gi0/3 | none, timer reset | Filter | none |

The table now looks like this. Port Gi0/3 has two MAC addresses, which is correct. (The `dynamic` keyword hides built-in entries for the switch's own CPU.)

```text
SW1# show mac address-table dynamic
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0011.2233.4401    DYNAMIC     Gi0/1
   1    0011.2233.4402    DYNAMIC     Gi0/2
   1    0011.2233.4403    DYNAMIC     Gi0/3
   1    0011.2233.4404    DYNAMIC     Gi0/3
Total Mac Addresses for this criterion: 4
```

**What ageing does next.** Suppose B stays silent for 300 seconds while the others keep talking. The switch removes B's row. The next frame to B is unknown unicast and gets flooded. B replies, and the switch learns it again. Nothing breaks, and a few frames go to extra places.

**Note on the hub:** a hub forces its switch port into half duplex, which brings back collisions. See [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md). Hubs are rare today, and one appears here only to make filtering easy to see. In modern networks, filtering happens mostly on links between switches, when a neighbour floods a frame back toward the side where the host already is.

### Two switches

A switch learns only "which port leads to this host", not which switch the host is on.

```text
 Host A ---Gi0/1 [SW1] Gi0/24---Gi0/24 [SW2] Gi0/1--- Host E
                                             Gi0/2--- Host F
```

After all hosts have talked, SW1 lists A on Gi0/1, and both E and F on Gi0/24. SW2 lists E on Gi0/1, F on Gi0/2, and A on Gi0/24.

### Find which port a host is on

This is a real daily task. First get the MAC address from the host.

Linux:

```bash
ip -br link show eth0
# eth0   UP   00:11:22:33:44:02 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

Windows (PowerShell):

```powershell
Get-NetAdapter -Name Ethernet | Select-Object Name, MacAddress
# Name     MacAddress
# ----     ----------
# Ethernet 00-11-22-33-44-02
```

Then ask the switch, using Cisco's dotted format (Cisco IOS):

```text
SW1# show mac address-table address 0011.2233.4402
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0011.2233.4402    DYNAMIC     Gi0/2
Total Mac Addresses for this criterion: 1
```

Other useful Cisco IOS commands:

```text
SW1# show mac address-table interface Gi0/3
SW1# show mac address-table count
SW1# clear mac address-table dynamic
SW1(config)# mac address-table static 0011.2233.4402 vlan 1 int Gi0/2
```

A host cannot show you the switch's table on Windows or Linux, unless that Linux machine is itself acting as a bridge.

## Going Deeper

### When the table is full

Tables have a limit. It can be a few thousand entries on a small switch and far more on a data centre switch, so check the datasheet. When the table is full, the switch cannot learn new addresses, and frames to those hosts are flooded as unknown unicast. An attacker can do this on purpose by sending frames with thousands of fake source MACs, then read the flooded traffic. See [Layer 2 Security](11-layer2-security.md).

### Loops and MAC flapping

Flooding plus a physical loop is dangerous. Ethernet frames have no hop limit, so a flooded frame can circle forever. [Spanning Tree](09-spanning-tree.md) blocks loops for this reason. A sign of a loop is the same MAC appearing on two ports in turn:

```text
%SW_MATM-4-MACFLAP_NOTIF: Host 0011.2233.4401 in vlan 1 is
flapping between port Gi0/1 and port Gi0/2
```

Spanning tree also changes ageing. With classic 802.1D, a topology change shortens the ageing time to the forward delay (15 seconds by default) for a while, so old entries clear faster.

### Timers that do not match

On Cisco IOS routers, the ARP cache timeout defaults to 4 hours, while the switch MAC timer is 5 minutes. Imagine a host that only receives data, such as a logging server. The router still knows its MAC, but the switch ages out the row because the host sends nothing. Now every frame to that server is flooded to the whole VLAN. The usual fixes are to lower the ARP timeout below the MAC ageing time, or to raise the MAC ageing time.

### Speed and congestion

A **non-blocking** switch can carry full traffic on all ports at once. A 24-port 1 Gbps switch needs 48 Gbps of capacity for that, because each port sends and receives. Non-blocking does not mean no drops. If two 1 Gbps ports send to one 1 Gbps port at full speed, the output buffer fills and frames are dropped. These appear as output drops in `show interfaces`.

### A Linux bridge is a software switch

Linux can do the same job in software. The Docker default network uses this. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).

```bash
# Linux
sudo ip link add br0 type bridge
sudo ip link set eth1 master br0
sudo ip link set eth2 master br0
sudo ip link set br0 up

bridge fdb show br br0 | grep -v permanent
# 00:11:22:33:44:01 dev eth1 master br0
# 00:11:22:33:44:02 dev eth2 master br0

ip -d link show br0 | grep -o 'ageing_time [0-9]*'
# ageing_time 30000
```

The value is in hundredths of a second, so 30000 means 300 seconds.

## Common Mistakes

- **"A switch learns from the destination MAC."** It learns only from the source. The destination is used for the lookup.
- **"Unknown unicast is the same as broadcast."** Both are flooded, but an unknown unicast names one real host, and other hosts discard it.
- **"A switch drops frames for unknown MACs."** It floods them. Dropping would break quiet hosts.
- **"Frames sent to a host keep its entry alive."** The timer resets on frames from that host. A silent host ages out.
- **"A flood goes out every port."** It goes out every port in that VLAN, except the arrival port.
- **"One port means one MAC."** A port to another switch, a hub or a virtual machine host can carry many MACs.
- **"The switch changes the MAC addresses."** A Layer 2 switch keeps them as they are. A router is the device that writes a new Layer 2 header at each hop.
- **"The MAC table and ARP table are the same."** The MAC table maps MAC to port, on a switch. The ARP table maps IP to MAC, on a host or router.
- **"Cut-through is always better."** It saves very little on busy links and it passes on damaged frames.
- **"TCAM stores the MAC table."** On most designs, MAC lookups use exact-match memory. TCAM is for wildcard matches, such as ACLs and routes.
- **"Switches stop broadcasts."** A switch floods broadcasts through the VLAN. Routers and VLAN boundaries stop them.

## Interview Questions

**Q: How does a switch build its MAC address table?**
A: It reads the source MAC of every incoming frame and records that MAC with the arrival port and VLAN. If the MAC is already listed, it resets the ageing timer. It builds the table from live traffic, with no setup needed.

**Q: What does a switch do when it does not know the destination MAC?**
A: It treats the frame as unknown unicast and floods it out all ports in the same VLAN, except the arrival port. When the real host replies, the switch learns its port and later frames go out that port only.

**Q: What are the possible actions for a frame?**
A: The switch always learns the source. For the destination, it floods broadcast, multicast and unknown unicast. It forwards a known unicast out one port. It filters the frame if the destination is on the arrival port.

**Q: What is the default MAC ageing time on a Cisco switch, and why have one?**
A: The default is 300 seconds. Ageing removes old entries so the table has space and stale locations disappear. The timer resets each time the switch sees a frame from that source MAC.

**Q: What happens if a host moves to another port?**
A: As soon as the host sends a frame, the switch sees its source MAC on the new port and updates the row. If the host stays quiet, frames still go to the old port until ageing or link-down clears the row. On Catalyst, unplugging the old port clears the row at once.

**Q: What is the difference between store-and-forward and cut-through?**
A: Store-and-forward receives the entire frame, checks the FCS, and drops it if damaged. Cut-through reads only the destination MAC and starts sending immediately. Cut-through has lower delay, but it can pass on bad frames, and it still has to buffer when the output is busy.

**Q: What is the difference between CAM and TCAM?**
A: CAM does an exact match on 0s and 1s, which suits MAC lookups. TCAM adds a third "don't care" state, so it can match ranges. That makes it the choice for ACLs and IP routes. TCAM is more expensive and uses more power.

**Q: What happens when the MAC table is full?**
A: The switch cannot learn new MACs, so frames to those hosts are flooded. Existing entries still work until they age out. Attackers cause this with MAC flooding, and port security is a common defence.

**Q: Can one port have multiple MAC addresses? Can one MAC be on multiple ports?**
A: One port can have many MACs, for example on a link to another switch. One MAC in one VLAN maps to one port at a time. If it keeps moving between two ports, that is MAC flapping, and it often means a loop.

**Q: What is the difference between a MAC table and an ARP table?**
A: The MAC table lives on a switch and maps a MAC and VLAN to a port. The ARP table lives on hosts and routers and maps an IP address to a MAC. The two work together, but they answer different questions.

**Q: How is a switch different from a hub?**
A: A hub repeats every bit out of every port and works at Layer 1. A switch reads the frame at Layer 2 and sends it only where needed. So hosts on a switch do not share bandwidth, and each full-duplex port can send and receive at the same time.

**Q: Host A pings Host B for the first time on a new switch. What are the first frames?**
A: A sends an ARP broadcast, the switch learns A and floods it. B sends an ARP reply, the switch learns B and forwards it to A only. After that, the ping and its reply go as known unicast between the two ports.

## Quick Recap

- A switch learns from the **source** MAC and decides using the **destination** MAC.
- Flood for broadcast, multicast and unknown unicast. Forward a known unicast. Filter when the destination is on the arrival port.
- A flood stays inside the VLAN and never goes back out the arrival port.
- Dynamic entries age out after 300 seconds by default on Cisco, and frames from the host reset the timer.
- Store-and-forward checks the whole frame. Cut-through starts after the destination MAC, so delay is lower but bad frames can pass.
- CAM is exact match for MAC lookups. TCAM adds "don't care" bits for ACLs and routes.
- `show mac address-table` is the first command to run when you need to find a host.
