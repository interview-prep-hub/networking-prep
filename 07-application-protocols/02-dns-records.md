# DNS Record Types

**Level:** Intermediate | **Prerequisites:** [DNS Basics](01-dns-basics.md), [IPv6 Basics](../04-ip-addressing/11-ipv6-basics.md)

## In One Line

A DNS record is one typed fact about a name, such as "this name has this IP address" or "mail for this name goes to that server". The record type tells the reader what kind of fact it is.

## Why It Matters

DNS answers more than "what is the IP of this website?". It also tells the world where to deliver email, which servers are in charge of a domain, who may issue certificates for it, and which protocols a site supports.

Each of those facts needs its own record type. Pick the wrong type, or break a rule of a type, and real things fail:

- A CNAME at the top of a domain can stop email for the whole company.
- A wrong MX record sends mail nowhere.
- A forgotten serial number update means secondary servers keep old data.
- A missing PTR record gets your mail server marked as spam.

Interviewers ask about records because the answers show if you have run real systems. Common questions are "why can a CNAME not sit at the root of a domain?", "what are the SOA fields?" and "how does reverse DNS work?".

## Simple Explanation

Think of a domain as a company's contact card. The card has labelled lines:

```text
Company: Example Ltd (example.com)
---------------------------------------------------
Street address (IPv4) ... 203.0.113.10          A
Street address (IPv6) ... 2001:db8:10::10       AAAA
Also known as ........... see "example.com"     CNAME
Send post to ............ mail1, then mail2     MX
Who keeps this card ..... ns1, ns2              NS
Notes ................... free text             TXT
Card version and rules .. version 2026103001    SOA
```

The label on each line is the record type. Without labels, you could not tell an address from a note.

**Where the analogy stops:** a real card sits in one place. DNS records are stored on several servers, and copies are kept for a limited time in caches all over the internet. Also, one line can hold many values, and a client asks for only the lines it needs.

## How It Works

### The parts of a record

Every record has the same five parts. A record is also called a resource record, or RR.

```text
www.example.com.   300   IN   A   203.0.113.10
|                  |     |    |   |
|                  |     |    |   +-- RDATA: the value itself
|                  |     |    +------ TYPE: what kind of fact
|                  |     +----------- CLASS: IN means internet
|                  +----------------- TTL: seconds a cache may keep it
+------------------------------------ NAME: who the fact is about
```

| Part | Meaning |
|---|---|
| Name | The domain name the record belongs to. Also called the owner name. |
| TTL | Time To Live. How many seconds a cache may reuse this answer before asking again. |
| Class | Almost always `IN` (internet). Other classes exist but are rare. |
| Type | The kind of record, such as A or MX. |
| RDATA | Record data. The value. Its format depends on the type. |

A few more terms appear in this chapter:

- **Zone:** the part of the DNS name tree that one group manages together, usually kept as one file. Example: everything in `example.com` that has not been handed to someone else.
- **Apex:** the very top name of a zone, here `example.com` itself. People also call it the root domain or naked domain.
- **Authoritative server:** a server that holds the real data for a zone.
- **Resolver:** the server that looks up names for clients and caches the answers. Your home router or your ISP usually points you to one.
- **RRset:** all records with the same name, class and type. Two A records for `example.com` form one RRset. DNS always returns a whole RRset, not a single record from it.

### Records at a glance

| Type | Purpose | Value example |
|---|---|---|
| A | Name to IPv4 address | `203.0.113.10` |
| AAAA | Name to IPv6 address | `2001:db8:10::10` |
| CNAME | This name is an alias of another name | `example.com.` |
| ALIAS | Provider feature: CNAME-like, works at the apex | Not a real DNS type |
| MX | Where to deliver email | `10 mail1.example.com.` |
| NS | Which servers are authoritative | `ns1.example.com.` |
| TXT | Free text, used for verification and policy | `"v=spf1 mx -all"` |
| SOA | Zone settings and version number | 7 fields |
| PTR | IP address back to a name | `mail1.example.com.` |
| SRV | Host and port for a service | `10 60 5060 sip1.example.com.` |
| CAA | Which certificate authorities may issue | `0 issue "letsencrypt.org"` |
| HTTPS | Connection details for a web service | `1 . alpn="h3,h2"` |
| SVCB | Same idea as HTTPS, for other services | `1 svc.example.com. port=8443` |

### A record

An A record maps a name to one IPv4 address. Its value is exactly 4 bytes (32 bits).

```text
example.com.   300   IN   A   203.0.113.10
```

A name can have many A records. This is a common way to spread clients across servers:

```text
example.com.   300   IN   A   203.0.113.10
example.com.   300   IN   A   203.0.113.11
```

Servers or resolvers may rotate the order. This is called round-robin DNS. It is a simple form of sharing. It is not real load balancing, because DNS does not check server health or load. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).

### AAAA record

An AAAA record (say "quad A") maps a name to one IPv6 address. Its value is 16 bytes (128 bits), four times the size of an IPv4 address. That is where the name comes from. It is defined in RFC 3596.

```text
example.com.   300   IN   AAAA   2001:db8:10::10
```

A and AAAA are separate questions. A query for type A never returns AAAA records. A dual-stack client (one with both IPv4 and IPv6) asks for both, then races the connections. This method is called Happy Eyeballs (RFC 8305).

Publish AAAA only when the service truly works over IPv6. A broken AAAA can cause slow connections or failures for IPv6 users.

### CNAME record and its rules

CNAME means canonical name. A CNAME says: "this name is only an alias. The real name is over there. Ask about that one."

```text
www.example.com.    300   IN   CNAME   example.com.
```

**Analogy:** a book index with the line "Cars: see Automobiles". The word "Cars" has no page numbers of its own. **Where it stops:** a resolver follows the pointer by itself and returns both steps. And a CNAME redirects every record type at that name, not only one topic.

Resolution works like this:

1. The client asks for the A record of `www.example.com`.
2. The server finds a CNAME to `example.com`.
3. The resolver then looks up the A record of `example.com`.
4. The client gets both records: the CNAME and the A.

**The rules. Memorise these:**

1. **A CNAME must be alone.** If a name has a CNAME, it can have no other data at that name. The only exception is DNSSEC records (the records that sign data, see [DNS Advanced](03-dns-advanced.md)). The rule comes from RFC 1034 and RFC 2181. The logic is simple: a CNAME says "I have no data of my own". An MX beside it would contradict that.
2. **No CNAME at the zone apex.** This follows from rule 1. The apex must have SOA and NS records, so a CNAME there would break rule 1.
3. **Only one CNAME per name.** A name cannot be an alias of two names.
4. **The target is a name, never an IP address.** `CNAME 203.0.113.10` is wrong. Use an A record.
5. **MX, NS and SRV targets must not be aliases.** They must point to names that have A or AAAA records directly. RFC 2181 states this for MX and NS, and RFC 2782 for SRV.
6. **Chains work but cost time.** `a` to `b` to `c` is legal, but each step can add a lookup. Loops are errors.
7. **A CNAME is not an HTTP redirect.** It carries no path, no port and no scheme. The browser still sends the original name in the `Host` header, and the TLS certificate must cover the original name.

### ALIAS, ANAME and CNAME flattening

People want `example.com` itself to point at a load balancer or a [CDN](../10-enterprise-and-datacenter/06-cdn.md) (content delivery network, a set of servers close to users) name, such as `lb.cdn.example.net`. Rule 2 forbids a CNAME there. DNS providers built a workaround.

**ALIAS is not a real DNS type.** It has no type number. It is a feature inside the provider's server. Names vary:

| Provider feature | Name used |
|---|---|
| Amazon Route 53 | Alias record |
| Azure DNS | Alias record set |
| Cloudflare | CNAME flattening |
| DNSimple, NS1 and others | ALIAS or ANAME |

The authoritative server resolves the target itself and returns plain A or AAAA records:

```text
Resolver            ns1.example.com            CDN name server
   |                       |                          |
   | A? example.com        |                          |
   |---------------------->|                          |
   |                       | A? lb.cdn.example.net    |
   |                       |------------------------->|
   |                       | A 198.51.100.80          |
   |                       |<-------------------------|
   | A 198.51.100.80       |                          |
   |<----------------------|                          |
```

The client sees a normal A record. It cannot tell an ALIAS was used. That is why it can sit beside SOA, NS and MX at the apex.

**Costs:**

- It only works inside that provider. You cannot move it as a zone file record.
- The address is resolved from the provider's location, not the user's. A CDN may then choose a less ideal edge server, unless the provider passes along client location data.
- The ANAME proposal at the IETF did not become a standard. The HTTPS record (below) is the standard path for this problem, but older clients do not use it.

| | CNAME | ALIAS | HTTPS AliasMode |
|---|---|---|---|
| Real DNS type | Yes | No | Yes (type 65) |
| Allowed at apex | No | Yes | Yes |
| Other records beside it | No | Yes | Yes |
| Who follows it | Resolver | Authoritative server | Client that supports it |
| Old clients work | Yes | Yes | No: also publish A and AAAA |

### MX record

MX means mail exchanger. It says which servers accept email for a domain. Each record has a preference number and a host name.

```text
example.com.   3600   IN   MX   10 mail1.example.com.
example.com.   3600   IN   MX   20 mail2.example.com.
```

**Analogy:** a company post room, with a backup post room if the first is closed. **Where it stops:** a real postman does not choose at random. If two MX records have the same preference, senders pick between them, which shares the load.

How a sending mail server uses it (RFC 5321):

1. Take the domain after the `@`. For `sara@example.com` that is `example.com`.
2. Ask for its MX records.
3. Try the **lowest** number first. Lower means more preferred.
4. If that server fails, try the next number.
5. If the domain has **no MX at all**, fall back to its A or AAAA record. This is called the implicit MX.

Key details:

- The target must be a host name, not an IP address, and must not be a CNAME.
- The preference is a 16-bit number (0-65535). Only the order matters. 10 and 20 work the same as 1 and 2.
- A domain that never receives mail can publish a null MX (RFC 7505): `example.com. IN MX 0 .`. The single dot means "no mail here". Senders then fail fast.

For the mail protocols, see [Email Protocols](10-email-protocols.md).

### NS record and glue

NS means name server. An NS record says which authoritative servers hold a zone.

```text
example.com.   86400   IN   NS   ns1.example.com.
example.com.   86400   IN   NS   ns2.example.com.
```

NS records appear in two places:

1. **In the parent zone.** The `.com` servers hold NS records for `example.com`. This is the delegation: the parent says "that part of the tree is managed over there". You set these at your registrar.
2. **In the child zone.** The `example.com` zone lists its own servers.

The two sets should match. If a listed server does not answer for the zone, that is called a lame delegation. It causes slow or failed lookups.

You can delegate your own subdomain the same way. Put NS records for `dev.example.com` in the `example.com` zone, and that subdomain becomes a separate zone, perhaps run by another team.

**Glue records.** There is a chicken-and-egg problem. To find `example.com`, a resolver must reach `ns1.example.com`. But to find `ns1.example.com`, it must ask the server for `example.com`, which is `ns1.example.com`.

The fix: the parent also stores the IP of the name server. This extra A or AAAA record is called a glue record.

```text
Resolver asks .com: "Where is example.com?"

.com answers:
  AUTHORITY:   example.com.      NS  ns1.example.com.
               example.com.      NS  ns2.example.com.
  ADDITIONAL:  ns1.example.com.  A   198.51.100.1     <-- glue
               ns2.example.com.  A   203.0.113.1      <-- glue
```

**Analogy:** a signpost says "ask Sara in the office". You do not know where the office is, so someone writes her phone number on the sign. **Where it stops:** glue is needed only when the name server's name sits inside the zone it serves. If `example.com` used `ns1.dnshost.example.net`, no glue would be needed, because that name can be found in a different zone.

Run at least two name servers, on different networks, so one failure does not take the whole domain offline.

### TXT record

A TXT record holds free text. DNS itself does not read it. Other systems give it meaning.

```text
example.com.   3600   IN   TXT   "v=spf1 mx -all"
```

A TXT value is one or more strings of up to 255 bytes each. Long values, such as DKIM keys, are split into several strings. The reader joins them together with no space between them.

Common uses:

| Use | Name | Example value |
|---|---|---|
| SPF: which servers may send mail for us | `example.com` | `"v=spf1 mx -all"` |
| DKIM: public key to check mail signatures | `s1._domainkey.example.com` | `"v=DKIM1; k=rsa; p=MIIB..."` |
| DMARC: what to do with failed mail | `_dmarc.example.com` | `"v=DMARC1; p=reject"` |
| Domain ownership proof | `example.com` | `"google-site-verification=abc123"` |
| Proof for automatic certificates (ACME, RFC 8555) | `_acme-challenge.example.com` | A one-time token |

The underscore names are normal DNS names. Underscores are not allowed in host names, so they mark data that is not a host (RFC 8552 covers this practice).

A name can have many TXT records. But it must have **only one SPF record**. Two records starting with `v=spf1` make SPF fail with an error. SPF is covered in [Email Protocols](10-email-protocols.md).

### SOA record and its fields

SOA means Start of Authority. Each zone has exactly one, at the apex. It holds the zone's version number and timers.

**Analogy:** the front page of a document: its author, its version, and how often readers should check for a new one. **Where it stops:** it also sets how long "not found" answers are remembered, which a front page does not do.

```text
example.com.  3600  IN  SOA  ns1.example.com. hostmaster.example.com. (
                             2026103001  ; SERIAL
                             7200        ; REFRESH
                             3600        ; RETRY
                             1209600     ; EXPIRE
                             300 )       ; MINIMUM
```

| Field | Example | Meaning |
|---|---|---|
| MNAME | `ns1.example.com.` | The primary server, where the master copy of the zone lives. |
| RNAME | `hostmaster.example.com.` | Admin email address. The first dot stands for `@`, so this is `hostmaster@example.com`. |
| SERIAL | `2026103001` | Zone version, a 32-bit number. It must go up after each change. |
| REFRESH | `7200` | Seconds between secondary checks for a new serial. |
| RETRY | `3600` | Seconds to wait before trying again after a failed check. |
| EXPIRE | `1209600` | Seconds after which a secondary stops answering if it still cannot reach the primary. 1209600 is 14 days. |
| MINIMUM | `300` | How long "not found" answers may be cached (RFC 2308). |

The timer values above are common examples. They are not a standard, and hosted DNS providers often set their own.

A secondary server is a server that holds a read-only copy of the zone. The SOA controls how it stays up to date:

```text
Primary (ns1)                        Secondary (ns2)
     |                                     |
     |  1. NOTIFY: "zone changed"          |
     |------------------------------------>|
     |  2. SOA query: "what is the serial?"|
     |<------------------------------------|
     |  3. SOA serial 2026103002           |
     |------------------------------------>|
     |                                     |  newer than
     |  4. Zone transfer request           |  2026103001
     |<------------------------------------|
     |  5. Zone data                       |
     |------------------------------------>|
```

NOTIFY (RFC 1996) is a push that makes this fast. Without it, the secondary only checks every REFRESH seconds. If a check fails, it retries every RETRY seconds. After EXPIRE seconds without success, it stops answering for the zone, because its data may be too old to trust.

Serial format is your choice. `YYYYMMDDnn` (date plus a counter) is a common habit, not a rule. Here `2026103001` means the first change on 30 October 2026. It fits in 32 bits, which holds up to 4294967295.

**Simple version vs fuller truth:** old documents describe MINIMUM as the lowest TTL for all records. RFC 2308 changed that. Today the time a "not found" answer is cached is the smaller of MINIMUM and the TTL of the SOA record itself.

### PTR record and reverse DNS

Normal lookups go from name to IP. Reverse DNS goes from IP to name, using a PTR (pointer) record.

**Analogy:** a reverse phone book, where you look up a number to find the owner. **Where it stops:** a forward entry and a reverse entry are separate records in separate zones, often run by different people. Nothing forces them to agree.

DNS names are read from most specific (left) to least specific (right). IP addresses are the opposite. So the address is flipped and placed under a special domain.

**IPv4:** reverse the four octets and add `in-addr.arpa.`

```text
IP address:   203.0.113.25
Reversed:     25.113.0.203
PTR name:     25.113.0.203.in-addr.arpa.

.  ---  arpa  ---  in-addr  ---  203  ---  0  ---  113  ---  25
                                           network part      host
```

**IPv6:** write the full address, reverse it one hex digit (nibble) at a time, and add `ip6.arpa.`

```text
IP address:   2001:db8::25
Full form:    2001:0db8:0000:0000:0000:0000:0000:0025
PTR name:     5.2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.
              8.b.d.0.1.0.0.2.ip6.arpa.
```

(That name is one line in real use. It is split here to fit.)

A PTR record then looks like this:

```text
25.113.0.203.in-addr.arpa.   3600   IN   PTR   mail1.example.com.
```

**Who controls it?** The holder of the IP block, not the owner of the domain. If your ISP or cloud provider owns `203.0.113.0/24`, the PTR lives in their reverse zone. You ask them to set it, or use their console if they offer a setting. Buying `example.com` gives you no power over PTR records.

Where PTR is used:

- **Mail:** many receiving servers reject or distrust mail from an IP without a PTR. They often check forward-confirmed reverse DNS (FCrDNS): the IP's PTR gives a name, and that name's A record must give the same IP.
- **Troubleshooting:** `traceroute` shows router names from PTR records.
- **Logs:** tools can show names instead of IPs.

### SRV record

An A record gives only an address. An SRV (service) record gives a host **and a port** for a named service, plus a way to share load. It is defined in RFC 2782.

**Analogy:** a reception desk that says "Sales is on floor 5, room 12. If that room is full, try room 14." **Where it stops:** only client software built to ask for SRV uses it. An ordinary client just ignores it.

The name has a fixed form: `_service._protocol.domain`.

```text
_sip._tcp.example.com.  3600 IN SRV  10  60  5060  sip1.example.com.
                                     |   |   |     |
                                     |   |   |     +-- target host
                                     |   |   +-------- port
                                     |   +------------ weight
                                     +---------------- priority
```

How a client picks a server:

1. Ask for the SRV records.
2. Take the group with the **lowest priority**.
3. Inside that group, choose at random in proportion to **weight**. Weights 60 and 40 give about 60 percent and 40 percent. Note the direction: higher weight means more traffic, while lower priority means more preferred.
4. Look up the target's A or AAAA record and connect to the port.
5. If that fails, try other servers, then the next priority.

The target must not be a CNAME. A target of `.` means the service is not offered at this domain.

Real uses:

- **Active Directory:** clients find domain controllers with names such as `_ldap._tcp.dc._msdcs.corp.example.com`. See [Directory and Authentication Protocols](12-directory-and-authentication-protocols.md).
- **VoIP (voice calls over IP):** the SIP call protocol uses `_sip._tcp` and `_sip._udp`.
- **Chat:** the XMPP chat protocol uses `_xmpp-client._tcp`.
- **Kubernetes:** cluster DNS creates SRV records for named ports of a service. See [Kubernetes Networking](../11-cloud-and-containers/08-kubernetes-networking.md).

Web browsers do not use SRV to find websites. HTTP was deployed long before SRV existed, and it never adopted it. The HTTPS record fills that gap.

### CAA record

CAA means Certification Authority Authorization (RFC 8659). It lists which certificate authorities may issue TLS certificates for a domain. A certificate authority, or CA, is a company trusted to sign certificates. See [TLS and HTTPS](07-tls-and-https.md).

**Analogy:** a note at the bank: "Only accept documents stamped by this notary." **Where it stops:** a bank checks on every visit, but a CAA is checked only when a certificate is being issued. Browsers do not read CAA records at all. A certificate that already exists keeps working.

```text
example.com.  3600  IN  CAA  0 issue "letsencrypt.org"
example.com.  3600  IN  CAA  0 issuewild ";"
example.com.  3600  IN  CAA  0 iodef "mailto:security@example.com"
```

Each record has three parts:

| Part | Meaning |
|---|---|
| Flags | `0` is normal. `128` is the critical bit: a CA that does not understand this tag must refuse to issue. |
| Tag | `issue`, `issuewild` or `iodef`. |
| Value | Depends on the tag. |

| Tag | Meaning |
|---|---|
| `issue` | This CA may issue certificates for this name. |
| `issuewild` | This CA may issue wildcard certificates (`*.example.com`). If absent, `issue` applies to wildcards too. |
| `iodef` | Where a CA may report a request that broke the policy. |

The value `";"` means "no CA at all". So the second record above bans wildcard certificates.

CAs must check CAA before issuing. The CA/Browser Forum, the group that sets rules for public CAs, has required this since September 2017.

The check climbs the tree. For `blog.shop.example.com`, the CA looks at that name, then `shop.example.com`, then `example.com`, then `com`. It stops at the first name that has CAA records. If it finds none, any CA may issue.

### HTTPS and SVCB records

SVCB (type 64, Service Binding) and HTTPS (type 65) are defined in RFC 9460. HTTPS is SVCB's version for websites. These records solve three problems:

1. **The apex problem.** You can point `example.com` to another name, because this is a new type that may sit beside SOA and NS.
2. **Slow protocol discovery.** A client learns about HTTP/3 before it connects. Without this, a server can only say so in the `Alt-Svc` HTTP header, which the client sees after its first connection.
3. **Extra connection data.** The record can carry ports, address hints and the key for Encrypted Client Hello.

**Analogy:** a shop sign: "We take cards, the fast lane is at door 2, and the back door is open." You know this before you walk in. **Where it stops:** a client that does not understand the sign walks in the old way, which is why A and AAAA records must stay.

The format is: priority, target, then parameters.

```text
example.com. 300 IN HTTPS 1 . alpn="h3,h2" ipv4hint=203.0.113.10
```

| Field | Meaning |
|---|---|
| SvcPriority | `0` means AliasMode. Above 0 means ServiceMode. In ServiceMode, lower numbers are preferred. |
| TargetName | The host to connect to. A single `.` in ServiceMode means "the same name as this record". |
| SvcParams | Key and value settings. |

**The two modes:**

```text
AliasMode: point this name at another one. Works at the apex.
  example.com.  300 IN HTTPS 0 lb.cdn.example.net.

ServiceMode: publish connection details.
  lb.cdn.example.net. 300 IN HTTPS 1 . alpn="h3,h2"
```

| SvcParam | Meaning |
|---|---|
| `alpn` | Application protocols offered, such as `h3` (HTTP/3) and `h2` (HTTP/2). See [HTTP Versions](06-http-versions.md). |
| `port` | Use this port instead of the default. |
| `ipv4hint`, `ipv6hint` | Addresses to try, which can save a lookup. They are only hints. |
| `ech` | Public key for Encrypted Client Hello, which hides the site name in the TLS handshake. |
| `no-default-alpn`, `mandatory` | Advanced controls over what a client must support. |

A usable HTTPS record also tells supporting clients that the site should be reached over `https://`, similar to HSTS (HTTP Strict Transport Security, a rule that forces HTTPS).

SVCB does the same job for other services. One real example: Discovery of Designated Resolvers (RFC 9462) uses SVCB records at `_dns.resolver.arpa` to find encrypted DNS servers.

Major browsers use HTTPS records, but details differ by browser, operating system and whether encrypted DNS is on. Treat it as an improvement, not as a replacement for A and AAAA.

### How to look up each record

`dig` is the usual tool on Linux and macOS. On Windows, use `Resolve-DnsName` in PowerShell, or `nslookup`.

| Record | Linux or macOS | Windows PowerShell |
|---|---|---|
| A | `dig example.com A +short` | `Resolve-DnsName example.com -Type A` |
| AAAA | `dig example.com AAAA +short` | `Resolve-DnsName example.com -Type AAAA` |
| CNAME | `dig www.example.com CNAME +short` | `Resolve-DnsName www.example.com -Type CNAME` |
| ALIAS | Cannot be seen. Query A and AAAA. | Cannot be seen. Query A and AAAA. |
| MX | `dig example.com MX +short` | `Resolve-DnsName example.com -Type MX` |
| NS | `dig example.com NS +short` | `Resolve-DnsName example.com -Type NS` |
| TXT | `dig example.com TXT +short` | `Resolve-DnsName example.com -Type TXT` |
| SOA | `dig example.com SOA +short` | `Resolve-DnsName example.com -Type SOA` |
| PTR | `dig -x 203.0.113.25 +short` | `Resolve-DnsName 203.0.113.25 -Type PTR` |
| SRV | `dig _sip._tcp.example.com SRV +short` | `Resolve-DnsName _sip._tcp.example.com -Type SRV` |
| CAA | `dig example.com CAA +short` | Use `dig` (see below) |
| HTTPS | `dig example.com HTTPS +short` | Use `dig` (see below) |

The classic Windows form also works:

```powershell
# Windows
nslookup -type=mx example.com
```

Built-in Windows tools do not reliably know CAA or HTTPS by name, and support depends on the version. The simple path is `dig` inside WSL (Windows Subsystem for Linux). Older versions of `dig` may not know `HTTPS` either. Then use the type number: `dig example.com TYPE65`.

To skip caches and ask the zone's own server, name it with `@`:

```bash
# Linux or macOS
dig @ns1.example.com example.com SOA
```

More options are in [DNS Tools](../12-troubleshooting-and-tools/04-dns-tools.md).

## Example

### A sample zone file

A zone file is the text format for a zone. Many DNS servers, such as BIND, read it directly. Hosted DNS panels hide it behind a web form, but the same fields are there.

Syntax rules first:

- `;` starts a comment.
- `$ORIGIN` sets the base name. A name **without** a final dot gets this added.
- `$TTL` sets the default TTL for records that do not give their own.
- `@` means the origin, here `example.com.`.
- A name **with** a final dot is complete. It is called a fully qualified domain name (FQDN).
- Parentheses let one record span several lines.

```text
$ORIGIN example.com.
$TTL 3600

; --- Zone settings ---------------------------------------------
@    IN  SOA  ns1.example.com. hostmaster.example.com. (
              2026103001  ; serial
              7200        ; refresh (2 hours)
              3600        ; retry (1 hour)
              1209600     ; expire (14 days)
              300 )       ; negative cache time (5 minutes)

; --- Name servers ----------------------------------------------
@    IN  NS   ns1.example.com.
@    IN  NS   ns2.example.com.
ns1  IN  A    198.51.100.1
ns2  IN  A    203.0.113.1

; --- Website ---------------------------------------------------
@     300  IN  A      203.0.113.10
@     300  IN  AAAA   2001:db8:10::10
@     300  IN  HTTPS  1 . alpn="h3,h2" ipv4hint=203.0.113.10
www   300  IN  CNAME  example.com.
shop  300  IN  CNAME  shops.cdn.example.net.

; --- Wildcard for test sites -----------------------------------
*.dev      IN  A      203.0.113.30

; --- Mail ------------------------------------------------------
@      IN  MX   10 mail1.example.com.
@      IN  MX   20 mail2.example.com.
mail1  IN  A    203.0.113.25
mail2  IN  A    198.51.100.25

; --- Text records ----------------------------------------------
@       IN  TXT  "v=spf1 mx -all"
_dmarc  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

; --- Certificate policy ----------------------------------------
@    IN  CAA  0 issue "letsencrypt.org"
@    IN  CAA  0 issuewild ";"
@    IN  CAA  0 iodef "mailto:security@example.com"

; --- VoIP service ----------------------------------------------
_sip._tcp  IN  SRV  10 60 5060 sip1.example.com.
_sip._tcp  IN  SRV  10 40 5060 sip2.example.com.
sip1       IN  A    192.0.2.50
sip2       IN  A    192.0.2.51
```

Things to notice:

- `ns1` and `ns2` sit inside the zone, so the `.com` parent must hold glue for them.
- `www` is a CNAME, so no other record is at `www`.
- The apex has A, AAAA, HTTPS, MX, TXT, CAA, SOA and NS. It could not hold all of these with a CNAME.
- `shops.cdn.example.net` is an outside name, so it ends with a dot. `example.net` is reserved for examples, like `example.com`.
- No ALIAS appears, because ALIAS does not exist in a standard zone file. It lives in the provider's own system.

### A sample reverse zone

If you hold `203.0.113.0/24`, or your provider set this up for you, the reverse zone is a separate file:

```text
$ORIGIN 113.0.203.in-addr.arpa.
$TTL 3600

@    IN  SOA  ns1.example.com. hostmaster.example.com. (
              2026103001 7200 3600 1209600 300 )
@    IN  NS   ns1.example.com.
@    IN  NS   ns2.example.com.

10   IN  PTR  example.com.
25   IN  PTR  mail1.example.com.
```

The address `198.51.100.25` of `mail2` needs a PTR too. That record lives in the reverse zone of whoever holds that block.

### Checking the file before you load it

BIND ships a checker. Always run it first.

```bash
# Linux, BIND tools
$ named-checkzone example.com db.example.com
zone example.com/IN: loaded serial 2026103001
OK
```

Now break the rules. Add `@ IN CNAME lb.cdn.example.net.` to the apex:

```bash
# Linux, BIND tools
$ named-checkzone example.com db.example.com
dns_master_load: db.example.com:23: example.com: CNAME and other data
zone example.com/IN: loading from master file db.example.com failed: CNAME and other data
zone example.com/IN: not loaded due to errors.
```

The error text states rule 1 directly.

### Looking up the records

All output below comes from the sample zone. The real `example.com` has different records. Try the commands on a domain you own. The option `+noall +answer` tells `dig` to show only the answer.

**A CNAME being followed:**

```bash
# Linux or macOS
$ dig www.example.com A +noall +answer
www.example.com.        300     IN      CNAME   example.com.
example.com.            300     IN      A       203.0.113.10
```

**MX on both systems:**

```bash
# Linux or macOS
$ dig example.com MX +noall +answer
example.com.            3600    IN      MX      10 mail1.example.com.
example.com.            3600    IN      MX      20 mail2.example.com.
```

```powershell
# Windows PowerShell
PS> Resolve-DnsName example.com -Type MX

Name                Type   TTL   Section    NameExchange         Preference
----                ----   ---   -------    ------------         ----------
example.com         MX     3600  Answer     mail1.example.com    10
example.com         MX     3600  Answer     mail2.example.com    20
```

```powershell
# Windows, nslookup
PS> nslookup -type=mx example.com
Server:  router.local
Address:  192.168.1.1

Non-authoritative answer:
example.com     MX preference = 10, mail exchanger = mail1.example.com
example.com     MX preference = 20, mail exchanger = mail2.example.com
```

"Non-authoritative" means the answer came from a resolver, not from the zone's own server. That is normal.

**SOA, short form.** The fields print in order:

```bash
# Linux or macOS
$ dig example.com SOA +short
ns1.example.com. hostmaster.example.com. 2026103001 7200 3600 1209600 300
```

**NS at the parent, showing glue.** This asks a `.com` server directly, and turns off recursion (`+norecurse`), so it shows what the parent stores:

```bash
# Linux or macOS
$ dig @a.gtld-servers.net example.com NS +norecurse +noall +authority +additional
example.com.            172800  IN      NS      ns1.example.com.
example.com.            172800  IN      NS      ns2.example.com.
ns1.example.com.        172800  IN      A       198.51.100.1
ns2.example.com.        172800  IN      A       203.0.113.1
```

**Reverse lookup, then the forward check (FCrDNS):**

```bash
# Linux or macOS
$ dig -x 203.0.113.25 +noall +answer
25.113.0.203.in-addr.arpa. 3600 IN      PTR     mail1.example.com.

$ dig mail1.example.com A +short
203.0.113.25
```

The two match, so this mail server passes the check.

```powershell
# Windows PowerShell
PS> Resolve-DnsName 203.0.113.25 -Type PTR

Name                           Type   TTL   Section    NameHost
----                           ----   ---   -------    --------
25.113.0.203.in-addr.arpa      PTR    3600  Answer     mail1.example.com
```

**TXT, SRV, CAA and HTTPS:**

```bash
# Linux or macOS
$ dig example.com TXT +short
"v=spf1 mx -all"

$ dig _sip._tcp.example.com SRV +short
10 60 5060 sip1.example.com.
10 40 5060 sip2.example.com.

$ dig example.com CAA +short
0 issue "letsencrypt.org"
0 issuewild ";"
0 iodef "mailto:security@example.com"

$ dig example.com HTTPS +short
1 . alpn="h3,h2" ipv4hint=203.0.113.10
```

**A name that does not exist.** The SOA comes back in the authority section:

```bash
# Linux or macOS
$ dig new.example.com A
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 5121
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; AUTHORITY SECTION:
example.com.  300  IN  SOA  ns1.example.com. hostmaster.example.com. 2026103001 7200 3600 1209600 300
```

(Other output lines are removed here.) The TTL is 300, the smaller of the SOA's TTL (3600) and MINIMUM (300). If you now create `new.example.com`, this resolver may still say "does not exist" for up to 300 seconds.

### Worked task: moving a website safely

The site at `example.com` moves from `203.0.113.10` to `203.0.113.40`. Suppose its A record still has a TTL of 3600, not the 300 shown in the sample zone. You want a fast switch.

1. **Lower the TTL first.** Change the TTL to 300 and raise the serial. Do this at least 3600 seconds before the move, because caches may hold the old TTL for that long.
2. **Keep both servers running.**
3. **Change the A record** to `203.0.113.40` and raise the serial again. Also update the `ipv4hint` in the HTTPS record, and the AAAA record if the address changes.
4. **Check the source directly**, so caches do not fool you:

   ```bash
   # Linux or macOS
   dig @ns1.example.com example.com A +short
   dig @ns2.example.com example.com A +short
   ```

5. **Wait.** Well-behaved caches drop the old answer within 300 seconds. Some clients and old resolvers hold it longer, so keep the old server up for a while.
6. **Raise the TTL** again once things are stable. Higher TTLs mean less query load and more time to ride out a DNS outage.

## Going Deeper

### Record layout on the wire

Every record in a DNS message uses this layout (RFC 1035):

| Field | Size | Meaning |
|---|---|---|
| NAME | Variable | Owner name. May use compression, a pointer to an earlier copy of the name. |
| TYPE | 16 bits | Record type number. |
| CLASS | 16 bits | 1 means IN. |
| TTL | 32 bits | Seconds. RFC 2181 limits it to 0 through 2147483647. |
| RDLENGTH | 16 bits | Length of RDATA in bytes. |
| RDATA | RDLENGTH bytes | The value, in a format set by TYPE. |

The value part differs by type:

| Type | RDATA layout |
|---|---|
| A | 4 bytes |
| AAAA | 16 bytes |
| CNAME, NS, PTR | One domain name |
| MX | 16-bit preference, then a name |
| TXT | One or more strings, each with a 1-byte length, so at most 255 bytes each |
| SOA | MNAME, RNAME, then five 32-bit numbers |
| SRV | 16-bit priority, weight and port, then a name |
| CAA | 1-byte flags, 1-byte tag length, tag, then value |
| HTTPS, SVCB | 16-bit priority, target name, then parameters (16-bit key, 16-bit length, value) |

### Type numbers

Tools show these numbers when they do not know a type's name. RFC 3597 defines this text form, such as `TYPE65`.

| Number | Type | Number | Type |
|---|---|---|---|
| 1 | A | 35 | NAPTR |
| 2 | NS | 39 | DNAME |
| 5 | CNAME | 43 | DS |
| 6 | SOA | 46 | RRSIG |
| 12 | PTR | 48 | DNSKEY |
| 15 | MX | 257 | CAA |
| 16 | TXT | 52 | TLSA |
| 28 | AAAA | 64 | SVCB |
| 33 | SRV | 65 | HTTPS |

### RRset rules

- All records in an RRset must have the same TTL (RFC 2181). If they differ, the behaviour is unreliable.
- Exact duplicates are removed. An RRset is a set, so there is no use in listing the same IP twice.
- Order inside an RRset carries no meaning. Do not depend on "the first A record".

### Two kinds of "nothing"

These are different, and interviewers like the difference:

| Result | Meaning | Status |
|---|---|---|
| NXDOMAIN | The name does not exist at all. | `NXDOMAIN` |
| NODATA | The name exists, but has no record of the asked type. | `NOERROR` with an empty answer |

Asking for the AAAA record of `mail1.example.com` gives NODATA. Asking about `nothing.example.com` gives NXDOMAIN. Both are cached using the SOA rule, and both carry the SOA in the authority section.

### Wildcards

A wildcard (RFC 4592) starts with `*` and answers for names that are not defined.

```text
*.dev   IN  A   203.0.113.30
```

- `a.dev.example.com` matches.
- `a.b.dev.example.com` also matches, if `b.dev.example.com` does not exist. **DNS wildcards can match more than one label.** A TLS wildcard certificate covers only one label, so the two rules differ.
- `dev.example.com` itself does not match. It returns NODATA.
- A name that exists is never covered. If `api.dev.example.com` has only a TXT record, an A query for it returns NODATA. The wildcard does not fill in the gap.

### Large answers

A long TXT or a big record set can pass the size a UDP response can safely hold. EDNS is an extension that allows DNS messages over 512 bytes on UDP, and many resolvers offer a size of 1232 bytes to avoid IP fragmentation. Larger answers are cut short with the TC (truncated) flag, and the client retries over TCP port 53. So a firewall that blocks TCP 53 can break big DNS answers. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### Classless reverse delegation

Reverse zones split on dot boundaries, so a block of `/24`, `/16` or `/8` delegates cleanly. A smaller block has no such boundary. RFC 2317 solves this with CNAMEs. If a customer holds `203.0.113.0/26`, the provider's zone can hold:

```text
25   IN  CNAME  25.0/26.113.0.203.in-addr.arpa.
```

The customer then runs the zone `0/26.113.0.203.in-addr.arpa.` and places the PTR there. Some providers use `0-63` in place of `0/26`.

### Other record types worth knowing

| Type | Purpose |
|---|---|
| DNAME | Aliases a whole subtree, not one name (RFC 6672). |
| DS, DNSKEY, RRSIG, NSEC, NSEC3 | DNSSEC signing and proof. See [DNS Advanced](03-dns-advanced.md). |
| TLSA | Ties a TLS certificate to a name, for DANE (RFC 6698). |
| SSHFP | Publishes SSH host key fingerprints. |
| NAPTR | Rewrite rules, used in telephony and with SRV. |
| SPF (type 99) | Dead. RFC 7208 removed it. SPF policy goes in TXT. |

### ANY queries

Type ANY (255) is a query type, not a record type. Many people think it returns "all records". It does not promise that. RFC 8482 lets servers give a small answer, because ANY was abused in amplification attacks. Ask for each type on its own.

### Small rules that catch people

- Names are not case sensitive: `Example.COM` equals `example.com`.
- A full name can be at most 255 bytes on the wire, and each label at most 63 bytes.
- A CAA check on a CNAME name also follows the alias, so a CDN's policy can affect your certificate.
- CAA can go further with `accounturi` and `validationmethods` parameters (RFC 8657), which limit issuance to a given account or method.

## Common Mistakes

- **Putting a CNAME at the apex.** It breaks SOA, NS and MX at that name. Use ALIAS, A and AAAA, or an HTTPS record.
- **Forgetting the final dot.** With `$ORIGIN example.com.`, the line `www IN CNAME example.com` makes `example.com.example.com.`, which does not exist.
- **Forgetting to raise the serial.** The primary has the new data, but secondaries never fetch it.
- **Thinking a higher MX number means higher priority.** Lower numbers are preferred.
- **Pointing MX or NS at a CNAME.** Some systems tolerate it, but the rules forbid it, and it can cause delivery failures.
- **Pointing MX at an IP address.** The target must be a host name.
- **Publishing two SPF records.** SPF needs one record. Merge them into a single `v=spf1` line.
- **Expecting a CNAME to redirect.** It changes the DNS answer only. The browser keeps the original name, and the certificate must match it.
- **Looking for the PTR at your domain's DNS host.** The owner of the IP block controls it.
- **Lowering the TTL at the moment of the change.** Caches still hold the old, longer TTL. Lower it at least one old TTL ahead.
- **Blaming DNS when the cached answer is correct for now.** Check the authoritative server directly with `dig @ns1...` before you change anything.
- **Believing CAA protects against a stolen certificate.** It is checked only at issue time, by CAs, not by browsers.
- **Removing A and AAAA after adding an HTTPS record.** Clients that do not support it would lose the site.
- **Pasting a long DKIM key as one 300-byte string.** A single TXT string has a 255-byte limit. Split it into several strings.

## Interview Questions

**Q: What is the difference between an A and a CNAME record?**

A: An A record maps a name directly to an IPv4 address. A CNAME maps a name to another name, and the resolver then looks up that name. Use a CNAME when the target's address can change and someone else manages it, such as a CDN.

**Q: Why can you not put a CNAME at the root of a domain?**

A: A CNAME must be the only record at its name. The zone apex must have SOA and NS records, and usually MX and TXT too. So a CNAME there breaks the rule. Providers offer ALIAS or CNAME flattening as a workaround, and the HTTPS record in AliasMode is the standard option for clients that support it.

**Q: What is an ALIAS record, and how is it different from a CNAME?**

A: ALIAS is a provider feature, not a real DNS type. The authoritative server resolves the target itself and returns normal A or AAAA records, so it can sit at the apex beside other records. A CNAME is returned to the resolver, which follows it.

**Q: How do MX records work? What happens if a domain has none?**

A: The sender looks up the MX records for the domain after the at sign and tries the lowest preference first. If that fails, it tries the next. Equal preferences share the load. With no MX, the sender falls back to the domain's A or AAAA record. A null MX, with target dot, says the domain takes no mail.

**Q: What are the fields in an SOA record?**

A: MNAME is the primary server. RNAME is the admin email with the first dot standing for the at sign. SERIAL is the zone version. REFRESH is how often secondaries check. RETRY is the wait after a failed check. EXPIRE is when a secondary stops serving. MINIMUM sets how long negative answers are cached.

**Q: What is a glue record, and when do you need one?**

A: Glue is an A or AAAA record for a name server, stored in the parent zone. You need it when the name server's name is inside the zone it serves, such as ns1.example.com for example.com. Without it, finding the server would need the server itself.

**Q: How does reverse DNS work, and who controls it?**

A: The address is reversed and placed under in-addr.arpa for IPv4 or ip6.arpa for IPv6, and a PTR record there gives the name. So 203.0.113.25 becomes 25.113.0.203.in-addr.arpa. The holder of the IP block controls it, usually the ISP or cloud provider, not the domain owner. Mail servers use it heavily as an anti-spam check.

**Q: What does an SRV record give you that an A record does not?**

A: It gives a port as well as a host, plus priority for failover and weight for load sharing. Clients find it under a name like `_sip._tcp.example.com`. Active Directory depends on SRV records to locate domain controllers.

**Q: What does a CAA record do? Does it stop a browser from accepting a bad certificate?**

A: It lists which CAs may issue certificates for a domain. CAs must check it before issuing. Browsers do not check it, so it does nothing against a certificate that already exists. It reduces the risk of wrong issuance, and it does not replace monitoring such as Certificate Transparency, the public logs of issued certificates.

**Q: What problem do HTTPS records solve?**

A: They give the client connection details in DNS, before it connects. That includes HTTP/3 support through alpn, address hints, and the key for Encrypted Client Hello. AliasMode also allows the apex to point to another name in a standard way. A and AAAA records must stay for older clients.

**Q: You changed an A record, but some users still reach the old server. Why?**

A: Resolvers and clients cache the old answer until its TTL ends, and some hold it longer. Secondaries may also be behind if the serial was not raised. I would query each authoritative server directly, compare serials, then check the TTL left on the resolver. Next time I would lower the TTL ahead of the change.

**Q: A new subdomain shows NXDOMAIN for some users, even though the record exists. Why?**

A: That is negative caching. A resolver asked before the record was created and stored "does not exist". It keeps that for the smaller of the SOA MINIMUM and the SOA record's TTL. It clears on its own, or the resolver's cache can be flushed.

## Quick Recap

- A record has a name, TTL, class, type and value. The type defines what the value means.
- A and AAAA give addresses. CNAME gives another name, must be alone at its name, and cannot sit at the apex.
- ALIAS is a provider trick that returns normal A and AAAA records. HTTPS AliasMode is the standard path for supporting clients.
- MX picks mail servers, lowest number first. NS delegates a zone, and glue breaks the loop when the name server is inside that zone.
- TXT carries SPF, DKIM, DMARC and ownership proofs. SOA carries the serial, the sync timers and the negative cache setting.
- PTR maps IP to name under in-addr.arpa or ip6.arpa, and the IP block's holder controls it.
- SRV adds port, priority and weight. CAA limits who issues certificates, checked by CAs only. HTTPS and SVCB publish connection details before the first connection.
