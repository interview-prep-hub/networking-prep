# IDS, IPS and SIEM

**Level:** Intermediate | **Prerequisites:** [Network Security Basics](01-security-basics.md), [Common Network Attacks](03-common-network-attacks.md), [Firewalls](04-firewalls.md)

## In One Line

An IDS watches traffic and raises an alarm, an IPS sits in the traffic path and blocks attacks, and a SIEM collects logs from everywhere so that people can see the whole attack.

## Why It Matters

A firewall mostly decides by address, port and connection state. Port 443 to your web server must stay open, so an attack sent to port 443 passes straight through. Something must look at what the traffic actually does.

Attacks also leave small traces in many places. One failed login means nothing. Five hundred failed logins, then one success from the same address, then a large upload at 3 a.m., mean a lot. Only a system that sees all these logs together can connect them.

Interviewers ask about this topic for three reasons:

- It tests whether you understand trade-offs. An IPS can stop an attack, but it can also block real customers.
- The words are often mixed up: IDS, IPS, SIEM, SOC, SOAR, EDR.
- Security roles, and many network roles, work with these tools every day.

## Simple Explanation

Think of a large office building.

- **IDS (Intrusion Detection System):** a security camera with a motion alarm. It sees a stranger and tells someone. It cannot stop him.
- **IPS (Intrusion Prevention System):** a guard standing at the door. He checks each person and can refuse entry. If he is slow, a queue forms. If he stops a real employee by mistake, that employee cannot get to work.
- **SIEM (Security Information and Event Management):** the control room. It receives the camera feeds, the door badge records and the alarm logs, all on one screen.
- **SOC (Security Operations Centre):** the people working in the control room.
- **Honeypot:** a fake safe left in an empty room. No staff member should ever touch it, so anyone who does is suspicious.

Where the analogy stops being true: a camera sees everything in its view, but a network sensor cannot read encrypted traffic. Also, a guard judges each person with common sense. An IPS only follows rules and models, so it can be tricked in ways a person would not be.

## How It Works

### 1. Terms You Need First

- **Packet:** one small piece of network data, with addresses and content.
- **Intrusion:** someone gaining, or trying to gain, access they should not have.
- **Sensor:** the device or software that looks at traffic. Snort and Suricata are sensor software.
- **Rule or signature:** a written pattern that describes something bad.
- **Alert:** a message from a sensor saying "this looked bad".
- **Log:** a text record of something that happened, such as a login.
- **Event:** one single logged thing. An alert is an event that someone should look at.

### 2. IDS vs IPS

| Feature | IDS | IPS |
|---|---|---|
| Main job | Detect and report | Detect and block |
| Position | Beside the traffic path | In the traffic path |
| Sees | A copy of the traffic | The real traffic |
| Can stop the first packet | No | Yes |
| If it fails | Network keeps working, you lose visibility | Traffic may stop, depending on design |
| Effect on latency | None | Adds some delay |
| Cost of a false alarm | An analyst wastes time | A real user or service is blocked |

Fuller truth: an IDS is not fully passive in every product. Some can send a TCP reset (a packet that tells both sides to close the connection) or ask a firewall to block an address. This is called active response. It is a race, because the bad packet has already arrived.

Most vendors now sell one engine that can run in either mode. Standards documents such as NIST SP 800-94 call the whole family **IDPS** (Intrusion Detection and Prevention Systems). The same software is an IDS or an IPS depending on where you put it and what actions you allow.

### 3. Out of Band vs Inline

**Out of band** means the sensor is not in the path of the traffic. It gets a copy. **Inline** means the traffic must pass through the sensor to reach its target.

```text
 Out of band (IDS)

 Client ------> Switch ------> Server
                  |
                  | copy of the traffic (SPAN or TAP)
                  v
               [ IDS ] ----> alert to analyst or SIEM


 Inline (IPS)

 Client ------> [ IPS ] ------> Server
                   |
                   +--> bad packet dropped, good packet forwarded
```

There are two common ways to give an out-of-band sensor its copy.

**SPAN port** (Switched Port Analyzer, also called port mirroring): the switch copies frames from some ports to one chosen port.

**Network TAP** (Test Access Point): a small hardware device placed on the cable. It passes the traffic through and gives monitoring ports a copy.

| Feature | SPAN port | Network TAP |
|---|---|---|
| Cost | Free, it is a switch feature | Extra hardware |
| Setup | Configuration only | Must break the link once to install |
| Under heavy load | Copies can be dropped | Designed to copy everything on the link |
| Bad frames | Often not copied | Passive TAPs copy what is on the wire |
| Risk to the switch | Uses switch resources | No switch involved |

A SPAN can also overflow. Mirror two 1 Gbps ports in both directions to one 1 Gbps port and you may need up to 4 Gbps of copies on a 1 Gbps link. The extra is lost.

**What happens when an inline device fails?** This is a favourite interview question.

- **Fail-open:** traffic keeps flowing without inspection. The business stays up, security is reduced.
- **Fail-closed:** traffic stops. Security stays strict, the business goes down.

Inline sensors often use a **bypass** network card or bypass switch. If the sensor dies, the hardware connects the two links directly. High-availability pairs are another common fix. A bank's internet banking link and a hospital's clinical network may choose differently, because the cost of downtime and the cost of a breach differ.

### 4. Where to Put Sensors

**North-south** traffic enters or leaves your network. **East-west** traffic moves between internal systems. Attackers who are already inside move east-west, so an edge sensor alone never sees them.

```text
 Internet
    |
 [A] outside the firewall: sees every attempt, very noisy
    |
 Firewall ---- [B] DMZ: watches public servers
    |
 [C] inside the firewall: sees only what got through
    |
 Core switch ---- [D] data centre: east-west traffic
    |
 Users ---------- [E] user VLANs: infected laptops calling out
```

A **DMZ** (demilitarised zone) is a network segment for public servers, separated from the internal LAN. Position C is the usual first choice, because alerts there are about traffic that really passed the firewall.

### 5. How Detection Works

#### Signature-based detection

The sensor compares traffic to a list of known bad patterns. It works like antivirus, or like a guard with photos of wanted people.

A signature could say: "TCP traffic to port 80, HTTP request, the URI contains `/etc/passwd`". (A **URI** is the path part of a web address.)

- Good: precise, fast and easy to explain. The alert names the exact rule.
- Bad: it cannot detect what it has no rule for. A **zero-day** is an attack that is not yet publicly known, so it usually has no signature.

#### Anomaly-based detection

The sensor first learns a **baseline**, meaning what normal looks like. Then it flags things that differ from normal.

Example: a laptop sends about 200 MB of DNS traffic a month. Today it sends 5 GB in one hour. No rule names this, but it may be data hidden inside DNS queries.

- Good: it can catch new attacks.
- Bad: "unusual" does not always mean "bad". A new backup job looks unusual too. Alerts are harder to explain, and the baseline can learn bad behaviour as normal if an attacker was already present.

#### Protocol analysis

NIST lists a third method called **stateful protocol analysis**. The sensor knows how a protocol should behave, and it flags traffic that breaks those rules, such as an FTP command sent before login. It differs from anomaly detection because the model comes from the protocol's specification, not from your traffic.

| Feature | Signature | Anomaly |
|---|---|---|
| Detects | Known attacks | Unusual behaviour |
| Zero-day attacks | Usually misses | May catch |
| False positives | Lower when rules are tuned | Usually higher |
| Needs | Rule updates | Training time and a clean baseline |
| Explaining an alert | Easy | Harder |

Real products use both. Many also add **threat intelligence**, which is a feed of known bad addresses, domains and file hashes.

### 6. True, False and Tuning

Every decision by a sensor falls into one of four boxes.

```text
                    Was it really an attack?
                    YES                 NO
               +-------------------+-------------------+
 Alert fired   | True positive     | False positive    |
               | (good catch)      | (false alarm)     |
               +-------------------+-------------------+
 No alert      | False negative    | True negative     |
               | (missed attack)   | (quiet, correct)  |
               +-------------------+-------------------+
```

A false negative is more dangerous, because nobody knows it happened. A false positive looks harmless, but thousands of them cause **alert fatigue**: people stop reading alerts, and the real one is ignored.

Why are there so many false alarms? Because real attacks are rare compared to normal traffic. Look at these numbers:

- 1,000,000 events per day, of which 10 are real attacks.
- The sensor catches 99% of attacks, so about 10 true alerts.
- It wrongly flags only 0.1% of normal events, which gives about 1,000 false alerts.

That is about 1,010 alerts, and only about 1 in 100 is real. A tool that sounds very accurate still produces mostly noise. This is known as the **base rate problem**.

**Tuning** means adjusting the system to your network. Common steps:

1. Define your own networks correctly, such as `HOME_NET` in Snort and Suricata.
2. Turn off rules for software you do not run. If you have no Oracle servers, Oracle rules only add noise.
3. Make noisy rules more exact.
4. Suppress alerts from known sources, such as your own vulnerability scanner.
5. Add **thresholds**, for example alert only after 5 hits in 60 seconds.
6. Run new rules in alert-only mode first. Move them to block mode after you trust them.

Tuning has a cost too. Tune too hard and false positives turn into false negatives.

### 7. NIDS vs HIDS

A **NIDS** (Network-based IDS) watches network traffic. A **HIDS** (Host-based IDS) is an agent, meaning software, running on one computer. It watches that computer's logs, files and processes.

| Feature | NIDS | HIDS |
|---|---|---|
| Watches | Packets on a network segment | One host: logs, files, processes |
| Coverage | Many hosts at once | Only hosts with the agent |
| Encrypted traffic | Sees metadata only | Sees activity after decryption |
| Knows if the attack worked | Often not | Often yes |
| Effect on hosts | None | Uses host CPU and memory |
| Examples | Snort, Suricata | Wazuh, OSSEC, Tripwire |

A HIDS commonly does **file integrity monitoring**: it records hashes of important files, then alerts when one changes. A **hash** is a short fingerprint of a file's content.

The raw data a HIDS reads looks like this:

```bash
# Linux (Debian and Ubuntu; RHEL uses /var/log/secure)
sudo grep "Failed password" /var/log/auth.log | tail -n 3
```

```powershell
# Windows, as Administrator. Event 4625 is a failed logon.
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 3
```

Today, **EDR** (Endpoint Detection and Response) products such as Microsoft Defender for Endpoint and CrowdStrike Falcon have largely taken over the HIDS role on company laptops and servers. They add process tracking and remote actions, such as isolating a host from the network.

Other sensor types exist as well. A **WIPS** (Wireless IPS) watches Wi-Fi for rogue access points. See [Enterprise Wi-Fi](../08-wireless-and-mobile/03-enterprise-wifi.md).

### 8. Snort and Suricata

These are the two best-known open-source network IDS and IPS engines. Both are licensed under the GPL version 2.

**Snort** was created by Martin Roesch in 1998. His company Sourcefire was bought by Cisco in 2013. Cisco's Secure Firewall products use Snort as their inspection engine. Snort 3, a major rewrite, became generally available in 2021.

**Suricata** is developed by the Open Information Security Foundation (OISF). Its first stable release came in 2010. It was built with multi-threading from the start.

| Feature | Snort 3 | Suricata |
|---|---|---|
| Threads | Multi-threaded (Snort 2 ran one packet thread per process) | Multi-threaded |
| Main config file | `snort.lua` | `suricata.yaml` |
| Rule language | Snort rules | Snort-style rules, with its own extensions |
| Main output | Fast alerts, JSON and others | EVE JSON, one stream for all events |
| Protocol logging | More limited | Rich: HTTP, DNS, TLS, flows and more |
| Common rule sets | Snort Community, Talos subscriber rules | ET Open, ET Pro |

**ET** means Emerging Threats, a rule set now run by Proofpoint. **Talos** is Cisco's threat research group.

The two rule languages share a history, but a rule from one does not always load in the other, so test before you copy.

**Zeek** (formerly Bro) is often mentioned here too. It is not mainly a signature engine. It turns traffic into detailed logs, such as `conn.log`, `dns.log` and `http.log`, and has its own scripting language. Many teams run Zeek beside Suricata. Security Onion is a free Linux platform that bundles such tools.

#### Anatomy of a rule

```text
alert tcp any any -> $HOME_NET 22 (msg:"SSH"; sid:1000002; rev:1;)
|     |   |   |   |  |         |  |
|     |   |   |   |  |         |  +-- options, inside the brackets
|     |   |   |   |  |         +-- destination port
|     |   |   |   |  +-- destination addresses
|     |   |   |   +-- direction
|     |   |   +-- source port
|     |   +-- source addresses
|     +-- protocol
+-- action
```

The rule has a **header** (who and where) and **options** (what to look for and what to say).

The main actions are `alert` (log it), `drop` (block and log), `reject` (block and tell the sender) and `pass` (stop checking this packet). In Suricata, `drop` only blocks when the engine runs in IPS mode. In IDS mode it just logs.

### 9. SIEM

A **SIEM** does not sit in the traffic path and does not block packets. It works on logs.

```text
 Firewall logs ----+
 IDS alerts -------+
 DNS and DHCP -----+--> Collect --> Parse --> Normalise --> Store
 Windows, Linux ---+                                          |
 Cloud logs -------+                                          v
                                            Correlate --> Alert --> SOC
                                                |
                                                +--> Search, dashboards
```

The steps:

1. **Collect.** Devices send logs by **syslog** (the standard log format and transport, RFC 5424) or through an installed agent, or the SIEM pulls them through an API.
2. **Parse.** The SIEM splits each log line into named fields.
3. **Normalise.** Different products use different words for the same thing. The SIEM maps them to one field name.
4. **Enrich.** It adds context: who owns this host, is this address on a threat feed, which country does it belong to.
5. **Correlate.** It applies rules across many sources and across time.
6. **Alert.** It creates a case for a human.
7. **Store.** It keeps logs for search, investigation and legal needs.

Here is normalisation in action. These two lines come from different systems:

```text
Cisco ASA firewall:
%ASA-4-106023: Deny tcp src outside:203.0.113.50/51544
dst inside:10.0.20.10/22 by access-group "OUTSIDE_IN" [0x0, 0x0]

Linux SSH server:
sshd[2211]: Failed password for root from 203.0.113.50 port 51602 ssh2
```

Both become `src_ip=203.0.113.50`. Now one search finds both.

**Correlation** turns weak signals into a strong one:

```text
IF   more than 20 failed logins from one source in 10 minutes
AND  one successful login from the same source to the same account
THEN raise a HIGH alert: possible password attack succeeded
```

Things that make a SIEM useful:

- **Time.** All devices need correct clocks, usually through NTP. If the firewall is 7 minutes off, the story falls apart. See [Network Management Protocols](../07-application-protocols/11-network-management-protocols.md).
- **DHCP logs.** An alert shows 10.0.60.25. DHCP logs tell you which laptop had that address at that time.
- **Retention.** Logs are costly to store. Rules such as PCI DSS (the card payment standard) require 12 months of audit logs, with the last 3 months available at once.
- **Good sources.** A SIEM can only find what it receives. No DNS logs means no DNS detection.

Common SIEM products include Splunk, Microsoft Sentinel, IBM QRadar, Elastic Security, Google Security Operations and the open-source Wazuh. Pricing is often based on data volume per day or **EPS** (events per second), so it varies a lot by vendor.

Sending logs is simple. These examples send to a SIEM collector at 10.0.30.50.

```text
! Cisco IOS
Router(config)# service timestamps log datetime msec
Router(config)# ntp server 10.0.30.1
Router(config)# logging host 10.0.30.50
Router(config)# logging trap informational
```

```bash
# Linux: /etc/rsyslog.d/90-siem.conf  (@@ means TCP, @ means UDP)
echo '*.* @@10.0.30.50:514' | sudo tee /etc/rsyslog.d/90-siem.conf
sudo systemctl restart rsyslog
```

```powershell
# Windows has no built-in syslog sender.
# Use an agent, or Windows Event Forwarding to a collector server:
winrm quickconfig    # on each source machine
wecutil qc           # on the collector machine
```

Plain syslog usually runs on UDP 514, which has no delivery guarantee and no encryption. Syslog over TLS uses TCP 6514 (RFC 5425). Use it when logs cross untrusted networks.

### 10. SOC Workflow

A **SOC** is the team, plus process and tools, that watches for attacks and responds. Many teams work in tiers.

| Role | Main work |
|---|---|
| Tier 1 analyst | Watches the queue, triages alerts, closes false positives, escalates |
| Tier 2 analyst | Deep investigation, finds the scope, starts containment |
| Tier 3 analyst | Advanced cases, malware analysis, threat hunting |
| Detection engineer | Writes and tunes detection rules |
| SOC manager | Runs the team, metrics and reporting |

Tier names differ between companies, and some modern teams drop tiers altogether.

```text
 SIEM alert
     |
     v
 1. Triage ---- false positive? ---> close, note why, tune rule
     | real or unsure
     v
 2. Investigate: how far did it go?
     |
     v
 3. Contain: stop it spreading
     |
     v
 4. Eradicate: remove the cause
     |
     v
 5. Recover: back to normal, watch closely
     |
     v
 6. Lessons learned: new rules, new fixes ---> back to detection
```

1. **Triage** means quickly sorting by importance, as in a hospital emergency room. The analyst checks the alert, the target's importance and the source's reputation. A **runbook** or **playbook** is the written steps for a known alert type.
2. **Investigate.** Which hosts and accounts are involved? When did it start? The analyst searches the SIEM and may pull a packet capture.
3. **Contain.** Block an address, isolate a host, disable an account.
4. **Eradicate.** Remove malware, close the hole, reset passwords.
5. **Recover.** Restore services and watch for a return.
6. **Lessons learned.** Write a report, improve detection, fix process gaps.

The phases above follow the classic lifecycle in NIST SP 800-61 Rev. 2. Rev. 3, published in 2025, maps incident response to the NIST Cybersecurity Framework 2.0 functions, but the classic steps are still what most interviewers expect.

Two common metrics are **MTTD** (mean time to detect) and **MTTR** (mean time to respond). Teams also tag alerts with **MITRE ATT&CK**, a public catalogue of attacker techniques, so everyone uses the same words.

**Threat hunting** is different from alert handling. A hunter does not wait for an alert. The hunter starts from an idea, such as "an attacker may be using DNS tunnels here", and searches the data for it.

### 11. Honeypots

A **honeypot** is a fake system built to attract attackers. It has no real business use, so no normal user should touch it. That gives it a very low false positive rate. Any login on it is worth a look.

```text
 Internet --> Firewall --+--> Web server (real)
                         |
                         +--> Honeypot 10.0.40.66 (fake, isolated VLAN)
                                 |            |
                                 |            +--> all events to SIEM
                                 X
                          no path to the real LAN
```

| Type | What it does | Risk | Example |
|---|---|---|---|
| Low interaction | Pretends to be a service, simple responses | Low | Dionaea, OpenCanary |
| Medium interaction | Fake shell with limited commands | Medium | Cowrie (SSH and Telnet) |
| High interaction | A real system that is watched closely | High | A real VM with heavy monitoring |

Related ideas:

- **Honeynet:** a whole network of honeypots.
- **Honeytoken:** a fake item, such as a fake password, fake database row or fake AWS key. If it is ever used, someone has been reading what they should not.
- **Deception platform:** a commercial product that places decoys across a network.

Honeypots are useful in two places. On the internet, they show which attacks are current. Inside the LAN, they catch attackers who are scanning for their next target, and there the alert is nearly always serious.

The main risk is that a compromised honeypot becomes a launching point for attacks. So isolate it, watch outbound traffic and rebuild it often. Legal rules on recording attackers vary by country, so check before you run one in production.

Where the "bait" idea stops being simple: a honeypot does not stop anyone. It only tells you someone is there.

## Example

A company runs a web shop. An attacker tries SQL injection, which means hiding database commands in a web request. This example follows one attack from packet to closed case.

### Topology

```text
 203.0.113.50 (attacker)
      |
 +-------------------+
 | Firewall          |  public 198.51.100.10
 | port 80 to web    |  inside 10.0.20.1
 +-------------------+
      |
 +-------------------+  Gi1/0/24 SPAN   +------------------+
 | Switch            |----------------->| eth1 (no IP)     |
 +-------------------+                  | Suricata sensor  |
      | Gi1/0/10                        | eth0 10.0.30.20  |
      |                                 +------------------+
 +-------------------+                           |
 | Web server        |                           | EVE JSON
 | 10.0.20.10        |---- nginx logs ----+      |
 +-------------------+                    v      v
                                       +------------------+
                                       | SIEM 10.0.30.50  |
                                       +------------------+
```

The capture port eth1 has no IP address, so attackers cannot reach the sensor through it. The sensor is managed through eth0.

### Step 1: Mirror the web server port

```text
! Cisco IOS
Switch(config)# monitor session 1 source interface Gi1/0/10 both
Switch(config)# monitor session 1 destination interface Gi1/0/24
Switch(config)# end
Switch# show monitor session 1
Session 1
---------
Type                   : Local Session
Source Ports           :
    Both               : Gi1/0/10
Destination Ports      : Gi1/0/24
    Encapsulation      : Native
          Ingress      : Disabled
```

Check that the sensor gets both directions. See [tcpdump](../12-troubleshooting-and-tools/06-tcpdump.md).

```bash
# Linux sensor
sudo tcpdump -ni eth1 -c 2 tcp port 80
```

```text
09:14:07.471204 IP 203.0.113.50.51544 > 10.0.20.10.80: Flags [S], seq 2871190034, win 64240, options [mss 1460,sackOK,TS val 410 ecr 0,nop,wscale 7], length 0
09:14:07.471987 IP 10.0.20.10.80 > 203.0.113.50.51544: Flags [S.], seq 918273645, ack 2871190035, win 65160, options [mss 1460,sackOK,TS val 99 ecr 410,nop,wscale 7], length 0
```

The firewall has already translated the public address 198.51.100.10 to 10.0.20.10. That is why the sensor sees the inside address. See [NAT and PAT](../04-ip-addressing/09-nat-and-pat.md).

### Step 2: Install Suricata and add a rule

```bash
# Ubuntu, using the OISF package repository
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update
sudo apt install suricata
sudo suricata-update      # downloads ET Open rules by default
```

Set these values in `/etc/suricata/suricata.yaml`:

```text
vars:
  address-groups:
    HOME_NET: "[10.0.0.0/8]"
    EXTERNAL_NET: "!$HOME_NET"

af-packet:
  - interface: eth1

rule-files:
  - suricata.rules
  - /etc/suricata/rules/local.rules
```

Create `/etc/suricata/rules/local.rules`. The backslashes let one rule span several lines:

```text
alert http $EXTERNAL_NET any -> $HOME_NET any ( \
    msg:"LOCAL Possible SQL injection in URI"; \
    flow:established,to_server; \
    http.uri; content:"union"; nocase; \
    content:"select"; nocase; distance:0; \
    classtype:web-application-attack; \
    sid:1000001; rev:1;)
```

| Part | Meaning |
|---|---|
| `alert http` | Log it, and match only traffic identified as HTTP |
| `flow:established,to_server` | Only client-to-server data on a finished TCP handshake |
| `http.uri` | Search the decoded URI, so `%20` is treated as a space |
| `content:"union"; nocase;` | Find "union" in any letter case |
| `distance:0` | "select" must come somewhere after "union" |
| `classtype` | Category of the attack, which sets default priority |
| `sid:1000001` | Unique rule ID. By convention, local rules use 1000000-1999999 |
| `rev:1` | Rule version. Raise it each time you edit the rule |

Test the config, then run the sensor:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo suricata -c /etc/suricata/suricata.yaml -i eth1
```

The test prints a line like the one below. The exact wording varies by version.

```text
Configuration provided was successfully loaded. Exiting.
```

### Step 3: The attack arrives

At 09:14 the attacker sends 37 requests in 45 seconds. Here is one of them. Run this kind of test only against your own lab.

```bash
curl "http://shop.example.com/products?id=1%20UNION%20SELECT%20password%20FROM%20users"
```

`/var/log/suricata/fast.log` shows a short alert (wrapped here to fit):

```text
10/30/2026-09:14:07.482113  [**] [1:1000001:1] LOCAL Possible SQL
injection in URI [**] [Classification: Web Application Attack]
[Priority: 1] {TCP} 203.0.113.50:51544 -> 10.0.20.10:80
```

The fields `[1:1000001:1]` mean generator 1, rule 1000001, revision 1.

`/var/log/suricata/eve.json` has the full version, trimmed here. This is what the SIEM reads.

```bash
sudo jq 'select(.event_type=="alert")' /var/log/suricata/eve.json
```

```text
{
  "timestamp": "2026-10-30T09:14:07.482113+0000",
  "in_iface": "eth1",
  "event_type": "alert",
  "src_ip": "203.0.113.50",
  "src_port": 51544,
  "dest_ip": "10.0.20.10",
  "dest_port": 80,
  "proto": "TCP",
  "app_proto": "http",
  "alert": {
    "action": "allowed",
    "signature_id": 1000001,
    "rev": 1,
    "signature": "LOCAL Possible SQL injection in URI",
    "category": "Web Application Attack",
    "severity": 1
  },
  "http": {
    "hostname": "shop.example.com",
    "url": "/products?id=1%20UNION%20SELECT%20password%20FROM%20users",
    "http_method": "GET",
    "http_user_agent": "curl/8.5.0"
  }
}
```

Note `"action": "allowed"`. This is an IDS, so the request reached the server.

### Step 4: The SIEM connects the dots

The web server's own log also reaches the SIEM (wrapped here to fit):

```text
203.0.113.50 - - [30/Oct/2026:09:14:07 +0000]
"GET /products?id=1%20UNION%20SELECT%20password%20FROM%20users HTTP/1.1"
200 48213 "-" "curl/8.5.0"
```

The status is 200 and the response is 48,213 bytes. This page is normally about 2,100 bytes. The server probably returned real data.

An analyst searches the SIEM. This is Splunk SPL syntax:

```text
index=ids event_type=alert alert.severity=1 earliest=-15m
| stats count values(dest_ip) AS targets BY src_ip, alert.signature
```

```text
src_ip         alert.signature                      count  targets
203.0.113.50   LOCAL Possible SQL injection in URI  37     10.0.20.10
```

### Step 5: The SOC handles the case

| Time | Who | Action |
|---|---|---|
| 09:14:07 | Suricata | First alert |
| 09:14:52 | Suricata | Last of 37 alerts |
| 09:15 | SIEM | Correlation rule fires: IDS alert plus HTTP 200 with a large response. Severity high |
| 09:21 | Tier 1 | Checks the alert. Not a known scanner. Target is a production shop. Escalates |
| 09:33 | Tier 2 | Finds this one source only. Maps it to ATT&CK T1190, Exploit Public-Facing Application |
| 09:36 | Tier 2 | Blocks 203.0.113.50 on the firewall and asks developers to take `/products` offline |
| 11:30 | Developers | Fix the code with parameterised queries |
| 12:00 | Incident lead | Resets exposed passwords. Starts a data breach review |
| Next week | Team | Lessons learned. New rule added. Web application firewall reviewed |

Blocking one address is only containment. The attacker can change address. The real fix is in the code.

### Step 6: A false positive, and tuning

Next day, the same rule fires for 198.51.100.24:

```text
"url": "/blog/credit-union-how-to-select-an-account"
```

The URI contains "union" and later "select". It is an ordinary reader on a blog post. The response is normal, and this source sends nothing else suspicious. The analyst closes it as a false positive, and the detection engineer tightens the rule:

```text
alert http $EXTERNAL_NET any -> $HOME_NET any ( \
    msg:"LOCAL Possible SQL injection in URI"; \
    flow:established,to_server; \
    http.uri; content:"union"; nocase; \
    content:"select"; nocase; distance:0; within:12; \
    classtype:web-application-attack; \
    sid:1000001; rev:2;)
```

`within:12` means "select" must end within 12 bytes after "union". The text ` all select` is 11 bytes, so it still matches. The text `-how-to-select` is 14 bytes, so it no longer matches.

The company's own weekly scanner at 10.0.5.20 also fires this rule. A suppression goes in the file named by `threshold-file` in `suricata.yaml`:

```text
suppress gen_id 1, sig_id 1000001, track by_src, ip 10.0.5.20
```

Apply the changes without a restart:

```bash
sudo suricatasc -c reload-rules
```

### Step 7: Turn it into an IPS

In this mode the sensor must be in the path, so now it is a Linux box routing between two networks. Change `alert` to `drop` in the rule, then send forwarded traffic through Suricata using NFQUEUE (a Linux feature that passes packets to a user program):

```bash
# Linux inline sensor
sudo iptables -I FORWARD -j NFQUEUE --queue-num 0 --queue-bypass
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

With `--queue-bypass`, packets are accepted if Suricata is not running. That is fail-open. Without it, packets are dropped. That is fail-closed. EVE now shows `"action": "blocked"`, and the server never sees the request.

If this had been the first version of the rule, the blog reader would have been blocked as well. That is why new rules start in alert-only mode.

### A short honeypot example

The company also runs Cowrie, a fake SSH server, on 10.0.40.66. Cowrie listens on port 2222 by default. Nothing real uses this host. One day the SIEM receives this event (trimmed):

```text
{"eventid":"cowrie.login.failed","username":"admin",
 "password":"admin","src_ip":"10.0.60.25",
 "timestamp":"2026-10-31T02:11:40.118Z"}
```

The source is inside the company. DHCP logs show who had 10.0.60.25 at 02:11. No laptop should try SSH logins on this host, so this one alert goes straight to Tier 2.

## Going Deeper

### How attackers evade a NIDS

Thomas Ptacek and Timothy Newsham described the core problem in a well-known 1998 paper: a sensor and the target can see the same packets differently.

- **Fragmentation:** split the attack across many IP fragments. See [IPv4 Header and Fragmentation](../04-ip-addressing/10-ipv4-header-and-fragmentation.md).
- **TCP segmentation and overlap:** split a pattern across segments, or send overlapping data. Different operating systems rebuild overlaps differently.
- **Encoding:** write `%55NION` in place of `UNION`, or use unusual comment and case tricks.
- **Low and slow:** one attempt per hour stays under every threshold.
- **Encryption:** the simplest evasion of all.

The defence is **normalisation** and **reassembly**. The sensor rebuilds streams the way the target would, and decodes the content before matching. That is why `http.uri` is better than searching raw bytes. Suricata can even apply different reassembly policies for different target operating systems.

### Encrypted traffic

Most web traffic now uses TLS, so a NIDS cannot read URIs and bodies. Options:

- **Metadata.** The sensor can still see addresses, ports, timing and sizes. It can often see the TLS server name (SNI), unless Encrypted Client Hello is in use. In TLS 1.3 the certificate itself is encrypted.
- **TLS fingerprints.** JA3 and the newer JA4 hash the client's TLS settings. Some malware has a recognisable fingerprint, but it can be faked.
- **Decrypt at a trusted point.** Put the sensor after the load balancer, where HTTPS has ended. See [Load Balancing](../10-enterprise-and-datacenter/04-load-balancing.md).
- **TLS inspection.** An inline device acts as a proxy, using a private CA that clients trust. It breaks some pinned applications and raises privacy issues, so banking and health sites are often excluded.
- **Host visibility.** EDR sees activity after decryption.

Passive decryption with the server's private key does not work for TLS 1.3, because every session uses temporary keys (forward secrecy). See [TLS and HTTPS](../07-application-protocols/07-tls-and-https.md).

QUIC also encrypts most of its headers. Some companies block UDP 443 so that browsers fall back to TCP, where their inspection tools work better. See [QUIC](../06-transport-layer/08-quic.md).

### Performance and dropped packets

A sensor that cannot keep up misses packets silently. An IDS that dropped 20% of packets may have dropped the attack.

```bash
# Linux: check the latest counter from Suricata
sudo grep capture.kernel_drops /var/log/suricata/stats.log | tail -n 1
```

```text
capture.kernel_drops                          | Total     | 0
```

Common ways to go faster:

- Use a fast capture method such as AF_PACKET, DPDK or special capture cards.
- Keep each flow on one worker thread, so a stream stays in order.
- Use a fast pattern-matching library, such as Hyperscan with Suricata.
- Remove unneeded rules, because every enabled rule costs CPU.

An inline IPS is worse here, because an overloaded IPS delays or drops real traffic. It is also a possible target: traffic built to be slow to inspect can become a denial of service against the IPS itself.

### Asymmetric routing

If requests pass through one sensor and replies pass through another, neither sees the full conversation. Stream tracking and many rules then fail. Fixes are to place sensors where both directions meet, or to feed both paths into one sensor.

### Where IPS lives today

Standalone IPS boxes are less common now. An IPS is usually a feature of a **next-generation firewall** (NGFW), as in [Firewalls](04-firewalls.md). A **web application firewall** (WAF) focuses on HTTP attacks and often sits where TLS has already ended. For a shop like the one in our example, a WAF is often the better blocking tool. See [Proxies and Reverse Proxies](../10-enterprise-and-datacenter/05-proxies-and-reverse-proxies.md).

In the cloud, you often have no physical switch to mirror. AWS offers VPC Traffic Mirroring, and AWS Network Firewall accepts Suricata-compatible rules. See [Security Groups, NACLs and VPC Connectivity](../11-cloud-and-containers/04-cloud-security-and-connectivity.md).

### The alphabet soup

| Term | Meaning |
|---|---|
| NDR | Network Detection and Response: mostly behaviour and machine-learning analysis of traffic |
| EDR | Endpoint Detection and Response: agent on hosts that detects and acts |
| XDR | Extended Detection and Response: joins endpoint, network, email and cloud data |
| UEBA | User and Entity Behaviour Analytics: finds unusual user or device behaviour |
| SOAR | Security Orchestration, Automation and Response: runs playbooks automatically |
| MDR | Managed Detection and Response: an outside company does this for you |
| Sigma | An open format for writing detection rules that can convert to many SIEMs |

A **SOAR** example: on a phishing alert, it pulls the email, checks links against threat feeds, opens a ticket and, after approval, disables the account. A SIEM finds the problem, and a SOAR acts on it.

Flow records from routers and switches are another useful SIEM source. See [Network Monitoring and Observability](../12-troubleshooting-and-tools/11-network-monitoring.md).

## Common Mistakes

- **Thinking an IDS blocks attacks.** It reports them. Only inline devices can stop the first packet.
- **Thinking a SIEM is a kind of IDS.** A SIEM works on logs from many sources and sees no packets unless something feeds it that data.
- **Thinking an IPS replaces a firewall.** A firewall enforces what is allowed. An IPS looks for attacks inside allowed traffic. You need both.
- **Turning on block mode on day one.** Untuned rules block real users. Start in alert-only mode.
- **Installing a sensor and never tuning it.** It becomes a noise machine, and people stop reading it.
- **Believing "no alerts" means "no attacks".** It may mean a blind spot: encrypted traffic, dropped packets, an expired rule feed or a broken SPAN.
- **Forgetting that a SPAN can drop traffic.** Under load, a SPAN port can lose copies with no warning to the analyst.
- **Watching only the internet edge.** An edge sensor does not see east-west movement.
- **Using anomaly detection without a clean baseline.** If the attacker was already there, the system learns attack traffic as normal.
- **Assuming signature detection is outdated.** It remains the most precise tool for known attacks. Good programs use both methods.
- **Unsynced clocks on log sources.** Events appear out of order, and correlation rules fail.
- **Putting a honeypot on the real LAN without isolation.** A compromised honeypot can become an attacker's foothold.

## Interview Questions

**Q: What is the difference between an IDS and an IPS?**
A: An IDS sits out of band, gets a copy of the traffic and raises alerts. An IPS sits inline, so traffic passes through it and it can drop bad packets before they arrive. The IDS has no effect on the network but cannot stop the first packet. The IPS can stop it, but it adds latency and can block real traffic if a rule is wrong.

**Q: Compare signature-based and anomaly-based detection.**
A: Signature detection matches traffic against patterns of known attacks. It is precise and easy to explain, but it misses attacks with no rule, such as zero-days. Anomaly detection learns a baseline and flags what differs from it. It can catch new attacks, but it gives more false positives and needs a clean baseline. Real systems use both.

**Q: What is a false positive, and which is worse, a false positive or a false negative?**
A: A false positive is an alert on harmless activity. A false negative is a real attack with no alert. A single false negative is worse, because nobody responds. But many false positives cause alert fatigue, which then causes missed attacks. So the goal is to tune for balance. On an IPS, a false positive also blocks a real user, so the bar for blocking rules is higher.

**Q: What does fail-open and fail-closed mean for an inline IPS?**
A: Fail-open means traffic keeps flowing without inspection if the IPS fails. Fail-closed means traffic stops. Fail-open favours availability and fail-closed favours security. The choice depends on the risk of the link. Bypass hardware or a high-availability pair reduces the problem.

**Q: What is the difference between a SPAN port and a network TAP?**
A: A SPAN is a switch feature that copies traffic to a monitoring port. It is free and easy to set up, but it can drop copies under load and may not copy bad frames. A TAP is hardware on the link that copies traffic without using the switch, so it is more reliable for full capture. It costs money and needs a short outage to install.

**Q: Compare NIDS and HIDS. When would you need a HIDS?**
A: A NIDS watches packets on a network segment and covers many hosts, but it cannot read encrypted content. A HIDS is an agent on one host. It reads logs, files and processes, so it sees activity after decryption and can often tell if an attack worked. I would use a HIDS or EDR on important servers and on laptops that leave the office, along with a NIDS at key points.

**Q: Snort or Suricata: what are the main differences?**
A: Both are open-source IDS and IPS engines that use similar rule formats. Suricata was multi-threaded from the start and has rich protocol logging through EVE JSON. Snort is the older project, now owned by Cisco, and Snort 3 is also multi-threaded. Cisco's firewalls use Snort, and Suricata is common in open-source stacks. Rules are not always portable, so they must be tested.

**Q: What does a SIEM do, and how is it different from an IDS?**
A: A SIEM collects logs from many sources, normalises them, correlates them and raises alerts for analysts. An IDS looks at traffic and is one of the sources that feeds the SIEM. The SIEM sees the wider story. For example, it can link a firewall event, failed logins and a successful VPN login. It usually does not block anything itself.

**Q: Your IDS raises a high severity alert on a server. What do you do?**
A: First I check if it is real. I look at the rule, the packet or log details, the source and how important the server is. I check server logs to see whether the attack worked, for example the HTTP status and response size. If it is a false positive, I close it and tune the rule. If it is real, I escalate, find the scope, contain it, remove the cause, recover and write up the lessons.

**Q: Most traffic is encrypted. Is a NIDS still useful?**
A: Yes, but less so for content. It can still use addresses, DNS, SNI when visible, TLS fingerprints, timing and volume. For full visibility, I would place a sensor where TLS ends, such as after the load balancer, use TLS inspection where policy allows, and use EDR on hosts. Passive decryption with a server key does not work with TLS 1.3 because of forward secrecy.

**Q: Where would you place sensors in a network?**
A: I start inside the firewall, because alerts there are for traffic that really passed. I add one for the DMZ and one for server or data centre east-west traffic, because attackers move sideways after getting in. A sensor outside the firewall sees everything, but it is very noisy. I also make sure the sensor sees both directions of a flow.

**Q: What is a honeypot, and what are the risks?**
A: A honeypot is a fake system with no real users, so any contact with it is suspicious. It gives high-quality alerts, and an internal one is good at catching attackers who are scanning. The main risk is that a compromised honeypot is used to attack others, so it must be isolated and closely watched. It also does not protect anything by itself, it only provides early warning and information.

## Quick Recap

- An IDS is out of band and alerts. An IPS is inline and blocks. Failure handling (fail-open or fail-closed) matters only for inline devices.
- Signature detection finds known attacks. Anomaly detection finds unusual behaviour. Good systems use both.
- False positives waste time and cause alert fatigue. False negatives are missed attacks. Tuning balances the two.
- A NIDS sees network traffic across many hosts. A HIDS or EDR sees inside one host, even after decryption.
- Snort and Suricata are the main open-source engines. Learn how to read a rule: action, protocol, addresses, ports, direction and options.
- A SIEM collects, normalises and correlates logs. A SOC is the team that triages, investigates, contains, removes, recovers and learns.
- A honeypot is a fake system with no real users, so any touch is a strong signal.
