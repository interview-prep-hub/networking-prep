# Zero Trust and Segmentation

**Level:** Advanced | **Prerequisites:** [Firewalls](04-firewalls.md), [VPNs](07-vpn.md), [Access Control Lists](05-access-control-lists.md), [VLANs](../03-ethernet-and-switching/06-vlans.md), [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md), [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md)

## In One Line

Zero trust means no user, device or program gets access just because of where it sits on the network: every access is checked against identity, device health and policy, and segmentation limits how far an attacker can move if a check fails.

## Why It Matters

For about 30 years, companies protected networks like castles. A strong firewall stood at the edge. Everything inside was trusted.

That model broke for three reasons:

- Staff now work from home, and apps now live in the cloud. There is no single "inside" any more.
- Attackers do not need to break the firewall. They steal a password, or they send a phishing email, and they log in.
- Once inside a flat network, an attacker can reach almost everything.

Real incidents show this:

- **Target, 2013:** attackers used stolen credentials from an air-conditioning vendor, then moved from that foothold to the payment systems.
- **NotPetya, 2017:** the malware spread inside company networks using stolen credentials and an SMB exploit. Whole global networks went down in hours.
- **Colonial Pipeline, 2021:** the company's CEO told the US Senate that attackers got in through a VPN account with a leaked password and no multi-factor login.

In all three cases, the deep problem was the same: one small entry gave access to far too much.

Interviewers ask about this because it joins many topics: firewalls, VLANs, VPNs, TLS, identity and cloud. A good answer shows you can think about design, not only commands. Expect questions such as:

- "Why is a VPN not zero trust?"
- "How does 802.1X work?"
- "What is the difference between SASE and SSE?"
- "How would you stop lateral movement in a data centre?"

## Simple Explanation

**The old model is a castle.** There is a high wall and one gate. A guard checks you at the gate. After that, you can walk into any room in the castle.

**Zero trust is a modern hospital.** Every door has a badge reader. The reader does not care that you are already inside the building. It asks: who are you, are you on shift today, and are you allowed into this room? A cleaner's badge opens the corridors. It does not open the medicine store. A surgeon's badge opens the operating theatre, but only while the surgeon is on duty.

Where the analogy stops being true: a hospital checks a person at each door. A network does not check every single packet against the identity system. It checks when a session starts, then keeps watching for changes during the session.

**Segmentation is the walls between rooms.** A ship has watertight compartments. If one compartment floods, the doors close and the ship stays afloat. In a network, a segment is a group of systems with a controlled border around it. If an attacker takes one server, the walls stop the attack from spreading.

Where the analogy stops being true: water spreads everywhere by itself. An attacker chooses a path, and uses allowed paths when it can. A wall with an open door (an allowed port) is still a route.

**The small number of words to remember:**

| Idea | Question it answers |
|---|---|
| Zero trust | What is the rule for giving access? |
| Identity | Who or what is asking? |
| Segmentation | How far can a problem spread? |
| NAC and 802.1X | Who may plug into the network at all? |
| ZTNA | How do remote users reach one private app? |
| SASE and SSE | Where do we run all these checks for a modern company? |

## How It Works

### Part 1: The perimeter model

A **perimeter** is the edge between "our network" and "everything else". The perimeter model, also called **castle and moat**, puts its security at that edge.

```text
              Internet (untrusted)
                     |
               +-----------+
               | Firewall  |   <- almost all checks live here
               +-----------+
                     |
    ========= "trusted" internal LAN =========
     |          |          |           |
  Laptops    Printers    Servers    Databases

    <------ free movement inside ------>
```

Two terms describe the direction of traffic:

- **North-south traffic** crosses the edge, such as a user going to a website.
- **East-west traffic** moves between systems inside, such as a web server talking to a database.

The castle model inspects north-south traffic well. It often does not see east-west traffic at all, because that traffic never passes the edge firewall.

A **DMZ** (demilitarised zone) improved this a little. It is a separate zone for public servers, so a hacked web server is not directly on the LAN. It is still a perimeter idea. See [Firewalls](04-firewalls.md).

### Part 2: Why the perimeter failed

1. **The edge dissolved.** Email moved to Microsoft 365 or Google Workspace. Apps moved to AWS, Azure and SaaS (software as a service: apps you rent over the internet). Laptops left the building. Traffic no longer passes one gate.
2. **Credentials became the easy way in.** A valid password passes the firewall and the VPN. The firewall sees a normal login.
3. **Implicit trust is dangerous.** "Implicit trust" means trust given only because of location. In a flat network, a hacked printer and a domain controller share the same trust.
4. **Lateral movement is easy.** Lateral movement means an attacker moving from one internal system to another. It is the step that turns one infected laptop into a company-wide disaster.
5. **The edge devices became targets.** VPN gateways and firewalls face the internet, so attackers hunt for their bugs. In January 2024, for example, exploited flaws in Ivanti Connect Secure VPN gateways led CISA (the US cyber defence agency) to issue an emergency directive.
6. **Insiders and third parties are already inside.** Contractors, vendors and unhappy staff do not need to cross the perimeter.

A typical attack path in a flat network:

```text
 Phishing email
      |
      v
 Laptop 10.20.0.57 infected
      |  steal saved credentials
      v
 File server 10.50.1.20   (SMB allowed from everywhere)
      |  find admin credentials
      v
 Domain controller 10.10.10.30
      |  push ransomware to all hosts
      v
 Whole company encrypted
```

The perimeter firewall was never involved in any step after the first one. The term **blast radius** means how much can be damaged from one compromised point. In a flat network, the blast radius is everything.

The fuller truth: perimeters are not useless. You still need edge firewalls. Zero trust removes the idea that the edge is enough. It does not remove the edge.

### Part 3: Zero trust principles

**History, briefly:**

- 2004: the Jericho Forum, a group of security leaders, argued that perimeters were fading. They called this "de-perimeterisation".
- 2010: John Kindervag at Forrester published the zero trust model.
- 2014: Google published its first BeyondCorp paper, on letting staff work from any network with no VPN. Google started that work after the Operation Aurora attacks of 2009.
- 2020: NIST (the US standards institute) published **SP 800-207, Zero Trust Architecture**. This is the document most people cite.
- 2021-2022: US Executive Order 14028 and OMB memo M-22-09 pushed US federal agencies toward zero trust.
- 2023: CISA released version 2.0 of its Zero Trust Maturity Model.

**The three core ideas.** These short forms are common in industry (the wording here follows Microsoft's popular version):

1. **Verify explicitly.** Check identity, device health, location, app and risk on each access. Do not assume.
2. **Use least privilege.** Give the smallest access needed, for the shortest time needed.
3. **Assume breach.** Design as if an attacker is already inside. Limit the damage and keep logs.

**The seven tenets of NIST SP 800-207, in plain words:**

1. All data sources and computing services count as resources, including personal devices that reach company data.
2. All communication is secured, wherever it comes from. Being on the office LAN earns nothing.
3. Access to each resource is granted per session.
4. Access is decided by dynamic policy: who is asking, what they ask for, the device they use, and behaviour and environment signals.
5. The organisation monitors the health and security posture of all its assets.
6. Authentication and authorisation are dynamic and strictly enforced before access is allowed.
7. The organisation collects as much information as it can about assets, network and traffic, and uses it to improve security.

**Terms used above:**

- **Authentication (AuthN):** proving who you are.
- **Authorisation (AuthZ):** deciding what you may do.
- **Posture:** the health of a device. Is the disk encrypted? Is the operating system patched? Is the security agent running?
- **Least privilege:** access to only what the job needs.

**What zero trust is not:**

- It is not a product. You cannot buy it in a box.
- It does not mean "trust nothing, ever". It means "no trust from network location alone". Trust is earned, measured and limited.
- It does not mean "remove firewalls".
- It is not finished in one project. It is a direction you move in.

### Part 4: The zero trust architecture

NIST splits every access into two parts: deciding and enforcing.

```text
             CONTROL PLANE (decides)
   +-------------------------------------------+
   |  Policy Decision Point (PDP)              |
   |    Policy Engine + Policy Administrator   |
   +-------------------------------------------+
        ^          |                 ^
        | ask      | allow or deny   | signals: identity
        |          v                 | provider, device
   +---------------------------+     | health, threat
   | Policy Enforcement Point  |     | intel, logs, SIEM
   | (PEP)                     |
   +---------------------------+
   Subject  ------> PEP ------> Resource
             DATA PLANE (carries traffic)
```

- **Subject:** the user, device or program that asks.
- **Resource:** the app, data or service it wants.
- **Policy Engine (PE):** makes the decision.
- **Policy Administrator (PA):** carries out the decision, for example by creating a session token or telling the PEP to open a path.
- **Policy Enforcement Point (PEP):** the gate that actually allows or blocks the traffic. It can be a switch port, a proxy, a firewall, an agent on a host or a sidecar.
- **SIEM:** the system that collects security logs from everywhere. See [IDS, IPS and SIEM](06-ids-and-ips.md).

The key rule: the subject never talks to the resource until the PDP says yes, and the PEP is the only path.

NIST describes three ways to build this. Real designs mix them:

| Approach | Main tool | Example |
|---|---|---|
| Enhanced identity governance | Identity provider and policy | Conditional access for SaaS apps |
| Microsegmentation | Host firewalls, gateways | Rules around each workload |
| Network infrastructure and software-defined perimeters | Overlay network, ZTNA | App reachable only through a broker |

### Part 5: Segmentation and microsegmentation

**Segmentation** means dividing a network into zones and controlling traffic between them. **Macrosegmentation** uses large zones: users, servers, guests, IoT. **Microsegmentation** goes down to single workloads or even single processes.

A **workload** is anything that runs a service: a server, a virtual machine (VM), a container.

Analogy: macrosegmentation is the walls between departments. Microsegmentation is a lock on every office door and every filing cabinet. It stops being true at one point: a filing cabinet does not need to talk to other cabinets, but workloads must talk to each other. So the job is not only to lock things. The job is to know which paths are real and keep those open.

Tools from coarse to fine:

| Method | Works at | Enforced by | Good for | Limit |
|---|---|---|---|---|
| VLAN plus ACL or firewall | L2 and L3 | Router, firewall | Zones | Coarse, tied to IP plans |
| VRF | L3 | Router | Fully separate routing tables | Many tables to manage |
| Private VLAN | L2 | Switch | Stop hosts in the same subnet talking | Small scope |
| VXLAN VNI with EVPN | Overlay | Leaf switch | Tenants in a data centre | Needs a fabric |
| Cisco TrustSec (SGT) | L2 and L3 tag | Switch, firewall | Policy by group, not IP | Needs vendor support |
| Hypervisor firewall | vNIC | Hypervisor | VMs, no agent in the guest | VMs only |
| Host agent | OS | iptables, nftables, Windows Filtering Platform | Mixed environments | Needs an agent on each host |
| Cloud security group | Instance or NIC | Cloud provider | Cloud workloads | Per-cloud rules |
| Kubernetes NetworkPolicy | Pod | CNI plugin | Containers | Plugin must enforce it |
| Service mesh | L7 | Sidecar or node proxy | Identity and mTLS between services | Added complexity |

Related chapters: [VLANs](../03-ethernet-and-switching/06-vlans.md), [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md), [Security Groups](../11-cloud-and-containers/04-cloud-security-and-connectivity.md), [Kubernetes Network Policies](../11-cloud-and-containers/09-kubernetes-ingress-and-policies.md), [Service Mesh](../11-cloud-and-containers/10-service-mesh-and-ebpf.md).

**How a microsegmentation project runs:**

1. **Discover.** Collect flow data for weeks. Learn who really talks to whom. Agent tools, NetFlow and cloud flow logs help.
2. **Label.** Tag each workload: role=db, app=shop, env=prod. Labels matter because IP addresses change.
3. **Write policy.** Default deny, with allow rules for real flows only.
4. **Test.** Run in alert-only mode. Log what would be blocked. Fix the gaps.
5. **Enforce.** Turn on blocking, one app at a time. Start with the most valuable systems.
6. **Maintain.** New apps need new rules. Tie this to change control and automation.

Kindervag calls that most valuable set the **protect surface**: the small group of critical data, apps and services. It is far smaller than the whole attack surface, so it is a sensible place to start.

**Why policy should follow labels, not addresses.** An IP rule such as "allow 10.50.2.10" breaks when autoscaling creates a new server. A label rule such as "allow role=app to role=db on 5432" keeps working. Also, the number of IP rules grows fast. Labels keep the policy small.

**Segmentation and compliance.** PCI DSS (the card payment security standard) does not require segmentation, but good segmentation can shrink which systems are in scope for audit. This is a common real-world reason for the project.

### Part 6: Network Access Control (NAC)

**NAC** means deciding which devices may join the network, and with what level of access. It answers: who are you, what is this device, and is it healthy?

Analogy: a bouncer at a club checks your ID, your age and the dress code. Then he gives you a wristband: general floor or VIP. It stops being true here: a bouncer checks you once at the door. A good NAC system can change your wristband while you are inside.

Common products: Cisco ISE, HPE Aruba ClearPass, Forescout, Fortinet FortiNAC. Their functions:

1. **Identify.** Find the user or the device, using 802.1X, MAC address or a web login.
2. **Profile.** Work out the device type. A profiler reads clues such as DHCP options, LLDP or CDP data, HTTP user agent and MAC vendor prefix. "This is a printer."
3. **Assess posture.** Check patches, disk encryption and antivirus. This usually needs an agent. Some tools can do a one-time check.
4. **Authorise.** Choose the result: a VLAN, an ACL or a security tag.
5. **Remediate.** Send an unhealthy device to a quarantine network where it can reach only patch servers.
6. **Monitor.** Keep logs. React to changes during the session.

**Pre-admission vs post-admission:**

- **Pre-admission** checks a device before it gets network access.
- **Post-admission** watches it after that. If the security tool reports malware, NAC can move the device to quarantine.

**Guest access.** A guest usually lands on a web page, called a **captive portal**, and gets internet-only access. That is web authentication, not 802.1X.

**Naming trap:** IEEE 802.1X is called "Port-Based Network Access Control". It is one tool. NAC products are full systems that use 802.1X and much else.

### Part 7: 802.1X in wired networks

**802.1X** is an IEEE standard for port-based access. A port stays closed until the device connected to it proves who it is. It works on wired switch ports and on Wi-Fi. This chapter covers wired. For Wi-Fi, see [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).

Analogy: a locked office door with a guard. You show your ID to the guard. The guard does not know you, so he phones head office and reads your ID out. Head office says yes, and tells him which floor you may use. It stops being true here: in 802.1X, the switch cannot read most of what passes through it. With EAP-TLS, the laptop and the server speak to each other through the switch, in a protected exchange.

**The three roles:**

| Role | Who | Job |
|---|---|---|
| Supplicant | Software on the laptop | Asks for access and gives proof |
| Authenticator | Switch | Holds the port closed, relays messages |
| Authentication server | RADIUS server, such as Cisco ISE, FreeRADIUS or Windows NPS | Decides yes or no |

**The protocols:**

- **EAP (Extensible Authentication Protocol, RFC 3748):** a framework that carries a login method. It does not define one single method.
- **EAPOL (EAP over LAN):** carries EAP inside Ethernet frames between laptop and switch. There is no IP address yet, so this must be Layer 2.
- **RADIUS (RFC 2865):** carries EAP inside UDP/IP between the switch and the server. It uses UDP 1812 for authentication and UDP 1813 for accounting. Older systems use 1645 and 1646.

```text
 Laptop              Switch               RADIUS server
 (supplicant)        (authenticator)      (auth server)
    |                    |                     |
    |<---- EAPOL ------->|<----- RADIUS ------>|
    |   Ethernet frames  |   UDP 1812 over IP  |
    |   EtherType 0x888E |                     |
    |                    |                     |
    |<======= EAP method runs end to end =====>|
```

**The steps:**

1. The laptop plugs in. The link comes up. The port is in the **unauthorized** state. Normal traffic is blocked, so there is no DHCP. On Cisco switches, only EAPOL, CDP and STP pass.
2. The switch sends **EAP-Request/Identity**. The laptop may also start by sending **EAPOL-Start**.
3. The laptop replies with **EAP-Response/Identity**, such as `alice@example.com`.
4. The switch wraps this in a RADIUS **Access-Request** and sends it to the server.
5. The server chooses a method and sends **Access-Challenge**. The EAP method now runs, with many round trips. The switch just relays.
6. The server sends **Access-Accept** or **Access-Reject**. An accept can carry extra orders: which VLAN, which ACL, when to log in again.
7. The switch sends **EAP-Success** to the laptop and moves the port to **authorized**. The laptop now does DHCP.
8. When the link drops, or on **EAPOL-Logoff**, the port closes again.

```text
 Laptop               Switch                  RADIUS 10.10.10.20
   |  link up           |                          |
   |<-EAP-Req/Identity--|                          |
   |--EAP-Resp/Identity>|                          |
   |                    |---Access-Request-------->|
   |                    |<--Access-Challenge-------|
   |<--EAP-Req/TLS------|                          |
   |--EAP-Resp/TLS----->|---Access-Request-------->|
   |   ... more rounds: TLS handshake inside EAP ...
   |                    |<--Access-Accept----------|
   |                    |   VLAN=20, timeout=28800 |
   |<--EAP-Success------|                          |
   |   port authorized  |                          |
   |---DHCP Discover--->|                          |
```

**The EAPOL frame on the wire:**

| Field | Size | Value or meaning |
|---|---|---|
| Destination MAC | 6 bytes | 01:80:C2:00:00:03 (PAE group address) |
| Source MAC | 6 bytes | MAC of the laptop or switch port |
| EtherType | 2 bytes | 0x888E |
| Protocol Version | 1 byte | Version of the 802.1X standard in use |
| Packet Type | 1 byte | 0 EAP, 1 Start, 2 Logoff, 3 Key, 5 MKA |
| Packet Body Length | 2 bytes | Length of what follows |
| EAP Code | 1 byte | 1 Request, 2 Response, 3 Success, 4 Failure |
| EAP Identifier | 1 byte | Matches a response to its request |
| EAP Length | 2 bytes | Length of the EAP packet |
| EAP Type | 1 byte | 1 Identity, 13 EAP-TLS, 25 PEAP, and so on |

"PAE" means port access entity, the 802.1X logic on a port. The group address has a useful property: normal bridges do not forward it, so EAPOL stays on one link.

**Common EAP methods:**

| Method | Proof from client | Server certificate | Notes |
|---|---|---|---|
| EAP-TLS | Client certificate | Yes | Strongest common choice. No password to steal. Needs PKI (a system for issuing certificates). RFC 5216, and RFC 9190 for TLS 1.3 |
| PEAP with MSCHAPv2 | Password | Yes | Common with Active Directory. Weak if the client does not validate the server certificate |
| EAP-TTLS | Password or others | Yes | Similar tunnel idea. RFC 5281 |
| EAP-FAST | Protected credential or password | Optional | Cisco origin. RFC 4851 |
| TEAP | Certificate, password or both | Yes | Newer standard tunnel. Can check machine and user in one exchange |
| EAP-MD5 | Password hash | No | Old. Do not use |

**Devices that cannot do 802.1X.** Printers, cameras and old phones often have no supplicant. **MAB (MAC Authentication Bypass)** handles them: the switch sends the device's MAC address to RADIUS as the login. This is weak, because a MAC address is easy to copy. Limit MAB devices with a tight ACL, and use profiling to notice when a "printer" starts acting like a laptop.

**What the RADIUS server can send back:**

- **Dynamic VLAN:** puts the port in the right VLAN, with attributes Tunnel-Type=VLAN, Tunnel-Medium-Type=802 and Tunnel-Private-Group-ID=20 (RFC 3580 covers this usage).
- **Downloadable ACL (dACL):** a Cisco feature in which the server pushes a per-session ACL.
- **Security group tag:** on TrustSec networks.
- **Session-Timeout:** when to authenticate again.

**Change of Authorization (CoA, RFC 5176).** Normally the switch asks and the server answers. CoA lets the server speak first, on UDP 3799. For example: "That laptop just failed its posture check. Move it to quarantine now." Post-admission NAC depends on this.

**Host modes (Cisco terms).** These control how many devices a port can serve:

| Mode | Behaviour |
|---|---|
| single-host | One device only |
| multi-host | First device logs in, then everyone on that port is allowed |
| multi-domain (MDA) | One data device and one voice device, such as a PC behind an IP phone |
| multi-auth | Many devices, each one authenticated separately |

Multi-host is weak. One valid login opens the port for everything behind a small switch.

### Part 8: ZTNA vs VPN

A traditional **remote access VPN** puts your device on the company network through an encrypted tunnel. See [VPNs](07-vpn.md) and [IPsec](08-ipsec.md).

**ZTNA (zero trust network access)** connects a user to one specific app, after checking identity and device. The user never joins the network.

Analogy: a VPN is a key to the office building. Once in, you can try every door. ZTNA is a hotel key card: it opens your room and the lift to your floor. Other guests' rooms do not even appear on your card. It stops being true here: with ZTNA, the other rooms are not just locked, they are invisible. You cannot find them with a scan.

**How most ZTNA works, the inside-out design:**

```text
 Remote laptop            ZTNA cloud            Company site
 +-----------+         +--------------+       +-------------+
 | Agent     |--TLS--->|   Broker     |<--TLS-| Connector   |
 +-----------+         | policy check |       | 10.50.2.20  |
                       +--------------+       +-------------+
                              |                     |
                       asks identity            wiki app
                       provider                 10.50.2.30
```

1. A small program called a **connector** runs near the app. It makes an **outbound** TLS connection to a **broker** and keeps it open.
2. So the site firewall needs no inbound rule. The app has no public address. Nothing listens for attackers on the internet.
3. The user requests the app. The agent, or a browser login, reaches the broker.
4. The broker sends the user to the identity provider and collects device health.
5. If the policy says yes, the broker joins the user's connection and the connector's connection.
6. The connector connects to that one app on that one port.
7. The check can repeat. Access can end during the session if something changes.

Forms of ZTNA:

- **Agent-based:** software on the device. It can carry many protocols (SSH, RDP, database) and can report posture.
- **Clientless (agentless):** a reverse proxy reached by browser, for web apps. Good for contractors and personal devices. Device checks are limited. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).
- **Self-hosted gateway:** a gateway sits in front of the apps. Some designs hide it with **single packet authorization**, an idea from the Cloud Security Alliance's Software-Defined Perimeter (SDP). The gateway stays silent until it gets a valid signed packet.

Products include Zscaler Private Access, Cloudflare Access, Palo Alto Prisma Access, Netskope Private Access, Microsoft Entra Private Access and Google's identity-aware proxy.

**VPN vs ZTNA:**

| Point | Remote access VPN | ZTNA |
|---|---|---|
| Access gives you | A place on the network | One app or service |
| Unit of trust | Network or subnet | User, device and app, per session |
| Device check | Often only at login, often weak | Part of each policy decision |
| Scanning internal hosts | Possible, unless restricted | Not possible, no route to them |
| Exposure | VPN gateway listens on the internet | Connector calls out, no inbound port |
| Lateral movement | Possible | Very limited |
| Traffic path | Often back through head office | Through nearby service point |
| Scale | Add appliance capacity | Service provider scales |
| Works for all protocols | Yes, it is IP | Best for TCP apps, varies for UDP and server-initiated traffic |
| Site-to-site links | Yes | No, not its job |
| Who runs it | You | Usually a cloud provider |

**The honest view.** A VPN can be restricted with per-group ACLs, and a modern VPN with posture checks is better than a flat one. ZTNA is not magic either. It adds a dependency on a cloud service, can raise latency if the nearest service point is far away, and may not fit old applications, such as those where the server calls the client. Many companies keep a small VPN for those cases, and for network engineers who need real network access.

Also: **ZTNA is a product type. Zero trust is an architecture.** Buying ZTNA does not make a company zero trust.

### Part 9: SWG, CASB, FWaaS, SSE and SASE

ZTNA protects access to private apps. Users also go to websites and SaaS apps. Modern companies need checks for those paths too, and they cannot send everything back through head office.

Terms first:

- **SD-WAN:** smart, software-controlled branch connectivity across many links. See [SD-WAN](../10-enterprise-and-datacenter/10-sd-wan.md).
- **SWG (secure web gateway):** a cloud proxy that filters web traffic. It blocks bad sites and malware, and applies URL rules.
- **CASB (cloud access security broker):** gives visibility and control over SaaS use. It finds unapproved apps (called **shadow IT**) and controls what users do in approved ones. It can work inline or through the SaaS provider's API.
- **FWaaS (firewall as a service):** a firewall run in the cloud.
- **DLP (data loss prevention):** finds and stops sensitive data leaving, such as card numbers in an upload.
- **RBI (remote browser isolation):** runs the risky website in a remote browser, and sends only safe rendering to the user.
- **PoP (point of presence):** a provider's site in a city, where your traffic enters its network.

**SASE (Secure Access Service Edge, said "sassy")** is a term Gartner introduced in 2019. It combines network and security into one cloud-delivered service:

```text
  SASE = SD-WAN (network) + SSE (security)

  SSE  = SWG + CASB + ZTNA, and usually FWaaS, DLP, RBI
```

**SSE (Security Service Edge)** is a term Gartner introduced in 2021. It is the security half of SASE, without the WAN part. A company that already has a good SD-WAN, or none at all, may buy SSE alone.

```text
  Branch office        Home user          Phone
  (SD-WAN edge)        (agent)            (agent)
       |                  |                  |
       +----- IPsec, TLS or QUIC tunnels ----+
                          |
                          v
              +-----------------------+
              |   Nearest SASE PoP    |
              |  FWaaS  SWG  CASB     |
              |  ZTNA   DLP  RBI      |
              |  one policy, one log  |
              +-----------------------+
                  |       |       |
                  v       v       v
              Internet   SaaS   Private apps
```

Analogy: in the old way, each branch office has its own security guard team, and all visitors to the outside world must travel to head office first. With SASE, you use a shared checkpoint network with a station in every city, and each person uses the nearest one. All stations follow the same rule book. It stops being true here: a checkpoint serves people one by one, but a PoP handles huge volumes of encrypted traffic, so it must decrypt to inspect it, and that creates its own privacy and trust questions.

**Why this design wins:**

- Before, a branch user's traffic to Microsoft 365 might go to head office, through the firewall, then out to the internet. This is called **backhaul**, and it adds delay.
- With SASE, the user goes to a nearby PoP, gets inspected, and goes straight out.
- Policy is the same in the office, at home and on the road.

**How to tell the terms apart:**

| Term | What it is | Includes network part? |
|---|---|---|
| Zero trust | A security model | Not a product |
| ZTNA | Access to private apps | No |
| SSE | Cloud security service set | No |
| SD-WAN | Branch network overlay | Yes, no full security stack |
| SASE | SSE plus SD-WAN, converged | Yes |

The fuller truth: SASE and SSE started as analyst market terms, not as protocols or IEEE or IETF standards. Vendors define them in slightly different ways. Some companies buy from one vendor (single-vendor SASE). Others join SD-WAN from one with SSE from another (dual-vendor SASE).

### Part 10: Identity as the perimeter

When the network wall disappears, the new border is **identity**: the question "who or what is this?" If location means nothing, the verified identity carries the trust.

Analogy: a passport. A traveller is not trusted because of the street they stand on. The passport, issued by a trusted government and checked at each border, is what counts. It stops being true here: a passport lasts for years, while a digital session should last hours. And a stolen digital session can be used from far away in seconds.

**The pieces:**

- **Identity provider (IdP):** the service that holds identities and performs logins, such as Microsoft Entra ID or Okta.
- **SSO (single sign-on):** one login, then many apps trust it. Every app sends the user to the IdP, so there is one place to apply policy and to shut off access.
- **SAML 2.0 and OpenID Connect (OIDC):** standard ways for an IdP to tell an app who the user is. OIDC is built on OAuth 2.0 (RFC 6749). See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).
- **MFA (multi-factor authentication):** proof from two or more types: something you know, have, or are.
- **Conditional access:** policy at login, such as "finance apps need a managed device and strong MFA".
- **PAM (privileged access management):** vaulting and tracking admin accounts.
- **JIT (just-in-time) access:** admin rights given for a short time, then removed. No permanent admin rights.

**Three kinds of identity:**

| Identity of | Proven by | Example |
|---|---|---|
| Person | Password plus MFA, passkey | Alice in the finance group |
| Device | Certificate, TPM, device management | Laptop LAP-117 is company-owned |
| Workload | Certificate, cloud role, SPIFFE ID | The `api` service in production |

A **TPM** is a security chip that protects keys inside a device. **SPIFFE** is an open standard for naming workloads and issuing them short-lived identity documents. **mTLS (mutual TLS)** is TLS in which both sides show a certificate.

**MFA strength is not equal:**

| Method | Resists phishing? |
|---|---|
| SMS code | No, and exposed to SIM swap |
| Authenticator app code | No, it can be typed into a fake site |
| Push approval | Weak, users can be tricked into approving |
| Push with number matching | Better, but not fully phishing resistant |
| FIDO2 security key or passkey | Yes, it is tied to the real site's domain |

**Identity is now the main target.** If identity is the perimeter, then attackers attack identity:

- **Phishing proxy (adversary in the middle):** a fake site relays the real login and steals the session cookie after MFA.
- **Push fatigue:** repeat prompts until the user taps approve. This featured in the Uber breach of 2022.
- **Help desk tricks:** an attacker phones support, and talks them into resetting MFA.
- **Token theft:** malware copies browser tokens from a device.

The defence is phishing-resistant MFA, short session lifetimes, tokens tied to devices, strict help desk identity checks, and alerts from the IdP. The IdP itself is a critical system: protect it like a domain controller.

### How the parts fit together

| Layer | Question | Tools |
|---|---|---|
| Identity | Who is asking? | IdP, SSO, MFA, PAM |
| Device | Is this device healthy and known? | MDM, EDR, certificates, NAC posture |
| Local network | May this device join this port? | 802.1X, MAB, NAC |
| Remote access | How do users reach private apps? | ZTNA, VPN |
| Internet and SaaS | Is this web or SaaS use safe? | SWG, CASB, DLP |
| Inside the network | How far can a problem spread? | Segmentation, microsegmentation |
| All of it | What happened? | Logs, SIEM, analytics |

**MDM** (mobile device management) sets and reports device configuration. **EDR** (endpoint detection and response) watches a device for attacks.

## Example

All four examples use one fictional company, Example Corp:

| Network | Purpose |
|---|---|
| 10.10.10.0/24 | Management: RADIUS 10.10.10.20, domain controller 10.10.10.30, bastion 10.10.10.40, switch 10.10.10.2 |
| 10.20.0.0/24 | VLAN 20, staff laptops, gateway 10.20.0.1 |
| 10.30.0.0/24 | VLAN 30, IP phones |
| 10.40.0.0/24 | VLAN 40, printers |
| 10.99.0.0/24 | VLAN 99, quarantine |
| 10.50.1.0/24 | Web tier, web01 is 10.50.1.10 |
| 10.50.2.0/24 | App tier, app01 is 10.50.2.10, ZTNA connector 10.50.2.20, wiki 10.50.2.30 |
| 10.50.3.0/24 | Data tier, db01 is 10.50.3.10 |

### Example 1: A policy decision, not a network location

Policy for the payroll app: allow if the user is in the finance group, used phishing-resistant MFA, the device is managed, and the EDR agent is healthy.

| Who | Group | MFA | Device | Where | Decision |
|---|---|---|---|---|---|
| Alice | finance | Passkey | Managed, healthy | Home, 198.51.100.7 | Allow |
| Alice | finance | Passkey | Personal laptop | Home | Deny |
| Alice | finance | Passkey | Managed, EDR stopped | Office, 10.20.0.57 | Deny, show fix steps |
| Bob | contractor | Passkey | Managed, healthy | Office, 10.20.0.84 | Deny |
| Carol | finance | SMS code | Managed, healthy | Office, 10.20.0.91 | Ask for stronger MFA |

Look at row 3 and row 4. Both are inside the office, and both are denied. Now look at row 1. It is on a home network, and allowed. That is the whole idea: location did not decide any row.

### Example 2: 802.1X on a Cisco access port

Goal: port Gi1/0/10 connects an IP phone with a PC behind it. Unknown devices stay in VLAN 99. RADIUS decides the real VLAN.

```text
! Cisco IOS or IOS XE, Catalyst access switch
! This uses the older command style. Newer IOS XE can also use
! "access-session" commands with control policies (IBNS 2.0).

aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius
aaa accounting dot1x default start-stop group radius
!
radius server ISE-1
 address ipv4 10.10.10.20 auth-port 1812 acct-port 1813
 key <shared-secret>
!
aaa server radius dynamic-author
 client 10.10.10.20 server-key <shared-secret>
!
dot1x system-auth-control
!
interface GigabitEthernet1/0/10
 description User port
 switchport mode access
 switchport access vlan 99
 switchport voice vlan 30
 authentication host-mode multi-domain
 authentication order dot1x mab
 authentication priority dot1x mab
 authentication port-control auto
 authentication event server dead action authorize vlan 20
 authentication event server alive action reinitialize
 authentication periodic
 authentication timer reauthenticate server
 mab
 dot1x pae authenticator
 dot1x timeout tx-period 10
 spanning-tree portfast
```

What the main lines do:

- `dot1x system-auth-control` turns on 802.1X for the switch.
- `aaa server radius dynamic-author` lets the RADIUS server send CoA messages.
- `authentication port-control auto` means the port needs authentication.
- `authentication order dot1x mab` tries 802.1X first, then MAB.
- `server dead action authorize vlan 20` is a **fail-open** choice. If RADIUS is down, users still get VLAN 20. Business keeps running, but security is reduced for that time. Some sites use a limited VLAN instead.
- `dot1x timeout tx-period 10` shortens how long the switch waits between identity requests.

The RADIUS policy on the server returns this for staff laptops:

```text
Access-Accept
  Tunnel-Type = VLAN (13)
  Tunnel-Medium-Type = IEEE-802 (6)
  Tunnel-Private-Group-ID = "20"
  Session-Timeout = 28800
```

Client side, on Windows. The service that handles wired 802.1X is **Wired AutoConfig**, and it is not always running:

```powershell
# Windows 11, run as Administrator
Set-Service dot3svc -StartupType Automatic
Start-Service dot3svc
netsh lan show interfaces
Get-WinEvent -LogName "Microsoft-Windows-Wired-AutoConfig/Operational" -MaxEvents 5
```

Client side, on Linux, with `wpa_supplicant` and EAP-TLS:

```text
# /etc/wpa_supplicant/wired.conf
ctrl_interface=/run/wpa_supplicant
ap_scan=0
network={
    key_mgmt=IEEE8021X
    eap=TLS
    identity="alice@example.com"
    ca_cert="/etc/ssl/corp/ca.pem"
    client_cert="/etc/ssl/corp/alice.pem"
    private_key="/etc/ssl/corp/alice.key"
}
```

```bash
# Linux, run as root. -D wired selects the wired driver.
wpa_supplicant -i eth0 -D wired -c /etc/wpa_supplicant/wired.conf
```

```text
eth0: Associated with 01:80:c2:00:00:03
eth0: CTRL-EVENT-EAP-STARTED EAP authentication started
eth0: CTRL-EVENT-EAP-METHOD EAP vendor 0 method 13 (TLS) selected
eth0: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
eth0: CTRL-EVENT-CONNECTED - Connection to 01:80:c2:00:00:03 completed
```

Notice the address `01:80:c2:00:00:03`. It is the PAE group address from the frame table, and method 13 is EAP-TLS.

Check on the switch. The layout of this output varies by IOS version. On newer versions, also try `show access-session`.

```text
Switch# show authentication sessions interface Gi1/0/10 details
          Interface:  GigabitEthernet1/0/10
        MAC Address:  001a.2b3c.4d5e
       IPv4 Address:  10.20.0.57
          User-Name:  alice@example.com
             Status:  Authorized
             Domain:  DATA
     Oper host mode:  multi-domain
   Oper control dir:  both
    Session timeout:  28800s (server), Remaining: 28112s
     Timeout action:  Reauthenticate
    Server Policies:
        Vlan Group:  Vlan: 20

Method status list:
       Method           State
       dot1x            Authc Success
```

Now suppose the posture agent reports that EDR has stopped on this laptop:

1. The NAC server sends a CoA to the switch on UDP 3799.
2. The switch re-authorizes the session.
3. The server now answers with VLAN 99 and a small ACL: DNS and the patch server only.
4. The laptop must get a new address in 10.99.0.0/24. This is why a VLAN change is messy, and why many designs prefer an ACL or tag change that keeps the same IP.
5. When EDR is back, another CoA restores access.

To test the RADIUS side alone, without a switch, use `eapol_test` (from the wpa_supplicant project):

```bash
# Linux. Sends the EAP login straight to the RADIUS server.
eapol_test -c /etc/wpa_supplicant/wired.conf -a 10.10.10.20 -p 1812 -s '<shared-secret>'
```

The last line of its output is `SUCCESS` or `FAILURE`. This tells you quickly whether a fault lives in the server and certificates, or in the switch and client.

**Quick 802.1X troubleshooting:**

| Symptom | Likely cause |
|---|---|
| Port stays unauthorized, no EAP traffic | Client has no supplicant running, so wait for MAB |
| Long delay, then device works | Device fell back to MAB after the 802.1X timeout |
| Access-Reject in the server log | Wrong password, expired certificate, wrong policy |
| Client refuses the server | Client does not trust the server's CA |
| Authenticated but no IP | Assigned VLAN has no DHCP scope or is not on the switch |
| Everyone fails at once | Switch cannot reach RADIUS, or shared secret mismatch |

### Example 3: Microsegmenting a three-tier shop app

**Before.** The web, app and data subnets route freely to each other. If an attacker takes web01, they can try any port on all 768 addresses in the three /24 subnets.

**After.** Allow only real flows:

| Source | Destination | Port | Action |
|---|---|---|---|
| Users 10.20.0.0/24 | web01 10.50.1.10 | TCP 443 | Allow |
| web01 10.50.1.10 | app01 10.50.2.10 | TCP 8443 | Allow |
| app01 10.50.2.10 | db01 10.50.3.10 | TCP 5432 | Allow |
| Bastion 10.10.10.40 | All three | TCP 22 | Allow |
| Anything else | Anything else | Any | Deny and log |

```text
 Users --443--> web01 --8443--> app01 --5432--> db01
                  |                               ^
                  +------------- X ---------------+
                       web01 cannot reach db01
```

A **bastion** is one hardened host that admins go through. It gives one controlled entry point for SSH.

Coarse rule on the Cisco VLAN 20 gateway. Staff may reach web01 on 443, not other internal hosts. They keep internet access:

```text
! Cisco IOS, on the VLAN 20 gateway
ip access-list extended STAFF-IN
 permit udp any eq bootpc any eq bootps
 permit udp 10.20.0.0 0.0.0.255 host 10.10.10.30 eq domain
 permit tcp 10.20.0.0 0.0.0.255 host 10.50.1.10 eq 443
 deny ip 10.20.0.0 0.0.0.255 10.0.0.0 0.255.255.255 log
 permit ip 10.20.0.0 0.0.0.255 any
!
interface Vlan20
 ip access-group STAFF-IN in
```

Fine rule on the host itself. This protects db01 even from a neighbour in the same subnet, where a router ACL cannot help:

```bash
# Linux db01 (10.50.3.10), nftables, run as root from the console.
# Add the SSH rule before you depend on it, or you may lock yourself out.
nft add table inet seg
nft add chain inet seg input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet seg input ct state established,related accept
nft add rule inet seg input iif lo accept
nft add rule inet seg input ip saddr 10.10.10.40 tcp dport 22 accept
nft add rule inet seg input ip saddr 10.50.2.10 tcp dport 5432 accept
nft add rule inet seg input limit rate 5/second log prefix '"seg-drop: "'
```

```powershell
# Windows Server with the same role, PowerShell as Administrator
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultInboundAction Block
New-NetFirewallRule -DisplayName "app01 to Postgres" -Direction Inbound `
  -Action Allow -Protocol TCP -LocalPort 5432 -RemoteAddress 10.50.2.10
New-NetFirewallRule -DisplayName "Bastion SSH" -Direction Inbound `
  -Action Allow -Protocol TCP -LocalPort 22 -RemoteAddress 10.10.10.40
```

See [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md) for the details of these rules.

Test it:

```bash
# On app01 (10.50.2.10): allowed
$ nc -vz -w 3 10.50.3.10 5432
Connection to 10.50.3.10 5432 port [tcp/postgresql] succeeded!

# On web01 (10.50.1.10): blocked
$ nc -vz -w 3 10.50.3.10 5432
nc: connect to 10.50.3.10 port 5432 (tcp) timed out: Operation now in progress
```

The drop also leaves a clue on db01. This is valuable, because a web server trying to reach a database directly may be an attacker:

```text
seg-drop: IN=eth0 OUT= SRC=10.50.1.10 DST=10.50.3.10 PROTO=TCP
SPT=48122 DPT=5432 SYN
```

(The real log line has more fields, shortened here.)

**The measured result:**

| If web01 falls | Before | After |
|---|---|---|
| Reachable internal addresses | 768 | 1 (app01) |
| Reachable ports on each | All | 1 (8443) |
| Direct path to the database | Yes | No |

The same idea in Kubernetes. Only pods labelled `app=api` may reach pods labelled `app=db`:

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-api
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

This uses labels, not addresses, so it survives pods being replaced. It only works if the cluster's network plugin enforces NetworkPolicy.

### Example 4: Replacing the VPN with ZTNA for the wiki

Alice works from home. Her laptop is 192.168.1.50 behind home NAT, with public address 198.51.100.7. She needs `https://wiki.example.com`, hosted on 10.50.2.30.

**With a full VPN.** She gets a tunnel address of 10.200.0.25, and a route to all of 10.0.0.0/8.

```bash
# Linux laptop on the VPN
$ ip route | grep tun0
10.0.0.0/8 dev tun0 scope link
$ nmap -sn 10.50.1.0/24 10.50.2.0/24 10.50.3.0/24 | tail -1
Nmap done: 768 IP addresses (41 hosts up) scanned in 9.87 seconds
```

She needed one app, and she can see 41 hosts. If malware runs on her laptop, it can see them too.

**With ZTNA:**

```text
 Alice's laptop       Broker 203.0.113.10       Connector 10.50.2.20
      |                     |                         |
      |                     |<==== outbound TLS 443 ==|  (1)
      |--- wiki request --->|                         |  (2)
      |<-- go to IdP -------|                         |  (3)
      |    passkey login, device check                |
      |--- signed token --->|                         |  (4)
      |                 policy: allow                 |
      |<=== joined tunnel ===========================>|  (5)
      |                                       TCP 443 |
      |                                               v
      |                                   wiki 10.50.2.30
```

1. The connector dials out to the broker on TCP 443 and stays connected. The site firewall has no inbound rule for it.
2. Alice opens the wiki in her browser. The agent catches the DNS name. Many agents answer with a local placeholder address and send that traffic into their tunnel.
3. The broker sends her to the IdP. She logs in with a passkey.
4. The agent reports device health. The policy engine checks group, device and risk.
5. The broker joins the two sides. The connector opens TCP 443 to 10.50.2.30 only.

Result:

```bash
# Same Linux laptop, ZTNA agent running, VPN off
$ ip route | grep -c '^10\.'
0
$ curl -sI https://wiki.example.com | head -1
HTTP/2 200
$ nmap -sn 10.50.1.0/24 10.50.2.0/24 10.50.3.0/24 | tail -1
Nmap done: 768 IP addresses (0 hosts up) scanned in 58.02 seconds
```

| Measure | VPN | ZTNA |
|---|---|---|
| Hosts Alice can reach | 41 discovered, potentially all of 10.0.0.0/8 | 1 |
| Inbound ports open at site | At least the VPN port | None for this path |
| Device check | At login only | At each policy decision |
| Internal network shown to home malware | Yes | No |

The fuller truth: if Alice's account or laptop is fully taken over while her session is valid, the attacker can still use the wiki. ZTNA reduces reach. It does not make a compromised user safe. That is why logs, session limits and behaviour checks still matter.

## Going Deeper

### Trust algorithms and failure design

NIST describes the Policy Engine's logic as a **trust algorithm**. Two choices shape it:

- **Criteria-based vs score-based.** Criteria-based means a fixed list of conditions that must all be true. Score-based turns the signals into a confidence number, and each resource sets a threshold.
- **Singular vs contextual.** Singular treats each request alone. Contextual also looks at recent behaviour, such as an account that suddenly reads 500 files.

**The PDP becomes critical.** If the policy service is down, no one gets in, or everyone does. You must choose **fail-closed** or **fail-open** for each system. A payment system and a hospital records system may need different answers. Design the PDP for high availability, and plan a tested emergency login, often called a **break-glass account**.

### Maturity and migration

CISA's Zero Trust Maturity Model has five pillars: Identity, Devices, Networks, Applications and Workloads, and Data. Three capabilities run across them: Visibility and Analytics, Automation and Orchestration, and Governance. There are four stages: Traditional, Initial, Advanced and Optimal.

A realistic migration order:

1. List users, devices, apps and data. You cannot protect what you cannot name.
2. Move logins to SSO with phishing-resistant MFA.
3. Make device health part of the login decision.
4. Map traffic flows.
5. Move one app at a time behind ZTNA. Begin with low-risk web apps.
6. Microsegment the protect surface first.
7. Shrink the VPN to the few cases that need it, then keep shrinking.
8. Feed logs into a SIEM, then automate responses.

Big-bang projects usually fail. Each step should reduce risk on its own.

### Securing 802.1X itself

**Deployment phases.** Turning on enforcement in one day can cut off a whole site. A common Cisco-recommended path:

| Phase | Behaviour | Why |
|---|---|---|
| Monitor mode | `authentication open`, traffic flows whether login passes or fails | See who would fail, no outage |
| Low-impact mode | Open plus a pre-auth ACL that permits only DHCP, DNS and similar | Limited access before login |
| Closed mode | Nothing passes before login, except EAPOL | Full enforcement |

**Timing problems.** By default on many Cisco platforms, the wait before MAB starts is (max-reauth-req + 1) x tx-period. With defaults of 2 and 30 seconds, that is 90 seconds. A printer may give up on DHCP before then, and a PXE network boot may time out. So admins lower tx-period, or use `authentication order mab dot1x`. These defaults vary by platform and version, so check yours.

**Machine and user authentication.** A laptop can log in as a machine at boot, so group policy and domain login work, then as a user after sign-in. This matters because without machine login, the port stays closed before the user signs in. TEAP can prove both together, which plain EAP-TLS or PEAP cannot do in one exchange.

**Attacks on 802.1X:**

- **MAC spoofing on MAB ports.** Copy the printer's MAC address from its label. Mitigate with a tight ACL and profiling.
- **Bump in the wire.** 802.1X opens the port once. It does not protect later frames. An attacker with a small device between a real PC and the switch can let the PC log in, then slip traffic in using the PC's addresses. **MACsec** stops this.
- **Fake server.** A client that does not validate the server certificate may give credentials to a rogue server. Configure the trusted CA and the server name on clients.
- **Multi-host ports.** One login opens the port for all devices behind it.

**MACsec (IEEE 802.1AE)** encrypts and authenticates each Ethernet frame, hop by hop, between two directly connected devices. The keys are agreed with **MKA**, carried in EAPOL frames (the EAPOL packet type 5 from the table). With MACsec, a device in the middle cannot inject frames.

**Securing RADIUS.** Classic RADIUS relies on a shared secret and MD5, and carries its traffic over UDP. In 2024, the **BlastRADIUS** flaw (CVE-2024-3596) showed that an attacker on the path could forge a server's reply in some setups. The fixes are to update servers and clients and to require the Message-Authenticator attribute. Keep RADIUS on a protected management network, or use **RadSec** (RADIUS over TLS, RFC 6614).

**RADIUS vs TACACS+.** RADIUS is the usual choice for network access (users and devices joining). TACACS+ is the usual choice for device administration (who may run which commands on a router). See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

### Cisco TrustSec and group-based policy

Traditional ACLs grow with the number of addresses. TrustSec uses a different key. Each user or device gets a **security group tag (SGT)**, a 16-bit value, usually assigned by the RADIUS server after login. Switches can carry that tag inside frames, or share IP-to-tag maps with **SXP** where hardware cannot tag.

The policy then reads as a matrix, not a list of IPs:

```text
               Destination
 Source        Payroll(50)  Printers(40)  Guests(60)
 Finance(10)   allow 443    allow 9100    deny
 Staff(20)     deny         allow 9100    deny
 Guests(60)    deny         deny          deny
```

You can add a thousand Finance users without changing one rule. Similar group-based models exist in other vendors' fabrics.

### Microsegmentation pitfalls

- **Shared services.** DNS, NTP, Active Directory, backup and monitoring talk to everything. Plan these first, or they break first.
- **Unseen dependencies.** A monthly batch job does not appear in a one-week flow capture. Keep the alert-only phase long enough.
- **Hosts that cannot run agents.** Old systems and operational technology (factory or building control systems) may not accept an agent, and may fail if scanned. Use a gateway or network ACL around them.
- **Agent trust.** If an attacker gets root on a host, they may turn off its own firewall. Enforcement outside the host (hypervisor, network, cloud) is harder to tamper with. Strong designs layer both.
- **Allowed paths are still paths.** Rules that allow SMB or RDP widely undo the design. Attackers love these ports.

### Continuous evaluation

A token issued at login may stay valid for an hour or more. If a user is fired in that hour, older designs wait for it to expire. Newer approaches send events between systems so that access ends in near real time. Examples are Microsoft's Continuous Access Evaluation, and the OpenID Foundation's Shared Signals framework. Sender-constrained tokens, such as DPoP (RFC 9449), tie a token to a key, so a stolen token alone is less useful.

### Workload identity

Service-to-service traffic needs zero trust too, and IP addresses are poor identities in systems with short-lived containers. A service mesh or SPIFFE-based system gives each workload a short-lived certificate, and uses mTLS on each call. Policy then says "`checkout` may call `payments`", whatever IP they have today. Non-human identities (service accounts, API keys, cloud roles) often outnumber people, and often hold too much access. See [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md).

### Encryption and visibility

An SWG cannot filter what it cannot read. To inspect HTTPS, it acts as a **TLS-intercepting proxy**: it ends the user's TLS, reads the content, and opens a new TLS session to the site. The endpoint must trust the company's CA for this. Costs and issues:

- It breaks apps that use certificate pinning, so they need bypass rules.
- It creates privacy and legal issues, so sites often exempt banking and health categories.
- The proxy becomes a very sensitive system, because it sees everything in clear text.

Without decryption, tools still see the server name in many TLS handshakes, but Encrypted Client Hello (ECH) reduces that.

### The trust you still have

"Zero trust" is a slogan. There is always trust somewhere. You trust the IdP, the certificate authority, the ZTNA provider, the device's hardware and the people who write policy. The honest goal is to make trust **explicit, small, and watched**, not to remove it. Good candidates say this in interviews.

## Common Mistakes

- **Thinking zero trust is a product.** A vendor can sell parts. Zero trust is an architecture and a way of operating.
- **Thinking zero trust means no trust.** It means no implicit trust from location. Trust is earned and limited.
- **Saying VLANs are a security boundary on their own.** A VLAN separates broadcast domains. Security comes from the firewall or ACL between VLANs, and from a secure trunk setup. See [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).
- **Saying "VPN is secure because it is encrypted".** Encryption protects data in transit. It does nothing about what a logged-in user or malware can reach.
- **Thinking ZTNA replaces every VPN.** Site-to-site IPsec links and some legacy apps still need network-level connectivity.
- **Mixing up SASE and SSE.** SASE includes the network part (SD-WAN). SSE is the security part only.
- **Saying 802.1X encrypts traffic.** Wired 802.1X authenticates at the port. Encryption on the wire needs MACsec. On Wi-Fi, the login result feeds the key process.
- **Thinking the switch checks the password.** The switch only relays EAP. The RADIUS server decides.
- **Thinking MAB is authentication.** A MAC address is not a secret. Treat MAB devices as weakly identified.
- **Treating EAPOL as an IP protocol.** It runs directly over Ethernet, because the client has no IP address yet.
- **Enforcing on day one.** Skipping monitor mode or alert-only mode causes outages and ends projects.
- **Writing microsegmentation rules with no flow data.** You will break unknown dependencies.
- **Forgetting east-west logs.** A segmented network with no alerts on denied traffic misses the best clues.
- **Treating MFA as finished.** SMS and push can be phished or tricked. Sessions can be stolen after MFA.
- **Ignoring non-human identities.** A service account with broad rights and an old password is a perfect target.
- **Assuming a VPN inside a zero trust plan is forbidden.** A VPN can be one transport. The test is whether access is narrow and policy-driven.

## Interview Questions

**Q: What is the perimeter security model, and why did it fail?**
A: It puts security at the network edge and trusts what is inside. It failed because users and apps moved outside that edge, attackers now log in with stolen credentials, and a flat inside network allows easy lateral movement. One small breach could reach everything.

**Q: Explain zero trust in simple terms.**
A: No access is granted because of network location. Each session is checked against identity, device health and policy, with least privilege, and we design assuming a breach has already happened. NIST SP 800-207 is the main reference. It is an architecture, not a product.

**Q: What are the Policy Decision Point and Policy Enforcement Point?**
A: The PDP decides. It is made of the Policy Engine and the Policy Administrator, and it uses signals from the identity provider, device tools and threat data. The PEP enforces, by allowing or blocking the traffic. It can be a proxy, a switch port, a firewall or a host agent. Separating them means policy is central but enforcement sits close to the resource.

**Q: What is the difference between segmentation and microsegmentation?**
A: Segmentation splits a network into large zones, such as users, servers and guests, usually with VLANs and firewalls. Microsegmentation controls traffic around each workload, usually by label, with host firewalls, hypervisors, cloud security groups or Kubernetes policies. It stops lateral movement even inside one subnet.

**Q: Walk me through 802.1X on a wired port.**
A: There are three roles: the supplicant on the client, the authenticator on the switch, and the RADIUS server. The port starts unauthorized. The client and switch use EAPOL over Ethernet, the switch relays EAP to RADIUS over UDP 1812, and the EAP method runs between client and server. On Access-Accept, the switch authorizes the port and can apply a VLAN or ACL from the server. Only then does the client do DHCP.

**Q: How do you handle a printer that cannot do 802.1X?**
A: I use MAC Authentication Bypass, in which the switch sends the MAC address to RADIUS as the login. Because a MAC address can be copied, I add profiling, put the device in its own VLAN and use a tight ACL so it reaches only the print server. That limits the damage if someone spoofs it.

**Q: What does NAC do beyond 802.1X?**
A: 802.1X is the authentication mechanism. A NAC system adds device profiling, posture checks, guest onboarding, quarantine and remediation, plus central policy. With RADIUS Change of Authorization, it can change a device's access during its session, for example moving it to quarantine after a bad posture result.

**Q: Why is ZTNA considered better than VPN for remote access?**
A: A VPN puts the user on the network, so the user or any malware on the device can often reach many hosts. ZTNA gives access to one app after checking identity and device, and with the common connector design the app has no inbound exposure. The limits are legacy protocols, dependency on the provider and cost, and it does not replace site-to-site VPNs.

**Q: What is the difference between SASE and SSE?**
A: SASE is a converged cloud service that combines networking and security, so SD-WAN plus security. SSE is just the security part: usually SWG, CASB and ZTNA, often with FWaaS and DLP. Both are Gartner-defined categories, not protocols.

**Q: What does "identity is the new perimeter" mean, and what is its weakness?**
A: Since apps and users are everywhere, the checkpoint moves from the network edge to the identity system. Every app relies on the IdP, so SSO, MFA and conditional access enforce policy. The weakness is that attackers now target identity with phishing proxies, stolen session tokens and help desk tricks, so we need phishing-resistant MFA like passkeys, short sessions and strong IdP protection.

**Q: A compromised web server tries to reach the database directly. How would your design stop and detect this?**
A: Default deny around the data tier. Only the app tier is allowed to reach the database port, enforced on the database host and at a point outside the host. The web server's attempt is dropped and logged, and an alert fires, because that flow should never occur. This limits the blast radius and gives early warning.

**Q: Your company wants zero trust. Where do you start?**
A: I would not start by buying a box. I would inventory users, devices, apps and data, then put SSO and phishing-resistant MFA in place. After that, I would add device health to access decisions, map flows, move apps one at a time to ZTNA, and microsegment the most critical systems first, using alert-only mode before enforcement. Each step should reduce risk on its own.

## Quick Recap

- The perimeter model trusts what is inside, and it fails because of cloud, remote work, stolen credentials and lateral movement.
- Zero trust removes trust based on location: verify explicitly, use least privilege, assume breach. NIST SP 800-207 defines it, with a PDP deciding and a PEP enforcing.
- Segmentation and microsegmentation limit the blast radius. Default deny, label-based rules, and alert-only mode before enforcement are the keys.
- NAC controls who joins the network and with what access. 802.1X uses a supplicant, an authenticator and a RADIUS server, with EAPOL on the link and RADIUS on UDP 1812. MAB covers devices with no supplicant, and CoA changes access mid-session.
- A VPN gives a place on the network. ZTNA gives access to one app, usually through a connector that calls outbound to a broker.
- SASE is SD-WAN plus SSE. SSE is SWG, CASB and ZTNA, often with FWaaS and DLP, delivered from cloud PoPs.
- Identity is the new perimeter, so it is the new target: use phishing-resistant MFA, short sessions and workload identity, and keep trust explicit, small and watched.
