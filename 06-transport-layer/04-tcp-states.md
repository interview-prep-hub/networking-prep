# TCP Connection States

**Level:** Advanced | **Prerequisites:** [Ports and Sockets](01-ports-and-sockets.md), [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md)

## In One Line

A TCP connection moves through 11 named states from birth to death, and the state tells you who has said what, who is waiting, and which side (the kernel or the application) must act next.

## Why It Matters

TCP is a conversation between two machines. Each machine must remember how far the conversation has gone. The state is that memory.

States turn into real outages:

- A web server stops accepting users and logs `Too many open files`. The cause is thousands of sockets stuck in CLOSE-WAIT because of a bug in the code.
- A busy proxy starts failing with `Cannot assign requested address`. The cause is that it ran out of local ports, and TIME-WAIT sockets are holding them.
- A team copies a "TCP tuning" blog post into production. Users behind one office NAT (a device that lets many machines share one public IP) can no longer connect.

Interviewers ask about states because the answers show if you have run real systems. These three questions come up again and again:

- Why does TIME-WAIT exist?
- Who ends up in TIME-WAIT?
- What do many CLOSE-WAIT sockets mean?

## Simple Explanation

Think of ending a phone call between Ali and Sara.

1. Ali says, "I have nothing more to say." (This is a FIN.)
2. Sara says, "OK, I heard you." (This is an ACK.) But Sara may still have things to say, and Ali keeps listening.
3. Sara finishes and says, "I have nothing more to say either." (Her FIN.)
4. Ali says, "OK, I heard you." (His ACK.)
5. Ali does not hang up at once. He waits a short time. If Sara did not hear his last "OK", she will repeat herself, and Ali can answer again. This wait is TIME-WAIT.

Now think about what goes wrong:

- If Sara hears "I am done", says "OK", and then just sits there forever and never says her own goodbye, the line stays busy. This is a CLOSE-WAIT leak.
- If Ali makes thousands of calls a minute and each finished call keeps a line busy for one minute, he runs out of lines. This is ephemeral port exhaustion.

**Where the analogy stops being true:** a phone call has no old words that arrive late. On a network, an old packet can arrive long after the connection is gone. That is the second reason TIME-WAIT exists, and it has no match in a phone call.

## How It Works

### What a state is

A **state** is one word stored by the operating system for each connection. It says what this side has seen and what it expects next.

A **state machine** is a set of states plus rules for moving between them. An event happens, such as a packet arriving or the application calling a function. The rule says what to send and which state comes next.

The state lives in a kernel record called the **TCB** (Transmission Control Block). The TCB also stores the sequence numbers, the timers and the address pair. The **kernel** is the core of the operating system. It runs TCP for your program.

Each side has its own state. The two sides are often in different states at the same moment. This is normal. For example, one side can be in FIN-WAIT-2 while the other is in CLOSE-WAIT.

Each connection is identified by its **4-tuple**: source IP, source port, destination IP, destination port. Change any one of the four and it is a different connection with a different state.

### Flags that drive the states

| Flag | Short meaning | Effect |
| --- | --- | --- |
| SYN | "Let us start, here is my first sequence number" | Opens a connection |
| ACK | "I received everything up to this point" | Confirms |
| FIN | "I have no more data to send" | Closes one direction, gently |
| RST | "Stop now, this connection is dead" | Kills the connection at once |

A **sequence number** counts bytes, so each side can spot lost or repeated data. SYN and FIN each use up one sequence number, so they can be acknowledged like data. See [TCP Header and the Three-Way Handshake](03-tcp-handshake-and-header.md) for the header fields.

### The full state diagram

The diagram has two parts because it does not fit on one screen. The two arrows at the bottom of the first part lead to the same-named boxes in the second part. "recv" means received.

**Part 1: opening a connection**

```text
                          +--------+
           +--------------| CLOSED |--------------+
           | passive open +--------+  active open |
           | send nothing             send SYN    |
           v                                      v
      +--------+        send SYN (rare)     +----------+
      | LISTEN |--------------------------->| SYN-SENT |
      +--------+                            +----------+
           |                                  |      |
           | recv SYN         recv SYN        |      | recv SYN+ACK
           | send SYN+ACK     send SYN+ACK    |      | send ACK
           |                  (simultaneous   |      |
           v                   open)          |      |
      +----------+ <--------------------------+      |
      | SYN-RCVD |                                   |
      +----------+                                   |
           | recv ACK of our SYN                     |
           v                                         |
      +-------------+ <------------------------------+
      | ESTABLISHED |
      +-------------+
         |       |
         |       | recv FIN, send ACK
         |       +-------> to CLOSE-WAIT (passive close, Part 2)
         |
         | close, send FIN
         +---------------> to FIN-WAIT-1 (active close, Part 2)
```

**Part 2: closing a connection**

```text
     ACTIVE CLOSE (we close first)               PASSIVE CLOSE

  +------------+ recv FIN  +---------+        +------------+
  | FIN-WAIT-1 |---------->| CLOSING |        | CLOSE-WAIT |
  +------------+ send ACK  +---------+        +------------+
    |      |                    |                   |
    |      | recv FIN and       | recv ACK          | close
    |      | ACK of our FIN     | of our FIN        | send FIN
    |      | send ACK           |                   v
    |      |                    |             +----------+
    |      +--------+           |             | LAST-ACK |
    |               |           |             +----------+
    | recv ACK      |           |                   |
    | of our FIN    |           |                   | recv ACK
    v               v           |                   | of our FIN
  +------------+  +-----------+ |                   v
  | FIN-WAIT-2 |->| TIME-WAIT |<+              +--------+
  +------------+  +-----------+                | CLOSED |
   recv FIN,            |                      +--------+
   send ACK             | wait 2 x MSL
                        v
                   +--------+
                   | CLOSED |
                   +--------+
```

Some arrows are left out to keep the drawing clean. This table lists them.

| From | Event | Action | To |
| --- | --- | --- | --- |
| LISTEN | Application closes | Nothing | CLOSED |
| SYN-SENT | Application closes, or timeout | Nothing | CLOSED |
| SYN-RCVD | Application closes | Send FIN | FIN-WAIT-1 |
| SYN-RCVD | Receive RST, and the socket came from LISTEN | Nothing | LISTEN |
| SYN-RCVD | Receive RST, and the socket came from SYN-SENT | Nothing | CLOSED |
| Any synchronized state | Receive a valid RST | Nothing | CLOSED |
| Any state except LISTEN and CLOSED | Application aborts | Send RST in most states | CLOSED |

**Synchronized** means both sides have agreed on sequence numbers. That is ESTABLISHED and every state after it.

### Every state explained

There are 11 states in the standard, RFC 9293. Tools use slightly different spellings. This table gives them all.

| RFC 9293 name | `ss` (Linux) | `netstat` (Linux) | `netstat` (Windows) |
| --- | --- | --- | --- |
| LISTEN | LISTEN | LISTEN | LISTENING |
| SYN-SENT | SYN-SENT | SYN_SENT | SYN_SENT |
| SYN-RECEIVED | SYN-RECV | SYN_RECV | SYN_RECEIVED |
| ESTABLISHED | ESTAB | ESTABLISHED | ESTABLISHED |
| FIN-WAIT-1 | FIN-WAIT-1 | FIN_WAIT1 | FIN_WAIT_1 |
| FIN-WAIT-2 | FIN-WAIT-2 | FIN_WAIT2 | FIN_WAIT_2 |
| CLOSE-WAIT | CLOSE-WAIT | CLOSE_WAIT | CLOSE_WAIT |
| CLOSING | CLOSING | CLOSING | CLOSING |
| LAST-ACK | LAST-ACK | LAST_ACK | LAST_ACK |
| TIME-WAIT | TIME-WAIT | TIME_WAIT | TIME_WAIT |
| CLOSED | UNCONN (when shown) | not shown | not shown |

**CLOSED.** There is no connection. RFC 9293 calls it fictional, because there is no TCB to hold the word "CLOSED". You rarely see it in tools.

**LISTEN.** A server socket is waiting for connections on a port. It has no peer. It accepts SYN packets. Example: a web server on port 443. Expect a line such as `0.0.0.0:443` with a peer of `0.0.0.0:*`.

**SYN-SENT.** A client sent a SYN and waits for SYN+ACK. This state should last about one round trip. On Linux the kernel retries the SYN `tcp_syn_retries` times (default 6), with the wait doubling each time. That gives a total of about 127 seconds before `Connection timed out`. Many stuck sockets here means a firewall is dropping packets or the server is down.

**SYN-RECEIVED.** A server got a SYN, sent SYN+ACK, and waits for the final ACK. It is sometimes called a half-open connection. Linux retries the SYN+ACK `tcp_synack_retries` times (default 5), about 63 seconds in total. Many here can mean a SYN flood, an attack that sends many SYNs and never finishes the handshake. It can also mean that the final ACK cannot get back, for example because of asymmetric routing (packets go out by one path and return by another, and a firewall sees only half). See [Common Network Attacks](../09-security/03-common-network-attacks.md) for SYN floods.

**ESTABLISHED.** The handshake is finished. Data flows both ways. There is no time limit by default. A connection can stay here for days with no packets at all.

**FIN-WAIT-1.** The application closed. The kernel sent a FIN and waits for its ACK. This usually lasts one round trip. It lasts longer if unsent data is still queued, because the FIN goes after the data. Many here can mean a peer that has disappeared. It can also mean a peer with a zero receive window: the peer has told us it has no buffer space, so we must stop sending.

**FIN-WAIT-2.** Our FIN was acknowledged. We wait for the peer's FIN. Our direction is closed. The peer may still send data to us. A socket stuck here means the other side has not closed yet, so look for a CLOSE-WAIT problem on the peer.

**CLOSE-WAIT.** The peer sent a FIN and our kernel acknowledged it. Now the kernel waits for **our own application** to call `close()`. TCP has no standard timer here. This is the state that leaks.

**CLOSING.** Both sides sent FIN at almost the same moment. We received their FIN before they acknowledged ours. We wait for that ACK. It is rare and lasts about one round trip.

**LAST-ACK.** We were the passive closer. We sent our FIN and wait for the final ACK. When it arrives, the connection is gone at once, with no TIME-WAIT on this side.

**TIME-WAIT.** We closed first, and the whole four-step close is done. We wait a fixed time for two safety reasons explained below.

### What moves a socket between states

Some moves come from your program. Others come from the kernel alone. This difference matters for debugging.

| Your program calls | What the kernel does | State change |
| --- | --- | --- |
| `listen()` | Marks the socket as a server socket | CLOSED to LISTEN |
| `connect()` | Sends SYN | CLOSED to SYN-SENT |
| `accept()` | Hands over a connection that is already finished | No change, it stays ESTABLISHED |
| `close()` first | Sends FIN after queued data | ESTABLISHED to FIN-WAIT-1 |
| `close()` after peer's FIN | Sends FIN | CLOSE-WAIT to LAST-ACK |
| `shutdown(SHUT_WR)` | Sends FIN, reading still works | ESTABLISHED to FIN-WAIT-1 |

A key fact: **the kernel finishes the handshake, not `accept()`**. A client can be fully ESTABLISHED before the server program has done anything.

Your program learns about a peer's FIN in one way: `read()` returns 0 bytes. Zero bytes means "end of the stream". If the program never reads, or treats zero as nothing special, it never finds out.

### Normal, rare and odd paths

The normal paths:

```text
Client (active open):
  CLOSED > SYN-SENT > ESTABLISHED

Server (passive open):
  LISTEN > SYN-RCVD > ESTABLISHED

Side that closes first (active close):
  ESTABLISHED > FIN-WAIT-1 > FIN-WAIT-2 > TIME-WAIT > CLOSED

Side that closes second (passive close):
  ESTABLISHED > CLOSE-WAIT > LAST-ACK > CLOSED
```

The less common paths:

- **Simultaneous open.** Both sides send SYN to each other at the same time. Each goes SYN-SENT to SYN-RCVD to ESTABLISHED. The result is one connection, not two. It is rare in normal use.
- **Simultaneous close.** Both sides send FIN at the same time. Each goes FIN-WAIT-1 to CLOSING to TIME-WAIT. Here **both** sides get TIME-WAIT.
- **Three-segment close.** If the peer's application closes straight away, the peer can put its ACK and FIN in one segment. The closer goes from FIN-WAIT-1 directly to TIME-WAIT. You will see this often in packet captures.
- **Abort with RST.** One side sends RST. The other side goes straight to CLOSED, and neither side enters TIME-WAIT.

### FIN versus RST

| | FIN | RST |
| --- | --- | --- |
| Meaning | "I am finished sending" | "Abort this connection" |
| Data in flight | Delivered first | Unsent data is thrown away |
| Needs ACK | Yes | No |
| Other direction | Stays open | Dies too |
| Peer's program sees | `read()` returns 0 | Error `ECONNRESET` |
| TIME-WAIT afterwards | Yes, on the first closer | No |

Common reasons for an RST:

- A SYN arrived for a port with no listener.
- A packet arrived for a connection this machine has forgotten.
- A program closed with unread data in its receive buffer. RFC 9293 allows this, and Linux does it.
- A firewall or load balancer killed the connection.

### TIME-WAIT in depth

**Who gets it.** The side that sends the first FIN gets TIME-WAIT. This is the **active closer**. It is not always the client. If a web server closes first, the TIME-WAIT is on the server.

**How long it lasts.** It lasts 2 x MSL. **MSL** means Maximum Segment Lifetime: the longest time a TCP segment is assumed to survive in the network.

| System | TIME-WAIT length |
| --- | --- |
| RFC 9293 standard | MSL is 2 minutes, so 2 x MSL is 4 minutes |
| Linux | 60 seconds, fixed by the kernel constant `TCP_TIMEWAIT_LEN` |
| FreeBSD | 60 seconds by default (`net.inet.tcp.msl` is 30000 ms) |
| Windows | Set by the `TcpTimedWaitDelay` registry value, commonly documented as 120 seconds on current versions, and older versions used 240 |

Real systems use less than the standard. In real networks, packets do not usually live for two minutes. The standard's value is a very safe guess.

**Reason 1: the last ACK may get lost.** The closer's final ACK has no ACK of its own. If it is lost, the peer in LAST-ACK resends its FIN. The closer must still remember the connection to answer it.

```text
 Without TIME-WAIT:

 Closer                                 Peer
   |<------------- FIN -------------------|  LAST-ACK
   |-------------- ACK --X  (lost)        |
 (forgets the connection)                 |
   |<------------- FIN again -------------|  retry
   |-------------- RST ------------------>|
                                       peer sees an error
                                       on a clean close
```

With TIME-WAIT, the closer still has the record. It sends the ACK again and restarts the timer. The peer closes cleanly.

**Reason 2: old packets must die first.** A later connection could use the same 4-tuple. A new connection with the same 4-tuple is called a new **incarnation**. A delayed packet from the old incarnation could arrive and its sequence number could fall inside the new connection's window (the range of sequence numbers a receiver accepts right now). The new connection would accept wrong data, without any error.

```text
 Time  Event
 ----  --------------------------------------------------------
 t=0   Old connection A:51498 to B:443 sends a segment
       with seq=9000. A router queue delays it.
 t=1   Old connection closes.
 t=2   New connection with the same four values starts.
       Its window happens to include seq=9000.
 t=3   The old segment arrives. The new connection takes
       it as valid data.

 TIME-WAIT blocks step t=2 until the old segment must be dead.
```

**Why two MSL and not one.** One MSL lets anything we sent die out. A second MSL covers the peer's reply, such as a resent FIN, to reach us or die. Together they cover one full round trip of the longest-lived packet.

Analogy: a phone company takes back a number and keeps it unused for a while. Late calls for the old owner stop before a new customer gets it. **Where it stops being true:** the phone company blocks the whole number. TIME-WAIT blocks only one exact 4-tuple. The server port stays fully usable for every other client.

**TIME-WAIT is usually harmless.** On Linux it does not use a file descriptor or any application memory. It uses a small kernel record of a few hundred bytes. A **file descriptor** is a small number a program uses to refer to an open file or socket.

**A server does not lose a port for each TIME-WAIT.** All its connections use local port 443. They differ by the client's IP and port. So inbound connections cannot exhaust a server's ports.

Fuller truth: TIME-WAIT does not fully block reuse of the tuple. RFC 9293 lets a host accept a new SYN during TIME-WAIT if the sequence number or timestamp proves the SYN is newer. Linux does this.

### CLOSE-WAIT in depth

CLOSE-WAIT means: "The peer is finished. I acknowledged it. My application has not closed yet."

A short stay here is normal. A server can receive the client's FIN and still spend a second sending the last of a response. A socket that stays here for minutes is a problem.

**The cause is almost always a bug in the application.** Typical bugs:

- An error path returns from a function without closing the socket.
- An HTTP response body is never closed. In Go, a missing `resp.Body.Close()` is a classic case.
- A connection pool hands out connections and never gets them back.
- A thread is blocked on something else, so it never reaches its `close()` call.
- The program never calls `read()`, so it never sees the end of the stream.

**Why the kernel cannot fix it.** The peer has closed only its sending direction. Your program may still be sending valid data. The kernel cannot guess that your program forgot. Only your program can say it is done.

**What it costs.** Each leaked socket holds a file descriptor and kernel buffers. When a process reaches its limit (`ulimit -n`, often 1024 by default for the soft limit), the `accept()` call fails with `Too many open files`. New clients then see hangs or refused connections, even though CPU and memory look fine.

Analogy: a customer leaves a restaurant. The waiter notes it, but only the manager can clear the table, and the manager never comes. Soon the restaurant is "full" of empty tables. **Where it stops being true:** a real waiter could clean up anyway. The kernel must not, because it cannot know if your program still needs the socket to send.

Fuller truth: the TCP connection can still die under a leaked socket. If your program writes, the peer may answer RST. If keepalive is on and the peer has gone, the connection is dropped. The state then leaves CLOSE-WAIT, but the file descriptor is still leaked until `close()`.

**The mirror image.** While you sit in CLOSE-WAIT, the peer sits in FIN-WAIT-2. On Linux, if the peer's program has closed the socket fully, it gives up after `tcp_fin_timeout` (default 60 seconds). Your side stays in CLOSE-WAIT with nobody left at the other end.

**Do not confuse the names.** CLOSE-WAIT means you must act. TIME-WAIT means you are done and are only waiting.

### Ephemeral ports and how they run out

An **ephemeral port** is a temporary local port the OS picks for an outgoing connection. Ephemeral means short-lived.

| System | Default range | Number of ports |
| --- | --- | --- |
| IANA suggested range (RFC 6335) | 49152-65535 | 16,384 |
| Linux default | 32768-60999 | 28,232 |
| Windows default | 49152-65535 | 16,384 |

**What is really limited.** The limit is not "28,232 connections in total". The limit is unique 4-tuples. When source IP, destination IP and destination port are fixed, only the source port can change. So you get one set of ports **for each destination**.

```text
 Source IP   Src port   Dest IP     Dst port
 ---------   --------   ---------   --------
 10.0.1.10   40001      10.0.2.20   8080       in use
 10.0.1.10   40002      10.0.2.20   8080       in use
 10.0.1.10   40001      10.0.2.21   8080       fine, new dest IP
 10.0.1.10   40001      10.0.2.20   8081       fine, new dest port
```

**Why TIME-WAIT matters here.** A closed connection keeps its 4-tuple for the whole TIME-WAIT period. So the sustained rate has a ceiling.

```text
 max new connections per second to one destination
     = usable ports / TIME-WAIT seconds
```

| Setup | Ports | TIME-WAIT | Ceiling to one destination |
| --- | --- | --- | --- |
| Linux default | 28,232 | 60 s | about 470 per second |
| Linux, range 10000-65535 | 55,536 | 60 s | about 925 per second |
| Windows default | 16,384 | 120 s | about 136 per second |

This hits **clients of one busy destination**. Typical victims are reverse proxies (servers that forward client requests to backend servers), API gateways, app servers that call one database or one cache without a pool, and load test tools. Plain web servers that only accept connections are not at risk.

**Symptoms:**

| Where | Error |
| --- | --- |
| Linux `connect()` | `Cannot assign requested address` (`EADDRNOTAVAIL`) |
| Windows Winsock | Error 10048 (`WSAEADDRINUSE`), sometimes 10055 (`WSAENOBUFS`) |
| Nginx log | `connect() failed (99: Cannot assign requested address)` |

Fuller truth: as the port table fills up, the kernel takes longer to find a free port, so `connect()` can slow down before it fails.

**NAT has the same problem.** A NAT gateway rewrites many private clients onto a few public IPs, and it has its own port limit per destination. For example, AWS documents a limit of 55,000 simultaneous connections from a NAT gateway to each unique destination. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

Analogy: a car park with 28,232 spaces. Each car stays 60 seconds after its driver has gone. If 800 cars arrive each second, it fills in about 35 seconds. **Where it stops being true:** each destination has its own separate car park, so adding a second destination gives you a second full set of spaces.

Fixes, best first:

1. **Reuse connections.** Use HTTP keep-alive (many requests on one connection) and connection pools (a set of open connections that the program shares). This removes the cause.
2. **Add more 4-tuples.** Add destination IPs or ports, or more source IPs.
3. **Widen the port range** (see the tuning section).
4. **Turn on `tcp_tw_reuse`** on the client side, on Linux, with timestamps on.
5. **Make the server close first,** if the protocol allows it, so that TIME-WAIT moves to the server.

### Seeing states: Linux

`ss` is the modern tool. It reads directly from the kernel and is fast with a large number of sockets. `netstat` comes from the older net-tools package and may not be installed.

Linux:

```bash
ss -tan                         # all TCP sockets, numeric
ss -tlnp                        # listeners, with process (use sudo)
ss -tan state time-wait         # only one state
ss -tan state close-wait        # the leak check
ss -tanp state close-wait       # with process name and PID
ss -tan 'sport = :443'          # local port 443
ss -tan 'dst 10.0.2.20'         # peer address
ss -tano                        # show timers
ss -s                           # summary counts
netstat -ant                    # the older tool
```

Meaning of the flags: `-t` TCP, `-a` all sockets, `-n` numbers instead of names, `-l` listeners only, `-p` process, `-o` timers, `-H` no header line.

Without `-a`, `ss` hides listening sockets and TIME-WAIT sockets. This fools many people, who think they have none.

Count sockets by state (Linux):

```bash
ss -Htan | awk '{print $1}' | sort | uniq -c | sort -rn
```

```text
  28944 TIME-WAIT
    186 ESTAB
    148 CLOSE-WAIT
     12 LISTEN
      6 FIN-WAIT-2
      3 SYN-RECV
      3 LAST-ACK
      2 FIN-WAIT-1
```

The same picture from the summary (Linux):

```bash
ss -s
```

```text
Total: 412
TCP:   29310 (estab 186, closed 28950, orphaned 2, timewait 28944)

Transport Total     IP        IPv6
RAW       1         0         1
UDP       6         4         2
TCP       360       352       8
INET      367       356       11
FRAG      0         0         0
```

An **orphan** is a socket the program has closed but the kernel is still finishing, for example while resending a FIN.

Read this machine like an engineer would:

- 28,944 TIME-WAIT: about 480 closes per second (28,944 / 60). Fine unless there are errors.
- 148 CLOSE-WAIT: a bug to chase.
- 186 ESTAB: the real load.

`Recv-Q` and `Send-Q` mean different things by state:

| Socket type | Recv-Q | Send-Q |
| --- | --- | --- |
| ESTAB and most others | Bytes received, not yet read by the program | Bytes sent, not yet acknowledged |
| LISTEN | Connections waiting for `accept()` | Maximum size of that queue |

### Seeing states: Windows

Windows:

```powershell
netstat -ano -p tcp                      # all TCP with PID
netstat -ano | findstr CLOSE_WAIT        # one state
Get-NetTCPConnection -State TimeWait     # PowerShell view

# Count by state
Get-NetTCPConnection | Group-Object State |
    Sort-Object Count -Descending | Select-Object Count, Name

# Who owns the CLOSE-WAIT sockets
Get-NetTCPConnection -State CloseWait |
    Group-Object OwningProcess | Sort-Object Count -Descending

# Ephemeral port range
netsh int ipv4 show dynamicport tcp
```

Sample `netstat` output:

```text
  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       4312
  TCP    10.0.0.20:443          10.0.0.5:51514         ESTABLISHED     4312
  TCP    10.0.0.20:443          10.0.0.5:51498         TIME_WAIT       0
  TCP    10.0.0.20:443          10.0.0.7:40112         CLOSE_WAIT      4312
```

A TIME_WAIT line shows PID 0 because no process owns that socket any more. The CLOSE_WAIT line shows a real PID. That PID is the process with the bug.

### Seeing states: Cisco IOS

A router's own TCP sessions have states too, for example BGP on port 179 or SSH on port 22. Traffic passing through the router is not shown. See [BGP](../05-routing/09-bgp.md).

Cisco IOS:

```text
Router# show tcp brief
TCB       Local Address           Foreign Address        (state)
6C2F1A30  10.0.12.1.179           10.0.12.2.34512        ESTAB
6C2F2B44  10.0.12.1.22            10.0.99.5.51022        ESTAB
```

IOS writes the port after the last dot. Column layout varies by release.

For more tools, see [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

### What a pile-up in each state tells you

| Many sockets in | Usual meaning | Where to look |
| --- | --- | --- |
| SYN-SENT | Server down, or a firewall drops packets | Network path, peer |
| SYN-RECV | SYN flood, or the final ACK is not returning | Attack, routing |
| ESTAB | Real load, or idle connections never closed | Pools, timeouts |
| FIN-WAIT-1 | Peer has gone, or its window is zero | Peer, network |
| FIN-WAIT-2 | Peer is not closing | CLOSE-WAIT on the peer |
| CLOSE-WAIT | **Your** program does not close | Your code |
| LAST-ACK | Peer vanished before the last ACK | Network, peer |
| CLOSING | Rare; lost ACKs if stuck | Network |
| TIME-WAIT | Many short connections | Usually fine; check for port errors |

## Example

### Example 1: a full close, packet by packet

The client is 10.0.0.5, port 51498. The server is 10.0.0.10, port 443. The client closes first.

```text
 Client 10.0.0.5:51498                 Server 10.0.0.10:443
 ESTABLISHED                                    ESTABLISHED
     |                                               |
     | close()                                       |
     |------- FIN  seq=1000 ack=5000 --------------->|
 FIN-WAIT-1                                    CLOSE-WAIT
     |                                               |
     |<------ ACK  seq=5000 ack=1001 ----------------|
 FIN-WAIT-2                                          |
     |                       server program sees EOF |
     |                                       close() |
     |<------ FIN  seq=5000 ack=1001 ----------------|
 TIME-WAIT                                      LAST-ACK
     |                                               |
     |------- ACK  seq=1001 ack=5001 --------------->|
     |                                            CLOSED
     | waits 60 s on Linux
 CLOSED
```

Notice that the FIN uses one sequence number. The ACK of `seq=1000` is `ack=1001`.

The same four packets in a capture on the server (Linux). `-S` shows absolute sequence numbers.

```bash
sudo tcpdump -nn -S -i eth0 'host 10.0.0.5 and tcp port 443'
```

```text
10:15:02.100200 IP 10.0.0.5.51498 > 10.0.0.10.443: Flags [F.], seq 1000, ack 5000, win 502, length 0
10:15:02.100650 IP 10.0.0.10.443 > 10.0.0.5.51498: Flags [.], ack 1001, win 509, length 0
10:15:02.104910 IP 10.0.0.10.443 > 10.0.0.5.51498: Flags [F.], seq 5000, ack 1001, win 509, length 0
10:15:02.105320 IP 10.0.0.5.51498 > 10.0.0.10.443: Flags [.], ack 5001, win 502, length 0
```

`[F.]` means FIN plus ACK. `[.]` means ACK only. The first ACK came quickly from the kernel. The server's FIN came about 4 ms later, when the server program closed. This gap shows why FIN and ACK are often separate.

On the client, right after the close (Linux):

```bash
ss -tano 'dst 10.0.0.10'
```

```text
State     Recv-Q Send-Q Local Address:Port  Peer Address:Port  Timer
TIME-WAIT 0      0      10.0.0.5:51498      10.0.0.10:443      timer:(timewait,57sec,0)
```

The timer counts down from 60 seconds. On the server there is no line at all. LAST-ACK ended with the final ACK.

### Example 2: building and finding a CLOSE-WAIT leak

This small Python 3 server has a bug on purpose. It accepts connections and never reads or closes them.

```text
# leak_server.py (Python 3, Linux)
import socket

srv = socket.socket()
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("0.0.0.0", 9000))
srv.listen(128)

keep = []                   # keeps sockets alive in Python
while True:
    conn, addr = srv.accept()
    keep.append(conn)       # BUG: never read, never closed
```

Run it on 10.0.0.10. From 10.0.0.5, open and close three connections (Linux, OpenBSD netcat):

```bash
for i in 1 2 3; do nc -z 10.0.0.10 9000; done
```

On the server (Linux):

```bash
sudo ss -tanp state close-wait '( sport = :9000 )'
```

```text
Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
1      0      10.0.0.10:9000      10.0.0.5:41022     users:(("python3",pid=2871,fd=4))
1      0      10.0.0.10:9000      10.0.0.5:41030     users:(("python3",pid=2871,fd=5))
1      0      10.0.0.10:9000      10.0.0.5:41038     users:(("python3",pid=2871,fd=6))
```

When you filter on one state, `ss` leaves out the State column.

How to read it:

- `pid=2871`: the process with the bug.
- `fd=4,5,6`: each leak holds a file descriptor.
- `Recv-Q` of 1: on Linux the FIN itself counts as one unread unit. The program has not even read to the end. A value of 0 usually means that it did read the end of the stream but still did not close.

On the client (Linux), the sockets sit in FIN-WAIT-2 for up to 60 seconds and then disappear. The server's three sockets stay.

Now follow the leak (Linux):

```bash
ls /proc/2871/fd | wc -l            # how many descriptors are open
grep 'open files' /proc/2871/limits # the limit
sudo lsof -p 2871 -a -i TCP         # list its TCP sockets
```

Next, take a thread or stack dump of the program to find where it stops. Tools vary by language: `jstack` for Java, `py-spy dump` for Python, and a goroutine dump for Go.

The fix is in the code. Read until the end of the stream, and close on every path.

```text
# Fixed loop (Python 3)
while True:
    conn, addr = srv.accept()
    with conn:                  # closes even on an exception
        while conn.recv(4096):  # b"" means the peer sent FIN
            pass
```

A restart clears the leak for now, but it will come back. In an emergency, `sudo ss -K state close-wait '( sport = :9000 )'` can destroy sockets if the kernel supports it. This does not fix the bug, and the program can get errors on those sockets.

### Example 3: port exhaustion at a reverse proxy

A reverse proxy at 10.0.1.10 sends traffic to one backend at 10.0.2.20:8080. It opens a new connection for each request and closes first. It handles 800 requests per second.

Do the math.

```text
Available ports (Linux default):    60999 - 32768 + 1 = 28,232
TIME-WAIT sockets needed:           800/s x 60 s      = 48,000
48,000 is greater than 28,232, so the ports run out.
Time until failure:                 28,232 / 800      = about 35 s
```

The log shows it (nginx on Linux):

```text
connect() failed (99: Cannot assign requested address) while connecting to upstream, upstream: "http://10.0.2.20:8080/"
```

Confirm it on the proxy (Linux):

```bash
sysctl net.ipv4.ip_local_port_range
ss -Htan state time-wait 'dst 10.0.2.20' | wc -l
```

```text
net.ipv4.ip_local_port_range = 32768	60999
28214
```

The count is almost equal to the port range. That is the proof.

**Correct fix:** reuse connections. With a pool, a few dozen long-lived connections carry the same load, and the TIME-WAIT count falls to nearly zero.

Nginx config:

```text
upstream backend {
    server 10.0.2.20:8080;
    keepalive 64;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

**Weaker fix:** a wider range gives 55,536 ports and a ceiling of about 925 per second. This only buys time, as the design still wastes a connection on every request.

## Going Deeper

### Tuning cautions

Rule one: **find the cause before you change a setting.** Most TCP state problems are design or code bugs. A setting can hide the bug and add a new one.

Rule two: change one thing, measure, and write down why.

Check these with `sysctl <name>` on your own kernel, as defaults can differ by version and distribution. The table lists values for current mainline Linux.

| Setting | Default | What it does | Caution |
| --- | --- | --- | --- |
| `net.ipv4.ip_local_port_range` | 32768 60999 | Ephemeral port range | Too wide can collide with your own services |
| `net.ipv4.ip_local_reserved_ports` | empty | Keeps listed ports out of the automatic range | Use it when you widen the range |
| `net.ipv4.tcp_tw_reuse` | 2 (loopback only) | Reuse TIME-WAIT for new outgoing connections | Needs timestamps; applies only to the connecting side |
| `net.ipv4.tcp_tw_recycle` | Removed | Was a faster TIME-WAIT expiry | Removed in Linux 4.12; broke NAT users |
| `net.ipv4.tcp_fin_timeout` | 60 | Time an orphaned socket stays in FIN-WAIT-2 | Does **not** change TIME-WAIT |
| `net.ipv4.tcp_max_tw_buckets` | Scales with memory | Cap on TIME-WAIT sockets | When exceeded, sockets die early and protection is lost |
| `net.ipv4.tcp_max_syn_backlog` | Scales with memory | Limit of half-finished handshakes | Masks real load or an attack |
| `net.core.somaxconn` | 4096 (128 before Linux 5.4) | Cap on `listen()` backlog | The program must also ask for a larger backlog |
| `net.ipv4.tcp_syncookies` | 1 on most systems | Survive SYN floods | Leave it on |
| `net.ipv4.tcp_keepalive_time` | 7200 | Idle seconds before the first keepalive | Lower it per socket if you can |

Specific traps:

- **`tcp_fin_timeout` does not shorten TIME-WAIT.** This is the most common wrong tip online. It applies to orphaned FIN-WAIT-2 sockets only.
- **Standard Linux has no sysctl for the TIME-WAIT length.** It is a compile-time constant. Some vendor kernels add a setting, so check yours.
- **`tcp_tw_recycle` is gone.** It compared timestamps per source IP. Many users behind one NAT have different clocks, so valid SYNs were dropped without any notice. If a guide still mentions it, the guide is old.
- **`tcp_tw_reuse=1` does not help a server** that receives connections. It only applies when this host makes an outgoing connection. It reuses a tuple after about one second since the last timestamp.
- **`SO_LINGER` with a zero timeout** skips TIME-WAIT by sending RST on close. This throws away unsent data, and the peer sees "connection reset". It treats a symptom by breaking the clean close.
- **Widening the range to 1024** can grab a port that a service needs at start-up. That service then fails with `Address already in use`. Keep some distance, or use `ip_local_reserved_ports`.
- **Parity of the range matters a little.** Linux prefers different parity for start and end (one even, one odd) and logs a warning otherwise. It uses even ports for `connect()` by preference.
- **Containers have their own values.** Many `net.ipv4.*` settings are per network namespace. Setting them on the host may not affect a pod.
- **Windows too:** change `TcpTimedWaitDelay` only with a reason. It is a registry value under `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters` (range 30-300 seconds) and needs a reboot.

Apply a Linux change so that it survives a reboot:

```bash
echo 'net.ipv4.ip_local_port_range = 10000 65535' | sudo tee /etc/sysctl.d/90-ports.conf
sudo sysctl --system
```

Windows equivalent (run as Administrator):

```powershell
netsh int ipv4 set dynamicport tcp start=10000 num=55535
```

### Why servers often get TIME-WAIT

HTTP/1.0 servers closed after each response. HTTP/1.1 keep-alive servers close idle connections on a timer. In both cases the server closes first, so TIME-WAIT sits on the server. This is fine, because it uses no ports and little memory.

At 5,000 closes per second with 60 seconds of TIME-WAIT, you hold 300,000 sockets. At a few hundred bytes each, that is tens of megabytes. Compare it with the cost of the workaround before you "fix" it.

### The accept queue

A listening socket has two queues on Linux:

```text
 SYN arrives        final ACK arrives         accept() called
     |                     |                        |
     v                     v                        v
 +-----------+       +--------------+         +-----------+
 | SYN queue | ----> | accept queue | ------> |  program  |
 | SYN-RECV  |       | ESTABLISHED  |         |           |
 +-----------+       +--------------+         +-----------+
```

The accept queue holds connections the kernel has finished, waiting for the program. Its size is the smaller of `somaxconn` and the program's `listen()` backlog.

If a program is slow, this queue fills. Check it with `ss -ltn`, where Recv-Q near Send-Q is a warning. Also check the counters:

```bash
nstat -az TcpExtListenOverflows TcpExtListenDrops
```

If these counters grow, packets are being dropped at the listener. Clients see delays or timeouts while the server looks healthy.

### TIME-WAIT assassination

An old RST can reach a socket in TIME-WAIT. Under the original rules, that RST ends TIME-WAIT early, and the protection is lost. RFC 1337 calls this TIME-WAIT assassination and says TIME-WAIT should ignore RST. Linux has `net.ipv4.tcp_rfc1337` for this. The default is 0, and setting 1 makes TIME-WAIT ignore RST.

### Timestamps and PAWS

The TCP timestamps option (RFC 7323) puts a clock value in each segment. **PAWS** (Protection Against Wrapped Sequence numbers) uses it to reject old segments.

Sequence numbers are 32-bit. At 10 Gbit/s, they wrap around in about 3.4 seconds. That is far shorter than any MSL. So on fast links, timestamps protect a connection where the sequence number alone cannot.

This is also why `tcp_tw_reuse` can be safe: the timestamp shows which segments are newer. RFC 6191 describes how to use timestamps to reopen a tuple from TIME-WAIT safely.

More on sequence numbers and retransmission is in [TCP Reliability and Flow Control](05-tcp-reliability-and-flow-control.md).

### SO_REUSEADDR, SO_REUSEPORT and restart errors

When a server restarts, old connections may still be in TIME-WAIT on port 443. Without an option, `bind()` can fail with `Address already in use`.

- `SO_REUSEADDR` (Linux): lets a listener bind even with TIME-WAIT sockets on that port. Most servers set it. It does **not** let two programs listen on the same port.
- `SO_REUSEPORT` (Linux 3.9 and later): lets several sockets listen on one port, and the kernel spreads new connections among them.
- Windows differs: `SO_REUSEADDR` there is looser and can let another process take over the port. `SO_EXCLUSIVEADDRUSE` is the protection.

### How Linux picks a local port

RFC 6056 describes port randomization, which makes blind attacks harder. Linux hashes the 4-tuple and an offset to choose a start point, then searches for a free port.

A trap: if a program calls `bind()` before `connect()`, the kernel must fix the port before it knows the destination. The port is then lost for other destinations too, so one port does less work. `IP_BIND_ADDRESS_NO_PORT` (Linux 4.2 and later) delays the choice until `connect()`.

### Half-close and half-open are different

**Half-close** means one direction is closed and the other is open. It is a real TCP feature, made by `shutdown(fd, SHUT_WR)`. The client states "I finished my request" and still reads the reply. One side is in FIN-WAIT-2 and the other in CLOSE-WAIT, and this is healthy. Note that the `tcp_fin_timeout` limit does not apply here, as the socket is not an orphan.

**Half-open** has two meanings:

- A handshake not yet finished (SYN-RECEIVED).
- A connection where one side has lost its state, for example after a crash or power loss. The other side still shows ESTABLISHED and does not know. Its next packet gets an RST. If it never sends, it can stay forever.

Keepalive finds the second case. **Keepalive** is an empty probe sent on an idle connection. On Linux the default waits 7200 seconds, then sends 9 probes at 75-second gaps. This is off unless the program sets `SO_KEEPALIVE`. Many applications use their own heartbeat instead.

Similarly, firewalls and NAT devices often forget idle connections. Both ends still say ESTABLISHED, yet their packets are dropped. Keepalive or a heartbeat shorter than the idle timeout avoids this. See [Firewalls](../09-security/04-firewalls.md).

### The states in the middle are separate

A firewall or NAT box tracks its own copy of the state. On Linux, this is `conntrack`, with its own timers. `nf_conntrack_tcp_timeout_established` defaults to 432000 seconds (5 days), and `nf_conntrack_tcp_timeout_time_wait` to 120 seconds.

Its table can fill even when the host looks fine. The log shows `nf_conntrack: table full, dropping packet`. See [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

### How QUIC differs

[QUIC](08-quic.md) runs over UDP and manages its own connection states in user space, so there is no kernel TIME-WAIT. RFC 9000 defines closing and draining states that last at least three times the probe timeout. Its connection IDs mean that a new connection does not depend on the 4-tuple the way TCP does.

## Common Mistakes

- **"The client always gets TIME-WAIT."** Wrong. The side that closes first gets it.
- **"TIME-WAIT is a bug to remove."** It protects data integrity. Remove the cause of too many connections, not the state.
- **"TIME-WAIT on a server uses up its ports."** Inbound connections share the listening port. They are told apart by the client's IP and port.
- **"`tcp_fin_timeout` sets the TIME-WAIT length."** It sets the orphaned FIN-WAIT-2 limit only.
- **"CLOSE-WAIT is a network problem."** It is almost always your own application not calling `close()`.
- **"Every CLOSE-WAIT socket is a leak."** A short stay is normal. A count that grows and never falls is the leak.
- **"CLOSE-WAIT will time out."** TCP has no standard timer for it.
- **"Restart fixed it, so it is solved."** A restart frees the sockets and leaves the bug.
- **"I have 28,232 ports, so I can only have 28,232 connections."** The limit is per destination 4-tuple. A busy server can hold far more connections.
- **"`close()` always sends FIN."** It sends RST if unread data remains, or if `SO_LINGER` is set to zero. FIN also waits for queued data.
- **"`accept()` does the handshake."** The kernel finishes the handshake. `accept()` only picks up the result.
- **"`ss -t` shows nothing in TIME-WAIT, so there are none."** Use `-a`. The default view hides them.
- **"Copy this list of sysctls for better performance."** Many old lists include removed settings or change things for no reason.
- **Mixing up FIN-WAIT-2 and CLOSE-WAIT.** They are two ends of the same wait. The first says "I wait for them." The second says "My program must act."

## Interview Questions

**Q: Name the TCP states and describe the normal path for each side.**
A: There are 11: CLOSED, LISTEN, SYN-SENT, SYN-RECEIVED, ESTABLISHED, FIN-WAIT-1, FIN-WAIT-2, CLOSE-WAIT, CLOSING, LAST-ACK and TIME-WAIT. A client goes SYN-SENT to ESTABLISHED. A server goes LISTEN to SYN-RECEIVED to ESTABLISHED. The first closer goes FIN-WAIT-1, FIN-WAIT-2, TIME-WAIT, then CLOSED. The second closer goes CLOSE-WAIT, LAST-ACK, then CLOSED.

**Q: Why does TIME-WAIT exist?**
A: Two reasons. First, the final ACK can be lost, so the closer must stay to answer a resent FIN. Second, it lets old delayed segments die, so they cannot be accepted by a new connection on the same 4-tuple.

**Q: Why is it 2 MSL and not 1?**
A: One MSL covers my last segment dying out. The second covers a reply from the peer, like a resent FIN, reaching me. Together they cover one full round trip at the worst-case packet lifetime. RFC 9293 sets MSL at 2 minutes, but Linux uses a fixed 60 seconds in total.

**Q: Who enters TIME-WAIT, the client or the server?**
A: Whichever side sends the first FIN, called the active closer. It can be either. In a simultaneous close, both sides enter it.

**Q: A server has 50,000 sockets in TIME-WAIT. Is that a problem?**
A: Usually not. It means many short connections and the server is closing first. They share the listening port, hold no file descriptor, and use little memory. I would check for real errors first. If there are none, I would leave it. If this host also makes outgoing connections to one destination, I would look at ports and use pooling.

**Q: A server has thousands of sockets in CLOSE-WAIT and keeps growing. What do you do?**
A: The peer closed and our application did not. It is a bug in our code, not the network. I use `ss -tanp state close-wait` to find the process, compare its open descriptors with its limit, then take a stack dump to find where it stops. The fix is to close on every path. A restart is only a short-term fix.

**Q: Why is closing four packets when opening is three?**
A: Each direction closes on its own. The kernel acknowledges a FIN at once, but the second FIN must wait for the application, which may still have data to send. So ACK and FIN are often separate. If the application is already done, they can share one segment, which gives three packets.

**Q: What does `Cannot assign requested address` on `connect()` mean?**
A: On Linux it usually means no local port is free for that destination, often because of TIME-WAIT sockets from many short connections. I compare the TIME-WAIT count to that destination with the port range. The ceiling is about 470 new connections per second to one destination on Linux defaults. The best fix is connection reuse. After that come more destination IPs, a wider range, or `tcp_tw_reuse` on the client.

**Q: What is the difference between `tcp_tw_reuse` and `tcp_tw_recycle`?**
A: `tcp_tw_reuse` lets a host reuse a TIME-WAIT tuple for a new outgoing connection when timestamps show it is safe. `tcp_tw_recycle` shortened TIME-WAIT based on per-IP timestamps and dropped valid SYNs from users behind NAT. It was removed in Linux 4.12.

**Q: What is the difference between FIN and RST?**
A: FIN is a graceful end of one direction. Queued data is delivered first, and the other side can keep sending. RST aborts the whole connection at once, discards data, needs no ACK, and the peer gets a reset error. RST also means no TIME-WAIT.

**Q: A server restart fails with `Address already in use`. Why?**
A: Old connections on that port may be in TIME-WAIT, or another process may be listening. I check with `ss -tlnp`. If only TIME-WAIT sockets remain, the server should set `SO_REUSEADDR` before `bind()`. It will not let two listeners share the port.

**Q: What can you learn from sockets stuck in FIN-WAIT-2?**
A: I closed and the peer acknowledged, but the peer has not sent its FIN. The peer's application is the usual suspect, and it probably shows CLOSE-WAIT. On Linux, orphaned sockets give up after `tcp_fin_timeout`, 60 seconds by default. A socket half-closed with `shutdown()` can stay here longer.

## Quick Recap

- TCP has 11 states. Each side tracks its own state, and the two sides often differ.
- The side that closes first ends in TIME-WAIT. The side that closes second passes through CLOSE-WAIT and LAST-ACK.
- TIME-WAIT lasts 2 x MSL (60 seconds on Linux) so that a lost final ACK can be resent and old segments can die.
- Growing CLOSE-WAIT means your application did not call `close()`. Fix the code, not the kernel.
- Port exhaustion is about unique 4-tuples per destination. It hits proxies and clients, and connection reuse is the real cure.
- Use `ss -tan` and `ss -s` on Linux, and `netstat -ano` or `Get-NetTCPConnection` on Windows. Remember that `ss` needs `-a` to show TIME-WAIT.
- Before tuning, find the cause. `tcp_fin_timeout` does not change TIME-WAIT, and `tcp_tw_recycle` no longer exists.
