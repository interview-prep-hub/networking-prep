# Real-Time and Modern Protocols

**Level:** Intermediate | **Prerequisites:** [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md), [HTTP Basics](05-http-basics.md), [HTTP/1.1, HTTP/2 and HTTP/3](06-http-versions.md), [TLS and HTTPS](07-tls-and-https.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md)

## In One Line

Plain HTTP is "ask once, get one answer", so these protocols add what it lacks: server push (WebSocket, SSE), fast service calls (gRPC), live voice and video (WebRTC, SIP, RTP), and light messaging for devices and back-end systems (MQTT, AMQP, CoAP).

## Why It Matters

Classic HTTP has one rule: the client asks and the server answers. The server cannot speak first. That is a poor fit for a chat app, a live score, a video call, or 50,000 sensors with weak batteries.

Each protocol here solves one part of that problem. Interviewers ask about them for three reasons:

- **System design:** "Design a chat app" or "design live notifications" needs a choice between WebSocket, SSE and polling, with reasons.
- **Troubleshooting:** "The call connects but only one person hears audio" is a classic VoIP and NAT question.
- **Trade-offs:** They want to hear why voice uses UDP, why gRPC needs layer 7 load balancing, and why MQTT has three QoS levels.

## Simple Explanation

Think of different ways to communicate with a friend.

| Real life | Protocol | Idea |
|---|---|---|
| You walk to the shop every 5 minutes and ask "is my parcel here?" | Short polling | Many wasted trips |
| You wait at the counter until the parcel arrives | Long polling | One trip, long wait |
| You subscribe to a radio station | SSE | Server talks, you listen |
| You make a phone call | WebSocket | Both talk any time |
| You use a fast internal office phone with a fixed script | gRPC | Quick, typed calls between services |
| You speak directly to your friend after a mutual contact gives you their number | WebRTC | Peers connect directly, helped by a middleman at the start |
| A telephone exchange rings the other phone, then your voices flow | SIP and RTP | Setup and media are separate |
| A notice board where anyone can post and anyone can follow a topic | MQTT | Publish and subscribe via a broker |
| A post office with sorting rules and delivery receipts | AMQP | Reliable routing into queues |
| A very short text message that fits a tiny device | CoAP | Small web-like requests over UDP |

Where the analogies stop: a radio does not resend what you missed, but SSE can resume from the last event ID. A phone call has a fixed path, but WebSocket frames are normal TCP segments that may take any route.

## How It Works

### The polling family: three ways to fake or provide push

**Short polling** means the client sends a normal request on a timer. It is simple, and most replies are empty.

**Long polling** means the client sends a request and the server holds it open until data exists or a timeout passes. The client then asks again at once. RFC 6202 describes this method and its known problems.

**Streaming** means the server keeps one response open and keeps writing to it. SSE is a standard form of this.

```text
Short polling  C --GET--> S   reply is empty, ask again in 5 s
               C <--204-- S

Long polling   C --GET--> S   server holds it, up to 30 s
               C <--200-- S   data arrives, client asks again

SSE            C --GET--> S   one response that never ends
               C <--event-- <--event-- <--event--

WebSocket      C --Upgrade--> S   then frames in both directions
               C <==========> S
```

### WebSocket upgrade

**WebSocket** (RFC 6455) gives one long-lived, two-way connection over TCP. TCP is the reliable, ordered transport under it. A **frame** is one small unit of WebSocket data.

It starts as HTTP so it can use ports 80 and 443 and pass through most firewalls. `ws://` is plain and `wss://` is WebSocket inside TLS.

1. The client opens a TCP connection, and a TLS session for `wss://`.
2. The client sends an HTTP/1.1 GET with upgrade headers.
3. The server checks them and replies `101 Switching Protocols`.
4. HTTP stops on this connection. Both sides now send WebSocket frames.
5. Either side ends the connection with a Close frame.

```text
GET /chat HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://www.example.com

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**The key and accept check.** The client sends a random 16-byte value, base64 encoded. The server appends a fixed GUID to it (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`), takes the SHA-1 hash, and base64 encodes the result. This proves the server understands WebSocket. It is not authentication or encryption.

**Frame layout** (RFC 6455):

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R|opcode |M| Payload len |  Extended length (16 or 64    |
|I|S|S|S|  (4)  |A|     (7)     |  bits, only if needed)        |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|  Masking key (32 bits, only on client-to-server frames)       |
+---------------------------------------------------------------+
|  Payload data                                                 |
+---------------------------------------------------------------+
```

| Opcode | Meaning |
|---|---|
| 0x0 | Continuation of an earlier frame |
| 0x1 | Text (UTF-8) |
| 0x2 | Binary |
| 0x8 | Close |
| 0x9 | Ping |
| 0xA | Pong |

**Masking.** Every client-to-server frame must be XORed with a random 4-byte key. This is not privacy, because the key travels in the same frame. It stops attackers from making browser traffic look like HTTP to old, broken proxies, which could poison their caches. Server frames are not masked.

### Server-Sent Events (SSE)

**SSE** is a one-way stream from server to browser. It is defined in the WHATWG HTML standard, not in an RFC. It is plain HTTP, so no upgrade is needed.

1. The browser creates `new EventSource("/stream")`, which sends a GET.
2. The server replies with `Content-Type: text/event-stream` and never closes the body.
3. The server writes text events, each ended by a blank line.
4. If the connection breaks, the browser reconnects by itself. It sends the `Last-Event-ID` header so the server can resume.

| Field | Meaning |
|---|---|
| `data:` | The message text. Several lines are allowed |
| `event:` | A name for the event type |
| `id:` | An ID the browser remembers for resuming |
| `retry:` | Reconnect delay, in milliseconds |
| `:` | A line starting with a colon is a comment, often a heartbeat |

Limits: SSE carries UTF-8 text only, not binary. On HTTP/1.1, browsers allow about 6 connections per host (a browser habit, not a protocol rule), so many tabs can run out. HTTP/2 solves this because many streams share one connection.

### Comparing polling, SSE and WebSocket

| Feature | Short poll | Long poll | SSE | WebSocket |
|---|---|---|---|---|
| Direction | Client asks | Client asks | Server to client | Both ways |
| Delay | Up to one interval | Low | Low | Lowest |
| Runs on | HTTP | HTTP | HTTP | Own framing after HTTP upgrade |
| Auto reconnect | Not needed | You write it | Built in | You write it |
| Binary data | Yes | Yes | No | Yes |
| Overhead per message | Full HTTP headers | Full HTTP headers | A few bytes | 2-14 bytes |
| Works with plain HTTP tools | Yes | Yes | Yes, if buffering is off | Needs upgrade support |

### gRPC on HTTP/2

**gRPC** is a framework for one program to call a function in another program over the network. This style is called **RPC** (remote procedure call). Google created gRPC, and it is now a CNCF open source project.

It has two key parts:

- **Protocol Buffers (protobuf):** a compact binary data format. You describe messages and services in a `.proto` file, and tools generate client and server code.
- **HTTP/2:** the transport. gRPC uses its streams, its flow control and its trailers. **Trailers** are headers sent after the body.

```text
syntax = "proto3";
package shop;

service Cart {
  rpc AddItem (AddItemRequest) returns (AddItemReply);  // unary
  rpc WatchCart (CartId) returns (stream CartEvent);  // server stream
  rpc Upload (stream Chunk) returns (UploadReply);  // client stream
  rpc Chat (stream ChatMsg) returns (stream ChatMsg);  // both
}
```

**One call on the wire:**

```text
Client                                              Server
  |--- HEADERS  :method POST                          |
  |             :path /shop.Cart/AddItem              |
  |             content-type: application/grpc        |
  |             te: trailers                          |
  |             grpc-timeout: 1S                      |
  |--- DATA     [flag 1 byte][length 4 bytes][proto]->|
  |                                                   |
  |<-- HEADERS  :status 200                           |
  |<-- DATA     [flag][length][proto]                 |
  |<-- HEADERS  grpc-status: 0  (trailers, end)       |
```

Each message has a 5-byte prefix: 1 byte says if it is compressed, and 4 bytes give the length. The real result is in the `grpc-status` trailer. `0` means OK. Others include `4` DEADLINE_EXCEEDED, `5` NOT_FOUND, `13` INTERNAL, `14` UNAVAILABLE and `16` UNAUTHENTICATED.

**Why not in a browser directly?** Browser JavaScript cannot control HTTP/2 frames or read trailers in the way gRPC needs. **gRPC-Web** is a variant that goes through a proxy such as Envoy, and it supports only unary and server streaming in common use.

There is no official port. 50051 is a common convention from the examples. In production it often runs on 443 with TLS.

### WebRTC

**WebRTC** lets two browsers or apps send audio, video and data directly to each other. It is a group of protocols, plus a W3C browser API. It became a W3C Recommendation in 2021, and the IETF overview is RFC 8825.

The hard part is finding a path when both devices sit behind **NAT** (a router that hides private addresses behind a public one).

| Part | Job | Standard |
|---|---|---|
| Signalling | Exchange setup messages. Not defined by WebRTC. You choose it, often WebSocket | Your choice |
| SDP | Text that describes codecs, keys and addresses | RFC 8866 |
| STUN | "What is my public IP and port?" | RFC 8489 |
| TURN | A relay server for when no direct path works | RFC 8656 |
| ICE | Tries all address pairs and picks the best that works | RFC 8445 |
| DTLS | TLS for UDP. Used here to agree on keys | RFC 6347 (1.2), RFC 9147 (1.3) |
| SRTP | Encrypted RTP for audio and video | RFC 3711 |
| SCTP | Carries data channels, over DTLS | RFC 8831 |

```text
           +------------------------------------+
           | Signalling server (your own)       |
           | often WebSocket on 443             |
           +---------+----------------+---------+
                     |                |
           offer and |                | answer and
           candidates|                | candidates
                     |                |
          +----------+-+            +-+----------+
          | Browser A  |<==========>| Browser B  |
          | 10.0.1.20  | SRTP, SCTP | 10.0.9.30  |
          +---+----+---+ over DTLS  +---+----+---+
              |    |     over UDP       |    |
              |    |                    |    |
              |    +--- STUN server ----+    |
              |        203.0.113.50          |
              +-------- TURN relay ----------+
                       198.51.100.20
```

**Call setup, step by step:**

1. A captures camera and microphone, then creates an SDP **offer**.
2. A sends the offer to B through the signalling server.
3. B creates an SDP **answer** and sends it back the same way.
4. Each side gathers **ICE candidates** (possible addresses). It asks STUN for its public address and asks TURN for a relay address.
5. The sides exchange candidates, often one at a time as found. This is called **trickle ICE**.
6. ICE sends STUN checks on every candidate pair and picks the best one that works.
7. The peers run a DTLS handshake on that path. Each checks the other's certificate against the `a=fingerprint` value in the SDP.
8. Keys from DTLS start SRTP for media. Data channels run as SCTP over the same DTLS session.

| Candidate type | Meaning | Example |
|---|---|---|
| `host` | Local interface address | 10.0.1.20 |
| `srflx` | Server reflexive: public address seen by STUN | 203.0.113.7 |
| `relay` | Address on a TURN server | 198.51.100.20 |

Analogy: STUN is a friend who tells you your own street address. TURN is a friend who carries every letter between you and the other person. Where it stops: a street address is fixed, but a NAT mapping expires and depends on the destination.

### SIP and RTP/RTCP for VoIP

**VoIP** (Voice over IP) carries phone calls on IP networks. It splits the job in two:

- **SIP** (RFC 3261) sets up, changes and ends the call. Port 5060 on UDP or TCP, 5061 for TLS.
- **RTP** (RFC 3550) carries the voice. **RTCP**, in the same RFC, reports on the quality.
- **SDP** inside SIP describes the media: codec, IP address and port.

**SIP roles:**

| Role | Job |
|---|---|
| User agent | A phone or softphone. Makes and takes calls |
| Registrar | Stores where each user is right now |
| Proxy | Forwards SIP messages toward the user |
| B2BUA | Back-to-back user agent. Ends one call leg and starts another, as most PBXs do |
| SBC | Session border controller. Guards the network edge, fixes NAT, hides topology |

SIP looks like HTTP: text requests and numbered responses.

| Group | Examples |
|---|---|
| Methods | `REGISTER`, `INVITE`, `ACK`, `BYE`, `CANCEL`, `OPTIONS` |
| 1xx | `100 Trying`, `180 Ringing`, `183 Session Progress` |
| 2xx | `200 OK` |
| 3xx | `302 Moved Temporarily` |
| 4xx | `401 Unauthorized`, `404 Not Found`, `486 Busy Here`, `487 Request Terminated` |
| 5xx | `503 Service Unavailable` |
| 6xx | `603 Decline` |

**A basic call:**

```text
Alice                 SIP proxy                  Bob
  |--- INVITE (SDP) ------>|                       |
  |<-- 100 Trying ---------|                       |
  |                        |--- INVITE (SDP) ----->|
  |                        |<-- 180 Ringing -------|
  |<-- 180 Ringing --------|                       |
  |                        |<-- 200 OK (SDP) ------|
  |<-- 200 OK (SDP) -------|                       |
  |--- ACK --------------------------------------->|
  |<========= RTP audio, both ways, direct =======>|
  |--- BYE --------------------------------------->|
  |<-- 200 OK -------------------------------------|
```

Key point: SIP goes through the proxy, but media usually flows directly between the endpoints. Fuller truth: ACK and BYE also go through the proxy if it adds a `Record-Route` header, and a B2BUA or SBC often stays in the media path too.

Analogy: SIP is the person at the hotel desk who connects your call. RTP is the talking. Where it stops: a hotel operator hears nothing after connecting you, but a PBX can record or relay the media.

**RTP header** (12 bytes, plus optional items):

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---+-+-+-------+-+-------------+-------------------------------+
| V |P|X|  CC   |M|     PT      |        Sequence number        |
+---+-+-+-------+-+-------------+-------------------------------+
|                           Timestamp                           |
+---------------------------------------------------------------+
|                             SSRC                              |
+---------------------------------------------------------------+
|                  CSRC list (0 to 15 items)                    |
+---------------------------------------------------------------+
```

| Field | Meaning |
|---|---|
| V | Version, always 2 |
| P, X | Padding present, header extension present |
| CC | Number of CSRC items |
| M | Marker, meaning depends on the media |
| PT | Payload type: the codec. 0 is PCMU, 8 is PCMA, 96-127 are dynamic |
| Sequence number | Adds 1 per packet, so the receiver finds loss and reordering |
| Timestamp | Sampling time, so the receiver can play at the right pace |
| SSRC | Random ID of this media source |
| CSRC | Sources mixed into this stream |

RTP does not resend, does not reorder and gives no quality guarantee. It only labels packets so the receiver can act.

**RTCP** sends small reports next to RTP. Traditional setups use an even port for RTP and the next odd port for RTCP, such as 16384 and 16385. WebRTC usually puts both on one port.

| RTCP packet | Purpose |
|---|---|
| SR (sender report) | Packets sent, and a time reference |
| RR (receiver report) | Loss, jitter, and data for round-trip time |
| SDES | Names the source |
| BYE | Source is leaving |

RFC 3550 suggests keeping RTCP to about 5 percent of session bandwidth, with a minimum report interval of 5 seconds in the basic rule.

**Why voice uses UDP.** A retransmitted packet comes too late to play, and TCP would stall all later audio while it waits. A tiny gap in sound is better than a growing delay. See [UDP in Depth](../06-transport-layer/07-udp.md).

### MQTT and its QoS levels

**MQTT** is a tiny publish and subscribe protocol for sensors and phones. **Publish and subscribe** means senders do not know receivers. They talk only to a **broker**, the central server that passes messages on.

MQTT runs on TCP. Port 1883 is plain and 8883 is TLS. OASIS publishes the standard. Version 3.1.1 is also ISO/IEC 20922, and version 5.0 came in 2019. Popular brokers include Mosquitto, EMQX and HiveMQ.

```text
                       +-----------------+
 sensor1 --PUBLISH---> |  MQTT broker    | --PUBLISH--> dashboard
 topic:                |  192.168.1.50   |              subscribed to
 home/kitchen/temp     |  port 1883      | --PUBLISH--> home/+/temp
                       +-----------------+              logger
                                                        subscribed to
                                                        home/#
```

**Topics** are text paths with `/` levels:

- `+` matches exactly one level. `home/+/temp` matches `home/kitchen/temp`.
- `#` matches all the remaining levels. `home/#` matches everything under `home`.

**QoS** (Quality of Service) here means how hard MQTT tries to deliver one message. It has nothing to do with network QoS markings.

| QoS | Name | Packets | Loss? | Duplicates? |
|---|---|---|---|---|
| 0 | At most once | `PUBLISH` | Possible | No |
| 1 | At least once | `PUBLISH`, `PUBACK` | No | Possible |
| 2 | Exactly once | `PUBLISH`, `PUBREC`, `PUBREL`, `PUBCOMP` | No | No |

```text
QoS 0   Sender --PUBLISH-->  Receiver   (fire and forget)

QoS 1   Sender --PUBLISH-->  Receiver   stores, then delivers
        Sender <--PUBACK---  Receiver
        No PUBACK? Sender resends with DUP=1, so a copy can appear.

QoS 2   Sender --PUBLISH-->  Receiver   stores the packet ID
        Sender <--PUBREC---  Receiver   "I have it"
        Sender --PUBREL-->   Receiver   "you may release it"
        Sender <--PUBCOMP--  Receiver   "finished, forget the ID"
```

Analogy: QoS 0 is a postcard. QoS 1 is a courier who returns until someone signs, so you might get two copies. QoS 2 adds a second confirmation so the parcel is counted once. Where it stops: QoS applies per hop, not end to end. See the rules below.

Rules that matter:

- **Two hops:** Publisher to broker and broker to subscriber are separate. The subscriber gets the lower of the publish QoS and the subscribe QoS.
- **Why QoS if TCP is reliable?** TCP only protects bytes on a live connection. If the connection dies, data in flight can be lost. QoS lets a client resend after reconnecting.
- **Offline delivery:** The broker queues QoS 1 and 2 messages for a subscriber only with a persistent session (Clean Session 0 in 3.1.1, or Clean Start 0 with a session expiry above 0 in 5.0).

Other key features:

| Feature | Meaning |
|---|---|
| Retained message | The broker keeps the last message per topic and gives it to new subscribers |
| Last Will | A message the broker publishes if a client vanishes without saying goodbye |
| Keep alive | Client sends `PINGREQ` when idle. The broker may drop it after 1.5 times the interval |
| Small headers | The fixed header can be 2 bytes. `PINGREQ` is exactly 2 bytes |

### AMQP

**AMQP** (Advanced Message Queuing Protocol) moves business messages reliably between back-end systems. It runs on TCP. Port 5672 is plain and 5671 is TLS.

There are two very different protocols with this name:

| Version | Model | Used by |
|---|---|---|
| AMQP 0-9-1 | Exchanges, queues, bindings are part of the protocol | RabbitMQ, its classic protocol |
| AMQP 1.0 | Wire protocol between peers, with no built-in broker model. OASIS and ISO/IEC 19464 | Azure Service Bus, ActiveMQ, Apache Qpid. Also RabbitMQ, where it became a core protocol in 4.0 |

The 0-9-1 model is what interviewers usually mean:

```text
Publisher --> Exchange --binding--> Queue --> Consumer
              (routes)              (stores)
```

```text
                               key "order.created"
              +----------+ ----------------> [billing queue] --> app A
Publisher --> | Exchange |
key:          |  (topic) | ----------------> [audit queue]   --> app B
order.created +----------+     key "order.#"
```

| Exchange type | Routing rule |
|---|---|
| direct | Routing key must match the binding key exactly |
| fanout | Copy to all bound queues, ignore the key |
| topic | Pattern match. `*` is one word, `#` is zero or more words |
| headers | Match on message header values |

Reliability has three parts. All three are needed:

1. **Publisher confirms:** the broker tells the publisher it took the message. This is a RabbitMQ extension to 0-9-1.
2. **Durable queues and persistent messages:** they survive a broker restart.
3. **Consumer acks:** the broker deletes a message only after the consumer confirms it. If the consumer dies, the message goes to another consumer.

One TCP connection carries many **channels** (light logical sessions), so an app does not need many sockets. **Prefetch** limits how many unacked messages a consumer holds, which spreads the load.

Analogy: the exchange is a mail sorter and the queue is a mailbox. Where it stops: a sorter puts one letter in one box, but one message can be copied into many queues.

### CoAP

**CoAP** (Constrained Application Protocol, RFC 7252) is a small version of HTTP for tiny devices. It has the same ideas: `GET`, `POST`, `PUT`, `DELETE`, URIs and response codes. It runs on UDP. Port 5683 is plain and 5684 is DTLS.

**Constrained** means little memory, a small battery and a weak radio link. See [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---+---+-------+---------------+-------------------------------+
|Ver| T |  TKL  |     Code      |          Message ID           |
+---+---+-------+---------------+-------------------------------+
|  Token (0 to 8 bytes, length given by TKL)                    |
+---------------------------------------------------------------+
|  Options (such as Uri-Path)                                   |
+---------------------------------------------------------------+
|  0xFF marker, then payload (only if there is a payload)       |
+---------------------------------------------------------------+
```

| Message type | Meaning |
|---|---|
| CON | Confirmable. The sender expects an ACK and resends if none comes |
| NON | Non-confirmable. No ACK |
| ACK | Answers a CON. Can carry the response in the same packet |
| RST | Reset. "I cannot process this message" |

Reliability is built into CoAP since UDP has none. The first resend waits 2-3 seconds. The wait doubles each time, for up to 4 resends (defaults from RFC 7252).

Response codes look like `2.05 Content`, `2.01 Created`, `4.04 Not Found` and `5.00 Internal Server Error`. The first digit works like the HTTP class: 2 is success, 4 is client error, 5 is server error.

Useful extensions:

- **Observe (RFC 7641):** a client registers, and the server sends each change. It works like a light subscription.
- **Block-wise transfer (RFC 7959):** splits large bodies across many small datagrams.
- **Discovery (RFC 6690):** `GET /.well-known/core` lists the device's resources.
- **Multicast:** one request can reach many devices, using 224.0.1.187 in IPv4.

### When to use which

| Need | Use | Why |
|---|---|---|
| Live feed to browser, server to client only | SSE | Simple, auto-resume, plain HTTP |
| Chat, games, collaborative editing | WebSocket | Two-way, low overhead |
| Rare updates, or strict proxies block the rest | Long polling | Works almost anywhere |
| Updates every few minutes, delay is fine | Short polling | Simplest thing that works |
| Calls between internal services | gRPC | Typed, fast, streaming, deadlines |
| Public API for many unknown clients | REST on HTTPS | Everyone can use it |
| Video or voice in a browser | WebRTC | Low delay, works through NAT, encrypted |
| Live video to huge audiences, seconds of delay is fine | HLS or DASH | Plain HTTP segments scale well on CDNs |
| Phone systems, PSTN connection | SIP and RTP | Telecom industry standard |
| Sensors on weak links, many subscribers | MQTT | Tiny, broker fan-out, keeps sessions |
| Business jobs that must not be lost | AMQP | Queues, routing, acks |
| Very small devices, UDP, REST style | CoAP | Fits in one small datagram |
| Huge event log with replay | Kafka | Not in this group, but often the right answer |

```text
Is the receiver a browser?
 |
 +-- Yes --> Need audio or video? ------- Yes --> WebRTC
 |           |
 |           No
 |           |
 |           Client must send often? ---- Yes --> WebSocket
 |           |
 |           No --> Server push only? --- Yes --> SSE
 |                  |
 |                  No --> Long or short polling
 |
 +-- No  --> Service to service?
             |
             +-- Instant answer needed --> gRPC
             +-- Work can wait, no loss -> AMQP
             |
             Small device?
             |
             +-- TCP is fine, broker ---> MQTT
             +-- UDP, very tiny --------> CoAP
```

## Example

### Example 1: Check a WebSocket upgrade

First compute the accept value by hand. This works on Linux or Git Bash:

```bash
# Linux
KEY='dGhlIHNhbXBsZSBub25jZQ=='
GUID='258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
printf '%s' "$KEY$GUID" | openssl dgst -sha1 -binary | openssl base64
```

```text
s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

Now ask a server to upgrade:

```bash
# Linux, or curl.exe on Windows
curl -i -N --http1.1 \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  http://chat.example.com/ws
```

```text
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

The same value proves the server did the work. A `200` here means the server or a proxy ignored the upgrade. A `426 Upgrade Required` means the version is not accepted.

The frame for the text "Hello" from the server, from RFC 6455:

```text
81 05 48 65 6c 6c 6f
81 = FIN set, opcode 1 (text)
05 = no mask, length 5
48 65 6c 6c 6f = "Hello"
```

The same text from a client, masked with key `37 fa 21 3d`:

```text
81 85 37 fa 21 3d 7f 9f 4d 51 58
```

So the overhead is 2 bytes from the server and 6 bytes from the client.

### Example 2: Cost of polling at scale

Assume 10,000 clients, one update per minute, and 800 bytes of HTTP headers per round trip. These are assumptions for the math. TCP, IP and TLS overhead are left out.

| Method | Requests or frames per second | Traffic |
|---|---|---|
| Short poll every 5 s | 10,000 / 5 = 2,000 req/s | 2,000 x 800 = 1.6 MB/s (12.8 Mbps) of headers |
| Long poll, 30 s hold | 10,000 x 2 / 60 = 333 req/s | 333 x 800 = 0.27 MB/s (2.1 Mbps) of headers |
| WebSocket, 100-byte update | 10,000 / 60 = 167 frames/s | 167 x 102 = 17 kB/s (0.14 Mbps) including data |

The long-poll rate is 2 per minute because each client gets one timeout and one real update in each 60-second cycle. WebSocket is far cheaper in bytes, but the server must hold 10,000 open connections.

### Example 3: MQTT QoS 1 and 2 with Mosquitto

The broker is at 192.168.1.50. First test that the port is reachable.

```bash
# Linux
nc -vz 192.168.1.50 1883
```

```text
Connection to 192.168.1.50 1883 port [tcp/*] succeeded!
```

```powershell
# Windows
Test-NetConnection -ComputerName 192.168.1.50 -Port 1883
```

```text
ComputerName     : 192.168.1.50
RemoteAddress    : 192.168.1.50
RemotePort       : 1883
InterfaceAlias   : Ethernet
SourceAddress    : 192.168.1.20
TcpTestSucceeded : True
```

Subscribe in one terminal, then publish in another. `-d` shows protocol packets. Exact output varies by Mosquitto version.

```bash
# Terminal 1 (Linux or Windows)
mosquitto_sub -h 192.168.1.50 -t 'home/+/temp' -q 2 -v

# Terminal 2, QoS 1
mosquitto_pub -h 192.168.1.50 -i sensor1 \
  -t home/kitchen/temp -m 22.5 -q 1 -d
```

```text
Client sensor1 sending CONNECT
Client sensor1 received CONNACK (0)
Client sensor1 sending PUBLISH (d0, q1, r0, m1, 'home/kitchen/temp', ... (4 bytes))
Client sensor1 received PUBACK (Mid: 1, RC:0)
Client sensor1 sending DISCONNECT
```

With `-q 2` the middle of the output changes:

```text
Client sensor1 sending PUBLISH (d0, q2, r0, m1, 'home/kitchen/temp', ... (4 bytes))
Client sensor1 received PUBREC (Mid: 1)
Client sensor1 sending PUBREL (m1)
Client sensor1 received PUBCOMP (Mid: 1, RC:0)
```

Terminal 1 prints the topic and the value:

```text
home/kitchen/temp 22.5
```

Here `q` is QoS, `r` is the retain flag, `d` is the duplicate flag and `m` is the packet ID.

### Example 4: Bandwidth of one VoIP call

G.711 is a codec that sends 64 kbps of voice. With a 20 ms packet interval, each packet holds 160 bytes of audio and there are 50 packets per second.

```text
Voice payload    160 bytes   (64,000 bits/s x 0.020 s / 8)
RTP header        12 bytes
UDP header         8 bytes
IPv4 header       20 bytes
                 ---------
IP packet        200 bytes
Ethernet          18 bytes   (14 header + 4 FCS)
                 ---------
Frame            218 bytes

IP bandwidth:       200 x 8 x 50 =  80.0 kbps, each direction
Ethernet bandwidth: 218 x 8 x 50 =  87.2 kbps, each direction
```

So a link with 10 calls needs about 872 kbps each way on Ethernet. G.729 uses only 20 bytes of voice per 20 ms packet, so the IP packet is 60 bytes. That is 24 kbps at IP and 31.2 kbps on Ethernet. The headers cost more than the voice.

RTP numbers for that G.711 stream:

```text
Packet   Seq    Timestamp    Sent at
1        1000   16000         0 ms
2        1001   16160        20 ms
3        1002   16320        40 ms
```

The timestamp adds 160 because audio is sampled 8,000 times per second, and 20 ms is 160 samples.

### Example 5: A real SIP INVITE

Alice's phone at 10.0.1.20 calls Bob. Note that the SDP says where to send the audio.

```text
INVITE sip:bob@example.com SIP/2.0
Via: SIP/2.0/UDP 10.0.1.20:5060;branch=z9hG4bK776asdhds
Max-Forwards: 70
From: Alice <sip:alice@example.com>;tag=1928301774
To: Bob <sip:bob@example.com>
Call-ID: a84b4c76e66710@10.0.1.20
CSeq: 314159 INVITE
Contact: <sip:alice@10.0.1.20:5060>
Content-Type: application/sdp
Content-Length: 156

v=0
o=alice 2890844526 2890844526 IN IP4 10.0.1.20
s=-
c=IN IP4 10.0.1.20
t=0 0
m=audio 49170 RTP/AVP 0 8
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
```

The `c=` line gives the media IP. The `m=` line gives the RTP port (49170) and two codec choices. `Call-ID` names the call, `CSeq` orders requests, and `Via` records the path back.

Capture a call on a Linux PBX. The RTP range is an example, since it depends on the system.

```bash
# Linux
sudo tcpdump -i eth0 -nn 'udp port 5060 or udp portrange 10000-20000'
```

In Wireshark, the display filter `sip || rtp || rtcp` shows the same traffic. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

### Example 6: gRPC with grpcurl

`grpcurl` is a command-line gRPC client, like curl for gRPC. `-plaintext` means HTTP/2 without TLS. This call needs server reflection, or a `-proto` file.

```bash
# Linux or Windows
grpcurl -plaintext -d '{"sku":"A100","qty":2}' \
  10.0.2.15:50051 shop.Cart/AddItem
```

```text
{
  "cartId": "c-7781",
  "total": 3
}
```

A failed call shows the gRPC status, even though HTTP said 200:

```text
ERROR:
  Code: NotFound
  Message: sku A100 not found
```

Protobuf is compact. A message with field 1 set to 150 takes just 3 bytes: `08 96 01`. On the wire, with the gRPC prefix, it is `00 00 00 00 03 08 96 01`. The JSON form `{"id":150}` takes 10 bytes.

### Example 7: CoAP request in bytes

Get the temperature from a sensor.

```bash
# Linux, libcoap client
coap-client -m get coap://[2001:db8::10]/temperature
```

```text
22.5 C
```

The request is 16 bytes of UDP payload:

```text
40 01 7d 34  bb  74 65 6d 70 65 72 61 74 75 72 65
40       Ver 1, type CON, token length 0
01       Code 0.01 = GET
7d 34    Message ID
bb       Option delta 11 (Uri-Path), length 11
74...65  "temperature"
```

The response is 11 bytes:

```text
60 45 7d 34  ff  32 32 2e 35 20 43
60       Ver 1, type ACK, token length 0
45       Code 2.05 Content
7d 34    Same message ID
ff       Payload marker
32...43  "22.5 C"
```

An HTTP request and response for the same data would usually take hundreds of bytes.

### Example 8: Choosing for a food delivery app

| Part of the system | Choice | Reason |
|---|---|---|
| Order status on a web page | SSE | One way, simple, resumes after a drop |
| Driver phone location to the back end | MQTT | Tiny messages on bad mobile links |
| Chat between customer and driver | WebSocket | Two-way and instant |
| In-app voice call | WebRTC | Private media, NAT traversal |
| Pricing service to order service | gRPC | Fast internal calls with deadlines |
| "Order paid" event to billing and email | AMQP | Must not be lost, many consumers |

## Going Deeper

### WebSocket in production

- **Proxies must pass the upgrade.** `Upgrade` and `Connection` are hop-by-hop headers, so a reverse proxy drops them unless told otherwise. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

```text
# NGINX
location /ws/ {
    proxy_pass http://chat_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

- **Idle timeouts kill quiet connections.** The NGINX default read timeout is 60 s, and the default AWS ALB idle timeout is also 60 s. Send a heartbeat more often than that. Browser JavaScript cannot send Ping frames, so many apps send their own small heartbeat message.
- **Connections are stateful.** Each user is tied to one server. To reach a user on another server, you need a shared message layer such as Redis pub/sub or NATS.
- **Reconnect storms.** When a server restarts, thousands of clients return at the same time. Use exponential backoff with random jitter.
- **Security.** Browsers do not apply CORS to WebSocket. The server must check the `Origin` header, or it risks cross-site WebSocket hijacking. Use `wss://` and authenticate during the upgrade.
- **Connection limits.** A server can serve many clients on port 443, because each connection has a unique 4-tuple. The real limits are memory, file descriptors and proxy-to-backend ports.
- **Close codes.** `1000` is normal, `1001` is going away, and `1011` is a server error. `1006` means abnormal closure, and it is never sent on the wire. The browser reports it when the connection simply dies.

### WebSocket over HTTP/2 and HTTP/3

HTTP/2 has no `101` upgrade, and it forbids the `Connection` header. RFC 8441 adds Extended CONNECT: a stream opens with `:method CONNECT` and `:protocol websocket`. RFC 9220 does the same for HTTP/3. Many frameworks and proxies still use HTTP/1.1 for WebSocket.

### SSE in production

Buffering is the usual enemy. A proxy that collects the response before forwarding makes events arrive in late bursts.

```text
# NGINX
location /stream {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_read_timeout 1h;
}
```

Send a comment line such as `: ping` every 15-30 seconds to keep the path alive.

### gRPC and load balancing

gRPC keeps one long-lived HTTP/2 connection and sends many calls over it. A layer 4 load balancer picks a backend per connection, so all calls go to one server while others sit idle.

Fixes:

- Use a layer 7 proxy that understands HTTP/2 and balances per request, such as Envoy.
- Use client-side balancing, where the client knows all the backends.
- Set a maximum connection age on the server so clients reconnect from time to time.

See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md) and [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md).

Other points worth knowing:

- **Deadlines:** the client sets a time limit, sent in `grpc-timeout`. Servers can pass the remaining time to the next service, so a timed-out request does not keep using resources.
- **Head-of-line blocking:** HTTP/2 runs on TCP, so one lost packet delays all streams on the connection. This is one reason HTTP/3 uses QUIC. See [QUIC](../06-transport-layer/08-quic.md).

### WebRTC details

**Why STUN sometimes fails.** Some NATs (often called symmetric NATs) give a different public port for each destination. The port STUN saw is not the port the peer will reach, so hole punching fails. Then TURN relays the traffic. Only a minority of calls need TURN, and the share depends on the networks. TURN does cost real bandwidth, because every byte goes through your server.

**STUN and TURN ports.** The default is 3478 on UDP or TCP, and 5349 for TLS or DTLS. Many teams also offer TURN over TLS on 443 for networks that block everything else.

**ICE priority.** RFC 8445 gives this formula, where the type preference is usually 126 for host, 100 for server reflexive and 0 for relay:

```text
priority = 2^24 x type_pref + 2^8 x local_pref + (256 - component)

host:  a=candidate:1 1 udp 2130706431 10.0.1.20 54321 typ host
srflx: a=candidate:2 1 udp 1694498815 203.0.113.7 61000 typ srflx
       raddr 10.0.1.20 rport 54321
relay: a=candidate:3 1 udp 16777215 198.51.100.20 49152 typ relay
       raddr 203.0.113.7 rport 61000
```

**Group calls.** Direct peer connections do not scale.

| Design | How it works | Cost |
|---|---|---|
| Mesh | Each peer sends to all the others | In a 6-person call, each uploads 5 streams |
| SFU | Selective forwarding unit. Each sends once, and the server forwards streams | Low server CPU. Most modern systems use it |
| MCU | Multipoint control unit. The server decodes and mixes one stream | Heavy server CPU |

**Encryption.** WebRTC media must be encrypted, using SRTP. With an SFU, media is decrypted at the server unless the app adds its own end-to-end layer. Do not assume "WebRTC" means "end-to-end encrypted".

### VoIP quality and NAT

**Jitter** is variation in packet arrival time. A **jitter buffer** holds packets briefly and plays them at an even rate. A bigger buffer removes more jitter but adds delay.

Common design targets are one-way delay under 150 ms (ITU-T G.114), jitter under 30 ms and loss under 1 percent. These are guidelines, not hard limits. Voice is usually marked DSCP EF. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

**One-way audio** is the classic problem. Signalling works, so the phone rings, but the SDP says `c=IN IP4 10.0.1.20`, a private address. The far side sends RTP there, and it goes nowhere.

Fixes include an SBC that rewrites SDP and relays media, STUN or ICE on the phone, opening the RTP range on the firewall, and turning off a broken **SIP ALG**. A SIP ALG is a router feature that rewrites SIP messages, and it often does this badly.

```text
! Cisco IOS: disable SIP ALG for NAT
no ip nat service sip udp port 5060
```

Also check that the two endpoints agree on a codec. A `488 Not Acceptable Here` response often means no shared codec.

**Security.** SIP can use TLS (often called SIPS, on port 5061) and RTP can use SRTP. Plain SIP and RTP can be captured and replayed with common tools. Registration uses digest authentication, which is why a `401` challenge is normal on a first `REGISTER`.

### MQTT at scale

- **Version 5.0 adds:** reason codes on most responses, message expiry, session expiry, topic aliases and user properties.
- **Shared subscriptions** (5.0) split one topic's messages across a group of consumers, like `$share/workers/orders/#`.
- **Browsers** cannot open raw TCP sockets, so MQTT there runs over WebSocket. Port varies by broker.
- **Client IDs must be unique.** If a second client connects with the same ID, the broker disconnects the first. Two devices with one ID will fight forever.
- **QoS 2 is rarely needed.** It costs four packets and stored state on both sides. Many teams use QoS 1 with idempotent handling. **Idempotent** means processing a message twice gives the same result as once.

### MQTT vs AMQP vs CoAP

| Feature | MQTT | AMQP 0-9-1 | CoAP |
|---|---|---|---|
| Transport | TCP | TCP | UDP |
| Ports | 1883, 8883 | 5672, 5671 | 5683, 5684 |
| Model | Pub/sub with a broker | Exchanges and queues | Request and response, plus Observe |
| Header cost | From 2 bytes | Frame header 7 bytes, plus 1 end byte | 4 bytes |
| Best for | Telemetry, mobile | Enterprise back ends | Tiny, battery devices |
| Wildcards | `+` and `#` | `*` and `#` | None |
| Delivery | QoS 0, 1, 2 | Acks and confirms | CON or NON |

Watch out: `+` in MQTT and `*` in AMQP both mean one level. The `#` is multi-level in both, though the MQTT level separator is `/` and the AMQP one is `.`.

### AMQP on the wire

An AMQP 0-9-1 connection starts when the client sends the bytes `AMQP` followed by `0 0 9 1`. After that, every frame has a type (1 byte), channel (2 bytes), size (4 bytes), payload and an end byte of `0xCE`.

Frame types are method, content header, content body and heartbeat. Messages that fail can go to a **dead letter exchange** for later study.

### CoAP vs HTTP, and security

- Since CoAP uses UDP, there is no handshake before the first request unless DTLS is used. DTLS brings back some cost, and small devices may use pre-shared keys.
- **OSCORE** (RFC 8613) protects CoAP messages at the object level, so a proxy can forward them without reading the content.
- Since UDP has no handshake, an open CoAP server can be abused for reflection attacks with a spoofed source address. Do not expose CoAP devices to the internet without protection. See [Common Network Attacks](../09-security/03-common-network-attacks.md).
- CoAP can also run over TCP, TLS and WebSockets (RFC 8323), which helps when firewalls block UDP.

### Other protocols an interviewer might name

| Protocol | Use | Notes |
|---|---|---|
| WebTransport | Browser streams and datagrams over HTTP/3 | Still maturing, and browser support varies. A possible future choice beyond WebSocket |
| RTSP | Control of media streams, common on IP cameras | TCP 554 by default. RFC 7826 is the 2.0 version |
| RTMP | Sending live video to streaming platforms | TCP 1935 by default. Older, still used for ingest |
| HLS, DASH | Video over plain HTTP in small segments | Scales with CDNs. Delay is usually seconds, not milliseconds |
| XMPP | Open messaging and presence | TCP 5222 client, 5269 server. RFC 6120 |
| H.323 | Older ITU VoIP suite | SIP has mostly replaced it |
| Webhook | A server calls your HTTP URL on an event | Server-to-server push, not for browsers |
| Kafka | Distributed event log | Own binary protocol on TCP, 9092 by convention |

## Common Mistakes

- **"WebSocket is not HTTP, so it needs a special port."** It starts as HTTP on 80 or 443 and then upgrades on the same connection.
- **"WebSocket masking is for security."** It protects broken proxies from cache poisoning. Encryption comes only from `wss://`.
- **"Sec-WebSocket-Key authenticates the client."** It only proves that the server speaks WebSocket. You still need tokens or cookies.
- **"Always use WebSocket for push."** If data flows one way, SSE is simpler and auto-resumes.
- **"SSE is limited to 6 streams."** That limit is for HTTP/1.1 in browsers. With HTTP/2 the many streams share one connection.
- **"A gRPC call that returned HTTP 200 succeeded."** gRPC errors are in the `grpc-status` trailer. Monitoring that looks only at HTTP status misses them.
- **"A normal TCP load balancer handles gRPC well."** It sends all calls on one connection to one backend. Balance per request at layer 7.
- **"WebRTC needs no servers."** It still needs signalling, usually STUN, and sometimes TURN.
- **"WebRTC defines signalling."** It does not. You choose how to carry the SDP.
- **"SIP carries the voice."** SIP only sets up the call. RTP carries the voice.
- **"RTP is a reliable protocol."** It only adds sequence numbers and timestamps. Nothing is resent.
- **"MQTT QoS and network QoS are the same."** MQTT QoS is about delivery, not about DSCP or priority.
- **"QoS 2 means the end application handles it once."** It covers each hop and the protocol layer. If the app crashes after receiving, the work can still fail.
- **"AMQP means RabbitMQ, and 1.0 is a newer version of 0-9-1."** They are different protocols. A 0-9-1 client cannot talk to a 1.0 peer.
- **"CoAP is just MQTT over UDP."** CoAP is request and response like HTTP. MQTT is pub/sub through a broker.
- **"A successful port test proves UDP works."** Tools like `nc -vz` are only reliable for TCP. For UDP protocols such as CoAP and RTP, use a real client or a packet capture.

## Interview Questions

**Q: Walk me through the WebSocket handshake.**
A: The client opens TCP and TLS, then sends an HTTP/1.1 GET with `Upgrade: websocket`, `Connection: Upgrade`, a random `Sec-WebSocket-Key` and version 13. The server appends a fixed GUID to the key, takes the SHA-1 hash, base64 encodes it, and returns it as `Sec-WebSocket-Accept` in a `101 Switching Protocols` response. After that the same TCP connection carries WebSocket frames in both directions.

**Q: Compare long polling, SSE and WebSocket. When do you pick each?**
A: Long polling is normal HTTP where the server holds the request until there is data. It works nearly everywhere but has full headers for each message. SSE is one long HTTP response for server-to-client text, with built-in reconnect. WebSocket is two-way with very small frames but needs upgrade support and custom reconnect logic. I pick SSE for feeds and notifications, WebSocket for chat and games, and long polling as a fallback.

**Q: Why are WebSocket client frames masked?**
A: To stop a malicious web page from crafting bytes that look like HTTP to an old intermediary, which could poison its cache. The mask key is random and is sent in the frame, so it gives no secrecy. Confidentiality comes from TLS with `wss://`.

**Q: Why does gRPC use HTTP/2, and what is the load balancing problem?**
A: HTTP/2 gives multiplexed streams on one connection, flow control and trailers, and gRPC uses them for streaming calls and status. The problem is that one long-lived connection carries many calls, so a layer 4 balancer sends them all to one backend. The fix is a layer 7 proxy that balances each request, or client-side balancing.

**Q: How do two WebRTC peers behind NAT connect? What is the difference between STUN and TURN?**
A: They exchange SDP offers and answers through a signalling server that the app provides. ICE gathers candidates: local addresses, public addresses learned from STUN, and relay addresses from TURN. It tests the pairs and selects the best one that works. STUN only tells a peer its public address, so media still goes direct. TURN relays all the media, so it works in harder NAT cases but costs bandwidth.

**Q: What do SIP, SDP, RTP and RTCP each do in a VoIP call?**
A: SIP sets up, changes and ends the call. SDP sits inside SIP and describes the media, such as codec, IP and port. RTP carries the audio on UDP with sequence numbers and timestamps. RTCP sends quality reports on loss, jitter and round-trip time.

**Q: A VoIP call connects, but only one side hears audio. What do you check?**
A: Signalling works, so I look at the media path. Most often the SDP has a private IP because of NAT, or a firewall blocks the RTP port range in one direction. I would capture packets and compare the SDP address with where RTP really goes. Then I would check for a broken SIP ALG, and fix it with an SBC or by opening the right ports.

**Q: Calculate the bandwidth of a G.711 call with 20 ms packets.**
A: Payload is 160 bytes. Add RTP 12, UDP 8 and IP 20 to get 200 bytes. At 50 packets per second that is 80 kbps at IP. With 18 bytes of Ethernet it is 218 bytes, which gives 87.2 kbps each way.

**Q: Explain MQTT QoS 0, 1 and 2.**
A: QoS 0 is at most once, with one PUBLISH and no ack. QoS 1 is at least once: the sender retries until it gets PUBACK, so duplicates can happen. QoS 2 is exactly once with a four-step exchange: PUBLISH, PUBREC, PUBREL and PUBCOMP. QoS applies per hop, and the subscriber gets the lower of the publish and subscribe levels.

**Q: Why do we need QoS in MQTT if TCP is already reliable?**
A: TCP protects data only while the connection is alive. If a mobile link drops, data in flight can be lost, and TCP cannot resend it on a new connection. MQTT stores unacknowledged messages and resends them after reconnecting. QoS 2 also removes duplicates across such reconnects.

**Q: MQTT vs AMQP: how would you choose?**
A: MQTT is small and made for many devices on poor links, using simple topics and a broker. AMQP is richer, with exchanges, queues, routing and consumer acks, and it suits business workflows between servers. I would use MQTT at the edge for telemetry and AMQP inside the data centre for work queues.

**Q: How does CoAP stay reliable on UDP, and why not simply use HTTP?**
A: CoAP has confirmable messages. The sender retries with exponential backoff until an ACK arrives. Duplicates are found by message ID. HTTP over TCP needs a handshake and large text headers, which waste battery and memory on tiny devices. CoAP has a 4-byte header and often fits in one small datagram.

## Quick Recap

- HTTP cannot push by itself. Long polling fakes it, SSE gives one-way streaming, and WebSocket gives two-way frames after a `101` upgrade.
- gRPC is RPC with protobuf on HTTP/2. Check `grpc-status`, not the HTTP code, and balance at layer 7.
- WebRTC uses your own signalling plus ICE, STUN and TURN to find a path, then DTLS and SRTP for security.
- In VoIP, SIP sets up the call, SDP describes the media, RTP carries it on UDP, and RTCP reports quality.
- MQTT is light pub/sub through a broker, with QoS 0, 1 and 2 applied per hop.
- AMQP 0-9-1 routes through exchanges into queues with acks. AMQP 1.0 is a different protocol.
- CoAP is small REST on UDP for tiny devices. Pick the simplest protocol that meets the direction, delay and reliability needs.
