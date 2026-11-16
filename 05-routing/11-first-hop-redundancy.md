# First Hop Redundancy (HSRP, VRRP, GLBP)

**Level:** Advanced | **Prerequisites:** [The Default Gateway](../04-ip-addressing/08-default-gateway.md), [ARP](../03-ethernet-and-switching/05-arp.md), [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md), [How Routers Work](01-how-routers-work.md)

## In One Line

A first hop redundancy protocol lets two or more routers share one virtual gateway IP and one virtual MAC address, so hosts keep working when a router fails and nothing changes on the hosts.

## Why It Matters

A host sends all traffic for other networks to its default gateway. The default gateway is the router address the host uses to leave its own subnet. A normal host has one default gateway, and it does not run a routing protocol. It has no way to learn that the gateway has died.

So a LAN with 500 PCs and one gateway has a single point of failure. A single point of failure is one part that stops everything when it breaks.

Adding a second router does not fix this by itself.

```text
                 Internet
                /        \
     +---------+          +---------+
     | R1      |          | R2      |
     | .1 DEAD |          | .2 OK   |
     +----+----+          +----+----+
          |                    |
     =====+====== switch ======+=====  192.168.10.0/24
                     |
               +-----+------+
               | PC .50     |
               | GW = .1    |
               +------------+

  R2 is healthy, but the PC still sends to 192.168.10.1.
  The PC will never try 192.168.10.2 on its own.
```

You could change the gateway on 500 PCs, but that is slow. DHCP does not save you either, because a host only asks for new settings when its lease is due for renewal. Users would wait minutes or hours. [DHCP](../07-application-protocols/04-dhcp.md) hands out one gateway address and then stays out of the way.

First hop redundancy solves this. The "first hop" is the first router a packet reaches on its way out of the LAN. A first hop redundancy protocol (FHRP) makes that first hop survive a failure.

Interviewers like this topic for three reasons:

- It tests whether you understand how a host really uses its gateway: IP, then ARP, then MAC.
- It has exact facts that show real experience: timers, states, multicast addresses, default preempt behaviour.
- It has classic design traps, such as tracking without preempt, or a gateway on a different switch from the spanning tree root. Spanning tree is the protocol that blocks spare switch links to stop loops, and the root is its central switch.

## Simple Explanation

Think of a company with one public phone number. Customers call that number and never think about who picks up. Behind the number are two receptionists. One is on duty. The other sits next to her and checks every few seconds that she is still there. If she falls ill, the second one starts taking calls on the same number. Customers never change the number they dial.

- The public phone number is the virtual IP address.
- The phone line behind that number is the virtual MAC address.
- The receptionist on duty is the active router.
- The second receptionist is the standby router.
- "Are you still there?" is the hello message.

**Where the analogy stops being true:** a real receptionist can tell her colleague "I feel sick". A router that loses power says nothing. The standby router only knows because the hello messages stop arriving, so there is always a short gap. Also, calls in progress drop when a receptionist leaves, but TCP sessions through a router usually survive, because the host keeps the same gateway and only loses a few packets.

## How It Works

### The single gateway problem, step by step

Here is what a host does to reach 8.8.8.8 through its gateway:

1. The host compares 8.8.8.8 with its own subnet. It is not local.
2. The host must send the packet to the gateway, 192.168.10.1.
3. The host needs the MAC address of 192.168.10.1, so it uses [ARP](../03-ethernet-and-switching/05-arp.md) (Address Resolution Protocol). ARP asks "who has this IP?" and gets back a MAC address.
4. The host stores the answer in its ARP cache and builds a frame to that MAC address.

The host depends on two values: the gateway IP and the gateway MAC. If both keep working after a failure, the host never notices. That is the whole trick.

### The big idea: virtual IP and virtual MAC

Every router keeps its own real IP address and its own real MAC address. On top of these, the routers in a group share two extra addresses:

- **Virtual IP (VIP):** an IP address that no single box permanently owns. Hosts use it as their default gateway.
- **Virtual MAC:** a MAC address that goes with the VIP. It is a value the protocol builds, not one burned into hardware.

The router in charge answers ARP requests for the VIP with the virtual MAC. It also accepts and forwards any frame sent to the virtual MAC.

```text
   R1 real: 192.168.10.2         R2 real: 192.168.10.3
   ACTIVE                        STANDBY
        \                           /
         \   VIP 192.168.10.1      /
          \  VMAC 0000.0c9f.f00a  /
           +----- shared by ----+
                     |
                  switch
                     |
              PC 192.168.10.50
              gateway = 192.168.10.1
              ARP: .1 is at 0000.0c9f.f00a
```

When the active router dies, this happens:

1. The standby router stops hearing hello messages. A hello is a small "I am alive" packet sent on a fixed timer.
2. After a set time, the standby router decides the active router is gone.
3. It starts answering for the VIP and accepting frames for the virtual MAC.
4. It sends a **gratuitous ARP**. This is an ARP message nobody asked for. It says "this IP is at this MAC".
5. The switch sees that frame arrive on a new port and moves the virtual MAC in its MAC address table. The MAC address table (also called the CAM table) tells a switch which port leads to which MAC. See [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md).
6. Frames for the virtual MAC now reach the new router.

Notice what did not change: the host's gateway IP, and the MAC in its ARP cache. The host does not need to do anything. Only the switch had to learn something new.

### HSRP

HSRP stands for Hot Standby Router Protocol. Cisco created it, and only Cisco devices normally run it. RFC 2281 describes version 1, but that RFC is informational, which means it documents the protocol and does not make it a standard. Cisco IOS configures it with the keyword `standby`.

A set of routers that share one VIP is called a **group**. Each group has a number. One LAN can have more than one group.

#### HSRP messages

HSRP runs over UDP port 1985. Messages go to a multicast address, which is one address that reaches every interested device on the link. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

| Item | HSRP version 1 | HSRP version 2 |
|---|---|---|
| Multicast address | 224.0.0.2 (all routers) | 224.0.0.102 |
| UDP port | 1985 | 1985 |
| Group numbers | 0-255 | 0-4095 |
| Virtual MAC | 0000.0C07.ACxx | 0000.0C9F.F000 to 0000.0C9F.FFFF |
| IPv6 | No | Yes |
| Millisecond timers in the hello | No | Yes |

Version 1 and version 2 do not work together. If you use one version on R1 and the other on R2, they cannot see each other.

HSRP has three main message types:

- **Hello:** "I am alive, and here are my state and priority."
- **Coup:** "I want to take over the active role." A coup is sent by a higher priority router.
- **Resign:** "I no longer want to be active." A router sends this on a clean shutdown, so failover can be almost instant.

#### HSRP virtual MAC

The last part of the virtual MAC holds the group number in hexadecimal. Hexadecimal is base 16, where 10 is written `A`. See [Binary and Hexadecimal](../01-foundations/07-binary-and-hexadecimal.md).

| Version | Group | Group in hex | Virtual MAC |
|---|---|---|---|
| 1 | 1 | 01 | 0000.0C07.AC01 |
| 1 | 10 | 0A | 0000.0C07.AC0A |
| 1 | 200 | C8 | 0000.0C07.ACC8 |
| 2 | 10 | 00A | 0000.0C9F.F00A |
| 2 | 1000 | 3E8 | 0000.0C9F.F3E8 |

`0000.0C` is a prefix assigned to Cisco. See [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md). This pattern lets you read a group number straight from an ARP table.

#### HSRP states

An HSRP router moves through these states:

```text
  Initial --> Learn --> Listen --> Speak --> Standby --> Active
                          ^          |          |          |
                          |          |          |          |
                          +----------+----------+----------+
                          (lost election, or another router
                           took the role)
```

| State | What it means | Sends hellos? |
|---|---|---|
| Initial | Just starting. HSRP was just configured or the interface just came up. | No |
| Learn | Has not seen a valid hello from the active router and does not know the VIP yet. Only seen if the VIP was not configured locally. | No |
| Listen | Knows the VIP. It is neither active nor standby. It watches hellos. | No |
| Speak | Sends hellos and takes part in the election. | Yes |
| Standby | Next in line. Only one router per group holds this state. | Yes |
| Active | Owns the VIP and virtual MAC, answers ARP and forwards traffic. | Yes |

A group with three routers shows this well. One is active, one is standby, and the third sits in Listen. It moves to Speak only when the active or standby role becomes empty.

#### HSRP timers

| Timer | Default | Meaning |
|---|---|---|
| Hello | 3 seconds | How often a hello is sent |
| Hold | 10 seconds | How long a router waits with no hello before it treats the sender as down |

The hold time must be longer than the hello time. Cisco's usual rule is that hold should be at least three times hello, so that one or two lost packets do not cause a failover.

With defaults, a sudden power failure on the active router means about 10 seconds with no gateway.

```text
  t=0s    Last hello from R1 reaches R2.
  t=0.1s  R1 loses power. Traffic to the VMAC is dropped.
  t=3s    No hello.
  t=6s    No hello.
  t=9s    No hello.
  t=10s   R2 hold timer expires. R2 becomes Active.
          R2 sends gratuitous ARP. Switch moves the VMAC.
  t=10s+  Traffic flows again through R2.
```

Hosts lose about 10 seconds of packets. TCP sessions normally survive, because TCP resends the lost data. A voice call would hear a clear gap. You can shorten the timers, for example hello 1 and hold 3, and [Going Deeper](#going-deeper) covers faster options.

#### Priority and election

Priority is a number that says how much a router wants to be active. The default is 100, and the value is an 8-bit field, so 255 is the top. Higher wins. If two routers tie, the one with the higher IP address on that interface wins.

Set priority on purpose. If you leave both at 100, the winner is chosen by an accident of addressing.

#### Preempt

Preempt means "take the active role as soon as I have the better priority". In HSRP, preempt is **off by default**.

This surprises many people. Without preempt, an active router keeps the role until it fails, even if a better router joins. So this happens:

1. R1 has priority 110. R2 has priority 100. R1 is active.
2. R1 reboots. R2 becomes active.
3. R1 comes back with priority 110.
4. Without preempt, R2 stays active. R1 goes to standby and waits.

Using the receptionist story: the senior receptionist returns from lunch. Preempt off means she sits down and waits her turn. Preempt on means she takes the phone back at once.

**Where the analogy stops being true:** a human returning from lunch is ready at once. A router that has just booted may have its LAN interface up long before its routing table is full. If it preempts too early, it takes traffic it cannot deliver. So use a preempt delay:

```text
standby 10 preempt delay minimum 60
```

This makes the router wait 60 seconds after it becomes able to preempt.

### VRRP

VRRP stands for Virtual Router Redundancy Protocol. It is an open IETF standard, so mixed vendors can run it together, as well as Linux. The idea is the same as HSRP, with some different words and different defaults.

- VRRP version 2 for IPv4 is RFC 3768.
- VRRP version 3 for IPv4 and IPv6 is RFC 9568, which replaced RFC 5798.

#### VRRP words

| HSRP word | VRRP word |
|---|---|
| Group | Virtual router, numbered by VRID (Virtual Router ID), 1-255 |
| Active | Master |
| Standby | Backup |
| Hello | Advertisement |

**Naming note:** RFC 9568 renamed the Master to **Active**, and it now names its states Initialize, Backup and Active. Most devices, commands and interview questions still say Master, so this chapter does too. Treat the two words as the same thing.

#### VRRP messages and virtual MAC

| Item | VRRP value |
|---|---|
| Transport | IP protocol 112 (no TCP or UDP) |
| IPv4 multicast address | 224.0.0.18 |
| IPv6 multicast address | FF02::12 |
| IPv4 virtual MAC | 0000.5E00.01xx (xx is the VRID in hex) |
| IPv6 virtual MAC | 0000.5E00.02xx (xx is the VRID in hex) |

For VRID 10, the IPv4 virtual MAC is 0000.5E00.010A.

Interviewers often ask "what port does VRRP use?" It is a trick question. VRRP has no port, because it sits directly on IP as protocol 112.

#### VRRP states and behaviour

VRRP has only three states:

```text
                  +------------+
       startup -->| Initialize |
                  +-----+------+
          priority 255  |  priority below 255
              +---------+---------+
              v                   v
        +-----------+  no ads  +-----------+
        |  Master   |<---------|  Backup   |
        |           |--------->|           |
        +-----------+ better   +-----------+
                      Master
                      seen
```

Key differences from HSRP:

- **Only the Master sends advertisements.** In HSRP, both active and standby send hellos. This makes VRRP slightly quieter on a LAN.
- **Preempt is on by default.** In HSRP it is off. This is a very common interview question.
- **Faster default timer.** The Master advertises every 1 second.
- **The VIP can be a real address.** A router whose real interface IP equals the VIP is called the **address owner**. Its priority is automatically 255, and it always takes the Master role when it is up. HSRP does not allow this. In HSRP the VIP must be a spare address.

#### VRRP priority

| Value | Meaning |
|---|---|
| 255 | Reserved for the address owner |
| 1-254 | Normal range. The default is 100. |
| 0 | Sent by a Master that is stopping, to say "I am leaving, take over now" |

If two routers tie on priority, the higher IP address wins.

#### How a Backup decides the Master is gone

VRRP uses a calculated timer called Master_Down_Interval. RFC 9568 calls it Active_Down_Interval.

```text
Skew_Time            = ((256 - Priority) x Advertisement_Interval) / 256
Master_Down_Interval = (3 x Advertisement_Interval) + Skew_Time
```

Worked with the defaults, a 1 second interval and priority 100:

```text
Skew_Time            = (156 x 1) / 256 = 0.609 s
Master_Down_Interval = 3 + 0.609       = 3.609 s
```

Why add a skew? It makes a higher priority Backup react sooner. A Backup with priority 200 has a skew of about 0.219 seconds, so it waits 3.219 seconds. It claims the Master role before a priority 100 Backup does. That avoids a fight between Backups.

### GLBP

GLBP stands for Gateway Load Balancing Protocol. It is also Cisco proprietary. It solves a problem that HSRP and VRRP leave open.

With HSRP or VRRP, one router forwards all the traffic in a group. The other router and its uplink sit idle. You paid for two links and use one. GLBP uses both from one VIP.

Think of a bank with a floor manager at the door and several cashiers. Every customer asks the manager "where do I go?" The manager sends the first to cashier 1, the next to cashier 2, and so on. All customers come through the same door, but the work is spread.

**Where the analogy stops being true:** the manager only assigns customers when they ask. A host asks only when it does an ARP. GLBP does not watch traffic. A host that sends 10 Gbps and a host that sends 1 kbps each count as one host.

#### GLBP roles

| Role | Full name | Job | How many per group |
|---|---|---|---|
| AVG | Active Virtual Gateway | Answers all ARP requests for the VIP and gives each host a virtual MAC | One active, one standby |
| AVF | Active Virtual Forwarder | Forwards traffic sent to its assigned virtual MAC | Up to 4 |

One router can hold both roles. In a two-router group, R1 is often AVG and AVF 1, and R2 is AVF 2.

#### How GLBP shares the load

```text
                      VIP 192.168.10.1
        R1 (AVG + AVF1)            R2 (AVF2)
        VMAC ...0a01               VMAC ...0a02
             |                         |
             +--------- switch --------+
                 |      |      |
                PC1    PC2    PC3

  PC1 ARP for .1 --> AVG replies 0007.b400.0a01 (uses R1)
  PC2 ARP for .1 --> AVG replies 0007.b400.0a02 (uses R2)
  PC3 ARP for .1 --> AVG replies 0007.b400.0a01 (uses R1)
```

All three PCs have the same gateway IP. They get different gateway MACs, and so they use different routers.

The virtual MAC format is 0007.B400.xxyy, where `xx` is the group number and `yy` is the forwarder number. Group 10 forwarder 2 gives 0007.B400.0A02.

#### GLBP facts

| Item | Value |
|---|---|
| Multicast address | 224.0.0.102 |
| UDP port | 3222 |
| Group numbers | 0-1023 |
| Hello and hold | 3 seconds and 10 seconds |
| Priority | Default 100, range 1-255, decides the AVG |
| Weight | Default 100, range 1-254, decides AVF health and load share |
| AVG preempt | Off by default |
| AVF preempt | On by default, with a 30 second delay |

GLBP and HSRP version 2 use the same multicast address, 224.0.0.102. They use different UDP ports, so they do not clash.

Load balancing methods:

| Method | How the AVG chooses | Good for |
|---|---|---|
| Round-robin (default) | Next AVF in turn for each ARP | General use |
| Weighted | In proportion to AVF weights, such as 3 of 4 hosts to the bigger link | Uneven uplinks |
| Host-dependent | Based on the host MAC, so a host always gets the same AVF | Cases that need a steady gateway, such as some NAT or stateful setups |

#### When a GLBP router fails

There are two cases.

**The AVG fails.** The standby AVG takes over the job of answering ARP.

**An AVF fails.** Hosts still hold its virtual MAC in their ARP cache, so someone must keep accepting that MAC. After the hold time, the AVG gives the failed forwarder's MAC to a surviving AVF. That AVF now forwards for two virtual MACs. Two timers then clean up:

- **Redirect timer (default 600 seconds):** while it runs, the AVG can still hand out the old MAC in ARP replies. When it expires, the AVG stops using that MAC for new ARP replies. The surviving AVF still forwards for hosts that already have it.
- **Timeout timer (default 14,400 seconds, which is 4 hours):** when it expires, the old virtual MAC is removed. By then hosts should have refreshed ARP and moved to a working MAC.

### Object tracking

Hello messages run on the LAN side. They only tell the routers that the LAN side is fine. They say nothing about the path out.

```text
           ISP 1                  ISP 2
             X (link dead)          |
             X                      |
        +----+----+            +----+----+
        |   R1    |   hellos   |   R2    |
        | Active  |<---------->| Standby |
        +----+----+  still OK  +----+----+
             |                      |
             +------- switch -------+
                        |
                       PCs
```

R1 is still healthy on the LAN. It keeps sending hellos, so R2 never takes over. Hosts send traffic to R1, and R1 has no way out. This is called a black hole.

**Fuller truth:** if R1 has another route, for example a routing protocol that points to R2 across the LAN, traffic may still get through by taking an extra hop. Without such a route, the traffic dies.

Object tracking fixes this. An object is a thing the router can check, such as an interface or a test result. A tracked object is either Up or Down. When the object goes Down, the FHRP lowers the router's priority.

The receptionist's outside phone line goes dead. She is still at her desk, so her colleague sees nothing wrong. Tracking is the rule "if your outside line dies, lower your rank and let your colleague take calls".

**Where the analogy stops being true:** the rank change is just arithmetic. The router does not hand over on purpose. It only lowers a number, and the takeover happens only if the other router has preempt on.

There are two common things to track:

| Tracks | Command form | Detects |
|---|---|---|
| Interface line protocol (whether the link layer is up) | `track 1 interface GigabitEthernet0/1 line-protocol` | The local link going down |
| IP SLA reachability | `track 1 ip sla 10 reachability` | A remote failure while the local link stays up |

IP SLA is a Cisco feature that sends test traffic, such as pings, on a timer. It matters because a link can look up while the path behind it is dead, for example when a provider device fails one hop away.

The arithmetic must work. With R1 at 110, R2 at 100 and a decrement of 20:

```text
Normal:    R1 = 110    R2 = 100    R1 is active
WAN down:  R1 = 110 - 20 = 90      90 is below 100
           R2 preempts and takes over
WAN back:  R1 = 110    R1 preempts and takes back
```

Two rules follow:

1. The decrement must push the priority **below** the other router's priority, not just down to it.
2. The other router must have **preempt** on. In HSRP, you must add it. For the failback to work, R1 needs it too.

GLBP differs. Tracking lowers the **weight** of an AVF, not its priority. When the weight falls below a lower threshold, that router stops forwarding, and another AVF takes over its virtual MAC.

### Comparison table

| Feature | HSRP | VRRP | GLBP |
|---|---|---|---|
| Standard | Cisco proprietary (RFC 2281 is informational) | IETF standard (RFC 3768, RFC 9568) | Cisco proprietary |
| Forwarding role | Active | Master (Active in RFC 9568) | AVF, up to 4 |
| Spare role | Standby, others in Listen | Backups | Standby AVG, Listen forwarders |
| Sends hellos | Active and Standby | Master only | AVG and AVFs |
| Transport | UDP 1985 | IP protocol 112 | UDP 3222 |
| IPv4 multicast | 224.0.0.2 (v1), 224.0.0.102 (v2) | 224.0.0.18 | 224.0.0.102 |
| Virtual MAC | 0000.0C07.ACxx (v1), 0000.0C9F.Fxxx (v2) | 0000.5E00.01xx | 0007.B400.xxyy |
| Group range | 0-255 (v1), 0-4095 (v2) | 1-255 | 0-1023 |
| Default priority | 100 | 100 | 100 |
| Default timers | Hello 3 s, hold 10 s | Advertise 1 s, down about 3.6 s | Hello 3 s, hold 10 s |
| Preempt default | Off | On | Off for AVG, on for AVF |
| VIP can be a real IP | No | Yes (address owner) | No |
| Load balancing | Only with several groups | Only with several groups | Built in, per host |
| IPv6 | Version 2 | Version 3 | On some Cisco platforms |
| Authentication | Plain text or MD5 | None in the current standard, vendor options exist | Plain text or MD5 |
| States | 6 | 3 | 6 for the AVG role, fewer for the AVF role |

### Which one should you choose?

- **Mixed vendors, or Linux gateways:** VRRP. It is the only open standard of the three.
- **All Cisco, simple design:** HSRP is common and well understood.
- **All Cisco, and you want both uplinks busy with one VIP:** GLBP, if your platform supports it. Support varies by model and software release, so check first.
- **New data centre or modern campus:** often none of the above. See [When you do not need an FHRP](#when-you-do-not-need-an-fhrp).

## Example

All three examples use this network.

```text
        ISP 1 203.0.113.1          ISP 2 198.51.100.1
              |                          |
     Gi0/1 203.0.113.2/30       Gi0/1 198.51.100.2/30
        +-----+-----+              +-----+-----+
        |    R1     |              |    R2     |
        +-----+-----+              +-----+-----+
     Gi0/0 192.168.10.2/24      Gi0/0 192.168.10.3/24
              |                          |
              +--------- SW1 ------------+
                          |
                 PC 192.168.10.50/24
                 Gateway 192.168.10.1 (VIP)

  VLAN 10, subnet 192.168.10.0/24
  Group number 10 (matching the VLAN makes it easy to read)
```

R1 should be the normal gateway. If R1 loses its ISP path, R2 should take over within seconds.

### Example 1: HSRP version 2 with IP SLA tracking

R1 (Cisco IOS):

```text
ip sla 10
 icmp-echo 203.0.113.1 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 10 life forever start-time now
!
track 1 ip sla 10 reachability
 delay down 10 up 15
!
interface GigabitEthernet0/0
 description LAN VLAN 10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt delay minimum 60
 standby 10 timers 1 3
 standby 10 authentication md5 key-string Example-Key-10
 standby 10 track 1 decrement 20
 no shutdown
```

R2 (Cisco IOS):

```text
ip sla 10
 icmp-echo 198.51.100.1 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 10 life forever start-time now
!
track 1 ip sla 10 reachability
 delay down 10 up 15
!
interface GigabitEthernet0/0
 description LAN VLAN 10
 ip address 192.168.10.3 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 100
 standby 10 preempt
 standby 10 timers 1 3
 standby 10 authentication md5 key-string Example-Key-10
 standby 10 track 1 decrement 20
 no shutdown
```

What each part does:

- `ip sla 10` pings the ISP next hop every 5 seconds from the WAN interface. This pings the next hop only, to keep the example simple. A real design often pings something further away, so it also catches failures inside the provider network.
- `track 1 ip sla 10 reachability` turns the ping result into an Up or Down state.
- `delay down 10 up 15` stops a flapping link from causing failovers. The object must stay down for 10 seconds, or up for 15 seconds, before its state changes.
- `standby 10 priority 110` makes R1 the preferred router.
- `standby 10 preempt delay minimum 60` lets R1 take the role back, but only after waiting.
- `standby 10 track 1 decrement 20` takes 110 down to 90 when the ISP path fails.

R2 tracks its own WAN too. If R2's path is dead while R1 is healthy, R2 must not hold the active role.

On a layer 3 switch, the same `standby` commands go under the VLAN interface, for example `interface Vlan10`. You also need `ip routing` turned on. See [Inter-VLAN Routing](../03-ethernet-and-switching/08-inter-vlan-routing.md).

**Check the state:**

```text
R1# show standby brief
                   P indicates configured to preempt.
                   |
Interface  Grp Pri P State   Active        Standby       Virtual IP
Gi0/0      10  110 P Active  local         192.168.10.3  192.168.10.1
```

```text
R1# show standby GigabitEthernet0/0 10
GigabitEthernet0/0 - Group 10 (version 2)
  State is Active
    2 state changes, last state change 00:12:41
  Virtual IP address is 192.168.10.1
  Active virtual MAC address is 0000.0c9f.f00a (MAC In Use)
    Local virtual MAC address is 0000.0c9f.f00a (v2 default)
  Hello time 1 sec, hold time 3 sec
    Next hello sent in 0.432 secs
  Authentication MD5, key-string
  Preemption enabled, delay min 60 secs
  Active router is local
  Standby router is 192.168.10.3, priority 100 (expires in 2.512 sec)
  Priority 110 (configured 110)
    Track object 1 state Up decrement 20
  Group name is "hsrp-Gi0/0-10" (default)
```

Check the virtual MAC: group 10 is 0x00A, so the MAC ends in F00A. That matches.

```text
R1# show track 1
Track 1
  IP SLA 10 reachability
  Reachability is Up
    1 change, last change 00:12:50
  Delay up 15 secs, down 10 secs
  Latest operation return code: OK
  Latest RTT (millisecs) 3
  Tracked by:
    HSRP GigabitEthernet0/0 10
```

**Check from the host:**

Windows:

```powershell
PS C:\> ipconfig | Select-String "IPv4|Gateway"
   IPv4 Address. . . . . . . . . . . : 192.168.10.50
   Default Gateway . . . . . . . . . : 192.168.10.1

PS C:\> arp -a 192.168.10.1

Interface: 192.168.10.50 --- 0xb
  Internet Address      Physical Address      Type
  192.168.10.1          00-00-0c-9f-f0-0a     dynamic
```

Linux:

```bash
$ ip route show default
default via 192.168.10.1 dev eth0

$ ip neigh show 192.168.10.1
192.168.10.1 dev eth0 lladdr 00:00:0c:9f:f0:0a REACHABLE
```

The host shows the virtual MAC, not R1's real MAC. That is correct.

**Test: R1 loses its ISP link.**

R1 log:

```text
%TRACK-6-STATE: 1 ip sla 10 reachability Up -> Down
%HSRP-5-STATECHANGE: GigabitEthernet0/0 Grp 10 state Active -> Speak
%HSRP-5-STATECHANGE: GigabitEthernet0/0 Grp 10 state Speak -> Standby
```

R2 log:

```text
%HSRP-5-STATECHANGE: GigabitEthernet0/0 Grp 10 state Standby -> Active
```

Here is the sequence:

1. The ICMP test to 203.0.113.1 starts failing.
2. After the 10 second down delay, track 1 goes Down. R1's priority falls to 90.
3. R2 has priority 100 and preempt, so it sends a coup.
4. R1 leaves the active role. R2 becomes active and sends a gratuitous ARP.
5. SW1 moves 0000.0c9f.f00a to the port facing R2.
6. The PC keeps its ARP entry. Its traffic now flows through R2 and ISP 2.

**Fuller truth:** the gateway failover is invisible to the PC, but this path change may not be invisible to its sessions. With two ISPs, R2 uses a different public address for [NAT](../04-ip-addressing/09-nat-and-pat.md), so existing sessions through R1 usually break and must reconnect. New sessions work at once.

```text
R1# show standby brief
                   P indicates configured to preempt.
                   |
Interface  Grp Pri P State   Active        Standby       Virtual IP
Gi0/0      10  90  P Standby 192.168.10.3  local         192.168.10.1
```

**Recovery:** when the ISP link returns, track 1 goes Up after 15 seconds. R1's priority returns to 110. R1 waits out its 60 second preempt delay, then takes the role back.

### Example 2: VRRP on the same network

Keep the same `ip sla 10` and `track 1` configuration from Example 1. Classic Cisco IOS syntax, R1:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.2 255.255.255.0
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 110
 vrrp 10 track 1 decrement 20
```

R2:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.3 255.255.255.0
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 100
 vrrp 10 track 1 decrement 20
```

There is no `preempt` line because VRRP preempts by default.

```text
R1# show vrrp brief
Interface  Grp Pri Time Own Pre State  Master addr   Group addr
Gi0/0      10  110 3570     Y   Master 192.168.10.2  192.168.10.1
```

The `Time` column is the Master_Down_Interval in milliseconds. Check it: skew is (256 - 110) / 256 = 0.570 seconds, plus 3 seconds, which gives 3.570 seconds.

```text
R1# show vrrp interface GigabitEthernet0/0
GigabitEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 110
    Track object 1 state Up decrement 20
  Master Router is 192.168.10.2 (local), priority is 110
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.570 sec
```

Newer Cisco IOS XE software uses a different syntax for VRRP version 3, and it covers both IPv4 and IPv6. Exact support varies by platform.

```text
fhrp version vrrp v3
!
interface GigabitEthernet0/0
 vrrp 10 address-family ipv4
  address 192.168.10.1 primary
  priority 110
  timers advertise 1000
 vrrp 20 address-family ipv6
  address FE80::1 primary
  address 2001:db8:10::1/64
  priority 110
```

`timers advertise 1000` means 1000 milliseconds.

**Linux with keepalived.** Keepalived is a common Linux program that speaks VRRP. On a Linux gateway, `/etc/keepalived/keepalived.conf` on the first node could look like this:

```text
vrrp_instance VLAN10 {
    state MASTER
    interface eth0
    virtual_router_id 10
    priority 110
    advert_int 1
    virtual_ipaddress {
        192.168.10.1/24
    }
    track_interface {
        eth1 weight -20
    }
}
```

The second node uses `state BACKUP` and `priority 100`. Watch the packets from either node:

```bash
$ sudo tcpdump -ni eth0 'ip proto 112'
09:21:04.100211 IP 192.168.10.2 > 224.0.0.18: VRRPv2, Advertisement,
  vrid 10, prio 110, authtype none, intvl 1s, length 20
```

tcpdump prints this as one line. It is wrapped here to fit. For more on the tool, see [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

**Fuller truth:** by default, keepalived moves the VIP between hosts and uses the host's real MAC, not 0000.5E00.01xx. Gratuitous ARP then has to update every neighbour's ARP cache. Keepalived has an option, `use_vmac`, to use a true virtual MAC.

### Example 3: GLBP on the same network

This also reuses `track 1` from Example 1. R1:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.2 255.255.255.0
 glbp 10 ip 192.168.10.1
 glbp 10 priority 110
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 weighting 100 lower 70 upper 90
 glbp 10 weighting track 1 decrement 40
```

R2:

```text
interface GigabitEthernet0/0
 ip address 192.168.10.3 255.255.255.0
 glbp 10 ip 192.168.10.1
 glbp 10 priority 100
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 weighting 100 lower 70 upper 90
 glbp 10 weighting track 1 decrement 40
```

The numbers work like this:

- Normal weight is 100.
- If track 1 goes down, the weight becomes 100 - 40 = 60.
- 60 is below the `lower` value of 70, so this router stops forwarding as an AVF.
- It forwards again only when its weight rises above the `upper` value of 90.

```text
R1# show glbp brief
Interface Grp Fwd Pri State  Address        Active router Standby router
Gi0/0     10  -   110 Active 192.168.10.1   local         192.168.10.3
Gi0/0     10  1   -   Active 0007.b400.0a01 local         -
Gi0/0     10  2   -   Listen 0007.b400.0a02 192.168.10.3  -
```

Reading this table:

- The row with `Fwd` set to `-` is the AVG role. R1 is the AVG and owns the VIP.
- The row with `Fwd` 1 shows R1 is forwarder 1 for MAC 0a01.
- The row with `Fwd` 2 shows R2 is forwarder 2. R1 is in Listen for that MAC, ready to take over.

With round-robin, 192.168.10.50 gets MAC 0a01, 192.168.10.51 gets 0a02, 192.168.10.52 gets 0a01, and so on.

If R1's WAN fails, R1 stops forwarding and R2 takes over MAC 0a01. R2 now forwards for all the PCs. R1 stays the AVG, because tracking changed its weight and not its priority. It still answers ARP on the LAN, which is fine because it only hands out MACs that R2 serves.

For uneven links, say a 300 Mbps uplink on R1 and 100 Mbps on R2:

```text
R1(config-if)# glbp 10 load-balancing weighted
R1(config-if)# glbp 10 weighting 75
R2(config-if)# glbp 10 load-balancing weighted
R2(config-if)# glbp 10 weighting 25
```

About 3 of every 4 hosts now get R1's MAC. Note this splits hosts, not bytes.

### Troubleshooting commands

Cisco IOS:

```text
show standby brief
show vrrp brief
show glbp brief
show track
show ip sla statistics
show mac address-table address 0000.0c9f.f00a
debug standby terse
```

Packet filters to see these protocols:

| Protocol | tcpdump (Linux) | Wireshark display filter |
|---|---|---|
| HSRP | `udp port 1985` | `udp.port == 1985` |
| VRRP | `ip proto 112` | `vrrp` |
| GLBP | `udp port 3222` | `udp.port == 3222` |

If **both routers show Active**, the hellos are not reaching each other. Check, in this order:

1. Is the VLAN allowed on the trunk between the routers? See [Trunking and 802.1Q](../03-ethernet-and-switching/07-trunking-8021q.md).
2. Do the group number, version and authentication key match?
3. Does an ACL or a port filter drop the multicast?
4. Is spanning tree blocking the path between them?

## Going Deeper

### Packet layouts

HSRP version 1 message (inside UDP port 1985), 20 bytes:

```text
 Byte 0      Byte 1      Byte 2      Byte 3
+-----------+-----------+-----------+-----------+
| Version   | Op Code   | State     | Hellotime |
+-----------+-----------+-----------+-----------+
| Holdtime  | Priority  | Group     | Reserved  |
+-----------+-----------+-----------+-----------+
| Authentication Data (8 bytes)                 |
|                                               |
+-----------+-----------+-----------+-----------+
| Virtual IP Address (4 bytes)                  |
+-----------+-----------+-----------+-----------+

 Op Code: 0 Hello, 1 Coup, 2 Resign
 State:   0 Initial, 1 Learn, 2 Listen,
          4 Speak, 8 Standby, 16 Active
```

HSRP version 2 uses a different format with type-length-value blocks, which is how it fits larger group numbers and millisecond timers.

VRRP version 3 message (directly inside IP, protocol 112):

```text
 0               8               16              24            31
+-------+-------+---------------+---------------+---------------+
|Version| Type  | VRID          | Priority      | IP Addr Count |
+-------+-------+---------------+---------------+---------------+
| Rsvd  | Max Advert Interval   | Checksum                      |
+-------+-----------------------+-------------------------------+
| IPv4 or IPv6 address(es) of the virtual router                |
+---------------------------------------------------------------+
```

In VRRP version 2, the second row is different: Auth Type (8 bits), Advert Interval in seconds (8 bits), and Checksum (16 bits). It also ends with 8 bytes of old authentication data. Version 3 stores the interval in centiseconds, so 100 means 1 second. This is how it supports sub-second timers.

VRRP packets must have a TTL (Time To Live) of 255 or the receiver drops them. A router lowers TTL by one at each hop, so a packet at 255 cannot have come from another subnet.

### Source MAC of hellos

The HSRP active router sends hellos with the **virtual MAC** as the source. This matters. Switches learn MAC addresses from source addresses, so a constant stream of hellos keeps the virtual MAC fresh in the MAC address table, even if no host is talking to it. Standby routers use their own real MAC for their hellos.

### Failover speed

Default timers give failover times in the region of 10 seconds for HSRP and 3.6 seconds for VRRP. There are three ways to do better:

1. **Lower the timers.** HSRP version 2 can do `standby 10 timers msec 200 msec 700`. VRRP version 3 can use short advertisement intervals. Very short timers cost CPU and can cause false failovers when a device is busy.
2. **Use BFD.** BFD (Bidirectional Forwarding Detection) is a very light hello protocol built only to detect failures fast. Many Cisco platforms can tie HSRP to BFD. Check support for your model.
3. **Avoid sudden failures.** A clean shutdown causes a Resign in HSRP or a priority 0 advertisement in VRRP, so takeover is nearly instant.

VRRP has a detail worth knowing for preemption. A higher priority Backup does not send a coup. It just ignores the lower priority Master's advertisements. Its down timer then expires, and it takes over. So a VRRP preempt can take about one Master_Down_Interval. HSRP's coup is immediate.

### Load sharing with several groups

HSRP and VRRP can still use both routers, just not with one group. There are two ways.

**Per VLAN.** This is the most common design. Each router is active for a different VLAN:

| VLAN | Subnet | VIP | Active | Standby |
|---|---|---|---|---|
| 10 | 192.168.10.0/24 | 192.168.10.1 | D1 | D2 |
| 20 | 192.168.20.0/24 | 192.168.20.1 | D2 | D1 |

**Two groups on one subnet (MHSRP, or Multiple HSRP).** R1 is active for VIP .1 and R2 is active for VIP .254. Half the hosts use .1 and half use .254. It works, but you now need DHCP to give different gateways to different hosts, and this is why it is rare.

### Match the gateway to spanning tree

A VLAN is a separate layer 2 network inside a switch. If your VLAN spans two distribution switches (the switches that connect the access switches), make the HSRP active switch the [spanning tree](../03-ethernet-and-switching/09-spanning-tree.md) root for that VLAN.

```text
  BAD                            GOOD
  D1 = HSRP Active               D1 = HSRP Active + STP root
  D2 = STP root                  D2 = backup for both

    D1 ------ D2                   D1 ------ D2
     X         |                    |         X
     X         |                    |         X
      \       /                      \       /
       Access                         Access

  X = blocked by STP
```

In the bad case, the uplink to D1 is blocked. Host frames go up to D2, then across to D1, which adds an extra link. The good case sends frames straight to the active gateway.

This same reasoning is why GLBP fits poorly when spanning tree blocks an uplink. Half the hosts use each router, so some traffic always has to take the longer path. GLBP is a better fit where both uplinks are forwarding. Good designs also use [link aggregation](../03-ethernet-and-switching/10-link-aggregation.md) to reduce blocked links.

### FHRP only protects the outbound path

HSRP controls which router carries traffic **leaving** the LAN. It does nothing for traffic **coming back**. The return path is chosen by routing protocols, and they may prefer a different router.

```text
  PC --> R1 (Active) --> Internet --> R2 --> PC
```

This is called asymmetric routing. For plain routers it is usually fine. It is a real problem for stateful devices, which remember each session, such as [firewalls](../09-security/04-firewalls.md) and [NAT](../04-ip-addressing/09-nat-and-pat.md) routers, because the return traffic reaches a device with no record of the session. In those cases, you need state sync between devices or routing designed so that both directions take the same path.

### IPv6

IPv6 hosts already have a built-in form of first hop redundancy. Routers announce themselves with Router Advertisements (RAs), so a host can learn more than one default router. Neighbor Unreachability Detection (NUD) then notices when one of them has stopped responding, and the host moves to another.

So why use an FHRP with IPv6? Speed. NUD can take tens of seconds, and the RA default router lifetime is often much longer than that. An FHRP gives quick, controlled failover.

HSRP for IPv6 needs version 2 and uses FF02::66 with UDP port 2029. The virtual MAC range is 0005.73A0.0000 to 0005.73A0.0FFF.

```text
interface GigabitEthernet0/0
 ipv6 address 2001:db8:10::2/64
 standby version 2
 standby 110 ipv6 autoconfig
 standby 110 priority 110
 standby 110 preempt
```

`ipv6 autoconfig` builds a virtual link-local address from the virtual MAC. Hosts then learn this link-local address as the gateway through RAs. The gateway in IPv6 is normally a link-local address, not a global one. For more, see [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### Security

An FHRP is a tempting target. If an attacker on the LAN sends hellos with priority 255, they could win the active role. All traffic then flows through their machine, which is a classic man-in-the-middle attack. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

Protections:

- **HSRP and GLBP:** use MD5 authentication. MD5 here means each packet carries a hash built with a shared key, so a sender without the key is ignored. HSRP's default is plain text with the string `cisco`, which gives no real protection.
- **VRRP:** the current standard has no authentication. The RFC reasoning is that it did not add real security on a LAN. Some vendors offer it for version 2 anyway, such as Cisco's `vrrp 10 authentication md5 key-string`.
- **At the switch:** filter these packets on host-facing access ports with an [ACL](../09-security/05-access-control-lists.md) (access control list, a set of permit and deny rules) or a VLAN ACL. Permit them only on ports toward real routers.
- **On the routers:** if you run ACLs on LAN interfaces, make sure you allow UDP 1985, UDP 3222 and IP protocol 112 as needed. Otherwise your own ACL can break the FHRP.

### When you do not need an FHRP

- **Switch stack (such as Cisco StackWise):** several switches act as one logical switch. There is one gateway, so there is nothing to fail over.
- **Two chassis as one (such as Cisco VSS or StackWise Virtual):** same idea, with one control plane across two boxes.
- **Cisco vPC:** two separate switches share a link bundle. They still run HSRP or VRRP, but both forward traffic sent to the virtual MAC.
- **Routed access:** the gateway moves to the access switch, so the host's first hop is the switch it plugs into. If that switch dies, the host is down anyway. See [Network Design Models](../10-enterprise-and-datacenter/01-network-design-models.md).
- **Distributed anycast gateway:** in [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md) fabrics, every leaf switch owns the same gateway IP and MAC. A host always uses its local leaf.

For the wider picture of redundancy, see [High Availability and Redundancy](../10-enterprise-and-datacenter/07-high-availability.md).

### Public cloud

VRRP depends on multicast and on ARP tricks. Public cloud networks are not plain Ethernet. They typically control which IP is at which instance themselves, so VRRP does not work the normal way. Cloud designs use managed gateways and load balancers, or they move an IP with an API call. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

### Older ways to solve this

These appear in older exam material and trick questions:

- **Proxy ARP:** a host with no gateway ARPs for a remote IP and a router answers for it. See [ARP](../03-ethernet-and-switching/05-arp.md). Failover is slow, because the host keeps the dead router's MAC until its ARP entry expires.
- **ICMP Router Discovery (RFC 1256):** routers announce themselves to hosts. Most hosts never used it.
- **Running a routing protocol on the host:** it works, but it adds complexity on every host.

An FHRP won because it needs no host changes.

## Common Mistakes

- **Thinking the hosts run the protocol.** They do not. Only the routers do. Hosts see one normal IP and one normal MAC.
- **Pointing hosts at a router's real IP.** If DHCP gives out 192.168.10.2 instead of the VIP, hosts lose their gateway when R1 dies, even with HSRP fully working.
- **Forgetting preempt in HSRP.** The higher priority router does not take the role back. Tracking also fails, because the other router cannot take over.
- **Assuming VRRP needs a preempt command.** It is already on by default. You turn it off if you want it off.
- **Decrementing too little.** The default track decrement is 10. With 110 versus 100, the priority falls only to a tie at 100. The result then depends on the IP tie-break and not on your design.
- **Tracking only the interface.** The line protocol can stay up while the ISP is down. Use IP SLA for that case.
- **Saying VRRP uses a UDP port.** It uses IP protocol 112 and has no port.
- **Mismatching versions or groups.** HSRP v1 and v2 cannot talk. Each router sees no peer, and both become active.
- **Thinking HSRP or VRRP doubles bandwidth.** One group means one forwarding router. Load sharing needs several groups, or GLBP.
- **Thinking GLBP balances packets.** It balances hosts at ARP time. It never looks at traffic volume.
- **Expecting an FHRP to fix inbound traffic.** It only controls the path out of the LAN.
- **Preempting with no delay.** A router that has just booted may take traffic before its routes are ready.
- **Using a VIP as a management address.** It moves between boxes, so it does not tell you which box you reached. Use real IPs for device management.
- **Leaving the gateway and the STP root on different switches.** Traffic takes an extra path for no reason.

## Interview Questions

**Q: What problem do first hop redundancy protocols solve?**

A: A host has one default gateway and no routing protocol, so if that router fails the host is cut off, even if another router is on the LAN. An FHRP lets routers share a virtual IP and virtual MAC, so a backup router can take over and the host changes nothing.

**Q: What is the difference between HSRP and VRRP?**

A: HSRP is Cisco proprietary and VRRP is an open standard. HSRP uses UDP 1985 and VRRP uses IP protocol 112. HSRP calls its roles active and standby, and VRRP calls them master and backup. VRRP preempts by default, and HSRP does not. VRRP can also use a router's real IP as the virtual IP.

**Q: List the HSRP states.**

A: Initial, Learn, Listen, Speak, Standby and Active. Only routers in Speak, Standby and Active send hellos. Only the active router forwards traffic for the virtual IP.

**Q: What are the default HSRP timers, and what do they mean for failover?**

A: Hello is 3 seconds and hold is 10 seconds. If the active router dies suddenly, the standby waits up to 10 seconds before taking over. I can lower that with `standby timers`, keeping hold at about three times hello, or use BFD where it is supported.

**Q: How does the host keep working after failover if its ARP cache still has the old entry?**

A: The ARP entry stays valid, because it holds the virtual MAC, not a router's real MAC. The new active router takes on that MAC and sends a gratuitous ARP. The switch moves the MAC to the new port, so the host's frames reach the new router.

**Q: What is preempt, and why would you add a delay?**

A: Preempt lets a higher priority router take the active role from a working lower priority router. It is off by default in HSRP. I add a delay because a router that has just booted may not have its routes ready, and it would black hole traffic if it took over at once.

**Q: R1 is the active HSRP router and its WAN link fails. What happens with no tracking?**

A: Nothing changes on the LAN side. R1 keeps sending hellos, so it stays active, and hosts keep sending traffic to a router with no way out. Object tracking fixes this by lowering R1's priority, and R2 must have preempt to take over.

**Q: You configured tracking but failover did not happen. What do you check?**

A: First, whether the other router has preempt, because HSRP has it off by default. Second, whether the decrement pushes priority below the peer. The default decrement of 10 only gives a tie on 110 versus 100. Third, whether the tracked object really went down, with `show track`. An interface can stay up while the path is dead, so IP SLA may be needed.

**Q: How does GLBP load balance when all hosts have the same gateway IP?**

A: The AVG answers every ARP request for the virtual IP, and it gives different hosts different virtual MACs. Each MAC belongs to an AVF, and a group can have up to 4 AVFs. By default it uses round-robin. So hosts share one gateway IP but use different routers. It balances per host, not per packet.

**Q: What happens in GLBP when an AVF fails?**

A: After the hold time, the AVG gives the failed router's virtual MAC to another AVF, which then forwards for both MACs. The redirect timer, 600 seconds by default, controls how long the AVG keeps handing out the old MAC. The timeout, 4 hours by default, removes the old MAC.

**Q: Why should the HSRP active router match the spanning tree root?**

A: If they differ, traffic from access switches goes up the unblocked uplink to the root, then crosses to the active router over an extra link. Aligning them gives a direct path. In a two-switch design, I usually split VLANs, with each switch active and root for different VLANs.

**Q: Does an FHRP give you full redundancy for a site?**

A: No. It covers only the first hop out of the LAN. Return traffic follows routing, so it can come back through the other router, which can break stateful firewalls or NAT. The hosts' switch can also still be a single point of failure, so routing, uplinks and state sync must be designed too.

## Quick Recap

- A host has one gateway and no routing protocol, so one router is a single point of failure. An FHRP shares a virtual IP and virtual MAC across routers.
- Failover works because the host keeps the same IP and MAC. A gratuitous ARP moves the MAC on the switch.
- HSRP: Cisco, UDP 1985, active and standby, hello 3 and hold 10, priority 100, preempt off by default.
- VRRP: open standard, IP protocol 112, 224.0.0.18, master and backup, 1 second advertisements, preempt on by default, VIP can be a real IP.
- GLBP: Cisco, UDP 3222, one AVG hands out virtual MACs to up to 4 AVFs, so one VIP uses several routers.
- Object tracking lowers priority when an uplink or remote path fails. It needs a big enough decrement and preempt on the peer.
- An FHRP protects only the outbound first hop. Stacks, vPC and anycast gateways can replace or change its role.
