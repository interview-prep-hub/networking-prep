# Content Delivery Networks

**Level:** Intermediate | **Prerequisites:** [DNS Basics](../07-application-protocols/01-dns-basics.md), [HTTP Basics](../07-application-protocols/05-http-basics.md), [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md), [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md), [Proxies and Reverse Proxies](05-proxies-and-reverse-proxies.md)

## In One Line

A Content Delivery Network (CDN) is a worldwide group of servers that sit close to users, keep copies of content, and stand in front of the main server. Pages load faster, and the main server does less work and takes fewer attacks.

## Why It Matters

Three real problems push people to use a CDN.

**Distance.** Data cannot travel faster than light. In optical fibre, light covers about 200 km per millisecond. So 1,000 km of fibre adds about 5 ms one way, or 10 ms for a round trip. Mumbai to Virginia is about 13,000 km in a straight line. Real paths are longer, so a round trip takes about 200 ms. No bigger server fixes this. The only fix is to move the content closer.

**Load.** One server in one city must carry every user. A popular image requested 2,000 times per second is sent 2,000 times per second. If 95% of those requests stop at a nearby copy, the main server sees only 100 per second.

**Attacks.** A single site has limited bandwidth. A CDN has far more, spread over many cities, so it can soak up floods of traffic that would kill one server.

Interviewers ask about CDNs because the topic joins many subjects: DNS, BGP, anycast, TCP, TLS, HTTP caching and DDoS defence. It also appears in almost every system design interview. If you can explain one request from start to finish, you show that you understand how these parts fit together.

## Simple Explanation

Think of a company that makes one product in one factory. Selling only from the factory means every customer must travel there, and the road gets jammed.

So the company opens local shops in every city.

| Real life | CDN term | Meaning |
|---|---|---|
| Factory | Origin | The real server that owns the content |
| Local shop | Edge POP | A site full of CDN servers near users |
| Shop shelf | Cache | Stored copies of content |
| Item on the shelf | Cache hit | Customer is served at once |
| Item not on the shelf | Cache miss | Shop must order from the factory |
| Expiry date | TTL | How long a copy is safe to use |
| Product recall | Purge | Remove copies before they expire |

A customer walks to the nearest shop. If the shelf has the item, they leave in a minute. If not, the shop orders one from the factory, sells it, and keeps a copy for the next customer.

**Where the analogy stops being true.** A shop sells each item once. A cache can serve the same copy millions of times, because digital copies do not run out. Also, a shop cannot make a custom order for you. A cache has the same limit: it cannot store your personal bank balance page and give it to other people. Those requests must still go to the origin.

## How It Works

### The main terms

- **Origin:** the server, or group of servers, that holds the true version of the content. Example: `origin.example.com` at 198.51.100.20. It may also be cloud storage such as an Amazon S3 bucket.
- **Edge:** the outer part of the CDN, closest to users.
- **POP (Point of Presence):** one physical site of a CDN, usually inside a data centre. It holds routers and servers. CDNs often name POPs by the nearest airport code, such as BOM (Mumbai), FRA (Frankfurt) or SIN (Singapore).
- **Cache:** a store of saved responses, so they can be reused.
- **TTL (Time To Live):** how many seconds a saved copy may be used without asking the origin.
- **RTT (Round-Trip Time):** the time for a packet to go somewhere and for the reply to come back.
- **Hit ratio:** the percentage of requests served from cache.

### Why a CDN helps: three effects

1. **Shorter distance.** The user talks to a server 5 ms away, not 200 ms away.
2. **Less origin work.** Most requests never reach the origin.
3. **Better connections.** The edge keeps long-lived, warmed-up connections to the origin, so even uncached requests can gain.

A CDN also lowers cost, because the origin sends fewer bytes. Cloud providers usually charge for data leaving their network.

### Edge POPs

A POP is more than one big server. A typical one looks like this:

```text
   Internet (transit providers, IXP peers, local ISPs)
                        |
                 +------+------+
                 | Edge routers|   BGP, filtering, ECMP
                 +------+------+
                        |
                 +------+------+
                 | L4 balancer |   hash of 5-tuple picks a server
                 +--+---+---+--+
                    |   |   |
                 +--+ +-+-+ +--+
                 |S1| |S2 | |S3|   TLS, WAF, cache (RAM + SSD)
                 +--+ +---+ +--+
```

- **Transit** is paid access to the whole internet through a bigger network.
- **Peering** is a direct link between two networks, so traffic takes a shorter path.
- **IXP (Internet Exchange Point)** is a building where many networks connect to each other for peering.
- **ECMP (Equal-Cost Multi-Path)** lets a router spread flows across several paths or servers.
- **5-tuple** means source IP, destination IP, source port, destination port and protocol. Hashing it keeps all packets of one connection on one server.
- **WAF (Web Application Firewall)** inspects HTTP requests and blocks attacks such as SQL injection.

More on balancers is in [Load Balancing](04-load-balancing.md).

CDNs use **consistent hashing** inside a POP. The URL decides which server stores the object, so one object is not stored on every server. That saves disk space. It also means that adding or removing a server moves only a small part of the objects. Very popular objects are the exception, and they may be copied to many servers so that one server does not overload.

Large CDNs also place caches **inside ISP networks**. Netflix Open Connect and Google Global Cache are well-known examples. Video then never has to cross the ISP's expensive outside links.

### Origin and tiers

The origin is where the content comes from. On a cache miss, the edge asks for it. If thousands of POPs all miss at once, the origin gets thousands of requests.

So many CDNs add a middle layer. It has different names: **tiered cache**, **mid-tier**, or **origin shield**. Edge POPs ask the shield first, and only the shield talks to the origin.

```text
                 +---------------------+
                 | Origin (Virginia)   |
                 | 198.51.100.20       |
                 +----------+----------+
                            |
                 +----------+----------+
                 | Shield POP          |
                 +---+------+------+---+
                     |      |      |
              +------+      |      +------+
              |             |             |
        +-----+----+  +-----+----+  +-----+----+
        | Edge BOM |  | Edge FRA |  | Edge SIN |
        +-----+----+  +-----+----+  +-----+----+
              |             |             |
            users         users         users
```

**Pull vs push.** There are two ways for content to reach the edge.

| Model | How it works | Good for |
|---|---|---|
| Pull | Edge fetches on the first request | Most websites and APIs |
| Push | You upload content to the CDN first | Big files and planned launches |

Pull is the common default. The first user in each POP pays the cost of a miss. Many vendors also let you **prefetch**, which means loading chosen objects into cache before users ask.

### Cache hit and cache miss

Every request ends in one of a few results.

```text
Request arrives
      |
Cacheable method and rules? --- no ---> pass through to origin
      | yes
Build the cache key
      |
Key found in cache? ----------- no ---> MISS: fetch, store, serve
      | yes
Copy still fresh? ------------- yes --> HIT: serve at once
      | no
Ask origin: has it changed?
(If-None-Match or If-Modified-Since)
      |
      +-- 304 Not Modified --> REVALIDATED: reuse copy, reset age
      +-- 200 OK ------------> EXPIRED: replace copy, serve new one
```

**Fresh** means the age of the copy is less than its allowed lifetime. **Stale** means it is older. A stale copy is not deleted at once. The edge can ask the origin if it is still good. That is **revalidation**, and a `304 Not Modified` reply has no body, so it saves bandwidth.

The origin controls this with response headers. Caching rules are standardised in RFC 9111.

| Header or directive | Meaning |
|---|---|
| `Cache-Control: max-age=600` | Fresh for 600 seconds in any cache |
| `s-maxage=3600` | Like max-age, but only for shared caches such as a CDN. Wins over max-age there |
| `public` | Any cache may store it |
| `private` | Only the user's browser may store it, not a CDN |
| `no-cache` | May be stored, but must be revalidated before each use |
| `no-store` | Must not be stored anywhere |
| `immutable` | This URL will never change, do not revalidate (RFC 8246) |
| `stale-while-revalidate=60` | Serve the old copy for up to 60 s while fetching a new one (RFC 5861) |
| `stale-if-error=86400` | If origin fails, serve the old copy for up to a day (RFC 5861) |
| `ETag: "5f3a-61b2"` | A version label for this content |
| `Last-Modified` | When the content last changed |
| `Age: 127` | Seconds this copy has been in the cache |
| `Vary: Accept-Encoding` | Keep separate copies per value of that request header |

`no-cache` is a confusing name. It does not mean "do not cache". `no-store` means that.

**Standard vs vendor behaviour.** The RFC sets the rules, but CDNs add their own defaults. Many do not cache HTML unless you ask them to. Some cache by file extension. Many will not store a response that has `Set-Cookie`. Most let you set an edge TTL that ignores the origin's header. There is also a `CDN-Cache-Control` header (RFC 9213) that targets CDNs only and leaves browser rules alone. Always read your vendor's documentation.

**What hit ratio means in practice.** Suppose 2,000 requests per second arrive.

| Hit ratio | Requests to origin per second |
|---|---|
| 90% | 200 |
| 95% | 100 |
| 99% | 20 |

Going from 95% to 99% looks small, but the origin load falls by five times.

There are two kinds of ratio. **Request hit ratio** counts requests. **Byte hit ratio** counts bytes. A site with many small hits and a few huge misses can have a high request ratio and a poor byte ratio.

### Getting the user to the right POP

First, the site must send users to the CDN. The usual way is a DNS **CNAME** record, which says "this name is an alias of that other name".

```text
www.example.com.  300  IN  CNAME  www.example.com.cdn.example.net.
```

Here, `example.net` stands for the CDN vendor's domain. The vendor's DNS now decides the answer. It has two main ways to choose a POP.

#### Anycast

**Anycast** means many sites announce the same IP address. The same prefix, such as 203.0.113.0/24, is announced by **BGP** (Border Gateway Protocol, the routing protocol between networks) from every POP. Each router on the internet sends packets to the path it prefers, which is usually a nearby POP.

```text
      203.0.113.0/24 announced from every POP (AS 64512)

  User in Mumbai      User in Frankfurt     User in Singapore
        |                    |                     |
   best BGP path        best BGP path         best BGP path
        |                    |                     |
   +----+-----+         +----+-----+          +----+-----+
   | POP BOM  |         | POP FRA  |          | POP SIN  |
   +----------+         +----------+          +----------+
   203.0.113.10         203.0.113.10          203.0.113.10
```

Analogy: a pizza chain has one phone number for the whole country, and the phone network connects you to the nearest branch. The analogy stops being true because BGP does not measure distance or speed. It chooses by policy and AS path length. A path that looks short to BGP can be slow.

If a POP fails, it stops announcing the prefix. Routers move to the next best path, usually within seconds to a few minutes. More on this in [BGP](../05-routing/09-bgp.md).

#### DNS steering

With **DNS steering**, every POP has its own IP addresses. The CDN's DNS server picks an answer for each query. It uses location data, POP load, POP health and live latency measurements.

```text
 Resolver in Mumbai asks:     www.example.com?  -->  203.0.113.10
 Resolver in Frankfurt asks:  www.example.com?  -->  203.0.113.30
```

Analogy: you phone a receptionist, who hears which town you are in and gives you the local branch's number. The analogy stops being true in two ways. The DNS server does not see you. It sees your **resolver**, which is the DNS server that asks on your behalf, and that may be far away. Also, the answer is remembered until its TTL ends, so a change is not instant.

The fix for the first problem is **EDNS Client Subnet** (RFC 7871). The resolver passes on a part of the client's address, such as a /24. Not every resolver sends it, and some leave it out for privacy.

#### Comparing the two

| Point | Anycast | DNS steering |
|---|---|---|
| Who decides | BGP routers on the internet | CDN's DNS servers |
| IP addresses | Same IP at every POP | Different IP per POP |
| Failover | BGP withdrawal, seconds to minutes | Waits for DNS TTL, plus caching that ignores it |
| Fine control | Weak, BGP is a blunt tool | Strong, can shift a percentage of users |
| Uses live load data | Only indirectly | Yes |
| Accuracy problem | BGP policy may pick a far POP | Resolver may be far from the user |
| Under DDoS | Spreads attack across POPs | One POP's IP can be flooded |
| Risk to TCP | A route change can move a flow to a POP with no state for it | Connection stays on the same IP |
| Well-known users | Cloudflare, Fastly | Akamai, Amazon CloudFront |

**Fuller truth:** the line is not sharp. Many CDNs mix both. For example, DNS may return one of several regional anycast addresses, which gives DNS control and anycast failover. CloudFront also offers static anycast IPs, and Akamai uses anycast for its DNS service.

### Cache keys

A **cache key** is the label used to find a stored copy. If two requests build the same key, they get the same copy.

A common default is:

```text
| scheme | host            | path          | query    |
| https  | www.example.com | /img/logo.png | ?v=2     |
```

Headers named in `Vary` are added to this. So `Vary: Accept-Encoding` keeps a separate Brotli copy and gzip copy.

The key must be **as small as possible, but no smaller**.

**Too wide (hit ratio falls).** These URLs show the same page, but they are three keys, and each one misses:

```text
/shoes?utm_source=mail
/shoes?utm_source=ad&utm_campaign=sale
/shoes?fbclid=abc123
```

The fix is to remove tracking parameters from the key, and to sort the rest. `Vary: User-Agent` is the same kind of problem, because there are thousands of browser strings. Group them into `mobile` and `desktop` instead. Putting cookies in the key usually destroys the hit ratio.

**Too narrow (wrong content).** Suppose a page changes by language, but the key does not include language. A French user fills the cache, and then everyone else gets French. Worse, if a personal page is cached, one user can see another user's data. This is a real type of incident, so treat it as a security bug.

### Invalidation

**Invalidation** means making stored copies stop being used. The hard part is that thousands of servers hold copies.

| Method | How it works | Notes |
|---|---|---|
| TTL expiry | Copies age out | Simple, but you wait |
| Purge by URL | Remove one URL | Exact, but needs every variant listed |
| Purge by tag | Remove everything with a label | Also called surrogate keys. Vendor feature |
| Purge all | Empty the whole cache | Dangerous, origin takes all the load |
| Soft purge | Mark stale, do not delete | Old copy stays usable if origin fails |
| Versioned URLs | Publish a new name | Best for static files |

**Versioned URLs** are the cleanest method. The build puts a hash of the file in its name:

```text
/assets/app.3f9a1c.js
Cache-Control: public, max-age=31536000, immutable
```

When the file changes, its name changes. The new HTML points to the new name, and nothing needs purging. The old file simply stops being requested. `max-age=31536000` is one year.

Then only the HTML needs care. Give it a short TTL, or purge it on each deploy.

**A purge only reaches the CDN.** It does not clean browser caches, or other caches on the path. If you sent `max-age=86400`, some browsers keep the old copy for a day. To keep control, use a long `s-maxage` with a short `max-age`.

Purge speed depends on the vendor. Many claim seconds. Do not assume it is instant or that it needs no testing.

### DDoS absorption

**DDoS (Distributed Denial of Service)** means many machines send traffic to make a service fail. Types and details are in [Common Network Attacks](../09-security/03-common-network-attacks.md). This is how a CDN helps.

| Attack layer | Example | CDN defence |
|---|---|---|
| L3/L4 volume | UDP reflection floods | Huge capacity, drop at the edge router |
| L4 state | SYN flood | SYN cookies, fast packet filtering |
| L7 | HTTP request flood | Cache, rate limits, WAF, bot checks |

**How anycast helps.** Bots are spread around the world, so each one reaches its nearest POP. The flood is split into many smaller streams. A flood too big for one POP may be small across all of them. If the attack comes from one region, the harm stays in that region's POP.

Analogy: a flood of water hits one small drain and the street floods. Send the same water to a thousand drains and each one copes. The analogy stops being true because attack traffic is not neutral water. Filters also have to tell bad packets from good ones, and that is the hard part at Layer 7.

**How the edge helps by being simple.** A web CDN only needs TCP 80 and 443, and UDP 443 for HTTP/3. Everything else can be dropped at the edge. Many reflection attacks use source ports such as UDP 53 or 123, and they are easy to filter by port and pattern.

**How caching helps.** A flood of `GET /` is served from cache, so the origin sees almost nothing.

**Scale.** Publicly reported attacks passed 20 Tbps in 2025. HTTP/2 Rapid Reset, disclosed in October 2023, reached a reported 398 million requests per second at Google. Few single sites have that kind of capacity.

**How attackers get around it:**

1. **Hit the origin directly.** If the attacker finds the origin IP, the CDN is skipped. The IP can leak through old DNS history, mail servers on the same host, or internet-wide certificate scans.
2. **Cache busting.** Random query strings such as `/?x=8127361` create a miss every time, so each request reaches the origin.
3. **Attack uncacheable paths.** Search, login and checkout must reach the origin and may be costly.

The defences are to hide the origin (next section), to ignore unknown query parameters, and to rate-limit costly paths.

### TLS at the edge

**TLS termination** means the TLS session ends at the CDN edge, not at the origin. There are two separate connections.

```text
  Browser                Edge (BOM)                Origin
     |                       |                        |
     |=== TLS session 1 ====>| decrypts, reads HTTP   |
     |   cert for            | checks cache           |
     |   www.example.com     |                        |
     |                       |=== TLS session 2 =====>|
     |                       |   cert for             |
     |                       |   origin.example.com   |
```

Why it must be this way: a CDN cannot cache what it cannot read. If the edge only forwarded encrypted bytes, it would be a tunnel. It could not see the URL, or store anything.

Why it helps: the TCP and TLS handshakes take several round trips. They are much faster with a server 8 ms away than with one 200 ms away.

**SNI (Server Name Indication).** One edge IP serves thousands of websites. The client states the host name in the TLS ClientHello, so the edge can pick the right certificate. SNI is defined in RFC 6066 and is normally visible on the wire. **Encrypted Client Hello (ECH)** hides it, and it works best behind a CDN, where many sites share one front.

**Certificates.** There are three common choices.

- The CDN gets and renews one for you, usually with ACME (RFC 8555), the automatic certificate protocol.
- You upload your own certificate and private key.
- You keep the key on your own server, and the edge asks it to sign during the handshake. Cloudflare calls this Keyless SSL.

Automation now matters more. The maximum life of a public TLS certificate fell to 200 days in March 2026, and it is scheduled to fall to 47 days by 2029.

**The trust cost.** The CDN sees your traffic in plain text, including passwords and cookies. You are trusting that company. This is often the first thing a security reviewer asks about. If the data is regulated, also check where POPs may store or decrypt it.

Analogy: a trusted local office opens your sealed letter, reads it, and puts it in a new sealed envelope for head office. The analogy stops being true at scale, because the "office" handles billions of letters per day. A single bug there could affect many customers. In 2017, the Cloudbleed bug made Cloudflare servers leak memory from other sites.

**Origin-side modes.** The second connection needs care too.

| Mode | Edge to origin | Safe? |
|---|---|---|
| Plain HTTP | No encryption | No. User sees a padlock, but the path is open |
| TLS, no verify | Encrypted, certificate unchecked | Weak. Vulnerable to a fake origin |
| TLS, verified | Encrypted, certificate checked | Yes. Use this |
| Verified plus mTLS | Both sides prove identity | Best |

**mTLS (mutual TLS)** means the client also shows a certificate. Cloudflare's names for the first and third modes are "Flexible" and "Full (strict)".

### How a request flows

Here is the whole path for one cached image.

```text
Browser       Resolver      CDN DNS       Edge POP      Origin
   |             |             |             |             |
   |-- 1 DNS --->|             |             |             |
   |             |-- 2 DNS --->|             |             |
   |             |<-- 3 IP ----|             |             |
   |<-- 4 IP ----|             |             |             |
   |                                         |             |
   |== 5 TCP + TLS (or QUIC) ===============>|             |
   |== 6 GET /img/logo.png =================>|             |
   |                                         | 7 WAF, key  |
   |                                         | 8 look up   |
   |                                         |             |
   |   on a MISS only:                       |-- 9 GET --->|
   |                                         |<-- 10 200 --|
   |                                         | 11 store    |
   |<========= 12 200 OK and bytes ==========|             |
```

1. The browser asks its resolver for `www.example.com`.
2. The resolver follows the CNAME to the CDN's DNS.
3. The CDN's DNS returns an edge IP, chosen by steering, or a fixed anycast IP.
4. The browser gets the address.
5. The browser opens TCP and TLS to the edge. Or it uses QUIC over UDP 443 if HTTP/3 is available. The router sends it to a POP, and the L4 balancer picks a server. SNI selects the certificate.
6. The browser sends the HTTP request.
7. The edge runs security rules, then builds the cache key.
8. The edge looks up the key.
9. On a miss, the edge asks a shield tier or the origin, usually over a reused connection. It adds `X-Forwarded-For` so the origin knows the real client IP.
10. The origin answers with content and cache headers.
11. The edge stores the copy if the rules allow.
12. The edge sends the response, often streaming it while it is still arriving.

On a hit, steps 9 to 11 do not happen.

## Example

### Setup

| Item | Value |
|---|---|
| Site | `www.example.com` |
| Origin | `origin.example.com`, 198.51.100.20, in Virginia |
| Anycast edge IP | 203.0.113.10 |
| User | Mumbai, public IP 192.0.2.50 |
| RTT, user to Mumbai POP | 8 ms |
| RTT, Mumbai POP to origin | 200 ms |
| RTT, user directly to origin | 210 ms |

### Step 1: find where the name goes

Linux or macOS:

```bash
dig www.example.com +noall +answer
```

```text
www.example.com.   300 IN CNAME www.example.com.cdn.example.net.
www.example.com.cdn.example.net. 60 IN A 203.0.113.10
```

Windows PowerShell:

```powershell
Resolve-DnsName www.example.com
```

A CNAME to a CDN name, with a low TTL on the final address, is the classic sign of a CDN.

### Step 2: see a miss

The commands are the same on Linux, macOS and Windows. In PowerShell, type `curl.exe`, because `curl` alone can run a different command on older versions.

```bash
curl -sI https://www.example.com/img/logo.png
```

```text
HTTP/2 200
date: Fri, 30 Oct 2026 10:00:00 GMT
content-type: image/png
content-length: 48213
cache-control: public, max-age=86400
etag: "5f3a-61b2"
last-modified: Mon, 05 Oct 2026 08:12:44 GMT
age: 0
cache-status: ExampleCDN; fwd=uri-miss; stored
alt-svc: h3=":443"; ma=86400
```

`cache-status` is the standard debug header (RFC 9211). Many vendors use their own instead, such as `cf-cache-status: MISS` at Cloudflare and `x-cache: Miss from cloudfront` at CloudFront. `alt-svc` tells the client that HTTP/3 is available.

### Step 3: see a hit

Run it again 127 seconds later:

```text
HTTP/2 200
date: Fri, 30 Oct 2026 10:02:07 GMT
content-type: image/png
content-length: 48213
cache-control: public, max-age=86400
etag: "5f3a-61b2"
last-modified: Mon, 05 Oct 2026 08:12:44 GMT
age: 127
cache-status: ExampleCDN; hit; ttl=86273
alt-svc: h3=":443"; ma=86400
```

The numbers agree: 86400 - 127 = 86273 seconds of life left. The copy stays fresh until 10:00:00 the next day.

The origin's log shows only one request, from an edge server's address:

```text
203.0.113.200 "GET /img/logo.png" 200 48213 xff=192.0.2.50
```

The first field is the edge. The real client is in `X-Forwarded-For`.

### Step 4: revalidate when it expires

After a day, the edge sends a conditional request to the origin:

```text
GET /img/logo.png HTTP/1.1
Host: www.example.com
If-None-Match: "5f3a-61b2"
```

The origin replies:

```text
HTTP/1.1 304 Not Modified
ETag: "5f3a-61b2"
Cache-Control: public, max-age=86400
```

No image bytes cross the ocean. The copy is fresh for another day.

### Step 5: work out the latency gain

A new HTTPS connection over TCP with TLS 1.3 needs 3 RTTs until the first response byte: 1 for TCP, 1 for TLS, and 1 for the request. This ignores DNS and server time.

| Case | Working | First byte |
|---|---|---|
| No CDN | 3 x 210 ms | 630 ms |
| CDN hit | 3 x 8 ms | 24 ms |
| CDN miss, warm origin connection | 24 + 200 ms | 224 ms |
| CDN miss, new origin connection | 24 + (3 x 200) ms | 624 ms |

Two lessons appear here.

A hit is 26 times faster than no CDN. Also, a warm connection is what makes a miss quick. A cold miss gains almost nothing.

With a 95% hit ratio and warm connections, the average is:

```text
(0.95 x 24 ms) + (0.05 x 224 ms) = 22.8 + 11.2 = 34 ms
```

### Step 6: work out the origin saving

Take 2,000 requests per second, each 200 KB on average.

```text
Total:  2,000 x 200 KB = 400 MB/s = 3.2 Gbps
At 95%: origin sends 5% of that = 20 MB/s = 160 Mbps
```

This assumes the byte hit ratio is also 95%.

### Step 7: test one POP, or the origin

`--resolve` sends the request to a chosen IP, but keeps the correct host name for SNI.

```bash
# Is the origin itself healthy? Skip the CDN.
curl -sI --resolve www.example.com:443:198.51.100.20 \
  https://www.example.com/img/logo.png
```

This tells you who is wrong. If the origin returns new content and the edge returns old content, the problem is in the cache.

### Step 8: see anycast on a real network

Cloudflare's public resolver at 1.1.1.1 uses anycast. This query asks which POP answered you:

```bash
dig @1.1.1.1 id.server CH TXT +short
```

```text
"BOM"
```

You get an airport code for a site near you, or at least near by network path. A friend in another country runs the same command, to the same IP, and gets a different code.

## Going Deeper

### Stopping a stampede

A **cache stampede** (or thundering herd) happens when a popular object expires, and thousands of requests miss at the same moment. All of them go to the origin.

Three defences work together.

- **Request collapsing (coalescing):** the edge sends one request to the origin, and the others wait for that response.
- **Stale-while-revalidate:** users get the old copy at once while one background request gets the new one.
- **Tiered cache:** many POPs ask one shield, which collapses again.

Collapsing can go wrong. If a personal response is shared between waiting users, data leaks. So responses marked `private` must not be shared this way.

### A tiny CDN node in nginx

This shows the main ideas in a real tool. It is one node only, without a global network.

```text
# nginx on Linux
proxy_cache_path /var/cache/nginx keys_zone=edge:100m
                 max_size=50g inactive=7d use_temp_path=off;

server {
    listen 443 ssl;
    server_name www.example.com;
    ssl_certificate     /etc/ssl/www.example.com.crt;
    ssl_certificate_key /etc/ssl/www.example.com.key;

    location / {
        proxy_pass https://origin.example.com;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;

        proxy_ssl_server_name on;           # send SNI to origin
        proxy_ssl_verify on;                # check origin's certificate
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;

        proxy_cache edge;
        proxy_cache_key $scheme$host$uri$is_args$args;
        proxy_cache_lock on;                # request collapsing
        proxy_cache_revalidate on;          # use ETag, get 304
        proxy_cache_background_update on;   # refresh in background
        proxy_cache_use_stale updating error timeout http_502;
        add_header X-Cache-Status $upstream_cache_status always;
    }
}
```

Here, `$upstream_cache_status` shows values such as HIT, MISS, EXPIRED, STALE and REVALIDATED.

### Dynamic content acceleration

Pages that cannot be cached can still go faster.

- **Connection reuse.** The edge holds open connections to the origin, so requests skip the handshakes. Example above: 224 ms vs 624 ms.
- **Better paths.** The CDN measures paths between its own POPs and picks a fast one, not the BGP default.
- **Better TCP.** Short edge-to-user loops recover from loss faster, because a lost packet is detected in 8 ms of RTT, not 200 ms. See [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).
- **HTTP/3 to the client.** The edge can speak [QUIC](../06-transport-layer/08-quic.md) to users, even if the origin only speaks HTTP/1.1 over TCP. Protocols on the two sides are independent. The same goes for IPv6: an edge can offer IPv6 to users for an IPv4-only origin.

### Edge compute and other edge features

Modern CDNs run customer code at the POP. Cloudflare Workers and AWS Lambda@Edge are examples. Uses include redirects, A/B tests, header rewriting and token checks.

Other common features are image resizing, Brotli compression, bot detection and API rate limiting.

### Private content

Paid video should not be open to anyone with the URL. A **signed URL** adds an expiry time and a cryptographic signature.

```text
https://www.example.com/video/ep1.mp4?expires=1793354400&sig=9a7f...
```

The edge checks the signature using a shared secret, without calling the origin. The signature must be left out of the cache key. Otherwise, every user creates a separate copy.

### Video delivery

Streaming with **HLS** or **DASH** cuts video into small files, often a few seconds long. Each one is a normal HTTP object, so normal caching applies. Very large files use HTTP **range requests** (RFC 9110), so the CDN can store and serve parts of a file. This is a large reason that streaming scales.

### Protecting the origin

Blocking direct access makes the CDN impossible to bypass.

1. Allow inbound traffic only from your CDN's published ranges.
2. Add proof of identity, because other customers of the same CDN share those ranges. Use mTLS from the edge, a secret header, or both.
3. Use an outbound tunnel if your vendor offers one, so the origin has no open inbound port at all.

Example for step 1, Linux nftables, with 203.0.113.0/24 as the CDN range:

```bash
nft add table inet origin
nft add chain inet origin input \
  '{ type filter hook input priority 0; policy accept; }'
nft add rule inet origin input tcp dport 443 \
  ip saddr 203.0.113.0/24 accept
nft add rule inet origin input tcp dport 443 drop
```

Keep the list up to date, because vendors change their ranges. See [iptables and nftables](../11-cloud-and-containers/02-iptables-and-nftables.md).

### Cache poisoning and cache deception

- **Cache poisoning:** an attacker sends a request whose response is harmful, and the cache stores it for everyone. A typical route is an input that changes the page but is not in the key, such as a reflected `X-Forwarded-Host` header.
- **Cache deception:** an attacker tricks a victim into opening `/account/profile.css`. The site ignores the fake suffix and returns the private page, but the CDN sees `.css` and caches it. The attacker then fetches the same URL.

Defences: cache only by explicit rules, send `Cache-Control: private` or `no-store` on personal pages, and strip headers you do not use.

### Anycast and long connections

A BGP change can move a flow to another POP mid-connection. That POP has no state for it, so it sends a TCP reset. In practice this is rare, because routes are mostly stable, and web requests are short. It matters more for long downloads and WebSockets, and clients should be ready to reconnect. QUIC connection IDs help with a client's network change, but they do not by themselves let a new POP take over a session.

Also, routes between networks accept /24 as the normal smallest IPv4 prefix and /48 for IPv6. So anycast needs at least that size.

### DNS problems at the apex

A CNAME cannot sit at the top of a zone, such as `example.com`, because the name must also hold SOA and NS records, and a CNAME cannot share a name with other records. That is why sites use `www`. DNS providers work around it with vendor features called CNAME flattening, ALIAS or ANAME. See [DNS Record Types](../07-application-protocols/02-dns-records.md).

### Multi-CDN and concentration risk

A CDN is a shared dependency. The Fastly outage of June 2021 and the Cloudflare outage of November 2025 took many large sites down at once. Big sites use **multi-CDN**, with DNS steering that moves traffic between vendors based on health and price. The cost is more complex setup: purge, TLS and logs must all work across every vendor.

### CDN, reverse proxy, load balancer

| Tool | Main job | Location |
|---|---|---|
| Reverse proxy | Stand in front of servers | Usually one site |
| Load balancer | Share work across servers | Usually one site |
| CDN | Globally distributed reverse proxy with cache | Worldwide |

A CDN is built from the other two. See [Load Balancing](04-load-balancing.md).

## Common Mistakes

- **"A CDN caches everything."** It caches only what headers and rules allow. An API with `no-store` gets no caching at all.
- **"`no-cache` means do not store."** It means store, but check before use. `no-store` means do not store.
- **"TTL is a promise that the copy stays."** It is a maximum. An object that nobody requests can be evicted early, to make room.
- **"A CDN hides my origin."** Only if the origin's IP is unknown and it refuses direct traffic. Old DNS records often reveal it.
- **"Anycast picks the fastest POP."** BGP picks by policy and AS path. It knows nothing about latency.
- **"Purging fixes every user."** Browsers may still hold the old copy until their own `max-age` ends.
- **"HTTPS to the CDN means it is encrypted all the way."** The edge decrypts it. The edge-to-origin path may be plain HTTP, even with a padlock in the browser.
- **Redirect loops.** The edge speaks HTTP to the origin, the origin redirects to HTTPS, and the edge fetches HTTP again. The browser reports too many redirects.
- **Rate limiting by IP at the origin.** All requests come from edge IPs, so everyone looks like the same few clients. Use the forwarded client IP, but trust it only from the CDN.
- **Long TTLs on HTML.** You then cannot ship a fix quickly. Use long TTLs on versioned files and short ones on HTML.
- **Reading a 5xx as a CDN failure.** A `502` or `504` from the edge usually means that the edge could not get a good answer from the origin. Check the origin first.
- **Ignoring the first request.** A fresh deploy with empty caches sends a burst to the origin. Plan for it with a shield or prefetching.

## Interview Questions

**Q: What is a CDN, and what problem does it solve?**

A: A CDN is a set of servers in many locations that sit between users and the origin. It solves three problems. It cuts latency by serving from nearby. It cuts origin load through caching. And it absorbs attacks with its large, spread-out capacity.

**Q: What is a cache hit, a cache miss and a hit ratio?**

A: A hit means the edge has a fresh copy and serves it directly. A miss means it must fetch from the origin, and it usually stores the result. Hit ratio is the share of requests served from cache. At 2,000 requests per second, 95% leaves 100 for the origin, and 99% leaves 20.

**Q: Walk me through a request to a site behind a CDN.**

A: DNS for the site returns a CNAME to the CDN, and the CDN returns an edge IP. The browser opens TCP and TLS, or QUIC, to the nearest POP. SNI selects the certificate. The edge runs security rules, builds the cache key and looks it up. On a hit, it answers at once. On a miss, it fetches from a shield or the origin over a reused connection, stores the response and sends it on.

**Q: Compare anycast and DNS steering.**

A: With anycast, every POP announces the same IP, and BGP sends each user to a preferred POP. It fails over quickly and spreads DDoS, but control is rough, because BGP does not know latency. With DNS steering, the CDN's DNS gives each user a POP's own IP. It allows fine control using load and measurements, but it sees the resolver rather than the user, and it depends on TTLs. Many CDNs combine the two.

**Q: What is a cache key, and what happens if it is wrong?**

A: It is the value used to find a stored object, normally scheme, host, path and query, plus any `Vary` headers. Too many parts, such as tracking parameters, and the hit ratio falls. Too few parts, and users can get the wrong language or even another person's data.

**Q: How do you invalidate content on a CDN?**

A: You can let the TTL expire, purge by URL or tag, or use versioned file names. I prefer versioned names for static files, with a one-year `immutable` TTL, since nothing needs purging. HTML gets a short TTL or a purge on deploy. I avoid purge all, because it sends a wave of traffic to the origin.

**Q: What is the difference between `no-cache`, `no-store`, `max-age` and `s-maxage`?**

A: `no-store` means never save the response. `no-cache` means it may be saved, but must be checked with the origin before use. `max-age` sets a freshness time for all caches. `s-maxage` overrides it for shared caches such as a CDN. So I can set a long time at the edge and a short time in browsers.

**Q: How does a CDN absorb a DDoS attack, and how can an attacker get around it?**

A: Anycast splits a flood across many POPs, so no single site takes it all. Edge routers drop traffic on ports that are not needed, SYN cookies handle SYN floods, and caching absorbs HTTP floods. An attacker can bypass this by finding the origin's IP, or by using random query strings to force misses. The defences are origin lockdown with allowlists and mTLS, cache key rules and rate limits.

**Q: Why does a CDN terminate TLS, and what is the risk?**

A: It must read the HTTP request to cache anything, so it has to decrypt. It also makes handshakes much faster, as the RTT is short. The risk is trust, because the CDN sees plain text and may hold your private key. Keyless options reduce the key risk. The edge-to-origin connection must also use verified TLS, or the padlock is misleading.

**Q: What is a cache stampede, and how is it prevented?**

A: When a popular object expires, many requests can miss together and hit the origin. CDNs prevent it with request collapsing, so that only one request goes to the origin. Stale-while-revalidate serves the old copy during the refresh, and a shield tier reduces the number of requests further.

**Q: Can a CDN help with content that cannot be cached?**

A: Yes. The edge shortens handshakes for the user, reuses warm connections to the origin, and may use faster internal paths. In one example with 8 ms and 200 ms RTTs, a warm connection gives 224 ms to first byte against 630 ms without a CDN. Security features also still apply.

**Q: Users still see old content after a deploy. How do you debug it?**

A: First I check the browser with a hard reload. Then I use `curl -I` and read `Age`, `ETag` and the cache status header. Next I use `curl --resolve` to fetch from the origin directly and compare. If the origin is new and the edge is old, I check the TTL, the purge result and whether the cache key has variants that were not purged. If only some regions are affected, I test other POPs.

## Quick Recap

- A CDN puts servers near users, so content travels less distance, and the origin does less work.
- A hit serves a stored copy, a miss fetches from the origin, and small gains in hit ratio cut origin load greatly.
- Anycast uses BGP and one shared IP. DNS steering uses the CDN's DNS and per-POP IPs. Many CDNs mix both.
- The cache key decides which requests share a copy, so wrong keys cause low hit ratios or data leaks.
- Prefer versioned URLs with long TTLs, and use purges for the rest. A purge does not reach browser caches.
- TLS ends at the edge so that the CDN can read and cache, which means you must trust it and must secure the edge-to-origin path.
- DDoS protection only works if the origin cannot be reached directly.
