# IPv6 Basics

**Level:** Intermediate | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [IPv4 Addressing Basics](01-ipv4-basics.md), [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md)

## In One Line

IPv6 is the newer version of the Internet Protocol. It uses 128-bit addresses instead of IPv4's 32-bit addresses, so the world does not run out of addresses, and it replaces broadcast with multicast.

## Why It Matters

IPv4 has 32 bits, which gives about 4.3 billion addresses. That is less than one address per person on Earth, and each person now has several devices.

The central pool of unused IPv4 blocks, run by IANA (the Internet Assigned Numbers Authority), ran out in February 2011. The regional registries that hand out addresses ran out in the years after that.

The Internet survived by using NAT (Network Address Translation), which lets many devices share one public address. NAT works, but it has a cost:

- Devices cannot easily reach each other directly.
- Carriers must run large and expensive shared NAT systems.
- Logs and troubleshooting become harder, because many users share one address.

Read more in [NAT and PAT](09-nat-and-pat.md).

IPv6 fixes the root problem by making the address much bigger. Today, Google's public statistics show that around half of its users arrive over IPv6. The number changes by day and by country. Some large mobile carriers run IPv6-only networks.

Interviewers ask about IPv6 because:

- Reading and shortening addresses is a quick test of real knowledge.
- Cloud, mobile and carrier networks use it every day.
- Questions like "why is there no broadcast?" show whether you understand how a LAN really works.

## Simple Explanation

Think of IP addresses as phone numbers.

A city started with 7-digit phone numbers. The city grew and ran out of numbers. People began sharing one number through a receptionist who passes calls along. That receptionist is NAT.

IPv6 is a new numbering plan with far longer numbers. Everyone can have their own number again, and no receptionist is needed.

An IPv6 address also has two parts, like a postal address:

- The **prefix** is the street. It tells the network where to deliver.
- The **interface ID** is the house number. It says which device on that street.

**Where the analogy stops being true:** old and new phone numbers can call each other through the same system. IPv4 and IPv6 cannot. They are different protocols, and a device that has only IPv6 cannot talk directly to a device that has only IPv4. It needs a translator or both protocols running side by side. See [IPv6 in Practice](12-ipv6-in-practice.md).

## How It Works

### 128 Bits: How Big Is That?

| Version | Bits | Total addresses |
|---|---|---|
| IPv4 | 32 | 2^32 = 4,294,967,296 (about 4.3 billion) |
| IPv6 | 128 | 2^128 = about 3.4 x 10^38 |

Each extra bit doubles the space. IPv6 does not have 4 times as many addresses as IPv4. It has 2^96 times as many.

This size changes how we plan networks. In IPv4 we count every address and save them. In IPv6 we do not worry about saving addresses inside a subnet. One normal LAN subnet has 2^64 addresses, which is about 18 quintillion. We plan by counting **subnets** instead.

### Hex Notation

Writing 128 bits in dotted decimal would be painful. IPv6 uses **hexadecimal** (base 16: the digits 0-9 and a-f). One hex digit holds exactly 4 bits, so 128 bits become 32 hex digits.

The digits are split into 8 groups of 4, with colons between them. Each group is 16 bits. People often call a group a **hextet**. That word is common but informal.

```text
2001:0db8:acad:0001:0000:0000:0000:0001
\__/ \__/ \__/ \__/ \__/ \__/ \__/ \__/
 16   16   16   16   16   16   16   16    = 128 bits
```

Letters can be upper or lower case, and both mean the same thing. `2001:DB8::1` and `2001:db8::1` are the same address. Cisco IOS prints upper case. Linux and Windows print lower case. RFC 5952, the standard for writing addresses, says software should print lower case.

The block `2001:db8::/32` is reserved for books and examples. It is never used on the real Internet, and this course uses it for that reason.

### The Two Shortening Rules

32 hex digits are hard to read, so two rules make addresses shorter.

**Rule 1: Drop leading zeros in any group.**

Only zeros at the **start** of a group can go.

```text
0db8 -> db8
0001 -> 1
0000 -> 0
0120 -> 120     (the last 0 stays)
1000 -> 1000    (nothing to drop)
```

**Rule 2: Replace one run of all-zero groups with `::`.**

The double colon means "fill in as many all-zero groups as needed to reach 8 groups". You can use it **only once** in an address.

Applying both rules step by step:

```text
Full:     2001:0db8:0000:0000:0000:0000:0000:0001
Rule 1:   2001:db8:0:0:0:0:0:1
Rule 2:   2001:db8::1
```

More examples:

| Full form | Shortest form |
|---|---|
| `2001:0db8:0000:0000:0000:0000:0000:0001` | `2001:db8::1` |
| `2001:0db8:0001:0000:0000:0000:0000:0000` | `2001:db8:1::` |
| `2001:0db8:0000:0001:0000:0000:0000:0001` | `2001:db8:0:1::1` |
| `fe80:0000:0000:0000:0000:0000:0000:0001` | `fe80::1` |
| `ff02:0000:0000:0000:0000:0001:ff12:3456` | `ff02::1:ff12:3456` |
| `0000:0000:0000:0000:0000:0000:0000:0001` | `::1` |
| `0000:0000:0000:0000:0000:0000:0000:0000` | `::` |

### Shortening Traps

**Trap 1: Trailing zeros stay.** `0120` becomes `120`, never `12`. Dropping a zero at the end changes the value. It is like turning 120 into 12.

**Trap 2: `::` can appear only once.** Look at `2001::1::1`. It has 3 written groups, so 5 zero groups are missing, but nobody can tell how they split between the two `::`. It could be any of these:

```text
2001:0:1:0:0:0:0:1      (1 zero, then 4 zeros)
2001:0:0:1:0:0:0:1      (2 zeros, then 3 zeros)
2001:0:0:0:1:0:0:1      (3 zeros, then 2 zeros)
2001:0:0:0:0:1:0:1      (4 zeros, then 1 zero)
```

Because it is ambiguous, `2001::1::1` is **invalid**.

**Trap 3: Shorten the longest run.** The address `2001:0:0:1:0:0:0:1` has two zero runs. The second one is longer, so it gets the `::`.

```text
Correct:         2001:0:0:1::1
Valid but poor:  2001::1:0:0:0:1
```

**Trap 4: On a tie, shorten the first run.**

```text
Address:   2001:db8:0:0:1:0:0:1
Correct:   2001:db8::1:0:0:1
```

**Trap 5: Do not use `::` for one zero group.** Write a single `0` instead.

```text
Address:   2001:db8:0:1:1:1:1:1
Correct:   2001:db8:0:1:1:1:1:1
Avoid:     2001:db8::1:1:1:1:1
```

**Fuller truth on traps 3, 4 and 5:** every form above can be parsed by software, because the addressing standard (RFC 4291) allows them. RFC 5952 sets the one **recommended** way to print an address, so that the same address always looks the same in logs and searches. Trap 2 is different: that form is simply invalid.

**Trap 6: Bad forms that are always invalid.**

| Bad address | Why it is wrong |
|---|---|
| `2001:db8:::1` | Three colons in a row |
| `2001:db8:12345::1` | A group has more than 4 digits |
| `2001:db8:1:2:3:4:5` | Only 7 groups and no `::` |
| `2001:db8::g1` | `g` is not a hex digit |

### Expanding an Address

To go from short to full form:

1. Count the groups you can see.
2. Subtract from 8. That is how many zero groups `::` hides.
3. Write those zero groups where `::` was.
4. Pad every group to 4 digits with leading zeros.

Example with `2001:db8:a::f`:

```text
Written groups: 2001, db8, a, f  = 4 groups
Missing:        8 - 4            = 4 groups
Fill in:        2001:db8:a:0:0:0:0:f
Pad:            2001:0db8:000a:0000:0000:0000:0000:000f
```

### Prefix and Interface ID

Like IPv4 CIDR, IPv6 uses a slash and a **prefix length**. The prefix length says how many bits, counted from the left, are the network part. IPv6 has no dotted subnet mask.

`2001:db8:acad:1::10/64` means the first 64 bits are the prefix and the last 64 bits are the **interface ID** (the part that names one interface on that subnet).

A typical global address has three parts:

```text
|<----- 48 bits ----->|<- 16 ->|<-------- 64 bits -------->|
+---------------------+--------+---------------------------+
| global routing      | subnet |       interface ID        |
| prefix              | ID     |                           |
+---------------------+--------+---------------------------+
  2001:0db8:acad        0001     0000:0000:0000:0010
```

- **Global routing prefix:** the block given to your site by an ISP or registry.
- **Subnet ID:** the part you choose, to number your own subnets.
- **Interface ID:** names the device interface.

Common prefix lengths:

| Length | Typical use | Contains |
|---|---|---|
| /32 | Common minimum block for an ISP | 65,536 /48s |
| /48 | A business site | 65,536 /64s |
| /56 | A home or small site | 256 /64s |
| /60 | A home, on some ISPs | 16 /64s |
| /64 | One LAN subnet | 2^64 addresses |
| /128 | One single address | 1 address |

What an ISP gives a customer varies by ISP. The table shows common choices, not fixed rules.

Each hex digit is 4 bits, so a prefix that is a multiple of 4 is easy to read:

```text
/48 = first 12 hex digits  -> 2001:db8:acad::/48
/52 = first 13 hex digits
/56 = first 14 hex digits  -> 2001:db8:acad:1200::/56
/64 = first 16 hex digits  -> 2001:db8:acad:1234::/64
```

Always read a group in its full 4-digit form. The prefix `2001:db8:acad:1200::/56` covers subnets `1200` to `12ff`. But `2001:db8:acad:12::` has `0012` in group 4, so its /56 prefix is `2001:db8:acad::/56`, and the `12` falls in the bits after the prefix.

Why is /64 the normal LAN size? The main automatic address method, SLAAC (Stateless Address Autoconfiguration, where a host builds its own address), expects a 64-bit interface ID. If you make a LAN something else, such as /80, SLAAC stops working there. The rule comes from the standards and the software built on them, not from routers being unable to forward other lengths.

Hosts get the interface ID in one of a few ways: random generation, a method based on the MAC address called EUI-64, DHCPv6, or manual setup. Those methods are explained in [IPv6 in Practice](12-ipv6-in-practice.md).

### Address Types at a Glance

IPv6 has three delivery types:

- **Unicast:** one sender to one receiver.
- **Multicast:** one sender to a group of receivers who joined.
- **Anycast:** one sender to the nearest of several devices that share an address.

There is no broadcast.

The first few hex digits tell you the type:

| Type | Prefix | Starts with | Reach | Closest IPv4 idea |
|---|---|---|---|---|
| Global unicast | `2000::/3` | `2` or `3` | Whole Internet | Public address |
| Link-local | `fe80::/10` | `fe8`, `fe9`, `fea`, `feb` | One link only | `169.254.0.0/16` |
| Unique local | `fc00::/7` | `fd` in practice | Inside your network | `10.0.0.0/8` and other private ranges |
| Multicast | `ff00::/8` | `ff` | Depends on scope | `224.0.0.0/4` |
| Anycast | None of its own | Looks like unicast | Nearest device | Anycast |
| Loopback | `::1/128` | `::1` | This device | `127.0.0.1` |
| Unspecified | `::/128` | `::` | Not a destination | `0.0.0.0` |

### Global Unicast: 2000::/3

A global unicast address is public and works across the Internet. It is like a public IPv4 address, but you usually do not need NAT in front of it.

The `/3` means the first 3 bits are `001`, so the first hex digit is `2` or `3`:

```text
0010 = 2
0011 = 3
```

`2000::/3` is the part that IANA currently hands out. The standard leaves other space for possible future use.

**Warning:** having a public address does not mean a device is open to the world. A firewall should still block unwanted incoming traffic. See [Firewalls](../09-security/04-firewalls.md).

### Link-Local: fe80::/10

A link-local address works only on one **link**, meaning one Ethernet segment, VLAN or Wi-Fi network. Routers never forward these packets to another link.

Every IPv6 interface has one. It is created automatically as soon as IPv6 is on, even with no router and no DHCP.

Analogy: a link-local address is like a house intercom. It works at once, with no help from outside, but it only reaches people in the same house. **Where the analogy stops being true:** an intercom is one shared system, while each link-local address belongs to one interface, and one device can sit on several links.

```text
|  10 bits   |      54 bits      |       64 bits       |
+------------+-------------------+---------------------+
| 1111111010 |    all zeros      |    interface ID     |
+------------+-------------------+---------------------+
```

The `/10` allows first groups from `fe80` to `febf`, but the standard format has zeros after the first 10 bits. So in real life you nearly always see `fe80::`.

Link-local addresses do real work:

- Neighbors find each other with them.
- A host's default gateway is usually the router's link-local address.
- Routing protocols such as OSPFv3 use them to talk between neighbors.

**Zone ID.** A device can have the same `fe80::/64` on many interfaces, so a link-local address alone does not say which interface to use. You add a **zone ID** after a `%` sign:

```text
Linux:    fe80::1%eth0     (interface name)
Windows:  fe80::1%12       (interface index number)
```

**IPv4 difference:** `169.254.x.x` appears only as a fallback when DHCP fails, and seeing it usually means trouble. `fe80::` is always present and is a normal sign of health.

### Unique Local: fc00::/7

A unique local address, or ULA, is for use inside an organization. Internet routers should not carry it. It is the nearest thing to private IPv4 ranges such as `10.0.0.0/8`.

```text
| 7 bits  | 1 |  40 bits  | 16 bits |   64 bits    |
+---------+---+-----------+---------+--------------+
| 1111110 | L | global ID | subnet  | interface ID |
+---------+---+-----------+---------+--------------+
```

The L bit decides the first two hex digits:

- L = 1 gives `fd00::/8`. This is the half in use today. You pick the 40-bit global ID at random.
- L = 0 gives `fc00::/8`. It has no defined way of being assigned, so you do not use it.

Example: `fd12:3456:789a:1::10`

| Part | Value |
|---|---|
| Fixed start and L bit | `fd` |
| Random global ID | `12:3456:789a` |
| Subnet ID | `0001` |
| Interface ID | `0000:0000:0000:0010` |

**Why random?** If two companies both use `10.0.0.0/8` and later merge, their addresses clash. Two random 40-bit IDs almost never match, so merged ULA networks usually fit together with no renumbering.

**Trap:** a device with a ULA can also have a global address at the same time. ULA is not a replacement for a global address if the device needs the Internet.

### Multicast: ff00::/8

A multicast address names a group, not one device. Devices join a group, and a packet sent to the group reaches its members. Multicast addresses are destinations only. A packet never uses one as its source.

Common groups on a link:

| Address | Who listens |
|---|---|
| `ff02::1` | All nodes on the link |
| `ff02::2` | All routers on the link |
| `ff02::5` | OSPFv3 routers |
| `ff02::6` | OSPFv3 designated routers |
| `ff02::9` | RIPng routers |
| `ff02::a` | EIGRP routers |
| `ff02::fb` | mDNS (local name lookup) |
| `ff02::1:2` | DHCPv6 relay agents and servers |
| `ff02::1:ffXX:XXXX` | Solicited-node group (see below) |

The second byte holds the **scope**, which says how far the packet may travel. The `2` in `ff02` means link-local. Full details are in Going Deeper.

### Anycast

An anycast address is one address given to several devices in different places. The network delivers each packet to the nearest one, based on routing.

Analogy: a national emergency number. Everyone dials the same number, and the call goes to a nearby center. **Where the analogy stops being true:** "nearest" here means best by routing, not closest on a map.

Anycast has no prefix of its own. It comes from normal unicast space, so you cannot tell it is anycast by looking at it. The address is configured and announced so that routing does the rest.

Real use: public DNS services and root DNS servers put the same address in many cities, so users reach a nearby copy. See [Unicast, Broadcast, Multicast and Anycast](13-unicast-broadcast-multicast-anycast.md).

### Loopback: ::1/128

`::1` means "this device". Packets sent to it never leave the machine. It is the same idea as `127.0.0.1`.

**Difference:** IPv4 reserves the whole `127.0.0.0/8` block. IPv6 reserves exactly one address.

### Unspecified: ::/128

`::` means "no address". It is all 128 bits set to zero.

It can appear as a **source** address when a device has no address yet. The clearest case is the check a host does to confirm that a new address is not already in use.

It must never be a destination and must never be assigned to an interface.

**Do not confuse it with these:**

- `::/0` is the **default route**, meaning all IPv6 destinations, like `0.0.0.0/0`.
- A server that listens on `[::]:443` is listening on all its IPv6 addresses.

### No Broadcast

IPv6 has no broadcast address. The idea is gone from the protocol.

**The problem with broadcast:** when an IPv4 host sends an ARP request, every device on the VLAN must receive it and pass it up for checking, even though only one device cares. On a big VLAN that wastes CPU, and battery on phones. See [ARP](../03-ethernet-and-switching/05-arp.md) and [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).

**The IPv6 answer:** send to a multicast group that contains only the devices that might care.

Analogy: broadcast is an announcement over the speakers in the whole building. Multicast is a message to one small group chat. Only people in that chat are disturbed.

**Where the analogy stops being true:** a group chat has known members. In IPv6, a device joins a group by itself, and the group can include devices that do not need the message. The solicited-node group below can have more than one member, though usually very few.

The job of ARP moves to **NDP** (Neighbor Discovery Protocol), which runs over ICMPv6:

```text
IPv4: ARP uses broadcast
  Host A: "Who has 192.168.1.20?"
     |
     +--> dest MAC ff:ff:ff:ff:ff:ff
          Every device on the VLAN reads this.

IPv6: NDP uses solicited-node multicast
  Host A: "Who has 2001:db8:acad:10::20?"
     |
     +--> dest IP  ff02::1:ff00:20
          dest MAC 33:33:ff:00:00:20
          Only devices in that group read this.
```

**Fuller truth:** a switch does not always know who joined a group. Without **MLD snooping** (a switch feature that watches group joins), it floods the multicast frame out of all ports in the VLAN, like a broadcast. Network cards can still filter it in hardware, so most hosts are not disturbed, but the bandwidth is still used.

`ff02::1` (all nodes) is close to a broadcast, but it is still multicast. Interviewers like this point. See [IPv6 in Practice](12-ipv6-in-practice.md) for full NDP steps.

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---|---|---|
| Address size | 32 bits | 128 bits |
| Format | Dotted decimal: `192.0.2.1` | Hex with colons: `2001:db8::1` |
| Total addresses | About 4.3 billion | About 3.4 x 10^38 |
| Network length | Mask or /prefix | /prefix only |
| Base header size | 20-60 bytes, varies | 40 bytes, fixed |
| Header checksum | Yes | No |
| Options | Inside the header | Extension headers |
| Who fragments | Sender and routers | Sender only |
| Minimum link MTU | 68 bytes | 1280 bytes |
| Lifetime field | TTL | Hop Limit (same idea) |
| Broadcast | Yes | No |
| Multicast | Optional in practice | Required, used for basic functions |
| Find MAC address | ARP | NDP (ICMPv6) |
| Auto setup | DHCP | SLAAC, DHCPv6, or both |
| Link-local | `169.254.0.0/16`, fallback | `fe80::/10`, always present |
| Private range | RFC 1918 ranges | ULA `fc00::/7` |
| Loopback | `127.0.0.0/8` | `::1/128` only |
| Default route | `0.0.0.0/0` | `::/0` |
| Addresses per interface | Usually one | Usually several |
| NAT | Very common | Rarely needed |
| DNS record | A | AAAA |
| Reverse DNS zone | `in-addr.arpa` | `ip6.arpa` |
| ICMP | ICMP, for errors and diagnostics | ICMPv6, which also carries neighbor discovery |

Also see [DNS Record Types](../07-application-protocols/02-dns-records.md) and [IPv4 Header and Fragmentation](10-ipv4-header-and-fragmentation.md).

### IPv6 in URLs

Colons in an address clash with the colon before a port number. So URLs put IPv6 addresses in square brackets:

```text
http://[2001:db8::10]:8080/index.html
```

## Example

This example follows one LAN from address to live commands.

The site owns `2001:db8:acad::/48`. A host on VLAN 10 has this address:

```text
2001:db8:acad:10:20c:29ff:fe3a:4b5c/64
```

### Step 1: Expand It

The address has 8 written groups, so nothing is hidden. Pad each group:

```text
2001:0db8:acad:0010:020c:29ff:fe3a:4b5c
```

### Step 2: Split It

```text
2001:0db8:acad | 0010   | 020c:29ff:fe3a:4b5c
  site /48     | subnet |   interface ID
```

### Step 3: Find the Subnet Details

```text
Prefix:         2001:db8:acad:10::/64
First address:  2001:db8:acad:10::
Last address:   2001:db8:acad:10:ffff:ffff:ffff:ffff
Size:           2^64 addresses
```

### Step 4: Identify the Type

The first digit is `2`, so it is global unicast. The block `2001:db8::/32` makes this a documentation address, but in form it matches a real global address.

The `ff:fe` in the middle of the interface ID is a clue that this ID was built from a MAC address with the EUI-64 method. Here the MAC is `00:0c:29:3a:4b:5c`.

### Step 5: Plan the Site

The /48 leaves 16 bits for subnets, which gives 65,536 /64 subnets. A neat trick is to match the subnet ID to the VLAN number so that people can read it easily:

| VLAN | Use | Prefix |
|---|---|---|
| 10 | Staff | `2001:db8:acad:10::/64` |
| 20 | Voice | `2001:db8:acad:20::/64` |
| 30 | Servers | `2001:db8:acad:30::/64` |

These are hex values, so subnet `10` is really 16 in decimal. That is fine, because the number is only a label. It just has to be unique.

### Step 6: Configure the Router (Cisco IOS)

```text
Router(config)# ipv6 unicast-routing
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:db8:acad:10::1/64
Router(config-if)# ipv6 address fe80::1 link-local
Router(config-if)# no shutdown
```

`ipv6 unicast-routing` turns on IPv6 forwarding. Without it, the device can have IPv6 addresses but does not route IPv6 packets. It also does not send the Router Advertisements that hosts need for SLAAC.

Setting `fe80::1` by hand is optional. It makes the gateway easy to read.

```text
Router# show ipv6 interface brief
GigabitEthernet0/0     [up/up]
    FE80::1
    2001:DB8:ACAD:10::1

Router# show ipv6 interface GigabitEthernet0/0
GigabitEthernet0/0 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::1
  Global unicast address(es):
    2001:DB8:ACAD:10::1, subnet is 2001:DB8:ACAD:10::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF00:1
  MTU is 1500 bytes
(output trimmed)
```

Look at the joined groups: all nodes, all routers, and one solicited-node group. Both of the router's addresses end in `00:0001`, so they share one group.

### Step 7: Check a Linux Host

The interface may have a different name, such as `ens33`.

```bash
# Linux: show IPv6 addresses
$ ip -6 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2001:db8:acad:10:20c:29ff:fe3a:4b5c/64 scope global dynamic
       valid_lft 86394sec preferred_lft 14394sec
    inet6 fe80::20c:29ff:fe3a:4b5c/64 scope link
       valid_lft forever preferred_lft forever

# Linux: show the IPv6 routing table
$ ip -6 route
::1 dev lo proto kernel metric 256 pref medium
2001:db8:acad:10::/64 dev eth0 proto ra metric 100 pref medium
fe80::/64 dev eth0 proto kernel metric 1024 pref medium
default via fe80::1 dev eth0 proto ra metric 100 pref medium
```

The host has two addresses on one interface. That is normal. Also note that the default gateway is the router's link-local address, not its global address.

### Step 8: Check a Windows Host

This host is on the same subnet. Windows uses random interface IDs by default, so its address has no `ff:fe` in the middle.

```powershell
# Windows: show addresses
PS> ipconfig

Ethernet adapter Ethernet:

   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:10:a4f1:3c7e:90b2:1d55
   Temporary IPv6 Address. . . . . . : 2001:db8:acad:10:6e2d:81c4:f07a:33b9
   Link-local IPv6 Address . . . . . : fe80::a4f1:3c7e:90b2:1d55%12
   IPv4 Address. . . . . . . . . . . : 192.168.10.50
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : fe80::1%12
                                       192.168.10.1
```

The **temporary address** has a random interface ID that changes over time. The host uses it for outgoing connections, so websites cannot easily track it. The `%12` is the zone ID.

### Step 9: Test It

```bash
# Linux: test the local IPv6 stack
$ ping -c 2 ::1
PING ::1 (::1) 56 data bytes
64 bytes from ::1: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from ::1: icmp_seq=2 ttl=64 time=0.045 ms

--- ::1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1024ms
rtt min/avg/max/mdev = 0.031/0.038/0.045/0.007 ms

# Linux: ping the gateway. Link-local needs the zone ID.
$ ping -c 1 fe80::1%eth0
PING fe80::1%eth0 (fe80::1%eth0) 56 data bytes
64 bytes from fe80::1%eth0: icmp_seq=1 ttl=64 time=0.612 ms

# Linux: ping all nodes on the link. Replies show your neighbors.
$ ping -c 2 ff02::1%eth0
PING ff02::1%eth0 (ff02::1%eth0) 56 data bytes
64 bytes from fe80::20c:29ff:fe3a:4b5c%eth0: icmp_seq=1 ttl=64 time=0.040 ms
64 bytes from fe80::1%eth0: icmp_seq=1 ttl=64 time=0.640 ms (DUP!)
(output trimmed)
```

```powershell
# Windows: test the local IPv6 stack and the gateway
PS> ping -6 ::1

Pinging ::1 with 32 bytes of data:
Reply from ::1: time<1ms

PS> ping fe80::1%12

Pinging fe80::1%12 with 32 bytes of data:
Reply from fe80::1%12: time=1ms
(output trimmed)
```

Some hosts do not answer the all-nodes ping. Windows Firewall is a common reason. No reply does not always mean no neighbor.

## Going Deeper

### The IPv6 Header

The base header is always 40 bytes. It is only twice the size of a minimal IPv4 header, even though addresses are four times longer.

```text
 0       4              12                                     31
+-------+---------------+---------------------------------------+
|Version| Traffic Class |              Flow Label               |
|   4   |       8       |                  20                   |
+-------+---------------+-------+---------------+---------------+
|       Payload Length          |  Next Header  |   Hop Limit   |
|              16               |       8       |       8       |
+-------------------------------+---------------+---------------+
|                                                               |
|                    Source Address (128 bits)                  |
|                                                               |
+---------------------------------------------------------------+
|                                                               |
|                 Destination Address (128 bits)                |
|                                                               |
+---------------------------------------------------------------+
```

| Field | Purpose | IPv4 equivalent |
|---|---|---|
| Version | Always 6 | Version |
| Traffic Class | Priority marking for QoS | ToS / DSCP |
| Flow Label | Marks packets of one flow, often used for load balancing | None |
| Payload Length | Bytes after the base header | Total Length (which includes the header) |
| Next Header | What comes next: TCP is 6, UDP is 17, ICMPv6 is 58 | Protocol |
| Hop Limit | Reduced by 1 at each router; packet dropped at 0 | TTL |

Fields removed from the base header:

- **Header Length:** not needed, since the size is fixed.
- **Header Checksum:** lower and upper layers already check for errors, and routers no longer need to recalculate it at each hop. Because of this, the UDP checksum is mandatory over IPv6, with a narrow exception for some tunnels.
- **Identification, Flags, Fragment Offset:** moved to a Fragment extension header.
- **Options:** moved to extension headers.

The full standard is RFC 8200.

### Extension Headers

Optional information sits in separate headers between the base header and the data. Each one points to the next through a Next Header field, forming a chain.

```text
+-----------+------------+----------------+
| Base      | Fragment   | TCP segment    |
| Next=44   | Next=6     |                |
+-----------+------------+----------------+
```

| Value | Extension header |
|---|---|
| 0 | Hop-by-Hop Options |
| 43 | Routing |
| 44 | Fragment |
| 50 | ESP (IPsec encryption) |
| 51 | AH (IPsec authentication) |
| 60 | Destination Options |

Routers do not need to read most of these, so forwarding stays simple. **Real-world catch:** some networks and firewalls drop packets with unusual extension headers, so they do not always work across the Internet.

### Fragmentation

Routers never fragment IPv6 packets. If a packet is too big for the next link, the router drops it and sends an ICMPv6 Packet Too Big message to the sender. The sender then uses smaller packets.

Every link must carry at least 1280 bytes. This is why blocking ICMPv6 breaks connections. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Prefixes That Do Not End on a Hex Digit

A prefix such as /62 splits a hex digit in two. Work in binary on that one digit.

Find the /62 prefix of `2001:db8:acad:1237::1`.

```text
/60 ends after the 3rd digit of group 4 (the "3").
/62 keeps 2 more bits, from the digit 7.

7 in binary:        01 11
Keep 2 bits:        01 00  = 4

Prefix:             2001:db8:acad:1234::/62
Covers group 4:     1234, 1235, 1236, 1237
```

### Multicast Address Format

```text
|  8 bits  | 4 bits | 4 bits |            112 bits             |
+----------+--------+--------+---------------------------------+
| 11111111 | flags  | scope  |            group ID             |
+----------+--------+--------+---------------------------------+
```

A flags value of 0 means a permanent, well-known group. Scope values:

| Hex | Scope | Example start |
|---|---|---|
| 1 | Interface-local | `ff01` |
| 2 | Link-local | `ff02` |
| 4 | Admin-local | `ff04` |
| 5 | Site-local | `ff05` |
| 8 | Organization-local | `ff08` |
| e | Global | `ff0e` |

### Solicited-Node Multicast

For each unicast address it owns, a host joins one solicited-node group. The group is built by a fixed recipe:

```text
Fixed part:      ff02::1:ff00:0/104
Add:             the last 24 bits of the unicast address
```

Worked example with `2001:db8:acad:10:20c:29ff:fe3a:4b5c`:

```text
Last 24 bits:          3a:4b5c
Solicited-node IP:     ff02::1:ff3a:4b5c
Ethernet MAC:          33:33:ff:3a:4b:5c
```

The Ethernet multicast MAC is always `33:33` plus the last 32 bits of the IPv6 multicast address. For `ff02::1`, that gives `33:33:00:00:00:01`.

A host looking for a neighbor knows the target IP, so it can compute this group by itself. Only devices with the same last 24 bits listen, and on a normal LAN that is usually one device.

### Other Special Addresses

| Prefix | Use |
|---|---|
| `2001:db8::/32` | Documentation (RFC 3849) |
| `3fff::/20` | Extra documentation space (RFC 9637) |
| `::ffff:0:0/96` | IPv4-mapped addresses |
| `64:ff9b::/96` | NAT64 translation (see [IPv6 in Practice](12-ipv6-in-practice.md)) |
| `fec0::/10` | Old site-local, deprecated (RFC 3879). Do not use. |

An **IPv4-mapped address** is how a dual-stack program shows an IPv4 client on an IPv6 socket. You may see `::ffff:192.0.2.1` in server logs. In pure hex it is `::ffff:c000:201`. These addresses are not sent on the wire as IPv6 packets.

Addresses that carry an IPv4 address in their last 32 bits may write those bits in dotted decimal, as shown here.

### Prefix Lengths Other Than /64

- **/128** for a router loopback, which is one address.
- **/127** for point-to-point links between routers. RFC 6164 recommends it, partly to avoid attacks that use the unused addresses in a larger subnet.
- **/64** on point-to-point links also works and is used by some operators.

Inside a /64, the all-zero interface ID is the **subnet-router anycast** address, which reaches a router on that subnet. It is not a "network address" in the IPv4 sense. IPv6 also has no broadcast address at the top of a subnet.

### Scanning a /64

At one million probes per second, a scan of all 2^64 addresses would take more than 500,000 years. Random scanning of a subnet is not practical.

**Fuller truth:** this gives no real protection. Attackers find addresses through DNS, logs and simple patterns such as `::1`, so you still need firewalls.

### What Happened to IPv5?

Version number 5 went to an experimental streaming protocol called ST, which never became a general replacement for IPv4. The next version of IP therefore got the number 6.

## Common Mistakes

- **Dropping trailing zeros.** `0120` shortens to `120`, not `12`.
- **Using `::` twice.** The reader cannot work out how many zeros each one hides.
- **Treating `::1` and `::` as similar.** One is loopback. The other is "no address".
- **Mixing up `::` and `::/0`.** One is a single address. The other is the default route covering everything.
- **Reading groups as decimal.** `2001:db8:10::` has hex `10`, which is 16 in decimal.
- **Reading a short group without padding.** `:12:` is really `0012`, not `1200`.
- **Expecting a host to have one address.** Link-local, global and temporary addresses on one interface is normal.
- **Thinking `fe80::` means a problem.** That is the IPv4 `169.254.x.x` habit. Link-local addresses are always there.
- **Forgetting the zone ID.** `ping fe80::1` can fail on a host because it does not know which interface to use.
- **Trying to route link-local addresses.** Routers never forward them to another link.
- **Thinking ULA is just RFC 1918.** The purpose is similar, but ULA uses a random global ID and is not designed to need NAT.
- **Using `fc00::/8` for ULA.** Use `fd00::/8`, with a random global ID.
- **Using a LAN prefix other than /64.** This breaks SLAAC on that subnet.
- **Blocking all ICMPv6.** This breaks neighbor discovery and Path MTU Discovery, so IPv6 stops working properly.
- **Believing IPv6 is more secure by itself.** IPsec support is not the same as IPsec in use. In practice, IPv6 traffic is protected in the same ways as IPv4, and hosts still need firewalls.
- **Believing NAT is gone, so devices are exposed.** Exposure depends on the firewall, not on NAT. A stateful firewall blocks unsolicited incoming traffic without NAT.
- **Thinking IPv4 and IPv6 talk directly.** They need dual stack, tunneling or translation.
- **Assuming IPv6 is always faster.** The speed depends on the path, not on the version number.

## Interview Questions

**Q: Why do we need IPv6?**
A: IPv4 has only about 4.3 billion addresses and the free pools ran out. NAT extended its life, but it adds cost and breaks direct connections. IPv6 uses 128 bits, so address shortage stops being a problem. It also removes broadcast and allows hosts to configure themselves.

**Q: How long is an IPv6 address and how is it written?**
A: It is 128 bits, written as 8 groups of 4 hex digits, with colons between them. Each group is 16 bits. An example is `2001:db8:acad:1::10`.

**Q: What are the rules for shortening an IPv6 address?**
A: First, remove leading zeros in any group. Second, replace one run of all-zero groups with a double colon. The double colon can be used only once, or the address would be ambiguous. RFC 5952 also says to shorten the longest run, pick the first on a tie, use lower case, and not use `::` for a single group.

**Q: Shorten `2001:0db8:0000:0000:0001:0000:0000:0100`.**
A: Dropping leading zeros gives `2001:db8:0:0:1:0:0:100`. There are two zero runs of the same length, so I shorten the first one. The answer is `2001:db8::1:0:0:100`.

**Q: What is the difference between the prefix and the interface ID?**
A: The prefix is the network part, and routers use it to forward packets. The interface ID names one interface on that network. On a normal LAN the split is 64 bits each, shown as /64.

**Q: Why is /64 the standard subnet size?**
A: SLAAC and the addressing standards expect a 64-bit interface ID. If a LAN uses another length, hosts cannot use SLAAC there. So even a small LAN gets a /64. We do not try to save addresses inside a subnet.

**Q: What is a link-local address and why is it needed?**
A: It is an address in `fe80::/10` that every IPv6 interface creates by itself. It works only on one link and is never routed. NDP uses it, routing protocols use it between neighbors, and a host's default gateway is usually a link-local address.

**Q: What is the difference between ULA and global unicast?**
A: Global unicast is in `2000::/3` and is routable on the Internet. ULA is in `fc00::/7`, in practice `fd00::/8`, and stays inside an organization. A ULA has a random 40-bit global ID, so networks are unlikely to clash when they merge.

**Q: Why does IPv6 have no broadcast, and what replaces it?**
A: Broadcast forces every device on a LAN to look at each packet, which wastes resources. IPv6 uses multicast so that only interested devices get it. For example, NDP uses a solicited-node group instead of an ARP broadcast. `ff02::1` reaches all nodes, but it is still multicast.

**Q: What replaces ARP in IPv6?**
A: NDP, which runs over ICMPv6. A host sends a Neighbor Solicitation to the target's solicited-node multicast address, and the target answers with a Neighbor Advertisement that holds its MAC address.

**Q: What is anycast and how is it different from multicast?**
A: With anycast, several devices share one address and the packet goes to only the nearest one, chosen by routing. With multicast, the packet goes to all members of a group. Anycast has no special prefix, because it uses normal unicast addresses.

**Q: What changed in the IPv6 header compared to IPv4?**
A: It has a fixed 40-byte size. The checksum is gone, and so are the fragmentation fields and options, which moved to extension headers. Routers do not fragment. TTL became Hop Limit, and Protocol became Next Header. A Flow Label was added.

## Quick Recap

- IPv6 addresses are 128 bits, written as 8 groups of hex digits, and they solve the IPv4 shortage.
- Drop leading zeros, and use `::` once for the longest zero run. Trailing zeros always stay.
- A LAN subnet is normally /64: 64 bits of prefix, then 64 bits of interface ID.
- Know these types: global `2000::/3`, link-local `fe80::/10`, ULA `fc00::/7`, multicast `ff00::/8`, loopback `::1`, unspecified `::`. Anycast uses normal unicast addresses.
- There is no broadcast. Multicast does that job, and NDP replaces ARP.
- An interface normally has several addresses, and every IPv6 interface has a link-local one.
- The header is a fixed 40 bytes, routers do not fragment, and ICMPv6 must not be fully blocked.
