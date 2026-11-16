# Mobile, Satellite and IoT Networks

**Level:** Intermediate | **Prerequisites:** [Bandwidth, Latency and Performance](../01-foundations/06-network-performance-basics.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [Wi-Fi Basics](01-wifi-basics.md)

## In One Line

A mobile network is a radio access part plus an IP core, joined by tunnels, so your IP address survives while you move. Satellite and IoT networks solve different problems: reaching places with no cables, and connecting small devices that have little power.

## Why It Matters

Wi-Fi only works close to an access point. People and machines need a connection on a train, in a field, and at sea. Mobile, satellite and IoT radio networks fill those gaps. Each one makes a different trade between range, speed, power use and cost.

Interviewers ask about this because:

- Backend and DevOps engineers meet mobile clients every day. Those clients sit behind CGNAT, change networks, and have changing latency.
- A network engineer may need to run a cellular backup link, a private APN, or a satellite site.
- Questions like "why can I not port forward on my phone connection?" and "why is GEO satellite slow?" test real understanding.
- IoT design questions test whether you can pick the right radio for a job.

## Simple Explanation

Think of a mobile network as a large courier company.

- You are the customer. In mobile language your phone is the **UE** (User Equipment).
- Your **SIM** card is your membership card. It proves who you are.
- The nearest **base station** is the local branch office. It only handles the last stretch to you.
- The **core network** is the head office. It checks your membership, gives you your address, and connects you to the outside world.
- Between branch and head office, your parcels travel in a sealed bag. This is a **tunnel**.
- When you travel to another town, a new branch takes over. Head office just sends the sealed bag to the new branch. Your address at head office stays the same.

That last point is the main idea. Your IP address lives at the core, not at the tower. So you can move between towers and your video call keeps running.

Where the analogy stops: a courier branch handles parcels one at a time, but a base station shares radio capacity among many users, and it makes new sharing decisions about every millisecond. A courier also has no idea about signal strength, which controls almost everything in radio.

The small radios are easier to picture as ways of talking:

| Technology | Like | Why |
|---|---|---|
| NFC | Passing a note by hand | Works only when almost touching |
| Bluetooth | Talking across a table | Short range, personal devices |
| Zigbee | Passing notes through neighbours | Each device relays for others (mesh) |
| LoRaWAN | Shouting slowly across a valley | Very far, very few words |
| NB-IoT | Sending a postcard by national post | Uses an operator's paid network |

Where these stop being true: radios do not "try harder" by being louder alone. They gain range by sending more slowly, by repeating, and by using lower frequencies.

## How It Works

### Key radio words

- **Frequency band:** the part of the radio spectrum in use. Lower frequencies travel further and pass through walls better. Higher frequencies carry more data but reach less far.
- **Licensed spectrum:** frequencies an operator pays the government to use alone. **Unlicensed spectrum** (such as 2.4 GHz) is shared by anyone who follows the rules.
- **Uplink:** data from device to network. **Downlink:** data from network to device.
- **Circuit switching:** a fixed path is held open for the whole call, like the old phone network. **Packet switching:** data is cut into packets that share the links, like the internet.
- **3GPP** (3rd Generation Partnership Project): the group that writes mobile standards. Its work comes in numbered "Releases".

### From 2G to 5G

Each generation changed something real. The speeds below are typical user experience, not peak marketing numbers. They vary a lot by operator, band and signal.

| Gen | First service | Main technology | Traffic type | Typical speed | Typical RTT |
|---|---|---|---|---|---|
| 1G | Around 1980 | AMPS, NMT | Analog voice | Voice only | Not relevant |
| 2G | 1991 | GSM, IS-95 CDMA | Circuit, plus packet data | 10-200 kbit/s | 300-1000 ms |
| 3G | 2001 | UMTS, CDMA2000, HSPA | Circuit voice, packet data | 0.4-10 Mbit/s | 100-200 ms |
| 4G | 2009 | LTE, LTE-Advanced | Packet only | 10-100 Mbit/s | 30-60 ms |
| 5G | 2019 | NR (New Radio) | Packet only | 100-900 Mbit/s (mid-band) | 10-30 ms |

RTT means round-trip time: the time to go there and come back.

What changed each time:

- **2G** made voice digital and added SMS text messages. GPRS and EDGE later added slow packet data.
- **3G** made mobile data useful. Voice still used circuits.
- **4G** removed circuits completely. Everything is IP. Voice became an app-like service called **VoLTE** (Voice over LTE).
- **5G** added a new radio, much wider channels, and a redesigned core. It also added three service goals:
  - **eMBB** (enhanced Mobile Broadband): high speed for phones.
  - **URLLC** (Ultra-Reliable Low-Latency Communication): very reliable and fast, for robots and control systems.
  - **mMTC** (massive Machine-Type Communication): huge numbers of small devices.

Two technologies help explain the speed growth:

- **OFDMA** (Orthogonal Frequency Division Multiple Access): the channel is cut into many thin subcarriers. The base station gives each user small blocks of them. LTE downlink and 5G NR use it.
- **MIMO** (Multiple Input, Multiple Output): many antennas send separate data streams on the same frequency at once.

5G uses three kinds of spectrum:

| Band type | Range | Reach | Speed |
|---|---|---|---|
| Low-band | Below 1 GHz | Many km | Similar to good 4G |
| Mid-band | About 1-7 GHz (3.5 GHz is common) | 1-3 km or so | Hundreds of Mbit/s |
| mmWave | 24 GHz and above | Hundreds of metres, weak through walls | Can pass 1 Gbit/s |

3GPP names these **FR1** (410 MHz to 7.125 GHz) and **FR2** (above 24.25 GHz).

The fuller truth about targets: the official 5G goals (IMT-2020, from the ITU) include 20 Gbit/s peak downlink and 1 ms latency for URLLC. These are design targets for ideal conditions. Few users will ever see them.

Many operators have already switched off 3G, and some have switched off 2G. **5G-Advanced** started with Release 18 (finished in 2024). **6G** is in early study, and operators and vendors talk about first service around 2030.

### The three parts of any mobile network

```text
+------+  radio  +-----------------+  backhaul  +---------+
|  UE  |<------->|       RAN       |<---------->|  Core   |--> Internet
| phone|  (Uu)   | towers/antennas | (fibre or  | network |
+------+         +-----------------+ microwave) +---------+
```

- **UE** (User Equipment): the phone, modem or sensor, plus its SIM.
- **RAN** (Radio Access Network): the towers and radios. It handles the air link and hands traffic to the core.
- **Core:** the central systems. They do authentication, IP address assignment, mobility tracking, billing and policy, and they connect to the internet.

Base stations have different names in each generation:

| Gen | Base station name | Controller above it |
|---|---|---|
| 2G | BTS (Base Transceiver Station) | BSC (Base Station Controller) |
| 3G | NodeB | RNC (Radio Network Controller) |
| 4G | eNodeB or eNB (evolved NodeB) | None, the eNB does the job itself |
| 5G | gNodeB or gNB (next generation NodeB) | None, but it can split into parts |

### 4G LTE architecture

The full 4G system is called **EPS** (Evolved Packet System). Its core is called **EPC** (Evolved Packet Core).

```text
                        +-----+
                        | HSS |  subscriber database
                        +--+--+
                           | S6a (Diameter)
                        +--+--+
              S1-MME    | MME |  control: login, mobility
           +----------->|     |
           |            +--+--+
           |               | S11
+----+  +--+--+  S1-U   +--+--+  S5/S8  +-----+  SGi
| UE |--| eNB |---------| SGW |---------| PGW |------- Internet
+----+  +-----+ GTP-U   +-----+  GTP    +--+--+
    Uu                                     | Gx
                                        +--+---+
                                        | PCRF |  policy, charging
                                        +------+
```

| Part | Full name | Job |
|---|---|---|
| eNodeB | Evolved NodeB | Radio, scheduling, handover decisions |
| MME | Mobility Management Entity | Control: attach, security, paging, tracking |
| HSS | Home Subscriber Server | Database of subscribers and keys |
| SGW | Serving Gateway | Local user-data anchor as you move between towers |
| PGW | Packet Data Network Gateway | Gives your IP, connects to the internet, enforces policy |
| PCRF | Policy and Charging Rules Function | Decides QoS and charging rules |

Two separate paths run through this network:

- **Control plane:** signalling, such as "this user logged in". It goes through the MME.
- **User plane:** your real data. It goes eNB, SGW, PGW. The MME never touches it.

A **bearer** is a logical pipe through the network with a set QoS level. Your phone always gets a **default bearer**. A VoLTE call adds a **dedicated bearer** with better priority. Each bearer has a **QCI** (QoS Class Identifier). For example, QCI 1 is used for conversational voice.

### How a phone joins a 4G network

1. The phone scans for cells and reads their broadcast information. It picks an allowed network.
2. It makes a short random-access request, then sets up an **RRC** (Radio Resource Control) connection with the eNB.
3. It sends an Attach Request to the MME, with its identity and sometimes an APN. This is a **NAS** (Non-Access Stratum) message: the radio just carries it to the core.
4. The MME gets authentication data from the HSS. The phone and network prove themselves to each other using a key in the SIM. This is called **AKA** (Authentication and Key Agreement).
5. Encryption and integrity protection start.
6. The MME picks an SGW and PGW, using the APN and DNS.
7. The PGW gives the phone an IP address (IPv4, IPv6 or both).
8. Tunnels are built for the default bearer. Data can flow.

### Tunnels: why your IP does not change

The network carries your packets inside **GTP** (GPRS Tunnelling Protocol). Each tunnel has a number called a **TEID** (Tunnel Endpoint Identifier).

```text
What the eNB/gNB sends toward the core:

+----------+-----+---------+------------+-----------------------+
| Outer IP | UDP | GTP-U   | Inner IP   | TCP/UDP + your data   |
| 20 bytes | 8   | 8 min   | phone IP   |                       |
+----------+-----+---------+------------+-----------------------+
  tower to    port  TEID     the packet the phone really sent
  gateway     2152
```

The GTP-U header (user-plane GTP) looks like this:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-----+-+-+-+-+-+---------------+-------------------------------+
| Ver |P|R|E|S|N| Message Type  |            Length             |
+-----+-+-+-+-+-+---------------+-------------------------------+
|                 Tunnel Endpoint ID (TEID)                     |
+---------------------------------------------------------------+
```

| Field | Bits | Meaning |
|---|---|---|
| Ver | 3 | Version, value 1 for GTP-U |
| P | 1 | Protocol type, 1 means GTP |
| R | 1 | Reserved |
| E, S, N | 3 | Optional extension header, sequence number, N-PDU number |
| Message Type | 8 | 255 means G-PDU, which is carried user data |
| Length | 16 | Bytes that follow the first 8 bytes |
| TEID | 32 | Which tunnel this packet belongs to |

**Handover** is when a moving phone passes from one tower to the next:

1. The phone reports the signal of nearby cells.
2. The old tower asks the new tower to prepare.
3. The phone switches radio to the new tower.
4. The tunnel to the gateway is updated, so it now ends at the new tower.
5. The gateway keeps the same IP for the phone.

Internet servers see no change, because the phone's address is anchored at the PGW (4G) or UPF (5G). Inside the operator, only the tunnel's far end moved.

### 5G architecture

The 5G radio network is **NG-RAN**. The 5G core is **5GC**. The big change is that the control plane is now a set of small services that talk over a shared bus, like microservices. This is **SBA** (Service-Based Architecture). They use HTTP/2 with JSON.

```text
 +------+ +------+ +-----+ +-----+ +------+
 | NSSF | | NRF  | | PCF | | UDM | | AUSF |
 +--+---+ +--+---+ +--+--+ +--+--+ +--+---+
    |        |        |       |       |
 ===+==+=====+========+==+====+=======+===  service bus (HTTP/2)
       |                 |
    +--+--+           +--+--+
    | AMF |           | SMF |
    +--+--+           +--+--+
       | N2 (and          | N4 (PFCP)
       | N1 from UE)      |
+----+ +--+--+  N3     +--+--+   N6
| UE |-| gNB |---------| UPF |--------- Internet
+----+ +-----+  GTP-U  +-----+
```

| 5G part | Full name | Job | Closest 4G part |
|---|---|---|---|
| gNodeB | Next generation NodeB | Radio | eNodeB |
| AMF | Access and Mobility Management Function | Registration, mobility | MME (part) |
| SMF | Session Management Function | Creates sessions, gives IPs, controls UPF | MME (part), control side of SGW and PGW |
| UPF | User Plane Function | Forwards user data | User side of SGW and PGW |
| UDM | Unified Data Management | Subscriber data | HSS |
| AUSF | Authentication Server Function | Authentication | HSS (part) |
| PCF | Policy Control Function | Policy | PCRF |
| NRF | Network Repository Function | Service discovery, like DNS for core services | New in 5G |
| NSSF | Network Slice Selection Function | Picks the slice | New in 5G |

What changed from 4G:

- 4G "Attach" becomes 5G **Registration**, followed by a **PDU Session** setup. A PDU Session is the data connection, and it replaces the PDN connection.
- The 4G bearer becomes the 5G **QoS Flow**. Each flow has a **QFI** (QoS Flow Identifier) and a **5QI** (5G QoS Identifier).
- The APN is now called **DNN** (Data Network Name). It does the same job.
- The UPF can be placed close to users, which cuts latency.
- The user plane still uses GTP-U. Only the control plane moved to HTTP/2.

**Non-Standalone (NSA) vs Standalone (SA):**

- **NSA:** 5G radio is added to a 4G network. The phone keeps a 4G anchor for control, and the core is still EPC. Early 5G launches used this. You get more speed, but no slicing or other 5G core features.
- **SA:** the phone uses only 5G radio and 5GC. This gives the full feature set.

So a "5G" icon on a phone does not tell you which one you have.

### APN

**APN** (Access Point Name) is a name the phone gives the network to say which gateway and which outside network it wants. Think of it as a door name at head office: "the internet door", "the voice door", or "the door to company X".

The APN decides:

- Which PGW or UPF serves you.
- Which IP pool you get (public, CGNAT or private).
- What type of address you get: IPv4, IPv6 or IPv4v6.
- What charging rules and QoS apply.
- Which network you reach: internet, the operator's voice system, or a company.

Common examples:

| APN | Used for |
|---|---|
| `internet` | Normal mobile data |
| `ims` | VoLTE and VoNR voice, through the IMS system |
| `mms` | Picture messages on some operators |
| `corp.example.com` | A private APN for one company |

The fuller truth about the name: an APN has a **Network Identifier** (such as `internet`) and an **Operator Identifier** (such as `mnc001.mcc001.gprs`). The MME builds a DNS name and looks it up inside the operator network to find a gateway:

```text
internet.apn.epc.mnc001.mcc001.3gppnetwork.org
```

**MCC** is the Mobile Country Code and **MNC** is the Mobile Network Code. Together they name an operator. MCC 001 with MNC 01 is reserved for test networks.

A **private APN** gives an enterprise device an address from the company's own range, with no internet path. Traffic goes from the gateway to the company over a VPN or leased line. This is common for payment terminals and remote sensors.

### CGNAT on mobile

**CGNAT** (Carrier-Grade NAT) is NAT run by the operator, shared by many customers. It exists because there are not enough public IPv4 addresses for every phone. NAT basics live in [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

```text
+-------+  100.72.14.9                     203.0.113.20
| Phone |------[ tower ]---[ PGW/UPF ]---[ CGNAT ]------ Internet
+-------+  shared space    tunnel ends     public side
```

- The phone often gets an address from **100.64.0.0/10**, the Shared Address Space from RFC 6598. It is neither public nor RFC 1918 private. Some operators use 10.0.0.0/8 instead.
- CGNAT usually sits at or just after the PGW or UPF.
- One public IP may be shared by many phones, using different ports.

What this breaks or changes:

- **No inbound connections.** The internet cannot start a connection to your phone. Port forwarding does not work, since you do not control the CGNAT box.
- **Shared reputation.** If one user on your IP misbehaves, a website may block or rate-limit all of them.
- **Idle timeouts.** Silent connections get dropped, so apps send keepalives, and these use battery.
- **Logging.** To trace abuse, the operator must record who had which IP and port at what time.

The long-term fix is IPv6. Many operators give each phone a **/64** prefix through Router Advertisement. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md) for how that works.

Some operators, such as T-Mobile US, went further and run IPv6-only for phones. They use **464XLAT** (RFC 6877) so that old IPv4-only apps still work:

```text
  Inside the phone                 Operator network
+-----------+  IPv4  +------+  IPv6  +------+  IPv4  +--------+
| IPv4 app  |------->| CLAT |------->| PLAT |------->| server |
| 192.0.0.4 |        +------+        +------+        +--------+
+-----------+      4 to 6 here     6 to 4 here, public IPv4 added
```

**CLAT** is the small translator on the phone. **PLAT** is a NAT64 box in the network.

### Bluetooth

**Bluetooth** is short-range radio in the 2.4 GHz unlicensed band, run by the Bluetooth SIG (Special Interest Group). There are two main types:

| | Bluetooth Classic (BR/EDR) | Bluetooth Low Energy (BLE) |
|---|---|---|
| Used for | Audio streams, older accessories | Sensors, wearables, beacons, smart locks |
| Channels | 79, each 1 MHz | 40, each 2 MHz |
| Speed | 1-3 Mbit/s | 1 or 2 Mbit/s, or slower coded modes for range |
| Power | Higher | Very low, sleeps most of the time |

Key ideas:

- **Frequency hopping:** Classic changes channel 1600 times per second. This avoids interference, including from Wi-Fi.
- **Piconet:** one central device with up to 7 active peripherals. A device in two piconets forms a **scatternet**.
- **BLE roles:** a device **advertises** on 3 special channels. A central connects. Data is organised with **GATT** (Generic Attribute Profile), a list of named values such as "heart rate".
- **Pairing** creates shared keys. LE Secure Connections (Bluetooth 4.2) uses ECDH key exchange. The simplest method, "Just Works", has no protection against a man-in-the-middle.
- **Bluetooth Mesh** lets BLE devices relay for each other, for example in building lighting.
- Bluetooth 6.0 (2024) added **Channel Sounding** for measuring distance.

Typical range is about 10 m indoors. The fuller truth: BLE coded mode can reach much further in open air, and the result depends on the antennas and transmit power.

### NFC

**NFC** (Near Field Communication) works at **13.56 MHz** with a practical range of about 4 cm. Speeds are low: 106, 212 or 424 kbit/s. Its short range is a feature, because touching a card to a reader acts as a clear "yes".

NFC works by magnetic coupling, not by radio waves that travel far. The reader creates a magnetic field, and a **passive tag** gets its power from that field. It needs no battery.

Three modes:

1. **Reader/writer:** a phone reads a tag on a poster.
2. **Card emulation:** a phone acts as a bank card or transit pass.
3. **Peer-to-peer:** two devices exchange small data.

NFC grew from **RFID** (Radio Frequency Identification), and shares standards with contactless cards such as ISO/IEC 14443. Tag data is commonly stored in **NDEF** (NFC Data Exchange Format). Phone payments use a token in place of your card number, plus a one-time code for each payment, so your real card number is not sent.

### Zigbee

**Zigbee** is a low-power mesh for homes and buildings. It is built on **IEEE 802.15.4**, which defines the radio and MAC layers. It usually runs at 2.4 GHz with 16 channels (11 to 26) and **250 kbit/s**.

```text
                   +-------------+
       +-----------| Coordinator |---------+
       |           +-------------+         |
  +----+---+                          +----+---+
  | Router |--------------------------| Router |
  +--+--+--+                          +---+----+
     |  |                                 |
 +---+  +----+                        +---+----+
 |End |  |End |                       |  End   |
 |dev.|  |dev.|                       | device |
 +----+  +----+                       +--------+
```

- **Coordinator:** starts the network. There is one per network.
- **Router:** relays for others. Usually mains-powered, such as a smart bulb.
- **End device:** usually a battery sensor. It sleeps and talks only to its parent.

Each hop reaches perhaps 10-20 m indoors, and mesh extends the total reach. Zigbee uses AES-128 encryption.

Zigbee channels 15, 20 and 25 fall in the gaps between Wi-Fi channels 1, 6 and 11, so they are good choices near busy Wi-Fi.

Related names: the Zigbee Alliance became the **Connectivity Standards Alliance** (CSA) in 2021. **Thread** also uses 802.15.4 but carries real IPv6. **Matter** is an application standard that runs over Wi-Fi, Ethernet and Thread. **Z-Wave** is a sub-GHz home mesh rival.

### LoRaWAN

Two words are often mixed up:

- **LoRa** is the physical radio method, using **chirp spread spectrum**. A chirp is a signal that sweeps across frequencies, and it can be decoded even when very weak.
- **LoRaWAN** is the network protocol on top, run by the LoRa Alliance.

It runs in unlicensed sub-GHz bands, such as EU868 (863-870 MHz) and US915 (902-928 MHz). Common quoted range is 2-5 km in cities and about 15 km or more in open country. Rates are about 0.3-50 kbit/s, and payloads are small, around 51 to 222 bytes depending on data rate in EU868.

```text
Sensor ---radio---> Gateway ---IP---> Network   ---> Application
Sensor ---radio---> Gateway ---IP---> Server         Server
```

This shape is called **star of stars**. A sensor does not join one gateway. Any gateway that hears it forwards the message, and the network server removes duplicates.

**Spreading Factor** (SF) runs from SF7 to SF12. A higher SF means longer range but much more airtime. **ADR** (Adaptive Data Rate) lets the network choose the fastest setting that still works.

Device classes:

| Class | Listens for downlink | Battery |
|---|---|---|
| A | Only two short windows after each uplink | Best |
| B | Also at scheduled times | Medium |
| C | Almost always | Worst, usually mains-powered |

Devices join with **OTAA** (Over-The-Air Activation, the better choice, using fresh session keys) or **ABP** (Activation By Personalization, using fixed keys). Two AES-128 layers protect it: one for the network and one for the application, so the network operator does not need to read your data.

### NB-IoT and LTE-M

**NB-IoT** (Narrowband IoT) comes from 3GPP Release 13. It is cellular for small devices and uses an operator's **licensed spectrum**.

- Carrier width is only 180 kHz.
- It can run inside an LTE carrier (in-band), in an LTE guard band, or standalone.
- Rates are tens of kbit/s for the first version. Later Cat-NB2 reaches roughly 127 kbit/s down and 159 kbit/s up.
- It reaches deep indoors and underground by repeating messages.
- It has a design goal of about 10 years of battery life.

Two sleep tools save the battery:

- **PSM** (Power Saving Mode): the device goes fully unreachable for hours or days, but stays registered.
- **eDRX** (extended Discontinuous Reception): the device checks for paging only every so often, not every few seconds.

**LTE-M** (Cat-M1) is its bigger sibling. It offers up to about 1 Mbit/s, moves between towers, and can carry voice. Trackers and wearables often use it.

**RedCap** (Reduced Capability, Release 17) is a simpler 5G class for mid-level devices such as cameras and industrial sensors. The fuller truth about 5G and IoT: NB-IoT and LTE-M are accepted as meeting 5G low-power goals, so they continue alongside 5G.

Devices on these links often use small protocols, such as **MQTT** or **CoAP**. See [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

### Comparing the IoT radios

Numbers are typical ballpark values.

| Tech | Band | Range | Speed | Power | Topology | Paid network? |
|---|---|---|---|---|---|---|
| NFC | 13.56 MHz | About 4 cm | Up to 424 kbit/s | Tag needs none | Point to point | No |
| BLE | 2.4 GHz | About 10 m+ | 1-2 Mbit/s | Very low | Star, mesh | No |
| Zigbee | 2.4 GHz (mostly) | 10-20 m per hop | 250 kbit/s | Low | Mesh | No |
| Wi-Fi | 2.4, 5, 6 GHz | 30 m or so | Very high | High | Star | No |
| LoRaWAN | Sub-GHz unlicensed | 2-15 km | 0.3-50 kbit/s | Very low | Star of stars | Optional |
| NB-IoT | Licensed cellular | Cell range, deep indoor | Tens of kbit/s | Very low | Star | Yes |
| LTE-M | Licensed cellular | Cell range | Up to 1 Mbit/s | Low | Star | Yes |

### Satellite: GEO, MEO and LEO

A satellite link sends your data to space and back down. Height decides the delay, because radio travels at about 300,000 km/s and no faster.

| Orbit | Height | Minimum possible RTT | Real RTT | Examples |
|---|---|---|---|---|
| LEO (Low Earth Orbit) | About 300-2000 km | 7 ms at 550 km | 25-60 ms | Starlink, Amazon Leo, Iridium, OneWeb |
| MEO (Medium Earth Orbit) | 2,000-35,786 km | 108 ms at 8,062 km | About 150 ms | SES O3b mPOWER |
| GEO (Geostationary Orbit) | 35,786 km | 477 ms | 550-700 ms | Viasat, Hughes |

```text
            GEO satellite (35,786 km), fixed over one point
                  *
                 / \
                /   \       one trip up + down = about 239 ms
               /     \      request + reply = about 477 ms
              /       \
             /         \
            /  LEO * --- * LEO (about 550 km), moving fast
           /    /         \
    +-----+----+-----------+---------+
    |  you     gateway               |  Earth
    +--------------------------------+
```

**GEO:** at 35,786 km, a satellite turns exactly with the Earth. It seems to stay still, so a simple fixed dish works. Three satellites can cover most of the planet, except the poles. The cost is high latency.

**LEO:** low satellites are close, so latency is low. But they cross the sky in minutes, and each one covers a small area. So you need thousands of them, and the user terminal must track and switch between them. Many use electronically steered flat antennas. Some, such as Starlink, pass traffic between satellites by laser.

Rain also weakens signals in the higher bands used by satellite (Ku and Ka). This is called **rain fade**.

Satellite can also reach normal phones, called **direct-to-device**. Examples include Apple's emergency SOS through Globalstar and T-Mobile with Starlink. 3GPP added support for such **NTN** (Non-Terrestrial Networks) in Release 17.

## Example

### Example 1: One packet from a 5G phone to a web server

Setup:

| Item | Value |
|---|---|
| Phone IP (from DNN `internet`) | 100.72.14.9 |
| gNB transport IP | 10.20.1.5 |
| UPF transport IP | 10.30.1.8 |
| Uplink TEID | 0x1A2B3C4D |
| CGNAT public IP | 203.0.113.20 |
| Web server | 198.51.100.80 port 443 |

Follow the packet:

```text
Step 1  Phone to gNB, over radio (encrypted on the air)
        src 100.72.14.9:51514   dst 198.51.100.80:443

Step 2  gNB to UPF, inside a GTP-U tunnel (N3)
        outer src 10.20.1.5     outer dst 10.30.1.8
        UDP dst 2152            TEID 0x1A2B3C4D
        inner src 100.72.14.9:51514   dst 198.51.100.80:443

Step 3  UPF removes the tunnel, sends plain IP out N6 to CGNAT
        src 100.72.14.9:51514   dst 198.51.100.80:443

Step 4  CGNAT to internet
        src 203.0.113.20:40112  dst 198.51.100.80:443

Step 5  Reply reverses: CGNAT maps 40112 back to the phone,
        UPF finds the phone's tunnel, gNB sends it over radio.
```

If the phone moves to a new gNB at 10.20.2.7, only Step 2's outer source changes. The server never knows.

### Example 2: Shared public IP and port blocks

Many CGNAT systems give each user a block of ports, so they log one line per block, not one line per connection.

```text
Public IP 203.0.113.20. Usable ports 1024-65535 = 64,512 ports.
Block size 1024 ports. Users on this IP = 64512 / 1024 = 63.

2026-10-30 09:14:07 ALLOC 100.72.14.9  203.0.113.20 40000-41023
2026-10-30 09:14:09 ALLOC 100.72.14.25 203.0.113.20 41024-42047
```

| Inside | Public | Destination |
|---|---|---|
| 100.72.14.9:51514 | 203.0.113.20:40112 | 198.51.100.80:443 |
| 100.72.14.9:51520 | 203.0.113.20:40113 | 198.51.100.80:443 |
| 100.72.14.25:49200 | 203.0.113.20:41030 | 198.51.100.80:443 |

The web server sees three connections from one IP. A server log must keep the source port as well as the IP, or the operator cannot tell these users apart.

### Example 3: Am I behind CGNAT?

Linux with a USB cellular modem, using ModemManager:

```bash
mmcli -L
mmcli -m 0 --simple-connect="apn=internet,ip-type=ipv4v6"
mmcli -b 1
```

```text
  Status             |      connected: yes
                     |      interface: wwan0
  Properties         |            apn: internet
                     |        ip type: ipv4v6
  IPv4 configuration |         method: static
                     |        address: 100.72.14.9
                     |         prefix: 30
                     |        gateway: 100.72.14.10
                     |            dns: 198.51.100.53
                     |            mtu: 1430
```

Now compare your interface address with what the internet sees. Replace `ip.example.com` with any "what is my IP" service.

```bash
# Linux
ip -4 addr show wwan0
curl -4 -s https://ip.example.com
```

```text
4: wwan0: <POINTOPOINT,UP,LOWER_UP> mtu 1430 qdisc fq_codel state UP
    inet 100.72.14.9/30 brd 100.72.14.11 scope global wwan0
203.0.113.20
```

```powershell
# Windows, built-in cellular adapter
netsh mbn show interfaces
Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias Cellular |
  Select-Object IPAddress, PrefixLength
(Invoke-RestMethod -Uri https://ip.example.com).Trim()
```

```text
IPAddress    PrefixLength
---------    ------------
100.72.14.9            30

203.0.113.20
```

The two addresses differ, and the local one is in 100.64.0.0/10. So this link is behind CGNAT. Running a server at home on this link will not work, unless you use IPv6 with inbound allowed, or an outbound tunnel to a public host.

Cisco IOS XE, for a cellular backup link on a router. Slot numbers and exact syntax vary by platform and software version, so check your model's guide:

```text
! Exec mode: create profile 1 with the APN
Router# cellular 0/2/0 lte profile create 1 internet

! Global config
interface Cellular0/2/0
 ip address negotiated
 ip nat outside
 dialer in-band
 dialer idle-timeout 0
 dialer-group 1
 pulse-time 1
!
dialer-list 1 protocol ip permit
ip route 0.0.0.0 0.0.0.0 Cellular0/2/0 200

! Check it
Router# show cellular 0/2/0 radio
Router# show cellular 0/2/0 profile
```

The `200` makes this a floating static route, used only when the main route fails.

### Example 4: Why a web page feels slow on GEO

Compare ping to the same server, 198.51.100.80:

```text
Link      Typical ping
GEO       612 ms
LEO        38 ms
4G         45 ms
5G         22 ms
```

Opening a new HTTPS site needs about 4 round trips before the first data arrives: DNS (1), TCP (1), TLS 1.3 (1), HTTP request (1).

```text
GEO: 4 x 612 ms = about 2.4 seconds
LEO: 4 x  38 ms = about 0.15 seconds
```

Now check the **bandwidth-delay product**: the amount of data "in flight" that TCP must keep sent but not yet acknowledged.

```text
50 Mbit/s x 0.612 s = 30.6 Mbit = about 3.8 MB
```

So the TCP window must grow to about 3.8 MB to fill the link. Starting from a small window, one round trip at a time, that takes several seconds on GEO. See [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).

### Example 5: LoRaWAN airtime and the duty cycle limit

A sensor in EU868 sends 12 bytes of application data. With 13 bytes of LoRaWAN overhead this is 25 bytes on the air, at 125 kHz bandwidth and coding rate 4/5. The standard LoRa airtime formula gives:

```text
SF7  : about  62 ms on air
SF12 : about 1.48 s on air (about 24 times longer)
```

Many EU868 sub-bands allow a 1 percent duty cycle. So after sending, a device must stay quiet for 99 times its airtime:

```text
SF7  : 0.062 s x 99 = about 6 seconds
SF12 : 1.48  s x 99 = about 147 seconds (nearly 2.5 minutes)
```

This is why LoRaWAN suits a soil sensor that reports each hour, not a video camera.

### Example 6: Pick the radio

| Need | Pick | Why |
|---|---|---|
| Water meter in a basement, 10 year battery | NB-IoT | Deep coverage, low power, operator network |
| Soil sensors across a large farm, no cellular | LoRaWAN | Long range, own gateway, unlicensed |
| Smart bulbs and door sensors in a house | Zigbee or Thread | Low-power mesh |
| Fitness band to phone | BLE | Very low power, every phone has it |
| Tap to pay on a bus | NFC | Touch range acts as consent |
| Tracker on a moving truck | LTE-M | Handover between towers, moderate data |
| Ship in mid-ocean | LEO or GEO satellite | No land network exists |

## Going Deeper

### Control and User Plane Separation

**CUPS** (Control and User Plane Separation, Release 14) split the SGW and PGW into control and user parts. 5G took the same idea further with SMF and UPF. They talk with **PFCP** (Packet Forwarding Control Protocol). This matters because you can run control centrally and put many user-plane nodes near customers.

### Protocols and ports in the core

| Purpose | 4G | 5G | Protocol |
|---|---|---|---|
| RAN to control | S1-MME | N2 | S1AP (SCTP 36412), NGAP (SCTP 38412) |
| RAN to user plane | S1-U | N3 | GTP-U (UDP 2152) |
| RAN to RAN | X2 | Xn | X2AP (SCTP 36422), XnAP (SCTP 38422) |
| Control to gateway | S11 | N4 | GTPv2-C (UDP 2123), PFCP (UDP 8805) |
| Gateway to gateway | S5/S8 | N9 | GTP |
| Subscriber data | S6a | Service bus | Diameter (3868), HTTP/2 |
| To outside networks | SGi | N6 | Plain IP |

**SCTP** (Stream Control Transmission Protocol) is a reliable transport like TCP, with multi-streaming and multi-homing.

### Splitting the gNodeB

A gNB can be split into three parts:

```text
+----+  fronthaul   +----+  midhaul   +----+  backhaul   +------+
| RU |--------------| DU |------------| CU |-------------| Core |
+----+  (often      +----+            +----+             +------+
 radio   eCPRI)      fast              slower
                     functions         functions
```

- **RU** (Radio Unit): antennas and radio hardware, at the mast.
- **DU** (Distributed Unit): time-critical work, such as scheduling.
- **CU** (Central Unit): less time-critical work.

**Open RAN** aims to make these parts work across vendors through open interfaces. Fronthaul has tight latency limits, usually needing fibre.

### Radio details that show up in interviews

- **FDD and TDD:** Frequency Division Duplex uses separate frequencies for up and down. Time Division Duplex shares one frequency, taking turns. Most mid-band 5G uses TDD.
- **Numerology:** 5G can use subcarrier spacing of 15, 30, 60 or 120 kHz and more. A wider spacing means shorter slots: 1 ms at 15 kHz, 0.5 ms at 30 kHz, 0.125 ms at 120 kHz. This is one reason 5G latency can be lower.
- **RRC states:** a phone is Idle or Connected in 4G. 5G adds **RRC Inactive**, which keeps context so that waking is quick. This saves battery and signalling.
- **Paging:** in Idle state the network knows only your rough area. To reach you, it broadcasts a page across that area.
- **Why tower speeds differ:** the base station's scheduler shares blocks among users. Good-signal users get faster modulation. So your speed depends on signal and on how many others are active.

### MTU on mobile

GTP-U adds at least 36 bytes in the transport network: outer IPv4 (20), UDP (8), GTP-U (8). Operators deal with this in two ways: raise MTU in the transport network, or give the UE a lower MTU, such as 1400-1430. Values vary by operator. A larger outer IPv6 header or IPsec on backhaul adds more. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Identity and security

- **IMSI** (International Mobile Subscriber Identity): the SIM identity, up to 15 digits: MCC, MNC, then a subscriber number.
- **IMEI** (International Mobile Equipment Identity): the device identity, not the user.
- **MSISDN:** your phone number. The network does not route data by it.
- **GUTI** (Globally Unique Temporary Identifier): a temporary ID, used to reduce exposure of the IMSI.
- **SUPI and SUCI** (5G): SUPI is the permanent ID. SUCI is the SUPI encrypted with the home network's public key. This blocks simple **IMSI catchers**, which are fake base stations that collect identities.
- **eSIM:** a SIM in software on a chip, updated remotely.

2G authenticated only the user and not the network. A fake tower can force a phone down to 2G, and this is called a **downgrade attack**. Newer phones can let you disable 2G. Old signalling systems between operators (SS7 and Diameter) also have known weaknesses.

### Roaming

- **Home-routed:** your traffic returns to your home operator's gateway, often over the shared **IPX** network between operators. Your public IP looks like home. This is the common default.
- **Local breakout:** traffic leaves through the visited country's network, giving lower latency.

5G adds **SEPP** (Security Edge Protection Proxy) at the edge of each operator to protect roaming signalling.

### Voice

- **CSFB** (Circuit-Switched Fallback): the phone drops to 2G or 3G for a call. This fails where those networks are gone.
- **VoLTE and VoNR:** voice as SIP and RTP over IP, using the **IMS** (IP Multimedia Subsystem).
- **VoWiFi:** the phone builds an IPsec tunnel over Wi-Fi to an **ePDG** (evolved Packet Data Gateway) in the operator network. See [IPsec](../09-security/08-ipsec.md).

### Slicing, edge and private networks

- **Network slicing:** several logical networks on the same 5G hardware. Each is named by an **S-NSSAI**, which holds a slice type (1 for eMBB, 2 for URLLC, 3 for massive IoT) and an optional differentiator. It needs a 5G core.
- **MEC** (Multi-access Edge Computing): servers close to the tower. A local UPF sends traffic straight to them.
- **Private 5G:** a company runs its own small cellular network, for example in a factory or port.
- **FWA** (Fixed Wireless Access): home broadband delivered over 4G or 5G.

### IPv6 detail for 464XLAT

- The CLAT address commonly used on phones is 192.0.0.4, from the 192.0.0.0/29 block set aside for this kind of use.
- The PLAT often uses the well-known NAT64 prefix **64:ff9b::/96** (RFC 6052), so 198.51.100.80 becomes `64:ff9b::c633:6450`.
- **DNS64** creates fake AAAA answers for IPv4-only names, so IPv6 apps can reach them through NAT64.
- NAT64 is RFC 6146. DNS64 is RFC 6147.

### Satellite and transport protocols

GEO links often use a **PEP** (Performance Enhancing Proxy, RFC 3135). It splits a TCP connection at each end of the satellite hop and acknowledges data locally, so the sender does not wait 600 ms.

Encryption of transport headers defeats this. [QUIC](../06-transport-layer/08-quic.md) and full-tunnel VPNs hide what a PEP needs to see. So these can be slower on GEO than plain TCP, unless the provider or the application tunes for it.

On LEO, the path changes as satellites move. This can cause small latency jumps and brief loss, which can look like congestion to some TCP algorithms. Starlink residential plans also commonly use CGNAT for IPv4, so Example 3 applies there too.

LEO with laser links has one surprising property: light in space is about 1.5 times faster than light in fibre. So over very long distances, LEO could in principle beat fibre, despite the trip up and down.

### IoT security

Weak default passwords on IoT devices fed the Mirai botnet in 2016. Good design uses a unique key per device, signed firmware updates, and a separate VLAN or private APN for devices. Radio encryption protects only one hop or layer. It does not replace securing the application.

## Common Mistakes

- **"5G always means gigabit speed."** Low-band 5G can feel like good 4G. Peak numbers need wide spectrum and ideal conditions.
- **"My phone shows 5G, so it uses a 5G core."** With NSA, control and core are still 4G.
- **"The tower gives my phone its IP."** The PGW (4G) or SMF with UPF (5G) does that. The tower is a radio relay with a tunnel.
- **"My IP changes at every tower."** Normal handover keeps the same IP, because the anchor is in the core. It can change if your session is rebuilt or you change network type or gateway.
- **"The APN is a password or a VPN."** It is a name that picks a gateway and policy. A private APN may use a VPN behind it, but the APN itself encrypts nothing.
- **"100.64.x.x is a private RFC 1918 address."** It is Shared Address Space from RFC 6598, intended for operators. It also is not routable on the public internet.
- **"5G replaced GTP."** Control moved to HTTP/2, but the user plane still uses GTP-U.
- **"Port forwarding on my 4G router will open my server."** With CGNAT, the operator holds the public side. Forwarding on your router does nothing for inbound internet traffic.
- **"LoRa and LoRaWAN are the same."** LoRa is the radio. LoRaWAN is the network protocol on top.
- **"NB-IoT is free like LoRaWAN."** NB-IoT uses licensed spectrum and usually a paid SIM plan. LoRaWAN uses unlicensed spectrum, though duty cycle rules apply.
- **"Zigbee and Bluetooth are the same thing."** Both can use 2.4 GHz, but they are different standards. Zigbee is built on 802.15.4 and designed around mesh.
- **"Satellite means slow."** GEO has high latency, but its bandwidth can be good. LEO has low latency and high bandwidth. Latency and bandwidth are different things.
- **"LEO has no delay because it is close."** Real RTT is 25-60 ms, because of ground stations, scheduling and the path to the server.

## Interview Questions

**Q: What are the main parts of a mobile network?**

A: Three parts. The UE is the phone or modem. The RAN is the radio network, with the eNodeB in 4G and gNodeB in 5G. The core does authentication, IP assignment, mobility and internet access. In 4G that core is the EPC, in 5G it is the 5GC.

**Q: What is the main difference between the 4G and 5G core?**

A: 4G uses fixed boxes such as MME, SGW and PGW, with Diameter and GTP signalling. 5G uses Service-Based Architecture, where functions like AMF and SMF talk over HTTP/2 APIs. The UPF handles user data and can be placed near users. The user plane still uses GTP-U. 5G also adds slicing and QoS Flows.

**Q: Why does my IP address stay the same when I move between towers?**

A: The IP is anchored at the PGW or UPF in the core, not at the tower. Packets travel between tower and gateway inside a GTP-U tunnel. On handover, only the tower end of the tunnel changes. So internet servers see no change.

**Q: What is an APN?**

A: The Access Point Name tells the network which gateway and outside network the device wants. It decides the IP pool, address type, QoS and charging. For example, `internet` for data and `ims` for voice. A private APN connects devices to a company network. In 5G it is called the DNN.

**Q: What is the difference between NSA and SA 5G?**

A: Non-Standalone adds 5G radio to a 4G network and keeps the 4G core, with LTE as the control anchor. Standalone uses 5G radio and the 5G core only. Slicing and other core features need SA.

**Q: What is CGNAT and how does it affect mobile users?**

A: CGNAT is operator-run NAT, where many customers share one public IPv4 address. The phone often gets an address from 100.64.0.0/10, from RFC 6598. Inbound connections and port forwarding do not work. Users share IP reputation, and the operator must log port mappings to trace abuse. IPv6 is the long-term fix.

**Q: How can an IPv6-only phone reach an IPv4-only server?**

A: With 464XLAT. A CLAT on the phone translates an IPv4 app's traffic into IPv6. A PLAT, which is NAT64 in the network, translates it back to IPv4 with a public address. For apps that use names, DNS64 can create synthetic AAAA records.

**Q: LoRaWAN or NB-IoT, how do you choose?**

A: LoRaWAN uses unlicensed spectrum, and you can run your own gateways. It suits private sites without cellular coverage, such as farms. NB-IoT uses an operator's licensed spectrum, so it has managed coverage and works deep indoors, but needs a SIM and plan. Both are low power and low data. The choice depends on coverage, cost and who runs the network.

**Q: How do Bluetooth Classic and BLE differ?**

A: Classic is for continuous streams, such as audio, with 79 channels and higher power use. BLE is for short bursts from low-power devices, with 40 channels, and sleeps most of the time. Both use 2.4 GHz and frequency hopping, but they are not directly compatible with each other.

**Q: What is NFC and why is short range useful?**

A: NFC runs at 13.56 MHz with a range of about 4 cm and speeds up to 424 kbit/s. A passive tag gets power from the reader's field. Short range means the user must tap on purpose, and that makes payments and pairing safer, though not immune to relay attacks.

**Q: Calculate the minimum GEO latency, and explain the LEO difference.**

A: GEO is 35,786 km up. One-way to a gateway is up and down, so about 72,000 km, which is about 239 ms at light speed. A request and reply doubles that to about 477 ms, and in real use it is 550 ms or more. LEO at 550 km gives about 7 ms minimum and 25-60 ms in practice. The cost of LEO is that you need thousands of moving satellites.

**Q: Why does TCP struggle on GEO, and how is that handled?**

A: High RTT means a large bandwidth-delay product, and slow start grows only once per RTT. So the window takes a long time to fill the link. Loss from weather can also cut the window. Providers use Performance Enhancing Proxies that split TCP at the satellite hop. These cannot help encrypted QUIC or VPN traffic in the same way.

## Quick Recap

- A mobile network is UE, RAN and core. Base stations are eNodeB in 4G and gNodeB in 5G.
- 4G core: MME, SGW, PGW, HSS, PCRF. 5G core: AMF, SMF, UPF, UDM, AUSF, PCF, with HTTP/2 services.
- GTP-U tunnels (UDP 2152) anchor your IP at the core, so handover keeps your connection alive.
- The APN (DNN in 5G) picks your gateway, IP pool and policy.
- Mobile IPv4 usually means CGNAT with 100.64.0.0/10 and no inbound access. IPv6 and 464XLAT are the way forward.
- Pick IoT radios by range, power, data and who runs the network: NFC, BLE, Zigbee, LoRaWAN, NB-IoT, LTE-M.
- GEO is at 35,786 km with RTT of at least 477 ms. LEO at about 550 km gives 25-60 ms, but needs thousands of satellites.
