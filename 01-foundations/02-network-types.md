# Types of Networks

**Level:** Beginner | **Prerequisites:** [What Is a Network?](01-what-is-a-network.md)

## In One Line

Networks are named by how far they reach (PAN, LAN, CAN, MAN, WAN), by what job they do (SAN), by how they are built (VPN), and by who is allowed in (intranet, extranet, internet).

## Why It Matters

A network between two buildings and a network between two countries have very different problems. The first can use your own cable. The second must use cables owned by someone else, it costs money every month, and it has more delay.

These names tell engineers which problem they are solving. When someone says "the WAN link is down", everyone knows the fault is probably between sites, and that a phone company may need a call.

Interviewers ask this topic early for three reasons:

- It shows whether you know the basic words.
- It leads to deeper questions, such as "why is a LAN faster than a WAN?"
- It is easy to give a shallow answer. A strong candidate says that distance is only one part of the answer. Ownership and purpose matter too.

## Simple Explanation

Think about how things move in real life.

| Network | Real-life picture |
|---|---|
| PAN | Passing a pen from one hand to the other |
| LAN | Walking between rooms in your own house |
| WLAN | Calling out to someone across the same room |
| CAN | Private roads inside a university |
| MAN | The bus network of one city |
| WAN | Airlines and ships between countries |
| SAN | A private conveyor belt between a shop and its warehouse |
| VPN | A sealed, locked box sent over public roads |
| Intranet | A staff-only area in a building |
| Extranet | The same area, with a visitor badge for a trusted supplier |
| Internet | The public street network of the whole world |

**Where the analogy stops being true:** a plane is faster than walking, but a long network link is usually slower than a short one. Your home network may run at 1 Gbps while your link to the rest of the world runs at 100 Mbps. Also, a network name is not only about distance. A SAN and a VPN have no fixed size at all.

## How It Works

### Three ways to sort networks

Every network name answers one of these questions:

1. **How far does it reach?** PAN, LAN, CAN, MAN, WAN.
2. **What job does it do?** SAN carries storage traffic. WLAN says the medium is radio.
3. **Who is allowed in, and how is it built?** Intranet, extranet, internet, VPN.

So one network can have more than one name. A company VPN can carry people into the company intranet over the internet. All three words are true at once.

Here is the size view. Each box is a bigger scale than the box inside it.

```text
+---------------------------------------------------------------+
| WAN / Internet (countries and continents)                     |
|  +---------------------------------------------------------+  |
|  | MAN (one city)                                          |  |
|  |  +---------------------------------------------------+  |  |
|  |  | CAN (one campus)                                  |  |  |
|  |  |  +---------------------------------------------+  |  |  |
|  |  |  | LAN / WLAN (one home or building)           |  |  |  |
|  |  |  |  +---------------------------------------+  |  |  |  |
|  |  |  |  | PAN (around one person)               |  |  |  |  |
|  |  |  |  +---------------------------------------+  |  |  |  |
|  |  |  +---------------------------------------------+  |  |  |
|  |  +---------------------------------------------------+  |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

This picture shows scale only. A real WAN does not have to contain a MAN. SAN and VPN are missing because size does not define them.

### PAN (Personal Area Network)

A PAN connects devices around one person.

- **Size:** about 1-10 m. Some links are much shorter. NFC (Near Field Communication, used for tap-to-pay) works at about 4 cm.
- **Technology:** Bluetooth, USB, NFC. Wireless versions are sometimes called WPAN (Wireless PAN).
- **Speed:** Bluetooth gives a few Mbps. Mbps means megabits per second, a measure of data speed. USB cables are far faster.
- **Examples:** earbuds linked to a phone, a smartwatch sending heart data to a phone, a phone sharing its connection with a laptop by USB.
- **Owner:** you.

### LAN (Local Area Network)

A LAN connects devices in one small area, such as a home, an office floor or one building.

- **Size:** a room to a building. A single copper Ethernet cable run has a 100 m limit. Ethernet is the standard wired LAN technology, defined by IEEE 802.3. See [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md).
- **Speed:** 1 Gbps to a desk is common. 10 Gbps or more is common between switches. 1 Gbps (gigabit per second) equals 1000 Mbps.
- **Devices:** a switch joins devices inside the LAN. A router joins the LAN to other networks. See [Network Devices](04-network-devices.md).
- **Example:** 30 office PCs and 2 printers on one switch, all using addresses in 192.168.1.0/24.
- **Owner:** you or your company. You buy it once and pay no monthly fee for the cables.

**Fuller truth:** engineers often give "LAN" a tighter meaning: a group of devices that can reach each other without passing through a router. That group is a broadcast domain. See [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md). In daily speech, people also say "our LAN" for a whole office that has several such groups.

### WLAN (Wireless LAN)

A WLAN is a LAN that uses radio in place of cable.

- **Size:** tens of metres from each access point. An access point is the box that links radio devices to the wired network. Walls reduce the range a lot.
- **Technology:** the IEEE 802.11 family, sold under the name Wi-Fi. Wi-Fi 6 is 802.11ax and Wi-Fi 7 is 802.11be. It uses the 2.4 GHz, 5 GHz and 6 GHz bands, and 6 GHz needs Wi-Fi 6E or newer.
- **Speed:** often a few hundred Mbps in real rooms. New equipment close to the access point can pass 1 Gbps.
- **Example:** phones and laptops on the Wi-Fi in a coffee shop.

A WLAN is a kind of LAN. In most homes, the wired and wireless devices share one address range and can talk to each other directly.

The main difference is the medium. Devices on the same radio channel share the air and mostly take turns. A switch gives each wired device its own link. That is why a wired PC is usually steadier than a laptop on busy Wi-Fi.

A WLAN also works with no internet at all. Wi-Fi joins you to a local network. The internet is a separate service that the local network may or may not have. Details are in [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md).

### CAN (Campus Area Network)

A CAN joins several LANs across a group of nearby buildings, all on land that one organisation controls.

- **Size:** about 1-5 km across.
- **Technology:** Ethernet over fibre between buildings. Fibre is a glass cable that carries light and can run much further than copper. See [Cables and Transmission Media](05-cables-and-media.md).
- **Speed:** often 10-100 Gbps between buildings.
- **Examples:** a university, a large hospital site, a factory, a military base.
- **Owner:** the organisation, including the cable between buildings.

Do not mix this up with the CAN bus (Controller Area Network) used inside cars. It has the same letters and is a different thing.

**Fuller truth:** many engineers just say "campus LAN". CAN is a textbook word more than a daily word.

### MAN (Metropolitan Area Network)

A MAN joins sites across one city or metropolitan area.

- **Size:** about 5-50 km. Some reach further.
- **Technology:** fibre rings run by a carrier, and Metro Ethernet, which is an Ethernet service sold by a telecom company across a city. A carrier is a company that sells network links.
- **Speed:** commonly 1-100 Gbps on the backbone. Customers buy a slice of that.
- **Examples:** a city government linking its offices and schools, a bank joining its branches in one city, a cable TV company's city network.
- **Owner:** usually a carrier or a city. The customer rents the service.

The clear difference from a CAN is land. A CAN stays on your property. A MAN crosses public roads, so a company with the legal right to dig or hang cable usually provides it.

### WAN (Wide Area Network)

A WAN joins networks across long distances: between cities, countries or continents.

- **Size:** from about 100 km to the whole planet. There is no upper limit.
- **Technology:** leased lines (a private link rented from a carrier), [MPLS](../05-routing/13-mpls.md) (a carrier service that forwards traffic by short labels), internet links, and undersea fibre. Almost all traffic between continents travels on undersea fibre. Details are in [WAN Technologies](../10-enterprise-and-datacenter/08-wan-technologies.md).
- **Speed:** from tens of Mbps for a small branch to 100 Gbps and more for large backbones.
- **Examples:** a bank joining its offices in 20 countries, or the internet itself.
- **Owner:** mostly carriers. You pay a monthly fee.

WAN links have more delay, called latency. Light in fibre travels about 200,000 km per second. That adds about 5 ms each way for every 1000 km (ms means millisecond, one thousandth of a second). No upgrade removes this delay. More in [Bandwidth, Latency and Performance](06-network-performance-basics.md).

The word WAN also appears on your home router. The port marked WAN is the one that goes toward your ISP (Internet Service Provider, the company that sells you internet access). The other ports are LAN ports.

### SAN (Storage Area Network)

A SAN is a separate, fast network that connects servers to shared disk storage.

- **Size:** usually inside one data centre. It can be stretched between sites over fibre for disaster recovery.
- **Technology:** Fibre Channel (a storage network standard, with common speeds of 16, 32 and 64 Gbps), iSCSI (storage commands over normal TCP/IP, on TCP port 3260), and NVMe over Fabrics (a newer, very low-delay method).
- **Examples:** a bank's database servers using a shared disk array, or a group of virtual machine hosts using one storage system.
- **Owner:** the organisation that runs the data centre.

A SAN gives **block** access. The server sees a raw disk and puts its own file system on it. To the server, the remote disk looks like a local one.

Do not confuse it with **NAS** (Network Attached Storage). A NAS is a storage box that sits on the normal LAN and shares **files**, using protocols such as SMB and NFS. See [File Sharing and Storage Protocols](../07-application-protocols/13-file-sharing-and-storage-protocols.md).

```text
  Users
    |
  normal LAN (user traffic)
    |
 +--+-----+    +--------+
 |Server 1|    |Server 2|
 +--+-----+    +---+----+
    |              |
 +--+--------------+--+
 |    SAN switches    |   separate network, storage traffic only
 +---------+----------+
           |
 +---------+----------+
 |  Shared disk array |
 +--------------------+
```

The storage is kept apart because storage traffic is heavy and very sensitive to delay. A busy disk should not fight with users watching videos. The other "SAN" you may meet is Subject Alternative Name in TLS certificates, which is unrelated.

### VPN (Virtual Private Network)

A VPN is a private path built on top of another network, usually the internet.

"Virtual" means no private cable exists. "Private" means outsiders cannot read or join the traffic. The method is a **tunnel**: each private packet is wrapped inside another packet, in the same way a letter goes inside a second envelope. A packet is one small piece of data sent on a network.

```text
 Laptop in a cafe                              Company office
 tunnel IP 10.8.0.5                            LAN 10.10.0.0/16
 public IP 198.51.100.40

 +--------+                               +-------------+
 | VPN    |======= encrypted tunnel ======| VPN gateway |--- LAN
 | client |       (over the internet)     | 203.0.113.1 |
 +--------+                               +-------------+

 What travels on the internet:
 +--------------------------+---------------------------------+
 | Outer header             | Encrypted inner packet          |
 | from 198.51.100.40       | from 10.8.0.5 to 10.10.5.20     |
 | to   203.0.113.1         | (the internet cannot read this) |
 +--------------------------+---------------------------------+
```

The two common forms are:

- **Remote access VPN:** one person connects to a company network, as in the picture.
- **Site-to-site VPN:** two offices connect through their gateways, and users do nothing special.

A VPN has no size of its own. It can link two rooms or two continents. Its speed is limited by the network under it. Protocols such as IPsec and WireGuard are covered in [VPNs](../09-security/07-vpn.md).

**Where the analogy stops being true:** the locked box still travels in normal traffic. A VPN hides the content, but it cannot make the road faster or stop a traffic jam.

**Fuller truth:** not every VPN encrypts. An MPLS VPN from a carrier keeps customers apart but does not encrypt by default.

### Intranet, extranet and internet

These three words describe who may enter, and they all use the same technology: IP, DNS and web pages. IP is the addressing system of networks. DNS turns names into addresses.

| | Intranet | Extranet | Internet |
|---|---|---|---|
| Who can enter | Staff only | Staff plus chosen outsiders | Anyone |
| Controlled by | One organisation | One organisation | Nobody as a whole |
| Login needed | Yes | Yes, with limited rights | Depends on the site |
| Example | HR portal at intranet.example.com | Supplier order portal | www.example.com |
| Typical address | Private, such as 10.10.5.20 | Public or through a VPN | Public, such as 203.0.113.10 |

- **Intranet:** a private network service for one organisation. Examples are staff portals, the wiki and internal chat.
- **Extranet:** a controlled part of that private system opened to known outsiders, such as suppliers, partners or customers. Each outsider sees only their own data.
- **Internet:** the global network of networks. Tens of thousands of independent networks join together and exchange routes using [BGP](../05-routing/09-bgp.md).

The small-letter word "internet" can mean any set of joined networks. The capital-letter "Internet" means the single public one. Many writers now use the small letter for both.

The Web is not the internet. The Web is one service (web pages over HTTP) that runs on the internet. Email is another.

Private addresses such as 10.x and 192.168.x are not routed on the public internet. See [Public, Private and Special IP Addresses](../04-ip-addressing/03-public-private-and-special-addresses.md).

### Comparison table

These sizes and speeds are common rules of thumb. No standards body says a LAN must end at a certain distance.

| Type | Typical reach | Typical speed | Usual owner | Real example |
|---|---|---|---|---|
| PAN | 1-10 m | A few Mbps (Bluetooth) | You | Earbuds and a phone |
| LAN | Up to one building | 1-10 Gbps | You or your company | Office floor |
| WLAN | Tens of metres per AP | Hundreds of Mbps | You or your company | Cafe Wi-Fi |
| CAN | 1-5 km | 10-100 Gbps | One organisation | University |
| MAN | 5-50 km | 1-100 Gbps | Carrier or city | City offices |
| WAN | 100 km to global | Tens of Mbps to 100+ Gbps | Carriers | Bank across countries |
| SAN | Usually one data centre | 16-64 Gbps (Fibre Channel) | Data centre owner | Database disks |
| VPN | Any | Limited by the network under it | Company or VPN provider | Staff working from home |

### How to tell them apart

Ask these questions in order:

1. **Is it mainly for storage between servers and disks?** Then it is a SAN.
2. **Is it a private path on top of another network?** Then it is a VPN.
3. **Is it about who may enter?** Staff only is an intranet. Staff plus partners is an extranet. Everyone is the internet.
4. **Does it cover only one person's devices?** Then it is a PAN.
5. **Does it stay in one building or home?** Then it is a LAN, or a WLAN if it uses radio.
6. **Does it stay on one organisation's land, across several buildings?** Then it is a CAN.
7. **Does it stay in one city, using rented links?** Then it is a MAN.
8. **Does it go further?** Then it is a WAN.

A simple ownership test helps a lot. If you own every cable, you are probably looking at a LAN or CAN. If you pay a carrier every month for the link, you are probably looking at a MAN or WAN.

### Other names you may hear

- **VLAN (Virtual LAN):** one physical switch split into several separate LANs. See [VLANs](../03-ethernet-and-switching/06-vlans.md).
- **WWAN (Wireless WAN):** a wide area network over mobile data, such as 4G and 5G. See [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).
- **HAN (Home Area Network):** a LAN inside a home, often with smart devices.
- **BAN (Body Area Network):** wearable or medical sensors on one body. It is a very small PAN.
- **GAN (Global Area Network):** a network spanning the whole planet, such as a global company backbone. It is a very large WAN.
- **SD-WAN:** software that chooses the best of several WAN links. See [SD-WAN](../10-enterprise-and-datacenter/10-sd-wan.md).
- **VPC (Virtual Private Cloud):** a private network you define inside a public cloud. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

## Example

### Example 1: name every network in one organisation

A hospital group has a main campus, a clinic in the same city, and a clinic in another country. All addresses here are examples.

```text
  Partner lab        Doctor at home       Patient on phone
  (extranet)         (VPN to intranet)    (public website)
       \                    |                   /
        \                   |                  /
     +-----------------------------------------------+
     |                PUBLIC INTERNET                |
     +-----------------------+-----------------------+
                             |
               firewall and VPN gateway 203.0.113.1
                             |
 +---------------------------+-------------------------+
 | MAIN CAMPUS (CAN)  10.10.0.0/16                     |
 |                                                     |
 |  Building A LAN   Building B LAN   Campus WLAN      |
 |  10.10.1.0/24     10.10.2.0/24     10.10.20.0/24    |
 |          \              |              /            |
 |           +------ core switches ------+             |
 |                         |                           |
 |                   server room                       |
 |                         |                           |
 |              SAN 10.10.200.0/24 (iSCSI)             |
 +---------+----------------------------+--------------+
           |                            |
     rented metro link           rented WAN link
           |                            |
      City clinic                 Clinic abroad
      10.20.0.0/16                10.30.0.0/16
```

Now look at ten events in one day:

| # | What happens | Network type | The clue |
|---|---|---|---|
| 1 | A doctor's watch sends data to her phone by Bluetooth | PAN | One person, a few metres |
| 2 | A PC at 10.10.1.15 prints to 10.10.1.50 in Building A | LAN | Same building, same range |
| 3 | A nurse's tablet at 10.10.20.31 joins Wi-Fi | WLAN | Radio to an access point |
| 4 | A PC in Building B reads a file from the server room | CAN | Many buildings, one owner |
| 5 | The city clinic opens a patient record on campus | MAN | One city, rented link |
| 6 | The clinic abroad sends an X-ray to campus | WAN | Another country, carrier link |
| 7 | The database server writes to the disk array | SAN | Server-to-disk traffic |
| 8 | A doctor at home reads the staff portal | VPN and intranet | Private path to a staff-only system |
| 9 | A partner lab uploads test results to its portal | Extranet | Known outsider, limited access |
| 10 | A patient opens www.example.com at 203.0.113.10 | Internet | Open to the public |

Event 8 uses two words because they answer two questions. VPN is how the doctor gets in. Intranet is what she reaches.

### Example 2: see the network types from your own computer

This home laptop is on Wi-Fi. Run the commands to see where your LAN ends.

Windows:

```powershell
PS C:\> ipconfig

Windows IP Configuration

Wireless LAN adapter Wi-Fi:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 192.168.1.20
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

Linux (a second laptop on the same Wi-Fi):

```bash
$ ip -4 addr show wlan0
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    inet 192.168.1.21/24 brd 192.168.1.255 scope global dynamic wlan0
       valid_lft 86147sec preferred_lft 86147sec

$ ip route
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.21 metric 600
```

Read the output like this:

1. Windows says "Wireless LAN adapter", so this device is on a WLAN.
2. The mask 255.255.255.0 is the same as /24. It means addresses from 192.168.1.1 to 192.168.1.254 are local.
3. The default gateway, 192.168.1.1, is the home router. It is the exit from the LAN. See [The Default Gateway](../04-ip-addressing/08-default-gateway.md).

Now trace the path to a server on another continent. Traceroute lists each router on the way.

```bash
# Linux
$ traceroute -n 203.0.113.10
traceroute to 203.0.113.10 (203.0.113.10), 30 hops max, 60 byte packets
 1  192.168.1.1  1.214 ms  1.102 ms  1.034 ms
 2  10.99.0.1  8.412 ms  8.130 ms  8.327 ms
 3  198.51.100.1  9.871 ms  9.640 ms  10.021 ms
 4  198.51.100.77  148.205 ms  148.612 ms  149.044 ms
 5  203.0.113.10  150.733 ms  151.108 ms  150.902 ms
```

```powershell
# Windows
PS C:\> tracert -d 203.0.113.10

Tracing route to 203.0.113.10 over a maximum of 30 hops

  1     1 ms     1 ms     1 ms  192.168.1.1
  2     8 ms     8 ms     8 ms  10.99.0.1
  3    10 ms    10 ms    10 ms  198.51.100.1
  4   148 ms   149 ms   149 ms  198.51.100.77
  5   151 ms   151 ms   151 ms  203.0.113.10

Trace complete.
```

| Hop | Address | Time | Where you are |
|---|---|---|---|
| 1 | 192.168.1.1 | About 1 ms | Your LAN or WLAN, at the home router |
| 2 | 10.99.0.1 | About 8 ms | The ISP's local network, across your city |
| 3 | 198.51.100.1 | About 10 ms | The ISP's core network, still nearby |
| 4 | 198.51.100.77 | About 148 ms | The far end of a long WAN link |
| 5 | 203.0.113.10 | About 151 ms | The destination server |

The big jump between hops 3 and 4 is the long distance. The time rose by about 138 ms, and the signal must travel out and back. At 5 ms per 1000 km each way, that fits a path of very roughly 14,000 km. You can see a WAN in the numbers. To read traceroute well, see [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

## Going Deeper

This section is optional at Beginner level.

**Physical and logical networks.** Older names describe physical things. Modern networks are often logical, meaning that software defines them on shared hardware. A VLAN splits one switch. A VPN runs over a shared internet. A cloud VPC is a private network with no cables you can see. The physical network below is called the **underlay**. The logical network on top is called the **overlay**. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

**Why LANs are cheap and WANs are costly.** Inside your building, a cable costs little and you pay once. Across a country, somebody must buy land rights, lay fibre, and repair it. You pay for that with a monthly fee. This is why WAN design tries to send less data, and why branch offices often use caching or local servers.

**The line between private WAN and internet is getting thinner.** Many companies have replaced some MPLS links with normal internet links plus a VPN, and they use SD-WAN to manage the mix. The WAN is now often a design choice rather than a single product.

**The trust boundary has moved.** The old model trusted everyone on the intranet. Modern designs check each user and device on each request, wherever they are. This is covered in [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

## Common Mistakes

- **"Wi-Fi and the internet are the same thing."** Wi-Fi is a local radio link. You can have full Wi-Fi signal and no internet.
- **"WLAN is a different network from LAN."** A WLAN is a LAN that uses radio. It usually shares the same address range as the wired devices.
- **"Size alone decides the type."** Ownership and purpose matter too. A 3 km fibre across a private factory site is a CAN. A 3 km rented link across a city is closer to a MAN or WAN service.
- **"A bigger network is always faster."** It is usually the opposite. A LAN is often much faster than the WAN link that leaves it.
- **"A SAN and a NAS are the same."** A SAN gives raw blocks over a dedicated network. A NAS shares files on the normal LAN.
- **"A VPN is a type of network like a LAN."** A VPN has no size. It is a private path built on top of another network.
- **"A VPN makes me anonymous."** It hides traffic from your local network and ISP. The VPN provider, and the sites you log in to, can still identify you.
- **"A VLAN and a VPN are nearly the same."** A VLAN splits a local network. A VPN joins distant networks across another network.
- **"Intranet means no internet."** An intranet can reach the internet, and staff can reach it from home through a VPN. It only means access to the service is limited to staff.
- **"The Web is the internet."** The Web is one service that runs on the internet.
- **"CAN means the car network."** In networking interviews, CAN usually means Campus Area Network.

## Interview Questions

**Q: What is the difference between a LAN and a WAN?**
A: A LAN covers a small area like a home or a building. You usually own it, and it is fast, often 1 Gbps or more. A WAN joins sites across cities or countries. It usually uses links rented from carriers, costs money every month, and has more latency. A router sits between them.

**Q: Is a WLAN a separate kind of network from a LAN?**
A: No, a WLAN is a LAN that uses radio instead of cable. It uses IEEE 802.11, which is sold as Wi-Fi. An access point links wireless devices to the wired network. The real difference is that wireless devices share the air and mostly take turns, so performance is less steady.

**Q: Give an example of a PAN and say how big it is.**
A: Earbuds connected to a phone by Bluetooth are a PAN. It serves one person and usually covers about 1-10 metres. A smartwatch linked to a phone is another example.

**Q: What is the difference between a CAN and a MAN?**
A: A CAN joins buildings on land that one organisation controls, such as a university, and the organisation usually owns the fibre. A MAN covers a city and crosses public land, so a carrier or a city usually runs it and customers rent the service.

**Q: Is the internet a WAN?**
A: Yes, it is the largest WAN. More exactly, it is a global network of networks. Tens of thousands of independent networks connect and exchange routes using BGP. Nobody owns the whole thing.

**Q: What is a SAN, and how is it different from a NAS?**
A: A SAN is a dedicated network that gives servers block-level access to shared disks, using Fibre Channel or iSCSI. The server treats the disk like a local one. A NAS is a storage device on the normal LAN that shares files using SMB or NFS. So a SAN gives blocks and a NAS gives files.

**Q: What is a VPN, and why does it not fit in the size list?**
A: A VPN is a private path built on top of another network, usually the internet. It wraps packets in a tunnel and normally encrypts them. Its size depends on the two ends, so it can link two rooms or two continents. That is why it is defined by how it works, not by how far it reaches.

**Q: What is the difference between an intranet, an extranet and the internet?**
A: An intranet is a private system for staff, like an HR portal. An extranet opens a controlled part of it to known outsiders, like a supplier portal, where each outsider sees only their own data. The internet is the public global network that anyone can use. All three use the same technology, and the difference is who can get in.

**Q: What is the difference between a VLAN and a VPN?**
A: A VLAN splits one local switched network into separate logical networks. It works at Layer 2 and normally does not encrypt anything. A VPN joins separated devices or sites across another network, usually with encryption. A VLAN divides a local network, and a VPN stretches a private network over distance.

**Q: Why does a home router have a port marked WAN?**
A: That port faces the ISP and leads to the outside world. The LAN ports and Wi-Fi face your own devices. The router moves traffic between the two sides, and it usually does NAT too, so your private addresses can share one public address.

**Q: A company has two offices 300 km apart. What kind of network joins them, and what are the options?**
A: That is a WAN, because it crosses a long distance over links the company does not own. The options are a leased line, an MPLS service from a carrier, or a site-to-site VPN over internet links, often managed with SD-WAN. The choice depends on cost, speed, and how much reliability is needed.

**Q: Why is a WAN usually slower and more costly than a LAN?**
A: Long fibre needs land rights, construction and repairs, so carriers charge a monthly fee and you buy limited bandwidth. Distance adds latency too, about 5 ms each way per 1000 km in fibre. A LAN uses short cables you already own, so high speed is cheap.

## Quick Recap

- By reach, from small to large: PAN, LAN, CAN, MAN, WAN. A WLAN is a LAN over radio.
- The sizes are rules of thumb. Ownership is a strong clue: if you own the cable it is usually a LAN or CAN, and if you rent it it is usually a MAN or WAN.
- A SAN is defined by purpose: fast block storage between servers and disks.
- A VPN is defined by method: a private tunnel over another network, at any size.
- Intranet, extranet and internet are defined by access: staff, staff plus partners, or everyone.
- One network can carry several names, because each name answers a different question.
- Short links are usually faster than long ones, and traceroute lets you see where the WAN starts.
