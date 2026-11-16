# Static and Default Routes

**Level:** Intermediate | **Prerequisites:** [How Routers Work](01-how-routers-work.md), [The Routing Table](02-routing-table.md), [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md)

## In One Line

A static route is a path that a person types into a router or host by hand, and a default route is the special static route (0.0.0.0/0) used when nothing more specific matches.

## Why It Matters

A router forwards a packet only if its routing table has a matching route. A router learns its own connected networks by itself. For every other network, something must teach it. A person can do this, or a routing protocol can.

Static routes are the person doing it. They solve real problems every day:

- A branch office with one link to head office needs one default route, not a full routing protocol.
- Your laptop has a default route right now. Without it, you could reach only your own subnet.
- A cloud route table, such as an AWS VPC route table, is a list of static routes. See [AWS VPC basics](../11-cloud-and-containers/03-aws-vpc-basics.md).
- A cheap backup link can sit unused until the main link fails.

Interviewers like this topic because it tests four things at once: longest prefix match, administrative distance, the return path, and how failures are detected. A candidate who can explain why a ping fails even though "the route is there" shows real understanding.

## Simple Explanation

Think of a route as directions to a city.

A **static route** is a printed road map on the wall. Someone wrote "To reach City C, take the road to Town B." It costs nothing to keep. It never changes by itself. If a bridge falls down far away, the map still says to use that bridge.

A **dynamic route** is a GPS app with live traffic. It reacts to problems, but it uses power and data, and it needs setup. See [Dynamic Routing Overview](04-dynamic-routing-overview.md).

A **default route** is the sign that says "All other places: take the highway." If a driver does not see a specific sign for a city, they follow this one.

A **floating static route** is the spare tyre. It is in the car all the time, but you use it only when the main tyre fails.

Where the analogy stops: a router is a little smarter than a paper map. If the router's own interface goes down, it removes the static routes that depend on that interface. But it knows nothing about failures further away, and that is exactly where the paper map comparison stays true.

## How It Works

### When to Use Static Routes

Use the right tool for the size and shape of the network.

| Situation | Better choice | Why |
|---|---|---|
| Stub network (only one way out) | Static default | There is only one possible path |
| Home or small office to ISP | Static default, or one learned from DHCP | One exit, no choice to make |
| Backup link for a dynamic protocol | Floating static | Simple and silent until needed |
| Host with two gateways on one LAN | Static route on the host | Sends one subnet to the second router |
| Discarding unwanted address space | Static to Null0 | No protocol is needed to drop traffic |
| Many routers with several paths | Dynamic routing | Automatic failover and less typing |
| Network that changes often | Dynamic routing | Manual changes do not scale |

A **stub network** is a network with a single exit, like a street with a dead end.

The cost of static routing grows fast. Take n routers in a line, with a LAN at each end. There are n+1 networks, and each router connects to 2 of them, so each one needs n-1 static routes. Three routers need 6 routes in total. Ten routers need 90, and one new subnet means logging in to every router. This is why large networks use dynamic routing, often with a few static routes at the edges.

The benefits: static routes use no bandwidth and almost no CPU, because routers send no route updates. The path is exactly what you typed. There are no protocol messages for an attacker to fake.

The costs: they do not adapt to remote failures, every change is manual, and a typing mistake can cut off a site.

Static routes do not forward packets faster. After a route is in the forwarding table, the source of the route does not change forwarding speed.

### When a Static Route Enters the Routing Table

Typing the command is not enough. Configuration and routing table are two separate things. On Cisco IOS, the router installs a static route only when all of these are true:

1. The next hop can be reached, or the exit interface is up.
2. No route to the same prefix with a lower administrative distance exists.

A **prefix** is a network address plus its length, such as 192.168.3.0/24. **Administrative distance (AD)** is a trust score for a route source. Lower wins. A static route has AD 1 by default on Cisco.

If the conditions stop being true, the router removes the route from the table. It stays in the configuration and returns when the conditions are true again.

### Syntax on Cisco IOS

The general form:

```text
! Cisco IOS, global configuration mode
ip route <prefix> <mask> <next-hop-ip> [distance] [name <text>]
         [track <number>] [tag <number>] [permanent]

ip route <prefix> <mask> <exit-interface> [next-hop-ip] [distance]
```

Note that IOS uses a dotted mask, not /24. The **next hop** is the IP address of the next router on the path. The **exit interface** is the local port the packet leaves through.

There are three forms. All three below tell R1 how to reach 192.168.3.0/24 through R2 (10.0.12.2).

```text
! Cisco IOS
! Form 1: next hop only (recursive route)
R1(config)# ip route 192.168.3.0 255.255.255.0 10.0.12.2

! Form 2: exit interface only
R1(config)# ip route 192.168.3.0 255.255.255.0 GigabitEthernet0/1

! Form 3: exit interface and next hop (fully specified)
R1(config)# ip route 192.168.3.0 255.255.255.0 GigabitEthernet0/1 10.0.12.2
```

| Form | Good for | Risk |
|---|---|---|
| Next hop | Most Ethernet links | Next hop may resolve through a default route after a failure |
| Exit interface | Point-to-point links only | Bad on Ethernet (see below) |
| Fully specified | Ethernet links, best practice for reliability | Slightly more typing |

Why is form 2 bad on Ethernet? An Ethernet segment can hold many devices. The router does not know which neighbor to send to. So it treats every destination as if it were on the local wire and sends an ARP request for the final destination's IP. This works only if the next router answers on behalf of that address. That behaviour is called **proxy ARP**. It fills the ARP table with one entry per destination, and it fails if proxy ARP is turned off. IOS warns you when you try this with a default route:

```text
R1(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/1
%Default route without gateway, if not a point-to-point interface,
may impact performance
```

ARP is covered in [ARP](../03-ethernet-and-switching/05-arp.md).

Useful optional words:

| Option | Meaning | Example |
|---|---|---|
| distance | Sets the AD, 1-255. AD 255 means never install | `... 10.0.13.2 200` |
| name | A comment stored with the route | `name BACKUP-TO-R3` |
| track | Route depends on a tracked object | `track 1` |
| tag | A number that route maps can match on | `tag 100` |
| permanent | Keep the route even if the interface goes down | `permanent` |

Read a static route in the table like this:

```text
S     192.168.3.0/24 [1/0] via 10.0.12.2
|     |                |  |      |
|     prefix           |  |      next hop
source (S = static)    AD metric
```

The **metric** is the cost of a path, used to compare routes from the same source. Static routes on IOS show metric 0.

Other Cisco operating systems differ. NX-OS accepts `ip route 192.168.3.0/24 10.0.12.2`. IOS XR puts static routes under `router static`. Always check which one you are on.

### Syntax on Linux

Host example: PC1 is 192.168.1.10/24. Its main gateway is 192.168.1.1. A second router at 192.168.1.254 leads to a lab network, 172.16.50.0/24.

```bash
# Linux, needs root
sudo ip route add 172.16.50.0/24 via 192.168.1.254 dev eth0
sudo ip route add default via 192.168.1.1
sudo ip route del 172.16.50.0/24

# Replace if it exists, add if it does not
sudo ip route replace 172.16.50.0/24 via 192.168.1.254

# See the table, and ask which route a destination would use
ip route show
ip route get 172.16.50.20
```

Example output:

```text
$ ip route show
default via 192.168.1.1 dev eth0 proto static metric 100
172.16.50.0/24 via 192.168.1.254 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10 metric 100

$ ip route get 172.16.50.20
172.16.50.20 via 192.168.1.254 dev eth0 src 192.168.1.10 uid 1000
    cache
```

Three Linux facts matter:

- `ip route add` lasts only until reboot. To keep a route, put it in the network configuration.
- Linux has no AD. A lower metric wins between routes to the same prefix.
- A Linux machine forwards packets for others only if `net.ipv4.ip_forward=1`. See [Linux Networking](../11-cloud-and-containers/01-linux-networking.md).

To make a route permanent on Ubuntu, use netplan:

```text
# /etc/netplan/01-eth0.yaml (Ubuntu, then run: sudo netplan apply)
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.10/24]
      routes:
        - to: default
          via: 192.168.1.1
        - to: 172.16.50.0/24
          via: 192.168.1.254
```

On systems that use NetworkManager, such as RHEL, the connection name may differ from the interface name:

```bash
# Linux with NetworkManager
sudo nmcli connection modify eth0 +ipv4.routes "172.16.50.0/24 192.168.1.254"
sudo nmcli connection up eth0
```

The older command `route add -net 172.16.50.0 netmask 255.255.255.0 gw 192.168.1.254` comes from the net-tools package. It still appears in old scripts, but `ip` is the current tool.

### Syntax on Windows

Same example, on a Windows PC. Run as Administrator.

```powershell
# Windows PowerShell
New-NetRoute -DestinationPrefix "172.16.50.0/24" -InterfaceAlias "Ethernet" -NextHop 192.168.1.254 -RouteMetric 10
Get-NetRoute -DestinationPrefix "172.16.50.0/24" -PolicyStore ActiveStore
Find-NetRoute -RemoteIPAddress 172.16.50.20
Remove-NetRoute -DestinationPrefix "172.16.50.0/24" -Confirm:$false
```

```powershell
# Windows, classic route.exe (works in cmd and PowerShell)
route print -4
route add 172.16.50.0 mask 255.255.255.0 192.168.1.254 metric 10
route -p add 172.16.50.0 mask 255.255.255.0 192.168.1.254 metric 10
route delete 172.16.50.0
```

The `-p` flag means persistent: the route survives a reboot. Without it, a `route add` route is lost at restart. `New-NetRoute` writes to both the active and the persistent store by default. Add `-PolicyStore ActiveStore` if you want a temporary route.

Example output:

```text
PS C:\> route print -4
...
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0      192.168.1.1     192.168.1.10     25
      172.16.50.0    255.255.255.0    192.168.1.254     192.168.1.10     35
      192.168.1.0    255.255.255.0         On-link      192.168.1.10    281
===========================================================================
Persistent Routes:
  Network Address          Netmask  Gateway Address  Metric
      172.16.50.0    255.255.255.0    192.168.1.254      10
```

The metric on the active route is 35, not 10. Windows adds the interface metric (25 here) to the route metric (10). Check interface metrics with `Get-NetIPInterface`. If a PC has Wi-Fi and Ethernet, each with a default gateway, the lowest total metric wins.

### Side by Side

| Task | Cisco IOS | Linux | Windows |
|---|---|---|---|
| Add a route | `ip route 172.16.50.0 255.255.255.0 192.168.1.254` | `ip route add 172.16.50.0/24 via 192.168.1.254` | `route add 172.16.50.0 mask 255.255.255.0 192.168.1.254` |
| Add a default | `ip route 0.0.0.0 0.0.0.0 192.168.1.1` | `ip route add default via 192.168.1.1` | `route add 0.0.0.0 mask 0.0.0.0 192.168.1.1` |
| Add a backup | AD: `... 192.168.1.2 200` | Metric: `... metric 200` | Metric: `... metric 200` |
| View table | `show ip route` | `ip route show` | `route print -4` |
| Which route is used? | `show ip route 172.16.50.20` | `ip route get 172.16.50.20` | `Find-NetRoute -RemoteIPAddress 172.16.50.20` |
| Survive reboot | `copy running-config startup-config` | netplan or nmcli | `route -p add` or `New-NetRoute` |

### IPv6 Static Routes

The ideas are the same. The default route is ::/0.

```text
! Cisco IOS
R1(config)# ipv6 unicast-routing
R1(config)# ipv6 route 2001:db8:3::/64 2001:db8:12::2
R1(config)# ipv6 route ::/0 GigabitEthernet0/1 fe80::2
```

```bash
# Linux
sudo ip -6 route add 2001:db8:3::/64 via 2001:db8:12::2
sudo ip -6 route add default via fe80::2 dev eth0
```

```powershell
# Windows PowerShell
New-NetRoute -DestinationPrefix "::/0" -InterfaceAlias "Ethernet" -NextHop fe80::2
```

A next hop that is a **link-local address** (one starting fe80::) exists only on one link. The same fe80:: address can appear on several links. So you must also name the interface. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### The Default Route: 0.0.0.0/0

A route matches a packet if the first N bits agree, where N is the prefix length. In 0.0.0.0/0, N is zero. Zero bits must agree, so every IPv4 address matches.

Routers pick the **longest prefix match**: the most specific matching route wins. A /0 is the least specific route possible, so it wins only if nothing else matches.

```text
Packet to 192.168.3.10. Which route wins?

  0.0.0.0/0          matches (0 bits compared)
  192.168.0.0/16     matches (16 bits compared)
  192.168.3.0/24     matches (24 bits compared)   <-- WINNER
  10.0.0.0/8         no match
```

On Cisco, the default route has a mask of 0.0.0.0:

```text
! Cisco IOS
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2

R1# show ip route | include Gateway|S\*
Gateway of last resort is 10.0.12.2 to network 0.0.0.0
S*    0.0.0.0/0 [1/0] via 10.0.12.2
```

The **gateway of last resort** is the Cisco name for the next hop of the default route. The star in `S*` marks a candidate default. If the output says `Gateway of last resort is not set`, a packet with no matching route is dropped. The router may then send back an ICMP Destination Unreachable message. See [ICMP, Ping and Traceroute](10-icmp-ping-and-traceroute.md).

The default gateway on your PC is the same idea. DHCP (or you) gives the host a default route that points to the local router. See [DHCP](../07-application-protocols/04-dhcp.md).

Related Cisco commands that often confuse people:

| Command | Used on | Meaning |
|---|---|---|
| `ip route 0.0.0.0 0.0.0.0 x.x.x.x` | Router, or Layer 3 switch with `ip routing` on | A real default route |
| `ip default-gateway x.x.x.x` | Layer 2 switch with routing off | Gateway for the switch's own management traffic only. Ignored when `ip routing` is on |
| `ip default-network x.x.x.x` | Router | Older way to mark a network as a default candidate. Rarely seen in new designs |
| `default-information originate` | Under a routing protocol such as OSPF | Tells other routers about your default. See [OSPF](06-ospf.md) |

A default route should point toward the exit. If two routers each use the other as default, and neither knows the destination, packets bounce between them. This is a routing loop.

### Floating Static Routes

A **floating static route** is a static route with an AD set higher than the route it backs up. It "floats" above the routing table, not installed, until the better route goes away.

Default AD values on Cisco IOS:

| Route source | Default AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP internal | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP external | 170 |
| iBGP | 200 |
| Default route learned by DHCP on a router interface | 254 |
| Unusable | 255 |

These values are vendor defaults, not a standard. Junos, for example, calls this route preference and uses 5 for static and 10 for OSPF internal.

To back up a route, pick an AD higher than the primary and below 255:

- Backing up a normal static (AD 1): any value from 2 to 254 works.
- Backing up OSPF (AD 110): use more than 110, such as 130.
- Backing up eBGP (AD 20): use more than 20, but remember that iBGP is 200 if the same prefix comes from there too.

```text
Normal state: both routes are known, only one is installed

  Candidates for 0.0.0.0/0
    AD 1    via 10.0.12.2   installed in the table
    AD 200  via 10.0.13.2   waiting (not visible in show ip route)

After the primary is removed

  Candidates for 0.0.0.0/0
    AD 1    via 10.0.12.2   next hop unreachable, removed
    AD 200  via 10.0.13.2   installed in the table
```

On Linux and Windows, the same effect comes from a higher metric. `sudo ip route add default via 192.168.1.2 metric 200` sits behind a default with metric 100.

Linux has a catch. If the cable is pulled, a plain kernel route often stays in the table, flagged `linkdown`, and the kernel may keep using it. Network managers may remove it for you, and the setting `net.ipv4.conf.all.ignore_routes_with_linkdown=1` tells the kernel to skip such routes.

### Recursive Lookup

A **recursive lookup** means the router looks in the routing table more than once for one packet. A route gives a next hop IP, but that IP is not an interface. So the router must look up the next hop too.

The analogy: you ask for Mr. Khan's shop, and the answer is "It is next to the post office." You still do not know which way to walk, so you ask where the post office is. "Down this road." Now you can start. The analogy stops here: a router usually works this out once in advance, and it does not stop and ask for each packet.

Suppose R1 has `ip route 192.168.3.0 255.255.255.0 10.0.12.2`:

```text
Packet to 192.168.3.10 arrives at R1

Lookup 1: 192.168.3.10
  matches 192.168.3.0/24 via 10.0.12.2
  This gives an IP, not an interface. Look again.

Lookup 2: 10.0.12.2
  matches 10.0.12.0/30, directly connected, GigabitEthernet0/1
  This gives an interface. Stop.

ARP: what is the MAC address of 10.0.12.2?
Send the frame out G0/1 to that MAC address.
```

A key interview point: the next hop IP is never written into the packet. The destination IP stays 192.168.3.10. The next hop is used only to find the MAC address for the Ethernet frame.

Recursion can go deeper. With `ip route 172.20.0.0 255.255.0.0 10.9.9.9` and `ip route 10.9.9.9 255.255.255.255 10.0.12.2`, a packet to 172.20.1.1 needs three lookups: the /16, then the /32, then the connected /30.

In fact, modern routers do not repeat these lookups per packet. Cisco Express Forwarding (CEF) resolves the chain ahead of time and stores the result in the FIB (Forwarding Information Base), the fast table used for real forwarding. You can see the result:

```text
R1# show ip route 192.168.3.10
Routing entry for 192.168.3.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.0.12.2
      Route metric is 0, traffic share count is 1

R1# show ip cef 192.168.3.10
192.168.3.0/24
  nexthop 10.0.12.2 GigabitEthernet0/1
```

The route says only "via 10.0.12.2". CEF has already added the interface.

Linux behaves differently. The gateway in `ip route add` must be reachable directly on a link. If it is not, the command fails with `Error: Nexthop has invalid gateway.` The kernel does not chase a chain of gateways. Routing software such as FRR can resolve recursion and then install the final, direct next hop. Linux also has an `onlink` flag, which tells the kernel to treat the gateway as directly reachable even if no route says so.

## Example

### Three-Router Lab

This lab connects two LANs through three routers and gives them an Internet exit. A backup link joins R1 and R3 for the floating route later. G0/0 is short for GigabitEthernet0/0.

```text
                                            ISP (203.0.113.1)
                                                  |
                                                  | 203.0.113.0/30
                                                  |
 PC1 ----- R1 ------------- R2 ------------------ R3 ----- PC3
   LAN A    |  10.0.12.0/30     10.0.23.0/30      |   LAN C
   192.168.1.0/24                                 |   192.168.3.0/24
            |                                     |
            +---------- backup link --------------+
                         10.0.13.0/30
```

| Device | Interface | Address | Connects to |
|---|---|---|---|
| PC1 | eth0 | 192.168.1.10/24 | LAN A, gateway 192.168.1.1 |
| R1 | G0/0 | 192.168.1.1/24 | LAN A |
| R1 | G0/1 | 10.0.12.1/30 | R2 |
| R1 | G0/2 | 10.0.13.1/30 | R3 (backup) |
| R2 | G0/0 | 10.0.12.2/30 | R1 |
| R2 | G0/1 | 10.0.23.1/30 | R3 |
| R3 | G0/0 | 10.0.23.2/30 | R2 |
| R3 | G0/1 | 192.168.3.1/24 | LAN C |
| R3 | G0/2 | 203.0.113.2/30 | ISP |
| R3 | G0/3 | 10.0.13.2/30 | R1 (backup) |
| PC3 | eth0 | 192.168.3.10/24 | LAN C, gateway 192.168.3.1 |

#### Step 1: Plan Who Needs What

Each router knows only its connected networks. Ask for each one: what is missing?

- R1 is a stub. Everything beyond it is one way out. One default route is enough.
- R2 sits in the middle. It needs a route to each LAN, and a default toward the Internet.
- R3 is the exit. Its default goes to the ISP. It needs specific routes back to everything behind R2.

The rule is: defaults point toward the Internet, and specific routes point back toward the LANs.

#### Step 2: Configure the Routes

```text
! Cisco IOS on R1
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2

! Cisco IOS on R2
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.12.1
R2(config)# ip route 192.168.3.0 255.255.255.0 10.0.23.2
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.23.2

! Cisco IOS on R3
R3(config)# ip route 192.168.1.0 255.255.255.0 10.0.23.1
R3(config)# ip route 10.0.12.0 255.255.255.252 10.0.23.1
R3(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

R3 also needs NAT before the LANs can really use the Internet, because an ISP does not route private addresses. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

#### Step 3: Verify the Tables

```text
R1# show ip route
Codes: L - local, C - connected, S - static, ...
       (rest of the legend removed)

Gateway of last resort is 10.0.12.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.12.2
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        10.0.12.0/30 is directly connected, GigabitEthernet0/1
L        10.0.12.1/32 is directly connected, GigabitEthernet0/1
C        10.0.13.0/30 is directly connected, GigabitEthernet0/2
L        10.0.13.1/32 is directly connected, GigabitEthernet0/2
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0

R2# show ip route static
(legend removed)
Gateway of last resort is 10.0.23.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.23.2
S     192.168.1.0/24 [1/0] via 10.0.12.1
S     192.168.3.0/24 [1/0] via 10.0.23.2
```

#### Step 4: Test

Ping from R1 using its LAN address as the source. A plain `ping` would use 10.0.12.1 as the source, which does not test the route back to LAN A.

```text
R1# ping 192.168.3.10 source 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.10, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/6 ms
```

From PC1:

```bash
# Linux on PC1
$ traceroute -n 192.168.3.10
traceroute to 192.168.3.10 (192.168.3.10), 30 hops max, 60 byte packets
 1  192.168.1.1  1.204 ms  1.121 ms  1.087 ms
 2  10.0.12.2  2.433 ms  2.390 ms  2.351 ms
 3  10.0.23.2  3.512 ms  3.480 ms  3.446 ms
 4  192.168.3.10  4.021 ms  3.988 ms  3.952 ms
```

The packet walk, in both directions:

| Direction | Device | Destination | Route used | Sent to |
|---|---|---|---|---|
| Forward | PC1 | 192.168.3.10 | Default | 192.168.1.1 (R1) |
| Forward | R1 | 192.168.3.10 | 0.0.0.0/0 | 10.0.12.2 (R2) |
| Forward | R2 | 192.168.3.10 | 192.168.3.0/24 | 10.0.23.2 (R3) |
| Forward | R3 | 192.168.3.10 | Connected | PC3 |
| Return | PC3 | 192.168.1.10 | Default | 192.168.3.1 (R3) |
| Return | R3 | 192.168.1.10 | 192.168.1.0/24 | 10.0.23.1 (R2) |
| Return | R2 | 192.168.1.10 | 192.168.1.0/24 | 10.0.12.1 (R1) |
| Return | R1 | 192.168.1.10 | Connected | PC1 |

Each router decides alone. R1 does not know the full path. It knows only the next step.

#### Step 5: Break the Return Path

Remove one route on R3 and see what happens:

```text
R3(config)# no ip route 192.168.1.0 255.255.255.0 10.0.23.1
```

```bash
# Linux on PC1
$ traceroute -n 192.168.3.10
traceroute to 192.168.3.10 (192.168.3.10), 30 hops max, 60 byte packets
 1  192.168.1.1  1.198 ms  1.102 ms  1.076 ms
 2  10.0.12.2  2.410 ms  2.377 ms  2.340 ms
 3  * * *
 4  * * *
```

The request reaches R3 and PC3, but the replies do not come back. R3 now matches 192.168.1.10 against its default and sends the reply to the ISP, which drops it. The forward path is fine. This is the most common static routing fault. Put the route back before going on:

```text
R3(config)# ip route 192.168.1.0 255.255.255.0 10.0.23.1
```

#### Step 6: Add a Floating Default on R1

```text
! Cisco IOS on R1
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.13.2 200 name BACKUP-VIA-R3

! Cisco IOS on R3: return path for LAN A over the backup link
R3(config)# ip route 192.168.1.0 255.255.255.0 10.0.13.1 200 name BACKUP-VIA-R1
```

Nothing changes in the routing table yet. The floating route is in the running configuration only. Check it with `show running-config | include ip route`.

Now fail the primary link:

```text
R1(config)# interface GigabitEthernet0/1
R1(config-if)# shutdown
%LINK-5-CHANGED: Interface GigabitEthernet0/1, changed state to
administratively down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1,
changed state to down

R1# show ip route static
(legend removed)
Gateway of last resort is 10.0.13.2 to network 0.0.0.0

S*    0.0.0.0/0 [200/0] via 10.0.13.2
```

R1 failed over. But PC1 still cannot ping PC3. Why?

#### Step 7: Find the Remote Failure Problem

R3 did not notice anything. Its link to R2 is still up, so its primary route stays:

```text
R3# show ip route 192.168.1.0
Routing entry for 192.168.1.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.0.23.1
      Route metric is 0, traffic share count is 1
```

R2 lost its link to 10.0.12.1. Its traffic for LAN A now follows its default route, back to R3. R3 sends it to R2 again. The two routers bounce the packet until its TTL (Time To Live, a counter that drops by one at each hop) reaches 0.

```text
PC1 --> R1 --> backup link --> R3 --> PC3     forward path works

PC3 --> R3 --> R2 --> R3 --> R2 --> R3 ...    reply loops, then dies
```

```text
R3# traceroute 192.168.1.1 numeric
Type escape sequence to abort.
Tracing the route to 192.168.1.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.23.1 2 msec 1 msec 1 msec
  2 10.0.23.2 2 msec 2 msec 2 msec
  3 10.0.23.1 3 msec 3 msec 3 msec
  4 10.0.23.2 3 msec 4 msec 3 msec
```

Alternating addresses in a traceroute are the signature of a routing loop.

This is the big lesson: **a floating static route reacts only to a failure the router can see by itself.**

#### Step 8: Fix It With Tracking

**IP SLA** is a Cisco feature that sends test traffic on a schedule. **Object tracking** links the result to a route. When the test fails, the route is removed.

On R3, probe R1's address on the primary link. Tie the primary route to that probe.

```text
! Cisco IOS on R3
R3(config)# ip sla 1
R3(config-ip-sla)# icmp-echo 10.0.12.1 source-interface GigabitEthernet0/0
R3(config-ip-sla-echo)# frequency 5
R3(config-ip-sla-echo)# exit
R3(config)# ip sla schedule 1 life forever start-time now
R3(config)# track 1 ip sla 1 reachability

R3(config)# no ip route 192.168.1.0 255.255.255.0 10.0.23.1
R3(config)# ip route 192.168.1.0 255.255.255.0 10.0.23.1 track 1
```

The probe target is covered by the untracked route to 10.0.12.0/30 via R2, and that route has no backup. So the probe can never escape over the backup link and succeed by accident. This matters. If a probe can use the backup path, it passes during a failure, the primary comes back, the probe fails, and the route flaps forever.

R1 needs the same protection against a failed R2-R3 link, because R1's G0/1 stays up in that case. Use the same `ip sla` and `track` commands on R1 with target 10.0.23.2 and source G0/1. Then pin the probe to the primary path with a host route, and track the default:

```text
! Cisco IOS on R1, after creating ip sla 1 and track 1
R1(config)# ip route 10.0.23.2 255.255.255.255 10.0.12.2
R1(config)# no ip route 0.0.0.0 0.0.0.0 10.0.12.2
R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.12.2 track 1
```

Now both ends fail over. Check with `show ip sla statistics` and `show track 1`. Notice how much configuration this took for three routers. That is a good sign that OSPF is the better tool once a network has more than one path.

## Going Deeper

### Why Next-Hop Routes Can Survive a Failure

On classic IOS, a next hop can resolve through any matching route, including the default. Take R3 from the lab, with `ip route 192.168.1.0 255.255.255.0 10.0.23.1`. If G0/0 goes down, the connected route to 10.0.23.0/30 disappears. But 10.0.23.1 still matches 0.0.0.0/0 via the ISP. So the static route can stay in the table, and traffic for LAN A goes to the ISP.

The fully specified form prevents this, because the route depends on the interface:

```text
ip route 192.168.1.0 255.255.255.0 GigabitEthernet0/0 10.0.23.1
```

Behaviour varies by platform and version, so test failover in a lab before you trust it.

### Silent Failures and Black Holes

A **black hole** is a place where packets are dropped with no warning. A classic one appears when a switch sits between two routers:

```text
 R1 G0/1 ---- [switch] ---- G0/0 R2 (crashed)
      link up        link up or down, R1 cannot tell
```

R1's own link to the switch stays up. Its next hop still looks reachable, so the static route stays. Traffic is sent to a dead router. ARP entries eventually time out, but the route remains in the table.

Fixes:

- IP SLA with tracking, as in the lab.
- BFD (Bidirectional Forwarding Detection, RFC 5880), a lightweight fast hello protocol. Some platforms can tie it to static routes.
- A dynamic routing protocol, whose hello messages detect a dead neighbor.

### Null0 Routes

**Null0** is a virtual interface that drops everything sent to it. It does not send a reply, and it uses almost no CPU.

Example. A customer owns 198.51.100.0/24 and uses only 198.51.100.0/25. The ISP sends the whole /24 to the customer, and the customer has a default back to the ISP.

```text
Packet to 198.51.100.200 (unused address)

ISP --> Customer --> default route --> ISP --> Customer ...
```

The packet loops until TTL expires. A discard route fixes this:

```text
! Cisco IOS on the customer router
ip route 198.51.100.0 255.255.255.0 Null0
```

Longest prefix match keeps real traffic working. The connected /25 beats the /24. Only the unused addresses hit Null0.

```bash
# Linux equivalent
sudo ip route add blackhole 198.51.100.0/24
```

### Summary Static Routes

One static route can cover many networks. If R3 had LANs from 192.168.0.0/24 through 192.168.3.0/24, R2 could use a single route:

```text
ip route 192.168.0.0 255.255.252.0 10.0.23.2
```

That is 192.168.0.0/22. See [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md).

### Equal-Cost Static Routes

Two static routes to one prefix with the same AD are both installed. This is called **ECMP** (Equal-Cost Multi-Path). With CEF, traffic is normally shared per flow, so packets of one conversation stay on one path.

```text
! Cisco IOS
ip route 0.0.0.0 0.0.0.0 10.0.12.2
ip route 0.0.0.0 0.0.0.0 10.0.13.2
```

```bash
# Linux
sudo ip route add default nexthop via 10.0.12.2 weight 1 nexthop via 10.0.13.2 weight 1
```

### Static Routes and Dynamic Protocols Together

A static route with AD 1 beats OSPF, EIGRP and RIP for the same prefix. A static route left behind after a migration can silently override the protocol. That is a common cause of "OSPF has the route but the router does not use it."

Static routes can also be shared with `redistribute static` under a routing protocol, and a default route can be advertised with `default-information originate` under OSPF. Details belong in [OSPF](06-ospf.md) and [Advanced Routing Techniques](12-advanced-routing-techniques.md).

In a network with VRFs (separate routing tables on one router), a static route belongs to one table: `ip route vrf CUSTOMER-A 0.0.0.0 0.0.0.0 10.0.12.2`.

### Tunnels and Recursive Routing

A tunnel has its own recursion trap. If a router learns the route to the tunnel's far end through the tunnel itself, the tunnel depends on itself. IOS then shuts it down with a message such as `%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing`. The fix is a static host route to the tunnel's far end through the physical interface. See [VPNs](../09-security/07-vpn.md).

### Troubleshooting Order

When a static route "does not work", go through this list:

1. Check interfaces: `show ip interface brief`. Is the exit interface up/up?
2. Check the table: `show ip route 192.168.3.10`. Is the route installed? `% Network not in table` or `% Subnet not in table` means it is missing, and no default applies to this query.
3. Check the next hop: `ping 10.0.12.2`. Is it in `show ip arp`?
4. Check the path: `traceroute`. Where does it stop, or does it alternate?
5. Check the return path: go to the far router and look up the source address. Use `ping <dest> source <lan-ip>` to test it.
6. Check that nothing else is blocking: ACLs, firewalls and NAT. See [Common Network Problems and Fixes](../12-troubleshooting-and-tools/09-common-network-problems.md).

## Common Mistakes

- **Forgetting the return route.** Traffic needs a path in both directions. A one-way route looks fine in `show ip route` while every ping fails.
- **Testing with the wrong source.** A router's plain ping uses its exit interface address. It can succeed while real LAN traffic fails. Use `ping <dest> source <lan-ip>`.
- **Mixing up mask and prefix.** `ip route 192.168.3.10 255.255.255.0 10.0.12.2` has host bits set. IOS answers `%Inconsistent address and mask`. Use the network address. A wildcard mask such as 0.0.0.255 is also wrong here.
- **Using your own address as next hop.** IOS answers `%Invalid next hop address (it's this router)`.
- **Assuming a configured route is an active route.** A typed next hop that does not resolve gives no error. The route just never appears in `show ip route`. Compare the running configuration with the table.
- **Thinking a missing floating route is a bug.** A floating route should be absent from the table while the primary works.
- **Choosing the wrong AD for a floating route.** A static with AD 100 backing up OSPF (110) is not a backup. It replaces OSPF at once.
- **Expecting a floating route to catch remote failures.** It reacts only to what the router sees locally, as Step 7 of the lab showed.
- **Building a loop with two defaults.** R1 default to R2 and R2 default to R1 with no other exit means unknown traffic bounces until TTL expires. Traceroute shows the same two addresses repeating.
- **Using an exit interface alone on Ethernet.** This depends on proxy ARP and fills the ARP table.
- **Letting a tracking probe use the backup path.** The route then flaps. Pin the probe to the primary path.
- **Forgetting `ip routing` on a Layer 3 switch.** With routing off, the switch does not route between VLANs and uses `ip default-gateway` for itself. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md).
- **Losing the route after a reboot.** On IOS, save with `copy running-config startup-config`. On Linux, use netplan or nmcli. On Windows, use `route -p` or `New-NetRoute`.
- **Adding a duplicate default on a host.** Linux answers `RTNETLINK answers: File exists` for the same prefix and metric. Windows `route add` answers `The route addition failed: The object already exists.` Use `ip route replace` or `route change`, or choose another metric.
- **Giving Linux an unreachable gateway.** The answer is `Error: Nexthop has invalid gateway.` Older tools say `Network is unreachable`. The gateway must be on a local subnet unless you use `onlink`.
- **Forgetting that Windows adds metrics.** The route metric is added to the interface metric. A route with metric 10 may not be the winner you expected.
- **Believing 0.0.0.0 always means the same thing.** In a route with /0, it means "all addresses." As a source address, it means "this host, before it has an address." In a server's listen address, it means "all local addresses."
- **Forgetting Linux forwarding.** Routes alone do not make a Linux box a router. It also needs `net.ipv4.ip_forward=1`.

## Interview Questions

**Q: What is a static route, and when would you use one?**
A: A static route is a route entered by hand. I use it for stub networks with one exit, for a default route to an ISP, for a backup path, and for discard routes to Null0. It uses no bandwidth or protocol overhead, but it does not adapt to remote failures, so I avoid it in large networks with many paths.

**Q: What does 0.0.0.0/0 mean, and why is it used last?**
A: It is the default route. The /0 means zero bits must match, so every IPv4 address matches. Because routers choose the longest prefix match, a /0 loses to any more specific route, so it is used only when nothing else matches.

**Q: What is the gateway of last resort?**
A: It is the Cisco term for the next hop of the selected default route. `show ip route` prints it near the top. If it says "not set" and no route matches, the router drops the packet and may return ICMP Destination Unreachable.

**Q: Should a static route use a next hop or an exit interface?**
A: On Ethernet, use a next hop, and preferably both. An exit interface alone makes the router ARP for each destination and depend on proxy ARP. On a true point-to-point link, an exit interface alone is fine. The fully specified form is the safest, because the route depends on the interface and cannot resolve through a default route after a failure.

**Q: What is a floating static route?**
A: It is a backup static route with an AD higher than the primary. It stays out of the routing table until the primary is removed. For example, to back up OSPF, which has AD 110, I would set the static to AD 130 on Cisco.

**Q: What is a recursive lookup?**
A: It is when the route's next hop is an IP, so the router must look up that IP to find the exit interface. For example, 192.168.3.0/24 via 10.0.12.2 needs a second lookup that finds 10.0.12.0/30 on G0/1. CEF resolves this ahead of time, so it does not happen per packet in normal forwarding.

**Q: A static route is in the configuration but not in the routing table. Why?**
A: The next hop is probably unreachable, or the exit interface is down, or another route to the same prefix has a lower AD. I would check `show ip interface brief`, then look up the next hop in the table, then check for a better route to that prefix.

**Q: R1 can ping a remote server, but a PC behind R1 cannot. What do you check?**
A: I suspect the return path. The router's ping uses its own exit address, which the far side may know. The far side may not have a route to the PC's subnet. I would test with `ping <server> source <lan-ip>`, then check the routes on the routers at the far end, and also check NAT if the traffic crosses the Internet.

**Q: The next hop router dies, but the link stays up. What happens to a static route?**
A: The route stays, and traffic goes into a black hole. This happens when a switch sits between the routers, because the local link never goes down. I would fix it with IP SLA and object tracking, with BFD where the platform supports it, or with a dynamic protocol.

**Q: What is the difference between `ip default-gateway` and `ip route 0.0.0.0 0.0.0.0`?**
A: `ip default-gateway` is for a device that is not routing, such as a Layer 2 switch, and it serves only the device's own management traffic. `ip route 0.0.0.0 0.0.0.0` is a real default route used when IP routing is on.

**Q: Why would you send a route to Null0?**
A: To drop traffic for addresses that are not in use and to prevent loops. If a router owns a summary and has a default route, traffic for an unused part of the summary would otherwise go out the default and come back. A discard route for the summary drops it, while longer real routes still win.

**Q: How do you add a permanent static route on Linux and Windows?**
A: On Linux, `ip route add` is temporary, so I put the route in netplan or use nmcli. On Windows, I use `route -p add` or `New-NetRoute`, which persists by default. On Windows, the effective metric is the route metric plus the interface metric.

## Quick Recap

- A static route is a manual route. It is simple and has no overhead, but it cannot see remote failures.
- The default route 0.0.0.0/0 (or ::/0 in IPv6) matches everything and is used only when no longer prefix matches.
- A route needs a reachable next hop, or an up interface, before it enters the table. The return path needs its own routes.
- On Ethernet, use a next hop, or both next hop and interface. Do not use an exit interface alone.
- A floating static has a higher AD (a higher metric on Linux and Windows) and appears only when the primary is removed.
- Recursive lookup resolves a next hop to an interface. CEF does this in advance.
- For failures the router cannot see, add IP SLA tracking, BFD, or a dynamic routing protocol.
