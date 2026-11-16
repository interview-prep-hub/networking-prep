# DNS Basics

**Level:** Beginner | **Prerequisites:** [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md)

## In One Line

DNS (Domain Name System) is the internet's distributed lookup system. It turns a name like `www.example.com` into an IP address like `203.0.113.10`.

## Why It Matters

Computers connect using IP addresses. An IP address is a number that identifies a device on a network. People cannot remember numbers for every site, and those numbers change. Names are easy to remember and can stay the same when the server moves.

Before DNS, the ARPANET (the early network that grew into the internet) used one shared file called `HOSTS.TXT`. Every computer downloaded a copy from one central place. This stopped working as the network grew: the file became too big, changes were slow, and two people could pick the same name.

DNS replaced that file in the 1980s. The core standards today are RFC 1034 and RFC 1035, both from 1987. DNS solved the problem with three ideas:

- **Split the data.** No single server holds all names.
- **Split the control.** Each owner manages their own names.
- **Save answers.** Answers are kept for a while, so most lookups never travel far.

Interviewers ask about DNS for a practical reason. Almost every network action starts with a DNS lookup, so many "the network is down" problems are really DNS problems. The classic question, "what happens when you open a website?", also starts here. See [What Happens When You Open a Website](../02-network-models/05-what-happens-when-you-open-a-website.md).

## Simple Explanation

Think of your phone's contacts. You tap "Mom" and the phone dials the number. You never need to remember the digits. DNS is the contacts list of the internet.

But no single phone could hold every name in the world. So imagine you have a personal assistant, and you ask for the phone number of the sales desk at Example Corp.

1. The assistant calls the national information office. It says, "I do not know that desk, but here is the office for all companies."
2. The assistant calls the company office. It says, "I do not know that desk, but here is Example Corp's own front desk."
3. The assistant calls Example Corp's front desk. It says, "The sales desk number is 555-0110."
4. The assistant gives you the number and writes it down for next time.

In DNS, these roles have names:

| In the story | In DNS |
|---|---|
| You | The stub resolver on your device |
| Your assistant | The recursive resolver |
| National information office | Root server |
| Company office | TLD server, such as the one for `.com` |
| Example Corp's front desk | Authoritative server for `example.com` |
| The note the assistant keeps | The cache |

**Where the analogy stops being true:** a contacts entry is one name and one number. In DNS, one name can have many addresses, and one address can serve many names. Also, your assistant keeps a note until they lose it. A DNS cache removes the note at an exact time set by the owner of the name.

## How It Works

### Names are read from right to left

A **domain name** is a name made of parts separated by dots. Each part is called a **label**. Take `www.example.com.`:

| Part | Meaning |
|---|---|
| `.` (the last dot) | The root, the top of the whole tree. Usually hidden. |
| `com` | The TLD (top-level domain) |
| `example` | The domain someone registered, sitting under `com` |
| `www` | A host (one service or machine) inside `example.com` |

A name written in full, all the way to the root, is an **FQDN** (fully qualified domain name). `www.example.com.` is the FQDN. Most tools add the last dot for you.

Limits: one label is at most 63 characters, and a full name is at most 253 characters in normal text form. DNS names are not case sensitive, so `WWW.Example.COM` is the same name.

### The hierarchy: root, TLD, authoritative

DNS is an upside-down tree. The top is the root. Each level hands control of a branch to the level below it. This handing over is called **delegation**.

```text
                         . (root)
                            |
         +---------------+--+------------+
         |               |               |
        com             org              uk         <- TLDs
         |               |               |
      example         wikipedia          co
         |                               |
     +---+---+                        example
     |       |
    www     mail                                    <- hosts
```

**Root servers** sit at the top. They do not know the address of `www.example.com`. They only know which servers handle each TLD.

There are 13 root server identities, named `a.root-servers.net` to `m.root-servers.net`. They are run by 12 independent organisations, including Verisign, NASA, ISC and RIPE NCC. Each identity is really many machines around the world that share one IP address. This method is called anycast, and your packets go to the nearest copy. Together there are more than 1,000 instances. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

**TLD servers** handle the last label of a name. There are generic TLDs (`.com`, `.org`, `.net`) and country-code TLDs (`.uk`, `.de`, `.pk`). A TLD server does not know your host's address either. It knows which servers are in charge of each domain under it. For `.com`, those are `a.gtld-servers.net` to `m.gtld-servers.net`, run by Verisign.

**Authoritative servers** hold the real records for a domain. "Authoritative" means "the official source". They give the final answer. A domain owner normally has at least two of them for safety, often through a DNS hosting service such as Amazon Route 53 or Cloudflare.

The set of records one group of authoritative servers is responsible for is called a **zone**. A zone is not always the same as a domain. The owner of `example.com` can delegate `dev.example.com` to other servers, and that becomes a separate zone.

### The players on the client side

**Stub resolver.** A resolver is software that finds DNS answers. The stub resolver is the small one built into your operating system. It cannot walk the tree by itself. It only asks one question to a configured server and waits. Applications reach it through system calls such as `getaddrinfo()`.

**Recursive resolver.** This is the server that does the real work. It takes the question, walks the tree, and returns the final answer. It usually belongs to your ISP (internet service provider), your company, or a public service such as Google (`8.8.8.8`), Cloudflare (`1.1.1.1`) or Quad9 (`9.9.9.9`). Your device normally learns its address from [DHCP](04-dhcp.md).

**Forwarder.** A forwarder is a DNS server that does not walk the tree. It passes questions to another resolver and caches the answers. Many home routers do this, which is why your DNS server often shows as `192.168.1.1`.

Check which server your device uses:

```powershell
# Windows (PowerShell)
Get-DnsClientServerAddress -AddressFamily IPv4
```

```bash
# Linux
resolvectl status          # on systems using systemd-resolved
cat /etc/resolv.conf       # the classic config file
```

**Note:** on many Linux systems `/etc/resolv.conf` shows `nameserver 127.0.0.53`. That is `systemd-resolved`, a small local service with a cache. It passes questions to the real resolver. `resolvectl status` shows the real one.

### Recursive vs iterative queries

A **query** is a DNS question. The two types differ in who does the work.

| | Recursive query | Iterative query |
|---|---|---|
| Who sends it | Stub resolver to recursive resolver | Recursive resolver to root, TLD, authoritative |
| What it means | "Give me the final answer or an error" | "Tell me the best you know" |
| Possible replies | Final answer, or an error | Answer, or a **referral** |
| Who does the walking | The server | The asker |

A **referral** is a reply that says, "I do not have the answer, but ask these servers."

**Analogy:** a recursive query is telling an assistant, "Find this out and call me back." An iterative query is asking at a help desk, and being pointed to another desk. **Where it stops being true:** a help desk may walk you over to the next desk. A DNS server never does this in an iterative query. It only gives the pointer.

**Fuller truth:** the difference is signalled by flags in the message. The client sets RD (recursion desired). The server sets RA (recursion available) if it offers recursion. Root and TLD servers do not offer recursion, so they always return referrals.

### Full lookup walk-through for www.example.com

Setup for this example, with every cache empty:

| Item | Name | IP address |
|---|---|---|
| Your laptop | | `192.168.1.20` |
| Recursive resolver | your ISP's resolver | `198.51.100.53` |
| Root server | `a.root-servers.net` | `198.41.0.4` |
| TLD server | `a.gtld-servers.net` | `192.5.6.30` |
| Authoritative server | `ns1.example.com` | `203.0.113.53` |
| The web server | `www.example.com` | `203.0.113.10` |

The root and TLD addresses are the real ones. The others are examples.

**Local checks, before any packet leaves the laptop:**

1. You open `https://www.example.com`. The browser checks its own DNS cache. Nothing is found.
2. The browser asks the OS. The stub resolver checks the hosts file and the OS cache. Nothing is found.

**On the network, 8 messages travel:**

```text
 Laptop      Recursive         Root       .com TLD    example.com
 (stub)      resolver          server     server      authoritative
   |             |               |            |             |
   | 1 A www?    |               |            |             |
   |------------>|               |            |             |
   |             | 2 A www?      |            |             |
   |             |-------------->|            |             |
   |             | 3 ask .com    |            |             |
   |             |<--------------|            |             |
   |             | 4 A www?                   |             |
   |             |--------------------------->|             |
   |             | 5 ask ns1.example.com      |             |
   |             |<---------------------------|             |
   |             | 6 A www?                                 |
   |             |----------------------------------------->|
   |             | 7 203.0.113.10, TTL 300                  |
   |             |<-----------------------------------------|
   | 8 203.0.113.10                                         |
   |<------------|
```

1. **Stub to resolver (recursive).** The laptop sends a UDP packet from a random source port, such as 52841, to `198.51.100.53` port 53. It asks for the A record of `www.example.com`. An A record maps a name to an IPv4 address.
2. **Resolver to root (iterative).** The resolver checks its cache and finds nothing. It has a built-in list of root server addresses, called root hints. It asks `198.41.0.4`.
3. **Root replies with a referral.** "I do not know that name. The `.com` servers are `a.gtld-servers.net` and others. Here are their addresses."
4. **Resolver to TLD (iterative).** The resolver asks `192.5.6.30` the same question.
5. **TLD replies with a referral.** "The servers for `example.com` are `ns1.example.com` and `ns2.example.com`. `ns1` is at `203.0.113.53`." An NS (name server) record names the servers for a zone.
6. **Resolver to authoritative (iterative).** The resolver asks `203.0.113.53`.
7. **Authoritative gives the answer.** "`www.example.com` is `203.0.113.10`, keep it for 300 seconds." The AA (authoritative answer) flag is set.
8. **Resolver to stub.** The resolver saves everything it learned, then answers the laptop.

**After the answer,** the stub caches the result and gives it to the browser. The browser opens a TCP connection to `203.0.113.10` port 443. DNS has finished its job.

**Glue records.** In step 5, the TLD server said to ask `ns1.example.com`. But that name is inside `example.com`, the zone we are trying to reach. It is a chicken-and-egg problem. So the TLD server includes the IP address of `ns1.example.com` in its reply. This extra address is called a **glue record**.

**Fuller truth, three simplifications:**

- Many modern resolvers do not send the full name to every server. They ask the root only about `com`, the TLD only about `example.com`, and so on. This is QNAME minimisation (RFC 9156), and it protects privacy.
- Devices usually ask for A and AAAA records at the same time. An AAAA record maps a name to an IPv6 address.
- The name may be a CNAME, an alias that points to another name. The resolver then follows that name too. Record types are covered in [DNS Record Types](02-dns-records.md).

### TTL and caching layers

A **cache** is a short-term memory of past answers. Without caching, the root servers would receive a query for almost every click on the internet.

**TTL (time to live)** is the number of seconds an answer may be kept. The zone owner sets it on each record. The authoritative server sends it with the answer. Common values are 60, 300, 3600 (1 hour) and 86400 (1 day).

**Analogy:** TTL is the "use by" date on milk. **Where it stops being true:** milk goes bad slowly and by nature. A DNS answer may still be correct when its TTL ends, and it may already be wrong before then. TTL is a rule set by the owner, not a sign of whether the data is good.

The TTL counts down in the cache. If a resolver received `300` and 13 seconds pass, it tells the next client `287`. So a lower cache cannot keep an answer longer than the original 300 seconds from when the resolver stored it, if every layer behaves.

An answer can be cached at each of these layers. Your query stops at the first layer that has it:

```text
 Browser cache         (browser process)
      | miss
      v
 OS cache + hosts file (stub resolver on your device)
      | miss
      v
 Home router           (forwarder, if there is one)
      | miss
      v
 Recursive resolver    (shared by many users)
      | miss
      v
 Root, then TLD, then authoritative
```

| Layer | Where it lives | How long it keeps data |
|---|---|---|
| Browser | Inside the browser | Usually short, varies by browser and version |
| Application | Some runtimes, such as the Java JVM | Set by the runtime's own config |
| Operating system | DNS Client service (Windows), `mDNSResponder` (macOS), `systemd-resolved` (many Linux systems) | Up to the record's TTL |
| Router | Forwarder on the router | Up to the TTL, varies by product |
| Recursive resolver | ISP, company or public service | Up to the TTL, with a configured upper limit |

**Note:** the authoritative server is not a cache. It is the source. Also, a Linux system with only the C library has no OS DNS cache. Caching needs a service such as `systemd-resolved`, `nscd` or `dnsmasq`.

Each resolver also has a limit on how long it will keep an answer, whatever the TTL says. This is a vendor setting. BIND defaults to one week and Unbound to one day.

The shared recursive resolver gives the biggest saving. After one user looks up `www.example.com`, every other user of that resolver gets the answer at once. The resolver also caches the steps along the way. The `.com` referral usually has a TTL of 172800 seconds (2 days), so the next lookup of any `.com` name skips the root.

**Failed lookups are cached too.** If a name does not exist, the server returns NXDOMAIN (non-existent domain), and resolvers cache that for a while. This is **negative caching** (RFC 2308). So if you create a record just after someone looked it up and failed, they may keep getting "not found" until the negative entry expires.

### The hosts file

The **hosts file** is a plain text file on your device that maps names to IP addresses. It is the direct descendant of `HOSTS.TXT`. It works with no DNS server at all.

| OS | Path |
|---|---|
| Linux, macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

```text
# IP address    name
127.0.0.1       localhost
192.168.1.50    intranet.example.com
```

**Analogy:** a sticky note on your own phone. It wins over the public directory, but it helps only you. **Where it stops being true:** a sticky note is easy to see. A hosts entry can quietly change where your traffic goes for months.

Rules to remember:

- It needs admin or root rights to edit.
- It applies to that device only.
- It has no wildcards. `*.example.com` does not work.
- It has no port numbers and no TTL.
- Malware sometimes edits it to send you to a fake site, so it is worth checking during a strange fault.

Common uses are testing a new server before changing public DNS, and blocking a name by pointing it at `0.0.0.0`.

**Fuller truth:** "hosts file first" is a default, not a law. On Linux the order is set in `/etc/nsswitch.conf`, in a line such as `hosts: files dns`. Distributions differ. On Windows, hosts entries are loaded into the DNS Client cache. A browser using DoH (DNS over HTTPS) may also send queries directly to its own resolver instead of the OS-configured server. See [DNS Advanced](03-dns-advanced.md).

### Flushing the cache

**Flushing** means deleting cached answers so the next lookup asks again. You do this after a DNS change, or when you suspect a stale answer. Stale means old and possibly wrong.

```powershell
# Windows
ipconfig /flushdns        # clear the OS cache
ipconfig /displaydns      # show the OS cache
Clear-DnsClientCache      # PowerShell way to clear it
Get-DnsClientCache        # PowerShell way to show it
```

```bash
# Linux with systemd-resolved
resolvectl flush-caches
resolvectl statistics     # shows cache size, hits and misses

# Linux with nscd or dnsmasq: restart the service
sudo systemctl restart nscd
sudo systemctl restart dnsmasq

# macOS
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

```text
! Cisco IOS
Router# show hosts
Router# clear host *
```

Browsers have their own cache. In Chrome, open `chrome://net-internals/#dns` and click "Clear host cache". In Firefox, open `about:networking#dns` and click "Clear DNS Cache".

On a server you run yourself:

```text
BIND:                   rndc flush
Windows DNS Server:     Clear-DnsServerCache   (PowerShell)
```

Flushing has limits:

- It clears only the cache you touched. Your PC flush does not clear your ISP's resolver.
- It does not close open connections. A browser may keep using a live connection to the old IP.
- It does not change the authoritative data.

## Example

The commands below run from the laptop in the walk-through, on Linux. `dig` is a DNS query tool. `@198.51.100.53` tells it which server to ask. Output is trimmed for clarity. For full tool usage see [DNS Tools](../12-troubleshooting-and-tools/04-dns-tools.md).

### Example 1: cold lookup, then cached lookup

First query, with an empty cache:

```bash
$ dig @198.51.100.53 www.example.com
```

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41873
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;www.example.com.               IN      A

;; ANSWER SECTION:
www.example.com.        300     IN      A       203.0.113.10

;; Query time: 86 msec
;; SERVER: 198.51.100.53#53(198.51.100.53) (UDP)
;; MSG SIZE  rcvd: 60
```

How to read this:

- `status: NOERROR` means the query worked.
- `rd ra` means recursion was desired and is available.
- There is no `aa` flag, because the resolver is not the official source.
- `300` is the TTL in seconds.
- `86 msec` is the time for the resolver to walk the tree.

The same command 13 seconds later:

```text
;; ANSWER SECTION:
www.example.com.        287     IN      A       203.0.113.10

;; Query time: 1 msec
```

The TTL fell by 13. The time fell from 86 ms to 1 ms. That is a cache hit: the answer was found in the cache.

### Example 2: see a referral and an authoritative answer

Ask a root server directly. `+norecurse` clears the RD flag, so this is a true iterative query:

```bash
$ dig +norecurse @198.41.0.4 www.example.com
```

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5120
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 27

;; AUTHORITY SECTION:
com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
;; (11 more NS records)

;; ADDITIONAL SECTION:
a.gtld-servers.net.     172800  IN      A       192.5.6.30
;; (more addresses)
```

`ANSWER: 0` shows the root gave no answer. The AUTHORITY section is the referral. The ADDITIONAL section holds the server addresses. There is no `ra` flag, because root servers do not do recursion.

Now ask the authoritative server:

```bash
$ dig +norecurse @203.0.113.53 www.example.com
```

```text
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
www.example.com.        300     IN      A       203.0.113.10
```

The `aa` flag proves this is the official answer. The TTL is the full 300, because the data did not come from a cache.

### Example 3: walk the whole tree yourself

`dig +trace` starts at the root and follows every referral. Your machine acts as the recursive resolver.

```bash
$ dig @198.51.100.53 +trace www.example.com
```

```text
.                  518400  IN  NS  a.root-servers.net.
;; (12 more root servers)
;; Received 525 bytes from 198.51.100.53#53(198.51.100.53) in 4 ms

com.               172800  IN  NS  a.gtld-servers.net.
;; (12 more .com servers)
;; Received 1170 bytes from 198.41.0.4#53(a.root-servers.net) in 20 ms

example.com.       172800  IN  NS  ns1.example.com.
example.com.       172800  IN  NS  ns2.example.com.
;; Received 120 bytes from 192.5.6.30#53(a.gtld-servers.net) in 28 ms

www.example.com.   300     IN  A   203.0.113.10
;; Received 60 bytes from 203.0.113.53#53(ns1.example.com) in 18 ms
```

Each block is one level of the tree: root, TLD, authoritative. If a lookup fails, this command shows which level broke.

### Example 4: Windows tools

```powershell
PS C:\> nslookup www.example.com
```

```text
Server:  UnKnown
Address:  198.51.100.53

Non-authoritative answer:
Name:    www.example.com
Address:  203.0.113.10
```

"Non-authoritative answer" only means the reply came through a resolver, not directly from the zone's own server. It does not mean the answer is wrong. "UnKnown" only means the resolver's IP has no reverse name.

```powershell
PS C:\> Resolve-DnsName www.example.com -Type A
```

```text
Name                  Type   TTL   Section    IPAddress
----                  ----   ---   -------    ---------
www.example.com       A      300   Answer     203.0.113.10
```

After a browser or `ping` lookup, you can see the OS cache:

```powershell
PS C:\> ipconfig /displaydns
```

```text
    www.example.com
    ----------------------------------------
    Record Name . . . . . : www.example.com
    Record Type . . . . . : 1
    Time To Live  . . . . : 287
    Data Length . . . . . : 4
    Section . . . . . . . : Answer
    A (Host) Record . . . : 203.0.113.10
```

Record Type 1 is the number for an A record.

### Example 5: the hosts file wins, but only for some tools

You have a new test server at `192.168.1.50`. You add this line to `/etc/hosts`:

```text
192.168.1.50    www.example.com
```

`getent` uses the OS name lookup path. `dig` talks straight to a DNS server.

```bash
$ getent hosts www.example.com
192.168.1.50    www.example.com

$ dig +short @198.51.100.53 www.example.com
203.0.113.10
```

Both results are correct. They took different paths. The browser, `ping` and `curl` follow the OS path, unless a browser uses its own DoH.

On Windows, `ping` shows the same effect, and `nslookup` still shows `203.0.113.10`:

```text
C:\> ping -n 1 www.example.com

Pinging www.example.com [192.168.1.50] with 32 bytes of data:
```

### Example 6: move a website with little downtime

You must move `www.example.com` from `203.0.113.10` to `203.0.113.20`. The current TTL is 86400 (1 day).

1. **Day 1, 09:00.** Lower the TTL to 300. The IP stays the same.
2. **Wait at least 24 hours.** Old copies with the 86400 TTL must expire first.
3. **Day 2, 09:30.** Change the record to `203.0.113.20`. Keep the old server running.
4. **Wait a few minutes.** Within about 5 minutes, well-behaved caches ask again and get the new IP.
5. **Watch traffic.** When the old server is quiet, turn it off.
6. **Raise the TTL** back to a higher value, such as 3600.

Why this order? If you change the IP on Day 1 with no preparation, some users could keep the old IP for up to a day.

## Going Deeper

This section is optional at this level, but it helps you read packet captures.

### DNS message header

Every DNS message starts with a 12-byte header:

```text
 16 bits wide
+----------------------------------------------+
|              Transaction ID (16)             |
+--+--------+--+--+--+--+-------+--------------+
|QR| Opcode |AA|TC|RD|RA|   Z   |    RCODE     |
|1 |   4    |1 |1 |1 |1 |   3   |      4       |
+--+--------+--+--+--+--+-------+--------------+
|           QDCOUNT: questions (16)            |
+----------------------------------------------+
|            ANCOUNT: answers (16)             |
+----------------------------------------------+
|           NSCOUNT: authority (16)            |
+----------------------------------------------+
|           ARCOUNT: additional (16)           |
+----------------------------------------------+
```

| Field | Meaning |
|---|---|
| Transaction ID | Random number that matches a reply to its query |
| QR | 0 = query, 1 = response |
| AA | Authoritative answer |
| TC | Truncated: the reply did not fit, retry over TCP |
| RD | Recursion desired |
| RA | Recursion available |
| Z | Reserved. Two of these bits are now used by DNSSEC (AD and CD) |
| RCODE | Result code |

The random Transaction ID and random source port make it harder for an attacker to fake a reply. See [Common Network Attacks](../09-security/03-common-network-attacks.md).

### Common result codes

| RCODE | Name | Meaning | First thing to check |
|---|---|---|---|
| 0 | NOERROR | The query worked | If the answer is empty, the name exists but not that record type |
| 2 | SERVFAIL | The server could not complete it | Broken authoritative server, or DNSSEC failure |
| 3 | NXDOMAIN | The name does not exist | Spelling, or the record was never created |
| 5 | REFUSED | The server will not answer you | Resolver ACL, or querying the wrong server |

### UDP and TCP

DNS uses port 53 on both UDP and TCP.

- **UDP** is the normal choice for queries, because one small question and one small answer need no connection setup.
- Classic DNS over UDP was limited to 512 bytes. EDNS(0) (RFC 6891) lets the client say it can accept larger UDP replies.
- If a reply is too big, the server sets the TC flag and the client retries over **TCP**.
- TCP is also used for zone transfers, which copy a zone between authoritative servers.

RFC 7766 makes TCP support a requirement for DNS software, so blocking TCP port 53 breaks lookups that need large replies.

Encrypted forms exist: DNS over TLS uses TCP port 853, and DNS over HTTPS uses port 443. See [DNS Advanced](03-dns-advanced.md).

### Why 13 root identities?

The list of root servers had to fit in a 512-byte UDP reply. That limit gave 13 names. Anycast later allowed each name to grow into many machines without changing the list.

### Search domains

A device may have a search list. On Linux, this is the `search` line in `/etc/resolv.conf`:

```text
nameserver 198.51.100.53
search corp.example.com
```

If you type `ping fileserver`, the stub tries `fileserver.corp.example.com`. A name ending with a dot, such as `fileserver.`, skips the list. Search lists are a common source of extra queries, especially inside Kubernetes. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

### DNS on Cisco IOS

```text
! Cisco IOS: let the router resolve names
Router(config)# ip domain lookup
Router(config)# ip name-server 198.51.100.53
! A static entry, like a hosts file
Router(config)# ip host intranet.example.com 192.168.1.50
```

Older IOS releases write `ip domain-lookup` with a hyphen. Lookup is on by default. That is why a typo at the router prompt can freeze the console for a while, as IOS tries to resolve the word as a hostname. Many engineers set `no ip domain lookup` on lab routers.

## Common Mistakes

- **"DNS uses only UDP."** It uses UDP and TCP on port 53. TCP is needed for large replies and zone transfers.
- **"There are only 13 root servers."** There are 13 identities, and each one is many anycast machines.
- **"The root servers know every website's IP."** They know only which servers handle each TLD.
- **"The DNS server in my settings is authoritative."** It is almost always a recursive resolver or forwarder.
- **"Non-authoritative answer means wrong."** It only means the answer came through a resolver.
- **"DNS propagation pushes changes around the world."** Nothing is pushed. Old copies simply expire when their TTL ends.
- **"Lowering the TTL works at once."** Caches already holding the old record follow the old TTL. Lower it ahead of time.
- **"Flushing my PC fixes it for everyone."** It clears one cache on one device.
- **"nslookup and dig show what my browser sees."** They ask a DNS server directly and skip the hosts file. Browsers use the OS path or their own DoH.
- **"The second DNS server is tried if the first gives a bad answer."** Clients usually move on only after a timeout. NXDOMAIN from the first server is a valid answer, so the second is usually not asked. Mixing internal and public resolvers therefore gives random-looking failures.
- **"`example.com` and `www.example.com` are the same."** They are separate names with separate records.
- **"The internet is down."** If `ping 203.0.113.10` works but `ping www.example.com` fails, routing is fine and DNS is the likely problem.
- **"DNS resolves URLs."** It resolves only the host name. The path, port and `https://` part are not part of DNS.

## Interview Questions

**Q: What is DNS and why do we need it?**
A: DNS is a distributed, hierarchical system that maps names to data such as IP addresses. We need it because computers connect by IP, but people use names, and a name can stay the same when the IP changes. One central file cannot scale to the internet, so DNS splits the data across many servers.

**Q: Walk me through a DNS lookup for www.example.com.**
A: The browser checks its cache, then the OS checks its cache and the hosts file. If nothing is found, the stub resolver sends a recursive query to the configured resolver. The resolver asks a root server, which refers it to the .com servers. The .com server refers it to the authoritative servers for example.com. The authoritative server returns the IP and a TTL. The resolver caches it and answers the client.

**Q: What is the difference between a recursive resolver and an authoritative server?**
A: A recursive resolver finds answers for clients. It holds no zone of its own, it walks the tree and caches results. An authoritative server is the official source for a zone. It answers only for its own data and does not walk the tree.

**Q: What is the difference between a recursive and an iterative query?**
A: In a recursive query the client asks for the final answer, and the server does all the work. In an iterative query the server gives the best it knows, often a referral, and the asker follows up. Normally the stub to resolver step is recursive and the resolver to root, TLD and authoritative steps are iterative.

**Q: What is a stub resolver?**
A: It is the small DNS client in the operating system. It does not walk the tree. It sends the query to a configured recursive resolver and returns the result to the application. On many systems it also has a cache.

**Q: How many root servers are there?**
A: There are 13 root server identities, lettered A to M, run by 12 organisations. Each identity is many physical machines using anycast, so there are well over a thousand instances worldwide. The number 13 came from fitting the list in a 512-byte UDP reply.

**Q: What is TTL in DNS, and how do you choose a value?**
A: TTL is how many seconds a resolver may cache a record. A low TTL gives fast changes but more queries and more load. A high TTL gives fewer queries and better speed, but slow changes. I would use a high value for stable records and a low value for records that may fail over, and lower it before a planned move.

**Q: You changed a DNS record, but some users still reach the old server. Why?**
A: Their resolvers or devices still hold the old answer until its TTL expires. Nothing is pushed out, so each cache updates on its own schedule. I would check the authoritative server first with dig, then check the public resolvers, then look at the old TTL to know when it should clear.

**Q: Does DNS use TCP or UDP?**
A: Both, on port 53. Normal queries use UDP because it is fast and needs no setup. TCP is used when a reply is too large, shown by the TC flag, and for zone transfers. DNS software is required to support TCP.

**Q: What is the hosts file, and how does it relate to DNS?**
A: It is a local text file that maps names to IPs on one device. By default it is checked before DNS, so it can override public DNS. It is useful for testing, but it has no TTL or wildcards, and tools like dig and nslookup do not read it.

**Q: You can ping 8.8.8.8 but not google.com. What do you check?**
A: Connectivity works, so I suspect DNS. I check which DNS server the device uses with resolvectl or Get-DnsClientServerAddress. Then I query it directly with dig or nslookup, and try a public resolver to compare. I also check for a firewall blocking port 53 and for bad hosts file entries.

**Q: What is a glue record?**
A: It is an IP address included in a referral for a name server that sits inside the zone being delegated. If ns1.example.com serves example.com, you would need example.com to find it, which is a loop. The parent zone includes the address to break that loop.

## Quick Recap

- DNS maps names to IP addresses using a distributed tree: root, TLD, then authoritative servers.
- The stub resolver on your device sends a recursive query to a recursive resolver, which sends iterative queries down the tree.
- Referrals point to the next level. Only the authoritative server gives the final answer, marked with the AA flag.
- TTL sets how long an answer may be cached, in seconds. Caches sit in the browser, OS, router and resolver.
- DNS changes spread by cache expiry, so lower the TTL before a planned change.
- The hosts file overrides DNS on one device by default, but `dig` and `nslookup` skip it.
- DNS uses port 53 on UDP and TCP, and flushing clears only the cache you touched.
