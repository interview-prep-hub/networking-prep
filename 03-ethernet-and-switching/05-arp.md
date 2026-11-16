# ARP (Address Resolution Protocol)

**Level:** Intermediate | **Prerequisites:** [MAC Addresses](01-mac-addresses.md), [Ethernet Frames and Standards](02-ethernet-frames.md), [How Switches Work](03-how-switches-work.md), [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md), [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md)

## In One Line

ARP finds the MAC address that belongs to an IPv4 address on the local network, so a device can build an Ethernet frame and send it.

## Why It Matters

A device on a LAN has two addresses. An **IP address** is the logical address, set by software, and it says where a device sits in the network. A **MAC address** is the hardware address of the network card (the NIC), and it is what Ethernet uses to deliver a frame on one link.

Software usually knows only the IP address. You type `ping 192.168.1.20`, and nothing in that command gives a MAC address. But an Ethernet frame cannot leave the card without a destination MAC. Something must connect the two addresses. That something is ARP.

Without ARP, every IPv4 packet on Ethernet would stop at the first step. No web page, no DNS lookup, no ping.

Interviewers ask about ARP for three reasons:

- It tests whether you really understand the difference between Layer 2 and Layer 3.
- "What happens when you ping a host on another subnet?" is a classic question, and the answer turns on ARP for the gateway.
- Many real faults show up as ARP faults: duplicate IPs, wrong masks, stale entries, and failover that does not work.

## Simple Explanation

Imagine a big open office. You have a letter for Ali, but you do not know which desk is his. Everyone has a desk number.

1. You stand up and shout to the whole room: "Who is Ali? Tell Sara at desk 10."
2. Everyone hears it. Only Ali reacts. He walks to your desk and says: "I am Ali, desk 20."
3. You write "Ali = desk 20" in your notebook, in pencil.
4. Next time you do not shout. You check your notebook. After a while you rub out old lines, because people move desks.

Now map it:

| Office | Network |
|---|---|
| A person's name | IP address |
| Desk number | MAC address |
| The shout to the room | ARP request (a broadcast) |
| Ali's quiet answer | ARP reply (a unicast) |
| The notebook | ARP cache |
| The room | Broadcast domain (one VLAN or one LAN) |

A **broadcast** is a message sent to every device on the LAN. A **unicast** is a message sent to one device. A **cache** is a short-term memory that saves you from repeating work.

**Where the analogy stops being true:** in an office you can see who is talking, and you might doubt a strange answer. ARP has no such sense. It believes almost any answer, which is why ARP spoofing works. Also, a shout in an office can be heard through an open door, but an ARP broadcast never passes through a router.

A second picture helps for other subnets. If Ali is in a different building, you do not shout for him. Your voice will not reach him. You shout only to find the mail room in your own building, and you give the letter to them. The mail room is your **default gateway**, the router that takes traffic off your subnet.

## How It Works

### Step 0: Decide Who You Need to Reach

ARP is not the first thing that happens. Before ARP, the sender asks: is the destination on my own subnet? A **subnet** is a group of addresses that can reach each other directly at Layer 2. The sender uses its IP address and **subnet mask** to work this out. The mask says which part of the address is the network part.

Example: PC-A is `192.168.1.10/24`.

| Destination | Network of destination | Same as 192.168.1.0? | Next hop (who to ARP for) |
|---|---|---|---|
| 192.168.1.20 | 192.168.1.0 | Yes | 192.168.1.20 |
| 192.168.2.50 | 192.168.2.0 | No | 192.168.1.1 (gateway) |
| 8.8.8.8 | 8.0.0.0 | No | 192.168.1.1 (gateway) |

The **next hop** is the device that should receive this frame on this link. ARP is always for the next hop, never for a far-away host.

Full process:

```text
Packet to send
  |
  v
Destination in my subnet?
  +-- yes --> next hop = destination
  +-- no  --> next hop = default gateway
  |
  v
Next hop in ARP cache?
  +-- yes --> build frame with that MAC and send
  +-- no  --> send ARP request, hold or drop packet, wait for reply
```

The fuller truth: a modern host checks its whole routing table, not just "local or gateway". For a simple PC, the table has only those two choices. See [The Default Gateway](../04-ip-addressing/08-default-gateway.md) and [The Routing Table](../05-routing/02-routing-table.md).

### Request and Reply Flow

Here PC-A wants to reach PC-B on the same LAN. This topology is used through the whole chapter:

| Device | IP address | MAC address |
|---|---|---|
| PC-A | 192.168.1.10 | 02:00:00:00:00:0a |
| PC-B | 192.168.1.20 | 02:00:00:00:00:14 |
| Printer | 192.168.1.50 | 02:00:00:00:00:32 |
| R1 G0/0 | 192.168.1.1 | 02:00:00:00:00:01 |
| R1 G0/1 | 192.168.2.1 | 02:00:00:00:02:01 |
| Server | 192.168.2.50 | 02:00:00:00:02:32 |

(These MACs are made up. The `02` start marks a locally assigned address, so they do not belong to a real vendor.)

```text
    PC-A              SW1               PC-B              R1
   (.10)                               (.20)             (.1)
      |                 |                 |                 |
   1. | who has .20?    |                 |                 |
      |---------------->|---------------->|                 |
      |                 |---------------------------------->| ignores
      |                 |                 |                 |
   2. |                 | .20 is at ...14 |                 |
      |<----------------|<----------------|                 |
      |                 |                 |                 |
   3. | IP packet, now inside a frame to ...14              |
      |---------------->|---------------->|                 |
```

Step by step:

1. PC-A looks in its ARP cache for `192.168.1.20`. It is not there.
2. PC-A sends an **ARP request** in a frame to the broadcast MAC `ff:ff:ff:ff:ff:ff`. The message means: "Who has 192.168.1.20? Tell 192.168.1.10."
3. Switch SW1 learns PC-A's MAC from the frame's source. Because the destination is broadcast, it floods the frame out of every other port in that VLAN.
4. Every device receives the request and reads it. R1 sees that the target IP is not its own, so it does not reply.
5. PC-B sees its own IP as the target. It saves PC-A's IP and MAC in its own cache, because it will almost surely need to answer PC-A soon.
6. PC-B sends an **ARP reply** in a unicast frame straight to PC-A. The message means: "192.168.1.20 is at 02:00:00:00:00:14."
7. PC-A saves the entry in its cache and sends the waiting packet.

The cache rule from the original standard: a receiver updates an entry it already has for the sender, even if the request was not for it. It adds a new entry only if the request was aimed at it. So R1 in step 4 refreshes PC-A's entry if it already has one, but does not create a new one. Real systems vary a little, but this is the normal behaviour.

ARP is defined in RFC 826 (1982). It was designed to work for any pair of address types, which is why the packet has fields for address type and length.

### Packet Fields

ARP has no IP header, no TCP, no UDP, and no port number. It sits directly inside an Ethernet frame. The frame's **EtherType** field, which tells the receiver what is inside, is `0x0806` for ARP. (IPv4 is `0x0800`.)

```text
+-----------------+----------------+-------------+-----------+-----+
| Destination MAC | Source MAC     | EtherType   | ARP       | FCS |
| ff:ff:ff:ff:... | 02:00:...:0a   | 0x0806      | 28 bytes  |     |
+-----------------+----------------+-------------+-----------+-----+
```

The ARP packet for Ethernet and IPv4 is 28 bytes:

```text
 0               8               16                              31
+-------------------------------+-------------------------------+
|     Hardware Type (HTYPE)     |     Protocol Type (PTYPE)     |
+---------------+---------------+-------------------------------+
|  HLEN         |  PLEN         |       Operation (OPER)        |
+---------------+---------------+-------------------------------+
|           Sender Hardware Address, SHA (bytes 0-3)            |
+-------------------------------+-------------------------------+
|       SHA (bytes 4-5)         |  Sender Protocol Addr (0-1)   |
+-------------------------------+-------------------------------+
|       SPA (bytes 2-3)         |  Target Hardware Addr (0-1)   |
+-------------------------------+-------------------------------+
|           Target Hardware Address, THA (bytes 2-5)            |
+---------------------------------------------------------------+
|           Target Protocol Address, TPA (bytes 0-3)            |
+---------------------------------------------------------------+
```

"Hardware" here means the MAC layer. "Protocol" means the upper address type, which is IPv4.

| Field | Size | Value for Ethernet + IPv4 | Meaning |
|---|---|---|---|
| HTYPE | 2 bytes | 1 | Hardware type: Ethernet |
| PTYPE | 2 bytes | 0x0800 | Protocol type: IPv4 |
| HLEN | 1 byte | 6 | MAC length in bytes |
| PLEN | 1 byte | 4 | IPv4 length in bytes |
| OPER | 2 bytes | 1 or 2 | 1 = request, 2 = reply |
| SHA | 6 bytes | sender MAC | Who is sending |
| SPA | 4 bytes | sender IP | Sender's IP |
| THA | 6 bytes | target MAC | Zeros in a request, because it is unknown |
| TPA | 4 bytes | target IP | The IP being asked about |

Here are both packets from the flow above:

| Field | ARP request | ARP reply |
|---|---|---|
| Frame destination MAC | ff:ff:ff:ff:ff:ff | 02:00:00:00:00:0a |
| Frame source MAC | 02:00:00:00:00:0a | 02:00:00:00:00:14 |
| OPER | 1 | 2 |
| SHA | 02:00:00:00:00:0a | 02:00:00:00:00:14 |
| SPA | 192.168.1.10 | 192.168.1.20 |
| THA | 00:00:00:00:00:00 | 02:00:00:00:00:0a |
| TPA | 192.168.1.20 | 192.168.1.10 |

Note that the request has two "destination MAC" values, and they differ. The Ethernet header says broadcast, so that everyone gets the frame. The ARP field THA is all zeros, because that value is the unknown.

The request in raw hex, as seen on the wire:

```text
ff ff ff ff ff ff  02 00 00 00 00 0a  08 06    Ethernet header
00 01  08 00  06  04  00 01                    HTYPE PTYPE LENS OPER
02 00 00 00 00 0a  c0 a8 01 0a                 SHA, SPA (192.168.1.10)
00 00 00 00 00 00  c0 a8 01 14                 THA, TPA (192.168.1.20)
```

An Ethernet frame must carry at least 46 bytes of payload. ARP has only 28, so the sender adds 18 bytes of padding. This gives the minimum frame: 14 header + 46 payload + 4 FCS = 64 bytes. That is why a capture can show the same kind of ARP packet as `length 28` on the sending machine and `length 46` on the receiving one.

### The ARP Cache

The **ARP cache** (or ARP table) is a list of IP-to-MAC pairs stored in memory. The device checks it before it sends any ARP request. This keeps broadcasts low.

Entries are of two kinds:

- **Dynamic:** learned from ARP, and removed or rechecked after a time.
- **Static:** typed by an administrator. They do not age out.

**View the cache**

Linux:

```bash
ip neigh show
# 192.168.1.1 dev eth0 lladdr 02:00:00:00:00:01 REACHABLE
# 192.168.1.20 dev eth0 lladdr 02:00:00:00:00:14 STALE

arp -n          # older net-tools command, may not be installed
```

Windows (Command Prompt or PowerShell):

```powershell
arp -a
```

```text
Interface: 192.168.1.10 --- 0xb
  Internet Address      Physical Address      Type
  192.168.1.1           02-00-00-00-00-01     dynamic
  192.168.1.20          02-00-00-00-00-14     dynamic
  192.168.1.255         ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  255.255.255.255       ff-ff-ff-ff-ff-ff     static
```

The static lines for broadcast and multicast were never learned from ARP. Those MACs are fixed or calculated from the IP, so no question is needed.

Windows PowerShell has a richer view:

```powershell
Get-NetNeighbor -AddressFamily IPv4 -InterfaceIndex 11
```

```text
ifIndex IPAddress       LinkLayerAddress   State      PolicyStore
------- ---------       ----------------   -----      -----------
11      192.168.1.20    02-00-00-00-00-14  Stale      ActiveStore
11      192.168.1.1     02-00-00-00-00-01  Reachable  ActiveStore
```

Cisco IOS:

```text
R1# show ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.1.1             -   0200.0000.0001  ARPA   GigabitEthernet0/0
Internet  192.168.1.10           12   0200.0000.000a  ARPA   GigabitEthernet0/0
Internet  192.168.1.99            0   Incomplete      ARPA
```

Read it like this: `Age` is minutes since the entry was learned. A dash means the address belongs to this router. `Incomplete` means a request went out and no reply came back. `ARPA` is the name IOS uses for normal Ethernet encapsulation.

**Clear and add entries**

| Task | Linux | Windows (run as admin) | Cisco IOS |
|---|---|---|---|
| Delete one | `ip neigh del 192.168.1.20 dev eth0` | `arp -d 192.168.1.20` | `clear ip arp 192.168.1.20` |
| Clear all | `ip neigh flush dev eth0` | `arp -d *` or `netsh interface ip delete arpcache` | `clear arp-cache` |
| Add static | `ip neigh replace 192.168.1.50 lladdr 02:00:00:00:00:32 dev eth0 nud permanent` | `netsh interface ipv4 add neighbors "Ethernet" 192.168.1.50 02-00-00-00-00-32` | `arp 192.168.1.50 0200.0000.0032 ARPA` (global config) |

Do not use a static entry casually. If the NIC is replaced, the entry stays wrong until someone removes it.

### ARP Timeouts

Why not keep entries forever? Because the mapping can change. A PC gets a new NIC, a laptop leaves, or an IP moves to a new server. A stale entry sends traffic to the wrong place.

RFC 826 does not set a timer value. So timers are vendor defaults:

| System | Default | Notes |
|---|---|---|
| Cisco IOS | 14400 seconds (4 hours) | Per interface: `arp timeout <seconds>` |
| Cisco NX-OS | 1500 seconds (25 minutes) | Set with `ip arp timeout` on the interface |
| Juniper Junos | 20 minutes | Configurable |
| Linux | About 15-45 seconds as REACHABLE | Then STALE, not deleted (see Going Deeper) |
| Windows (Vista and later) | About 15-45 seconds as Reachable | Same model as Linux |

These values can change with version and platform, so check your own device. On IOS:

```text
R1# show interfaces GigabitEthernet0/0 | include ARP type
  ARP type: ARPA, ARP Timeout 04:00:00
R1# configure terminal
R1(config)# interface GigabitEthernet0/0
R1(config-if)# arp timeout 1200
```

Short timeout: the table stays fresh, but more ARP traffic. Long timeout: less traffic, but wrong entries last longer.

### ARP Across Subnets (ARP for the Gateway)

This is the point many beginners miss.

ARP requests are broadcasts, and routers do not forward broadcasts. So ARP works only inside one broadcast domain. A PC can never ARP directly for a host on another subnet.

Instead, the PC ARPs for its **default gateway** and sends the frame there. The frame's destination MAC is the router. The packet's destination IP is still the final host.

```text
 192.168.1.0/24                             192.168.2.0/24

 PC-A .10 ---+                                 
             +--- SW1 --- [G0/0  R1  G0/1] --- SW2 --- Server .50
 PC-B .20 ---+            .1          .1
```

MAC addresses change at each hop. IP addresses stay the same (when no NAT is involved).

| Hop | Source MAC | Destination MAC | Source IP | Destination IP |
|---|---|---|---|---|
| PC-A to R1 | 02:00:00:00:00:0a | 02:00:00:00:00:01 | 192.168.1.10 | 192.168.2.50 |
| R1 to Server | 02:00:00:00:02:01 | 02:00:00:00:02:32 | 192.168.1.10 | 192.168.2.50 |

Every router on a path does the same thing on the next link. It finds the next hop in its routing table, then ARPs for that next hop if the link is Ethernet. On a path across the internet, ARP may run on many links, and each one is separate. See [How Routers Work](../05-routing/01-how-routers-work.md).

Links that are not shared, such as a PPP serial link, do not need ARP. There is only one other end, so no address has to be looked up.

### Gratuitous ARP

**Gratuitous** means "not asked for". A **gratuitous ARP (GARP)** is an ARP message a device sends about itself when no one asked. It uses its own IP as both sender IP and target IP.

Analogy: a person who moved desks stands up and says, "Ali now sits at desk 20." Nobody asked, but everyone with Ali in their notebook updates it. The analogy stops here: in an office, people know Ali's voice. ARP cannot check who is speaking, so a liar can send the same message.

Why devices send it:

1. **Detect duplicate IPs.** If anyone answers, or sends a GARP for the same IP from another MAC, there is a conflict.
2. **Update other devices' caches.** After a failover or a MAC change, neighbours learn the new MAC right away and do not wait for a timer.
3. **Update switch MAC tables.** The GARP frame has the device's MAC as source. So switches learn the port it is now on.

Real uses:

- **First hop redundancy:** when a router takes over a virtual gateway IP, it sends GARP so that switches learn the new port for the virtual MAC. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).
- **VM live migration:** a virtual machine moves to a new host, and a GARP or similar frame tells switches where it went.
- **Server NIC teaming and floating IPs:** keepalived and similar tools send GARP when a virtual IP moves.
- **Bare-metal Kubernetes load balancers:** MetalLB in Layer 2 mode uses GARP when a service IP moves to another node.

RFC 5227 (IPv4 Address Conflict Detection) gives clear names for the related messages:

| Message | OPER | Sender IP (SPA) | Target IP (TPA) | Purpose |
|---|---|---|---|---|
| ARP Probe | 1 | 0.0.0.0 | Candidate IP | "Is anyone using this?" |
| ARP Announcement | 1 | Own IP | Own IP | "I am now using this." |
| GARP reply form | 2 | Own IP | Own IP | Same idea, used by some systems |

A probe uses `0.0.0.0` as sender on purpose. The device does not own the address yet, so it must not make others cache it. Strictly, a probe is not a gratuitous ARP, but some tools and people call it that.

RFC 5227 sends three probes spaced 1-2 seconds apart, then two announcements 2 seconds apart. Implementations do not all follow this. Link-local addressing (`169.254.0.0/16`, RFC 3927) uses the same idea. DHCP clients are also told to check an offered address before using it (see [DHCP](../07-application-protocols/04-dhcp.md)).

Send one by hand on Linux (iputils `arping`):

```bash
sudo arping -U -c 3 -I eth0 192.168.1.20   # announce with ARP requests
sudo arping -A -c 3 -I eth0 192.168.1.20   # announce with ARP replies
sudo arping -D -c 3 -I eth0 192.168.1.20   # check for duplicate address
```

### Proxy ARP

**Proxy ARP** means a router answers an ARP request on behalf of a host on another network. The router gives its own MAC, so the sender thinks the far host is local.

Analogy: a receptionist. You ask, "Where is Ali?" The receptionist says, "Give it to me." You think you reached Ali's desk, but the receptionist passes it on. The analogy stops being true at this point: the receptionist would tell you Ali is elsewhere, but proxy ARP gives no such hint. The sender has no idea a router was involved.

Example: PC-A is configured by mistake with mask `/16`, so it thinks `192.168.2.50` is local and needs no gateway.

```text
 PC-A (192.168.1.10/16)        R1 G0/0         R1 G0/1      Server
   |                             |                |            |
   | who has 192.168.2.50?       |                |            |
   |---------------------------->|                |            |
   |                             | route via G0/1 exists       |
   | 192.168.2.50 is at ...00:01 |                |            |
   |<----------------------------|                |            |
   | IP packet to R1's MAC       |                |            |
   |---------------------------->|--------------->|----------->|
```

R1 replies because it has a route to the target, and that route leaves through a different interface. This is how Cisco IOS behaves.

PC-A's cache now has a clue: several IPs show the same MAC.

```text
192.168.1.1   lladdr 02:00:00:00:00:01
192.168.2.50  lladdr 02:00:00:00:00:01
192.168.2.77  lladdr 02:00:00:00:00:01
```

Standard and defaults:

- Proxy ARP for subnets is described in RFC 1027 (1987).
- **Cisco IOS:** enabled by default on interfaces.
- **Linux:** disabled by default (`net.ipv4.conf.<if>.proxy_arp = 0`).

Check and disable it:

```text
R1# show ip interface GigabitEthernet0/0 | include Proxy
  Proxy ARP is enabled
  Local Proxy ARP is disabled
R1# configure terminal
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no ip proxy-arp
```

```bash
# Linux
sysctl net.ipv4.conf.eth0.proxy_arp
sudo sysctl -w net.ipv4.conf.eth0.proxy_arp=1
```

Trade-offs:

- **Good:** it can hide a wrong mask or support an old host with no gateway. It also has real uses: NAT devices and firewalls often answer ARP for public IPs in their outside subnet, and some container networks (Calico, for example) use it on purpose.
- **Bad:** it hides configuration errors. It makes one broadcast domain act larger. It grows ARP tables. It can let a rogue device receive traffic for a subnet. Many hardened networks turn it off on user-facing interfaces.

A related trap is on routers. A static route that points only to an exit interface, like `ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/0`, makes the router treat every destination as directly connected. It ARPs for each one and relies on the next router's proxy ARP. Use a next-hop IP instead.

### RARP: History Only

**RARP (Reverse ARP)**, RFC 903 from 1984, does the opposite job: "My MAC is this. What is my IP?" It helped diskless workstations that had no place to store an IP address.

Facts worth knowing:

- It uses the same packet format, with OPER 3 (request) and 4 (reply).
- It uses its own EtherType, `0x8035`.
- It needed a RARP server on each LAN, because broadcasts do not cross routers.
- It delivered only an IP address, with no mask, gateway or DNS server.

BOOTP (RFC 951) replaced it, and then DHCP (RFC 2131) replaced BOOTP. DHCP runs on UDP, can cross routers through a relay agent, and delivers full settings. You will not deploy RARP today, but interviewers like the comparison.

Another relative is **Inverse ARP** (InARP, RFC 2390, OPER 8 and 9). It finds an IP from a known link address. It was used on Frame Relay, which is now legacy.

### ARP Spoofing (Pointer)

ARP has no authentication. Many systems accept a reply they never asked for, at least to update an entry they already have. An attacker on the LAN can send: "192.168.1.1 is at my MAC." Victims update their caches and send gateway traffic to the attacker, who can read it, change it, or drop it. This is **ARP spoofing**, also called ARP poisoning.

The attacker must be on the same broadcast domain. The usual defense is **Dynamic ARP Inspection** on switches, which checks ARP against the DHCP snooping table. Full details are in [Layer 2 Security](11-layer2-security.md) and [Common Network Attacks](../09-security/03-common-network-attacks.md).

## Example

### Example 1: Watch ARP on the Same Subnet

PC-A (Linux) pings PC-B. A second terminal captures ARP.

```bash
# Terminal 1 on PC-A: capture only ARP, no name lookups, no timestamps
sudo tcpdump -nn -t -i eth0 arp

# Terminal 2 on PC-A
sudo ip neigh flush dev eth0
ping -c 1 192.168.1.20
ip neigh show dev eth0
```

Capture output:

```text
ARP, Request who-has 192.168.1.20 tell 192.168.1.10, length 28
ARP, Reply 192.168.1.20 is-at 02:00:00:00:00:14, length 46
```

Add `-e` to tcpdump to see the frame MACs. The request's length is 28 because tcpdump saw it before padding was added. The reply's is 46 because it arrived with padding.

Ping and cache output:

```text
PING 192.168.1.20 (192.168.1.20) 56(84) bytes of data.
64 bytes from 192.168.1.20: icmp_seq=1 ttl=64 time=1.12 ms

192.168.1.20 lladdr 02:00:00:00:00:14 REACHABLE
```

Same check on Windows, in an admin PowerShell:

```powershell
arp -d *
ping -n 1 192.168.1.20
arp -a
```

In Wireshark, the display filter `arp` shows all ARP, `arp.opcode == 1` shows requests, and `arp.isgratuitous` shows gratuitous ARP.

### Example 2: Ping Across Subnets From Empty Caches

PC-A pings the Server at `192.168.2.50`. All caches are empty.

1. PC-A compares the addresses. `192.168.2.50` is not in `192.168.1.0/24`. So the next hop is the gateway, `192.168.1.1`.
2. PC-A broadcasts: "Who has 192.168.1.1? Tell 192.168.1.10."
3. R1 replies from G0/0: "192.168.1.1 is at 02:00:00:00:00:01." R1 also caches PC-A.
4. PC-A sends the ICMP packet in a frame to R1's MAC. Destination IP stays `192.168.2.50`.
5. R1 removes the frame, checks the IP, and lowers the TTL by 1. Its routing table says `192.168.2.0/24` is connected on G0/1.
6. R1 needs the Server's MAC. It broadcasts on G0/1: "Who has 192.168.2.50? Tell 192.168.2.1."
7. The Server replies and also caches R1's G0/1 address. R1 can now send frames to the Server. A Cisco router drops the packet that started this ARP (see Example 3), so PC-A's very first echo may time out.
8. The Server sends its echo reply. `192.168.1.10` is remote for it, so the next hop is `192.168.2.1`. It already has that MAC, so no ARP is needed.

Look at PC-A's cache afterwards:

```bash
ip neigh show dev eth0
# 192.168.1.1 lladdr 02:00:00:00:00:01 REACHABLE
```

There is no entry for `192.168.2.50`. PC-A never learned the Server's MAC, and never will. That is the correct result.

R1 shows both sides:

```text
R1# show ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.1.1             -   0200.0000.0001  ARPA   GigabitEthernet0/0
Internet  192.168.1.10            0   0200.0000.000a  ARPA   GigabitEthernet0/0
Internet  192.168.2.1             -   0200.0000.0201  ARPA   GigabitEthernet0/1
Internet  192.168.2.50            0   0200.0000.0232  ARPA   GigabitEthernet0/1
```

### Example 3: The Dropped First Ping

Clear R1's ARP cache, then ping from the router itself:

```text
R1# clear arp-cache
R1# ping 192.168.2.50
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.50, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
```

The `.` is a timeout. IOS does not hold the first packet while ARP completes, so it drops it. By the second packet, ARP is done. This is normal and is not a network fault.

Other systems differ. RFC 1122 says a host should save at least the latest packet for each unresolved address. Linux keeps a small queue per entry, so a normal first ping from a Linux host usually succeeds.

### Example 4: Reading a Failure

PC-A pings a host that is switched off, on its own subnet.

Linux:

```text
$ ping -c 1 192.168.1.30
PING 192.168.1.30 (192.168.1.30) 56(84) bytes of data.
From 192.168.1.10 icmp_seq=1 Destination Host Unreachable
```

Windows:

```text
Reply from 192.168.1.10: Destination host unreachable.
```

The message comes from `192.168.1.10`, which is PC-A's own IP. No router said this. PC-A's own ARP failed, so it reports the error to itself. Windows even counts it as a "reply", so its summary can say 0% loss.

If the message came from a router's IP, the fault would be further away. This one small clue saves a lot of time.

Linux shows the failed entry in the cache:

```bash
ip neigh show 192.168.1.30
# 192.168.1.30 dev eth0 FAILED
```

## Going Deeper

### Linux Neighbor States

Linux does not simply "time out" entries. It runs a small state machine. Windows uses the same model, which comes from IPv6 Neighbor Discovery.

```text
          send request
 (none) ----------------> INCOMPLETE --no reply--> FAILED
                              |
                              | reply received
                              v
                          REACHABLE <----------+
                              |                |
                              | timer ends     | reply to probe
                              v                |
                            STALE            PROBE --no reply--> FAILED
                              |                ^
                              | packet sent    | no upper-layer
                              v                | confirmation
                            DELAY -------------+
```

| State | Meaning |
|---|---|
| INCOMPLETE | Request sent, no answer yet |
| REACHABLE | Recently confirmed. Use without checking |
| STALE | Not confirmed for a while. Still usable |
| DELAY | A packet used a stale entry. Wait for proof from upper layers |
| PROBE | Sending unicast ARP requests to confirm |
| FAILED | No answer |
| PERMANENT | Static entry |
| NOARP | No ARP needed for this entry |

Key point: STALE is not an error. The entry still works. Linux just rechecks it on the next use. Unicast checks also avoid disturbing the whole LAN with broadcasts. If a TCP connection is making progress, the kernel treats that as proof of reachability and skips the probe.

Upstream Linux kernel defaults (distributions may change them):

| Setting under `net.ipv4.neigh.<if>.` | Default | Meaning |
|---|---|---|
| `base_reachable_time_ms` | 30000 | Base timer, actual value is random 0.5x to 1.5x |
| `delay_first_probe_time` | 5 | Seconds in DELAY before PROBE |
| `gc_stale_time` | 60 | Seconds before an unused entry can be cleaned up |
| `retrans_time_ms` | 1000 | Gap between retries |
| `mcast_solicit` | 3 | Broadcast tries before FAILED |
| `ucast_solicit` | 3 | Unicast tries in PROBE |

### When ARP and MAC Timers Disagree

A switch MAC table and a router ARP table are separate, and their timers differ. Cisco switches age MAC entries after 300 seconds by default. Cisco IOS routers keep ARP for 14400 seconds.

This can cause **unicast flooding** with asymmetric routing, where traffic goes out through one router and returns through another. Say a host sends its traffic through Router A, but the replies come back through Router B. Router B has a valid ARP entry, so it keeps sending frames to the host's MAC. But the switch next to Router B no longer sees frames from that host, so it ages out the MAC. It now floods every frame for that host out of all ports in the VLAN.

The usual fix is to make the ARP timeout shorter than the MAC aging time, or the MAC aging time longer. Either way, ARP refreshes the switch's memory before it forgets.

### ARP at Scale

ARP costs CPU and memory on routers.

- **Table limits.** Linux has `gc_thresh1`, `gc_thresh2` and `gc_thresh3`, with upstream defaults 128, 512 and 1024 entries. A busy node with many neighbours can log `neighbour: arp_cache: neighbor table overflow!` and lose connections. Kubernetes nodes in large flat networks hit this often. The fix is to raise these limits.
- **Scanning load.** If someone scans a large connected subnet, the router must ARP for many empty addresses. On Cisco CEF platforms, such traffic hits a "glean" entry and goes to the CPU. Control-plane rate limits protect it.
- **Broadcast load.** All hosts in a VLAN must process each ARP broadcast. This is one reason large Layer 2 domains are avoided. See [VLANs](06-vlans.md).
- **ARP suppression.** In VXLAN EVPN fabrics, the switch can answer ARP locally from what it already knows, so fewer broadcasts cross the fabric. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).
- **Cloud networks.** An AWS VPC has no real broadcast. The virtual network itself answers ARP. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

### ARP Flux on Linux

By default, Linux treats an IP as belonging to the host, not to one NIC. So with two NICs on one subnet, both may answer ARP for the same IP. The result is that the answering MAC can change, which is called **ARP flux**.

Two settings control this, and they are also needed for load balancers using direct server return:

```bash
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1    # reply only if the IP is on the receiving NIC
sudo sysctl -w net.ipv4.conf.all.arp_announce=2  # use the best local source IP in requests
```

Also note `arp_accept`. Its default of 0 means Linux does not create new entries from gratuitous ARP, but it does update existing ones.

### Troubleshooting a Duplicate IP

Symptoms: a host works, then stops, then works. Or the MAC for one IP keeps changing. Use broadcast ARP, so each owner of the IP answers:

```bash
# Linux, iputils arping. -b keeps sending broadcast requests.
sudo arping -b -c 2 -I eth0 192.168.1.20
```

```text
ARPING 192.168.1.20 from 192.168.1.10 eth0
Unicast reply from 192.168.1.20 [02:00:00:00:00:14]  0.812ms
Unicast reply from 192.168.1.20 [02:00:00:00:00:77]  1.041ms
Unicast reply from 192.168.1.20 [02:00:00:00:00:14]  0.790ms
Unicast reply from 192.168.1.20 [02:00:00:00:00:77]  1.022ms
Sent 2 probes (2 broadcast(s))
Received 4 response(s)
```

Two MACs for one IP is the proof. Next, find the port on the switch with `show mac address-table address 0200.0000.0077`. Windows also reports IP conflicts in the System event log.

### Which Layer Is ARP?

There is no perfect answer. ARP is carried directly in Ethernet frames, so it needs no IP. It exists to serve IP. A good interview answer: "It works between Layer 2 and Layer 3. It is carried at Layer 2 and serves Layer 3. Many people call it Layer 2 or Layer 2.5."

### IPv6 Has No ARP

IPv6 uses **Neighbor Discovery (NDP)**, RFC 4861. It runs inside ICMPv6, which means it does use IP. It sends Neighbor Solicitation to a multicast address, not a broadcast. It replies with Neighbor Advertisement. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

## Common Mistakes

- **"A PC ARPs for the final destination."** Only if it is on the same subnet. Otherwise it ARPs for the default gateway.
- **"ARP crosses routers."** It does not. It stays in one broadcast domain. A new ARP may happen on the next link, but it is a separate exchange.
- **"The ARP reply is a broadcast."** A normal reply is unicast. The request is the broadcast.
- **"ARP uses IP" or "ARP uses a port."** ARP has no IP header and no port. It sits directly in an Ethernet frame with EtherType `0x0806`.
- **"The ARP table and MAC table are the same."** An ARP table maps IP to MAC, on hosts and routers. A MAC table maps MAC to port, on switches.
- **"Layer 2 switches do ARP to forward frames."** They do not. They only flood the broadcast and forward the reply. A switch does ARP only for its own management IP.
- **"STALE means broken."** In Linux and Windows, STALE means "use it, but check it soon".
- **"Dropped first ping means a bad link."** On a Cisco router, a `.!!!!` result usually means ARP was resolving.
- **"Gratuitous ARP is an attack."** It is a normal tool. Attackers can misuse the same format, but failover depends on it.
- **"Proxy ARP means the PC has a gateway."** It is the opposite. Proxy ARP lets a host reach other networks without using a gateway, and it can hide a wrong mask.
- **"Static ARP is free security."** It can stop poisoning of that one entry on that one host. But it does not scale, and it breaks when hardware changes.
- **"IPv6 uses ARP."** IPv6 uses NDP over ICMPv6.

## Interview Questions

**Q: What is ARP and why do we need it?**

A: ARP maps an IPv4 address to a MAC address on the local network. IP tells us the logical destination, but Ethernet needs a MAC to deliver a frame. ARP finds that MAC with a broadcast question and a unicast answer, then stores it in a cache.

**Q: Walk me through an ARP request and reply.**

A: The sender checks its cache. If there is no entry, it broadcasts a frame to `ff:ff:ff:ff:ff:ff` asking who has the target IP. The switch floods it in the VLAN. Everyone reads it, but only the owner of that IP answers, with a unicast reply containing its MAC. The sender caches it and sends the data. The target also caches the sender.

**Q: PC-A wants to reach a host on a different subnet. Whose MAC does it ARP for?**

A: Its default gateway. It compares the destination with its own subnet using the mask, sees that it is remote, and ARPs for the gateway IP. The frame goes to the router's MAC, while the destination IP stays as the final host.

**Q: What are the main fields in an ARP packet?**

A: Hardware type, protocol type, hardware length, protocol length, operation, then sender MAC, sender IP, target MAC and target IP. For Ethernet and IPv4 it is 28 bytes, and operation is 1 for request and 2 for reply. In a request, the target MAC field is zero.

**Q: What is gratuitous ARP and where is it used?**

A: It is an unsolicited ARP message where a device announces its own IP and MAC. It is used to detect duplicate IPs, and to update neighbours and switches after a change. Common cases are first hop redundancy failover, VM migration, and floating IPs.

**Q: What is proxy ARP? Would you leave it on?**

A: A router answers ARP for a host on another network with its own MAC, then forwards the traffic. It can make hosts work with a wrong mask or no gateway. I would turn it off where it is not needed, because it hides mistakes and increases ARP load, unless a design like NAT depends on it.

**Q: How long do ARP entries last?**

A: It depends on the system, since the ARP standard does not fix a timer. Cisco IOS defaults to 4 hours. Linux and Windows treat an entry as reachable for about 15 to 45 seconds, then mark it stale and recheck it on next use.

**Q: Your ping says "Destination host unreachable" and it comes from your own IP. What does it mean?**

A: My own machine could not reach the next hop, most likely because ARP failed. No router sent this. I would check the ARP cache for an incomplete or failed entry, then check the mask, the VLAN, the cable, and whether the target is powered on.

**Q: Why does the first ping from a Cisco router often fail?**

A: IOS drops the packet that triggers ARP. The first echo times out, ARP completes, and the other four succeed. So you see `.!!!!` and 80 percent success. It is expected behaviour.

**Q: What is RARP and why is it gone?**

A: Reverse ARP let a diskless device ask for an IP using its MAC. It only provided an IP address, and it needed a server on every LAN. BOOTP and then DHCP replaced it, because they give full settings and can cross routers through a relay.

**Q: Is ARP secure? How do you protect it?**

A: No. It has no authentication, so an attacker on the LAN can send false replies and redirect traffic. This is ARP spoofing. Dynamic ARP Inspection on switches is the main defense, using the DHCP snooping table to check ARP messages.

**Q: How does ARP differ from NDP in IPv6?**

A: ARP uses broadcast and runs directly over Ethernet. NDP runs over ICMPv6 and uses solicited-node multicast, so far fewer hosts need to process each query. NDP also does router discovery, which ARP does not.

## Quick Recap

- ARP maps an IPv4 address to a MAC address on the local link. It uses a broadcast request and a unicast reply, carried in Ethernet frames with EtherType `0x0806`.
- A host ARPs for the next hop: the destination if local, or the default gateway if remote. ARP never crosses a router.
- The packet is 28 bytes for Ethernet and IPv4. OPER 1 is request, 2 is reply, and RARP uses 3 and 4.
- The cache saves broadcasts. Timers are vendor defaults: 4 hours on Cisco IOS, and a reachable/stale model on Linux and Windows.
- Gratuitous ARP is an unasked announcement used for conflict detection and failover. Proxy ARP is a router answering for another network, and it is on by default in Cisco IOS.
- ARP has no authentication, so it can be spoofed. Dynamic ARP Inspection is the switch-side defense.
- IPv6 replaces ARP with NDP over ICMPv6.
