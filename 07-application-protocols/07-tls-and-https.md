# TLS and HTTPS

**Level:** Intermediate | **Prerequisites:** [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [HTTP Basics](05-http-basics.md), [Cryptography Basics](../09-security/02-cryptography-basics.md)

## In One Line

TLS (Transport Layer Security) builds a private, tamper-proof, identity-checked channel on top of a normal connection. HTTPS is plain HTTP sent inside that channel, normally on TCP port 443.

## Why It Matters

Plain HTTP is sent as readable text. Every device between you and the server can read it: the cafe Wi-Fi router, the ISP, or an attacker on the same LAN. They can also change it.

Here is a real login sent over plain HTTP, as anyone on the path would see it:

```text
POST /login HTTP/1.1
Host: www.example.com
Content-Type: application/x-www-form-urlencoded

user=alice&password=Summer2026
```

TLS solves three problems at once:

- Nobody on the path can read the data.
- Nobody on the path can change the data without being caught.
- You know you are talking to the real `www.example.com`, not a fake.

Interviewers love this topic because it joins many areas: TCP, DNS, cryptography, HTTP, load balancers and troubleshooting. Expect questions like these:

- Walk me through the TLS handshake.
- What changed from TLS 1.2 to TLS 1.3?
- What is forward secrecy?
- Why does a site work in the browser but fail in `curl`?

## Simple Explanation

Imagine you must send secret letters to your bank, through a postal system you do not trust.

1. **Check identity.** The bank shows you an ID card issued by a government you trust.
2. **Agree on a secret.** You and the bank do a clever trick to create a key that only the two of you know.
3. **Use a locked box.** Every letter now travels in a locked box. Only your shared key opens it.
4. **Use a tamper seal.** Every box has a seal. If anyone opens or changes it, you will notice.

TLS does these same four things in a few milliseconds. The first two happen in the **handshake**, which is the opening conversation where both sides set up the connection. The last two happen for every piece of data after that.

**Where the analogy stops being true:** a real post office sees your letter once. In TLS the attacker can record everything, replay it, delete it and inject new data. TLS is designed for this stronger attacker. Also, the "ID card" (the certificate) is public and proves nothing by itself. The server must also prove it holds a matching secret key.

Key words used in the rest of this chapter:

| Term | Plain meaning |
|---|---|
| Plaintext | Readable data before encryption |
| Ciphertext | Scrambled data after encryption |
| Key | A secret number that controls encryption |
| Symmetric encryption | One shared key both locks and unlocks. Fast. Example: AES |
| Asymmetric encryption | A key pair. A **public key** that anyone may see, and a **private key** kept secret |
| Hash | A one-way fingerprint of data. Example: SHA-256 |
| Digital signature | Proof made with a private key that anyone can check with the public key |
| Cipher | An encryption algorithm |
| Certificate | A signed document that links a public key to a name |

For the full cryptography background, see [Cryptography Basics](../09-security/02-cryptography-basics.md).

## How It Works

### What TLS gives you

| Property | Question it answers | How TLS does it |
|---|---|---|
| Confidentiality | Can anyone else read this? | Symmetric encryption such as AES-GCM or ChaCha20-Poly1305 |
| Integrity | Was this changed on the way? | An authentication tag on every record. A tag is a small value that only the key holders can compute |
| Authentication | Am I talking to the right party? | A certificate plus a signature made with the private key |

Authentication is normally one-way: the client checks the server, and the server does not check the client at the TLS level. The client usually logs in later with a password or a token. Two-way checking is called mutual TLS, covered below.

### What TLS does not give you

TLS protects data on the wire, and nothing more.

- **It does not hide the IP addresses or ports.** Routers need them.
- **It usually does not hide the hostname.** The name is often visible in the SNI field and in DNS queries.
- **It does not hide size and timing.** An observer can guess a lot from traffic patterns.
- **It does not make a site honest.** A phishing site can have a valid certificate. Chrome replaced the lock icon with a neutral icon in 2023 because people thought the lock meant "safe".
- **It does not protect data at rest.** Once data reaches the server, TLS has finished its job.

### Where TLS sits

TLS runs above a reliable transport and below the application.

```text
+-----------------------------------+
| HTTP (or SMTP, IMAP, LDAP, ...)   |  application
+-----------------------------------+
| TLS (encrypts and protects)       |  between L4 and L7
+-----------------------------------+
| TCP (reliable byte stream)        |  transport
+-----------------------------------+
| IP (addresses and routing)        |  network
+-----------------------------------+
```

TLS does not fit neatly into one OSI layer. If asked, say that it sits between the transport and application layers, and that the OSI model maps to it only loosely.

Many protocols use TLS. A few well-known ones:

| Protocol | Port | Notes |
|---|---|---|
| HTTPS | 443 | HTTP over TLS |
| SMTP submission | 587 | Starts plain, upgrades with STARTTLS |
| SMTPS | 465 | TLS from the first byte |
| IMAPS | 993 | Mail reading |
| LDAPS | 636 | Directory access |
| DNS over TLS | 853 | See [DNS Advanced](03-dns-advanced.md) |

HTTP/3 is a special case. It runs on QUIC over UDP, and QUIC has the TLS 1.3 handshake built in. See [QUIC](../06-transport-layer/08-quic.md).

### TLS versions

SSL (Secure Sockets Layer) is the old name. People still say "SSL certificate", but real SSL is dead.

| Version | Year | Status in 2026 |
|---|---|---|
| SSL 2.0 | 1995 | Broken. Prohibited by RFC 6176 |
| SSL 3.0 | 1996 | Broken. Prohibited by RFC 7568 |
| TLS 1.0 | 1999 | Deprecated by RFC 8996 |
| TLS 1.1 | 2006 | Deprecated by RFC 8996 |
| TLS 1.2 | 2008 (RFC 5246) | Still fine with strong ciphers |
| TLS 1.3 | 2018 (RFC 8446) | Current. Use it wherever you can |

### Why TLS uses two kinds of encryption

Asymmetric cryptography can work between strangers, but it is slow. Symmetric encryption is very fast, but both sides must already share a key.

So TLS uses each for what it does best:

1. The handshake uses asymmetric cryptography to prove identity and agree on a key.
2. All real data uses symmetric encryption with that key.

### Certificates

A **certificate** is a small signed file that says: "this public key belongs to this name, until this date". The format is called **X.509**, profiled for the internet in RFC 5280.

**Analogy:** a passport. It has your name, your photo, an expiry date and an issuer that others trust.

**Where it stops being true:** a passport is hard to copy and you keep it private. A certificate is public. Anyone can download it and send it to you. Holding the certificate proves nothing. The server proves its identity by using the private key, which is like the face matching the photo.

Main fields of a certificate:

| Field | Meaning | Example |
|---|---|---|
| Serial number | Unique number from the issuer | `05:A3:C1:F2:...` |
| Issuer | Who signed it | `CN=R12, O=Let's Encrypt` |
| Validity | Not Before and Not After dates | Oct 5 2026 to Jan 3 2027 |
| Subject | Who it was issued to | `CN=www.example.com` |
| Public key | The server's public key | RSA 2048-bit or ECDSA P-256 |
| Subject Alternative Name (SAN) | The list of names it is valid for | `DNS:example.com, DNS:www.example.com` |
| Key Usage, Extended Key Usage | What the key may do | `TLS Web Server Authentication` |
| Basic Constraints | Is this a CA certificate? | `CA:FALSE` |
| Authority Information Access (AIA) | URL of the issuer's certificate | `http://r12.i.lencr.org/` |
| CRL Distribution Points | Where to find the revocation list | A URL |
| Signature | The issuer's signature over all of the above | `sha256WithRSAEncryption` |

Rules worth knowing:

- **Browsers check the SAN only.** They ignore the old Common Name (CN) field. A certificate with a correct CN but no matching SAN fails.
- **A wildcard covers exactly one label.** `*.example.com` matches `a.example.com`. It does not match `a.b.example.com` or `example.com` itself.
- **Validation level** tells how much the CA checked. DV (Domain Validated) proves control of the domain. OV (Organization Validated) and EV (Extended Validation) also check the company. Encryption strength is the same for all three. Browsers stopped showing a special EV indicator in 2019.

### The CA chain

A **CA (Certificate Authority)** is an organisation trusted to sign certificates. Your device already trusts a set of **root CAs**. This list is the **trust store**.

Roots are too valuable to use every day. Their private keys are kept offline, and they sign a few **intermediate CAs**. The intermediates sign the server certificates, which are also called **leaf** certificates.

```text
  Trust store on your device
  +--------------------------------+
  | Root CA: ISRG Root X1          |  self-signed, already trusted
  +--------------------------------+
                  | signs
                  v
  +--------------------------------+
  | Intermediate CA: R12           |  sent by the server
  +--------------------------------+
                  | signs
                  v
  +--------------------------------+
  | Leaf: www.example.com          |  sent by the server
  +--------------------------------+
```

**Analogy:** you do not know the clerk at a local office, but the clerk has a letter from the ministry, and the ministry has a letter from the government you trust.

**Where it stops being true:** letters can be judged by eye. Here the check is pure mathematics. Each signature either verifies or it does not. There is no "probably fine".

The server must send the leaf and the intermediates. It should not need to send the root, because the client already has it. A server that forgets the intermediate causes one of the most common real-world TLS failures. See the worked example below.

If an intermediate key is stolen, the CA can revoke that intermediate and keep the root. That is why the middle layer exists.

Trust stores come from Mozilla, Microsoft, Apple and Google (the Chrome Root Store). On a machine you can see them here:

```bash
# Linux (Debian, Ubuntu): the combined trust store file
ls -l /etc/ssl/certs/ca-certificates.crt
```

```powershell
# Windows: list trusted root CAs for the machine
Get-ChildItem Cert:\LocalMachine\Root | Select-Object -First 5 Subject, NotAfter
```

### PKI: the whole system

**PKI (Public Key Infrastructure)** is the full system around certificates: the CAs, the policies, the trust stores, the logs and the revocation services.

| Part | Role |
|---|---|
| Root CA | Top of trust, key kept offline |
| Intermediate CA | Signs certificates each day |
| Trust store | The list of roots a device trusts |
| CA/Browser Forum | Group of CAs and browser makers. Writes the Baseline Requirements that public CAs must follow |
| Certificate Transparency logs | Public append-only logs of issued certificates |
| Revocation services | Tell clients that a certificate was cancelled |

How a server gets a certificate:

1. The server creates a key pair. The private key never leaves the server.
2. It creates a **CSR (Certificate Signing Request)**. This contains the public key and the wanted names, signed by the private key.
3. The CA checks that the applicant controls the domain.
4. The CA checks the domain's **CAA** DNS record, which says which CAs may issue for it. See [DNS Record Types](02-dns-records.md).
5. The CA signs the certificate and logs it.
6. The admin installs the certificate, the intermediates and the private key.

Companies also run **private CAs** for internal systems. Their root is added to company devices by policy. Public clients do not trust it, and they should not.

### How a client verifies a certificate

For a server certificate, the client checks all of these:

1. **Chain:** can I build a path from the leaf up to a root in my trust store?
2. **Signatures:** does each certificate verify with its issuer's public key?
3. **Dates:** is the current time inside every certificate's validity period?
4. **Name:** does the hostname I asked for match a SAN entry? (RFC 9525 covers this.)
5. **Usage:** is the leaf allowed for server authentication? Are the upper certificates real CAs?
6. **Revocation:** has it been cancelled? (How well this works varies. See below.)
7. **Certificate Transparency:** browsers such as Chrome and Safari also require proof of logging.
8. **Possession:** did the server sign this handshake with the matching private key?

Step 8 is what stops someone from replaying a stolen public certificate.

### Certificate lifetimes are shrinking

Shorter lifetimes limit the damage from a stolen key. The CA/Browser Forum set this schedule in 2025 for publicly trusted TLS certificates:

| Issued from | Maximum lifetime |
|---|---|
| September 2020 (browser rule, before the Forum adopted it) | 398 days |
| March 15, 2026 | 200 days |
| March 15, 2027 | 100 days |
| March 15, 2029 | 47 days |

Manual renewal cannot keep up with this. Automation is now required in practice. Private CAs set their own lifetimes.

### Key exchange: making a secret in public

The two sides must agree on a key while an attacker watches every message. **Diffie-Hellman** (DH) is the trick that makes this possible. **ECDHE** is the modern version: Elliptic Curve Diffie-Hellman, Ephemeral. "Ephemeral" means the key pair is used once and thrown away.

**Analogy:** paint mixing.

1. Both agree in public on a yellow base colour.
2. Each adds a secret colour and sends the mix to the other.
3. Each adds their own secret colour to the mix they received.
4. Both end with the same brown. A watcher saw only the mixes and cannot unmix them.

**Where it stops being true:** real security comes from a hard maths problem, not paint. It also gives no identity by itself. An attacker in the middle could mix paint with both sides. That is why the server signs its share using its certificate key.

A toy example with tiny numbers (real values are far larger):

```text
Public:  p = 23, g = 5
Client secret a = 6    sends A = 5^6  mod 23 = 8
Server secret b = 15   sends B = 5^15 mod 23 = 19
Client computes 19^6  mod 23 = 2
Server computes 8^15  mod 23 = 2
Shared secret = 2. It was never sent on the wire.
```

### Forward secrecy

**Forward secrecy** means that stealing the server's long-term private key later does not expose past recorded traffic.

**Analogy:** a spy burns the code sheet after each message. Capturing the spy tomorrow does not reveal yesterday's messages.

**Where it stops being true:** a code sheet is burned once. A real server must delete keys correctly from memory, and session tickets can keep key material alive for longer (see Going Deeper).

Compare the two key exchange styles:

| | Static RSA key exchange (old) | ECDHE |
|---|---|---|
| How the secret travels | Client encrypts it with the server's RSA public key | Never sent. Both sides compute it |
| Key used | Same long-term key for every session | Fresh key pair per session |
| Attacker records traffic, steals key a year later | Can decrypt everything recorded | Cannot decrypt anything |
| Forward secrecy | No | Yes |

In modern TLS the certificate key **signs**. It does not encrypt the session key. This surprises many candidates.

TLS 1.3 removed static RSA key exchange completely. TLS 1.2 can do either, so server configuration matters.

### TLS 1.2 full handshake

This flow uses an ECDHE suite, the normal safe choice for TLS 1.2. It takes **2 round trips**. A **round trip (RTT)** is the time for a message to go there and back.

```text
Client                                                Server
  |                                                      |
  |--- ClientHello ------------------------------------->|
  |      versions, client random, cipher list,           |
  |      SNI, ALPN, supported curves                     |
  |                                                      |
  |<-- ServerHello --------------------------------------|
  |      chosen version, chosen cipher, server random    |
  |<-- Certificate --------------------------------------|
  |<-- ServerKeyExchange (signed ECDHE public value) ----|
  |<-- ServerHelloDone ----------------------------------|
  |                                                      |
  |--- ClientKeyExchange (ECDHE public value) ---------->|
  |--- ChangeCipherSpec -------------------------------->|
  |--- Finished (encrypted) ---------------------------->|
  |                                                      |
  |<-- ChangeCipherSpec ---------------------------------|
  |<-- Finished (encrypted) -----------------------------|
  |                                                      |
  |<========== encrypted application data ==============>|
```

Step by step:

1. **ClientHello.** The client lists the versions and cipher suites it supports, and sends a 32-byte random number.
2. **ServerHello.** The server picks one version and one suite, and sends its own 32-byte random.
3. **Certificate.** The server sends its certificate chain, in clear text.
4. **ServerKeyExchange.** The server sends its ECDHE public value, and signs it (together with both randoms) with its private key. This proves it owns the certificate.
5. **ServerHelloDone.** The server says it is finished for now.
6. **ClientKeyExchange.** The client verifies the certificate and signature, then sends its ECDHE public value.
7. **Key creation.** Both sides compute the same **pre-master secret**. They mix it with both randoms to get the **master secret**, then derive the session keys.
8. **ChangeCipherSpec.** Each side says "I now switch to encryption".
9. **Finished.** Each side sends a value computed over the whole handshake so far. If an attacker changed any message, the values will not match and the connection fails.

### TLS 1.3 full handshake

TLS 1.3 takes **1 round trip**. The trick is simple: the client guesses a key exchange group and sends its public value in the very first message. **Curly braces** below mean the message is encrypted.

```text
Client                                                Server
  |                                                      |
  |--- ClientHello ------------------------------------->|
  |      supported_versions, client random, ciphers,     |
  |      key_share (client ECDHE value), SNI, ALPN       |
  |                                                      |
  |<-- ServerHello --------------------------------------|
  |      server random, chosen cipher,                   |
  |      key_share (server ECDHE value)                  |
  |        ... both sides can now compute keys ...       |
  |<-- {EncryptedExtensions} ----------------------------|
  |<-- {Certificate} ------------------------------------|
  |<-- {CertificateVerify} ------------------------------|
  |<-- {Finished} ---------------------------------------|
  |                                                      |
  |--- {Finished} -------------------------------------->|
  |<========== encrypted application data ==============>|
  |                                                      |
  |<-- {NewSessionTicket} (later, for resumption) -------|
```

Step by step:

1. **ClientHello.** It has the same goal as before, but now also carries `key_share`, which holds one or more ECDHE public values. A common choice is X25519.
2. **ServerHello.** The server picks a suite and sends its own key share. Only these two hello messages are in clear text.
3. **Key creation.** Both sides compute the shared secret and derive handshake keys.
4. **EncryptedExtensions.** Remaining settings, such as the chosen ALPN value, now go encrypted.
5. **Certificate.** The chain is encrypted, so a passive observer cannot read it.
6. **CertificateVerify.** The server signs a hash of the whole handshake so far with its private key. This proves possession of the key.
7. **Finished.** A keyed check over the handshake. This proves nothing was changed.
8. **Client Finished.** The client checks everything and sends its own Finished. It can send the HTTP request right behind it.

If the client guessed wrong and the server supports none of the offered key shares, the server replies with a **HelloRetryRequest**. It names a group to use, and the client sends a new ClientHello. This costs one extra round trip.

TLS 1.3 also hides its version. For compatibility, the old version fields say TLS 1.2 (0x0303), and the real version goes in the `supported_versions` extension. Old middleboxes, which are network boxes such as firewalls that inspect traffic, broke when they saw a new version number.

### TLS 1.2 vs TLS 1.3

| Feature | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Full handshake | 2 RTT | 1 RTT |
| Resumption | 1 RTT | 1 RTT, or 0-RTT with early data |
| Forward secrecy | Optional, depends on suite | Always, except the rare PSK-only mode |
| Static RSA key exchange | Allowed | Removed |
| Server and client certificates on the wire | Clear text | Encrypted |
| Cipher modes | CBC and AEAD | AEAD only |
| Weak items (RC4, 3DES, MD5, SHA-1 for signatures) | Exist unless disabled | Removed |
| Compression | Allowed (unsafe) | Removed |
| Renegotiation | Exists | Removed, replaced by KeyUpdate |
| Cipher suite list | Hundreds of registered suites | 5 defined in RFC 8446 |

### Cipher suites

A **cipher suite** is a named set of algorithms for one connection. Think of it as a set menu: pick one name, and you get a fixed combination of dishes.

**Where the analogy stops being true:** a menu is chosen by taste. Here both sides need a suite in common, or the handshake fails.

A TLS 1.2 suite name lists four jobs:

```text
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    |     |        |       |   |
    |     |        |       |   +-- SHA256: hash for key derivation
    |     |        |       +------ GCM: mode, gives integrity too
    |     |        +-------------- AES_128: cipher, 128-bit key
    |     +----------------------- RSA: server signs with an RSA key
    +----------------------------- ECDHE: key exchange
```

A TLS 1.3 suite name is shorter:

```text
TLS_AES_128_GCM_SHA256
```

It names only the cipher and the hash. Key exchange and signature are negotiated apart, in the `key_share` and `signature_algorithms` extensions.

The five TLS 1.3 suites:

| Suite | Code | Common use |
|---|---|---|
| `TLS_AES_128_GCM_SHA256` | 0x1301 | Mandatory to implement, very common |
| `TLS_AES_256_GCM_SHA384` | 0x1302 | Very common |
| `TLS_CHACHA20_POLY1305_SHA256` | 0x1303 | Good on devices without AES hardware |
| `TLS_AES_128_CCM_SHA256` | 0x1304 | Constrained devices |
| `TLS_AES_128_CCM_8_SHA256` | 0x1305 | Constrained devices, shorter tag |

**AEAD (Authenticated Encryption with Associated Data)** means the cipher encrypts and protects integrity in one step. GCM, CCM and Poly1305 are AEAD. The older CBC mode with a separate check caused many attacks, which is why TLS 1.3 dropped it.

For TLS 1.2, a good server allows only ECDHE key exchange with AEAD ciphers. Avoid anything with `RSA_WITH`, `CBC`, `3DES`, `RC4` or `NULL`.

```bash
# Linux: show which TLS 1.3 suites and strong TLS 1.2 suites OpenSSL knows
openssl ciphers -v 'ECDHE+AESGCM'
```

```powershell
# Windows: list cipher suites enabled in Schannel, the built-in TLS library
Get-TlsCipherSuite | Select-Object -First 5 Name
```

### SNI: which site do you want?

Many websites can share one IP address. In HTTP that is simple, because the `Host` header names the site. But the certificate is chosen before HTTP starts. How does the server know which certificate to send?

**SNI (Server Name Indication)** solves this. It is a TLS extension (RFC 6066) that carries the hostname in the ClientHello.

**Analogy:** you arrive at an office building that holds 100 companies. At the front desk you say "I am here for Acme".

**Where it stops being true:** at a real desk only the receptionist hears you. SNI is in clear text, so everyone on the path can read it.

Two results follow from this:

- One IP can serve thousands of sites with different certificates.
- Firewalls and ISPs can see or block sites by name, even with HTTPS.

```text
ClientHello in clear text, seen by anyone on the path:

  Client 192.168.1.50:51514  --->  Server 203.0.113.10:443
  server_name = shop.example.com    <-- readable
```

If a client sends no SNI, the server usually returns its default certificate. This often causes a name mismatch.

### ECH: hiding the name

**ECH (Encrypted Client Hello)** closes the SNI gap. It is specified by the IETF TLS working group, and it replaces an older idea called ESNI.

ECH splits the hello in two:

- The **outer ClientHello** is visible. It carries a harmless shared public name such as `cover.example.com`.
- The **inner ClientHello** is encrypted. It carries the real name.

```text
Outer ClientHello (clear text)
  SNI: cover.example.com
  +--------------------------------------+
  | Inner ClientHello (encrypted)        |
  |   SNI: shop.example.com              |
  |   ALPN, key shares, other settings   |
  +--------------------------------------+
```

The client gets the server's ECH public key from a DNS **HTTPS record** (RFC 9460).

**Analogy:** you hand the front desk a sealed note. Only the building manager can open it, and it names the company you are visiting.

**Where it stops being true:** it helps only when many sites share the same front. If one small site is alone on an IP, the IP itself reveals the site.

Limits you should mention in an interview:

- DNS must also be encrypted (for example DoH), or the name leaks in the DNS query.
- It works best on big shared platforms such as CDNs.
- Support in 2026 is growing but is not universal. Firefox and Chrome support it, and some networks block it.
- Company security tools that filter by SNI lose that view.

### ALPN: which application protocol?

**ALPN (Application-Layer Protocol Negotiation, RFC 7301)** lets client and server agree on the application protocol during the handshake. The client offers `h2, http/1.1`, and the server picks one. This saves a round trip and is how HTTP/2 is chosen on TLS. See [HTTP/1.1, HTTP/2 and HTTP/3](06-http-versions.md).

### Session resumption

A full handshake costs signature work and round trips. **Session resumption** lets a client that connected recently skip most of it.

**Analogy:** a stamp on your hand at a club. You showed ID once, and the stamp gets you back in without another ID check.

**Where it stops being true:** a stamp can be copied by looking at it. A TLS ticket is useless without the secret key that goes with it, which is never sent on the wire.

| Method | Version | Who stores state | How it works |
|---|---|---|---|
| Session ID | TLS 1.2 | Server keeps a cache | Client sends the old ID, and server finds the old keys |
| Session ticket (RFC 5077) | TLS 1.2 | Client keeps it | Server encrypts session state into a ticket that the client stores |
| PSK from NewSessionTicket | TLS 1.3 | Client keeps it, server may too | After the handshake, server sends a ticket. Later the client uses it as a pre-shared key |

**PSK (Pre-Shared Key)** means a key that both sides already know. In TLS 1.3 tickets can last at most 7 days (RFC 8446 limit). The usual mode, `psk_dhe_ke`, still runs a fresh ECDHE, so forward secrecy is kept.

TLS 1.3 resumption skips the certificate and its signature check:

```text
Client                                                Server
  |--- ClientHello (key_share, pre_shared_key) --------->|
  |<-- ServerHello (key_share, pre_shared_key) ----------|
  |<-- {EncryptedExtensions} ----------------------------|
  |<-- {Finished} ---------------------------------------|
  |--- {Finished} -------------------------------------->|
  |<========== encrypted application data ==============>|
```

Behind a load balancer, resumption needs thought. All servers must share ticket keys or a session cache, or a client that lands on another server falls back to a full handshake. That is not an error, just slower.

### 0-RTT early data

With TLS 1.3 resumption, the client may send application data **inside its first flight**, before any reply. This is **0-RTT** or early data.

```text
Client                                                Server
  |--- ClientHello + PSK + early_data ------------------>|
  |--- (GET /index.html, encrypted with early key) ----->|
  |<-- ServerHello ... {Finished} -----------------------|
  |--- {EndOfEarlyData} {Finished} --------------------->|
  |<========== encrypted application data ==============>|
```

It has a serious cost: **replay**. An attacker can copy the first flight and send it again. The server may process it twice. Early data also has weaker forward secrecy, since its key depends only on the PSK.

So use 0-RTT only for safe, repeatable requests such as a plain `GET`. Never use it for "transfer $100". RFC 8470 defines the `Early-Data: 1` header and the `425 Too Early` status so that servers can refuse to act on such requests.

### Revocation: cancelling a certificate

If a private key leaks, the certificate must stop working before its expiry date. This is **revocation**.

**Analogy:** a lost credit card. The card still looks valid, so the shop needs some way to learn that it was cancelled.

**Where it stops being true:** a card payment always checks with the bank and fails if the bank says no. TLS clients have often gone ahead when they could not reach the checker. This is called **soft fail**.

| Method | How it works | Problem |
|---|---|---|
| CRL (Certificate Revocation List) | CA publishes a signed list of revoked serial numbers. Client downloads it | Lists can be large and out of date |
| OCSP (Online Certificate Status Protocol, RFC 6960) | Client asks the CA "is this serial OK?" | Adds delay, tells the CA which sites you visit, usually soft fails |
| OCSP stapling | Server fetches a signed, time-limited OCSP reply and "staples" it to the handshake | Server must keep it fresh |
| Must-Staple (RFC 7633) | Certificate flag: refuse the connection if no staple is present | Rarely used, and needs a CA that supports OCSP |
| Short-lived certificates | Expire so soon that revocation hardly matters | Requires solid automation |

```text
OCSP (privacy leak)                OCSP stapling

Client ---- is it OK? ---> CA      Server --- is it OK? ---> CA
Client <--- good --------- CA      Server <-- signed "good" CA
Client ---- TLS --------> Server   Client <-- TLS + staple - Server
```

**Stapling analogy:** instead of each visitor phoning your doctor, you carry a fresh stamped note from the doctor.

Current practice has moved away from live OCSP:

- The CA/Browser Forum made OCSP optional for public CAs and kept CRLs mandatory.
- Let's Encrypt shut down its OCSP service in August 2025 and uses CRLs. Privacy was a main reason.
- Browsers mostly do not query OCSP live. Chrome uses CRLSets, a compact list it pushes to browsers. Firefox uses CRLite, a compressed form of the revocation data of all CAs.
- Shrinking lifetimes reduce the need further.

So stapling knowledge is still asked in interviews, but do not assume every site supports it in 2026.

### HSTS: never use plain HTTP

A user types `example.com`. The browser first tries `http://`, and the server redirects to `https://`. That first plain request is a weak spot. An attacker in the middle can keep the user on HTTP and speak HTTPS to the server. This is **SSL stripping**.

**HSTS (HTTP Strict Transport Security, RFC 6797)** fixes it. The server sends this header over HTTPS:

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

| Part | Meaning |
|---|---|
| `max-age=31536000` | Remember this rule for one year (in seconds) |
| `includeSubDomains` | Apply to every subdomain too |
| `preload` | Owner agrees to be placed in the browser's built-in list |

From then on the browser:

1. Rewrites every `http://` request for that site to `https://` before sending anything.
2. Turns certificate errors into hard failures. There is no "proceed anyway" button.

**Analogy:** a note you keep in your wallet that says "this bank never sends postcards. If you get one, it is fake".

**Where it stops being true:** you must visit the bank once to get the note. The first visit is still exposed.

**Preloading** solves this first-visit problem. Browsers ship a built-in list of HSTS sites (see hstspreload.org). Some whole TLDs are on it, such as `.dev` and `.app`.

Be careful with rollout. `includeSubDomains` also covers any internal subdomain that has no HTTPS. Start with a short `max-age`, then raise it. Leaving the preload list is slow, so plan before you join it.

### mTLS: both sides show ID

In normal TLS only the server has a certificate. In **mutual TLS (mTLS)** the client has one too.

**Analogy:** at a secure building, the guard shows a badge to you, and you show yours to the guard.

**Where it stops being true:** real badges can be checked by eye. Here each side must prove it holds the private key, so a copied certificate is useless.

```text
Client                                                Server
  |--- ClientHello ------------------------------------->|
  |<-- ServerHello --------------------------------------|
  |<-- {EncryptedExtensions} ----------------------------|
  |<-- {CertificateRequest} -----------------------------|
  |<-- {Certificate} {CertificateVerify} {Finished} -----|
  |--- {Certificate} {CertificateVerify} {Finished} ---->|
  |<========== encrypted application data ==============>|
```

The server sends **CertificateRequest**. The client answers with its own certificate and a signature. The server checks it against a CA it trusts, usually a private CA.

Common uses:

- Service-to-service traffic in a mesh. See [Service Mesh and eBPF](../11-cloud-and-containers/10-service-mesh-and-ebpf.md).
- Zero trust networks, where the network location grants no trust. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).
- Partner and banking APIs, and IoT devices.
- Enterprise Wi-Fi with EAP-TLS, and many VPNs.

The hard part is operations: issuing, rotating and revoking a certificate per client. Service meshes automate this with short-lived certificates.

mTLS proves **who** the client is. It does not say **what** that client may do. You still need authorization rules.

### Let's Encrypt and ACME

**Let's Encrypt** is a free public CA run by the nonprofit ISRG (Internet Security Research Group). It has issued certificates since 2015 and issues DV certificates only.

**ACME (Automatic Certificate Management Environment, RFC 8555)** is the protocol it uses. Software requests, proves and renews certificates with no human involved. Many other CAs support ACME too.

**Analogy:** a passport robot. It says "prove this is your house by placing this exact code on your front door". Once it sees the code, it prints your document.

**Where it stops being true:** it proves control at that moment, not who you are. ACME DV says nothing about your legal identity.

The three common challenges:

| Challenge | How you prove control | Notes |
|---|---|---|
| HTTP-01 | Serve a token at `http://DOMAIN/.well-known/acme-challenge/TOKEN` | Port 80 must be reachable. No wildcards |
| DNS-01 | Create a TXT record at `_acme-challenge.DOMAIN` | Works for wildcards and for hosts not on the internet. Needs a DNS API for automation |
| TLS-ALPN-01 (RFC 8737) | Answer on port 443 with a special certificate | Useful when port 80 is closed |

The flow:

```text
ACME client (certbot)                         ACME server (CA)
  |                                                      |
  |--- new order: example.com, www.example.com --------->|
  |<-- challenge token for each name --------------------|
  |                                                      |
  |  writes the token where the CA can fetch it          |
  |--- ready, please check ----------------------------->|
  |<=== CA fetches token over HTTP from many places =====|
  |                                                      |
  |--- finalize: here is my CSR ------------------------>|
  |<-- signed certificate and chain ---------------------|
```

Each request is signed with an ACME **account key**, which is separate from the certificate key. The CA checks from several network locations, which makes a local network attack harder.

Lifetimes: Let's Encrypt certificates last 90 days today. It has announced a move to 45 days by 2028, and it offers an optional profile with 6-day certificates. Check its site for the current state.

Renewal should run on its own. Certbot installs a timer on most Linux systems. In Kubernetes, cert-manager does the same job.

### Common errors

| Symptom | Cause | First check |
|---|---|---|
| `NET::ERR_CERT_DATE_INVALID`, OpenSSL error 10 | Certificate expired, or the client clock is wrong | `openssl x509 -noout -dates`, then the client's clock |
| `NET::ERR_CERT_COMMON_NAME_INVALID`, `hostname mismatch` | Hostname is not in the SAN, or the wrong default certificate was served | SAN list, and whether SNI was sent |
| `NET::ERR_CERT_AUTHORITY_INVALID`, OpenSSL error 18 or 19 | Self-signed or private root | Is the root in the trust store? |
| OpenSSL errors 20 and 21, `unable to get local issuer certificate` | Missing intermediate, or the client's trust store is out of date | Number of certificates in `-showcerts` |
| `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`, alert 70 or 40 | No version or cipher in common | Test with `-tls1_2` and `-tls1_3` |
| `wrong version number` | Client speaks TLS to a port that is not TLS, such as plain HTTP | Correct port? Plain HTTP on 443 by mistake? |
| `certificate required`, alert 116, nginx `400 No required SSL certificate was sent` | mTLS is on, but the client sent no certificate | Add `-cert` and `-key` |
| Mixed content warning | HTTPS page loads a script or image over HTTP | Browser console |
| Handshake hangs after ClientHello | Firewall or MTU problem with large packets | See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md) |
| Certificate shows a company CA, not the expected one | Corporate TLS inspection proxy | Issuer field |

For practice, the public site badssl.com offers broken test hosts such as `expired.badssl.com` and `incomplete-chain.badssl.com`.

### openssl s_client: your main tool

`openssl s_client` is a simple TLS client for testing. It prints the full handshake result. It is on most Linux systems. On Windows it comes with Git for Windows or a separate OpenSSL install.

```bash
# Linux: basic check. </dev/null closes the connection after the handshake
openssl s_client -connect www.example.com:443 -servername www.example.com </dev/null

# Show every certificate the server sends
openssl s_client -connect www.example.com:443 -servername www.example.com -showcerts </dev/null

# Force a TLS version
openssl s_client -connect www.example.com:443 -tls1_2 </dev/null
openssl s_client -connect www.example.com:443 -tls1_3 </dev/null

# Offer ALPN values
openssl s_client -connect www.example.com:443 -alpn h2,http/1.1 </dev/null

# Ask for an OCSP staple
openssl s_client -connect www.example.com:443 -status </dev/null

# Check the hostname too, and stop on any verify failure
openssl s_client -connect www.example.com:443 -verify_hostname www.example.com -verify_return_error </dev/null

# Upgrade a mail connection with STARTTLS
openssl s_client -connect mail.example.com:587 -starttls smtp </dev/null

# Send a client certificate (mTLS)
openssl s_client -connect api.example.com:443 -cert client.crt -key client.key </dev/null
```

Two details catch people out:

- By default `s_client` **does not check the hostname**. `Verify return code: 0 (ok)` only means the chain is good. Use `-verify_hostname`.
- OpenSSL 1.1.1 and later send SNI on their own when `-connect` has a name. With an IP address they send none, so add `-servername`.

On Windows without OpenSSL, use `curl.exe` or .NET:

```powershell
# Windows: curl.exe is built in. Do not type plain curl in Windows PowerShell 5.1,
# where it is an alias for Invoke-WebRequest.
curl.exe -vI https://www.example.com

# Windows: open a TLS connection and read the certificate with .NET
$tcp = [System.Net.Sockets.TcpClient]::new("www.example.com", 443)
$ssl = [System.Net.Security.SslStream]::new($tcp.GetStream())
$ssl.AuthenticateAsClient("www.example.com")
$ssl.SslProtocol
$ssl.RemoteCertificate.Subject
$ssl.RemoteCertificate.Issuer
$ssl.RemoteCertificate.GetExpirationDateString()
$ssl.Dispose(); $tcp.Dispose()
```

## Example

All examples use the same setup. The client is `192.168.1.50`. The server is `www.example.com` at `203.0.113.10`. The round-trip time between them is 40 ms. Names and dates are illustrative, and real output changes slightly with the tool version.

### Example 1: what each handshake costs

The time from the first SYN packet until data can flow, with 40 ms RTT. Times ignore server processing.

| Setup | Client can send GET at | First response byte at |
|---|---|---|
| TCP + TLS 1.2 full | 120 ms | 160 ms |
| TCP + TLS 1.2 resumed | 80 ms | 120 ms |
| TCP + TLS 1.3 full | 80 ms | 120 ms |
| TCP + TLS 1.3 resumed | 80 ms | 120 ms (saves CPU, not RTT) |
| TCP + TLS 1.3 0-RTT | 40 ms | 80 ms |
| QUIC first connection | 40 ms | 80 ms |

TCP + TLS 1.3 worked through:

```text
Time    Client                                   Server
   0    SYN -------------------------------------->
  40    <---------------------------------- SYN-ACK
  40    ACK + ClientHello ------------------------>
  80    <---- ServerHello ... {Finished}
  80    {Finished} + GET / ----------------------->
 120    <---------------------------- HTTP response
```

### Example 2: watch a handshake with curl

```bash
# Linux: -v shows the handshake, -I sends a HEAD request
curl -vI https://www.example.com
```

```text
* Connected to www.example.com (203.0.113.10) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / RSASSA-PSS
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=www.example.com
*  start date: Oct  5 08:00:00 2026 GMT
*  expire date: Jan  3 07:59:59 2027 GMT
*  subjectAltName: host "www.example.com" matched cert's "www.example.com"
*  issuer: C=US; O=Let's Encrypt; CN=R12
*  SSL certificate verify ok.
```

This output matches the TLS 1.3 diagram message for message. The numbers in brackets are the real handshake message type codes.

### Example 3: read the output of s_client

```bash
# Linux
openssl s_client -connect www.example.com:443 -servername www.example.com -alpn h2,http/1.1 </dev/null
```

```text
CONNECTED(00000003)
depth=2 C = US, O = Internet Security Research Group, CN = ISRG Root X1
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = R12
verify return:1
depth=0 CN = www.example.com
verify return:1
---
Certificate chain
 0 s:CN = www.example.com
   i:C = US, O = Let's Encrypt, CN = R12
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Oct  5 08:00:00 2026 GMT; NotAfter: Jan  3 07:59:59 2027 GMT
 1 s:C = US, O = Let's Encrypt, CN = R12
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 13 00:00:00 2024 GMT; NotAfter: Mar 12 23:59:59 2027 GMT
---
Server certificate
subject=CN = www.example.com
issuer=C = US, O = Let's Encrypt, CN = R12
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3114 bytes and written 412 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
ALPN protocol: h2
Early data was not sent
Verify return code: 0 (ok)
---
```

How to read it:

| Line | Meaning |
|---|---|
| `depth=2`, `depth=1`, `depth=0` | Chain check from root down to leaf |
| `Certificate chain` entries 0 and 1 | Leaf and intermediate were sent. The root was not, which is correct |
| `s:` and `i:` | Subject and issuer. Each `i:` must equal the next `s:` |
| `Server Temp Key: X25519` | Ephemeral key exchange, so forward secrecy is present |
| `New, TLSv1.3` | A full handshake using TLS 1.3 (`Reused` would mean resumption) |
| `ALPN protocol: h2` | HTTP/2 was selected |
| `Secure Renegotiation IS NOT supported` | Normal for TLS 1.3, which has no renegotiation |
| `Verify return code: 0 (ok)` | Chain is valid. The hostname was not checked |

### Example 4: inspect the certificate

```bash
# Linux: fetch the leaf certificate and print selected fields
openssl s_client -connect www.example.com:443 -servername www.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -serial -dates -ext subjectAltName
```

```text
subject=CN = www.example.com
issuer=C = US, O = Let's Encrypt, CN = R12
serial=05A3C1F27E9B44D08812C6E03F7A19BD22C4
notBefore=Oct  5 08:00:00 2026 GMT
notAfter=Jan  3 07:59:59 2027 GMT
X509v3 Subject Alternative Name:
    DNS:example.com, DNS:www.example.com
```

`shop.example.com` is not in this SAN list, so using this certificate on that name would fail.

### Example 5: fix "works in Chrome, fails in curl"

A user reports that the site works in Chrome, but a Python script and `curl` fail.

```text
$ curl https://www.example.com
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

Check what the server sends:

```bash
# Linux: count the certificates in the chain
openssl s_client -connect www.example.com:443 -servername www.example.com -showcerts </dev/null \
  | grep -E "^ [0-9] s:|Verify return code"
```

```text
 0 s:CN = www.example.com
Verify return code: 21 (unable to verify the first certificate)
```

Only one certificate arrived. The intermediate is missing.

Why does Chrome work? Browsers can fetch a missing intermediate from the AIA URL in the leaf, or may already have it cached. `curl` and most libraries do not do this.

The fix is to point the server to the full chain file, not the leaf alone:

```text
# nginx on Linux
# Wrong: cert.pem has the leaf only
ssl_certificate /etc/letsencrypt/live/example.com/cert.pem;

# Right: fullchain.pem has the leaf and intermediate
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
```

```bash
# Linux: test the config, reload, then check again
sudo nginx -t && sudo systemctl reload nginx
```

After the reload, the output shows entries `0 s:` and `1 s:` and `Verify return code: 0 (ok)`.

### Example 6: get a certificate and turn on HTTPS

```bash
# Linux: certbot with the nginx plugin, HTTP-01 challenge
sudo certbot --nginx -d example.com -d www.example.com
```

```text
Requesting a certificate for example.com and www.example.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2027-01-03.
```

```bash
# Linux: test that auto-renewal will work
sudo certbot renew --dry-run
```

A good nginx configuration:

```text
# nginx on Linux: /etc/nginx/conf.d/example.com.conf
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

Port 80 stays open for two reasons: the redirect and the HTTP-01 renewal.

### Example 7: see SNI in clear text

```bash
# Linux: tshark prints the destination and name from every ClientHello
sudo tshark -i eth0 -Y "tls.handshake.type == 1" \
  -T fields -e ip.src -e ip.dst -e tls.handshake.extensions_server_name
```

```text
192.168.1.50    203.0.113.10    www.example.com
```

This is what an ISP or firewall can see without any decryption. See [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md).

### Example 8: build mTLS in the lab

```bash
# Linux: create a private CA
openssl req -x509 -newkey rsa:2048 -nodes -keyout ca.key -out ca.crt \
  -days 3650 -subj "/CN=Example Internal CA"

# Create a client key and CSR
openssl req -newkey rsa:2048 -nodes -keyout client.key -out client.csr \
  -subj "/CN=billing-service"

# Sign the client certificate for 30 days
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt -days 30
```

```text
# nginx on Linux: require a client certificate
ssl_client_certificate /etc/nginx/ca.crt;
ssl_verify_client on;
```

```bash
# Linux: without a client certificate, nginx returns 400
curl https://api.example.com/

# Linux: with a client certificate. The server certificate here comes
# from the same private CA, so --cacert trusts it too
curl --cacert ca.crt --cert client.crt --key client.key https://api.example.com/
```

## Going Deeper

### Record layer

After the handshake, data moves in **records**. Each record has a 5-byte header.

```text
+----------------+---------------------+------------------+
| Content type   | Legacy version      | Length           |
| 1 byte         | 2 bytes             | 2 bytes          |
+----------------+---------------------+------------------+
| Payload: encrypted data plus authentication tag         |
| Plaintext limit per record: 2^14 = 16384 bytes          |
+---------------------------------------------------------+
```

| Content type | Value |
|---|---|
| change_cipher_spec | 20 |
| alert | 21 |
| handshake | 22 |
| application_data | 23 |

In TLS 1.3 every encrypted record shows outer type 23. The real type is inside the encryption, so an observer cannot easily tell handshake from data. The version field stays 0x0303.

Each record uses a **nonce**, a number used only once, built from a record counter. This stops an attacker from replaying or reordering records. If a tag check fails, the connection is closed with a fatal alert.

### TLS 1.3 key schedule

TLS 1.3 derives keys in stages with **HKDF**, a standard key derivation function (RFC 5869):

```text
PSK (or zeros)     -->  Early Secret      -->  0-RTT keys
ECDHE shared key   -->  Handshake Secret  -->  handshake keys
(no new input)     -->  Master Secret     -->  application keys
```

Each stage mixes in a hash of the handshake so far. Keys are therefore tied to this exact conversation. Client and server use different keys in each direction.

A long connection can change keys with a **KeyUpdate** message, with no new handshake.

### Alerts

Alerts are short TLS messages that report closing or errors. They are very useful in a packet capture.

| Alert | Code | Typical meaning |
|---|---|---|
| close_notify | 0 | Normal close |
| unexpected_message | 10 | Protocol state error |
| bad_record_mac | 20 | Record failed its integrity check |
| handshake_failure | 40 | No acceptable parameters |
| bad_certificate | 42 | Certificate was corrupt or unacceptable |
| certificate_expired | 45 | Expired certificate |
| unknown_ca | 48 | Peer does not trust the issuer |
| decrypt_error | 51 | Signature or Finished check failed |
| protocol_version | 70 | No common version |
| internal_error | 80 | A bug or a failure on the sender's side |
| unrecognized_name | 112 | Server has no site for that SNI |
| certificate_required | 116 | mTLS needed a client certificate |
| no_application_protocol | 120 | No common ALPN value |

### Downgrade protection

An attacker may try to change the ClientHello so that both sides use an older, weaker version. Two defences work together:

- The Finished messages cover the whole handshake, so a change is detected.
- A TLS 1.3 server that is forced to negotiate TLS 1.2 or lower places a special marker in the last 8 bytes of its random value. A TLS 1.3 client that sees it aborts.

The signature in `CertificateVerify` covers the transcript, so the attacker cannot fix the mismatch.

### Certificate Transparency

Before 2011 a bad CA could quietly issue a certificate for any site. In 2011 the Dutch CA DigiNotar was hacked, and fake `google.com` certificates were used against real users. DigiNotar was removed from trust stores and went bankrupt.

**CT (Certificate Transparency, RFC 6962)** makes issuance public. Before use, a certificate goes into public logs, and the log returns a signed promise called an **SCT (Signed Certificate Timestamp)**. Browsers such as Chrome and Safari require SCTs.

CT does not block a bad certificate. It makes it visible. Domain owners can watch the logs. Sites like crt.sh let you search them, and this is also why internal hostnames on public certificates are not private.

### Post-quantum key exchange

A large future quantum computer could break ECDHE and RSA. An attacker can record traffic today and decrypt it later. This is called **harvest now, decrypt later**.

TLS now deploys a **hybrid** key exchange named **X25519MLKEM768**. It combines classic X25519 with **ML-KEM**, a post-quantum algorithm from NIST FIPS 203. The session stays safe if either one holds.

- Chrome, Firefox and other major clients have shipped it, and OpenSSL 3.5 supports it.
- In `s_client` output on newer OpenSSL it appears as the negotiated group.
- The client key share is about 1200 bytes, so the ClientHello often no longer fits in one packet. This exposed bugs in some old middleboxes.

Signatures in certificates remain classic in public PKI as of 2026. This is the slower part of the migration, because post-quantum signatures are much larger.

### Attacks worth knowing

| Attack | Year | Target | Lesson |
|---|---|---|---|
| BEAST | 2011 | CBC in TLS 1.0 | Predictable IVs are unsafe |
| CRIME | 2012 | TLS compression | Compression can leak secrets |
| Lucky13 | 2013 | CBC padding timing | CBC with a separate MAC is fragile |
| Heartbleed | 2014 | OpenSSL bug, not the protocol | Could leak private keys from server memory |
| POODLE | 2014 | SSL 3.0 | Disable SSL 3.0 |
| FREAK, Logjam | 2015 | Old export-grade crypto | Weak options that stay enabled get abused |
| DROWN | 2016 | SSL 2.0 on a shared key | One old service can hurt a modern one |
| Sweet32 | 2016 | 64-bit block ciphers such as 3DES | Remove 3DES |
| ROBOT | 2017 | RSA key exchange | Remove static RSA |

TLS 1.3 was designed to remove the features behind most of these.

### TLS termination at load balancers

**Termination** means a device ends the TLS connection and sees plain HTTP. This allows HTTP routing, caching and web filtering.

```text
Termination:
Client ==== TLS ====> Load balancer ---- HTTP ----> Backend

Re-encryption:
Client ==== TLS ====> Load balancer ==== TLS =====> Backend

Passthrough (load balancer routes on SNI only):
Client ================ TLS =======================> Backend
```

Termination hides the real client from the backend, so the proxy passes it in `X-Forwarded-For`. If the link to the backend crosses untrusted networks, use re-encryption. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md) and [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

### TLS inspection

A company firewall can act as a man in the middle. It ends the client's TLS, using a certificate it creates on the fly, and opens a new TLS connection to the server. This works only because company devices trust the firewall's root CA.

Side effects:

- The issuer shown in the browser is the company CA.
- Apps that use **certificate pinning**, where the app accepts only a known key, will fail.
- mTLS to outside servers breaks, because the firewall does not hold the client's private key.

### Session ticket keys and forward secrecy

In TLS 1.2, a ticket is encrypted with a server-side ticket key. If that key is stolen, an attacker can decrypt recorded sessions that used those tickets. This quietly weakens forward secrecy. Rotate ticket keys often, and never leave them on disk for long. TLS 1.3 with `psk_dhe_ke` reduces this risk, since it adds fresh ECDHE.

### Other details interviewers may touch

- **RSA vs ECDSA certificates.** ECDSA P-256 keys are much smaller and signing is cheaper on the server. Many sites serve both and pick one per client.
- **Certificate pinning.** The browser feature HPKP was removed because a mistake could lock users out of a site for a long time. Mobile apps still pin, with care.
- **DANE (RFC 6698).** Publishes certificate data in DNS, protected by DNSSEC. It is used in some mail systems, but browsers do not use it.
- **DTLS.** TLS adapted for UDP. DTLS 1.3 is RFC 9147.
- **STARTTLS.** A protocol starts in plain text, then upgrades. An attacker in the path can strip the upgrade command, so strict policies are needed.
- **Key log file.** Set `SSLKEYLOGFILE` in Chrome, Firefox or curl, and the session keys are written to a file. Wireshark can then decrypt your own traffic. This needs access to the client, not the server's private key, and it works with forward secrecy.
- **TLS fingerprints.** The fields of a ClientHello differ per client. Tools such as JA3 and JA4 use this to identify software without decrypting anything.

Network devices also serve HTTPS for management. On a Cisco IOS router, turn off plain HTTP:

```text
! Cisco IOS: allow HTTPS management only
Router(config)# no ip http server
Router(config)# ip http secure-server
```

## Common Mistakes

- **"TLS encrypts all traffic with the server's public key."** No. Public-key work happens only in the handshake. Data uses fast symmetric keys.
- **"The certificate encrypts the session key."** That was static RSA. In ECDHE and TLS 1.3, the certificate key only signs.
- **"The lock icon means the site is safe."** It means the connection is encrypted to the named domain. Phishing sites have valid certificates too.
- **"HTTPS hides which site I visit."** The IP is visible. DNS and SNI usually show the name, unless DoH and ECH are in use. The path, query, headers and body are hidden.
- **"An SSL certificate."** The name stuck, but the protocol is TLS. Certificates do not have a version tied to SSL or TLS, so the phrase "TLS 1.3 certificate" is also wrong.
- **"A longer key or paid certificate is more secure."** A free DV certificate and an expensive EV certificate give the same encryption.
- **Installing only the leaf certificate.** Browsers may hide this mistake. `curl`, Java and mobile apps do not.
- **Using `s_client` and assuming the hostname was checked.** It was not, unless you added `-verify_hostname`.
- **Forgetting to automate renewal.** With short lifetimes, expiry outages are a matter of when, not if. Monitor expiry dates as well.
- **Setting `includeSubDomains` before checking every subdomain.** A subdomain with no HTTPS becomes unreachable for browsers that saw the header.
- **Enabling 0-RTT for everything.** Early data can be replayed. Limit it to safe requests.
- **Saying TLS 1.3 forward secrecy needs no thought.** The PSK-only mode and 0-RTT data do not have the same protection as a full ECDHE handshake.
- **Thinking mTLS replaces authorization.** It proves identity. It does not grant permissions.

## Interview Questions

**Q: What does TLS provide?**
A: Three things. Confidentiality through symmetric encryption, integrity through an authentication tag on each record, and authentication through certificates and signatures. Usually only the server is authenticated. It does not hide IP addresses, ports or traffic patterns, and it usually does not hide the hostname.

**Q: Walk me through the TLS 1.3 handshake.**
A: The client sends a ClientHello with its supported ciphers, SNI, ALPN and a key share. The server replies with a ServerHello and its own key share, and now both can compute keys. The rest is encrypted: EncryptedExtensions, Certificate, CertificateVerify, which signs the handshake with the private key, and Finished. The client verifies these, sends its Finished, and can send its request at once. It takes one round trip.

**Q: What are the main differences between TLS 1.2 and TLS 1.3?**
A: TLS 1.3 takes one round trip for a full handshake instead of two. It removes static RSA, so forward secrecy is the norm. It allows only AEAD ciphers and removes old items like CBC, RC4 and compression. It encrypts the certificate. It removes renegotiation and adds 0-RTT resumption, which has a replay risk.

**Q: What is forward secrecy and how is it achieved?**
A: It means a stolen long-term private key cannot decrypt past recorded traffic. It is achieved with ephemeral Diffie-Hellman, usually ECDHE. Each session creates a fresh key pair that is deleted afterwards, and the session key is never sent. The certificate key only signs, so stealing it later reveals nothing about old sessions.

**Q: How does a browser decide a certificate is valid?**
A: It builds a chain from the leaf through intermediates to a root in its trust store, and checks each signature. It checks the dates, that the hostname is in the SAN, and that the key usage allows server authentication. It also checks revocation data and, in Chrome and Safari, Certificate Transparency proofs. Finally, the handshake signature proves that the server holds the private key.

**Q: Why does TLS use both asymmetric and symmetric cryptography?**
A: Asymmetric cryptography works between strangers but is slow. Symmetric encryption is very fast but needs a shared key. So TLS uses asymmetric methods in the handshake to prove identity and agree on a key, then symmetric encryption for all the data.

**Q: What is SNI, and what problem does ECH solve?**
A: SNI is a field in the ClientHello that names the wanted hostname, so one IP can host many sites with different certificates. It is in clear text, so anyone on the path can see the name. ECH encrypts the real ClientHello, using a key from a DNS HTTPS record, and shows only a shared public name on the outside. It needs encrypted DNS to be fully useful.

**Q: Can someone on the network see the URL I visit over HTTPS?**
A: They cannot see the path, query string, headers or body. They can see the IP address and port, and usually the hostname from SNI or DNS. So they might see `shop.example.com` but not `/orders/1234`.

**Q: Compare CRL, OCSP and OCSP stapling. What is used now?**
A: A CRL is a signed list of revoked serial numbers that the client downloads. OCSP is a live question to the CA, which adds delay and leaks browsing data. With stapling, the server attaches a signed fresh OCSP reply to the handshake. Today the trend is away from live OCSP. Let's Encrypt shut down OCSP in 2025, browsers use compressed lists such as CRLSets and CRLite, and shorter certificate lifetimes reduce the risk.

**Q: What is HSTS, and what attack does it stop?**
A: HSTS is a response header that tells the browser to use only HTTPS for a site for a set time. It stops SSL stripping, where an attacker keeps the victim on plain HTTP. It also makes certificate errors non-bypassable. The first visit is still exposed, and the preload list solves that by building the rule into browsers.

**Q: What is mTLS, and when would you use it?**
A: In mTLS both sides present certificates and prove they hold the private keys. It is used for service-to-service traffic, zero trust networks, partner APIs and devices. The hard part is managing certificates at scale, which is why service meshes automate short-lived ones. It proves identity, so you still need authorization.

**Q: A site works in Chrome but `curl` says "unable to get local issuer certificate". What is wrong?**
A: Most likely the server sends only its leaf certificate and not the intermediate. Chrome can fetch the missing one through the AIA URL or has it cached, but `curl` does not. I confirm with `openssl s_client -showcerts` and count the certificates. The fix is to configure the full chain, such as `fullchain.pem`. If the chain is complete, I check the client's trust store instead.

## Quick Recap

- TLS gives confidentiality, integrity and authentication. HTTPS is HTTP inside TLS, usually on port 443.
- A certificate links a public key to names in the SAN. Trust flows from a root in the trust store, through intermediates, to the leaf. The server proves possession with its private key.
- TLS 1.2 takes 2 RTT. TLS 1.3 takes 1 RTT, encrypts the certificate, removes static RSA and allows only AEAD ciphers.
- ECDHE gives forward secrecy: the certificate key signs, and the session key is never sent.
- SNI shows the hostname in clear text and allows many sites per IP. ECH encrypts it. Resumption saves work, and 0-RTT saves time but can be replayed.
- Revocation has moved from live OCSP toward CRLs, browser-pushed lists and short lifetimes. HSTS forces HTTPS, and mTLS authenticates both sides.
- Let's Encrypt and ACME automate DV certificates. Public certificate lifetimes are falling to 47 days by 2029, so automate renewal and use `openssl s_client` to debug.
