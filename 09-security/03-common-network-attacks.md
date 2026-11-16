# Common Network Attacks

**Level:** Intermediate | **Prerequisites:** [Network Security Basics](01-security-basics.md), [ARP](../03-ethernet-and-switching/05-arp.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [DNS Basics](../07-application-protocols/01-dns-basics.md)

## In One Line

Most network attacks abuse one of three things: protocols that believe whatever they are told, machines with limited resources, or people and settings that allow something weak.

## Why It Matters

The core protocols of the internet were designed for a small group of people who trusted each other. ARP, DNS, BGP and IP all accept messages with little or no proof of who sent them. Those protocols still carry almost all traffic today, so the weaknesses are still real.

Interviewers ask about attacks for two reasons:

- It shows you understand the protocol. To explain a SYN flood, you must know the TCP handshake. To explain ARP spoofing, you must know how a host finds a MAC address.
- It shows you can defend a network. The common question is not "what is a SYN flood?" but "your web server stops accepting connections, what do you check and what do you do?"

The rule for this chapter: for every attack, be able to say **how it works**, **what it breaks**, **how you detect it** and **how you stop it**.

> Only test systems you own or have written permission to test. Scanning or attacking other people's networks is illegal in many countries.

## Simple Explanation

Think of a network as the postal system of a city.

- **Reconnaissance:** a thief walks around the building and notes which doors and windows exist.
- **Spoofing:** someone writes a fake return address on an envelope.
- **Sniffing:** someone reads postcards as they pass through the sorting room.
- **Man-in-the-middle:** a dishonest postman opens each letter, reads it, maybe changes it, seals it and delivers it.
- **Denial of service:** someone sends so many fake parcels that the real mail cannot reach the door.
- **Replay:** someone copies a signed cheque request and sends it again next week.
- **Session hijacking:** someone steals your visitor badge, so the guard lets them in as you.
- **DNS poisoning:** someone changes a page in the phone book, so you call the wrong number.
- **BGP hijack:** someone changes the road signs, so all trucks drive to the wrong town.
- **Downgrade:** someone tricks you into using an old lock that is easy to pick.

**Where the analogy stops being true:** a real letter is slow and one person handles it. Network attacks run at machine speed, across the world, from thousands of devices at once. Also, encryption has no good postal equivalent. A strong encrypted message cannot be opened by the postman at all.

Every attack damages at least one of three goals, known as the **CIA triad**:

| Goal | Plain meaning | Attack examples |
|---|---|---|
| Confidentiality | Only the right people can read it | Sniffing, man-in-the-middle |
| Integrity | Nobody can change it unnoticed | DNS poisoning, man-in-the-middle |
| Availability | It works when you need it | DoS, DDoS, BGP hijack |

## How It Works

Each section below gives the idea, the steps and the defence. A full summary table comes at the end of this section.

### 1. Reconnaissance

**Reconnaissance** (recon) means collecting information about a target before attacking it. It is usually the first step.

**Passive recon** does not touch the target directly, so the target cannot see it:

- WHOIS records (who owns a domain or address block).
- Public DNS records.
- Certificate Transparency logs. These are public lists of every certificate issued by trusted authorities, and they often reveal hostnames such as `vpn.example.com`.
- Search engines for exposed devices, such as Shodan and Censys.
- Job adverts and staff profiles ("must know Cisco ASA" tells the attacker what you run).

**Active recon** sends packets to the target, so it can be logged:

| Technique | What it does | What it reveals |
|---|---|---|
| Ping sweep | Sends echo requests or ARP requests across a range | Which hosts are alive |
| Port scan | Sends packets to many ports | Which services are open |
| SYN scan | Sends SYN, reads the reply, never finishes the handshake | Open (SYN-ACK), closed (RST), filtered (no reply) |
| Banner grabbing | Connects and reads the first message from the service | Software name and version |
| OS fingerprinting | Studies small differences in TCP/IP replies | Likely operating system |
| Traceroute | Sends packets with rising TTL | Network path and devices |
| DNS zone transfer | Asks a DNS server for its full zone | Every hostname, if the server is misconfigured |

Scanning itself is covered in [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

**Defence:**

- Reduce what is exposed. Close unused ports and use a default-deny [firewall](04-firewalls.md).
- Hide version details in banners where you can.
- Allow zone transfers (AXFR) only to your secondary DNS servers.
- Turn off CDP and LLDP on ports that face users or outside networks. These protocols announce device details.
- Detect and rate limit scans with an [IDS or IPS](06-ids-and-ips.md).
- Scan yourself first, so you see what an attacker sees.

**Fuller truth:** you cannot stop passive recon. The goal is to make the information less useful, not to hide everything.

### 2. Sniffing

**Sniffing** means capturing packets that pass over a network and reading them. It is also called packet capture or eavesdropping. The same tools (Wireshark, tcpdump) are used by engineers every day. The attack is capturing traffic you have no right to read.

A network card normally ignores frames that are not addressed to it. In **promiscuous mode**, it accepts every frame it sees.

**Why a switch does not fully stop sniffing.** A hub repeated every frame to every port, so sniffing was trivial. A switch sends a frame only to the correct port, so an attacker must first get the traffic to come to them:

- **ARP spoofing:** pull traffic through the attacker's machine (section 3).
- **MAC flooding:** fill the switch's MAC table so it floods frames to all ports. See [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).
- **Mirror port:** an attacker with switch access configures a SPAN port, which copies traffic to another port.
- **Wi-Fi:** radio reaches everyone in range. See [Wi-Fi Security](../08-wireless-and-mobile/02-wifi-security.md).
- **Physical tap:** a small device placed on a cable.

**What sniffing can capture:** passwords in FTP, Telnet and plain HTTP, session cookies, and files sent in clear text. With encrypted traffic, the attacker still sees metadata: IP addresses, ports, packet sizes, timing, and often the server name in the TLS handshake.

**Defence:**

- Encrypt everything: HTTPS, SSH instead of Telnet, SFTP instead of FTP, a [VPN](07-vpn.md) on untrusted networks.
- Use Layer 2 protections (port security, Dynamic ARP Inspection).
- Limit who can log in to switches, and alert on new SPAN sessions.
- Lock wiring closets.

**Fuller truth:** sniffing is passive, so it is very hard to detect. Plan as if someone is listening, and rely on encryption.

### 3. Spoofing

**Spoofing** means faking an identity in a packet: a source IP, a MAC address, an ARP mapping or a DNS answer. Spoofing is usually a tool used inside a bigger attack, not the end goal.

#### IP spoofing

The sender writes a false source address in the IP header. Routers forward packets by looking at the destination, so they normally do not check the source.

```text
Real sender: 192.0.2.50

+-----------------------------+
| Src IP: 198.51.100.77  FAKE |
| Dst IP: 203.0.113.10        |
+-----------------------------+

Reply goes to 198.51.100.77. The real sender never sees it.
```

This creates a key limit. The attacker does not get the replies. So spoofing works well for one-way attacks (floods, reflection) and badly for anything that needs a conversation.

**Can you spoof a full TCP connection?** Normally no. The SYN-ACK goes to the real owner of the address, and the attacker cannot see the server's random 32-bit initial sequence number. A spoofed SYN can be sent, but the handshake cannot be finished, unless the attacker can also see the traffic on the path.

**Defence:**

- **Ingress filtering (BCP 38, RFC 2827):** an ISP drops packets from a customer whose source address does not belong to that customer. This stops spoofing near the source. It helps everyone else, not the network that deploys it, so it is still not universal.
- **Unicast Reverse Path Forwarding (uRPF):** the router checks that it has a route back to the source through the interface where the packet arrived.
- **Edge [ACLs](05-access-control-lists.md):** drop inbound packets that claim a private source address or one of your own addresses.

```text
! Cisco IOS: strict uRPF on an interface
interface GigabitEthernet0/0
 ip verify unicast source reachable-via rx
```

```bash
# Linux: strict reverse path filter (1 = strict, 2 = loose)
sysctl -w net.ipv4.conf.all.rp_filter=1
```

Use strict mode only where traffic is symmetric. If packets can arrive on a different link from the one used to reply, strict mode drops good traffic. Loose mode only checks that a route to the source exists somewhere.

#### MAC spoofing

An attacker changes the source MAC address of their network card. This is simple because the operating system lets you set any value.

Uses: bypassing MAC allow-lists, bypassing a paid Wi-Fi login that remembers MAC addresses, or taking over a port with a known MAC.

**Defence:**

- Do not treat a MAC address as proof of identity.
- Use **802.1X**, which makes a device prove itself with a certificate or password before the port opens.
- Use switch port security to limit MAC addresses per port.

#### ARP spoofing (ARP poisoning)

**ARP** finds the MAC address that belongs to an IP on the local network. It has no authentication, and hosts usually accept replies they never asked for.

**Analogy:** in an office, someone shouts "I am the manager, give me all the manager's mail". Everybody writes that down and believes it. In real offices people know faces. ARP has no faces.

The attack, using the network from the [Example](#example-1-finding-and-stopping-arp-spoofing):

1. Attacker sends a forged ARP reply to the victim: "192.168.10.1 is at my MAC".
2. Attacker sends a forged ARP reply to the gateway: "192.168.10.20 is at my MAC".
3. Both devices update their ARP caches.
4. Traffic in both directions now goes to the attacker.
5. Attacker repeats the forged replies every few seconds, so the cache stays poisoned.
6. Attacker forwards packets on to the real destination, so nothing looks broken.

This is what the forged reply to the victim contains:

| Field | Value | Meaning |
|---|---|---|
| Hardware type | 1 | Ethernet |
| Protocol type | 0x0800 | IPv4 |
| Hardware size | 6 | MAC is 6 bytes |
| Protocol size | 4 | IPv4 is 4 bytes |
| Opcode | 2 | ARP reply |
| Sender MAC | 00:1a:2b:3c:4d:66 | Attacker's real MAC |
| Sender IP | 192.168.10.1 | The lie: this is the gateway's IP |
| Target MAC | 00:1a:2b:3c:4d:20 | Victim |
| Target IP | 192.168.10.20 | Victim |

**Defence:**

- **Dynamic ARP Inspection (DAI):** the switch checks ARP messages against a trusted table and drops lies. It builds that table with DHCP snooping.
- **DHCP snooping:** the switch records which IP and MAC it saw on each port.
- **Static ARP entries:** workable for a few key devices, too much work at scale.
- **Encryption:** ARP spoofing still redirects traffic, but TLS or a VPN stops the attacker reading it.

ARP spoofing only works inside one Layer 2 segment. Small [VLANs](../03-ethernet-and-switching/06-vlans.md) reduce the reach.

#### DNS spoofing

**DNS spoofing** means sending a forged DNS answer, so a client gets the wrong IP address. An attacker on the same network can see the client's query and reply first. The client takes the first answer that matches.

DNS spoofing and DNS poisoning are often used as the same term. A more careful version:

| | DNS spoofing | DNS cache poisoning |
|---|---|---|
| Target | One client's query | The resolver's cache |
| Who is affected | One user | Every user of that resolver |
| Lasts | One lookup | Until the record's TTL ends |
| Attacker position | Usually on the path or local LAN | Can be remote (off-path) |

Section 10 covers poisoning and the defences.

#### Other kinds of spoofing

- **DHCP spoofing:** a fake DHCP server hands out the attacker's IP as the gateway. Stopped by DHCP snooping.
- **Email spoofing:** a fake sender address. Stopped by SPF, DKIM and DMARC, covered in [Email Protocols](../07-application-protocols/10-email-protocols.md).
- **IPv6 Neighbor Discovery spoofing:** the IPv6 version of ARP spoofing (see Going Deeper).

### 4. Man-in-the-Middle (MITM)

In a **man-in-the-middle** attack, the attacker sits between two parties and relays their traffic. Each side thinks it is talking directly to the other. The attacker can read, change or block messages. Newer writing also uses the term **on-path attack**.

**Analogy:** two people speak through a translator. A dishonest translator can change any sentence. Neither person can tell. Where it stops being true: with good encryption, the "translator" sees only random noise and cannot change a word without being caught.

```text
Normal path:

  Victim ------------------------------> Gateway ---> Internet
  192.168.10.20                          192.168.10.1

During the attack:

  Victim ----> Attacker ----> Gateway ---> Internet
         <---- 192.168.10.66 <----
               reads, may change,
               forwards
```

MITM needs two steps.

**Step 1: get in the path.** Common ways:

| Method | Layer | Stopped by |
|---|---|---|
| ARP spoofing | 2 | DAI, DHCP snooping |
| Rogue DHCP server | 2/7 | DHCP snooping |
| Evil twin (fake Wi-Fi network) | 1/2 | WPA3, WPA2/3-Enterprise, user caution |
| DNS spoofing or poisoning | 7 | DNSSEC, encrypted DNS |
| BGP hijack | 3 | RPKI, filtering |
| Malicious proxy or software on the device | 7 | Endpoint security |

**Step 2: deal with encryption.** Modern traffic uses TLS, so being in the path is not enough. The attacker must do one of these:

- Show a fake certificate and hope the user clicks through the warning.
- Hold a certificate the victim trusts (for example, a rogue root certificate installed on the device).
- Strip the encryption (see section 13).

**Defence:**

- **TLS with certificate checks:** the browser checks that a trusted authority issued the certificate for that exact name. See [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).
- **Never click through certificate warnings.**
- **HSTS:** tells the browser to use only HTTPS for a site.
- **Mutual TLS (mTLS):** both sides show certificates. Common between services.
- **SSH host keys:** SSH remembers each server's key and warns when it changes.
- **VPNs** on untrusted networks, and strong Wi-Fi security.

**Fuller truth:** some companies do a legal form of MITM. Their firewall or proxy decrypts TLS for inspection, using a root certificate installed on company laptops. The technique is the same. The difference is consent and ownership of the device.

### 5. Denial of Service: DoS and DDoS

A **Denial of Service (DoS)** attack makes a service unavailable to real users. It does not need to break in. It only needs to use up something limited.

A **Distributed DoS (DDoS)** attack does the same from many sources at once, often a **botnet**. A botnet is a group of hacked devices controlled by one attacker. The Mirai botnet (2016) took over cameras and routers that still had default passwords, and it was used against the DNS provider Dyn in October 2016, which made many large websites hard to reach.

**Why DDoS is harder than DoS.** With one source, you block one address. With 100,000 sources from all over the world, there is no single address to block, and bad traffic looks close to good traffic.

Every DDoS attack targets one of three limits:

| Class | What runs out | Measured in | Examples | Main defence |
|---|---|---|---|---|
| Volumetric | Link bandwidth | Bits per second (bps) | UDP flood, ICMP flood, amplification | Scrubbing service, anycast, CDN |
| Protocol (state exhaustion) | Connection tables in servers, firewalls, load balancers | Packets per second (pps) | SYN flood, ACK flood, fragment flood | SYN cookies, SYN proxy, stateless filters |
| Application (Layer 7) | CPU, memory, database | Requests per second (rps) | HTTP flood, Slowloris, Rapid Reset | WAF, rate limits, caching, bot checks |

**WAF** means web application firewall, a filter that understands HTTP.

Named attacks you should recognise:

| Attack | How it works | Status |
|---|---|---|
| SYN flood | Many SYNs, handshake never finished | Common (section 6) |
| UDP flood | Many UDP packets to random ports; the host checks each and may send ICMP errors | Common |
| ICMP flood | Many echo requests, using bandwidth and CPU | Common but simple |
| ACK or RST flood | Packets that match no connection, forcing state lookups | Common |
| Fragment flood | IP fragments that make the target hold memory for reassembly | Common |
| Amplification | Small spoofed request, large reply to the victim | Very common (section 7) |
| Slowloris | Opens many HTTP connections and sends headers very slowly, keeping all workers busy | Still works on some servers |
| HTTP flood | Many valid-looking requests, often for slow pages | Very common |
| HTTP/2 Rapid Reset | Opens and instantly cancels streams, so the server does work for nothing (CVE-2023-44487, 2023) | Patched, but a key lesson |
| DNS water torture | Queries for random names under a domain, such as `x7f2.example.com`, so caches miss and the authoritative server overloads | Common |
| Smurf | Spoofed ping to a directed broadcast address; every host replies to the victim | Historic: routers now drop directed broadcasts by default |
| Ping of death | ICMP fragments that reassemble to over 65,535 bytes and crash old systems | Historic |
| Teardrop | Overlapping fragments crash old reassembly code | Historic |

**Defence, by location:**

1. **On the server:** SYN cookies, connection limits, timeouts, rate limits per client.
2. **At your edge:** ACLs, stateless filters, dropping unused protocols and UDP ports you do not serve.
3. **Upstream:** this is the key point. If 100 Gbit/s hits your 10 Gbit/s link, the link is full before any of your devices can filter. You must filter before your link.
   - **Scrubbing service:** traffic is sent through a provider that removes bad packets and forwards the rest.
   - **Anycast:** one IP is announced from many places, so attack traffic is split among them. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).
   - **[CDN](../10-enterprise-and-datacenter/06-cdn.md):** caches content and absorbs traffic with huge capacity.
   - **RTBH (Remote Triggered Black Hole, RFC 5635):** you ask your ISP to drop all traffic to the attacked IP. This saves the rest of your network but takes that IP offline.
   - **BGP Flowspec (RFC 8955):** sends finer filter rules, such as "drop UDP port 123 to this IP", using BGP.

**Fuller truth:** there is no one fix. Large providers have reported attacks above 20 Tbit/s in 2025, and records keep rising. Few networks can absorb that alone.

### 6. SYN Flood

A **SYN flood** is a protocol attack on the TCP handshake.

**Analogy:** a restaurant takes phone bookings and holds a table for each one. An attacker makes hundreds of bookings with fake names. Tables stay empty but "reserved", and real guests are turned away. Where it stops being true: a restaurant might hold a table for hours, but a server gives up after about a minute. So the attacker has to keep sending.

**The problem:** when a server receives a SYN, it saves a small record and replies with SYN-ACK. This half-finished connection is in the SYN-RECEIVED state. It stays there until the final ACK arrives or a timer ends. The queue of these records has a fixed size.

```text
Attacker                     Server                     Spoofed IP
(fake sources)               203.0.113.10:443           (unused)
   |                             |                          |
   |-- SYN src=198.51.100.77 --->|                          |
   |                             | saves half-open entry    |
   |                             |-- SYN-ACK -------------->| lost
   |                             |   retries for ~1 minute  |
   |-- SYN src=198.51.100.78 --->|                          |
   |                             | saves half-open entry    |
   |                             |-- SYN-ACK -------------->| lost
   :                             :                          :
   |                        queue full: real SYNs dropped
```

**Why spoofed addresses work:** the SYN-ACK goes to the fake address. If a real host owns that address, it replies with RST and the entry is cleared. So attackers prefer addresses where nothing answers.

**The cheap cost for the attacker:** this attack needs very little bandwidth (see Example 2).

**Defence:**

- **SYN cookies:** the main defence. The server stores nothing for a new SYN. It puts the needed information, protected by a secret hash, inside its own initial sequence number. When the ACK returns, the server checks the number and only then creates the connection. A fake source never replies, so it costs the server no memory.

```text
Normal SYN handling:    SYN -> save state -> SYN-ACK -> wait
With SYN cookies:       SYN -> compute seq -> SYN-ACK -> forget

  seq = hash(src IP, src port, dst IP, dst port, secret, time)
        plus a small code for the MSS

  When ACK arrives: check (ack number - 1) against the hash.
  Valid -> create the connection now.
```

- **SYN proxy:** a firewall or load balancer finishes the handshake with the client, then opens the connection to the server. Cisco IOS calls one version TCP Intercept.
- **Bigger queue, shorter retries:** this buys time but does not solve the problem.
- **Upstream filtering** for very large floods.

```bash
# Linux: check and set
sysctl net.ipv4.tcp_syncookies          # 1 = use cookies when queue overflows
sysctl net.ipv4.tcp_max_syn_backlog     # half-open queue size
sysctl net.ipv4.tcp_synack_retries      # SYN-ACK retries, default 5
```

```powershell
# Windows: count half-open connections
netstat -an | findstr SYN_RECEIVED
```

Windows Server has built-in SYN attack protection that is on by default.

**Fuller truth:** SYN cookies have costs. Classic cookies have little room for TCP options, so only a few MSS values can be encoded. Linux can store some options in the TCP timestamp. For this reason, Linux uses cookies only when the queue overflows. RFC 4987 describes SYN flooding and its common defences.

### 7. Amplification and Reflection

**Reflection** means bouncing traffic off a third party. The attacker sends a request with the victim's address as the source. The third party, called a **reflector**, sends the reply to the victim.

**Amplification** adds a size gain: the reply is much bigger than the request.

**Analogy:** you phone a shop, give a neighbour's address, and ask for their huge catalogue to be mailed there. A short call creates a heavy parcel. Do this to 10,000 shops and the neighbour's letterbox is buried. Where it stops being true: the shops would notice a pattern, but servers on the internet just answer every packet.

```text
Attacker                 Open DNS resolver            Victim
192.0.2.50               198.51.100.53                203.0.113.10
   |                           |                           |
   |-- 64-byte query --------->|                           |
   |   src=203.0.113.10 (fake) |                           |
   |                           |-- 3,200-byte reply ------>|
   |                           |                           |
   Amplification factor = 3200 / 64 = 50 times
```

Three things must be true:

1. The attacker can spoof the source address.
2. The protocol uses UDP, which has no handshake to check the source.
3. A server replies with more data than it received.

**Why not TCP?** The three-way handshake proves the client can receive at the address it claims, so a spoofed source cannot get data sent. (Fuller truth: TCP can still be reflected in a small way, because the SYN-ACK itself goes to the victim. Middlebox-based TCP amplification has also been found by researchers. It is far less common than UDP.)

Well-known amplifiers. These factors come from the US government alert TA14-017A and are rough values:

| Protocol | UDP port | Rough factor | Reason |
|---|---|---|---|
| DNS | 53 | 28 to 54 | ANY and large signed answers |
| NTP | 123 | 556 | Old `monlist` command returned a list of recent clients |
| SSDP | 1900 | 30 | Home devices answering discovery on the internet |
| CharGEN | 19 | 358 | Old test service |
| CLDAP | 389 | 56 to 70 | Exposed directory servers |
| Memcached | 11211 | 10,000 to 51,000 | Cache servers exposed on UDP |

The February 2018 attack on GitHub reached 1.35 Tbit/s using memcached.

**Defence:**

- **Stop spoofing:** BCP 38 removes the root cause, but only when networks deploy it.
- **Do not run open reflectors:** do not offer a DNS resolver to the whole internet. Restrict `monlist`-style queries. Do not expose memcached.
- **Rate limit responses:** DNS Response Rate Limiting (RRL).
- **Minimal answers:** RFC 8482 lets DNS servers avoid large answers to ANY queries.
- **Victim side:** upstream scrubbing, or filtering on source port. For example, if you do not use NTP from the outside, drop UDP source port 123 at your provider's edge.
- **New protocol design:** QUIC limits a server to sending 3 times the data it received until the client's address is proved (RFC 9000).

```text
# BIND DNS server: allow recursion only for your own network
options {
    allow-recursion { 192.168.10.0/24; };
    rate-limit { responses-per-second 5; };
};

# ntpd: block status queries from strangers
restrict default kod nomodify notrap nopeer noquery
disable monitor
```

Modern software has safer defaults. `monlist` was removed from ntpd in version 4.2.7p26, and memcached has turned UDP off by default since 1.5.6. Old unpatched systems are the problem.

### 8. Session Hijacking

A **session** is an ongoing conversation between a client and a server. After you log in, the site gives your browser a **session token**, usually in a cookie. The token is proof that you already logged in. **Session hijacking** means stealing or guessing that token, so the attacker becomes you without needing your password.

**Analogy:** at a concert, you show your ticket once, then get a wristband. Anyone wearing the wristband walks in. Steal the wristband and you skip the ticket check. Where it stops being true: a wristband is hard to copy, but a token is just text.

Ways to get a token:

| Method | How | Defence |
|---|---|---|
| Sniffing | Cookie sent over plain HTTP on open Wi-Fi (the 2010 Firesheep tool made this one click) | HTTPS everywhere, `Secure` cookie flag |
| Cross-site scripting (XSS) | Script injected into a page reads the cookie | `HttpOnly` flag, input handling, Content Security Policy |
| Malware on the device | Steals stored cookies | Endpoint security, short session life |
| Session fixation | Attacker sets a known session ID before the victim logs in | New session ID after login |
| Guessing | Weak, predictable tokens | Long random tokens |

**TCP session hijacking** is an older, lower-layer version. The attacker sends a spoofed packet with the right IP addresses, ports and sequence numbers, so it is accepted as part of the victim's connection. Kevin Mitnick's well-known 1994 attack used predictable sequence numbers. Modern systems use random initial sequence numbers, so a blind attacker cannot guess them. An attacker on the path can still read them, which is why encryption matters. TLS rejects injected data, because the data fails the integrity check.

**TCP reset injection** is a simple form: a forged RST closes a connection. RFC 5961 made this harder, because a RST must match the exact next sequence number, not just fall inside the window. BGP sessions are a classic target, so they use TCP-AO (RFC 5925) or the older TCP MD5 option.

**Defence summary:**

- HTTPS on all pages.
- Cookie flags: `Secure`, `HttpOnly`, `SameSite`.
- New session ID on login and on privilege change.
- Short lifetimes and idle timeouts.
- Ask for the password or a second factor again before sensitive actions.
- Alert when a session suddenly changes country or device.

```text
Set-Cookie: session=7f3a9c21e4b8d60a; Secure; HttpOnly; SameSite=Lax; Path=/
```

### 9. Replay Attack

In a **replay attack**, the attacker captures a valid message and sends it again later. The attacker does not need to read or decrypt it. The message is accepted because it was genuine the first time.

**Analogy:** you record the sound of a person saying the password at a door, then play the recording. The door hears the right voice. Where it stops being true: a door cannot tell old from new, but a good protocol adds something fresh each time, so the old recording no longer fits.

```text
Client ---- "pay 100, signed" ----> Server       (real, accepted)
              |
              | attacker records a copy
              v
Attacker -- "pay 100, signed" ----> Server       (same bytes, accepted?)
```

A signature proves who wrote a message. It does not prove when. So a defence must add **freshness**:

| Method | How it works | Example |
|---|---|---|
| Nonce | Server gives a random value used once; the reply must include it | HTTP Digest, TLS random values, CHAP |
| Timestamp | Message carries the time; old ones are refused | Kerberos (commonly 5 minutes clock skew allowed) |
| Sequence number | Each message has a rising number; repeats are refused | IPsec ESP, TLS records |
| Session keys | A new key per session, so old captures fail | TLS, SSH, WPA2 |
| One-time codes | Code works once or for 30 seconds | TOTP apps |

[IPsec](08-ipsec.md) uses an **anti-replay window**. Each ESP packet has a sequence number. The receiver tracks a sliding window of numbers already seen, commonly 64, and drops repeats and packets that are too old. Window size depends on the vendor.

**Fuller truth:** TLS 1.3 has an optional fast start called 0-RTT, where the client sends data in its first flight. RFC 8446 warns that this early data can be replayed. Servers should accept it only for requests that are safe to repeat, such as a simple GET.

**Replay vs relay.** In a replay, the message is sent later. In a **relay**, the attacker passes a live login challenge to the real server and sends the answer back. NTLM relay is a famous example. A nonce stops replay but does not stop relay, because the attacker uses the live nonce. Relay needs a different fix, such as binding the login to the channel or using SMB signing.

### 10. DNS Cache Poisoning

A **DNS resolver** asks other servers on your behalf and caches the answers. **Cache poisoning** puts a false record in that cache, so every user of the resolver gets the wrong address until the record's TTL ends.

**Analogy:** someone swaps a page in the phone book at the library. Everyone who looks up that name after that dials the wrong number. Where it stops being true: a phone book is fixed, but a DNS record expires when its TTL ends.

DNS mostly uses UDP. A resolver accepts a reply if it matches the query:

- Same 16-bit transaction ID.
- Same source port (the reply comes to the port the query came from).
- Same question name and type.
- It comes from the server that was asked. (That IP can be spoofed.)
- It arrives first.

So an off-path attacker races the real server:

```text
Client      Resolver 10.0.0.53      Real server      Attacker
  |              |                       |               |
  |-- www? ----->|                       |               |
  |              |-- query (ID, port) -->|               |
  |              |<==== many forged replies, guessing ===|
  |              |      www = 203.0.113.66               |
  |              |<-- real reply: too late, ignored      |
  |<-- 203.0.113.66                                      |
  |
  Cached until TTL ends. Every user now gets the fake IP.
```

**The Kaminsky attack (2008).** Before this, a failed guess meant waiting for the cached record to expire. Dan Kaminsky found that an attacker could ask for random names that are not in the cache: `a1.example.com`, `a2.example.com`, and so on. Each name is a new race. A forged reply also carries an authority record that says "for all of example.com, use my name server". One win takes over the whole domain.

**Defence:**

- **Source port randomisation:** the main fix after 2008. The attacker must guess the port as well as the ID. 65,536 IDs multiplied by about 30,000 ports gives about 2 billion combinations.
- **DNSSEC:** zone owners sign records. A validating resolver rejects anything not correctly signed. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).
- **Bailiwick checks:** the resolver ignores extra records for names the answering server has no authority over.
- **0x20 encoding:** some resolvers randomise the letter case in the query, like `wWw.eXaMpLe.CoM`, and require the same case in the reply.
- **TCP fallback and DNS cookies:** make forging harder.
- **DoT (RFC 7858) and DoH (RFC 8484):** these encrypt client-to-resolver traffic, so they stop spoofing on that link. They do not protect the resolver's own queries to other servers, and they do not prove that the record is true. DNSSEC does that.

**Related: DNS hijacking.** Here nothing is forged on the wire. The attacker changes the real settings, such as a home router's DNS server, or the domain's name servers at the registrar. Defend with strong login, multi-factor authentication and a registrar lock.

### 11. BGP Hijack

**BGP** is the protocol that networks use to tell each other which addresses they can reach. By default, a router believes what a neighbour announces, and the longest (most specific) matching prefix wins. See [BGP](../05-routing/09-bgp.md).

A **BGP hijack** happens when a network announces addresses it does not own, so traffic goes to it instead of the true owner.

**Analogy:** someone puts up new road signs saying that the fastest way to the airport is through their street. All drivers follow them. Where it stops being true: drivers can see that a road looks wrong, but routers have no such sense. A router just follows the most specific sign.

```text
Before the hijack:

  AS 64512 (real owner) announces 198.51.100.0/24
  AS 64513 (ISP) table:
     198.51.100.0/24      via AS 64512

After the hijack:

  AS 64514 (attacker) announces two more specific prefixes
  AS 64513 (ISP) table:
     198.51.100.0/24      via AS 64512
     198.51.100.0/25      via AS 64514   <-- longer match, wins
     198.51.100.128/25    via AS 64514   <-- longer match, wins
```

**Note on the example:** documentation ranges are only /24, so this example uses /25. On the real internet, most networks drop routes longer than /24. So a real attacker would announce a /24 from inside a shorter prefix such as a /22.

**Real cases:**

- **Pakistan Telecom and YouTube (2008):** Pakistan ordered YouTube blocked and an ISP there announced 208.65.153.0/24, more specific than YouTube's 208.65.152.0/22. The route leaked upstream, and YouTube was down for much of the world for about two hours.
- **Amazon Route 53 (2018):** attackers hijacked DNS server addresses, then answered queries for a cryptocurrency site with their own IP. The fake site had an invalid certificate. Users who clicked through the warning lost funds. This shows how attacks chain together.

**Hijack vs route leak.** A hijack announces addresses that are not yours. A **route leak** (RFC 7908) passes on routes that should have been kept private, usually by accident. Both send traffic the wrong way.

**Defence:**

- **Prefix filters:** ISPs accept only prefixes the customer is allowed to announce, often built from Internet Routing Registry data.
- **RPKI:** owners publish a signed record called a **ROA** (Route Origin Authorization): "AS 64512 may announce 198.51.100.0/24, maximum length /24".
- **Route Origin Validation (ROV, RFC 6811):** routers compare each route to the ROAs and mark it valid, invalid or not found. Then policy drops invalid routes.
- **Max-prefix limits:** close a session if a peer sends far too many routes.
- **BGP roles (RFC 9234):** help stop route leaks.
- **Session protection:** TCP-AO or MD5, and GTSM (RFC 5082), which checks the TTL.
- **Monitoring:** alert when your prefix appears from a new origin.

```text
! Cisco IOS: drop RPKI invalid routes (syntax varies by version)
router bgp 64513
 bgp rpki server tcp 192.168.100.5 port 3323 refresh 600
 neighbor 192.0.2.1 route-map RPKI-IN in
!
route-map RPKI-IN deny 10
 match rpki invalid
route-map RPKI-IN permit 20
```

**Fuller truth:** ROV only checks the origin AS. An attacker can fake the path by placing the real owner's AS at the end. **BGPsec** (RFC 8205) signs the path, but almost nobody has deployed it. **ASPA** is a newer method that checks provider relationships, and its standard and use are still growing.

### 12. Sniffing, Spoofing and MITM: How They Differ

These three are often mixed up:

| | Sniffing | Spoofing | MITM |
|---|---|---|---|
| Action | Listens | Lies about identity | Sits between and relays |
| Sends packets? | No (passive) | Yes | Yes |
| Can change data? | No | No, it only adds new packets | Yes, if not protected |
| Main fix | Encryption | Authentication | Encryption and authentication together |

### 13. Downgrade Attack

In a **downgrade attack**, the attacker makes two parties use an older, weaker version of a protocol or a weaker cipher, then attacks that weakness. Both sides support the strong option, but the attacker interferes with the negotiation.

**Analogy:** two people can speak in a private code, but a person in the room keeps shouting that the code is broken, so they fall back to plain language. Where it stops being true: in modern protocols, both sides check at the end that they heard the same negotiation, so this interference gets detected.

Well-known examples:

| Attack | Year | What is forced | Lesson |
|---|---|---|---|
| TLS stripping | 2009 | HTTPS down to plain HTTP | Use HSTS |
| POODLE | 2014 | Browser fallback to SSL 3.0 | Disable SSL 3.0 |
| FREAK | 2015 | Weak export-grade RSA keys | Remove export ciphers |
| Logjam | 2015 | Weak 512-bit Diffie-Hellman | Use strong groups or ECDHE |
| DROWN | 2016 | Uses SSLv2 support to attack TLS | Disable SSLv2 everywhere |
| Dragonblood | 2019 | WPA3 transition mode down to WPA2 | Avoid transition mode when possible |
| Terrapin | 2023 | Removes SSH handshake messages, weakening extensions | Patch SSH clients and servers |

**TLS stripping** is the classic case. Most people type `example.com`, and the first request is plain HTTP. An on-path attacker keeps the victim on HTTP while using HTTPS to the server.

```text
Browser <---- plain HTTP ----> Attacker <---- HTTPS ----> Server
          attacker reads                 server sees a
          everything                     normal secure client
```

**STARTTLS stripping** is similar. Mail servers upgrade to TLS with a plain command. An attacker can delete that offer, so mail flows without encryption. MTA-STS (RFC 8461) tells senders that TLS is required.

**Defence:**

- **Turn off old versions:** SSL 2.0, SSL 3.0, TLS 1.0 and TLS 1.1 (RFC 8996 deprecates TLS 1.0 and 1.1). Keep TLS 1.2 with strong ciphers, and TLS 1.3.
- **HSTS (RFC 6797):** the browser refuses plain HTTP for that site. The **preload list** ships inside browsers, which also protects the first visit.
- **Authenticated negotiation:** TLS signs or checks the handshake, so tampering fails. TLS 1.3 also puts a special value in the server's random field, so a client that supports 1.3 can detect being pushed down to an older version.
- **Fail closed:** do not retry with weaker settings when a connection fails.
- **Other protocols:** `ip ssh version 2` on Cisco, SMB signing on, NTLMv1 off.

```text
# nginx: allow only modern TLS and send HSTS
ssl_protocols TLSv1.2 TLSv1.3;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Summary Table

| Attack | Layer | How it works | Breaks | Main defence |
|---|---|---|---|---|
| Reconnaissance | Any | Collects facts, scans ports | Prepares others | Reduce exposure, firewall, IDS |
| Sniffing | 2 | Captures traffic | Confidentiality | Encryption, Layer 2 protection |
| IP spoofing | 3 | Fake source IP | Trust in addresses | BCP 38, uRPF, ACLs |
| MAC spoofing | 2 | Fake source MAC | Port or Wi-Fi access control | 802.1X, port security |
| ARP spoofing | 2 | Fake IP-to-MAC mapping | Traffic path | DAI, DHCP snooping |
| DNS spoofing | 7 | Forged answer to a client | Name lookups | DNSSEC, DoT/DoH |
| MITM | 2-7 | Relays and may change traffic | Confidentiality, integrity | TLS, mTLS, VPN, HSTS |
| SYN flood | 4 | Fills half-open queue | Availability | SYN cookies, SYN proxy |
| Volumetric DDoS | 3-4 | Fills the link | Availability | Scrubbing, anycast, CDN |
| Application DDoS | 7 | Costly requests | Availability | WAF, rate limits, caching |
| Amplification | 3-7 | Spoofed small query, large reply | Availability | BCP 38, closed reflectors, RRL |
| Session hijacking | 4-7 | Steals or forges a session | Authentication | HTTPS, cookie flags, rotation |
| Replay | Any | Resends a captured message | Authentication, integrity | Nonce, timestamp, sequence number |
| DNS cache poisoning | 7 | False record in resolver | Integrity | Random ports, DNSSEC |
| BGP hijack | 3 | False route announcement | Availability, integrity | RPKI, ROV, prefix filters |
| Downgrade | 6-7 | Forces weak version or cipher | Confidentiality | Disable old versions, HSTS |

## Example

### Example 1: Finding and Stopping ARP Spoofing

**The network**

```text
                 +------------------+
                 | Gateway          |
                 | 192.168.10.1     |
                 | 00:1a:2b:3c:4d:01|
                 +--------+---------+
                          | Gi0/1 (trusted uplink)
                 +--------+---------+
                 | Access switch    |
                 +---+----------+---+
               Gi0/5 |          | Gi0/10
       +-------------+--+    +--+-------------+
       | Victim         |    | Attacker       |
       | 192.168.10.20  |    | 192.168.10.66  |
       | ...:4d:20      |    | ...:4d:66      |
       +----------------+    +----------------+
```

**Symptom:** a user says the network feels slow and a website showed a certificate warning.

**Step 1: check the ARP table.** A healthy Linux host has one distinct MAC for each IP:

```bash
# Linux, on the victim, healthy state
$ ip neigh show
192.168.10.1 dev eth0 lladdr 00:1a:2b:3c:4d:01 REACHABLE
192.168.10.66 dev eth0 lladdr 00:1a:2b:3c:4d:66 STALE
```

During the attack, the gateway has the same MAC as another host. That is the red flag:

```bash
# Linux, on the victim, during the attack
$ ip neigh show
192.168.10.1 dev eth0 lladdr 00:1a:2b:3c:4d:66 REACHABLE
192.168.10.66 dev eth0 lladdr 00:1a:2b:3c:4d:66 STALE
```

```powershell
# Windows, on the same victim
PS> arp -a

Interface: 192.168.10.20 --- 0x7
  Internet Address      Physical Address      Type
  192.168.10.1          00-1a-2b-3c-4d-66     dynamic
  192.168.10.66         00-1a-2b-3c-4d-66     dynamic
```

**Step 2: capture the ARP traffic.** Unrequested replies repeat every few seconds:

```bash
$ sudo tcpdump -n -e -i eth0 arp
10:15:02.114233 00:1a:2b:3c:4d:66 > 00:1a:2b:3c:4d:20, ethertype ARP (0x0806), length 42: Reply 192.168.10.1 is-at 00:1a:2b:3c:4d:66, length 28
10:15:04.114871 00:1a:2b:3c:4d:66 > 00:1a:2b:3c:4d:20, ethertype ARP (0x0806), length 42: Reply 192.168.10.1 is-at 00:1a:2b:3c:4d:66, length 28
```

**Step 3: find the port.** On the switch, look up the MAC:

```text
Switch# show mac address-table address 001a.2b3c.4d66
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    001a.2b3c.4d66    DYNAMIC     Gi0/10
```

**Step 4: stop it.** Turn on DHCP snooping and DAI (Cisco IOS):

```text
ip dhcp snooping
ip dhcp snooping vlan 10
ip arp inspection vlan 10
!
interface GigabitEthernet0/1
 description Uplink to gateway
 ip dhcp snooping trust
 ip arp inspection trust
```

Hosts with static IPs have no DHCP binding, so DAI would drop their ARP. They need an ARP ACL or a static binding.

**Step 5: confirm.** The switch now drops the forged reply and logs it:

```text
%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Res) on Gi0/10, vlan 10.
([001a.2b3c.4d66/192.168.10.1/001a.2b3c.4d20/192.168.10.20/10:21:37 UTC
Fri Oct 30 2026])
```

The log reads as: attacker MAC, claimed IP, target MAC, target IP. After the ARP entry refreshes, the victim sees the true gateway MAC again.

### Example 2: A SYN Flood on a Web Server

**Symptom:** the web server at 203.0.113.10 is up, and SSH works, but new HTTPS connections time out. CPU is low.

**Step 1: count half-open connections.**

```bash
$ ss -tn state syn-recv sport = :443 | head -4
Recv-Q  Send-Q  Local Address:Port     Peer Address:Port
0       0       203.0.113.10:443       198.51.100.77:41822
0       0       203.0.113.10:443       192.0.2.141:9913
0       0       203.0.113.10:443       198.51.100.203:55107

$ ss -tn state syn-recv sport = :443 | tail -n +2 | wc -l
4096
```

Thousands of half-open entries from random sources is the sign of a SYN flood. A normal server has a handful.

**Step 2: look at the packets.** All have only the SYN flag, and no handshake finishes:

```bash
$ sudo tcpdump -n -c 3 'tcp[tcpflags] == tcp-syn and dst port 443'
11:02:41.100213 IP 198.51.100.77.41822 > 203.0.113.10.443: Flags [S], seq 2215907311, win 1024, length 0
11:02:41.100248 IP 192.0.2.141.9913 > 203.0.113.10.443: Flags [S], seq 871220459, win 1024, length 0
11:02:41.100271 IP 198.51.100.203.55107 > 203.0.113.10.443: Flags [S], seq 3390012876, win 1024, length 0
```

**Step 3: check the kernel.**

```bash
$ sudo dmesg | grep -i 'syn flooding'
TCP: request_sock_TCP: Possible SYN flooding on port 443. Sending cookies.  Check SNMP counters.

$ nstat -az TcpExtSyncookiesSent
#kernel
TcpExtSyncookiesSent            584213             0.0
```

The exact log text varies by kernel version.

**Step 4: do the maths to see why this is cheap.**

```text
Queue size:                       4096 entries
Time one entry stays:             about 63 s
                                  (1+2+4+8+16+32 with 5 retries)
SYNs needed to keep it full:      4096 / 63 = about 65 per second
Size of one SYN on the wire:      about 60 bytes
Bandwidth needed:                 65 x 60 x 8 = about 31 kbit/s
```

That is slower than an old dial-up modem. The attack costs almost nothing, which is why it targets state and not bandwidth. This is a rough estimate. Real kernels drop old entries sooner when the queue fills, so a real attack needs a higher rate.

**Step 5: respond.**

1. Confirm `net.ipv4.tcp_syncookies = 1`.
2. If the rate is high, enable SYN proxy on the firewall or load balancer.
3. If the link is full, call your ISP or scrubbing provider.
4. Keep the capture for a report.

### Example 3: Amplification Maths

An attacker has a 100 Mbit/s uplink and uses reflectors.

```text
DNS, factor 50:        100 Mbit/s x 50  =   5 Gbit/s at the victim
NTP monlist, 556:      100 Mbit/s x 556 =  55.6 Gbit/s at the victim
```

The source IP seen by the victim is the reflector, such as 198.51.100.53, not the attacker. Blocking reflectors one at a time does not scale, so you filter by pattern upstream, such as "UDP source port 123".

### Example 4: A Quick Audit of Your Own Domain

Check for downgrade risk and DNS validation on a system you own.

```bash
# Which TLS versions does the server allow?
$ nmap --script ssl-enum-ciphers -p 443 www.example.com
PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_128_GCM_SHA256 (ecdh_x25519) - A
|_  least strength: A

# Is HSTS on?
$ curl -sI https://www.example.com | grep -i strict-transport
strict-transport-security: max-age=31536000; includeSubDomains

# Did the resolver validate DNSSEC? Look for the "ad" flag.
$ dig +dnssec www.example.com A | grep flags
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
; EDNS: version: 0, flags: do; udp: 1232
```

Good result: no TLS 1.0 or 1.1, HSTS present, `ad` flag set.

## Going Deeper

### How attacks chain together

Real attacks are not one technique. A typical path looks like this:

```text
Recon --> Get in the path --> Remove protection --> Steal --> Reuse
nmap      ARP spoof           TLS strip             cookie    hijack
          or evil twin        or fake cert                    session
```

This is why defence has layers. Breaking any one link stops the chain. DAI stops the ARP spoof, HSTS stops the strip, and a `Secure` cookie protects the token.

### Spotting attacks in practice

| Attack | What you would see |
|---|---|
| Port scan | One source hits many ports; many SYNs with no full handshake |
| ARP spoofing | One MAC for many IPs, gateway MAC changes, unrequested replies |
| SYN flood | Many SYN-RECEIVED entries, `SyncookiesSent` rising, CPU fine |
| Amplification | Large UDP replies from port 53, 123 or 11211 that you never asked for |
| HTTP flood | Rising requests per second, slow pages, many IPs with same user agent |
| DNS poisoning | Resolver answer differs from the authoritative server |
| DNS water torture | Many NXDOMAIN answers for random names |
| BGP hijack | Traceroute changes suddenly, prefix alert from a monitor |
| Session hijack | Same session from two countries or devices |
| Downgrade | Logs showing old TLS versions from clients that support new ones |

Flow data and alerts are covered in [Network Monitoring and Observability](../12-troubleshooting-and-tools/11-network-monitoring.md).

### IPv6 does not fix local attacks

IPv6 replaces ARP with **Neighbor Discovery Protocol (NDP)**, which uses ICMPv6. NDP is also unauthenticated by default, so similar attacks exist:

- **Neighbor spoofing:** forged Neighbor Advertisements, the same idea as ARP spoofing.
- **Rogue Router Advertisement:** a host announces itself as the default router. Devices believe it.
- **Rogue DHCPv6 server.**

Defences are **RA Guard** (RFC 6105), DHCPv6 Guard, and IPv6 snooping on switches. SEND (RFC 3971) adds cryptography to NDP, but it is rarely deployed. Note that many "IPv4-only" networks are exposed, because hosts run IPv6 by default.

### Layer 2 attacks owned by another chapter

[Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md) covers MAC flooding, VLAN hopping, DHCP starvation and STP root takeover. Interviewers often ask them next to ARP spoofing. Know each in one line.

### Other attacks interviewers mention

| Attack | One-line explanation | Defence |
|---|---|---|
| Brute force | Trying many passwords on one account (SSH, RDP) | MFA, keys, lockout, rate limits |
| Password spraying | One common password across many accounts | MFA, banned password lists |
| Credential stuffing | Using leaked passwords on other sites | MFA, breach checks |
| DNS tunnelling | Hiding data inside DNS queries, to leak data or control malware | Inspect DNS, block unknown resolvers |
| ICMP tunnelling | Hiding data inside ping payloads | Limit ICMP size and rate |
| IP fragmentation evasion | Splitting an attack so that a filter misses the pattern | Reassemble before inspecting |
| ICMP redirect abuse | Fake redirect changes a host's route | Ignore ICMP redirects on hosts |
| Typosquatting | Registering `exarnple.com` to catch typos | Register variants, user training |
| Pass-the-hash | Reusing a stolen password hash, with no need to crack it | Disable NTLM where possible, credential protection |

### Why some attacks persist

Fixing a protocol is slow because millions of devices must change, and the two ends must both be upgraded. Deployment also has poor incentives: BCP 38 protects other networks, not the one paying for it. Knowing this explains why a 1990s attack such as IP spoofing still drives large attacks in 2026.

## Common Mistakes

- **"HTTPS stops all MITM."** It helps only if the certificate is checked. If users click through a warning, or a rogue root is installed, the attacker can still read traffic.
- **"Switches prevent sniffing."** They reduce it. ARP spoofing, MAC flooding and SPAN ports bring the traffic back to an attacker.
- **"MAC filtering is real security."** MAC addresses are sent in clear and are easy to change.
- **"A firewall stops DDoS."** A stateful firewall can be the first thing to fall, because its state table is a target too. A full link cannot be fixed by anything behind it.
- **"DDoS is only about bandwidth."** A SYN flood or Slowloris can take down a server with very little bandwidth.
- **"SYN cookies stop every DDoS."** They only fix half-open queue exhaustion. They do nothing for a full link or an HTTP flood.
- **"Reflection and amplification are the same."** Reflection hides the source. Amplification adds the size gain. Most attacks use both, but they are separate ideas.
- **"DNSSEC encrypts DNS."** It signs records, so it gives integrity. DoT and DoH give privacy. They solve different problems.
- **"Replay requires breaking encryption."** The attacker just resends the bytes without reading them.
- **"Blocking ping hides you."** Port scans still work without ICMP.
- **"RPKI stops all BGP hijacks."** It stops wrong-origin announcements only where networks enforce it. A forged path can still pass.
- **"IPv6 removes ARP attacks."** ARP is gone, but NDP has the same trust problem.

## Interview Questions

**Q: What is the difference between DoS and DDoS?**
A: Both make a service unavailable. A DoS comes from one source, so blocking that source can stop it. A DDoS comes from many sources, usually a botnet, so there is no single address to block and the traffic mixes with real users. Defending a DDoS usually needs help upstream, such as a scrubbing service or CDN.

**Q: Explain a SYN flood and how to stop it.**
A: The attacker sends many SYN packets, often with spoofed sources, and never finishes the handshake. The server keeps a half-open entry for each one, so the queue fills and real clients are refused. The main defence is SYN cookies: the server stores nothing and encodes the state in its sequence number. A SYN proxy on a firewall or load balancer also works, and very large floods need upstream filtering.

**Q: How does ARP spoofing work and how do you prevent it?**
A: ARP has no authentication, so a host accepts forged replies. An attacker tells the victim that the gateway's IP has the attacker's MAC, and tells the gateway the same about the victim. Traffic then flows through the attacker. On a managed switch, I enable DHCP snooping and Dynamic ARP Inspection, which drops ARP that does not match the snooping table. Encryption protects the data even if it happens.

**Q: What is a DNS amplification attack?**
A: The attacker sends small DNS queries to open resolvers with the victim's IP as the source. The resolvers send much larger replies to the victim, often 28 to 54 times the size. The fixes are closing open resolvers, rate limiting responses, filtering spoofed traffic with BCP 38, and scrubbing at the victim's upstream.

**Q: How is DNS cache poisoning different from DNS spoofing?**
A: Spoofing gives one client a fake answer. Poisoning puts a fake record in a resolver's cache, so every user of the resolver gets it until the TTL ends. Source port randomisation makes guessing much harder, and DNSSEC lets the resolver reject unsigned or wrong records.

**Q: Can you spoof a source IP on a TCP connection?**
A: You can send the packet, but you cannot normally finish the handshake. The SYN-ACK goes to the real address and you cannot see the server's random sequence number. That is why spoofing suits floods and UDP reflection, and not full conversations, unless the attacker is on the path.

**Q: What is a replay attack and how do protocols stop it?**
A: The attacker captures a valid message and sends it again later, with no need to decrypt it. Protocols stop this by adding freshness: a nonce, a timestamp, a sequence number or a per-session key. For example, IPsec ESP uses sequence numbers with an anti-replay window, and Kerberos uses timestamps.

**Q: How can someone hijack a session even when my password is strong?**
A: After login, the session token is the proof of who you are. If the attacker steals it through cross-site scripting, malware or an unencrypted connection, they skip the password. I would use HTTPS everywhere, set `Secure`, `HttpOnly` and `SameSite` on cookies, rotate the ID at login, keep lifetimes short, and ask for a second factor on sensitive actions.

**Q: What is a BGP hijack and how do we defend against it?**
A: A network announces prefixes it does not own, and because BGP trusts neighbours and prefers the most specific route, traffic is pulled to it. An example is the 2008 YouTube outage caused by Pakistan Telecom's more specific /24. Defences are prefix filters, RPKI with route origin validation, max-prefix limits and monitoring. RPKI checks only the origin, so it does not stop every case.

**Q: What is a downgrade attack? Give an example.**
A: The attacker forces two sides to use a weaker protocol or cipher than both support. TLS stripping keeps the victim on plain HTTP, and POODLE used fallback to SSL 3.0. The defence is to disable old versions, use HSTS with preload, and rely on TLS's authenticated handshake, which detects tampering.

**Q: Your server is reachable by SSH, but web users time out, and CPU is low. What do you check?**
A: I suspect a state problem, such as a SYN flood. I run `ss -tn state syn-recv` to count half-open connections, use tcpdump to see whether they are SYNs from random sources with no handshake, and check the kernel log and SYN cookie counters. Then I confirm SYN cookies are on, use a SYN proxy, and contact my provider if the link itself is full.

**Q: Why does a 100 Gbit/s attack not get fixed by your own firewall?**
A: Because the link is full before traffic reaches the firewall. If my link is 10 Gbit/s, real packets are lost at the ISP's side. Filtering has to happen upstream, through a scrubbing service, anycast, a CDN, or RTBH and Flowspec with my ISP.

## Quick Recap

- Attacks exploit three things: protocols that trust without proof, limited resources, and weak settings or people.
- Reconnaissance and sniffing collect information. Spoofing fakes identity. MITM uses a path position to read or change traffic.
- DoS and DDoS target bandwidth, connection state or application resources, and big floods must be filtered upstream.
- SYN cookies protect the half-open queue. BCP 38 and closed reflectors fight amplification.
- Session hijacking steals proof of login, while replay resends a valid message. Freshness and cookie flags are the fixes.
- DNS poisoning is stopped by random ports and DNSSEC. BGP hijacks are reduced by RPKI and filters. Downgrades are stopped by disabling old versions and using HSTS.
- For any attack, be ready to say how it works, what it breaks, how you detect it and how you stop it.
