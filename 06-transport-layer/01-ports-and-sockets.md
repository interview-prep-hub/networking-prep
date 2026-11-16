# Ports and Sockets

**Level:** Beginner | **Prerequisites:** [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md), [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md)

## In One Line

An IP address gets data to the right computer. A port number gets it to the right program on that computer, and a socket is the endpoint that a program uses to send and receive.

## Why It Matters

Your laptop usually has one IP address, but many programs use the network at once: a browser, a music app, a chat app, a system update. When a packet (a small piece of data sent over a network) arrives, the laptop must decide which program gets it. The IP address cannot answer that, because all those programs share it.

Ports solve this. They let one IP address serve many programs and many conversations at the same time.

Interviewers ask about this because so much depends on it:

- Firewalls allow or block traffic by port.
- NAT (sharing one public IP among many devices) rewrites ports.
- Load balancers often split traffic using port numbers.
- Everyday errors such as "connection refused" and "address already in use" are about ports and sockets.

A classic test question is: "How does a server handle thousands of clients on port 443?" Many candidates get this wrong. By the end of this page, you will not.

## Simple Explanation

Think of an apartment building.

- The **IP address** is the street address of the building.
- The **port** is the apartment number.
- The **program** is the person living in that apartment.

The postal service uses the street address to reach the building. Then the apartment number decides who gets the letter. A letter that has only the street address is not enough.

**Where the analogy stops:** apartments exist all the time. A port is only a number. It means nothing until a program opens it. A closed port is not an empty room. It is simply a number that nobody is using right now.

Now think of a call centre to understand how one server talks to many clients.

All customers dial the same phone number. The company does not give each customer a new phone number. Each call is a separate conversation, and the phone system tells calls apart by who is calling. One person answers the main line, then passes each call to an agent.

- The main phone number is the **listening** port.
- Each active call is an **established** connection.
- The caller's number is what makes each call different.

**Where the analogy stops:** in a real call centre, an agent is one person. In a server, one program can handle thousands of connections at once.

## How It Works

### Where ports live in a packet

The transport layer is the network layer that moves data between programs (not just between computers). Its two classic protocols are TCP and UDP. A protocol is a set of rules that both sides follow. See [TCP vs UDP](02-tcp-vs-udp.md) for how they differ.

A port is a 16-bit number carried in the TCP or UDP header. A header is the control information placed in front of the data. With 16 bits, the possible values are 0 to 65535, which is 65,536 values.

```text
What one packet looks like, wrapped layer by layer:

+-----------+------------+-----------------+----------------+
| Ethernet  | IP header  | TCP/UDP header  | Program data   |
| MAC addrs | IP addrs   | PORT numbers    | (for example   |
|           |            |                 |  HTTP)         |
+-----------+------------+-----------------+----------------+

First 32 bits of both the TCP and the UDP header:

 bit 0            bit 15 bit 16           bit 31
+----------------------+----------------------+
|     Source port      |   Destination port   |
|      (16 bits)       |      (16 bits)       |
+----------------------+----------------------+
```

Every TCP or UDP packet carries two ports:

- **Source port:** the port of the sender.
- **Destination port:** the port of the receiver.

A reply swaps them. The old destination becomes the new source.

Ports belong to TCP and UDP, not to IP. Protocols without these headers have no ports. ICMP, which `ping` uses, is one example. See [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md).

TCP and UDP have separate port spaces. TCP port 53 and UDP port 53 are two different things. One program can use both, and DNS does exactly that.

### Port ranges

IANA (Internet Assigned Numbers Authority) is the group that keeps the official list of port numbers. RFC 6335 describes three ranges. An RFC is a published internet standards document.

| Range | Size | IANA name | Common name | Examples |
|---|---|---|---|---|
| 0-1023 | 1,024 | System ports | Well-known | 22 SSH, 53 DNS, 80 HTTP, 443 HTTPS |
| 1024-49151 | 48,128 | User ports | Registered | 3306 MySQL, 3389 RDP, 5432 PostgreSQL |
| 49152-65535 | 16,384 | Dynamic or private | Ephemeral | Temporary client ports |

"Ephemeral" means short-lived. An ephemeral port is a temporary port that the operating system picks for a client.

Three details that interviewers like:

- **Port 0 is reserved.** It is not used on the wire in normal traffic. If a program asks to use port 0, the operating system picks a free port for it.
- **Low ports can be protected.** On Linux, by default, a program needs root or the `CAP_NET_BIND_SERVICE` permission to use a port below 1024. Windows has no such rule by default.
- **The OS range can differ from the IANA range.** Modern Windows uses 49152-65535, which matches IANA. Linux usually uses 32768-60999, which is 28,232 ports. That Linux range overlaps the registered range. Distributions and administrators can change it.

For a full list of ports, see the [Common Ports Reference](10-common-ports-reference.md).

**Fuller truth:** a port number is a convention, not a guarantee. You can run SSH on port 2222 or a web server on port 8080. Traffic on port 443 does not prove it is HTTPS. It only proves someone used that number.

### Who picks which port

The server uses a fixed port, such as 443, chosen by the administrator or the program. It must be known in advance, so that clients know where to go.

The client normally uses an ephemeral port, such as 49822, chosen by the operating system. It can be any free port, because the server learns it from the first packet.

### The 5-tuple

A tuple is an ordered list of values. The **5-tuple** is the set of five values that identifies one flow of traffic:

1. Protocol (TCP or UDP)
2. Source IP address
3. Source port
4. Destination IP address
5. Destination port

Example:

| Connection | Protocol | Source IP | Src port | Destination IP | Dst port |
|---|---|---|---|---|---|
| 1 | TCP | 10.0.0.21 | 49822 | 10.0.0.10 | 443 |
| 2 | TCP | 10.0.0.21 | 49823 | 10.0.0.10 | 443 |
| 3 | TCP | 10.0.0.22 | 49822 | 10.0.0.10 | 443 |

Each row differs from the others in at least one value, so all three can exist together. Rows 1 and 3 even use the same source port. That is fine, because the source IP differs.

Many devices use the 5-tuple:

- A **host** uses it to find the right connection.
- A **stateful firewall** uses it to track connections. See [Firewalls](../09-security/04-firewalls.md).
- A **NAT device** uses it to map replies back. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).
- A **load balancer** or router can hash it, so that all packets of one flow take the same path. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

### Sockets

The word "socket" has two meanings. Be ready to name both.

1. **The address meaning:** a socket is an IP address plus a port, such as `10.0.0.10:443`. TCP standards use the word this way.
2. **The programming meaning:** a socket is an object inside the operating system that a program uses to talk to the network. The program holds a handle to it, and reads and writes through it, much like a file. On Linux, this handle is called a file descriptor.

So a port is a number. A socket is an endpoint built with that number.

A TCP connection is defined by a pair of sockets, one at each end, such as `10.0.0.21:49822` and `10.0.0.10:443`.

Programs usually use a stream socket for TCP and a datagram socket for UDP. A datagram is one independent message.

The diagram below shows the standard system calls. A system call is a request from a program to the operating system.

```text
        SERVER                              CLIENT

   socket()   make a socket
   bind()     claim 0.0.0.0:443
   listen()   start waiting
   accept()   wait for a client         socket()  make a socket
       |                                connect() to 10.0.0.10:443
       |                                    |     (OS picks a port)
       |<------ three-way handshake ------->|
       |                                    |
   accept() returns a NEW socket        connect() returns
   recv() <--------- request ---------- send()
   send() ---------- response --------> recv()
   close()                              close()
```

The three-way handshake is how TCP starts a connection. See [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md).

`bind()` also chooses which local address to accept traffic on:

| Bind address | Who can reach it |
|---|---|
| `127.0.0.1:8080` | Only programs on the same machine |
| `10.0.0.10:8080` | Clients that reach that one address |
| `0.0.0.0:8080` | Clients on any IPv4 address of the machine |
| `[::]:8080` | Clients on any IPv6 address (on Linux, usually IPv4 too) |

### Listening vs established

A server has two kinds of TCP socket.

| | Listening socket | Established socket |
|---|---|---|
| Job | Waits for new connections | Carries data for one client |
| Remote side | Unknown, shown as `*` | Known IP and port |
| Number on a port | Normally one | Many |
| Carries program data | No | Yes |
| Created by | `listen()` | `accept()` on the server, `connect()` on the client |

Think of the listening socket as a receptionist. It takes the new arrival and creates a dedicated socket for that client, then goes back to waiting.

UDP has no connections, so it has no listening or established state. Tools show a bound UDP socket as `UNCONN` on Linux. For all the TCP states, see [TCP Connection States](04-tcp-states.md).

### How one server port serves many clients

1. The server opens a listening socket on port 443.
2. A client sends a connection request from `10.0.0.21:49822` to `10.0.0.10:443`.
3. The handshake finishes. The operating system creates a new socket for this client.
4. `accept()` gives that socket to the program.
5. The new socket **still uses local port 443**. No new port is opened.
6. The listening socket keeps waiting for the next client.

The server tells clients apart by the remote IP and remote port. Local port 443 is the same for all of them.

```text
                     SERVER 10.0.0.10
                  +---------------------------------------+
 10.0.0.21:49822 =|=> socket 10.0.0.10:443 <-> .21:49822  |
 10.0.0.21:49823 =|=> socket 10.0.0.10:443 <-> .21:49823  |
 10.0.0.22:49822 =|=> socket 10.0.0.10:443 <-> .22:49822  |
                  |                                       |
 new client  -----|-> LISTEN 0.0.0.0:443 (waits for more) |
                  +---------------------------------------+
```

The program must also share its time among all those sockets. It can use one process or thread per connection (a thread is one line of work inside a process). It can also use an event loop, where one thread watches many sockets and handles whichever is ready. Linux offers `epoll` for this, and Windows offers I/O completion ports.

### Multiplexing and demultiplexing

**Multiplexing** happens at the sender. Data from many sockets is labelled with ports and combined onto one network path.

**Demultiplexing** happens at the receiver. Port and address fields are used to hand each packet to the right socket.

```text
 CLIENT 10.0.0.21                         SERVER 10.0.0.10

 Browser conn 1 :49822 --+              +--> web socket 1
 Browser conn 2 :49823 --+==> network ==+--> web socket 2
 SSH client     :49900 --+              +--> SSH socket

     multiplexing                          demultiplexing
```

How the receiver looks up the right socket depends on the protocol:

**TCP** uses all of the 5-tuple:

1. Look for an established socket that matches all five values.
2. If there is none and the packet is a new connection request (a SYN), look for a listening socket on that port.
3. If there is none, send a reset (RST). The client reports "connection refused".

**UDP** uses only the destination IP and port, for an ordinary server socket. Packets from all clients land in the same socket, and the program reads the sender's address from each message. If no socket matches, the host normally sends an ICMP "port unreachable" message.

**Fuller truth:** a UDP socket can call `connect()`, and then the operating system matches the remote side too. A protocol built on UDP can also do its own sorting. [QUIC](08-quic.md) uses connection IDs, so a connection can survive a change of client IP address.

The word "multiplexing" also has other uses. HTTP/2 multiplexes many streams inside one connection. That is a different layer, covered in [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

### Port exhaustion: a first look

Port exhaustion means a client has no free port left for a new connection.

It happens to a **client**, or to anything acting as one. Common cases are an app server calling a database, and a proxy calling a backend.

Why it happens: for a fixed destination (one IP and one port), the source IP is fixed too. The source port is the only value left to change. So a default Linux host can have about 28,232 connections at once to that one destination.

Closed connections can also hold a port for a while. The side that closes first keeps the connection in a state called TIME_WAIT. On Linux this lasts 60 seconds.

```text
App server 10.0.0.30                      Database 10.0.0.40:5432

 ports 32768-60999  ==== one new connection ====>  always the
 (28,232 ports)          per query, then close     same 3 values

 28,232 ports / 60 seconds TIME_WAIT = about 470 new connections
 per second, if the client closes first and ports are not reused.
```

Above that rate, new connections start to fail. This is an estimate for a default Linux host, not a hard law.

The usual fixes, best first:

1. Reuse connections with a connection pool or keep-alive.
2. Add more destination IPs or ports.
3. Add more source IPs.
4. Widen the ephemeral port range, while avoiding ports that local services listen on.

NAT devices hit the same problem, because many inside hosts share a few public IPs and their ports. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

## Example

**Setup:** a Linux web server at 10.0.0.10 runs nginx on TCP 443 and SSH on TCP 22. Client A (Windows, 10.0.0.21) opens two connections. Client B (Linux, 10.0.0.22) opens one. All are on one LAN, so there is no NAT.

**Step 1: see what the server is listening on (Linux).** `-t` means TCP, `-l` means listening, and `-n` means show numbers.

```bash
ss -tln
```

```text
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       128     0.0.0.0:22          0.0.0.0:*
LISTEN  0       511     0.0.0.0:443         0.0.0.0:*
```

The `*` shows that no remote side exists yet.

**Step 2: look at client A (Windows).**

```powershell
netstat -ano | findstr ":443"
```

```text
  TCP    10.0.0.21:49822    10.0.0.10:443    ESTABLISHED    6120
  TCP    10.0.0.21:49823    10.0.0.10:443    ESTABLISHED    6120
```

The operating system picked two ephemeral ports. The last column is the process ID, which identifies the program.

**Step 3: look at client B (Linux).**

```bash
ss -tn 'dport = :443'
```

```text
State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
ESTAB  0       0       10.0.0.22:49822     10.0.0.10:443
```

Client B also chose 49822. This causes no problem. Client IPs differ.

**Step 4: look at the server again (Linux).** `-a` shows all sockets, including listening ones.

```bash
ss -tan 'sport = :443'
```

```text
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
LISTEN  0       511     0.0.0.0:443         0.0.0.0:*
ESTAB   0       0       10.0.0.10:443       10.0.0.21:49822
ESTAB   0       0       10.0.0.10:443       10.0.0.21:49823
ESTAB   0       0       10.0.0.10:443       10.0.0.22:49822
```

One port, four sockets. Local port 443 appears in every row.

**Step 5: trace how arriving packets are sorted.**

| Packet source | Packet destination | Type | Result |
|---|---|---|---|
| 10.0.0.21:49822 | 10.0.0.10:443 | Data | Matches row 2, goes to that socket |
| 10.0.0.22:49822 | 10.0.0.10:443 | Data | Matches row 4, same source port as above, different IP |
| 10.0.0.22:50010 | 10.0.0.10:443 | SYN | No established match, goes to the listener, new connection |
| 10.0.0.22:50011 | 10.0.0.10:8080 | SYN | No listener on 8080, server sends RST |

**Useful commands side by side:**

| Task | Linux | Windows (PowerShell) |
|---|---|---|
| Show listeners | `ss -tuln` | `Get-NetTCPConnection -State Listen` |
| Show UDP sockets | `ss -uan` | `Get-NetUDPEndpoint` |
| Find owner of a port | `sudo ss -tlnp 'sport = :443'` | `Get-NetTCPConnection -LocalPort 443 -State Listen` then read `OwningProcess` |
| Show ephemeral range | `sysctl net.ipv4.ip_local_port_range` | `netsh int ipv4 show dynamicport tcp` |
| Count TIME_WAIT | `ss -s` | `(Get-NetTCPConnection -State TimeWait).Count` |
| Test a remote port | `nc -vz 10.0.0.10 443` | `Test-NetConnection 10.0.0.10 -Port 443` |

More tools are in [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

**Second example: spotting exhaustion.** The app server 10.0.0.30 fails with `Cannot assign requested address` when it connects to the database. On Linux, that message is the normal sign of this problem. On Windows, programs often report error 10055 instead.

```bash
ss -s
```

```text
Total: 30412
TCP:   28650 (estab 410, closed 28101, orphaned 0, timewait 28101)
```

About 28,000 sockets are in TIME_WAIT, close to the 28,232 port limit. The app opens a new connection for each query. The right fix is a connection pool.

## Going Deeper

**Backlog queues.** Linux tracks connection setup in two steps. A half-open connection waits for the handshake to finish. A finished connection then waits in the accept queue until the program calls `accept()`. In `ss -ltn`, Recv-Q shows the current accept queue size and Send-Q shows its limit. A slow program can fill this queue while the machine still looks healthy.

**The real limits on connections.** A listening port has no fixed connection limit of 65,535. The 5-tuple allows about 65,000 connections per client IP, per server IP and port. With many client IPs, the total is far higher. Real limits are memory, CPU and open file limits. On Linux, each socket uses a file descriptor, and `ulimit -n` shows the per-process limit.

**Sharing a port.** Normally only one socket can listen on a given IP, port and protocol. A second program gets `Address already in use`. Two exceptions matter:

- `SO_REUSEPORT` (Linux 3.9 and later) lets several sockets listen on one port. The kernel spreads new connections among them. Multi-worker servers can use it.
- `SO_REUSEADDR` lets a restarting server bind its port even if old connections remain in TIME_WAIT. On Windows, this option behaves differently and is less safe, so do not assume it matches Linux.

**Client port reuse.** On Linux, a client can reuse one source port for different destinations, because the 5-tuple is still unique. So the 28,232 limit is per destination, not per machine. Windows has its own rules for reuse, so check its documentation for the version you run.

**Reusing TIME_WAIT ports.** The Linux setting `net.ipv4.tcp_tw_reuse` can let new outgoing connections reuse such ports when it is safe. The old `tcp_tw_recycle` setting caused problems behind NAT and was removed in Linux 4.12. Do not recommend it.

## Common Mistakes

- **"A server opens a new port for each client."** False for TCP. All accepted sockets keep local port 443. The remote IP and port tell them apart.
- **"A server can only have 65,535 connections."** False. That is the size of one port field, not a limit on 5-tuples. Memory and file limits come first.
- **"A port and a socket are the same thing."** A port is a number. A socket is an endpoint that uses that number.
- **"A network port is a physical port."** A switch port is a physical connector for a cable. A TCP or UDP port is a number in a header.
- **"Port 443 always means HTTPS."** It is a convention. Any program can use the port.
- **"TCP 53 and UDP 53 are one port."** They are separate.
- **"Ephemeral ports always start at 49152."** Linux normally uses 32768-60999.
- **"Ping works, so the port is open."** Ping uses ICMP, which has no ports. Test the port itself.
- **"Listening on 127.0.0.1 is reachable from other machines."** It is not. This is a common reason why a service works locally but not remotely. Bind to `0.0.0.0` or a real address when remote access is needed.
- **"Port exhaustion happens on the server."** It is mainly a client-side problem, and it is commonly seen on proxies and NAT devices.

## Interview Questions

**Q: What is a port, and why do we need it if we have an IP address?**

A: A port is a 16-bit number in the TCP or UDP header. The IP address reaches the host, and the port reaches the right program on that host. Without ports, many programs could not share one IP address.

**Q: What are the port ranges?**

A: 0-1023 are well-known, 1024-49151 are registered, and 49152-65535 are dynamic or ephemeral, by IANA. Operating systems may differ: Linux usually uses 32768-60999 for ephemeral ports.

**Q: What is the 5-tuple?**

A: Protocol, source IP, source port, destination IP and destination port. It uniquely identifies a flow. Hosts use it to find the right socket, and firewalls, NAT devices and load balancers use it to track or hash flows.

**Q: What is the difference between a port and a socket?**

A: A port is just a number. A socket is an endpoint: in standards, an IP and port pair, and in programming, an operating system object used to send and receive. A TCP connection is a pair of sockets.

**Q: How does a server handle thousands of clients on port 443?**

A: One listening socket waits on 443. For each client, `accept()` creates a new socket that also uses local port 443. The kernel tells them apart by the client IP and port, so no extra server ports are needed.

**Q: What is the difference between listening and established?**

A: A listening socket waits for new connections and has no remote side. An established socket belongs to one connection, has a known remote IP and port, and carries the data.

**Q: What are multiplexing and demultiplexing?**

A: Multiplexing is the sender combining data from many sockets onto the network, labelled with ports. Demultiplexing is the receiver using those fields to deliver each packet to the right socket. TCP matches the full 5-tuple. An ordinary UDP socket matches only destination IP and port.

**Q: Can TCP and UDP use the same port number at the same time?**

A: Yes. They have separate port spaces. DNS uses port 53 on both.

**Q: Can two programs listen on the same port?**

A: Normally not on the same IP and protocol: the second gets "address already in use". They can if they bind to different IPs, use different protocols, or use an option such as `SO_REUSEPORT`.

**Q: What happens when a client connects to a closed port?**

A: With TCP, the host replies with a reset, and the client sees "connection refused". With UDP, the host usually returns ICMP port unreachable. If a firewall silently drops the packet, the client sees a timeout instead.

**Q: What is the maximum number of connections to one server port?**

A: There is no 65,535 limit, since each connection only needs a unique 5-tuple. One client IP is limited by its available ports, but many client IPs can connect. In practice, memory, CPU and file descriptor limits decide.

**Q: What is port exhaustion, and how do you fix it?**

A: A client runs out of ephemeral ports to one destination, often because many short connections sit in TIME_WAIT. On Linux, you see "cannot assign requested address". The best fix is to reuse connections with pooling or keep-alive. Other fixes are more source IPs, more destinations, or a wider port range.

## Quick Recap

- IP finds the host, and the port finds the program. Ports are 16-bit numbers in TCP and UDP headers, from 0 to 65535.
- IANA ranges: well-known 0-1023, registered 1024-49151, dynamic 49152-65535. Linux usually uses 32768-60999 for ephemeral ports.
- The 5-tuple (protocol, source IP, source port, destination IP, destination port) identifies a flow.
- A socket is an endpoint. A listening socket waits for clients, and each established socket carries one connection.
- A server does not use a new port for each client. All accepted sockets keep port 443 and differ by the client IP and port.
- TCP demultiplexes on the full 5-tuple. An ordinary UDP socket uses only the destination IP and port.
- Port exhaustion is mostly a client-side limit to one destination. Connection reuse is the best fix.
