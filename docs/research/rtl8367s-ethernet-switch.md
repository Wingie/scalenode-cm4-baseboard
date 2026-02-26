# RTL8367S Ethernet Switch Research

## Overview

The **Realtek RTL8367S** is a 5-port 10/100/1000Mbps Ethernet switch controller with
5 integrated Gigabit PHYs. It's ideal for the Agentosaurus cluster board because it
can directly interface with the CM4's transformer-side Ethernet outputs through magnetics.

**Note:** The RTL8367S is the successor to the RTL8367RB. The Turing Pi 2 uses the
related **RTL8370MB** (8-port variant). For our 4-node design, the 5-port RTL8367S
is sufficient (4 CM4 ports + 1 uplink).

## Key Specifications

| Parameter | Value |
|-----------|-------|
| Ports | 5× 10/100/1000BASE-T |
| Integrated PHYs | 5 |
| RGMII interfaces | 2 (EXT0, EXT1) — for external MAC connections |
| Package | LQFP-128 (14×14mm) or QFN-128 (10×10mm) |
| Core voltage | 1.1V (internal LDO from 3.3V or external) |
| I/O voltage | 3.3V |
| Power consumption | ~1.5W typical |
| Operating temp | 0°C to 70°C (commercial) |

## Functional Block Diagram

```
                    MDIO/I2C
                       │
                ┌──────┴──────┐
                │  RTL8367S   │
                │             │
    PHY Port 0 ─┤  ┌───────┐ │
    PHY Port 1 ─┤  │Switch │ │
    PHY Port 2 ─┤  │Matrix │ ├── EXT RGMII 0 (optional)
    PHY Port 3 ─┤  │       │ ├── EXT RGMII 1 (optional)
    PHY Port 4 ─┤  └───────┘ │
                │             │
                └─────────────┘
```

## Connection to CM4 Modules

### PHY-to-PHY through Magnetics

Each CM4 has an integrated Ethernet PHY (BCM54210PE) that outputs transformer-side
differential pairs. The RTL8367S also has integrated PHYs. To connect them:

```
CM4 Module          Magnetics (1:1)         RTL8367S
┌──────┐         ┌─────────────┐         ┌──────────┐
│ BCM  │ TRD0_P ─┤ ┌─────────┐│  TXDP0 ─┤ PHY      │
│54210 │ TRD0_N ─┤ │ Pair A  ││  TXDN0 ─┤ Port 0   │
│  PE  │ TRD1_P ─┤ │ Pair B  ││  TXDP1 ─┤          │
│      │ TRD1_N ─┤ │ Pair C  ││  TXDN1 ─┤          │
│ PHY  │ TRD2_P ─┤ │ Pair D  ││  TXDP2 ─┤          │
│      │ TRD2_N ─┤ └─────────┘│  TXDN2 ─┤          │
│      │ TRD3_P ─┤             │  TXDP3 ─┤          │
│      │ TRD3_N ─┤             │  TXDN3 ─┤          │
└──────┘         └─────────────┘         └──────────┘
```

### Magnetics Selection

For PHY-to-PHY connections, use **1:1 ratio** Ethernet magnetics (transformers).
Common options:
- **H6096NL** (already in upstream design for single-port)
- **HX1188NL** (Pulse Electronics, 4-port integrated magnetics)
- Individual per-pair transformers or quad magnetics modules

For 4 CM4 slots + 1 uplink = **5 sets of magnetics** needed.
The uplink port also needs an RJ45 jack with integrated magnetics (like the
upstream SS-74800-144 or similar).

## RGMII Interfaces (Not Used for CM4)

The RTL8367S has 2 external RGMII interfaces (EXT0, EXT1) for connecting external
MAC devices. These are NOT needed for CM4 connections (which use the integrated PHYs)
but could be used for:
- Future expansion with SoCs that have RGMII output
- An additional external device

## Configuration Interface

### Strap Pins
The RTL8367S uses hardware strap pins at power-up to configure:
- Management interface type (MDIO or I2C)
- PHY address
- LED mode
- RGMII timing

### MDIO/I2C Management
After boot, the switch can be configured via:
- **MDIO** (Management Data Input/Output) — standard IEEE 802.3 clause 22/45
- **I2C** — alternative management interface
- **SMI** (Serial Management Interface) — Realtek proprietary

The ESP32-S3 BMC can manage the switch via I2C or MDIO to:
- Configure VLANs
- Monitor port status and counters
- Control LED behavior
- Set QoS policies

## Power Supply Requirements

| Rail | Voltage | Current (typ) | Notes |
|------|---------|---------------|-------|
| AVDD_3.3V | 3.3V | ~200mA | Analog PHY power |
| DVDD_3.3V | 3.3V | ~150mA | Digital I/O |
| DVDD_1.1V | 1.1V | ~300mA | Core logic (internal LDO or external) |

**Total:** ~1.5W typical

### Decoupling
- 100nF on every VDD pin
- 10µF bulk per rail
- Ferrite beads to separate analog and digital 3.3V

## PCB Layout Guidelines

### Differential Pairs (PHY Side)
- **Impedance:** 100Ω differential
- **Trace width:** ~4-5 mil (depends on stackup)
- **Spacing:** Tight coupling (gap = trace width)
- **Length matching:** ±50 mil within a pair, ±500 mil between pairs
- **Max trace length:** 6 inches from IC to magnetics

### Power
- Solid ground plane under the IC
- Short, wide power traces
- Star-topology for 3.3V to minimize crosstalk between PHY ports
- Separate analog and digital grounds, joined at one point under the IC

### Crystal
- 25MHz crystal oscillator (or external clock)
- Keep crystal traces short (<10mm)
- Guard ring around crystal area

## LED Control

The RTL8367S supports per-port LED control with configurable modes:
- Link/Activity
- Speed indication (10/100/1000)
- Duplex
- Custom via register

For the Agentosaurus, we'll use WS2812B LEDs driven by the ESP32-S3 BMC instead,
reading port status via MDIO/I2C from the switch.

## Turing Pi 2 Reference

The Turing Pi 2 uses the **RTL8370MB** (8-port variant) in a similar configuration:
- 4 CM4 modules connected through magnetics to 4 PHY ports
- 1 uplink port to RJ45
- Managed via BMC (their BMC is an Allwinner T113-S3)
- Power control per slot
- USB muxing for eMMC flashing

Our design follows the same proven architecture but uses the smaller RTL8367S
(5-port sufficient for 4 nodes + 1 uplink).
