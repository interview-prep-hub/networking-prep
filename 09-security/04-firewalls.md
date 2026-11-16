# Firewalls

**Level:** Intermediate | **Prerequisites:** [Network Security Basics](01-security-basics.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md)

## In One Line

A firewall sits between networks (or on a host) and decides which traffic may pass, based on a list of rules. Anything the rules do not allow should be dropped.

## Why It Matters

Every device on a network listens for something. A database listens on a port. A printer listens on a port. If any computer on the Internet can reach those ports, attackers will find them. Automated scanners sweep the whole IPv4 Internet all the time.

A firewall solves this by reducing what can be reached. A company may run 500 services, but the Internet should reach only two or three of them. The rest stay out of reach, even if they have bugs.

Interviewers ask about firewalls because the topic tests several skills at once:

- Do you understand layers? Different firewalls read different layers.
- Do you understand state? The stateless vs stateful question is a classic.
- Can you design? "Draw a DMZ" and "Write the rules for this network" are common tasks.
- Can you troubleshoot? "The app freezes after one hour idle" is often a firewall timeout.

## Simple Explanation

Think of an office building with a security desk at the entrance.

- **Packet filter:** a guard who reads only the address label on each parcel. The guard has no memory. Each parcel is judged alone.
- **Stateful firewall:** the same guard with a logbook. When an employee sends a request out, the guard writes it down. When the answer comes back, the guard checks the logbook and lets it in.
- **Proxy firewall:** a receptionist. Visitors never meet the staff. The receptionist takes the message, reads it, and carries a fresh copy inside.
- **Next-generation firewall (NGFW):** a guard with a logbook, a staff ID list and an X-ray machine. The guard knows who is sending what, and can look inside parcels.
- **Web application firewall (WAF):** a specialist at one counter who reads every web form and rejects forged or dangerous ones.

**Where the analogy stops:** a real firewall has no judgment. It follows rules, and it makes decisions in microseconds. It also controls only traffic that passes through it. Two people talking inside the same room (two hosts on the same subnet) never pass the security desk.

## How It Works

### Words You Need First

- **Packet:** one chunk of data sent across a network, with headers that say where it is going.
- **Port:** a number from 0 to 65535 that identifies a service on a host. HTTPS normally uses TCP 443.
- **Ephemeral port:** a short-lived port the client picks for its side of a connection. Linux uses 32768-60999 by default. Windows uses 49152-65535 by default.
- **5-tuple:** the five values that identify a connection: source IP, destination IP, protocol, source port, destination port.
- **Rule:** one line of policy, such as "allow TCP from any to 203.0.113.10 port 443".
- **Rule base (or policy):** the full ordered list of rules.
- **Drop:** throw the packet away and say nothing.
- **Reject:** throw the packet away and tell the sender, with a TCP reset or an ICMP error.

### What Each Firewall Can See

A packet is like a set of nested envelopes. Each firewall type opens a different number of them.

```text
+----------+----------+----------+----------+-----------------+
| Ethernet | IP       | TCP/UDP  | TLS      | HTTP            |
| MACs     | src, dst | ports,   | SNI,     | URL, headers,   |
| VLAN tag | protocol | flags    | versions | cookies, body   |
+----------+----------+----------+----------+-----------------+
| Layer 2  | Layer 3  | Layer 4  | Layer 5-6| Layer 7         |
+----------+----------+----------+----------+-----------------+

Packet filter : IP and L4 headers, one packet at a time
Stateful      : IP and L4 headers, plus connection memory
Proxy         : everything, for the protocols it understands
NGFW          : stateful, plus application, user and threat checks
WAF           : HTTP in full, including the request body
```

The layer labels are a teaching model. Real traffic does not fit the OSI model perfectly. TLS, for example, is hard to place on a single layer.

### Type 1: Packet Filter (Stateless)

A **packet filter** checks each packet alone against the rules. "Stateless" means it keeps no memory of earlier packets.

It can match on:

- Source and destination IP
- Protocol (TCP, UDP, ICMP)
- Source and destination port
- TCP flags, such as SYN and ACK
- Interface and direction

The problem is return traffic. A client at 203.0.113.20 connects to a web server. The reply comes back to the client's ephemeral port, for example 52344. A stateless filter does not know this reply belongs to anything. So you must add a rule that allows it.

Here is a stateless filter on Cisco IOS. In this small example the hosts have public addresses and there is no NAT.

```text
! Cisco IOS: stateless filter, applied inbound on the outside interface
ip access-list extended OUTSIDE-IN
 permit tcp any host 203.0.113.10 eq 443
 permit tcp any 203.0.113.0 0.0.0.255 established
 deny   ip any any log
!
interface GigabitEthernet0/1
 ip access-group OUTSIDE-IN in
```

The word `established` looks like state, but it is not. It only means "the ACK or RST flag is set". Any attacker can craft a packet with the ACK flag set, and it will pass. A tool like nmap uses this in an ACK scan to map filter rules.

Packet filters are still useful. They are very fast and they have no state to run out of. You find them in router [ACLs](05-access-control-lists.md) and in cloud network ACLs ([AWS NACLs](../11-cloud-and-containers/04-cloud-security-and-connectivity.md)). They often drop obvious junk at the network edge before a more expensive device sees it.

### Type 2: Stateful Firewall

A **stateful firewall** remembers connections. It keeps a **state table** (also called a session table or connection table). Each active connection has one entry.

This gives two big benefits:

1. You write a rule only for the direction that starts the connection.
2. Packets that do not belong to any known connection get dropped, even if the ACK flag is set.

**Analogy:** when you call a friend and they answer, the line stays open both ways. Nobody asks for permission for each sentence. A stranger cannot join by just speaking on your line. The analogy stops here: a firewall forgets a "call" after an idle timer, and a phone line does not.

### Connection Tracking Step by Step

**Connection tracking** is the process of building and using the state table. This is what happens for an outbound HTTPS connection. NAT is left out to keep it simple.

```text
LAN client               Firewall                 Web server
10.0.1.20                                         198.51.100.80
   |                         |                         |
   |-- SYN, dport 443 ------>|                         |
   |          no entry: check rule list, rule allows   |
   |          create entry in state table              |
   |                         |-- SYN, dport 443 ------>|
   |                         |<-- SYN-ACK -------------|
   |          entry found: no rule check needed        |
   |<-- SYN-ACK -------------|                         |
   |-- ACK ----------------->|-- ACK ----------------->|
   |          handshake done, entry is established     |
   |<===== data both ways, matched by the entry ======>|
   |-- FIN ----------------->|-- FIN ----------------->|
   |          closing: short timer, then entry removed |
```

The logic for each packet is:

1. Read the 5-tuple from the packet.
2. Look it up in the state table. This uses a hash lookup, so it is fast even with many entries.
3. **Entry found:** check that the packet fits the connection. For TCP, this means sensible flags and sequence numbers. Forward it and reset the idle timer. The rule list is not read.
4. **No entry, and the packet could start a connection:** walk the rule list. If a rule allows it, create an entry and forward it. If not, drop it.
5. **No entry, and the packet cannot start a connection:** for example, a stray SYN-ACK. Most firewalls drop it as invalid.
6. **Connection ends or goes idle:** remove the entry when the timer runs out.

Step 3 has a side effect that surprises many people. **A new deny rule does not stop a connection that already exists.** The packets match the state table first, so they never reach your new rule. You must also clear the entry:

```bash
# Linux: delete tracked connections from one source
sudo conntrack -D -s 10.0.1.66
```

```text
! Cisco ASA: clear connections from one host
clear conn address 10.0.1.66
```

#### What About UDP and ICMP?

UDP has no handshake and no "end of connection". ICMP has no ports. Stateful firewalls track them anyway, with a pseudo-connection.

- **UDP:** the first packet creates an entry. Replies with the swapped 5-tuple are allowed until an idle timer ends.
- **ICMP echo:** an echo request creates an entry. The matching echo reply is allowed, matched by the ICMP identifier.
- **ICMP errors:** an error such as "fragmentation needed" carries a copy of the original packet header. The firewall reads it and links the error to the existing connection. Linux calls this state RELATED.

#### Linux conntrack States

Linux has a stateful engine inside the kernel called **conntrack**. Its states are firewall states, not [TCP states](../06-transport-layer/04-tcp-states.md).

| State | Meaning |
|---|---|
| NEW | First packet of a connection the firewall has not seen before |
| ESTABLISHED | Packets have been seen in both directions |
| RELATED | A new flow linked to a known one, such as an ICMP error |
| INVALID | Does not fit any known connection, or is malformed |

#### Timers

Idle timers are not part of any standard. They are vendor defaults, and they vary by version.

| Platform | Setting | Typical default |
|---|---|---|
| Linux | `nf_conntrack_tcp_timeout_established` | 432000 s (5 days) |
| Linux | `nf_conntrack_tcp_timeout_time_wait` | 120 s |
| Linux | `nf_conntrack_udp_timeout` | 30 s |
| Linux | `nf_conntrack_udp_timeout_stream` | 120 s |
| Linux | `nf_conntrack_icmp_timeout` | 30 s |
| Cisco ASA | `timeout conn` (TCP idle) | 1 hour |
| Cisco ASA | `timeout udp` | 2 minutes |

Always check on the real device. On Linux, use `sysctl -a | grep nf_conntrack`.

### Type 3: Proxy Firewall

An **application proxy** (or proxy firewall) does not forward packets. It ends the client's connection and opens a second connection to the server. It then reads the application data in the middle.

```text
Client                     Proxy                    Server
10.0.1.20             10.0.2.30:3128            198.51.100.80
   |                         |                         |
   |<=== connection 1 ======>|                         |
   |                         |<=== connection 2 ======>|
   |                         |                         |
   |   proxy reads the request, applies policy,        |
   |   then sends its own new request to the server    |
```

The server sees the proxy's address, not the client's. No packet passes directly from the client to the server. This also blocks many attacks that depend on odd packets, because the proxy builds new, clean ones.

**Analogy:** a translator between two people. Neither one hears the other directly. The analogy stops with encrypted traffic. If the two people speak in code, the translator can only pass on what it can read.

There are two main forms:

- **Forward proxy:** protects clients going out. Example: Squid, whose default port is 3128.
- **Reverse proxy:** protects servers. It accepts requests from the Internet for them. Example: NGINX or HAProxy.

Details are in [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

A client can send traffic through a forward proxy like this:

```bash
# Linux: send one request through a forward proxy
curl -x http://10.0.2.30:3128 https://example.com/
```

For HTTPS, the client sends a CONNECT request, and the proxy just relays encrypted bytes. It sees the host name but not the URL path. To see the full URL it must decrypt the traffic (see TLS Inspection under Going Deeper).

A lighter form is the **circuit-level gateway**. It relays a connection after checking who opens it, but does not read the application data. SOCKS5 (RFC 1928) is the common example.

### Type 4: Next-Generation Firewall (NGFW)

An **NGFW** is a stateful firewall that also understands applications, users and threats. Gartner made the term popular around 2009. It is a market term, not a standard, so features vary by vendor.

Common NGFW features:

- **Application identification:** it knows that a flow is Zoom or BitTorrent, based on traffic patterns, not on the port number.
- **User identity:** it matches rules to people and groups, using sources such as Active Directory.
- **Intrusion prevention (IPS):** it checks traffic against attack signatures. See [IDS and IPS](06-ids-and-ips.md).
- **TLS inspection:** it decrypts traffic so that it can inspect it.
- **URL filtering and threat feeds:** it blocks known bad sites and addresses.
- **Deep packet inspection (DPI):** this is the general term for reading past the headers, into the payload.

A classic rule says "allow port 443". An NGFW rule says who and what, as in this example in the Palo Alto Networks style:

| Name | From | To | User | Application | Service | Action |
|---|---|---|---|---|---|---|
| Sales-CRM | LAN | Outside | group: sales | salesforce | application-default | allow |
| Block-P2P | LAN | Outside | any | bittorrent | any | deny |

**Fuller truth:** the NGFW must see a few packets of a flow before it can name the application. So the TCP handshake and the first data pass under a temporary decision. Also, every extra feature costs speed. Vendor datasheets usually show a much lower "threat prevention throughput" than "firewall throughput". Size the device on the first number.

### Type 5: Web Application Firewall (WAF)

A **WAF** protects web applications and APIs. It reads HTTP requests and blocks attacks hidden inside them, such as:

- SQL injection (putting database commands into a form field)
- Cross-site scripting (XSS, putting scripts into pages that others will load)
- Path traversal (asking for files like `../../etc/passwd`)
- Abuse, such as login floods (through rate limits)

A network firewall cannot stop these attacks. It sees a normal connection to port 443 from an allowed source. This request looks fine at Layer 4:

```text
GET /products?id=1%27%20OR%20%271%27%3D%271 HTTP/1.1
Host: shop.example.com
```

After URL decoding, the parameter is `id=1' OR '1'='1`. A WAF decodes it, sees a SQL injection pattern, and returns HTTP 403.

Where a WAF sits:

```text
Browser --> Internet --> Network firewall --> WAF --> Web app
                         allows TCP 443       ends TLS,
                                              reads HTTP
```

A WAF normally acts as a reverse proxy, so it must end the TLS connection to read the request. It comes in several forms:

- **Cloud service:** AWS WAF, Cloudflare WAF, Azure WAF
- **Appliance:** F5 and others
- **Software module:** ModSecurity or Coraza, often with the OWASP Core Rule Set

WAFs use two main models:

- **Negative model:** block what matches known attack patterns.
- **Positive model:** allow only what matches the expected API shape, and block the rest.

Start in detection (log-only) mode and tune it. A WAF that blocks real customers is a production outage.

**Fuller truth:** a WAF reduces risk but does not fix the code. It cannot see business logic flaws, such as a user who changes an order number and reads another customer's order. Attackers can also skip it if they can reach the server's real IP, so the origin must accept traffic only from the WAF.

### Comparison Table

| Feature | Packet filter | Stateful | Proxy | NGFW | WAF |
|---|---|---|---|---|---|
| Main layers | 3-4 | 3-4 | 7 for supported protocols | 3-7 | 7 (HTTP) |
| Reads | IP, ports, flags | Same, plus connection state | Full application request | State, app, user, threats | URL, headers, body |
| Remembers connections | No | Yes | Yes (two separate ones) | Yes | Yes |
| Return traffic | Needs its own rule | Automatic | Handled by proxy | Automatic | Handled by proxy |
| Protects | Network edge | Networks | Clients or servers | Whole network | Web apps and APIs |
| Reads encrypted data | No | No | Only if it ends TLS | Only with TLS inspection | Yes, it ends TLS |
| Hides internal addresses | No | Only with NAT | Yes | Only with NAT | Yes |
| Relative speed | Fastest | Fast | Slower | Slower with all features on | Slower per request |
| Main blind spot | Crafted packets, no context | Attacks inside allowed ports | Unsupported protocols | Encrypted traffic, tuning effort | Non-HTTP traffic, logic flaws |
| Examples | Cisco ACL, AWS NACL | Cisco ASA, nftables with conntrack, AWS security group | Squid | Palo Alto, FortiGate, Check Point, Cisco Secure Firewall | AWS WAF, Cloudflare, ModSecurity |

The lines blur in practice. A modern NGFW includes stateful filtering, and some include proxy functions. Real networks use several types in layers.

### Host Firewall vs Network Firewall

A **host firewall** runs on the computer it protects. A **network firewall** is a separate device or virtual appliance that protects many machines.

| Point | Host firewall | Network firewall |
|---|---|---|
| Location | On the server or laptop | At the boundary between networks |
| Examples | Windows Defender Firewall, nftables, ufw, firewalld | Cisco, Palo Alto, Fortinet, pfSense |
| Sees traffic between hosts on the same subnet | Yes | No |
| Knows which program owns a socket | Yes, on some systems | No |
| If the host is compromised | Attacker with admin rights can turn it off | Keeps working |
| Management at scale | Needs central tools | One central policy |
| Cost | Uses host CPU | Needs hardware or licences |
| Follows a laptop to a cafe | Yes | No |

Use both. This is **defence in depth**, which means several independent layers, so that one failure does not expose everything.

**Analogy:** a network firewall is the gate of a housing estate. A host firewall is the lock on each house. If someone gets through the gate, the locks still work. The analogy stops at laptops, which take their lock with them wherever they go.

### Zones

A **zone** is a group of interfaces or networks that have the same trust level. You write policy between zones, not between single interfaces. This keeps rule bases short.

| Zone | Contains | Trust |
|---|---|---|
| Outside (untrust) | The Internet | None |
| DMZ | Public-facing servers | Low |
| Guest | Visitor Wi-Fi | Low |
| LAN (trust) | Staff devices | Medium |
| Servers | Internal apps and databases | High |
| Management | Admin access to devices | Highest |

Note that a zone is not a [VLAN](../03-ethernet-and-switching/06-vlans.md). A VLAN is a Layer 2 broadcast domain. A zone is a label in firewall policy. Five VLANs can belong to one zone.

Zone behaviour depends on the vendor:

- **Cisco IOS zone-based firewall:** traffic between different zones is dropped unless a zone pair and policy allow it. Traffic inside one zone passes by default.
- **Palo Alto:** default rules allow intrazone traffic and deny interzone traffic.
- **Cisco ASA:** each interface has a security level from 0 to 100. By default, traffic may go from a higher level to a lower one. Traffic from lower to higher needs an access rule.

Example of a Cisco IOS zone-based firewall:

```text
! Cisco IOS: LAN may start web and DNS traffic to the outside
zone security INSIDE
zone security OUTSIDE
!
class-map type inspect match-any CM-WEB
 match protocol http
 match protocol https
 match protocol dns
!
policy-map type inspect PM-IN-TO-OUT
 class type inspect CM-WEB
  inspect
 class class-default
  drop log
!
zone-pair security ZP-IN-OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM-IN-TO-OUT
!
interface GigabitEthernet0/0
 zone-member security INSIDE
interface GigabitEthernet0/1
 zone-member security OUTSIDE
```

The `inspect` action makes the session stateful, so replies return without a second zone pair.

### DMZ Design

A **DMZ** (demilitarised zone, also called a screened subnet) is a separate network for servers that the Internet must reach. The name comes from a strip of land between two armies.

The key idea: **assume that a public server will be hacked one day, and limit what the attacker can do next.**

**Analogy:** a bank has a public hall and a vault. Anyone can walk into the hall. Staff carry only the needed items between the two, and customers cannot go past the counter. The analogy stops because a DMZ is not an open place. It is a tightly limited network, with default deny in every direction.

#### Single Firewall (Three-Legged) Design

One firewall has three interfaces.

```text
                  Internet
                     |
               +-----------+
               | Firewall  |
               +-----------+
                 |       |
          +------+       +------+
          |                     |
      DMZ zone              LAN zone
  web, mail relay       users, databases
```

It is cheap and simple. But one firewall is a single point of failure, in both senses: if it is breached or misconfigured, everything is exposed.

#### Dual Firewall (Back-to-Back) Design

```text
Internet
   |
+----------------+
| Outer firewall |
+----------------+
   |
DMZ zone: web, mail relay, reverse proxy
   |
+----------------+
| Inner firewall |
+----------------+
   |
Internal zone: users, databases
```

An attacker must beat two firewalls. Some designs use two vendors, so that one software bug does not affect both. The cost is more devices, more rules and more operational work.

| Point | Single firewall | Dual firewall |
|---|---|---|
| Cost | Lower | Higher |
| Rule management | One place | Two places |
| Isolation | Depends on one policy | Two independent layers |
| Common use | Small and mid-size sites | Banks, regulated sectors, large sites |

#### DMZ Rules to Remember

1. **Internet to internal:** never. No direct path at all.
2. **Internet to DMZ:** only the published ports, to the exact servers.
3. **DMZ to internal:** as little as possible. One source, one destination, one port.
4. **Internal to DMZ:** only for admin tasks, from named admin hosts.
5. **DMZ to Internet:** denied by default. This stops a hacked server from downloading tools or calling home.

Typical DMZ residents: reverse proxies, public web servers, mail relays, public DNS servers, VPN gateways. Databases do not belong there.

Also limit traffic **inside** the DMZ. Two servers on the same subnet talk through the switch, so the firewall never sees it. Use separate subnets for each service, or host firewalls. See [Zero Trust and Segmentation](09-zero-trust-and-segmentation.md).

**Fuller truth:** the "DMZ" setting on a home router is not a real DMZ. It just forwards all unsolicited inbound traffic to one device on your LAN, and that device can still reach the rest of your LAN.

### Inbound and Outbound Rules

These words have two meanings, and this causes many mistakes.

- **Policy meaning:** inbound means a connection started from outside. Outbound means a connection started from inside.
- **Interface meaning:** on a Cisco ACL, `in` means "packets entering this interface". `out` means "packets leaving this interface". It says nothing about who started the connection.

On a stateful firewall, think about **who starts the connection**. Replies do not need their own rule.

**Inbound rules** allow only published services. Also drop obvious fakes at the edge, such as packets from the Internet with a source address in the [private ranges](../04-ip-addressing/03-public-private-and-special-addresses.md) (RFC 1918) or in your own public range.

**Outbound rules** are often forgotten. Many networks allow everything out, and this helps attackers. Malware needs to call home and send stolen data. **Egress filtering** means controlling outbound traffic.

| Outbound rule | Why |
|---|---|
| Staff PCs may use TCP 80 and 443 to the Internet | Limits malware tools and tunnels |
| DNS to the Internet only from the internal resolver | Stops DNS bypass and makes DNS tunnelling harder |
| TCP 25 only from the mail relay | Stops infected PCs from sending spam |
| Block TCP 445 to the Internet | SMB should not leave the network |
| DMZ servers reach only named update hosts | Blocks reverse shells and tool downloads |
| Source address must belong to your networks | Stops spoofed packets from leaving (BCP 38, RFC 2827) |

### Rule Order

Most firewalls read rules from top to bottom and stop at the **first match**. The rest of the list is not read.

**Analogy:** a door guard reads a checklist from the top and acts on the first line that fits you. The analogy stops with vendors that work differently, as listed below.

This order has a bug:

```text
Rule 10: allow tcp from 10.0.1.0/24 to any port 443
Rule 20: deny  tcp from 10.0.1.66/32 to any port 443
```

Rule 20 never matches, because 10.0.1.66 is inside 10.0.1.0/24. Rule 10 catches it first. This is called a **shadowed rule**. The fix is to put the specific rule first:

```text
Rule 10: deny  tcp from 10.0.1.66/32 to any port 443
Rule 20: allow tcp from 10.0.1.0/24 to any port 443
```

A good layout for a rule base:

1. Anti-spoofing and known-bad blocks
2. Access to the firewall itself (very limited)
3. Published inbound services
4. Traffic between internal zones
5. User traffic to the Internet
6. Default deny with logging (the cleanup rule)

Not every system uses first match:

| System | Behaviour |
|---|---|
| Cisco ACL, ASA, Palo Alto, FortiGate | Top-down, first match wins |
| iptables and nftables | Top-down, a terminating action ends the chain |
| OpenBSD pf | **Last** match wins, unless a rule has `quick` |
| Windows Defender Firewall | A matching block rule beats a matching allow rule, in general |
| AWS security groups | No order. Allow rules only, and any match allows |

Put busy rules high when you can, because some engines walk the list. But never move a rule if that changes the result.

### Default Deny

**Default deny** means: anything not clearly allowed is blocked. It is an allow list. The opposite is **default allow**, which is a block list.

| Point | Default deny | Default allow |
|---|---|---|
| New unknown service | Blocked | Exposed |
| Forgotten rule | Something breaks, and you notice | A hole stays open, and nobody notices |
| Work needed | More at the start | More risk later |

**Analogy:** a private party with a guest list. If your name is not on it, you do not get in. The analogy stops because a party does not break when a guest is missing. A firewall does: one missing rule can stop a business system. So test changes and read the logs.

Details by platform:

- Cisco ACLs end with an invisible `deny ip any any`. It does not log. Add an explicit `deny ip any any log` if you want a record.
- Palo Alto ends with `interzone-default` (deny), which does not log unless you turn logging on.
- Windows Defender Firewall blocks unsolicited inbound and allows outbound by default.
- nftables and iptables have a default **policy** on each chain. A policy of accept is default allow.

#### Drop or Reject?

| Point | Drop | Reject |
|---|---|---|
| Sender sees | Nothing, then a timeout | Instant RST or ICMP error |
| Good for | The Internet edge | Internal networks |
| Effect on troubleshooting | Slow, unclear | Fast, clear |

Dropping gives scanners less information and sends no extra traffic. Rejecting saves your own staff from long timeouts. A common choice: drop at the Internet edge and reject inside.

## Example

A small company needs a firewall policy. Here it is built on a Linux firewall called `fw1` with four interfaces.

### Topology

```text
                     Internet
                        |
              203.0.113.1 (ISP router)
                        |
              eth0: 203.0.113.2/24
              (also holds public web IP 203.0.113.10)
                        |
             +----------------------+
             |    Firewall fw1      |
             +----------------------+
               |        |         |
             eth1      eth2      eth3
               |        |         |
              LAN      DMZ      Servers
```

| Zone | Interface | Firewall address | Subnet | Hosts |
|---|---|---|---|---|
| Outside | eth0 | 203.0.113.2 | 203.0.113.0/24 | ISP router 203.0.113.1 |
| LAN | eth1 | 10.0.1.1 | 10.0.1.0/24 | Staff PCs, admin PC 10.0.1.50 |
| DMZ | eth2 | 172.16.10.1 | 172.16.10.0/24 | Web server 172.16.10.10 |
| Servers | eth3 | 10.0.2.1 | 10.0.2.0/24 | Database 10.0.2.20, DNS resolver 10.0.2.53 |

### The Policy on Paper

Write the matrix first. It shows what each zone may start.

| From \ To | Outside | DMZ | LAN | Servers |
|---|---|---|---|---|
| Outside | - | TCP 80, 443 to web | Deny | Deny |
| DMZ | TCP 443 to patch mirror only | - | Deny | TCP 5432 to db only |
| LAN | TCP 80, 443 | SSH from admin only | - | DNS to resolver only |
| Servers | DNS from resolver only | Deny | Deny | - |

PostgreSQL uses port 5432. The patch mirror is at 198.51.100.40.

### The Rule Set

This is nftables. Full syntax is in [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

```bash
#!/usr/sbin/nft -f
# Linux nftables: /etc/nftables.conf on fw1
flush ruleset

table inet fw {
  chain input {
    type filter hook input priority filter; policy drop;
    ct state established,related accept
    ct state invalid drop
    iif "lo" accept
    # Only the admin PC may SSH to the firewall itself
    iifname "eth1" ip saddr 10.0.1.50 tcp dport 22 accept
    log prefix "FW-IN-DROP: "
  }

  chain forward {
    type filter hook forward priority filter; policy drop;
    ct state established,related accept
    ct state invalid drop

    # R1 Outside to DMZ: public website (address is after DNAT)
    iifname "eth0" oifname "eth2" ip daddr 172.16.10.10 \
      tcp dport { 80, 443 } accept
    # R2 DMZ to Servers: web app to database
    iifname "eth2" oifname "eth3" ip saddr 172.16.10.10 \
      ip daddr 10.0.2.20 tcp dport 5432 accept
    # R3 LAN to Servers: DNS to the internal resolver
    iifname "eth1" oifname "eth3" ip saddr 10.0.1.0/24 \
      ip daddr 10.0.2.53 meta l4proto { tcp, udp } th dport 53 accept
    # R4 Servers to Outside: resolver may ask Internet DNS
    iifname "eth3" oifname "eth0" ip saddr 10.0.2.53 \
      meta l4proto { tcp, udp } th dport 53 accept
    # R5 LAN to Outside: web browsing
    iifname "eth1" oifname "eth0" ip saddr 10.0.1.0/24 \
      tcp dport { 80, 443 } accept
    # R6 Admin PC to DMZ: SSH
    iifname "eth1" oifname "eth2" ip saddr 10.0.1.50 \
      ip daddr 172.16.10.10 tcp dport 22 accept
    # R7 DMZ to Outside: patch mirror only
    iifname "eth2" oifname "eth0" ip saddr 172.16.10.10 \
      ip daddr 198.51.100.40 tcp dport 443 accept
    # R8 Default deny: log it, then the chain policy drops it
    log prefix "FW-DROP: "
  }
}

table ip nat {
  chain prerouting {
    type nat hook prerouting priority dstnat;
    iifname "eth0" ip daddr 203.0.113.10 tcp dport { 80, 443 } \
      dnat to 172.16.10.10
  }
  chain postrouting {
    type nat hook postrouting priority srcnat;
    oifname "eth0" ip saddr { 10.0.1.0/24, 10.0.2.53, 172.16.10.10 } \
      snat to 203.0.113.2
  }
}
```

The firewall must also have routing turned on (`sysctl net.ipv4.ip_forward=1`). IPv6 packets here all reach the drop policy. A dual-stack site needs IPv6 rules as well.

### Walking Packets Through It

**Packet 1: a customer opens the website.** It is 198.51.100.7:51000 to 203.0.113.10:443.

1. It arrives on eth0. DNAT changes the destination to 172.16.10.10.
2. There is no conntrack entry, so the forward chain reads the rules.
3. It matches R1. It is accepted and an entry is created.

**Packet 2: the web server replies.** It is 172.16.10.10:443 to 198.51.100.7:51000.

1. conntrack finds the entry, so `ct state established,related accept` matches at the top.
2. The source is changed back to 203.0.113.10.
3. No rule for the reply direction was needed.

**Packet 3: the web server is hacked and tries the admin PC.** It is 172.16.10.10:48122 to 10.0.1.50:445.

1. It has no entry. It is a new connection from eth2 to eth1.
2. No rule matches. The log line is written, and the policy drops it.

**Packet 4: a scanner tries the database from the Internet.** It is 198.51.100.99:40001 to 203.0.113.2:5432.

1. The address belongs to the firewall itself, so the packet goes to the input chain.
2. No rule allows it. It is logged and dropped.

### Checking It

```bash
# Linux on fw1: check syntax, load, then view connections
sudo nft -c -f /etc/nftables.conf
sudo nft -f /etc/nftables.conf
sudo conntrack -L -p tcp --dport 443
```

```text
tcp      6 431985 ESTABLISHED src=198.51.100.7 dst=203.0.113.10
sport=51000 dport=443 src=172.16.10.10 dst=198.51.100.7 sport=443
dport=51000 [ASSURED] mark=0 use=1
conntrack v1.4.8 (conntrack-tools): 1 flow entries have been shown.
```

The real output is one long line per entry. It is wrapped here. It shows two directions. The first half is the original packet. The second half is the expected reply. The source of the reply is 172.16.10.10, not 203.0.113.10, and this shows that DNAT is active. The number 431985 is the seconds left on the 5-day timer.

The log shows packet 3, in this form:

```text
Oct 30 10:14:02 fw1 kernel: FW-DROP: IN=eth2 OUT=eth1 MAC=...
SRC=172.16.10.10 DST=10.0.1.50 LEN=60 TOS=0x00 PREC=0x00 TTL=63
ID=4211 DF PROTO=TCP SPT=48122 DPT=445 WINDOW=64240 RES=0x00 SYN URGP=0
```

The error message also tells you about the path. Run this from the web server:

```bash
# Linux: test TCP ports from 172.16.10.10 to the database host
nc -vz -w 3 10.0.2.20 5432
nc -vz -w 3 10.0.2.20 5433
```

```text
Connection to 10.0.2.20 5432 port [tcp/postgresql] succeeded!
nc: connect to 10.0.2.20 port 5433 (tcp) timed out: Operation now in progress
```

Port 5432 passes R2. Port 5433 has no rule, so fw1 drops it silently and `nc` times out. If the firewall allowed the port and nothing listened there, you would see `Connection refused`. That message means the packet reached something that replied. More tools are in [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

### Adding a Host Firewall

The network firewall does not see traffic between hosts on 10.0.2.0/24. So the database server gets its own firewall. Run this from the console, or add your SSH rule first, or you will lock yourself out.

```bash
# Linux (Ubuntu ufw) on db 10.0.2.20
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 172.16.10.10 to any port 5432 proto tcp
sudo ufw enable
sudo ufw status verbose
```

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
5432/tcp                   ALLOW IN    172.16.10.10
```

The same rule on a Windows Server:

```powershell
# Windows: allow the web server to reach PostgreSQL
New-NetFirewallRule -DisplayName "PostgreSQL from web" `
  -Direction Inbound -Protocol TCP -LocalPort 5432 `
  -RemoteAddress 172.16.10.10 -Action Allow

# Windows: view the effective default actions
Get-NetFirewallProfile -PolicyStore ActiveStore |
  Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

```text
Name    Enabled DefaultInboundAction DefaultOutboundAction
----    ------- -------------------- ---------------------
Domain     True                Block                 Allow
Private    True                Block                 Allow
Public     True                Block                 Allow
```

## Going Deeper

### Asymmetric Routing

A stateful firewall must see both directions of a connection. Suppose the SYN leaves through firewall A, but the SYN-ACK returns through firewall B. Firewall B has no entry, so it drops the SYN-ACK as invalid.

```text
Client --> Router --> Firewall A --> Internet --> Server
Client <-- Router <-- Firewall B <-- Internet <-- Server
                      no entry: drop
```

Fixes: make routing symmetric, or use a firewall cluster that shares state. A related Linux detail: with `nf_conntrack_tcp_loose` set to 1 (the default), conntrack can pick up a connection in the middle, so a mid-stream ACK can show as NEW. This is one more reason to require the SYN flag in strict rules.

### High Availability and State Sync

Firewalls usually run in pairs.

- **Active/passive:** one device forwards, the other waits.
- **Active/active:** both forward traffic.

The pair copies the state table across a sync link. Without this, every connection dies at failover. Check whether the vendor syncs all state: some application or decryption state may not be copied. See [High Availability and Redundancy](../10-enterprise-and-datacenter/07-high-availability.md).

Also decide how a device should fail. **Fail closed** means traffic stops, so security comes first. **Fail open** means traffic flows without inspection, so availability comes first. Firewalls normally fail closed. Inline IPS bypass units may fail open.

### State Table Exhaustion

The state table lives in memory and it has a limit. This makes the firewall itself a target. A **SYN flood** sends many connection attempts and fills the table. See [Common Network Attacks](03-common-network-attacks.md).

On Linux you can check the usage:

```bash
# Linux: current entries and the limit
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
```

When the table fills, the kernel logs `nf_conntrack: table full, dropping packet`, and new connections fail. Defences include SYN cookies or a SYN proxy, per-source limits, shorter timers for half-open connections, and stateless filtering upstream. For very busy public services, teams sometimes skip connection tracking on those flows.

When you size a firewall, check three numbers: throughput, maximum concurrent sessions and new connections per second.

### Idle Timeouts Breaking Applications

This is a classic real incident. An app keeps a database connection open in a pool. The connection sits idle for more than the firewall's TCP idle timer, for example 1 hour on a default ASA. The firewall removes the entry and tells nobody. The next query is dropped as invalid, and the app hangs.

The fix is TCP keepalive at a shorter interval than the firewall timer. The Linux default `net.ipv4.tcp_keepalive_time` is 7200 seconds, which is 2 hours. That is too long for a 1 hour timer. Set it lower on the host or in the connection pool.

### Application Layer Gateways

Some protocols put IP addresses or port numbers inside their payload. In FTP active mode, the client tells the server which port to connect back to. A firewall cannot allow this without reading the control channel.

An **application layer gateway (ALG)** reads the protocol and opens a temporary hole for the new flow. Linux has conntrack helpers for this. Since kernel 4.7, helpers are no longer auto-assigned by default, so you attach them in your rules. ALGs fail if the control channel is encrypted, and SIP ALGs have a reputation for breaking calls. See [File Transfer Protocols](../07-application-protocols/08-file-transfer-protocols.md).

### TLS Inspection

Most traffic is encrypted. Without decryption, a firewall sees only IPs, ports, flow patterns and a few fields of the TLS handshake. In TLS 1.3 the server certificate is encrypted too. Encrypted Client Hello (ECH) hides the server name as well.

An outbound TLS inspection firewall acts as a planned man in the middle:

1. The client starts TLS to example.com.
2. The firewall opens its own TLS connection to the real server and checks the real certificate.
3. The firewall creates a fake certificate for example.com, signed by a company CA (certificate authority).
4. The client accepts it, because the company installed that CA on its devices.
5. The firewall decrypts, inspects and encrypts again.

Costs and risks:

- It uses a lot of CPU, so throughput falls.
- Certificate pinning apps will break.
- Banking and health traffic is often excluded for privacy and legal reasons.
- The firewall holds sensitive plaintext, and its CA key becomes a high-value target.

For inbound traffic, the firewall usually holds the server's own key or ends the TLS itself. More on TLS is in [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

### QUIC

[QUIC](../06-transport-layer/08-quic.md) runs over UDP, usually port 443, and it encrypts most of its headers. Firewalls can see much less of it than TCP with TLS. A common enterprise choice is to block UDP 443, so that browsers fall back to TCP and TLS, where inspection works. This is a policy trade-off, not a requirement.

### IPv6

IPv6 usually has no NAT, so a host may have a globally routable address. The firewall is therefore the only thing that blocks inbound traffic. A dual-stack device has separate rule sets. A common mistake is a tight IPv4 policy and an open IPv6 policy.

Do not block all ICMPv6. Neighbor Discovery and Path MTU Discovery depend on it. RFC 4890 gives filtering advice.

### NAT and Rule Matching

Does a rule match the address before or after NAT? It depends on the vendor.

- **Linux:** DNAT happens in prerouting, so forward rules see the private address. The example above used 172.16.10.10 in R1.
- **Cisco ASA (8.3 and later):** access rules use the real (private) address.
- **Palo Alto:** security rules use the pre-NAT address with the post-NAT zone.

NAT itself is not a firewall. Stateful NAT drops unsolicited inbound packets as a side effect, because there is no mapping. It does not inspect or apply policy.

### Routed vs Transparent Mode

- **Routed mode:** the firewall is a Layer 3 hop, with an IP on each interface.
- **Transparent mode:** the firewall acts as a Layer 2 bridge. You can insert it without changing addresses.

### Cloud and Service Firewalls

Cloud platforms split the job. AWS security groups are stateful and attach to network interfaces. AWS network ACLs are stateless and attach to subnets, so they need rules for ephemeral return ports. See [Security Groups, NACLs and VPC Connectivity](../11-cloud-and-containers/04-cloud-security-and-connectivity.md). **Firewall as a service (FWaaS)** delivers firewall inspection from the cloud, so that remote users do not need to backhaul traffic to head office.

### Rule Base Hygiene

Real rule bases grow for years and become dangerous. Good teams:

- Link each rule to a ticket, an owner and a review date.
- Use named objects and groups instead of raw IPs.
- Check hit counters, and remove rules with no hits after a review.
- Test before a change. Cisco ASA has `packet-tracer`, which simulates a packet through the policy.
- Log denied traffic at the edges and send it to a [SIEM](06-ids-and-ips.md).

## Common Mistakes

- **"A firewall makes us secure."** It only controls traffic through it. It does not stop an attack that uses an allowed port, or a phishing link that a user clicks.
- **Allowing `any any` "just for testing"**, and then leaving it. Temporary rules often become permanent.
- **Writing a rule for return traffic on a stateful firewall.** It is not needed, and a wide high-port rule opens a real hole.
- **Forgetting return traffic on a stateless device.** A NACL or ACL needs ephemeral ports allowed, or replies die.
- **Thinking that a new deny rule kills live sessions.** Existing connections stay until the state entry is cleared.
- **Putting a broad rule above a specific one.** The specific rule then never matches.
- **Filtering inbound only.** Without egress rules, malware and data leave freely.
- **Blocking all ICMP.** This breaks Path MTU Discovery and causes [hanging connections](../06-transport-layer/09-mss-mtu-and-pmtud.md). Allow what you need, such as fragmentation needed and packet too big.
- **Confusing `in` and `out` on a Cisco ACL with inbound and outbound policy.** The first is about the interface. The second is about who started the connection.
- **Placing a database in the DMZ** because the web server needs it. Keep the data inside and allow one port.
- **Allowing wide DMZ-to-LAN access.** This turns the DMZ into a bridge for attackers.
- **Treating a WAF as a replacement for secure code.** It is a filter, not a fix.
- **Treating NAT as a firewall.** It hides addresses but has no policy.
- **Trusting port numbers.** Port 443 does not prove the traffic is HTTPS. Anything can run on it.
- **Having strict IPv4 rules and open IPv6 rules.** Attackers will use the open path.

## Interview Questions

**Q: What is a firewall, and what can it not do?**

A: A firewall enforces a policy on traffic between networks or on a host. It permits or blocks traffic based on rules. It cannot stop attacks inside allowed traffic unless it inspects that traffic, it cannot see traffic that does not pass through it, and it does not fix weak passwords or insecure code.

**Q: What is the difference between a stateless and a stateful firewall?**

A: A stateless firewall judges each packet alone, using headers such as IPs, ports and flags, so I must write rules for both directions. A stateful firewall tracks connections in a state table. I write a rule for the side that starts the connection, and replies are matched to the table entry. Stateless is faster and has no table to fill. Stateful is safer and easier to manage.

**Q: How does a stateful firewall let return traffic in without a rule?**

A: When an allowed packet starts a connection, the firewall stores the 5-tuple in its state table. Each later packet is looked up first. If the reply matches the swapped 5-tuple and fits the connection state, it passes without reading the rule list. The entry is removed when the connection closes or the idle timer ends.

**Q: How can a firewall track UDP if UDP has no connections?**

A: It creates a pseudo-connection. The first packet creates an entry with the 5-tuple and a short idle timer. Replies that match are allowed. On Linux the default is 30 seconds, and 120 seconds once traffic has flowed in both directions. These are vendor defaults, not a standard.

**Q: How does a proxy firewall differ from a stateful firewall?**

A: A stateful firewall forwards the original packets and tracks them. A proxy ends the client connection and opens a new one to the server, so there are two connections. The proxy can read the application content and it hides the client address. The cost is more processing, and it works only with protocols it supports.

**Q: What makes a firewall "next generation"?**

A: It adds application awareness, user identity, integrated intrusion prevention, and often TLS inspection and threat feeds to stateful filtering. So I can write a rule such as "the sales group may use this CRM" instead of "allow TCP 443". NGFW is a market term, so features vary by vendor.

**Q: What is the difference between an NGFW and a WAF? Do you need both?**

A: An NGFW protects a whole network across many protocols. A WAF focuses on HTTP and protects web apps from attacks such as SQL injection and XSS. It understands URLs, parameters and sessions in much more depth. For public web apps I would use both, because they cover different layers. The WAF normally ends TLS, and it sits in front of the app.

**Q: Design a DMZ for a public website with a database.**

A: I put the web server or reverse proxy in the DMZ and the database in an internal zone. The Internet reaches only 443 on the web server. The web server reaches only the database port on one database host. There is no direct Internet path to the inside. Outbound from the DMZ is denied except for named update hosts. Admin access comes only from a jump host. For higher assurance, I use two firewalls, with the DMZ between them.

**Q: Why does rule order matter? What is a shadowed rule?**

A: Most firewalls stop at the first match. A shadowed rule never fires, because a broader rule above it catches the same traffic. For example, an allow for 10.0.1.0/24 above a deny for 10.0.1.66 makes the deny useless. I put specific rules first and end with a logged default deny. Some systems differ: OpenBSD pf uses last match unless `quick` is set, and Windows Firewall lets block rules win.

**Q: What is default deny, and why filter outbound traffic too?**

A: Default deny blocks anything not clearly allowed, so unknown services and forgotten cases stay closed. Outbound filtering matters because malware has to call home and data has to leave. If only web traffic may leave, and DNS goes only through my resolver, an attacker has fewer paths. It also stops my network from sending spoofed packets.

**Q: Host firewall or network firewall: which do you use?**

A: Both. A network firewall gives a central policy at boundaries and keeps working if a host is hacked. A host firewall sees traffic within the same subnet, protects laptops away from the office, and can be specific to one server. Together they give defence in depth.

**Q: Connections through a firewall work at first, but freeze after being idle for a long time. Why?**

A: The firewall probably expired the state entry at its idle timer. The endpoints do not know this, so their next packet is dropped and the app hangs. To confirm it, I compare the time to failure with the timeout and check the logs or a packet capture. The fix is to enable TCP keepalive below the timer, or to raise the timeout for that flow.

## Quick Recap

- A firewall enforces rules on traffic. A packet filter reads headers, a stateful firewall adds connection memory, a proxy rebuilds connections, an NGFW adds app and user awareness, and a WAF deeply reads HTTP.
- Connection tracking stores the 5-tuple with an idle timer, so that replies pass without a rule. Existing sessions survive new deny rules until cleared.
- Zones group networks by trust. A DMZ holds public servers, with no direct Internet path to the inside, and tight rules in every direction.
- Most firewalls use top-down, first match. Put specific rules first, and end with a logged default deny.
- Filter outbound traffic as well as inbound. On a stateful device, think about who starts the connection.
- Use host and network firewalls together, and treat IPv6 policy with the same care as IPv4.
- A firewall reduces what can be reached. It does not fix insecure applications.
