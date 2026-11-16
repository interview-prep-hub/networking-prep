# Software-Defined Networking (SDN)

**Level:** Advanced | **Prerequisites:** [How Routers Work](../05-routing/01-how-routers-work.md), [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md), [Access Control Lists](../09-security/05-access-control-lists.md), [Spine-Leaf Architecture](02-spine-leaf.md)

## In One Line

Software-Defined Networking moves the decisions about a network out of each individual device and into central software (a controller), so you can program the whole network through one API instead of configuring box after box.

## Why It Matters

In a classic network, every switch and router is its own small computer. Each one has its own configuration and makes its own decisions. That design survives failures well, but it is painful to change.

A real example. A company has 40 switches and needs a new VLAN with a firewall rule on all of them. An engineer logs in to each switch. At 5 minutes per switch, that is 200 minutes of careful typing. One typing mistake on switch 27 can cause an outage. Nobody can easily prove that all 40 switches now match.

SDN solves four real problems:

- **Slow change.** One API call can replace many logins.
- **No global view.** Each device only knows what its neighbours tell it. A controller sees the whole network.
- **Inconsistent config.** Software generates the config, so devices do not drift apart.
- **Fixed behaviour.** With an open interface, you can add new network features with software, without waiting for a vendor to ship new firmware.

Interviewers ask about SDN because it tests whether you understand how a network really works inside. You cannot explain SDN without knowing what a control plane is. They also want to see honest thinking. A strong candidate can say what happens when the controller dies, and when SDN is the wrong tool.

## Simple Explanation

Think of a city with many road junctions.

**Traditional network:** a police officer stands at each junction. Each officer talks only to nearby officers and decides how to direct cars. No officer sees the whole city. If you want to change traffic across the city, you must visit every officer and explain the new plan.

**SDN:** the city builds a control room with a map of every road. The control room sends each officer a simple instruction card: "cars going north, send left". Officers no longer think about the whole city. They just follow the card, very fast.

If the control room wants to change traffic, it sends new cards. One operator can change the whole city from one desk.

**Where the analogy stops being true:** a control room does not watch each car, and a controller does not touch each packet. Once the card is written, the device forwards packets by itself at full speed. Also, in most real networks the officers did not lose their brains. Many modern "SDN" products keep routing protocols running on the devices, and the controller gives policy and configuration. Pure central control is one design among several.

## How It Works

### Step 1: Understand the three planes

Every network device does three kinds of work. We call them **planes**. A plane is just a group of related jobs.

| Plane | Job | Examples | Speed |
|---|---|---|---|
| Data plane (forwarding plane) | Move each packet from an input port to an output port | MAC table lookup, IP lookup, ACL check, VXLAN encapsulation | Nanoseconds to microseconds per packet, often in hardware chips |
| Control plane | Decide where packets should go and build the tables | OSPF, BGP, STP, ARP | Milliseconds to seconds, on the device CPU |
| Management plane | Let humans and tools configure and watch the device | SSH, SNMP, NETCONF, syslog | Human speed to seconds |

A packet going **through** a router uses the data plane. An OSPF message going **to** the router uses the control plane. An engineer logging in by SSH uses the management plane.

Inside a router, the control plane builds the RIB (Routing Information Base, the full routing table). It then copies the best routes to the FIB (Forwarding Information Base, the fast table that the data plane reads). In a switch, the fast table often sits in TCAM (Ternary Content-Addressable Memory), a special memory that can search all its entries at the same time.

> **Fuller truth:** RFC 7426, an informational document on SDN terms, also defines an operational plane (the state of the device, such as "port 3 is up") and an application plane. Three planes is the version you will hear in interviews.

### Step 2: See what SDN changes

The original SDN idea, from the Open Networking Foundation (ONF, an industry group formed in 2011), has two parts:

1. Separate the control plane from the data plane.
2. Let one control plane manage many data plane devices.

```text
  TRADITIONAL                         SDN

  +---------+    +---------+          +-----------------------+
  | control |<-->| control |          | controller (control)  |
  |---------|    |---------|          +-----------------------+
  |  data   |    |  data   |             |        |        |
  +---------+    +---------+          +------+ +------+ +------+
  Each box thinks for itself          | data | | data | | data |
  and talks to its neighbours         +------+ +------+ +------+
```

The switch becomes simpler. The intelligence moves to a server.

> **Fuller truth:** the data plane device never becomes fully "dumb". It still runs an operating system and some local agents. It often handles fast local jobs such as LACP (link bundling, see [Link Aggregation](../03-ethernet-and-switching/10-link-aggregation.md)) itself.

### Step 3: Learn the three-layer architecture

```text
+------------------------------------------------------------+
| APPLICATION LAYER                                          |
| firewall app, load balancer, monitoring, scripts, CI/CD    |
+------------------------------------------------------------+
         ^
         |  Northbound API (REST with JSON, gRPC)
         v
+------------------------------------------------------------+
| CONTROL LAYER: SDN controller (cluster of 3 or more)       |
| topology, path computation, policy, device inventory       |
+------------------------------------------------------------+
         ^
         |  Southbound API (OpenFlow, NETCONF, gNMI, OpFlex)
         v
+------------------------------------------------------------+
| INFRASTRUCTURE LAYER: physical and virtual switches        |
| flow tables, forwarding hardware                           |
+------------------------------------------------------------+
```

An **API** (Application Programming Interface) is a defined way for one program to ask another program to do something. The words north and south just describe where things sit on this picture. Apps are on top (north). Devices are at the bottom (south).

**Restaurant analogy.** A customer gives an order to the head chef (northbound: "one pizza, no onions"). The head chef gives exact tickets to the cooks (southbound: "stone oven, 90 seconds"). The cooks do the fast physical work (data plane). The customer never talks to a cook.

Where the analogy stops being true: in a restaurant every dish needs a ticket. In SDN, one instruction can handle millions of packets.

### Step 4: Know what a controller does

A **controller** is server software that holds the network's brain. It usually runs as a cluster on standard servers or as virtual machines. Its main jobs:

1. **Discover the network.** Find devices, links and hosts.
2. **Keep a global state.** Store topology and policy in one logical database.
3. **Compute.** Work out paths and turn policy into device rules.
4. **Program devices.** Push rules through the southbound API.
5. **Watch and report.** Collect counters and events. Offer them through the northbound API.

**How a controller finds links.** This is a classic interview detail. In OpenFlow networks it works with LLDP (Link Layer Discovery Protocol, a small "this is me" message that devices send to neighbours):

1. The controller builds an LLDP frame for switch A, port 1.
2. It tells switch A to send that frame out of port 1.
3. Switch B receives the frame on port 3. It has a rule that sends LLDP to the controller.
4. The controller now knows: A port 1 connects to B port 3.
5. It repeats this for every port of every switch, every few seconds.

**Common controllers:**

| Controller | Type | Where you see it |
|---|---|---|
| OpenDaylight | Open source, Java | Multi-protocol controller, Linux Foundation project |
| ONOS | Open source, Java | Service provider designs |
| Ryu | Open source, Python | Labs and teaching. Development has slowed, so check its status before using it for anything real |
| Cisco APIC | Commercial | Cisco ACI data centre fabrics |
| Cisco Catalyst Center (formerly DNA Center) | Commercial | Campus networks, SD-Access |
| Cisco Nexus Dashboard Fabric Controller (formerly DCNM) | Commercial | NX-OS VXLAN EVPN fabrics |
| VMware NSX Manager (VMware is now part of Broadcom) | Commercial | Overlay networks for virtual machines |
| Juniper Apstra (Juniper is now part of HPE) | Commercial | Multi-vendor data centre fabrics |
| Arista CloudVision | Commercial | Arista fabrics, state and telemetry |
| OVN (Open Virtual Network) | Open source | Control plane for Open vSwitch, used with OpenStack and some Kubernetes setups |

### Step 5: Southbound APIs

A **southbound API** is how the controller talks down to devices. There are two main styles:

- **Imperative:** the controller gives the exact steps. "Install this rule in table 0." OpenFlow works like this.
- **Declarative:** the controller states the wanted result, and the device works out the steps. "This group may talk to that group." OpFlex works like this.

| Protocol | What it does | Transport or standard |
|---|---|---|
| OpenFlow | Programs flow tables directly | TCP 6653, ideally with TLS. ONF specification |
| NETCONF | Reads and changes device config, written in XML, with transactions | SSH port 830. RFC 6241 |
| RESTCONF | Like NETCONF, but over HTTP methods | HTTPS. RFC 8040 |
| YANG | Not a protocol. A language that describes the data model | RFC 7950 |
| gNMI | Config and streaming telemetry | gRPC over HTTP/2. OpenConfig project |
| P4Runtime | Programs tables on P4-programmable chips | gRPC. P4 project |
| OVSDB | Manages Open vSwitch settings (bridges, ports, tunnels) | RFC 7047 |
| OpFlex | Declarative policy from controller to device | Cisco ACI, originally an IETF draft |
| BGP-LS | Exports link-state topology to a controller | RFC 9552 |
| PCEP | Lets a path computation server tell routers which path to use | RFC 5440 |
| SSH with CLI, SNMP | Old-style methods, still used by many controllers for older devices | Vendor-specific |

The important interview point: **OpenFlow is one southbound protocol, not the definition of SDN.** Many real SDN systems never use it.

NETCONF, RESTCONF and gNMI return structured data, not screen text, so a program can read the result without guessing column positions. This is the biggest practical change from CLI-based management. Worked examples live in [Network Automation](12-network-automation.md).

### Step 6: Northbound and east-west APIs

A **northbound API** is how applications talk to the controller. It usually means a REST API (web-style requests such as GET and POST) that sends JSON (a simple text data format). Some controllers also offer gRPC or Python libraries.

There is **no single standard** for northbound APIs. Each controller has its own. A script written for Cisco APIC will not work on Juniper Apstra. This is one reason SDN does not fully remove vendor lock-in.

**East-west** communication is controller to controller. Members of one cluster share state with each other. Many clusters use a consensus method such as Raft, where a majority must agree before a change is accepted. This is why clusters have an odd size. With 3 nodes, the cluster survives 1 failure. With 5 nodes, it survives 2.

### Step 7: OpenFlow in detail

OpenFlow is the protocol that started the SDN movement. The paper "OpenFlow: Enabling Innovation in Campus Networks" came out of Stanford in 2008. Version 1.0 arrived in 2009, version 1.3 in 2012 and version 1.5 in 2014. Version 1.3 is the one most often deployed and taught.

OpenFlow runs between a switch and a controller over TCP port 6653. Older software used port 6633, so you may still see it. The session can and should use TLS, but it is optional in the specification.

#### The flow table

An OpenFlow switch stores **flow entries**. A flow entry says: "if a packet looks like this, do that."

| Field | Meaning | Example |
|---|---|---|
| Match fields | What to look for in the packet | in_port=1, eth_type=0x0800, ipv4_dst=10.0.0.3 |
| Priority | Higher wins when several entries match, 0-65535 | 100 |
| Counters | Packets and bytes that hit this entry | 2041 packets |
| Instructions | What to do | output to port 3, drop, go to table 1 |
| Timeouts | When to remove it | idle 30 s, hard 300 s (0 means never) |
| Cookie | A label chosen by the controller | 0x10 |

An **idle timeout** removes an entry after that many seconds with no matching packets. A **hard timeout** removes it after that many seconds no matter what.

Sample flow table:

| Priority | Match | Action | Why it exists |
|---|---|---|---|
| 200 | tcp, ipv4_dst=10.0.0.3, tcp_dst=23 | drop | Block Telnet to server |
| 100 | ipv4_dst=10.0.0.3 | output:3 | Normal traffic to server |
| 100 | ipv4_dst=10.0.0.2 | output:2 | Normal traffic to host 2 |
| 0 | (match anything) | send to controller | Table-miss entry |

The last row is the **table-miss entry**. It has priority 0 and matches everything, so it catches what no other entry caught.

> **Version detail:** in OpenFlow 1.3, if no table-miss entry exists, the switch drops unmatched packets by default. That is why a controller installs a table-miss entry as soon as a switch connects. In OpenFlow 1.0, the default was to send unmatched packets to the controller.

A flow entry can match many fields at once, from different layers. A normal router looks at the destination IP. A flow rule can look at the input port, MAC address, VLAN, IP addresses and TCP port together. So one device can act as a switch, a router or a simple firewall, depending on the rules.

#### What happens to a packet

The controller can add rules in two ways. **Reactive** means it adds a rule only when the first packet of a flow arrives. **Proactive** means it adds rules before any traffic arrives.

Reactive walk:

```text
 Host h1         Switch s1                  Controller       Host h2
    |               |                           |               |
 1  |--- packet --->|                           |               |
    |               | 2 no entry matches,       |               |
    |               |   table-miss hits         |               |
    |               |------- PACKET_IN -------->|               |
    |               |                           | 3 decide      |
    |               |<------ FLOW_MOD ----------| 4 add rule    |
    |               |<------ PACKET_OUT --------| 5 send this   |
    |               |                           |   one packet  |
    |               |-------------- packet -------------------->|
    |               |                                           |
 6  |--- packet --->| matches the rule, forwarded locally       |
    |               |-------------- packet -------------------->|
```

Only the first packet of the flow pays the controller delay. Later packets never leave the switch data path.

| Approach | Good | Bad |
|---|---|---|
| Reactive | Saves table space, rules exist only when needed | First packet is slow, controller load grows with new flows, new flows fail if controller is down |
| Proactive | No first-packet delay, keeps working if controller is down | Needs more table space, rules must be planned ahead |

Most production designs lean proactive.

#### OpenFlow messages

| Message | Direction | Purpose |
|---|---|---|
| HELLO | Both | Start the session and agree on a version (the lower of the two) |
| FEATURES_REQUEST / REPLY | Controller asks | Learn the switch ID (datapath ID), tables and capabilities |
| PACKET_IN | Switch to controller | "I need help with this packet" |
| PACKET_OUT | Controller to switch | "Send this packet out of this port" |
| FLOW_MOD | Controller to switch | Add, change or delete flow entries |
| FLOW_REMOVED | Switch to controller | "This entry expired" |
| PORT_STATUS | Switch to controller | "A port went up or down" |
| MULTIPART_REQUEST / REPLY | Controller asks | Read statistics (these were called STATS messages in 1.0) |
| BARRIER_REQUEST / REPLY | Controller asks | "Finish all earlier messages, then tell me" |
| ECHO_REQUEST / REPLY | Both | Keepalive, check the session is alive |
| ROLE_REQUEST / REPLY | Controller to switch | Set this controller's role when a switch has several |

Every OpenFlow message starts with the same 8-byte header:

```text
 0               8               16                             31
+---------------+---------------+-------------------------------+
|    version    |     type      |            length             |
+---------------+---------------+-------------------------------+
|                         xid (32 bits)                         |
+---------------------------------------------------------------+
```

| Field | Size | Meaning |
|---|---|---|
| version | 8 bits | 0x01 for OpenFlow 1.0, 0x04 for 1.3 |
| type | 8 bits | Message type. In 1.3: 0 HELLO, 10 PACKET_IN, 13 PACKET_OUT, 14 FLOW_MOD |
| length | 16 bits | Total message length in bytes, including this header |
| xid | 32 bits | Transaction ID, so a reply can be matched to its request |

#### Multiple tables, groups and meters

OpenFlow 1.0 had one table. Version 1.1 added a **pipeline**, a chain of tables. A packet starts at table 0, and a "goto-table" instruction can send it on to a later table. This avoids a huge table with every combination.

For example, table 0 checks security, table 1 learns the destination, table 2 picks the output port. With 100 security rules and 1,000 destinations, separate tables need about 1,100 entries. One flat table might need 100,000.

OpenFlow 1.1 also added the **group table**, and 1.3 added the **meter table**.

- A group holds a list of action sets (called buckets). Group types are: all (copy to every bucket, for flooding), select (pick one bucket, for load sharing), indirect (a shared next hop) and fast failover (use the first bucket whose port is alive).
- A meter measures the rate of packets and can drop those above a limit. It is a simple rate limiter.

#### When the controller is lost

An OpenFlow switch keeps its existing flow entries when the session drops. What happens next depends on the fail mode:

| Mode | Behaviour when controller connection is lost |
|---|---|
| Fail secure | Keeps forwarding by existing entries. New flows that need the controller are dropped. Entries still expire by their timeouts |
| Fail standalone | Acts like a normal learning Ethernet switch |

A switch can also connect to several controllers. Roles are equal, master and slave in the specification (newer text often says primary and secondary). A master has full control. A slave receives only limited events and cannot change the flow tables.

### Step 8: The families of SDN

The word SDN now covers several different designs. Being able to name them is a mark of a strong candidate.

| Family | Idea | Where the control plane lives | Examples |
|---|---|---|---|
| Open SDN | Controller programs simple switches directly | Mostly on the controller | OpenFlow networks, Google B4 style WAN control |
| Controller as manager | Devices keep routing protocols, controller automates config | On devices, policy on controller | Catalyst Center, Nexus Dashboard Fabric Controller |
| Overlay | Build virtual networks in software over any IP network | Overlay on the controller, underlay on devices | VMware NSX, OVN, cloud VPCs |
| Integrated fabric | Controller, hardware and overlay sold as one system | Split: fabric protocols on devices, policy on controller | Cisco ACI, Cisco SD-Access |

An **overlay** is a virtual network built on top of another network, usually with tunnels. The real network under it is called the **underlay**. See [VXLAN and EVPN](11-vxlan-and-evpn.md) for how tunnels work.

Cloud networks are probably the largest SDN systems in the world. When you create a subnet in an AWS VPC, nobody touches a switch. Software updates the mappings used by virtual switches on the hosts. See [Cloud Networking Basics: AWS VPC](../11-cloud-and-containers/03-aws-vpc-basics.md).

### Step 9: Cisco ACI

**Cisco ACI (Application Centric Infrastructure)** is Cisco's SDN for data centres. It combines three things:

- **Nexus 9000 switches** in ACI mode, wired as a spine-leaf fabric. ACI mode runs different software from standalone NX-OS.
- **APIC (Application Policy Infrastructure Controller)**, a cluster of controllers. Three is the usual production minimum.
- **A policy model** that describes applications, not VLANs and ports.

```text
             APIC cluster: holds policy, is not in the packet path
                                   |
                                   | OpFlex (policy to switches)
                                   v
         +----------+                        +----------+
         | Spine201 |                        | Spine202 |
         +----------+                        +----------+
              |    \                          /    |
              |      \                      /      |
              |        \                  /        |
              |          \              /          |
              |            \          /            |
         +----------+                        +----------+
         | Leaf101  |   Every leaf links to  | Leaf102  |
         +----------+   every spine          +----------+
              |                                    |
         Web server                           App server
         10.10.1.10                           10.10.2.20
         EPG Web                              EPG App
```

#### The ACI policy model

This model is the part interviewers ask about.

| Object | Plain meaning | Rough traditional equivalent |
|---|---|---|
| Tenant | A top-level container for one customer, team or environment | Separate admin domain |
| VRF (also called context) | A separate routing table | VRF |
| Bridge domain (BD) | A Layer 2 forwarding space, with subnets attached | VLAN plus its gateway |
| Subnet | The gateway address in a BD | SVI address |
| Application profile | A folder for the EPGs of one application | No equivalent |
| EPG (endpoint group) | A group of endpoints that share the same policy | A security zone |
| Contract | Rules for what one EPG may do to another | ACL between zones |
| Subject and filter | The parts of a contract. A filter describes ports, for example TCP 8443 | ACL lines |

An **endpoint** is anything that connects to the fabric and has an address: a server, a virtual machine, a firewall interface.

**Analogy.** Picture an office building where staff wear coloured badges. Rules say: "Blue badges may enter the green room through door 8443." No badge rule means no entry. You change a person's group by changing the badge, not by rebuilding doors.

Where the analogy stops being true: a person chooses to walk to a door. In ACI, the leaf switch decides the group automatically, based on things like the port and VLAN, or the virtual machine attributes.

#### Allow list by default

This is the key behaviour:

- Endpoints in the **same EPG** can talk freely by default.
- Endpoints in **different EPGs cannot talk** unless a contract allows it.
- A contract has a **provider** (the EPG offering a service) and a **consumer** (the EPG using it). The App EPG provides. The Web EPG consumes.

This is a form of segmentation, see [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md). It is a big change from a normal VLAN, where everything is allowed unless an ACL blocks it.

> **Fuller truth:** you can relax this. A VRF can be set to unenforced, which turns contract checking off for that VRF. There are also shortcuts such as vzAny and preferred groups. Many customers first migrate with one VLAN mapped to one EPG and a very open contract. That is called network-centric mode.

#### What runs inside the fabric

ACI is not OpenFlow, and the switches are not empty. Each one runs fabric protocols:

| Protocol | Job in ACI |
|---|---|
| LLDP and DHCP | Fabric discovery. The APIC finds a new switch and gives it an address |
| IS-IS | Underlay routing between tunnel addresses, called TEPs (Tunnel Endpoints) |
| COOP (Council of Oracle Protocol) | Leafs report endpoints to the spines. Spines hold the full endpoint database |
| MP-BGP | Carries external routes inside the fabric |
| VXLAN (Cisco extended form, often called iVXLAN) | Carries traffic between leafs, along with a policy group tag |
| OpFlex | Delivers policy from APIC to the switches |

Every leaf is the default gateway for its subnets, with the same gateway address on all leafs. This is an **anycast gateway**, so a server always finds its gateway one hop away.

#### When the APIC fails

The APIC is the management and policy point. It is **not** in the data path. If all APICs fail, the fabric keeps forwarding and keeps enforcing existing policy. You just cannot make changes through it. An interviewer will likely test this.

The APIC cluster splits its data into shards, and each shard is copied to three APICs. If a majority of the copies of a shard are lost, that shard becomes read-only until the cluster recovers.

### Step 10: Intent-based networking

**Intent-based networking (IBN)** means you tell the network the result you want, and the system works out the config, applies it, and keeps checking that the result stays true.

**Intent** is a statement of what you want, not how to do it.

- **Not intent:** "Add `deny ip 10.50.0.0 0.0.255.255 10.20.0.0 0.0.255.255` to VLAN 50."
- **Intent:** "Guest users must never reach the payment servers."

**Analogy.** A phone map app. You say "take me to the airport by 5 pm". The app picks the route. If a road closes on the way, it changes the route. You did not give turn-by-turn instructions.

Where the analogy stops being true: a destination is simple and exact. Business intent is often vague, and translating it can go wrong. A human still has to check that the system understood.

The idea became popular in 2017, from a Gartner description and Cisco's product launch. RFC 9315 (2022, informational, from the IRTF) gives shared definitions. The loop has these stages:

```text
        Business goal: "Guests must never reach payments"
                               |
                               v
   +--------------------------------------------------------+
   | 1 TRANSLATE  Turn the goal into policy, check it for   |
   |              conflicts before any change               |
   +--------------------------------------------------------+
                               |
                               v
   +--------------------------------------------------------+
   | 2 ACTIVATE   Push config to the devices automatically  |
   +--------------------------------------------------------+
                               |
                               v
   +--------------------------------------------------------+
   | 3 ASSURE     Collect telemetry, test, compare the real |
   |              state with the intent                     |
   +--------------------------------------------------------+
                               |
                               v
             Drift found? Alert or fix, then back to 1
```

A worked picture of that goal:

**Translate.** The system knows guest users are in 10.50.0.0/16 and payments are in 10.20.0.0/16. It produces:

```text
! Cisco IOS, generated by the controller
ip access-list extended GUEST-IN
 deny   ip 10.50.0.0 0.0.255.255 10.20.0.0 0.0.255.255
 permit ip 10.50.0.0 0.0.255.255 any
!
interface Vlan50
 ip access-group GUEST-IN in
```

**Activate.** The controller pushes this to the correct switches and records the change.

**Assure.** A test probe in VLAN 50 tries to reach 10.20.0.10 on a schedule. The system also analyses the config. One month later, someone adds a new payment subnet, 10.21.0.0/16, and a new route makes it reachable from guests. A plain script would never notice. An intent system compares reality with the goal and raises an alert.

**Automation vs intent:** automation does what you told it, once. Intent keeps checking that the outcome is still true. That closed loop is the difference.

Real products: Cisco Catalyst Center (campus), Cisco ACI with Nexus Dashboard (data centre), Juniper Apstra (multi-vendor data centre). Tools such as Batfish (open source) and Forward Networks model the network and verify config, which supports the assurance step.

> **Fuller truth:** in many products, "intent" means a higher-level form or template, and the assurance part is limited. When you hear the term, ask what the system actually verifies.

### Step 11: Network Functions Virtualisation (NFV)

**NFV** runs network functions as software on standard servers, instead of on special hardware boxes.

A **network function** is a job the network needs done: firewall, router, load balancer, NAT, or a mobile core function. Software that does this job is a **VNF (Virtual Network Function)**, usually running in a virtual machine. If it runs in containers, it is called a **CNF (Cloud-native Network Function)**.

**Analogy.** A smartphone replaced your camera, torch, music player and map book with apps on one device. NFV does the same to racks of network boxes.

Where the analogy stops being true: a torch app has no speed limit worth caring about. A virtual firewall must handle many gigabits per second, and a general CPU has to work much harder than a purpose-built chip to do it.

NFV began with telecom operators. An ETSI (European Telecommunications Standards Institute) group published the first NFV white paper in 2012. Operators wanted to stop buying a different box for each service.

The ETSI framework has three parts:

```text
+-----------------------------------------+  +-------------------+
| VNFs and CNFs                           |  | MANO              |
| vFirewall, vRouter, vLB, 5G UPF         |<-| Management and    |
+-----------------------------------------+  | Orchestration     |
| NFVI (NFV Infrastructure)               |  |                   |
| Virtual compute, storage, network       |  |  NFVO  (services) |
| Hypervisor or container runtime         |<-|  VNFM  (each VNF) |
| Physical servers, NICs, switches        |  |  VIM   (resources)|
+-----------------------------------------+  +-------------------+
```

| Part | Meaning | Example |
|---|---|---|
| NFVI | The servers and virtual layer that VNFs run on | x86 servers with KVM |
| VIM (Virtualised Infrastructure Manager) | Controls compute, storage and network | OpenStack, Kubernetes |
| VNFM (VNF Manager) | Starts, scales and heals each VNF | Vendor VNF manager |
| NFVO (NFV Orchestrator) | Builds services from several VNFs | Open Source MANO, ONAP |

Real examples: Cisco Catalyst 8000V (a virtual router, successor to CSR 1000V), F5 BIG-IP Virtual Edition, and virtual firewalls from many vendors. 5G core networks are designed as software functions, so CNFs are now a normal part of telecom.

**Service function chaining (SFC)** sends traffic through several functions in order, such as firewall, then IDS, then load balancer. RFC 7665 gives the architecture, and RFC 8300 defines the Network Service Header that can carry the path. An SDN controller can steer traffic through this chain without moving any cables.

**SDN vs NFV.** They solve different problems. You can have either one alone.

| | SDN | NFV |
|---|---|---|
| Main question | Where does control live? | Where does the function run? |
| Main change | Separates control from forwarding | Separates software from special hardware |
| Result | Programmable network | Network boxes become software |
| Born in | Campus and data centre research | Telecom operators |
| Standards body | ONF and IETF work | ETSI |
| Can exist alone? | Yes | Yes |

They work well together. NFV creates a virtual firewall. SDN steers the right traffic to it.

### Step 12: Benefits and limits

| Benefit | What it means in practice |
|---|---|
| Central control | One place for policy and one view of the network |
| Speed of change | Minutes instead of days, can be part of a CI/CD pipeline |
| Consistency | Fewer hand-typed errors, easier audit |
| Better path use | Global view allows smarter traffic engineering |
| Fine segmentation | Policy can follow groups, not IP addresses |
| Programmability | Standard tools and APIs instead of screen scraping |
| Possible hardware choice | Open interfaces can reduce dependence on one vendor |

| Limit | What it means in practice |
|---|---|
| Controller availability | Needs a cluster and careful design. It concentrates risk |
| Blast radius | One bad policy can reach every device in seconds |
| Security target | Anyone who controls the controller controls the network |
| Controller-to-switch delay | Reactive setups add first-packet delay, and long distances hurt |
| Table size | Hardware TCAM holds a limited number of rules |
| Scale of reactive flows | Too many new flows can overload the controller and the switch CPU |
| Vendor lock-in moves | From the switch to the controller and its API |
| Harder troubleshooting | Overlay, underlay and controller state must all be checked |
| Skills | Teams need software skills as well as protocol skills |
| Cost | Licences can cancel the expected hardware savings |
| Legacy devices | Older gear may not support the needed APIs |

For a small network with 5 switches that rarely change, a controller is usually more trouble than it is worth. SDN pays off with scale, frequent change, or a need for strong segmentation.

## Example

### Example 1: Build a tiny SDN with Open vSwitch and Ryu

This lab shows every idea above on one Linux machine. It uses:

- **Mininet**: creates virtual hosts and switches on one Linux machine.
- **Open vSwitch (OVS)**: a software switch that speaks OpenFlow.
- **Ryu**: a small Python controller.

These tools run on Linux. On Windows, use a Linux VM. The lab here uses a VM at 192.168.56.10.

```text
                 Ryu controller 127.0.0.1:6653 (southbound)
                 REST API       0.0.0.0:8080   (northbound)
                         |
                         | OpenFlow 1.3
                         |
                    +---------+
                    |   s1    |
                    +---------+
              eth1 /    | eth2  \ eth3
                  /     |        \
                h1      h2        h3
          10.0.0.1  10.0.0.2  10.0.0.3
       MAC ...:01   MAC ...:02  MAC ...:03
```

**Step 1. Start the controller.** It loads a learning switch and a REST app.

```bash
# Linux, terminal 1
ryu-manager ryu.app.simple_switch_13 ryu.app.ofctl_rest
```

**Step 2. Start the network.**

```bash
# Linux, terminal 2
sudo mn --topo single,3 --mac \
  --switch ovsk,protocols=OpenFlow13 \
  --controller remote,ip=127.0.0.1,port=6653
```

**Step 3. Check that the switch reached the controller.**

```bash
# Linux, terminal 3
sudo ovs-vsctl show
```

```text
    Bridge s1
        Controller "tcp:127.0.0.1:6653"
            is_connected: true
        fail_mode: secure
        Port s1-eth1
            Interface s1-eth1
        Port s1-eth2
            Interface s1-eth2
        Port s1-eth3
            Interface s1-eth3
        Port s1
            Interface s1
                type: internal
```

**Step 4. Look at the empty table.** The only entry is the table-miss rule.

```bash
sudo ovs-ofctl -O OpenFlow13 dump-flows s1
```

```text
 cookie=0x0, duration=8.3s, table=0, n_packets=0, n_bytes=0,
   priority=0 actions=CONTROLLER:65535
```

Output lines in this example are wrapped to fit the page. The real tool prints each entry on one line. CONTROLLER:65535 means "send the whole packet to the controller".

**Step 5. Send traffic.** Watch the first ping.

```text
mininet> h1 ping -c 3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=6.41 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.298 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.071 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2021ms
```

The first ping took about 6 ms. The others took under 0.3 ms. The first one paid for the ARP exchange and the controller trips. Later packets never left the switch.

The Ryu terminal shows three PACKET_IN events:

```text
packet in 0000000000000001 00:00:00:00:00:01 ff:ff:ff:ff:ff:ff 1
packet in 0000000000000001 00:00:00:00:00:02 00:00:00:00:00:01 2
packet in 0000000000000001 00:00:00:00:00:01 00:00:00:00:00:02 1
```

Read each line as: switch ID, source MAC, destination MAC, input port.

1. The ARP request from h1 to broadcast. The controller learns h1 is on port 1 and floods the packet.
2. The ARP reply from h2 to h1. The controller learns h2 is on port 2 and adds a rule for h2 to h1.
3. The first ICMP echo request from h1 to h2. The controller adds a rule for h1 to h2.

**Step 6. See the rules the controller added.**

```bash
sudo ovs-ofctl -O OpenFlow13 dump-flows s1
```

```text
 cookie=0x0, duration=9.8s, table=0, n_packets=3, n_bytes=294,
   priority=1,in_port="s1-eth2",dl_src=00:00:00:00:00:02,
   dl_dst=00:00:00:00:00:01 actions=output:"s1-eth1"
 cookie=0x0, duration=9.8s, table=0, n_packets=2, n_bytes=196,
   priority=1,in_port="s1-eth1",dl_src=00:00:00:00:00:01,
   dl_dst=00:00:00:00:00:02 actions=output:"s1-eth2"
 cookie=0x0, duration=42.1s, table=0, n_packets=3, n_bytes=182,
   priority=0 actions=CONTROLLER:65535
```

Check the numbers. Each ping frame is 98 bytes (56 data, 8 ICMP, 20 IP, 14 Ethernet).

| Entry | Packets | Bytes | Why |
|---|---|---|---|
| h2 to h1 | 3 | 3 x 98 = 294 | All 3 echo replies |
| h1 to h2 | 2 | 2 x 98 = 196 | Echo requests 2 and 3. Request 1 went by PACKET_OUT |
| Table-miss | 3 | 42 + 42 + 98 = 182 | Two ARP frames and the first echo request |

The counts assume no other traffic. On a real run, background traffic such as ARP refresh can add to them.

**Step 7. Use the northbound API.** Now act as an application. We block h1 from reaching h3 with one HTTP call. We never touch the switch.

```bash
# Linux: northbound call to the controller.
# eth_type 2048 is 0x0800 (IPv4). Empty actions means drop.
curl -X POST http://127.0.0.1:8080/stats/flowentry/add -d '{
  "dpid": 1,
  "table_id": 0,
  "priority": 100,
  "match": {
    "eth_type": 2048,
    "ipv4_src": "10.0.0.1",
    "ipv4_dst": "10.0.0.3"
  },
  "actions": []
}'
```

The same call from a Windows host:

```powershell
# Windows PowerShell: the same northbound call to the lab VM
$body = @{
  dpid = 1; table_id = 0; priority = 100
  match = @{ eth_type = 2048; ipv4_src = "10.0.0.1"; ipv4_dst = "10.0.0.3" }
  actions = @()
} | ConvertTo-Json -Depth 4
Invoke-RestMethod -Method Post -Body $body `
  -Uri http://192.168.56.10:8080/stats/flowentry/add
```

The controller turned JSON from the north into a FLOW_MOD to the south. Test it:

```text
mininet> h1 ping -c 2 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.

--- 10.0.0.3 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1016ms
```

```bash
sudo ovs-ofctl -O OpenFlow13 dump-flows s1 | grep priority=100
```

```text
 cookie=0x0, duration=14.2s, table=0, n_packets=2, n_bytes=196,
   priority=100,ip,nw_src=10.0.0.1,nw_dst=10.0.0.3 actions=drop
```

ARP still worked, because the rule only matches IPv4. ICMP was dropped in the switch, and the counter shows the 2 lost pings.

**Step 8. Kill the controller.** Press Ctrl+C in terminal 1, then test.

```text
mininet> h1 ping -c 1 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.088 ms

mininet> h2 ping -c 1 h3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
From 10.0.0.2 icmp_seq=1 Destination Host Unreachable
```

This is fail secure mode in action. The h1 to h2 flow still works because its rules are in the switch and have no timeout. The new h2 to h3 conversation fails. h2's ARP request reaches the table-miss entry, and no controller is there to answer it. This one test explains the main risk of reactive SDN.

To watch the messages, capture on the loopback interface with Wireshark. The display filter `openflow_v4.type == 10` shows PACKET_IN, and `openflow_v4.type == 14` shows FLOW_MOD. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

### Example 2: An application policy in Cisco ACI

**Goal:** the Shop application has Web servers in 10.10.1.0/24 and App servers in 10.10.2.0/24. Web may reach App on TCP 8443 only.

Objects needed:

```text
Tenant Shop
  |
  +-- VRF Prod
  |
  +-- BD WebBD (10.10.1.1/24) ---+
  +-- BD AppBD (10.10.2.1/24) ---|--+
  |                              |  |
  +-- App profile ShopApp        |  |
  |     +-- EPG Web  (uses BD) --+  |
  |     |     consumes web-to-app   |
  |     +-- EPG App  (uses BD) -----+
  |           provides web-to-app
  |
  +-- Contract web-to-app
  |     +-- Subject s1 --> Filter tcp8443
  |
  +-- Filter tcp8443 (TCP destination port 8443)
```

Log in to the APIC, then post the whole tenant in one call. The call describes the wanted state.

```bash
# Linux: log in and keep the session cookie
curl -k -c cookie.txt -X POST \
  https://apic1.example.com/api/aaaLogin.json \
  -d '{"aaaUser":{"attributes":{"name":"admin","pwd":"<password>"}}}'

# Post the tenant policy from a file
curl -k -b cookie.txt -X POST \
  https://apic1.example.com/api/mo/uni.json \
  -d @shop-tenant.json
```

File shop-tenant.json:

```text
{"fvTenant": {"attributes": {"name": "Shop"}, "children": [
  {"fvCtx": {"attributes": {"name": "Prod"}}},
  {"fvBD": {"attributes": {"name": "WebBD"}, "children": [
    {"fvRsCtx": {"attributes": {"tnFvCtxName": "Prod"}}},
    {"fvSubnet": {"attributes": {"ip": "10.10.1.1/24"}}}
  ]}},
  {"fvBD": {"attributes": {"name": "AppBD"}, "children": [
    {"fvRsCtx": {"attributes": {"tnFvCtxName": "Prod"}}},
    {"fvSubnet": {"attributes": {"ip": "10.10.2.1/24"}}}
  ]}},
  {"vzFilter": {"attributes": {"name": "tcp8443"}, "children": [
    {"vzEntry": {"attributes": {"name": "e1", "etherT": "ip",
      "prot": "tcp", "dFromPort": "8443", "dToPort": "8443"}}}
  ]}},
  {"vzBrCP": {"attributes": {"name": "web-to-app"}, "children": [
    {"vzSubj": {"attributes": {"name": "s1"}, "children": [
      {"vzRsSubjFiltAtt": {"attributes": {"tnVzFilterName": "tcp8443"}}}
    ]}}
  ]}},
  {"fvAp": {"attributes": {"name": "ShopApp"}, "children": [
    {"fvAEPg": {"attributes": {"name": "Web"}, "children": [
      {"fvRsBd": {"attributes": {"tnFvBDName": "WebBD"}}},
      {"fvRsCons": {"attributes": {"tnVzBrCPName": "web-to-app"}}}
    ]}},
    {"fvAEPg": {"attributes": {"name": "App"}, "children": [
      {"fvRsBd": {"attributes": {"tnFvBDName": "AppBD"}}},
      {"fvRsProv": {"attributes": {"tnVzBrCPName": "web-to-app"}}}
    ]}}
  ]}}
]}}
```

A success returns:

```text
{"totalCount":"0","imdata":[]}
```

> **Fuller truth:** a real tenant also links each EPG to a domain and to ports or a virtual machine manager, so the fabric knows which endpoints belong in it. Those lines are left out to keep the focus on the policy.

Notice what is **not** here: no VLAN number on a switch, no ACL with IP addresses, and no mention of leaf 101 or 102. Policy names groups, not addresses.

On the leaf, the APIC's policy becomes hardware rules. Output trimmed to fit:

```text
leaf-101# show zoning-rule scope 2555904
Rule  SrcEPG  DstEPG  Filter    Dir             Action
4126  49153   49154   45        bi-dir          permit
4127  49154   49153   45        uni-dir-ignore  permit
4098  any     any     implicit  uni-dir         deny,log
```

- 49153 is the tag (called a pcTag, policy control tag) of EPG Web, and 49154 is EPG App. Tag values are examples. The fabric assigns them.
- 2555904 identifies the VRF.
- The final line is the implicit deny. No contract means no traffic.

**Packet walk**, Web 10.10.1.10 on Leaf 101 to App 10.10.2.20 on Leaf 102, port 8443:

1. The frame arrives at Leaf 101. The port and VLAN map it to EPG Web, tag 49153.
2. Leaf 101 is the gateway, so it routes the packet. It looks up 10.10.2.20.
3. If it knows the endpoint, it uses Leaf 102's TEP address, for example 10.0.72.64. If not, it sends the packet to the spine proxy, and the spine finds the location in its COOP database.
4. If Leaf 101 knows the destination EPG, it checks the policy here. If not, the check happens at Leaf 102.
5. The packet travels in VXLAN. The header carries the VRF identifier, the source tag 49153 and a bit that says whether policy was already applied.
6. Leaf 102 removes the tunnel header and delivers the packet to the App server.

The APIC took no part in this walk.

## Going Deeper

### Why pure OpenFlow did not take over

OpenFlow was a brilliant research idea, but it never replaced normal networking in most enterprises. Reasons:

- **Hardware mismatch.** Switch chips were built around fixed pipelines (MAC table, then routing, then ACL). A generic match-anything flow table fits only into TCAM, which is small, expensive and power-hungry.
- **Table sizes.** Wildcard table capacity on many switches is thousands to tens of thousands of entries, depending on the chip.
- **Slow flow setup.** Published tests of older hardware switches showed hundreds to a few thousand flow installs per second.
- **Vendor fragmentation.** Each chip handled the optional features differently.
- **Good enough alternatives.** BGP, EVPN and VXLAN gave scale and multi-tenancy without a central single point in the data path.

OpenFlow lived on in specific places: software switches such as OVS, research networks and some WAN designs. The big idea won, even though this protocol did not become universal. Central policy and APIs are now normal.

### Control plane scale

In reactive mode, the controller's work grows with the rate of **new flows**, not with bandwidth. Take a simple illustrative case. A rack sees 20,000 new flows per second. If the switch CPU can send only 2,000 PACKET_IN messages per second, 90 percent of new flows wait or fail. Bandwidth was never the problem.

This also creates an attack. An attacker can send packets with random addresses, so every packet misses the table. This floods both switch CPU and controller. Defences:

- Install proactive rules, so few packets miss.
- Use wider matches instead of exact ones.
- Rate limit traffic to the controller.
- Protect the switch CPU. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

### Why group tags beat IP rules

Say 500 web servers may reach 100 app servers on one port.

- **IP-based ACL:** 500 x 100 = 50,000 entries, if each pair is listed.
- **Group-based policy:** 1 rule, from tag Web to tag App.

Add a new web server, and the IP-based list grows by 100 entries. In the group model, the rule stays the same. The server just gets a tag. This is why ACI, Cisco TrustSec and cloud security groups all use groups.

> **Fuller truth:** subnet summaries reduce the IP-based count in tidy networks. But real servers are often not in tidy ranges, and groups still give the cleaner policy.

### Consistency and the CAP trade-off

A controller cluster is a distributed database. The CAP theorem says that during a network partition (the cluster splits into parts that cannot talk), a system must choose between consistency (everybody sees the same data) and availability (everybody can still write).

- **Prefer consistency:** the minority side refuses changes. The APIC read-only shard behaviour is an example of this idea. Safe, but you may lose the ability to change the network.
- **Prefer availability:** both sides accept changes and merge later. Faster, but you may get conflicts.

Many controllers mix the two: strong consistency for critical data, and eventual consistency (replicas catch up later) for fast-changing data such as topology.

### Consistent network updates

Changing a path is not atomic across switches, because messages arrive at different times.

Say traffic moves from path A-B-D to path A-C-D. If A changes first but C is not ready, packets reach C and are dropped. A bad order can also form a short loop. The safe method:

1. Program the new path on downstream switches first (C, then D).
2. Use a BARRIER_REQUEST to confirm the rules are installed.
3. Change the ingress switch A last.
4. Remove the old rules after in-flight packets are gone.

For stronger safety, a two-phase update tags packets with a version. Each packet follows entirely the old rules or entirely the new ones. Traditional protocols have a similar problem: microloops during routing convergence.

### Fast failover without the controller

A trip to the controller can take many milliseconds. Fast reroute cannot wait for that. OpenFlow fast failover groups solve this by loading a backup in advance:

```text
Group 1 (type fast failover)
  Bucket 1: watch port 1, output to port 1  (primary)
  Bucket 2: watch port 2, output to port 2  (backup)
```

When port 1 fails, the switch moves to bucket 2 by itself. The controller later computes a better path. This is a general SDN design rule: **fast local reaction on the device, slow global optimisation on the controller.**

### P4: programming the pipeline itself

OpenFlow has a fixed list of match fields. A new protocol requires a new specification version. P4 (Programming Protocol-independent Packet Processors) goes further. You write a program that says which headers exist and how tables use them. The compiled program runs on a programmable chip, a smart NIC or a software target. P4Runtime then fills the tables at run time.

So OpenFlow says "use my fields". P4 says "define your own fields". eBPF offers a related form of programmability in the Linux kernel, see [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md).

### SDN in the WAN

Wide area links are expensive, so better use of them saves real money. Google's B4 paper (SIGCOMM 2013) described a private WAN between data centres, using OpenFlow and central traffic engineering. It reported many links running near 100 percent utilisation and an average around 70 percent, compared with the 30-40 percent typical of WAN links. Global knowledge of demand and paths made this possible.

Service providers often use a related design: a PCE (Path Computation Element) computes paths, BGP-LS feeds it the topology, and PCEP tells the head-end router which path to use, often with segment routing. See [MPLS](../05-routing/13-mpls.md). In enterprises, the same ideas appear in [SD-WAN](10-sd-wan.md).

### NFV performance

A normal Linux network path is too slow for a heavy VNF, because each packet passes through interrupts, the kernel and the hypervisor. Common fixes:

| Technique | What it does | Cost |
|---|---|---|
| DPDK (Data Plane Development Kit) | The application polls the NIC directly from user space, bypassing the kernel | CPU cores stay at 100 percent busy polling |
| SR-IOV (Single Root I/O Virtualization) | The NIC exposes virtual functions straight to a VM | The virtual switch is bypassed, so live migration and overlay policy get harder |
| CPU pinning, NUMA awareness | Keeps a VM on fixed cores, near its memory and NIC | Less flexible resource sharing |
| Huge pages | Reduces memory lookup misses | Memory is reserved up front |

Many telcos found that a VNF was not just "a VM". Each one needed tuning, and some were large single programs moved from hardware. This is a key reason for the move to CNFs.

### Securing the controller

The controller is the most valuable target in the network. A minimum checklist:

- Mutual TLS on the southbound channel. Plain TCP lets an attacker on the path change flow rules.
- Strong login, role-based access control and audit logs on the northbound API.
- Keep management on a separate network, for example out-of-band.
- Treat policy as code: peer review, pre-checks and staged rollout, to reduce the blast radius.
- Protect discovery. Fake LLDP frames can trick a controller into believing a link exists.

## Common Mistakes

- **"SDN means OpenFlow."** OpenFlow is one southbound protocol. ACI, NSX and Catalyst Center are SDN products that do not depend on it in the classic way.
- **"Every packet goes to the controller."** Only table misses do, and only in reactive designs. Forwarding stays on the device.
- **"If the controller dies, the network dies."** It depends on the design. Existing flows usually keep working. In ACI, forwarding continues. What you lose is change, and in reactive designs, new flows.
- **"SDN removes the need for network protocol knowledge."** You still need routing and switching to build the underlay and to debug. ACI still runs IS-IS and BGP inside.
- **"The control plane moved, so switches need no CPU."** Devices still run an OS, agents and local protocols.
- **Mixing up northbound and southbound.** North is toward apps. South is toward devices. Remember the diagram.
- **Thinking SDN and NFV are the same.** SDN is about control. NFV is about moving functions from hardware to software. Either can exist alone.
- **Thinking overlay equals SDN.** VXLAN is an encapsulation. It can run with a controller or with plain BGP EVPN and no controller at all.
- **Thinking a VLAN and an EPG are the same.** A VLAN is a Layer 2 construct. An EPG is a policy group. Several VLANs can map to one EPG.
- **Forgetting the ACI default.** New engineers build two EPGs and wonder why ping fails. No contract means no traffic.
- **Calling a script "intent".** A script runs once. Intent includes checking that the goal stays true.
- **Assuming SDN is always cheaper.** Controller licences and new skills can cost more than the saved labour.
- **Assuming no vendor lock-in.** Northbound APIs are not standard, so lock-in often just moves to the controller.

## Interview Questions

**Q: What is Software-Defined Networking?**

A: SDN separates the control plane, which decides where traffic goes, from the data plane, which forwards it, and manages the control logic centrally in software. Applications program the network through APIs. The result is one policy point, a global view and fast, consistent change.

**Q: What are the data, control and management planes? Give an example of each.**

A: The data plane forwards packets, for example a FIB lookup in hardware. The control plane builds the forwarding information, for example OSPF or BGP. The management plane is how we configure and monitor the device, for example SSH or SNMP. Traffic through the box is data plane, and traffic to the box is control or management.

**Q: What is the difference between northbound and southbound APIs?**

A: Southbound is between the controller and the devices, for example OpenFlow, NETCONF, gNMI or OpFlex. Northbound is between the controller and applications, usually REST with JSON. Southbound has several real standards. Northbound is mostly specific to each controller.

**Q: Walk me through how an OpenFlow switch handles a new packet.**

A: The switch checks the flow table in priority order. If an entry matches, it updates the counters and runs the instructions. If nothing else matches, the table-miss entry sends a PACKET_IN to the controller. The controller decides, sends a FLOW_MOD to install a rule, and sends a PACKET_OUT for that packet. Later packets match the rule in the switch.

**Q: What is the difference between reactive and proactive flow installation?**

A: Reactive installs a rule when the first packet arrives. It saves table space, but it adds first-packet delay and depends on the controller for each new flow. Proactive installs rules in advance. There is no first-packet delay, and traffic survives a controller outage, but it uses more table space. Production designs lean proactive.

**Q: What happens if the SDN controller fails?**

A: The switches keep their installed rules, so existing traffic normally continues. In fail secure mode, new flows that need the controller are dropped. In fail standalone mode, the switch behaves like a normal learning switch. In ACI, the fabric keeps forwarding and enforcing policy, and only changes stop. To reduce this risk, controllers run as clusters of three or more, usually an odd number.

**Q: Is ACI true SDN? How does it differ from OpenFlow?**

A: ACI is SDN in the sense of central policy and a full API, but it is not the classic model. Switches still run IS-IS, COOP and MP-BGP themselves. The APIC sends declarative policy through OpFlex, and the leaf works out how to apply it. OpenFlow is imperative: the controller programs the exact flow entries. The APIC is also out of the data path.

**Q: What are an EPG and a contract in ACI?**

A: An EPG is a group of endpoints that share the same policy, such as all web servers. A contract defines which traffic is allowed between EPGs, with a provider and a consumer. Inside an EPG, traffic is allowed by default. Between EPGs, it is denied unless a contract allows it, so ACI is an allow-list model.

**Q: What is the difference between SDN and NFV?**

A: SDN separates control from forwarding, so it changes how the network is controlled. NFV moves functions such as firewalls and routers from hardware boxes to software on standard servers. They are independent, but they complement each other: NFV creates a virtual firewall, and SDN steers traffic to it.

**Q: What is intent-based networking, and how is it different from automation?**

A: You state the outcome you want, such as "guests must not reach payments". The system translates it to config, activates it, then keeps checking that the outcome stays true. Automation runs the steps you defined. Intent adds validation before the change and a closed assurance loop after it.

**Q: Why did OpenFlow not replace traditional networking?**

A: Generic flow tables need TCAM, which is small and expensive, so rule capacity was limited. Flow setup on hardware switches was slow, and reactive designs had scale and availability problems. Meanwhile, BGP EVPN and VXLAN solved data centre needs in a distributed way. The central idea survived, but mostly through APIs, overlays and policy controllers.

**Q: When would you not recommend SDN?**

A: For a small, stable network, with few changes and a team without software skills, a controller adds cost and a new failure point. I would also be careful where the team cannot support clustering, or where old devices lack APIs. In those cases, good templates and simple automation may give most of the value.

## Quick Recap

- SDN separates the control plane from the data plane and manages control centrally. Devices keep forwarding on their own once they have rules.
- The controller talks south to devices (OpenFlow, NETCONF, gNMI, OpFlex) and north to apps (usually REST with JSON). Northbound has no single standard.
- OpenFlow uses prioritised flow tables, PACKET_IN, FLOW_MOD and PACKET_OUT over TCP 6653. Rules can be reactive or proactive.
- Cisco ACI uses APIC, spine-leaf Nexus 9000 switches and a policy model of EPGs and contracts. It denies traffic between EPGs by default, and the APIC is not in the data path.
- Intent-based networking adds translate, activate and assure, so the outcome keeps being checked.
- NFV runs network functions as software (VNFs, CNFs) under ETSI's NFVI and MANO framework. It is separate from SDN and works well with it.
- SDN brings speed, consistency and a global view, but it adds controller risk, table limits, a large blast radius and lock-in at the API.
