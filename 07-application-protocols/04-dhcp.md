# DHCP

**Level:** Beginner | **Prerequisites:** [IPv4 Addressing Basics](../04-ip-addressing/01-ipv4-basics.md), [The Default Gateway](../04-ip-addressing/08-default-gateway.md), [Ports and Sockets](../06-transport-layer/01-ports-and-sockets.md)

## In One Line

DHCP (Dynamic Host Configuration Protocol) gives a device an IP address, subnet mask, default gateway and DNS servers automatically when it joins a network, for a limited time called a lease.

## Why It Matters

A device needs four things before it can reach the internet:

| Setting | Example | Why it is needed |
|---|---|---|
| IP address | 192.168.10.21 | The device's own address |
| Subnet mask | 255.255.255.0 | Tells it which addresses are local |
| Default gateway | 192.168.10.1 | The router that leads off the local network |
| DNS server | 192.168.10.5 | Turns names like example.com into IP addresses |

Typing these by hand on 2,000 laptops is slow. People make typing mistakes. Two people can pick the same address, which causes an address conflict. Phones and guests come and go all day.

DHCP solves this. One server holds a list of free addresses and hands them out. When a device leaves, its address goes back on the list.

Interviewers love DHCP because one short topic tests many things at once: broadcast, UDP ports, how routers treat broadcasts, relays, timers, and troubleshooting. "A user got a 169.254.x.x address. What do you check?" is one of the most common interview questions in networking.

## Simple Explanation

Think of a hotel front desk.

1. You walk in and call out, "I need a room."
2. The receptionist says, "Room 21 is free."
3. You say, "Yes, I will take room 21."
4. The receptionist confirms and gives you the key, plus a card with the Wi-Fi details and the checkout time.

The room is your IP address. The card is your gateway and DNS. The checkout time is the lease. If you want to stay longer, you ask to extend before checkout.

**Where the analogy stops:** a hotel guest knows who the receptionist is. A new DHCP client knows nothing, not even its own address, so it must shout to everyone on the local network. Also, several DHCP servers can answer at once, and a guest would never be offered rooms by two front desks in the same lobby.

## How It Works

### Key words

- **Client:** the device asking for an address.
- **Server:** the device handing out addresses. It can be a home router, a Cisco router, Windows Server, or Kea on Linux.
- **Pool (or scope):** the range of addresses the server may give out, for example 192.168.10.21 to 192.168.10.254. Windows calls it a scope.
- **Lease:** permission to use an address for a limited time.
- **Binding:** the server's record that says "this address is leased to this client".
- **Broadcast:** a message sent to every device on the local network. See [Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).
- **Unicast:** a message sent to one device.
- **MAC address:** the hardware address of a network card, such as 00:1a:2b:3c:4d:5e. See [MAC Addresses](../03-ethernet-and-switching/01-mac-addresses.md).

### Why UDP, and why two ports

DHCP runs over UDP. UDP sends a single message with no connection setup. TCP needs a handshake between two known IP addresses, and a new client has no IP address yet and does not know the server's address. So TCP cannot work here.

- **UDP 67:** the server listens here.
- **UDP 68:** the client listens here.

Two fixed ports matter because some replies are broadcast. If clients used random ports, a broadcast reply would have no known port to land on.

DHCP is an application layer protocol that also does a job for the network layer. See [UDP in Depth](../06-transport-layer/07-udp.md) for more on UDP.

### DORA: the four messages

DORA stands for Discover, Offer, Request, Acknowledge.

```text
 Client                                          Server
 (no IP yet)                                     192.168.10.1
    |                                                |
    | 1. DISCOVER  0.0.0.0:68 -> 255.255.255.255:67  |
    |    "Is any DHCP server there?"                 |
    |----------------------------------------------->|
    |                                                |
    | 2. OFFER     192.168.10.1:67 -> client:68      |
    |    "You can have 192.168.10.21"                |
    |<-----------------------------------------------|
    |                                                |
    | 3. REQUEST   0.0.0.0:68 -> 255.255.255.255:67  |
    |    "I accept the offer from 192.168.10.1"      |
    |----------------------------------------------->|
    |                                                |
    | 4. ACK       192.168.10.1:67 -> client:68      |
    |    "Confirmed. Lease is 86400 seconds"         |
    |<-----------------------------------------------|
```

1. **Discover.** The client has no IP, so it uses source 0.0.0.0. It does not know the server, so the destination is 255.255.255.255 at Layer 3 and ff:ff:ff:ff:ff:ff at Layer 2. Every device in the same broadcast domain receives it.
2. **Offer.** Each server that has a free address replies with one. The address sits in a field called `yiaddr` ("your IP address"). Many servers first check that the address is not in use. Cisco IOS sends 2 pings and waits 500 ms for each by default.
3. **Request.** The client picks one offer, usually the first. It broadcasts the Request and includes option 54 (server identifier) to name the chosen server, and option 50 (requested IP address). It is still a broadcast so that other servers learn their offers were refused and can put those addresses back in the pool.
4. **Ack.** The chosen server commits the lease and sends the full settings. If the address is no longer free, it sends a NAK (negative acknowledgement) and the client starts again.

After the Ack, the client should check that nobody else uses the address, normally with an ARP probe. See [ARP](../03-ethernet-and-switching/05-arp.md). If someone does, it sends DHCPDECLINE and starts again.

**Is the Offer unicast or broadcast?** Both are allowed. The client sets a bit called the broadcast flag. If the flag is set, the server broadcasts the reply to 255.255.255.255. If it is clear, the server sends it directly to the offered address and the client's MAC. Which one you see depends on the client.

### All message types

| Option 53 value | Message | Sent by | Meaning |
|---|---|---|---|
| 1 | DISCOVER | Client | Find servers |
| 2 | OFFER | Server | Offer an address |
| 3 | REQUEST | Client | Accept, renew or confirm an address |
| 4 | DECLINE | Client | The address is already in use |
| 5 | ACK | Server | Confirmed |
| 6 | NAK | Server | Refused, start again |
| 7 | RELEASE | Client | I am done with this address |
| 8 | INFORM | Client | I have an IP, just send me other settings |

### Packet layout

DHCP grew out of an older protocol called BOOTP (Bootstrap Protocol), so it keeps the same format. This is why packet tools often say "BOOTP". The numbers in brackets are bytes.

```text
 0               8               16              24            31
+---------------+---------------+---------------+---------------+
|    op (1)     |   htype (1)   |   hlen (1)    |   hops (1)    |
+---------------+---------------+---------------+---------------+
|                            xid (4)                            |
+-------------------------------+-------------------------------+
|           secs (2)            |           flags (2)           |
+-------------------------------+-------------------------------+
|                ciaddr (4): client's current IP                |
+---------------------------------------------------------------+
|                yiaddr (4): "your" offered IP                  |
+---------------------------------------------------------------+
|                siaddr (4): next server (boot)                 |
+---------------------------------------------------------------+
|                giaddr (4): relay agent IP                     |
+---------------------------------------------------------------+
|                chaddr (16): client MAC                        |
+---------------------------------------------------------------+
|                sname (64): server host name                   |
+---------------------------------------------------------------+
|                file (128): boot file name                     |
+---------------------------------------------------------------+
|                options (variable)                             |
+---------------------------------------------------------------+
```

The `xid` (transaction ID) is a random number that ties replies to the right request. The `op` field is 1 for anything from a client and 2 for anything from a server.

### Lease time and T1/T2 renewal

A lease is like renting a flat. You do not wait for the last day to ask for more time. The renewal timers work the same way. Where the analogy stops: the client does all of this silently, and no human agrees to anything.

Using a lease of 24 hours (86400 seconds):

```text
 0h                     12h                   21h      24h
 |----------------------|---------------------|--------|
 Lease starts           T1 (50%)              T2       Expiry
 Bound: use the IP      Renewing:             87.5%    Stop using
                        unicast to the        Rebind:  the IP
                        same server           broadcast
                                              to any
                                              server
```

1. **Bound.** The client uses the address normally.
2. **T1, at 50% of the lease.** The client unicasts a Request to the server that gave the lease. If the server answers Ack, the timers restart.
3. **T2, at 87.5% of the lease.** The first server has not answered, so the client broadcasts a Request. Any server that knows this client may answer.
4. **Expiry.** If nobody answered, the client must stop using the address and go back to Discover.

T1 and T2 are defaults from the standard (RFC 2131). A server can set other values with options 58 and 59.

Lease length is a design choice. Windows Server scopes default to 8 days. Cisco IOS defaults to 1 day. Home routers vary, and 24 hours is common. Guest Wi-Fi often uses 1 to 4 hours so that addresses come back quickly.

**INIT-REBOOT.** A client that restarts and remembers a valid lease skips Discover. It broadcasts a Request for its old address. The server sends Ack if the address is still right and NAK if the client has moved to a wrong subnet. This is why a laptop joins a known network quickly.

### Options: the extra settings

Everything beyond the address travels as options. Each option has three parts: code, length, value. This format is called TLV (type, length, value).

For example, a router of 192.168.10.1 is encoded in 6 bytes:

```text
 Code   Length   Value
 03     04       C0 A8 0A 01
 (3)    (4)      192.168.10.1
```

| Option | Name | Example |
|---|---|---|
| 1 | Subnet mask | 255.255.255.0 |
| 3 | Router (default gateway) | 192.168.10.1 |
| 6 | DNS servers | 192.168.10.5, 192.168.10.6 |
| 15 | Domain name | corp.example.com |
| 42 | NTP servers (time) | 192.168.10.7 |
| 43 | Vendor specific | Used by some wireless access points to find their controller |
| 50 | Requested IP address | 192.168.10.21 |
| 51 | Lease time | 86400 |
| 53 | Message type | 1 to 8 from the table above |
| 54 | Server identifier | 192.168.10.1 |
| 55 | Parameter request list | The list of options the client wants |
| 58, 59 | T1, T2 | 43200, 75600 |
| 60 | Vendor class | PXEClient |
| 61 | Client identifier | Often 01 plus the MAC |
| 66, 67 | TFTP server, boot file | Network boot (PXE) |
| 82 | Relay agent information | Added by relays and switches |
| 119 | Domain search list | corp.example.com |
| 121 | Classless static routes | Extra routes for the client |
| 150 | TFTP server addresses | Cisco IP phones (Cisco specific) |

### DHCP relay and ip helper-address

Routers do not forward broadcasts. So a Discover from VLAN 20 never reaches a server in VLAN 99. Running one server in every VLAN is wasteful. A **VLAN** is a separate logical network on a switch. See [VLANs](../03-ethernet-and-switching/06-vlans.md).

A **DHCP relay agent** solves this. It is a router or Layer 3 switch that listens for DHCP broadcasts and forwards them as unicast to a server. On Cisco, the command is `ip helper-address`.

Analogy: an office floor has a receptionist with a phone. You ask her for a parking pass, and she calls the head office for you. Where the analogy stops: she also writes her floor number on the request, and the head office uses that number to choose what to give you.

```text
 Client           Relay (router)                    Server
 VLAN 20          Vlan20: 10.20.0.1                 10.99.0.10
                  Vlan99: 10.99.0.1
    |                     |                             |
    | DISCOVER (broadcast)|                             |
    |-------------------->| giaddr = 10.20.0.1          |
    |                     | DISCOVER (unicast)          |
    |                     |---------------------------->|
    |                     |                             | Picks pool
    |                     |  OFFER to 10.20.0.1, UDP 67 | for VLAN 20
    |                     |<----------------------------|
    |    OFFER on VLAN 20 |                             |
    |<--------------------|                             |
```

What the relay does:

1. It receives the broadcast on the client's interface.
2. It writes that interface's IP into `giaddr` (gateway IP address) and adds 1 to `hops`.
3. It unicasts the message to the server. Both source and destination ports are UDP 67.
4. The server reads `giaddr` to choose the right pool, because the address must fit the client's subnet.
5. The server replies to `giaddr`, and the relay passes the reply to the client on UDP 68.

Two more facts about `ip helper-address`:

- It goes on the interface facing the clients, not the one facing the server.
- By default it forwards UDP broadcasts for 8 services, not just DHCP: Time (37), TACACS (49), DNS (53), BOOTP/DHCP (67, 68), TFTP (69), NetBIOS name (137) and NetBIOS datagram (138). Turn off what you do not need with `no ip forward-protocol udp <port>`.

### Reservations

A **reservation** tells the server: "always give this client this address." The client still uses DHCP, but the answer never changes. Reservations suit printers, IP cameras and access points.

Compared with static configuration, a reservation is managed in one place, and the device still gets DNS and gateway changes automatically. Routers and core servers usually stay fully static, because they must work even when DHCP is down.

The server identifies the client by MAC address or client identifier (option 61). Note that phones and laptops now often use random MAC addresses on Wi-Fi, so a MAC reservation can silently stop matching.

### What happens on failure, and APIPA

**No server answers a new client.** The client retries Discover with growing gaps. RFC 2131 suggests about 4, 8, 16, 32 and 64 seconds, with some random change.

Then many operating systems give themselves an address from 169.254.0.0/16. Windows calls this **APIPA** (Automatic Private IP Addressing). The general standard is IPv4 link-local addressing, RFC 3927. The client picks a random address from 169.254.1.0 to 169.254.254.255 and uses ARP to check that nobody else has it.

Analogy: at a meeting with no name tags, you write your own. People in the room can speak to you, but nobody outside knows you. Where the analogy stops: two APIPA devices can really talk to each other, but only on the same local link, because there is no gateway and routers do not forward these addresses.

Windows keeps checking for a DHCP server, about every 5 minutes, and replaces the APIPA address when a server appears.

Linux is different. Whether it uses link-local depends on the network software. Many Linux systems simply stay without an IPv4 address.

**Server dies while clients hold leases.** Nothing happens at first. Clients keep working until the lease expires. With an 8-day lease, you have days to fix the server. New clients fail right away.

**Pool is full.** The server has nothing to offer, so new clients look the same as if there were no server. The fix is a larger subnet, shorter leases, or removing old bindings.

### Rogue DHCP servers

A **rogue DHCP server** is one that nobody approved. Often it is an accident: someone plugs a home router in by its LAN port. Sometimes it is an attack.

Clients take the first offer, so a rogue server can win the race. It can hand out wrong addresses, which breaks users. Worse, it can give its own IP as gateway or DNS, so all traffic passes through the attacker. That is a man-in-the-middle attack.

Analogy: someone in a fake uniform sets up a desk in the lobby and hands out keys. Where the analogy stops: guests would notice the extra desk, but clients check nothing, because DHCP has no sign-in or proof of identity in normal use.

**DHCP starvation** is a related attack. The attacker sends many Discovers with fake MAC addresses until the pool is empty, then often runs a rogue server.

The main defence is **DHCP snooping** on switches. Each port is either trusted or untrusted:

- Server messages (Offer, Ack, NAK) are dropped on untrusted ports.
- Only ports toward real servers and uplinks are trusted.
- The switch builds a table of MAC, IP, VLAN and port. Dynamic ARP Inspection and IP Source Guard use this table.

Full details are in [Layer 2 Security](../03-ethernet-and-switching/11-layer2-security.md).

### DHCPv6

IPv6 clients can get addresses without DHCP at all, using **SLAAC** (Stateless Address Autoconfiguration), where a device builds its own address from information sent by the router. See [IPv6 in Practice](../04-ip-addressing/12-ipv6-in-practice.md). So DHCPv6 is optional, and the router tells clients what to do with flags in its Router Advertisement (RA):

| Mode | RA flags | Address from | DNS from |
|---|---|---|---|
| SLAAC only | A=1, M=0, O=0 | SLAAC | RA (RDNSS option, if sent) |
| Stateless DHCPv6 | A=1, M=0, O=1 | SLAAC | DHCPv6 |
| Stateful DHCPv6 | A=0, M=1 | DHCPv6 | DHCPv6 |

The four messages are SARR: Solicit, Advertise, Request, Reply. With the Rapid Commit option, Solicit and Reply are enough.

| Point | DHCPv4 | DHCPv6 |
|---|---|---|
| Ports | UDP 67 (server), 68 (client) | UDP 547 (server), 546 (client) |
| Messages | DORA | SARR |
| Client sends to | Broadcast 255.255.255.255 | Multicast ff02::1:2 |
| Client source address | 0.0.0.0 | Its link-local address (fe80::/10) |
| Client identity | MAC or option 61 | DUID (DHCP Unique Identifier) |
| Default gateway | Option 3 | Not sent. Comes from RA |
| Standard | RFC 2131, options RFC 2132 | RFC 9915 (replaced RFC 8415) |

The main trap: DHCPv6 has no default gateway option. A host with a DHCPv6 address still needs a Router Advertisement to get its gateway.

DHCPv6 can also give a whole network, not just one address. This is **prefix delegation**. Your ISP may hand your home router a /56, and the router splits it into /64 networks.

Support varies by device. Android has historically not supported getting an address from stateful DHCPv6, so many networks keep SLAAC turned on.

## Example

### Example 1: DHCP server on a Cisco router

Network: 192.168.10.0/24. The router is 192.168.10.1, DNS servers are .5 and .6, and .1 to .20 are kept for fixed devices.

```text
! Cisco IOS
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp ping packets 2
!
ip dhcp pool LAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.10.5 192.168.10.6
 domain-name corp.example.com
 lease 1
!
ip dhcp pool PRINTER1
 host 192.168.10.50 255.255.255.0
 client-identifier 0100.1122.3344.55
```

`lease 1` means 1 day. `PRINTER1` is a reservation. The value 01 at the start of the client identifier means Ethernet, followed by the MAC 00:11:22:33:44:55. Some clients need `hardware-address` instead. The host pool inherits the gateway and DNS from `LAN10`, because its address is inside that network.

Capture the exchange on the client:

```bash
# Linux client, in one terminal
sudo tcpdump -ni eth0 'udp port 67 or udp port 68'
# in a second terminal: give up the lease, then ask again
sudo dhclient -r eth0
sudo dhclient -v eth0
```

```text
09:14:02.101234 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP,
    Request from 00:1a:2b:3c:4d:5e, length 300
09:14:03.110482 IP 192.168.10.1.67 > 192.168.10.21.68: BOOTP/DHCP,
    Reply, length 300
09:14:03.112056 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP,
    Request from 00:1a:2b:3c:4d:5e, length 300
09:14:03.114590 IP 192.168.10.1.67 > 192.168.10.21.68: BOOTP/DHCP,
    Reply, length 300
```

These are the four DORA messages. Each line is wrapped here to fit. tcpdump prints "Request" and "Reply" from the old BOOTP `op` field, so Discover and Request look the same. Add `-v` to see `DHCP-Message (53)` with the real type. The gap of about 1 second before the Offer is the two pings at 500 ms each.

Check the result on each client.

```bash
# Linux
ip -4 addr show eth0
ip route show default
```

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    inet 192.168.10.21/24 brd 192.168.10.255 scope global dynamic eth0
       valid_lft 86212sec preferred_lft 86212sec
default via 192.168.10.1 dev eth0 proto dhcp metric 100
```

The word `dynamic` and the `valid_lft` countdown show this came from DHCP.

```powershell
# Windows, same client details if it were a Windows laptop
ipconfig /all
```

```text
Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : corp.example.com
   Physical Address. . . . . . . . . : 00-1A-2B-3C-4D-5E
   DHCP Enabled. . . . . . . . . . . : Yes
   IPv4 Address. . . . . . . . . . . : 192.168.10.21(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Lease Obtained. . . . . . . . . . : Friday, October 30, 2026 9:14:03 AM
   Lease Expires . . . . . . . . . . : Saturday, October 31, 2026 9:14:03 AM
   Default Gateway . . . . . . . . . : 192.168.10.1
   DHCP Server . . . . . . . . . . . : 192.168.10.1
   DNS Servers . . . . . . . . . . . : 192.168.10.5
                                       192.168.10.6
```

So T1 is at 21:14 tonight, T2 is at 06:14 tomorrow, and expiry is at 09:14 tomorrow.

On the router:

```text
Router# show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address       Client-ID/            Lease expiration      Type
                 Hardware address/
                 User name
192.168.10.21    0100.1a2b.3c4d.5e     Oct 31 2026 09:14 AM  Automatic
192.168.10.50    0100.1122.3344.55     Infinite              Manual
```

### Example 2: relay for VLAN 20

Clients are in 10.20.0.0/24 and the server is 10.99.0.10. Everything is on one Layer 3 switch.

```text
! Cisco IOS, the relay
interface Vlan20
 ip address 10.20.0.1 255.255.255.0
 ip helper-address 10.99.0.10
!
interface Vlan99
 ip address 10.99.0.1 255.255.255.0
```

The server needs a pool for 10.20.0.0/24 and a route back to 10.20.0.1. Here is that pool on Kea, the current DHCP server from ISC, on Linux:

```text
// /etc/kea/kea-dhcp4.conf (path can vary by package)
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": [ "eth0" ] },
    "lease-database": { "type": "memfile" },
    "valid-lifetime": 86400,
    "renew-timer": 43200,
    "rebind-timer": 75600,
    "subnet4": [
      {
        "id": 20,
        "subnet": "10.20.0.0/24",
        "pools": [ { "pool": "10.20.0.50 - 10.20.0.250" } ],
        "option-data": [
          { "name": "routers", "data": "10.20.0.1" },
          { "name": "domain-name-servers", "data": "10.99.0.53" },
          { "name": "domain-name", "data": "corp.example.com" }
        ],
        "reservations": [
          {
            "hw-address": "00:11:22:33:44:55",
            "ip-address": "10.20.0.60"
          }
        ]
      }
    ]
  }
}
```

The older ISC DHCP server (`dhcpd`) reached end of life in 2022, so you will still see it on old systems, but new builds should use Kea. For small networks, `dnsmasq` is also common.

### Example 3: rogue server protection

Access ports are Gi1/0/1-23. The real server is reached through uplink Gi1/0/24.

```text
! Cisco IOS switch
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
!
interface GigabitEthernet1/0/24
 ip dhcp snooping trust
!
interface range GigabitEthernet1/0/1 - 23
 ip dhcp snooping limit rate 15
```

### Command reference

```text
! Cisco IOS
show ip dhcp pool                 ! pool size and leases in use
show ip dhcp binding              ! who has which address
show ip dhcp conflict             ! addresses found already in use
show ip dhcp server statistics    ! message counters
show ip interface vlan 20         ! look for "Helper address is"
show ip dhcp snooping             ! snooping state and trusted ports
show ip dhcp snooping binding     ! snooping table
show dhcp lease                   ! when the router itself is a client
debug ip dhcp server packet       ! live view, use with care
clear ip dhcp binding *           ! remove automatic bindings
clear ip dhcp conflict *          ! clear the conflict list
```

```bash
# Linux client
ip addr show eth0
nmcli device show eth0             # NetworkManager systems
networkctl status eth0             # systemd-networkd systems
sudo dhclient -r eth0              # release, if dhclient is in use
sudo dhclient -v eth0              # renew, if dhclient is in use
journalctl -u NetworkManager | grep -i dhcp

# Linux server (Kea)
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf   # test the config file
```

Which client is present depends on the distribution. Many modern systems use the DHCP client built into NetworkManager or systemd-networkd rather than `dhclient`.

```powershell
# Windows client
ipconfig /all
ipconfig /release
ipconfig /renew
ipconfig /renew6
Get-NetIPAddress -AddressFamily IPv4 | Select-Object IPAddress, PrefixOrigin

# Windows Server, DHCP role
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId 192.168.10.0
Add-DhcpServerv4Reservation -ScopeId 192.168.10.0 -IPAddress 192.168.10.50 -ClientId "00-11-22-33-44-55"
```

## Going Deeper

**Stateful DHCPv6 on Cisco.** The RA tells hosts to use DHCPv6 and not SLAAC.

```text
! Cisco IOS
ipv6 unicast-routing
ipv6 dhcp pool V6LAN
 address prefix 2001:db8:10::/64 lifetime 86400 43200
 dns-server 2001:db8:10::53
 domain-name corp.example.com
!
interface GigabitEthernet0/1
 ipv6 address 2001:db8:10::1/64
 ipv6 dhcp server V6LAN
 ipv6 nd managed-config-flag
 ipv6 nd prefix 2001:db8:10::/64 2592000 604800 no-autoconfig
```

Check it with `show ipv6 dhcp binding`, `show ipv6 dhcp pool` and `show ipv6 dhcp interface`. To relay DHCPv6, put `ipv6 dhcp relay destination 2001:db8:ffff::5` on the client-facing interface. RFC 9915 suggests T1 and T2 of 0.5 and 0.8 times the shortest preferred lifetime, which differs from the DHCPv4 defaults.

**Option 82.** A relay or snooping switch can add this option to identify the client's switch and port, using a circuit ID and a remote ID. The server can use it for policy, and the relay removes it before passing the reply to the client.

**Redundancy.** Two simple servers with separate pools is the classic 80/20 rule: one server holds 80% of the addresses and the other 20%. Better is true failover, where servers share lease state. Windows Server has failover in load balance and hot standby modes, and Kea has a High Availability hook. Cisco IOS needs a separate approach, so large sites use dedicated servers.

**Option 121 and VPNs.** Option 121 can install routes more specific than a VPN's routes. The TunnelVision attack (CVE-2024-3661, disclosed in 2024) used a rogue DHCP server and this option to pull traffic out of a VPN tunnel. It is one more reason to deploy snooping.

**Windows AD authorization.** In Active Directory, a Windows DHCP server must be authorized before it serves clients. This only stops Windows servers, not a home router or an attacker's tool.

**DNS updates.** A DHCP server can register client names in DNS, so `laptop21.corp.example.com` follows the address. See [DNS Basics](01-dns-basics.md).

## Common Mistakes

- **Thinking DHCP only gives an IP address.** It also gives the mask, gateway, DNS and many other settings.
- **Saying DHCP uses TCP.** DHCPv4 uses UDP 67 and 68. A client with no address cannot do a TCP handshake.
- **Mixing up the ports.** 67 is the server, 68 is the client. In DHCPv6, 547 is the server and 546 is the client.
- **Thinking the Request is unicast in DORA.** The first Request is broadcast so other servers learn the choice. Renewal at T1 is the unicast case.
- **Forgetting that a client takes the first offer.** It is not the "best" offer, and that is why rogue servers work.
- **Putting `ip helper-address` on the wrong interface.** It goes on the client-facing interface or SVI (the virtual Layer 3 interface of a VLAN).
- **Forgetting the route back.** The server must be able to reach `giaddr`. Firewalls and ACLs between relay and server must allow UDP 67.
- **Assuming a 169.254.x.x address means the server is down.** It can also be a wrong VLAN, a missing helper, a full pool, a blocked port, or snooping dropping replies.
- **Trusting no ports with snooping on.** If the uplink is not trusted, every Offer is dropped and all clients fail.
- **Leaving option 82 insertion on with a Cisco IOS server.** By default a snooping switch adds option 82 with `giaddr` 0, and an IOS server may drop that. Use `no ip dhcp snooping information option` or `ip dhcp relay information trust-all` on the server.
- **Forgetting exclusions.** Without `ip dhcp excluded-address`, the router can offer addresses used by your servers.
- **Expecting DHCPv6 to give a default gateway.** It does not. The gateway comes from Router Advertisements.

## Interview Questions

**Q: Explain the DORA process.**

A: The client broadcasts a Discover. Servers answer with an Offer. The client broadcasts a Request naming the chosen server. That server sends an Ack with the lease and options. The client then normally checks with ARP that the address is free.

**Q: Which ports does DHCP use, and why UDP?**

A: The server uses UDP 67 and the client uses UDP 68. It uses UDP because the client has no IP address yet and does not know the server, so a TCP handshake is impossible. Broadcasts also do not work with TCP.

**Q: Why is the DHCP Request a broadcast when the client already knows the server?**

A: So that every server that made an offer hears the decision. The servers that were not chosen can return their offered addresses to the pool.

**Q: What are T1 and T2?**

A: T1 is at 50% of the lease. The client unicasts a renewal to its server. T2 is at 87.5%. If there has been no answer, the client broadcasts to any server. If the lease expires, the client stops using the address and starts DORA again.

**Q: How does DHCP work across subnets when routers block broadcasts?**

A: A relay agent does it. On Cisco we set `ip helper-address` on the client-facing interface. The router puts its interface IP in `giaddr` and unicasts the message to the server. The server uses `giaddr` to choose the pool, and it replies to the relay.

**Q: A user has a 169.254.x.x address. What do you check?**

A: That is a link-local address, so the client got no DHCP reply. I check the cable and link, the VLAN on the port, then whether the server is up and the pool has free addresses. Across subnets I check the helper address and the route back. I also check snooping and ACLs. I confirm with a packet capture to see if Discovers leave and Offers return.

**Q: What is a rogue DHCP server and how do you stop it?**

A: It is an unapproved server on the network. Clients take the first offer, so it can give a wrong gateway or DNS and capture traffic. I enable DHCP snooping on the switches, trust only the ports toward real servers, and rate limit access ports to reduce starvation attacks.

**Q: What is the difference between a reservation and a static IP?**

A: A static IP is set on the device itself. A reservation is set on the server, and the device still uses DHCP but always gets the same address. Reservations keep management in one place and still deliver option changes.

**Q: What happens if the DHCP server fails?**

A: Clients with a valid lease keep working until it expires, and they keep trying to renew. New clients get nothing, and many will use a 169.254 address. So lease length affects how long you have to repair it.

**Q: How is DHCPv6 different from DHCPv4?**

A: It uses UDP 546 and 547, multicast to ff02::1:2 instead of broadcast, and the messages are Solicit, Advertise, Request, Reply. Clients are identified by a DUID. It does not provide a default gateway, which comes from Router Advertisements. It can also delegate prefixes to routers.

## Quick Recap

- DHCP gives an address plus options for a limited lease, over UDP 67 (server) and 68 (client).
- DORA is Discover, Offer, Request, Ack. Discover and the first Request are broadcasts.
- T1 at 50% renews by unicast, T2 at 87.5% rebinds by broadcast, and expiry means stop using the address.
- Common options are 3 (router), 6 (DNS), 15 (domain) and 51 (lease time).
- Routers block broadcasts, so a relay with `ip helper-address` sets `giaddr` and unicasts to the server.
- No reply usually leads to a 169.254.x.x address. Rogue servers are stopped by DHCP snooping.
- DHCPv6 uses UDP 546 and 547 and SARR, and the default gateway comes from Router Advertisements.
