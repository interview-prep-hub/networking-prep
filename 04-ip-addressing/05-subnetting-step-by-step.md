# Subnetting Step by Step

**Level:** Intermediate | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [IPv4 Addressing Basics](01-ipv4-basics.md), [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md)

## In One Line

Subnetting cuts one big network into smaller equal pieces, and the block size method lets you find any piece in seconds with only subtraction and multiples, with no binary on paper.

## Why It Matters

A single flat network with thousands of devices has real problems:

- **Too much broadcast traffic.** A broadcast is a message sent to every device in the network. More devices means more broadcasts, and every device must look at each one. See [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).
- **Weak security.** If Guest Wi-Fi and the finance servers share one network, nothing stands between them. Traffic between subnets must pass through a router (a device that moves traffic between networks) or a firewall (a device that allows or blocks traffic), and that is where you can filter it.
- **Wasted addresses.** A team of 20 people does not need 65,534 addresses.

Interviewers ask about subnetting because it tests three things at once: do you understand addresses, can you think under pressure, and can you plan a real network? A typical question is: "What are the network, broadcast and usable range of 172.16.37.200/20?" A strong candidate answers in under 30 seconds, out loud, with no calculator. This chapter teaches you to do that.

## Simple Explanation

Imagine a street with house numbers 0 to 255. The city wants to split the street into equal blocks of 64 houses.

```text
 Block 1: houses   0 to  63
 Block 2: houses  64 to 127
 Block 3: houses 128 to 191
 Block 4: houses 192 to 255
```

Each block has two special numbers:

- The **first number** is the sign that names the block. Nobody lives there. This is the **network address**.
- The **last number** is the notice board that speaks to everyone in the block. Nobody lives there either. This is the **broadcast address**.

Everyone else lives in between. Those are the **usable host addresses**.

Now someone asks: "Which block is house 75 in?" You think: multiples of 64 are 0, 64, 128, 192. The biggest one that is not above 75 is 64. So house 75 is in the block that starts at 64 and ends at 127. That is the whole method. The rest of the chapter only adds detail.

**Where the analogy stops being true:**

- A real street can have blocks of any size. Subnets can only be a power of two in size: 2, 4, 8, 16, 32, 64 and so on. This is because addresses are binary.
- A subnet must start at a multiple of its own size. A block of 64 cannot start at 50.
- A real subnet can span many "streets". A /20 covers 16 whole values of the third octet, which is 4,096 addresses.

## How It Works

### Words You Need First

| Term | Plain meaning | Example |
|---|---|---|
| Bit | A single 0 or 1 | `1` |
| Octet | One of the four numbers in an IPv4 address, 8 bits, value 0-255 | In 192.168.10.75, the third octet is 10 |
| Host | Any device with an IP address: PC, phone, server, router interface | A laptop |
| Subnet | A smaller network cut out of a bigger one | 192.168.10.64/26 |
| Prefix length | How many bits, counted from the left, are the network part. Written with a slash | /26 |
| Subnet mask | The same information as the prefix, written like an address | 255.255.255.192 |
| Network bits | The left part of the address. They name the subnet | First 26 bits in a /26 |
| Host bits | The right part of the address. They name a device inside the subnet | Last 6 bits in a /26 |
| Network address | The first address in a subnet. All host bits are 0. Not given to a device | 192.168.10.64 |
| Broadcast address | The last address in a subnet. All host bits are 1. Not given to a device | 192.168.10.127 |
| Usable range | Everything between those two | .65 to .126 |
| Interesting octet | The octet that holds the last network bit. All the work happens here | Fourth octet for /26 |
| Block size (magic number) | The distance between one subnet and the next, counted in the interesting octet | 64 for /26 |

### What the Block Size Really Is

One octet can hold 256 values (0 to 255). The mask decides how many of its bits are fixed and how many are free.

Take a mask octet of 192. In binary it is `11000000`. Two bits are network bits. Six bits are host bits.

Six free bits can count 2^6 = 64 values. So the network number changes every 64 steps. That step is the block size.

You do not need binary to get it. Use this formula:

```text
Block size = 256 - (mask value in the interesting octet)
```

Check: 256 - 192 = 64. It matches.

Why does this trick work? The bit values in an octet are:

```text
 128   64   32   16    8    4    2    1
```

Mask 192 is 128 + 64. The bits left over are 32 + 16 + 8 + 4 + 2 + 1 = 63. Add 1 and you get 64, the value of the lowest network bit. All eight bits add up to 255, so 256 minus the mask gives the same result. The block size is always the value of the last network bit.

### The Eight Mask Values to Memorise

Only eight values can appear in a mask octet (nine with 0). Learn this short table and everything else follows.

| Network bits in the octet | Mask value | Block size |
|---|---|---|
| 1 | 128 | 128 |
| 2 | 192 | 64 |
| 3 | 224 | 32 |
| 4 | 240 | 16 |
| 5 | 248 | 8 |
| 6 | 252 | 4 |
| 7 | 254 | 2 |
| 8 | 255 | 1 |

Each mask value is the previous one plus the next bit value: 128, then +64 = 192, then +32 = 224, then +16 = 240, then +8 = 248, then +4 = 252, then +2 = 254, then +1 = 255.

Block sizes simply halve going down the table.

### The Full Prefix Table

"Block" is counted in the interesting octet. "Addresses" is the total size of the subnet.

| Prefix | Subnet mask | Interesting octet | Block | Addresses | Usable hosts |
|---|---|---|---|---|---|
| /8 | 255.0.0.0 | 1st | 1 | 16,777,216 | 16,777,214 |
| /9 | 255.128.0.0 | 2nd | 128 | 8,388,608 | 8,388,606 |
| /10 | 255.192.0.0 | 2nd | 64 | 4,194,304 | 4,194,302 |
| /11 | 255.224.0.0 | 2nd | 32 | 2,097,152 | 2,097,150 |
| /12 | 255.240.0.0 | 2nd | 16 | 1,048,576 | 1,048,574 |
| /13 | 255.248.0.0 | 2nd | 8 | 524,288 | 524,286 |
| /14 | 255.252.0.0 | 2nd | 4 | 262,144 | 262,142 |
| /15 | 255.254.0.0 | 2nd | 2 | 131,072 | 131,070 |
| /16 | 255.255.0.0 | 2nd | 1 | 65,536 | 65,534 |
| /17 | 255.255.128.0 | 3rd | 128 | 32,768 | 32,766 |
| /18 | 255.255.192.0 | 3rd | 64 | 16,384 | 16,382 |
| /19 | 255.255.224.0 | 3rd | 32 | 8,192 | 8,190 |
| /20 | 255.255.240.0 | 3rd | 16 | 4,096 | 4,094 |
| /21 | 255.255.248.0 | 3rd | 8 | 2,048 | 2,046 |
| /22 | 255.255.252.0 | 3rd | 4 | 1,024 | 1,022 |
| /23 | 255.255.254.0 | 3rd | 2 | 512 | 510 |
| /24 | 255.255.255.0 | 3rd | 1 | 256 | 254 |
| /25 | 255.255.255.128 | 4th | 128 | 128 | 126 |
| /26 | 255.255.255.192 | 4th | 64 | 64 | 62 |
| /27 | 255.255.255.224 | 4th | 32 | 32 | 30 |
| /28 | 255.255.255.240 | 4th | 16 | 16 | 14 |
| /29 | 255.255.255.248 | 4th | 8 | 8 | 6 |
| /30 | 255.255.255.252 | 4th | 4 | 4 | 2 |
| /31 | 255.255.255.254 | 4th | 2 | 2 | 2 (point-to-point links only) |
| /32 | 255.255.255.255 | 4th | 1 | 1 | 1 (a single address) |

Notice the pattern: the block column repeats 128, 64, 32, 16, 8, 4, 2, 1 in each octet. You do not need to memorise the whole table, only the pattern.

### The Six-Step Method for Any Address

This works for every address and every prefix from /8 to /30. We use **192.168.10.75/26** as the running example.

1. **Find the interesting octet.** It holds the last network bit.
   - /1 to /8: 1st octet
   - /9 to /16: 2nd octet
   - /17 to /24: 3rd octet
   - /25 to /32: 4th octet

   /26 is between /25 and /32, so it is the 4th octet. The address value there is 75.

2. **Find the block size.** /26 has mask 255.255.255.192. 256 - 192 = **64**.

3. **Find the network address.** List the multiples of the block size. Take the biggest one that is not above the address value. Multiples of 64: 0, **64**, 128, 192. Value 75 falls in the block starting at 64.
   - Octets to the left of the interesting octet: copy them unchanged.
   - Octets to the right: set them to 0.

   Network = **192.168.10.64**

4. **Find the broadcast address.** Find the next network, then step back by one. Next network = 64 + 64 = 128, so 192.168.10.128. One address before that is **192.168.10.127**. In the interesting octet this is network value + block - 1. Octets to the right become 255. In the last block the next network is 256, which no octet can hold, so use this formula: for 192.168.10.200/26 the network is .192 and the broadcast is 192 + 64 - 1 = .255.

5. **Find the first and last host.** First host = network + 1 = **192.168.10.65**. Last host = broadcast - 1 = **192.168.10.126**.

6. **Count the hosts.** Host bits = 32 - 26 = 6. Usable hosts = 2^6 - 2 = **62**.

```text
 One /26 block: 64 addresses in total

 .64      .65                   .126     .127
 +--------+---------------------+--------+
 | network|  62 usable hosts    | bcast  |
 +--------+---------------------+--------+
    ^            ^                  ^
    |            |                  +-- all host bits are 1
    |            +-- .75 lives here
    +-- all host bits are 0
```

And here is the same /24 as a number line:

```text
 192.168.10.0/24 cut into four /26 blocks

 .0          .64         .128        .192        (next /24)
 |-----------|-----------|-----------|-----------|
 |  subnet 1 |  subnet 2 |  subnet 3 |  subnet 4 |
                  ^
                 .75
```

**Why subtract 2 in step 6?** A subnet of 64 addresses loses the network address and the broadcast address. The rule is: usable hosts = 2^(host bits) - 2.

**The fuller truth:** the "-2" rule has two exceptions, /31 and /32. Both are covered in Going Deeper. Also, one usable address normally goes to the router (the default gateway), so real devices get one less. See [The Default Gateway](08-default-gateway.md).

### How to Handle /8, /16 and /24

These look strange because the mask value in the interesting octet is 255, and 256 - 255 = 1. A block size of 1 means every value is a multiple. So the interesting octet stays the same, and everything to its right is the host part.

Example: 10.20.30.40/24. Interesting octet is the 3rd. Block is 1, so 30 stays 30.

```text
 Network:   10.20.30.0
 Broadcast: 10.20.30.255
```

Example: 172.16.5.9/16. Interesting octet is the 2nd. Block is 1, so 16 stays 16.

```text
 Network:   172.16.0.0
 Broadcast: 172.16.255.255
```

Example: 10.20.30.40/8. Interesting octet is the 1st. Block is 1, so 10 stays 10.

```text
 Network:   10.0.0.0
 Broadcast: 10.255.255.255
```

### Why the Method Works: the Binary Truth

A device does not use block sizes. It uses two bit operations.

- **AND** (the result is 1 only when both bits are 1). Address AND mask gives the network address.
- **OR** (the result is 1 when either bit is 1). Network OR inverted mask gives the broadcast address.

Here is the fourth octet of 192.168.10.75/26:

```text
 Address .75        0 1 0 0 1 0 1 1
 Mask    .192       1 1 0 0 0 0 0 0
                    --- -----------
                    net  host bits

 AND     = network  0 1 0 0 0 0 0 0  = 64
 Inverted mask      0 0 1 1 1 1 1 1  = 63
 OR      = bcast    0 1 1 1 1 1 1 1  = 127
```

Setting the host bits to 0 rounds the value down to a multiple of 64. That is exactly what "biggest multiple not above the value" does. The block method is the same calculation, done in decimal.

### Splitting a Network into N Subnets

Here you start with one network and need a certain number of pieces. You take bits from the host part and give them to the network part. This is called **borrowing bits**.

1. Find the smallest number of borrowed bits **s** where 2^s is at least N.
2. New prefix = old prefix + s.
3. Find the block size of the new prefix.
4. Count up in block-size steps from the original network address.

**Example:** split 192.168.1.0/24 into 6 subnets.

1. 2^2 = 4 is too small. 2^3 = 8 is enough. So s = 3.
2. New prefix = 24 + 3 = /27.
3. Mask = 255.255.255.224. Block = 256 - 224 = 32.
4. Subnets start at 0, 32, 64, 96, 128, 160, 192, 224.

You get 8 subnets, more than you asked for. You cannot get exactly 6 equal subnets, because the count is always a power of two. The spare two stay free for later.

```text
 Original /24:   nnnnnnnn.nnnnnnnn.nnnnnnnn.hhhhhhhh

 After /27:      nnnnnnnn.nnnnnnnn.nnnnnnnn.ssshhhhh
                                            ^^^
                                            3 borrowed bits: 8 subnets
                                               ^^^^^
                                               5 host bits: 30 hosts
```

### Splitting a Network by Hosts per Subnet

Here you know how many devices each subnet must hold.

1. Find the smallest number of host bits **h** where 2^h - 2 is at least the host count.
2. New prefix = 32 - h.
3. Number of subnets = 2^(new prefix - old prefix).
4. Count up in block-size steps.

**Example:** from 192.168.1.0/24, make subnets that hold 25 hosts each.

1. 2^4 - 2 = 14 is too small. 2^5 - 2 = 30 is enough. So h = 5.
2. New prefix = 32 - 5 = /27.
3. Number of subnets = 2^(27 - 24) = 8.
4. Same list as before: block 32.

Use this quick lookup for h:

| Hosts needed (max) | Host bits h | Prefix |
|---|---|---|
| 2 | 2 | /30 |
| 6 | 3 | /29 |
| 14 | 4 | /28 |
| 30 | 5 | /27 |
| 62 | 6 | /26 |
| 126 | 7 | /25 |
| 254 | 8 | /24 |
| 510 | 9 | /23 |
| 1,022 | 10 | /22 |
| 2,046 | 11 | /21 |
| 4,094 | 12 | /20 |
| 65,534 | 16 | /16 |

### When You Need Both a Subnet Count and a Host Count

Sometimes the question gives both. Add the bits together and check that they fit.

```text
 s + h must be at most (32 - original prefix)
```

- 6 subnets of 25 hosts from a /24: s = 3, h = 5, total 8. A /24 has 8 host bits. It fits exactly.
- 10 subnets of 20 hosts from a /24: s = 4, h = 5, total 9. Only 8 bits exist. It does not fit. Ten blocks of 32 would need 320 addresses, and a /24 only has 256. You need a bigger network.

If subnets need different sizes, use [VLSM](07-vlsm-and-supernetting.md), which lets you mix sizes inside one network.

### Finding the Nth Subnet Without Listing Them All

When there are thousands of subnets, you cannot write them all. Use counting.

First, how many subnets fit in one value of the octet to the left? That is 256 / block.

Say we have 10.0.0.0/8 cut into /18s. Block is 64 in the 3rd octet. So 256 / 64 = 4 subnets fit in each value of the 2nd octet.

Count subnets from 0. To find subnet number 701:

```text
 701 / 4 = 175, remainder 1

 2nd octet = 175
 3rd octet = remainder x block = 1 x 64 = 64

 Subnet 701 = 10.175.64.0/18
```

If someone says "the 702nd subnet" and counts from 1, that is number 701 when counting from 0. Always say which way you are counting.

### Same Subnet or Different Subnet?

Two devices talk directly only if they are in the same subnet. Otherwise traffic goes through a router. To test, find the network address of each one. If they match, it is the same subnet.

Are 192.168.1.60/26 and 192.168.1.70/26 in the same subnet?

- Block = 64.
- 60 falls in the block starting at 0. Network is 192.168.1.0.
- 70 falls in the block starting at 64. Network is 192.168.1.64.

They are different subnets, even though the addresses are only 10 apart. They need a router.

### Reading a Range Backwards

Interviewers sometimes give a range and ask for the prefix.

**Question:** what prefix covers exactly 10.1.8.0 to 10.1.11.255?

1. The 3rd octet changes, from 8 to 11. That is 4 values, so block = 4.
2. Mask value = 256 - 4 = 252.
3. Mask = 255.255.252.0, which is /22.
4. Check alignment: the start value 8 must be a multiple of 4. It is.

So the answer is 10.1.8.0/22.

Alignment matters. **Is 10.1.6.0/22 a network address?** No. Multiples of 4 are 0, 4, 8. So 6 falls in the block starting at 4. The network is 10.1.4.0/22, and 10.1.6.0 is an ordinary host in it.

### The Fast Mental Method for Interviews

Use these shortcuts when speaking.

**1. Get the block size directly from the prefix.** Take the distance from the prefix up to the next multiple of 8. If the prefix is already a multiple of 8, the distance is 0. Block = 2 to that power.

| Prefix | Next multiple of 8 | Distance | Block |
|---|---|---|---|
| /26 | 32 | 6 | 2^6 = 64 |
| /20 | 24 | 4 | 2^4 = 16 |
| /11 | 16 | 5 | 2^5 = 32 |
| /29 | 32 | 3 | 2^3 = 8 |
| /24 | 24 | 0 | 2^0 = 1 |

This is faster than converting to a mask first. It also tells you the octet: next multiple 32 means 4th octet, 24 means 3rd, 16 means 2nd, 8 means 1st.

**2. Round down by dividing.** Value divided by block, drop the remainder, multiply back.

```text
 91 with block 32:  91 / 32 = 2 (remainder dropped)
                    2 x 32  = 64
```

**3. Get the broadcast by adding block - 1.** Network 64, block 32: 64 + 31 = 95. Octets to the right become 255.

**4. Get host counts from powers of two.** Learn these five anchors and work from them.

```text
 2^8  =        256
 2^10 =      1,024
 2^16 =     65,536
 2^20 =  1,048,576
 2^24 = 16,777,216
```

Host bits 12? That is 2^10 x 4 = 4,096, then minus 2 gives 4,094.

**5. Use the halving rule.** Adding 1 to the prefix halves the addresses. Removing 1 doubles them. If a /24 has 256 addresses, a /23 has 512 and a /25 has 128.

**6. Count subnets by subtracting prefixes.** How many /27s fit in a /24? 27 - 24 = 3, 2^3 = 8. How many /24s fit in a /16? 2^8 = 256.

**7. Say it out loud in a fixed order.** Use the same sentence every time:

> "172.16.37.200 slash 20. The interesting octet is the third. 24 minus 20 is 4, so the block is 16. Multiples of 16 near 37 are 32 and 48. The network is 172.16.32.0, the next network is 172.16.48.0, so the broadcast is 172.16.47.255. First host 172.16.32.1, last host 172.16.47.254. 12 host bits, so 4,094 hosts."

A fixed order keeps you calm, and it shows the interviewer how you think.

## Example

### Example 1: /24 Space

#### 1a. Find everything for 192.168.5.130/28

- Interesting octet: 4th, value 130.
- Block: next multiple of 8 is 32, distance 4, 2^4 = 16. Mask 255.255.255.240.
- Multiples of 16 near 130: 128, 144. Value 130 falls in the block starting at 128.

| Item | Result |
|---|---|
| Network | 192.168.5.128 |
| First host | 192.168.5.129 |
| Last host | 192.168.5.142 |
| Broadcast | 192.168.5.143 |
| Usable hosts | 2^4 - 2 = 14 |

#### 1b. Split 192.168.1.0/24 into 6 subnets

As calculated above, the answer is /27, mask 255.255.255.224, block 32, 8 subnets, 30 hosts each.

| # | Network | First host | Last host | Broadcast |
|---|---|---|---|---|
| 0 | 192.168.1.0 | 192.168.1.1 | 192.168.1.30 | 192.168.1.31 |
| 1 | 192.168.1.32 | 192.168.1.33 | 192.168.1.62 | 192.168.1.63 |
| 2 | 192.168.1.64 | 192.168.1.65 | 192.168.1.94 | 192.168.1.95 |
| 3 | 192.168.1.96 | 192.168.1.97 | 192.168.1.126 | 192.168.1.127 |
| 4 | 192.168.1.128 | 192.168.1.129 | 192.168.1.158 | 192.168.1.159 |
| 5 | 192.168.1.160 | 192.168.1.161 | 192.168.1.190 | 192.168.1.191 |
| 6 | 192.168.1.192 | 192.168.1.193 | 192.168.1.222 | 192.168.1.223 |
| 7 | 192.168.1.224 | 192.168.1.225 | 192.168.1.254 | 192.168.1.255 |

Check the pattern: every broadcast is one less than the next network, and the last broadcast is .255. If your table does not end at .255, something is wrong.

### Example 2: /16 Space

#### 2a. Find everything for 172.16.37.200/20

- Interesting octet: 3rd, value 37.
- Block: 24 - 20 = 4, 2^4 = 16. Mask 255.255.240.0.
- Multiples of 16 near 37: 32, 48. Value 37 falls in the block starting at 32.
- The 4th octet is entirely host bits. So its value, 200, does not affect the network.

| Item | Result |
|---|---|
| Network | 172.16.32.0 |
| First host | 172.16.32.1 |
| Last host | 172.16.47.254 |
| Broadcast | 172.16.47.255 |
| Usable hosts | 2^12 - 2 = 4,094 |

See how the host range runs across many values of the third octet:

```text
 172.16.32.0    network
 172.16.32.1    first host
 172.16.32.2
 ...
 172.16.32.255  a normal host, not a broadcast
 172.16.33.0    a normal host, not a network
 ...
 172.16.47.254  last host
 172.16.47.255  broadcast
 ---------------------------------
 172.16.48.0    start of the next subnet
```

Addresses ending in .0 and .255 can be valid hosts in subnets bigger than /24. Candidates often get this wrong.

#### 2b. Split 172.16.0.0/16 into 30 subnets

1. 2^4 = 16 is too small. 2^5 = 32 is enough. s = 5.
2. New prefix = 16 + 5 = /21.
3. Mask = 255.255.248.0. Block = 256 - 248 = 8, in the 3rd octet.
4. There are 32 subnets, each with 2^11 - 2 = 2,046 usable hosts.

| # | Network | First host | Last host | Broadcast |
|---|---|---|---|---|
| 0 | 172.16.0.0/21 | 172.16.0.1 | 172.16.7.254 | 172.16.7.255 |
| 1 | 172.16.8.0/21 | 172.16.8.1 | 172.16.15.254 | 172.16.15.255 |
| 2 | 172.16.16.0/21 | 172.16.16.1 | 172.16.23.254 | 172.16.23.255 |
| ... | ... | ... | ... | ... |
| 31 | 172.16.248.0/21 | 172.16.248.1 | 172.16.255.254 | 172.16.255.255 |

#### 2c. From 172.20.0.0/16, make subnets of 500 hosts each

1. 2^8 - 2 = 254 is too small. 2^9 - 2 = 510 is enough. h = 9.
2. New prefix = 32 - 9 = /23.
3. Number of subnets = 2^(23 - 16) = 128.
4. Mask = 255.255.254.0. Block = 2, in the 3rd octet.

| # | Network | First host | Last host | Broadcast |
|---|---|---|---|---|
| 0 | 172.20.0.0/23 | 172.20.0.1 | 172.20.1.254 | 172.20.1.255 |
| 1 | 172.20.2.0/23 | 172.20.2.1 | 172.20.3.254 | 172.20.3.255 |
| ... | ... | ... | ... | ... |
| 127 | 172.20.254.0/23 | 172.20.254.1 | 172.20.255.254 | 172.20.255.255 |

#### 2d. A split that crosses an octet: 172.31.0.0/16 with 100 hosts each

1. 2^6 - 2 = 62 is too small. 2^7 - 2 = 126 is enough. h = 7.
2. New prefix = 32 - 7 = /25.
3. Number of subnets = 2^(25 - 16) = 512.
4. Mask = 255.255.255.128. Block = 128, in the 4th octet.

The subnets go: 172.31.0.0, 172.31.0.128, 172.31.1.0, 172.31.1.128, 172.31.2.0, and so on. Two subnets fit in each value of the 3rd octet.

Find subnet 299 (counting from 0):

```text
 299 / 2 = 149, remainder 1

 3rd octet = 149
 4th octet = 1 x 128 = 128

 Subnet 299 = 172.31.149.128/25
 Hosts: 172.31.149.129 to 172.31.149.254
 Broadcast: 172.31.149.255
```

### Example 3: /8 Space

#### 3a. Find everything for 10.91.130.5/11

- Interesting octet: 2nd, value 91.
- Block: 16 - 11 = 5, 2^5 = 32. Mask 255.224.0.0.
- 91 / 32 = 2, and 2 x 32 = 64. Next multiple is 96.
- Host bits: 32 - 11 = 21.

| Item | Result |
|---|---|
| Network | 10.64.0.0 |
| First host | 10.64.0.1 |
| Last host | 10.95.255.254 |
| Broadcast | 10.95.255.255 |
| Usable hosts | 2^21 - 2 = 2,097,150 |

#### 3b. Find everything for 10.200.77.9/18

- Interesting octet: 3rd, value 77.
- Block: 24 - 18 = 6, 2^6 = 64. Mask 255.255.192.0.
- Multiples of 64 near 77: 64, 128.
- The 2nd octet, 200, is to the left of the interesting octet, so it is copied unchanged.

| Item | Result |
|---|---|
| Network | 10.200.64.0 |
| First host | 10.200.64.1 |
| Last host | 10.200.127.254 |
| Broadcast | 10.200.127.255 |
| Usable hosts | 2^14 - 2 = 16,382 |

#### 3c. Split 10.0.0.0/8 into 16 subnets

1. 2^4 = 16. s = 4.
2. New prefix = 8 + 4 = /12.
3. Mask = 255.240.0.0. Block = 16, in the 2nd octet.
4. Each subnet has 2^20 - 2 = 1,048,574 usable hosts.

| # | Network | Broadcast |
|---|---|---|
| 0 | 10.0.0.0/12 | 10.15.255.255 |
| 1 | 10.16.0.0/12 | 10.31.255.255 |
| 2 | 10.32.0.0/12 | 10.47.255.255 |
| ... | ... | ... |
| 15 | 10.240.0.0/12 | 10.255.255.255 |

#### 3d. Split 10.0.0.0/8 into 1,000 subnets

1. 2^9 = 512 is too small. 2^10 = 1,024 is enough. s = 10.
2. New prefix = 8 + 10 = /18.
3. Mask = 255.255.192.0. Block = 64, in the 3rd octet.
4. There are 1,024 subnets, each with 2^14 - 2 = 16,382 usable hosts.

The list goes:

```text
 #0    10.0.0.0/18
 #1    10.0.64.0/18
 #2    10.0.128.0/18
 #3    10.0.192.0/18
 #4    10.1.0.0/18     (the 3rd octet rolled over, so the 2nd adds 1)
 #5    10.1.64.0/18
 ...
 #1023 10.255.192.0/18
```

Check the last one: 1023 / 4 = 255, remainder 3. 2nd octet is 255. 3rd octet is 3 x 64 = 192.

#### 3e. From 10.0.0.0/8, make subnets of 70,000 hosts each

1. 2^16 - 2 = 65,534 is too small. 2^17 - 2 = 131,070 is enough. h = 17.
2. New prefix = 32 - 17 = /15.
3. Number of subnets = 2^(15 - 8) = 128.
4. Mask = 255.254.0.0. Block = 2, in the 2nd octet.

The first subnet is 10.0.0.0/15, with hosts 10.0.0.1 to 10.1.255.254 and broadcast 10.1.255.255. The next starts at 10.2.0.0/15. The last is 10.254.0.0/15.

### Example 4: Trap Addresses

Interviewers like addresses that only look wrong, or only look right.

| Address | Block | Network | Broadcast | What it really is |
|---|---|---|---|---|
| 10.20.30.255/23 | 2 (3rd octet) | 10.20.30.0 | 10.20.31.255 | A valid host. It ends in .255 but is not the broadcast |
| 172.16.33.0/20 | 16 (3rd octet) | 172.16.32.0 | 172.16.47.255 | A valid host. It ends in .0 but is not the network |
| 10.1.2.3/30 | 4 (4th octet) | 10.1.2.0 | 10.1.2.3 | The broadcast. It cannot be given to a host |
| 192.168.1.64/26 | 64 (4th octet) | 192.168.1.64 | 192.168.1.127 | The network address. It cannot be given to a host |
| 192.168.1.64/25 | 128 (4th octet) | 192.168.1.0 | 192.168.1.127 | A valid host. Same address as above, different mask |

The last two rows teach the main lesson: **an address alone tells you nothing. You always need the mask.**

### Example 5: A Small Office, from Plan to Commands

**The task:** an office owns 192.168.50.0/24. It has four departments: Sales, Engineering, Guest and Servers. Each needs up to 50 devices.

**The plan:**

1. 50 devices + 1 router address = 51 addresses needed.
2. 2^5 - 2 = 30 is too small. 2^6 - 2 = 62 is enough. h = 6.
3. New prefix = /26. Mask 255.255.255.192. Block 64.
4. Number of subnets = 2^(26 - 24) = 4. That is exactly what we need.

| Department | Subnet | Router (gateway) | Hosts | Broadcast |
|---|---|---|---|---|
| Sales | 192.168.50.0/26 | 192.168.50.1 | .2 to .62 | .63 |
| Engineering | 192.168.50.64/26 | 192.168.50.65 | .66 to .126 | .127 |
| Guest | 192.168.50.128/26 | 192.168.50.129 | .130 to .190 | .191 |
| Servers | 192.168.50.192/26 | 192.168.50.193 | .194 to .254 | .255 |

Using the first usable address for the router is a common convention, not a rule. Some teams use the last one.

```text
 Sales                                       Engineering
 192.168.50.0/26                             192.168.50.64/26
     |                                           |
 [switch]                                    [switch]
     |                                           |
     | Gi0/0 .1                         Gi0/1 .65|
     +--------------+            +---------------+
                    |            |
                 +--+------------+--+
                 |      Router      |
                 +--+------------+--+
                    |            |
     +--------------+            +---------------+
     | Gi0/2 .129                      Gi0/3 .193|
     |                                           |
 [switch]                                    [switch]
     |                                           |
 Guest                                       Servers
 192.168.50.128/26                           192.168.50.192/26
```

**Cisco IOS: configure the router.**

```text
Router(config)# interface GigabitEthernet0/0
Router(config-if)# description Sales
Router(config-if)# ip address 192.168.50.1 255.255.255.192
Router(config-if)# no shutdown
Router(config-if)# interface GigabitEthernet0/1
Router(config-if)# description Engineering
Router(config-if)# ip address 192.168.50.65 255.255.255.192
Router(config-if)# no shutdown
Router(config-if)# interface GigabitEthernet0/2
Router(config-if)# description Guest
Router(config-if)# ip address 192.168.50.129 255.255.255.192
Router(config-if)# no shutdown
Router(config-if)# interface GigabitEthernet0/3
Router(config-if)# description Servers
Router(config-if)# ip address 192.168.50.193 255.255.255.192
Router(config-if)# no shutdown
Router(config-if)# end
```

**Cisco IOS: check the routes.** The router works out each network from the address and mask you gave it.

```text
Router# show ip route connected
      192.168.50.0/24 is variably subnetted, 8 subnets, 2 masks
C        192.168.50.0/26 is directly connected, GigabitEthernet0/0
L        192.168.50.1/32 is directly connected, GigabitEthernet0/0
C        192.168.50.64/26 is directly connected, GigabitEthernet0/1
L        192.168.50.65/32 is directly connected, GigabitEthernet0/1
C        192.168.50.128/26 is directly connected, GigabitEthernet0/2
L        192.168.50.129/32 is directly connected, GigabitEthernet0/2
C        192.168.50.192/26 is directly connected, GigabitEthernet0/3
L        192.168.50.193/32 is directly connected, GigabitEthernet0/3
```

`C` means a connected subnet. `L` means a local route: the router's own address, as a /32. Each interface gives one of each, which is why the output says 8 subnets.

**Linux: an Engineering host with address .75.** The `brd` value is the broadcast address.

```bash
$ ip -4 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.50.75/26 brd 192.168.50.127 scope global eth0
       valid_lft forever preferred_lft forever

$ ip route
default via 192.168.50.65 dev eth0
192.168.50.64/26 dev eth0 proto kernel scope link src 192.168.50.75
```

**Windows: the same host.**

```powershell
PS C:\> ipconfig

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 192.168.50.75
   Subnet Mask . . . . . . . . . . . : 255.255.255.192
   Default Gateway . . . . . . . . . : 192.168.50.65
```

### Check Your Answer with a Tool

Tools are for checking work, not for the interview.

**Linux:** `ipcalc` is available as a package on most distributions. This is the Debian and Ubuntu version. Other distributions ship a different `ipcalc` with different output.

```bash
$ ipcalc 192.168.50.75/26
Address:   192.168.50.75        11000000.10101000.00110010.01 001011
Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
Wildcard:  0.0.0.63             00000000.00000000.00000000.00 111111
=>
Network:   192.168.50.64/26     11000000.10101000.00110010.01 000000
HostMin:   192.168.50.65        11000000.10101000.00110010.01 000001
HostMax:   192.168.50.126       11000000.10101000.00110010.01 111110
Broadcast: 192.168.50.127       11000000.10101000.00110010.01 111111
Hosts/Net: 62                    Class C, Private Internet
```

The space inside the binary shows where the network bits end and the host bits begin.

**Linux or Windows with Python installed:** the standard `ipaddress` module works on both. On Windows, use `python` in place of `python3`.

```bash
$ python3 -c "import ipaddress as i; n=i.ip_interface('192.168.50.75/26').network; print(n, n.netmask, n.broadcast_address, n.num_addresses-2)"
192.168.50.64/26 255.255.255.192 192.168.50.127 62
```

```powershell
PS C:\> python -c "import ipaddress as i; [print(s) for s in i.ip_network('192.168.50.0/24').subnets(new_prefix=26)]"
192.168.50.0/26
192.168.50.64/26
192.168.50.128/26
192.168.50.192/26
```

More problems to try are in [Subnetting Practice Problems](06-subnetting-practice.md).

## Going Deeper

### /31 and /32: the Two Exceptions

The "-2" rule does not apply here.

- **/31** has 2 addresses and no spare ones. RFC 3021 allows /31 on point-to-point links, meaning a link with exactly two ends, such as between two routers. Both addresses are usable, with no network or broadcast address. On a link like 10.0.0.0/31, one router takes 10.0.0.0 and the other takes 10.0.0.1. Each router-to-router link then uses 2 addresses in place of the 4 of a /30, which saves a lot of space in large networks. Cisco IOS accepts /31 and warns if the interface is not point-to-point, such as Ethernet. Support varies, so check your platform.
- **/32** is one address. There is no subnet here at all. It is used for loopback interfaces (a virtual interface inside a router) and for routes that point to one exact host. The `L` routes in the example above are /32 routes.

### Subnet Zero and the All-Ones Subnet

Old study guides use 2^s - 2 for the number of subnets, which is wrong for modern networks. Here is the history.

Under the early rules in RFC 950, the first subnet (all subnet bits 0) and the last subnet (all subnet bits 1) were not to be used. The reason was to avoid confusion with the network and broadcast addresses of the whole network. Classless routing (routing by prefix length, not by the old address classes) removed this limit. On Cisco IOS, `ip subnet-zero` has been on by default since version 12.0.

Today the correct count is **2^s**. In the /27 example, 192.168.1.0/27 and 192.168.1.224/27 are both valid. If an exam or an old textbook says otherwise, tell the interviewer you know both, and that 2^s is current practice.

### Reserved Addresses in Cloud Subnets

Clouds take extra addresses out of every subnet, so the "-2" rule gives the wrong count there.

| Cloud | Reserved per subnet | Usable in a /24 |
|---|---|---|
| AWS VPC | 5 | 251 |
| Azure VNet | 5 | 251 |
| Google Cloud VPC | 4 (in a primary range) | 252 |

These are provider rules, not standards, so check current documentation. More in [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md). Interviews for cloud or DevOps roles often test this.

### Wildcard Masks

A wildcard mask is the inverse of a subnet mask. In the interesting octet, it equals block - 1.

```text
 Prefix /26
 Subnet mask    255.255.255.192
 Wildcard mask    0.  0.  0. 63     (block 64, minus 1)
```

Subtract each mask octet from 255 and you get the same result. Cisco access control lists (ACLs, rules that permit or deny traffic) and `network` statements in OSPF (a routing protocol) use wildcard masks. Example: `192.168.50.64 0.0.0.63` matches the whole Engineering subnet. See [Access Control Lists](../09-security/05-access-control-lists.md).

### Prefixes Shorter Than /8

The method still works. 192.0.0.0/6 has mask 252.0.0.0, block 4 in the 1st octet. Short prefixes like this are seen in route summaries (one route that stands for many smaller networks), not in LANs. See [VLSM, Route Summarisation and Supernetting](07-vlsm-and-supernetting.md).

### Detecting Overlapping Subnets

Two subnets overlap if one contains the other's network address. Since subnets are aligned to their own size, they are either fully separate, or one sits entirely inside the other. A partial overlap cannot happen.

Do 10.0.0.0/16 and 10.0.128.0/17 overlap? 10.0.0.0/16 covers 10.0.0.0 to 10.0.255.255, which contains 10.0.128.0. So yes.

Overlaps cause real trouble. Two networks that use the same range cannot be cleanly joined by a VPN (an encrypted tunnel between networks) or cloud peering (a private link between two cloud networks). A common real case: Docker's default bridge network, the private network it gives to containers, is 172.17.0.0/16. If your office also uses that range, containers may not reach those office hosts, because the local route wins. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).

Routers can hold overlapping routes on purpose, as a larger and a smaller prefix. They then pick the longest match, meaning the route with the longest prefix that fits the destination. See [The Routing Table](../05-routing/02-routing-table.md).

### Choosing a Size in Real Networks

The maths gives you the smallest subnet that fits. Good design leaves room.

- **Plan for growth.** If you need 50 addresses today, /26 fits, but 62 usable leaves little room. Many designers go bigger when space is cheap, as it is inside 10.0.0.0/8.
- **Think about broadcast size.** There is no fixed limit, but many enterprises use a /24 per VLAN as a habit, since one [VLAN](../03-ethernet-and-switching/06-vlans.md) (a virtual LAN, a separate network inside a switch) is usually one subnet and one broadcast domain (the group of devices that hear each other's broadcasts).
- **Make addresses readable.** A plan like 10.[site].[vlan].0/24 lets you read the site and VLAN straight from the address. For example, 10.12.30.0/24 is site 12, VLAN 30.
- **Use the right private range.** The three private blocks are 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. Apply the method to 172.16.0.0/12: block 16 in the 2nd octet, so it ends at 172.31.255.255. This is why 172.32.0.0 is public. See [Public, Private and Special IP Addresses](03-public-private-and-special-addresses.md).

### How IPv6 Differs

IPv6 subnets have no broadcast address, and a LAN is normally a /64, so hosts are not counted this way. The block idea still applies, but you count in hexadecimal. A prefix can end on any bit, though most plans end it on a hex digit boundary (a multiple of 4 bits) because that is easier to read. See [IPv6 Basics](11-ipv6-basics.md).

## Common Mistakes

- **Using the block size in the wrong octet.** /20 has block 16 in the 3rd octet, not the 4th. 172.16.37.200/20 is not in a block near .200.
- **Forgetting the octets to the right.** The broadcast of 172.16.32.0/20 is 172.16.47.255, not 172.16.47.0. Octets right of the interesting one become 0 for the network and 255 for the broadcast.
- **Assuming .0 is always a network and .255 is always a broadcast.** That is only true for /24 and smaller subnets. In a /23 or /20, many .0 and .255 addresses are normal hosts.
- **Saying a /26 holds 64 hosts.** It has 64 addresses and 62 usable hosts. Needing exactly 64 hosts means /25.
- **Subtracting 2 from the subnet count.** The -2 applies to hosts, not subnets. Three borrowed bits give 8 subnets, not 6.
- **Getting the size direction backwards.** A bigger prefix number means a smaller subnet. /30 is tiny. /8 is huge.
- **Starting a subnet at a value that is not a multiple of its block.** 192.168.1.50/26 is a host, not a subnet start.
- **Forgetting the router when counting hosts.** A subnet for 30 devices plus a gateway needs 31 usable addresses, so /27 is too small.
- **Assuming the old class sets the mask.** 10.1.2.3 does not have to be /8. Since CIDR (Classless Inter-Domain Routing, which replaced fixed classes with prefixes of any length), the prefix decides. See [IPv4 Classes](02-ipv4-classes.md).
- **Mixing up counting from 0 and from 1.** The "5th subnet" is #4 when counting from 0. Say which one you mean.
- **Confusing the mask and the wildcard.** `0.0.0.63` is a wildcard, used in ACLs. It is not a subnet mask.
- **Using the -2 rule in the cloud or on /31.** AWS leaves 251 usable in a /24, and a /31 has 2 usable addresses.

## Interview Questions

**Q: What is the block size and how do you find it?**

A: It is the gap between one subnet and the next, in the octet that holds the last network bit. I find it with 256 minus the mask value in that octet. For 255.255.255.224 that is 256 minus 224, which is 32, so subnets start at 0, 32, 64 and so on.

**Q: Find the network, broadcast, and usable range for 172.16.37.200/20.**

A: /20 is in the third octet, and the block is 16. 37 falls between 32 and 48, so the network is 172.16.32.0 and the next network is 172.16.48.0. The broadcast is 172.16.47.255. Usable hosts run from 172.16.32.1 to 172.16.47.254, which is 4,094 hosts.

**Q: Why do we subtract 2 when counting hosts?**

A: The first address, with all host bits 0, identifies the subnet. The last address, with all host bits 1, is the broadcast. Neither can be given to a device. The exceptions are /31 on point-to-point links, where both addresses are usable, and /32, which is a single address.

**Q: How many usable hosts are in a /27 and a /22?**

A: A /27 has 5 host bits, so 2 to the 5 minus 2 is 30. A /22 has 10 host bits, so 1,024 minus 2 is 1,022.

**Q: Split 192.168.1.0/24 into 5 equal subnets. What prefix do you use?**

A: I need 3 borrowed bits, because 2 bits give only 4 subnets and 3 give 8. So the prefix is /27, with block 32 and 30 hosts each. The subnets start at 0, 32, 64, 96, 128, 160, 192 and 224. Three stay spare for growth.

**Q: What is the smallest subnet that holds 500 hosts?**

A: I need 9 host bits, since 2 to the 9 minus 2 is 510 and 8 bits only gives 254. So it is a /23, mask 255.255.254.0.

**Q: Is 10.20.30.255/23 a valid host address?**

A: Yes. The block is 2 in the third octet, so the subnet is 10.20.30.0 to 10.20.31.255. The address ends in .255, but it is in the middle of the range. The real broadcast is 10.20.31.255.

**Q: Can 192.168.1.60/26 and 192.168.1.70/26 talk without a router?**

A: No. The block is 64, so .60 is in the subnet starting at .0 and .70 is in the subnet starting at .64. They are in different subnets, so traffic has to go through a router.

**Q: What is a /31 used for?**

A: Point-to-point links, as defined in RFC 3021. A link with two ends does not need a broadcast address, so both addresses are used for the two routers. It saves two addresses per link compared with a /30.

**Q: Is the formula for subnets 2^s or 2^s - 2?**

A: It is 2^s today. The minus 2 came from old rules that banned the first and last subnets. Modern classless networks use them, and Cisco has allowed subnet zero by default since IOS 12.0.

**Q: Can you make 10 subnets of 20 hosts each from a /24?**

A: It cannot be done. 20 hosts needs 5 host bits, and 10 subnets needs 4 subnet bits. That is 9 bits, and a /24 has only 8. I would ask for a larger block, such as a /23, which gives the 9 bits needed.

**Q: Why does an AWS /24 subnet give only 251 usable addresses?**

A: AWS reserves five addresses in every subnet. Those are the network address, the broadcast address, and three it keeps for the VPC router, DNS and future use. So the normal 2^h - 2 rule does not apply.

## Quick Recap

- Block size = 256 minus the mask value in the interesting octet. It is the gap between subnets.
- Network = biggest multiple of the block that is not above the address value, with octets to the right set to 0.
- Broadcast = next network minus 1, first host = network + 1, last host = broadcast - 1.
- Usable hosts = 2^(host bits) - 2, except for /31 and /32 and in cloud subnets.
- For N subnets, borrow s bits where 2^s is at least N. For N hosts, keep h bits where 2^h - 2 is at least N.
- Fast block size: distance from the prefix up to the next multiple of 8, then 2 to that power.
- An address alone means nothing. Always read it with its mask.
