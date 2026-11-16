# NAT and PAT

**Level:** Intermediate | **Prerequisites:** [IPv4 Addressing Basics](01-ipv4-basics.md), [Public, Private and Special IP Addresses](03-public-private-and-special-addresses.md), [The Default Gateway](08-default-gateway.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md)

## In One Line

NAT (Network Address Translation) rewrites IP addresses in packets as they pass through a router. PAT (Port Address Translation) also rewrites port numbers, so thousands of devices can share one public IP address.

## Why It Matters

IPv4 has only 2^32 addresses, about 4.3 billion. That is fewer than the number of phones, laptops, servers and sensors in use. IANA, the body at the top of address allocation, gave out its last free blocks in February 2011.

NAT is the main reason IPv4 still works. Your home, your office, your phone network, your cloud VPC and your Docker host all use it.

RFC 1918 set aside private ranges that anyone can reuse: 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. Internet routers do not carry routes for these ranges. So a device with a private address needs something to swap that address for a public one. That something is NAT.

NAT also has other uses:

- It joins two networks that use the same private range, for example after a company merger.
- It lets outside users reach a private server (port forwarding).
- It lets you change ISP without changing every inside address.

Interviewers ask about NAT because it tests many things at once: addressing, ports, stateful devices, routing order and troubleshooting. Questions such as "why can two people not both host port 443 on one public IP?" or "why does a call work with one friend but not with another?" quickly show who really understands it.

## Simple Explanation

Think of an office with 200 staff and one public phone number. Each person has a private extension.

When Sara calls a customer, the call goes out through the reception desk. The customer sees only the company number. The receptionist writes in a notebook: "Sara called this customer." When the customer calls back, the receptionist reads the notebook and connects the call to Sara.

Map this to networking:

| Office | Network |
|---|---|
| Staff member | Device on the LAN |
| Extension number | Private IP address and port |
| Company phone number | Public IP address |
| Receptionist | NAT router |
| Notebook | NAT translation table |

Three forms of the idea:

- **Static NAT:** one person gets their own permanent public phone number.
- **Dynamic NAT:** the company owns a few public numbers. Staff borrow one while they are on a call.
- **PAT:** everyone shares one public number, and the receptionist tells calls apart with a tag.

**Where the analogy stops being true:** a receptionist understands who is calling. A NAT router does not. It only rewrites numbers in packet headers, using simple table lookups. It also never decides "this call looks useful". If no table entry matches an incoming packet, it usually drops the packet.

## How It Works

### Basic words first

- **Packet:** a piece of data with a header. The header holds the source address (who sent it) and the destination address (where it goes).
- **Port:** a 16-bit number (0-65535) that says which program or conversation on a device a packet belongs to.
- **Private address:** an address from the RFC 1918 ranges. It is valid only inside a local network.
- **Public address:** an address that is globally unique and routed on the Internet.
- **Inside interface:** the router port facing your LAN.
- **Outside interface:** the router port facing the ISP or the Internet.
- **Translation table:** the list of current address and port mappings kept by the NAT device.
- **Stateful:** the device remembers past packets and uses that memory to handle new ones. NAT with dynamic entries is stateful.

### The topology used in this chapter

All examples below use this small office, unless they say otherwise.

```text
 PC-A  10.0.0.10 --+
 PC-B  10.0.0.11 --+-- [switch] --+
 SRV   10.0.0.20 --+              |
                                  |
                        G0/0 10.0.0.1/24  (inside)
                               [ R1 ]
                        G0/1 203.0.113.2/28  (outside)
                                  |
                        ISP gateway 203.0.113.1
                                  |
                              Internet
                                  |
              Web servers 198.51.100.7 and 198.51.100.8
```

The office owns the public block 203.0.113.0/28. Usable addresses are 203.0.113.1 to 203.0.113.14.

### Inside local, inside global, outside local, outside global

Cisco uses four terms. They confuse many people, but the rule is simple.

- The **first word** says whose address it is. **Inside** means our device. **Outside** means the remote device.
- The **second word** says where you stand when you look at it. **Local** means seen from the inside network. **Global** means seen from the outside network.

| Term | Whose address | Seen from | Value in our example |
|---|---|---|---|
| Inside local | Our PC | Inside | 10.0.0.10 |
| Inside global | Our PC | Outside | 203.0.113.2 |
| Outside local | Web server | Inside | 198.51.100.7 |
| Outside global | Web server | Outside | 198.51.100.7 |

Outside local and outside global are normally the same, because normally we do not translate the remote side. They differ only when you also translate outside addresses. See [translating both sides](#translating-both-sides-overlapping-networks) later.

```text
       INSIDE                  R1                  OUTSIDE
 +-----------------+        +------+        +-----------------+
 | Inside local    |        |      |        | Inside global   |
 | 10.0.0.10       |------->| NAT  |------->| 203.0.113.2     |
 | Outside local   |        |      |        | Outside global  |
 | 198.51.100.7    |        |      |        | 198.51.100.7    |
 +-----------------+        +------+        +-----------------+
    Local = the view          Global = the view from out there
    from in here
```

### Static NAT

Static NAT is a fixed one-to-one mapping between one private and one public address. An admin configures it by hand, and it stays in the table all the time.

```text
 Inside local      Inside global
 10.0.0.20    <->  203.0.113.10
```

It works in both directions. A PC inside can go out, and anyone on the Internet can start a connection in.

Use it for servers that need a fixed public address. It saves no addresses, because each inside host uses one public address.

### Dynamic NAT

Dynamic NAT maps inside hosts to a pool of public addresses. The mapping is created when traffic starts, and removed when the entry times out.

```text
 Pool: 203.0.113.3 to 203.0.113.6 (4 addresses)

 Inside local      Inside global
 10.0.0.10    ->   203.0.113.3
 10.0.0.11    ->   203.0.113.4
 10.0.0.12    ->   203.0.113.5
 10.0.0.13    ->   203.0.113.6
 10.0.0.14    ->   none left, new traffic fails
```

It is still one-to-one at any moment. If five hosts want to talk at once and the pool has four addresses, the fifth fails until an entry times out. For this reason, dynamic NAT without ports is rare today.

### PAT (also called NAPT or NAT overload)

PAT maps many inside hosts to one public address (or a few), and uses the port number to tell conversations apart.

Names differ, but the idea is the same:

- **PAT:** Port Address Translation, the common Cisco term.
- **NAPT:** Network Address and Port Translation, the term in RFC 3022.
- **NAT overload:** the Cisco IOS keyword.
- **Masquerade:** the Linux form that uses the outgoing interface address.

When people say "NAT" at home or in the cloud, they nearly always mean PAT.

### PAT translation table walk-through

Follow four events on R1, which uses PAT with its address 203.0.113.2.

**Event 1. PC-A opens a connection to 198.51.100.7 on port 443.**

PC-A picks a temporary source port, 51000. The router checks whether 203.0.113.2:51000 is free. It is, so it keeps the same port.

```text
 Before R1:  SRC 10.0.0.10:51000     DST 198.51.100.7:443
 After  R1:  SRC 203.0.113.2:51000   DST 198.51.100.7:443
```

**Event 2. PC-B opens a connection to the same server, and by chance also picks port 51000.**

Keeping the port would create a clash, since two inside hosts would then look identical to the server. So R1 picks a free port, here 1024.

```text
 Before R1:  SRC 10.0.0.11:51000     DST 198.51.100.7:443
 After  R1:  SRC 203.0.113.2:1024    DST 198.51.100.7:443
```

**Event 3. PC-A opens a second connection, to another server.**

```text
 Before R1:  SRC 10.0.0.10:51001     DST 198.51.100.8:443
 After  R1:  SRC 203.0.113.2:51001   DST 198.51.100.8:443
```

R1's table now looks like this:

| Pro | Inside local | Inside global | Outside global |
|---|---|---|---|
| TCP | 10.0.0.10:51000 | 203.0.113.2:51000 | 198.51.100.7:443 |
| TCP | 10.0.0.11:51000 | 203.0.113.2:1024 | 198.51.100.7:443 |
| TCP | 10.0.0.10:51001 | 203.0.113.2:51001 | 198.51.100.8:443 |

**Event 4. The server replies to PC-B.**

```text
 Arrives at R1:  SRC 198.51.100.7:443  DST 203.0.113.2:1024
 R1 finds the entry for inside global 203.0.113.2:1024
 Leaves R1:      SRC 198.51.100.7:443  DST 10.0.0.11:51000
```

PC-B gets the reply. It never knows that NAT took place. The server also has no idea that PC-A and PC-B are two machines.

The fuller truth about port choice: routers differ. Cisco documents that IOS tries to keep the original port, and if that port is taken, it looks for a free port in the same group (0-511, 512-1023 or 1024-65535). Linux also tries to keep the port. Other devices pick a random port on purpose. Do not rely on any exact number.

### What NAT really changes in a packet

For the outbound SNAT in Event 1:

| Header field | Changed? | Why |
|---|---|---|
| Source IP | Yes | 10.0.0.10 becomes 203.0.113.2 |
| Source port | Sometimes | Changed if the port clashes, or by policy |
| Destination IP | No | Still the web server |
| Destination port | No | Still 443 |
| TTL | Yes | Reduced by 1, as on any router hop |
| IPv4 header checksum | Yes | The header changed, so the check value must change |
| TCP or UDP checksum | Yes | It covers the IP addresses as well, through a "pseudo-header" |

A **checksum** is a small number computed from the data, used to detect damage. NAT must fix these or the receiver will discard the packet.

### SNAT and DNAT

These two terms say which address in the packet is rewritten when the connection starts.

- **SNAT (Source NAT):** rewrites the source address. Typical use: inside hosts going out to the Internet.
- **DNAT (Destination NAT):** rewrites the destination address. Typical use: outside users coming in to a private server.

| | SNAT | DNAT |
|---|---|---|
| Rewrites | Source | Destination |
| Typical direction | Inside to outside | Outside to inside |
| Common example | Internet access, PAT | Port forwarding, load balancer |
| Linux chain | POSTROUTING | PREROUTING |
| When applied on Linux | After the routing decision | Before the routing decision |

```text
 SNAT
 10.0.0.10 -> 198.51.100.7   becomes   203.0.113.2 -> 198.51.100.7

 DNAT
 198.51.100.7 -> 203.0.113.2   becomes   198.51.100.7 -> 10.0.0.20
```

Reply packets are reversed by themselves. If you SNAT outgoing traffic, the NAT device un-does it on the reply (the reply's destination is changed back). You do not write a second rule for that.

**Masquerade** is a special SNAT. It uses whatever address the outgoing interface has now. This suits links with a changing address from DHCP or PPPoE. If the address is fixed, plain SNAT is slightly cheaper for the kernel.

The Cisco mapping: `ip nat inside source` does SNAT on the way out. Its reverse does DNAT on replies. A static entry also allows connections to start from outside, which is DNAT in practice.

### Port forwarding

Port forwarding is a static DNAT rule for one port. It says: "Anything that reaches my public address on this port, send to this inside host and port."

```text
 Rule: TCP 203.0.113.2:8443 -> 10.0.0.20:443

 Internet client                R1                     SRV
 192.0.2.50:40000
      |                          |                      |
      |-- DST 203.0.113.2:8443 ->|                      |
      |                          |-- DST 10.0.0.20:443->|
      |                          |<-- SRC 10.0.0.20:443-|
      |<-- SRC 203.0.113.2:8443 -|                      |
```

A public IP and port pair can point to just one inside target. Two inside web servers cannot both own public TCP 443 on one address. Your choices are to use another public port, get a second public IP, or put a [reverse proxy](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md) in front of them that reads the host name.

For port forwarding to work, all of these must be true:

1. The server's default gateway points to the NAT router (or the reply must come back through it another way).
2. The server's own firewall allows the port.
3. The router's firewall allows the traffic.
4. The address on the router's WAN port is truly public. Behind CGNAT, this fails.
5. The ISP does not block the port. Some home ISPs block ports such as 25 or 80.

### Hairpin NAT

Hairpin NAT (also called NAT loopback or NAT reflection) lets an inside host reach an inside server by using the public address.

The packet enters the router on the LAN side and leaves on the same LAN side. It makes a U-turn, like a hairpin.

**Why it breaks without care.** PC-A browses to 203.0.113.2:443, which forwards to SRV. Here is what happens with DNAT only:

```text
 PC-A 10.0.0.10          R1 10.0.0.1          SRV 10.0.0.20
    |                        |                     |
    |--- DST 203.0.113.2 --->|                     |
    |                        |-- DST 10.0.0.20 --->|
    |                        |  (SRC is still      |
    |                        |   10.0.0.10)        |
    |<------- SRC 10.0.0.20, sent directly --------|
    |
    PC-A expected a reply from 203.0.113.2.
    This reply does not match, so it is rejected.
```

The server sees a source on its own subnet, so it replies directly through the switch. The router never sees the reply, so it cannot reverse the translation.

**The fix:** the router also SNATs the hairpin traffic, so the server replies to the router.

```text
 PC-A 10.0.0.10          R1 10.0.0.1          SRV 10.0.0.20
    |                        |                     |
    |--- DST 203.0.113.2 --->|                     |
    |                        |-- SRC 10.0.0.1 ---->|
    |                        |   DST 10.0.0.20     |
    |                        |<-- reply to R1 -----|
    |<--- SRC 203.0.113.2 ---|                     |
```

The side effect: the server's logs show the router's address, not the real client.

A common alternative is **split DNS** (split-horizon DNS): inside users get 10.0.0.20 for the name, outside users get 203.0.113.2. The traffic then avoids the router altogether.

Vendor support varies. Many home routers and firewalls do hairpin. On Cisco IOS, classic `ip nat inside` and `ip nat outside` NAT does not handle this case by default. NAT Virtual Interface (`ip nat enable` on the interfaces, with `ip nat source` commands) can, on platforms that support it. See the Linux rules in [Example 2](#example-2-a-linux-router).

### CGNAT

CGNAT (Carrier-Grade NAT, also written CGN or LSN, Large-Scale NAT) is NAT run by the ISP, for many customers at once. It exists because ISPs do not have enough public IPv4 addresses to give one to each customer.

```text
  Home LAN              ISP network              Internet
 192.168.1.0/24        100.64.0.0/10

 [PC]-----[home router]-----[CGNAT]------[web server]
 192.168.1.10   100.72.15.9   198.51.100.77   192.0.2.80
           NAT 1                NAT 2
```

This double translation is often called **NAT444**: three IPv4 address zones with two NAT steps between them.

**The shared address range.** RFC 6598 reserves 100.64.0.0/10 (100.64.0.0 to 100.127.255.255, about 4.19 million addresses) for this link between the customer router and the CGNAT. ISPs do not use RFC 1918 space there, because a customer's LAN might already use the same range, and the home router would then see the same subnet on both sides.

**How to tell if you are behind CGNAT.** Compare two values:

1. The WAN address on your router's status page.
2. The address a public service sees.

```bash
# Linux or macOS
curl -s https://api.ipify.org
```

```powershell
# Windows
Invoke-RestMethod https://api.ipify.org
```

If the router shows 100.72.15.9 and the service shows 198.51.100.77, you are behind CGNAT. Some ISPs use other ranges, so the mismatch is the true test, not the range alone.

**Effects of CGNAT:**

- Port forwarding on your home router does nothing, because the ISP's NAT drops the traffic first.
- Many customers share one public IP. If one is banned by a website, others can be hit too.
- Each customer gets a limited number of ports. Apps that open many connections can run out.
- The ISP must log who used which port at what time, so it can answer legal requests.
- Peer-to-peer apps and online games can have more trouble.

To save logs, ISPs often give each customer a block of ports. For example, sharing one public IP among 64 customers gives each about 1,008 ports (64,512 / 64). The ISP then logs one block rather than each connection. This is an example, not a standard size. Deterministic NAT (RFC 7422) goes further: it computes the mapping by formula, so little logging is needed.

The real fix for address shortage is [IPv6](11-ipv6-basics.md).

## Example

### Example 1: Cisco IOS, step by step

**Option A: static NAT.** SRV always appears as 203.0.113.10.

```text
! Cisco IOS on R1
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/1
 ip address 203.0.113.2 255.255.255.240
 ip nat outside
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
!
ip nat inside source static 10.0.0.20 203.0.113.10
```

The ISP sends traffic for 203.0.113.10 to the /28 link, and R1 answers ARP for it, because the address is in the connected subnet. If the address were in another subnet, the ISP would need a route to R1.

**Option B: dynamic NAT with a pool.**

```text
! Cisco IOS on R1 (interfaces as above)
access-list 10 permit 10.0.0.0 0.0.0.255
!
ip nat pool OFFICE 203.0.113.3 203.0.113.6 netmask 255.255.255.240
ip nat inside source list 10 pool OFFICE
```

The ACL (Access Control List, a list of match rules) chooses which inside hosts get translated. The `0.0.0.255` is a wildcard mask: 0 bits must match, 1 bits are ignored. More in [Access Control Lists](../09-security/05-access-control-lists.md).

Add `overload` to the last line to get PAT over the pool.

**Option C: PAT on the interface address, plus port forwarding.** This is the most common setup.

```text
! Cisco IOS on R1 (interfaces as above)
access-list 10 permit 10.0.0.0 0.0.0.255
!
ip nat inside source list 10 interface GigabitEthernet0/1 overload
ip nat inside source static tcp 10.0.0.20 443 203.0.113.2 443
```

**Verify after PC-A and PC-B browse to 198.51.100.7:**

```text
R1# show ip nat translations
Pro Inside global     Inside local    Outside local    Outside global
tcp 203.0.113.2:443   10.0.0.20:443   ---              ---
tcp 203.0.113.2:51000 10.0.0.10:51000 198.51.100.7:443 198.51.100.7:443
tcp 203.0.113.2:1024  10.0.0.11:51000 198.51.100.7:443 198.51.100.7:443
```

(Columns are tightened here to fit the page. IOS spacing differs slightly.)

The first line, with `---`, is the static port forward. It has no outside host until someone connects.

```text
R1# show ip nat statistics
Total active translations: 3 (1 static, 2 dynamic; 3 extended)
Outside interfaces:
  GigabitEthernet0/1
Inside interfaces:
  GigabitEthernet0/0
Hits: 1542  Misses: 37
```

Reading this output:

- **Extended** entries include port numbers.
- **Hits** are packets that matched an existing entry.
- **Misses** are packets that needed a new entry.
- If inside and outside interfaces are swapped here, NAT will not work. Check this first.

Other useful commands:

```text
R1# show ip nat translations verbose
R1# clear ip nat translation *
R1# debug ip nat
```

Use `debug ip nat` with care on a busy router, since it can flood the console. `clear ip nat translation *` removes dynamic entries and cuts active sessions.

### Example 2: A Linux router

Here `eth0` is the WAN side (203.0.113.2/28) and `eth1` is the LAN side (10.0.0.1/24).

**Step 1. Turn on IP forwarding.** Linux does not route between interfaces by default.

```bash
# Linux, now
sudo sysctl -w net.ipv4.ip_forward=1

# Linux, after reboots
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-router.conf
```

**Step 2. Add rules with iptables.**

```bash
# Linux: PAT for the LAN. Use one of these two lines.
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 \
  -j SNAT --to-source 203.0.113.2

# Linux: port forward 203.0.113.2:443 to SRV
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 \
  -j DNAT --to-destination 10.0.0.20:443

# Linux: allow the traffic through the filter table
sudo iptables -A FORWARD -i eth1 -o eth0 -s 10.0.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp -d 10.0.0.20 \
  --dport 443 -j ACCEPT
sudo iptables -A FORWARD -m conntrack \
  --ctstate ESTABLISHED,RELATED -j ACCEPT

# Linux: hairpin, so LAN users can use the public address
sudo iptables -t nat -A PREROUTING -i eth1 -d 203.0.113.2 \
  -p tcp --dport 443 -j DNAT --to-destination 10.0.0.20:443
sudo iptables -t nat -A POSTROUTING -o eth1 -s 10.0.0.0/24 \
  -d 10.0.0.20 -p tcp --dport 443 -j MASQUERADE
```

The `nat` table changes addresses. The `filter` table (the FORWARD rules) decides what may pass. A DNAT rule alone does not allow traffic if FORWARD drops it.

**The same PAT and port forward in nftables.** nftables is the modern Linux packet filter framework.

```text
# Linux: /etc/nftables.conf (load with: sudo nft -f /etc/nftables.conf)
table ip nat {
  chain prerouting {
    type nat hook prerouting priority dstnat; policy accept;
    iifname "eth0" tcp dport 443 dnat to 10.0.0.20:443
  }
  chain postrouting {
    type nat hook postrouting priority srcnat; policy accept;
    oifname "eth0" masquerade
  }
}
```

More in [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

**Step 3. Check the table.** Linux keeps NAT state in **conntrack**, the kernel's connection tracking system.

```bash
# Linux: needs the conntrack tool, often in a package named conntrack
sudo conntrack -L -p tcp --dport 443
```

```text
tcp 6 431982 ESTABLISHED src=10.0.0.10 dst=198.51.100.7 sport=51000
  dport=443 src=198.51.100.7 dst=203.0.113.2 sport=443 dport=51000
  [ASSURED] mark=0 use=1
```

(One line in real output, wrapped here.)

The first set of addresses is the original direction. The second set is the reply the kernel expects. The reply has `dst=203.0.113.2`, not 10.0.0.10. That difference tells you that SNAT is active.

**Step 4. Watch the rewrite on the wire.**

```bash
# Linux: run in two terminals on the router
sudo tcpdump -ni eth1 host 198.51.100.7 and tcp port 443
sudo tcpdump -ni eth0 host 198.51.100.7 and tcp port 443
```

```text
eth1: IP 10.0.0.10.51000 > 198.51.100.7.443: Flags [S] ...
eth0: IP 203.0.113.2.51000 > 198.51.100.7.443: Flags [S] ...
```

Same packet, two views. See [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

### Example 3: Windows

Windows clients do not normally act as NAT routers. For labs and virtual machines, Windows 10, Windows 11 and Windows Server have the built-in NetNat module. It needs an internal virtual interface, such as a Hyper-V internal switch.

```powershell
# Windows, elevated PowerShell. vEthernet already has 172.16.0.1/24.
New-NetNat -Name LabNat -InternalIPInterfaceAddressPrefix 172.16.0.0/24

# Port forward host port 8080 to a VM on port 80
Add-NetNatStaticMapping -NatName LabNat -Protocol TCP `
  -ExternalIPAddress 0.0.0.0 -ExternalPort 8080 `
  -InternalIPAddress 172.16.0.10 -InternalPort 80

Get-NetNat
Get-NetNatStaticMapping
Get-NetNatSession
```

Windows also has Internet Connection Sharing, a simple home feature, and Windows Server has Routing and Remote Access for full NAT.

## Going Deeper

### Order of operations

NAT changes addresses, so other features see different values depending on when they run.

**Cisco IOS (classic NAT):**

- Inside to outside: routing happens first, then NAT.
- Outside to inside: NAT happens first, then routing.

So an inbound ACL on the outside interface sees the **public** destination, because NAT has not yet run.

**Linux:**

```text
 Packet in
    |
 PREROUTING   <- DNAT happens here
    |
 Routing decision
    |
 FORWARD      <- filter rules see the private destination
    |
 POSTROUTING  <- SNAT happens here
    |
 Packet out
```

So a Linux FORWARD rule must match the **private** destination (10.0.0.20) and the original **private** source.

### Translating both sides (overlapping networks)

Suppose two merged companies both use 10.0.0.0/24, and they connect by a private link. Host 10.0.0.10 in Company A needs host 10.0.0.50 in Company B. To Company A's router, that destination looks like a local address. Traffic would never leave.

The fix is to translate both source and destination. This is sometimes called **twice NAT**.

```text
! Cisco IOS on Company A's edge router
ip nat inside source static 10.0.0.10 172.16.10.10
ip nat outside source static 10.0.0.50 172.16.20.50 add-route
```

| Term | Value |
|---|---|
| Inside local | 10.0.0.10 |
| Inside global | 172.16.10.10 |
| Outside local | 172.16.20.50 |
| Outside global | 10.0.0.50 |

Here all four differ. Host A sends to 172.16.20.50. Company B's host sees traffic from 172.16.10.10. Neither side sees a clash.

`add-route` matters since routing runs before NAT on the inside-to-outside path. The router needs a route to 172.16.20.50 before it can translate it.

### NAT behaviour: mapping and filtering

Not all NATs act alike. RFC 4787 describes two separate behaviours.

**Mapping:** when does the NAT reuse the same public IP and port?

| Mapping type | Same public port is reused for |
|---|---|
| Endpoint-independent | All destinations |
| Address-dependent | The same destination IP |
| Address and port-dependent | The same destination IP and port |

**Filtering:** who may send packets in through an existing mapping?

| Filtering type | Inbound packet is allowed from |
|---|---|
| Endpoint-independent | Anyone |
| Address-dependent | An IP the inside host already sent to |
| Address and port-dependent | The exact IP and port already sent to |

Older books use the "cone" names from RFC 3489, which is now obsolete. They still come up in interviews.

| Old name | Mapping | Filtering |
|---|---|---|
| Full cone | Endpoint-independent | Endpoint-independent |
| Restricted cone | Endpoint-independent | Address-dependent |
| Port-restricted cone | Endpoint-independent | Address and port-dependent |
| Symmetric | Address (and port) dependent | Address and port-dependent |

RFC 4787 requires endpoint-independent mapping for UDP, because it makes peer-to-peer work. Real devices do not always follow it.

### Timeouts

Dynamic entries are deleted after a period of no traffic. The values matter: too short and idle connections break, too long and the table fills.

| Item | Value | Source |
|---|---|---|
| Minimum UDP mapping timeout | 2 minutes, 5 or more recommended | RFC 4787 |
| Minimum TCP established idle timeout | 2 hours 4 minutes | RFC 5382 |
| Cisco IOS TCP | 24 hours | Common default |
| Cisco IOS UDP | 5 minutes | Common default |
| Linux conntrack TCP established | 5 days (432000 s) | Common default |
| Linux conntrack UDP, unreplied | 30 seconds | Common default |

Defaults vary by version and platform. Check with `show ip nat translations verbose` on Cisco, and `sysctl net.netfilter | grep timeout` on Linux.

This is why long-lived idle connections need keepalives: small packets sent to keep the entry alive.

```text
! Cisco IOS: change timers (seconds)
ip nat translation tcp-timeout 3600
ip nat translation udp-timeout 120
```

### Port exhaustion

One public IP has 64,512 ports above 1023, per protocol. Does that mean only 64,512 connections? No, not quite.

A connection is identified by its **5-tuple**: protocol, source IP, source port, destination IP, destination port. A NAT may reuse the same public port for different destinations, because the 5-tuple is still unique.

So the true limit is about 64,512 connections **to one destination IP and port**. That is exactly what hurts in the cloud, where thousands of workers call one API or one database. AWS documents that a NAT gateway supports up to 55,000 simultaneous connections to each unique destination, per assigned IP address.

Fixes for exhaustion:

- Add more public IPs to the NAT.
- Reuse connections with connection pooling.
- Use private endpoints, so traffic skips the NAT.

On Linux, a separate limit is the conntrack table. When full, the kernel logs `nf_conntrack: table full, dropping packet`. Check it this way:

```bash
# Linux
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
```

### Problems NAT causes

- **Breaks end-to-end addressing.** Hosts behind NAT cannot be reached directly unless a rule exists.
- **Breaks protocols that put addresses inside the data.** FTP active mode sends the client's IP in the `PORT` command. SIP (voice calling) puts addresses in its message body. NAT changes headers but not this payload, so the other side gets a private address it cannot reach. A device feature called **ALG** (Application Layer Gateway) can read the payload and fix it, but ALGs often cause bugs, and encrypted traffic hides the payload anyway. See [File Transfer Protocols](../07-application-protocols/08-file-transfer-protocols.md).
- **Breaks IPsec in some modes.** AH protects the IP header with a hash, so changing the address fails the check. ESP has no port numbers for PAT to use. **NAT-T** (NAT Traversal) wraps ESP in UDP port 4500 (RFC 3948). See [IPsec](../09-security/08-ipsec.md).
- **Makes tracing harder.** The outside sees one IP for many users. To find a user, you need IP, port and exact time. RFC 6302 tells Internet servers to log source ports for this reason.
- **Adds state, so failure hurts.** If a NAT device dies, active sessions die too, unless state is synced to a standby device. NAT also needs **symmetric routing**: replies must pass through the same device that holds the entry. Two NAT routers with asymmetric paths will drop traffic.
- **Adds work and a little delay.** Each packet needs a lookup and a rewrite, and each flow uses memory.
- **Has trouble with fragments.** Only the first IP fragment has the port numbers. NAT must track or reassemble the rest. See [IPv4 Header and Fragmentation](10-ipv4-header-and-fragmentation.md).
- **Punishes neighbours.** Many users share one IP, so one abuser can cause rate limits or CAPTCHAs for all.

ICMP needs special care too. Echo messages have no port, so NAT uses the ICMP identifier instead. For ICMP error messages, NAT must also fix the copy of the original header inside the message (RFC 5508). See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

### NAT traversal: STUN, TURN and ICE

**The problem.** Alice and Bob each sit behind a NAT and want a video call. Neither can receive a new inbound connection. Sending all media through a central server works, but costs money and adds delay. We want a direct path if one is possible.

A **signalling server** first helps them swap information, such as addresses. It does not carry the media.

#### STUN: "What do I look like from outside?"

STUN (Session Traversal Utilities for NAT, RFC 8489) is a simple request and reply. The default port is 3478 on UDP or TCP, and 5349 for TLS.

```text
 Alice              Her NAT              STUN server
 10.0.0.10:50000    203.0.113.2          192.0.2.10:3478
    |                  |                      |
    |-- Binding req -->|                      |
    |                  |-- SRC 203.0.113.2:   |
    |                  |   40001 ------------>|
    |                  |<-- "I see you as ----|
    |<-- same reply ---|   203.0.113.2:40001" |
```

Alice now knows her public mapping. The address in the reply is XOR-encoded (XOR-MAPPED-ADDRESS), so that a NAT with an overeager ALG will not rewrite it.

Analogy: you phone a friend and ask, "What number shows on your screen?" The analogy stops here: STUN only tells you the mapping. It does not open a path by itself, and it never carries your call.

#### UDP hole punching

Alice learns 203.0.113.2:40001. Bob learns 198.51.100.20:50002. They swap these through the signalling server, then both send at the same time.

```text
 Alice's NAT                                  Bob's NAT
 203.0.113.2:40001                     198.51.100.20:50002
    |                                             |
    |--- 1. Alice sends to Bob ------------------>| dropped, no
    |    (her NAT now allows Bob's replies)       | state yet
    |                                             |
    |<-- 2. Bob sends to Alice -------------------|
    |    allowed, Alice's NAT expects Bob         |
    |    (his NAT now allows Alice too)           |
    |                                             |
    |<========= 3. Direct path is open ==========>|
```

Each side's outgoing packet "punches a hole" in its own NAT, and the other's packet goes through that hole.

It works when mappings are endpoint-independent. It usually fails when both NATs are symmetric, or when one is symmetric and the other uses address and port-dependent filtering. In those cases, the public port learned from STUN is not the one used for the peer. TCP hole punching exists too, but works less reliably than UDP.

#### TURN: "Please relay for me"

TURN (Traversal Using Relays around NAT, RFC 8656) is the fallback. The client asks a public TURN server to allocate a relay address. The peer sends to that address, and the server passes data along.

```text
 Alice ----> TURN server 192.0.2.30:49160 ----> Bob
       <----                              <----
```

TURN is an extension of STUN and normally uses the same ports. TURN works almost everywhere, because all traffic is outbound from the client's view, and it can even run over TLS on TCP. But it costs bandwidth, since the server carries all the media, and it adds delay.

Analogy: a friend who holds two phones and repeats everything between you. It always works, but your friend must stay on the line, and that costs them.

#### ICE: "Try everything and choose the best"

ICE (Interactive Connectivity Establishment, RFC 8445) is the process that uses STUN and TURN together. WebRTC, the browser real-time media system, uses it. See [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

Steps:

1. **Gather candidates.** A candidate is an IP and port where the peer might reach you.
   - **Host:** your local address, 10.0.0.10:50000.
   - **Server-reflexive (srflx):** your public mapping from STUN, 203.0.113.2:40001.
   - **Relayed:** your TURN address, 192.0.2.30:49160.
   - **Peer-reflexive (prflx):** an address discovered during checks.
2. **Exchange candidates** through the signalling server. WebRTC commonly carries them in SDP (Session Description Protocol).
3. **Pair and check.** Each side pairs its candidates with the peer's, and tests each pair with STUN packets.
4. **Choose the best working pair,** preferring direct paths over relays.

Candidates as they look in SDP, using the priorities recommended in RFC 8445:

```text
a=candidate:1 1 udp 2130706431 10.0.0.10 50000 typ host
a=candidate:2 1 udp 1694498815 203.0.113.2 40001 typ srflx
  raddr 10.0.0.10 rport 50000
a=candidate:3 1 udp 16777215 192.0.2.30 49160 typ relay
  raddr 203.0.113.2 rport 40001
```

(Candidates 2 and 3 are one line each in real SDP, wrapped here.)

The large number is the priority. Higher is better, so host beats srflx, and srflx beats relay. **Trickle ICE** (RFC 8838) sends candidates as they are found, so calls connect faster.

| Tool | Purpose | Carries media? |
|---|---|---|
| STUN | Finds your public address and port | No |
| TURN | Relays traffic when direct fails | Yes |
| ICE | Tests all options, picks the best | No, it is a process |

### Asking the router directly: UPnP, NAT-PMP and PCP

Some apps can ask the router to open a port for them.

- **UPnP IGD:** an older, popular method on home routers. It has a poor security record, because any program on the LAN can open ports. Many admins disable it.
- **NAT-PMP (RFC 6886):** a simpler Apple-designed protocol.
- **PCP (Port Control Protocol, RFC 6887):** the successor to NAT-PMP, and can also work with a carrier's NAT if the ISP allows it.

### NAT and IPv6

IPv6 has enough addresses that NAT for address sharing is not needed. Every device can have a global address, and a stateful firewall still gives protection.

**NAT64** (RFC 6146) with **DNS64** (RFC 6147) is a different thing: it lets IPv6-only hosts reach IPv4 servers. Many mobile networks use it. See [IPv6 in Practice](12-ipv6-in-practice.md).

### NAT in other places

- **Cloud:** private subnets use a NAT gateway for outbound Internet access. See [AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).
- **Containers:** Docker uses masquerade for outbound traffic, and DNAT for published ports. See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md). Kubernetes Services often use DNAT as well. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).
- **Load balancers:** many use DNAT to send traffic to back-end servers. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).
- **Firewalls:** Cisco ASA and Firepower use different commands from IOS (object NAT and twice NAT), but the same ideas. See [Firewalls](../09-security/04-firewalls.md).

## Common Mistakes

- **"NAT is a firewall."** NAT drops unmatched inbound packets as a side effect, but it does not inspect traffic and it was not built for security. Hosts inside still download malware through their own outbound connections. Use a real [firewall](../09-security/04-firewalls.md).
- **Marking the wrong interface.** `ip nat inside` goes on the LAN side, `ip nat outside` on the WAN side. Swapping or forgetting one is the most common Cisco NAT fault.
- **Writing the ACL with the public address.** The ACL in `ip nat inside source list` must match the inside local addresses, not the translated one.
- **Mixing up inside global and outside local.** Inside global is your own public address. Outside local is the remote host, as seen from inside.
- **Forgetting `net.ipv4.ip_forward=1` on Linux.** The NAT rules look fine, but no packets get forwarded.
- **Forgetting the FORWARD rule on Linux.** A DNAT rule changes the address but does not permit the traffic.
- **Putting SNAT in PREROUTING or DNAT in POSTROUTING.** SNAT belongs in POSTROUTING, DNAT in PREROUTING.
- **Adding a port forward behind CGNAT.** The rule is valid, but traffic never reaches your router.
- **Forgetting the server's default gateway.** With a port forward, a server whose gateway points elsewhere sends replies the wrong way, and the connection fails.
- **Testing a port forward from inside the LAN.** Without hairpin NAT, it fails even when the rule is correct. Test from outside, such as over mobile data.
- **Thinking PAT allows only 65,536 users.** The limit is per destination, not total. Many users can share a port number when their destinations differ.
- **Thinking dynamic NAT saves addresses like PAT does.** Without ports, it is still one public address per active inside host.
- **Thinking STUN carries the call.** STUN only reports your address. TURN relays media.
- **Thinking IPv6 needs NAT for security.** A stateful firewall gives the same inbound protection, without translation.

## Interview Questions

**Q: What is NAT and why do we need it?**

A: NAT rewrites IP addresses in packets as they pass through a router. We need it because public IPv4 addresses ran short. Private hosts use RFC 1918 addresses, which are not routed on the Internet, and NAT swaps them for a public address at the edge.

**Q: What is the difference between static NAT, dynamic NAT and PAT?**

A: Static NAT is a fixed one-to-one mapping, set by hand, and it allows inbound connections. Dynamic NAT maps hosts to a pool of public addresses, one to one, as needed. PAT maps many hosts to one public address by also translating ports, so it is the one that really saves addresses.

**Q: Explain inside local, inside global, outside local and outside global.**

A: Inside or outside says whose address it is, and local or global says from which side you view it. Inside local is my host's private address, and inside global is how the Internet sees my host. Outside local is how my network sees the remote host, and outside global is the remote host's real address. The two outside values are usually equal unless we also translate the outside.

**Q: Walk me through what happens when two PCs behind PAT use the same source port.**

A: Both send to the same server from port 51000. The router keeps 51000 for the first PC. For the second, it picks a free public port, say 1024, and records it in the table. When the reply reaches port 1024, the router looks up the entry and restores the second PC's private address and port.

**Q: What is the difference between SNAT and DNAT?**

A: SNAT changes the source address, usually for inside users going out. DNAT changes the destination, usually for outside users reaching a private server. On Linux, DNAT runs in PREROUTING, before routing, and SNAT runs in POSTROUTING, after routing. Replies are reversed automatically from connection state.

**Q: A port forward is set up, but the server is unreachable. How do you troubleshoot it?**

A: I test from outside, not from the LAN. I check that the WAN address is really public, not CGNAT. Then I capture on the WAN side to see if the packet arrives, and on the LAN side to see if it is rewritten. Then I check the server's listener, its firewall and its default gateway, and whether the ISP blocks the port.

**Q: What is hairpin NAT and why does it need SNAT?**

A: Hairpin NAT lets an inside host reach an inside server using the public address. With DNAT alone, the server sees an inside source and replies directly, so the client gets a reply from an address it does not expect. Adding SNAT makes the server reply through the router, which reverses both translations. Split DNS is a common alternative.

**Q: What is CGNAT, and what does it break?**

A: It is NAT run by the ISP so many customers can share one public IPv4 address. The customer usually gets an address from 100.64.0.0/10, from RFC 6598. It breaks inbound port forwarding, and shared IPs can suffer from bans and port limits. The ISP also needs logs with ports and times to trace users.

**Q: Why does NAT cause problems for IPsec and FTP?**

A: FTP active mode puts an IP address inside the application data, and NAT does not change it without an ALG. IPsec AH hashes the IP header, so a changed address fails the check, and ESP has no ports for PAT. NAT-T solves the ESP case by wrapping it in UDP port 4500.

**Q: Explain STUN, TURN and ICE.**

A: STUN tells a client its public IP and port as seen from outside. TURN is a relay server used when no direct path works. ICE is the process that gathers all candidate addresses, tests the pairs with STUN checks, and picks the best one, preferring direct paths over relays.

**Q: Why does a peer-to-peer call fail with symmetric NAT?**

A: A symmetric NAT uses a different public port for each destination. The port learned from the STUN server is not the port used towards the peer, so the peer sends to the wrong port. If the other side also filters strictly, hole punching fails, and the call falls back to TURN.

**Q: Is NAT a security feature?**

A: Not really. It blocks unsolicited inbound traffic only as a side effect of having no matching entry. It does not inspect traffic or stop attacks on outbound connections. Security comes from a stateful firewall, and IPv6 gets the same protection with no NAT at all.

## Quick Recap

- NAT rewrites addresses. PAT also rewrites ports, so many hosts can share one public IP.
- Static is fixed one-to-one, dynamic is one-to-one from a pool, PAT is many-to-one.
- Inside or outside says whose address it is. Local or global says from which side you see it.
- SNAT changes the source (POSTROUTING). DNAT changes the destination (PREROUTING). Port forwarding is static DNAT.
- Hairpin NAT needs both DNAT and SNAT. CGNAT is ISP-level NAT using 100.64.0.0/10, and it stops port forwarding.
- NAT breaks end-to-end reachability, payload addresses, some IPsec modes and easy tracing, and it is not a firewall.
- STUN finds your public mapping, TURN relays when direct paths fail, and ICE tries the options and picks the best.
