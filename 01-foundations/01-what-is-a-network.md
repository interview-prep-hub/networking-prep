# What Is a Network?

**Level:** Beginner | **Prerequisites:** None

## In One Line

A network is two or more devices connected so they can exchange data. The internet is a huge group of separate networks that agree on one common language, and data crosses it in small pieces called packets.

## Why It Matters

Every other topic in this course builds on the words in this chapter. If "host", "packet" and "ISP" are not clear, later topics like routing and TCP will feel like magic.

### Why networks exist

A computer alone can only use what is inside it. Networks remove that limit. People build them for these reasons:

- **Communication.** Email, chat, and video calls all need two devices to exchange data.
- **Sharing resources.** Ten people in an office can use one printer and one internet connection. Nobody needs ten of each.
- **Sharing data.** A company keeps customer records on one central system. Everyone sees the same correct copy.
- **Remote access.** An engineer in Lahore can manage a server in Frankfurt without travelling.
- **Reliability.** If data lives on many machines, one broken machine does not stop the service.
- **Scale.** A website can use thousands of machines working together. One machine could never do that work alone.

### Why shared equipment matters

Imagine you connect every device directly to every other device. The number of cables is `n x (n - 1) / 2`, where `n` is the number of devices.

| Devices | Direct cables needed |
|---|---|
| 2 | 1 |
| 10 | 45 |
| 1,000 | 499,500 |

That cost grows far too fast. So real networks share cables and use special devices to move data between them. Sharing creates the hard questions in networking: who may use the cable now, and how does data find the right place? The rest of this course answers those questions.

### Why interviewers ask

Interviewers use these basics to check if you really understand, or only memorised words. Good first-round questions include:

- What is the difference between the internet and the web?
- Is a server a type of hardware?
- Why does the internet use packets?

A clear, simple answer here builds trust for the rest of the interview.

## Simple Explanation

### The road analogy

Think of a network like a road system.

- A **house** is a device, such as your laptop.
- A **road** is the connection, such as a cable or a Wi-Fi signal.
- A **street address** tells people where to deliver something.
- **Traffic rules** let everyone use the roads without crashing.

Each town has its own local roads. Highways connect the towns. No single person owns all the roads in the world, yet you can drive from one town to another because the roads connect and the rules match.

The internet works the same way. Each company, school, and home has its own network. Larger networks connect them. Nobody owns the whole thing.

**Where the analogy stops being true:** a car stays one car for the whole trip. Data is cut into many small pieces, and each piece travels on its own. A traffic jam makes cars wait, but a very busy network can also throw pieces away.

### The postcard analogy for packets

Suppose you must send a whole book to a friend, but the post office only accepts postcards.

1. You copy one page onto each postcard.
2. You write your friend's address on each card.
3. You number each card: 1 of 300, 2 of 300, and so on.
4. You post them all.
5. Your friend puts them in order. If card 57 is missing, your friend asks for it again.

Those postcards are **packets**. The address and number are the **header**. The page of text is the **payload**.

**Where the analogy stops being true:** the post office tries hard never to lose a card. Network devices may drop a packet on purpose when they are full, and that is normal. Also, the post office does not tell you about lost cards. In a network, a protocol (a set of agreed rules) such as TCP (Transmission Control Protocol) notices the loss and sends the data again.

## How It Works

### The parts of a network

Every network has four things:

1. **Nodes:** the devices.
2. **Links:** the connections between them.
3. **Addresses:** the way to name each device.
4. **Protocols:** the agreed rules for talking. See [Protocols, Standards and Why We Use Layers](../02-network-models/01-why-layers-and-protocols.md).

Here are the words you need:

| Term | Plain meaning | Examples |
|---|---|---|
| Node | Any device on a network that can send, receive, or pass on data | Laptop, phone, printer, router |
| Host | A node that is an end point of communication. It runs programs for users and is not there to pass on other people's traffic | Laptop, phone, web server |
| Link | The physical or radio connection between two nodes | Ethernet cable, fibre, Wi-Fi |
| Interface | The point where a device connects to a link | A laptop's Wi-Fi card |
| Router | A device that moves data between different networks | Your home router |
| Switch | A device that connects devices inside one local network | A 24-port office switch |
| IP address | A number that identifies an interface on a network. IP stands for Internet Protocol | `192.168.1.10` |
| Bit | The smallest unit of data, a 0 or a 1 | `1` |
| Byte | A group of 8 bits | `01000001` |

Devices are covered in [Network Devices](04-network-devices.md). IP addresses are covered in [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md).

**Every host is a node, but not every node is a host.** A router is a node, but it is normally not a host, because its job is to pass on traffic for others.

**Fuller truth:** the word "host" is used loosely. Many people say "host" for anything with an IP address. The IPv6 standard (RFC 8200) is strict: a *node* is a device that runs IPv6, a *router* is a node that forwards packets not addressed to itself, and a *host* is any node that is not a router. Use the same idea in an interview, then mention that everyday use is looser.

### A small network

```text
                 Internet
                     |
             Public IP 203.0.113.25
              +-------------+
              | Home router |  LAN side: 192.168.1.1
              +-------------+
               /     |     \
              /      |      \
      +--------+ +--------+ +---------+
      | Laptop | | PC     | | Printer |
      | .1.10  | | .1.11  | | .1.20   |
      +--------+ +--------+ +---------+
```

The laptop, PC, and printer are hosts. In the diagram, `.1.10` is short for `192.168.1.10`. The home router is a node that connects this small network to the rest of the world. It is really a router, a small switch, and a Wi-Fi radio in one box. A LAN (local area network) is a network in one small place, such as a home. See [Types of Networks](02-network-types.md).

A **public IP address** can be used across the internet. A private address, such as one starting with `192.168`, works only inside a local network. See [Public, Private and Special IP Addresses](../04-ip-addressing/03-public-private-and-special-addresses.md).

### Client and server

Most network communication follows one pattern.

- A **client** is the program that asks for something.
- A **server** is the program that waits for requests and answers them.

When you open a website:

1. Your browser (the client) sends a request: "Please send me this page."
2. The web server receives the request.
3. The web server sends back the page.
4. Your browser shows it.

```text
  Client                                Server
  (browser)                             (web server)
     |                                       |
     |  1. Request: give me /index.html      |
     |-------------------------------------->|
     |                                       |
     |  2. Response: here is the page        |
     |<--------------------------------------|
     |                                       |
```

Three facts matter here:

- **Client and server are roles, not types of machine.** A cheap old laptop can be a server. A big machine can be a client.
- **The client usually starts the conversation.** The server listens on a known number called a [port](../06-transport-layer/01-ports-and-sockets.md), such as 443 for HTTPS (secure web traffic).
- **One machine can play both roles.** A web server is a server to your browser, but it is a client when it asks a database for data.

A restaurant is a good picture. The customer orders, the kitchen prepares, and the food comes back. **Where the analogy stops being true:** a restaurant fills up at a few dozen people, but one good server can handle many thousands of connections. Also, a server program can run on the same machine as its client, which is like cooking your own order.

The word "server" also has two meanings. It can mean the program (the web server software) or the computer that runs it. Listen to the context.

### Peer-to-peer

In **peer-to-peer** (P2P), each device is both a client and a server. Devices are called **peers** because they are equals. Peers share directly with each other.

```text
  Client-server                  Peer-to-peer

    C     C     C                  P ------- P
     \    |    /                   | \     / |
      \   |   /                    |   \ /   |
        Server                     |   / \   |
      /   |   \                    | /     \ |
     /    |    \                   P ------- P
    C     C     C
```

**Real examples:**

- **BitTorrent.** Each peer downloads pieces of a file from many others and uploads pieces too.
- **Direct file sharing** between two computers.
- **WebRTC calls.** Browsers can send audio and video directly to each other. A server still helps them find each other first.

| Point | Client-server | Peer-to-peer |
|---|---|---|
| Who provides the service | Central servers | The users themselves |
| Control and security | Easier, one place to manage | Harder, every peer is different |
| Cost | Owner pays for the servers | Users share the cost |
| Single point of failure | Yes, unless made redundant | No single one in pure P2P |
| Growth | More users means more load on servers | More users also means more capacity |
| Finding data | Simple, ask the server | Harder, must find which peer has it |
| Typical use | Websites, banking, email | File sharing, some calls |

**Fuller truth:** most real P2P systems are hybrids. BitTorrent often uses a tracker (a helper server) or a shared lookup system to find peers. P2P also has a practical problem: many homes are behind [NAT](../04-ip-addressing/09-nat-and-pat.md) (network address translation, where many devices share one public address), which makes it hard for one peer to reach another directly.

### What the internet really is

The **internet** is a global system of connected networks that all use the TCP/IP family of protocols. This family is named after its two best-known members, TCP and IP. The internet is a **network of networks**.

It is not a single thing in a single place. It is physical:

- **Fibre optic cables** (thin glass threads that carry light) under streets, along railways, and across oceans.
- **Submarine cables** on the sea floor, which carry almost all traffic between continents.
- **Routers** that pass data from one network to the next.
- **Data centres**, which are large buildings full of servers.
- **Access links** that reach homes and phones: fibre, cable TV lines, DSL (internet over telephone lines), and mobile radio.

Wi-Fi only covers the last few metres. Soon after that, your data is on a cable.

Each separate network that runs under one organisation's control and routing policy is called an **autonomous system** (AS). Each has a number. There are more than 70,000 active autonomous systems today. [BGP](../05-routing/09-bgp.md) is the protocol they use to tell each other which addresses they can reach.

```text
  Network edge                 Network core            Network edge
  (hosts live here)            (routers only)        (hosts live here)

  [Your laptop] -- [ISP] ---- [Big networks] ---- [ISP] -- [Server]
  [Your phone]  --   |            |     |           |   -- [Server]
                     |            |     |           |
                  [Other ISP] ----+     +------ [Data centre]
```

**Who runs it?** Nobody owns the internet. Different groups own different parts or coordinate different jobs:

| Who | Job |
|---|---|
| ISPs and network owners | Own and run the cables and routers |
| IETF (Internet Engineering Task Force) | Writes open technical standards, published as RFCs |
| ICANN and IANA | Coordinate global names, numbers, and protocol values |
| RIRs (Regional Internet Registries) | Give IP address blocks and AS numbers to their regions. There are five: ARIN, RIPE NCC, APNIC, LACNIC, AFRINIC |

An **RFC** (Request for Comments) is a numbered public document. Many internet standards are published this way.

**Internet vs the web.** They are different things.

- The **internet** is the network itself, the roads.
- The **World Wide Web** is one service that runs on it, the delivery vans. It is made of web pages reached with HTTP (Hypertext Transfer Protocol) and its secure form, HTTPS.

Email, SSH (secure remote login), online games, and DNS (the system that turns names into addresses) all use the internet, but they are not the web.

**Short history:**

| Year | Event |
|---|---|
| 1960s | Paul Baran and Donald Davies separately develop the idea of packet switching |
| 1969 | ARPANET, a US research network, connects its first sites |
| 1983 | ARPANET moves to TCP/IP on 1 January |
| 1989 | Tim Berners-Lee proposes the World Wide Web at CERN, the European physics laboratory |

### ISPs and tiers

An **ISP** (Internet Service Provider) is a company that connects customers to the internet. Your home ISP is one. Companies that connect other ISPs are ISPs too.

ISPs connect to each other in two main ways:

- **Transit:** you pay a bigger network to carry your traffic to the whole internet. It is like buying a ticket that works everywhere.
- **Peering:** two networks agree to exchange traffic directly, but only traffic to and from themselves and their customers. Often no money changes hands. This is called settlement-free peering.

Networks often peer at an **IXP** (Internet Exchange Point). An IXP is a place with shared switches, often spread across several data centres in one city, where many networks connect at once. Well-known examples are DE-CIX in Frankfurt, AMS-IX in Amsterdam, and LINX in London.

ISPs are often grouped into three tiers:

| Tier | Plain meaning | Pays for transit? | Examples |
|---|---|---|---|
| Tier 1 | Can reach the whole internet by peering alone | No | Lumen, Arelion, NTT, Cogent |
| Tier 2 | Peers with some networks, buys transit for the rest, sells service to others | Yes, for some traffic | Many national and regional carriers |
| Tier 3 | Mostly buys transit and sells access to end users | Yes | Many local and small access ISPs |

```text
  Tier 1 (AS64512) <---- peering, no fee ----> Tier 1 (AS64513)
       ^                                            ^
       | transit, paid                              | transit, paid
       |                                            |
  Tier 2 (AS64520) <---- peering at IXP -----> Tier 2 (AS64530)
       ^                                            ^
       | transit, paid                              | transit, paid
       |                                            |
  Access ISP (AS64540)                        Hosting (AS64550)
       ^                                            ^
       | monthly bill                               | monthly bill
       |                                            |
  Your home                                   example.com server
```

Each up arrow points from a customer to its provider. Money moves in the same direction as the arrow.

The AS numbers above are made-up private values, used only for this example.

**The road analogy again:** Tier 3 is your local street, Tier 2 is the regional road, and Tier 1 is the international highway. **Where the analogy stops being true:** "Tier 1" says who pays whom. It does not mean faster or better. A Tier 2 network can give a better path for your local traffic.

**Fuller truth:** tiers are industry language, not a formal standard. No official body gives out the title, and the list of Tier 1 networks changes and is sometimes argued about.

### Packets

A **packet** is a small unit of data sent across a network. It has two main parts.

```text
  +------------------------+----------------------------------+
  | Header                 | Payload                          |
  | delivery information   | a piece of your actual data      |
  +------------------------+----------------------------------+
```

The **header** is like the envelope. The **payload** is the letter inside. Routers read the header and normally ignore the payload.

Important fields in an IPv4 header, in simple form:

| Header field | What it means |
|---|---|
| Source address | Who sent this packet |
| Destination address | Where it must go |
| TTL (Time To Live) | A hop counter. It stops a lost packet from looping forever |
| Protocol | What is inside, such as TCP or UDP |
| Total length | Size of the packet |

The card number from the postcard story is not here. It is a sequence number in the TCP header, which travels inside the IP payload.

The full layout is in [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).

**How big is a packet?** On a normal Ethernet network, the largest IP packet is usually 1,500 bytes. This limit is called the **MTU** (maximum transmission unit). See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

**Why cut data into packets?**

- **Fair sharing.** Many users take turns on one link. A big download cannot block everyone until it finishes.
- **Easy recovery.** If one packet is lost, only that piece is sent again, not the whole file.
- **Flexible paths.** If a link fails, routers can send later packets another way.
- **Simple devices.** A router only needs to handle small pieces, one at a time.

**How a packet travels:**

1. A program has data to send.
2. The sending host cuts the data into pieces.
3. The host puts a header on each piece. This wrapping is called [encapsulation](../02-network-models/04-encapsulation.md).
4. The host sends each packet to the first router. A router that takes traffic out of a local network is called the [default gateway](../04-ip-addressing/08-default-gateway.md). If the destination is on the same local network, the host sends directly and no router is needed.
5. Each [router](../05-routing/01-how-routers-work.md) reads the destination address, looks in its table, and sends the packet to the next router. One step across a link to the next device is called a **hop**.
6. If a router is too busy, the packet waits in a memory queue. If the queue is full, the packet is dropped.
7. The destination host receives the packets and rebuilds the data. TCP puts them in order and asks for missing data again.

**Fuller truth:** the word "packet" is often used for any unit of data. Strictly, a TCP piece is a **segment**, an IP piece is a **packet**, and an Ethernet piece is a **frame**. See [Encapsulation and Decapsulation](../02-network-models/04-encapsulation.md).

### Sharing a network: circuit switching vs packet switching

**Switching** means moving data through a network from one link to the next. There are two main ways to share links among users.

#### Circuit switching

In **circuit switching**, the network reserves a fixed path and fixed capacity for one conversation, before any data flows. The classic telephone network works this way.

1. **Setup.** You dial. The network finds a path and reserves capacity on every link.
2. **Transfer.** You talk. The reserved path is only yours for the whole call.
3. **Teardown.** You hang up. The capacity is released.

```text
  Phone A ===== Exchange 1 ===== Exchange 2 ===== Phone B
          \_____________ reserved for this call __________/
```

A traditional digital voice channel uses 64 kbit/s (kilobits per second). This rate is reserved even when nobody is speaking.

- **Good:** steady quality, fixed delay, and no waiting in queues once connected.
- **Bad:** silence wastes capacity. If all circuits are busy, a new call is refused.

**Analogy:** a private rail track built for one train. **Where the analogy stops being true:** a circuit does not need its own wire. Many circuits share one cable by taking turns in fixed time slots. The turns are reserved even when unused.

#### Packet switching

In **packet switching**, nothing is reserved. Data is cut into packets, and each packet uses the link when it needs it. The internet works this way.

```text
  Host A  [A3][A2][A1] \
                        \
                         Router ==[B2][A2][B1][A1]==> Next router
                        /   (one shared link)
  Host B  [B2][B1]     /
```

Packets from A and B mix on the same link. When A is silent, B can use all of the link. Nothing is wasted.

Routers usually work by **store-and-forward**: a router receives the whole packet, checks it, then sends it onward.

- **Good:** very efficient use of links, and a failed link can be routed around.
- **Bad:** delay can change, and packets can be lost when links are busy. A change in delay from packet to packet is called **jitter**.

**Analogy:** cars on a shared highway. No one reserves the road. Most of the time this works well, but rush hour causes delays. **Where the analogy stops being true:** a car is never thrown away because the road is full. A packet can be.

The internet's core promise is called **best effort**. IP tries to deliver each packet, but it gives no guarantee about delivery, order, or timing. Protocols such as TCP add reliability on top. See [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md).

#### Side by side

| Point | Circuit switching | Packet switching |
|---|---|---|
| Path | Reserved before data flows | Decided as packets travel |
| Capacity | Fixed and dedicated | Shared on demand |
| Unused capacity | Wasted | Used by others |
| Setup step | Required | Not required at the IP layer |
| Delay | Steady once connected | Varies with load |
| When busy | New calls are refused | Packets wait or are dropped |
| When a link fails | Call drops | Traffic can be routed around |
| Fits best | Steady traffic, such as classic voice | Bursty traffic, such as web use |
| Classic example | Traditional telephone network | The internet |

**Bursty** traffic comes in short bursts with long gaps. Web browsing is bursty: you load a page, then read for a minute. That is why packet switching fits data so well.

**Fuller truth:** modern phone calls are moving to packets too. Voice over 4G (VoLTE) and voice over 5G are carried as packets, and many countries are retiring old circuit-switched phone equipment. Voice quality on packet networks is protected with [QoS](../10-enterprise-and-datacenter/03-quality-of-service.md) (Quality of Service, which gives voice packets priority).

## Example

### Example 1: See your place in the network

**Find your own address and gateway on Linux:**

```bash
$ ip -4 -br addr show wlan0
wlan0            UP             192.168.1.10/24

$ ip route show default
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
```

**The same on Windows:**

```powershell
PS> ipconfig

Wireless LAN adapter Wi-Fi:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 192.168.1.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

This host is `192.168.1.10`. Traffic for other networks goes to the router at `192.168.1.1`. The `/24` in the Linux output and the subnet mask `255.255.255.0` in the Windows output mean the same thing: they show which addresses are on the local network.

`ping` sends a small test message and waits for a reply. It shows if another host can be reached.

**Now test with ping (Linux):**

```bash
$ ping -c 4 example.com
PING example.com (198.51.100.80) 56(84) bytes of data.
64 bytes from 198.51.100.80: icmp_seq=1 ttl=58 time=23.1 ms
64 bytes from 198.51.100.80: icmp_seq=2 ttl=58 time=22.8 ms
64 bytes from 198.51.100.80: icmp_seq=3 ttl=58 time=24.2 ms
64 bytes from 198.51.100.80: icmp_seq=4 ttl=58 time=23.1 ms

--- example.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 22.800/23.300/24.200/0.534 ms
```

**The same on Windows:**

```powershell
PS> ping -n 4 example.com

Pinging example.com [198.51.100.80] with 32 bytes of data:
Reply from 198.51.100.80: bytes=32 time=23ms TTL=58
Reply from 198.51.100.80: bytes=32 time=23ms TTL=58
Reply from 198.51.100.80: bytes=32 time=24ms TTL=58
Reply from 198.51.100.80: bytes=32 time=23ms TTL=58

Ping statistics for 198.51.100.80:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 23ms, Maximum = 24ms, Average = 23ms
```

How to read this:

- **`time`** is the round trip: from you to the server and back, in milliseconds (ms). See [Bandwidth, Latency and Performance](06-network-performance-basics.md).
- **`ttl=58`** is the hop counter left in the reply. Each router lowers it by 1. Many Linux servers start at 64, so 58 suggests the reply crossed 6 routers. Starting values vary by operating system. Windows often starts at 128.
- **0% packet loss** means every packet came back.

The address `198.51.100.80` is a documentation example. The real example.com uses a different address.

### Example 2: Become a server in one minute

This shows that "server" is a role. Any computer can do it.

**On the laptop (192.168.1.10), start a small web server:**

```bash
# Linux. On Windows, use: python -m http.server 8000
$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

**On a second computer (192.168.1.11), act as the client:**

```bash
# Linux. On Windows, use curl.exe in place of curl.
$ curl -I http://192.168.1.10:8000/
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Date: Fri, 30 Oct 2026 10:15:02 GMT
Content-type: text/html; charset=utf-8
Content-Length: 412
```

**Back on the laptop, the server logs the request:**

```text
192.168.1.11 - - [30/Oct/2026 10:15:02] "HEAD / HTTP/1.1" 200 -
```

Your ordinary laptop just acted as a server. It shares the folder you started it in, so start it in an empty folder. Press Ctrl+C to stop it. If the connection fails, a firewall (software that blocks unwanted traffic) on the laptop is the likely cause. Run this only on a network you trust.

### Example 3: Cut a file into packets

You send a 1 MB file (here, 1,000,000 bytes) using TCP over IPv4 on Ethernet.

| Item | Value |
|---|---|
| Largest IP packet (MTU) | 1,500 bytes |
| IPv4 header | 20 bytes (minimum) |
| TCP header | 20 bytes (minimum) |
| File data per packet | 1,500 - 20 - 20 = 1,460 bytes |

**Step 1, count the packets:**

```text
1,000,000 / 1,460 = 684.9...  so 685 packets are needed
```

**Step 2, check the last packet:**

```text
684 full packets x 1,460 bytes = 998,640 bytes
Remaining data = 1,000,000 - 998,640 = 1,360 bytes (packet 685)
```

**Step 3, count the header cost:**

```text
685 packets x 40 header bytes = 27,400 bytes
27,400 / 1,000,000 = 2.74% extra
```

**Step 4, see what a loss costs.** If packet 300 is lost, TCP sends again only the missing data, about 1,460 bytes. It does not need to send the whole 1 MB again. Without packets, one error could force a repeat of the entire file.

**Fuller truth:** real traffic differs a little. Common TCP options take extra header space, so data per packet is often 1,448 bytes on Linux. Each Ethernet frame also adds its own overhead. The idea stays the same.

### Example 4: Watch data cross the internet

Traceroute lists the routers on the path. Each line is one hop.

**Linux:**

```bash
$ traceroute -n example.com
traceroute to example.com (198.51.100.80), 30 hops max, 60 byte packets
 1  192.168.1.1     1.214 ms   1.102 ms   1.067 ms
 2  203.0.113.1     7.482 ms   7.311 ms   7.650 ms
 3  203.0.113.129   8.905 ms   8.772 ms   9.041 ms
 4  192.0.2.17     14.318 ms  14.206 ms  14.590 ms
 5  192.0.2.146    19.884 ms  19.702 ms  20.115 ms
 6  198.51.100.1   22.436 ms  22.310 ms  22.671 ms
 7  198.51.100.80  23.104 ms  22.958 ms  23.377 ms
```

**Windows:**

```powershell
PS> tracert -d example.com

Tracing route to example.com [198.51.100.80]
over a maximum of 30 hops:

  1     1 ms     1 ms     1 ms  192.168.1.1
  2     7 ms     7 ms     8 ms  203.0.113.1
  3     9 ms     9 ms     9 ms  203.0.113.129
  4    14 ms    14 ms    15 ms  192.0.2.17
  5    20 ms    20 ms    20 ms  192.0.2.146
  6    22 ms    22 ms    23 ms  198.51.100.1
  7    23 ms    23 ms    23 ms  198.51.100.80

Trace complete.
```

Now match each hop to the ISP diagram above. In this example, the address blocks belong to these networks:

| Hop | Address | Network | Role |
|---|---|---|---|
| 1 | 192.168.1.1 | Your home | Home router, your default gateway |
| 2 | 203.0.113.1 | AS64540, 203.0.113.0/24 | Access ISP, first router |
| 3 | 203.0.113.129 | AS64540, 203.0.113.0/24 | Access ISP, core router |
| 4 | 192.0.2.17 | AS64520, 192.0.2.0/25 | Tier 2 network |
| 5 | 192.0.2.146 | AS64530, 192.0.2.128/25 | Second Tier 2 network, reached by peering |
| 6 | 198.51.100.1 | AS64550, 198.51.100.0/24 | Hosting network's router |
| 7 | 198.51.100.80 | AS64550, 198.51.100.0/24 | The web server, a host |

```text
  Home        Access ISP      Tier 2       Tier 2       Hosting
  hop 1        hops 2-3       hop 4        hop 5        hops 6-7
 +------+    +----------+    +------+     +------+    +----------+
 | LAN  |--->| AS64540  |--->|64520 |---->|64530 |--->| AS64550  |
 +------+    +----------+    +------+     +------+    +----------+
```

What this teaches:

- There are six routers and one server. This agrees with `ttl=58` from Example 1.
- Your packets crossed five separate networks.
- The path never touched a Tier 1 network, because the two Tier 2 networks peer directly.
- Your home address `192.168.1.10` is private, so it does not appear on the internet. Your home router replaces it with the public address `203.0.113.25` using [NAT](../04-ip-addressing/09-nat-and-pat.md).

More detail on these tools is in [ICMP, Ping and Traceroute](../05-routing/10-icmp-ping-and-traceroute.md) and [Using Ping, Traceroute and MTR](../12-troubleshooting-and-tools/03-ping-traceroute-and-mtr.md).

## Going Deeper

This section is for readers who want more than the basics. You can return to it later.

### Statistical multiplexing: the number behind packet switching

**Multiplexing** means sharing one link among many users. **Statistical multiplexing** means sharing it on demand, based on the fact that users are rarely all active at once.

Take a 100 Mbps (megabits per second) link. Each user needs 10 Mbps when active, and is active 10% of the time.

- **Circuit switching:** you must reserve 10 Mbps per user, so only 100 / 10 = **10 users** fit.
- **Packet switching:** suppose you allow **35 users**. The average total load is 35 x 10% x 10 Mbps = 35 Mbps, well under the limit. There is trouble only when more than 10 users are active at the same moment. With independent users, that probability is about 0.0004, or 0.04% of the time.

So packet switching serves 3.5 times as many users, with almost no congestion.

**Limit of this calculation:** it assumes that users act independently. Real traffic is not always like that. If everyone streams the same football final, demand rises together. That is why operators keep spare capacity, and why [TCP congestion control](../06-transport-layer/06-tcp-congestion-control.md) exists.

### Store-and-forward delay

A store-and-forward router must receive the whole packet before it sends it on. The time to put a packet on a link is:

```text
Transmission time = packet size in bits / link rate in bits per second
```

For a 1,500-byte packet on a 100 Mbps link:

```text
1,500 x 8 = 12,000 bits
12,000 / 100,000,000 = 0.00012 s = 120 microseconds
```

A microsecond is one millionth of a second. Across three such links, this adds 3 x 120 = 360 microseconds. Queueing and the travel time of the signal come on top. Smaller packets reduce this delay per hop. This delay is one reason for splitting data into packets instead of sending one huge block. Some data centre switches use cut-through forwarding, where they start sending a frame before all of it has arrived, to avoid it. See [How Switches Work](../03-ethernet-and-switching/03-how-switches-work.md).

### Message switching

An older third method is **message switching**. Each whole message is stored and forwarded from node to node, as in old telegraph relay systems. It needs no reserved path, but a long message holds a link for a long time, and an error can mean a full resend. Packet switching keeps the store-and-forward idea and fixes this problem by limiting the size of each unit.

### Datagrams vs virtual circuits

Packet switching itself has two forms. A **datagram** is a packet that carries its full destination and is handled on its own. A **virtual circuit** is a path set up in advance that a group of packets then follows.

| Point | Datagram | Virtual circuit |
|---|---|---|
| Setup before data | No | Yes, a path is set up |
| Per-path state in routers | No | Yes |
| What each packet carries | Full destination address | A short label or circuit number |
| Path | Each packet is decided independently | All packets follow the set-up path |
| Examples | IP | MPLS, older X.25, Frame Relay, ATM (Asynchronous Transfer Mode) |

A virtual circuit is not a true circuit. Packets still share links and can still queue. See [MPLS](../05-routing/13-mpls.md).

**A TCP connection is not a circuit.** Only the two end hosts keep connection state. Routers in the middle do not reserve anything for it and normally do not know it exists.

**Fuller truth on paths:** you will often hear that packets of one connection take different paths. That can happen, but routers try to avoid it. A **flow** is the set of packets between the same two programs. With load sharing across equal paths, routers usually run a quick calculation (a hash) on the addresses and ports, which keeps each flow on one path so that packets are not reordered. Paths mainly change when routing changes. Stateful devices such as firewalls and NAT also need traffic to stay consistent.

Also, packet networks often run on top of circuit-like layers. Long-distance fibre commonly carries fixed optical wavelengths between routers. To IP, each wavelength just looks like a link.

### The end-to-end principle

The internet keeps the core simple and puts the smart work in the hosts. Routers forward packets at best effort. Hosts handle reliability, ordering, and encryption. This idea was described in a well-known 1984 paper by Saltzer, Reed, and Clark.

This design lets new applications appear without changing the core. HTTP and QUIC (a newer transport protocol) needed no permission from routers. In practice, the principle is bent by devices in the middle, such as NAT, firewalls, and load balancers (devices that spread requests across many servers).

### The flattening of the internet

The neat tier pyramid is now only part of the picture. Large content companies run their own global networks. They peer directly with access ISPs at IXPs. Some place cache servers (machines that keep copies of popular content) inside ISP networks, such as Netflix Open Connect and Google Global Cache. As a result, a large share of traffic goes from a nearby content network to the user without crossing a Tier 1 network. See [Content Delivery Networks](../10-enterprise-and-datacenter/06-cdn.md).

The difference between transit and peering is also important in routing. Transit gives you reach to the whole internet. Peering only gives you reach to that peer and its customers. If AS64540 peers with AS64541, it can reach AS64541's customers for free, but it cannot use AS64541 to reach the rest of the world. These rules are expressed through [BGP](../05-routing/09-bgp.md).

### Small i and capital I

In older writing, an "internet" with a small i meant any set of connected networks, and the "Internet" with a capital I meant the global one. Today many style guides use lowercase for the global one too. An **intranet** is a private network within an organisation that uses internet technology. See [Types of Networks](02-network-types.md).

## Common Mistakes

- **"The internet and the web are the same."** The web is one service. The internet also carries email, DNS, SSH, games, and more.
- **"A server is a special kind of computer."** A server is a role. Any device that answers requests can be one. Data centre hardware is simply built for the job.
- **"A client is always a person's device."** A web server becomes a client when it calls a database.
- **"Wi-Fi is the internet."** Wi-Fi is only a local radio link to your router. You can have strong Wi-Fi and no internet.
- **"The internet is mostly wireless or in the sky."** Most long-distance traffic travels through fibre, including cables on the sea floor.
- **"There is one internet backbone."** There are many backbones owned by many companies, joined by peering and transit.
- **"Tier 1 means the fastest or best."** The term describes who pays whom. A Tier 2 network can give better local performance.
- **"Someone owns or controls the whole internet."** Many organisations run parts of it and coordinate it. Nobody owns all of it.
- **"Every packet is 1,500 bytes."** 1,500 is a common maximum on Ethernet. Many packets are smaller, such as a TCP acknowledgement (a short "I got it" message). Some networks use larger jumbo frames (Ethernet frames above the normal size), and the size varies by network.
- **"Packets of one flow always take different routes."** They can, but routers usually keep one flow on one path.
- **"Packet loss always means something is broken."** Routers drop packets when queues are full, and TCP usually treats this as a sign to slow down.
- **"Packet switching is always better."** It is usually the better fit for data. Fixed delay and guaranteed capacity are where circuits are stronger.
- **"A TCP connection reserves a path."** It does not. State exists only at the two ends.
- **"The internet was built to survive nuclear war."** Paul Baran's work at RAND, a US research organisation, did study survivable networks, but ARPANET's main purpose was sharing computing resources among researchers.

## Interview Questions

**Q: What is a computer network?**

A: A network is two or more devices connected by links so they can exchange data and share resources. It needs nodes, links, addresses to identify devices, and protocols, which are the rules they follow to communicate.

**Q: What is the difference between a node and a host?**

A: A node is any device on the network that can send, receive, or forward data. A host is an end device that runs applications, such as a laptop or server. A router is a node, but it is normally not a host, because its job is forwarding traffic for others. In daily use, people sometimes say host for anything with an IP address.

**Q: Is a server a piece of hardware?**

A: Not necessarily. Client and server are roles. A server is a program that listens for requests and answers them, and the word also describes the machine that runs it. A laptop can run a web server, and a web server can act as a client to a database.

**Q: Compare client-server and peer-to-peer. When would you use each?**

A: In client-server, central servers provide the service, so control, security, and data consistency are easier, but the owner pays for capacity. In peer-to-peer, devices act as both client and server, so capacity grows with users and there is no single central failure point, but security and finding data are harder. I would use client-server for banking or normal web apps, and P2P for large file distribution. Many real systems are hybrids.

**Q: What is the internet, and who owns it?**

A: The internet is a global network of independent networks that communicate using the TCP/IP protocols. Nobody owns all of it. ISPs and other companies own the cables and routers, while groups such as the IETF write standards, and ICANN, IANA, and the regional registries coordinate names and numbers.

**Q: What is the difference between the internet and the World Wide Web?**

A: The internet is the global network infrastructure. The web is a service that runs on it, made of pages and resources reached over HTTP. Email and SSH use the internet without being part of the web.

**Q: What is an ISP, and what are Tier 1, 2, and 3?**

A: An ISP is a company that connects customers to the internet. A Tier 1 network can reach the whole internet through peering alone, without paying for transit. A Tier 2 network peers with some networks and buys transit for the rest. A Tier 3 network mostly buys transit and sells access to end users. These terms are industry language, not an official standard.

**Q: What is the difference between transit and peering?**

A: Transit is a paid service in which a provider carries your traffic to the entire internet. Peering is an agreement in which two networks exchange traffic directly, only between themselves and their customers, often without payment. Peering is often done at an internet exchange point.

**Q: What is a packet, and why do we split data into packets?**

A: A packet is a small unit of data with a header, which holds delivery details such as source and destination addresses, and a payload, which holds the data. We split data so that many users can share links fairly, so that only lost pieces need to be sent again, and so that traffic can be routed around failures.

**Q: Explain circuit switching and packet switching.**

A: Circuit switching reserves a fixed path and capacity before communication starts, like a classic phone call. Quality is steady, but unused capacity is wasted. Packet switching reserves nothing. Data is split into packets that share links on demand, like the internet. It uses links far more efficiently, but delay can vary and packets can be lost.

**Q: Why does the internet use packet switching rather than circuit switching?**

A: Internet traffic is bursty. Users send a little, then stay quiet. Reserving capacity for each user would waste most of it. Statistical multiplexing lets many more users share the same link, and there is no call setup for each short exchange. Routers can also send traffic around a failed link.

**Q: Does a TCP connection create a circuit?**

A: No. TCP is connection-oriented only at the two end hosts, which keep state such as sequence numbers. Routers in the path reserve no capacity and normally keep no state about the connection. The packets still travel through a shared, best-effort IP network.

## Quick Recap

- A network is devices (nodes) joined by links, using addresses and protocols to exchange data.
- Every host is a node, but routers are nodes that normally are not hosts.
- Client and server are roles, not hardware. In peer-to-peer, each device plays both roles.
- The internet is a network of networks using TCP/IP. Nobody owns all of it, and the web is only one service on it.
- ISPs connect through paid transit and direct peering. Tier 1 networks reach everyone without paying for transit.
- A packet is a header plus a payload. Packets allow fair sharing and easy recovery from loss.
- Circuit switching reserves capacity and gives steady quality. Packet switching shares capacity on demand and fits bursty traffic, so the internet uses it.
