# The Default Gateway

**Level:** Beginner | **Prerequisites:** [IPv4 Addressing Basics](01-ipv4-basics.md), [Subnet Masks and CIDR](04-subnet-masks-and-cidr.md), [ARP](../03-ethernet-and-switching/05-arp.md)

## In One Line

The default gateway is the IP address of the router on your own network. Your device sends traffic there whenever the destination is outside your subnet.

## Why It Matters

A device can reach neighbours on its own subnet by itself. It cannot reach anything else without help. It does not know the path to a server in another city, and it does not need to. It only needs to know one thing: "if the destination is not local, give it to this router."

Without a default gateway, your laptop can print to the printer in the same room, but it cannot open a website.

Interviewers love this topic because one small question tests several skills at once: subnet masks, ARP, MAC addresses versus IP addresses, and what a router really does. The classic question is: "You ping a server on another network. Whose MAC address is in the frame?" Many candidates get this wrong. By the end of this chapter you will not.

## Simple Explanation

Think of an office building with a mailroom.

- If you have a note for someone on your floor, you walk to their desk yourself.
- If you have a letter for someone in another country, you do not travel there. You take it to the mailroom, and the mailroom handles it.

You do not need to know the route to the other country. You only need to know where the mailroom is. The default gateway is the mailroom.

Now the terms, in plain words:

- **Host:** any device with an IP address, such as a laptop, phone or server.
- **Router:** a device that moves traffic between different networks.
- **Subnet:** a group of addresses that can reach each other directly, without a router.
- **Subnet mask:** a number that shows which part of an IP address is the network part. For example, 255.255.255.0 (also written /24) means the first three numbers name the network.
- **MAC address:** the hardware address of a network card. It is used for delivery on the local network only.
- **Packet:** the IP-level unit of data. It carries the source and destination IP addresses.
- **Frame:** the local delivery wrapper around a packet. On Ethernet, it carries the source and destination MAC addresses.

**Where the analogy stops being true:** a mailroom is the only exit in the building. A real host can have extra routes that send some traffic to other routers. Also, a mailroom sends your letter the whole way as one item, while the real network hands it from router to router, and each router makes its own choice.

## How It Works

### Step 1: The host decides "local or remote"

Before sending anything, the host compares the destination with its own subnet. It uses **its own mask** on both addresses. A bitwise AND keeps the network part and sets the rest to zero. If the two results match, the destination is local.

Host: 192.168.1.10 with mask 255.255.255.0 (/24).

| Destination | Host network | Destination network | Result |
|---|---|---|---|
| 192.168.1.50 | 192.168.1.0 | 192.168.1.0 | Same: local |
| 198.51.100.7 | 192.168.1.0 | 198.51.100.0 | Different: remote |

The host never knows the destination's real mask. It only has its own. That is why a wrong mask on the host causes wrong decisions.

```text
        Packet to send to destination IP D
                      |
                      v
       +-------------------------------+
       | Is D inside my own subnet?    |
       | (compare using MY mask)       |
       +-------------------------------+
            | yes                | no
            v                    v
     ARP for D itself      ARP for the gateway
            |                    |
            v                    v
     Frame to D's MAC      Frame to gateway's MAC
                           (packet is still for D)
```

> **Fuller truth:** a real host does not run a separate "mask test". It looks up the destination in its routing table. The local subnet is one entry in that table, and the default route is another. The most specific match wins. The mask test is a simple way to see the same result. See [The Routing Table](../05-routing/02-routing-table.md).

### Step 2: The host finds the gateway's MAC address with ARP

To put a frame on the wire, the host needs a destination MAC address. **ARP** (Address Resolution Protocol) finds the MAC address that belongs to a local IP address.

For a remote destination, the host asks for the **gateway's** MAC, not the remote server's MAC:

1. The host checks its ARP cache (a short-term memory of IP-to-MAC pairs).
2. If the entry is missing, it broadcasts: "Who has 192.168.1.1? Tell 192.168.1.10."
3. The router replies directly: "192.168.1.1 is at 02:bb:00:00:00:01."
4. The host saves this in its cache for a while. The time varies by operating system.

The host cannot ARP for the far server. ARP is a broadcast, and broadcasts stop at the router. See [ARP](../03-ethernet-and-switching/05-arp.md) for the details.

### Step 3: The host builds the frame and the packet

This is the key idea of the chapter. The packet and the frame have **different destinations**.

```text
|<------------------- Ethernet frame ----------------->|
                     |<--------- IP packet ----------->|
+--------------------+----------------+----------------+
| Ethernet header    | IP header      | Data           |
| dst MAC: gateway   | src IP: host   | (TCP, UDP,     |
| src MAC: host      | dst IP: server |  ICMP ...)     |
+--------------------+----------------+----------------+
```

| Field | Value | Who is it? |
|---|---|---|
| Source MAC | Host's MAC | This host |
| Destination MAC | Gateway's MAC | The next stop |
| Source IP | Host's IP | This host |
| Destination IP | Server's IP | The final stop |

The **gateway's IP address is not in the packet at all**. The host uses it only to find the gateway's MAC. After that, the IP has done its job.

An analogy: the IP packet is a letter with the final address on it. The frame is a courier's label that says "take this to the next office". Each office removes the old label and adds a new one. The address on the letter stays the same.

**Where the analogy stops being true:** on most home and office links to the internet, one router changes the source IP with NAT (Network Address Translation). So the "letter" is not always untouched. See [NAT and PAT](09-nat-and-pat.md).

### Step 4: The router forwards the packet

1. The router receives the frame and sees its own MAC as the destination, so it accepts it.
2. It removes the frame and reads the destination IP.
3. It lowers the TTL (Time To Live, a counter that stops packets from looping forever) by 1 and updates the IP header checksum.
4. It looks up the destination IP in its own routing table and finds the next hop. A **hop** is one step from one router to the next.
5. It builds a **new frame** with new source and destination MAC addresses and sends it out.

Details are in [How Routers Work](../05-routing/01-how-routers-work.md).

### Rules a gateway must follow

- **It must be in the host's own subnet.** The host must reach it with ARP, and ARP works only on the local network.
- **It is an IP address on a router interface.** An **interface** is a network port with its own address. A router has one address per subnet it touches, so each subnet has its own gateway address.
- **It can be any usable address in the subnet.** The first or last address (192.168.1.1 or 192.168.1.254) is only a habit, not a rule.
- **It is needed only for remote traffic.** Two hosts on the same subnet talk directly and never touch the gateway.

### Where a host gets its gateway

| Source | How |
|---|---|
| DHCP | The server hands out the gateway with the IP and mask (the "Router" option, number 3). See [DHCP](../07-application-protocols/04-dhcp.md). |
| Manual | An administrator types it in. Common on servers and printers. |
| IPv6 Router Advertisement | Routers announce themselves on the link. DHCPv6 does not give a gateway. |

DHCP (Dynamic Host Configuration Protocol) is the service that gives a device its network settings automatically.

### Default gateway vs default route

These two terms are close, but not the same.

- A **route** is one line in a routing table: "for this destination, go this way."
- The **default route** is the catch-all line: 0.0.0.0/0 in IPv4 (::/0 in IPv6). The /0 means zero bits must match, so every address matches. It is used only when no more specific route fits.
- The **default gateway** is the next-hop IP address that the default route points to.

```text
default via 192.168.1.1 dev eth0
|-----|     |---------|
default     default gateway
route       (next hop of that route)
```

So the default route is the **rule**, and the default gateway is the **place the rule points to**. On a host, setting a default gateway simply creates a default route.

Routers have default routes too. Cisco calls the next hop the **gateway of last resort**. Routers are not usually said to have a "default gateway". More in [Static and Default Routes](../05-routing/03-static-and-default-routes.md).

### The word "gateway" in other senses

"Gateway" means a door between two different worlds. Early internet documents used the word for what we now call a router. That is why we still say Border Gateway Protocol, and why the column in a routing table is called Gateway. Today the word has many meanings, so always ask "which gateway?"

| Term | What it is | Layer | Works on |
|---|---|---|---|
| Default gateway | The router address a host uses for remote traffic | 3 | IP packets |
| Internet gateway | Cloud component that connects a VPC to the internet | 3 | IP packets |
| NAT gateway | Lets private hosts start outbound connections, while blocking new inbound ones | 3 and 4 | IP addresses and ports |
| API gateway | Single front door for web APIs | 7 | HTTP requests |
| VoIP gateway | Connects internet phone calls to the old phone network | Mixed | Voice calls |

**Internet gateway.** This is an AWS term. A VPC (Virtual Private Cloud) is your private network inside a cloud provider. The internet gateway is attached to the VPC. A public subnet's route table has 0.0.0.0/0 pointing to it. Your server does not use it as its default gateway directly. The server's gateway is the VPC router, at the second address of the subnet (10.0.1.1 in 10.0.1.0/24), and the route table sends traffic on from there. See [AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

**NAT gateway.** A managed cloud service (AWS, Azure and GCP all offer one, with different names and details) or any device doing NAT. A private subnet's route table points 0.0.0.0/0 at it. Servers can download updates, but the internet cannot start a connection to them. See [NAT and PAT](09-nat-and-pat.md).

**API gateway.** Not a router at all. It is software that receives HTTP requests, checks login and rate limits, then sends each request to the right backend service. An API (Application Programming Interface) is a way for programs to call each other. Examples: Amazon API Gateway, Kong. Related topic: [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

**VoIP gateway.** VoIP (Voice over IP) means phone calls over an IP network. This box connects those calls to the PSTN (Public Switched Telephone Network, the traditional phone system). It converts both the call setup, such as SIP, and the voice itself, usually carried in RTP. See [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

Others you may hear: **VPN gateway** (the end point of encrypted tunnels, see [VPNs](../09-security/07-vpn.md)) and **application layer gateway** (a firewall or NAT helper that understands a specific protocol).

### Wrong or missing gateway: the symptoms

The pattern to remember: **local works, remote fails.**

| Problem | What you see | Why |
|---|---|---|
| No gateway | Linux: `Network is unreachable`. Windows: usually `General failure` | The host has no route, so it never sends the packet |
| Gateway IP has no device behind it | `Destination host unreachable`, sent by your own host | ARP for the gateway gets no reply |
| Gateway is a live device, but not a router | Gateway pings fine. Remote pings time out | The device does not forward traffic |
| Gateway is outside the subnet | Windows warns. Linux refuses with an error | The host cannot reach it with ARP |
| Router has no route onward | `Destination net unreachable` from the router's IP | The problem is past the gateway |
| No gateway on a server | Local clients work. Remote clients get no answer | The request arrives, but the reply cannot leave |
| Wrong gateway from DHCP | Some devices fail, others work | A bad scope or a rogue DHCP server, see [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md) |

Error text varies by operating system version.

A wrong **mask** can look like a gateway problem:

- Mask too long (/24 on a host in a real /16 network): the host thinks a nearby host is remote and sends to the gateway. This may work through the router, or it may fail.
- Mask too short (/16 on a host in a real /24 network): the host thinks a remote host is local and ARPs for it. Nobody replies, so it fails, unless the router answers with proxy ARP (see Going Deeper).

A failed gateway often looks like a DNS problem. If your DNS server is on another subnet, names stop working first, and the browser reports a DNS error. Test with an IP address to tell them apart.

**A quick check order:**

1. Read your settings: IP, mask, gateway.
2. Ping the gateway. If this fails, look at the ARP table.
3. Ping a public IP address, such as 8.8.8.8 (Google Public DNS).
4. Ping a name, such as example.com. If step 3 works but this fails, suspect DNS.

Some gateways block ping. A failed ping to the gateway does not prove it is down if the internet still works. Full method: [Troubleshooting Methodology](../12-troubleshooting-and-tools/01-troubleshooting-method.md).

### How to view and set it on each OS

**Windows.** Open PowerShell as Administrator to make changes.

```powershell
# View
ipconfig
route print -4
Get-NetRoute -DestinationPrefix 0.0.0.0/0

# Set IP, mask and gateway together (static)
netsh interface ipv4 set address name="Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1

# Change only the gateway
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -Confirm:$false
New-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -NextHop 192.168.1.1

# Go back to DHCP
netsh interface ipv4 set address name="Ethernet" dhcp
```

**Linux.** The `ip route` changes below are lost after a reboot.

```bash
# View
ip route show default
ip route get 172.16.5.20        # which way would this destination go?

# Set or change (temporary)
sudo ip route add default via 192.168.1.1 dev eth0
sudo ip route replace default via 192.168.1.1 dev eth0

# Remove
sudo ip route del default

# Permanent, on systems using NetworkManager
sudo nmcli con mod "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1
sudo nmcli con up "Wired connection 1"
```

For permanent settings on Ubuntu Server, use Netplan, in a YAML file under `/etc/netplan/`. Then run `sudo netplan apply`. The older `gateway4` key is deprecated.

```text
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.10/24]
      routes:
        - to: default
          via: 192.168.1.1
```

**macOS.**

```bash
# View
route -n get default
networksetup -getinfo "Wi-Fi"

# Set IP, mask and gateway (permanent)
sudo networksetup -setmanual "Wi-Fi" 192.168.1.10 255.255.255.0 192.168.1.1

# Go back to DHCP
sudo networksetup -setdhcp "Wi-Fi"
```

**Cisco IOS.** A router uses a default route. A Layer 2 switch has an IP address only for management, and it uses `ip default-gateway` so that you can reach it from other subnets.

```text
! Router: default route, plus the LAN gateway address
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2

! Layer 2 switch: management address and its gateway
SW1(config)# interface vlan 1
SW1(config-if)# ip address 192.168.1.2 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
SW1(config)# ip default-gateway 192.168.1.1
```

`ip default-gateway` applies when IP routing is off. On a Layer 3 switch with `ip routing` on, use a default route instead.

More commands: [IP Configuration Commands](../12-troubleshooting-and-tools/02-ip-configuration-commands.md).

## Example

PC-A pings a server on another network. Two routers sit between them. All addresses are private, so no NAT is involved.

```text
         192.168.1.0/24      10.0.0.0/30      172.16.5.0/24
 +------+            +----+              +----+            +--------+
 | PC-A |------------| R1 |--------------| R2 |------------| Server |
 +------+       Gi0/0+----+Gi0/1    Gi0/0+----+Gi0/1       +--------+
```

| Device | Interface | IP address | MAC address | Gateway |
|---|---|---|---|---|
| PC-A | eth0 | 192.168.1.10/24 | 02:aa:00:00:00:10 | 192.168.1.1 |
| R1 | Gi0/0 | 192.168.1.1/24 | 02:bb:00:00:00:01 | - |
| R1 | Gi0/1 | 10.0.0.1/30 | 02:bb:00:00:00:02 | - |
| R2 | Gi0/0 | 10.0.0.2/30 | 02:cc:00:00:00:01 | - |
| R2 | Gi0/1 | 172.16.5.1/24 | 02:cc:00:00:00:02 | - |
| Server | eth0 | 172.16.5.20/24 | 02:ee:00:00:00:20 | 172.16.5.1 |

**1. Check the settings on PC-A.**

Linux:

```bash
$ ip -4 addr show eth0 | grep inet
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
$ ip route
default via 192.168.1.1 dev eth0 proto static
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

The same host, if it ran Windows:

```text
C:\> ipconfig
Ethernet adapter Ethernet:
   IPv4 Address. . . . . . . . . . . : 192.168.1.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

C:\> route print -4
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0      192.168.1.1    192.168.1.10     25
      192.168.1.0    255.255.255.0         On-link     192.168.1.10    281
```

**2. Local or remote?**

```text
192.168.1.10 AND 255.255.255.0 = 192.168.1.0   (me)
172.16.5.20  AND 255.255.255.0 = 172.16.5.0    (destination)
Different, so the destination is remote. Use the gateway.
```

Linux confirms this. Note the word `via`:

```bash
$ ip route get 172.16.5.20
172.16.5.20 via 192.168.1.1 dev eth0 src 192.168.1.10 uid 1000
    cache
$ ip route get 192.168.1.50
192.168.1.50 dev eth0 src 192.168.1.10 uid 1000
    cache
```

**3. Watch ARP and the ping on the wire.** Clear the ARP cache, start a capture with tcpdump (a packet capture tool), then ping from a second terminal. The `-e` flag shows the MAC addresses.

```bash
$ sudo ip neigh flush dev eth0
$ sudo tcpdump -eni eth0 -t 'arp or icmp'
02:aa:00:00:00:10 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.1.1 tell 192.168.1.10, length 28
02:bb:00:00:00:01 > 02:aa:00:00:00:10, ethertype ARP (0x0806), length 60: Reply 192.168.1.1 is-at 02:bb:00:00:00:01, length 46
02:aa:00:00:00:10 > 02:bb:00:00:00:01, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 172.16.5.20: ICMP echo request, id 3021, seq 1, length 64
02:bb:00:00:00:01 > 02:aa:00:00:00:10, ethertype IPv4 (0x0800), length 98: 172.16.5.20 > 192.168.1.10: ICMP echo reply, id 3021, seq 1, length 64
```

Read the third line slowly. The IP destination is the server, 172.16.5.20. The MAC destination is the router, 02:bb:00:00:00:01. There is no ARP for 172.16.5.20. ICMP (Internet Control Message Protocol) is the protocol that ping uses.

**4. Follow the packet.** The MAC addresses change on every link. The IP addresses do not.

| Link | Source MAC | Destination MAC | Source IP | Destination IP | TTL |
|---|---|---|---|---|---|
| PC-A to R1 | 02:aa:00:00:00:10 | 02:bb:00:00:00:01 | 192.168.1.10 | 172.16.5.20 | 64 |
| R1 to R2 | 02:bb:00:00:00:02 | 02:cc:00:00:00:01 | 192.168.1.10 | 172.16.5.20 | 63 |
| R2 to Server | 02:cc:00:00:00:02 | 02:ee:00:00:00:20 | 192.168.1.10 | 172.16.5.20 | 62 |

64 is the usual starting TTL on Linux. Windows usually starts at 128.

**5. R1 has no "default gateway". It has a default route.** R1 also needs no special entry to reach PC-A, because 192.168.1.0/24 is directly connected.

```text
R1# show ip route
Gateway of last resort is 10.0.0.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.0.2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, GigabitEthernet0/1
L        10.0.0.1/32 is directly connected, GigabitEthernet0/1
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
```

**6. The reply needs its own path.** The server makes the same decision in reverse. It sees that 192.168.1.10 is remote and sends to **its** gateway, 172.16.5.1. R2 needs a route back, for example `ip route 192.168.1.0 255.255.255.0 10.0.0.1`. If the server had no gateway, the ping would fail even though the request arrived.

```bash
$ ping -c 2 172.16.5.20
PING 172.16.5.20 (172.16.5.20) 56(84) bytes of data.
64 bytes from 172.16.5.20: icmp_seq=1 ttl=62 time=2.41 ms
64 bytes from 172.16.5.20: icmp_seq=2 ttl=62 time=1.87 ms
```

The reply's ttl=62 fits: the server started at 64, and two routers each took off 1.

**7. Now break it.** Set a gateway address where no device exists.

```bash
$ sudo ip route replace default via 192.168.1.254 dev eth0
$ ping -c 1 172.16.5.20
PING 172.16.5.20 (172.16.5.20) 56(84) bytes of data.
From 192.168.1.10 icmp_seq=1 Destination Host Unreachable
$ ip neigh show 192.168.1.254
192.168.1.254 dev eth0 FAILED
```

The error comes from PC-A itself. The ARP entry says `FAILED`. Those two clues point to the gateway address, not to the server.

## Going Deeper

Optional for now. These points often come up as follow-up questions.

**Metrics and two gateways.** A laptop on Ethernet and Wi-Fi has two default routes. The host picks one using a **metric**, a cost number where lower is better. On Linux, the lowest route metric wins. On Windows, the route metric and interface metric are added, and the lowest sum wins. Having two gateways does not normally give you double the speed.

**Proxy ARP.** A router can answer an ARP request on behalf of a remote host. This lets a host with a wrong mask, or no gateway, still reach remote hosts. Cisco IOS has it on by default on interfaces. Many security guides advise turning it off.

**ICMP Redirect.** If a host sends a packet to its gateway, and the gateway sees that a better router is on the same subnet, it may send an ICMP Redirect message to tell the host. Many systems ignore or disable these for security.

**Gateway failure.** A host has one gateway address. If that router dies, the host does not fix itself. First hop redundancy protocols such as VRRP and HSRP let two routers share one virtual IP address, so the gateway survives. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

**IPv6.** Hosts usually learn the gateway from Router Advertisements, and the address is usually a link-local one (starting with fe80::), which is valid only on that link. NDP (Neighbor Discovery Protocol) does the job of ARP. See [IPv6 in Practice](12-ipv6-in-practice.md).

```bash
$ ip -6 route show default
default via fe80::1 dev eth0 proto ra metric 100
```

**On-link exceptions.** The "same subnet" rule has rare exceptions. Some cloud and point-to-point setups give a host a /32 address and use a special route that says the gateway is directly reachable. On Linux, this is `ip route add default via 192.168.1.1 dev eth0 onlink`. The gateway is still one step away. It is just not inside the mask.

**Virtual gateways.** In Docker, the gateway of a container on the default bridge is usually 172.17.0.1, an address on the host. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md). In a VLAN network, the gateway is often an address on a Layer 3 switch. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md).

## Common Mistakes

- **Thinking the destination MAC is the server's MAC.** For a remote server, it is the gateway's MAC. The host never learns the far server's MAC.
- **Thinking the gateway's IP goes in the packet.** It does not. The destination IP stays the server's IP. The gateway's IP is used only for ARP.
- **Thinking traffic on the same subnet goes through the gateway.** Same-subnet traffic goes directly, usually through a switch.
- **Thinking two hosts on one switch can always talk.** If they are in different subnets, they need a router, even on the same switch.
- **Setting a gateway outside the subnet.** The host cannot ARP for it, so it cannot use it.
- **Believing .1 is always the gateway.** That is a habit, not a rule. Check the settings.
- **Using "default gateway" and "default route" as if identical.** The route is the table entry. The gateway is its next hop.
- **Forgetting the return path.** A server with no gateway receives requests but cannot answer remote clients.
- **Forgetting `ip default-gateway` on a Layer 2 switch.** You can manage the switch from its own subnet, but not from others.
- **Assuming that "gateway" always means a router.** An API gateway works on HTTP requests. It does not route packets.
- **Blaming DNS too early.** A dead gateway can look like a DNS failure. Test with an IP address first.

## Interview Questions

**Q: What is a default gateway?**
A: It is the IP address of the router on my local subnet. My host sends traffic there when the destination is outside my subnet.

**Q: How does a host know whether to use the gateway?**
A: It applies its own subnet mask to its own IP and to the destination IP. If the network parts match, the destination is local and it ARPs for it directly. If not, it sends to the gateway. Really, this is a routing table lookup, and the connected route or the default route wins.

**Q: You ping a server on another network. What are the source and destination MAC and IP in the first frame?**
A: Source MAC is my MAC. Destination MAC is my gateway's MAC. Source IP is my IP. Destination IP is the server's IP.

**Q: What changes at each router?**
A: The MAC addresses change on every link because each link gets a new frame. The IP addresses stay the same, unless NAT is used. The TTL goes down by one, and the header checksum is updated.

**Q: Does the host ARP for the remote server?**
A: No. ARP works only on the local network. It ARPs for the gateway and reuses that entry for all remote traffic.

**Q: What is the difference between a default route and a default gateway?**
A: The default route is the routing table entry 0.0.0.0/0, which matches anything with no better route. The default gateway is the next-hop IP of that route. On a host, setting the gateway creates the route.

**Q: Why must the gateway be in the same subnet as the host?**
A: Because the host has to reach it directly with ARP to learn its MAC. If it is outside the subnet, the host would need a router to reach its router. Rare on-link setups are the exception.

**Q: A user can ping their gateway but not the internet. What do you check?**
A: The gateway is reachable, so the local side is fine. I would check whether the router has a default route, whether its uplink is up, and whether NAT is working. Traceroute shows where the path stops. I would also test with an IP to rule out DNS.

**Q: A host gets "Destination host unreachable" from its own IP. What does that suggest?**
A: The host could not complete ARP for the next hop. If the destination is remote, the gateway may be wrong, down or disconnected. I would check the configured gateway and the ARP table.

**Q: What does `ip default-gateway` do on a Cisco switch?**
A: It gives a Layer 2 switch a gateway for its own management traffic, so that I can reach it from other subnets. It does not route user traffic. On a Layer 3 switch with routing on, I would use a default route.

**Q: What is the difference between an internet gateway and a NAT gateway in AWS?**
A: An internet gateway connects a VPC to the internet and allows inbound and outbound traffic for instances with public IPs. A NAT gateway lets instances in a private subnet start outbound connections, but the internet cannot start connections to them.

**Q: Is an API gateway the same kind of gateway?**
A: No. A default gateway forwards IP packets at Layer 3. An API gateway works at Layer 7 on HTTP requests, doing things like authentication, rate limits and routing to backend services.

## Quick Recap

- The default gateway is the local router's IP address, used for all traffic with no better route.
- The host uses its own mask to decide local or remote. Local goes direct. Remote goes to the gateway.
- The host ARPs for the gateway, so the frame carries the gateway's MAC while the packet carries the server's IP.
- Each router builds a new frame. MAC addresses change on every link, and IP addresses stay the same unless NAT is used.
- The default route (0.0.0.0/0) is the rule. The default gateway is its next hop.
- A missing or wrong gateway means local works and remote fails. Check settings, ping the gateway, read the ARP table.
- "Gateway" also names other things (internet, NAT, API, VoIP gateways). Ask which one is meant.
