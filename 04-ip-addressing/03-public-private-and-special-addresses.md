# Public, Private and Special IP Addresses

**Level:** Beginner | **Prerequisites:** [IPv4 Addressing Basics](01-ipv4-basics.md), [IPv4 Classes (Classful Addressing)](02-ipv4-classes.md)

## In One Line

Not every IPv4 address is the same: some work across the whole internet (public), some work only inside your own network (private), and some blocks have a special job, such as talking to yourself (loopback) or reaching everyone on the local network (broadcast).

## Why It Matters

IPv4 has only about 4.3 billion addresses (exactly 4,294,967,296). That is far too few for every phone, laptop, server and sensor on Earth. IANA (Internet Assigned Numbers Authority, the group that manages global number resources) gave out its last free blocks in 2011.

Private addresses solve this problem. Millions of homes and companies reuse the same private addresses, and only a few devices need a scarce public address.

Special addresses solve other problems. A machine needs a way to test itself, ask for an address before it has one, and reach a group of devices at once.

Interviewers ask about this topic because it reveals real understanding, and because these addresses appear in daily troubleshooting. When a laptop shows `169.254.37.112`, a good engineer knows at once that DHCP failed. When a server listens on `127.0.0.1`, a good engineer knows why nobody else can reach it.

## Simple Explanation

Think of IP addresses like addresses in a city with many hotels.

- A **public address** is the full street address of a hotel. It is unique in the world, and any post office can deliver to it.
- A **private address** is a room number, such as Room 101. Every hotel has a Room 101. The number means something only inside that hotel.
- The **reception desk** connects the two worlds. Guests send letters through reception, and reception writes the hotel's street address on them. In networking, this job is called NAT (Network Address Translation, covered in [NAT and PAT](09-nat-and-pat.md)).

Other special addresses fit the same picture:

- **Loopback** is a note you write to yourself. It never leaves your room.
- **Link-local (APIPA)** is a temporary name tag. Reception is closed, so guests on one floor make up names and talk only to each other.
- **Broadcast** is an announcement over the floor's loudspeaker. Everyone on that floor hears it.
- **Multicast** is a group chat. Only people who joined the group get the message.
- **Documentation addresses** are like the fake phone numbers used in movies. They exist so that examples never reach a real person.
- **Reserved addresses** are empty land the city has not opened for building.

**Where the analogy stops:** a real hotel has one reception desk. A packet can pass through several NAT devices. Also, a private address is not secret or protected by itself. It is simply not routed on the public internet.

## How It Works

### A quick note on the /number

You will see blocks written like `10.0.0.0/8`. The number after the slash says how many bits at the start of the address are fixed.

An IPv4 address has 32 bits. So `/8` fixes 8 bits and leaves 24 bits free, which gives 2^24 = 16,777,216 addresses. A bigger slash number means a smaller block. The full story is in [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md).

### The map of the IPv4 space

```text
First octet   What lives there
-----------   ------------------------------------------------
0             "This network" (0.0.0.0/8)
1-126         Mostly public. Contains 10.0.0.0/8 (private)
              and 100.64.0.0/10 (CGNAT shared space)
127           Loopback
128-191       Mostly public. Contains 169.254.0.0/16 (link-local)
              and 172.16.0.0/12 (private)
192-223       Mostly public. Contains 192.168.0.0/16 (private)
              and the documentation and test blocks
224-239       Multicast
240-255       Reserved, including 255.255.255.255 (broadcast)
```

An **octet** is one of the four numbers in an address. It holds 8 bits, so its value is 0-255.

### Public addresses

A **public address** is unique on the whole internet. Routers on the internet know how to reach it. We say it is **routable**, which means routers will carry traffic to it.

Public addresses follow a chain of ownership:

1. IANA hands large blocks to five RIRs (Regional Internet Registries): ARIN, RIPE NCC, APNIC, LACNIC and AFRINIC.
2. Each RIR gives smaller blocks to ISPs (Internet Service Providers, the companies that sell internet access) and to large organisations.
3. The ISP gives an address to your home router or server.

You cannot simply choose a public address. It must be assigned to you.

### Private addresses (RFC 1918)

An **RFC** (Request for Comments) is a numbered document that describes an internet standard or practice. RFC 1918 sets aside three blocks for private use. Anyone may use them without asking permission.

| Block | Range | Number of addresses | Typical use |
|---|---|---|---|
| 10.0.0.0/8 | 10.0.0.0-10.255.255.255 | 16,777,216 | Large companies, cloud networks |
| 172.16.0.0/12 | 172.16.0.0-172.31.255.255 | 1,048,576 | Medium networks, Docker default bridge (172.17.0.0/16) |
| 192.168.0.0/16 | 192.168.0.0-192.168.255.255 | 65,536 | Homes and small offices |

The middle block confuses many people. It ends at 172.31, not 172.255. Here is why:

```text
/12 fixes the first 12 bits: all 8 of the first octet + 4 of the second

172.16  =  10101100.0001 0000     second octet 16 = 0001 0000
172.31  =  10101100.0001 1111     second octet 31 = 0001 1111
                    ----
                    these 4 bits stay the same

172.15  =  10101100.0000 1111     outside the block: public
172.32  =  10101100.0010 0000     outside the block: public
```

ISPs drop private addresses at the internet edge, so these addresses never travel across the public internet. To reach the internet, a private host needs NAT:

```text
   Private side                         Public side
   (your home)                          (the internet)

   Laptop 192.168.1.50 ---+
                          +--- Router ------ Internet ------ Server
   Phone  192.168.1.51 ---+    inside:  192.168.1.1   198.51.100.10
                               outside: 203.0.113.25

   The server sees both devices as 203.0.113.25
```

### Loopback (127.0.0.0/8)

The **loopback** address lets a machine talk to itself. Traffic sent to it turns around inside the operating system. It never reaches the network card or the cable.

- The whole block 127.0.0.0-127.255.255.255 is reserved for this, more than 16 million addresses.
- `127.0.0.1` is the one everyone uses.
- The name `localhost` normally points to `127.0.0.1` and to the IPv6 loopback `::1`.
- A packet with a 127.x.x.x address must never appear on a real network. Linux calls such a packet a **martian** and drops it.

`ping 127.0.0.1` tests the IP software on your machine. It does not prove that your cable, network card or network work.

### APIPA and link-local (169.254.0.0/16)

**Link-local** means valid only on the local link, the network segment a device sits on directly. Routers do not forward these addresses.

APIPA (Automatic Private IP Addressing) is Microsoft's name for this feature. The standard is RFC 3927, which calls it IPv4 link-local addressing. It works like this:

1. A host asks for an address using [DHCP](../07-application-protocols/04-dhcp.md), the protocol that hands out addresses automatically.
2. No DHCP server answers.
3. The host picks a random address from 169.254.1.0-169.254.254.255. The first 256 and last 256 addresses of the block are reserved and not picked.
4. The host uses [ARP](../03-ethernet-and-switching/05-arp.md) to check that no neighbour already has this address.
5. It uses the address with mask 255.255.0.0 and **no default gateway**.
6. It keeps trying DHCP in the background. Windows retries about every 5 minutes by default.

The result: the host can talk to neighbours that also have a 169.254 address, but it cannot reach the internet. Windows and macOS use APIPA by default. Most Linux systems do not unless configured to do so.

### The many meanings of 0.0.0.0

`0.0.0.0` has no single meaning. It depends on where you see it.

| Where you see it | What it means |
|---|---|
| Source address of a packet | "I have no address yet." A host sends a DHCP Discover from 0.0.0.0 |
| Routing table, as 0.0.0.0/0 | The default route: "any destination not matched by a better route" |
| A server listening on it | "Accept connections on all IPv4 addresses of this machine" |
| Shown on an interface | The interface has no address yet |
| DNS blocklist or hosts file | A blocked name, so the traffic goes nowhere |
| As a subnet mask, 0.0.0.0 | Zero bits must match, so every address matches |
| As a Cisco ACL wildcard, 0.0.0.0 | Every bit must match, so it means one exact address |

The formal block is 0.0.0.0/8, called "this network". The single address 0.0.0.0/32 means "this host on this network". `0.0.0.0` is not a valid destination on a network.

Compare these two listen addresses:

```text
Listening on 127.0.0.1:5432   Only this machine can connect
Listening on 0.0.0.0:80       Other machines can connect too
```

### Limited broadcast (255.255.255.255)

`255.255.255.255` means "everyone on this local network". It is called the **limited broadcast** because routers do not forward it. It stays inside the broadcast domain, the set of devices that hear each other's broadcasts (see [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md)).

Its most famous use is DHCP, because a new host does not yet know its subnet or the DHCP server's address:

```text
New host                                    DHCP server
    |                                            |
    |  DHCP Discover                             |
    |  Source IP:      0.0.0.0                   |
    |  Destination IP: 255.255.255.255           |
    |  UDP port 68 to port 67                    |
    |------------------------------------------->|
```

To cross a router, a DHCP relay must carry the request. Plain broadcast cannot do it.

### CGNAT shared space (100.64.0.0/10)

An ISP may not have enough public addresses for every customer. So it adds a second layer of NAT inside its own network. This is **CGNAT** (Carrier-Grade NAT).

The customer's home router needs an address on the ISP side. RFC 1918 addresses are a poor choice here, because the customer may already use the same range at home, and the two would clash. So RFC 6598 created a separate block for this job:

- Block: 100.64.0.0/10
- Range: 100.64.0.0-100.127.255.255
- Size: 4,194,304 addresses

```text
Home LAN          ISP network              Internet
192.168.1.0/24    100.64.0.0/10            public addresses

Laptop ---- Home router ---- CGNAT device ---- Server
192.168.1.50  in:  192.168.1.1  in:  100.64.0.1  198.51.100.10
              out: 100.72.14.9  out: 203.0.113.25
```

Your traffic is translated twice. Many customers share one public address. This creates a real cost: you cannot run a home server or use port forwarding, because you do not control the public address.

Shared space is not RFC 1918 space, and you should not use it for an ordinary LAN. Some products outside ISPs do use it on purpose. For example, the VPN product Tailscale gives its devices addresses from this block, because they rarely clash with home and office LANs.

### Multicast (224.0.0.0/4)

A **unicast** packet goes to one receiver. A **broadcast** goes to everyone on the network. A **multicast** packet goes to a group, and only members of that group process it.

- Block: 224.0.0.0/4 (the old Class D)
- Range: 224.0.0.0-239.255.255.255
- The first four bits are always `1110`.
- A multicast address is used only as a destination, never as a source.

| Range or address | Use |
|---|---|
| 224.0.0.0/24 | Local network control. Routers do not forward these |
| 224.0.0.1 | All hosts on this network |
| 224.0.0.2 | All routers on this network |
| 224.0.0.5 and 224.0.0.6 | OSPF routers and OSPF designated routers |
| 224.0.0.9 | RIPv2 routers |
| 224.0.0.10 | EIGRP routers |
| 224.0.0.18 | VRRP |
| 224.0.0.251 | Multicast DNS (mDNS) |
| 232.0.0.0/8 | Source-specific multicast |
| 239.0.0.0/8 | Administratively scoped: the "private" multicast range for use inside one organisation |

The full story is in [Unicast, Broadcast, Multicast and Anycast](13-unicast-broadcast-multicast-anycast.md) and [Multicast Routing](../05-routing/14-multicast-routing.md).

### Documentation ranges

RFC 5737 sets aside three blocks for books, manuals and examples. They should never appear on a real network.

| Block | Name |
|---|---|
| 192.0.2.0/24 | TEST-NET-1 |
| 198.51.100.0/24 | TEST-NET-2 |
| 203.0.113.0/24 | TEST-NET-3 |

IPv6 has `2001:db8::/32` for the same purpose. This course uses these blocks for all made-up public addresses, so no real company is affected if someone copies an example.

### Reserved (240.0.0.0/4)

The block 240.0.0.0-255.255.255.255 is the old Class E. It was set aside for future use and never opened for normal use. The block holds over 268 million addresses.

People often ask, "Why not release it to solve the shortage?" The problem is software. Many operating systems and routers treat these addresses as invalid and drop them. Linux accepts them, but Windows and much network equipment do not. Fixing all that would take many years, and the block would last only a short time. IPv6 is the real answer.

### Network and broadcast addresses

Every normal subnet gives up two addresses for special use. A **subnet** is one block of addresses that forms a single network.

- **Network address:** the host bits are all 0. It names the subnet itself. You use it in routing tables, but you do not assign it to a device.
- **Broadcast address:** the host bits are all 1. A packet sent there reaches all hosts in that subnet. This is the **directed broadcast**.

```text
192.168.1.10/24

               |<----- network part ----->|<- host ->|
Address:        11000000.10101000.00000001.00001010    192.168.1.10
Host bits 0:    11000000.10101000.00000001.00000000    192.168.1.0
Host bits 1:    11000000.10101000.00000001.11111111    192.168.1.255

Network:    192.168.1.0
Usable:     192.168.1.1 - 192.168.1.254  (254 hosts)
Broadcast:  192.168.1.255
```

**Analogy:** the network address is the name of a building. It labels the whole building, not one room. The broadcast address is the building's loudspeaker. **Where this stops:** a loudspeaker has one fixed location, but the broadcast address changes with the subnet size.

| | Limited broadcast | Directed broadcast |
|---|---|---|
| Example | 255.255.255.255 | 192.168.1.255 |
| Target | The local network | All hosts in a named subnet |
| Sender must know the subnet? | No | Yes |
| Crosses routers? | Never | Possible in theory, blocked by default on modern routers |

Exceptions exist:

- A **/31** has 2 addresses and no network or broadcast address. Both are usable on a point-to-point link (RFC 3021).
- A **/32** is a single address. It is often used for a router's loopback interface.

### All special ranges in one table

"On the internet?" means whether the block is routed on the public internet as normal unicast.

| Block | Range | Addresses | Purpose | On the internet? | Standard |
|---|---|---|---|---|---|
| 0.0.0.0/8 | 0.0.0.0-0.255.255.255 | 16,777,216 | "This network" | No | RFC 791, RFC 1122 |
| 10.0.0.0/8 | 10.0.0.0-10.255.255.255 | 16,777,216 | Private | No | RFC 1918 |
| 100.64.0.0/10 | 100.64.0.0-100.127.255.255 | 4,194,304 | Shared space for CGNAT | No | RFC 6598 |
| 127.0.0.0/8 | 127.0.0.0-127.255.255.255 | 16,777,216 | Loopback | No | RFC 1122 |
| 169.254.0.0/16 | 169.254.0.0-169.254.255.255 | 65,536 | Link-local (APIPA) | No | RFC 3927 |
| 172.16.0.0/12 | 172.16.0.0-172.31.255.255 | 1,048,576 | Private | No | RFC 1918 |
| 192.0.0.0/24 | 192.0.0.0-192.0.0.255 | 256 | IETF protocol assignments | Mostly no | RFC 6890 |
| 192.0.2.0/24 | 192.0.2.0-192.0.2.255 | 256 | Documentation (TEST-NET-1) | No | RFC 5737 |
| 192.88.99.0/24 | 192.88.99.0-192.88.99.255 | 256 | Old 6to4 relay, deprecated | No longer in use | RFC 7526 |
| 192.168.0.0/16 | 192.168.0.0-192.168.255.255 | 65,536 | Private | No | RFC 1918 |
| 198.18.0.0/15 | 198.18.0.0-198.19.255.255 | 131,072 | Network device benchmarking | No | RFC 2544 |
| 198.51.100.0/24 | 198.51.100.0-198.51.100.255 | 256 | Documentation (TEST-NET-2) | No | RFC 5737 |
| 203.0.113.0/24 | 203.0.113.0-203.0.113.255 | 256 | Documentation (TEST-NET-3) | No | RFC 5737 |
| 224.0.0.0/4 | 224.0.0.0-239.255.255.255 | 268,435,456 | Multicast | Not as unicast | RFC 5771 |
| 240.0.0.0/4 | 240.0.0.0-255.255.255.255 | 268,435,456 | Reserved, last address is the limited broadcast | No | RFC 1112 |
| 255.255.255.255/32 | 255.255.255.255 | 1 | Limited broadcast | No | RFC 919 |

IANA keeps the official list, the IPv4 Special-Purpose Address Registry. It also lists a few small service blocks that are rarely asked about, such as those for AS112 DNS and AMT (a multicast tunnelling method).

## Example

### Example 1: classify any address

Check in this order: first the special blocks, then the private and shared ranges. If nothing matches, the address is public.

| Address | How to decide | Answer |
|---|---|---|
| 10.20.30.40 | First octet is 10 | Private |
| 172.31.255.1 | Second octet 31 is within 16-31 | Private |
| 172.32.0.1 | Second octet 32 is outside 16-31 | Public |
| 192.168.50.7 | Starts with 192.168 | Private |
| 192.169.1.1 | Starts with 192.169, not 192.168 | Public |
| 169.254.10.5 | Starts with 169.254 | Link-local, DHCP probably failed |
| 100.72.14.9 | Second octet 72 is within 64-127 | Shared space, CGNAT |
| 100.128.0.1 | Second octet 128 is outside 64-127 | Public |
| 127.0.0.53 | First octet is 127 | Loopback |
| 224.0.0.5 | First octet is within 224-239 | Multicast, OSPF |
| 198.18.0.20 | Within 198.18.0.0-198.19.255.255 | Benchmarking |
| 203.0.113.7 | Within 203.0.113.0/24 | Documentation |
| 250.1.2.3 | First octet is within 240-255 | Reserved |

### Example 2: a laptop with no internet

A Windows user says, "I am connected, but nothing works."

```powershell
# Windows
ipconfig
```

```text
Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::a1b2:c3d4:e5f6:1234%12
   Autoconfiguration IPv4 Address. . : 169.254.37.112
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :
```

Reading the output:

1. "Autoconfiguration" and 169.254 mean no DHCP server answered.
2. The Default Gateway line is empty, so the laptop has no way off the local network.
3. The physical link is probably up, since the laptop tried DHCP on it.
4. Next, check the DHCP server, the VLAN of the switch port, and the DHCP relay. Then run:

```powershell
# Windows: ask for a new address
ipconfig /release
ipconfig /renew
```

### Example 3: who can reach this service?

A developer says, "My database works on the server, but my app on another machine cannot connect."

```bash
# Linux
ss -tln
```

```text
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN  0       511           0.0.0.0:80          0.0.0.0:*
LISTEN  0       244         127.0.0.1:5432        0.0.0.0:*
```

```powershell
# Windows
netstat -an -p tcp
```

```text
  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING
  TCP    127.0.0.1:5432         0.0.0.0:0              LISTENING
```

Port 80 listens on all addresses, so remote machines can connect. Port 5432 (PostgreSQL) listens only on loopback, so only local programs can connect. The fix is to change the database's listen address, and to restrict it with a firewall.

### Example 4: are you behind CGNAT?

Compare two values:

1. The WAN address shown on your home router's status page. Say it shows `100.72.14.9`.
2. The address the internet sees for you:

```bash
# Linux or macOS
curl -s https://ifconfig.me
```

```powershell
# Windows (curl.exe avoids the PowerShell curl alias)
curl.exe -s https://ifconfig.me
```

```text
203.0.113.25
```

The router's WAN address is in 100.64.0.0/10, and it differs from the address the internet sees. So you are behind CGNAT. If the two values matched, your router would hold the public address directly.

### Example 5: find the network and broadcast addresses

| Address | Network | First host | Last host | Broadcast | Usable hosts |
|---|---|---|---|---|---|
| 192.168.1.10/24 | 192.168.1.0 | 192.168.1.1 | 192.168.1.254 | 192.168.1.255 | 254 |
| 192.168.10.70/26 | 192.168.10.64 | 192.168.10.65 | 192.168.10.126 | 192.168.10.127 | 62 |
| 10.0.0.5/30 | 10.0.0.4 | 10.0.0.5 | 10.0.0.6 | 10.0.0.7 | 2 |
| 172.16.0.255/23 | 172.16.0.0 | 172.16.0.1 | 172.16.1.254 | 172.16.1.255 | 510 |

Look at the last row. `172.16.0.255` ends in 255, yet it is a normal host address. The mask decides, not the last number. You will learn this method in [Subnetting Step by Step](05-subnetting-step-by-step.md).

## Going Deeper

### Bogon filtering

A **bogon** is an address that should never appear on the public internet. Good network operators drop such traffic at the edge. Attackers often fake source addresses, a trick called **spoofing**, and this filter removes the lazy cases.

```text
! Cisco IOS: inbound filter on an internet-facing interface
ip access-list extended FROM-INTERNET
 deny ip 0.0.0.0 0.255.255.255 any
 deny ip 10.0.0.0 0.255.255.255 any
 deny ip 100.64.0.0 0.63.255.255 any
 deny ip 127.0.0.0 0.255.255.255 any
 deny ip 169.254.0.0 0.0.255.255 any
 deny ip 172.16.0.0 0.15.255.255 any
 deny ip 192.168.0.0 0.0.255.255 any
 deny ip 224.0.0.0 15.255.255.255 any
 deny ip 240.0.0.0 15.255.255.255 any
 permit ip any any
```

These lines filter by source address. The final `permit` is here only to keep the example short. A real edge policy is tighter. Also check your own design first: if your ISP uses 100.64.0.0/10 on the link to you, do not block it blindly. See [Access Control Lists](../09-security/05-access-control-lists.md).

### Choosing a private range

The choice looks free, but it affects you later:

- Companies usually pick 10.0.0.0/8 because it has room for growth.
- Avoid 192.168.0.0/24 and 192.168.1.0/24 for a company network. They are the common defaults on home routers. A remote worker at home on 192.168.1.0/24 cannot reach an office subnet with the same numbers over a VPN, because the two ranges overlap.
- Plan cloud networks with the same care. Overlapping ranges block [peering and hybrid connections](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).

### The cloud metadata address

Major cloud platforms, including AWS, Azure and Google Cloud, serve instance metadata at `169.254.169.254`. It is a link-local address, so a virtual machine can reach it without any route. Attackers who trick a server into fetching a URL on their behalf (an attack called SSRF, Server-Side Request Forgery) often aim at this address, since metadata may include credentials.

### Directed broadcast and security

Forwarding directed broadcasts allowed the old **Smurf attack**: one ping to a subnet's broadcast address made every host reply to a faked victim. RFC 2644 changed the router default to not forwarding them, and Cisco IOS has defaulted to `no ip directed-broadcast` since release 12.0. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

### Router loopback interfaces are different

A router's `Loopback0` is a virtual interface that stays up as long as the router runs. It usually has a normal routable or private address, not 127.x.x.x. Other devices can reach it. It is often used as a stable identity for the router.

```text
! Cisco IOS
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
```

### IPv6 equivalents

| IPv4 idea | IPv6 version |
|---|---|
| Private (RFC 1918) | Unique Local Addresses, fc00::/7, in practice fd00::/8 |
| Loopback 127.0.0.1 | ::1 |
| Link-local 169.254.0.0/16 | fe80::/10, on every IPv6 interface, a normal part of the design |
| Unspecified 0.0.0.0 | :: |
| Default route 0.0.0.0/0 | ::/0 |
| Multicast 224.0.0.0/4 | ff00::/8 |
| Broadcast | Does not exist, multicast does the job |
| Documentation | 2001:db8::/32 and 3fff::/20 |

See [IPv6 Basics](11-ipv6-basics.md).

## Common Mistakes

- **"172.16.0.0 to 172.255.255.255 is private."** Only 172.16-172.31 is private. An address such as 172.32.0.1 is public.
- **"Private addresses cannot be routed."** They route normally inside a company. They are only filtered on the public internet. "Private" is a rule of use, not a feature of routers.
- **"A private address means I am secure."** It reduces direct exposure, but it is not a firewall. Security needs real filtering.
- **"Only 127.0.0.1 is loopback."** The whole of 127.0.0.0/8 is loopback.
- **"Ping to 127.0.0.1 works, so my network is fine."** That test never leaves the machine.
- **"169.254 is a private address like 192.168."** It is link-local, not RFC 1918. Routers do not forward it, and seeing it usually signals a DHCP problem. It is not a routable design choice.
- **"0.0.0.0 is an address I can ping."** It is a placeholder with different meanings by context. Some systems treat a connection to it as local, but that is a quirk, not a rule.
- **"An address ending in .0 or .255 is always invalid."** It depends on the mask. In 172.16.0.0/23, the address 172.16.0.255 is a normal host.
- **"100.64.0.0/10 is good for my office LAN."** It is meant for carrier NAT and similar shared use. Use RFC 1918 space for normal networks.
- **Using real public addresses in private labs.** If a network uses someone else's public range internally, it can no longer reach the real owner of that range.

## Interview Questions

**Q: What are the three private IPv4 ranges?**

A: They come from RFC 1918: 10.0.0.0/8, 172.16.0.0/12, which runs to 172.31.255.255, and 192.168.0.0/16. Anyone can use them inside a network, but they are not routed on the public internet.

**Q: What is the difference between a public and a private IP address?**

A: A public address is unique worldwide and routed on the internet. A private address is unique only inside one network, and many networks reuse it. A private host reaches the internet through NAT, which swaps its address for a public one.

**Q: What does an address starting with 169.254 tell you?**

A: The host used APIPA, or IPv4 link-local addressing. It asked for an address by DHCP, got no answer, and chose one itself. It can talk only to neighbours on the same link and has no gateway. So I would check the DHCP server, the relay and the VLAN.

**Q: What is 127.0.0.1, and what does pinging it prove?**

A: It is the loopback address. Traffic to it stays inside the machine. A successful ping proves the local TCP/IP software works. It proves nothing about the network card, the cable or the wider network.

**Q: What does 0.0.0.0 mean?**

A: It depends on context. As a source address, it means the host has no address yet, as in a DHCP Discover. In a routing table, 0.0.0.0/0 is the default route. For a server, listening on 0.0.0.0 means accepting connections on all local IPv4 addresses.

**Q: What is the difference between 255.255.255.255 and a subnet's broadcast address?**

A: 255.255.255.255 is the limited broadcast. It reaches the local network and is never routed. A subnet broadcast, such as 192.168.1.255 for a /24, is a directed broadcast. Modern routers block it by default because it was used in attacks such as Smurf.

**Q: What is 100.64.0.0/10, and why do we not just use RFC 1918 for it?**

A: It is shared address space for Carrier-Grade NAT, defined in RFC 6598. ISPs use it between the customer's router and the CGNAT device. RFC 1918 could clash with the customer's home LAN, so a separate range avoids that overlap.

**Q: Why can a network not assign its network and broadcast addresses to hosts?**

A: The network address, with all host bits 0, names the subnet. The broadcast address, with all host bits 1, targets every host in it. So a /24 has 256 addresses but 254 usable hosts. The exceptions are /31 point-to-point links and /32 single addresses.

**Q: What is the multicast range, and can an address in it be a source?**

A: It is 224.0.0.0/4, which runs from 224.0.0.0 to 239.255.255.255. A multicast address is a destination only, and the source is the sender's normal unicast address. For example, OSPF sends hellos to 224.0.0.5 from the router's interface address.

**Q: Why not open 240.0.0.0/4 to solve the IPv4 shortage?**

A: Many operating systems and devices treat that block as invalid and drop it, so updating everything would take years. The block would also give only a small, temporary gain. IPv6 is the long-term solution.

**Q: A web service runs, but only local users can connect. What do you check first?**

A: I would check the listen address with `ss -tln` on Linux or `netstat -an` on Windows. If it shows 127.0.0.1, it accepts only local connections, so it must bind to 0.0.0.0 or a specific interface. Then I would check the firewall.

## Quick Recap

- Private ranges are 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. They are reused everywhere and reach the internet through NAT.
- 127.0.0.0/8 is loopback, and 169.254.0.0/16 is link-local, usually a sign that DHCP failed.
- 0.0.0.0 means "no address", "all addresses" or "default route", depending on context. 255.255.255.255 is the limited broadcast and never crosses a router.
- 100.64.0.0/10 is shared space for CGNAT, and it is not RFC 1918.
- 224.0.0.0/4 is multicast, 240.0.0.0/4 is reserved, and 192.0.2.0/24, 198.51.100.0/24 and 203.0.113.0/24 are for documentation.
- In a normal subnet, all host bits 0 is the network address and all host bits 1 is the broadcast address. The mask decides, not the last number.
