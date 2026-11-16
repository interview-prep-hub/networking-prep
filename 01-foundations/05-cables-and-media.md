# Cables and Transmission Media

**Level:** Beginner | **Prerequisites:** [What Is a Network?](01-what-is-a-network.md), [Network Devices](04-network-devices.md)

## In One Line

Transmission media are the physical paths that carry bits between devices: copper carries electrical signals, glass fibre carries light, and radio waves carry signals through the air.

## Why It Matters

Every network runs on a physical link. If the link is bad, nothing above it works. A perfect router config cannot fix a cracked fibre or a cable with one broken wire.

Media choice is also a real design decision. Copper Ethernet stops at 100 m. Two buildings 600 m apart need fibre. A server 2 m from its switch can use a cheap copper cable made for that short hop. Pick wrong and the link fails or costs far more than it should.

Interviewers ask about this for three reasons:

- They want to know that you can choose the right cable for a given speed and distance.
- They want to see you start troubleshooting at Layer 1. Layer 1 is the physical layer of the [OSI Model](../02-network-models/02-osi-model.md).
- Classic questions (Cat6 distance limits, crossover cables, single-mode vs multi-mode) quickly show who has touched real hardware.

## Simple Explanation

Think of three ways to send a message to a friend.

- **Copper cable** is like a speaking tube between two rooms. It is cheap and easy. Over distance, your voice fades, and noise from outside leaks in.
- **Fibre** is like flashing a torch down a long mirrored pipe. The light stays inside, travels very far, and outside noise cannot touch it.
- **Wireless** is like shouting across an open field. No pipe is needed, but everyone shares the same air, and walls and distance weaken your voice.

The fading of your voice has a name: **attenuation**. It means a signal gets weaker as it travels. All media suffer from it. Most of this chapter is about how each medium fights it.

Where the analogy stops: real links do not send sound, and fibre does not use mirrors. Fibre keeps light inside through an effect called total internal reflection. Also, a real link sends billions of signal changes per second, in both directions.

## How It Works

### Copper Twisted Pair

A twisted pair cable has 8 copper wires, grouped into 4 pairs. Each pair is twisted together.

```text
   Outer jacket
  +----------------------------------+
  |   (orange pair)   (green pair)   |
  |      \\//             \\//       |
  |      //\\             //\\       |
  |   (blue pair)     (brown pair)   |
  +----------------------------------+
  Each pair twists at its own rate.
```

**Why twist the wires?** A pair sends one signal as two opposite voltages. This is called differential signalling. The receiver looks only at the difference between the two wires.

Noise from outside hits both wires almost equally. The difference between them stays the same, so the noise cancels out. Twisting makes sure both wires get an equal share of the noise. It also reduces **crosstalk**, which is a signal leaking from one pair into a nearby pair.

Analogy: two friends stand on a boat. Waves lift both of them together, but the height difference between them does not change. Where it stops: real noise is never perfectly equal, which is why shielded cables exist.

### Cable Categories

A **category** (Cat) is a quality grade for twisted pair cable. A higher category is tested at higher frequencies. Frequency here is shown in MHz (million cycles per second). This is the electrical range the cable is rated for. It is not the same as data speed in Mbps (million bits per second).

| Category | Rated to | Typical Ethernet use | Max distance |
|---|---|---|---|
| Cat5e | 100 MHz | 1 Gbps, 2.5 Gbps | 100 m |
| Cat6 | 250 MHz | 1 Gbps, 5 Gbps | 100 m |
| Cat6 | 250 MHz | 10 Gbps | about 37-55 m |
| Cat6a | 500 MHz | 10 Gbps | 100 m |
| Cat7 | 600 MHz | 10 Gbps | 100 m |
| Cat8 | 2000 MHz | 25 Gbps, 40 Gbps | 30 m |

Key points for each one:

- **Cat5e** ("enhanced") is the minimum for new gigabit wiring. It is common in older buildings. Plain Cat5 is obsolete.
- **Cat6** handles 10 Gbps only on shorter runs. The limit depends on **alien crosstalk**, which is noise from neighbouring cables in the same bundle.
- **Cat6a** ("augmented") is the normal choice for new office wiring. It gives full 10 Gbps at 100 m. It is thicker and stiffer than Cat6.
- **Cat7 caveat:** Cat7 is an ISO/IEC class (Class F), not a TIA category. TIA is the US body behind the main cabling standards. Cat7 was designed for non-RJ45 connectors (GG45, TERA). A "Cat7" cable with RJ45 plugs does not meet the Cat7 standard as written. For 10 Gbps, Cat6a is the safer buy.
- **Cat8** is for short links inside a data centre rack row, not for desks. It is always shielded. 25GBASE-T and 40GBASE-T exist, but they are rare in practice, because fibre and DAC cables (covered below) won that market.

**The 100 m rule.** The 100 m limit is for the whole **channel**: at most 90 m of fixed cable in the wall, plus at most 10 m of patch cords. A **patch cord** is a short flexible cable that connects a device to a wall socket or a switch.

**Reading the Ethernet names.** In `1000BASE-T`, 1000 means 1000 Mbps, BASE means baseband (one signal on the wire at a time), and T means twisted pair. More in [Ethernet Frames and Standards](../03-ethernet-and-switching/02-ethernet-frames.md).

### UTP vs STP

**UTP** (Unshielded Twisted Pair) relies only on the twists. **STP** (Shielded Twisted Pair) adds metal foil or braid around the pairs to block noise. Noise from motors and power cables is called **EMI** (electromagnetic interference).

| | UTP | STP |
|---|---|---|
| Noise protection | Good | Better |
| Cost | Lower | Higher |
| Handling | Thin, flexible | Thicker, stiffer |
| Grounding | Not needed | Needed |
| Where used | Offices, homes | Factories, near motors, lift shafts, dense 10G bundles |

The shield must be grounded correctly. A badly grounded shield can act like an antenna and pick up more noise.

You will see names like U/UTP, F/UTP and S/FTP. The letter before the slash describes the overall shield. The letters after it describe each pair. U is unshielded, F is foil, S is braid (screen). So S/FTP has a braid around the whole cable and foil around each pair.

Two more cable facts come up on the job:

- **Solid** wire is for fixed runs in walls. **Stranded** wire is for flexible patch cords.
- **Plenum** cable (rated CMP in the US) is for air-handling spaces and gives off less smoke when it burns. Local fire rules decide where it is required.

### Straight-Through vs Crossover

Copper Ethernet plugs use the RJ45 connector, which has 8 pins. (The exact name is 8P8C, but everyone says RJ45.) Two wiring orders exist: T568A and T568B.

| Pin | T568A | T568B |
|---|---|---|
| 1 | White/Green | White/Orange |
| 2 | Green | Orange |
| 3 | White/Orange | White/Green |
| 4 | Blue | Blue |
| 5 | White/Blue | White/Blue |
| 6 | Orange | Green |
| 7 | White/Brown | White/Brown |
| 8 | Brown | Brown |

Neither is faster. Pick one and use it everywhere. T568B is very common in business installs.

- **Straight-through cable:** the same standard on both ends. Pin 1 connects to pin 1.
- **Crossover cable:** T568A on one end, T568B on the other. It swaps the transmit and receive pairs.

Why does this matter? At 10 and 100 Mbps, each side sends on one pair and listens on another. Think of two people talking through tubes: my mouth tube must reach your ear tube. If mouth meets mouth, nobody hears anything.

There are two port types. **MDI** (Medium Dependent Interface) ports are on PCs, routers and firewalls. They transmit on pins 1,2. **MDI-X** ports are on switches and hubs. They transmit on pins 3,6. The X means the crossing is done inside the port.

```text
Straight-through: unlike port types
 PC (MDI)                   Switch (MDI-X)
 Pin 1 (TX+) ----------->   Pin 1 (RX+)
 Pin 2 (TX-) ----------->   Pin 2 (RX-)
 Pin 3 (RX+) <-----------   Pin 3 (TX+)
 Pin 6 (RX-) <-----------   Pin 6 (TX-)

Crossover: same port types
 PC (MDI)                   PC (MDI)
 Pin 1 (TX+) ----------->   Pin 3 (RX+)
 Pin 2 (TX-) ----------->   Pin 6 (RX-)
 Pin 3 (RX+) <-----------   Pin 1 (TX+)
 Pin 6 (RX-) <-----------   Pin 2 (TX-)
```

TX means transmit. RX means receive. The classic rule follows from this diagram:

| Connection | Port types | Classic cable |
|---|---|---|
| PC to switch | MDI to MDI-X | Straight-through |
| Router to switch | MDI to MDI-X | Straight-through |
| Switch to switch | MDI-X to MDI-X | Crossover |
| PC to PC | MDI to MDI | Crossover |
| Router to router | MDI to MDI | Crossover |
| PC to router | MDI to MDI | Crossover |

The last row is the favourite trick question. A PC and a router look unlike, but both are MDI.

### Auto MDI-X

**Auto MDI-X** lets a port detect which pairs the other side is using and swap its own transmit and receive pairs if needed. With it, any working cable just works. The port does the crossing inside its electronics, so crossover cables are mostly history.

1. The port starts in MDI or MDI-X mode.
2. It listens for link pulses from the other side on its receive pair.
3. If it hears none after a short random time, it swaps modes.
4. The random timing stops both ends from swapping at the same moment forever.

Auto MDI-X is an optional feature in the 1000BASE-T standard, and nearly every modern gigabit port has it. It is still good to confirm, because old 10/100 ports may lack it, and on some Cisco platforms it needs speed and duplex set to auto.

```text
! Cisco IOS: copper access port with auto settings
interface GigabitEthernet1/0/1
 speed auto
 duplex auto
 mdix auto
```

### Coaxial Cable

Coaxial cable (coax) has one centre conductor, wrapped in insulation, then a metal shield, then a jacket. The name means both conductors share one axis.

```text
  Jacket | Shield | Insulator | Centre | Insulator | Shield | Jacket
  ======   ------   ........    ####     ........    ------   ======
```

- **Old Ethernet:** 10BASE5 (thick coax, 500 m) and 10BASE2 (thin coax, 185 m). Both were shared buses at 10 Mbps. Both are history.
- **Today:** cable TV and cable internet (RG-6, 75 ohm, F connector), CCTV (RG-59, BNC connector), and radio antennas (50 ohm types).
- **Cable broadband** runs a standard named DOCSIS over coax. See [WAN Technologies](../10-enterprise-and-datacenter/08-wan-technologies.md).

**Ohm** rating here means impedance, an electrical property of the cable. Do not mix 50 ohm and 75 ohm parts, because the mismatch reflects part of the signal back.

**Twinax** is a modern relative: two inner conductors in one shield. It is used in **DAC** (Direct Attach Cable), a short cable with a transceiver plug fused onto each end. A passive DAC reaches a few metres, which suits links inside a rack.

### Fibre Optic Cable

Fibre carries pulses of light through a thin glass strand. The **core** carries the light. The **cladding** around it is glass of a slightly different type, which bends the light back into the core. This effect is called total internal reflection.

Sizes are in microns (one micron is a thousandth of a millimetre). A human hair is roughly 70 microns wide.

```text
Single-mode (9 micron core): one straight path
 +--------------------------------------------+ cladding
 |--------------------------------------------| core
 +--------------------------------------------+ cladding

Multi-mode (50 micron core): many paths
 +--------------------------------------------+
 |   /\      /\      /\      /\      /\       |
 |  /  \    /  \    /  \    /  \    /  \      | core
 | /    \  /    \  /    \  /    \  /    \     |
 |/      \/      \/      \/      \/      \    |
 +--------------------------------------------+
```

**Multi-mode fibre (MMF)** has a wide core. Light takes many paths, called modes. Some paths are longer, so parts of a pulse arrive at different times. This smearing is called **modal dispersion**, and it limits MMF to a few hundred metres at 10 Gbps.

**Single-mode fibre (SMF)** has a core so narrow that light takes only one main path. There is almost no modal dispersion, so it reaches kilometres, and with amplifiers, across oceans.

Analogy: MMF is a wide corridor where runners take different routes and finish at different times. SMF is a narrow lane where everyone runs the same line. Where it stops: single-mode pulses still spread a little, because different colours of light travel at slightly different speeds.

| | Multi-mode | Single-mode |
|---|---|---|
| Core | 50 or 62.5 microns | About 9 microns |
| Typical light source | Low-cost VCSEL laser, 850 nm | Laser, 1310 or 1550 nm |
| Reach at 10 Gbps | 300-400 m | 10 km and beyond |
| Fibre cable cost | Higher per metre | Lower per metre |
| Transceiver cost | Usually lower | Usually higher |
| Typical use | Inside a building or data centre | Campus, metro, long distance |

The unit nm is nanometres, the wavelength or "colour" of the light. These wavelengths are infrared, so you cannot see them.

Fibre types and common jacket colours (colours are convention, so always check the printing on the cable):

| Type | Core | Jacket colour | Notes |
|---|---|---|---|
| OM1 | 62.5 | Orange | Legacy |
| OM2 | 50 | Orange | Legacy |
| OM3 | 50 | Aqua | Laser-optimised |
| OM4 | 50 | Aqua or violet | Common in data centres |
| OM5 | 50 | Lime green | Built for several wavelengths |
| OS2 | 9 | Yellow | Modern single-mode default |

Common reach figures from IEEE 802.3 (IEEE is the body that writes the Ethernet standards):

| Standard | Fibre | Wavelength | Reach |
|---|---|---|---|
| 1000BASE-SX | OM2 | 850 nm | 550 m |
| 1000BASE-LX | OS2 | 1310 nm | 5 km |
| 10GBASE-SR | OM3 / OM4 | 850 nm | 300 m / 400 m |
| 10GBASE-LR | OS2 | 1310 nm | 10 km |
| 10GBASE-ER | OS2 | 1550 nm | 40 km |
| 25GBASE-SR | OM3 / OM4 | 850 nm | 70 m / 100 m |
| 40GBASE-SR4 | OM3 / OM4 | 850 nm | 100 m / 150 m |
| 100GBASE-SR4 | OM3 / OM4 | 850 nm | 70 m / 100 m |
| 100GBASE-LR4 | OS2 | Around 1310 nm | 10 km |

Memory aid: S means short reach, L means long, E means extended. Faster speeds shorten multi-mode reach.

### Connectors

| Connector | Used on | Looks like |
|---|---|---|
| RJ45 | Twisted pair Ethernet | Clear plastic plug with a clip |
| RJ11 | Phone lines | Smaller than RJ45 |
| F-type | Cable TV, cable modems | Screw-on coax |
| BNC | CCTV, old Ethernet, test gear | Twist-lock coax |
| LC | Fibre, most SFP modules | Small square, usually a duplex pair |
| SC | Fibre, older gear, home fibre | Larger square, push-pull |
| ST | Fibre, legacy | Round, twist-lock |
| FC | Fibre, telecom, test gear | Round, screw-on |
| MPO/MTP | Many fibres, 40G and 100G | Wide flat ribbon plug, often 12 fibres |

A fibre link normally needs two strands, one for each direction. The transmit side at one end must reach the receive side at the other end. If a link stays down, try swapping the two strands at one end.

Fibre ends also come in polish types. **UPC** ends are flat-ish and usually have blue connector bodies. **APC** ends are cut at an 8 degree angle and are green. Never mate UPC with APC, since it damages the connectors and hurts the signal.

### SFP Transceivers

A **transceiver** is a small module that both transmits and receives. It turns the switch's electrical signal into light (or into copper signals). An **SFP** (Small Form-factor Pluggable) is a thumb-sized transceiver that slides into a slot on a switch.

Analogy: a travel plug adapter. The switch port is the same, and you swap the adapter to suit the cable. Where it stops: both ends of a link must match in standard and wavelength. Unlike a plug adapter, a mismatch gives you no link at all, even though each end looks fine.

| Module | Lanes | Speed |
|---|---|---|
| SFP | 1 | 1 Gbps |
| SFP+ | 1 | 10 Gbps |
| SFP28 | 1 | 25 Gbps |
| QSFP+ | 4 | 40 Gbps |
| QSFP28 | 4 | 100 Gbps |
| QSFP-DD, OSFP | 8 | 400 or 800 Gbps |

A **lane** is one independent data path inside the module. The Q in QSFP means quad, or four lanes.

Useful facts:

- Modules are **hot-pluggable**: you can add or remove them while the switch is on.
- The form factors come from **MSAs** (multi-source agreements), which are deals among vendors, not IEEE standards.
- SFP does not always mean fibre. Copper RJ45 SFPs exist, and so do DAC cables.
- Some vendors restrict third-party modules, so check the switch's compatibility list before you buy.
- **DOM** (Digital Optical Monitoring), also called DDM, lets you read temperature, voltage and light power from the module.
- A **BiDi** module sends and receives on one strand using two wavelengths. The two ends are different parts and must be a matched set.
- A **media converter** is a small box that joins copper to fibre when a device has no SFP slot.

### Wireless Media

Wireless sends data as radio waves, so the air is the medium. It is shared by everyone in range. Usually only one device on a channel can send at a time. That is why a Wi-Fi speed on the box is never what each user gets.

| Technology | Frequency | Typical reach | Notes |
|---|---|---|---|
| Wi-Fi | 2.4, 5, 6 GHz | Tens of metres indoors | Details in [Wi-Fi Basics](../08-wireless-and-mobile/01-wifi-basics.md) |
| Bluetooth | 2.4 GHz | Metres, more in some classes | Personal devices |
| Cellular (4G, 5G) | Many bands | Hundreds of metres to kilometres | Run by a mobile carrier |
| Microwave link | Often 6-80 GHz | Kilometres | Needs line of sight |
| Satellite (GEO) | Several bands | Very wide area | 35,786 km up, about 600 ms round trip |
| Satellite (LEO) | Several bands | Moves with the satellites | About 550 km up, often 25-60 ms round trip |
| Infrared | Light, not radio | A room | Blocked by walls, needs near line of sight |

GEO means geostationary orbit: the satellite seems to stay still in the sky. LEO means low Earth orbit: many satellites pass quickly overhead.

**The basic trade-off:** a higher frequency offers more capacity but reaches less far and passes through walls less well. So 2.4 GHz Wi-Fi reaches further than 6 GHz, but 6 GHz can carry much more.

More on phones and satellites in [Mobile, Satellite and IoT Networks](../08-wireless-and-mobile/04-mobile-and-iot-networks.md).

### Signal Attenuation

Attenuation is measured in **decibels (dB)**. A dB is a ratio, and losses simply add up along a path.

| Loss | Power left |
|---|---|
| 3 dB | 50% |
| 6 dB | 25% |
| 10 dB | 10% |
| 20 dB | 1% |
| 30 dB | 0.1% |

**dBm** is an absolute power level compared with 1 milliwatt: 0 dBm = 1 mW, and -3 dBm = 0.5 mW.

Why signals weaken:

- **Copper:** resistance turns signal into heat, and the loss grows with frequency. A Cat6 cable may lose about 20 dB over 100 m at 100 MHz.
- **Fibre:** glass absorbs and scatters a little light. Loss is tiny: about 0.35 dB per km at 1310 nm and about 0.2 dB per km at 1550 nm. Dirty connectors, poor joins and tight bends add more.
- **Wireless:** energy spreads out. Each doubling of distance costs about 6 dB in open space. Walls cost more.

What matters in the end is the **signal-to-noise ratio (SNR)**: how strong the signal is compared with the noise. A weak signal in a quiet place can work. A strong signal in heavy noise can fail.

How to fight attenuation:

1. Stay inside the rated distance.
2. Use better media, such as Cat6a or fibre instead of Cat5e.
3. Put a **repeater** or switch in the middle. A switch rebuilds the signal fresh, so each segment gets a new 100 m.
4. On long fibre, use optical amplifiers. They boost light without turning it into electricity, but they also boost noise.

### Testing the Media

A **cable tester** checks wiring order and broken wires. A **certifier** proves a run meets its category. A **TDR** (Time Domain Reflectometer) sends a pulse and times the echo to find how far away a fault is, and many switches have one built in. An **OTDR** does the same for fibre with light. A **visual fault locator** shines red light down a fibre so breaks glow.

### Choosing a Medium

| Need | Good choice |
|---|---|
| Desk or access point, under 100 m | Cat6 or Cat6a |
| Server to the switch in its own rack, under 5 m | DAC |
| Across a data centre, under 100 m | OM4 multi-mode |
| Between buildings | OS2 single-mode |
| Noisy factory floor | STP or fibre |
| Phones and laptops that move | Wi-Fi |
| Cannot dig a trench | Microwave link or satellite |

Between buildings, prefer fibre. Glass carries no electricity, so it blocks lightning surges and ground problems.

**PoE** (Power over Ethernet) sends power and data on one twisted pair cable, so access points and IP phones need only one wire. Power at the source is 15.4 W for 802.3af (PoE), 30 W for 802.3at (PoE+), and 60 W or 90 W for 802.3bt.

## Example

### Example 1: Choosing Media for Two Buildings

A company has two buildings 600 m apart. Desks are at most 70 m of cable from the wiring closet. Access points are 45 m from the closet. One server sits 2 m from the core switch. The buildings need 10 Gbps between them.

```text
 Building A                              Building B
 +-------------+  OS2 fibre, 600 m      +-------------+
 | Core switch |========================| Floor switch|
 | SFP+ LR     |  LC duplex, 10 Gbps    | SFP+ LR     |
 +-------------+                        +-------------+
    |       |                               |
    | Cat6a | Cat6a, PoE+                   | Cat6a
    |       |                               |
   PC    Wi-Fi AP                          PC
  (70 m)
```

| Link | Choice | Reason |
|---|---|---|
| Desk to closet, 70 m | Cat6a | Inside 100 m, ready for 10 Gbps |
| Access point, 45 m | Cat6a with PoE+ | One cable gives power and data |
| Building to building, 600 m | OS2, 10GBASE-LR | SR on OM4 stops at 400 m. LR goes 10 km |
| Server to switch, 2 m | SFP+ DAC | Cheap, simple and low power |

Copper is ruled out between the buildings, because 600 m is far past 100 m. Multi-mode is ruled out because 10GBASE-SR reaches 400 m on OM4.

### Example 2: A Gigabit Link Stuck at 100 Mbps

A user complains that copying files takes too long. The PC and the switch are both gigabit. A 1 GB file takes about 90 seconds, which is about 11 MB/s. Since 100 Mbps divided by 8 is 12.5 MB/s, this looks exactly like a 100 Mbps link.

Check the PC, on Linux or on Windows.

```bash
# Linux
$ sudo ethtool eth0
Settings for eth0:
        Supported link modes:   100baseT/Full
                                1000baseT/Full
        Speed: 100Mb/s
        Duplex: Full
        Port: Twisted Pair
        Auto-negotiation: on
        MDI-X: off (auto)
        Link detected: yes
```

```powershell
# Windows
PS> Get-NetAdapter | Format-Table Name, Status, LinkSpeed, MediaType

Name     Status LinkSpeed MediaType
----     ------ --------- ---------
Ethernet Up     100 Mbps  802.3
```

Check the switch.

```text
! Cisco IOS
Switch# show interfaces GigabitEthernet1/0/7 status

Port      Name     Status     Vlan  Duplex  Speed  Type
Gi1/0/7   PC-Anna  connected  10    a-full  a-100  10/100/1000BaseTX
```

The `a-` prefix means auto-negotiated. **Auto-negotiation** is how both ends agree on speed and duplex. Duplex says whether both sides can send at once (full) or must take turns (half). Both ends can do gigabit but they settled on 100 Mbps. That points to the cable.

Here is why. 100BASE-TX uses only 2 pairs (pins 1,2 and 3,6). 1000BASE-T needs all 4 pairs. If one wire on pins 4,5,7 or 8 is broken, gigabit fails but 100 Mbps still works.

A cable tester shows pin 7 open. Replacing the patch cord brings the link up at `a-1000`, and the same file copies in about 10 seconds.

Fuller truth: not all adapters drop to 100 Mbps cleanly. Some keep trying gigabit, and the link goes up and down instead. See [Common Network Problems and Fixes](../12-troubleshooting-and-tools/09-common-network-problems.md).

### Example 3: A Fibre Link Budget

A **link budget** checks that enough light reaches the far end. The values below are typical, not from one product's data sheet. Always use the real data sheet.

| Item | Value |
|---|---|
| Transmit power | -3 dBm |
| Receiver sensitivity (weakest usable signal) | -14 dBm |
| Fibre loss: 8 km of OS2 at 0.35 dB/km | 2.8 dB |
| Two connectors at 0.5 dB each | 1.0 dB |
| Two splices (joins) at 0.1 dB each | 0.2 dB |

1. Budget: -3 - (-14) = 11 dB of allowed loss.
2. Total loss: 2.8 + 1.0 + 0.2 = 4.0 dB.
3. Light received: -3 - 4.0 = -7 dBm.
4. Margin: 11 - 4.0 = 7 dB spare. Good. The margin covers ageing, repairs and dirty connectors.

Confirm it on the live link. This works only on modules with DOM, and the layout varies by platform.

```text
! Cisco IOS
Switch# show interfaces TenGigabitEthernet1/1/1 transceiver

                                         Optical   Optical
          Temperature  Voltage  Current  Tx Power  Rx Power
Port      (Celsius)    (Volts)  (mA)     (dBm)     (dBm)
--------  -----------  -------  -------  --------  --------
Te1/1/1   31.2         3.31     34.1     -3.0      -7.0
```

On Linux, `ethtool -m eth0` shows the same kind of data, if the driver and module support it.

## Going Deeper

**MHz is not Mbps.** Encoding packs several bits into each signal change. 1000BASE-T uses PAM-5, a code with five voltage levels. It runs at 125 million symbols per second on each of 4 pairs, and carries 2 data bits per symbol: 4 x 125 x 2 = 1000 Mbps. All 4 pairs also send in both directions at the same time, with echo cancellation separating the two directions. So the simple TX pair and RX pair picture fits only 10 and 100 Mbps. Auto MDI-X still applies at gigabit, because the two ends must agree on which pair is which.

**Why 100 m?** Early on, the timing of collision detection in half-duplex shared Ethernet also mattered. Today's full-duplex links have no collisions, so the real limit is signal quality: loss, crosstalk and delay across the cable. See [Collision and Broadcast Domains](../03-ethernet-and-switching/04-collision-and-broadcast-domains.md).

**Fibre is not faster in the way people think.** Signals in glass and in copper both travel at about two thirds of the speed of light in a vacuum, or roughly 200,000 km per second. Fibre wins on capacity and distance, not on travel time. More in [Bandwidth, Latency and Performance](06-network-performance-basics.md).

**WDM** (Wavelength Division Multiplexing) sends many colours of light down one fibre, each carrying its own data. CWDM uses a few widely spaced colours. DWDM packs in many, and carriers use it to reach terabits per second on one strand.

**Breakout cables.** A 40G QSFP+ port has 4 lanes of 10G, and a 100G QSFP28 port has 4 lanes of 25G. If the switch supports it, a breakout cable splits one port into 4 separate links. This is common in [Spine-Leaf Architecture](../10-enterprise-and-datacenter/02-spine-leaf.md).

**Too much light.** A long-reach module on a very short fibre can overload the receiver. The fix is a small in-line attenuator that reduces the light on purpose.

**10GBASE-T vs SFP+.** 10GBASE-T uses more power and adds more delay than SFP+ optics or DAC, which is why data centres tend to avoid it for server links. Cheap copper reuse is its main benefit.

## Common Mistakes

- Thinking Cat6 always does 10 Gbps at 100 m. It does not. You need Cat6a for that.
- Treating 100 m as the wall cable length. It is the whole channel, including patch cords.
- Buying Cat7 with RJ45 plugs because the number is bigger. Cat6a is the standard, tested choice.
- Believing a shielded cable is always better. Without proper grounding it can make things worse.
- Using a crossover cable on modern gear "to be safe". With Auto MDI-X, a normal straight-through cable works.
- Saying a PC to router link uses straight-through. Classically it needs crossover, because both are MDI.
- Assuming SFP means fibre. Copper SFPs and DACs exist.
- Connecting SR at one end and LR at the other. Both ends need matching standard and wavelength.
- Putting a single-mode module on multi-mode fibre. Mismatched fibre gives high loss or no link.
- Confusing MHz with Mbps. MHz is the cable's rated frequency. Mbps is data speed.
- Saying fibre is "the speed of light" and copper is slow. Signal travel speed is similar. Capacity and distance are the real difference.
- Looking into a fibre end. The light is invisible and can hurt your eyes. Always assume the link is live.
- Ignoring dirt. A dirty fibre end is one of the most common causes of optical faults. Clean and inspect before you replace parts.

## Interview Questions

**Q: What is the maximum distance for Cat6 at 10 Gbps?**
A: About 55 m, and as low as 37 m where alien crosstalk is heavy. For 10 Gbps over a full 100 m, I would use Cat6a.

**Q: What is the difference between Cat5e, Cat6 and Cat6a?**
A: All three do 1 Gbps at 100 m. Cat5e is rated to 100 MHz, Cat6 to 250 MHz and Cat6a to 500 MHz. Cat6 does 10 Gbps only on shorter runs, while Cat6a does it at 100 m.

**Q: What is the catch with Cat7?**
A: Cat7 is an ISO/IEC class, not a TIA category, and it was designed for non-RJ45 connectors. So a Cat7 cable with RJ45 plugs does not match its standard. I would choose Cat6a for 10 Gbps, or Cat8 for short data centre links.

**Q: Why are the wires in Ethernet cable twisted?**
A: Each pair carries a signal as two opposite voltages, and the receiver reads the difference. Noise hits both wires about equally, so it cancels out. Twisting also cuts crosstalk between pairs.

**Q: What is the difference between UTP and STP?**
A: UTP uses only twisting to fight noise. STP adds foil or braid shielding. STP suits high-noise places, but it costs more, is stiffer, and must be grounded correctly.

**Q: When do you use a crossover cable?**
A: Classically, between ports of the same type: switch to switch, PC to PC, router to router, and PC to router. Different port types use straight-through. Today, Auto MDI-X makes this mostly automatic.

**Q: What does Auto MDI-X do?**
A: It detects which pairs the other end transmits on and swaps its own transmit and receive pairs if needed. So either cable type works. On some platforms it needs speed and duplex set to auto.

**Q: What is the difference between single-mode and multi-mode fibre?**
A: Single-mode has a core of about 9 microns, and light takes one path, so it reaches many kilometres. Multi-mode has a 50 or 62.5 micron core, and light takes many paths, so modal dispersion limits it to hundreds of metres at 10 Gbps. Multi-mode suits data centres and single-mode suits campus and long-distance links.

**Q: What is an SFP, and how is it different from SFP+ and QSFP?**
A: An SFP is a hot-pluggable transceiver module for a switch port. SFP is 1 Gbps, SFP+ is 10 Gbps, and SFP28 is 25 Gbps, all on one lane. QSFP uses four lanes, giving 40 Gbps with QSFP+ and 100 Gbps with QSFP28.

**Q: What is attenuation and how do you deal with it?**
A: Attenuation is a signal weakening as it travels. It is measured in decibels, where 3 dB means half the power. I deal with it by staying inside distance limits, using better media, or adding a switch, repeater or optical amplifier.

**Q: A new fibre link will not come up. What do you check?**
A: First, that both modules match in standard and wavelength, and that the fibre type is right. Then I swap the two strands in case transmit and receive are crossed. Then I clean the connectors and read the receive power with DOM to see whether light is arriving.

**Q: A gigabit port shows 100 Mbps. What is your first guess?**
A: A cable fault. 100 Mbps needs 2 pairs and gigabit needs all 4, so one bad wire can drop the link to 100 Mbps. I would test or replace the cable, then check the port settings at both ends.

## Quick Recap

- Media carry bits as electricity (copper), light (fibre) or radio waves (wireless). All of them suffer attenuation.
- Twisted pair: Cat5e for 1 Gbps, Cat6 for 10 Gbps up to about 55 m, Cat6a for 10 Gbps at 100 m, Cat8 for 25-40 Gbps at 30 m. Cat7 is not a TIA category.
- Same port type needs crossover and different types need straight-through, but Auto MDI-X now handles this.
- Single-mode (9 microns, yellow) goes far. Multi-mode (50 microns, aqua) is for shorter runs.
- SFP family modules plug into switch ports. Both ends of a link must match.
- Wireless is a shared medium. Higher frequency means more capacity but less reach.
- Losses in dB add up. 3 dB is half the power, and a link budget proves a link will work.
