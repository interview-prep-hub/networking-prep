# WAN Technologies

**Level:** Intermediate | **Prerequisites:** [Types of Networks](../01-foundations/02-network-types.md), [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [How Routers Work](../05-routing/01-how-routers-work.md)

## In One Line

A WAN (Wide Area Network) joins sites that are far apart. WAN technologies are the different ways to rent, build and run those long links, from private circuits to tunnels over the public internet.

## Why It Matters

A company has an office in one city and a data centre in another. It cannot dig its own trench between them. It must buy a service from a carrier (a company that owns long-distance network cables, also called a provider or telco).

There are many services to buy. They differ in five things: cost, speed, delay, privacy and reliability. No single option wins on all five. A bank branch, a shop, a home worker and a data centre all need different answers.

Interviewers ask about WANs because the questions test real judgement:

- Can you explain why a leased line costs ten times more than broadband for the same speed?
- Do you know that MPLS is private but not encrypted?
- Can you explain why a PPPoE link has an MTU of 1492?
- Can you design 100 branches with DMVPN and say why you did not use a full mesh?

## Simple Explanation

Imagine your company must move parcels between its offices.

| WAN option | Parcel analogy |
|---|---|
| Leased line | A private road between two offices. Only your trucks use it. |
| MPLS | A private courier company. It puts a barcode on each parcel and moves it on its own network. |
| Metro Ethernet | A very long office corridor. Both offices feel like the same building. |
| DSL, cable, PON | The public road near your home. Cheap, but everyone shares it. |
| 4G or 5G | A motorbike courier. It can reach almost anywhere, but traffic is unpredictable. |
| GRE tunnel | Putting a letter inside a second envelope with a new address. The envelope has no lock. |
| IPsec | A locked steel box around that envelope. |
| DMVPN | A head office phone book. Branches ask it for each other's address, then talk directly. |

Where the analogy stops: real links carry many customers at once by using labels, time slots or wavelengths. Even a "private road" often shares the same fibre as other customers. You get a guaranteed share, not a physically separate cable.

## How It Works

### WAN Building Blocks

These words appear everywhere in WAN work.

```text
   Site A                                                 Site B
 +-----+    +--------+ local +---------+ local +--------+    +-----+
 | LAN |----| Router |-loop--| Carrier |-loop--| Router |----| LAN |
 +-----+    | (CPE)  |   |   | network |   |   | (CPE)  |    +-----+
            +--------+   |   +---------+   |   +--------+
                         |                 |
                    demarcation       demarcation
                       point             point
```

- **CPE (Customer Premises Equipment):** gear at your site, such as your router or modem.
- **Local loop (last mile):** the cable between your building and the carrier's nearest site. It is often the slowest and most expensive part.
- **Demarcation point (demarc):** the line where the carrier's duty ends and yours begins. If the fault is on your side of it, you fix it.
- **SLA (Service Level Agreement):** a contract with numbers, such as 99.99% uptime and a maximum delay.
- **CIR (Committed Information Rate):** the speed the carrier promises. The port can be faster than the CIR.
- **Contention:** sharing. A contended link is shared with neighbours, so speed can drop at busy times.
- **Underlay:** the real network that carries packets, such as the internet.
- **Overlay:** a virtual network built on top of the underlay, such as a GRE tunnel.

A WAN is either a **private WAN** (carrier service with an SLA, such as MPLS) or an **internet WAN** (tunnels over public internet links). Many companies use both.

### Circuit Switching and Packet Switching

**Circuit switching** reserves a fixed path and fixed bandwidth for the whole call. The old telephone network worked this way. The capacity is yours even when you send nothing.

**Packet switching** cuts data into packets and shares the links between all users. Capacity is used better, but delay can vary. MPLS, Metro Ethernet and the internet are all packet switched.

### Leased Lines

A leased line is a private, fixed-speed link that you rent. It has the same speed in both directions, and the speed does not depend on what your neighbours do.

The classic leased lines were TDM circuits. TDM (Time Division Multiplexing) gives each user a repeating time slot on a shared line.

| Name | Region | Speed | How it is built |
|---|---|---|---|
| T1 | North America | 1.544 Mbps | 24 x 64 kbps channels + 8 kbps framing |
| E1 | Europe, much of the world | 2.048 Mbps | 32 x 64 kbps time slots |
| T3 | North America | 44.736 Mbps | 28 T1s plus overhead |
| E3 | Europe, much of the world | 34.368 Mbps | 16 E1s plus overhead |
| OC-3 / STM-1 | SONET / SDH | 155.52 Mbps | Optical carrier |
| OC-12 / STM-4 | SONET / SDH | 622.08 Mbps | Optical carrier |
| OC-48 / STM-16 | SONET / SDH | 2,488.32 Mbps | Optical carrier |
| OC-192 / STM-64 | SONET / SDH | 9,953.28 Mbps | Optical carrier |

SONET and SDH are older standards for carrying data over fibre. SONET is used in North America, and SDH in most other regions.

On a Cisco router, an old serial leased line needs a Layer 2 protocol. **HDLC** (High-Level Data Link Control) is the Cisco default on serial interfaces. **PPP** is the open choice, covered below. The **CSU/DSU** (Channel Service Unit / Data Service Unit) is the box, often built into the interface card, that connects the router to a digital carrier line.

Today, the phrase "leased line" usually means something else: a dedicated Ethernet service over fibre, at 100 Mbps, 1 Gbps or 10 Gbps. Two forms are common:

- **Point to point:** a private link between two of your sites.
- **DIA (Dedicated Internet Access):** an uncontended link from your site to the ISP, with an SLA.

For very high speed between data centres, you can also buy a **wavelength** (one colour of light on a carrier's fibre, often 10, 100 or 400 Gbps) or **dark fibre** (fibre with no equipment on it, so you light it yourself).

### MPLS

MPLS (Multiprotocol Label Switching) is a carrier technology. Routers forward packets using short labels instead of doing a full IP lookup at every hop. The full detail lives in the [MPLS chapter](../05-routing/13-mpls.md). Here we care about what you buy.

The most common product is an **MPLS L3VPN**: a private routed network. Your sites connect to the carrier, and the carrier keeps your routes separate from other customers. Its standard is RFC 4364.

```text
 Your site   Carrier edge     Carrier core   Carrier edge    Your site
  +----+        +-----+         +-----+        +-----+        +----+
  | CE |--------| PE1 |---------|  P  |--------| PE2 |--------| CE |
  +----+        +-----+         +-----+        +-----+        +----+

        plain IP     [T1][V5] IP    [T2][V5] IP     plain IP
```

- **CE (Customer Edge):** your router. It knows nothing about labels.
- **PE (Provider Edge):** the carrier router that faces you. It keeps a separate routing table for each customer, called a **VRF** (Virtual Routing and Forwarding).
- **P (Provider):** a core router. It swaps labels and knows nothing about your networks.

In the diagram, T1 and T2 are transport labels. They move the packet across the carrier. V5 is the VPN label. It tells PE2 which customer the packet belongs to.

To your CE, the whole carrier network looks like one big router. Normally, you share routes with the PE using BGP or static routes.

Why companies pay for MPLS:

- **Any-to-any routing:** the carrier provides the mesh. You buy one connection per site.
- **QoS (Quality of Service):** the carrier can honour traffic classes across its network. See [QoS](03-quality-of-service.md).
- **SLA:** written limits for delay, loss and uptime.

The big limit is cost per Mbps. Also, MPLS gives **separation, not encryption**. If your data must be secret from the carrier, add IPsec on top.

The other main product is an **L2VPN**, where the carrier carries Ethernet frames for you. VPLS (Virtual Private LAN Service) gives a multipoint service, and pseudowires give a point-to-point service.

### Metro Ethernet

Metro Ethernet is a carrier service that gives you an Ethernet port at each site. Plug in a router or a switch, and the far site feels like it is on the same long cable. It began inside one city (a metro area), but carriers now offer it across countries.

The MEF, an industry group, defined the service names that interviewers expect:

| Service | Shape | Plain meaning |
|---|---|---|
| E-Line | Point to point | A virtual cable between two sites |
| E-LAN | Multipoint to multipoint | A virtual switch joining many sites |
| E-Tree | Rooted multipoint | Leaves talk to the root, not to each other |

Each can be **private** (one customer on the port, such as EPL for E-Line) or **virtual** (several services on one port, split by VLAN, such as EVPL).

Carriers often use **Q-in-Q** (IEEE 802.1ad) to carry customer VLANs. They add an outer VLAN tag around your own tag. See [Trunking and 802.1Q](../03-ethernet-and-switching/07-trunking-8021q.md).

Port speed and purchased speed can differ. A carrier may give you a 1 Gbps port with a CIR of 200 Mbps, then drop everything above 200 Mbps. If your router sends bursts at 1 Gbps, you see packet loss on a link that looks empty. The fix is to shape on your side.

```text
! Cisco IOS: shape to the 200 Mbps CIR before the carrier drops it
policy-map SHAPE-200M
 class class-default
  shape average 200000000
!
interface GigabitEthernet0/1
 service-policy output SHAPE-200M
```

Fuller truth: many Metro Ethernet services run over the carrier's MPLS network. The customer sees Ethernet, and the carrier sees pseudowires or VPLS.

### DSL

DSL (Digital Subscriber Line) sends data over old telephone copper. Voice uses frequencies below 4 kHz. DSL uses much higher frequencies on the same wire.

```text
 Home                  Copper pair               Exchange or cabinet
+-----------+                                   +-------+
| DSL modem |-----------------------------------| DSLAM |---> ISP
+-----------+                                   +-------+
```

The **DSLAM** (DSL Access Multiplexer) is the carrier box that ends many DSL lines. High frequencies fade quickly on copper, so **speed falls as distance grows**.

| Type | Typical top speed (down / up) | Good distance |
|---|---|---|
| ADSL2+ | Up to 24 Mbps / about 1 Mbps | Falls fast after 2-3 km |
| VDSL2 | Up to about 100 Mbps / 40 Mbps | Best under about 1 km |
| VDSL2 35b (supervectoring) | Up to about 300 Mbps down | Best under about 500 m |
| G.fast | Hundreds of Mbps, up to about 1 Gbps total | Best under about 100-200 m |

The "A" in ADSL means asymmetric: download is faster than upload. That suits homes, but not a server site.

To shorten the copper, carriers run fibre to a street cabinet and use VDSL2 for the last few hundred metres. This design is called **FTTC** (Fibre to the Cabinet). Many carriers are now replacing copper with fibre to the home.

### Cable

Cable broadband uses the coaxial cable built for cable TV. The standard is **DOCSIS** (Data Over Cable Service Interface Specification).

```text
                         fibre            coax (shared)
+------+   +------------+      +-------+      +--------------+
| CMTS |---| Fibre ring |------| Optic |------| Cable modems |
+------+   +------------+      | node  |      | in many homes|
                               +-------+      +--------------+
```

The **CMTS** (Cable Modem Termination System) is the carrier box. Fibre reaches a neighbourhood node, and coax covers the final stretch. This mix is called **HFC** (Hybrid Fibre-Coaxial).

| DOCSIS version | Top speed in the standard (down / up) |
|---|---|
| 3.0 | About 1 Gbps / about 200 Mbps, depending on channels |
| 3.1 | About 10 Gbps / 1-2 Gbps |
| 4.0 | About 10 Gbps / about 6 Gbps |

The coax segment is **shared**, so everyone on the node competes for the same capacity. Speed can drop in the evening. Upload is usually much slower than download. Real plans are far below the top numbers above.

### Fibre PON

PON (Passive Optical Network) brings fibre to the home or office without any powered gear in the street. "Passive" means the splitter uses no electricity.

```text
 Exchange              Street cabinet          Customers
+-----+   one fibre   +----------+    fibre   +-----+
| OLT |---------------| Passive  |------------| ONT | Customer 1
+-----+   up to 20 km | splitter |            +-----+
                      |   1:32   |------------| ONT | Customer 2
                      |          |            +-----+
                      |          |------------| ONT | ... up to 32
                      +----------+            +-----+
```

- **OLT (Optical Line Terminal):** the carrier equipment at the exchange.
- **ONT (Optical Network Terminal):** the box at the customer. Some standards say ONU (Optical Network Unit).
- **Splitter:** a piece of glass that copies light into many fibres. Ratios of 1:32 and 1:64 are common.

Traffic moves differently in each direction:

- **Downstream:** the OLT sends all data to all ONTs. Each ONT keeps only its own data. Encryption stops a neighbour reading it.
- **Upstream:** the OLT gives each ONT its own time slots, so their light never collides. This is TDMA (Time Division Multiple Access).

| Standard | Downstream | Upstream | Wavelengths (down / up) |
|---|---|---|---|
| GPON (ITU-T G.984) | 2.488 Gbps | 1.244 Gbps | 1490 nm / 1310 nm |
| XGS-PON (ITU-T G.9807.1) | About 10 Gbps | About 10 Gbps | 1577 nm / 1270 nm |
| EPON (IEEE 802.3ah) | 1 Gbps | 1 Gbps | 1490 nm / 1310 nm |

Those speeds are shared by every customer on that splitter. With 32 users on GPON, the average is about 78 Mbps each if all download at once. In practice they do not all download at once, so a 1 Gbps plan works well. Faster systems, such as 50G-PON (ITU-T G.9804), are being rolled out in some networks.

Because the fibre is shared, PON broadband is not the same as a dedicated fibre leased line, even though both are "fibre".

### 4G and 5G WAN

A router with a SIM card can use the mobile network as a WAN link. The radio side is covered in [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md). Here we look at using it as a WAN.

```text
+--------+  radio  +--------------+      +-------------+
| Router |---------| Base station |------| Mobile core |---> Internet
| + SIM  |         | eNB / gNB    |      | EPC / 5GC   |     or private
+--------+         +--------------+      +-------------+     network
```

The base station is called an eNodeB in 4G and a gNodeB in 5G. The mobile core is the EPC in 4G and the 5GC in 5G.

| Type | Peak on paper | Common real speed | Common real latency |
|---|---|---|---|
| 4G LTE (Cat 4) | 150 Mbps down, 50 Mbps up | 10-100 Mbps | 30-60 ms |
| 5G mid-band | Hundreds of Mbps to over 1 Gbps | 100-500 Mbps | 15-40 ms |

Real values depend heavily on signal, cell load and carrier.

Common uses:

- **Backup link** when the fixed line fails.
- **Temporary sites**, such as a construction site or an event.
- **Hard-to-reach sites**, such as a kiosk, an ATM or a truck.
- **FWA (Fixed Wireless Access):** 5G used as the main broadband for a building.

Things that catch people out:

- **CGNAT (Carrier-Grade NAT):** many SIMs get a private address behind the carrier's NAT. You can start connections outward, but nobody can reach you from outside. This works for a DMVPN spoke, but not for a hub. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).
- **Private APN:** an APN (Access Point Name) picks the network your SIM joins. A private APN can connect SIMs straight to your company network. In 5G the same idea is called a DNN (Data Network Name).
- **Variable delay:** jitter can be high, so voice quality may suffer.
- **Data caps:** a backup link can run up a large bill if traffic stays on it.
- **Lower MTU:** some carriers recommend an MTU below 1500. Check with the carrier.

### PPP

PPP (Point-to-Point Protocol) is a Layer 2 protocol for one link between two devices. Its core standard is RFC 1661. It does four useful jobs that plain HDLC does not: it negotiates options, authenticates, assigns addresses and detects link quality problems.

PPP has two main parts:

- **LCP (Link Control Protocol):** brings up the link, agrees on options and tests it.
- **NCP (Network Control Protocol):** one per network protocol. **IPCP** sets up IPv4, and **IPv6CP** sets up IPv6.

A PPP link comes up in phases:

```text
 Dead --> Establish --> Authenticate --> Network  --> Open --> Terminate
          (LCP)         (PAP or CHAP)    (IPCP,       data
                                         IPv6CP)      flows
```

1. **Dead:** no carrier signal.
2. **Establish:** LCP agrees on options, such as the maximum frame size and the authentication method.
3. **Authenticate:** optional. Each side proves who it is.
4. **Network:** IPCP gives the addresses, and the link can carry IP.
5. **Terminate:** either side can close the link cleanly.

PPP frame format:

| Field | Size | Value or meaning |
|---|---|---|
| Flag | 1 byte | 0x7E, marks the start |
| Address | 1 byte | 0xFF, there is only one possible receiver |
| Control | 1 byte | 0x03 |
| Protocol | 2 bytes | 0x0021 IPv4, 0x0057 IPv6, 0xC021 LCP, 0x8021 IPCP, 0xC023 PAP, 0xC223 CHAP |
| Information | Variable | The payload, default maximum 1500 bytes |
| FCS | 2 bytes | Error check (can be 4 bytes) |
| Flag | 1 byte | 0x7E, marks the end |

There are two authentication methods:

- **PAP (Password Authentication Protocol):** sends the username and password in clear text. Avoid it.
- **CHAP (Challenge Handshake Authentication Protocol):** never sends the secret on the wire.

```text
 R1 (checks identity)                      R2 (proves identity)
   |                                               |
   |---- 1. Challenge: ID + random number -------->|
   |                                               |
   |<--- 2. Response: MD5(ID + secret + random) ---|
   |                                               |
   |---- 3. Success or Failure ------------------->|
```

R1 calculates the same hash from its own copy of the secret and compares the two. Because the random number changes each time, an attacker cannot replay an old answer.

Cisco IOS example on a serial link. Each router stores the other router's hostname with the shared secret.

```text
! R1
hostname R1
username R2 password LabSecret1
!
interface Serial0/0/0
 ip address 10.255.0.1 255.255.255.252
 encapsulation ppp
 ppp authentication chap

! R2
hostname R2
username R1 password LabSecret1
!
interface Serial0/0/0
 ip address 10.255.0.2 255.255.255.252
 encapsulation ppp
 ppp authentication chap
```

```text
R1#show interfaces serial 0/0/0
Serial0/0/0 is up, line protocol is up
  Hardware is GT96K Serial
  Internet address is 10.255.0.1/30
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
  Encapsulation PPP, LCP Open
  Open: IPCP, CDPCP, loopback not set
```

"LCP Open" means the link is negotiated. "Open: IPCP" means IPv4 is ready.

### PPPoE

PPPoE (PPP over Ethernet, RFC 2516) carries PPP frames inside Ethernet frames. Ethernet has no login, so it cannot tell an ISP which customer is connecting. PPP has a login. PPPoE gives broadband ISPs the login, per-customer sessions and address assignment that they had on dial-up.

On the ISP side, the session ends on a **BNG (Broadband Network Gateway)**, also called a BRAS (Broadband Remote Access Server). The BNG usually checks your login against a RADIUS server.

PPPoE has two stages. **Discovery** finds the BNG. **Session** carries the data.

```text
 Client                                         BNG (ISP)
   |--- PADI: anyone there? (broadcast) ------------>|
   |<-- PADO: I offer service (unicast) -------------|
   |--- PADR: I choose you ------------------------->|
   |<-- PADS: confirmed, Session ID = 7 -------------|
   |                                                 |
   |=== PPP: LCP, then CHAP, then IPCP =============>|
   |=== IP data inside PPPoE session 7 =============>|
   |                                                 |
   |--- PADT: end session (either side can send) --->|
```

The names mean: Active Discovery Initiation, Offer, Request, Session-confirmation and Terminate.

Ethernet uses EtherType 0x8863 for discovery and 0x8864 for session data.

```text
 PPPoE session packet on Ethernet

+----------+-------+-----+---------+-------------------+
| Ethernet | PPPoE | PPP | IP      | TCP/UDP + data    |
| header   | 6 B   | 2 B | header  |                   |
+----------+-------+-----+---------+-------------------+
           |<--- 8 B --->|<----- at most 1492 B ------>|
           |<----- Ethernet payload: 1500 B ---------->|
```

| PPPoE field | Size | Meaning |
|---|---|---|
| Version | 4 bits | 1 |
| Type | 4 bits | 1 |
| Code | 1 byte | 0x00 session data, 0x09 PADI, 0x07 PADO, 0x19 PADR, 0x65 PADS, 0xA7 PADT |
| Session ID | 2 bytes | Number that identifies this session |
| Length | 2 bytes | Payload length, including the PPP Protocol field |

**This is why the MTU is 1492.** Ethernet allows 1500 bytes of payload. PPPoE uses 6 bytes and the PPP Protocol field uses 2. That leaves 1492 for IP. The TCP MSS (Maximum Segment Size) then becomes 1492 - 20 - 20 = 1452. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

Fuller truth: many fibre ISPs now use **IPoE** (IP over Ethernet) instead, which is plain DHCP with no PPP layer. The customer's line is identified by the access network instead of a login.

### GRE

GRE (Generic Routing Encapsulation, RFC 2784) wraps one packet inside another. It creates a virtual point-to-point link across a network that does not know about your private addresses. GRE is IP protocol number 47.

```text
                     Original packet
                 +-----------+----------+
                 | IP header | Payload  |
                 | 10.1.0.5  |          |
                 | 10.2.0.9  |          |
                 +-----------+----------+
                         | GRE wraps it
                         v
 +----------------+------+-----------+----------+
 | New IP header  | GRE  | IP header | Payload  |
 | 203.0.113.2 to | 4 B  | 10.1.0.5  |          |
 | 198.51.100.2   |      | 10.2.0.9  |          |
 | Protocol 47    |      |           |          |
 +----------------+------+-----------+----------+
```

Internet routers look only at the new header. They never see the 10.x addresses.

| GRE field | Size | Meaning |
|---|---|---|
| C flag | 1 bit | A checksum is present |
| K flag | 1 bit | A key is present (RFC 2890) |
| S flag | 1 bit | A sequence number is present (RFC 2890) |
| Reserved | 10 bits | Set to zero (bit 1, then bits 4-12) |
| Version | 3 bits | 0 |
| Protocol Type | 2 bytes | Type of inner packet, 0x0800 for IPv4, 0x86DD for IPv6 |
| Optional fields | 4 bytes each | Checksum and reserved, key, sequence number |

Basic overhead is **24 bytes**: 20 for the new IPv4 header and 4 for GRE. A 1500-byte path leaves 1476 bytes for the inner packet.

GRE has three key facts:

- **It does not encrypt.** Anyone on the path can read the inner packet.
- **It carries multicast and many protocols.** So routing protocols, such as OSPF and EIGRP, can run over it. See [OSPF](../05-routing/06-ospf.md).
- **It keeps no state.** The tunnel has no handshake. On Cisco IOS, a tunnel can show up/up while the far end is dead, unless you add keepalives.

GRE and IPsec complement each other. GRE gives you multicast and routing, and IPsec gives you encryption. Together, this is called **GRE over IPsec**. See [IPsec](../09-security/08-ipsec.md) and [VPNs](../09-security/07-vpn.md).

### DMVPN

Plain GRE needs one tunnel configuration per peer. With 100 branches, that means 100 tunnels at the hub and a lot of typing. **DMVPN (Dynamic Multipoint VPN)** solves this. It is a Cisco design built from standard parts.

| Part | Job |
|---|---|
| mGRE (multipoint GRE) | One tunnel interface that talks to many peers. There is no fixed `tunnel destination`. |
| NHRP (Next Hop Resolution Protocol, RFC 2332) | A directory that maps tunnel IP to public IP |
| Routing protocol | EIGRP, OSPF or BGP over the tunnel, to share LAN routes |
| IPsec | Optional but nearly always used, to encrypt the traffic |

Two more terms:

- **NHS (Next Hop Server):** the hub. It keeps the directory.
- **NHC (Next Hop Client):** a spoke. It registers with the hub.

**NBMA address** is the DMVPN word for the public underlay address. NBMA means Non-Broadcast Multi-Access: many devices are connected, but no broadcast reaches them all.

Only the hub needs a fixed public address. Spokes can have dynamic addresses, because they tell the hub their current address when they register.

```text
                     +-------------------+
                     | Hub (NHS)         |
                     | Tunnel 172.16.100.1
                     | Public 203.0.113.10
                     +---+-----------+---+
          permanent      |           |      permanent
          tunnel         |           |      tunnel
                         |           |
             +-----------+--+     +--+-----------+
             | Spoke 1      |     | Spoke 2      |
             | LAN          |     | LAN          |
             | 10.11.0.0/24 |     | 10.12.0.0/24 |
             +------+-------+     +-------+------+
                    |                     |
                    +---------------------+
                     on-demand direct tunnel
```

DMVPN has three phases. They describe how spoke-to-spoke traffic works.

| Phase | Spoke tunnel type | Spoke to spoke | Can the hub summarise routes? | Notes |
|---|---|---|---|---|
| 1 | Point-to-point GRE | Always through the hub | Yes | Simple, but the hub is a choke point |
| 2 | mGRE | Direct, on demand | No, spokes need the exact next hop | Routing protocol must keep the far spoke as next hop |
| 3 | mGRE | Direct, on demand | Yes | Hub sends a redirect. Best scaling. |

How Phase 3 builds a direct tunnel:

1. Each spoke boots, builds a tunnel to the hub and sends an **NHRP registration**: "tunnel IP 172.16.100.11 is at public IP 198.51.100.11."
2. A host on Spoke 1 sends traffic to Spoke 2's LAN. Spoke 1's route points to the hub, so the first packets go through the hub.
3. The hub sees the packet enter and leave on the same tunnel. It forwards the packet, and it also sends an **NHRP redirect** to Spoke 1: "there is a better path."
4. Spoke 1 sends an **NHRP resolution request** for the destination. The hub passes it to Spoke 2.
5. Spoke 2 replies directly to Spoke 1 with its public address and LAN prefix.
6. Spoke 1 installs a shortcut route. If IPsec is on, it first builds an IPsec session to Spoke 2. New packets now go straight across.
7. When traffic stops, the entry expires and the direct tunnel goes away.

### Hub and Spoke vs Full Mesh

Topology means the shape of the connections. See [Network Topologies](../01-foundations/03-network-topologies.md).

```text
 Hub and spoke (5 sites, 4 links)     Full mesh (4 sites, 6 links)

     S1      S2                            A-------B
       \    /                              | \   / |
        \  /                               |  \ /  |
         HUB                               |   X   |
        /  \                               |  / \  |
       /    \                              | /   \ |
     S3      S4                            C-------D
```

In **hub and spoke**, all sites connect to a central site. For n sites, you need n - 1 links.

In **full mesh**, every site connects to every other site. For n sites, you need n(n - 1) / 2 links.

| Sites | Hub and spoke links | Full mesh links |
|---|---|---|
| 5 | 4 | 10 |
| 10 | 9 | 45 |
| 50 | 49 | 1,225 |
| 200 | 199 | 19,900 |

| Point | Hub and spoke | Full mesh |
|---|---|---|
| Cost and setup | Low | High, grows with the square of n |
| Spoke-to-spoke delay | Higher, traffic detours through the hub | Lowest, direct path |
| Failure risk | Hub is a single point of failure | Very resilient |
| Bandwidth | Hub needs a large link | Each site needs less |
| Fits | Branches that mostly talk to a data centre | Small numbers of busy core sites |

A **partial mesh** sits in between: the busy sites are meshed and the small ones connect to two hubs. Most real networks look like this. DMVPN gives a hub-and-spoke configuration that behaves like a mesh when traffic needs it. An MPLS L3VPN gives any-to-any routing without you building any tunnels.

Delay example with made-up but realistic numbers. Spoke A to hub is 40 ms, and hub to Spoke B is 50 ms. Through the hub, the one-way delay is 90 ms. If a direct path takes 15 ms, then a voice call on the hub path has 6 times the delay. This detour is called **hairpinning** or **tromboning**.

### Comparison Table

Speeds and latency below are common 2026 values. They vary a lot by country, contract and distance. Cost is a relative guide.

| Technology | What you buy | Common speeds | Symmetric? | Latency and jitter | Privacy | SLA | Cost | Best fit |
|---|---|---|---|---|---|---|---|---|
| Leased line (TDM) | Layer 1 circuit | 1.5 Mbps to 10 Gbps | Yes | Very low, stable | Private path | Strong | Very high | Legacy systems |
| Dedicated Ethernet or DIA | Layer 2 circuit or uncontended internet | 100 Mbps to 100 Gbps | Yes | Low, stable | Private for point to point. DIA is internet. | Strong | High | HQ, data centres |
| MPLS L3VPN | Private routed network | 10 Mbps to 10 Gbps | Yes | Low, stable | Separated, not encrypted | Strong, with QoS | High | Branches needing voice and SLA |
| Metro Ethernet | Layer 2 service | 10 Mbps to 100 Gbps | Yes | Low, stable | Separated, not encrypted | Strong | Medium to high | Sites in the same region |
| DSL | Shared internet | 10-300 Mbps down | No | Medium | Public | Weak | Low | Small sites, backup |
| Cable | Shared internet | 100 Mbps to 2 Gbps down | Mostly no | Medium, can rise at peak | Public | Weak | Low | Small sites, home workers |
| Fibre PON | Shared internet | 100 Mbps to 10 Gbps | Often yes | Low | Public | Weak to medium | Low to medium | Small and mid sites |
| 4G LTE | Mobile data | 10-100 Mbps | No | Medium to high, jitter | Public or private APN | Weak | Medium, data charged | Backup, temporary sites |
| 5G | Mobile data or FWA | 100-500 Mbps | No | Low to medium, jitter | Public or private APN | Weak to medium | Medium | Fast backup, FWA |

Overlay methods run on top of the links above:

| Method | Encrypts? | Multicast and routing protocols? | Spoke-to-spoke on demand? | Config per site |
|---|---|---|---|---|
| GRE | No | Yes | No | One tunnel per peer |
| Policy-based IPsec | Yes | No | No | ACLs per peer |
| GRE over IPsec | Yes | Yes | No | One tunnel per peer |
| DMVPN with IPsec | Yes | Yes | Yes (Phase 2 and 3) | Same small config at each spoke |
| SD-WAN | Yes | Vendor dependent | Yes | Pushed from a controller |

[SD-WAN](10-sd-wan.md) has its own chapter. It uses several links at once and picks the best path per application. For connecting to cloud providers, see [Hybrid Cloud Connectivity](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).

## Example

### Example 1: GRE Tunnel Between Two Sites

Two sites want their private LANs to talk across the internet.

```text
   LAN A                                               LAN B
 10.1.0.0/24                                        10.2.0.0/24
     |                                                   |
 +---+----+ G0/0   203.0.113.2     198.51.100.2 G0/0 +---+----+
 |   R1   |------------- Internet -------------------|   R2   |
 +--------+                                          +--------+
 Tunnel0 ==================== GRE ======================= Tunnel0
 172.16.0.1/30                                      172.16.0.2/30
```

Cisco IOS on R1. R2 is the mirror image.

```text
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.252
!
interface GigabitEthernet0/1
 ip address 10.1.0.1 255.255.255.0
!
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 keepalive 10 3
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.2
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
ip route 10.2.0.0 255.255.255.0 172.16.0.2
```

The default route reaches the far public address. The second route sends LAN B traffic into the tunnel. `ip mtu 1400` leaves room to add IPsec later, and `ip tcp adjust-mss 1360` makes TCP use segments that fit (1400 - 40 = 1360).

Verify:

```text
R1#show interfaces tunnel 0
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 172.16.0.1/30
  MTU 17916 bytes, BW 100 Kbit/sec, DLY 50000 usec,
  Encapsulation TUNNEL, loopback not set
  Keepalive set (10 sec), retries 3
  Tunnel source 203.0.113.2 (GigabitEthernet0/0), destination 198.51.100.2
  Tunnel protocol/transport GRE/IP

R1#ping 10.2.0.1 source 10.1.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.1.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 18/20/24 ms
```

The same tunnel on a Linux router:

```bash
# Linux (iproute2). Run the mirror image on the far side.
sudo ip tunnel add gre1 mode gre local 203.0.113.2 remote 198.51.100.2 ttl 255
sudo ip addr add 172.16.0.1/30 dev gre1
sudo ip link set gre1 mtu 1400 up
sudo ip route add 10.2.0.0/24 via 172.16.0.2
```

Windows desktop editions have no simple built-in command for GRE. Windows Server can do it through the Routing and Remote Access role.

### Example 2: DMVPN Phase 3 With One Hub and Two Spokes

| Device | Public address | Tunnel address | LAN |
|---|---|---|---|
| R1-HUB | 203.0.113.10 | 172.16.100.1/24 | 10.10.0.0/24 |
| R2-SPOKE | 198.51.100.11 | 172.16.100.11/24 | 10.11.0.0/24 |
| R3-SPOKE | 192.0.2.12 | 172.16.100.12/24 | 10.12.0.0/24 |

Cisco IOS on the hub:

```text
interface Tunnel0
 ip address 172.16.100.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication NHRPKEY1
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp redirect
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
router eigrp 100
 network 10.10.0.0 0.0.0.255
 network 172.16.100.0 0.0.0.255
```

Cisco IOS on R2-SPOKE. R3 is the same except for its tunnel address and LAN.

```text
interface Tunnel0
 ip address 172.16.100.11 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication NHRPKEY1
 ip nhrp network-id 100
 ip nhrp holdtime 600
 ip nhrp nhs 172.16.100.1 nbma 203.0.113.10 multicast
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel key 100
!
router eigrp 100
 network 10.11.0.0 0.0.0.255
 network 172.16.100.0 0.0.0.255
```

Key lines:

- `ip nhrp map multicast dynamic`: the hub sends routing multicast to every spoke that registers.
- `ip nhrp redirect` (hub) and `ip nhrp shortcut` (spoke): these two lines make it Phase 3.
- `ip summary-address eigrp ...`: spokes learn one 10.0.0.0/8 route toward the hub.
- `ip nhrp nhs ... nbma ... multicast`: this one-line form needs a modern IOS. Older releases use separate `ip nhrp map` lines.

For encryption, add `tunnel protection ipsec profile <name>` to each tunnel. See [IPsec](../09-security/08-ipsec.md).

Check the hub. Both spokes have registered, and D means they were learned dynamically.

```text
R1-HUB#show dmvpn
(legend removed)
Interface: Tunnel0, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1   198.51.100.11   172.16.100.11    UP 00:21:12     D
     1      192.0.2.12   172.16.100.12    UP 00:20:47     D
```

Now trace from Spoke 1 to Spoke 2's LAN twice. The first time, the path goes through the hub. That traffic triggers the redirect.

```text
R2-SPOKE#traceroute 10.12.0.1 source 10.11.0.1
Type escape sequence to abort.
Tracing the route to 10.12.0.1
VRF info: (vrf in name/id, vrf out name/id)
  1 172.16.100.1 44 msec 40 msec 44 msec
  2 172.16.100.12 88 msec 84 msec 84 msec

R2-SPOKE#traceroute 10.12.0.1 source 10.11.0.1
Type escape sequence to abort.
Tracing the route to 10.12.0.1
VRF info: (vrf in name/id, vrf out name/id)
  1 172.16.100.12 22 msec 20 msec 22 msec
```

The second trace goes direct, and the round trip falls from about 88 ms to about 22 ms.

```text
R2-SPOKE#show ip nhrp
10.12.0.0/24 via 172.16.100.12
   Tunnel0 created 00:00:14, expire 00:09:45
   Type: dynamic, Flags: router rib nho
   NBMA address: 192.0.2.12
172.16.100.1/32 via 172.16.100.1
   Tunnel0 created 00:21:30, never expire
   Type: static, Flags: used
   NBMA address: 203.0.113.10

R2-SPOKE#show ip route
(other routes removed)
D        10.0.0.0/8 [90/26880256] via 172.16.100.1, 00:21:05, Tunnel0
H        10.12.0.0/24 [250/1] via 172.16.100.12, 00:00:14, Tunnel0
```

The route with code H is the NHRP shortcut. It is more specific than the /8 summary, so it wins.

### Example 3: Prove the PPPoE MTU

A client has a PPPoE session. Cisco IOS client config:

```text
interface GigabitEthernet0/0
 no ip address
 pppoe enable group global
 pppoe-client dial-pool-number 1
!
interface Dialer1
 ip address negotiated
 encapsulation ppp
 dialer pool 1
 ppp chap hostname user1@example.com
 ppp chap password ExamplePass1
 ip mtu 1492
 ip tcp adjust-mss 1452
!
ip route 0.0.0.0 0.0.0.0 Dialer1
```

```text
R1#show ip interface brief | include Dialer
Dialer1              203.0.113.77    YES IPCP   up          up
```

The method "IPCP" shows that the ISP assigned this address through PPP.

Now test from a host that holds the PPPoE session itself. The 28 bytes added to each ping are 20 for the IP header and 8 for the ICMP header. So 1464 + 28 = 1492.

Linux:

```bash
$ ip link show ppp0
5: ppp0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1492 qdisc fq_codel state UNKNOWN mode DEFAULT group default qlen 3
    link/ppp

$ ping -M do -s 1464 -c 1 198.51.100.50
PING 198.51.100.50 (198.51.100.50) 1464(1492) bytes of data.
1472 bytes from 198.51.100.50: icmp_seq=1 ttl=57 time=14.2 ms

$ ping -M do -s 1465 -c 1 198.51.100.50
PING 198.51.100.50 (198.51.100.50) 1465(1493) bytes of data.
ping: local error: message too long, mtu=1492
```

Windows:

```powershell
PS> ping -f -l 1464 198.51.100.50
Pinging 198.51.100.50 with 1464 bytes of data:
Reply from 198.51.100.50: bytes=1464 time=14ms TTL=57

PS> ping -f -l 1465 198.51.100.50
Pinging 198.51.100.50 with 1465 bytes of data:
Packet needs to be fragmented but DF set.
```

The size that works and the size that fails sit exactly on the 1492 limit.

| Link | MTU for inner IP | TCP MSS |
|---|---|---|
| Plain Ethernet | 1500 | 1460 |
| PPPoE | 1492 | 1452 |
| GRE over Ethernet | 1476 | 1436 |
| GRE over PPPoE | 1468 | 1428 |
| Common DMVPN setting | 1400 | 1360 |

### Example 4: Choosing a WAN for a Retail Chain

A chain has 120 shops and 2 data centres. Shops send card payments and stock data to the data centres. Shops almost never talk to each other.

1. **Traffic pattern:** branch to data centre. Hub and spoke fits. Full mesh would need 122 x 121 / 2 = 7,381 links, for no benefit.
2. **Shop primary link:** PON or cable where possible, DSL otherwise. Each shop only needs about 50 Mbps.
3. **Shop backup:** 5G or LTE router, as a different carrier and a different cable path.
4. **Overlay:** DMVPN Phase 3 with IPsec, or SD-WAN. Card data crosses the internet, so encryption is required.
5. **Hubs:** two hubs, one in each data centre. Each shop has a tunnel to both.
6. **Hub bandwidth:** 120 x 50 Mbps = 6 Gbps at worst. If we assume about 20% are busy at once, the need is 1.2 Gbps. Buy 2 Gbps of DIA at each data centre. That 20% is an assumption, so check it with real traffic data.

Result: low cost per shop, encrypted traffic, and a backup path. MPLS is not needed here, because the shops do not need strict delay guarantees.

## Going Deeper

### Older Technologies Still Asked About

- **ISDN (Integrated Services Digital Network):** digital dial-up over phone lines. BRI gave two 64 kbps channels. It is largely retired.
- **Frame Relay:** a shared packet WAN that used virtual circuits. **DLCI** numbers identified each circuit. It introduced many people to NBMA problems, such as OSPF needing special network types.
- **ATM (Asynchronous Transfer Mode):** a carrier technology using fixed 53-byte cells. Many older DSL lines ran over it.

MPLS, Metro Ethernet and internet VPNs have replaced these in almost all new builds. Know the names, not the configuration.

### MPLS From the Customer's Side

If all your sites use one BGP AS number, a route from Site 1 reaches Site 2 with the site's own AS in the path. BGP drops it as a loop. Two fixes exist:

- The carrier sets `as-override` on the PE, which replaces your AS with its own in the path.
- The customer sets `allowas-in` on the CE.

Using a different private AS per site also avoids it. Example CE using AS 65010 toward a carrier in AS 64512:

```text
router bgp 65010
 neighbor 10.255.1.1 remote-as 64512
 address-family ipv4
  neighbor 10.255.1.1 activate
  network 10.20.1.0 mask 255.255.255.0
```

Interface speed is rarely the same as the bandwidth you paid for, so shape on the CE and mark traffic so that the carrier can honour your classes. Carriers often sell a small number of classes, such as real-time, business and best effort. See [BGP](../05-routing/09-bgp.md).

### DMVPN Routing Details

- **Phase 1:** the hub can change the next hop to itself, since all traffic flows through it.
- **Phase 2:** the hub must keep the original spoke as next hop. On EIGRP, that means `no ip next-hop-self eigrp 100` and `no ip split-horizon eigrp 100`. With OSPF, use network type broadcast, and make the hub the DR with spokes at priority 0.
- **Phase 3:** routing can be plain, because NHRP fixes the path. OSPF point-to-multipoint or BGP are common. The hub can send just a default route or a summary.
- **OSPF limits:** all spokes in one area see every topology change. Large networks tend to prefer EIGRP or BGP.
- **Dual hub:** run two hubs, either as two NHS servers on one tunnel or as two separate tunnels (dual cloud). See [High Availability and Redundancy](07-high-availability.md).
- **NAT:** IPsec NAT traversal wraps traffic in UDP port 4500. Spokes behind NAT can build tunnels to a hub with a reachable address. Direct spoke-to-spoke paths may not form when both sides are behind NAT, so traffic may stay on the hub path.
- **Other vendors:** DMVPN is Cisco's design. Others have similar ones, such as Fortinet ADVPN. New builds often choose SD-WAN instead.

### Recursive Routing

A tunnel needs a route to its destination through the underlay. If the routing protocol learns the far public address through the tunnel itself, the tunnel tries to run over itself. Cisco IOS then shuts the tunnel for a while:

```text
%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
```

Fixes:

- Do not advertise public addresses into the tunnel routing process.
- Use static routes for the underlay, or filter routes.
- Use **front-door VRF**: put the public interface in its own VRF, so that the underlay and overlay use separate routing tables.

### GRE and Tunnel Details

- A GRE key (RFC 2890) adds 4 bytes, so `tunnel key 100` makes the overhead 28 bytes. It lets a router tell several tunnels apart when they share one source address.
- GRE keepalives work on point-to-point tunnels. mGRE does not support them, so DMVPN uses NHRP and routing protocol hellos to detect failed peers.
- GRE has no ports, so normal PAT cannot track it easily. IPsec with NAT traversal avoids that problem.
- GRE can run over IPv6 too, with a larger header. An IPv6 outer header is 40 bytes.
- `show interfaces tunnel` may show MTU 17916 on modern IOS. That is the interface MTU, not what the path allows. The effective default IP MTU over a 1500-byte path is 1476. Set `ip mtu` yourself.

### PPP Extras

- **Multilink PPP (RFC 1990):** joins several links into one logical link, such as several E1s. It splits packets across the links. It is rarely used on modern fibre.
- **RFC 4638:** lets PPPoE use a 1500-byte MTU, if the Ethernet path supports payloads of at least 1508 bytes. Support depends on the ISP and on the equipment.
- **VLAN on PPPoE:** many fibre ISPs require a specific VLAN tag on the PPPoE link. If discovery gets no PADO reply, check this first.

### SLA Numbers

| Uptime | Downtime per year |
|---|---|
| 99.9% | About 8.8 hours |
| 99.99% | About 53 minutes |
| 99.999% | About 5.3 minutes |

Two links only protect you if they do not share a failure. Two lines from two ISPs can still run through the same street duct. Ask carriers for **path diversity** in writing.

## Common Mistakes

- **"MPLS is encrypted."** MPLS separates customers using VRFs and labels. The carrier can still read your packets. Add IPsec if you need secrecy.
- **"GRE is a VPN, so it is secure."** GRE only wraps packets. It gives no encryption and no authentication.
- **"PON is the same as a leased line because both are fibre."** PON is shared and usually has a weak SLA. A leased line is dedicated.
- **"A 1 Gbps port means 1 Gbps of service."** You get the CIR. The carrier may drop anything above it. Shape to the CIR.
- **Forgetting MTU on tunnels and PPPoE.** Small pings work, but big transfers hang. Set `ip mtu` and `ip tcp adjust-mss`, and make sure ICMP "fragmentation needed" messages are not blocked.
- **Testing a tunnel with a default ping.** A 100-byte ping proves little. Test with large packets and DF set.
- **Trusting tunnel status.** A GRE tunnel with no keepalives can show up/up while the far end is dead. Use keepalives or a routing protocol.
- **Encapsulation mismatch on a serial link.** If one side uses HDLC and the other uses PPP, the interface shows up/down. The same happens when CHAP fails.
- **Believing a DMVPN spoke needs a static public address.** Only the hub does. Spokes register their current address.
- **Trying to make a CGNAT mobile link a DMVPN hub.** No one can start a connection inward to it. Use a public address at the hub.
- **Calling DMVPN a full mesh.** The tunnels to the hub are permanent. Spoke-to-spoke tunnels exist only when traffic needs them.
- **Thinking two links always mean redundancy.** They may share the same duct, the same exchange or the same upstream carrier.
- **Choosing full mesh out of habit.** Check the traffic. If branches only talk to the data centre, a mesh wastes money.

## Interview Questions

**Q: How do a leased line, MPLS and an internet VPN differ?**
A: A leased line is a dedicated link with a fixed speed between two points, with a strong SLA but high cost. MPLS is a private carrier network that connects many sites, with any-to-any routing, QoS and an SLA, but it does not encrypt. An internet VPN uses cheap public links with IPsec on top, so it is encrypted, but delay is not guaranteed.

**Q: What do CE, PE and P mean in MPLS?**
A: CE is the customer's router, and it does not use labels. PE is the carrier's edge router. It holds a VRF for each customer and adds and removes labels. P is a core router that only switches labels, and it has no customer routes.

**Q: What is the difference between E-Line, E-LAN and E-Tree?**
A: E-Line is a point-to-point Ethernet service, like a virtual cable. E-LAN is multipoint, like a virtual switch between many sites. E-Tree is rooted multipoint, where leaves talk to the root but not to each other.

**Q: Why does DSL slow down with distance, and how do carriers fix it?**
A: DSL uses high frequencies on copper, and those fade quickly with distance. So carriers shorten the copper by running fibre to a street cabinet and using VDSL2 for the last few hundred metres. The long-term fix is fibre to the premises.

**Q: How does PON avoid collisions on the upstream?**
A: Downstream is broadcast to all ONTs, and each one keeps only its own data, which is encrypted. Upstream uses time division. The OLT gives every ONT its own time slots, so their signals never overlap on the shared fibre.

**Q: Why is the PPPoE MTU 1492?**
A: Ethernet carries 1500 bytes of payload. PPPoE adds a 6-byte header and PPP adds a 2-byte protocol field. That leaves 1492 bytes for IP, so the TCP MSS should be 1452.

**Q: What are the phases of PPP, and what do LCP and NCP do?**
A: The link goes through establish, authenticate, network and terminate. LCP sets up and tests the link and agrees on options. NCP then sets up each network protocol, such as IPCP for IPv4, which can also assign an address.

**Q: How does CHAP differ from PAP?**
A: PAP sends the password in clear text. CHAP sends a random challenge, and the peer replies with an MD5 hash of the ID, the secret and the challenge. So the secret never crosses the wire, and old answers cannot be replayed.

**Q: What is GRE, and why use it with IPsec?**
A: GRE is a tunnel protocol, IP protocol 47, that adds 24 bytes in total. It carries multicast and routing protocols but does not encrypt. IPsec encrypts but policy-based IPsec does not carry multicast, so GRE over IPsec gives both.

**Q: What makes up DMVPN, and what does each part do?**
A: mGRE gives one tunnel interface for many peers. NHRP is the directory that maps tunnel addresses to public addresses. A routing protocol shares LAN routes, and IPsec provides encryption. Together they let one small hub config serve many spokes, and spokes can have dynamic addresses.

**Q: How do DMVPN Phase 1, 2 and 3 differ?**
A: Phase 1 sends all spoke traffic through the hub. Phase 2 allows direct spoke-to-spoke tunnels, but the hub cannot summarise, because spokes need the exact next hop. Phase 3 lets the hub send an NHRP redirect, so the spokes build shortcuts even when the hub only sends a summary or default route. That makes Phase 3 the best choice for scale.

**Q: You have 200 branches. Would you use a full mesh?**
A: No. A full mesh needs 200 x 199 / 2, which is 19,900 links. If branches mostly talk to the data centre, I would use hub and spoke with two hubs. For direct branch traffic, I would use DMVPN Phase 3 or SD-WAN, since these build direct paths only when needed.

## Quick Recap

- A WAN links far-apart sites. You trade cost, speed, delay, privacy and reliability.
- Dedicated links (leased line, Metro Ethernet, MPLS) give SLAs. Shared links (DSL, cable, PON, mobile) give low cost.
- MPLS separates traffic but does not encrypt it. CE, PE and P are the three router roles.
- PPP gives authentication and address assignment. PPPoE puts it on Ethernet and lowers the MTU to 1492.
- GRE adds 24 bytes, carries multicast and does not encrypt, so pair it with IPsec.
- DMVPN combines mGRE, NHRP, a routing protocol and IPsec. Phase 3 gives direct spoke tunnels with hub summarisation.
- Full mesh needs n(n - 1) / 2 links, while hub and spoke needs n - 1. Choose based on the traffic pattern.
