# Protocols, Standards and Why We Use Layers

**Level:** Beginner | **Prerequisites:** [What Is a Network?](../01-foundations/01-what-is-a-network.md), [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md)

## In One Line

A protocol is an agreed set of rules for how devices talk, a standard is a published protocol that everyone can follow, and layers split the huge job of networking into small parts that can change on their own.

## Why It Matters

A Dell laptop running Windows can load a page from a Linux server in another country. The path crosses Wi-Fi, Ethernet, and undersea fibre, using equipment from many companies. None of these companies needed to talk to each other first.

This works for two reasons:

- **Shared rules.** Every device follows the same public documents.
- **Separation of jobs.** Wi-Fi does not need to know what a web page is. The browser does not need to know what radio is.

Interviewers ask about this topic for three reasons. It shows you understand why the OSI and TCP/IP models exist, and not only their layer names. It tests whether you know who defines what (for example, the IETF defines TCP, while the IEEE defines Ethernet). And senior candidates are expected to read an RFC when a vendor manual is not enough.

## Simple Explanation

**A protocol is like the rules of a phone call.** Someone answers with "Hello". You say who you are. You take turns. You end with "Goodbye". If one person speaks only Spanish and the other only Korean, the call fails even though the phone line works. A working cable is not enough. Both sides need the same rules.

Where the analogy stops: people can guess and recover from confusion. Computers cannot guess. One wrong bit in the wrong place can make a device drop the whole message. So protocols must define everything exactly.

**A standard is like the shape of a wall socket.** If every maker builds plugs to the same public shape, any lamp works in any socket. You do not need to buy your lamp from the company that built your house.

Where the analogy stops: nobody inspects network products by law the way electrical safety is inspected. Internet standards are voluntary. Companies follow them because a device that cannot talk to others is hard to sell.

**Layers are like sending a letter by post.** You write the letter. The post office puts it on a truck. A plane may carry it. You do not care about the truck. The truck driver does not read your letter. If the post office changes from trucks to trains, you still write letters the same way.

Where the analogy stops: a post office does not split your letter into 1,000 small pieces, and it does not resend pieces that get lost. Networks do both of these things. Also, a letter gets one envelope, but network data gets a new wrapper at several layers.

## How It Works

### What a protocol is

A **protocol** is a set of rules that two or more devices agree to follow when they exchange data. It answers three questions:

| Part | Question it answers | Plain words |
|------|---------------------|-------------|
| **Syntax** | What does a message look like? | Format: which field goes where, and how long it is |
| **Semantics** | What does each part mean? | Meaning: what a value says, and what to do about it |
| **Timing** | When and how fast? | Order and speed: who speaks first, how long to wait |

Think of a train ticket. The **syntax** is the layout: date in the top corner, seat number on the right. The **semantics** is the meaning: "12C" means carriage 12, seat C. The **timing** is the rule that you must board before the train leaves, not after.

Where the analogy stops: a ticket is one object, while a protocol is a whole conversation with many messages.

**Note:** some textbooks say "synchronization" in place of "timing". The idea is the same.

### Syntax, semantics and timing in real protocols

| Protocol | Syntax | Semantics | Timing |
|----------|--------|-----------|--------|
| IPv4 | First 4 bits are the version, header is at least 20 bytes | TTL reaches 0: drop the packet and report it | Each packet travels alone, so no order is promised |
| TCP | Source port is the first 16 bits, flags are single bits | SYN flag means "I want to start a connection" | SYN, then SYN-ACK, then ACK. Resend if no reply comes |
| DNS | 12-byte header, then the question | Response code 3 means "this name does not exist" | Client asks first, retries if no answer comes in time |
| HTTP/1.1 | Text lines, for example `GET / HTTP/1.1` | 200 means success, 404 means not found | Client speaks first, server replies to each request |

Timing has two sides:

- **Sequence:** who speaks first, and what comes next. In HTTP the client speaks first.
- **Speed:** how fast one side may send. A fast server must not bury a slow phone in data. TCP handles this with [flow control](../06-transport-layer/05-tcp-reliability-and-flow-control.md).

Timers are part of timing too. For example, RFC 6298 says TCP should start with a retransmission timeout of 1 second. Another example: two [OSPF](../05-routing/06-ospf.md) routers will not become neighbours if their hello timers do not match, even though every packet is well formed.

### Common ways to describe a protocol

Interviewers like these labels. Each is a design choice:

| Label | Meaning | Example |
|-------|---------|---------|
| Connection-oriented | Sides set up a session before sending data | TCP |
| Connectionless | Each message is sent with no setup | UDP, IP |
| Stateful | The protocol remembers past messages | TCP |
| Stateless | Each message stands alone | HTTP (at the protocol level) |
| Text-based | Humans can read the messages | HTTP/1.1, SMTP |
| Binary | Messages are packed bits | TCP, IP, HTTP/2 |
| Open | Anyone can read and build it | OSPF, BGP |
| Proprietary | Controlled by one company | Cisco HSRP was created this way |

**Fuller truth:** these labels can mix. HTTP is stateless, but websites add state with cookies. EIGRP started as Cisco-only and was later published as RFC 7868, an informational document. HSRP is described in RFC 2281, which is also informational. Publishing a protocol does not make it an IETF standard.

A group of protocols designed to work together is called a **protocol suite**. The TCP/IP suite is the main example. When a device runs a suite, the software is called its **protocol stack**, since the layers sit on top of each other.

### Why layering exists

Networking is a very large job. It must turn bits into signals, find a path across the world, recover lost data, and show a page. Doing this in one giant program would be impossible to maintain.

So engineers split the work into **layers**. A layer is a group of related jobs. Each layer does three things:

1. It **gives a service** to the layer above. Example: TCP gives "reliable delivery" to the browser.
2. It **uses a service** from the layer below. Example: TCP asks IP to carry its segments.
3. It **talks to its peer**, meaning the same layer on the other device, using a protocol.

The meeting point between two layers on one device is called an **interface**. A protocol works sideways between devices. An interface works up and down inside one device.

```text
     Laptop                                 Web server

 +-------------+     HTTP rules          +-------------+
 | Application | <=====================> | Application |
 +-------------+                         +-------------+
       |                                       ^
       v                                       |
 +-------------+     TCP rules           +-------------+
 | Transport   | <=====================> | Transport   |
 +-------------+                         +-------------+
       |                                       ^
       v                                       |
 +-------------+     IP rules            +-------------+
 | Internet    | <=====================> | Internet    |
 +-------------+                         +-------------+
       |                                       ^
       v                                       |
 +-------------+     Ethernet or Wi-Fi   +-------------+
 | Link        | ----------------------> | Link        |
 +-------------+                         +-------------+

 <===>  logical talk between peers (feels direct, is not)
 ---->  the real path the bits take
```

The Application layer on the laptop feels like it talks directly to the Application layer on the server. In fact, the data goes down one side, across the wire, and up the other side.

This picture shows four layers, as in the [TCP/IP model](03-tcp-ip-model.md). The [OSI model](02-osi-model.md) uses seven. The number differs, but the idea does not. How each layer adds its own wrapper is covered in [Encapsulation](04-encapsulation.md).

### What layering gives us

**Fewer things to build.** Say you have 5 applications and 4 network types (Ethernet, Wi-Fi, 5G, satellite). Without layers, each application needs code for each network: 5 x 4 = 20 versions. With layers, each application talks to one shared layer, and each network connects to it once: 5 + 4 = 9 pieces.

```text
  Web     Mail     Video     Games     <- many applications
     \      \       /       /
      \      \     /       /
           TCP   UDP                   <- few transports
              \  /
               IP                      <- one shared layer
              /  \
             /    \
  Ethernet  Wi-Fi  5G  Satellite       <- many networks
```

This shape is called the **hourglass**. IP is the narrow middle. Anything above can run over anything below, as long as it passes through IP.

**Other benefits:**

- **Change one part at a time.** HTTP/3 replaced TCP with QUIC under the web, and web pages did not change.
- **Find faults faster.** If `ping` works but the website fails, the lower layers are likely fine. This idea is used in [troubleshooting](../12-troubleshooting-and-tools/01-troubleshooting-method.md).
- **Divide the work.** A switch maker can focus on Ethernet and a browser team can focus on HTTP.
- **Mix vendors.** A Cisco router and a Juniper router can run the same protocol.

**Costs of layering:**

- **Overhead.** Each layer adds its own **header** (extra control data placed before the real data). A TCP segment over IPv4 has at least 40 bytes of headers. In a 1,500-byte packet that is about 2.7 percent.
- **Hidden information.** A layer does not see details below it, which can cause bad choices. Going Deeper below shows an example.
- **Repeated work.** Ethernet, IPv4 and TCP all carry their own error check.

### What a standard is

A **standard** is a public document that defines a protocol or a format, so that different makers can build things that work together. This ability to work together is called **interoperability**.

A standard is not the same as an **implementation**. The standard is the recipe. An implementation is the actual software or hardware built from it. Linux TCP and Windows TCP are different code, but both follow RFC 9293, so they work together.

Standards also leave room for choice. For example, the starting TTL of a packet is commonly 64 on Linux, 128 on Windows and 255 on Cisco IOS. These are vendor defaults, not one fixed value from the standard.

Types you should know:

| Type | Meaning | Example |
|------|---------|---------|
| De jure ("by law") | Formally approved by a recognised body | IEEE 802.3 Ethernet |
| De facto ("in fact") | Became standard because everyone used it | The Berkeley sockets API |
| Open | Public, anyone can implement it | TCP, OSPF |
| Proprietary | Owned and controlled by one company | Cisco GLBP |

### The standards bodies

| Body | Full name | What it controls | Famous work |
|------|-----------|------------------|-------------|
| **IETF** | Internet Engineering Task Force | Internet protocols | TCP, IP, DNS, BGP, HTTP, TLS |
| **IEEE** | Institute of Electrical and Electronics Engineers | Local network technology | 802.3 Ethernet, 802.11 Wi-Fi |
| **ISO** | International Organization for Standardization | Standards for almost every industry | OSI model (ISO/IEC 7498-1) |
| **ITU** | International Telecommunication Union | Global telecom and radio | X.509, E.164, H.264 |
| **W3C** | World Wide Web Consortium | Web technologies | CSS, WebAuthn, WCAG |
| **ICANN** | Internet Corporation for Assigned Names and Numbers | Global coordination of names and numbers | Top-level domain policy |
| **IANA** | Internet Assigned Numbers Authority | The master lists of numbers | Port numbers, root zone, IP address pools |

**IETF.** An open group of engineers. There is no formal membership. Anyone can join its mailing lists. Work is done in **working groups**, each with one topic. It meets three times a year. Its well-known method is "rough consensus and running code": most people agree, and real software proves the idea works. Its output is the RFC.

**IEEE.** A professional association. Its **802 committee** handles local and metro networks. Names you must know:

- 802.1Q: [VLAN tagging](../03-ethernet-and-switching/07-trunking-8021q.md)
- 802.1X: login control on network ports
- 802.3: [Ethernet](../03-ethernet-and-switching/02-ethernet-frames.md)
- 802.11: [Wi-Fi](../08-wireless-and-mobile/01-wifi-basics.md)

Letters after a number are usually additions to the standard, for example 802.3ab (gigabit Ethernet over copper) and 802.11ax (Wi-Fi 6). IEEE also hands out the first part of [MAC addresses](../03-ethernet-and-switching/01-mac-addresses.md).

**ISO.** Works through national standards bodies from many countries. Its best-known networking work is the OSI model. The OSI protocols themselves mostly lost to TCP/IP, but the OSI model survived as a teaching tool. One OSI protocol, [IS-IS](../05-routing/08-is-is.md), is still used in large provider networks. Fun fact: "ISO" is not an acronym. It comes from the Greek word "isos", meaning equal.

**ITU.** A United Nations agency whose members are mainly governments, with companies taking part too. It has three sectors. **ITU-T** writes telecom standards, called Recommendations. **ITU-R** manages radio spectrum. **ITU-D** supports development. Well-known ITU-T documents are X.509 (the certificate format used by TLS), E.164 (phone number format) and G.711 (a voice codec, meaning a way to encode sound).

**W3C.** Founded in 1994 by Tim Berners-Lee, who invented the web. It writes standards for what runs in the browser, such as CSS. Two facts catch people out:

- HTTP belongs to the IETF, not W3C.
- HTML is now maintained as a "living standard" by another group, WHATWG (Web Hypertext Application Technology Working Group), which browser makers lead.

**ICANN and IANA.** These are not protocol writers. They manage shared numbers and names, so no two people use the same one by accident. Think of a land registry: it does not build houses, it records who holds which plot.

Where the analogy stops: IANA does not record every final owner. It hands large blocks to regional groups, and they pass smaller blocks down.

ICANN is a non-profit founded in 1998. It sets policy for domain names. IANA is the name for the technical work of keeping the lists. Since 2016, that work has been done by PTI (Public Technical Identifiers), an affiliate of ICANN. IANA looks after three things:

1. **Domain names:** the DNS root zone, the top of the name tree.
2. **Number resources:** IP addresses and Autonomous System (AS) numbers.
3. **Protocol parameters:** port numbers, IP protocol numbers, and more.

IP addresses flow down like this:

```text
 IANA
  |   gives large blocks to
  v
 RIR  (Regional Internet Registry)
      ARIN, RIPE NCC, APNIC, LACNIC, AFRINIC
  |   gives smaller blocks to
  v
 ISP or large organisation
  |   gives addresses to
  v
 Customer
```

### Other bodies worth knowing

| Body | What it does |
|------|--------------|
| **3GPP** (3rd Generation Partnership Project) | Writes the mobile standards: LTE and 5G |
| **Wi-Fi Alliance** | Tests products and owns the "Wi-Fi 6" style names. The technical standard is still IEEE 802.11 |
| **TIA** (Telecommunications Industry Association) | Cabling standards, such as TIA-568 |
| **ISOC** (Internet Society) | Non-profit that supports the IETF |
| **IAB** and **IESG** | IETF leadership: the IAB (Internet Architecture Board) looks at architecture, and the IESG (Internet Engineering Steering Group) approves documents |

### What an RFC is

**RFC** means **Request for Comments**. An RFC is a numbered document published by the RFC Editor. It is the main way internet protocols are written down. The series started with RFC 1 in April 1969, written by Steve Crocker. The name stayed, although a modern RFC is a finished document, not a draft asking for ideas.

Key rules:

- **An RFC never changes after publication.** The text is frozen. A fix comes as a new RFC, or as an **erratum** (a listed correction on the RFC Editor website).
- **A new RFC can replace an old one.** RFC 9293 **obsoletes** RFC 793, so RFC 793 is now only history. An RFC can also **update** another RFC, meaning it adds to it without fully replacing it.
- **Not every RFC is a standard.** This is the most common mistake.

Think of a law book where no page is ever erased. To change a law, parliament passes a new law that says "this replaces the old one".

Where the analogy stops: nobody goes to jail for breaking an RFC. The only penalty is that your product may not work with others.

| Status | Meaning | Example |
|--------|---------|---------|
| Proposed Standard | Reviewed and stable. Many protocols stay here for years | RFC 9000 (QUIC) |
| Internet Standard | Mature and widely used. Gets a STD number | RFC 9293 (TCP), STD 7 |
| Best Current Practice (BCP) | Advice on how to operate | RFC 1918 (private addresses), BCP 5 |
| Informational | For information only, not a standard | RFC 7868 (EIGRP) |
| Experimental | Being tried out | RFC 7413 (TCP Fast Open) |
| Historic | Replaced or no longer advised | RFC 6101 (SSL 3.0) |

A **STD number** names a topic and can point to a newer RFC over time. TCP was STD 7 under RFC 793 and is still STD 7 under RFC 9293.

Some RFCs are jokes, published on 1 April. RFC 1149 describes carrying IP packets by pigeon. RFC 2324 gave us the "I'm a teapot" error. This is another reason that "it is an RFC" does not mean "it is a standard".

### How a protocol becomes an RFC (IETF path)

```text
 Idea
  |
  v
 Internet-Draft   Anyone can write one. Expires after about 6 months
  |
  v
 Working group    Review, changes, rough consensus
  |
  v
 IETF Last Call   Whole community can comment
  |
  v
 IESG approval
  |
  v
 RFC Editor       Assigns a number, publishes, text is now frozen
```

An **Internet-Draft** is work in progress. It is not a standard, even if some products already use it.

**Fuller truth:** this is the IETF path. RFCs can also come from other streams, such as the IRTF (Internet Research Task Force), the IAB, and independent authors.

### How to read an RFC

Do not read an RFC from start to end like a novel. RFC 9293 alone runs to nearly 100 pages. Use this order:

1. **Find the current one.** Search at `rfc-editor.org` or `datatracker.ietf.org`. Look for "Obsoleted by" at the top. If you see it, go to the newer RFC.
2. **Read the header block.** It gives the number, date, status, and what it obsoletes or updates.
3. **Read the Abstract and Introduction.** They tell you the purpose and the scope.
4. **Read the terminology section.** RFCs define words very exactly.
5. **Jump to the part you need.** This is usually the message format or the state machine (a list of states and what moves you between them).
6. **Skim Security Considerations.** Modern RFCs must have this section. It often explains real attacks.
7. **Check the IANA Considerations.** This lists numbers the protocol registers.
8. **Check the errata.** A known mistake may already be listed.

### The words MUST, SHOULD and MAY

RFC 2119 and RFC 8174 give certain words an exact meaning. Together they are known as BCP 14. The meaning applies only when the word is in CAPITALS.

| Keyword | Meaning | Daily-life comparison |
|---------|---------|-----------------------|
| **MUST** | Required. No exceptions | You must stop at a red light |
| **MUST NOT** | Forbidden | You must not drive the wrong way |
| **SHOULD** | Do it, unless you have a good reason and understand the result | You should wear a coat in snow |
| **SHOULD NOT** | Avoid it, unless you have a good reason and understand the result | You should not drive when very tired |
| **MAY** | Truly optional | You may play the radio |

The words REQUIRED and OPTIONAL have similar exact meanings. In lowercase, all of these words are just normal English.

## Example

### Example 1: Who defines what when you open a website

You open `https://example.com` on a laptop on Wi-Fi. Here is who wrote the rules for each step:

| Step | Technology | Who defines it | Document |
|------|------------|----------------|----------|
| Radio to access point | Wi-Fi | IEEE | 802.11 |
| Access point to router | Ethernet | IEEE | 802.3 |
| Find the IP of the name | DNS | IETF | RFC 1034, RFC 1035 |
| Domain name `.com` level | Name system | ICANN policy, IANA root zone | Not a protocol |
| Server IP address block | Address allocation | IANA, then an RIR | Not a protocol |
| Move packets | IPv4 | IETF | RFC 791 |
| Reliable connection | TCP | IETF | RFC 9293 |
| Port 443 for HTTPS | Port registry | IANA | Online registry |
| Encryption | TLS 1.3 | IETF | RFC 8446 |
| Server certificate format | X.509 | ITU-T, with IETF profile | X.509, RFC 5280 |
| Web request | HTTP | IETF | RFC 9110 |
| Page styling | CSS | W3C | CSS specifications |

Now plug in an Ethernet cable and turn Wi-Fi off. Only the first row changes. The browser, TLS, TCP and HTTP work the same. That is layering in action. The full journey is in [What Happens When You Open a Website](05-what-happens-when-you-open-a-website.md).

### Example 2: See syntax, semantics and timing in HTTP

The output below shows a plain HTTP request. The server IP is a documentation address, used here as an example. Real output will show the real address.

Linux:

```bash
curl -v http://example.com/
```

Windows (type `curl.exe`, because in Windows PowerShell 5.1 plain `curl` is an alias for another command):

```powershell
curl.exe -v http://example.com/
```

Output (shortened):

```text
*   Trying 203.0.113.10:80...
* Connected to example.com (203.0.113.10) port 80
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html
< Content-Length: 1256
<
<!doctype html>
```

Lines with `>` are sent by curl. Lines with `<` come from the server.

- **Syntax:** the first line must be a method, a path and a version, with single spaces between them. An empty line ends the headers.
- **Semantics:** `GET` means "send me this resource". `200` means success. `Content-Length: 1256` means the body is 1,256 bytes long.
- **Timing:** TCP connects first (`Connected to`). Then the client speaks. Then the server answers. The server never starts this conversation.

Now break the syntax on purpose. This Linux command sends text that is not valid HTTP, using `nc` (netcat, a tool that opens a raw connection):

```bash
printf 'HELLO THERE\r\n\r\n' | nc example.com 80
```

A typical reply starts like this (the exact text varies by server):

```text
HTTP/1.1 400 Bad Request
```

The network path worked. TCP worked. But the rules of HTTP were broken, so the server refused the message.

### Example 3: Read a real RFC header and a real format

Download the start of the TCP standard.

Linux:

```bash
curl -s https://www.rfc-editor.org/rfc/rfc9293.txt | head -n 10
```

Windows PowerShell:

```powershell
curl.exe -s https://www.rfc-editor.org/rfc/rfc9293.txt |
  Select-Object -First 10
```

The header block looks like this:

```text
Internet Engineering Task Force (IETF)                      W. Eddy, Ed.
STD: 7                                                       MTI Systems
Request for Comments: 9293                                   August 2022
Obsoletes: 793, 879, 2873, 6093, 6429, 6528, 6691
Updates: 1011, 1122, 5961
Category: Standards Track
ISSN: 2070-1721
```

Read it line by line:

- `Internet Engineering Task Force (IETF)`: this RFC came through the IETF.
- `STD: 7`: it is a full Internet Standard, number 7.
- `Obsoletes: 793, ...`: stop using RFC 793 and these others.
- `Updates: 1011, 1122, 5961`: those RFCs are still valid, but this one changes parts of them.
- `Category: Standards Track`: this is a standard, not just information.

RFCs draw message formats as bit diagrams. This is the start of the [TCP header](../06-transport-layer/03-tcp-handshake-and-header.md):

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

How to read it: the numbers on top count bits, 0 to 31. So each row is 32 bits, or 4 bytes. Each field spans the bits under it. Source Port spans 16 bits (2 bytes).

Now decode four real bytes from the start of a TCP segment: `C3 50 01 BB`.

| Bytes | Field | Hex value | Decimal | Meaning |
|-------|-------|-----------|---------|---------|
| `C3 50` | Source Port | 0xC350 | 50000 | Client's temporary port |
| `01 BB` | Destination Port | 0x01BB | 443 | HTTPS on the server |

The math: 0x01BB = (1 x 256) + (11 x 16) + 11 = 256 + 176 + 11 = 443.

You just read a protocol the way a device does. Syntax told you where the fields are. Semantics told you that 443 means HTTPS, using the IANA port registry.

The same RFC also gives clear requirements. It labels them, so you can search for them. For example, MUST-2 says the sender must generate the TCP checksum, and MUST-3 says the receiver must check it.

### Example 4: Find IANA numbers on your own machine

Your computer keeps a small local copy of some IANA lists.

Linux:

```bash
grep -E '^(icmp|tcp|udp)[[:space:]]' /etc/protocols
grep -w 443 /etc/services
```

```text
icmp    1       ICMP        # internet control message protocol
tcp     6       TCP         # transmission control protocol
udp     17      UDP         # user datagram protocol
https   443/tcp             # http protocol over TLS/SSL
https   443/udp             # HTTP/3
```

Windows PowerShell (the file is named `protocol`, with no "s"):

```powershell
$etc = "C:\Windows\System32\drivers\etc"
Get-Content "$etc\protocol" | Select-String '^(icmp|tcp|udp)\s'
Get-Content "$etc\services" | Select-String '\b443/'
```

```text
icmp       1     ICMP         # Internet control message protocol
tcp        6     TCP          # Transmission control protocol
udp        17    UDP          # User datagram protocol
https             443/tcp    MCom                   #HTTP over TLS/SSL
https             443/udp    MCom                   #HTTP over TLS/SSL
```

Wording and spacing vary by operating system and version. The numbers do not, because they come from IANA. So when an IPv4 header says protocol 6, every device in the world reads that as TCP.

## Going Deeper

**Layering is a guide, not a law.** Real systems often cross layers on purpose:

- [NAT](../04-ip-addressing/09-nat-and-pat.md) is done by a router, a network layer device, but PAT rewrites TCP and UDP port numbers.
- [MSS](../06-transport-layer/09-mss-mtu-and-pmtud.md) (Maximum Segment Size) in TCP depends on the MTU (Maximum Transmission Unit), the largest packet a lower layer can carry.
- TCP often treats packet loss as a sign of congestion. On a weak radio link, loss may come from noise instead. TCP cannot see this, since the cause sits below it.

RFC 3439 has a section titled "Layering Considered Harmful". It warns that too many layers can add complexity and hide useful information.

**End-to-end principle.** A 1984 paper by Saltzer, Reed and Clark argued that some functions can only be done correctly at the end points, not inside the network. This is why the internet core stays fairly simple and routers mainly forward packets. Reliable delivery is left to the hosts.

**Robustness principle and its limits.** Early TCP text said to be strict in what you send and liberal in what you accept. This helped early systems work together. But accepting bad input also allowed bugs to spread and become permanent. RFC 9413 discusses this problem and argues for keeping protocols healthy through active use.

**Protocol ossification.** Ossification means a protocol becomes hard to change, like soft tissue turning to bone. Firewalls and other middleboxes (devices in the path that inspect traffic) can block anything they do not recognise. Two real results:

- TLS 1.3 places an older version number in a legacy field and signals the real version in an extension, so that old middleboxes let it through.
- QUIC runs over UDP and encrypts most of its headers, so that middleboxes cannot depend on them.

**Designing for change.** Good protocols plan for the future with version fields, reserved bits, and option or extension fields. The first 4 bits of an IP packet give the version: 4 for IPv4 and 6 for IPv6. Binary protocols also fix the byte order. Internet protocols usually send the most significant byte first, called **network byte order** (big-endian). This is why `01 BB` above reads as 0x01BB.

## Common Mistakes

- **"Every RFC is a standard."** Many are informational, experimental, historic, or even jokes. Check the status.
- **"RFC 793 defines TCP."** That was true for decades. The current document is RFC 9293.
- **"An RFC can be edited to fix a mistake."** It cannot. Fixes come as errata or a new RFC.
- **"W3C owns HTTP."** The IETF owns HTTP. W3C focuses on browser-side standards.
- **"Wi-Fi Alliance writes Wi-Fi."** IEEE writes 802.11. Wi-Fi Alliance certifies products and owns the brand names.
- **"ICANN and IANA create protocols."** They coordinate names and numbers. The IETF writes protocols.
- **"IANA gives out IP addresses to ISPs."** Normally IANA gives blocks to RIRs, and RIRs serve ISPs.
- **"A protocol and an implementation are the same."** OpenSSL is software. TLS is the protocol.
- **"SHOULD means optional."** SHOULD is much stronger than MAY. You skip it only with a good reason.
- **"Layers are physical parts of a device."** A layer is a way of organising jobs. Many layers exist only as software.
- **"The OSI model is what the internet runs."** The internet runs on the TCP/IP suite. OSI is mostly a reference model.
- **"An open standard must be free of patents."** Open means publicly documented and available to implement. Patent rules differ by body.

## Interview Questions

**Q: What is a protocol?**
A: A protocol is an agreed set of rules for how devices exchange data. It defines the message format, what each part means, and when messages are sent. Those are syntax, semantics and timing.

**Q: Explain syntax, semantics and timing with one example.**
A: In HTTP, syntax is the format of a line like `GET / HTTP/1.1`. Semantics is the meaning, for example status 404 means not found. Timing is the order: the client sends a request first, then the server replies.

**Q: Why do networks use layers?**
A: Layers split a big problem into small jobs. Each layer gives a service to the one above and can change without affecting the others. For example, you can move from Wi-Fi to Ethernet without changing the browser. This also makes it easier to troubleshoot and lets different vendors work together.

**Q: What are the downsides of layering?**
A: Each layer adds header overhead, so less of each packet is real data. Layers hide details from each other, so TCP cannot tell if loss came from radio noise or congestion. Some work is also repeated, such as error checks at several layers.

**Q: What is the difference between a protocol and an interface in a layered model?**
A: A protocol is between the same layers on two different devices. An interface is between two neighbouring layers on the same device. So protocols work sideways, and interfaces work up and down.

**Q: What is an RFC? Is every RFC a standard?**
A: RFC means Request for Comments. It is a numbered, permanent document in the series that describes internet protocols and practices. No, not every RFC is a standard. Some are informational, experimental or historic. You check the status at the top of the document.

**Q: Can an RFC be changed after it is published?**
A: No. The text is frozen. Known mistakes go on an errata list, and real changes need a new RFC that updates or obsoletes the old one. RFC 9293 obsoleted RFC 793, for example.

**Q: What is the difference between MUST, SHOULD and MAY in an RFC?**
A: MUST is an absolute requirement. SHOULD is a strong recommendation that you can skip only with a good reason and full understanding of the result. MAY is truly optional. These meanings apply when the words are in capitals, as RFC 2119 and RFC 8174 explain.

**Q: Which organisations define Ethernet, Wi-Fi, TCP and HTML?**
A: IEEE defines Ethernet as 802.3 and Wi-Fi as 802.11. The IETF defines TCP in RFC 9293. HTML is maintained by WHATWG as a living standard, and W3C works with it. CSS stays with W3C.

**Q: What is the difference between ICANN and IANA?**
A: ICANN is the non-profit that coordinates global policy for names and numbers. IANA is the set of technical functions that keeps the lists: the DNS root zone, IP address pools, and protocol parameters like port numbers. Today PTI, an ICANN affiliate, performs the IANA work.

**Q: How does an IP address reach a customer?**
A: IANA gives large blocks to the five Regional Internet Registries: ARIN, RIPE NCC, APNIC, LACNIC and AFRINIC. They give smaller blocks to ISPs and large organisations. The ISP then assigns addresses to customers.

**Q: What is the difference between a standard and an implementation?**
A: A standard is the written rules. An implementation is real software or hardware built from those rules. Linux and Windows have different TCP code, but both follow RFC 9293, so they interoperate. Defaults can still differ, such as starting TTL values.

## Quick Recap

- A protocol is a set of rules with three parts: syntax (format), semantics (meaning), timing (order and speed).
- Layers split networking into jobs. Each layer serves the one above, uses the one below, and talks to its peer.
- Layering reduces work and allows change, but it adds overhead and hides details.
- IETF writes internet protocols, IEEE writes Ethernet and Wi-Fi, ISO gave us OSI, ITU covers telecom, W3C covers the web platform.
- ICANN and IANA manage names and numbers, and they do not write protocols.
- An RFC is a permanent numbered document, and not every RFC is a standard. Always check its status and whether it is obsoleted.
- In RFCs, capital MUST, SHOULD and MAY have exact meanings.
