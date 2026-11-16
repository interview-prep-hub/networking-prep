# IPv6 in Practice

**Level:** Advanced | **Prerequisites:** [IPv6 Basics](11-ipv6-basics.md), [ARP](../03-ethernet-and-switching/05-arp.md), [IPv4 Header and Fragmentation](10-ipv4-header-and-fragmentation.md), [Subnetting Step by Step](05-subnetting-step-by-step.md)

## In One Line

IPv6 in practice means a simpler fixed header, Neighbor Discovery instead of ARP, hosts that build their own addresses (SLAAC) or use DHCPv6, one /64 per LAN, and a set of tools (dual stack, tunnels, NAT64) for living with IPv4.

## Why It Matters

IPv4 has about 4.3 billion addresses, and the free pools ran out years ago. Mobile networks, cloud providers and large content networks now run IPv6 at scale. Google's public statistics show close to half of its users reaching it over IPv6.

Knowing the 128-bit address format is not enough. Real IPv6 problems look like this:

- A laptop has an address but no default gateway, because RAs (Router Advertisements) are blocked.
- A phone has no IPv6 address, because the network offers only stateful DHCPv6 and Android does not use it for addresses.
- Big downloads hang, because a firewall drops ICMPv6 Packet Too Big messages.
- A rogue device sends RAs and takes over a whole VLAN.
- Websites feel slow, because IPv6 is broken and each connection waits before it falls back to IPv4.

Interviewers ask about IPv6 to see if you have operated it, not only read about it. Expect questions such as "What replaced ARP?", "How does a host get an address with no DHCP server?" and "How does an IPv6-only client reach an IPv4-only server?"

## Simple Explanation

Think of moving into a new apartment block.

**IPv4 with DHCP:** you go to the front desk. The clerk writes down your name and gives you a flat number.

**IPv6 with SLAAC:** a notice on the wall says "This building is 2001:db8:acad:10". You choose your own flat number. You call out, "Does anyone already use this number?" If nobody answers, the number is yours. The building manager (the router) keeps no record of you.

**Neighbor Discovery:** in IPv4, when you need someone you shout in the hallway and everyone must stop and listen (ARP broadcast). In IPv6, each person also joins a small group based on the last digits of their flat number. You call only that small group.

**Where the analogy stops:** a flat number is unique in a building forever. An IPv6 host usually has many addresses at the same time, and some change every day on purpose. Also, the router still has a job in SLAAC: it announces the prefix and acts as the default gateway.

Three more terms before we start:

- **Prefix:** the network part of an address. In `2001:db8:acad:10::/64` the first 64 bits are the prefix.
- **Interface ID (IID):** the host part, the last 64 bits on a normal LAN.
- **Link:** one layer 2 network, such as one VLAN. Routers do not forward link-only traffic.

## How It Works

### 1. IPv6 Header vs IPv4 Header

The IPv6 base header is always 40 bytes. IPv4 is 20 to 60 bytes, because options can be added inside the header. A fixed size lets hardware find every field at a known position.

```text
IPv6 base header (RFC 8200), 40 bytes

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |              Flow Label               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Payload Length         |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                  Source Address (128 bits)                    +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                Destination Address (128 bits)                 +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

How each IPv4 field maps to IPv6:

| IPv4 field | IPv6 | Why |
|---|---|---|
| Version (4) | Version (6) | Same field, new value |
| IHL (header length) | Removed | Header size is fixed |
| DSCP and ECN | Traffic Class | Same QoS (traffic priority) job, same 8 bits |
| Total Length | Payload Length | Counts everything after the 40-byte base header |
| Identification, Flags, Fragment Offset | Moved to Fragment header | Used only when needed |
| TTL | Hop Limit | New name for what it really does: count hops |
| Protocol | Next Header | Points to a transport or an extension header |
| Header Checksum | Removed | Layer 2 and the transport layer already check errors |
| Options | Extension headers | Sit after the base header, not inside it |
| (none) | Flow Label | New, 20 bits, marks packets of one flow |

Key differences to say in an interview:

- **No header checksum.** In IPv4, each router must recalculate the checksum, because TTL changes at every hop. IPv6 removes that work. To cover the gap, the UDP checksum is mandatory in IPv6. It is optional in IPv4. (A narrow exception exists for some tunnels, but the normal rule is mandatory.)
- **Routers do not fragment.** Only the source host can fragment. A router that gets a packet too big for the next link drops it and sends ICMPv6 Packet Too Big.
- **Minimum link MTU is 1280 bytes.** MTU (Maximum Transmission Unit) is the largest packet a link can carry. IPv4's minimum is 68 bytes.
- **No broadcast.** IPv6 uses multicast (one sender, a group of receivers) instead.
- **Addresses are 4 times longer, but the header is only 2 times longer** than a basic IPv4 header.

**Flow Label:** a source sets the same value on all packets of one flow. Routers and load balancers can hash on it to keep a flow on one path, without reading deeper into the packet. Many networks still hash on the address and port fields, so support varies.

**Jumbograms:** the Payload Length field is 16 bits, so the normal limit is 65,535 bytes. RFC 2675 defines a Hop-by-Hop option for larger packets. It is rarely used.

### 2. Extension Headers

Optional information lives in separate headers placed between the base header and the transport header. Each header has a Next Header field that points to the next one, like a chain.

Analogy: a parcel with extra forms stuck on it. Most post offices read only the address label. Only the final receiver opens the forms. **Where it stops:** real parcels do not get dropped for having forms on them, but real IPv6 packets often do (see below).

```text
+----------------+----------------+-----------------+--------------+
| IPv6 header    | Routing header | Fragment header | TCP segment  |
| Next Header=43 | Next Header=44 | Next Header=6   | (data)       |
+----------------+----------------+-----------------+--------------+
```

| Next Header value | Meaning | Read by |
|---|---|---|
| 0 | Hop-by-Hop Options | Routers on the path, if configured to read it |
| 43 | Routing | The listed nodes |
| 44 | Fragment | Destination |
| 50 | ESP (IPsec encryption) | Destination |
| 51 | AH (IPsec authentication) | Destination |
| 60 | Destination Options | Destination |
| 59 | No Next Header | Nothing follows |
| 6 / 17 / 58 | TCP / UDP / ICMPv6 | Upper layer, end of chain |

The recommended order (RFC 8200) is:

1. IPv6 header
2. Hop-by-Hop Options
3. Destination Options (only for nodes listed in a Routing header)
4. Routing
5. Fragment
6. Authentication Header
7. ESP
8. Destination Options (for the final destination)
9. Upper-layer header

Details worth knowing:

- **Hop-by-Hop:** the only one meant for routers on the path. MLD (see section 3) uses it to carry the Router Alert option. RFC 8200 says routers may ignore it, and many process it slowly or drop it.
- **Routing header:** Type 0 was deprecated by RFC 5095, because it allowed traffic amplification between two hosts. Type 4 is the Segment Routing Header (SRv6, RFC 8754), used by some service provider and data centre networks.
- **Fragment header:** 8 bytes. It holds Next Header, a 13-bit Fragment Offset, an M (more fragments) flag and a 32-bit Identification. IPv4's ID is only 16 bits.

**Worked fragmentation example.** A host must send a 3,000-byte UDP datagram (UDP header included) over a path with MTU 1280.

- Room per fragment: 1280 - 40 (IPv6) - 8 (Fragment header) = 1232 bytes.
- Fragment data must be a multiple of 8 bytes, except in the last fragment. 1232 is 154 x 8, so it fits.

| Fragment | Data bytes | Offset (8-byte units) | M flag |
|---|---|---|---|
| 1 | 1232 | 0 | 1 |
| 2 | 1232 | 154 | 1 |
| 3 | 536 | 308 | 0 |

**The real-world problem:** many firewalls and routers drop packets with extension headers. RFC 7872 measured this on the Internet. Fragments are a special pain, because later fragments have no port numbers for a firewall to check. RFC 7112 requires the first fragment to hold the whole header chain, up to and including the upper-layer header. This is one reason DNS moved away from large UDP answers. DNS Flag Day 2020 suggested an EDNS buffer size of 1232 bytes to avoid fragmentation.

### 3. Neighbor Discovery Protocol (NDP) Replaces ARP

NDP (RFC 4861) runs on top of ICMPv6, inside normal IPv6 packets. ICMPv6 (RFC 4443) is the control and error protocol for IPv6. IPv4 needed a separate protocol, ARP, that is not even IP. IPv6 does it all in one protocol family.

| ICMPv6 type | Name | Sent by | Sent to | Purpose |
|---|---|---|---|---|
| 133 | RS (Router Solicitation) | Host | ff02::2 | "Any routers here?" |
| 134 | RA (Router Advertisement) | Router | ff02::1 or unicast | "Here is the prefix, and I am a gateway" |
| 135 | NS (Neighbor Solicitation) | Any node | Solicited-node group, or unicast | "Who has this address?" or "Are you still there?" |
| 136 | NA (Neighbor Advertisement) | Any node | Asker, or ff02::1 | "That address is mine, here is my MAC" |
| 137 | Redirect | Router | Host | "Use this better next hop" |

`ff02::1` is all nodes on the link. `ff02::2` is all routers on the link.

**Security detail:** NDP messages are sent with Hop Limit 255, and the receiver drops any with a lower value. A packet that crossed a router would have a lower value, so this stops remote attackers from faking NDP. It does not stop an attacker on the same link.

| | ARP (IPv4) | NDP (IPv6) |
|---|---|---|
| Runs over | Ethernet directly (EtherType 0x0806) | ICMPv6 over IPv6 (EtherType 0x86DD) |
| Request goes to | Broadcast, every host | Solicited-node multicast, few hosts |
| Finds routers | No | Yes (RS and RA) |
| Checks neighbour is alive | No standard way | Yes (NUD) |
| Duplicate check | Gratuitous ARP | DAD |
| Filtered by IP ACLs and firewalls | No, it sits below IP | Yes, it is normal IP traffic |

#### Solicited-node multicast

Each IPv6 address has a matching multicast group. Take the last 24 bits of the address and add them to `ff02::1:ff00:0/104`. The Ethernet MAC for an IPv6 multicast is `33:33` plus the last 32 bits of the group address.

```text
Address:     2001:db8:acad:10:5054:ff:fe12:3456
Last 24 bits:                            12:3456
Group:       ff02::1:ff12:3456
Dest MAC:    33:33:ff:12:34:56
```

Good switches use **MLD snooping** to send this group only to ports that joined. MLD (Multicast Listener Discovery) is how a host says "I want this group". Without snooping, the switch floods the frame, but the network cards of other hosts usually filter it out in hardware, so their CPUs are not disturbed.

**Fuller truth:** the group is not always one host. Two addresses with the same last 24 bits share a group. That is rare and harmless.

#### Address resolution: NS and NA

Here a host, A, wants the MAC of its gateway, 2001:db8:acad:10::1.

```text
Host A                                         Router (::1)
  |                                                |
  |  NS  dst ff02::1:ff00:1                        |
  |      dst MAC 33:33:ff:00:00:01                 |
  |      "Who has 2001:db8:acad:10::1?"            |
  |      option: my MAC is 52:54:00:12:34:56       |
  |----------------------------------------------->|
  |                                                |
  |  NA  dst Host A (unicast)                      |
  |      flags R=1 S=1 O=1                         |
  |      option: my MAC is 00:1a:2b:3c:4d:01       |
  |<-----------------------------------------------|
```

NA flags: R means the sender is a router. S means this answers a solicitation. O means override any cached entry.

#### Neighbor Unreachability Detection (NUD)

Hosts track each neighbour in a cache with these states:

```text
              NS sent                  NA received
 (no entry) ----------> INCOMPLETE ------------------> REACHABLE
                                                          |
                                     reachable time ends  |
                                                          v
 REACHABLE <--------- PROBE <-------- DELAY <--------- STALE
   confirmed         3 NS, 1s    5s    packet sent to this neighbour
```

- **REACHABLE:** confirmed recently. RFC 4861 base time is 30 seconds, randomised.
- **STALE:** not confirmed for a while, but still usable. No traffic is sent just to check it.
- **DELAY:** a packet was sent. Wait up to 5 seconds for an upper-layer hint, such as a TCP ACK.
- **PROBE:** send up to 3 unicast NS messages, 1 second apart. No answer means the entry is deleted.

A host sees a dead gateway quickly with this, and moves to another default router if it has one.

#### Redirect

Same idea as ICMP Redirect in IPv4. The router tells a host: "For that destination, use this other router on your link", or "That destination is on your link, talk directly."

### 4. Router Advertisements and Their Flags

Routers send RAs on a timer and in reply to an RS. Cisco IOS sends one about every 200 seconds by default. RFC 4861's default maximum interval is 600 seconds.

A router lifetime above 0 means "you may use me as a default gateway". A value of 0 means "do not use me as a default gateway, but the rest of this RA is valid". Cisco's default is 1800 seconds.

Important fields of an RA:

| Field | Meaning |
|---|---|
| Source address | Always the router's link-local address |
| M flag (Managed) | Get addresses from stateful DHCPv6 |
| O flag (Other) | Get other settings (such as DNS) from DHCPv6 |
| Router Lifetime | How long to use this router as a default gateway |
| Router Preference | High, Medium or Low (RFC 4191), chooses between routers |
| Hop Limit, MTU | Values hosts should use |
| PIO: prefix | Prefix Information Option, one per prefix |
| PIO: L flag | This prefix is on-link |
| PIO: A flag | Hosts may build a SLAAC address from this prefix |
| PIO: lifetimes | Valid lifetime and preferred lifetime |
| RDNSS | DNS server addresses (RFC 8106) |
| PREF64 | NAT64 prefix (RFC 8781) |

**The default gateway is always learned from an RA.** DHCPv6 has no default gateway option. This surprises people who come from IPv4.

### 5. SLAAC (Stateless Address Autoconfiguration)

"Stateless" means no server keeps a list of who has which address. RFC 4862 defines it.

```text
Host                                              Router
 |                                                   |
 | 1. Build link-local fe80::IID                     |
 | 2. DAD: NS to solicited-node, source ::           |
 |    (no reply in about 1 second, address is good)  |
 | 3. RS to ff02::2                                  |
 |-------------------------------------------------->|
 | 4. RA: prefix, flags, lifetime, DNS               |
 |<--------------------------------------------------|
 | 5. Build global address: prefix + IID             |
 | 6. DAD again for the global address               |
 | 7. Default route via router's fe80:: address      |
 | 8. If O=1 or M=1, start DHCPv6                    |
```

**DAD (Duplicate Address Detection):** before using an address, a host sends an NS for it from the unspecified address `::`. If someone replies with an NA, the address is taken. While DAD runs, the address is **tentative** and cannot be used for normal traffic. Hosts send one probe by default. Optimistic DAD (RFC 4429) lets a host start using the address sooner, for faster connections.

**Why /64 matters:** SLAAC needs prefix length + IID length = 128 bits, and the IID is 64 bits on Ethernet. A /64 works. A /62 or /80 will not produce a SLAAC address.

**Address lifetimes:**

```text
 tentative --> preferred ------------> deprecated ---------> invalid
   (DAD)      any traffic             existing flows only    removed
                      |                          |
                preferred lifetime          valid lifetime
                     ends                        ends
```

RFC 4861 default values are 7 days preferred and 30 days valid. Cisco IOS uses the same values by default. Short lifetimes make renumbering faster.

### 6. EUI-64

Modified EUI-64 is one way to build a 64-bit IID from a 48-bit MAC address. EUI stands for Extended Unique Identifier.

Steps, using MAC `00:1A:2B:3C:4D:5E`:

1. Split the MAC in half: `00:1A:2B` and `3C:4D:5E`.
2. Insert `FF:FE` in the middle: `00:1A:2B:FF:FE:3C:4D:5E`.
3. Flip the 7th bit from the left (the Universal/Local bit).

```text
First byte: 00 hex = 0000 0000
                            ^ 7th bit
Flip it:         = 0000 0010 = 02 hex

Result IID:   021A:2BFF:FE3C:4D5E
Short form:   21a:2bff:fe3c:4d5e
Link-local:   fe80::21a:2bff:fe3c:4d5e
Global:       2001:db8:10:20:21a:2bff:fe3c:4d5e
```

Quick tip: the second hex digit changes by 2 (0 to 2, 2 to 0, 8 to A, 4 to 6, and so on). A real example: `52` becomes `50`.

**Problem:** the MAC address stays the same on every network, so a server on the Internet can track the device as it moves. For that reason, host operating systems moved away from it. EUI-64 is still common on Cisco router interfaces and in some small devices.

### 7. Privacy: Stable and Temporary Addresses

There are two separate fixes. Most modern hosts use both.

| Method | RFC | How IID is made | Changes when |
|---|---|---|---|
| Modified EUI-64 | RFC 4291 | From MAC | Never (trackable) |
| Stable opaque IID | RFC 7217 | Hash of prefix, interface, secret | Only on a new network |
| Temporary address | RFC 8981 | Random | About daily |
| DHCPv6 lease | RFC 9915 | Chosen by server | Server policy |

- **Stable opaque (RFC 7217):** same address each time on one network, so firewall rules and logs keep working. It differs between networks, so nobody can follow the device across networks. RFC 8064 recommends this over EUI-64.
- **Temporary (RFC 8981):** random address used for outgoing connections. RFC 8981's defaults are 1 day preferred and 2 days valid. RFC 8981 replaced RFC 4941. Vendors may differ. Linux, for example, has long defaulted to 7 days valid.

Analogy: a stable address is your home address. A temporary address is a mail-forwarding box you replace daily, so shops cannot link your orders. **Where it stops:** the prefix (your street) still shows. For a home user, a /56 or /64 still points to one household.

Effect on operations: one host may have 3 or more IPv6 addresses at once. Logs show the temporary one, so tracking "who did this?" needs NDP cache data or 802.1X and switch logs. Some enterprises prefer DHCPv6 or disable temporary addresses for this reason, but note the Android limit in the next section.

**Which source address is used?** RFC 6724 sets rules for choosing a source and for sorting destinations. Default policy: IPv6 global is preferred over IPv4, but ULA (Unique Local Address, `fc00::/7`) ranks below IPv4. Hosts usually prefer temporary addresses for outgoing traffic, in line with RFC 8981 (Windows does, and Linux does when `use_tempaddr=2`). An update to the ULA ranking has been in progress in the IETF, so behaviour may change.

```bash
# Linux: address selection policy lives in this file
cat /etc/gai.conf
```

```powershell
# Windows: show the same policy table
Get-NetPrefixPolicy
```

### 8. DHCPv6: Stateful and Stateless

DHCPv6 (RFC 9915, which replaced RFC 8415 in January 2026) runs over UDP. Clients use port 546 and servers use port 547. Clients send to `ff02::1:2` (all DHCP relays and servers on the link) from their link-local address.

Clients are identified by a **DUID** (DHCP Unique Identifier), not by MAC. The DUID stays with the device, or the OS install, across network cards. One client can have many leases, each tied to an **IAID** (Identity Association Identifier).

General DHCP background is in [DHCP](../07-application-protocols/04-dhcp.md).

**Stateful DHCPv6:** the server hands out addresses and remembers them. Four messages, often remembered as SARR:

```text
Client (fe80::...)                          Server or Relay
 |  Solicit        to ff02::1:2                |
 |-------------------------------------------->|
 |  Advertise      "I can serve you"           |
 |<--------------------------------------------|
 |  Request        "I take that address"       |
 |-------------------------------------------->|
 |  Reply          address + lifetimes         |
 |<--------------------------------------------|
```

With Rapid Commit, it shrinks to Solicit and Reply. Other messages: Renew, Rebind, Release, Decline, Confirm, and Reconfigure.

**Stateless DHCPv6:** the host builds its address by SLAAC, then asks only for options, such as DNS server and domain. Two messages: Information-Request and Reply. The server keeps no lease.

**What the RA flags tell the host:**

| M | O | A (in PIO) | Result |
|---|---|---|---|
| 0 | 0 | 1 | SLAAC only. DNS from RDNSS in the RA |
| 0 | 1 | 1 | SLAAC address, DNS from stateless DHCPv6 |
| 1 | any | 0 | Stateful DHCPv6 only. M=1 means O does not matter |
| 1 | any | 1 | Both: SLAAC address and DHCPv6 address |

With no RA at all, a host keeps only its link-local address, even if a DHCPv6 server exists.

**Fuller truth:** flags are hints, not commands. Each OS decides what to do with them.

**Android warning:** Android has long not supported stateful DHCPv6 for addresses (IA_NA). A network with M=1 and A=0 leaves those phones without a global IPv6 address. This is a main reason to keep SLAAC on networks with phones.

**DHCPv6 Prefix Delegation (PD):** a router asks an upstream router for a whole prefix (IA_PD), then splits it for its LANs. This is how a home router gets its prefix.

```text
       IA_PD request               RA on LAN
ISP ------------------> CPE router ---------> Hosts
 gives 2001:db8:5500::/56           2001:db8:5500:1::/64
```

CPE means Customer Premises Equipment, the home or office edge router.

Status: temporary addresses in DHCPv6 (IA_TA) and server unicast are obsolete since RFC 9915. If you meet them, you are looking at older software or documents.

**Relay:** like IPv4, a router forwards client messages to a central server. The server tells which LAN the client is on by the link address in the relay header.

### 9. /48 and /64 Subnetting

The usual plan:

- An ISP or RIR (Regional Internet Registry) gets a /32 or larger.
- A site gets a /48 (or a /56 for a small site or home).
- Each LAN gets a /64.
- Point-to-point router links often use /127 (RFC 6164).
- Loopbacks use /128.

RFC 6177 does not fix one size for every site. In practice, many ISPs give homes a /56 or /60, and some still give only a /64 (which forces you to bridge, not route).

```text
 | 48 bits global prefix |16 subnet| 64 bits interface ID        |
 |2001 : 0db8 : acad     | 0010    | 0000 : 0000 : 0000 : 0001   |
 |<---- from provider -->|<- you ->|<---- host part ------------>|
```

How many /64 networks?

| Prefix | Number of /64s | Typical use |
|---|---|---|
| /32 | 4,294,967,296 | ISP allocation (65,536 /48s) |
| /48 | 65,536 | Enterprise site |
| /52 | 4,096 | Region or large campus part |
| /56 | 256 | Home or small branch |
| /60 | 16 | Small home |
| /64 | 1 | One LAN, 2^64 addresses |

Formula: subnets = 2^(new prefix - old prefix). Each hex digit is 4 bits (a **nibble**).

**Rule: stay on nibble boundaries** (/48, /52, /56, /60, /64) when you can. Then each subnet step is one whole hex digit, and you can read the plan directly. Also the reverse DNS tree in `ip6.arpa` works one hex digit at a time.

Splitting 2001:db8:acad::/48 into /50s is not on a nibble, so it is harder to read:

```text
2001:db8:acad:0000::/50    0000 to 3fff
2001:db8:acad:4000::/50    4000 to 7fff
2001:db8:acad:8000::/50    8000 to bfff
2001:db8:acad:c000::/50    c000 to ffff
```

Compare that to /52, where the first hex digit counts 0 to f. See the worked example below.

**Why not smaller than /64 on a LAN?** You lose SLAAC, some privacy features and things that assume /64. The waste does not matter. Even a /48 gives 65,536 LANs. Think in terms of "number of subnets", not "number of hosts".

**Exception, point-to-point links:** using /64 here allows a router to be attacked by NDP cache exhaustion (see Going Deeper). Reserve a /64 for the link in your plan, but configure a /127 on it.

### 10. Dual Stack

Dual stack means a device runs IPv4 and IPv6 side by side, with both addresses on one interface (RFC 4213 covers this basic idea). There is no translation. Each packet is pure IPv4 or pure IPv6.

Analogy: a person who speaks two languages and picks one per conversation. **Where it stops:** the person needs to learn both, and you still run two of everything. Two address plans, two routing tables, two sets of firewall rules.

```text
Host: 192.168.10.25 and 2001:db8:acad:10::25
  |
  |  DNS gives A and AAAA records
  |
  +-- IPv6 preferred --> IPv6 Internet --> Server
  +-- IPv4 fallback  --> NAT --> IPv4 Internet --> Server
```

Rules of dual stack:

- Enable IPv6 routing, then add an IPv6 address next to the IPv4 one.
- Publish AAAA records only when IPv6 really works. An AAAA record maps a name to an IPv6 address. See [DNS Record Types](../07-application-protocols/02-dns-records.md).
- Copy your security policy to IPv6. A host that speaks IPv6 can pass a firewall that has IPv4 rules only.
- Routing protocols need IPv6 too: OSPFv3 in [OSPF](../05-routing/06-ospf.md), and multiprotocol BGP in [BGP](../05-routing/09-bgp.md).

Dual stack has one weakness: it does not save IPv4 addresses. Each host still needs IPv4. That is why large networks now move to IPv6-only with translation, also called IPv4 as a Service.

### 11. Tunnels and 6to4

A tunnel puts an IPv6 packet inside an IPv4 packet, so it can cross an IPv4-only network. The carrier does not know about the inner packet.

Analogy: a letter in a foreign language placed inside a local envelope. Local mail staff read only the outer envelope. **Where it stops:** the outer packet adds bytes, so the path MTU shrinks, and a firewall cannot see inside without extra work.

```text
+-------------------+--------------------+-----------------+
| IPv4 header       | IPv6 header        | TCP and data    |
| Protocol = 41     | src, dst IPv6      |                 |
| src 203.0.113.5   |                    |                 |
| dst 198.51.100.2  |                    |                 |
+-------------------+--------------------+-----------------+
  20 bytes added: MTU 1500 becomes 1480 inside the tunnel
```

IP protocol number 41 means "IPv6 inside IPv4". Firewalls must allow it.

| Tunnel type | Key idea | Status in 2026 |
|---|---|---|
| Manual 6in4 (RFC 4213) | Fixed endpoints, protocol 41 | Still used, for example with tunnel brokers |
| GRE | Generic tunnel, protocol 47 | Common between routers |
| 6to4 (RFC 3056) | `2002::/16`, automatic, IPv4 built into the prefix | Avoid. Anycast relay prefix deprecated by RFC 7526 |
| 6rd (RFC 5969) | Like 6to4 but uses ISP's own prefix | Used by some ISPs |
| ISATAP (RFC 5214) | IPv4 inside IID, for inside a site | Old, removed or off in modern systems |
| Teredo (RFC 4380) | UDP port 3544, `2001::/32`, crosses NAT | Old, mostly off today |

**How 6to4 builds a prefix:** take a public IPv4 address, write it in hex, and add it after `2002:`.

```text
Public IPv4:  203.0.113.5
In hex:       cb . 00 . 71 . 05
6to4 prefix:  2002:cb00:7105::/48
```

Why 6to4 failed: return traffic from native IPv6 hosts had to find a relay. The anycast address 192.88.99.1 could lead to any relay (anycast means one address served from many places, see [Unicast, Broadcast, Multicast and Anycast](13-unicast-broadcast-multicast-anycast.md)). Relays were run by unknown parties, with no one responsible for their quality. Traffic could go one way through one relay and back another way. It often failed or was slow, so RFC 7526 deprecated the anycast relay prefix. Today it is best left off. If you see `2002:` addresses in a capture, it is usually an old device.

### 12. NAT64 and DNS64

NAT64 lets an IPv6-only client talk to an IPv4-only server. It translates between the two protocols (RFC 6146). DNS64 (RFC 6147) gives the client an IPv6 address to aim at, when the server has only IPv4.

Analogy: a phone call through an interpreter. You think the interpreter is the other person. **Where it stops:** an interpreter understands meaning. NAT64 only rewrites headers. An app that puts an IPv4 address inside its own data breaks.

The trick is an address format, RFC 6052. Place the IPv4 address in the last 32 bits of a /96 prefix:

```text
Well-known prefix:  64:ff9b::/96
IPv4 server:        203.0.113.80 = cb 00 71 50 hex
Synthesised:        64:ff9b::cb00:7150
Same thing:         64:ff9b::203.0.113.80
```

Rules: the well-known prefix `64:ff9b::/96` must not be used with private IPv4 addresses (RFC 6052). RFC 8215 sets aside `64:ff9b:1::/48` for local use. Many operators use their own prefix.

```text
 IPv6-only client        DNS64          NAT64          IPv4 server
 2001:db8:acad:10::99                 v4 198.51.100.10  203.0.113.80
     |                     |              |                |
     |-- AAAA? legacy.example.com ->|     |                |
     |                     | (no AAAA, has A)              |
     |<- AAAA 64:ff9b::cb00:7150 --|      |                |
     |                     |              |                |
     |-- IPv6 to 64:ff9b::cb00:7150 ----->|                |
     |                     |              |-- IPv4 ------->|
     |                     |              |  src 198.51.100.10
     |                     |              |<-- reply ------|
     |<----- IPv6 reply ------------------|                |
```

**Stateful vs stateless:** stateful NAT64 keeps a session table and shares IPv4 addresses, like PAT in [NAT and PAT](09-nat-and-pat.md). Stateless translation (SIIT, RFC 7915) maps addresses one to one and holds no sessions.

**What breaks:**

- Apps that use IPv4 literal addresses, such as `http://203.0.113.80/`. DNS is not used, so DNS64 cannot help.
- Protocols that carry IPv4 addresses in the data.
- DNSSEC-validating clients (DNSSEC means signed DNS). A synthesised AAAA is not signed, so a client that validates for itself will reject it.

**464XLAT (RFC 6877)** fixes literals. A small translator on the phone, called a CLAT, gives apps a fake IPv4 address and turns their IPv4 into IPv6. The network's NAT64 is called the PLAT. T-Mobile US is a well-known user of this design.

```text
App (IPv4) -> CLAT on phone -> IPv6 network -> PLAT (NAT64) -> IPv4 Net
```

**Discovering the prefix:** hosts can look up `ipv4only.arpa` (RFC 7050) or read the PREF64 option in RAs (RFC 8781).

**Other IPv4 as a Service options:** DS-Lite (RFC 6333) tunnels IPv4 in IPv6 to an ISP NAT. MAP-E (RFC 7597) and MAP-T (RFC 7599) give each customer a slice of a shared IPv4 address.

**IPv6-mostly:** RFC 8925 defines DHCPv4 Option 108. A host that can run without IPv4 tells the server so, and the server tells it to stop using IPv4. Hosts that need IPv4 still get it on the same network.

### 13. Happy Eyeballs

Problem: a host prefers IPv6, but the IPv6 path to a site is broken. Without help, the user waits for a TCP timeout before IPv4 is tried. That can take many seconds.

Happy Eyeballs (RFC 8305, version 2; the first version was RFC 6555) races the two families with a small head start for IPv6.

Analogy: you call a taxi company. If it does not answer quickly, you call a second one while the first keeps ringing. You take whichever answers first. **Where it stops:** a real second taxi costs someone a trip. Here the losing connection is simply closed.

The steps:

1. Ask DNS for AAAA and A at the same time. If A arrives first, wait a short time for AAAA. The **Resolution Delay** recommended value is 50 ms.
2. Sort addresses. Prefer IPv6 and mix in the other family.
3. Start connecting to the first address.
4. If no answer after the **Connection Attempt Delay** (recommended 250 ms), start the next attempt. Do not stop the first one.
5. Use the first connection that completes. Cancel the rest.

```text
time (ms)  event
  0        AAAA and A queries sent
 12        A answer arrives, wait for AAAA
 20        AAAA answer arrives, list: IPv6 first, then IPv4
 20        SYN to 2001:db8:10::80
270        No reply after 250 ms, SYN to 203.0.113.80
285        IPv4 SYN-ACK arrives, use IPv4, cancel IPv6 attempt
```

Real implementations use their own timers. For example, Go's default fallback delay is 300 ms and curl's default is 200 ms. Browsers and operating systems have their own logic.

Version 3 is being developed in the IETF (it adds HTTPS DNS records and QUIC). It was still an Internet-Draft in 2026, so check the current status.

**Warning:** Happy Eyeballs hides broken IPv6. Users do not complain, so your monitoring must test IPv6 on its own, with `curl -6` or a similar probe.

### 14. Security Basics Specific to IPv6

| Threat | Fix |
|---|---|
| Rogue RA (fake router) | RA Guard on access ports (RFC 6105) |
| Fake DHCPv6 server | DHCPv6 Guard |
| Fake NA (like ARP spoofing) | ND Inspection, Source Guard |
| NDP cache exhaustion | /127 on links, rate limits, ACL on unused space |
| IPv6 past IPv4-only firewall | Same rules in both families |

SEND (Secure Neighbor Discovery, RFC 3971) signs NDP messages but is rarely used.

**Do not block all ICMPv6.** A host needs NS, NA, RS and RA on the link. All paths need Packet Too Big, or PMTUD (Path MTU Discovery) breaks. RFC 4890 lists what to allow in firewalls. More on [MTU and PMTUD](../06-transport-layer/09-mss-mtu-and-pmtud.md) and on the wider tools in [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).

## Example

### Example 1: Dual-Stack LAN With SLAAC and Stateless DHCPv6

Network:

```text
                       Internet
                          |
                 2001:db8:ffff:1::2/64
                  203.0.113.2/30
                        [R1] G0/0
                          |  G0/1
                 2001:db8:acad:10::1/64, fe80::1
                  192.168.10.1/24
                          |
                       [Switch]
                       /      \
                  [Linux]    [Windows]
```

**Router (Cisco IOS):**

```text
ipv6 unicast-routing
!
ipv6 dhcp pool LAN10-INFO
 dns-server 2001:db8:acad:1::53
 domain-name example.com
!
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.252
 ipv6 address 2001:db8:ffff:1::2/64
!
interface GigabitEthernet0/1
 ip address 192.168.10.1 255.255.255.0
 ipv6 address fe80::1 link-local
 ipv6 address 2001:db8:acad:10::1/64
 ipv6 nd other-config-flag
 ipv6 dhcp server LAN10-INFO
 no shutdown
!
ipv6 route ::/0 2001:db8:ffff:1::1
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

`ipv6 unicast-routing` is required. Without it, the router does not forward IPv6 or send RAs. The DNS server `2001:db8:acad:1::53` sits on a server VLAN that is not shown here.

**What happens when the Linux host comes up.** Its MAC is `52:54:00:12:34:56` and it uses EUI-64 in this lab:

```text
$ sudo tcpdump -ni eth0 icmp6
IP6 :: > ff02::1:ff12:3456: ICMP6, neighbor solicitation,
    who has fe80::5054:ff:fe12:3456, length 24
IP6 fe80::5054:ff:fe12:3456 > ff02::2: ICMP6, router solicitation,
    length 16
IP6 fe80::1 > ff02::1: ICMP6, router advertisement, length 64
IP6 :: > ff02::1:ff12:3456: ICMP6, neighbor solicitation,
    who has 2001:db8:acad:10:5054:ff:fe12:3456, length 24
```

(Real tcpdump prints each entry on one line. Wrapped here to fit. MLD reports are left out.)

Note the second DAD uses the same multicast group as the first, because both addresses share the last 24 bits.

**Linux results:**

```text
$ ip -6 addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2001:db8:acad:10:5054:ff:fe12:3456/64 scope global dynamic
       valid_lft 2591981sec preferred_lft 604781sec
    inet6 fe80::5054:ff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever

$ ip -6 route
2001:db8:acad:10::/64 dev eth0 proto ra metric 100 pref medium
fe80::/64 dev eth0 proto kernel metric 1024 pref medium
default via fe80::1 dev eth0 proto ra metric 100 pref medium

$ ip -6 neigh
fe80::1 dev eth0 lladdr 00:1a:2b:3c:4d:01 router REACHABLE
```

The default gateway is a link-local address. That is normal.

**Checking the RA with `rdisc6` (Linux, from the `ndisc6` package):**

```text
$ rdisc6 eth0
Soliciting ff02::2 (ff02::2) on eth0...

Hop limit                 :           64 (      0x40)
Stateful address conf.    :           No
Stateful other conf.      :          Yes
Router preference         :       medium
Router lifetime           :         1800 (0x00000708) seconds
 Prefix                   : 2001:db8:acad:10::/64
  On-link                 :          Yes
  Autonomous address conf.:          Yes
  Valid time              :      2592000 (0x00278d00) seconds
  Pref. time              :       604800 (0x00093a80) seconds
 MTU                      :         1500 (0x000005dc) bytes
 Source link-layer address: 00:1A:2B:3C:4D:01
 from fe80::1
```

"Stateful other conf: Yes" is the O flag. The host will now send an Information-Request for DNS.

**Windows results:**

```text
PS> ipconfig

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : example.com
   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:10:8d2f:1b7e:44c1:9a02
   Temporary IPv6 Address. . . . . . : 2001:db8:acad:10:3c91:e5a0:6f12:b7d4
   Link-local IPv6 Address . . . . . : fe80::8d2f:1b7e:44c1:9a02%12
   IPv4 Address. . . . . . . . . . . : 192.168.10.25
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : fe80::1%12
                                       192.168.10.1
```

The `%12` is a **zone ID**. It says which interface to use. A link-local address exists on each link, so the OS needs this hint. Windows uses the interface index. Linux uses the name: `ping fe80::1%eth0`.

**Router view:**

```text
R1#show ipv6 interface GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::1
  Global unicast address(es):
    2001:DB8:ACAD:10::1, subnet is 2001:DB8:ACAD:10::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:2
    FF02::1:FF00:1
  MTU is 1500 bytes
  ND DAD is enabled, number of DAD attempts: 1
  ND reachable time is 30000 milliseconds (using 30000)
  ND router advertisements are sent every 200 seconds
  ND router advertisements live for 1800 seconds
  ND advertised default router preference is Medium
  Hosts use stateless autoconfig for addresses.
  Hosts use DHCP to obtain other configuration.

R1#show ipv6 neighbors
IPv6 Address                             Age Link-layer Addr State Interface
2001:DB8:ACAD:10:5054:FF:FE12:3456         0 5254.0012.3456  REACH Gi0/1
FE80::5054:FF:FE12:3456                    2 5254.0012.3456  STALE Gi0/1
```

(Output is shortened to the useful lines.) `FF02::1:FF00:1` serves both `::1` addresses, as they share the last 24 bits.

### Example 2: Address Plan for a /48

You get `2001:db8:acad::/48` for a company with 3 regions, each with up to 16 sites, each with up to 256 VLANs. You have 16 bits of subnet ID.

```text
Bits:   4        4       8
      region    site    VLAN number
      (hex)     (hex)   (hex)

2001:db8:acad: R   S   VV ::/64
```

Plan:

| Level | Prefix | Count at this level |
|---|---|---|
| Region 1 | `2001:db8:acad:1000::/52` | 4,096 /64s |
| Site 3 in region 1 | `2001:db8:acad:1300::/56` | 256 /64s |
| VLAN 20 (hex 0x14) | `2001:db8:acad:1314::/64` | 1 LAN |
| VLAN 20 with decimal style | `2001:db8:acad:1320::/64` | Reads as "20" |

Many teams use the decimal-style method, so the VLAN number can be read in the address, as long as VLAN numbers stay at or below 99. Choose one method and write it down.

Reserve one region for infrastructure:

```text
2001:db8:acad:f000::/52        Infrastructure
2001:db8:acad:f000::/64        Router loopbacks, each a /128
2001:db8:acad:f100::/56        Links: reserve /64, configure /127
2001:db8:acad:f100::/127       R1 = ...f100::, R2 = ...f100::1
```

### Example 3: Stateful DHCPv6 With a Relay (Cisco IOS)

```text
! Server router
ipv6 dhcp pool LAN20-STATEFUL
 address prefix 2001:db8:acad:20::/64 lifetime 86400 43200
 dns-server 2001:db8:acad:1::53
 domain-name example.com
!
interface GigabitEthernet0/2
 ipv6 address 2001:db8:acad:1::1/64
 ipv6 dhcp server LAN20-STATEFUL
```

```text
! Router with the clients on its LAN
interface GigabitEthernet0/1
 ipv6 address 2001:db8:acad:20::1/64
 ipv6 nd managed-config-flag
 ipv6 nd prefix 2001:db8:acad:20::/64 no-autoconfig
 ipv6 dhcp relay destination 2001:db8:acad:1::1
```

`no-autoconfig` clears the A flag, so hosts do not also build SLAAC addresses. Then Android devices on this VLAN get no global IPv6 address, as covered earlier.

Verify:

```text
R1#show ipv6 dhcp pool
R1#show ipv6 dhcp binding
R1#show ipv6 dhcp interface
```

### Example 4: NAT64 Test With DNS64

From an IPv6-only Linux client whose DNS resolver does DNS64:

```text
$ dig +short A legacy.example.com
203.0.113.80

$ dig +short AAAA legacy.example.com
64:ff9b::cb00:7150

$ ping -c 1 64:ff9b::203.0.113.80
PING 64:ff9b::203.0.113.80(64:ff9b::cb00:7150) 56 data bytes
64 bytes from 64:ff9b::cb00:7150: icmp_seq=1 ttl=52 time=22.4 ms
```

The same setup on Cisco IOS XE, on platforms that support NAT64 (commands and support vary by platform and licence):

```text
interface GigabitEthernet0/0/0
 description IPv6-only LAN
 ipv6 address 2001:db8:acad:10::1/64
 nat64 enable
!
interface GigabitEthernet0/0/1
 description IPv4 uplink
 ip address 198.51.100.2 255.255.255.0
 nat64 enable
!
nat64 prefix stateful 64:ff9b::/96
ipv6 access-list NAT64-ACL
 permit ipv6 2001:db8:acad:10::/64 any
nat64 v4 pool NAT64-POOL 198.51.100.10 198.51.100.20
nat64 v6v4 list NAT64-ACL pool NAT64-POOL overload
```

```text
Router#show nat64 translations
Router#show nat64 statistics
```

### Example 5: A 6in4 Tunnel

Cisco IOS side:

```text
interface Tunnel0
 ipv6 address 2001:db8:ffff:2::1/64
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.2
 tunnel mode ipv6ip
!
ipv6 route ::/0 Tunnel0
```

Linux side (the far end, 198.51.100.2):

```bash
# Linux: sit is the kernel's IPv6-in-IPv4 tunnel type
sudo ip tunnel add tun6 mode sit remote 203.0.113.2 local 198.51.100.2 ttl 64
sudo ip link set tun6 up mtu 1480
sudo ip -6 addr add 2001:db8:ffff:2::2/64 dev tun6

# Test: ping the Cisco end of the tunnel
ping -6 -c 1 2001:db8:ffff:2::1
```

## Command Reference

### Configuration

| Task | Cisco IOS |
|---|---|
| Enable routing | `ipv6 unicast-routing` |
| Static address | `ipv6 address 2001:db8:acad:10::1/64` |
| EUI-64 address | `ipv6 address 2001:db8:acad:10::/64 eui-64` |
| Link-local only | `ipv6 enable` |
| Set link-local | `ipv6 address fe80::1 link-local` |
| SLAAC client | `ipv6 address autoconfig` |
| DHCPv6 client | `ipv6 address dhcp` |
| Static route via global | `ipv6 route ::/0 2001:db8:ffff:1::1` |
| Static route via link-local | `ipv6 route ::/0 GigabitEthernet0/0 fe80::1` |
| Set M flag | `ipv6 nd managed-config-flag` |
| Set O flag | `ipv6 nd other-config-flag` |
| Stop RAs on an interface | `ipv6 nd ra suppress all` |
| Stop being default gateway | `ipv6 nd ra lifetime 0` |
| Set router preference | `ipv6 nd router-preference High` |

A next hop that is link-local needs the exit interface, because the address exists on every link. Details are in [Static and Default Routes](../05-routing/03-static-and-default-routes.md). Newer IOS XE releases can also place RDNSS in the RA, and the command varies by release.

**Linux static config:**

```bash
sudo ip -6 addr add 2001:db8:acad:20::10/64 dev eth0
sudo ip -6 route add default via fe80::1 dev eth0
```

**Windows static config:**

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 2001:db8:acad:20::10 -PrefixLength 64 -DefaultGateway 2001:db8:acad:20::1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 2001:db8:acad:1::53
```

**Linux as a router sending RAs (radvd, `/etc/radvd.conf`):**

```text
interface eth1 {
    AdvSendAdvert on;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    prefix 2001:db8:acad:20::/64 {
        AdvOnLink on;
        AdvAutonomous on;
    };
    RDNSS 2001:db8:acad:1::53 { };
};
```

**Linux settings that often matter:**

```bash
sysctl net.ipv6.conf.all.forwarding      # 1 = this host routes
sysctl net.ipv6.conf.eth0.accept_ra      # 1 = accept RAs, 2 = even if routing
sysctl net.ipv6.conf.eth0.use_tempaddr   # 0 off, 1 on, 2 prefer temporary
sysctl net.ipv6.conf.eth0.addr_gen_mode  # 0 EUI-64, 2 or 3 stable privacy
```

A Linux host with `forwarding=1` ignores RAs unless `accept_ra=2`. Defaults for these values vary by distribution and network manager.

### Verification

| Goal | Linux | Windows | Cisco IOS |
|---|---|---|---|
| Show addresses | `ip -6 addr` | `ipconfig /all` or `Get-NetIPAddress -AddressFamily IPv6` | `show ipv6 interface brief` |
| Show routes | `ip -6 route` | `Get-NetRoute -AddressFamily IPv6` | `show ipv6 route` |
| Neighbour cache | `ip -6 neigh` | `Get-NetNeighbor -AddressFamily IPv6` | `show ipv6 neighbors` |
| Ping | `ping -6 2001:db8:acad:10::1` | `ping -6 2001:db8:acad:10::1` | `ping ipv6 2001:db8:acad:10::1` |
| Trace | `traceroute -6 example.com` | `tracert -6 example.com` | `traceroute ipv6 2001:db8:ffff:1::1` |
| Ping link-local | `ping fe80::1%eth0` | `ping fe80::1%12` | `ping fe80::1` then enter interface |
| See RAs | `rdisc6 eth0` | capture with Wireshark | `debug ipv6 nd` |
| Clear cache | `ip -6 neigh flush dev eth0` | `netsh interface ipv6 delete neighbors` | `clear ipv6 neighbors` |
| Test HTTP over v6 | `curl -6 -I https://example.com` | `curl.exe -6 -I https://example.com` | N/A |
| Privacy state | `sysctl ...use_tempaddr` | `Get-NetIPv6Protocol` | N/A |

Capture NDP and DHCPv6 on Linux:

```bash
# All ICMPv6
sudo tcpdump -ni eth0 icmp6

# Only NDP, when no extension headers exist (byte 40 is the ICMPv6 type)
sudo tcpdump -ni eth0 'icmp6 and ip6[40] >= 133 and ip6[40] <= 137'

# DHCPv6
sudo tcpdump -ni eth0 'udp port 546 or udp port 547'
```

More on capture filters is in [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md). A longer Cisco list is in [Cisco IOS Command Reference](../12-troubleshooting-and-tools/10-cisco-ios-commands.md).

### Troubleshooting Order

```text
1. Link-local present?       No  -> interface down, IPv6 disabled
2. Global address present?   No  -> no RA reaching host, or A=0,
                                    or M=1 with no DHCPv6
3. Default route present?    No  -> no RA, or Router Lifetime = 0
4. Ping gateway (fe80::)?    No  -> L2, VLAN, RA Guard, NDP blocked
5. Ping a remote IPv6?       No  -> routing, ISP, firewall
6. DNS gives AAAA?           No  -> DNS server has no AAAA, wrong DNS
7. TCP works, big data hangs -> MTU, Packet Too Big filtered
```

## Going Deeper

### PMTUD and Why It Matters More in IPv6

Routers cannot fragment, so the sender must learn the path MTU. Without ICMPv6 Packet Too Big (type 2), a large packet vanishes and nothing tells the host. TCP handshakes work (they use small packets), then data transfer hangs. This is called a PMTUD black hole. Tunnels make it more common. Fixes: allow ICMPv6 type 2, clamp TCP MSS (the largest TCP data size) on tunnels, or use packetization-layer discovery (RFC 4821).

### ICMPv6 Types to Remember

| Type | Message |
|---|---|
| 1 | Destination Unreachable |
| 2 | Packet Too Big |
| 3 | Time Exceeded |
| 4 | Parameter Problem |
| 128 / 129 | Echo Request / Echo Reply |
| 130-132, 143 | MLD |
| 133-137 | NDP |

More in [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

### NDP Cache Exhaustion

A /64 has 2^64 addresses. An attacker sends packets to thousands of random addresses in your LAN prefix. The router creates an INCOMPLETE entry for each and sends an NS. The cache fills, and real hosts may not get resolved.

Protection: rate limits, cache size limits per interface, ACLs that allow only real addresses to reach infrastructure links, and /127 on point-to-point links. The huge space also helps the other side: an attacker cannot scan a /64 from end to end at any normal rate, so scanning shifts to DNS names, logs and predictable addresses such as `::1`.

### Multiple Prefixes and Multihoming

A host can have addresses from many prefixes at once. The same interface can hold a ULA, a global address from ISP A and a global address from ISP B. Then source address selection (RFC 6724) and routing decide which one is used. A wrong pair can hit ingress filtering, where an ISP drops packets whose source is not in its own ranges. Enterprises often solve this with their own Provider Independent (PI) space and BGP. Without that, NPTv6 (RFC 6296) rewrites the prefix 1:1 and does not track connections, but it is experimental and not the normal design.

### ULA

ULA (RFC 4193) is `fc00::/7`. In practice, it uses `fd00::/8` with a 40-bit random Global ID, for example `fd12:3456:789a::/48`. Think of it as a private range, like 10.0.0.0/8. It is not routed on the Internet. The random part means two companies that merge are unlikely to clash. Use it for internal-only systems, or alongside global addresses.

### No NAT, But What About Security?

IPv6 does not need NAT to save addresses. A stateful firewall gives the same inbound protection: allow outgoing, block new incoming. NAT only hid addresses as a side effect. See [Firewalls](../09-security/04-firewalls.md).

### Reverse DNS

IPv6 PTR records sit under `ip6.arpa`, one label for each hex digit, in reverse order. The address `2001:db8::1` becomes `1.0.0.0...8.b.d.0.1.0.0.2.ip6.arpa` with 32 labels in full. See [DNS Record Types](../07-application-protocols/02-dns-records.md).

### Cloud Note

Cloud platforms offer IPv6 with their own rules. AWS, for example, gives a VPC a /56 and each subnet a /64, and uses an egress-only internet gateway to allow outbound IPv6 only. See [AWS VPC Basics](../11-cloud-and-containers/03-aws-vpc-basics.md).

### Transition Options at a Glance

| Method | IPv4 still in LAN? | Good for |
|---|---|---|
| Dual stack | Yes | Most enterprises today |
| 6in4 or GRE tunnel | Yes | Reaching IPv6 across IPv4 |
| NAT64 + DNS64 | No | IPv6-only networks |
| 464XLAT | Not in the network, but apps can use IPv4 | Mobile, full app support |
| DS-Lite, MAP | Inside tunnel or translated | ISPs short of IPv4 |

## Common Mistakes

- **"DHCPv6 gives the default gateway."** It does not. Only RAs do. If RAs are blocked, the host has no IPv6 gateway, even with a DHCPv6 lease.
- **Blocking all ICMPv6, as was done for ICMP in IPv4.** This breaks NDP, SLAAC and PMTUD.
- **Using /120 or /112 on a LAN to "save addresses".** SLAAC stops, and you gain nothing.
- **Forgetting `ipv6 unicast-routing`.** The interface has an address, but the router sends no RAs and forwards nothing.
- **Thinking a link-local gateway is a bug.** It is normal, and it survives a global renumbering.
- **Using a link-local next hop with no interface.** The router cannot tell which link is meant.
- **Writing an IPv6 ACL with `deny ipv6 any any` and forgetting NDP.** On Cisco, IPv6 ACLs include hidden permits for NS and NA before the final hidden deny. A manual `deny ipv6 any any` at the end removes them. See [Access Control Lists](../09-security/05-access-control-lists.md).
- **Thinking EUI-64 is the default on modern hosts.** Windows, macOS, iOS and many Linux setups use random or hashed IIDs. Do not expect to find the MAC in the address.
- **Treating a /64 like a small IPv4 subnet.** Scanning it fully is not practical, and host counts are not a design concern.
- **Assuming 6to4 is a recommended transition tool.** It is obsolete in practice. Use native IPv6, 6rd or tunnels with named endpoints.
- **Disabling IPv6 on hosts "to be safe".** One missed host, and a rogue RA can still give it a hidden IPv6 path. Microsoft also advises against disabling IPv6 on Windows, as some components can break. The better fix is RA Guard and firewall rules for IPv6.
- **Saying "ping works, so IPv6 is fine".** Small packets can pass where large ones fail. Test with real traffic.
- **Thinking NAT64 needs no DNS change.** Without DNS64 or a known prefix, the client has no IPv6 address to use.
- **Forgetting IPv6 in firewall rules.** IPv4 rules do not apply to IPv6 packets.

## Interview Questions

**Q: What are the main differences between the IPv6 and IPv4 headers?**
A: The IPv6 header is a fixed 40 bytes, while IPv4 is 20 to 60. IPv6 has no header checksum, no fragmentation fields and no options inside the header. Options moved to extension headers, and fragmentation moved to a Fragment header used only by the source. TTL became Hop Limit, Protocol became Next Header, and a 20-bit Flow Label was added.

**Q: What replaced ARP in IPv6, and how does it work?**
A: Neighbor Discovery, which runs over ICMPv6. To find a MAC, a host sends a Neighbor Solicitation to the solicited-node multicast group of the target, built from the last 24 bits of the address. The target replies with a Neighbor Advertisement with its MAC. NDP also handles router discovery, duplicate address checks and neighbour reachability.

**Q: Walk me through SLAAC.**
A: The host builds a link-local address and runs Duplicate Address Detection on it. It sends a Router Solicitation to ff02::2. The router replies with a Router Advertisement with the prefix and flags. The host adds its own 64-bit interface ID to the prefix, runs DAD again, and uses the router's link-local address as its gateway. No server tracks the address.

**Q: What do the M and O flags do?**
A: They are in the RA and tell hosts how to use DHCPv6. M=1 means get addresses from stateful DHCPv6. O=1 means get other settings, such as DNS, from DHCPv6, usually with a SLAAC address. Both zero means SLAAC only, with DNS from the RDNSS option. Separately, the A flag in each prefix allows SLAAC with that prefix.

**Q: What is the difference between stateful and stateless DHCPv6?**
A: Stateful DHCPv6 gives out addresses and tracks leases, using Solicit, Advertise, Request and Reply. Stateless DHCPv6 gives only options, such as DNS, using Information-Request and Reply, and the address comes from SLAAC. Neither gives a default gateway. That still comes from RAs.

**Q: How does EUI-64 work, and why is it avoided on hosts?**
A: It splits the 48-bit MAC in half, inserts FFFE, and flips the seventh bit. That creates a 64-bit interface ID. It is avoided on hosts because the same ID follows the device between networks, so it can be tracked. Hosts now use stable opaque IDs from RFC 7217 and temporary addresses from RFC 8981.

**Q: You have a /48. How many /64s, and how would you plan them?**
A: 65,536, because there are 16 subnet bits. I would stay on hex digit boundaries, so one digit for region, one for site and two for VLAN, and reserve one block for infrastructure. Each LAN gets a /64, loopbacks get /128, and point-to-point links get a reserved /64 with a /127 configured.

**Q: Why is 6to4 no longer recommended?**
A: It depended on anycast relays run by unknown parties. Return paths were often broken or slow, and nobody was responsible for them. RFC 7526 deprecated the anycast relay prefix. Today you use native IPv6, 6rd if your ISP offers it, or a tunnel with known endpoints.

**Q: How does an IPv6-only client reach an IPv4-only website?**
A: With DNS64 and NAT64. The client asks for AAAA. The DNS64 resolver finds only an A record, and creates an AAAA by embedding the IPv4 address in a prefix such as 64:ff9b::/96. The client sends IPv6 to that address. The NAT64 gateway reads the IPv4 address out of the last 32 bits, translates the packet and keeps session state. Apps that use IPv4 literals need 464XLAT.

**Q: What is Happy Eyeballs, and why is it needed?**
A: It stops broken IPv6 from causing long delays on dual-stack hosts. The client gets A and AAAA records, tries IPv6 first, and if there is no answer in a short time (250 ms recommended in RFC 8305) it also tries IPv4. The first connection to succeed wins. The side effect is that it hides IPv6 failures, so you need to monitor IPv6 on its own.

**Q: A user has an IPv6 address, but cannot reach anything off the LAN. What do you check?**
A: First, is there a default route? If the host has no route, RAs may be missing or the Router Lifetime may be 0. Then I ping the gateway's link-local address with the zone ID, and check the neighbour cache. Next I test a remote IPv6 address to separate routing from DNS. I also check for a rogue RA, and for firewall rules on ICMPv6 and IPv6 traffic.

**Q: Why should ICMPv6 not be fully blocked?**
A: NDP runs on ICMPv6, so blocking it on a LAN stops address resolution, SLAAC and gateway discovery. Routers do not fragment in IPv6, so hosts need Packet Too Big messages to find the path MTU. If those are blocked, connections hang on large packets. Filter by type, as RFC 4890 advises.

## Quick Recap

- The IPv6 header is a fixed 40 bytes, with no checksum and no router fragmentation. Extras go in extension headers, which some networks drop.
- NDP over ICMPv6 replaces ARP: RS and RA for routers, NS and NA for neighbours, plus DAD and NUD. It uses solicited-node multicast, not broadcast.
- SLAAC builds an address from an RA prefix and needs a /64. Gateways come only from RAs.
- DHCPv6 stateful gives addresses, and stateless gives only options. M, O and A flags steer the host. Android does not use stateful DHCPv6 for addresses.
- Modern hosts avoid EUI-64 and use stable opaque IIDs and temporary addresses.
- A /48 holds 65,536 /64s. Plan on nibble boundaries, and use /127 on router links.
- Dual stack is the common start. NAT64 with DNS64 (and 464XLAT) serves IPv6-only networks. 6to4 is obsolete in practice. Happy Eyeballs hides failures, so monitor IPv6 on its own.
