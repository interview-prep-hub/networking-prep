# File Transfer Protocols

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP vs UDP](../06-transport-layer/02-tcp-vs-udp.md), [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md), [TLS and HTTPS](07-tls-and-https.md)

## In One Line

File transfer protocols move files between computers: FTP is the old two-connection protocol, FTPS adds TLS to it, SFTP and SCP run inside SSH, TFTP is a tiny UDP protocol for booting devices, and rsync sends only the parts of a file that changed.

## Why It Matters

Files must move all the time. Backups go to a backup server. A bank sends payment files to a partner. A new switch needs a software image. A developer deploys code to a server.

Each job has different needs. A bank needs strong security. A switch that is still booting needs a very small program. A nightly backup of 500 GB needs to skip data that has not changed.

Interviewers like this topic for three reasons:

- FTP shows if you really understand TCP connections, NAT and stateful firewalls. "Login works but the file list hangs" is a classic question.
- FTPS and SFTP sound the same but are completely different. Many candidates mix them up.
- Choosing the right protocol for a job shows real-world judgement.

## Simple Explanation

Think of different ways to send a package.

- **FTP** is a shop with a phone line and a delivery van. You give orders on the phone. The goods move in the van. Anyone on the road can read your order and see the goods.
- **FTPS** is the same shop, but the phone line is scrambled and the van is armored.
- **SFTP** is a different shop. Orders and goods travel through one private armored tunnel.
- **SCP** is a simple courier in that same tunnel. It carries a package from A to B. It does not chat, and it has no menu.
- **TFTP** is a child passing notes. One small note at a time, and each note must be answered before the next one. No name check at all.
- **rsync** is a clever editor. Your friend already has a copy of your 500-page book. You only post the pages that changed.

Where the analogy stops being true: a real protocol is not one fixed object. FTPS can encrypt the phone but not the van. rsync can run through the SFTP tunnel's road (SSH) but it is still its own protocol, and it does not know about "pages". It finds matching data at any byte position.

## How It Works

### FTP: two connections

FTP means File Transfer Protocol. Its ideas date from 1971. The current standard is RFC 959 (1985). It runs on TCP.

FTP uses two separate connections:

- **Control connection:** carries commands and replies as plain text. The client connects to server port 21. It stays open for the whole session.
- **Data connection:** carries the file bytes and directory listings. A new one is normally opened for each file or listing, then closed.

Why a new data connection each time? In the usual mode (stream mode), closing the connection is how FTP says "the file has ended".

This design is called **out-of-band** control: commands travel separately from the data. Because of it, you can send ABOR (abort) while a transfer is running.

### FTP commands and replies

Commands are short text words. Replies start with a 3-digit code. You can read them with your own eyes in a packet capture.

| Command | Meaning |
|---|---|
| USER, PASS | Send user name and password |
| PWD, CWD | Show or change the remote directory |
| LIST, MLSD | Get a directory listing (over the data connection) |
| RETR | Download a file (retrieve) |
| STOR | Upload a file (store) |
| TYPE A, TYPE I | ASCII mode or binary mode |
| PORT, EPRT | Active mode: "connect to me here" |
| PASV, EPSV | Passive mode: "tell me where to connect" |
| REST | Restart a transfer from a byte offset (used to resume) |
| QUIT | End the session |

The first digit of a reply tells you the result:

| First digit | Meaning | Example |
|---|---|---|
| 1xx | Started, wait for more | 150 Opening data connection |
| 2xx | Success | 226 Transfer complete |
| 3xx | Need more from you | 331 Need password |
| 4xx | Temporary failure, try again | 425 Can't open data connection |
| 5xx | Permanent failure | 530 Not logged in |

**ASCII vs binary:** ASCII is a simple text encoding. In ASCII mode (TYPE A), FTP may change line endings between systems (for example, Linux LF to Windows CR LF). That is helpful for text files and it destroys images, archives and programs. Always use binary mode (TYPE I) unless you have a clear reason not to.

### Active mode

The difference between active and passive is one question: **who opens the data connection?**

In active mode, the server opens it, back to the client.

1. The client connects from a random high port to server port 21.
2. The client logs in.
3. The client picks a port and starts listening on it.
4. The client sends PORT with its IP address and that port.
5. The server opens a TCP connection, usually from its port 20, to the client's port.
6. File data flows. The data connection closes at the end of the file.

```text
Client 192.168.1.10                         Server 203.0.113.10

:49500  ---- TCP connect ------------------->  :21   control
:49500  ---- PORT 192,168,1,10,195,149 ----->  :21
:49500  ---- RETR q3.csv ------------------->  :21
:50069  <--- TCP connect ---------------------  :20   data (server
:50069  <=== file bytes ======================  :20   opens this)
:49500  <--- 226 Transfer complete ----------  :21
```

**Reading the PORT numbers:** the six numbers are the four parts of the IP, then two bytes of the port. Port = (first byte x 256) + second byte.

```text
PORT 192,168,1,10,195,149
     IP = 192.168.1.10
     Port = 195 x 256 + 149 = 49920 + 149 = 50069
```

### Passive mode

In passive mode, the client opens both connections. This is the normal choice today.

1. The client connects to server port 21 and logs in.
2. The client sends PASV.
3. The server starts listening on a high port and replies with its IP and that port.
4. The client opens a TCP connection to that port.
5. File data flows. The data connection closes at the end of the file.

```text
Client 192.168.1.10                         Server 203.0.113.10

:49500  ---- TCP connect ------------------->  :21     control
:49500  ---- PASV -------------------------->  :21
:49500  <--- 227 (203,0,113,10,195,80) ------  :21
:49501  ---- TCP connect ------------------->  :50000  data (client
:49501  <=== file bytes ======================  :50000  opens this)
:49500  <--- 226 Transfer complete ----------  :21
```

The port in that reply is 195 x 256 + 80 = 50000.

**Analogy:** in active mode, the shop's van comes to your house, and your gate guard turns it away because you never invited a van. In passive mode, the shop says "come to door 50000", and you walk out and collect it. Your own guard lets you leave, but the shop's guard must allow visitors at door 50000.

Where the analogy stops being true: a firewall does not remember the phone call unless it reads the FTP text, as the next section explains.

| | Active | Passive |
|---|---|---|
| Control connection | Client to server port 21 | Client to server port 21 |
| Who opens data | Server | Client |
| Data source port | Server 20 (usually) | Client high port |
| Data destination port | Client high port | Server high port |
| Command | PORT or EPRT | PASV or EPSV |
| Who has the firewall problem | Client side | Server side |

### Why FTP and firewalls fight

A **stateful firewall** remembers connections that were started from inside. It lets the return traffic in and it blocks new inbound connections. See [Firewalls](../09-security/04-firewalls.md).

**NAT** (Network Address Translation) lets many private addresses share one public address. A router rewrites addresses in the IP header.

FTP breaks both because **it writes IP addresses and ports inside the application data**. Most protocols do not do this.

**Problem 1: active mode, client behind a firewall.** The server's data connection is a new inbound connection. The client's firewall blocks it.

**Problem 2: active mode, client behind NAT.** The client sends `PORT 192,168,1,10,...`. The server cannot reach a private address.

```text
 Client              NAT router             Server
 192.168.1.10   198.51.100.7 (public)    203.0.113.10
      |                  |                    |
      |-- PORT 192,168,1,10,195,149 --------->|
      |                  |                    |
      |                  |     Server tries to connect to
      |                  |     192.168.1.10 and fails.
```

**Problem 3: passive mode, server firewall.** The client's data connection arrives on a random high port. The server's firewall blocks it unless that port is allowed.

**Problem 4: passive mode, server behind NAT.** The server has private address 10.0.10.5 and it replies `227 (10,0,10,5,...)`. The client cannot reach it.

### The two fixes

**Fix 1: a fixed passive port range.** Pick a small range on the server, such as 50000-50100. Tell the FTP server to use only that range. Open it on the firewall. Set the public IP that the server should advertise. Example 2 shows this.

**Fix 2: an FTP-aware firewall.** An **ALG** (Application Layer Gateway) is firewall or NAT code that reads inside a protocol. An FTP ALG does two jobs:

1. It reads PORT, PASV and their replies on the control connection, then opens a short pinhole for the coming data connection.
2. On a NAT device, it rewrites the IP and port inside the text.

```text
Before ALG:  PORT 192,168,1,10,195,149
After ALG:   PORT 198,51,100,7,238,77     (public IP, NAT port 61005)
```

Check: 238 x 256 + 77 = 61005.

Examples of FTP helpers:

Cisco ASA (FTP inspection is in the default policy):

```text
policy-map global_policy
 class inspection_default
  inspect ftp
```

Linux netfilter (kernel 4.7 and later do not attach helpers automatically, so you add a rule):

```bash
# Linux: load the helper and attach it to FTP control traffic
modprobe nf_conntrack_ftp
iptables -t raw -A PREROUTING -p tcp --dport 21 -j CT --helper ftp
```

**The catch:** an ALG can only read what is not encrypted. If FTPS encrypts the control connection, the ALG sees random bytes and cannot help. This is the main reason FTPS is painful, and it is a favourite interview point.

### FTPS: FTP with TLS

FTPS is normal FTP protected by **TLS** (Transport Layer Security), the same encryption used by HTTPS. It is still two connections. The standard is RFC 4217. It builds on the security commands in RFC 2228.

There are two forms:

| | Explicit FTPS | Implicit FTPS |
|---|---|---|
| Control port | 21 | 990 |
| How TLS starts | Client sends AUTH TLS after connecting | TLS starts at once, on connect |
| Standard | Yes, RFC 4217 | No, an older practice; ports 989 and 990 are registered with IANA |
| Used today | Preferred | Legacy systems |

An explicit FTPS session looks like this:

```text
C: (connects to TCP 21)
S: 220 Service ready
C: AUTH TLS
S: 234 Proceed with negotiation.
   ... TLS handshake on the control connection ...
C: USER alice               (now encrypted)
C: PASS ********            (now encrypted)
S: 230 Login successful.
C: PBSZ 0
S: 200 PBSZ set to 0.
C: PROT P                   (P = private, encrypt the data too)
S: 200 PROT now Private.
C: PASV
S: 227 Entering Passive Mode (203,0,113,10,195,80).
   ... client connects, TLS handshake on the data connection ...
```

The PROT command is key:

- **PROT P:** encrypt the data connection.
- **PROT C:** clear, no encryption on data.

Control and data are protected separately. A badly configured server can encrypt the password and still send the file in clear text. Set the server to require both.

FTPS uses **X.509 certificates**, the same digital ID cards as HTTPS. The server proves its identity with one. The client can also use one instead of, or with, a password.

### SFTP: file transfer inside SSH

SFTP means SSH File Transfer Protocol. **It is not FTP over anything.** It is a separate binary protocol that shares no code or commands with FTP.

**SSH** (Secure Shell) is an encrypted remote login protocol. See [Remote Access Protocols](09-remote-access-protocols.md).

How it works:

1. The client opens one TCP connection to port 22.
2. SSH encrypts the connection and checks the server's host key.
3. The user logs in with a password or an SSH key.
4. The client asks SSH to start the `sftp` subsystem.
5. Messages like OPEN, READ, WRITE, READDIR and CLOSE travel inside the encrypted channel.

```text
Client                                         Server

:49700 ======== one encrypted TCP connection ===> :22
       |  SSH encryption                     |
       |  +-- sftp messages -------------+   |
       |  |  OPEN, READ, WRITE, READDIR  |   |
       |  |  STAT, CLOSE and file bytes  |   |
       |  +------------------------------+   |
```

Because there is only one connection on one fixed port, there are no PORT commands and no IP addresses in the data. NAT and firewalls have nothing to break. You allow TCP 22 and you are done.

A surprising fact: SFTP was never finished as an RFC. It stayed as an IETF draft. OpenSSH, the most common implementation, uses version 3 of it.

**Analogy:** SFTP is one sealed pipe. FTPS is two sealed pipes, and you must tell every guard about the second one.

Where the analogy stops being true: inside the SFTP pipe, many file requests can be running at once, each with its own ID.

### SCP: simple secure copy

SCP means Secure Copy. It also runs over SSH on TCP 22. It came from the old BSD `rcp` command.

The original SCP protocol has no formal standard. It simply started `scp` on the remote side and streamed the file. It had no resume and no directory browsing. The remote shell expanded file names, and old clients trusted the file names that the server sent back. This caused security problems (for example CVE-2019-6111).

**Current practice:** the OpenSSH project called the old SCP protocol outdated and recommends SFTP or rsync. Since OpenSSH 9.0 (2022), the `scp` command uses the SFTP protocol underneath by default. The `scp -O` flag forces the old protocol, for old servers.

So today, `scp` is often just a simple command line in front of SFTP.

### TFTP: the tiny one

TFTP means Trivial File Transfer Protocol. The standard is RFC 1350. It runs on **UDP** port 69.

It has no login, no directory listing and no encryption. It can only read a file or write a file. This is on purpose. It is small enough to fit in boot code in a chip.

Where you see it:

- **PXE boot:** a computer with an empty disk gets an IP and a boot file name from [DHCP](04-dhcp.md) (options 66 and 67), then fetches the boot file with TFTP.
- **Network devices:** copying configs and software images, and recovery from a broken flash.
- **IP phones:** downloading their config files.

UDP has no delivery guarantee, so TFTP adds a simple one: **lock-step** (also called stop-and-wait). Send one block, wait for its ACK, send the next.

1. The client sends RRQ (read request) to server port 69.
2. The server picks a new random port. It sends block 1 from that port.
3. The client sends ACK 1 to that new port.
4. The server sends block 2, and so on.
5. A block smaller than the block size means the file is finished. The default block size is 512 bytes.
6. If a block or ACK is lost, a timeout causes it to be sent again.

```text
Client 10.0.50.11                        Server 10.0.50.20

:49152 ---- RRQ "boot.img" octet ------------> :69
:49152 <--- DATA block 1 (512 bytes) --------- :51003
:49152 ---- ACK 1 ---------------------------> :51003
:49152 <--- DATA block 2 (512 bytes) --------- :51003
:49152 ---- ACK 2 ---------------------------> :51003
:49152 <--- DATA block 3 (130 bytes, last) --- :51003
:49152 ---- ACK 3 ---------------------------> :51003
```

Notice that the server moved from port 69 to port 51003. A simple firewall that allowed only port 69 would drop the data. Firewalls need a TFTP helper (for example `inspect tftp` on a Cisco ASA, or `nf_conntrack_tftp` on Linux).

TFTP packet layouts:

```text
RRQ / WRQ (opcode 1 / 2)
+----------+-----------+------+-----------+------+
| Opcode   | Filename  | 0    | Mode      | 0    |
| 2 bytes  | text      | 1 B  | text      | 1 B  |
+----------+-----------+------+-----------+------+

DATA (opcode 3)
+----------+----------+------------------+
| Opcode   | Block #  | Data             |
| 2 bytes  | 2 bytes  | 0 to 512 bytes   |
+----------+----------+------------------+

ACK (opcode 4)
+----------+----------+
| Opcode   | Block #  |
| 2 bytes  | 2 bytes  |
+----------+----------+

ERROR (opcode 5)
+----------+------------+-----------+------+
| Opcode   | Error code | Message   | 0    |
| 2 bytes  | 2 bytes    | text      | 1 B  |
+----------+------------+-----------+------+
```

The mode is `octet` (binary) or `netascii` (text). A third old mode, `mail`, is obsolete.

Because TFTP has no security, keep it on a trusted management network. Do not expose it to the internet.

### rsync: send only the differences

rsync is a tool and a protocol for keeping files the same in two places. Andrew Tridgell and Paul Mackerras created it in the 1990s.

It has two ways to connect:

- **Over SSH (the common way):** TCP 22, encrypted, uses your SSH login. rsync must be installed on both ends.
- **rsync daemon:** a background service on TCP 873 that shares named folders called modules. It has its own login, and the file data is **not encrypted**. Use it only on trusted networks.

First, rsync decides which files need work. By default this is a quick check: if size and modified time are the same, it skips the file.

Then, for each changed file, it runs the **delta-transfer algorithm**:

1. The receiver splits its old copy into fixed-size blocks.
2. For each block, the receiver computes two checksums. A **checksum** is a short fingerprint of data. One is fast and weak, and it can be "rolled" along the file. One is strong.
3. The receiver sends this list to the sender.
4. The sender slides a window over its new file, one byte at a time, looking for a match on the fast checksum.
5. If the fast one matches, it confirms with the strong one.
6. The sender sends instructions: "copy your block 7", "here are 2,000 new bytes", "copy your block 8".
7. The receiver builds a new file from old blocks and new bytes, checks it, then replaces the old file.

```text
Old file at receiver:   [ A ][ B ][ C ][ D ]

New file at sender:     [ A ][ B ][ x x x ][ C ][ D ]
                                     ^
                                  inserted

Sent over the network:  copy A, copy B, "x x x", copy C, copy D
```

The rolling checksum is why an insert at the start does not ruin everything. Later blocks are found at their new byte positions.

**Analogy:** posting only the changed pages of a book.

Where the analogy stops being true: a page-based approach would fail if you add one line, since every later page shifts. rsync finds the same content even after it shifts by any number of bytes.

### Comparison table

| Protocol | Transport and ports | Connections | Encryption | Login | Resume | Typical use |
|---|---|---|---|---|---|---|
| FTP active | TCP 21 control, TCP 20 data from server | 2+ | None | Clear text | Yes, with REST | Legacy only |
| FTP passive | TCP 21 control, high port on server | 2+ | None | Clear text | Yes, with REST | Legacy, public downloads |
| FTPS explicit | TCP 21 plus data ports | 2+ | TLS | Password, certificate | Yes | Partner exchange, compliance |
| FTPS implicit | TCP 990 plus data ports (989 for active data) | 2+ | TLS | Password, certificate | Yes | Legacy systems |
| SFTP | TCP 22 | 1 | SSH | Password, SSH key | Yes | Default secure choice |
| SCP | TCP 22 | 1 | SSH | Password, SSH key | No | Quick one-off copy |
| TFTP | UDP 69, then random ports | No connection | None | None | No | PXE boot, device images |
| rsync over SSH | TCP 22 | 1 | SSH | Password, SSH key | Yes, with --partial | Backups, sync, deploys |
| rsync daemon | TCP 873 | 1 | None | Module password | Yes, with --partial | Trusted LAN mirrors |

For the full port list, see [Common Ports Reference](../06-transport-layer/10-common-ports-reference.md).

### How to choose

- Need a secure default for people or scripts? **SFTP.**
- Partner system or compliance demand requires certificates and TLS? **FTPS, explicit.**
- Quick single copy on the command line? **scp** (or SFTP).
- Repeat sync, backups, big files that change a little? **rsync over SSH.**
- Booting a device, or a device that supports nothing else? **TFTP on a trusted network.**
- Plain **FTP**? Only for public files with no login, or a legacy system you cannot change. Even then, prefer HTTPS.

## Example

### Example 1: read a real passive FTP session

The client is 192.168.1.10. The server is ftp.example.com at 203.0.113.10. This is curl on Linux. The output is trimmed.

```bash
curl -v --disable-epsv -u alice:secret123 \
  ftp://ftp.example.com/reports/q3.csv -o q3.csv
```

```text
* Connected to ftp.example.com (203.0.113.10) port 21
< 220 (vsFTPd 3.0.5)
> USER alice
< 331 Please specify the password.
> PASS secret123
< 230 Login successful.
> PWD
< 257 "/home/alice" is the current directory
> CWD reports
< 250 Directory successfully changed.
> PASV
< 227 Entering Passive Mode (203,0,113,10,195,80).
* Connecting to 203.0.113.10 port 50000
* Connected to ftp.example.com (203.0.113.10) port 50000
> TYPE I
< 200 Switching to Binary mode.
> SIZE q3.csv
< 213 184320
> RETR q3.csv
< 150 Opening BINARY mode data connection for q3.csv (184320 bytes).
< 226 Transfer complete.
> QUIT
< 221 Goodbye.
```

What this shows:

- The password travels as plain text. Anyone on the path can read it.
- 195 x 256 + 80 = 50000, and curl connects to that port.
- The file is 184,320 bytes, over the data connection, in binary mode.

Prove the clear text yourself on a test network:

```bash
# Linux: print packet contents for FTP control traffic
sudo tcpdump -l -i eth0 -A -n 'tcp port 21' | grep -E 'USER|PASS'
```

```text
USER alice
PASS secret123
```

See [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md) for more.

### Example 2: publish an FTPS server behind NAT

The server has private IP 10.0.10.5. The firewall forwards public IP 203.0.113.10 to it.

**Step 1.** Configure the passive range and public IP (vsftpd on Linux, `/etc/vsftpd.conf`):

```text
pasv_enable=YES
pasv_min_port=50000
pasv_max_port=50100
pasv_address=203.0.113.10
ssl_enable=YES
force_local_logins_ssl=YES
force_local_data_ssl=YES
rsa_cert_file=/etc/ssl/certs/ftp.example.com.pem
rsa_private_key_file=/etc/ssl/private/ftp.example.com.key
```

**Step 2.** Allow the ports on the host firewall (Linux):

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A INPUT -p tcp --dport 50000:50100 -j ACCEPT
```

**Step 3.** Forward TCP 21 and TCP 50000-50100 from 203.0.113.10 to 10.0.10.5 on the edge firewall.

**Step 4.** Test TLS and the ports:

```bash
# Linux: confirm that explicit TLS starts and show the certificate
openssl s_client -connect ftp.example.com:21 -starttls ftp
```

```powershell
# Windows: check that the control and one data port are reachable
Test-NetConnection ftp.example.com -Port 21
Test-NetConnection ftp.example.com -Port 50000
```

Note: a passive port is only listening during a transfer, so a closed result on 50000 may be normal. A timeout usually means a firewall is dropping it, while a refused connection means the packet got there. See [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

### Example 3: SFTP, SCP and rsync side by side

Linux, macOS, and Windows 10 and later all include the OpenSSH client.

SFTP session:

```bash
sftp deploy@files.example.com
```

```text
Connected to files.example.com.
sftp> cd /srv/reports
sftp> put q3-report.pdf
Uploading q3-report.pdf to /srv/reports/q3-report.pdf
q3-report.pdf                    100% 2048KB  11.4MB/s   00:00
sftp> ls
q3-report.pdf
sftp> bye
```

The same copy with scp. Port 2222 is shown because a custom port is common. Note the capital `-P`:

```bash
scp -P 2222 q3-report.pdf deploy@files.example.com:/srv/reports/
```

```powershell
# Windows PowerShell: same command, OpenSSH client
scp -P 2222 .\q3-report.pdf deploy@files.example.com:/srv/reports/
```

rsync pushes a 1 GiB database dump. Only about 10 MB has changed since the last run. Linux:

```bash
rsync -avz --partial --progress /var/backups/db.dump \
  deploy@10.0.20.5:/srv/backup/
```

```text
sending incremental file list
db.dump
  1,073,741,824 100%   78.77MB/s    0:00:13 (xfr#1, to-chk=0/1)

sent 11,204,118 bytes  received 69,905 bytes  805,287.36 bytes/sec
total size is 1,073,741,824  speedup is 95.24
```

The flags mean:

- `-a`: archive mode, keep permissions, times, owners and links, and recurse into folders.
- `-v`: verbose.
- `-z`: compress data in transit.
- `--partial`: keep a partly sent file, so a retry can continue.
- `--progress`: show progress.

Speedup = 1,073,741,824 / (11,204,118 + 69,905) = 95.24. rsync moved about 11 MB of network traffic to sync a 1 GiB file.

rsync is not built in on Windows. Use it in WSL (Windows Subsystem for Linux), or use `robocopy` for copies to SMB shares.

### Example 4: back up a router with TFTP

The router R1 and the TFTP server 10.0.50.20 are on a management network. This is Cisco IOS.

```text
R1# copy running-config tftp:
Address or name of remote host []? 10.0.50.20
Destination filename [r1-confg]? r1-backup.cfg
!!
1842 bytes copied in 0.312 secs (5904 bytes/sec)
```

Each `!` shows progress. Many networks now use SCP instead:

```text
R1# copy running-config scp://backup@10.0.50.20/r1-backup.cfg
```

Fetch a boot file from a host, with packets on the wire:

```bash
# Linux
curl -O tftp://10.0.50.20/pxe.0
```

```powershell
# Windows: the TFTP client is an optional feature
Enable-WindowsOptionalFeature -Online -FeatureName TFTP
tftp -i 10.0.50.20 GET pxe.0
```

```text
IP 10.0.50.11.49152 > 10.0.50.20.69: TFTP, length 14, RRQ "pxe.0" octet
IP 10.0.50.20.51003 > 10.0.50.11.49152: UDP, length 516
IP 10.0.50.11.49152 > 10.0.50.20.51003: UDP, length 4
```

The 516 is 512 bytes of data plus 4 bytes of TFTP header. The 4 is the ACK.

## Going Deeper

### EPSV and EPRT: FTP for IPv6

PORT and PASV can only carry a 4-byte IPv4 address. RFC 2428 added EPRT and EPSV.

```text
EPRT |2|2001:db8::10|50069|
229 Entering Extended Passive Mode (|||50000|)
```

The EPSV reply carries **only a port**. The client reuses the IP from the control connection. This also removes Problem 4 from the client's view, because no private IP can leak. Many modern clients also ignore the IP in a 227 reply for the same reason. The server's firewall must still allow the port.

### How an ALG changes TCP itself

When a NAT ALG rewrites `192,168,1,10` to `198,51,100,7`, the text can change length. That shifts all later TCP sequence numbers. The ALG must then fix the sequence and acknowledgement numbers for the rest of the connection. So an ALG is complex code that sits in the data path, and that is why it is both a source of bugs and a risk.

### FTP security problems

- **Clear text:** passwords and files can be read with a packet capture.
- **Bounce attack:** a client sends PORT with a third party's IP. The server then connects to that victim, and this can be used to scan or to hide the attacker's source. Modern servers refuse PORT to any IP other than the client's. RFC 2577 covers this.
- **Data connection theft:** an attacker connects to the passive port first. Servers should check that the data connection comes from the same IP as the control connection. vsftpd does this by default.
- **Brute force:** internet-facing login ports get constant password guessing.

### FTPS and the control connection

Three more details come up in real work:

- **Session reuse:** many servers require the data connection's TLS session to reuse the control connection's session. This proves that the same client opened both. vsftpd does this by default (`require_ssl_reuse=YES`). A client that does not support it can fail with a 522 error.
- **CCC (Clear Command Channel):** this command, from RFC 2228, turns off TLS on the control connection after login, so an ALG can read it. It weakens security and it is rarely a good idea.
- **Idle control connection:** during a 2-hour transfer, the control connection carries no traffic. A firewall with a short idle timer may drop it. The file arrives, but the 226 reply never does, and the client reports a failure. Use keepalives, or raise the timer.

### SFTP security and hardening

SFTP and shell share port 22, so a firewall cannot allow one and not the other. Limit file-only users on the server. OpenSSH on Linux, `/etc/ssh/sshd_config`:

```text
Match Group sftponly
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

**Chroot** means that a user sees one folder as the top of the whole disk. They cannot go above it. OpenSSH requires that this folder is owned by root and not writable by group or others, so give users a writable subfolder.

The client checks the server's **host key** on first connect and stores it in `~/.ssh/known_hosts`. In scripts, install known host keys ahead of time. Do not set `StrictHostKeyChecking no`, as that removes protection against a machine in the middle.

Throughput note: SFTP runs on top of SSH's own flow-control window, which sits on top of TCP's. On long, fast links, this can limit speed well below the link rate. Parallel streams or tools built for this can help. See [TCP Reliability and Flow Control](../06-transport-layer/05-tcp-reliability-and-flow-control.md).

### TFTP math and extensions

Lock-step is slow when delay is high, because you send only one block per round trip. Take a 100 MiB image (104,857,600 bytes) and a 20 ms round trip time (RTT):

```text
Default 512-byte blocks:
  104,857,600 / 512 = 204,800 blocks
  204,800 x 0.020 s = 4,096 s = about 68 minutes

1468-byte blocks:
  104,857,600 / 1468 = about 71,430 blocks
  71,430 x 0.020 s = about 1,429 s = about 24 minutes
```

Why 1468? An Ethernet MTU of 1500, minus 20 for the IPv4 header, 8 for UDP, and 4 for TFTP. This avoids fragmentation. See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md).

Extensions that clients and servers can negotiate:

| RFC | Option | What it adds |
|---|---|---|
| RFC 2347 | Option negotiation | The OACK packet (opcode 6) |
| RFC 2348 | blksize | Block size from 8 to 65464 bytes |
| RFC 2349 | timeout, tsize | Timeout value and file size |
| RFC 7440 | windowsize | Several blocks per ACK |

**The 32 MB limit:** the block number is 16 bits, so the highest is 65,535. At 512 bytes per block, that is 33,553,920 bytes, about 32 MiB. Many implementations let the block number wrap around to go further, but this is not in RFC 1350. So the 100 MiB image above needs wraparound or a bigger block size, and both ends must agree.

### Check that the file is correct

TCP and UDP checksums are weak, and they do not cover the file on disk. After a large or important transfer, compare a hash on both ends:

```bash
# Linux
sha256sum c9300-universalk9.bin
```

```powershell
# Windows
Get-FileHash .\c9300-universalk9.bin -Algorithm SHA256
```

### Related options beyond the classics

- **HTTPS:** today, most public downloads use HTTPS with no special ports. See [HTTP Basics](05-http-basics.md).
- **Object storage:** cloud stores such as Amazon S3 are accessed through an HTTPS API.
- **Managed file transfer (MFT):** products that add logs, retries, approvals and audit on top of SFTP or FTPS. AWS Transfer Family is one cloud example.
- **AS2:** a business-to-business protocol that signs and encrypts files over HTTP (RFC 4130).
- **SMB and NFS:** these mount remote files like a local disk, a different job from one-off copying. See [File Sharing and Storage Protocols](13-file-sharing-and-storage-protocols.md).

### Troubleshooting table

| Symptom | Likely cause | Fix |
|---|---|---|
| Login works, `ls` hangs, then 425 | Data connection blocked | Use passive, open the passive range |
| 227 reply shows a private IP | Server behind NAT, no public IP set | Set `pasv_address` or use EPSV |
| FTP works, FTPS hangs at `ls` | Firewall ALG cannot read encrypted text | Fixed passive range, opened on the firewall |
| Client fails, 522 on data connection | TLS session reuse required | Update client or relax the server setting |
| Image or zip file is corrupted | ASCII mode | Use binary (`TYPE I`) |
| Large transfer works, then error at end | Firewall dropped the idle control connection | Keepalives, longer idle timer |
| TFTP first packet arrives, then timeouts | Firewall drops replies from the new port | TFTP helper, or allow the traffic |
| `Received message too long` in SFTP | Shell startup script prints text | Print only in interactive shells |
| SFTP login is accepted, then the connection closes, with chroot in use | Chroot folder owner or mode is wrong | Root-owned, not group or world writable |

Also see [Common Network Problems and Fixes](../12-troubleshooting-and-tools/09-common-network-problems.md).

## Common Mistakes

- **"SFTP is FTP over SSL."** No. SFTP runs inside SSH on port 22 and is a separate protocol. FTP over TLS is FTPS.
- **"SFTP and FTPS are basically the same, so either firewall rule works."** SFTP needs TCP 22 only. FTPS needs the control port and a data port range.
- **"Port 20 is always used in FTP."** It is used only as the server's source port in active mode, and not by every server. Passive mode does not use it.
- **"Passive mode is more secure than active mode."** Neither encrypts anything. Passive only changes who opens the data connection.
- **"Passive means the client is passive."** The server is the passive one, because it waits. The client opens both connections.
- **Opening all high ports for passive FTP.** Set a small fixed range and open only that.
- **Thinking encryption is on because login was encrypted.** With FTPS, check for PROT P. Without it, the file may be in clear text.
- **Using `-p` for port with scp.** With scp and sftp, the port flag is capital `-P`. Lowercase `-p` in scp preserves times. With ssh, the port flag is lowercase `-p`.
- **"TFTP is unreliable because it uses UDP."** TFTP adds its own ACKs and retries. It is reliable but slow, and it is not secure.
- **"TFTP stays on port 69."** Only the first request goes there. The transfer moves to random ports chosen by both sides.
- **Forgetting the rsync trailing slash.** `rsync -a src/ dest/` copies what is inside src. `rsync -a src dest/` creates `dest/src`.
- **Running `rsync --delete` without a test.** It removes files at the destination that are missing at the source. An empty or wrong source can wipe a backup. Run with `-n` (dry run) first.
- **Exposing an rsync daemon to the internet.** Port 873 has no encryption, and open modules leak data. Also keep rsync updated: version 3.4.0 (January 2025) fixed serious security flaws.
- **Using the old Windows `ftp.exe` through a firewall.** It works only in active mode, so it often hangs. Use a client that supports passive mode.

## Interview Questions

**Q: Why does FTP use two connections?**

A: One connection, on port 21, carries commands and replies. A second carries the file. In the default stream mode, closing the data connection marks the end of the file, so the session must stay alive separately. It also lets me abort a transfer while it is running.

**Q: What is the difference between active and passive FTP?**

A: It is about who opens the data connection. In active mode, the client sends PORT, and the server connects back, usually from port 20, to a client high port. In passive mode, the client sends PASV, the server replies with an IP and high port, and the client connects to it. Passive is the default today because it works with client firewalls and NAT.

**Q: A user can log in to an FTP server, but `ls` hangs. What is wrong?**

A: The control connection works, and the data connection is blocked. `ls` needs a data connection. I would check the mode first. If it is active, the client's firewall is blocking the inbound connection, so I would switch to passive. If it is passive, I would check that the server's passive range is open, and that the 227 reply carries a reachable public IP, not a private one.

**Q: Which firewall rules do you need for an FTP server in passive mode?**

A: Allow inbound TCP 21 for control, and a fixed passive range such as 50000-50100 for data, plus return traffic. I would set the same range in the FTP server config. If the server is behind NAT, I would also set the public IP that it advertises.

**Q: Why does FTPS often fail through firewalls when plain FTP works?**

A: A firewall ALG reads the PORT and PASV text on the control connection to open pinholes. FTPS encrypts that text, so the ALG cannot see the port. The fix is a fixed passive range opened by hand on the firewall.

**Q: What is the difference between FTPS and SFTP?**

A: FTPS is FTP with TLS. It uses two connections, ports 21 or 990 plus data ports, and X.509 certificates. SFTP is a separate protocol inside SSH. It uses one connection on port 22 and usually SSH keys. For most new setups, I would pick SFTP because it is easier through firewalls. I would pick FTPS if a partner or policy requires certificates.

**Q: What is explicit versus implicit FTPS?**

A: Explicit FTPS connects to port 21, and the client upgrades with AUTH TLS. Implicit FTPS uses port 990 and starts TLS at once. Explicit is the standard in RFC 4217 and is preferred. Implicit is a legacy method.

**Q: Why does TFTP use UDP, and how does it stay reliable?**

A: UDP makes the code very small, which fits in boot firmware. TFTP gets reliability from lock-step: one block, one ACK, with a retry on timeout. A block shorter than the block size ends the transfer. The cost is speed, since only one block moves per round trip.

**Q: Where would you use TFTP, and what are the risks?**

A: I would use it for PXE boot and to load images on network devices, on a trusted management network. It has no login and no encryption, so anyone who can reach it may read or write files. I would keep it off the internet, limit it to a folder, and use SCP or SFTP on devices that support them.

**Q: SCP or SFTP or rsync: when do you use each?**

A: SFTP is my default for interactive or scripted transfers, since it can list files and resume. scp is fine for a quick copy, and modern OpenSSH uses the SFTP protocol under it. rsync is best for repeat syncs and backups, because it transfers only changes and can resume.

**Q: How does rsync avoid sending the whole file?**

A: The receiver splits its old copy into blocks and sends a weak rolling checksum and a strong checksum for each. The sender slides over the new file byte by byte to find matching blocks. It then sends only new bytes, plus instructions to reuse old blocks. The rolling checksum means blocks are found even if data has shifted.

**Q: A bank partner needs daily file exchange over the internet. What do you design?**

A: I would use SFTP with SSH key login, a chroot for each partner, and a source IP allow-list on TCP 22. I would pin host keys in scripts, verify files with a hash, and log every transfer. If they require certificates, I would use explicit FTPS with PROT P and a fixed passive range. Plain FTP is not an option.

## Quick Recap

- FTP uses a control connection on TCP 21 and a separate data connection, and it sends everything, including passwords, in clear text.
- Active means the server opens the data connection; passive means the client does. Passive is the normal choice, and the server must open a fixed port range.
- FTP breaks firewalls and NAT because it puts IPs and ports in its text. ALGs can fix this, but not when FTPS encrypts the control connection.
- FTPS is FTP with TLS (ports 21 or 990, plus data ports). SFTP is a different protocol inside SSH (port 22 only).
- SCP is a simple copy over SSH, and modern OpenSSH uses the SFTP protocol under the `scp` command.
- TFTP is UDP 69 with lock-step ACKs, no login and no encryption, used for PXE boot and device images on trusted networks.
- rsync sends only changed data using rolling checksums, and it usually runs over SSH (TCP 22), with a clear-text daemon option on TCP 873.
