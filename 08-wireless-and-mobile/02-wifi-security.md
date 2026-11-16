# Wi-Fi Security

**Level:** Intermediate | **Prerequisites:** [Wi-Fi Basics](01-wifi-basics.md), [Cryptography Basics](../09-security/02-cryptography-basics.md)

## In One Line

Wi-Fi security proves who is joining the network, then gives each device fresh keys so that nobody nearby can read or change its traffic. Today that means WPA2 at minimum and WPA3 where possible.

## Why It Matters

A cable stays inside a wall. A radio signal does not. Anyone in the car park with a 30 dollar USB adapter can receive every frame your access point sends. So on Wi-Fi, an attacker needs no physical access to your network.

Real damage has come from this. The TJX retail breach, made public in 2007, began with attackers getting in through a store network that used WEP. Over 45 million payment card records were reported stolen.

Interviewers ask about this topic because it joins many ideas in one flow: encryption, hashing, key exchange, authentication servers and protocol design mistakes. They want to know three things:

- Can you explain why each generation replaced the last one?
- Can you walk through the four-way handshake?
- Can you pick the right design for a home, a cafe and a company?

## Simple Explanation

Imagine two people talking in a crowded public square. Everyone can hear them. They need three things:

1. **Proof of identity.** Is this really my friend, or someone pretending?
2. **A secret language.** Others can hear the sounds but cannot understand them.
3. **A seal on every message.** If someone changes a message, the seal breaks.

Wi-Fi security gives exactly these three things. The formal names are **authentication** (proving identity), **encryption** (scrambling data so only key holders can read it) and **integrity** (detecting changes).

There are two ways to prove identity:

- **Personal mode (PSK)** is like one door code for a whole house. Everyone uses the same code. It is simple, but if one person leaves, you must change the code for everybody.
- **Enterprise mode (802.1X)** is like a staff badge. Each person has their own badge. A guard checks it with head office. When someone leaves, only that badge is cancelled.

Where the analogy stops: people in a square use one language for the whole talk. Wi-Fi creates a fresh secret key for each device on each connection, even when everyone typed the same password.

## How It Works

### The Connection Steps

**Terms first.** A **station (STA)** is a client device such as a phone or laptop. An **access point (AP)** is the radio that clients join. The **SSID** is the network name you see. The **BSSID** is the MAC address of one AP radio. A **frame** is one unit of data sent at layer 2. A **beacon** is a frame the AP sends about 10 times per second to announce the network and its security settings.

A client joins a protected network in this order:

```text
Client (STA)                                      Access Point (AP)
   |                                                     |
   |  1. Beacon or probe response: "WPA2/WPA3, CCMP"     |
   |<----------------------------------------------------|
   |  2. 802.11 authentication (Open, or SAE for WPA3)   |
   |<--------------------------------------------------->|
   |  3. Association: client agrees to the security type |
   |<--------------------------------------------------->|
   |  4. 802.1X/EAP login (Enterprise mode only)         |
   |<--------------------------------------------------->|
   |  5. Four-way handshake: fresh keys for this session |
   |<--------------------------------------------------->|
   |  6. Encrypted data, DHCP, then normal traffic       |
   |<===================================================>|
```

Step 2 is confusing because of its name. With WPA2, "Open System authentication" proves nothing. It is an old formality. The real proof happens in step 4 or 5. With WPA3-Personal, step 2 is real, because SAE happens there.

### The History in One Table

| Name | Year | Cipher | Integrity check | Status in 2026 |
|---|---|---|---|---|
| Open | 1997 | None | None | Still used in guest networks, avoid |
| WEP | 1997 | RC4 | CRC-32 | Broken, never use |
| WPA | 2003 | RC4 with TKIP | Michael | Broken, deprecated |
| WPA2 | 2004 | AES-CCMP (128-bit) | CBC-MAC | Acceptable minimum |
| WPA3 | 2018 | AES-CCMP or AES-GCMP | CBC-MAC or GMAC | Current best choice |

A **cipher** is the algorithm that scrambles data. **WPA** stands for Wi-Fi Protected Access. **IEEE 802.11i** is the IEEE security standard ratified in 2004, and **WPA2** is the Wi-Fi Alliance certification of it. The IEEE writes standards. The Wi-Fi Alliance is an industry group that tests and certifies products. WPA, WPA2 and WPA3 are Alliance names, not IEEE names.

### Open Networks

Open means no authentication and no encryption at the Wi-Fi layer. Anyone nearby with a capture tool can read every frame.

A **captive portal** is the web page that asks you to accept terms at a hotel or airport. It controls access. It does not encrypt anything. An open network with a login page is still an open network.

HTTPS protects the content of web traffic on open Wi-Fi, as covered in [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md). It does not hide everything. Observers can still see the IP addresses you reach and often the DNS names.

### WEP: Why It Failed

**WEP** means Wired Equivalent Privacy. It uses the **RC4** cipher. RC4 is a **stream cipher**: it produces a long random-looking series of bytes, called the **keystream**, and combines it with the data using **XOR**. XOR is a simple bit operation where doing it twice gets the original back.

```text
 24-bit IV (sent in clear)     40 or 104-bit shared key
            \                        /
             +----> RC4 seed <------+
                       |
                       v
                  RC4 keystream
                       |
 data + CRC-32 ------> XOR ------> ciphertext
```

An **IV (initialization vector)** is a value that makes each packet's keystream different. WEP had four fatal problems:

1. **The IV is too short.** 24 bits gives only 16,777,216 values. After about 5,000 frames there is a 50 percent chance that one IV has repeated. Same IV plus same key means same keystream, and that lets an attacker recover data.
2. **The key is static and shared.** All users have the same key, and it never changes until an admin changes it everywhere.
3. **RC4 was used badly.** Fluhrer, Mantin and Shamir showed in 2001 that weak IVs leak key bytes. The PTW attack in 2007 made recovery of a 104-bit key take about a minute with roughly 40,000 to 85,000 captured frames.
4. **CRC-32 is not a security check.** It finds accidental errors. An attacker can flip bits in the data and fix the CRC without knowing the key. WEP also had no protection against **replay**, which means capturing a valid frame and sending it again.

The so-called 128-bit WEP is really a 104-bit key plus the 24-bit IV. IEEE deprecated WEP in 2004.

Analogy: WEP is a lock with only a small set of keys in the whole world. Wait long enough and the same key comes around again. Where it stops: the real failure was also in how RC4 was seeded, not just the number of IVs.

### WPA and TKIP: The Emergency Fix

By 2003 millions of devices had WEP hardware that could not run AES. The Wi-Fi Alliance released **WPA** as a fast fix, based on a draft of 802.11i. It could be added with a firmware update.

**TKIP (Temporal Key Integrity Protocol)** wraps RC4 with better practices:

- A new key for every packet, made by mixing the session key with a counter.
- A 48-bit counter that also blocks replays.
- **Michael**, a 64-bit integrity check called a **MIC** (Message Integrity Code). Wi-Fi says MIC because MAC already means Media Access Control.
- Countermeasures: two Michael failures within 60 seconds make the AP stop for 60 seconds.

TKIP was a bridge, never a final design. The Beck-Tews attack in 2008 could decrypt short packets such as ARP frames (address lookup messages) and inject a few forged ones. IEEE later deprecated TKIP. Also, 802.11n high rates do not allow TKIP, so a network forced to TKIP is usually capped at 54 Mbps. "WPA2 mixed mode is slow" is often this exact problem.

### WPA2: The Long-Lived Standard

WPA2 brought **CCMP**, which is based on **AES**, a strong block cipher used across the world. A **block cipher** encrypts fixed chunks of 16 bytes. CCMP uses AES with a 128-bit key for two jobs:

- **Counter mode (CTR)** for encryption.
- **CBC-MAC** for an 8-byte MIC.

Each frame carries a 48-bit **packet number (PN)**. This number must never repeat under the same key. It also blocks replay, because the receiver drops any number it has already seen. Remember this fact, as KRACK depends on it.

CCMP protected data frame (field table):

| Field | Size | Encrypted? | Covered by MIC? |
|---|---|---|---|
| 802.11 MAC header | 24-36 bytes | No | Most fields |
| CCMP header (PN, key ID) | 8 bytes | No | PN feeds the nonce |
| Data (payload) | Variable | Yes | Yes |
| MIC | 8 bytes | Yes | Not applicable |
| FCS (frame checksum) | 4 bytes | No | No |

WPA2 has lasted over 20 years because AES-CCMP itself has held up. The weak parts are around it:

- Personal mode allows **offline guessing** of the password.
- Management frames are unprotected unless PMF is turned on.
- The password also unlocks past traffic if someone recorded it.

### WPA3: What Changed

The Wi-Fi Alliance announced WPA3 in 2018. From mid-2020 it required WPA3 support in newly certified devices.

| Mode | Authentication | Encryption | PMF |
|---|---|---|---|
| WPA3-Personal | SAE | CCMP-128 | Required |
| WPA3-Personal transition | SAE and PSK on one SSID | CCMP-128 | Optional for WPA2 clients |
| WPA3-Enterprise | 802.1X/EAP | CCMP-128 | Required in WPA3-only mode |
| WPA3-Enterprise 192-bit | EAP-TLS with 384-bit curves | GCMP-256 | Required |
| Enhanced Open (separate cert) | None (OWE) | CCMP-128 | Required |

The main gains:

- **SAE** blocks offline password guessing.
- **Forward secrecy**: stealing the password later does not unlock recorded traffic.
- **PMF is mandatory**, so spoofed deauth frames stop working.
- **Server certificate checks** are required on WPA3-Enterprise clients.

WPA3 keeps 128-bit CCMP for Personal. The improvement is in the key exchange, not in a bigger cipher.

### PSK vs Enterprise

**PSK (Pre-Shared Key)** means every device knows the same secret ahead of time. The **passphrase** you type is 8-63 printable ASCII characters, or you can enter exactly 64 hex digits as the raw key.

| Topic | Personal (PSK or SAE) | Enterprise (802.1X) |
|---|---|---|
| Credential | One shared password | Per-user password or certificate |
| Extra server | None | RADIUS server required |
| Staff leaves | Change password on all devices | Disable one account |
| Who connected? | Only a MAC address | Real username in logs |
| VLAN (virtual LAN) by user | Not in the standard | Yes, RADIUS can assign it |
| Best for | Homes, very small offices | Companies, universities |

There is also a middle option. **Per-user PSK** gives each device its own passphrase on one SSID. Vendors call it iPSK (Cisco), MPSK (Aruba) or similar. It is not a single standard, and it needs a server or controller to store the keys. It suits printers and IoT devices that cannot do 802.1X.

### 802.1X and EAP

**802.1X** is an IEEE standard for port-based access control. A **port** here means a point of network access. On Wi-Fi it is the logical link between one client and the AP. Until login succeeds, the port passes only login traffic.

**EAP (Extensible Authentication Protocol, RFC 3748)** is a framework that carries many login methods. It defines the message types. The actual method plugs in.

Three roles:

| Role | Who | Job |
|---|---|---|
| Supplicant | Laptop or phone | Asks to join, gives credentials |
| Authenticator | AP or wireless controller | Passes EAP messages, opens the port |
| Authentication server | RADIUS server | Checks credentials, says yes or no |

**RADIUS** is the protocol that carries login requests to the central server, normally on UDP 1812 (authentication) and UDP 1813 (accounting). See [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

Analogy: the supplicant is a visitor, the AP is a door guard, and the RADIUS server is head office. The guard does not read your sealed envelope. The guard passes it to head office and waits for the answer. Where it stops: a real guard forgets you after you enter. In 802.1X, the server also sends a key to the AP, and both sides use it to protect everything that follows.

```text
Supplicant           Authenticator (AP)           RADIUS server
(laptop)             10.10.10.2                   10.10.20.5
   |                         |                          |
   |  EAP Request/Identity   |                          |
   |<------------------------|                          |
   |  EAP Response/Identity  |  RADIUS Access-Request   |
   |------------------------>|------------------------->|
   |                         |                          |
   |   EAP method (for example a TLS tunnel), many      |
   |   rounds: EAPOL on the air, RADIUS on the wire     |
   |<==================================================>|
   |                         |                          |
   |                         |  Access-Accept + key     |
   |  EAP Success            |<-------------------------|
   |<------------------------|                          |
   |                         |                          |
   |  Four-way handshake     |                          |
   |<----------------------->|                          |
```

**EAPOL (EAP over LAN)** carries EAP between client and AP, using EtherType 0x888E. Between the AP and the server, the same EAP messages travel inside RADIUS.

Common EAP methods:

| Method | Client proves with | Server proves with | Notes |
|---|---|---|---|
| EAP-TLS | Client certificate | Server certificate | Strongest common choice, needs a PKI |
| PEAP-MSCHAPv2 | Password inside tunnel | Server certificate | Very common with Active Directory (Microsoft's user directory) |
| EAP-TTLS | Password or other inner method | Server certificate | Like PEAP, flexible inner methods |
| EAP-FAST | PAC (stored secret file) or certificate | PAC or certificate | Cisco design, now mostly legacy |
| EAP-SIM, EAP-AKA | SIM card | Mobile network | Used for carrier Wi-Fi offload |
| LEAP | Password hash | Password hash | Cisco legacy, broken, do not use |

A **certificate** is a signed digital ID. A **PKI (Public Key Infrastructure)** is the system that issues and manages certificates. The "tunnel" in PEAP and TTLS is a TLS session that hides the inner password exchange from anyone listening.

Simple rule: EAP-TLS needs no password that can be stolen, so it resists phishing. PEAP is easier to roll out, but it is only safe if clients check the server certificate.

### The Key Hierarchy

Wi-Fi never uses your password directly to encrypt data. It builds a chain of keys.

```text
  Personal (WPA2)                  Enterprise
  passphrase + SSID                EAP method output (MSK)
  PBKDF2-SHA1, 4096 rounds         first 256 bits
           \                          /
            +-------> PMK <----------+       (WPA3: PMK from SAE)
                       |
    PRF(PMK, AP MAC, client MAC, ANonce, SNonce)
                       |
                       v
                      PTK  (per client, per session)
          +------------+------------+
          |            |            |
         KCK          KEK           TK
       signs the    wraps the    encrypts
       handshake    GTK          unicast data

  AP makes one GTK per SSID: shared by all clients, for broadcast
```

| Key | Full name | Lifetime | Purpose |
|---|---|---|---|
| PMK | Pairwise Master Key | Long-lived (PSK), per login (Enterprise) | Top secret shared by client and AP |
| PTK | Pairwise Transient Key | One session | Parent of the three keys below |
| KCK | Key Confirmation Key | One session | Creates the handshake MIC |
| KEK | Key Encryption Key | One session | Encrypts the GTK in transit |
| TK | Temporal Key | One session | Encrypts unicast frames |
| GTK | Group Temporal Key | Until rekey | Encrypts broadcast and multicast |

**PBKDF2** is a function that makes password guessing slow by repeating a hash many times. A **hash** is a one-way fingerprint of data. A **PRF** (pseudo-random function) stretches inputs into a long key. With CCMP the PTK is 384 bits: 128 each for KCK, KEK and TK.

The SSID acts as the **salt** in PBKDF2. A salt is public extra input that stops one precomputed list from working on every network. This is why very common names such as "linksys" were targets for ready-made tables.

### The Four-Way Handshake

The four-way handshake proves that both sides have the PMK, without sending it. It then creates a fresh PTK and delivers the GTK.

A **nonce** is a random number used once. **ANonce** comes from the AP (authenticator). **SNonce** comes from the client (supplicant).

Analogy: two friends share a secret recipe. Each rolls a dice and shouts the number. Each mixes the recipe with both numbers to bake today's cake. Each shows a slice with a tasting note that only someone who baked it could write. A listener hears the numbers but has no recipe. Where it stops: nothing is "shown" except a short MIC, and the key is never sent at all.

```text
Client (Supplicant)                          AP (Authenticator)
has PMK                                      has PMK
   |                                               |
   |  M1: ANonce                                   |
   |<----------------------------------------------|
   |                                               |
 picks SNonce                                      |
 derives PTK                                       |
   |                                               |
   |  M2: SNonce, MIC, client RSN info             |
   |---------------------------------------------->|
   |                                               |
   |                                   derives PTK, checks MIC
   |                                               |
   |  M3: ANonce, AP RSN info, GTK (encrypted      |
   |      with KEK), MIC, "install key"            |
   |<----------------------------------------------|
   |                                               |
 checks MIC, installs PTK and GTK                  |
   |                                               |
   |  M4: MIC, "done"                              |
   |---------------------------------------------->|
   |                                               |
   |                                     installs PTK
   |  <=========== encrypted traffic ===========>  |
```

What each message proves:

1. **M1** has no MIC. It only gives the ANonce. The client now has everything to compute the PTK.
2. **M2** carries the SNonce and a MIC. A correct MIC proves the client knows the PMK. If the password is wrong, the check fails here.
3. **M3** proves the AP knows the PMK, because its MIC is correct. It repeats the AP's security info so the client can detect a downgrade. It also delivers the GTK.
4. **M4** confirms. Both sides now install keys and the 802.1X port opens.

A separate two-message **group key handshake** replaces the GTK later. Many networks rekey when a client leaves, or on a timer. Both are configurable and vary by vendor.

### SAE: The WPA3-Personal Handshake

**SAE (Simultaneous Authentication of Equals)** is based on the **Dragonfly** key exchange (RFC 7664). It is a **PAKE**, a password-authenticated key exchange. It runs in 802.11 authentication frames, before association.

Analogy: two people mix a secret word with their own private paint colour, then swap mixed paints. Only with the same word do they end with the same final colour. A watcher holding the paints still cannot test words at home, because each guess needs a live exchange with one of the two people. Where it stops: the real thing uses elliptic curve math, and the mathematical hardness is what protects it.

```text
Client                                                  AP
   |  SAE Commit: scalar + element                       |
   |---------------------------------------------------->|
   |  SAE Commit: scalar + element                       |
   |<----------------------------------------------------|
   |     both compute a shared secret                    |
   |  SAE Confirm: proof of key                          |
   |---------------------------------------------------->|
   |  SAE Confirm: proof of key                          |
   |<----------------------------------------------------|
   |     256-bit PMK now exists on both sides            |
   |  Association, then the normal four-way handshake    |
   |<===================================================>|
```

Key points:

- **Random in every run.** Each side picks fresh random values, so each PMK is unique, even with one shared password.
- **No offline guessing.** A capture does not let an attacker test password guesses. Each guess needs a live attempt that the AP can limit.
- **Forward secrecy.** If the password leaks tomorrow, recorded traffic from today stays safe.
- **Equal peers.** Either side can start. There is no fixed client and server role.
- **Four-way handshake still runs.** It uses the SAE PMK to derive the PTK and send the GTK.

SAE commit processing costs CPU. To resist floods of fake commits, the AP can demand an **anti-clogging token**, a cookie that proves the sender's address is real before it does the hard math.

### PMF: Protected Management Frames

**Management frames** run the Wi-Fi connection: beacons, authentication, association, deauthentication and more. **IEEE 802.11w** (2009) added protection for some of them, known as PMF.

Analogy: without PMF, a stranger can slide an unsigned note under your door saying "You are fired, leave the building", and you obey. PMF means you ignore notes without a real company stamp. Where it stops: PMF cannot stop someone from making so much noise that you hear nothing. That is radio jamming.

PMF does three things:

1. **Unicast protection.** Deauthentication, disassociation and some action frames are encrypted and integrity protected with the PTK.
2. **Broadcast protection.** Broadcast deauth frames get a MIC using the **IGTK** (Integrity Group Temporal Key), through a method called **BIP**. The content is readable but cannot be forged.
3. **SA Query.** If the AP gets a new association request for a client that is still connected, it sends a protected query to the real client. If that client answers, the AP refuses the new request for a short time. This stops an attacker from kicking a client off by faking a new join.

PMF does not protect beacons or probes in its original form. A later feature called **beacon protection** adds a MIC on beacons with a key called the BIGTK. Client support varies.

Settings are usually **disabled**, **optional (capable)** or **required**. WPA3 needs required mode. WPA2 can use it too, if clients support it.

### OWE: Enhanced Open

**OWE (Opportunistic Wireless Encryption, RFC 8110)**, certified as **Wi-Fi Enhanced Open**, gives encryption on networks with no password. It runs a **Diffie-Hellman** exchange during association, which lets two sides agree on a secret over an open channel.

Each user gets different keys, so a nearby listener cannot read traffic passively. But there is no authentication. A fake AP can run OWE too. Think of it as a private talk with a stranger: secret, but you still do not know who the stranger is.

### Common Attacks

Run these only on networks you own, or with written permission. In many countries, attacking other people's Wi-Fi is a crime.

#### Evil Twin

An **evil twin** is a fake AP that copies a real SSID, and often its security type, so that clients join it. Clients often choose the strongest signal, so a closer fake AP wins. Attackers then read traffic or show a fake login page.

Analogy: a fake bank branch with the real logo. Where it stops: a fake bank needs a building, but a fake AP is one laptop in a backpack.

The security type changes what the attacker can do:

| Real network | What the evil twin needs |
|---|---|
| Open | Nothing, just the same SSID |
| WPA2 or WPA3 Personal, unknown password | Cannot finish the handshake, so it often offers an open SSID with a fake "enter Wi-Fi password" page |
| Personal, known password (cafe wall) | Password from the wall, full impersonation works |
| Enterprise, clients check certificates | Client refuses the fake server |
| Enterprise, clients skip checks | Fake RADIUS collects the MSCHAPv2 exchange for cracking |

A related trick is **KARMA**: a fake AP answers client probes for networks the client remembers, such as "Airport_Free".

Defences: Enterprise certificate validation, wireless intrusion detection (WIDS/WIPS), removing saved open networks, VPN on untrusted Wi-Fi, and SAE-PK for shared public passwords (see Going Deeper).

A **rogue AP** is different. It is an unauthorized AP plugged into your wired network, often by a staff member. An evil twin imitates. A rogue AP bypasses.

#### Deauthentication Attack

A **deauthentication (deauth) frame** tells a station that the connection is over. Before PMF, it had no protection. An attacker with an adapter that can send raw frames can forge one with the AP's MAC as the source, and the client obeys.

Uses of the attack:

- **Denial of service (DoS)**: keep clients off the network.
- **Handshake capture**: the client reconnects at once, so the attacker captures M1 and M2.
- **Evil twin push**: kick clients off the real AP so they join the fake one.

The fix is PMF. With PMF required, the client ignores forged deauth frames. Jamming the radio band is still possible, because no protocol can fix physics.

#### Offline Dictionary Attack on WPA2-PSK

This is the most common real attack on home and small-office WPA2.

1. Capture M1 and M2 of a handshake (passive, or after a deauth).
2. Take a word from a list. Run PBKDF2 with the SSID to get a PMK.
3. Use the nonces and MAC addresses to compute the PTK.
4. Compute the MIC and compare it with M2.
5. Match means the password is found. No match means try the next word.

The attack never touches the AP again, so no lockout can stop it. A **dictionary attack** tries likely words instead of every combination.

The **PMKID** variant, published in 2018, removes the need for a client. Some APs include a PMKID value in M1, and it is computed from the PMK and MAC addresses. One frame gives enough for offline guessing.

The defence is a long random passphrase, or SAE.

#### KRACK

**KRACK (Key Reinstallation Attack)** was published in 2017 by Mathy Vanhoef and Frank Piessens. It attacks the handshake logic, not AES and not the password.

1. The attacker sits between client and AP using a fake AP on another channel, a position called **man-in-the-middle**.
2. The client gets M3 and installs the key. Its packet number starts at 0.
3. The attacker blocks M4 from reaching the AP.
4. The AP thinks M3 was lost, so it sends M3 again. This is normal behaviour.
5. The client accepts M3 again and **reinstalls the same key**, which resets its packet number.
6. The client now reuses packet numbers with the same key. That reuses keystream, so traffic can be decrypted, and replays work.

Analogy: a one-time pad sheet is used only once. Someone shouts "Sorry, say that again", and you reset to the first sheet. Where it stops: KRACK did not expose all data at once. The attacker needed to be close, and HTTPS still protected the content.

Key facts for interviews:

- The password is not revealed, so changing it does nothing. The fix is a software patch.
- It was a flaw in the standard's logic, so WPA2 and WPA were both affected.
- Clients were the main targets, and patching clients mattered most. AP fixes mattered for 802.11r fast roaming.
- Linux and Android 6.0 using wpa_supplicant 2.4 or later had a worse bug: they installed an all-zero key.
- The fix: never reinstall a key already in use.

#### WPS

**WPS (Wi-Fi Protected Setup)**, from 2006, lets you join by pushing a button or typing an 8-digit PIN. The PIN method was badly designed.

- The last digit is a checksum, leaving 7 real digits.
- The protocol confirms each half separately, so the first half (4 digits) falls in at most 10,000 tries and the second half (3 digits) in at most 1,000 tries.
- The total is at most **11,000 guesses**, not 100,000,000.

Stefan Viehboeck published this in December 2011, and the Reaver tool made it easy. Many APs had no lockout. Once the PIN is known, the AP hands over the WPA password, however long it is.

**Pixie Dust** (2014) is worse: on some chipsets, weak random numbers let attackers compute the PIN offline, in seconds or minutes, with only one exchange.

Analogy: a combination lock that clicks when the first half is right. Where it stops: a real lock does not then give you a copy of the master key.

The fix is to disable WPS completely. On some consumer routers the menu setting has not fully disabled it in the past, so test it. The modern replacement is **Wi-Fi Easy Connect**, also called DPP, which uses QR codes and public keys.

## Example

Network used in these examples: SSID `ExampleNet`, AP BSSID `02:11:22:33:44:55`, client MAC `02:aa:bb:cc:dd:01`, passphrase `correct horse battery`.

### Example 1: Check What Your Wi-Fi Is Using

Windows:

```powershell
netsh wlan show interfaces
```

```text
    Name                   : Wi-Fi
    State                  : connected
    SSID                   : ExampleNet
    AP BSSID               : 02:11:22:33:44:55
    Radio type             : 802.11ax
    Authentication         : WPA3-Personal
    Cipher                 : CCMP
    Channel                : 36
    Signal                 : 92%
```

Output is trimmed, and labels vary a little by Windows build. Use `netsh wlan show networks mode=bssid` to list what nearby networks offer.

Linux:

```bash
nmcli -f SSID,BSSID,SECURITY,SIGNAL dev wifi list
```

```text
SSID        BSSID              SECURITY    SIGNAL
ExampleNet  02:11:22:33:44:55  WPA2 WPA3   92
GuestWifi   02:11:22:33:44:66  --          88
OldPrinter  02:77:88:99:aa:bb  WPA1 WPA2   41
```

`--` means open. `WPA2 WPA3` means transition mode. For full detail, read the RSN section of a scan. **RSN (Robust Security Network)** is the information block the AP advertises:

```bash
sudo iw dev wlan0 scan | grep -A 6 "RSN:"
```

```text
        RSN:     * Version: 1
                 * Group cipher: CCMP
                 * Pairwise ciphers: CCMP
                 * Authentication suites: SAE
                 * Capabilities: 16-PTKSA-RC 1-GTKSA-RC MFP-required MFP-capable (0x00cc)
                 * Group mgmt cipher: AES-128-CMAC
```

This is WPA3-Personal only: SAE, CCMP, PMF required. MFP (Management Frame Protection) is the same thing as PMF.

### Example 2: Compute WPA2 Keys by Hand

`wpa_passphrase` (Linux) turns a passphrase into a PMK:

```bash
wpa_passphrase ExampleNet "correct horse battery"
```

```text
network={
	ssid="ExampleNet"
	#psk="correct horse battery"
	psk=30d134634e5e95f61c81f8072e33ba99928ce20d607be55c0a5cb341cc3ac39e
}
```

The `psk` line is the PMK. Same password with another SSID gives a totally different PMK.

This Python script does what each side does in the handshake. The nonces are fake and easy to read, so you can repeat the result.

```bash
python3 - <<'EOF'
import hashlib, hmac
pmk = hashlib.pbkdf2_hmac("sha1", b"correct horse battery",
                          b"ExampleNet", 4096, 32)
aa, spa = bytes.fromhex("021122334455"), bytes.fromhex("02aabbccdd01")
anonce, snonce = b"\xaa" * 32, b"\x55" * 32
data = min(aa, spa) + max(aa, spa) + min(anonce, snonce) + max(anonce, snonce)
ptk = b""
for i in range(3):
    ptk += hmac.new(pmk, b"Pairwise key expansion\x00" + data + bytes([i]),
                    hashlib.sha1).digest()
pmkid = hmac.new(pmk, b"PMK Name" + aa + spa, hashlib.sha1).digest()[:16]
print("PMK  ", pmk.hex())
print("KCK  ", ptk[:16].hex())
print("KEK  ", ptk[16:32].hex())
print("TK   ", ptk[32:48].hex())
print("PMKID", pmkid.hex())
EOF
```

```text
PMK   30d134634e5e95f61c81f8072e33ba99928ce20d607be55c0a5cb341cc3ac39e
KCK   aa6059f661bb5dbd109298938e32556b
KEK   e523ad0b057ff5033ba060d86e066d7e
TK    9c45815ff545c892a80ffcec25f5c22e
PMKID 93c874426b9b90686ac1c6bbb2ea5909
```

Now you can see why WPA2-PSK fails against a weak password. Everything except the passphrase is public on the air. An attacker simply repeats this script with a new guess, again and again. It also shows why one WPA2 user with the password can decrypt another user's traffic, if they captured that user's handshake.

The script does not apply to SAE. SAE does not compute the PMK from the password alone.

### Example 3: See the Handshake in a Capture

Use a second adapter in **monitor mode**, which lets a card receive all frames on a channel. Do this on your own network (Linux):

```bash
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 36
sudo tshark -i wlan1 -Y "eapol"
```

Connect a phone to `ExampleNet` and watch (frame lengths vary by driver):

```text
 1 0.000000000 02:11:22:33:44:55 -> 02:aa:bb:cc:dd:01 EAPOL 155 Key (Message 1 of 4)
 2 0.004120000 02:aa:bb:cc:dd:01 -> 02:11:22:33:44:55 EAPOL 177 Key (Message 2 of 4)
 3 0.006880000 02:11:22:33:44:55 -> 02:aa:bb:cc:dd:01 EAPOL 211 Key (Message 3 of 4)
 4 0.009310000 02:aa:bb:cc:dd:01 -> 02:11:22:33:44:55 EAPOL 155 Key (Message 4 of 4)
```

Useful Wireshark display filters:

| Filter | Shows |
|---|---|
| `eapol` | Handshake and 802.1X messages |
| `wlan.fc.type_subtype == 0x0c` | Deauthentication frames |
| `wlan.fc.type_subtype == 0x0a` | Disassociation frames |
| `wlan.fc.type_subtype == 0x0b` | Authentication frames, including SAE |
| `wlan.fixed.reason_code` | Frames with a reason code |

A burst of deauth frames with no reason in your plans suggests an attack, or an AP issue. For WPA2-PSK, Wireshark can decrypt a capture if you add `correct horse battery:ExampleNet` as a `wpa-pwd` key under the IEEE 802.11 protocol settings and the capture holds that client's handshake. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md). With SAE, the password alone is not enough.

### Example 4: How Long Would a WPA2 Password Last?

Assume an attacker tests 1,000,000 guesses per second. That is a round number for one strong GPU (graphics card). Real speed depends on hardware. Attackers often rent many GPUs, and word lists beat pure brute force for human-chosen passwords.

| Password type | Possibilities | Time to try all |
|---|---|---|
| 8 digits (often a phone number) | 100,000,000 | 100 seconds |
| 8 lowercase letters | 208,827,064,576 | About 58 hours |
| 4 random common words (from 7,776) | About 3.66 x 10^15 | About 116 years |
| 12 random letters and digits | About 3.23 x 10^21 | About 100 million years |

WPA3-SAE takes this table away, because guessing must happen live against the AP.

### Example 5: Configure a Modern Network

Linux AP with `hostapd`, WPA2/WPA3 transition mode:

```text
# /etc/hostapd/hostapd.conf
interface=wlan0
ssid=ExampleNet
hw_mode=a
channel=36
ieee80211n=1
ieee80211ac=1

wpa=2
wpa_key_mgmt=WPA-PSK SAE
rsn_pairwise=CCMP
wpa_passphrase=correct horse battery

# 1 = PMF optional, needed so that old WPA2 clients can join
ieee80211w=1
# SAE clients must use PMF, even in transition mode
sae_require_mfp=1
# 2 = allow both SAE password methods (see H2E in Going Deeper)
sae_pwe=2
```

For WPA3-only, set `wpa_key_mgmt=SAE` and `ieee80211w=2`. Some old clients will then fail to connect, so test first.

Cisco Catalyst 9800 wireless controller (IOS XE), WPA3-Personal. Commands can differ by release, so check your version's guide:

```text
wlan HOME-WPA3 1 HOME-WPA3
 no security ft adaptive
 no security wpa wpa2
 no security wpa akm dot1x
 security wpa wpa3
 security wpa akm sae
 security wpa psk set-key ascii 0 <passphrase>
 security pmf mandatory
 no shutdown
```

Linux client for EAP-TLS, with `wpa_supplicant`. The `ca_cert` and `domain_match` lines are the ones that defeat an evil twin:

```text
network={
    ssid="CorpNet"
    key_mgmt=WPA-EAP
    eap=TLS
    identity="alice@example.com"
    ca_cert="/etc/ssl/certs/corp-ca.pem"
    domain_match="radius.example.com"
    client_cert="/etc/ssl/certs/alice.pem"
    private_key="/etc/ssl/private/alice.key"
    ieee80211w=1
}
```

## Going Deeper

### RSN Information Element and Suite Numbers

The AP advertises security in the RSN element (element ID 48). Suite values use the IEEE OUI `00-0F-AC` plus a number. Old WPA used a vendor-specific element (ID 221) instead.

| AKM suite | Meaning | Cipher suite | Meaning |
|---|---|---|---|
| 1 | 802.1X | 2 | TKIP |
| 2 | PSK | 4 | CCMP-128 |
| 3 | FT over 802.1X | 8 | GCMP-128 |
| 4 | FT with PSK | 9 | GCMP-256 |
| 5 | 802.1X with SHA-256 | 10 | CCMP-256 |
| 6 | PSK with SHA-256 | 6 | BIP-CMAC-128 |
| 8 | SAE | | |
| 9 | FT with SAE | | |
| 12 | 802.1X, 192-bit mode | | |
| 18 | OWE | | |

**AKM** means Authentication and Key Management. **FT** is Fast Transition (802.11r), which speeds up roaming between APs. Roaming design is covered in [Enterprise Wi-Fi](03-enterprise-wifi.md).

The handshake protects this data. M2 and M3 repeat the RSN information under a MIC, and a mismatch with what was seen in the beacon ends the connection. That blocks a simple downgrade, such as an attacker removing CCMP from a beacon.

### Dragonblood and H2E

SAE was not perfect at first. **Dragonblood**, from Mathy Vanhoef and Eyal Ronen in 2019, showed two kinds of issue:

- **Downgrade**: in transition mode, a fake AP that offers only WPA2 makes a client do a WPA2 handshake, and the attacker can capture enough for offline guessing.
- **Side channels**: the first SAE password-to-group-element method, called **hunting and pecking**, took a different time or cache pattern for different passwords. A **side channel** leaks secrets through timing or similar effects instead of breaking the math.

The fix for the second issue is **H2E (Hash-to-Element)**, a constant-time method added to the standard. In `hostapd`, `sae_pwe=1` means H2E only, and `sae_pwe=2` allows both. WPA3 also added **transition disable**, which tells a client to stop using the weaker mode on this network from now on.

### Why Transition Mode Is a Risk

Transition mode serves WPA2 and WPA3 clients on one SSID. It is a practical migration step, but the WPA2 side keeps its weakness: an attacker can bring up a WPA2-only copy of the SSID, capture what they need, and guess offline. Use transition disable where clients support it, and move to WPA3-only once all clients can.

### 6 GHz and Newer Generations

In the 6 GHz band (Wi-Fi 6E and later), the Wi-Fi Alliance dropped legacy options:

- Allowed: WPA3-Personal (SAE only, with H2E), WPA3-Enterprise and OWE.
- Not allowed: open without OWE, WPA2 and transition mode.
- PMF is required.

Wi-Fi 7 raises this further with newer key management (AKM 24) and GCMP-256 support. Exact rules depend on the certification version and vendor release, so read the product notes.

### Other Weaknesses Worth Knowing

- **Hole 196 (GTK abuse).** Named by AirTight Networks in 2010. Every client has the GTK, so an inside user can send fake broadcast frames, for example to poison [ARP](../03-ethernet-and-switching/05-arp.md). Client isolation and ARP inspection reduce the risk.
- **FragAttacks (2021).** Mathy Vanhoef found design flaws in frame fragmentation and aggregation, some dating back to the first 802.11. Fixed through vendor patches.
- **SSID confusion (2024).** In some setups the SSID is not fully bound to the key exchange. A client can be tricked into a different network that shares credentials. Do not reuse credentials across SSIDs of different trust levels.
- **MSCHAPv2.** Its security reduces to breaking a single key of DES, an old cipher, shown publicly in 2012. It must stay inside a tunnel with a checked server certificate, and even then, EAP-TLS is the better long-term goal.

### SAE-PK

**SAE-PK** suits places with a public password, such as a cafe. The password itself encodes a fingerprint of a public key. A fake AP that knows the password still cannot prove ownership of the private key, so clients detect the fake. Client support is limited.

### Layered Defence for a Real Company

Wi-Fi security is not one setting. A good design:

- WPA3-Enterprise or WPA2-Enterprise with EAP-TLS, PMF required, and checked server certificates pushed by device management.
- Separate SSIDs and [VLANs](../03-ethernet-and-switching/06-vlans.md) for staff, guests and IoT, with firewalls between them.
- Client isolation on guest networks.
- WIPS to spot rogue APs and deauth floods.
- WPS off, TKIP off, WEP off.
- For sensitive traffic, use end-to-end protection, such as a [VPN](../09-security/07-vpn.md) or [Zero Trust](../09-security/09-zero-trust-and-segmentation.md) access. Wi-Fi encryption stops at the AP.

## Common Mistakes

- **"WPA2 is broken because of KRACK."** KRACK was a logic flaw that patches fix. WPA2 with patches and a strong setup is still acceptable.
- **"After KRACK you must change your password."** It does not reveal the password. You must patch.
- **"Hiding the SSID is security."** The name still appears in client probes and association frames.
- **"MAC filtering stops attackers."** MAC addresses are sent in clear and are easy to copy. Modern phones also randomize theirs.
- **"WPA3 stops evil twins."** It does not on its own. If the attacker knows the shared password, they can run the same network. SAE protects the password from offline guessing, not users from a fake AP.
- **"Encryption on Wi-Fi covers the whole path."** Wi-Fi encryption ends at the AP. After that, traffic needs its own protection.
- **"The four-way handshake sends the password."** Neither the password nor the PMK crosses the air. Only nonces and MICs do.
- **"Each client has its own key in WPA2-PSK, so clients cannot spy on each other."** Each has its own PTK, but anyone with the password and the victim's handshake can compute it.
- **"Enterprise is safe by default."** PEAP without server certificate validation gives away credentials to an evil twin.
- **"WEP 128-bit has a 128-bit key."** It is a 104-bit key plus a 24-bit IV.
- **"WPA3 means stronger encryption."** WPA3-Personal still uses 128-bit AES. The main change is the handshake.
- **"PMF stops all denial of service."** It stops forged management frames, not jamming.
- **"AES is what was attacked in KRACK."** AES was fine. The attack made the system reuse a nonce.
- **"Open Wi-Fi with a portal is encrypted."** Portals control access. They do not encrypt.

## Interview Questions

**Q: Why was WEP replaced?**
A: WEP used RC4 with a 24-bit IV, so IVs repeated quickly and weak ones leaked key bytes. The key was static and shared, and CRC-32 could not stop tampering. An attacker can recover the key within minutes, so it has no real security.

**Q: What is the difference between WPA, WPA2 and WPA3?**
A: WPA was a 2003 stopgap that used TKIP on RC4 hardware. WPA2 uses AES-CCMP and is the full 802.11i standard. WPA3 keeps AES but replaces PSK with SAE, requires PMF, and adds forward secrecy, plus a 192-bit enterprise mode.

**Q: Walk me through the four-way handshake.**
A: Both sides start with the PMK. M1 gives the AP's ANonce. The client creates an SNonce, derives the PTK, and sends M2 with the SNonce and a MIC. The AP derives the same PTK, checks the MIC, and sends M3 with the encrypted GTK and its own MIC. The client installs keys and sends M4, then the AP installs keys.

**Q: What is the difference between PSK and Enterprise?**
A: PSK uses one shared password and no server, so it is simple but weak on accountability and revocation. Enterprise uses 802.1X with a RADIUS server and per-user credentials. You can disable one user, see who connected, and assign VLANs by identity.

**Q: Describe the three roles in 802.1X.**
A: The supplicant is the client. The authenticator is the AP or controller. It relays EAP but does not check credentials. The authentication server, usually RADIUS, makes the decision and sends key material to the authenticator after success.

**Q: PEAP or EAP-TLS, and why?**
A: EAP-TLS is stronger because it uses certificates on both sides, so there is no password to phish. PEAP-MSCHAPv2 is easier if you have Active Directory but no client certificates. PEAP is only safe when clients validate the server certificate, so I choose EAP-TLS when device management can deploy certificates.

**Q: Why can WPA2-PSK be cracked offline, and how does SAE fix it?**
A: In WPA2-PSK, the handshake MIC is a function of the password and public values, so an attacker can test guesses at home. SAE uses a password-authenticated exchange with fresh random values. A capture gives nothing to test, so each guess needs a live attempt with the AP.

**Q: What is forward secrecy on Wi-Fi?**
A: It means a future password leak does not expose traffic recorded earlier. In WPA2-PSK, someone with the password and a recorded handshake can decrypt that session. In SAE, each session's PMK comes from fresh random values, so the password alone cannot recover it.

**Q: What is PMF, and what does it stop?**
A: PMF (802.11w) protects deauthentication, disassociation and some action frames. It stops forged deauth attacks and fake association tricks through SA Query. It does not stop radio jamming. It is required in WPA3.

**Q: How does KRACK work, and how was it fixed?**
A: The attacker blocks M4, so the AP resends M3. The client reinstalls the same key and resets its packet number. Nonce reuse then lets the attacker decrypt or replay frames. The fix is a patch that refuses to reinstall an installed key. The password is not exposed.

**Q: Why is WPS PIN weak?**
A: The 8th digit is a checksum, and the protocol confirms the two halves separately. So there are at most 10,000 plus 1,000 tries, or 11,000 total. Once the PIN is found, the AP gives the password. The fix is to turn WPS off.

**Q: A user says Wi-Fi drops every few minutes for everyone on one floor. How do you check for a deauth attack?**
A: I capture on that channel in monitor mode and filter for subtype 0x0c and 0x0a. Many deauth frames, often with one source MAC or a strange reason code, suggest an attack. I check the WIPS logs, then enable PMF where clients support it. I also rule out interference and AP faults, since those are more common.

## Quick Recap

- Wi-Fi security gives authentication, encryption and integrity over a medium that anyone nearby can hear.
- History: WEP (broken), WPA with TKIP (stopgap), WPA2 with AES-CCMP (minimum today), WPA3 with SAE and PMF (best choice).
- PSK means one shared secret. Enterprise means 802.1X with EAP, and RADIUS for per-user login.
- The password or EAP result leads to a PMK. The four-way handshake turns it into a session PTK and delivers the GTK.
- SAE stops offline guessing and adds forward secrecy. PMF stops forged deauth frames. OWE encrypts open networks without authenticating them.
- Key attacks: evil twin (fake AP), deauth (forged kick), offline dictionary, PMKID, KRACK (key reinstallation, fixed by patch), WPS PIN (11,000 tries).
- Turn off WEP, TKIP and WPS, require PMF where you can, and validate server certificates on Enterprise clients.
