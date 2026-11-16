# MAC Addresses

**Level:** Beginner | **Prerequisites:** [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md), [The OSI Model](../02-network-models/02-osi-model.md)

## In One Line

A MAC address is a 48-bit address that identifies a network interface on the local network, and devices use it to deliver data over one link at a time.

## Why It Matters

Many devices can share one local network. When data arrives, each device must know: "Is this for me?" The MAC address answers that question.

MAC stands for Media Access Control. It works at Layer 2 of the [OSI model](../02-network-models/02-osi-model.md), the layer that moves data across one link.

Every Ethernet and Wi-Fi device uses MAC addresses, all day, for every packet. Without them, your laptop could not even reach your home router.

Interviewers like this topic because it tests whether you understand how networks really deliver data. Expect these questions:

- Why do we need both a MAC address and an IP address?
- What changes when a packet crosses a router?
- What is the broadcast MAC address?
- Why does a phone show a different MAC address on each Wi-Fi network?

## Simple Explanation

Think of sending a parcel from one city to another.

The box has the final address written on it. That is the **IP address**. It stays the same for the whole trip.

The parcel does not fly straight to the house. It moves in short trips: van to depot, depot to truck, truck to local van. For each short trip, a sticker says "from this vehicle, to that vehicle". That sticker is the **MAC address** pair. A new sticker goes on at every depot.

So:

- IP address: where the data must finally go.
- MAC address: who hands it to whom on this one link.

**Where the analogy stops:** a sticker is written fresh for each trip, but a MAC address belongs to a network interface and is normally stored in advance. The analogy shows how MAC addresses are used, not where they come from.

## How It Works

### Terms you need first

- **Bit:** a single 0 or 1.
- **Byte:** 8 bits.
- **Hexadecimal (hex):** a way to write numbers with digits 0-9 and letters A-F. One hex digit equals 4 bits, so two hex digits equal one byte.
- **NIC (network interface card):** the hardware that connects a device to a network. It can be a separate card or a chip inside a phone or laptop.
- **Frame:** the unit of data at Layer 2. A frame carries the source and destination MAC addresses.
- **LAN (local area network):** a network in one place, such as a home or office.

### Format: 48 bits, 6 bytes, 12 hex digits

A MAC address is 48 bits long. We write it as 6 bytes, which is 12 hex digits.

48 bits give 2^48 possible values. That is 281,474,976,710,656, or about 281 trillion.

The same address can be written in three common styles:

| Style | Example | Where you see it |
|---|---|---|
| Hyphens | `00-00-0C-12-34-56` | Windows |
| Colons | `00:00:0c:12:34:56` | Linux, macOS |
| Dots, in groups of 4 | `0000.0c12.3456` | Cisco IOS |

All three are the same address. Upper case and lower case mean the same thing too.

### The two parts: OUI and device part

```text
  Byte:     1      2      3      4      5      6
         +------+------+------+------+------+------+
         |  00  |  00  |  0C  |  12  |  34  |  56  |
         +------+------+------+------+------+------+
         |<------ OUI ------->|<--- device part -->|
              (vendor)           (chosen by vendor)
```

The first 3 bytes are the **OUI (Organizationally Unique Identifier)**. The IEEE, the standards body behind Ethernet and Wi-Fi, assigns this prefix to a company. `00:00:0C` belongs to Cisco.

The last 3 bytes are chosen by the company. Each OUI gives 2^24 addresses, which is 16,777,216. Large vendors hold many OUIs.

**Analogy:** think of a car's serial number. The first part shows the maker. The rest identifies the single car.

**Where the analogy stops:** a car's number is hard to change. A MAC address can be changed in software, as you will see below.

The address set in the hardware at the factory is called the **burned-in address (BIA)**.

**Fuller truth:** the OUI often names the maker of the network chip, not the brand of the device. A Dell laptop can show an Intel OUI. Smaller blocks with longer prefixes also exist. See [Going Deeper](#going-deeper).

### The two special bits

Two bits in the **first byte** have a special meaning. They are the two lowest bits of that byte.

```text
  First byte of the address (example value 0x5A)

   bit:    7   6   5   4   3   2   1   0
         +---+---+---+---+---+---+---+---+
   value:| 0 | 1 | 0 | 1 | 1 | 0 | 1 | 0 |
         +---+---+---+---+---+---+---+---+
                                   |   |
            U/L bit (1 = local)----+   +---- I/G bit (1 = group)
```

| Bit | Name | Value 0 means | Value 1 means |
|---|---|---|---|
| Bit 0 | I/G (Individual/Group) | Unicast: one interface | Multicast: a group of interfaces |
| Bit 1 | U/L (Universal/Local) | Globally unique, assigned through a vendor | Locally administered, set by software or an admin |

**Unicast** means one sender to one receiver. **Multicast** means one sender to a group of receivers that asked for it.

A **locally administered** address does not come from the vendor's block. Software or a person chose it. Virtual machines and randomised phone addresses use this kind of address.

### The quick trick: read the second hex digit

You do not need to convert the whole address to binary. Look at the second hex digit of the address. Those 4 bits hold both special bits.

| Second hex digit | Bit 1 (U/L) | Bit 0 (I/G) | Meaning |
|---|---|---|---|
| 0, 4, 8, C | 0 | 0 | Global unicast (normal factory address) |
| 2, 6, A, E | 1 | 0 | Local unicast (randomised or set by software) |
| 1, 5, 9, D | 0 | 1 | Global multicast |
| 3, 7, B, F | 1 | 1 | Local multicast |

### The broadcast address

The broadcast MAC address is all 48 bits set to 1:

```text
FF:FF:FF:FF:FF:FF
```

A frame sent to this address goes to every device on the local network. A switch floods it out of all ports in the same VLAN, except the port it came from. A router does not forward it. The edge of this area is the [broadcast domain](04-collision-and-broadcast-domains.md).

Two well-known users are the ARP request and the DHCP Discover message (see [ARP](05-arp.md) and [DHCP](../07-application-protocols/04-dhcp.md)).

**Fuller truth:** broadcast is a special case of a group address. Its I/G bit is 1, like every other group address.

### Well-known special addresses

| MAC address | Used for |
|---|---|
| `FF:FF:FF:FF:FF:FF` | Broadcast |
| `01:00:5E:xx:xx:xx` | IPv4 multicast (only `01:00:5E:00:00:00` to `01:00:5E:7F:FF:FF`) |
| `33:33:xx:xx:xx:xx` | IPv6 multicast |
| `01:80:C2:00:00:00` | Spanning Tree BPDUs (IEEE standard) |
| `01:80:C2:00:00:02` | LACP (Slow Protocols) |
| `01:80:C2:00:00:0E` | LLDP |
| `01:00:0C:CC:CC:CC` | Cisco CDP, VTP and DTP |

### What a NIC does with each frame

1. A frame arrives at the NIC.
2. The NIC reads the destination MAC address.
3. If it matches the NIC's own address, the NIC accepts the frame.
4. If it is the broadcast address, the NIC accepts the frame.
5. If it is a multicast address this device has joined, the NIC accepts the frame.
6. In all other cases, the NIC drops the frame. The CPU never has to look at it.

**Fuller truth:** a NIC in **promiscuous mode** accepts every frame it sees. Capture tools such as [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) use this mode.

A source MAC address is always unicast. Only a destination can be a group address.

### MAC vs IP: why we need both

| | MAC address | IP address |
|---|---|---|
| OSI layer | Layer 2 (data link) | Layer 3 (network) |
| Size | 48 bits | 32 bits (IPv4), 128 bits (IPv6) |
| Set by | Vendor, or software | Admin, or [DHCP](../07-application-protocols/04-dhcp.md) |
| Structure | Flat: no location information | Hierarchical: network part and host part |
| Reach | One link or one LAN | End to end, across networks |
| Changes on the way? | Yes, at every router | No (unless [NAT](../04-ip-addressing/09-nat-and-pat.md) rewrites it) |

Here are the real reasons we need both:

1. **MAC addresses cannot be summarised.** Two NICs with close addresses can be on opposite sides of the world. A router would need a separate entry for every device on Earth. IP addresses are grouped by network, so one route can cover thousands of hosts.
2. **IP addresses show location.** When your laptop moves from home to the office, it gets a new IP address that fits the new network.
3. **The layers are independent.** IP can run over Ethernet, Wi-Fi, or links with no MAC addresses at all. Ethernet can carry IPv4, IPv6 or other protocols. Each layer can change without breaking the other.
4. **Local delivery needs a local address.** On a shared LAN, a NIC must filter frames quickly in hardware. MAC addresses do this job.

**Fuller truth:** a point-to-point link, such as PPP, has only two ends, so it does not need MAC addresses. MAC addresses solve the problem of many devices on one shared network.

[ARP](05-arp.md) connects the two. It finds the MAC address that goes with an IPv4 address on the local network. IPv6 uses Neighbor Discovery for the same job.

### Randomised MAC addresses on phones

A fixed MAC address has a privacy problem. Wi-Fi devices send their MAC address over the air. Anyone nearby can record it, for example a shop or an airport. The same address seen in many places lets them track one device over time.

So phones now hide their factory address on Wi-Fi:

- **Android 10 (2019)** made a random address for each network the default.
- **iOS 14 (2020)** added "Private Wi-Fi Address", on by default, also per network.
- **Windows 10 and 11** offer a "Random hardware addresses" setting. You must turn it on yourself.

How it works:

1. The phone generates a random address for a Wi-Fi network.
2. It sets the U/L bit to 1 and the I/G bit to 0. So the second hex digit is 2, 6, A or E.
3. By default it keeps using that address for that network, so your home router still recognises the phone.
4. A different network gets a different address.

Newer versions can also change the address over time on the same network. iOS 18, for example, offers Off, Fixed and Rotating.

This can break anything that identifies a device by MAC: MAC filtering on a home router, DHCP reservations (a fixed IP address for one device), parental controls, and hotel or airport login pages. The fix is simple: turn off the private address for that one network in the Wi-Fi settings.

**Fuller truth:** randomisation makes tracking harder, not impossible. Devices can still be recognised by other details, such as the way they scan for networks. The factory address still exists in the phone.

### How to find your MAC address

The sample addresses below come from `00:00:5E:00:53:00` to `00:00:5E:00:53:FF`, a block set aside for documentation. Your output will show real vendor addresses.

**Windows (PowerShell or Command Prompt):**

```powershell
ipconfig /all
```

```text
Wireless LAN adapter Wi-Fi:

   Description . . . . . . . . . . . : Intel(R) Wi-Fi 6 AX201 160MHz
   Physical Address. . . . . . . . . : 00-00-5E-00-53-10
   DHCP Enabled. . . . . . . . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.1.10(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

Windows calls it the "Physical Address". In PowerShell, `Get-NetAdapter` gives a shorter list with a MacAddress column.

**Linux:**

```bash
ip -brief link
```

```text
lo        UNKNOWN   00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp3s0    UP        00:00:5e:00:53:20 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

The loopback interface `lo` is software only, so its MAC address is all zeros. If the address was changed, `ethtool -P enp3s0` shows the factory one.

**macOS:**

```bash
ifconfig en0 | grep ether
networksetup -listallhardwareports
```

```text
	ether 00:00:5e:00:53:30

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: 00:00:5e:00:53:30
```

`ifconfig` shows the address in use now. `networksetup` normally shows the hardware address. On Wi-Fi with a private address turned on, the two can differ.

**Cisco IOS:**

```text
Router# show interfaces GigabitEthernet0/0 | include address
  Hardware is iGbE, address is 0000.5e00.5340 (bia 0000.5e00.5340)
```

`address` is the one in use. `bia` is the burned-in address.

**iPhone and Android:** open the details of the Wi-Fi network. The address shown there is the one used on that network. Menu names vary by version and phone maker.

### Changing a MAC address

Changing a MAC address in software is called **MAC spoofing** when it is done to pretend to be another device. It is easy. This example is for Linux, and the change is lost after a reboot:

```bash
sudo ip link set dev enp3s0 down
sudo ip link set dev enp3s0 address 02:00:5e:00:53:20
sudo ip link set dev enp3s0 up
```

The new address starts with `02`, so it is correctly marked as locally administered.

## Example

### Example 1: decode four addresses

| Address | First byte in binary | Bit 0 (I/G) | Bit 1 (U/L) | Result |
|---|---|---|---|---|
| `00:00:0C:12:34:56` | `0000 0000` | 0 | 0 | Global unicast, Cisco OUI, device part `12:34:56` |
| `5A:1C:7E:93:D4:02` | `0101 1010` | 0 | 1 | Local unicast, probably a randomised phone address |
| `01:00:5E:00:00:FB` | `0000 0001` | 1 | 0 | Multicast, the IPv4 group 224.0.0.251 (mDNS) |
| `FF:FF:FF:FF:FF:FF` | `1111 1111` | 1 | 1 | Broadcast |

### Example 2: one packet crossing two routers

PC-A sends data to a server on another network in the same company. No NAT is involved.

```text
  PC-A            R1                R2              Server
10.1.1.10     .1      .1        .2      .1        10.1.3.80
    |          |      |          |      |            |
    +----------+      +----------+      +------------+
      Link 1            Link 2             Link 3
    10.1.1.0/24       10.1.2.0/30        10.1.3.0/24
```

| Device and interface | IP address | MAC address |
|---|---|---|
| PC-A | 10.1.1.10 | `00:00:5E:00:53:0A` |
| R1 LAN side | 10.1.1.1 | `00:00:5E:00:53:11` |
| R1 link to R2 | 10.1.2.1 | `00:00:5E:00:53:12` |
| R2 link to R1 | 10.1.2.2 | `00:00:5E:00:53:21` |
| R2 server side | 10.1.3.1 | `00:00:5E:00:53:22` |
| Server | 10.1.3.80 | `00:00:5E:00:53:80` |

Step by step:

1. PC-A compares 10.1.3.80 with its own network, 10.1.1.0/24. The server is on a different network.
2. So PC-A sends the frame to its [default gateway](../04-ip-addressing/08-default-gateway.md), 10.1.1.1. It does not look for the server's MAC address.
3. PC-A uses ARP to find the gateway's MAC address.
4. R1 receives the frame, removes the old Layer 2 header, and looks up 10.1.3.80 in its routing table.
5. R1 builds a new frame for link 2. R2 does the same for link 3.

Here is what the packet looks like on each link:

| Link | Source MAC | Destination MAC | Source IP | Destination IP |
|---|---|---|---|---|
| 1 | `..:53:0A` (PC-A) | `..:53:11` (R1) | 10.1.1.10 | 10.1.3.80 |
| 2 | `..:53:12` (R1) | `..:53:21` (R2) | 10.1.1.10 | 10.1.3.80 |
| 3 | `..:53:22` (R2) | `..:53:80` (Server) | 10.1.1.10 | 10.1.3.80 |

The IP addresses never change. The MAC addresses change on every link. The server never learns PC-A's MAC address.

You can see this on PC-A. Its neighbour table holds the gateway, not the server:

```powershell
# Windows
arp -a
```

```text
Interface: 10.1.1.10 --- 0x5
  Internet Address      Physical Address      Type
  10.1.1.1              00-00-5e-00-53-11     dynamic
```

```bash
# Linux
ip neigh show
```

```text
10.1.1.1 dev eth0 lladdr 00:00:5e:00:53:11 REACHABLE
```

## Going Deeper

### EUI-48, MAC-48 and block sizes

The IEEE's current name for this 48-bit format is **EUI-48** (Extended Unique Identifier). "MAC-48" is an older term. A 64-bit version, **EUI-64**, also exists. Ethernet, Wi-Fi and Bluetooth all use 48-bit addresses of this shape.

The IEEE sells three block sizes: MA-L (24-bit prefix, the classic OUI, 16,777,216 addresses), MA-M (28-bit prefix, 1,048,576 addresses) and MA-S (36-bit prefix, 4,096 addresses).

### Bit order on the wire

Ethernet sends each byte with the lowest bit first. So the I/G bit is the very first address bit on the wire. This is why many books say "the first bit" even though it is the last bit when you write the byte in binary.

### Multicast mapping

An IPv4 multicast group maps to a MAC address by copying its low 23 bits into `01:00:5E:00:00:00`. For example, 224.0.0.5 (OSPF) becomes `01:00:5E:00:00:05`. An IPv6 group copies its low 32 bits after `33:33`, so ff02::1 becomes `33:33:00:00:00:01`.

The IPv4 mapping loses 5 bits, so 32 different groups share each MAC address. More in [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

### Modified EUI-64 in IPv6

IPv6 can build a 64-bit host part from a MAC address: split the MAC in the middle, insert `FF:FE`, and flip the U/L bit. `00:00:0C:12:34:56` becomes `0200:0CFF:FE12:3456`, so the link-local address is `fe80::200:cff:fe12:3456`.

This puts the MAC address inside an IP address, which harms privacy. Modern operating systems usually generate random host parts instead (RFC 7217, RFC 8981). Cisco IOS still uses EUI-64 for link-local addresses by default. See [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md).

### Virtual and shared MAC addresses

One NIC can use several MAC addresses, and several devices can share one.

| Prefix | Used by |
|---|---|
| `00:50:56`, `00:0C:29` | VMware virtual machines |
| `52:54:00` | QEMU/KVM virtual machines (by convention) |
| `00:15:5D` | Hyper-V virtual machines |
| `02:42` | Docker containers (traditional default) |
| `00:00:5E:00:01:xx` | VRRP virtual router |
| `0000.0C07.ACxx` | HSRP version 1 virtual router |

With [VRRP and HSRP](../05-routing/11-first-hop-redundancy.md), two routers share one virtual gateway MAC. Hosts keep the same gateway when one router fails.

### Uniqueness is only needed locally

A MAC address must be unique within its Layer 2 network. Two devices on opposite sides of the world can share an address, because their frames never meet. If two devices on one LAN share an address, the [switch](03-how-switches-work.md) sees it move between ports, and traffic reaches the wrong device or gets lost.

## Common Mistakes

- **"A MAC address cannot be changed."** The burned-in address is fixed, but the address in use can be changed in software.
- **"Websites can see my MAC address."** They cannot. Your MAC address stops at the first router. The old EUI-64 form of IPv6 was one exception, and modern systems avoid it.
- **"The MAC address stays the same across the internet."** It changes at every router. The IP address is the one that stays the same.
- **"A switch rewrites MAC addresses."** A Layer 2 switch forwards frames unchanged. A router builds a new Layer 2 header.
- **"To reach a remote server, a PC needs the server's MAC address."** It needs the MAC address of its default gateway.
- **"Routers forward broadcasts."** They do not. A broadcast stays in its broadcast domain.
- **"MAC filtering is real security."** Wi-Fi addresses can be seen by anyone nearby and copied easily. It stops casual users only.
- **"Multicast and broadcast are the same."** Broadcast targets everyone on the LAN. Multicast targets only a group, although a switch may flood it to all ports if it does not track group members.
- **"Each device has one MAC address."** Each interface has its own. A laptop with Wi-Fi and Ethernet has at least two, and a router has one per interface or more.
- **"Check the first hex digit for the special bits."** Check the second hex digit. `02` is local, while `20` is not.
- **"A random MAC changes on every connection."** By default, phones keep one random address per network.

## Interview Questions

**Q: What is a MAC address?**

A: It is a 48-bit Layer 2 address for a network interface, written as 12 hex digits. Devices use it to deliver frames on the local network. The first 24 bits are the OUI, which identifies the vendor, and the vendor chooses the last 24 bits.

**Q: How can you tell if a MAC address is unicast or multicast?**

A: Look at the lowest bit of the first byte, the I/G bit. If it is 0, the address is unicast. If it is 1, it is a group address. In practice, an odd second hex digit means multicast.

**Q: What is a locally administered MAC address?**

A: It is an address set by software or an admin, not assigned from a vendor block. The U/L bit is 1, so for unicast the second hex digit is 2, 6, A or E. Virtual machines and randomised phone addresses use this kind.

**Q: What is the broadcast MAC address, and who receives it?**

A: It is FF:FF:FF:FF:FF:FF, all 48 bits set to 1. Every device in the same broadcast domain receives it. A switch floods it within the VLAN, and a router does not forward it. ARP requests are a common example.

**Q: Why do we need both MAC and IP addresses?**

A: They solve different problems. A MAC address delivers a frame on one link. An IP address gets a packet across many networks. MAC addresses are flat, so routers could not summarise them. IP addresses are hierarchical, so one route can cover a whole network. ARP links the two.

**Q: What changes when a packet crosses a router?**

A: The router removes the old frame and builds a new one, so the source and destination MAC addresses change. The source and destination IP addresses stay the same, unless NAT is used. The router also lowers the TTL.

**Q: How does a host send data to a device on another network?**

A: It sees that the destination is outside its subnet. It uses ARP to get the MAC address of its default gateway. It then sends a frame to the gateway's MAC address, with the IP packet still addressed to the final destination.

**Q: Why do phones use randomised MAC addresses, and what problems can this cause?**

A: They do it to stop tracking, because Wi-Fi addresses can be seen by anyone nearby. Each network gets its own random address. This can break MAC filtering, DHCP reservations and some captive portals. The fix is to turn off the private address for that network.

**Q: Is MAC filtering a good security control?**

A: No. An attacker can capture an allowed address and copy it. It can be a small extra barrier, but real access control needs something like 802.1X or WPA3.

## Quick Recap

- A MAC address is 48 bits, or 6 bytes, written as 12 hex digits. The first 3 bytes are the OUI.
- Bit 0 of the first byte is the I/G bit: 0 is unicast, 1 is group.
- Bit 1 of the first byte is the U/L bit: 0 is global, 1 is local.
- `FF:FF:FF:FF:FF:FF` is broadcast. It reaches the whole broadcast domain and stops at routers.
- MAC addresses work link by link and change at each router. IP addresses work end to end.
- Phones use a random, locally administered address per Wi-Fi network for privacy.
- Find yours with `ipconfig /all` (Windows), `ip link` (Linux) or `ifconfig en0` (macOS).
