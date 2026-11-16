# HTTP/1.1, HTTP/2 and HTTP/3

**Level:** Intermediate | **Prerequisites:** [HTTP Basics](05-http-basics.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md)

## In One Line

HTTP versions keep the same meaning (methods, status codes, headers) but change how bytes move: HTTP/1.1 sends text, one exchange at a time per connection. HTTP/2 mixes many exchanges on one TCP connection. HTTP/3 moves to QUIC over UDP, so one lost packet no longer stalls everything.

## Why It Matters

A modern web page needs many files: HTML, CSS, JavaScript, images, fonts and API calls. Each request costs at least one round trip. A **round trip time (RTT)** is the time for a message to reach the other side and for the reply to come back. On a mobile network, 100 ms is common.

If files load one at a time, a page of 30 files takes 30 RTT, which is 3 seconds of pure waiting. Buying more bandwidth does not fix this. Every HTTP version since 1.0 tries to reduce that waiting.

Interviewers ask about this topic because it joins several layers. A good answer connects application behaviour (HTTP) to transport behaviour (TCP and QUIC). The classic questions are:

- What is head-of-line blocking?
- Why did HTTP/3 move to UDP?
- If HTTP/2 has multiplexing, why was HTTP/3 needed?

## Simple Explanation

Think of a shop with a clerk behind a counter. You need several items.

| Version | Shop story |
|---|---|
| HTTP/1.0 | You walk to the shop, buy one item, walk home, then walk back for the next item. |
| HTTP/1.1 persistent | You stay at the counter, but ask for one item at a time. |
| HTTP/1.1 pipelining | You hand over a list of three items. The clerk must give them in list order, so a slow first item holds up the two easy ones. |
| HTTP/2 | The clerk hands items over as they are ready, each with an order number. But everything passes through one narrow door in single file. If one person trips, everyone behind waits. |
| HTTP/3 | The shop has many doors. A person who trips at one door only holds up that door. |

The door is the transport. TCP is the single door: it delivers bytes strictly in order. QUIC is the many doors: each stream has its own order.

**Where the analogy stops being true:** the doors are not separate network paths. All QUIC streams use the same UDP path and share one speed limit (congestion control). A lost packet is also not a person falling over. It is data that must be sent again, and the wait is usually about one RTT.

## How It Works

### The Layers Under Each Version

```text
  HTTP/1.1         HTTP/2           HTTP/3
+----------+     +----------+     +----------+
| HTTP     |     | HTTP     |     | HTTP     |  same methods,
| as text  |     | binary   |     | binary   |  status codes,
|          |     | frames   |     | frames   |  header meaning
+----------+     +----------+     +----------+
| TLS      |     | TLS      |     | QUIC     |
| optional |     | (browser |     | with TLS |
|          |     | required)|     | 1.3 built|
+----------+     +----------+     | inside   |
| TCP      |     | TCP      |     +----------+
|          |     |          |     | UDP      |
+----------+     +----------+     +----------+
| IP       |     | IP       |     | IP       |
+----------+     +----------+     +----------+
```

**TCP** is a transport protocol that delivers a reliable, ordered stream of bytes. **UDP** just sends separate packets, with no ordering and no resend. **TLS** adds encryption, covered in [TLS and HTTPS](07-tls-and-https.md). **QUIC** is a newer transport built on UDP, covered in [QUIC](../06-transport-layer/08-quic.md).

Today the HTTP standards reflect this split. RFC 9110 defines the meaning of HTTP for all versions. Then each version has its own wire format: RFC 9112 for HTTP/1.1, RFC 9113 for HTTP/2 and RFC 9114 for HTTP/3.

### Evolution From 0.9 to 3

| Version | Year | Standard | What it added |
|---|---|---|---|
| HTTP/0.9 | 1991 | Short description, no RFC | Only `GET`. No headers, no status codes. Reply is raw HTML. |
| HTTP/1.0 | 1996 | RFC 1945 | Headers, status codes, `Content-Type`, `POST`, `HEAD` |
| HTTP/1.1 | 1997 | RFC 2068, then 2616 (1999), 7230-7235 (2014), now 9110, 9111, 9112 (2022) | Persistent connections by default, required `Host` header, chunked encoding, pipelining, better caching |
| SPDY | 2009 | Google experiment, never a standard | Binary framing and multiplexing, tested at web scale |
| HTTP/2 | 2015 | RFC 7540, now RFC 9113 (2022) with HPACK in RFC 7541 | Binary frames, streams, multiplexing, header compression |
| HTTP/3 | 2022 | RFC 9114 with QPACK in RFC 9204 | HTTP over QUIC (RFC 9000, 2021), no TCP head-of-line blocking |

HTTP/0.9 was so small that a whole request was one line:

```text
GET /index.html
```

The server sent the HTML and closed the connection. There was no way to say "not found" in a machine-readable form, and no way to send an image with a type label.

HTTP/1.0 added this shape, which you still see today:

```text
GET /index.html HTTP/1.0
User-Agent: demo/1.0

HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 137

<html>...
```

### Persistent Connections (HTTP/1.1)

In HTTP/1.0, each file normally needed a new TCP connection. Each connection costs one RTT for the TCP handshake before any request can go. With TLS, it costs more.

```text
HTTP/1.0: new connection for each file

Client                                Server
  |---- SYN ----------------------------->|
  |<--- SYN-ACK --------------------------|
  |---- ACK + GET /a.css ---------------->|
  |<--- 200 OK + a.css -------------------|
  |<--- connection closed --------------->|
  |---- SYN ----------------------------->|  start again
  |<--- SYN-ACK --------------------------|  for b.js
  |---- ACK + GET /b.js ----------------->|
  |<--- 200 OK + b.js --------------------|


HTTP/1.1: one connection, many files in sequence

Client                                Server
  |---- SYN ----------------------------->|
  |<--- SYN-ACK --------------------------|
  |---- ACK + GET /a.css ---------------->|
  |<--- 200 OK + a.css -------------------|
  |---- GET /b.js ----------------------->|  no new handshake
  |<--- 200 OK + b.js --------------------|
```

A **persistent connection** (also called keep-alive) stays open after a response, so the next request can reuse it. This also helps because TCP starts slowly and speeds up over time. A reused connection has already sped up. See [TCP Congestion Control](../06-transport-layer/06-tcp-congestion-control.md).

The rules changed between versions:

- In HTTP/1.0, keep-alive was an unofficial extension. The client had to send `Connection: keep-alive`.
- In HTTP/1.1, persistence is the default. Either side can end it with `Connection: close`.

Servers still close idle connections to save memory. The timeout is a vendor default, not a standard. For example, nginx `keepalive_timeout` defaults to 75 seconds, and Apache `KeepAliveTimeout` defaults to 5 seconds.

A persistent connection raises a new question: where does one response end? Closing the connection no longer marks the end. HTTP/1.1 uses either `Content-Length` (the exact size) or `Transfer-Encoding: chunked` (the body comes in pieces, each with its size, ending with a zero-size piece).

### Pipelining (HTTP/1.1)

**Pipelining** means sending several requests on one connection without waiting for each response. HTTP/1.1 allows this, but the server must send responses in the same order as the requests.

```text
Without pipelining           With pipelining

C: GET /a                    C: GET /a
S: response a                C: GET /b
C: GET /b                    C: GET /c
S: response b                S: response a
C: GET /c                    S: response b
S: response c                S: response c

3 RTT                        1 RTT, if /a is fast
```

Pipelining failed in practice for three reasons:

1. Responses must stay in order, so one slow response delays the rest.
2. Some proxies and servers handled it badly. A **proxy** is a middle server that forwards web traffic. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).
3. If a connection breaks, the client may not know which requests were processed. Repeating a `POST` could charge a card twice.

Almost no modern browser uses pipelining. It remains in the standard, but treat it as history.

### Head-of-Line Blocking

**Head-of-line (HoL) blocking** happens when the first item in a queue is stuck and everything behind it must wait, even if those items are ready. This happens at two different layers. Always name the layer in an interview.

**HTTP level (HTTP/1.1).** One connection carries one response at a time, in order. A 5-second database report blocks a 1 KB icon behind it.

```text
HTTP/1.1 connection

  request order:  /report  /icon.png
  response order: [====== /report, 5 s ======][icon]
                                               ^
                        ready long ago, had to wait
```

Browsers work around this by opening several connections per hostname. Most use up to 6 per host for HTTP/1.1. This is a browser choice, not a limit in the current standard. Web developers added more tricks:

- **Domain sharding:** split files across `img1.example.com`, `img2.example.com` to get more connections.
- **Bundling:** join many JavaScript files into one.
- **Sprites:** put many icons into one image.

**TCP level (HTTP/2).** TCP gives bytes to the application strictly in order. If one segment (one TCP piece of data) is lost, later bytes wait in the kernel until the resend arrives. TCP cannot see HTTP/2 streams. To TCP, it is all one byte stream. So a loss that touches one stream stalls all streams.

```text
HTTP/2 over TCP. Each box is one TCP segment.
S1 = data for stream 1, and so on.

 Seg 1   Seg 2   Seg 3   Seg 4   Seg 5   Seg 6
 [S1 ]   [S3 ]   [S5 ]   [S1 ]   [S3 ]   [S5 ]
                   X
                  lost

 Seg 1, 2 : delivered to the browser
 Seg 3    : missing, resend takes about 1 RTT
 Seg 4, 5 : arrived, but TCP holds them, stall
 Seg 6    : arrived, held

 Streams 1 and 3 did not need Seg 3, yet they wait.
```

HTTP/3 fixes this because QUIC knows about streams. Packet 3 lost means only stream 5 waits:

```text
HTTP/3 over QUIC. Same loss.

 Pkt 1   Pkt 2   Pkt 3   Pkt 4   Pkt 5   Pkt 6
 [S1 ]   [S3 ]   [S5 ]   [S1 ]   [S3 ]   [S5 ]
                   X

 Pkt 4, 5 : delivered at once, streams 1 and 3 go on
 Pkt 6    : held for stream 5 only, until Pkt 3's
            data is sent again
```

**Fuller truth:** HTTP/3 removes blocking between streams, not all waiting. All streams still share one congestion window (the limit on how much unacknowledged data may be in flight), so a loss still slows the whole connection a little. A single QUIC packet can also carry data from more than one stream. If that packet is lost, each of those streams waits.

### HTTP/2: Binary Framing

HTTP/1.1 is text. The receiver must search for line endings to find where each header stops. HTTP/2 replaces this with a binary layer. **Binary** here means fixed fields with known sizes, not readable words.

Analogy: HTTP/1.1 sends a letter, and you read until the end. HTTP/2 cuts each letter into numbered postcards. Postcards from different letters can share one mailbag, and the number tells you how to rebuild each letter. The analogy stops being true in one way: postcards can arrive in any order, but HTTP/2 frames on one stream arrive in order, because TCP keeps them in order.

Three words form the core of HTTP/2:

- **Frame:** the smallest unit sent on the wire. It has a 9-byte header plus a payload.
- **Message:** a complete request or response, made of one or more frames.
- **Stream:** one two-way exchange, carrying a request and its response. Every stream has a number.

The frame header layout:

```text
Byte: 0   1   2     3      4      5   6   7   8     9...
    +-----------+------+------+---------------+----------+
    | Length    | Type | Flags| R + Stream ID | Payload  |
    | 24 bits   | 8    | 8    | 1 + 31 bits   | variable |
    +-----------+------+------+---------------+----------+
```

| Field | Size | Meaning |
|---|---|---|
| Length | 24 bits | Payload size, not counting the 9-byte header |
| Type | 8 bits | What the frame is |
| Flags | 8 bits | Extra signals, such as END_STREAM |
| R | 1 bit | Reserved, always 0 |
| Stream ID | 31 bits | Which stream it belongs to. 0 means the whole connection |

The default maximum payload is 16,384 bytes. A receiver can raise it, up to 16,777,215 bytes.

| Type | Name | Purpose |
|---|---|---|
| 0x0 | DATA | Body bytes |
| 0x1 | HEADERS | Compressed headers, opens a stream |
| 0x2 | PRIORITY | Old priority signal, deprecated in RFC 9113 |
| 0x3 | RST_STREAM | Cancel one stream |
| 0x4 | SETTINGS | Set connection options |
| 0x5 | PUSH_PROMISE | Announce a server push |
| 0x6 | PING | Measure RTT, check the connection is alive |
| 0x7 | GOAWAY | Stop accepting new streams, shut down cleanly |
| 0x8 | WINDOW_UPDATE | Flow control credit |
| 0x9 | CONTINUATION | More header bytes if HEADERS was too small |

HTTP/2 also changes how headers look. The request line becomes **pseudo-headers**, which start with a colon: `:method`, `:scheme`, `:authority` (replaces `Host`) and `:path`. A response has `:status`. All header names must be lowercase. Headers tied to one connection, such as `Connection` and `Transfer-Encoding`, are not allowed, and chunked encoding is gone because frames already mark the boundaries.

### HTTP/2: Streams and Multiplexing

**Multiplexing** means many conversations share one connection at the same time. In HTTP/2, frames from different streams can be mixed:

```text
One TCP connection:

 [H:1][H:3][H:5][D:5][D:1][D:3][D:1][D:5 end][D:1 end][D:3 end]

 H = HEADERS frame, D = DATA frame, number = stream ID
```

Stream rules to remember:

- Clients open odd-numbered streams: 1, 3, 5. Servers use even numbers for push.
- Stream 0 is for the connection itself, such as SETTINGS and PING.
- Stream numbers only go up and are never reused. When they run out, the client opens a new connection.
- `SETTINGS_MAX_CONCURRENT_STREAMS` limits open streams. The spec recommends at least 100. nginx defaults to 128. Values vary by server.
- `RST_STREAM` cancels one stream without closing the connection. In HTTP/1.1, stopping a large download mid-way usually means closing the connection.

The result is that a browser normally needs just one HTTP/2 connection per origin. **Origin** means the scheme, host and port together, such as `https://www.example.com:443`. Tricks like domain sharding now hurt, because they split traffic back into more connections.

**Flow control** stops a fast sender from overrunning a slow receiver. HTTP/2 has it per stream and for the whole connection, on top of TCP's own flow control. Each side starts with a 65,535-byte window. A receiver gives more credit with WINDOW_UPDATE frames. Only DATA frames count against the window.

### HPACK: Header Compression

HTTP headers repeat. Every request to a site may send the same `cookie`, `user-agent` and `accept` values, often hundreds of bytes each time. A page with 100 requests repeats this 100 times. Bodies were usually compressed, but HTTP/1.1 headers never were.

HPACK (RFC 7541) is the HTTP/2 answer. It uses three tools:

1. **Static table:** 61 fixed common entries known to everyone. Entry 2 is `:method: GET`. Entry 7 is `:scheme: https`. Entry 8 is `:status: 200`.
2. **Dynamic table:** a table built during the connection. New headers are added, and later requests refer to them by a small number. The default size is 4,096 bytes, and old entries are removed when it fills up. Dynamic entries start at index 62.
3. **Huffman coding:** a fixed code that gives common characters fewer bits, to shrink literal strings.

Analogy: a regular customer at a cafe. The first time, you say your full order and the barista writes it down as "number 62". Next time you just say "62". The analogy stops being true in one way: the table is not for life. It exists only for that connection, and each direction has its own table.

**Why not just use gzip?** SPDY used DEFLATE compression on headers, and the CRIME attack (2012) showed the danger. By guessing a secret and watching how the compressed size changed, an attacker could recover cookies. HPACK avoids this kind of partial matching: it matches only whole names or whole name-value pairs. Sensitive headers can also be marked "never indexed", so they stay out of the table.

**Catch:** both sides must apply table updates in exactly the same order. That is easy on TCP, because TCP keeps order. It becomes a problem on QUIC.

### Server Push and Why Browsers Removed It

**Server push** lets the server send a file before the client asks. The idea: when a client asks for `/index.html`, the server knows it will need `/style.css`, so it saves one RTT.

Analogy: a waiter brings water before you order. That is useful, unless you already brought your own bottle. Then the water is wasted effort. The analogy stops being true because a pushed file is not free: it uses your network bandwidth.

How it worked:

1. The client sends HEADERS for `/index.html` on stream 1.
2. The server sends PUSH_PROMISE on stream 1, saying it will send `/style.css` on stream 2.
3. The server sends the response on stream 2.
4. The client can refuse with RST_STREAM, or disable push entirely with `SETTINGS_ENABLE_PUSH = 0`.

Why it failed:

- The server does not know what the browser has in its cache. It often pushed files the client already had.
- By the time the client cancelled, the bytes were often already on the way.
- Getting a real speed gain was hard, and some sites got slower.
- Few sites used it, while browsers and servers carried complex code for it.

What happened:

- Chrome turned it off by default in version 106 (2022).
- Firefox removed it in version 132 (2024).
- nginx removed HTTP/2 push support in version 1.25.1 (2023).
- It remains in RFC 9113 and RFC 9114 as a protocol feature, but do not design around it. In HTTP/3, a server can push only if the client allows it first.

The modern replacement is the status code **103 Early Hints** (RFC 8297). The server sends a quick hint before the final response, and the browser decides what to fetch. Because the browser decides, it can check its cache first.

```text
HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style

HTTP/1.1 200 OK
Content-Type: text/html
...
```

### HTTP/3 Over QUIC

HTTP/2 could not fix TCP-level blocking, because that behaviour lives inside TCP. Changing TCP is very hard for two reasons:

- TCP lives in operating system kernels (the core part of the operating system), which update slowly.
- Firewalls and other devices in the path, often called **middleboxes**, may drop anything that looks unusual. This is called **ossification**: a protocol becomes frozen because old equipment expects it to look a certain way.

A brand-new IP protocol would be blocked by many networks. UDP, however, already passes through most of them. So QUIC runs on UDP and builds reliability on top. It runs in **user space**, meaning as normal application code, not in the kernel.

QUIC gives HTTP/3 these features (details are in [QUIC](../06-transport-layer/08-quic.md)):

- **Independent streams:** loss on one stream does not stall the others.
- **Combined handshake:** transport and TLS 1.3 setup happen together in 1 RTT.
- **0-RTT resumption:** a returning client can send a request in its first flight.
- **Connection migration:** a connection is named by a connection ID, not by IP and port. A phone moving from Wi-Fi to mobile data can keep the same connection.
- **Encryption always on:** almost all of each packet is protected. There is no cleartext HTTP/3.

How HTTP maps onto QUIC:

```text
QUIC connection (UDP port 443 by default)
 |
 |-- Bidirectional streams, one per request
 |     Stream 0:  GET /      and its response
 |     Stream 4:  GET /a.css and its response
 |     Stream 8:  GET /b.js  and its response
 |
 |-- Unidirectional streams, each side opens its own
       Control stream:        SETTINGS, GOAWAY
       QPACK encoder stream:  table updates
       QPACK decoder stream:  acknowledgements
```

HTTP/3 frames are simpler than HTTP/2 frames, because QUIC already does much of the work:

| Job | HTTP/2 | HTTP/3 |
|---|---|---|
| Which stream? | Stream ID in each frame header | QUIC stream carries it, no ID in frame |
| Frame header | Fixed 9 bytes | Variable-length Type and Length |
| Flow control | WINDOW_UPDATE frame | QUIC MAX_DATA, MAX_STREAM_DATA |
| Cancel a stream | RST_STREAM frame | QUIC RESET_STREAM, STOP_SENDING |
| Keepalive | PING frame | QUIC PING frame |
| Encryption | TLS below | Built into QUIC |

### QPACK: Header Compression for HTTP/3

QPACK (RFC 9204) exists because HPACK needs strict order. QUIC streams can arrive in any order relative to each other. Suppose stream 8 refers to dynamic table entry 62, but the stream 4 HEADERS that created entry 62 was lost. The decoder cannot decode stream 8 yet. Using HPACK here would bring back the same blocking that QUIC removed.

QPACK solves this with a different design:

- A **static table of 99 entries**, numbered from 0. It is larger than HPACK's and chosen from real traffic. Entry 17 is `:method: GET`. Entry 25 is `:status: 200`.
- Dynamic table updates travel on a **dedicated one-way encoder stream**, not on the request streams.
- Each header block states the **Required Insert Count**: how far the table must have grown before this block can be decoded.
- A separate **decoder stream** tells the encoder what has been received.

Analogy: two offices share a phone book of nicknames. Updates go by a separate courier. Each letter says "you need phone book update 5 to read this". If update 5 has not come, that one letter waits, and other letters are read as normal.

The encoder has a choice:

- Use only entries the peer has confirmed. There is no blocking, but compression is lower.
- Use newer entries too. Compression is better, but a stream can briefly block.

The peer controls this limit with `SETTINGS_QPACK_BLOCKED_STREAMS`. It and `SETTINGS_QPACK_MAX_TABLE_CAPACITY` both default to 0, so the dynamic table is not used until the peer enables it.

### Finding the Version: ALPN and Alt-Svc

A client needs to learn which version a server speaks. Three tools do this.

**ALPN (Application-Layer Protocol Negotiation, RFC 7301)** is a TLS extension. During the TLS handshake, the client lists what it can speak, and the server picks one. Analogy: at a hotel desk you say "I speak English or French", and the clerk picks one before the real conversation starts. Unlike the hotel, ALPN adds no extra round trip, because it rides in messages that are sent anyway.

```text
Client                                     Server
  |--- ClientHello, ALPN: h2, http/1.1 ----->|
  |<-- TLS reply, ALPN selected: h2 ---------|
  |                                          |
  |--- "PRI * HTTP/2.0..." + SETTINGS ------>|  HTTP/2 begins
  |<-- SETTINGS -----------------------------|
```

ALPN names are short tokens: `http/1.1`, `h2`, `h3`. QUIC requires ALPN, so an HTTP/3 connection always uses the token `h3`.

**Alt-Svc (Alternative Services, RFC 7838)** solves a chicken-and-egg problem. HTTP/2 can be chosen by ALPN inside a TCP connection. But the browser must decide to try UDP before any TCP or TLS exchange, so ALPN alone cannot announce HTTP/3. Instead, a server tells the client on an existing connection:

```text
HTTP/2 200
alt-svc: h3=":443"; ma=86400
```

This means: "HTTP/3 is available on UDP port 443 of this host. Remember this for 86,400 seconds (24 hours)." `ma` means max age. Analogy: a shop sign that says "the faster entrance is around the back, valid today". `Alt-Svc: clear` removes earlier advice. A client that follows it still verifies the server's certificate, so the sign cannot send you to a fake host.

**HTTPS DNS records (RFC 9460)** give the same hint earlier. The browser sees `h3` in DNS before connecting, so it can use HTTP/3 even on the first visit. See [DNS Record Types](02-dns-records.md).

The usual flow for a first visit, without an HTTPS record:

1. The browser connects over TCP and TLS, using ALPN to pick `h2`.
2. The response includes `Alt-Svc: h3=":443"`.
3. The browser remembers this.
4. On a later connection, it tries QUIC. Browsers usually race it against TCP, so a blocked UDP port causes little delay.
5. If QUIC fails, it stays on HTTP/2. The fallback is silent.

### Comparison Table

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Current standard | RFC 9112 | RFC 9113 | RFC 9114 |
| Transport | TCP | TCP | QUIC over UDP |
| Default port | TCP 80, or TCP 443 with TLS | TCP 443 in browsers | UDP 443 |
| Wire format | Text | Binary frames | Binary frames |
| Encryption | Optional | Optional in spec, required by browsers | Always, TLS 1.3 inside QUIC |
| Parallel requests per connection | 1 at a time | Many (multiplexed) | Many (multiplexed) |
| Typical connections per origin | Up to 6 in browsers | 1 | 1 |
| HTTP-level HoL blocking | Yes | No | No |
| Transport-level HoL blocking | Yes, within each connection | Yes, across all streams | No, between streams |
| Header compression | None | HPACK | QPACK |
| Server push | No | In spec, removed in major browsers | In spec, rarely used |
| Priorities | None | Tree deprecated, RFC 9218 available | RFC 9218 |
| Cancel one request | Close the connection | RST_STREAM | RESET_STREAM, STOP_SENDING |
| New connection, TLS 1.3 | 2 RTT before request | 2 RTT before request | 1 RTT before request |
| Resumed connection | TCP 1 RTT, then early data | TCP 1 RTT, then early data | 0-RTT possible |
| Survives IP change | No | No | Yes, if both sides support it |
| How chosen | Default, or ALPN `http/1.1` | ALPN `h2` | Alt-Svc or HTTPS DNS record, then ALPN `h3` |
| Network support | Everywhere | Nearly everywhere | UDP 443 is blocked on some networks |

## Example

### Example 1: Check Which Version a Site Uses

Output below uses the documentation address 203.0.113.10 and is illustrative. Real results depend on the site and your curl version. `curl` with `-v` shows ALPN at work.

Linux or macOS:

```bash
curl -v --http2 -o /dev/null -s https://www.example.com/
```

Windows PowerShell (use `curl.exe`, because `curl` can be an alias for another command):

```powershell
curl.exe -v --http2 -o NUL -s https://www.example.com/
```

Output:

```text
* Connected to www.example.com (203.0.113.10) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN: server accepted h2
* using HTTP/2
> GET / HTTP/2
> Host: www.example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/2 200
< content-type: text/html
< alt-svc: h3=":443"; ma=86400
```

Read it in order: curl offered `h2`, the server accepted `h2`, and the response advertises HTTP/3 for later. Curl displays `> GET / HTTP/2` in a friendly form. On the wire, that request is binary frames.

A short way to print only the version:

```bash
# Linux or macOS
for v in --http1.1 --http2 --http3; do
  curl -s -o /dev/null -w "$v -> %{http_version}\n" "$v" https://www.example.com/
done
```

```text
--http1.1 -> 1.1
--http2 -> 2
--http3 -> 3
```

`--http3` needs a curl built with HTTP/3 support, and it falls back to an older version if QUIC fails. Use `--http3-only` to prove that QUIC works. Check your build with `curl -V`, and look for `HTTP3` on the `Features` line. The curl included with Windows and many Linux distributions may not have it.

Check ALPN directly with OpenSSL:

```bash
# Linux, macOS, or Windows with OpenSSL installed
openssl s_client -connect www.example.com:443 -alpn h2,http/1.1 </dev/null 2>/dev/null | grep -i "ALPN"
```

```text
ALPN protocol: h2
```

Check for an HTTPS DNS record:

```bash
dig +short HTTPS www.example.com
```

```text
1 . alpn="h3,h2" ipv4hint=203.0.113.10
```

See QUIC packets on the wire. Note that it is UDP, not TCP:

```bash
# Linux
sudo tcpdump -ni eth0 udp port 443
```

```text
10:15:02.114221 IP 10.0.0.20.51544 > 203.0.113.10.443: UDP, length 1250
10:15:02.139018 IP 203.0.113.10.443 > 10.0.0.20.51544: UDP, length 1200
```

Newer tcpdump versions may print `quic` details in place of `UDP, length`. The packet contents stay encrypted. For a quick check, open browser DevTools, go to the Network tab, and enable the **Protocol** column. It shows `http/1.1`, `h2` or `h3` for each request.

### Example 2: Why Multiplexing Saves Time

A page needs 30 small files. RTT is 100 ms. Connections are already open. Each file fits in one round trip.

| Setup | Calculation | Waiting time |
|---|---|---|
| HTTP/1.1, 1 connection | 30 rounds x 100 ms | 3,000 ms |
| HTTP/1.1, 6 connections | 30 / 6 = 5 rounds x 100 ms | 500 ms |
| HTTP/2, 1 connection | All 30 in 1 round | 100 ms |

**Fuller truth:** this ignores bandwidth and slow start. A new TCP connection usually sends about 10 segments, roughly 14 KB, in its first round. If the 30 files total 600 KB, HTTP/2 still needs several rounds. The gain is real, but not always this large.

### Example 3: Setup Time for Each Version

RTT is 50 ms, using TLS 1.3. We measure until the first response arrives.

| Case | Steps | RTT | Time |
|---|---|---|---|
| HTTP/1.1 or HTTP/2, new connection | TCP 1, TLS 1, HTTP 1 | 3 | 150 ms |
| HTTP/3, new connection | QUIC with TLS 1, HTTP 1 | 2 | 100 ms |
| HTTP/3, 0-RTT resumption | Request in first flight | 1 | 50 ms |

```text
HTTP/2 over TCP and TLS 1.3         HTTP/3 over QUIC

C ---- SYN ----------> S            C ---- Initial --------> S
C <--- SYN-ACK ------- S            C <--- Initial, ------- S
C ---- ACK ----------> S                   Handshake
C ---- ClientHello --> S            C ---- Finished ------> S
C <--- ServerHello --- S                   + request
C ---- Finished -----> S            C <--- response ------- S
       + request
C <--- response ------ S
```

With TLS 1.2, TCP needs one extra RTT. QUIC always uses TLS 1.3.

### Example 4: Read an HTTP/2 Frame Header

These 9 bytes are a frame header, in hexadecimal:

```text
00 00 1c 01 05 00 00 00 01
```

| Bytes | Field | Value |
|---|---|---|
| `00 00 1c` | Length | 28 bytes of payload |
| `01` | Type | HEADERS |
| `05` | Flags | 0x04 END_HEADERS + 0x01 END_STREAM |
| `00 00 00 01` | R and Stream ID | Stream 1 |

So this is a complete request with no body, such as a `GET`, on the first client stream.

### Example 5: HPACK Shrinks the Second Request

These bytes come from the examples in RFC 7541. The first request carries four headers:

```text
:method: GET                  82         entry 2
:scheme: http                 86         entry 6
:path: /                      84         entry 4
:authority: www.example.com   41 + 13    literal, added as entry 62
                              more bytes
Total: 17 bytes
```

The second request on the same connection adds one header:

```text
:method: GET                  82         entry 2
:scheme: http                 86         entry 6
:path: /                      84         entry 4
:authority: www.example.com   be         entry 62
cache-control: no-cache       58 + 7     literal, added to the table
                              more bytes
Total: 12 bytes, even with one extra header
```

`be` is one byte that replaces the whole hostname. In a real page, the biggest wins come from long cookies and user-agent strings.

### Example 6: Enable All Three on a Server

For nginx 1.25.1 or later, built with HTTP/3 support. Older versions use `listen 443 ssl http2`, and the exact options vary by version.

```text
server {
    listen 443 ssl;                # TCP: HTTP/1.1 and HTTP/2
    listen 443 quic reuseport;     # UDP: HTTP/3
    http2 on;

    server_name www.example.com;
    ssl_certificate     /etc/nginx/tls/fullchain.pem;
    ssl_certificate_key /etc/nginx/tls/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Tell clients that HTTP/3 exists
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

Then open UDP 443. This step is the most commonly forgotten one.

```bash
# Linux with ufw
sudo ufw allow 443/udp
```

```powershell
# Windows Server
New-NetFirewallRule -DisplayName "HTTP3 UDP 443" -Direction Inbound -Protocol UDP -LocalPort 443 -Action Allow
```

## Going Deeper

### The HTTP/2 Connection Start

After ALPN picks `h2`, the client sends a fixed 24-byte **connection preface**:

```text
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```

It looks like an HTTP/1.x request with an odd method. An old server that wrongly receives it will fail clearly and will not treat it as real HTTP/2 traffic. A SETTINGS frame follows. The server also starts with a SETTINGS frame, and each side acknowledges the other.

Key settings and their defaults:

| Setting | Default in RFC 9113 |
|---|---|
| SETTINGS_HEADER_TABLE_SIZE | 4,096 bytes |
| SETTINGS_ENABLE_PUSH | 1 (enabled) |
| SETTINGS_MAX_CONCURRENT_STREAMS | No limit (at least 100 is recommended) |
| SETTINGS_INITIAL_WINDOW_SIZE | 65,535 bytes |
| SETTINGS_MAX_FRAME_SIZE | 16,384 bytes |
| SETTINGS_MAX_HEADER_LIST_SIZE | No limit |

### HTTP/2 Without TLS: h2c

The HTTP/2 spec allows cleartext, called `h2c`. Browsers do not support it, so on the public web HTTP/2 means TLS. With TLS, HTTP/2 requires version 1.2 or later. RFC 9113 deprecated the old `Upgrade: h2c` method. Today, `h2c` is mostly used inside private networks, with **prior knowledge**: the client already knows the server speaks HTTP/2 and starts with the preface. Example: `curl --http2-prior-knowledge http://10.0.0.5:8080/`.

### Priorities

With many streams at once, the server must decide which to send first. CSS should normally beat a large image. RFC 7540 defined a dependency tree, but it was complex and implemented unevenly, so RFC 9113 deprecated it. RFC 9218 replaced it with a simple `Priority` header that works in both HTTP/2 and HTTP/3:

```text
priority: u=1, i
```

`u` is urgency, from 0 (highest) to 7 (lowest), with default 3. `i` means incremental: the resource is useful in parts, so the server can share bandwidth with others of the same urgency.

### Connection Coalescing

HTTP/2 and HTTP/3 can reuse one connection for different hostnames. A browser may do this when the certificate covers both names and, in HTTP/2 practice, both names resolve to an overlapping IP address. If the server cannot handle that hostname on this connection, it replies with **421 Misdirected Request**, and the client opens a new connection.

### When HTTP/2 Is Slower Than HTTP/1.1

On a lossy link, one TCP connection can do worse than six. A loss on one HTTP/2 connection stalls every stream, and TCP cuts the speed of all traffic. With six HTTP/1.1 connections, a loss stalls and slows only one of them. This was a main reason for HTTP/3.

### The Costs of HTTP/3

HTTP/3 is not free:

- **CPU:** a kernel TCP stack has had years of tuning and hardware help. QUIC does more work per packet in user space. Features like UDP segmentation offload reduce the gap, but high-volume servers can still pay more.
- **Blocked UDP:** some networks block or rate-limit UDP 443. Enterprises may do this on purpose, to force traffic through TLS-inspecting proxies. Clients must always keep a TCP fallback.
- **Explicit proxies:** a browser set to use a classic web proxy normally tunnels over TCP, so it does not use HTTP/3.
- **Load balancing:** a load balancer that hashes by IP and port breaks migration, because those can change. QUIC-aware balancers route by connection ID. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).
- **NAT:** home routers and firewalls often time out UDP mappings faster than TCP ones, so QUIC sends keepalives. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).
- **Amplification protection:** before validating a client's address, a QUIC server may send at most 3 times the bytes it received. Client Initial packets are padded to at least 1,200 bytes for this reason.

### 0-RTT Risk

0-RTT early data can be **replayed**: an attacker can capture it and send it again. So it should carry only requests that are safe to repeat, such as a plain `GET`. A server can reject early data, or reply **425 Too Early** (RFC 8470), so the client retries after the handshake. Never allow a payment through 0-RTT.

### Security Topics Interviewers Like

- **HTTP/2 Rapid Reset (CVE-2023-44487):** an attacker opens a stream and cancels it with RST_STREAM right away, over and over. Cancelled streams stop counting toward the concurrent stream limit, yet the server has already started work. Attacks disclosed in October 2023 reached 398 million requests per second, as reported by Google. Fixes count and limit resets.
- **CONTINUATION Flood (2024):** an attacker sends endless CONTINUATION frames with no end flag. Some servers kept buffering and ran out of memory or CPU.
- **Request smuggling:** in HTTP/1.1, the body length is text in `Content-Length` and `Transfer-Encoding`. A proxy and a backend can disagree, and an attacker can hide a second request inside the first. HTTP/2 and HTTP/3 frames carry binary lengths, which removes this problem end to end. The risk returns when a proxy converts HTTP/2 to HTTP/1.1 for the backend, called **downgrading**.

### Related Uses

- **Backends:** a common setup is HTTP/2 or HTTP/3 from browser to proxy, then HTTP/1.1 from proxy to app server. Latency inside a data centre is low, so the gains are smaller there.
- **gRPC** runs on HTTP/2 and depends on streams and trailers. Because one connection carries many calls, a Layer 4 load balancer can send all calls to one backend. See [Real-Time and Modern Protocols](14-realtime-and-modern-protocols.md).
- **WebSockets** use the `Upgrade` header in HTTP/1.1. On HTTP/2 and HTTP/3, they use extended CONNECT (RFC 8441 and RFC 9220).

## Common Mistakes

- **"HTTP/2 needs encryption."** The spec allows cleartext. Browsers are the ones that require TLS.
- **"HTTP/2 removed head-of-line blocking."** It removed HTTP-level blocking. TCP-level blocking stays, and on one connection it now affects all streams.
- **"HTTP/3 is just HTTP/2 over UDP."** QUIC is a full transport with its own reliability, congestion control and encryption. HTTP/3 also needs QPACK.
- **"UDP means HTTP/3 is unreliable."** QUIC adds acknowledgements and resends on top of UDP. Delivery is as reliable as TCP.
- **"New versions change the meaning of HTTP."** `GET`, `404` and `Cache-Control` mean the same in all versions. The wire format changes.
- **"HTTP/3 is always faster."** On a stable low-loss wired link, the difference may be tiny. The biggest gains are on slow or lossy mobile networks.
- **"HTTP/3 replaced HTTP/2."** Servers normally run all three. The first visit often uses HTTP/2, and fallback depends on it.
- **"Multiplexing means parallel."** Frames still go on the wire one after another. They are interleaved, not sent at the same instant.
- **Enabling HTTP/3 but not opening UDP 443.** TCP 443 rules do not cover UDP. The site still works through fallback, so the problem stays hidden.
- **Keeping old HTTP/1.1 tricks.** Domain sharding on HTTP/2 hurts, because more connections mean less multiplexing and worse compression.
- **Saying push is the modern way to speed up pages.** Major browsers removed it. Use Early Hints and preload.
- **Confusing `Connection: keep-alive` with TCP keepalive.** The first reuses a connection for more requests. The second is a TCP probe that checks if an idle peer is still there.

## Interview Questions

**Q: What are the main differences between HTTP/1.1, HTTP/2 and HTTP/3?**

A: The meaning stays the same, but transport changes. HTTP/1.1 is text over TCP and handles one exchange at a time per connection. HTTP/2 is binary over TCP and multiplexes many streams on one connection, with HPACK header compression. HTTP/3 runs on QUIC over UDP, with QPACK. It has streams at the transport layer, so one lost packet does not stall the others.

**Q: What is head-of-line blocking?**

A: It is when the first item in a queue holds up everything behind it. In HTTP/1.1 it happens at the HTTP level, because responses go in order. In HTTP/2 that is fixed, but it still happens at the TCP level, because TCP delivers bytes in strict order. One lost segment stalls all streams. HTTP/3 fixes this, because QUIC orders each stream separately.

**Q: If HTTP/2 has multiplexing, why do we need HTTP/3?**

A: HTTP/2 multiplexes above TCP, and TCP sees only one byte stream. So a single lost packet blocks every stream until it is resent. On lossy networks, this can be worse than HTTP/1.1 with six connections. HTTP/3 also gets a faster handshake and connection migration.

**Q: Why does HTTP/3 use UDP and not a new transport or a changed TCP?**

A: TCP lives in operating system kernels, which update slowly, and middleboxes often break changes to it. A new IP protocol would be dropped by many firewalls and NATs. UDP already passes through most networks, so QUIC builds reliability on top of it in user space, where it can be updated with the application.

**Q: What is the difference between a frame, a message and a stream in HTTP/2?**

A: A frame is the smallest unit, with a 9-byte header and a payload. A message is a full request or response, made of frames. A stream is a numbered two-way channel carrying one request and its response. Frames from different streams can be mixed on one connection.

**Q: How does HPACK work, and why does HTTP/3 use QPACK?**

A: HPACK uses a static table of 61 common headers, a dynamic table built during the connection, and Huffman coding. It needs headers decoded in exact order, which TCP guarantees. QUIC streams are not ordered with each other, so HPACK would bring back blocking. QPACK sends table updates on separate streams and lets each header block say which table version it needs.

**Q: What was server push, and why did browsers remove it?**

A: It let a server send files before the client asked, using PUSH_PROMISE. But the server cannot see the browser cache, so it often sent files the client already had. Real gains were rare, and the code was complex. Chrome disabled it in version 106 and Firefox removed it in version 132. 103 Early Hints is the replacement, because the browser decides what to fetch.

**Q: How does a browser discover that a site supports HTTP/3?**

A: Usually with the `Alt-Svc` response header, such as `h3=":443"; ma=86400`, received over HTTP/2 or HTTP/1.1. It can also learn from an HTTPS DNS record before connecting. Then it tries QUIC on UDP, often racing it against TCP, and uses ALPN token `h3` in the handshake. If UDP is blocked, it falls back to TCP.

**Q: What is ALPN, and why can it not announce HTTP/3 on its own?**

A: ALPN is a TLS extension where the client lists protocols and the server picks one, without an extra round trip. It selects `h2` or `http/1.1` inside a TCP connection. HTTP/3 needs a different transport, so the client must already know to try UDP before ALPN runs. That is why Alt-Svc and DNS records exist.

**Q: How much faster is an HTTP/3 connection setup?**

A: With TLS 1.3 over TCP, you spend 2 RTT before sending the request: one for TCP, one for TLS. QUIC combines them into 1 RTT. On resumption, QUIC can send the request in the first flight with 0-RTT. Early data can be replayed, so use it only for safe requests.

**Q: A site has HTTP/3 enabled, but all users still connect with HTTP/2. What do you check?**

A: First, check that UDP 443 is open on every firewall, security group and load balancer. Then check that the `Alt-Svc` header is really sent, or that the HTTPS DNS record exists. Test with `curl --http3-only`. Check the client network too, because some networks block UDP 443. Users would fall back silently, so the site would appear healthy.

**Q: When would you not use HTTP/3?**

A: Between services in a data centre, loss is low and TCP is efficient, so HTTP/1.1 or HTTP/2 is often better. It can cost more CPU. Networks that block UDP, or tools that need TLS inspection, also limit it. On the public edge, you normally offer HTTP/3 and keep HTTP/2 as a fallback.

## Quick Recap

- HTTP versions keep the same meaning and change the wire format: text in 1.1, binary frames in 2, binary frames on QUIC in 3.
- HTTP/1.1 brought persistent connections. Pipelining existed but failed in practice, so browsers open up to 6 connections per host.
- HTTP/2 multiplexes streams on one TCP connection and removes HTTP-level head-of-line blocking, but TCP-level blocking remains.
- HTTP/3 runs on QUIC over UDP, so a loss affects only the streams in the lost packet. It also has a 1-RTT setup and connection migration.
- HPACK needs ordered delivery. QPACK uses separate encoder and decoder streams to work with unordered QUIC streams.
- Server push is removed from major browsers. 103 Early Hints took its place.
- ALPN picks the version during TLS. Alt-Svc and HTTPS DNS records tell clients that HTTP/3 exists. UDP 443 must be open, and TCP must remain as a fallback.
