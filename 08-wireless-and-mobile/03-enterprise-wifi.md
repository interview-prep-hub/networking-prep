# Enterprise Wi-Fi

**Level:** Advanced | **Prerequisites:** [Wi-Fi Basics](01-wifi-basics.md), [Wi-Fi Security](02-wifi-security.md), [VLANs](../03-ethernet-and-switching/06-vlans.md), [Trunking and 802.1Q](../03-ethernet-and-switching/07-trunking-8021q.md), [DHCP](../07-application-protocols/04-dhcp.md)

## In One Line

Enterprise Wi-Fi is many access points run as one planned system, usually through a central controller. It gives thousands of users good coverage, safe access and smooth movement across a building or campus.

## Why It Matters

A home has one Wi-Fi router and a few devices. A hospital can have 2,000 access points and 30,000 devices. The home approach fails at that size for five reasons:

- **Configuration:** you cannot log in to 2,000 devices to change one password.
- **Movement:** a nurse on a voice call walks past 20 access points. The call must not drop.
- **Shared air:** every radio on a channel takes turns. More access points can make the network slower if they are placed badly.
- **Visitors:** guests need the internet but must never reach internal servers.
- **Vague complaints:** "the Wi-Fi is bad" can mean weak signal, a full channel, a broken RADIUS server, slow DHCP or a problem that is not Wi-Fi at all.

Interviewers ask about this topic because it joins many skills: radio knowledge, VLANs, tunnels, security and structured troubleshooting. A strong answer shows you can think about the air and the wire together.

## Simple Explanation

Think of a large restaurant.

The **waiters** are the access points. They stand near the customers and do the fast work: they listen, they carry orders and they answer right away.

The **manager** is the controller. The manager does not carry plates. The manager decides which waiter covers which tables, what the rules are and who is allowed in. When a rule changes, the manager tells every waiter at once.

In a small cafe, each waiter works alone and keeps their own rule book. That is an **autonomous** access point. In a big restaurant, waiters follow one manager. That is a **lightweight** access point.

Where the analogy stops: a real waiter still knows the job if the manager leaves. A lightweight access point in the classic design stops serving new users if it loses its controller, unless it was set up for that case. Also, customer "orders" (data) often really do travel through the controller, which a restaurant manager would never do.

## How It Works

### Terms used in this chapter

| Term | Plain meaning |
| --- | --- |
| AP (access point) | The box on the ceiling. It joins Wi-Fi devices to the wired network. |
| WLC (wireless LAN controller) | The central device or software that manages many APs. |
| Client or STA (station) | Any Wi-Fi device: laptop, phone, scanner. |
| SSID | The Wi-Fi network name people see, such as CORP. |
| BSSID | The MAC address of one AP radio for one SSID. One SSID has many BSSIDs across a building. |
| ESS (extended service set) | All the APs that share one SSID and act as one network. |
| dBm | A unit of radio power. Values are usually negative. -60 dBm is stronger than -75 dBm. |
| RSSI | Received signal strength, shown in dBm. How loud the AP sounds to the client. |
| Noise floor | The background radio noise, often near -90 dBm. |
| SNR (signal-to-noise ratio) | Signal minus noise. -67 dBm with -92 dBm noise gives 25 dB. Higher is better. |
| Channel | A slice of radio frequency. Radios on one channel share it. |
| Airtime | The share of time a channel is busy. This is the real limited resource in Wi-Fi. |
| CSMA/CA | The Wi-Fi rule "listen first, send only if the air is quiet". |
| VLAN | A separate Layer 2 network on shared switches. See [VLANs](../03-ethernet-and-switching/06-vlans.md). |
| Trunk | A switch link that carries many VLANs with tags. |
| 802.1X | Login for network access, with a username or certificate. |
| RADIUS | The server that checks those logins. See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md). |
| PoE (Power over Ethernet) | Power sent through the network cable. Most APs use it. |

A quick dB rule: every 3 dB means double or half the power, and every 10 dB means ten times or one tenth.

### 1. Autonomous APs vs lightweight APs

An **autonomous AP** is a complete device. It holds its own SSIDs, security settings and channel. You manage it one box at a time.

A **lightweight AP** holds almost no lasting configuration. It boots, finds a controller and downloads its settings.

| Feature | Autonomous | Lightweight (controller-based) |
| --- | --- | --- |
| Where config lives | On each AP | On the WLC |
| Change one setting on 500 APs | 500 logins, or a script | One change |
| Channel and power tuning | Manual or per AP | Automatic across all APs |
| Roaming support | Basic | Strong, with shared key and client state |
| Client traffic | Bridged to the local switch | Usually tunnelled to the WLC, or local if chosen |
| Works without a WLC | Yes | Limited or no, depends on mode |
| Best fit | Very small sites, labs | Offices, campuses, hospitals |

A third group now matters as much as these two.

- **Cloud-managed APs:** the management system lives in the cloud (for example Cisco Meraki, Juniper Mist, HPE Aruba Central). Client traffic usually stays local. If the cloud link fails, APs keep forwarding, but you lose central management.
- **Embedded controller:** one AP runs controller software for its neighbours. Cisco's Embedded Wireless Controller is one example. It suits small sites.
- **Virtual controller:** the same controller software runs as a virtual machine. Cisco's Catalyst 9800-CL is one example.

The fuller truth: "autonomous" and "lightweight" are the classic exam words. Real products mix the ideas. A cloud AP makes more local decisions than a classic lightweight AP, but it is still centrally managed.

### 2. What the WLC does

The WLC is the brain of a controller-based network. It does these jobs:

- Holds the SSID, security and VLAN settings for all APs.
- Pushes software images to APs.
- Tunes channels and power (see RRM below).
- Tracks each client: which AP, which VLAN, which security keys.
- Talks to RADIUS. In the classic central design, the WLC is the 802.1X authenticator, not the AP.
- Joins client traffic to the wired VLANs when traffic is tunnelled.
- Hands off clients between APs and between controllers.

Common Cisco platforms today are the Catalyst 9800 family, which runs IOS XE. Older Cisco controllers ran AireOS, and you still see them in the field and in old exam material.

### 3. Split MAC: who does what

A lightweight AP and its WLC share the work of the 802.11 MAC layer. This design is called **split MAC**. The rule is simple: anything that needs a reply in microseconds stays on the AP. Anything that needs a wide view goes to the WLC.

| Job | AP (real time) | WLC (wide view) |
| --- | --- | --- |
| Send beacons | Yes | |
| Answer probes from clients | Yes | |
| Send ACKs, retransmit frames | Yes | |
| Queue frames by priority | Yes | |
| Encrypt and decrypt over the air | Yes (Cisco local mode) | |
| Accept or reject association | | Yes |
| 802.1X and RADIUS | | Yes |
| Roaming state | | Yes |
| RF decisions across APs | | Yes |
| Security and QoS policy | | Yes |

The fuller truth: the exact split varies by vendor and mode. In some tunnel designs, the controller itself does the wireless encryption. In local-switching modes, the AP takes on much more.

### 4. Client traffic paths: central vs local switching

There are two common paths for user data.

**Central switching (Cisco "local mode").** All client traffic goes through a tunnel from the AP to the WLC. The WLC puts it on the correct VLAN. The confusing name is a trap: local mode means the AP is local to the WLC, and traffic is central.

```text
 Client ))) AP ===== CAPWAP tunnel ===== WLC --- VLAN 20 --- servers
             |                            |
          access port                  trunk port
          (VLAN 30)                    (VLAN 20,30,40,100)
```

**Local switching (Cisco "FlexConnect").** The AP puts client traffic straight on its own switch port. Only control traffic goes to the WLC. This is the usual choice for branch offices, where sending all data over a WAN to head office and back would waste bandwidth.

```text
 Branch site                     WAN               Head office
 Client ))) AP --- trunk --- router ===== control ===== WLC
                      |
                 VLAN 20, local printers, local internet
```

| Question | Central switching | Local switching |
| --- | --- | --- |
| Where does data leave? | At the WLC | At the AP's switch port |
| AP switch port | Access port | Usually a trunk |
| VLANs needed at the edge | Only the AP VLAN | Every client VLAN |
| WAN use | High if the WLC is remote | Low |
| Single point of policy | Yes | Less so |
| If the WLC link fails | Service stops | Existing clients usually stay; new logins depend on settings |

Other AP modes you should know by name:

| Mode (Cisco names) | Purpose |
| --- | --- |
| Local | Normal service, data tunnelled to the WLC |
| FlexConnect | Service at a remote site, data switched locally |
| Monitor | No clients. The radio only listens, for rogue APs and location. |
| Sniffer | Captures frames on one channel and sends them to a packet analyser |
| Bridge or Flex+Bridge | Mesh operation |

### 5. CAPWAP

**CAPWAP** means Control And Provisioning of Wireless Access Points. It is the standard tunnel protocol between an AP and a WLC. RFC 5415 defines the protocol and RFC 5416 defines its use with 802.11. It grew out of Cisco's older LWAPP protocol.

Analogy: CAPWAP is like the pneumatic tube in an old bank. The teller puts a sealed capsule in the tube and it arrives at the back office. The building's walls (the routers between) never open it. Where the analogy stops: a CAPWAP tunnel is not private by default for user data. See below.

CAPWAP has two channels:

| Channel | Port | Carries | Protected by |
| --- | --- | --- | --- |
| Control | UDP 5246 | Join, config, image updates, keepalives | DTLS (TLS for datagrams) |
| Data | UDP 5247 | Client frames, data keepalives | DTLS is optional, and often off by default |

That last cell matters. The wireless encryption (WPA2 or WPA3) only protects the radio path. In a common Cisco setup, the AP decrypts the frame and sends it inside CAPWAP without DTLS. Anyone with access to that wired path can read the client data unless data DTLS is on or the application uses TLS.

The tunnel can cross routers, so AP and WLC can live in different subnets. Each client packet gets wrapped like this:

```text
+----------+----------+------+--------+----------+-----------+
| Outer L2 | Outer IP | UDP  | CAPWAP | Client   | Client IP |
| Ethernet | AP - WLC | 5247 | header | L2 frame | packet    |
+----------+----------+------+--------+----------+-----------+
```

#### How a lightweight AP joins a WLC

1. **Power and IP.** The AP boots from PoE and gets an IP address, usually by DHCP.
2. **Build a controller list.** The AP collects WLC addresses from several sources:
   - Controllers it remembers from before.
   - DHCP option 43, a vendor-specific option that carries WLC addresses.
   - A DNS lookup. Cisco APs look for `CISCO-CAPWAP-CONTROLLER.<local-domain>`.
   - A local subnet broadcast, which only works if a WLC is on the same subnet.
   - Addresses set by hand on the AP.
3. **Discovery.** The AP sends Discovery Requests. Each WLC replies with a Discovery Response.
4. **Pick a WLC.** The AP prefers a configured primary, then looks at load.
5. **DTLS handshake.** AP and WLC prove who they are with certificates.
6. **Join.** The AP sends a Join Request. The WLC sends a Join Response.
7. **Image check.** If the software version differs, the AP downloads the image and reboots.
8. **Configure.** The AP receives its SSIDs, channels and policy.
9. **Run.** The AP starts beaconing. It sends Echo keepalives so that both sides can detect a dead link. RFC 5415 sets the default Echo interval to 30 seconds.

```text
 AP                                                  WLC
 |---- Discovery Request (UDP 5246) ------------------>|
 |<--- Discovery Response -----------------------------|
 |<=== DTLS handshake (certificates) =================>|
 |---- Join Request ---------------------------------->|
 |<--- Join Response ----------------------------------|
 |---- Image Data, only if versions differ ----------->|
 |---- Configuration Status Request ------------------>|
 |<--- Configuration Status Response ------------------|
 |---- Change State Event, Data keepalive (5247) ----->|
 |<=== Echo keepalives, client traffic ===============>|
```

Vendors differ here too. CAPWAP is an open standard, but not every vendor uses it. HPE Aruba, for example, has long used its own control protocol with GRE tunnels.

### 6. Roaming

**Roaming** means a client moves from one AP to another without leaving the network. The key fact: **the client decides when to roam**, not the AP and not the WLC. The network can only give hints.

A client usually starts looking for a better AP when its signal falls. Many phones begin near -70 to -75 dBm. This varies by device and driver, and vendors tune it.

Steps in a plain roam:

1. The client's signal gets weaker.
2. The client scans for other APs with the same SSID.
3. The client picks a target.
4. The client sends Authentication and Reassociation frames to the new AP.
5. Security runs again: full 802.1X and a 4-way handshake, unless something speeds it up.
6. The network updates where the client is. Traffic now flows through the new AP.

The slow parts are scanning and 802.1X. A full 802.1X login can take hundreds of milliseconds, or more if RADIUS is far away. A common target for voice is a gap under 50 ms. That is an industry rule of thumb, not a standard.

Two common failure types:

- **Sticky client:** the client stays on a far AP, even when a closer one is available. It talks slowly and takes extra airtime from everyone.
- **Ping-pong:** the client flips between two APs of similar strength.

#### Layer 2 and Layer 3 roaming

| Type | What happens | Does the client's IP change? |
| --- | --- | --- |
| Intra-controller | Both APs use the same WLC. The WLC updates its table. | No |
| Inter-controller Layer 2 | New WLC also has the client's VLAN. Client record moves. | No |
| Inter-controller Layer 3 | New WLC does not have the client's VLAN. A tunnel keeps the old subnet. | No |

Layer 3 roaming uses two roles. The **anchor** WLC is where the client first joined. It stays the client's home. The **foreign** WLC is where the client is now. Traffic flows through a mobility tunnel between them. Controllers that may do this are placed in a **mobility group**.

```text
            VLAN 20 (client's home subnet)
                  |
           +-------------+  mobility tunnel  +-------------+
           | WLC-A       |===================| WLC-B       |
           | anchor      |  UDP 16666/16667  | foreign     |
           +-------------+                   +-------------+
                  |                                 |
            AP in Building A                  AP in Building B
                                                    )))
                                                  client
```

Analogy: this is like mail forwarding after you move house. Letters still go to the old address first, then get sent on. Where the analogy stops: the forward stops when the client leaves the network, and there is no paperwork. The WLCs do it in the background.

On Cisco's mobility design, UDP 16666 carries control and UDP 16667 carries data between controllers.

### 7. 802.11k, 802.11v and 802.11r

These three amendments make roaming faster and smarter. All three are now part of the base 802.11 standard, but people still use the letters.

Analogy: you are walking through an airport with many help desks.

- **802.11k** gives you a printed list of nearby desks, so you do not have to look down every hallway.
- **802.11v** is a staff member who says, "Please go to desk 5, this one is full." You may ignore them.
- **802.11r** is a pre-approved pass, so you do not show your passport again at the next desk.

Where the analogy stops: these are radio frames and keys, and each feature only works if both the client and the AP support it.

| Amendment | Name | What it does | Which roam step it improves |
| --- | --- | --- | --- |
| 802.11k | Radio Resource Measurement | AP gives a Neighbor Report: nearby APs and channels | Scanning |
| 802.11v | Wireless Network Management | AP can send a BSS Transition Management (BTM) request | Choosing a target |
| 802.11r | Fast BSS Transition (FT) | Security keys are ready before the client arrives | Authentication |

**802.11k in detail.** Without it, a client scans many channels. On 5 GHz that is slow, and scanning DFS channels is slower because the client must listen quietly first. With 802.11k, the client asks its AP for a list and scans only a few channels.

**802.11v in detail.** BTM lets the network say, "Please move, and here are better APs." The client can accept, delay or refuse. A BTM request can include a warning that the AP will soon disconnect the client. It is also useful for moving clients away from a full AP. 802.11v covers other features too, such as sleep-power helpers, but BTM is the one interviewers ask about.

**802.11r in detail.** After the first full login, key material is shared ahead of time across a **mobility domain**, a group of APs. When the client roams, four frames replace the separate EAP login and 4-way handshake.

```text
 Client                                   Target AP
   |--- Authentication (FT, client nonce) ---->|
   |<-- Authentication (FT, AP nonce) ---------|
   |--- Reassociation Request (with MIC) ----->|
   |<-- Reassociation Response (with group key)|
   |========= protected data resumes =========>|
```

This form is **FT over the air**. There is also **FT over the DS**, where the client talks to the target through its current AP. Over the air is the usual choice today.

Roaming speed-up methods compared:

| Method | Standard? | What it skips | Client support |
| --- | --- | --- | --- |
| Full 802.1X | Yes | Nothing | All |
| PMK caching | Yes | EAP, when returning to an AP used before | Wide |
| OKC (Opportunistic Key Caching) | No, a common industry method | EAP, on a first visit to an AP | Varies by OS |
| 802.11r FT | Yes | EAP and the separate 4-way handshake | Most modern clients; old ones can fail |
| CCKM | Cisco legacy | EAP on supported Cisco-compatible clients | Legacy only |

Warning: some older clients cannot even see an SSID that advertises only FT, and some have driver bugs with it. Test real devices before you enable 802.11r on a shared SSID.

### 8. Mesh

In a **wireless mesh**, some APs have no network cable. They reach the wired network through other APs by radio. The radio link between APs is called the **backhaul**.

- **RAP (root AP):** has a wired connection.
- **MAP (mesh AP):** has only power, and uses a radio link to a parent.
- **Hop:** one radio jump between APs.

```text
                 wired LAN
                     |
                +---------+
                |   RAP   |  root AP, has a cable
                +---------+
                 ))     ((    5 GHz backhaul
        +---------+     +---------+
        |  MAP-1  |     |  MAP-2  |  1 hop
        +---------+     +---------+
             ))
        +---------+
        |  MAP-3  |                  2 hops
        +---------+
```

Analogy: a line of people passing buckets of water. A longer line works, but each person can only hold one bucket at a time. Where the analogy stops: radios can also interfere with each other across the whole line, not only with the next person.

Cisco mesh APs choose a path with AWPP (Adaptive Wireless Path Protocol). 802.11s is the IEEE mesh standard. The Wi-Fi Alliance's EasyMesh targets homes. These do not usually work with each other.

Trade-offs:

- **Each hop costs throughput.** On a single shared radio, a simple model halves the rate each hop: 400 Mbps at the RAP becomes about 200 Mbps at hop 1 and about 100 Mbps at hop 2.
- **Each hop adds latency and jitter.**
- **The radio link can change.** Weather, doors and DFS radar events can break a path.
- **Keep it short.** Vendors allow several hops, but a common rule of thumb is two or three.

The fuller truth: a dedicated radio for backhaul avoids much of the halving, because client traffic and backhaul use different channels. Tri-band APs do this.

Use mesh where cable is impossible: parking areas, old stone buildings, outdoor yards, temporary sites. Use a cable anywhere you can. A wire is always better than a backhaul.

### 9. Site survey

A **site survey** is the study of how radio signals behave in a real building. It answers: where do APs go, how many, and what channels and power?

Analogy: before you buy lamps for a new building, you test how light moves through the rooms. Walls block it, glass passes it and mirrors bounce it. Where the analogy stops: radio goes through walls, and you can hear far weaker signals than you can see. So a wall that "blocks" 15 dB still leaks.

#### Survey types

| Type | How it is done | When |
| --- | --- | --- |
| Predictive | Software models the floor plan and walls | Before buying hardware |
| AP-on-a-stick | Temporary AP on a pole, measure real signal | Before install, in hard buildings |
| Passive | Walk with a survey tool, only listening | After install, to check coverage and interference |
| Active | Client joins the network and tests throughput and roaming | After install |
| Spectrum analysis | Special hardware sees all RF, not only Wi-Fi | When non-Wi-Fi interference is suspected |

Common survey tools include Ekahau and Hamina.

#### Survey process

1. **Gather requirements.** How many users and devices? Which apps: voice, video, scanners? Which areas matter most? Which device has the worst radio? Design for that one.
2. **Get floor plans.** Scale them. Mark wall types.
3. **Predict.** Model walls and place APs.
4. **Validate on site.** Walk the building. Look for lifts, metal shelves, thick walls and fish tanks.
5. **Install.**
6. **Post-install survey.** Measure again. Fix the real gaps.
7. **Document.** Save the heat maps so that later changes can be compared.

Rough planning losses at 5 GHz. Real values depend on thickness and material, so measure when it matters:

| Material | Rough loss |
| --- | --- |
| Drywall | 3-4 dB |
| Clear glass | 2-4 dB |
| Brick | 6-10 dB |
| Concrete | 10-20 dB |
| Metal, lift doors | 20 dB or more |

Low-E (coated, energy-saving) glass can block far more than clear glass. Loss grows as frequency rises, so 6 GHz loses more than 5 GHz, and 5 GHz loses more than 2.4 GHz.

Common design targets. These are rules of thumb from vendor design guides, not standards:

| Metric | Data | Voice and real-time |
| --- | --- | --- |
| Signal at the client | -70 dBm or better | -67 dBm or better |
| Second-best AP heard | Helpful | -70 to -67 dBm, for roaming |
| SNR | 20 dB or more | 25 dB or more |
| Channel utilisation | Below about 50-60% | Lower is better |
| Retry rate | Below about 15% | Below about 10% |

#### Coverage design vs capacity design

**Coverage design** asks: does every spot get a usable signal? **Capacity design** asks: can the air carry everyone at once?

A lecture hall with 300 students can be "covered" by one AP. It cannot be served by one. High-density design uses more radios, lower power, narrower channels and directional antennas, so that each radio serves a small zone.

### 10. Channel planning

Analogy: a big hall holds many separate conversations. Two groups in one corner speaking at the same time means nobody can follow. You spread the groups out and ask them to speak quietly. Where the analogy stops: Wi-Fi radios on one channel do not talk over each other on purpose. They wait their turn, so the channel slows instead of turning to noise.

Two kinds of interference:

- **CCI (co-channel interference):** two nearby radios on the same channel. They hear each other, so they take turns. Result: less airtime for each. This is really contention, and it is the most common problem in busy networks.
- **ACI (adjacent-channel interference):** two radios on partly overlapping channels. They do not wait for each other, so frames collide and need retries. This is usually worse. It is why 2.4 GHz channels 1 and 3 is a bad plan.

Channel choices by band:

| Band | Channels to plan with | Usual width | Notes |
| --- | --- | --- | --- |
| 2.4 GHz | 1, 6, 11 | 20 MHz | Only three clean channels. Never use 40 MHz in an enterprise. |
| 5 GHz (US) | 25 at 20 MHz, 12 at 40 MHz, 6 at 80 MHz | 20 or 40 MHz in offices | Many channels need DFS |
| 6 GHz (US) | 59 at 20 MHz, 14 at 80 MHz, 3 at 320 MHz | 80 MHz is common | Needs WPA3 or OWE |
| 6 GHz (EU lower band) | 24 at 20 MHz, 6 at 80 MHz | 80 MHz is common | Less spectrum than the US |

Channel rules differ by country. Always check the local rules. Set the correct country on the controller.

**DFS (Dynamic Frequency Selection)** applies to parts of 5 GHz that radar also uses. In the US, those are channels 52-144. An AP must check for radar before using the channel, and must leave within a short time if it hears radar. The FCC rules use a 60-second check, a 10-second move time and a 30-minute stay-away time. Near airports or ports, test DFS channels before you trust them.

Main rules for a channel plan:

1. Same channel, as far apart as possible, including on the floor above and below.
2. Narrower channels give you more channels to reuse. In a dense site, 20 or 40 MHz usually beats 80 MHz on 5 GHz.
3. Do not raise AP power to fix coverage for clients. An AP at high power may reach a phone, but the phone may not reach the AP. Phones usually transmit at much lower power than an AP can. Keep the link balanced.
4. Turn off some 2.4 GHz radios. With three channels, you rarely need all of them. 2.4 GHz travels further, so it overlaps more.
5. Disable low data rates. Removing 1, 2, 5.5 and 11 Mbps stops old slow frames and shrinks the cell. Old 802.11b devices then cannot join.
6. Limit SSIDs. Three to five per radio is a common limit, because each one sends beacons.

### 11. RRM

**RRM (Radio Resource Management)** is the system that watches the air and tunes APs automatically. RRM is Cisco's name. Other vendors use their own: HPE Aruba has AirMatch and earlier ARM, and Meraki has Auto RF.

Analogy: a thermostat. It measures, compares against a target and adjusts. Where the analogy stops: a thermostat cannot move a window. RRM cannot fix bad AP placement, and it cannot create more air.

Steps:

1. **Measure.** APs briefly go off-channel to listen. They hear neighbours, noise, interference and rogue APs.
2. **Report.** APs send this data to the WLC.
3. **Decide.** The WLC finds the best plan for the group.
4. **Apply.** The WLC changes channels and power.

| Function | Cisco name | Job |
| --- | --- | --- |
| Channel selection | DCA (Dynamic Channel Assignment) | Picks channels to reduce interference |
| Power selection | TPC (Transmit Power Control) | Lowers power when neighbours are close, raises it when they are far |
| Hole repair | CHD (Coverage Hole Detection) | Raises power when many clients report weak signal |
| Width selection | DBS (Dynamic Bandwidth Selection) | Picks channel width from what clients use |
| Radio role | FRA (Flexible Radio Assignment) | Moves a dual-band radio between roles when 2.4 GHz is overserved |

Good practice:

- Cisco's classic TPC needs at least four APs that hear each other to work well.
- A channel change disconnects or disturbs clients briefly. The default Cisco DCA interval is short, and many engineers change it to run at night. Exact settings vary by version.
- Run a survey first. RRM tunes a good design. It does not rescue a bad one.
- Newer systems use long-term data to plan more carefully.

Cisco 9800 verification:

```text
WLC# show ap dot11 5ghz summary
WLC# show ap auto-rf dot11 5ghz
```

### 12. Guest networks

Analogy: a hotel lobby. Visitors can sit, use the toilets and get coffee. They cannot walk into the staff offices or the safe room. Where the analogy stops: on Wi-Fi, one guest can also attack another guest, so you must separate visitors from each other too.

A good guest design:

1. **Separate SSID**, such as GUEST.
2. **Separate VLAN**, such as VLAN 40 with 172.16.40.0/23. This gives 510 usable addresses. Use a short DHCP lease, such as 1-4 hours.
3. **Filter by default.** Block all private ranges. Allow DHCP, DNS and the internet.
4. **Client isolation.** Stop guest devices talking to each other on the AP and WLC.
5. **Login.** Use a captive portal, a daily password or an account from a sponsor.
6. **Rate limit.** Stop one guest from filling the channel. For example, 5 Mbps per device.
7. **Terms and logging.** Show the rules. Keep the logs your local law requires.

**Captive portal** means the web page that appears before you can use the network. Here is how it works:

1. The guest joins GUEST and gets an IP address by DHCP.
2. The device tests the internet by requesting a known web address. Apple, Google and Microsoft each run their own test URL.
3. The WLC catches the request and redirects it to the portal.
4. The guest accepts the terms or logs in.
5. The WLC moves the client to the run state and allows traffic.

An HTTPS request cannot be silently redirected without a certificate warning. That is why portals depend on those plain HTTP test URLs. RFC 8910 gives a better way: DHCP option 114 and an IPv6 RA option tell the client the portal address directly. RFC 8908 defines an API that the client uses to check its status.

An open guest SSID has no over-the-air encryption. **OWE (Opportunistic Wireless Encryption, RFC 8110)**, sold as Wi-Fi Enhanced Open, fixes this. It encrypts traffic with no password. It does not prove who the network is. See [Wi-Fi Security](02-wifi-security.md).

**Guest anchor design.** Large sites do not want guest traffic loose inside the network. The internal WLC sends guest traffic through a mobility tunnel to an **anchor WLC** in the DMZ (a separate network zone at the edge). Guest traffic only comes out there.

```text
 Guest ))) AP === CAPWAP === Internal WLC
                                  ||
                           mobility tunnel
                                  ||
              Firewall --- Anchor WLC (DMZ) --- Firewall --- Internet
```

Cisco's enterprise version often uses Cisco ISE (a RADIUS and policy server) for the portal. After login, ISE tells the WLC to change the client's rights with a RADIUS Change of Authorization message (RFC 5176).

### 13. Troubleshooting poor Wi-Fi

"Poor Wi-Fi" is not a diagnosis. Use the general method from [Troubleshooting Methodology](../12-troubleshooting-and-tools/01-troubleshooting-method.md), with these Wi-Fi steps.

#### Step 1: Define the scope

- One user, one area or everyone?
- One device type or all?
- Always, at certain times or only when moving?
- What changed recently?

The scope tells you where to look. One device means a client or driver problem. One area means RF or one AP. Everyone means a central service: WLC, RADIUS, DHCP or a wired link.

#### Step 2: Find how far the client got

A client climbs these steps in order. Find the last one that worked.

```text
 Scan --> 802.11 auth --> Associate --> 802.1X/EAP --> 4-way key
                                                          |
   App <-- Gateway and DNS <-- DHCP <---------------------+
```

- Cannot see SSID: radio off, wrong band, SSID not on this AP, client too old.
- Associates, then fails login: RADIUS, certificate or password.
- Login works, no IP: DHCP or VLAN not on a trunk.
- IP works, no internet: gateway, DNS, firewall or ACL.

#### Step 3: Read the radio from the client

**Windows:**

```powershell
# Current AP, band, channel, signal and rate
netsh wlan show interfaces

# All nearby APs for each SSID
netsh wlan show networks mode=bssid

# Detailed history report as an HTML file (run as Administrator)
netsh wlan show wlanreport
```

**Linux:**

```bash
# Current AP, signal and rate
iw dev wlan0 link

# All nearby APs
sudo iw dev wlan0 scan | grep -E "^BSS|signal:|SSID:"
```

**macOS:** hold Option and click the Wi-Fi icon, or run `sudo wdutil info`.

Ask these questions:

- Is RSSI better than -67 dBm?
- Is SNR 25 dB or more?
- Is the client on the nearest AP, or is it sticky?
- Is it on 5 or 6 GHz, or stuck on 2.4 GHz?

#### Step 4: Check the air

**Channel utilisation** is the share of time the channel is busy. If it is high, adding "signal" will not help. Look for too many clients, too many SSIDs, neighbouring networks, low data rates or wide channels with heavy overlap.

**Retry rate** is the share of frames sent again. High retries point to interference, weak signal or **hidden nodes** (two clients that cannot hear each other, so they both send to the AP at once).

Non-Wi-Fi noise, like microwave ovens, Bluetooth, wireless cameras and old cordless phones, needs a spectrum tool. Cisco's CleanAir feature can detect many of these from the AP.

#### Step 5: Check the controller

```text
WLC# show ap summary
WLC# show wlan summary
WLC# show wireless client summary
WLC# show wireless client mac-address 3c58.c2a1.7e10 detail
WLC# show wireless mobility summary
WLC# show wireless stats ap join summary
```

For a deep client trace, Cisco 9800 has Radioactive Trace (`debug wireless mac <client-mac>`), which records one client's full story across the WLC's processes.

#### Step 6: Prove whether the problem is Wi-Fi

Test the wire and the air apart. Put a laptop on a cable beside the AP. If the app is still slow, Wi-Fi is innocent. Use `iperf3` to a local server, as shown in [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

#### Common symptoms

| Symptom | Likely cause | Check | Fix |
| --- | --- | --- | --- |
| Slow in one room at busy times | Full channel | Utilisation, clients per AP | More radios, narrower channels, less power |
| Calls drop when walking | Slow roam, coverage gap | RSSI overlap, roam logs | Add coverage, enable k/v/r, tune thresholds |
| Device stays on far AP | Sticky client | Client RSSI and BSSID | Lower AP power, disable low rates, BTM |
| Connects, no IP | VLAN missing on trunk, DHCP pool full | Trunk allowed list, DHCP leases | Add VLAN, enlarge pool |
| Login fails for all | RADIUS down, certificate expired | RADIUS logs, WLC reachability | Fix server, renew certificate |
| Login fails for one user | Wrong password, profile, clock | RADIUS reject reason | Correct user or device |
| AP will not join | No WLC address, firewall, bad time, country | AP console, join stats | Fix DHCP option 43, ports, NTP, country |
| 2.4 GHz is unusable | Crowded, non-Wi-Fi noise | Spectrum tool | Move clients to 5 or 6 GHz |
| Only one laptop model fails | Driver | Compare with another model | Update driver, check 802.11r and PMF |
| Fine on speed test, video choppy | QoS, jitter | Retry rate, WMM, DSCP | See QoS section in Going Deeper |

## Example

### Example 1: Design one office floor

Requirements:

- Floor: 60 m x 30 m = 1,800 m2, drywall inside.
- Staff: 150 people, each with a laptop and a phone: 300 devices.
- Peak: 70% active at the same time: 210 active devices.
- Peak traffic: 2 Mbps per active device: 420 Mbps.

**Step 1: Capacity.** Planning assumption: a 5 GHz radio on 40 MHz gives 200 Mbps of usable shared throughput. This is deliberately cautious.

```text
Throughput:  420 Mbps / 200 Mbps per radio   = 2.1  -> 3 radios
Clients:     210 active / 40 per radio       = 5.25 -> 6 radios
```

**Step 2: Coverage.** Assumption: one AP covers about 225 m2 at -67 dBm in this drywall office. The survey will prove it.

```text
Coverage:    1800 m2 / 225 m2 per AP         = 8 APs
```

**Step 3: Take the biggest number.** Coverage wins: 8 APs. Each gets about 26 active clients and 53 Mbps at peak. This leaves room to grow.

**Step 4: Channel plan.** Use four non-DFS 40 MHz channels (primary 36, 44, 149, 157). This example uses US rules.

```text
+---------------------------------------------------------------+
|  AP-FL3-01     AP-FL3-02     AP-FL3-03     AP-FL3-04          |
|  5G ch 36      5G ch 44      5G ch 149     5G ch 157          |
|  2.4G ch 1     2.4G off      2.4G ch 6     2.4G off           |
|                                                               |
|  AP-FL3-05     AP-FL3-06     AP-FL3-07     AP-FL3-08          |
|  5G ch 149     5G ch 157     5G ch 36      5G ch 44           |
|  2.4G off      2.4G ch 11    2.4G off      2.4G ch 1          |
+---------------------------------------------------------------+
```

Same-channel APs are two columns apart. The floors above and below shift the pattern so that the same channel is not directly on top. If you buy Wi-Fi 6E or Wi-Fi 7 APs, add a 6 GHz plan as well.

**Step 5: Addressing.**

| Purpose | VLAN | Subnet | Gateway |
| --- | --- | --- | --- |
| AP management | 30 | 10.20.30.0/24 | 10.20.30.1 |
| Corporate clients (SSID CORP) | 20 | 10.20.0.0/22 | 10.20.0.1 |
| Guest (SSID GUEST) | 40 | 172.16.40.0/23 | 172.16.40.1 |
| Servers and WLC | 100 | 10.10.100.0/24 | 10.10.100.1 |

Servers: WLC 10.10.100.5, DNS 10.10.100.10, RADIUS 10.10.100.20, DHCP 10.10.100.30.

**Step 6: Power.** Each AP is planned at 30 W (802.3at class 4). That is 240 W for eight APs. The floor switch has a 370 W PoE budget, leaving 130 W spare.

### Example 2: Join an AP to the WLC and publish SSIDs

Switch port for a local-mode AP (Cisco IOS):

```text
interface GigabitEthernet1/0/11
 description AP-FL3-01
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
```

A FlexConnect AP that switches VLANs 20 and 40 locally would use a trunk instead:

```text
interface GigabitEthernet1/0/24
 description AP-BRANCH-01 FlexConnect
 switchport mode trunk
 switchport trunk native vlan 30
 switchport trunk allowed vlan 20,30,40
 spanning-tree portfast trunk
```

DHCP scope for the APs, with option 43 (Cisco IOS):

```text
ip dhcp pool AP-MGMT
 network 10.20.30.0 255.255.255.0
 default-router 10.20.30.1
 dns-server 10.10.100.10
 option 43 hex f104.0a0a.6405
```

Decode that hex:

| Bytes | Meaning |
| --- | --- |
| f1 | Type for Cisco lightweight APs |
| 04 | Length: 4 bytes, one WLC address |
| 0a 0a 64 05 | 10.10.100.5 |

Check the tunnel from a Linux box on a SPAN port (a switch port that copies traffic):

```bash
sudo tcpdump -ni eth0 'udp port 5246 or udp port 5247'
```

```text
12:04:10.112233 IP 10.20.30.11.51234 > 10.10.100.5.5246: UDP, length 96
12:04:10.113871 IP 10.10.100.5.5246 > 10.20.30.11.51234: UDP, length 112
12:04:19.402116 IP 10.20.30.11.51234 > 10.10.100.5.5247: UDP, length 72
```

WLAN and policy on a Catalyst 9800. A WLAN holds the SSID and security. A policy profile holds the VLAN. A policy tag joins the two. The AP receives the tag. This is shortened: a full setup needs more items, such as guest web-auth settings.

```text
aaa new-model
radius server ISE-1
 address ipv4 10.10.100.20 auth-port 1812 acct-port 1813
 key Example-Shared-Secret
aaa group server radius ISE-GROUP
 server name ISE-1
aaa authentication dot1x CORP-AUTH group ISE-GROUP

wlan CORP 1 CORP
 security dot1x authentication-list CORP-AUTH
 security ft
 security wpa akm ft dot1x
 no shutdown

wireless profile policy CORP-POLICY
 vlan 20
 no shutdown

wireless tag policy HQ-POLICY
 wlan CORP policy CORP-POLICY

ap 7069.5a1c.3300
 policy-tag HQ-POLICY
```

This WLAN keeps both normal 802.1X and FT 802.1X. Clients with FT use the fast path. Older clients fall back to normal login. Test this with your oldest devices.

Now view the result (columns shortened to fit):

```text
WLC# show ap summary
Number of APs: 8

AP Name     Slots AP Model     Radio MAC      State      IP Address
-------------------------------------------------------------------
AP-FL3-01   2     C9120AXI-B   706d.153a.4b00 Registered 10.20.30.11
AP-FL3-02   2     C9120AXI-B   706d.153a.4c20 Registered 10.20.30.12
AP-FL3-03   2     C9120AXI-B   706d.153a.4d40 Registered 10.20.30.13
...
```

### Example 3: The airtime cost of too many SSIDs

An AP sends one beacon per SSID about every 102.4 ms. That is 9.77 beacons per second per SSID. Assume a 380-byte beacon. Ignore waiting time for simplicity.

**At 1 Mbps (old default on 2.4 GHz):**

```text
Beacon time  = 380 x 8 / 1 Mbps + 192 us preamble
             = 3040 us + 192 us = about 3.2 ms
8 SSIDs      = 8 x 9.77 = 78 beacons per second
Airtime      = 78 x 3.2 ms = about 250 ms per second = 25%
```

**At 6 Mbps (low rates disabled):**

```text
Beacon time  = 380 x 8 / 6 Mbps + 20 us preamble
             = 507 us + 20 us = about 0.53 ms
Airtime      = 78 x 0.53 ms = about 41 ms per second = about 4%
```

One AP with 8 SSIDs at 1 Mbps uses a quarter of the air before any user sends data. Nearby APs on the same channel add theirs too. This is why removing low rates and extra SSIDs gives such a big gain for no money.

### Example 4: "Wi-Fi is slow in Room 3A"

This uses the floor from Example 1.

**Report:** at 14:00, people in Room 3A say video calls freeze. Room 3A is next to AP-FL3-03. Other areas are mostly fine. This morning, it was fine.

**Step 1: Read a client.**

```powershell
netsh wlan show interfaces
```

```text
    Name                   : Wi-Fi
    State                  : connected
    SSID                   : CORP
    AP BSSID               : 70:6d:15:3a:4c:2f
    Band                   : 5 GHz
    Channel                : 44
    Radio type             : 802.11ax
    Authentication         : WPA2-Enterprise
    Cipher                 : CCMP
    Receive rate (Mbps)    : 137.6
    Transmit rate (Mbps)   : 137.6
    Signal                 : 58%
    Rssi                   : -71
```

The client should be on AP-FL3-03 (channel 149). Instead it is on channel 44, which belongs to AP-FL3-02, one room further away. RSSI of -71 dBm is weak for video.

**Step 2: Ask the WLC about this client.**

```text
WLC# show wireless client mac-address 3c58.c2a1.7e10 detail
Client MAC Address : 3c58.c2a1.7e10
Client IPv4 Address : 10.20.1.57
AP Name : AP-FL3-02
Wireless LAN Name : CORP
Channel : 44
Vlan : 20
Policy Manager State : Run
Radio Signal Strength Indicator : -71 dBm
Signal to Noise Ratio : 21 dB
```

The login, IP and VLAN are fine. It is a radio problem. SNR 21 dB is below the 25 dB target.

**Step 3: Check that AP's load.**

```text
WLC# show ap name AP-FL3-02 auto-rf dot11 5ghz
AP Name                       : AP-FL3-02
Load Information
  Load Profile                : Failed
  Receive Utilization         : 28 %
  Transmit Utilization        : 41 %
  Channel Utilization         : 84 %
  Attached Clients            : 61 clients
Noise Information
  Channel 44                  : -92 dBm
Nearby APs
  AP-FL3-01  slot 1 : -66 dBm on (36, 40 MHz)
  AP-FL3-06  slot 1 : -64 dBm on (157, 40 MHz)
  AP-FL3-08  slot 1 : -84 dBm on (44, 40 MHz)
```

The channel is 84% busy, with 61 clients where the plan expected about 26. The noise is normal, so this is load, not interference. The AP's own traffic is 69% (28 + 41). The other 15% is other radios on channel 44. AP-FL3-03 is missing from the nearby list.

**Step 4: Look for the missing AP.**

```text
WLC# show ap summary
Number of APs: 7

AP Name     Slots AP Model     Radio MAC      State      IP Address
-------------------------------------------------------------------
AP-FL3-01   2     C9120AXI-B   706d.153a.4b00 Registered 10.20.30.11
AP-FL3-02   2     C9120AXI-B   706d.153a.4c20 Registered 10.20.30.12
AP-FL3-04   2     C9120AXI-B   706d.153a.4e60 Registered 10.20.30.14
...
```

Only 7 of 8 APs are registered. AP-FL3-03 is gone.

**Step 5: Check its switch port.**

```text
SW3# show power inline
Module   Available     Used     Remaining
          (Watts)     (Watts)    (Watts)
------   ---------   --------   ---------
1           370.0      364.0        6.0

Interface  Admin  Oper       Power   Device     Class  Max
---------- ------ ---------- ------- ---------- -----  ----
Gi1/0/11   auto   on         30.0    C9120AXI-B 4      30.0
Gi1/0/12   auto   on         30.0    C9120AXI-B 4      30.0
Gi1/0/13   auto   denied     0.0     n/a        n/a    30.0
```

**Root cause.** At noon, a contractor added 10 PoE cameras to this switch, each allocated 15.4 W. That is 154 W. A ceiling repair unplugged AP-FL3-03 at the same time. When it was plugged back in:

```text
7 APs at 30 W           = 210.0 W
10 cameras at 15.4 W    = 154.0 W
Used                    = 364.0 W
Remaining of 370 W      =   6.0 W, but the AP needs 30 W
```

The switch denied power. All Room 3A users moved to the next AP, and it filled up. Nothing was wrong with Wi-Fi settings. The cause was on the wire.

**Fix.** Move five cameras to another PoE switch. That frees 77 W. The AP gets 30 W and joins.

**Verify.**

```text
Client: AP-FL3-03, channel 149, RSSI -58 dBm, rate 458.8 Mbps
AP-FL3-02: Channel Utilization 31 %, Attached Clients 27 clients
```

**Prevent.** Add a monitoring alert for AP down, and track PoE budget before adding devices. Watch this with [Network Monitoring and Observability](../12-troubleshooting-and-tools/11-network-monitoring.md).

## Going Deeper

### CAPWAP header

The CAPWAP header is at least 8 bytes.

```text
 0               8        13       18       23           31
+---------------+--------+--------+--------+-+-+-+-+-+-+-----+
| Preamble (8)  |HLEN (5)|RID (5) |WBID (5)|T|F|L|W|M|K|Flags|
+---------------+--------+--------+--------+-+-+-+-+-+-+-----+
| Fragment ID (16)            | Frag Offset (13)     |Rsvd(3)|
+-----------------------------+----------------------+-------+
| Optional Radio MAC and Wireless Specific fields            |
+------------------------------------------------------------+
```

| Field | Meaning |
| --- | --- |
| Preamble | Version and payload type. Type also says if DTLS follows. |
| HLEN | Header length |
| RID | Radio ID within the AP |
| WBID | Wireless binding. 1 means IEEE 802.11. |
| T | 1 means the payload is the native wireless frame. 0 means 802.3. |
| F, L | Fragment and last-fragment flags |
| W, M | Optional wireless info and radio MAC are present |
| K | Keepalive packet |

### MTU inside the tunnel

CAPWAP adds an outer IPv4 header (20 bytes), UDP (8 bytes), CAPWAP (8 bytes or more) and the inner frame header. DTLS adds more. So a 1,500-byte client packet does not fit in a 1,500-byte wired MTU.

CAPWAP can fragment at its own layer, but this wastes CPU and hurts throughput. Common fixes are a larger MTU on the wired path or lowering TCP MSS on the WLC. Check that ICMP is not blocked on the path. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

### 802.11r key hierarchy

Normal WPA2-Enterprise derives one PMK (Pairwise Master Key) per association. FT adds levels, so each AP can hold its own key without seeing the master.

```text
 MSK from 802.1X (or the PSK)
        |
        v
    PMK-R0   held by R0KH (the first key holder, e.g. WLC)
        |
        +---------------+----------------+
        v               v                v
    PMK-R1 (AP-1)   PMK-R1 (AP-2)    PMK-R1 (AP-3)
        |
        v
    PTK   per-session key, built from PMK-R1 and two nonces
```

- **R0KH and R1KH** are key holders on the network side.
- **MDID** (Mobility Domain ID) tells clients which APs share this key family. It appears in the beacon.
- **FT over SAE** gives the same speed-up for WPA3-Personal. FT with 802.1X applies to WPA3-Enterprise.

A compromise of one AP's PMK-R1 does not reveal PMK-R0. This is why the design has levels.

### QoS over Wi-Fi

Wi-Fi handles priority through **WMM (Wi-Fi Multimedia)**, which has four access categories.

| Category | Traffic | Typical DSCP |
| --- | --- | --- |
| AC_VO | Voice | EF (46) |
| AC_VI | Video | AF41 (34) |
| AC_BE | Best effort | 0 |
| AC_BK | Background | CS1 (8) |

RFC 8325 gives guidance on mapping DSCP to 802.11. For QoS to work end to end, three parts must agree: the client marks traffic, the AP and WLC trust or apply the correct policy, and CAPWAP's outer DSCP must carry that value so that wired switches treat it right. See [Quality of Service](../10-enterprise-and-datacenter/03-quality-of-service.md).

### Band steering, load balancing and airtime fairness

- **Band steering:** the AP delays or ignores some 2.4 GHz probes so that dual-band clients choose 5 GHz.
- **Client load balancing:** the WLC refuses or redirects new clients to a less busy AP. It can harm voice roaming if too strict.
- **Airtime fairness:** the AP gives devices equal airtime rather than equal frames, so a slow device does not hold back a fast one.

### Multicast and broadcast

Multicast frames on Wi-Fi have no ACK, and they go at a low basic rate so that every client can hear them. A busy mDNS or video stream can eat airtime. Common fixes are multicast-to-unicast conversion, mDNS gateways and blocking needless broadcast.

Devices like printers and TV casting use mDNS, which stays inside one VLAN. If users cannot find a printer on another VLAN, you need an mDNS gateway. See [Unicast, Broadcast, Multicast and Anycast](../04-ip-addressing/13-unicast-broadcast-multicast-anycast.md).

### High availability

- **SSO pair (Stateful Switchover):** two WLCs act as one. If the active one fails, APs and clients stay up. See [High Availability](../10-enterprise-and-datacenter/07-high-availability.md).
- **N+1:** a spare WLC takes over the APs. APs must re-join, which causes a short outage.
- **Primary, secondary, tertiary WLC:** each AP keeps an ordered list.
- **Geographic split:** place APs in a building on two WLCs, so that one failure does not take out every AP in the area.

### Firewall ports for a controller network

| Flow | Port | Notes |
| --- | --- | --- |
| AP to WLC, control | UDP 5246 | CAPWAP |
| AP to WLC, data | UDP 5247 | CAPWAP |
| WLC to WLC | UDP 16666, 16667 | Cisco mobility control and data |
| WLC to RADIUS | UDP 1812, 1813 | Auth and accounting |
| RADIUS to WLC, CoA | UDP 3799 or 1700 | RFC 5176 uses 3799. Cisco gear commonly uses 1700. |

### PoE and wired uplinks

Three IEEE PoE levels matter:

| Standard | Name | Power at the switch port |
| --- | --- | --- |
| 802.3af | PoE | 15.4 W |
| 802.3at | PoE+ | 30 W |
| 802.3bt | PoE++ | 60 W or 90 W |

Modern APs can need more than 802.3af. With too little power, an AP may shut off a radio, reduce spatial streams, or refuse to start. Check the AP's datasheet.

The uplink matters too. A fast Wi-Fi 6E or Wi-Fi 7 AP can pass more than 1 Gbps, which is why multi-gigabit ports (2.5 or 5 Gbps) are common. A Wi-Fi speed limited by a 1 Gbps port is a real issue on the newest APs.

### 6 GHz and Wi-Fi 7 design points

- **Discovery.** There are too many 6 GHz channels to scan. Clients usually learn about 6 GHz from the same AP's 2.4 or 5 GHz radio, through a Reduced Neighbor Report, or by using Preferred Scanning Channels.
- **Power classes.** Indoor APs commonly run as Low Power Indoor. Standard Power, used outdoors and with higher power, needs an Automated Frequency Coordination service in the US.
- **Coverage.** 6 GHz loses more through walls, so do not assume a 1:1 swap of 5 GHz coverage. Survey first.
- **Security.** 6 GHz has no open or WPA2 mode. Mixed SSIDs need planning.
- **BSS Colouring (Wi-Fi 6 onward).** Frames carry a "colour". A radio can sometimes treat another network's far-away frame as ignorable, and send anyway. This lets channels be reused more.
- **MLO (Multi-Link Operation, Wi-Fi 7).** One client uses links in more than one band at once, or picks the best one per frame. This reduces delay when one band is busy. It does not remove the need for roaming.
- **Next generation.** 802.11bn (Wi-Fi 8) is still in draft as of 2026. Reliability and smoother roaming are stated goals, but details can change.

### Over-the-air capture

A normal `tcpdump` on a laptop shows only Ethernet-like frames. To see 802.11 frames (beacons, retries, roams), use monitor mode.

```bash
# Linux: capture on channel 44, 40 MHz wide (adapter must support it)
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 44 HT40+
sudo tcpdump -i wlan0 -w roam.pcap
```

Useful [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) display filters:

| Filter | Shows |
| --- | --- |
| `wlan.fc.type_subtype == 0x08` | Beacons |
| `wlan.fc.type_subtype == 0x0b` | Authentication |
| `wlan.fc.type_subtype == 0x02` | Reassociation requests |
| `wlan.fc.type_subtype == 0x0c` | Deauthentication |
| `wlan.fc.retry == 1` | Retried frames |
| `eapol` | 802.1X and 4-way handshake |

Roaming captures need one radio per channel, or you may miss the roam. Capture near the client, because that is what the client hears.

### Rogue APs

A **rogue AP** is an AP on or near your network without permission. Controllers detect them during off-channel scans and through wired checks. See [Wi-Fi Security](02-wifi-security.md) and [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md). Be careful with automatic containment: sending deauth frames against other people's networks can be illegal.

## Common Mistakes

- **Thinking more APs always means better Wi-Fi.** Extra APs on too few channels add CCI and beacon overhead. Often the fix is fewer, better-placed, lower-power radios.
- **Turning AP power to maximum.** The client hears the AP, but the AP cannot hear the client. Roaming gets worse too.
- **Reading "bars" as speed.** Strong signal on a full channel is still slow.
- **Thinking the AP or WLC decides roaming.** The client decides. 802.11k and 802.11v only help it choose.
- **Saying 802.11r makes a client roam.** It only speeds up security once the client has chosen to move.
- **Enabling 802.11r on a shared SSID without testing.** Old printers, scanners and medical devices may stop joining.
- **Using 80 MHz on 5 GHz in a dense office.** Fewer channels means more CCI. The peak speed looks good on paper.
- **Thinking WPA2 protects data all the way to the server.** It only protects the radio hop. Data DTLS and TLS cover the rest.
- **Confusing Cisco "local mode" with local switching.** Local mode tunnels data to the WLC. FlexConnect switches locally.
- **Forgetting the VLAN on the trunk.** The client joins, then cannot get an IP address. This is very common.
- **Forgetting 2.4 GHz is still used.** Old IoT devices often live there. Plan it. Do not just turn it all off.
- **Skipping the post-install survey.** Real furniture, doors and people change RF.
- **Trusting RRM to fix a bad design.** It tunes what exists. It cannot move an AP out of a metal cupboard.
- **Blocking only one internal subnet for guests.** Block all private ranges so that new subnets stay safe by default.
- **Assuming mesh is as fast as a cable.** Each hop costs capacity and adds delay.
- **Blaming Wi-Fi first.** Test the same app on a cable. RADIUS, DNS, DHCP and even PoE are often the real cause.

## Interview Questions

**Q: What is the difference between an autonomous AP and a lightweight AP?**
A: An autonomous AP holds its own config and works alone, so you manage each one by hand. A lightweight AP gets its config from a controller and shares the MAC work with it. Lightweight APs scale better, with central policy, automatic RF tuning and better roaming. A cloud-managed AP is a modern middle ground: central management, local forwarding.

**Q: What is split MAC?**
A: Split MAC divides 802.11 work between the AP and the WLC. Real-time jobs like beacons, ACKs and retransmissions stay on the AP because they need replies in microseconds. Jobs that need a wide view, like association, authentication, roaming and RF decisions, go to the WLC.

**Q: How does a lightweight AP find and join a controller, and which ports does it use?**
A: The AP gets an IP by DHCP, then builds a list of WLCs from memory, DHCP option 43, DNS and broadcast. It sends Discovery Requests, picks a WLC, builds a DTLS session using certificates and sends a Join Request. It updates its image if needed, gets its config and goes to Run. CAPWAP control is UDP 5246, and data is UDP 5247.

**Q: When do you use central switching and when do you use FlexConnect?**
A: I use central switching on a campus with the WLC nearby, because policy sits in one place and only the AP VLAN is needed at the edge. I use FlexConnect at branches with a WAN link, so local traffic stays local and the branch can keep working if the WAN fails. The trade-off is that I must carry client VLANs to the AP on a trunk.

**Q: Explain 802.11k, 802.11v and 802.11r.**
A: 802.11k gives the client a list of neighbouring APs, so it scans less. 802.11v lets the network suggest a better AP with a BSS Transition request, and the client can refuse. 802.11r, Fast BSS Transition, shares keys in advance so the client can skip the full 802.1X login on a roam. So k and v help choose the AP, and r speeds up the move.

**Q: Who decides to roam, and how do you handle sticky clients?**
A: The client decides. To reduce sticky clients, I lower AP power to match client power and disable low data rates, so the far AP stops looking attractive. I enable 802.11k and 802.11v, and fix coverage overlap. Forced disconnects at low RSSI are a last option, because they can cause more trouble than they fix.

**Q: What is the difference between Layer 2 and Layer 3 roaming?**
A: In both, the client keeps its IP. In Layer 2 roaming, the new AP or WLC has the same VLAN, so the client just moves. In Layer 3 roaming, the new WLC is on a different subnet, so a mobility tunnel carries traffic between the foreign WLC and the anchor WLC, which holds the client's original VLAN.

**Q: What are the types of site survey, and what numbers do you design for?**
A: Predictive is a software model before you buy. AP-on-a-stick measures with a temporary AP. Passive and active are done after install to check coverage and real performance. Spectrum analysis finds non-Wi-Fi noise. For voice I commonly design for -67 dBm or better, SNR of 25 dB, and two APs heard at each spot. These are rules of thumb, and I check vendor guides.

**Q: How do you plan channels in a dense office, and what does RRM do?**
A: On 2.4 GHz I use only 1, 6 and 11 at 20 MHz, and I disable radios I do not need. On 5 GHz I use 20 or 40 MHz to get more channels, and I test DFS. I spread same-channel APs apart in three dimensions, with moderate power. RRM then watches the air and tunes channels and power, with coverage hole repair. It tunes a good design, but it does not replace one.

**Q: What are the pros and cons of mesh?**
A: Mesh gives coverage where cable is impossible, like yards and old buildings, and it can self-heal. But each hop adds delay and cuts throughput, especially if one radio handles both clients and backhaul. I keep it to two or three hops, use a dedicated backhaul radio when possible and use a cable whenever I can.

**Q: How would you design a guest network?**
A: I use a separate SSID and VLAN, with an ACL that blocks all private ranges and allows DHCP, DNS and the internet. I turn on client isolation and rate limits, and use a captive portal or sponsor login, plus OWE for encryption if clients support it. On a large site, I tunnel guests to an anchor WLC in the DMZ so that they never appear on internal VLANs.

**Q: A user says the Wi-Fi is slow. How do you troubleshoot?**
A: First I find the scope: one user, one area or everyone, and what changed. Then I find how far the client got: association, login, DHCP, DNS. Next I check RSSI, SNR, which AP it is on, channel utilisation and retries. I look at the WLC for the client and the AP. Finally I test on a cable or with iperf to prove whether Wi-Fi is the real cause.

## Quick Recap

- Autonomous APs work alone. Lightweight APs use a WLC through CAPWAP, on UDP 5246 for control and UDP 5247 for data. Cloud management is the modern third option.
- Split MAC keeps real-time work on the AP and wide-view work on the WLC. Central switching tunnels data to the WLC, and FlexConnect switches it locally.
- The client decides when to roam. 802.11k finds neighbours, 802.11v suggests moves, and 802.11r speeds up security.
- Airtime is the real resource. Good design means right AP placement, reused channels, moderate power, few SSIDs and no low data rates.
- Site surveys come before and after installation. RRM tunes a good plan but cannot fix a bad one.
- Guest networks need a separate VLAN, a block on all private ranges, client isolation and a login step. Mesh trades capacity for reach.
- Troubleshoot by scope, then by connection stage, then RF, then the wire. The cause is often not Wi-Fi at all.
