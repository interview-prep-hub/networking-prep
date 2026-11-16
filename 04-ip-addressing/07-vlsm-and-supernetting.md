# VLSM, Route Summarisation and Supernetting

**Level:** Intermediate | **Prerequisites:** [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md), [Subnetting Step by Step](05-subnetting-step-by-step.md), [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md)

## In One Line

VLSM cuts one address block into subnets of different sizes, so each network gets only what it needs. Summarisation does the opposite: it joins many neighbouring networks into one short route.

## Why It Matters

Networks are not all the same size. A sales floor may have 100 devices. A link between two routers has only 2.

If every subnet must be the same size, you have a bad choice. Make them big and the small networks waste addresses. Make them small and the big networks do not fit.

This costs real money and real effort:

- Public IPv4 addresses are scarce. The central free pool at IANA (the body that hands out address blocks to regions) ran out in 2011.
- Private space is free, but it is not unlimited in practice. Cloud networks are often small, and a sloppy plan collides with other networks later.
- Routers have limited memory and CPU. The global Internet routing table holds around one million IPv4 routes. Without aggregation it would be far larger.

Interviewers ask about this topic for three reasons:

1. It shows you can do the binary, not only recite a formula.
2. It shows you can design an address plan, not only read one.
3. It connects addressing to routing, because a good plan makes short routing tables.

## Simple Explanation

**VLSM: cutting land into plots.**

Imagine you own a large piece of land. You can divide it into equal plots, so a tea stall and a factory get the same plot. The tea stall wastes most of it, and the factory may not fit.

A better way is to cut by need. Give the factory half of the land. Cut the other half in two and give a quarter to a shop. Keep halving for smaller and smaller tenants.

That is VLSM. The name means Variable Length Subnet Mask: the mask length changes from subnet to subnet.

Where the analogy stops: land can be cut at any line. An address block can only be cut in half, then in half again. Every piece is a power of 2 in size, and it can only start at certain positions.

**Summarisation: the post office.**

A post office in another country does not know every street in Lahore. It only needs one fact: "Anything for Pakistan goes on this plane." People closer to the destination read the finer details.

A summary route works the same way. A far-away router keeps one line for a whole group of networks. Routers near the destination know the details.

Where the analogy stops: a letter with a bad street name usually comes back to you. A packet that follows a summary to a network that does not exist is simply dropped near the end of its trip. The sender may get an error message or nothing at all.

**Overlap: two deeds for one plot.**

Imagine two owners holding deeds for the same piece of land. Nobody notices until both try to build.

Overlapping subnets are the same. Two networks claim the same addresses, and things seem fine until a device uses one of the shared addresses.

Where the analogy stops: a land dispute goes to a court. A router just follows its rules, without warning you, so some traffic works and some vanishes.

## How It Works

### Words You Need First

| Term | Plain meaning |
|---|---|
| Prefix | The network part of an address. `/26` means the first 26 bits are the network. |
| Host bits | The bits left over after the prefix. They number the devices. |
| Block size | How many addresses a subnet holds. It is 2 to the power of the host bits. |
| Network address | The first address in a block. It names the subnet. |
| Broadcast address | The last address in a block. It reaches every host in the subnet. |
| Usable hosts | Block size minus 2, because the first and last are reserved. |
| Contiguous | Next to each other, with no gap. |
| Aligned | Starting at a multiple of the block size. |
| FLSM | Fixed Length Subnet Masking: every subnet uses the same mask. |

### Part 1: Why Fixed-Size Subnets Waste Addresses

Suppose you have `192.168.10.0/24` (256 addresses) and these needs:

| Network | Devices |
|---|---|
| Sales | 100 |
| Engineering | 50 |
| Guest Wi-Fi | 25 |
| Servers | 10 |
| Router link 1 | 2 |
| Router link 2 | 2 |
| Router link 3 | 2 |

That is 7 networks and 191 devices. With one mask for all, nothing works:

- `/25` holds 126 hosts, enough for Sales. But a `/24` gives only 2 of them. You need 7.
- `/27` gives 8 subnets, enough to count. But each holds only 30 hosts. Sales and Engineering do not fit.

To make fixed `/25` subnets work, you would need 7 blocks of 128 addresses. That is 896 addresses, so you must ask for a `/22` (1024 addresses).

Compare the two plans:

| Network | Hosts needed | Fixed /25 usable | Fixed unused | VLSM prefix | VLSM usable | VLSM unused |
|---|---|---|---|---|---|---|
| Sales | 100 | 126 | 26 | /25 | 126 | 26 |
| Engineering | 50 | 126 | 76 | /26 | 62 | 12 |
| Guest Wi-Fi | 25 | 126 | 101 | /27 | 30 | 5 |
| Servers | 10 | 126 | 116 | /28 | 14 | 4 |
| Link 1 | 2 | 126 | 124 | /30 | 2 | 0 |
| Link 2 | 2 | 126 | 124 | /30 | 2 | 0 |
| Link 3 | 2 | 126 | 124 | /30 | 2 | 0 |
| **Total** | **191** | **882** | **691** | | **238** | **47** |

The fixed plan leaves 691 usable addresses idle. The VLSM plan leaves 47, and it fits in the `/24` you already have.

Fixed masks were once forced on people. Old routing protocols such as RIPv1 do not send the mask in their updates, so a router had to assume one mask across a network. Modern protocols send the mask with each route. VLSM needs this, as Going Deeper explains.

### Part 2: The VLSM Design Process, Largest First

Use this table for every design. Keep it near you.

| Usable hosts (max) | Host bits | Block size | Prefix | Mask |
|---|---|---|---|---|
| 2 | 2 | 4 | /30 | 255.255.255.252 |
| 6 | 3 | 8 | /29 | 255.255.255.248 |
| 14 | 4 | 16 | /28 | 255.255.255.240 |
| 30 | 5 | 32 | /27 | 255.255.255.224 |
| 62 | 6 | 64 | /26 | 255.255.255.192 |
| 126 | 7 | 128 | /25 | 255.255.255.128 |
| 254 | 8 | 256 | /24 | 255.255.255.0 |
| 510 | 9 | 512 | /23 | 255.255.254.0 |
| 1022 | 10 | 1024 | /22 | 255.255.252.0 |

The process has seven steps:

1. **List every network**, including router links. Links are the ones people forget.
2. **Count real hosts for each.** Include the router's own address, printers, access points and growth.
3. **Sort from largest to smallest.**
4. **Choose a block for each.** Add 2 to the hosts, then round up to a power of 2.
5. **Check the total.** Add the block sizes. The sum must not be more than your starting block.
6. **Allocate in sorted order.** Start at the first address. Each new subnet starts right after the last one ended.
7. **Write down every subnet**, with its network, usable range, broadcast and mask. Record the free space too.

**Why largest first works.** A subnet must start at a multiple of its block size. If you place blocks from large to small, everything used so far adds up to a multiple of the current block size. So the next free address is always a valid start. You never skip addresses.

Here is what goes wrong if you allocate in a random order and only move forward. All numbers are the last octet of `192.168.10.x`:

```text
Step 1: Guest /27 at 0-31. Next free is 32.
Step 2: Sales /25 must start at 0 or 128. 32 is not valid.
        Jump to 128-255. Addresses 32-127 are skipped.
Step 3: Engineering /26. Next free after 255 is... nothing.
        You think you are out of space, but 96 addresses sit unused.
```

**The fuller truth:** largest first is not the only correct order. If you track the gaps with care, other orders can also fit. Largest first is simply the method that makes mistakes hard. Also, real designs rarely pack subnets this tightly. Engineers leave room to grow, and they choose boundaries that summarise cleanly.

### Part 3: Supernetting and Route Aggregation

Subnetting makes the prefix **longer** and gives you more, smaller networks. Supernetting makes the prefix **shorter** and gives you one bigger network.

```text
   Subnetting: borrow bits, one block becomes many

        172.16.12.0/22
       /      |      |      \
  .12.0/24 .13.0/24 .14.0/24 .15.0/24

   Summarisation: give bits back, many routes become one
```

These words are close, and people often mix them up:

| Term | Meaning |
|---|---|
| Route summarisation | Advertising one route that covers many smaller ones. Common word in OSPF, EIGRP and general routing. |
| Route aggregation | The same idea. Common word in BGP and among ISPs. |
| Supernet | Strictly, a block with a prefix shorter than the old classful mask for that address. |
| CIDR | Classless Inter-Domain Routing. The system that dropped classes, so any prefix length is allowed. Current standard: RFC 4632. |

An example shows the strict difference. `192.168.16.0/21` is a true supernet, because a Class C address had a `/24` mask and `/21` is shorter. `172.16.12.0/22` is a summary but not strictly a supernet, because a Class B mask was `/16` and `/22` is longer. In interviews, most people use "supernet" and "summary" for the same thing, and that is fine. See [IPv4 Classes](02-ipv4-classes.md) for the history.

**What summarisation gives you:**

- **Smaller routing tables.** Lookups use less memory.
- **Fewer updates.** If `172.16.14.0/24` goes up and down, routers beyond the summary see no change. The problem stays local.
- **Less CPU work.** Link-state protocols such as OSPF do less recalculation.

**What it costs you:**

- Far-away routers lose detail, so they may not pick the best path.
- A summary can attract traffic for a network that is down or does not exist. We deal with this below.

### Part 4: How to Compute a Summary Route in Binary

Use these six steps:

1. Find the **lowest** address and the **highest** address in the group.
2. Write both in binary. You only need the octets where they differ.
3. Count matching bits from the left. Stop at the first difference.
4. The new prefix is the bits in the identical octets plus the matching bits you counted.
5. Copy the matching bits. Set everything after them to 0. That is the summary's network address.
6. Check what else the summary covers.

You only need the lowest and highest because every value between them shares their common bits.

**Example A: a perfect summary.** Summarise `172.16.12.0/24`, `172.16.13.0/24`, `172.16.14.0/24` and `172.16.15.0/24`.

The first two octets are identical, so that is 16 matching bits. The third octet differs:

```text
Network         Third octet
172.16.12.0     000011 00
172.16.13.0     000011 01
172.16.14.0     000011 10
172.16.15.0     000011 11
                ^^^^^^ ^^
                match  differ
```

The prefix is 16 + 6 = 22. Set the two differing bits to 0, and the third octet is `00001100`, which is 12.

Answer: `172.16.12.0/22`, mask `255.255.252.0`. It covers exactly the four networks, nothing more.

**Example B: a summary that covers too much.** Summarise `192.168.1.0/24`, `192.168.2.0/24`, `192.168.3.0/24` and `192.168.4.0/24`.

```text
Lowest   192.168.1.0    00000 001
Highest  192.168.4.0    00000 100
                        ^^^^^ ^^^
                        match differ
```

The prefix is 16 + 5 = 21, and the summary is `192.168.0.0/21`. This covers `192.168.0.0` to `192.168.7.255`, so it also includes `.0`, `.5`, `.6` and `.7`. You may not own those.

The smallest exact cover needs three routes: `192.168.1.0/24`, `192.168.2.0/23` and `192.168.4.0/24`.

**Example C: neighbours can still be far apart in binary.** Summarise `10.1.127.0/24` and `10.1.128.0/24`.

```text
Lowest   10.1.127.0     0 1111111
Highest  10.1.128.0     1 0000000
                        ^
                        differ at the first bit
```

The prefix is 16 + 0 = 16. To join two adjacent `/24` networks, you would need `10.1.0.0/16`, which holds 65,536 addresses. That is a terrible summary. Keep two routes.

**The shortcut for perfect summaries.** A group of same-size networks joins exactly when all three of these are true:

1. They are contiguous.
2. Their count is a power of 2.
3. The first network number is a multiple of the count.

Test `172.16.12.0/24` to `172.16.15.0/24`: 4 networks, 4 is a power of 2, and 12 is a multiple of 4. It works. Removing 2 bits from `/24` gives `/22`.

Test `172.16.13.0/24` and `172.16.14.0/24`: 2 networks, but 13 is not a multiple of 2. They do not form a `/23`. They are neighbours, but they do not belong to the same parent block. The smallest single route covering both is `172.16.12.0/22`.

**The fuller truth:** summaries can also join networks of different sizes. For example, `10.1.0.0/25`, `10.1.0.128/26` and `10.1.0.192/26` fill `10.1.0.0/24` exactly. When sizes differ, take the highest *last* address, not the highest network address. That last address is the one to compare with the lowest address.

### Part 5: Overlap Traps

Two subnets overlap when they share any address.

With proper prefixes there is a neat fact: partial overlap is impossible. Two blocks are either fully separate, or one sits completely inside the other.

**How to test for overlap by hand:**

1. Take the shorter of the two prefixes.
2. Apply it to both addresses.
3. If both give the same network, they overlap.

Test `192.168.10.0/25` and `192.168.10.64/26`. The shorter prefix is `/25`. Apply it to `.64`: the result is `.0`. Same network, so they overlap. The `/26` sits inside the `/25`.

Test `192.168.10.128/26` and `192.168.10.192/27`. The shorter prefix is `/26`. Apply it to `.192`: the result is `.192`, not `.128`. They do not overlap.

**Trap 1: overlap across two routers.** A Cisco router refuses an overlap on its own interfaces:

```text
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 192.168.10.65 255.255.255.192
% 192.168.10.64 overlaps with GigabitEthernet0/0
```

This check only sees that one router. Put `192.168.10.1/25` on R1 and `192.168.10.65/26` on R2, and neither one complains. Other routers then send everything for `.64` to `.127` towards R2, because the longer prefix wins. Any host on R1 with an address in that range loses traffic from outside its LAN.

**Trap 2: a start address that is not aligned.** `192.168.10.96/26` looks like a subnet, but 96 is not a multiple of 64. It is only a host inside `192.168.10.64/26`. So this "subnet" really covers `.64` to `.127`, and it may eat the subnet below it. Cisco IOS accepts this on an interface as a host address, with no warning. In a route, it refuses:

```text
R1(config)# ip route 192.168.10.96 255.255.255.192 10.0.0.2
%Inconsistent address and mask
```

**Trap 3: a mask mismatch on one LAN.** Two hosts share one switch:

```text
Host A  10.0.0.10/24    thinks 10.0.0.130 is on the local LAN
Host B  10.0.0.130/25   thinks 10.0.0.10 is on another network
```

Host A sends directly with [ARP](../03-ethernet-and-switching/05-arp.md). Host B sends its reply to the router. So the traffic can work in one direction and look strange or fail in the other. This is painful to debug.

**Trap 4: a summary that is too wide.** Branch A owns `10.20.16.0/22`, and Branch B owns `10.20.20.0/22`. Someone configures `10.20.16.0/21` on Branch A by mistake. While Branch B's routes are up, longer prefixes hide the error. When Branch B fails, its traffic goes to Branch A and dies there.

**Trap 5: overlap between organisations.** This one is very common:

- A home network uses `192.168.1.0/24`, and so does the office. On [VPN](../09-security/07-vpn.md), the laptop sends office traffic to its own home LAN.
- Two companies merge, and both use `10.0.0.0/16`.
- AWS VPC peering does not allow overlapping CIDR blocks.
- Docker's default bridge uses `172.17.0.0/16`, which can hide a real corporate network with the same range.

The fix is to renumber one side. If you cannot, use [NAT](09-nat-and-pat.md) to translate between them. Better still, choose unusual private ranges from the start.

**Overlapping routes are not the same as overlapping subnets.** This is an important difference:

- **Overlapping subnets assigned to LANs** are a design bug.
- **Overlapping routes in a routing table** are normal and useful. A default route overlaps everything. The router picks the longest match, as described in [The Routing Table](../05-routing/02-routing-table.md).

So this table is healthy:

```text
Route              Next hop
10.20.16.0/22      R1
10.20.19.0/24      R3

Packet to 10.20.18.5   matches only the /22  -> R1
Packet to 10.20.19.5   matches both          -> R3 (the /24 is longer)
```

## Example

### Example 1: Full VLSM Design

Given `192.168.10.0/24`, build the plan from Part 1.

**Steps 1 to 4: list, count, sort, choose a block.**

| Network | Hosts | Hosts + 2 | Block size | Prefix |
|---|---|---|---|---|
| Sales | 100 | 102 | 128 | /25 |
| Engineering | 50 | 52 | 64 | /26 |
| Guest Wi-Fi | 25 | 27 | 32 | /27 |
| Servers | 10 | 12 | 16 | /28 |
| Link 1 | 2 | 4 | 4 | /30 |
| Link 2 | 2 | 4 | 4 | /30 |
| Link 3 | 2 | 4 | 4 | /30 |

**Step 5: check the total.** 128 + 64 + 32 + 16 + 4 + 4 + 4 = 252. The block has 256, so it fits, with 4 addresses spare.

**Step 6: allocate.** Each subnet starts where the last one ended.

```text
Start at .0
Sales        block 128   .0   + 128 = next start .128
Engineering  block 64    .128 +  64 = next start .192
Guest        block 32    .192 +  32 = next start .224
Servers      block 16    .224 +  16 = next start .240
Link 1       block 4     .240 +   4 = next start .244
Link 2       block 4     .244 +   4 = next start .248
Link 3       block 4     .248 +   4 = next start .252
Free         block 4     .252 to .255
```

**Step 7: write the plan.**

| Network | Subnet | Mask | First usable | Last usable | Broadcast |
|---|---|---|---|---|---|
| Sales | 192.168.10.0/25 | 255.255.255.128 | 192.168.10.1 | 192.168.10.126 | 192.168.10.127 |
| Engineering | 192.168.10.128/26 | 255.255.255.192 | 192.168.10.129 | 192.168.10.190 | 192.168.10.191 |
| Guest Wi-Fi | 192.168.10.192/27 | 255.255.255.224 | 192.168.10.193 | 192.168.10.222 | 192.168.10.223 |
| Servers | 192.168.10.224/28 | 255.255.255.240 | 192.168.10.225 | 192.168.10.238 | 192.168.10.239 |
| Link 1 | 192.168.10.240/30 | 255.255.255.252 | 192.168.10.241 | 192.168.10.242 | 192.168.10.243 |
| Link 2 | 192.168.10.244/30 | 255.255.255.252 | 192.168.10.245 | 192.168.10.246 | 192.168.10.247 |
| Link 3 | 192.168.10.248/30 | 255.255.255.252 | 192.168.10.249 | 192.168.10.250 | 192.168.10.251 |
| Free | 192.168.10.252/30 | 255.255.255.252 | 192.168.10.253 | 192.168.10.254 | 192.168.10.255 |

The same plan as repeated halving:

```text
192.168.10.0/24
|-- 192.168.10.0/25        Sales
`-- 192.168.10.128/25
    |-- 192.168.10.128/26      Engineering
    `-- 192.168.10.192/26
        |-- 192.168.10.192/27      Guest Wi-Fi
        `-- 192.168.10.224/27
            |-- 192.168.10.224/28      Servers
            `-- 192.168.10.240/28
                |-- 192.168.10.240/30  Link 1
                |-- 192.168.10.244/30  Link 2
                |-- 192.168.10.248/30  Link 3
                `-- 192.168.10.252/30  Free
```

And as a map of the last octet:

```text
0                  128       192   224  240       255
+------------------+---------+-----+----+--+--+--+--+
| Sales /25        | Eng /26 |Guest|Srv |L1|L2|L3|F |
+------------------+---------+-----+----+--+--+--+--+
                              /27   /28  each /30
```

Notice one more thing. Every subnet came from `192.168.10.0/24`, so the rest of the network can reach all of them through that one route. VLSM cuts the block up inside the site, and summarisation presents it as one block to the outside.

### Example 2: A Design That Crosses Octets

Given `172.16.4.0/22` (`172.16.4.0` to `172.16.7.255`, 1024 addresses), plan for LAN A with 400 hosts, LAN B with 200 hosts, LAN C with 60 hosts and two router links.

The total is 512 + 256 + 64 + 4 + 4 = 840 addresses, which fits in 1024.

| Network | Hosts | Block | Subnet | Usable range | Broadcast |
|---|---|---|---|---|---|
| LAN A | 400 | 512 | 172.16.4.0/23 | 172.16.4.1 - 172.16.5.254 | 172.16.5.255 |
| LAN B | 200 | 256 | 172.16.6.0/24 | 172.16.6.1 - 172.16.6.254 | 172.16.6.255 |
| LAN C | 60 | 64 | 172.16.7.0/26 | 172.16.7.1 - 172.16.7.62 | 172.16.7.63 |
| Link 1 | 2 | 4 | 172.16.7.64/30 | 172.16.7.65 - 172.16.7.66 | 172.16.7.67 |
| Link 2 | 2 | 4 | 172.16.7.68/30 | 172.16.7.69 - 172.16.7.70 | 172.16.7.71 |

Free space is `172.16.7.72` to `172.16.7.255`. As valid subnets, that is `172.16.7.72/29`, `172.16.7.80/28`, `172.16.7.96/27` and `172.16.7.128/25`.

Note that in LAN A, `172.16.4.255` and `172.16.5.0` are normal host addresses. They only look strange because the subnet is larger than one octet.

### Example 3: Summarising in OSPF on Cisco IOS

R1 is a branch router. Its four LANs are `10.20.16.0/24` to `10.20.19.0/24`. It sits between area 1 (the branch) and area 0 (the backbone), which makes it an ABR (Area Border Router). OSPF allows inter-area summaries only at this boundary. See [OSPF](../05-routing/06-ospf.md).

```text
 10.20.16.0/24 --+
 10.20.17.0/24 --+               10.20.0.0/30
 10.20.18.0/24 --+-- [ R1 ] .1 ---------------- .2 [ R2 ] --> core
 10.20.19.0/24 --+   ABR
                     Area 1  |      Area 0
```

Without a summary, R2 learns four routes. With one, it learns a single route.

```text
! Cisco IOS on R1
router ospf 1
 network 10.20.0.0 0.0.0.3 area 0
 network 10.20.16.0 0.0.3.255 area 1
 area 1 range 10.20.16.0 255.255.252.0
```

The value `0.0.3.255` is a wildcard mask: the mask with each bit flipped. `area 1 range` tells R1 to send one summary into area 0.

Check R2:

```text
R2# show ip route ospf
      10.0.0.0/8 is variably subnetted, 3 subnets, 3 masks
O IA     10.20.16.0/22 [110/2] via 10.20.0.1, 00:03:05,
                       GigabitEthernet0/0
```

`O IA` means OSPF inter-area. The four `/24` routes are gone. "Variably subnetted" is how IOS says this network has more than one mask length. The three masks here are the `/22` summary, the `/30` link and the `/32` local address on R2.

Check R1. It built a discard route by itself:

```text
R1# show ip route | include Null0
O        10.20.16.0/22 is a summary, 00:03:12, Null0
```

`Null0` is a virtual interface that drops every packet sent to it. Going Deeper explains why this route saves you from a loop.

A Linux or Windows machine can use a summary as well. Here it sits on the same `/30` link in R2's place, at `10.20.0.2`:

```bash
# Linux
sudo ip route add 10.20.16.0/22 via 10.20.0.1
ip route show 10.20.16.0/22
# 10.20.16.0/22 via 10.20.0.1 dev eth0
```

```powershell
# Windows (run as Administrator)
New-NetRoute -DestinationPrefix "10.20.16.0/22" -InterfaceAlias "Ethernet" -NextHop 10.20.0.1
Get-NetRoute -DestinationPrefix "10.20.16.0/22" -PolicyStore ActiveStore
# ifIndex DestinationPrefix  NextHop    RouteMetric ifMetric PolicyStore
# ------- -----------------  -------    ----------- -------- -----------
# 12      10.20.16.0/22      10.20.0.1  256         25       ActiveStore
```

### Example 4: An ISP Aggregates Customer Routes

An ISP holds `203.0.113.0/24`. It gives each of four customers a `/26`. Its upstream provider does not need four routes.

```text
Customer A 203.0.113.0/26   --+
Customer B 203.0.113.64/26  --+--> [ ISP ] --> upstream sees one route:
Customer C 203.0.113.128/26 --+    AS 64512    203.0.113.0/24
Customer D 203.0.113.192/26 --+
```

In binary, the last octets are `00000000` and `11000000`. They differ at the first bit, so the prefix is 24 + 0 = 24.

```text
! Cisco IOS on the ISP router
router bgp 64512
 aggregate-address 203.0.113.0 255.255.255.0 summary-only
```

`summary-only` sends the aggregate and hides the `/26` routes. Another common method is to `network` the `/24` in BGP and add a static route to `Null0`. See [BGP](../05-routing/09-bgp.md).

If Customer C moves to another ISP but keeps its `/26`, that new ISP announces the `/26`. Longest match then pulls the traffic to the new ISP. The table has gained a route, because the customer block no longer fits inside its provider's summary. This is why moving addresses between providers hurts aggregation.

### Example 5: Check a Plan With a Script

Hand checks are good for interviews. For a real plan, let a tool check it. This uses Python's standard `ipaddress` module, so it runs the same on Linux and Windows. Save it as `check_plan.py`:

```text
import ipaddress as ip
from itertools import combinations

plan = [
    "192.168.10.0/25", "192.168.10.128/26", "192.168.10.192/27",
    "192.168.10.224/28", "192.168.10.240/30", "192.168.10.244/30",
    "192.168.10.248/30", "192.168.10.252/30",
    "192.168.10.64/26",   # a mistake, added on purpose
]
nets = [ip.ip_network(p) for p in plan]

for a, b in combinations(nets, 2):
    if a.overlaps(b):
        print("OVERLAP:", a, "and", b)

print("Summary:", *ip.collapse_addresses(nets))
```

```bash
# Linux
python3 check_plan.py
```

```powershell
# Windows
python check_plan.py
```

Output on both:

```text
OVERLAP: 192.168.10.0/25 and 192.168.10.64/26
Summary: 192.168.10.0/24
```

`collapse_addresses` joins everything it can. It also shows that the plan fills the `/24` exactly.

## Going Deeper

### VLSM Needs a Classless Routing Protocol

A routing protocol must carry the mask in each update, or a router cannot tell a `/26` from a `/30` under the same major network.

- **Classful (no mask in updates):** RIPv1, and Cisco's old IGRP. These cannot support VLSM.
- **Classless (mask in updates):** RIPv2, OSPF, EIGRP, IS-IS and BGP-4. These support VLSM.

Static routes include a mask, so they work with any design.

### Auto-Summary and Discontiguous Networks

Auto-summary means a protocol shrinks routes down to the old classful boundary. That fails with a **discontiguous network**, which means parts of one major network separated by a different one.

```text
10.1.1.0/24 --[ R1 ]---192.168.5.0/30---[ R2 ]-- 10.1.2.0/24
```

With auto-summary, R1 and R2 both advertise `10.0.0.0/8` across the link. Each router sees the other's route as "the same" `10.0.0.0/8` and cannot reach the far LAN properly.

On Cisco IOS, EIGRP has had auto-summary off by default since IOS 15.0. Defaults vary by protocol and version, so check `show ip protocols`, and use `no auto-summary` where needed. OSPF never does this.

### Where Each Protocol Can Summarise

| Protocol | Where | Cisco IOS command |
|---|---|---|
| OSPF, between areas | ABR only | `area 1 range 10.20.16.0 255.255.252.0` |
| OSPF, external routes | ASBR only | `summary-address 10.20.16.0 255.255.252.0` |
| EIGRP | Any interface | `ip summary-address eigrp 100 10.20.16.0 255.255.252.0` |
| RIPv2 | Any interface | `ip summary-address rip 10.20.16.0 255.255.252.0` |
| IS-IS | L1/L2 boundary | `summary-address 10.20.16.0 255.255.252.0` under `router isis` |
| BGP | Any BGP router | `aggregate-address 10.20.16.0 255.255.252.0` |

An ASBR (Autonomous System Boundary Router) is a router that brings routes from outside OSPF into OSPF. OSPF cannot summarise inside an area, because every router in an area must hold the same link-state database.

Commands differ on other vendors, and EIGRP named mode has its own syntax.

### The Discard Route and Routing Loops

Here is the danger. R2 has the summary `10.20.16.0/22` pointing at R1. R1 has a default route pointing at R2. No LAN exists for `10.20.18.0/24` right now.

```text
Packet to 10.20.18.99, with no discard route on R1:

R2: matches 10.20.16.0/22       ---> send to R1
R1: no /24 for it, uses default ---> send to R2
R2: matches 10.20.16.0/22       ---> send to R1
... repeats until the TTL reaches 0
```

TTL (Time To Live) is a counter in each packet. Every router lowers it by 1, and the packet dies at 0. See [IPv4 Header and Fragmentation](10-ipv4-header-and-fragmentation.md). A single bad packet is small, but a flood of them can fill the link.

The fix is a **discard route** on R1 for the summary:

```text
R1 routing table:
10.20.16.0/24    connected   (longer match, used first)
10.20.17.0/24    connected
10.20.19.0/24    connected
10.20.16.0/22    Null0       (catches what is left)
0.0.0.0/0        R2          (never used for this range)
```

Real networks still match their `/24`. The packet for the missing `.18` network falls to `Null0` and dies at once.

OSPF, EIGRP and BGP's `aggregate-address` create this route for you on Cisco IOS. For a hand-made summary, add it yourself:

```text
! Cisco IOS
ip route 10.20.16.0 255.255.252.0 Null0
```

```bash
# Linux
sudo ip route add blackhole 10.20.16.0/22
```

### When Summaries Hurt

**Black holes with two exits.** A black hole is a place where traffic disappears with no warning. Suppose R1 and R3 both advertise `10.20.16.0/22`, and R1 loses `10.20.18.0/24`. R1 still advertises the summary, because at least one part of it is alive. Traffic for `.18` that reaches R1 is dropped, though R3 could deliver it.

A summary only disappears when all of its parts are gone. This is good for stability, but it hides partial failures. Fixes include a link between R1 and R3, or more specific routes between them.

**Suboptimal paths.** A far-away router sees one route for the whole group, so it cannot choose the closest exit for each LAN.

**Loss of detail in BGP.** An aggregate hides the paths of the routes inside it. See [BGP](../05-routing/09-bgp.md).

### Deaggregation and Route Hijacks

Longest match is also a tool, and a weapon.

Some companies split a `/22` into `/24` announcements on purpose, to steer traffic between providers. This is called deaggregation. It gives more control but adds routes to every Internet router.

Most operators reject IPv4 routes longer than `/24` from the global table. This is common practice, not a standard. So a `/24` is the smallest block you can normally announce.

In 2008, Pakistan Telecom announced a `/24` from inside YouTube's `/22`. The longer prefix won across much of the Internet, and YouTube was unreachable for hours. The same longest-match rule that makes summaries safe made this outage possible. Tools such as RPKI (a way to prove who may announce a prefix) now reduce this risk.

### /31 and /32 Subnets

Each `/30` link uses 4 addresses for 2 devices. RFC 3021 allows `/31` on point-to-point links, where the 2 addresses are both usable, with no network or broadcast. A point-to-point link is one that joins exactly two devices.

In Example 1, three `/31` links would use 6 addresses instead of 12. Most modern routers support this, but check old gear.

A `/32` is one address. Routers use it for loopback interfaces. A loopback is a virtual interface that stays up while the router is up.

### Building a Plan That Summarises Well

Good plans follow the shape of the network, so each site or region falls inside one block.

```text
10.20.0.0/16   the company
|-- 10.20.0.0/20      links and shared services
|-- 10.20.16.0/21     Region North
|   |-- 10.20.16.0/22     Branch A
|   `-- 10.20.20.0/22     Branch B
`-- 10.20.24.0/21     Region South (reserved)
```

The core needs one route per region. If Branch A grows, the region's block can stay the same.

Do not assign networks in the order people ask for them. That scatters each site's addresses and ruins the summaries.

Remember what your platform takes for itself. In AWS, every subnet loses 5 addresses, not 2. See [AWS VPC basics](../11-cloud-and-containers/03-aws-vpc-basics.md).

### The Same Ideas in IPv6

IPv6 has so many addresses that nobody saves hosts inside a LAN. Most LANs are a `/64`. See [IPv6 Basics](11-ipv6-basics.md).

Summarisation still matters very much. A site might get `2001:db8:10::/48`. Networks `2001:db8:10:100::/64` to `2001:db8:10:1ff::/64` all share their first 56 bits, so they summarise to `2001:db8:10:100::/56`. Plans often stay on 4-bit boundaries, so each hex digit keeps its meaning.

Router links may use `/127`, as RFC 6164 describes.

### Subnet Zero: A Note on History

Old guidance (RFC 950) avoided the first and last subnet of a block. In our design, that would have banned `192.168.10.0/25`. This is out of date. Cisco IOS has allowed subnet zero by default since version 12.0, and modern designs use every subnet.

## Common Mistakes

- **Forgetting the router.** "100 hosts" needs room for the default gateway too.
- **Forgetting the link subnets.** Every router-to-router link needs its own subnet.
- **Sizing with no growth.** A `/27` for exactly 30 devices has no room for one more.
- **Off by two at the limit.** 62 hosts fit a `/26`. 63 hosts do not. A block of 64 holds 62 usable hosts.
- **Allocating small subnets first, moving forward only.** You skip addresses and think you ran out.
- **Starting a subnet at a bad address.** The network address must be a multiple of the block size.
- **Calling any neighbours summarisable.** `.13.0/24` and `.14.0/24` are adjacent, but they do not form a `/23`.
- **Not checking what a summary also covers.** It may include networks you do not own or that live elsewhere.
- **Thinking summarisation changes subnets.** It changes what routers advertise. Hosts and their masks stay the same.
- **Thinking routers catch every overlap.** IOS only checks its own interfaces.
- **Thinking summaries need to be perfect to work.** They work without being exact, but extra coverage can draw traffic to the wrong place.
- **Removing the `Null0` route as "junk".** It is there to stop loops.
- **Comparing only the highest network address when sizes differ.** Use its last address.

## Interview Questions

**Q: What is VLSM, and why do we use it?**
A: VLSM means Variable Length Subnet Mask. It lets one address block hold subnets of different sizes. A 100-host LAN gets a /25 and a router link gets a /30. It saves addresses and fits subnets to real needs.

**Q: Why do we allocate the largest subnet first?**
A: A subnet must start at a multiple of its block size. If you go from large to small, the next free address is always a valid start, so you skip nothing. Small first can leave gaps that are easy to lose track of.

**Q: Design subnets from 192.168.1.0/24 for 60, 28 and 12 hosts and one router link.**
A: 60 hosts need a /26, so 192.168.1.0/26. 28 hosts need a /27, so 192.168.1.64/27. 12 hosts need a /28, so 192.168.1.96/28. The link gets a /30, 192.168.1.112/30. Everything from .116 up is free.

**Q: Find the summary for 10.1.4.0/24 through 10.1.7.0/24.**
A: The third octet runs from 00000100 to 00000111. The first six bits match, so the prefix is 16 plus 6, which is /22. The summary is 10.1.4.0/22, mask 255.255.252.0, and it covers exactly those four networks.

**Q: Can 172.16.13.0/24 and 172.16.14.0/24 be joined into a /23?**
A: No. A /23 must start on an even third octet, and 13 is odd. In binary they differ at the seventh bit of that octet. The smallest single route covering both is 172.16.12.0/22, which also covers .12 and .15.

**Q: What is the difference between subnetting and supernetting?**
A: Subnetting makes the prefix longer, splitting one block into smaller ones. Supernetting makes the prefix shorter, joining blocks into a larger one. Strictly, a supernet is shorter than the old classful mask, but people often use it to mean any summary.

**Q: What are the benefits and risks of route summarisation?**
A: Benefits are smaller routing tables, fewer updates, and hiding local flaps. Risks are less accurate paths and black holes, since a summary can stay up while one network inside it is down. It can also draw traffic for addresses that do not exist.

**Q: Why does a router install a summary route to Null0?**
A: To prevent a loop. Without it, a packet for an unused address inside the summary could follow the default route back to the sender, then come back again until the TTL expires. Real subnets have longer routes, so they are not affected.

**Q: How do you check if two subnets overlap?**
A: Take the shorter prefix and apply it to both addresses. If you get the same network, they overlap. For 192.168.1.64/26 and 192.168.1.96/27, applying /26 to .96 gives .64, so the /27 sits inside the /26.

**Q: Does VLSM depend on the routing protocol?**
A: Yes. The protocol must carry the mask in its updates. RIPv2, OSPF, EIGRP, IS-IS and BGP do. RIPv1 and IGRP do not, so they cannot support VLSM. Static routes always include a mask.

**Q: Where can OSPF summarise, and why not everywhere?**
A: On an ABR between areas, and on an ASBR for external routes. Inside an area, every router must keep the same link-state database, so one router cannot hide details from its neighbours.

**Q: My VPN connects, but I cannot reach the office server at 192.168.1.20. What might be wrong?**
A: The home LAN may also be 192.168.1.0/24. The laptop then treats the server as a local address and never sends the traffic into the tunnel. The fix is to change the home range, or to renumber or use NAT at the office.

## Quick Recap

- VLSM gives each network a subnet that fits, in place of one size for all. Fixed masks waste addresses or simply do not fit.
- To design: list everything, add 2 to hosts, round up to a power of 2, sort largest first, allocate in order, and record it.
- Summarisation joins many routes into one with a shorter prefix. Aggregation is the same idea, and a strict supernet is shorter than the classful mask.
- To compute a summary, compare the lowest and highest addresses in binary, and count the matching bits from the left.
- Always check what a summary covers beyond what you meant. Use a discard route to `Null0` to stop loops.
- Overlapping subnets are a bug that routers may not catch. Overlapping routes are normal, and the longest match wins.
- Summaries are clean only if the address plan follows the network's shape, so design addresses and routing together.
