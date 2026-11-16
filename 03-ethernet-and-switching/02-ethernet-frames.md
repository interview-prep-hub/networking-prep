# Ethernet Frames and Standards

**Level:** Intermediate | **Prerequisites:** [MAC Addresses](01-mac-addresses.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md)

## In One Line

An Ethernet frame is the fixed-format package that carries data across one local link: 6 bytes of destination, 6 of source, 2 of type, 46-1500 of payload, and 4 of error check, so 64-1518 bytes in total.

## Why It Matters

Two computers on a cable only exchange bits. A bit is a single 0 or 1. The receiver needs to know where the data starts, who it is for, what kind of data it is, and whether it arrived damaged. The Ethernet frame answers all four questions.

Nearly every wired network in the world uses this format. So do most data centres and the wired side of every Wi-Fi access point.

Interviewers ask about it because the numbers explain real faults:

- Why does a small ping work but a big file transfer hang? Often an MTU (maximum transmission unit) problem.
- Why is a link "up" but painfully slow? Often a duplex mismatch.
- Why is a gigabit link only delivering about 940 Mb/s? Frame overhead.
- Why is 64 bytes the minimum? The answer is a piece of history called CSMA/CD.

## Simple Explanation

Think of a frame as a parcel sent between two desks in the same office.

| Parcel | Ethernet frame |
|---|---|
| Courier knocks and says "parcel coming" | Preamble and SFD |
| "To" label | Destination MAC address |
| "From" label | Source MAC address |
| "Contents: for the IPv4 team" sticker | EtherType |
| The goods inside | Payload |
| Seal that shows if the box was opened or crushed | FCS (error check) |

A MAC address is the hardware address of a network card, such as `00:00:5e:00:53:01`. A network card is also called a NIC (network interface card). The chapter uses addresses from `00:00:5e:00:53:xx`, a range set aside for documentation.

**Where the analogy stops:** a parcel keeps its label for the whole trip. A frame does not. A frame only lives for one network segment. Each router removes the old frame and builds a new one with new MAC addresses, while the IP packet inside travels on. See [How Routers Work](../05-routing/01-how-routers-work.md).

## How It Works

### The frame layout

```text
+----------+-----+------+------+-----------+----------+-----+
| Preamble | SFD | Dest | Src  | EtherType | Payload  | FCS |
| 7 bytes  | 1   | 6    | 6    | 2         | 46-1500  | 4   |
+----------+-----+------+------+-----------+----------+-----+
 \____ not counted ___/ \_____ counted in frame size ______/
                         (64 minimum, 1518 maximum)
```

The layout above is called Ethernet II. It is the format used by IPv4, IPv6 and ARP on almost all networks today.

| Field | Size | What it does |
|---|---|---|
| Preamble | 7 bytes | Pattern `10101010` repeated. Lets the receiver lock on to the sender's timing. |
| SFD (Start Frame Delimiter) | 1 byte | `10101011`. The final `11` says "the real frame starts at the next bit". |
| Destination MAC | 6 bytes | Who should accept the frame. Can be unicast, multicast or broadcast. |
| Source MAC | 6 bytes | Who sent it. Switches learn from this field. |
| EtherType | 2 bytes | Names the protocol inside the payload. |
| Payload | 46-1500 bytes | The data, usually an IP packet. Padded if too short. |
| FCS (Frame Check Sequence) | 4 bytes | A CRC-32 value used to detect damage. |

The destination comes first for a reason. A switch can read it and start forwarding before the rest of the frame arrives. See [How Switches Work](03-how-switches-work.md).

**Frame or packet?** The IEEE (the body that writes the Ethernet standards, in its 802.3 working group) calls the frame the part from destination to FCS. Preamble and SFD are outside it. That is why they are not in the 64 and 1518 numbers.

### Common EtherType values

| EtherType | Protocol |
|---|---|
| `0x0800` | IPv4 |
| `0x0806` | ARP |
| `0x86DD` | IPv6 |
| `0x8100` | 802.1Q VLAN tag |
| `0x88A8` | 802.1ad outer tag (Q-in-Q) |
| `0x8847` | MPLS unicast |
| `0x88CC` | LLDP |
| `0x888E` | 802.1X (EAPOL) |
| `0x8809` | Slow protocols, including LACP |
| `0x8863`, `0x8864` | PPPoE discovery and session |

The `0x` prefix means the number is in hexadecimal.

The same two bytes can mean something else. If the value is 1500 (`0x05DC`) or less, it is a length, and the frame uses the older IEEE 802.3 format. If it is 1536 (`0x0600`) or more, it is an EtherType. See "Ethernet II versus 802.3 LLC" in Going Deeper.

### How the frame sizes add up

Frame size counts from destination to FCS.

```text
Header:  6 (dest) + 6 (src) + 2 (type) = 14 bytes
Trailer: 4 (FCS)                       =  4 bytes
Overhead                               = 18 bytes

Minimum:  18 + 46 payload   =   64 bytes
Maximum:  18 + 1500 payload = 1518 bytes
```

If the data is shorter than 46 bytes, the sender adds padding. Padding is zero bytes that fill the gap. It has no meaning. For example, an ARP message is 28 bytes, so the sender adds 18 bytes of padding.

### What happens when a frame is sent

1. The operating system gives the NIC a packet, such as an IPv4 packet.
2. The NIC (with help from the driver) adds destination MAC, source MAC and EtherType.
3. If the payload is under 46 bytes, it adds padding.
4. It calculates a CRC-32 over destination, source, type and payload, and places the result in the FCS field.
5. It sends the preamble and SFD, then the frame, as signals on the cable.
6. It stays quiet for the interframe gap, at least 96 bit times (12 bytes of silence). A bit time is how long one bit takes at that speed: 96 ns of gap at 1 Gb/s.

### What happens when a frame is received

1. The receiver sees the preamble and locks on to the timing.
2. It sees the SFD and starts reading the frame.
3. It checks the destination MAC. If the frame is not for this NIC, a broadcast, or a multicast group it has joined, the NIC ignores it. (Capture tools can turn on promiscuous mode, which keeps everything.)
4. It recalculates the CRC-32. If the result does not match the FCS, the frame is dropped and an error counter goes up.
5. It reads the EtherType and passes the payload to the correct protocol.

Notice what is missing. Ethernet has no acknowledgement and no retransmission. A damaged frame is simply dropped. If the data must arrive, a higher layer such as [TCP](../06-transport-layer/05-tcp-reliability-and-flow-control.md) resends it.

### The 802.1Q VLAN tag: 1522 bytes

A VLAN (virtual LAN) splits one switch into separate logical networks. To mark which VLAN a frame belongs to, 802.1Q inserts a 4-byte tag between the source MAC and the EtherType.

```text
+-----+-----+------+-----+--------+-----------+---------+-----+
| Pre | SFD | Dest | Src | 802.1Q | EtherType | Payload | FCS |
| 7   | 1   | 6    | 6   | 4      | 2         | 46-1500 | 4   |
+-----+-----+------+-----+--------+-----------+---------+-----+
```

The tag itself has this layout:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-----+-+-----------------------+
|         TPID (0x8100)         | PCP |D|     VID (VLAN ID)     |
+-------------------------------+-----+-+-----------------------+
```

- **TPID (Tag Protocol Identifier):** `0x8100`. It sits where the EtherType would be, so a receiver knows a tag is here.
- **PCP (Priority Code Point):** 3 bits of priority for [QoS](../10-enterprise-and-datacenter/03-quality-of-service.md).
- **DEI (Drop Eligible Indicator):** 1 bit, shown as D. It marks a frame that may be dropped first under pressure.
- **VID (VLAN Identifier):** 12 bits, so 4096 values. VLANs 0 and 4095 are reserved, which leaves 1-4094.

The payload limit does not change. A tagged frame carries the same 1500 bytes, so the maximum grows from 1518 to 1522. The tag inserts bytes, so the FCS must be recalculated when a tag is added or removed. More in [Trunking and 802.1Q](07-trunking-8021q.md).

| Frame kind | Overhead | Max payload | Max frame |
|---|---|---|---|
| Basic Ethernet | 18 | 1500 | 1518 |
| One 802.1Q tag | 22 | 1500 | 1522 |
| Q-in-Q (two tags) | 26 | 1500 | 1526 |

Q-in-Q is common at service providers, and links that carry it must be set to allow the bigger frame. The 802.3 standard also defines "envelope frames" up to 2000 bytes, which leave room for extra headers. The payload limit is still 1500.

### MTU 1500

MTU (Maximum Transmission Unit) is the largest payload a link will carry in one frame. On Ethernet the default is 1500 bytes.

The MTU does **not** count the Ethernet header or FCS. So MTU 1500 means a full frame of 1518 bytes.

```text
|<----------------- frame: 1518 bytes ----------------->|
+-----------+-------------------------------------+-----+
| Header 14 | IP packet: up to 1500 bytes (MTU)   | FCS |
+-----------+-------------------------------------+-----+
```

Each extra header inside the frame eats into that 1500:

| Case | Usable size |
|---|---|
| Plain IPv4 packet | 1500 |
| TCP data in IPv4 (no options) | 1460 (1500 - 20 - 20) |
| ICMP ping data in IPv4 | 1472 (1500 - 20 - 8) |
| IP MTU over PPPoE | 1492 (PPPoE takes 8) |

Why 1500? It was a design trade-off when Ethernet was a shared wire and memory was expensive. A longer frame would hold the wire longer and need bigger buffers in every card. The number stayed because everything was built to it.

MSS (Maximum Segment Size) and Path MTU Discovery live in [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md). IP fragmentation lives in [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

### Jumbo frames

A jumbo frame is any Ethernet frame with a payload above 1500 bytes. The usual MTU is 9000. Many switches accept frames of about 9216 bytes, so 9000-byte hosts fit with room to spare.

**Jumbo frames are not part of the IEEE 802.3 standard.** They are an industry practice, and the maximum size varies by vendor.

Why use them:

- **Less CPU work:** one 9000-byte frame replaces about six 1500-byte frames, so fewer frames to process. Modern NIC offloads already combine segments, so this gain is smaller than it used to be.
- **Slightly more usable bandwidth:** about 99% versus about 95% of line rate for TCP.
- **Room for tunnels:** [VXLAN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) adds about 50 bytes, so the underlying network needs an MTU of at least 1550 to carry a normal 1500-byte packet.

Where you see them: storage networks (iSCSI, NFS), VM migration, backup networks, and data centre fabrics. For example, AWS instances inside a VPC support an MTU of 9001 on current instance types. Traffic on the public internet stays around 1500.

The risk: **every device on the Layer 2 path must allow the size.** A switch cannot fragment a frame. If a frame is bigger than the port allows, the switch drops it and may count it as a "giant". Nobody tells the sender. So small pings work and large transfers hang.

### CSMA/CD

Early Ethernet used one shared cable. Later it used a hub, a simple device that repeats every signal to all ports. In both cases only one station could send at a time. If two sent together, their signals mixed and both were ruined. That event is a **collision**.

CSMA/CD (Carrier Sense Multiple Access with Collision Detection) is the rule for sharing the medium.

- **Carrier Sense:** listen before you talk.
- **Multiple Access:** many stations share the medium.
- **Collision Detection:** keep listening while you talk.

Analogy: people talking in a dark room. You wait for silence, then speak. If someone else starts too, you both stop, wait a random moment, and try again. **Where it stops:** people can often make out two voices at once. A collision destroys both frames completely.

The steps:

1. Have a frame to send. Listen to the medium.
2. If busy, wait until it is idle, plus the interframe gap.
3. Start sending, and keep listening at the same time.
4. If no collision occurs while sending, the frame is done.
5. If a collision occurs, send a short 32-bit jam signal so everyone notices.
6. Pick a random backoff. After collision number n, choose a random k from 0 to 2^n - 1 (n stops growing at 10). Wait k slot times.
7. Go to step 1. After 16 failed attempts, give up and discard the frame.

A slot time is 512 bit times. At 10 Mb/s that is 51.2 microseconds.

### Why 64 bytes is the minimum

A sender can only detect a collision while it is still sending. If it finishes first, it assumes success and forgets the frame. So a frame must last long enough for news of a collision to get back from the far end of the network.

```text
Station A                                       Station B
    |                                               |
    |==== A's first bit travels =================>  |
    |                                               | B sends just
    |                                         XXXX  | before A's bit
    |                                 collision     | arrives
    |  <============= collision news travels back   |
    |                                               |
    A must STILL be sending when this arrives.
```

The designers chose 512 bits (64 bytes) as the worst-case round trip. At 10 Mb/s, that is 51.2 microseconds of sending, which covered the maximum network size of about 2500 m including repeaters.

So the 64-byte rule exists for collision detection, not because a frame needs 64 bytes of content.

**Fuller truth:** at 100 Mb/s, 64 bytes lasts one tenth as long, so the collision domain shrank to about 200 m. At 1 Gb/s half duplex it would shrink to almost nothing, so gigabit half duplex extends the slot to 512 bytes using carrier extension (extra symbols added after short frames). Nobody really uses gigabit half duplex.

### Why CSMA/CD no longer matters

**Half duplex** means a device can send or receive, but not both at once. **Full duplex** means it can do both at the same time.

A modern port connects one device to one switch port, using separate paths in each direction.

```text
Half duplex with a hub            Full duplex with a switch

 A ---+                            A ===tx===> [switch port 1]
      |                            A <==rx==== [switch port 1]
 B ---+--- [HUB]
      |                            Only two talkers on the link.
 C ---+                            Each direction has its own path.
 One shared collision domain.      No collisions are possible.
```

With only one sender per direction, nothing can collide. So in full duplex:

- CSMA/CD is turned off.
- Devices send whenever they have a frame, keeping the interframe gap.
- A link's bandwidth is available in both directions at once. So 1 Gb/s full duplex can carry 1 Gb/s each way.
- If traffic arrives faster than a switch port can send, the switch queues it in a buffer. When a buffer fills, frames are dropped. Those are drops, not collisions.

An analogy: half duplex is a one-lane bridge where cars must take turns. Full duplex is a two-lane road. **Where it stops:** a real road can still have crashes. A full-duplex link cannot have collisions at all.

Higher speeds removed half duplex entirely. 10 Gigabit Ethernet and everything above it, plus 2.5G and 5G, are full duplex only.

Why keep the 64-byte minimum then? Compatibility. Every chip, tool and standard assumes it, and changing it would gain very little.

**Fuller truth:** CSMA/CD is not quite dead. It is still in the standard for old 10 and 100 Mb/s links. Also, 10BASE-T1S, a newer shared single-pair Ethernet for cars and factories, can run half duplex on a shared bus. Wi-Fi uses a different method, CSMA/CA, covered in [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md). The collision domain idea is covered in [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md).

### Reading Ethernet standard names

```text
   1000 BASE - T
    |     |    |
    |     |    +-- medium: T = twisted pair copper
    |     +------- baseband (one signal on the medium)
    +------------- speed in Mb/s

   100G BASE - S R 4
    |          | | |
    |          | | +-- number of lanes (parallel paths)
    |          | +---- coding family (R = 64B/66B or similar)
    |          +------ reach (S = short wavelength)
    +----------------- speed
```

| Letter | Meaning |
|---|---|
| T | Twisted pair copper |
| S | Short wavelength light (850 nm), multimode fibre |
| L | Long wavelength (1310 nm), usually single-mode fibre, about 10 km |
| E | Extra long wavelength (1550 nm), about 40 km |
| D | About 500 m on single-mode fibre |
| F | About 2 km on single-mode fibre |
| C | Short copper cable, twinax (direct attach) |
| K | Backplane inside a chassis |

Multimode fibre is for short runs in a building. Single-mode fibre is for long runs. More in [Cables and Transmission Media](../01-foundations/05-cables-and-media.md).

### Ethernet standards table

Years are when the IEEE amendment was approved. Reach values are the standard's targets, and they depend on the exact cable grade.

| Name | Speed | IEEE | Year | Medium | Reach |
|---|---|---|---|---|---|
| 10BASE5 | 10 Mb/s | 802.3 | 1983 | Thick coax | 500 m |
| 10BASE2 | 10 Mb/s | 802.3a | 1985 | Thin coax | 185 m |
| 10BASE-T | 10 Mb/s | 802.3i | 1990 | Cat3 copper | 100 m |
| 100BASE-TX | 100 Mb/s | 802.3u | 1995 | Cat5 copper, 2 pairs | 100 m |
| 100BASE-FX | 100 Mb/s | 802.3u | 1995 | Multimode fibre | 2 km |
| 1000BASE-SX | 1 Gb/s | 802.3z | 1998 | Multimode fibre | 220-550 m |
| 1000BASE-LX | 1 Gb/s | 802.3z | 1998 | Single-mode fibre | 5 km |
| 1000BASE-T | 1 Gb/s | 802.3ab | 1999 | Cat5e copper, 4 pairs | 100 m |
| 10GBASE-SR | 10 Gb/s | 802.3ae | 2002 | Multimode fibre | 300 m on OM3, 400 m on OM4 |
| 10GBASE-LR | 10 Gb/s | 802.3ae | 2002 | Single-mode fibre | 10 km |
| 10GBASE-ER | 10 Gb/s | 802.3ae | 2002 | Single-mode fibre | 40 km |
| 10GBASE-T | 10 Gb/s | 802.3an | 2006 | Cat6a copper | 100 m (Cat6 about 55 m) |
| 40GBASE-SR4 | 40 Gb/s | 802.3ba | 2010 | Multimode, 4 lanes | 100 m on OM3, 150 m on OM4 |
| 40GBASE-LR4 | 40 Gb/s | 802.3ba | 2010 | Single-mode, 4 wavelengths | 10 km |
| 100GBASE-LR4 | 100 Gb/s | 802.3ba | 2010 | Single-mode, 4 wavelengths | 10 km |
| 100GBASE-SR4 | 100 Gb/s | 802.3bm | 2015 | Multimode, 4 lanes | 70 m on OM3, 100 m on OM4 |
| 25GBASE-SR | 25 Gb/s | 802.3by | 2016 | Multimode fibre | 70 m on OM3, 100 m on OM4 |
| 2.5GBASE-T | 2.5 Gb/s | 802.3bz | 2016 | Cat5e copper | 100 m |
| 5GBASE-T | 5 Gb/s | 802.3bz | 2016 | Cat6 copper (often Cat5e) | 100 m |
| 400GBASE-DR4 | 400 Gb/s | 802.3bs | 2017 | Single-mode, 4 lanes | 500 m |
| 400GBASE-FR8 | 400 Gb/s | 802.3bs | 2017 | Single-mode, 8 wavelengths | 2 km |
| 400GBASE-LR8 | 400 Gb/s | 802.3bs | 2017 | Single-mode, 8 wavelengths | 10 km |
| 400GBASE-SR8 | 400 Gb/s | 802.3cm | 2020 | Multimode, 8 lanes | 100 m on OM4 |

Notes:

- 800 Gb/s Ethernet is defined by IEEE 802.3df (2024). Work on 1.6 Tb/s is under the 802.3dj project.
- Names like Fast Ethernet (100 Mb/s) and Gigabit Ethernet (1 Gb/s) are common nicknames.
- Longer-reach "ZX" 1G optics are vendor products, not an IEEE standard.

### Autonegotiation

Autonegotiation lets two ends of a copper link agree on the best speed and duplex automatically.

Analogy: two people meet, and each lists the languages they speak. They pick the best one they share. **Where it stops:** people can recover from a wrong guess by talking more. A link with wrong settings just performs badly with no warning.

How it works on twisted pair (defined with 802.3u for 10/100, extended for 1000BASE-T):

1. When the link starts, each side sends FLPs (fast link pulses), short bursts of pulses that carry a list of its abilities.
2. Each side reads the other's list and the two lists are compared.
3. Both pick the highest common mode. Faster wins over slower, and full duplex wins over half duplex.
4. For 1000BASE-T, the two sides also decide master and slave, which sets who provides the timing.
5. The link comes up.

The result is the same on both sides, because both apply the same priority order.

**If only one side negotiates**, the other side sends no FLPs. The negotiating side falls back to parallel detection:

- It can sense the speed from the signal it hears (10 or 100 Mb/s).
- It **cannot** sense the duplex.
- The standard says it must assume half duplex.

Vendors can differ at the edges. Cisco documents half duplex as the fallback at 10 and 100 Mb/s, and full duplex at 1000 Mb/s, when negotiation fails.

| End A | End B | Result |
|---|---|---|
| auto | auto | Best common mode. Correct. |
| forced 100 full | forced 100 full | Works. Both must stay in sync forever. |
| forced 100 half | auto | Works at 100 half. Slow but matched. |
| forced 100 full | auto | **Duplex mismatch.** B picks 100 half. |
| forced 100 | forced 10 | **Link stays down.** Speed mismatch. |

Current best practice is to leave both ends on auto. The old habit of hard-coding every port came from the 1990s, when early products did not always negotiate correctly. It is now more likely to cause a fault than to fix one.

### Duplex mismatch

A duplex mismatch means one end runs full duplex and the other runs half duplex. The link is up. Lights are green. Small traffic works. Under load it collapses.

Analogy: one person on a walkie-talkie who stops whenever they hear a voice, and one person on a phone who never stops talking. The first keeps backing off. The second never notices. **Where it stops:** humans would ask "hello, are you there?". Ethernet does not.

What happens on the wire:

1. The full-duplex side sends whenever it wants. It does not listen first.
2. The half-duplex side is sending at the same moment. It sees incoming signal and treats it as a collision.
3. It stops, sends jam, and backs off. Its frame is cut short.
4. The full-duplex side receives a broken, short frame.
5. If the half-duplex side is more than 64 bytes into its frame when this happens, it is a **late collision**. Late collisions are not resent by Ethernet, so the frame is lost and TCP must recover.

Each side sees different errors.

| Side | Counters that rise |
|---|---|
| Half-duplex side | Collisions, late collisions |
| Full-duplex side | CRC/FCS errors, runts |

A **runt** is a frame under 64 bytes. A **giant** is a frame over the allowed maximum. On a full-duplex link, a healthy network should show zero collisions.

A speed mismatch is different. If two ends use different speeds, the signals do not match at all, and the link does not come up.

## Example

### Example 1: an ARP request, byte by byte

Host `192.168.1.10` with MAC `00:00:5e:00:53:01` wants the MAC of `192.168.1.1`. It sends an ARP request as a broadcast. See [ARP](05-arp.md).

Linux capture on the sender:

```bash
sudo tcpdump -i eth0 -e -n -XX arp
```

```text
10:15:02.114233 00:00:5e:00:53:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.1.1 tell 192.168.1.10, length 28
    0x0000:  ffff ffff ffff 0000 5e00 5301 0806 0001
    0x0010:  0800 0604 0001 0000 5e00 5301 c0a8 010a
    0x0020:  0000 0000 0000 c0a8 0101
```

Read the bytes:

| Bytes | Value | Meaning |
|---|---|---|
| 1-6 | `ffff ffff ffff` | Destination: broadcast |
| 7-12 | `0000 5e00 5301` | Source MAC |
| 13-14 | `0806` | EtherType: ARP |
| 15-42 | rest | 28-byte ARP message. `c0a8010a` is 192.168.1.10, `c0a80101` is 192.168.1.1 |

Now count the frame.

```text
Header                   14
ARP message              28
                        ---
Captured on sender       42
Padding                  18   (to reach 46 bytes of payload)
FCS                       4
                        ---
Frame on the wire        64   minimum size, exactly
Preamble + SFD            8
Interframe gap           12
                        ---
Total time on the wire   84 byte times
```

Capture the same frame on the receiver, and it shows `length 60`: the 42 bytes plus 18 bytes of padding. The padded version looks like this:

```text
    0x0020:  0000 0000 0000 c0a8 0101 0000 0000 0000
    0x0030:  0000 0000 0000 0000 0000 0000
```

Why does the capture never show 64? Capture tools normally do not see the FCS, because the NIC checks and removes it. Why does the sender show 42? The capture copy is taken before the NIC adds padding and FCS.

### Example 2: why a gigabit link gives about 941 Mb/s

A TCP download fills each frame. What fraction of the line is real data?

```text
TCP data                      1460    (1448 with timestamps)
TCP header                      20    (32 with timestamps)
IP header                       20
                              ----
IP packet (MTU)               1500
Ethernet header + FCS           18
                              ----
Frame                         1518
Preamble + SFD + gap            20
                              ----
Total line time               1538
```

| Case | Efficiency | At 1 Gb/s |
|---|---|---|
| MTU 1500, no TCP options | 1460 / 1538 = 94.9% | 949 Mb/s |
| MTU 1500, TCP timestamps | 1448 / 1538 = 94.1% | 941 Mb/s |
| MTU 9000, no TCP options | 8960 / 9038 = 99.1% | 991 Mb/s |

Linux uses TCP timestamps by default, so iperf3 on a clean gigabit link usually reports about 941 Mb/s. The link is not faulty. That is the overhead.

### Example 3: finding a duplex mismatch

Users say a file server at `10.10.20.50` is slow. Ping shows almost no loss. A switch check shows half duplex.

Cisco IOS, on the switch:

```text
Switch# show interfaces status

Port      Name       Status     Vlan Duplex  Speed Type
Gi1/0/10  app-srv    connected  20   a-full a-1000 10/100/1000BaseTX
Gi1/0/11  file-srv   connected  20   a-half  a-100 10/100/1000BaseTX
```

The `a-` prefix means the value was negotiated or detected. A half-duplex server port is a red flag on a modern network.

```text
Switch# show interfaces GigabitEthernet1/0/11
GigabitEthernet1/0/11 is up, line protocol is up (connected)
  Hardware is Gigabit Ethernet, address is 0000.5e00.530b (bia 0000.5e00.530b)
  MTU 1500 bytes, BW 100000 Kbit/sec, DLY 100 usec,
  Encapsulation ARPA, loopback not set
  Half-duplex, 100Mb/s, media type is 10/100/1000BaseTX
     48213 packets input, 6120344 bytes, 0 no buffer
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     52110 packets output, 71244810 bytes, 0 underruns
     0 output errors, 1842 collisions, 0 interface resets
     0 babbles, 317 late collision, 0 deferred
```

`Encapsulation ARPA` is Cisco's name for Ethernet II. The late collisions are the proof.

Linux, on the server:

```bash
sudo ethtool eth0
```

```text
Settings for eth0:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Speed: 100Mb/s
        Duplex: Full
        Auto-negotiation: off
        Link detected: yes
```

There it is. Someone forced the server to 100 full, so it sends no negotiation pulses. The switch detected 100 Mb/s and assumed half duplex.

Windows equivalent, if the server ran Windows:

```powershell
Get-NetAdapter | Format-Table Name, Status, LinkSpeed, FullDuplex, MtuSize
Get-NetAdapterAdvancedProperty -Name "Ethernet" -DisplayName "Speed & Duplex"
```

The display name and accepted values depend on the NIC driver.

The fix: set both ends to auto.

```bash
# Linux server
sudo ethtool -s eth0 autoneg on
```

```text
! Cisco IOS switch
Switch(config)# interface GigabitEthernet1/0/11
Switch(config-if)# speed auto
Switch(config-if)# duplex auto
Switch(config-if)# end
Switch# clear counters GigabitEthernet1/0/11
```

The port now shows `a-full a-1000`, and the error counters stay at zero. Clearing counters is what lets you prove the fix.

### Example 4: testing a jumbo path

Two storage hosts, `10.20.0.11` and `10.20.0.12`, are set to MTU 9000. Transfers hang. Test it.

The largest ping that fits in MTU 9000 is 8972 bytes of data (9000 - 20 IP - 8 ICMP). The DF (Don't Fragment) flag stops the packet being split.

Set and check the MTU:

```bash
# Linux
sudo ip link set dev eth1 mtu 9000
ip link show eth1
```

```powershell
# Windows: the keyword is standard, the value depends on the driver
Get-NetAdapterAdvancedProperty -Name "Ethernet" -RegistryKeyword "*JumboPacket"
Set-NetAdapterAdvancedProperty -Name "Ethernet" -RegistryKeyword "*JumboPacket" -RegistryValue 9014
```

Many Windows drivers include the 14-byte header in that value, so 9014 means an MTU of 9000.

Test:

```bash
# Linux
ping -M do -s 8972 -c 3 10.20.0.12
```

```powershell
# Windows
ping -f -l 8972 10.20.0.12
```

Output on Linux when a switch in the middle still uses 1500:

```text
PING 10.20.0.12 (10.20.0.12) 8972(9000) bytes of data.

--- 10.20.0.12 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2044ms
```

A normal ping works:

```text
$ ping -M do -s 1472 -c 1 10.20.0.12
PING 10.20.0.12 (10.20.0.12) 1472(1500) bytes of data.
1480 bytes from 10.20.0.12: icmp_seq=1 ttl=64 time=0.412 ms
```

So the path passes 1500 but not 9000. That is the signature of a switch that was missed. If you see `ping: local error: message too long`, the problem is the local interface, not the network.

Fix on the switch. The command varies by platform. On a Cisco Catalyst 9000:

```text
Switch(config)# system mtu 9198
```

Older Catalyst models use different commands, and some need a reload. Nexus switches use a different method again. Check the guide for your model.

## Going Deeper

### Ethernet II versus 802.3 LLC

Two frame formats exist.

```text
Ethernet II (DIX):
| Dest 6 | Src 6 | EtherType 2 | Payload        | FCS 4 |

IEEE 802.3 with LLC:
| Dest 6 | Src 6 | Length 2 | DSAP | SSAP | Ctrl | Data | FCS 4 |
                              1      1      1-2

IEEE 802.3 with LLC and SNAP (after Dest, Src and Length):
| AA | AA | 03 | OUI 3 | Type 2 | Data | FCS 4 |
```

Ethernet II is named DIX after DEC, Intel and Xerox, the companies behind it. IEEE 802.3 used a length field instead, and added LLC (Logical Link Control) to identify the protocol. SNAP (Subnetwork Access Protocol) is an LLC extension that adds a type field back.

Since the 802.3x revision in 1997, the IEEE standard has accepted the Type meaning as well.

Rule for a receiver looking at those 2 bytes:

- 1500 or less: it is a length, so read LLC next.
- 1536 or more: it is an EtherType.
- 1501-1535: not defined.

Where LLC still appears today: classic STP BPDUs use LLC with DSAP and SSAP of `0x42`. Cisco CDP uses SNAP. Almost all user traffic is Ethernet II.

### Ethernet II has no length field

The receiver finds the end of a frame from the physical signal, not from a header. So how does it remove padding? It does not. The padding goes up to the next layer, which uses its own length. For example, the IPv4 Total Length field says where the real data ends. See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

### Bit order on the wire

Ethernet sends each byte least significant bit first. That is why the preamble byte `10101010` as sent appears as `0x55` in many documents, and the SFD `10101011` appears as `0xD5`.

It also explains a MAC address detail: the lowest bit of the first byte is the first bit sent, and that bit marks group addresses. See [MAC Addresses](01-mac-addresses.md). The FCS is an exception, sent most significant bit first.

### What the FCS does and does not do

CRC-32 is a calculation that treats the frame as one huge number and divides it by a fixed value. The remainder is the FCS.

It catches all error bursts up to 32 bits long, and nearly all other damage. It only detects errors. It cannot fix them. It also gives no security: anyone changing a frame can compute a new FCS. Protection against that needs MACsec (802.1AE) or encryption at a higher layer.

The FCS is checked hop by hop. Each switch and router checks it, and rewrites it when the frame changes. It does not protect data end to end.

A **cut-through** switch starts forwarding before the FCS arrives, so it cannot drop a bad frame. It can pass it on, and the next device counts the error. Some switches "stomp" the CRC, meaning they deliberately write a bad FCS to mark it. This is why CRC errors can appear on a port that has a perfectly good cable. Look upstream.

### Tags and minimum size

The 64-byte minimum stays the same with a tag. So a tagged frame needs only 42 bytes of payload. When a switch removes a tag from a 64-byte frame, the frame would become 60 bytes, so the switch must pad it back to 64 and write a new FCS.

### Line rate math

Every frame on the wire costs 20 extra bytes: 8 for preamble and SFD, 12 for the gap.

```text
64-byte frame:   64 + 20 =   84 bytes =   672 bits
1518-byte frame: 1518 + 20 = 1538 bytes = 12304 bits
```

| Speed | Max frames per second, 64 bytes | Max frames per second, 1518 bytes |
|---|---|---|
| 1 Gb/s | 1,488,095 | 81,274 |
| 10 Gb/s | 14,880,952 | 812,744 |
| 100 Gb/s | 148,809,524 | 8,127,438 |

At 10 Gb/s, a device has only 67.2 ns per minimum-size frame. This is why router and switch datasheets quote packets per second, and why small-packet floods hurt more than bandwidth alone suggests.

### Line coding, lanes and FEC

Line coding turns bits into signals.

| Standard | Coding |
|---|---|
| 10BASE-T | Manchester |
| 100BASE-TX | 4B/5B with MLT-3 |
| 1000BASE-X (fibre) | 8B/10B |
| 1000BASE-T | PAM-5 on all 4 pairs, both directions at once |
| 10GBASE-R | 64B/66B |
| 50G and 100G electrical and optical lanes | PAM4, which sends 2 bits per symbol |

1000BASE-T needs all four pairs. If one pair is broken, a link may drop to 100 Mb/s, which uses only two. A "gigabit port stuck at 100" is usually a bad cable or connector, not a config problem.

High-speed lanes get many errors, so they use FEC (forward error correction). FEC adds extra data that lets the receiver repair errors without a resend. It is required at 400G. On 25G and 100G links, both ends must use the same FEC mode, or the link may not come up. This is a common fault when mixing vendors.

### Autonegotiation variants

- **Twisted pair:** negotiates speed and duplex. Required for 1000BASE-T.
- **1000BASE-X fibre:** negotiates duplex and flow control, not speed. Both ends must agree to use it.
- **10G fibre:** none. The optic sets the speed.
- **Backplane and direct attach copper:** a separate method that also covers FEC on higher speeds.

Auto-MDIX is a different feature. It detects whether a cable is straight-through or crossover, and swaps the transmit and receive pairs if needed. A port can have it on or off regardless of speed and duplex settings.

### Flow control

Full duplex removed collisions, but a fast sender can still overrun a slow receiver.

- **PAUSE (802.3x):** the receiver sends a special frame to `01:80:c2:00:00:01` with EtherType `0x8808`, asking the neighbour to stop for a set time. It stops all traffic on the link. Many networks leave it off.
- **PFC (Priority Flow Control, 802.1Qbb):** pauses only one traffic class. Lossless traffic such as RoCE storage networks uses it.

### Power over Ethernet

**PoE (Power over Ethernet)** sends power over the same cable. Phones, cameras and access points use it.

| Standard | Type | Power at switch | Power at device |
|---|---|---|---|
| 802.3af | 1 | 15.4 W | 12.95 W |
| 802.3at | 2 | 30 W | 25.5 W |
| 802.3bt | 3 | 60 W | 51 W |
| 802.3bt | 4 | 90 W | 71.3 W |

### Preamble on modern links

The preamble was vital when the line was silent between frames. Modern links send idle signals all the time, so the clock stays locked. The 8 bytes remain for compatibility. At 10 Gb/s and above, the first byte is replaced by a start control character, but the size stays the same.

## Common Mistakes

- **"The maximum frame is 1500 bytes."** No. 1500 is the MTU, the payload. The frame maximum is 1518, or 1522 with a tag.
- **"The minimum frame is 46 bytes."** 46 is the minimum payload. The frame minimum is 64.
- **Counting the preamble in frame size.** The 64 and 1518 numbers start at the destination MAC. The preamble, SFD and gap are extra.
- **"802.1Q reduces the payload by 4 bytes."** It does not. The frame grows by 4 bytes and the payload stays 1500. Trouble only arises on old gear that drops frames above 1518.
- **"A capture showing a 60-byte frame is a runt."** Capture tools do not show the FCS. 60 captured bytes is a valid 64-byte frame.
- **Believing the switch retries a dropped frame.** Ethernet never retransmits a frame damaged in transit. (Half-duplex backoff after a normal collision is a separate thing.)
- **"Collisions happen on switched networks."** Not on full-duplex links. Rising collisions on a switch port mean half duplex is in use, usually by mistake.
- **Forcing speed and duplex on only one end.** This is the classic cause of duplex mismatch. Set both ends, or better, neither.
- **Forcing 1 Gb/s with autonegotiation off on copper.** 1000BASE-T needs negotiation for master and slave selection. Depending on vendor, this is refused, ignored, or gives a dead link.
- **Enabling jumbo frames on hosts only.** Every switch port on the path must allow them.
- **Mixing up MTU numbers across vendors.** Linux and Cisco MTU figures usually exclude the Ethernet header. Some vendors, such as Juniper for interface MTU, include it. A 9000 here may not equal a 9000 there.
- **Treating full duplex as doubling a link's speed.** A 1 Gb/s link is 1 Gb/s in each direction. One TCP transfer still tops out near 941 Mb/s.
- **Trusting the green light.** Link up says the physical layer is working. It says nothing about duplex, errors or MTU.

## Interview Questions

**Q: What are the fields of an Ethernet frame?**

A: On the wire, there is a 7-byte preamble and 1-byte start frame delimiter. The frame itself has a 6-byte destination MAC, a 6-byte source MAC, a 2-byte EtherType, a payload of 46 to 1500 bytes, and a 4-byte frame check sequence, which is a CRC-32.

**Q: Why is the minimum frame size 64 bytes?**

A: It comes from CSMA/CD. A sender can only detect a collision while still sending, so the frame must last as long as the worst-case round trip on the network. That was set at 512 bit times, which is 64 bytes. Full duplex no longer needs it, but it stays for compatibility.

**Q: What is the maximum frame size, and how does a VLAN tag change it?**

A: 1518 bytes: 1500 of payload plus 14 of header and 4 of FCS. An 802.1Q tag adds 4 bytes between the source MAC and EtherType, so the maximum is 1522. The payload stays at 1500.

**Q: What is the difference between MTU and frame size?**

A: MTU is the biggest payload the link carries, normally 1500 bytes. It excludes the Ethernet header and FCS. Frame size includes them, so an MTU of 1500 gives a frame of 1518 bytes.

**Q: What does the EtherType field do? Give examples.**

A: It tells the receiver which protocol is in the payload. `0x0800` is IPv4, `0x0806` is ARP, `0x86DD` is IPv6, and `0x8100` marks an 802.1Q tag. A value of 1500 or less means the field is a length instead, which is the older 802.3 format.

**Q: How does the FCS work, and what happens when it fails?**

A: The sender runs a CRC-32 over the frame and puts the result in the FCS. The receiver repeats the calculation. If they differ, the frame is dropped and the error counter rises. Ethernet does not ask for a resend, so TCP or the application must recover.

**Q: What is CSMA/CD, and why is it not used today?**

A: It is the rule for sharing a half-duplex medium: listen first, send, watch for collisions, and back off randomly if one occurs. Switches gave each device its own full-duplex link with separate send and receive paths, so collisions cannot happen and CSMA/CD is turned off.

**Q: What are jumbo frames? When would you use them, and what is the risk?**

A: Jumbo frames carry more than 1500 bytes of payload, usually 9000. They reduce CPU work and overhead for storage, backups and VM migration, and leave room for tunnel headers. They are not an IEEE standard. Every device on the path must allow them, otherwise large frames are silently dropped.

**Q: How do you test whether a path really supports MTU 9000?**

A: I send a ping with Don't Fragment set and 8972 bytes of data, which is 9000 minus 20 for IP and 8 for ICMP. On Linux that is `ping -M do -s 8972`, on Windows `ping -f -l 8972`. If 1472 works and 8972 fails, a device on the path is still at 1500.

**Q: What is autonegotiation, and what happens if only one side uses it?**

A: Both ends exchange abilities using fast link pulses and pick the best common speed and duplex. If one side is forced, it sends no pulses. The auto side can detect speed from the signal but not duplex, so it falls back to half duplex. If the forced side is full duplex, that is a mismatch.

**Q: What are the symptoms of a duplex mismatch? How do you fix it?**

A: The link is up and pings work, but throughput is terrible under load. The half-duplex side shows collisions and late collisions, and the full-duplex side shows CRC errors and runts. I confirm with `show interfaces` and `ethtool`, set both ends to auto, then clear the counters and watch them stay at zero.

**Q: A gigabit iperf test shows 941 Mb/s. Is something wrong?**

A: No, that is normal. A 1500-byte packet takes 1538 bytes of line time, once you add the Ethernet header, FCS, preamble and gap. With TCP timestamps, only 1448 of those bytes are data, and 1448 / 1538 is about 94%, or 941 Mb/s.

## Quick Recap

- Ethernet II frame: destination 6, source 6, EtherType 2, payload 46-1500, FCS 4. Preamble and SFD (8 bytes) come first but are not counted.
- Frame size is 64-1518 bytes, or up to 1522 with an 802.1Q tag. MTU 1500 is the payload only.
- Each frame costs 20 more bytes of line time, which is why 1 Gb/s gives about 941 Mb/s of TCP data.
- The 64-byte minimum comes from CSMA/CD. Full-duplex switched links have no collisions, so CSMA/CD is off, but the minimum stays.
- Jumbo frames (usually MTU 9000) are not an IEEE standard, and every device on the Layer 2 path must allow them.
- 10G and above are full duplex only. Read names as speed, BASE, then medium and lanes.
- Leave autonegotiation on at both ends. A forced end against an auto end causes a duplex mismatch: late collisions on one side, CRC errors and runts on the other.
