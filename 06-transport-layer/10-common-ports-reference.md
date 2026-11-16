# Common Ports Reference

**Level:** Beginner | **Prerequisites:** [Ports and Sockets](01-ports-and-sockets.md), [TCP vs UDP](02-tcp-vs-udp.md)

A port is a number from 0 to 65535 that tells a computer which program should receive a piece of network traffic. Interviewers ask about ports all the time, because port knowledge shows up in every firewall rule, every packet capture and every outage.

## How to Read This Page

- **Port:** a number that points to one service on a machine. Think of an IP address as a building, and a port as a flat number inside it. The analogy stops here: one machine has two separate sets of numbers, one for TCP and one for UDP. TCP 53 and UDP 53 are two different "flats".
- **Transport:** the protocol that carries the data. TCP is reliable and sets up a connection first. UDP sends each message with no connection and no delivery promise.
- **Service:** a program that waits for traffic on a port, such as a web server on 443.
- **TLS:** Transport Layer Security, the encryption layer behind HTTPS and many other secure protocols.
- **STARTTLS:** a command that upgrades a plain connection to TLS on the same port.
- **QUIC:** a newer encrypted transport built on top of UDP.
- **IPsec:** a set of protocols that encrypt IP packets, often used for VPNs.
- **VPN:** Virtual Private Network, an encrypted tunnel across another network.
- **NAT:** Network Address Translation, where a router rewrites addresses so many devices can share one public IP.
- **IANA:** the Internet Assigned Numbers Authority. It keeps the official list of port numbers.
- **Vendor default:** a port a product picked for itself. It may or may not be on the IANA list.

A port number is a convention, not a law. You can run SSH on 443 or a web server on 2222. The numbers here are defaults. Real networks sometimes change them.

Some memory tricks below are silly on purpose. Silly things stick. A few are based on real facts, and those are marked as true stories.

## Port Number Ranges

The port field in TCP and UDP headers is 16 bits. That gives 2^16 = 65,536 values, numbered 0 to 65535. IANA splits them into three ranges (RFC 6335).

```text
0            1023 1024                 49151 49152           65535
+----------------+--------------------------+-------------------+
|  Well-known    |  Registered              |  Dynamic          |
|  (system)      |  (user)                  |  (ephemeral)      |
+----------------+--------------------------+-------------------+
 SSH 22, DNS 53     MySQL 3306, RDP 3389       client side ports
 HTTPS 443          Redis 6379, Kafka 9092     such as 50124
```

| Range | Name | Used for |
|---|---|---|
| 0-1023 | Well-known | Core services. On Linux, a normal user cannot listen on these without extra rights. |
| 1024-49151 | Registered | Products and applications that registered a number with IANA. |
| 49152-65535 | Dynamic, private or ephemeral | Short-lived client ports. IANA never assigns these to a service. |

Port 0 is reserved. It is not used on the wire by normal traffic. In programming, binding to port 0 means "operating system, pick a free port for me".

An ephemeral port is a temporary source port the client's operating system picks for one connection. The real ranges differ from the IANA range:

| System | Default ephemeral range | How to check |
|---|---|---|
| Linux | 32768-60999 | `cat /proc/sys/net/ipv4/ip_local_port_range` |
| Windows (Vista and later) | 49152-65535 | `netsh int ipv4 show dynamicport tcp` |

## The Core Reference Table

Sorted by port number, so you can look up a number fast. "Both" means TCP and UDP.

| Port | Transport | Full name and job | Memory trick |
|---|---|---|---|
| 20 | TCP | FTP data: File Transfer Protocol, carries file bytes in active mode | Blackjack table: 21 is the dealer in control, 20 is the cards moving |
| 21 | TCP | FTP control: login and commands | F, S, T in alphabet order give 21, 22, 23 |
| 22 | TCP | SSH, Secure Shell: encrypted remote login. SFTP and SCP run on it too | Two S letters, two 2 digits |
| 23 | TCP | Telnet: remote login with no encryption | T comes right after S, so 23 follows 22 |
| 25 | TCP | SMTP, Simple Mail Transfer Protocol: mail moving between servers | A 25-cent stamp on a letter |
| 49 | TCP | TACACS+: Terminal Access Controller Access-Control System Plus, admin login control for network devices | San Francisco 49ers, and Cisco takes its name from San Francisco |
| 53 | Both | DNS, Domain Name System: turns names into IP addresses | 5 looks like S, 3 looks like a backward E: SE for SEarch |
| 67 | UDP | DHCP server: Dynamic Host Configuration Protocol, gives out IP addresses | Server wakes first, so it gets the lower number |
| 68 | UDP | DHCP client | Client is server plus one |
| 69 | UDP | TFTP, Trivial File Transfer Protocol: small file transfer, no login | 67, 68, 69 run together. In network boot, DHCP gives the address and TFTP gives the boot file |
| 80 | TCP | HTTP, Hypertext Transfer Protocol: web with no encryption | HTTP has 4 letters, 4 x 20 = 80 |
| 88 | Both | Kerberos: ticket-based login, used by Active Directory | 88 looks like two fat guards at a door |
| 110 | TCP | POP3, Post Office Protocol 3: download mail to one device | One, one, then zero left on the server, because POP normally removes the mail |
| 123 | UDP | NTP, Network Time Protocol: keeps clocks correct | Count the clock: 1, 2, 3 |
| 135 | TCP | Microsoft RPC endpoint mapper: tells clients which port a Windows service is on | Remote Procedure Call, "1-3-5 odd steps to find the door" |
| 137 | UDP | NetBIOS Name Service | Meeting someone: learn the name (137), pass a note (138), sit and talk (139) |
| 138 | UDP | NetBIOS Datagram Service | Datagram is the pass-a-note step |
| 139 | TCP | NetBIOS Session Service: older way to carry SMB | Session is the sit-and-talk step |
| 143 | TCP | IMAP, Internet Message Access Protocol: mail stays on the server, synced across devices | "I love you" is 1, 4, 3 letters |
| 161 | UDP | SNMP, Simple Network Management Protocol: manager polls the device here | Manager asks on 161 |
| 162 | UDP | SNMP trap: the device sends alerts to the manager here | Device shouts one door up, on 162 |
| 179 | TCP | BGP, Border Gateway Protocol: routing between networks on the internet | Three odd digits for the odd politics of internet routing |
| 389 | TCP (UDP for CLDAP) | LDAP, Lightweight Directory Access Protocol: looks up users and groups | RDP 3389 is LDAP with a 3 in front |
| 443 | TCP, plus UDP for HTTP/3 | HTTPS: HTTP over TLS | 4-4-3: the words HTTP, SAFE, TLS have 4, 4, 3 letters |
| 445 | TCP | SMB, Server Message Block: Windows file and printer sharing | Two doors after HTTPS |
| 464 | Both | Kerberos password change | One below SMTPS 465 |
| 465 | TCP | SMTPS or "submissions": mail client sends mail, TLS from the first byte | Client mail ports go up then down: 4-6-5 and 5-8-7 |
| 500 | UDP | IKE, Internet Key Exchange: sets up IPsec VPN keys | "Ike" was Eisenhower, a five-star general (true story) |
| 514 | UDP | Syslog: sends log messages to a collector | 5 looks like S, 1 looks like L: SysLog, 4 everyone |
| 587 | TCP | Mail submission: mail client hands mail to its server, usually with STARTTLS | Up then down: 5-8-7. Clients use this, servers use 25 |
| 636 | TCP | LDAPS: LDAP over TLS | A mirror number: same both ways, Secure and Symmetric |
| 853 | TCP (UDP for DoQ) | DoT, DNS over TLS. DoQ, DNS over QUIC, uses the same number on UDP | An 8 in front of DNS 53 |
| 989, 990 | TCP | FTPS, FTP over TLS with TLS from the start: 989 data, 990 control | Data comes just before control, as in plain FTP (20, 21). 990 is the one people mean |
| 993 | TCP | IMAPS: IMAP over TLS | Alphabet: I before P, so 993 before 995 |
| 995 | TCP | POP3S: POP3 over TLS | The P comes second, so 995 |
| 1194 | UDP by default, TCP possible | OpenVPN | Say it as a year, eleven ninety-four. Picture a secret castle tunnel |
| 1433 | TCP | Microsoft SQL Server, default instance | IMAP 143 with one more 3 |
| 1434 | UDP | SQL Server Browser: tells clients the port of a named instance | 1433 plus one |
| 1521 | TCP | Oracle Database listener (vendor default) | Ages 15 to 21: go ask the oracle |
| 1701 | UDP | L2TP, Layer 2 Tunneling Protocol | Star Trek ship NCC-1701 |
| 1723 | TCP, plus GRE | PPTP, Point-to-Point Tunneling Protocol: old and broken VPN | Sounds like an old date, and PPTP is old too |
| 1812 | UDP | RADIUS authentication: Remote Authentication Dial-In User Service | The 1812 Overture and the War of 1812 |
| 1813 | UDP | RADIUS accounting | Next door: login first, then count usage |
| 2049 | Both, TCP in modern use | NFS, Network File System | Blade Runner 2049: files of the future |
| 3306 | TCP | MySQL and MariaDB | 3 + 3 = 6, and it ends in 6 |
| 3389 | TCP and UDP | RDP, Remote Desktop Protocol | LDAP 389 with a 3 in front |
| 4500 | UDP | IPsec NAT traversal: VPN packets wrapped so they survive NAT | 500 with a 4 in front: "4 NAT" |
| 5060 | Both | SIP, Session Initiation Protocol: sets up voice calls | Count by tens: 50, 60 |
| 5061 | TCP | SIP over TLS | Secure is plus one |
| 5432 | TCP | PostgreSQL | Countdown: 5, 4, 3, 2 |
| 5672 | TCP | AMQP, Advanced Message Queuing Protocol: used by RabbitMQ | Rabbit hops 5, 6, 7, then falls back to 2 |
| 6379 | TCP | Redis: in-memory data store | Phone keypad spells MERZ (true story, the author picked it from a name) |
| 6443 | TCP | Kubernetes API server | A 6 in front of HTTPS 443, because it is an HTTPS API |
| 9092 | TCP | Apache Kafka broker | 90, skip 91, 92 |
| 27017 | TCP | MongoDB | 27, then 0, then 17: ten smaller |

Notes on the table:

- **DNS:** normal queries use UDP. TCP is used for large answers and zone transfers. RFC 7766 requires DNS software to support TCP.
- **NetBIOS:** the Network Basic Input/Output System is an old Windows naming and session layer. Modern Windows prefers SMB directly on 445. TCP 137 and UDP 139 exist on the IANA list but are not what you see in practice.
- **Syslog:** TCP 514 is also common in real networks, even though IANA lists TCP 514 for the old `shell` service. For encrypted syslog, see 6514 below.
- **Kubernetes API:** 6443 is the default for self-managed clusters. Managed services such as EKS, GKE and AKS usually expose the API on 443. Minikube commonly uses 8443.
- **SQL Server:** named instances use a dynamic port by default. That is why the Browser service on UDP 1434 exists.
- **465 history:** this port was assigned for SMTPS, then removed, then officially returned for mail submission by RFC 8314.

## More Ports by Area

These add depth for specific roles. You do not need all of them in a first interview.

### Web, Proxies and Developer Defaults

| Port | Transport | Service | Note |
|---|---|---|---|
| 1080 | TCP | SOCKS proxy | Sounds like 1080p video through a proxy |
| 3128 | TCP | Squid web proxy | Vendor default |
| 8080 | TCP | Alternative HTTP | Common for app servers and proxies. Think "80 twice" |
| 8443 | TCP | Alternative HTTPS | Common for admin panels. Think "8 + 443" |
| 3000 | TCP | Grafana, and many Node.js dev servers | Convention, not a standard |
| 5000 | TCP | Flask dev server, Docker Registry | macOS AirPlay Receiver can also take this port |
| 5173 | TCP | Vite dev server | Vendor default |
| 8000 | TCP | Django dev server, Python `http.server` | Vendor default |

### File Transfer and Storage

See [File Transfer Protocols](../07-application-protocols/08-file-transfer-protocols.md) and [File Sharing and Storage Protocols](../07-application-protocols/13-file-sharing-and-storage-protocols.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 111 | Both | rpcbind: maps RPC services to ports | Needed by NFS v3. NFS v4 needs only 2049 |
| 873 | TCP | rsync daemon | rsync over SSH uses 22 instead |
| 3260 | TCP | iSCSI: disk blocks over IP | Internet Small Computer Systems Interface |
| 9418 | TCP | Git protocol | No login and no encryption. Most Git traffic uses 22 or 443 |

### Remote Access

See [Remote Access Protocols](../07-application-protocols/09-remote-access-protocols.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 5900 | TCP | VNC, Virtual Network Computing | Display 1 is often 5901, display 2 is 5902 |
| 5985 | TCP | WinRM over HTTP | Windows Remote Management |
| 5986 | TCP | WinRM over HTTPS | Secure is plus one |

### DNS and Name Discovery

See [DNS Basics](../07-application-protocols/01-dns-basics.md) and [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 443 | TCP | DoH, DNS over HTTPS | Hard to block because it looks like web traffic |
| 546 | UDP | DHCPv6 client | The client is lower here, which is the opposite of IPv4 |
| 547 | UDP | DHCPv6 server | |
| 1900 | UDP | SSDP: Simple Service Discovery Protocol, used by UPnP | Used in reflection attacks when exposed |
| 5353 | UDP | mDNS, Multicast DNS | Local name lookups: DNS 53 with 53 in front |
| 5355 | Both | LLMNR, Link-Local Multicast Name Resolution | Windows fallback, often disabled for security |

### Network Management and Time

See [Network Management Protocols](../07-application-protocols/11-network-management-protocols.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 319, 320 | UDP | PTP, Precision Time Protocol | Finer time than NTP. 319 event, 320 general |
| 830 | TCP | NETCONF over SSH | Device configuration protocol |
| 2055 | UDP | NetFlow | Common choice, not a fixed standard. 9995 and 9996 are also seen |
| 4739 | UDP or TCP | IPFIX: IP Flow Information Export | The standard version of NetFlow |
| 6343 | UDP | sFlow | Sampled traffic data |
| 6514 | TCP | Syslog over TLS | Encrypted, and reliable because it is TCP |

### Routing and Redundancy

Many routing protocols do not use a port at all. See the IP protocol table below for the ones that sit directly on IP.

| Port | Transport | Service | Note |
|---|---|---|---|
| 520 | UDP | RIP, Routing Information Protocol | Also listed in [RIP](../05-routing/05-rip.md) |
| 521 | UDP | RIPng | RIP for IPv6 |
| 646 | Both | LDP, Label Distribution Protocol | Used in [MPLS](../05-routing/13-mpls.md) |
| 1985 | UDP | HSRP, Hot Standby Router Protocol | Cisco |
| 3222 | UDP | GLBP, Gateway Load Balancing Protocol | Cisco |
| 3784 | UDP | BFD, Bidirectional Forwarding Detection (single hop) | Fast link failure detection |

### Directory and Authentication

See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 1645, 1646 | UDP | Old RADIUS ports | Some older gear still uses them |
| 2083 | TCP | RadSec: RADIUS over TLS | Fixes RADIUS weak protection |
| 3268 | TCP | Active Directory Global Catalog | Search across a whole AD forest |
| 3269 | TCP | Global Catalog over TLS | Secure is plus one |
| 3868 | TCP | Diameter | Successor of RADIUS, used in mobile networks |
| 9389 | TCP | Active Directory Web Services | Used by PowerShell AD tools |

### VPN and Tunnels

See [VPNs](../09-security/07-vpn.md), [IPsec](../09-security/08-ipsec.md) and [VXLAN and EVPN](../10-enterprise-and-datacenter/11-vxlan-and-evpn.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 443 | TCP | SSTP, Secure Socket Tunneling Protocol | Microsoft VPN that hides in HTTPS |
| 4789 | UDP | VXLAN, Virtual Extensible LAN | RFC 7348 |
| 6081 | UDP | GENEVE, Generic Network Virtualization Encapsulation | RFC 8926 |
| 8472 | UDP | VXLAN on Linux, old default | Linux kernel default and Flannel default, from before 4789 was assigned |
| 51820 | UDP | WireGuard | A common choice, not an IANA assignment |

### Databases and Caches

| Port | Transport | Service | Note |
|---|---|---|---|
| 5984 | TCP | CouchDB | HTTP API |
| 7000 | TCP | Cassandra node-to-node | Vendor default |
| 7687 | TCP | Neo4j Bolt | Vendor default |
| 8086 | TCP | InfluxDB | Vendor default |
| 8123 | TCP | ClickHouse HTTP | Native protocol is on 9000 |
| 9042 | TCP | Cassandra client (CQL) | Vendor default |
| 9200 | TCP | Elasticsearch REST API | Clients talk here |
| 9300 | TCP | Elasticsearch node-to-node | Nodes talk here |
| 11211 | TCP, UDP on old defaults | Memcached | UDP was used in huge amplification attacks in 2018 |
| 16379 | TCP | Redis Cluster bus | Always client port plus 10000 |
| 27018, 27019 | TCP | MongoDB shard server and config server | One and two above 27017 |

### Messaging and Streaming

| Port | Transport | Service | Note |
|---|---|---|---|
| 1883 | TCP | MQTT, used in IoT messaging | Franz Kafka was born in 1883, a funny link to the Kafka port |
| 8883 | TCP | MQTT over TLS | |
| 2181 | TCP | Apache ZooKeeper | Kafka 4.0 removed ZooKeeper, older Kafka needs it |
| 4222 | TCP | NATS | Vendor default |
| 5671 | TCP | AMQP over TLS | Secure is minus one here, unusual |
| 9093 | TCP | Common choice for Kafka TLS or KRaft controller | Set by config, not a fixed default. Also Prometheus Alertmanager |
| 15672 | TCP | RabbitMQ management web page | 5672 with a 1 in front |

### Containers, Orchestration and Observability

See [Docker Networking](../11-cloud-and-containers/07-docker-networking.md) and [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 2375 | TCP | Docker daemon with no TLS | Dangerous if exposed |
| 2376 | TCP | Docker daemon with TLS | Secure is plus one |
| 2377 | TCP | Docker Swarm cluster management | Swarm also uses 7946 (TCP and UDP) and 4789 UDP |
| 2379 | TCP | etcd client API | The Kubernetes API server talks here |
| 2380 | TCP | etcd peer traffic | etcd nodes talk here |
| 4317 | TCP | OTLP over gRPC | OpenTelemetry data |
| 4318 | TCP | OTLP over HTTP | Plus one for HTTP |
| 5601 | TCP | Kibana | Vendor default |
| 8200 | TCP | HashiCorp Vault | Vendor default |
| 8500 | TCP | HashiCorp Consul HTTP API | Vendor default |
| 9090 | TCP | Prometheus | Vendor default |
| 9100 | TCP | Prometheus node exporter | Same number as raw printing |
| 10250 | TCP | Kubelet API | Runs on every Kubernetes node |
| 10256 | TCP | kube-proxy health check | |
| 10257 | TCP | kube-controller-manager | HTTPS, local to the control plane |
| 10259 | TCP | kube-scheduler | HTTPS, local to the control plane |
| 30000-32767 | TCP and UDP | Kubernetes NodePort default range | Changeable with an API server flag |

### Voice, Video and Chat

See [Real-Time and Modern Protocols](../07-application-protocols/14-realtime-and-modern-protocols.md).

| Port | Transport | Service | Note |
|---|---|---|---|
| 554 | TCP, UDP possible | RTSP, Real Time Streaming Protocol | IP cameras |
| 1720 | TCP | H.323 call setup | Older video conferencing |
| 1935 | TCP | RTMP, Real-Time Messaging Protocol | Live stream upload |
| 2000 | TCP | Cisco SCCP (Skinny) | Cisco IP phones |
| 3478 | UDP or TCP | STUN and TURN | Helps WebRTC calls cross NAT |
| 5349 | TCP | STUN and TURN over TLS | |
| 5222 | TCP | XMPP client | Extensible Messaging and Presence Protocol |
| 5269 | TCP | XMPP server to server | |
| 6667 | TCP | IRC, Internet Relay Chat | 6697 is the TLS version |
| None fixed | UDP | RTP, Real-time Transport Protocol: the actual voice | Port is agreed during call setup. Cisco phones commonly use 16384-32767 |

### Printing, Wireless, IoT and Classics

| Port | Transport | Service | Note |
|---|---|---|---|
| 7 | Both | Echo | Old test service, normally off. Wake-on-LAN packets often use UDP 7 or 9 |
| 9 | Both | Discard | Old test service, normally off |
| 19 | Both | Chargen | Old test service, abused in amplification attacks |
| 37 | Both | Time | Replaced by NTP |
| 43 | TCP | WHOIS | Domain and IP ownership lookup |
| 79 | TCP | Finger | Old user lookup, obsolete |
| 113 | TCP | Ident | Old user identification, sometimes used by IRC |
| 119 | TCP | NNTP, Network News Transfer Protocol | Usenet |
| 502 | TCP | Modbus | Industrial control, no security in the basic protocol |
| 515 | TCP | LPD, Line Printer Daemon | Old Unix printing |
| 631 | TCP | IPP, Internet Printing Protocol | Used by CUPS |
| 4011 | UDP | PXE boot service | Preboot Execution Environment |
| 5246 | UDP | CAPWAP control | Wi-Fi access point to controller |
| 5247 | UDP | CAPWAP data | Wi-Fi access point to controller |
| 5683 | UDP | CoAP, Constrained Application Protocol | IoT devices |
| 9100 | TCP | Raw printing (JetDirect) | Same number as node exporter |

## Not Ports: IP Protocol Numbers

Some protocols do not use TCP or UDP at all. They sit directly on top of IP. They have an 8-bit **IP protocol number** in the IP header, not a port. This is the biggest trap in port questions.

```text
+------------------+-----------------+-----------------+
| IP header        | TCP/UDP header  | Data            |
| Protocol = 6     | Dst port = 443  | TLS bytes       |
+------------------+-----------------+-----------------+
  says "TCP is       says "give this
  inside"            to HTTPS"
```

| IP protocol number | Protocol | Common confusion |
|---|---|---|
| 1 | ICMP | Ping has no port |
| 2 | IGMP | Multicast group joins |
| 4 | IP in IP | Tunnels, including Calico IPIP mode |
| 6 | TCP | |
| 17 | UDP | |
| 41 | IPv6 in IPv4 | 6in4 tunnels |
| 47 | GRE | Generic Routing Encapsulation. PPTP needs it |
| 50 | ESP | Encapsulating Security Payload, the IPsec data |
| 51 | AH | Authentication Header, part of IPsec |
| 58 | ICMPv6 | |
| 88 | EIGRP | Not the same as Kerberos TCP 88 |
| 89 | OSPF | OSPF has no port |
| 103 | PIM | Protocol Independent Multicast |
| 112 | VRRP | Virtual Router Redundancy Protocol |
| 132 | SCTP | A third transport, with its own ports |

Other things with no port:

- **ARP** has no port and no IP header. It sits directly on Ethernet. See [ARP](../03-ethernet-and-switching/05-arp.md).
- **IS-IS** runs directly on Layer 2, not on IP. See [IS-IS](../05-routing/08-is-is.md).
- **Ping** uses ICMP. See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).
- **Traceroute** differs by system. Linux `traceroute` sends UDP to ports starting at 33434 by default. Windows `tracert` sends ICMP.

Routing protocol quick list, a favourite interview question:

| Protocol | Runs on |
|---|---|
| RIP | UDP 520 |
| OSPF | IP protocol 89 |
| EIGRP | IP protocol 88 |
| IS-IS | Directly on Layer 2 |
| BGP | TCP 179 |

## Plain and Secure Pairs

Two ways exist to add TLS to an old protocol:

- **Implicit TLS:** a separate port where TLS starts with the first byte. Example: 993.
- **STARTTLS:** the connection begins as plain text on the normal port, then both sides agree to upgrade. Example: 587.

The full story is in [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

| Plain | Secure | How |
|---|---|---|
| HTTP 80 | HTTPS 443 | Implicit TLS |
| Telnet 23 | SSH 22 | A different protocol |
| FTP 21 | SFTP 22, or FTPS 990 | SFTP is a part of SSH and not related to FTP. FTPS can also upgrade on 21 |
| SMTP 25 (client use) | 465 or 587 | 465 is implicit, 587 is STARTTLS |
| POP3 110 | POP3S 995 | Implicit TLS |
| IMAP 143 | IMAPS 993 | Implicit TLS |
| LDAP 389 | LDAPS 636 | Implicit TLS. STARTTLS on 389 also works |
| DNS 53 | DoT 853, DoH 443, DoQ UDP 853 | Encrypted DNS |
| Syslog 514 | Syslog TLS 6514 | RFC 5425 |
| SIP 5060 | SIP TLS 5061 | |
| MQTT 1883 | MQTT TLS 8883 | |
| AMQP 5672 | AMQPS 5671 | |
| WinRM 5985 | WinRM 5986 | |
| Docker 2375 | Docker 2376 | |
| SNMP v1 or v2c 161 | SNMPv3 161 | Same port. Security is inside the protocol |
| MySQL 3306, PostgreSQL 5432 | Same port | The client asks for TLS inside the connection |

## Why This Transport?

| Service | Transport | Reason |
|---|---|---|
| DHCP | UDP | The client has no IP address yet and must broadcast. TCP cannot broadcast |
| DNS | UDP, then TCP if needed | One small question and one small answer. TCP setup would be wasted time |
| NTP | UDP | Time data is small, and retransmission delays would spoil the time measurement |
| TFTP | UDP | Simple enough to fit in a device's boot code |
| SNMP | UDP | Light for polling thousands of devices, and it can still work when the network is struggling |
| Syslog | UDP (classic) | Fire and forget, so logging never slows the sender. Messages can be lost |
| RADIUS | UDP | Fast short exchanges. Retries are handled by the client |
| TACACS+ | TCP | Reliable delivery. It also encrypts the whole body, while RADIUS hides only the password |
| BGP | TCP | Routes must arrive complete and in order. TCP handles this so BGP does not need to |
| HTTP/3 | UDP | QUIC, a transport built on UDP, handles reliability itself. See [QUIC](08-quic.md) |

## Ports That Travel in Sets

Opening only one port is a classic reason why a service "half works".

| Service | All you need | What breaks if one is missing |
|---|---|---|
| Active FTP | TCP 21, then server connects back from TCP 20 | Login works, file listing hangs |
| Passive FTP | TCP 21, then a high port chosen by the server | Same symptom. Firewalls need a configured range or an FTP helper |
| DHCP | UDP 67 and 68 | Client falls back to a self-assigned 169.254.x.x address |
| DNS | UDP 53 and TCP 53 | Small lookups work, large answers fail |
| SNMP | UDP 161 inbound to device, UDP 162 inbound to manager | Polling works, traps never arrive, or the other way round |
| RADIUS | UDP 1812 and 1813 | Login works, accounting logs stay empty |
| PPTP | TCP 1723 and IP protocol 47 (GRE) | Connects, then no data flows |
| IPsec | UDP 500, UDP 4500, IP protocol 50 (ESP) | Tunnel builds, data stalls. Behind NAT, ESP goes inside UDP 4500 |
| L2TP over IPsec | The IPsec set, plus UDP 1701 inside the tunnel | L2TP has no encryption itself |
| VoIP | SIP 5060 or 5061, plus RTP ports | Phone rings, but no audio |
| HTTPS today | TCP 443 and UDP 443 | Works anyway. Browsers fall back from HTTP/3 to TCP, sometimes slower |

### Port Sets by Job

Mail server:

| Direction | Ports |
|---|---|
| From other mail servers | TCP 25 |
| From your users | TCP 465, 587, 993 (995 if you offer POP3) |

Many ISPs and cloud providers block outbound TCP 25 by default to stop spam.

Active Directory domain controller:

| Function | Ports |
|---|---|
| DNS | 53 TCP and UDP |
| Kerberos | 88 TCP and UDP |
| Time | 123 UDP |
| RPC endpoint mapper | 135 TCP |
| LDAP and LDAPS | 389, 636 |
| SMB | 445 TCP |
| Password change | 464 TCP and UDP |
| Global Catalog | 3268, 3269 TCP |
| RPC dynamic ports | 49152-65535 TCP |

Kubernetes (self-managed, kubeadm defaults):

| Node type | Ports |
|---|---|
| Control plane | 6443, 2379-2380, 10250, 10257, 10259 |
| Worker | 10250, 10256, 30000-32767 |

Check your installation guide, because your network plugin adds its own ports. For example, Calico with BGP uses TCP 179, and VXLAN plugins often use UDP 4789 or 8472.

## Ports to Keep Off the Internet

An open port is a door. Some doors must never face the street.

| Port | Service | Why it is risky |
|---|---|---|
| 23 | Telnet | Passwords travel in clear text |
| 21 | FTP | Passwords travel in clear text |
| 139, 445 | NetBIOS and SMB | WannaCry spread through SMBv1 on TCP 445 in 2017 |
| 3389 | RDP | Constant password guessing, plus serious past bugs such as BlueKeep |
| 1433, 3306, 5432, 27017 | Databases | Only your apps should reach them |
| 6379, 11211 | Redis, Memcached | Weak or no login in careless setups |
| 9200 | Elasticsearch | Large data leaks have come from open clusters |
| 2375 | Docker daemon | Anyone who reaches it can control the host |
| 2379, 10250 | etcd, kubelet | Cluster secrets and control |
| 161 | SNMP v1 and v2c | Community strings travel in clear text |

For more, see [Firewalls](../09-security/04-firewalls.md) and [Common Network Attacks](../09-security/03-common-network-attacks.md).

## Example

### Example 1: Ports in One Page Load

A laptop joins a network and opens `https://www.example.com`. The laptop runs Windows, so its ephemeral ports sit in 49152-65535.

```text
Laptop                     Router                  Web server
192.168.1.20               192.168.1.1             203.0.113.10
    |                          |                        |
    |-- UDP 68 -> 67 (DHCP) -->|                        |
    |<- UDP 67 -> 68 (DHCP) ---|                        |
    |                          |                        |
    |-- UDP 50123 -> 53 (DNS)->|                        |
    |<- UDP 53 -> 50123 -------|                        |
    |                          |                        |
    |-- TCP 50124 -> 443 (HTTPS) ---------------------->|
    |<- TCP 443 -> 50124 -------------------------------|
```

Steps:

1. DHCP gives the laptop 192.168.1.20. Before this step the client has no address, so it broadcasts to UDP 67.
2. DNS turns `www.example.com` into 203.0.113.10, using UDP 53.
3. The browser opens a TCP connection to 203.0.113.10 on port 443.
4. The reply comes back with source port 443 and destination port 50124. The destination is the client's ephemeral port, not 443.

Fuller truth: the router's [NAT](../04-ip-addressing/09-nat-and-pat.md) rewrites the source address and may rewrite the source port on the way out. The browser may also try HTTP/3 on UDP 443.

### Example 2: Read a Server's Open Ports

Linux:

```bash
sudo ss -tuln
```

```text
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port
udp   UNCONN 0      0            0.0.0.0:53         0.0.0.0:*
udp   UNCONN 0      0            0.0.0.0:123        0.0.0.0:*
tcp   LISTEN 0      4096         0.0.0.0:53         0.0.0.0:*
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*
tcp   LISTEN 0      511          0.0.0.0:443        0.0.0.0:*
tcp   LISTEN 0      200        127.0.0.1:5432       0.0.0.0:*
tcp   LISTEN 0      511        127.0.0.1:6379       0.0.0.0:*
```

Reading it:

- UDP and TCP 53: a DNS server.
- UDP 123: an NTP service.
- TCP 22: SSH. TCP 443: a web server.
- TCP 5432 and 6379: PostgreSQL and Redis. They listen on `127.0.0.1`, so only programs on this same machine can reach them. This is good.
- `0.0.0.0` means "all IPv4 addresses on this machine".

Windows (PowerShell):

```powershell
Get-NetTCPConnection -State Listen | Sort-Object LocalPort |
  Select-Object LocalAddress, LocalPort, OwningProcess
```

Find what is using port 8080:

```bash
# Linux
sudo ss -ltnp 'sport = :8080'
```

```powershell
# Windows
netstat -ano | findstr :8080
tasklist /FI "PID eq 4312"
```

Replace 4312 with the process ID in the last column of the `netstat` output.

### Example 3: Test a Remote Port

Linux, using OpenBSD netcat:

```bash
nc -vz 10.20.0.15 5432
```

```text
Connection to 10.20.0.15 5432 port [tcp/postgresql] succeeded!
```

Windows PowerShell:

```powershell
Test-NetConnection db01.example.com -Port 5432
```

```text
ComputerName     : db01.example.com
RemoteAddress    : 10.20.0.15
RemotePort       : 5432
InterfaceAlias   : Ethernet
SourceAddress    : 10.20.0.50
TcpTestSucceeded : True
```

Scan a host you own or have permission to test:

```bash
nmap -p 22,80,443,3306 192.168.1.10
```

```text
Nmap scan report for 192.168.1.10
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
443/tcp  open     https
3306/tcp filtered mysql
```

`filtered` means a firewall likely dropped the probe. The `SERVICE` column here is only a guess from the port number. Add `-sV` to probe what is really running. More tools are in [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

### Example 4: Look Up a Port on Your Own Machine

Both systems ship with a services file. It is a simple list of names and numbers.

```bash
# Linux
grep -w 5432 /etc/services
```

```text
postgresql      5432/tcp        postgres        # PostgreSQL Database
```

```powershell
# Windows
Select-String -Path C:\Windows\System32\drivers\etc\services -Pattern "445/"
```

### Example 5: Firewall Rules for a Three-Tier App

A company has a load balancer subnet, an app subnet, a database subnet and an admin subnet.

```text
Internet
   |  TCP 443, UDP 443
   v
Load balancers   10.0.1.0/24
   |  TCP 8080
   v
App servers      10.0.2.0/24
   |  TCP 5432
   v
PostgreSQL       10.0.3.0/24

Admin hosts      10.0.9.0/24 --- TCP 22 ---> all three subnets
```

| Source | Destination | Port | Why |
|---|---|---|---|
| Any | 10.0.1.0/24 | TCP 443, UDP 443 | Public HTTPS and HTTP/3 |
| Any | 10.0.1.0/24 | TCP 80 | Only to redirect to HTTPS |
| 10.0.1.0/24 | 10.0.2.0/24 | TCP 8080 | Load balancer to app |
| 10.0.2.0/24 | 10.0.3.0/24 | TCP 5432 | App to database |
| 10.0.9.0/24 | 10.0.0.0/16 | TCP 22 | Admin access |

Nothing from the internet reaches 5432 or 22. This is the pattern interviewers want to see.

## Top 20 to Memorise

If you learn only one thing from this page, learn this table. Cover the right side and test yourself.

| # | Port | Transport | Service | Say it aloud |
|---|---|---|---|---|
| 1 | 21 | TCP | FTP control | File transfer commands, data on 20 |
| 2 | 22 | TCP | SSH | Secure login, also SFTP and SCP |
| 3 | 23 | TCP | Telnet | Plain text login, do not use |
| 4 | 25 | TCP | SMTP | Mail between servers |
| 5 | 53 | UDP and TCP | DNS | Names to IPs, TCP for big answers |
| 6 | 67, 68 | UDP | DHCP | Server 67, client 68 |
| 7 | 80 | TCP | HTTP | Plain web |
| 8 | 88 | TCP and UDP | Kerberos | Ticket login for Active Directory |
| 9 | 110 | TCP | POP3 | Download mail |
| 10 | 123 | UDP | NTP | Time |
| 11 | 143 | TCP | IMAP | Mail stays on the server |
| 12 | 161, 162 | UDP | SNMP | Poll on 161, traps on 162 |
| 13 | 389 | TCP | LDAP | Directory lookups |
| 14 | 443 | TCP, UDP for HTTP/3 | HTTPS | Secure web |
| 15 | 445 | TCP | SMB | Windows file sharing |
| 16 | 636 | TCP | LDAPS | Secure directory |
| 17 | 3306 | TCP | MySQL | Database |
| 18 | 3389 | TCP and UDP | RDP | Windows remote desktop |
| 19 | 5432 | TCP | PostgreSQL | Database, countdown |
| 20 | 6379 | TCP | Redis | Cache, keypad MERZ |

### Next 10 for Extra Marks

| Port | Service |
|---|---|
| 69 UDP | TFTP |
| 179 TCP | BGP |
| 514 UDP | Syslog |
| 587 TCP | Mail submission |
| 993 TCP | IMAPS |
| 995 TCP | POP3S |
| 1433 TCP | Microsoft SQL Server |
| 1812 UDP | RADIUS |
| 6443 TCP | Kubernetes API |
| 27017 TCP | MongoDB |

Security engineers should add 500, 4500, 1194 and 51820. DevOps engineers should add 2379, 9092 and 10250.

### A Five-Minute Daily Drill

1. Say the chain: FTP 21, SSH 22, Telnet 23, SMTP 25.
2. Say the DHCP chain: 67, 68, TFTP 69.
3. Say the mail set: 25, 110, 143, then secure 465, 587, 993, 995.
4. Say the directory pair: 389, 636. Then say RDP 3389.
5. Say the databases: 1433, 3306, 5432, 6379, 27017.

## Common Mistakes

- **"Ping uses port 7."** Ping uses ICMP, which has no ports. Port 7 is the old Echo service, which is a different thing.
- **"OSPF runs on port 89."** 89 is an IP protocol number, not a port. EIGRP 88 and Kerberos 88 share only the number.
- **"DNS is only UDP."** DNS uses TCP for zone transfers and large answers. Blocking TCP 53 causes strange failures.
- **"HTTPS is always TCP."** HTTP/3 runs over QUIC on UDP 443.
- **"A port is secure because the number is secure."** Port 443 only says what is usual. The protocol on it decides the security. Encrypted traffic can also carry attacks.
- **"Changing SSH to 2222 protects it."** It reduces noise from bots, but a scan finds it in seconds. Use keys and firewall rules.
- **"The client and server both use port 443."** Only the server side does. The client side uses an ephemeral port.
- **"Port open means the service is healthy."** A TCP connect proves something is listening. It says nothing about whether the app works.
- **"A service bound to 127.0.0.1 is reachable from the network."** It is not. Only local programs can reach it.
- **"SFTP and FTPS are the same."** SFTP runs inside SSH on 22. FTPS is FTP with TLS.
- **"SMTP 25 is what my mail app should use."** Mail clients use 465 or 587. Port 25 is for server-to-server mail.
- **"Well-known ports are the only safe ones."** Many important services use high ports. And a high port alone does not mean an attack.
- **"Only one program can use a port."** The real rule is about the full address. Two programs can share a number if they use different transports or different local IPs. On Linux, `SO_REUSEPORT` also lets related sockets share one.

## Interview Questions

**Q: What are the three port ranges?**
A: Well-known is 0 to 1023, registered is 1024 to 49151, and dynamic or ephemeral is 49152 to 65535. Linux by default uses 32768 to 60999 for ephemeral ports.

**Q: Why is the highest port 65535?**
A: The port field in TCP and UDP headers is 16 bits. That gives 65,536 values, from 0 to 65535.

**Q: Which port does DNS use, and does it use TCP?**
A: DNS uses port 53 on both UDP and TCP. Normal queries go over UDP. TCP is used for zone transfers and answers too big for UDP.

**Q: What is the difference between ports 25, 465 and 587?**
A: Port 25 is for mail servers talking to each other. Ports 465 and 587 are for mail clients sending mail. 465 uses TLS from the first byte, and 587 usually starts plain and upgrades with STARTTLS. Both normally need a login.

**Q: Why does DHCP use two ports?**
A: The server listens on UDP 67, and the client listens on UDP 68. Replies can be broadcast, so a separate client port means other servers on the LAN do not have to process them.

**Q: Why does FTP use two ports?**
A: Port 21 carries commands and stays open for the whole session. Data goes on a separate connection, which is port 20 in active mode or a high port in passive mode. Separating them lets you send commands during a transfer.

**Q: What port does ping use?**
A: None. Ping uses ICMP echo messages. ICMP sits directly on IP as protocol number 1, and it has no ports.

**Q: Which ports or protocol numbers do RIP, OSPF, EIGRP and BGP use?**
A: RIP uses UDP 520. OSPF is IP protocol 89. EIGRP is IP protocol 88. BGP uses TCP 179.

**Q: What does a site-to-site IPsec VPN need open on the firewall?**
A: UDP 500 for IKE, UDP 4500 for NAT traversal, and IP protocol 50 for ESP. AH is IP protocol 51, but it is rarely used, and it does not work through NAT.

**Q: What is the difference between SNMP ports 161 and 162?**
A: The manager polls the device on UDP 161. The device sends unrequested alerts, called traps, to the manager on UDP 162.

**Q: Can two services use the same port number at the same time?**
A: Yes, if the transport or the local IP differs. For example, TCP 53 and UDP 53 are separate. A service on 10.0.0.1:80 and another on 10.0.0.2:80 can also run together.

**Q: A web app works, but the database on port 5432 refuses outside connections. It shows 127.0.0.1:5432 in `ss`. Why?**
A: The database is bound only to the loopback address, so it accepts only local connections. Change the listen address and open the firewall for the app subnet only, not the whole internet.

**Q: How do you find which program uses a port?**
A: On Linux, I run `sudo ss -ltnp` and look at the process column. On Windows, I run `netstat -ano`, take the PID, and look it up with `tasklist`.

## Self-Test

Name the service, then check the answers below.

1. TCP 445
2. UDP 123
3. TCP 636
4. UDP 69
5. TCP 3389
6. TCP 1433
7. UDP 1812
8. UDP 500
9. TCP 6443
10. TCP 179
11. TCP 993
12. TCP 9092
13. IP protocol 89
14. IP protocol 47
15. UDP 4789

Answers: 1 SMB. 2 NTP. 3 LDAPS. 4 TFTP. 5 RDP. 6 Microsoft SQL Server. 7 RADIUS authentication. 8 IKE. 9 Kubernetes API. 10 BGP. 11 IMAPS. 12 Kafka. 13 OSPF (not a port). 14 GRE (not a port). 15 VXLAN.

Now go the other way: give the port for SSH, HTTPS, PostgreSQL, Redis, MongoDB, syslog, submission and L2TP.

Answers: 22, 443, 5432, 6379, 27017, 514, 587, 1701.

## Quick Recap

- Ports are 16-bit numbers: 0-1023 well-known, 1024-49151 registered, 49152-65535 dynamic.
- Default ports are conventions. Any service can run on any port.
- TCP and UDP have separate port sets, so TCP 53 and UDP 53 are different.
- Learn the top 20 first: 21, 22, 23, 25, 53, 67/68, 80, 88, 110, 123, 143, 161/162, 389, 443, 445, 636, 3306, 3389, 5432, 6379.
- ICMP, ARP, OSPF, EIGRP, GRE and ESP have no ports. Know their protocol numbers or layer.
- Many services need sets of ports, so check all of them when something half works.
- Keep databases, caches, SMB, Telnet and admin APIs off the public internet.
