# IPv4 Classes (Classful Addressing)

**Level:** Beginner | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [IPv4 Addressing Basics](01-ipv4-basics.md)

## In One Line

Classful addressing was the original IPv4 plan: the first few bits of an address put it in class A, B, C, D or E, and that class fixed the size of the network. The Internet stopped using this plan in 1993, but the words are still everywhere.

## Why It Matters

IPv4 has about 4.3 billion addresses. Someone had to decide how to share them out. Classes were the first answer.

The plan had a clear benefit. A router could look at the first bits of an address and know where the network part ended. It needed no extra information.

The plan also had a serious problem. It offered only three network sizes, and the gaps between them were huge. Addresses were wasted, and the Internet was running out of them. This failure is the reason CIDR, the modern system, exists.

Interviewers still ask for three reasons:

- It tests whether you can work with binary, not only repeat facts.
- The history explains why subnet masks and CIDR look the way they do.
- Classful words are still in tools, exams and daily talk, so you must understand them.

## Simple Explanation

Imagine a city that sells land, but only in three plot sizes:

- **Huge plot (class A):** room for 16 million houses.
- **Medium plot (class B):** room for 65 thousand houses.
- **Small plot (class C):** room for 254 houses.

The first digit of a plot number tells you the size. No map is needed.

Now a school needs room for 300 houses. A small plot is too small, so it must buy a medium plot. More than 65,000 house spaces stay empty, and nobody else can use them. This is exactly what went wrong with IPv4 classes.

The city has two more zones. Zone D is like a set of public notice boards: a message posted there reaches everyone who chose to read that board. Zone E is fenced off for future use.

**Where the analogy stops:** real land is sold once and then stays put. IP blocks can be split up, joined together and moved between owners. Later rules (subnetting and CIDR) made this normal.

## How It Works

### Terms You Need First

- **Bit:** a single 0 or 1.
- **Octet:** a group of 8 bits. An IPv4 address has 4 octets, such as 192.168.1.10. The **first octet** here is 192.
- **Network part:** the bits that name the network, like a street name.
- **Host part:** the bits that name one device on that network, like a house number.
- **Subnet mask:** a 32-bit pattern that marks which bits are the network part. In 255.255.255.0, the 255s mark network bits.
- **Prefix length:** the same idea written as a count of network bits, such as /24.
- **Class:** one of five groups, A to E, chosen by the first bits of the address.
- **Leading bits:** the first bits of the first octet, read from the left.

### The Five Classes

| Class | Leading bits | First octet range | Default mask | Prefix | Network bits / host bits | Networks | Usable hosts per network | Used for |
|---|---|---|---|---|---|---|---|---|
| A | 0 | 1-126 (bit range 0-127) | 255.0.0.0 | /8 | 8 / 24 | 126 usable (128 by math) | 16,777,214 | Very large networks |
| B | 10 | 128-191 | 255.255.0.0 | /16 | 16 / 16 | 16,384 | 65,534 | Medium to large networks |
| C | 110 | 192-223 | 255.255.255.0 | /24 | 24 / 8 | 2,097,152 | 254 | Small networks |
| D | 1110 | 224-239 | None | None | No split | Not applicable | Not applicable | Multicast |
| E | 1111 | 240-255 | None | None | No split | Not applicable | Not applicable | Reserved |

Learn this table. Most classful interview questions come straight from it.

### How the Leading Bits Create the Ranges

The ranges look random, but they are not. They come from the leading bits. This layout shows how each class uses the 32 bits:

```text
         Octet 1    Octet 2    Octet 3    Octet 4
        +----------+----------+----------+----------+
Class A |0 Net (7) |        Host (24 bits)          |
        +----------+----------+----------+----------+
Class B |10 Network (14 bits) |   Host (16 bits)    |
        +----------+----------+----------+----------+
Class C |110 Network (21 bits)           |Host (8)  |
        +----------+----------+----------+----------+
Class D |1110 Multicast group (28 bits)             |
        +----------+----------+----------+----------+
Class E |1111 Reserved (28 bits)                    |
        +----------+----------+----------+----------+
```

Fix the leading bits, then set the other bits of the first octet to all 0s and all 1s. That gives you the lowest and highest value:

| Class | Lowest first octet | Highest first octet | Decimal |
|---|---|---|---|
| A | **0**0000000 | **0**1111111 | 0-127 |
| B | **10**000000 | **10**111111 | 128-191 |
| C | **110**00000 | **110**11111 | 192-223 |
| D | **1110**0000 | **1110**1111 | 224-239 |
| E | **1111**0000 | **1111**1111 | 240-255 |

Each class starts where the previous one ends. Classes A to D each take half of the space that was left:

```text
First octet: 0              128        192    224  240  255
             |---------------|----------|------|----|----|
             |    Class A    | Class B  |  C   | D  | E  |
             |      50%      |   25%    |12.5% |6.25|6.25|
                                             (not to exact scale)
```

**Memory trick:** add the bit values 128, 64, 32 and 16. Class B starts at 128, class C at 128 + 64 = 192, class D at 192 + 32 = 224, and class E at 224 + 16 = 240.

### How to Find the Class of an Address

You have two methods. Both give the same answer.

**Fast method:** compare the first octet with the ranges. 172 is between 128 and 191, so it is class B.

**Binary method:** this is what a classful device does. It reads the bits from the left and stops at the first 0.

```text
       Read the first octet in binary
                   |
        first bit 0?  --- yes ---> Class A (0...)
                   | no
       second bit 0?  --- yes ---> Class B (10...)
                   | no
        third bit 0?  --- yes ---> Class C (110...)
                   | no
       fourth bit 0?  --- yes ---> Class D (1110...)
                   | no
                   +-------------> Class E (1111...)
```

### Default Masks

A **default mask** is the mask that matches the class, as listed in the table above. It is also called the **classful mask** or **natural mask**.

In the first version of IPv4, addresses had no mask at all. The class alone gave the boundary. Masks arrived in 1985, when subnetting was added (RFC 950).

Classes D and E have no default mask because they have no network part and host part.

### How the Counts Are Calculated

Two simple formulas give every number in the table.

**Number of networks = 2^(network bits - leading bits).** Leading bits are fixed, so they cannot change to create new networks.

- Class A: 8 - 1 = 7 free bits, so 2^7 = 128. Remove 0 and 127, and 126 are usable.
- Class B: 16 - 2 = 14 free bits, so 2^14 = 16,384.
- Class C: 24 - 3 = 21 free bits, so 2^21 = 2,097,152.

**Usable hosts per network = 2^(host bits) - 2.** Two values are removed in every network:

- **All host bits 0:** this is the network address, the name of the network itself.
- **All host bits 1:** this is the broadcast address, which means "every device on this network".

So class A has 2^24 - 2 = 16,777,214 hosts, class B has 2^16 - 2 = 65,534, and class C has 2^8 - 2 = 254.

**Fuller truth:** some older books list 16,382 class B and 2,097,150 class C networks. Those counts subtract two reserved networks. Those reservations ended long ago, so the normal answer today is 16,384 and 2,097,152. Some blocks inside each class are still special, as [Public, Private and Special IP Addresses](03-public-private-and-special-addresses.md) explains.

### Why 127 Is Missing From Class A

By its bits, 127 belongs to class A: 01111111 starts with 0. It is missing from the usual table because the whole block 127.0.0.0/8 is reserved for **loopback**.

Loopback means a device sends traffic to itself. A packet sent to any 127.x.x.x address must stay inside the device. It must never appear on a network cable or Wi-Fi link. RFC 1122 requires this behaviour.

Think of writing a note and placing it in your own desk tray. It never goes to the post office. The analogy stops here: the note still passes through the device's own network software. That is why a loopback ping proves the local TCP/IP stack works.

127.0.0.1 is the usual loopback address, but the entire block works. It covers more than 16 million addresses. That is 1/256 of all IPv4 space, spent on one local function.

The first octet 0 is not usable either. 0.0.0.0/8 means "this network", and 0.0.0.0 can mean "this host" or an unknown address. So class A has 128 possible networks but only 126 usable ones. Books that show class A as 0-127 are listing the bit range. Books that show 1-126 are listing the usable range.

### Class D: Multicast

**Multicast** means one sender reaches a group of devices that chose to receive it. Think of a TV channel: only people tuned in receive it. Details are in [Unicast, Broadcast, Multicast and Anycast](13-unicast-broadcast-multicast-anycast.md).

A class D address names a group, not a device. So it has no host part and no default mask. It is used as a destination, not as a source. Common examples are 224.0.0.1 (all hosts on the local link), 224.0.0.2 (all routers on the local link), 224.0.0.5 (all OSPF routers) and 224.0.0.9 (RIPv2 routers).

Today this block is written 224.0.0.0/4. The /4 only describes the size of the block. It is not a subnet mask for hosts.

### Class E: Reserved

Class E, written today as 240.0.0.0/4, is reserved for future use. The standards do not allow it on the public Internet. Many devices and systems refuse these addresses, or treat them differently, so support varies.

The address 255.255.255.255 sits in this range. It is the **limited broadcast** address, which means "every device on my local network". Routers do not forward it. DHCP clients use it when they do not yet have an address. See [DHCP](../07-application-protocols/04-dhcp.md).

### Why Classes Were Retired

Three problems ended the classful system.

**1. Sizes did not fit.** The sizes jump from 254 to 65,534 to 16,777,214. There is nothing in between.

| Hosts needed | Class required | Addresses left unused |
|---|---|---|
| 300 | B | 65,234 |
| 5,000 | B | 60,534 |
| 70,000 | A | 16,707,214 |

Class A alone held 50% of all IPv4 addresses, shared by at most 126 organisations. Early holders included Apple (17.0.0.0/8) and MIT (18.0.0.0/8).

**2. Class B ran low.** Class C was too small for most organisations, so they all wanted class B. By the early 1990s, there was real concern that class B would run out.

**3. Routing tables grew too fast.** A **routing table** is a router's list of known networks. Suppose a company took eight class C networks instead of one class B. Every Internet backbone router now needed eight entries for one company. Routers of that time had limited memory and CPU.

### What Replaced Classes

**CIDR** (Classless Inter-Domain Routing) replaced classes in 1993. It removes the fixed sizes. Any prefix length can be used, and the mask or prefix says where the network part ends. The first bits of an address no longer decide its size. CIDR is covered in [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md).

| Year | Event | Reference |
|---|---|---|
| 1981 | IPv4 published with classes A, B and C | RFC 791 |
| 1985 | Subnetting added, so one network can be split | RFC 950 |
| 1989 | Class D multicast standard published | RFC 1112 |
| 1993 | CIDR replaces classful allocation and routing | RFC 1519 |
| 1996 | Private address ranges published | RFC 1918 |
| 2006 | Current CIDR document replaces RFC 1519 | RFC 4632 |
| 2011 | IANA gives out its last free IPv4 blocks | None |

CIDR slowed the shortage but did not end it. [NAT](09-nat-and-pat.md) and [IPv6](11-ipv6-basics.md) deal with the rest.

### Where Classes Still Appear

Classes are dead for allocation and Internet routing. The ideas still appear in many places:

- **Daily speech:** engineers say "a class C" when they mean a /24, even for a /24 inside 10.0.0.0.
- **Private ranges:** the RFC 1918 ranges are often described in class terms. 10.0.0.0/8 is one class A network, 172.16.0.0/12 is 16 class B networks, and 192.168.0.0/16 is 256 class C networks.
- **Windows:** in the IPv4 settings dialog, Windows fills in the default mask for the address you type. Always check it.
- **Linux:** the old `ifconfig eth0 10.1.1.1` command picks 255.0.0.0 from the class. The modern `ip addr add 10.1.1.1 dev eth0` uses /32 instead, so always give a prefix.
- **Cisco IOS:** routing tables and some `network` commands still use classful boundaries, as shown under Going Deeper.
- **Exams:** certification and entry-level exams still test the table.

## Example

### Example 1: Analyse an Address

**Task:** find everything classful about 172.20.5.9.

1. First octet: 172.
2. In binary: 172 = 128 + 32 + 8 + 4 = 10101100.
3. It starts with 10, so it is class B. The range check agrees: 128-191.
4. Default mask: 255.255.0.0 (/16).
5. Network part: 172.20. Host part: 5.9.
6. Network address (all host bits 0): 172.20.0.0.
7. Broadcast address (all host bits 1): 172.20.255.255.
8. First and last usable host: 172.20.0.1 and 172.20.255.254.
9. Usable hosts: 2^16 - 2 = 65,534.

You can check this with `ipcalc`.

```bash
# Debian/Ubuntu ipcalc package (Red Hat systems ship a different tool)
$ ipcalc 172.20.5.9/16
Address:   172.20.5.9           10101100.00010100. 00000101.00001001
Netmask:   255.255.0.0 = 16     11111111.11111111. 00000000.00000000
Wildcard:  0.0.255.255          00000000.00000000. 11111111.11111111
=>
Network:   172.20.0.0/16        10101100.00010100. 00000000.00000000
HostMin:   172.20.0.1           10101100.00010100. 00000000.00000001
HostMax:   172.20.255.254       10101100.00010100. 11111111.11111110
Broadcast: 172.20.255.255       10101100.00010100. 11111111.11111111
Hosts/Net: 65534                 Class B, Private Internet
```

The last line shows that class and private are separate facts. This address is class B and also private.

### Example 2: Class Does Not Set the Real Mask

This Linux host uses a class A address with a /24 mask:

```bash
# Linux
$ ip -4 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.20.30.40/24 brd 10.20.30.255 scope global eth0
       valid_lft forever preferred_lft forever
```

This Windows host has the same setup:

```powershell
# Windows
PS> ipconfig

Ethernet adapter Ethernet:

   IPv4 Address. . . . . . . . . . . : 10.20.30.40
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.20.30.1
```

The first octet is 10, so the address is class A by range. But the configured mask is /24. So the real network is 10.20.30.0, the broadcast is 10.20.30.255, and it has 254 usable hosts. The configured mask always wins.

### Example 3: Loopback Stays Local

Any address in 127.0.0.0/8 answers locally, not only 127.0.0.1.

```bash
# Linux
$ ping -c 2 127.55.66.77
PING 127.55.66.77 (127.55.66.77) 56(84) bytes of data.
64 bytes from 127.55.66.77: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 127.55.66.77: icmp_seq=2 ttl=64 time=0.052 ms
```

```powershell
# Windows
PS> ping -n 2 127.55.66.77

Pinging 127.55.66.77 with 32 bytes of data:
Reply from 127.55.66.77: bytes=32 time<1ms TTL=128
Reply from 127.55.66.77: bytes=32 time<1ms TTL=128
```

### Example 4: One Company, Classful and Classless

A company needs about 2,000 addresses. The example uses private addresses as safe sample values.

**Classful options:**

- One class B: 65,534 hosts, so about 63,500 addresses stay unused.
- Eight class C networks, 192.168.8.0 to 192.168.15.0: 2,032 hosts, but other routers need eight routes.

**Classless option:** one block, 192.168.8.0/21. It holds 2,046 usable hosts and needs one route. CIDR fixes both the waste and the routing table problem.

### Practice: Name the Class

Cover the right side and test yourself on these first octets.

| First octet | Answer | Reason |
|---|---|---|
| 126 | A | Last usable class A value |
| 127 | Loopback | In class A by bits, reserved |
| 128 | B | First class B value |
| 169 | B | 169.254.0.0/16 inside it is link-local |
| 223 | C | Last class C value |
| 224 | D | First multicast value |
| 240 | E | First reserved value |

## Going Deeper

This part is optional at Beginner level, but it answers common follow-up questions.

**Classful routing protocols.** A routing protocol lets routers share networks. RIP version 1 sends no mask in its updates, so the receiver must guess the mask, using the class and its own interface masks. This is called **classful routing**. RIPv2, OSPF, EIGRP, IS-IS and BGP carry the mask, so they are **classless**. See [Dynamic Routing Overview](../05-routing/04-dynamic-routing-overview.md).

**Automatic summarisation.** A router may announce only the class boundary, such as 10.0.0.0/8, at the edge between two major networks. This breaks a design where parts of 10.0.0.0/8 are separated by a different network. Such a design is called **discontiguous**. In Cisco IOS, EIGRP auto-summary has been off by default since 15.0(1)M, and it was on in older releases. RIP behaviour also depends on version and configuration, so check the running config.

**Classful `network` statements in Cisco IOS.** The `network` command under `router rip` takes only a classful network, so IOS rewrites what you type.

```text
! Cisco IOS
R1(config)# router rip
R1(config-router)# version 2
R1(config-router)# network 172.16.10.0
R1(config-router)# end
R1# show running-config | section router rip
router rip
 version 2
 network 172.16.0.0
```

**Classful headings in the routing table.** Modern IOS still groups routes under their classful network.

```text
! Cisco IOS (legend omitted)
R1# show ip route connected
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.1.0/24 is directly connected, GigabitEthernet0/0
L        10.1.1.1/32 is directly connected, GigabitEthernet0/0
```

The /8 heading is display grouping. The router still forwards using the longest matching prefix, explained in [The Routing Table](../05-routing/02-routing-table.md).

## Common Mistakes

- **Thinking the class sets the real mask.** Only the configured mask does. 10.20.30.40/24 is a normal setup.
- **Saying 192.168.1.0/24 is class C because of the /24.** It is class C because its first octet is 192. 10.1.1.0/24 has the same mask and is still class A by range.
- **Mixing up class and private.** Class comes from the first bits. Private comes from RFC 1918. Every private range is inside a class, and most addresses in each class are not private.
- **Saying class A is 1-127.** Value 127 is loopback. The bit range is 0-127, and the usable range is 1-126.
- **Thinking only 127.0.0.1 is loopback.** The whole 127.0.0.0/8 block is reserved for it.
- **Forgetting to subtract 2 from the host count.** A class C has 256 addresses but 254 usable hosts.
- **Applying a default mask to classes D and E.** They have no default mask.
- **Saying classes are in use on the Internet today.** Allocation and Internet routing have been classless since CIDR. Classes survive as words, habits and legacy tool behaviour.
- **Taking "class C" literally in conversation.** It usually just means a /24.

## Interview Questions

**Q: What are the IPv4 classes and their first octet ranges?**

A: Class A is 1-126, class B is 128-191, class C is 192-223, class D is 224-239 for multicast, and class E is 240-255, which is reserved. 127 is kept for loopback.

**Q: Why is 127 not included in class A?**

A: By its bits, it is class A, but 127.0.0.0/8 is reserved for loopback. Traffic to those addresses stays inside the device, so the block cannot be used as a normal network.

**Q: How many networks and hosts does each class have?**

A: Class A has 126 usable networks with 16,777,214 hosts each. Class B has 16,384 networks with 65,534 hosts each. Class C has 2,097,152 networks with 254 hosts each.

**Q: How do the leading bits relate to the ranges?**

A: Class A starts with 0, B with 10, C with 110, D with 1110 and E with 1111. Filling the remaining first-octet bits with all 0s and all 1s gives each range. For example, 110 gives 192 to 223.

**Q: Why do we subtract 2 when counting hosts?**

A: The address with all host bits 0 is the network address, and the one with all host bits 1 is the broadcast address. Neither is given to a normal host.

**Q: Why were classes retired?**

A: The fixed sizes wasted addresses, because there was nothing between 254 and 65,534 hosts. Class B was running low, and many small allocations were making routing tables too big. CIDR replaced them in 1993 with flexible prefix lengths.

**Q: Can a class A address have a /24 mask?**

A: Yes. Class only describes the first octet range. The real network is decided by the configured mask, so 10.20.30.40/24 is valid and sits in the network 10.20.30.0.

**Q: What is the difference between class and private?**

A: They answer different questions. Class is based on the first bits of the address. Private means the address is in an RFC 1918 range. For example, 172.16.0.1 is class B and private.

**Q: What is the difference between classful and classless routing?**

A: A classful protocol, like RIPv1, does not send masks in updates, so networks are assumed to follow class boundaries. A classless protocol, like OSPF or BGP, sends the prefix length with each route, so any size of network can be used.

**Q: If classes are obsolete, why do people still say "class C"?**

A: It is a short name for a /24, and old terms stay in use. They also still appear in tools, like Windows filling in a default mask, and in Cisco IOS routing output.

## Quick Recap

- Classes A, B and C were unicast network sizes of /8, /16 and /24. D is multicast, and E is reserved.
- Leading bits 0, 10, 110, 1110 and 1111 create the ranges 0-127, 128-191, 192-223, 224-239 and 240-255.
- Usable hosts are 16,777,214, 65,534 and 254. The count of networks is 126 usable, 16,384 and 2,097,152.
- 127.0.0.0/8 is loopback, and 0.0.0.0/8 is also reserved, so class A has 126 usable networks.
- Classes were retired because of wasted addresses, class B shortage and routing table growth.
- CIDR replaced them in 1993, and the configured mask, not the class, defines a network today.
- Interviewers ask because this tests binary skills and explains why CIDR exists.
