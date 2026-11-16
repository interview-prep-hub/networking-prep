# Linux Networking

**Level:** Intermediate | **Prerequisites:** [The Routing Table](../05-routing/02-routing-table.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md), [ARP](../03-ethernet-and-switching/05-arp.md), [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md), [DNS Basics](../07-application-protocols/01-dns-basics.md)

## In One Line

Linux keeps its network state in a few simple tables (interfaces, addresses, routes, neighbours), and with namespaces, veth pairs and bridges you can build a whole virtual network inside one machine, which is exactly how containers work.

## Why It Matters

Almost every server, cloud VM, container and Kubernetes node runs Linux. When a service cannot be reached, you will be logged in to a Linux box with a shell and no diagram. You need to read the machine's own view of the network.

Containers add a second reason. Docker and Kubernetes did not invent new networking. They use Linux features that you can type by hand: a namespace, a veth pair, a bridge, a route, a NAT rule. If you can build that by hand, container networking stops being magic.

Interviewers ask this topic because it separates people who memorised terms from people who can debug. Typical questions:

- Show me how you check the IP, the route and the ARP entry on a Linux box.
- Why does `/etc/resolv.conf` say `127.0.0.53`?
- What is a network namespace? How would you connect two of them?
- Why can a container reach the host but not the internet?

## Simple Explanation

Think of a Linux machine as an office building.

| Linux idea | Office building idea |
|---|---|
| Interface | A door to the outside, or a door to another building |
| IP address | The address written on that door |
| Routing table | The map at reception: "parcels for this city leave by this door" |
| Neighbour table | A list of the people next door: name and face |
| `resolv.conf` | The phone number of the directory service |
| Network namespace | A separate office with its own doors, map and list |
| veth pair | A private tube between two offices. Push at one end, it comes out of the other |
| Bridge | A shared hallway that connects many offices on one floor |
| IP forwarding | Permission for reception to pass along parcels meant for someone else |

**Where the analogy stops being true.** Offices are physically separate. Namespaces are not. All namespaces share one kernel (the core program of the operating system), the same CPU and the same memory. A namespace separates the network view only. It is not a security wall on its own.

## How It Works

### 1. The big picture

The kernel holds all network state. Tools like `ip` only read and change that state. They do not hold any state of their own.

```text
                 Linux host
  +-----------------------------------------+
  |  Applications (curl, nginx, sshd)       |
  |  ------------- sockets ---------------  |
  |  Kernel network stack                   |
  |    interfaces       (ip link, ip addr)  |
  |    routing table    (ip route)          |
  |    neighbour table  (ip neigh)          |
  |    firewall rules   (nftables)          |
  +-------+-------------+-----------+-------+
          |             |           |
          lo          ens3        docker0
       (virtual)   (real card)   (virtual bridge)
```

The `ip` command comes from a package called iproute2. Older tools (`ifconfig`, `route`, `arp`, `netstat`) come from a package called net-tools. That package is deprecated, which means it still exists but you should not rely on it. Many minimal servers do not have it.

| Old tool | Modern tool |
|---|---|
| `ifconfig` | `ip addr`, `ip link` |
| `route -n` | `ip route` |
| `arp -n` | `ip neigh` |
| `netstat -tulpn` | `ss -tulpn` |
| `brctl show` | `bridge link`, `ip link show type bridge` |

Useful shortcuts work with every `ip` command:

```bash
# Linux
ip a                # short for ip addr
ip r                # short for ip route
ip n                # short for ip neigh
ip -br a            # brief: one line per interface
ip -c a             # coloured output
ip -j a             # JSON output, good for scripts
```

Windows has the same ideas under different names. The full comparison is in [IP Configuration Commands](../12-troubleshooting-and-tools/02-ip-configuration-commands.md).

```powershell
# Windows PowerShell
Get-NetAdapter                  # like: ip link
Get-NetIPAddress                # like: ip addr
Get-NetRoute                    # like: ip route
Get-NetNeighbor                 # like: ip neigh
Get-DnsClientServerAddress      # like: reading resolv.conf
```

### 2. Interfaces

An **interface** is the point where the kernel sends and receives packets. Some interfaces are real hardware, such as a NIC (network interface card, the part that plugs in to the cable). Others exist only in software.

Common names:

| Name | What it is |
|---|---|
| `lo` | Loopback. A virtual interface for traffic to the machine itself, `127.0.0.1` |
| `eth0` | Ethernet, older naming style. Still common in VMs and containers |
| `ens3`, `enp0s3` | Ethernet, predictable naming style (see below) |
| `wlan0`, `wlp2s0` | Wi-Fi |
| `docker0`, `br0` | Software bridge |
| `veth1a2b3c` | One end of a veth pair |
| `tun0`, `wg0` | VPN tunnel |
| `bond0` | Several links acting as one, see [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md) |
| `ens3.100` | VLAN 100 on top of `ens3`, see [VLANs](../03-ethernet-and-switching/06-vlans.md) |

**Predictable names.** The old names `eth0`, `eth1` could swap after a reboot if the kernel found the cards in a different order. systemd now builds the name from the hardware location. In `enp0s3`, `en` means Ethernet, `p0` means PCI bus 0, and `s3` means slot 3. The name stays the same across reboots.

Show interfaces with `ip link`. This command shows Layer 2 facts only, with no IP addresses.

```bash
# Linux
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
```

How to read it:

| Field | Meaning |
|---|---|
| `2:` | Interface index, a number the kernel uses to identify it |
| `UP` | An administrator turned it on (`ip link set ens3 up`) |
| `LOWER_UP` | The physical link works: there is a carrier signal |
| `NO-CARRIER` | Turned on, but no signal: cable unplugged or far end down |
| `mtu 1500` | MTU, the largest packet this interface will send, in bytes |
| `qdisc fq_codel` | Queueing discipline: how the kernel orders packets waiting to leave |
| `state UP` | Overall working state |
| `link/ether` | MAC address, the Layer 2 address of the interface |

`UP` and `LOWER_UP` answer different questions. `UP` is your intent. `LOWER_UP` is reality. You need both. The loopback shows `state UNKNOWN`, and that is normal, because it has no carrier to measure.

Common changes:

```bash
# Linux, run as root. These do not survive a reboot.
ip link set ens3 up
ip link set ens3 down         # careful: this cuts your SSH session
ip link set ens3 mtu 9000     # jumbo frames, every device on the path must agree
ip -s link show ens3          # counters: bytes, packets, errors, drops
```

The counters are often the fastest clue on a bad link:

```text
    RX:  bytes packets errors dropped  missed   mcast
     981234567  812345      0       3       0    1204
    TX:  bytes packets errors dropped carrier collsns
      45123987  402118      0       0       0       0
```

### 3. ip addr

`ip addr` shows the Layer 3 addresses on each interface.

```bash
# Linux
$ ip addr show ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 192.168.1.50/24 brd 192.168.1.255 scope global dynamic ens3
       valid_lft 86012sec preferred_lft 86012sec
    inet6 fe80::5054:ff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever
```

How to read it:

| Field | Meaning |
|---|---|
| `inet 192.168.1.50/24` | IPv4 address and prefix length |
| `brd 192.168.1.255` | Broadcast address of this subnet |
| `scope global` | Valid anywhere |
| `scope link` | Valid only on this link, such as the IPv6 `fe80::` address |
| `scope host` | Valid only inside this machine, such as `127.0.0.1` |
| `dynamic` | Learned from DHCP (or IPv6 autoconfiguration), so it has a lifetime |
| `valid_lft` | Seconds until the address expires |

Add and remove addresses like this:

```bash
# Linux, run as root. These do not survive a reboot.
ip addr add 192.168.1.51/24 dev ens3     # one interface can hold many
ip addr del 192.168.1.51/24 dev ens3
ip addr flush dev ens3                   # removes all. Over SSH, you are gone
```

Adding an address with a prefix does a second job. The kernel also adds a route for that subnet. That is why `/24` matters. If you write `192.168.1.51/32`, there is no subnet route, and the machine does not know its neighbours are directly reachable.

**Making it permanent.** `ip` changes live in kernel memory, so they are lost on reboot. Saved settings belong to a network manager, and this differs by distribution:

| Distribution family | Usual tool | Where config lives |
|---|---|---|
| Ubuntu | Netplan, which drives systemd-networkd or NetworkManager | `/etc/netplan/*.yaml` |
| RHEL, Fedora, Rocky | NetworkManager, using `nmcli` | `/etc/NetworkManager/` |
| Debian (classic install) | ifupdown | `/etc/network/interfaces` |

```text
# /etc/netplan/01-static.yaml (Ubuntu)
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: false
      addresses: [192.168.1.50/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1]
```

```bash
# Ubuntu: test with automatic rollback, then apply
netplan try
netplan apply

# RHEL family: same result with NetworkManager
nmcli con mod ens3 ipv4.method manual ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 ipv4.dns 192.168.1.1
nmcli con up ens3
```

### 4. ip route

The routing table answers one question: to reach this destination, which interface do I use, and do I need a gateway? The theory is in [The Routing Table](../05-routing/02-routing-table.md). Here is the Linux view.

```bash
# Linux
$ ip route show
default via 192.168.1.1 dev ens3 proto dhcp src 192.168.1.50 metric 100
10.20.0.0/16 via 192.168.1.254 dev ens3
10.20.30.0/24 via 192.168.1.253 dev ens3
192.168.1.0/24 dev ens3 proto kernel scope link src 192.168.1.50 metric 100
```

| Field | Meaning |
|---|---|
| `default` | The route used when nothing else matches. Same as `0.0.0.0/0` |
| `via 192.168.1.1` | Send to this next-hop router |
| No `via` | The destination is directly on this link |
| `dev ens3` | Leave through this interface |
| `proto kernel` | Added by the kernel when you set an address |
| `proto dhcp` | Added by a DHCP client |
| `proto static` | Added by a config tool |
| `src 192.168.1.50` | Source address to use when this host starts the traffic |
| `metric 100` | Cost. If two routes have the same prefix, the lower metric wins |

Selection works like this:

1. Find every route that matches the destination.
2. Pick the longest prefix (the most specific one).
3. If prefixes tie, pick the lowest metric.

Using the table above:

| Destination | Matching routes | Winner | Next hop |
|---|---|---|---|
| `10.20.30.40` | default, /16, /24 | `10.20.30.0/24` | `192.168.1.253` |
| `10.20.99.1` | default, /16 | `10.20.0.0/16` | `192.168.1.254` |
| `192.168.1.20` | default, /24 | `192.168.1.0/24` | Direct |
| `198.51.100.7` | default | default | `192.168.1.1` |

You do not need to work this out by hand. Ask the kernel:

```bash
# Linux
$ ip route get 10.20.30.40
10.20.30.40 via 192.168.1.253 dev ens3 src 192.168.1.50 uid 1000
    cache
```

This is the single best routing command. It shows the real decision, including the source address.

Changes:

```bash
# Linux, run as root. These do not survive a reboot.
ip route add 10.30.0.0/16 via 192.168.1.254 dev ens3
ip route replace default via 192.168.1.1 dev ens3   # add, or update if present
ip route del 10.30.0.0/16
ip route add blackhole 10.99.0.0/16                 # silently drop this traffic
```

### 5. ip neigh

The neighbour table maps an IP address to a MAC address, for machines on the same link. IPv4 fills it with [ARP](../03-ethernet-and-switching/05-arp.md). IPv6 fills it with Neighbor Discovery. The same command shows both.

```bash
# Linux
$ ip neigh show
192.168.1.1 dev ens3 lladdr 52:54:00:aa:bb:01 REACHABLE
192.168.1.20 dev ens3 lladdr 52:54:00:aa:bb:14 STALE
192.168.1.99 dev ens3 FAILED
fe80::1 dev ens3 lladdr 52:54:00:aa:bb:01 router REACHABLE
```

`lladdr` means link-layer address, which is the MAC address.

| State | Meaning |
|---|---|
| `REACHABLE` | Confirmed recently |
| `STALE` | Not confirmed recently. Still used. The kernel rechecks on next use |
| `DELAY` | Used while stale. Waiting briefly for proof from upper layers |
| `PROBE` | Actively asking again |
| `INCOMPLETE` | Asked, no answer yet |
| `FAILED` | Asked, no answer at all. The neighbour is not there |
| `PERMANENT` | Added by hand. Never expires |

`STALE` is normal and healthy. `FAILED` is the one that means trouble.

A key rule: to reach an off-subnet address, the host looks up the MAC of the **gateway**, not of the far destination. So when you ping `198.51.100.7`, the entry you should look for is `192.168.1.1`.

```bash
# Linux, run as root
ip neigh flush dev ens3      # clear entries on this interface
ip neigh add 192.168.1.30 lladdr 52:54:00:aa:bb:1e dev ens3 nud permanent
```

### 6. How the tables work together

Here is what happens when you run `ping 198.51.100.7` on the host above.

1. The application asks to send a packet to `198.51.100.7`.
2. The kernel checks the routing table. The default route matches: `via 192.168.1.1 dev ens3`.
3. The kernel picks the source address: `192.168.1.50`.
4. The kernel checks the neighbour table for `192.168.1.1`, the next hop.
5. If there is no entry, it sends an ARP broadcast out of `ens3` and waits.
6. It builds an Ethernet frame. The destination MAC is the gateway's MAC. The destination IP is still `198.51.100.7`.
7. The frame goes through the queue and out of the interface.

```text
 dest IP --> [ ip route ] --> next hop + interface
                                    |
                                    v
                             [ ip neigh ] --> next hop MAC
                                    |
                                    v
                       frame leaves through interface
```

Each table failure gives a different symptom. See the table at the end of the Example section.

### 7. resolv.conf and systemd-resolved

Names are not handled by the kernel. They are handled in user space, which means by normal programs and libraries outside the kernel. There is no "DNS table" in `ip`.

**The path of a lookup.** An application calls a library function, usually `getaddrinfo()`. The C library then reads `/etc/nsswitch.conf`, which lists where to look and in what order.

```text
 hosts: files dns        <- a common line in /etc/nsswitch.conf
```

That line means: try `/etc/hosts` first, then DNS. Real lines on Ubuntu and Fedora are longer, and they differ by distribution.

```text
 curl example.com
    |
    | getaddrinfo()
    v
 C library  reads /etc/nsswitch.conf
    |
    +-- files --> /etc/hosts
    |
    +-- dns ----> /etc/resolv.conf says: nameserver 127.0.0.53
                       |
                       v
                  systemd-resolved (cache, per-interface DNS)
                       |
                       v
                  upstream DNS server 192.168.1.1
```

**`/etc/resolv.conf`** is a small text file read by the stub resolver. A stub resolver is a simple client that asks a DNS server for an answer and does not walk the DNS tree itself.

```text
# /etc/resolv.conf
nameserver 192.168.1.1
nameserver 192.168.1.2
search corp.example.com
options ndots:1 timeout:5 attempts:2
```

| Line | Meaning |
|---|---|
| `nameserver` | DNS server to ask. Up to 3 are used, tried in order |
| `search` | Domains to try for short names. `web` is tried as `web.corp.example.com` |
| `ndots:1` | A name with at least this many dots is tried as-is first |
| `timeout:5` | Seconds to wait for each server. 5 is the default |
| `attempts:2` | Rounds through the server list. 2 is the default |

The second `nameserver` is a fallback for a server that does not answer. It is not load balancing, and it is not used when the first server says "no such name".

**systemd-resolved** is a local service, used by default on Ubuntu and Fedora. RHEL-family systems usually do not run it, and NetworkManager writes `/etc/resolv.conf` directly there. It does three things:

1. It caches answers.
2. It keeps a separate DNS server list per interface. This helps with VPNs: `corp.example.com` goes to the VPN's DNS, and everything else goes to your normal DNS. This is called split DNS.
3. It runs a small DNS listener on `127.0.0.53` port 53, called the stub listener.

That is why you see this:

```bash
# Linux (Ubuntu)
$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search .

$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Mar 10 09:14 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

`127.0.0.53` is not a mistake. It is the local cache. To see the real DNS servers, ask `resolvectl`:

```bash
# Linux
$ resolvectl status
Global
         Protocols: +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (ens3)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.1.1
       DNS Servers: 192.168.1.1
        DNS Domain: corp.example.com
```

Useful commands:

```bash
# Linux
resolvectl query example.com     # resolve through systemd-resolved
resolvectl statistics            # cache hits and misses
resolvectl flush-caches          # clear the cache
resolvectl dns ens3 192.168.1.2  # temporary: set DNS for one interface
getent hosts example.com         # resolve the same way applications do
```

The four things `/etc/resolv.conf` can be on such a system:

| What `/etc/resolv.conf` is | Effect |
|---|---|
| Link to `/run/systemd/resolve/stub-resolv.conf` | Normal mode. Apps use `127.0.0.53` |
| Link to `/usr/lib/systemd/resolv.conf` | Static file pointing at `127.0.0.53` |
| Link to `/run/systemd/resolve/resolv.conf` | Lists the real upstream servers, skips the local stub |
| A plain file | Managed by something else. resolved leaves it alone |

To change DNS for good, change the source: Netplan, `nmcli`, or `/etc/systemd/resolved.conf`. If you edit the linked file by hand, your change gets overwritten.

For more on DNS itself, see [DNS Basics](../07-application-protocols/01-dns-basics.md) and [DNS Tools](../12-troubleshooting-and-tools/04-dns-tools.md).

### 8. Network namespaces

A **network namespace** is a separate copy of the network stack. Each namespace has its own:

- interfaces, including its own `lo`
- IP addresses
- routing table
- neighbour table
- firewall rules
- port numbers, so two namespaces can both listen on port 80
- most `net.*` sysctl settings

A new namespace starts almost empty. It has only `lo`, and `lo` is down.

The first namespace, which exists at boot, is called the **root namespace** or host namespace. Every interface you normally see is there.

```bash
# Linux, run as root
ip netns add red                 # creates /run/netns/red
ip netns list
ip netns exec red ip addr        # run any command inside red
ip -n red addr                   # shorter form for ip commands only
ip netns exec red bash           # open a shell inside. Type exit to leave
ip netns del red
```

```bash
$ ip netns exec red ip -br link
lo               DOWN           00:00:00:00:00:00 <LOOPBACK>
```

A process lives in exactly one network namespace. Moving one real NIC into a namespace is also possible (`ip link set ens4 netns red`), and then the host can no longer see it. An interface also belongs to exactly one namespace at a time.

**Fuller truth on deletion.** `ip netns del` removes the name. The namespace itself lives until the last process inside it exits. When it finally goes away, real NICs return to the root namespace, and virtual interfaces such as veth are destroyed.

### 9. veth pairs

A new namespace is cut off from everything. A **veth pair** (virtual Ethernet) connects it to the world. It is two virtual interfaces joined like a cable. A frame sent into one end comes out of the other end.

**Analogy:** a network cable with a plug at each end. **Where it stops being true:** a real cable has a length and a speed limit, and a veth has neither. It is a memory copy inside the kernel, so the limit is CPU. You also cannot cut one end off. Delete either end and both disappear.

```bash
# Linux, run as root
ip link add veth-red type veth peer name veth-blue   # create both ends
ip link set veth-red netns red                       # move one end
ip link set veth-blue netns blue                     # move the other end
```

Things to remember:

- Both ends must be up, or the link shows no carrier.
- Interface names are limited to 15 characters.
- In `ip link`, a name like `veth-red@if6` means "my other end has index 6". The other end may be in a different namespace, where that index means something else.

### 10. Linux bridges

A veth connects exactly two things. For three or more, you need a switch. A **Linux bridge** is a software [switch](../03-ethernet-and-switching/03-how-switches-work.md) inside the kernel.

It behaves like a real switch:

1. It learns which MAC address is behind which port.
2. It forwards a frame to the right port if it knows the MAC.
3. It floods the frame to all other ports if it does not.
4. It forgets entries after an ageing time, 300 seconds by default.

**Analogy:** a hallway connecting several rooms. **Where it stops being true:** a hallway has no memory, while a bridge remembers who is in which room. Also, a bridge can have its own IP address. That makes the host one more member of the LAN, as if the hallway itself were a person you could talk to.

```bash
# Linux, run as root
ip link add br0 type bridge
ip link set br0 up
ip link set veth-r-br master br0         # plug a veth end into the bridge
bridge link show                         # list ports
bridge fdb show br br0                   # the MAC table
```

`fdb` means forwarding database, which is the bridge's MAC table.

An interface that is plugged in to a bridge is now just a port. It no longer does IP work. If you add a real NIC such as `ens3` to a bridge, its IP address must move to the bridge, or the host loses connectivity. This is the classic way to break a remote server.

### 11. sysctl basics

**sysctl** is the tool for reading and changing kernel settings while the system runs. Each setting is also a file under `/proc/sys`. The dots in the name become slashes in the path.

```text
 net.ipv4.ip_forward   <->   /proc/sys/net/ipv4/ip_forward
```

```bash
# Linux
sysctl net.ipv4.ip_forward                 # read one setting
cat /proc/sys/net/ipv4/ip_forward          # same thing
sysctl -a | grep rp_filter                 # search all settings
sudo sysctl -w net.ipv4.ip_forward=1       # write now, lost at reboot
```

To keep a setting across reboots, put it in a file and reload:

```bash
# Linux, run as root
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/90-forwarding.conf
sysctl --system       # reload all sysctl config files
```

Settings that interviewers expect you to know:

| Setting | Purpose | Usual default |
|---|---|---|
| `net.ipv4.ip_forward` | Route IPv4 packets for others | 0 |
| `net.ipv6.conf.all.forwarding` | Route IPv6 packets for others | 0 |
| `net.ipv4.conf.all.rp_filter` | Drop packets that arrive on the wrong side | Kernel 0, many distributions change it |
| `net.ipv4.ip_local_port_range` | Client source port range | 32768 60999 |
| `net.core.somaxconn` | Limit on a socket's accept queue | 4096 on kernel 5.4 and later, 128 before |
| `net.ipv4.tcp_syncookies` | Defence against SYN floods | 1 on most distributions |
| `net.ipv4.tcp_fin_timeout` | Seconds in the FIN-WAIT-2 state | 60 |
| `net.ipv4.tcp_congestion_control` | TCP congestion algorithm | cubic on most distributions |
| `net.ipv4.neigh.default.gc_thresh3` | Hard limit on the neighbour table | 1024 |
| `net.ipv4.ip_unprivileged_port_start` | First port a normal user can bind | 1024 |

Defaults vary by kernel and distribution, so always read the live value.

### 12. IP forwarding

A normal host has a strict rule: if a packet arrives and the destination IP is not mine, drop it. **IP forwarding** turns that rule off. With forwarding on, the kernel checks the routing table and sends the packet onward. That is what makes a machine a router.

```text
 packet arrives, dest IP is not mine
          |
          v
   ip_forward = 0 ?  --yes-->  drop it, no error sent
          |
          no
          v
   look up route, reduce TTL by 1, send it out
```

**Analogy:** a receptionist. With forwarding off, a parcel for someone else goes in the bin. With forwarding on, the receptionist reads the map and sends it onward. **Where it stops being true:** a router does not keep a record of each parcel, and it does not hand back a receipt. Forwarding is per packet.

Docker and Kubernetes nodes need this on, because container traffic passes through the host as forwarded traffic. VPN gateways and NAT boxes need it too.

Forwarding alone does not give internet access. The private source address must be rewritten too, using [NAT](../04-ip-addressing/09-nat-and-pat.md), and the firewall must allow the packets through. See [iptables and nftables](02-iptables-and-nftables.md).

## Example

Build a network by hand, in four labs. Each lab adds one idea. You need a Linux machine or VM with `iproute2`. Nothing here needs a real second machine.

Run everything as root (`sudo -i`). Use a test VM, not a production server. Commands are for Linux.

### Lab 1: two namespaces and one cable

The goal is the smallest possible network: two namespaces joined by a veth pair.

```text
  namespace red                      namespace blue
 +------------------+               +------------------+
 | veth-red         |   veth pair   |        veth-blue |
 | 10.0.0.1/24      o===============o      10.0.0.2/24 |
 +------------------+               +------------------+
```

```bash
# 1. Create the two namespaces
ip netns add red
ip netns add blue

# 2. Create the cable, then put one end in each namespace
ip link add veth-red type veth peer name veth-blue
ip link set veth-red netns red
ip link set veth-blue netns blue

# 3. Give each end an address
ip -n red addr add 10.0.0.1/24 dev veth-red
ip -n blue addr add 10.0.0.2/24 dev veth-blue

# 4. Bring everything up, including loopback
ip -n red link set lo up
ip -n blue link set lo up
ip -n red link set veth-red up
ip -n blue link set veth-blue up

# 5. Test
ip netns exec red ping -c 2 10.0.0.2
```

```text
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.062 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.041 ms

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1023ms
rtt min/avg/max/mdev = 0.041/0.051/0.062/0.010 ms
```

Now look at what each namespace knows:

```bash
$ ip -br -n red link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
veth-red@if6     UP             6a:1f:3c:9d:2e:70 <BROADCAST,MULTICAST,UP,LOWER_UP>

$ ip -n red route
10.0.0.0/24 dev veth-red proto kernel scope link src 10.0.0.1

$ ip -n red neigh
10.0.0.2 dev veth-red lladdr 6a:1f:3c:9d:2e:71 REACHABLE

$ ip netns exec red ping -c 1 198.51.100.7
ping: connect: Network is unreachable
```

Notes on that output:

- The route appeared by itself when we added the `/24` address.
- The `ping` output shows TTL 64 with no reduction, so no router was involved.
- The last ping fails at once because `red` has no route to that address.
- Run `ip link` on the host. Neither veth end is there. They are hidden inside the namespaces.

Clean up. Deleting the namespaces also destroys the veth pair.

```bash
ip netns del red
ip netns del blue
```

### Lab 2: two namespaces on a bridge

A cable only joins two. Now use a bridge, as Docker does. The bridge also gives the host an address on this network, and that address becomes the namespaces' gateway.

```text
  namespace red              namespace blue
 +---------------+          +---------------+
 | veth-r        |          | veth-b        |
 | 10.200.0.2/24 |          | 10.200.0.3/24 |
 +-------o-------+          +-------o-------+
         |                          |
 ........|..........................|........ host
         o veth-r-br                o veth-b-br
         |                          |
     +---+--------------------------+---+
     |        br0  10.200.0.1/24        |
     +-----------------+----------------+
                       |
                 host routing
                       |
                 ens3 192.168.1.50
```

```bash
# 1. Namespaces
ip netns add red
ip netns add blue

# 2. The bridge, with the gateway address
ip link add br0 type bridge
ip addr add 10.200.0.1/24 dev br0
ip link set br0 up

# 3. Two cables. One end goes in a namespace, one end in the bridge
ip link add veth-r type veth peer name veth-r-br
ip link add veth-b type veth peer name veth-b-br
ip link set veth-r netns red
ip link set veth-b netns blue
ip link set veth-r-br master br0
ip link set veth-b-br master br0
ip link set veth-r-br up
ip link set veth-b-br up

# 4. Inside the namespaces
ip -n red addr add 10.200.0.2/24 dev veth-r
ip -n blue addr add 10.200.0.3/24 dev veth-b
ip -n red link set lo up
ip -n blue link set lo up
ip -n red link set veth-r up
ip -n blue link set veth-b up
ip -n red route add default via 10.200.0.1
ip -n blue route add default via 10.200.0.1

# 5. Test red to blue, and host to red
ip netns exec red ping -c 2 10.200.0.3
ping -c 2 10.200.0.2
```

Watch it from a second terminal:

```bash
$ tcpdump -ni br0 'arp or icmp'
09:41:07.112201 ARP, Request who-has 10.200.0.3 tell 10.200.0.2, length 28
09:41:07.112233 ARP, Reply 10.200.0.3 is-at 6a:1f:3c:9d:2e:03, length 28
09:41:07.112240 IP 10.200.0.2 > 10.200.0.3: ICMP echo request, id 4321, seq 1, length 64
09:41:07.112256 IP 10.200.0.3 > 10.200.0.2: ICMP echo reply, id 4321, seq 1, length 64
```

Then see what the bridge learned. The `grep` hides permanent entries, such as the bridge's own ports.

```bash
$ bridge fdb show br br0 | grep -v permanent
6a:1f:3c:9d:2e:02 dev veth-r-br master br0
6a:1f:3c:9d:2e:03 dev veth-b-br master br0
```

What happened, step by step:

1. `red` checks its routes. `10.200.0.3` is on the local subnet, so no gateway is needed.
2. `red` has no MAC for it, so it sends an ARP broadcast from `veth-r`.
3. The frame comes out of `veth-r-br` in to the bridge. The bridge learns that `...:02` is behind `veth-r-br`.
4. A broadcast goes to all other ports, so it reaches `veth-b-br`, then `veth-b` in `blue`.
5. `blue` replies. The bridge learns that `...:03` is behind `veth-b-br`, and sends the reply only to `veth-r-br`.
6. `red` now sends the ICMP echo request directly to that MAC.

Red and blue are on the same subnet, so this is pure Layer 2 switching. `ip_forward` is not used at all.

**If the ping fails on a Docker host.** Docker loads the `br_netfilter` module, which makes bridged frames pass through the iptables FORWARD rules. Docker also sets the FORWARD policy to DROP. So traffic inside your own bridge is dropped. Allow it:

```bash
iptables -I FORWARD -i br0 -o br0 -j ACCEPT
```

### Lab 3: reach the outside world

Keep Lab 2 running. From `red`, the outside fails, because the host will not forward packets and the source address `10.200.0.2` is private.

Assume the host's uplink is `ens3` with `192.168.1.50`. Write down the current value of `ip_forward`, so you can restore it later.

```bash
# 1. Note the old value, then allow the host to route
sysctl net.ipv4.ip_forward
sysctl -w net.ipv4.ip_forward=1

# 2. Rewrite the private source address when leaving the host
iptables -t nat -A POSTROUTING -s 10.200.0.0/24 ! -o br0 -j MASQUERADE

# 3. Allow forwarding through the firewall, if it filters
iptables -I FORWARD -i br0 -j ACCEPT
iptables -I FORWARD -o br0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# 4. Test with a well-known public address
ip netns exec red ping -c 2 1.1.1.1
```

```text
64 bytes from 1.1.1.1: icmp_seq=1 ttl=55 time=12.4 ms
```

On the uplink, the source address has been rewritten to the host's address:

```bash
$ tcpdump -ni ens3 icmp
09:47:15.300412 IP 192.168.1.50 > 1.1.1.1: ICMP echo request, id 5120, seq 1, length 64
09:47:15.312801 IP 1.1.1.1 > 192.168.1.50: ICMP echo reply, id 5120, seq 1, length 64
```

Now try a name. This may fail on Ubuntu:

```bash
$ ip netns exec red ping -c 1 example.com
ping: example.com: Temporary failure in name resolution
```

The namespace reads the same `/etc/resolv.conf`, which says `127.0.0.53`. But inside `red`, that address is `red`'s own loopback, and nothing is listening there. `ip netns exec` has a fix: if a file exists at `/etc/netns/<name>/resolv.conf`, it is shown to the namespace in place of the real file.

```bash
mkdir -p /etc/netns/red
echo 'nameserver 1.1.1.1' > /etc/netns/red/resolv.conf
ip netns exec red getent hosts example.com
```

This is the same problem Docker solves. Docker detects the local stub and gives containers the real upstream servers instead.

Clean up:

```bash
iptables -D FORWARD -o br0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -D FORWARD -i br0 -j ACCEPT
iptables -D FORWARD -i br0 -o br0 -j ACCEPT   # only if you added it in Lab 2
iptables -t nat -D POSTROUTING -s 10.200.0.0/24 ! -o br0 -j MASQUERADE
ip netns del red
ip netns del blue
ip link del br0
rm -rf /etc/netns/red
sysctl -w net.ipv4.ip_forward=0   # only if it was 0 before. Not on a Docker host
```

### Lab 4: a router namespace between two subnets

This lab shows IP forwarding on its own, with no NAT. A third namespace acts as a router between two subnets.

```text
 +-----------+      +---------------------+      +-----------+
 | red       |      | router              |      | blue      |
 | eth0      |      | to-red     to-blue  |      | eth0      |
 | 10.1.0.2  o======o 10.1.0.1   10.2.0.1 o======o 10.2.0.2  |
 +-----------+      +---------------------+      +-----------+
      subnet 10.1.0.0/24        subnet 10.2.0.0/24
```

```bash
ip netns add red
ip netns add blue
ip netns add router

# Create each cable directly in the right namespaces.
# Both red and blue can have an interface called eth0: names are per namespace.
ip link add eth0 netns red type veth peer name to-red netns router
ip link add eth0 netns blue type veth peer name to-blue netns router

ip -n red addr add 10.1.0.2/24 dev eth0
ip -n blue addr add 10.2.0.2/24 dev eth0
ip -n router addr add 10.1.0.1/24 dev to-red
ip -n router addr add 10.2.0.1/24 dev to-blue

for ns in red blue router; do ip -n $ns link set lo up; done
ip -n red link set eth0 up
ip -n blue link set eth0 up
ip -n router link set to-red up
ip -n router link set to-blue up

ip -n red route add default via 10.1.0.1
ip -n blue route add default via 10.2.0.1

# Start with forwarding off, so the test is fair
ip netns exec router sysctl -w net.ipv4.ip_forward=0
```

The first test crosses the router and fails silently:

```bash
$ ip netns exec red ping -c 2 -W 1 10.2.0.2
PING 10.2.0.2 (10.2.0.2) 56(84) bytes of data.

--- 10.2.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1032ms
```

`red` can still ping `10.1.0.1`, because that address belongs to the router. Only transit traffic is dropped. Now turn forwarding on, only in the router namespace:

```bash
$ ip netns exec router sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

$ ip netns exec red ping -c 2 10.2.0.2
64 bytes from 10.2.0.2: icmp_seq=1 ttl=63 time=0.074 ms
64 bytes from 10.2.0.2: icmp_seq=2 ttl=63 time=0.053 ms

$ ip netns exec red traceroute -n 10.2.0.2
traceroute to 10.2.0.2 (10.2.0.2), 30 hops max, 60 byte packets
 1  10.1.0.1  0.071 ms  0.021 ms  0.018 ms
 2  10.2.0.2  0.052 ms  0.030 ms  0.027 ms
```

TTL is 63 now, not 64, because the router reduced it by one. The host's own `ip_forward` value did not change, because this setting is per namespace.

Clean up:

```bash
ip netns del red
ip netns del blue
ip netns del router
```

### Reading the symptoms

Use the same order every time: link, address, route, neighbour, then capture.

| Symptom | Likely cause | Check |
|---|---|---|
| `Network is unreachable` | No matching route, or `lo` is down | `ip route` |
| `Destination Host Unreachable` from your own IP | ARP failed, target not on the link | `ip neigh` shows `FAILED` |
| `100% packet loss`, no error | Packet left, no reply came back | `tcpdump` at each hop |
| `Name or service not known`, `Temporary failure in name resolution` | DNS problem, while IP works | `ping` by IP, `resolvectl status` |
| Interface shows `NO-CARRIER` | Far end down, or the other veth end is down | `ip link` on both ends |
| Works from host, fails from namespace | Missing default route, forwarding or NAT | Labs 2 and 3 |

## Going Deeper

### The path of a packet inside the kernel

Netfilter is the kernel framework that firewall rules attach to. It has five hooks, or checkpoints. Both iptables and nftables use them. Details are in [iptables and nftables](02-iptables-and-nftables.md).

```text
 NIC --> [PREROUTING] --> routing decision
                            |            |
                       for this host   for someone else
                            |            |
                         [INPUT]      ip_forward on?
                            |          no: drop
                            v          yes:
                        local app        |
                            |         [FORWARD]
                         [OUTPUT]        |
                            |            |
                            v            v
                          [POSTROUTING] --> NIC
```

This shows why DNAT (rewriting the destination) happens in PREROUTING, before the routing decision, and why SNAT (rewriting the source) happens in POSTROUTING, after it.

On the way in, the path starts earlier. The NIC writes the frame in to a memory ring by DMA (direct memory access, meaning the card writes without the CPU). It raises an interrupt, and the kernel then polls the ring in a low-priority context called a softirq, using a design called NAPI. If the system cannot keep up, the ring overflows and packets are lost before any firewall or `tcpdump` sees them. Check `ip -s link`, `ethtool -S ens3` and `/proc/net/softnet_stat`.

### Policy routing

The `ip route` output shows one table, but Linux has several. **Policy routing** picks a table based on more than the destination, such as the source address. Rules are checked in priority order.

```bash
# Linux
$ ip rule show
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

The `local` table holds this machine's own addresses and is checked first. See it with `ip route show table local`. The `main` table is the one `ip route` shows.

A common use is a host with two uplinks. Replies must leave through the uplink the request came from:

```bash
# Linux, run as root
ip route add default via 192.168.2.1 dev ens4 table 100
ip rule add from 192.168.2.50 lookup 100
```

### rp_filter and asymmetric routing

**Reverse path filtering** checks each incoming packet: "if I replied to this source, would I use this same interface?" In strict mode (1), a packet that fails is dropped silently. Loose mode (2) only needs a route to the source through any interface. 0 turns the check off.

This stops some spoofed traffic. It also breaks asymmetric routing, where traffic goes in by one path and comes back by another. If a multi-homed host loses packets that `tcpdump` can see arriving, check this setting. The value used is the higher of `conf.all.rp_filter` and the interface's own value, so setting only one of them may do nothing.

Set `net.ipv4.conf.all.log_martians=1` to log such dropped packets. A "martian" is a packet with an impossible source or destination address.

### all, default and per-interface settings

Many settings exist in three forms:

```text
 net.ipv4.conf.all.rp_filter        all interfaces (rules vary, see below)
 net.ipv4.conf.default.rp_filter    interfaces created later
 net.ipv4.conf.ens3.rp_filter       this interface only
```

`all` does not simply overwrite the others. How it combines with the interface value depends on the setting. For some it is the maximum, for some a logical OR or AND. The kernel document `ip-sysctl` gives the rule for each.

Forwarding has its own details:

- `net.ipv4.conf.<if>.forwarding` controls whether packets **received** on that interface can be forwarded. The check is on the way in, not the way out.
- Writing `net.ipv4.ip_forward` changes this setting on all interfaces, and it resets host and router parameters to their defaults. So set it first, then fine-tune.
- IPv6 differs. Turning on `net.ipv6.conf.all.forwarding` also changes host behaviour: the system stops accepting Router Advertisements unless `accept_ra` is set to 2. A server that loses its IPv6 default route after you enable forwarding has hit this.

A new namespace inherits IPv4 settings from the root namespace at creation, and it resets IPv6 settings to defaults. This is the default behaviour, controlled by `net.core.devconf_inherit_init_net`. That is why Lab 4 sets `ip_forward=0` first: on a Docker host, a new namespace may start with 1.

### Weak host model and ARP flux

Linux uses the **weak host model**: an IP address belongs to the host, not to one interface. So by default, Linux will answer an ARP request for any of its local addresses on any interface. Two NICs on the same subnet can then give different answers, which is called ARP flux.

Two settings control this:

| Setting | Use |
|---|---|
| `arp_ignore=1` | Answer ARP only if the target IP is on the interface that received it |
| `arp_announce=2` | Use the best local address as the source of ARP requests |

These are also needed for direct-server-return [load balancing](../10-enterprise-and-datacenter/04-load-balancing.md), where backends hold the virtual IP but must not answer ARP for it.

### Other virtual interface types

| Type | Layer | Use |
|---|---|---|
| `dummy` | 3 | A stable local address with no real link |
| `vlan` | 2 | 802.1Q tagged sub-interface |
| `bond` | 2 | Combine NICs for speed or failover |
| `macvlan` | 2 | Many MAC addresses on one NIC, no bridge needed |
| `ipvlan` | 2 or 3 | Many IPs that share one MAC, for places that limit MACs |
| `tun` | 3 | User-space program sends and receives IP packets (VPNs) |
| `tap` | 2 | User-space program sends and receives Ethernet frames (VMs) |
| `vxlan` | Overlay | Layer 2 over UDP, see [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) |
| `wireguard` | 3 | Encrypted tunnel, see [VPNs](../09-security/07-vpn.md) |
| `vrf` | 3 | Separate routing table domain |

```bash
# Linux, run as root: VLAN 100 sub-interface
ip link add link ens3 name ens3.100 type vlan id 100
ip addr add 10.100.0.5/24 dev ens3.100
ip link set ens3.100 up
```

### Namespace or VRF?

Both give separate routing. They are not the same.

| | Network namespace | VRF |
|---|---|---|
| Separates | Whole stack: interfaces, sockets, firewall, sysctl | Routing tables mainly |
| Typical use | Containers | Multi-tenant routers |
| Same listening port twice | Yes | Depends on socket setup |
| Host tools see it | Must enter the namespace | Visible in the root namespace |

### Bridge details

- STP (see [Spanning Tree](../03-ethernet-and-switching/09-spanning-tree.md)) is off by default on a bridge made with `ip link`. Turn it on with `ip link set br0 type bridge stp_state 1`. A loop with STP off will melt traffic, as on a real switch.
- With `vlan_filtering 1`, the bridge becomes a VLAN-aware switch. Manage it with `bridge vlan`.
- `ip -d link show br0` shows bridge details. `ageing_time 30000` is in hundredths of a second, so it means 300 seconds.
- With `br_netfilter` loaded and `net.bridge.bridge-nf-call-iptables=1`, bridged frames pass through iptables. [Kubernetes](08-kubernetes-networking.md) setups normally need this, so Service rules apply to traffic between pods on one node.

### How this maps to containers

```text
 Lab step                       Docker equivalent
 ------------------------------ -----------------------------------
 ip netns add red               One network namespace per container
 ip link add ... type veth      One veth pair per container
 br0 with 10.200.0.1/24         docker0 with 172.17.0.1/16 (default)
 ip route add default           Container default route via docker0
 sysctl ip_forward=1            Docker enables it at start
 MASQUERADE rule                Added by Docker
```

More in [Docker Networking](07-docker-networking.md).

`ip netns list` does not show Docker's namespaces, because Docker does not create the names under `/run/netns`. Enter one through a process instead. This keeps the host's tools and uses the container's network:

```bash
# Linux, run as root
PID=$(docker inspect -f '{{.State.Pid}}' web)
nsenter -t "$PID" -n ip addr
nsenter -t "$PID" -n ss -tln
```

To find which host veth belongs to a container, read the peer's index inside the container, then look it up on the host:

```bash
$ docker exec web cat /sys/class/net/eth0/iflink
14
$ ip -o link | grep '^14:'
14: veth3f2a1b@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
```

## Common Mistakes

- **Forgetting that `ip` changes are temporary.** Everything set with `ip addr`, `ip route` or `sysctl -w` is lost on reboot. Save it in Netplan, `nmcli` or `/etc/sysctl.d/`.
- **Forgetting to bring up an interface.** A new veth is down, and in a new namespace `lo` is down too. `ip addr add` does not bring an interface up.
- **Bringing up only one end of a veth.** Both ends must be up, or the link has no carrier.
- **Thinking `STALE` means broken.** It means "not confirmed recently". The entry still works. `FAILED` is the bad state.
- **Looking for the remote server's MAC in `ip neigh`.** For off-subnet traffic, you only ever learn the gateway's MAC.
- **Editing `/etc/resolv.conf` by hand on a managed system.** The change disappears at the next reconnect or reboot. Change the source of the setting.
- **Treating `127.0.0.53` as a bug.** It is the local systemd-resolved listener. Use `resolvectl status` to see the real servers.
- **Testing DNS only with `dig`.** `dig` reads `/etc/resolv.conf` directly and ignores `/etc/hosts` and `nsswitch.conf`. Applications do not. Use `getent hosts name` to see what an application gets.
- **Thinking a second `nameserver` is tried on "not found".** It is tried only when the first one does not answer. Two servers with different private zones will give random-looking failures.
- **Enabling `ip_forward` and expecting internet access.** Forwarding moves the packet. Private sources still need NAT, and the firewall must allow it.
- **Enabling `ip_forward` for two namespaces on the same bridge.** Same subnet means Layer 2 switching. Forwarding is not involved.
- **Adding a real NIC to a bridge over SSH without moving the IP.** The NIC stops doing IP work and the session dies. Do it from a console, or in one scripted step.
- **Gateway outside the subnet.** `ip route add default via 10.9.9.9` fails with `Error: Nexthop has invalid gateway.` when no route says that address is reachable on the link.
- **Thinking a namespace is a security boundary.** It separates network state only. A process running as root in it may still reach the host through other paths. Containers also use more namespaces, capabilities and seccomp.
- **Using names longer than 15 characters.** `ip link add` rejects them. Container tools use short random veth names for this reason.

## Interview Questions

**Q: How do you check the network setup of a Linux box from scratch?**

A: I go bottom-up. `ip -br link` for link state, `ip -br addr` for addresses, `ip route` for the default route, and `ip route get <target>` for the real decision. Then `ip neigh` to see if the gateway's MAC resolved, `resolvectl status` for DNS, and `ss -tulpn` for listening ports. If it still does not make sense, I capture with `tcpdump`.

**Q: What is a network namespace?**

A: It is a separate copy of the kernel's network stack: its own interfaces, addresses, routes, neighbour table, firewall rules and port space. Processes inside see only that copy. Containers use one per container, which is why each can listen on port 80. It separates networking only, and all namespaces share one kernel.

**Q: What is a veth pair, and why do containers need one?**

A: It is two virtual interfaces joined like a cable. A frame sent in one end comes out of the other. A namespace starts cut off, so you put one end inside it and the other end on the host, usually in a bridge. That is the container's link to the outside.

**Q: How would you connect two namespaces? And ten?**

A: For two, one veth pair with an end in each, addresses in one subnet, and both ends up. For ten, I create a Linux bridge and give each namespace a veth pair with the host end in the bridge. The bridge learns MACs and switches frames. If I give the bridge an IP, it can be their default gateway.

**Q: Two namespaces share a bridge and one subnet. Do you need ip_forward?**

A: No. They talk at Layer 2 through the bridge, so the kernel switches frames and never routes them. `ip_forward` matters only when traffic crosses subnets through the host's IP layer, such as going from a namespace out through `ens3`.

**Q: What does ip_forward do, and how do you make it permanent?**

A: By default a host drops packets not addressed to it. With `net.ipv4.ip_forward=1`, it routes them using its routing table, so it acts as a router. I set it with `sysctl -w` for now, and put it in a file under `/etc/sysctl.d/` and run `sysctl --system` for persistence. It is per namespace.

**Q: A container can reach its host but not the internet. What do you check?**

A: First, does the container have a default route through the bridge? Second, is `ip_forward` 1 on the host? Third, is there a MASQUERADE rule for the container subnet? Fourth, does the FORWARD chain allow it? I would run `tcpdump` on the bridge and on the uplink. If the packet appears on the uplink with the private source, NAT is missing.

**Q: Why does /etc/resolv.conf show 127.0.0.53?**

A: That is systemd-resolved's local stub listener. Applications ask it, and it caches answers and forwards to the real servers, which it tracks per interface. The file is normally a link to `/run/systemd/resolve/stub-resolv.conf`. I would run `resolvectl status` to see the real servers.

**Q: dig returns the right answer, but curl goes to the wrong address. Why?**

A: They use different paths. `dig` talks directly to the server in `resolv.conf`. `curl` uses the system resolver, which follows `nsswitch.conf`, usually `/etc/hosts` first. So a stale `/etc/hosts` entry is the likely cause. A local cache is another. `getent hosts name` shows what applications really get.

**Q: What does STALE mean in ip neigh?**

A: The entry has not been confirmed recently, but it is still valid and still used. On next use, the kernel moves it to DELAY and then PROBE if nothing confirms it. Only FAILED means the neighbour did not respond.

**Q: What is rp_filter, and when does it cause trouble?**

A: Reverse path filtering drops a packet if the route back to its source would use a different interface. Strict mode is 1, loose is 2. It helps against spoofing, but it breaks asymmetric routing. The symptom is packets visible in `tcpdump` that the application never receives. The fix is usually loose mode on those interfaces.

**Q: How do you find which host veth belongs to a container?**

A: Inside the container, read `/sys/class/net/eth0/iflink`, which gives the peer's interface index. Then on the host, run `ip -o link` and find that index. `ip link` inside the container also shows it as `eth0@ifN`.

## Quick Recap

- The kernel keeps separate state per namespace: interfaces (`ip link`, `ip addr`), routes (`ip route`) and neighbours (`ip neigh`). `ip route get` shows the real decision.
- DNS is not in the kernel. Applications follow `nsswitch.conf`, then `resolv.conf`. `127.0.0.53` is systemd-resolved, and `resolvectl status` shows the real servers.
- A namespace is a separate network stack, a veth pair is a virtual cable, and a bridge is a software switch.
- Same subnet on a bridge is Layer 2. Crossing subnets needs a router and `ip_forward=1`. Reaching the internet from private addresses also needs NAT.
- `sysctl` changes kernel settings live, and `/etc/sysctl.d/` makes them permanent. `ip` changes are also lost on reboot.
- Containers are these same pieces put together: namespace, veth, bridge, route, forwarding and NAT.
