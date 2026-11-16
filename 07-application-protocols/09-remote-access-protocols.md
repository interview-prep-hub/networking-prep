# Remote Access Protocols (SSH, Telnet, RDP)

**Level:** Intermediate | **Prerequisites:** [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md), [TCP Header and the Three-Way Handshake](../06-transport-layer/03-tcp-handshake-and-header.md), [Cryptography Basics](../09-security/02-cryptography-basics.md)

## In One Line

Remote access protocols let you control a distant computer as if you sat in front of it: Telnet does it with no encryption, SSH does it with strong encryption for text, and RDP and VNC do it for a full graphical desktop.

## Why It Matters

Almost no server sits on your desk. Servers live in data centres and clouds. Routers and switches live in locked rooms in other cities. Admins need to reach all of them over a network.

That creates a hard problem. The login traffic crosses networks you do not control. If anyone on the path can read it, they get the password. If anyone can change it, they can run commands as you.

Interviewers ask about this topic because it joins many ideas in one place: ports, TCP, encryption, keys, trust, firewalls and NAT. A good answer to "what happens when you run `ssh`?" shows real depth. Port forwarding and jump hosts also come up in almost every DevOps and cloud interview, because private subnets are reached that way.

## Simple Explanation

Imagine you must send instructions to a worker in a distant office.

**Telnet is a postcard.** You write "my password is S3cret" on the card. Every postman on the way can read it.

**SSH is a locked box with a checked identity.** Before you say anything, the office shows you an ID card. You check it against the copy you saved last time. Then you and the office agree on a lock that only you two can open. Only then do you give your name and prove who you are.

**RDP and VNC are like a window and a remote arm.** You do not send written notes. You see the worker's screen, and your mouse and keyboard move on that screen.

Where the analogy stops: a real locked box must be carried by someone. In SSH nothing physical moves. Both sides do maths in public, and they end up with the same secret key without ever sending it.

## How It Works

First, some terms used in the rest of this chapter:

- **Terminal:** the text window where you type commands.
- **Shell:** the program that reads your commands and runs them, such as `bash` or PowerShell.
- **Daemon:** a background program that waits for connections. The SSH daemon is `sshd`.
- **Client and server:** the client starts the connection. The server listens for it.

### Telnet, and why not

Telnet is one of the oldest internet protocols. Work on it started in 1969, and the current standard is RFC 854 from 1983. It uses **TCP port 23**.

Telnet carries your keystrokes to the server and the text output back. It works in plain text. Nothing is encrypted, so usernames, passwords and every command can be read by anyone who can capture the packets.

Here is what a person sees in a packet capture of a Telnet login, using "Follow TCP Stream" in [Wireshark](../12-troubleshooting-and-tools/07-wireshark.md):

```text
login: aalliiccee
Password: S3cret-Pass
```

The login name looks doubled. The client sends each letter, and the server echoes it back. The password appears once, because the server does not echo it. It is still fully visible.

Telnet has three big failures:

| Problem | What it means |
|---|---|
| No confidentiality | Anyone on the path can read the password and commands |
| No integrity | An attacker in the path can change or inject commands |
| No server identity | You cannot tell if you reached the real server |

Telnet still appears in old network devices, lab gear and small embedded devices. The Mirai botnet of 2016 grew by scanning for Telnet on such devices and trying default passwords.

The fuller truth: encryption extensions for Telnet exist, but they were rarely deployed. In practice, treat Telnet as plain text.

The Telnet client program is still used as a quick port test, for example `telnet 10.10.20.30 25`. Better tools are `nc -vz` on Linux and `Test-NetConnection` on Windows. See [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md).

The old Unix tools `rlogin` and `rsh` had the same weakness and are also gone from good practice.

### What SSH is

SSH means **Secure Shell**. It gives you an encrypted connection to a remote machine. It uses **TCP port 22**.

Tatu Ylonen created it in 1995 after a password-sniffing attack at his university in Finland. The version used today is SSH-2, standardised in 2006. SSH-1 had design flaws and must not be used. OpenSSH, the most common implementation, removed SSH-1 support in version 7.6.

SSH does three jobs:

1. **Confidentiality:** outsiders cannot read the traffic.
2. **Integrity:** outsiders cannot change the traffic without being noticed.
3. **Authentication:** the client checks the server, and the server checks the user.

SSH-2 is built as three layers on top of TCP:

```text
+------------------------------------------------------+
| Connection protocol (RFC 4254)                       |
| Many channels in one connection: shell, commands,    |
| port forwards, file transfer                         |
+------------------------------------------------------+
| User authentication protocol (RFC 4252)              |
| Proves who the user is: public key, password, ...    |
+------------------------------------------------------+
| Transport protocol (RFC 4253)                        |
| Key exchange, server identity, encryption, integrity |
+------------------------------------------------------+
| TCP, normally port 22                                |
+------------------------------------------------------+
```

File transfer with SFTP and SCP also runs over SSH. See [File Transfer Protocols](08-file-transfer-protocols.md).

### The SSH handshake step by step

```text
Client (laptop)                                     Server
   |                                                   |
   |<========= TCP three-way handshake, port 22 ======>|
   |                                                   |
   |-- "SSH-2.0-OpenSSH_9.6p1 ..." ------------------->|  1. Versions
   |<------------------- "SSH-2.0-OpenSSH_9.6p1 ..." --|
   |                                                   |
   |-- KEXINIT: algorithms I support ----------------->|  2. Algorithms
   |<----------------- KEXINIT: algorithms I support --|
   |                                                   |
   |-- KEX INIT: client temporary public value ------->|  3. Key
   |<-- KEX REPLY: host key, server temporary value, --|     exchange
   |    signature made with the host private key       |
   |                                                   |
   |   Client checks host key against known_hosts      |  4. Verify
   |   Client verifies the signature                   |     server
   |                                                   |
   |-- NEWKEYS --------------------------------------->|  5. Encryption
   |<--------------------------------------- NEWKEYS --|     starts
   |                                                   |
   |============ everything below is encrypted ========|
   |                                                   |
   |-- request "ssh-userauth" ------------------------>|  6. User
   |-- user alice, public key, signature ------------->|     login
   |<---------------------------------- auth success --|
   |                                                   |
   |-- open channel "session", request shell --------->|  7. Session
   |<=================== shell input and output ======>|
```

1. **Version exchange.** Each side sends a plain text line such as `SSH-2.0-OpenSSH_9.6p1`. This is why a port scanner can see the SSH version.
2. **Algorithm negotiation.** Each side lists the key exchange methods, host key types, ciphers and message checks it supports. They pick the first entry in the client's list that the server also supports. A **cipher** is the method used to encrypt data.
3. **Key exchange.** Both sides use a Diffie-Hellman style exchange. Each side makes a temporary key pair and sends the public part. Each side mixes its own private part with the other's public part, and both reach the same shared secret. An eavesdropper sees only the public parts and cannot compute the secret.
4. **Server check.** The server signs a hash of the whole exchange with its long-term **host private key**. The client checks the signature and checks that the host key is the one it expects. This stops a person in the middle.
5. **Encryption starts.** Both sides derive the session keys from the shared secret and send `NEWKEYS`. All later data is encrypted and protected.
6. **User authentication.** Only now, inside the encrypted tunnel, does the user prove who they are.
7. **Channels.** The client opens a channel for a shell, a single command or a port forward.

A fact interviewers like: **the server proves itself before the user sends any secret.** A password is never sent to an unverified server, as long as the user takes the host key check seriously.

Because the temporary keys are thrown away, SSH has **forward secrecy**. If someone records the traffic and steals the host key next year, they still cannot decrypt the old session.

The fuller truth: RFC 4253 also recommends changing keys during long sessions, after about 1 GB of data or one hour. This is called rekeying, and OpenSSH does it automatically, with limits based on the cipher.

### User authentication methods

| Method | How it works | Notes |
|---|---|---|
| Password | Password is sent inside the encrypted tunnel | Can be guessed. The server sees the password |
| Public key | User proves they hold a private key | Standard for servers |
| Keyboard-interactive | Server asks questions, such as a one-time code | Often used for multi-factor login |
| Certificate | A trusted authority signed the user's key | Best at large scale. See Going Deeper |
| GSSAPI (Kerberos) | Uses an existing corporate ticket | Common in Active Directory style setups |

### Key-based authentication

A **key pair** is two linked keys. What one signs, the other can verify.

- The **private key** stays on your laptop, for example `~/.ssh/id_ed25519`.
- The **public key** goes on each server, in `~/.ssh/authorized_keys`.

Here is what happens at login:

1. The client says: "I am alice, and I want to use this public key."
2. The server looks in alice's `authorized_keys` file. If the key is listed, it replies "that key is acceptable".
3. The client creates a signature with the private key. The signed data includes the **session identifier**, a value unique to this SSH connection.
4. The server verifies the signature with the public key. If it is valid, login succeeds.

The private key is never sent. A stolen signature is useless, because it is tied to one session and cannot be replayed on another.

Analogy: the server holds a special padlock that only your key fits. You prove you own the key by using it in front of the server. You never hand the key over. Where this stops being true: a signature is not really locking. It is a mathematical proof that anyone with the public key can check.

Which key type should you use?

| Type | Status in 2026 |
|---|---|
| Ed25519 | Best default for most users. Short and fast |
| RSA | Fine with 3072 bits or more. Needed for some old systems |
| ECDSA | Works, but most people choose Ed25519 |
| DSA | Obsolete. Removed from OpenSSH in version 10.0 |
| `ed25519-sk` | Private key is tied to a hardware security key |

Protect the private key with a **passphrase**, a password that encrypts the key file on disk. To avoid typing it each time, use **ssh-agent**, a small program that holds the unlocked key in memory and signs on your behalf.

### Host keys and known_hosts

There are two kinds of key, and people often mix them up:

| | Host key | User key |
|---|---|---|
| Proves the identity of | The server | The user |
| Private part lives on | Server, in `/etc/ssh/ssh_host_*_key` | Laptop, in `~/.ssh/id_*` |
| Public part stored on | Client, in `~/.ssh/known_hosts` | Server, in `~/.ssh/authorized_keys` |

On the first connection, the client has no record of the server, so it asks:

```text
The authenticity of host '203.0.113.10 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:gYncO2hQ0tOTf5rv3CRDXI8jhZEQmuOAkX12awF+JlQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

A **fingerprint** is a short hash of the public key. It is easy for a person to compare. To answer properly, get the fingerprint through a safe path, such as the cloud console, and compare it. On the server, this command prints it:

```bash
# Linux server
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# 256 SHA256:gYncO2hQ0tOTf5rv3CRDXI8jhZEQmuOAkX12awF+JlQ root@bastion (ED25519)
```

If you say yes, the client stores the key in `known_hosts`. Each line holds a host name or address, a key type and the key:

```text
203.0.113.10 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJC0iomobLhz9zijXxznWr4lqewJuk1NCZt6i/z8wTq4
```

Some systems hash the host name so a stolen file does not reveal a list of your servers. Those lines start with `|1|`. This is the `HashKnownHosts` option.

This trust model is called **TOFU, trust on first use**. The first connection is the weak moment. Every later connection is checked against the saved key.

Analogy: the first time you meet a new colleague, you remember their face. If a stranger later claims to be them, you notice. Where it stops being true: a face can change slowly and honestly, but a host key either matches exactly or does not.

If the key changes, SSH refuses to connect:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
The fingerprint for the ED25519 key sent by the remote host is
SHA256:Vb3kQ2mXf8sYtR1nHcZp6uLwE9aJgD4oKiT7yNqM0xU.
Offending ED25519 key in /home/alice/.ssh/known_hosts:3
Host key verification failed.
```

(Part of the message is trimmed here.) Normal reasons include a rebuilt server, a reinstalled OS, or an IP address now used by a different machine. The dangerous reason is a man-in-the-middle attack. Do this:

1. Stop. Do not just delete the line.
2. Find out why the key changed. Ask the owner or check the change log.
3. Get the new fingerprint through a separate, trusted path.
4. Only then remove the old entry and reconnect.

```bash
# Linux, macOS and Windows PowerShell
ssh-keygen -R 203.0.113.10
```

The `StrictHostKeyChecking` client option controls this behaviour:

| Value | Behaviour |
|---|---|
| `ask` (default) | Prompt for new hosts, refuse changed keys |
| `accept-new` | Save new hosts silently, refuse changed keys |
| `yes` | Refuse any host not already known |
| `no` | Accept almost anything. Avoid this |

`accept-new` is a fair choice for automation. `no`, often copied from the internet with `UserKnownHostsFile=/dev/null`, removes the protection completely.

### Port forwarding (SSH tunnels)

An SSH connection can carry other TCP traffic inside it. This is called **port forwarding** or **tunnelling**.

Analogy: SSH is a private tube through a wall. You can pass letters for other people through the tube. Where it stops being true: the tube only protects the part between the two SSH ends. After the far end, the traffic travels normally.

There are three types. The test network for the examples:

```text
 Laptop                  Bastion                    Private subnet
 192.168.1.50            203.0.113.10 (public)      10.10.20.0/24
 (behind home NAT)       10.10.0.5    (private)

 +----------+   SSH     +----------+   plain TCP   +--------------+
 |  client  |==========>|  sshd    |-------------->| db           |
 |          |  port 22  |          |               | 10.10.20.40  |
 +----------+           +----------+               +--------------+
```

**Local forwarding (`-L`): bring a remote service to my machine.**

```bash
# Linux, macOS and Windows PowerShell
ssh -L 5433:10.10.20.40:5432 alice@203.0.113.10
```

The format is `-L local_port:destination_host:destination_port`.

1. The client listens on `127.0.0.1:5433` on the laptop.
2. A program on the laptop connects to that port.
3. The bytes travel through the SSH tunnel to the bastion.
4. `sshd` on the bastion opens a normal TCP connection to `10.10.20.40:5432`.

The destination is looked up **from the server's point of view**. So `localhost` in `-L 8080:localhost:80` means the SSH server, not your laptop.

**Remote forwarding (`-R`): publish my local service on the remote machine.**

```bash
ssh -R 8080:127.0.0.1:3000 alice@203.0.113.10
```

The bastion listens on port 8080. Anything that connects there travels back through the tunnel to port 3000 on the laptop.

This is how a machine behind [NAT](../04-ip-addressing/09-nat-and-pat.md) can offer a service without any inbound firewall rule, because the laptop made the outbound connection. It can help with support and demos. It is also a classic way for attackers to get around firewalls.

By default the remote listener binds only to loopback on the server. To accept outside clients, the server needs `GatewayPorts yes` or `GatewayPorts clientspecified` in `sshd_config`.

**Dynamic forwarding (`-D`): a SOCKS proxy.**

```bash
ssh -D 1080 -N alice@203.0.113.10
```

The client becomes a local **SOCKS proxy**, a helper that takes "connect me to this host and port" requests from applications. `-N` means "do not run a shell, only forward". Each application tells the proxy where it wants to go, and the bastion makes that connection. So you do not need to pick one fixed destination. More on proxies in [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

| Flag | Who listens | Who makes the final connection | Typical use |
|---|---|---|---|
| `-L` | SSH client | SSH server | Reach a private database from a laptop |
| `-R` | SSH server | SSH client | Expose a local service to the remote side |
| `-D` | SSH client | SSH server, destination varies | Browse an internal network |

Memory trick: **L**ocal means the listener is on the **L**aptop. **R**emote means the listener is on the **R**emote machine.

Useful flags: `-N` runs no remote command, and `-f` goes to the background after login.

### Jump hosts (bastions)

A **jump host**, also called a **bastion**, is a hardened machine at the edge of a private network. It is the only machine that accepts SSH from outside. To reach anything inside, you pass through it.

Analogy: the reception desk of a secure building. Everyone signs in at one point, so there is one door to guard and one log to read. Where it stops being true: with `ProxyJump`, the receptionist only passes sealed boxes. They cannot read your conversation with the person upstairs.

```text
                     Firewall: allow 22 only to bastion
                                  |
 Laptop ===== SSH to bastion =====|==> Bastion 203.0.113.10
    ||                            |       |
    ||                            |       | carries encrypted bytes
    ||                            |       v
    ++========= inner SSH, end to end =========> App 10.10.20.30
```

The modern way is the `-J` flag, added in OpenSSH 7.3:

```bash
ssh -J alice@203.0.113.10 alice@10.10.20.30
```

The client makes one SSH connection to the bastion. It asks the bastion to open a TCP path to `10.10.20.30:22`. It then runs a **second, separate SSH session** through that path. Your key stays on your laptop, and the host key of the target is checked by your laptop. The bastion sees only encrypted bytes. The bastion must allow TCP forwarding for this to work.

**Agent forwarding (`-A`) is the old way, and it is riskier.** You log in to the bastion, and the bastion can ask the agent on your laptop to sign things. Anyone with root on the bastion can use your agent while you are connected. They cannot copy your private key, but they can log in as you. Prefer `-J`.

Also, never copy your private key onto a bastion.

### RDP

**RDP (Remote Desktop Protocol)** is Microsoft's protocol for a full graphical Windows desktop. It uses **TCP port 3389**, and modern versions also use **UDP port 3389** for better speed and smoothness.

The client sends keyboard and mouse events. The server sends back the screen. RDP also carries extras over **virtual channels**, which are logical streams inside the connection. These cover the clipboard, printers, local drives, sound and more.

Connection steps, simplified:

1. TCP connection to port 3389.
2. Client and server negotiate which security to use.
3. TLS starts. See [TLS and HTTPS](07-tls-and-https.md).
4. **NLA (Network Level Authentication)** checks the user. This uses a protocol called CredSSP, which runs Kerberos or NTLM.
5. Only then does the server create a Windows session, and the desktop appears.

Why NLA matters: without it, the server draws a login screen first. That spends server resources on anyone who connects, and it exposes more code to attackers before login. BlueKeep (CVE-2019-0708) was a serious pre-login flaw in Remote Desktop Services on older Windows versions. NLA reduced the risk, because an attacker needed valid credentials first. It did not fix the flaw. Patching did.

Facts worth knowing:

- Windows Home editions can connect out as a client, but they cannot be an RDP server.
- Windows desktop editions allow one interactive user at a time.
- Windows Server allows two admin sessions by default. More users need the Remote Desktop Services role and client licences.
- The server normally uses a self-signed certificate, so users see a warning. This is the same trust problem as the SSH host key. Use a certificate from your company's CA to remove it.

Commands:

```powershell
# Windows client
mstsc /v:10.10.20.50

# Windows server: turn on RDP (run as admin)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

```bash
# Linux client, using FreeRDP (the command is xfreerdp3 on some newer distros)
xfreerdp /v:10.10.20.50 /u:alice
```

Linux can also act as an RDP server by using `xrdp`.

### VNC

**VNC (Virtual Network Computing)** shows a remote desktop using the **RFB (Remote Framebuffer)** protocol, RFC 6143. A **framebuffer** is the block of memory that holds the image on the screen. VNC works on Linux, macOS and Windows.

VNC uses **TCP port 5900 + display number**. Display `:1` means port 5901.

The server sends rectangles of changed pixels. This is simple and works across operating systems, but it can use more bandwidth than RDP.

The important weakness: the classic VNC security type uses a weak DES challenge with a password that most software limits to 8 characters, and base RFB does **not encrypt** the session. Some products add their own encryption, but do not assume it. The standard fix is to keep VNC on loopback and tunnel it over SSH:

```bash
# On the server (TigerVNC on Linux): listen on loopback only
vncserver :1 -localhost yes

# On the client: tunnel port 5901 to the server's loopback
ssh -L 5901:127.0.0.1:5901 alice@10.10.20.30

# Then, in a second terminal, point the viewer at the local end
# (display 1 means port 5901)
vncviewer 127.0.0.1:1
```

### Comparing the protocols

| | Telnet | SSH | RDP | VNC |
|---|---|---|---|---|
| Default port | TCP 23 | TCP 22 | TCP and UDP 3389 | TCP 5900+ |
| Interface | Text | Text, plus tunnels | Graphical | Graphical |
| Encrypted | No | Yes | Yes (TLS) | Not in base RFB |
| Main OS | Old devices | Linux, network gear, Windows too | Windows | Cross-platform |
| Screen method | None | None | Drawing commands and codecs | Pixel rectangles |
| Sessions | Many | Many | Separate user sessions | Often shares one screen |

The fuller truth on sessions: some VNC servers, such as TigerVNC's `vncserver`, make a separate virtual desktop rather than share the physical screen.

### Hardening basics

**Hardening** means removing what you do not need and making attacks harder.

**SSH servers (OpenSSH):**

1. Use keys and turn off password login.
2. Do not allow direct root login. The OpenSSH default is `prohibit-password`, which still allows root with a key.
3. Allow only named users or groups.
4. Restrict port 22 by source address, using a [firewall](../09-security/04-firewalls.md) or cloud security group. Put a bastion or VPN in front.
5. Add rate limits, such as `fail2ban`, a tool that bans addresses after repeated failures.
6. Keep OpenSSH patched. regreSSHion (CVE-2024-6387) was a remote code execution flaw in OpenSSH 8.5p1 to 9.7p1 on glibc Linux, fixed in 9.8p1.
7. Turn off features you do not use, such as X11 forwarding (showing remote graphical apps) and agent forwarding.
8. Require a second factor for admins. A hardware key is strongest.

A sample file for Ubuntu or Debian:

```text
# /etc/ssh/sshd_config.d/00-hardening.conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups ssh-users
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowAgentForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Create the `ssh-users` group and add your own user to it first. Otherwise `AllowGroups` locks everyone out.

Apply it safely:

```bash
sudo sshd -t                       # test the syntax
sudo sshd -T | grep -i passwordauth   # show the effective value
sudo systemctl reload ssh          # the service is "sshd" on RHEL
```

Two warnings from real life:

- **Keep your current session open** and test login from a second terminal before you close it. Otherwise, one mistake can lock you out.
- **The first value wins in `sshd_config`.** Files in `sshd_config.d` are read first on Debian and Ubuntu. A cloud image file such as `50-cloud-init.conf` may set `PasswordAuthentication yes`. That is why the file above starts with `00-`. Always check with `sshd -T`.

Changing the port from 22 to something like 2222 reduces noise in logs. It is not real security, because a scan finds it quickly.

**RDP:**

1. Never expose TCP 3389 directly to the internet. Exposed RDP is a common way in for ransomware.
2. Use a VPN or an **RD Gateway**, which wraps RDP in HTTPS on port 443.
3. Require NLA and multi-factor login.
4. Use account lockout and strong passwords.
5. Turn off drive and clipboard redirection on sensitive servers.

**VNC:** bind to loopback and tunnel it. Never expose 5900 to the internet.

**Network devices:** turn off Telnet, use SSH version 2 only, limit the VTY lines (virtual terminal lines for remote logins) with an [ACL](../09-security/05-access-control-lists.md), use central login through TACACS+ or RADIUS (see [Directory and Authentication Protocols](12-directory-and-authentication-protocols.md)), and keep **out-of-band** access. That means a separate path, such as a serial console server, that still works when the main network is down.

## Example

Alice works from a laptop at `192.168.1.50`. She needs to reach a private app server and a private database through a bastion.

```text
 Laptop            NAT             Bastion            Private subnet
 192.168.1.50 --> 198.51.100.7 --> 203.0.113.10 ----> app 10.10.20.30
                                   10.10.0.5     |    db  10.10.20.40
                                                 +--> win 10.10.20.50
```

### Step 1: create a key

```bash
# Linux, macOS and Windows PowerShell
ssh-keygen -t ed25519 -C "alice@laptop"
```

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/alice/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/alice/.ssh/id_ed25519
Your public key has been saved in /home/alice/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:39Rk4xnvQJTQsO//3rzTqtwjcBc5JhagNGQ8h+FaPBM alice@laptop
```

The Windows OpenSSH client is built into Windows 10 and 11, so the same commands work there. On Windows, keys go in `C:\Users\alice\.ssh\`.

### Step 2: put the public key on the server

```bash
# Linux and macOS
ssh-copy-id alice@203.0.113.10
```

```powershell
# Windows PowerShell has no ssh-copy-id. This appends the key on a Linux server.
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh alice@203.0.113.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

The line added to the server's `authorized_keys` is the public key:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGjoWgx2Ggw0INR4z3I2CAkHTBTU+XX85IpSfBpFZEgV alice@laptop
```

Permissions matter on Linux. With the default `StrictModes yes`, `sshd` ignores the file if `~/.ssh` or `authorized_keys` is writable by others. Use `700` on the folder and `600` on the file.

### Step 3: log in and read the debug output

`-v` shows the handshake steps:

```bash
ssh -v alice@203.0.113.10
```

```text
debug1: Connecting to 203.0.113.10 [203.0.113.10] port 22.
debug1: Connection established.
debug1: Local version string SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.6p1 Ubuntu-3ubuntu13.5
debug1: kex: algorithm: sntrup761x25519-sha512@openssh.com
debug1: kex: host key algorithm: ssh-ed25519
debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: Server host key: ssh-ed25519 SHA256:gYncO2hQ0tOTf5rv3CRDXI8jhZEQmuOAkX12awF+JlQ
debug1: Host '203.0.113.10' is known and matches the ED25519 host key.
debug1: SSH2_MSG_NEWKEYS sent
debug1: Authentications that can continue: publickey
debug1: Offering public key: /home/alice/.ssh/id_ed25519 ED25519 SHA256:39Rk4xnvQJTQsO//3rzTqtwjcBc5JhagNGQ8h+FaPBM
debug1: Server accepts key: /home/alice/.ssh/id_ed25519 ED25519 SHA256:39Rk4xnvQJTQsO//3rzTqtwjcBc5JhagNGQ8h+FaPBM
Authenticated to 203.0.113.10 ([203.0.113.10]:22) using "publickey".
```

(Lines are trimmed here.) You can match each line to a handshake step: versions, algorithms, host key check, `NEWKEYS`, then user login. `<implicit>` means the cipher has its own integrity check built in, so no separate MAC (message authentication code) is needed.

### Step 4: save the settings in a client config file

Put this in `~/.ssh/config`:

```text
Host bastion
    HostName 203.0.113.10
    User alice

Host app
    HostName 10.10.20.30
    User alice
    ProxyJump bastion

Host db-tunnel
    HostName 203.0.113.10
    User alice
    LocalForward 5433 10.10.20.40:5432
```

Now `ssh app` passes through the bastion with no extra flags.

### Step 5: reach the private database

```bash
# On the laptop, terminal 1
ssh -N db-tunnel

# On the laptop, terminal 2: check the listener
ss -tln | grep 5433
```

```text
LISTEN 0      128        127.0.0.1:5433      0.0.0.0:*
```

```bash
# Terminal 2: connect to Postgres through the tunnel
psql -h 127.0.0.1 -p 5433 -U appuser appdb
```

The path is: `psql` to `127.0.0.1:5433`, through encrypted SSH to the bastion, then plain TCP from `10.10.0.5` to `10.10.20.40:5432`. The last hop is not protected by SSH. The database sees the connection coming from the bastion, not the laptop.

### Step 6: browse internal sites through a SOCKS proxy

```bash
ssh -D 1080 -N bastion

# In a second terminal
curl --socks5-hostname 127.0.0.1:1080 http://10.10.20.30:8080/health
```

`--socks5-hostname` makes the proxy resolve names too, so internal DNS names work and your local DNS server does not see them.

### Step 7: RDP through the bastion from Windows

The Windows server at `10.10.20.50` has no public address. Alice tunnels it:

```powershell
# Windows laptop, terminal 1
ssh -N -L 13389:10.10.20.50:3389 bastion

# Terminal 2
mstsc /v:127.0.0.1:13389
```

She uses local port 13389 because 3389 may already be taken by the laptop's own RDP service. Only the TCP part of RDP passes through this tunnel, so the UDP speed-up is not used. A session may feel a little slower.

### Step 8: replace Telnet on a Cisco router

```text
! Cisco IOS
hostname R1
ip domain name example.com
username admin privilege 15 algorithm-type scrypt secret Str0ng-Example-Pass
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2
!
ip access-list standard MGMT
 permit 10.10.0.0 0.0.255.255
!
line vty 0 4
 login local
 transport input ssh
 access-class MGMT in
 exec-timeout 10 0
```

The hostname and domain name are needed because the RSA key is labelled with them. `transport input ssh` removes Telnet from these lines. Some platforms have more VTY lines, such as `0 15`, and each range needs this config.

```text
R1# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 2
```

The exact output and supported algorithms vary by IOS version.

## Going Deeper

### The SSH binary packet

After the version lines, SSH data travels in packets with this layout (RFC 4253):

| Field | Size | Meaning |
|---|---|---|
| `packet_length` | 4 bytes | Length of what follows, not counting the MAC |
| `padding_length` | 1 byte | Number of padding bytes |
| `payload` | Variable | The actual message |
| `random padding` | At least 4 bytes | Makes the total fit the cipher block size |
| `MAC` | Depends on algorithm | Integrity check |

The fuller truth: how this is encrypted depends on the cipher. With `chacha20-poly1305@openssh.com`, the length is encrypted too, and the integrity tag replaces the MAC. With "encrypt-then-MAC" modes, the length stays in clear.

### Post-quantum key exchange

A future quantum computer could break today's Diffie-Hellman. An attacker could record traffic now and decrypt it later. OpenSSH has defended against this with hybrid key exchange, which mixes a classic method with a post-quantum one. `sntrup761x25519-sha512` became the default in OpenSSH 9.0. Since OpenSSH 10.0, `mlkem768x25519-sha256` is preferred. Host and user signature keys are a separate matter and are not post-quantum in normal use today.

These key exchange messages are larger. Larger packets can reveal path MTU problems. See below.

### SSH certificates

With plain keys, every server needs every user's key, and every client needs every host key. This does not scale. SSH certificates solve it. A **certificate** here is a public key signed by a trusted **CA (certificate authority)**, with a name and an expiry time.

```bash
# On the CA machine: sign alice's key, valid for 8 hours, for login name alice
ssh-keygen -s ca_key -I alice-laptop -n alice -V +8h id_ed25519.pub
```

```text
# On each server, in sshd_config: trust this CA for user logins
TrustedUserCAKeys /etc/ssh/user_ca.pub

# On each client, in known_hosts: trust this CA for host keys
@cert-authority *.example.com ssh-ed25519 AAAAC3...
```

Short-lived certificates reduce the need to remove old keys, and host certificates end the TOFU prompt. These are not X.509 certificates like those in TLS. OpenSSH uses its own simpler format.

### SSH trust compared with TLS

| | SSH | TLS |
|---|---|---|
| How the server is trusted | TOFU, or an SSH CA | Public CA system, chain of certificates |
| Normal client identity | Key inside the protocol | Often login at the application layer |
| Typical use | Admin access, tunnels | Web and API traffic |

Another option is SSHFP, a DNS record that publishes host key fingerprints (RFC 4255). It is only safe with DNSSEC.

### Terrapin

Terrapin (CVE-2023-48795) is a 2023 attack on the SSH transport. A person in the middle could delete messages at the start of the encrypted phase, which could weaken some extension negotiation. It needs an active attacker in the path and specific ciphers. The fix, called "strict key exchange", arrived in OpenSSH 9.6. Both client and server must support it.

### Limiting what a key can do

Options can go in front of a key in `authorized_keys`:

```text
from="10.10.0.0/16",command="/usr/local/bin/backup",no-port-forwarding,no-pty ssh-ed25519 AAAAC3...
```

This key works only from one network and can run only one program. It is a good pattern for automation.

### Server limits on forwarding

`AllowTcpForwarding no` in `sshd_config` disables port forwarding. The OpenSSH manual notes this does not help if users have a shell, because they can run their own forwarder. For a bastion that only relays, use a user with no shell. Note that blocking forwarding also breaks `-J`.

### Tunnel performance

With `-L`, `-R` and `-D`, the application's TCP connection ends at the SSH process. The bytes then ride in the single SSH TCP connection. So this is not true TCP inside TCP.

`ssh -w` is different. It creates a tun device (a virtual network interface) and carries IP packets inside SSH, which is true TCP inside TCP, and it can perform badly under packet loss. For that job, use a real [VPN](../09-security/07-vpn.md). Also, all channels share one TCP connection, so one lost packet delays all of them.

### Windows credential exposure over RDP

In a normal RDP login, your credentials reach the remote machine and can sit in its memory. If that machine is compromised, an attacker may steal them. Two options reduce this: `mstsc /restrictedAdmin` and `mstsc /remoteGuard` (Remote Credential Guard). Both have conditions and limits, so check them before use.

### Related tools

- **Windows OpenSSH server:** an optional feature, installed with `Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0`. Keys for admins go in `C:\ProgramData\ssh\administrators_authorized_keys`.
- **PuTTY:** a popular Windows client with its own key format, `.ppk`.
- **Mosh:** starts with SSH, then moves to UDP (by default in the range 60000-61000). It survives IP changes and poor mobile links.
- **SSH escape keys:** press Enter, then `~.` to kill a frozen session or `~#` to list forwards.
- **Cloud options:** AWS Systems Manager Session Manager gives a shell with no inbound port open on the instance. Azure Bastion gives SSH and RDP through the Azure portal, so the VM needs no public IP. This fits [Zero Trust](../09-security/09-zero-trust-and-segmentation.md) thinking. Security groups are covered in [Security Groups, NACLs and VPC Connectivity](../11-cloud-and-containers/04-cloud-security-and-connectivity.md).
- **Privileged access tools:** large companies place a proxy in front of admin access to record sessions and hand out short-lived credentials.

### Troubleshooting by error message

| Message | Likely meaning | What to check |
|---|---|---|
| `Connection timed out` | Packets get no reply. Often a firewall drop or routing | Security group, firewall, route |
| `Connection refused` | Host is up, nothing is listening (TCP RST) | `systemctl status ssh`, port number |
| `kex_exchange_identification: Connection closed` | Server closed before the handshake | Rate limits, `fail2ban`, `MaxStartups`, proxy |
| `no matching key exchange method found` | No shared algorithm with an old device | Upgrade device, or allow one old method for that host only |
| `Host key verification failed` | Host key changed or unknown | Follow the checks in the host key section |
| `Permission denied (publickey)` | Key not accepted | User name, `ssh -v`, permissions, server log |
| `bind [127.0.0.1]:5433: Address already in use` | Local port already used | Pick another port |
| Hangs at `expecting SSH2_MSG_KEX_ECDH_REPLY` | Often a path MTU black hole | See [MTU, MSS and Path MTU Discovery](../06-transport-layer/09-mss-mtu-and-pmtud.md) |

Server logs live in `/var/log/auth.log` on Debian and Ubuntu, and `/var/log/secure` on RHEL. `journalctl -u ssh` also works (`-u sshd` on RHEL).

For an old device only, you can allow an old method per connection instead of weakening your whole client:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa admin@10.10.0.1
```

## Common Mistakes

- **Mixing up host keys and user keys.** The host key proves the server. The user key proves the user.
- **Thinking keys are safe without a passphrase.** A stolen, unprotected key file gives full access.
- **Copying the private key to a server.** Only the public key goes on servers. Use `-J` to pass through a bastion.
- **Deleting the `known_hosts` line without checking why it changed.** That warning can be a real attack.
- **Using `StrictHostKeyChecking no` in scripts.** Use `accept-new`, or manage host keys properly.
- **Thinking the tunnel protects the whole path.** With `-L`, SSH protects only the part up to the SSH server. The rest is ordinary traffic.
- **Reading `localhost` in `-L` as the laptop.** It is evaluated on the SSH server.
- **Expecting `-R` to be open to everyone.** It binds to loopback on the server unless `GatewayPorts` allows more.
- **Assuming a changed SSH port is security.** It only reduces noise.
- **Thinking the session is encrypted in VNC.** Base RFB does not encrypt. Tunnel it.
- **Saying NLA encrypts RDP.** NLA is about authenticating before a session is created. TLS provides the encryption.
- **Using `ClientAliveInterval` as an idle timeout.** It checks that the client is alive. An idle but connected client still answers, so the session stays open.
- **Thinking `Connection refused` is a firewall drop.** A refusal is an active reply. A drop usually shows as a timeout, although some firewalls do send a reset.
- **Locking yourself out.** Keep one session open while you test any `sshd` change.

## Interview Questions

**Q: Why do we use SSH instead of Telnet?**

A: Telnet sends everything in plain text on port 23, including the password. It has no integrity check and no way to verify the server. SSH on port 22 encrypts the session, detects tampering, verifies the server with a host key, and supports strong login with keys. Telnet should be off on any real network.

**Q: Walk me through what happens when I run `ssh alice@server`.**

A: First TCP connects to port 22. Both sides exchange version strings, then agree on algorithms. They run a Diffie-Hellman style key exchange to create a shared secret. The server signs the exchange with its host key, and the client checks that key against known_hosts. Then encryption starts. Inside the encrypted tunnel, alice authenticates, usually with a public key signature. Finally the client opens a channel and gets a shell.

**Q: What is the difference between a host key and a user key?**

A: A host key proves the server's identity. Its private part stays on the server, and clients save the public part in known_hosts. A user key proves the user's identity. Its private part stays on the laptop, and the public part goes in authorized_keys on the server.

**Q: How does public key login work without sending the private key?**

A: The client tells the server which public key it wants to use. If that key is in authorized_keys, the client signs data that includes the unique session identifier. The server checks the signature with the public key. The private key never leaves the client, and the signature cannot be reused in another session.

**Q: You see "REMOTE HOST IDENTIFICATION HAS CHANGED". What do you do?**

A: I stop and do not bypass it. The server's host key is different from the saved one. It could be a rebuild or a replaced machine, or it could be a man-in-the-middle attack. I confirm the reason with the owner, get the new fingerprint through a trusted path, then remove the old entry with `ssh-keygen -R` and reconnect.

**Q: Explain local, remote and dynamic port forwarding.**

A: With `-L`, the client listens, and the SSH server connects on to a fixed destination. I use it to reach a private database from my laptop. With `-R`, the server listens, and the traffic comes back to my side. I use it to expose a local service. With `-D`, the client runs a SOCKS proxy, and each application picks its own destination, with the SSH server making the connection.

**Q: A database is in a private subnet. How do you reach it from your laptop?**

A: I go through a bastion in a public subnet. For example, `ssh -L 5433:10.10.20.40:5432 alice@203.0.113.10`, then point my client at 127.0.0.1 port 5433. The firewall allows SSH only to the bastion, and allows the bastion to reach the database. The last hop is not covered by SSH, so the database connection should also use TLS.

**Q: Why is ProxyJump better than agent forwarding?**

A: With ProxyJump, my client runs a second SSH session end to end through the bastion. The bastion only relays encrypted bytes. With agent forwarding, the bastion can ask my agent to sign, so anyone with root on the bastion can log in as me while I am connected.

**Q: What does "connection refused" tell you compared with "connection timed out"?**

A: Refused means a TCP reset came back, so the host was reached but nothing listens on that port, or a device sent a reset for it. Timed out means no reply came at all, which usually points to a firewall drop, a routing problem or a host that is down. So they send me to different places first.

**Q: How would you harden an SSH server?**

A: I use key login and turn off passwords. I block root login and allow only certain groups. I limit port 22 by source, ideally through a bastion or VPN. I turn off unused forwarding, add rate limits like fail2ban, keep OpenSSH patched and require multi-factor for admins. At scale I use short-lived SSH certificates. I test with `sshd -t` and keep a session open while I apply the change.

**Q: What is the difference between RDP and VNC?**

A: RDP is Microsoft's protocol on port 3389, with TLS and NLA, and it usually gives each user a separate Windows session. VNC uses the RFB protocol on port 5900 plus the display number. It is cross-platform and sends pixel updates, and base RFB is not encrypted, so it should be tunnelled through SSH or a VPN.

**Q: What is NLA, and how would you expose RDP safely?**

A: NLA is Network Level Authentication. The user must log in before the server creates a session, which saves resources and reduces what an attacker can reach before login. For safety, I would not open 3389 to the internet. I would use a VPN or RD Gateway on 443, with NLA, multi-factor login, lockout and patching.

## Quick Recap

- Telnet (TCP 23) is plain text, so it must not be used for real logins. SSH (TCP 22) replaced it.
- In SSH, the server proves itself with a host key first, then the user logs in inside the encrypted tunnel.
- Keys are stronger than passwords. The private key stays with you, and the public key goes in `authorized_keys`.
- `known_hosts` stores trusted host keys. A changed key is a warning to investigate, not to bypass.
- `-L` listens on the client, `-R` listens on the server, and `-D` makes a SOCKS proxy. Use `-J` to pass through a bastion.
- RDP (3389) and VNC (5900+) give graphical access. Never expose either directly to the internet, and tunnel VNC.
- Hardening means keys only, no root login, restricted sources, patching, least features and a tested way back in.
