# VPNs

**Level:** Intermediate | **Prerequisites:** [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [The Routing Table](../05-routing/02-routing-table.md), [Cryptography Basics](02-cryptography-basics.md)

## In One Line

A VPN (Virtual Private Network) wraps your packets inside other packets, usually encrypted, so that two networks or a device and a network can talk privately across a network they do not trust, such as the internet.

## Why It Matters

Companies have a problem. Their servers sit on private addresses like 10.20.0.25. Those addresses cannot be reached from the internet. Yet staff work from home, and branch offices need the same servers.

There are two bad answers. You can rent a private line to every site, which costs a lot and does not help a person in a hotel. Or you can put the servers on the public internet, which exposes them to every attacker.

A VPN is the third answer. It uses the cheap public internet as the road, and it builds a private path on top of it.

Interviewers ask about VPNs because the topic tests many skills at once: routing, NAT, encryption, MTU and firewalls. A common question is "the VPN says connected but I cannot reach the server, what do you check?". You can only answer that if you understand how the tunnel really works.

VPNs also matter for security. A VPN gateway sits on the public internet and leads straight into the company. In the 2021 Colonial Pipeline attack, reports said the attackers entered through a VPN account with a stolen password and no second login factor. Gateways from several large vendors have also been attacked through software flaws.

## Simple Explanation

Imagine you want to send a private letter to a colleague in another office.

The postal service is public. Anyone handling the envelope can read the address, and could steam it open. So you do this:

1. You write the letter and put it in an envelope addressed to your colleague's desk number.
2. You lock that envelope inside a small steel box.
3. You put the box in a plain outer envelope addressed to the front office of the other building.
4. At the other building, the front office has the key. It opens the box and hands the inner envelope to your colleague's desk.

The postal workers only see the outer envelope. They know one building is writing to another building. They do not know which desk, and they cannot read the letter.

In this picture:

- The inner envelope is your original packet, with private addresses.
- The steel box is encryption (scrambling data so only a key holder can read it).
- The outer envelope is the new packet header, with public addresses.
- The front office is the VPN gateway (the device that opens and closes the tunnel).
- Putting one envelope inside another is called **tunnelling**.

**Where the analogy stops being true:** the word "tunnel" suggests a private pipe. There is no pipe. Your packets still travel over the same public routers as everyone else. They can be delayed, dropped or blocked. An observer can still see who talks to whom, when, and how much data moves. Only the contents are hidden.

## How It Works

### What a VPN does and what it does not do

A good VPN gives you three protections and one routing trick.

| A VPN gives you | Plain meaning |
|---|---|
| Confidentiality | Nobody on the path can read the contents |
| Integrity | Any change to a packet on the path is detected, and the packet is dropped |
| Authentication | Each end proves who it is before traffic flows |
| Private reachability | You can reach private addresses across a public network |

A VPN does not do these things:

| A VPN does not | Why not |
|---|---|
| Stop malware or phishing | It protects packets on the road, not what the packets contain |
| Make you anonymous | Logins, cookies and browser fingerprints still identify you |
| Protect traffic after the exit | From the gateway to the final server, traffic is outside the tunnel |
| Hide traffic from the VPN operator | The gateway decrypts everything it carries |
| Replace a firewall | A connected user can attack whatever the routes allow |
| Make a slow link fast | It adds headers, encryption work and often a longer path |

Not every VPN does all four things in the first table. Some "VPN" technologies have no encryption at all. MPLS L3VPN (a carrier service that keeps customers apart) and plain GRE give separation or reachability only. So "VPN" does not always mean "encrypted". This is a favourite interview trap.

### Tunnelling step by step

**Tunnelling** means putting one complete packet inside the payload of another packet. It is one more use of [encapsulation](../02-network-models/04-encapsulation.md).

Here is a real flow. A laptop in a hotel reaches an office server.

- Laptop on hotel Wi-Fi: 192.168.1.50
- Hotel public address after NAT: 198.51.100.77
- Laptop tunnel address: 10.8.0.2
- VPN gateway public address: 203.0.113.10
- Office server: 10.20.0.25

The steps:

1. An app on the laptop sends a packet. Source 10.8.0.2, destination 10.20.0.25.
2. The laptop checks its routing table. A route says 10.20.0.0/24 goes to the virtual interface `wg0`.
3. A **virtual interface** is a network card made of software. Packets sent to it go to the VPN program, not to a cable.
4. The VPN program encrypts the whole packet, including the private addresses.
5. It adds a new outer header. Source 192.168.1.50, destination 203.0.113.10, UDP port 51820.
6. The outer packet leaves through the real Wi-Fi card.
7. The hotel router does NAT (it swaps the private source address for its own public one), so the outer source becomes 198.51.100.77.
8. Internet routers forward the packet using only the outer header.
9. The gateway receives it, checks that it is genuine, and decrypts it.
10. The gateway now holds the original packet and routes it to 10.20.0.25.
11. The reply follows the same path backward.

```text
INNER packet (what the app sent):

  +--------------+----------------+---------------+
  | src 10.8.0.2 | dst 10.20.0.25 | TCP + data    |
  +--------------+----------------+---------------+

OUTER packet (what the internet sees):

  +-------------------+------------------+-----+-----------------+
  | src 198.51.100.77 | dst 203.0.113.10 | UDP | encrypted inner |
  +-------------------+------------------+-----+-----------------+
                                               ^^^^^^^^^^^^^^^^^^^
                                               private addresses
                                               and data are in here
```

Two words are used all the time:

- **Underlay:** the real network that carries the outer packets (here, the internet).
- **Overlay:** the virtual network that the tunnel creates (here, 10.8.0.0/24).

Think of it this way: **a tunnel is just a route to a virtual interface.** If the routing table does not send a packet to the tunnel, the packet is not protected. Most VPN problems are routing problems.

### Who sees what

| Observer | Can see | Cannot see |
|---|---|---|
| Hotel Wi-Fi owner, ISP | Your public IP, the gateway IP, volume, timing | Inner addresses, data |
| VPN gateway operator | Inner addresses, and data that is not separately encrypted | Contents of HTTPS sessions |
| The final website | The gateway's public IP (full tunnel), your login and cookies | Your real IP, unless something leaks it |

### Site-to-site VPN

A **site-to-site VPN** joins two whole networks. The gateways build the tunnel. Users and servers need no VPN software, and often do not know it exists.

```text
   HQ LAN                                       Branch LAN
 10.1.0.0/24                                    10.2.0.0/24

 +--------+    +--------+    Internet    +--------+    +--------+
 | Server |----|   R1   |=== tunnel =====|   R2   |----|   PC   |
 |10.1.0.5|    | 203.0. |   (encrypted)  | 198.51.|    |10.2.0.9|
 +--------+    | 113.1  |                | 100.2  |    +--------+
               +--------+                +--------+

  Plain traffic inside each LAN. Encrypted only between R1 and R2.
```

- It is always on. The gateways keep it up, or bring it up when traffic appears.
- The usual protocol is [IPsec](08-ipsec.md) with IKEv2. GRE over IPsec, DMVPN and [SD-WAN](../10-enterprise-and-datacenter/10-sd-wan.md) are common too.
- Cloud providers sell managed versions. See [Hybrid Cloud Connectivity](../11-cloud-and-containers/06-hybrid-cloud-connectivity.md).

With many sites, the number of tunnels matters. A full mesh (every site to every site) needs n(n-1)/2 tunnels. 10 sites need 45 tunnels, and 100 sites need 4,950. So most designs use hub-and-spoke (every branch connects to a central hub) or a dynamic system such as DMVPN, described in Going Deeper.

### Remote access VPN

A **remote access VPN** joins one device to a network. The device runs a VPN client (a small program). The gateway at the other end is often called a **concentrator**, because many clients end there.

```text
 Home laptop --\
 (10.8.0.2)     \
                 \   Internet    +---------+     Office LAN
 Hotel phone ------ tunnels ---->| Gateway |---> 10.20.0.0/24
 (10.8.0.3)      /               +---------+     file server
                /                203.0.113.10    10.20.0.25
 Cafe tablet --/
 (10.8.0.4)
```

| Point | Site-to-site | Remote access |
|---|---|---|
| Connects | Network to network | Device to network |
| Who runs the tunnel | Routers or firewalls | Software on the device |
| Software on the PC | None | VPN client (or a browser, see SSL VPN) |
| Who is authenticated | The gateway | The user, and often the device |
| Typical login | Certificate or pre-shared key | Password plus MFA, or a certificate |
| Time up | Always on | When the user needs it (or always-on by policy) |
| Common protocols | IPsec IKEv2, GRE over IPsec | SSL VPN, OpenVPN, WireGuard, IKEv2 |

**MFA** means multi-factor authentication: a password plus a second proof, such as a phone prompt.

A **pre-shared key (PSK)** is a secret that both sides are given in advance, like a shared password.

### Full tunnel vs split tunnel

Once a client connects, one question remains: **which traffic goes into the tunnel?** The routes decide.

```text
FULL TUNNEL: everything goes through the office

  Laptop ==== tunnel ==== Gateway ---> Office servers
                                  \--> Internet (Zoom, YouTube)

SPLIT TUNNEL: only office traffic uses the tunnel

  Laptop ==== tunnel ==== Gateway ---> Office servers
     \
      \---- direct ---------> Internet (Zoom, YouTube)
```

| Point | Full tunnel | Split tunnel |
|---|---|---|
| Client route | Default route points to the tunnel | Only company subnets point to the tunnel |
| Internet traffic | Passes through company security tools | Goes straight out |
| Gateway load | High | Lower |
| Video call quality | Often worse, because of the extra trip | Better |
| Security visibility | Company sees all traffic | Company misses direct traffic |
| Main risk | Slowness, gateway overload | The device sits on two networks at once |

Some setups use the reverse rule: everything goes through the tunnel except a short list, such as a video service. This is often called **split exclude** or inverse split tunnelling.

DNS is the system that turns names into IP addresses. Split tunnelling also needs **split DNS**: names like `intranet.corp.example.com` go to the internal DNS server, and other names go to the normal resolver.

How full tunnel is done in practice:

- OpenVPN's `redirect-gateway def1` adds two routes, 0.0.0.0/1 and 128.0.0.0/1. These are more specific than the real default route, so they win by [longest prefix match](../05-routing/02-routing-table.md). The real default route stays in place.
- A /32 route to the VPN server stays on the real gateway. Without it, the encrypted packets would try to enter the tunnel themselves, and the tunnel would collapse.

```text
Linux client, full tunnel with OpenVPN:

$ ip route
0.0.0.0/1 via 10.9.0.1 dev tun0
default via 192.168.1.1 dev wlan0
10.9.0.0/24 dev tun0 proto kernel scope link src 10.9.0.2
128.0.0.0/1 via 10.9.0.1 dev tun0
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.50
203.0.113.10 via 192.168.1.1 dev wlan0
```

Windows built-in client, choosing split or full (PowerShell):

```powershell
# Create an IKEv2 connection in split tunnel mode
Add-VpnConnection -Name "Office" -ServerAddress "vpn.example.com" `
  -TunnelType Ikev2 -AuthenticationMethod Eap -SplitTunneling

# Send only the office subnet into the tunnel
Add-VpnConnectionRoute -ConnectionName "Office" -DestinationPrefix 10.20.0.0/24

# Change to full tunnel later
Set-VpnConnection -Name "Office" -SplitTunneling $false
```

### SSL VPN

An **SSL VPN** is a remote access VPN that uses [TLS](../07-application-protocols/07-tls-and-https.md), the same security protocol as HTTPS. The name is old. SSL itself is deprecated, and these products use TLS today. People kept the name.

It usually runs on TCP port 443, because firewalls almost always allow that port. So it works from hotels and guest networks where other VPNs get blocked.

There are two forms:

| Form | What the user does | What they get |
|---|---|---|
| Clientless (portal) | Logs in to a web page in a browser | Access to chosen web apps, sometimes file shares, through the portal. Not a real network connection. |
| Full client | Installs a client such as Cisco Secure Client (formerly AnyConnect) | A virtual interface and a full IP tunnel |

Important details:

- "SSL VPN" is a product category, not one protocol. Each vendor has its own protocol on top of TLS, so clients from one vendor generally do not work with gateways from another.
- A tunnel over TCP has a known problem, explained under TCP meltdown in Going Deeper. So good clients add DTLS (Datagram TLS, a form of TLS that runs on UDP), commonly on UDP 443, and fall back to TLS on TCP when UDP is blocked.
- Because the gateway is a big web application facing the internet, it is a popular target. Keep it patched, and require MFA. Some vendors now steer customers toward IPsec for this reason.

### OpenVPN

**OpenVPN** is an open-source VPN program that runs on almost every operating system. It is not an IETF standard (IETF is the group that publishes internet standards). It has its own protocol.

How it works:

- It has two channels. The **control channel** uses TLS to log in and agree on keys. The **data channel** carries your packets, encrypted with those keys, commonly using AES-256-GCM.
- The registered port is UDP 1194. It can also run on TCP, often on port 443.
- It normally runs in user space (as a normal program). Each packet moves between the kernel and the program, which costs speed. Newer versions can use Data Channel Offload (DCO) to do the data work in the kernel, which helps a lot.
- It authenticates with certificates, and it can add a username, password and MFA on top.

It offers two kinds of virtual interface:

| Interface | Carries | Use it for |
|---|---|---|
| `tun` | Layer 3, IP packets | Almost everything (routed VPN) |
| `tap` | Layer 2, Ethernet frames | Rare cases that need broadcasts or a stretched LAN |

Server config sketch (Linux):

```text
port 1194
proto udp
dev tun
topology subnet
server 10.9.0.0 255.255.255.0
ca ca.crt
cert server.crt
key server.key
dh none
tls-crypt ta.key
data-ciphers AES-256-GCM:CHACHA20-POLY1305
keepalive 10 60

# Split tunnel: send only this route to clients
push "route 10.20.0.0 255.255.255.0"

# Full tunnel: use this line instead
# push "redirect-gateway def1"
# push "dhcp-option DNS 10.20.0.53"
```

`tls-crypt` encrypts the control channel with a shared key. It helps hide the handshake and blocks random scanners, because packets without the key are dropped quickly.

### WireGuard

**WireGuard** is a modern VPN with a very small design. It has been part of the Linux kernel since version 5.6 (2020). It has clients for Windows, macOS, iOS and Android. It is not an IETF standard either.

Main ideas:

- **UDP only.** 51820 is the common choice, not a fixed standard. You can pick any port.
- **Fixed crypto.** It uses Curve25519, ChaCha20-Poly1305 and BLAKE2s. There is no menu of ciphers, so there is no way to trick the two sides into choosing a weak one. If one of these is ever broken, a new protocol version replaces the set.
- **Keys, not accounts.** Each peer has a key pair, as with SSH. A **peer** is the other end of the tunnel.
- **Small code.** About 4,000 lines of kernel code at its first release, far less than typical VPN stacks. Less code is easier to review.
- **Silent.** It does not reply to packets that do not prove knowledge of its public key. A port scan sees nothing.
- **Roaming.** A peer may change IP, for example from Wi-Fi to mobile data. The tunnel continues, because peers are identified by key, not by address.
- **Forward secrecy.** It makes new session keys about every two minutes. **Forward secrecy** means that stealing a long-term key later does not unlock old recorded traffic.

Its core idea is **cryptokey routing**. `AllowedIPs` ties a public key to a list of addresses. It does two jobs:

- Sending: a packet to one of these addresses is encrypted for this peer.
- Receiving: a packet from this peer is accepted only if its inner source is in this list.

What WireGuard leaves out:

- No user login, MFA or central user database. Other tools add these.
- No automatic address assignment. Addresses are set in config.
- No TCP mode. Networks that block UDP block WireGuard.
- The server must hold each peer's last IP in memory while it works. Privacy-focused services add extra handling for this.

Products such as Tailscale build on WireGuard. They add central login, key distribution and NAT traversal, so devices form a mesh.

### L2TP/IPsec

**L2TP** (Layer 2 Tunnelling Protocol, RFC 2661) carries PPP sessions over IP. **PPP** (Point-to-Point Protocol) is the old protocol from dial-up days. It handles login and gives the client an IP address.

**L2TP has no encryption of its own.** So it is paired with IPsec. RFC 3193 describes this pairing. L2TP does the tunnelling and user login. IPsec does the protection, usually in transport mode.

```text
L2TP/IPsec packet, field by field:

+-------+-----+-----+------+-----+----------+------+---------+
| Outer | ESP | UDP | L2TP | PPP | Inner IP | Data | ESP     |
| IP    |     |1701 |      |     | packet   |      | trailer |
+-------+-----+-----+------+-----+----------+------+---------+
              |<----------- encrypted ----------------->|
```

Ports to allow through a firewall:

- UDP 500 for IKE (the key exchange)
- UDP 4500 for NAT traversal, when either side is behind NAT
- IP protocol 50 for ESP (the protected data), when there is no NAT
- UDP 1701 is carried inside the protection, so you do not open it to the internet

It has two login layers: the machine (a PSK or certificate for IPsec) and the user (through PPP).

Its weak points:

- Double wrapping costs more overhead.
- Many deployments used one shared PSK for all users, which is weak.
- NAT and firewalls cause trouble.

It is a legacy choice in 2026. It still appears because old systems have a built-in client. For new designs, use IKEv2, OpenVPN or WireGuard.

### PPTP: insecure history

**PPTP** (Point-to-Point Tunneling Protocol, RFC 2637, 1999) was one of the first widely used VPNs, thanks to a built-in client in Windows.

- It uses TCP 1723 for control and an enhanced form of GRE (IP protocol 47) for data.
- It encrypts with MPPE, which uses the RC4 cipher.
- It normally logs in with MS-CHAPv2.

Why it is broken:

- Researchers found weaknesses in it in the late 1990s.
- In 2012, Moxie Marlinspike and David Hulton showed that MS-CHAPv2 can be reduced to a single DES key search. They offered a cloud service to do this in about a day.
- RC4 is itself broken and is banned in modern TLS.

So anyone who captures a PPTP login can recover what they need to read the session. Apple removed PPTP from iOS 10 and macOS Sierra in 2016.

**The interview answer:** PPTP must never be used. Know it only as history, and know its ports so you can find and remove it.

### GRE

**GRE** (Generic Routing Encapsulation, RFC 2784) is a simple tunnel. It wraps almost any protocol inside IP. It is not a port number. It is **IP protocol 47**, so it sits directly on IP, next to TCP (6) and UDP (17).

**GRE has no encryption and no authentication.** It is a tunnel, not a security tool.

Why use it at all? It carries things that basic IPsec does not:

- Multicast, which routing protocols such as OSPF need to find neighbours
- Non-IP traffic
- Routing protocol adjacencies, as if the two routers were directly connected

So the classic design is **GRE over IPsec**. GRE provides a real routed interface. IPsec provides the security.

```text
GRE base header (RFC 2784), 4 bytes:

 Bit 0     Bits 1-12     Bits 13-15  Bits 16-31
+-----+----------------+-----------+---------------------+
|  C  |   Reserved0    |  Version  |   Protocol Type     |
+-----+----------------+-----------+---------------------+
| Checksum (optional)              | Reserved1 (optional)|
+----------------------------------+---------------------+

C = 1 means the checksum fields are present.
Protocol Type 0x0800 means the payload is IPv4.
RFC 2890 uses two reserved bits to add an optional Key and
Sequence Number.
```

Overhead is 24 bytes over IPv4: 20 for the new IP header and 4 for GRE. So an interface MTU of 1500 leaves 1476 for the inner packet.

### Comparison table

The ports below are the normal ones. Most can be changed by the admin, except IP protocol numbers.

| Technology | Encrypted? | Transport | Usual use | Status in 2026 |
|---|---|---|---|---|
| PPTP | Yes, but broken | TCP 1723 + GRE (IP 47) | Old remote access | Do not use |
| L2TP alone | No | UDP 1701 | Rarely alone; carrier tunnels | Not a security tool |
| L2TP/IPsec | Yes | UDP 500, UDP 4500, ESP | Old remote access | Legacy |
| GRE | No | IP protocol 47 | Router tunnels | Current, with IPsec on top |
| IPsec IKEv2 | Yes | UDP 500, UDP 4500, ESP | Site-to-site, mobile users | Current, open standard |
| SSL VPN (vendor) | Yes | TCP 443, often UDP 443 DTLS | Remote access | Common, gateways need care |
| SSTP (Microsoft) | Yes | TCP 443 | Windows remote access | Niche |
| OpenVPN | Yes | UDP 1194 or TCP | Remote access, small sites | Current, flexible |
| WireGuard | Yes | UDP, often 51820 | Remote access, mesh, sites | Current, fast |
| MPLS L3VPN | No, only separated | Provider MPLS network | Carrier WAN | Current, trust the carrier |

And in terms of trade-offs:

| Technology | Strength | Weakness |
|---|---|---|
| IPsec IKEv2 | Works across vendors; hardware support | Complex settings; mismatches are common |
| SSL VPN | Passes strict firewalls; portal option | Vendor lock-in; TCP tunnel issues; large attack surface |
| OpenVPN | Runs anywhere; flexible logins | Slower without DCO; large config |
| WireGuard | Fast, simple, small code | No built-in user management; UDP only |
| GRE | Simple, carries multicast | No security at all |

Rough overhead on the outer path. These are approximate and depend on cipher and options:

| Technology | Extra bytes per packet (IPv4 outer) |
|---|---|
| GRE | 24 |
| WireGuard | 60 |
| IPsec ESP tunnel mode, AES-GCM | About 55, plus 8 with NAT traversal |
| OpenVPN over UDP with AES-GCM | About 50 to 70 |
| L2TP/IPsec | About 70 to 100 |

### Commercial VPN myths

Consumer VPN services and company VPNs use the same technology for opposite goals.

```text
COMPANY VPN: reach a private network from outside

  You === tunnel === Company gateway ---> Private servers

CONSUMER VPN: reach the public internet from someone else's IP

  You === tunnel === Provider server ---> Public websites
```

A consumer VPN **moves trust**. It does not remove it. Your ISP can no longer see which sites you use, but now the VPN provider can.

| Advert says | What is true |
|---|---|
| "You are anonymous" | Sites still know you from logins, cookies, payments and browser fingerprints. The provider also knows your real IP. |
| "Stops hackers" | It does not stop malware, phishing or bad passwords. |
| "You need it to be safe on Wi-Fi" | Most sites already use HTTPS. A VPN still hides your DNS and destinations from the Wi-Fi owner, so it helps, but the old fear of passwords being stolen on cafe Wi-Fi is much smaller now. |
| "Military-grade encryption" | A marketing phrase, usually meaning AES-256. HTTPS uses the same kind of cipher. |
| "No-logs policy" | A promise you cannot check from outside. An audit shows one moment in time. Trust depends on the company. |
| "Free forever" | Running servers costs money. Free services may earn it from ads, data or selling your bandwidth. |
| "Makes the internet faster" | It usually adds delay. It may help only if your ISP slows a certain kind of traffic, or the provider has a better route. |
| "A VPN makes it legal" | Laws still apply. VPN use itself is restricted in some countries. |

Honest reasons to use one: hide browsing from your ISP or a network you do not trust, appear in another region, or get a different public IP. A browser "VPN" extension is often just a [proxy](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md): it covers only that browser's traffic.

## Example

### Example 1: WireGuard remote access, split tunnel

**Goal:** a laptop in a hotel reaches an office server at 10.20.0.25. Other internet traffic stays direct.

```text
 Laptop                 Hotel NAT      Gateway       Office LAN
                                                     10.20.0.0/24
 192.168.1.50 ---------- router ------ eth0: 203.0.113.10
 wg0: 10.8.0.2          198.51.100.77  eth1: 10.20.0.1 --- 10.20.0.25
                                       wg0:  10.8.0.1      (server)
```

**Step 1: make a key pair on each side (Linux).**

```bash
umask 077
wg genkey | tee private.key | wg pubkey > public.key
```

The private key never leaves its machine. You only copy public keys.

**Step 2: server config, `/etc/wireguard/wg0.conf`.**

```text
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server private key>
# NAT VPN clients to the gateway's LAN address, so LAN hosts can reply
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth1 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth1 -j MASQUERADE

[Peer]
# Laptop
PublicKey = <laptop public key>
AllowedIPs = 10.8.0.2/32
```

Note that the laptop's `/32` means the gateway accepts only 10.8.0.2 from this key.

**Step 3: laptop config. The same file works on Linux and in the Windows client.**

```text
[Interface]
Address = 10.8.0.2/32
PrivateKey = <laptop private key>
DNS = 10.20.0.53

[Peer]
PublicKey = <server public key>
Endpoint = 203.0.113.10:51820
AllowedIPs = 10.8.0.0/24, 10.20.0.0/24
PersistentKeepalive = 25
```

`AllowedIPs` makes this a split tunnel. For a full tunnel, write `AllowedIPs = 0.0.0.0/0, ::/0`. The gateway then also needs NAT out of `eth0`, its public side.

`PersistentKeepalive = 25` sends a small packet every 25 seconds. This keeps the hotel's NAT mapping alive so the gateway can reach the laptop.

**Step 4: start it.**

```bash
# Linux gateway: allow routing, then bring the tunnel up
sudo sysctl -w net.ipv4.ip_forward=1
sudo wg-quick up wg0

# Linux laptop
sudo wg-quick up wg0
```

```powershell
# Windows laptop, from an admin PowerShell (or use the WireGuard app)
& "C:\Program Files\WireGuard\wireguard.exe" /installtunnelservice "C:\vpn\wg0.conf"
```

**Step 5: check the tunnel on the gateway.**

```text
$ sudo wg show
interface: wg0
  public key: kX3vYwq8Lr0Zt1mN6hJ2bFd9cS4uEaP7oGiR5yTnQlM=
  private key: (hidden)
  listening port: 51820

peer: 9HqTzB2xVn4LfW7sKpY1cJm8RdU3eAg6oNiZ5tXwQbE=
  endpoint: 198.51.100.77:62011
  allowed ips: 10.8.0.2/32
  latest handshake: 42 seconds ago
  transfer: 18.42 MiB received, 96.13 MiB sent
```

The endpoint shows the hotel's NAT address, not 192.168.1.50. The port has also changed. The gateway learned it from the arriving packet.

**Step 6: check the routes on the laptop.**

```bash
# Linux
$ ip route
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
10.8.0.0/24 dev wg0 scope link
10.20.0.0/24 dev wg0 scope link
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.50

$ ip route get 10.20.0.25
10.20.0.25 dev wg0 src 10.8.0.2 uid 1000
    cache

$ ip route get 198.51.100.200
198.51.100.200 via 192.168.1.1 dev wlan0 src 192.168.1.50 uid 1000
    cache
```

```powershell
# Windows
PS> Get-NetRoute -DestinationPrefix 10.20.0.0/24

ifIndex DestinationPrefix NextHop RouteMetric ifMetric PolicyStore
------- ----------------- ------- ----------- -------- -----------
23      10.20.0.0/24      0.0.0.0           0        5 ActiveStore
```

Office traffic uses `wg0`. Other traffic uses `wlan0`. That is split tunnelling, shown by the routing table.

**Step 7: prove the traffic is encrypted.** Run `ping 10.20.0.25` and capture on both interfaces.

```text
# On the Wi-Fi card: only UDP, no private addresses
$ sudo tcpdump -ni wlan0 udp port 51820
12:01:33.104512 IP 192.168.1.50.41822 > 203.0.113.10.51820: UDP, length 148
12:01:33.151208 IP 203.0.113.10.51820 > 192.168.1.50.41822: UDP, length 92
12:01:33.151990 IP 192.168.1.50.41822 > 203.0.113.10.51820: UDP, length 128
12:01:33.199410 IP 203.0.113.10.51820 > 192.168.1.50.41822: UDP, length 128

# On the tunnel interface: the plain packets before encryption
$ sudo tcpdump -ni wg0 icmp
12:01:33.151800 IP 10.8.0.2 > 10.20.0.25: ICMP echo request, id 7, seq 1, length 64
12:01:33.199600 IP 10.20.0.25 > 10.8.0.2: ICMP echo reply, id 7, seq 1, length 64
```

Reading the sizes:

- 148 and 92 are the two WireGuard handshake messages.
- The ping is 84 bytes (56 data, 8 ICMP, 20 IP). WireGuard pads it to 96, then adds 32 bytes of its own, which gives 128.

**Step 8: check the MTU.** WireGuard's default MTU from `wg-quick` is usually 1420, which is 1500 minus 80. The 80 covers the worst case, an IPv6 outer path.

```text
Outer IPv6 header         40 bytes
UDP header                 8 bytes
WireGuard header and tag  32 bytes
Total                     80 bytes  (60 with an IPv4 outer header)
```

```bash
# Linux: 1392 data + 28 headers = 1420
$ ping -M do -s 1392 -c 1 10.20.0.25
1400 bytes from 10.20.0.25: icmp_seq=1 ttl=63 time=48.1 ms

$ ping -M do -s 1393 -c 1 10.20.0.25
ping: local error: message too long, mtu=1420
```

```powershell
# Windows: 1392 works, 1393 reports that it needs fragmenting
ping -f -l 1392 10.20.0.25
ping -f -l 1393 10.20.0.25
```

### Example 2: GRE tunnel between two Cisco routers

**Goal:** join HQ and a branch, then see why GRE alone is not enough.

```text
  10.1.0.0/24                                       10.2.0.0/24
       |                                                 |
    Gi0/1 10.1.0.1                               10.2.0.1 Gi0/1
     +----+  Gi0/0                       Gi0/0   +----+
     | R1 |  203.0.113.1  ~ Internet ~ 198.51.100.2 | R2 |
     +----+                                      +----+
       Tunnel0 172.16.0.1/30 ===== Tunnel0 172.16.0.2/30
```

```text
! Cisco IOS, R1. R2 mirrors this with source and destination swapped,
! address 172.16.0.2, and a route to 10.1.0.0/24.
interface Tunnel0
 description GRE to Branch
 ip address 172.16.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.2
 keepalive 10 3
!
ip route 10.2.0.0 255.255.255.0 172.16.0.2
```

`tunnel mode gre ip` is the default, so it does not appear here. `ip mtu 1400` and `ip tcp adjust-mss 1360` are common values that leave room for IPsec to be added later. For GRE alone, 1476 and 1436 would fit.

The same tunnel on a Linux router:

```bash
sudo ip tunnel add gre1 mode gre local 203.0.113.1 remote 198.51.100.2 ttl 255
sudo ip addr add 172.16.0.1/30 dev gre1
sudo ip link set gre1 mtu 1476 up
sudo ip route add 10.2.0.0/24 via 172.16.0.2
```

Verify on R1:

```text
R1#show interfaces Tunnel0
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Description: GRE to Branch
  Internet address is 172.16.0.1/30
  Keepalive set (10 sec), retries 3
  Tunnel source 203.0.113.1 (GigabitEthernet0/0), destination 198.51.100.2
  Tunnel protocol/transport GRE/IP

R1#ping 10.2.0.1 source 10.1.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.1.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/14/18 ms
```

Now capture the same ping on the WAN link, from a Linux machine that can see it:

```text
$ sudo tcpdump -ni eth0 ip proto 47
13:10:02.511204 IP 203.0.113.1 > 198.51.100.2: GREv0, length 104: IP 10.1.0.1 > 10.2.0.1: ICMP echo request, id 5, seq 0, length 80
```

**The inner addresses and ICMP are in plain view.** Anyone on the path can read them. To fix this, add IPsec. On Cisco, this is one line on the tunnel interface, `tunnel protection ipsec profile <name>`, after you build the profile. See [IPsec](08-ipsec.md).

## Going Deeper

### TCP meltdown

Running TCP inside a TCP tunnel can go badly wrong.

1. A packet is lost on the outer path.
2. The outer TCP retransmits it and slows down.
3. While it waits, the inner TCP sees a delay and thinks its own data was lost.
4. The inner TCP also retransmits, and adds more data to a tunnel that has already slowed.
5. Both layers back off and retry at once. Throughput can collapse.

This is why tunnels prefer UDP. The outer layer just carries packets. Only the inner TCP handles loss and [congestion](../06-transport-layer/06-tcp-congestion-control.md). Use TCP 443 tunnels only as a fallback for networks that block UDP. QUIC-based tunnels are a newer approach to the same issue.

### MTU and fragmentation

Each tunnel header takes space. A full 1500-byte inner packet will not fit.

- If the packet has the Don't Fragment bit set, a router must drop it and send an ICMP error.
- If a firewall blocks that ICMP, the sender never learns. This is a **black hole**.
- The classic symptom: ping works, small pages load, but file transfers or large pages hang.

Fixes: set a lower tunnel MTU, and clamp the TCP MSS (the largest TCP data size) so TCP sessions start with a smaller size. Cisco uses `ip tcp adjust-mss`. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Leaks and kill switches

A **leak** is traffic that escapes the tunnel while the user thinks it is protected.

| Leak | Cause | Fix |
|---|---|---|
| DNS leak | The system still asks the local resolver | Push DNS through the tunnel; force it with policy |
| IPv6 leak | The tunnel handles IPv4 only, and the device also has IPv6 | Tunnel IPv6 too, or block it |
| Tunnel drop | The tunnel fails, and traffic goes out the normal way | **Kill switch**: firewall rules that block all non-tunnel traffic |
| WebRTC | A browser feature can reveal local or public addresses to a site | Browser settings or extension |
| Route attack | The local network pushes routes more specific than the tunnel's | See below |

**TunnelVision** (CVE-2024-3661, a CVE being a public ID for a security flaw) is a good example of the last row. A malicious DHCP server on the local network uses option 121 (classless static routes) to install specific routes. These beat the VPN's routes, so traffic skips the tunnel while the client still shows "connected". This reminds you that a tunnel is only a route. Firewall rules that force traffic into the tunnel are a stronger control.

### Overlapping subnets

The laptop is on a home network, 192.168.1.0/24. The office LAN is also 192.168.1.0/24. Which way does 192.168.1.20 go? The connected home route usually wins or conflicts with the tunnel route, so the office server cannot be reached.

The same thing happens when two companies merge. Fixes:

- Avoid common ranges for company LANs. Choose something unusual inside 10.0.0.0/8.
- Use a /32 route for the one server, which is more specific.
- Use NAT across the tunnel, so each side sees a different range.

### Recursive routing in GRE

The route to the tunnel destination must not go through the tunnel. Suppose R1 learns 198.51.100.2 through a routing protocol running inside Tunnel0. The router would need the tunnel to reach the tunnel's own endpoint.

Cisco IOS detects this, shuts the tunnel for a while, and logs a message like `%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing`. The tunnel then flaps up and down.

The fix: keep underlay and overlay routes separate. Use a static route for the endpoint, or a filter, or put the underlay in a separate VRF.

### DMVPN

**DMVPN** (Dynamic Multipoint VPN) is a Cisco design for many sites. It combines three parts:

- **mGRE** (multipoint GRE): one tunnel interface reaches many peers.
- **NHRP** (Next Hop Resolution Protocol): a directory on the hub. A spoke asks it for another spoke's public IP.
- **IPsec**: protects the traffic.

Spokes build a tunnel to the hub. When one spoke needs another, they can build a direct tunnel on demand. You get the small config of hub-and-spoke and the direct path of a mesh. In newer networks, SD-WAN often takes this role.

### MPLS L3VPN is a VPN without encryption

A carrier's [MPLS](../05-routing/13-mpls.md) L3VPN (RFC 4364) keeps customers apart with labels and VRFs (separate routing tables). Traffic is private because it is separated, not because it is encrypted. You trust the carrier.

So two terms are useful: a **trusted VPN** relies on the provider keeping traffic separate, and a **secure VPN** relies on encryption. Some companies run IPsec on top of MPLS if they do not want to trust the carrier.

### IKEv2 for remote access

The protocol [IPsec](08-ipsec.md) uses for keys, IKEv2 (RFC 7296), is also strong for mobile users. Its MOBIKE extension (RFC 4555) lets a session survive when a phone moves from Wi-Fi to mobile data. It is built into Windows, macOS and iOS. It uses UDP 500 and UDP 4500, so networks that block those ports will block it.

### Other things called tunnels

| Tool | What it is | How it differs from a VPN |
|---|---|---|
| `ssh -D 1080 user@host` | SOCKS proxy over [SSH](../07-application-protocols/09-remote-access-protocols.md) | Per-app, TCP mainly, no virtual interface |
| Tor | Traffic relayed through three volunteer nodes | Aims at anonymity, not private network access |
| [VXLAN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) | Layer 2 over UDP in data centres | No encryption; for scale, not secrecy |

### Forward look: post-quantum

Attackers can record encrypted traffic today and hope to decrypt it with a future quantum computer. Key exchange is the part at risk. WireGuard offers an optional `PresharedKey` per peer. It mixes a symmetric secret into the handshake as an extra layer. IKEv2 has similar additions, and vendors are adding post-quantum key exchange. Check what your version supports.

### VPN vs Zero Trust

A classic VPN gives a user a place on the network, so a stolen login can reach a lot. ZTNA (Zero Trust Network Access) gives access to one app at a time, after checking the user and device. Many firms are moving this way, but VPNs remain very common, especially site-to-site. See [Zero Trust and Segmentation](09-zero-trust-and-segmentation.md). Whichever you use, limit what connected users can reach with [firewall](04-firewalls.md) rules.

### Connected but cannot reach: a checklist

| Step | Check | Tool |
|---|---|---|
| 1 | Did the handshake succeed? | `wg show`, client logs, gateway logs |
| 2 | Does the outer path work? UDP or the port may be blocked | Client log, `tcpdump` on the gateway |
| 3 | Does the client route send this destination to the tunnel? | `ip route get`, `route print` |
| 4 | Does the gateway forward packets? | `sysctl net.ipv4.ip_forward` |
| 5 | Is a firewall blocking the inner traffic? | Gateway rules |
| 6 | **Does the server have a route back to the client subnet?** | `ip route`, NAT rule |
| 7 | Does a name fail but the IP work? Then it is DNS | `nslookup`, split DNS config |
| 8 | Do small packets work and big ones fail? Then it is MTU | `ping -M do -s` |

Step 6 is the most often missed. The packet reaches the server, but the reply goes to the default gateway, which does not know 10.8.0.0/24.

## Common Mistakes

- **"VPN means encrypted."** GRE, L2TP alone and MPLS L3VPN are VPNs or tunnels with no encryption.
- **"A VPN makes me anonymous."** It hides your IP from sites and your browsing from your ISP. Logins and cookies still identify you, and the provider sees your traffic metadata.
- **"GRE uses port 47."** GRE has no ports. It is IP protocol 47. A firewall rule for TCP or UDP 47 does nothing.
- **"PPTP is fine for small offices."** It is broken. Do not use it anywhere.
- **"Connected means working."** The client only proves the handshake worked. Routes, return routes, firewalls and DNS can still fail.
- **Forgetting the return route.** The server must have a route back to the VPN client range, or the gateway must NAT.
- **Using TCP tunnels by default.** TCP in TCP can collapse under loss. Prefer UDP.
- **Ignoring MTU.** Ping works and big transfers hang. That usually means the tunnel overhead was not handled.
- **Using 192.168.1.0/24 for the office LAN.** It will clash with home networks.
- **Single password on a VPN gateway.** Always require MFA. The gateway faces the whole internet.
- **Thinking OpenVPN on TCP 443 looks like HTTPS.** Deep packet inspection (firewalls reading traffic patterns) can often still identify it.
- **Giving every VPN user the whole LAN.** A VPN is not access control. Add firewall rules.

## Interview Questions

**Q: What is a VPN and what does it not protect against?**

A: A VPN builds a tunnel across an untrusted network, so that devices or sites can communicate privately. A good one gives confidentiality, integrity and authentication. It does not stop malware or phishing, it does not make you anonymous, and it does not protect traffic after the tunnel ends at the gateway.

**Q: What is the difference between site-to-site and remote access VPN?**

A: Site-to-site joins two networks. Routers or firewalls build the tunnel, it stays up, and users need no software. Remote access joins one device to a network. The device runs a client, the user logs in, usually with MFA, and gets a tunnel IP address. IPsec is the usual choice for the first. SSL VPN, OpenVPN, WireGuard or IKEv2 are usual for the second.

**Q: Explain full tunnel and split tunnel. Which would you choose?**

A: In a full tunnel, the default route points to the VPN, so all traffic goes through the company. In a split tunnel, only company subnets use the VPN, and the rest goes out directly. Full tunnel gives more visibility but adds load and delay. Split tunnel is faster but the company cannot see direct traffic. I would pick by policy. A common middle ground is full tunnel with exceptions for trusted video services, plus endpoint security on the device.

**Q: Why is GRE often used with IPsec?**

A: GRE has no security, but it carries multicast and creates a real routed interface, so OSPF can run across it. Basic IPsec protects traffic but does not carry multicast. So GRE does the tunnelling and routing, and IPsec adds encryption and integrity. The overhead is the sum of the two, so I would lower the MTU and clamp the MSS.

**Q: How does WireGuard differ from OpenVPN?**

A: WireGuard runs in the kernel on Linux, uses UDP only, and has a fixed set of modern ciphers, so it is small, fast and hard to misconfigure. OpenVPN normally runs in user space, uses TLS for its control channel, offers many options, runs on UDP or TCP, and supports certificates with user logins. WireGuard has no built-in user management. OpenVPN is easier to fit into older networks, for example over TCP 443.

**Q: Why is PPTP considered insecure?**

A: It uses MS-CHAPv2 for login and RC4 through MPPE for encryption. MS-CHAPv2 can be reduced to cracking one DES key, which was shown publicly in 2012. An attacker who captures the login can then read the session. So PPTP should be removed. L2TP/IPsec is better but also legacy, and IKEv2, OpenVPN or WireGuard are the modern choices.

**Q: Is L2TP secure by itself?**

A: No. L2TP only tunnels PPP sessions and has no encryption. That is why it is paired with IPsec. On a firewall, you allow UDP 500, UDP 4500 and ESP. L2TP's UDP 1701 stays inside the protection.

**Q: Why do VPNs prefer UDP over TCP?**

A: When TCP runs inside TCP, both layers react to loss. The outer layer retransmits and slows down. The inner layer sees delay and retransmits too. This can collapse throughput, which is called TCP meltdown. With UDP, the outer layer just carries the packets and the inner TCP handles loss alone.

**Q: A user says the VPN is connected, but they cannot reach a file server. What do you check?**

A: First I confirm the handshake and whether traffic is counted both ways. Then I check that the client route sends the server's address into the tunnel. Then the gateway: IP forwarding and firewall rules. Then the return path, because the server must have a route back to the client range, or the gateway must NAT. Then I test by IP to rule out DNS, and with large pings to rule out MTU.

**Q: Web pages over the VPN hang, but ping works. Why?**

A: That points to MTU. Ping packets are small, and large packets plus the tunnel overhead exceed the path MTU. If the ICMP "fragmentation needed" message is blocked, the sender never reduces size. I test with `ping -M do -s` to find the working size, then reduce the tunnel MTU or clamp the TCP MSS.

**Q: Does a consumer VPN make you safe and anonymous?**

A: Not really. It moves trust from your ISP to the VPN provider. It hides your traffic from the local network and your IP from sites. But the provider can see your metadata, and sites can track you by logins and cookies. A no-logs claim is hard to verify. It is useful for an untrusted network or changing region, not for anonymity.

**Q: What is a DNS leak, and what is a kill switch?**

A: A DNS leak is when DNS queries skip the tunnel and go to the local resolver, which shows which sites you visit. You fix it by pushing DNS through the tunnel. A kill switch is a set of firewall rules that blocks all traffic if the tunnel drops, so nothing goes out unprotected.

## Quick Recap

- A VPN wraps whole packets inside new packets, usually encrypted, and a tunnel is really just a route to a virtual interface.
- Site-to-site joins networks through gateways. Remote access joins one device through a client. Full tunnel sends everything; split tunnel sends only chosen subnets.
- Current choices: IPsec IKEv2, SSL VPN, OpenVPN and WireGuard. L2TP/IPsec is legacy. PPTP is broken.
- GRE (IP protocol 47) and L2TP have no encryption. Pair them with IPsec.
- Tunnels prefer UDP because of TCP meltdown, and they reduce MTU, so plan for MTU and MSS.
- When a VPN is connected but failing, check routes, return routes, firewall, DNS and MTU, in that order.
- A consumer VPN moves trust to the provider. It does not give anonymity or stop malware.
