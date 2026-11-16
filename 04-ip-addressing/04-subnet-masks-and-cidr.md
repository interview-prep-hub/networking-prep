# Subnet Masks and CIDR

**Level:** Beginner | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [IPv4 Addressing Basics](01-ipv4-basics.md), [IPv4 Classes (Classful Addressing)](02-ipv4-classes.md)

## In One Line

A subnet mask tells a device which part of an IP address names the network and which part names the host. CIDR is the short way to write that mask, such as `/24`.

## Why It Matters

An IP address alone does not tell you where the network ends. `192.168.10.130` could live in a network of 2 addresses or 16 million addresses. The mask answers that question.

Every device needs this answer before it sends a packet. It must decide: "Is this destination on my own network, or must I send it to the router?" A wrong mask gives a wrong answer, and traffic fails in confusing ways.

Interviewers ask about this topic for three reasons:

- It tests whether you understand addressing or only memorised it.
- Mask errors are a common real fault.
- Masks appear in every area of the job: routing, firewalls, cloud networks and containers.

## Simple Explanation

Think of a phone number: `021-5550123`.

The first part, `021`, is the city code. The rest is the person. If a friend has the same city code, you call directly. If the code is different, the call goes through a long-distance exchange.

An IP address works the same way:

- The **network part** is like the city code.
- The **host part** is like the person. A host is any device with an IP address.
- The **subnet mask** is a note that says where the city code ends.
- The **default gateway** (the router that leads out of your network) is like the long-distance exchange.

A **subnet** is one network block. It is a group of addresses that share the same network part.

Where the analogy stops being true:

- Phone codes are fixed by the phone company. You choose the mask yourself.
- A city code ends between two digits. A mask can end in the middle of a decimal number, because it works on bits, not on decimal digits.
- Each device holds its own copy of the mask. Nobody checks that two devices agree.

## How It Works

### What a mask does

An IPv4 address is 32 **bits** (a bit is a single 0 or 1). A subnet mask is also 32 bits. We write both as four decimal numbers called **octets** (groups of 8 bits).

The mask follows one strict rule: all the 1s come first, then all the 0s.

- A `1` in the mask means: this bit of the address is the network part.
- A `0` in the mask means: this bit of the address is the host part.

```text
Mask 255.255.255.0

  11111111.11111111.11111111.00000000
  |<---- network part ---->| |<host>|
           24 ones             8 zeros
```

Because of that rule, only nine values can appear in a mask octet:

| Mask octet | Binary | Ones in the octet | Block size (256 minus octet) |
|---|---|---|---|
| 0 | 00000000 | 0 | 256 |
| 128 | 10000000 | 1 | 128 |
| 192 | 11000000 | 2 | 64 |
| 224 | 11100000 | 3 | 32 |
| 240 | 11110000 | 4 | 16 |
| 248 | 11111000 | 5 | 8 |
| 252 | 11111100 | 6 | 4 |
| 254 | 11111110 | 7 | 2 |
| 255 | 11111111 | 8 | 1 |

**Block size** means how far apart two neighbouring networks are in that octet. You will use it as a shortcut below.

Two addresses in every normal subnet are reserved:

- **Network address**: all host bits are 0. It names the subnet itself.
- **Broadcast address**: all host bits are 1. A packet sent there reaches every host in the subnet.

### Binary AND finds the network

**AND** is a simple operation on two bits. The result is 1 only when both bits are 1.

| Address bit | Mask bit | Result |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

So a mask bit of 1 keeps the address bit, and a mask bit of 0 wipes it to 0. The network part survives and the host part becomes zero. The result is the network address.

To find the network address:

1. Write the address in binary.
2. Write the mask in binary below it.
3. AND each pair of bits.
4. Convert the result back to decimal.

```text
Address  192.168.10.130   11000000.10101000.00001010.10000010
Mask     255.255.255.192  11111111.11111111.11111111.11000000
                          ----------------------------------- AND
Network  192.168.10.128   11000000.10101000.00001010.10000000
```

You only need to do real work on an octet where the mask is not 255 or 0. A 255 keeps the octet as it is. A 0 turns the octet into 0.

### How a host uses this

Before sending, a host does this:

```text
Host A wants to send to destination D

  A's IP AND A's mask  --->  A's network
  D's IP AND A's mask  --->  D's network (as A sees it)

              Same network?
              /           \
            yes            no
             |              |
        ARP for D      ARP for the default gateway
        send directly  send the frame to the gateway
```

[ARP](../03-ethernet-and-switching/05-arp.md) is the protocol that finds the MAC address for an IP address on the local network. See [The Default Gateway](08-default-gateway.md) for the off-network case.

Fuller truth: a real operating system does not run this exact check. It looks up the destination in its routing table. Setting an address and mask creates a local route for that subnet, so the result is the same in the normal case.

### CIDR slash notation

**CIDR** means Classless Inter-Domain Routing. It has two parts:

1. A short way to write a mask: `/n`, where n is the number of 1 bits. This n is called the **prefix length**.
2. The freedom to use any network size, not only the three fixed sizes of the old [class system](02-ipv4-classes.md).

`192.168.10.130/26` and `192.168.10.130 255.255.255.192` say the same thing.

Why CIDR was needed: under classes, a company that needed 2,000 addresses could not fit in a Class C network, which has 254 hosts. It usually got a Class B network with 65,534 hosts, and most of those were wasted. Internet routers were also filling with too many routes. CIDR was introduced in 1993 to fix both problems. The current standard is RFC 4632. With CIDR, that company gets a `/21` with 2,046 hosts.

**Convert a prefix to a mask.** Take `/20`:

1. Split 20 into groups of 8: 8 + 8 + 4.
2. Each 8 gives 255.
3. The remaining 4 bits give `11110000`, which is 240.
4. Fill the rest with 0: `255.255.240.0`.

**Convert a mask to a prefix.** Take `255.255.252.0`:

1. 255 has 8 ones, and 252 has 6 ones.
2. 8 + 8 + 6 = 22, so the prefix is `/22`.

**Count addresses.** Let h = 32 minus the prefix length. This is the number of host bits.

- Total addresses = 2^h
- Usable hosts = 2^h - 2 (we remove the network and broadcast addresses)

For `/26`: h = 6, so 64 addresses and 62 usable hosts. `/31` and `/32` are special cases, covered below. To split a network into pieces, see [Subnetting Step by Step](05-subnetting-step-by-step.md).

### Full reference table: /8 to /32

Each step down the table adds one network bit and cuts the size in half.

| Prefix | Subnet mask | Wildcard mask | Addresses | Usable hosts |
|---|---|---|---|---|
| /8 | 255.0.0.0 | 0.255.255.255 | 16,777,216 | 16,777,214 |
| /9 | 255.128.0.0 | 0.127.255.255 | 8,388,608 | 8,388,606 |
| /10 | 255.192.0.0 | 0.63.255.255 | 4,194,304 | 4,194,302 |
| /11 | 255.224.0.0 | 0.31.255.255 | 2,097,152 | 2,097,150 |
| /12 | 255.240.0.0 | 0.15.255.255 | 1,048,576 | 1,048,574 |
| /13 | 255.248.0.0 | 0.7.255.255 | 524,288 | 524,286 |
| /14 | 255.252.0.0 | 0.3.255.255 | 262,144 | 262,142 |
| /15 | 255.254.0.0 | 0.1.255.255 | 131,072 | 131,070 |
| /16 | 255.255.0.0 | 0.0.255.255 | 65,536 | 65,534 |
| /17 | 255.255.128.0 | 0.0.127.255 | 32,768 | 32,766 |
| /18 | 255.255.192.0 | 0.0.63.255 | 16,384 | 16,382 |
| /19 | 255.255.224.0 | 0.0.31.255 | 8,192 | 8,190 |
| /20 | 255.255.240.0 | 0.0.15.255 | 4,096 | 4,094 |
| /21 | 255.255.248.0 | 0.0.7.255 | 2,048 | 2,046 |
| /22 | 255.255.252.0 | 0.0.3.255 | 1,024 | 1,022 |
| /23 | 255.255.254.0 | 0.0.1.255 | 512 | 510 |
| /24 | 255.255.255.0 | 0.0.0.255 | 256 | 254 |
| /25 | 255.255.255.128 | 0.0.0.127 | 128 | 126 |
| /26 | 255.255.255.192 | 0.0.0.63 | 64 | 62 |
| /27 | 255.255.255.224 | 0.0.0.31 | 32 | 30 |
| /28 | 255.255.255.240 | 0.0.0.15 | 16 | 14 |
| /29 | 255.255.255.248 | 0.0.0.7 | 8 | 6 |
| /30 | 255.255.255.252 | 0.0.0.3 | 4 | 2 |
| /31 | 255.255.255.254 | 0.0.0.1 | 2 | 2 (point-to-point link only) |
| /32 | 255.255.255.255 | 0.0.0.0 | 1 | 1 (a single host) |

Memorise `/24` to `/32` first. You can rebuild the rest, because the pattern repeats: `/16` to `/23` has the same mask values, one octet to the left.

Prefixes shorter than `/8` also exist. `/0` is the default route, explained in Going Deeper.

### Wildcard masks

A **wildcard mask** is a subnet mask with every bit flipped. It tells a device which address bits to check and which to ignore.

- `0` means: this bit must match.
- `1` means: ignore this bit.

Think of a security guard checking an ID number. The guard has a card that says "check these digits, skip those digits". The analogy stops at this point: a guard reads digits, but a wildcard works on single bits.

To calculate one, subtract each mask octet from 255:

```text
  255.255.255.255
- 255.255.255.192    (the /26 mask)
  ---------------
    0.  0.  0. 63    (the /26 wildcard)
```

Wildcard masks are used on Cisco IOS for [access control lists (ACLs)](../09-security/05-access-control-lists.md), which are rules that permit or deny traffic, and for the `network` command in [OSPF](../05-routing/06-ospf.md).

```text
! Cisco IOS: match every address in 192.168.10.128/26
access-list 10 permit 192.168.10.128 0.0.0.63

! Cisco IOS: run OSPF on interfaces inside the same range
router ospf 1
 network 192.168.10.128 0.0.0.63 area 0
```

Two shortcuts exist in Cisco ACLs:

| Keyword | Long form | Matches |
|---|---|---|
| `host 10.1.1.1` | `10.1.1.1 0.0.0.0` | One address |
| `any` | `0.0.0.0 255.255.255.255` | Every address |

Vendor note: this is a Cisco IOS habit, not a universal rule. Cisco ASA firewall ACLs use normal subnet masks. Linux tools and cloud platforms use CIDR notation.

Fuller truth: a wildcard does not have to be a flipped subnet mask. Its 0s and 1s can be mixed. For example, `192.168.1.0 0.0.0.254` checks the first three octets and only the lowest bit of the last octet, so it matches the even addresses from .0 to .254. A subnet mask can never do this.

### Why subnet at all

Subnetting means splitting one large network into smaller ones. There are five main reasons.

1. **Less broadcast traffic.** Hosts send broadcasts often, for example for ARP and [DHCP](../07-application-protocols/04-dhcp.md). Every device in the broadcast area must process them. Smaller networks mean fewer devices hear each one.
2. **Better security.** Traffic between subnets must pass through a router or firewall. You can filter it there. Guest Wi-Fi can be kept away from your servers.
3. **Less wasted address space.** A link between two routers needs 2 addresses, not 254. Public IPv4 addresses are scarce and cost money.
4. **Smaller failures.** A broadcast storm (a flood of broadcast traffic) or a bad device stays inside its own network.
5. **Clear organisation.** If each building or team has its own block, addresses tell you where a device is. Routes can also be grouped together. See [VLSM, Route Summarisation and Supernetting](07-vlsm-and-supernetting.md).

```text
Before: one flat /16. Every host hears every broadcast.

  [ 10.1.0.0/16: office + servers + cameras + guests ]

After: one subnet per purpose, with a firewall between them.

  10.1.10.0/24  office  ---+
  10.1.20.0/24  servers ---+---[ router/firewall ]--- internet
  10.1.30.0/24  cameras ---+
  10.1.40.0/24  guests  ---+
```

Fuller truth: a subnet is a Layer 3 idea, and a broadcast area is a Layer 2 idea. The mask alone does not stop a broadcast. A router or a [VLAN](../03-ethernet-and-switching/06-vlans.md) boundary does. In good designs each VLAN holds one subnet, so the two match. See [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).

### /31 and /32: the two special cases

**`/30`, the old way.** A **point-to-point link** connects exactly two devices, such as two routers. A `/30` gives 4 addresses, and 2 of them are lost:

| Address | Role in 10.0.0.0/30 |
|---|---|
| 10.0.0.0 | Network, not usable |
| 10.0.0.1 | Router 1 |
| 10.0.0.2 | Router 2 |
| 10.0.0.3 | Broadcast, not usable |

**`/31`, the modern way.** RFC 3021 allows a `/31` on point-to-point links. It has 2 addresses and both are usable. There is no network or broadcast address, because a link with two ends has no need for them. If you have 1,000 links, you save 2,000 addresses.

```text
   10.0.0.0/31                  10.0.0.1/31
  [ R1 ]-----------------------------[ R2 ]
```

```text
! Cisco IOS: /31 on an Ethernet link
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.0.0 255.255.255.254
% Warning: use /31 mask on non point-to-point interface cautiously
```

That warning is normal on Ethernet, because Ethernet could carry more than two devices. Do not use a `/31` on a shared LAN. Support on other devices varies, so check before you rely on it.

**`/32` means one exact address.** The whole address is the network part, so nothing is left for hosts. Common uses:

- **Loopback interfaces.** A loopback is a virtual interface on a router. It stays up even when physical links fail, so it gives a stable address for managing the router.
- **Host routes.** A route to one device only.
- **Firewall and security rules.** "Allow SSH only from 203.0.113.10/32."
- **VPN clients.** Many VPNs give each client one address, for example `AllowedIPs = 10.8.0.2/32` in WireGuard.
- **Anycast services.** Several servers announce the same `/32`. See [Unicast, Broadcast, Multicast and Anycast](13-unicast-broadcast-multicast-anycast.md).

```text
! Cisco IOS: a loopback address and a host route
interface Loopback0
 ip address 10.255.0.1 255.255.255.255

ip route 198.51.100.25 255.255.255.255 10.0.0.1
```

```bash
# Linux: add a host route, then allow one source address
ip route add 198.51.100.25/32 via 10.0.0.1
iptables -A INPUT -s 203.0.113.10/32 -p tcp --dport 22 -j ACCEPT
```

## Example

### Example 1: full analysis of 192.168.10.130/26

**Step 1: find the mask.** `/26` = 8 + 8 + 8 + 2, so the mask is `255.255.255.192`.

**Step 2: find the interesting octet.** The fourth octet is where the mask is not 255 or 0.

**Step 3: AND that octet.**

```text
130   10000010
192   11000000
      -------- AND
128   10000000
```

The network address is `192.168.10.128`.

The shortcut gives the same answer. Block size is 256 - 192 = 64. The multiples of 64 are 0, 64, 128, 192. The address 130 falls in the block that starts at 128.

**Step 4: find the broadcast address.** Set all 6 host bits to 1: `10111111` = 191. You can also take 128 + 64 - 1 = 191.

**Step 5: write the result.**

| Item | Value |
|---|---|
| Network address | 192.168.10.128 |
| First usable host | 192.168.10.129 |
| Last usable host | 192.168.10.190 |
| Broadcast address | 192.168.10.191 |
| Usable hosts | 62 |
| Wildcard mask | 0.0.0.63 |

**See it on real systems.** Here the router uses `192.168.10.129`.

```bash
# Linux
$ ip -4 addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.10.130/26 brd 192.168.10.191 scope global eth0
       valid_lft forever preferred_lft forever

$ ip route
default via 192.168.10.129 dev eth0
192.168.10.128/26 dev eth0 proto kernel scope link src 192.168.10.130
```

```powershell
# Windows
PS> ipconfig
Ethernet adapter Ethernet:

   IPv4 Address. . . . . . . . . . . : 192.168.10.130
   Subnet Mask . . . . . . . . . . . : 255.255.255.192
   Default Gateway . . . . . . . . . : 192.168.10.129
```

```text
! Cisco IOS
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 192.168.10.129 255.255.255.192
Router(config-if)# no shutdown
Router(config-if)# end
Router# show ip interface GigabitEthernet0/0 | include Internet
  Internet address is 192.168.10.129/26
```

**Check your work with a tool.** The `ipcalc` package prints the bits. Its output differs between Linux distributions. This is the Debian and Ubuntu version.

```bash
$ ipcalc 192.168.10.130/26
Address:   192.168.10.130       11000000.10101000.00001010.10 000010
Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
Wildcard:  0.0.0.63             00000000.00000000.00000000.00 111111
=>
Network:   192.168.10.128/26    11000000.10101000.00001010.10 000000
HostMin:   192.168.10.129       11000000.10101000.00001010.10 000001
HostMax:   192.168.10.190       11000000.10101000.00001010.10 111110
Broadcast: 192.168.10.191       11000000.10101000.00001010.10 111111
Hosts/Net: 62                    Class C, Private Internet
```

### Example 2: a mask that ends in the third octet

Find the network for `172.16.45.200/20`.

1. `/20` is `255.255.240.0`. The interesting octet is the third.
2. AND the third octet: 45 is `00101101` and 240 is `11110000`. The result is `00100000`, which is 32.
3. The fourth octet has mask 0, so it becomes 0. The network is `172.16.32.0`.
4. Block size is 256 - 240 = 16, so the next network starts at `172.16.48.0`.
5. The broadcast is one address before that: `172.16.47.255`.
6. Usable hosts run from `172.16.32.1` to `172.16.47.254`. That is 4,094 hosts.

Notice that `172.16.33.0` and `172.16.40.255` are normal host addresses here. An address ending in 0 or 255 is not always special. It depends on the mask.

### Example 3: same subnet or not?

Host A is `192.168.10.130/26`. Host B is `192.168.10.200/26`. Can A reach B directly?

```text
A: 130 AND 192 = 128   network 192.168.10.128
B: 200 AND 192 = 192   network 192.168.10.192
```

They are on different networks, even though they look alike. A must send through its default gateway.

### Example 4: mismatched masks

Host A is `192.168.1.10/24`. Host B is `192.168.1.200/25`. They share one switch, with a router at `192.168.1.129`.

```text
A's view: B is in 192.168.1.0/24   -> "local, I will ARP for B"
B's view: A is not in 192.168.1.128/25 -> "remote, send to my gateway"
```

A sends directly to B. B sends its reply to the router. A is outside the router's `/25`, so the router may send the reply somewhere else or drop it. Traffic may work by luck, may be slow, or may fail completely. No device shows a clear error, which is why this fault is hard to find.

## Going Deeper

- **`/0` means everything.** `0.0.0.0/0` has no network bits, so it matches every IPv4 address. This is the default route. See [Static and Default Routes](../05-routing/03-static-and-default-routes.md).
- **The mask is not in the packet.** The [IPv4 header](10-ipv4-header-and-fragmentation.md) has no mask field. Each host uses its own local mask. Routing protocols are different: RIPv2, OSPF, EIGRP, IS-IS and BGP all carry the prefix length with each route. RIPv1 did not, which is one reason it could not work with CIDR.
- **Longer prefix wins.** A router that matches both `10.0.0.0/8` and `10.1.2.0/24` uses the `/24`, because it is more specific. See [The Routing Table](../05-routing/02-routing-table.md).
- **Cloud subnets lose more addresses.** AWS reserves 5 addresses in every subnet, not 2, and its smallest IPv4 subnet is `/28`. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).
- **Public routing has a practical limit.** Most networks on the public internet do not accept IPv4 routes longer than `/24`. This is common practice, not a standard. See [BGP](../05-routing/09-bgp.md).
- **IPv6 has no dotted mask.** It always uses a prefix length, such as `2001:db8:1:1::/64`. See [IPv6 Basics](11-ipv6-basics.md).

## Common Mistakes

- Thinking `/24` means Class C. `10.1.2.0/24` is a `/24` inside old Class A space. Prefix length and class are separate ideas.
- Forgetting the router. A `/26` has 62 usable hosts, but only 61 are left for devices once the gateway takes one.
- Assuming an address ending in `.0` or `.255` is always invalid. In `10.0.0.0/23`, the address `10.0.0.255` is a normal host.
- Using a host address as a network. `192.168.10.130/26` is valid on an interface, but as a route the network must be `192.168.10.128/26`.
- Applying the -2 rule to `/31` and `/32`. The formula gives 0 and -1, which is wrong. A `/31` has 2 usable addresses and a `/32` has 1.
- Writing a subnet mask where a wildcard is needed on Cisco IOS. `access-list 10 permit 192.168.10.0 255.255.255.0` does not match that subnet. IOS reads it as a wildcard, so it matches any address that ends in `.0`.
- Using an invalid mask. `255.255.253.0` is not a valid mask, because 253 is `11111101`, with a 0 before a 1.
- Believing a mask stops broadcasts by itself. A router or VLAN boundary stops them.
- Repeating the old rule that the first and last subnets cannot be used. That was true long ago. Modern equipment uses them normally.

## Interview Questions

**Q: What does a subnet mask do?**
A: It splits an IP address into a network part and a host part. The 1 bits mark the network and the 0 bits mark the host. A device uses it to decide if a destination is local or must go to the gateway.

**Q: How do you find the network address from an IP and a mask?**
A: I do a binary AND between the address and the mask. Mask bits of 1 keep the address bits, and mask bits of 0 clear them. The result is the network address.

**Q: What is CIDR and why was it created?**
A: CIDR is Classless Inter-Domain Routing. It replaced fixed class sizes with any prefix length, written as a slash number. It was created to reduce wasted addresses and to slow the growth of internet routing tables by allowing route aggregation.

**Q: For a /27, give me the mask, the wildcard and the number of hosts.**
A: The mask is 255.255.255.224 and the wildcard is 0.0.0.31. There are 5 host bits, so 32 addresses and 30 usable hosts.

**Q: What is the network, broadcast and host range for 10.50.100.77/29?**
A: A /29 has a block size of 8. The multiple of 8 at or below 77 is 72. So the network is 10.50.100.72, the broadcast is 10.50.100.79, and the hosts are .73 to .78.

**Q: How many /24 networks fit in a /20?**
A: Sixteen. The difference is 4 bits, and 2 to the power of 4 is 16.

**Q: What is a wildcard mask and how is it different from a subnet mask?**
A: It is the bit inverse of a subnet mask in the normal case. A 0 means match this bit and a 1 means ignore it. Cisco IOS uses it in ACLs and in the OSPF network command. Unlike a subnet mask, its 0s and 1s may be mixed.

**Q: Why would you subnet a network?**
A: To reduce broadcast traffic, to create security boundaries, to avoid wasting addresses, to limit the effect of failures, and to organise addresses so routes can be summarised.

**Q: Why can a /31 work when it has no host bits left after removing network and broadcast?**
A: RFC 3021 defines /31 for point-to-point links. With only two ends, there is no need for network or broadcast addresses, so both addresses are used by the two devices. It saves half the addresses compared with a /30.

**Q: Where would you use a /32?**
A: On loopback interfaces, for host routes, in firewall rules for one source, for VPN client addresses, and for anycast services.

**Q: Two hosts on one switch use different masks. What happens?**
A: Each host makes its own decision. One may treat the other as local while the other sends through the gateway. Traffic then takes different paths in each direction, and it can fail in ways that look random. I would compare the masks on both hosts.

**Q: Is the subnet mask carried inside the IP packet?**
A: No. The IPv4 header has only source and destination addresses. Each device uses its own configured mask. Routing protocols such as OSPF and BGP do carry prefix lengths in their own messages.

## Quick Recap

- A mask marks the network part with 1s and the host part with 0s, and the 1s always come first.
- Address AND mask gives the network address. Set all host bits to 1 to get the broadcast address.
- `/n` counts the 1 bits. Addresses = 2^(32 - n), and usable hosts = that number minus 2.
- A wildcard mask is 255.255.255.255 minus the subnet mask. 0 means match and 1 means ignore.
- We subnet for smaller broadcast areas, security, less waste, smaller failures and clear organisation.
- `/31` gives 2 usable addresses for point-to-point links, and `/32` names one exact address.
- Every device uses its own mask, and the mask is never in the packet, so a wrong mask causes silent, confusing faults.
