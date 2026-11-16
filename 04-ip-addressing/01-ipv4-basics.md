# IPv4 Addressing Basics

**Level:** Beginner | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [The TCP/IP Model](../02-network-models/03-tcp-ip-model.md)

## In One Line

An IPv4 address is a 32-bit number, written like 192.168.1.10, that identifies a network connection on a device and tells routers where to deliver data.

## Why It Matters

Data on a network travels in small pieces called packets. Every packet must say where it came from and where it is going. Without an address, a router (the device that moves packets between networks) has no idea where to send it.

IP addresses solve this problem. They give every connection a number, and they group those numbers by network, so a router can find any destination without knowing every device on Earth.

Interviewers ask about this first because everything else builds on it: subnetting, routing, NAT, firewalls, cloud networks. The first step of almost every network fault is also "what IP address does this machine have?". If your answer is shaky here, the rest of the interview is hard.

## Simple Explanation

Think of an IP address as a postal address.

- A letter needs an address so the post office can deliver it.
- An address has two parts: the street and the house number.
- The sorting office only looks at the street at first. The local postman uses the house number at the end.

An IP address works the same way:

- The **network part** is the street. Routers use it to get the packet to the right network.
- The **host part** is the house number. It picks one device on that network. A host is any device with an IP address: a laptop, phone, server or printer.

In 192.168.1.10, for a typical home network, 192.168.1 is the street and 10 is the house.

**Where the analogy stops:** on a real address you can see which words are the street. With an IP address you cannot. The address alone does not show where the street ends. A second number, the subnet mask, tells you. A device also gets a new address when it moves to a new network, even for a day, while your home address stays with your house.

## How It Works

### What an IP address is

IP stands for Internet Protocol. A protocol is an agreed set of rules for communication. IP is the rule set for addressing packets and moving them between networks. It works at Layer 3, the network layer.

IPv4 is version 4 of IP. It was defined in RFC 791 in 1981, and it still carries a large share of internet traffic. IPv6 is the newer version with much longer addresses, covered in [IPv6 Basics](11-ipv6-basics.md).

Every IPv4 packet carries two addresses in its header (the control data at the front of the packet):

- Source address: who sent it.
- Destination address: who should receive it.

The full header is in [IPv4 Header and Fragmentation](10-ipv4-header-and-fragmentation.md).

An IP address belongs to an **interface**, not to a whole device. An interface is one network connection point, such as a Wi-Fi adapter or an Ethernet port. A laptop using both Wi-Fi and Ethernet has two IP addresses. A router with four connected ports has four.

### 32 bits, split into 4 octets

A bit is a single 0 or 1. An IPv4 address is 32 bits long. To a computer, it looks like this:

```text
11000000101010000000000100001010
```

That is hard for people to read. So we cut it into 4 groups of 8 bits. Each group is called an **octet** (8 bits, which is also one byte).

```text
  Octet 1       Octet 2       Octet 3       Octet 4
  (bits 1-8)    (bits 9-16)   (bits 17-24)  (bits 25-32)
+-------------+-------------+-------------+-------------+
|  11000000   |  10101000   |  00000001   |  00001010   |
+-------------+-------------+-------------+-------------+
|     192     |     168     |      1      |     10      |
+-------------+-------------+-------------+-------------+
```

### Dotted decimal

**Dotted decimal** means we write each octet as a normal decimal number, with dots between them. So the bits above become 192.168.1.10.

Each bit position in an octet has a value. Add up the values where the bit is 1.

```text
Bit value:  128  64  32  16   8   4   2   1
---------------------------------------------
192    =     1   1   0   0   0   0   0   0     128+64
168    =     1   0   1   0   1   0   0   0     128+32+8
  1    =     0   0   0   0   0   0   0   1     1
 10    =     0   0   0   0   1   0   1   0     8+2
```

The smallest octet is 00000000, which is 0. The largest is 11111111, which is 255. So every octet must be in the range 0-255.

So 192.168.1.256 is not valid, because 256 does not fit in 8 bits. An address also needs exactly four octets, so 192.168.1 and 192.168.1.1.5 are not valid either.

Dotted decimal is only for humans. Computers handle the address as one 32-bit number. For example, 192.168.1.10 is the number 3,232,235,786.

### 4.3 billion addresses

Each bit doubles the number of possible values. With 32 bits you get:

```text
2^32 = 4,294,967,296 addresses (about 4.3 billion)
```

That sounded huge in 1981. Today there are more than 8 billion people, and many own several devices. There are not enough addresses, and not all of them can be used on the public internet anyway:

| Block | Addresses | Used for |
|---|---|---|
| 224.0.0.0/4 | 268,435,456 | Multicast (one sender, many receivers) |
| 240.0.0.0/4 | 268,435,456 | Reserved, not used on the public internet |
| 127.0.0.0/8 | 16,777,216 | Loopback (a device talking to itself) |
| 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | 17,891,328 | Private networks |

After these and a few smaller blocks, about 3.7 billion remain for public use. The details are in [Public, Private and Special IP Addresses](03-public-private-and-special-addresses.md).

IANA, the body at the top of the global address system, gave out its last free blocks in February 2011. The world kept going because of NAT, private addresses and IPv6, which you will meet later.

### Network part and host part

Every IPv4 address has two parts:

- **Network part:** identifies the network. All devices on the same network share it.
- **Host part:** identifies one interface on that network. It must be unique on that network.

The **subnet mask** shows where the split is. A mask is a 32-bit pattern. Its 1 bits cover the network part, and its 0 bits cover the host part.

```text
Address:      192 . 168 .   1 .  10
Mask:         255 . 255 . 255 .   0
              |---network---|   |host|
```

The same mask can be written in short form as /24, which means "the first 24 bits are the network part". This is called CIDR notation, or prefix notation. See [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md).

Here are three common splits:

| Address | Mask | Network part | Host part | Usable hosts |
|---|---|---|---|---|
| 10.20.30.40/8 | 255.0.0.0 | 10 | 20.30.40 | 16,777,214 |
| 172.16.5.20/16 | 255.255.0.0 | 172.16 | 5.20 | 65,534 |
| 192.168.1.10/24 | 255.255.255.0 | 192.168.1 | 10 | 254 |

Why "usable"? In a normal network, two host values are reserved:

- All host bits 0 is the **network address**, the name of the network. Example: 192.168.1.0.
- All host bits 1 is the **broadcast address**, which means "everyone on this network". Example: 192.168.1.255.

So a /24 has 256 addresses, and 254 can go to devices.

**Fuller truth:** the split does not have to fall on a dot. A mask such as /26 cuts through the middle of the last octet. That is what [Subnetting Step by Step](05-subnetting-step-by-step.md) teaches. Old "classes" fixed the split by the first number of the address, but modern networks do not work that way. See [IPv4 Classes](02-ipv4-classes.md).

### Why the split exists

Imagine that routers had to store one entry for every device on the internet. That would be billions of entries, changing all the time.

With a network part, a router stores one entry for a whole network. Only the last router needs to care about the host part. This is the same reason the post office sorts by city first.

A sending device uses the same split to make its first decision:

1. It compares the destination's network part with its own.
2. If they match, the destination is local. It delivers the packet directly.
3. If they differ, it sends the packet to its [default gateway](08-default-gateway.md), the router that leads out of the local network.

### IP address vs MAC address

A **MAC address** (Media Access Control address) is a 48-bit address used by Ethernet and Wi-Fi to move data between devices on the same local network. Details are in [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md).

| | IP address (IPv4) | MAC address |
|---|---|---|
| Layer | Layer 3 (network) | Layer 2 (data link) |
| Size | 32 bits | 48 bits |
| Written as | 192.168.1.10 | 00:00:5e:00:53:10 |
| Given by | Network admin or DHCP | Maker of the network card |
| Has a network part? | Yes, so routers can group it | No routing structure |
| Reach | End to end, across networks | One local link only |
| Changes when you move network? | Yes | No, normally |
| Changes at each router? | No (unless NAT rewrites it) | Yes |

Analogy: the IP address is your delivery address. The MAC address is the name on the local van that carries the parcel for one part of the trip. The delivery address stays on the parcel. The van changes at every depot.

**Where the analogy stops:** the "van" is really the frame, the wrapper around the packet at Layer 2. Each router removes the old frame and builds a new one.

Why do we need both?

- IP gives a location, so packets can cross many networks.
- MAC does the last short step on the local wire or radio link.

[ARP](../03-ethernet-and-switching/05-arp.md) links the two: it finds the MAC address that matches a local IP address.

**Fuller truth:** a MAC is not truly fixed. Software can change it, and modern phones and laptops often use a random MAC on Wi-Fi for privacy. Also, some links, such as point-to-point serial links, do not use MAC addresses at all.

### Static vs dynamic addresses

A device can get its IP address in two ways.

**Static:** a person types the settings in by hand. They stay until someone changes them.

**Dynamic:** the device asks a server for settings when it joins the network. This uses **DHCP** (Dynamic Host Configuration Protocol). The server gives an address on a **lease**, which is a loan for a set time. The device renews the lease before it ends. See [DHCP](../07-application-protocols/04-dhcp.md).

Both ways supply the same four values: IP address, subnet mask, default gateway, and DNS server (which turns names like example.com into IP addresses).

| | Static | Dynamic (DHCP) |
|---|---|---|
| Set by | A person | A DHCP server |
| Address can change? | No | Yes, though it often stays the same |
| Effort at scale | High | Low |
| Risk | Typing errors, duplicate addresses | Depends on the DHCP server |
| Good for | Routers, switches, servers | Laptops, phones, guest devices |

A middle option is a **DHCP reservation**. The DHCP server always gives the same address to the same MAC address. You get a fixed address, but you still manage it in one place. Many printers and small servers use this.

If a device asks for DHCP and gets no reply, it may give itself an address in 169.254.0.0/16. This range is called link-local, and Windows calls the feature APIPA (Automatic Private IP Addressing). If you see 169.254.x.x, read it as a warning: DHCP failed. The device can only talk to its own local link.

Static vs dynamic has nothing to do with public vs private. Any mix is possible:

| | Private | Public |
|---|---|---|
| Static | Office printer, 192.168.1.50 | Web server, 203.0.113.80 |
| Dynamic | Your laptop, 192.168.1.10 | Your home router's address from the ISP |

### Public IP vs local IP

A **public IP** is unique on the whole internet and can be reached across it. Your ISP (Internet Service Provider, the company that sells you internet) gives it to you.

A **private IP** comes from a block set aside by RFC 1918: 10.0.0.0/8, 172.16.0.0/12 or 192.168.0.0/16. Anyone can reuse these inside their own network. Routers on the public internet do not carry them.

Your **local IP** is the address on your own interface. On a home network, it is almost always a private address.

Analogy: a block of flats. The building has one street address. That is the public IP. Each flat has a number inside. That is the local IP. Many buildings have a flat 10, and that causes no problem.

**Where the analogy stops:** at a real building, the post carries both the street and the flat number. On a network, packets from inside leave with only the public address. Your router rewrites the source address and keeps a table so it can match the replies. This is **NAT** (Network Address Translation), covered in [NAT and PAT](09-nat-and-pat.md).

```text
   Private side (your home)              Public side

   Laptop 192.168.1.10 ---+
                          |   +----------------+
   Phone  192.168.1.11 ---+---| 192.168.1.1    |
                          |   |  Home router   |---- Internet
   TV     192.168.1.12 ---+   |  203.0.113.45  |
                              +----------------+

   All three devices appear on the internet as 203.0.113.45
```

**Fuller truth:** "local IP" is an everyday phrase, not a formal term. It usually means private, but not always. A server can have a public address directly on its interface, so its local IP and public IP are the same.

### How to find your IP address

**Windows**

```powershell
# Short summary
ipconfig

# Full detail: MAC, DHCP, lease times, DNS
ipconfig /all

# PowerShell view
Get-NetIPAddress -AddressFamily IPv4 |
  Select-Object InterfaceAlias, IPAddress, PrefixLength, PrefixOrigin
```

**Linux**

```bash
# All interfaces and addresses (short form: ip a)
ip addr show

# IPv4 only, one line per interface
ip -4 -brief addr

# Just the addresses
hostname -I

# Default gateway
ip route show default
```

The older `ifconfig` command still appears on many systems, but `ip` is the modern tool on Linux.

**macOS**

```bash
# IP of interface en0 (often Wi-Fi on a laptop)
ipconfig getifaddr en0

# Full details for that interface
ifconfig en0

# Which port name matches which interface
networksetup -listallhardwareports
```

**Phones**

- iPhone: Settings > Wi-Fi > tap the (i) next to your network.
- Android: Settings > Network and internet > Internet > tap your network. The menu names vary by phone maker.

**Cisco IOS (router or switch)**

```text
Router# show ip interface brief
Interface              IP-Address      OK? Method Status    Protocol
GigabitEthernet0/0     192.168.1.1     YES manual up        up
GigabitEthernet0/1     203.0.113.45    YES DHCP   up        up
```

The Method column shows how the address was set: `manual` means typed in, `DHCP` means leased, and `NVRAM` means loaded from the saved configuration. The spacing is shortened here to fit the page.

**Public IP (from any OS)**

Your device cannot see its public IP in its own settings when NAT is in use. Ask a server on the internet what source address it sees.

```bash
# Linux and macOS
curl https://api.ipify.org
```

```powershell
# Windows. Use curl.exe, because plain curl is an alias
# for Invoke-WebRequest in Windows PowerShell 5.1
curl.exe https://api.ipify.org
```

More commands are in [IP Configuration Commands](../12-troubleshooting-and-tools/02-ip-configuration-commands.md).

## Example

### Worked example 1: read and explain your Windows settings

You run `ipconfig` on a laptop.

```text
Windows IP Configuration


Wireless LAN adapter Wi-Fi:

   Connection-specific DNS Suffix  . : home
   IPv4 Address. . . . . . . . . . . : 192.168.1.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

Work through it:

1. **Address in bits:** 11000000.10101000.00000001.00001010.
2. **Mask:** 255.255.255.0 is /24, so the first 24 bits are the network part.
3. **Network part:** 192.168.1. **Host part:** 10.
4. **Network address:** set all host bits to 0, which gives 192.168.1.0.
5. **Broadcast address:** set all host bits to 1, which gives 192.168.1.255.
6. **Usable hosts:** 192.168.1.1 to 192.168.1.254, which is 254 addresses.
7. **Type:** 192.168.x.x is private, so this is a local address, not a public one.

Now test two destinations:

| Destination | Network part | Same as 192.168.1? | What the laptop does |
|---|---|---|---|
| 192.168.1.50 (printer) | 192.168.1 | Yes | Delivers it directly |
| 192.168.2.20 (file server) | 192.168.2 | No | Sends it to 192.168.1.1 |

Next, find out whether the address is static or dynamic with `ipconfig /all`.

```text
Wireless LAN adapter Wi-Fi:

   Physical Address. . . . . . . . . : 00-00-5E-00-53-10
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.1.10(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Friday, 30 October 2026 08:15:02
   Lease Expires . . . . . . . . . . : Saturday, 31 October 2026 08:15:02
   Default Gateway . . . . . . . . . : 192.168.1.1
   DHCP Server . . . . . . . . . . . : 192.168.1.1
   DNS Servers . . . . . . . . . . . : 192.168.1.1
```

`DHCP Enabled: Yes` and the lease lines tell you the address is dynamic. The lease here lasts 24 hours. The Physical Address is the MAC address.

Finally, compare it with the public address.

```powershell
curl.exe https://api.ipify.org
```

```text
203.0.113.45
```

The laptop's local IP is 192.168.1.10. The internet sees 203.0.113.45. They differ because the home router performs NAT.

If the router's status page shows the same 203.0.113.45 on its internet side, the router holds the public address. If it shows something in 100.64.0.0/10, such as 100.64.12.7, the ISP is sharing one public address among many customers. This is called CGNAT (Carrier-Grade NAT).

### Worked example 2: the same laptop on Linux and macOS

Linux:

```bash
ip addr show eth0
```

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:00:5e:00:53:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 86102sec preferred_lft 86102sec
```

The word `dynamic` and the countdown in `valid_lft` show a leased address. `link/ether` is the MAC, and `brd` is the broadcast address.

macOS:

```bash
ifconfig en0 | grep "inet "
```

```text
inet 192.168.1.10 netmask 0xffffff00 broadcast 192.168.1.255
```

macOS shows the mask in hexadecimal. 0xffffff00 is the same as 255.255.255.0, because ff in hex is 255.

### Worked example 3: IP stays, MAC changes

A PC in an office sends a packet to a server on another network. One router sits between them.

```text
   PC                     Router                    Server
   10.1.1.10         10.1.1.1 | 10.2.2.1            10.2.2.20
   MAC ...:11       MAC ...:01 | MAC ...:02         MAC ...:22
      |                    |        |                  |
      +------- Hop 1 ------+        +------ Hop 2 -----+
```

| Field | Hop 1 | Hop 2 |
|---|---|---|
| Source IP | 10.1.1.10 | 10.1.1.10 |
| Destination IP | 10.2.2.20 | 10.2.2.20 |
| Source MAC | ...:11 (PC) | ...:02 (router) |
| Destination MAC | ...:01 (router) | ...:22 (server) |

The IP addresses describe the whole trip. The MAC addresses describe one hop. A hop is one step between two neighbouring devices.

## Going Deeper

This section is optional on a first read.

**Who hands out addresses.** IANA (Internet Assigned Numbers Authority) gives large blocks to five Regional Internet Registries (RIRs): AFRINIC, APNIC, ARIN, LACNIC and RIPE NCC. RIRs give blocks to ISPs and large organisations. ISPs give addresses to customers. With free pools gone, organisations now also buy and sell IPv4 blocks through transfers.

**Exceptions to the two reserved addresses.** A /31 network has two addresses, and both are usable on a point-to-point link (RFC 3021). A /32 means one single address, often used for a router's loopback interface.

**Leading zeros are dangerous.** Some tools read an octet with a leading zero as octal (base 8). On such a tool, 010.0.0.1 means 8.0.0.1, not 10.0.0.1. Never pad octets with zeros in configuration.

## Common Mistakes

- Saying "a device has one IP address". The address belongs to an interface. A router has one per connected interface, and every device also has a loopback address.
- Thinking the first numbers alone show the network part. You need the mask. 10.1.2.3 could be /8, /16, /24 or anything else.
- Giving the same IP to two devices. This is an IP conflict, and both devices may lose connections. Keep static addresses outside the DHCP pool.
- Assigning the network or broadcast address to a host, such as 192.168.1.0/24 or 192.168.1.255/24.
- Forgetting the gateway on a static setup. Local traffic works, but nothing can leave the network.
- Treating 127.0.0.1 as your LAN address. It is loopback. It only ever reaches the same machine.
- Treating 169.254.x.x as a working address. It usually means the device could not reach a DHCP server.
- Confusing private with secure. A private address hides nothing by itself and does not replace a firewall.
- Confusing public with static. Most home connections get a public address that can change.
- Believing MAC addresses travel end to end. They are rewritten at every router.
- Believing a public IP lookup shows your device's own address. It shows the address the server saw, which may belong to your router, your ISP's CGNAT system, a VPN or a proxy.
- Saying IPv4 has 4.3 billion public addresses. That is the total. Large blocks are reserved.

## Interview Questions

**Q: What is an IP address?**

A: It is a logical address given to a network interface, so that devices can find each other across networks. In IPv4 it is a 32-bit number written in dotted decimal, such as 192.168.1.10. It has a network part and a host part, and routers use the network part to forward packets.

**Q: How many IPv4 addresses are there, and can all of them be used?**

A: There are 2 to the power of 32, which is about 4.3 billion. Not all are usable on the public internet. Large blocks are set aside for multicast, future use, loopback and private networks, so about 3.7 billion are left for public use.

**Q: What is the difference between the network part and the host part?**

A: The network part says which network the device is on, and all devices on that network share it. The host part picks one interface on that network. The subnet mask shows where one ends and the other begins. For 192.168.1.10/24, the network is 192.168.1.0 and the host is 10.

**Q: What is the difference between an IP address and a MAC address?**

A: An IP address is a logical Layer 3 address of 32 bits in IPv4. It is used end to end, and it changes when you move networks. A MAC address is a 48-bit Layer 2 address used only on the local link, and it is normally set by the card's maker. When a packet crosses a router, the IP addresses stay the same and the MAC addresses change.

**Q: If we have MAC addresses, why do we need IP addresses?**

A: MAC addresses have no routing structure, so a router could not group them and would need an entry for every device in the world. IP addresses are grouped by network, so routers can store one entry per network. IP finds the network, and MAC does the delivery on the local link.

**Q: What is the difference between a static and a dynamic IP address? When would you use each?**

A: A static address is set by hand and does not change. A dynamic address is leased from a DHCP server. I use static addresses or DHCP reservations for routers, servers and printers, because others need to find them at a known address. I use DHCP for laptops and phones, because it scales and avoids typing errors.

**Q: A user's PC shows 169.254.20.5. What does that tell you?**

A: That is a link-local address. The PC asked for a DHCP address and got no answer, so it gave itself one. I would check the cable or Wi-Fi connection, whether the port is on the right network, and whether the DHCP server is running and has free addresses.

**Q: What is the difference between a public and a private IP address?**

A: A public address is globally unique and routable on the internet. A private address comes from the RFC 1918 ranges: 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. These can be reused in any private network, and they need NAT to reach the internet.

**Q: Your laptop shows 192.168.1.10, but a website says your IP is 203.0.113.45. Why?**

A: The laptop shows its local private address. The website shows the address it received traffic from, which is the public address after NAT on the router. Both are correct. They describe the two sides of the router.

**Q: Can two devices have the same IP address?**

A: Not on the same network, because that causes a conflict. In two separate private networks, yes. Millions of homes use 192.168.1.10. NAT keeps them apart on the internet, since each home has a different public address.

**Q: How do you find your IP address on Windows, Linux and macOS?**

A: On Windows I use `ipconfig`, or `ipconfig /all` for more detail. On Linux I use `ip addr`. On macOS I use `ipconfig getifaddr en0` or `ifconfig`. For the public address, I query a service such as `curl https://api.ipify.org`, because NAT hides it from the device.

## Quick Recap

- An IPv4 address is a 32-bit number written as four octets, each 0-255, such as 192.168.1.10.
- There are 2^32, or about 4.3 billion, addresses in total, and fewer than that are usable in public.
- The subnet mask splits an address into a network part (for routers) and a host part (for the device).
- IP is Layer 3, works end to end and changes with your network. MAC is Layer 2, works on one link and changes at each router.
- Static addresses are set by hand. Dynamic addresses are leased through DHCP. 169.254.x.x means DHCP failed.
- Your local IP is usually private. Your public IP is what the internet sees, usually after NAT.
- Use `ipconfig`, `ip addr` or `ifconfig` for the local IP, and an outside service for the public IP.
