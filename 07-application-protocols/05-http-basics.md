# HTTP Basics

**Level:** Beginner | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [DNS Basics](01-dns-basics.md)

## In One Line

HTTP (HyperText Transfer Protocol) is the set of rules a client uses to ask a server for something, and the server uses to answer. Every exchange is one request followed by one response.

## Why It Matters

Web pages, mobile apps and most APIs all talk HTTP. An API (Application Programming Interface) is a way for one program to ask another program for data or work.

Before HTTP, every service needed its own custom format. HTTP gives one simple format that any client, server, proxy or cache can read.

Interviewers ask about HTTP because it joins many topics: TCP, TLS, DNS, load balancers, caches and security. They also ask because real problems show up as HTTP status codes. If you can explain why a site returns 502 instead of 504, you can debug real systems.

## Simple Explanation

Think of a restaurant.

- You are the **client**. The kitchen is the **server**.
- You give the waiter an order: "One soup, no salt." That is the **request**.
- The waiter returns with the soup, or says "Sorry, no soup today." That is the **response**.
- The waiter forgets you after each order. If you ask for more water, you must say which table you are at. This is what **stateless** means: the server keeps no memory of earlier requests.

Where the analogy stops: a waiter can start talking to you, but in plain HTTP the server only answers. It does not start a conversation. Also, a real waiter does remember your face. HTTP needs an extra tool, the cookie, to do that.

## How It Works

### Where HTTP sits

HTTP is an application layer protocol. It uses a transport protocol below it.

| Version | Runs on | Default port |
|---|---|---|
| HTTP/1.1, HTTP/2 | TCP | 80 for `http://`, 443 for `https://` |
| HTTP/3 | QUIC, which runs on UDP | 443 |

HTTPS is the same HTTP, sent inside an encrypted TLS tunnel. See [TLS and HTTPS](07-tls-and-https.md). The differences between versions are in [HTTP/1.1, HTTP/2 and HTTP/3](06-http-versions.md). This chapter uses HTTP/1.1 because it is plain text and easy to read. Methods, status codes and headers mean the same in all versions.

### The steps of one exchange

1. The client reads the URL and finds the host name.
2. DNS turns the host name into an IP address.
3. The client opens a TCP connection to that address and port.
4. For HTTPS, the client and server do a TLS handshake.
5. The client sends the request.
6. The server sends the response.
7. The connection stays open for reuse, or it is closed.

```text
Client                                   Server
  |                                        |
  |--- TCP handshake (SYN, SYN-ACK, ACK) ->|
  |---- TLS handshake (HTTPS only) ------->|
  |                                        |
  |---- GET /index.html HTTP/1.1 --------->|
  |<--- HTTP/1.1 200 OK + page ------------|
  |                                        |
  |---- next request on same connection -->|
```

For the full journey of a page load, see [What Happens When You Open a Website](../02-network-models/05-what-happens-when-you-open-a-website.md).

### Parts of a URL

A URL (Uniform Resource Locator) says where a resource is. A **resource** is anything the server can name: a page, an image, one customer record.

```text
https://www.example.com:8443/products/42?color=red&size=m#reviews
\___/   \_____________/ \__/\__________/ \______________/ \_____/
scheme       host       port    path          query       fragment
```

The fragment (`#reviews`) is never sent to the server. The browser uses it to jump to a place on the page.

### Request format

An HTTP/1.1 request is plain text with four parts:

1. **Request line:** method, target, version.
2. **Headers:** one `Name: value` per line. A header is a small piece of extra information.
3. **Blank line:** this marks the end of the headers.
4. **Body:** optional data, such as a form or JSON.

```text
POST /api/orders HTTP/1.1                <- request line
Host: api.example.com                    <- headers start
Content-Type: application/json
Content-Length: 28
Accept: application/json
                                         <- blank line
{"item":"book","quantity":2}             <- body
```

The notes on the right are for you. They are not part of the message.

Each line ends with two characters: CR and LF (carriage return, line feed). `Content-Length` counts the bytes in the body, here 28. JSON (JavaScript Object Notation) is a simple text format for data.

The `Host` header is required in HTTP/1.1. One IP address can serve many sites, so the server needs to know which site you want.

### Response format

A response has the same shape. Only the first line changes.

```text
HTTP/1.1 201 Created                     <- status line
Location: /api/orders/1042               <- headers start
Content-Type: application/json
Content-Length: 38
                                         <- blank line
{"id":1042,"item":"book","quantity":2}   <- body
```

The status line holds the version, a three-digit status code and a short reason phrase. Programs should act on the code. The reason phrase is only for humans, and HTTP/2 and HTTP/3 drop it completely.

### Methods

A method (also called a verb) says what you want to do with the resource.

| Method | Meaning | Body in request? | Safe | Idempotent |
|---|---|---|---|---|
| GET | Read a resource | No | Yes | Yes |
| HEAD | Same as GET, but headers only | No | Yes | Yes |
| OPTIONS | Ask what is allowed | Rarely | Yes | Yes |
| TRACE | Echo the request back (for testing, usually disabled) | No | Yes | Yes |
| POST | Send data for processing, often to create | Yes | No | No |
| PUT | Create or fully replace at this URL | Yes | No | Yes |
| PATCH | Change part of a resource | Yes | No | No (see below) |
| DELETE | Remove a resource | Rarely | No | Yes |
| CONNECT | Open a tunnel through a proxy | No | No | No |

### Safe and idempotent

**Safe** means the method is read-only in intent. The client does not ask for any change on the server. The server may still write a log line, and that does not break the rule.

**Idempotent** means sending the same request many times has the same effect on the server as sending it once.

Analogy: a lift (elevator) button. Press "floor 5" once or ten times, and the lift still goes to floor 5. Pressing it is idempotent. A vending machine "buy" button is not: press it twice and you pay twice. The analogy stops at the response: idempotent is about the effect on the server, not the reply. A first `DELETE /orders/1042` can return 204 and a second can return 404, yet the end state is the same. The order is gone.

Why this matters: networks fail. If a client sends a request and the connection dies, it does not know whether the server acted. A safe or idempotent request can be retried without harm. A POST retried blindly can charge a card twice. Payment APIs often solve this with a unique key the client sends with each POST, commonly in a header named `Idempotency-Key`. This lets the server spot a repeat.

The fuller truth: PATCH is not idempotent by rule, but a given PATCH can be. "Set quantity to 2" is idempotent. "Add 1 to quantity" is not. Also, safe and idempotent are promises in the standard (RFC 9110). A badly written server can still delete data on a GET.

### Status codes

The first digit gives the class.

| Class | Meaning | Whose problem? |
|---|---|---|
| 1xx | Informational: keep going | Nobody, just a note |
| 2xx | Success | Nobody |
| 3xx | Redirection: look elsewhere | Client must take one more step |
| 4xx | Client error | The request is wrong |
| 5xx | Server error | The request looked fine, the server failed |

**1xx and 2xx**

| Code | Name | When it is used |
|---|---|---|
| 100 | Continue | Server says "send the body now" after the client sent `Expect: 100-continue` |
| 101 | Switching Protocols | Connection changes protocol, for example to WebSocket |
| 103 | Early Hints | Server hints which files to load before the final response |
| 200 | OK | Normal success |
| 201 | Created | New resource made, often with a `Location` header |
| 202 | Accepted | Work queued, not finished yet |
| 204 | No Content | Success, nothing to return |
| 206 | Partial Content | Only the requested byte range is returned |

**3xx**

| Code | Name | Lasting? | Method on the next request |
|---|---|---|---|
| 301 | Moved Permanently | Yes | Clients may change POST to GET |
| 302 | Found | No | Clients may change POST to GET |
| 303 | See Other | No | Always GET, used after a form POST |
| 304 | Not Modified | Not a real redirect | Use your cached copy, see caching below |
| 307 | Temporary Redirect | No | Method and body must stay the same |
| 308 | Permanent Redirect | Yes | Method and body must stay the same |

**4xx**

| Code | Name | When it is used |
|---|---|---|
| 400 | Bad Request | Broken syntax or invalid request |
| 401 | Unauthorized | You are not logged in, or your login is bad. Really means "unauthenticated". Comes with `WWW-Authenticate` |
| 403 | Forbidden | The server knows who you are (or does not care), and you may not do this |
| 404 | Not Found | Nothing at this URL. Also used to hide that something exists |
| 405 | Method Not Allowed | URL exists, method is not allowed there. Comes with an `Allow` header |
| 408 | Request Timeout | Client was too slow to send the request |
| 409 | Conflict | Clashes with current state, for example two people edit the same record |
| 410 | Gone | It existed and was removed on purpose |
| 413 | Content Too Large | Body is bigger than the server accepts |
| 415 | Unsupported Media Type | Server cannot read this `Content-Type` |
| 422 | Unprocessable Content | Syntax is fine, but the data makes no sense, for example quantity of -1 |
| 429 | Too Many Requests | Rate limit hit. May come with `Retry-After` |

**5xx**

| Code | Name | When it is used |
|---|---|---|
| 500 | Internal Server Error | Generic crash or bug in the application |
| 501 | Not Implemented | Server does not support this method at all |
| 502 | Bad Gateway | A proxy got a bad reply, or no usable reply, from the backend |
| 503 | Service Unavailable | Overloaded or in maintenance. May come with `Retry-After` |
| 504 | Gateway Timeout | A proxy waited too long for the backend |

A **proxy** is a server in the middle that passes traffic along. A **reverse proxy** sits in front of your web servers. A **backend** is the server behind it that does the real work. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

```text
Client ---> Reverse proxy ---> Backend app
                 |
                 +-- backend refused or sent garbage  = 502
                 +-- backend too slow, timer expired  = 504
                 +-- no healthy backend (often)       = 503
```

Exact behaviour depends on the proxy product. nginx, for example, returns 502 when a connection to the backend is refused, and 504 when its timeout expires.

### Headers

Header names are not case sensitive. `Host` and `host` are the same. HTTP/2 and HTTP/3 send them in lower case.

**Common request headers**

| Header | Purpose | Example |
|---|---|---|
| Host | Which site | `Host: www.example.com` |
| User-Agent | Which client software | `User-Agent: curl/8.7.1` |
| Accept | Formats the client wants | `Accept: application/json` |
| Accept-Encoding | Compression it can read | `Accept-Encoding: gzip, br` |
| Accept-Language | Preferred languages | `Accept-Language: en, ur;q=0.8` |
| Authorization | Login proof | `Authorization: Bearer abc123` |
| Cookie | Cookies for this site | `Cookie: session_id=7f3a9c21` |
| Referer | Page the link was on (the misspelling is official) | `Referer: https://www.example.com/` |
| Origin | Site that started the request | `Origin: https://shop.example.com` |

**Common response headers**

| Header | Purpose | Example |
|---|---|---|
| Content-Type | Format of the body | `Content-Type: text/html; charset=utf-8` |
| Content-Length | Size of the body in bytes | `Content-Length: 47` |
| Content-Encoding | Compression used | `Content-Encoding: gzip` |
| Location | Where to go next | `Location: /login` |
| Set-Cookie | Ask client to store a cookie | `Set-Cookie: theme=dark` |
| Cache-Control | Caching rules | `Cache-Control: max-age=3600` |
| ETag | Version tag of the content | `ETag: "v1-abc123"` |
| Server | Server software | `Server: nginx` |
| Retry-After | How long to wait | `Retry-After: 120` |
| Strict-Transport-Security | Use HTTPS only from now on | `Strict-Transport-Security: max-age=31536000` |

Some headers are not in the standard. `X-Forwarded-For`, for example, is a widely used convention. Proxies add it to carry the real client IP address.

### Cookies and sessions

HTTP is stateless, yet websites remember that you logged in. Cookies make this possible.

A **cookie** is a small name and value pair. The server asks the browser to store it, and the browser sends it back on later requests to that site. Browsers commonly allow about 4 KB per cookie.

A **session** is the server's record of one logged-in user. The server stores the data and gives the browser only a random ID.

Analogy: a coat check. You hand over your coat and get ticket 42. Later, you show the ticket and get your coat. The ticket holds no coat. The cookie is the ticket, and the session is the coat on the rack. The analogy stops here: the browser shows the ticket automatically on every matching request, and anyone who steals the ticket can use it. That is why session IDs must be long, random and sent only over HTTPS.

```text
Browser                                 Server
   |--- POST /login (user, password) ---->|
   |                                      | creates session 7f3a9c21
   |<-- 302 + Set-Cookie: session_id=... -|
   |                                      |
   |--- GET /account -------------------->|
   |    Cookie: session_id=7f3a9c21       | looks up session: user asha
   |<-- 200 OK + account page ------------|
```

Cookie attributes control how the cookie is used:

| Attribute | Meaning |
|---|---|
| `Expires` or `Max-Age` | When it dies. Without either, it is a session cookie, removed when the browser session ends |
| `Domain` | Which hosts get it. `Domain=example.com` includes subdomains. If left out, only the exact host gets it |
| `Path` | Which URL paths get it |
| `Secure` | Send only over HTTPS |
| `HttpOnly` | JavaScript cannot read it. This limits theft through XSS |
| `SameSite` | Limits sending on cross-site requests: `Strict`, `Lax` or `None`. This helps against CSRF |

`SameSite=None` needs `Secure` in modern browsers. If `SameSite` is missing, Chrome treats the cookie as `Lax`, but this default is not the same in all browsers. XSS and CSRF are covered in [Common Network Attacks](../09-security/03-common-network-attacks.md).

**Sessions and load balancers.** With many backend servers, a session stored in one server's memory is lost if the next request goes elsewhere. The usual fixes are a shared session store (often Redis) or sticky sessions. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

**Sessions and tokens.** Many APIs use a signed token such as a JWT (JSON Web Token) instead. The server can check the signature without looking up any stored session. The cost is that a token is harder to cancel before it expires.

### Caching headers

A **cache** keeps a copy of a response so the next request is faster. Caches exist in browsers (private, for one user) and in proxies or CDNs (shared by many users). A CDN (Content Delivery Network) is a set of servers around the world that hold copies of content. See [Content Delivery Networks](../10-enterprise-and-datacenter/06-cdn.md).

Analogy: a printed bus timetable at home. While it is valid, you read your copy and skip the trip to the station. When you doubt it, you phone and ask, "Is version 12 still current?" A short "yes" saves a long trip. The analogy stops because HTTP has clear numbers for how long a copy stays fresh, down to the second.

Caching has two ideas:

- **Freshness:** for how long the copy can be used with no check at all.
- **Validation:** after that, ask the server if the copy is still good. If so, it answers 304 with no body, which saves bandwidth.

| Header | Meaning |
|---|---|
| `Cache-Control: max-age=3600` | Fresh for 3600 seconds |
| `Cache-Control: s-maxage=600` | Same idea, but for shared caches only |
| `Cache-Control: no-cache` | May store, but must check with the server before each use |
| `Cache-Control: no-store` | Do not store at all. Use for private data |
| `Cache-Control: private` | Browser may store it, shared caches may not |
| `Cache-Control: public` | Any cache may store it |
| `Cache-Control: must-revalidate` | Once stale, must check and never serve it stale |
| `Cache-Control: immutable` | Will not change while fresh, so do not recheck |
| `Expires` | Older way to give a date. `max-age` wins if both exist |
| `ETag` / `If-None-Match` | Version tag from server, sent back by client to check |
| `Last-Modified` / `If-Modified-Since` | Same check, based on a date |
| `Age` | Seconds the response has already spent in a cache |
| `Vary: Accept-Encoding` | Keep a separate copy for each value of this request header |

Caching rules are in RFC 9111.

### Keep-alive (persistent connections)

Opening a TCP connection costs time. Keep-alive means reusing one connection for many requests.

- **HTTP/1.0:** closes after each response by default. Clients could ask for reuse with `Connection: keep-alive`.
- **HTTP/1.1:** connections stay open by default. Either side sends `Connection: close` to end after the response.
- **HTTP/2 and HTTP/3:** one connection carries many requests at once, and the `Connection` header is not used.

```text
Without keep-alive                With keep-alive
(RTT = 50 ms, 10 small files)     (same 10 files)

connect   50 ms                   connect   50 ms
GET 1     50 ms                   GET 1     50 ms
close                             GET 2     50 ms
connect   50 ms                   ...
GET 2     50 ms                   GET 10    50 ms
...
total about 1000 ms               total about 550 ms
```

RTT (round-trip time) is the time for a message to go there and back. This sum is simplified. It ignores file size and TCP slow start, and HTTPS adds more time per new connection, so reuse helps even more.

Servers close idle connections to save resources. The idle timer is a server setting and varies. For example, the default is 75 seconds in nginx and 5 seconds in Apache httpd 2.4.

In HTTP/1.1, one connection handles one request at a time, so most browsers open up to 6 connections per host. This limit is browser practice, not a rule in the standard.

HTTP keep-alive is different from **TCP keepalive**. TCP keepalive sends empty probe packets to check that an idle peer is still alive. On Linux it first fires after 7200 seconds of idle time by default. HTTP keep-alive is about reusing a connection for more requests.

### CORS basics

The **same-origin policy** is a browser rule. A script on one origin cannot read responses from another origin. An **origin** is scheme + host + port.

| URL compared to `https://shop.example.com` | Same origin? | Why |
|---|---|---|
| `https://shop.example.com/cart` | Yes | Only the path differs |
| `http://shop.example.com` | No | Scheme differs |
| `https://shop.example.com:8443` | No | Port differs |
| `https://api.example.com` | No | Host differs |

Without this rule, any website you visit could call your bank with your cookies and read the reply.

**CORS** (Cross-Origin Resource Sharing) is the way a server relaxes that rule. The server lists who may read its responses, using headers. CORS is defined in the WHATWG Fetch Standard, not in an RFC.

Analogy: a security guard at an office door (the browser). A staff member from Shop wants to read a letter from Api. The guard checks whether Api wrote "Shop may read this" on the letter. The analogy stops because the guard exists only in browsers. curl, mobile apps and other servers have no guard. So CORS does not protect your API. It protects users of browsers.

**Simple requests** have no check before they are sent. These are GET, HEAD or POST with basic headers, and a `Content-Type` of `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`. The server handles the request, then the browser decides if the script may read the reply.

**Other requests** need a **preflight**. Examples are PUT, DELETE, a POST with `application/json`, or a custom header. The browser first sends an OPTIONS request to ask for permission.

```text
OPTIONS /orders HTTP/1.1
Host: api.example.com
Origin: https://shop.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://shop.example.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 600
Vary: Origin
```

`Access-Control-Max-Age: 600` lets the browser reuse this answer for 600 seconds. If the headers allow it, the browser sends the real POST. If not, it stops and shows a CORS error in the console.

To send cookies across origins, the server must add `Access-Control-Allow-Credentials: true`, and `Access-Control-Allow-Origin` must name the exact origin. The `*` wildcard is not allowed with credentials.

### REST

REST (Representational State Transfer) is a style for designing APIs. It is not a protocol. Roy Fielding described it in his doctoral dissertation in 2000.

The main idea: each thing is a resource with a URL, and you act on it with standard HTTP methods.

| Request | Meaning | Usual success code |
|---|---|---|
| `GET /orders` | List orders | 200 |
| `POST /orders` | Create an order | 201 with `Location` |
| `GET /orders/1042` | Read one order | 200 |
| `PUT /orders/1042` | Replace the order | 200 or 204 |
| `PATCH /orders/1042` | Change part of it | 200 or 204 |
| `DELETE /orders/1042` | Delete the order | 204 |

Good REST URLs use nouns, not verbs. Use `DELETE /orders/1042`, not `POST /deleteOrder?id=1042`.

REST's formal constraints are: client-server, stateless, cacheable, uniform interface, layered system, and code on demand (optional).

The fuller truth: most "REST APIs" follow only part of this. Fielding's uniform interface also says responses should carry links to the next actions, and few APIs do. In practice, "REST" usually means JSON over HTTP with clean URLs and correct status codes.

### Using curl

curl is a command line HTTP client. It shows you exactly what is sent and received.

On Windows 10 and 11, `curl.exe` is built in. In Windows PowerShell 5.1, plain `curl` is an alias for `Invoke-WebRequest`, a different tool. Always type `curl.exe` there.

Commands below work on Linux and macOS. On Windows, swap `curl` for `curl.exe`.

```bash
# Linux and macOS
curl http://www.example.com/              # print the body
curl -i http://www.example.com/           # include response headers
curl -I http://www.example.com/           # HEAD request, headers only
curl -v http://www.example.com/           # verbose, show both directions
curl -L http://www.example.com/old        # follow redirects
curl -o page.html http://www.example.com/ # save to a file
curl -s -o /dev/null -w "%{http_code}\n" http://www.example.com/  # code only
curl -H "Accept: application/json" http://api.example.com/orders  # add header
curl -u asha:test123 http://api.example.com/orders  # Basic authentication
curl --connect-timeout 5 --max-time 20 http://www.example.com/  # set limits
curl --resolve www.example.com:80:203.0.113.10 http://www.example.com/  # skip DNS
```

The same requests with a JSON body, where quoting differs by shell:

```bash
# Linux and macOS
curl -X POST http://api.example.com/orders \
  -H "Content-Type: application/json" \
  -d '{"item":"book","quantity":2}'
```

```powershell
# Windows PowerShell: keep the JSON in a file to avoid quoting problems
curl.exe -X POST http://api.example.com/orders `
  -H "Content-Type: application/json" `
  --data-binary "@order.json"

# PowerShell native alternative
Invoke-RestMethod -Uri http://api.example.com/orders -Method Post `
  -ContentType "application/json" -Body '{"item":"book","quantity":2}'
```

`-d` already makes curl use POST, so `-X POST` is optional here. Avoid `-k`, which turns off certificate checks. It hides real TLS problems. For a list of other connection tools, see [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

## Example

The site `www.example.com` here is imaginary, at the documentation address 203.0.113.10. Your own output will differ, and the debug lines change a little between curl versions.

### Example 1: Read a real exchange with curl -v

```bash
curl -v http://www.example.com/index.html
```

```text
* Host www.example.com:80 was resolved.
* IPv4: 203.0.113.10
*   Trying 203.0.113.10:80...
* Connected to www.example.com (203.0.113.10) port 80
> GET /index.html HTTP/1.1
> Host: www.example.com
> User-Agent: curl/8.7.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Fri, 30 Oct 2026 09:15:00 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 47
< Cache-Control: max-age=3600
< ETag: "v1-abc123"
<
<html><body><h1>Hello, world</h1></body></html>
* Connection #0 to host www.example.com left intact
```

Reading it:

- Lines with `*` are curl's own notes. DNS gave 203.0.113.10, and TCP connected on port 80.
- Lines with `>` are what the client sent. The lone `>` is the blank line that ends the headers.
- Lines with `<` are what the server sent.
- `Content-Length: 47` matches the 47 bytes of the HTML body.
- `left intact` means keep-alive is working. curl did not close the connection itself.

If you try a real `https://` site, you will often see `HTTP/2 200` and lower-case header names. That is normal. Add `--http1.1` to force the older form.

### Example 2: Caching and 304

The response above is fresh for 3600 seconds, until 10:15:00. For that hour, the browser sends nothing and uses its copy.

At 11:20:00, the copy is stale, so the browser checks it with the ETag:

```text
GET /index.html HTTP/1.1
Host: www.example.com
If-None-Match: "v1-abc123"
```

The page has not changed, so the server replies with no body:

```text
HTTP/1.1 304 Not Modified
Date: Fri, 30 Oct 2026 11:20:00 GMT
Cache-Control: max-age=3600
ETag: "v1-abc123"
```

The browser keeps its 47-byte copy, which is now fresh for one more hour. If the page had changed, the server would send a normal 200 with a new ETag and the new body.

### Example 3: Login with a cookie

This logs in to a test site and stores the cookie in a file. `-c` saves cookies and `-b` sends them. Use a test account only, since a password on the command line can land in your shell history.

```bash
curl --http1.1 -i -c jar.txt -d "user=asha&password=test123" \
  https://shop.example.com/login
```

```text
HTTP/1.1 302 Found
Location: /account
Set-Cookie: session_id=7f3a9c21; Path=/; Secure; HttpOnly; SameSite=Lax
Content-Length: 0
```

```bash
curl --http1.1 -i -b jar.txt https://shop.example.com/account
```

```text
> Cookie: session_id=7f3a9c21     (sent by curl, shown with -v)

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: private, no-store
Content-Length: 512
```

The account page says `no-store` because personal data must not be cached.

### Example 4: Speak HTTP by hand

HTTP/1.1 is plain text, so you can type it. This sends a raw request with netcat and no HTTP client.

```bash
# Linux, netcat installed
printf 'GET /index.html HTTP/1.1\r\nHost: www.example.com\r\nConnection: close\r\n\r\n' \
  | nc www.example.com 80
```

You get the same response as Example 1, and the connection closes because you asked for that. Remove the `Host` line, and an HTTP/1.1 server should reply 400 Bad Request.

## Going Deeper

**How the receiver knows where the body ends.** TCP is a stream of bytes with no message borders. HTTP/1.1 marks the end in one of these ways:

- `Content-Length`: the exact byte count.
- `Transfer-Encoding: chunked`: the body comes in pieces, each with its size in hexadecimal first. A size of 0 ends it. This is useful when the size is not known in advance, such as live generated output.
- Closing the connection, an older and weaker way.

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

5
Hello
7
, world
0

```

**Virtual hosting.** Many sites can share one IP address. The server picks the site using the `Host` header. For HTTPS, it must pick a certificate before it sees that header, so the client also sends the name in the TLS handshake (SNI). See [TLS and HTTPS](07-tls-and-https.md).

**Content negotiation.** The client states what it can accept, and the server picks the best match. With `Accept-Encoding: gzip, br`, the server may compress the body and reply with `Content-Encoding: gzip`. This often shrinks text a lot. Images and video are already compressed, so they gain little.

**Range requests.** A client can ask for part of a file with `Range: bytes=0-1023`. The server replies 206 Partial Content with the first 1024 bytes. Download managers and video players use this to resume and to seek.

**Authentication.** With Basic authentication, the client sends `Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==`. That value is Base64 of `Aladdin:open sesame`. Base64 is encoding, not encryption, so anyone can decode it. Use it only over HTTPS. Bearer tokens (`Authorization: Bearer <token>`) are the usual choice for modern APIs.

**Beyond request and response.** WebSocket starts as an HTTP request and upgrades with 101 to a two-way channel. It is covered in [Real-Time and Modern Protocols](14-realtime-and-modern-protocols.md).

**Standards.** HTTP meaning is in RFC 9110, caching in RFC 9111, and the HTTP/1.1 wire format in RFC 9112. Cookies are in RFC 6265.

## Common Mistakes

- **Thinking 401 means "forbidden".** 401 means the server does not know who you are. 403 means it knows, or does not care, and still says no.
- **Mixing up 502 and 504.** 502 is a bad or failed reply from the backend. 504 is no reply in time. Both come from a proxy, not from the app itself.
- **Thinking `no-cache` means "do not cache".** It means store it, but check before each use. `no-store` is the one that forbids storing.
- **Thinking CORS protects the server.** The browser enforces it. curl ignores it completely. You still need real authentication.
- **Fixing CORS on the client.** The allow headers must come from the server that receives the request.
- **Thinking CORS blocks the request itself.** A simple cross-origin POST still reaches the server and can change data. Only reading the reply is blocked.
- **Using GET to change data.** Crawlers, previews and caches may repeat GETs. A GET "delete" link can wipe data.
- **Thinking POST is more secure than GET.** Both are plain text without HTTPS. POST only keeps data out of the URL, and so out of history and logs.
- **Thinking idempotent means the same response.** It means the same effect on the server. The status code can differ.
- **Returning 200 for errors.** A body of `{"error":"not found"}` with code 200 misleads clients, caches and monitoring.
- **Using 301 for a short-term move.** Browsers can cache a 301 for a long time, which makes it hard to undo.
- **Confusing HTTP keep-alive with TCP keepalive.** One reuses a connection for requests. The other probes an idle peer.
- **Typing `curl` in Windows PowerShell 5.1.** That runs `Invoke-WebRequest`, so curl flags fail. Use `curl.exe`.

## Interview Questions

**Q: What is the structure of an HTTP request and response?**

A: A request has a request line with method, target and version, then headers, a blank line, and an optional body. A response has the same shape, but the first line is a status line with the version, status code and reason phrase.

**Q: What is the difference between GET and POST?**

A: GET reads a resource. It is safe, idempotent and usually cacheable, and its parameters go in the URL. POST sends data in the body for the server to process. It is neither safe nor idempotent, and it is not normally cached. Without HTTPS, neither is private.

**Q: What do safe and idempotent mean? Give examples.**

A: Safe means the request should not change server state, like GET or HEAD. Idempotent means repeating it has the same effect as doing it once, like PUT or DELETE. Every safe method is idempotent, but not the reverse. This matters because a client can retry idempotent requests after a network failure.

**Q: What is the difference between PUT, PATCH and POST?**

A: PUT replaces the whole resource at a known URL and is idempotent. PATCH changes part of a resource. POST sends data to be processed, often creating a new resource at a URL chosen by the server. POST is not idempotent, so two POSTs can create two records.

**Q: What is the difference between 401 and 403?**

A: 401 means I have not proved who I am, so the server asks me to authenticate. 403 means the server understood the request and refuses it, and logging in again will usually not help.

**Q: What is the difference between 502, 503 and 504?**

A: All three usually come from a proxy or load balancer. 502 means it got an invalid reply or a failed connection from the backend. 503 means the service cannot handle the request now, because of overload or maintenance. 504 means the backend did not reply in time.

**Q: What is the difference between 301, 302, 307 and 308?**

A: 301 and 308 are permanent, 302 and 307 are temporary. With 301 and 302, clients may change a POST into a GET. With 307 and 308, the method and body must stay the same.

**Q: HTTP is stateless, so how does login work?**

A: After login, the server creates a session and sends a random ID in a `Set-Cookie` header. The browser sends this ID in the `Cookie` header on every later request, and the server looks it up. Each request still stands alone, but the cookie carries the identity.

**Q: What do the HttpOnly, Secure and SameSite cookie flags do?**

A: HttpOnly stops JavaScript from reading the cookie, which limits theft through XSS. Secure sends it only over HTTPS. SameSite limits sending it on cross-site requests, which helps against CSRF.

**Q: What is the difference between no-cache and no-store? How does a 304 work?**

A: `no-store` means never keep a copy. `no-cache` means keep it, but check with the server first. To check, the client sends `If-None-Match` with the saved ETag. If nothing changed, the server replies 304 with no body, and the client uses its copy.

**Q: What is HTTP keep-alive and why does it help?**

A: It reuses one TCP connection for many requests, which avoids a new TCP and TLS handshake each time. That reduces latency and server load. It is the default in HTTP/1.1, and `Connection: close` turns it off.

**Q: What is CORS, and why does my API work in curl but fail in the browser?**

A: The browser's same-origin policy stops a script from reading responses from another origin, meaning a different scheme, host or port. CORS lets the server allow this with `Access-Control-Allow-Origin` and related headers. curl has no such policy, so it works there. The fix is to add the right headers on the server, including the OPTIONS preflight reply.

## Quick Recap

- HTTP is a stateless request and response protocol: start line, headers, blank line, optional body.
- Safe methods change nothing, and idempotent methods can be repeated with the same effect. POST is neither.
- Status classes: 1xx info, 2xx success, 3xx redirect, 4xx client error, 5xx server error.
- Cookies carry a session ID so a stateless protocol can remember users. Protect them with Secure, HttpOnly and SameSite.
- `Cache-Control` sets freshness, ETag with 304 handles validation, and `no-store` forbids storing.
- Keep-alive reuses connections. CORS is enforced by browsers, not servers.
- REST maps resources to URLs and actions to methods, and `curl -v` shows you every step.
