# VLANs

**Level:** Intermediate | **Prerequisites:** [How Switches Work](03-how-switches-work.md), [Collision and Broadcast Domains](04-collision-and-broadcast-domains.md), [ARP](05-arp.md)

## In One Line

A VLAN (Virtual LAN) splits one physical switch into several separate Layer 2 networks, so ports in different VLANs behave as if they were on different switches.

## Why It Matters

A switch sends a broadcast out of every port. A broadcast is a frame addressed to everyone, such as an ARP request. A frame is the unit of data at Layer 2.

Take a company with 300 PCs on one flat network. Three problems appear:

- **Noise.** Every broadcast reaches all 300 PCs, and each PC must stop and look at it.
- **No separation.** The finance PCs and the guest laptops share one Layer 2 network. They can reach each other directly, with no firewall or filter in between.
- **Wasted hardware.** Without VLANs, separating teams means buying one switch per team, and rewiring when someone changes team.

VLANs fix all three in software. Moving a user to another team becomes one line of config, not a new cable.

Interviewers ask about VLANs because nearly every wired enterprise network uses them. VLANs also link many other topics: broadcast domains, subnets, trunks, routing, voice, and security. A candidate who explains VLANs well usually understands Layer 2.

## Simple Explanation

Think of a large open hall where everyone can hear everyone. One person shouts, and 300 people look up. This is a flat network.

Now build soundproof walls and make three rooms: Sales, Engineering, and Guests. The hall is the same size, and the floor is the same. But a shout in Sales stays in Sales. Each room is a VLAN.

To talk to another room, you must go through a door with a guard. The door is a router or firewall. The guard can check who passes.

People in the rooms do not need to know the room number. The building manager keeps the list: "desk 5 is in room Sales". In the same way, the switch keeps the list "port 5 is in VLAN 10", and the PC knows nothing about it.

**Where the analogy stops being true:** real walls are hard to move and hard to break. VLAN walls are only entries in a config file. One wrong line puts a port in the wrong room. A VLAN is a strong way to organise traffic, but it is not as strong as a real physical gap.

```text
  One physical switch, three logical switches

  +--------------------------------------------------+
  | SW1                                              |
  |  +--------------+ +--------------+ +-----------+ |
  |  | VLAN 10      | | VLAN 20      | | VLAN 40   | |
  |  | SALES        | | ENGINEERING  | | GUESTS    | |
  |  | port 1, 2    | | port 3, 4    | | port 7, 8 | |
  |  +--------------+ +--------------+ +-----------+ |
  +--------------------------------------------------+

  A broadcast from port 1 reaches port 2 only.
  Ports 3, 4, 7 and 8 never see it.
```

## How It Works

### Key words

- **Port:** a physical socket on the switch.
- **VLAN ID:** a number that names a VLAN, for example 10.
- **Broadcast domain:** the group of devices that receive each other's broadcasts. One VLAN is one broadcast domain.
- **Flood:** the switch sends a frame out of many ports because it has no single destination port.
- **Access port:** a switch port that belongs to one VLAN and connects to an end device, such as a PC or printer.
- **Trunk port:** a port that carries many VLANs between switches. Details are in [Trunking and 802.1Q](07-trunking-8021q.md).
- **Tag:** a small field added to a frame that says which VLAN the frame belongs to.

### What the switch does with a frame

The switch keeps one extra fact for each port: its VLAN. Here is the path of a frame through a switch that uses VLANs.

1. An admin creates the VLAN and assigns ports to it.
2. A PC sends a normal Ethernet frame. The frame has no VLAN information.
3. The frame enters an access port. The switch marks it internally with that port's VLAN.
4. The switch learns the source MAC address. It stores the VLAN, the MAC, and the port together.
5. The switch looks up the destination MAC, but only inside that VLAN.
6. If it finds the MAC, it sends the frame out of that one port.
7. If the frame is a broadcast, or the MAC is unknown, the switch floods it. It floods only to other ports in the same VLAN.
8. An access port sends the frame out with no VLAN information. A trunk port normally adds a tag first.

So the MAC address table works as if each VLAN had its own table. The same MAC can even appear in two VLANs as two separate entries.

```text
  Normal switch lookup:     MAC           -> port
  Switch with VLANs:        VLAN + MAC    -> port
```

The key rule is simple: **a switch never forwards a frame from one VLAN into another VLAN.** To cross VLANs, traffic must go up to Layer 3 through a router or a Layer 3 switch. See [Inter-VLAN Routing](08-inter-vlan-routing.md).

### Access ports

An access port has these properties:

- It belongs to exactly one data VLAN, called the access VLAN.
- It sends and expects untagged frames. "Untagged" means a normal frame with no VLAN tag.
- The device on the port does not know its VLAN. It sees a normal Ethernet network.

The access VLAN is a setting on the switch, not on the PC. Plug the same PC into a port in a different VLAN and it joins that VLAN. It then usually needs a new IP address.

**Fuller truth:** the voice VLAN feature, covered below, lets an access port carry a second VLAN. Also, Cisco documents that a tagged frame arriving on an access port is dropped. The exact handling of a frame tagged with the port's own VLAN can vary by platform.

**Other vendors use other words.** "Access port" is mostly Cisco language. The 802.1Q standard talks about a PVID (Port VLAN ID), which is the VLAN given to untagged frames that enter a port. HPE Aruba switches describe the same idea as an "untagged" port in that VLAN.

### VLAN IDs and ranges

A VLAN ID lives in a 12-bit field in the 802.1Q tag. IEEE 802.1Q is the standard that defines VLAN tagging.

Twelve bits give 2 to the power 12, which is 4096 values: 0 to 4095. Two values are reserved, so **4094 usable VLANs: 1 to 4094.**

| VLAN ID | Who defines it | Meaning |
|---|---|---|
| 0 | IEEE 802.1Q | Reserved. Means "no VLAN, this tag only carries priority". |
| 1 | Cisco default (and most vendors) | Default VLAN. On Cisco, you cannot delete or rename it. |
| 2-1001 | Cisco normal range | Free for normal use. |
| 1002-1005 | Cisco reserved | Legacy FDDI and Token Ring VLANs. You cannot use, change, or delete them. |
| 1006-4094 | Cisco extended range | Usable, with some limits on older setups. |
| 4095 | IEEE 802.1Q | Reserved. |

Only 0 and 4095 come from the standard. The 1002-1005 block is a Cisco rule. On a non-Cisco switch you can often use those numbers.

Why do 1002-1005 still exist? FDDI and Token Ring were older LAN types. Cisco kept VLANs for them and never removed them. They still appear in `show vlan brief`.

Each 1002-1005 VLAN has a fixed name:

| VLAN | Name |
|---|---|
| 1002 | fddi-default |
| 1003 | token-ring-default |
| 1004 | fddinet-default |
| 1005 | trnet-default |

**Real switches can have lower limits.** The standard allows 4094 VLANs, but a small switch may support only a few hundred active at the same time. Some platforms also reserve a block for internal use. Many Cisco Nexus models, for example, reserve 3968-4095 by default. Check the guide for your model.

### Types of VLAN

"Type" here describes the job of a VLAN, not a different technology. They are all VLANs.

| Name | Job | Example |
|---|---|---|
| Data VLAN | Carries user traffic | VLAN 10 SALES |
| Voice VLAN | Carries IP phone traffic | VLAN 30 VOICE |
| Management VLAN | Used to reach the switches themselves by SSH or SNMP | VLAN 99 |
| Default VLAN | Where all ports start | VLAN 1 |
| Native VLAN | Carries untagged frames on a trunk | VLAN 1 by default |
| Parking VLAN | Unused ports go here (also called a black hole VLAN) | VLAN 999 |

### Default VLAN

The default VLAN is the VLAN that every port belongs to on a new switch. On Cisco, this is VLAN 1. A new switch works out of the box because all ports share VLAN 1. No setup is needed.

Facts about Cisco VLAN 1:

- It cannot be deleted or renamed. Trying `no vlan 1` gives the error `%Default VLAN 1 may not be deleted.`
- Control protocols such as CDP and VTP use VLAN 1 on trunks by default. CDP (Cisco Discovery Protocol) lets neighbouring Cisco devices identify each other.
- The first management interface on the switch is `interface vlan 1`.

Good practice is to keep user traffic out of VLAN 1. An attacker on a port left in VLAN 1 lands in the same VLAN as the default management interface. Using VLAN 1 for users also means one mistake exposes everything that was never configured.

### Native VLAN (intro)

The native VLAN is a trunk idea. A trunk tags its frames so the other end knows the VLAN. One VLAN is the exception: its frames cross the trunk with no tag. This is the native VLAN.

- On Cisco, the native VLAN is VLAN 1 by default.
- Both ends of a trunk must use the same native VLAN, or traffic ends up in the wrong VLAN.
- An access port has an access VLAN, not a native VLAN.

Default VLAN and native VLAN are both VLAN 1 on a new Cisco switch, so people mix them up. They are different ideas. Default means "where ports start". Native means "which VLAN is untagged on a trunk". Change one and the other stays the same.

| Term | Applies to | Cisco default |
|---|---|---|
| Default VLAN | Whole switch: where unconfigured ports sit | 1, fixed |
| Access VLAN | One access port | 1, changeable |
| Native VLAN | One trunk port | 1, changeable |

Full details are in [Trunking and 802.1Q](07-trunking-8021q.md).

### Voice VLAN

Many offices connect a desk like this: wall socket to IP phone, IP phone to PC. One cable serves two devices. An IP phone sends voice over the network, and it contains a small built-in switch for this purpose.

The problem is that the phone and PC should be in different VLANs, but they share one switch port. The voice VLAN solves it.

```text
  +------+  untagged   +-------+  PC: untagged (VLAN 10)  +--------+
  |  PC  |------------>| Phone |------------------------->|  SW1   |
  +------+             +-------+  Voice: tagged VLAN 30   | Gi1/0/5|
                                                          +--------+
```

How it works:

1. You set a data VLAN and a voice VLAN on the same access port.
2. The switch tells the phone the voice VLAN ID. Cisco uses CDP. Other vendors commonly use LLDP-MED, an extension of LLDP, the open standard neighbour discovery protocol.
3. The phone tags its own voice frames with that VLAN.
4. The PC sends untagged frames, and the phone passes them along unchanged.
5. The switch puts tagged frames in the voice VLAN and untagged frames in the data VLAN.
6. If it does not hear about a voice VLAN, the phone may stay in the data VLAN. So a phone with no VLAN info often gets the wrong IP address.

Why separate voice from data?

- **Quality.** Voice breaks up with delay and loss. A separate VLAN makes it easy to apply QoS (Quality of Service), which means giving some traffic priority. Phones commonly mark voice as CoS 5 at Layer 2 and DSCP EF at Layer 3. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).
- **Security.** Users on PCs do not sit in the same Layer 2 network as the phones, so sniffing or attacking calls is harder.
- **Addressing.** Phones get their own subnet and their own DHCP scope. See [DHCP](../07-application-protocols/04-dhcp.md).
- **Less noise.** PC broadcasts do not reach the phones.

"Voice VLAN" is a vendor feature, not a term from 802.1Q. The standard only provides the tag. The port stays an access port in config, even though it carries two VLANs. Some people call it a "multi-VLAN access port".

### Benefits: broadcast control

Each VLAN is its own broadcast domain. Fewer devices share each broadcast.

Assume 300 PCs, each sending one broadcast per minute:

| Design | Broadcasts seen by one PC, per minute |
|---|---|
| One flat VLAN of 300 | 299 |
| Three VLANs of 100 each | 99 |

The same limit applies to flooded frames with an unknown destination. Flooding also stays inside the VLAN, so a problem in one VLAN, such as a broadcast storm, is contained. Each VLAN has its own failure area. A Layer 2 loop in VLAN 10 is still bad, because shared links and switch CPU can suffer, but devices in VLAN 20 are usually not flooded by it.

There is no standard number for "the right size". Many designs use a /24 subnet per VLAN, which allows 254 hosts.

### Benefits: security

- Traffic between VLANs must go through a Layer 3 device. There, an [ACL](../09-security/05-access-control-lists.md) or [firewall](../09-security/04-firewalls.md) can allow or block it.
- A guest in VLAN 40 cannot ARP for a finance server in VLAN 50. The ARP request never reaches it.
- A sniffer on a port sees only traffic for its own VLAN. A sniffer is a tool that captures frames.

**What a VLAN does not do:** a VLAN does not encrypt anything, and it does not filter inside itself. Two PCs in the same VLAN can still attack each other. VLANs can also be broken by mistakes or by VLAN hopping attacks. VLAN hopping means an attacker gets frames into a VLAN they should not reach. See [Layer 2 Security](11-layer2-security.md). Modern designs go further with [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### VLANs and subnets

The common design rule is **one VLAN, one IP subnet.** VLAN 10 uses 10.1.10.0/24, and VLAN 20 uses 10.1.20.0/24.

A VLAN is Layer 2. A subnet is Layer 3. The rule keeps the two matched, so hosts know that an address outside their subnet needs the gateway, and hosts inside are reachable by ARP.

Matching the VLAN number to the third octet of the subnet, as above, is only a habit, not a rule. But it makes troubleshooting easier.

### VLANs across many switches

Sales users may sit on two floors with two switches. VLAN 10 can stretch across both. The switches connect with a trunk, and each frame carries a 4-byte 802.1Q tag that says "this is VLAN 10".

```text
  SW1                     trunk                     SW2
  +---------------+   tagged frames    +---------------+
  | Gi1/0/1  V10  |====================| Gi1/0/1  V10  |
  | Gi1/0/3  V20  |  VLAN 10, 20, 30   | Gi1/0/3  V20  |
  +---------------+                    +---------------+
```

Tag layout and trunk setup are in [Trunking and 802.1Q](07-trunking-8021q.md).

## Example

### The network

SW1 is a 12-port Cisco Catalyst switch running IOS XE.

```text
                     to distribution switch DS1
                               |
                         trunk | Gi1/0/12
                     +---------+---------+
                     |        SW1        |
                     |  mgmt 10.1.99.2   |
                     +-+---+---+---+---+-+
                       |   |   |   |   |
                     Gi1/0/1 to Gi1/0/5, left to right
                       |   |   |   |   |
                     PC1 PC2 PC3 PC4 Phone --- PC5
                     V10 V10 V20 V20 V30       V10
```

| Device | Port | VLAN | IP address |
|---|---|---|---|
| PC1 | Gi1/0/1 | 10 SALES | 10.1.10.11/24 |
| PC2 | Gi1/0/2 | 10 SALES | 10.1.10.12/24 |
| PC3 | Gi1/0/3 | 20 ENGINEERING | 10.1.20.13/24 |
| PC4 | Gi1/0/4 | 20 ENGINEERING | 10.1.20.14/24 |
| Phone | Gi1/0/5 | 30 VOICE | 10.1.30.15/24 |
| PC5 | Gi1/0/5 through the phone | 10 SALES | 10.1.10.15/24 |
| Unused | Gi1/0/6-11 | 999 PARKED | none |
| SW1 management | interface vlan 99 | 99 MANAGEMENT | 10.1.99.2/24 |

The default gateway for management is 10.1.99.1 on DS1.

### Step 1: create and name the VLANs

Cisco IOS:

```text
SW1# configure terminal
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config-vlan)# vlan 20
SW1(config-vlan)# name ENGINEERING
SW1(config-vlan)# vlan 30
SW1(config-vlan)# name VOICE
SW1(config-vlan)# vlan 99
SW1(config-vlan)# name MANAGEMENT
SW1(config-vlan)# vlan 999
SW1(config-vlan)# name PARKED
SW1(config-vlan)# exit
```

### Step 2: configure the access ports

Cisco IOS:

```text
SW1(config)# interface range gigabitEthernet 1/0/1 - 2
SW1(config-if-range)# description Sales PC
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# exit

SW1(config)# interface range gigabitEthernet 1/0/3 - 4
SW1(config-if-range)# description Engineering PC
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 20
SW1(config-if-range)# exit
```

`switchport mode access` makes the port an access port for good. `switchport access vlan 10` chooses the VLAN. You need both.

### Step 3: configure the phone port

Cisco IOS:

```text
SW1(config)# interface gigabitEthernet 1/0/5
SW1(config-if)# description Desk 5 phone and PC
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport voice vlan 30
SW1(config-if)# exit
```

### Step 4: park the unused ports

Cisco IOS:

```text
SW1(config)# interface range gigabitEthernet 1/0/6 - 11
SW1(config-if-range)# description UNUSED
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 999
SW1(config-if-range)# shutdown
SW1(config-if-range)# exit
```

### Step 5: add the uplink and the management address

An SVI (Switched Virtual Interface) is a virtual interface for a VLAN. It gives the switch itself an IP address in that VLAN. A plain Layer 2 switch uses one SVI for management.

An SVI comes up only when the VLAN exists and at least one active port carries that VLAN. Here, that port is the trunk.

Cisco IOS:

```text
SW1(config)# interface gigabitEthernet 1/0/12
SW1(config-if)# description Uplink to DS1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99
SW1(config-if)# exit

SW1(config)# interface vlan 99
SW1(config-if)# ip address 10.1.99.2 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit
SW1(config)# ip default-gateway 10.1.99.1
SW1(config)# end
SW1# copy running-config startup-config
```

Older switches may need `switchport trunk encapsulation dot1q` before `switchport mode trunk`.

### Step 6: verify

**See all VLANs and their ports:**

```text
SW1#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   SALES                            active    Gi1/0/1, Gi1/0/2, Gi1/0/5
20   ENGINEERING                      active    Gi1/0/3, Gi1/0/4
30   VOICE                            active    Gi1/0/5
99   MANAGEMENT                       active
999  PARKED                           active    Gi1/0/6, Gi1/0/7, Gi1/0/8
                                                Gi1/0/9, Gi1/0/10, Gi1/0/11
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

Read this output closely:

- Gi1/0/5 appears twice, once as data in VLAN 10 and once as voice in VLAN 30.
- Gi1/0/12 does not appear at all. `show vlan brief` does not list trunk ports. Use `show interfaces trunk` for them.
- VLAN 99 shows no ports, but it is active. It is carried on the trunk.

**Check one port in detail:**

```text
SW1#show interfaces gigabitEthernet 1/0/5 switchport
Name: Gi1/0/5
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: native
Negotiation of Trunking: Off
Access Mode VLAN: 10 (SALES)
Trunking Native Mode VLAN: 1 (default)
Administrative Native VLAN tagging: enabled
Voice VLAN: 30 (VOICE)
(output trimmed)
```

**Check status of all ports:**

```text
SW1#show interfaces status

Port         Name               Status       Vlan       Duplex  Speed
Gi1/0/1      Sales PC           connected    10         a-full a-1000
Gi1/0/2      Sales PC           connected    10         a-full a-1000
Gi1/0/3      Engineering PC     connected    20         a-full a-1000
Gi1/0/4      Engineering PC     connected    20         a-full a-1000
Gi1/0/5      Desk 5 phone and   connected    10         a-full a-1000
Gi1/0/6      UNUSED             disabled     999          auto   auto
(ports 7 to 11 trimmed, same as port 6)
Gi1/0/12     Uplink to DS1      connected    trunk      a-full a-1000
```

**See which MAC addresses live in which VLAN:**

```text
SW1#show mac address-table interface gigabitEthernet 1/0/5
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    0050.7966.6805    DYNAMIC     Gi1/0/5
  30    0050.7966.6830    DYNAMIC     Gi1/0/5
Total Mac Addresses for this criterion: 2
```

One port, two VLANs. The first line is PC5, and the second is the phone. The phone's MAC can also appear in VLAN 10, because a phone sends some untagged frames, such as CDP. This matters when you set port security limits.

### Step 7: test from the PCs

From PC1 (Linux), to PC2 in the same VLAN:

```bash
$ ping -c 2 10.1.10.12
PING 10.1.10.12 (10.1.10.12) 56(84) bytes of data.
64 bytes from 10.1.10.12: icmp_seq=1 ttl=64 time=0.612 ms
64 bytes from 10.1.10.12: icmp_seq=2 ttl=64 time=0.438 ms

--- 10.1.10.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.438/0.525/0.612/0.087 ms
```

From PC1, to PC3 in VLAN 20, with no gateway set on PC1:

```bash
$ ping -c 2 10.1.20.13
ping: connect: Network is unreachable
```

Windows gives a similar failure, usually `PING: transmit failed. General failure.`

This failure is the expected result. VLAN 20 is a different network. Only a router or Layer 3 switch can connect them.

### Trace: an ARP broadcast from PC1

PC1 wants to reach PC2, so it sends an ARP request to `ff:ff:ff:ff:ff:ff`. Here is what SW1 does.

| Port | VLAN | Does it get the frame? | Why |
|---|---|---|---|
| Gi1/0/1 | 10 | No | This is the source port. |
| Gi1/0/2 | 10 | Yes, untagged | Same VLAN. PC2 replies. |
| Gi1/0/3-4 | 20 | No | Different VLAN. |
| Gi1/0/5 | 10 | Yes, untagged | PC5 is in VLAN 10. |
| Gi1/0/6-11 | 999 | No | Different VLAN, and shut down. |
| Gi1/0/12 | trunk | Yes, tagged VLAN 10 | VLAN 10 is allowed on the trunk. |

PC3 and PC4 never see the broadcast.

## Going Deeper

### Where VLANs are stored on Cisco

This surprises many people.

- Normal-range VLANs are stored in a file in flash named `vlan.dat`, not in the startup config.
- They are saved to that file as you make them. You do not need `copy running-config startup-config` for them.
- But port settings, such as `switchport access vlan 10`, live in the running config. You must save those.
- `erase startup-config` followed by a reload does **not** remove normal-range VLANs. To clear them too, also run `delete flash:vlan.dat`.

In VTP transparent mode, VLANs also appear in the running config. Extended-range VLANs are normally kept in the running config with VTP version 1 or 2. Details vary by VTP version, so check your platform.

### VTP in brief

VTP (VLAN Trunking Protocol) is a Cisco protocol that shares the VLAN list across trunks. You create a VLAN on one switch, and others learn it.

- **Server mode:** can create VLANs and shares them.
- **Client mode:** cannot create VLANs, only learns them. Trying gives an error: `VTP VLAN configuration not allowed when device is in CLIENT mode.`
- **Transparent mode:** keeps its own VLAN list and passes VTP messages along.

VTP versions 1 and 2 handle only the normal range. Version 3 also handles the extended range.

The danger: a switch with a higher VTP revision number can overwrite the VLAN list of the whole domain. Adding an old lab switch has wiped out VLANs in real networks. For this reason, many engineers run VTP in transparent mode or turn it off, and manage VLANs with automation. Default modes vary by platform, so check `show vtp status`.

### Dynamic VLAN assignment

So far, each port's VLAN has been fixed by config. This is called a **static VLAN**. A VLAN can also be chosen when a user connects.

With 802.1X, a user or device logs in, and a RADIUS server replies with the VLAN to use. A finance laptop lands in the finance VLAN on any desk. See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md). Cisco also had an older system named VMPS, which is mostly history now.

### Priority tags: VLAN 0

A frame can carry an 802.1Q tag with VLAN ID 0. The tag then carries only the priority bits. The switch treats the frame as part of the port's own VLAN. IOS supports this for phones with `switchport voice vlan dot1p`. The command also accepts `untagged` and `none`.

### VLANs on Linux and Windows hosts

A normal PC on an access port needs no VLAN setup. But a server connected to a **trunk** port can join VLANs itself, by adding the tag on its own.

Linux, joining VLAN 10:

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 10.1.10.50/24 dev eth0.10
sudo ip link set eth0.10 up
ip -d link show eth0.10
```

```text
4: eth0.10@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vlan protocol 802.1Q id 10 <REORDER_HDR>
```

Windows, on a NIC whose driver supports VLAN IDs:

```powershell
Set-NetAdapter -Name "Ethernet" -VlanID 10
Get-NetAdapter -Name "Ethernet" | Format-List Name, VlanID
```

Hypervisors do the same for virtual machines.

```powershell
# Windows Hyper-V: put a VM's virtual NIC in VLAN 10
Set-VMNetworkAdapterVlan -VMName web01 -Access -VlanId 10
```

### Private VLANs

Normal VLANs do not isolate hosts from each other inside the VLAN. A private VLAN does. It splits one VLAN into sub-VLANs.

- **Promiscuous port:** can talk to everyone. Usually the gateway.
- **Isolated port:** can talk only to the promiscuous port.
- **Community port:** can talk to its own community and the promiscuous port.

Hotels and hosting providers use this. All guests share one subnet, but they cannot reach each other.

### VLAN and spanning tree

Cisco's default STP modes run one spanning tree per VLAN, so each VLAN you add creates one more STP instance for the switch to run. See [Spanning Tree Protocol](09-spanning-tree.md).

### When 4094 is not enough

Large data centres and cloud providers need more than 4094 segments. Two common answers:

- **802.1ad (QinQ):** a second tag on top of the first. Often used by service providers.
- **VXLAN:** uses a 24-bit ID, which gives over 16 million segments. See [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

### Troubleshooting two PCs that cannot talk

| Check | Command (Cisco IOS) | What you are looking for |
|---|---|---|
| Is the port up? | `show interfaces status` | `connected`, not `notconnect` or `disabled` |
| Right VLAN? | `show interfaces status` | The VLAN column matches the plan |
| Does the VLAN exist? | `show vlan brief` | The VLAN is listed and `active` |
| Access or trunk? | `show interfaces <port> switchport` | `Operational Mode: static access` |
| Is the VLAN allowed across? | `show interfaces trunk` | The VLAN is forwarded on the trunk |
| Was the MAC learned? | `show mac address-table vlan 10` | Both PCs appear in the VLAN |
| Same subnet? | `ip addr` (Linux), `ipconfig` (Windows) | Addresses and masks match the VLAN plan |

## Common Mistakes

- **Thinking a VLAN is a subnet.** A VLAN is Layer 2 and a subnet is Layer 3. They are usually matched one to one, but they are separate ideas.
- **Thinking a switch can route between VLANs on its own.** A pure Layer 2 switch cannot. You need a router or a Layer 3 switch with routing enabled.
- **Leaving out `switchport mode access`.** Setting only `switchport access vlan 10` may leave the port in a dynamic mode. Many Cisco models default to dynamic auto, so the port can become a trunk if the other side asks. DTP (Dynamic Trunking Protocol) does this negotiation, and it is an attack path.
- **Confusing default VLAN and native VLAN.** Both are VLAN 1 on a new Cisco switch, but they are different things.
- **Thinking a VLAN is full security.** A VLAN separates traffic, but it does not filter or encrypt. Add ACLs or firewalls between VLANs.
- **Assuming that `show vlan brief` lists every port.** It does not list operational trunk ports.
- **Deleting a VLAN and expecting ports to return to VLAN 1.** They do not. Ports stay assigned to the deleted VLAN and become inactive. Traffic stops. `show interfaces <port> switchport` shows `Access Mode VLAN: 10 (Inactive)`.
- **Assigning a port to a VLAN that was never created.** Most Catalyst switches create it for you with the message `% Access VLAN does not exist. Creating vlan 50`. In VTP client mode, that does not work, and the port goes inactive.
- **Forgetting that the same VLAN must exist on every switch in the path.** The trunk must allow the VLAN, and each switch must know it.
- **Trying to use VLAN 1002.** It is reserved on Cisco, along with 1003-1005.
- **Saying the VLAN range is 0-4095.** The field holds 4096 values, but only 1-4094 are usable.
- **Leaving users and unused ports in VLAN 1.** Move users to their own VLANs. Shut down unused ports and park them in an unused VLAN.
- **Believing `erase startup-config` clears VLANs.** Normal-range VLANs live in `vlan.dat`.
- **Saying a host is "in the same VLAN" because the IP looks right.** IP address does not decide the VLAN. On an access port, the switch port decides.

## Interview Questions

**Q: What is a VLAN and why would you use one?**

A: A VLAN is a logical Layer 2 network on a switch. Each VLAN is its own broadcast domain, even when VLANs share the same hardware. I would use VLANs to reduce broadcast traffic, separate teams or device types for security, and move users between groups by config instead of rewiring.

**Q: Can two hosts in different VLANs talk to each other?**

A: Not directly at Layer 2, because a switch does not forward frames between VLANs. They need a Layer 3 device, such as a router or a Layer 3 switch. That device is also a good place to apply ACLs or a firewall.

**Q: What is an access port?**

A: An access port belongs to one data VLAN and connects to an end device such as a PC. It sends and receives untagged frames, so the device does not know about the VLAN. The switch adds the VLAN information internally. A voice VLAN can be added to the same port for an IP phone.

**Q: How many VLANs can you have, and why?**

A: 802.1Q uses a 12-bit VLAN ID, which gives 4096 values. VLAN 0 and 4095 are reserved, so 4094 are usable. On Cisco, 1002-1005 are also reserved for legacy FDDI and Token Ring. A real switch may support fewer active VLANs than that.

**Q: What is the difference between the default VLAN and the native VLAN?**

A: The default VLAN is where all ports start on a new switch. On Cisco, that is VLAN 1, and it cannot be deleted. The native VLAN is a trunk setting: it is the VLAN whose frames cross the trunk untagged. It is VLAN 1 by default, but you can change it per trunk. Both ends must match.

**Q: What are normal-range and extended-range VLANs?**

A: On Cisco, normal range is 1-1005 and extended range is 1006-4094. Normal-range VLANs are stored in `vlan.dat` in flash. With VTP version 1 or 2, extended VLANs are usually kept in the running config and are not shared by VTP. Version 3 can share them.

**Q: What is a voice VLAN, and is a port with one a trunk?**

A: A voice VLAN lets one access port carry an IP phone and a PC. The phone tags voice frames with the voice VLAN, and the PC traffic stays untagged in the data VLAN. Voice stays separate, which helps with QoS and security. The port is still configured as an access port, though it acts a bit like a small two-VLAN trunk.

**Q: Why should you not use VLAN 1 for users?**

A: VLAN 1 is the default for every unconfigured port, and control protocols also use it. If users share it, one forgotten port drops a person straight into that network. So I put users in their own VLANs, use a separate management VLAN, and shut down unused ports in a parking VLAN.

**Q: You delete VLAN 20 from a switch. What happens to its ports?**

A: They stay assigned to VLAN 20, but they become inactive and stop passing traffic. They do not move to VLAN 1. The fix is to create VLAN 20 again or assign the ports to another VLAN. I would check with `show interfaces switchport`.

**Q: Two PCs have IPs in the same subnet but are in different VLANs. Can they ping?**

A: No, not through a normal switch. The ARP request stays in the sender's VLAN and never reaches the other PC. So the ping fails with no ARP reply, even though the subnets match. This shows that the VLAN, and not the IP address, decides who is reachable at Layer 2.

**Q: Which commands do you use to check VLANs on a Cisco switch?**

A: `show vlan brief` lists VLANs and access ports. `show interfaces status` shows each port's state and VLAN. `show interfaces <port> switchport` shows access VLAN, voice VLAN, and mode. `show interfaces trunk` is for trunks, and `show mac address-table vlan <id>` shows what was learned in a VLAN.

**Q: Do VLANs make a network secure?**

A: They help, but they are not enough alone. VLANs limit who shares Layer 2, and they force traffic through a point where you can filter it. But they do not encrypt, and they do not filter inside a VLAN. They can also be bypassed by VLAN hopping if the switch is badly configured. So I combine them with ACLs, firewalls, and Layer 2 hardening.

## Quick Recap

- A VLAN is a logical Layer 2 network. One VLAN equals one broadcast domain, and usually one subnet.
- An access port belongs to one data VLAN, sends untagged frames, and hides the VLAN from the host.
- Usable VLANs are 1-4094 (12-bit field, 0 and 4095 reserved). On Cisco, 1 is the default and 1002-1005 are reserved.
- A voice VLAN lets a phone and PC share one port: voice is tagged, and PC data is untagged.
- Default VLAN means where ports start. Native VLAN means which VLAN is untagged on a trunk.
- Configure with `vlan`, `switchport mode access`, and `switchport access vlan`, then verify with `show vlan brief` and `show interfaces switchport`.
- VLANs reduce broadcasts and support security, but crossing VLANs needs Layer 3, and real protection needs filtering there.
