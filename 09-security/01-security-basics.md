# Network Security Basics

**Level:** Beginner | **Prerequisites:** [What Is a Network?](../01-foundations/01-what-is-a-network.md), [Network Devices](../01-foundations/04-network-devices.md)

## In One Line

Network security keeps data secret, correct and available by deciding who may do what, and by adding several layers of protection so that one failure does not become a disaster.

## Why It Matters

Every device on the internet gets attacked. A new server with a public address often sees automatic login attempts within minutes. No person chose that server. Programs scan all public addresses, all day.

Security is how you stay safe from this without stopping the business. A network that is perfectly safe but unusable has failed too. So security is a set of trade-offs, and you need shared words to discuss them.

Interviewers ask these basics because they show how you think. A good candidate can look at any design and ask three questions: what are we protecting, from whom, and what happens when one protection fails? Later topics such as [firewalls](04-firewalls.md), [VPNs](07-vpn.md) and [zero trust](09-zero-trust-and-segmentation.md) all build on the ideas below.

## Simple Explanation

Think of a bank.

- The bank keeps your balance **secret** from strangers.
- The bank keeps your balance **correct**. Nobody can change 100 to 10.
- The bank is **open** when you need your money.

These three goals are the heart of security.

Now think about how the bank protects itself:

- A thief is a **threat**. A broken back window is a **vulnerability**. How likely and how bad a robbery would be is the **risk**.
- A teller checks your ID (who are you?), looks up what you may do (may you touch this account?), and records the action (what did you do?). That is **AAA**.
- A cleaner has a key to the lobby, not to the vault. That is **least privilege**.
- The bank has a guard, a locked door, a vault and cameras. If one fails, the next still works. That is **defence in depth**.
- Every door and window is a place to break in. Together they are the **attack surface**.

Where the analogy stops: a bank has walls you can see, and a thief must travel to it. A network has no real walls. An attacker can be in another country, can try a million doors per hour, and can copy data without removing it. Your copy is still there, so you may never notice.

## How It Works

### The CIA Triad

The **CIA triad** is the three goals of security: Confidentiality, Integrity and Availability. It has nothing to do with the American spy agency. When you study any attack or defence, ask which of the three it affects.

```text
                    Confidentiality
                 (only the right people)
                          /\
                         /  \
                        /    \
                       / DATA \
                      /        \
                     /__________\
              Integrity        Availability
         (not changed)         (ready when needed)
```

| Goal | Meaning | Example of failure | Typical protection |
|---|---|---|---|
| Confidentiality | Only allowed people can read the data | Someone captures a Telnet password from the network | Encryption, access control |
| Integrity | Data is not changed without anyone noticing | An attacker changes the bank account number in an invoice | Hashes, digital signatures |
| Availability | Data and services work when needed | A flood of traffic takes a website offline | Redundancy, backups, DDoS protection |

Some terms from the table:

- **Encryption** scrambles data so that only someone with the key can read it. See [Cryptography Basics](02-cryptography-basics.md).
- A **hash** is a short fingerprint of data. Change one letter and the fingerprint changes completely.
- A **digital signature** proves who created some data and that it was not changed.
- **Redundancy** means keeping a spare, such as a second server or a second link.
- **DDoS** (Distributed Denial of Service) means many machines flood a target until real users cannot reach it.

The three goals can conflict. Strong login checks protect confidentiality but can slow down a doctor in an emergency. A hospital may rank availability first, a bank may rank integrity first, and a military network may rank confidentiality first. There is no single correct order.

The opposite of CIA is **DAD**: Disclosure, Alteration and Denial. It is the same idea from the attacker's side.

Two more goals are often added:

- **Authenticity:** the data or person is really who it claims to be.
- **Non-repudiation:** a person cannot later deny an action, because there is proof. A digital signature gives this. A shared password does not, because anyone with the password could have done it.

### Threat, Vulnerability and Risk

These words are not the same. Interviewers check this.

- An **asset** is anything of value that you protect: customer data, a server, your reputation.
- A **threat** is anything that could cause harm: an attacker, a virus, a flood, a tired employee.
- A **vulnerability** is a weakness that a threat can use: unpatched software, a default password, an open port.
- A **risk** is the chance of harm and how bad that harm would be.
- An **exploit** is the actual code or method that uses a vulnerability.

```text
   THREAT           VULNERABILITY            ASSET
   (what may  --->  (the weakness    --->    (what you
    cause harm)      it uses)                 protect)
                         |
                         v
          RISK = likelihood x impact
```

A simple formula is:

```text
Risk = Likelihood x Impact
```

If there is no threat, or no vulnerability, or the asset has no value, then the risk is close to zero. A bug in software that you do not run is not your risk.

The fuller truth: real risk work uses many methods, and likelihood is often an estimate. The formula is a thinking tool, not physics.

Once you know a risk, you have four choices:

| Choice | Meaning | Example |
|---|---|---|
| Reduce (mitigate) | Add a protection | Patch the server |
| Avoid | Stop the risky activity | Remove an old service that nobody uses |
| Transfer | Move the cost to someone else | Buy cyber insurance |
| Accept | Live with it, on purpose | Accept a small risk because the fix costs more than the harm |

The risk left after you act is **residual risk**. It is never zero.

You will also hear these words:

- **CVE** (Common Vulnerabilities and Exposures) is a public ID for a known vulnerability. Example: CVE-2021-44228 is Log4Shell, a flaw in the Java logging library Log4j.
- **CVSS** (Common Vulnerability Scoring System) rates how severe a vulnerability is, from 0.0 to 10.0. Log4Shell scored 10.0. A high score does not always mean high risk for you, because the score does not know your network.
- A **zero-day** is a vulnerability that the attacker knows about before a fix exists.
- **Patching** means installing a software update that fixes a flaw.

### AAA: Authentication, Authorization, Accounting

**AAA** is the three steps for controlling access. The analogy is a hotel. You show your ID at check-in. Your key card opens only your room. The door lock records each entry.

| Step | Question | Hotel | Network example |
|---|---|---|---|
| Authentication | Who are you? | Show ID at the desk | SSH key login to a router |
| Authorization | What may you do? | Key opens room 305 only | Help desk may run `show` commands, not change config |
| Accounting | What did you do? | Lock logs each entry | Log says who ran which command, and when |

Where the analogy stops: a hotel checks you once. A good network system can check again and again, and can cancel access in the middle of a session.

The order matters. You must prove who you are before the system can decide what you may do.

**Identification** comes first and is a separate idea. Typing a username is a claim. A password or key proves the claim.

Authentication uses **factors**, which are kinds of proof:

- Something you **know**: a password, a PIN.
- Something you **have**: a phone, a hardware security key.
- Something you **are**: a fingerprint, a face.

**MFA** (Multi-Factor Authentication) needs two or more different kinds. A password plus a code from a phone app is MFA. Two passwords is not, because both are the same kind. MFA stops many attacks that use stolen passwords. It is not perfect: some attackers trick users into approving a login.

In networks, a central server usually does the AAA work, so you do not create users on 500 routers one by one. Two protocols are common:

| | RADIUS | TACACS+ |
|---|---|---|
| Transport | UDP 1812 (auth), 1813 (accounting) | TCP 49 |
| Encryption | Hides only the password | Encrypts the whole message body |
| Typical use | Network access: Wi-Fi, VPN users | Admin login to routers and switches |

The full detail is in [Directory and Authentication Protocols](../07-application-protocols/12-directory-and-authentication-protocols.md).

Basic AAA on a router looks like this:

```text
! Cisco IOS: admin login checks the TACACS+ server first.
! The local user works only if the server cannot be reached.
aaa new-model
username breakglass privilege 15 secret Ch4nge-This-Value
tacacs server TAC1
 address ipv4 10.0.50.10
 key Ch4nge-This-Shared-Key
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
```

Test this from a second session before you log out, or a mistake can lock you out.

### Least Privilege

**Least privilege** means giving each user, device and program only the access it needs for its job, and only for as long as it needs it.

A hotel cleaner has a key for floor 3 during a work shift, not a master key forever. If that key is stolen, the thief gets one floor. Where the analogy stops: a stolen digital key can be copied and used from anywhere, often with no sign of theft.

Least privilege limits the damage, or **blast radius**, when something goes wrong. It also limits honest mistakes. A junior engineer who cannot edit the core router cannot break it by accident.

It applies at every level:

- **Users:** Bob can restart one service, but he is not a full administrator.
- **Networks:** the web server may reach the database on one port, and nothing else.
- **Programs:** a web server runs as a normal user, not as root.

Linux example, a `sudoers` rule (edit with `visudo`):

```text
# Linux: bob may run only this one command as root
bob ALL=(root) /usr/bin/systemctl restart nginx
```

Network example, an [access control list](05-access-control-lists.md) (ACL), which is an ordered list of allow and deny rules:

```text
! Cisco IOS: only these two flows may enter the database network
ip access-list extended TO-DB
 permit tcp host 10.0.20.10 host 10.0.30.10 eq 5432
 permit tcp host 10.0.99.5 host 10.0.30.10 eq 22
 deny ip any any log
interface GigabitEthernet0/1
 ip access-group TO-DB out
```

Related ideas:

- **Default deny:** block everything, then allow only what is needed. Cisco ACLs end with a hidden "deny all" rule.
- **Need to know:** you see data only if your job needs it, even if you have a high rank.
- **Separation of duties:** one person should not control a whole sensitive task. The person who writes a change is not the person who approves it.
- **Just-in-time access:** admin rights are granted for a short time, then removed.
- **Privilege creep:** people change jobs and keep their old rights. Regular access reviews fix this.

### Defence in Depth

**Defence in depth** means using several independent layers of protection, so that one failure does not cause total loss.

Think of an old castle: a moat, a wall, gates, guards, then a strong inner tower. Where the analogy stops: a castle trusts everyone inside the wall. Modern networks cannot do this, because staff work from home, data lives in the cloud, and attackers often get inside. That is why [zero trust](09-zero-trust-and-segmentation.md) exists.

```text
  Layer                 Example protection
  --------------------  ------------------------------------
  People and policy     Training, clear rules, incident plan
  Physical              Locked rooms, badge access
  Network edge          Firewall, DDoS filtering
  Internal network      VLANs, ACLs, IDS/IPS
  Host                  Patching, endpoint protection
  Application           Input checks, secure login, MFA
  Data                  Encryption, backups
```

A **VLAN** is a logical group of ports on a switch that acts like a separate network. An **IDS** watches traffic and raises alerts, and an **IPS** can also block it. See [IDS, IPS and SIEM](06-ids-and-ips.md). **Endpoint protection** is security software on a laptop or server.

Good layers are independent. If the firewall, the VPN and the servers all use the same admin password, you have three walls with one key.

### Attack Surface

The **attack surface** is the total of all the places where an attacker could try to get in or take data out. A house with 20 windows has more attack surface than a house with 2. Where the analogy stops: a house rarely gains new windows, but a network gains them each time someone starts a test server.

It has three parts:

- **Digital:** open ports, web apps, APIs, VPN gateways, admin pages, cloud storage.
- **Physical:** unlocked server rooms, spare network sockets, lost laptops.
- **Human:** staff who can be tricked. Tricking people is called **social engineering**, and a fake email is called **phishing**. See [Common Network Attacks](03-common-network-attacks.md).

An **attack vector** is one specific path that an attacker uses, such as a phishing email or an exposed RDP port. The attack surface is all the possible paths. The vector is the one chosen.

You cannot defend what you do not know about, so the first step is to list what you have. **Hardening** means reducing the attack surface:

1. Find what is exposed.
2. Remove services that nobody needs.
3. Limit who can reach the rest.
4. Patch what stays.
5. Change default passwords and settings.
6. Check again, because networks change.

Hardening a Cisco router means removing things like this:

```text
! Cisco IOS: turn off the web admin pages and allow only SSH for login
no ip http server
no ip http secure-server
line vty 0 4
 transport input ssh
```

Many tools for finding what is exposed are covered in [Port and Connection Tools](../12-troubleshooting-and-tools/05-port-and-connection-tools.md) and [nmap, iperf and Other Tools](../12-troubleshooting-and-tools/08-nmap-iperf-and-more.md).

### Types of Attacker

Attackers differ in motive, skill and patience. An attacker is also called a **threat actor**. Your defence depends on who is likely to target you. A small shop faces different attackers from a defence company.

| Attacker | Motive | Skill | Example behaviour |
|---|---|---|---|
| Script kiddie | Fun, fame | Low | Runs ready-made tools without understanding them |
| Hacktivist | Political or social cause | Low to medium | Takes a website offline, leaks documents |
| Organised criminals | Money | Medium to high | Ransomware, stolen card data, fraud |
| Nation-state (APT) | Spying, sabotage | Very high | Stays hidden for months, uses zero-days |
| Insider | Money, anger, or just a mistake | Varies | Copies customer data, or emails it by accident |
| Competitor | Business advantage | Varies | Steals designs or price lists |

- **Ransomware** is malicious software that encrypts your files and demands payment. Many groups also steal the data and threaten to publish it.
- **APT** means Advanced Persistent Threat: a skilled, funded group that stays in a network for a long time.
- An **insider** already has access, so firewalls do not stop them. Least privilege and accounting matter most here. Note that many insider incidents are accidents, not bad intent.

Real examples: the Mirai botnet (2016) took over IoT devices that used default passwords, then launched huge DDoS attacks. The Colonial Pipeline ransomware attack (2021) caused a US fuel pipeline to stop for days.

You will also hear hat colours:

- **White hat:** an ethical hacker who has permission.
- **Black hat:** a criminal attacker.
- **Grey hat:** breaks in without permission, but without clear harmful intent. This is usually still illegal.

Inside companies, the **red team** plays the attacker and the **blue team** defends. A **penetration test** is a paid, approved attack that finds weaknesses before real attackers do.

### Security Controls

A **security control** is anything that reduces risk. It can be a device, a rule or a process. Controls are sorted in two ways: what kind of thing they are, and what job they do.

By kind:

| Kind | Also called | Examples |
|---|---|---|
| Administrative | Managerial | Policies, training, background checks |
| Technical | Logical | Firewalls, encryption, MFA, ACLs |
| Physical | | Locks, cameras, badge readers, guards |

By job:

| Job | What it does | Examples |
|---|---|---|
| Preventive | Stops the bad event | Firewall rule, MFA, locked door |
| Detective | Notices the bad event | IDS, log review, camera footage |
| Corrective | Fixes damage or the cause | Patching, removing malware |
| Recovery | Returns to normal service | Restoring from backup |
| Deterrent | Discourages the attacker | Warning banner, visible camera |
| Compensating | Substitutes when the normal control is impossible | Isolating an old machine that cannot be patched |

Think of a house. A lock prevents, an alarm detects, a "guarded by cameras" sign deters, and home insurance helps you recover. Where the analogy stops: one network control can do several jobs. A visible camera both deters and detects.

The fuller truth: names vary by framework. NIST and CompTIA group the kinds differently, and some lists add "operational" or "directive" controls. Know the idea, and do not argue over labels.

A good design has a mix. If you have only preventive controls, you will not notice when one fails. Plan for failure: assume that something will get through, then make sure you detect it and recover.

## Example

Let us secure a small online shop, `shop.example.com`. All the ideas above appear here.

```text
                     Internet
                        |
              203.0.113.10 (public address)
                        |
                  [ Firewall ]
                        |
                  [ Router R1 ]
                 /      |      \
             Gi0/0    Gi0/1    Gi0/2
               |        |        |
            VLAN 20  VLAN 30  VLAN 99
               |        |        |
              web      db     jump host

  web        = 10.0.20.10
  db         = 10.0.30.10
  jump host  = 10.0.99.5
```

A **jump host** is one hardened server that admins must log in to first, before they reach other servers.

**Step 1: Name the assets and the CIA goal.**

| Asset | Most important goal | Why |
|---|---|---|
| Customer database | Confidentiality | It holds names and addresses |
| Order records | Integrity | A changed price or address costs money |
| Website | Availability | No website means no sales |

**Step 2: Measure the attack surface from outside.** Scan only systems that you own or have written permission to test. Here, a test machine outside the network scans the public address.

```bash
# Linux: scan five TCP ports on the shop's public address
nmap -Pn -p 21,22,80,443,5432 203.0.113.10
```

```text
Nmap scan report for 203.0.113.10
Host is up (0.012s latency).

PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
5432/tcp open  postgresql
```

Only 80 and 443 should be public. FTP sends passwords in clear text. Port 5432 is the database, and a contractor left it open months ago.

**Step 3: Look at the server itself.**

```bash
# Linux: list listening TCP ports on the web server (10.0.20.10)
sudo ss -tlnp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      32           0.0.0.0:21      0.0.0.0:* "vsftpd",pid=733
LISTEN 0      128          0.0.0.0:22      0.0.0.0:* "sshd",pid=640
LISTEN 0      511          0.0.0.0:80      0.0.0.0:* "nginx",pid=812
LISTEN 0      511          0.0.0.0:443     0.0.0.0:* "nginx",pid=812
```

(The Process column is shortened here. Real output wraps it as `users:((...))`.)

The same check on a Windows server:

```powershell
# Windows: list listening TCP ports
Get-NetTCPConnection -State Listen |
  Select-Object LocalAddress, LocalPort, OwningProcess |
  Sort-Object LocalPort
```

```text
LocalAddress LocalPort OwningProcess
------------ --------- -------------
0.0.0.0            135           948
0.0.0.0            445             4
0.0.0.0           3389          1120
```

**Step 4: Check the logs. This is the accounting step.**

```bash
# Debian, Ubuntu: auth.log. RHEL family: /var/log/secure.
sudo grep "Failed password" /var/log/auth.log | tail -n 2
sudo grep -c "Failed password" /var/log/auth.log
```

```text
Oct 30 09:14:22 web01 sshd[2114]: Failed password for root from 203.0.113.50 port 51122 ssh2
Oct 30 09:14:25 web01 sshd[2116]: Failed password for admin from 203.0.113.50 port 51130 ssh2
18342
```

This proves that the threat is real: password guessing, or **brute force**, is already happening.

**Step 5: Rate the risks.** Here each value is from 1 (low) to 5 (high). This scale is a common simple method, not a standard.

| Threat | Vulnerability | Likelihood | Impact | Risk |
|---|---|---|---|---|
| Bots guess the SSH password | SSH open to the internet, passwords allowed | 4 | 4 | 16 |
| Criminal steals customer data | Database port is public | 3 | 5 | 15 |
| Ransomware locks the servers | Backups stored on the same network | 2 | 5 | 10 |
| DDoS during a sale | One server, no filtering | 3 | 3 | 9 |

**Step 6: Apply controls, highest risk first.**

```bash
# Ubuntu, on 10.0.20.10. Run from the jump host to avoid a lockout.
sudo systemctl disable --now vsftpd
sudo ufw default deny incoming
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 10.0.99.5 to any port 22 proto tcp
sudo ufw enable
```

Set these two lines in `/etc/ssh/sshd_config`, then reload the service. The service is `ssh` on Debian and Ubuntu, and `sshd` on RHEL family systems.

```text
PasswordAuthentication no
PermitRootLogin no
```

On the firewall, remove the forwards for ports 21, 22 and 5432. On R1, apply the `TO-DB` ACL shown earlier, outbound on Gi0/1 (the database network).

| Control | Risk it treats | Kind | Job |
|---|---|---|---|
| Firewall forwards only 80 and 443 | SSH, database exposure | Technical | Preventive |
| SSH keys only, through the jump host | Password guessing | Technical | Preventive |
| `TO-DB` ACL with `log` | Database exposure | Technical | Preventive, detective |
| Logs sent to a central server | All | Technical | Detective |
| Offline backups, restore tested monthly | Ransomware | Technical | Recovery |
| Written rule: no public ports without approval | Future mistakes | Administrative | Preventive |
| DDoS filtering from the provider | DDoS | Technical | Preventive |

The last row is partly a **transfer** of risk to the provider. Backups are also defence in depth: if prevention fails, you can still recover.

**Step 7: Verify.** Always test from the outside again.

```text
Nmap scan report for 203.0.113.10
Host is up (0.012s latency).

PORT     STATE    SERVICE
21/tcp   filtered ftp
22/tcp   filtered ssh
80/tcp   open     http
443/tcp  open     https
5432/tcp filtered postgresql
```

`filtered` means the firewall dropped the probe. A log entry now shows normal, traceable admin access:

```text
Oct 30 11:02:41 web01 sshd[3120]: Accepted publickey for alice from 10.0.99.5 port 49822 ssh2
```

**A small integrity example.** A hash shows that a file changed. Here one digit changes, and the whole hash changes.

```bash
# Linux
printf 'Pay Bob 100\n' > order.txt
sha256sum order.txt
printf 'Pay Bob 900\n' > order.txt
sha256sum order.txt
```

```text
ac0cdf00d662ca32951a1a3a59d633804413ef2f1d088a682bb0a22dacde51d5  order.txt
8675108ec20a5879066b72f9b76e32848983b5778d97f627df8acdcef721efbc  order.txt
```

```powershell
# Windows
Get-FileHash .\order.txt -Algorithm SHA256
```

## Going Deeper

**Fail-open and fail-closed.** What should a security device do when it breaks? **Fail-closed** blocks all traffic. It protects confidentiality but stops the business. **Fail-open** lets traffic pass. It keeps the business running but drops the protection. A firewall normally fails closed. An inline IPS is often set to fail open, because an IPS failure should not cut the whole network. This choice is a CIA trade-off, so decide it on purpose.

**Threat modelling.** This means finding threats during design, not after an attack. A well-known method is STRIDE: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. Elevation of privilege means gaining more rights than you should have.

**The Cyber Kill Chain.** Lockheed Martin described seven stages of an attack: reconnaissance, weaponization, delivery, exploitation, installation, command and control, and actions on objectives. Stop any stage and the attack fails. Today, **MITRE ATT&CK** is a widely used, detailed public list of real attacker techniques.

**Frameworks and rules.** You should know these names:

- **NIST Cybersecurity Framework 2.0:** six functions: Govern, Identify, Protect, Detect, Respond, Recover.
- **ISO/IEC 27001:** an international standard for managing information security.
- **PCI DSS:** rules for handling payment card data.
- **HIPAA:** US rules for health data.
- **GDPR:** EU rules for personal data.

## Common Mistakes

- **Using "threat", "vulnerability" and "risk" as if they mean the same thing.** The attacker is the threat, the open port is the vulnerability, and the risk is the likelihood and cost of what could happen.
- **Mixing up authentication and authorization.** Login proves who you are. Permissions decide what you may do. In HTTP, 401 means "we do not know who you are", and 403 means "we know you, and you may not do this".
- **Thinking encryption solves everything.** Encryption mainly protects confidentiality. An HTTPS site can still serve malware, and it can still go offline.
- **Thinking a firewall makes you secure.** A firewall cannot stop phishing, a stolen password, or an insider who is already allowed through.
- **Trusting everything inside the network.** Attackers often start inside after a phishing email, then move sideways.
- **Calling two passwords MFA.** They are the same factor. MFA needs different kinds of proof.
- **Thinking risk can reach zero.** You reduce it to a level the business accepts. Some residual risk always stays.
- **Treating availability as "not security".** A DDoS attack and a lost backup are both security failures.
- **Thinking defence in depth means buying more products.** Layers must fail independently. Five tools that nobody watches add little.
- **Relying on hiding.** Moving SSH from port 22 to 2222 reduces log noise, but a port scan still finds it. It is not a real control by itself.
- **Forgetting access after it is granted.** Rights pile up over time unless you review them.
- **Ignoring backups until they are needed.** A backup that you have never restored is only a hope.

## Interview Questions

**Q: What is the CIA triad?**
A: It is the three main goals of security. Confidentiality means only allowed people can read the data, for example through encryption. Integrity means the data is not changed without detection, for example through hashes. Availability means the service works when needed, for example through redundancy and backups.

**Q: Which part of CIA does each attack hit: DDoS, ransomware, a man-in-the-middle attack?**
A: DDoS hits availability. Ransomware hits availability, and also confidentiality if the attackers steal the data first. A man-in-the-middle attack, where the attacker secretly sits between two parties, hits confidentiality when they read traffic and integrity when they change it.

**Q: What is the difference between a threat, a vulnerability and a risk?**
A: A threat is something that can cause harm, like an attacker. A vulnerability is a weakness, like an unpatched server. Risk is the likelihood that the threat uses the weakness, multiplied by the impact. A burglar is the threat, an unlocked door is the vulnerability, and the chance and cost of a theft is the risk.

**Q: How can you deal with a risk?**
A: There are four options. Reduce it with a control, avoid it by stopping the activity, transfer it, for example with insurance, or accept it on purpose. Some residual risk always remains, and the business owner should approve it.

**Q: What is the difference between authentication and authorization?**
A: Authentication proves who you are, for example with a password and a phone code. Authorization decides what you may do after that, for example read-only access to a switch. Authentication comes first.

**Q: What is AAA, and which protocols support it?**
A: AAA is Authentication, Authorization and Accounting: who you are, what you may do, and a record of what you did. RADIUS uses UDP 1812 and 1813 and is common for Wi-Fi and VPN users. TACACS+ uses TCP 49, encrypts the whole message body, and is common for admin login to network devices.

**Q: What is least privilege, and how do you apply it on a network?**
A: Give each user, device and program only the access needed for its job, and only for as long as needed. On a network, I use default deny, then allow specific flows. For example, only the web server can reach the database on port 5432. Admins log in through a jump host. This limits the damage if an account is stolen.

**Q: What is defence in depth?**
A: It means using several independent layers, so that one failure does not lead to total loss. For a web app, I would use a firewall, network segmentation, patching, MFA, encryption and backups. If an attacker gets past the firewall, the other layers still protect the data, and logs help us detect the attacker.

**Q: What is an attack surface, and how is it different from an attack vector?**
A: The attack surface is all the points where an attacker could try to enter, such as open ports, web apps, and staff who can be tricked. An attack vector is one specific path, such as a phishing email. I reduce the surface by closing unused ports, removing services, patching, and restricting admin access.

**Q: What types of attackers exist?**
A: Script kiddies use ready-made tools. Hacktivists act for a cause. Organised criminals want money, often through ransomware. Nation-state groups spy or cause sabotage, and they stay hidden for a long time. Insiders already have access, and they may act on purpose or by mistake. Knowing the likely attacker helps you choose your controls.

**Q: What are the types of security controls?**
A: By kind, they are administrative, technical and physical, for example a policy, a firewall and a door lock. By job, they are preventive, detective, corrective, recovery, deterrent and compensating. A firewall rule prevents, an IDS detects, and a backup recovers. A good design mixes them, because prevention alone will not catch every failure.

**Q: You get a new server with a public IP address. What do you do first?**
A: First I find out what it runs and what is exposed, with `ss` on the host and `nmap` from outside. I remove unused services and allow only the needed ports with default deny. I patch it, change default credentials, and use SSH keys with no root login. I restrict admin access to a VPN or jump host. Then I send logs to a central server, set up tested backups, and scan again from outside to verify.

## Quick Recap

- The CIA triad is Confidentiality, Integrity and Availability. Use it to judge any attack or defence.
- A threat is what can harm you, a vulnerability is the weakness, and risk is likelihood times impact. You reduce, avoid, transfer or accept risk.
- AAA is Authentication (who), Authorization (what) and Accounting (record). MFA needs different kinds of proof.
- Least privilege gives only the access needed, and default deny blocks the rest.
- Defence in depth uses independent layers, so one failure is not fatal.
- Attack surface is all possible entry points. Hardening makes it smaller.
- Attackers differ in motive and skill, and controls differ by kind and job. Plan to prevent, detect and recover.
