# Subnetting Practice Problems

**Level:** Intermediate | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md), [Subnetting Step by Step](05-subnetting-step-by-step.md)

This is a practice set of 89 graded problems. Every problem has a full worked answer. Subnetting is a skill, like mental arithmetic. You get it by doing it, not by reading about it.

## How to Use This Set

1. Cover the answer with a sheet of paper or your hand.
2. Solve the problem on paper, without a calculator.
3. Compare your working with the answer, not only the final number.
4. If you got it wrong, redo it the next day.
5. Move to the next level when you get most of a level right without looking.

Good targets are about one minute for a Level 1 problem and about two minutes for a Level 2 problem. These are practice targets, not an official standard. A VLSM design takes longer, and an interviewer expects that.

### Problem Map

| Level | Problems | What it trains |
|---|---|---|
| Level 1: Warm-Up | 1-23 | Easy masks, the basic method |
| Level 2: Core | 24-54 | Masks in any octet, design, ACL wildcards |
| Level 3: Hard | 55-77 | Reverse problems, VLSM, tricky wildcards, summaries |
| Level 4: Interview Challenge | 78-89 | Real situations: cloud, IPv6, faults, full designs |

| Topic | Problems |
|---|---|
| Find network and broadcast | 1-8, 24-30 |
| Count hosts, convert masks | 9-14 |
| Is this a valid host? | 31-33 |
| Same subnet or not | 15-17, 34-37 |
| Design N subnets | 38-43 |
| Pick the smallest mask | 18-20, 44-47 |
| Wildcard masks | 21-23, 48-51, 68-71 |
| Summarisation | 52-54, 72-77 |
| Counting and reverse problems | 55-61 |
| VLSM design | 62-67, 89 |
| Real-world situations | 78-88 |

## The Tools You Need

### Terms Used in This Set

- **IP address:** a 32-bit number that names one interface, written as four numbers from 0 to 255. Each of the four numbers is an **octet** (8 bits).
- **Subnet:** one group of addresses that sit together on the same network.
- **Subnet mask:** a 32-bit pattern that shows which bits name the network (the 1s) and which bits name the host (the 0s). Example: 255.255.255.0.
- **Prefix:** the same mask written as a count of 1 bits. /24 means 24 ones, the same as 255.255.255.0.
- **Network address:** the first address of a subnet. All host bits are 0. It names the subnet.
- **Broadcast address:** the last address of a subnet. All host bits are 1. A packet sent to it reaches every host in that subnet.
- **Host:** any device with an address, such as a PC, server or router interface.
- **Usable hosts:** the addresses you can give to devices. Normally this is all addresses minus 2 (the network and the broadcast).
- **Block size:** how many numbers one subnet covers in the octet where the mask changes. A subnet always starts on a multiple of its block size.
- **Interesting octet:** the octet where the mask is not 255 and not 0. All the work happens there.
- **Borrowing bits:** taking host bits and using them as extra network bits, to make more subnets.
- **Wildcard mask:** the inverse of a subnet mask. A 0 bit means "must match". A 1 bit means "do not care". Used in Cisco ACLs and some OSPF commands.
- **VLSM (Variable Length Subnet Mask):** using different mask sizes inside one address space, so each group gets only the size it needs.
- **Summarisation:** replacing many routes with one shorter route that covers them all. Also called aggregation or supernetting.
- **ACL (Access Control List):** an ordered list of permit and deny rules on a router or firewall. See [Access Control Lists](../09-security/05-access-control-lists.md).

### The Block Size Method

Think of a long street with numbered houses. The city cuts the street into equal groups. If each group has 16 houses, the groups start at 0, 16, 32, 48 and so on. To find a house's group, find the group start just below it.

- The first house in the group holds the group's name sign. That is the network address.
- The last house holds the loudspeaker for the group. That is the broadcast address.
- The houses between are for people. Those are the usable hosts.

The analogy stops here: a real street can start its groups anywhere. In IP, a block must always start on an exact multiple of its size, because the rule comes from binary bits.

The recipe:

1. Find the interesting octet: the octet where the mask is not 255 and not 0.
2. Block size = 256 - the mask value in that octet.
3. Network: find the largest multiple of the block size that is not bigger than the address value in that octet. Set all octets to the right to 0.
4. Broadcast: take the network value, add the block size, subtract 1. Set all octets to the right to 255.
5. First host = network + 1. Last host = broadcast - 1.
6. Usable hosts = 2^(host bits) - 2. Host bits = 32 - prefix.

If the mask falls exactly on an octet edge (/8, /16, /24), there is no interesting octet. The host octets simply go from 0 to 255.

The same answer comes from binary. The network address is the address AND the mask (a bit stays 1 only if both bits are 1). The block size method is a fast shortcut for this.

### Powers of 2

| n | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 2^n | 2 | 4 | 8 | 16 | 32 | 64 | 128 | 256 | 512 | 1024 | 2048 | 4096 |

### Mask Values in One Octet

| Bits set | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Mask value | 128 | 192 | 224 | 240 | 248 | 252 | 254 | 255 |
| Block size | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

### Full Prefix Table

| Prefix | Subnet mask | Wildcard | Block size | Addresses | Usable hosts |
|---|---|---|---|---|---|
| /8 | 255.0.0.0 | 0.255.255.255 | 1 in octet 1 | 16,777,216 | 16,777,214 |
| /9 | 255.128.0.0 | 0.127.255.255 | 128 in octet 2 | 8,388,608 | 8,388,606 |
| /10 | 255.192.0.0 | 0.63.255.255 | 64 in octet 2 | 4,194,304 | 4,194,302 |
| /11 | 255.224.0.0 | 0.31.255.255 | 32 in octet 2 | 2,097,152 | 2,097,150 |
| /12 | 255.240.0.0 | 0.15.255.255 | 16 in octet 2 | 1,048,576 | 1,048,574 |
| /13 | 255.248.0.0 | 0.7.255.255 | 8 in octet 2 | 524,288 | 524,286 |
| /14 | 255.252.0.0 | 0.3.255.255 | 4 in octet 2 | 262,144 | 262,142 |
| /15 | 255.254.0.0 | 0.1.255.255 | 2 in octet 2 | 131,072 | 131,070 |
| /16 | 255.255.0.0 | 0.0.255.255 | 1 in octet 2 | 65,536 | 65,534 |
| /17 | 255.255.128.0 | 0.0.127.255 | 128 in octet 3 | 32,768 | 32,766 |
| /18 | 255.255.192.0 | 0.0.63.255 | 64 in octet 3 | 16,384 | 16,382 |
| /19 | 255.255.224.0 | 0.0.31.255 | 32 in octet 3 | 8,192 | 8,190 |
| /20 | 255.255.240.0 | 0.0.15.255 | 16 in octet 3 | 4,096 | 4,094 |
| /21 | 255.255.248.0 | 0.0.7.255 | 8 in octet 3 | 2,048 | 2,046 |
| /22 | 255.255.252.0 | 0.0.3.255 | 4 in octet 3 | 1,024 | 1,022 |
| /23 | 255.255.254.0 | 0.0.1.255 | 2 in octet 3 | 512 | 510 |
| /24 | 255.255.255.0 | 0.0.0.255 | 1 in octet 3 | 256 | 254 |
| /25 | 255.255.255.128 | 0.0.0.127 | 128 in octet 4 | 128 | 126 |
| /26 | 255.255.255.192 | 0.0.0.63 | 64 in octet 4 | 64 | 62 |
| /27 | 255.255.255.224 | 0.0.0.31 | 32 in octet 4 | 32 | 30 |
| /28 | 255.255.255.240 | 0.0.0.15 | 16 in octet 4 | 16 | 14 |
| /29 | 255.255.255.248 | 0.0.0.7 | 8 in octet 4 | 8 | 6 |
| /30 | 255.255.255.252 | 0.0.0.3 | 4 in octet 4 | 4 | 2 |
| /31 | 255.255.255.254 | 0.0.0.1 | 2 in octet 4 | 2 | 2 (point-to-point links only) |
| /32 | 255.255.255.255 | 0.0.0.0 | 1 in octet 4 | 1 | 1 (one single address) |

## Level 1: Warm-Up

### Find the Network and Broadcast

For problems 1-8, find the network address, broadcast address, usable host range and number of usable hosts.

**Problem 1.** 192.168.10.57/24

The mask ends on an octet edge. The last octet is all host bits.

- Network: 192.168.10.0
- Broadcast: 192.168.10.255
- Hosts: 192.168.10.1 to 192.168.10.254
- Usable: 2^8 - 2 = 254

**Problem 2.** 10.20.30.40/8

Only the first octet is network. The other three are host bits.

- Network: 10.0.0.0
- Broadcast: 10.255.255.255
- Hosts: 10.0.0.1 to 10.255.255.254
- Usable: 2^24 - 2 = 16,777,214

**Problem 3.** 172.16.45.9/16

The first two octets are network.

- Network: 172.16.0.0
- Broadcast: 172.16.255.255
- Hosts: 172.16.0.1 to 172.16.255.254
- Usable: 2^16 - 2 = 65,534

**Problem 4.** 192.168.1.130/25

Mask 255.255.255.128. Block = 256 - 128 = 128. Multiples: 0, 128. The value 130 falls in the block that starts at 128.

- Network: 192.168.1.128
- Broadcast: 128 + 128 - 1 = 255, so 192.168.1.255
- Hosts: 192.168.1.129 to 192.168.1.254
- Usable: 2^7 - 2 = 126

**Problem 5.** 192.168.1.100/26

Mask 255.255.255.192. Block = 64. Multiples: 0, 64, 128, 192. The value 100 is in the block that starts at 64.

- Network: 192.168.1.64
- Broadcast: 64 + 64 - 1 = 127, so 192.168.1.127
- Hosts: 192.168.1.65 to 192.168.1.126
- Usable: 2^6 - 2 = 62

**Problem 6.** 192.168.5.200/27

Mask 255.255.255.224. Block = 32. Multiples: 0, 32, 64, 96, 128, 160, 192, 224. The value 200 is in the block that starts at 192.

- Network: 192.168.5.192
- Broadcast: 192 + 32 - 1 = 223, so 192.168.5.223
- Hosts: 192.168.5.193 to 192.168.5.222
- Usable: 2^5 - 2 = 30

**Problem 7.** 10.0.0.77/28

Mask 255.255.255.240. Block = 16. 16 x 4 = 64 and 16 x 5 = 80, so 77 is in the block that starts at 64.

- Network: 10.0.0.64
- Broadcast: 64 + 16 - 1 = 79, so 10.0.0.79
- Hosts: 10.0.0.65 to 10.0.0.78
- Usable: 2^4 - 2 = 14

**Problem 8.** 192.168.50.9/30

Mask 255.255.255.252. Block = 4. 4 x 2 = 8 and 4 x 3 = 12, so 9 is in the block that starts at 8.

- Network: 192.168.50.8
- Broadcast: 8 + 4 - 1 = 11, so 192.168.50.11
- Hosts: 192.168.50.9 to 192.168.50.10
- Usable: 2^2 - 2 = 2

### Count Hosts and Convert Masks

**Problem 9.** How many usable hosts does a /29 give?

Host bits = 32 - 29 = 3. 2^3 - 2 = **6 usable hosts**.

**Problem 10.** How many usable hosts does a /23 give?

Host bits = 32 - 23 = 9. 2^9 - 2 = **510 usable hosts**.

**Problem 11.** How many usable hosts does a /20 give?

Host bits = 32 - 20 = 12. 2^12 - 2 = **4,094 usable hosts**.

**Problem 12.** Write 255.255.255.240 as a prefix. How many usable hosts?

The first three octets give 24 ones. 240 is 1111 0000, which adds 4 ones. 24 + 4 = **/28**. Host bits = 4, so 2^4 - 2 = **14 usable hosts**.

**Problem 13.** Write /21 as a dotted mask.

21 = 16 + 5. Two full octets are 255.255. The third octet has 5 ones: 1111 1000 = 248. The last octet is 0. Answer: **255.255.248.0**.

**Problem 14.** Write 255.255.252.0 as a prefix. How many usable hosts?

255.255 gives 16 ones. 252 is 1111 1100, which adds 6 ones. 16 + 6 = **/22**. Host bits = 10, so 2^10 - 2 = **1,022 usable hosts**.

### Same Subnet or Not

To test two addresses, find the network of each one. Same network means same subnet.

**Problem 15.** Are 192.168.1.10 and 192.168.1.200 in the same subnet with mask /24?

Both have network 192.168.1.0. **Yes.**

**Problem 16.** Are 192.168.1.10 and 192.168.1.200 in the same subnet with mask /25?

Block = 128. The value 10 is in block 0. The value 200 is in block 128. Networks are 192.168.1.0 and 192.168.1.128. **No.**

The addresses did not change from Problem 15. Only the mask did. Same subnet is a question about the addresses and the mask together.

**Problem 17.** Are 10.1.1.5 and 10.1.2.5 in the same subnet with mask /24?

Networks are 10.1.1.0 and 10.1.2.0. **No.** With /24, the third octet must match, and it does not.

### Pick the Smallest Mask (Easy)

"Smallest subnet" means the fewest addresses that still fit. It is the longest prefix that works.

| Usable hosts needed, up to | 2 | 6 | 14 | 30 | 62 | 126 | 254 | 510 | 1022 |
|---|---|---|---|---|---|---|---|---|---|
| Prefix | /30 | /29 | /28 | /27 | /26 | /25 | /24 | /23 | /22 |

**Problem 18.** You need 10 usable hosts. Pick the smallest subnet.

/29 gives 6, which is too small. /28 gives 14, which fits. Answer: **/28**.

**Problem 19.** You need 50 usable hosts. Pick the smallest subnet.

/27 gives 30, which is too small. /26 gives 62, which fits. Answer: **/26**.

**Problem 20.** A link joins two routers. Pick the smallest subnet.

The classic answer is **/30**, which gives exactly 2 usable hosts. On modern gear you may also use /31 on a point-to-point link. See Problem 78.

### Wildcard Masks (Easy)

To get the wildcard, subtract each mask octet from 255.

Think of a wildcard as a stencil on a form. A 0 says "check this box". A 1 says "ignore this box". The analogy stops at the bit level: the rule applies to every single bit, not to whole octets.

**Problem 21.** What is the wildcard of 255.255.255.0?

255 - 255 = 0 for the first three octets. 255 - 0 = 255 for the last one. Answer: **0.0.0.255**.

**Problem 22.** What is the wildcard of /26?

The mask is 255.255.255.192. The last octet is 255 - 192 = 63. Answer: **0.0.0.63**.

**Problem 23.** A wildcard is 0.0.0.15. What are the mask and prefix?

255 - 15 = 240. The mask is 255.255.255.240, which is **/28**.

## Level 2: Core

### Network and Broadcast with Any Mask

These problems have the interesting octet in the middle of the address. The method is the same. Just remember that octets to the right of the interesting octet become 0 for the network and 255 for the broadcast.

**Problem 24.** 172.16.35.123/20. Find network, broadcast, host range and usable hosts.

/20 means 255.255.240.0. The interesting octet is the third. Block = 256 - 240 = 16.

```text
Third octet, block size 16

 0       16      32      48      64
 |-------|-------|-------|-------|
                  ^
                  35 is here, so the block is 32 to 47
```

- Network: 172.16.32.0
- Broadcast: 32 + 16 - 1 = 47, so 172.16.47.255
- Hosts: 172.16.32.1 to 172.16.47.254
- Usable: 2^12 - 2 = 4,094

Binary proof of the network value:

```text
Address, third octet    35 = 0010 0011
Mask, third octet      240 = 1111 0000
AND result              32 = 0010 0000
```

**Problem 25.** 10.10.99.200/21

Mask 255.255.248.0. Block = 8 in the third octet. 8 x 12 = 96 and 8 x 13 = 104, so 99 is in the block that starts at 96.

- Network: 10.10.96.0
- Broadcast: 96 + 8 - 1 = 103, so 10.10.103.255
- Hosts: 10.10.96.1 to 10.10.103.254
- Usable: 2^11 - 2 = 2,046

**Problem 26.** 192.168.18.5/22

Mask 255.255.252.0. Block = 4 in the third octet. 4 x 4 = 16 and 4 x 5 = 20, so 18 is in the block that starts at 16.

- Network: 192.168.16.0
- Broadcast: 16 + 4 - 1 = 19, so 192.168.19.255
- Hosts: 192.168.16.1 to 192.168.19.254
- Usable: 2^10 - 2 = 1,022

**Problem 27.** 172.20.200.17/19

Mask 255.255.224.0. Block = 32 in the third octet. 32 x 6 = 192 and 32 x 7 = 224, so 200 is in the block that starts at 192.

- Network: 172.20.192.0
- Broadcast: 192 + 32 - 1 = 223, so 172.20.223.255
- Hosts: 172.20.192.1 to 172.20.223.254
- Usable: 2^13 - 2 = 8,190

**Problem 28.** 10.200.15.250/12

Mask 255.240.0.0. The interesting octet is the second one. Block = 16. 16 x 12 = 192 and 16 x 13 = 208, so 200 is in the block that starts at 192.

- Network: 10.192.0.0
- Broadcast: 192 + 16 - 1 = 207, so 10.207.255.255
- Hosts: 10.192.0.1 to 10.207.255.254
- Usable: 2^20 - 2 = 1,048,574

**Problem 29.** 192.168.77.129/29

Mask 255.255.255.248. Block = 8. 8 x 16 = 128 and 8 x 17 = 136, so 129 is in the block that starts at 128.

- Network: 192.168.77.128
- Broadcast: 128 + 8 - 1 = 135, so 192.168.77.135
- Hosts: 192.168.77.129 to 192.168.77.134
- Usable: 2^3 - 2 = 6

**Problem 30.** 10.77.130.9/17

Mask 255.255.128.0. Block = 128 in the third octet. Multiples: 0, 128. The value 130 is in the block that starts at 128.

- Network: 10.77.128.0
- Broadcast: 128 + 128 - 1 = 255, so 10.77.255.255
- Hosts: 10.77.128.1 to 10.77.255.254
- Usable: 2^15 - 2 = 32,766

### Is This a Valid Host Address?

Find the network and broadcast. If the address is one of them, it cannot be given to a device. Anything between them can.

**Problem 31.** Is 192.168.4.255/23 a valid host address?

Block = 2 in the third octet. The value 4 is a multiple of 2, so the subnet runs from 192.168.4.0 to 192.168.5.255. The address is in the middle. **Yes, it is a valid host.**

An address that ends in .255 is not always a broadcast. With a mask shorter than /24, it can be a normal host.

**Problem 32.** Is 172.16.31.255/20 a valid host address?

Block = 16 in the third octet. The value 31 is in the block that starts at 16. The subnet runs from 172.16.16.0 to 172.16.31.255. **No, this is the broadcast address.**

**Problem 33.** With mask /21, what are 10.0.8.0 and 10.0.9.0?

Block = 8 in the third octet. Both are in the block that starts at 8, so the subnet is 10.0.8.0 to 10.0.15.255.

- 10.0.8.0 is the **network address**. It is not a valid host.
- 10.0.9.0 is a **valid host**. It ends in .0, but it is in the middle of the subnet.

### Same Subnet or Not (Harder)

**Problem 34.** Are 172.16.33.5 and 172.16.47.200 in the same subnet with /20?

Block = 16 in the third octet. Both 33 and 47 are in the block 32-47. Both networks are 172.16.32.0. **Yes.**

**Problem 35.** Are 172.16.33.5 and 172.16.48.1 in the same subnet with /20?

The value 33 is in block 32-47. The value 48 is in block 48-63. Networks are 172.16.32.0 and 172.16.48.0. **No.**

**Problem 36.** Are 10.1.63.254 and 10.1.64.1 in the same subnet with /18?

Block = 64 in the third octet. The value 63 is in block 0-63. The value 64 is in block 64-127. **No.** They look close, but a block edge sits between them.

**Problem 37.** Host A is 192.168.1.10/24. Host B is 192.168.1.130/25. They are on the same switch and the same VLAN. What happens when A pings B?

Each host decides using its own mask.

- A: "My network is 192.168.1.0/24. B is inside it." So A sends an ARP request for B and sends the ping directly.
- B: "My network is 192.168.1.128/25. A is outside it." So B sends the reply to its default gateway, not to A.

If B has a working gateway, the reply may arrive through the router, and the ping can still work on an uneven path. If B has no gateway, the reply is lost and the ping fails. The real fix is to use the same mask on every host in a subnet.

For how a host uses ARP, see [ARP](../03-ethernet-and-switching/05-arp.md). For the gateway decision, see [The Default Gateway](08-default-gateway.md).

### Design N Subnets

Find the smallest number of borrowed bits n where 2^n is at least the number of subnets you need. New prefix = old prefix + n.

Old books subtract 2 from the number of subnets, because the first and last subnets were once avoided. That rule is history. Modern networks use all of them. Cisco IOS has had `ip subnet-zero` on by default since release 12.0.

**Problem 38.** Split 192.168.1.0/24 into 4 equal subnets.

2^2 = 4, so borrow 2 bits. New prefix = /26. Block = 64. Each subnet has 62 usable hosts.

| Subnet | Network | Usable hosts | Broadcast |
|---|---|---|---|
| 1 | 192.168.1.0/26 | .1 to .62 | .63 |
| 2 | 192.168.1.64/26 | .65 to .126 | .127 |
| 3 | 192.168.1.128/26 | .129 to .190 | .191 |
| 4 | 192.168.1.192/26 | .193 to .254 | .255 |

**Problem 39.** Split 192.168.10.0/24 into at least 6 subnets. Give the prefix, hosts per subnet and all networks.

2^2 = 4 is too few. 2^3 = 8 is enough. Borrow 3 bits. New prefix = /27. Block = 32. Each subnet has 30 usable hosts. You get 8 subnets, 2 more than you need.

Networks: 192.168.10.0, .32, .64, .96, .128, .160, .192, .224, all /27.

**Problem 40.** Split 172.16.0.0/16 into at least 30 subnets. Give the prefix, the hosts per subnet, the first three subnets and the last subnet.

2^4 = 16 is too few. 2^5 = 32 is enough. Borrow 5 bits. New prefix = /21. Mask = 255.255.248.0. Block = 8 in the third octet. Hosts = 2^11 - 2 = 2,046.

- Subnet 1: 172.16.0.0/21 (to 172.16.7.255)
- Subnet 2: 172.16.8.0/21 (to 172.16.15.255)
- Subnet 3: 172.16.16.0/21 (to 172.16.23.255)
- Last, subnet 32: 172.16.248.0/21 (to 172.16.255.255)

**Problem 41.** Split 10.0.0.0/8 into at least 1000 subnets. Give the prefix, hosts per subnet and the first five subnets.

2^9 = 512 is too few. 2^10 = 1024 is enough. Borrow 10 bits. New prefix = /18. Mask = 255.255.192.0. Block = 64 in the third octet. Hosts = 2^14 - 2 = 16,382.

When the third octet passes 192, the next block carries into the second octet:

- 10.0.0.0/18
- 10.0.64.0/18
- 10.0.128.0/18
- 10.0.192.0/18
- 10.1.0.0/18

**Problem 42.** You have 192.168.100.0/24. Each subnet must hold at least 25 hosts. How many subnets can you make?

25 hosts needs /27 (30 usable). From /24 to /27 you borrow 3 bits. 2^3 = **8 subnets**, each with 30 usable hosts.

**Problem 43.** You cut 172.16.0.0/16 into /26 subnets. How many subnets, and how many usable hosts in each?

Borrowed bits = 26 - 16 = 10, so 2^10 = **1,024 subnets**. Host bits = 6, so 2^6 - 2 = **62 usable hosts** in each.

### Pick the Smallest Mask (Harder)

**Problem 44.** You need 500 usable hosts.

2^8 - 2 = 254 is too small. 2^9 - 2 = 510 fits. Host bits = 9. Prefix = 32 - 9 = **/23**.

**Problem 45.** You need 1000 usable hosts.

2^9 - 2 = 510 is too small. 2^10 - 2 = 1,022 fits. Host bits = 10. Prefix = **/22**.

**Problem 46.** A LAN has 30 PCs, 1 router and 1 network printer. Pick the smallest subnet.

This is a trap. The router and the printer also need addresses. Total = 32. /27 gives 30, which is too small. **/26** gives 62, which fits.

**Problem 47.** A VLAN has 120 users today. You expect 20% growth. You also need 3 addresses for the gateway: two routers and one shared virtual address. Pick the smallest subnet.

120 x 1.2 = 144 users. 144 + 3 = 147 addresses. /25 gives 126, which is too small. **/24** gives 254, which fits. For the shared gateway idea, see [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

### Wildcard Masks and ACLs

Wildcard masks appear in Cisco IOS ACLs and in the OSPF `network` command.

**Problem 48.** What is the wildcard of /20?

Mask = 255.255.240.0. 255 - 240 = 15 and 255 - 0 = 255. Answer: **0.0.15.255**.

**Problem 49.** Write a Cisco standard ACL line that permits all of 192.168.10.0/24.

```text
! Cisco IOS
access-list 10 permit 192.168.10.0 0.0.0.255
```

Standard ACLs use the wildcard, not the subnet mask. Writing 255.255.255.0 here is a common error.

**Problem 50.** Which addresses does this line match?

```text
! Cisco IOS
access-list 10 permit 172.16.32.0 0.0.15.255
```

The wildcard 15 in the third octet means block 16. The start is 32. The range is **172.16.32.0 to 172.16.47.255**. This is 172.16.32.0/20.

**Problem 51.** A router has this OSPF command. Do interfaces with 10.1.6.1 and 10.1.8.1 match?

```text
! Cisco IOS
router ospf 1
 network 10.1.4.0 0.0.3.255 area 0
```

Wildcard 3 in the third octet means block 4, so the command covers 10.1.4.0 to 10.1.7.255.

- 10.1.6.1 **matches**, so OSPF runs on that interface.
- 10.1.8.1 **does not match**.

For more, see [OSPF](../05-routing/06-ospf.md).

### Summarisation (Easy)

Think of a sorting office. It does not need a list of every street in a city. It only needs "all of this city goes on that truck". A summary route does the same for routers.

The analogy stops here: the summary can also cover addresses that do not exist. Problem 77 shows what that can cause.

The quick rule for a clean summary:

1. The count of networks must be a power of 2.
2. The first network must be a multiple of that count.
3. New prefix = old prefix - n, where 2^n is the count.

**Problem 52.** Summarise 192.168.0.0/24, 192.168.1.0/24, 192.168.2.0/24 and 192.168.3.0/24.

Count = 4 = 2^2. Start 0 is a multiple of 4. New prefix = 24 - 2 = /22. Answer: **192.168.0.0/22**.

Binary proof:

```text
192.168.0.0   third octet   0000 00|00
192.168.1.0   third octet   0000 00|01
192.168.2.0   third octet   0000 00|10
192.168.3.0   third octet   0000 00|11
                            same   |differ

16 bits (octets 1 and 2) + 6 same bits = /22
```

**Problem 53.** Summarise 10.1.4.0/24, 10.1.5.0/24, 10.1.6.0/24 and 10.1.7.0/24.

Count = 4. Start 4 is a multiple of 4. New prefix = /22. Answer: **10.1.4.0/22**.

**Problem 54.** Summarise 172.16.16.0/24 through 172.16.23.0/24.

Count = 8 = 2^3. Start 16 is a multiple of 8. New prefix = 24 - 3 = /21. Answer: **172.16.16.0/21**.

## Level 3: Hard

### Counting and Reverse Problems

**Problem 55.** 172.16.0.0/16 is cut into /20 subnets. What are the 10th subnet and its broadcast?

Block = 16 in the third octet. Subnets start at 0, so the 10th starts at 9 x 16 = 144.

- 10th subnet: 172.16.144.0/20
- Broadcast: 144 + 16 - 1 = 159, so 172.16.159.255

**Problem 56.** What is the 300th usable host in 192.168.0.0/22?

Usable host number n is the network address plus n. 300 = 1 x 256 + 44, so carry 1 into the third octet. Answer: **192.168.1.44**.

Note that 192.168.0.255 and 192.168.1.0 are normal hosts in this /22. They are counted.

**Problem 57.** How many /30 subnets fit in a /22? How many /26 subnets fit in a /20?

Difference in prefix = n. Count = 2^n.

- /22 to /30: 8 bits, so 2^8 = **256**
- /20 to /26: 6 bits, so 2^6 = **64**

**Problem 58.** Do these pairs overlap?

(a) 10.1.0.0/22 and 10.1.2.0/24

(b) 10.1.4.0/23 and 10.1.6.0/23

(a) 10.1.0.0/22 covers third octet 0-3. 10.1.2.0/24 is inside it. **They overlap.**

(b) 10.1.4.0/23 covers 4-5. 10.1.6.0/23 covers 6-7. **They do not overlap.** They are neighbours, and together they form 10.1.4.0/22.

**Problem 59.** Are these valid network addresses? 172.16.6.0/22, 192.168.1.64/25, 10.4.8.0/21

A network address must have all host bits 0. In the block method, the value must be a multiple of the block size.

- 172.16.6.0/22: block 4, and 6 is not a multiple of 4. **Not valid.** It is a host inside 172.16.4.0/22.
- 192.168.1.64/25: block 128, and 64 is not a multiple of 128. **Not valid.** It is a host inside 192.168.1.0/25.
- 10.4.8.0/21: block 8, and 8 is a multiple of 8. **Valid.**

**Problem 60.** A subnet has network 172.16.40.0 and broadcast 172.16.47.255. What is the mask?

The third octet covers 40 to 47, which is 8 values. Total addresses = 8 x 256 = 2,048 = 2^11. Host bits = 11. Prefix = **/21**, mask 255.255.248.0. Check: 40 is a multiple of 8, so it is a valid start.

**Problem 61.** What is the longest prefix that keeps 10.1.2.3 and 10.1.2.60 in one subnet?

Write the last octets in binary and count the bits they share from the left.

```text
  3 = 0000 0011
 60 = 0011 1100
      ^^
      2 shared bits, then they differ
```

24 + 2 = **/26**. Check: with /26 (block 64), both are in 0-63. With /27 (block 32), 3 is in 0-31 and 60 is in 32-63, so they split.

### VLSM Design

Think of packing boxes onto a shelf. Put the biggest box first. Then each smaller box slots in right behind it, with no gaps. The analogy stops here: a real box can go anywhere on the shelf, but a subnet can only start on a multiple of its own size.

The recipe:

1. Sort the needs from largest to smallest.
2. Pick the smallest prefix for each need.
3. Give the largest need the first address.
4. Start the next subnet at the next free address.
5. Write down what is left over.

The next problems use the full method. For the theory, see [VLSM, Route Summarisation and Supernetting](07-vlsm-and-supernetting.md).

**Problem 62.** You have 192.168.1.0/24. Sales needs 100 hosts, HR needs 50, IT needs 20, and two router links need 2 hosts each. Make a VLSM plan.

| Group | Needs | Prefix | Subnet | Usable hosts | Broadcast |
|---|---|---|---|---|---|
| Sales | 100 | /25 | 192.168.1.0/25 | .1 to .126 | .127 |
| HR | 50 | /26 | 192.168.1.128/26 | .129 to .190 | .191 |
| IT | 20 | /27 | 192.168.1.192/27 | .193 to .222 | .223 |
| Link 1 | 2 | /30 | 192.168.1.224/30 | .225 to .226 | .227 |
| Link 2 | 2 | /30 | 192.168.1.228/30 | .229 to .230 | .231 |

Free space: 192.168.1.232/29 and 192.168.1.240/28.

```text
192.168.1.0/24 (not to scale)

.0          .128      .192  .224 .228 .232     .255
+-----------+---------+-----+----+----+---------+
| Sales     | HR      | IT  | L1 | L2 | free    |
| /25       | /26     | /27 |/30 |/30 | 24 addr |
+-----------+---------+-----+----+----+---------+
```

**Problem 63.** You have 172.16.10.0/23. Group A needs 200 hosts, B needs 120, C needs 60, D needs 25, and three links need 2 hosts each. Make a plan.

The /23 covers 172.16.10.0 to 172.16.11.255.

| Group | Needs | Prefix | Subnet | Usable hosts | Broadcast |
|---|---|---|---|---|---|
| A | 200 | /24 | 172.16.10.0/24 | 172.16.10.1 to 172.16.10.254 | 172.16.10.255 |
| B | 120 | /25 | 172.16.11.0/25 | 172.16.11.1 to 172.16.11.126 | 172.16.11.127 |
| C | 60 | /26 | 172.16.11.128/26 | 172.16.11.129 to 172.16.11.190 | 172.16.11.191 |
| D | 25 | /27 | 172.16.11.192/27 | 172.16.11.193 to 172.16.11.222 | 172.16.11.223 |
| Link 1 | 2 | /30 | 172.16.11.224/30 | 172.16.11.225 to 172.16.11.226 | 172.16.11.227 |
| Link 2 | 2 | /30 | 172.16.11.228/30 | 172.16.11.229 to 172.16.11.230 | 172.16.11.231 |
| Link 3 | 2 | /30 | 172.16.11.232/30 | 172.16.11.233 to 172.16.11.234 | 172.16.11.235 |

Free space: 172.16.11.236/30 and 172.16.11.240/28.

**Problem 64.** You have 10.0.0.0/22. Site X needs 400 hosts, Y needs 200, Z needs 90, and two WAN links must use /31.

The /22 covers 10.0.0.0 to 10.0.3.255.

| Group | Needs | Prefix | Subnet | Usable addresses | Broadcast |
|---|---|---|---|---|---|
| X | 400 | /23 | 10.0.0.0/23 | 10.0.0.1 to 10.0.1.254 | 10.0.1.255 |
| Y | 200 | /24 | 10.0.2.0/24 | 10.0.2.1 to 10.0.2.254 | 10.0.2.255 |
| Z | 90 | /25 | 10.0.3.0/25 | 10.0.3.1 to 10.0.3.126 | 10.0.3.127 |
| WAN 1 | 2 | /31 | 10.0.3.128/31 | 10.0.3.128 and 10.0.3.129 | None |
| WAN 2 | 2 | /31 | 10.0.3.130/31 | 10.0.3.130 and 10.0.3.131 | None |

Free space: 10.0.3.132/30, 10.0.3.136/29, 10.0.3.144/28, 10.0.3.160/27 and 10.0.3.192/26.

**Problem 65.** You have 172.31.0.0/20. You need subnets for 1000, 500, 250 and 100 hosts, plus four /30 links.

The /20 covers 172.31.0.0 to 172.31.15.255.

| Needs | Prefix | Subnet | Usable hosts | Broadcast |
|---|---|---|---|---|
| 1000 | /22 | 172.31.0.0/22 | 172.31.0.1 to 172.31.3.254 | 172.31.3.255 |
| 500 | /23 | 172.31.4.0/23 | 172.31.4.1 to 172.31.5.254 | 172.31.5.255 |
| 250 | /24 | 172.31.6.0/24 | 172.31.6.1 to 172.31.6.254 | 172.31.6.255 |
| 100 | /25 | 172.31.7.0/25 | 172.31.7.1 to 172.31.7.126 | 172.31.7.127 |
| Link 1 | /30 | 172.31.7.128/30 | 172.31.7.129 to 172.31.7.130 | 172.31.7.131 |
| Link 2 | /30 | 172.31.7.132/30 | 172.31.7.133 to 172.31.7.134 | 172.31.7.135 |
| Link 3 | /30 | 172.31.7.136/30 | 172.31.7.137 to 172.31.7.138 | 172.31.7.139 |
| Link 4 | /30 | 172.31.7.140/30 | 172.31.7.141 to 172.31.7.142 | 172.31.7.143 |

Free space: 172.31.7.144/28, 172.31.7.160/27, 172.31.7.192/26 and 172.31.8.0/21.

**Problem 66.** Find the bug in this plan and fix it.

| VLAN | Subnet |
|---|---|
| 10 | 10.1.0.0/24 |
| 20 | 10.1.1.0/25 |
| 30 | 10.1.1.64/26 |
| 40 | 10.1.1.128/26 |

VLAN 20 covers 10.1.1.0 to 10.1.1.127. VLAN 30 covers 10.1.1.64 to 10.1.1.127. VLAN 30 sits inside VLAN 20, so they **overlap**.

Fix: move VLAN 30 to the free block 10.1.1.192/26. It covers .192 to .255 and touches nothing else.

Cisco IOS refuses a subnet that overlaps another interface on the same router. Separate routers may not notice, so the fault can stay hidden until traffic fails.

**Problem 67.** Why go largest first? Use 192.168.1.0/24 with needs of 20, 100 and 50 hosts, and show the wrong order.

Wrong order, smallest first:

1. 20 hosts gets 192.168.1.0/27, which ends at .31.
2. 100 hosts needs a /25. The next free address is .32, but a /25 must start on a multiple of 128. You must jump to 192.168.1.128/25. Addresses .32 to .127 are now a gap.
3. 50 hosts needs a /26. Nothing is free after .255. You must go back and fit it at 192.168.1.64/26.

Right order, largest first:

1. 100 hosts gets 192.168.1.0/25.
2. 50 hosts gets 192.168.1.128/26.
3. 20 hosts gets 192.168.1.192/27.
4. Free space is one clean block: 192.168.1.224/27.

Fuller truth: the wrong order can sometimes be rescued by going back to fill the gaps. Largest first is not the only valid plan. It is the simple plan with no gaps: if the blocks add up to no more than the space, they always fit.

### Harder Wildcards

A wildcard does not need to be one solid run of 1s. A subnet mask must be solid, but an ACL wildcard can check any bit. This makes some unusual matches possible.

**Problem 68.** What does this line match?

```text
! Cisco IOS
access-list 30 permit 192.168.1.0 0.0.0.254
```

Last octet of the wildcard: 254 = 1111 1110. The only checked bit in that octet is the last one. It must match the 0 in the address. So the last octet must be even.

Answer: **all even addresses in 192.168.1.x** (.0, .2, .4 and so on up to .254). That is 128 addresses.

**Problem 69.** Permit exactly 192.168.10.0 to 192.168.13.255 using the fewest ACL lines.

These are 4 /24 networks, but 10 is not a multiple of 4, so one line cannot do it exactly. Split on real block edges:

- 192.168.10.0/23 covers third octet 10-11
- 192.168.12.0/23 covers third octet 12-13

```text
! Cisco IOS
access-list 40 permit 192.168.10.0 0.0.1.255
access-list 40 permit 192.168.12.0 0.0.1.255
```

Answer: **two lines**.

**Problem 70.** VLAN subnets follow the pattern 192.168.X.0/24. Permit only those where X is odd, using one ACL line.

An odd number has a last bit of 1. Check only that bit in the third octet, and ignore the last octet.

```text
! Cisco IOS
access-list 50 permit 192.168.1.0 0.0.254.255
```

Third octet of the wildcard: 254 = 1111 1110, so only the last bit is checked. It must match the 1 in the address.

- 192.168.3.77 matches (3 is odd).
- 192.168.4.1 does not match (4 is even).

**Problem 71.** Permit only hosts 192.168.5.32 to 192.168.5.47. Block the rest.

That is 16 addresses. 32 is a multiple of 16, so this is the clean block 192.168.5.32/28. Wildcard = 0.0.0.15.

```text
! Cisco IOS
access-list 20 permit 192.168.5.32 0.0.0.15
```

No deny line is needed. Every Cisco ACL ends with an implicit "deny any".

### Summarisation (Harder)

**Problem 72.** Summarise 192.168.1.0/24 and 192.168.2.0/24.

This is a trap. It looks like 2 networks and a /23, but it is not.

A /23 has block 2, so the pairs are 0-1 and 2-3. Networks 1 and 2 are in different pairs.

```text
1 = 0000 0001
2 = 0000 0010
    ^^^^ ^^
    6 shared bits, so 16 + 6 = /22
```

The smallest single route is **192.168.0.0/22**. It also covers 192.168.0.0/24 and 192.168.3.0/24, which are not ours. For an exact match, keep **two routes**.

**Problem 73.** Summarise 10.10.32.0/24, 10.10.33.0/24, 10.10.34.0/23 and 10.10.36.0/22.

Count the third octet values: 32, 33, 34-35, 36-39. That is 32 to 39, with no gaps. Count = 8 and 32 is a multiple of 8. Eight /24s make a /21.

Answer: **10.10.32.0/21**.

**Problem 74.** Summarise 192.168.100.0/26, 192.168.100.64/26 and 192.168.100.128/25.

- .0/26 and .64/26 join to make 192.168.100.0/25.
- 192.168.100.0/25 and 192.168.100.128/25 join to make 192.168.100.0/24.

Answer: **192.168.100.0/24**.

**Problem 75.** Summarise 172.16.0.0/16 through 172.31.0.0/16.

The second octet runs from 16 to 31. Count = 16 = 2^4, and 16 is a multiple of 16. New prefix = 16 - 4 = /12.

Answer: **172.16.0.0/12**. This is also the private range from RFC 1918. See [Public, Private and Special IP Addresses](03-public-private-and-special-addresses.md).

**Problem 76.** Summarise 172.16.12.0/24, 172.16.13.0/24, 172.16.14.0/24, 172.16.15.0/24 and 172.16.16.0/24. Give the single route and the exact routes.

Count = 5, which is not a power of 2, so no single route is exact.

```text
12 = 0000 1100
16 = 0001 0000
     ^^^
     3 shared bits, so 16 + 3 = /19
```

- Smallest single route: **172.16.0.0/19**. It covers third octet 0-31, so it includes 27 /24 networks you do not own.
- Exact routes: **172.16.12.0/22** and **172.16.16.0/24**.

**Problem 77.** R2 has 10.1.0.0/24, 10.1.1.0/24 and 10.1.2.0/24. It sends the summary 10.1.0.0/22 to R1. A packet for 10.1.3.9 arrives at R1. What happens?

R1 matches the summary and forwards the packet to R2. But 10.1.3.0/24 does not exist on R2. If R2 has a default route back towards R1, the packet bounces between them until its TTL (a hop counter) reaches 0. That is a routing loop.

```text
           10.1.0.0/22 summary
  R1 <--------------------------- R2
  R1 ---- packet for 10.1.3.9 --> R2
  R1 <--- default route back ---- R2     loop until TTL = 0
```

The fix is a discard route: R2 keeps a route for the summary that points to Null0 (a virtual interface that drops packets). The longer /24 routes still win for real subnets, because routers use the longest prefix match. Only addresses with no real subnet reach Null0.

Cisco IOS adds this discard route by default for EIGRP and OSPF summaries, and for BGP aggregates. Other platforms vary, so check yours. For longest prefix match, see [The Routing Table](../05-routing/02-routing-table.md).

## Level 4: Interview Challenge

**Problem 78.** What are the usable addresses in 10.0.0.0/31? Why does it work when the formula gives 0?

There are 2 addresses, **10.0.0.0 and 10.0.0.1**, and both are usable. The formula 2^1 - 2 = 0 does not apply here.

RFC 3021 defines /31 for point-to-point links. A link with two ends has no need for a network address or a broadcast address.

```text
! Cisco IOS
interface GigabitEthernet0/0
 ip address 10.0.0.0 255.255.255.254
```

IOS accepts this and may show a caution on an Ethernet interface. Check support on any older device first. A /31 saves half the addresses compared with /30, and this matters on networks with thousands of links.

**Problem 79.** What are /32 and /0 used for?

- **/32** is one single address. It is used for router loopback interfaces, such as 10.255.0.1/32, and for host routes in a routing table.
- **/0** matches every address. 0.0.0.0/0 is the default route.

They are the two ends of the scale. Longest prefix match means /32 always wins and /0 always loses, when both match.

**Problem 80.** A PC has 192.168.10.70/26 and gateway 192.168.10.1. It can reach local devices but nothing else. What is wrong?

Find the PC's subnet. Block = 64, and 70 is in the block 64-127. The subnet is 192.168.10.64 to 192.168.10.127. The gateway 192.168.10.1 is in a different subnet, 192.168.10.0/26.

A gateway must be in the same subnet as the host, because the host must reach it directly. Some systems refuse this setting. Others accept it, but traffic to other networks usually fails.

Fix: set the gateway to the router address inside 192.168.10.64/26, or correct the mask if the mask is the mistake.

Check the settings like this:

```bash
# Linux
ip -4 addr show dev eth0
ip route show default
```

```text
inet 192.168.10.70/26 brd 192.168.10.127 scope global eth0
default via 192.168.10.1 dev eth0
```

```powershell
# Windows
ipconfig
Get-NetIPConfiguration
```

**Problem 81.** A router refuses `ip address 192.168.1.127 255.255.255.128`. Why?

Block = 128, so the subnet is 192.168.1.0 to 192.168.1.127. The address .127 is the broadcast. A router interface cannot use it. IOS rejects it with a message such as "Bad mask /25 for address 192.168.1.127".

The message blames the mask, but the real problem is the pair of values. Fix: use a host address, such as 192.168.1.126 or 192.168.1.1.

**Problem 82.** A site uses 192.168.1.0/24 and it is full. Your manager says, "Just add 192.168.2.0/24 and make it one /23." Will that work?

**No.** A /23 has block 2 in the third octet. Valid /23 networks start on even numbers: 192.168.0.0/23 or 192.168.2.0/23. Networks 1 and 2 fall in different /23 blocks.

Options:

- Keep the /24 and add 192.168.2.0/24 as a second subnet, with routing between them.
- Renumber to 192.168.2.0/23, or use 192.168.0.0/23 if 192.168.0.0/24 is free. The site keeps its addresses in the second case.

**Problem 83.** You have 10.0.0.0/8. You need 16 regions, each with 64 sites, each with 16 VLANs. What prefix does each VLAN get? How many hosts? What is the network for region 1, site 3, VLAN 2, counting each from 0?

Bits needed: 16 regions = 4 bits, 64 sites = 6 bits, 16 VLANs = 4 bits. Total = 14 bits. 8 + 14 = **/22**. Each VLAN has 2^10 - 2 = **1,022 usable hosts**.

```text
| 8 bits | 4 bits | 6 bits | 4 bits | 10 bits |
| 10     | region | site   | VLAN   | hosts   |
```

Write the values in binary and fill in the bits:

```text
region 1 = 0001    site 3 = 000011    VLAN 2 = 0010

second octet:  0001 0000  (region, then first 4 site bits)  = 16
third octet:   11 0010 00 (last 2 site bits, VLAN, 2 hosts) = 200
```

Answer: **10.16.200.0/22**.

This kind of plan helps summarising. All of region 1 is 10.16.0.0/12, and all of its site 3 is 10.16.192.0/18.

**Problem 84.** An AWS subnet is 10.0.1.0/24. How many addresses can you assign? What about a /28? How many /24 subnets fit in a /16 VPC?

AWS reserves 5 addresses in each subnet: the first four and the last one.

- /24: 256 - 5 = **251**
- /28: 16 - 5 = **11**. This is also the smallest subnet AWS allows.
- /24 subnets in a /16: 2^8 = **256**

An interview trap is to answer 254. For more, see [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

**Problem 85.** A Kubernetes cluster uses the pod range 10.244.0.0/16. Each node gets a /24. How many nodes can the cluster address?

2^(24-16) = 2^8 = **256 nodes**. Each node has a block of 256 pod addresses.

The kubelet default limit is 110 pods per node, and you can change it. So the address block is often bigger than the pod limit. For more, see [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

**Problem 86.** An organisation has 2001:db8:abcd::/48. How many /64 subnets does it have? How many /64s does a /56 hold?

IPv6 uses the same math, with more bits.

- /48 to /64: 16 bits, so 2^16 = **65,536 subnets**.
- /56 to /64: 8 bits, so 2^8 = **256 subnets**.

IPv6 has no broadcast address, so the "minus 2" rule does not apply. A normal LAN uses /64. Router links may use /127 (RFC 6164). For more, see [IPv6 Basics](11-ipv6-basics.md).

**Problem 87.** Find the network and the last address for 2001:db8:abcd:1234:5678::1/60.

Each hex digit is 4 bits. 60 / 4 = 15, so keep the first 15 hex digits and zero the rest.

The first four groups have 16 hex digits: 2001, 0db8, abcd, 1234. Keep up to the 3 in 1234. Set the last digit of that group to 0.

- Network: **2001:db8:abcd:1230::/60**
- Last address: **2001:db8:abcd:123f:ffff:ffff:ffff:ffff**

It holds 16 /64 subnets, from :1230 to :123f.

**Problem 88.** List the /64 subnets inside 2001:db8:abcd:14::/62.

A /62 leaves 2 free bits in the fourth group, so there are 4 /64 subnets. Hex 14 is binary 0001 0100, and its last 2 bits are 0, so it is a valid start.

Answer: **2001:db8:abcd:14::/64, :15::/64, :16::/64 and :17::/64**.

**Problem 89.** Full design. You have 10.50.0.0/21. HQ has 600 users and 100 servers. A branch has 250 users and 100 IP phones. One /30 links HQ to the branch.

(a) Give a plan.

(b) HQ must hold one route for all of the branch. What is it?

(c) Write an ACL line that permits the server subnet.

(a) Plain largest-first order would place HQ servers between the branch subnets and break the summary. So keep the branch together first. Then apply largest first inside each area. This is what real designs do: think about summaries before you allocate.

| Group | Needs | Prefix | Subnet | Usable hosts |
|---|---|---|---|---|
| HQ users | 600 | /22 | 10.50.0.0/22 | 10.50.0.1 to 10.50.3.254 |
| Branch users | 250 | /24 | 10.50.4.0/24 | 10.50.4.1 to 10.50.4.254 |
| Branch phones | 100 | /25 | 10.50.5.0/25 | 10.50.5.1 to 10.50.5.126 |
| HQ servers | 100 | /25 | 10.50.6.0/25 | 10.50.6.1 to 10.50.6.126 |
| WAN link | 2 | /30 | 10.50.6.128/30 | 10.50.6.129 to 10.50.6.130 |

```text
10.50.0.0/21
 |
 +-- 10.50.0.0/22       HQ users
 +-- 10.50.4.0/23       branch block
 |    +-- 10.50.4.0/24      branch users
 |    +-- 10.50.5.0/25      branch phones
 |    +-- 10.50.5.128/25    free, kept for branch growth
 +-- 10.50.6.0/25       HQ servers
 +-- 10.50.6.128/30     WAN link
 +-- free: 10.50.6.132/30, 10.50.6.136/29, 10.50.6.144/28,
           10.50.6.160/27, 10.50.6.192/26, 10.50.7.0/24
```

(b) **10.50.4.0/23**. It covers both branch subnets and the free block kept for that branch.

(c) Server wildcard = 0.0.0.127:

```text
! Cisco IOS
access-list 60 permit 10.50.6.0 0.0.0.127
```

## Check Your Answers with Tools

Practise by hand first, because interviews have no calculator. Then use tools to check.

Python works the same on Linux and Windows:

```bash
# Linux
python3 -c "import ipaddress as i; n = i.ip_interface('172.16.35.123/20').network; print(n, n.broadcast_address, n.num_addresses - 2)"
```

```powershell
# Windows
python -c "import ipaddress as i; n = i.ip_interface('172.16.35.123/20').network; print(n, n.broadcast_address, n.num_addresses - 2)"
```

```text
172.16.32.0/20 172.16.47.255 4094
```

On Debian and Ubuntu, the `ipcalc` package gives a full view. Other distributions ship a different `ipcalc` with a different output format.

```bash
# Linux (Debian or Ubuntu ipcalc package)
ipcalc 172.16.35.123/20
```

```text
Address:   172.16.35.123        10101100.00010000.0010 0011.01111011
Netmask:   255.255.240.0 = 20   11111111.11111111.1111 0000.00000000
Wildcard:  0.0.15.255           00000000.00000000.0000 1111.11111111
=>
Network:   172.16.32.0/20       10101100.00010000.0010 0000.00000000
HostMin:   172.16.32.1          10101100.00010000.0010 0000.00000001
HostMax:   172.16.47.254        10101100.00010000.0010 1111.11111110
Broadcast: 172.16.47.255        10101100.00010000.0010 1111.11111111
Hosts/Net: 4094                  Class B, Private Internet
```

On a Cisco router, this shows the address and prefix of an interface:

```text
! Cisco IOS
R1# show ip interface GigabitEthernet0/0 | include Internet address
  Internet address is 172.16.35.123/20
```

## How to Say It Aloud

Interviewers often want to hear your thinking. Use the same order every time.

For Problem 24, say this: "/20 means the mask is 255.255.240.0. The interesting octet is the third, and the block size is 16. Multiples of 16 near 35 are 32 and 48, so the network is 172.16.32.0. The next subnet starts at 48, so the broadcast is 172.16.47.255. There are 12 host bits, so 4,094 usable hosts."

For a VLSM task, say this: "I sort the needs from largest to smallest. I pick the smallest prefix for each. I allocate from the start, so there are no gaps. Then I list what is free. If a summary is needed, I keep each site in one block first."

## Common Mistakes

- Forgetting the router, printer or other devices when counting hosts. See Problem 46.
- Forgetting the minus 2 for usable hosts. A /24 has 256 addresses but 254 usable hosts.
- Applying minus 2 on a /31. Both addresses are usable on a point-to-point link.
- Thinking .255 is always a broadcast or .0 is always a network. See Problems 31 and 33.
- Using the block size on the wrong octet. For /20, the block of 16 is in the third octet, not the fourth.
- Leaving the host octets unchanged. The network of 172.16.35.123/20 is 172.16.32.0, not 172.16.32.123.
- Starting a subnet on an address that is not a multiple of its block size.
- Saying "number of subnets" with 2^n - 2. That rule is out of date.
- Writing a subnet mask in an IOS ACL where a wildcard belongs.
- Summarising without checking that the first network is a multiple of the count. See Problem 72.
- Summarising over addresses that are in use elsewhere, or with no discard route. See Problem 77.
- Using 2^n - 2 in the cloud. AWS and Azure reserve 5 addresses per subnet, not 2.
- Doing VLSM smallest first, then running out of clean blocks.

## Quick-Fire Interview Questions

**Q: Why do we subtract 2 from the host count?**

A: The first address, with all host bits 0, names the subnet. The last address, with all host bits 1, is the broadcast. Neither is given to a device. The exceptions are /31 and /32.

**Q: How do you check if two hosts are in the same subnet?**

A: I AND each address with the mask, or find each network with the block size. If the two networks match, they are in the same subnet. Each host only uses its own mask, so a wrong mask on one host breaks this.

**Q: What is the difference between a subnet mask and a wildcard mask?**

A: In a mask, a 1 bit marks the network part. In a wildcard, a 0 bit means must match and a 1 bit means ignore. A wildcard is usually the inverse of the mask. Unlike a mask, a wildcard can have non-contiguous bits.

**Q: How do you pick a subnet size?**

A: I count all devices, including routers and virtual gateway addresses, and add growth. Then I pick the smallest block where 2^h - 2 is at least that number.

**Q: Why is VLSM done largest first?**

A: Each block must start on a multiple of its own size. Going largest first keeps every next block aligned, so there are no gaps and the free space stays together.

**Q: When is a route summary a bad idea?**

A: When it hides subnets that live elsewhere, or covers unused addresses with no discard route. That can send traffic to the wrong place or cause a loop. It also needs a clean address plan to work.

**Q: How many /64 subnets are in a /48?**

A: 65,536, because that is 16 bits of subnet space. IPv6 has no broadcast, so there is no minus 2 on the host side.

## Quick Recap

- Block size = 256 - the mask value in the interesting octet.
- A subnet always starts on a multiple of its block size. Most problems reduce to this one fact.
- Usable hosts = 2^(host bits) - 2, except /31, /32 and cloud subnets.
- Subnets from borrowing n bits = 2^n.
- Wildcard = 255 - each mask octet. A 0 bit must match, a 1 bit is ignored.
- A clean summary needs a power-of-2 count and a start that is a multiple of that count.
- VLSM goes largest first, but group each site in one block first if you plan to summarise.
