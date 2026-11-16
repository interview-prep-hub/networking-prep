# Email Protocols

**Level:** Intermediate | **Prerequisites:** [DNS Basics](01-dns-basics.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md)

## In One Line

Email is sent with SMTP from server to server, found with DNS MX records, read with IMAP or POP3, and protected from forgery with SPF, DKIM and DMARC.

## Why It Matters

Email is one of the oldest internet systems, and it still carries password resets, invoices and alerts. When it breaks, a business feels it within minutes.

SMTP was designed in a time when the people on the network trusted each other. It had no login, no encryption and no proof of who the sender was. Almost everything modern in email is a fix added later for one of those three gaps.

Interviewers like this topic because one question touches many layers:

- DNS (MX, TXT and PTR records)
- TCP and ports (25, 465, 587, 993, 995)
- TLS (encryption on each hop)
- Security (spoofing, spam, phishing)
- Cloud practice (why a new VM cannot send mail)

A very common real ticket is "our emails go to spam" or "our server cannot send mail". You should be able to work through both.

## Simple Explanation

Think of email as the postal service.

1. You write a letter. Your email app is your desk and pen.
2. You drop it at your local post office. That is your outgoing mail server.
3. Post offices pass it along until it reaches the town of the person you are writing to.
4. A local worker puts it in that person's private mailbox.
5. The person opens the mailbox and reads the letter.

Two more ideas fit this picture well.

**Envelope and letter.** The post office reads only the address on the envelope. The letter inside can say anything, even a false sender name. Email works the same way, and this is the root of email spoofing (faking the sender).

**Keep it or take it home.** IMAP is like leaving your letters at the post office and reading them there from any town. POP3 is like taking them home, so they are only in one place.

Where the analogy stops being true:

- A post office moves one physical letter. Mail servers make copies, and each server can keep one.
- An email usually takes only two or three hops, not a long chain of offices.
- The servers can read the message unless it has end-to-end encryption. The "sealed envelope" idea is much weaker than it sounds.

## How It Works

### The Three Main Roles

| Name | Full name | Plain meaning | Real examples |
|---|---|---|---|
| MUA | Mail User Agent | The app a person uses to write and read mail | Outlook, Thunderbird, Apple Mail, Gmail app |
| MTA | Mail Transfer Agent | A server that moves mail to other servers | Postfix, Exim, Sendmail, Microsoft Exchange |
| MDA | Mail Delivery Agent | The software that puts mail into the user's mailbox | Dovecot, Procmail |

**Fuller truth:** there is a fourth role, the MSA (Mail Submission Agent). It is the part of a server that accepts new mail from logged-in users on port 587 or 465. In most small setups, the same Postfix program acts as both MSA and MTA, so people often just say MTA.

**Webmail note:** if you use Gmail in a browser, your browser speaks HTTPS to the web server. The browser does not speak SMTP or IMAP at all. Webmail is the MUA, running on the provider's side. SMTP starts after that, between servers.

### The Full Path of One Email

Here Alice (`alice@example.com`) sends to Bob (`bob@example.net`). Both are reserved names used for examples.

```text
 Alice's laptop                                    Bob's phone
 +---------+                                       +---------+
 |   MUA   |                                       |   MUA   |
 +---------+                                       +---------+
      |                                                 ^
      | SMTP submission                                 | IMAP, 993
      | port 587 or 465                                 | or POP3, 995
      | login required                                  | login required
      v                                                 |
 +---------+   SMTP, port 25    +---------+       +-----------+
 | MSA/MTA | -----------------> |   MTA   | ----> |    MDA    |
 +---------+   no user login    +---------+       | + mailbox |
 smtp.example.com               mx1.example.net   +-----------+
 203.0.113.10                   198.51.100.25
```

1. Alice clicks Send. Her MUA connects to `smtp.example.com` on port 587, turns on TLS (encryption), and logs in.
2. The server checks her login, accepts the message and adds a DKIM signature (explained below).
3. The server looks at the part after the `@` in the recipient address: `example.net`. It asks DNS for the MX records of that domain.
4. DNS answers with `mx1.example.net`, and then the IP address `198.51.100.25`.
5. The sending MTA opens a TCP connection to port 25 on that IP and passes the message with SMTP.
6. The receiving MTA checks SPF, DKIM, DMARC and spam rules.
7. The MTA hands the message to the MDA. The MDA runs any filter rules and saves it in Bob's mailbox.
8. Bob's phone connects with IMAP on port 993 and shows the message.

Remember this split: **SMTP pushes mail. IMAP and POP3 pull mail.** SMTP never reads your inbox.

**Fuller truth:** large providers have many more internal hops, such as spam filters and storage clusters. A company may also put a filtering service in front of its mail servers. The basic roles stay the same.

### SMTP: The Sending Protocol

SMTP means Simple Mail Transfer Protocol. The current standard is RFC 5321. It runs over TCP, and it is a plain text protocol, so a person can read every line.

The modern form is ESMTP (Extended SMTP). A client starts with `EHLO`, and the server replies with a list of extra features it supports. The old greeting, `HELO`, gets no such list.

Here is a real style conversation between two MTAs. `S:` is the server and `C:` is the client. The prefixes are added for reading and are not sent on the wire.

```text
S: 220 mx1.example.net ESMTP Postfix
C: EHLO smtp.example.com
S: 250-mx1.example.net
S: 250-PIPELINING
S: 250-SIZE 52428800
S: 250-STARTTLS
S: 250-ENHANCEDSTATUSCODES
S: 250-8BITMIME
S: 250 SMTPUTF8
C: STARTTLS
S: 220 2.0.0 Ready to start TLS
   (TLS handshake happens here, everything after is encrypted)
C: EHLO smtp.example.com
S: 250-mx1.example.net
S: 250 (same list, without STARTTLS)
C: MAIL FROM:<alice@example.com>
S: 250 2.1.0 Ok
C: RCPT TO:<bob@example.net>
S: 250 2.1.5 Ok
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: Alice <alice@example.com>
C: To: Bob <bob@example.net>
C: Subject: Lunch on Friday?
C: Date: Tue, 13 Oct 2026 09:15:02 +0000
C: Message-ID: <a1b2c3@example.com>
C:
C: Are you free at noon?
C: .
S: 250 2.0.0 Ok: queued as 4FkT2b1xQKz9
C: QUIT
S: 221 2.0.0 Bye
```

What each step means:

| Command | Meaning |
|---|---|
| `EHLO` | "Hello, I am this host. What can you do?" |
| `STARTTLS` | "Let us switch this same connection to TLS." |
| `AUTH` | Log in. Used on submission ports, not normally between MTAs. |
| `MAIL FROM` | The envelope sender. Bounce messages go here. |
| `RCPT TO` | The envelope recipient. Repeat once per recipient. |
| `DATA` | "The message comes next." It ends with a line holding one dot. |
| `QUIT` | Close the session. |

Note that the server speaks first with a `220` greeting line, called the banner. The `250 ... queued` line is the key moment: the receiver now takes responsibility for the message, and the sender can delete its copy.

On a submission port, the client also logs in after TLS:

```text
C: AUTH PLAIN AGFsaWNlAHNlY3JldA==
S: 235 2.7.0 Authentication successful
```

That string is Base64 (a text encoding, not encryption) of the username and password. This is why `AUTH` must only happen inside TLS. Decoding it takes one second.

### SMTP Reply Codes

The first digit tells you what to do next.

| Code | Meaning | Sender should |
|---|---|---|
| 2xx | Success | Carry on |
| 3xx | Send more data | Send the message body |
| 4xx | Temporary failure | Keep the message and retry later |
| 5xx | Permanent failure | Stop and send a bounce |

Common ones you will see in logs:

| Code | Typical meaning |
|---|---|
| `421 4.7.0` | Try later, often rate limiting |
| `451 4.7.1` | Temporary reject, often greylisting |
| `452 4.2.2` | Mailbox full (some systems use a 5xx code) |
| `530 5.7.0` | Login required |
| `535 5.7.8` | Login failed |
| `550 5.1.1` | No such user |
| `554 5.7.1` | Rejected, often for policy or spam |

The second number, like `5.1.1`, is an enhanced status code. It gives more detail than the first three digits.

### Envelope vs Header

This is the single most useful idea for understanding spoofing.

| | Envelope | Header |
|---|---|---|
| Set by | `MAIL FROM` and `RCPT TO` commands | Lines inside `DATA` |
| Used by | Mail servers for routing and bounces | Mail apps, for display |
| Seen by the user | Usually not | Yes |
| Standard | SMTP, RFC 5321 | Message format, RFC 5322 |

The two do not need to match, and SMTP itself does not check them. A sender can write `From: ceo@yourbank.example` in the header and use any other address in the envelope.

This is also how Bcc (blind carbon copy) works. A Bcc person gets a `RCPT TO` command, but their address is not in the header lines. Other recipients cannot see them.

### Ports 25, 465 and 587

| Port | Service name | Who uses it | Encryption | Login |
|---|---|---|---|---|
| 25 | SMTP | Server to server | Optional, with STARTTLS | No |
| 587 | Submission | App to its own mail server | STARTTLS | Required |
| 465 | Submissions | App to its own mail server | Implicit TLS | Required |
| 2525 | Not a standard | Relay services, as a backup | Varies | Required |

**STARTTLS** starts in plain text, then upgrades the same connection to TLS. **Implicit TLS** means TLS starts at once, before any SMTP line is sent. See [TLS and HTTPS](07-tls-and-https.md) for the handshake.

**Why are there two submission ports?** Port 465 was given to SMTP over TLS in the 1990s, and the assignment was then withdrawn in favour of STARTTLS on 587. People kept using 465 anyway. In 2018, RFC 8314 made it official again and said implicit TLS should be preferred. So today both are valid. Use whichever your provider documents.

**Why does port 25 still exist?** Because the MX system has no place to name a port. Every MTA in the world knows to connect to port 25. It cannot be changed without changing all of them.

### MX Lookup

MX means Mail Exchanger. An MX record tells the world which server receives mail for a domain.

```text
example.net.   3600  IN  MX  10 mx1.example.net.
example.net.   3600  IN  MX  20 mx2.example.net.
```

The number is the preference. **The lowest number is tried first.** Here, `mx1` is the main server and `mx2` is the backup. If two records have the same number, senders share the load between them.

The lookup steps:

1. Take the domain after the `@`.
2. Ask DNS for its MX records.
3. Sort them by preference.
4. Find the A or AAAA record (the IP address) of the best name.
5. Connect to port 25. If that fails, try the next MX.

Three rules to remember:

- An MX record must point to a name, not an IP address. The name must not be a CNAME (an alias).
- If a domain has no MX record, senders fall back to its A or AAAA record. This is called the implicit MX.
- A domain that never receives mail can say so with a "null MX": `example.org. MX 0 .` (RFC 7505).

See [DNS Record Types](02-dns-records.md) for record details.

### POP3 vs IMAP

Both let a user read mail. They think about the mailbox in opposite ways.

| | POP3 | IMAP |
|---|---|---|
| Full name | Post Office Protocol v3 | Internet Message Access Protocol |
| Standard | RFC 1939 | RFC 9051 (IMAP4rev2), RFC 3501 (IMAP4rev1) |
| Main idea | Download the mail | Work on the mail where it sits |
| Mail stays on server | Usually deleted, can be kept | Yes |
| Folders on server | No, inbox only | Yes |
| Many devices stay in sync | No | Yes, read and flag states sync |
| Fetch part of a message | Whole message, in practice | Yes, such as headers only |
| Push of new mail | No, the client polls | Yes, with the `IDLE` command |
| Plain port | 110 | 143 |
| TLS port | 995 | 993 |
| Server storage used | Low | High |

Use IMAP in almost every modern case. POP3 fits only one device, poor connections, or a need to keep mail off the server.

A short POP3 session, over TLS on port 995. A POP3 server replies with `+OK` or `-ERR`.

```text
S: +OK Dovecot ready.
C: USER bob
S: +OK
C: PASS ********
S: +OK Logged in.
C: STAT
S: +OK 2 4831
C: LIST
S: +OK 2 messages:
S: 1 2210
S: 2 2621
S: .
C: RETR 2
S: +OK 2621 octets
S:    (the whole message)
S: .
C: DELE 2
S: +OK Marked to be deleted.
C: QUIT
S: +OK Logging out, messages deleted.
```

`STAT` shows 2 messages and 4831 bytes in total. `DELE` only marks the message. The delete really happens at `QUIT`.

A short IMAP session, over TLS on port 993. Each command starts with a tag (like `a1`), so replies can be matched to commands. This lets a client run several commands at the same time.

```text
S: * OK [CAPABILITY IMAP4rev1 IDLE] Dovecot ready.
C: a1 LOGIN bob ********
S: a1 OK Logged in
C: a2 SELECT INBOX
S: * 2 EXISTS
S: a2 OK [READ-WRITE] Select completed
C: a3 FETCH 2 (FLAGS BODY.PEEK[HEADER.FIELDS (FROM SUBJECT)])
S: * 2 FETCH (FLAGS () BODY[HEADER.FIELDS (FROM SUBJECT)] {62}
S: From: Alice <alice@example.com>
S: Subject: Lunch on Friday?
S:
S: )
S: a3 OK Fetch completed
C: a4 IDLE
S: + idling
```

`IDLE` leaves the connection open, and the server tells the client when new mail arrives. This is what makes mail feel instant on a phone.

### SPF, DKIM and DMARC: Stopping Forged Mail

SMTP does not check who the sender is. These three DNS-based systems add that check. They work together, and each answers a different question.

**SPF: "Is this server allowed to send for this domain?"**

SPF means Sender Policy Framework (RFC 7208). The domain owner publishes a TXT record listing the servers allowed to send.

```text
example.com.  3600  IN  TXT  "v=spf1 ip4:203.0.113.10 -all"
```

- `v=spf1`: this is an SPF record.
- `ip4:203.0.113.10`: this IP may send.
- `-all`: everything else fails.

Other common parts are `include:_spf.google.com` (also trust Google's list), `mx` (trust the domain's MX hosts), and `a` (trust the domain's A record).

The ending sets how strict the record is:

| Ending | Name | Meaning |
|---|---|---|
| `-all` | Fail | Not allowed |
| `~all` | Softfail | Probably not allowed, accept but mark |
| `?all` | Neutral | No opinion |
| `+all` | Pass | Anyone may send. Never use this. |

The check:

1. The receiver sees the connecting IP, here `203.0.113.10`.
2. It takes the domain from `MAIL FROM`, here `example.com`.
3. It reads the SPF record of that domain.
4. If the IP is listed, SPF passes.

Analogy: a company gives the guard a list of its approved delivery drivers. Where it stops being true: the guard checks only the driver, not the content of the parcel. Also, SPF looks at the envelope, not the From line a person sees.

**Fuller truth:** SPF allows at most 10 DNS lookups per check. Each `include`, `a`, `mx`, `ptr`, `exists` and `redirect` counts. Going over gives a `permerror` (permanent error), and many long `include` chains hit this limit. A domain must also have only one SPF record. Two records also give a `permerror`. Receivers may check the `HELO` name too, which matters for bounces, as they have an empty `MAIL FROM`.

**DKIM: "Was this message really signed by this domain, and is it unchanged?"**

DKIM means DomainKeys Identified Mail (RFC 6376). The sending server signs the message with a private key. The public key is published in DNS.

The sender adds a header like this (values are shortened):

```text
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
  d=example.com; s=mail2026;
  h=from:to:subject:date:message-id;
  bh=2jUSOH9N...zv8=;
  b=dGhpcyBpcyBzaG9ydGVuZWQ...
```

| Tag | Meaning |
|---|---|
| `d=` | Domain that signed |
| `s=` | Selector, the name that finds the key in DNS |
| `h=` | Header fields included in the signature |
| `bh=` | Hash of the message body |
| `b=` | The signature itself |

The receiver joins the selector and domain to find the key:

```text
mail2026._domainkey.example.com.  TXT
  "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOC..."
```

The check:

1. Read `d=` and `s=` from the header.
2. Fetch the public key from DNS.
3. Hash the body and compare it to `bh=`.
4. Verify the signature in `b=` with the public key.

If anyone changes a signed part, the check fails. See [Cryptography Basics](../09-security/02-cryptography-basics.md) for how signatures work.

Analogy: a wax seal on a letter, made with the company's stamp. Where it stops being true: the seal proves the domain signed it, not that a certain person wrote it. DKIM also does not hide the message. It signs. It does not encrypt.

**Fuller truth:** selectors let a domain have many keys, one per service, and change keys over time. Use RSA keys of at least 2048 bits. RFC 8301 says verifiers must not accept keys under 1024 bits, and rejects SHA-1 for signing.

**DMARC: "What should you do if the checks fail, and does the From line match?"**

DMARC means Domain-based Message Authentication, Reporting and Conformance (RFC 7489). It fixes a big hole. SPF and DKIM can both pass for some other domain while the visible From line says `yourbank.example`.

DMARC adds **alignment**: the domain that passed SPF or DKIM must match the domain in the visible From header.

```text
              Message arrives
                    |
          +---------+---------+
          |                   |
     SPF check           DKIM check
     on MAIL FROM        on d= domain
          |                   |
     Pass, and domain    Pass, and domain
     aligns with From?   aligns with From?
          |                   |
          +---------+---------+
                    |
          At least one "yes"?
            |             |
           yes            no
            |             |
       DMARC pass     DMARC fail
                          |
                  Apply the p= policy
```

**Only one of the two has to pass with alignment.** This is why a message can survive when SPF breaks.

The record lives at `_dmarc` under the domain:

```text
_dmarc.example.com.  TXT
  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; adkim=r; aspf=r"
```

| Tag | Meaning |
|---|---|
| `p=` | Policy: `none` (only report), `quarantine` (spam folder), `reject` (refuse) |
| `rua=` | Where to send daily summary reports |
| `adkim=` | DKIM alignment: `r` relaxed or `s` strict |
| `aspf=` | SPF alignment: `r` relaxed or `s` strict |
| `sp=` | Policy for subdomains |
| `pct=` | Percent of failing mail the policy applies to |

Alignment examples, with a visible `From:` of `alice@example.com`:

| Domain that passed | Relaxed | Strict |
|---|---|---|
| `example.com` | Aligned | Aligned |
| `news.example.com` | Aligned, same main domain | Not aligned |
| `bounces.example.org` | Not aligned | Not aligned |

The safe way to roll it out:

1. Publish SPF and DKIM.
2. Publish DMARC with `p=none` and a `rua` address.
3. Read the reports for a few weeks. Find every real sender, like your CRM, billing tool or newsletter service.
4. Fix those senders.
5. Move to `p=quarantine`, then to `p=reject`.

Analogy: SPF is the driver list, DKIM is the seal, and DMARC is the note from the company to the guard: "Our name must match. If it does not, throw the parcel away, and send me a report." Where it stops being true: receivers may ignore the note. A DMARC policy is a request, and each mail provider makes the final call.

**Fuller truth:** RFC 7489 is an Informational document, not a formal standard. The IETF has been working on a revision, often called DMARCbis, which changes some tags. Check for the latest RFC before you quote fine details.

| | SPF | DKIM | DMARC |
|---|---|---|---|
| Checks | Sending server's IP | Signature on the message | Alignment with visible From |
| Looks at | Envelope `MAIL FROM` | `d=` domain in signature | Header From |
| DNS name | The domain | `selector._domainkey.domain` | `_dmarc.domain` |
| Survives plain forwarding | No | Usually yes | Yes, if DKIM survives |
| Sets a policy | Only a hint | No | Yes |

### Why Port 25 Is Blocked by ISPs and Clouds

Most home ISPs and most cloud providers block **outbound** connections to port 25.

The reason is spam. Port 25 needs no login, so any machine that can reach a mail server's port 25 can try to deliver mail directly. Without a block:

1. A home PC is infected and joins a botnet (a group of hacked machines controlled by one attacker).
2. It sends spam straight to mail servers, skipping the ISP's own server.
3. The ISP's IP addresses land on blocklists (shared lists of known bad senders).
4. Honest customers can no longer deliver mail.

A cloud has the same problem, made worse. Anyone can rent a VM for a few dollars, send spam for an hour and vanish, leaving a bad reputation on the IP for the next renter.

```text
 Home PC or cloud VM
        |
        |  to port 25 (any mail server)
        X  blocked by ISP or cloud
        |
        |  to port 587 or 465 (login required)
        v
 +--------------------+                 +-----------+
 | Provider or relay  |  port 25 ---->  | Target MX |
 | (smart host)       |                 +-----------+
 +--------------------+
```

A **smart host** (or relay) is a mail server you log in to, which sends the mail onward for you. Blocking 25 pushes people through 587 and 465, where a login exists, so abuse can be traced and stopped. The mail still travels on port 25 at the end, but from a server with a good reputation.

What real clouds do today (these policies change, so check the current docs):

- **AWS:** EC2 restricts port 25 by default. You can ask AWS to remove the limit.
- **Azure:** outbound port 25 is blocked for most subscription types. Enterprise Agreement customers are treated differently.
- **Google Cloud:** outbound connections to port 25 on external addresses are blocked by default.

The usual fix is not to fight the block. Use a mail service such as Amazon SES, SendGrid, Mailgun or Microsoft 365, over port 587 or 465.

Companies copy the same idea with an [ACL](../09-security/05-access-control-lists.md) on their edge router: only the real mail server may send to port 25, because any other PC doing so is likely infected.

**Fuller truth:** the block is usually on the destination port, so receiving mail may still work. Some home ISPs block inbound 25 too. Also, even with port 25 open, home IP ranges are often on policy blocklists such as the Spamhaus PBL, so big providers refuse them. Open port 25 alone is not enough. You need a good IP reputation and a matching PTR record.

## Example

### Finding Where Mail Goes

Linux (dig), looking up the MX records for `example.net`:

```bash
$ dig +short MX example.net
10 mx1.example.net.
20 mx2.example.net.

$ dig +short A mx1.example.net
198.51.100.25
```

Windows (PowerShell):

```powershell
PS> Resolve-DnsName example.net -Type MX

Name          Type  TTL   Section  NameExchange     Preference
----          ----  ---   -------  ------------     ----------
example.net   MX    3600  Answer   mx1.example.net  10
example.net   MX    3600  Answer   mx2.example.net  20
```

Windows also has `nslookup -type=mx example.net`. More on these in [DNS Tools](../12-troubleshooting-and-tools/04-dns-tools.md).

### Checking a Domain's Email Security

```bash
$ dig +short TXT example.com
"v=spf1 ip4:203.0.113.10 -all"

$ dig +short TXT mail2026._domainkey.example.com
"v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOC..."

$ dig +short TXT _dmarc.example.com
"v=DMARC1; p=reject; rua=mailto:dmarc@example.com; adkim=r; aspf=r"

$ dig +short -x 203.0.113.10
smtp.example.com.
```

The last line is a reverse lookup. The PTR record says `203.0.113.10` is `smtp.example.com`, which matches the server's name. Receivers want to see this.

### Reading the Headers Bob Receives

In most mail apps you can choose "Show original" or "View source" and see this:

```text
Return-Path: <alice@example.com>
Authentication-Results: mx1.example.net;
  spf=pass smtp.mailfrom=alice@example.com;
  dkim=pass header.d=example.com header.s=mail2026;
  dmarc=pass header.from=example.com
Received: from smtp.example.com (smtp.example.com [203.0.113.10])
  by mx1.example.net (Postfix) with ESMTPS id 4FkT2b1xQKz9
  for <bob@example.net>; Tue, 13 Oct 2026 09:15:04 +0000
Received: from [192.168.1.20] (unknown [203.0.113.77])
  by smtp.example.com (Postfix) with ESMTPSA id 7C2A1B0
  for <bob@example.net>; Tue, 13 Oct 2026 09:15:03 +0000
From: Alice <alice@example.com>
To: Bob <bob@example.net>
Subject: Lunch on Friday?
Date: Tue, 13 Oct 2026 09:15:02 +0000
Message-ID: <a1b2c3@example.com>
```

How to read it:

1. **Read `Received:` lines from the bottom up.** Each server adds its line at the top.
2. Bottom line: Alice's laptop has private IP `192.168.1.20`. The server sees her public IP `203.0.113.77`, because of [NAT](../04-ip-addressing/09-nat-and-pat.md). `ESMTPSA` means ESMTP with TLS and a login.
3. Next line: `smtp.example.com` passed it to `mx1.example.net` over TLS (`ESMTPS`, no login).
4. The times show one second per hop, so there was no delay.
5. `Authentication-Results` shows all three checks passed.

When mail is slow, compare the times. The gap shows which hop held the message.

### Testing a Mail Server by Hand

Linux or macOS. These commands open a TLS session so you can type SMTP or IMAP yourself:

```bash
# STARTTLS on the submission port
openssl s_client -starttls smtp -crlf -connect smtp.example.com:587

# Implicit TLS on port 465
openssl s_client -crlf -connect smtp.example.com:465

# IMAP over TLS
openssl s_client -crlf -connect imap.example.net:993
```

### Why Can't My Cloud VM Send Mail?

A new app on a VM (private IP `10.0.1.15`) tries to send directly, and every message hangs. First test the ports.

Linux:

```bash
$ nc -vz -w 5 mx1.example.net 25
nc: connect to mx1.example.net (198.51.100.25) port 25 (tcp) timed out: Operation now in progress

$ nc -vz -w 5 smtp.example.com 587
Connection to smtp.example.com (203.0.113.10) 587 port [tcp/submission] succeeded!
```

Windows:

```powershell
PS> Test-NetConnection mx1.example.net -Port 25
WARNING: TCP connect to (198.51.100.25 : 25) failed

ComputerName           : mx1.example.net
RemoteAddress          : 198.51.100.25
RemotePort             : 25
InterfaceAlias         : Ethernet
SourceAddress          : 10.0.1.15
PingSucceeded          : True
PingReplyDetails (RTT) : 24 ms
TcpTestSucceeded       : False
```

What this tells you:

- Port 25 **times out**. A timeout means packets are silently dropped, which points to a filter. A "connection refused" would mean the host answered but nothing was listening.
- Port 587 works, so the network is fine.
- **Fix:** send through a relay on 587 with a login, or ask the cloud to lift the block.

## Going Deeper

### Encryption Between Servers

TLS on port 25 is **opportunistic**. This means "use it if the other side offers it". Plain text is the fallback, so mail still gets through.

This leaves a hole. An attacker on the path can delete `STARTTLS` from the server's feature list. The sender then continues in plain text without any warning. This is a **STARTTLS downgrade attack**. Two systems fix it:

- **MTA-STS** (RFC 8461): the domain publishes a policy over HTTPS saying "always use TLS, and only to these MX names". A TXT record at `_mta-sts.example.net` points to it.
- **DANE for SMTP** (RFC 7672): the domain publishes its TLS key details in a TLSA record, at a name like `_25._tcp.mx1.example.net`. It needs DNSSEC. See [DNS Advanced](03-dns-advanced.md).

An MTA-STS policy, served from `https://mta-sts.example.net/.well-known/mta-sts.txt`:

```text
version: STSv1
mode: enforce
mx: mx1.example.net
mx: mx2.example.net
max_age: 86400
```

**TLS-RPT** (RFC 8460) lets senders report TLS failures back to the domain owner.

All of this protects each hop only. Every server on the path still sees the plain message. For end-to-end encryption, where only the recipient can read it, you need **S/MIME** (based on certificates) or **OpenPGP** (based on user keys). Both are rare outside certain companies and security-minded groups.

### Queues, Retries and Bounces

An MTA stores each message in a queue on disk until delivery works. This makes email tolerant of outages.

- On a 4xx reply or a failed connection, the MTA retries. RFC 5321 suggests waiting at least 30 minutes between tries and not giving up for at least 4-5 days. Postfix keeps a message for 5 days by default.
- On a 5xx reply, or after it gives up, the MTA sends a **bounce**, which is a failure report sent to the envelope sender. Bounces use an empty sender, `MAIL FROM:<>`, so a bounce can never cause another bounce.

**Greylisting** uses this retry rule to fight spam. A server rejects the first attempt from an unknown sender with a 4xx code. Real MTAs retry a little later. Many spam tools never do.

**Backscatter** is a side effect. If a server accepts spam with a forged sender and bounces it later, the bounce hits an innocent person. So good servers reject during the SMTP session, before they give the final `250`.

### Reputation and Open Relays

An **open relay** is a server that accepts mail from anyone and sends it to anyone. It is a gift to spammers and lands on blocklists quickly. Port 25 on your MX should accept mail only for your own domains. Sending onward to other domains must need a login.

Receivers judge senders with more than DMARC:

- **Reverse DNS:** the IP needs a PTR record, and that name should resolve back to the same IP. This match is called FCrDNS (forward-confirmed reverse DNS).
- **DNSBLs:** DNS-based blocklists such as Spamhaus, which receivers query for each sending IP.
- **Reputation:** past complaint and bounce rates. New IPs must build volume slowly, which is called IP warm-up.

Since February 2024, Google and Yahoo have required bulk senders (over about 5000 messages a day to Gmail) to have SPF, DKIM and DMARC, one-click unsubscribe and a low spam rate. Microsoft added similar rules for Outlook consumer addresses in 2025. So authentication is no longer optional for anyone sending at volume.

### Forwarding and Mailing Lists

Forwarding is where these checks run into trouble.

```text
 alice@example.com --> bob@example.net --> carol@example.org
   203.0.113.10         198.51.100.25      receiver checks here
```

Bob forwards all his mail to Carol. Carol's server sees the connection from `198.51.100.25` with `MAIL FROM:<alice@example.com>`. That IP is not in Alice's SPF record, so **SPF fails**. The DKIM signature travels with the message and still passes, so **DMARC passes through DKIM**.

Mailing lists are harder. They often add a tag to the subject or a footer to the body. This breaks DKIM too, so both fail. Common fixes:

- **SRS** (Sender Rewriting Scheme): the forwarder rewrites `MAIL FROM` to its own domain, so SPF passes.
- **ARC** (Authenticated Received Chain, RFC 8617): each hop records the results it saw and signs them, so the final server can trust a known forwarder.
- **From rewriting:** the list sends as its own address, with the sender's name kept.

### Message Format and MIME

SMTP was built for short lines of 7-bit ASCII text. A line can be at most 1000 bytes, including the line ending.

**MIME** (Multipurpose Internet Mail Extensions, RFC 2045) allows attachments, HTML and other languages. Files are turned into text with **Base64**, which makes 4 characters for every 3 bytes. So a 10 MB file becomes about 13.7 MB on the wire, counting line breaks. That is why a 25 MB limit does not mean a 25 MB attachment.

Also useful:

- **Dot stuffing:** a line of text that starts with a dot gets an extra dot, so it is not read as the end of the message. The receiver removes it.
- **SMTPUTF8** (RFC 6531) allows non-ASCII addresses.
- **LMTP** (Local Mail Transfer Protocol, RFC 2033) is an SMTP-like protocol often used to pass mail from the MTA to the MDA.

**SMTP smuggling**, published in December 2023, is a good advanced example. Two servers disagreed on what marks the end of `DATA` in odd line endings. An attacker could hide a second message inside the first, passing as a trusted sender. Postfix, Exim and Sendmail all released fixes. The lesson is that small parsing differences between servers can become security holes.

### Modern Login and Newer Protocols

Sending a password in `AUTH` or `LOGIN` is being phased out. Large providers now prefer **OAuth 2.0** tokens (short-lived permissions) instead. Microsoft has turned off basic password login for IMAP and POP in Exchange Online, and Google requires OAuth or app-specific passwords. Old printers and scripts often break for this reason.

**JMAP** (JSON Meta Application Protocol, RFC 8620 and RFC 8621) is a modern HTTP and JSON option to replace IMAP. It is used by some providers but is far from replacing IMAP.

## Common Mistakes

- **"SMTP is used to read mail."** SMTP only sends and relays. IMAP, POP3 or a web API reads.
- **"The MX with the biggest number is the main server."** The lowest number is tried first.
- **"An MX record can hold an IP address."** It must hold a hostname, and that name must have an A or AAAA record.
- **"Port 587 is always encrypted."** It starts in plain text. It is encrypted only after STARTTLS works, so the client must require it.
- **"Using 587 between mail servers."** MX servers listen on 25. Port 587 is for users and apps logging in to their own server.
- **"SPF checks the From address I see."** SPF checks the envelope `MAIL FROM`. DMARC is what links it to the visible From.
- **"DKIM encrypts the mail."** DKIM signs it. Anyone can still read the message.
- **"DMARC needs both SPF and DKIM to pass."** One aligned pass is enough.
- **"`p=none` protects my domain."** It only collects reports. Spoofed mail still gets through.
- **Publishing two SPF records.** This gives a `permerror`. Merge them into one record.
- **Jumping to `p=reject` on day one.** Your CRM or billing tool may be sending as your domain, and that real mail will vanish.
- **"TLS means my email is private."** TLS covers each hop only. Each server can read the message.
- **"IMAP is more secure than POP3."** Both are safe with TLS. The difference is where mail is stored.
- **"A blocked port 25 means the mail server is down."** Test port 25 from another network first. It may be your ISP or cloud blocking you.
- **Forgetting the PTR record.** A self-hosted server with correct SPF, DKIM and DMARC may still be rejected for having no reverse DNS.

## Interview Questions

**Q: Walk me through what happens when you send an email.**

A: My mail app connects to my provider's server on port 587 or 465, over TLS, and logs in. The server accepts the message and signs it with DKIM. It does an MX lookup on the recipient's domain, then connects to that server on port 25 and passes the mail using SMTP. The receiver checks SPF, DKIM and DMARC, then a delivery agent stores it in the mailbox. The recipient's app then fetches it with IMAP or POP3.

**Q: What is the difference between an MUA, an MTA and an MDA?**

A: The MUA is the user's mail app, like Outlook. The MTA moves mail between servers, like Postfix. The MDA puts the mail into the user's mailbox, like Dovecot. In short: write, transfer, deliver.

**Q: What is the difference between ports 25, 465 and 587?**

A: Port 25 is for server-to-server transfer, with no user login and optional STARTTLS. Port 587 is for users and apps to submit mail, with a required login and STARTTLS. Port 465 is also for submission, but TLS starts immediately. RFC 8314 made 465 official again, and it prefers implicit TLS.

**Q: POP3 or IMAP? How do they differ?**

A: POP3 downloads mail to one device and usually removes it from the server. IMAP keeps the mail on the server, with folders, and syncs state across devices. POP3 uses 110 or 995 with TLS. IMAP uses 143 or 993 with TLS. I would pick IMAP for nearly every modern user.

**Q: How does a mail server know where to send a message?**

A: It takes the domain after the `@` and asks DNS for MX records. It tries the lowest preference number first, resolves that name to an IP, and connects to port 25. If that fails, it tries the next one. With no MX record, it falls back to the domain's A or AAAA record.

**Q: Explain SPF, DKIM and DMARC.**

A: SPF is a DNS list of IPs allowed to send for a domain. It is checked against the envelope sender. DKIM is a cryptographic signature that proves the domain signed the message and it was not changed, using a public key in DNS. DMARC requires that one of them passes and aligns with the visible From domain. It also tells the receiver what to do on failure, and sends reports.

**Q: Why does SPF fail on forwarded mail when DKIM does not?**

A: SPF checks the IP of the connecting server. After forwarding, that IP belongs to the forwarder, which is not on the original domain's list. DKIM is carried in the message itself, so it stays valid if the signed content is unchanged. So DMARC can still pass through DKIM.

**Q: What is the difference between the envelope and the header?**

A: The envelope is the `MAIL FROM` and `RCPT TO` used by servers in the SMTP session. The header is the From, To and Subject lines inside the message, shown to the user. They can be different, which is how spoofing and Bcc both work.

**Q: Why do ISPs and cloud providers block port 25?**

A: Port 25 has no login, so infected machines and throwaway VMs can send spam directly to any mail server. That damages the provider's IP reputation. Blocking it forces mail through 587 or 465, where a login makes abuse traceable. The fix is to use a relay, or ask the provider to lift the block.

**Q: Our emails are going to spam. How do you troubleshoot?**

A: I would look at the raw headers of a test message and read `Authentication-Results`. Then I check that SPF, DKIM and DMARC pass and align, for every system that sends for us. Next, I check that the IP has a matching PTR record and is not on blocklists. After that I look at content, complaint rates and sudden volume changes.

**Q: Is email encrypted in transit? Is that enough?**

A: Usually yes, hop by hop, with STARTTLS. But on port 25 it is opportunistic, so an attacker could strip it. MTA-STS or DANE enforce it. Even then, each server sees the plain message. If only the recipient must read it, you need end-to-end encryption, like S/MIME or OpenPGP.

**Q: What do 4xx and 5xx mean in SMTP?**

A: A 4xx reply is a temporary failure, so the sender keeps the message and retries. A 5xx is permanent, so the sender stops and returns a bounce. For example, greylisting uses 451, and an unknown user gets 550.

## Quick Recap

- MUA writes, MTA transfers, MDA delivers. SMTP pushes mail. IMAP and POP3 pull it.
- Mail servers find each other with MX records, and the lowest preference number is tried first.
- Port 25 is server to server. Ports 587 (STARTTLS) and 465 (implicit TLS) are for logged-in users.
- IMAP keeps mail on the server and syncs devices. POP3 downloads it to one device.
- SPF checks the sending IP, DKIM checks a signature, and DMARC checks alignment with the visible From and sets the policy.
- Clouds and ISPs block outbound port 25 to stop spam, so use a relay on 587 or 465.
- TLS protects each hop only. End-to-end privacy needs S/MIME or OpenPGP.
