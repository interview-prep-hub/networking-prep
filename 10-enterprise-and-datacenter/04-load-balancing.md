# Load Balancing

**Level:** Intermediate | **Prerequisites:** [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [DNS Basics](../07-application-protocols/01-dns-basics.md), [HTTP Basics](../07-application-protocols/05-http-basics.md), [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md)

## In One Line

A load balancer is a device or program that accepts traffic on one address and shares it across many servers, so that no single server is overloaded and a dead server does not take the service down.

## Why It Matters

One server has limits. It can handle only so many connections, and it can crash, or need a reboot for a patch.

If `www.example.com` points to one server, every failure is an outage. A load balancer solves three problems at once:

- **Scale.** You add servers instead of buying a bigger one.
- **Availability.** Dead servers are removed from service automatically.
- **Safe change.** You can take a server out, patch it, and put it back while users keep working.

Interviewers ask about load balancing because it joins many topics into one: TCP, HTTP, TLS, NAT, DNS and BGP. It is also a core part of nearly every system design interview. Expect questions such as "L4 or L7?", "How do you keep a user on one server?", and "What happens to live connections when you remove a server?"

## Simple Explanation

Think of a large bank branch with one front door and ten counters.

A staff member stands at the door. Each customer walks to that person, and that person points them to a free counter. Customers do not need to know how many counters exist. If counter 4 closes, the staff member stops sending people there. If the queue grows, the bank opens more counters.

In this picture:

| Bank | Network |
|---|---|
| The front door | The virtual IP address (VIP): one public address for the service |
| The staff member | The load balancer |
| The counters | The backend servers |
| Checking that a counter is open | A health check |
| "Go back to the same clerk, she has your file" | Session persistence |
| "We are closing, finish your business but no new customers" | Connection draining |

**Where the analogy stops being true:** a real load balancer can handle millions of packets per second. It can also stay in the middle of the conversation, passing each message back and forth, while the bank staff member steps away after pointing. Only some modes (such as DSR, explained later) behave more like the bank, where the reply does not pass through the load balancer.

## How It Works

### The basic parts

These words appear everywhere, so learn them first.

- **Virtual IP (VIP):** the address clients connect to. It belongs to the service, not to one physical server.
- **Frontend or listener:** the IP and port where the load balancer accepts traffic, for example `203.0.113.10:443`.
- **Backend, real server, upstream or target:** a server that does the actual work. These words mean nearly the same thing. HAProxy says "server in a backend", NGINX says "upstream", and AWS says "target".
- **Pool or server farm:** the group of backends for one service.
- **Algorithm:** the rule that picks a backend.
- **Health check:** a regular test of whether a backend can take traffic.

```text
                                      +----------------+
                                 +--->| web1 10.0.1.11 |
                                 |    +----------------+
 Client           +--------------+    +----------------+
 198.51.100.77 -->| LB           |--->| web2 10.0.1.12 |
                  | 203.0.113.10 |    +----------------+
                  +--------------+    +----------------+
                                 +--->| web3 10.0.1.13 |
                                      +----------------+
```

The basic flow is:

1. DNS resolves `www.example.com` to the VIP.
2. The client connects to the VIP.
3. The load balancer picks a healthy backend with its algorithm.
4. The load balancer sends the traffic to that backend.
5. The reply returns to the client, usually through the load balancer.

### L4 vs L7

The numbers come from the [OSI model](../02-network-models/02-osi-model.md). Layer 4 is the transport layer (TCP, UDP). Layer 7 is the application layer (HTTP and others).

**Analogy.** An L4 load balancer is like a post room worker who reads only the envelope. An L7 load balancer opens the letter and reads it, then sends it to the right department. The analogy stops being true because an L7 device can also change the letter before passing it on.

**L4 load balancing** uses only the IP addresses, ports and protocol. It does not know about URLs or cookies.

A connection is identified by its **5-tuple**: source IP, source port, destination IP, destination port and protocol. The load balancer picks a backend when the connection starts, then records the 5-tuple in a table. Every later packet with the same 5-tuple goes to the same backend.

**L7 load balancing** reads the application data. For HTTP, it sees the method, path, headers and cookies. It can then make smart decisions:

```text
GET /api/orders HTTP/1.1
Host: www.example.com
Cookie: SRV=web2
```

From this request, an L7 load balancer can see the path `/api/` and send it to the API pool. It can read `Host` and serve many sites from one VIP. It can read `SRV=web2` and keep the user on web2.

| Point | L4 | L7 |
|---|---|---|
| Looks at | IP, port, protocol | HTTP method, path, headers, cookies |
| Decision made | When the connection starts | After reading the request, can be per request |
| Protocols | Any TCP or UDP service | Only protocols it understands, mostly HTTP, gRPC |
| Routing by URL or host | No | Yes |
| Cookie persistence | No | Yes |
| TLS | Usually passes it through | Must decrypt it to see inside |
| Can change data | No | Yes, can add headers, rewrite paths, retry |
| CPU per request | Lower | Higher |
| Typical use | Databases, mail, game servers, very high traffic | Websites, APIs, microservices |

**Fuller truth: "L4" hides two very different designs.** Interviewers like this point.

```text
Packet forwarding: one TCP connection, end to end

Client ================= TCP =================> Backend
                  LB only rewrites addresses

Full proxy: two separate TCP connections

Client ==== TCP 1 ====> LB ==== TCP 2 ====> Backend
```

- A **packet-forwarding** L4 balancer (Linux IPVS, Google Maglev, many cloud network load balancers) changes packet headers and forwards them. The client and backend do the TCP handshake with each other.
- A **full proxy** ends the client's TCP connection and opens a new one to the backend. HAProxy in `mode tcp` and the NGINX `stream` module work this way, even though they make only L4 decisions. All L7 load balancers are full proxies.

Here is how the addresses change in NAT mode, for a packet going to the backend:

| Step | Source | Destination |
|---|---|---|
| Client sends | 198.51.100.77:51000 | 203.0.113.10:443 |
| DNAT only (destination rewritten) | 198.51.100.77:51000 | 10.0.1.11:8080 |
| DNAT plus SNAT (both rewritten) | 10.0.1.5:40001 | 10.0.1.11:8080 |

With DNAT only, the backend sees the real client IP, but its replies must go back through the load balancer, so the load balancer must be its gateway. With SNAT as well, replies return naturally, but the backend sees only the load balancer's address. "Going Deeper" shows how to recover the client IP.

### Load balancing algorithms

An algorithm answers one question: which backend gets this next connection or request?

**Round robin.** Take turns: web1, web2, web3, web1, and so on. It is simple and is the default in both HAProxy and NGINX. It suits servers of equal size and requests of similar cost.

**Weighted round robin.** A stronger server gets a bigger share. With weights a=5, b=1, c=1, server a gets 5 of every 7 requests. NGINX uses a "smooth" version that spreads them out instead of sending five in a row: `a a b a c a a`.

**Least connections.** Send to the server with the fewest active connections. Use it when connections last different lengths of time, such as database sessions or WebSockets.

| Server | Active connections | Next request? |
|---|---|---|
| web1 | 10 | No |
| web2 | 2 | Yes, the lowest |
| web3 | 5 | No |

**Weighted least connections.** Divide connections by weight and pick the lowest result. With web1 at weight 4 and 20 connections, the score is 20 / 4 = 5. With web2 at weight 1 and 6 connections, the score is 6 / 1 = 6. web1 wins even though it holds more connections, because it is four times stronger.

**Least response time.** Send to the server that is answering fastest, often combined with connection count. NGINX Plus calls this `least_time`. Some proxies, such as Linkerd, use a moving average of latency (EWMA, exponentially weighted moving average, which gives recent samples more weight).

**Random.** Pick any server. It works well at large scale, and many load balancers can use it without sharing state.

**Power of two choices.** Pick two servers at random, then use the one with fewer connections. This is close to least connections without needing to compare every server, and it avoids many load balancers all choosing the same "best" server. It is available as `balance random(2)` in HAProxy and `random two least_conn` in NGINX.

**Hash-based.** Compute a hash (a function that turns any input into a fixed number) from some field and use it to pick a server. The field can be the source IP, the URL, a cookie or a header. The same input always gives the same server while the pool stays the same. Use it for caches and simple persistence. See "Consistent hashing" below for what happens when the pool changes.

| Algorithm | Best for | Weakness |
|---|---|---|
| Round robin | Equal servers, short requests | Ignores current load |
| Weighted round robin | Mixed server sizes | Ignores current load |
| Least connections | Long or uneven requests | Connection count is not the same as work |
| Least response time | Mixed latency | Needs measurement, can chase noise |
| Power of two choices | Many load balancers, large pools | Slightly less exact than a full comparison |
| Source IP hash | Basic persistence at L4 | Uneven when many users share one IP |
| URL hash | Cache servers | Popular URLs can overload one server |

### Health checks

A health check is a regular test to see whether a backend should get traffic. Without it, the load balancer keeps sending users to a dead server.

**Analogy.** A manager phones each shop every few seconds and asks "Are you open?" The analogy stops being true because a good check does more than ask. It places a small real order to see that the shop can serve one.

**Active checks.** The load balancer sends its own test traffic. There are several levels:

| Level | Test | What it proves |
|---|---|---|
| L3 | ICMP ping | The host is up, nothing about the service |
| L4 | TCP connect to the port | Something is listening |
| TLS | Complete a TLS handshake | TLS works and the service answers |
| L7 | `GET /healthz`, expect 200 | The application can answer HTTP |
| Deep | Endpoint also tests the database | The whole stack works, but see the warning below |

**Passive checks.** The load balancer watches real traffic. If real requests fail with connection errors or timeouts, it marks the server as bad. No extra traffic is needed, but real users must fail first.

Three settings control active checks:

- **Interval:** how often to check, for example every 2 seconds.
- **Fall (unhealthy threshold):** how many failures in a row mark the server down.
- **Rise (healthy threshold):** how many successes in a row bring it back.

With an interval of 2 seconds and fall of 3, detection takes about 6 seconds. During that time, some requests may fail. A shorter interval detects faster but sends more check traffic.

Why need several failures? A single lost check should not remove a server. Going up and down again and again is called **flapping**. Different fall and rise values prevent it.

Defaults differ by product. HAProxy uses `inter 2s`, `fall 3`, `rise 2`. Open source NGINX has passive checks with `max_fails=1` and `fail_timeout=10s`. Active checks with the `health_check` directive are an NGINX Plus (commercial) feature.

**Warning about deep checks.** Suppose every web server's `/healthz` tests the shared database. If the database slows down for 10 seconds, every web server fails its check at once. The load balancer removes all of them. A small problem becomes a full outage. Keep the load balancer check focused on "can this server serve?", and monitor dependencies separately.

**What if all backends fail?** Some systems "fail open" and send traffic to all of them anyway, on the theory that a bad check is more likely than a total failure. AWS Application Load Balancer does this when all targets are unhealthy. Envoy has a similar "panic threshold", which is 50% healthy hosts by default.

### Session persistence

**Session persistence**, also called **stickiness** or **affinity**, means sending the same user to the same backend for a period of time.

Do not confuse the two meanings of "session". A TCP connection stays on one backend by nature. Persistence is about a user who makes many requests, often over several connections.

It is needed when a server keeps user data in its own memory, such as a shopping cart. If the next request lands on another server, the cart looks empty.

**Analogy.** You always see the same doctor, because she has your notes in her desk. It works until she is sick. The better fix is a shared medical record that any doctor can open. That is the point of a shared session store. The analogy stops being true because a server can lose your data in an instant when it crashes.

| Method | How it works | Layer | Problem |
|---|---|---|---|
| Source IP | Hash the client IP | L4 | Many users behind NAT land on one server. Mobile users change IP |
| LB-inserted cookie | LB adds `Set-Cookie: SRV=web1` | L7 | Needs decryption if HTTPS |
| Application cookie | LB tracks a cookie such as `JSESSIONID` | L7 | LB must store a table |
| URL or header | Hash a token in the request | L7 | Client must send it each time |
| TLS session ID | Track the TLS session | L4 | Weak. Not usable in TLS 1.3, where session IDs carry no such meaning |

Here is how an inserted cookie works:

1. The first request has no cookie. The load balancer picks web1 with its normal algorithm.
2. The load balancer adds `Set-Cookie: SRV=web1` to the response.
3. The browser sends `Cookie: SRV=web1` on each next request.
4. The load balancer reads it and picks web1. It can remove the cookie before the backend sees it.
5. If web1 dies, the load balancer picks a new server. The cookie is reset, and data held only on web1 is lost.

**The best design** is often to avoid the need: store session data in a shared place such as Redis or a database, or use signed tokens that the client carries. Then any server can take any request. Stickiness causes uneven load, slows draining, and loses data when a server fails. Use it only when you must.

### SSL termination

TLS is the encryption behind HTTPS. SSL is the older, retired name, but the industry still says "SSL termination" and "SSL offload". Details are in [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

**SSL termination** means the load balancer ends the TLS session. It holds the certificate and private key, decrypts the traffic, and handles the request. There are three common designs.

```text
1. Termination (offload)
   Client ===TLS===> LB ---plain HTTP---> Backend

2. Bridging (re-encryption)
   Client ===TLS===> LB ===new TLS======> Backend

3. Passthrough
   Client =================TLS===========> Backend
                     LB only forwards
```

| Point | Termination | Bridging | Passthrough |
|---|---|---|---|
| Where the certificate lives | LB | LB and backend | Backend |
| LB can read HTTP | Yes | Yes | No |
| L7 routing and cookies | Yes | Yes | No |
| Backend CPU for TLS | None | Yes | Yes |
| Encrypted inside the network | No | Yes | Yes |
| Typical use | Trusted internal network | Most security-minded setups, zero trust | Strict end-to-end rules |

**Analogy.** Termination is a reception desk that opens all sealed mail and delivers it unsealed inside the building. Bridging is the same desk resealing it for internal delivery. Passthrough is a desk that hands over the sealed envelope. The analogy stops being true because reading the outside of a TLS hello is still possible: see SNI below.

Why terminate at the load balancer?

- One place to install and renew certificates.
- The load balancer can read HTTP, so it can do L7 routing, cookies, and web application firewall checks.
- Backends spend no CPU on the TLS handshake and encryption.

Things to know:

- **SNI (Server Name Indication):** the client sends the hostname in the TLS hello. One VIP can then pick the correct certificate for many sites. A passthrough load balancer can also read SNI to choose a backend without decrypting. Encrypted Client Hello (ECH), a newer TLS extension, is designed to hide this name, which limits this trick where it is used.
- **ALPN (Application-Layer Protocol Negotiation):** client and server agree on HTTP/2 or HTTP/1.1 during the handshake.
- **Inspection after TLS 1.3:** TLS 1.3 removed static RSA key exchange, so a device cannot decrypt recorded traffic by holding the server's private key. It must terminate the session itself.
- **Client certificates (mTLS):** if the client must prove its identity with a certificate, the load balancer must verify it, or use passthrough so the backend can.

### DSR (Direct Server Return)

In normal modes, requests and replies both pass through the load balancer. With **Direct Server Return (DSR)**, requests go through the load balancer, but replies go straight from the backend to the client.

Why bother? Responses are usually much bigger than requests. A request may be 1 KB and the answer 1 MB. With DSR, the load balancer carries only the small side, so it can serve far more traffic.

**Analogy.** You order by phone through a call centre. The order goes through an agent, but the delivery van comes from the warehouse to your house, and it does not go via the call centre. The analogy stops being true because the van driver does not pretend to be the call centre, while a DSR backend must send replies using the VIP as the source address.

```text
              (1) request to VIP
Client ---------> Router ---------> Load balancer
198.51.100.77                       VIP 203.0.113.10
   ^                                    |
   |                                    | (2) same IP packet,
   |                                    |     new destination MAC
   |                                    v
   |                                Backend 10.0.1.11
   |                                VIP also on loopback
   |                                    |
   +------------- Router <--------------+
              (3) reply: src 203.0.113.10
                         dst 198.51.100.77
```

The steps for L2 DSR (also called direct routing) are:

1. The client sends a packet to the VIP.
2. The load balancer picks a backend. It changes only the destination MAC address to the backend's MAC. The IP header stays the same.
3. The backend gets a packet addressed to the VIP. It accepts it because the VIP is configured on its loopback interface.
4. The backend replies directly, with the VIP as source address, through its normal gateway.
5. The client sees replies from the VIP and notices nothing.

**Requirements and limits:**

- In L2 DSR, the load balancer and backends must be in the same broadcast domain, because only the MAC is changed.
- Backends must not answer ARP for the VIP, or they would steal traffic from the load balancer. On Linux this is what `arp_ignore` and `arp_announce` control.
- The load balancer sees only half of each conversation, so it cannot do L7 work on responses. It cannot terminate TLS, insert cookies or retry failed responses.
- Passive checks are limited because the load balancer does not see the response.
- Some networks drop packets whose source address looks wrong for that network, so check anti-spoofing rules on the reply path.

**L3 DSR.** To cross subnets, the load balancer wraps each packet in a tunnel, such as IP-in-IP or GRE, and the backend unwraps it. The extra header lowers the room for data, so watch [MTU and MSS](../06-transport-layer/09-mss-mtu-and-pmtud.md). This design is used by Google Maglev and Meta's Katran.

### Connection draining

**Connection draining** means a backend gets no new work, but its existing work is allowed to finish. AWS calls it "deregistration delay". Others call it "graceful shutdown".

**Analogy.** A shop at closing time locks the entrance, but customers already inside can finish and pay. The analogy stops being true because a customer who never leaves (like a WebSocket connection that stays open for days) must eventually be asked to leave, so a time limit is needed.

```text
time -->

state:     UP          DRAINING            REMOVED
new work:  yes         no                  no
old work:  yes         finishing           ended by deadline
           |           |                   |
           |           drain starts        timeout, for example 300 s
```

The steps for a safe deployment are:

1. Tell the load balancer to drain the server.
2. The load balancer stops sending new connections there.
3. Existing connections finish. HTTP/1.1 servers can send `Connection: close`. HTTP/2 servers can send a GOAWAY frame, which tells the client to open a new connection for further requests.
4. Wait until the active count reaches 0, or the drain timeout expires.
5. Stop the service, patch it and restart it.
6. Run health checks. When they pass, bring it back, ideally with slow start (see "Going Deeper").

Timeout choice matters. AWS target groups default to 300 seconds. If requests finish in 1 second, 300 seconds slows every deployment for nothing. If you have 2-hour file uploads, 300 seconds cuts them off.

**A trap with stickiness:** in HAProxy `drain` state, a server still receives requests from clients that are already stuck to it through persistence. Draining can take much longer than expected if the sticky session lifetime is long.

### Global server load balancing

So far, one load balancer has shared traffic inside one site. **Global Server Load Balancing (GSLB)** shares traffic between sites, such as Frankfurt and Singapore. It gives users a nearby site and lets the service survive the loss of a whole data centre.

**Analogy.** A national help line asks where you live, and gives you the number of your local branch. If that branch burns down, it gives you the next nearest. The analogy stops being true because people may keep the old number written down, just as DNS caches keep old answers.

**DNS-based GSLB.** The authoritative DNS server gives a different answer based on the client's location and site health. See [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

```text
         "where is www.example.com?"
Client in Germany ----> Resolver ----> GSLB DNS 192.0.2.53
                                             |
       Frankfurt healthy, closest to client  |
       answer 203.0.113.10, TTL 30           |
                <----------------------------+

+----------------------+        +----------------------+
| Frankfurt            |        | Singapore            |
| VIP 203.0.113.10     |        | VIP 203.0.113.20     |
| LB + backends        |        | LB + backends        |
+----------------------+        +----------------------+
```

The steps are:

1. GSLB monitors each site, usually from several places.
2. The client's resolver asks for `www.example.com`.
3. GSLB chooses a site using health first, then a policy such as location, latency, weight or failover.
4. GSLB returns that site's VIP with a short TTL (time to live, how long the answer may be cached).
5. The client connects directly to that site. GSLB is not in the data path.

Limits of DNS-based GSLB:

- **Caching.** Failover takes about the detection time plus the TTL. With 3 checks at 10-second intervals and a TTL of 60 seconds, plan for up to about 90 seconds. Some resolvers and applications hold answers longer than the TTL.
- **Resolver location.** GSLB sees the resolver, not the user. A user may be far from their resolver. EDNS Client Subnet (RFC 7871) lets a resolver pass part of the client's address, if it chooses to.
- **Crude balance.** DNS decides where to send a resolver's users, but it cannot see how many users sit behind one resolver.

**Anycast.** Many sites announce the same IP prefix with [BGP](../05-routing/09-bgp.md), and the internet's routing sends each user to a nearby site. Failover happens when a site withdraws its route, with no DNS cache to wait for. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md). The catch: "nearby" means BGP's idea of nearest, not lowest latency, and a route change can move a live TCP connection to a site that does not know it. Large CDNs and DNS providers use anycast widely.

| Method | Failover speed | Control | Works at |
|---|---|---|---|
| DNS GSLB | Seconds to minutes (TTL) | Good policy, poor precision | DNS |
| Anycast | Usually fast (BGP convergence) | Limited, depends on routing | IP |
| HTTP redirect | Immediate per request | Very precise | HTTP, adds a round trip |

### Consistent hashing

Simple hashing has a problem. Suppose you pick a server like this:

```text
server = hash(key) mod number_of_servers
```

`mod` means the remainder after division. Here are ten keys with hash values, first with 3 servers, then with 4:

| Hash | mod 3 | mod 4 | Moved? |
|---|---|---|---|
| 4 | 1 | 0 | Yes |
| 7 | 1 | 3 | Yes |
| 12 | 0 | 0 | No |
| 15 | 0 | 3 | Yes |
| 22 | 1 | 2 | Yes |
| 31 | 1 | 3 | Yes |
| 36 | 0 | 0 | No |
| 41 | 2 | 1 | Yes |
| 58 | 1 | 2 | Yes |
| 63 | 0 | 3 | Yes |

Adding one server moved 8 of 10 keys. Going from 3 to 4 servers moves 75% of keys on average. For a cache, that means a sudden flood of misses to the origin or database. For sticky sessions, it means most users lose their server.

**Consistent hashing** fixes this. When a server is added or removed, only about 1/N of keys move, where N is the number of servers.

**Analogy.** Picture a clock face. Servers stand at certain times. Each key lands on a time, then walks clockwise until it meets a server. If a new server arrives, it takes only the keys from a short stretch of the clock. The analogy stops being true because one server needs many positions in a real system, not one.

The steps are:

1. Picture a ring of hash values, here 0-99 to keep it simple. Real systems use a range such as 0 to 2^32 - 1.
2. Hash each server to a position on the ring.
3. Hash each key to a position on the ring.
4. Walk clockwise from the key to the first server. That is its owner. Past the top, wrap to 0.

Put A at 20, B at 50 and C at 80:

```text
                0
           90       10
                         A (20)
       C (80)                30

       70                    40

           60       B (50)
```

| Key | Owner with A, B, C | After adding D at 35 |
|---|---|---|
| 4, 7, 12, 15 | A | A |
| 22, 31 | B | **D** |
| 36, 41 | B | B |
| 58, 63 | C | C |

Only 2 of 10 keys moved, and they moved only to the new server. Compare that with 8 of 10 before.

If B is removed instead, its keys go to the next server clockwise, C. Keys on A and C do not move.

**Virtual nodes.** With one position per server, the ring is uneven. One server might own 60% of it. The fix is to give each server many positions, often 100 or more. Keys spread evenly, and when a server dies, its load spreads across all the others rather than falling on one neighbour. A stronger server gets more positions, which acts as a weight.

**Where it is used:** cache clusters, URL hash load balancing, databases such as Apache Cassandra, and L4 load balancer fleets.

**Fuller truth.** The ring is not the only design:

- **Rendezvous hashing** (highest random weight): score each server against the key, and pick the highest score. It needs no ring.
- **Maglev hashing** (Google): builds a fixed lookup table, which gives a very even spread and fast lookups. A pool change moves slightly more keys than the ideal.
- **Bounded-load consistent hashing:** caps how much more than the average one server may carry, so one popular key cannot overload a server. HAProxy exposes this as `hash-balance-factor`.

## Example

### Example 1: HAProxy with TLS, routing, stickiness and health checks

```text
                           VIP 203.0.113.10
                                  |
                         +-----------------+
Client 198.51.100.77 --> | HAProxy         |
                         | inside 10.0.1.5 |
                         +-----------------+
                           |             |
                 path = other        path = /api/
                           |             |
        +------------------+             +-----------+
        |  be_web (cookie, roundrobin)   be_api      |
        |                                (leastconn) |
   web1 10.0.1.11:8080                 api1 10.0.2.21:9000
   web2 10.0.1.12:8080                 api2 10.0.2.22:9000
   web3 10.0.1.13:8080
```

Config for HAProxy on Linux, `/etc/haproxy/haproxy.cfg`. It assumes the syntax of HAProxy 2.2 or later:

```text
global
    log /dev/log local0
    maxconn 50000
    stats socket /run/haproxy/admin.sock mode 660 level admin

defaults
    mode http
    log global
    option httplog
    option forwardfor
    option redispatch
    retries 3
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend fe_www
    bind 203.0.113.10:80
    bind 203.0.113.10:443 ssl crt /etc/haproxy/certs/www.pem alpn h2,http/1.1
    http-request redirect scheme https code 301 unless { ssl_fc }
    http-request set-header X-Forwarded-Proto https
    acl is_api path_beg /api/
    use_backend be_api if is_api
    default_backend be_web

backend be_web
    balance roundrobin
    cookie SRV insert indirect nocache httponly secure
    option httpchk
    http-check send meth GET uri /healthz ver HTTP/1.1 hdr Host www.example.com
    http-check expect status 200
    default-server inter 2s fall 3 rise 2 slowstart 30s
    server web1 10.0.1.11:8080 check cookie web1
    server web2 10.0.1.12:8080 check cookie web2
    server web3 10.0.1.13:8080 check cookie web3

backend be_api
    balance leastconn
    option httpchk
    http-check send meth GET uri /healthz ver HTTP/1.1 hdr Host www.example.com
    http-check expect status 200
    default-server inter 2s fall 3 rise 2
    server api1 10.0.2.21:9000 check maxconn 500
    server api2 10.0.2.22:9000 check maxconn 500
```

What the main lines do:

- `ssl crt` terminates TLS. The `.pem` file holds the certificate chain and private key.
- `option forwardfor` adds the `X-Forwarded-For` header with the client IP.
- `option redispatch` lets a request go to another server if the sticky server is down.
- `cookie SRV insert` makes HAProxy add the cookie. `indirect` removes it before the backend sees it.
- `path_beg /api/` is the L7 routing rule.
- `maxconn 500` limits each API server. Extra requests wait in a queue at HAProxy, not at the server.

Check the config, then load it:

```bash
# Linux
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

```text
Configuration file is valid
```

Test the balancing. Each test request comes with no cookie, and we assume a `/whoami` page that prints the server name:

```bash
# Linux
for i in 1 2 3 4 5 6; do curl -s https://www.example.com/whoami; done
```

```powershell
# Windows
1..6 | ForEach-Object { curl.exe -s https://www.example.com/whoami }
```

```text
web1
web2
web3
web1
web2
web3
```

Other traffic can change the order. Now look at the cookie and test stickiness:

```bash
# Linux and Windows (use curl.exe on Windows)
curl -s -D - -o /dev/null https://www.example.com/whoami
curl -s -b "SRV=web2" https://www.example.com/whoami
curl -s -b "SRV=web2" https://www.example.com/whoami
```

```text
HTTP/2 200
content-type: text/plain
set-cookie: SRV=web1; path=/; HttpOnly; Secure

web2
web2
```

Next, suppose the app on web2 starts returning 503 on `/healthz`. After three failed checks, about 6 seconds, HAProxy logs one long line like this:

```text
Server be_web/web2 is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 2ms. 2 active and 0 backup servers left. 3 sessions active, 0 requeued, 0 remaining in queue.
```

Users with `SRV=web2` now go to web1 or web3, because of `option redispatch`.

To drain web2 for planned work, use the runtime API:

```bash
# Linux
echo "set server be_web/web2 state drain" | \
  sudo socat stdio /run/haproxy/admin.sock

echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | \
  cut -d, -f1,2,5,18 | grep -E 'pxname|be_web'
```

```text
# pxname,svname,scur,status
be_web,web1,18,UP
be_web,web2,3,DRAIN
be_web,web3,21,UP
be_web,BACKEND,42,UP
```

Here `scur` is the current number of sessions. Wait until web2 reaches 0, do the work, then return it:

```bash
# Linux
echo "set server be_web/web2 state ready" | \
  sudo socat stdio /run/haproxy/admin.sock
```

For a cache pool, consistent hashing needs only two lines:

```text
backend be_cache
    balance uri
    hash-type consistent
    server cache1 10.0.3.11:8080 check
    server cache2 10.0.3.12:8080 check
    server cache3 10.0.3.13:8080 check
```

Without `hash-type consistent`, HAProxy uses its default map-based hashing, and a pool change moves most keys.

### Example 2: the same site in NGINX

Config for NGINX on Linux, inside the `http` block:

```text
upstream be_web {
    least_conn;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.13:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.14:8080 backup;
    keepalive 32;
}

upstream be_api {
    hash $cookie_sessionid consistent;
    server 10.0.2.21:9000;
    server 10.0.2.22:9000;
}

server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name www.example.com;

    ssl_certificate     /etc/nginx/certs/www.crt;
    ssl_certificate_key /etc/nginx/certs/www.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 5s;
    proxy_read_timeout 30s;

    location /api/ {
        proxy_pass http://be_api;
    }

    location / {
        proxy_pass http://be_web;
        proxy_next_upstream error timeout http_502 http_503;
    }
}
```

What differs from HAProxy:

- `max_fails=3 fail_timeout=30s`: three failed real requests within 30 seconds mark the server unavailable for 30 seconds. This is a passive check.
- `backup`: used only when all main servers are unavailable.
- `keepalive 32`: keeps up to 32 idle connections to the upstream per worker. It needs `proxy_http_version 1.1` and an empty `Connection` header.
- `hash ... consistent`: consistent hashing, here on a cookie named `sessionid`.
- `proxy_next_upstream`: retries on another server. By default, NGINX does not retry non-idempotent requests such as POST once they have been sent.
- `http2 on`: this directive has been the way to turn on HTTP/2 since NGINX 1.25.1.

Open source NGINX has `ip_hash` and `hash` for persistence. The `sticky` cookie directive has been a commercial NGINX Plus feature, so check the documents for your version.

Check the config and reload:

```bash
# Linux
sudo nginx -t
sudo nginx -s reload
```

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

To drain in open source NGINX, add `down` to the server line and reload. New workers skip that server, and old workers finish their current requests before they exit.

### Example 3: L4 load balancing for PostgreSQL

Neither config below reads the database protocol. They balance by connection only.

HAProxy on Linux:

```text
frontend fe_pg
    mode tcp
    bind 10.0.3.5:5432
    default_backend be_pg

backend be_pg
    mode tcp
    balance leastconn
    timeout server 30m
    server db1 10.0.3.31:5432 check
    server db2 10.0.3.32:5432 check
```

NGINX on Linux, in the top-level `stream` block (outside `http`):

```text
stream {
    upstream be_pg {
        least_conn;
        server 10.0.3.31:5432;
        server 10.0.3.32:5432;
    }
    server {
        listen 10.0.3.5:5432;
        proxy_pass be_pg;
        proxy_timeout 30m;
    }
}
```

`check` in `mode tcp` is a plain TCP connect. It proves only that the port is open, not that this replica is healthy or in sync.

### Example 4: DSR with Linux IPVS

IPVS is the L4 balancer built into the Linux kernel. `ipvsadm` is its control tool.

On the load balancer:

```bash
# Linux, load balancer
sudo ip addr add 203.0.113.10/32 dev eth0
sudo ipvsadm -A -t 203.0.113.10:80 -s wlc
sudo ipvsadm -a -t 203.0.113.10:80 -r 10.0.1.11:80 -g -w 1
sudo ipvsadm -a -t 203.0.113.10:80 -r 10.0.1.12:80 -g -w 1
```

`-s wlc` means weighted least connections. `-g` means gatewaying, which is DSR by MAC rewrite.

On each backend:

```bash
# Linux, each backend
sudo ip addr add 203.0.113.10/32 dev lo
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1
sudo sysctl -w net.ipv4.conf.all.arp_announce=2
```

`arp_ignore=1` stops the backend answering ARP for an address that is not on the incoming interface. `arp_announce=2` makes it use its best local address in ARP requests. Check the result on the load balancer:

```bash
sudo ipvsadm -Ln
```

```text
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  203.0.113.10:80 wlc
  -> 10.0.1.11:80                 Route   1      12         3
  -> 10.0.1.12:80                 Route   1      11         5
```

`Route` confirms direct routing mode. IPVS does no health checking by itself, so it is normally paired with a tool such as keepalived.

### Example 5: GSLB answers from two sites

The same name gives different answers by client location:

```bash
# Linux, from a host in Germany
dig +noall +answer www.example.com @192.0.2.53
```

```text
www.example.com.    30    IN    A    203.0.113.10
```

```bash
# Linux, from a host in Singapore
dig +noall +answer www.example.com @192.0.2.53
```

```text
www.example.com.    30    IN    A    203.0.113.20
```

If Frankfurt fails its health checks, the Germany query starts returning `203.0.113.20`. Clients with the old answer keep trying Frankfurt until the 30-second TTL expires.

```powershell
# Windows equivalent
Resolve-DnsName www.example.com -Server 192.0.2.53 -Type A
```

## Going Deeper

### Keeping the real client IP

A full proxy, or SNAT, hides the client's IP from the backend. Logs show only the load balancer, and rate limits break. There are three main fixes:

- **`X-Forwarded-For` header.** A common convention at L7. The standard form is the `Forwarded` header (RFC 7239), but `X-Forwarded-For` is still more common. The header can be faked by clients, so backends should trust only values added by proxies they know.
- **PROXY protocol.** Created by the HAProxy project. The proxy sends a short line at the start of the TCP connection, before any application data. Version 1 is text, version 2 is binary. Both sides must agree to use it, or the connection breaks. It works for non-HTTP traffic too.

```text
PROXY TCP4 198.51.100.77 203.0.113.10 51000 443
```

- **DSR or transparent modes.** The original source IP stays in the packet.

In HAProxy, add `send-proxy` to a `server` line and `accept-proxy` to the receiving `bind` line. In NGINX, use `listen 443 ssl proxy_protocol;` to receive it, and `proxy_protocol on;` in `stream` to send it.

### Port exhaustion

A full proxy opens its own connection to each backend. A TCP connection is unique by source IP, source port, destination IP and destination port. From one load balancer IP to one backend IP and port, the only thing that can vary is the source port.

Linux's default ephemeral port range is 32768-60999, which is about 28,000 ports. Ports in TIME_WAIT also count for a while. Under heavy load, new connections fail even when the CPU is fine.

Fixes: reuse backend connections with keep-alive, add source IPs to the load balancer, add backend IPs or ports, and widen the port range.

### Keep-alive and the idle timeout race

Load balancers often reuse idle connections to backends. If the backend closes an idle connection at the same moment the load balancer sends a request on it, the user gets a 502 or a reset. The fix is to make the backend's keep-alive timeout longer than the load balancer's idle timeout, so that the load balancer is the one that closes the idle connection.

### HTTP/2, gRPC and long-lived connections

HTTP/2 carries many requests on one TCP connection. An L4 load balancer sees one connection and sends it all to one backend. If each gRPC client opens one connection and holds it for hours, the load stays uneven, even as new servers are added.

The fixes are an L7 load balancer that balances per request, client-side load balancing, or a maximum connection age that forces clients to reconnect. See [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

### QUIC and HTTP/3

[QUIC](../06-transport-layer/08-quic.md) runs over UDP, and it can move between networks by using a connection ID. A phone that goes from Wi-Fi to mobile data keeps the connection but changes its IP and port. A balancer that hashes on the 5-tuple will then send it to the wrong backend. A load balancer must either terminate QUIC, or route using the connection ID. The IETF QUIC group has worked on a design called QUIC-LB for this.

### Retries and idempotency

An L7 load balancer can retry a failed request on another server. This hides failures, but it is only safe for **idempotent** requests, where running twice has the same result as running once. A GET is safe. A POST that charges a card may not be. Retries can also multiply load during an outage, so limit them.

### Slow start and the thundering herd

A newly started server has empty caches and may be slow. With least connections, it has 0 connections, so it gets a sudden rush and may fail again. **Slow start** raises its share gradually. In HAProxy, `slowstart 30s` does this.

### Outlier detection

A health check tests one URL. A server may pass it and still fail 30% of real requests. **Outlier detection** watches real results and ejects a server that behaves worse than its peers for a while. Envoy has this built in. It is an advanced form of passive checking.

### Who balances the load balancer?

A single load balancer is a single point of failure. There are two common designs, with details in [High Availability and Redundancy](07-high-availability.md):

- **Active-passive.** Two nodes share a VIP using VRRP, often with keepalived. See [First Hop Redundancy](../05-routing/11-first-hop-redundancy.md). Failover is simple, but half of the capacity sits idle.
- **Active-active with ECMP.** Several load balancers announce the same VIP to the routers with BGP. ECMP (equal-cost multi-path) splits flows among them by hashing. See [Advanced Routing Techniques](../05-routing/12-advanced-routing-techniques.md).

Large systems stack the layers:

```text
Internet
   |
Routers (ECMP, stateless flow hashing)
   |
L4 tier (Maglev, Katran, IPVS): consistent hash, DSR
   |
L7 tier (HAProxy, NGINX, Envoy): TLS, routing, retries
   |
Application servers
```

If an L4 node dies, ECMP sends its flows to other nodes. Those nodes hold no state for the flows, but consistent hashing makes them choose the same backend, so most connections survive. Google Maglev (described in a 2016 paper) and Meta's Katran (open source, built on XDP and eBPF) use this idea.

### Where the load balancer lives

| Type | Examples | Note |
|---|---|---|
| Hardware appliance | F5 BIG-IP | Strong L4-L7 features, higher cost |
| Software | HAProxy, NGINX, Envoy, IPVS | Runs on normal servers and VMs |
| Cloud | AWS ALB (L7) and NLB (L4), Azure Load Balancer and Application Gateway, Google Cloud Load Balancing | Managed, see [Azure and GCP Networking Compared](../11-cloud-and-containers/05-azure-and-gcp-networking.md) |
| Container | Kubernetes Service, Ingress | See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md) |
| Client-side | gRPC client load balancing, service mesh sidecars | No central box, see [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md) |

A load balancer differs from a general [reverse proxy](05-proxies-and-reverse-proxies.md) mainly in purpose. Most L7 load balancers are reverse proxies that focus on spreading work. [Link aggregation](../03-ethernet-and-switching/10-link-aggregation.md) is a different thing again: it shares traffic across cables, not across servers.

**DNS round robin**, which returns several A records, is the simplest form of sharing. It has no real health checking, and a dead server's address can stay in caches, so it does not replace a load balancer.

### Numbers to watch

Monitor these per backend: active connections, request rate, 5xx rate, p99 latency (the time that 99% of requests beat), queue length, health status changes, and TLS handshake rate. On the load balancer itself, watch CPU, connection table use and source port use.

## Common Mistakes

- **Thinking a load balancer gives full availability on its own.** It is itself a single point of failure unless you deploy it in a redundant pair or cluster.
- **Thinking L4 means "not a proxy".** HAProxy in `mode tcp` is an L4-deciding full proxy with two TCP connections.
- **Assuming `least connections` equals "least loaded".** One connection may be idle and another may be running a heavy report.
- **Using source IP hash for public users.** A whole company or mobile carrier may share one IP through NAT, so one backend gets far too much traffic.
- **Using a TCP check for a web app.** The port can be open while the app returns 500 on every request.
- **Putting the database in the health check.** One slow dependency can then remove every server at once.
- **Treating `mod N` hashing as consistent hashing.** A pool change moves most keys.
- **Forgetting that TLS blocks L7 features.** With passthrough, the load balancer cannot read cookies or paths.
- **Terminating TLS, then assuming that the app knows about HTTPS.** Without `X-Forwarded-Proto`, apps may create bad redirects or loops.
- **Assuming DNS failover is instant.** TTLs, resolvers and clients can all delay it.
- **Using DSR and expecting L7 features.** The load balancer does not see the replies.
- **Killing a server without draining it.** In-flight requests get errors, even if the failure is detected within seconds.
- **Setting one timeout for everything.** A WebSocket and a short API call need different values. A low `timeout server` may cut off long requests.
- **Trusting `X-Forwarded-For` from any source.** Clients can send a fake value.

## Interview Questions

**Q: What is the difference between L4 and L7 load balancing?**

A: L4 decides using IP addresses, ports and protocol, and it does not look inside the data. It works for any TCP or UDP service and costs less per connection. L7 reads the application request, such as the HTTP path, host and cookies. It can route by URL, keep sessions with cookies, and retry requests, but it must terminate the connection and decrypt TLS, so it does more work.

**Q: Which algorithm would you use for WebSockets, and why?**

A: Least connections, or power of two choices. WebSockets stay open for a long time, so round robin can leave some servers with many more live connections than others. Least connections sends new sockets to the server holding the fewest.

**Q: How do health checks work, and what happens when one fails?**

A: The load balancer tests each backend at a set interval, for example an HTTP GET to `/healthz` every 2 seconds. After a set number of failures in a row, such as 3, it takes the server out of rotation. After a set number of successes it adds the server back. Using separate thresholds prevents flapping. Passive checks add a second signal by watching real traffic errors.

**Q: What is session persistence, and what are its downsides?**

A: It keeps a user on the same backend across requests, usually with a load balancer cookie or a source IP hash. Downsides are uneven load, slower draining, and lost sessions when a server dies. Source IP also breaks behind NAT. I prefer stateless application servers with sessions in a shared store such as Redis.

**Q: Compare SSL termination, bridging and passthrough.**

A: Termination decrypts at the load balancer and sends plain HTTP to backends, so it is simple and saves backend CPU, but the inner traffic is unencrypted. Bridging decrypts, then opens a new TLS connection to the backend, so you keep L7 features and internal encryption at some CPU cost. Passthrough never decrypts, so it is end-to-end encrypted, but the load balancer can use only L4 data and SNI.

**Q: What is DSR, and when would you use it?**

A: Direct Server Return means requests pass through the load balancer, but backends reply straight to the client. The load balancer rewrites only the destination MAC in L2 DSR, and backends hold the VIP on loopback and do not answer ARP for it. It suits traffic with large responses, such as video, because the load balancer carries only the small request side. You lose L7 features, TLS termination and response visibility.

**Q: How do you deploy a new version with zero downtime behind a load balancer?**

A: I drain one server so that it gets no new connections, and I wait for existing ones to finish or for a timeout. Then I stop it, deploy, and start it. It rejoins only after it passes health checks, ideally with slow start. I repeat this in batches, and I keep enough spare capacity to handle traffic with those servers out. Sticky sessions and long-lived connections need extra care.

**Q: What is GSLB, and how does it differ from a local load balancer?**

A: A local load balancer shares traffic between servers in one site, and it stays in the data path. GSLB chooses between sites, usually through DNS answers based on health and location, and then it leaves the path. Its limit is DNS caching, so failover is at least as slow as the TTL. Anycast is another global method, where BGP directs users to a nearby site.

**Q: Explain consistent hashing and why it is better than hash mod N.**

A: With hash mod N, changing N moves most keys. Going from 3 to 4 servers moves about 75%. Consistent hashing puts servers and keys on a ring, and each key belongs to the next server clockwise. Adding or removing a server moves only about 1/N of the keys. Virtual nodes give each server many ring positions, which evens out the load.

**Q: A backend log shows only the load balancer's IP. How do you fix it?**

A: That is expected with a full proxy or SNAT. For HTTP, I add `X-Forwarded-For` at the load balancer, and I configure the app or web server to trust it only from the load balancer's addresses. For non-HTTP TCP, I use the PROXY protocol, with both sides configured. Another option is a forwarding mode that preserves the source IP, such as DSR.

**Q: Users get random 502 errors, but all backends pass health checks. What do you check?**

A: First, the load balancer logs, to see which backend and which error type it reports. A common cause is the keep-alive race, where the backend closes an idle connection just as the load balancer reuses it. The fix is a backend idle timeout longer than the load balancer's. I also check for backend crashes, response timeouts, oversized headers and source port exhaustion.

**Q: What happens if your load balancer fails? How do you remove it as a single point of failure?**

A: Without redundancy, the whole service goes down, even if all backends are healthy. I run an active-passive pair sharing a VIP with VRRP, or several active nodes announcing the VIP with BGP and ECMP. For stateful failover, I share connection or stick-table state. With consistent hashing, stateless L4 nodes can usually pick the same backend for existing flows.

## Quick Recap

- A load balancer gives one VIP for many backends, which adds scale, availability and safe maintenance.
- L4 uses IPs and ports. L7 reads the request. "L4" can mean packet forwarding or a full proxy.
- Pick an algorithm by traffic type: round robin for equal short requests, least connections for long ones, hashing for caches.
- Use health checks with rise and fall thresholds, and keep them from depending too much on shared systems.
- Prefer stateless apps over stickiness. Termination gives L7 features, and passthrough keeps TLS end to end.
- DSR removes reply traffic from the load balancer at the cost of L7 features. Draining lets work finish before removal.
- GSLB chooses a site, usually by DNS or anycast. Consistent hashing moves only about 1/N of keys when the pool changes.
