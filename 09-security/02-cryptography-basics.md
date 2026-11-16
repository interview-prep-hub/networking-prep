# Cryptography Basics

**Level:** Intermediate | **Prerequisites:** [Network Security Basics](01-security-basics.md), [Binary and Hexadecimal for Networking](../01-foundations/07-binary-and-hexadecimal.md)

## In One Line

Cryptography uses math and secret keys to keep data private, prove it was not changed, and prove who sent it, even when the network in the middle cannot be trusted.

## Why It Matters

Your packets cross devices you do not own: a cafe Wi-Fi access point, your ISP's routers, and many networks after that. Any of them could read, copy or change your data. A firewall cannot help once the data has left your network. Cryptography is what protects the data on that path.

Cryptography answers four questions:

| Goal | Question it answers | Main tool |
|---|---|---|
| Confidentiality | Can only the right people read this? | Encryption (AES, ChaCha20) |
| Integrity | Was this changed on the way? | Hash, HMAC, AEAD tag |
| Authentication | Who am I really talking to? | Digital signature, HMAC |
| Non-repudiation | Can the sender later deny sending it? | Digital signature |

The same few building blocks appear in [TLS](../07-application-protocols/07-tls-and-https.md), [IPsec](08-ipsec.md), [SSH](../07-application-protocols/09-remote-access-protocols.md), [Wi-Fi security](../08-wireless-and-mobile/02-wifi-security.md) and [VPNs](07-vpn.md). Learn the blocks once and all of these become easier.

Interviewers ask about this because the questions quickly show who understands and who only memorised names. Typical questions are "Why does TLS use both RSA or ECC and AES?", "What is the difference between encoding, encryption and hashing?" and "What is forward secrecy?"

## Simple Explanation

Alice and Bob live in different cities. Every letter between them passes through the hands of strangers. They need these tools:

| Real-life idea | Crypto name | What it does |
|---|---|---|
| A strong box with one key, and both people hold a copy | Symmetric encryption | Fast privacy, but both sides must already share the key |
| An open padlock that anyone can click shut, but only Bob can open | Asymmetric encryption | A stranger can send Bob a secret |
| A fingerprint of a document | Hash | A short value that changes if the document changes |
| A stamp that only Alice and Bob own | HMAC | Proves the message came from someone holding the shared secret |
| A seal only Alice can make, but anyone can check | Digital signature | Proves Alice made it |
| Two people mixing paint in public to get a secret colour | Diffie-Hellman | Creates a shared key with no secret ever sent |

**Where the analogy stops:** a padlock can be cut, and a box can be broken with enough force. Good cryptography has no known shortcut of that kind. Also, these tools are math on numbers, so one public "padlock" can be copied freely and used by millions of people.

Some words you will see everywhere:

- **Plaintext:** the original readable data.
- **Ciphertext:** the scrambled data after encryption.
- **Cipher:** the algorithm that encrypts and decrypts.
- **Key:** a secret number that controls the cipher.
- **Bits of security:** a measure of strength. "128-bit security" means an attacker needs about 2^128 steps.

The algorithm is public. Only the key is secret. This rule is called **Kerckhoffs's principle**. AES has been studied in public for over 20 years, which is why we trust it. A secret home-made cipher has had no such testing.

## How It Works

### Encoding, Encryption and Hashing

These three are often confused. An interviewer may well start here.

| | Encoding | Encryption | Hashing |
|---|---|---|---|
| Purpose | Change the format | Hide the content | Make a fingerprint |
| Key needed? | No | Yes | No (HMAC adds one) |
| Reversible? | Yes, by anyone | Yes, with the key | No, one way |
| Security? | None | Confidentiality | Integrity check |
| Examples | Base64, hex, URL encoding | AES, RSA, ChaCha20 | SHA-256, SHA-3 |

**Encoding is not encryption.** Base64 turns bytes into safe text characters so they can pass through systems built for text. There is no key, so anyone can reverse it. In [HTTP](../07-application-protocols/05-http-basics.md), Basic authentication sends this header:

```text
Authorization: Basic YWRtaW46c2VjcmV0
```

That looks secret, but it decodes to `admin:secret` in one command. Only TLS around the HTTP request protects it.

### Symmetric Encryption

Symmetric means the same key encrypts and decrypts.

```text
              same secret key
             |               |
             v               v
plaintext -> [ encrypt ] -> ciphertext -> [ decrypt ] -> plaintext
```

**AES** (Advanced Encryption Standard, published by the US government as FIPS 197) is the most used cipher in the world. It is a **block cipher**: it works on fixed chunks of data, always 128 bits (16 bytes). Keys can be 128, 192 or 256 bits. AES runs 10, 12 or 14 rounds of mixing, depending on key size.

**How strong is that?** A 128-bit key has 2^128 possible values, about 3.4 x 10^38. If a machine tested 10^18 keys every second, trying them all would take about 10 trillion years. The universe is about 13.8 billion years old. Nobody breaks AES by guessing keys.

**Why it is fast:** many CPUs have built-in AES instructions (AES-NI on x86). One core can often encrypt gigabytes per second.

**The weakness is key sharing.** Both sides need the same key before they start, and you cannot send it over the network in the clear. The number of keys also grows quickly. For 1,000 people to each share a private key with every other person, you need 1000 x 999 / 2 = 499,500 keys.

**ChaCha20** is the main alternative. It is a **stream cipher**, which produces a stream of random-looking bytes to mix with the data, rather than working on blocks. It is fast in software, so it suits devices without AES hardware.

**Old ciphers to avoid:** DES (56-bit key, too short), 3DES (64-bit block, disallowed for encryption after 2023 by NIST, the US standards institute) and RC4 (prohibited in TLS by RFC 7465).

### Block Modes

AES alone encrypts exactly 16 bytes. Real data is longer. A **mode of operation** is the rule for using AES on many blocks.

**Analogy:** AES is a machine that seals one small box. A mode is the packing plan for moving a whole house in many boxes. **Where it stops:** a bad packing plan leaks patterns even though each box is strong.

Two terms first:

- **XOR** is a bit operation: the result is 1 when two bits differ. It undoes itself: `(A XOR K) XOR K = A`.
- **Nonce** means "number used once". **IV** (initialization vector) is a similar start value. Neither is secret, and both are sent or derived openly. A nonce must never repeat under the same key.

| Mode | How it works | Status |
|---|---|---|
| ECB | Each block is encrypted alone | Never use for data |
| CBC | Each block is XORed with the previous ciphertext block | Legacy, needs a separate integrity check |
| CTR | AES encrypts a counter, then the output is XORed with data | Good base, but no integrity |
| GCM | CTR for privacy, plus an authentication tag | Current standard |

**ECB (Electronic Codebook)** leaks patterns, because identical plaintext blocks give identical ciphertext:

```text
Plaintext blocks :  AAAA  BBBB  AAAA  AAAA
ECB ciphertext   :  x7Q2  m0Zk  x7Q2  x7Q2    <-- pattern leaks
GCM ciphertext   :  r9Tc  1vHs  Ke4W  pL0d    <-- no pattern
(Simplified: real blocks are 16 bytes.)
```

A famous demo encrypts a picture of the Linux penguin with ECB. You can still see the penguin.

**GCM (Galois/Counter Mode)** encrypts and also protects integrity:

```text
  counter 1      counter 2      counter 3
      |              |              |
  [AES, key]     [AES, key]     [AES, key]
      |              |              |
  keystream      keystream      keystream
      |              |              |
 P1->XOR        P2->XOR        P3->XOR
      |              |              |
     C1             C2             C3
      |              |              |
      +--------------+--------------+
                     |
 associated data -> GHASH -> 16-byte authentication tag
```

`P` means a plaintext block and `C` a ciphertext block. **Keystream** is the random-looking output of AES. GHASH is a fast keyed math function that combines all the ciphertext into the tag. The receiver recomputes the tag. If one bit changed, the tags differ and the data is rejected.

GCM is an **AEAD** cipher: Authenticated Encryption with Associated Data. "Associated data" is data that stays readable but is still protected from change, such as a record header. ChaCha20-Poly1305 (RFC 8439) is the other common AEAD.

**Why integrity matters:** in CTR mode without a tag, flipping a ciphertext bit flips the same plaintext bit. The ASCII code for `1` is 0x31 and for `9` is 0x39, a one-bit difference. An attacker who knows where the digit sits can turn `100` into `900` without knowing the key.

### Asymmetric Encryption

Asymmetric encryption uses a pair of linked keys:

- The **public key** can be shown to everyone.
- The **private key** is never shared.

What one key does, only the other can undo. This allows two uses:

```text
Privacy:   anyone encrypts with Bob's PUBLIC key
           only Bob decrypts with Bob's PRIVATE key

Signing:   only Alice signs with Alice's PRIVATE key
           anyone verifies with Alice's PUBLIC key
```

**RSA** is built on one fact: multiplying two huge prime numbers is easy, but splitting the result back into those primes is very hard. The product is public. The primes stay secret.

**ECC** (Elliptic Curve Cryptography) uses points on a curve. Moving forward along the curve many times is easy. Working out how many moves were made is very hard. The best known attacks on ECC are much weaker than those on RSA, so ECC needs much smaller keys.

| Strength | RSA key | ECC key |
|---|---|---|
| 112 bits | 2048 bits | 224 bits |
| 128 bits | 3072 bits | 256 bits (P-256, Curve25519) |
| 192 bits | 7680 bits | 384 bits (P-384) |

Common modern ECC names are **X25519** (key agreement), **Ed25519** (signatures) and **P-256**, also called secp256r1 (both uses, through ECDH and ECDSA).

**Asymmetric is slow and small.** One CPU core does only a few thousand RSA-2048 private key operations per second at best. RSA also encrypts only small inputs: with a 2048-bit key and OAEP-SHA-256 padding (extra structured data added before encryption), the maximum is 190 bytes.

So real systems use **hybrid encryption**: asymmetric math sets up a key, then symmetric encryption carries the data.

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared key | Public and private pair |
| Speed | Very fast | Much slower |
| Key sharing | Hard | Easy, the public key is public |
| Typical job | Bulk data | Key agreement, signatures |
| Examples | AES, ChaCha20 | RSA, ECC |

### Hashing

A **hash function** turns any input into a fixed-size output called a **digest**. SHA-256 always gives 256 bits, which is 32 bytes or 64 hex characters, for one byte or a 4 GB file.

A good hash has these properties:

1. **Deterministic:** the same input always gives the same digest.
2. **One way:** you cannot work back from the digest to the input.
3. **Collision resistant:** nobody can find two inputs with the same digest.
4. **Avalanche effect:** a tiny input change changes about half the output bits.

**Analogy:** a fingerprint identifies a person, but you cannot rebuild a person from it. **Where it stops:** collisions must exist, because endless possible inputs map to a fixed number of outputs. With a good hash they exist, but nobody can find one.

**The SHA-2 family** (FIPS 180-4) includes SHA-224, SHA-256, SHA-384 and SHA-512. The number is the output size in bits.

| Hash | Output | Status |
|---|---|---|
| MD5 | 128 bits | Broken. Collisions shown in 2004 |
| SHA-1 | 160 bits | Broken. Public collision in 2017 (SHAttered) |
| SHA-256 | 256 bits | Standard choice |
| SHA-384, SHA-512 | 384, 512 bits | Safe, used for higher strength |
| SHA-3 | 224 to 512 bits | Safe, different internal design (FIPS 202) |

Hashes are used for file downloads, digital signatures, TLS handshake records, and Git object names.

**A plain hash does not prove integrity against an attacker.** If someone changes a file, they can also recompute its hash. A hash on the same channel only catches accidents. To stop attackers you need a key, which means HMAC or a signature.

### Salts and Password Storage

Servers must never store passwords in plain form. They store a hash and compare hashes at login. But plain hashing fails for two reasons:

1. People use the same weak passwords, so equal passwords give equal hashes.
2. Attackers build **rainbow tables**, which are huge precomputed lists of password and hash pairs.

A **salt** is a random value made for each user, stored next to the hash. The server hashes salt + password.

```text
Without salt:  both users store the same hash
  alice  SHA-256("Summer2026")          = ed3c9886c697...
  bob    SHA-256("Summer2026")          = ed3c9886c697...

With a unique salt:  the hashes are different
  alice  SHA-256("a1f4c9e2Summer2026")  = f632a9a3685a...
  bob    SHA-256("7b03de58Summer2026")  = 401bf1c0cb38...
```

**Analogy:** two cooks follow the same recipe, but each adds a different spice, so the dishes taste different. **Where it stops:** the salt is not secret. It sits beside the hash in the database. Its job is to make every hash unique, so one precomputed table cannot attack all users at once.

Salt alone is not enough, because SHA-256 is too fast. A GPU (graphics card) can test billions of SHA-256 guesses per second. Passwords need a **slow** function with a work setting: **Argon2id** (RFC 9106), **scrypt** (RFC 7914), **bcrypt** or **PBKDF2**. Use a salt of at least 16 random bytes.

A **pepper** is an extra secret kept outside the database, for example in a hardware module or a secrets manager. If only the database leaks, the attacker still lacks the pepper.

### HMAC

**HMAC** (Hash-based Message Authentication Code, RFC 2104) is a hash that also needs a secret key. It proves two things: the message was not changed, and it was made by someone who holds the key.

```text
HMAC(key, message) = H( (key XOR opad) + H( (key XOR ipad) + message ) )
```

`H` is a hash such as SHA-256, `+` joins bytes together, and `opad` and `ipad` are fixed constants. In simple words: hash twice, mixing in the key each time.

```text
Sender                                  Receiver
message + key -> HMAC tag               message + key -> HMAC tag
                                        Tags equal?  Yes: accept
                                                     No: reject
          message + tag
      ---------------------------->
```

**Analogy:** Alice and Bob own the same private stamp. A stranger can read the letter, but cannot make a matching stamp. **Where it stops:** since both own the stamp, a judge cannot tell which of them used it. HMAC gives no non-repudiation.

HMAC appears in IPsec integrity, SNMPv3 authentication, [OSPF](../05-routing/06-ospf.md) authentication (RFC 5709), JSON Web Tokens signed with HS256, and API request signing. Inside TLS 1.3 it powers key derivation and the Finished messages. [BGP](../05-routing/09-bgp.md) sessions traditionally use the older keyed TCP MD5 option (RFC 2385), and TCP-AO (RFC 5925) is its modern replacement.

### Digital Signatures

A **digital signature** proves who created data, and that it did not change. It uses the signer's private key, so anyone with the public key can check it, and nobody else can create it.

```text
SIGNING (Alice)
  message -> SHA-256 -> digest -> sign with Alice's PRIVATE key
                                            |
                                            v
                                        signature

VERIFYING (anyone)
  received message -> SHA-256 -> digest --+
                                          +--> match? valid
  signature + Alice's PUBLIC key ---------+
```

We sign the digest, not the whole message, because the digest is small and fixed in size.

**Analogy:** a king's wax seal. Anyone who knows the seal can check it, but only the king owns the ring. **Where it stops:** a real seal can be lifted and copied onto another letter. A digital signature depends on every bit of the message, so it is useless on any other message.

**Fuller truth:** you may read that "a signature is the hash encrypted with the private key". That is a loose picture of basic RSA only. ECDSA and Ed25519 do no encryption at all. They are separate signing math. Say "sign" and "verify", not "encrypt".

| | Hash | HMAC | Digital signature |
|---|---|---|---|
| Key | None | One shared secret | Private to sign, public to verify |
| Detects accidents | Yes | Yes | Yes |
| Stops an attacker | No | Yes | Yes |
| Proves who to a third party | No | No | Yes |
| Speed | Fastest | Fast | Slowest |

**How do you trust a public key?** With a **certificate**. It is a document that says "this public key belongs to example.com", signed by a Certificate Authority (CA) that your device already trusts. The details belong to the [TLS chapter](../07-application-protocols/07-tls-and-https.md).

### Diffie-Hellman Key Exchange

**The problem:** Alice and Bob have never met, and everything they send can be read. How can they get a shared key?

**Diffie-Hellman (DH)** lets them build a shared secret without ever sending it. The classic picture uses paint, based on one fact: mixing colours is easy, but un-mixing them is practically impossible.

```text
     ALICE                 PUBLIC                  BOB
  secret: red          common: yellow         secret: blue

  yellow + red                               yellow + blue
    = orange                                    = green

        orange ------------------------------->
        <------------------------------- green

       (An eavesdropper sees yellow, orange and green)

  green + red                                orange + blue
    = brown                                     = brown

              Both now hold the same brown.
```

Step by step:

1. Alice and Bob agree in public on a common colour, yellow.
2. Each picks a secret colour and never shares it.
3. Each mixes the secret colour with yellow.
4. They swap the mixtures in public.
5. Each adds their own secret to the mixture they received.
6. Both get the same brown, since it holds yellow, red and blue.

An eavesdropper holds yellow, orange and green. To make brown, they would need to pull red out of orange, or blue out of green.

**Where the analogy stops:** paint is really just a picture of the math. Real DH uses numbers. Each secret is an exponent, and the "mixing" is `g^secret mod p`. "Un-mixing" is the **discrete logarithm problem**, which is very hard at proper sizes. ECDH does the same with elliptic curve points, and is the usual form today. The Example section shows it with small numbers.

**DH alone does not prove identity.** An attacker in the middle can run one DH exchange with Alice and another with Bob, then read everything. This is a **man-in-the-middle (MITM) attack**, covered in [Common Network Attacks](03-common-network-attacks.md). Nothing in the paint tells Alice who she is mixing with. This is why TLS adds a digital signature over the exchange.

### Forward Secrecy

**Forward secrecy** (or perfect forward secrecy, PFS) means that stealing a server's long-term private key later does not expose past sessions.

It works by using **ephemeral** keys: fresh DH keys for each session, deleted after use. The names **DHE** and **ECDHE** both end in E for ephemeral.

```text
2025  Attacker records encrypted traffic to a server.
2027  Attacker steals the server's long-term private key.

Old RSA key exchange:
  The session key was sent encrypted with the server's public key.
  The stolen private key decrypts it. All recorded traffic is exposed.

ECDHE:
  The session key was never sent, and its parts were deleted.
  The stolen key opens nothing that was recorded.
```

**Analogy:** RSA key exchange is like keeping every diary in a single safe, so one stolen key opens all of them. ECDHE is like writing each day's notes on paper that burns after reading. **Where it stops:** if an attacker controls a computer during a live session, they can read that session in memory. Forward secrecy protects the past, not an active compromise. A stolen signing key still lets an attacker impersonate the server until the certificate is revoked or expires.

TLS 1.3 removed static RSA key exchange, so every certificate-based full handshake is forward secret.

**Fuller truth:** session resumption can weaken this. A resumed session that skips fresh (EC)DHE depends on a saved key, so servers must rotate their ticket keys often. TLS 1.3 0-RTT early data is also not forward secret.

### How It All Combines in TLS

No single tool does the whole job. TLS 1.3 uses all of them in one connection. Here is a simplified view. The [TLS chapter](../07-application-protocols/07-tls-and-https.md) has the full protocol.

```text
Client                                              Server
  |                                                   |
  | ClientHello: cipher suites, random,               |
  | key_share (X25519 public value)                   |
  |-------------------------------------------------->|
  |                                                   |
  | ServerHello: chosen suite, random, key_share      |
  |<--------------------------------------------------|
  |                                                   |
  |   Both run ECDHE, then derive keys with HKDF.     |
  |   All messages below are encrypted.               |
  |                                                   |
  | Certificate, CertificateVerify, Finished          |
  |<--------------------------------------------------|
  | Finished                                          |
  |-------------------------------------------------->|
  |                                                   |
  | Application data, protected with AES-GCM          |
  |<=================================================>|
```

| Step | Tool | Job |
|---|---|---|
| key_share messages | ECDHE (X25519) | Create a shared secret, with forward secrecy |
| Key derivation | HKDF, built on HMAC-SHA-256 | Turn one secret into many keys |
| Certificate | CA signature | Bind the name example.com to a public key |
| CertificateVerify | Server's signature over a hash of the handshake | Prove the server owns the private key, and link it to this ECDHE exchange |
| Finished | HMAC over the handshake | Prove nothing in the handshake was changed |
| Application data | AES-GCM | Fast privacy and integrity for all data |

**HKDF** (HMAC-based Key Derivation Function, RFC 5869) stretches one shared secret into several independent keys. TLS uses different keys for each direction.

**Reading cipher suite names.** A cipher suite is a named set of algorithms.

```text
TLS 1.2:  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
              |     |        |           |
              |     |        |           +-- hash for key derivation
              |     |        +-------------- AES-256 in GCM mode
              |     +----------------------- certificate signature type
              +----------------------------- key exchange

TLS 1.3:  TLS_AES_256_GCM_SHA384
              |           |
              |           +----------------- hash for HKDF
              +----------------------------- AEAD cipher
```

TLS 1.3 names are shorter because key exchange and signature type are negotiated separately. Its three common suites are `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384` and `TLS_CHACHA20_POLY1305_SHA256`.

**A protected TLS 1.3 record** looks like this on the wire:

```text
+------+---------+--------+-------------------------+------+
| Type | Version | Length | Encrypted data          | Tag  |
| 1 B  | 2 B     | 2 B    | data + 1 B real type    | 16 B |
| 0x17 | 0x0303  |        |                         |      |
+------+---------+--------+-------------------------+------+
|<-- authenticated only ->|<--- encrypted and protected -->|
```

The 5-byte header is the "associated data" of AEAD. The version field says 0x0303, which means TLS 1.2, on purpose, for compatibility with old middle devices. Overhead with AES-GCM is at least 22 bytes per record. The nonce is not sent. Both sides compute it from a sequence number and a shared IV.

## Example

### Example 1: Run Each Tool Yourself

Linux commands below use OpenSSL 3.x. Windows 11 does not include OpenSSL by default, but Git for Windows ships it, so Git Bash can run these same commands.

**Encoding: no key needed.**

```bash
# Linux
printf 'admin:secret' | base64
# YWRtaW46c2VjcmV0
printf 'YWRtaW46c2VjcmV0' | base64 -d
# admin:secret
```

```powershell
# Windows PowerShell
$b = [Text.Encoding]::UTF8.GetBytes('admin:secret')
[Convert]::ToBase64String($b)
# YWRtaW46c2VjcmV0
$s = [Convert]::FromBase64String('YWRtaW46c2VjcmV0')
[Text.Encoding]::UTF8.GetString($s)
# admin:secret
```

**Hashing: one capital letter changes everything.**

```bash
# Linux
printf 'hello' | sha256sum
# 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824  -
printf 'Hello' | sha256sum
# 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969  -
```

```powershell
# Windows PowerShell (hash a file with no line ending)
[IO.File]::WriteAllText('C:\lab\hello.txt', 'hello')
(Get-FileHash C:\lab\hello.txt -Algorithm SHA256).Hash
# 2CF24DBA5FB0A30E26E83B2AC5B9E29E1B161E5C1FA7425E73043362938B9824
certutil -hashfile C:\lab\hello.txt SHA256
```

Note: use `printf`, not `echo`. Plain `echo` adds a newline, and that changes the hash.

**HMAC: key and message both matter.**

```bash
# Linux
printf 'transfer 100 to bob' \
  | openssl dgst -sha256 -hmac 'k3y-shared-by-both' \
  | awk '{print $2}'
# cb7e68b2b3544543865eb71a277918955cb08b274f65735cff8633e2a6142866

printf 'transfer 900 to bob' \
  | openssl dgst -sha256 -hmac 'k3y-shared-by-both' \
  | awk '{print $2}'
# a0b48c948c1bf616aa58aa16f1f4dc30aaa3b658d52db0360ef5fa1482cad260
```

An attacker who changes 100 to 900 cannot compute the second tag without the key.

**Symmetric encryption.** The OpenSSL `enc` tool refuses GCM:

```bash
# Linux
printf 'attack at dawn' > msg.txt
openssl enc -aes-256-gcm -pbkdf2 -pass pass:demo -in msg.txt -out x.bin
# enc: AEAD ciphers not supported
```

Programs use GCM through a crypto library. For a command-line demo we use CBC:

```bash
# Linux
openssl enc -aes-256-cbc -pbkdf2 -pass pass:demo -a \
  -in msg.txt -out msg.enc
cat msg.enc
# U2FsdGVkX19sL3uqge4826PcSiyuqbAPtZwsCJwQOKg=
openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:demo -a -in msg.enc
# attack at dawn
```

Your output will differ, because the salt is random each run. Notice the two layers. AES did the encryption. Base64 (`-a`) only made the result printable. Decoding the Base64 reveals the text `Salted__` and then the salt, which is stored openly.

**Signatures: the verifier needs only the public key.**

```bash
# Linux
openssl ecparam -name prime256v1 -genkey -noout -out sign.key
openssl ec -in sign.key -pubout -out sign.pub
printf 'Router config v7' > doc.txt

openssl dgst -sha256 -sign sign.key -out doc.sig doc.txt
openssl dgst -sha256 -verify sign.pub -signature doc.sig doc.txt
# Verified OK

printf 'Router config v8' > doc.txt
openssl dgst -sha256 -verify sign.pub -signature doc.sig doc.txt
# Verification failure
```

**Key agreement: both sides reach the same secret.**

```bash
# Linux
openssl genpkey -algorithm X25519 -out alice.key
openssl genpkey -algorithm X25519 -out bob.key
openssl pkey -in alice.key -pubout -out alice.pub
openssl pkey -in bob.key -pubout -out bob.pub

openssl pkeyutl -derive -inkey alice.key -peerkey bob.pub | xxd -p -c 32
# f1b70e3002b8812bd937ba058e42159d10442abd0e4a06f0b6cd3a1a8e989023
openssl pkeyutl -derive -inkey bob.key -peerkey alice.pub | xxd -p -c 32
# f1b70e3002b8812bd937ba058e42159d10442abd0e4a06f0b6cd3a1a8e989023
```

Alice used her private key and Bob's public key. Bob did the opposite. The result is the same 32 bytes. Your values will differ, since the keys are random.

**Look at a real TLS connection.**

```bash
# Linux, OpenSSL 3.5 or newer
openssl s_client -connect example.com:443 -brief </dev/null
```

```text
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN=example.com
Hash used: SHA256
Signature type: ecdsa_secp256r1_sha256
Verification: OK
Negotiated TLS1.3 group: X25519MLKEM768
```

The exact values depend on the server and your OpenSSL version. Here is how to read this sample:

| Output | Meaning |
|---|---|
| TLS_AES_256_GCM_SHA384 | Data uses AES-256-GCM, and key derivation uses SHA-384 |
| ecdsa_secp256r1_sha256 | The server signed with an ECC P-256 key |
| Verification: OK | The certificate chain was trusted |
| X25519MLKEM768 | Key exchange used classic X25519 plus post-quantum ML-KEM |

### Example 2: Diffie-Hellman With Small Numbers

Public values: prime `p = 23`, generator `g = 5`. A **generator** is simply the agreed base number.

```text
Alice picks secret a = 6       Bob picks secret b = 15

A = 5^6 mod 23 = 8             B = 5^15 mod 23 = 19

         A = 8 sent in public ------>
         <------ B = 19 sent in public

s = 19^6 mod 23 = 2            s = 8^15 mod 23 = 2

Shared secret = 2
```

`mod` means "keep the remainder after division". This works since both sides compute `5^(6 x 15) mod 23`.

An eavesdropper knows 23, 5, 8 and 19. To find the secret they must solve `5^a mod 23 = 8`. With such small numbers this is easy. With a 2048-bit prime, or a 256-bit curve, it is out of reach. These tiny numbers are for learning only.

### Example 3: RSA With Small Numbers

```text
1. Pick primes             p = 61, q = 53
2. Multiply                n = 61 x 53 = 3233
3. Compute                 (p-1)(q-1) = 60 x 52 = 3120
4. Pick public exponent    e = 17
5. Find private exponent   d = 2753  (17 x 2753 mod 3120 = 1)

Public key:   (n = 3233, e = 17)
Private key:  (n = 3233, d = 2753)

Encrypt m = 65:   c = 65^17 mod 3233 = 2790
Decrypt c:        m = 2790^2753 mod 3233 = 65
```

An attacker who could split 3233 into 61 x 53 could find `d`. Real RSA uses an `n` of at least 2048 bits, and a typical `e` of 65537. Real RSA also adds random padding, because this bare form is unsafe.

### Example 4: Cisco Passwords Show the Difference

On Cisco IOS, the number in a config tells you how a password is stored.

| Type | Meaning | Status |
|---|---|---|
| 0 | Plaintext | Never leave it this way |
| 7 | Reversible scrambling with a fixed, public key table | Not real protection |
| 5 | Salted MD5 hash | Legacy |
| 8 | PBKDF2 with SHA-256 | Good |
| 9 | scrypt | Good |

Type 7 is the classic trap:

```text
! Cisco IOS
R1(config)# service password-encryption
R1(config)# username ops password cisco
R1(config)# end
R1# show running-config | include username
username ops password 7 094F471A1A0A
```

`094F471A1A0A` looks strong, but anyone can reverse it to `cisco`, since the algorithm and its table are public. Despite the command name, it only stops someone reading over your shoulder. Treat it as encoding, not encryption.

A safer form uses a slow one-way hash (type 9):

```text
! Cisco IOS, on versions that support it
R1(config)# enable algorithm-type scrypt secret Lab-Pass-2026
R1(config)# username netadmin algorithm-type scrypt secret Lab-Pass-2026
```

## Going Deeper

### Three Kinds of Hash Attack

- **Preimage:** given a digest, find any input for it.
- **Second preimage:** given one input, find a different input with the same digest.
- **Collision:** find any two inputs with the same digest.

Collisions are easier because of the **birthday paradox**. In a room of only 23 people, there is about a 50% chance that two share a birthday. An n-bit hash has about 2^(n/2) collision strength, so SHA-256 gives about 128 bits against collisions and about 256 bits against preimages.

### Why HMAC Instead of SHA-256(key + message)

MD5, SHA-1, SHA-256 and SHA-512 use the Merkle-Damgard design, where the digest is the internal state of the function. An attacker who sees `SHA-256(key + message)` can continue hashing from that state and produce a valid digest for `message + padding + extra`, without knowing the key. This is a **length extension attack**. HMAC's double hashing blocks it.

### What Happens When a GCM Nonce Repeats

GCM with a repeated nonce and key is a disaster:

1. The keystream repeats, so `C1 XOR C2 = P1 XOR P2`, which leaks plaintext.
2. An attacker can recover the internal authentication key and forge messages.

TLS 1.3 avoids this by building each nonce from a 64-bit record sequence number XORed with a fixed IV. Limits exist too. One GCM message cannot exceed about 64 GiB, and RFC 8446 advises changing keys after about 2^24.5 full-size records with AES-GCM. TLS 1.3 does this with the KeyUpdate message. AES-GCM-SIV (RFC 8452) is a variant built to survive nonce mistakes better.

### Order Matters: Encrypt, Then Authenticate

TLS 1.2 CBC suites used MAC-then-encrypt. The receiver had to decrypt and check padding before it could check integrity. Small differences in error or timing leaked data to attackers. This is called a **padding oracle**. Lucky Thirteen and POODLE are well-known attacks in this family. AEAD modes avoid this, as encryption and integrity are one operation. This is one reason TLS 1.3 allows only AEAD ciphers.

### RSA Padding

Textbook RSA is unsafe, because the same message always gives the same ciphertext. Real RSA adds structured randomness: **OAEP** for encryption and **PSS** for signatures. TLS 1.3 requires PSS when an RSA key signs CertificateVerify. The older PKCS#1 v1.5 encryption padding allowed the **Bleichenbacher** attack, which came back in forms such as ROBOT (2017).

### Randomness and Side Channels

Keys, nonces and salts need a **CSPRNG** (cryptographically secure pseudo-random number generator). Normal `rand()` functions are predictable. Use `/dev/urandom` or `getrandom()` on Linux, and the system crypto API on Windows. The 2008 Debian OpenSSL bug left only a small number of possible keys, so even strong ciphers failed.

Code can also leak secrets through time or power use. Comparing an HMAC tag byte by byte and stopping at the first difference leaks how many bytes matched. Compare tags in constant time, for example with `hmac.compare_digest` in Python.

### Downgrade Attacks

An attacker in the middle may try to push both sides to weaker settings. **Logjam** (2015) forced some servers to use 512-bit export-grade DH. In TLS 1.3, the signature in CertificateVerify and the Finished messages cover the full handshake, so changed offers are detected.

### DH Groups in IPsec

[IPsec](08-ipsec.md) configs name key exchange by group number. Interviewers like these:

| IKE group | Type |
|---|---|
| 2 | 1024-bit MODP, too weak today |
| 14 | 2048-bit MODP, minimum for classic DH |
| 19 | 256-bit ECC (P-256) |
| 20 | 384-bit ECC (P-384) |
| 31 | Curve25519 |

MODP means classic DH with modular math.

### Post-Quantum Cryptography

A large, stable quantum computer running **Shor's algorithm** would break RSA, DH and ECC. No such machine exists today. The risk is already real for long-lived secrets, through **harvest now, decrypt later**: record traffic today, and decrypt it in the future.

| Standard | Name | Used for |
|---|---|---|
| FIPS 203 (2024) | ML-KEM | Key exchange |
| FIPS 204 (2024) | ML-DSA | Signatures |
| FIPS 205 (2024) | SLH-DSA | Signatures, built only on hashes |

**ML-KEM** is a key encapsulation method: one side uses a public key to wrap a fresh shared secret for the other side. Deployed TLS uses hybrid key exchange, such as X25519MLKEM768, so that the connection stays safe unless both X25519 and ML-KEM fail. Major browsers and large [content delivery networks](../10-enterprise-and-datacenter/06-cdn.md) already use it. The cost is size: an ML-KEM-768 public key is 1184 bytes, while an X25519 key is 32 bytes. NIST's transition plan aims to retire RSA and ECC from its standards by about 2035.

Symmetric ciphers are affected far less. **Grover's algorithm** could in theory reduce a key's strength toward half, which is why AES-256 is often chosen for long-term data. Hash functions like SHA-256 are also seen as safe.

### What to Use Today

| Area | Use | Avoid |
|---|---|---|
| Encryption | AES-GCM, ChaCha20-Poly1305 | DES, 3DES, RC4, ECB mode |
| Hash | SHA-256, SHA-384, SHA-512, SHA-3 | MD5, SHA-1 |
| Key exchange | X25519, P-256 ECDHE, X25519MLKEM768 | Static RSA, DH under 2048 bits |
| Signature | Ed25519, ECDSA P-256, RSA-2048+ with PSS | RSA-1024, SHA-1 signatures |
| Passwords | Argon2id, scrypt, bcrypt, PBKDF2 | Plain or fast hashes |

## Common Mistakes

- **Saying "Base64 encrypted".** Base64 has no key. Anyone can decode it.
- **Saying "hashes are decrypted".** A hash cannot be decrypted. Attackers crack hashes by guessing inputs and hashing each guess.
- **Thinking a salt must be secret.** It is stored in the open. Its job is to be unique for each user.
- **Reusing one salt for all users.** This lets an attacker test each guess against all users at once.
- **Using SHA-256 alone for passwords.** It is too fast. Use Argon2id, scrypt, bcrypt or PBKDF2.
- **Believing a hash stops tampering.** An attacker can recompute a plain hash. Use HMAC or a signature.
- **Saying HTTPS encrypts data with RSA.** The data is protected by a symmetric cipher such as AES-GCM. RSA or ECC is used only in the handshake, and in TLS 1.3, for signatures only.
- **Saying the server's public key encrypts the session key in TLS 1.3.** That was the old RSA key exchange in TLS 1.2 and earlier. TLS 1.3 uses ephemeral (EC)DHE.
- **Thinking DH proves identity.** Plain DH is open to a man-in-the-middle attack unless it is authenticated.
- **Thinking "signing" means encrypting with a private key.** That is only a loose picture of basic RSA, and it is wrong for ECDSA and Ed25519.
- **Comparing RSA and ECC by key size alone.** A 256-bit ECC key is about as strong as a 3072-bit RSA key.
- **Reusing a nonce with AES-GCM.** It breaks both privacy and integrity.
- **Thinking `service password-encryption` is strong.** Cisco type 7 is reversible in seconds.
- **Inventing your own crypto.** Use a well-reviewed library and standard protocols.
- **Assuming a capture can always be decrypted with the server's key.** This worked for RSA key exchange only. With ECDHE, [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md) needs the session keys, for example from a browser's `SSLKEYLOGFILE` file.

## Interview Questions

**Q: What is the difference between encoding, encryption and hashing?**

A: Encoding changes the format so data can be carried, like Base64. It has no key and anyone can reverse it. Encryption hides data with a key, and only a key holder can reverse it. Hashing makes a fixed-size fingerprint and is one way, so there is nothing to decrypt.

**Q: Why does TLS use both asymmetric and symmetric cryptography?**

A: Symmetric ciphers like AES are very fast, but both sides need the same key. Asymmetric cryptography solves that key problem, but it is slow and only handles small data. So TLS uses asymmetric math in the handshake to agree a key and prove identity, then AES-GCM or ChaCha20-Poly1305 for the actual data.

**Q: Compare RSA and ECC.**

A: Both are asymmetric. RSA relies on the difficulty of factoring large numbers, and ECC relies on the elliptic curve discrete log problem. ECC gives the same strength with much smaller keys: 256-bit ECC is similar to 3072-bit RSA. Smaller keys mean smaller handshakes and less CPU work for the private key, so ECC is the usual choice for new systems.

**Q: What is a salt, and why is it not secret?**

A: A salt is a random value added to each password before hashing, and it is stored with the hash. It makes the same password produce different hashes, which defeats rainbow tables and forces attackers to crack each user separately. It does not need to be secret to do that. For real password storage I would use it with a slow function like Argon2id.

**Q: What is the difference between a hash, an HMAC and a digital signature?**

A: A hash has no key, so it only detects accidental change. An HMAC uses a shared secret, so it proves integrity and that a key holder made it, but it cannot prove which of the two parties did. A digital signature uses a private key, and anyone can verify it with the public key, so it also gives non-repudiation.

**Q: Explain Diffie-Hellman simply. Does it stop a man-in-the-middle attack?**

A: Both sides pick secrets, mix each secret with a public value, swap the mixtures, and mix again with their own secret. Both reach the same value, but the secret was never sent. Mixing is easy and un-mixing is the hard discrete log problem. Alone it does not stop a man-in-the-middle, because it proves no identity. TLS fixes this with a signature from the server's certificate key.

**Q: What is forward secrecy, and how does TLS provide it?**

A: Forward secrecy means a stolen long-term key cannot decrypt past recorded sessions. TLS provides it with ephemeral ECDHE, where each session creates a fresh key pair and deletes it. The long-term key only signs the handshake, so it never protects the session key. TLS 1.3 removed static RSA key exchange for this reason.

**Q: Why is ECB bad, and why is GCM preferred?**

A: ECB encrypts each block on its own, so identical plaintext blocks make identical ciphertext blocks and patterns leak. GCM uses a counter, so no two blocks use the same keystream, and it adds an authentication tag. It gives privacy and integrity together, and it runs fast with hardware support. The key rule is to never reuse a nonce with the same key.

**Q: Walk me through the cryptography in a TLS 1.3 connection.**

A: The client and server exchange ephemeral public values and run ECDHE, for example with X25519. They use HKDF to derive traffic keys. The server sends its certificate, then signs the handshake hash to prove it owns the private key. Both send Finished messages, which are HMACs over the handshake, so tampering is detected. Then application data flows under AES-GCM or ChaCha20-Poly1305.

**Q: Is Cisco type 7 password storage secure?**

A: No. Type 7 is reversible with a public algorithm, so it is really obfuscation. It only stops casual viewing. For local accounts and the enable password, I would use a one-way type 8 or type 9 secret, or central authentication.

**Q: You receive a file and a SHA-256 value on the same web page. What does checking the hash prove?**

A: It proves the file was not corrupted in transit. It does not prove the file is genuine, because an attacker who can change the file on that page can also change the hash. To prove the source, you need a digital signature, or a hash delivered through a separately trusted channel.

**Q: What does quantum computing change?**

A: A large quantum computer would break RSA, DH and ECC, using Shor's algorithm. Symmetric ciphers and hashes are much less affected, so AES-256 remains fine. Because traffic can be recorded now and decrypted later, TLS is already moving to hybrid key exchange like X25519MLKEM768, which combines X25519 with NIST's ML-KEM.

## Quick Recap

- Encoding changes format, encryption hides data with a key, and hashing makes a one-way fingerprint.
- Symmetric ciphers (AES) are fast but need a shared key. Asymmetric cryptography (RSA, ECC) solves key sharing and signatures, but it is slow.
- Use AEAD modes like AES-GCM, never ECB, and never repeat a nonce under one key.
- SHA-256 is the standard hash. Add a salt and a slow function for passwords, and use HMAC when you need a key.
- Digital signatures prove who made something, using a private key to sign and a public key to verify.
- Diffie-Hellman creates a shared secret in public. Ephemeral DH gives forward secrecy, and a signature stops a man in the middle.
- TLS 1.3 combines them all: ECDHE for keys, HKDF to derive, signatures for identity, HMAC for handshake checks, and AEAD for data.
