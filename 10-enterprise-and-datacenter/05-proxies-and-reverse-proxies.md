# Proxies and Reverse Proxies

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [HTTP Basics](../07-application-protocols/05-http-basics.md), [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [Load Balancing](04-load-balancing.md)

## In One Line

A proxy is a program that stands between two sides of a connection and speaks to each side for the other. A forward proxy works for clients, and a reverse proxy works for servers.

## Why It Matters

Almost every web request passes through at least one proxy. A company may send staff traffic through a filter. A website may put NGINX, a cloud load balancer, or a Content Delivery Network (CDN) in front of its servers.

Proxies solve real problems:

- **Control:** a company can block sites, log traffic, and stop servers from reaching the internet directly.
- **Protection:** a website can hide its servers and expose one small, hardened entry point.
- **Speed:** a proxy can cache (store and reuse) answers and handle slow clients.
- **Central work:** one place can handle TLS, routing, and rate limits, so every app does not need to do it.

Interviewers ask about proxies because the topic tests many layers at once. A good answer touches TCP, HTTP, TLS, and addressing. Common questions are "forward vs reverse", "how does HTTPS pass through a proxy", and "how does the backend learn the real client IP".

## Simple Explanation

Think of a large office building.

A **forward proxy** is like an assistant who makes all outside phone calls for the staff. The person on the other end talks to the assistant and never learns which employee asked. The assistant can also refuse: "Sorry, you may not call that number."

A **reverse proxy** is like the reception desk. Visitors come to one desk and never walk straight to a worker's room. Reception decides who should handle the visit and sends the request there. Visitors do not know how many workers exist or where they sit.

The key question is simple: **who does the proxy work for?**

- The client side chose it and uses it to reach many servers: forward proxy.
- The server side chose it and uses it to receive many clients: reverse proxy.

Where the analogy stops being true: an assistant is a person who understands every call. A real proxy understands only the protocols it was built for. With encrypted traffic, it may see only the destination name and some byte counts.

## How It Works

### Terms used in this section

- **Client:** the program that starts the request, such as a browser or `curl`.
- **Origin server:** the server that really holds the content.
- **Backend (or upstream):** a server that sits behind a proxy. NGINX uses the word "upstream" for it.
- **Intermediary:** anything in the middle of a connection, such as a proxy.
- **Layer 4 (L4):** the transport layer. It sees IP addresses and ports.
- **Layer 7 (L7):** the application layer. It sees HTTP methods, URLs, and headers.

### A proxy makes two connections

This is the most important idea in the chapter. A normal L7 proxy does not forward your packets. It ends your TCP connection and opens a second one.

```text
 Client                    Proxy                     Server
 192.168.1.20              10.0.2.10                 10.0.2.21:8080
    |                        |                         |
    |=== TCP connection 1 ===|                         |
    |   HTTP request ------> |                         |
    |                        |=== TCP connection 2 ====|
    |                        |   HTTP request ------>  |
    |                        |   <------ HTTP response |
    |   <------ HTTP response|                         |
```

Each connection has its own handshake, sequence numbers, window size, and timeouts. This has three results:

1. The server sees the proxy's IP address as the source, not the client's.
2. The proxy can read and change the request, if it is not encrypted.
3. The proxy can give each side a different protocol. For example, HTTP/2 with the client and HTTP/1.1 with the backend.

Fuller truth: some L4 devices forward packets and only change addresses. Those act like NAT, not like a full proxy. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md) and [Load Balancing](04-load-balancing.md).

### Forward proxy

A forward proxy sits near the clients. The client knows about it and sends every request to it. This is called an **explicit proxy**.

```text
 +----------------- Company network -----------------+
 |                                                   |
 |  Laptop 10.0.3.25 ---+                            |
 |                      |                            |
 |  Laptop 10.0.3.26 ---+--> Forward proxy 10.0.3.5 -+--> Internet
 |                      |    (port 3128)             |    web server
 |  Server 10.0.3.40 ---+                            |
 |                                                   |
 +---------------------------------------------------+
```

The steps for plain HTTP:

1. The client opens TCP to the proxy, not to the website.
2. The client sends the **full URL** in the request line. This is called the absolute form.
3. The proxy checks its rules. It may also ask the user to log in.
4. The proxy opens its own connection to the website and sends the request.
5. The proxy returns the answer to the client.

A normal request to a server, and the same request to a proxy, look like this:

```text
Direct to the server:

GET /index.html HTTP/1.1
Host: www.example.com

Sent to a forward proxy:

GET http://www.example.com/index.html HTTP/1.1
Host: www.example.com
```

If the proxy needs a login, it answers `407 Proxy Authentication Required` with a `Proxy-Authenticate` header. The client retries with a `Proxy-Authorization` header. Companies often use Kerberos or NTLM logins here, so staff do not type a password.

Common forward proxy uses:

- Web filtering, for example blocking malware sites.
- Logging who visited what.
- Egress control. **Egress** means traffic leaving a network. Data centre servers often have no direct internet route. They reach package mirrors only through a proxy with an allow list.
- Data loss prevention, which looks for private data leaving the company.
- Getting one exit IP address, so a partner can allow only that address.

Common products are Squid (default port 3128), and cloud services called Secure Web Gateways.

### How a client finds its proxy

| Method | How it works | Used by |
|---|---|---|
| Manual setting | Address and port typed into the OS or browser | Small setups |
| Environment variables | `http_proxy`, `https_proxy`, `no_proxy` | Linux tools, containers, build jobs |
| PAC file | A small JavaScript file with a `FindProxyForURL(url, host)` function | Companies |
| WPAD | The client finds a PAC file through DHCP or DNS | Some Windows networks |
| Interception | The client knows nothing; the network redirects traffic | See "Transparent proxy" below |

A **PAC file** (Proxy Auto-Configuration) decides per URL. For example, it can return `DIRECT` for internal sites and `PROXY 10.0.3.5:3128` for everything else.

### HTTPS through a proxy: the CONNECT tunnel

There is a problem. With HTTPS, the client wants encryption all the way to the website. The proxy has no keys, so it cannot read the request. The client cannot send a full URL in clear text either, because that would leak the path.

The answer is the HTTP `CONNECT` method (RFC 9110). The client asks the proxy to open a raw TCP tunnel. After that, the proxy only copies bytes in both directions.

**Analogy:** you ask a telephone operator to connect you to a number. The operator connects the line, then stops listening. Unlike a real operator, the proxy can still see the time and the amount of data, and it can close the line at any moment.

```text
 Client                 Proxy 10.0.3.5:3128        www.example.com:443
    |                         |                             |
    |--- TCP handshake ------>|                             |
    |                         |                             |
    |--- CONNECT              |                             |
    |    www.example.com:443 >|                             |
    |                         |--- TCP handshake ---------->|
    |<-- 200 Connection       |                             |
    |    established ---------|                             |
    |                         |                             |
    |=========== TLS handshake (proxy copies bytes) ========|
    |=========== encrypted HTTP (proxy cannot read) ========|
```

The steps:

1. The client opens TCP to the proxy.
2. The client sends a clear text request:

```text
CONNECT www.example.com:443 HTTP/1.1
Host: www.example.com:443
Proxy-Authorization: Basic dXNlcjpwYXNz
```

3. The proxy checks the rules. It should allow only a small list of ports, usually 443. Otherwise users could tunnel to anything, such as a mail server on port 25.
4. The proxy opens TCP to `www.example.com:443`.
5. The proxy answers with any 2xx status, normally `HTTP/1.1 200 Connection established`.
6. From now on, the proxy copies bytes. The client runs TLS straight to the website.

What the proxy can see in a CONNECT tunnel:

- The host name and port, from the CONNECT line.
- The client IP, the time, and the byte counts.
- The Server Name Indication (SNI, the site name sent in the TLS hello), unless Encrypted Client Hello is used.

What it cannot see: the URL path, the cookies, the body, and the response content.

The `Basic` login above is only Base64, which is encoding and not encryption. Anyone on the path to the proxy can decode it. Use a stronger method, or use TLS to the proxy itself. Then the client has TLS inside TLS. curl supports this with `-x https://proxy.example.com:443`.

To see inside HTTPS, a company must perform **TLS interception**, which is covered in "Going Deeper".

### Reverse proxy

A reverse proxy sits near the servers. Clients do not know it is a proxy. They think it is the website, because DNS points to it.

```text
                             +--> 10.0.2.21:8080  web app 1
 Internet      Reverse proxy |
 clients ----> 203.0.113.10 -+--> 10.0.2.22:8080  web app 2
                443          |
                             +--> 10.0.2.31:9000  API service
```

The steps:

1. DNS for `app.example.com` returns 203.0.113.10.
2. The client connects and completes TLS with the proxy.
3. The proxy decrypts and reads the request. This is called **TLS termination**.
4. The proxy picks a backend by host name or path. `/api/` may go to one service, and `/` to another.
5. The proxy adds headers, such as the client's IP, and sends the request.
6. The proxy returns the answer, and may store a copy.

If the proxy cannot get a good answer, the client sees one of these codes:

| Code | Meaning at a proxy | Typical cause |
|---|---|---|
| 502 Bad Gateway | The backend gave a bad answer or refused | App is down, wrong port, connection reset |
| 503 Service Unavailable | No backend can take the request | All backends failed, or a limit was reached |
| 504 Gateway Timeout | The backend did not answer in time | Slow query, firewall dropping packets |

What a reverse proxy gives you:

- **TLS termination:** one place for certificates.
- **Load balancing:** one name, many servers. See [Load Balancing](04-load-balancing.md).
- **Path routing:** different URLs go to different services.
- **Caching and compression:** less work for the app.
- **Buffering:** the proxy deals with a slow phone, so an app worker is not held for minutes.
- **Protection:** rate limits, size limits, and a Web Application Firewall (a filter for HTTP attacks).
- **Hidden servers:** backends can use private addresses only.

Common products are NGINX, HAProxy, Envoy, Apache httpd, Caddy, and Traefik. The HTTP standard uses the word **gateway** for a reverse proxy, which explains the names of the 502 and 504 errors.

### Forward vs reverse at a glance

| Question | Forward proxy | Reverse proxy |
|---|---|---|
| Who runs it? | The client's organisation | The website's owner |
| Who is hidden? | The client | The servers |
| Where is it? | Near clients | Near servers |
| Does the client know? | Usually yes, it is configured | No, it looks like the website |
| What does it reach? | Any allowed site | A fixed set of backends |
| Main goals | Control, logging, egress | Scale, TLS, routing, protection |
| Typical port | 3128 or 8080 | 80 and 443 |
| Typical products | Squid, Secure Web Gateways | NGINX, HAProxy, Envoy |

### Transparent proxy

A **transparent proxy** catches traffic without any client setting. The client thinks it is talking straight to the website. It is also called an **interception proxy**.

**Analogy:** a mail room opens all letters passing through the building, though nobody asked it to. Your letter still has the address you wrote. Unlike a mail room, a proxy can only read what is not encrypted.

The network must steer the traffic to the proxy. Common ways are:

- `iptables` or `nftables` rules on a Linux gateway.
- Policy-based routing on a router.
- **WCCP** (Web Cache Communication Protocol), a Cisco protocol that redirects traffic to a cache.

```text
 Client 10.0.3.25                 Gateway 10.0.3.1
 "connect to                      +-----------------------+
  203.0.113.80:80" -------------> | iptables REDIRECT     |
                                  |   port 80 -> 3129     |
                                  |          |            |
                                  |          v            |
                                  | Squid intercept :3129 |---> Website
                                  +-----------------------+
```

Linux gateway example, with the LAN on `eth1`:

```bash
# Linux: redirect LAN web traffic to Squid on this machine
sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
  -j REDIRECT --to-ports 3129
```

```text
# /etc/squid/squid.conf (Squid, any OS)
http_port 3128
http_port 3129 intercept
```

Cisco IOS example with WCCP:

```text
! Cisco IOS: redirect web traffic arriving on the LAN port
ip wccp web-cache
!
interface GigabitEthernet0/1
 description LAN
 ip wccp web-cache redirect in
```

The steps:

1. The client opens TCP to 203.0.113.80:80. It sends a normal request with a short path.
2. The gateway redirects the packets. The proxy accepts the connection.
3. The proxy finds the real target from the `Host` header. On Linux it can also ask the kernel for the original destination with the `SO_ORIGINAL_DST` socket option.
4. The proxy fetches the page and returns it, using the website's address as its source.

**Explicit vs transparent proxy:**

| Point | Explicit | Transparent |
|---|---|---|
| Client setup | Needed | None |
| Request sent to | The proxy's address | The real website's address |
| Request line | Absolute URL or CONNECT | Short path |
| User login | Can use 407 | Cannot use 407, the client does not know of a proxy |
| HTTPS | CONNECT keeps it simple | Needs SNI peek or TLS interception |
| Good for | Managed laptops | Guest Wi-Fi, IoT, devices you cannot configure |

Note on words: older HTTP standards used "transparent proxy" for a proxy that does not change requests. Today the word almost always means an intercepting proxy, and that is what an interviewer means too.

HTTPS does not work with port 80 tricks. A transparent proxy on port 443 has two choices. It can read the SNI, apply a rule, and copy bytes without decrypting. Or it can decrypt, which needs a trusted certificate on the device. QUIC (HTTP/3 on UDP 443) also avoids TCP interception, so many companies block UDP 443 to force browsers back to TCP. See [QUIC](../06-transport-layer/08-quic.md).

### The lost client IP: X-Forwarded-For

Because a proxy makes a second connection, the backend sees the proxy's IP. Logs, rate limits, and geo rules all break. HTTP proxies fix this with a header.

`X-Forwarded-For` (XFF) lists the addresses a request has passed through. Each proxy **appends the IP address of whoever connected to it**:

```text
X-Forwarded-For: <client>, <proxy1>, <proxy2>
```

Note that the last proxy's own address is not in the list. The next hop sees that address as the TCP source.

XFF is a common custom, not a formal standard. The standard version is the `Forwarded` header (RFC 7239), but XFF is still far more widely used:

```text
Forwarded: for=198.51.100.25;proto=https;host=app.example.com
```

Two related headers are common:

- `X-Forwarded-Proto`: the protocol the client used, such as `https`. The app needs it to build correct links.
- `X-Forwarded-Host`: the host name the client asked for.

**The security trap:** a client can send any headers it likes. So the left side of the list may be false. Only values added by your own proxies can be trusted.

Worked chain. The client at 198.51.100.25 sends a false XFF value to pretend it is 192.0.2.66.

```text
 Client ---> CDN edge ---> Load balancer ---> NGINX ---> App
 198.51.100.25  203.0.113.50  10.0.1.5     10.0.2.10
```

| Hop | TCP source seen | XFF after this hop |
|---|---|---|
| Client sends | (none) | `192.0.2.66` |
| CDN edge | 198.51.100.25 | `192.0.2.66, 198.51.100.25` |
| Load balancer | 203.0.113.50 | `192.0.2.66, 198.51.100.25, 203.0.113.50` |
| NGINX | 10.0.1.5 | `192.0.2.66, 198.51.100.25, 203.0.113.50, 10.0.1.5` |
| App | 10.0.2.10 | Reads the header |

How to find the real client:

1. Make a list of trusted proxy ranges: here 203.0.113.0/24 and 10.0.0.0/8.
2. Start at the **right end** of XFF.
3. Skip every address that is a trusted proxy.
4. The first address that is not trusted is the client: **198.51.100.25**.

Taking the left-most value would give 192.0.2.66, the false address. An attacker could then avoid rate limits or fake a trusted IP.

NGINX can do this with its real IP module (not always compiled in, but most Linux packages include it):

```text
# NGINX: trust these sources, and take the client from XFF
set_real_ip_from 203.0.113.0/24;
set_real_ip_from 10.0.0.0/8;
real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```

With `real_ip_recursive on`, NGINX does the right-to-left walk and sets `$remote_addr` to 198.51.100.25. After that, `$proxy_add_x_forwarded_for` appends this new value, not the TCP peer. The TCP peer address remains in `$realip_remote_addr`.

### PROXY protocol

XFF is an HTTP header, so it has two limits:

- It does not work for non-HTTP traffic, such as SMTP, database, or raw TCP.
- With TLS passthrough, the proxy cannot add a header, because the data is encrypted.

The **PROXY protocol**, created by the HAProxy project, solves this at the TCP level. The sender adds a small header **at the very start of the TCP connection**, before any application data. This is a published specification from HAProxy, not an IETF standard, but it is widely supported.

**Analogy:** XFF is a note written inside a letter. PROXY protocol is a slip stapled to the outside of the sealed envelope, so nobody has to open it.

```text
 Client                L4 load balancer         NGINX
 198.51.100.25:51514   203.0.113.20:443         10.0.2.10:443
    |                       |                       |
    |== TCP, TLS hello ====>|                       |
    |                       |== TCP handshake ======|
    |                       |-- PROXY header ------>|
    |                       |-- TLS hello --------->|
```

**Version 1** is one line of text, ending in CRLF (carriage return and line feed):

```text
PROXY TCP4 198.51.100.25 203.0.113.20 51514 443\r\n
```

The fields are: protocol, source IP, destination IP, source port, destination port. The longest possible line is 107 bytes.

**Version 2** is binary, so it is faster to parse and can carry extra fields. The same connection looks like this:

| Bytes | Field | Value in hex |
|---|---|---|
| 1-12 | Fixed signature | `0D 0A 0D 0A 00 0D 0A 51 55 49 54 0A` |
| 13 | Version and command | `21` (version 2, PROXY) |
| 14 | Family and transport | `11` (IPv4, TCP) |
| 15-16 | Length of what follows | `00 0C` (12 bytes) |
| 17-20 | Source IP 198.51.100.25 | `C6 33 64 19` |
| 21-24 | Destination IP 203.0.113.20 | `CB 00 71 14` |
| 25-26 | Source port 51514 | `C9 3A` |
| 27-28 | Destination port 443 | `01 BB` |

Version 2 can add extra TLV items (type, length, value), for example a cloud VPC endpoint ID. Command `20` means LOCAL: the connection was made by the proxy itself, such as a health check, so the address fields are ignored.

Configuration on both sides:

```text
# HAProxy: L4 sender, inside a mode tcp backend
server nginx1 10.0.2.10:443 send-proxy-v2
```

```text
# NGINX: receiver (accepts version 1 and version 2)
server {
    listen 443 ssl proxy_protocol;

    set_real_ip_from 10.0.1.0/24;     # load balancer subnet only
    real_ip_header   proxy_protocol;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
    }
}
```

Three rules matter:

1. **Both sides must agree.** The receiver must be set to expect the header. The specification says it must not try to guess. If only the sender is on, the header looks like garbage. If only the receiver is on, NGINX logs a `broken header` error.
2. **Accept it only from trusted sources.** If the port is open to the internet, anyone can send a false PROXY line. Firewall the port.
3. **Remember health checks.** A load balancer health check to that port must also speak the protocol, or the backend marks it as failed.

| Point | X-Forwarded-For | PROXY protocol |
|---|---|---|
| Layer | HTTP header | Start of TCP stream |
| Traffic | HTTP only | Any TCP (version 2 also supports UDP) |
| Works with TLS passthrough | No | Yes |
| Times sent | Per request | Once per connection |
| Who must support it | Proxy and app | Sender and receiver software |
| Shows the path of hops | Yes | No, only the original source |

### Caching proxy

A **caching proxy** keeps copies of responses and serves them without asking the backend.

**Analogy:** a local shop keeps popular items on the shelf, so it does not call the warehouse for each customer. The analogy breaks because the shop's goods do not go out of date in seconds, while web pages can.

Key words:

- **Hit:** the answer was in the cache.
- **Miss:** the proxy had to ask the backend.
- **Fresh:** a stored copy that may still be used without checking.
- **Stale:** a stored copy past its time limit.
- **Revalidate:** ask the backend "has this changed?" A `304 Not Modified` answer means the copy is still good, so the body is not sent again.

```text
 Client          Caching proxy             Backend
   |                  |                       |
   |-- GET /logo ---->|  MISS                 |
   |                  |-- GET /logo --------->|
   |                  |<-- 200, max-age=3600 -|
   |<-- 200 ----------|  store for 1 hour     |
   |                  |                       |
   |-- GET /logo ---->|  HIT, no backend call |
   |<-- 200, Age: 60 -|                       |
```

Caching follows RFC 9111. The main response headers are:

| Header | Meaning |
|---|---|
| `Cache-Control: max-age=3600` | Fresh for 3600 seconds |
| `Cache-Control: s-maxage=600` | Shared caches (proxies) use 600 instead |
| `Cache-Control: no-store` | Never store this |
| `Cache-Control: no-cache` | May store, but must revalidate before each use |
| `Cache-Control: private` | One user only, so shared caches must not store it |
| `ETag: "abc123"` | A version label, used with `If-None-Match` to revalidate |
| `Vary: Accept-Encoding` | Keep separate copies per value of this request header |
| `Age: 60` | The copy has been in the cache for 60 seconds |

The number that matters is the **hit ratio**. If 1000 requests per second arrive with a 90% hit ratio, only 100 per second reach the backend.

**Forward vs reverse caching:** forward caching used to save company bandwidth. HTTPS changed that. A proxy cannot cache inside a CONNECT tunnel, so forward caching is now useful mainly for plain package mirrors or with TLS interception. Reverse caching is still very common, and a CDN is a global network of caching reverse proxies. See [Content Delivery Networks](06-cdn.md).

### SOCKS

**SOCKS** is a general proxy protocol that works below HTTP. It relays TCP connections, and in version 5 also UDP. It does not understand what the data means.

**Analogy:** an HTTP proxy is a translator who reads your letter. A SOCKS proxy is a courier who carries a sealed box to the address you give and does not care what is inside. The analogy breaks at one point: a courier carries one box, but SOCKS keeps a connection open for both directions.

SOCKS4 handles TCP over IPv4 only and has no password login. SOCKS4a adds name lookup at the proxy. SOCKS5 (RFC 1928) adds UDP, IPv6, and login methods, such as user and password (RFC 1929). Use SOCKS5 today.

The usual port is 1080. **SOCKS does not encrypt anything.** Privacy comes only from what you run inside it, or from the tunnel carrying it, such as SSH.

The SOCKS5 exchange to reach `www.example.com:443`:

```text
 Client                                  SOCKS5 proxy
   |--- 05 01 00 ------------------------->|  Hello: 1 method, no auth
   |<-- 05 00 -----------------------------|  Chosen: no auth
   |--- 05 01 00 03 0F <15-byte name> 01BB>|  Connect to name, port 443
   |<-- 05 00 00 01 <IPv4> <port> ---------|  Success
   |=========== raw bytes both ways =======|
```

Fields of the connect request:

| Field | Size | Example | Meaning |
|---|---|---|---|
| VER | 1 byte | `05` | Version 5 |
| CMD | 1 byte | `01` | 01 CONNECT, 02 BIND, 03 UDP ASSOCIATE |
| RSV | 1 byte | `00` | Reserved |
| ATYP | 1 byte | `03` | 01 IPv4, 03 domain name, 04 IPv6 |
| DST.ADDR | Varies | `0F` then 15 name bytes | Domain with a length prefix |
| DST.PORT | 2 bytes | `01 BB` | Port 443 |

| Point | HTTP proxy | SOCKS5 proxy |
|---|---|---|
| Understands HTTP | Yes | No |
| Traffic | HTTP, or any TCP with CONNECT | Any TCP, also UDP |
| Can filter by URL or cache | Yes, for plain HTTP | No |
| Encrypts | No | No |
| Typical use | Company web access | SSH tunnels, Tor client access, apps needing raw TCP or UDP |

### API gateway vs reverse proxy

An **API gateway** is a reverse proxy with extra features for managing APIs. An **API** (Application Programming Interface) is a way for programs to call each other. So every API gateway is a reverse proxy, but a plain reverse proxy is not an API gateway.

**Analogy:** a reverse proxy is the reception desk that sends visitors to the right room. An API gateway is reception plus a security office: it checks your badge, knows your visit plan, counts your visits, and may translate for you. The analogy stops being true at scale, because the gateway handles thousands of requests per second automatically.

| Feature | Reverse proxy | API gateway |
|---|---|---|
| TLS termination | Yes | Yes |
| Routing and load balancing | Yes | Yes |
| Basic rate limits | Yes, often per IP | Yes, per user, key, or plan |
| Authentication | Basic support | Core feature: API keys, JWT, OAuth 2.0 |
| Quotas and billing plans | Rare | Common |
| Request and response transformation | Limited | Common, such as REST to gRPC |
| API versioning | By manual rules | Built in |
| Developer portal, API analytics | No | Common |

A **JWT** (JSON Web Token) is a signed token that proves who is calling. **OAuth 2.0** is a standard way to give apps limited access.

Products often overlap. Kong is built on NGINX, and Envoy is used in several gateways. Cloud examples include Amazon API Gateway and Apigee. In a real system, you may have a CDN, then a load balancer, then an API gateway, then a service mesh inside. See [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md) and [Kubernetes Ingress and Network Policies](../11-cloud-and-containers/09-kubernetes-ingress-and-policies.md).

### Proxy vs similar devices

A router forwards packets. NAT rewrites packets in the same connection. A VPN encrypts all IP traffic in a tunnel. An L7 proxy runs two connections for one application protocol. An L7 load balancer is a reverse proxy with several backends.

See [Firewalls](../09-security/04-firewalls.md) and [VPNs](../09-security/07-vpn.md).

## Example

### Example 1: NGINX reverse proxy with TLS, caching, and rate limits

The setup:

```text
 Client 198.51.100.25
        |
        | HTTPS to app.example.com (203.0.113.10:443)
        v
 +---------------------------+
 | NGINX                     |
 | public  203.0.113.10      |
 | private 10.0.2.10         |
 +---------------------------+
        | HTTP
        +---------> 10.0.2.21:8080
        +---------> 10.0.2.22:8080
```

```text
# /etc/nginx/conf.d/app.conf (Linux, NGINX open source)
# This file is loaded inside the http { } block.

proxy_cache_path /var/cache/nginx/app levels=1:2
                 keys_zone=app_cache:10m max_size=1g
                 inactive=60m use_temp_path=off;

limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;

upstream app_backend {
    least_conn;
    server 10.0.2.21:8080 max_fails=3 fail_timeout=10s;
    server 10.0.2.22:8080 max_fails=3 fail_timeout=10s;
    keepalive 32;
}

server {
    listen 443 ssl;
    http2 on;
    server_name app.example.com;

    ssl_certificate     /etc/nginx/tls/app.example.com.fullchain.pem;
    ssl_certificate_key /etc/nginx/tls/app.example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    client_max_body_size 10m;

    proxy_http_version 1.1;
    proxy_connect_timeout 5s;
    proxy_read_timeout    30s;

    proxy_set_header Connection        "";
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        proxy_pass http://app_backend;
    }

    location /static/ {
        proxy_pass http://app_backend;
        proxy_cache app_cache;
        proxy_cache_valid 200 10m;
        proxy_cache_lock on;
        proxy_cache_use_stale error timeout updating http_502 http_503;
        add_header X-Cache-Status $upstream_cache_status;
    }

    location /api/ {
        limit_req zone=per_ip burst=20 nodelay;
        proxy_pass http://app_backend;
    }

    location /ws/ {
        proxy_pass http://app_backend;
        proxy_read_timeout 1h;

        # A proxy_set_header here cancels ALL the ones above,
        # so repeat what the backend needs.
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

What the main lines do:

- `http2 on` turns on HTTP/2 for clients. This form needs NGINX 1.25.1 or later. Older versions use `listen 443 ssl http2;`.
- `upstream` names the backend pool. `least_conn` picks the backend with the fewest active connections.
- `max_fails=3 fail_timeout=10s` stops using a backend for 10 seconds after 3 failures within 10 seconds.
- `keepalive 32` reuses backend connections. The `Connection ""` line clears the default `Connection: close` value, so keepalive can work.
- `proxy_http_version 1.1` is the default from NGINX 1.29.7, which also turned on upstream keepalive by default. Before that the default was HTTP/1.0, so setting these lines is safe on any version.
- `Host $host` passes the name the client asked for. Without it, NGINX sends `app_backend`, the name of the upstream.
- `proxy_cache_lock on` lets one request fill a missing cache entry while others wait. This avoids a rush of requests to the backend.
- `limit_req` allows 10 requests per second per IP, with a burst of 20. Extra requests get 503 by default.
- The `/ws/` block supports WebSocket, a two-way connection that starts as an HTTP request with `Upgrade`. It needs a long read timeout.

NGINX default timeouts are 60 seconds for both connect and read. The config above makes the connect timeout short, so a dead backend fails quickly.

Test and load it:

```bash
# Linux
sudo nginx -t
sudo nginx -s reload
```

See the two TCP connections on the proxy with `ss`:

```bash
# Linux, on the NGINX server
ss -tn state established '( sport = :443 or dport = :8080 )'
```

```text
Recv-Q Send-Q   Local Address:Port      Peer Address:Port
0      0         203.0.113.10:443      198.51.100.25:51514
0      0            10.0.2.10:41822        10.0.2.21:8080
```

Line 1 is the client connection. Line 2 is the backend connection, with a source port chosen by NGINX.

Check the cache. The backend sends `Cache-Control: public, max-age=3600` here:

```bash
# Linux
URL=https://app.example.com/static/logo.png
curl -sI $URL | grep -iE '^(HTTP|cache-control|x-cache)'
curl -sI $URL | grep -iE '^(HTTP|cache-control|x-cache)'
```

```text
HTTP/2 200
cache-control: public, max-age=3600
x-cache-status: MISS

HTTP/2 200
cache-control: public, max-age=3600
x-cache-status: HIT
```

The backend's cache header wins over `proxy_cache_valid`, so the copy stays fresh for one hour, not 10 minutes.

The backend at 10.0.2.21 now receives this:

```text
TCP source:        10.0.2.10
GET /api/orders HTTP/1.1
Host: app.example.com
X-Real-IP: 198.51.100.25
X-Forwarded-For: 198.51.100.25
X-Forwarded-Proto: https
```

The app must read these headers **only** when the TCP source is the proxy, 10.0.2.10.

### Example 2: Squid forward proxy and CONNECT

A minimal Squid setup on 10.0.3.5:

```text
# /etc/squid/squid.conf (Squid on Linux)
acl localnet src 10.0.3.0/24
acl SSL_ports port 443
acl Safe_ports port 80 443
acl CONNECT method CONNECT
acl blocked dstdomain .blocked.example.com

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access deny blocked
http_access allow localnet
http_access deny all

http_port 3128
```

The rules are checked from top to bottom, and the first match wins. The `deny CONNECT !SSL_ports` line stops tunnels to any port but 443.

From a client, trimmed output:

```bash
# Linux
curl -v -x http://10.0.3.5:3128 https://www.example.com/ -o /dev/null
```

```powershell
# Windows: curl.exe avoids the older PowerShell alias for curl
curl.exe -v -x http://10.0.3.5:3128 https://www.example.com/ -o NUL
```

```text
* Connected to 10.0.3.5 (10.0.3.5) port 3128
* Establish HTTP proxy tunnel to www.example.com:443
> CONNECT www.example.com:443 HTTP/1.1
> Host: www.example.com:443
> Proxy-Connection: Keep-Alive
>
< HTTP/1.1 200 Connection established
<
* CONNECT phase completed
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=www.example.com
> GET / HTTP/2
< HTTP/2 200
```

Notice the order: the CONNECT is in clear text, and the `GET` is inside TLS. Squid's access log has one line for the whole tunnel (wrapped here to fit):

```text
1761820000.123   1240 10.0.3.25 TCP_TUNNEL/200 5320 CONNECT
  www.example.com:443 - HIER_DIRECT/203.0.113.80 -
```

The fields are time, duration in milliseconds, client, result, bytes, method, target, user, and how the target was reached. No URL path appears, because the proxy never saw it.

A request to `www.blocked.example.com` through this proxy returns `403`.

Set a proxy for all tools:

```bash
# Linux
export http_proxy=http://10.0.3.5:3128
export https_proxy=http://10.0.3.5:3128
export no_proxy=localhost,127.0.0.1,.corp.example.com
```

```powershell
# Windows: for this PowerShell session
$env:HTTP_PROXY = "http://10.0.3.5:3128"
$env:HTTPS_PROXY = "http://10.0.3.5:3128"
$env:NO_PROXY = "localhost,127.0.0.1,.corp.example.com"

# Windows: system-wide WinHTTP proxy (needs an admin shell)
netsh winhttp set proxy proxy-server="10.0.3.5:3128"
netsh winhttp show proxy
```

WinHTTP settings are used mainly by Windows services. The proxy set in Windows Settings is separate, and it is what most user apps follow.

### Example 3: a quick SOCKS proxy with SSH

OpenSSH can act as a SOCKS proxy. The commands are the same on Linux and Windows 11.

```bash
# Linux or Windows: open local SOCKS port 1080, tunnel it over SSH
ssh -N -D 127.0.0.1:1080 admin@jump.example.com
```

In a second terminal:

```bash
# Linux: the proxy resolves the name, so local DNS sees nothing
curl --socks5-hostname 127.0.0.1:1080 https://ip.example.com/
```

```text
203.0.113.7
```

The site sees 203.0.113.7, the jump host's address. The hop from curl to the jump host is encrypted by SSH. The hop from the jump host to the site is not, unless the site uses TLS.

`--socks5` resolves the name on the client, and `--socks5-hostname` resolves it at the proxy. The second one prevents a **DNS leak**, which is when local DNS servers see the names you visit even though the traffic is tunnelled.

## Going Deeper

### TLS at a reverse proxy: three modes

| Mode | Client to proxy | Proxy to backend | Proxy reads HTTP? | Use |
|---|---|---|---|---|
| Termination | TLS | Plain HTTP | Yes | Trusted private network |
| Re-encryption | TLS | New TLS | Yes | Zero trust, compliance |
| Passthrough | TLS | Same TLS, untouched | No | Backend must own the key |

In passthrough, the proxy can still route by SNI, because the name is in the first TLS message:

```text
# NGINX stream module (L4), outside the http { } block
stream {
    map $ssl_preread_server_name $backend {
        app.example.com   10.0.2.21:8443;
        pay.example.com   10.0.2.31:8443;
        default           10.0.2.21:8443;
    }

    server {
        listen 443;
        ssl_preread on;
        proxy_pass $backend;
        proxy_protocol on;     # send the client IP as a PROXY header
    }
}
```

**Mutual TLS (mTLS)** means the client also shows a certificate. With termination, the proxy checks it and passes the identity in a header. Backends must only trust that header from the proxy. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

### TLS interception

A forward proxy can see inside HTTPS by acting as a **man in the middle with permission**. The company installs its own Certificate Authority (CA) on all managed devices. Squid calls this feature SSL Bump.

```text
 Client ===== TLS 1 =====> Proxy ===== TLS 2 =====> Website
         fake cert for             real cert for
         www.example.com,          www.example.com
         signed by company CA
```

Costs and limits:

- The proxy must validate the real certificate, because the client can no longer do so.
- Apps with **certificate pinning** (a built-in list of accepted certificates or keys) will fail, so they are placed on a bypass list.
- Banking and health sites are often excluded for legal and privacy reasons.
- The proxy holds decrypted data and a powerful CA key. It becomes a prime target.
- Encrypted Client Hello hides SNI from passive observers, so SNI-only filters lose visibility. An intercepting proxy still sees the name, because it ends the TLS session.

### Hop-by-hop vs end-to-end headers

A proxy must treat two header kinds differently:

- **End-to-end** headers, such as `Cache-Control`, pass through to the final receiver.
- **Hop-by-hop** headers apply only to one connection, and each proxy removes them: `Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`, and any header named in `Connection`.

This is why NGINX needs explicit `Upgrade` and `Connection` lines for WebSocket. The `Upgrade` header stops at the first hop unless the proxy passes it on deliberately.

The `Via` header (RFC 9110) records protocol versions and proxies on the path, for example `Via: 1.1 proxy.corp.example.com (squid/6.10)`. It is useful for spotting a hidden proxy and for detecting loops.

### Port exhaustion at the proxy

A proxy opens many connections to the same backend IP and port. Each connection needs a unique tuple of source IP, source port, destination IP, and destination port. So with one source IP and one backend, the source port is the only variable.

Linux's usual default range is 32768-60999, which is 28,232 ports. Closed connections also stay in TIME_WAIT for about 60 seconds. See [TCP Connection States](../06-transport-layer/04-tcp-states.md). At 1000 new connections per second with no reuse, the ports run out in about 28 seconds.

Fixes:

- Reuse backend connections with keepalive. This is the best fix.
- Add source IPs, for example with `proxy_bind`.
- Add more backend IP and port pairs.
- Widen `net.ipv4.ip_local_port_range`.

### Buffering, streaming, and retries

By default NGINX buffers the response, and also the request body. This protects app workers from slow clients. But it adds delay for live streams, such as Server-Sent Events (a one-way live feed over HTTP). Use `proxy_buffering off` for those paths. `proxy_read_timeout` applies between two reads, not to the whole response, so a long stream is fine if it keeps sending data.

If a backend fails, a proxy may retry on another. NGINX's default is `proxy_next_upstream error timeout`. It does not retry non-idempotent methods such as POST once the request has been sent, because a repeat could charge a card twice. **Idempotent** means doing it twice has the same result as once.

### HTTP request smuggling

A proxy reuses backend connections for many users. So the proxy and backend must agree on where each request ends. HTTP/1.1 has two ways to state a body length: `Content-Length` and `Transfer-Encoding: chunked`.

If they disagree, an attacker can hide a second request inside the first one. The backend then treats the hidden part as the start of the next request, which may belong to another user.

Defences:

- Reject requests with both or malformed length headers.
- Keep proxy and backend software patched.
- Use HTTP/2 to the backend where possible, because it has built-in message framing. Downgrading HTTP/2 to HTTP/1.1 at the proxy can bring back the risk.

### Cache poisoning and cache deception

- **Cache poisoning:** an attacker uses a header that changes the response but is not part of the cache key. The bad response is stored and served to everyone. Fix: put such inputs in the key, or strip them.
- **Cache deception:** an attacker tricks a victim into loading `/account/profile.css`. The app ignores the suffix and returns private data, and the cache stores it as a static file. Fix: cache based on `Cache-Control` and strict paths, not only file extensions.

NGINX's default cache key is `$scheme$proxy_host$request_uri`. It does not include cookies, so it refuses to store responses with `Set-Cookie` by default. Never ignore that rule without care.

### Modern tunnelling

CONNECT now exists in newer HTTP versions. In HTTP/2 and HTTP/3, a tunnel is one stream on a shared connection, not a whole TCP connection. See [HTTP/1.1, HTTP/2 and HTTP/3](../07-application-protocols/06-http-versions.md).

The **MASQUE** work at the IETF extends this idea:

- CONNECT-UDP (RFC 9298) tunnels UDP, so QUIC can pass through an HTTP proxy.
- CONNECT-IP (RFC 9484) tunnels IP packets.

These make a proxy behave more like a VPN, and they look like ordinary HTTPS on the wire.

### Proxy abuse

- **Open proxy:** a proxy that accepts anyone on the internet. Attackers use it to hide. Always limit by source and authenticate.
- **SSRF** (Server-Side Request Forgery): a reverse proxy or app is tricked into calling an internal address, such as a cloud metadata service. Never build a `proxy_pass` target from user input without an allow list.

## Common Mistakes

- "A reverse proxy is just a backwards forward proxy." The real difference is who it serves. The mechanics of two connections are the same.
- Thinking a proxy works like NAT. NAT rewrites packets in one connection. An L7 proxy runs two separate connections.
- Believing a forward proxy can read HTTPS by default. With CONNECT it only sees host, port, timing, and byte counts.
- Taking the left-most X-Forwarded-For value as the client IP, or trusting the header from any source. Clients can forge it, so accept it only from known proxies.
- Turning on the PROXY protocol on only one side. The connection then breaks, usually with a `broken header` error or a TLS failure.
- Leaving a PROXY protocol port open to the internet, which allows false client addresses.
- Assuming SOCKS encrypts traffic. It does not.
- Thinking a reverse proxy replaces a firewall. It hides servers and limits exposure, but it does not filter all traffic.
- Forgetting `proxy_set_header` inheritance in NGINX. One such line in a `location` cancels all lines inherited from `server`.
- Mixing up `proxy_pass` paths. With `location /api/`, `proxy_pass http://app;` keeps `/api/users`, while `proxy_pass http://app/;` sends `/users`.
- Using a host name in `proxy_pass` and expecting DNS changes to apply. Open source NGINX normally resolves it at start or reload, unless you use a `resolver` with a variable, or the `resolve` option in an upstream on versions that support it.
- Expecting NGINX to work as a forward HTTPS proxy. Open source NGINX has no CONNECT support without a third-party module. Use Squid for this.
- Caching responses with cookies or per-user data, which can show one user's data to another.
- Setting a proxy read timeout shorter than the app's normal work time. The client gets a 504 even though the backend finishes the job.

## Interview Questions

**Q: What is the difference between a forward proxy and a reverse proxy?**
A: A forward proxy works for clients. It sits near them, hides them, and controls their access to many outside servers. A reverse proxy works for servers. It sits in front of them, hides them, and takes in requests from many clients, adding TLS, load balancing, and caching.

**Q: How does HTTPS work through a forward proxy?**
A: The client sends a clear text `CONNECT host:443` request. The proxy checks policy, opens TCP to that host, and replies 200. After that, it just copies bytes. The client runs TLS straight to the website, so the proxy sees the host name and port, but not the URL or content.

**Q: What is a transparent proxy, and how is it different from an explicit proxy?**
A: A transparent proxy intercepts traffic without any client setting, using iptables redirects, policy routing, or WCCP. An explicit proxy is configured on the client, which sends it absolute URLs or CONNECT. Transparent proxies cannot use a 407 login, and HTTPS needs SNI inspection or TLS interception.

**Q: A backend only logs the load balancer's IP. How do you get the real client IP?**
A: For HTTP, have the proxy add X-Forwarded-For, then read it at the backend, trusting only known proxies. Walk from the right and skip trusted addresses. For non-HTTP traffic or TLS passthrough, use the PROXY protocol. Some load balancers can also preserve the source IP at L4.

**Q: Why can X-Forwarded-For not be trusted blindly?**
A: Clients can send it themselves, so the left values may be false. Each proxy only appends the peer it saw. You can trust only the entries added by your own proxies, so you read from the right and stop at the first address that is not a trusted proxy.

**Q: What is the PROXY protocol, and when would you use it instead of X-Forwarded-For?**
A: It is a small header sent once at the start of a TCP connection, giving the original source and destination addresses and ports. I would use it for non-HTTP TCP services or TLS passthrough, where a proxy cannot add an HTTP header. Both sides must enable it, and the receiver must accept it only from trusted sources.

**Q: What is the difference between an API gateway and a reverse proxy?**
A: An API gateway is a reverse proxy with API management features. A reverse proxy does TLS, routing, load balancing, and caching. A gateway adds API keys, JWT or OAuth checks, per-user quotas, request transformation, versioning, and analytics. Every gateway is a reverse proxy, but not the other way round.

**Q: How does a SOCKS proxy differ from an HTTP proxy?**
A: SOCKS works below the application. It relays TCP, and in version 5 also UDP, without understanding the data. An HTTP proxy understands HTTP, so it can filter URLs, add headers, and cache. Neither encrypts by itself. `ssh -D` is a common way to create a SOCKS proxy with SSH encryption.

**Q: Users get 502 errors from NGINX, and sometimes 504. What do you check?**
A: 502 means NGINX got a bad answer or no connection, so I check that the app is running, the port is right, and the error log for "connection refused" or a reset. 504 means the backend was too slow, so I check slow queries, `proxy_read_timeout`, and any firewall dropping packets between them.

**Q: How does a caching proxy decide what to store?**
A: It follows HTTP caching rules from RFC 9111. It looks at the method, usually GET or HEAD, and at status and headers such as `Cache-Control`, `Expires`, and `Vary`. `no-store` and `private` block shared caching. It serves fresh copies, and revalidates stale ones with `ETag`, getting a 304 if unchanged.

**Q: Why might a proxy run out of source ports, and how do you fix it?**
A: One proxy IP talking to one backend IP and port has only the ephemeral port range, about 28,000 ports on default Linux. Without reuse, new connections and TIME_WAIT use them all up. The best fix is backend keepalive. Adding source IPs or backends also helps.

**Q: How can a company inspect HTTPS traffic, and what are the downsides?**
A: It installs a company CA on managed devices. The proxy terminates TLS from the client with a generated certificate, then opens its own TLS to the site. Downsides are broken certificate pinning, privacy and legal issues, extra CPU load, and a new high-value target holding decrypted data.

## Quick Recap

- A proxy runs two connections and speaks for each side. Forward proxies serve clients, and reverse proxies serve servers.
- HTTPS passes through a forward proxy with a CONNECT tunnel. The proxy sees the host and port, not the content, unless it uses TLS interception.
- A transparent proxy intercepts traffic with no client setting, using iptables, policy routing, or WCCP.
- X-Forwarded-For carries the client IP in HTTP. Read it from the right, and trust only your own proxies.
- PROXY protocol carries the client address at the start of a TCP connection. It works for any TCP service and for TLS passthrough, but both sides must enable it.
- An API gateway is a reverse proxy plus auth, quotas, transformation, and analytics. A CDN is a global caching reverse proxy.
- SOCKS relays raw TCP and UDP without encryption. NGINX is a strong reverse proxy, but it is not a CONNECT forward proxy without a third-party module.
