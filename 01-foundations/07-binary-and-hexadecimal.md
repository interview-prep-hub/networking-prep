# Binary and Hexadecimal for Networking

**Level:** Beginner | **Prerequisites:** None

## In One Line

Network devices work only with bits (0s and 1s), and people write those bits in shorter forms: decimal for IPv4, hexadecimal for MAC and IPv6 addresses.

## Why It Matters

An IP address looks like four normal numbers, such as 192.168.10.130. A router does not see that. It sees 32 bits.

Subnets, masks and routing decisions all happen at the bit level. If you cannot see the bits, subnetting feels like magic and you can only memorise. If you can see them, subnetting becomes simple arithmetic.

Hexadecimal is just as common in daily work. You see it in MAC addresses, IPv6 addresses, packet captures and error logs.

Interviewers ask about this for three reasons:

- They want to see you convert a number quickly by hand, for example 192 to binary.
- They want to know that you understand what a subnet mask really does (bitwise AND).
- Binary is the base of [subnetting](../04-ip-addressing/05-subnetting-step-by-step.md). Weak binary skills show up as weak subnetting skills.

## Simple Explanation

Picture a row of 8 light switches. Each switch has a label: 128, 64, 32, 16, 8, 4, 2, 1.

A switch is either off (0) or on (1). To read the row, add the labels of the switches that are on.

```text
Label:   128   64   32   16    8    4    2    1
Switch:   on  off   on  off   on   on  off  off
Bit:       1    0    1    0    1    1    0    0

Add the "on" labels: 128 + 32 + 8 + 4 = 172
```

So the binary number 10101100 means 172. That is the whole idea of binary.

**Where the analogy stops:** a real bit on a cable is not a switch that stays in place. It is a short signal, such as a voltage or a pulse of light, that passes by and is gone. The idea of "two states" is what carries over.

Hexadecimal is a short way to write bits. Think of a long phone number. People split it into small groups so it is easy to read. Hex does the same thing: every group of 4 bits becomes one symbol.

**Where the analogy stops:** a phone number group has no maths behind it. Hex is a real number system, so you can count and calculate in it.

## How It Works

### Why computers use binary

A computer chip is made of billions of tiny electronic switches called transistors. A transistor is either passing current or not. Two states are easy to build and easy to tell apart, even with some electrical noise.

If a chip had to separate ten voltage levels (for digits 0-9), small noise could turn one digit into another. With two levels, there is a wide safety gap between them.

So all data is stored as bits: text, images, addresses and packets.

**Fuller truth:** some fast links send more than two signal levels. PAM4, used in many high-speed Ethernet links, has four levels, so one symbol carries 2 bits. The data itself is still bits.

### Bits, bytes and octets

| Term | Size | Meaning |
|---|---|---|
| Bit | 1 binary digit | The smallest unit: 0 or 1 |
| Nibble | 4 bits | Half a byte, equal to one hex digit |
| Byte | 8 bits (in practice) | The usual unit of computer storage |
| Octet | Exactly 8 bits | The word network standards use |

Why have both "byte" and "octet"? Some old computers used bytes that were not 8 bits long. Standards writers wanted a word with only one meaning, so they said "octet". Today the two words mean the same thing in daily use.

Watch the letter case. A small **b** means bits and a large **B** means bytes. Link speeds use bits, so 100 Mbps is 100 megabits per second. File sizes use bytes. Since 1 byte is 8 bits, a 100 Mbps link moves at most 12.5 MB per second. More on this in [Bandwidth, Latency and Performance](06-network-performance-basics.md).

### What n bits can hold

Each extra bit doubles the number of possible values. With n bits you get 2^n values, from 0 up to 2^n - 1.

| Bits | Values (2^n) | Largest number | Where you see it |
|---|---|---|---|
| 1 | 2 | 1 | A single flag, on or off |
| 4 | 16 | 15 | One hex digit |
| 8 | 256 | 255 | One IPv4 octet |
| 16 | 65,536 | 65,535 | [Port numbers](../06-transport-layer/01-ports-and-sockets.md) |
| 32 | 4,294,967,296 | 4,294,967,295 | IPv4 address |
| 48 | about 281 trillion | 2^48 - 1 | MAC address |
| 128 | about 3.4 x 10^38 | 2^128 - 1 | IPv6 address |

This table answers a classic question: why can an IPv4 octet not be 256? Because 8 bits stop at 255. There are 256 values, but counting starts at 0.

### Place values

In decimal (base 10), each position is worth 10 times the one on its right: 1, 10, 100. The "base" is how many digits a number system uses.

Binary is base 2. Each position is worth 2 times the one on its right.

```text
Power:   2^7  2^6  2^5  2^4  2^3  2^2  2^1  2^0
Value:   128   64   32   16    8    4    2    1
```

Learn this row by heart. All 8 bits on gives 128 + 64 + 32 + 16 + 8 + 4 + 2 + 1 = 255.

### Binary to decimal

1. Write the 128-64-32-16-8-4-2-1 row above the 8 bits.
2. Mark each place that has a 1.
3. Add those place values.

Example: 10110101

```text
128   64   32   16    8    4    2    1
  1    0    1    1    0    1    0    1

128 + 32 + 16 + 4 + 1 = 181
```

### Decimal to binary

Work from the left. You have one banknote of each value, and you must pay the exact amount, largest note first.

1. Start at 128. Ask: does this place value fit in what is left?
2. If yes, write 1 and subtract it.
3. If no, write 0 and leave the number as it is.
4. Move to the next place and repeat until you finish the 1 place.

Example: 172

```text
Place   Does it fit?   Bit   Left over
128     yes            1     172 - 128 = 44
64      no             0     44
32      yes            1     44 - 32 = 12
16      no             0     12
8       yes            1     12 - 8 = 4
4       yes            1     4 - 4 = 0
2       no             0     0
1       no             0     0

Read the Bit column from top to bottom: 10101100
```

Always write all 8 bits in an octet. The number 10 is 00001010, not 1010.

**Where the analogy stops:** with real money you can use two notes of the same value. In binary each place is used once at most, because a bit can only be 0 or 1.

### The mask values to memorise

Subnet masks use ones on the left and zeros on the right. So only these values appear in a mask octet. Knowing them saves time in every subnetting question.

| Binary | Decimal | Binary | Decimal |
|---|---|---|---|
| 00000000 | 0 | 11111000 | 248 |
| 10000000 | 128 | 11111100 | 252 |
| 11000000 | 192 | 11111110 | 254 |
| 11100000 | 224 | 11111111 | 255 |
| 11110000 | 240 | | |

### Hexadecimal

Hexadecimal, or hex, is base 16. It needs 16 digits, so it uses 0-9 and then the letters A-F. Case does not matter: `af` and `AF` are the same value.

| Dec | Hex | Binary | Dec | Hex | Binary |
|---|---|---|---|---|---|
| 0 | 0 | 0000 | 8 | 8 | 1000 |
| 1 | 1 | 0001 | 9 | 9 | 1001 |
| 2 | 2 | 0010 | 10 | A | 1010 |
| 3 | 3 | 0011 | 11 | B | 1011 |
| 4 | 4 | 0100 | 12 | C | 1100 |
| 5 | 5 | 0101 | 13 | D | 1101 |
| 6 | 6 | 0110 | 14 | E | 1110 |
| 7 | 7 | 0111 | 15 | F | 1111 |

The key fact: **one hex digit is exactly 4 bits.** So one byte is always exactly two hex digits.

The prefix `0x` tells the reader that a number is hex. `10` is ten, but `0x10` is sixteen. MAC and IPv6 addresses skip this prefix because everyone knows they are hex.

### Binary and hex conversion

**Binary to hex:**

1. Split the bits into groups of 4, starting from the right.
2. Turn each group into one hex digit.

```text
Binary:   1011 0101
Hex:         B    5      so 10110101 = 0xB5
```

**Hex to binary:**

1. Turn each hex digit into exactly 4 bits. Keep the leading zeros.
2. Join the groups.

```text
Hex:         2    F
Binary:   0010 1111      so 0x2F = 00101111
```

### Hex and decimal conversion

**Hex to decimal:** in a two-digit hex number, the left digit is worth 16 times its value and the right digit is worth 1 times its value.

```text
0xB5  =  (11 x 16) + 5  =  176 + 5  =  181
0x2F  =  (2 x 16) + 15  =  32 + 15  =  47
```

Longer numbers use more place values: 4096, 256, 16, 1. For example, 0x86DD is (8 x 4096) + (6 x 256) + (13 x 16) + 13 = 34525.

**Decimal to hex:** divide by 16. The answer is the left digit and the remainder is the right digit.

```text
200 / 16  =  12 remainder 8
12 is C, so 200 = 0xC8
```

You can also go through binary: 200 is 1100 1000, which is C8.

Values worth knowing on sight: 0x0F = 15, 0x10 = 16, 0x7F = 127, 0x80 = 128, 0xC0 = 192, 0xE0 = 224, 0xF0 = 240, 0xFF = 255.

### Why IPv4 is shown in decimal, and MAC and IPv6 in hex

The device always uses bits. The written form exists only for people.

| Address | Bits | Written as | Example |
|---|---|---|---|
| IPv4 | 32 | 4 decimal octets | 192.168.10.130 |
| MAC | 48 | 12 hex digits | 00:00:5E:00:53:AF |
| IPv6 | 128 | 32 hex digits in 8 groups | 2001:0db8:0000:0000:0000:0000:0000:0001 |

**IPv4 is short.** Four small decimal numbers are easy to read and easy to say out loud.

**Bits still matter for IPv4.** Masks and routing work on bits, and network boundaries often fall inside an octet. That is why you must convert to binary to subnet.

**MAC and IPv6 are long.** IPv6 in dotted decimal would be 16 numbers:

```text
Hex:      2001:db8::1
Decimal:  32.1.13.184.0.0.0.0.0.0.0.0.0.0.0.1
```

Hex is shorter. It also lines up cleanly with bits, because each digit is 4 bits. An IPv6 /64 prefix is simply the first 16 hex digits. Decimal has no such clean match: you cannot tell which bits are set in 172 without working it out.

The `::` above is a short form for a run of zeros, explained in [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md). For more on hardware addresses, see [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md).

The same MAC address also looks different across systems. Only the punctuation changes, not the value:

| System | Style |
|---|---|
| Linux | `00:00:5e:00:53:af` |
| Windows | `00-00-5E-00-53-AF` |
| Cisco IOS | `0000.5e00.53af` |

### Bitwise AND

"Bitwise" means the operation works on one pair of bits at a time, with no carrying between columns. AND gives 1 only when both bits are 1.

| A | B | A AND B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

A subnet mask is a 32-bit pattern. Ones mark the network part of an address, and zeros mark the host part. When you AND an address with its mask:

- Where the mask has a 1, the address bit stays the same.
- Where the mask has a 0, the result becomes 0.

The result is the **network address**, the number that names the whole subnet.

Think of the mask as a stencil laid over the address. Holes (ones) let the bits show through. Solid parts (zeros) cover them.

**Where the analogy stops:** covered bits do not disappear. They become zeros, and the result is still a full 32-bit address.

Two more operations appear in networking:

- **OR** gives 1 if either bit is 1. Network address OR inverted mask gives the broadcast address.
- **NOT** flips every bit. Flipping 255.255.255.192 gives 0.0.0.63, which is the [wildcard mask](../09-security/05-access-control-lists.md) used in Cisco ACLs.

## Example

### Worked example: find the network address

A host has the address 192.168.10.130 with mask 255.255.255.192 (a /26, which means the first 26 bits are ones).

**Step 1: convert each octet to binary.**

```text
Decimal:   192       168       10        130
Hex:       C0        A8        0A        82
Binary:    11000000  10101000  00001010  10000010
```

**Step 2: write the mask under the address and AND each column.**

```text
Address    11000000  10101000  00001010  10000010
Mask       11111111  11111111  11111111  11000000
           --------  --------  --------  --------
AND        11000000  10101000  00001010  10000000
Decimal    192       168       10        128
```

The network address is **192.168.10.128**.

**Step 3: read the rest from the host bits.** The host part is the last 6 bits.

- All host bits 0 gives the network address: 192.168.10.128
- All host bits 1 gives the broadcast address: 10111111 = 192.168.10.191
- Usable hosts are the addresses between them: 192.168.10.129 to 192.168.10.190

**Step 4: use it.** The host wants to reach 192.168.10.70, so it checks that address with the same mask.

```text
70         01000110
Mask       11000000
AND        01000000  =  64
```

The other address is in network 192.168.10.64, and ours is in 192.168.10.128. They are different, so the packet goes to the [default gateway](../04-ip-addressing/08-default-gateway.md).

### Check your work with commands

Linux (bash):

```bash
echo $((2#11000000))              # binary to decimal: 192
echo $((16#AF))                   # hex to decimal: 175
printf '%02X\n' 175               # decimal to hex: AF
echo 'obase=2; 172' | bc          # decimal to binary: 10101100
echo $((130 & 192))               # bitwise AND: 128
```

Windows (PowerShell):

```powershell
[Convert]::ToInt32('11000000', 2)            # binary to decimal: 192
[Convert]::ToInt32('AF', 16)                 # hex to decimal: 175
'{0:X2}' -f 175                              # decimal to hex: AF
[Convert]::ToString(172, 2)                  # decimal to binary: 10101100
[Convert]::ToString(10, 2).PadLeft(8, '0')   # keep 8 bits: 00001010
130 -band 192                                # bitwise AND: 128
```

Any OS with Python 3 (use `python` on Windows if `python3` is not found):

```bash
python3 -c "import ipaddress; print(ipaddress.ip_interface('192.168.10.130/26').network)"
```

```text
192.168.10.128/26
```

Tools are good for checking. Most interviews still expect you to do this by hand.

### Practice problems

Work these on paper. Answers are in the next section.

**Set A: binary to decimal**

1. 00001111
2. 10000001
3. 11110000
4. 10101010
5. 01100100
6. 11111100

**Set B: decimal to binary (write 8 bits)**

7. 7
8. 64
9. 99
10. 127
11. 200
12. 224

**Set C: hexadecimal**

13. 0x1F to decimal
14. 0xC8 to decimal
15. 0xE0 to binary
16. 11001010 to hex
17. 172 to hex
18. 250 to hex

**Set D: addresses**

19. Write 172.16.5.9 in binary.
20. Write 10.0.0.1 in hex, octet by octet.
21. Write 11000000.10101000.00000001.11111110 in dotted decimal.
22. How many hex digits are in a full IPv6 address? How many bits are in each of its 8 groups?
23. The last byte of a MAC address is 2C. What is it in decimal and in binary?
24. Write the IPv6 group `0db8` in binary.

**Set E: bitwise AND**

25. 200 AND 240
26. Find the network address of 10.20.30.77 with mask 255.255.255.224.
27. Find the network address of 172.16.45.200 with mask 255.255.240.0.
28. With mask 255.255.255.192, are 192.168.1.60 and 192.168.1.70 in the same subnet?

### Answers

| No. | Answer | Working |
|---|---|---|
| 1 | 15 | 8 + 4 + 2 + 1 |
| 2 | 129 | 128 + 1 |
| 3 | 240 | 128 + 64 + 32 + 16 |
| 4 | 170 | 128 + 32 + 8 + 2 |
| 5 | 100 | 64 + 32 + 4 |
| 6 | 252 | 255 - 3 |
| 7 | 00000111 | 4 + 2 + 1 |
| 8 | 01000000 | 64 only |
| 9 | 01100011 | 64 + 32 + 2 + 1 |
| 10 | 01111111 | 128 is off, the rest are on |
| 11 | 11001000 | 128 + 64 + 8 |
| 12 | 11100000 | 128 + 64 + 32 |
| 13 | 31 | (1 x 16) + 15 |
| 14 | 200 | (12 x 16) + 8 |
| 15 | 11100000 | E = 1110, 0 = 0000 |
| 16 | 0xCA | 1100 = C, 1010 = A |
| 17 | 0xAC | 10101100, or 172 / 16 = 10 remainder 12 |
| 18 | 0xFA | 250 / 16 = 15 remainder 10 |
| 19 | 10101100.00010000.00000101.00001001 | Convert each octet by itself |
| 20 | 0A.00.00.01 | 10 is 0A, and each octet has two digits |
| 21 | 192.168.1.254 | Last octet is 255 - 1 |
| 22 | 32 digits, 16 bits | 128 / 4 = 32 and 128 / 8 = 16 |
| 23 | 44, 00101100 | (2 x 16) + 12 = 44 |
| 24 | 0000 1101 1011 1000 | 0, d, b, 8 in turn |
| 25 | 192 | 11001000 AND 11110000 = 11000000 |
| 26 | 10.20.30.64 | 77 = 01001101, AND 11100000 = 01000000 |
| 27 | 172.16.32.0 | 45 = 00101101, AND 11110000 = 00100000 |
| 28 | No | 60 gives network .0, and 70 gives network .64 |

## Going Deeper

### Reading bits in packet headers

Header fields are often only a few bits wide, so a capture tool prints them in hex. Your conversion skills apply directly.

The first four bytes of a typical IPv4 header might be `45 00 00 3C`.

```text
Byte value   Bits        Meaning
0x45         0100 0101   Left nibble 4 = IP version 4
                         Right nibble 5 = header length in 4-byte
                         units, so 5 x 4 = 20 bytes
0x00         0000 0000   DSCP and ECN (traffic marking), none set
0x003C       (2 bytes)   Total length = (3 x 16) + 12 = 60 bytes
```

See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

TCP flags are one byte, and the same place-value row works:

```text
Value:   128  64   32   16   8    4    2    1
Flag:    CWR  ECE  URG  ACK  PSH  RST  SYN  FIN

0x02 = 0000 0010 = SYN
0x12 = 0001 0010 = SYN + ACK (16 + 2)
0x18 = 0001 1000 = PSH + ACK (16 + 8)
```

See [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md). This is also why [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) filters such as `tcp[13] & 2 != 0` work: byte 13 of the TCP header holds the flags, and the AND tests the SYN bit.

### Single bits with meaning in a MAC address

The last two bits of the first MAC byte have special jobs:

- The 1 place: 0 means a single device (unicast), 1 means a group. So an odd first byte means group. 01:00:5E:00:00:01 is a multicast MAC, and FF:FF:FF:FF:FF:FF is broadcast.
- The 2 place: 1 means locally assigned, not set by the maker. The address 02:00:00:00:00:01 is an example.

### Known hex values

Ethernet frames use a 2-byte EtherType field to name what is inside: 0x0800 is IPv4, 0x0806 is ARP and 0x86DD is IPv6. See [Ethernet Frames](../03-ethernet-and-switching/02-ethernet-frames.md).

Some systems also print masks in hex. On macOS, `ifconfig` shows 255.255.255.0 as `0xffffff00`.

### Byte order

Multi-byte values on the wire are sent with the most significant (largest) byte first. This is called network byte order, or big-endian. Many CPUs, including x86, store values the other way (little-endian). That is why network code converts values with functions such as `htons()`. If a port number looks wrong in your own code, check the byte order first.

## Common Mistakes

- **Reading the row backwards.** The 128 place is on the left, and the 1 place is on the right.
- **Dropping leading zeros.** An octet always has 8 bits. Writing 10 as 1010 will break your column alignment when you AND it with a mask.
- **Saying an octet can hold 256.** It has 256 values, but the highest is 255.
- **Converting a whole IP as one number.** Convert each octet by itself.
- **Reading 0x10 as ten.** It is sixteen.
- **Losing zeros in hex to binary.** 0x2F is 0010 1111, not 10 1111.
- **Treating AND like addition.** 1 AND 1 is 1, not 2. Nothing carries to the next column.
- **Mixing up Mb and MB.** Bits measure speed, and bytes measure file size. There is a factor of 8 between them.
- **Thinking an IPv6 group is one byte.** Each group is 4 hex digits, which is 16 bits or 2 bytes.
- **Thinking an IP address is "really" decimal.** Decimal is only how it is displayed. The device uses 32 bits.

## Interview Questions

**Q: Why do computers use binary?**
A: Chips are built from transistors, which are either on or off. Two states are easy to build and hard to confuse, even with electrical noise. So all data, including packets, is stored and processed as bits.

**Q: What is the difference between a bit, a byte and an octet?**
A: A bit is a single 0 or 1. A byte is 8 bits in practice. An octet is exactly 8 bits, and network standards prefer that word because it has only one meaning.

**Q: Why is the highest value in an IPv4 octet 255?**
A: An octet has 8 bits, so it has 2^8 = 256 values. Counting starts at 0, so the range is 0-255.

**Q: Convert 192 to binary. Then convert 11111100 to decimal.**
A: 192 is 128 + 64, so it is 11000000. 11111100 is 128 + 64 + 32 + 16 + 8 + 4, which is 252.

**Q: How many bits are in IPv4, MAC and IPv6 addresses?**
A: IPv4 has 32 bits, MAC has 48 bits and IPv6 has 128 bits.

**Q: Why are MAC and IPv6 addresses written in hex?**
A: They are long, and hex keeps them short. One hex digit is exactly 4 bits, so hex maps cleanly onto the underlying bits. IPv4 is short enough to read in dotted decimal.

**Q: What is 0xFF in decimal and in binary?**
A: It is 255 in decimal and 11111111 in binary. Each F is four 1 bits.

**Q: What does a subnet mask do at the bit level?**
A: Its ones mark the network part and its zeros mark the host part. A bitwise AND of the address and mask keeps the network bits and zeros the host bits, which gives the network address.

**Q: How does a host decide if a destination is on its local subnet?**
A: It applies its mask to its own address and to the destination. If both results match, the destination is local. If not, it sends the packet to the default gateway. Modern systems do this through a routing table lookup, but the bit logic is the same.

**Q: What is the network address of 192.168.10.130/26?**
A: The /26 mask has a last octet of 192. 130 is 10000010, and AND with 11000000 gives 10000000, which is 128. So the network is 192.168.10.128.

## Quick Recap

- Devices use bits. Decimal and hex are written forms for people.
- A byte or octet is 8 bits, so it holds 0-255. The place values are 128, 64, 32, 16, 8, 4, 2, 1.
- To convert decimal to binary, subtract place values from the left. To convert binary to decimal, add the places that hold a 1.
- One hex digit is exactly 4 bits, and one byte is two hex digits.
- IPv4 is 32 bits (dotted decimal), MAC is 48 bits (hex) and IPv6 is 128 bits (hex).
- Address AND mask gives the network address. This is the base of all subnetting.
