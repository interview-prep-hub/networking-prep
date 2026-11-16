# Trunking and 802.1Q

**Level:** Intermediate | **Prerequisites:** [VLANs](06-vlans.md), [Ethernet Frames and Standards](02-ethernet-frames.md), [How Switches Work](03-how-switches-work.md)

## In One Line

A trunk is one link that carries many VLANs. 802.1Q is the IEEE standard that adds a 4-byte tag to each frame, so the receiving device knows which VLAN the frame belongs to.

## Why It Matters

A VLAN (Virtual LAN) splits one physical network into several separate Layer 2 networks. That is easy on one switch. It becomes a problem when a VLAN must stretch across two switches.

Say SW1 and SW2 both have users in 20 VLANs. Without trunking you need 20 cables between them, one per VLAN. That wastes 20 ports on each switch. With trunking you need one cable.

But once frames from 20 VLANs share one cable, the receiving switch has a new problem. A normal Ethernet frame says nothing about its VLAN. Something must mark each frame. That mark is the 802.1Q tag.

Interviewers ask about this topic for three reasons:

- It tests whether you know what a frame looks like on the wire, not just what a VLAN is.
- Many real outages come from trunks: a VLAN missing from the allowed list, a native VLAN mismatch, or a used switch that wiped the VLAN database through VTP.
- It has real security issues: DTP and the native VLAN are the base of VLAN hopping attacks.

## Simple Explanation

Picture two office buildings joined by one corridor. Each building has the same departments: Sales, Finance and HR.

A mail cart goes along the corridor. Mail from all departments shares the same cart. At the corridor door, a worker puts a coloured sticker on each envelope: blue for Sales, green for Finance, red for HR. At the other end, a worker reads the sticker, removes it, and delivers the envelope to the right department in that building.

- The corridor is the **trunk**.
- The sticker is the **802.1Q tag**.
- Staff inside a department never see the sticker. Their mail arrives clean, like a normal frame on an **access port** (a switch port that belongs to one VLAN and carries no tags).

There is one more rule. Envelopes with no sticker are not thrown away. They go to one chosen department. That department is the **native VLAN**.

**Where the analogy stops being true:** a sticker sits on the outside of an envelope. A tag sits inside the frame, in the header, and the switch must recalculate the frame's checksum after adding it. Also, a tag gives no secrecy. Anyone who can capture traffic on the trunk can read every frame in every VLAN.

## How It Works

### Access Ports and Trunk Ports

A switch port usually works in one of two ways.

| Port type | Carries | Tags on the wire | Usual neighbour |
|---|---|---|---|
| Access port | One VLAN | None | PC, printer, camera |
| Trunk port | Many VLANs | Yes, except the native VLAN | Switch, router, AP, server host |

A normal PC sends and receives untagged frames. It does not know VLANs exist. The switch keeps track of the VLAN internally.

```text
  PC1                                                 PC2
VLAN 10                                             VLAN 10
   |                                                   |
   | untagged                                 untagged |
   |                                                   |
 Gi0/1                                               Gi0/1
+-------+   Gi0/24     tagged frames     Gi0/24   +-------+
|  SW1  |-----------------------------------------|  SW2  |
+-------+          VLAN 10, 20, 99 share          +-------+
   |               one cable                           |
 Gi0/2                                               Gi0/2
   |                                                   |
 VLAN 20                                             VLAN 20
```

### The 802.1Q Tag

802.1Q is an IEEE standard, so it works between vendors. It does not wrap the frame in a new header. It inserts 4 bytes inside the existing Ethernet header, right after the source MAC address.

An **EtherType** is a 2-byte field that says what comes next in the frame. For example, 0x0800 means IPv4 and 0x0806 means ARP. The **FCS** (Frame Check Sequence) is a 4-byte checksum at the end of the frame. It lets the receiver detect damage.

```text
Untagged frame (maximum 1518 bytes)

+---------+---------+-----------+--------------+-----+
| Dst MAC | Src MAC | EtherType | Payload      | FCS |
| 6 bytes | 6 bytes | 2 bytes   | up to 1500   | 4   |
+---------+---------+-----------+--------------+-----+

Tagged frame (maximum 1522 bytes)

+---------+---------+---------+-----------+------------+-----+
| Dst MAC | Src MAC | 802.1Q  | EtherType | Payload    | FCS |
| 6 bytes | 6 bytes | tag: 4  | 2 bytes   | up to 1500 | 4   |
+---------+---------+---------+-----------+------------+-----+
                         ^                                ^
                         |                                |
                  inserted here          recalculated, because
                                         the frame contents changed
```

The tag has four fields. The first 16 bits are the TPID. The next 16 bits are called the **TCI** (Tag Control Information), and they hold the other three fields.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-----+-+-----------------------+
|        TPID (16 bits)         | PCP |D|     VID (12 bits)     |
|            0x8100             | (3) |E|                       |
|                               |     |I|                       |
+-------------------------------+-----+-+-----------------------+
```

| Field | Bits | Full name | Purpose |
|---|---|---|---|
| TPID | 16 | Tag Protocol Identifier | Value 0x8100 says "this frame has an 802.1Q tag". It sits where the EtherType would normally be. |
| PCP | 3 | Priority Code Point | Priority from 0 to 7, used by Layer 2 QoS. Also known as 802.1p or CoS (Class of Service). |
| DEI | 1 | Drop Eligible Indicator | 1 means "drop this frame first if there is congestion". This bit was once called CFI (Canonical Format Indicator). |
| VID | 12 | VLAN Identifier | The VLAN number. |

**Why 4094 VLANs?** 12 bits give 4096 values (0 to 4095). Two are reserved.

| VID | Meaning |
|---|---|
| 0 | No VLAN. The frame carries only a priority value. Called a priority-tagged frame. |
| 1-4094 | Normal VLAN numbers. VLAN 1 is the default VLAN on most switches. |
| 4095 | Reserved by the standard. |

Vendors reserve more. For example, Cisco keeps VLANs 1002-1005 for old FDDI and Token Ring use, and some switches keep a few high numbers for internal use.

For how to read PCP in a real QoS design, see [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md). If binary and hex are new to you, see [Binary and Hexadecimal](../01-foundations/07-binary-and-hexadecimal.md).

### What a Switch Does With a Frame

**When a frame enters a trunk port:**

1. The switch checks if the 2 bytes after the source MAC equal 0x8100.
2. If yes, the frame is tagged. The switch reads the VID.
3. If no, the frame is untagged. The switch places it in the native VLAN.
4. If that VLAN is not allowed on this trunk, the switch drops the frame.
5. The switch learns the source MAC in its table for that VLAN.
6. The switch forwards the frame out ports in the same VLAN only.

**When a frame leaves a port:**

1. Access port: send it untagged.
2. Trunk port, VLAN is native: send it untagged (by default).
3. Trunk port, any other VLAN: add the tag with that VID.
4. If the tag was added or removed, recalculate the FCS.

If removing a tag leaves a frame shorter than the 64-byte Ethernet minimum, the switch adds padding first.

### Native VLAN

The **native VLAN** is the one VLAN whose frames cross a trunk with no tag. It is how 802.1Q handles untagged frames that arrive on a trunk. "Native VLAN" is the Cisco term. The standard calls the same idea the **PVID** (Port VLAN Identifier), and some vendors call it the "untagged VLAN".

Key facts:

- The default native VLAN on Cisco switches is VLAN 1.
- It is set per trunk port, not per switch.
- Both ends of a trunk should use the same native VLAN.
- The native VLAN is not the same idea as the default VLAN. The default VLAN is where ports start. The native VLAN is what a trunk does with untagged frames. They are both VLAN 1 out of the box, which causes the confusion.

Why does it exist? Early networks had shared segments with old devices that could not read tags. A device that did not understand 802.1Q could still join in, using untagged frames.

**What a mismatch does.** Suppose SW1 uses native VLAN 10 and SW2 uses native VLAN 20.

```text
       VLAN 10 frame           no tag            arrives as VLAN 20
PC1 --------------> [SW1] =================> [SW2] --------------> PC2
                  native 10    trunk       native 20
```

SW1 sends the VLAN 10 frame with no tag. SW2 sees no tag and puts it in VLAN 20. Traffic has leaked between two VLANs. The trunk stays up, so nothing looks broken at first.

Cisco switches detect this in two ways. CDP (Cisco Discovery Protocol, a neighbour discovery protocol) logs a message like this:

```text
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on
GigabitEthernet0/24 (10), with SW2 GigabitEthernet0/24 (20).
```

Per-VLAN spanning tree also notices it and blocks the affected VLANs on that port (shown as an inconsistent port state). So on Cisco gear the common symptom is lost traffic in those VLANs, not a clean leak. Other vendors may not detect it at all.

**A real-life way to think of it:** two post rooms agree on a rule for mail without a sticker. If each post room uses a different rule, the mail reaches the wrong department. The analogy stops here: a post room would not notice the mistake, but a Cisco switch often does.

### Allowed VLANs

By default a Cisco trunk carries all VLANs, 1-4094. You should limit this to what is needed.

Why limit it?

- **Less flooding.** Broadcasts in a VLAN are sent to every trunk that carries that VLAN, even if nobody on the far side needs them.
- **Less spanning tree work.** Fewer VLANs on a link means fewer spanning tree instances to run there.
- **Smaller blast radius.** A problem in one VLAN (such as a loop) cannot spread over a trunk that does not carry that VLAN.

The command has a trap.

Cisco IOS, interface configuration:

```text
switchport trunk allowed vlan 10,20      ! REPLACES list: only 10, 20
switchport trunk allowed vlan add 30     ! adds 30 to the list
switchport trunk allowed vlan remove 20  ! removes 20 from the list
switchport trunk allowed vlan except 30  ! all VLANs except 30
switchport trunk allowed vlan none       ! no VLANs at all
switchport trunk allowed vlan all        ! back to the default
```

If a trunk carries 10, 20 and 99 and you type `switchport trunk allowed vlan 30`, the trunk now carries only VLAN 30. If 99 was your management VLAN, you may lose your remote session. Always use `add` on a live trunk.

### ISL: The History

Before 802.1Q, Cisco used its own trunking method: **ISL** (Inter-Switch Link). You will meet it only as history or as an interview question.

ISL did not insert a tag. It wrapped the whole frame in a new header and trailer, like putting a letter into a bigger envelope.

```text
+----------------+--------------------------------+---------+
| ISL header     | Original frame, unchanged      | ISL FCS |
| 26 bytes       | up to 1518 bytes               | 4 bytes |
+----------------+--------------------------------+---------+
```

| | 802.1Q | ISL |
|---|---|---|
| Owner | IEEE standard | Cisco proprietary |
| Method | Inserts a tag | Wraps the whole frame |
| Overhead | 4 bytes | 30 bytes |
| Largest frame | 1522 bytes | 1548 bytes |
| Native (untagged) VLAN | Yes | No, every frame is wrapped |
| Original FCS | Recalculated | Kept, plus a new outer FCS |
| Status in 2026 | Used everywhere | Obsolete |

Cisco itself moved to 802.1Q, and many Catalyst lines, such as the 2960 and the Catalyst 9000 family, support only 802.1Q.

### DTP: Dynamic Trunking Protocol

**DTP** is a Cisco protocol that lets two directly connected switch ports agree on whether the link becomes a trunk. It is proprietary, so it does not help with other vendors' gear.

Think of two neighbours asking each other, "Shall we share this corridor?" One who is **desirable** asks actively. One who is **auto** waits to be asked. Two who both wait never start. The analogy stops being true here: real neighbours know each other, but a switch port trusts any device that sends the right DTP frame. That is the security problem.

| Mode | Command | Behaviour |
|---|---|---|
| Access | `switchport mode access` | Always an access port. |
| Trunk | `switchport mode trunk` | Always a trunk. Still sends DTP frames. |
| Dynamic auto | `switchport mode dynamic auto` | Becomes a trunk only if the other side asks. |
| Dynamic desirable | `switchport mode dynamic desirable` | Actively asks to become a trunk. |
| Nonegotiate | `switchport nonegotiate` (added to access or trunk) | Stops sending DTP frames. |

Result when both ends use these modes:

| | Access | Dynamic auto | Dynamic desirable | Trunk |
|---|---|---|---|---|
| **Access** | Access | Access | Access | Mismatch |
| **Dynamic auto** | Access | Access | Trunk | Trunk |
| **Dynamic desirable** | Access | Trunk | Trunk | Trunk |
| **Trunk** | Mismatch | Trunk | Trunk | Trunk |

The two results to remember:

- **Auto and auto gives no trunk.** Both ends wait.
- **Trunk plus nonegotiate facing a dynamic port gives no trunk.** The dynamic side never hears a DTP frame, so it stays an access port.

The default mode varies by model. Many current Catalyst switches default to dynamic auto, and some older ones (such as the 2950 and 3550 families) defaulted to dynamic desirable. Nexus switches do not use DTP. Check with `show interfaces <port> switchport`.

DTP also carries the VTP domain name. If the two switches report different domains, negotiation fails with a log such as `%DTP-5-DOMAINMISMATCH`.

**Why disable DTP:**

1. **Security.** A laptop running an attack tool can send DTP frames to a port in dynamic mode. If the port agrees to trunk, the attacker can reach every VLAN on it. This is **switch spoofing**.
2. **Predictability.** A link should be what you configured, not what two devices decided.
3. **Other vendors.** DTP does nothing useful on a link to a non-Cisco device.

The clean rule is:

Cisco IOS:

```text
! Port to a user
interface GigabitEthernet0/1
 switchport mode access
 switchport nonegotiate

! Port to another switch
interface GigabitEthernet0/24
 switchport mode trunk
 switchport nonegotiate
```

`switchport nonegotiate` is rejected on a dynamic port, because a dynamic port needs DTP to work. Set the mode first.

### Cisco Trunk Configuration

Steps to configure a trunk on a Cisco switch:

1. Create the VLANs. A trunk carries only VLANs that exist on the switch.
2. On older models that support both ISL and 802.1Q, set the encapsulation first.
3. Set the mode to trunk.
4. Turn off DTP.
5. Set the native VLAN.
6. Set the allowed list.

Cisco IOS:

```text
SW1(config)# interface GigabitEthernet0/24
SW1(config-if)# description TRUNK to SW2 Gi0/24
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport nonegotiate
SW1(config-if)# switchport trunk native vlan 999
SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
```

The `encapsulation` line is needed only on switches that offered both methods, such as the Catalyst 3560 and 3750. On a switch that supports only 802.1Q, the command does not exist. Leave it out. If you skip it on a switch that needs it, you see this:

```text
Command rejected: An interface whose trunk encapsulation is "Auto"
can not be configured to "trunk" mode.
```

Show commands you should know:

| Command | What it tells you |
|---|---|
| `show interfaces trunk` | All active trunks, native VLAN, allowed VLANs, active VLANs |
| `show interfaces gi0/24 switchport` | Admin and operational mode, encapsulation, DTP, native VLAN, allowed VLANs |
| `show interfaces status` | The VLAN column says `trunk` for trunk ports |
| `show vlan brief` | Which VLANs exist and their access ports. Trunk ports are NOT listed. |
| `show mac address-table vlan 10` | MAC addresses learned in one VLAN |
| `show dtp interface gi0/24` | DTP state of a port |
| `show cdp neighbors` | What is connected at the far end |
| `show spanning-tree vlan 10` | Whether the trunk is forwarding in that VLAN |

On newer switches ports have names like Gi1/0/24. The commands are the same.

### VTP: VLAN Trunking Protocol

Trunks carry VLAN traffic. They do not create VLANs. If you have 50 switches, you must create VLAN 30 on all 50. **VTP** (VLAN Trunking Protocol) is a Cisco protocol that copies the VLAN list from one switch to the others, over trunk links.

Analogy: head office keeps the master list of departments and sends copies to every branch. Each copy has a version number, and every branch keeps the newest. The analogy stops being true at one dangerous point: in VTP, "newest" means only "highest number", even if the list came from a test lab.

VTP copies the VLAN list only: numbers and names. It does not assign ports to VLANs. Switches in the same **VTP domain** (a group of switches with the same domain name) share the list.

| Mode | Can create or change VLANs? | Learns from others? | Relays adverts? |
|---|---|---|---|
| Server (default) | Yes | Yes | Yes |
| Client | No | Yes | Yes |
| Transparent | Yes, local only | No | Yes |
| Off | Yes, local only | No | No |

Off mode came with VTP version 3 and is not on every software version.

**Versions:**

| Version | Notes |
|---|---|
| 1 | Default on most Catalyst switches. Carries VLANs 1-1005. |
| 2 | Adds Token Ring support and some consistency checks. Same VLAN range. |
| 3 | Carries extended VLANs (1006-4094). Only a chosen primary server can make changes, which blocks the accident below. |

With versions 1 and 2, you must use transparent mode to create extended VLANs on a switch.

Servers send a summary advert every 5 minutes and also right after each change. Each advert has a **configuration revision number**. The number goes up by one on every VLAN change. A switch that sees a higher number, in the same domain and with the same password (if one is set), replaces its VLAN list with that newer one.

**The classic disaster (sometimes called a VTP bomb):**

1. A used lab switch has domain name CORP and revision 48. Its VLAN list holds only VLANs 1 and 5.
2. Production has revision 12 and many VLANs.
3. Someone connects the lab switch with a trunk.
4. Production switches see revision 48 and accept the small list.
5. VLANs vanish from every switch. Access ports in deleted VLANs go inactive. Users lose the network.

A client can do this too. Client mode stops local edits. It does not stop a switch from advertising a higher revision.

**Prevention:**

- Before connecting a used switch, run `show vtp status` and check the revision.
- With versions 1 and 2, reset the revision to 0 by changing the domain name, or by moving to transparent mode and back.
- Use VTP version 3 with a primary server if you use VTP at all.
- Or skip the risk: many networks use transparent or off mode and manage VLANs with automation.

Cisco IOS, a safe choice for a switch that should not join VTP:

```text
SW1(config)# vtp mode transparent
```

Cisco IOS, VTP version 3 with a primary server:

```text
SW1(config)# vtp domain CORP
SW1(config)# vtp version 3
SW1(config)# vtp mode server
SW1(config)# exit
SW1# vtp primary vlan
```

Check it. Values vary by switch:

```text
SW1# show vtp status
VTP Version capable             : 1 to 3
VTP version running             : 3
VTP Domain Name                 : CORP
VTP Pruning Mode                : Disabled
VTP Traps Generation            : Disabled
Device ID                       : 00d0.5800.1a00

Feature VLAN:
--------------
VTP Operating Mode                : Primary Server
Number of existing VLANs          : 9
Number of existing extended VLANs : 1
Maximum VLANs supported locally   : 4096
Configuration Revision            : 12
```

**VTP pruning** is a feature that stops flooded traffic from crossing a trunk when the far side has no active ports in that VLAN. It works on VLANs 2-1001 in versions 1 and 2. Turn it on with `vtp pruning`. A manual allowed list does a similar job, with more control and more typing.

## Example

### Example 1: Build a Trunk and Follow a Frame

**Topology.** Two switches, joined on Gi0/24.

```text
  PC1 10.0.10.11                         PC2 10.0.10.12
  MAC 0200.0000.0011                     MAC 0200.0000.0012
  VLAN 10                                VLAN 10
     |                                      |
   Gi0/1                                  Gi0/1
  +------+  Gi0/24               Gi0/24  +------+
  | SW1  |===============================| SW2  |
  +------+   802.1Q trunk                +------+
             allowed 10,20,99,999
             native 999
```

| VLAN | Name | Subnet | Purpose |
|---|---|---|---|
| 10 | STAFF | 10.0.10.0/24 | User PCs |
| 20 | VOICE | 10.0.20.0/24 | IP phones |
| 99 | MGMT | 10.0.99.0/24 | Switch management |
| 999 | NATIVE-UNUSED | None | Native VLAN, no users |

**Configuration.** Do the same on SW2, with the description changed.

Cisco IOS:

```text
SW1(config)# vlan 10
SW1(config-vlan)# name STAFF
SW1(config-vlan)# vlan 20
SW1(config-vlan)# name VOICE
SW1(config-vlan)# vlan 99
SW1(config-vlan)# name MGMT
SW1(config-vlan)# vlan 999
SW1(config-vlan)# name NATIVE-UNUSED
SW1(config-vlan)# exit
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport nonegotiate
SW1(config-if)# exit
SW1(config)# interface GigabitEthernet0/24
SW1(config-if)# description TRUNK to SW2 Gi0/24
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport nonegotiate
SW1(config-if)# switchport trunk native vlan 999
SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
```

**Verify.** Learn to read all four sections of this output.

```text
SW1# show interfaces trunk

Port        Mode         Encapsulation  Status        Native vlan
Gi0/24      on           802.1q         trunking      999

Port        Vlans allowed on trunk
Gi0/24      10,20,99,999

Port        Vlans allowed and active in management domain
Gi0/24      10,20,99,999

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/24      10,20,99,999
```

- `Mode on` means forced trunk. You may also see `desirable` or `auto`.
- `802.1q` means set by hand. `n-802.1q` means negotiated with DTP.
- Section 2 is what you configured.
- Section 3 is section 2 minus VLANs that do not exist on this switch.
- Section 4 is section 3 minus VLANs that spanning tree blocks or VTP prunes.

```text
SW1# show interfaces gi0/24 switchport
Name: Gi0/24
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 999 (NATIVE-UNUSED)
Administrative Native VLAN tagging: enabled
Trunking VLANs Enabled: 10,20,99,999
Pruning VLANs Enabled: 2-1001
```

`Negotiation of Trunking: Off` confirms that DTP is disabled.

**Follow a frame.** PC1 pings PC2. PC1 does not yet know PC2's MAC address, so it sends an [ARP](05-arp.md) broadcast.

1. PC1 sends an untagged broadcast frame. It arrives on SW1 Gi0/1.
2. SW1 knows Gi0/1 is in VLAN 10. It learns `0200.0000.0011` in VLAN 10.
3. SW1 floods the frame to other ports in VLAN 10. Gi0/2 in VLAN 20 does not get it.
4. VLAN 10 is allowed on Gi0/24, so SW1 sends it out there with a tag.

The bytes on the trunk look like this:

```text
ff ff ff ff ff ff   02 00 00 00 00 11   81 00   00 0a   08 06   ...
|-- Dst MAC -----|  |-- Src MAC ----|   TPID    TCI     Type
                                                VID=10  ARP
```

5. SW2 reads TPID 0x8100 and VID 10. It learns `0200.0000.0011` in VLAN 10 on Gi0/24.
6. SW2 removes the tag, recalculates the FCS, and floods the frame out its VLAN 10 access ports.
7. PC2 gets a plain untagged frame. It never knows a tag existed.

After the ping finishes, SW2 holds both MACs in VLAN 10:

```text
SW2# show mac address-table vlan 10
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  10    0200.0000.0011    DYNAMIC     Gi0/24
  10    0200.0000.0012    DYNAMIC     Gi0/1
```

### Example 2: Decode a Tag by Hand

A capture shows these 4 bytes after the source MAC: `81 00 A0 14`.

`81 00` is the TPID, so this is an 802.1Q tag. `A0 14` is the TCI.

```text
Hex:     A    0    1    4
Binary:  1010 0000 0001 0100

PCP = first 3 bits  = 101             = 5
DEI = next 1 bit    = 0               = 0
VID = last 12 bits  = 0000 0001 0100  = 20
```

The frame is in VLAN 20 with priority 5, which is the usual choice for voice. This is a VOICE frame from Example 1.

### Example 3: VLAN Allowed but Not Crossing

Users in VLAN 30 on SW1 cannot reach a server in VLAN 30 on SW3. VLANs 10 and 20 work fine. VLAN 30 exists on SW1 and SW3.

```text
+-----+ Gi0/24         Gi0/24 +-----+ Gi0/23         Gi0/24 +-----+
| SW1 |=======================| SW2 |=======================| SW3 |
+-----+                       +-----+                       +-----+
users, VLAN 30              no ports in               server, VLAN 30
                              VLAN 30
```

Check the switch in the middle:

```text
SW2# show interfaces trunk

Port        Mode         Encapsulation  Status        Native vlan
Gi0/23      on           802.1q         trunking      999
Gi0/24      on           802.1q         trunking      999

Port        Vlans allowed on trunk
Gi0/23      10,20,30,99,999
Gi0/24      10,20,30,99,999

Port        Vlans allowed and active in management domain
Gi0/23      10,20,99,999
Gi0/24      10,20,99,999

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/23      10,20,99,999
Gi0/24      10,20,99,999
```

VLAN 30 is in section 2 but missing from section 3. It is allowed, but it does not exist on SW2. SW2 is a transit switch: it has no VLAN 30 users, so nobody created it. A switch cannot forward a VLAN it does not have.

The fix:

```text
SW2(config)# vlan 30
SW2(config-vlan)# name SERVERS
```

Use this as your reading guide:

| VLAN missing from | Likely cause |
|---|---|
| Section 2 (allowed) | Not in the allowed list. Fix with `allowed vlan add`. |
| Section 3 (active) | VLAN does not exist on this switch. Create it. |
| Section 4 (forwarding) | Spanning tree is blocking it, or VTP has pruned it. |

### Example 4: Tagged Traffic on a Host

A server can have a trunk too. This is common for hypervisors and routers running on Linux.

Linux, create a sub-interface for VLAN 10 on eth0:

```bash
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 10.0.10.50/24 dev eth0.10
sudo ip link set eth0.10 up
ip -d link show eth0.10
```

```text
5: eth0.10@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vlan protocol 802.1Q id 10 <REORDER_HDR>
```

Windows, set one VLAN on an adapter whose driver supports it, or set VLANs on a Hyper-V virtual adapter:

```powershell
Set-NetAdapter -Name "Ethernet" -VlanID 10
Set-VMNetworkAdapterVlan -VMName web01 -Trunk -AllowedVlanIdList "10,20" -NativeVlanId 99
```

See the tag on Linux with [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md). The `-e` flag prints the Layer 2 header:

```bash
sudo tcpdump -e -n -i eth0 vlan 10
```

```text
12:04:31.201144 52:54:00:12:34:56 > 00:1a:2b:3c:4d:10, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 10.0.10.50 > 10.0.10.1: ICMP echo request, id 4242, seq 1, length 64
```

Check the length: 64 (ICMP) + 20 (IP) + 14 (Ethernet) + 4 (tag) = 102. `p 0` is the PCP value.

Network card drivers may add or remove tags in hardware. On some systems, especially Windows, a capture may then show no tag even though the wire has one.

## Going Deeper

### Q-in-Q (802.1ad)

**The problem.** A service provider carries Layer 2 traffic for many customers. Customer A and Customer B both use VLAN 10. They both want their own VLAN numbers kept. One 12-bit VLAN field cannot keep them apart.

**The idea.** Add a second tag. The provider's tag goes outside the customer's tag.

Analogy: put a sealed envelope inside a larger courier envelope. The courier reads only the outer address. The analogy stops being true here: an envelope hides the contents, but Q-in-Q hides nothing. It only means the provider's switches ignore the inner tag.

The names:

- **S-tag** (Service tag): the outer tag, owned by the provider.
- **C-tag** (Customer tag): the inner tag, owned by the customer.

Q-in-Q was standardised as IEEE 802.1ad (provider bridging), later merged into IEEE 802.1Q. The standard S-tag TPID is 0x88A8. The C-tag keeps 0x8100.

```text
+-----+-----+-----------+-----------+-----------+---------+-----+
| Dst | Src | S-tag     | C-tag     | EtherType | Payload | FCS |
| MAC | MAC | 0x88A8    | 0x8100    | 0x0800    |         |     |
|     |     | VID 500   | VID 10    | IPv4      |         |     |
+-----+-----+-----------+-----------+-----------+---------+-----+
             provider    customer
```

**The numbers.** Two 12-bit fields give up to 4094 x 4094 = 16,760,836 combinations.

**The flow.**

```text
Customer A                 Provider                     Customer A
site 1                                                  site 2

CE-A1 -- VID 10 --> PE1 === VID 500+10 === PE2 -- VID 10 --> CE-A2
CE-B1 -- VID 10 --> PE1 === VID 600+10 === PE2 -- VID 10 --> CE-B2

                 push S-tag              pop S-tag
```

1. CE-A1 (the customer's edge switch) sends a frame tagged VID 10 on its trunk.
2. PE1 (the provider's edge switch) receives it on a tunnel port for Customer A, mapped to VLAN 500.
3. PE1 pushes S-tag 500 without reading or changing the C-tag.
4. Provider switches forward on the outer tag only.
5. PE2 pops the S-tag and sends the frame out to CE-A2, with C-tag 10 unchanged.

Customer B can use VID 10 at the same time, because its frames carry S-tag 600.

Cisco IOS (Catalyst style, the provider edge):

```text
PE1(config)# vlan 500
PE1(config-vlan)# name CUSTOMER-A
PE1(config-vlan)# exit
PE1(config)# vlan dot1q tag native
PE1(config)# interface GigabitEthernet0/1
PE1(config-if)# description Customer A site 1
PE1(config-if)# switchport access vlan 500
PE1(config-if)# switchport mode dot1q-tunnel
```

Verify with `show dot1q-tunnel` and `show vlan dot1q tag native`.

The customer's own port stays a normal trunk, so the link has a trunk on one end and a tunnel port on the other. The customer's untagged native VLAN frames get only the S-tag.

Things that often go wrong:

- **MTU.** Each tag adds 4 bytes on the wire: 1518 untagged, 1522 with one tag, 1526 with two. Every provider link must accept the larger frame. On some Catalyst models you set `system mtu 1504` or higher, and the command differs by platform.
- **TPID.** Cisco Catalyst tunnel ports use 0x8100 for the outer tag by default, not 0x88A8. Some platforms let you change it. Check this when you connect to another vendor.
- **Native VLAN on provider trunks.** If an S-VLAN is the native VLAN of a provider trunk, the outer tag is not sent. The inner tag then looks like the outer tag to the next switch. That is why the example turns on `vlan dot1q tag native`.
- **Control protocols.** Customer CDP, STP and VTP frames do not cross by default. `l2protocol-tunnel` on the tunnel port carries them.
- **MAC scale.** Provider switches still learn customer MAC addresses. IEEE 802.1ah (Provider Backbone Bridges, also called MAC-in-MAC) was made to solve this.

Today many providers carry such services with [MPLS](../05-routing/13-mpls.md), and data centres use [VXLAN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md), whose 24-bit ID gives about 16 million segments.

### VLAN Hopping

**VLAN hopping** means sending traffic into a VLAN you should not reach, without going through a router. There are two classic methods. Full defences live in [Layer 2 Security](11-layer2-security.md).

**Switch spoofing.** The attacker's device speaks DTP and turns a dynamic port into a trunk. Defence: `switchport mode access` and `switchport nonegotiate`.

**Double tagging.** The attacker sits in the same VLAN as the trunk's native VLAN and sends a frame with two tags.

```text
Attacker frame: [outer tag VLAN 1][inner tag VLAN 20][data]

   Attacker (VLAN 1, also the trunk's native VLAN)
      |
      v
    [SW1]  Reads outer tag: VLAN 1, the native VLAN.
      |    Native frames leave untagged, so it strips the outer tag.
      |    The inner tag now looks like the real tag.
      v
    [SW2]  Reads tag: VLAN 20. Sends the frame into VLAN 20.
      |
      v
    Victim in VLAN 20
```

It works in one direction only, because replies have no way back. That is still enough to send harmful traffic.

Defences:

- Make the native VLAN an unused VLAN with no user ports, such as 999.
- Never put users in the native VLAN.
- Optionally tag the native VLAN too.

### Tagging the Native VLAN

On Cisco Catalyst switches, this global command makes trunks tag native VLAN frames too:

```text
SW1(config)# vlan dot1q tag native
```

With it on, untagged data frames that arrive on a trunk are dropped. Control traffic is still accepted untagged. It applies to all trunks on the switch, so plan it across both ends.

### VLAN 1 and Control Traffic

Cisco control protocols such as CDP, DTP, VTP and PAgP keep using VLAN 1 even if you remove VLAN 1 from a trunk's allowed list. So removing it is safe for them and is a common hardening step: user traffic and spanning tree for VLAN 1 stop on that trunk.

### PCP and the Native VLAN

PCP lives inside the tag. A frame with no tag has no PCP. So native VLAN frames lose their Layer 2 priority across a trunk. This is one more reason not to put real traffic there. A priority-tagged frame (VID 0) solves the reverse case: it keeps a priority value without choosing a VLAN.

### Frame Size

A 1500-byte payload stays 1500 bytes with a tag. The frame grows, the payload does not. IEEE 802.3ac raised the maximum frame to 1522 bytes for this reason. Very old gear may count these as errors, and Cisco calls such slightly large frames **baby giants**. IP MTU is unchanged. For more on MTU, see [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Trunks in Other Places

- **Router on a stick:** a router's sub-interfaces use `encapsulation dot1Q 10` to route between VLANs over one trunk. See [Inter-VLAN Routing](08-inter-vlan-routing.md).
- **Voice VLAN:** a phone tags its voice traffic while the PC behind it stays untagged. The port is still an access port with one extra voice VLAN, not a full trunk. See [VLANs](06-vlans.md).
- **EtherChannel:** apply trunk settings on the port-channel interface. All member ports must match. See [Link Aggregation](10-link-aggregation.md).
- **Spanning tree:** a trunk is a loop risk, and per-VLAN spanning tree runs one instance per VLAN on it. See [Spanning Tree Protocol](09-spanning-tree.md).
- **Hypervisors:** the physical switch port is a trunk, and the virtual switch hands tagged frames to the right VM. In VMware port groups, VLAN ID 4095 has a special meaning: pass all tags to the guest.

### Vendor Words

The same word can mean different things.

| Term | Cisco | HPE Aruba (ProCurve style) |
|---|---|---|
| Link carrying many VLANs | Trunk | Port with tagged VLANs |
| Link aggregation | EtherChannel or port-channel | Trunk |
| Untagged VLAN on a multi-VLAN port | Native VLAN | Untagged VLAN |

If an HPE engineer says "trunk", ask which one they mean. Juniper uses "trunk" the way Cisco does.

## Common Mistakes

- **"All frames on a trunk are tagged."** Native VLAN frames are untagged by default.
- **"The native VLAN must be VLAN 1."** It is only the default. It can be changed, and an unused VLAN is safer.
- **"The native VLAN is the management VLAN."** They are separate ideas. Management traffic should be in its own tagged VLAN.
- **"802.1Q wraps the frame."** It inserts a 4-byte tag inside the header. ISL was the one that wrapped. The Cisco word "encapsulation" adds to the confusion.
- **"There are 4096 VLANs."** There are 4094 usable IDs, because 0 and 4095 are reserved.
- **"A tag reduces the payload to 1496 bytes."** Payload stays 1500. The frame maximum grows to 1522.
- **"A trunk creates VLANs."** VLANs must exist on each switch, including transit switches. An allowed list does not create anything.
- **Forgetting `add`.** `switchport trunk allowed vlan 30` replaces the list and can cut off your own session.
- **Looking for trunk ports in `show vlan brief`.** They are not there. Use `show interfaces trunk`.
- **Believing `switchport trunk native vlan 999` works on an access port.** The setting is stored, but it does nothing until the port is a trunk. The same is true of `switchport access vlan` on a trunk.
- **"VTP client mode is safe."** A client with a higher revision can overwrite the whole domain.
- **"VTP assigns ports to VLANs."** VTP shares only the VLAN list. Port membership is still set on each switch.
- **"Trunking gives security."** Tags give separation, not encryption. Anyone capturing a trunk sees all the VLANs on it.

## Interview Questions

**Q: What is a trunk port, and how is it different from an access port?**

A: An access port carries one VLAN and sends untagged frames, usually to an end device. A trunk port carries many VLANs over one link, and it marks frames with an 802.1Q tag so the far end knows each frame's VLAN.

**Q: Describe the 802.1Q tag.**

A: It is 4 bytes, inserted after the source MAC. The first 2 bytes are the TPID, 0x8100. The next 2 bytes hold PCP (3 bits of priority), DEI (1 bit, drop eligible) and VID (12 bits, the VLAN number). Adding the tag means the FCS must be recalculated, and the maximum frame grows from 1518 to 1522 bytes.

**Q: What is the native VLAN, and what happens with a mismatch?**

A: It is the VLAN whose frames cross a trunk untagged, and any untagged frame received on the trunk goes into it. If the two ends differ, frames leave one side in one VLAN and arrive in another. On Cisco, CDP logs a mismatch, and per-VLAN spanning tree usually blocks the affected VLANs. The link itself stays up.

**Q: Why should you change the native VLAN from VLAN 1?**

A: VLAN 1 is the default for ports, so users may sit in it, and that enables double-tagging attacks. Using an unused VLAN with no user ports removes that path. You can also tag the native VLAN with `vlan dot1q tag native`.

**Q: How many VLANs does 802.1Q support, and why?**

A: 4094 usable VLANs. The VID has 12 bits, which gives 4096 values, and 0 and 4095 are reserved. VID 0 means a priority-only frame.

**Q: What is the difference between ISL and 802.1Q?**

A: ISL is Cisco proprietary, wraps the whole frame with 30 bytes of overhead, and has no native VLAN. 802.1Q is an IEEE standard, inserts a 4-byte tag, and has a native VLAN. ISL is obsolete and many current switches do not support it.

**Q: What is DTP, and why would you disable it?**

A: DTP is a Cisco protocol that negotiates trunking between two ports, using modes such as dynamic auto and dynamic desirable. I would disable it because an attacker could use it to form a trunk and reach many VLANs. I set ports to access or trunk by hand and add `switchport nonegotiate`.

**Q: Two switches are both set to dynamic auto. What happens?**

A: No trunk forms. Both sides wait for the other to ask, so the link stays in access mode. It needs at least one side set to trunk or dynamic desirable.

**Q: What is VTP, and what is the main risk?**

A: VTP is a Cisco protocol that shares the VLAN list across switches in a domain, over trunks. The risk is that a switch with a higher revision number, in the same domain, can replace everyone's VLAN list. A used lab switch can delete production VLANs. To avoid it, reset the revision, use VTP version 3 with a primary server, or use transparent mode.

**Q: A VLAN is in the allowed list, but its traffic does not cross the trunk. How do you troubleshoot it?**

A: I run `show interfaces trunk` and read the sections. If the VLAN is allowed but not active, it does not exist on that switch, so I create it, and I check transit switches too. If it is active but not in the last section, spanning tree is blocking it or VTP has pruned it. I also check the native VLAN and the other end.

**Q: What is Q-in-Q, and when is it used?**

A: Q-in-Q, or 802.1ad, adds a second tag. A provider puts an outer S-tag on customer frames and leaves the customer's C-tag alone. Many customers can then reuse the same VLAN numbers. The standard S-tag TPID is 0x88A8, and the links must allow larger frames because each tag adds 4 bytes.

**Q: What can go wrong when you type `switchport trunk allowed vlan 30` on a live trunk?**

A: It replaces the whole list, so the trunk carries only VLAN 30. Every other VLAN drops, maybe including management. The correct command is `switchport trunk allowed vlan add 30`.

## Quick Recap

- A trunk carries many VLANs on one link. An access port carries one VLAN with no tag.
- The 802.1Q tag is 4 bytes after the source MAC: TPID 0x8100, PCP 3 bits, DEI 1 bit, VID 12 bits. Usable VLANs are 1-4094, and the maximum frame is 1522 bytes.
- Native VLAN frames are untagged. Match it on both ends and use an unused VLAN.
- Limit trunks with an allowed list, use `add` on live links, and remember that VLANs must exist on every switch in the path, including transit switches.
- ISL is obsolete. Disable DTP with fixed modes plus `switchport nonegotiate`.
- VTP shares VLAN lists. A higher revision number can wipe a network, so use version 3 or transparent mode.
- Q-in-Q adds an outer S-tag so a provider can carry customers with overlapping VLAN numbers.
