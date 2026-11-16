# Layer 2 Security

**Level:** Advanced | **Prerequisites:** [MAC Addresses](01-mac-addresses.md), [How Switches Work](03-how-switches-work.md), [ARP](05-arp.md), [VLANs](06-vlans.md), [Trunking and 802.1Q](07-trunking-8021q.md), [DHCP](../07-application-protocols/04-dhcp.md)

## In One Line

Ethernet, ARP and DHCP trust everyone by default, so Layer 2 security adds switch features that check who is really on each port and what that device is allowed to say.

## Why It Matters

Layer 2 is the part of the network where devices talk directly through switches, using MAC addresses. A MAC address is the hardware address of a network card.

Most Layer 2 protocols were built for friendly networks. None of these has any built-in proof of identity:

- A switch learns any source MAC address it sees.
- A host believes almost any ARP reply it receives.
- A client accepts the first DHCP offer it receives.
- A switch port may agree to become a trunk because a device asked.

This creates a real problem. A firewall at the edge of the network never sees traffic between two hosts in the same VLAN. So one infected laptop, or one person plugging into a meeting room socket, can attack every host in that VLAN. The firewall will not see it.

Interviewers ask about this topic for three reasons:

1. It proves you understand how switches, ARP and DHCP really work, not only the names.
2. The defences depend on each other, so the questions go deep quickly. For example: "Why does Dynamic ARP Inspection need DHCP snooping?"
3. These features can cut off real users when set up badly, so they show whether you have real operational sense.

## Simple Explanation

Think of an office building with no security staff.

- The **switch** is a mail room. A clerk keeps a notebook that says which person sits at which desk.
- **VLANs** are separate floors. Mail stays on its own floor.
- **DHCP** is the front desk. It gives each new visitor a desk number and a map of the building.
- **ARP** is shouting across the room: "Who is Sara? Please tell me your desk!"

Now look at how a dishonest person can abuse this:

| Attack | In the office | On the network |
|---|---|---|
| MAC flooding | Fills the clerk's notebook with fake names, so the clerk reads all mail aloud | Fills the switch's address table so it floods frames |
| DHCP starvation | Takes every free desk using fake names | Takes every free IP address |
| Rogue DHCP | Sets up a fake front desk with a false map | Gives clients a false gateway or DNS server |
| ARP poisoning | Shouts "I am Sara!" and receives her mail | Claims the gateway's IP address |
| Switch spoofing | Wears a staff uniform and gets a key to every floor | Pretends to be a switch and gets a trunk |
| Double tagging | Hides a letter for floor 20 inside an envelope for floor 1 | Puts a VLAN tag inside another VLAN tag |

The defences are like adding trained guards:

- A guard limits each door to a small number of ID cards (port security).
- A guard makes sure only the real front desk hands out desks, and writes down who got what (DHCP snooping).
- A guard checks that shouted claims match that written record (Dynamic ARP Inspection).
- A guard checks the return address on letters leaving each desk (IP Source Guard).
- Empty rooms stay locked (unused port practices).

**Where the analogy stops being true:** a human clerk would notice something strange. A switch does not think. It follows simple rules at very high speed, so a fake message that fits the rules works every time. Also, an office has one front door, while a network has hundreds of ports, and each one needs its own guard.

## How It Works

All Cisco commands in this chapter are for Cisco IOS or IOS XE on Catalyst access switches. The ideas apply to other vendors, but the commands differ. Exact options also vary by platform and software version.

### The Trust Map

Each attack abuses a trust that was built into a protocol. Each defence replaces that trust with a check.

| Attack | Trust that is abused | Main defence |
|---|---|---|
| MAC flooding | Switch learns any source MAC | Port security |
| DHCP starvation | Server serves any client request | Port security, DHCP snooping rate limit |
| Rogue DHCP | Client accepts any offer | DHCP snooping |
| ARP poisoning | Host accepts any ARP reply | Dynamic ARP Inspection |
| IP spoofing | Hosts can write any source IP | IP Source Guard |
| Switch spoofing | Port negotiates a trunk | Static access mode, no DTP |
| Double tagging | Native VLAN frames travel untagged | Unused native VLAN, tag native |
| Broadcast storm | Switch floods broadcasts | Storm control, STP |

### 1. MAC Flooding

**Background.** A switch keeps a table that maps each MAC address to a port. Cisco calls it the CAM table. CAM means content addressable memory, a kind of very fast lookup memory. Entries are removed if not seen for a while. The Cisco default aging time is 300 seconds.

A switch does three things with a frame (a frame is one Ethernet message):

- It **learns** the source MAC and the port it arrived on.
- If it knows the destination MAC, it **forwards** the frame out of one port.
- If it does not know the destination, it **floods** the frame out of every other port in that VLAN.

The attack uses a fact: the table has a fixed size.

1. The attacker sends a very large number of frames, each with a different fake source MAC. Old tools like `macof` do this.
2. The switch learns every fake address, and the table fills up.
3. Once the table is full, the switch cannot learn new real addresses.
4. Real entries age out over time. Traffic to those hosts becomes "unknown destination".
5. The switch floods that traffic. The attacker's port now receives frames meant for other people.

```text
Before the attack                After the table is full

 PC-A ---> [SW] ---> PC-B         PC-A ---> [SW] ---> PC-B
            |                                |
            +--  Attacker sees               +---> Attacker sees
                 nothing                           a copy of it
```

The numbers show why this is fast. Assume a small switch has room for 8,000 MAC addresses (this is only an example; real sizes depend on the model). A 1 Gbps port can carry about 1,488,000 of the smallest frames per second. So in theory, the table fills in under 6 milliseconds.

**Analogy:** someone adds thousands of fake names to the mail clerk's notebook. There are no blank lines left, so the clerk starts reading all mail aloud. **Where it stops being true:** the flooding only happens inside the attacker's VLAN. It is more like reading aloud on one floor than in the whole building.

**Fuller truth:**

- The switch keeps entries it already learned until they age out, so the effect builds up over time. The attacker must keep sending.
- The table is usually a hash table, so some addresses may fail to fit before it is 100% full.
- Flooding also wastes bandwidth and can slow every host in the VLAN.

### 2. Port Security

Port security limits which MAC addresses, and how many, can use a switch port. It is a Cisco name. Other vendors often call it MAC limiting.

**Analogy:** a door that lets in only the first two ID cards it sees. **Where it stops being true:** a MAC address is not a real ID card. Anyone can copy it, so port security limits numbers but does not prove identity.

```text
! Cisco IOS: basic port security on an access port
interface GigabitEthernet0/3
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
```

The port must be set as a static access port first. If it is still in a dynamic mode, IOS rejects the command with an error that says the port is a dynamic port.

**Defaults on Cisco Catalyst switches once port security is enabled:**

| Setting | Default |
|---|---|
| Maximum MAC addresses | 1 |
| Violation mode | shutdown |
| Aging | disabled |
| Sticky learning | disabled |

#### Types of secure MAC address

| Type | How it is learned | Survives a reload? |
|---|---|---|
| Static secure | You type it by hand | Yes, if you save the config |
| Dynamic secure | Learned from traffic | No |
| Sticky secure | Learned from traffic, then written to the running config | Yes, if you save the config |

#### How sticky works

Sticky gives you the best of both. You do not need to type each MAC address, but the switch remembers it.

1. The first device sends a frame.
2. The switch learns the MAC address.
3. The switch adds a line to the running config.

```text
interface GigabitEthernet0/3
 switchport port-security mac-address sticky 0050.7966.6803
```

The running config is the live config in memory. If you do not save it to the startup config, the sticky addresses are lost when the switch reloads. The command to save is `copy running-config startup-config`.

#### Violation modes

A **violation** happens when the port sees one MAC address too many, or sees a MAC address that is already secured on a different secure port in the same VLAN.

| Mode | Drops bad frames | Sends log or SNMP trap | Counts violation | Disables port |
|---|---|---|---|---|
| protect | Yes | No | No | No |
| restrict | Yes | Yes | Yes | No |
| shutdown (default) | Yes | Yes | Yes | Yes |

An easy way to remember the order: from the quietest to the loudest, **p**rotect, **r**estrict, **s**hutdown (P, R, S are in alphabetical order).

SNMP is a protocol for monitoring network devices. A trap is an alert it sends.

**When to use each mode:**

- Use `shutdown` on sensitive ports where you want a clear stop.
- Use `restrict` when the good device must stay online, and you still want an alert.
- Avoid `protect` in most cases, because you get no evidence of an attack.

Some platforms also offer `shutdown vlan`, which disables only the VLAN that broke the rule. That is useful on a port shared by a phone and a PC.

#### Err-disabled state

In `shutdown` mode the port goes to the **err-disabled** state. This means the switch has turned the port off because of an error. The link goes down. It stays down until someone acts.

```text
! Manual recovery
interface GigabitEthernet0/3
 shutdown
 no shutdown

! Automatic recovery after 600 seconds (default interval is 300)
errdisable recovery cause psecure-violation
errdisable recovery interval 600
```

Automatic recovery is off by default. If the attacker is still there, the port will just shut down again.

### 3. DHCP Starvation and Rogue DHCP

DHCP gives a host its IP address, mask, gateway and DNS servers. The normal exchange has four messages, known as DORA: Discover, Offer, Request, Acknowledge. Clients use UDP port 68 and servers use UDP port 67. See [DHCP](../07-application-protocols/04-dhcp.md) for the details.

#### DHCP starvation

**Goal:** use up all addresses in the pool, so real clients get nothing. A pool is the range of addresses a server can give out.

1. The attacker sends many DHCP Discover messages. Each one has a different fake client MAC address. Tools like `yersinia` and `dhcpig` do this.
2. The server offers an address to each fake client and reserves it.
3. The attacker completes the exchange so the leases are stored.
4. The pool becomes empty.

If the pool is 10.10.10.100-199, it has only 100 addresses. A few hundred fake messages use them all.

Starvation is a denial of service attack: it stops a service from working. It is also often the first step before the next attack, because a real server that is empty cannot compete with a fake one.

#### Rogue DHCP

A rogue DHCP server is one that nobody approved. It sends offers to clients. The client usually takes the first offer it receives. A nearby rogue server can easily answer faster than a real server that is far away.

The rogue server hands out bad settings:

| Setting given | What the attacker gains |
|---|---|
| Default gateway = attacker | All traffic to other networks goes through the attacker |
| DNS server = attacker | The attacker can send names to wrong addresses |
| Bad gateway or fake subnet | Clients cannot reach anything (denial of service) |

When traffic is pushed through the attacker, this is an **on-path attack**. The older name is man-in-the-middle.

```text
Real server is far away, rogue server is on the same switch

 PC-A       Attacker (rogue DHCP)    Real DHCP server
  |                  |                      |
  |--- Discover (broadcast) ------+-------->|
  |<-- Offer (GW 10.10.10.102) ---|         |
  |                                         |
  |<-------------- Offer (GW 10.10.10.1) ---|  arrives late
  |
  +-- PC-A takes the first offer, attacker is now its gateway
```

Rogue DHCP does not always mean an attack. A common real case is a user who plugs a small home router into an office socket, with the wrong port connected. That router starts giving out 192.168.1.x addresses. Users suddenly lose access. DHCP snooping stops this too.

### 4. DHCP Snooping

DHCP snooping is a switch feature that watches DHCP traffic and blocks server replies from ports that should not have a server.

**Analogy:** a guard stands near the front desk. Only staff at the real desk may hand out room keys. The guard also writes down each key: who got it, which room and which door they came through. **Where it stops being true:** the guard reads only DHCP messages. It does nothing about a host that never uses DHCP.

The switch sorts its ports into two groups:

| Port type | Typical use | DHCP server replies |
|---|---|---|
| Trusted | Uplink toward the real server, or the port of that server | Allowed |
| Untrusted (default) | User ports | Dropped |

```text
                [ Real DHCP server ]
                         |
                 Gi0/24  TRUSTED
                         |
                     [ SW1 ]
                    /       \
           Gi0/3 untrusted  Gi0/4 untrusted
                /               \
             PC-A             Attacker
                               (Offer dropped here)
```

#### What snooping checks

On an untrusted port, the switch drops:

1. **Server messages:** DHCPOFFER, DHCPACK, DHCPNAK and DHCPLEASEQUERY replies.
2. **Client messages with a MAC mismatch:** the Ethernet source MAC differs from `chaddr`, the client hardware address field inside the DHCP message. This check is on by default and is controlled by `ip dhcp snooping verify mac-address`.
3. **Fake releases:** DHCPRELEASE or DHCPDECLINE for a client whose binding is on another port. This stops an attacker from ending someone else's lease.
4. **Relay-style messages:** a packet with a non-zero `giaddr` (the relay agent address field), or one with Option 82 when that is not allowed.

The second check matters for starvation. A clever attacker keeps one real source MAC and changes only `chaddr`. Port security sees only one MAC and does nothing. DHCP snooping sees the mismatch and drops the message.

#### The binding table

This is the most valuable result of DHCP snooping. When the switch sees a DHCPACK, it records the lease.

```text
SW1#show ip dhcp snooping binding
MacAddress         IpAddress     Lease(sec) Type          VLAN Interface
-----------------  ------------  ---------- ------------- ---- ---------
00:50:79:66:68:03  10.10.10.101  85912      dhcp-snooping 10   Gi0/3
00:50:79:66:68:66  10.10.10.102  86011      dhcp-snooping 10   Gi0/4
Total number of bindings: 2
```

Read one row as a statement: "The device with this MAC, using this IP, in this VLAN, is on this port." Dynamic ARP Inspection and IP Source Guard both depend on this table.

#### Basic configuration

```text
! Cisco IOS
ip dhcp snooping
ip dhcp snooping vlan 10,20
!
interface GigabitEthernet0/24
 description UPLINK-TOWARD-DHCP-SERVER
 ip dhcp snooping trust
!
interface GigabitEthernet0/3
 ip dhcp snooping limit rate 15
```

Both global commands are needed: the first turns on the feature, and the second chooses the VLANs.

The rate limit is in packets per second. By default there is no limit. If a port goes over its limit, it becomes err-disabled, with the cause `dhcp-rate-limit`.

**Why 15?** It is a common choice, not a standard. A normal host sends only a few DHCP messages when it joins. Choose a value that real clients never reach.

### 5. ARP Poisoning

ARP (Address Resolution Protocol) finds the MAC address that belongs to an IP address on the local network. See [ARP](05-arp.md). ARP has no authentication. A host cannot check whether a reply is true.

ARP poisoning, also called ARP spoofing, means sending false ARP replies so a victim stores a wrong entry.

Here is the setup, used through the rest of this chapter:

| Device | IP address | MAC address |
|---|---|---|
| Gateway | 10.10.10.1 | 0050.7966.6801 |
| PC-A (victim) | 10.10.10.101 | 0050.7966.6803 |
| Attacker | 10.10.10.102 | 0050.7966.6866 |

1. The attacker tells PC-A: "10.10.10.1 is at 0050.7966.6866."
2. The attacker tells the gateway: "10.10.10.101 is at 0050.7966.6866."
3. Both devices update their ARP cache. An ARP cache is the local list of IP-to-MAC mappings.
4. PC-A now sends gateway traffic to the attacker. The gateway sends PC-A's traffic to the attacker.
5. The attacker forwards the packets, so nobody notices. It can read them, or change anything that is not encrypted.
6. The attacker repeats the false replies every few seconds, so the entries stay wrong.

```text
Normal path                     After poisoning

PC-A --------> Gateway          PC-A          Gateway
                                   \          /
                                    v        v
                                    Attacker
                             (reads and forwards both ways)
```

**Analogy:** a stranger stands in the hallway and says "I am the bank courier, give me your letters". Then he passes the letters along after reading them. **Where it stops being true:** a person might ask to see a badge. ARP has no way to ask.

**Fuller truth:** behaviour differs by operating system. For example, Linux by default does not create a new entry from an unsolicited ARP reply, but it can still update an entry that exists. This detail does not make Linux safe, because the attacker can repeat replies until they land at the right moment.

#### What it looks like on a host

Check the gateway entry on PC-A.

```bash
# Linux
ip neigh show 10.10.10.1
```

```text
10.10.10.1 dev eth0 lladdr 00:50:79:66:68:66 REACHABLE
```

```powershell
# Windows
arp -a
```

```text
Interface: 10.10.10.101 --- 0x6
  Internet Address      Physical Address      Type
  10.10.10.1            00-50-79-66-68-66     dynamic
  10.10.10.102          00-50-79-66-68-66     dynamic
```

The warning sign is clear: two different IP addresses share the same MAC, and the gateway's MAC has changed from `00-50-79-66-68-01`.

A proxy ARP device or a router with several addresses can also show one MAC for several IPs. So this is a strong hint, not proof.

A static ARP entry on a host blocks poisoning for that one address. It does not scale, but it can be useful on a few critical servers.

```bash
# Linux: fix the gateway entry
sudo ip neigh replace 10.10.10.1 lladdr 00:50:79:66:68:01 dev eth0 nud permanent
```

```powershell
# Windows: fix the gateway entry (run as Administrator)
netsh interface ipv4 add neighbors "Ethernet" 10.10.10.1 00-50-79-66-68-01
```

### 6. Dynamic ARP Inspection

Dynamic ARP Inspection (DAI) is a switch feature that checks ARP messages against a list of known good IP-to-MAC-to-port records. It uses the DHCP snooping binding table.

**Analogy:** when someone shouts "I am Sara", the guard opens the register. If the register does not show that this person at this door is Sara, the guard stops the shout. **Where it stops being true:** the register only knows people who came through the front desk. A device with a manually typed IP address is not in the table.

```text
ARP arrives on a port
        |
        v
  Is the port trusted? ---- yes ---> forward, no check
        |
        no
        |
        v
  Rate over the limit? ---- yes ---> err-disable the port
        |
        no
        |
        v
  ARP ACL match? (if one exists) -- permit ---> forward
        |                        -- deny -----> drop and log
        no match
        |
        v
  Sender IP + sender MAC + VLAN + port
  found in the binding table?
        |              |
       yes             no
        |              |
        v              v
     forward      drop and log
```

The ARP body contains these fields. DAI reads the sender fields.

| Field | Example in a false reply |
|---|---|
| Sender MAC | 0050.7966.6866 |
| Sender IP | 10.10.10.1 (false claim) |
| Target MAC | 0050.7966.6803 |
| Target IP | 10.10.10.101 |

The binding table says 0050.7966.6866 owns 10.10.10.102 on Gi0/4. It does not own 10.10.10.1. So the switch drops the reply.

```text
! Cisco IOS
ip arp inspection vlan 10,20
ip arp inspection validate src-mac dst-mac ip
!
interface GigabitEthernet0/24
 ip arp inspection trust
!
interface GigabitEthernet0/3
 ip arp inspection limit rate 15
```

All ports start as untrusted. On an untrusted port the default limit is 15 ARP packets per second. Going over the limit makes the port err-disabled. A trusted port has no limit by default.

#### Extra validation checks

`ip arp inspection validate` adds optional checks:

| Keyword | What it checks |
|---|---|
| `src-mac` | Ethernet source MAC equals the sender MAC in the ARP body |
| `dst-mac` | Ethernet destination MAC equals the target MAC in ARP replies |
| `ip` | Rejects invalid IPs such as 0.0.0.0, 255.255.255.255 and multicast addresses |

Put all keywords on one line. Each new `validate` command replaces the one before it, so typing them on three lines leaves only the last one active.

#### Hosts with static IP addresses

Servers and printers often have a fixed IP, so DHCP never saw them. Without help, DAI drops their ARP and they go offline. Use an **ARP ACL** (an access list that matches IP and MAC pairs):

```text
arp access-list STATIC-HOSTS
 permit ip host 10.10.10.50 mac host 0050.7966.6805
!
ip arp inspection filter STATIC-HOSTS vlan 10
```

Note that the gateway reaches SW1 through the trusted uplink, so its ARP is not checked on SW1.

### 7. IP Source Guard

IP Source Guard (IPSG) filters normal IP traffic on a port, so a host can only send from the IP address it was given. It uses the same binding table.

- DAI protects the mapping in ARP traffic.
- IPSG protects the source address in IP traffic.

**Analogy:** a guard at the door checks the return address on each outgoing letter. If the sender writes someone else's address, the letter stays inside. **Where it stops being true:** the guard stands at one door only. IPSG works at the edge port. Once a frame passes a port without IPSG, no later switch checks it in this way.

How it works:

1. DHCP snooping fills the binding table.
2. The switch installs a per-port filter that permits only the bound source IP. Cisco calls this kind of filter a port ACL.
3. Before a binding exists, only DHCP traffic is allowed, so the client can still get an address.
4. Traffic with any other source IP is dropped.

```text
! Cisco IOS
interface GigabitEthernet0/3
 ip verify source
```

There are two filtering levels:

| Command | Filters on | Notes |
|---|---|---|
| `ip verify source` | Source IP | Simple and common |
| `ip verify source port-security` | Source IP and source MAC | Needs port security, and Cisco documents that the DHCP server must support Option 82 |

Newer platforms may use a different keyword, such as `ip verify source mac-check`. Check the guide for your platform.

Hosts with static IPs need a manual binding:

```text
ip source binding 0050.7966.6805 vlan 10 10.10.10.50 interface Gi0/5
```

**Why spoof a source IP?** An attacker can hide their identity or send traffic that looks like it comes from a trusted host. A common use is a reflection attack, covered in [Common Network Attacks](../09-security/03-common-network-attacks.md).

### 8. VLAN Hopping

VLANs keep traffic apart at Layer 2. A VLAN hopping attack sends traffic into a VLAN the attacker should not reach, without going through a router or firewall.

A **trunk** is a link that carries many VLANs. Each frame carries a small label called a tag, defined in IEEE 802.1Q. The **native VLAN** is the one VLAN whose frames travel across a trunk with no tag. See [Trunking and 802.1Q](07-trunking-8021q.md).

```text
802.1Q tag (4 bytes, placed after the source MAC)
+---------------------+-------+-------+-------------+
| TPID = 0x8100       | PCP   | DEI   | VID         |
| 16 bits             | 3 bit | 1 bit | 12 bits     |
+---------------------+-------+-------+-------------+
TPID: says "this is a tagged frame"
PCP:  priority
DEI:  drop eligible
VID:  VLAN number
```

#### Switch spoofing

**DTP (Dynamic Trunking Protocol)** is a Cisco protocol that lets two connected ports agree to form a trunk. The attacker's device pretends to be a switch and sends DTP messages.

| Local mode | Neighbour: auto | Neighbour: desirable | Neighbour: trunk |
|---|---|---|---|
| dynamic auto | Access | Trunk | Trunk |
| dynamic desirable | Trunk | Trunk | Trunk |

Many Catalyst switches use `dynamic auto` as the port default. Some older models used `dynamic desirable`. Both will form a trunk if the other side asks, so the attack runs like this:

1. The port is in a dynamic mode because nobody changed the default.
2. The attacker sends a DTP message that asks for a trunk.
3. The port becomes a trunk.
4. The attacker sends frames tagged with any allowed VLAN, and receives traffic from all of them.

```text
SW1#show interfaces GigabitEthernet0/4 switchport
Name: Gi0/4
Switchport: Enabled
Administrative Mode: dynamic auto
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Negotiation of Trunking: On
Access Mode VLAN: 10 (USERS)
Trunking Native Mode VLAN: 1 (default)
```

`Operational Mode: trunk` on a user port is a serious warning sign.

**Analogy:** a stranger puts on a staff uniform, and the door system hands over a key to every floor. **Where it stops being true:** a uniform can fool only a person. Here the machine agrees because it was told to agree with anyone who asks in the right format.

**Defence:**

```text
! User port
interface GigabitEthernet0/3
 switchport mode access
 switchport nonegotiate

! Link to another switch
interface GigabitEthernet0/24
 switchport mode trunk
 switchport nonegotiate
```

`switchport mode access` alone is enough to make a port a non-trunk. `switchport nonegotiate` also stops DTP messages from being sent, which is cleaner. IOS only accepts `nonegotiate` on a port with a static mode.

DTP is Cisco-specific. Cisco Nexus switches do not support it, and other vendors generally do not run it.

#### Double tagging

This attack does not need DTP. It needs these conditions:

1. The attacker's access port is in the same VLAN as the native VLAN of a trunk.
2. The trunk does not tag the native VLAN.
3. The target is on a different switch across that trunk.

Assume the trunk's native VLAN is VLAN 1, the attacker is in VLAN 1, and the target is in VLAN 20.

```text
Frame sent by the attacker

+---------+---------+---------+---------+------+---------+-----+
| Dst MAC | Src MAC | Tag     | Tag     | Type | Payload | FCS |
|         |         | VID 1   | VID 20  | IPv4 |         |     |
+---------+---------+---------+---------+------+---------+-----+
```

```text
Attacker           SW1                SW2               Target
VLAN 1                                                  VLAN 20
   |                |                  |                   |
   |--[1][20] data->|                  |                   |
   |                | SW1 removes [1]  |                   |
   |                | VLAN 1 is native |                   |
   |                |---[20] data----->|                   |
   |                |                  | SW2 reads [20]    |
   |                |                  |---- data -------->|
   |                                                       |
   |   Reply goes to VLAN 20 and never reaches the attacker|
```

1. SW1 receives the frame and reads the outer tag, VLAN 1.
2. SW1 sees that the frame must cross a trunk whose native VLAN is 1.
3. Native frames are sent with no tag, so SW1 removes the outer tag. It does not look deeper.
4. The frame still has the inner tag, VLAN 20.
5. SW2 receives it and believes it belongs to VLAN 20.

This is **one-way**. The attacker can send into VLAN 20, but replies never come back. So a normal TCP connection cannot form. One-way attacks are still possible, for example sending a flood of UDP.

**Analogy:** a letter sits inside a second envelope. The first sorter removes the outer envelope, then sends the letter on. The next sorter reads the inner one and delivers it to a floor the sender could not reach. **Where it stops being true:** a person would notice two envelopes. A switch reads only the tag it needs, then moves on.

**Fuller truth:** an access port may treat a received tagged frame differently, depending on platform and software. Many modern switches drop it. Do not rely on that. Fix the design instead.

**Defence:**

```text
! Move the native VLAN to a VLAN that no host uses
interface GigabitEthernet0/24
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20
 switchport nonegotiate

! Optional: tag native VLAN frames on all trunks (platform dependent)
vlan dot1q tag native
```

Why this works: if no access port is in VLAN 999, no attacker has a frame that SW1 will send untagged. With native tagging, SW1 keeps the outer tag on the trunk. SW2 then reads VLAN 1, not VLAN 20.

Change the native VLAN on both ends of the trunk. A mismatch can leak traffic between VLANs, and CDP (Cisco Discovery Protocol) reports it as a native VLAN mismatch.

### 9. Storm Control

A **broadcast storm** happens when broadcast frames multiply and fill the network. The usual cause is a Layer 2 loop or a faulty network card. An attacker can also cause one on purpose.

Storm control measures incoming traffic on a port. If a traffic type goes above a level, the switch drops that type until it falls again. It is off by default.

**Analogy:** a safety valve on a water pipe. If the flow gets too high, the valve closes until the pressure drops. **Where it stops being true:** a valve does not care about the kind of water. Storm control looks at traffic type, and it cannot tell good broadcasts from bad ones. Both get dropped.

```text
! Cisco IOS
interface GigabitEthernet0/3
 storm-control broadcast level 1.00 0.50
 storm-control multicast level 5.00 3.00
 storm-control action trap
```

- The first number is the **rising** level, in percent of port bandwidth. Above it, the switch starts dropping.
- The second number is the **falling** level. Below it, the switch starts forwarding again.
- If you give no falling level, the falling level equals the rising level.
- Cisco also accepts absolute values with `level bps` and `level pps` on many platforms.
- Traffic is measured over short intervals, commonly 1 second on Catalyst switches.

**The numbers:** on a 1 Gbps port, 1% is 10 Mbps. That is about 14,880 of the smallest frames per second, which is far more broadcast than a normal user device sends.

**Default action and options:**

| Action | Result |
|---|---|
| None configured | Filter: drop excess traffic, port stays up |
| `storm-control action trap` | Filter, and also send an SNMP trap |
| `storm-control action shutdown` | Port becomes err-disabled |

```text
SW1#show storm-control GigabitEthernet0/3 broadcast
Interface  Filter State   Upper        Lower        Current
---------  -------------  -----------  -----------  ----------
Gi0/3      Forwarding     1.00%        0.50%        0.12%
```

**Be careful:**

- Multicast limits can hit real protocols. OSPF, video and voice paging use multicast, so a low limit can break them.
- On some platforms `unicast` means all unicast, and on others only unknown unicast. Check the guide before setting a low value.
- Storm control does not replace STP (Spanning Tree Protocol). STP removes the loop. Storm control limits the damage. See [Spanning Tree Protocol](09-spanning-tree.md).

### 10. Unused Port Practices

Every unused port is an open door. Lock it the same way on every switch.

1. Shut the port down with `shutdown`.
2. Set it as an access port.
3. Put it in a **parking VLAN**, also called a black hole VLAN. This is a VLAN with no users, no gateway and no trunk access.
4. Turn off DTP.
5. Add a clear description.

```text
! Cisco IOS
vlan 998
 name PARKING
!
interface range GigabitEthernet0/6 - 23
 description UNUSED
 switchport mode access
 switchport access vlan 998
 switchport nonegotiate
 shutdown
```

Why do all of this when `shutdown` should be enough? Because someone may run `no shutdown` by mistake later. The other settings are a second layer: the port still lands in an empty VLAN.

More good habits:

- Do not use VLAN 1 for users or management. It is the factory default, so it is what attackers try first.
- Do not use the native VLAN or the parking VLAN for any host.
- Keep the wiring closet locked. Physical access beats most of these controls.
- Check for ports with no use on a regular schedule. `show interfaces status` shows which ports are up, and each `show interfaces` output shows the time since last input.

### 11. How the Features Depend on Each Other

```text
DHCP traffic
     |
     v
DHCP snooping ---> builds ---> Binding table
                                |        |
                                v        v
                              DAI       IP Source Guard
                         (checks ARP)  (checks IP packets)

Port security      works alone (MAC limit)
Storm control      works alone (traffic levels)
DTP off            works alone (port mode)
```

So the order of setup matters: DHCP snooping first, then DAI and IPSG.

## Example

Here is a complete access switch setup, with five attack attempts and the results.

### Topology

```text
                 [ DHCP server 10.10.50.10 ]
                             |
                 [ D1 distribution switch ]
                   SVI VLAN 10: 10.10.10.1
                   helper to 10.10.50.10
                             |
                      Gi0/24 | trunk, TRUSTED
                             |
                     [ SW1 access switch ]
          +--------+---------+---------+
        Gi0/3    Gi0/4     Gi0/5    Gi0/1-2, Gi0/6-23
        PC-A    Attacker  Printer   unused
```

An SVI (switched virtual interface) is a virtual Layer 3 interface for a VLAN. Here it is the gateway. A helper address makes the switch relay DHCP to a server in another subnet.

| Item | Value |
|---|---|
| VLAN 10 | Users, 10.10.10.0/24, gateway 10.10.10.1 |
| VLAN 20 | Finance, 10.10.20.0/24 |
| VLAN 998 | Parking |
| VLAN 999 | Native VLAN on trunks, no hosts |
| DHCP pool | 10.10.10.100-199 |
| PC-A | 10.10.10.101, 0050.7966.6803, Gi0/3 |
| Attacker | 10.10.10.102, 0050.7966.6866, Gi0/4 |
| Printer (static IP) | 10.10.10.50, 0050.7966.6805, Gi0/5 |

### Configuration on SW1

Cisco IOS. Apply it in stages on a live network (see Going Deeper).

```text
! VLANs
vlan 10
 name USERS
vlan 20
 name FINANCE
vlan 998
 name PARKING
vlan 999
 name NATIVE-UNUSED
!
! DHCP snooping
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
ip dhcp snooping database flash:dhcp-snoop.db
!
! Dynamic ARP Inspection
ip arp inspection vlan 10,20
ip arp inspection validate src-mac dst-mac ip
!
! Printer: static IP, so add manual entries
arp access-list STATIC-HOSTS
 permit ip host 10.10.10.50 mac host 0050.7966.6805
ip arp inspection filter STATIC-HOSTS vlan 10
ip source binding 0050.7966.6805 vlan 10 10.10.10.50 interface Gi0/5
!
! Automatic recovery from err-disabled
errdisable recovery cause psecure-violation
errdisable recovery cause arp-inspection
errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 600
!
! User ports
interface range GigabitEthernet0/3 - 5
 description USER-PORT
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 15
 ip verify source
 storm-control broadcast level 1.00 0.50
 storm-control action trap
 spanning-tree portfast
 spanning-tree bpduguard enable
 no cdp enable
!
! Unused ports
interface range GigabitEthernet0/1 - 2 , GigabitEthernet0/6 - 23
 description UNUSED
 switchport mode access
 switchport access vlan 998
 switchport nonegotiate
 shutdown
!
! Uplink to D1
interface GigabitEthernet0/24
 description UPLINK-TO-D1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
```

Older switches also need `switchport trunk encapsulation dot1q` before `switchport mode trunk`. D1 needs the same native VLAN on its side of the trunk.

Why `no ip dhcp snooping information option`? By default, snooping adds Option 82 to client requests and leaves `giaddr` as zero. An upstream relay or server may drop those messages. Turning off insertion avoids this. Going Deeper explains the fuller picture.

### Verify the baseline

```text
SW1#show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
10,20
DHCP snooping is operational on following VLANs:
10,20
Insertion of option 82 is disabled
Option 82 on untrusted port is not allowed
Verification of hwaddr field is enabled
Verification of giaddr field is enabled
```

```text
SW1#show ip verify source
Interface  Filter-type  Filter-mode  IP-address    Mac-address  Vlan
---------  -----------  -----------  ------------  -----------  ----
Gi0/3      ip           active       10.10.10.101               10
Gi0/4      ip           active       10.10.10.102               10
Gi0/5      ip           active       10.10.10.50                10
```

Each test below starts with a clean port.

### Test 1: MAC flooding from Gi0/4

The port has already learned the attacker's real MAC. The first fake MAC reaches the limit of 2. The next fake MAC breaks the rule.

```text
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred,
 caused by MAC address 8e02.19ab.44c1 on port GigabitEthernet0/4.
%PM-4-ERR_DISABLE: psecure-violation error detected on Gi0/4,
 putting Gi0/4 in err-disable state
```

```text
SW1#show port-security interface GigabitEthernet0/4
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 2
Last Source Address:Vlan   : 8e02.19ab.44c1:10
Security Violation Count   : 1
```

The attack stops after one bad frame. Only one fake MAC entered the sticky list.

**A cleanup detail:** the fake MAC stays as a sticky address. Clear it before putting the port back in service, or the real device may hit the limit.

```text
SW1#clear port-security sticky interface GigabitEthernet0/4
```

### Test 2: Rogue DHCP on Gi0/4

The attacker answers PC-A's Discover with an Offer that names itself as the gateway. Gi0/4 is untrusted, so the switch drops the Offer.

```text
%DHCP_SNOOPING-5-DHCP_SNOOPING_UNTRUSTED_PORT: DHCP_SNOOPING drop
 message on untrusted port, message type: DHCPOFFER,
 MAC sa: 0050.7966.6866
```

PC-A receives only the real Offer and gets gateway 10.10.10.1.

### Test 3: DHCP starvation on Gi0/4

The attacker sends 200 Discover messages per second, from one source MAC, with random `chaddr` values.

- The first check fails: source MAC differs from `chaddr`, so each message is dropped.
- The rate check then fires: the port is over 15 packets per second, so it goes err-disabled with the cause `dhcp-rate-limit`.

Port security alone would not have stopped this, because the source MAC never changed.

### Test 4: ARP poisoning on Gi0/4

The attacker claims to be 10.10.10.1.

```text
%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Res) on Gi0/4, vlan 10.
([0050.7966.6866/10.10.10.1/0050.7966.6803/10.10.10.101/
09:42:17 UTC Mon Mar 2 2026])
```

The format is: sender MAC, sender IP, target MAC, target IP, time.

```text
SW1#show ip arp inspection statistics vlan 10
 Vlan      Forwarded        Dropped     DHCP Drops      ACL Drops
 ----      ---------        -------     ----------      ---------
   10          14231             37             37              0
```

PC-A's gateway entry stays correct at `00:50:79:66:68:01`.

### Test 5: Switch spoofing on Gi0/4

The attacker sends DTP messages. The port ignores them.

```text
SW1#show interfaces GigabitEthernet0/4 switchport
Name: Gi0/4
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Negotiation of Trunking: Off
Access Mode VLAN: 10 (USERS)
```

Result of all five tests:

| Test | Stopped by | Port state afterwards |
|---|---|---|
| MAC flooding | Port security | err-disabled |
| Rogue DHCP | DHCP snooping | up, bad message dropped |
| DHCP starvation | Snooping MAC check, rate limit | err-disabled |
| ARP poisoning | DAI | up, bad message dropped |
| Switch spoofing | Static access mode, no DTP | up, still an access port |

## Going Deeper

### Choosing trust boundaries across several switches

"Trust" means "this port's traffic is not checked here." So a trust setting must match the real path.

- **DHCP snooping:** trust only ports that lead toward a real DHCP server. A downlink to another access switch should stay untrusted, because no server sits there.
- **DAI:** Cisco advises that ports between switches that both run DAI should be trusted, because each edge switch already checks its own hosts.
- **Danger case:** if switch A connects to switch B and B does not run DAI, do not trust that link on A. Otherwise a host on B can poison hosts on A. Use ARP ACLs for those hosts instead.

Every host should be checked once, at the switch nearest to it. This is why these features belong on the access layer.

### Option 82

Option 82 is the DHCP Relay Agent Information option (RFC 3046). It lets a switch add details to a client's request, such as the switch ID and port number. A server can use this to pick an address or policy. ISPs and large campuses use it.

The known trap on Cisco:

1. Snooping is on, and insertion is on by default.
2. The access switch adds Option 82 but leaves `giaddr` as 0.0.0.0, since it is not a relay.
3. An upstream Cisco relay or snooping switch sees Option 82 with a zero `giaddr` on an untrusted port and drops it.
4. Clients stop getting addresses.

Three fixes exist:

| Fix | Where | When to use it |
|---|---|---|
| `no ip dhcp snooping information option` | Access switch | You do not need Option 82 |
| `ip dhcp snooping information option allow-untrusted` | Upstream snooping switch | You want to keep Option 82 from downstream |
| `ip dhcp relay information trusted` | Upstream relay interface | The relay should accept these messages |

### Rolling out without an outage

If you enable DAI first, hosts have no bindings, so their ARP is dropped and they go offline. Use this order:

1. Find every static-IP device in the VLAN, and every DHCP server path.
2. Enable DHCP snooping and set the trusted uplinks. This alone is low risk.
3. Wait for hosts to renew. Under RFC 2131, clients normally start to renew at 50% of the lease time, so a 24-hour lease needs at least 12 hours.
4. Check `show ip dhcp snooping binding`. Compare the count with the number of active hosts.
5. Add ARP ACLs and static bindings for static devices.
6. Enable DAI on one VLAN, then watch the logs.
7. Enable IP Source Guard on a few ports, then widen.
8. Set the binding database, so bindings survive a reload.

Without a binding database, a reload wipes the table. Then DAI and IPSG block hosts until they renew. The database can be stored on flash or on a remote server, depending on the platform.

### Control plane load

On many platforms, DHCP snooping and DAI send the packets to the switch CPU for checking. A flood of ARP or DHCP could then overload the CPU and hurt the whole switch. This is the deeper reason for the rate limits: they protect the switch, as well as the network.

Too low a limit causes problems as well. A port used by a hypervisor, a wireless access point or a downstream switch carries many clients, so it needs a higher limit.

### What port security does not stop

Port security limits MAC counts. It does not confirm who is there.

Suppose an attacker unplugs a printer and connects a laptop with the printer's MAC. The MAC matches, and the port stays up. This is **MAC spoofing**. To defeat it you need 802.1X.

### 802.1X and MACsec

**802.1X** checks identity before a port opens. It has three roles:

- **Supplicant:** the client device.
- **Authenticator:** the switch.
- **Authentication server:** usually RADIUS.

The client proves its identity using EAP, usually with a certificate or username. Until then, the port passes only authentication traffic. For devices that cannot run it, such as old printers, MAC Authentication Bypass (MAB) uses the MAC as a weak fallback. See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

**MACsec (IEEE 802.1AE)** encrypts frames hop by hop on a wired link. It protects against someone tapping a cable between two devices. The traffic is decrypted at each MACsec device, so it is not end-to-end like TLS.

### IPv6 first hop security

DHCP snooping, DAI and IPSG on IPv4 do not cover IPv6. Hosts have IPv6 on by default, even on many "IPv4 only" networks, so these attacks can work there too.

| IPv4 attack | IPv6 version | Defence |
|---|---|---|
| Rogue DHCP | Rogue Router Advertisement | RA Guard (RFC 6105) |
| Rogue DHCP | Rogue DHCPv6 server | DHCPv6 Guard |
| ARP poisoning | Neighbor Discovery spoofing | ND Inspection, or IPv6 Snooping |
| IP spoofing | IPv6 spoofing | IPv6 Source Guard |

A Router Advertisement (RA) tells hosts about a router and a prefix, as part of Neighbor Discovery (RFC 4861). A fake RA can make the attacker a host's IPv6 gateway. RFC 7113 gives advice on making RA Guard harder to evade.

```text
! Cisco IOS: syntax varies by platform
ipv6 nd raguard policy HOSTS
 device-role host
!
interface GigabitEthernet0/3
 ipv6 nd raguard attach-policy HOSTS
```

On newer Catalyst platforms, a device tracking feature set, known as SISF, builds a shared table for these checks. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### Other Layer 2 attacks to know

| Target | Attack | Defence |
|---|---|---|
| STP | Fake root bridge changes the traffic path | BPDU Guard, Root Guard |
| VTP | Rogue switch with a higher revision wipes the VLAN list | VTP transparent or off, VTP password |
| CDP and LLDP | Attacker reads model, version and management IP | Turn them off on user ports |
| Host to host | Attacks inside one VLAN | Private VLANs or port isolation |

- BPDU Guard err-disables an edge port if it receives a BPDU, the control message of STP. See [Spanning Tree Protocol](09-spanning-tree.md).
- VTP (VLAN Trunking Protocol) shares VLAN lists between Cisco switches. See [VLANs](06-vlans.md).
- IP phones may need CDP or LLDP to learn their voice VLAN, so keep those on for phone ports.
- A private VLAN stops hosts in the same subnet from talking directly, and leaves only the gateway reachable. This suits guest networks and hosting.

### Virtual switches and cloud

These attacks also exist in software switches.

- **VMware vSwitch** has three security settings: Promiscuous Mode, MAC Address Changes and Forged Transmits. Set them to Reject unless a workload really needs them. A network appliance or a nested hypervisor might need them.
- **Public clouds** remove much of this attack surface. AWS VPC does not support broadcast, and the platform controls the real MAC and IP of each interface. See [AWS VPC basics](../11-cloud-and-containers/03-aws-vpc-basics.md).

### Layered defence

No single feature is enough. Encryption is the final layer: even if ARP poisoning works, [TLS](../07-application-protocols/07-tls-and-https.md) stops the attacker from reading the traffic. Segmentation limits how far a problem can spread. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### Other vendors

The concepts travel, but the names change. Juniper, for example, uses terms such as MAC limiting, DHCP security, Dynamic ARP Inspection and IP Source Guard. Say this in an interview: it shows you know the mechanism, not only one vendor's commands.

### Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Port is err-disabled | Port security, BPDU Guard, DAI or DHCP rate limit | `show interfaces status err-disabled` |
| Clients get no address after snooping is on | Uplink not trusted, or Option 82 dropped upstream | `show ip dhcp snooping`, then server logs |
| Static host loses network after DAI | No ARP ACL or binding | `show ip arp inspection statistics` |
| All hosts fail after a reload | Binding table lost | `show ip dhcp snooping database` |
| Laptop moved desks, now blocked | Sticky MAC on old port | `show port-security address` |
| New PC blocked at a desk | Old sticky MAC fills the limit | `clear port-security sticky interface` |
| Phone and PC port shuts down | Maximum too low | Raise it (commonly 2 or 3, depending on platform) |
| Routing neighbours drop | Multicast storm level too low | `show storm-control` |

See the [Cisco IOS Command Reference](../12-troubleshooting-and-tools/10-cisco-ios-commands.md) for more commands.

## Common Mistakes

- **Thinking VLANs alone are a security wall.** A VLAN separates traffic, but a badly set trunk can break that wall.
- **Enabling DAI before DHCP snooping has bindings.** Every host loses ARP, and the VLAN goes down.
- **Forgetting static-IP devices.** Printers, servers and gateways on access ports have no binding and need an ARP ACL or static entry.
- **Forgetting to trust the uplink.** All real DHCP replies get dropped, so no client gets an address.
- **Trusting too many ports.** A trusted port skips all checks. Trust as few as possible.
- **Believing port security stops MAC spoofing.** It limits how many MAC addresses a port uses. Anyone can copy a valid MAC.
- **Not saving sticky addresses.** They live in the running config until you save it.
- **Leaving ports in dynamic mode.** A port in `dynamic auto` can still become a trunk.
- **Setting `validate` on several lines.** Each `ip arp inspection validate` command replaces the last one.
- **Thinking double tagging gives two-way access.** It is one-way, since replies cannot return the same way.
- **Using the same VLAN for native and users.** That meets the first condition for double tagging.
- **Treating `shutdown` as enough for unused ports.** Add a parking VLAN and static access mode, so a later mistake is not an open door.
- **Securing IPv4 and ignoring IPv6.** Rogue RAs work even where nobody planned to run IPv6.
- **Setting storm control without measuring traffic first.** A low multicast level can break OSPF, voice or video.
- **Thinking storm control fixes loops.** It limits the damage. STP or a design change removes the loop.

## Interview Questions

**Q: What is MAC flooding, and how do you stop it?**

A: The attacker sends frames with many fake source MACs until the switch's CAM table is full. The switch can then no longer learn real addresses, so it floods traffic for unknown destinations out of all ports in that VLAN, and the attacker can capture it. Port security stops it by limiting the number of MAC addresses on a port.

**Q: Compare the port security violation modes.**

A: All three drop frames from the disallowed MAC. Protect does so silently. Restrict also logs, sends a trap and raises the violation counter, but keeps the port up. Shutdown, the default, does the same and also puts the port in the err-disabled state until recovery.

**Q: What does sticky MAC do, and what can go wrong?**

A: Sticky learns MAC addresses dynamically and writes them into the running config as secure addresses. If the config is not saved, they disappear at reload. Sticky can also cause trouble when a user moves a device, because the old entry stays bound to the old port until it is cleared.

**Q: How does DHCP snooping work?**

A: Ports are trusted or untrusted, and all ports are untrusted by default. The switch drops DHCP server messages on untrusted ports, which stops rogue servers. It checks client messages too, for example that the source MAC matches chaddr. It also builds a binding table of MAC, IP, VLAN and port from the DHCP ACK messages.

**Q: Why does Dynamic ARP Inspection need DHCP snooping?**

A: DAI has to compare each ARP message with a trusted record, and the snooping binding table is that record. It checks the sender IP and MAC against the table for that port and VLAN. Without snooping, DAI needs ARP ACLs that you write by hand. So in practice, you set up snooping first.

**Q: A user's PC with a static IP lost its connection right after DAI was enabled. Why, and how do you fix it?**

A: The PC never used DHCP, so it has no entry in the binding table, and DAI drops its ARP. I would confirm it in the DAI statistics and logs, then add an ARP ACL for that IP and MAC and apply it with ip arp inspection filter. For IP Source Guard, I would also add a static ip source binding.

**Q: What is the difference between DAI and IP Source Guard?**

A: DAI checks ARP messages, so it protects the IP-to-MAC mappings in hosts' ARP caches. IP Source Guard checks ordinary IP traffic on a port, and it drops packets whose source IP is not bound to that port. Both use the same table, and they cover different kinds of spoofing.

**Q: Explain the two types of VLAN hopping.**

A: Switch spoofing abuses DTP. The attacker pretends to be a switch, forms a trunk with a dynamic port, and can then reach all allowed VLANs. Double tagging puts two tags on a frame. The first switch strips the outer tag because it matches the trunk's native VLAN, and the second switch delivers the frame based on the inner tag.

**Q: Why is double tagging one-way, and what stops it?**

A: The frame reaches the target because of a trick on the first switch, but the reply is a normal frame in the target's VLAN, which has no path back into the attacker's VLAN. To stop it, use a native VLAN that has no hosts, keep users out of it, and tag the native VLAN where the platform supports it. Allowing only needed VLANs on trunks helps as well.

**Q: You enabled DHCP snooping and now no client gets an address. What do you check?**

A: First, the uplink toward the server must be set as trusted, otherwise the switch drops the Offers. Second, Option 82: the switch inserts it with a zero giaddr, and an upstream device may drop that. I would try no ip dhcp snooping information option, or allow it on the upstream device. Then I would check the VLAN is in the snooping list and read the logs.

**Q: What is storm control, and when does it cause harm?**

A: It watches incoming broadcast, multicast or unicast traffic on a port. Above a set level, it drops that traffic type, or shuts the port if configured that way. It is a safety net for loops and faulty devices. It can harm real traffic if the level is too low, for example a multicast limit that breaks OSPF or video.

**Q: How would you harden a new access switch?**

A: I would put user ports in static access mode with DTP off, then add port security, BPDU Guard and PortFast. I would enable DHCP snooping, then DAI and IP Source Guard, with trust only on uplinks. Trunks get an unused native VLAN and an allowed VLAN list. Unused ports get shut down and moved to a parking VLAN. I would add storm control and IPv6 RA Guard, and use 802.1X where possible.

## Quick Recap

- Layer 2 protocols have no built-in identity check, and firewalls do not see traffic inside a VLAN, so the access switch must enforce the rules.
- Port security limits MAC addresses per port. Sticky remembers them. Violation modes are protect, restrict and shutdown (the default).
- DHCP snooping blocks server messages on untrusted ports and builds the binding table of MAC, IP, VLAN and port.
- DAI uses that table to stop ARP poisoning. IP Source Guard uses it to stop source IP spoofing.
- Stop switch spoofing with static access mode and no DTP. Stop double tagging with an unused native VLAN and tagged native traffic.
- Storm control limits floods. Unused ports should be shut down, in a parking VLAN, with no trunking.
- Roll out in order: snooping first, wait for bindings, then DAI and IPSG. Remember static devices and IPv6.
