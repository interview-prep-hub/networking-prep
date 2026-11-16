# Inter-VLAN Routing

**Level:** Intermediate | **Prerequisites:** [VLANs](06-vlans.md), [Trunking and 802.1Q](07-trunking-8021q.md), [ARP](05-arp.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md)

## In One Line

Inter-VLAN routing moves IP packets from one VLAN to another using a Layer 3 device, because a switch working at Layer 2 keeps VLANs fully apart. The three classic designs are a router with one port per VLAN, router-on-a-stick, and a Layer 3 switch with SVIs.

## Why It Matters

A VLAN (Virtual LAN) is a group of switch ports that act like their own separate network. Companies use VLANs to keep broadcast traffic small and to keep departments apart.

But the separation is total. If Sales is in VLAN 10 and the file server is in VLAN 30, Sales cannot reach the server. A printer in another VLAN is also out of reach. So is the internet, unless something connects the VLANs.

Inter-VLAN routing is that connection. It gives you a controlled door between VLANs, and you can put rules on that door. Without routing, VLANs are islands.

Interviewers love this topic because one question tests many skills at once: VLAN tags, trunks, ARP, the default gateway, MAC rewriting, and TTL. It is also a very common real fault. "Same switch, but the two PCs cannot ping" is a daily support ticket.

## Simple Explanation

Think of a company building with several departments. Each department has its own internal mail. A person in Sales can hand a note to anyone else in Sales directly. That is a VLAN.

A Sales person cannot walk into Engineering. The doors are locked. To send a letter there, they drop it at the mail room. The mail room is the only place connected to every department. That is the router, and the mail room's address is the default gateway.

The mail room can be built in three ways:

- **One port per VLAN:** the mail room has a separate door for each department. Simple, but each door costs money.
- **Router-on-a-stick:** the mail room has one door. Everyone uses it, and each letter carries a coloured label that shows its department. Cheap, but the door gets crowded.
- **Layer 3 switch:** the mail sorting machine is built into the main hallway. Letters are sorted as they pass, with no trip to another room. Fast.

**Where the analogy stops being true:** VLANs are not physical walls. Many VLANs can share the same cable, and the only thing that separates them is a small tag and the switch's rules. Also, a real mail room might read every letter. A router only reads the address section and normally ignores the content.

## How It Works

### Why VLANs Need Routing

Four rules create the problem.

1. **A VLAN is one broadcast domain.** A broadcast is a frame sent to every device on the local network. A broadcast sent in VLAN 10 never reaches VLAN 20. See [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md).
2. **A switch forwards frames only inside their VLAN.** Its MAC address table is kept per VLAN. There is no Layer 2 path from VLAN 10 to VLAN 20.
3. **One VLAN normally maps to one IP subnet.** A subnet is a block of IP addresses, such as 192.168.10.0/24 (see [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md)).
4. **A host sends traffic for other subnets to its default gateway.** It uses its subnet mask to decide if the destination is local. If not, it gives the frame to the gateway.

```text
+---------------------- SW1 ----------------------+
|                                                 |
|   VLAN 10                 !   VLAN 20           |
|   PC A                    !   PC B              |
|   192.168.10.11/24        !   192.168.20.22/24  |
|                           !                     |
|          No frame ever crosses this line        |
+-------------------------------------------------+
```

Both PCs are on the same physical switch, and still they cannot talk.

Every inter-VLAN design needs the same four parts:

1. A Layer 3 interface with an IP address inside each VLAN. This address is the gateway.
2. IP routing turned on in the device that owns those interfaces.
3. Hosts configured with the correct gateway.
4. A Layer 2 path from each host to its gateway interface.

**Fuller truth:** one VLAN to one subnet is a design rule, not a law of physics. A VLAN can carry several subnets, for example with secondary addresses on the gateway. This is messy and is mostly used during migrations.

### Method 1: Router With One Port Per VLAN

Each VLAN gets its own physical cable to the router. The switch ports are ordinary access ports. An access port belongs to one VLAN and sends frames with no tag.

```text
              +-----------------------+
              |          R1           |
              | Gi0/0          Gi0/1  |
              | 192.168.10.1   .20.1  |
              +----+---------------+--+
                   |               |
        Gi1/0/23   |               |  Gi1/0/24
        (VLAN 10)  |               |  (VLAN 20)
              +----+---------------+--+
              |          SW1          |
              +----+---------------+--+
          Gi1/0/1  |               |  Gi1/0/2
          (VLAN 10)|               |  (VLAN 20)
                 PC A            PC B
```

Config on Cisco IOS:

```text
! R1 (Cisco IOS)
interface GigabitEthernet0/0
 description Gateway for VLAN 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 description Gateway for VLAN 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

! SW1 (Cisco IOS)
interface GigabitEthernet1/0/23
 description To R1 Gi0/0
 switchport mode access
 switchport access vlan 10
!
interface GigabitEthernet1/0/24
 description To R1 Gi0/1
 switchport mode access
 switchport access vlan 20
```

The router has no idea VLANs exist. It sees two normal networks on two normal ports. Only the switch knows the VLAN numbers.

This is easy to understand, and each VLAN gets a full link. The problem is scale. Twenty VLANs would need twenty router ports and twenty switch ports. Router ports are expensive.

### Method 2: Router-on-a-Stick

Router-on-a-stick uses one physical link that carries all VLANs. The name comes from the drawing: the router sits on top of one line, like a ball on a stick.

Two features make this work:

- **Trunk:** a link that carries many VLANs. Each frame gets an 802.1Q tag that holds its VLAN number. See [Trunking and 802.1Q](07-trunking-8021q.md).
- **Subinterface:** a logical interface made in software on top of one physical port. Each subinterface acts like a separate port with its own IP address.

```text
+--------+--------+------------------+-----------+---------+-----+
| Dst    | Src    | 802.1Q tag       | EtherType | Payload | FCS |
| MAC    | MAC    | (VLAN ID inside) |           |         |     |
+--------+--------+------------------+-----------+---------+-----+
| 6 B    | 6 B    | 4 B              | 2 B       | 46-1500 | 4 B |
+--------+--------+------------------+-----------+---------+-----+
```

The router uses the tag like this:

1. A frame arrives with tag 10. The router hands it to the subinterface set for VLAN 10.
2. The router routes the packet.
3. To send a packet into VLAN 20, the router uses the VLAN 20 subinterface, which adds tag 20.

The key config line is `encapsulation dot1Q 10`. It means: "this subinterface receives frames tagged 10, and adds tag 10 to frames it sends." The full config is in the Example section.

Trade-offs:

- **Good:** only one router port is needed, and new VLANs need no new cables.
- **Bad:** all routed traffic shares one link and crosses it twice, once in and once out. This is called hairpinning, because the traffic turns around like a hairpin.
- **Bad:** the router and its one link are a single point of failure.

**Bandwidth example.** The trunk is 1 Gbps, full duplex (each direction has its own 1 Gbps). Traffic of 600 Mbps goes from VLAN 10 to VLAN 20, and another 600 Mbps goes from VLAN 30 to VLAN 40. Toward the router, the link must carry 1200 Mbps. Away from the router, it must also carry 1200 Mbps. It can carry only 1000 Mbps each way, so packets are dropped or delayed, even though the switch itself has plenty of capacity.

### Method 3: Layer 3 Switch With SVIs

A Layer 3 switch, also called a multilayer switch, does normal switching and also does IP routing. It routes in hardware, on a special chip called an ASIC (Application-Specific Integrated Circuit).

Instead of cabling to a router, you create an **SVI** (Switched Virtual Interface). An SVI is a virtual Layer 3 interface for a whole VLAN. It is not tied to any one port. Any port in that VLAN can reach it. You create it with `interface vlan 10`.

```text
+------------------------- DSW1 --------------------------+
|                                                         |
|   SVI Vlan10 <---- internal routing ----> SVI Vlan20    |
|   192.168.10.1                            192.168.20.1  |
|       |                                       |         |
|   VLAN 10 ports                         VLAN 20 ports   |
+---- Gi1/0/1 --------------------------------- Gi1/0/2 --+
         |                                         |
        PC A                                      PC B
```

Three things must be done:

1. Create the VLAN.
2. Create the SVI, give it an IP address, and enable it with `no shutdown`.
3. Turn on routing with `ip routing`. On many Catalyst models this is off by default, so always check.

Packets never leave the box to be routed, so there is no hairpin on any link. A typical campus Layer 3 switch can route at full port speed. This is the standard design in campus networks today.

**Note on limits:** some entry-level switches route only in a limited way. For example, the Catalyst 2960 family with LAN Base software supports static routes only, and it needs an SDM template change plus a reload first. SDM (Switch Database Management) is a setting that shares out the switch's memory tables. Check your model.

### Routed Ports

A **routed port** is a physical port on a Layer 3 switch that acts like a router port. It belongs to no VLAN. You put an IP address directly on the port.

On Cisco Catalyst switches, you make one with `no switchport`:

```text
! DSW1 (Cisco IOS)
interface GigabitEthernet1/0/24
 description Routed link to R1
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
```

Routed ports are used for point-to-point links, such as a link from a Layer 3 switch to a router or firewall, or to another Layer 3 switch. Why use one rather than an SVI with a one-port VLAN?

- No VLAN to create or manage.
- It does not take part in Spanning Tree, so there is no wait for a port to start forwarding. See [Spanning Tree](09-spanning-tree.md).
- When the cable fails, the interface goes down at once and the route is removed.
- There is no MAC learning and no flooding on the port. Only two devices are on the link.

**Fuller truth:** whether a port starts as a switchport or a routed port varies by platform. Many Catalyst access models start as switchports. Some Catalyst 6500 and many Nexus setups start as routed. Check with `show interfaces <name> switchport`.

### Comparison

| Feature | One port per VLAN | Router-on-a-stick | Layer 3 switch (SVIs) |
|---|---|---|---|
| Router ports used | One per VLAN | One | None needed |
| Link between switch and router | Access ports | 802.1Q trunk | None (internal) |
| Gateway interface | Physical port | Subinterface | SVI |
| Scale | Very few VLANs | Tens of VLANs, if traffic is light | Hundreds or more, varies by model |
| Inter-VLAN throughput | One link per VLAN | Shared link, crossed twice | Usually full port speed |
| Forwarding done by | Router | Router | Switch hardware (ASIC) |
| Cost | High | Low | Medium |
| WAN, NAT, VPN features | Yes | Yes | Often limited |
| Single point of failure | Router | Router and one link | Switch (add a second one) |
| Common use today | Labs, CCNA study | Small sites, labs, firewalls | Campus and data centre |

A common full design combines them: the Layer 3 switch routes between VLANs, and a router or firewall at the edge handles [NAT](../04-ip-addressing/09-nat-and-pat.md) and internet access. A routed port connects the two.

Another variation is a firewall on a stick. A small office makes the firewall the gateway for every VLAN, using subinterfaces on one trunk. It works just like router-on-a-stick, and all inter-VLAN traffic gets inspected. See [Firewalls](../09-security/04-firewalls.md).

### Packet Walk-Through: Router-on-a-Stick

**Setup**

| Device | IP address | MAC address | Notes |
|---|---|---|---|
| PC A | 192.168.10.11/24 | 0200.0000.00aa | VLAN 10, gateway 192.168.10.1 |
| PC B | 192.168.20.22/24 | 0200.0000.00bb | VLAN 20, gateway 192.168.20.1 |
| R1 Gi0/0.10 | 192.168.10.1/24 | 0200.0000.00f1 | Tag 10 |
| R1 Gi0/0.20 | 192.168.20.1/24 | 0200.0000.00f1 | Tag 20 |

PC A pings PC B. Assume all caches are empty.

```text
 PC A       SW1          R1          SW1        PC B
  |          |            |           |           |
  |-frame 1->|            |           |           |
  | untagged |-frame 2--->|           |           |
  |          |  tag 10    |-frame 3-->|           |
  |          |            |  tag 20   |-frame 4-->|
  |          |            |           | untagged  |
```

SW1 is drawn twice because the frame passes through it twice.

1. **PC A checks the destination.** 192.168.20.22 is not in 192.168.10.0/24, so the packet must go to the gateway, 192.168.10.1.
2. **PC A runs ARP for the gateway.** ARP (Address Resolution Protocol) finds the MAC address for an IP address. PC A broadcasts "who has 192.168.10.1?".
3. **SW1 floods the ARP request** to all VLAN 10 ports. On the trunk it adds tag 10. The broadcast does not reach VLAN 20 ports.
4. **R1 replies.** The tag sends the frame to Gi0/0.10, and that subinterface owns 192.168.10.1. R1 replies with MAC 0200.0000.00f1, tagged 10.
5. **PC A sends frame 1.** The destination MAC is R1, but the destination IP is PC B. No tag is added.
6. **SW1 forwards it as frame 2.** It finds R1's MAC in VLAN 10 on the trunk port, then adds tag 10.
7. **R1 receives frame 2.** Tag 10 maps it to Gi0/0.10. The destination MAC is its own, so it removes the Ethernet header and tag.
8. **R1 routes the packet.** It finds 192.168.20.0/24 connected on Gi0/0.20, and it lowers the TTL from 64 to 63. TTL (Time To Live) is a hop counter in the IP header. It stops packets from looping forever.
9. **R1 runs ARP for PC B** through Gi0/0.20. This request carries tag 20, so it is flooded only in VLAN 20. PC B replies.
10. **R1 sends frame 3.** It builds a new Ethernet header with its own MAC as source and PC B's MAC as destination, and adds tag 20. The frame leaves the same physical port it came in on.
11. **SW1 delivers it as frame 4.** It finds PC B's MAC in VLAN 20, removes the tag because the port is an access port, and sends the frame.
12. **PC B answers.** The reply is for another subnet, so it goes to 192.168.20.1. The same path is used in reverse.

Here is what changed at each stage (PC A is Linux, so the first TTL is 64):

| Field | Frame 1 | Frame 2 | Frame 3 | Frame 4 |
|---|---|---|---|---|
| Link | PC A to SW1 | SW1 to R1 | R1 to SW1 | SW1 to PC B |
| Source MAC | 0200.0000.00aa | 0200.0000.00aa | 0200.0000.00f1 | 0200.0000.00f1 |
| Destination MAC | 0200.0000.00f1 | 0200.0000.00f1 | 0200.0000.00bb | 0200.0000.00bb |
| 802.1Q tag | None | 10 | 20 | None |
| Source IP | 192.168.10.11 | 192.168.10.11 | 192.168.10.11 | 192.168.10.11 |
| Destination IP | 192.168.20.22 | 192.168.20.22 | 192.168.20.22 | 192.168.20.22 |
| TTL | 64 | 64 | 63 | 63 |

Remember this pattern. **IP addresses stay the same. MAC addresses and the VLAN tag change at the router. TTL drops by one.** (With NAT, IP addresses can change too, but plain routing does not change them.)

### Packet Walk-Through: Layer 3 Switch

Same PCs, but now the gateways are SVIs on DSW1 with MAC 0200.0000.00e1.

1. PC A sends a frame to the MAC of 192.168.10.1, the Vlan10 SVI. ARP works as before, but now the switch itself answers.
2. The frame enters Gi1/0/1, which is an access port in VLAN 10.
3. DSW1 sees that the destination MAC is its own router MAC. So it does a Layer 3 lookup, not a normal Layer 2 forward.
4. The routing lookup finds 192.168.20.0/24 connected on Vlan20.
5. DSW1 lowers the TTL and rewrites the MAC addresses: source 0200.0000.00e1, destination 0200.0000.00bb.
6. It does a Layer 2 lookup in VLAN 20 and finds PC B on Gi1/0/2.
7. The frame leaves Gi1/0/2 with no tag.

The result is the same as with a router, but all of it happens inside one chip. No link is crossed twice.

If there were a trunk between DSW1 and an access switch, the routed frame would leave that trunk with tag 20. Tags still matter on trunks.

## Example

### Example 1: Full Router-on-a-Stick Build

**Design**

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | SALES | 192.168.10.0/24 | 192.168.10.1 |
| 20 | ENGINEERING | 192.168.20.0/24 | 192.168.20.1 |
| 99 | MGMT | 192.168.99.0/24 | 192.168.99.1 |

```text
                 +------------------+
                 |        R1        |
                 +--------+---------+
                          | Gi0/0
                          |
                          | 802.1Q trunk (VLAN 10, 20, 99)
                          |
                          | Gi1/0/24
                 +--------+---------+
                 |       SW1        |
                 |   192.168.99.2   |
                 +---+----------+---+
             Gi1/0/1 |          | Gi1/0/2
           (VLAN 10) |          | (VLAN 20)
                  +--+--+    +--+--+
                  | PC A|    | PC B|
                  +-----+    +-----+
           192.168.10.11/24  192.168.20.22/24
```

**Switch config (Cisco IOS):**

```text
vlan 10
 name SALES
vlan 20
 name ENGINEERING
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 description PC A
 switchport mode access
 switchport access vlan 10
!
interface GigabitEthernet1/0/2
 description PC B
 switchport mode access
 switchport access vlan 20
!
interface GigabitEthernet1/0/24
 description Trunk to R1 Gi0/0
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
interface Vlan99
 description Management address of this switch
 ip address 192.168.99.2 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.99.1
```

On older switches that support both ISL and 802.1Q, add `switchport trunk encapsulation dot1q` before `switchport mode trunk`. ISL was an old Cisco trunking method, and newer switches accept only 802.1Q.

SW1 is doing no routing here. Its Vlan99 SVI exists only so you can log in to manage the switch.

**Router config (Cisco IOS):**

```text
interface GigabitEthernet0/0
 description Trunk to SW1
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.10
 description Gateway for SALES
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 description Gateway for ENGINEERING
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
!
interface GigabitEthernet0/0.99
 description Gateway for MGMT
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.0
```

`no shutdown` goes on the physical port. Subinterfaces follow their parent. The `encapsulation` line must come before `ip address`, or IOS rejects the address.

**Check the trunk on SW1:**

```text
SW1# show interfaces trunk

Port        Mode             Encapsulation  Status        Native vlan
Gi1/0/24    on               802.1q         trunking      1

Port        Vlans allowed on trunk
Gi1/0/24    10,20,99

Port        Vlans allowed and active in management domain
Gi1/0/24    10,20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/24    10,20,99
```

**Check the interfaces and routes on R1:**

```text
R1# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     unassigned      YES unset  up                    up
GigabitEthernet0/0.10  192.168.10.1    YES manual up                    up
GigabitEthernet0/0.20  192.168.20.1    YES manual up                    up
GigabitEthernet0/0.99  192.168.99.1    YES manual up                    up

R1# show ip route connected
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, GigabitEthernet0/0.10
L        192.168.10.1/32 is directly connected, GigabitEthernet0/0.10
      192.168.20.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.20.0/24 is directly connected, GigabitEthernet0/0.20
L        192.168.20.1/32 is directly connected, GigabitEthernet0/0.20
      192.168.99.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.99.0/24 is directly connected, GigabitEthernet0/0.99
L        192.168.99.1/32 is directly connected, GigabitEthernet0/0.99
```

The `C` lines are connected networks. The `L` lines are the router's own addresses. No static route is needed for these VLANs, because the router is directly connected to all of them.

**After a ping, check ARP and the MAC table:**

```text
R1# show ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.10.1            -   0200.0000.00f1  ARPA   GigabitEthernet0/0.10
Internet  192.168.10.11           2   0200.0000.00aa  ARPA   GigabitEthernet0/0.10
Internet  192.168.20.1            -   0200.0000.00f1  ARPA   GigabitEthernet0/0.20
Internet  192.168.20.22           2   0200.0000.00bb  ARPA   GigabitEthernet0/0.20

SW1# show mac address-table dynamic
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    0200.0000.00aa    DYNAMIC     Gi1/0/1
  10    0200.0000.00f1    DYNAMIC     Gi1/0/24
  20    0200.0000.00bb    DYNAMIC     Gi1/0/2
  20    0200.0000.00f1    DYNAMIC     Gi1/0/24
```

R1's MAC appears twice, once per VLAN, on the same port. This is normal, because the switch keeps a separate table for each VLAN.

**Test from PC A (PC B is a Linux machine, so its first TTL is 64):**

Linux:

```bash
$ ip route
default via 192.168.10.1 dev eth0
192.168.10.0/24 dev eth0 proto kernel scope link src 192.168.10.11

$ ping -c 2 192.168.20.22
PING 192.168.20.22 (192.168.20.22) 56(84) bytes of data.
64 bytes from 192.168.20.22: icmp_seq=1 ttl=63 time=1.02 ms
64 bytes from 192.168.20.22: icmp_seq=2 ttl=63 time=0.84 ms

$ traceroute -n 192.168.20.22
traceroute to 192.168.20.22 (192.168.20.22), 30 hops max, 60 byte packets
 1  192.168.10.1  0.512 ms  0.478 ms  0.466 ms
 2  192.168.20.22  0.911 ms  0.874 ms  0.860 ms
```

Windows:

```powershell
PS C:\> ipconfig

Ethernet adapter Ethernet:

   IPv4 Address. . . . . . . . . . . : 192.168.10.11
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.10.1

PS C:\> ping -n 2 192.168.20.22

Pinging 192.168.20.22 with 32 bytes of data:
Reply from 192.168.20.22: bytes=32 time=1ms TTL=63
Reply from 192.168.20.22: bytes=32 time<1ms TTL=63

PS C:\> tracert -d 192.168.20.22

Tracing route to 192.168.20.22 over a maximum of 30 hops

  1    <1 ms    <1 ms    <1 ms  192.168.10.1
  2     1 ms    <1 ms    <1 ms  192.168.20.22

Trace complete.
```

TTL 63 proves the reply crossed exactly one router. The traceroute shows the gateway as hop 1.

### Example 2: Full Layer 3 Switch Build With a Routed Uplink

Now the routing moves into DSW1. An edge router, R1, connects through a routed port.

```text
                    +--------------+
                    |      R1      |  (edge router, NAT, internet)
                    +------+-------+
                           | Gi0/0  10.0.0.2/30
                           |
                           | Gi1/0/24  10.0.0.1/30  (routed port)
           +---------------+----------------+
           |              DSW1              |
           |  Vlan10 192.168.10.1/24        |
           |  Vlan20 192.168.20.1/24        |
           +-----+-----------------------+--+
         Gi1/0/1 |                       | Gi1/0/2
                PC A                    PC B
```

**DSW1 config (Cisco IOS):**

```text
ip routing
!
vlan 10
 name SALES
vlan 20
 name ENGINEERING
!
interface GigabitEthernet1/0/1
 description PC A
 switchport mode access
 switchport access vlan 10
!
interface GigabitEthernet1/0/2
 description PC B
 switchport mode access
 switchport access vlan 20
!
interface Vlan10
 description Gateway for SALES
 ip address 192.168.10.1 255.255.255.0
 no shutdown
!
interface Vlan20
 description Gateway for ENGINEERING
 ip address 192.168.20.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0/24
 description Routed link to R1
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.0.2
```

**R1 config (Cisco IOS):**

```text
interface GigabitEthernet0/0
 description Routed link to DSW1
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
ip route 192.168.10.0 255.255.255.0 10.0.0.1
ip route 192.168.20.0 255.255.255.0 10.0.0.1
```

R1 needs those two routes so that return traffic can find the VLANs. For more detail, see [Static and Default Routes](../05-routing/03-static-and-default-routes.md). In larger networks, a routing protocol such as [OSPF](../05-routing/06-ospf.md) does this job.

**Verify on DSW1:**

```text
DSW1# show ip interface brief | exclude unassigned
Interface              IP-Address      OK? Method Status                Protocol
Vlan10                 192.168.10.1    YES manual up                    up
Vlan20                 192.168.20.1    YES manual up                    up
GigabitEthernet1/0/24  10.0.0.1        YES manual up                    up

DSW1# show ip route
Gateway of last resort is 10.0.0.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.0.2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, GigabitEthernet1/0/24
L        10.0.0.1/32 is directly connected, GigabitEthernet1/0/24
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, Vlan10
L        192.168.10.1/32 is directly connected, Vlan10
      192.168.20.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.20.0/24 is directly connected, Vlan20
L        192.168.20.1/32 is directly connected, Vlan20

DSW1# show interfaces GigabitEthernet1/0/24 switchport
Name: Gi1/0/24
Switchport: Disabled
```

`Switchport: Disabled` confirms that Gi1/0/24 is a routed port.

### Example 3: Linux as Router-on-a-Stick

The same idea works on Linux. Here `eth0` connects to a switch trunk. Each VLAN interface works like a Cisco subinterface.

```bash
# Linux: create tagged interfaces for VLANs 10 and 20
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip link add link eth0 name eth0.20 type vlan id 20

sudo ip addr add 192.168.10.1/24 dev eth0.10
sudo ip addr add 192.168.20.1/24 dev eth0.20

sudo ip link set eth0 up
sudo ip link set eth0.10 up
sudo ip link set eth0.20 up

# Turn on IP forwarding (off by default on most distributions)
sudo sysctl -w net.ipv4.ip_forward=1
```

These commands do not survive a reboot. Make them permanent with your distribution's network tools. A firewall rule in the forward path can still block traffic. See [Linux Networking](../11-cloud-and-containers/01-linux-networking.md).

## Going Deeper

### Subinterface, SVI, Routed Port: Side by Side

| | Subinterface | SVI | Routed port |
|---|---|---|---|
| What it is | Logical slice of a physical port | Virtual interface for a VLAN | Physical port with an IP address |
| Example name | Gi0/0.10 | Vlan10 | Gi1/0/24 |
| Found on | Routers, some Layer 3 switches | Layer 3 switches | Layer 3 switches |
| Linked to a VLAN | Yes, through a tag | Yes, the whole VLAN | No |
| Status depends on | Parent physical port | Active ports in the VLAN | Its own cable |
| Takes part in STP | No | The VLAN's ports do | No |

Names differ by vendor. Cisco NX-OS uses `interface vlan 10`, but you must first run `feature interface-vlan`. Arista EOS also uses `interface Vlan10`. Juniper calls this an IRB (Integrated Routing and Bridging) interface, for example `irb.10`.

### When Is an SVI Up?

On Cisco, an SVI comes up only when all of these are true:

1. The VLAN exists in the VLAN database and is active.
2. The SVI has no `shutdown` command.
3. At least one port in the VLAN is up and in Spanning Tree forwarding state. A trunk that carries the VLAN counts.

This logic is called **autostate**. If the last active port in VLAN 10 goes down, Vlan10 goes down too, and its connected route is removed. That is useful, because other routers stop sending traffic to a dead subnet.

| Status / Protocol | Likely cause |
|---|---|
| administratively down / down | `no shutdown` is missing |
| down / down | VLAN does not exist, or is shut down |
| up / down | No active port in the VLAN, a common sight in labs with no PCs plugged in |
| up / up | Working |

The exact status words can vary by version. Ports can be left out of this check with `switchport autostate exclude`, for example ports used only for monitoring.

### How Hardware Routing Works

On Cisco, Layer 3 switches use **CEF** (Cisco Express Forwarding). The CPU builds two tables, and the ASIC uses copies of them:

- **FIB (Forwarding Information Base):** a copy of the routing table, tuned for fast lookups.
- **Adjacency table:** the ready-made Layer 2 headers for next hops, built from ARP data.

These tables sit in **TCAM** (Ternary Content-Addressable Memory), a special memory that finds a match in one step, so lookups do not slow down as the table grows.

Control work stays on the CPU. This includes ARP, routing protocols, and packets sent to the switch itself. Packets that need special handling, such as an expired TTL, are also sent to the CPU. So high CPU on a Layer 3 switch does not always mean the data path is slow.

This is also why the first ping often fails on a Cisco device. When a device starts a ping and has no ARP entry yet, the first packet is usually dropped while ARP resolves. Output of `.!!!!` is normal. Hosts usually queue the packet instead.

### Routed Port Details

- On a point-to-point link, /30 gives two usable addresses. Many networks use /31 instead, which saves addresses (RFC 3021).
- On some platforms, each routed port uses up one hidden internal VLAN from the extended range. Check with `show vlan internal usage`, where the platform supports it.
- A link between two Layer 3 switches can also be a Layer 3 EtherChannel, which bundles ports into one routed link. See [Link Aggregation](10-link-aggregation.md).
- Some platforms support subinterfaces on routed ports. Support varies.

### The Native VLAN on a Router

The native VLAN is the VLAN whose frames cross a trunk without a tag. A router subinterface can be marked as native:

```text
! R1 (Cisco IOS): this subinterface sends and receives untagged frames
interface GigabitEthernet0/0.999
 encapsulation dot1Q 999 native
```

The native VLAN must match on both ends of the trunk. Otherwise, untagged traffic lands in the wrong VLAN. In the examples above, VLAN 1 is the native VLAN but it is left off the allowed list, so no user data crosses in VLAN 1. For safety, many networks also set the native VLAN to an unused number. See [Layer 2 Security](11-layer2-security.md).

### DHCP Across VLANs

A DHCP client finds its server with a broadcast, and a broadcast cannot leave its VLAN. If the server is in another VLAN, the gateway must relay the request. Here the server is 192.168.10.5, and the clients are in VLAN 20:

```text
! DSW1 (Cisco IOS): on the VLAN where the clients are
interface Vlan20
 ip helper-address 192.168.10.5
```

The gateway turns the broadcast into a unicast to the server and writes its own address, 192.168.20.1, into the `giaddr` field. The server uses that field to pick an address from the correct pool. See [DHCP](../07-application-protocols/04-dhcp.md).

### Controlling Traffic Between VLANs

A Layer 3 switch routes between all its SVIs by default. Once `ip routing` is on, every VLAN with an SVI can reach every other. If that is not what you want, add rules:

```text
! DSW1 (Cisco IOS): stop Sales from reaching Engineering
ip access-list extended SALES_IN
 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip any any
!
interface Vlan10
 ip access-group SALES_IN in
```

Here `in` means traffic coming from VLAN 10 hosts into the SVI. This ACL (Access Control List) has no memory of connections. It also blocks VLAN 10's replies to anything that VLAN 20 starts. See [Access Control Lists](../09-security/05-access-control-lists.md).

Other choices:

- Send traffic through a [firewall](../09-security/04-firewalls.md) for stateful inspection. A stateful device remembers connections and allows replies by itself.
- Use **VRF** (Virtual Routing and Forwarding), which gives one device several separate routing tables, so groups of VLANs stay apart unless you link them. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### Gateway Redundancy

If the only gateway fails, the whole VLAN loses contact with other subnets. Real networks use two Layer 3 switches with a shared virtual gateway address, using HSRP or VRRP. Each SVI has its own real IP, and hosts point at the virtual IP. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

Newer data centres use an anycast gateway, where every leaf switch holds the same gateway address. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

### IPv6

Nothing changes in the design. You add IPv6 addresses to the same subinterfaces or SVIs.

```text
! DSW1 (Cisco IOS)
ipv6 unicast-routing
!
interface Vlan10
 ipv6 address 2001:db8:10::1/64
!
interface Vlan20
 ipv6 address 2001:db8:20::1/64
```

Hosts normally learn their gateway from router advertisements, not from a manual setting. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### Where the Layer 3 Boundary Lives

The boundary is the point where Layer 2 ends and routing starts. In a classic campus, it sits at the distribution switches, and trunks run down to Layer 2 access switches. In a routed access design, it moves to the access switch, so each uplink is a routed port and Spanning Tree has less to do. See [Network Design Models](../10-enterprise-and-datacenter/01-network-design-models.md).

### Troubleshooting Order

When two VLANs cannot talk, check from the bottom up:

1. **Host:** is the IP, mask and gateway correct? Use `ipconfig` on Windows or `ip addr` and `ip route` on Linux.
2. **Gateway reachable?** Ping the gateway from the host. If this fails, the problem is in that VLAN.
3. **Access port:** is it in the right VLAN? Use `show vlan brief`.
4. **Trunk:** is the VLAN allowed and active? Use `show interfaces trunk`.
5. **Gateway interface:** is it up/up? Use `show ip interface brief`.
6. **Tag match:** does `encapsulation dot1Q` match the VLAN? Use `show running-config interface`.
7. **Routing:** on a Layer 3 switch, is `ip routing` on? Use `show ip route`. If you see only a default gateway line and no routing table, routing is off.
8. **Filters:** is an ACL or a host firewall blocking traffic? Windows Firewall blocks inbound ping by default on many systems.
9. **Other side:** repeat for the destination. Traffic must work in both directions.

## Common Mistakes

- **Forgetting `ip routing` on a Layer 3 switch.** SVIs show up/up and PCs can ping their own gateway, but not the other VLAN.
- **Thinking `ip routing` is the end of the job.** The switch also needs a default route for traffic that leaves the site, and the far router needs routes back.
- **Using `ip default-gateway` on a routing switch.** Once `ip routing` is on, that command is ignored. Use `ip route 0.0.0.0 0.0.0.0 <next-hop>` instead.
- **Leaving out a VLAN.** An SVI stays down if the VLAN was never created, and creating `interface vlan 30` does not create VLAN 30 on Cisco.
- **Thinking the subinterface number sets the VLAN.** Gi0/0.10 is only a name. The `encapsulation dot1Q` line sets the VLAN. Matching them is good practice only for readability.
- **Wrong tag.** Gi0/0.10 with `encapsulation dot1Q 20` puts the gateway for one VLAN in the other. Ping fails with no error message.
- **VLAN missing from the trunk's allowed list.** The router never sees that VLAN's frames.
- **Access port facing a router configured for subinterfaces.** Untagged frames reach the physical port and no subinterface handles them. The port must be a trunk.
- **Thinking a VLAN tag survives routing.** The router removes the old frame and builds a new one. Tag 10 goes in and tag 20 comes out.
- **Thinking routing changes the IP addresses.** Only MACs, the tag and TTL change (and the header checksum, because TTL changed). NAT is a separate job.
- **Treating a Layer 3 switch as a full router.** NAT, VPN and many WAN features are often missing or limited, depending on the model.
- **Believing VLANs alone give security after routing is on.** With SVIs, all VLANs can talk until you add ACLs or a firewall.
- **Wrong subnet mask on a host.** A host with 192.168.10.11/16 thinks 192.168.20.22 is local and sends ARP directly. That broadcast stays in VLAN 10. It can still work by luck, because proxy ARP is on by default on Cisco interfaces: the router answers for the far host. This hides the real error. Proxy ARP means a router answers ARP on behalf of another device.
- **Using a patch cable between two VLANs.** Joining an access port in VLAN 10 to an access port in VLAN 20 merges them into one broadcast domain. That is not routing, and it is a fault, not a fix.

## Interview Questions

**Q: Why can't two hosts in different VLANs talk through a switch?**

A: Each VLAN is a separate broadcast domain and normally a separate subnet. A Layer 2 switch forwards frames only inside their VLAN, so there is no path between VLANs. A Layer 3 device has to route the packets, and each host sends the traffic to its default gateway.

**Q: What are the ways to do inter-VLAN routing?**

A: There are three. A router with one physical port per VLAN, router-on-a-stick with one trunk and one subinterface per VLAN, and a Layer 3 switch with one SVI per VLAN. Most networks today use a Layer 3 switch, because it routes in hardware at full speed.

**Q: What is router-on-a-stick, and what is its main weakness?**

A: One router port connects to a switch trunk. The router has a subinterface for each VLAN, each with its own IP and `encapsulation dot1Q` tag. The weakness is that all inter-VLAN traffic shares one link and crosses it twice, so the link is a bottleneck and a single point of failure.

**Q: What is an SVI, and when is it up?**

A: An SVI is a virtual Layer 3 interface for a VLAN, created with `interface vlan 10`. It is usually the gateway for that VLAN. On Cisco it comes up when the VLAN exists, the SVI is not shut down, and at least one port in the VLAN is up and forwarding, including a trunk that carries the VLAN.

**Q: What is the difference between an SVI and a routed port?**

A: An SVI is virtual and serves a whole VLAN of ports. A routed port is one physical port with an IP address, set with `no switchport`, and it belongs to no VLAN. I use SVIs as gateways for host VLANs, and routed ports for point-to-point links, because they fail fast and need no Spanning Tree.

**Q: Walk me through a packet from a host in VLAN 10 to a host in VLAN 20 using router-on-a-stick.**

A: The host sees that the destination is in another subnet and sends the frame to the gateway's MAC. The switch forwards it on the trunk with tag 10. The router receives it on the VLAN 10 subinterface, lowers the TTL, and finds the destination on the VLAN 20 subinterface. It builds a new frame with new MACs and tag 20, and sends it back down the same link. The switch removes the tag and delivers the frame. IP addresses stay the same.

**Q: On a router-on-a-stick, all subinterfaces have the same MAC address. Is that a problem?**

A: No. The switch keeps a MAC table per VLAN, so the same MAC can appear in VLAN 10 and VLAN 20 on the same port. The router tells the VLANs apart by the tag, not by the MAC.

**Q: A Layer 3 switch has SVIs that are up, and PCs can ping their gateway, but not the other VLAN. What do you check?**

A: First I check `ip routing`, since it is often off by default and this is the classic symptom. Then I check that the hosts have the right gateway, and `show ip route` for connected routes. After that, I look for ACLs on the SVIs and for firewalls on the hosts.

**Q: Which fields change when a packet is routed between VLANs?**

A: The source and destination MAC addresses change, and the VLAN tag changes if the link is a trunk. TTL goes down by one, and the IP header checksum is recalculated because of that. The source and destination IP addresses do not change unless NAT is involved.

**Q: Why would you still have a router if you have a Layer 3 switch?**

A: A Layer 3 switch is great at fast LAN routing, but it usually lacks strong WAN, NAT and VPN features, and it may have small route tables. So the Layer 3 switch handles inter-VLAN traffic, and a router or firewall sits at the edge. A routed port usually connects them.

**Q: How do clients in VLAN 20 get DHCP from a server in VLAN 10?**

A: DHCP discovery is a broadcast, so it cannot cross VLANs. I add `ip helper-address` with the server's IP on the VLAN 20 gateway. The gateway relays the request as a unicast and adds its own address in the `giaddr` field, so the server picks the right pool.

**Q: Can two hosts in the same subnet but in different VLANs talk?**

A: Not normally. Each host thinks the other is local, so it sends ARP directly, and that broadcast stays in its own VLAN. The router gets no chance to help, because the hosts never use the gateway. The design is wrong: each VLAN should have its own subnet.

## Quick Recap

- VLANs are separate broadcast domains, so traffic between them needs a Layer 3 device.
- One port per VLAN is simple but does not scale. Router-on-a-stick saves ports, but its trunk is a bottleneck, and traffic crosses it twice.
- A Layer 3 switch uses SVIs and hardware routing, and it is the normal choice today. Remember `ip routing`, and create the VLAN first.
- A routed port (`no switchport`) is a physical Layer 3 port for point-to-point links. It belongs to no VLAN.
- When a packet is routed, the MACs and VLAN tag change, TTL drops by one, and IP addresses stay the same.
- Once routing is on, all VLANs can talk, so use ACLs or a firewall to control traffic, and `ip helper-address` to relay DHCP.
- Troubleshoot in order: host, access port, trunk, gateway interface, routing, filters.
