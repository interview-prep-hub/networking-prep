# MPLS

**Level:** Advanced | **Prerequisites:** [How Routers Work](01-how-routers-work.md), [The Routing Table](02-routing-table.md), [OSPF](06-ospf.md), [BGP](09-bgp.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md)

## In One Line

MPLS (Multiprotocol Label Switching) puts a short label in front of a packet, and routers forward the packet by that label instead of reading its IP address at every hop. Providers use this to build private VPNs, control traffic paths, and keep customer routes out of the core.

## Why It Matters

A normal router reads the destination IP address of every packet. It then finds the longest matching prefix in its routing table. A full internet table has about 1 million IPv4 routes. Every core router in a provider must hold that table, and that table says nothing about which customer a packet belongs to.

This creates three real problems:

1. **Overlapping customers.** Customer A and Customer B both use 192.168.1.0/24. One shared routing table cannot keep the two apart.
2. **Core routers know too much.** Every core router must carry every route. Memory, CPU and convergence time all suffer.
3. **No path control.** IP routing sends traffic down the shortest path, even when that path is full and a longer one is empty.

MPLS solves all three:

- A label can mean "this packet belongs to Customer A".
- Core routers only need routes to other provider routers.
- A label path can follow any route the operator chooses.

Interviewers ask about MPLS for ISP, carrier, large enterprise and network engineer roles. They want to know if you understand the difference between the control plane and the data plane. They often ask you to walk through a packet, explain RD vs RT, explain penultimate hop popping, and compare MPLS with SD-WAN.

**Short history.** In the 1990s, the main goal was speed, because a label lookup was faster than an IP lookup in the routers of that time. Cisco called its version tag switching. The IETF standardised MPLS as RFC 3031 and RFC 3032 in 2001. Modern hardware does IP lookups at full speed, so speed is no longer the reason. The reason today is services: VPNs, traffic engineering and a simpler core.

## Simple Explanation

Think of a large courier company.

Without labels, each sorting hub opens the box, reads the full home address, and looks it up in a giant book of every street in the country. This works, but every hub needs the giant book.

With labels, the first hub reads the address once. It then puts a small sticker on the box, for example "21". The next hub does not read the address. It only reads the sticker. Its small table says: "Sticker 21 goes to Bay 4, and replace it with sticker 22." The last hub removes the sticker and delivers the box.

The courier also has business customers. Two companies both have a "Room 101". The first hub adds a second sticker under the first one. This inner sticker says which company the box belongs to. Middle hubs never look at it. Only the last hub reads it.

**Where the analogy stops being true:**

- A real sticker is often global. An MPLS label has meaning only to the router that chose it. The number changes at each hop.
- Hubs in the analogy could peek inside the box. MPLS core routers can read the IP header too, for hashing and other reasons, but they do not need it for forwarding.
- The sticker does not hide anything. MPLS gives separation, not secrecy.

## How It Works

### Key terms

Read these slowly. Everything else in this chapter uses them.

| Term | Full name | Plain meaning |
|---|---|---|
| Label | | A 20-bit number that tells the next router what to do |
| LSP | Label Switched Path | The one-way path a labelled packet follows |
| FEC | Forwarding Equivalence Class | A group of packets treated the same way, for example "all packets going to 10.255.0.4/32" |
| LSR | Label Switching Router | A router that forwards by label in the middle of the network |
| LER | Label Edge Router | A router at the edge, where packets enter or leave the MPLS network |
| LDP | Label Distribution Protocol | The protocol routers use to tell each other which label to use |
| LIB | Label Information Base | All labels learned from all neighbours |
| LFIB | Label Forwarding Information Base | The table used to forward labelled packets |
| FIB | Forwarding Information Base | The table used to forward IP packets |

Providers use a second set of names:

| Name | Full name | Role | Same as |
|---|---|---|---|
| CE | Customer Edge | Customer router. Knows nothing about MPLS | Not part of MPLS |
| PE | Provider Edge | Provider router facing the customer | LER |
| P | Provider | Provider core router | LSR |

**Control plane** means the work of building tables (LDP, OSPF, BGP). **Data plane** means the work of moving packets using those tables. Keep them apart in your head. Many MPLS questions are really about this split.

### Where MPLS sits

The label goes between the Ethernet header and the IP header. For this reason, people often call MPLS "Layer 2.5".

```text
Normal IP packet on Ethernet:

+----------+--------+---------+------+
| Ethernet | Type   | IP      | Data |
| header   | 0x0800 | header  |      |
+----------+--------+---------+------+

Same packet with one MPLS label:

+----------+--------+---------+---------+------+
| Ethernet | Type   | MPLS    | IP      | Data |
| header   | 0x8847 | label   | header  |      |
+----------+--------+---------+---------+------+
```

The Ethernet Type value 0x8847 tells the receiver that a label comes next. (Value 0x8848 is used for upstream-assigned labels, and was first defined for multicast.)

**Fuller truth:** "Layer 2.5" is a nickname, not part of the OSI model. MPLS also runs over links that are not Ethernet. The "Multiprotocol" part of the name means it can carry IPv4, IPv6, Ethernet frames and more.

### The MPLS header

Each label entry is exactly 32 bits (4 bytes).

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Label (20 bits)          | TC  |S|   TTL (8)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Bits | Meaning |
|---|---|---|
| Label | 20 | The label value, 0 to 1,048,575 |
| TC | 3 | Traffic Class, used for QoS. Older documents call it EXP (RFC 5462 renamed it) |
| S | 1 | Bottom of Stack. 1 means this is the last label. 0 means another label follows |
| TTL | 8 | Time To Live. Stops loops, same idea as the IP TTL |

Labels 0 to 15 are reserved. Normal labels start at 16. The ones to remember:

| Label | Name | Use |
|---|---|---|
| 0 | IPv4 Explicit Null | Sent on the wire. Means "pop me here, this is IPv4" |
| 1 | Router Alert | Send to the router CPU for special handling |
| 2 | IPv6 Explicit Null | Same as 0, for IPv6 |
| 3 | Implicit Null | Used only in signalling. Never seen on the wire. Means "pop before sending to me" |

The MPLS header has no field that says what is inside it. There is no "next protocol" number. The router that pops the last label must already know the payload type, and it knows this from what the label was bound to.

### The label stack

A packet can carry more than one label. This group of labels is the **label stack**.

```text
+----------+--------+-----------+-----------+---------+------+
| Ethernet | Type   | Label 21  | Label 30  | IP      | Data |
| header   | 0x8847 | S=0       | S=1       | header  |      |
+----------+--------+-----------+-----------+---------+------+
                      ^           ^
                      |           |
             top (outer) label    bottom (inner) label
             used by core         used by the last PE
```

Routers act only on the **top** label. Lower labels wait their turn.

Think of a letter inside an envelope inside another envelope. The postal system reads only the outer envelope. The person at the destination opens it and reads the one inside. The analogy stops here: routers can look deeper when they want to, and they do so for load sharing.

This idea is the source of MPLS power. The outer label answers "which PE router?". The inner label answers "which customer?". The core needs only the first answer.

### Push, swap, pop

There are only three label actions:

| Action | What happens | Who does it |
|---|---|---|
| Push | Add a label | Ingress LER, where the packet enters |
| Swap | Replace the top label with a new one | LSR in the middle |
| Pop | Remove the top label | Second-to-last router, or the egress LER |

**Ingress** means the entry point. **Egress** means the exit point.

```text
              push 21        swap 21->22     pop 22
 IP packet   +-------+      +-------+      +-------+    IP packet
 ----------> |  LER  | ---> |  LSR  | ---> |  LSR  | ---> LER --->
             +-------+  21  +-------+  22  +-------+  no label
              ingress                      second-to-last  egress
```

A P router does an **exact match** on the label, not a longest prefix match. It reads the label, finds that one row in the LFIB, changes the label, and sends the packet out. It does not read the destination IP address to make this choice.

An LSP is **one way**. Two-way traffic needs two LSPs, and they can take different paths.

### Labels are local

Each router picks its own label for each FEC. The router that will **receive** labelled traffic picks the label, then tells its upstream neighbour. This is called downstream allocation.

Compare it to a phone extension. You tell your friend, "To reach me, dial extension 22." Your friend's extension may be 21. Nobody needs a single global list. The analogy stops being true at one point: a label is only an instruction for one hop, so it is replaced at each router.

Two routers can use the same number with different meanings. This is normal.

### LDP: how routers agree on labels

LDP (RFC 5036) gives each IGP prefix a label on each router. **IGP** means Interior Gateway Protocol, such as OSPF or IS-IS. LDP does not choose paths. It follows the IGP. If OSPF says the next hop is P1, the label from P1 is the one used.

Steps:

1. **Discovery.** Routers send LDP Hello messages on UDP port 646 to 224.0.0.2 (all routers on this link).
2. **Session.** The neighbours form a TCP session on port 646. The router with the higher transport address opens the connection. The transport address is usually the LDP router ID, so it is usually a loopback address. Each loopback must be reachable.
3. **Initialisation.** The routers agree on session settings and start sending keepalives.
4. **Address exchange.** Each router lists its IP addresses. This lets a neighbour link "next hop 10.0.12.2" to "that LDP peer".
5. **Label exchange.** Each router sends Label Mapping messages: "for this FEC, use my label N".
6. **Install.** All received labels go into the LIB. The label from the IGP next hop goes into the LFIB.

```text
Traffic direction ---------------------------------------->

  PE1              P1               P2              PE2
                                                 10.255.0.4/32
       <-- use 21       <-- use 22       <-- imp-null

<--------------------- Labels are advertised this way
```

An LDP ID looks like `10.255.0.2:0`. The first part is the router ID. The `:0` means the labels are shared by the whole router (a platform-wide label space).

**Vendor defaults (these vary):**

| Item | Cisco IOS default |
|---|---|
| Hello interval / hold time | 5 s / 15 s |
| Keepalive interval / session hold time | 60 s / 180 s |
| Label range | Often 16 to 100000, depends on platform. Check `show mpls label range` |
| Prefixes that get a label | All IGP prefixes. Junos by default labels only its loopback |

### Penultimate hop popping

"Penultimate" means second to last. In **penultimate hop popping (PHP)**, the second-to-last router removes the top label, so the last router does not have to.

Why? Without PHP, the egress router does two lookups for one packet:

1. A label lookup, which says "pop this, it is for me".
2. An IP lookup, to find where the packet goes next.

With PHP, the second-to-last router pops the label. The egress router receives a plain IP packet and does one lookup.

Think of a taxi carrying a parcel with a company tag. The driver removes the tag one stop before the end, so the receiver does not spend time cutting it off. The analogy stops here: on modern hardware the time saved is small, and PHP remains the default mostly because it is the established behaviour.

How it is signalled: the egress router advertises **label 3 (Implicit Null)** for its own prefixes. Its upstream neighbour reads this as "pop, do not swap".

```text
Without PHP                       With PHP (default)

P2 swaps 22 -> 16                 P2 pops 22
PE2 gets [16][IP]                 PE2 gets [IP]
PE2: label lookup, then IP lookup PE2: IP lookup only
```

Two important details:

- PHP removes only the **top** label. In an L3VPN, the VPN label stays on the packet. PE2 then does one lookup on the VPN label.
- Popping the label also removes its TC value. If the provider needs that QoS marking at the egress router, the egress can advertise **label 0 (Explicit Null)** instead. Then a label with value 0 arrives on the wire, carrying the TC bits. On Cisco IOS the command is `mpls ldp explicit-null`.

### MPLS L3VPN

**L3VPN** (RFC 4364) lets a provider carry many customers over one core. Each customer sees a private routed network. Customers can use the same IP addresses.

It needs four parts:

| Part | Job |
|---|---|
| VRF | A separate routing table on the PE, one per customer |
| RD | Makes each customer route unique inside BGP |
| RT | Decides which VRFs accept which routes |
| MP-BGP | Carries customer routes and VPN labels between PEs |

**VRF (Virtual Routing and Forwarding)** is like a separate router inside the PE. A customer-facing interface belongs to one VRF.

**MP-BGP (Multiprotocol BGP, RFC 4760)** extends BGP so it can carry more than plain IPv4 routes. Here it carries VPNv4 routes (AFI 1, SAFI 128). AFI and SAFI are the numbers BGP uses to name the type of route.

#### RD: Route Distinguisher

Customer A and Customer B both have 192.168.1.0/24. If both go into one BGP table, BGP sees one prefix and picks one best path. One customer loses.

The **RD** is a 64-bit value placed in front of the IPv4 prefix. The result is a 96-bit **VPNv4** address:

```text
RD (64 bits)   IPv4 prefix (32 bits)    VPNv4 route
64512:100    + 192.168.1.0/24       =   64512:100:192.168.1.0/24
64512:200    + 192.168.1.0/24       =   64512:200:192.168.1.0/24
```

These are now two different routes. Think of two apartment towers, and both have Flat 5. Adding the tower name makes each address unique. The analogy stops being true at this point: the RD is not a location, and it does not say where a route may go. It only makes the route unique.

#### RT: Route Target

The **RT** is a BGP extended community (an 8-byte tag attached to a route). It controls which VRFs get the route.

- **Export RT:** the tag a VRF attaches to the routes it sends.
- **Import RT:** the VRF installs any route with a tag on its import list.

Think of a party guest list. A route arrives with a badge. The VRF checks its list. Badge on the list, the route comes in. The analogy stops here: one route can carry many badges, and one VRF can accept many badges.

| | RD | RT |
|---|---|---|
| Purpose | Make the route unique | Control route import and export |
| How many | One per VRF on a PE | One or many per VRF and per route |
| Used for filtering | No | Yes |
| Form | Prefix of the VPNv4 address | BGP extended community |

**Mental check:** RD is about **identity**. RT is about **membership**.

Because RT is separate from RD, you can build designs other than full mesh:

```text
Hub and spoke (spokes talk only through the hub)

              exports RT 64512:300   imports RT 64512:301
                          +-------+
                          |  Hub  |
                          +-------+
                         /         \
                  +---------+   +---------+
                  | Spoke 1 |   | Spoke 2 |
                  +---------+   +---------+
Each spoke:   exports RT 64512:301   imports RT 64512:300

Spokes do not import 64512:301, so they do not learn each
other's routes directly.
```

#### Two labels, two jobs

| Label | Also called | Learned from | Meaning |
|---|---|---|---|
| Outer | Transport label | LDP (or RSVP-TE, or Segment Routing) | Which PE to reach |
| Inner | VPN label | MP-BGP | Which VRF or prefix on that PE |

#### L3VPN control plane, step by step

1. CE2 sends 192.168.2.0/24 to PE2. The protocol can be eBGP, OSPF, static or others.
2. PE2 stores it in VRF CUSTOMER-A.
3. PE2 builds the VPNv4 route, attaches RT 64512:100, and allocates VPN label 30.
4. PE2 sends it by MP-iBGP to PE1, with its own loopback as next hop.
5. PE1 checks the RT. It matches the import list of VRF CUSTOMER-A.
6. PE1 installs the route in that VRF, with next hop 10.255.0.4 and VPN label 30.
7. PE1 sends the route to CE1.

P routers are not part of this. They hold no VRFs and no customer routes.

### Traffic engineering (high level)

**Traffic engineering (TE)** means moving traffic onto paths other than the shortest one, to use the network better.

Plain IGP routing has a well-known weakness, often called the **fish problem**:

```text
                +---- R3 ---- R4 ----+     long path, 10G, empty
                |                    |
  R1 ---------- R2                   R6 ---------- R7
                |                    |
                +-------- R5 --------+     short path, 10G
```

Two flows go from R1 to R7, at 8 Gbps and 6 Gbps. Both take the short path. That is 14 Gbps on 10 Gbps links, so packets drop. The long path stays empty.

With MPLS TE, the operator sends the 8 Gbps flow through R5 and the 6 Gbps flow through R3 and R4.

The classic method has four parts:

1. **Link data.** OSPF or IS-IS extensions flood each link's bandwidth and other attributes.
2. **Path calculation.** The ingress router runs CSPF (Constrained Shortest Path First). CSPF removes links that fail the rules, for example "not enough free bandwidth", then runs shortest path on what is left.
3. **Signalling.** RSVP-TE (RFC 3209) sets up the path hop by hop, reserves bandwidth, and assigns labels.
4. **Forwarding.** Traffic enters the tunnel, which is simply an LSP.

Think of a convoy that books a route with the road authority before leaving. The analogy stops here: the bandwidth "reservation" is only bookkeeping in the control plane. It does not police traffic on the wire. A tunnel can send more than it reserved unless other tools stop it.

TE also gives **Fast Reroute (FRR, RFC 4090)**: a backup path is built in advance, so the router next to a failure switches traffic without waiting for the network to reconverge. The commonly quoted target is about 50 ms. Real results depend on how fast the failure is detected.

Today, many networks use Segment Routing instead. See the Going Deeper section.

### MPLS vs SD-WAN

**SD-WAN** (Software-Defined WAN) builds encrypted tunnels over any transport, such as internet or LTE or MPLS. A central controller manages policy. For full detail see [SD-WAN](../10-enterprise-and-datacenter/10-sd-wan.md).

Think of MPLS as a private toll road run by one company, with a promised travel time. SD-WAN is a smart GPS with armoured cars, which can use any road and picks the best one at the moment. The analogy stops here: an MPLS "private road" still shares the same physical core with other customers. It is logical separation.

| Point | MPLS L3VPN | SD-WAN |
|---|---|---|
| Who runs it | Usually a carrier | Usually the enterprise or a managed partner |
| Transport | Provider's own network | Any: internet, LTE or 5G, MPLS |
| Encryption | None by default | Usually IPsec on all tunnels |
| Quality promise | Contract SLA on delay, loss, uptime | Depends on the underlay. Internet has no real SLA |
| Path choice | Set by provider routing | Per application, based on measured loss and delay |
| Cloud and SaaS access | Often goes through a central data centre | Direct local internet exit is common |
| New site | Often weeks, needs a provider circuit | Can be fast over existing internet |
| Bandwidth cost | Usually higher per Mbps | Usually lower per Mbps |
| Where it is strongest | Stable delay for voice and critical apps | Flexibility, cost, cloud access |

They are not true opposites. MPLS is a transport. SD-WAN is an overlay, and it can run on top of MPLS. Many companies use both: MPLS for critical traffic and internet for the rest.

**Fuller truth:** prices and delivery times depend on the country and the provider. Treat the table as the general pattern. Also, MPLS the technology and "an MPLS circuit" the product are different things. Carriers still use MPLS inside their backbones, even when a customer buys SD-WAN over internet.

## Example

### Topology

One provider (AS 64512). Customer A has two sites. Customer B also uses 192.168.1.0/24.

```text
Customer A, Site 1                        Customer A, Site 2
192.168.1.0/24, AS 65001                  192.168.2.0/24, AS 65002

  +-----+                                        +-----+
  | CE1 |                                        | CE2 |
  +-----+                                        +-----+
     | .2                                           | .2
     | 172.16.1.0/30                  172.16.2.0/30 |
     | .1 Gi0/0                               Gi0/0 | .1
  +-----+       +----+       +----+          +-----+
  | PE1 |-------| P1 |-------| P2 |----------| PE2 |
  +-----+       +----+       +----+          +-----+
      10.0.12.0/30  10.0.23.0/30  10.0.34.0/30
      (.1 PE1,      (.1 P1,       (.1 P2,
       .2 P1)        .2 P2)        .2 PE2)

  |<------------ Provider AS 64512 -------------->|
```

Loopbacks and interfaces:

| Router | Loopback0 | Interfaces |
|---|---|---|
| PE1 | 10.255.0.1/32 | Gi0/0 172.16.1.1 (to CE1), Gi0/1 10.0.12.1 (to P1) |
| P1 | 10.255.0.2/32 | Gi0/0 10.0.12.2 (to PE1), Gi0/1 10.0.23.1 (to P2) |
| P2 | 10.255.0.3/32 | Gi0/0 10.0.23.2 (to P1), Gi0/1 10.0.34.1 (to PE2) |
| PE2 | 10.255.0.4/32 | Gi0/0 172.16.2.1 (to CE2), Gi0/1 10.0.34.2 (to P2) |

Customer data:

| Customer | RD | RT | Site 1 |
|---|---|---|---|
| CUSTOMER-A | 64512:100 | 64512:100 | 192.168.1.0/24 |
| CUSTOMER-B | 64512:200 | 64512:200 | 192.168.1.0/24 (overlap is fine) |

### Configuration (Cisco IOS XE)

PE1:

```text
ip cef
mpls ldp router-id Loopback0 force
!
vrf definition CUSTOMER-A
 rd 64512:100
 route-target export 64512:100
 route-target import 64512:100
 !
 address-family ipv4
 exit-address-family
!
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
!
interface GigabitEthernet0/0
 description To CE1
 vrf forwarding CUSTOMER-A
 ip address 172.16.1.1 255.255.255.252
!
interface GigabitEthernet0/1
 description To P1
 ip address 10.0.12.1 255.255.255.252
 mpls ip
!
router ospf 1
 router-id 10.255.0.1
 network 10.255.0.1 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0
!
router bgp 64512
 bgp router-id 10.255.0.1
 no bgp default ipv4-unicast
 neighbor 10.255.0.4 remote-as 64512
 neighbor 10.255.0.4 update-source Loopback0
 !
 address-family vpnv4
  neighbor 10.255.0.4 activate
  neighbor 10.255.0.4 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER-A
  redistribute connected
  neighbor 172.16.1.2 remote-as 65001
  neighbor 172.16.1.2 activate
 exit-address-family
```

Note the order: `vrf forwarding` removes any IP address already on the interface, so set the VRF first and the address second.

P1 has no BGP and no VRF:

```text
mpls ldp router-id Loopback0 force
!
interface Loopback0
 ip address 10.255.0.2 255.255.255.255
!
interface GigabitEthernet0/0
 ip address 10.0.12.2 255.255.255.252
 mpls ip
!
interface GigabitEthernet0/1
 ip address 10.0.23.1 255.255.255.252
 mpls ip
!
router ospf 1
 router-id 10.255.0.2
 network 10.255.0.2 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
```

P2 is the same with its own addresses. PE2 mirrors PE1, peers with 10.255.0.1, and has CE2 (172.16.2.2, AS 65002) as its VRF neighbour.

CE1 runs no MPLS:

```text
router bgp 65001
 network 192.168.1.0
 neighbor 172.16.1.1 remote-as 64512
```

### Check LDP (Cisco IOS)

```text
P1# show mpls interfaces
Interface              IP            Tunnel   BGP Static Operational
GigabitEthernet0/0     Yes (ldp)     No       No  No     Yes
GigabitEthernet0/1     Yes (ldp)     No       No  No     Yes

P1# show mpls ldp neighbor 10.255.0.1
    Peer LDP Ident: 10.255.0.1:0; Local LDP Ident 10.255.0.2:0
        TCP connection: 10.255.0.1.646 - 10.255.0.2.39125
        State: Oper; Msgs sent/rcvd: 58/57; Downstream
        Up time: 00:21:14
        LDP discovery sources:
          GigabitEthernet0/0, Src IP addr: 10.0.12.1
        Addresses bound to peer LDP Ident:
          10.255.0.1      10.0.12.1
```

The TCP line shows port 646 on PE1 and a random port on P1. P1 has the higher address (10.255.0.2), so P1 opened the connection.

### Read the label tables (Cisco IOS)

```text
P2# show mpls ldp bindings 10.255.0.4 32
  lib entry: 10.255.0.4/32, rev 10
        local binding:  label: 22
        remote binding: lsr: 10.255.0.4:0, label: imp-null
        remote binding: lsr: 10.255.0.2:0, label: 21

P1# show mpls ldp bindings 10.255.0.4 32
  lib entry: 10.255.0.4/32, rev 12
        local binding:  label: 21
        remote binding: lsr: 10.255.0.3:0, label: 22
        remote binding: lsr: 10.255.0.1:0, label: 20
```

P1 keeps PE1's label 20, even though PE1 is not the next hop. This is **liberal label retention**. If the IGP path changes, the label is already there.

```text
P1# show mpls forwarding-table
Local     Outgoing   Prefix          Bytes Label  Outgoing  Next Hop
Label     Label      or Tunnel Id    Switched     interface
18        Pop Label  10.255.0.1/32   83904        Gi0/0     10.0.12.1
19        Pop Label  10.255.0.3/32   0            Gi0/1     10.0.23.2
20        Pop Label  10.0.34.0/30    0            Gi0/1     10.0.23.2
21        22         10.255.0.4/32   84312        Gi0/1     10.0.23.2

P2# show mpls forwarding-table 10.255.0.4 32
Local     Outgoing   Prefix          Bytes Label  Outgoing  Next Hop
Label     Label      or Tunnel Id    Switched     interface
22        Pop Label  10.255.0.4/32   84312        Gi0/1     10.0.34.2
```

How to read a row: "If a packet arrives with label 21, change it to 22 and send it out Gi0/1 to 10.0.23.2." `Pop Label` means this router is the second-to-last hop for that prefix.

### Read the VPN routes (Cisco IOS)

```text
PE1# show bgp vpnv4 unicast all
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 64512:100 (default for vrf CUSTOMER-A)
 *>  172.16.1.0/30    0.0.0.0                  0         32768 ?
 *>i 172.16.2.0/30    10.255.0.4               0    100      0 ?
 *>  192.168.1.0      172.16.1.2               0             0 65001 i
 *>i 192.168.2.0      10.255.0.4               0    100      0 65002 i
Route Distinguisher: 64512:200 (default for vrf CUSTOMER-B)
 *>  172.16.3.0/30    0.0.0.0                  0         32768 ?
 *>  192.168.1.0      172.16.3.2               0             0 65003 i
```

Prefix 192.168.1.0 appears twice with no conflict, because the RDs are different.

```text
PE1# show bgp vpnv4 unicast vrf CUSTOMER-A 192.168.2.0/24
BGP routing table entry for 64512:100:192.168.2.0/24, version 9
Paths: (1 available, best #1, table CUSTOMER-A)
  65002
    10.255.0.4 (metric 4) from 10.255.0.4 (10.255.0.4)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Extended Community: RT:64512:100
      mpls labels in/out nolabel/30

PE1# show ip cef vrf CUSTOMER-A 192.168.2.0/24
192.168.2.0/24
  nexthop 10.0.12.2 GigabitEthernet0/1 label 21 30

PE2# show mpls forwarding-table vrf CUSTOMER-A
Local   Outgoing   Prefix             Bytes Label  Outgoing  Next Hop
Label   Label      or Tunnel Id       Switched     interface
30      No Label   192.168.2.0/24[V]  500          Gi0/0     172.16.2.2
31      Aggregate  172.16.2.0/30[V]   0
```

The CEF line is the key output. `label 21 30` is the stack PE1 pushes: 21 outer, 30 inner. On PE2, `No Label` means "send as plain IP". `[V]` marks a VPN prefix. Exact output format varies by software version. CEF (Cisco Express Forwarding) is Cisco's name for its FIB-based forwarding.

### Follow one packet

Host 192.168.1.10 sends to 192.168.2.20. The host sets TTL to 64.

| Step | Router | Action | Stack leaving router | TTL |
|---|---|---|---|---|
| 1 | CE1 | Normal IP routing | None | IP 63 |
| 2 | PE1 | VRF lookup, push 30 then 21 | 21, 30 | Labels 62 |
| 3 | P1 | Swap 21 to 22 | 22, 30 | Top 61 |
| 4 | P2 | Pop 22 (PHP) | 30 | 60 |
| 5 | PE2 | Look up 30, pop, send to VRF next hop | None | IP 59 |
| 6 | CE2 | Normal IP routing | None | IP 58 |

```text
Link        Packet on the wire      What the sender did
CE1 -> PE1  [IP]                    normal IP routing
PE1 -> P1   [21][30][IP]            PE1 pushed 30, then 21
P1  -> P2   [22][30][IP]            P1 swapped 21 for 22
P2  -> PE2  [30][IP]                P2 popped 22 (PHP)
PE2 -> CE2  [IP]                    PE2 popped 30
```

The host receives TTL 58. Six routers forwarded the packet, and 64 - 6 = 58.

The two labels on the wire between PE1 and P1, as bytes:

```text
00 01 50 3E   label=21  TC=0  S=0  TTL=62
00 01 E1 3E   label=30  TC=0  S=1  TTL=62
45 00 ...     IPv4 header starts
```

Check the first entry. 0x00015 is 21 (the first 20 bits). The next hex digit is 0, so TC=0 and S=0. Last byte 0x3E is 62. In the second entry, the digit after 0x0001E is 1, so S=1.

Wireshark shows these layers (lines shortened here to fit):

```text
Ethernet II, Type: MPLS label switched packet (0x8847)
MPLS Header, Label: 21, Exp: 0, S: 0, TTL: 62
MPLS Header, Label: 30, Exp: 0, S: 1, TTL: 62
Internet Protocol Version 4, Src: 192.168.1.10, Dst: 192.168.2.20
```

Useful capture filters. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) and [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

```bash
# Linux: capture MPLS frames on a provider-side link
sudo tcpdump -ni eth0 mpls

# Linux: capture LDP hellos and sessions
sudo tcpdump -ni eth0 port 646
```

```text
Wireshark display filters (any OS):
mpls.label == 21
ldp
```

### Traceroute through the VPN (Cisco IOS)

```text
PE1# traceroute vrf CUSTOMER-A 192.168.2.20
Type escape sequence to abort.
Tracing the route to 192.168.2.20
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.12.2 [MPLS: Labels 21/30 Exp 0] 16 msec 16 msec 16 msec
  2 10.0.23.2 [MPLS: Labels 22/30 Exp 0] 16 msec 12 msec 16 msec
  3 172.16.2.1 [MPLS: Label 30 Exp 0] 12 msec 12 msec 12 msec
  4 172.16.2.2 16 msec 12 msec 16 msec
  5 192.168.2.20 16 msec 16 msec 16 msec
```

Look at hop 1 and hop 2: the delay is almost the same as the far end. P1 has no route to the source of this traceroute, because it holds no customer routes. It sends the ICMP error onward along the LSP. PE2 then routes it back. So the ICMP reply from P1 travels almost the whole path first.

### Check the path with LSP ping (Cisco IOS)

A normal ping can succeed over plain IP while the LSP is broken. LSP ping (RFC 8029) tests the label path itself.

```text
PE1# ping mpls ipv4 10.255.0.4/32
Sending 5, 100-byte MPLS Echos to 10.255.0.4/32,
     timeout is 2 seconds, send interval is 0 msec:

Type escape sequence to abort.
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/10/12 ms
```

### Linux can forward MPLS too

Hosts almost never see labels. But the Linux kernel has an MPLS data plane, which is useful for a lab. These commands make a Linux box act like P1. Routes are static here. A real deployment would add a control plane, such as FRRouting with its LDP daemon.

```bash
# Linux
sudo modprobe mpls_router
sudo modprobe mpls_iptunnel
sudo sysctl -w net.mpls.platform_labels=1000
sudo sysctl -w net.mpls.conf.eth0.input=1

# Swap: label 21 comes in, label 22 goes out
sudo ip -f mpls route add 21 as 22 via inet 10.0.23.2 dev eth1

# Ingress: push two labels on IP traffic (PE-style)
sudo ip route add 192.168.2.0/24 encap mpls 21/30 via 10.0.12.2 dev eth1

# Show the MPLS table
ip -f mpls route show
```

Windows has no built-in MPLS forwarding. On Windows, you can only inspect MPLS packets in a capture, for example by opening a capture file in Wireshark.

## Going Deeper

### LDP label modes

| Setting | Options | Common choice |
|---|---|---|
| Advertisement | Downstream Unsolicited (send labels without being asked), or Downstream on Demand | Unsolicited on IP networks |
| Retention | Liberal (keep all labels), or Conservative (keep only next-hop labels) | Liberal |
| Control | Independent (assign a label at once), or Ordered (wait for the downstream label first) | Cisco: Independent. Junos: Ordered |

### Why loopbacks must be /32

The LSP to a PE ends at its loopback /32. If an IGP summary or a wrong mask hides that /32, routers do not have a matching FEC for the BGP next hop, and the LSP breaks in the middle.

What failure looks like: P1 has no outgoing label for 10.255.0.4/32. Its LFIB says `No Label`. It removes the transport label and sends the packet on. The next router now sees label 30 at the top. That router did not create label 30, so it drops the packet. The control plane looks fine, BGP is up, and traffic disappears.

On Cisco IOS, OSPF advertises a loopback as a /32 by default. The risk comes from summarisation, or from a non-/32 mask on the loopback.

### LDP and IGP sync

The IGP can converge faster than LDP. After a link comes up, OSPF may send traffic over it before LDP has labels for it. VPN traffic then disappears, because a P router that receives the packet without its transport label has no VPN route and drops it.

LDP-IGP Synchronisation fixes this. The IGP advertises the link with maximum metric until LDP is ready.

```text
! Cisco IOS
router ospf 1
 mpls ldp sync
```

### TTL handling

By default, the ingress copies the IP TTL into the label TTL. Each LSR reduces it. The egress copies it back. This is called TTL propagation, and it is the reason a traceroute shows core hops.

Many providers hide their core:

```text
! Cisco IOS: stop copying TTL for forwarded customer traffic
no mpls ip propagate-ttl forwarded
```

The label TTL then starts at 255, and the whole core looks like one hop in a customer traceroute. RFC 3443 describes TTL processing in MPLS.

### MTU

Each label adds 4 bytes. An IP packet of 1500 bytes with two labels needs 1508 bytes on a core link. TE, FRR or other services can add more labels.

A common symptom: small pings work, and large transfers hang. Test it:

```text
! Cisco IOS
PE1# ping vrf CUSTOMER-A 192.168.2.20 size 1500 df-bit
```

Providers normally run the core with a large MTU, often 9000 bytes or more. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### QoS in MPLS

The 3 TC bits give 8 classes. P routers queue packets using the TC bits and do not read DSCP (the IP QoS field). Cisco's default on imposition copies the top 3 bits of DSCP into TC. For QoS basics see [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

RFC 3270 describes three tunnel models for marking:

| Model | Behaviour |
|---|---|
| Uniform | One QoS domain. Provider changes to marking are visible to the customer |
| Pipe | Provider marking is used across the core and at egress. The customer's marking stays unchanged |
| Short Pipe | Like Pipe, but the egress queues using the customer's own marking |

### Load sharing

Routers balance traffic across equal paths using a hash. Many P routers look past the label stack at the IP header. If the first bits after the stack look like IPv4 or IPv6, they hash those fields. This guess can go wrong when the payload is an Ethernet frame. Two tools address this: the pseudowire control word, and entropy labels (RFC 6790), which carry a hash value inside the stack.

### VPN design details

- **Route reflectors.** PEs do not need a full mesh of iBGP. They peer with VPNv4 route reflectors. See [BGP](09-bgp.md).
- **RT filter on PEs.** A PE drops VPNv4 routes whose RT no local VRF imports. This saves memory. A route reflector has no VRFs, so it turns this filter off and keeps all routes.
- **RD design.** Using a unique RD per PE per VRF lets a route reflector keep both paths for a dual-homed site. With the same RD, the reflector sees the same VPNv4 prefix and passes on only one best path.
- **Same AS at many sites.** If all CEs use AS 65001, normal eBGP loop prevention rejects routes that contain 65001. The PE can use `as-override`, or the CE can use `allowas-in`. Site of Origin (SoO) protects a dual-homed site from loops.
- **Label allocation.** A PE can assign one VPN label per prefix, per CE or per VRF. Defaults vary by vendor. Per-VRF uses fewer labels, but it forces an IP lookup in the VRF at egress.
- **OSPF as PE-CE protocol.** Possible, but it has special rules, such as the down bit and sham links for backdoor links.
- **Security.** PE-CE links do not run LDP, and a PE does not accept labelled packets from a CE on a plain VRF interface. A customer cannot inject a packet into another VPN this way. The traffic is still not encrypted. If you need secrecy, run [IPsec](../09-security/08-ipsec.md) over the VPN.

### BGP-free core for internet traffic

The same trick works without VPNs. A PE learns internet routes by BGP, with another PE's loopback as next hop. It pushes the LDP label for that loopback. P routers forward by label and do not need the 1 million routes.

### IPv6

| Method | Meaning | RFC |
|---|---|---|
| 6PE | Carry global IPv6 over an IPv4 MPLS core | 4798 |
| 6VPE | IPv6 VPNs in VRFs (VPNv6), with an IPv4 core | 4659 |
| LDPv6 | LDP running natively over IPv6 | 7552 |

### Layer 2 VPNs

MPLS can also carry Ethernet frames. The stack is a transport label plus a label that identifies the circuit.

| Service | Shape | Notes |
|---|---|---|
| VPWS (pseudowire) | Point to point | Acts like a virtual cable between two sites. Cisco calls it AToM, configured with `xconnect` |
| VPLS | Multipoint | Acts like one big switch across sites. RFC 4761 (BGP) and RFC 4762 (LDP) |
| EVPN | Multipoint | RFC 7432. Uses BGP to learn MACs. It is the modern choice over VPLS |

Targeted LDP sessions (between routers that are not direct neighbours) are used for LDP-signalled pseudowires. For EVPN in data centres, see [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

### Multi-domain and multicast

- **Inter-AS VPN.** RFC 4364 gives three options to link two providers. Option A: VRF-to-VRF back to back, simple but needs a VRF per customer at the border. Option B: border routers exchange VPNv4 routes. Option C: reflectors exchange VPNv4 routes and borders carry only labelled PE loopbacks.
- **Multicast.** mLDP and P2MP RSVP-TE build point-to-multipoint LSPs. See [Multicast Routing](14-multicast-routing.md).

### Segment Routing

**Segment Routing (SR, RFC 8402)** removes much of the old state and is the main direction for new designs.

- **SR-MPLS (RFC 8660)** keeps the same MPLS data plane. OSPF or IS-IS send the labels, so LDP is not needed.
- The ingress router lists the path as a stack of labels, called segments. Think of a trip plan written on the parcel: "go to hub 5, then hub 9, then home." The analogy stops being true when the label is a prefix segment: each router still picks the shortest path to that segment.
- Each node has a **prefix SID**, an index into a shared label block called the SRGB. Cisco's default SRGB is 16000 to 23999. So with index 4, a router may use label 16004 on every hop. Unlike LDP, the label is often the same on all routers. The fuller truth is that it stays the same only if all routers use the same SRGB.
- P routers keep no per-tunnel state. RSVP-TE keeps state for each tunnel at each router along the path.
- **TI-LFA** (Topology Independent Loop-Free Alternate) gives fast reroute without RSVP.
- **SRv6 (RFC 8986)** uses IPv6 headers instead of MPLS labels.

### Troubleshooting order

1. `show ip route 10.255.0.4` - is the /32 present on every router?
2. `show mpls interfaces` - is MPLS enabled and operational on each core link?
3. `show mpls ldp discovery` - are hellos sent and received?
4. `show mpls ldp neighbor` - state must be Oper.
5. `show mpls forwarding-table 10.255.0.4 32` - does each hop have a label or `Pop Label`?
6. `ping mpls ipv4 10.255.0.4/32` - does the LSP work end to end?
7. `show bgp vpnv4 unicast all summary` - are the PEs peered, and are routes received?
8. `show bgp vpnv4 unicast vrf CUSTOMER-A` - are routes imported? If not, compare RTs.
9. `ping vrf CUSTOMER-A 192.168.2.20` - finally, test the customer path.

## Common Mistakes

- **"MPLS is encrypted."** It is not. L3VPN separates traffic logically. A person with access to a core link can read the packets.
- **"MPLS is faster than IP."** This was the old reason. Modern hardware forwards IP just as fast. The value is in services.
- **"RD filters routes."** It does not. RD makes routes unique. RT decides import and export.
- **"Labels are global."** A label is local to the router that assigned it, and it normally changes at each hop. With Segment Routing the number is often the same on all routers, but LDP labels are not like that.
- **"LDP chooses the path."** LDP follows the IGP. If OSPF is wrong, the LSP is wrong.
- **"PHP removes all the labels."** It removes only the top label. The VPN label stays until the egress PE.
- **"Implicit null is a label on the wire."** Label 3 appears only in signalling. Explicit null, label 0, does appear on the wire.
- **"P routers need customer routes."** They do not. P routers hold no VRFs, and in a BGP-free core they do not run BGP at all.
- **"An LSP is two-way."** It is one-way. The return path is a separate LSP, and it may use a different route.
- **"S=1 means the top label."** It marks the bottom label, the last one in the stack.
- **Forgetting MPLS on one core link.** IGP looks fine, and the labels stop there. VPN traffic is dropped at that point.
- **Summarising PE loopbacks.** This breaks the /32 FEC, so LSPs break.
- **Ignoring MTU.** Each label adds 4 bytes. Small pings pass while large packets fail.
- **Setting an IP address before `vrf forwarding`.** The IP address is removed when the VRF is applied.
- **"SD-WAN replaces MPLS in every case."** SD-WAN can run on top of MPLS, and many networks use both. Internet paths have no end-to-end delay guarantee.

## Interview Questions

**Q: What is MPLS and why is it used today?**

A: MPLS forwards packets by a short label instead of an IP lookup at each hop. Speed was the first reason, but not today. It is used for L3VPN and L2VPN, traffic engineering, fast reroute, and a BGP-free core.

**Q: Describe the MPLS header.**

A: It is 32 bits: a 20-bit label, 3-bit Traffic Class for QoS, a 1-bit Bottom of Stack flag, and an 8-bit TTL. It sits between Layer 2 and the IP header. Ethernet uses type 0x8847 for it.

**Q: What is the difference between an LER and an LSR?**

A: An LER sits at the edge and pushes or removes labels as traffic enters or leaves. An LSR sits in the middle and mainly swaps labels. In provider terms, an LER is a PE and an LSR is a P router. A router can play both roles for different traffic.

**Q: How does LDP work?**

A: Routers find each other with Hello messages on UDP 646 to 224.0.0.2. They then form a TCP session on port 646, and the higher transport address opens it. They exchange addresses and label mappings. Each router keeps all labels in the LIB, and uses the one from the IGP next hop in the LFIB. LDP follows the IGP and does not compute paths.

**Q: What is penultimate hop popping and why does it exist?**

A: The second-to-last router removes the top label, so the egress router skips a label lookup and needs only one lookup. The egress asks for this by advertising implicit null, label 3. In a VPN, only the transport label goes. The VPN label stays for the egress PE.

**Q: Why would you use explicit null instead?**

A: PHP removes the top label and its TC bits before the last hop. With explicit null, label 0 reaches the egress, so the egress can still use the provider's QoS marking on that last link.

**Q: What is the difference between an RD and an RT?**

A: The RD is a 64-bit value that makes a customer route unique, so overlapping prefixes can live in BGP. The RT is an extended community that controls which VRFs import a route. RD is identity, and RT is membership. One RD does not stop or allow anything.

**Q: Why does an L3VPN packet carry two labels?**

A: The outer transport label, from LDP, gets the packet across the core to the right PE. The inner VPN label, from MP-BGP, tells that PE which customer or prefix it is for. P routers read only the outer label, so they need no customer routes.

**Q: Walk me through a packet crossing an MPLS L3VPN.**

A: The CE sends plain IP to the PE. The ingress PE looks in the VRF and finds the remote PE and VPN label. It pushes the VPN label, then the transport label. P routers swap the top label. The second-to-last router pops it. The egress PE reads the VPN label, removes it, and forwards plain IP to the correct CE.

**Q: VPN traffic fails but BGP is up and the PE loopbacks ping. Where do you look?**

A: Plain ping can pass while the LSP is broken. I check that every core link has MPLS on and every LDP neighbour is Oper. Then I check the forwarding table for 10.255.0.4/32 at each hop, looking for `No Label`. I use `ping mpls ipv4`. Then I compare RTs, and test a large packet in case of MTU trouble.

**Q: How does MPLS TE help, and what is replacing RSVP-TE?**

A: IGP sends everything on the shortest path, even when it is full. TE lets the head-end router place traffic on other paths using CSPF and RSVP-TE reservations, and adds fast reroute. Many new networks use Segment Routing, where the IGP carries the labels and P routers keep no per-tunnel state.

**Q: MPLS or SD-WAN for a company with 50 branches?**

A: It depends on the traffic. MPLS gives a contract SLA and steady delay, but costs more and is slower to deliver. SD-WAN gives encrypted tunnels over any link, per-application path choice, and direct cloud access. Often I would use SD-WAN with an internet link at each branch, and keep MPLS at sites with critical real-time traffic. They are not exclusive, since SD-WAN can run over MPLS.

## Quick Recap

- MPLS forwards by a 32-bit header: 20-bit label, 3-bit TC, 1-bit S, 8-bit TTL. Each router acts only on the top label.
- LERs push and pop at the edge. LSRs swap in the core. Labels are local to each router, and an LSP is one-way.
- LDP uses UDP 646 for discovery and TCP 646 for sessions. It follows the IGP.
- PHP pops the top label one hop early, signalled by implicit null (label 3). Explicit null (label 0) keeps the TC value to the last hop.
- L3VPN uses VRFs, MP-BGP, an RD for uniqueness, and an RT for import and export. The outer label finds the PE, and the inner label finds the customer.
- Traffic engineering places traffic on chosen paths using CSPF and RSVP-TE, and Segment Routing is the modern path.
- MPLS gives separation and an SLA, not encryption. SD-WAN gives flexibility over any transport, and the two often work together.
