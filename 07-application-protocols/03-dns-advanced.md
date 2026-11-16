# DNS Advanced

**Level:** Advanced | **Prerequisites:** [DNS Basics](01-dns-basics.md), [DNS Record Types](02-dns-records.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md)

## In One Line

Advanced DNS is about how DNS moves large answers, copies zones between servers, proves answers are real, hides queries from watchers, gives different answers to different users, and survives attacks and migrations.

## Why It Matters

Basic DNS turns a name into an address. That is the easy part. Real systems have harder problems.

- A signed answer can be 2,000 bytes, but the original DNS limit on UDP was 512 bytes.
- A company needs two or more name servers, and they must hold the same data.
- Plain DNS has no signature. A forged reply can send a whole bank's customers to the wrong server.
- Anyone on the network path can read plain DNS queries.
- A website move can fail for a day if nobody lowered the TTL first.
- A firewall that allows only UDP 53 will break DNS in ways that look random.

Interviewers ask about these topics because they separate people who memorised "DNS uses port 53" from people who have run it. Typical questions are "When does DNS use TCP?", "How does DNSSEC work?", "What was the Kaminsky attack?" and "How do you move a website with no downtime?"

## Simple Explanation

A few words appear in every section, so here they are in plain language.

- **Client (stub resolver):** the small DNS code in your laptop or phone. It only asks questions.
- **Recursive resolver:** the server that does the work for clients. It walks the DNS tree and remembers answers. Examples are 8.8.8.8 and your company DNS server.
- **Authoritative server:** the server that holds the official data for a domain.
- **Zone:** the part of the DNS tree one group manages, stored as a set of records. `example.com` is a zone.
- **TTL (Time To Live):** how many seconds a resolver may keep an answer.
- **Cache:** the short-term memory of a resolver.

Now think of DNS as a large phone directory service.

- Most questions are short, so they go by **postcard** (UDP). Big answers need a **phone call** (TCP).
- The head office keeps the master directory. Branch offices keep **copies** (primary and secondary servers).
- A criminal can post a fake answer card. So the directory adds a **notary stamp** (DNSSEC).
- Postcards can be read by anyone who handles them. So some people use a **sealed envelope** (DoT and DoH).
- The receptionist may give staff one number and the public another (split-horizon).
- A national helpline can send you to your nearest branch (GeoDNS and anycast).
- Every answer has a **use-by date** (TTL). If you move office, you must plan around that date.

The analogy stops being true in one big way: DNS has no single head office. Control is split across millions of zones, and each one is run separately.

## How It Works

### 1. DNS Over UDP and TCP on Port 53

Every DNS server listens on **port 53** for both UDP and TCP. A port is a number that tells the operating system which program should get the data. See [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md).

**UDP** is the default. It sends one packet and gets one packet back. There is no connection setup, so it is fast and cheap for a busy server.

**TCP** needs a three-way handshake first, which costs one extra round trip. It is reliable and can carry large data. See [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md).

Analogy: UDP is a postcard. TCP is a phone call. You do not phone a shop to ask the time they open if a postcard will do. The analogy stops being true here: a real postcard takes days, while a UDP query often takes under 20 ms.

The original DNS standard (RFC 1035) limited a UDP message to **512 bytes**. If the answer does not fit, the server sets the **TC bit** (TC means truncated, which means "cut short") and the client must ask again over TCP.

```text
 Client                                  Server
   |  UDP: TXT example.com?                 |
   |--------------------------------------->|
   |  UDP: reply, TC=1, no full answer      |
   |<---------------------------------------|
   |  TCP: SYN, SYN-ACK, ACK                |
   |<======================================>|
   |  TCP: TXT example.com?                 |
   |--------------------------------------->|
   |  TCP: full answer, 1,874 bytes         |
   |<---------------------------------------|
```

The header of every DNS message carries these flags. TC, AD and CD matter most here.

```text
 Bit:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
      +--+-----------+--+--+--+--+--+--+--+-----------+
      |QR|  Opcode   |AA|TC|RD|RA| Z|AD|CD|  RCODE    |
      +--+-----------+--+--+--+--+--+--+--+-----------+
```

| Flag | Name | Meaning |
|---|---|---|
| QR | Query/Response | 0 is a question, 1 is an answer |
| AA | Authoritative Answer | The reply came from the official server |
| TC | Truncated | The reply was cut short, ask again over TCP |
| RD | Recursion Desired | The client asks the server to do the full lookup |
| RA | Recursion Available | The server offers that service |
| AD | Authenticated Data | The resolver says DNSSEC checks passed |
| CD | Checking Disabled | The client says "do not check DNSSEC for me" |

**When DNS uses TCP:**

| Case | Why |
|---|---|
| Reply is too big for UDP | The server set TC=1, so the client retries over TCP |
| AXFR zone transfer | AXFR must use TCP (RFC 5936) |
| Most IXFR zone transfers | Can start on UDP, but large ones need TCP, and most software uses TCP |
| DNSSEC keys and signatures | DNSKEY replies are often larger than the UDP limit |
| Rate limiting | A server under attack may send TC=1 so that real clients prove themselves over TCP |
| DNS over TLS | DoT runs on TCP port 853, not 53 |
| Client setting | An admin or tool can force it, such as `dig +tcp` |

On TCP, each message starts with a **2-byte length field**. So a DNS message can never be larger than **65,535 bytes**. A client can send many queries on one TCP connection, and the server may answer out of order. The ID number in each message matches answers to questions (RFC 7766).

TCP is not optional. RFC 7766 says general-purpose DNS software must support both UDP and TCP. RFC 9210 tells operators to keep TCP working. "TCP 53 is only for zone transfers" is an old myth. Blocking TCP 53 breaks real lookups.

### 2. EDNS0: Making UDP Answers Bigger

**EDNS0 (Extension Mechanisms for DNS, version 0, RFC 6891)** lets a client say how large a UDP reply it can accept. It was needed because the DNS header had no spare room for new features.

It works by adding one fake record to the message, called the **OPT pseudo-record**. "Pseudo" means it is not real zone data. It lives only for one message, and it is never cached.

Analogy: you add a note to your letter that says "I can receive parcels up to 1 kg, and I understand the new forms." The analogy stops being true because the note is only a limit on what the receiver accepts. It does not promise that the road between you can carry that size.

The OPT record tells the other side three main things:

1. **UDP payload size:** the largest reply this side accepts, such as 1232 bytes.
2. **DO bit (DNSSEC OK):** "please include DNSSEC signatures".
3. **Options:** extras such as DNS Cookies, Client Subnet and Extended DNS Errors.

The field layout is in [Going Deeper](#the-opt-record-layout).

**Why 1232 bytes?** A big UDP reply may not fit in one packet, so IP must cut it into **fragments** (pieces). Fragments are often dropped by firewalls, and they can help attackers. So DNS should fit in one packet.

```text
 Lowest MTU that IPv6 allows       1280 bytes
 minus IPv6 header                  -40 bytes
 minus UDP header                    -8 bytes
 -------------------------------------------
 Safe DNS payload                  1232 bytes
```

MTU (Maximum Transmission Unit) is the largest packet a link can carry. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

Older software often advertised **4096 bytes**, which caused fragments. **DNS Flag Day 2020** was a shared effort by DNS vendors and operators to move the default to **1232**. The real standard does not fix one number. The value is a recommendation, and it varies by software and version.

An older event, **DNS Flag Day 2019**, ended the workarounds that big resolvers kept for broken servers that did not reply to EDNS queries.

### 3. Primary and Secondary Servers

A zone should have at least two authoritative servers, in different networks. If one fails, the other still answers.

- **Primary server:** holds the original copy of the zone. Admins edit data here. Old name: master.
- **Secondary server:** holds a read-only copy that it pulls from the primary. Old name: slave.

Both give equally valid answers. Clients and resolvers do not know or care which is which. Both are listed in the zone's NS records.

Analogy: head office writes the price list. Branch offices keep exact copies. A customer can ask any branch. The analogy stops being true because a DNS "branch" checks for updates on a timer and can also be told about them at once.

The **SOA record (Start of Authority)** controls how copying works.

```text
example.com. 3600 IN SOA ns1.example.com. hostmaster.example.com. (
        2026103001 ; serial
        7200       ; refresh: 2 hours
        900        ; retry: 15 minutes
        1209600    ; expire: 14 days
        300 )      ; minimum: negative cache time, 5 minutes
```

| Field | Meaning |
|---|---|
| Serial | Version number of the zone. It must go up after every change. Many people use YYYYMMDDnn |
| Refresh | How often a secondary asks "has the serial changed?" |
| Retry | How soon to try again if that check fails |
| Expire | If the secondary cannot reach the primary for this long, it stops answering for the zone |
| Minimum | Today this is used for negative caching, see section 5 |

A **hidden primary** is a common design, and it differs from the simple setup used in the later examples. The primary is not listed in NS records and is not open to the public. Only the secondaries face the internet. Attackers cannot easily find it, and the place where data is edited stays safe.

```text
      Admin edits zone
            |
            v
   +------------------+
   | Hidden primary   |   Not in NS records. Firewall allows
   | 10.0.0.53        |   only the secondaries.
   +--------+---------+
            | zone transfers
     +------+-----------+
     |                  |
     v                  v
 +--------------+   +---------------+
 | ns1          |   | ns2           |   Public secondaries,
 | 203.0.113.53 |   | 198.51.100.54 |   listed in NS records.
 +--------------+   +---------------+
```

### 4. Zone Transfers: AXFR and IXFR

A **zone transfer** copies zone data from one server to another.

| | AXFR | IXFR |
|---|---|---|
| Full name | Authoritative (full) transfer | Incremental transfer |
| Sends | The entire zone | Only the changes since a given serial |
| Standard | RFC 5936 | RFC 1995 |
| Transport | TCP only | Usually TCP |
| Good for | First copy, or when history is missing | Small changes in big zones |

A zone transfer goes through these steps:

1. An admin changes a record on the primary and raises the serial to 2026103001.
2. The primary sends a **NOTIFY** message to its secondaries (RFC 1996). It means "the zone changed". Without NOTIFY, secondaries wait for the refresh timer.
3. The secondary asks for the SOA record and compares serials.
4. If the primary's serial is higher, the secondary asks for an IXFR and gives its current serial.
5. If the primary still has the change history, it sends only the changes. If not, it sends the whole zone, in AXFR style.
6. The secondary applies the data and starts serving the new version.

```text
 Primary 192.0.2.53                    Secondary 198.51.100.53
    |  NOTIFY example.com (UDP 53)                |
    |-------------------------------------------->|
    |                  SOA? example.com (UDP 53)  |
    |<--------------------------------------------|
    |  SOA serial 2026103001                      |
    |-------------------------------------------->|
    |     IXFR? serial 2026103000 (TCP 53)        |
    |<--------------------------------------------|
    |  Changes only                               |
    |-------------------------------------------->|
```

**Zone transfers must be protected.** An open AXFR gives an attacker every hostname in your zone, which is a ready-made target list. There are three layers of protection:

1. **IP allow list:** only the secondaries may ask.
2. **TSIG (Transaction Signature, RFC 8945):** a shared secret key signs each message with an HMAC. HMAC is a keyed hash that proves who sent the data and that it did not change. It does not hide the data. TSIG includes a timestamp, so server clocks must agree. RFC 8945 recommends a time allowance of 300 seconds.
3. **XoT (zone transfer over TLS, RFC 9103):** encrypts the transfer as well.

Windows note: Active Directory integrated zones copy through AD replication, not AXFR. Classic Windows secondary zones use normal zone transfers.

### 5. Negative Caching

**Negative caching (RFC 2308)** means a resolver also remembers bad news, such as "this name does not exist". Without it, a broken application asking for a missing name every second would load the servers each time.

Two kinds of negative answer exist:

| Type | Status | Meaning | Example |
|---|---|---|---|
| NXDOMAIN | RCODE 3 | The name does not exist for any record type | `nosuchname.example.com` |
| NODATA | NOERROR, 0 answers | The name exists, but not with this record type | AAAA for a name that has only A |

NODATA is not a real status code. It is the name for "NOERROR with an empty answer".

The server puts the SOA record in the Authority section of the reply. The resolver finds the negative TTL like this:

```text
 Negative TTL = the smaller of (TTL of the SOA record, SOA minimum)
 Example      = min(3600, 300) = 300 seconds
```

Analogy: you call a shop and they say "we have no such product". You write that down and do not call again for 5 minutes. The analogy stops being true because a resolver cannot tell that the product arrived in the meantime. It will repeat "no such product" until the time ends.

That is the classic trap. If you query `new.example.com` before you create it, resolvers cache the NXDOMAIN. Then you add the record, and users still fail until the negative TTL ends.

Resolvers also put their own limits on this. For example, BIND limits negative caching with `max-ncache-ttl` (default 3 hours), and Unbound with `cache-max-negative-ttl` (default 1 hour). Check your version, because defaults can change.

Failures can be cached too. RFC 9520 requires resolvers to cache failed lookups for a short time, so that a broken zone does not get a storm of retries.

### 6. Cache Poisoning and the Kaminsky Attack

**Cache poisoning** means an attacker puts a false record into a resolver's cache. Every client that uses the resolver then gets the false answer until the TTL ends.

Plain DNS over UDP has no connection and no signature. A resolver accepts a reply if it matches these things:

1. It comes from the IP address it asked.
2. It arrives at the source port it used.
3. It has the same 16-bit Transaction ID (a random number in the query).
4. It has the same question.

The first reply that matches wins. An attacker who is not on the path cannot see the query, but UDP source addresses are easy to forge. So the attacker guesses.

Analogy: you phone a bank and ask for a call back with a ticket number. A criminal calls you first and guesses the ticket number. You believe them, and you ignore the real bank when it calls. The analogy stops being true because the attacker can try hundreds of guesses in the moment before the real reply arrives.

**The old attack had a limit.** If `www.bank.example` was already cached, the resolver sent no query, so nothing could be forged. If the guess failed, the attacker had to wait for the TTL to end before trying again.

**The Kaminsky attack (2008, CVE-2008-1447)** removed that limit. Dan Kaminsky saw that the attacker can ask for a new random name each time, so the name is never in cache.

1. The attacker makes the resolver look up `a1.bank.example`, which does not exist. It can do this directly or by making a victim load a web image from that name.
2. The resolver sends a query to the `bank.example` name servers.
3. The attacker floods the resolver with forged replies, each with a different Transaction ID.
4. Each forged reply says: "I do not know `a1`, but the name server for `bank.example` is `ns.bank.example` at 203.0.113.66."
5. If one reply wins, the resolver stores the false name server. Now the attacker controls the whole `bank.example` domain in that cache, not just one name.
6. If it fails, the attacker tries `a2`, then `a3`. No waiting is needed.

```text
 Attacker               Resolver              Real name server
    | Q: a1.bank.example    |                        |
    |---------------------->| Q: a1.bank.example     |
    |                       |----------------------->|
    | Fake reply, ID 1000   |                        |
    |---------------------->|                        |
    | Fake reply, ID 1001   |                        |
    |---------------------->|                        |
    | ... hundreds more ... |                        |
    |                       |     Real reply         |
    |                       |<-----------------------|
    |                       |                        |
    | Did a fake win? If not, retry with a2, a3 ...  |
```

The extra records pass the **bailiwick check**. That check means a server can only give extra records for names inside its own zone. Here the forged data is inside `bank.example`, so it looks fine.

The numbers show why this was serious. Suppose the source port is fixed, so only the ID is unknown, and the attacker sends 100 forged replies per try.

```text
 Chance per try:          100 / 65,536 = about 0.15%
 Tries for a 50% chance:  about 450
```

Each try takes well under a second, so a resolver could fall quickly. The fix was **source port randomisation**, released by many vendors together in July 2008. With tens of thousands of possible ports, the attacker must guess from billions of combinations.

```text
 ID only:          65,536 options
 ID and 30,000 ports: 65,536 x 30,000 = about 2 billion options
```

Port randomisation only makes guessing harder. It does not prove anything. The full list of defences:

| Defence | What it does |
|---|---|
| Source port randomisation (RFC 5452) | Adds many bits of randomness |
| 0x20 encoding | Randomly mixes upper and lower case in the query name, and the reply must copy it |
| Bailiwick checks | Rejects extra records for unrelated domains |
| DNS Cookies (RFC 7873) | A small token proves the reply came from the server that was asked |
| Do not run open resolvers | Strangers cannot trigger queries directly |
| DNSSEC | The real fix: a forged answer has no valid signature |
| TCP or encrypted transport | Forged packets cannot join the connection without the sequence numbers |

### 7. DNSSEC: Proving Answers Are Real

**DNSSEC (DNS Security Extensions, RFC 4033, 4034, 4035)** adds digital signatures to DNS data. The zone owner signs records with a private key. Anyone can check them with the public key. For the basics of signatures, see [Cryptography Basics](../09-security/02-cryptography-basics.md).

DNSSEC gives two things:

- **Authenticity:** the data truly came from the zone owner.
- **Integrity:** nobody changed it on the way.

It does **not** give privacy. All queries and answers are still readable.

Analogy: each document carries a notary stamp. The notary's own licence is signed by a higher office, and that one by a national office you already trust. The analogy stops being true because the check is done by software in milliseconds, and the stamp proves only who wrote the data, not that the data is honest or safe.

#### The Records

| Record | Purpose |
|---|---|
| **DNSKEY** | The public keys of a zone |
| **RRSIG** | A signature over a set of records of the same name and type |
| **DS** | Delegation Signer: a hash (a short fingerprint) of the child's key, stored in the parent zone |
| **NSEC** | Signed proof that a name does not exist, by naming the next name in order |
| **NSEC3** | Same idea, but it uses hashed names (RFC 5155) |
| **CDS / CDNSKEY** | The child tells the parent what DS it wants (RFC 7344) |

A group of records with the same name and type is called an **RRset**. DNSSEC signs whole RRsets, not single records.

Most zones use two kinds of key:

- **ZSK (Zone Signing Key):** signs the normal records in the zone. DNSKEY flags value 256.
- **KSK (Key Signing Key):** signs only the DNSKEY RRset. Its hash goes to the parent as DS. DNSKEY flags value 257.

Why two keys? Changing a ZSK needs no help from anyone. Changing a KSK means updating the DS in the parent zone, usually through a registrar (the company where you bought the domain). So the KSK can stay the same for longer while the ZSK changes more often. The split is a common practice, not a rule. A single key that does both jobs (a CSK, Combined Signing Key) is also valid.

#### The Chain of Trust

The chain starts from one thing the resolver already trusts: the **root trust anchor**. This is the public root KSK built into the resolver's configuration.

```text
 Resolver has: root KSK (trust anchor)
      |
      | matches
      v
 +-----------------------------------------------+
 | ROOT ZONE "."                                 |
 |   DNSKEY (KSK, ZSK)   signed by root KSK      |
 |   DS for com          signed by root ZSK      |
 +----------------------+------------------------+
                        | DS = hash of com KSK
                        v
 +-----------------------------------------------+
 | COM ZONE                                      |
 |   DNSKEY (KSK, ZSK)   signed by com KSK       |
 |   DS for example.com  signed by com ZSK       |
 +----------------------+------------------------+
                        | DS = hash of example.com KSK
                        v
 +-----------------------------------------------+
 | EXAMPLE.COM ZONE                              |
 |   DNSKEY (KSK, ZSK)   signed by its KSK       |
 |   A 203.0.113.10      signed by its ZSK       |
 +-----------------------------------------------+
```

A validating resolver checks the chain like this:

1. Fetch the root DNSKEY RRset. Check that its signature comes from the trusted KSK.
2. Fetch the DS for `com` from the root. Check its signature with the root ZSK.
3. Fetch the `com` DNSKEY RRset. Hash the KSK and compare the hash with the DS.
4. Check the DNSKEY RRset's signature with that KSK.
5. Do the same for the DS of `example.com`, signed by the `com` ZSK.
6. Fetch the `example.com` keys and match the KSK to that DS.
7. Check the signature on the A record with the `example.com` ZSK.
8. All checks pass, so the answer is **secure**, and the resolver sets the **AD** flag.

The DS record is the link between zones. It is why a domain owner must give the DS to the registrar. If the DS and key do not match, the chain breaks.

#### Validation Results

| State | Meaning | What the client sees |
|---|---|---|
| Secure | The chain is complete and all signatures are good | Answer with AD=1 |
| Insecure | A signed parent proves that this zone has no DS | Normal answer, AD=0 |
| Bogus | A signature or key failed, or data is missing | **SERVFAIL** (the "server failure" status code) |
| Indeterminate | The resolver cannot tell which of the above it is | Depends on the resolver |

This is the operational risk of DNSSEC. An expired signature or a wrong DS makes your whole domain fail for users of validating resolvers, even though your servers are healthy. Slack had a public outage in September 2021 during a DNSSEC rollout.

Key facts to remember:

- The root zone was signed in **2010**.
- The first root KSK rollover took place in **October 2018**, to the key with key tag 20326.
- A second key (KSK-2024, key tag 38696) was published in the root zone in 2025, with the move to it planned for 2026. Keep trust anchors up to date with RFC 5011 automation or software updates.
- Most domains in big zones such as `.com` are still not signed.
- Signed replies are large. An RSA-2048 signature is 256 bytes. An ECDSA P-256 signature is 64 bytes. That is why algorithm 13 (ECDSA P-256 with SHA-256) is popular for new zones.

### 8. Encrypted DNS: DoT, DoH and DoQ

Plain DNS is readable by anyone on the path: your Wi-Fi owner, your ISP, or an attacker. It can also be changed on the way. Encrypted DNS protects the link between the client and the resolver.

Analogy: plain DNS is a postcard, and encrypted DNS is a sealed envelope. The analogy stops being true because the sealed envelope still goes to one company that can read it. You move your trust to the resolver. You do not remove the need for trust.

| Protocol | Standard | Transport | Port | Main point |
|---|---|---|---|---|
| Plain DNS | RFC 1035 | UDP or TCP | 53 | No privacy |
| **DoT** (DNS over TLS) | RFC 7858 | TLS over TCP | 853 | Clear to see and easy for admins to allow or block |
| **DoH** (DNS over HTTPS) | RFC 8484 | HTTP over TLS | 443 | Looks like normal web traffic |
| **DoQ** (DNS over QUIC) | RFC 9250 | QUIC over UDP | 853 | No TCP head-of-line blocking, quick setup |

For the encryption details, see [TLS and HTTPS](07-tls-and-https.md) and [QUIC](../06-transport-layer/08-quic.md).

**DoT** opens a TCP connection, does a TLS handshake, then sends normal DNS messages with the 2-byte length prefix. The client keeps the connection open so that later queries are fast. There are two profiles (RFC 8310):

- **Strict:** the client must verify the server's identity. If it fails, the query fails.
- **Opportunistic:** the client tries encryption, but falls back if needed. It protects against watchers, but not against an active attacker.

**DoH** sends the same binary DNS message inside an HTTPS request, with the media type `application/dns-message`. It can use GET or POST. RFC 8484 recommends HTTP/2 as the minimum version.

People debate which is better:

| Point | DoT | DoH |
|---|---|---|
| Network admin can see the traffic is DNS | Yes, port 853 | No, it mixes with web traffic |
| Easy to block | Yes | Harder |
| Works where only web traffic is allowed | Often no | Yes |
| Built into | Operating systems, such as Android Private DNS | Browsers, and Windows 11 |

Limits of encrypted DNS:

- It protects only the client-to-resolver link. Resolver-to-authoritative traffic is mostly still plain.
- The resolver still sees all your queries.
- Watchers still see the IP address you connect to. They may also see the server name in the TLS handshake unless Encrypted Client Hello (ECH) is in use.
- It does not prove the data is genuine. DNSSEC does that. The two work together.

| | Plain DNS | DNSSEC | DoT / DoH / DoQ |
|---|---|---|---|
| Hides queries from the path | No | No | Yes |
| Proves data came from the zone owner | No | Yes | No |
| Stops changes on the client link | No | Only if the client validates | Yes |

**Enterprise problem:** if a browser sends DoH directly to a public resolver, it skips the company DNS server. Internal names stop working, and security filters are bypassed. Firefox checks a special **canary domain**, `use-application-dns.net`. If the local resolver says it does not exist, Firefox turns off its automatic DoH. Admins also use browser policies, and they block known DoH servers by IP or name.

### 9. Split-Horizon DNS

**Split-horizon DNS** (also called split-brain DNS) gives different answers for the same name, based on who asks. Staff inside the company get an internal address. The public gets an external one.

```text
                 app.example.com ?
                        |
          +-------------+--------------+
          |                            |
  Client 10.0.8.15              Client 198.51.100.77
  (inside network)              (on the internet)
          |                            |
          v                            v
  Internal view                  External view
  app = 10.0.5.20                app = 203.0.113.20
```

Analogy: a receptionist has two lists. Staff who ask get the direct office line. Visitors get the public front desk number. The analogy stops being true because DNS decides by source IP, not by knowing the person, so a staff laptop at home counts as a visitor unless it is on a VPN.

Why people use it:

- Hides internal hostnames and addresses from the public.
- Keeps internal traffic on a short internal path, without going out and back in (hairpinning).
- Lets the internal zone list things that the public should not see.

Ways to build it:

- BIND **views** (see the [configuration below](#example-3-split-horizon-with-bind-views)).
- Windows Server 2016 and later: DNS policies with zone scopes.
- AWS Route 53: a private hosted zone and a public hosted zone with the same name.
- Two separate DNS servers, one inside and one outside.

Problems to expect:

- Laptops move. A cached internal answer fails when the laptop goes home, until the TTL ends.
- VPN clients may use the wrong resolver.
- A client that uses public DoH will get the external answer.
- If the zone is signed, the internal view needs its own handling, because changed data will not match public signatures.
- Two copies of the zone can drift apart. Automate them.

### 10. GeoDNS

**GeoDNS** is an authoritative server that gives different answers based on the location of the asker. Users in Europe get a European server's address. Users in Asia get an Asian one. It is a common way to do global load balancing, and it is covered more in [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md) and [Content Delivery Networks](../10-enterprise-and-datacenter/06-cdn.md).

1. A user in Paris asks the resolver for `www.example.com`.
2. The resolver asks the authoritative server.
3. The authoritative server looks up the **resolver's** source IP in a location database.
4. It answers with 203.0.113.20, the Frankfurt site, and a short TTL such as 60 seconds.

Analogy: a national helpline asks your postcode, then gives you the number of your nearest branch. The analogy stops being true because the helpline sees where your **resolver** is, not you. A user in Karachi who uses a resolver in Singapore can look like a user in Singapore.

**EDNS Client Subnet (ECS, RFC 7871)** fixes this. The resolver adds part of the client's address to the query, such as `192.0.2.0/24` rather than the full address. The authoritative server can then choose by the client's real network. RFC 7871 suggests sending no more than 24 bits for IPv4 and 56 bits for IPv6. It is an Informational RFC, and resolvers differ. Google Public DNS sends ECS. Cloudflare 1.1.1.1 does not, for privacy reasons.

Limits of GeoDNS:

- Location databases are sometimes wrong, and they say nothing about real network speed.
- TTLs mean a failed site keeps getting traffic until caches expire.
- It is not real load balancing. One resolver can stand for thousands of users.
- Health checks must remove dead sites from the answers.

Managed services add more policies, such as AWS Route 53's latency, geolocation, weighted and failover routing.

### 11. Anycast DNS

**Anycast** means the same IP address is announced from many places. Routers send each packet to the nearest site, as measured by routing. On the internet this is done with BGP, the routing protocol between networks. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md) and [BGP](../05-routing/09-bgp.md).

```text
    Same address 192.0.2.53 announced from all three sites by BGP

   +-------------+      +-------------+      +-------------+
   | Site London |      | Site Dubai  |      | Site Tokyo  |
   | 192.0.2.53  |      | 192.0.2.53  |      | 192.0.2.53  |
   +------+------+      +------+------+      +------+------+
          |                    |                    |
   ===================== The Internet (BGP) ====================
          ^                    ^                    ^
          |                    |                    |
     User in Paris        User in Lahore       User in Seoul
```

Analogy: an emergency number such as 112 is one number everywhere, and you reach your local centre. The analogy stops being true because the choice is made by network routing, so "nearest" can mean fewest network hops or best policy, not the shortest distance on a map.

Benefits:

- **Speed:** users reach a nearby site.
- **Failover:** if a site fails, it stops announcing the route, and traffic moves to the next site. Clients need no change.
- **DDoS absorption:** a DDoS (Distributed Denial of Service) attack floods a target from many machines. With anycast, the attack is split among many sites, so one site's failure does not take down the service.

The 13 root servers are the classic case. There are 13 names (`a.root-servers.net` to `m.root-servers.net`), which comes from fitting them in the old 512-byte reply. But each name is anycast, and in total there are well over a thousand physical instances worldwide. "There are only 13 root servers" is a common mistake. Public resolvers such as 8.8.8.8 and 1.1.1.1 use anycast too.

UDP suits anycast well, because a query is usually one packet each way. TCP is riskier: if routes change during a connection, packets may reach a new site that knows nothing about the connection. In practice, DNS TCP sessions are short and routes are mostly stable, so it works well.

Anycast has a trap. In October 2021, Facebook's DNS servers lost contact with its data centres after a backbone failure. They were built to stop announcing their anycast routes when that happened. So the DNS servers vanished from the internet, and facebook.com could not be resolved. Failing safe per site can become total failure if all sites fail together.

**GeoDNS vs anycast:**

| | GeoDNS | Anycast |
|---|---|---|
| Layer | DNS answer | IP routing |
| Decision made | At lookup time | For every packet |
| Addresses | Different IPs for different users | One IP everywhere |
| Failover speed | Limited by TTL | Limited by BGP convergence |
| Often combined? | Yes: anycast name servers answer with geo-based addresses | |

### 12. DNS Tunnelling

**DNS tunnelling** hides other data inside DNS messages. It works because almost every network allows DNS out, even when other traffic is blocked. Resolvers forward queries to the internet for you.

Analogy: a prisoner cannot send letters, but may post a "change of address" card. So the prisoner writes the secret message in the address line. The analogy stops being true because a tunnel works both ways and can run a whole IP connection, slowly.

1. The attacker owns `t.example.com` and runs a special authoritative server for it.
2. Software on the inside machine encodes data in Base32, which uses only letters and digits that fit in a hostname.
3. It asks for a name such as:

```text
ONSWG4TFOQQHEZLQN5ZHIIDQMFZHIIBR.t.example.com
```

That label is the text "secret report part 1".

4. The local resolver cannot answer, so it forwards the query. The data reaches the attacker's server, even if the machine cannot reach the internet directly.
5. The server replies with data in a TXT, NULL or CNAME record.

```text
 Inside PC       Company resolver       Internet     Attacker's server
    |  data in the name  |                 |                |
    |------------------->|-----------------|--------------->|
    |                    |                 |                |
    |  data in TXT reply |                 |                |
    |<-------------------|<----------------|----------------|
```

A DNS name is at most 253 characters, and each label is at most 63. So one query carries only about 140 bytes of data. That is slow, but enough to steal passwords or send malware commands. Known tools are **iodine** and **dnscat2**.

**Signs of tunnelling:**

- Very long labels with random-looking text (high entropy).
- A huge number of unique names under one domain.
- Many TXT, NULL or other unusual query types.
- High volume from one host to one domain, day and night.
- Many NXDOMAIN replies, often seen with malware that uses DGA (Domain Generation Algorithms, code that creates many random names).

**Defences:**

- Force all clients to use your resolvers. Block outbound 53 and 853 from everything else.
- Log and study DNS queries.
- Use a **Response Policy Zone (RPZ)** or a DNS filtering service to block bad domains.
- Control DoH, with policy and by blocking known DoH servers.

Tunnelling is not only used by attackers. Some people use it to get past paid Wi-Fi login pages, and some VPN tools offer it as a last resort.

### 13. TTL Trade-Offs During Migration

The TTL is a trade-off between **control** and **load**.

| TTL | Queries per day from one resolver | Time to undo a change | Good for |
|---|---|---|---|
| 60 s | 1,440 | About 1 minute | Failover records, migrations |
| 300 s | 288 | About 5 minutes | Busy web front ends |
| 3,600 s | 24 | About 1 hour | Normal records |
| 86,400 s | 1 | About 1 day | Records that rarely change, such as MX |

| Short TTL | Long TTL |
|---|---|
| Changes take effect fast | Changes take effect slowly |
| More queries, so more load and cost | Fewer queries |
| Users wait for lookups more often | Most lookups are served from cache |
| A DNS outage hurts sooner | Caches keep working during a DNS outage |

There is no such thing as "DNS propagation" as a push. Nothing is sent out to the world. Old answers simply **expire** from each cache at its own time.

Analogy: you print leaflets with your phone number and a "valid until" date. If you move office on Monday, old leaflets that say "valid for 30 days" are still in people's hands. So print short-date leaflets a month early. The analogy stops being true because a few resolvers do not follow the TTL exactly.

The golden rule: **lower the TTL first, and wait at least the old TTL, before you make the change.** Full steps are in [Example 5](#example-5-moving-a-website-with-no-downtime).

Parts that catch people out:

- **Lowering the TTL has no effect on copies already cached.** They keep the old TTL.
- **Some software caches for longer.** Browsers, Java applications, and long-lived connection pools can stay on the old IP.
- **A connection that is already open does not use DNS again.** It stays on the old server. Keep the old server up to drain.
- **Check every record in a CNAME chain.** Each one has its own TTL.
- **Delegation TTLs are not yours.** The NS records in `.com` have a TTL of 172,800 seconds (2 days), which the registry sets.
- **Secondaries must be updated before the change counts.** Check every name server.

## Example

### Example 1: Watching Truncation and the Move to TCP

Zone `example.com` has a large TXT set. The server is 192.0.2.53, and the client is 10.0.0.25 on Linux.

Ask with a small UDP size, and tell `dig` not to retry:

```bash
# Linux, macOS (BIND dig)
dig @192.0.2.53 example.com TXT +bufsize=512 +ignore +norecurse
```

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41872
;; flags: qr aa tc; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.                   IN      TXT

;; Query time: 11 msec
;; SERVER: 192.0.2.53#53(192.0.2.53) (UDP)
;; WHEN: Fri Oct 30 10:15:22 UTC 2026
;; MSG SIZE  rcvd: 40
```

Read it like this:

- `tc` is set. The answer did not fit, and `ANSWER: 0` shows nothing useful came back.
- `udp: 1232` is the size the server itself can accept.
- The reply is 40 bytes: 12 for the header, 17 for the question, 11 for the OPT record.

Now let `dig` behave normally:

```bash
dig @192.0.2.53 example.com TXT +bufsize=512 +norecurse
```

```text
;; Truncated, retrying in TCP mode.
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8120
;; flags: qr aa; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1
...
;; SERVER: 192.0.2.53#53(192.0.2.53) (TCP)
;; MSG SIZE  rcvd: 1874
```

The same test on Windows:

```powershell
# Windows PowerShell: force TCP
Resolve-DnsName example.com -Type TXT -Server 192.0.2.53 -TcpOnly

# Windows nslookup: -vc means "use a virtual circuit", which is TCP
nslookup -vc -type=TXT example.com 192.0.2.53
```

If UDP works and TCP does not, check the firewall. On Cisco IOS, allow both:

```text
! Cisco IOS: inbound ACL for a public name server
ip access-list extended OUTSIDE-IN
 permit udp any host 192.0.2.53 eq domain
 permit tcp any host 192.0.2.53 eq domain
```

Also check DNS inspection on firewalls. On Cisco ASA, the default DNS inspection map has long used a 512-byte `message-length` limit. Setting `message-length maximum client auto` makes it honour the size that the client advertised in EDNS.

### Example 2: Following a DNSSEC Chain

The key tags and signatures here are example values, shortened to fit. Text after the last `;` on a data line is a note added for the reader.

Ask for the record and its signature:

```bash
dig example.com A +dnssec +noall +comments +answer
```

```text
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
example.com. 3600 IN A 203.0.113.10
example.com. 3600 IN RRSIG A 13 2 3600 20261113000000 (
                  20261030000000 34505 example.com. k3Yq...== )
```

The RRSIG fields mean:

| Value | Field | Meaning |
|---|---|---|
| A | Type covered | This signs the A records |
| 13 | Algorithm | ECDSA P-256 with SHA-256 |
| 2 | Labels | The name has 2 labels: `example`, `com` |
| 3600 | Original TTL | The TTL that was signed |
| 20261113000000 | Expiration | Not valid after 13 Nov 2026 00:00 UTC |
| 20261030000000 | Inception | Not valid before 30 Oct 2026 00:00 UTC |
| 34505 | Key tag | Short ID of the key that signed it |
| example.com. | Signer | The zone that signed it |

The `ad` flag shows that the resolver validated the answer.

Now get the keys:

```bash
dig example.com DNSKEY +noall +answer
```

```text
example.com. 3600 IN DNSKEY 256 3 13 mdsswU...== ; ZSK, tag 34505
example.com. 3600 IN DNSKEY 257 3 13 kXKkvW...== ; KSK, tag 31589
```

Key tag 34505 is the ZSK, so it signed the A record. Flag 257 marks the KSK.

Now move up one level. The DS lives in the parent, `com`:

```bash
dig example.com DS +noall +answer
```

```text
example.com. 86400 IN DS 31589 13 2 8f3ac2...b71e
```

The DS has key tag **31589**, the same as the KSK. Algorithm is 13, and digest type 2 is SHA-256. The resolver hashes the KSK and compares. The same step repeats from `com` up to the root.

For a full automatic check:

```bash
# Linux: delv comes with BIND tools and validates by itself
delv @1.1.1.1 example.com A
```

```text
; fully validated
example.com.   3600 IN A 203.0.113.10
example.com.   3600 IN RRSIG A 13 2 3600 20261113000000 ...
```

When a zone fails, use `+cd` to compare:

```bash
dig @1.1.1.1 dnssec-failed.org A        # status: SERVFAIL
dig @1.1.1.1 dnssec-failed.org A +cd    # status: NOERROR
```

`dnssec-failed.org` is a real test domain that is broken on purpose. If an answer fails normally but works with `+cd`, the problem is DNSSEC, not the server.

On Windows:

```powershell
Resolve-DnsName example.com -Type A -DnssecOk -Server 1.1.1.1
Resolve-DnsName example.com -Type DS -Server 1.1.1.1
```

### Example 3: Split-Horizon With BIND Views

Internal clients use 10.0.0.0/8 and get the internal answer.

```text
// named.conf (BIND 9)
acl "inside" { 10.0.0.0/8; };

view "internal" {
    match-clients { inside; };
    recursion yes;
    zone "example.com" {
        type primary;
        file "zones/example.com.internal";  // app = 10.0.5.20
    };
};

view "external" {
    match-clients { any; };
    recursion no;
    zone "example.com" {
        type primary;
        file "zones/example.com.external";  // app = 203.0.113.20
    };
};
```

BIND uses the first view that matches, so order matters. When any view exists, all zones must sit inside views.

```bash
# From inside, client 10.0.8.15
dig @10.0.0.53 app.example.com +short
10.0.5.20

# From the internet, client 198.51.100.77
dig @192.0.2.53 app.example.com +short
203.0.113.20
```

### Example 4: Securing a Zone Transfer and Reading a Negative Answer

The primary is 192.0.2.53. The secondary is 198.51.100.53. First, create a shared key:

```bash
# Linux, BIND: creates a key block to copy to both servers
tsig-keygen -a hmac-sha256 xfer-key > /etc/bind/xfer.key
```

On the primary:

```text
// named.conf on 192.0.2.53
include "/etc/bind/xfer.key";

zone "example.com" {
    type primary;
    file "zones/example.com";
    allow-transfer { key xfer-key; };
    also-notify { 198.51.100.53; };
};
```

On the secondary:

```text
// named.conf on 198.51.100.53
include "/etc/bind/xfer.key";

zone "example.com" {
    type secondary;
    file "secondary/example.com";
    primaries { 192.0.2.53 key xfer-key; };
};
```

Older BIND versions use `master`, `slave` and `masters`. The newer words work in current releases.

Test it. A stranger with no key should fail:

```bash
dig @192.0.2.53 example.com AXFR
```

```text
; Transfer failed.
```

With the key, it works, and it shows the whole zone over TCP:

```bash
dig -k /etc/bind/xfer.key @192.0.2.53 example.com AXFR
```

```text
example.com.     3600  IN SOA ns1.example.com. hostmaster.example.com.
                         2026103001 7200 900 1209600 300
example.com.     3600  IN NS  ns1.example.com.
example.com.     3600  IN NS  ns2.example.com.
ns1.example.com. 3600  IN A   192.0.2.53
ns2.example.com. 3600  IN A   198.51.100.53
www.example.com. 86400 IN A   203.0.113.10
example.com.     3600  IN SOA ns1.example.com. hostmaster.example.com.
                         2026103001 7200 900 1209600 300
;; SERVER: 192.0.2.53#53(192.0.2.53) (TCP)
;; XFR size: 7 records (messages 1, bytes 341)
```

The transfer starts and ends with the SOA. That is how the receiver knows it is complete.

To ask only for changes since an older serial:

```bash
dig -k /etc/bind/xfer.key @192.0.2.53 example.com IXFR=2026103000
```

On Windows Server, the same kind of control looks like this:

```powershell
# Allow transfers only to the named secondary
Set-DnsServerPrimaryZone -Name example.com `
  -SecureSecondaries TransferToSecureServers `
  -SecondaryServers 198.51.100.53

# On a Windows secondary: start a transfer now
Start-DnsServerZoneTransfer -ZoneName example.com
```

Now read a negative answer from the same zone:

```bash
dig @192.0.2.53 new.example.com A +norecurse
```

```text
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 5120
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; AUTHORITY SECTION:
example.com. 3600 IN SOA ns1.example.com. hostmaster.example.com.
                         2026103001 7200 900 1209600 300
```

The SOA TTL is 3600. The SOA minimum is 300. The negative TTL is the smaller value, so a resolver keeps this NXDOMAIN for **300 seconds**. If you add `new.example.com` right after this query, users of that resolver can wait up to 5 minutes.

### Example 5: Moving a Website With No Downtime

`www.example.com` has TTL 86,400 and points to 203.0.113.10. It must move to 203.0.113.20 on Friday at 10:00 UTC.

```text
 Wed 09:00   Set TTL to 300. Address does not change.
     |
     |       Wait at least 24 hours (the OLD TTL).
     |       Old caches holding TTL 86400 run out.
     v
 Fri 10:00   Change A record to 203.0.113.20. TTL stays 300.
     |       Both servers keep running.
     |       In about 5 minutes, new lookups use the new server.
     v
 Fri 10:30   Check results. Keep a way back. If it is bad,
     |       change back, and that also takes about 5 minutes.
     v
 Mon         Old server logs show no real traffic. Shut it down.
     |
     v
 Mon         Raise TTL to 3600 or 86400.
```

Step by step:

1. Lower the TTL on Wednesday and raise the serial.
2. Check that **every** name server has the new TTL.

```bash
# Linux: both must show 300
dig @192.0.2.53 www.example.com A +norecurse +noall +answer
dig @198.51.100.53 www.example.com A +norecurse +noall +answer
```

```text
www.example.com. 300 IN A 203.0.113.10
www.example.com. 300 IN A 203.0.113.10
```

3. Copy the data and test the new server before the cutover, by connecting to its IP directly:

```bash
# Linux: test the new server with the real name, without changing DNS
curl --resolve www.example.com:443:203.0.113.20 https://www.example.com/
```

```powershell
# Windows: curl.exe has the same option
curl.exe --resolve www.example.com:443:203.0.113.20 https://www.example.com/
```

4. On Friday, change the record. Check authoritative servers first, then public resolvers:

```bash
dig @192.0.2.53 www.example.com +short
dig @8.8.8.8 www.example.com +short
dig @1.1.1.1 www.example.com +short
```

5. Keep the old server up. Clients with long-lived connections or sticky caches will stay for a while.
6. Clear your own caches when testing.

```powershell
# Windows
ipconfig /flushdns
```

```bash
# Linux with systemd-resolved
resolvectl flush-caches
```

7. When the old server is quiet, turn it off, and raise the TTL. If you leave it at 300 forever, you pay in load and lose some protection from DNS outages.

**Moving the DNS provider** is a different job, because you change the name servers:

1. Copy the whole zone to the new provider. Test it with `dig @new-ns`.
2. Change the NS records at the registrar.
3. Keep **both** providers serving the same data for at least the delegation TTL (2 days in `.com`), and make any change on both.
4. If the zone is signed, plan first. Either remove the DS, wait for it to expire (`.com` DS TTL is 86,400 seconds), and move while unsigned, or use a multi-signer setup (RFC 8901). If you move while the old DS is still in place and the new provider signs with other keys, validating resolvers get SERVFAIL.

## Going Deeper

### The OPT Record Layout

OPT reuses the normal record format, but gives the fields new meanings. Its type is 41.

| Field | Size | Normal meaning | In OPT |
|---|---|---|---|
| NAME | 1 byte | Owner name | 0, meaning the root |
| TYPE | 2 bytes | Record type | 41 |
| CLASS | 2 bytes | Usually IN | Sender's UDP payload size |
| TTL | 4 bytes | Cache time | See below |
| RDLENGTH | 2 bytes | Data length | Length of the options |
| RDATA | Variable | Record data | Options as code, length, value |

The 32-bit TTL field is split like this:

```text
 Bit:  0        7 8       15 16 17                   31
      +----------+----------+--+----------------------+
      | Ext RCODE| Version  |DO|          Z           |
      +----------+----------+--+----------------------+
```

RCODE (response code) is the status of a reply. Extended RCODE adds upper bits, so the 4-bit field can grow. Common options:

| Code | Option | Standard |
|---|---|---|
| 3 | NSID: says which server instance answered | RFC 5001 |
| 8 | Client Subnet | RFC 7871 |
| 10 | Cookie | RFC 7873 |
| 12 | Padding: makes encrypted messages the same size | RFC 7830 |
| 15 | Extended DNS Error | RFC 8914 |

**Extended DNS Errors** explain a SERVFAIL. For example, code 6 is "DNSSEC Bogus" and code 7 is "Signature Expired". Recent `dig` versions print them in the OPT section.

### Proving That Something Does Not Exist

A signature needs data to sign. "Nothing here" has no data. Signing replies on the spot needs the private key on every server, so DNSSEC signs a statement about the gap between names.

```text
 alpha.example.com. NSEC mail.example.com. A RRSIG NSEC
```

This says: "after `alpha`, the next name is `mail`, and `alpha` has only A, RRSIG and NSEC records." If you ask for `beta.example.com`, this record proves that no name exists between them.

The side effect is **zone walking**. An attacker can follow the NSEC chain and list every name in the zone.

**NSEC3** uses hashes of names rather than the names themselves. This makes walking harder, but offline guessing of common names is still possible. RFC 9276 now advises 0 extra iterations and no salt, because extra hashing costs resolvers more than it costs attackers.

Many online signers avoid the problem in another way. They create a narrow NSEC record on the spot for each negative answer, often called **white lies** or compact denial. This needs the private key on the serving systems.

### Key Rollovers

A **key rollover** replaces a DNSSEC key. It must be done carefully, because caches may hold old keys and old signatures.

- **ZSK pre-publish:** add the new key, wait for the DNSKEY TTL, sign with the new key, wait for old signatures to expire from caches, then remove the old key.
- **KSK double-DS:** add the new DS to the parent, wait for the DS TTL, swap keys, then remove the old DS.

Most mistakes come from skipping the waiting steps. Tools such as BIND's `dnssec-policy` automate these steps. **CDS and CDNSKEY** records let the child publish the new DS so that a parent that supports them can pick it up without a manual change.

### AD Flag and Where Validation Happens

Most laptops do not validate. They trust the AD flag from the resolver. If the path between them is plain, an attacker can set AD=1 or change the data. So the AD flag is only as trustworthy as that last link. Either validate locally or use encrypted DNS to a resolver you trust.

**DANE (RFC 6698)** builds on DNSSEC. A TLSA record publishes which TLS certificate a service should use. It is used mostly for email between mail servers, and web browsers do not support it. See [TLS and HTTPS](07-tls-and-https.md).

### Newer Poisoning and DNSSEC Attacks

- **SAD DNS (2020, CVE-2020-25705):** used a side channel in ICMP (the IP error message protocol) rate limiting to find the resolver's source port. That removed most of the port randomness. Fixes randomise the ICMP limit.
- **Fragmentation attacks:** an attacker can replace the second fragment of a large UDP reply. Fragments have no DNS ID or port, which is another reason to keep replies at 1232 bytes or less.
- **KeyTrap (CVE-2023-50387, disclosed 2024):** crafted DNSSEC data made validators do huge amounts of work, so one small packet could stall a resolver. Resolver vendors fixed it by limiting the work per query.

### Other DNS Attacks to Know

Common attacks are covered in [Common Network Attacks](../09-security/03-common-network-attacks.md). The DNS-specific ones are:

- **Reflection and amplification:** the attacker forges the victim's address and sends small queries. Servers send large replies to the victim. A 60-byte query that gets a 3,000-byte reply gives a 50 times gain. Open resolvers make it worse. RFC 8482 lets servers give short answers to ANY queries.
- **Response Rate Limiting (RRL):** an authoritative server limits identical replies per client network. BIND has this in `rate-limit { responses-per-second 5; };`. Some dropped replies are sent as TC=1, so real clients can retry over TCP and attackers get no gain.
- **Random subdomain attack (water torture):** a botnet (a group of infected machines) asks through real resolvers for random names under a victim domain. Nothing is cached, so all of it reaches the victim's name servers. Aggressive use of NSEC (RFC 8198) helps for signed zones.
- **DNS rebinding:** a hostile domain first answers with a public IP, then with a private one such as 192.168.1.1. The browser then treats a home router as the same site. Defences are private-address filters on resolvers and checks of the Host header.
- **Dangling CNAME (subdomain takeover):** `shop.example.com` still points to a cloud service you deleted. Someone else registers that service name and takes your subdomain. Remove old records when a service is removed.
- **Registrar or account takeover:** with stolen login details, an attacker can change your NS records. DNSSEC does not help if they can change the DS too. Use registrar lock and multi-factor login.

### Keeping Resolvers Healthy and Private

- **QNAME minimisation (RFC 9156):** the resolver tells each server only the part it needs. The root sees just `com`, not `www.example.com`.
- **Serve stale (RFC 8767):** if authoritative servers cannot be reached, a resolver may give an expired answer rather than fail.
- **Prefetching:** a resolver refreshes popular records just before they expire, so users do not wait.
- **Aggressive NSEC (RFC 8198):** a validating resolver can create NXDOMAIN from cached NSEC records, without asking again.
- **RPZ (Response Policy Zone):** a zone that holds rules such as "answer NXDOMAIN for this bad domain". It is a de facto standard, supported by BIND and others, not an RFC.
- **Finding encrypted resolvers:** DDR (Discovery of Designated Resolvers, RFC 9462) lets a client learn that its plain resolver also offers an encrypted one. DNR (RFC 9463) lets DHCP or IPv6 RA announce it.
- **Oblivious DoH (RFC 9230):** a relay hides the client IP from the resolver, and the resolver's encryption hides the query from the relay. No single party sees both.

### Encrypted DNS Commands

```bash
# Linux: dig from BIND 9.18 or newer
dig +tls @1.1.1.1 example.com A
dig +https @1.1.1.1 example.com A

# Linux, macOS: DoH with curl. This JSON form is a Cloudflare
# convenience, it is not the RFC 8484 binary format.
curl -s -H 'accept: application/dns-json' \
  'https://cloudflare-dns.com/dns-query?name=example.com&type=A'
```

```text
# Linux: /etc/systemd/resolved.conf, for DoT with strict checks
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 9.9.9.9#dns.quad9.net
DNSOverTLS=yes
```

```powershell
# Windows 11: show resolvers that Windows knows support DoH
Get-DnsClientDohServerAddress
```

To turn on DoH in Windows 11, use Settings, Network and internet, your adapter, then DNS server assignment.

A company can force all DNS through its own resolver at 10.0.0.53, which helps stop plain tunnels and bypass:

```text
! Cisco IOS: LAN outbound ACL
ip access-list extended LAN-OUT
 permit udp host 10.0.0.53 any eq domain
 permit tcp host 10.0.0.53 any eq domain
 deny   udp 10.0.0.0 0.255.255.255 any eq domain
 deny   tcp 10.0.0.0 0.255.255.255 any eq domain
 deny   tcp 10.0.0.0 0.255.255.255 any eq 853
 deny   udp 10.0.0.0 0.255.255.255 any eq 853
 permit ip any any
```

DoH on port 443 cannot be stopped this way. It needs lists of known servers, or a proxy that can inspect TLS.

### Telling Anycast Sites Apart

When an address is anycast, you need to know which site answered. Many servers answer special CHAOS class queries:

```bash
# Linux: which root instance did I reach?
dig @f.root-servers.net hostname.bind CH TXT +short

# Linux: which 1.1.1.1 site did I reach? (returns a site code)
dig @1.1.1.1 id.server CH TXT +short

# Linux: ask for the NSID EDNS option
dig @192.0.2.53 example.com A +nsid
```

Run the same command from two cities, and the names differ. Each operator chooses its own naming format.

### Many Name Servers, Many Providers

Big sites use two DNS providers, so that one provider's outage does not stop them. In October 2016, a large attack from the Mirai botnet hit the DNS provider Dyn, and many well-known sites that relied only on Dyn became unreachable. The cost of two providers is that you must keep two copies in step, and advanced features may differ. If the zone is signed, you need the multi-signer models in RFC 8901.

### Standards Table

| Topic | RFC |
|---|---|
| DNS core | 1034, 1035 |
| DNS over TCP | 7766, 9210 |
| EDNS | 6891 |
| AXFR, IXFR, NOTIFY | 5936, 1995, 1996 |
| TSIG, XoT | 8945, 9103 |
| Negative caching, failure caching | 2308, 9520 |
| DNSSEC core, NSEC3, NSEC3 settings | 4033-4035, 5155, 9276 |
| DoT, DoH, DoQ | 7858, 8484, 9250 |
| Client Subnet | 7871 |
| Forgery resilience, Cookies | 5452, 7873 |

## Common Mistakes

- **"DNS is UDP only" or "TCP 53 is only for zone transfers."** TCP is a required part of DNS. Blocking it breaks large answers, and some of them look like random failures.
- **Allowing only 512-byte replies through a firewall.** EDNS and DNSSEC replies are larger. Old firewall limits drop them with no clear error.
- **Forgetting to raise the SOA serial.** The secondaries see no change and keep the old data. Some tools do this for you, but hand edits need care.
- **Thinking DNSSEC encrypts DNS.** It signs. Anyone can read a signed answer. DoT and DoH encrypt, and they do not prove the origin of the data.
- **Thinking encrypted DNS replaces DNSSEC.** DoH protects one link only. A lying upstream server can still send false data over a perfect TLS link.
- **Letting RRSIG signatures expire.** Signing is not a one-time job. Expired signatures mean SERVFAIL. Automate re-signing and monitor it.
- **Changing keys without updating the DS.** A broken chain takes the domain down for validating users, and the servers look healthy in a plain test.
- **Lowering the TTL at the same time as the change.** Caches already hold the old TTL. The wait must come first.
- **Saying "DNS takes 48 hours to propagate".** Nothing spreads. Caches expire by TTL.
- **Assuming NXDOMAIN and NODATA are the same.** NXDOMAIN means no such name. NODATA means the name exists without that type.
- **Testing a name before creating it.** You create a negative cache entry. Then the new record seems not to work.
- **Leaving AXFR open to anyone.** This gives away your full host list.
- **Treating TSIG as encryption.** It proves who sent the data and that it is unchanged. The data is still readable.
- **"There are only 13 root servers."** There are 13 names and addresses, and far more physical servers through anycast.
- **Treating GeoDNS as a client-location system.** It usually sees the resolver, not the client, unless ECS is in use.
- **Running a public open resolver.** It can be used for amplification and is easier to poison. Limit recursion to your own networks.
- **Ignoring DNS in security logs.** Tunnelling and malware commands pass straight through if nobody watches queries.

## Interview Questions

**Q: When does DNS use TCP instead of UDP?**

A: DNS uses UDP by default. It uses TCP when a reply is too large for UDP, because the server sets the TC bit and the client retries over TCP. AXFR always uses TCP, and IXFR usually does. Servers can also force TCP during rate limiting. TCP support is mandatory under RFC 7766, so blocking TCP 53 can break DNSSEC and other large replies.

**Q: What is EDNS0 and why is 1232 bytes a common buffer size?**

A: EDNS0 adds an OPT pseudo-record that lets a client advertise a UDP size bigger than the old 512-byte limit. It also carries the DO bit for DNSSEC and options like cookies. 1232 comes from the IPv6 minimum MTU of 1280, minus 40 for IPv6 and 8 for UDP. It avoids IP fragmentation, which is often dropped and can be abused. Anything bigger moves to TCP.

**Q: Explain the difference between AXFR and IXFR, and how a secondary learns about a change.**

A: AXFR sends the full zone over TCP. IXFR sends only changes since a given serial, so it suits large zones. The primary sends a NOTIFY when the zone changes. The secondary then compares the SOA serial and asks for a transfer if the primary's serial is higher. If NOTIFY is lost, the SOA refresh timer still catches the change. I would protect transfers with an IP allow list and TSIG, and use XoT if the content must be private.

**Q: What happens if a secondary cannot reach the primary?**

A: It keeps serving its copy and retries at the retry interval. If the expire time passes with no successful refresh, it stops answering for the zone, usually with SERVFAIL, because the data may be too old to trust. For example, with an expire of 1209600 that happens after 14 days.

**Q: Walk me through the DNSSEC chain of trust.**

A: The resolver starts with the root KSK as a trust anchor. It validates the root's DNSKEY set, then the DS record for com, which is signed by the root. The DS is a hash of com's KSK, so it links the two zones. It repeats this for example.com. Finally, it checks the RRSIG on the A record using the example.com ZSK. If everything passes, it sets the AD flag. If validation fails, the client gets SERVFAIL.

**Q: What is the difference between DNSSEC and DoH?**

A: DNSSEC gives authenticity and integrity: it proves the data came from the zone owner, but it hides nothing. DoH gives privacy for the link between the client and resolver, but it proves nothing about the data's origin. They solve different problems, and they work well together.

**Q: What was the Kaminsky attack and how was it fixed?**

A: In 2008, Dan Kaminsky showed that an attacker could ask a resolver for random, uncached subdomains. For each one, the attacker floods forged replies, guessing the 16-bit ID. Each reply includes a false name server for the whole domain. Because the names are new, the attacker never waits for a TTL. The urgent fix was source port randomisation, which raises guessing from about 65,000 options to billions. The real fix is DNSSEC, because forged data has no valid signature.

**Q: What is negative caching and how is its TTL chosen?**

A: A resolver caches NXDOMAIN and NODATA answers, so it does not ask again for missing data. The authoritative server includes the SOA in the reply, and the TTL is the smaller of the SOA record's TTL and its minimum field. This is why a record can seem missing after you create it, if someone looked it up just before.

**Q: Explain split-horizon DNS and one risk.**

A: The same name gets different answers based on who asks, normally by source IP. Internal users get 10.0.5.20, and the public gets 203.0.113.20. It hides internal details and keeps traffic short. A risk is that clients using public DoH or moving off the network get the wrong view. The two zone versions can also drift apart.

**Q: How do GeoDNS and anycast differ?**

A: GeoDNS is a DNS decision: the authoritative server gives different IPs based on the location of the resolver or the client subnet. Failover depends on TTLs. Anycast is a routing decision: one IP is announced from many sites, and BGP sends each packet to the nearest. Failover depends on BGP. Large DNS providers run their name servers on anycast and may also use GeoDNS in their answers.

**Q: How would you detect and stop DNS tunnelling?**

A: Look for long, random-looking labels, many unique subdomains under one domain, unusual volumes of TXT or NULL queries, and hosts that query all day. To stop it, force all clients through company resolvers, block outbound 53 and 853 elsewhere, add RPZ or DNS filtering, and control DoH with policy and server blocklists.

**Q: How would you migrate a website to a new IP with no downtime?**

A: First, lower the TTL, for example from 86400 to 300, and wait at least the old TTL so that old caches expire. Test the new server directly with `curl --resolve`. Then change the record and keep both servers running. Monitor traffic, and keep the short TTL so I can roll back in minutes. When the old server is quiet, turn it off, then raise the TTL again. I would also check CNAME chains and long-lived connections.

## Quick Recap

- DNS uses both UDP and TCP on port 53. TC=1 moves a query to TCP, and AXFR is always TCP.
- EDNS0 adds the OPT pseudo-record for larger UDP sizes, the DO bit and options. 1232 bytes avoids fragmentation.
- Primaries copy zones to secondaries with NOTIFY, AXFR and IXFR. The SOA serial and timers control this, and TSIG protects it.
- Negative answers are cached for the smaller of the SOA TTL and SOA minimum.
- Kaminsky used random subdomains to poison whole domains. Port randomisation slows it, and DNSSEC stops forged data.
- DNSSEC signs data (DNSKEY, RRSIG, DS, NSEC) and gives authenticity. DoT, DoH and DoQ encrypt the link and give privacy.
- Split-horizon answers by who asks, GeoDNS answers by location, and anycast routes one IP to the nearest site. Before a migration, lower the TTL and wait the old TTL first.
