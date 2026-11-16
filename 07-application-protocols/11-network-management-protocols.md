# Network Management Protocols

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [UDP in Depth](../06-transport-layer/07-udp.md), [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md)

## In One Line

Network management protocols let engineers watch, measure and understand devices from one place: SNMP reads their health, syslog collects their messages, NTP and PTP agree the time, NetFlow, sFlow and IPFIX show who is talking to whom, and LLDP and CDP show what is plugged into what.

## Why It Matters

A company may run 500 switches and routers. Nobody can log in to each one every morning. Without these protocols, the first alert is an angry phone call from a user.

Each protocol answers one question:

| Question | Protocol |
|---|---|
| Is the device healthy? How busy is this link? | SNMP |
| What did the device say happened, and when? | Syslog |
| Can I trust the timestamps? | NTP, PTP |
| Who is using the bandwidth? | NetFlow, sFlow, IPFIX |
| What is connected to this port? | LLDP, CDP |

Interviewers ask about this topic because it shows real operations experience. Common checks are: the difference between a poll and a trap, why SNMPv3 replaced community strings, the eight syslog levels in order, what stratum means, and why flow data is different from a packet capture.

## Simple Explanation

Think of a hospital.

- **SNMP** is the nurse's chart. Every patient has the same fields: temperature, pulse, blood pressure. The nurse walks round and reads them. That is polling. If a heart monitor screams, the alarm comes to the nurse without being asked. That is a trap.
- **Syslog** is the ward diary. Staff write what happened, and each line has a level of seriousness.
- **NTP** keeps all the wall clocks the same. If the clocks differ, nobody can say which event came first.
- **NetFlow** is the visitor book. It says who came, who they saw and how long they stayed. It does not record what they said.
- **LLDP and CDP** are name badges. Each patient tells the next bed, "I am Ali, bed 4."

Where the analogy stops: hospital staff think and choose what matters. Network devices only send what they are configured to send, and much of it goes over UDP, so a message can be lost with no warning.

## How It Works

### Summary of ports

Learn this table first. It comes up in many interviews.

| Protocol | Transport and port | Direction |
|---|---|---|
| SNMP queries | UDP 161 | Manager to agent |
| SNMP traps and informs | UDP 162 | Agent to manager |
| SNMP over TLS or DTLS | 10161, 10162 | Rarely used |
| Syslog | UDP 514 | Device to server |
| Syslog over TLS | TCP 6514 | Device to server |
| NTP | UDP 123 | Client and server |
| NTS key setup | TCP 4460 | Client to server |
| PTP | UDP 319 and 320, or EtherType 0x88F7 | Both ways |
| NetFlow | UDP, no official port, often 2055, 9995 or 9996 | Device to collector |
| IPFIX | UDP, TCP or SCTP, default port 4739 | Device to collector |
| sFlow | UDP 6343 | Device to collector |
| LLDP | EtherType 0x88CC, no IP needed | One hop only |
| CDP | LLC/SNAP, protocol ID 0x2000, no IP needed | One hop only |

### SNMP: the parts

SNMP means Simple Network Management Protocol. It reads and sometimes changes values on a device.

- **Manager:** the software that asks questions and receives alerts. It is also called the NMS (Network Management Station). LibreNMS, Zabbix and SolarWinds are examples.
- **Agent:** a small program on the device. It answers the manager and sends alerts.
- **MIB (Management Information Base):** a description of what can be asked. It is a text file that lists each value, its type and its meaning. The MIB is the menu. The live numbers are kept by the agent.
- **OID (Object Identifier):** the unique number of one value in that menu, written as numbers with dots.

### OIDs and the MIB tree

OIDs form a tree. It works like a postal address that goes from country to city to street.

```text
root
 +-- iso (1)
      +-- org (3)
           +-- dod (6)
                +-- internet (1)
                     +-- mgmt (2)
                     |    +-- mib-2 (1)
                     |         +-- system (1)
                     |         |    +-- sysDescr (1)
                     |         |    +-- sysUpTime (3)
                     |         |    +-- sysName (5)
                     |         +-- interfaces (2)
                     |         +-- ifMIB (31)
                     +-- private (4)
                          +-- enterprises (1)
                               +-- cisco (9)
```

Follow the numbers down and you get `1.3.6.1.2.1.1.5`, which is `sysName`. Add `.0` for a single-value item: `1.3.6.1.2.1.1.5.0`. For table items, the last number is the row. In `ifDescr.2`, the 2 is the interface index (ifIndex), a number the device gives to each port.

Where the analogy stops: a postal address names one place forever. An ifIndex can change after a reboot on some devices, unless the device is told to keep it (Cisco: `snmp-server ifindex persist`).

OIDs worth knowing:

| Name | OID | Meaning |
|---|---|---|
| sysDescr.0 | 1.3.6.1.2.1.1.1.0 | Device description |
| sysUpTime.0 | 1.3.6.1.2.1.1.3.0 | Time since the agent started |
| sysName.0 | 1.3.6.1.2.1.1.5.0 | Host name |
| ifDescr | 1.3.6.1.2.1.2.2.1.2 | Interface name |
| ifOperStatus | 1.3.6.1.2.1.2.2.1.8 | Port up or down |
| ifInErrors | 1.3.6.1.2.1.2.2.1.14 | Bad packets received |
| ifHCInOctets | 1.3.6.1.2.1.31.1.1.1.6 | 64-bit bytes in |
| ifHCOutOctets | 1.3.6.1.2.1.31.1.1.1.10 | 64-bit bytes out |

Standard values sit under `1.3.6.1.2.1`. Vendor values sit under `1.3.6.1.4.1`, followed by the vendor's number. Cisco is 9.

### SNMP operations

SNMP has a small set of messages, called PDUs (Protocol Data Units).

| PDU | Sender | Purpose | Added in |
|---|---|---|---|
| GetRequest | Manager | Read named values | v1 |
| GetNextRequest | Manager | Read the next value in the tree | v1 |
| SetRequest | Manager | Change a value | v1 |
| Response | Agent | Answer a request | v1 |
| Trap | Agent | Alert with no acknowledgement | v1 |
| GetBulkRequest | Manager | Read many values in one request | v2c |
| InformRequest | Agent | Alert that must be acknowledged | v2c |
| Report | Either | Internal v3 errors and discovery | v3 |

A walk is not a separate PDU. A tool such as `snmpwalk` repeats GetNext (or GetBulk) until it leaves the part of the tree you asked for.

```text
Manager 10.0.0.50                          Agent 10.0.0.2

      | Get sysUpTime.0                        |
      | src 51022  dst 161                     |
      |--------------------------------------->|
      |                                        |
      | Response                               |
      | src 161    dst 51022                   |
      |<---------------------------------------|
      |                                        |
      | Trap linkDown (no reply)               |
      | src 57144  dst 162                     |
      |<---------------------------------------|
```

So the manager must listen on UDP 162, and the agent listens on UDP 161. A firewall needs both directions if you use traps.

**Trap versus inform.** A trap is a shout. Nobody confirms it. If the UDP packet is lost, the alert is lost. An inform is a shout that waits for "I heard you" and repeats if nothing comes back. Informs are safer, but they use more memory on the device. Both go to port 162.

### SNMP message layout

Version 1 and 2c messages have this simple shape:

```text
+---------+-----------+-------------------------------------------+
| Version | Community | PDU                                       |
|         | string    | type, request-id, error fields, varbinds  |
+---------+-----------+-------------------------------------------+
```

A **varbind** (variable binding) is an OID and value pair. One message can carry many of them.

The whole message is encoded with ASN.1 BER, a compact binary format. ASN.1 describes the structure and BER (Basic Encoding Rules) describes the bytes. It is not text, so you need Wireshark to read it easily.

### SNMP versions

| Feature | v1 | v2c | v3 |
|---|---|---|---|
| Security | Community string | Community string | Users, auth, encryption |
| Sent in clear text | Yes | Yes | No, with authPriv |
| GetBulk | No | Yes | Yes |
| Inform | No | Yes | Yes |
| 64-bit counters | No | Yes | Yes |
| Standard | RFC 1157 | RFC 1901 and others | RFC 3411 to 3418 |

A **community string** is a shared word that works like a password. It travels in clear text, so anyone capturing the traffic can read it. The names `public` for read and `private` for write are old conventions that some devices shipped with. Current Cisco IOS has no community until you make one.

SNMPv2 first appeared with a complex security design that did not win support. The "c" in v2c means community-based: it kept v2's protocol features and went back to v1's weak security. It is still very common today, mostly on trusted management networks.

SNMPv3 adds three security levels:

| Level | Checks who sent it | Encrypts data |
|---|---|---|
| noAuthNoPriv | User name only | No |
| authNoPriv | Yes, with a hash | No |
| authPriv | Yes, with a hash | Yes |

Always use **authPriv** in production. Prefer SHA over MD5 and AES over DES. SHA-2 for SNMPv3 is defined in RFC 7860, but older devices may support only SHA-1. Some newer platforms no longer accept MD5 and DES.

### SNMP configuration

Cisco IOS, SNMPv3 with authPriv:

```text
ip access-list standard SNMP-MGRS
 permit host 10.0.0.50
!
snmp-server group NMS-GROUP v3 priv access SNMP-MGRS
snmp-server user nms-user NMS-GROUP v3 auth sha LabAuth-Pass1 priv aes 128 LabPriv-Pass1
snmp-server host 10.0.0.50 version 3 priv nms-user
snmp-server enable traps snmp linkdown linkup
snmp-server location Head-Office-Rack-2
snmp-server contact noc@example.com
```

The `snmp-server user` line does not appear in `show running-config`. Check it with `show snmp user`.

Linux, querying a device with Net-SNMP tools:

```bash
# Linux: read one value with SNMPv3
snmpget -v3 -l authPriv -u nms-user -a SHA -A 'LabAuth-Pass1' \
  -x AES -X 'LabPriv-Pass1' 10.0.0.2 1.3.6.1.2.1.1.3.0

# Output
SNMPv2-MIB::sysUpTime.0 = Timeticks: (123456789) 14 days, 6:56:07.89
```

Timeticks are hundredths of a second. On Debian and Ubuntu, you may see `iso.3.6.1...` instead of names until MIB files are installed (package `snmp-mibs-downloader`).

Windows has no built-in SNMP query tool. The Windows SNMP service is deprecated but remains an optional feature:

```powershell
# Windows: check if the optional SNMP feature is installed
Get-WindowsCapability -Online -Name "SNMP*"

# Windows: install it if you need it
Add-WindowsCapability -Online -Name "SNMP.Client~~~~0.0.1.0"
```

### Syslog

Syslog is a standard way for a device to send text event messages to a central server. The server is called a **collector**. The current standard is RFC 5424. The older BSD format is described in RFC 3164, and many devices still send it.

Every message has a facility (the part of the system it came from) and a severity (how serious it is).

**Severity levels. Lower number means worse.**

| Level | Name | Meaning | Example |
|---|---|---|---|
| 0 | Emergency | System unusable | Device about to crash |
| 1 | Alert | Act now | Temperature limit passed |
| 2 | Critical | Critical condition | Hardware failure |
| 3 | Error | Error condition | Interface went down |
| 4 | Warning | Warning condition | Config mismatch |
| 5 | Notice | Normal but important | Line protocol changed, login |
| 6 | Informational | Normal event | ACL hit logged |
| 7 | Debug | Debug detail | Output of `debug` commands |

A common memory aid for 0 to 7: **E**very **A**wesome **C**isco **E**ngineer **W**ill **N**eed **I**ce cream **D**aily.

Setting a level means "this level and everything more serious." So `logging trap warning` sends levels 0 to 4.

**The PRI value.** Facility and severity are packed into one number at the start of the message:

```text
PRI = (facility x 8) + severity
```

Facility numbers include 0 kernel, 3 daemon, 4 auth, and 16 to 23 for `local0` to `local7`. Cisco IOS uses `local7` by default.

So a Cisco level 3 message is (23 x 8) + 3 = 187, and it starts with `<187>`. To decode, divide by 8: 187 / 8 = 23 remainder 3. Facility 23, severity 3.

**RFC 5424 message layout:**

| Field | Example | Meaning |
|---|---|---|
| PRI | `<187>` | Facility and severity |
| VERSION | `1` | Syslog format version |
| TIMESTAMP | `2026-10-30T09:14:03.482Z` | When it happened, UTC here |
| HOSTNAME | `sw1.example.com` | Sender |
| APP-NAME | `ifmgr` | Program name |
| PROCID | `2841` | Process ID |
| MSGID | `LINKDOWN` | Message type |
| STRUCTURED-DATA | `-` | Optional key and value data, `-` if none |
| MSG | `Gi0/2 went down` | Free text |

On the wire:

```text
<187>1 2026-10-30T09:14:03.482Z sw1.example.com ifmgr 2841 LINKDOWN - Gi0/2 went down
```

**Transport.** UDP 514 is the classic choice. It is fast, but messages can be lost, and they can be read or faked. TLS on TCP 6514 (RFC 5425) fixes this. Many collectors also accept plain syslog on TCP 514. That is a common practice, but IANA actually lists TCP 514 for the old `rsh` shell service, so do not call it the standard.

Cisco IOS:

```text
service timestamps log datetime msec show-timezone
logging host 10.0.0.51
logging trap informational
logging source-interface Loopback0
logging buffered 64000 informational
```

`logging trap` sets the level sent to the server. The word "trap" here has nothing to do with SNMP traps.

Linux, with rsyslog:

```bash
# Linux: /etc/rsyslog.d/90-forward.conf
# One @ means UDP, two @@ means TCP
*.info @10.0.0.51:514

# Apply, then send a test message over UDP
sudo systemctl restart rsyslog
logger -n 10.0.0.51 -P 514 -d -p local0.notice "syslog test from web1"

# Read local errors and worse from the journal
journalctl -p err -n 20
```

Windows uses the Event Log, not syslog. It has no built-in syslog sender, so a third-party agent forwards events.

```powershell
# Windows: show recent critical (1) and error (2) events
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2} -MaxEvents 5
```

### NTP: why time matters

NTP means Network Time Protocol. The current version is NTPv4, RFC 5905. It uses UDP port 123.

Bad clocks cause real failures:

- **Log correlation:** if two devices disagree by 7 minutes, the cause appears after the effect.
- **TLS:** certificates have start and end dates. A wrong clock makes valid certificates fail. See [TLS and HTTPS](07-tls-and-https.md).
- **Kerberos:** logins fail when clocks differ too much. The usual default limit is 5 minutes. See [Directory and Authentication Protocols](12-directory-and-authentication-protocols.md).
- **One-time codes:** phone login codes (TOTP) usually change every 30 seconds, based on the clock.
- **Databases and billing:** ordering and records depend on timestamps.
- **Law:** a security investigation needs believable times.

### NTP stratum

**Stratum** means how many steps a clock is from a reference source.

```text
stratum 0    [GPS receiver]     [atomic clock]    not on the network
                   |                  |           (direct cable)
stratum 1    [time server A]    [time server B]
                   \                  /
stratum 2       [company NTP 10.0.0.60]
                    |            |
stratum 3         [R1]         [SW1]
                                 |
stratum 4                 [a server syncing
                           from SW1]

stratum 16 = not synchronised
```

Think of copying the time from person to person. The person with the atomic clock is stratum 0. The person reading it directly is stratum 1. The person asking them is stratum 2.

Where the analogy stops: a higher stratum does not always mean a worse clock. A nearby stratum 2 server on a clean LAN can beat a stratum 1 server far away on a busy Internet path. Stratum counts hops, not accuracy.

The valid range for synchronised clocks is 1 to 15. Stratum 16 means unsynchronised.

### NTP exchange

The client works out how far its clock is from the server using four timestamps.

```text
 Client                                   Server
   | T1  ---------- request ------------->  | T2
   |                                        |
   | T4  <--------- reply ----------------  | T3
```

- T1: client sends.
- T2: server receives.
- T3: server sends.
- T4: client receives.

Two formulas:

```text
delay  = (T4 - T1) - (T3 - T2)
offset = ((T2 - T1) + (T3 - T4)) / 2
```

Worked numbers, only showing the seconds after 10:00:00:

| Stamp | Clock read | Value |
|---|---|---|
| T1 | Client | 0.000 |
| T2 | Server | 0.130 |
| T3 | Server | 0.131 |
| T4 | Client | 0.041 |

- Delay = (0.041 - 0.000) - (0.131 - 0.130) = 0.040 s, so 40 ms round trip.
- Offset = (0.130 + (0.131 - 0.041)) / 2 = (0.130 + 0.090) / 2 = 0.110 s.

The client is 110 ms behind the server. NTP assumes each direction takes the same time. If the path is not equal, the answer has an error of up to half the difference between the two directions.

**Correcting the clock.** A small error is fixed by gently speeding up or slowing down the clock. This is called **slewing**. A large error may be fixed with a jump, called **stepping**. Slewing is safer, because time never goes backwards for running software. ntpd's default step threshold is 128 ms. It also gives up if the error is above 1000 s, unless you allow it. chrony's sample configs usually use `makestep 1.0 3`, which allows a step in the first 3 updates only.

**Poll interval.** ntpd and chrony usually poll every 64 to 1024 seconds. They poll less often once the clock is stable.

**Accuracy.** Often under a millisecond on a good LAN, and from one to tens of milliseconds over the Internet. It depends on the network.

### NTP packet layout

```text
 0     2     5     8               16              24            31
+-----+-----+-----+---------------+---------------+---------------+
| LI  | VN  |Mode |    Stratum    |     Poll      |   Precision   |
+-----+-----+-----+---------------+---------------+---------------+
|                          Root Delay (32)                        |
+-----------------------------------------------------------------+
|                       Root Dispersion (32)                      |
+-----------------------------------------------------------------+
|                         Reference ID (32)                       |
+-----------------------------------------------------------------+
|                     Reference Timestamp (64)                    |
+-----------------------------------------------------------------+
|                      Origin Timestamp (64)                      |
+-----------------------------------------------------------------+
|                      Receive Timestamp (64)                     |
+-----------------------------------------------------------------+
|                      Transmit Timestamp (64)                    |
+-----------------------------------------------------------------+
```

The header is 48 bytes. LI is the Leap Indicator, VN is the version, and Mode 3 means client and Mode 4 means server. On a stratum 1 server, the Reference ID names the source, such as `GPS`. At stratum 2 and above it is usually the upstream server's IPv4 address.

### NTP configuration

Cisco IOS:

```text
ntp server 10.0.0.60 prefer
ntp server 10.0.0.61
ntp source Loopback0
```

Linux with chrony (`/etc/chrony/chrony.conf` on Debian and Ubuntu, `/etc/chrony.conf` on RHEL):

```bash
server 10.0.0.60 iburst
server 10.0.0.61 iburst
makestep 1.0 3
```

```bash
# Linux: check status
chronyc sources

# Output
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 10.0.0.60                     2   6   377    23    -85us[ -102us] +/-   12ms
^+ 10.0.0.61                     2   6   377    41   +210us[ +193us] +/-   18ms
```

`^*` is the source in use. `Reach` is an octal record of the last 8 tries, so 377 means all 8 worked.

Windows:

```powershell
# Windows: set NTP servers (0x8 means client mode)
w32tm /config /manualpeerlist:"10.0.0.60,0x8 10.0.0.61,0x8" /syncfromflags:manual /update
Restart-Service w32time
w32tm /query /status
```

```text
Leap Indicator: 0(no warning)
Stratum: 3 (secondary reference - syncd by (S)NTP)
Precision: -23 (119.209ns per tick)
Root Delay: 0.0012041s
Root Dispersion: 0.0512000s
ReferenceId: 0x0A00003C (source IP:  10.0.0.60)
Last Successful Sync Time: 10/30/2026 9:12:44 AM
Source: 10.0.0.60
Poll Interval: 10 (1024s)
```

PCs joined to an Active Directory domain normally get time from the domain, not from manual peers.

### PTP

PTP means Precision Time Protocol, defined by IEEE 1588. The versions in use are 1588-2008 (PTPv2) and 1588-2019. It is built for accuracy of under a microsecond (a millionth of a second), and tens of nanoseconds on good hardware.

Why NTP cannot do this: software stamps the time, and packets wait in switch queues. Both add unpredictable delay.

PTP fixes this in two ways:

1. **Hardware timestamps.** The network card stamps the packet as it passes the wire, not when software gets round to it.
2. **Network help.** PTP-aware switches measure or remove their own delay.

Roles:

- **Grandmaster:** the best clock in the network, usually fed by GPS.
- **Ordinary clock:** a single endpoint that follows or provides time.
- **Boundary clock:** a switch that follows upstream on one port and acts as a time source on the others.
- **Transparent clock:** a switch that measures how long each PTP packet stayed inside it, and writes that into the packet.

The **BMCA** (Best Master Clock Algorithm) elects the grandmaster automatically, using values carried in Announce messages.

Newer IEEE text uses "timeTransmitter" and "timeReceiver" instead of "master" and "slave". Many tools still print the old words.

```text
 Master                                   Slave
   | t1  ---------- Sync ---------------->  | t2
   |     ---------- Follow_Up (holds t1) -> |
   |                                        |
   | t4  <--------- Delay_Req ------------  | t3
   |     ---------- Delay_Resp (holds t4) > |
```

```text
offset = ((t2 - t1) - (t4 - t3)) / 2
delay  = ((t2 - t1) + (t4 - t3)) / 2
```

Follow_Up carries the exact t1 after it has been measured. This is called two-step mode. In one-step mode, hardware writes t1 into Sync as it leaves.

The analogy: NTP is asking a friend the time by phone. PTP is a relay race, where each runner hands over a stopwatch and records exactly how long they held it. Where it stops: PTP still assumes equal delay in both directions, so an unequal fibre path still creates an error.

PTP runs over UDP 319 (timed event messages) and 320 (general messages), or directly on Ethernet with EtherType 0x88F7. Time in PTP is normally TAI (atomic time, with no leap seconds), not UTC.

Linux with linuxptp:

```bash
# Linux: does this network card support hardware timestamps?
ethtool -T eth0

# Linux: run PTP on eth0, log to the screen
sudo ptp4l -i eth0 -m

# Output (offset and delay in nanoseconds, s2 means locked)
ptp4l[5374.018]: master offset        -17 s2 freq  -13205 path delay      2310

# Linux: copy card time to the system clock
sudo phc2sys -s eth0 -w -m
```

Where PTP is used: mobile networks (5G time-division sites need about 1.5 microseconds), stock exchanges (European MiFID II rules require 100 microseconds for high-frequency trading), TV production, power grids and factories.

| Point | NTP | PTP |
|---|---|---|
| Typical accuracy | Milliseconds | Sub-microsecond |
| Timestamps | Usually software | Hardware |
| Network help needed | No | Best with PTP-aware switches |
| Cost | Very low | Special cards and switches |
| Port | UDP 123 | UDP 319, 320, or Ethernet |

### Flow data: NetFlow, sFlow and IPFIX

SNMP says a link is 90 percent full. It cannot say why. Flow data can.

A **flow** is a set of packets that share the same key. Classic NetFlow uses seven fields: source IP, destination IP, source port, destination port, IP protocol, ToS (the priority byte) and input interface.

Analogy: a flow record is a phone bill. It lists who called whom, when, for how long. It does not record the conversation. That is where the analogy holds well. It stops at this point: a phone call has one clear start and end, but a router has to guess when a flow ended, using timers or TCP flags.

```text
           (1) packets pass through
 hosts ----------> [ router + flow cache ] ----------> Internet
                           |
                           | (2) flow ends or timer expires
                           | (3) records exported over UDP
                           v
                   [ collector 10.0.0.70 ]
                           |
                           v
                   [ reports: top talkers ]
```

Steps for NetFlow and IPFIX:

1. A packet arrives. The device reads the key fields.
2. If there is no match in the flow cache, it creates an entry. If there is, it adds to the packet and byte counters.
3. The record is exported when the flow ends, or when a timer fires. Classic Cisco IOS defaults are 30 minutes active and 15 seconds inactive. Many engineers set the active timer to 1 minute, so long flows appear in reports quickly.
4. The collector stores and summarises records.

**Versions:**

- **NetFlow v5:** old Cisco format, fixed fields, IPv4 only.
- **NetFlow v9:** template based (RFC 3954). The device first sends a template that says what each field is. It supports IPv6 and MPLS.
- **IPFIX:** the IETF standard built from v9 (RFC 7011, information model in RFC 7012). It uses version number 10 in the header, so it is sometimes called "NetFlow v10". It adds more fields and vendor-defined fields.

**sFlow** works differently. It does not keep a flow cache. It copies 1 in every N packets (for example 1 in 1024), often the first 128 bytes, and sends them at once on UDP 6343. It also sends interface counters on a timer. The current version is sFlow v5, published by sFlow.org.

Analogy: NetFlow counts every bottle leaving a factory. sFlow picks one bottle in a thousand and inspects it. Where it stops: the sample is random, so small flows can be missed entirely, and a short burst may not appear.

| Point | NetFlow v9 and IPFIX | sFlow |
|---|---|---|
| Method | Cache of flows | Packet sampling |
| Accuracy | Exact, if unsampled | Statistical estimate |
| Device load | Higher at high speed | Low, often in hardware |
| Sees Layer 2 | Depends on template | Yes, in the packet header |
| Best for | Billing, security, routers | Fast data centre switches |

Other vendors have their own names, such as J-Flow on Juniper. Sampled NetFlow also exists, which reduces load but gives up exact numbers.

Cisco IOS XE, Flexible NetFlow. Syntax differs by platform:

```text
flow record REC-V4
 match ipv4 source address
 match ipv4 destination address
 match ipv4 protocol
 match transport source-port
 match transport destination-port
 match interface input
 collect counter bytes long
 collect counter packets long
!
flow exporter EXP-1
 destination 10.0.0.70
 source Loopback0
 transport udp 2055
 export-protocol ipfix
!
flow monitor MON-1
 record REC-V4
 exporter EXP-1
 cache timeout active 60
!
interface GigabitEthernet0/1
 ip flow monitor MON-1 input
```

### LLDP and CDP

Both protocols let a device tell its direct neighbours what it is. They work at Layer 2, so they need no IP address. They are one-hop only. Switches consume these frames and do not forward them.

They only announce. There is no request and no reply. Each device sends a frame on a timer and stores what it hears.

| Point | LLDP | CDP |
|---|---|---|
| Owner | Open standard, IEEE 802.1AB | Cisco proprietary |
| Frame type | EtherType 0x88CC | LLC/SNAP, protocol ID 0x2000 |
| Destination MAC | 01:80:C2:00:00:0E (usual) | 01:00:0C:CC:CC:CC |
| Default send timer | 30 s | 60 s |
| Default hold time | 120 s | 180 s |
| Default on Cisco IOS | Often off | On |

The timers above are the common defaults. LLDP's hold time is the send timer times a multiplier of 4. Vendors can change these values.

**LLDP frame layout.** LLDP data is a list of TLVs (Type, Length, Value):

```text
+---------+---------+--------+---------+-------+-----+-------+
| Dst MAC | Src MAC | 0x88CC | Chassis | Port  | TTL | Other |
|         |         |        | ID TLV  | ID    | TLV | TLVs  |
+---------+---------+--------+---------+-------+-----+-------+

One TLV:
+----------------+------------------+---------------------+
| Type (7 bits)  | Length (9 bits)  | Value (0-511 bytes) |
+----------------+------------------+---------------------+
```

Chassis ID, Port ID and TTL are required. Optional TLVs include Port Description, System Name, System Description, System Capabilities and Management Address.

**LLDP-MED** (Media Endpoint Discovery) is an extension for IP phones. It can tell the phone its voice VLAN and power needs. CDP can also give a Cisco phone its voice VLAN.

Cisco IOS:

```text
! Turn LLDP on for the whole device
lldp run

! Stop CDP and LLDP on a port facing the outside
interface GigabitEthernet0/48
 no cdp enable
 no lldp transmit
 no lldp receive
```

Linux with the `lldpd` package:

```bash
# Linux: list neighbours
sudo lldpcli show neighbors

# Linux: capture LLDP frames, no IP filter works here
sudo tcpdump -i eth0 -nn -v ether proto 0x88cc

# Linux: capture CDP frames (they have no EtherType)
sudo tcpdump -i eth0 -nn -v ether dst 01:00:0c:cc:cc:cc
```

## Example

**One bad morning at a small site.** All these protocols work together here.

```text
                 [ISP] 203.0.113.1
                   |
                   | Gi0/0 203.0.113.2/30, ifIndex 1
                 [R1]  mgmt 10.0.0.1
                   | Gi0/1
                 [SW1] mgmt 10.0.0.2
                   | Gi0/2
                   | Gi0/24
                 [SW2] mgmt 10.0.0.3
                   |
              backup server 10.20.0.15

 Management subnet 10.0.0.0/24
   .50 SNMP manager    .51 syslog collector
   .60 NTP server      .70 flow collector
```

Users report that the Internet is slow at 09:10. The ISP link is 200 Mbps.

**Step 1: SNMP shows the link is busy.** Read the output byte counter on R1 Gi0/0, then read it again 300 seconds later.

```bash
# Linux: first poll at 09:05:00
snmpget -v3 -l authPriv -u nms-user -a SHA -A 'LabAuth-Pass1' \
  -x AES -X 'LabPriv-Pass1' 10.0.0.1 1.3.6.1.2.1.31.1.1.1.10.1
IF-MIB::ifHCOutOctets.1 = Counter64: 918273645000

# Same command at 09:10:00
IF-MIB::ifHCOutOctets.1 = Counter64: 925023645000
```

Work it out:

```text
bytes  = 925023645000 - 918273645000 = 6,750,000,000
bits   = 6,750,000,000 x 8           = 54,000,000,000
rate   = 54,000,000,000 / 300        = 180,000,000 bps = 180 Mbps
usage  = 180 / 200                   = 90 percent
```

SNMP says the link is 90 percent full. It does not say who is using it.

**Step 2: flow data shows who.** The collector shows the top record for those 5 minutes:

| Field | Value |
|---|---|
| Source | 10.20.0.15:51544 |
| Destination | 198.51.100.80:443 |
| Protocol | TCP (6) |
| Input interface | Gi0/1 |
| Packets | 3,100,000 |
| Bytes | 4,500,000,000 |

The rate is 4,500,000,000 x 8 / 300 = 120 Mbps. The backup server alone uses two thirds of the traffic. The fix is to move the backup job out of work hours, or limit it with [QoS](../10-enterprise-and-datacenter/03-quality-of-service.md).

**Step 3: an alert arrives.** At 09:14:03 the manager receives a trap and the collector receives syslog.

```text
Trap received by snmptrapd on 10.0.0.50, wrapped here for reading:

2026-10-30 09:14:03 sw1.example.com
[UDP: [10.0.0.2]:57144->[10.0.0.50]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (123456789) 14 days, 6:56:07.89
SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkDown
IF-MIB::ifIndex.2 = INTEGER: 2
IF-MIB::ifOperStatus.2 = INTEGER: down(2)
```

```text
Syslog on 10.0.0.51:

Oct 30 09:14:03.482 UTC: %LINK-3-UPDOWN: Interface GigabitEthernet0/2, changed state to down
Oct 30 09:14:04.483 UTC: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/2, changed state to down
```

Read `%LINK-3-UPDOWN`: `LINK` is the Cisco component, `3` is the severity (Error), and `UPDOWN` is the message name.

**Step 4: LLDP shows what is on that port.** The port came back after 40 seconds, so the neighbour entry has returned.

```text
SW1# show lldp neighbors
Capability codes:
    (R) Router, (B) Bridge, (T) Telephone, (C) DOCSIS Cable Device
    (W) WLAN Access Point, (P) Repeater, (S) Station, (O) Other

Device ID           Local Intf     Hold-time  Capability      Port ID
SW2                 Gi0/2          120        B               Gi0/24

Total entries displayed: 1
```

No need to walk to the rack. Port Gi0/2 connects to SW2 port Gi0/24. While a link is down, its entry is gone, so keep saved copies of your neighbour tables for outages that last.

**Step 5: time gets in the way.** The team reads SW2's logs and sees the matching event at 09:21:11, seven minutes after SW1. It looks like two different events.

```text
SW2# show ntp status
Clock is unsynchronized, stratum 16, no reference clock
```

SW2 was never given an NTP server. Stratum 16 confirms it. The fix:

```text
SW2(config)# ntp server 10.0.0.60
```

A few minutes later:

```text
SW2# show ntp status
Clock is synchronized, stratum 3, reference is 10.0.0.60
SW2# show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.0.0.60       192.0.2.123      2     41     64   377  1.204   0.318  0.512
 * sys.peer, # selected, + candidate, - outlyer, x falseticker, ~ configured
```

The five tools each gave one piece of the answer: load (SNMP), cause of the load (flow), event (trap and syslog), location (LLDP), and trust in the timeline (NTP).

## Going Deeper

### SNMP details that catch people out

**Counter wrap.** A Counter32 goes back to zero after 2^32 (about 4.29 billion). In bytes, that is:

| Link speed | Time to wrap at full load |
|---|---|
| 100 Mbps | about 344 seconds |
| 1 Gbps | about 34 seconds |
| 10 Gbps | about 3.4 seconds |

A 5-minute poll can miss whole wraps on fast links, so use the 64-bit `ifHC` counters. They need v2c or v3, because v1 has no Counter64 type.

**Device restarts.** Counters reset on reboot. If `sysUpTime` is smaller than at the last poll, throw that sample away. `sysUpTime` itself wraps at about 497 days, since it is a 32-bit count of hundredths of a second.

**SNMPv3 discovery.** Every v3 agent has an **engine ID**, a unique identity. The manager first learns the agent's engine ID, boot count and time through a Report message. Keys are tied to the engine ID. Messages outside a 150-second time window are rejected. This stops old captured messages from being replayed.

**Traps in v3.** For a trap, the sender is the authoritative engine, so the manager needs that device's engine ID and user details. For an inform, the receiver is authoritative.

**Trap content.** A v2c or v3 notification begins with `sysUpTime.0` and `snmpTrapOID.0`. The second one says which event happened. For example, `1.3.6.1.6.3.1.1.5.3` is linkDown and `.4` is linkUp.

**Security.** SNMP has a poor history. Treat it as sensitive:

- Use v3 authPriv where possible.
- Allow access only from manager addresses, using ACLs.
- Avoid write access. A device with write access can often be reconfigured through SNMP.
- Do not expose UDP 161 to the Internet. Open agents can also be abused for traffic amplification, because a GetBulk reply is far larger than its request.

### Syslog at scale

- UDP syslog can be dropped silently during a burst, which is exactly when you need it most.
- A larger site may use relays: devices send to a nearby relay, and the relay sends to the main store. RFC 5424 describes this model.
- Long-term stores are often a [SIEM](../09-security/06-ids-and-ips.md) (Security Information and Event Management) system.
- Use `logging source-interface` so that a device always appears with the same IP. Otherwise the source address depends on the exit interface.
- Log in UTC. It removes daylight saving confusion when sites are in different zones.

### NTP in production

- **Use at least four sources.** RFC 8633 (best practice) recommends four or more. With two servers that disagree, you cannot know which is wrong. With enough sources, the client can vote out a bad clock, which is called a **falseticker**.
- **Internal hierarchy.** Run two or more internal servers that use outside sources or GPS. Point all devices at the internal ones.
- **Security.** Classic NTP has no encryption. Symmetric keys exist, but they are hard to manage at scale. **NTS** (Network Time Security, RFC 8915) is the modern answer. A TLS session on TCP 4460 sets up keys, and normal NTP on UDP 123 then carries authentication. chrony supports it: `server time.cloudflare.com iburst nts`.
- **Amplification.** Old ntpd servers answered the `monlist` command with a large reply. Attackers faked the source address to flood victims (CVE-2013-5211). Current versions have removed it, but you should still restrict queries on public servers.
- **SNTP** (Simple NTP) is a lighter client mode with less filtering. It suits small devices that have no clients of their own.
- **Leap seconds.** The Leap Indicator warns of an extra second. Some large operators "smear" it across many hours instead. The last leap second was at the end of 2016, and the international standards body decided in 2022 to stop adding them by 2035.
- **Era rollover.** NTP's 32-bit seconds field starts from 1900 and wraps in 2036. The protocol handles this with era numbers, but old software may not.

### PTP details

- **Profiles** set rules for an industry: default profile, telecom (ITU-T G.8275.1 and G.8275.2), power (IEEE C37.238), broadcast (SMPTE ST 2059) and gPTP (IEEE 802.1AS) for time-sensitive networks.
- **Domains** are numbers that keep separate PTP systems apart on the same network.
- **Delay mechanisms.** End-to-end uses Delay_Req and Delay_Resp between the slave and master. Peer-to-peer uses Pdelay messages on each link.
- **Multicast.** Over IPv4, PTP commonly uses 224.0.1.129, and peer delay uses 224.0.0.107.
- **Asymmetry.** One fibre direction being longer than the other creates an offset of half the difference. It cannot be measured from the protocol alone.

### Flow details

- **Templates over UDP.** A v9 or IPFIX collector cannot decode data until it receives the template. After a collector restart, there may be a gap until the template is sent again.
- **Double counting.** If you enable both ingress and egress on all interfaces, one flow is counted twice. Pick one method and use it everywhere.
- **Loss.** Export over UDP can drop. Sequence numbers in the headers let a collector detect the loss.
- **Sampling error.** For sFlow, accuracy depends on the number of samples of that traffic class. With 10,000 samples, the error is about 2 percent with 95 percent confidence. With 100 samples, it is about 20 percent.
- **Security use.** Flows can reveal scans, data theft and command-and-control traffic. They show who talked, even when the content is encrypted.
- **Privacy.** IP addresses and connection records can be personal data, so check retention rules.

### LLDP and CDP security

Discovery frames are not encrypted or authenticated. They reveal the device model, software version, management IP, VLAN and port. An attacker on an access port can read all of this. Disable them on ports that face users or outside networks.

The 2020 "CDPwn" disclosure showed serious bugs in the CDP code of Cisco devices, some of which allowed remote code execution from the same Layer 2 network. Keep the feature limited and the software patched. Also see [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).

### Other management tools to recognise

| Tool | Purpose | Note |
|---|---|---|
| RMON | Extra traffic statistics through SNMP | RFC 2819, older |
| NETCONF | Structured config over SSH, TCP 830 | RFC 6241 |
| RESTCONF | Config over HTTP | RFC 8040 |
| YANG | Data modelling language | RFC 7950 |
| gNMI, streaming telemetry | Device pushes data every few seconds | Replaces polling at scale |

Streaming telemetry reverses the SNMP model. The device pushes data at a chosen rate, which suits large and fast networks better than a 5-minute poll. These tools are covered in [Network Automation](../10-enterprise-and-datacenter/12-network-automation.md). How to turn this data into dashboards and alerts is covered in [Network Monitoring and Observability](../12-troubleshooting-and-tools/11-network-monitoring.md).

**Out-of-band management.** All these protocols run in-band by default, which means over the same network they watch. If the network breaks, the alerts break too. An out-of-band network uses separate links, such as a dedicated management port, a console server or a cellular link, so that engineers can still reach devices during an outage.

## Common Mistakes

- **Thinking the agent is on the server and the manager is on the device.** It is the opposite. The agent runs on the managed device.
- **Mixing up 161 and 162.** Queries go to 161 on the agent. Traps go to 162 on the manager.
- **Thinking the MIB stores live data.** A MIB file is a description. The live numbers are inside the agent.
- **Thinking traps are reliable.** They are UDP with no acknowledgement. Use informs for important alerts, and always keep polling too.
- **Using `public` and `private` in production.** This is one of the first things attackers try.
- **Reversing syslog order.** Level 0 is the worst and level 7 is the least serious. Setting `logging trap warning` sends levels 0 to 4, not 4 to 7.
- **Confusing Cisco's message name with the facility.** In `%LINK-3-UPDOWN`, `LINK` is a Cisco component name. The syslog facility on the wire is still `local7` by default.
- **Leaving debug level on.** It can overload the CPU and fill the collector, especially on a production router.
- **Thinking a lower stratum always means better time.** Stratum counts steps from the reference clock. Path quality can matter more.
- **Thinking stratum 0 is on the network.** Stratum 0 devices connect directly to a stratum 1 server and do not speak NTP.
- **Using one NTP server.** If it drifts, every device drifts with it, and nothing notices.
- **Expecting flow data to show payload.** It has counters and headers only. For content, you need a packet capture.
- **Treating sFlow numbers as exact.** They are estimates from samples.
- **Expecting to see LLDP neighbours two switches away.** LLDP and CDP reach direct neighbours only.
- **Filtering CDP by EtherType.** CDP uses LLC/SNAP, so filter by its destination MAC.

## Interview Questions

**Q: What are the roles of the manager, agent, MIB and OID in SNMP?**

A: The manager is the monitoring system that polls devices and receives alerts. The agent runs on each device and answers it. The MIB describes which values exist, and an OID is the unique dotted number of one value, such as 1.3.6.1.2.1.1.3.0 for sysUpTime.

**Q: What is the difference between SNMP v1, v2c and v3?**

A: v1 is the original and uses a clear-text community string. v2c keeps that weak security but adds GetBulk, informs and 64-bit counters. v3 adds users, message authentication and encryption, so it is the right choice today, at the authPriv level.

**Q: What is the difference between polling, a trap and an inform?**

A: In polling, the manager asks on a schedule, using UDP 161. A trap is an unsolicited alert from the agent to UDP 162 with no acknowledgement. An inform is the same kind of alert, but the receiver acknowledges it and the agent retries, so it is more reliable.

**Q: Why should you use ifHCInOctets instead of ifInOctets?**

A: ifInOctets is 32 bits and wraps after about 4.29 GB. On a busy 1 Gbps link that takes about 34 seconds, so a 5-minute poll gives wrong results. The ifHC counters are 64 bits and do not wrap in practice.

**Q: How do you work out link usage from SNMP?**

A: Poll the byte counter twice, take the difference, multiply by 8 to get bits, and divide by the seconds between polls. Then divide by the link speed to get a percentage.

**Q: Name the syslog severity levels in order.**

A: 0 Emergency, 1 Alert, 2 Critical, 3 Error, 4 Warning, 5 Notice, 6 Informational, 7 Debug. Lower numbers are more serious. Setting a level includes that level and everything more serious.

**Q: What port does syslog use, and what are its weaknesses?**

A: Classic syslog uses UDP 514. It can lose messages, and it has no encryption or sender check. For secure delivery, use syslog over TLS on TCP 6514 from RFC 5425.

**Q: What is NTP stratum, and why does accurate time matter?**

A: Stratum is the number of steps from a reference clock. Stratum 1 connects directly to a source such as GPS, and stratum 16 means unsynchronised. Time matters because log correlation, TLS certificate checks and Kerberos logins all depend on it.

**Q: How does NTP calculate clock offset?**

A: It uses four timestamps: client send, server receive, server send and client receive. Offset is ((T2 - T1) + (T3 - T4)) / 2. It assumes the path takes equal time each way, so an unequal path adds error.

**Q: When would you use PTP instead of NTP?**

A: When you need accuracy under a microsecond, such as in 5G networks, trading systems or broadcast. PTP uses hardware timestamps and PTP-aware switches, so it costs more. For most IT systems, NTP is enough.

**Q: What is the difference between NetFlow, sFlow and IPFIX?**

A: NetFlow is Cisco's method of tracking flows in a cache and exporting records. IPFIX is the IETF standard based on NetFlow v9, with templates. sFlow samples 1 in N packets and sends them at once, so it is light on fast switches but gives estimates, not exact counts.

**Q: What is the difference between LLDP and CDP?**

A: Both are one-hop Layer 2 neighbour discovery protocols. LLDP is an open IEEE standard (802.1AB), with EtherType 0x88CC and defaults of 30 s and 120 s. CDP is Cisco proprietary, with defaults of 60 s and 180 s. Both should be off on untrusted ports, because they reveal device details.

## Quick Recap

- SNMP: the manager polls agents on UDP 161, and agents send traps or informs to UDP 162. The MIB is the description, and an OID names one value.
- Use SNMPv3 authPriv and 64-bit `ifHC` counters. Community strings in v1 and v2c are sent in clear text.
- Syslog severity runs from 0 Emergency to 7 Debug. Classic syslog uses UDP 514, and TLS uses TCP 6514.
- NTP uses UDP 123. Stratum counts steps from a reference clock, stratum 16 means unsynchronised, and four sources are recommended.
- PTP (IEEE 1588) uses hardware timestamps for sub-microsecond accuracy, on UDP 319 and 320 or EtherType 0x88F7.
- NetFlow and IPFIX track flows in a cache, and sFlow samples packets. All show who talked to whom, not what they said.
- LLDP (IEEE, 0x88CC) and CDP (Cisco) show direct neighbours at Layer 2, and they should be off on untrusted ports.
