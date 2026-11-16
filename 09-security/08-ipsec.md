# IPsec

**Level:** Advanced | **Prerequisites:** [VPNs](07-vpn.md), [Cryptography Basics](02-cryptography-basics.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md)

## In One Line

IPsec is a set of protocols that protects IP packets at the network layer: IKE agrees on keys, then ESP or AH protects each packet between two peers.

## Why It Matters

The internet does not protect your packets. Any router on the path can read them or change them. A company with two offices needs to send private data over this public network.

IPsec solves that problem at Layer 3, the IP layer. It gives four things:

- **Confidentiality:** nobody on the path can read the data. This is done with encryption.
- **Integrity:** nobody can change the data without being noticed.
- **Authentication:** each side proves who it is.
- **Anti-replay:** an attacker cannot record a packet and send it again later.

Because IPsec works at the IP layer, applications do not need to change. TCP, UDP, ICMP and routing traffic can all pass through it.

Interviewers ask about IPsec because it tests many skills at once: packet formats, cryptography, NAT, MTU and structured troubleshooting. A site-to-site tunnel is also a daily job for network, cloud and security engineers. "Phase 1 is up but Phase 2 is down, what do you check?" is a classic question.

## Simple Explanation

Imagine two company offices that must send secret papers through a public courier.

**Step 1: the managers meet.** Before sending anything, the two managers meet in a private room. They check each other's ID and agree on the rules: which lock to use and who holds which key. This is **IKE** (Internet Key Exchange).

**Step 2: they write agreements.** They write one agreement for papers going from Office A to Office B. They write a second agreement for papers going back. Each agreement is a **Security Association (SA)**. Each one has a number. That number is the **SPI**. It goes on every box, so the receiver knows which agreement and key to use.

**Step 3: the boxes travel.** Each box is locked and has a tamper seal. This is **ESP**. There is also a clear plastic bag with a tamper seal. Anyone can read what is inside, but any change is noticed. This is **AH**.

**Two ways to label a box:**

- **Transport mode:** keep the original address label and protect only the contents.
- **Tunnel mode:** put the whole envelope, label and all, inside a new box. The new label has only the two office addresses. Outsiders cannot see which employee sent it or who will receive it.

**Where the analogy stops being true:** a real meeting takes place once, but IKE stays active. It replaces keys on a timer and checks that the peer is alive. Also, the "managers" never send the key across the room. With Diffie-Hellman, both sides calculate the same secret without ever sending it.

## How It Works

### The Three Parts of IPsec

IPsec is a framework, not one protocol. It has a control part and a data part.

| Part | Job | How it travels |
|---|---|---|
| IKE | Authenticates peers, agrees on algorithms, creates keys | UDP 500, or UDP 4500 when NAT is found |
| ESP | Encrypts and authenticates data packets | IP protocol 50 |
| AH | Authenticates data packets, no encryption | IP protocol 51 |

```text
   Control plane (sets up the tunnel)
   R1 <==== IKE over UDP 500 or 4500 ====> R2
              (creates SAs and keys)

   Data plane (carries user traffic)
   R1 <==== ESP (IP protocol 50) =========> R2
              (uses the SAs and keys)
```

The core standards are RFC 4301 (architecture), RFC 4302 (AH), RFC 4303 (ESP) and RFC 7296 (IKEv2).

### Protocol Numbers Are Not Ports

This confuses many people.

Every IP header has a **Protocol** field. It says what comes next: 6 means TCP, 17 means UDP, 50 means ESP and 51 means AH.

ESP and AH sit directly on top of IP. They are not carried inside TCP or UDP, so they have **no port numbers**. "Port 50" is wrong. Say "IP protocol 50".

IKE is different. It is a normal UDP application, so it does use ports.

| Traffic | Identified by | Value |
|---|---|---|
| ESP | IP protocol | 50 |
| AH | IP protocol | 51 |
| IKE | UDP port | 500 |
| IKE after NAT is found, and ESP inside UDP | UDP port | 4500 |

Firewall rule for a tunnel: allow UDP 500, UDP 4500 and IP protocol 50 between the two peers. Add protocol 51 only if you really use AH.

### AH vs ESP

**AH (Authentication Header)** proves that the packet is genuine and was not changed. It does not hide anything.

**ESP (Encapsulating Security Payload)** encrypts the data. It normally also authenticates the data.

| Feature | AH | ESP |
|---|---|---|
| IP protocol | 51 | 50 |
| Encryption | No | Yes |
| Integrity and authentication | Yes | Yes |
| Anti-replay | Yes | Yes |
| Protects the outer IP header | Yes, the fields that do not change | No |
| Works through NAT | No | Yes, with NAT-T |
| Support in the standard | Optional (RFC 4301) | Required (RFC 4301) |
| Used today | Rarely | Almost always |

Why AH breaks with NAT: AH calculates its integrity value over the source and destination IP addresses. NAT changes the source address. The receiver then calculates a different value and drops the packet.

AH skips fields that routers must change, such as TTL, DSCP and the header checksum. It sets them to zero for the calculation. The addresses are not skipped.

**Fuller truth:** ESP can run with NULL encryption, so it gives integrity without secrecy. This lets firewalls inspect traffic and still works with NAT-T. It is one reason AH is rarely needed.

### AH Header

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Header  |  Payload Len  |           Reserved            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Security Parameters Index (SPI)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Integrity Check Value (ICV, variable size)         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The fixed part is 12 bytes. The **ICV (Integrity Check Value)** is a keyed checksum. Only someone with the key can produce the correct value.

### ESP Packet Format

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Security Parameters Index (SPI)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Initialization Vector (IV, size depends on cipher)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Payload Data (variable size)                |
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |             Padding (0-255 bytes)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |  Pad Length   |  Next Header  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Integrity Check Value (ICV, variable size)         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | Plain meaning |
|---|---|---|
| SPI | 4 bytes | Which SA this packet belongs to |
| Sequence Number | 4 bytes | Counter that stops replay |
| IV | 16 for AES-CBC, 8 for AES-GCM | Random or unique starting value, so the same data encrypts differently each time |
| Payload Data | Variable | The protected packet or segment |
| Padding | 0-255 bytes | Fills the data to the size the cipher needs |
| Pad Length | 1 byte | How many padding bytes were added |
| Next Header | 1 byte | What the payload is: 4 for an IPv4 packet, 41 for IPv6, 6 for TCP, 17 for UDP |
| ICV | Commonly 16 bytes | Integrity check value |

Only the SPI and Sequence Number are in clear text. Next Header sits inside the encrypted part, so an observer cannot see what kind of traffic is inside.

**Fuller truth:** RFC 4303 treats the IV as the start of the Payload Data field. It is drawn apart here because it adds to the overhead.

### Transport Mode vs Tunnel Mode

The mode decides **what** gets protected.

**Transport mode** keeps the original IP header and protects the payload. It is used when the two communicating hosts are also the two IPsec peers. Examples are L2TP/IPsec, GRE over IPsec, and host-to-host rules between servers.

**Tunnel mode** protects the whole original packet and adds a new outer IP header. It is the normal mode for site-to-site VPNs, because gateways protect traffic for other machines.

```text
Original packet
+--------+-----+------+
| IP hdr | TCP | Data |
+--------+-----+------+

ESP transport mode
+--------+-----+-----+------+---------+-----+
| IP hdr | ESP | TCP | Data | Trailer | ICV |
+--------+-----+-----+------+---------+-----+
                |<--- encrypted ----->|
         |<----- authenticated ------>|

ESP tunnel mode
+--------+-----+--------+-----+------+---------+-----+
| New IP | ESP | IP hdr | TCP | Data | Trailer | ICV |
+--------+-----+--------+-----+------+---------+-----+
                |<------ encrypted ---------->|
         |<--------- authenticated ---------->|

AH transport mode
+--------+----+-----+------+
| IP hdr | AH | TCP | Data |
+--------+----+-----+------+
|<----- authenticated ---->|  (except changing IP fields)

AH tunnel mode
+--------+----+--------+-----+------+
| New IP | AH | IP hdr | TCP | Data |
+--------+----+--------+-----+------+
|<--------- authenticated --------->|  (except changing IP fields)
```

| Point | Transport mode | Tunnel mode |
|---|---|---|
| Original IP header | Stays visible | Hidden inside (with ESP) |
| New IP header | No | Yes, 20 bytes for IPv4 |
| Typical use | Host to host, GRE or L2TP inside IPsec | Gateway to gateway, remote access |
| Hides real endpoints | No | Yes, with ESP |

### Security Association (SA)

A **Security Association** is a one-way agreement that says how to protect traffic. It holds:

- The protocol (ESP or AH) and mode.
- The algorithms and keys.
- The SPI.
- The sequence counter and replay window.
- The lifetime.
- The traffic it applies to, called **traffic selectors**.

**SAs are one-way.** A normal tunnel needs two IPsec SAs, one per direction. Each direction has its own keys, its own sequence counter and its own replay window.

A running tunnel between two peers therefore has this:

```text
                 1 IKE SA (two-way, control)
        R1 <=====================================> R2

        R1 ----- IPsec SA, SPI 0x4D7E91AA -------> R2
        R1 <---- IPsec SA, SPI 0xC8A1F2B3 -------- R2
```

### SPI (Security Parameters Index)

The **SPI** is a 32-bit number in every ESP and AH packet. It tells the receiver which SA to use.

Think of a hotel with many guests. The SPI is the room number on a key tag. It does not open anything alone, but it shows which record to look up.

Key facts:

- **The receiver chooses the SPI.** That way, the number is unique in the receiver's own table.
- When R1 sends to R2, it writes the SPI that R2 chose.
- Values 1-255 are reserved, and 0 is not used on the wire for an SA.
- The SPI is not secret. It is sent in clear text.

**Fuller truth:** older texts say an SA is found by the triple of destination IP, protocol and SPI. RFC 4301 allows a unicast SA to be found by SPI alone, and implementations may use more fields.

### What Happens to One Packet

**Sending (R1, tunnel mode, ESP):**

1. A packet arrives from the LAN.
2. R1 checks its policy. Does this traffic need protection?
3. If an SA exists, R1 uses it. If not, R1 starts IKE. The packet that triggered IKE is often dropped.
4. R1 adds padding, Pad Length and Next Header.
5. R1 encrypts the packet and trailer.
6. R1 adds the ESP header with the SPI and the next sequence number.
7. R1 calculates the ICV and adds it to the end.
8. R1 adds a new outer IP header with protocol 50 and routes the packet.

**Receiving (R2):**

1. R2 sees protocol 50 and finds the SA by SPI. An unknown SPI means a drop.
2. R2 checks the sequence number against the replay window.
3. R2 checks the ICV. A wrong ICV means a drop. This happens before decryption, so forged packets cost little work.
4. R2 decrypts the payload and removes padding and the ESP header.
5. R2 checks that the inner packet matches the SA's traffic selectors.
6. R2 forwards the inner packet to the LAN.

**Fuller truth:** with AES-GCM, integrity and decryption happen together in one operation. GCM is an **AEAD** cipher (Authenticated Encryption with Associated Data), which means one algorithm does both jobs.

### IKE: Building the Tunnel

IKE does four jobs: it authenticates the peers, agrees on algorithms, creates shared keys and maintains the SAs.

Some terms first:

- **Diffie-Hellman (DH):** a way for two sides to create a shared secret over a public network without sending the secret. See [Cryptography Basics](02-cryptography-basics.md).
- **DH group:** the size and type of the math used. Group 14 is 2048-bit MODP. Groups 19 and 20 are 256-bit and 384-bit elliptic curves.
- **Pre-shared key (PSK):** a password configured on both peers.
- **Nonce:** a random number used once. It makes each session's keys different.
- **Perfect Forward Secrecy (PFS):** new keys come from a fresh DH exchange, so stealing an old key does not reveal other sessions.

There are two versions. IKEv1 uses "phases". IKEv2 uses "exchanges". Engineers still say "Phase 1" and "Phase 2" when talking about IKEv2.

| Common name | IKEv1 term | IKEv2 term |
|---|---|---|
| Phase 1 | ISAKMP SA or IKE SA | IKE SA |
| Phase 2 | IPsec SA | Child SA |

### IKEv1 Phase 1: Main Mode

**Phase 1** builds a secure, two-way control channel. Main Mode uses 6 messages.

```text
R1 (initiator)                              R2 (responder)
   |  1. SA: my proposals                         |
   |--------------------------------------------->|
   |  2. SA: the proposal I chose                 |
   |<---------------------------------------------|
   |  3. KE (DH public value), nonce              |
   |--------------------------------------------->|
   |  4. KE (DH public value), nonce              |
   |<---------------------------------------------|
   |  5. Encrypted: ID, proof of identity         |
   |--------------------------------------------->|
   |  6. Encrypted: ID, proof of identity         |
   |<---------------------------------------------|
```

- **Messages 1-2:** negotiate the policy.
- **Messages 3-4:** run DH and exchange nonces. Both sides now hold keys.
- **Messages 5-6:** prove identity. These are encrypted, so identities stay hidden.

The Phase 1 values must match. The memory aid is **HAGLE**: Hash, Authentication, Group, Lifetime, Encryption.

**Fuller truth:** lifetime matching depends on the vendor. On Cisco IOS, the responder accepts a peer's lifetime if it is equal to or shorter than its own.

### IKEv1 Phase 1: Aggressive Mode

Aggressive Mode does the same job in 3 messages, but it gives up protection.

```text
R1 (initiator)                              R2 (responder)
   |  1. SA, KE, nonce, ID (in clear)             |
   |--------------------------------------------->|
   |  2. SA, KE, nonce, ID, proof (in clear)      |
   |<---------------------------------------------|
   |  3. Proof                                    |
   |--------------------------------------------->|
```

Two weaknesses:

- Identities are sent before encryption starts.
- With a PSK, an attacker can capture message 2 and guess the key offline.

Why did it exist? In Main Mode with a PSK, the responder must pick the key before it can read the encrypted ID. So it can only pick the key by the source IP. A peer with a changing IP broke this, and Aggressive Mode was the old workaround.

### IKEv1 Phase 2: Quick Mode

**Phase 2** creates the IPsec SAs that carry data. Quick Mode uses 3 messages, all protected by the Phase 1 SA.

```text
R1 (initiator)                              R2 (responder)
   |  1. Encrypted: SA, nonce, [KE], proxy IDs    |
   |--------------------------------------------->|
   |  2. Encrypted: SA, nonce, [KE], proxy IDs    |
   |<---------------------------------------------|
   |  3. Encrypted: hash (acknowledgement)        |
   |--------------------------------------------->|
```

[KE] appears only if PFS is on. **Proxy IDs** are the local and remote subnets to protect. They must mirror each other on the two peers.

Phase 2 agrees on:

- ESP or AH, the cipher and the integrity algorithm. Cisco calls this a **transform set**.
- The mode (tunnel or transport).
- The proxy IDs.
- The lifetime.
- The PFS group, if PFS is used.

Total IKEv1 messages: 9 with Main Mode, 6 with Aggressive Mode.

**Phase 1.5:** remote-access IKEv1 often added XAuth (user login) and Mode Config (assigning an IP to the client) between the phases. These were vendor-driven extensions and not part of the core standard.

### IKEv2 Exchanges

IKEv2 (RFC 7296) brings up a tunnel in **4 messages**.

```text
R1 (initiator)                              R2 (responder)
   |  IKE_SA_INIT request                         |
   |  SAi1, KEi, Ni, NAT detection                |
   |--------------------------------------------->|
   |  IKE_SA_INIT response                        |
   |  SAr1, KEr, Nr, NAT detection                |
   |<---------------------------------------------|
   |  (keys now exist, the rest is encrypted)     |
   |  IKE_AUTH request                            |
   |  IDi, AUTH, SAi2, TSi, TSr                   |
   |--------------------------------------------->|
   |  IKE_AUTH response                           |
   |  IDr, AUTH, SAr2, TSi, TSr                   |
   |<---------------------------------------------|
   |  IKE SA and first Child SA are now up        |
```

The letters: **SA** is the algorithm proposal, **KE** is the DH value, **N** is the nonce, **AUTH** is the identity proof and **TS** is a traffic selector. The ending **i** means initiator and **r** means responder.

| Exchange | Messages | Job |
|---|---|---|
| IKE_SA_INIT | 2 | Agree on algorithms, run DH, detect NAT |
| IKE_AUTH | 2 | Prove identity, create the first Child SA |
| CREATE_CHILD_SA | 2 | Add another Child SA, or rekey an SA |
| INFORMATIONAL | 2 | Delete SAs, report errors, liveness check |

Every request gets a response. If no response arrives, the sender retransmits. This makes IKEv2 more reliable than IKEv1.

Two extra round trips can appear, and neither is a failure:

- **Cookie:** a busy responder can ask the initiator to repeat its request with a cookie. This proves the source address is real before the responder spends CPU.
- **INVALID_KE_PAYLOAD:** the initiator sent a DH value for a group the responder does not want. The responder names the group it prefers, and the initiator tries again.

### IKEv1 vs IKEv2

| Point | IKEv1 | IKEv2 |
|---|---|---|
| Standard | RFC 2409, now deprecated by RFC 9395 | RFC 7296, the current standard |
| Messages for first tunnel | 9 or 6 | 4 |
| Modes | Main, Aggressive, Quick | One clean flow |
| NAT traversal | Add-on (RFC 3947) | Built in |
| Dead peer detection | Add-on (RFC 3706) | Built in |
| User login | Vendor extensions | EAP built in |
| Client IP assignment | Vendor extensions | Configuration payload built in |
| Different auth per side | No | Yes, for example certificate and EAP |
| DoS protection | Weak | Cookies |
| Lifetimes | Negotiated | Local, each side rekeys on its own |
| Mobile address change | No | Yes, MOBIKE (RFC 4555) |

**EAP (Extensible Authentication Protocol)** is a framework for user login methods. See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

Use IKEv2 for all new work. You will still meet IKEv1 on old equipment.

### NAT Traversal (NAT-T)

Plain ESP and NAT do not mix well. There are three problems:

1. **No ports.** PAT tracks flows by port. ESP has none, so PAT cannot map replies to the right inside host.
2. **Changed addresses.** AH fails, as explained above.
3. **Changed UDP 500 source port.** NAT may change it, and old IKE code expected 500.

**NAT-T** fixes this by putting ESP inside a UDP packet. NAT then sees a normal UDP flow with ports. The standards are RFC 3947 (detection in IKEv1) and RFC 3948 (UDP encapsulation). IKEv2 includes detection in RFC 7296.

```text
Normal ESP tunnel mode
+---------------+-----+------------------+---------+-----+
| IP (proto 50) | ESP | Encrypted packet | Trailer | ICV |
+---------------+-----+------------------+---------+-----+

ESP with NAT-T
+---------------+----------+-----+------------------+---------+-----+
| IP (proto 17) | UDP 4500 | ESP | Encrypted packet | Trailer | ICV |
+---------------+----------+-----+------------------+---------+-----+
                 8 bytes extra
```

The process:

1. **Detect support.** In IKEv1, peers send a Vendor ID payload in messages 1-2. In IKEv2, support shows through the NAT detection notifications.
2. **Detect NAT.** Each peer sends a hash of its IP address and port. The other peer recalculates the hash from the packet it received. A different hash means something changed the address, so a NAT is present.
3. **Move to port 4500.** IKE moves from UDP 500 to UDP 4500. In IKEv1 this starts at message 5. In IKEv2 it starts at IKE_AUTH.
4. **Wrap ESP in UDP.** ESP packets now travel in UDP 4500.
5. **Send keepalives.** A one-byte packet with value 0xFF keeps the NAT mapping alive. RFC 3948 suggests about every 20 seconds.

Port 4500 carries both IKE and ESP, so the receiver must tell them apart. IKE messages on port 4500 start with **4 zero bytes**, called the **Non-ESP Marker**. A valid ESP packet can never have an SPI of 0, so there is no confusion.

```text
IKE on port 4500
+----+-----+--------------------+------------+
| IP | UDP | 00 00 00 00        | IKE header |
+----+-----+--------------------+------------+

ESP on port 4500
+----+-----+--------------------+------------+
| IP | UDP | SPI (never zero)   | Rest of ESP|
+----+-----+--------------------+------------+

NAT keepalive
+----+-----+----+
| IP | UDP | FF |
+----+-----+----+
```

### Packet Overhead

IPsec makes each packet bigger. If the result is larger than the path MTU, the packet must be fragmented or dropped. **MTU (Maximum Transmission Unit)** is the largest IP packet a link can carry, usually 1500 bytes on Ethernet.

Overhead in bytes, IPv4:

| Part | AES-CBC with HMAC-SHA-256-128 | AES-GCM with 16-byte ICV |
|---|---|---|
| New outer IP header (tunnel only) | 20 | 20 |
| UDP header (NAT-T only) | 8 | 8 |
| ESP header (SPI and sequence) | 8 | 8 |
| IV | 16 | 8 |
| Padding | 0-15 | 0-3 |
| Pad Length and Next Header | 2 | 2 |
| ICV | 16 | 16 |

Totals:

| Setup | AES-CBC with SHA-256 | AES-GCM |
|---|---|---|
| Transport mode | 42-57 | 34-37 |
| Tunnel mode | 62-77 | 54-57 |
| Tunnel mode with NAT-T | 70-85 | 62-65 |

AES-CBC needs padding to a 16-byte block. AES-GCM has no block rule, but ESP still aligns the trailer to 4 bytes, so its padding is small.

AH transport mode adds 12 bytes plus the ICV. That is 24 bytes with HMAC-SHA1-96 and 28 with HMAC-SHA-256-128. Tunnel mode adds 20 more. IPv6 needs 8-byte alignment for AH, so an extra padding block may be added.

## Example

### Example 1: Site-to-Site Tunnel on Cisco IOS with IKEv2

**Goal:** LAN A and LAN B talk privately over the internet.

```text
 LAN A 10.1.1.0/24                         LAN B 10.2.2.0/24
 +-----------+   +----+                +----+   +-----------+
 | Host A    |---| R1 |====Internet====| R2 |---| Host B    |
 | 10.1.1.10 |   +----+                +----+   | 10.2.2.20 |
 +-----------+                                  +-----------+

 R1 Gi0/0   10.1.1.1/24          R2 Gi0/0   10.2.2.1/24
 R1 Gi0/1   198.51.100.2/30      R2 Gi0/1   203.0.113.2/30
 R1 gateway 198.51.100.1         R2 gateway 203.0.113.1
 R1 Tu0     172.16.12.1/30       R2 Tu0     172.16.12.2/30
```

This design uses a **VTI (Virtual Tunnel Interface)**. A VTI is a virtual interface where everything routed into it gets encrypted. Normal routing decides what enters the tunnel.

**R1 config (Cisco IOS or IOS XE):**

```text
! Phase 1 algorithms
crypto ikev2 proposal PROP-1
 encryption aes-gcm-256
 prf sha384
 group 20
!
crypto ikev2 policy POL-1
 proposal PROP-1
!
! Who the peer is and which key to use
crypto ikev2 keyring KR-1
 peer R2
  address 203.0.113.2
  pre-shared-key Example-Only-Use-A-Long-Random-Key
!
crypto ikev2 profile PROF-1
 match identity remote address 203.0.113.2 255.255.255.255
 identity local address 198.51.100.2
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-1
 dpd 10 3 periodic
!
! Phase 2 algorithms
crypto ipsec transform-set TS-GCM esp-gcm 256
 mode tunnel
!
crypto ipsec profile IPSEC-PROF
 set transform-set TS-GCM
 set ikev2-profile PROF-1
!
! Tunnel interface
interface Tunnel0
 ip address 172.16.12.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/1
 tunnel mode ipsec ipv4
 tunnel destination 203.0.113.2
 tunnel protection ipsec profile IPSEC-PROF
!
ip route 0.0.0.0 0.0.0.0 198.51.100.1
ip route 10.2.2.0 255.255.255.0 Tunnel0
```

**R2 config:** the same, with these values swapped.

```text
crypto ikev2 keyring KR-1
 peer R1
  address 198.51.100.2
  pre-shared-key Example-Only-Use-A-Long-Random-Key
!
crypto ikev2 profile PROF-1
 match identity remote address 198.51.100.2 255.255.255.255
 identity local address 203.0.113.2
!
interface Tunnel0
 ip address 172.16.12.2 255.255.255.252
 tunnel destination 198.51.100.2
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
ip route 10.1.1.0 255.255.255.0 Tunnel0
```

AES-GCM needs no separate integrity setting, because it does both jobs. The PRF (pseudo-random function) is still needed to make keys.

**Test:**

```text
R1# ping 10.2.2.1 source 10.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.2.1, timeout is 2 seconds:
Packet sent with a source address of 10.1.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 34/36/41 ms
```

**Check Phase 1:**

```text
R1# show crypto ikev2 sa
 IPv4 Crypto IKEv2  SA

Tunnel-id Local              Remote             fvrf/ivrf   Status
1         198.51.100.2/500   203.0.113.2/500    none/none   READY
      Encr: AES-GCM, keysize: 256, PRF: SHA384, Hash: None,
      DH Grp:20, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/412 sec
```

READY means the IKE SA is up. Output is wrapped here to fit the page.

**Check Phase 2:**

```text
R1# show crypto ipsec sa

interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 198.51.100.2

   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 203.0.113.2 port 500
    #pkts encaps: 5, #pkts encrypt: 5, #pkts digest: 5
    #pkts decaps: 5, #pkts decrypt: 5, #pkts verify: 5
    #send errors 0, #recv errors 0

    local crypto endpt.: 198.51.100.2, remote crypto endpt.: 203.0.113.2
     plaintext mtu 1446, path mtu 1500, ip mtu 1500
     current outbound spi: 0x4D7E91AA(1300140458)

     inbound esp sas:
      spi: 0xC8A1F2B3(3366056627)
        transform: esp-gcm 256 ,
        in use settings ={Tunnel, }

     outbound esp sas:
      spi: 0x4D7E91AA(1300140458)
        transform: esp-gcm 256 ,
        in use settings ={Tunnel, }
```

How to read this:

- **encaps** counts packets sent into the tunnel. **decaps** counts packets received from it.
- Both are 5, so traffic works both ways.
- Selectors are 0.0.0.0/0 because a VTI accepts anything routed into it.
- There are two SPIs, because there are two SAs.

**See it on the wire (Linux host on the path, or a SPAN port):**

```bash
sudo tcpdump -tni eth0 'udp port 500 or udp port 4500 or ip proto 50'
```

```text
IP 198.51.100.2.500 > 203.0.113.2.500: isakmp: parent_sa ikev2_init[I]
IP 203.0.113.2.500 > 198.51.100.2.500: isakmp: parent_sa ikev2_init[R]
IP 198.51.100.2.500 > 203.0.113.2.500: isakmp: child_sa  ikev2_auth[I]
IP 203.0.113.2.500 > 198.51.100.2.500: isakmp: child_sa  ikev2_auth[R]
IP 198.51.100.2 > 203.0.113.2: ESP(spi=0x4d7e91aa,seq=0x1), length 136
IP 203.0.113.2 > 198.51.100.2: ESP(spi=0xc8a1f2b3,seq=0x1), length 136
```

Check the numbers:

- R1 to R2 uses SPI 0x4D7E91AA. R2 chose it, and it is R1's outbound SPI.
- Length 136 is the ESP size for a 100-byte inner packet: 8 (ESP header) + 8 (IV) + 100 + 2 (padding) + 2 (trailer) + 16 (ICV) = 136.
- On the wire, the real addresses 10.1.1.1 and 10.2.2.1 are hidden.

### Example 2: The Classic IKEv1 Crypto Map

Many interviews and old networks still use this style. It is **policy-based**: an ACL decides what gets encrypted.

R1, Cisco IOS:

```text
! Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 28800
!
crypto isakmp key Example-Only-Use-A-Long-Random-Key address 203.0.113.2
!
! Phase 2
crypto ipsec transform-set TS-AES esp-aes 256 esp-sha256-hmac
 mode tunnel
!
! Interesting traffic
ip access-list extended VPN-TRAFFIC
 permit ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
!
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 203.0.113.2
 set transform-set TS-AES
 set pfs group14
 match address VPN-TRAFFIC
!
interface GigabitEthernet0/1
 ip address 198.51.100.2 255.255.255.252
 crypto map VPN-MAP
!
! Do not NAT traffic that belongs in the tunnel
ip access-list extended NAT-LIST
 deny   ip 10.1.1.0 0.0.0.255 10.2.2.0 0.0.0.255
 permit ip 10.1.1.0 0.0.0.255 any
!
ip nat inside source list NAT-LIST interface GigabitEthernet0/1 overload
```

R2 has the mirror ACL: `permit ip 10.2.2.0 0.0.0.255 10.1.1.0 0.0.0.255`.

The first ping often shows `.!!!!`. The first packet triggers IKE and is dropped while the tunnel builds.

```text
R1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.2     198.51.100.2    QM_IDLE           1001 ACTIVE
```

QM_IDLE means Phase 1 is complete and ready for Quick Mode. It is the good state, even though "idle" sounds bad.

### Example 3: The Same Idea on Linux and Windows

**Linux with strongSwan.** This is a policy-based site-to-site tunnel. It suits a peer that uses the same subnets. To pair with R1's VTI above, you would need 0.0.0.0/0 selectors and a route-based setup.

File `/etc/swanctl/swanctl.conf` (the path can differ by distribution):

```text
connections {
  site-a {
    version = 2
    local_addrs = 203.0.113.2
    remote_addrs = 198.51.100.2
    proposals = aes256gcm16-prfsha384-ecp384
    local {
      auth = psk
      id = 203.0.113.2
    }
    remote {
      auth = psk
      id = 198.51.100.2
    }
    children {
      lan {
        local_ts = 10.2.2.0/24
        remote_ts = 10.1.1.0/24
        esp_proposals = aes256gcm16
        start_action = trap
      }
    }
  }
}
secrets {
  ike-1 {
    id = 198.51.100.2
    secret = "Example-Only-Use-A-Long-Random-Key"
  }
}
```

```bash
# Linux with strongSwan: load config, start the tunnel, view state
sudo swanctl --load-all
sudo swanctl --initiate --child lan
sudo swanctl --list-sas

# Linux kernel view: the SAD and the SPD
sudo ip xfrm state
sudo ip xfrm policy
```

Linux splits the work. The strongSwan daemon runs IKE. The kernel XFRM layer handles ESP.

**Windows.** Windows has IPsec built in. It appears as Connection Security Rules in Windows Defender Firewall. The built-in VPN client also supports IKEv2.

```powershell
# Windows 10/11 or Server, run as Administrator
Get-NetIPsecRule | Select-Object DisplayName, Enabled, Mode
Get-NetIPsecMainModeSA      # Phase 1 SAs
Get-NetIPsecQuickModeSA     # Phase 2 SAs

# Create an IKEv2 client VPN profile
Add-VpnConnection -Name "Office" -ServerAddress vpn.example.com `
  -TunnelType Ikev2 -AuthenticationMethod Eap
```

### Example 4: Overhead on a Real Packet

**Setup:** tunnel mode, AES-256-CBC with HMAC-SHA-256-128, no NAT-T. The inner IP packet is 1400 bytes.

1. Find the padding. Inner packet + padding + 2 must divide by 16.
2. 1400 + 2 = 1402. The next multiple of 16 is 1408, so padding is 6 bytes.
3. Add everything: 20 (outer IP) + 8 (ESP header) + 16 (IV) + 1400 + 6 + 2 + 16 (ICV) = **1468 bytes**.
4. Overhead: 1468 - 1400 = **68 bytes**.

**Largest inner packet on a 1500-byte path, same setup:**

1. Space after fixed parts: 1500 - 20 - 8 - 16 - 16 = 1440.
2. 1440 divides by 16 exactly, so all of it is usable.
3. Remove 2 trailer bytes: 1440 - 2 = **1438 bytes**.

Other results on a 1500-byte path:

| Setup | Largest inner IP packet |
|---|---|
| AES-CBC with SHA-256, tunnel | 1438 |
| AES-CBC with SHA-256, tunnel, NAT-T | 1422 |
| AES-GCM, tunnel | 1446 |
| AES-GCM, tunnel, NAT-T | 1438 |

The fix most teams use is simple. Set the tunnel IP MTU to 1400 and the TCP MSS to 1360 (1400 - 20 - 20). That leaves a safe margin, and the same values work with NAT-T.

### Example 5: NAT-T Through a Home Router

A laptop behind a home router connects to VPN gateway R2.

```text
 Laptop               Home router (PAT)              R2
 192.168.1.10         198.51.100.7                   203.0.113.2
    |                       |                            |
    | src 192.168.1.10:500  |  src 198.51.100.7:40123    |
    |---------------------->|--------------------------->|
    |     IKE_SA_INIT, with hash of 192.168.1.10:500     |
    |                                                    |
    | R2 hashes 198.51.100.7:40123, result is different. |
    | NAT detected. Both sides move to UDP 4500.         |
    |                                                    |
    | src 192.168.1.10:4500 |  src 198.51.100.7:40999    |
    |---------------------->|--------------------------->|
    |     IKE_AUTH, then all ESP inside UDP              |
```

On a Cisco gateway, the signs of NAT-T are a port that is not 500 and the words UDP-Encaps:

```text
R2# show crypto ipsec sa | include current_peer|in use settings
   current_peer 198.51.100.7 port 40999
        in use settings ={Tunnel UDP-Encaps, }
```

### Example 6: Broken Tunnel, Phase 2 Fails

**Symptom:** the crypto map tunnel from Example 2 will not pass traffic.

**Step 1: check Phase 1.**

```text
R1# show crypto isakmp sa
dst             src             state          conn-id status
203.0.113.2     198.51.100.2    QM_IDLE           1001 ACTIVE
```

Phase 1 is good. So reachability, the Phase 1 policy and the PSK are all correct.

**Step 2: check Phase 2.**

```text
R1# show crypto ipsec sa | include pkts
    #pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0
    #pkts decaps: 0, #pkts decrypt: 0, #pkts verify: 0
```

No SAs carry traffic. Phase 2 failed.

**Step 3: read the log on the responder, R2.**

```text
R2# debug crypto ipsec
IPSEC(validate_proposal_request): proposal part #1
    local_proxy= 10.2.2.0/255.255.255.0/0/0 (type=4),
    remote_proxy= 10.1.1.0/255.255.255.0/0/0 (type=4),
IPSEC(validate_transform_proposal): proxy identities not supported
```

Message text can differ by version, but "proxy identities" points to the selectors.

**Step 4: compare ACLs.**

```text
R2# show ip access-lists VPN-TRAFFIC
Extended IP access list VPN-TRAFFIC
    10 permit ip 10.2.2.0 0.0.0.255 10.1.2.0 0.0.0.255
```

There is a typo: 10.1.2.0 should be 10.1.1.0.

**Step 5: fix and test.**

```text
R2(config)# ip access-list extended VPN-TRAFFIC
R2(config-ext-nacl)# no 10
R2(config-ext-nacl)# 10 permit ip 10.2.2.0 0.0.0.255 10.1.1.0 0.0.0.255
R2(config-ext-nacl)# end
R2# clear crypto sa
```

Then ping again from R1 with a LAN source. encaps and decaps should now rise.

## Going Deeper

### Troubleshooting Phase 1 and Phase 2 Failures

Work from the bottom up. Stop at the first step that fails.

```text
1. Can the peers reach each other's public IP?
        | yes
2. Do UDP 500, UDP 4500 and protocol 50 pass?
        | yes
3. Does traffic trigger the tunnel? (policy-based only)
        | yes
4. Is Phase 1 up?
        | yes
5. Is Phase 2 up?
        | yes
6. Do encaps and decaps both rise?
        | yes
7. Do large packets work?
```

**Useful commands by platform:**

| Task | Cisco IOS | Linux strongSwan | Windows |
|---|---|---|---|
| Phase 1 state | `show crypto ikev2 sa`, `show crypto isakmp sa` | `swanctl --list-sas` | `Get-NetIPsecMainModeSA` |
| Phase 2 state | `show crypto ipsec sa` | `ip xfrm state` | `Get-NetIPsecQuickModeSA` |
| Live logs | `debug crypto ikev2`, `debug crypto isakmp`, `debug crypto ipsec` | `swanctl --log` | Event Viewer, Security log |
| Capture | `monitor capture` | `tcpdump` | `pktmon` |
| Reset | `clear crypto ikev2 sa`, `clear crypto isakmp`, `clear crypto sa` | `swanctl --terminate --ike site-a` | Disconnect the VPN or restart the rule |

Debugs can be heavy on a busy router. Use `debug crypto condition peer ipv4 203.0.113.2` to limit output on Cisco IOS. See [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) and [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) for capture work. In Wireshark, the display filters `isakmp` and `esp` are the usual start.

**IKEv1 Main Mode states on Cisco IOS:**

| State | Meaning | If stuck here |
|---|---|---|
| MM_NO_STATE | Phase 1 just started, nothing agreed | No reply from peer, or no matching policy |
| MM_SA_SETUP | Policy agreed | DH step is not finishing, or UDP 500 is one-way |
| MM_KEY_EXCH | DH done, keys made | Often a PSK mismatch |
| MM_KEY_AUTH | Identity being checked | Identity or authentication problem |
| QM_IDLE | Phase 1 complete | This is the good state |

State names vary by product. Cisco ASA uses names such as MM_WAIT_MSG2 and MM_ACTIVE.

**Phase 1 failures:**

| What you see | Likely cause | What to do |
|---|---|---|
| Requests sent, no replies | Routing, firewall, wrong peer IP, peer service not running | Capture on both sides. Does UDP 500 arrive? |
| `no offers accepted`, or `atts are not acceptable` (IKEv1) | Encryption, hash, group or auth method differs | Compare policies line by line |
| NO_PROPOSAL_CHOSEN (IKEv2) | No common algorithm set | Compare proposals |
| Stuck at MM_KEY_EXCH, `IKMP_BAD_MESSAGE` (IKEv1) | PSK mismatch, so message 5 cannot be decrypted | Re-enter the key on both sides |
| AUTHENTICATION_FAILED (IKEv2) | PSK, certificate or identity mismatch | Check key, local ID and remote ID |
| Certificate rejected | Expired, wrong CA, wrong clock, revocation check unreachable | Check dates, NTP and the CA chain |
| IKE_SA_INIT works, IKE_AUTH never completes | UDP 4500 blocked after NAT, or IKE packets fragmented and dropped | Allow UDP 4500. Turn on IKEv2 fragmentation (RFC 7383) |
| Works until a NAT is added | NAT-T disabled, or AH in use | Enable NAT-T. Use ESP |

Large IKE_AUTH messages with certificates can exceed the MTU. Some networks drop UDP fragments. So a tunnel with PSK works, and the same tunnel with certificates fails. That pattern points to fragmentation.

**Phase 2 failures:**

| What you see | Likely cause | What to do |
|---|---|---|
| NO_PROPOSAL_CHOSEN after Phase 1 is up | Transform set mismatch | Match cipher, integrity and mode |
| `proxy identities not supported`, INVALID_ID_INFORMATION (IKEv1) | ACLs not mirrored | Compare selectors on both peers |
| TS_UNACCEPTABLE (IKEv2) | Traffic selectors do not match policy | Compare selectors |
| Quick Mode fails, everything else matches | PFS on one side only, or different groups | Match PFS settings |
| IKEv2 tunnel works for an hour, then drops | PFS group mismatch shows only at first rekey | Match the group, then test a rekey |
| Tunnel fine, then drops at lifetime expiry | Rekey problem | Check logs at the rekey time |

In IKEv2, a failed Child SA in IKE_AUTH does not always kill the IKE SA. So the IKE SA can show as up with no Child SA. Always check both.

**SAs are up but traffic fails:**

| Counters and logs | Likely cause | What to do |
|---|---|---|
| encaps 0 (crypto map) | Traffic does not match the ACL, or NAT changed it first | Add NAT exemption, check the ACL |
| encaps 0 (VTI) | No route into the tunnel | Check the route to the remote LAN |
| encaps rising, decaps 0 | Return traffic missing | Protocol 50 blocked, or peer lacks a return route |
| decaps rising, hosts see nothing | Peer LAN routing or host firewall | Check the inside path |
| `RECVD_PKT_INV_SPI` log | One peer lost its SAs, for example after a reboot | Enable DPD, clear SAs |
| `PKT_REPLAY_ERR` log | Packets arrive far out of order, often after QoS | Widen the replay window |
| Ping works, file transfers hang | MTU black hole | Lower tunnel MTU and MSS |
| Tunnel flaps every few minutes | DPD failing, or an unstable link | Check packet loss |

Test for MTU problems with a DF-bit ping across the tunnel:

```bash
# Linux: 1372 data + 8 ICMP + 20 IP = 1400 bytes, do not fragment
ping -M do -s 1372 10.2.2.20
```

```powershell
# Windows: same test
ping -f -l 1372 10.2.2.20
```

```text
! Cisco IOS: same test
R1# ping 10.2.2.1 source 10.1.1.1 size 1400 df-bit
```

### SPD, SAD and PAD

RFC 4301 defines three logical databases.

- **SPD (Security Policy Database):** the rules. For each packet, it says PROTECT, BYPASS or DISCARD.
- **SAD (Security Association Database):** the live SAs, with keys and counters.
- **PAD (Peer Authorization Database):** which peers are allowed and what identities they may use.

On Linux, `ip xfrm policy` shows the SPD and `ip xfrm state` shows the SAD. A Cisco ACL in a crypto map is effectively an SPD entry.

### Anti-Replay and Extended Sequence Numbers

The sender adds 1 to the sequence number for each packet. It starts at 1.

The receiver keeps a sliding window. A new number is accepted. A repeated number is dropped. A number older than the window is dropped.

RFC 4303 requires support for a window of at least 32 and suggests 64 as the default. High-speed links often need a much larger window, because QoS and parallel paths reorder packets.

A sequence number must never repeat under the same key. The 32-bit counter allows about 4.29 billion packets. At 10 Gbps with 1400-byte packets, that is about 893,000 packets per second. The counter runs out in about 80 minutes.

**ESN (Extended Sequence Numbers)** fixes this with a 64-bit counter. Only the low 32 bits travel in the packet. The high bits are included in the ICV calculation.

### Keys and PFS in IKEv2

After IKE_SA_INIT, both sides compute a value called SKEYSEED from the nonces and the DH secret. From this they derive seven keys, including:

- **SK_d:** used to derive Child SA keys.
- **SK_ei and SK_er:** encrypt IKE messages.
- **SK_ai and SK_ar:** protect IKE message integrity (not used with AEAD ciphers).
- **SK_pi and SK_pr:** used in the AUTH calculation.

The first Child SA comes from IKE_AUTH, which has no DH exchange of its own. So its keys come from SK_d. A separate PFS group applies only when a Child SA is created or rekeyed with CREATE_CHILD_SA.

That explains a confusing failure. A PFS mismatch does not stop the tunnel from coming up. It breaks the tunnel at the first rekey.

### Policy-Based vs Route-Based VPNs

| Point | Policy-based | Route-based |
|---|---|---|
| What selects traffic | ACL or policy | Routing table |
| Cisco example | Crypto map | VTI |
| Selectors | Specific subnets | Usually 0.0.0.0/0 |
| Dynamic routing (OSPF, BGP) | Not directly | Yes |
| Multicast | No | Yes, on VTI |
| Interface for counters and QoS | No | Yes |

Route-based is the normal choice now. Mixing the two can fail: a policy-based peer may offer specific subnets while a VTI expects any-to-any. Vendors handle this differently, so match the selectors. For more on VPN designs, see [VPNs](07-vpn.md).

### GRE over IPsec and L2TP/IPsec

Plain crypto maps carry only unicast IP. **GRE (Generic Routing Encapsulation)** can carry multicast and routing protocols. So older designs put GRE inside IPsec. GRE adds 24 bytes (20 IP + 4 GRE). Transport mode is common here, since GRE already gives a new IP header and a second one would waste 20 bytes.

L2TP/IPsec is another transport mode case. L2TP uses UDP 1701 and has no encryption, so IPsec transport mode protects it. See [VPNs](07-vpn.md).

### Fragmentation Details

Fragmenting after encryption forces the far gateway to reassemble before it can decrypt. That is slow. Many platforms, including Cisco IOS, can fragment before encryption. Each fragment is then a separate ESP packet.

The DF (Don't Fragment) bit is commonly copied from the inner header to the outer header. A router on the path may then return an ICMP "fragmentation needed" message to the gateway. The gateway must pass it back to the host for path MTU discovery to work. If ICMP is blocked, you get a black hole. That is why MSS clamping (`ip tcp adjust-mss`) is so common: it avoids the whole problem for TCP.

### The IKE Header

Every IKE message starts with a 28-byte header.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       IKE SA Initiator SPI                    |
|                            (8 bytes)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       IKE SA Responder SPI                    |
|                            (8 bytes)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Payload | MjVer | MnVer | Exchange Type |     Flags     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Message ID                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Length                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Note that IKE SPIs are 8 bytes each. ESP and AH SPIs are 4 bytes. They are different numbers with different jobs.

| Exchange Type value | Name |
|---|---|
| 2 | IKEv1 Main Mode (Identity Protection) |
| 4 | IKEv1 Aggressive Mode |
| 32 | IKEv1 Quick Mode |
| 34 | IKE_SA_INIT |
| 35 | IKE_AUTH |
| 36 | CREATE_CHILD_SA |
| 37 | INFORMATIONAL |

### Algorithm Choices in 2026

| Purpose | Good choice | Avoid |
|---|---|---|
| Encryption | AES-GCM (128 or 256), ChaCha20-Poly1305 | DES, 3DES |
| Integrity (non-AEAD) | HMAC-SHA-256 or stronger | MD5, SHA-1 |
| DH group | 14 or higher MODP, 19, 20, 21, 31 (Curve25519) | 1, 2, 5 |
| Authentication | Certificates, or long random PSK | Short or human-chosen PSK |
| IKE version | IKEv2 | IKEv1, Aggressive Mode |

The current requirements are in RFC 8221 (ESP and AH) and RFC 8247 (IKEv2). Each vendor has its own defaults, so always read the configuration.

With AES-GCM, a repeated key and nonce pair destroys security. So GCM must be used with IKE, never with manually set static keys (RFC 4106).

### Lifetimes, Rekeying and DPD

An SA has a limited life, so a stolen key has limited value. **Rekeying** creates a new SA before the old one ends. Traffic moves over without a gap.

Cisco IOS defaults, which other vendors do not share:

| Item | Default |
|---|---|
| IKE SA lifetime | 86400 seconds (24 hours) |
| IPsec SA lifetime | 3600 seconds (1 hour) |
| IPsec SA volume | 4,608,000 KB |

**DPD (Dead Peer Detection)** checks whether the peer is still alive. Without it, a peer can keep sending into a dead tunnel until the SA expires. IKEv1 uses RFC 3706. In IKEv2, an empty INFORMATIONAL message is the check.

### Load Sharing Limits

Routers using ECMP (equal-cost paths) usually hash on IPs and ports to pick a path. Plain ESP has no ports, and all traffic in a tunnel has the same outer IPs. So one tunnel usually takes one path, and often one CPU core.

Ways around this: use several tunnels, use several Child SAs where supported, or use NAT-T with varied source ports where the platform supports it.

### When UDP Is Blocked

Some networks block all UDP except DNS. RFC 9329 defines how to carry IKE and ESP inside TCP. Vendors also have their own methods. These add overhead and suffer TCP-in-TCP problems, so they are a last resort.

### IPsec and IPv6

In IPv6, AH and ESP are extension headers. They use the same values, 51 and 50, in the Next Header chain. The formats and modes are the same.

A common myth says IPv6 requires IPsec. The early rule was close to that, but RFC 6434 relaxed it to "SHOULD" for nodes, and RFC 8504 keeps that level. IPsec is not automatically on for IPv6 traffic.

### Post-Quantum Planning

A future quantum computer could break classic DH. An attacker could record traffic today and decrypt it later. RFC 8784 mixes an extra pre-shared key into IKEv2 keys. RFC 9370 lets IKEv2 run several key exchanges together, which allows a classic exchange plus a post-quantum one such as ML-KEM. Support varies by vendor and version.

### Cloud Use

Cloud VPN gateways speak standard IPsec, usually IKEv2 with BGP over a route-based tunnel. AWS Site-to-Site VPN gives two tunnels per connection for redundancy. See [Hybrid Cloud Connectivity](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).

## Common Mistakes

- **Saying "ESP uses port 50".** ESP is IP protocol 50. It has no ports.
- **Opening only UDP 500.** Most tunnels also need protocol 50 and UDP 4500.
- **Thinking AH encrypts.** AH gives integrity only. Anyone can read the data.
- **Thinking ESP protects the outer IP header.** It does not. In tunnel mode it protects the inner header, and the outer one stays open.
- **Believing tunnel mode means encrypted.** Tunnel mode is about packet shape. Encryption comes from ESP. AH in tunnel mode encrypts nothing.
- **Saying "an SA is two-way".** IPsec SAs are one-way. The IKE SA is two-way.
- **Thinking the sender chooses the SPI.** The receiver chooses it.
- **Reading QM_IDLE as a problem.** It is the healthy Phase 1 state.
- **Forgetting NAT exemption.** With a crypto map, NAT can change the source before the ACL match, so traffic never enters the tunnel.
- **Using ACLs that are not mirrors.** R1's source must be R2's destination, and the other way around.
- **Blaming the tunnel when a ping passes and file copies hang.** That pattern usually means MTU.
- **Testing from the router's WAN IP.** Traffic sourced from the public interface does not match a LAN-to-LAN policy. Use a LAN source address.
- **Thinking NAT-T is a tunnel by itself.** It only wraps ESP in UDP, so NAT can handle it.
- **Assuming IKEv2 needs matching lifetimes.** Lifetimes are local in IKEv2, and the side with the shorter value rekeys first.
- **Using a short PSK.** With Aggressive Mode, it can be cracked offline. A long random key is the minimum.

## Interview Questions

**Q: What is the difference between AH and ESP?**
A: AH is IP protocol 51 and gives integrity and authentication, but no encryption. ESP is IP protocol 50 and gives encryption, and normally integrity too. AH also covers the fixed parts of the outer IP header, so it breaks with NAT. ESP is what almost everyone uses.

**Q: Why does AH not work through NAT, while ESP can?**
A: AH includes the IP addresses in its integrity check, so when NAT changes an address, the check fails. ESP does not cover the outer header. With NAT-T, ESP is wrapped in UDP 4500, which gives NAT the ports it needs.

**Q: What is the difference between transport mode and tunnel mode?**
A: Transport mode keeps the original IP header and protects only the payload. It is used between hosts or for GRE and L2TP. Tunnel mode protects the whole original packet and adds a new IP header. It is used between gateways, and it hides the real source and destination.

**Q: What are an SA and an SPI? Why are there two IPsec SAs?**
A: An SA is a one-way agreement holding the keys, algorithms, counters and lifetime. The SPI is a 32-bit number in each packet that tells the receiver which SA to use. Two SAs exist because each direction has its own keys and counters. The receiver picks the SPI.

**Q: Walk me through IKEv1 Phase 1 and Phase 2.**
A: Phase 1 builds a secure control channel. Main Mode uses 6 messages: two to agree on policy, two for Diffie-Hellman, two encrypted ones for identity. Aggressive Mode does it in 3 but exposes identities. Phase 2 is Quick Mode with 3 encrypted messages. It agrees on the transform set and proxy IDs, and it creates the two IPsec SAs.

**Q: Why is IKEv2 better than IKEv1?**
A: It uses 4 messages instead of 6 or 9. NAT-T, dead peer detection, EAP and client configuration are built in. It has cookie protection against DoS, and each request gets a response, so it is more reliable. It also supports MOBIKE for clients that change IP. IKEv1 is deprecated by RFC 9395.

**Q: Which ports and protocols must a firewall allow for IPsec?**
A: UDP 500 for IKE, IP protocol 50 for ESP, and UDP 4500 for NAT-T. I would add IP protocol 51 only if the design uses AH.

**Q: How does NAT-T work?**
A: Peers detect NAT by exchanging hashes of their IP and port. If a hash does not match, a NAT changed the packet. Both sides move IKE to UDP 4500 and wrap ESP in UDP 4500. Keepalives hold the NAT mapping open. A 4-byte zero marker separates IKE from ESP on that port.

**Q: Phase 1 is up but Phase 2 fails. What do you check?**
A: Phase 1 being up proves reachability and the PSK. So I compare the transform sets, then the proxy IDs or traffic selectors, which must mirror each other. Then I check PFS on both sides. In logs I look for NO_PROPOSAL_CHOSEN, TS_UNACCEPTABLE or "proxy identities not supported".

**Q: Phase 1 will not come up. How do you troubleshoot?**
A: First I check reachability between public IPs and capture on UDP 500 at both ends. If packets arrive, I compare encryption, hash, DH group and authentication method. If the state sticks at MM_KEY_EXCH or I see AUTHENTICATION_FAILED, I check the PSK or certificates and the peer identities.

**Q: The tunnel is up, encaps rise, decaps stay at zero. What is wrong?**
A: Our packets go out, but nothing comes back through the tunnel. I check whether ESP reaches the peer, since protocol 50 may be blocked. Then I look at the peer's decaps and its return route. A NAT rule on the peer can also send replies outside the tunnel.

**Q: How much overhead does IPsec add, and how do you handle it?**
A: For ESP tunnel mode with AES-GCM, about 54 to 57 bytes, and 8 more with NAT-T. With AES-CBC and SHA-256 it is about 62 to 77. So a 1500-byte packet will not fit. I set the tunnel MTU to 1400 and clamp TCP MSS to 1360.

## Quick Recap

- IPsec protects IP packets using IKE for setup and ESP or AH for data.
- ESP is IP protocol 50 and encrypts. AH is IP protocol 51, does not encrypt and breaks with NAT. IKE uses UDP 500, and NAT-T uses UDP 4500.
- Transport mode keeps the IP header. Tunnel mode wraps the whole packet in a new one.
- An SA is one-way, so a tunnel has one IKE SA plus two IPsec SAs. The receiver picks the SPI.
- IKEv1 uses 6 or 3 messages in Phase 1 and 3 in Phase 2. IKEv2 uses 4 messages in total and is the choice for new work.
- ESP tunnel mode adds about 54-77 bytes, plus 8 with NAT-T, so lower the tunnel MTU and clamp TCP MSS.
- Phase 1 failures point to reachability, policy or authentication. Phase 2 failures point to transform sets, selectors or PFS.
