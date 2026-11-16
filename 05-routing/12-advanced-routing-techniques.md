# Advanced Routing Techniques

**Level:** Advanced | **Prerequisites:** [The Routing Table](02-routing-table.md), [Static and Default Routes](03-static-and-default-routes.md), [OSPF](06-ospf.md), [BGP](09-bgp.md), [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md)

## In One Line

Advanced routing is a set of tools to share load across paths, join routing domains safely, choose paths by policy instead of destination, keep customers apart on one router, shrink routing tables, and notice failures in under a second.

## Why It Matters

A basic network needs one routing protocol and a few routes. A real network has more problems:

- A company has two links to the same place and wants to use both. ECMP solves this.
- Two companies merge. One runs OSPF and the other runs EIGRP. Route redistribution joins them, and it can also cause routing loops.
- A router must accept some routes and reject others. Prefix lists and route maps do this.
- Guest Wi-Fi must leave through the cheap internet link, while staff use the good one. Policy-based routing does this.
- A provider has two customers who both use 10.1.1.0/24. VRF keeps them apart on one router.
- A large network has 50,000 routes and every change disturbs every router. Summarisation hides the detail.
- OSPF can take 40 seconds to notice a dead neighbour. Voice calls drop long before that. BFD finds the failure in less than one second.

Interviewers ask about these topics because they separate people who have run networks from people who have only read about protocols. Typical questions are:

- "How does a router split traffic on two equal paths?"
- "Why is two-way redistribution dangerous?"
- "What is the difference between `ge` and `le`?"
- "What happens to a packet that does not match a PBR route map?"
- "What is the difference between VRF and VRF-lite?"
- "Why does a summary create a route to Null0?"
- "How does BFD help OSPF?"

## Simple Explanation

First, six words used all through this chapter.

| Word | Plain meaning |
|---|---|
| Prefix | A network written as address and length, such as 10.1.1.0/24 |
| Route | A line that says "to reach this prefix, send to that next hop" |
| Next hop | The neighbouring router that gets the packet next |
| Metric | A protocol's score for a path. Lower is better. It only compares routes inside one protocol |
| Administrative distance (AD) | A router's trust score for a route source. Lower wins. It compares different sources |
| RIB and FIB | RIB (Routing Information Base) is the routing table. FIB (Forwarding Information Base) is the fast copy used to forward packets |

Now think of a large delivery company.

**ECMP is two equal roads.** Equal-Cost Multi-Path means a router keeps several best paths to one place and uses all of them. Two roads lead to the same city and take the same time. The dispatcher sends some trucks on each road. Each customer always gets the same road, so their parcels arrive in order. *Where the analogy stops:* a dispatcher can look at how busy each road is. A normal router does not. It picks with a fixed formula and does not check load.

**Redistribution is a translator between two offices.** One office speaks OSPF and the other speaks RIP. A translator copies addresses from one list to the other. If two translators work at the same time and nobody marks which list an address came from, an address can be copied around in a circle forever. *Where the analogy stops:* a human would notice the circle. Routers do not, because each protocol loses the other protocol's loop protection at the border.

**A prefix list is a guest list. A route map is the security guard.** The list only says who is on it. The guard decides what to do: let them in, turn them away, or give them a badge. *Where the analogy stops:* a guest not on any list is simply unknown. In a route map, a route that matches no rule at all is dropped by an invisible rule at the end.

**PBR is a special lane.** Normal roads send every truck by address alone. Policy-Based Routing (PBR) adds a rule such as "all refrigerated trucks use the north road, whatever their address". *Where the analogy stops:* the special lane does not update itself. If the north road closes, PBR only notices if you told it how to check.

**VRF is an apartment building.** Virtual Routing and Forwarding (VRF) splits one router into several separate routing tables. Each flat has its own letterboxes. Flat 1 and Flat 2 can both have a "Room 10" with no confusion. *Where the analogy stops:* flats share walls and a building, and VRFs share the CPU and memory. A VRF separates traffic, but it does not encrypt anything.

**Summarisation is a postal code.** A post office in another country does not need every street in your city. It only needs "everything for this city goes there". *Where the analogy stops:* a postal code does not tell you the street exists. A summary can attract traffic for addresses that are not there, which is why routers add a discard route.

**BFD is a walkie-talkie check.** Two guards say "OK" to each other every 300 milliseconds. If three checks are missed, each knows the other has gone, and they tell the manager (the routing protocol). *Where the analogy stops:* BFD does not find new friends. It only checks neighbours that a protocol has already told it about.

## How It Works

### 1. ECMP: using equal paths together

#### When a router installs several paths

A router picks routes in this order:

1. Longest prefix match decides which route applies to a packet.
2. For the same prefix, the lowest AD wins.
3. For the same AD, the lowest metric wins.
4. If two or more routes tie on the same prefix, AD and metric, the router can install all of them, up to its `maximum-paths` limit.

This is ECMP. Routes from different protocols do not share a prefix in this way.

| Source | ECMP behaviour on Cisco IOS |
|---|---|
| Static | Two static routes with the same prefix and AD both install |
| OSPF | On by default, usually 4 paths, changed with `maximum-paths` |
| EIGRP | On by default, usually 4 paths. `variance` allows unequal paths |
| IS-IS | On by default, usually 4 paths |
| BGP | Off by default (1 path). Enable with `maximum-paths` |

The upper limit depends on the platform. Campus routers often allow 16 or 32. Data centre switches often allow 64 or more. NX-OS and other systems have different defaults, so check yours.

#### Per-flow hashing

A flow is a stream of packets that belong together, such as one TCP connection. The router wants all packets of one flow on one path, because different paths can have different delay. If packets arrive out of order, TCP may think packets were lost and slow down.

So the router uses a hash. A hash is a small maths formula that turns input values into a number. The same input always gives the same number.

```text
   Packet fields                 Hash          Path chosen
  +----------------------+
  | src IP   10.1.1.10   |
  | dst IP   10.50.0.20  |      +-------+      number MOD 2
  | protocol TCP (6)     | ---> | hash  | ---> 0 = path A
  | src port 40000       |      +-------+      1 = path B
  | dst port 443         |
  +----------------------+
```

These five fields are the 5-tuple. Using all five gives a better spread than using only the two IP addresses, because one client with many connections can then use many paths.

Here is an illustration. The hash numbers are made up to show the idea.

| Source | Destination | Src port | Dst port | Hash | MOD 2 | Path |
|---|---|---|---|---|---|---|
| 10.1.1.10 | 10.50.0.20 | 40000 | 443 | 7813 | 1 | B |
| 10.1.1.10 | 10.50.0.20 | 40001 | 443 | 2946 | 0 | A |
| 10.1.1.11 | 10.50.0.20 | 51000 | 443 | 1205 | 1 | B |
| 10.1.1.12 | 10.50.0.21 | 52000 | 53 | 6024 | 0 | A |

#### Limits of ECMP

- **It balances flows, not bytes.** One 10 Gbps backup (an elephant flow) stays on one path. A second path can sit almost empty.
- **Fragments hide ports.** Only the first fragment of an IP packet has the TCP or UDP header. Routers often hash fragments on IP addresses only. See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).
- **Tunnels hide the inner flow.** All traffic in one tunnel can look like one flow. VXLAN handles this by putting a hash of the inner packet in the outer UDP source port. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).
- **A path change can move flows.** When a path fails, flows on that path must move. With simple hashing, some flows on healthy paths also move. Resilient hashing keeps those flows in place. This matters for [anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md) services and [load balancers](../10-enterprise-and-datacenter/04-load-balancing.md).

Cisco IOS with CEF (Cisco Express Forwarding, the fast forwarding engine) hashes on source and destination IP by default, with a per-router ID mixed in. To add ports:

```text
! Cisco IOS: include Layer 4 ports in the hash
ip cef load-sharing algorithm include-ports source destination
```

The command and its options vary by platform.

#### Hash polarisation

Polarisation means that later routers cannot spread traffic because an earlier router has already sorted it.

```text
                 hash result 0        +-----+  all hash 0 again
              +---------------------->| R2  |-----> link 1 busy
  +-----+     |                       +-----+-----> link 2 empty
  | R1  |-----+
  +-----+     |  hash result 1        +-----+  all hash 1 again
              +---------------------->| R3  |-----> link 1 empty
                                      +-----+-----> link 2 busy
```

R1 sends all "even" flows to R2. If R2 uses the same formula with the same inputs, every flow gives an even result again, so all of them use one link. The fix is to make each router's hash different, with a per-router seed or ID, or with different hash fields. Modern gear usually does this by default. It is still a common interview topic.

#### Unequal-cost paths

ECMP needs equal cost. There are a few ways around this:

- **EIGRP `variance`:** `variance 2` installs paths with a metric less than 2 times the best, if they pass the feasibility check. Traffic shares follow the metrics. See [EIGRP](07-eigrp.md).
- **BGP link bandwidth:** a BGP community can carry link speed so that paths get weighted shares. Support varies by vendor.
- **Linux `weight`:** each next hop can have a weight.

OSPF and IS-IS have no built-in unequal-cost load sharing. Same-speed uplinks are the normal design. [Spine-leaf networks](../10-enterprise-and-datacenter/02-spine-leaf.md) are built this way on purpose, so that ECMP has many equal paths to use.

#### ECMP on a Linux host or router

```bash
# Linux: two equal next hops for one prefix
sudo ip route add 10.50.0.0/24 \
    nexthop via 10.0.12.2 dev eth1 weight 1 \
    nexthop via 10.0.13.2 dev eth2 weight 1

# Hash on the 5-tuple instead of only the IP addresses
sudo sysctl -w net.ipv4.fib_multipath_hash_policy=1

# Show the route
ip route show 10.50.0.0/24
```

```text
10.50.0.0/24
	nexthop via 10.0.12.2 dev eth1 weight 1
	nexthop via 10.0.13.2 dev eth2 weight 1
```

The hash policy values are 0 (Layer 3), 1 (Layer 4), 2 (Layer 3, using inner headers when present) and 3 (custom fields, on newer kernels).

On Windows you can read the table, but a normal Windows host is not used as a multipath router:

```powershell
# Windows: list routes for one prefix
Get-NetRoute -DestinationPrefix "10.50.0.0/24"
```

### 2. Route redistribution

Redistribution means copying routes from one routing source into another. The source can be a protocol, static routes, or connected networks.

Common reasons:

- A merger joins two networks that use different protocols.
- A migration moves slowly from one protocol to another.
- A static route on one router must be known by the whole OSPF domain.
- Connected networks must be advertised without running the protocol on them.
- Customer or data centre routes must move between BGP and an IGP (Interior Gateway Protocol, such as OSPF).

#### Three rules that explain most of it

1. **Redistribution happens on one router, called the border router.** It is configured under the receiving protocol. In OSPF this router is called an ASBR (Autonomous System Boundary Router).
2. **Only routes in the routing table get redistributed**, along with connected networks that the source protocol runs on. If a RIP route lost to a better route and is not in the table, it is not copied.
3. **Metrics do not translate.** A RIP hop count means nothing to OSPF. The new protocol needs a starting metric, called the seed metric.

Cisco IOS seed metric defaults:

| Into | Default seed metric | What it means for you |
|---|---|---|
| RIP | Infinity for routes from another protocol | Routes do not appear until you set a metric |
| EIGRP | Infinity for routes from another protocol | Routes do not appear until you set a metric |
| OSPF | 20, type E2 (1 for BGP routes) | Works without setting anything. E2 means the metric stays fixed across the OSPF domain |
| IS-IS | 0, into Level 2 | Works, but set a real metric |
| BGP | MED copied from the IGP metric | Origin code becomes incomplete (`?`) |

MED (Multi-Exit Discriminator) is a BGP path attribute, explained in [BGP](09-bgp.md).

Another Cisco detail: `redistribute` into OSPF needs the `subnets` keyword on older IOS, or only classful networks are copied. Newer versions add it for you. Type it anyway.

#### Why redistribution causes loops

Each routing protocol has its own loop protection. OSPF has a shared map. BGP has AS paths. RIP has hop counts. When a route crosses a border, that history is gone. The new protocol treats it as a fresh route.

With one border router, this is usually safe. With two border routers and redistribution in both directions, routes can return to where they started. This return trip is called route feedback.

```text
          +------+
    +-----+  R1  +-----+       R1 and R2 both redistribute
    |     +------+     |       RIP <-> OSPF
    |                  |
 +--+---+           +--+---+
 |  R3  |           |  R4  |
 +--+---+           +--+---+
    |                  |
    |     +------+     |
    +-----+  R2  +-----+
          +------+
  RIP side         OSPF side
  192.168.50.0/24
  is on R3
```

Follow one route:

1. R3 advertises 192.168.50.0/24 by RIP to R1 and R2.
2. R1 redistributes it into OSPF as an external route.
3. R2 now hears it twice: RIP with AD 120, and OSPF with AD 110. OSPF wins, so R2 removes the RIP route. **Problem 1:** R2 is one hop from R3, but its traffic now goes through R4, R1 and R3. This is a suboptimal path.
4. R2 redistributes OSPF into RIP, so it advertises 192.168.50.0/24 back to R3.
5. R3 ignores this while its LAN is up, because a connected route wins.
6. Now R3's LAN fails. R3 hears the route from R2 and believes it. R3 tells R1.
7. R1 still has a RIP route, so it keeps the route in OSPF. R2 still has the OSPF route, so it keeps it in RIP.

**Problem 2:** the route never dies. Packets go R3, R2, R4, R1, R3 and around, until TTL (Time To Live, a hop limit in each packet) reaches zero. This is a routing loop with a ghost route.

#### How to stop it

| Tool | How it helps |
|---|---|
| Route tags | Mark a route when it enters a domain. Refuse tagged routes on the way back. This is the classic fix |
| Prefix lists | Allow only known prefixes across the border |
| AD changes | Make the native route win, so R2 keeps its direct RIP path |
| One-way redistribution | Send specifics one way, send only a default route the other way |
| Single border router | No feedback path, but it is a single point of failure |

A route tag is a number attached to a route. It does not change forwarding. It is a sticky note for the next router to read. OSPF external routes, EIGRP external routes, RIPv2, and IS-IS (with wide metrics) can carry tags. Example 2 below shows the complete fix.

### 3. Prefix lists

A prefix list matches routes. It looks at two things: the network bits and the prefix length. An ACL (Access Control List) with a wildcard mask mostly looks at address bits, so it cannot say "only /24 or shorter" in a clean way. See [Access Control Lists](../09-security/05-access-control-lists.md).

```text
ip prefix-list NAME seq 10 permit 10.0.0.0/8 ge 16 le 24
               |        |     |       |         |     |
               name     |     action  |         |     longest allowed
                        order         network   shortest allowed
```

Rules:

1. Entries are checked in sequence order. The first match decides.
2. If you do not type `seq`, numbers go up by 5.
3. If nothing matches, the route is denied. This is the implicit deny.
4. Without `ge` or `le`, the length must match exactly.
5. `ge` means "greater than or equal". `le` means "less than or equal". They apply to the prefix length.
6. The values must follow this order: the prefix length is less than `ge`, `ge` is not more than `le`, and `le` is not more than 32.

| Entry | Matches | Does not match |
|---|---|---|
| `10.0.0.0/8` | Only 10.0.0.0/8 | 10.1.0.0/16 |
| `10.0.0.0/8 le 32` | 10.0.0.0/8 and everything inside it | 11.0.0.0/8 |
| `10.0.0.0/8 ge 24` | Everything inside 10/8 from /24 to /32 | 10.1.0.0/16 |
| `10.0.0.0/8 ge 16 le 24` | Inside 10/8, length /16 to /24 | 10.1.1.0/25 |
| `10.0.0.0/8 ge 24 le 24` | Only /24s inside 10/8 | 10.1.0.0/23 |
| `0.0.0.0/0` | Only the default route | Everything else |
| `0.0.0.0/0 le 32` | Every IPv4 route | Nothing |
| `0.0.0.0/0 ge 32` | Every /32 host route | 10.1.1.0/24 |

Note the trap: `10.0.0.0/8` does not mean "all of 10.x". It means exactly that one route.

For IPv6 use `ipv6 prefix-list`, with the same logic and lengths up to /128.

### 4. Route maps

A route map is a small program of ordered clauses. Each clause has a number, an action, conditions (`match`) and changes (`set`).

```text
route-map SAMPLE permit 10
 match ip address prefix-list CUSTOMER-A
 set metric 50
route-map SAMPLE deny 20
 match tag 200
route-map SAMPLE permit 30
```

How a route moves through it:

```text
  route arrives
       |
       v
  +-----------+   match    +------------------------------+
  | clause 10 |----------->| permit: apply set, accept    |
  +-----------+            | deny:   drop, stop           |
       | no match          +------------------------------+
       v
  +-----------+   match
  | clause 20 |----------->  same as above
  +-----------+
       | no match
       v
  implicit deny at the end: route is dropped
```

Key rules:

- Clauses run in number order. The first matching clause ends the check.
- A clause with no `match` line matches everything. This is how you write "permit the rest" (clause 30 above).
- A clause with no `set` line accepts or drops the route without changing it.
- Different match types are AND. `match tag 100` and `match ip address prefix-list X` in one clause must both be true.
- Several values in one match command are OR. `match tag 100 200` means tag 100 or tag 200.
- Every route map ends with an implicit deny.
- `continue` lets a route go on to a later clause after a match. It is mainly used in BGP route maps.

Route maps are used in redistribution, BGP neighbour policy, PBR, and conditional default routes.

#### The permit and deny trap

A prefix list inside a route map has its own permit and deny. They answer different questions. The prefix list asks "does this route match?" The route map clause asks "what do we do with a match?"

| Prefix list says | Route map clause | Result |
|---|---|---|
| permit | permit | Route is accepted, `set` lines run |
| permit | deny | Route is dropped |
| deny or no entry | permit | No match. Go to the next clause |
| deny or no entry | deny | No match. Go to the next clause |

So a `deny` in a prefix list does not drop a route. It only means "this clause does not apply".

#### Short BGP example

```text
! Cisco IOS: accept only the customer's prefix, and prefer it
ip prefix-list CUSTOMER-A seq 10 permit 203.0.113.0/24
!
route-map FROM-CUSTOMER-A permit 10
 match ip address prefix-list CUSTOMER-A
 set local-preference 200
!
router bgp 64512
 neighbor 192.0.2.1 remote-as 64513
 neighbor 192.0.2.1 route-map FROM-CUSTOMER-A in
```

Anything other than 203.0.113.0/24 hits the implicit deny and is rejected. This is correct for a customer session, because you do not want them to announce someone else's prefix. Local preference is covered in [BGP](09-bgp.md).

### 5. Policy-based routing

Normal forwarding looks only at the destination address. PBR lets a router choose the next hop from other facts: source address, protocol, port, packet size or a QoS mark (a priority label in the packet header). See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

The word "policy" here means "a rule a human wrote", not a routing protocol feature. PBR does not change the routing table. It runs before the normal lookup.

```text
  packet enters an interface that has "ip policy route-map X"
       |
       v
  +-------------------------+   no match   +------------------+
  | Does it match a permit  |------------->| Normal routing   |
  | clause in route map X?  |              | (routing table)  |
  +-------------------------+              +------------------+
       | yes                                        ^
       v                                            |
  +-------------------------+   next hop down       |
  | Use "set" next hop      |-----------------------+
  +-------------------------+
       | next hop usable
       v
  packet sent out by policy
```

Key facts for Cisco IOS:

- PBR applies to packets **coming in** on an interface. It does not apply to packets going out of that interface.
- Packets that the router itself creates need a separate command: `ip local policy route-map X`.
- A packet that matches no clause is routed normally. It is not dropped. PBR's implicit deny means "do not policy route".
- A packet matching a `deny` clause is also routed normally.
- `set ip next-hop` is used first, before the routing table.
- `set ip default next-hop` is used only if the routing table has no explicit route for the destination. A default route does not count as explicit.
- If the next hop is not usable, the router falls back to normal routing. By default it only makes simple local checks, so a dead path further away goes unnoticed. Use `verify-availability` with object tracking for real checks.

PBR matches packets, so it normally uses an ACL. Prefix lists are for routes.

Common uses:

- Send guest traffic out through a second internet link.
- Send voice over one WAN link and backups over another.
- Send web traffic through a proxy or inspection device.

Risks:

- `show ip route` no longer tells the full truth. A path looks correct but PBR sends packets elsewhere.
- Return traffic can use a different path. A stateful firewall, which tracks connections, may drop it. See [Firewalls](../09-security/04-firewalls.md).
- Each ISP needs its own NAT. Traffic leaving by ISP B must use ISP B's addresses. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).
- On hardware switches, PBR uses limited special memory called TCAM. Large policies can run out.
- It is manual. Too much PBR turns a network into a list of exceptions.

#### PBR on Linux

Linux has the same idea with several routing tables and rules that choose among them.

```bash
# Linux: guests (10.20.0.0/24) use table 100, which points to ISP B
sudo ip route add default via 203.0.113.1 dev eth2 table 100
sudo ip rule add from 10.20.0.0/24 lookup 100 priority 1000
ip rule show
```

```text
0:	from all lookup local
1000:	from 10.20.0.0/24 lookup 100
32766:	from all lookup main
32767:	from all lookup default
```

Rules are checked by priority number, lowest first. For rules on ports or firewall marks, use `fwmark`, covered in [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

A normal Windows host routes by destination. It has no simple match for `ip rule`.

### 6. VRF and VRF-lite

A VRF is a separate routing table and forwarding table inside one router. A router with three VRFs behaves like three routers for Layer 3.

```text
 +----------------------------------------------------+
 |                      Router R1                     |
 |                                                    |
 |  +----------------+  +----------------+            |
 |  | VRF RED        |  | VRF BLUE       |            |
 |  | RIB and FIB    |  | RIB and FIB    |            |
 |  | 10.1.1.0/24    |  | 10.1.1.0/24    |            |
 |  +---+--------+---+  +---+--------+---+            |
 |      |        |          |        |                |
 +------|--------|----------|--------|----------------+
     Gi0/0    Gi0/2.10    Gi0/1    Gi0/2.20
```

Rules:

1. An interface or subinterface belongs to one VRF only. A subinterface is a logical part of a physical port, usually one per VLAN. An interface with no VRF is in the global table, which is the router's normal routing table.
2. Each VRF has its own routes, ARP entries and forwarding table.
3. Two VRFs can use the same addresses, because they never look in each other's table.
4. Traffic cannot cross VRFs unless you configure route leaking, or send it through a router or firewall.

VRF compared with VLAN: a [VLAN](../03-ethernet-and-switching/06-vlans.md) separates Layer 2 (frames on a switch). A VRF separates Layer 3 (routing). They are often used together, with one VLAN mapped to one VRF on each link.

#### VRF-lite versus MPLS L3VPN

VRF-lite means VRFs with no MPLS (Multiprotocol Label Switching). It uses only the VRF feature.

```text
 +-----+  VLAN 10 (RED)  +-----+  VLAN 10 (RED)  +-----+
 | R1  |=================| R2  |=================| R3  |
 |     |  VLAN 20 (BLUE) |     |  VLAN 20 (BLUE) |     |
 +-----+                 +-----+                 +-----+
   every hop has each VRF, a subinterface per VRF per link,
   and a routing process per VRF
```

| Item | VRF-lite | MPLS L3VPN |
|---|---|---|
| Who keeps VRFs | Every router on the path | Only edge routers, called PE (Provider Edge) |
| Transport | One VLAN per VRF per link | Label switched core |
| Routing | IGP or BGP per VRF on every hop | MP-BGP (BGP extended to carry VPN routes) between edge routers |
| Fits | Few VRFs and few hops | Many customers and many sites |
| Standard | Local feature | RFC 4364 |

The two terms below come from the L3VPN world:

- **RD (Route Distinguisher):** a value added to a prefix inside BGP, so 10.1.1.0/24 from two customers stays unique. It does not control which routes go where.
- **RT (Route Target):** a BGP community that says which VRFs should import a route. It controls where routes go.

Full detail is in [MPLS](13-mpls.md). The same VRF idea is used in [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) fabrics.

Common uses of VRF:

- Separate customers on one router.
- Separate a management network from user traffic. Many devices ship with a management VRF, such as `management` on NX-OS and `Mgmt-intf` on some IOS XE platforms.
- Separate guest, staff and partner traffic in a campus.
- Keep internet routes away from internal routes.

#### VRF on Linux

```bash
# Linux: create a VRF that uses table 10, then move eth1 into it
sudo ip link add vrf-red type vrf table 10
sudo ip link set dev vrf-red up
sudo ip link set dev eth1 master vrf-red

# Show routes and run a command in the VRF
ip route show vrf vrf-red
sudo ip vrf exec vrf-red ping -c 2 10.2.2.10
```

Fuller truth: a Linux VRF is only a Layer 3 split. A network namespace splits the whole network stack, which is why containers use namespaces. See [Linux Networking](../11-cloud-and-containers/01-linux-networking.md).

### 7. Summarisation in routing protocols

Summarisation replaces many specific routes with one shorter route. The maths is in [VLSM, Route Summarisation and Supernetting](../04-ip-addressing/07-vlsm-and-supernetting.md). This section is about how protocols do it.

Benefits:

- Smaller routing tables and less memory.
- Fewer update messages.
- **Hidden flapping.** If 10.10.2.0/24 goes up and down (flaps), routers that know only 10.10.0.0/22 see no change. In OSPF, this means less SPF work. SPF (Shortest Path First) is the map calculation.
- In EIGRP, a summary limits how far queries travel.

Costs:

- **Less exact paths.** Traffic may take a longer path, because routers no longer see the best exit for each subnet.
- **Black holes.** A router can advertise a summary while one piece of it is unreachable.

Where each protocol can summarise:

| Protocol | Where | Cisco IOS command |
|---|---|---|
| OSPF | Only at ABR (between areas) | `area 1 range 10.10.0.0 255.255.252.0` |
| OSPF | Only at ASBR (external routes) | `summary-address 192.168.0.0 255.255.0.0` |
| IS-IS | At Level 1/2 routers and redistribution points | `summary-address 10.10.0.0 255.255.252.0 level-2` |
| EIGRP | Any interface | `ip summary-address eigrp 100 10.10.0.0 255.255.252.0` |
| RIP | Any interface | `ip summary-address rip 10.10.0.0 255.255.252.0` |
| BGP | Any BGP router, with a component route in BGP | `aggregate-address 10.10.0.0 255.255.252.0 summary-only` |

ABR means Area Border Router. OSPF cannot summarise inside an area, because all routers in an area must hold the same map. EIGRP and BGP do not share a map, so they are more free.

#### The discard route

When a router creates a summary, it usually adds a route to Null0. Null0 is a fake interface that drops packets. This looks strange, but it stops a real loop.

```text
 Packet to 10.10.3.5. This subnet does not exist.

 Without a discard route:

  Internet router                     R1
  "10.10.0.0/22 via R1"               has 10.10.0.0/24 to .2.0/24
  +--------+      packet              default route to Internet
  |        |------------------------> +--------+
  |  ISP   |                          |   R1   |
  |        |<------------------------ +--------+
  +--------+      packet              no match, uses default
      loop until TTL expires

 With a discard route on R1:

  10.10.0.0/22 via Null0
  Longest match for 10.10.3.5 is the /22, so R1 drops it.
```

The longer real routes (/24) still win for real subnets. Only unused addresses fall through to Null0. No loop, and the router can send ICMP unreachable. See [ICMP, Ping and Traceroute](10-icmp-ping-and-traceroute.md).

#### Summary black holes

Two ABRs both advertise 10.10.0.0/22. ABR1 loses its path to 10.10.1.0/24, but it still has other /24s, so it keeps advertising the summary. Backbone routers keep sending some traffic for 10.10.1.0/24 to ABR1, and ABR1 drops it at Null0. The cure is good design: keep paths inside the area so that ABR1 can still reach all parts, and never split an area.

### 8. BFD: fast failure detection

#### The problem

If a cable to a neighbour is pulled, the local interface goes down and the protocol reacts at once. But many links pass through a switch, a provider circuit or another device. The neighbour can die while the local interface stays up.

```text
 +----+   port stays UP   +--------+   this side dies   +----+
 | R1 |-------------------| switch |--------X-----------| R2 |
 +----+                   +--------+                    +----+
  R1 sees link up. Only missing hellos can tell it R2 is gone.
```

Protocol timers are slow by default:

| Protocol | Hello or keepalive | Dead or hold time | Note |
|---|---|---|---|
| OSPF | 10 s | 40 s | Broadcast and point-to-point networks |
| EIGRP | 5 s | 15 s | High-speed LAN links |
| IS-IS | 10 s | 30 s | Cisco default, multiplier 3 |
| BGP | 60 s | 180 s | Cisco default. RFC 4271 suggests 90 s hold |

Lowering protocol timers is one answer, but it loads the control plane (the part of the router that runs protocols on its CPU), and each protocol needs its own tuning. BFD (Bidirectional Forwarding Detection, RFC 5880) is one light detector that all protocols can share.

#### What BFD does

1. A protocol, such as OSPF, forms a neighbour. It tells BFD "please watch 10.0.12.2".
2. BFD starts a session with that neighbour. BFD needs a configured client: it does not discover anyone itself.
3. Both sides move through the states Down, Init and Up.
4. Both sides send small packets at a short interval.
5. If packets stop for the detection time, BFD declares the session Down.
6. BFD tells every client protocol. OSPF drops the neighbour at once and recalculates.

BFD only detects. The routing protocol still does the repair work.

The name says "bidirectional" because it proves the forwarding path works in both directions.

#### Session states

```text
 R1 state     packet sent        R2 state
 --------     -----------        --------
 Down         State=Down  --->   Down -> Init
 Down -> Up   <--- State=Init    Init
 Up           State=Up    --->   Init -> Up
 Up           <--> regular packets  <--> Up
```

The fourth state, AdminDown, means an operator turned it off.

#### Timers and detection time

Each side announces three values:

- **Desired Min TX:** the shortest gap at which I want to send.
- **Required Min RX:** the shortest gap at which I can receive.
- **Detect Mult:** how many packets may be missed.

The actual send rate in one direction is the slower of the sender's Desired Min TX and the receiver's Required Min RX. Detection time is that interval multiplied by the remote side's Detect Mult.

| Interval | Multiplier | Detection time |
|---|---|---|
| 300 ms | 3 | 900 ms |
| 100 ms | 3 | 300 ms |
| 50 ms | 3 | 150 ms |

Example of negotiation: R1 wants to send every 100 ms. R2 says it can only receive every 300 ms. R1 must send every 300 ms. With multiplier 3 on R1, R2 waits 900 ms.

Fuller truth: real send times have small random jitter so that sessions do not sync up, and the sessions use a slow rate of at least one second until they are Up.

#### Modes

- **Asynchronous mode:** both sides send control packets all the time. This is the normal mode.
- **Demand mode:** after the session is up, packets stop, and each side checks only when needed. It is rarely used.
- **Echo function:** a router sends special packets that the neighbour forwards straight back, without its CPU reading them. This tests the real forwarding path and lets control packets run slower. Cisco enables it by default when both sides support it, and it can be turned off with `no bfd echo`.

#### Packet details

BFD runs over UDP. For single-hop sessions (RFC 5881) the destination port is 3784 and TTL must be 255. Echo uses port 3785. Multi-hop sessions (RFC 5883) use port 4784. BFD on each member of a link bundle (RFC 7130) uses port 6784.

```text
 BFD control packet (24 bytes without authentication)

  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-----+---------+---+-----------+---------------+---------------+
 | Ver |  Diag   |Sta|P F C A D M|  Detect Mult  |    Length     |
 +-----+---------+---+-----------+---------------+---------------+
 |                       My Discriminator                        |
 +---------------------------------------------------------------+
 |                      Your Discriminator                       |
 +---------------------------------------------------------------+
 |                    Desired Min TX Interval                    |
 +---------------------------------------------------------------+
 |                   Required Min RX Interval                    |
 +---------------------------------------------------------------+
 |                 Required Min Echo RX Interval                 |
 +---------------------------------------------------------------+
```

| Field | Size | Meaning |
|---|---|---|
| Ver | 3 bits | Version, 1 |
| Diag | 5 bits | Reason for the last state change |
| Sta | 2 bits | State: AdminDown, Down, Init or Up |
| Flags | 6 bits | Poll, Final, and other control flags |
| Detect Mult | 8 bits | Missed packets allowed |
| Length | 8 bits | Packet length in bytes |
| My Discriminator | 32 bits | My unique ID for this session |
| Your Discriminator | 32 bits | The neighbour's ID, 0 until known |
| Intervals | 32 bits each | Values in microseconds |

Discriminators let several BFD sessions run between the same two routers.

#### Configuration

```text
! Cisco IOS: timers on the interface, then attach clients
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3
!
router ospf 1
 bfd all-interfaces
!
router bgp 64512
 neighbor 10.0.12.2 remote-as 64513
 neighbor 10.0.12.2 fall-over bfd
!
! Static route that follows a BFD session
ip route static bfd GigabitEthernet0/1 10.0.12.2
ip route 10.50.0.0 255.255.255.0 GigabitEthernet0/1 10.0.12.2
```

On Linux, FRRouting (an open source routing suite) offers BFD. In its `vtysh` shell:

```text
! FRRouting
bfd
 peer 10.0.12.2
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
  no shutdown
```

## Example

### Example 1: ECMP with BFD

```text
                     +------+
          +----------|  R2  |----------+
          |          +------+          |
        Gi0/1                          |
          |                            |
       +------+                     +------+
  -----|  R1  |                     |  R4  |-----
       +------+                     +------+
          |                            |
        Gi0/2                          |
          |          +------+          |
          +----------|  R3  |----------+
                     +------+

  R1 LAN:   10.1.1.0/24
  R4 LAN:   10.50.0.0/24
  R1-R2:    10.0.12.0/30 (R1 .1, R2 .2)
  R1-R3:    10.0.13.0/30 (R1 .1, R3 .2)
  R2-R4:    10.0.24.0/30 (R2 .1, R4 .2)
  R3-R4:    10.0.34.0/30 (R3 .1, R4 .2)
  Router IDs: 10.255.0.2 (R2), 10.255.0.4 (R4)
```

All links are Gigabit Ethernet with OSPF cost 1. The R1 to R2 link goes through a switch (not drawn). The R4 LAN also has cost 1.

R1 reaches 10.50.0.0/24 with a total cost of 1 + 1 + 1 = 3 on both sides. It installs both paths.

```text
R1# show ip route 10.50.0.0
Routing entry for 10.50.0.0/24
  Known via "ospf 1", distance 110, metric 3, type intra area
  Last update from 10.0.13.2 on GigabitEthernet0/2, 00:04:10 ago
  Routing Descriptor Blocks:
  * 10.0.12.2, from 10.255.0.4, 00:04:10 ago, via GigabitEthernet0/1
      Route metric is 3, traffic share count is 1
    10.0.13.2, from 10.255.0.4, 00:04:10 ago, via GigabitEthernet0/2
      Route metric is 3, traffic share count is 1
```

See which path one flow will use:

```text
R1# show ip cef exact-route 10.1.1.10 10.50.0.20
10.1.1.10 -> 10.50.0.20 => IP adj out of GigabitEthernet0/1, addr 10.0.12.2
```

Add BFD on R1. R2 needs the same commands on its link to R1:

```text
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3
router ospf 1
 bfd all-interfaces
```

```text
R1# show bfd neighbors

IPv4 Sessions
NeighAddr              LD/RD         RH/RS     State     Int
10.0.12.2              1/1           Up        Up        Gi0/1
```

Now R2 loses power. R1's port stays up, because the switch is still alive. Within about 900 ms:

```text
%BFDFSM-6-BFD_SESS_DOWN: BFD-SYSLOG: BFD session ld:1 handle:1,is going Down Reason: ECHO FAILURE
%OSPF-5-ADJCHG: Process 1, Nbr 10.255.0.2 on GigabitEthernet0/1 from FULL to DOWN, Neighbor Down: BFD node down
```

The route now has one path, through 10.0.13.2. Without BFD, R1 would keep sending half of the flows into a dead path for up to 40 seconds.

### Example 2: Safe two-way redistribution

Use the RIP and OSPF topology from earlier, with these addresses:

| Router | Interface | Address | Domain |
|---|---|---|---|
| R1 | Gi0/1 | 10.0.13.1/30 to R3 | RIP |
| R1 | Gi0/2 | 10.0.14.1/30 to R4 | OSPF |
| R2 | Gi0/1 | 10.0.23.1/30 to R3 | RIP |
| R2 | Gi0/2 | 10.0.24.1/30 to R4 | OSPF |
| R3 | LAN | 192.168.50.1/24 | RIP |
| R4 | LAN | 10.40.0.1/24 | OSPF |

Plan:

- Routes born in RIP get tag 100 when they enter OSPF.
- Routes born in OSPF get tag 200 when they enter RIP.
- Tagged routes are refused on the way back.
- A prefix list allows only the RIP LANs.
- External OSPF routes get AD 130 on the border routers, so that native RIP (120) wins.

Put the same config on R1 and R2:

```text
! Cisco IOS on R1 and R2
ip prefix-list RIP-LANS seq 10 permit 192.168.0.0/16 le 24
!
route-map RIP-TO-OSPF deny 10
 match tag 200
route-map RIP-TO-OSPF permit 20
 match ip address prefix-list RIP-LANS
 set metric 50
 set metric-type type-1
 set tag 100
!
route-map OSPF-TO-RIP deny 10
 match tag 100
route-map OSPF-TO-RIP permit 20
 set metric 3
 set tag 200
!
router ospf 1
 redistribute rip subnets route-map RIP-TO-OSPF
 distance ospf external 130
!
router rip
 version 2
 no auto-summary
 redistribute ospf 1 route-map OSPF-TO-RIP
```

Line by line:

- Clause `deny 10` in `RIP-TO-OSPF`: if a route has tag 200, it was born in OSPF. Do not send it back.
- Clause `permit 20`: other routes that match the list enter OSPF as type E1 (a type that adds internal cost to the metric), with metric 50 and tag 100. Link addresses such as 10.0.13.0/30 do not match the list, so they stay in RIP.
- `OSPF-TO-RIP` is the mirror. Note that no prefix list is needed here for this small lab, but a real network should have one.
- `distance ospf external 130` is local to each border router. It does not travel to other routers.

Check R2. The direct RIP path is back in use:

```text
R2# show ip route 192.168.50.0
Routing entry for 192.168.50.0/24
  Known via "rip", distance 120, metric 1
  Redistributing via ospf 1, rip
  Advertised by ospf 1 subnets route-map RIP-TO-OSPF
  Last update from 10.0.23.2 on GigabitEthernet0/1, 00:00:12 ago
  Routing Descriptor Blocks:
  * 10.0.23.2, from 10.0.23.2, 00:00:12 ago, via GigabitEthernet0/1
      Route metric is 1, traffic share count is 1
```

Check R4. Both borders now advertise the route, so R4 has ECMP. The metric is 50 plus the cost of 1 to each border:

```text
R4# show ip route 192.168.50.0
Routing entry for 192.168.50.0/24
  Known via "ospf 1", distance 110, metric 51
  Tag 100, type extern 1
  Last update from 10.0.24.1 on GigabitEthernet0/2, 00:02:05 ago
  Routing Descriptor Blocks:
    10.0.14.1, from 10.255.0.1, 00:02:05 ago, via GigabitEthernet0/1
      Route metric is 51, traffic share count is 1
      Route tag 100
  * 10.0.24.1, from 10.255.0.2, 00:02:05 ago, via GigabitEthernet0/2
      Route metric is 51, traffic share count is 1
      Route tag 100
```

Now test the failure that caused the ghost route. R3's LAN goes down:

1. R3 stops advertising 192.168.50.0/24.
2. R1 and R2 lose the RIP route. Each may briefly see the other's OSPF route.
3. `OSPF-TO-RIP` denies tag 100, so neither router sends the route back into RIP.
4. Each removes its external route from OSPF. The route is gone everywhere.

Check that prefix lists are getting hits:

```text
R1# show ip prefix-list detail RIP-LANS
Prefix-list with the last deletion/insertion: RIP-LANS
ip prefix-list RIP-LANS:
   count: 1, range entries: 1, sequences: 10 - 10, refcount: 2
   seq 10 permit 192.168.0.0/16 le 24 (hit count: 4, refcount: 1)
```

### Example 3: Guest traffic by policy

```text
                               +-------+  198.51.100.1
                       Gi0/1   |       |
                    +----------+ ISP A |
 Staff 10.10.0.0/24 |          +-------+
 Guest 10.20.0.0/24 |
    +-----+  Gi0/0  +------+
    | LAN +---------+  R1  |
    +-----+         +------+
                    |          +-------+  203.0.113.1
                    +----------+ ISP B |
                       Gi0/2   |       |
                               +-------+
 R1 addresses: 198.51.100.2/30 and 203.0.113.2/30
```

The default route on R1 points to ISP A, 198.51.100.1. Guests must use ISP B, but guests may still reach internal printers on 10.x in the normal way.

```text
! Cisco IOS on R1
ip access-list extended GUEST-NET
 deny   ip 10.20.0.0 0.0.0.255 10.0.0.0 0.255.255.255
 permit ip 10.20.0.0 0.0.0.255 any
!
ip sla 1
 icmp-echo 203.0.113.1 source-interface GigabitEthernet0/2
 frequency 5
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
!
route-map GUEST-PBR permit 10
 match ip address GUEST-NET
 set ip next-hop verify-availability 203.0.113.1 10 track 1
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.20.0.1 255.255.255.0
 ip policy route-map GUEST-PBR
```

IP SLA (Service Level Agreement) is a Cisco probe. Here it pings ISP B every 5 seconds. Object tracking turns that result into up or down. If ISP B fails, the route map has no usable next hop, so guests use the normal routing table and leave by ISP A.

The `deny` line in the ACL does not block guest traffic to 10.x. It only means "not matched", so the packet is routed normally.

```text
R1# show ip policy
Interface      Route map
Gi0/0.20       GUEST-PBR

R1# show route-map GUEST-PBR
route-map GUEST-PBR, permit, sequence 10
  Match clauses:
    ip address (access-lists): GUEST-NET
  Set clauses:
    ip next-hop verify-availability 203.0.113.1 10 track 1  [up]
  Policy routing matches: 18342 packets, 2210345 bytes
```

Trace of one guest packet, 10.20.0.50 to 192.0.2.80:

1. It enters Gi0/0.20, so PBR runs.
2. It is not going to 10.x, so the ACL deny does not apply, and the permit line matches.
3. Track 1 is up, so R1 sends it to 203.0.113.1.
4. NAT on R1 changes the source to an ISP B address, so the reply returns through ISP B.

### Example 4: VRF-lite with overlapping addresses

Two tenants use the same addresses. R1 and R2 are joined by one trunk, with one VLAN per VRF.

```text
  RED LAN                                        RED LAN
  10.1.1.0/24                                    10.2.2.0/24
      |                                              |
    Gi0/0                                          Gi0/0
      |                                              |
    +----+  Gi0/2    VLAN 10 (RED)      Gi0/2      +----+
    | R1 |=========================================| R2 |
    +----+           VLAN 20 (BLUE)                +----+
      |                                              |
    Gi0/1                                          Gi0/1
      |                                              |
  BLUE LAN                                       BLUE LAN
  10.1.1.0/24                                    10.2.2.0/24

  Both VLANs use 172.16.0.0/30 (R1 .1, R2 .2)
```

```text
! Cisco IOS on R1. R2 mirrors this, using 172.16.0.2 and 10.2.2.1
vrf definition RED
 address-family ipv4
 exit-address-family
vrf definition BLUE
 address-family ipv4
 exit-address-family
!
interface GigabitEthernet0/0
 vrf forwarding RED
 ip address 10.1.1.1 255.255.255.0
interface GigabitEthernet0/1
 vrf forwarding BLUE
 ip address 10.1.1.1 255.255.255.0
!
interface GigabitEthernet0/2.10
 encapsulation dot1Q 10
 vrf forwarding RED
 ip address 172.16.0.1 255.255.255.252
interface GigabitEthernet0/2.20
 encapsulation dot1Q 20
 vrf forwarding BLUE
 ip address 172.16.0.1 255.255.255.252
!
router ospf 10 vrf RED
 network 10.1.1.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
router ospf 20 vrf BLUE
 network 10.1.1.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
```

The same IP on two interfaces is normally refused. Here it works because each interface is in a different VRF. Note the order: `vrf forwarding` removes any IP address already on the interface, so type it first.

```text
R1# show vrf
  Name          Default RD      Protocols   Interfaces
  BLUE          <not set>       ipv4        Gi0/1
                                            Gi0/2.20
  RED           <not set>       ipv4        Gi0/0
                                            Gi0/2.10

R1# show ip route vrf RED

Routing Table: RED
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C        10.1.1.0/24 is directly connected, GigabitEthernet0/0
L        10.1.1.1/32 is directly connected, GigabitEthernet0/0
O        10.2.2.0/24 [110/2] via 172.16.0.2, 00:01:44, GigabitEthernet0/2.10
      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.16.0.0/30 is directly connected, GigabitEthernet0/2.10
L        172.16.0.1/32 is directly connected, GigabitEthernet0/2.10

R1# ping vrf RED 10.2.2.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.2.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

A plain `ping 10.2.2.10` fails, because it looks in the global table, and that table has no such route.

### Example 5: OSPF summary at an ABR

Area 1 has four LANs: 10.10.0.0/24, 10.10.1.0/24, 10.10.2.0/24 and 10.10.3.0/24. R1 is the ABR. R2 is in area 0.

```text
 Area 1                  ABR                    Area 0
 10.10.0.0/24 --+      +----+  10.0.0.0/30    +----+
 10.10.1.0/24 --+------| R1 |-----------------| R2 |
 10.10.2.0/24 --+      +----+ .1           .2 +----+
 10.10.3.0/24 --+
```

Find the summary. Only the third byte changes:

```text
 10.10.0.0  third byte = 000000 00
 10.10.1.0  third byte = 000000 01
 10.10.2.0  third byte = 000000 10
 10.10.3.0  third byte = 000000 11
                         ^^^^^^
 8 + 8 + 6 = 22 shared bits, so the summary is 10.10.0.0/22
```

Before the change, R2 has four routes marked `O IA` (OSPF inter-area). Then on R1:

```text
router ospf 1
 area 1 range 10.10.0.0 255.255.252.0
```

R2 now has one route:

```text
R2# show ip route ospf
      10.0.0.0/8 is variably subnetted, 3 subnets, 3 masks
O IA     10.10.0.0/22 [110/2] via 10.0.0.1, 00:00:41, GigabitEthernet0/0
```

R1 has the discard route, and it still keeps the four real /24 routes:

```text
R1# show ip route | include 10.10.0.0/22
O        10.10.0.0/22 is a summary, 00:00:45, Null0
```

Now, when 10.10.2.0/24 flaps, R2 sees no change, because the summary stays up as long as at least one LAN in the range is alive.

## Going Deeper

### The full picture of fast convergence

Convergence means all routers have agreed on the new paths. BFD fixes only the first step. In the diagram, LSAs (Link State Advertisements) are the messages OSPF uses to share its map.

```text
  failure --> detect --> tell others --> recalculate --> update FIB
              BFD        flood LSAs      SPF             FIB install
              (ms)       (ms)            (ms)            (ms to s)
```

Each step has its own tuning, including SPF timers that wait a moment for more changes. With many prefixes, the FIB step can be the slowest.

**IP Fast Reroute** skips the wait. A router computes a backup next hop in advance. When the link fails, it switches at once and the protocol catches up later.

- LFA (Loop-Free Alternate, RFC 5286): a neighbour that can reach the destination without sending traffic back. Not all topologies have one.
- Remote LFA (RFC 7490): a tunnel to a farther node that is safe.
- TI-LFA (Topology-Independent LFA): uses segment routing to cover nearly every failure.

**BGP PIC** (Prefix Independent Convergence) is similar for BGP. It shares one next-hop object among many prefixes, so one change repairs many prefixes at once.

### BFD versus graceful restart

Graceful Restart (RFC 3623 for OSPF, RFC 4724 for BGP) lets a router keep forwarding while its control plane restarts. Neighbours keep their routes for a moment. Cisco calls a related feature NSF (Nonstop Forwarding).

The two features pull in opposite directions. BFD says "report failure fast". Graceful restart says "do not panic during a restart". With very low BFD timers, a supervisor switchover (the backup control card taking over on a large router) may look like a failure. Platforms that run BFD in hardware survive this better. Test it before you rely on it.

Very low timers on a busy router can also cause false alarms. If BFD runs on a CPU and the CPU spikes, sessions flap. 300 ms x 3 is a common safe start. Use 50 ms only where hardware offload supports it.

### BGP multipath conditions

`maximum-paths` alone is not enough. On Cisco, paths must tie in weight, local preference, AS path length, origin, MED and IGP metric, and they must have different next hops. The neighbouring AS must also be the same, unless you enable `bgp bestpath as-path multipath-relax`. Even with multipath, BGP advertises only one best path to neighbours. Data centres that run BGP use multipath heavily (RFC 7938).

### AD traps by protocol pair

Default AD on Cisco:

| Source | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| EIGRP summary | 5 |
| eBGP (BGP between different AS numbers) | 20 |
| EIGRP internal | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP external | 170 |
| iBGP (BGP inside one AS) | 200 |

Other vendors differ. Juniper, for example, uses 10 for internal OSPF, 150 for external OSPF and 170 for BGP.

This table explains why some pairs are worse than others:

- **EIGRP with OSPF:** EIGRP internal (90) beats OSPF, so native EIGRP routes are safe. EIGRP external routes (170) lose to OSPF (110), so a route from a third domain can be pulled the wrong way.
- **RIP with OSPF:** RIP (120) always loses to OSPF (110), including OSPF externals. This is the case in Example 2.
- **OSPF has one AD for all route types on many platforms.** That is why IOS gives a separate `distance ospf external` command.

### Redistribution into and out of BGP

- On Cisco, `redistribute ospf 1` into BGP takes only internal OSPF routes by default. To include externals, add `match internal external 1 external 2`.
- Routes learned from iBGP are not redistributed into an IGP by default. The command `bgp redistribute-internal` changes this, and it carries loop risk.
- Never redistribute a full BGP table into an IGP. The IPv4 internet table has about one million prefixes, and an IGP is not built for that. Use a default route instead.
- It is better to use the BGP `network` command for exact prefixes than to redistribute. It gives clearer control.

### Filtering routes in link-state protocols

In distance-vector and path-vector protocols, a router can hide a route from a neighbour with a `distribute-list` or a route map.

Link-state protocols share a map, so this is different. In OSPF, `distribute-list in` only stops a route from entering that router's own routing table. The LSAs are still in its database and still flood on. To really filter, use a boundary:

- `area 1 filter-list prefix NAME in` filters Type 3 LSAs at an ABR.
- `area 1 range ... not-advertise` hides a range at an ABR.
- `summary-address ... not-advertise` hides a range at an ASBR.
- Stub area types block external routes.

### VRF route leaking and shared services

Sometimes VRFs need one shared thing, such as DNS or the internet.

- **Static leak:** a route in one VRF that points to an interface or next hop in another.

```text
! Cisco IOS: RED reaches a shared DNS subnet through the global table
ip route vrf RED 10.99.0.0 255.255.255.0 10.0.99.2 global
```

  The return path needs a matching route. This approach is simple, but it grows messy.

- **BGP import and export:** with route targets, a VRF imports selected routes from another. This scales better.
- **Fusion router or firewall:** each VRF connects to an outside device, which routes between them and applies security. This is common in campus networks, because all crossing traffic gets inspected.

Other details:

- Routing processes must be VRF aware. In BGP, use `address-family ipv4 vrf RED`.
- When OSPF runs in a VRF on an edge router, IOS applies some provider edge checks. `capability vrf-lite` turns these off for VRF-lite designs.
- Services are also per VRF: NAT, DHCP relay, SNMP traps, syslog and SSH source all need VRF settings.
- The older syntax is `ip vrf RED` with `ip vrf forwarding RED`. The newer `vrf definition` form supports IPv6.
- Each VRF uses memory for its own tables. Thousands of VRFs need a platform built for it.

### More about summaries

- **OSPF summary cost:** the summary takes one metric from its component routes. RFC 2328 uses the highest component cost. Older behaviour used the lowest, so results vary with vendor and compatibility settings.
- **EIGRP summary metric:** the lowest metric of the components. If that component flaps, the summary metric can change and updates can spread. A fixed metric can stop this on platforms that support it.
- **BGP aggregate:** `aggregate-address` needs at least one component route in the BGP table. `summary-only` hides the specifics. `as-set` keeps the AS numbers of the components, which keeps loop protection but can make the aggregate change when a component changes. Internet providers usually accept IPv4 prefixes only up to /24 and IPv6 up to /48, so do not split or summarise blindly.
- **Default route:** the widest summary of all. Stub networks often need nothing else.
- **IPv6:** summaries work in the same way, for example `area 1 range 2001:db8:10::/46` in OSPFv3.

### uRPF and asymmetric paths

uRPF (Unicast Reverse Path Forwarding, RFC 3704) drops packets if the source address would not route back through the arriving interface. It stops spoofing. Strict mode can break designs with ECMP, PBR or other asymmetric paths. Loose mode, or strict mode that considers all paths, avoids this. Always check this before you add PBR at a network edge.

### Traceroute on ECMP paths

Classic traceroute changes the UDP port in each probe. On an ECMP path, this changes the hash, so one traceroute can show hops from different paths and draw a route that does not exist. Paris traceroute keeps the flow fields fixed. See [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

### Choosing the right tool

| Need | Use | Do not use |
|---|---|---|
| Two equal links | ECMP or [EtherChannel](../03-ethernet-and-switching/10-link-aggregation.md) | PBR |
| Reject some routes | Prefix list with route map | Standard ACL for new designs |
| Change path for an app | PBR, or [SD-WAN](../10-enterprise-and-datacenter/10-sd-wan.md) | Route redistribution |
| Separate tenants | VRF with VLAN | ACLs alone |
| Join two IGPs for a short time | Controlled redistribution | Redistribution with no tags |
| Faster failure detection | BFD | Very low OSPF timers on many links |
| Smaller tables | Summarisation | Filtering that breaks reachability |

## Common Mistakes

- **Thinking ECMP splits bytes evenly.** It splits flows by a hash. One big flow can fill one link.
- **Using per-packet load sharing to "fix" balance.** It reorders packets and hurts TCP. Per-flow is the normal choice.
- **Forgetting BGP has ECMP off.** Two equal eBGP paths give one active path until you set `maximum-paths`.
- **Assuming OSPF does unequal load sharing.** Only EIGRP has `variance`.
- **Redistributing without setting a metric into RIP or EIGRP.** Routes silently do not appear, because the seed metric is infinity.
- **Forgetting `subnets` on older IOS.** Only classful networks get into OSPF.
- **Two-way redistribution at two points with no tags.** This is how ghost routes begin.
- **Believing that a tag stops a loop by itself.** A tag does nothing unless a route map checks it.
- **Reading `10.0.0.0/8` in a prefix list as "all of 10.x".** It matches one route. Add `le 32` for everything inside.
- **Mixing up `ge` and `le`.** `ge` sets the shortest allowed length and `le` sets the longest.
- **Thinking a prefix list deny drops the route in a route map.** It only makes that clause not match.
- **Forgetting the end of a route map.** A last clause with no match is needed if you want to "permit the rest". Otherwise, the implicit deny drops those routes.
- **Applying PBR on the outbound interface.** PBR on Cisco works on the interface where packets arrive.
- **Expecting unmatched PBR traffic to be dropped.** It uses normal routing.
- **Using PBR with no tracking.** If the next hop looks up but the path behind it is dead, traffic is lost.
- **Calling a VRF a firewall.** A VRF stops routes from mixing, but it has no stateful checks and no encryption.
- **Pinging without `vrf`.** The ping uses the global table and fails, which looks like a network fault.
- **Trying to summarise inside an OSPF area.** OSPF allows it only at ABRs and ASBRs.
- **Removing the Null0 route because it "looks wrong".** It prevents a loop.
- **Thinking BFD replaces the routing protocol.** It only detects. It needs a client and does not discover or carry routes.
- **Blocking BFD with an ACL.** Single-hop sessions use UDP 3784 and echo uses 3785. Multi-hop sessions use UDP 4784. An interface ACL can stop the session from coming up.
- **Setting 50 ms BFD on any platform.** If BFD runs on the CPU, this causes false failures.

## Interview Questions

**Q: What is ECMP, and how does a router decide which path a packet takes?**

A: ECMP means a router installs several routes to one prefix when they tie on AD and metric. For each packet, it hashes header fields, usually the IPs and often the ports and protocol. The result picks a path. Packets of one flow get the same hash, so they stay in order on one path.

**Q: What is hash polarisation, and how do you fix it?**

A: It happens when several layers of routers use the same hash on the same fields. The first router sorts flows, then the next router gets only one type and sends them all down one link. The fix is a per-router seed or different hash inputs, so each router makes its own choice.

**Q: Why is two-way redistribution at two points dangerous?**

A: Each protocol loses the other's loop protection at the border. A route can go out of one border router and come back through the other. It can cause suboptimal paths, because of AD, and routing loops where a dead route keeps itself alive. I fix it with route tags, prefix lists, and AD changes, or I use one-way redistribution with a default route.

**Q: What is a seed metric? Give some Cisco defaults.**

A: It is the starting metric a route gets when it enters a new protocol, because metrics do not translate. On Cisco IOS, routes from another protocol into RIP or EIGRP have an infinite seed metric, so you must set one. OSPF uses 20 and type E2, or 1 for BGP routes. IS-IS uses 0.

**Q: What does `ip prefix-list X permit 10.0.0.0/8 ge 16 le 24` match?**

A: It matches any prefix inside 10.0.0.0/8 with a length from /16 to /24. For example, it matches 10.1.0.0/16 and 10.1.1.0/24. It does not match 10.0.0.0/8 itself or 10.1.1.0/25. Anything else hits the implicit deny.

**Q: In a route map, what happens if a route matches nothing, and how do match lines combine?**

A: It hits the implicit deny and is dropped. For PBR, the same case means normal routing. Different match types in one clause are AND. Several values in one match line are OR. A clause with no match line matches everything.

**Q: What is PBR, and what happens when the next hop is down?**

A: PBR forwards packets based on source, port or other fields instead of only the destination. On Cisco, it is applied inbound on an interface using a route map. Non-matching traffic uses the normal routing table. If the set next hop is not available, the router falls back to normal routing, and I use IP SLA tracking so that it can detect real failures beyond the next hop.

**Q: What is the difference between VRF, VRF-lite and a VLAN?**

A: A VLAN separates traffic at Layer 2. A VRF gives a router separate Layer 3 routing tables, and it allows overlapping addresses. VRF-lite means VRF without MPLS: every router on the path has the VRFs, with a separate subinterface per VRF per link. In MPLS L3VPN, only edge routers hold VRFs and MP-BGP carries the routes.

**Q: What are RD and RT?**

A: The RD makes a VPN route unique in BGP, so two customers can both use 10.1.1.0/24. The RT is a BGP community that controls which VRFs import a route. In short, RD gives uniqueness and RT gives policy.

**Q: Why does summarisation add a route to Null0?**

A: The summary may cover addresses that do not exist. If a packet arrives for one of them and the router has a default route, it can send it back toward the sender, which forms a loop. The Null0 route matches those packets and drops them. Real subnets still win because they have longer prefixes.

**Q: Where can OSPF summarise, and why not inside an area?**

A: Only at an ABR for inter-area routes, and at an ASBR for external routes. Inside an area, every router must have the same link-state database to run SPF correctly, so routers cannot hide detail.

**Q: What is BFD, and how quickly does it detect a failure?**

A: BFD is a light protocol that checks whether the forwarding path to a neighbour works. It runs over UDP, and clients such as OSPF and BGP register with it. Detection time is the interval multiplied by the multiplier, so 300 ms x 3 gives 900 ms. Compare that to OSPF's default 40-second dead time. It is most useful when a switch sits between routers, because the local port stays up.

## Quick Recap

- ECMP uses several tied paths at once. It hashes flows, so it does not balance bytes, and it needs seeds to avoid polarisation.
- Redistribution copies routes between protocols and needs a seed metric. Two-way redistribution at two points needs tags, filters and AD care to stop loops.
- Prefix lists match network and length, with `ge` and `le`. Route maps add ordered logic with `match` and `set`, and they end in an implicit deny.
- PBR chooses a next hop by policy, inbound on Cisco. Unmatched packets use normal routing, and you should add tracking.
- VRFs give separate routing tables and allow overlapping addresses. VRF-lite does this hop by hop without MPLS.
- Summarisation shrinks tables and hides flaps. OSPF does it only at ABRs and ASBRs, and the Null0 route prevents loops.
- BFD detects failures in milliseconds for client protocols. It only detects: the routing protocol still repairs the path.
