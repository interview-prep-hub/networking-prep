# File Sharing and Storage Protocols

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md), [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md)

## In One Line

Storage protocols let a computer use disks on another machine, as shared files (SMB, NFS), as raw disks (iSCSI, Fibre Channel, FCoE), or as objects over HTTP (S3-style APIs).

## Why It Matters

A disk inside one server helps only that server. If the server dies, the data is stuck inside it. If ten servers need the same data, you must copy it ten times.

Network storage solves this. One storage system serves many machines. You can back it up, grow it, and protect it in one place. A failed server can be replaced, and the new server finds its data still there.

Interviewers ask about this topic for four reasons:

- It tests whether you know what each layer does. "Who owns the file system?" separates people who memorised terms from people who understand them.
- SMB on port 445 is a classic security topic, because of the WannaCry worm in 2017.
- Firewall problems with NFS and SMB are common real tickets.
- Cloud and system design questions ask you to choose between block, file and object storage.

## Simple Explanation

Think of three ways to keep your things outside your home.

**Block storage is an empty rented warehouse.** You get numbered shelves and nothing else. The owner does not know what is on them. You decide how to organise everything, and you keep the index. To get a box you say, "Bring me shelf 52,400."

**File storage is a library with a librarian.** You ask by name: "Please give me the budget file from the finance folder." The librarian finds it, checks that you may read it, and stops two people from changing it at once.

**Object storage is a coat check.** You hand in a bag and get a ticket. The ticket is the only way to find it. There are no rooms or shelves for you to see. If you want to change something in the bag, you usually swap the whole bag.

**Where the analogy stops:** a real warehouse has one customer at a time, but a block storage system serves many hosts, each with its own private set of "shelves". Also, a coat check holds a few hundred bags, while object stores hold billions of objects.

Now two words that people mix up:

- **NAS (Network Attached Storage)** is the library. It is a device that gives out files.
- **SAN (Storage Area Network)** is a private road to the warehouses. It is a network built to carry disk traffic.

## How It Works

### 1. Block, file and object

The main difference is what the client asks for.

```text
 Block :  "Read 8 blocks starting at block 52,400 on LUN 3"
 File  :  "Open \finance\budget.xlsx, read 64 KB at offset 0"
 Object:  "GET bucket reports, key 2026/budget.xlsx"
```

A **block** is a fixed-size piece of disk, often 512 bytes or 4 KB. A **file system** (NTFS, ext4, XFS) is the software that turns blocks into named files and folders. A **LUN (Logical Unit Number)** is the number of a virtual disk on a storage system. An **object** is a whole piece of data stored with a key (its name) and metadata (extra information about it).

| | Block | File | Object |
|---|---|---|---|
| Address | LUN + block number | Folder path | Bucket + key |
| Who owns the file system | The client | The server | No normal file system, flat names |
| Change a small part | Yes, one block | Yes, a byte range | Usually replace the whole object |
| Protocols | iSCSI, FC, FCoE, NVMe-oF | SMB, NFS | HTTP(S) APIs such as S3 |
| Sharing | One host, unless a cluster file system is used | Many clients | Very many clients |
| Typical latency | Lowest | Medium | Highest |
| Good for | Databases, VM disks, boot disks | Team shares, home folders | Backups, media, logs, data lakes |
| Cloud example | AWS EBS, Azure Managed Disks | AWS EFS, Azure Files | Amazon S3, Azure Blob, Google Cloud Storage |

**Fuller truth:** a "/" in an object key looks like a folder, but in plain S3-style storage it is only part of the name. Consoles draw folders to help you. Some newer services add real hierarchy or append operations, but "replace the whole object" is still the normal model.

Object storage has no special network protocol. It is normal HTTPS, usually TCP 443, so it works through firewalls and across the internet. See [HTTP Basics](05-http-basics.md).

### 2. NAS, SAN and DAS

**DAS (Direct Attached Storage)** is a disk connected straight to one server, with no network. The other two options use a network.

The key test for NAS vs SAN is simple: **where does the file system live?**

```text
       NAS (file access)              SAN (block access)
 +--------------------------+   +--------------------------+
 | Application              |   | Application              |
 | OS + SMB/NFS client      |   | OS + file system         |
 +------------+-------------+   | (NTFS, XFS, VMFS)        |
              |                 | iSCSI or FC driver       |
   "read budget.xlsx"           +------------+-------------+
   (file request on LAN)                     |
              |                 "read block 52,400"
 +------------v-------------+   (block request on SAN)
 | NAS device               |                |
 | File system lives HERE   |   +------------v-------------+
 | Disks                    |   | Storage array            |
 +--------------------------+   | Knows blocks, not files  |
                                | Disks                    |
                                +--------------------------+
```

| | NAS | SAN |
|---|---|---|
| Client sees | A shared folder | A local-looking disk |
| Unit of access | File | Block |
| File system on | NAS device | Each host |
| Protocols | SMB, NFS | iSCSI, FC, FCoE, NVMe-oF |
| Network | Usually the normal IP LAN | Dedicated FC fabric or separated Ethernet |
| Sharing | Built in | Needs a cluster file system |
| Typical use | User files, shared content | Databases, virtual machine hosts |

**Fuller truth:** the line is blurred today. Many "unified" storage arrays speak SMB, NFS and iSCSI at once. Hyperconverged systems put the disks inside the compute servers and share them across the cluster. Even so, the question "who owns the file system?" still gives the right answer.

### 3. SMB

**SMB (Server Message Block)** is the file sharing protocol of Windows. It also shares printers and carries some remote management traffic. IBM created it in the 1980s, and Microsoft developed it from there.

It is not only for Windows. **Samba** is open-source software that gives Linux and Unix an SMB server and client. macOS also uses SMB as its main file sharing protocol.

#### Key terms

- **Share:** a folder made available on the network.
- **UNC path (Universal Naming Convention):** the address of a share, such as `\\fs1.example.com\Finance`.
- **Dialect:** a version of SMB, chosen when the connection starts.
- **Administrative shares:** hidden shares that Windows creates, such as `C$`, `ADMIN$` and `IPC$`. The `$` hides them from normal browsing.

#### Ports and history

The modern port is **TCP 445**. Old systems ran SMB on top of **NetBIOS**, an older naming and session system from small LANs.

| Port | Protocol | Use |
|---|---|---|
| TCP 445 | SMB directly over TCP | Modern SMB (Windows 2000 and later), called "direct hosting" |
| TCP 139 | NetBIOS session service | Old SMB transport |
| UDP 137 | NetBIOS name service | Old name lookup |
| UDP 138 | NetBIOS datagram service | Old broadcasts and browsing |
| UDP 443 | SMB over QUIC | Newer option for access over untrusted networks |

Disabling NetBIOS does not stop SMB. SMB keeps working on 445, and names come from [DNS](01-dns-basics.md).

#### SMB versions

| Dialect | First in | Main additions |
|---|---|---|
| SMB1 / CIFS | Windows NT era | Chatty, no modern security, deprecated |
| SMB 2.0.2 | Vista SP1, Server 2008 | Cut commands from over 100 to 19, compounding, larger reads and writes |
| SMB 2.1 | Windows 7, Server 2008 R2 | Leases, large MTU support |
| SMB 3.0 | Windows 8, Server 2012 | Encryption, Multichannel, SMB Direct, transparent failover |
| SMB 3.0.2 | Windows 8.1, Server 2012 R2 | Refinements to 3.0 |
| SMB 3.1.1 | Windows 10, Server 2016 | Pre-authentication integrity, AES-128-GCM |

**CIFS (Common Internet File System)** is Microsoft's 1990s name for its SMB1 dialect. People still say "CIFS share", and Linux still uses `mount -t cifs`, but that mount normally speaks SMB 3.

Windows 11 and Server 2022 kept dialect 3.1.1 and added features to it, such as AES-256 encryption and SMB compression. SMB over QUIC first came to Server 2022 Datacenter: Azure Edition and then more widely with Windows Server 2025.

SMB1 is not installed by default on fresh Windows 10 version 1709 and later. Treat it as something to remove, not something to manage.

#### A session, step by step

```text
 Client                                      Server
   |--- TCP handshake to port 445 ------------->|
   |--- NEGOTIATE: "I speak 2.1, 3.0, 3.1.1" -->|
   |<-- NEGOTIATE: "We will use 3.1.1" ---------|
   |--- SESSION_SETUP: prove who I am --------->|
   |<-- SESSION_SETUP: accepted ----------------|
   |--- TREE_CONNECT: \\fs1\Finance ----------->|
   |<-- TREE_CONNECT: tree id 1 ----------------|
   |--- CREATE: open budget.xlsx -------------->|
   |<-- CREATE: file handle --------------------|
   |--- READ / WRITE -------------------------->|
   |<-- data -----------------------------------|
   |--- CLOSE, TREE_DISCONNECT, LOGOFF -------->|
```

1. **Negotiate:** both sides pick the highest dialect they share.
2. **Session setup:** the user proves their identity, normally with Kerberos in a domain, or NTLM as a fallback. See [Directory and Authentication Protocols](12-directory-and-authentication-protocols.md).
3. **Tree connect:** the client attaches to one share.
4. **Create:** despite the name, this opens an existing file too.
5. **Read and write:** data moves.
6. **Close:** the client releases the file and the share.

Connecting by IP address usually forces NTLM instead of Kerberos, because Kerberos tickets are normally tied to a server name.

#### SMB security

- **Signing** adds a cryptographic check to each message. It stops tampering but does not hide data.
- **Encryption** (SMB 3.0 and later) hides data on the wire. You can turn it on per share or for the whole server.
- **Pre-authentication integrity** (SMB 3.1.1) uses SHA-512 to protect the negotiation, so an attacker cannot force a weaker setup.

The SMB client in Windows 11 24H2 and Windows Server 2025 requires signing by default. Server-side defaults vary by edition and role, so check the actual system.

**Never expose port 445 to the internet.** In 2017 the EternalBlue exploit (CVE-2017-0144, patched by MS17-010) attacked SMB1. The WannaCry and NotPetya malware then spread through it at great speed. Many ISPs block 445 outbound, which is why a home user often cannot reach a cloud file share. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

### 4. NFS

**NFS (Network File System)** is the classic file sharing protocol of Unix and Linux. Sun Microsystems released it in 1984. Unlike SMB, it is an open IETF standard, while SMB is documented by Microsoft.

NFS is built on **RPC (Remote Procedure Call)**, a way to call a function on another machine as if it were local. The main port is **TCP 2049**.

A server **exports** a folder, which is the NFS word for sharing it. Clients **mount** the export, which attaches it to a local folder. Once mounted, it looks like a normal directory.

The server gives the client a **file handle**, an opaque ID for a file. If the server loses track of that ID, the client sees the error "Stale file handle".

#### NFS versions

| Version | Standard | Transport | Main points |
|---|---|---|---|
| NFSv2 | RFC 1094 (1989) | Mainly UDP | 2 GB file limit, obsolete |
| NFSv3 | RFC 1813 (1995) | UDP or TCP | 64-bit sizes, safe async writes with COMMIT, stateless server |
| NFSv4.0 | RFC 7530 (first in 2000) | TCP | Stateful, one port, built-in locks, ACLs, compound requests |
| NFSv4.1 | RFC 8881 (first in 2010) | TCP | Sessions, pNFS |
| NFSv4.2 | RFC 7862 (2016) | TCP | Server-side copy, sparse files |

**Stateless** means the server keeps no record of which clients have which files open. A stateless server is simple and recovers easily after a crash. **Stateful** means the server tracks opens and locks. That needs more work, but it allows safe locking and better caching.

Modern Linux clients try the highest version the server offers, so you usually get 4.2 without asking.

#### NFSv3 needs many helpers

This is the most common NFS interview point.

```text
 NFSv3 client                         NFSv3 server
   |--- "Where is mountd?" ---------> rpcbind, port 111
   |<-- "Port 20048" ---------------- rpcbind
   |--- "Mount /srv/projects" ------> mountd, port 20048
   |<-- file handle ----------------- mountd
   |--- READ, WRITE ----------------> nfsd, port 2049
   |--- lock request ---------------> lockd, dynamic port
```

**rpcbind** is a directory of RPC services on port 111. **mountd** handles mount requests. **lockd** and **statd** handle file locks and crash recovery. Several of these use dynamic ports by default, which makes firewall rules painful.

The port 20048 above is common on Linux, but do not rely on it. If you must pass NFSv3 through a firewall, pin these services to fixed ports in the NFS configuration.

**NFSv4 solves this.** Everything goes to TCP 2049. The client does not need to reach rpcbind, mountd or lockd. So a firewall needs one rule.

#### NFS security

The default mode is **AUTH_SYS**. The client simply states the user's numeric ID (UID) and group ID (GID), and the server believes it. A person with root on a client can pretend to be any user. This is weak.

- **root_squash** maps the client's root user to an unprivileged user on the server. It is the Linux default for exports. It does not protect normal users.
- **Kerberos** gives real security through RPCSEC_GSS. The mount options are `sec=krb5` (authentication), `sec=krb5i` (adds integrity) and `sec=krb5p` (adds encryption).
- **RPC over TLS** (RFC 9289) is a newer way to encrypt NFS traffic. Support depends on the kernel and distribution.

Limit exports by client network and keep NFS on trusted networks.

#### SMB vs NFS

| | SMB | NFS |
|---|---|---|
| Home | Windows | Unix and Linux |
| Main port | TCP 445 | TCP 2049 |
| Standard | Microsoft published specs | IETF RFCs |
| Identity | User logs in to the session | Numeric UID and GID, or Kerberos |
| Connection | Per-user sessions | Usually one mount for the whole machine |
| Encryption | Built in since SMB 3.0 | Kerberos `krb5p` or RPC over TLS |

### 5. SCSI: the language under the SAN

**SCSI (Small Computer System Interface)** is an old standard for talking to disks. It defines commands like "read these blocks". An **initiator** sends commands, and a **target** carries them out and answers.

iSCSI and Fibre Channel are mostly ways to carry those SCSI commands over a network, so the OS thinks a local disk is attached.

### 6. iSCSI

**iSCSI (Internet SCSI)** carries SCSI commands inside TCP/IP. It is defined in RFC 7143 and uses **TCP 3260**. It runs on normal Ethernet switches and normal network cards, so it is the cheap way to build a SAN.

#### Key terms

- **Initiator:** the client, such as a database server. It can be software or a dedicated card.
- **Target:** the storage side that holds the LUNs.
- **Portal:** the IP address and port of a target, such as `10.10.40.10:3260`.
- **IQN (iSCSI Qualified Name):** a unique name, such as `iqn.2026-01.com.example:array1.db`. The format is `iqn.`, a year and month, a reversed domain name, then any label.
- **PDU (Protocol Data Unit):** one iSCSI message. Each begins with a 48-byte header.
- **CHAP (Challenge-Handshake Authentication Protocol):** a login check using a shared secret. It can run one way or both ways (mutual CHAP).

#### Packet layout

```text
 +----------+--------+---------------+--------------+-----------+
 | Ethernet | IPv4   | TCP           | iSCSI header | SCSI data |
 | header   | header | dst port 3260 | 48 bytes     |           |
 +----------+--------+---------------+--------------+-----------+
```

#### Steps

1. **Discovery:** the initiator asks the portal which targets exist, with a SendTargets request. Larger sites may use **iSNS (Internet Storage Name Service)** on TCP 3205, or static config.
2. **TCP connection** to port 3260.
3. **Login:** optional CHAP, then both sides agree on settings.
4. **Full feature phase:** SCSI commands and data flow in PDUs.
5. The OS sees a new blank disk, and you create a file system on it.

CHAP checks identity at login only. It does **not** encrypt data. Protect iSCSI with an isolated network, and use [IPsec](../09-security/08-ipsec.md) if you need encryption.

Good practice for iSCSI:

- Put it on a separate VLAN or separate switches, with no default gateway. See [VLANs](../03-ethernet-and-switching/06-vlans.md).
- Use **jumbo frames** (MTU 9000) if every device on the path supports them. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).
- Use two paths with **multipath I/O (MPIO)**, which gives failover and shared load. Do not use [link aggregation](../03-ethernet-and-switching/10-link-aggregation.md) for this.

### 7. Fibre Channel

**Fibre Channel (FC)** is a separate network technology built only for storage. It has its own switches, cards, addressing and routing. The spelling "Fibre" is deliberate, because it also runs on copper for short distances.

It does not use Ethernet or IP. T11, a committee of the standards body INCITS, develops it. Common speeds today are 16, 32 and 64 Gbit/s, which vendors call Gen 5, Gen 6 and Gen 7. The speed names are rounded labels, not exact line rates.

#### Key terms

- **HBA (Host Bus Adapter):** the FC card in a server.
- **Fabric:** the set of connected FC switches.
- **WWN (World Wide Name):** a fixed 8-byte address, like a MAC address. A **WWPN** names a port, and a **WWNN** names a node (a card or device).
- **FCID:** a 24-bit address the fabric gives a port at login. It has three parts: domain (switch), area, and port. In `0x0A0100`, the domain is 0x0A.
- **Port types:** an **N_Port** is on a node, an **F_Port** is the switch port facing a node, and an **E_Port** links two switches.
- **Zoning:** rules in the fabric that decide which ports may talk.
- **FCP (Fibre Channel Protocol):** the mapping that carries SCSI over FC.

**Analogy:** the WWPN is like a person's passport number, which never changes. The FCID is like a hotel room number, given at check-in and used to deliver things. **Where it stops:** unlike hotel guests, most devices get the same FCID each time, and FC routing uses the domain part only.

#### FC layers

| Layer | Job | Rough match |
|---|---|---|
| FC-4 | Maps protocols such as SCSI (FCP) and NVMe | Application mapping |
| FC-3 | Common services | Rarely discussed |
| FC-2 | Frames, flow control, classes of service | Data link |
| FC-1 | Line encoding | Physical coding |
| FC-0 | Cables, lasers, speeds | Physical |

#### How a server joins the fabric

```text
 HBA (N_Port)        Switch (F_Port)              Array port
   |--- FLOGI ---------->|                            |
   |<-- accept + FCID ---|                            |
   |--- PLOGI to name server ->|                      |
   |--- "Who can I reach?" --->|                      |
   |<-- zoned ports only ------|                      |
   |--- PLOGI ------------------------------------->  |
   |--- PRLI -------------------------------------->  |
   |--- SCSI commands (FCP) ----------------------->  |
```

1. **FLOGI (Fabric Login):** the port logs in to the switch and gets its FCID.
2. **Name server login:** the port registers itself. The fabric keeps this list.
3. **Query:** the port asks what it can see. Zoning filters the answer.
4. **PLOGI (Port Login):** the host logs in to the storage port.
5. **PRLI (Process Login):** the two sides agree to use SCSI.

#### Why FC never drops frames

Ethernet can drop a frame when a switch is full, and TCP sends it again. SCSI recovers from loss very slowly, so FC prevents loss.

It uses **buffer-to-buffer credits**. Each side tells the other how many frames it can hold.

```text
 Sender has 3 credits
   send frame  -> credits 2
   send frame  -> credits 1
   send frame  -> credits 0   sender must stop and wait
   <- R_RDY    -> credits 1   receiver freed one buffer
```

**Analogy:** a car park with a fixed number of tickets. You enter only with a ticket, so no one is ever turned away inside. **Where it stops:** a car park does not make the road behind it slow down, but a slow FC device can do this (see "Slow drain" under Going Deeper).

#### Zoning vs LUN masking

- **Zoning** happens on the FC switch. It controls which ports can see each other.
- **LUN masking** happens on the storage array. It controls which hosts can see which LUN.

Use both. The standard practice is **single-initiator zoning**: one host port and its storage ports per zone. Build two separate fabrics, A and B, so one failure cannot stop all paths.

### 8. FCoE

**FCoE (Fibre Channel over Ethernet)** puts whole FC frames inside Ethernet frames. The aim was to use one set of cards and switches for both LAN and SAN traffic in a data centre.

**FCoE is not IP.** There is no IP header and no TCP. It uses its own EtherType, **0x8906**, so it stays inside one Layer 2 network. A second EtherType, **0x8914**, is used by **FIP (FCoE Initialization Protocol)**, which finds the FC switch and logs in.

The hardware names are:

- **CNA (Converged Network Adapter):** one card that acts as both a network card and an HBA.
- **FCF (FCoE Forwarder):** a switch with FC logic inside it.

#### Lossless Ethernet

FC expects no loss, but normal Ethernet drops frames. So FCoE needs **DCB (Data Center Bridging)**:

| Feature | Standard | Job |
|---|---|---|
| PFC (Priority-based Flow Control) | IEEE 802.1Qbb | Pauses one traffic class, not the whole link |
| ETS (Enhanced Transmission Selection) | IEEE 802.1Qaz | Shares bandwidth between classes |
| DCBX (DCB Exchange) | IEEE 802.1Qaz | Neighbours agree on settings, using LLDP |

See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md) for the idea of traffic classes.

#### FCoE frame layout

| Field | Bytes | Notes |
|---|---|---|
| Destination MAC | 6 | |
| Source MAC | 6 | |
| 802.1Q tag | 4 | FCoE VLAN, usually CoS 3 by convention |
| EtherType | 2 | 0x8906 |
| FCoE header | 14 | Version, reserved bits, start-of-frame |
| FC header | 24 | Same as native FC |
| FC payload | 0-2112 | SCSI commands or data |
| FC CRC | 4 | |
| End of frame + reserved | 4 | |
| Ethernet FCS | 4 | |

A full frame adds up to 2180 bytes, which is larger than the normal 1518-byte maximum for an untagged Ethernet frame. So FCoE links need "baby jumbo" frames of about 2.5 KB.

**Current practice:** FCoE adoption stayed limited. It is usually seen from the server to the first switch, not across many hops. New designs usually choose native FC, iSCSI or NVMe/TCP. You should still know it for interviews.

### 9. Comparing the transports

```text
   iSCSI           FC              FCoE            FCIP
 +----------+   +----------+   +----------+   +----------+
 | SCSI     |   | SCSI     |   | SCSI     |   | SCSI     |
 | iSCSI    |   | FCP      |   | FCP      |   | FCP      |
 | TCP      |   | FC-2     |   | FC-2     |   | FC-2     |
 | IP       |   | FC-1     |   | FCoE     |   | FCIP     |
 | Ethernet |   | FC-0     |   | Lossless |   | TCP      |
 |          |   |          |   | Ethernet |   | IP       |
 |          |   |          |   |          |   | Ethernet |
 +----------+   +----------+   +----------+   +----------+
```

**FCIP (Fibre Channel over IP)** is a tunnel. It links two FC fabrics across an IP network, often between data centres, using TCP 3225 (RFC 3821). Do not confuse it with FCoE.

| | iSCSI | Native FC | FCoE |
|---|---|---|---|
| Runs on | Any IP network | Dedicated FC fabric | Lossless Ethernet |
| Hardware | Normal NIC and switch | HBA and FC switch | CNA and FCoE switch |
| Loss handling | TCP retransmits | Credits prevent loss | PFC prevents loss |
| IP routable | Yes | No | No |
| Cost and skill | Lowest | Highest | High |

#### Ports at a glance

| Protocol | Port or ID |
|---|---|
| SMB | TCP 445 |
| NetBIOS | UDP 137, UDP 138, TCP 139 |
| SMB over QUIC | UDP 443 |
| NFS | TCP 2049 |
| rpcbind (NFSv3) | TCP and UDP 111 |
| iSCSI | TCP 3260 |
| iSNS | TCP 3205 |
| FCIP | TCP 3225 |
| FCoE | EtherType 0x8906, no port |
| FIP | EtherType 0x8914, no port |
| Native FC | Not IP, no port |
| S3-style object storage | HTTPS, usually TCP 443 |

The full list is in [Common Ports Reference](../06-transport-layer/10-common-ports-reference.md).

## Example

A small company has three VLANs. Users are in VLAN 30, file servers in VLAN 20, and storage traffic in VLAN 40. VLAN 40 has no gateway, so it cannot be reached from other subnets.

```text
  VLAN 30 users, 10.10.30.0/24      gateway 10.10.30.1
  +----------------+  +----------------+
  | pc25 (Windows) |  | app1 (Linux)   |
  | 10.10.30.25    |  | 10.10.30.40    |
  +-------+--------+  +-------+--------+
          |                   |
  +-------+-------------------+-----------------+
  |       Layer 3 switch with ACL               |
  +-------+-------------------+-----------+-----+
          |                   |           |
  VLAN 20 servers, 10.10.20.0/24, gateway 10.10.20.1
  +-------+--------+  +-------+-----+  +--+------------+
  | fs1 (SMB)      |  | nas1 (NFS)  |  | db1 (Linux)   |
  | 10.10.20.15    |  | 10.10.20.20 |  | 10.10.20.30   |
  +----------------+  +-------------+  | 10.10.40.21   |
                                       +--+------------+
  VLAN 40 storage, 10.10.40.0/24          |
  MTU 9000, no gateway                    |
                                       +--+------------+
                                       | array1 iSCSI  |
                                       | 10.10.40.10   |
                                       +---------------+
```

### Step 1: Allow only what is needed

```text
! Cisco IOS: applied to traffic entering from VLAN 30
ip access-list extended USERS-TO-SERVERS
 permit tcp 10.10.30.0 0.0.0.255 host 10.10.20.15 eq 445
 permit tcp 10.10.30.0 0.0.0.255 host 10.10.20.20 eq 2049
 deny   ip  10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255
 permit ip any any
!
interface Vlan30
 ip access-group USERS-TO-SERVERS in
```

Only one NFS rule is needed, because the clients use NFSv4. With NFSv3, this list would break mounts.

### Step 2: SMB share

Create the share on fs1, turn off SMB1, and require signing and encryption.

```powershell
# Windows Server (fs1)
New-Item -Path D:\Shares\Finance -ItemType Directory
New-SmbShare -Name Finance -Path D:\Shares\Finance -FullAccess "CORP\Finance-Users"
Set-SmbShare -Name Finance -EncryptData $true -Force
Set-SmbServerConfiguration -EnableSMB1Protocol $false -RequireSecuritySignature $true -Force

Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol, RequireSecuritySignature
```

```text
EnableSMB1Protocol EnableSMB2Protocol RequireSecuritySignature
------------------ ------------------ ------------------------
             False               True                     True
```

`EnableSMB2Protocol` covers SMB 2 and SMB 3. There is no separate switch for SMB 3.

Test and connect from pc25.

```powershell
# Windows client (pc25)
Test-NetConnection fs1.example.com -Port 445
```

```text
ComputerName     : fs1.example.com
RemoteAddress    : 10.10.20.15
RemotePort       : 445
InterfaceAlias   : Ethernet
SourceAddress    : 10.10.30.25
TcpTestSucceeded : True
```

```powershell
# Windows client (pc25)
New-SmbMapping -LocalPath F: -RemotePath \\fs1.example.com\Finance
Get-SmbConnection
```

```text
ServerName       ShareName UserName    Credential  Dialect NumOpens
----------       --------- --------    ----------  ------- --------
fs1.example.com  Finance   CORP\amina  CORP\amina  3.1.1   1
```

The Dialect column shows the negotiated version. The same share from Linux looks like this:

```bash
# Linux client (app1)
sudo mkdir -p /mnt/finance
sudo mount -t cifs //fs1.example.com/Finance /mnt/finance \
  -o username=amina,domain=CORP,vers=3.1.1,seal
ss -tn state established '( dport = :445 )'
```

```text
Recv-Q Send-Q   Local Address:Port    Peer Address:Port
0      0          10.10.30.40:48622    10.10.20.15:445
```

A packet capture of the opening of `budget.xlsx` shows the steps from the session diagram. This is a simplified view of what [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) shows:

```text
 No.  Dir  Info
 1-3  C-S  TCP handshake to port 445
 4    C>S  Negotiate Protocol Request
 5    S>C  Negotiate Protocol Response (dialect 3.1.1)
 6    C>S  Session Setup Request
 7    S>C  Session Setup Response
 8    C>S  Tree Connect Request \\fs1.example.com\Finance
 9    S>C  Tree Connect Response
 10+  C-S  Encrypted SMB3 (file names and data now hidden)
```

Because the share requires encryption, the file names after the tree connect are not visible.

### Step 3: NFSv4 export

```bash
# Linux server (nas1): line in /etc/exports
/srv/projects 10.10.30.0/24(rw,sync,root_squash,no_subtree_check)
```

```bash
# Linux server (nas1): apply and list exports
sudo exportfs -ra
sudo exportfs -v
```

```text
/srv/projects  10.10.30.0/24(sync,wdelay,hide,no_subtree_check,
               sec=sys,rw,secure,root_squash,no_all_squash)
```

```bash
# Linux client (app1)
sudo mkdir -p /mnt/projects
sudo mount -t nfs -o vers=4.2 nas1.example.com:/srv/projects /mnt/projects
nfsstat -m
```

```text
/mnt/projects from nas1.example.com:/srv/projects
 Flags: rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,
 namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,
 clientaddr=10.10.30.40,local_lock=none,addr=10.10.20.20
```

(The long lines in the last two outputs are wrapped to fit the page.)

Check three things here: `vers=4.2` is the version, `hard` means the client retries forever if the server goes away, and `sec=sys` means AUTH_SYS, so this mount trusts the client's UID.

Windows can also mount NFS after you add the Client for NFS feature. In PowerShell, write `mount.exe`, because plain `mount` is an alias for `New-PSDrive`.

```powershell
# Windows client with the Client for NFS feature
mount.exe -o anon \\nas1.example.com\srv\projects Z:
```

### Step 4: iSCSI disk

First prove that jumbo frames work end to end. The ICMP payload is 8972 bytes, because 8972 + 8 (ICMP) + 20 (IPv4) = 9000.

```bash
# Linux host (db1): do not fragment
ping -c 1 -M do -s 8972 10.10.40.10
```

```text
8980 bytes from 10.10.40.10: icmp_seq=1 ttl=64 time=0.21 ms
```

The Windows form is `ping -f -l 8972 10.10.40.10`.

Now discover the target and log in.

```bash
# Linux host (db1)
sudo iscsiadm -m discovery -t sendtargets -p 10.10.40.10:3260
```

```text
10.10.40.10:3260,1 iqn.2026-01.com.example:array1.db
```

```bash
sudo iscsiadm -m node -T iqn.2026-01.com.example:array1.db \
  -p 10.10.40.10:3260 --login
```

```text
Logging in to [iface: default, target: iqn.2026-01.com.example:
array1.db, portal: 10.10.40.10,3260]
Login to [iface: default, target: iqn.2026-01.com.example:array1.db,
portal: 10.10.40.10,3260] successful.
```

(These two output lines are wrapped here to fit the page.)

A new disk appears. It is blank, so this host must create the file system. That is the sign of block storage.

```bash
lsblk /dev/sdb
```

```text
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb    8:16   0  500G  0 disk
```

```bash
sudo mkfs.xfs /dev/sdb
sudo mkdir -p /data
sudo mount -o _netdev /dev/sdb /data
```

`_netdev` tells the system that this disk needs the network. Use the same option in `/etc/fstab`, so the mount waits for the network at boot.

The Windows steps are:

```powershell
# Windows host
Start-Service MSiSCSI
New-IscsiTargetPortal -TargetPortalAddress 10.10.40.10
Connect-IscsiTarget -NodeAddress iqn.2026-01.com.example:array1.db -IsPersistent $true
Get-Disk
```

### Step 5: Object storage, for contrast

There is no mount and no login to a share. It is an API call over HTTPS.

```bash
# Linux, with the AWS CLI
aws s3 cp budget.xlsx s3://example-reports/2026/budget.xlsx
```

```text
upload: ./budget.xlsx to s3://example-reports/2026/budget.xlsx
```

### Step 6: FC zoning, for contrast

This is a Cisco MDS switch running NX-OS. The zone lets one host port reach one array port in VSAN 10.

```text
! Cisco MDS NX-OS
zone name Z_DB1_ARRAY1 vsan 10
  member pwwn 21:00:00:24:ff:4a:10:01
  member pwwn 50:06:01:60:3b:20:11:22
zoneset name ZS_FABRIC_A vsan 10
  member Z_DB1_ARRAY1
zoneset activate name ZS_FABRIC_A vsan 10
```

To check that the ports logged in, use `show flogi database`. Node name column removed to fit:

```text
INTERFACE  VSAN  FCID      PORT NAME
fc1/1      10    0x0a0100  21:00:00:24:ff:4a:10:01
fc1/9      10    0x0a0200  50:06:01:60:3b:20:11:22
```

## Going Deeper

### SMB internals

The SMB 2 and 3 message layout has a fixed 64-byte header. On TCP 445, a 4-byte length field comes before it.

| Field | Bytes | Notes |
|---|---|---|
| ProtocolId | 4 | 0xFE 'S' 'M' 'B' |
| StructureSize | 2 | Always 64 |
| CreditCharge | 2 | Credits this request costs |
| Status / ChannelSequence | 4 | Result in a response |
| Command | 2 | Such as READ or WRITE |
| CreditRequest / CreditResponse | 2 | Credits asked for or granted |
| Flags | 4 | |
| NextCommand | 4 | Offset to the next compounded command |
| MessageId | 8 | |
| Reserved + TreeId, or AsyncId | 8 | Identifies the share or async request |
| SessionId | 8 | |
| Signature | 16 | Message signature |

You can tell the version from the first byte in a capture: SMB1 starts with 0xFF, SMB 2 and 3 with 0xFE, and encrypted SMB 3 messages with 0xFD.

- **Credits** are SMB's flow control. The server grants credits, and each request spends them. This lets it send many requests at once over high-latency links.
- **Compounding** puts several commands in one message, which saves round trips.
- **Oplocks and leases** let a client cache a file locally. If another client wants the file, the server tells the first client to write back its changes. This is why SMB feels fast on a LAN.
- **Multichannel** (SMB 3.0) uses several TCP connections at once, across several NICs if present. It needs no special setup and gives both speed and failover.
- **SMB Direct** (SMB 3.0) uses **RDMA (Remote Direct Memory Access)**, where network cards move data straight between memory with little CPU work.
- **Transparent failover** (SMB 3.0) lets a client move to another cluster node without the application seeing an error, on shares built for it.
- **SMB over QUIC** runs SMB inside TLS 1.3 on UDP 443, which gives a VPN-free option for remote users. See [QUIC](../06-transport-layer/08-quic.md).

### NFSv4 details

- **COMPOUND:** v4 puts several operations in one RPC. A read can be PUTFH + READ + GETATTR in one round trip. v3 needs separate calls.
- **Leases:** a client must renew its state regularly. If it crashes, the server frees its locks after the lease ends. After a server restart, a **grace period** lets clients reclaim locks.
- **Delegations:** the server lets one client cache a file and handle it locally until someone else wants it.
- **ID mapping:** v4 can send names as `user@domain` rather than numbers. If the two sides disagree on the domain, files can show as owned by `nobody`.
- **Sessions (v4.1):** add exactly-once behaviour, so a retried request is not run twice.
- **pNFS (parallel NFS, v4.1):** a metadata server tells clients where data lives, and clients read directly from many data servers at once.
- **nconnect:** a Linux client mount option (kernel 5.3 and later) that opens several TCP connections to the server.

**Hard vs soft mounts:** with `hard`, a process waits forever when the server is down, so `df` or `ls` can hang. With `soft`, the client gives up and returns an error, which can cause silent data loss on writes. Use `hard` for data you care about.

### FC header and fabric details

The FC frame has a 24-byte header, up to 2112 bytes of payload, a 4-byte CRC, and 4-byte start and end markers. The maximum total is 2148 bytes.

```text
 Bits   0        8        16       24       31
       +--------+--------+--------+--------+
 Word0 | R_CTL  | D_ID (destination FCID)  |
       +--------+--------+--------+--------+
 Word1 | CS_CTL | S_ID (source FCID)       |
       +--------+--------+--------+--------+
 Word2 | TYPE   | F_CTL                    |
       +--------+--------+--------+--------+
 Word3 | SEQ_ID | DF_CTL | SEQ_CNT         |
       +--------+--------+--------+--------+
 Word4 | OX_ID           | RX_ID           |
       +--------+--------+--------+--------+
 Word5 | Parameter                         |
       +--------+--------+--------+--------+
```

TYPE 0x08 means SCSI FCP. OX_ID identifies the exchange, which is one complete SCSI operation.

- **FSPF (Fabric Shortest Path First):** the link-state routing protocol between FC switches. It works much like [OSPF](../05-routing/06-ospf.md).
- **VSAN (Virtual SAN):** a Cisco feature that splits one physical fabric into isolated logical fabrics, like a VLAN.
- **NPIV (N_Port ID Virtualization):** lets one physical port hold many FCIDs and WWPNs, so each virtual machine can have its own.
- **Slow drain:** one slow device stops returning credits. Frames back up on the switch, and the delay spreads to other devices that share those links. A lossless network has this risk, while a lossy one would drop frames instead.
- **Long distance:** frames in flight need credits, so longer links need more buffer credits to keep full speed.

### FCoE addressing

During FIP login, the fabric builds the host's MAC address from a fixed prefix and the FCID. This is called an **FPMA (Fabric Provided MAC Address)**. With the default prefix 0E:FC:00 and FCID 0x0A0100, the MAC is `0E:FC:00:0A:01:00`.

### NVMe over Fabrics

**NVMe (Non-Volatile Memory Express)** is the modern command set for flash drives. It has far less overhead than SCSI and supports many parallel queues.

**NVMe-oF (NVMe over Fabrics)** carries NVMe over a network, and is the modern successor to iSCSI and FCP for fast flash storage:

- **NVMe/FC:** uses existing FC fabrics.
- **NVMe/RoCE:** uses RDMA over Ethernet. **RoCEv2** runs on UDP 4791 and wants lossless or carefully tuned Ethernet.
- **NVMe/TCP:** runs on normal TCP networks, commonly TCP 4420 for data and TCP 8009 for discovery.

### Sizing with simple numbers

**IOPS (input/output operations per second)** counts how many reads and writes a system handles. Throughput is IOPS times I/O size.

A 10 Gbit/s link carries 1,250 MB/s raw. After headers, say 1,150 MB/s of usable data. With 8 KB I/O, the link alone limits you to about 140,000 IOPS (1,150,000 KB / 8 KB). Disk speed or latency usually limits you first.

Latency matters more for databases. One synchronous write needs a full round trip. A 1 ms network round trip adds at least 1 ms to each write, so storage should stay close to its hosts.

### Cloud mapping

Cloud services follow the same three types. AWS EFS speaks NFSv4.1, and Azure Files speaks SMB or NFS. Cloud block disks such as AWS EBS attach through the provider's own virtual layer, so you do not configure iSCSI for them.

Tools such as FTP, SFTP and rsync copy files rather than mount them. See [File Transfer Protocols](08-file-transfer-protocols.md).

## Common Mistakes

- **"NAS and SAN are the same thing, spelled differently."** NAS gives files, and SAN gives blocks. Ask who owns the file system.
- **"SAN means Fibre Channel."** A SAN can run on iSCSI over Ethernet.
- **"CIFS is a separate modern protocol."** CIFS is an old SMB1 dialect. The word lives on in product names and in `mount -t cifs`.
- **"Disabling NetBIOS turns off SMB."** SMB runs directly on TCP 445 without it.
- **"Signing encrypts traffic."** Signing proves that data was not changed. Only encryption hides it.
- **"SMB is Windows only."** Samba, Linux, macOS and most NAS devices use it every day.
- **Opening port 2049 alone for NFSv3.** The mount also needs rpcbind, mountd and lock services.
- **Trusting AUTH_SYS.** Anyone with root on a client can claim any UID. `root_squash` protects only against root.
- **Different UIDs on different machines.** User `amina` may be UID 1001 on one host and 1005 on another. She then sees wrong owners or "permission denied".
- **Mounting one LUN on two normal servers.** Each server believes it owns the disk, so caches conflict and the file system is corrupted. Use a cluster file system such as VMFS, GFS2 or Windows Cluster Shared Volumes.
- **Thinking CHAP encrypts iSCSI.** CHAP protects the login only.
- **Using LACP for iSCSI paths.** Use MPIO, which understands storage paths and failover.
- **Enabling jumbo frames on only part of the path.** Small pings work while large transfers hang. Test with a do-not-fragment ping of full size.
- **Thinking FCoE is routable like IP.** It has no IP header. iSCSI and FCIP use IP, but FCoE does not.
- **Thinking "Fibre" means fibre optic only.** The name refers to the technology, and FC also runs on copper for short links.
- **Forgetting that SMB has two permission layers.** Share permissions and NTFS permissions both apply, and the stricter one wins.
- **Treating object storage as a normal disk.** Tools can mount a bucket, but a small edit usually still rewrites the whole object.

## Interview Questions

**Q: What is the difference between SMB on port 445 and port 139?**
A: Port 139 is SMB carried over NetBIOS, the old method. Port 445 is SMB directly over TCP, used by Windows 2000 and later. Modern networks should use 445 and can usually turn NetBIOS off.

**Q: Why should SMB1 be disabled?**
A: SMB1 is old and lacks modern protection. It has no pre-authentication integrity and no encryption. EternalBlue attacked it in 2017, and WannaCry and NotPetya spread that way. Windows no longer installs it by default on fresh systems since Windows 10 version 1709. Use SMB 3.x.

**Q: What is the difference between SMB and CIFS?**
A: CIFS is the 1990s name for one SMB1 dialect. SMB is the whole protocol family, up to SMB 3.1.1. Today, "CIFS" in product names usually just means SMB.

**Q: Compare NFSv3 and NFSv4.**
A: NFSv3 is stateless and depends on helpers such as rpcbind, mountd and the lock manager, some on dynamic ports. NFSv4 is stateful and uses only TCP 2049. It has built-in locking, compound requests and support for strong security with Kerberos. So NFSv4 is much easier through firewalls.

**Q: What does root_squash do, and is it enough?**
A: It maps root on the client to an unprivileged user on the server, so client root cannot act as root on the export. It is not enough, because AUTH_SYS trusts the client's UID. A client root user can still become any normal user. For real security, use Kerberos and limit which networks can mount.

**Q: Explain block, file and object storage, with a use for each.**
A: Block gives a raw disk addressed by block number, and the client owns the file system. It suits databases and VM disks. File gives shared folders through SMB or NFS, which suits team shares. Object stores whole items by key through an HTTP API. It suits backups, media and logs at huge scale.

**Q: What is the difference between SAN and NAS?**
A: NAS is a device that serves files over the LAN with SMB or NFS, and the file system lives on the NAS. A SAN is a network that gives hosts block access through iSCSI or Fibre Channel, and each host has its own file system. The host sees a NAS as a share and a SAN LUN as a local disk.

**Q: When would you choose iSCSI over Fibre Channel?**
A: I would choose iSCSI when cost matters and I already have good Ethernet skills and hardware. It works on standard NICs and switches, and TCP handles loss. I would choose FC for critical workloads that need predictable low latency and lossless delivery on a dedicated fabric, if the budget and skills exist.

**Q: What is the difference between iSCSI, FCoE and FCIP?**
A: iSCSI carries SCSI over TCP/IP with no Fibre Channel at all. FCoE puts FC frames directly into Ethernet frames with no IP, so it needs lossless Ethernet and stays in Layer 2. FCIP tunnels FC traffic through TCP/IP to link two FC fabrics, often across sites.

**Q: Why is Fibre Channel called lossless, and how does it achieve this?**
A: It uses buffer-to-buffer credits. A sender can send a frame only if the receiver has a free buffer, and the receiver returns credits with R_RDY. Frames wait rather than drop. The risk is a slow device, which can back up traffic for other devices.

**Q: What is the difference between zoning and LUN masking?**
A: Zoning is set on the FC switch and controls which ports can see each other. LUN masking is set on the storage array and controls which hosts can see which LUNs. I use both, with single-initiator zones, for defence in depth.

**Q: What happens if two servers mount the same iSCSI LUN with a normal file system?**
A: Both think they own the disk. Each caches metadata and writes without telling the other, so the file system is corrupted. The array does not prevent it, because it only sees blocks. You need a cluster file system such as VMFS or GFS2, or a design where only one server is active.

## Quick Recap

- Block storage gives raw disks, file storage gives shared folders, and object storage gives keyed items over HTTP.
- NAS serves files, and the file system lives on the NAS. SAN serves blocks, and the file system lives on the host.
- SMB uses TCP 445. Use SMB 3.x, disable SMB1, and never expose 445 to the internet.
- NFS uses TCP 2049. NFSv3 needs helper services such as rpcbind on 111, while NFSv4 needs only one port and is stateful.
- iSCSI carries SCSI over TCP 3260, with initiators, targets, IQNs and LUNs. CHAP does not encrypt data.
- Fibre Channel is a dedicated lossless network with WWNs, FCIDs, zoning and buffer credits.
- FCoE puts FC in Ethernet with EtherType 0x8906, has no IP, and needs DCB. FCIP is the IP tunnel, and NVMe-oF is the modern path for flash.
