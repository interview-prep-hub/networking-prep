# Wi-Fi Basics

**Level:** Beginner | **Prerequisites:** [Network Devices](../01-foundations/04-network-devices.md), [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md), [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md)

## In One Line

Wi-Fi is a family of radio standards (IEEE 802.11) that replaces the Ethernet cable with a shared radio channel, where devices take turns talking through an access point.

## Why It Matters

A cable gives each device its own private path. Radio does not. Everyone nearby shares the same air, so Wi-Fi must solve problems a switch never sees: who may talk now, how to survive noise, and how to find a network with no cable to plug in.

Interviewers ask about Wi-Fi because it shows whether you understand shared media. Common questions are: "Why is Wi-Fi half duplex?", "What is CSMA/CA?", "Why use channels 1, 6 and 11?" and "Why is my speed test far below the number on the box?"

## Simple Explanation

Picture a room with one speaker system and many people. Only one person can talk at a time. If two talk together, nobody understands either of them.

So people follow polite rules. Listen first. If the room is quiet, wait a short random moment, then speak. The listener says "got it" after each sentence. If you hear no "got it", you assume it was lost and try again.

That room is one Wi-Fi channel. The people are your phone, laptop and TV. The host in the middle is the access point (AP), a device that connects radio clients to the wired network.

**Where the analogy stops:** people in a room hear everyone. Radio devices do not. Two laptops can both hear the AP but not hear each other. That problem has a name, the hidden node, and we cover it below.

## How It Works

### 802.11 and the Wi-Fi names

**IEEE 802.11** is the standard that defines Wi-Fi. IEEE is the engineering body that writes it. 802.11 covers the two lowest layers: the physical layer (the radio) and the MAC layer (the rules for taking turns).

**Wi-Fi Alliance** is a separate industry group. It tests products and owns the "Wi-Fi" brand. In 2018 it added simple generation numbers, because "802.11ax" means little to a shopper.

| Wi-Fi name | IEEE standard | Around | Bands (GHz) | Max width | Max modulation | Max speed (theory) |
|---|---|---|---|---|---|---|
| Wi-Fi 4 | 802.11n | 2009 | 2.4, 5 | 40 MHz | 64-QAM | 600 Mbps |
| Wi-Fi 5 | 802.11ac | 2013 | 5 | 160 MHz | 256-QAM | 6.9 Gbps |
| Wi-Fi 6 | 802.11ax | 2019-2021 | 2.4, 5 | 160 MHz | 1024-QAM | 9.6 Gbps |
| Wi-Fi 6E | 802.11ax in 6 GHz | 2021 | 2.4, 5, 6 | 160 MHz | 1024-QAM | 9.6 Gbps |
| Wi-Fi 7 | 802.11be | 2024 | 2.4, 5, 6 | 320 MHz | 4096-QAM | 46 Gbps |

Three notes on this table:

- **Wi-Fi 6E is not a new standard.** It is Wi-Fi 6 with permission to use the 6 GHz band.
- **Max speed is a lab number.** Wi-Fi 5's 6.9 Gbps needs 8 streams, and Wi-Fi 6's 9.6 Gbps needs 8 streams at 160 MHz. A phone usually has 2 streams.
- **46 Gbps assumes 16 streams at 320 MHz.** Real Wi-Fi 7 products use at most 8 streams, which caps at about 23 Gbps. A 2-stream Wi-Fi 7 phone reaches about 5.8 Gbps on paper.

Older standards still appear in interviews. They have no official generation number, though people sometimes call them Wi-Fi 1 to 3.

| Standard | Year | Band | Max speed |
|---|---|---|---|
| 802.11 (original) | 1997 | 2.4 GHz | 2 Mbps |
| 802.11b | 1999 | 2.4 GHz | 11 Mbps |
| 802.11a | 1999 | 5 GHz | 54 Mbps |
| 802.11g | 2003 | 2.4 GHz | 54 Mbps |

### The three bands: 2.4, 5 and 6 GHz

A **band** is a block of radio frequencies. GHz (gigahertz) means billions of waves per second. Higher frequency carries more spectrum (room for data) but loses strength faster, especially through walls.

**Analogy:** a low bass note from a neighbour's speaker passes through the wall. High notes do not. **Where it stops:** sound and radio are different physics. The fuller truth is that signal loss depends on frequency, antenna size and wall material, so results differ by building.

| Property | 2.4 GHz | 5 GHz | 6 GHz |
|---|---|---|---|
| Range and wall penetration | Best | Medium | Shortest |
| Total spectrum | 83.5 MHz | About 500-600 MHz | Up to 1200 MHz (US), 480 MHz (EU) |
| Non-overlapping 20 MHz channels | 3 | About 25 (US) | Up to 59 (US) |
| Crowding | High | Medium | Low |
| Other users | Bluetooth, microwave ovens, Zigbee, baby monitors | Some radar | Few, varies by country |
| Client support | Almost all | Most | Wi-Fi 6E and 7 only |
| Security | Any | Any | WPA3 or OWE required |

Choose 2.4 GHz for range and small devices, such as sensors and smart plugs. Choose 5 GHz for general use. Choose 6 GHz for top speed in the same room or the next one.

Allowed channels and power levels are set by each country's regulator, so always check your region. **Band steering** is an AP feature that nudges dual-band clients toward the faster band.

### Channels and overlap

A **channel** is a slice of a band. Two devices must use the same channel to talk.

In 2.4 GHz, channel numbers are only 5 MHz apart, but each signal is about 20 MHz wide (22 MHz for old 802.11b). So nearby channels overlap. Center frequency is 2407 + (5 x channel) MHz, which gives channel 1 at 2412 MHz, channel 6 at 2437 MHz and channel 11 at 2462 MHz.

```text
MHz    2400      2420      2440      2460      2480
       |         |         |         |         |
ch 1    [=========]
ch 3         [=========]
ch 6                 [=========]
ch 11                            [=========]

ch 1 and ch 6 do not overlap. ch 6 and ch 11 do not overlap.
ch 3 overlaps both ch 1 and ch 6.
```

That is why channels 1, 6 and 11 are the standard plan in the US. Europe also allows channels 12 and 13, so a 1, 5, 9, 13 plan is possible with 20 MHz signals. Channel 14 exists only in Japan, and only for old 802.11b.

**Analogy:** a road where trucks are wider than the painted lanes. Trucks in lanes 1 and 6 are safe. A truck in lane 3 scrapes both sides.

**Where it stops:** a real truck blocks the road. A weak signal on a nearby channel only adds noise.

Two kinds of interference matter:

- **Co-channel interference (CCI):** two networks on the same channel. They hear each other and take turns. Each gets less airtime, but the rules still work.
- **Adjacent-channel interference (ACI):** two networks on partly overlapping channels. They often cannot decode each other, so they do not take turns. They just corrupt each other's frames.

This surprises beginners: sharing channel 6 with a neighbour is usually better than sitting on channel 4 or 8 between two networks.

5 GHz channels are numbered at 20 MHz steps (36, 40, 44, 48, and so on), so they do not overlap at 20 MHz. Some 5 GHz channels (52-144 in the US and much of Europe) use **DFS**, Dynamic Frequency Selection. The AP must check for radar first, usually for 60 seconds, and must leave the channel if it detects radar.

### Channel width

**Channel width** is how much spectrum one transmission uses. You can bond 20 MHz channels into 40, 80, 160 or even 320 MHz channels.

**Analogy:** a wider road carries more cars at once. **Where it stops:** a road does not get noisier when widened. A wider radio channel picks up about 3 dB more noise each time you double it.

| Width | Bands | Rough effect |
|---|---|---|
| 20 MHz | 2.4, 5, 6 GHz | Most stable. Best for crowded places |
| 40 MHz | 2.4, 5, 6 GHz | Avoid in 2.4 GHz: it uses most of the band |
| 80 MHz | 5, 6 GHz | Common home default on 5 GHz |
| 160 MHz | 5, 6 GHz | Very fast, few channels available in 5 GHz |
| 320 MHz | 6 GHz, Wi-Fi 7 only | Needs a clean band. The EU has room for only one |

Doubling width slightly more than doubles the raw rate. The cost is fewer channels to choose from, so networks collide more. Offices often use 20 or 40 MHz in 5 GHz for this reason.

### SSID, BSSID and ESSID

**BSS** (Basic Service Set) means one AP radio plus its connected clients. Three names follow from it.

- **SSID** (Service Set Identifier): the network name people see, such as `HomeNet`. It can be up to 32 bytes.
- **BSSID** (Basic Service Set Identifier): the unique ID of one AP radio. It is a 48-bit MAC address, such as `00:00:5e:00:53:a1`.
- **ESSID** (Extended Service Set Identifier): the same SSID used across many APs, which together form an **ESS** (Extended Service Set). In daily speech, ESSID and SSID mean the same name.

```text
                    Wired LAN (distribution system)
             +-----------------+-----------------+
             |                                   |
          +--+--+                             +--+--+
          | AP1 |                             | AP2 |
          +--+--+                             +--+--+
             |                                   |
  SSID:  HomeNet                             HomeNet
  BSSID: 00:00:5e:00:53:a1                   00:00:5e:00:53:b1
  Chan:  36                                  149

  One ESS (name HomeNet), two BSSs, two BSSIDs.
```

**Analogy:** the SSID is a shop chain's name. The BSSID is the address of one branch. **Where it stops:** one AP can act like many "branches". A dual-band AP has a different BSSID per radio, and each extra SSID usually adds another BSSID.

Clients move between BSSIDs under one SSID. This is **roaming**, and the client decides when to do it. See [Enterprise Wi-Fi](./03-enterprise-wifi.md).

A **beacon** is a small frame that an AP broadcasts to announce itself, usually every 102.4 milliseconds (100 time units). Hiding the SSID removes the name from beacons, but it adds no real security, because clients still send the name in other frames.

### The association process

**Association** is how a client joins an AP, like signing in at a hotel desk. There are three states: not authenticated, authenticated but not associated, and associated.

1. **Scan.** The client finds APs. In passive scanning it listens for beacons. In active scanning it sends a probe request, and APs send probe responses.
2. **Authenticate.** With Open System authentication this is a simple two-frame hello, not real security. In WPA3-Personal, the password exchange (SAE) happens at this step.
3. **Associate.** The client sends an association request with its abilities (speeds, streams, security). The AP replies with an association response and an **Association ID**, a number identifying this client.
4. **Secure the link.** WPA2 and WPA3 run a 4-way handshake (four EAPOL frames) to create encryption keys. Details are in [Wi-Fi Security](./02-wifi-security.md).
5. **Get on the network.** The client now uses normal LAN protocols, such as [DHCP](../07-application-protocols/04-dhcp.md) for an IP address and [ARP](../03-ethernet-and-switching/05-arp.md) to find the gateway.

```text
Client                                            AP
  |     Beacon (SSID=HomeNet)                     |
  |<----------------------------------------------|
  |     Probe Request                             |
  |---------------------------------------------->|
  |     Probe Response                            |
  |<----------------------------------------------|
  |     Authentication Request                    |
  |---------------------------------------------->|
  |     Authentication Response                   |
  |<----------------------------------------------|
  |     Association Request                       |
  |---------------------------------------------->|
  |     Association Response (AID=1)              |
  |<----------------------------------------------|
  |     4-way handshake (EAPOL 1 to 4)            |
  |<--------------------------------------------->|
  |     DHCP, ARP, then normal traffic            |
  |<--------------------------------------------->|
```

### CSMA/CA: taking turns on the air

**CSMA/CA** means Carrier Sense Multiple Access with Collision Avoidance. Break it into parts:

- **Carrier sense:** listen before you talk.
- **Multiple access:** many devices share one channel.
- **Collision avoidance:** try hard to prevent collisions, because you cannot detect them.

Wired Ethernet used CSMA/CD (Collision Detection), which stops when it detects a collision. Wi-Fi cannot do this. While a radio transmits, its own signal drowns out anything it could receive. Also, a collision at the receiver may be silent at the sender.

Here is one frame sent on 5 GHz with OFDM timing:

1. **Listen.** The device checks if the channel is busy. This is physical carrier sense.
2. **Check the timer.** The **NAV** (Network Allocation Vector) is a countdown set from other frames' Duration field. It says how long the air is reserved. This is virtual carrier sense.
3. **Wait DIFS.** If idle, wait a fixed gap, the DCF Interframe Space, 34 microseconds here.
4. **Back off.** Pick a random number of slots from 0 to CW (the contention window, 15 at the start). One slot is 9 microseconds. If it picks 6, it waits 54 microseconds. The count freezes if someone else starts.
5. **Send.** Transmit when the count reaches zero.
6. **Get an ACK.** The receiver waits SIFS (Short Interframe Space, 16 microseconds here) and sends an acknowledgement.
7. **Retry if needed.** No ACK means something failed. The sender doubles CW (15, 31, 63, up to 1023) and tries again, marking the frame as a retry.

```text
 busy  | DIFS |  backoff   |   DATA frame   | SIFS | ACK |
-------+------+------------+----------------+------+-----+-->
                                                         time
 |<-other->|<-------- one successful exchange ---------->|
```

**Analogy:** in a meeting, you wait for silence, count a few random seconds, then speak. **Where it stops:** humans can notice when two people start together. Radios only find out from a missing ACK.

Timers differ by band and standard. For example, SIFS is 10 microseconds in 2.4 GHz. With WMM, a QoS feature, voice and video wait shorter gaps than other traffic. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

Unicast frames are acknowledged. **Broadcast and multicast frames are not**, and they are usually sent at a low rate, so heavy broadcast traffic wastes airtime.

### Hidden node

Two clients can reach the AP but be out of range of each other. Each hears silence, each sends, and the frames collide at the AP.

```text
   <------ range of A ------>
                 <------ range of C ------>
       [A] .......... [AP] .......... [C]

   A hears AP. C hears AP. A cannot hear C.
   Both transmit together. The AP receives garbage.
```

**Analogy:** two people stand in separate rooms and talk to a friend in the doorway. They cannot hear each other, so they both speak. **Where it stops:** real people can ask the friend to repeat. Wi-Fi has no way to know why a frame failed, so it simply backs off and retries.

Signs of a hidden node are high retry counts and low speed despite good signal. Fixes include moving the AP, adding an AP, reducing cell size, and RTS/CTS.

The opposite case is the **exposed node**: a device stays quiet because it hears another sender, even though sending would not hurt anyone.

### RTS/CTS

**RTS** means Request to Send. **CTS** means Clear to Send. These are tiny control frames that reserve the air before a long data frame.

1. A sends RTS to the AP, stating how long the whole exchange will take.
2. The AP replies with CTS. Everyone who hears the AP, including hidden C, reads the Duration field and sets its NAV.
3. A sends DATA while the others stay quiet.
4. The AP sends ACK, and the reservation ends.

```text
A:     [RTS]                   [---- DATA ----]
AP:           [CTS]                             [ACK]
C:            <--- NAV set: C stays quiet ---------->
       ---------------------------------------------> time
```

**Analogy:** in the doorway case, the friend shouts "A has the floor for 10 seconds". The person in the other room hears the friend, even without hearing A. **Where it stops:** RTS frames themselves can still collide, but losing a tiny frame costs far less than losing a large one.

RTS/CTS adds overhead. Many devices use it only above a size threshold, and that threshold varies by vendor, so check yours. Many APs ship with it set so high (2347 bytes is a classic value) that it is effectively off for normal frames.

### Signal strength: dBm

**dBm** is power compared to 1 milliwatt (mW), on a logarithmic scale. A logarithmic scale counts multiples, not additions.

| dBm | Power |
|---|---|
| +30 | 1000 mW (1 W) |
| +20 | 100 mW |
| +10 | 10 mW |
| 0 | 1 mW |
| -30 | 0.001 mW |
| -60 | 0.000001 mW |

Two rules cover most interview questions: **+3 dB doubles the power and +10 dB multiplies it by 10.** Going the other way, -3 dB halves it and -10 dB divides it by 10.

**Analogy:** a staircase where each step multiplies the power, instead of adding to it. **Where it stops:** dB is a ratio and dBm is an absolute level. So -60 dBm is a real power, while "10 dB" only compares two powers.

Values are negative at the receiver because signals arrive far weaker than 1 mW. Closer to zero is stronger, so -50 is stronger than -70.

| Received signal | Meaning (rule of thumb) |
|---|---|
| -30 to -50 dBm | Excellent |
| -50 to -67 dBm | Good |
| -67 dBm | Common target for voice and video |
| -67 to -75 dBm | Usable, web works |
| -75 to -85 dBm | Poor, slow and unstable |
| Below -85 dBm | Often unusable |

Signal alone is not enough. **Noise floor** is the background radio noise, often around -90 to -95 dBm. **SNR** (Signal-to-Noise Ratio) is signal minus noise. At -60 dBm signal and -92 dBm noise, SNR is 32 dB. Many designs aim for 25 dB or more.

**RSSI** (Received Signal Strength Indicator) is a device's own measure of signal. Its scale varies by vendor, so compare readings on the same device only.

### Why Wi-Fi is half duplex

**Half duplex** means a device can send or receive, but not both at once, on the same channel. A walkie-talkie works this way. Full duplex, like a phone call, allows both.

Wi-Fi is half duplex for two reasons:

1. **One channel, shared by both directions.** Sending and receiving use the same frequency, so they would collide.
2. **Self-interference.** A radio sends at about +20 dBm and may receive at about -70 dBm. That is a 90 dB gap, so its own signal is about one billion times stronger than the one it wants to hear.

The whole channel works like an old Ethernet hub: one shared collision domain. So 1 Gbps of Wi-Fi capacity is shared, and it covers both upload and download.

**Fuller truth:** MU-MIMO and OFDMA let several clients share one transmission, but the channel still carries one direction at a time. Wi-Fi 7 Multi-Link devices can use two bands, yet each channel stays half duplex.

### MIMO: more antennas, more streams

**MIMO** means Multiple Input, Multiple Output. Devices use several antennas to send different data at the same time on the same channel. Each separate flow of data is a **spatial stream**.

**Analogy:** two people speak different sentences at once from different spots. A listener with two ears can tell them apart. **Where it stops:** ears do not do this cleanly. Radios use signal paths bouncing off walls, plus heavy math, to separate streams. In open space with no reflections, it works less well.

Notation like **2x2:2** means 2 transmit antennas, 2 receive antennas and 2 spatial streams. The link uses the lower number of streams on either side, so a 4x4 AP with a 2x2 phone gets 2 streams.

| Type | Meaning | Introduced |
|---|---|---|
| SU-MIMO | Single User: many streams to one client at a time | Wi-Fi 4 |
| MU-MIMO (downlink) | Multi User: AP sends to several clients at once | Wi-Fi 5 |
| MU-MIMO (uplink) | Several clients send to the AP at once | Wi-Fi 6 |

MU-MIMO uses **beamforming**, which shapes the signal toward a client so that streams stay apart. It works best with strong signals and clients that are far apart in space.

### OFDMA: sharing one channel in smaller pieces

**OFDM** (Orthogonal Frequency Division Multiplexing) splits a channel into many tiny carrier waves called **subcarriers**. It has been used since 802.11a. With plain OFDM, one client uses the whole channel for each transmission.

**OFDMA** adds Multiple Access, and it arrived with Wi-Fi 6. The AP divides the channel into small blocks called **Resource Units (RUs)** and gives different RUs to different clients in the same transmission.

```text
OFDM (one client at a time)      OFDMA (several clients together)

freq ^ [ A ][ B ][ C ][ D ]      freq ^ [ A ][ A ][ B ]
     | [ A ][ B ][ C ][ D ]           | [ C ][ D ][ B ]
     | [ A ][ B ][ C ][ D ]           | [ C ][ D ][ E ]
     +---------------------> time     +----------------> time
```

**Analogy:** OFDM sends one truck per customer, even for one small parcel. OFDMA loads parcels for many customers on one truck. **Where it stops:** the truck in the analogy still has to visit every customer. An OFDMA frame reaches everyone in the same moment.

OFDMA helps most with many clients and small packets, such as chat, smart devices and voice. It lowers delay and wasted airtime. It does not raise the top speed of one client. In a 20 MHz channel, the smallest RU is 26 subcarriers, so up to 9 clients can share one transmission.

| | MU-MIMO | OFDMA |
|---|---|---|
| Splits by | Space (different streams) | Frequency (different RUs) |
| Best for | Large transfers, strong signal | Many small packets |
| Main gain | More total capacity | Lower latency, better efficiency |

## Example

### Example 1: Read your connection

Linux, using `iw`, the standard wireless tool:

```bash
# Linux
$ iw dev wlan0 link
Connected to 00:00:5e:00:53:a1 (on wlan0)
    SSID: HomeNet
    freq: 5180
    RX: 184213 bytes (1523 packets)
    TX: 20311 bytes (214 packets)
    signal: -52 dBm
    rx bitrate: 1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
    tx bitrate: 1200.9 MBit/s 80MHz HE-MCS 11 HE-NSS 2 HE-GI 0 HE-DCM 0
```

Windows shows the same link (fields vary by Windows version):

```powershell
# Windows
PS> netsh wlan show interfaces

    Name                   : Wi-Fi
    Physical address       : 00:00:5e:00:53:10
    State                  : connected
    SSID                   : HomeNet
    AP BSSID               : 00:00:5e:00:53:a1
    Band                   : 5 GHz
    Channel                : 36
    Radio type             : 802.11ax
    Authentication         : WPA2-Personal
    Cipher                 : CCMP
    Receive rate (Mbps)    : 1201
    Transmit rate (Mbps)   : 1201
    Signal                 : 96%
    Rssi                   : -52
```

Read it like this:

- Frequency 5180 MHz is channel 36 (5000 + 5 x 36).
- HE means High Efficiency, the technical name of Wi-Fi 6. HE-NSS 2 means 2 spatial streams.
- -52 dBm is an excellent signal.
- 1201 Mbps is the **link rate**, the raw radio speed. It is not your download speed.

A speed test to a wired server at 192.168.1.10 shows the real figure:

```bash
# Linux, tested over Wi-Fi
$ iperf3 -c 192.168.1.10 -t 10
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   830 MBytes   696 Mbits/sec   24   sender
[  5]   0.00-10.00  sec   827 MBytes   694 Mbits/sec        receiver
```

696 Mbps out of 1201 Mbps is 58%. The rest goes to waiting, ACKs, headers and retries. A common rule is to expect about half to two-thirds of the link rate in good conditions. If two laptops transfer at once, they share this capacity. See [iperf](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

### Example 2: Pick a 2.4 GHz channel

Scan your surroundings:

```bash
# Linux (NetworkManager). SIGNAL here is a percentage, not dBm.
$ nmcli -f SSID,BSSID,CHAN,FREQ,SIGNAL,SECURITY dev wifi list
SSID       BSSID              CHAN  FREQ      SIGNAL  SECURITY
HomeNet    00:00:5E:00:53:A1  36    5180 MHz  96      WPA2
HomeNet    00:00:5E:00:53:A0  1     2412 MHz  100     WPA2
Flat4B     00:00:5E:00:53:B0  1     2412 MHz  62      WPA2
Cafe-Down  00:00:5E:00:53:C0  3     2422 MHz  48      WPA2
Flat7      00:00:5E:00:53:D0  6     2437 MHz  41      WPA3
```

```powershell
# Windows: lists every BSSID nearby. Location access may need to be on.
PS> netsh wlan show networks mode=bssid
```

Now analyse it:

1. `HomeNet` 2.4 GHz is on channel 1. It shares airtime with `Flat4B` (CCI).
2. `Cafe-Down` on channel 3 overlaps channels 1 and 6, causing ACI for both.
3. Channel 11 is empty.
4. **Decision:** move `HomeNet` 2.4 GHz to channel 11 at 20 MHz width. Do not pick 9 or 10, which would overlap channel 11 users that may appear later.

### Example 3: Full join, frame by frame

A laptop with MAC `00:00:5e:00:53:10` joins `HomeNet` through AP `00:00:5e:00:53:a1`. The gateway is 192.168.1.1, and DHCP gives the laptop 192.168.1.50/24.

| # | From, to | Frame | Purpose |
|---|---|---|---|
| 1 | AP, all | Beacon | Announces SSID, channel 36, abilities |
| 2 | Laptop, all | Probe Request | Asks "is HomeNet here?" |
| 3 | AP, laptop | Probe Response | Answers with details |
| 4 | Laptop, AP | Authentication | Open System hello |
| 5 | AP, laptop | Authentication | Accepted |
| 6 | Laptop, AP | Association Request | Lists rates, 2 streams, WPA2 |
| 7 | AP, laptop | Association Response | Success, AID 1 |
| 8-11 | Both | EAPOL 1 to 4 | Creates encryption keys |
| 12-15 | Both | DHCP Discover, Offer, Request, Ack | Laptop gets 192.168.1.50/24 |
| 16-17 | Both | ARP Request, Reply | Laptop learns the MAC of 192.168.1.1 |

Each unicast frame above also gets an ACK, not shown. A failure at step 8-11 usually means a wrong password. A failure at step 12-15 means Wi-Fi is fine but DHCP is not.

### Example 4: Signal math

An AP sends at 17 dBm with a 3 dBi antenna. **dBi** is antenna gain, meaning how much the antenna focuses power. **EIRP** (Effective Isotropic Radiated Power) is the total radiated power: 17 + 3 = 20 dBm, or 100 mW.

Free-space loss over 10 metres is about 60 dB at 2.4 GHz and about 67 dB at 5 GHz:

```text
2.4 GHz:  +20 dBm - 60 dB              = -40 dBm
5 GHz:    +20 dBm - 67 dB              = -47 dBm
With one 12 dB wall on 5 GHz: -47 - 12 = -59 dBm
```

This is why the higher band loses first. The wall figure is an example, because real losses vary widely by material. Clients also often transmit more weakly than APs. So a phone can hear the AP well, while the AP struggles to hear the phone.

## Going Deeper

Optional for beginners, but useful in stronger interviews.

**Where speed numbers come from.** A link rate is calculated from these values:

```text
Rate = data subcarriers x bits per subcarrier x coding rate
       x streams / symbol time

Wi-Fi 6, 80 MHz, 2 streams, 1024-QAM, 5/6 coding, 0.8 us guard:
Rate = 980 x 10 x 5/6 x 2 / 13.6 us = about 1201 Mbps
```

**QAM** (Quadrature Amplitude Modulation) packs bits into each subcarrier: 64-QAM carries 6 bits, 256-QAM 8, 1024-QAM 10 and 4096-QAM 12. Higher QAM needs a stronger, cleaner signal. **MCS** (Modulation and Coding Scheme) is the index for each QAM and coding pair. Devices lower their MCS when signal drops, which is called rate adaptation.

**802.11 frame header.** It differs from an [Ethernet frame](../03-ethernet-and-switching/02-ethernet-frames.md) and can carry up to four addresses.

| Field | Bytes | Purpose |
|---|---|---|
| Frame Control | 2 | Type (management, control, data), To DS, From DS, Retry |
| Duration/ID | 2 | Time to reserve the air, used to set NAV |
| Address 1 | 6 | Receiver of this radio frame |
| Address 2 | 6 | Transmitter of this radio frame |
| Address 3 | 6 | Other end, such as final destination |
| Sequence Control | 2 | Frame number, finds duplicates |
| Address 4 | 6 | Only in some bridge and mesh frames |
| QoS Control | 2 | Only in QoS frames, holds traffic priority |
| HT Control | 4 | Optional, high-throughput settings |
| Body | Variable | The payload |
| FCS | 4 | Checksum to detect errors |

The laptop from Example 3 sending to gateway MAC `00:00:5e:00:53:01` uses To DS = 1:

```text
Address 1 = 00:00:5e:00:53:a1  (AP receives the radio frame)
Address 2 = 00:00:5e:00:53:10  (laptop transmits it)
Address 3 = 00:00:5e:00:53:01  (final destination, the gateway)
```

The AP then acts as a bridge. It rewrites this into a normal Ethernet frame, with the laptop as source and the gateway as destination.

**Aggregation.** Waiting for the air on every frame wastes time. Since Wi-Fi 4, **A-MPDU** joins many frames into one burst, and **Block ACK** acknowledges them together. Without this, high link rates would mean little.

**Clear Channel Assessment (CCA).** This is the physical carrier sense check. A device treats a 20 MHz channel as busy if it decodes a Wi-Fi preamble at about -82 dBm or stronger, or senses any energy at about -62 dBm or stronger.

**Other Wi-Fi 6 features.** **BSS Coloring** tags frames with a small network ID, so a device can sometimes ignore a distant network on the same channel and transmit anyway. **Target Wake Time** lets clients sleep until an agreed time, which saves battery.

**6 GHz details.** Only Wi-Fi 6E and newer devices use this band, so no slow legacy clients share it. **Preferred Scanning Channels**, every 80 MHz, reduce scan time. Power rules include low-power indoor and, in some countries, standard power controlled by an **AFC** (Automated Frequency Coordination) system. These rules differ by country.

**Wi-Fi 7 features.** **MLO** (Multi-Link Operation) lets one device use links in two or three bands together, for speed or backup. **Puncturing** skips a blocked part of a wide channel rather than giving up the whole channel.

**What comes next.** IEEE 802.11bn, expected to be branded Wi-Fi 8, was still in development in 2026. Its focus is reliability rather than higher peak speed.

## Common Mistakes

- **"The speed on the box is my speed."** That is a sum across bands and streams at ideal settings. Real throughput is far lower.
- **"Wi-Fi is switched, like Ethernet."** Clients on one channel share the air like a hub. The AP is a bridge, not a switch for the radio side.
- **"Wi-Fi uses CSMA/CD."** It uses CSMA/CA, because radios cannot detect collisions while sending.
- **"Full signal bars mean a good connection."** Bars ignore noise, retries and crowding. Check dBm, SNR and retry rate.
- **"More power fixes everything."** Higher AP power can make clients hear the AP while the AP still cannot hear them. It also adds interference for others.
- **"Wider channels are always better."** Wide channels catch more noise and raise overlap. In crowded places, narrower is often faster overall.
- **"Channel 4 avoids the crowd on 1 and 6."** It creates ACI for both. Share a clean channel instead.
- **"SSID and BSSID are the same."** SSID is the shared name. BSSID is the MAC address of one radio.
- **"Wi-Fi 6E is the same as Wi-Fi 6 with a higher speed."** Its top speed is the same. The gain is clean spectrum in 6 GHz.
- **"OFDMA makes one client faster."** It improves sharing among many clients, not peak speed.
- **"A hidden SSID is secure."** The name still appears in other frames. Use WPA2 or WPA3 instead.
- **"-80 dBm is bigger than -60 dBm, so it is stronger."** The opposite is true. -60 is 100 times stronger.

## Interview Questions

**Q: What is the difference between 802.11 and Wi-Fi?**

A: 802.11 is the IEEE standard that defines wireless LAN physical and MAC layers. Wi-Fi is the Wi-Fi Alliance brand for tested products. For example, 802.11ax is sold as Wi-Fi 6.

**Q: Compare 2.4, 5 and 6 GHz.**

A: 2.4 GHz has the best range but only three clean channels and lots of interference. 5 GHz has more channels and higher speed with moderate range. 6 GHz has the most spectrum and the least crowding, but the shortest range, and only Wi-Fi 6E or newer devices can use it.

**Q: Why do we use channels 1, 6 and 11 in 2.4 GHz?**

A: Channels are 5 MHz apart, but each signal is about 20 MHz wide, so nearby channels overlap. Channels 1, 6 and 11 are the set that do not overlap. Overlap causes adjacent-channel interference, which is worse than sharing one channel, because devices on overlapping channels cannot take turns properly.

**Q: What is the difference between SSID, BSSID and ESSID?**

A: The SSID is the network name. The BSSID is the MAC address that identifies one AP radio. The ESSID is the same name used across several APs that form one extended network, and in daily use it just means the SSID.

**Q: Walk me through how a client connects to Wi-Fi.**

A: The client scans with beacons or probes. It does 802.11 authentication, then association, where the AP gives it an Association ID. With WPA2 or WPA3 it then completes a 4-way handshake to make keys. After that it uses DHCP to get an IP and ARP to find the gateway.

**Q: Why does Wi-Fi use CSMA/CA and not CSMA/CD?**

A: A radio cannot hear while it transmits, because its own signal is far stronger than any incoming signal. A collision also happens at the receiver, so the sender may never notice it. So Wi-Fi avoids collisions with listening and random backoff, and uses ACKs to confirm delivery.

**Q: What is the hidden node problem and how does RTS/CTS help?**

A: Two clients can hear the AP but not each other, so they transmit together and collide at the AP. With RTS/CTS, the sender asks the AP first, and the AP's CTS tells every client in its range to stay quiet for a set time. It costs overhead, so it is usually used for larger frames or only when needed.

**Q: Why is Wi-Fi half duplex?**

A: Sending and receiving share one channel, and a device's own transmission is about a billion times stronger than the signal it wants to receive. So a radio cannot do both at once on one channel. All devices on the channel also share it, like an Ethernet hub.

**Q: What do dBm values like -67 mean?**

A: dBm is power relative to 1 milliwatt on a log scale. Closer to zero is stronger, +3 dB doubles power and +10 dB is ten times the power. -67 dBm is a common minimum target for voice and video, and I would also check SNR.

**Q: What is the difference between MIMO, MU-MIMO and OFDMA?**

A: MIMO uses several antennas to send multiple streams to one client. MU-MIMO sends separate streams to several clients at once, split by space. OFDMA splits a channel into frequency blocks for several clients, which helps most with many small packets.

**Q: My laptop shows a 1201 Mbps link, but a speed test shows 650 Mbps. Is something broken?**

A: Probably not. The link rate is the raw radio speed. Wi-Fi spends airtime on waiting, backoff, ACKs, headers and retries, and the channel is shared. Getting about half to two-thirds of the link rate is normal in good conditions.

**Q: What does channel width change, and why not always use the widest?**

A: Doubling width roughly doubles the speed. But it reduces the number of separate channels, increases overlap with neighbours and adds about 3 dB of noise. So crowded sites often choose 20 or 40 MHz.

## Quick Recap

- 802.11 is the standard and Wi-Fi is the brand: Wi-Fi 4 (n), 5 (ac), 6 and 6E (ax), 7 (be).
- Lower bands reach farther. Higher bands offer more spectrum and speed. Use 1, 6, 11 in 2.4 GHz.
- SSID is the name, BSSID is one radio's MAC, and ESSID is the name across many APs.
- A client joins by scan, authenticate, associate, then 4-way handshake, DHCP and ARP.
- Wi-Fi is half duplex and shared, so it uses CSMA/CA, ACKs and RTS/CTS for hidden nodes.
- dBm is logarithmic: +3 dB doubles, +10 dB is ten times, and closer to zero is stronger.
- MIMO adds streams by space, OFDMA shares a channel by frequency, and link rate is not throughput.
