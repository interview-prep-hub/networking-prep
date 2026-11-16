# SD-WAN

**Level:** Advanced | **Prerequisites:** [WAN Technologies](08-wan-technologies.md), [MPLS](../05-routing/13-mpls.md), [IPsec](../09-security/08-ipsec.md), [Software-Defined Networking (SDN)](09-software-defined-networking.md), [Quality of Service (QoS)](03-quality-of-service.md)

## In One Line

SD-WAN (Software-Defined Wide Area Network) builds one encrypted network on top of any mix of links, such as MPLS, Internet and LTE. A central system then controls it, and each application is sent down the path that suits it best right now.

## Why It Matters

A WAN (Wide Area Network) connects sites that are far apart, such as a head office and 200 branches. For about two decades, most companies built this WAN on MPLS, a private network service sold by a telecom provider. MPLS is reliable, but it became a poor fit for how companies now work.

The main changes were these:

- Applications moved from the company data centre to the cloud (Microsoft 365, Salesforce, AWS).
- Fast business Internet became cheap compared with MPLS bandwidth.
- Companies opened and closed sites faster than a provider could deliver a new circuit.

SD-WAN solves this. It lets a company use cheap links for most traffic, keep the good links for traffic that needs them, and manage every branch from one screen.

Interviewers ask about SD-WAN because it tests several skills at once: tunnels, routing, encryption, QoS, control plane design, and failure thinking. A good answer shows you understand that SD-WAN is an overlay, not a new kind of cable. It also shows you can explain what happens when a link gets bad but does not go down.

## Simple Explanation

Imagine a company that ships valuable goods between its offices.

- It has a private toll road (MPLS). It is smooth and safe, but very expensive.
- It has public roads (Internet). They are cheap and wide, but traffic jams happen.
- It has motorbike couriers (LTE or 5G). They go almost anywhere, but carry little.

In the old model, each office had its own rules, written by hand. "Always use the toll road. If the toll road is closed, use the public road." Nobody looked at traffic. If the toll road was open but full of potholes, the drivers still used it.

SD-WAN adds three things:

1. **Armoured vans.** Every load goes inside a locked van. The van is safe on any road, even a public one. This is the encrypted tunnel.
2. **Scouts.** Each office tests every road, every second, and measures delay and damage. This is path measurement.
3. **A head office dispatcher.** The dispatcher writes one rule book for all offices, for example "Medicine must take a road with less than 150 ms delay." Local drivers then choose the road that meets the rule. This is the controller and its policy.

Where the analogy stops being true: a van is one object on one road, but a network can copy a packet and send it down two paths at once. Also, the dispatcher does not touch the goods. In SD-WAN, user traffic does not go through the controller.

## How It Works

### Problems With a Traditional WAN

Here is a classic design. Every branch connects to a provider MPLS network. All Internet traffic goes through the head office, where the big firewall lives. Sending traffic to a central point and back like this is called **backhaul**.

```text
 Branch A ---+
             |    MPLS cloud          +--> Data centre apps
 Branch B ---+==== (provider) ==> Hub +
             |                        +--> Firewall --> Internet -> SaaS
 Branch C ---+
```

SaaS (Software as a Service) means an application you use over the Internet, such as Microsoft 365. Here are the problems with this design.

| Problem | What it looks like in practice |
|---|---|
| High cost per Mbps | The branch has 20 Mbps MPLS, while a business Internet link at the same site could offer far more for less money. Prices vary a lot by country. |
| Slow delivery | A new MPLS circuit often takes weeks or months. A pop-up store needs a link next week. |
| Cloud trombone | A user in a branch reaches Microsoft 365 by going to the hub first. Traffic goes out and back like a trombone slide. If the branch-to-hub path is 40 ms, every round trip gets at least 40 ms worse. |
| Idle backup links | A backup Internet link sits unused until the main link fails. You pay for two links and use one. |
| Blind routing | OSPF and BGP know if a neighbour is up or down. They do not know if a link has 5 percent packet loss. |
| Box-by-box work | Changing one rule on 500 branch routers means 500 configuration changes, with risk of typing errors on each. |
| Provider lock-in | One provider rarely covers every country you need, so global sites need several contracts. |

The "blind routing" problem is the most important one for interviews. A link has two kinds of failure:

- **Blackout:** the link is fully down. Routing protocols notice this.
- **Brownout:** the link is up, but quality is bad. Routing protocols see nothing wrong, and voice calls sound broken.

Engineers tried to fix this with hand-built tools such as IP SLA with policy-based routing, and later with Cisco DMVPN (Dynamic Multipoint VPN) and PfR (Performance Routing), packaged as Cisco IWAN. These worked, but they were complex and were configured router by router. [WAN Technologies](08-wan-technologies.md) covers the older options.

### Four Core Words

- **Transport:** a link you buy from someone, such as MPLS, broadband, fibre Internet, LTE, 5G or satellite.
- **Underlay:** the real network underneath, made of those transports. You do not control it.
- **Overlay:** a logical network built on top of the underlay using tunnels. You fully control it.
- **WAN edge:** the SD-WAN router at a site. It may be a physical box or a virtual machine. It connects the site LAN to the transports.

```text
 OVERLAY (what the company sees and controls)

    Branch-20 <===== one logical network, VPN 10 =====> Hub-100

 UNDERLAY (what really carries the packets)

    Branch-20 ------- MPLS provider network ------- Hub-100
        |                                              |
        +------------ Internet, any ISP ---------------+
        |                                              |
        +------------ LTE or 5G operator --------------+
```

### Overlay Across Any Transport

A **tunnel** puts a complete packet inside another packet. The inner packet has private addresses. The outer packet has addresses that the transport can route. The transport only sees the outer packet, so it does not need to know your private network.

SD-WAN encrypts the tunnel, normally with IPsec. IPsec is a set of protocols that encrypts and authenticates IP packets (see [IPsec](../09-security/08-ipsec.md)). Encryption is what makes the public Internet safe enough for company traffic.

Here is a typical packet layout. The exact fields vary by vendor. This shows ESP with AES-GCM and a small label that marks the customer network.

```text
+----------+-----+-----+----+-------+-------------+---------+-----+
| Outer IP | UDP | ESP | IV | Label | Original IP | Trailer | ICV |
| 20       | 8   | 8   | 8  | 4     | packet      | 2+pad   | 16  |
+----------+-----+-----+----+-------+-------------+---------+-----+
                                                     (sizes in bytes)
 |<--- not encrypted ---->|<----- encrypted ---------------->|
```

| Field | Purpose |
|---|---|
| Outer IP | Transport addresses, such as 172.16.20.2 to 172.16.100.2 on MPLS. |
| UDP | Lets the tunnel pass through NAT. NAT is the address translation most Internet routers do. |
| ESP | ESP (Encapsulating Security Payload) header: a security ID and a sequence number. |
| IV | IV (Initialisation Vector): a per-packet value used by the cipher. |
| Label | Says which customer VPN the packet belongs to. |
| Original IP packet | The user packet, such as 10.20.0.50 to 10.100.0.50. |
| Trailer | Padding and a field that says what comes next. |
| ICV | ICV (Integrity Check Value): proves nobody changed the packet. |

This totals about 66 bytes plus padding. That is why overlay packets must be smaller than the link MTU. See the MTU section in Going Deeper.

**Analogy:** a shipping container. A truck, a train and a ship can all carry the same sealed container. The carrier never opens it.

**Where it stops:** a container travels on one carrier at a time. A flow in SD-WAN can move to another transport mid-way through, and the edges at each end hide this from the application.

Each edge also runs a fast health check on every tunnel. The common tool is BFD (Bidirectional Forwarding Detection), a tiny hello protocol. On Cisco Catalyst SD-WAN, BFD runs inside each tunnel, with a default of 1 hello per second and a multiplier of 7. So a dead tunnel is declared down after about 7 seconds. Those are Cisco defaults. Other vendors use their own probes and timers.

**Segmentation** also lives in the overlay. A company can run separate networks over the same tunnels, for example VPN 10 for staff, VPN 20 for payment terminals and VPN 30 for guests. The label in the packet keeps them apart, the same way VRFs (Virtual Routing and Forwarding tables) keep routes apart inside a router. In SD-WAN, "VPN" here means a separated routing table, not a remote access VPN.

### Controllers: Splitting the Planes

Traditional routers do everything inside each box: they hold the config, learn routes and forward packets. SD-WAN moves the "thinking" to central systems, in the spirit of [SDN](09-software-defined-networking.md). Cisco Catalyst SD-WAN is the model most interviewers know, so this section uses its terms. Its older names are still very common.

| Role | Current Cisco name | Old name | Job |
|---|---|---|---|
| Management | SD-WAN Manager | vManage | GUI, API, templates, monitoring, software upgrades |
| Control | SD-WAN Controller | vSmart | Distributes routes, encryption keys and policy |
| Orchestration | SD-WAN Validator | vBond | First contact point. Authenticates new devices and helps with NAT |
| Data | WAN edge | vEdge, cEdge | Forwards user traffic |

```text
 +-----------------------+
 | Manager (vManage)     |  Management plane: GUI, API, config
 +-----------+-----------+
             | config push
 +-----------+-----------+     +------------------------+
 | Controller (vSmart)   |     | Validator (vBond)      |
 | Control plane: OMP    |     | Orchestration: joining |
 +---+---------------+---+     +-----------+------------+
     | routes, keys, |                     |
     | policy        |                     | first contact
 +---+----+      +---+----+                |
 | Edge   |<====>| Edge   |<---------------+
 | Site   | IPsec| Site   |
 | 20     | data | 100    |
 +--------+      +--------+
```

**Analogy:** an airport. The management office writes the schedule (Manager). The control tower tells pilots what to do (Controller). Security at the gate checks who you are (Validator). The planes carry the passengers (edges).

**Where it stops:** a plane cannot safely fly without the tower for long. A WAN edge can keep forwarding for hours on its last known routes and keys.

Key point: **user data does not pass through the controllers.** Packets go edge to edge. Controllers can sit in the cloud or a data centre, and a controller outage does not stop forwarding right away.

### OMP: How Sites Learn About Each Other

OMP (Overlay Management Protocol) is Cisco's control protocol for the overlay. It runs between each edge and the Controller, inside an encrypted DTLS or TLS connection. DTLS is TLS adapted to run over UDP. OMP is proprietary, not an IETF standard.

OMP works much like BGP with a route reflector. A route reflector is a router that passes routes between peers, so they do not all need to talk to each other. Edges do not form routing sessions with each other. They only talk to the Controller, and it reflects the information. See [BGP](../05-routing/09-bgp.md).

OMP carries three main kinds of information:

- **OMP routes:** LAN prefixes at each site, such as 10.20.0.0/24.
- **TLOC routes:** how to reach each tunnel endpoint.
- **Service routes:** where services such as a firewall sit.

A **TLOC** (Transport Locator) names one tunnel endpoint. It has three parts: system IP, color and encapsulation.

- **System IP:** a fixed ID for the router, written like an IPv4 address. It is not tied to any interface.
- **Color:** a label for the transport, such as `mpls`, `biz-internet` or `lte`.
- **Encapsulation:** usually `ipsec`.

So the next hop of a route is not an interface address. It is a TLOC, such as "system IP 10.255.0.100, color mpls, ipsec". This is why a route can stay valid even if the ISP changes the public IP address of the site.

Colors split into two groups:

- **Private colors**, such as `mpls`, `metro-ethernet` and `private1`. Two private colors use private addresses to build a tunnel.
- **Public colors**, such as `biz-internet`, `public-internet` and `lte`. The tunnel uses public addresses, and it assumes NAT may be in the path.

The `restrict` keyword tells an edge to build tunnels only to the same color. It stops an MPLS TLOC from trying to reach an Internet TLOC when no such path exists.

### Zero Touch Provisioning

ZTP (Zero Touch Provisioning) means a new router configures itself. Someone with no network skills plugs in power and a WAN cable, and the site comes up.

**Analogy:** a new employee who gets a company phone by mail. They turn it on, it contacts the company and it installs everything by itself.

**Where it stops:** a phone asks the user to log in. A router proves who it is with a built-in certificate, with no person involved.

Before shipping, the admin adds the serial numbers of the routers to the allowed list in the Manager. The admin also connects each device to a template, with site values such as system IP and site ID. Then the flow is as follows.

```text
 New edge         PnP cloud      Validator    Manager   Controller
    |                 |              |           |          |
 1  |-- gets IP by DHCP on WAN link  |           |          |
 2  |-- who am I? --->|              |           |          |
 3  |<-- go to here --|              |           |          |
 4  |-------- DTLS and certificates->|           |          |
 5  |<----- Manager, Controller IPs -|           |          |
 6  |------------- join ------------------------>|          |
 7  |<------------ config, software -------------|          |
 8  |------------- join, OMP ------------------------------>|
 9  |<=========== IPsec and BFD to other edges ============>|
```

1. The router boots with no config. It gets an address by DHCP on a WAN port, plus a DNS server and a default gateway.
2. It uses DNS to find the vendor's cloud redirect service. On Cisco, IOS XE routers use the Plug and Play service at `devicehelper.cisco.com`. Older vEdge routers use `ztp.viptela.com`.
3. The service finds the device by serial number and says which Validator belongs to its organisation.
4. The router opens a secure connection to the Validator. Both sides check certificates. The Validator also checks that the serial number is on the allowed list and that the organisation name matches.
5. The Validator gives the addresses of the Manager and the Controllers, then its job for this device ends.
6. The router connects to the Manager.
7. The Manager pushes config (over NETCONF) and a software upgrade if needed.
8. The router connects to the Controllers and starts OMP.
9. It learns routes, TLOCs and keys, then builds IPsec tunnels and starts BFD.

ZTP needs a few things to work:

- DHCP on the WAN, or a small bootstrap file on USB for sites with static IPs.
- Working DNS, and a clock close to correct, because certificates have valid dates.
- A firewall in the path that allows the control ports. On Cisco the base port is UDP 12346, with more ports in that range for port offset and port hopping. Check the Cisco firewall ports guide for your release.

**Why is this safe?** A thief who plugs in a random router gets nowhere, because its serial number is not on the list. An allowed router that gets stolen can be removed from the list, which cuts it off.

### Application-Aware Routing

Normal routing asks "Where is this destination?" Application-aware routing also asks "What is this traffic, and which path meets its needs right now?" An SLA (Service Level Agreement) is a promise about quality. In SD-WAN, an **SLA class** is a set of limits you define.

The process has four steps.

1. **Identify the application.** The edge looks at the traffic. It can use DSCP marks (QoS tags in the IP header), IP addresses and ports, or DPI (Deep Packet Inspection), which looks at traffic patterns and fields. With TLS, DPI can often still see the server name in the handshake. Encrypted Client Hello can hide this, so vendors also use DNS data and known address lists.
2. **Measure every path.** The edges measure loss, latency and jitter on every tunnel. **Latency** is delay. **Jitter** is variation in delay. **Loss** is the percentage of packets that never arrive.
3. **Compare against the SLA class.** A path is in SLA if all its numbers are within limits.
4. **Choose a path.** The edge picks a path that meets the SLA, and prefers the transport you named. If no path meets it, a fallback action applies, such as the best available path.

Here are example SLA classes. These are design choices, not standards. The 150 ms figure comes from the ITU-T G.114 guidance on one-way delay for voice.

| Class | Max loss | Max latency | Max jitter | Used for |
|---|---|---|---|---|
| Voice | 1 percent | 150 ms | 30 ms | Phone calls |
| Business | 3 percent | 300 ms | 100 ms | SAP, CRM |
| Bulk | Not set | Not set | Not set | Backup, updates |

**Analogy:** a sat-nav app. It knows the road network, and it also knows the traffic right now, so it can move you off a road that is open but slow.

**Where it stops:** a sat-nav reacts for each car. SD-WAN decisions are usually per flow, based on measurements from the recent past, so reaction is not instant.

A **flow** is one conversation between two endpoints. Most SD-WANs keep all packets of a flow on one path, to avoid out-of-order delivery. Sending packets of the same flow on two paths at once is possible on some products, but it is not the normal case.

### Direct Internet Access

DIA (Direct Internet Access), also called local breakout, lets a branch send SaaS and web traffic straight to its own Internet link, with no trip to the hub.

```text
 Before: Branch --> tunnel --> Hub firewall --> Internet --> SaaS
 After:  Branch --> NAT at branch edge ------> Internet --> SaaS
```

This removes the trombone, but it creates a new question: the hub firewall used to inspect that traffic, so who inspects it now? There are two usual answers. You can run a firewall function at the branch, or you can send the traffic to a cloud security service. The second answer leads to SASE, covered below.

### SD-WAN vs MPLS

First, a framing point that interviewers like. **MPLS is a transport. SD-WAN is an overlay.** They are not true alternatives, and MPLS can be one of the links under SD-WAN. The real question is "How much MPLS do we still need?"

| Topic | MPLS VPN | SD-WAN over Internet |
|---|---|---|
| What it is | Provider private network | Encrypted overlay on any transport |
| Quality promise | Contract SLA, end to end | No promise from the Internet itself. Good path choice reduces the pain |
| Encryption | Separated, not encrypted by default | Encrypted by default |
| Delivery time | Often weeks or months | Days, if a link already exists |
| Bandwidth cost | Usually high | Usually much lower |
| Who controls routing | Provider, partly | The customer |
| Cloud access | Often through a hub | Direct, or through a cloud on-ramp |
| QoS across the path | Provider honours class marks | Internet ignores marks. Only edge shaping helps |
| Visibility | Limited to link stats | Per-application, per-path data |
| Failover | Depends on backup design | Automatic across all links |

When does MPLS still make sense?

- Sites where only a poor Internet service is available.
- Strict voice or real-time needs across a long distance, where the Internet path is unstable.
- Rules that require a private path. Check the actual rule, because many only ask for encryption.

The common result is **hybrid WAN**: critical sites keep some MPLS, plus Internet, and small branches run on two Internet links, or Internet and LTE.

### The Link to SASE

SD-WAN answers "How do I connect sites well?" It does not fully answer "How do I secure users and data, when apps are in the cloud and users are at home?"

SASE (Secure Access Service Edge, said "sassy") is a term Gartner introduced in 2019. It joins network functions and security functions into one cloud-delivered service. In 2021 Gartner also named the security half SSE (Security Service Edge).

```text
 SASE = SD-WAN (network side) + SSE (security side)

 SSE usually includes:
   SWG    Secure Web Gateway: filters web traffic, blocks bad sites
   CASB   Cloud Access Security Broker: controls use of SaaS apps
   ZTNA   Zero Trust Network Access: per-app access for each user
   FWaaS  Firewall as a Service: a firewall run in the cloud
   DLP    Data Loss Prevention: stops sensitive data leaving
```

These functions run in a vendor's PoPs (Points of Presence), which are data centres placed close to users around the world.

```text
 Branch edge ==== IPsec tunnel ====+
                                   |    +--> SaaS
 Home user ---- agent, TLS or -----+-> SSE PoP --> Internet
                IPsec tunnel       |    +--> Private apps in cloud or DC
 Mobile user ----------------------+
             (policy follows the identity, not the location)
```

**Analogy:** a company used to check every visitor at one main gate. Now people work everywhere, so it opens a small checkpoint in every city. The rules are the same at every checkpoint.

**Where it stops:** a checkpoint looks at people once. An SSE service checks each session, and it uses identity and device health.

On an SD-WAN edge, a typical steering plan looks like this:

| Traffic | Path | Inspected by |
|---|---|---|
| Internal apps in DC | Overlay tunnel to hub | Hub firewall |
| Trusted SaaS, such as Microsoft 365 | DIA from the branch | Branch firewall rules, or SaaS tenant controls |
| General web | Tunnel to nearest SSE PoP | SWG, FWaaS |
| Guest Wi-Fi | DIA, separate VPN | Basic filter only |

SASE can be single-vendor, where one company supplies both parts, or multi-vendor, where you link an SD-WAN product to a separate SSE product by tunnels. [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md) explains the access model behind ZTNA.

## Example

### Scenario

A company has a hub at site 100 and a branch at site 20. Both have MPLS and business Internet. This example uses Cisco Catalyst SD-WAN. Device output has trimmed columns to fit the page.

```text
          Controllers (site 1): 192.0.2.10 to .12
                         |
                     Internet
                  /            \
   198.51.100.20                203.0.113.100
   biz-internet                 biz-internet
        |                            |
   +----+------+                +----+------+
   | Branch-20 |                | Hub-100   |
   | site 20   |                | site 100  |
   +----+--+---+                +---+---+---+
        |  |                        |   |
  LAN   |  | mpls              mpls |   | LAN
  10.20.0.0/24 172.16.20.2  172.16.100.2  10.100.0.0/24
                  \                /
                   MPLS provider
```

| Item | Branch-20 | Hub-100 |
|---|---|---|
| Site ID | 20 | 100 |
| System IP | 10.255.0.20 | 10.255.0.100 |
| MPLS interface | 172.16.20.2/30, gateway 172.16.20.1 | 172.16.100.2/30, gateway 172.16.100.1 |
| Internet interface | 198.51.100.20/28, gateway 198.51.100.17 | 203.0.113.100/28, gateway 203.0.113.97 |
| LAN (VPN 10) | 10.20.0.0/24, edge is 10.20.0.1 | 10.100.0.0/24, edge is 10.100.0.1 |

| Controller role | System IP | Address |
|---|---|---|
| Manager | 10.255.0.1 | 192.0.2.12 |
| Controller | 10.255.0.2 | 192.0.2.11 |
| Validator | 10.255.0.3 | 192.0.2.10 |

### Branch Config

In real life, the Manager pushes this from a template. It is shown here so you can see what ends up on the router. This is simplified Cisco IOS XE SD-WAN config.

```text
system
 system-ip 10.255.0.20
 site-id 20
 organization-name "Example Corp"
 vbond 192.0.2.10
!
! Transport side (VPN 0)
interface GigabitEthernet1
 description MPLS
 ip address 172.16.20.2 255.255.255.252
!
interface GigabitEthernet2
 description INTERNET
 ip address 198.51.100.20 255.255.255.240
!
ip route 0.0.0.0 0.0.0.0 172.16.20.1
ip route 0.0.0.0 0.0.0.0 198.51.100.17
!
interface Tunnel1
 ip unnumbered GigabitEthernet1
 tunnel source GigabitEthernet1
 tunnel mode sdwan
!
interface Tunnel2
 ip unnumbered GigabitEthernet2
 tunnel source GigabitEthernet2
 tunnel mode sdwan
!
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color mpls restrict
 !
 interface GigabitEthernet2
  tunnel-interface
   encapsulation ipsec
   color biz-internet
 !
 omp
  address-family ipv4 vrf 10
   advertise connected
!
! Service side (VPN 10)
vrf definition 10
 address-family ipv4
!
interface GigabitEthernet3
 description LAN-VPN10
 vrf forwarding 10
 ip address 10.20.0.1 255.255.255.0
```

### Check the Control Plane

The branch should have connections to the Manager and the Controller. It normally does not stay connected to the Validator.

```text
Branch-20# show sdwan control connections
PEER     PROT  SYSTEM IP   SITE  PUBLIC IP   PORT   LOCAL COLOR   STATE
vsmart   dtls  10.255.0.2  1     192.0.2.11  12346  mpls          up
vsmart   dtls  10.255.0.2  1     192.0.2.11  12346  biz-internet  up
vmanage  dtls  10.255.0.1  1     192.0.2.12  12346  biz-internet  up

Branch-20# show sdwan omp peers
PEER        TYPE    DOMAIN  OVERLAY  SITE  STATE  R/I/S
10.255.0.2  vsmart  1       1        1     up     2/2/2
```

R/I/S means routes received, installed and sent. The branch received the hub LAN twice, once for each hub TLOC. It sent its own LAN twice.

### Check the Tunnels

```text
Branch-20# show sdwan bfd sessions
SYSTEM IP     SITE  STATE  SRC COLOR     DST COLOR     DST IP
10.255.0.100  100   up     mpls          mpls          172.16.100.2
10.255.0.100  100   up     biz-internet  biz-internet  203.0.113.100
```

There are two tunnels to the hub, one per transport. `restrict` on `mpls` stops any cross-color tunnel.

### Check the Routes

```text
Branch-20# show sdwan omp routes vpn 10
VPN  PREFIX         FROM PEER   LABEL  STATUS TLOC IP       COLOR
10   10.100.0.0/24  10.255.0.2  1003   C,I,R  10.255.0.100  mpls
10   10.100.0.0/24  10.255.0.2  1003   C,I,R  10.255.0.100  biz-internet

Branch-20# show ip route vrf 10 omp
m    10.100.0.0/24 [251/0] via 10.255.0.100, 01:12:44, Sdwan-system-intf
```

The status letters mean Chosen, Installed and Resolved. Both paths are usable, and OMP is the source of the route. Its administrative distance on IOS XE is 251 by default. Notice that the next hop is the hub's system IP, not a transport address.

### Build the Policy

The admin writes this policy on the Manager, and the Controller pushes it to the edges. This is simplified Controller-style syntax.

```text
policy
 sla-class VOICE
  loss 1
  latency 150
  jitter 30
 !
 app-route-policy AAR-BRANCHES
  vpn-list VPN10
   sequence 10
    match
     dscp 46
    !
    action
     sla-class VOICE preferred-color mpls
    !
   !
  !
 !
 lists
  vpn-list VPN10
   vpn 10
  !
  site-list BRANCHES
   site-id 20-29
  !
 !
!
apply-policy
 site-list BRANCHES
  app-route-policy AAR-BRANCHES
```

In plain words: for traffic in VPN 10 with DSCP 46 (the usual voice mark), use a path that meets the VOICE class, and prefer MPLS if it does.

The edges use faster probing than the Cisco default in this example. This is the IOS XE form:

```text
bfd app-route poll-interval 120000
bfd app-route multiplier 3
```

So each bucket lasts 120 seconds, and the result is the average of the last 3 buckets.

### A Brownout, Minute by Minute

The MPLS provider has a fault at 09:00. The link stays up, but loss jumps to 3 percent. Internet stays clean at 0.1 percent loss, 35 ms latency and 6 ms jitter.

```text
 Time    Buckets (MPLS loss %)    Average   SLA 1%     Voice path
 08:58   0, 0, 0                  0.0       met        mpls
 09:02   0, 0, 3                  1.0       met        mpls
 09:04   0, 3, 3                  2.0       violated   biz-internet
 09:06   3, 3, 3                  3.0       violated   biz-internet
 (provider fixes the fault at 09:06)
 09:08   3, 3, 0                  2.0       violated   biz-internet
 09:10   3, 0, 0                  1.0       met        mpls
```

```text
Branch-20# show sdwan app-route stats
app-route statistics 172.16.20.2 172.16.100.2 ipsec 12346 12346
 remote-system-ip 10.255.0.100
 local-color      mpls
 remote-color     mpls
 mean-loss        2
 mean-latency     30
 mean-jitter      9
```

At no time did the MPLS interface go down. OSPF or BGP alone would have left calls on the bad path. Note what the timeline also shows: callers had 3 percent loss for about 4 minutes before the move. Going Deeper covers why, and what reduces this.

### The Same Path From a Host

From a host, the whole overlay looks like one hop. A user on 10.20.0.50 tests the hub host 10.100.0.50.

Linux:

```bash
$ traceroute -n 10.100.0.50
traceroute to 10.100.0.50, 30 hops max, 60 byte packets
 1  10.20.0.1    0.612 ms  0.580 ms  0.561 ms
 2  10.100.0.1   29.114 ms 29.087 ms 29.062 ms
 3  10.100.0.50  29.804 ms 29.771 ms 29.749 ms
```

Windows:

```powershell
PS> tracert -d 10.100.0.50

Tracing route to 10.100.0.50 over a maximum of 30 hops

  1    <1 ms    <1 ms    <1 ms  10.20.0.1
  2    29 ms    29 ms    29 ms  10.100.0.1
  3    30 ms    29 ms    30 ms  10.100.0.50

Trace complete.
```

The many provider routers in the MPLS network do not appear, because the host's packet is sealed inside the tunnel. That is the overlay in action.

### Local Breakout Policy

To send Microsoft 365 straight out of the branch, a data policy can use NAT toward VPN 0. This is simplified syntax. `SAAS-APPS` is an application list defined elsewhere.

```text
data-policy DIA-SAAS
 vpn-list VPN10
  sequence 10
   match
    app-list SAAS-APPS
   !
   action accept
    nat use-vpn 0
   !
  !
 !
```

The WAN interface also needs NAT enabled, and the branch is now reaching the Internet directly, so security must be planned for it.

### Tunnel Count Math

Take 100 sites, each with two transports, and same-color tunnels only.

```text
 Full mesh, site pairs:   100 x 99 / 2 = 4950
 Tunnels:                 4950 x 2     = 9900
 Tunnels per router:      99 x 2       = 198

 Hub and spoke (1 hub):   99 x 2       = 198 tunnels in total
 Tunnels per spoke:       2
 Tunnels on the hub:      198
```

A small branch router may not handle 198 tunnels plus BFD on each one. This is why large networks control which sites build tunnels to each other.

## Going Deeper

### Standards and Vendors

There is no single SD-WAN protocol standard. Products from different vendors do not form one overlay with each other. Each vendor has its own control plane. MEF (a telecom industry forum) publishes service definitions in its MEF 70 series, which describe SD-WAN as a service, not how devices must talk.

Terms differ by vendor, so learn the role and not only the name.

| Vendor product | Management | Control approach | Note |
|---|---|---|---|
| Cisco Catalyst SD-WAN | Manager | Controllers run OMP | Came from the Viptela purchase in 2017 |
| Cisco Meraki | Dashboard (cloud) | Vendor cloud | Simple, managed from one cloud dashboard |
| VeloCloud (now Arista) | Orchestrator | Controllers and Gateways | Bought by VMware, then Broadcom, then Arista |
| Fortinet Secure SD-WAN | FortiManager | Often IPsec, BGP and ADVPN | Firewall-first design |
| Palo Alto Prisma SD-WAN | Cloud controller | Vendor cloud | From the CloudGenix purchase |
| HPE Aruba EdgeConnect | Orchestrator | Vendor system | From the Silver Peak purchase |
| Juniper Session Smart | Conductor | Session-based | Uses session metadata in place of classic tunnels |

The Fortinet and Juniper rows show a fuller truth: "SD-WAN means a central controller plus IPsec full mesh" is a simplification. The shared idea is overlay plus central policy plus path choice.

### Topology Choices

- **Full mesh:** every site tunnels to every site. Best latency for site-to-site traffic, worst scale.
- **Hub and spoke:** spokes tunnel only to hubs. Simple and scalable, but spoke-to-spoke traffic takes a detour.
- **Partial or regional mesh:** sites mesh inside a region, and regions connect through hubs.
- **On-demand tunnels:** spokes build a direct tunnel only when traffic starts, then remove it when idle. Cisco has on-demand tunnels, and Fortinet has ADVPN.

Topology is set by policy, not by wiring. On Cisco, a centralized control policy on the Controller can filter which TLOCs a site learns. If a spoke never learns another spoke's TLOC, it cannot build a tunnel to it.

### What Fails When Controllers Fail

This is a favourite interview topic. The table uses Cisco behaviour.

| What failed | Effect on current traffic | What stops working |
|---|---|---|
| Manager | None | Config changes, monitoring, GUI |
| Validator | None for joined edges | New edges cannot join. Edges that lost control sessions struggle to return |
| All Controllers | Forwarding goes on using stored routes and keys | No new routes, no policy changes |
| One transport | Flows move to other transports, about 7 s for a dead tunnel at default BFD | Capacity drops |
| Whole edge | Site down, unless it has a second edge | Everything at that site |

OMP graceful restart is what keeps forwarding alive when Controllers are gone. The Cisco default timer is 43200 seconds, which is 12 hours. During this time the network is frozen, not dead. If a link fails, edges still use BFD and can pick another stored path, but the overlay cannot learn any new information.

Design for this: run at least two Controllers, in different locations. Edges connect to several of them.

### How Encryption Keys Work

In a classic IPsec design, each pair of peers runs IKE. IKE is the protocol that negotiates keys. With 9900 tunnels, that is a lot of work.

Cisco SD-WAN avoids this by default. Each edge makes keys for its TLOCs and sends them to the Controller over the secure OMP session. The Controller reflects them to other edges. This scales well because one central trust point replaces thousands of peer negotiations. The default rekey time is 86400 seconds (1 day). Cisco also supports pairwise keys, and IKE for tunnels to devices outside the fabric, such as cloud security services.

### NAT Traversal

Many edges sit behind NAT, and LTE links often sit behind carrier NAT. A tunnel needs the real public address and port of the far end.

The Cisco Validator acts like a STUN server. STUN is a method that tells a device how the outside world sees it. The edge contacts the Validator. The Validator sees the source address and port after NAT and reports them back. The edge then advertises both private and public addresses in its TLOC.

Two edges that are both behind a symmetric NAT usually cannot build a direct tunnel, because each new destination gets a new mapped port. Plan a hub with a public address for such sites. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

### How Fast Can It React?

There are two very different speeds:

- **Blackout on Cisco:** BFD hello 1 s, multiplier 7, about 7 seconds.
- **Brownout on Cisco default:** poll interval of 10 minutes (600000 ms), and app-route multiplier of 6. That is an average over a 60-minute window.

Suppose a link goes from 0 to a steady 3 percent loss, with a 1 percent SLA. This simple model shows when the average crosses the limit.

```text
 Default: 6 buckets of 10 minutes
   After bucket 1:  3 / 6 = 0.5 percent   in SLA
   After bucket 2:  6 / 6 = 1.0 percent   in SLA
   After bucket 3:  9 / 6 = 1.5 percent   out of SLA (up to 30 min)

 Tuned: 3 buckets of 2 minutes
   After bucket 1:  3 / 3 = 1.0 percent   in SLA
   After bucket 2:  6 / 3 = 2.0 percent   out of SLA (about 4 min)
```

Do not tune too hard, though. With 1 hello per second, a 2-minute bucket holds 120 probes. One lost probe is already 0.83 percent. A very short window makes the result jumpy, and traffic moves back and forth. This is called **flapping**. Newer Cisco releases add Enhanced Application-Aware Routing, which measures with real data traffic and aims to react in seconds. Check what your release supports.

### Fixing Bad Paths Instead of Avoiding Them

Sometimes every path is imperfect, for example a site with two poor Internet links. Then edges can repair the traffic. These are vendor features, and availability varies.

- **FEC (Forward Error Correction):** send extra recovery data. If you send one parity packet for every 4 data packets, the receiver can rebuild any 1 lost packet in that group. The cost is 25 percent more bandwidth.
- **Packet duplication:** send every packet over two transports, and the receiver drops the second copy. It gives great protection and costs double the bandwidth, so use it only for small, critical flows such as voice.
- **Packet reordering:** the receiving edge puts packets back in order before delivering them.

### Moving Flows Is Not Always Free

Inside the overlay, moving a flow is easy because the inner addresses do not change. The application sees a short delay or small loss.

DIA is different. If a flow moves from one Internet link to another, the NAT source address changes. The server sees a new address, so TCP sessions break. Good designs keep existing DIA flows on their path, and only send new flows to the new link.

Path choice can also cause **asymmetric routing**, where traffic goes out on one path and returns on another. Stateful firewalls that see only one direction will drop the traffic. See [Firewalls](../09-security/04-firewalls.md).

### MTU and Fragmentation

Tunnels take space. On a 1500-byte link, the room for an inner packet is usually in the range of about 1400-1450 bytes, depending on vendor and cipher. Read the real number from the device. On Cisco, `show sdwan tunnel statistics` includes the tunnel MTU.

Common signs of an MTU problem: ping works, small pages work, and large file transfers hang. The usual fix is to clamp TCP MSS (the largest TCP data size) on the LAN side of the edge:

```text
interface GigabitEthernet3
 ip tcp adjust-mss 1350
```

To test with a tunnel MTU of 1440, the largest ICMP payload that fits is 1440 - 28 = 1412 bytes.

Linux:

```bash
ping -M do -s 1412 -c 3 10.100.0.50
```

Windows:

```powershell
ping -f -l 1412 -n 3 10.100.0.50
```

See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### QoS on an Overlay

Edges usually copy the inner DSCP mark to the outer IP header. MPLS providers may honour it, if your contract includes classes. Internet providers normally ignore it.

So real control sits at the edge. The edge should:

1. Shape traffic to the true link speed, so queues build in the router, where you control them, and not at the provider.
2. Queue voice ahead of bulk traffic.
3. On hubs, use per-tunnel shaping, so a 1 Gbps hub does not flood a 50 Mbps branch.

Remember that Internet links are often asymmetric, such as 200 Mbps down and 20 Mbps up. Details are in [Quality of Service (QoS)](03-quality-of-service.md).

### Site Redundancy

Many sites use two edges. Each has one transport, but ideally each should reach both.

**TLOC extension** does this. Edge A gives Edge B a path to its transport through a local link between them, so both edges can use both transports, even if each is wired to just one.

```text
   MPLS            Internet
     |                 |
 +---+----+  TLOC  +---+----+
 | Edge A |--ext---| Edge B |
 +---+----+        +---+----+
     |                 |
     +---- LAN core ---+
       (VRRP or OSPF or BGP)
```

The LAN side normally uses VRRP, OSPF or BGP, as covered in [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md).

LTE is often set as a last-resort link. Cisco has a `last-resort-circuit` tunnel option, so the LTE path carries data only when the main links have failed. This keeps usage low on metered plans.

### Migrating From MPLS

Nobody converts 500 sites in a night. A common plan is this:

1. Deploy controllers and put edges at the hubs first.
2. Make each hub a bridge. It learns SD-WAN routes through OMP and legacy MPLS routes through BGP, and passes them between both sides.
3. Convert branches a few at a time. New sites reach old sites through the hub.
4. Add Internet links and start moving non-critical traffic.
5. Reduce MPLS bandwidth, or drop it at sites where data proves it is not needed.

Watch for routing loops at the hubs, where routes get redistributed both ways. Use tags or communities to mark where a route came from.

### SD-WAN and Cloud

Vendors offer virtual edges that run inside AWS, Azure and Google Cloud. A virtual edge joins the overlay like any other site. Vendors also automate the links to native cloud services, such as AWS Transit Gateway or Azure Virtual WAN. See [Hybrid Cloud Connectivity](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).

### Troubleshooting Order

Work upward from the underlay.

1. **Underlay:** can the edge ping its gateway and the far transport address?
2. **Time and identity:** is the clock right? Is the certificate valid?
3. **Control:** `show sdwan control connections`. If not up, check `show sdwan control connections-history`.
4. **Routes and TLOCs:** `show sdwan omp routes`, `show sdwan omp tlocs`.
5. **Data tunnels:** `show sdwan bfd sessions`.
6. **Policy:** `show sdwan policy from-vsmart`.
7. **Quality:** `show sdwan app-route stats`.

The history command gives reason codes. Three useful ones on Cisco:

| Code | Meaning | Common cause |
|---|---|---|
| `CRTVERFL` | Failed to verify the peer certificate | Wrong clock, wrong root certificate |
| `CTORGNMMIS` | Organisation name mismatch | Typing error in the organisation name |
| `DCONFAIL` | DTLS connection failure | Firewall blocks UDP, no route to peer |

## Common Mistakes

- **"SD-WAN replaces the Internet or MPLS."** It does not. It needs transports underneath. It adds no bandwidth and cannot fix a link with no good path.
- **"SD-WAN makes the Internet as good as MPLS."** It only picks the best of what exists. If all paths are bad at once, users suffer.
- **"Traffic flows through the controller."** It does not. Controllers handle control and management. Data goes edge to edge.
- **"If the controllers die, the WAN dies."** Forwarding keeps working for hours on stored state, but changes stop.
- **"DSCP marks give me priority on the Internet."** ISPs normally ignore them. Your only real tool there is shaping at the edge.
- **Forgetting probe timers.** Teams assume path changes are instant, then find that default settings average over a long window.
- **Tuning timers too low.** Short windows cause flapping, and traffic bounces between paths.
- **Ignoring MTU.** Pings pass but big transfers hang. Clamp MSS.
- **Two Internet links from the same ISP on the same cable.** They look redundant but share one failure point. Check physical path diversity.
- **Treating DIA as only a routing change.** Local breakout removes the central firewall from the path. You must add security at the branch or in the cloud.
- **Full mesh everywhere.** Tunnel counts grow fast, and most branches rarely talk to each other.
- **Confusing the VPN number with a remote access VPN.** In SD-WAN, VPN 10 is a separated routing table, like a VRF.
- **Confusing SD-WAN with SDN.** SDN is the general idea of separating control from forwarding. SD-WAN is one application of it, on the WAN.
- **Thinking SASE is a product you add to a router.** SASE is an architecture. Its security side runs in cloud PoPs.

## Interview Questions

**Q: What is SD-WAN, and what problem does it solve?**

A: SD-WAN is an encrypted overlay that runs on any transport, such as MPLS, Internet or LTE, with central management and policy. It solves the high cost and slow delivery of MPLS, the backhaul of cloud traffic through a hub, and router-by-router config. It also makes routing aware of link quality, not only of up or down.

**Q: What is the difference between an underlay and an overlay?**

A: The underlay is the real network of links and provider routers. The overlay is a logical network of tunnels on top. In SD-WAN you own the overlay and buy the underlay. The provider routers see only outer tunnel addresses, and they know nothing about my private addresses.

**Q: Why can routing protocols not solve the brownout problem?**

A: OSPF and BGP track whether a neighbour or route is reachable. A link with 5 percent loss still passes hellos, so the route stays. SD-WAN measures loss, latency and jitter on every tunnel, compares them with an SLA class, and moves the affected application to a path that meets it.

**Q: Walk me through zero touch provisioning.**

A: The serial number is pre-loaded on the allowed list, and a template is attached. On site, the router gets an address by DHCP, uses DNS to reach the vendor redirect service, and learns its Validator. It authenticates with certificates. The Validator gives it the Manager and Controller addresses. The Manager pushes config, then the router joins the Controller by OMP, learns routes and keys, and builds IPsec tunnels.

**Q: What are the main components in Cisco SD-WAN?**

A: The Manager, or vManage, does management and monitoring. The Controller, or vSmart, runs OMP and distributes routes, keys and policy. The Validator, or vBond, authenticates devices when they join and helps with NAT. WAN edges forward the data. User traffic never goes through the first three.

**Q: What is a TLOC?**

A: A TLOC is a transport locator, the endpoint of a tunnel. It is made of the system IP, color and encapsulation. OMP routes use a TLOC as the next hop. So one site prefix can have several next hops, one for each transport, and the edge can pick between them.

**Q: What happens if all controllers go down?**

A: Forwarding continues. Edges keep their last known routes, TLOCs and keys through OMP graceful restart, which is 12 hours by default on Cisco. BFD still works, so an edge can fail over to a stored path. But no new routes or policy can come in, and new edges cannot join. If only the Manager fails, the network is unaffected, but you lose config and monitoring.

**Q: Is SD-WAN a replacement for MPLS?**

A: Not exactly. MPLS is a transport, and SD-WAN is an overlay that can run on it. Many companies do reduce MPLS, since Internet is cheaper and faster to deliver. But MPLS gives a provider-backed SLA and class-based QoS, so critical sites often keep it. The usual result is a hybrid WAN.

**Q: How does SD-WAN differ from DMVPN?**

A: Both build encrypted overlays over any transport. DMVPN uses mGRE, NHRP and IPsec, and each router is configured on its own, with a normal routing protocol. SD-WAN adds central control and policy, zero touch provisioning, built-in per-application path selection and rich visibility. DMVPN needs extra tools, such as PfR, for quality-based routing.

**Q: A voice call sounds bad, but the SD-WAN shows every tunnel up. What do you check?**

A: Up only means BFD is passing. I check app-route statistics for loss, latency and jitter per tunnel. Then I check that voice is really being classified, for example that it arrives marked DSCP 46, and that the policy is applied to this site. I look at poll interval and multiplier, because defaults can be slow. Then I check shaping, because a full upload queue causes jitter. If all paths are poor, I consider FEC or packet duplication.

**Q: What is the difference between SD-WAN and SASE?**

A: SD-WAN is about connecting sites and picking paths. SASE combines SD-WAN with cloud-delivered security, known as SSE: secure web gateway, CASB, zero trust network access and firewall as a service. SD-WAN is the on-ramp, and SASE applies the same policy for branch users, home users and mobile users based on identity.

**Q: Why might a flow break when it moves to another Internet link, when overlay flows do not?**

A: Overlay flows keep the same inner addresses, so moving tunnels is hidden from both ends. With direct Internet access, the source is translated to the address of the link. A new link means a new source address, so the server sees a different client, and TCP breaks. So designs keep existing NAT flows on their link and move only new ones.

## Quick Recap

- SD-WAN is an encrypted overlay on any transport, with central management and policy. It is not a new link type.
- It solves MPLS cost and delivery time, cloud backhaul, idle backup links, box-by-box config, and blindness to link quality.
- The Cisco roles are Manager (management), Controller (routes, keys, policy), Validator (joining) and edges (data). User data goes edge to edge.
- Application-aware routing identifies an app, measures loss, latency and jitter, then picks a path that meets the SLA class. Timer settings decide how fast it reacts.
- ZTP lets a pre-listed router get an address, find its Validator, prove its identity with certificates, then download config by itself.
- MPLS is a transport and can sit under SD-WAN. Hybrid WAN is the common result.
- SASE combines SD-WAN with cloud security (SSE), because direct Internet access moves traffic away from the central firewall.
