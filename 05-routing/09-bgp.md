# BGP

**Level:** Advanced | **Prerequisites:** [Dynamic Routing Overview](04-dynamic-routing-overview.md), [OSPF](06-ospf.md), [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md)

## In One Line

BGP (Border Gateway Protocol) is the routing protocol that connects independent networks into one internet, and it chooses paths by business policy, not by speed.

## Why It Matters

The internet is not one network. It is tens of thousands of networks run by different companies. Each one has its own routers, its own rules and its own money to protect.

These networks need a way to tell each other, "I can reach these addresses." They also need a way to say, "I will carry this traffic but not that traffic, because that customer pays me and this other network does not." A protocol like [OSPF](06-ospf.md) cannot do this. OSPF assumes all routers trust each other and share one goal: the lowest cost path.

BGP solves three real problems:

1. **Scale.** The global IPv4 table holds roughly 1 million routes in 2026, and the IPv6 table holds over 200,000. Link-state protocols are not built to flood that much data.
2. **Policy.** A company must be able to say, "Use the cheap link first," or "Never send my traffic through that network."
3. **Independence.** No network shows its inside to any other network. Each one shares only what it wants to share.

Interviewers ask about BGP because it tests many things at once: TCP, routing, policy and design. It also has a clear order of rules to remember. And when it fails, it makes headlines. Network, cloud, SRE and security interviews all use it.

BGP is also no longer only for ISPs. It runs inside data centres, in cloud connections and in Kubernetes clusters.

## Simple Explanation

Think of countries and international mail.

Each country runs its own postal system inside its borders. One country uses trucks, another uses trains. Nobody outside cares. That inside system is like an IGP (Interior Gateway Protocol), which is a routing protocol that runs inside one network, such as OSPF.

Between countries, there must be an agreement. Each country's border office tells its neighbours, "I can deliver to these cities, and the mail would pass through these countries to get there." That agreement is BGP.

Each border office then decides which road to use. The choice is not always the shortest. A country may say, "I have a contract with this neighbour, so I send mail that way, even if it is longer." It may also say, "I refuse to send mail through that country at all."

So BGP is like a signpost system. Each sign says where you can get to, and it lists the countries you pass through on the way.

**Where the analogy stops:** real mail has a central body with global rules. BGP has none. No one is in charge of the global routing table. Every network decides for itself what to believe, and by default BGP trusts what its neighbours say. That is the reason hijacks are possible, as you will see later.

## How It Works

### 1. Autonomous Systems

An **Autonomous System (AS)** is a group of IP networks and routers run under one routing policy. In simple words, it is one organisation's network, as seen from the outside. Examples: Google is AS15169, Cloudflare is AS13335, Meta is AS32934.

A **prefix** is a block of IP addresses, such as 203.0.113.0/24. BGP carries prefixes, not single hosts.

A **BGP speaker** is a router that runs BGP. A **peer** (also called a **neighbor**) is another BGP speaker that it has a session with.

BGP is the **Exterior Gateway Protocol (EGP)** in use today. Note a small naming trap. "EGP" is the general type of protocol that runs between networks. It was also the name of one old protocol that BGP replaced.

### 2. AS numbers

Every public AS has a unique number, called an **ASN**. IANA gives blocks of numbers to the five Regional Internet Registries (RIRs): ARIN, RIPE NCC, APNIC, LACNIC and AFRINIC. The RIRs then give numbers to organisations.

ASNs were 16 bits long at first. They ran out, so 32-bit ASNs were added (RFC 6793).

| Range | Use |
|---|---|
| 0 | Reserved. Not used in routing |
| 1-23455, 23457-64495 | Public 16-bit ASNs |
| 23456 | AS_TRANS, a stand-in for 32-bit ASNs on old routers |
| 64496-64511 | Documentation (RFC 5398) |
| 64512-65534 | Private 16-bit ASNs (RFC 6996) |
| 65535 | Reserved |
| 65536-65551 | Documentation (RFC 5398) |
| 4200000000-4294967294 | Private 32-bit ASNs (RFC 6996) |
| 4294967295 | Reserved |

Private ASNs are like private IP addresses. They must not appear on the public internet. An ISP strips them when a customer uses one.

This chapter uses private ASNs for made-up networks and real ASNs only when it talks about real companies.

### 3. How networks connect: transit and peering

Two commercial deals shape almost everything in BGP.

- **Transit:** you pay a provider, and it carries your traffic to the whole internet.
- **Peering:** two networks swap traffic between their own customers, often for free. Neither one gets access to the rest of the internet through the other.

A **Tier 1** network can reach the whole internet through peering alone, without paying anyone for transit. Examples often named are Arelion (AS1299), Lumen (AS3356) and NTT (AS2914). The exact list is debated.

An **Internet Exchange Point (IXP)** is a place, usually a big switch fabric, where many networks connect and peer with each other at low cost.

```text
             peering (no money)
  [Tier 1 A] <--------------> [Tier 1 B]
      ^                           ^
      | transit (customer pays)   | transit
      |                           |
  [Regional ISP]              [Regional ISP]
      ^        ^                  ^
      |        |                  |
  [Company] [Company]          [Company]
```

The money explains the usual policy. A network wants to send traffic to customers first (they pay), then to peers (free), and to providers last (it pays). The usual export rule follows from this:

| Route learned from | Tell customers? | Tell peers? | Tell providers? |
|---|---|---|---|
| A customer | Yes | Yes | Yes |
| A peer | Yes | No | No |
| A provider | Yes | No | No |

Fuller truth: this is a common model (often called Gao-Rexford, after the researchers who described it). It is not a protocol rule. BGP itself will not enforce it. When a network breaks this table by mistake, you get a route leak.

### 4. BGP is a path vector protocol

OSPF floods a map of the network. Every router knows the full picture. BGP does not work like that.

BGP is a **path vector** protocol. A router tells its neighbour the prefix and the list of ASes the route has passed through. That list is called the **AS_PATH**. Each AS adds its own number to the front when it sends the route to another AS.

```text
Origin: AS 64512 announces 203.0.113.0/24

 AS 64512 ------> AS 65001 ------> AS 65002 ------> AS 65003
           path:           path:            path:
           64512           65001 64512      65002 65001 64512
```

The right-most number is the **origin AS**, the network that first announced the prefix. The left-most number is the neighbour that just sent it.

**Loop prevention** is simple. If a router sees its own ASN in the AS_PATH of a route, it drops the route. The route has already passed through this AS.

### 5. eBGP and iBGP

BGP sessions come in two kinds.

- **eBGP (external BGP):** between routers in different ASes.
- **iBGP (internal BGP):** between routers in the same AS.

The protocol is the same. Some rules differ.

```text
           AS 64512                     AS 65001
 +---------------------------+       +-----------+
 |  R2 ------ iBGP ------ R1 |<eBGP->|  ISP-A    |
 |  (OSPF runs inside too)   |       |           |
 +---------------------------+       +-----------+
```

| Topic | eBGP | iBGP |
|---|---|---|
| Peers are in | Different ASes | Same AS |
| Peering address | Usually the directly connected link | Usually loopbacks |
| TTL of packets | 1 by default | 255 on Cisco |
| Adds own ASN to AS_PATH | Yes | No |
| NEXT_HOP | Changed to self | Not changed by default |
| LOCAL_PREF | Not sent | Sent |
| Passes on routes learned from a peer of the same kind | Yes | No, this is iBGP split horizon |
| Cisco administrative distance | 20 | 200 |

**Why does iBGP exist at all?** A big AS has many edge routers. The route learned at the edge in London must reach the edge in Tokyo, along with all its attributes. You should not put 1 million routes into OSPF. So iBGP carries external routes across the AS, while the IGP only carries the internal addresses (loopbacks and links) that iBGP needs. This is the key split:

- The IGP answers: "How do I reach the other routers in my AS?"
- iBGP answers: "Which edge router leads to this outside prefix?"

**Why loopbacks for iBGP?** A loopback is a virtual interface that stays up as long as the router is alive. If one physical link fails, the IGP finds another way to the loopback, and the session stays up.

Peers do not need to be directly connected in iBGP. The only need is IP reachability.

### 6. BGP runs on TCP port 179

BGP does not build its own reliability. It runs on **TCP port 179**. See [the TCP handshake](../06-transport-layer/03-tcp-handshake-and-header.md) for how TCP starts.

Facts that interviewers like:

- BGP gets ordered, reliable delivery from TCP. So it does not refresh the whole table on a timer as RIP does. It sends the full table once, then only changes.
- Peers are configured by hand. BGP has no automatic neighbour discovery (data centre features exist in some vendors, but the standard has none).
- One side connects from a random high port to port 179 on the other side.
- If both connect at once, this is a **connection collision**. The session opened by the router with the higher BGP Identifier stays, and the other one is closed.
- A firewall must allow TCP 179 in both directions, or at least the direction that starts the session.
- Because BGP uses TCP, a person could call it an application on top of TCP. Its job is still to build the Layer 3 routing table.

### 7. The Finite State Machine (FSM)

A **Finite State Machine** is a list of states with clear rules for moving between them. A BGP session moves through six states.

```text
              +------+
   start ---> | Idle | <------------- any error
              +--+---+
                 | begin TCP connect
                 v
            +---------+  TCP failed   +--------+
            | Connect | ------------> | Active |
            +----+----+ <------------ +---+----+
                 |      retry timer       |
                 | TCP up                 | TCP up
                 +----------+-------------+
                            | send OPEN
                            v
                      +----------+
                      | OpenSent |  wait for the peer's OPEN
                      +----+-----+
                           | good OPEN, send KEEPALIVE
                           v
                     +-------------+
                     | OpenConfirm |  wait for KEEPALIVE
                     +------+------+
                            | KEEPALIVE received
                            v
                     +-------------+
                     | Established |  send UPDATE messages
                     +-------------+
```

| State | What is happening | If stuck here, suspect |
|---|---|---|
| Idle | Not trying yet, or backing off after an error | Neighbor shut down, no route to the peer, prefix limit hit |
| Connect | Waiting for TCP to finish | Often passes in a moment |
| Active | TCP failed. Router listens and tries again | Wrong peer IP, firewall, peer has no config for you, wrong update-source |
| OpenSent | OPEN sent, waiting for reply | Peer not answering, TCP problem |
| OpenConfirm | OPEN accepted, waiting for KEEPALIVE | Rare. Peer restarted, or its KEEPALIVE was lost |
| Established | Up. Routes can flow | Nothing. This is the goal |

**Trap:** the name "Active" sounds good, but it is bad. It means the router is actively trying to get a TCP connection and failing. The healthy state is Established.

On Cisco, `show ip bgp summary` shows a number in the State/PfxRcd column when a session is Established. If you see a word there (Idle, Active), the session is not up.

### 8. Message types

All BGP messages share a 19-byte header.

```text
BGP common header (19 bytes)
+----------------------------------------------+
| Marker: 16 bytes, all ones                   |
+-----------------------+----------------------+
| Length: 2 bytes       | Type: 1 byte         |
+-----------------------+----------------------+
```

| Type | Name | Purpose |
|---|---|---|
| 1 | OPEN | Starts the session, shares settings |
| 2 | UPDATE | Announces or withdraws routes |
| 3 | NOTIFICATION | Reports an error, then the session closes |
| 4 | KEEPALIVE | Says "I am still here" |
| 5 | ROUTE-REFRESH | Asks the peer to send routes again (RFC 2918) |

The maximum message size is 4096 bytes in RFC 4271. RFC 8654 allows up to 65535 bytes for UPDATE and some other messages, if both sides support it.

**OPEN** carries these fields:

```text
+------------------------------+-----------+
| Field                        | Size      |
+------------------------------+-----------+
| Version (4)                  | 1 byte    |
| My AS                        | 2 bytes   |
| Hold Time                    | 2 bytes   |
| BGP Identifier               | 4 bytes   |
| Optional Parameters Length   | 1 byte    |
| Optional Parameters          | variable  |
+------------------------------+-----------+
```

- **My AS:** if the real ASN is larger than 65535, this field holds 23456, and the real number is sent in a capability.
- **Hold Time:** the longest silence allowed before the peer is declared dead. Both sides use the lower of the two values. It must be 0 (off) or at least 3 seconds.
- **BGP Identifier:** a 32-bit router ID, written like an IPv4 address.
- **Capabilities:** optional features, such as IPv6 support, route refresh and 4-byte ASNs.

**UPDATE** carries these parts:

```text
+-----------------------------------+-----------+
| Withdrawn Routes Length           | 2 bytes   |
| Withdrawn Routes (prefixes)       | variable  |
| Total Path Attribute Length       | 2 bytes   |
| Path Attributes                   | variable  |
| NLRI (prefixes being announced)   | variable  |
+-----------------------------------+-----------+
```

**NLRI** means Network Layer Reachability Information. It is a long name for "the prefixes." One UPDATE has one set of attributes and can list many prefixes that share them.

A sample UPDATE:

```text
UPDATE from R1 (AS 64512) to ISP-A (AS 65001)
  Withdrawn routes: none
  Path attributes:
    ORIGIN:    IGP
    AS_PATH:   64512
    NEXT_HOP:  198.51.100.2
  NLRI: 203.0.113.0/24
```

**KEEPALIVE** is only the 19-byte header. RFC 4271 suggests a hold time of 90 seconds and a keepalive of one third of that. Cisco IOS defaults are 60 seconds keepalive and 180 seconds hold. Junos defaults to 30 and 90. Check your platform.

**NOTIFICATION** carries an error code and subcode, and then the session is closed.

| Code | Meaning | Common example |
|---|---|---|
| 1 | Message Header Error | Bad length |
| 2 | OPEN Message Error | Subcode 2: Bad Peer AS |
| 3 | UPDATE Message Error | Malformed attribute |
| 4 | Hold Timer Expired | Keepalives stopped arriving |
| 5 | Finite State Machine Error | Message in the wrong state |
| 6 | Cease | Admin shutdown, prefix limit hit |

RFC 7606 changed how many UPDATE errors are handled. A router can now treat a bad attribute as a withdrawal of those routes, not as a reason to drop the whole session. One bad route should not reset a table of 1 million routes.

### 9. What a router does with a route

A **RIB** (Routing Information Base) is a table of routes. BGP keeps three logical tables.

```text
 peer --> Adj-RIB-In --> inbound policy --> best path --> Loc-RIB
                                                             |
                                                             v
                                                   IP routing table
                                                             |
 peer <-- Adj-RIB-Out <-- outbound policy <------------------+
```

1. Routes from a peer land in **Adj-RIB-In**.
2. Inbound filters drop or change some of them.
3. The router compares all paths for each prefix and picks one best path. It goes in the **Loc-RIB**.
4. The best path is offered to the main routing table. It wins only if no source with a lower administrative distance has the same prefix. If it loses, Cisco marks it `r` (RIB-failure).
5. Outbound filters decide what each peer will get, and the result is **Adj-RIB-Out**.

**Key point:** by default a router only tells its peers about its best path, not all paths. This matters for route reflectors later.

Fuller truth: real routers do not always store three full copies. Adj-RIB-In in particular is often not kept after policy, which is why `soft-reconfiguration inbound` exists.

### 10. Path attributes

A **path attribute** is a piece of information attached to a route. Attributes fall into four groups.

| Group | Meaning | Examples |
|---|---|---|
| Well-known mandatory | All routers must know it. Must be in every UPDATE that has a route | ORIGIN, AS_PATH, NEXT_HOP |
| Well-known discretionary | All routers must know it. May be left out | LOCAL_PREF, ATOMIC_AGGREGATE |
| Optional transitive | A router may not know it, but must pass it on | COMMUNITY, AGGREGATOR |
| Optional non-transitive | A router that does not know it drops it | MED, ORIGINATOR_ID, CLUSTER_LIST |

Fuller truth: routes for IPv6 are carried in MP_REACH_NLRI, which has its own next hop field. So the classic NEXT_HOP attribute is not needed in those UPDATEs.

| Code | Attribute | What it does |
|---|---|---|
| 1 | ORIGIN | How the route entered BGP: IGP (i), EGP (e), Incomplete (?) |
| 2 | AS_PATH | The list of ASes passed |
| 3 | NEXT_HOP | The IP address to send traffic to |
| 4 | MULTI_EXIT_DISC (MED) | A hint to a neighbour AS about which way to enter |
| 5 | LOCAL_PREF | Preferred exit, inside one AS |
| 6 | ATOMIC_AGGREGATE | Says a summary lost some path detail |
| 7 | AGGREGATOR | Which AS and router made the summary |
| 8 | COMMUNITY | Tags for policy |
| 9 | ORIGINATOR_ID | Route reflector loop prevention |
| 10 | CLUSTER_LIST | Route reflector loop prevention |
| 14, 15 | MP_REACH_NLRI, MP_UNREACH_NLRI | Other address families, such as IPv6 |
| 16 | EXTENDED COMMUNITIES | 8-byte tags, used in MPLS VPN and EVPN |
| 17 | AS4_PATH | Carries 32-bit paths across old routers |
| 32 | LARGE_COMMUNITY | 12-byte tags |

**ORIGIN** values: `i` means the route was put into BGP by a `network` command or similar. `e` means the old EGP protocol, which is history. `?` means it came from redistribution. The names confuse people: IGP here does not mean the route is from OSPF. It only means "put in on purpose."

**NEXT_HOP** causes many real problems. An eBGP router sets itself as next hop when it sends a route out. But when it passes that route to an iBGP peer, the next hop stays the same by default.

```text
 ISP-A 198.51.100.1 ----- R1 ----- iBGP ----- R2

 R2 learns 192.0.2.0/24 with next hop 198.51.100.1.
 If R2 has no route to 198.51.100.1, the route is not usable.
```

There are two fixes: use `next-hop-self` on R1, or carry the link subnet in the IGP. Most networks use `next-hop-self`.

### 11. Best path selection

For each prefix, a router may have several paths. It picks one with a strict order of rules. It stops at the first rule that picks a winner.

First, the path must be usable: its NEXT_HOP must be reachable. (A route with your own ASN in its AS_PATH was already dropped when it arrived.)

This is the Cisco IOS order. The steps in bold are the ones people are asked about most.

| Step | Rule | Winner | Notes |
|---|---|---|---|
| 1 | **Weight** | Highest | Cisco only. Local to the router. Not sent to peers |
| 2 | **LOCAL_PREF** | Highest | Shared inside the AS. Common default is 100 |
| 3 | Locally originated | Local wins | `network`, `aggregate-address`, redistribution |
| 4 | **AS_PATH length** | Shortest | |
| 5 | ORIGIN | Lowest | IGP < EGP < Incomplete |
| 6 | **MED** | Lowest | By default only compared for the same neighbour AS |
| 7 | **eBGP over iBGP** | eBGP | |
| 8 | IGP metric to next hop | Lowest | The closest exit |
| 9 | Multipath check | | If enabled, equal paths are all installed |
| 10 | Oldest eBGP path | Oldest | Keeps routes stable |
| 11 | Lowest BGP router ID | Lowest | |
| 12 | Shortest CLUSTER_LIST | Shortest | With route reflectors |
| 13 | Lowest neighbor IP | Lowest | Final tie breaker |

**Mnemonic:** "**We Love Oranges AS Oranges Mean Pure Refreshment**"

```text
 We           Weight
 Love         Local preference
 Oranges      Originated locally
 AS           AS path length
 Oranges      Origin
 Mean         MED
 Pure         Paths: eBGP over iBGP
 Refreshment  Router ID
```

A second version that also holds the next-hop check and IGP metric is "**N WLLA OMNI**": Next hop, Weight, Local pref, Locally originated, AS path, Origin, MED, Neighbour type, IGP metric.

**Who decides what:**

- Steps 1-2 are your own policy. They control how traffic **leaves** your network.
- Step 4 is shaped by others and by you. Prepending is your way to affect it.
- Step 6 is the only step where a neighbour's direct wish is used.
- Step 8 gives **hot potato routing**: hand the traffic to the next network as soon as possible, so it costs you less. This is one reason internet paths are often asymmetric, which means the forward and return paths differ.

**Standards versus vendors:** RFC 4271 defines LOCAL_PREF, AS_PATH, ORIGIN, MED, eBGP over iBGP, IGP cost, router ID and peer address. Weight is Cisco's, and Junos has no weight. Learn the Cisco order for interviews, and say that vendors differ a little.

Rules that change the order on Cisco:

- `bgp always-compare-med` compares MED across different neighbour ASes.
- `bgp bestpath med missing-as-worst` treats a missing MED as the worst value. By default a missing MED counts as 0, which is the best value.
- `bgp bestpath compare-routerid` skips the oldest-path step.
- `bgp deterministic-med` groups paths by neighbour AS before comparing, so the result does not depend on the order paths arrived.

### 12. iBGP full mesh and split horizon

**The rule:** a route learned from an iBGP peer must not be sent to another iBGP peer.

**Why?** Inside an AS, the AS_PATH does not change. So the normal loop check does not work. BGP has no way to see a route going round in a circle inside the AS. The simplest safe answer is to forbid the second hop. This is **iBGP split horizon**.

**Result:** every iBGP router must peer directly with every other iBGP router. This is a **full mesh**.

```text
 Learned from eBGP, R1 tells all its iBGP peers.
 R2 and R3 do not pass it on.

        R1
       /  \        Sessions needed = n(n-1)/2
      /    \         4 routers  =    6
    R2 ---- R3      10 routers  =   45
                   100 routers  = 4950
```

Analogy: in a small team, the boss tells each person directly, and no one may pass it on as gossip. It works for 4 people. For 100 people, it does not.

**Where the analogy stops:** the session count is only part of the cost. Each extra session also adds memory, CPU and config work on every router.

Two tools fix this scale problem: route reflectors and confederations. Route reflectors are far more common.

History note: old BGP had a **synchronization** rule. An iBGP route could only be used if the IGP also knew it. That rule is off by default on Cisco since IOS 12.2(8)T, and you can treat it as history.

### 13. Route reflectors

A **route reflector (RR)** is a router that is allowed to break the split horizon rule (RFC 4456). The routers it serves are its **clients**. Clients peer only with the RR, not with each other.

Analogy: a news editor. Reporters send stories only to the editor, and the editor sends them on to everyone.

**Where the analogy stops:** an RR only passes on the one path it thinks is best. It is also not in the data path by need. Traffic flows between the edge routers, and need not pass through the RR.

```text
                +------+
       +------->|  RR  |<-------+
       |        +--+---+        |     Sessions: 3, not 6
       |           ^            |
       v           v            v
   [Client 1]  [Client 2]   [Client 3]
```

**Reflection rules:**

| Route came from | RR sends it to |
|---|---|
| eBGP peer | All clients, all non-clients, eBGP peers |
| Client | All clients (not the sender), non-clients, eBGP peers |
| Non-client iBGP peer | Clients and eBGP peers only |

**Loop prevention with two new attributes:**

- **ORIGINATOR_ID:** the router ID of the router that first put the route into the AS. If a router sees its own ID here, it drops the route.
- **CLUSTER_LIST:** a list of clusters the route has passed. A **cluster** is an RR and its clients. Each RR adds its cluster ID. If an RR sees its own ID, it drops the route.

**Config is needed only on the RR.** Clients run normal iBGP.

```text
! Cisco IOS, on the route reflector
router bgp 64512
 neighbor 10.0.0.11 remote-as 64512
 neighbor 10.0.0.11 update-source Loopback0
 address-family ipv4
  neighbor 10.0.0.11 activate
  neighbor 10.0.0.11 route-reflector-client
```

**Design notes:**

- Use two RRs for each cluster, or the RR becomes a single point of failure. Clients peer with both.
- An RR does not change NEXT_HOP by default. Clients still need a route to it.
- An RR picks its best path from its own position. A client may be given an exit that is close to the RR, but not close to the client. Add-Path and Optimal Route Reflection are fixes (see Going Deeper).
- RRs can be built in levels for very large networks.

### 14. Confederations

A **confederation** (RFC 5065) splits one AS into smaller **sub-ASes**. The outside world still sees one AS.

Analogy: a federal country with states. Each state has its own affairs, but foreign countries deal with one country.

**Where the analogy stops:** the sub-ASes are not really independent. They normally share one IGP design and one policy.

```text
        The outside world sees only AS 64600
 +------------------------------------------------------+
 |                                                      |
 |  +----------------+  confed eBGP  +----------------+ |
 |  | Sub-AS 65010   |<------------->| Sub-AS 65020   | |
 |  | iBGP full mesh |               | iBGP full mesh | |
 |  +----------------+               +----------------+ |
 |                                                      |
 +------------------------------------------------------+
```

How it works:

1. Inside a sub-AS, iBGP rules apply (often a full mesh, sometimes RRs).
2. Between sub-ASes, routers use **confederation eBGP**. It looks like eBGP, but NEXT_HOP, LOCAL_PREF and MED are kept as if it were iBGP.
3. Sub-AS numbers go in special AS_PATH segments (AS_CONFED_SEQUENCE). These stop loops and are not counted in path length.
4. At the edge of the confederation, these segments are removed. Only the real AS number is sent out.

```text
! Cisco IOS, router in sub-AS 65010
router bgp 65010
 bgp confederation identifier 64600
 bgp confederation peers 65020
 neighbor 10.1.1.2 remote-as 65010
 neighbor 10.2.2.2 remote-as 65020
```

| Point | Route reflector | Confederation |
|---|---|---|
| Migration effort | Low, change only the RR | High, all routers change AS config |
| Used for | Most large networks | Very large networks, or company mergers |
| Hides the full mesh by | One router reflecting routes | Splitting the AS |
| Loop prevention | ORIGINATOR_ID, CLUSTER_LIST | AS_CONFED segments |

The two can be used together.

### 15. Traffic engineering: LOCAL_PREF, MED, prepending

Traffic engineering means steering traffic on purpose. The first thing to know is the direction, since the tools differ.

| Goal | Tool | Works how |
|---|---|---|
| Control how traffic **leaves** your AS | LOCAL_PREF (or weight on one router) | You decide. Strong and reliable |
| Control how traffic **enters** your AS | More-specific prefixes, prepending, MED, provider communities | You can only ask. Others decide |

**You control your own outbound path. You only influence inbound.** Say that sentence in an interview.

**LOCAL_PREF**

- Higher wins. The common default is 100 on Cisco and Junos.
- It is shared with all iBGP peers, so the whole AS agrees on an exit.
- It is not sent to eBGP peers.
- Set it inbound at the edge.

```text
! Cisco IOS: prefer routes from ISP-A
route-map ISPA-IN permit 10
 set local-preference 200
```

**MED (Multi-Exit Discriminator)**

- Lower wins.
- Used when two ASes connect in more than one place. It tells the neighbour which link you prefer it to use to enter.
- It is sent to a directly connected neighbour AS and goes no further.
- It is only a hint. Step 2 and step 4 come first, so the neighbour's LOCAL_PREF or AS_PATH can ignore it.

```text
          AS 64512                 AS 65001
       [R1] MED 10  ----link 1---- [ISP router]
       [R2] MED 50  ----link 2---- [ISP router]

 AS 65001 prefers link 1 for traffic into AS 64512.
```

**AS path prepending**

You add your own ASN extra times to the AS_PATH on one link. The path looks longer, so others are less likely to use it.

```text
! Cisco IOS: make the path through ISP-B look longer
route-map ISPB-OUT permit 10
 set as-path prepend 64512 64512 64512
```

Limits to know:

- It only helps at step 4. If a far AS sets LOCAL_PREF, prepending has no effect there.
- Past a few prepends, it adds little. A very long path also makes it easier for a hijacker's shorter path to win.
- It is a blunt tool.

**More-specific prefixes** are the strongest tool, because of **longest prefix match**: a router always prefers the most specific route that fits, before BGP best path even matters. Announce 2001:db8:100::/40 on both links, and 2001:db8:100::/48 on only one link. Traffic for that /48 follows the one link. IPv4 works the same way, for example a /23 on both links and a /24 on one. The cost is more routes in the global table, and a /24 is usually the smallest IPv4 prefix accepted globally (/48 for IPv6).

### 16. Communities

A **community** is a tag attached to a route (RFC 1997). It does nothing alone. It means something only when a router's policy acts on it.

Analogy: a coloured sticker on a parcel. The sticker does not move the parcel. Staff are told, "Red means handle with care."

**Where the analogy stops:** communities can be dropped or changed at any AS along the way, and each ISP defines its own meanings.

A standard community is 32 bits. It is written as `ASN:value`, such as `65001:80`.

**Well-known communities:**

| Name | Value | Meaning |
|---|---|---|
| NO_EXPORT | 65535:65281 | Do not send outside this AS (or confederation) |
| NO_ADVERTISE | 65535:65282 | Do not send to any peer |
| NO_EXPORT_SUBCONFED | 65535:65283 | Do not send outside this sub-AS |
| NOPEER | 65535:65284 | Do not send to bilateral peers (RFC 3765) |
| BLACKHOLE | 65535:666 | Drop traffic to this prefix (RFC 7999) |

**Provider communities** let a customer control the ISP's behaviour. Many ISPs publish a list like this (each ISP makes its own, and these values are only an example):

| Community | Effect inside the ISP |
|---|---|
| 65001:80 | Set LOCAL_PREF to 80, a backup path |
| 65001:120 | Set LOCAL_PREF to 120 |
| 65001:3003 | Prepend three times toward peers |

Example: tag a route on the customer side, and act on it on the ISP side.

```text
! Customer router (Cisco IOS)
route-map ISPA-OUT permit 10
 set community 65001:80
!
router bgp 64512
 address-family ipv4
  neighbor 198.51.100.1 send-community

! ISP router (Cisco IOS)
ip community-list standard BACKUP permit 65001:80
!
route-map FROM-CUSTOMER permit 10
 match community BACKUP
 set local-preference 80
route-map FROM-CUSTOMER permit 20
```

On Cisco IOS, communities are not sent to a peer until you configure `send-community`. This catches a lot of people.

Other community types:

- **Extended communities** (RFC 4360): 8 bytes. Used for route targets in [MPLS](13-mpls.md) VPNs and [EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).
- **Large communities** (RFC 8092): 12 bytes, written `ASN:function:parameter`. Made because 32-bit ASNs do not fit in a standard community.

### 17. Route filtering

BGP believes what it hears, unless you tell it not to. Filters are your defence.

**What to filter inbound from any peer:**

- Your own prefixes.
- Bogons: addresses that must never appear on the public internet, such as 10.0.0.0/8, 127.0.0.0/8 and 169.254.0.0/16.
- Bogon ASNs: private and reserved ASNs in the AS_PATH.
- Prefixes that are too long: longer than /24 in IPv4, or /48 in IPv6.
- RPKI invalid routes (covered below).
- For a customer: everything except what that customer is allowed to send.

**What to filter outbound:**

- Allow only your own prefixes and your customers' prefixes. Never pass one provider's routes to another provider.

**Tools on Cisco IOS:**

| Tool | Matches |
|---|---|
| Prefix list | Prefix and length |
| AS path access list | AS_PATH with a regular expression |
| Community list | Community value |
| Route map | Joins the above with match and set |
| Maximum prefix | Shuts the session if a peer sends too many routes |

Prefix list length matching:

```text
! Exactly this /8
ip prefix-list A permit 10.0.0.0/8
! This block, with any length from /8 to /24
ip prefix-list B permit 10.0.0.0/8 le 24
! Any prefix with a length from /8 to /24
ip prefix-list C permit 0.0.0.0/0 ge 8 le 24
```

AS path regular expressions:

| Regex | Matches |
|---|---|
| `^$` | Empty path: routes from your own AS |
| `^65001_` | Routes learned from neighbour 65001 |
| `_65100$` | Routes that started in AS 65100 |
| `_65002_` | Routes that passed through AS 65002 |

Where to get the data for customer filters: the **IRR** (Internet Routing Registry), which are public databases of who says they announce what, and RPKI. Tools such as `bgpq4` turn this data into prefix lists.

RFC 8212 says an eBGP speaker should not accept or send routes until a policy is set. IOS XR and recent FRR follow this idea. Classic Cisco IOS has not behaved this way by default. So never count on it. RFC 7454 is the operations guide for BGP security.

## Example

### The network

A company, AS 64512, owns 203.0.113.0/24. It has two routers and two ISPs. It wants:

- Outbound traffic to use ISP-A (the faster contract).
- Inbound traffic to also use ISP-A.
- ISP-B as a working backup.

```text
                     AS 65100
                 prefix 192.0.2.0/24
                 /               \
     198.51.100.9                 198.51.100.13
              /                     \
       [ISP-A AS 65001]        [ISP-B AS 65002]
       198.51.100.1            198.51.100.5
              |                     |
       198.51.100.2            198.51.100.6
            [R1] --- iBGP, OSPF --- [R2]
   Lo0 10.0.0.1      10.0.12.0/30      Lo0 10.0.0.2
            \                       /
             +----- AS 64512 ------+
             owns 203.0.113.0/24
```

Links: R1 to ISP-A is 198.51.100.0/30. R2 to ISP-B is 198.51.100.4/30. The links from the ISPs up to AS 65100 are 198.51.100.8/30 and 198.51.100.12/30, and .9 and .13 are the ISP ends. To keep the output short, pretend each ISP sends one route.

OSPF already runs between R1 and R2, so each can reach the other's loopback.

### R1 config

```text
! R1, Cisco IOS
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
!
ip route 203.0.113.0 255.255.255.0 Null0
!
ip prefix-list OUR-NET seq 10 permit 203.0.113.0/24
!
ip prefix-list FROM-ISP seq 5  deny 0.0.0.0/8 le 32
ip prefix-list FROM-ISP seq 10 deny 10.0.0.0/8 le 32
ip prefix-list FROM-ISP seq 15 deny 100.64.0.0/10 le 32
ip prefix-list FROM-ISP seq 20 deny 127.0.0.0/8 le 32
ip prefix-list FROM-ISP seq 25 deny 169.254.0.0/16 le 32
ip prefix-list FROM-ISP seq 30 deny 172.16.0.0/12 le 32
ip prefix-list FROM-ISP seq 35 deny 192.168.0.0/16 le 32
ip prefix-list FROM-ISP seq 40 deny 203.0.113.0/24 le 32
ip prefix-list FROM-ISP seq 50 permit 0.0.0.0/0 ge 8 le 24
!
route-map ISPA-IN permit 10
 match ip address prefix-list FROM-ISP
 set local-preference 200
!
route-map ISPA-OUT permit 10
 match ip address prefix-list OUR-NET
!
router bgp 64512
 bgp router-id 10.0.0.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 198.51.100.1 remote-as 65001
 neighbor 198.51.100.1 description ISP-A
 neighbor 198.51.100.1 password Example-Key-1
 neighbor 10.0.0.2 remote-as 64512
 neighbor 10.0.0.2 description R2-iBGP
 neighbor 10.0.0.2 update-source Loopback0
 !
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.0
  neighbor 198.51.100.1 activate
  neighbor 198.51.100.1 route-map ISPA-IN in
  neighbor 198.51.100.1 route-map ISPA-OUT out
  neighbor 198.51.100.1 maximum-prefix 1200000 90
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
 exit-address-family
```

What each key line does:

- `ip route ... Null0`: the `network` command only announces a prefix that is already in the routing table. This static route puts it there. It also drops traffic for unused addresses in the block, which prevents a loop.
- `update-source Loopback0`: makes iBGP use the loopback as its source. It must match what R2 has as its neighbour address.
- `next-hop-self`: lets R2 use R1 as next hop, so R2 does not need the ISP link in OSPF.
- `maximum-prefix`: shuts the session if the peer sends far too many routes. The full table is about 1 million routes, so 1200000 leaves room. Review this number as the table grows.
- The real bogon list would also deny the documentation ranges. They are left out here since this example uses them.

### R2 config (only the parts that differ)

R2 also has the same FROM-ISP and OUR-NET prefix lists, the same Null0 static route, and its own Loopback0 (10.0.0.2).

```text
! R2, Cisco IOS
route-map ISPB-IN permit 10
 match ip address prefix-list FROM-ISP
 ! no local-preference set: stays at the default of 100
!
route-map ISPB-OUT permit 10
 match ip address prefix-list OUR-NET
 set as-path prepend 64512 64512 64512
!
router bgp 64512
 bgp router-id 10.0.0.2
 no bgp default ipv4-unicast
 neighbor 198.51.100.5 remote-as 65002
 neighbor 198.51.100.5 description ISP-B
 neighbor 198.51.100.5 password Example-Key-2
 neighbor 10.0.0.1 remote-as 64512
 neighbor 10.0.0.1 update-source Loopback0
 !
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.0
  neighbor 198.51.100.5 activate
  neighbor 198.51.100.5 route-map ISPB-IN in
  neighbor 198.51.100.5 route-map ISPB-OUT out
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
 exit-address-family
```

### Check the sessions

```text
R2# show ip bgp summary
BGP router identifier 10.0.0.2, local AS number 64512
BGP table version is 9, main routing table version 9
2 network entries using 496 bytes of memory
4 path entries using 544 bytes of memory

Neighbor      V    AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down  St/Pfx
10.0.0.1      4 64512      62      60      9   0    0 00:51:12      2
198.51.100.5  4 65002      71      58      9   0    0 00:50:48      1
```

Both sessions show a number, so both are Established. (The columns are shortened here to fit.)

### Check the routes

```text
R2# show ip bgp
BGP table version is 9, local router ID is 10.0.0.2
Status codes: * valid, > best, i - internal, r RIB-failure
Origin codes: i - IGP, e - EGP, ? - incomplete

     Network        Next Hop        Metric LocPrf Weight Path
 *   192.0.2.0      198.51.100.5                       0 65002 65100 i
 *>i                10.0.0.1                  200      0 65001 65100 i
 *>  203.0.113.0    0.0.0.0              0         32768 i
 * i                10.0.0.1             0    100      0 i
```

How to read the first prefix:

- Line 1: the path through ISP-B. Valid, but not best.
- Line 2: the path through R1 and ISP-A. `*>` means valid and best. `i` after it means learned from iBGP.

For 203.0.113.0, R2 prefers its own local route, because the weight of 32768 beats 0 at step 1.

Now the detail view for one prefix:

```text
R2# show ip bgp 192.0.2.0
BGP routing table entry for 192.0.2.0/24, version 9
Paths: (2 available, best #2, table default)
  65002 65100
    198.51.100.5 from 198.51.100.5 (198.51.100.5)
      Origin IGP, localpref 100, valid, external
  65001 65100
    10.0.0.1 (metric 2) from 10.0.0.1 (10.0.0.1)
      Origin IGP, localpref 200, valid, internal, best
```

### Walk the best path decision on R2

| Step | Path via ISP-B | Path via R1 | Result |
|---|---|---|---|
| Next hop reachable | Yes | Yes (OSPF metric 2) | Both pass |
| 1. Weight | 0 | 0 | Tie |
| 2. LOCAL_PREF | 100 | 200 | R1 path wins, stop |

Note what does **not** matter. R2's path via ISP-B is eBGP and R1's is iBGP, and eBGP is normally preferred. But that rule is step 7, and step 2 ended the contest.

```text
R2# show ip route bgp
B     192.0.2.0/24 [200/0] via 10.0.0.1, 00:12:04

R1# show ip route bgp
B     192.0.2.0/24 [20/0] via 198.51.100.1, 00:12:20
```

The `[200/0]` means administrative distance 200 (iBGP) and metric 0. `[20/0]` means eBGP.

**A detail worth saying in an interview:** R1 has only one path to 192.0.2.0/24. Why? R2 chose R1's path as its best, and a router only advertises its best path. So R2 does not tell R1 about ISP-B.

### Check inbound from far away

A router in AS 65100 sees:

```text
AS65100# show ip bgp 203.0.113.0
BGP routing table entry for 203.0.113.0/24, version 40
Paths: (2 available, best #1, table default)
  65001 64512
    198.51.100.9 from 198.51.100.9 (198.51.100.9)
      Origin IGP, localpref 100, valid, external, best
  65002 64512 64512 64512 64512
    198.51.100.13 from 198.51.100.13 (198.51.100.13)
      Origin IGP, localpref 100, valid, external
```

LOCAL_PREF ties, and the shorter AS_PATH wins at step 4. Inbound traffic comes through ISP-A.

### What happens when the ISP-A link fails

1. The link goes down. On Cisco, `bgp fast-external-fallover` is on by default, so R1 resets the directly connected eBGP session at once. Without that, or if a switch sits in the middle so the interface stays up, R1 would wait for the 180 second hold timer (or for BFD, covered later).
2. R1 removes the ISP-A routes and sends R2 an UPDATE that withdraws 192.0.2.0/24.
3. R2 now has only the ISP-B path, so it becomes best. Outbound traffic switches.
4. ISP-A withdraws 203.0.113.0/24 from its peers. Far routers fall back to the longer path. Inbound traffic switches.

Outbound recovery is your own work, so it is quick. Inbound recovery depends on the outside world, and it takes longer.

### The same idea on Linux and Windows

FRRouting (FRR) is open-source routing software for Linux. Its config language is close to Cisco's.

```text
! /etc/frr/frr.conf on Linux (FRR)
! Route-map and prefix-list lines are the same as IOS, left out here
router bgp 64512
 bgp router-id 10.0.0.1
 neighbor 198.51.100.1 remote-as 65001
 address-family ipv4 unicast
  network 203.0.113.0/24
  neighbor 198.51.100.1 route-map ISPA-IN in
  neighbor 198.51.100.1 route-map ISPA-OUT out
 exit-address-family
```

Modern FRR needs a route-map on eBGP peers, or it will not exchange routes (the `bgp ebgp-requires-policy` setting, on by default).

```bash
# Linux with FRR: session and route checks
sudo vtysh -c "show ip bgp summary"
sudo vtysh -c "show ip bgp 192.0.2.0/24"

# Linux: see the TCP session and BGP packets
ss -tn '( sport = :179 or dport = :179 )'
sudo tcpdump -ni eth0 tcp port 179
```

```powershell
# Windows: test whether TCP 179 is reachable
Test-NetConnection 198.51.100.1 -Port 179

# Windows Server with the RRAS BGP role
Get-BgpPeer
Get-BgpRouteInformation
```

A router usually accepts TCP 179 only from a configured peer. So a failed test from your laptop does not prove the BGP port is broken.

To see the internet's routes yourself, use a public **looking glass**, which is a website that shows the BGP view from another network. RIPEstat, bgp.he.net and the Route Views project are well known. See also how to trace paths in [Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

### Fault finding: a session will not come up

Suppose R1 has a typo: `neighbor 198.51.100.1 remote-as 65011`, but the ISP is really AS 65001.

```text
R1# show ip bgp summary
Neighbor      V    AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down  St/Pfx
198.51.100.1  4 65011       2       3      1   0    0 never    Idle

%BGP-3-NOTIFICATION: sent to neighbor 198.51.100.1 active 2/2
(peer in wrong AS) 2 bytes FDE9
```

Code 2/2 is OPEN Message Error, Bad Peer AS. The bytes FDE9 in hex equal 65001 in decimal, the ASN the peer really sent. So the log shows you the answer. (Exact log text varies by version.)

**Checklist:**

| Check | Command (Cisco IOS) |
|---|---|
| Can I ping the peer from the right source? | `ping 10.0.0.1 source Loopback0` |
| Is TCP 179 blocked by an ACL or firewall? | `show ip access-lists` |
| Do the ASNs match on both sides? | `show ip bgp neighbors 198.51.100.1` |
| Is `update-source` right for loopback peering? | `show run \| section bgp` |
| eBGP over loopbacks without `ebgp-multihop`? | `show ip bgp neighbors` |
| MD5 password match? | Log shows `Invalid MD5 digest` or similar |
| Am I sending the route? | `show ip bgp neighbors X advertised-routes` |
| Is the peer sending the route? | `show ip bgp neighbors X routes` |
| Is the next hop reachable? | `show ip route <next hop>` |
| Did a filter drop it? | `show ip bgp neighbors X received-routes` (needs soft reconfiguration) |

Change a policy without dropping the session:

```text
R1# clear ip bgp 198.51.100.1 soft in
R1# clear ip bgp 198.51.100.1 soft out
```

Full reset (`clear ip bgp 198.51.100.1`) drops the session. Avoid that on a live link.

## Going Deeper

### Why there is no real security by default

BGP was designed when the networks on the internet knew each other. A peer can announce any prefix, and by default nothing checks that it is allowed. Two facts make this dangerous:

1. Routers believe announcements from neighbours, unless filtered.
2. Longest prefix match means a more-specific route always wins.

### Hijacks and leaks

A **BGP hijack** is when a network announces addresses that are not its own, whether by mistake or on purpose. Traffic then flows to the wrong place. The attacker can drop it, read it, or change it. If traffic is passed on after being read, the user sees no outage at all.

```text
 Real owner: AS 65100 announces 2001:db8:100::/40

 Attacker:   AS 65099 announces 2001:db8:100::/48   (more specific)

 Result: routers everywhere use the /48, by longest prefix match.
```

| Type | What happens | Stopped by RPKI origin check? |
|---|---|---|
| Exact prefix origin hijack | Same prefix, wrong origin AS. Only part of the internet is fooled | Yes, if a ROA exists |
| More-specific hijack | Longer prefix, wrong origin. Most of the internet follows | Yes, if a ROA exists |
| Forged origin hijack | Path is faked to end in the real origin AS | No |
| Route leak | Real routes passed on beyond their allowed scope | Not by origin check |

A **route leak** (RFC 7908) is often an honest mistake. Classic case: a customer with two ISPs passes routes from ISP-1 to ISP-2. ISP-2 now sees a path through that small customer, and huge traffic may flow through a tiny network that cannot carry it.

```text
 [ISP-1] --- full table ---> [Small customer] --- leak ---> [ISP-2]
                                                               |
                  Traffic for half the internet now goes here. |
```

Analogy for a hijack: someone puts up a fake sign that says "Post office, this way," and drivers follow it, because no one checks signs.

**Where the analogy stops:** a BGP hijack is not local. One announcement can spread around the world within minutes.

### RPKI

**RPKI (Resource Public Key Infrastructure, RFC 6480)** is a system of signed records that link IP prefixes to ASNs. It uses certificates that begin with the five RIRs. It lets a router check, "Is this AS allowed to originate this prefix?"

Analogy: a land registry. The owner of the land signs a deed saying who may use it.

**Where the analogy stops:** the deed only names who may announce the prefix first. It says nothing about the roads the announcement took to reach you.

**A ROA** (Route Origin Authorization, RFC 9582) is the signed record. It has three fields:

```text
 Prefix:     203.0.113.0/24
 maxLength:  24
 Origin AS:  64512
```

**maxLength** is the longest prefix length this ROA allows. A loose maxLength can help an attacker, so RFC 9319 advises keeping it tight.

**How it reaches a router:**

```text
 [RIR repositories] --fetch--> [Validator] --RTR protocol--> [Router]
    signed ROAs                 checks                        checks
                                signatures                    each route
```

Common validators include Routinator and rpki-client. The **RPKI-to-Router protocol** (RTR, RFC 8210) sends the checked list to routers. The router does not do any cryptography itself.

**Origin validation** (ROV, RFC 6811) gives each route one of three results:

| Route seen | Against the ROA above | Result |
|---|---|---|
| 203.0.113.0/24 from AS 64512 | Prefix and AS match | Valid |
| 203.0.113.0/24 from AS 65099 | Covered, but wrong AS | Invalid |
| 203.0.113.0/25 from AS 64512 | Longer than maxLength | Invalid |
| 198.51.100.0/24 from AS 65099 | No ROA covers it | NotFound |

**Policy:** drop Invalid. Accept Valid and NotFound. You must keep NotFound, because many prefixes have no ROA yet. Dropping those would cut you off from a large part of the internet.

```text
! Cisco IOS: connect to a validator, then drop invalid routes
router bgp 64512
 bgp rpki server tcp 10.0.0.50 port 3323 refresh 300
!
route-map ISPA-IN deny 5
 match rpki invalid
route-map ISPA-IN permit 10
 match ip address prefix-list FROM-ISP
 set local-preference 200
```

Port 3323 is common for Routinator. The IANA port for RPKI-RTR is 323. Use what your validator is set to. Run at least two validators, because they are now part of your routing security.

**What RPKI does not do:**

- It does not check the AS_PATH. An attacker can add the real ASN at the end of a fake path and pass the check.
- It does not stop route leaks, since those routes have the right origin.
- It only helps when others publish ROAs and when others drop invalid routes.

**Fuller picture of fixes:**

| Tool | Status | Protects against |
|---|---|---|
| RPKI ROV | Widely deployed. Over half of IPv4 routes had ROAs by 2024 | Origin hijacks |
| BGPsec (RFC 8205) | Standard, but almost unused | Path forgery. Costly, since each router must sign |
| ASPA | In development at the IETF | Route leaks and many forged paths. Check its current status |
| BGP Roles and OTC (RFC 9234) | Standard, growing | Route leaks |
| IRR filters | Old and common | Bad customer prefixes. Data quality varies |
| MANRS | A community programme | Good habits: filtering, anti-spoofing, coordination |

Also use monitoring (RIPE RIS, Cloudflare Radar, or the open-source BGPalerter) to learn quickly if your prefix is announced by someone else.

### Real incidents

**Facebook, 4 October 2021.** This is the one interviewers ask about most.

1. During routine maintenance, a command meant to check backbone capacity took down all backbone connections between Facebook's data centres.
2. A checking tool should have stopped the command, but a bug let it pass.
3. Facebook's DNS servers were built to withdraw their BGP routes if they could not reach the data centres. That design makes sense when one site has a problem: traffic goes to a healthy site. This time every site did it at the same moment.
4. The prefixes for Facebook's authoritative DNS servers vanished from the internet, starting about 15:40 UTC.
5. No DNS meant facebook.com, Instagram and WhatsApp did not resolve. Resolvers around the world also got a flood of retries.
6. Staff tools ran on the same network, so engineers could not get in remotely. People had to go to the data centres in person, and physical security slowed that down.
7. Service came back after about six hours.

**The lesson to state:** BGP was not the root cause. The backbone failure came first, and the BGP withdrawal was a planned safety feature that made it much worse. Keep your out-of-band network (a separate management path) truly separate, and make sure your DNS, your tools and your recovery paths do not all depend on one thing. See [DNS Basics](../07-application-protocols/01-dns-basics.md) and [Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

**Pakistan Telecom and YouTube, February 2008.** Pakistan's government ordered YouTube blocked. Pakistan Telecom (AS17557) sent YouTube traffic into a black hole by announcing 208.65.153.0/24 internally. The route leaked to its provider, PCCW (AS3491), which passed it on. YouTube's real route was a /22, so the /24 won worldwide by longest prefix match. YouTube was down for most of the world for around two hours. YouTube fought back with its own /24 and then /25 announcements. Lesson: filter your customers, and more-specific routes beat everything.

**Verizon and DQE, June 2019.** A network used a BGP optimiser, a tool that creates more-specific routes for its own internal use. Those routes leaked through a customer (Allegheny Technologies) to Verizon (AS701), and Verizon accepted and spread them. Big services saw traffic go through a small path and fail. Lesson: a max-prefix limit and customer filters would have stopped it.

**Amazon Route 53, April 2018.** Attackers announced more-specific routes for Amazon's DNS address space through a provider in Ohio. For about two hours, some DNS replies were faked to send users of a cryptocurrency wallet site to a fake site, and funds were stolen. Lesson: BGP hijacks can attack the layers above, and TLS warnings only help if users do not click through them.

**Rogers, Canada, July 2022.** During a staged config change, a routing filter was removed on some distribution routers. A huge number of routes flooded the core routers and overwhelmed them. One of Canada's largest carriers was down for most of a day. Lesson: a filter removal can cause the same flood as a route leak, and management access must stay up when the main network fails.

**AS7007, April 1997.** An early case. A router error turned a large part of the routing table into /24 routes and announced them as its own. Parts of the internet failed. People have used this as the classic warning ever since.

### Other route control tools

**Aggregation** (summarisation) combines many prefixes into a shorter one. See [Route Summarisation](../04-ip-addressing/07-vlsm-and-supernetting.md).

```text
router bgp 64512
 address-family ipv4
  aggregate-address 10.20.0.0 255.255.0.0 summary-only
```

`summary-only` hides the smaller prefixes. The `as-set` keyword keeps AS path information from those prefixes. If it is not kept, the summary carries ATOMIC_AGGREGATE. Use of `as-set` is discouraged today.

**Private AS removal:** `neighbor X remove-private-as` strips private ASNs before sending to a provider.

**Local AS:** `neighbor X local-as 65005` makes a router look like a different AS to one peer. It is useful during a merger, so peers do not have to change config all at once.

**allowas-in:** accepts routes with your own ASN in the path. It is needed for a company with sites in the same ASN joined only by an ISP. The ISP side option is `as-override`. Use with care, since it weakens loop prevention.

**Default route to BGP:** a small company often takes only a default route from an ISP, or a default plus partial routes. A full table is only useful if you have two or more ISPs and want to make per-prefix choices.

```text
! ISP side (Cisco IOS)
neighbor 198.51.100.2 default-originate
```

**Multipath:** BGP normally installs one best path. `maximum-paths 4` installs up to 4 equal eBGP paths, and `maximum-paths ibgp 4` does the same for iBGP. Paths must tie through to the IGP metric step. This is also called ECMP (Equal-Cost Multi-Path).

### Faster failure detection and softer restarts

- **BFD (Bidirectional Forwarding Detection):** a small, fast hello protocol that detects a dead path in well under a second and tells BGP. Without it, you wait for the hold timer. Cisco: `neighbor X fall-over bfd`.
- **Graceful Restart (RFC 4724):** a router can restart its control plane while it keeps forwarding, and peers keep its routes for a short time as stale.
- **Route Refresh (RFC 2918):** asks a peer to resend its routes after a policy change. This replaced the older `soft-reconfiguration inbound`, which stores all received routes and uses more memory.
- **Route flap damping (RFC 2439):** punishes a route that keeps going up and down, by hiding it for some time. Cisco defaults are a penalty of 1000 for each flap, suppress at 2000, reuse at 750, and a half-life of 15 minutes. It caused more harm than good when used with aggressive values, and it is not widely used. RFC 7196 advises a higher threshold.
- **Graceful Shutdown (RFC 8326):** a community (65535:0) that tells peers to lower preference on a route before planned work, so traffic moves first.

### Convergence

BGP is slow to settle. Three reasons:

1. **Timers.** MRAI (Minimum Route Advertisement Interval) spaces out updates. RFC 4271 suggests 30 seconds for eBGP. Vendors vary.
2. **Path hunting.** When a prefix is withdrawn, routers try each backup path in turn, each one also failing, before giving up. So a full withdrawal can take minutes.
3. **Scale.** A 1 million route table takes time to send.

### Multiprotocol BGP (MP-BGP)

RFC 4760 lets BGP carry more than IPv4 unicast routes. An **address family** is the type of route. It is named by AFI and SAFI numbers.

| Family | AFI / SAFI | Used for |
|---|---|---|
| IPv4 unicast | 1 / 1 | Normal internet routes |
| IPv6 unicast | 2 / 1 | IPv6 internet routes |
| VPNv4 | 1 / 128 | [MPLS](13-mpls.md) L3VPN |
| L2VPN EVPN | 25 / 70 | [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) |

The session can run over IPv4 and carry IPv6, or the reverse. IPv6 example:

```text
! Cisco IOS
router bgp 64512
 neighbor 2001:db8:100::1 remote-as 65001
 address-family ipv6
  network 2001:db8:200::/48
  neighbor 2001:db8:100::1 activate
```

BGP's job has grown. It is now a general way to share reachability data of many kinds, not just internet routes.

### Add-Path

A route reflector normally hides all but one path. **Add-Path** (RFC 7911) lets it send several paths for a prefix, each with a path ID. This gives faster failover, better load sharing and less risk of hidden paths.

**Optimal Route Reflection** solves a related problem. It lets an RR pick the best path as if it stood at the client's position, not its own. Support varies by vendor.

### Session security

| Method | What it does |
|---|---|
| MD5 password (RFC 2385) | Old, weak, still very common. `neighbor X password` |
| TCP-AO (RFC 5925) | The modern replacement for MD5. Needs platform support |
| GTSM (RFC 5082) | Send with TTL 255, and accept only packets whose TTL is still close to 255. A far-away attacker cannot fake that, since each hop lowers the TTL. Cisco: `neighbor X ttl-security hops 1` |
| ACL for TCP 179 | Only allow known peers. See [ACLs](../09-security/05-access-control-lists.md) |

Note that `ttl-security` and `ebgp-multihop` cannot be used together on the same neighbour on Cisco.

**eBGP multihop:** eBGP packets have TTL 1 by default, so the peer must be one hop away. To peer between loopbacks, use `neighbor X ebgp-multihop 2` along with `update-source`, and add a static route to the peer's loopback.

### DDoS tools

- **RTBH (Remotely Triggered Black Hole):** during a DDoS attack, you announce the target's /32 to your ISP with the BLACKHOLE community. The ISP drops all traffic to that address. The target goes offline, but the rest of your network survives. See [Common Network Attacks](../09-security/03-common-network-attacks.md).
- **BGP FlowSpec (RFC 8955):** carries firewall-like rules in BGP, for example "drop UDP port 123 to this address." It is more exact than RTBH, but a bad rule spreads fast.

### BGP in other places

- **Data centres:** large fabrics use eBGP between all switches, often with a private ASN per leaf (RFC 7938). It is simple, has clean failure behaviour, and supports ECMP well. See [Spine-Leaf](../10-enterprise-and-datacenter/02-spine-leaf.md). Many use **unnumbered** BGP, where peers use IPv6 link-local addresses, and IPv4 routes can use IPv6 next hops (RFC 8950).
- **Cloud links:** AWS Direct Connect, Azure ExpressRoute and Google Cloud Interconnect all use BGP. See [Hybrid Cloud Connectivity](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).
- **Kubernetes:** Calico and MetalLB can announce pod or service addresses with BGP. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).
- **Anycast:** many sites announce the same prefix, and BGP takes users to a nearby one. DNS root servers, 1.1.1.1 and 8.8.8.8 work this way. See [CDN](../10-enterprise-and-datacenter/06-cdn.md).
- **IXP route servers (RFC 7947):** instead of hundreds of sessions, a network peers with one route server at an exchange. The server passes on routes without adding its own ASN.
- **Monitoring:** BMP (BGP Monitoring Protocol, RFC 7854) streams a router's BGP data to a collector, so you can see what peers really sent.

## Common Mistakes

- **"BGP picks the fastest path."** It does not know speed or latency. It uses policy, then AS_PATH length. A path through 2 ASes can be slower than a path through 4.
- **"Active state means the session is working."** Active means TCP is failing. Look for Established.
- **"Higher MED is better."** Lower MED wins. Higher LOCAL_PREF wins. Mixing these up is very common.
- **"MED works across the whole internet."** It only goes to a neighbour AS, and it is only a hint.
- **"Weight is a BGP attribute."** It is a Cisco local value. It is never sent to a peer.
- **"LOCAL_PREF goes to the ISP."** It stays inside the AS (and across a confederation).
- **"iBGP changes the next hop."** It does not by default. Routes then look fine, but they are not usable, and this is the most common real iBGP bug. Use `next-hop-self`.
- **"iBGP peers pass routes along like eBGP."** They do not. You need a full mesh, RRs or a confederation.
- **"A route reflector forwards all the traffic."** It only reflects route information. Traffic usually goes directly between edge routers.
- **"One RR is enough."** One RR is a single point of failure for the control plane.
- **"Prepending fully controls inbound traffic."** Others can ignore it with LOCAL_PREF.
- **"Redistribute BGP into OSPF."** Pushing a full table into an IGP can crash it. Use a default route.
- **"RPKI stops all hijacks."** It checks only the origin AS. It does not check the path, and it does not stop leaks.
- **"Drop all routes that are not Valid."** Drop Invalid only. NotFound is normal for many prefixes.
- **"The `network` command is like OSPF's."** In BGP, it does not turn on BGP on an interface. It announces a prefix that must already be in the routing table, with the exact mask.
- **"Facebook's outage was a BGP failure."** BGP was only a step in the chain. The first cause was a backbone change.
- **"No filter is fine since the ISP filters me."** Every network should filter both ways. Your mistake must stop at your own edge.
- **"Peering means free internet access."** A peer gives access only to its own routes and its customers' routes, not the whole internet.

## Interview Questions

**Q: Why does the internet use BGP and not OSPF?**

A: OSPF is built for one network that trusts itself. It floods full topology to all routers and picks the lowest cost. The internet has many independent companies that do not share topology, and they need to make decisions based on business policy. BGP is a path vector protocol. It shares only prefixes and paths, it scales to around a million routes, and each network can apply its own policy.

**Q: What are the BGP session states, and what does Active mean?**

A: The states are Idle, Connect, Active, OpenSent, OpenConfirm and Established. Active means the TCP connection failed, so the router is trying again and listening. If a session stays in Active, I check IP reachability, the peer address, firewall rules on TCP 179, and update-source. Established is the good state.

**Q: What is the difference between eBGP and iBGP?**

A: eBGP runs between different ASes, and iBGP runs within one AS. eBGP adds the ASN to the AS_PATH and changes the next hop to itself. iBGP keeps the AS_PATH and next hop the same, and it carries LOCAL_PREF. eBGP packets use TTL 1 by default. On Cisco, the administrative distance is 20 for eBGP and 200 for iBGP. iBGP also has a split horizon rule, so a route from one iBGP peer is not passed to another.

**Q: Walk me through the BGP best path selection.**

A: First the next hop must be reachable. Then, on Cisco: highest weight, highest LOCAL_PREF, locally originated, shortest AS_PATH, lowest origin, lowest MED, eBGP over iBGP, lowest IGP metric to next hop. After that come tie breakers: oldest eBGP path, lowest router ID, shortest cluster list and lowest neighbor address. I remember it with "We Love Oranges AS Oranges Mean Pure Refreshment."

**Q: Why does iBGP need a full mesh, and how do you avoid it?**

A: The AS_PATH does not change inside an AS, so BGP cannot detect a loop there. To stay safe, a router must not send a route from one iBGP peer to another. So each router must peer with every other router, which is n(n-1)/2 sessions. To avoid this, I use route reflectors, or confederations in some cases. A route reflector passes routes to its clients and uses ORIGINATOR_ID and CLUSTER_LIST to stop loops.

**Q: You have two ISPs. How do you control outbound and inbound traffic?**

A: For outbound I set LOCAL_PREF inbound on routes from the preferred ISP. This is fully under my control. For inbound I can only influence others. I can prepend my ASN toward the backup ISP, use MED if both links go to the same ISP, use the ISP's communities, or announce a more-specific prefix on the preferred link. A more-specific prefix is the strongest, because longest prefix match comes first.

**Q: What are BGP communities, and how are they used?**

A: A community is a tag on a route that policy can act on. A common use is to let a customer signal an ISP, such as "treat this as backup," which the ISP turns into a lower LOCAL_PREF. There are well-known ones too, such as NO_EXPORT, NO_ADVERTISE, and BLACKHOLE for dropping DDoS traffic. On Cisco you must enable `send-community`, or they are not sent.

**Q: What is a BGP hijack, and what is the difference from a route leak?**

A: A hijack is when a network announces addresses that are not its own, so traffic goes to the wrong place. A route leak is when real routes are passed beyond their allowed scope, like a customer sending one provider's routes to another provider. A leak is often an honest mistake, and the origin stays correct. Pakistan Telecom and YouTube in 2008 is the classic hijack case.

**Q: What does RPKI do, and what does it not do?**

A: RPKI lets a prefix owner publish a signed ROA, which says which AS may originate a prefix and up to what length. Routers get the checked data from a validator and mark each route Valid, Invalid or NotFound. Most networks drop Invalid. RPKI checks only the origin, so it does not catch a forged path or a route leak, and it only helps when many networks use it.

**Q: What happened in the Facebook outage of October 2021?**

A: A maintenance command by mistake took down the links between Facebook's data centres, and an audit tool failed to stop it. Facebook's DNS servers then lost their link to the data centres, and by design they withdrew their BGP routes. So the authoritative DNS servers vanished from the internet. Remote tools failed too, and staff had to go on site, so the outage lasted around six hours. BGP spread the failure, but it was not the root cause.

**Q: A route shows in `show ip bgp`, but it is not in the routing table. Why?**

A: The usual causes are: the next hop is not reachable, often a missing `next-hop-self` in iBGP; or another source with lower administrative distance has the same prefix, which Cisco shows as RIB-failure; or a different path was chosen as best. I check `show ip bgp <prefix>` and then `show ip route <next hop>`.

**Q: Is BGP a Layer 3 or a Layer 7 protocol?**

A: It depends on what you mean. It runs on top of TCP port 179, so as a message it is an application in the TCP/IP stack. But its purpose is to build Layer 3 routing tables, so it belongs to the network control plane. A good answer gives both points.

## Quick Recap

- BGP connects Autonomous Systems. It is a path vector protocol that runs on TCP 179 and picks paths by policy, not speed.
- A session goes through Idle, Connect, Active, OpenSent, OpenConfirm and Established. The messages are OPEN, UPDATE, NOTIFICATION, KEEPALIVE and ROUTE-REFRESH.
- Best path order: next hop check, Weight, LOCAL_PREF, Originated, AS_PATH, Origin, MED, eBGP over iBGP, IGP metric, then tie breakers. "We Love Oranges AS Oranges Mean Pure Refreshment."
- iBGP needs a full mesh because of its split horizon rule. Route reflectors and confederations solve the scaling problem.
- You control outbound with LOCAL_PREF. You only influence inbound with prepending, MED, communities and more-specific prefixes.
- BGP trusts by default, so filter both ways. Hijacks and leaks are real, and RPKI checks the origin only.
- The Facebook outage shows that BGP can spread a failure that started somewhere else, and that recovery paths must not depend on the broken network.
