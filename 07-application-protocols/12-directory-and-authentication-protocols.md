# Directory and Authentication Protocols

**Level:** Advanced | **Prerequisites:** [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md), [DNS Basics](01-dns-basics.md), [TLS and HTTPS](07-tls-and-https.md), [Cryptography Basics](../09-security/02-cryptography-basics.md)

## In One Line

A directory stores who everyone is, and authentication protocols prove it: LDAP reads the directory, Kerberos gives tickets inside a company network, RADIUS and TACACS+ control logins to network gear and network access, and OAuth, OIDC and SAML carry identity across the web.

## Why It Matters

Picture a company with 5,000 staff, 300 switches, 40 VPN gateways and 200 applications. If each system keeps its own list of users, one new hire needs 200 accounts. When that person leaves, somebody must find and remove all of them. One missed account is a security hole.

The fix is central identity. Keep each person in one place. Let every other system ask that place. Disable one account and access ends everywhere.

Different systems ask in different ways, so there are several protocols:

| Who is asking | Typical protocol |
|---|---|
| An application that wants to look up a user | LDAP or LDAPS |
| A Windows laptop opening a file share | Kerberos |
| A Wi-Fi access point or VPN gateway | RADIUS |
| A router, when an admin logs in over SSH | TACACS+ (or RADIUS) |
| A web app that wants "log in with your company account" | SAML or OIDC |
| An app that wants limited access to your data in another app | OAuth 2.0 |

Interviewers ask about this area for three reasons:

- It tests whether you can separate similar words: authentication vs authorization, LDAP vs Active Directory, OAuth vs SAML, RADIUS vs TACACS+.
- Kerberos is a classic "explain it step by step" question.
- Login failures are real outages. A 6-minute clock error can stop a whole office from logging in. An engineer who knows why is valuable.

## Simple Explanation

Imagine a large office building with a theme park next door.

**The directory is the staff list.** It holds each person's name, department, email and group. LDAP is the way you ask the staff list a question, like calling the front desk and saying "Which department is Alice in?"

**AAA is the security guard's job.** AAA stands for Authentication, Authorization, Accounting.

- Authentication: "Who are you? Show your ID."
- Authorization: "What may you do? You can enter floor 3, not floor 9."
- Accounting: "What did you do? You entered at 09:02 and left at 17:30."

**RADIUS and TACACS+ are the guard phoning head office.** The guard at a side door does not keep a list of all staff. The guard phones head office and asks "Is this person allowed in?"

**Kerberos is the theme park.** You show your ID once at the main gate and get a wristband. That wristband is the Ticket Granting Ticket. At each ride you do not show your ID again. You take your wristband to a ticket booth, and the booth gives you a ticket for that one ride. The ride operator checks only the ticket.

**SAML is a signed letter from your home office.** You visit a partner company. They do not know you. You bring a sealed letter from your employer saying "This is Alice, she is in Finance." They trust your employer's seal.

**OAuth is a valet key.** You give a parking valet a special key. It starts the car but does not open the boot. You did not give your full key, and you can cancel the valet key later. An app gets a limited token, not your password.

Where these analogies stop being true:

- A real wristband can be seen and copied. A Kerberos ticket is encrypted, and the user cannot read it. Stolen tickets can still be misused, though, as [Going Deeper](#going-deeper) shows.
- A guard's phone call is private. Classic RADIUS hides only the password, and everything else travels in clear text.
- A valet key says nothing about who the driver is. That is correct for OAuth too, and it is the reason OIDC exists.

## How It Works

### AAA: the framework behind everything else

AAA is a security model, not one protocol. Protocols carry out its parts.

| Part | Question | Example |
|---|---|---|
| Authentication (AuthN) | Who are you? | Password, certificate, security key |
| Authorization (AuthZ) | What may you do? | Alice may run `show` commands, not `configure` |
| Accounting | What did you do? | "alice ran `reload` at 14:02 from 10.20.0.50" |

**Authentication factors** are the kinds of proof a person can give:

- Something you know: a password or PIN.
- Something you have: a phone or hardware security key.
- Something you are: a fingerprint or face.

MFA (multi-factor authentication) means using two or more different kinds. Two passwords are not MFA, because they are the same kind.

**Order matters.** Authorization comes after authentication. You cannot decide what someone may do until you know who they are. Accounting runs through the whole session.

### What a directory is

A directory is a database built for many reads and few writes. People look up an email address far more often than they change it. It is arranged as a tree, not as tables.

Examples of directory products: Microsoft Active Directory, OpenLDAP, FreeIPA, 389 Directory Server.

### LDAP: the language for talking to a directory

LDAP means Lightweight Directory Access Protocol. It is an open standard. The current version is LDAPv3, described in the RFC 4510 series (RFC 4511 is the protocol itself).

Basic facts:

- Port: TCP 389 for LDAP, TCP 636 for LDAPS.
- Encoding: binary, using ASN.1 BER (a compact way to pack structured data). You cannot read it like HTTP.
- Model: client sends a request, server sends a response.

**The tree.** The structure is called the DIT (Directory Information Tree).

```text
                 DC=example,DC=com        <- root of this tree
                 /               \
           OU=Staff            OU=Groups
           /       \                 \
  CN=Alice Khan   CN=Bilal Ahmed    CN=VPN-Users
```

Each short label has a meaning:

| Label | Stands for | Meaning |
|---|---|---|
| DC | Domain Component | One part of a domain name |
| OU | Organizational Unit | A folder for grouping entries |
| CN | Common Name | The name of the entry |
| UID | User ID | Login name, common in Unix directories |

**DN (Distinguished Name)** is the full path to an entry, written from the entry up to the root:

```text
CN=Alice Khan,OU=Staff,DC=example,DC=com
```

Think of it as a full postal address. The **RDN (Relative Distinguished Name)** is only the first part, `CN=Alice Khan`, like a house number on one street.

**Entries and attributes.** Each item in the tree is an entry. An entry holds attributes, which are name and value pairs. An attribute can have many values.

```text
dn: CN=Alice Khan,OU=Staff,DC=example,DC=com
objectClass: user
cn: Alice Khan
mail: alice@example.com
memberOf: CN=VPN-Users,OU=Groups,DC=example,DC=com
```

**Schema** is the rulebook. It says which object classes exist and which attributes each one may hold.

**LDAP operations:**

| Operation | What it does |
|---|---|
| Bind | Log in to the directory |
| Search | Find entries |
| Compare | Check if an attribute has a given value |
| Add, Delete | Create or remove an entry |
| Modify | Change attributes |
| ModifyDN | Rename or move an entry |
| Unbind | End the session |
| Abandon | Cancel a running request |
| Extended | Extra functions, such as StartTLS |

**Bind types:**

- Anonymous: no identity at all. Often blocked.
- Simple: a DN and a password. The password crosses the network as it is, so TLS is required.
- SASL: a framework for stronger methods. GSSAPI (Kerberos) and EXTERNAL (client certificate) are the common ones.

**A search has four main parts:**

1. Base: where in the tree to start.
2. Scope: `base` (only that entry), `one` (direct children), or `sub` (the whole subtree).
3. Filter: the condition.
4. Attributes: which fields to return.

**Filters** use prefix notation, so the operator comes first:

```text
(sAMAccountName=alice)                      equal to
(mail=*)                                    attribute exists
(cn=Ali*)                                   starts with
(&(objectClass=user)(department=Sales))     AND
(|(department=Sales)(department=Support))   OR
(!(department=Sales))                       NOT
```

**How an app logs a user in using LDAP.** This "search then bind" pattern is very common:

```text
App 10.20.0.80                            DC01 10.10.10.10
   |--- TCP handshake to port 636 ------------->|
   |<== TLS handshake =========================>|
   |--- Bind (service account, password) ------>|
   |<-- BindResponse: success ------------------|
   |--- Search (sAMAccountName=alice) --------->|
   |<-- SearchResultEntry (DN of Alice) --------|
   |<-- SearchResultDone -----------------------|
   |--- Bind (Alice's DN, typed password) ----->|
   |<-- BindResponse: success or code 49 -------|
   |--- Unbind -------------------------------->|
```

The app first finds Alice's DN. Then it tries to bind as Alice with the password she typed. If the bind works, the password is right.

**Result codes you should know:**

| Code | Name | Usual meaning |
|---|---|---|
| 0 | success | It worked |
| 8 | strongerAuthRequired | Server wants signing or TLS |
| 32 | noSuchObject | Wrong base DN |
| 49 | invalidCredentials | Wrong DN or password |
| 50 | insufficientAccessRights | Logged in, but not allowed |
| 52 | unavailable | Server cannot serve the request now |

### LDAPS and StartTLS

Plain LDAP on port 389 sends data in clear text. There are two ways to add TLS. TLS itself is covered in [TLS and HTTPS](07-tls-and-https.md).

| | LDAPS | StartTLS |
|---|---|---|
| Port | TCP 636 | TCP 389 |
| When TLS starts | At once, before any LDAP | After an LDAP Extended request |
| Like | HTTPS | A plain call that upgrades |
| Standard status | Very common, but not defined in the LDAPv3 RFCs | Defined by RFC 4511 and RFC 4513 |
| URL | `ldaps://dc01.example.com` | `ldap://dc01.example.com` plus a StartTLS option |
| Main risk | Certificate problems | If TLS is not enforced, an attacker can remove the upgrade step |

Both need a valid certificate on the server. The name your client connects to must match the certificate, so connecting to an IP address often fails.

On Active Directory, port 636 only works after a suitable certificate is installed on the domain controller.

Fuller truth: Windows tools talk to AD on port 389 and are still protected, because they use SASL with Kerberos signing and sealing. Port 389 does not always mean clear text. Simple binds on port 389 are clear text.

### Active Directory: where LDAP and Kerberos meet

Active Directory (AD) is Microsoft's directory service. The link to remember:

> LDAP is a protocol. Active Directory is a product that speaks LDAP.

It is like HTTP and a website. AD uses LDAP to read data, Kerberos to log people in, DNS to find servers, and SMB to share policy files.

**Key terms:**

| Term | Plain meaning |
|---|---|
| Domain | A group of users and computers under one database, such as `example.com` |
| Domain controller (DC) | A server that holds a copy of that database |
| Tree | Domains that share a name space, such as `sales.example.com` under `example.com` |
| Forest | The top boundary: one or more trees that share a schema |
| Trust | A link that lets one domain accept logins from another |
| OU | A folder used to organise objects and apply policy |
| GPO | Group Policy Object, a set of settings pushed to computers |
| Global Catalog | A partial copy of all objects in the forest, for fast searches |
| Site | A group of well-connected subnets, used to pick a nearby DC |
| `NTDS.dit` | The database file on each DC |

**Replication** is multi-master. You can change data on any DC, and the DCs copy the change to each other.

Fuller truth: a few jobs must have one owner. These are the five FSMO roles (Flexible Single Master Operations):

- Forest-wide: Schema Master, Domain Naming Master.
- Domain-wide: RID Master, PDC Emulator, Infrastructure Master.

The PDC Emulator matters for Kerberos, because it is the top time source in a domain.

**What happens when a domain PC logs on:**

```text
WS-12 (10.20.0.50)                   DC01 (10.10.10.10)

1. DNS (53)         "Where are the DCs for example.com?"
                    SRV _ldap._tcp.dc._msdcs.example.com
2. CLDAP (UDP 389)  "Are you alive? Which site am I in?"
3. Kerberos (88)    Get a TGT, then service tickets
4. LDAP (389)       Read user, group and policy data
5. SMB (445)        Download policies from \\example.com\SYSVOL
```

CLDAP means Connectionless LDAP, a small LDAP query over UDP. AD uses it to find a DC quickly.

The SRV records are explained in [DNS Record Types](02-dns-records.md). If DNS is wrong, AD breaks. Domain PCs must use the AD DNS servers, not a public resolver.

**AD ports a firewall must allow from clients to DCs:**

| Service | Port |
|---|---|
| DNS | TCP and UDP 53 |
| Kerberos | TCP and UDP 88 |
| Kerberos password change (kpasswd) | TCP and UDP 464 |
| NTP (Windows Time) | UDP 123 |
| LDAP, CLDAP | TCP and UDP 389 |
| LDAPS | TCP 636 |
| Global Catalog (LDAP, LDAPS) | TCP 3268, TCP 3269 |
| SMB | TCP 445 |
| RPC endpoint mapper | TCP 135 |
| Dynamic RPC | TCP 49152-65535 |

**AD is not Entra ID.** Microsoft Entra ID was named Azure AD until 2023. It is a cloud identity service built on web protocols such as OIDC, OAuth and SAML. AD uses LDAP and Kerberos inside a network. Despite the old name, Entra ID is not AD running in the cloud. Many companies link the two and sync users between them.

### Kerberos: the terms first

Kerberos is a network login protocol from MIT. Version 5 is defined in RFC 4120. It runs on port 88, on both UDP and TCP.

Its main idea: prove who you are without sending your password over the network, and log in once for many services.

| Term | Plain meaning |
|---|---|
| Principal | Any identity, such as `alice@EXAMPLE.COM` |
| Realm | The Kerberos area, written in capitals. In AD, it matches the domain name |
| KDC | Key Distribution Center, the trusted server. In AD, it runs on every DC |
| AS | Authentication Service, the part of the KDC that checks your identity |
| TGS | Ticket Granting Service, the part of the KDC that gives service tickets |
| TGT | Ticket Granting Ticket, your "wristband" |
| Service ticket | A pass for one service |
| Session key | A short-term secret shared by two parties |
| Authenticator | A small encrypted note holding your name and the current time |
| SPN | Service Principal Name, the name of a service, such as `cifs/fs01.example.com` |
| Keytab | A file holding a service's long-term key, used on Linux |
| `krbtgt` | The special account whose key protects all TGTs |

**The starting secret.** Every principal shares one long-term key with the KDC. For a user, the key is derived from the password. For a service, it comes from the service account or computer account.

Fuller truth: for AES, the key comes from the password plus a salt, using many hash rounds. It is not a plain hash of the password.

### Kerberos step by step

Alice sits at WS-12 and wants to open `\\fs01.example.com\reports`.

```text
  +---------+                          +---------------------+
  |         |  1. AS-REQ               |   KDC on DC01       |
  |         |------------------------->|  +---------------+  |
  | Client  |  2. AS-REP (TGT)         |  |  AS           |  |
  | WS-12   |<-------------------------|  +---------------+  |
  | (Alice) |  3. TGS-REQ (TGT + SPN)  |  +---------------+  |
  |         |------------------------->|  |  TGS          |  |
  |         |  4. TGS-REP (ticket)     |  +---------------+  |
  |         |<-------------------------|                     |
  +---------+                          +---------------------+
     |   ^
     |   |  5. AP-REQ (service ticket + authenticator)
     |   |  6. AP-REP (optional: server proves itself)
     v   |
  +---------+
  |  fs01   |  never talks to the KDC for this login
  +---------+
```

**Part 1: Get the TGT (once per login)**

**Step 1, AS-REQ.** The client tells the AS: "I am alice. I want a ticket for `krbtgt/EXAMPLE.COM`." It adds a timestamp encrypted with Alice's key. This is called pre-authentication. It proves she knows the password.

**Step 2, AS-REP.** The AS decrypts the timestamp using its copy of Alice's key. If the time is fresh, the AS creates a new session key and returns two things:

- The TGT, encrypted with the `krbtgt` key. Alice cannot read it. It holds her name, the session key and the expiry time.
- A part encrypted with Alice's key. It holds the same session key.

The client decrypts the second part using the key from her password. Now it holds the TGT and the session key. From this point, it no longer needs the password for this exchange.

**Part 2: Get a service ticket (once per service)**

**Step 3, TGS-REQ.** The client sends the TGS three things: the TGT, the SPN it wants (`cifs/fs01.example.com`), and an authenticator encrypted with the session key.

**Step 4, TGS-REP.** The TGS decrypts the TGT with the `krbtgt` key and finds the session key. It uses that to read the authenticator. The name and time must match. Then it creates a new service session key and returns:

- The service ticket, encrypted with fs01's key. Alice cannot read it. It holds her name, the service session key and the expiry time.
- A part encrypted with the first session key. It holds the service session key.

**Part 3: Use the service**

**Step 5, AP-REQ.** The client sends fs01 the service ticket and a new authenticator, encrypted with the service session key. This travels inside the application protocol, here SMB.

**Step 6, AP-REP (optional).** fs01 decrypts the ticket with its own key, finds the service session key, and reads the authenticator. If that is good, Alice is logged in. If the client asked for mutual authentication, fs01 replies with a value encrypted with the service session key. This proves fs01 is the real server.

**Why this design is clever:**

- The password never crosses the network.
- The KDC does not need to remember sessions. The TGT carries its own state.
- fs01 does not contact the KDC. It proves the ticket is real because only the KDC could have encrypted it with fs01's key.

**Who can open what:**

| Item | Encrypted with | Who can read it |
|---|---|---|
| TGT | `krbtgt` key | KDC only |
| Session key in AS-REP | Alice's key | Alice and KDC |
| Service ticket | fs01's key | fs01 and KDC |
| Service session key in TGS-REP | First session key | Alice and KDC |
| Authenticator to TGS | First session key | Alice and KDC |
| Authenticator to fs01 | Service session key | Alice and fs01 |

**Common lifetimes in AD (defaults):** TGT 10 hours, renewable for up to 7 days, service ticket 10 hours. Other Kerberos systems use other values, set in their own config.

Terminology note: people often say "TGS ticket" to mean a service ticket. TGS is really the service that issues it.

### Clock skew: why time matters

**Clock skew** is the difference between two computers' clocks.

An authenticator holds the current time. The server accepts it only if that time is close to its own clock. It also remembers recent authenticators in a replay cache and refuses repeats.

This stops a **replay attack**, where an attacker records a valid message and sends it again later. Old authenticators are refused, and recent ones are in the cache.

The normal limit is **5 minutes**. In AD this is the policy "Maximum tolerance for computer clock synchronization". In MIT Kerberos it is `clockskew` in `krb5.conf`, 300 seconds by default.

Worked numbers, with the DC at 10:00:00 UTC:

| Client clock (UTC) | Gap | Result |
|---|---|---|
| 10:04:00 | 4 min | Accepted |
| 09:56:30 | 3.5 min | Accepted |
| 10:06:30 | 6.5 min | Refused |
| 09:54:00 | 6 min | Refused |

The error is `KRB_AP_ERR_SKEW`, error code 37 (0x25). On a DC this can appear in event 4771 with failure code 0x25.

Time zones do not matter. Kerberos compares absolute time (UTC). A client in Karachi and a DC in London can both be correct even though their local clocks show different hours.

In AD, time flows down a chain:

```text
Client PC -> its DC -> domain PDC Emulator -> forest root PDC
          -> external NTP source
```

Time synchronisation uses NTP, covered in [Network Management Protocols](11-network-management-protocols.md).

### RADIUS

RADIUS means Remote Authentication Dial-In User Service. It began at Livingston Enterprises in 1991 for dial-up modems. Today it controls Wi-Fi, VPN and switch port access.

- Authentication: RFC 2865, UDP 1812.
- Accounting: RFC 2866, UDP 1813.
- Old ports 1645 and 1646 still appear on older gear.

**The confusing part: who is the client?** The user is not the RADIUS client. The network device is. That device is called a **NAS (Network Access Server)**. A NAS can be a switch, an access point or a VPN gateway.

```text
 Laptop            NAS (RADIUS client)      RADIUS server     Directory
 (user)            switch, AP or VPN        NPS, FreeRADIUS   AD or LDAP
   |                     |                        |               |
   |-- user + password ->|                        |               |
   |                     |-- Access-Request ----->|               |
   |                     |                        |-- check ----->|
   |                     |                        |<- result -----|
   |                     |<- Access-Accept -------|               |
   |<-- connected -------|                        |               |
   |                     |-- Accounting Start --->|               |
   |                     |<- Accounting Response -|               |
```

Common RADIUS servers: Microsoft NPS (Network Policy Server), FreeRADIUS, Cisco ISE.

**Packet layout:**

| Field | Size | Purpose |
|---|---|---|
| Code | 1 byte | Packet type |
| Identifier | 1 byte | Matches a reply to a request |
| Length | 2 bytes | Total packet size |
| Authenticator | 16 bytes | Used to hide the password and check replies |
| Attributes | Variable | The actual data, in type-length-value form |

**Codes:**

| Code | Name |
|---|---|
| 1 | Access-Request |
| 2 | Access-Accept |
| 3 | Access-Reject |
| 4 | Accounting-Request |
| 5 | Accounting-Response |
| 11 | Access-Challenge (server asks for more, used by EAP and one-time codes) |

**Useful attributes:** User-Name, User-Password, NAS-IP-Address, Calling-Station-Id (often the client MAC address), Filter-Id, Vendor-Specific (26), and for dynamic VLANs, Tunnel-Type, Tunnel-Medium-Type and Tunnel-Private-Group-ID.

That last group means the server can say "put this user in VLAN 20" in its Access-Accept.

**The shared secret.** The NAS and the server share a password called the shared secret. It hides the User-Password attribute and signs replies. The rest of the packet, including the username, is visible on the network.

**Accounting** uses records with an Acct-Status-Type of Start, Interim-Update or Stop. They carry the session ID, session time and bytes in and out. This is how ISPs once billed by usage.

**RADIUS with EAP.** EAP (Extensible Authentication Protocol) lets the laptop and server run a stronger login, such as with certificates, using RADIUS as the carrier. This is the basis of 802.1X, covered in [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md) and [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).

### TACACS+

TACACS+ means Terminal Access Controller Access-Control System Plus. Cisco created it. It is now documented in RFC 8907, which is an Informational RFC, so it describes the protocol and is not a formal Internet Standard. Despite the name, it is not compatible with the older TACACS.

- Transport: TCP port 49.
- Use: admin logins to routers, switches and firewalls.
- Key strength: it separates authentication, authorization and accounting, and it can check every single command.

A router admin session with per-command checks:

```text
Router                               TACACS+ server (TCP 49)
  |--- Authentication START (alice) ---------->|
  |<-- REPLY: GETPASS -------------------------|
  |--- CONTINUE (password) ------------------->|
  |<-- REPLY: PASS ----------------------------|
  |--- Authorization REQUEST (shell) --------->|
  |<-- RESPONSE: PASS_ADD (priv-lvl=15) -------|
  |--- Accounting REQUEST (start) ------------>|
  |<-- REPLY: SUCCESS -------------------------|
  |      alice types: show running-config      |
  |--- Authorization REQUEST (cmd=show ...) -->|
  |<-- RESPONSE: PASS_ADD ---------------------|
  |      alice types: reload                   |
  |--- Authorization REQUEST (cmd=reload) ---->|
  |<-- RESPONSE: FAIL -------------------------|
```

The router shows "Command authorization failed" and does not reload. A junior engineer can read configs but not restart devices.

### RADIUS vs TACACS+

| Feature | RADIUS | TACACS+ |
|---|---|---|
| Created by | Livingston Enterprises | Cisco |
| Standard | IETF Standards Track (RFC 2865) | Informational (RFC 8907) |
| Transport | UDP | TCP |
| Ports | 1812 auth, 1813 accounting | 49 |
| What is protected | Password only | Whole packet body |
| AAA design | Authentication and authorization joined in one reply | All three separate |
| Per-command authorization | Not in the base protocol | Yes |
| Main use | Network access: Wi-Fi, VPN, 802.1X | Device administration |
| EAP and 802.1X support | Yes | No |
| Accounting | Strong, widely used | Yes, including commands |
| Vendor support | Very wide | Cisco first, now many others |

An easy way to remember: **RADIUS for users reaching the network. TACACS+ for admins managing the network.**

Fuller truth on protection: TACACS+ body protection is an MD5-based pad, which RFC 8907 calls obfuscation, not real encryption. The 12-byte header stays clear. Still, it hides more than RADIUS does.

### OAuth 2.0: delegated access

OAuth 2.0 (RFC 6749) answers this: "How can an app use my data in another service without learning my password?"

It is about **authorization**, not login. Example: a photo printing app wants to read your photos from a storage service.

**Roles:**

| Role | Meaning | Example |
|---|---|---|
| Resource owner | The user | Alice |
| Client | The app that wants access | Photo printing app |
| Authorization server | Gives out tokens | `auth.example.com` |
| Resource server | The API holding the data | `api.example.com` |

**Key words:**

- **Access token**: a short-lived pass sent to the API, usually as `Authorization: Bearer <token>` over HTTPS.
- **Refresh token**: a longer-lived token used to get new access tokens.
- **Scope**: what the token allows, such as `photos.read`.
- **Consent**: the screen where the user approves the request.

**Authorization code flow with PKCE.** This is the normal modern flow. PKCE (Proof Key for Code Exchange, RFC 7636, said "pixy") stops a stolen code being used by someone else.

```text
Browser            App            Auth server          API
   |-- click ------->|                 |                 |
   |<- redirect -----|                 |                 |
   |-- /authorize (scope, challenge) ->|                 |
   |<- login and consent screen ------>|                 |
   |<- redirect with short code -------|                 |
   |-- code -------->|                 |                 |
   |                 |-- code + verifier ->|             |
   |                 |<- access token -----|             |
   |                 |-- Bearer token ------------------>|
   |                 |<- photos -------------------------|
```

The app never sees the password. The user types it only on the authorization server's page.

**Other flows:**

- Client credentials: a machine calling a machine, with no user.
- Device authorization (RFC 8628): smart TVs and command line tools. The device shows a code and the user approves on a phone.

The implicit flow and the password flow are old. The security best practice in RFC 9700 says not to use them. OAuth 2.1 is an IETF draft that removes them.

### OIDC: login on top of OAuth

OAuth alone does not tell the app who the user is. OIDC (OpenID Connect) adds that. It is built on OAuth 2.0 and published by the OpenID Foundation, not as an RFC.

OIDC adds an **ID token**. It is a signed JWT (JSON Web Token, RFC 7519) that says who logged in. "Sign in with your company account" on a modern web or mobile app is usually OIDC.

### SAML: single sign-on with signed XML

SAML means Security Assertion Markup Language. SAML 2.0 is an OASIS standard from 2005. It is common for company single sign-on to business web apps.

**SSO (single sign-on)** means one login gives you access to many apps.

| Role | Meaning |
|---|---|
| IdP (Identity Provider) | The system that logs the user in, such as Entra ID or Okta |
| SP (Service Provider) | The app the user wants |
| Assertion | A signed XML statement: "This is Alice, she logged in at this time" |

**SP-initiated login flow:**

```text
Browser                  SP app.example.com     IdP idp.example.com
   |-- GET /reports ---------->|                        |
   |<- redirect + AuthnRequest-|                        |
   |-- AuthnRequest ------------------------------------>|
   |<-- login page, password and MFA ------------------->|
   |<-- HTML form holding signed SAML Response ----------|
   |-- POST Response to /acs ->|                        |
   |              SP checks signature, audience, time   |
   |<- session cookie + page --|                        |
```

ACS means Assertion Consumer Service, the SP's URL that receives the response.

Note that the SP and IdP do not talk directly during login. The browser carries everything. They only need to exchange metadata, including the IdP's public certificate, once at setup.

Analogy limit: a sealed letter stays valid for a long time. A SAML assertion is valid for only a few minutes, so clocks matter here too.

### Choosing between the protocols

| Protocol | Main job | Format | Runs over | Typical place |
|---|---|---|---|---|
| LDAP | Read and write a directory | Binary (BER) | TCP 389, 636 | App to directory |
| Kerberos | Ticket login | Binary (ASN.1) | UDP and TCP 88 | Inside a company network |
| RADIUS | Network access AAA | Binary attributes | UDP 1812, 1813 | NAS to server |
| TACACS+ | Device admin AAA | Binary | TCP 49 | Router to server |
| SAML 2.0 | Web SSO | XML | HTTPS via browser | Business web apps |
| OAuth 2.0 | Delegated access | Tokens | HTTPS | App to API |
| OIDC | Login for web and mobile | JWT | HTTPS | Modern apps |

## Example

### Example 1: LDAPS search for a user

Linux with OpenLDAP tools, searching AD:

```bash
# Linux
ldapsearch -H ldaps://dc01.example.com:636 \
  -D "CN=svc-app,OU=Service Accounts,DC=example,DC=com" -W \
  -b "OU=Staff,DC=example,DC=com" -s sub \
  "(&(objectClass=user)(sAMAccountName=alice))" \
  cn mail memberOf
```

Flags: `-H` server, `-D` bind DN, `-W` prompt for password, `-b` search base, `-s` scope. Then come the filter and the attributes.

```text
# extended LDIF
#
# LDAPv3
# base <OU=Staff,DC=example,DC=com> with scope subtree
# filter: (&(objectClass=user)(sAMAccountName=alice))
# requesting: cn mail memberOf
#

# Alice Khan, Staff, example.com
dn: CN=Alice Khan,OU=Staff,DC=example,DC=com
cn: Alice Khan
mail: alice@example.com
memberOf: CN=VPN-Users,OU=Groups,DC=example,DC=com

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

`numResponses: 2` counts one entry plus one "search done" message.

The same on Windows with the Active Directory PowerShell module:

```powershell
# Windows
Get-ADUser alice -Properties mail,memberOf
```

```text
DistinguishedName : CN=Alice Khan,OU=Staff,DC=example,DC=com
Enabled           : True
GivenName         : Alice
mail              : alice@example.com
MemberOf          : {CN=VPN-Users,OU=Groups,DC=example,DC=com}
Name              : Alice Khan
ObjectClass       : user
SamAccountName    : alice
Surname           : Khan
UserPrincipalName : alice@example.com
```

Check that a DC can be found and its port is open:

```bash
# Linux
dig +short SRV _ldap._tcp.dc._msdcs.example.com
# 0 100 389 dc01.example.com.
openssl s_client -connect dc01.example.com:636 </dev/null
```

```powershell
# Windows
nslookup -type=SRV _ldap._tcp.dc._msdcs.example.com
Test-NetConnection dc01.example.com -Port 636
```

### Example 2: Kerberos tickets on real systems

Setup: client WS-12 at 10.20.0.50, DC01 at 10.10.10.10, file server fs01 at 10.10.10.30.

On Linux, get a TGT, then a service ticket:

```bash
# Linux, MIT Kerberos
kinit alice@EXAMPLE.COM
kvno cifs/fs01.example.com
klist
```

```text
Password for alice@EXAMPLE.COM:
cifs/fs01.example.com@EXAMPLE.COM: kvno = 3
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: alice@EXAMPLE.COM

Valid starting     Expires            Service principal
10/30/26 09:00:12  10/30/26 19:00:12  krbtgt/EXAMPLE.COM@EXAMPLE.COM
        renew until 11/06/26 09:00:12
10/30/26 09:00:30  10/30/26 19:00:12  cifs/fs01.example.com@EXAMPLE.COM
```

`kvno` here means key version number, a counter that goes up when the service key changes.

Read it like this: the `krbtgt` line is the TGT, and the `cifs` line is the service ticket. CIFS is the SPN prefix used for SMB file shares.

On Windows:

```powershell
# Windows
klist
```

```text
Current LogonId is 0:0x3e7a1

Cached Tickets: (2)

#0>  Client: alice @ EXAMPLE.COM
     Server: krbtgt/EXAMPLE.COM @ EXAMPLE.COM
     KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
     Start Time: 10/30/2026 9:00:12 (local)
     End Time:   10/30/2026 19:00:12 (local)
     Renew Time: 11/6/2026 9:00:12 (local)
     Kdc Called: dc01.example.com

#1>  Client: alice @ EXAMPLE.COM
     Server: cifs/fs01.example.com @ EXAMPLE.COM
     KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
     Start Time: 10/30/2026 9:00:30 (local)
     End Time:   10/30/2026 19:00:12 (local)
     Kdc Called: dc01.example.com
```

A packet capture of the same login looks like this (simplified):

```text
No  Source       Destination  Proto  Info
1   10.20.0.50   10.10.10.10  KRB5   AS-REQ
2   10.10.10.10  10.20.0.50   KRB5   KRB Error: PREAUTH_REQUIRED
3   10.20.0.50   10.10.10.10  KRB5   AS-REQ (with pre-auth)
4   10.10.10.10  10.20.0.50   KRB5   AS-REP
5   10.20.0.50   10.10.10.10  KRB5   TGS-REQ
6   10.10.10.10  10.20.0.50   KRB5   TGS-REP
7   10.20.0.50   10.10.10.30  SMB2   Session Setup Request (AP-REQ)
8   10.10.10.30  10.20.0.50   SMB2   Session Setup Response (AP-REP)
```

Packet 2 looks like a failure but is normal. The client's first try often has no pre-auth, and the KDC replies that it is required. Do not report this as a fault.

### Example 3: Fixing a clock skew failure

A laptop returns after months in storage. Logins fail.

```bash
# Linux
kinit alice@EXAMPLE.COM
```

```text
kinit: Clock skew too great while getting initial credentials
```

Measure the gap. The sign does not matter, only the size:

```powershell
# Windows: compare this PC's clock to the DC
w32tm /stripchart /computer:dc01.example.com /samples:1 /dataonly
```

```text
Tracking dc01.example.com [10.10.10.10:123].
Collecting 1 samples.
The current time is 10/30/2026 9:02:11 AM.
09:02:11, +421.3070142s
```

The gap is 421 seconds, about 7 minutes. That is above the 300-second limit. Fix the time source:

```powershell
# Windows
w32tm /resync
w32tm /query /status
```

```bash
# Linux with chrony
chronyc makestep
chronyc tracking
```

### Example 4: Testing RADIUS

Linux, FreeRADIUS test tool. The NAS is 10.10.10.5 and the server is 10.10.10.21. `testing123` is a lab secret.

```bash
# Linux
radtest alice Passw0rd 10.10.10.21 0 testing123
```

```text
Sent Access-Request Id 42 from 0.0.0.0:48201
    to 10.10.10.21:1812 length 75
        User-Name = "alice"
        User-Password = "Passw0rd"
        NAS-IP-Address = 10.10.10.5
        NAS-Port = 0
        Message-Authenticator = 0x00
Received Access-Accept Id 42 from 10.10.10.21:1812
    to 0.0.0.0:48201 length 20
```

Long lines are wrapped here. The tool prints the password for your reading. On the wire, it is hidden.

Cisco IOS test from the router:

```text
Router# test aaa group radius alice Passw0rd new-code
User successfully authenticated
```

### Example 5: Cisco IOS with TACACS+ for admins

Goal: admins log in with central accounts, every level 15 command is checked and logged, and a local account works only if servers are down. Addresses match the examples above: 10.10.10.20 is the TACACS+ server.

```text
! Cisco IOS XE
aaa new-model
!
username localadmin privilege 15 secret <strong-password>
!
tacacs server ISE-1
 address ipv4 10.10.10.20
 key <shared-secret>
!
aaa group server tacacs+ TAC-SERVERS
 server name ISE-1
!
ip tacacs source-interface Loopback0
!
aaa authentication login default group TAC-SERVERS local
aaa authorization exec default group TAC-SERVERS local
aaa authorization commands 15 default group TAC-SERVERS local
aaa accounting exec default start-stop group TAC-SERVERS
aaa accounting commands 15 default start-stop group TAC-SERVERS
!
line vty 0 4
 transport input ssh
```

Each `aaa` line is a **method list**: an ordered list of places to try.

`group TAC-SERVERS local` means "ask TACACS+ first, and use the local user only if no server answers". A Reject from the server does **not** fall through to local. Fallback happens on an error or timeout, not on a "no".

`ip tacacs source-interface Loopback0` fixes the source IP. The server only accepts devices it knows, so that IP must match the device entry there.

Safety tip: keep one session open while you apply this. Test from a second session. If you made a mistake, the open session lets you fix it.

### Example 6: One VPN login through the whole chain

Alice connects to a VPN. Firewall (NAS) is 10.10.10.1, NPS is 10.10.10.21, DC01 is 10.10.10.10.

1. Alice's VPN client sends her login to the firewall. See [VPNs](../09-security/07-vpn.md) for the tunnel itself.
2. The firewall sends `Access-Request` to 10.10.10.21 on UDP 1812.
3. NPS checks the password against AD on DC01.
4. NPS sees she is in `VPN-Users` and matches its network policy.
5. NPS sends `Access-Accept` with an attribute such as `Filter-Id = VPN-Staff`.
6. The firewall applies the `VPN-Staff` rule set and sends `Accounting-Request` (Start) to UDP 1813.
7. When she disconnects, the firewall sends `Accounting-Request` (Stop) with the session time and byte counts.

That is authentication in step 3, authorization in steps 4-6, and accounting in steps 6-7.

### Example 7: OAuth messages

Wrapped and not URL-encoded, for easy reading.

Step one, the browser is sent to the authorization server:

```text
GET https://auth.example.com/authorize
    ?response_type=code
    &client_id=photo-app
    &redirect_uri=https://app.example.com/callback
    &scope=photos.read
    &state=af0ifjsldkj
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
```

`state` is a random value that stops cross-site request forgery. `code_challenge` is a hash of a secret the app keeps.

After approval, the app swaps the code for a token, proving it holds that secret:

```text
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://app.example.com/callback
&client_id=photo-app
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

```text
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "photos.read"
}
```

The app then calls the API:

```text
GET /v1/photos HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

### Example 8: A trimmed SAML assertion

```text
<saml:Assertion ID="_a1b2c3" IssueInstant="2026-10-30T09:05:00Z">
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  <ds:Signature>...</ds:Signature>
  <saml:Subject>
    <saml:NameID>alice@example.com</saml:NameID>
  </saml:Subject>
  <saml:Conditions NotBefore="2026-10-30T09:04:30Z"
                   NotOnOrAfter="2026-10-30T09:09:30Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://app.example.com</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AttributeStatement>...</saml:AttributeStatement>
</saml:Assertion>
```

The SP must check the signature, that `Audience` is itself, and that the time is inside the `Conditions` window.

## Going Deeper

### LDAP details that bite in production

**The empty password trap.** LDAP allows an "unauthenticated bind": a DN with an empty password. Many servers return success (RFC 4513 warns about this). If an app does "search then bind" and does not reject an empty password itself, anyone can log in as any user by leaving the password blank. Always reject empty passwords in the app, or disable this on the server.

**LDAP injection.** If an app pastes user input straight into a filter, input containing `*`, `(` or `)` can change the query. Escape input as RFC 4515 describes.

**Size limits.** AD returns at most 1,000 entries per page by default. Larger results need the paged results control. If a script finds exactly 1,000 users, suspect this limit.

**Referrals.** A server can answer "this data is on another server" (result code 10). Searching a whole AD domain from the root can return referrals that slow a client. Use the Global Catalog port for forest-wide reads.

**AD bind error details.** Code 49 hides extra data. The text contains a sub-code such as:

| Sub-code | Meaning |
|---|---|
| 52e | Wrong password |
| 525 | User not found |
| 530 | Not allowed to log on at this time |
| 532 | Password expired |
| 533 | Account disabled |
| 701 | Account expired |
| 773 | User must reset password |
| 775 | Account locked |

**Signing and channel binding.** LDAP signing checks that messages were not changed. Channel binding ties an LDAP login to the TLS session, which blocks relay attacks. Microsoft has moved toward requiring both, and defaults depend on the Windows Server version. Test old apps and printers before you enforce them.

**Load balancing.** A VIP (virtual IP) in front of LDAP servers needs a certificate holding the VIP's name. Otherwise TLS checks fail after the change. Note also that AD clients already find DCs by SRV records.

### Kerberos details

**The PAC.** In AD, tickets carry a Privilege Attribute Certificate. It lists the user's SID and group SIDs. The service uses it to decide access. This has two results:

- Group changes do not take effect until the user gets a new ticket. Log off and on, or run `klist purge`.
- Users in very many groups get large tickets. This can break HTTP Negotiate logins with "header too long" errors. It is also why Kerberos moved from UDP to TCP in many cases.

**UDP vs TCP.** If a reply does not fit in UDP, the KDC returns `KRB_ERR_RESPONSE_TOO_BIG` and the client retries on TCP. Modern Windows clients use TCP by default. Firewalls must allow both on port 88.

**Offline checking has a cost.** Because a service does not call the KDC, disabling an account does not kill tickets already issued. They can work until they expire.

**Cross-realm trust.** For a service in another realm, the home KDC gives a referral TGT such as `krbtgt/PARTNER.EXAMPLE@EXAMPLE.COM`. The client takes it to the partner's KDC. The two realms share a trust key.

**Delegation.** Sometimes a web server must reach a database as the user. This is a double hop.

| Type | Risk | Idea |
|---|---|---|
| Unconstrained | High | The server receives the user's TGT and can act as them anywhere |
| Constrained | Lower | The server may act as users only to listed services |
| Resource-based constrained | Lower | The target service lists who may delegate to it |

**NTLM, the fallback.** NTLM is the older Windows challenge-response login. There is no ticket and no KDC involved in the client's steps. The server passes the proof to a DC. Windows falls back to NTLM when Kerberos cannot be used, for example:

- The user connects by IP address, so there is no matching SPN (unless one was registered for that IP).
- The SPN is missing.
- The client cannot reach a DC.

Microsoft has deprecated NTLM. NTLMv1 was removed in Windows 11 24H2 and Windows Server 2025. NTLM is weaker than Kerberos and open to relay attacks, so many teams audit and reduce its use.

**Encryption types.** AES (etype 17 and 18) is the normal choice. RC4 (etype 23) is legacy, and RFC 8429 deprecates it along with 3DES. DES is long gone. Microsoft has been moving Windows away from RC4 by default, so check the behaviour of your patch level. Old appliances that only speak RC4 are a common failure.

**Other mechanisms.**

- PKINIT (RFC 4556) replaces the password step with a certificate. Smart cards use it.
- SPNEGO lets a client and server agree on Kerberos or NTLM. On the web it appears as `Authorization: Negotiate`.

**Kerberos troubleshooting table:**

| Symptom | Likely cause | First check |
|---|---|---|
| `KRB_AP_ERR_SKEW` (37) | Clock gap over 5 minutes | `w32tm`, `chronyc tracking` |
| `KDC_ERR_PREAUTH_FAILED` (24) | Wrong password | Reset, check lockout |
| `KDC_ERR_C_PRINCIPAL_UNKNOWN` (6) | User not found, wrong realm | Spelling, realm in capitals |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` (7) | SPN missing | `setspn -Q cifs/fs01.example.com` |
| `KRB_AP_ERR_MODIFIED` (41) | Server cannot decrypt: duplicate SPN, stale keytab, DNS points to wrong host | `setspn -X`, keytab version |
| `KDC_ERR_ETYPE_NOSUPP` (14) | No shared encryption type | Compare allowed etypes |
| Silent fall back to NTLM | IP used, or SPN missing | Use the name, run `klist` |

Three things must be right for Kerberos: **DNS, time and SPNs**. Check them in that order.

### Attacks that interviewers name

These are the defender's view. More on attacks in general is in [Common Network Attacks](../09-security/03-common-network-attacks.md).

| Attack | How it works | Defence |
|---|---|---|
| Kerberoasting | Any domain user can request a service ticket. It is encrypted with the service account's key, so it can be guessed offline | Long random passwords, gMSA, AES only |
| AS-REP roasting | Accounts with pre-auth disabled give away data encrypted with their key | Keep pre-auth on |
| Golden Ticket | With the `krbtgt` key, an attacker forges any TGT | Protect DCs, reset `krbtgt` twice after a breach |
| Silver Ticket | With a service's key, an attacker forges tickets for that service | Rotate service keys, validate the PAC |
| Pass-the-Ticket | Stolen ticket from memory is reused | Limit admin logins, protect endpoints |
| Pass-the-Hash | NTLM hash is used as the password | Reduce NTLM, restrict admin rights |

gMSA means group Managed Service Account: AD sets and rotates a long random password by itself.

`krbtgt` is reset twice because AD keeps the current and previous keys, and a ticket made with either still works.

### RADIUS security and evolution

**BlastRADIUS (CVE-2024-3596, disclosed July 2024).** An attacker in the path between NAS and server can forge an Access-Accept for some login types by using an MD5 collision. It is a flaw in the protocol design, not one product.

The main fix is to require the `Message-Authenticator` attribute on Access packets, and to update both sides. EAP logins already need that attribute.

**RadSec.** This carries RADIUS inside TLS (RFC 6614, TCP 2083), or DTLS (RFC 7360). It protects the whole packet and is safer across untrusted links. eduroam, the worldwide campus Wi-Fi service, uses RADIUS proxies between institutions, with roaming based on the realm part of `alice@uni.example`.

**CoA (Change of Authorization, RFC 5176, UDP 3799).** Classic RADIUS only answers when asked. CoA lets the server contact the NAS first, for example to disconnect a user or move them to a quarantine VLAN.

**Diameter (RFC 6733, port 3868).** A later AAA protocol used mainly inside mobile networks. It did not replace RADIUS in enterprises. The name is a play on words: a diameter is twice a radius.

**Retries.** RADIUS runs on UDP, so the NAS handles retries. Timers are vendor settings. On Cisco IOS the usual defaults are a 5-second timeout and 3 retransmits. Slow back-end checks can cause duplicate requests, and that is why TCP vs UDP is an interview point.

### Token and federation security

**Bearer tokens.** Anyone holding a bearer token can use it. This is why they need HTTPS, short lifetimes and careful storage. They should never be in logs or URLs. RFC 6750 defines bearer use.

**JWT checks.** An API that accepts a JWT must verify:

- The signature, using an algorithm it expects. Never accept `alg: none`.
- `iss`: the issuer it trusts.
- `aud`: that the token was meant for this API.
- `exp`: that it has not expired.

**SAML risks.** XML signature wrapping tricks a weak parser into reading an unsigned part of the message. A stolen IdP signing key lets an attacker forge assertions for any user, which is the "Golden SAML" attack. Use maintained libraries, and protect signing keys.

**OAuth vs OIDC vs SAML:**

| | OAuth 2.0 | OIDC | SAML 2.0 |
|---|---|---|---|
| Purpose | Delegated access | Login | Login (SSO) |
| Token format | Not fixed, often JWT | ID token is JWT | XML assertion |
| Best for | APIs | Web and mobile apps | Enterprise web apps |
| Year of current spec | 2012 | 2014 | 2005 |

**Provisioning.** SSO handles login, but who creates and removes accounts in each app? SCIM (System for Cross-domain Identity Management, RFC 7643 and RFC 7644) is a REST standard for that.

**Passwordless.** Passkeys, built on FIDO2 and WebAuthn, use a key pair stored on a device instead of a password. They resist phishing because the signature is bound to the website's name.

**Zero trust** builds on all of this: identity is checked on each access, not only at the network edge. See [Zero Trust and Segmentation](../09-security/09-zero-trust-and-segmentation.md).

## Common Mistakes

- **"LDAP and Active Directory are the same."** LDAP is a protocol. AD is a product that uses LDAP, Kerberos, DNS and SMB.
- **"Kerberos encrypts my file transfer."** Kerberos handles login and creates keys. The application decides whether to use those keys for its data.
- **"The server calls the KDC to check each ticket."** It does not. It decrypts the ticket with its own key.
- **"The password travels inside the AS-REQ."** It never does. A timestamp encrypted with a password-based key travels instead.
- **"A 2-hour time difference between offices breaks Kerberos."** Time zones are fine. Only the real UTC gap counts.
- **"The PREAUTH_REQUIRED error in my capture is the fault."** It is a normal first round.
- **"The laptop is the RADIUS client."** The NAS is the client. The laptop only talks to the NAS.
- **"RADIUS encrypts everything."** Only the password attribute is hidden. Usernames and other attributes are visible.
- **"TACACS+ uses strong encryption."** It uses an MD5-based pad. Keep it on a trusted management network.
- **"`local` fallback runs when TACACS+ says no."** It runs only on errors or timeouts.
- **"OAuth is a login protocol."** It is for authorization. Use OIDC for login.
- **"SAML and OIDC need the app to reach the IdP during login."** In the SAML POST flow, the browser carries the messages.
- **"Port 636 will work once AD is installed."** A certificate must be installed first.
- **"Accepting any cert is fine, it's internal."** Skipping certificate checks in LDAPS allows a fake server to collect passwords.
- **Using DNS outside AD on domain PCs.** SRV lookups then fail, and logins break in strange ways.

## Interview Questions

**Q: What is the difference between authentication, authorization and accounting?**
A: Authentication proves who you are, such as with a password. Authorization decides what you may do, such as read-only access. Accounting records what you did, such as login times and commands. They happen in that order, and accounting continues through the session.

**Q: What is the difference between LDAP and Active Directory?**
A: LDAP is an open protocol for reading and changing a directory. Active Directory is Microsoft's directory product. AD speaks LDAP on port 389, but it also uses Kerberos for login, DNS to find domain controllers, and SMB to share policy files.

**Q: LDAPS or StartTLS: what is the difference?**
A: LDAPS uses port 636 and starts TLS before any LDAP traffic, like HTTPS. StartTLS uses port 389, starts in plain text, and then upgrades with an extended request. Both need a valid certificate. With StartTLS, the client must require the upgrade, or an attacker can strip it.

**Q: Walk me through Kerberos.**
A: The client sends an AS-REQ with an encrypted timestamp. The AS checks it and returns a TGT, encrypted with the krbtgt key, plus a session key encrypted with the user's key. To use a service, the client sends the TGT, an authenticator and the SPN to the TGS. It gets a service ticket encrypted with the service's key, plus a service session key. The client sends that ticket and a new authenticator to the server in an AP-REQ. The server decrypts it with its own key, without calling the KDC. It can reply with an AP-REP for mutual authentication.

**Q: Why does Kerberos need synchronised clocks?**
A: Authenticators hold a timestamp, and servers accept them only inside a time window, normally 5 minutes. Together with a replay cache, this stops an attacker from re-sending a captured authenticator. If clocks differ by more than the limit, you get KRB_AP_ERR_SKEW and logins fail. In AD, the PDC Emulator is the top time source for a domain.

**Q: A user can open a share by name but not by IP. Why might that be?**
A: Kerberos asks for a ticket for an SPN, which is normally based on a name, such as cifs/fs01.example.com. Using the IP usually means no SPN matches, so Windows falls back to NTLM. If NTLM is blocked, the login fails. I would confirm with klist and check the NTLM policy.

**Q: Compare RADIUS and TACACS+.**
A: RADIUS uses UDP 1812 and 1813, hides only the password, and joins authentication with authorization. It is mainly for network access such as Wi-Fi, VPN and 802.1X. TACACS+ uses TCP 49, protects the full packet body, and separates all three As. It can authorize each command, so it fits device administration.

**Q: In RADIUS, who is the client, and what is a shared secret?**
A: The client is the network device, the NAS, such as a switch or VPN gateway. The user talks to the NAS, not the server. The shared secret is a password known to both NAS and server. It hides the User-Password attribute and verifies replies. It does not encrypt the whole packet.

**Q: What is the difference between OAuth, OIDC and SAML?**
A: OAuth 2.0 is for delegated access: it gives an app a scoped token to call an API. OIDC adds login on top of OAuth with a signed ID token. SAML is an older XML-based standard for web single sign-on, where a signed assertion from an identity provider reaches the app through the browser.

**Q: What is PKCE and why is it used?**
A: PKCE protects the authorization code flow. The app makes a random secret and sends only its hash in the first request. When it swaps the code for a token, it sends the original secret. If someone steals the code, they cannot use it without that secret.

**Q: What is Kerberoasting and how do you defend against it?**
A: Any domain user can request a service ticket for an account with an SPN. The ticket is encrypted with that account's key, so an attacker can take it offline and guess the password. Defences are long random passwords, group Managed Service Accounts, and AES instead of RC4.

**Q: You changed a user's group, but access did not change. Why?**
A: In AD, group membership is stored in the PAC inside the ticket. The user still holds old tickets. Logging off and on, or running klist purge and reconnecting, gets new tickets with the new groups.

## Quick Recap

- AAA means authentication (who), authorization (what), accounting (what happened). It is a model, and protocols carry it out.
- LDAP is the protocol for reading a tree-shaped directory, using TCP 389 or LDAPS on TCP 636. Active Directory uses LDAP, Kerberos, DNS and SMB together.
- Kerberos uses port 88: the AS gives a TGT, the TGS gives service tickets, and the service checks tickets with its own key. The password never crosses the network.
- Clock skew over about 5 minutes breaks Kerberos, because timestamps stop replay attacks. Check DNS, time and SPNs first.
- RADIUS (UDP 1812, 1813) is for users reaching the network. TACACS+ (TCP 49) is for admins managing devices, with per-command control.
- OAuth 2.0 gives apps limited access to APIs, OIDC adds login, and SAML gives XML-based web SSO through the browser.
