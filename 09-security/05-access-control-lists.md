# Access Control Lists (ACLs)

**Level:** Intermediate | **Prerequisites:** [Subnet Masks and CIDR](../04-ip-addressing/04-subnet-masks-and-cidr.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [Firewalls](04-firewalls.md)

## In One Line

An ACL is an ordered list of permit and deny rules that a router or switch checks against each packet, from the top down, to decide whether the packet passes.

## Why It Matters

A router forwards every packet it has a route for. It does not ask whether the traffic should be there. Without a filter, a guest laptop can reach the payroll server, as long as a route exists.

An ACL adds the missing question: "Is this packet allowed?"

ACLs solve real problems:

- Stop the guest network from reaching internal servers.
- Allow SSH to a router only from the IT team's PC.
- Drop fake source addresses at the Internet edge.
- Select traffic for other features, such as NAT, QoS and VPNs.

Interviewers like ACLs because one small topic tests many skills at once: binary (wildcard masks), Layer 3 and Layer 4 headers, packet flow through a router, TCP flags, and careful logic. A single question such as "Why did my ACL lock everyone out?" shows quickly who has really used them.

## Simple Explanation

Think of a guard at a gate who holds a printed list.

- Each person who arrives is checked against the list, starting at line 1.
- The guard stops at the first line that fits and does what it says: let in, or turn away.
- The guard never reads the rest of the list after that.
- At the very bottom is a rule that nobody printed: "If you are not on this list, you do not enter."

That is an ACL. The packet is the person. The list is the ACL. The invisible last rule is called the **implicit deny**.

Where the analogy stops being true:

- A real guard remembers faces. A basic ACL remembers nothing. Each packet is judged alone, even the reply to a request it allowed a moment ago. This is called being **stateless**.
- A guard can look inside a bag. An ACL reads only header fields (addresses, protocol, ports, some flags). It does not read the content of the packet.

## How It Works

### The parts of an ACL

- **ACL:** the whole list, with a number or a name.
- **ACE (Access Control Entry):** one line in the list.
- **Permit:** let the packet continue.
- **Deny:** drop the packet.
- **Sequence number:** the position of a line. The default numbers are 10, 20, 30 and so on, which leaves gaps for later inserts.

An ACL does nothing until you attach it to something. A list in the configuration that is not applied filters no traffic.

The commands below are for Cisco IOS, the operating system on many Cisco routers and switches. Other vendors use the same ideas with different words.

### Standard vs extended

A **standard ACL** matches only the source IP address of the packet. It cannot see where the packet is going or which application it belongs to.

An **extended ACL** can match protocol, source address, source port, destination address, destination port, and more.

| Feature | Standard | Extended |
|---|---|---|
| Source IP | Yes | Yes |
| Destination IP | No | Yes |
| Protocol (TCP, UDP, ICMP, OSPF) | No | Yes |
| Source and destination ports | No | Yes |
| TCP flags (`established`) | No | Yes |
| Numbered ranges | 1-99, 1300-1999 | 100-199, 2000-2699 |
| Where to place it | Near the destination | Near the source |
| Typical use | Simple source filter, VTY access, NAT selection | Most traffic filtering |

Standard ACL line:

```text
access-list 10 permit 192.168.10.0 0.0.0.255
```

Extended ACL line, read from left to right:

```text
access-list 110 permit tcp 192.168.20.0 0.0.0.255 host 172.16.50.10 eq 443
```

| Part | Value | Meaning |
|---|---|---|
| ACL number | `110` | Extended, because it is in 100-199 |
| Action | `permit` | Let it pass |
| Protocol | `tcp` | TCP only. `ip` would mean all IP traffic |
| Source | `192.168.20.0 0.0.0.255` | Any address in 192.168.20.0/24 |
| Destination | `host 172.16.50.10` | This one address |
| Destination port | `eq 443` | Port equal to 443 |

**Port position matters.** A port written after the source is a source port. A port written after the destination is a destination port.

Port operators: `eq` (equal), `neq` (not equal), `lt` (less than), `gt` (greater than), `range 1024 65535`.

### Named vs numbered

A **numbered ACL** is identified by a number, and the number also sets its type. A **named ACL** has a text name, and you state its type (`standard` or `extended`) when you create it.

| Feature | Numbered | Named |
|---|---|---|
| Identifier | Number such as `110` | Name such as `GUEST-IN` |
| Type shown by | The number range | The `standard` or `extended` keyword |
| Configured from | Global config, one long line each | Its own sub-mode, short lines |
| Self-describing | No | Yes |
| Delete one line in old-style syntax | Risky (see Common Mistakes) | Easy with `no <sequence>` |

The two forms below are the same filter:

```text
! Cisco IOS: numbered
access-list 110 permit tcp 192.168.20.0 0.0.0.255 any eq 443

! Cisco IOS: named
ip access-list extended GUEST-IN
 permit tcp 192.168.20.0 0.0.0.255 any eq 443
```

On modern IOS, a numbered ACL can also be edited in the named style, with `ip access-list extended 110`. Names are case sensitive. `Guest-In` and `GUEST-IN` are two different ACLs.

Good practice: use named ACLs. Six months later, `GUEST-IN` still tells you what it does. `110` does not.

### Wildcard masks

A **wildcard mask** tells the ACL which bits of an address to check and which to ignore.

- Bit `0` means: this bit must match.
- Bit `1` means: ignore this bit.

This looks like a subnet mask upside down. In a subnet mask, 1 marks the network part. In a wildcard mask, 0 marks the part you care about.

**Analogy:** imagine a card with holes laid over the address. You read the digits through the holes (the 0 bits). The covered parts (the 1 bits) can be anything. The analogy stops at this point: the holes are per bit, not per decimal digit, so an octet can be partly covered.

For a normal subnet, get the wildcard by subtracting the mask from 255.255.255.255:

```text
  255.255.255.255
- 255.255.255.224   (mask for /27)
-----------------
    0.  0.  0. 31   (wildcard)
```

| Prefix | Subnet mask | Wildcard | Addresses matched |
|---|---|---|---|
| /32 | 255.255.255.255 | 0.0.0.0 | 1 |
| /30 | 255.255.255.252 | 0.0.0.3 | 4 |
| /27 | 255.255.255.224 | 0.0.0.31 | 32 |
| /26 | 255.255.255.192 | 0.0.0.63 | 64 |
| /24 | 255.255.255.0 | 0.0.0.255 | 256 |
| /20 | 255.255.240.0 | 0.0.15.255 | 4096 |
| /16 | 255.255.0.0 | 0.0.255.255 | 65,536 |
| /0 | 0.0.0.0 | 255.255.255.255 | All of them |

Two shortcuts save typing:

| Shortcut | Long form | Meaning |
|---|---|---|
| `host 172.16.50.10` | `172.16.50.10 0.0.0.0` | Exactly one address |
| `any` | `0.0.0.0 255.255.255.255` | Every address |

Does `192.168.10.64 0.0.0.63` match the address 192.168.10.77? Compare only the bits where the wildcard is 0:

```text
                      last octet in binary
Packet address  .77   01 001101
ACL address     .64   01 000000
Wildcard        .63   00 111111
                      ^^ ^^^^^^
                      |  ignored (wildcard bits are 1)
                      must match (wildcard bits are 0)

The first two bits are 01 in both, so the packet matches.
```

Try 192.168.10.200. Its last octet is `11001000`. The first two bits are `11`, not `01`, so it does not match. The line covers .64 to .127.

**Matching a range of addresses.** Use a block size that is a power of 2. The wildcard value is the block size minus 1, and the range must start on a multiple of the block size.

- 192.168.16.0 to 192.168.31.255 is 16 blocks of /24, and 16 is a multiple of 16. So use `192.168.16.0 0.0.15.255`.
- 192.168.10.0 to 192.168.25.255 is also 16 blocks, but 10 is not a multiple of 16. One line cannot do it. You need several lines.

**Wildcards can do what subnet masks cannot.** The 1 bits do not have to be next to each other.

| Line | Check | Matches |
|---|---|---|
| `192.168.0.0 0.0.254.255` | Last bit of the third octet must be 0 | 192.168.0.x, 192.168.2.x, 192.168.4.x (even third octets) |
| `192.168.10.1 0.0.0.254` | Last bit of the fourth octet must be 1 | 192.168.10.1, .3, .5 (odd hosts) |

Real networks rarely need this, but it is a favorite interview question.

### Top-down processing and the implicit deny

The router checks lines in sequence order and stops at the first match.

```text
 packet
   |
   v
 line 10 match? -- yes --> do its action, STOP
   | no
   v
 line 20 match? -- yes --> do its action, STOP
   | no
   v
 line 30 match? -- yes --> do its action, STOP
   | no
   v
 implicit deny: DROP (not shown, not counted)
```

Three results follow from this:

1. **Order changes the meaning.** Put specific lines before general lines.
2. **Every ACL ends with an invisible deny.** In a standard ACL it acts like `deny any`. In an extended ACL it acts like `deny ip any any`.
3. **An ACL needs at least one permit.** A list with only deny lines blocks everything, because whatever the denies miss falls to the implicit deny.

Wrong order:

```text
10 permit 192.168.10.0 0.0.0.255
20 deny   192.168.10.66
```

The address 192.168.10.66 matches line 10 first and is permitted. Line 20 can never match. Some IOS versions refuse a standard line like this for that reason. Swap the lines to fix it.

If your goal is "block one thing and allow the rest", you must add the permit yourself:

```text
access-list 110 deny tcp 192.168.20.0 0.0.0.255 host 172.16.50.10 eq 23
access-list 110 permit ip any any
```

### Inbound vs outbound

An ACL is attached to an interface in one direction. The direction is always from the router's point of view, not from your LAN's point of view.

- **Inbound (`in`):** the packet is entering the router through this interface.
- **Outbound (`out`):** the packet is leaving the router through this interface.

```text
            in                             out
 Host A ---------> [Gi0/0]   R1   [Gi0/2] ---------> Server
 Host A <--------- [Gi0/0]   R1   [Gi0/2] <--------- Server
            out                            in
```

The same packet is "in" at one interface and "out" at the other. Here is the order of events inside the router:

```text
 packet arrives on an interface
          |
          v
 inbound ACL on this interface?
   none or permit --> continue
   deny or no match --> DROP
          |
          v
 routing table lookup picks the exit interface
          |
          v
 outbound ACL on the exit interface?
   none or permit --> forward the packet
   deny or no match --> DROP
```

Inbound filtering saves work, because the router drops the packet before the route lookup. Outbound filtering is handy when traffic from many interfaces leaves through one exit and they all need the same rule.

**The three Ps:** you can apply one ACL **per protocol, per interface, per direction**. So one interface can hold one IPv4 ACL in, one IPv4 ACL out, one IPv6 ACL in and one IPv6 ACL out. A second `ip access-group ... in` replaces the first. It does not add to it.

### The placement rule

| ACL type | Place it | Why |
|---|---|---|
| Standard | As close to the **destination** as possible | It sees only the source, so near the source it would block that source from everywhere |
| Extended | As close to the **source** as possible | It knows the exact traffic, so it can drop unwanted packets early and save bandwidth |

**Analogy:** a standard ACL is a guard who knows only who you are, not where you are going. If that guard stands at the front door and turns you away, you cannot reach any room. So the guard must stand at the door of the one protected room. An extended ACL is a guard who reads your full ticket, so the guard can stand at the front door and turn you away before you walk through the whole building. The analogy stops here: walking costs you time, but a dropped packet costs the network bandwidth on every link it would have crossed.

```text
 Standard ACL near the SOURCE (wrong):

   Guests --> [Gi0/1 in: deny guests] R1 --> Servers   blocked (good)
                                       \---> Internet  blocked (bad)

 Standard ACL near the DESTINATION (right):

   Guests --> R1 --> [Gi0/2 out: deny guests] --> Servers  blocked
               \------------------------------> Internet   still open
```

Fuller truth: this is a guideline, not a law. You may have to filter where you have control, because the source network may belong to someone else. Many networks also use extended ACLs everywhere and skip standard ACLs for traffic filtering.

### Creating and applying an ACL

1. Write down the policy in plain words: who may reach what.
2. Choose the type, the router, the interface and the direction.
3. Build the ACL.
4. Apply it. Use `ip access-group` on an interface, or `access-class` on VTY lines (the virtual lines used for SSH and Telnet logins to the device).
5. Test both what should pass and what should fail.
6. Read the counters.

### ACLs are stateless, so replies need a rule too

Suppose a PC opens a web page. The request goes to port 443. The reply comes back **from** port 443 **to** a high port on the PC. A plain ACL has no memory of the request, so the reply must match a line of its own if an ACL filters that direction.

```text
Request:  192.168.10.25:51514  --> 172.16.50.10:443
Reply:    172.16.50.10:443     --> 192.168.10.25:51514
```

A stateful firewall remembers the request and allows the reply on its own. See [Firewalls](04-firewalls.md). The `established` keyword below is an old and partial answer to this problem.

### The established keyword

`established` works only on TCP lines in extended ACLs. It matches a segment that has the **ACK or RST flag** set. It reads those bits and nothing more. It does not check for a real session.

The very first SYN that opens a connection has no ACK flag. After that, normal TCP segments all carry ACK. So `established` means, in effect, "not a new connection attempt".

| TCP segment | Flags | Matches `established`? |
|---|---|---|
| First packet of a connection | SYN | No |
| Server reply | SYN, ACK | Yes |
| Handshake finish | ACK | Yes |
| Normal data | ACK (often PSH too) | Yes |
| Closing | FIN, ACK | Yes |
| Reset | RST (or RST, ACK) | Yes |

**Analogy:** a door guard who lets in anyone with a stamp on their hand. New visitors have no stamp, so they are stopped. But the guard never checks the stamp against a list, so a forged stamp also works. An attacker can craft a TCP packet with ACK set, and it will pass the ACL. The analogy stops here: the forged packet does not open a session. The target has no matching connection, so it normally answers with a reset. Still, the packet got through, and scanners use this to map a network.

`established` does nothing for UDP or ICMP, because they have no such flags.

## Example

All examples use this network. R1 runs PAT, so inside hosts reach the Internet as 203.0.113.2.

```text
                        Internet
                           |
                    ISP 203.0.113.1
                           |
                    Gi0/3  203.0.113.2/30
                  +--------+--------+
  Staff LAN       |                 |       Server LAN
  192.168.10.0/24 |                 |       172.16.50.0/24
  ----------------+ Gi0/0  R1 Gi0/2 +----------------
  192.168.10.1    |                 |       172.16.50.1
  Admin PC .5     |                 |       Web .10, File .20
  Contractor .66  +--------+--------+       DNS .53
                           |
                    Gi0/1  192.168.20.1/24
                           |
                     Guest LAN
                     192.168.20.0/24
```

### Example 1: Standard numbered ACL

**Policy:** only the Staff LAN may reach the Server LAN, except the contractor PC at 192.168.10.66.

A standard ACL goes near the destination, so it goes outbound on Gi0/2. The specific line goes first.

```text
! Cisco IOS
R1(config)# access-list 10 remark Contractor PC must not reach servers
R1(config)# access-list 10 deny host 192.168.10.66
R1(config)# access-list 10 remark Rest of Staff LAN may reach servers
R1(config)# access-list 10 permit 192.168.10.0 0.0.0.255
R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip access-group 10 out
```

`remark` adds a comment for humans. It has no effect on traffic.

```text
R1# show access-lists 10
Standard IP access list 10
    10 deny   192.168.10.66 (14 matches)
    20 permit 192.168.10.0, wildcard bits 0.0.0.255 (2381 matches)
```

The implicit deny now stops everything else going out of Gi0/2, including guests. It also stops Internet traffic toward the servers, which would matter if a server needed replies from the Internet. Think through all flows before you apply a rule, not only the one you were asked about.

### Example 2: Extended named ACL

**Policy for guests:**

- Get an address by DHCP.
- Use the internal DNS server, 172.16.50.53.
- Browse the web.
- Reach nothing else inside.

An extended ACL goes near the source, so it goes inbound on Gi0/1.

```text
! Cisco IOS
R1(config)# ip access-list extended GUEST-IN
R1(config-ext-nacl)# remark DHCP from clients with no address yet
R1(config-ext-nacl)# permit udp any eq bootpc any eq bootps
R1(config-ext-nacl)# remark Internal DNS only
R1(config-ext-nacl)# permit udp 192.168.20.0 0.0.0.255 host 172.16.50.53 eq domain
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 host 172.16.50.53 eq domain
R1(config-ext-nacl)# remark No access to internal networks
R1(config-ext-nacl)# deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 log
R1(config-ext-nacl)# deny ip 192.168.20.0 0.0.0.255 172.16.50.0 0.0.0.255 log
R1(config-ext-nacl)# remark Web browsing, including HTTP/3 over UDP 443
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# permit udp 192.168.20.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# deny ip any any log
R1(config-ext-nacl)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group GUEST-IN in
```

Notes on this ACL:

- The DHCP line uses `any` because a client with no address sends from 0.0.0.0 to 255.255.255.255. The lines that name 192.168.20.0/24 would not match it.
- The explicit `deny ip any any log` does what the implicit deny does, but it adds counters and log messages.
- The reply from a web server comes back through Gi0/3 and leaves through Gi0/1. No ACL filters that outbound direction here, so replies pass.

Trace some packets through it. Stop at the first match.

| Source | Destination | Service | First matching line | Result |
|---|---|---|---|---|
| 192.168.20.25 | 172.16.50.53 | UDP 53 | Line 20 | Permit |
| 192.168.20.25 | 172.16.50.10 | TCP 443 | Line 50 | Deny. Line 70 is never reached |
| 192.168.20.25 | 192.168.10.5 | ICMP | Line 40 | Deny |
| 192.168.20.25 | 198.51.100.80 | TCP 443 | Line 70 | Permit |
| 192.168.20.25 | 198.51.100.80 | TCP 22 | Line 90 | Deny |
| 10.9.9.9 (fake) | 198.51.100.80 | TCP 443 | Line 90 | Deny, wrong source network |

### Example 3: The established keyword at the Internet edge

**Policy:** inside users may use the Internet. Nobody on the Internet may start a TCP connection to R1 or to anything behind it.

Because R1 runs PAT, packets from the Internet arrive addressed to 203.0.113.2. An inbound ACL on Gi0/3 sees that address, not the private one. The DNS server sends its queries to an outside resolver at 198.51.100.53.

```text
! Cisco IOS
R1(config)# ip access-list extended OUTSIDE-IN
R1(config-ext-nacl)# remark Return TCP traffic only, no new connections
R1(config-ext-nacl)# permit tcp any host 203.0.113.2 established
R1(config-ext-nacl)# remark DNS replies from the outside resolver
R1(config-ext-nacl)# permit udp host 198.51.100.53 eq domain host 203.0.113.2
R1(config-ext-nacl)# remark ICMP replies and errors
R1(config-ext-nacl)# permit icmp any host 203.0.113.2 echo-reply
R1(config-ext-nacl)# permit icmp any host 203.0.113.2 unreachable
R1(config-ext-nacl)# permit icmp any host 203.0.113.2 time-exceeded
R1(config-ext-nacl)# deny ip any any log
R1(config-ext-nacl)# exit
R1(config)# interface GigabitEthernet0/3
R1(config-if)# ip access-group OUTSIDE-IN in
```

| Packet arriving on Gi0/3 | Flags | Result |
|---|---|---|
| 198.51.100.80:443 to 203.0.113.2:51514, a reply to a web request | SYN, ACK | Line 10 permits it |
| 198.51.100.77:40000 to 203.0.113.2:22, a new SSH attempt | SYN | Line 60 denies it |
| 198.51.100.77:40000 to 203.0.113.2:22, a crafted packet | ACK | Line 10 **permits** it. R1 finds no session and resets it. |

The third row is the weakness of `established`.

There is a second limit. Line 80 of `GUEST-IN` lets QUIC (HTTP/3 over UDP) go out, but `OUTSIDE-IN` drops the UDP replies, and `established` cannot help. Browsers then fall back to TCP. To allow those replies, you would need `permit udp any eq 443 host 203.0.113.2`, which lets anyone send from UDP port 443. A stateful firewall solves this cleanly, so it is the better tool at a real Internet edge.

### Example 4: Lock down SSH access to the router

**Policy:** only the Admin PC, 192.168.10.5, may log in to R1.

`access-class` filters logins to the device itself. It does not filter traffic passing through. The placement rule does not apply here, because the filter sits right on the thing it protects.

```text
! Cisco IOS
R1(config)# ip access-list standard MGMT-ONLY
R1(config-std-nacl)# permit host 192.168.10.5
R1(config-std-nacl)# deny any log
R1(config-std-nacl)# exit
R1(config)# line vty 0 4
R1(config-line)# access-class MGMT-ONLY in
R1(config-line)# transport input ssh
```

Apply this to every VTY line the device has. Some devices have 16 (`line vty 0 15`), and an unprotected line is an open door. See [Remote Access Protocols](../07-application-protocols/09-remote-access-protocols.md).

### Example 5: Edit a live ACL

Sequence numbers let you change one line without retyping the list. Here, guests get access to the web server at 172.16.50.10. The new line must sit before the deny at line 50.

```text
! Cisco IOS
R1(config)# ip access-list extended GUEST-IN
R1(config-ext-nacl)# 45 permit tcp 192.168.20.0 0.0.0.255 host 172.16.50.10 eq 443
R1(config-ext-nacl)# no 80
R1(config-ext-nacl)# exit
R1(config)# ip access-list resequence GUEST-IN 10 10
```

- `45 permit ...` inserts a line between 40 and 50.
- `no 80` removes the QUIC line.
- `ip access-list resequence GUEST-IN 10 10` renumbers the list from 10 in steps of 10, which brings back gaps for later edits.

**Working over a remote link?** Protect yourself first. `reload in 10` schedules a restart in 10 minutes. If a bad ACL locks you out, the router restarts with the saved config. If all goes well, run `reload cancel`. Do not save the config until you have tested the change.

### Verifying an ACL

**1. See the lines and the hit counts.** Counters tell you which lines are matching. This output shows `GUEST-IN` as built in Example 2, before the edits in Example 5.

```text
R1# show ip access-lists GUEST-IN
Extended IP access list GUEST-IN
    10 permit udp any eq bootpc any eq bootps (62 matches)
    20 permit udp 192.168.20.0 0.0.0.255 host 172.16.50.53 eq domain (3410 matches)
    30 permit tcp 192.168.20.0 0.0.0.255 host 172.16.50.53 eq domain (41 matches)
    40 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 log (17 matches)
    50 deny ip 192.168.20.0 0.0.0.255 172.16.50.0 0.0.0.255 log (9 matches)
    60 permit tcp 192.168.20.0 0.0.0.255 any eq www (1288 matches)
    70 permit tcp 192.168.20.0 0.0.0.255 any eq 443 (40912 matches)
    80 permit udp 192.168.20.0 0.0.0.255 any eq 443 (2206 matches)
    90 deny ip any any log (3 matches)
```

Note that IOS shows port 80 as `www`, even if you typed 80. The implicit deny never appears here. To count what it drops, add the explicit last line, as in line 90. Remarks appear in the running config, and classic IOS leaves them out of this output. Newer IOS XE versions may also show sequence numbers in the running config.

**2. Check that it is applied, and in which direction.**

```text
R1# show ip interface GigabitEthernet0/1 | include access list
  Outgoing access list is not set
  Inbound  access list is GUEST-IN
```

Some versions also print "Common access list" lines. You can also use `show running-config interface GigabitEthernet0/1`.

**3. Reset the counters, then test.**

```text
R1# clear access-list counters GUEST-IN
```

**4. Test from a real host**, not from the router, because of how routers treat their own traffic (see Going Deeper).

```bash
# Linux: test from a guest PC (192.168.20.25)
nc -vz 172.16.50.53 53
nc -vz 172.16.50.20 445
ping -c 1 172.16.50.20
```

```text
Connection to 172.16.50.53 53 port [tcp/domain] succeeded!
nc: connect to 172.16.50.20 port 445 (tcp) failed: No route to host
From 192.168.20.1 icmp_seq=1 Packet filtered
```

```powershell
# Windows: test from a guest PC (192.168.20.25)
Test-NetConnection 172.16.50.53 -Port 53
```

```text
ComputerName     : 172.16.50.53
RemoteAddress    : 172.16.50.53
RemotePort       : 53
InterfaceAlias   : Ethernet
SourceAddress    : 192.168.20.25
TcpTestSucceeded : True
```

"No route to host" looks like a routing fault, but here it means the router sent back an ICMP "administratively prohibited" message (type 3, code 13) when the ACL dropped the packet. If the interface has `no ip unreachables`, the router stays silent, and the test simply times out.

**5. Read the log for denied traffic.** Lines with the `log` keyword produce messages like this:

```text
R1# show logging | include IPACCESSLOG
*Oct 30 09:14:22.101: %SEC-6-IPACCESSLOGP: list GUEST-IN denied tcp 192.168.20.25(51514) -> 172.16.50.20(445), 1 packet
```

**A troubleshooting order that works:**

1. Is the ACL applied, on the right interface, in the right direction?
2. Is the name spelled exactly right?
3. Do the counters rise when you send test traffic? On which line?
4. Is a broad line above the line you expected to match?
5. Is the reply traffic allowed, if the other direction is filtered too?
6. Add a temporary `deny ip any any log` as the last line to see what the implicit deny was dropping.

## Going Deeper

### What ACLs see and do not see

| Traffic | Inbound ACL | Outbound ACL |
|---|---|---|
| Passing through the router | Filtered | Filtered |
| Addressed to the router (OSPF hello, BGP, SSH, ping) | **Filtered** | Does not apply |
| Created by the router itself (its own pings, routing updates) | Does not apply | **Not filtered** on Cisco IOS |
| ARP and other non-IP traffic | Not filtered by an IP ACL | Not filtered by an IP ACL |
| Hosts in the same subnet, talking through a switch | Never reaches the router | Never reaches the router |

The second row causes real outages. An inbound ACL's implicit deny will also drop routing protocol packets arriving on that interface. If OSPF neighbors fall down right after you apply an ACL, this is the likely cause. Add a line such as `permit ospf any any`. For BGP, allow TCP port 179 from the peer.

Fuller truth: a device's control plane (the part that runs its protocols) needs stronger protection than interface ACLs give. Control Plane Policing (CoPP) applies ACL-based rules to traffic headed for the device's CPU.

### ACLs and NAT together

ACLs and NAT both touch the packet, so the order matters. Cisco IOS checks an inbound ACL before NAT and an outbound ACL after NAT. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

For the staff PC 192.168.10.25, translated by PAT to 203.0.113.2:

| ACL position | Direction of traffic | Address the ACL sees |
|---|---|---|
| Inbound on the inside interface | Going out | Source 192.168.10.25 (private) |
| Outbound on the outside interface | Going out | Source 203.0.113.2 (public) |
| Inbound on the outside interface | Coming back | Destination 203.0.113.2 (public) |
| Outbound on the inside interface | Coming back | Destination 192.168.10.25 (private) |

This is why `OUTSIDE-IN` in Example 3 names 203.0.113.2. A line naming 192.168.10.0/24 there would never match.

### Anti-spoofing at the edge

**Spoofing** means faking the source address. Packets arriving from the Internet should never claim a private source. Put lines like these at the top of the Internet-facing inbound ACL:

```text
! Cisco IOS: add at the top of the outside inbound ACL
deny ip 10.0.0.0 0.255.255.255 any log
deny ip 172.16.0.0 0.15.255.255 any log
deny ip 192.168.0.0 0.0.255.255 any log
deny ip 127.0.0.0 0.255.255.255 any log
```

Remember the three Ps: this goes into the same ACL as your other inbound rules, because a second ACL would replace the first. Filtering spoofed sources at the network edge is described in BCP 38 (RFC 2827).

### Do not block all ICMP

Some ICMP messages keep the network working. Path MTU Discovery depends on "fragmentation needed" messages (`packet-too-big` in IOS syntax). Blocking them can cause connections that open and then hang. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md). Example 3 allows them through the `unreachable` line.

### Fragments

Only the first fragment of a split IP packet carries the TCP or UDP header. Later fragments have no ports to check. For a non-initial fragment on a line that has Layer 4 information, IOS checks only the Layer 3 fields:

- On a `permit` line, a Layer 3 match is enough, and the fragment passes.
- On a `deny` line, that line is skipped, and the check moves to the next line.

The `fragments` keyword makes a line match non-initial fragments only, for example `deny ip any host 172.16.50.10 fragments`. Background: [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

### Logging and performance

- The `log` keyword sends matching packets toward the CPU, at least the first of each flow. On a busy line this can raise CPU load. Log denies and rare events, not your busiest permits.
- IOS limits how often it prints these messages. After the first message, it batches them. So the log does not show every packet.
- `log-input` also records the arrival interface and source MAC address.
- On software-based routers, lines are checked one by one, so the busiest permits should sit near the top where policy allows.
- Multilayer switches and many modern platforms store ACLs in TCAM (a special fast memory that checks all entries at once). Speed stays high, but TCAM space is limited, so very large ACLs can exceed it. Behavior varies by platform.

### Time-based ACLs

A line can apply only at certain times.

```text
! Cisco IOS
R1(config)# time-range OFFICE-HOURS
R1(config-time-range)# periodic weekdays 8:00 to 18:00
R1(config-time-range)# exit
R1(config)# ip access-list extended STAFF-IN
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 any eq 443 time-range OFFICE-HOURS
```

This uses the router's own clock, so keep the clock correct with NTP.

### Object groups

An object group names a set of addresses or services once, such as `WEB-SERVERS`, so that one ACL line can refer to the whole set. This keeps large ACLs short. Support and syntax depend on platform and version.

### Reflexive ACLs

A **reflexive ACL** is an older way to add simple session tracking to IOS. A `reflect` keyword on an outgoing line creates a temporary mirror-image entry for each session. An `evaluate` line in the inbound ACL checks those temporary entries. It handles UDP as well as TCP, and it is stronger than `established`, because a forged ACK has no matching entry.

Today, a stateful firewall or the IOS Zone-Based Firewall is the normal choice. See [Firewalls](04-firewalls.md).

### IPv6 ACLs

IPv6 ACLs follow the same top-down logic, with key differences:

| Point | IPv4 | IPv6 |
|---|---|---|
| Types | Standard and extended | One type, always named, works like extended |
| Address format | Wildcard mask | Prefix length, such as `2001:db8:20::/64` |
| Apply to an interface | `ip access-group NAME in` | `ipv6 traffic-filter NAME in` |
| Apply to VTY lines | `access-class NAME in` | `ipv6 access-class NAME in` |
| Implicit rules at the end | Deny all | Permit neighbor discovery, then deny all |

```text
! Cisco IOS
R1(config)# ipv6 access-list V6-GUEST-IN
R1(config-ipv6-acl)# permit udp 2001:db8:20::/64 host 2001:db8:50::53 eq domain
R1(config-ipv6-acl)# deny ipv6 2001:db8:20::/64 2001:db8:50::/64 log
R1(config-ipv6-acl)# permit tcp 2001:db8:20::/64 any eq 443
R1(config-ipv6-acl)# exit
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ipv6 traffic-filter V6-GUEST-IN in
```

Every IPv6 ACL ends with these invisible lines:

```text
permit icmp any any nd-na
permit icmp any any nd-ns
deny ipv6 any any
```

Those two permits matter. Neighbor Discovery does the job in IPv6 that ARP does in IPv4, and it runs over ICMPv6, so an IPv6 ACL can see it. If you add your own `deny ipv6 any any` at the end, it matches before the implicit permits, and neighbors can stop resolving. In that case, add the two `nd` permits above your deny. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### ACLs that select instead of filter

ACLs are also used as a matching tool. Here, "permit" means "select this traffic", and "deny" means "do not select it". The packet is not dropped because of the ACL itself.

| Feature | What the ACL selects |
|---|---|
| NAT | Which inside addresses to translate ([NAT and PAT](../04-ip-addressing/09-nat-and-pat.md)) |
| QoS | Which traffic belongs in a class ([Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md)) |
| Policy-based routing | Which traffic takes a special path ([Advanced Routing Techniques](../05-routing/12-advanced-routing-techniques.md)) |
| IPsec | Which traffic is protected ([IPsec](08-ipsec.md)) |
| SNMP, NTP | Which clients a service answers |
| Debugging | Which packets a debug command shows |

### Where ACLs are applied

| Name | Applied on | Filters |
|---|---|---|
| RACL (router ACL) | Routed interface or SVI | Traffic routed through it |
| PACL (port ACL) | Layer 2 switch port | Traffic entering that port |
| VACL (VLAN ACL, or VLAN map) | A whole VLAN | Traffic inside the VLAN, including same-subnet traffic |
| dACL (downloadable ACL) | A user's port, sent by a RADIUS server after login | That one user's traffic |

An **SVI** (Switch Virtual Interface) is the routed interface of a VLAN on a Layer 3 switch. On an SVI, `in` means traffic coming from the VLAN into the routing engine. People often get this backward. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md). Platform support for PACLs and VACLs varies.

### The same idea on other platforms

| Platform | Equivalent | Key difference |
|---|---|---|
| Cisco ASA | `access-list` with `access-group` | Uses subnet masks, not wildcards. The device is stateful |
| Juniper Junos | Firewall filter with terms | Uses prefixes. Also ends with an implicit discard |
| Linux | [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md) | The chain has a default policy, often ACCEPT, in place of an implicit deny |
| AWS | [Network ACLs and security groups](../11-cloud-and-containers/04-cloud-security-and-connectivity.md) | Network ACLs are numbered and stateless. Security groups are stateful |
| Windows | Windows Defender Firewall | Not simple top-down. Block rules generally win over allow rules |

```bash
# Linux: first-match rules, similar in spirit to GUEST-IN
sudo iptables -A FORWARD -s 192.168.20.0/24 -d 172.16.50.53 -p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.20.0/24 -d 172.16.50.0/24 -j DROP
sudo iptables -L FORWARD -n -v --line-numbers
```

```powershell
# Windows: block inbound Telnet on this host
New-NetFirewallRule -DisplayName "Block Telnet" -Direction Inbound -Protocol TCP -LocalPort 23 -Action Block
```

Fuller truth: an ACL is a basic packet filter. It complements a stateful firewall and does not replace it. Many networks use ACLs on internal routers and switches, and firewalls at trust boundaries.

## Common Mistakes

- **Forgetting the implicit deny.** You add `deny` lines only, apply the ACL, and all traffic stops. End with `permit ip any any` if you want to block a few things and allow the rest.
- **Wrong direction.** `in` and `out` are from the router's view. "Inbound to my LAN" is `out` on the LAN interface.
- **Putting a general line above a specific line.** A broad permit at the top hides every line below it.
- **Using a subnet mask in place of a wildcard.** `192.168.10.0 255.255.255.0` is accepted without an error on IOS, but it matches the wrong thing. It checks only the last octet.
- **Placing the port on the wrong side.** `permit tcp any eq 80 host 172.16.50.10` matches source port 80, not traffic going to a web server.
- **Applying an ACL before it is complete.** One line in an applied ACL brings the implicit deny with it. Build the list first, then apply it.
- **A misspelled ACL name.** On Cisco IOS routers, applying an ACL that does not exist usually permits all traffic, so you have no filter and see no error. Check with `show ip interface`. Behavior can vary by platform.
- **Running `no access-list 110 deny ...` in global config.** On a numbered ACL, this classic trap removes the **whole** ACL, not one line. Enter `ip access-list extended 110`, then use `no <sequence>`.
- **Expecting a second ACL to add to the first.** One ACL per protocol, per interface, per direction. The new one replaces the old one.
- **Forgetting reply traffic.** ACLs are stateless. If you filter both directions, replies need their own lines.
- **Forgetting routing protocols and DHCP.** The implicit deny on an inbound ACL can drop OSPF, BGP and DHCP as well.
- **Using `host` or `any` in the wrong place.** In a standard ACL, a bare address such as `access-list 10 permit 192.168.10.5` assumes wildcard 0.0.0.0, which is one host. In an extended ACL, every address needs a wildcard, `host` or `any`.
- **Testing from the router.** A router's own traffic is not filtered by outbound ACLs on IOS, so `ping` from R1 can succeed while hosts fail.
- **Treating `established` as stateful.** It reads flag bits. It does not track sessions, and a crafted ACK passes.
- **Adding ACLs to a remote device with no way back.** One wrong line can cut your own SSH session. Use `reload in 10` first.

## Interview Questions

**Q: What is the difference between a standard and an extended ACL?**

A: A standard ACL matches only the source IP address. An extended ACL can match the protocol, source and destination addresses, and source and destination ports. Standard numbered ranges are 1-99 and 1300-1999. Extended ranges are 100-199 and 2000-2699. Extended gives much more precise control.

**Q: What is a wildcard mask, and what is the wildcard for a /27?**

A: A wildcard mask tells the ACL which bits to check. A 0 means the bit must match, and a 1 means ignore it. For a /27, the mask is 255.255.255.224, so the wildcard is 0.0.0.31. For example, `192.168.1.32 0.0.0.31` matches 192.168.1.32 to 192.168.1.63.

**Q: What is the implicit deny, and how can you see what it drops?**

A: Every ACL ends with an invisible deny of all traffic, so anything that matches no line is dropped. It has no counter and does not appear in `show access-lists`. To see those drops, I add an explicit `deny ip any any log` as the last line. That gives me counters and log messages.

**Q: Where should you place standard and extended ACLs, and why?**

A: A standard ACL goes near the destination, because it sees only the source. Near the source, it would block that source from everywhere. An extended ACL goes near the source, because it can match the exact traffic and drop it before it uses bandwidth. It is a guideline, and in practice you filter where you have control.

**Q: What is the difference between inbound and outbound ACLs?**

A: Direction is from the router's view. An inbound ACL checks packets as they enter an interface, before the routing lookup. An outbound ACL checks packets as they leave an interface, after routing. Inbound saves router work. On Cisco IOS, outbound ACLs do not filter traffic created by the router itself.

**Q: How many ACLs can you apply to one interface?**

A: One per protocol, per interface, per direction. So one interface can have one IPv4 ACL in, one IPv4 ACL out, one IPv6 ACL in and one IPv6 ACL out. Applying a second IPv4 ACL in the same direction replaces the first one.

**Q: What does the established keyword do? Is it secure?**

A: It matches TCP segments with the ACK or RST bit set. That lets replies to inside connections come back, while it blocks a first SYN from outside. It is not stateful. It only reads flag bits, so a crafted ACK packet passes, and it does nothing for UDP. A stateful firewall is the stronger choice.

**Q: Write an ACL that blocks Telnet from 192.168.20.0/24 to server 172.16.50.10 and allows everything else. Where do you apply it?**

A: `access-list 110 deny tcp 192.168.20.0 0.0.0.255 host 172.16.50.10 eq 23`, then `access-list 110 permit ip any any`. It is extended, so I apply it near the source: `ip access-group 110 in` on the interface facing 192.168.20.0/24. Without the permit line, the implicit deny would drop everything.

**Q: How do you add a line in the middle of an existing ACL?**

A: I use sequence numbers. I enter `ip access-list extended NAME`, then type the new line with a free number, such as `45 permit tcp ...`. If I run out of gaps, I use `ip access-list resequence NAME 10 10`. This also works for numbered ACLs on modern IOS.

**Q: What is the difference between an ACL and a stateful firewall?**

A: A basic ACL is stateless. It judges each packet alone, using header fields, so replies need their own rules. A stateful firewall keeps a table of connections and allows replies automatically. It can also check that packets fit a real session. ACLs are fast and simple, and they suit routers and switches. Firewalls suit trust boundaries.

**Q: You applied an ACL and users lost access. How do you troubleshoot it?**

A: First I check the interface and direction with `show ip interface`. Then I read the counters in `show access-lists` while I send test traffic. I look for a broad line sitting above the line I expected, and for missing return traffic. Then I add `deny ip any any log` at the end to see what the implicit deny is dropping.

**Q: How do IPv6 ACLs differ from IPv4 ACLs?**

A: IPv6 ACLs are always named, have only one type, and use prefix lengths, not wildcards. You apply them with `ipv6 traffic-filter`. Their implicit end rules first permit neighbor solicitation and neighbor advertisement, then deny everything. A manual `deny ipv6 any any` overrides those permits, so you must add them back yourself.

## Quick Recap

- An ACL is an ordered list checked top-down. The first match wins, and the rest is skipped.
- Every ACL ends with an invisible deny, so each ACL needs at least one permit.
- Standard ACLs match the source only and go near the destination. Extended ACLs match much more and go near the source.
- In a wildcard mask, 0 means check and 1 means ignore. `host` is 0.0.0.0, and `any` is 255.255.255.255.
- Apply one ACL per protocol, per interface, per direction. `in` and `out` are from the router's view.
- ACLs are stateless. `established` matches ACK or RST bits, and it is not true session tracking.
- Verify with `show access-lists`, `show ip interface`, and tests from a real host.
