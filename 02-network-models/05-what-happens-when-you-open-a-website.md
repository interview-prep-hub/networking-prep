# What Happens When You Open a Website

**Level:** Intermediate | **Prerequisites:** [The TCP/IP Model](03-tcp-ip-model.md), [Encapsulation and Decapsulation](04-encapsulation.md)

## In One Line

Your browser finds the server's address, builds a secure connection to it, asks for the page, and draws what comes back. Dozens of systems help along the way, usually in less than a second.

## Why It Matters

This is the most famous question in networking interviews. It is usually worded like this: "What happens when you type a URL into a browser and press Enter?"

Interviewers love it for three reasons:

- It touches almost every layer: Ethernet, IP, TCP, TLS, DNS and HTTP.
- It has no fixed depth. A beginner can answer in one minute. An expert can talk for an hour.
- The interviewer can stop you at any step and dig deeper. That shows what you truly understand.

It also solves a real work problem. When a website is broken or slow, you need to know which step failed. An engineer who knows this path can say "DNS works, TCP works, TLS fails" and go straight to the cause.

## Simple Explanation

Imagine you want to buy a book from a big bookshop, and you live in a large apartment building.

| Real life | Network |
|---|---|
| You write down the shop name and the book title | The URL |
| You check your own shelf first | Browser and OS cache |
| You look up the shop's street address in a directory | DNS |
| You find the mail room in your building | ARP for the default gateway |
| You call the shop: "Can you hear me?" "Yes, can you hear me?" "Yes." | TCP handshake |
| You check the shop's ID and agree on a secret code | TLS handshake |
| You ask for the book | HTTP request |
| The mail room puts the building address on your letter | NAT |
| Post offices pass the letter from town to town | Routers |
| A local branch of the shop has the book, so the main shop is not needed | CDN |
| The front desk sends you to a free staff member | Load balancer |
| The book arrives and you read it | Response and rendering |
| You keep the phone line open for your next question | Connection reuse |

**Where the analogy stops being true:** a letter is one object, but a web page is split into many small packets, and each one is routed on its own. A page also needs many separate fetches, often from several servers, all at the same time. And the whole thing takes milliseconds, not days.

## How It Works

### The Example Network

Every step below uses the same setup, so you can follow one set of numbers.

```text
Laptop            Home router       ISP and transit      CDN edge
192.168.1.50      LAN 192.168.1.1   routers              203.0.113.80
                  WAN 198.51.100.10                           |
   |-----Wi-Fi-------|-----link-----[R]---[R]---[R]-----------|
                                                              |
                                              only on a cache miss
                                                              |
                                                              v
                                              Load balancer 192.0.2.10
                                                  /            \
                                            10.0.1.11       10.0.1.12
                                              (app servers)
```

| Item | Value |
|---|---|
| URL typed | `https://www.example.com/products?id=42#reviews` |
| Laptop IP and mask | 192.168.1.50/24 |
| Laptop MAC | aa:bb:cc:11:22:33 |
| Default gateway | 192.168.1.1, MAC 00:11:22:33:44:55 |
| Home router public IP | 198.51.100.10 |
| DNS server given to the laptop | 192.168.1.1 (the router forwards queries) |
| ISP recursive resolver | 198.51.100.53 |
| CDN edge server | 203.0.113.80 |
| Origin load balancer | 192.0.2.10 |
| Round-trip time to the edge | 20 ms |

The laptop already has its IP address, mask, gateway and DNS server. It got them from [DHCP](../07-application-protocols/04-dhcp.md) when it joined the Wi-Fi.

### The 60 Second Answer

Learn this version first. Say it out loud until it flows.

> I type the URL. The browser parses it into scheme, host, port and path, and checks whether it already has a fresh copy in its cache. If not, it needs the server's IP address. It checks the browser and OS DNS caches, then asks a recursive resolver, which can walk the root, TLD and authoritative servers. The destination is outside my subnet, so my machine sends to the default gateway, and uses ARP to learn the gateway's MAC address. Then comes the TCP three-way handshake, followed by the TLS handshake, which verifies the certificate and agrees on encryption keys. The browser sends an HTTP GET. Along the way, my router does NAT, and each router forwards the packet hop by hop. MAC addresses change at each hop, but the IP addresses stay the same after NAT. The request usually lands on a CDN edge or a load balancer, which answers from cache or passes it to a backend server. The response comes back, the browser parses the HTML, fetches CSS, JavaScript and images, often over the same connection, and renders the page.

After you say this, stop and ask: "Which part would you like me to go deeper on?" That shows confidence and lets the interviewer steer.

### The Full Deep Answer

| Step | What happens | Main protocol | TCP/IP layer |
|---|---|---|---|
| 1 | Parse the URL | None (browser logic) | Application |
| 2 | Check caches | HTTP caching rules | Application |
| 3 | Find the IP address | DNS | Application |
| 4 | Find the next hop | ARP | Link |
| 5 | Open a connection | TCP | Transport |
| 6 | Secure it | TLS | Between transport and application |
| 7 | Ask for the page | HTTP | Application |
| 8 | Cross the networks | IP, NAT, BGP | Internet |
| 9 | Reach the edge | CDN | Application and internet |
| 10 | Reach a server | Load balancing | Transport or application |
| 11 | Get the response | HTTP over TLS over TCP | All |
| 12 | Draw the page | HTML, CSS, JavaScript | Above the network |
| 13 | Reuse or close | HTTP, TLS, TCP | Application and transport |

### Step 1: Parse the URL

A URL (Uniform Resource Locator) is the full address of something on the web. Its syntax comes from RFC 3986.

```text
https://www.example.com:443/products?id=42#reviews
\___/   \_____________/\__/\_______/\____/\______/
scheme       host      port  path   query  fragment
```

| Part | Value | Meaning |
|---|---|---|
| Scheme | https | The protocol to use |
| Host | www.example.com | The name of the server |
| Port | 443 | The door on the server. Not written, so the default for https is used |
| Path | /products | Which resource |
| Query | id=42 | Extra input for the resource |
| Fragment | reviews | A place inside the page |

Three small things to know:

- **The address bar first decides what you typed.** `example.com` looks like a site. `best pizza` looks like a search. A search goes to your search engine, so the URL that gets opened is the search engine's.
- **The fragment never goes to the server.** The browser keeps `#reviews` and uses it to scroll the page after it loads.
- **Non-English names are converted.** A name in another script is turned into an ASCII form called Punycode, because DNS names are normally sent in ASCII.

The browser then applies its security rules. HSTS (HTTP Strict Transport Security, RFC 6797) is a rule a site can set: "only ever talk to me over HTTPS." If the site is on the browser's HSTS list, then `http://` is changed to `https://` before any packet leaves.

### Step 2: Check the Caches

A cache is a nearby saved copy, kept so that you do not fetch the same thing twice. The fastest network request is the one you never send.

The browser checks these, in rough order:

1. **Service worker.** This is a small script a site can install in your browser. It can catch requests and answer them itself.
2. **Memory cache and disk cache.** Copies of earlier responses.

Each cached response has a freshness rule from the server, such as `Cache-Control: max-age=300` (keep it for 300 seconds). There are three outcomes:

| Cache state | What the browser does | Network used |
|---|---|---|
| Fresh copy exists | Uses it | None |
| Old copy exists | Asks "has it changed?" with `If-None-Match` | Small request. A `304 Not Modified` reply means reuse the copy |
| No copy | Full fetch | Full request |

If a fresh copy exists, the story ends here for that file. The rest of this chapter assumes a first visit, with nothing cached. HTML pages are often marked as short-lived or not cacheable, so this path is common.

The details live in [HTTP Basics](../07-application-protocols/05-http-basics.md).

### Step 3: DNS Resolution

The browser has a name, `www.example.com`, but IP packets need a number. DNS (Domain Name System) turns names into IP addresses.

**Fuller truth:** a modern browser usually asks for three record types at the same time: A (IPv4), AAAA (IPv6) and HTTPS (RFC 9460). The HTTPS record can tell the browser that the site supports HTTP/3, before it connects.

The lookup tries the closest place first:

1. **Browser DNS cache.** Kept for a short time.
2. **OS cache and hosts file.** The hosts file is a local list of names and IPs. On Linux it is `/etc/hosts`. On Windows it is `C:\Windows\System32\drivers\etc\hosts`.
3. **The configured DNS server.** The small DNS client inside your OS is called a stub resolver. It sends one question and waits for a final answer.
4. **The recursive resolver.** This server does the real work for you. If it has no cached answer, it asks a chain of servers.

```text
                         a root server
                        ^  "ask .com"
                        |  |
                     1  |  v 2
Laptop --> Home --> Recursive ---3---> .com TLD server
(stub)     router   resolver   <---4--- "ask ns1.example.com"
           (forward) 198.51.100.53
              ^         |  ^
              |      5  |  | 6
              |         v  |
              |      Authoritative server for example.com
              |      "www is a CNAME for www.cdn.example.com"
              |
              +--- final answer: 203.0.113.80 (TTL 60)
```

Here, the terms are:

- **Root server:** the top of the DNS tree. It knows who runs each top-level domain.
- **TLD server:** the server for a top-level domain, such as `.com`.
- **Authoritative server:** the server that holds the real records for a domain.
- **CNAME:** a record that says "this name is another name for that name."
- **TTL (Time To Live):** in DNS, how many seconds an answer may stay in a cache.

In the example, `www.example.com` is a CNAME that points to a name run by the CDN. In real life, that name usually sits in the CDN company's own domain. This is how a website hands its traffic to a CDN without changing the URL. So step 3 needs one more lookup, this time at the CDN's authoritative servers, before the final A record arrives. The diagram leaves that lookup out to stay readable.

DNS normally uses UDP port 53. It can use TCP port 53, for example when an answer is too large for UDP. Many browsers can also send DNS inside HTTPS (DoH, RFC 8484). In that case, the browser may skip the OS resolver path.

Go deeper in [DNS Basics](../07-application-protocols/01-dns-basics.md) and [DNS Advanced](../07-application-protocols/03-dns-advanced.md).

### Step 4: ARP for the Gateway

The laptop now has a target, 203.0.113.80. It must decide where to send the first frame.

First, it compares the target with its own subnet. A subnet is the group of addresses reachable directly, without a router.

```text
Laptop:  192.168.1.50  /24  -> network 192.168.1.0
Target:  203.0.113.80  /24  -> network 203.0.113.0
Different network, so send to the default gateway, 192.168.1.1
```

The [default gateway](../04-ip-addressing/08-default-gateway.md) is the router that leads out of your subnet.

An Ethernet or Wi-Fi frame needs a MAC address, which is the hardware address of a network card. The laptop knows the gateway's IP but may not know its MAC. [ARP](../03-ethernet-and-switching/05-arp.md) (Address Resolution Protocol) fills this gap:

```text
Laptop -> broadcast: "Who has 192.168.1.1? Tell 192.168.1.50"
Router -> laptop:    "192.168.1.1 is at 00:11:22:33:44:55"
```

The laptop saves the answer in its ARP cache for later packets. How long it stays there varies by OS.

Two key points:

- The laptop never ARPs for 203.0.113.80. ARP only works inside the local subnet. It ARPs for the gateway.
- **Real order:** the classic answer lists DNS before ARP, but the DNS query is itself a packet. If the ARP cache is empty and the DNS server is the gateway itself or sits outside the subnet, ARP for the gateway runs first, just to send the DNS query. Say this in an interview and you will stand out.

On IPv6, Neighbor Discovery (RFC 4861) does this job instead of ARP.

### Step 5: TCP Handshake

TCP (Transmission Control Protocol) gives a reliable, ordered stream of bytes on top of IP, which by itself makes no delivery promise. Before sending data, both sides must agree to talk.

```text
Laptop 192.168.1.50:52344              Edge 203.0.113.80:443
   |                                                 |
   | ---- SYN, seq=x ------------------------------> |
   | <--- SYN-ACK, seq=y, ack=x+1 ------------------ |
   | ---- ACK, ack=y+1 ----------------------------> |
   |                                                 |
   |              connection is open                 |
```

The OS picks a temporary source port, such as 52344. This is called an ephemeral port. The four values (source IP, source port, destination IP, destination port) identify this connection.

During the handshake, the two sides also share settings:

- **MSS (Maximum Segment Size):** the largest data chunk per segment. On normal Ethernet, this is 1460 bytes (1500 MTU minus 20 for IP and 20 for TCP).
- **Window scale:** allows a larger receive window.
- **SACK permitted:** lets the receiver report exactly which pieces arrived.
- **Sequence numbers:** random starting numbers, so that old or forged packets are harder to mistake for real ones.

The handshake costs one RTT. RTT (round-trip time) means the time for a packet to go and for the reply to come back. Here that is 20 ms.

More in [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md).

### Step 6: TLS Handshake

Without TLS (Transport Layer Security), anyone on the path could read and change the traffic. TLS gives three things:

- **Encryption:** others cannot read it.
- **Integrity:** others cannot change it without being noticed.
- **Authentication:** you know you reached the real server.

This is how TLS 1.3 works (RFC 8446):

```text
Laptop                                                  Edge
  | ---- ClientHello ----------------------------------> |
  |      versions, ciphers, key share, SNI, ALPN         |
  |                                                      |
  | <--- ServerHello (server key share) ---------------- |
  |      ==== encrypted from here on ====                |
  | <--- EncryptedExtensions, Certificate,               |
  |      CertificateVerify, Finished ------------------- |
  |                                                      |
  | ---- Finished + first HTTP request ----------------> |
```

New terms:

- **SNI (Server Name Indication):** the hostname, sent inside ClientHello. One IP address can host thousands of sites, so the server needs it to pick the right certificate.
- **ALPN (Application-Layer Protocol Negotiation):** the two sides agree on the HTTP version here, such as `h2` for HTTP/2. This saves one more round trip.
- **Key share:** public values for a key exchange. Both sides mix them and get the same secret, and that secret never crosses the wire.
- **Certificate:** a signed document that says "this public key belongs to www.example.com." A CA (Certificate Authority) signs it. Your OS or browser ships with a list of CAs it trusts.

The browser checks the certificate carefully:

1. The chain leads up to a trusted root CA.
2. The hostname matches a name in the certificate's SAN (Subject Alternative Name) list.
3. The current date is within the valid dates.
4. The server proves it owns the private key, using `CertificateVerify`.
5. Revocation and certificate logging rules pass. These checks vary by browser.

TLS 1.3 costs one RTT. TLS 1.2 costs two RTTs. Later visits can resume with a saved secret, which lowers the CPU work. They can even send data in the first flight (0-RTT), but that early data can be replayed by an attacker, so it should only carry safe requests.

More in [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

### Step 7: HTTP Request

The connection is ready. The browser sends its request, encrypted inside TLS. This is the readable HTTP/1.1 form:

```text
GET /products?id=42 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, br, zstd
Accept-Language: en
Cookie: session=abc123
```

- `GET` is the method: "give me this resource."
- `Host` says which site, because many sites share one IP.
- `Accept-Encoding` lists the compression formats the browser can read.
- `Cookie` carries data the site stored earlier, such as a login session.
- `#reviews` is not there. The fragment stayed in the browser.

**Fuller truth:** in this example, ALPN chose HTTP/2, so the real message is binary, not text. The same fields travel as compressed headers, with special names such as `:method`, `:path` and `:authority`. See [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

Here is the first request as it leaves the laptop, showing each layer wrapped around the next. This is [encapsulation](04-encapsulation.md) in real life.

```text
+-----------+-----------+-----------+-----------------+--------+
| Ethernet  | IPv4      | TCP       | TLS record      | FCS    |
| 14 bytes  | 20 bytes  | 20+ bytes | encrypted HTTP  | 4 bytes|
+-----------+-----------+-----------+-----------------+--------+
| dst MAC   | src IP    | src port  | HTTP GET        | error  |
|  gateway  | 192.168.  |  52344    | /products?id=42 | check  |
| src MAC   |  1.50     | dst port  |                 |        |
|  laptop   | dst IP    |  443      | (nobody on the  |        |
|           | 203.0.    |           |  path can read  |        |
|           |  113.80   |           |  this part)     |        |
+-----------+-----------+-----------+-----------------+--------+
```

**Fuller truth:** on Wi-Fi, the frame format between laptop and access point is 802.11, not Ethernet. The access point usually converts it to Ethernet for the wired side.

### Step 8: Routing and NAT Along the Way

This step is not a single moment. It happens to every packet, from the first DNS query and SYN onward.

**NAT at the home router.** The address 192.168.1.50 is private, so it cannot be used on the public internet. The router does [NAT](../04-ip-addressing/09-nat-and-pat.md) (Network Address Translation). Home routers use a kind called PAT (Port Address Translation), where many devices share one public IP and are told apart by port number.

```text
Before NAT:  192.168.1.50:52344   -> 203.0.113.80:443
After NAT:   198.51.100.10:40001  -> 203.0.113.80:443

Router NAT table
Inside (private)      Outside (public)      Remote
192.168.1.50:52344    198.51.100.10:40001   203.0.113.80:443
```

The first packet creates this entry. Replies arrive at port 40001, and the router uses the table to send them back to the laptop.

**Routing across the internet.** A router reads the destination IP, finds the longest matching prefix in its [routing table](../05-routing/02-routing-table.md), lowers the IP TTL by 1, and forwards the packet to the next hop. Here, TTL means something different from DNS TTL: it is a hop limit, which stops packets from looping forever.

Networks run by different organisations are called Autonomous Systems (AS). They share reachability with [BGP](../05-routing/09-bgp.md). In the example, the path is AS64512 (the ISP), then AS64513 (a transit network), then AS64514 (the CDN).

This table shows what changes at each hop. It is a favourite interview topic.

| Link | Source MAC | Dest MAC | Source IP:port | Dest IP:port | IP TTL |
|---|---|---|---|---|---|
| Laptop to home router | laptop | router LAN | 192.168.1.50:52344 | 203.0.113.80:443 | 64 |
| Home router to ISP R1 | router WAN | R1 | 198.51.100.10:40001 | 203.0.113.80:443 | 63 |
| R1 to R2 | R1 | R2 | 198.51.100.10:40001 | 203.0.113.80:443 | 62 |
| R2 to R3 | R2 | R3 | 198.51.100.10:40001 | 203.0.113.80:443 | 61 |

The rule to remember: **MAC addresses change at every hop. IP addresses stay the same from end to end, except where NAT changes them.**

**Fuller truth:** not every link uses MAC addresses. Some WAN links use other link types, such as PPP, and carriers may use MPLS inside their networks. The idea is the same: the link-layer header is local to one hop. The starting TTL also depends on the OS. Linux commonly uses 64 and Windows 128.

Many ISPs add a second NAT, called CGNAT (Carrier-Grade NAT), because public IPv4 addresses are scarce. Then your router's WAN address is itself not public.

### Step 9: The CDN Edge

A [CDN](../10-enterprise-and-datacenter/06-cdn.md) (Content Delivery Network) is a large set of servers placed close to users. Each one is called an edge server. The real website behind the CDN is called the origin.

The target IP, 203.0.113.80, belongs to an edge server, not to the origin. A CDN can pull users to a nearby edge in two common ways:

- **DNS steering:** the authoritative server returns a different IP based on where the query came from.
- **Anycast:** the same IP is announced from many sites, and BGP sends you to a nearby one. See [Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

The TCP and TLS handshakes in steps 5 and 6 ended at the edge. The edge decrypts the request and checks its cache.

- **Cache hit:** the edge has the page. It replies at once. The origin is never contacted.
- **Cache miss:** the edge fetches the page from the origin, stores it if allowed, and replies.

**Analogy:** the edge is a local branch of a big shop. **Where it stops being true:** a branch carries a fixed stock, while a CDN usually fills its "shelves" on demand, on the first request for each item.

The edge often keeps open connections to the origin, so a cache miss does not pay for new handshakes.

### Step 10: The Load Balancer

On a cache miss, the edge connects to the origin address, 192.0.2.10. This is a VIP (virtual IP): an address that belongs to a service, not to one machine. A [load balancer](../10-enterprise-and-datacenter/04-load-balancing.md) owns it and shares requests across healthy servers.

```text
                  +----------------------+
Edge ---------->  | Load balancer        |
                  | VIP 192.0.2.10:443   |
                  +----------+-----------+
                     /              \
          health check ok      health check ok
                   /                  \
          App 10.0.1.11:8080    App 10.0.1.12:8080
```

- **Layer 4 load balancer:** chooses a server using IP and port only. It is fast, and it does not read the HTTP request.
- **Layer 7 load balancer:** reads HTTP, so it can route `/api` and `/images` to different server groups. For HTTPS, it must end TLS to read the request.

The server may then call databases and other services. It builds the HTML and sends it back. These servers do not see the laptop's IP as the connection source. They usually learn the client IP from a header such as `X-Forwarded-For`, which trusted proxies add. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

### Step 11: The Response

The edge replies. In this example it already holds a cached copy, so the headers show a hit. Here is the reply in readable form:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
content-encoding: br
cache-control: public, max-age=300
etag: "v42"
age: 120
cache-status: ExampleCDN; hit; ttl=180
```

- `200` means success.
- `content-encoding: br` means Brotli compression.
- `cache-control` sets the freshness rule from step 2.
- `age: 120` means the edge has held this copy for 120 seconds. That leaves 180 of the 300 seconds, which matches `ttl=180`.

The way back reverses the path:

1. The edge splits the response into TCP segments of up to one MSS and encrypts it in TLS records.
2. Routers forward each packet toward 198.51.100.10.
3. The home router finds port 40001 in the NAT table, and rewrites the destination to 192.168.1.50:52344.
4. The laptop's TCP stack puts the bytes in order, and asks again for anything lost.
5. TLS decrypts and checks the data. The browser decompresses it.

**Fuller truth:** the way back may use different routers. Internet paths are often asymmetric.

TCP does not send everything at once. It starts small and grows. A common starting window is 10 segments (RFC 6928), which is about 14 KB. So the first bytes of a page matter most for speed. See [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).

### Step 12: Rendering

The network part is mostly done. Now the browser turns bytes into a page.

```text
HTML bytes --> parse --> DOM ---------+
                                      +--> render tree --> layout
CSS bytes ---> parse --> CSSOM -------+                       |
                                                              v
JavaScript can change both              composite <-- paint <-+
```

1. **Parse HTML** into the DOM (Document Object Model), a tree of the page's elements.
2. **Parse CSS** into the CSSOM, a tree of the style rules.
3. **Build the render tree**, which holds only what will be shown.
4. **Layout:** work out the size and position of each element.
5. **Paint:** fill in the pixels.
6. **Composite:** join the painted layers and show them.

While parsing, the browser finds more files to fetch: CSS, JavaScript, fonts and images. Each one repeats some of the steps above. Files on another hostname need a new DNS lookup and a new connection. A normal script tag can pause HTML parsing, so sites often use `defer` or `async`.

**One page is not one request.** A page often needs tens of requests, sometimes hundreds. After the page loads, the browser scrolls to `#reviews`.

### Step 13: Connection Reuse

Building a connection costs two RTTs, so browsers keep it open.

| HTTP version | How reuse works |
|---|---|
| HTTP/1.1 | Keep-alive is the default. One request at a time per connection, so browsers open several, commonly up to 6 per host |
| HTTP/2 | Many requests share one TCP connection at once, called multiplexing |
| HTTP/3 | Many requests share one QUIC connection, over UDP |

An idle connection does not last forever. Either side can close it. The idle time depends on the software: for example, nginx's default `keepalive_timeout` is 75 seconds. Browsers also have their own limits, which vary.

A clean close uses a TLS `close_notify` alert and then TCP FIN packets. The side that closes first enters the TIME_WAIT state for a short while. On Linux that is 60 seconds. See [TCP Connection States](../06-transport-layer/04-tcp-states.md).

The next page on the same site skips DNS, ARP, TCP and TLS, and goes straight to the HTTP request.

## Example

### See Each Step With Real Commands

You can watch most of this path yourself. All outputs below use the example network.

**DNS, Linux:**

```bash
dig +noall +answer www.example.com
```

```text
www.example.com.        300   IN  CNAME  www.cdn.example.com.
www.cdn.example.com.    60    IN  A      203.0.113.80
```

**DNS, Windows PowerShell:**

```powershell
Resolve-DnsName www.example.com -Type A
```

```text
Name                  Type   TTL   Section    NameHost
----                  ----   ---   -------    --------
www.example.com       CNAME  300   Answer     www.cdn.example.com

Name       : www.cdn.example.com
QueryType  : A
TTL        : 60
Section    : Answer
IP4Address : 203.0.113.80
```

**Route and gateway, Linux:**

```bash
ip route get 203.0.113.80
ip neigh show 192.168.1.1
```

```text
203.0.113.80 via 192.168.1.1 dev wlan0 src 192.168.1.50 uid 1000
    cache
192.168.1.1 dev wlan0 lladdr 00:11:22:33:44:55 REACHABLE
```

**Route and gateway, Windows PowerShell:**

```powershell
Find-NetRoute -RemoteIPAddress 203.0.113.80 | Select-Object -Last 1 NextHop
arp -a 192.168.1.1
```

```text
NextHop
-------
192.168.1.1

Interface: 192.168.1.50 --- 0xc
  Internet Address      Physical Address      Type
  192.168.1.1           00-11-22-33-44-55     dynamic
```

**The path, Linux (`tracert` on Windows):**

```bash
traceroute -n 203.0.113.80
```

```text
traceroute to 203.0.113.80 (203.0.113.80), 30 hops max, 60 byte packets
 1  192.168.1.1     1.1 ms   1.0 ms   1.2 ms
 2  198.51.100.1    7.9 ms   8.1 ms   8.0 ms
 3  198.51.100.33   9.8 ms   9.6 ms   9.9 ms
 4  * * *
 5  203.0.113.1     18.7 ms  18.9 ms  18.8 ms
 6  203.0.113.80    20.1 ms  19.9 ms  20.2 ms
```

Hop 4 did not reply. Many routers limit or block these replies, so stars here do not mean a fault. See [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

**NAT, Cisco IOS (if the edge router is a Cisco device):**

```text
Router# show ip nat translations
Pro Inside global        Inside local        Outside local     Outside global
tcp 198.51.100.10:40001  192.168.1.50:52344  203.0.113.80:443  203.0.113.80:443
```

**Open connection, Linux and Windows:**

```bash
ss -tn dst 203.0.113.80
```

```powershell
netstat -n | Select-String "203.0.113.80"
```

```text
Linux:
State  Recv-Q Send-Q  Local Address:Port    Peer Address:Port
ESTAB  0      0        192.168.1.50:52344   203.0.113.80:443

Windows:
  TCP    192.168.1.50:52344     203.0.113.80:443       ESTABLISHED
```

### A Packet Capture of the First Visit

This capture runs on the laptop, so it shows the private address, from before NAT. Options are trimmed and the AAAA query is left out to keep it short.

```bash
sudo tcpdump -ni wlan0 'arp or port 53 or host 203.0.113.80'
```

```text
10:15:30.100000 ARP, Request who-has 192.168.1.1 tell 192.168.1.50, length 28
10:15:30.101200 ARP, Reply 192.168.1.1 is-at 00:11:22:33:44:55, length 28
10:15:30.101500 IP 192.168.1.50.49731 > 192.168.1.1.53: 12345+ A? www.example.com. (33)
10:15:30.141500 IP 192.168.1.1.53 > 192.168.1.50.49731: 12345 2/0/0 CNAME www.cdn.example.com., A 203.0.113.80 (71)
10:15:30.142100 IP 192.168.1.50.52344 > 203.0.113.80.443: Flags [S], seq 1204567890, win 64240, length 0
10:15:30.162100 IP 203.0.113.80.443 > 192.168.1.50.52344: Flags [S.], seq 3344556677, ack 1204567891, win 65160, length 0
10:15:30.162200 IP 192.168.1.50.52344 > 203.0.113.80.443: Flags [.], ack 1, win 502, length 0
10:15:30.162900 IP 192.168.1.50.52344 > 203.0.113.80.443: Flags [P.], seq 1:518, ack 1, win 502, length 517
```

Read it from top to bottom. ARP comes first, then DNS, then the SYN, SYN-ACK and ACK. The last line is the TLS ClientHello. DNS took 40 ms. The TCP handshake took 20 ms, which is one RTT.

### Measure Each Step With curl

`curl` can report the time of each phase. All values are counted from the start, so each one includes the steps before it.

Linux:

```bash
curl -o /dev/null -s -w 'dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n' https://www.example.com/
```

Windows PowerShell (use `curl.exe`, because plain `curl` can be an alias for another command):

```powershell
curl.exe -o NUL -s -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" https://www.example.com/
```

```text
dns:0.042000 tcp:0.062000 tls:0.084000 ttfb:0.108000 total:0.131000
```

TTFB means time to first byte. Here is how to read those numbers, with a 20 ms RTT:

| Phase | Time | Running total | Why |
|---|---|---|---|
| DNS (cold, plus setup and ARP) | 42 ms | 42 ms | Recursive lookup |
| TCP handshake | 20 ms | 62 ms | 1 RTT |
| TLS 1.3 handshake | 22 ms | 84 ms | 1 RTT plus 2 ms of crypto |
| Request to first byte | 24 ms | 108 ms | 1 RTT plus 4 ms at the edge (cache hit) |
| Rest of the HTML | 23 ms | 131 ms | Remaining data |

So a first visit needs three round trips before the first byte arrives, plus DNS:

```text
Client                                         Server
  | ---- SYN ---------------------------------> |
  | <--- SYN-ACK ------------------------------ |  1 RTT
  | ---- ACK + ClientHello -------------------> |
  | <--- ServerHello ... Finished ------------- |  2 RTT
  | ---- Finished + HTTP GET -----------------> |
  | <--- HTTP response ------------------------ |  3 RTT
```

At a 200 ms RTT, those same three round trips cost 600 ms before any content arrives. This is why CDNs exist: lowering RTT helps more than raising bandwidth for loading a page.

## Going Deeper

### HTTP/3 Changes the Middle

HTTP/3 (RFC 9114) runs over [QUIC](../06-transport-layer/08-quic.md) (RFC 9000), which runs over UDP, usually port 443.

| | HTTP/1.1 or HTTP/2 | HTTP/3 |
|---|---|---|
| Transport | TCP | QUIC over UDP |
| Encryption | TLS on top of TCP | TLS 1.3 built into QUIC |
| New connection | 2 RTTs (TCP + TLS 1.3) | 1 RTT |
| To first response byte | 3 RTTs | 2 RTTs |
| One lost packet | Holds up all streams on the connection | Holds up only the affected stream |
| Changing network | Connection breaks | Can move to a new IP with connection IDs |

How does the browser know to use it? It can learn from the HTTPS DNS record, or from an `Alt-Svc` response header (RFC 7838) on an earlier visit. Browsers often race QUIC and TCP, because some networks block UDP 443. They use the one that works.

For an interview, say "TCP and TLS are replaced by one combined QUIC handshake." The rest of the path stays the same.

### The IPv6 Version

If the host has IPv6 and the name has a AAAA record:

- ARP is replaced by Neighbor Discovery.
- There is usually no NAT. The laptop's global address goes end to end.
- Routers lower the Hop Limit, which is the IPv6 name for TTL.
- The default gateway is commonly a link-local address, learned from Router Advertisements.

When both IPv4 and IPv6 work, browsers use Happy Eyeballs (RFC 8305). They try IPv6 first and start IPv4 after a short delay, then use whichever connects first. A broken IPv6 path therefore causes little harm to the user. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md).

### Redirects Add Whole Trips

If someone types `http://example.com`, this can happen:

1. Connect to port 80. Server replies `301` to `https://example.com/`.
2. Connect to port 443, with TLS. Server replies `301` to `https://www.example.com/`.
3. Fetch the real page.

Each redirect adds round trips, and each new hostname may need DNS, TCP and TLS. This is why sites use HSTS and link straight to the final URL. Browsers also try HTTPS first in many cases when you type a bare name.

**Security note:** an HSTS header only protects you after your first visit. The HSTS preload list ships inside browsers, which protects the first visit too.

### What Changes on a Second Visit

| Step | First visit | Next page, same site, soon after |
|---|---|---|
| Browser cache | Miss | Hits for CSS, JS and images, if fresh |
| DNS | Full lookup | Cached |
| ARP | May be needed | Cached |
| TCP and TLS | Full handshakes | Skipped if the connection is still open. If not, TLS can resume |
| CDN | May be a miss | More likely a hit |
| Result | Slower | Much faster |

### Who Can See What

A common follow-up question is "Does HTTPS hide everything?" It does not.

| Item | Wi-Fi owner or ISP can see? | Note |
|---|---|---|
| Server IP | Yes | Needed for routing |
| Port | Yes | |
| Hostname in DNS | Yes, for plain DNS | Hidden from the local network by DoH or DoT (RFC 7858), but the DNS provider still sees it |
| Hostname in SNI | Usually yes | ECH (Encrypted Client Hello) hides it where both sides support it |
| Certificate | No in TLS 1.3 | Sent in the clear in TLS 1.2 |
| URL path and query | No | Encrypted inside TLS |
| Cookies and page content | No | Encrypted inside TLS |
| Traffic size and timing | Yes | Can reveal patterns |

The CDN is different. It ends TLS, so it can read the full request.

### Proxies and VPNs Change the Story

- **Corporate forward proxy:** the browser connects to the proxy, and the proxy does the DNS lookup. For HTTPS, the browser sends `CONNECT www.example.com:443`, and the proxy builds a tunnel. Some companies decrypt traffic by installing their own CA on staff laptops.
- **VPN:** packets to the site go first, encrypted, to the VPN server. Your ISP sees only VPN traffic, and the site sees the VPN server's IP. See [VPNs](../09-security/07-vpn.md).

### Find the Broken Step From the Symptom

This table turns the walkthrough into a troubleshooting tool. The browser messages are from Chrome. Other browsers word them differently.

| Symptom | Likely step | First check |
|---|---|---|
| `DNS_PROBE_FINISHED_NXDOMAIN` | DNS: the name does not exist | `dig www.example.com` |
| Sites work by IP but not by name | DNS server unreachable | Query another resolver: `dig @198.51.100.53 www.example.com` |
| No internet at all, gateway ping fails | ARP, Wi-Fi or local link | `ip neigh`, `arp -a` |
| `ERR_CONNECTION_TIMED_OUT` | SYN gets no reply: routing or firewall drop | `traceroute`, firewall rules |
| `ERR_CONNECTION_REFUSED` | Reached the host, nothing listening, TCP RST sent | Server port and service |
| `NET::ERR_CERT_DATE_INVALID` | TLS: certificate expired or laptop clock wrong | Certificate dates, local time |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | TLS: hostname not in the certificate | Certificate SAN list |
| HTTP 502 or 504 | Proxy reached, backend failed or timed out | Load balancer, app logs |
| HTTP 503 | Service overloaded or no healthy backends | Health checks |
| HTTP 404 or 500 | Network is fine, application replied | App logs |
| Small pages work, large ones hang | Path MTU problem | [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md) |
| Page loads but looks broken | A sub-resource failed | Browser developer tools, Network tab |

Work from the bottom up. For a full method, see [Troubleshooting Methodology](../12-troubleshooting-and-tools/01-troubleshooting-method.md).

### Making It Faster

Interviewers for backend roles may ask how to improve load time. Link each fix to a step:

| Step | Improvement |
|---|---|
| Cache | Long `max-age` on versioned files, such as `app.3f2a.js` |
| DNS | Fewer hostnames, sensible TTLs, `dns-prefetch` hints |
| TCP and TLS | `preconnect` hints, TLS 1.3, session resumption, HTTP/3 |
| Distance | A CDN, to lower RTT |
| Transfer | Brotli compression, smaller images |
| Rendering | Inline critical CSS, `defer` on scripts |
| Server | Backend caching, faster queries |

## Common Mistakes

- **"DNS always comes before any network traffic."** The DNS query is a packet too. If the ARP cache is empty, ARP for the gateway comes first.
- **"The laptop ARPs for the website's IP."** ARP stays inside the subnet. For a remote target, the laptop ARPs for the default gateway.
- **"Routers change the IP addresses at each hop."** They replace the link-layer header. The IPs stay the same, except where NAT rewrites them. The IP TTL and header checksum do change.
- **"The IP from DNS is the web server."** It is often a CDN edge or a load balancer, and the application server may be several hops behind it.
- **"Every request goes to DNS."** Answers are cached in the browser, OS and resolver, within their TTL.
- **"TTL means one thing."** DNS TTL is a cache time in seconds. IP TTL is a hop limit.
- **"HTTPS hides which site I visit."** The server IP, and usually the SNI hostname, are still visible. The path, query, cookies and body are hidden.
- **"TLS comes before TCP."** On HTTPS over TCP, TCP comes first. TLS messages travel inside the TCP connection. Only QUIC merges them.
- **"HTTP/3 uses TCP."** It uses QUIC over UDP.
- **"The server sees my 192.168.x.x address."** The site sees your public IP after NAT. An origin behind a CDN usually sees the CDN's IP at the network level, and gets the client IP from a header.
- **"A page is one request and one response."** A page usually triggers many requests, sometimes to many hostnames.
- **"The fragment is sent to the server."** It stays in the browser.
- **"If ping works, the website must work."** Ping only tests ICMP reachability. TCP, TLS, HTTP and the app can all still fail. Some sites also block ping, so a failed ping does not prove the site is down.

## Interview Questions

**Q: What happens when you type a URL and press Enter? Give the short version.**

A: The browser parses the URL and checks its cache. It finds the IP through DNS. My machine sends to the default gateway, using ARP for the gateway's MAC. It opens a TCP connection, then does a TLS handshake. It sends an HTTP GET. NAT and routers carry the packets to a CDN edge or load balancer, which answers from cache or a backend server. The browser then renders the page and fetches the other files, reusing the connection.

**Q: Does ARP happen before or after DNS?**

A: It can be either, but usually before on a cold start. The DNS query must reach the DNS server. If that server is off-subnet or is the gateway, and the ARP cache is empty, the machine must ARP for the gateway first. If the entry is already cached, no ARP happens at all.

**Q: Which addresses change as a packet crosses the internet?**

A: The source and destination MAC addresses change on each hop, because they are local to one link. The IP addresses stay the same from end to end. The exception is NAT, which rewrites them. The home router, for example, changes 192.168.1.50:52344 to its public address and a new port. Each router also lowers the IP TTL by 1.

**Q: How many round trips before the first byte of the response?**

A: On a new connection with TCP and TLS 1.3, three: one for TCP, one for TLS, and one for the HTTP request and response. TLS 1.2 adds one more. HTTP/3 over QUIC takes two, because it merges the transport and TLS handshakes. DNS time comes on top of that if the name is not cached.

**Q: Why does the client send SNI?**

A: One IP address can serve thousands of sites, especially on a CDN. The server needs the hostname before it picks a certificate, and TLS happens before the HTTP Host header is sent. SNI puts the name in the ClientHello. It is normally visible on the wire, and ECH was created to hide it.

**Q: How does the browser know the certificate is real?**

A: It builds a chain from the server certificate up to a root CA that the OS or browser trusts, and it checks each signature. It checks that the hostname is in the SAN list and that the dates are valid. The server also signs the handshake with its private key, which proves it owns the certificate. Revocation and transparency checks vary by browser.

**Q: What does the website see as my IP address?**

A: It sees the public address after NAT, here 198.51.100.10, not my private 192.168.1.50. If the site uses a CDN, the origin sees the CDN's IP on the connection, and it learns my address from a header like `X-Forwarded-For`, which it should only trust from its own proxies.

**Q: What does a CDN change in this walkthrough?**

A: DNS returns an edge IP, often through a CNAME, instead of the origin. TCP and TLS end at the edge, which is closer, so RTT is lower. If the edge has the content, the origin is not contacted at all. If it does not, it fetches from the origin, usually over an already open connection, and may store a copy.

**Q: A site shows ERR_CONNECTION_TIMED_OUT but another shows ERR_CONNECTION_REFUSED. What is the difference?**

A: A timeout means the SYN got no answer. The packet was dropped by a firewall, was lost in routing, or the host is down. Refused means the host answered with a TCP RST, so the network path is fine, but nothing is listening on that port, or something actively rejected the connection.

**Q: What is the difference between a 304 response and a cache hit in the browser?**

A: With a fresh cache hit, no request is sent at all. With a 304, the copy was stale, so the browser asked the server whether it had changed, using a header such as `If-None-Match`. The server replied "not modified" with no body. That still costs a round trip, but it saves the download.

**Q: Why do browsers reuse connections, and how do HTTP/1.1 and HTTP/2 differ here?**

A: A new connection costs handshakes and a slow start in TCP. HTTP/1.1 keeps connections open but handles one request at a time on each, so browsers open several per host, commonly up to 6. HTTP/2 multiplexes many requests on one connection. HTTP/3 does the same over QUIC, and a lost packet affects only one stream there.

**Q: Does HTTPS hide the site I visit from my ISP?**

A: Not fully. The ISP sees the server IP, and usually the hostname from SNI. Plain DNS also shows the name. Encrypted DNS and ECH close those gaps, but the IP remains. HTTPS does hide the path, query string, cookies and content.

## Quick Recap

- The classic order is: URL parse, caches, DNS, ARP for the gateway, TCP, TLS, HTTP, routing and NAT, CDN and load balancer, response, rendering, connection reuse.
- If the ARP cache is empty, ARP usually happens before DNS, because the DNS query usually needs the gateway too.
- MAC addresses change at each hop. IPs stay the same, except at NAT.
- A new HTTPS connection costs 3 RTTs to the first response byte with TCP and TLS 1.3, and 2 RTTs with HTTP/3, plus any DNS time.
- The IP from DNS is often a CDN edge or a load balancer, not the application server.
- HTTPS hides the path, cookies and content, but not the server IP, and usually not the hostname.
- Knowing this path lets you find which step failed, and that is the real value of the question.
