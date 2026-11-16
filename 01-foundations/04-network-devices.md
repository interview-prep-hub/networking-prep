# Network Devices

**Level:** Beginner | **Prerequisites:** [What Is a Network?](01-what-is-a-network.md), [Network Topologies](03-network-topologies.md)

## In One Line

Network devices are the boxes that carry, forward, filter and share traffic. The biggest difference between them is how much of each message they read before they decide what to do.

## Why It Matters

Two computers can talk over one cable. Real networks have thousands of computers, long distances, wireless users, attackers and busy websites. No single box solves all of that, so each problem got its own device:

- Signals get weak over distance: repeater.
- Many computers need to share one network: switch.
- Separate networks need to reach each other: router.
- Laptops need to connect without cables: access point.
- Bad traffic must be stopped: firewall.
- One server cannot handle all the users: load balancer.

Interviewers ask about devices because the answer shows if you understand layers. "Hub vs switch vs router" is one of the most common networking questions. A good answer also helps in real work: when something breaks, you need to know which box makes which decision.

## Simple Explanation

Think of a large office building and its mail.

- **NIC:** the mail slot in your office door. Your mail enters and leaves here.
- **Repeater:** a person in a long hallway who hears a message and shouts it again, clearly, so it reaches the far end.
- **Hub:** a person who receives a letter and reads it aloud to everybody on the floor. Everyone hears it. Only one person can speak at a time.
- **Switch:** a mail clerk who knows which office each person sits in, and walks the letter to that office only.
- **Router:** the post office. It does not care about office numbers. It looks at the city and sends the letter toward that city.
- **Modem:** a translator who turns your letter into a form that can travel on the carrier's line.
- **Access point:** a radio operator who passes messages between people outside the building and the mail system inside.
- **Firewall:** a security guard at the door who checks every visitor against a list.
- **Proxy:** an assistant who goes out and buys things for you, so the shop sees the assistant and not you.
- **Load balancer:** a receptionist who sends each new customer to the least busy clerk.
- **Gateway:** a border office where things are translated from one country's system to another's.

Where this analogy stops: a real clerk thinks and can read the letter's meaning. Devices follow fixed rules, and they handle millions of messages per second. Most of them never look at the content, only at the address labels. Also, one real box often does several of these jobs.

## How It Works

### The idea that organises everything: layers

Networking is split into **layers**. A layer is one level of the job, with its own kind of address. The standard list is the OSI model, with seven layers. You will learn it fully in [The OSI Model](../02-network-models/02-osi-model.md). For now you only need this map:

| Layer | Name | Unit of data | What identifies the target | Devices that decide here |
|---|---|---|---|---|
| 7 | Application | Message | Website name, URL path | Proxy, L7 load balancer, next-generation firewall, gateway |
| 5-6 | Session, Presentation | Data | Rarely used to classify a device | None commonly |
| 4 | Transport | Segment | Port number | L4 load balancer, stateful firewall |
| 3 | Network | Packet | IP address | Router, Layer 3 switch |
| 2 | Data Link | Frame | MAC address | NIC, bridge, switch, access point |
| 1 | Physical | Bits | Nothing, just signals | NIC, repeater, hub, modem |

Some terms in that table:

- **Bit:** a single 0 or 1.
- **Frame:** a package of data for one local network. It carries MAC addresses.
- **Packet:** the data inside a frame. It carries IP addresses and can cross many networks.
- **MAC address:** a 48-bit hardware address such as `00:00:5e:00:53:20`. It is used on the local network.
- **IP address:** a logical address such as `192.168.1.20`. It shows which network a device is on.
- **Port:** a number that picks the program on a device. HTTPS normally uses TCP port 443. TCP and UDP are the two main Layer 4 protocols.

The rule to remember: **the higher the layer, the more the device reads.** A hub reads nothing. A switch reads the MAC address. A router reads the IP address. A proxy can read the web request.

### NIC (Network Interface Card)

A **NIC** is the hardware that connects a device to a network. It is usually built into the motherboard. It can also be a Wi-Fi chip or a USB adapter.

- **Layer:** 1 and 2. It sends and receives signals (Layer 1) and builds and checks frames (Layer 2).
- **Key fact:** every NIC has a MAC address, which the maker sets. One computer with Ethernet and Wi-Fi has two NICs and two MAC addresses.
- **Filtering:** a NIC accepts frames sent to its own MAC address, broadcasts (frames for everyone on the network) and multicast groups it has joined. It drops the rest. **Promiscuous mode** turns this filter off, and capture tools such as Wireshark use it.
- **Fuller truth:** modern operating systems can change the MAC address in software, and phones often use a random one on each Wi-Fi network. Virtual machines have virtual NICs with no hardware at all.

### Repeater

A **repeater** receives a weak signal and sends a clean, strong copy.

- **Layer:** 1. It does not understand frames, only signals.
- **Why needed:** signals fade with distance. Copper Ethernet is limited to 100 metres per cable run. A repeater allows a longer path.
- **Repeater vs amplifier:** an amplifier makes everything louder, including noise. A repeater rebuilds the 0s and 1s, so the noise is removed.
- **Today:** you rarely buy one for wired Ethernet. Switches do this as a side effect, because each port sends a fresh signal. See [Cables and Transmission Media](05-cables-and-media.md).

### Hub

A **hub** is a repeater with many ports. A bit that comes in on one port is copied out of all other ports.

- **Layer:** 1. It has no idea who the data is for.
- **Problem 1, no privacy:** every computer receives every frame.
- **Problem 2, shared speed:** ten computers on a 100 Mbps hub share 100 Mbps in total.
- **Problem 3, collisions:** a **collision** happens when two devices send at the same time and their signals destroy each other. All devices on a hub are in one **collision domain**, the group of devices whose signals can collide. Each device must listen first and retry after a collision. This method is called **CSMA/CD** (Carrier Sense Multiple Access with Collision Detection).
- **Half duplex:** a hub link works in **half duplex**, which means send or receive, but not both at once.

Hubs are obsolete. They matter today because interviewers use them to test your understanding of switches.

```text
 HUB: one frame in, copied to all     SWITCH: one frame in, sent to one

      A --->[  HUB  ]---> B                A --->[ SWITCH ]---> B
              |    |                                |    |
              v    v                             (quiet) (quiet)
              C    D                                C    D
```

### Bridge

A **bridge** joins two network segments and reads MAC addresses to decide if a frame needs to cross. A **segment** is one section of a network.

- **Layer:** 2.
- **How it helps:** if the sender and receiver are on the same side, the bridge does not forward the frame. Traffic stays local, and each side gets its own collision domain.
- **Bridge vs switch:** a switch is a bridge with many ports and fast hardware. Classic bridges had few ports and often worked in software. Standards from the IEEE (the group that writes the Ethernet and Wi-Fi standards) still use the word "bridge" for what we call a switch.
- **Today:** standalone bridges are rare, but the idea is everywhere. Linux bridges connect virtual machines and containers. A Wi-Fi bridge joins two wired networks over radio.

### Switch

A **switch** connects devices inside one network and sends each frame only to the port that needs it. This is the main device of a modern wired LAN (Local Area Network, the network in one building or site).

**Layer:** 2. Some models also route, as Going Deeper explains below.

A switch keeps a **MAC address table**, a list of which MAC address is on which port. It works like this:

1. A frame arrives on a port.
2. The switch reads the **source** MAC address and writes down "this MAC is on this port". This is **learning**.
3. The switch reads the **destination** MAC address and looks it up.
4. If found, it sends the frame out of that one port. This is **forwarding**.
5. If not found, or if it is a broadcast, it sends the frame out of all ports except the one it came in on. This is **flooding**.

Results:

- Each port is its own collision domain.
- Links run in **full duplex**, which means sending and receiving at the same time, so collisions do not happen.
- A 1 Gbps switch can give each port 1 Gbps in each direction, if its internal capacity is large enough.
- All ports are still in one **broadcast domain** by default. A broadcast domain is the group of devices that receive each other's broadcasts. A **VLAN** (Virtual LAN) splits one switch into several broadcast domains. See [VLANs](../03-ethernet-and-switching/06-vlans.md).

Full detail is in [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md).

### Router

A **router** connects different IP networks and picks the path for each packet. Your home network and your internet provider's network are different networks, so a router sits between them.

**Layer:** 3.

For each packet, a router does this:

1. It receives a frame addressed to its own MAC address.
2. It removes the frame and reads the destination IP address.
3. It looks in its **routing table**, a list of known networks and where to send traffic for each. The most specific match wins.
4. It lowers the **TTL** (Time To Live) by 1. TTL is a counter in the packet. At 0, the packet is dropped, so it cannot loop forever. IPv6 calls this the hop limit.
5. It builds a **new frame** with new MAC addresses for the next link.
6. It sends the frame out of the correct interface.

Results:

- Routers do not forward broadcasts, so each interface is its own broadcast domain.
- If no route matches and there is no **default route** (the "send everything else here" entry), the packet is dropped.
- The IP addresses stay the same along the path, unless NAT changes them. The MAC addresses change at every hop.

See [How Routers Work](../05-routing/01-how-routers-work.md).

### Modem

**Modem** means **mo**dulator-**dem**odulator. **Modulation** means putting digital data onto a signal that suits a certain line. A modem does this for the provider's line, and reverses it for incoming data.

| Connection | Line | Device |
|---|---|---|
| DSL | Telephone copper | DSL modem |
| Cable internet | Coaxial cable | Cable modem (uses the DOCSIS standard) |
| Fibre to the home | Optical fibre | ONT (Optical Network Terminal) |
| Mobile | Radio | 4G or 5G modem |

- **Layer:** usually taught as Layer 1. **Fuller truth:** real modems also handle the Layer 2 protocol of the provider's link, so "Layers 1 and 2" is more exact.
- **Not a router:** a modem gets you onto the provider's line. A router chooses paths and joins networks.
- **ONT note:** an ONT is not strictly a modem, but people often call it one, because it does the same job: it turns the provider's light signal into Ethernet.

### Access Point

An **access point (AP)** connects wireless devices to a wired network.

- **Layer:** 1 and 2. It handles radio signals and converts between Wi-Fi frames (IEEE 802.11) and Ethernet frames (IEEE 802.3).
- **Not a router:** a pure AP has no routing job. A phone on the AP is on the same network as the wired PCs. Its IP address comes from the same network.
- **SSID:** the **Service Set Identifier** is the Wi-Fi network name that users see.
- **Shared air:** radio is a shared medium, more like a hub than a switch. Wi-Fi devices avoid collisions with a method called CSMA/CA (the "A" means avoidance), so more users means less speed each. **Fuller truth:** Wi-Fi 6 and later can serve several devices at once with OFDMA, which splits a channel among users. This reduces the problem but does not remove it.

See [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md).

### Firewall

A **firewall** allows or blocks traffic using rules. A rule looks like this:

| # | Direction | Source | Destination | Service | Action |
|---|---|---|---|---|---|
| 1 | Inbound | Any | 203.0.113.50 | TCP 443 | Allow |
| 2 | Inbound | Any | Any | Any | Deny |

Rules are usually checked from the top. The first match decides. The last rule usually denies everything else.

- **Packet filter (Layers 3 and 4):** checks IP addresses and ports, one packet at a time.
- **Stateful firewall (Layers 3 and 4):** also remembers each connection in a **state table**. Replies to traffic you started are allowed automatically.
- **Next-generation firewall (up to Layer 7):** also recognises applications and users, and can inspect content.

Firewalls can be separate boxes, or software on a host, such as Windows Defender Firewall or nftables on Linux. See [Firewalls](../09-security/04-firewalls.md).

### Gateway

**Gateway** is the most confusing word in this chapter, because it has several meanings.

1. **Classic meaning:** a device that connects two systems that use different protocols and translates between them. Example: a VoIP gateway connects internet phone calls to the traditional phone network. Because it may need to understand the whole message, textbooks say it can work at all layers, up to Layer 7.
2. **Default gateway:** the router your computer sends traffic to when the destination is outside its own network. This is a Layer 3 job. See [The Default Gateway](../04-ip-addressing/08-default-gateway.md).
3. **Product names:** NAT gateway, VPN gateway, API gateway, internet gateway in cloud networks. Each is a specific service, and they work at different layers.

When an interviewer says "gateway", answer with the meaning, then state which one you are using.

### Proxy

A **proxy** sits between a client and a server. It ends one connection and opens another. The server talks to the proxy, not the real client.

**Layer:** mostly 7, because it understands application protocols such as HTTP. Some proxies, such as SOCKS proxies, are more generic and only relay connections.

```text
Forward proxy: works for the clients

  Client A --+
             +--> [Forward proxy] ---> Internet ---> Any website
  Client B --+

Reverse proxy: works for the servers

                                                  +--> Server 1
  Any client ---> Internet ---> [Reverse proxy] --+
                                                  +--> Server 2
```

- **Forward proxy:** the client knows about it, or the company forces it. Uses: web filtering, logging, caching (keeping copies of content to answer faster), hiding client addresses.
- **Reverse proxy:** clients think it is the website. Uses: ending TLS (the encryption behind HTTPS), caching, hiding servers, sending requests to the right backend. NGINX is a common example.
- **HTTPS note:** a normal forward proxy cannot read HTTPS content. The client asks it to open a tunnel, so the proxy sees the target hostname but not the page. Reading content needs TLS interception, where company devices must trust the proxy's certificate.

Because the server sees the proxy's IP address, proxies often add the real client address in an HTTP header such as `X-Forwarded-For`. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

### Load Balancer

A **load balancer** spreads incoming traffic across several servers. Users connect to one **virtual IP (VIP)**, an address that belongs to the service and not to one server.

```text
                                       +--> Web 1  10.0.0.11  healthy
 Clients ---> VIP 203.0.113.80 [ LB ] -+--> Web 2  10.0.0.12  healthy
                                       +-X  Web 3  10.0.0.13  failed
```

- **Layer 4 load balancer:** decides using IP addresses and ports. It does not read the request. It is fast and works for any TCP or UDP service. AWS Network Load Balancer is an example.
- **Layer 7 load balancer:** reads the request, such as the URL path. It can send `/images` to one group of servers and `/api` to another. AWS Application Load Balancer is an example. NGINX and HAProxy can do both Layer 4 and Layer 7.
- **Health checks:** the load balancer tests each server regularly and stops sending traffic to failed ones. This gives reliability as well as capacity.
- **Common methods:** round robin (take turns), least connections (pick the least busy), IP hash (the same client goes to the same server).

A Layer 7 load balancer is a kind of reverse proxy. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

### Why a home router is several devices in one box

The box your provider gives you is called a "router", but it does many jobs. People at home want one cheap box and one power plug, so makers put all these parts together.

```text
                    Provider line
                         |
 +-----------------------|-------------------------------+
 | Home "router" box     |                               |
 |                 +-----+------+                        |
 |                 |   Modem    |  (only in combo boxes) |
 |                 +-----+------+                        |
 |                       | WAN side                      |
 |                 +-----+------+                        |
 |                 | Firewall   |                        |
 |                 | and NAT    |                        |
 |                 +-----+------+                        |
 |                       |                               |
 |                 +-----+------+    +---------------+   |
 |                 |   Router   |----| DHCP and DNS  |   |
 |                 +-----+------+    | services      |   |
 |                       | LAN side  +---------------+   |
 |                +------+-------+                       |
 |                | Internal     |                       |
 |                | bridge       |                       |
 |                +--+--------+--+                       |
 |                   |        |                          |
 |            +------+---+ +--+---------+                |
 |            | 4-port   | | Wi-Fi      |                |
 |            | switch   | | access pt  |                |
 |            +-+-+-+-+--+ +-----+------+                |
 +--------------|-|-|-|----------|-----------------------+
             LAN ports      Radio waves
             (PC, TV)       (phones, laptops)
```

| Part inside the box | Job at home | Layer |
|---|---|---|
| Modem (some boxes) | Connects to the provider's line | 1, 2 |
| Router | Moves traffic between home network and internet | 3 |
| NAT | Shares one public IP address among all home devices | 3, 4 |
| Firewall | Blocks unrequested traffic from the internet | 3, 4 |
| Switch | Connects the wired LAN ports | 2 |
| Access point | Connects Wi-Fi devices | 1, 2 |
| DHCP server | Gives each device an IP address automatically | 7 |
| DNS forwarder | Passes name lookups to the provider or a public resolver | 7 |

**NAT** (Network Address Translation) changes addresses in packets as they pass through. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md). **DHCP** is covered in [DHCP](../07-application-protocols/04-dhcp.md).

In the diagram, the WAN (Wide Area Network) side faces the provider, and the LAN side faces your home. A **public IP address** is one that can be reached from the internet. **DNS** turns names such as example.com into IP addresses.

In an office, these parts are usually separate boxes. Separate boxes cost more, but they are faster, can be replaced one at a time, and can handle thousands of users. The internal order in the diagram is simplified, and it varies by product.

### Comparison table

| Device | OSI layer | Reads | Main job | Separates collision domains? | Separates broadcast domains? |
|---|---|---|---|---|---|
| NIC | 1, 2 | Own frames | Connects a host to the network | Not applicable | Not applicable |
| Repeater | 1 | Nothing, just signal | Regenerates the signal | No | No |
| Hub | 1 | Nothing, just signal | Copies bits to all ports | No | No |
| Bridge | 2 | MAC address | Joins segments, filters frames | Yes | No |
| Switch | 2 | MAC address | Forwards frames inside a LAN | Yes, per port | No, unless VLANs are used |
| Router | 3 | IP address | Joins networks, chooses paths | Yes | Yes |
| Modem | 1 (with some 2) | Line signal | Converts for the provider's line | Not applicable | No |
| Access point | 1, 2 | MAC address | Joins Wi-Fi to the wired LAN | No, radio is shared | No |
| Firewall | 3, 4, up to 7 | IP, port, state, sometimes app | Allows or blocks traffic | Depends on setup | Depends on setup |
| Gateway | Up to 7 | Depends on type | Translates between systems | Depends on type | Depends on type |
| Proxy | Mostly 7 | Application request | Acts for clients or servers | Not its purpose | Not its purpose |
| Load balancer | 4 or 7 | Ports, or request content | Spreads traffic across servers | Not its purpose | Not its purpose |

### Other devices you may be asked about

- **Media converter and transceiver:** turn one physical medium into another, such as copper to fibre. Layer 1. See [Cables and Transmission Media](05-cables-and-media.md).
- **Wi-Fi extender:** repeats a Wi-Fi network by radio. Each repeat costs airtime, so speed drops.
- **PoE switch:** a switch that sends electrical power over the Ethernet cable (Power over Ethernet) to phones, cameras and APs. The IEEE 802.3bt standard allows up to 90 W at the switch port.
- **Wireless LAN controller:** manages many APs from one place. See [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).
- **IDS and IPS:** an Intrusion Detection System reports attacks, and an Intrusion Prevention System also blocks them. See [IDS, IPS and SIEM](../09-security/06-ids-and-ips.md).
- **VPN concentrator:** ends many encrypted remote connections. See [VPNs](../09-security/07-vpn.md).

## Example

### Example 1: follow one web request through every device

A laptop on home Wi-Fi opens a website. These addresses are used:

| Item | Value |
|---|---|
| Laptop IP and mask | 192.168.1.20/24 |
| Laptop MAC | 00:00:5e:00:53:20 |
| Router LAN IP (default gateway) | 192.168.1.1 |
| Router LAN MAC | 00:00:5e:00:53:01 |
| Router WAN IP (public) | 203.0.113.50 |
| Provider's router | 203.0.113.1 |
| Web server for example.com, made up for this example | 198.51.100.10 |

```text
 Laptop     AP       Switch/bridge   Router+NAT+FW   Modem   Provider
 [NIC]~~~~[Wi-Fi]------[ L2 ]----------[ L3 ]-------[ L1 ]-----> ...
 .1.20                                 .1.1 | .50

 ~~~~ radio      ---- wired inside the home box or by cable
```

First, look at the laptop's own view.

Linux:

```bash
$ ip -br link
lo       UNKNOWN  00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
wlp2s0   UP       00:00:5e:00:53:20 <BROADCAST,MULTICAST,UP,LOWER_UP>

$ ip route
default via 192.168.1.1 dev wlp2s0 proto dhcp src 192.168.1.20 metric 600
192.168.1.0/24 dev wlp2s0 proto kernel scope link src 192.168.1.20 metric 600

$ ip neigh show 192.168.1.1
192.168.1.1 dev wlp2s0 lladdr 00:00:5e:00:53:01 REACHABLE
```

Windows:

```powershell
PS> Get-NetAdapter -Name "Wi-Fi" | Format-Table Name, Status, MacAddress, LinkSpeed

Name  Status MacAddress        LinkSpeed
----  ------ ----------        ---------
Wi-Fi Up     00-00-5E-00-53-20 866.7 Mbps

PS> Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Format-Table NextHop, InterfaceAlias

NextHop     InterfaceAlias
-------     --------------
192.168.1.1 Wi-Fi

PS> arp -a 192.168.1.1

Interface: 192.168.1.20 --- 0xc
  Internet Address      Physical Address      Type
  192.168.1.1           00-00-5e-00-53-01     dynamic
```

The outputs show the NIC, the default gateway and the gateway's MAC address. **ARP** is the protocol that finds that MAC address. See [ARP](../03-ethernet-and-switching/05-arp.md).

Now the request travels:

| Step | Device | Layer | What happens |
|---|---|---|---|
| 1 | NIC | 1, 2 | Builds a frame. Destination MAC is the router, 00:00:5e:00:53:01, not the web server. Destination IP is 198.51.100.10. Sends it by radio. |
| 2 | Access point | 1, 2 | Receives the Wi-Fi frame and passes it toward the wired side. IP addresses are not touched. |
| 3 | Internal bridge | 2 | Sees the destination MAC is the router and delivers it. |
| 4 | Router | 3 | Finds no specific route, so uses the default route to 203.0.113.1. TTL drops from 64 to 63. Linux starts at 64 and Windows at 128. |
| 5 | Firewall | 3, 4 | Outbound traffic is allowed. Saves the connection in its state table. |
| 6 | NAT | 3, 4 | Changes the source from 192.168.1.20:51514 to 203.0.113.50:40112 and saves the mapping. |
| 7 | Modem | 1, 2 | Converts the data into the provider's line signal. |
| 8 | Provider routers | 3 | Each one lowers the TTL and builds a new frame, until the packet reaches 198.51.100.10. |

The reply comes back to 203.0.113.50:40112. The firewall finds the matching state entry and allows it. NAT changes the destination back to 192.168.1.20:51514. Then the AP delivers it by radio.

Now use traceroute to see the routers on the path. Traceroute works by using the TTL. See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

Linux:

```bash
$ traceroute -n 198.51.100.10
traceroute to 198.51.100.10 (198.51.100.10), 30 hops max, 60 byte packets
 1  192.168.1.1  1.214 ms  1.102 ms  1.067 ms
 2  203.0.113.1  8.431 ms  8.390 ms  8.512 ms
 3  198.51.100.1  14.220 ms  14.301 ms  14.175 ms
 4  198.51.100.10  15.012 ms  14.987 ms  15.103 ms
```

Windows:

```powershell
PS> tracert -d 198.51.100.10

Tracing route to 198.51.100.10 over a maximum of 30 hops

  1     1 ms     1 ms     1 ms  192.168.1.1
  2     8 ms     8 ms     8 ms  203.0.113.1
  3    14 ms    14 ms    14 ms  198.51.100.1
  4    15 ms    15 ms    15 ms  198.51.100.10

Trace complete.
```

Only routers and the final host appear. The AP, the switch and the modem are not listed, because they do not lower the TTL. This is a quick way to prove that they work below Layer 3.

If a business built the same network with separate Cisco devices, the tables would look like this.

Cisco IOS, on the switch:

```text
Switch# show mac address-table dynamic
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0000.5e00.5301    DYNAMIC     Gi0/24
   1    0000.5e00.5320    DYNAMIC     Gi0/1
```

Cisco IOS, on the router (output trimmed):

```text
Router# show ip route
Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
      203.0.113.0/24 is variably subnetted, 2 subnets, 2 masks
C        203.0.113.0/24 is directly connected, GigabitEthernet0/1
L        203.0.113.50/32 is directly connected, GigabitEthernet0/1
```

The switch thinks in MAC addresses and ports. The router thinks in networks and next hops. That is the Layer 2 and Layer 3 difference in one view.

### Example 2: count the domains

This is a classic interview exercise.

```text
                    [ Router ]
                    /        \
            [ Switch A ]   [ Hub B ]
             /   |   \      /  |  \
           PC1  PC2  PC3  PC4 PC5 PC6
```

**Broadcast domains:** only the router separates them. It has two interfaces in use, so the answer is **2**.

**Collision domains:**

1. Switch A has four ports in use (PC1, PC2, PC3, router link). That gives 4 domains.
2. Hub B and everything on it share 1 domain, including the router link on that side.
3. Total: 4 + 1 = **5**.

On full-duplex switch links, collisions do not really occur, but the counting convention stays the same. More practice is in [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).

## Going Deeper

- **Layer 3 switch:** a switch that also routes between VLANs, using fast hardware chips called ASICs (Application-Specific Integrated Circuits). It is common inside campus networks and data centres. A traditional router usually offers more WAN (long-distance link) types and richer edge features. The line is blurry today, and vendors sell boxes that do both. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md).
- **Layer labels describe the forwarding decision, not the whole box:** a Layer 2 switch can have an IP address so that admins can log in to it with SSH (secure remote login). It still forwards user traffic using MAC addresses.
- **Firewall modes:** most firewalls run in routed mode, where they act like a router. Many can also run in transparent mode, where they filter traffic while acting like a bridge.
- **Virtual devices:** many of these devices now exist as software. A hypervisor (software that runs virtual machines) has virtual switches, Docker uses a Linux bridge, and cloud platforms offer routers, firewalls and load balancers as services. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md).

## Common Mistakes

- **"A router and a modem are the same thing."** A modem connects you to the provider's line. A router moves packets between networks. Many home boxes contain both, which causes the confusion.
- **"Wi-Fi is the same as the internet."** Wi-Fi only connects you to the AP. Internet access depends on the router and provider behind it.
- **"A switch stops broadcasts."** A plain switch floods them. Only a router, or VLANs, separates broadcast domains.
- **"A switch needs IP addresses to forward traffic."** A Layer 2 switch forwards using MAC addresses only.
- **"An AP is a router."** A pure AP puts wireless clients on the same network as the wired LAN. The home box only appears to route because it contains a router too.
- **"The destination MAC address is the final server."** When the server is on another network, the destination MAC is the default gateway's. The MAC address changes at each router hop.
- **"A firewall and a router do the same job."** A router asks "where should this go?". A firewall asks "should this be allowed?". One box can do both, but the jobs differ.
- **"A proxy and a VPN are the same."** A proxy usually handles one application's traffic. A VPN usually carries all of the device's IP traffic through an encrypted tunnel.
- **"A load balancer only adds capacity."** Health checks also keep the service alive when a server fails.
- **"Gateway always means default gateway."** The word has several meanings. State which one you mean.

## Interview Questions

**Q: What is the difference between a hub, a switch and a router?**
A: A hub is Layer 1 and copies bits out of every port. A switch is Layer 2 and uses a MAC address table to send a frame only to the right port. A router is Layer 3 and uses IP addresses and a routing table to move packets between networks.

**Q: Which OSI layer does each common device work at?**
A: Repeaters and hubs are Layer 1. Bridges and switches are Layer 2. Routers are Layer 3. A NIC and an access point work at Layers 1 and 2. Firewalls work at Layers 3 and 4, and next-generation ones go up to 7. Proxies are mostly Layer 7, and load balancers are Layer 4 or 7.

**Q: What is the difference between a bridge and a switch?**
A: They do the same job: both learn MAC addresses and forward frames. A switch is basically a bridge with many ports and fast hardware. Each switch port is its own collision domain.

**Q: Does a switch split broadcast domains?**
A: Not by default. It separates collision domains, one per port, but all ports share one broadcast domain. To split it, use VLANs or a router.

**Q: What does a switch do with a frame when the destination MAC is unknown?**
A: It floods the frame out of every port in that VLAN except the one it arrived on. When the destination replies, the switch learns its port from the source MAC of the reply.

**Q: What is the difference between a modem and a router?**
A: A modem converts data into the signal used on the provider's line, such as DSL or cable. A router joins networks and chooses where packets go. A home box often has both inside.

**Q: What is inside a typical home router?**
A: A router, NAT, a stateful firewall, a small switch, a Wi-Fi access point, a DHCP server and a DNS forwarder, and sometimes a modem. They are combined to reduce cost and make setup simple.

**Q: What is the difference between an access point and a router?**
A: An access point is a Layer 2 device that connects wireless clients to an existing network. A router is a Layer 3 device that connects different networks. Clients on an AP stay in the same IP network as the wired devices.

**Q: What is a gateway, and how is it different from a router?**
A: Classically, a gateway connects systems with different protocols and translates between them, possibly up to Layer 7. A default gateway is simply the router a host uses to reach other networks. So the same word covers both, and I would clarify which one is meant.

**Q: What is the difference between a forward proxy and a reverse proxy?**
A: A forward proxy works for clients: it reaches the internet for them and can filter or log traffic. A reverse proxy works for servers: clients connect to it, and it passes requests to backend servers. It often handles TLS and caching too.

**Q: What is the difference between a Layer 4 and a Layer 7 load balancer?**
A: A Layer 4 load balancer decides using IP addresses and ports, without reading the content, so it is fast and protocol-independent. A Layer 7 load balancer reads the request, so it can route by URL path or headers. It does more work for each request.

**Q: Why don't switches appear in a traceroute?**
A: Traceroute depends on devices lowering the TTL and sending back an ICMP error message (a network control message) when it reaches zero. Routers do that. Layer 2 switches forward frames without touching the TTL, so they stay invisible.

## Quick Recap

- The higher the layer, the more a device reads: hub nothing, switch MAC, router IP, proxy the request.
- Repeater and hub: Layer 1. Bridge and switch: Layer 2. Router: Layer 3.
- NIC, access point and, in practice, modem work at Layers 1 and 2.
- Firewalls filter at Layers 3 and 4, and up to 7 on next-generation models. Proxies are mostly Layer 7. Load balancers are Layer 4 or 7.
- Switches separate collision domains. Routers also separate broadcast domains.
- "Gateway" has several meanings, so always say which one you mean.
- A home router is a router, NAT, firewall, switch, access point, DHCP server and DNS forwarder in one box.
